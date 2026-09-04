# 标题

**RFC: 将 post-set async spill(SSD 副本)与 eviction 解耦,实现 D2H 对象"发布即入可靠落盘管道"(write-behind)**

> 状态:Draft　|　提交日期:2026-09-04　|　分支建议:`codex/pr2257-async-release-16`

---

## 背景与目标描述

**现象。** 开启 `enable_async_spill_replica_after_set=1`(D2H 整对象发布后异步建 SSD 副本)的预埋(fill)阶段,SSD 副本覆盖率远低于预期。以 2026-09-05 case2 现场日志为准:

- 发布 4,444,160 个不同对象(offer 4,446,282),异步副本**完成仅 735,091(16.5%)**,**丢弃 3,709,069(83.4%)**;
- `distinct − completed == dropped_total`(3,709,069)严格相等,即**缺口全部来自被丢弃**,没有写失败(`FAILED` 未增长);
- submitted 737,213 vs completed 735,091:提交后因瞬时资源满被丢的仅 ~2.1k —— **绝大多数丢发生在"还没发起任何 IO"的准入阶段**;
- 异步活跃窗口内 key 重复率 0.00%(每个 key 只被 offer 一次),排除"重复发布误判";
- 全 run 内存反复触发 drop-trigger/高水位 flip 9,893 对,内存长期钉在 0.80–0.82;
- 磁盘并非瓶颈:引擎 `avg_inflight≈0.16/48`,写吞吐 p90≈934 MiB/s、峰值窗口 1521 MiB/s,IO 时延均值 ~104µs;ingest 均值 ~451 MiB/s。磁盘有 3 倍以上余量。

**根因(现状机制缺陷)。**
1. **一次性准入丢弃、不重试**:`TrySpillAfterPublish` 对每个对象只"试一次",任一闸口拒绝即 `DROPPED` 永久放弃;
2. **副本写所有权被 async 与 eviction 共享**:二者共用一张 `registeredSpills_`(按 key、不含 version)去重表"抢注册,谁先注册谁写",后到者让路/丢弃——异步与回收写同一份 SSD 副本是"你写还是我写"的互斥关系,而非明确分工;
3. **新对象发布后毫秒级即可被 eviction 选中回收**,存在"内存释放先于 SSD 副本落地"的竞态窗口;drop 压力下普通无副本对象被 `RETAIN` 卡住或可丢对象被直接丢。

**目标。**
1. D2H 对象 SSD 副本覆盖率从 ~16.5% 收敛到 ~100%(`DROPPED→~0`);
2. 把"写 SSD 副本"收编为**一个不可丢、由副本消费方驱动的可靠职责**,eviction 只负责"副本落地后释放内存",从机制上消除双写、让位与误伤;
3. 在**不丢唯一副本**的前提下保证内存可按需回收:释放速度 = 磁盘副本 drain 速度;极端压力用**背压(发布变慢)** 代替丢弃;
4. 减少 drop-trigger 抖动,多数回收轮直接走 `FREE_MEMORY`。

---

## 建议的方案

### 语义取舍(先定)

| 选项 | 语义 | 代价 |
|---|---|---|
| **A. write-behind(推荐,本文默认)** | SET ack 快;回收内存前必落盘 | 崩溃窗口 = 写 DRAM 成功→SSD publish 之间(与现状普通 spill 一致) |
| **B. 同步 write-through(T5)** | SET ack 即已持久(组提交 barrier) | ingest 上限 = 磁盘写吞吐(~2 GiB/s 量级),每写多一跳 |

本文按 A 展开;B 作为可选后续,不改变 A 的接口形态。

---

### 方案总览

#### 一、修改前的方案(现状)

**职责划分。** "写 SSD 副本"不是一个单一职责,而是两个发起方共享同一动作。把现状按等价分层结构画出来,能直接看到它**不是"没有角色",而是"角色被共享基础设施揉在一起、靠注册表仲裁先后"**:

> 读图:`[ ]`=队列/集合,`◈`=线程池;同一色块表示共享、无归属的构件。现状的队列/线程池**全都存在,但全是共享的、无背压**,async 并没有自己的副本队列。

```
发布路径(MSetD2H)                         回收路径(memEvictTask 线程)
  │ ①TrySpillAfterPublish(每 key 准入)        │ EvictionTask:挑候选→GetObjectNextAction
  ▼ (async 没有自有队列)                       ▼
  │  过闸→丢进共享线程池;被拒→DROPPED   ┌── 仲裁: registeredSpills_(key→ver) ──┐
  ▼                                   │  eviction TryRegisterSpill 抢注册;    │
 ┌──────────────── 共享执行层(无角色分层) ────────│  失败→K_TRY_AGAIN 让路          │
 │  ◈ spillTaskThreadPool_(8 线程,共用任务队列)  └──────────────▲────────────────┘
 │   async 与 eviction 都把"写副本"任务丢进这里                  │
 │    │ PrepareAsyncSpill(锁 + bounce 快照)                    │
 │    ▼                                                        │
 │  SubmitAsync ─► [asyncReservations_:条数/字节门限]+ bounce 池 │
 │    ▼                                                        │
 │  SpillIoEngine:[writeQueue_ ≤192/24MB]─► io_uring [inflight_ ≤48]─► SSD │
 └─────────────────────────────────────────────────────────────┘
   完成 → FinalizeAsyncSpill:publish 副本 + (按来源)置 IsSpilled / FREE 内存
```

**这张图的要点:** 三个"队列/池"(`spillTaskThreadPool_` 任务队列、引擎 `writeQueue_`、io_uring `inflight_`)都是**跨两条路径共享、后进先被仲裁**的;唯一像"队列"的 `asyncReservations_` 只做容量门限,满了就 `K_TRY_AGAIN` 拒绝而不是排队。**没有任何一个有界、可背压、归属 async 的副本队列**——这就是覆盖率缺口的结构性成因。

**隐式角色(边界模糊)。** 现状里存在 4 个角色,但没有明确归属:

| 隐式角色 | 现状承担者 | 问题 |
|---|---|---|
| 副本发起方 | **async 与 eviction 两个**,无单一 owner | 同一动作两个入口 → 需要仲裁谁写 |
| 仲裁者 | `registeredSpills_`(按 key,先到先得) | 仲裁 = 让位 + 丢弃,产生 83% DROPPED 与旧版本误伤 |
| 执行者 | 共享 `spillTaskThreadPool_` + `SubmitAsync` + 引擎 | async 与 eviction 抢同一 8 线程/同一 qd |
| 失败语义 | 后到者被仲裁者否决 | async 后到=永久丢;eviction 后到=让路等下轮 |

**双方共用/共享的东西:**
- 同一张 `registeredSpills_` 去重表(仅按 `key`,不含 version)——既是 async 的去重表,也是 eviction 的"正在写"注册表 → **所有权互斥**:谁先注册谁写,后到者让路(async 后到=永久丢副本;eviction 后到=本轮放弃等下一轮);
- 同一个 `spillTaskThreadPool_`(8 线程)与同一个 `SpillIoEngine`(io_uring,`qd=48`,writeQueue 192/24MB);
- eviction 的释放决策依赖"这次回收有没有把副本写出来":无副本对象只有 `SPILL`(写完才 FREE)或 `RETAIN`(卡住)或 drop 模式下被丢。

**为什么现状长成这样(成因):**
1. **叠加而非设计**:async spill replica 是后加的特性(post-set 兜底),当时直接**复用**了成熟可靠的 eviction→spill 落盘管线(`PrepareAsyncSpill` / `SubmitAsync` / `registeredSpills_` 防并发),只在准入处加了闸口区分 → 两条调用路径共享同一个"写并 publish 副本"动作,不得不用共享注册表仲裁;
2. **语义天生冲突**:async 要"冗余副本、不急释放"(`releaseMemoryOnSuccess=false`),eviction 要"写后 free 腾内存"。两个不同目标却调同一个原语 → 谁先触发谁就成了事实上的写作者;
3. **复用错了抽象**:`registeredSpills_` 按 key 去重本是为"同一对象同一时刻至多一个在途写"(防双写同一文件区);被 async/eviction 复用成"所有权仲裁"后,arrival order 决定一切 → eviction 抢先注册,async 就永久放弃;
4. **只有 push,没有 pull**:async 是"发布那一刻推一把",eviction 是"回收那一刻推一把",都没有"消费队列/重试/背压"概念 → 一旦某次 push 被拒,副本就永远缺。

**关键流程(修改前),以预埋对象 X 为例:**
1. MSetD2H 把 X 写进 DRAM;发布后 `multi_publish` 收集 D2H 对象列表调 `TrySpillAfterPublish`(`worker_oc_eviction_manager.cpp:425`),同时对 X 做 `Add(key)`(`worker_oc_service_multi_publish_impl.cpp:797`)令其毫秒级可被 eviction 选中;
2. async 对 X 试一次:若 eviction 已先注册同一 key → X 判 `duplicate` → **`DROPPED`**(83% 落在此);若通过 → `PrepareAsyncSpill`(`:3059`)→ `SubmitAsync`(`worker_oc_spill.cpp:945`)→ 落盘 `publish`(`FinalizeAsyncSpill :3102`)→ 置 `IsSpilled`,**不释放内存**(`releaseMemoryOnSuccess=false`);
3. 内存压力 → `EvictionTask`(`:1291`):X 若已 SPILLED → `FREE_MEMORY` 直接释放;若仍无副本 → `RETAIN` 卡住(稳态)或被当作可丢对象丢掉(drop 压力);内存于是钉在 0.80–0.82 反复 flip。

**现状代码坐标:**
- 准入丢弃:`worker_oc_eviction_manager.cpp:425-532`(`TrySpillAfterPublish`),丢弃点 :448/:452/:522
- 去重/所有权:`:440` emplace、`:3158` `TryRegisterSpill`、`:3228` `SubmitSpillTask`(`K_TRY_AGAIN already running`)
- 释放决策:`:1103` `GetObjectNextAction`、`:1127-1139` drop 分支、`:1427-1433/:1471-1475` dropOnPressure latch
- 副本写:`:3059` `PrepareAsyncSpill`、`:3093` `SubmitAsync`、`:3102` `FinalizeAsyncSpill`、`:3183` `CompletePostPublishSpill`(`K_TRY_AGAIN→DROPPED :3197`)
- 引擎:`spill_io_engine.cpp:160`(容量 192/24MB)、`:212`(refill 到 qd48)、`:252`(读写 credits)

**现状缺陷(为什么覆盖率只有 16.5%):**
1. async 是"一次性准入",eviction 抢先注册或瞬时满就直接丢、不补;
2. 关键路径上大部分 X 由 eviction 抢写(或没人写),async 计数却是 `DROPPED`,导致"发布对象 ↔ 有 SSD 副本"被解绑;
3. "内存释放"与"副本落地"之间没有强保证,drop 模式留下数据丢失窗口。

---

#### 二、修改后的方案

**职责划分。** 把三个角色彻底分开:写副本只归"副本消费方",eviction 只做"副本落地后释放内存",发布方只负责"注册意图 + 被背压时等待":

> 读图:`[ ]`=队列/集合,`◈`=线程池。与修改前相比,**结构性的新增**是三处:① 归属副本的有界队列;② 专属消费线程池;③ 队列满→背压、副本完成→唤醒的边。

```
发布路径(MSetD2H)                            eviction(memEvictTask 线程)
  │ ① 写 DRAM + 注册副本意图 (key,version)         │ 已 SPILLED            → FREE_MEMORY
  ▼                                               │ PENDING 唯一副本      → 不写不丢不 free
  │                                               │ legacy 无意图对象      → 兜底 SPILL
  ▼  满→背压:阻塞/限速发布(而非丢)   ◄───────────  │ 只剩 PENDING 无可释放  → 停(不空转)
 [replica 队列 = 有界 PENDING 意图集合]             │
  │  (条数/字节门限;副本完成才出队并清 PENDING)        │ 副本完成 → Add(key) 进候选(之后可被 FREE)
  │ ② 摘意图                                         │
  ▼                                               │
 ◈ 副本写线程池(专属、受控并发)                       │
  │  锁 + bounce 快照 → SubmitAsync(tag=replica)     │
  ▼ ③                                             │
 SpillIoEngine:[writeQueue_]─► io_uring [inflight_ ≤48]─► SSD
  ▲  写内优先级: replica > reclaim(T4)               │
  │  ◄──────────────────────────────────────────────│
  │  eviction 的 legacy 兜底写 tag=reclaim,          │ ④ 完成回调:
  │  与副本写共享引擎但按优先级让位                    │   · 置 IsSpilled + 清 PENDING
  │                                                │   · 唤醒被背压的发布(背压解除)
  └────────────────────────────────────────────────┘
```

**这张图的要点:** 副本写从"往共享线程池推一把"变成"入一个有界 replica 队列、由专属线程池按磁盘 drain 速率消费";`PENDING 集合`(队列)+ `背压/唤醒边`是相对现状**真正新增的构件**;eviction 拆到右侧,不再抢注册、不再对 PENDING 对象发起写。

**与修改前的本质区别:**
- **所有权单一化**:SSD 副本写只有一个 owner(副本消费方);`registeredSpills_` 退化为纯并发保护(同一 `(key,version)` 同一时刻只允许一个在途写);
- **交付从"试一次"变"保证送达"**:每个发布对象打 PENDING 意图,由消费线程可靠落盘,瞬时失败有限重试,不再有"本可写却放弃"的 `DROPPED`;
- **释放条件分层**:唯一副本对象 = 副本落地才可 free(dropSafe 对象不受影响,保留旧 drop 语义);PENDING 对象不进 eviction 候选,从机制上消除"放内存先于副本"的竞态与空转;
- **过载用背压代替丢弃**:evictor 无可释放即停,信号回传发布分配等待,把 ingest 钳到磁盘 drain 速率。

**关键流程(修改后),以预埋对象 X 为例:**
1. MSetD2H 写 DRAM + X 注册 PENDING(幂等,`(key,version)`);X **暂不进 eviction 候选**;
2. 副本写线程锁+X、快照(bounce)、`SubmitAsync` → 落盘 `publish` → 置 `SPILLED`、清 PENDING、X 进 eviction 候选;
3. 内存压力 → eviction:X 已 SPILLED → 一轮 `FREE_MEMORY` 释放;若磁盘暂时跟不上,PENDING 堆积,evictor 发现无可释放即停,发布侧被有界背压 —— **X 不会在副本落地前被丢**。

---

#### 三、修改前 / 修改后 对比

**3.1 角色结构对照(现状隐式结构 vs 提议角色分层)**

| 维度 | 现状 | 提议(角色分层) |
|---|---|---|
| 副本发起方 | 两个(async + eviction) | 一个(副本消费方) |
| 所有权 | 靠先到先得仲裁 | 显式:发布即注册 PENDING,归消费方 |
| 仲裁表 | `registeredSpills_` 兼当仲裁器 | 退化为纯防并发保护(同 (key,version) 单在途) |
| 失败语义 | 丢弃/让路(覆盖率 16.5%) | 回队重试 + 背压(覆盖率 ~100%) |
| 副本写时机 | 发布推一次 + eviction 在水位后补写 | 发布即持续 drain,水位不再承担写副本 |
| 内存释放 | 依赖 eviction 写副本才 FREE 或 RETAIN 卡住 | 副本落地后 Add 进候选 → FREE;只丢 dropSafe |
| 结构与可诊断性 | 职责隐藏在共享管道里,靠注释与日志分辨 | 三个角色各司其职,单 owner、单执行者、明确的等待方 |

**3.2 机制总览**

| 维度 | 修改前(现状) | 修改后(方案) |
|---|---|---|
| 副本交付 | 单次尝试,三闸口拒绝即 `DROPPED`,不重试不补 | 发布即注册 PENDING,可靠消费 + 有限重试,`DROPPED` 只留给不可恢复对象 |
| 写副本所有权 | async 与 eviction 共用 `registeredSpills_` 抢注册,谁先谁写,后到让路/丢 | 副本写只归副本消费方;eviction 对 PENDING 对象不再写;所有权单一 |
| 去重键 | `key` | `(key, version)`,消除旧版本占表误伤新版本 |
| 内存释放条件 | 已有副本→FREE;无副本靠 eviction 写后 FREE 或 `RETAIN` 被动卡;drop 压力下无副本可丢对象被丢 | 分层护栏:mustPersist 唯一副本 → 副本落盘才可 FREE;dropSafe → 保留旧 drop 语义;极端压力 → 背压而非丢唯一副本 |
| eviction 候选时机 | 发布后毫秒级 `Add(key)` 即被选中回收(与副本竞态) | 副本落地(SPILLED)后才进候选,消除"放内存先于副本"竞态与空转 |
| 资源 | 副本写与回收写共享同一线程池/引擎,无区分 | 副本写独立并发 + 引擎内写优先级(副本 > 回收),回收洪峰饿不死副本 |
| 过载处理 | 丢对象、drop 抖动 | 队列满/内存满 → 背压发布路径,数据不丢 |
| 可观测性 | DROPPED 计数(含义混杂准入/让位/瞬时满) | 新增 replica_pending、副本 drain 速率、无可释放停顿等指标;DROPPED 语义收窄 |

**3.3 内存释放决策表(`GetObjectNextAction`)**

| 对象状态 | 修改前 | 修改后(稳态) | 修改后(drop 压力) |
|---|---|---|---|
| 已 SPILLED | FREE_MEMORY | FREE_MEMORY(不变) | FREE_MEMORY(或 dropSafe 旧 drop) |
| PENDING、唯一副本(mustPersist) | RETAIN 卡住 或 极端情况被丢 | 不写不丢不 free,等副本落地自动转 FREE;evictor 见只剩此类即停 | 停 + 背压,不丢唯一副本 |
| PENDING、dropSafe | 同上 | 同左(副本会落,不抢写) | 允许旧 drop(DELETE/END_LIFE) |
| 无副本无意图(legacy) | SPILL(写后 free) | SPILL(现状路径不变) | 旧 drop 语义 |
| 有 L2 / 非 primary 等 dropSafe | DELETE/END_LIFE | 同左 | 同左 |

**3.4 关键路径(对象 X)对比**

| 阶段 | 修改前 | 修改后 |
|---|---|---|
| 发布 | 写 DRAM;`TrySpillAfterPublish` 试一次;X 即刻可被 eviction 选中 | 写 DRAM + X 注册 PENDING;X 暂不进 eviction 候选 |
| 副本 | eviction 先注册→ X `DROPPED`(83%);通过才落盘,不 free 内存 | 副本写线程可靠落盘 + publish + 置 SPILLED + 清 PENDING + X 进候选 |
| 回收 | X 无副本 → `RETAIN` 卡住或被 drop;内存 0.80–0.82 反复 flip | X 已 SPILLED → 一轮 FREE_MEMORY;磁盘跟不上 → 发布被背压,X 不被丢 |

---

### 分模块改动(T1–T5 落地)

1. **可靠入队与消费(T1)**:
   - 改 `worker_oc_eviction_manager.cpp:425 TrySpillAfterPublish`:删除准入永久丢弃闸口,改为按 `(key,version)` 幂等注册 PENDING 并调度;**瞬时失败(锁竞争/引擎满)→ 有限重试/回队**,`CompletePostPublishSpill:3183` 的 `K_TRY_AGAIN→DROPPED` 改为回队语义,`DROPPED` 仅保留给确定不可恢复(对象已被删)的对象。
   - 释放条件可借用现状便利:`worker_oc_spill.cpp:988` 提交时已 `memcpy` bounce 快照,DRAM 依赖窗口 = "提交成功(拿到快照)之前";放宽到"快照在手即可放 DRAM"作为后续优化,首版仍以 publish 完成为准。
2. **释放规则分层(T3')**:改 `GetObjectNextAction:1103 / EvictObject:1171 / EvictionTask:1291` 实现上表;把 `multi_publish:797 Add(key)` 时机从"发布后即可回收"改为"副本落地后再 Add",PENDING 对象天然不进候选。
3. **所有权与版本化(T2)**:`registeredSpills_` 判重含 version(`TryRegisterSpill:3158`);`SubmitSpillTask:3228` 只对无副本意图的 legacy 对象触发(兜底),`K_TRY_AGAIN` 退化为纯并发保护,消除让路仲裁与双写。
4. **资源隔离/配额(T4)**:写请求打 `replica`/`reclaim` tag,`spill_io_engine.cpp:252 TakeNextRequest` 增加写内优先级(仿 read/write credits,默认副本权重高);副本消费并发独立。
5. **反压回环(T3' 第 3 条)**:evictor 发现只剩 PENDING 无可释放即停止不空转;信号回传发布分配等待(有界),把持续 ingest 钳制到磁盘 drain 速率。`memory_drop_trigger` 在稳态不应被触发。
6. **同步 write-through(T5,可选)**:发布路径对该批副本 publish 做组提交 barrier 后再回 ack。

### 分阶段上线与回滚

- Phase A(T1 意图 + 可靠消费):先验证覆盖率收敛到 ~100%,最小改动、最可回滚;
- Phase B(T3' 分层释放 + Add 时机后移);
- Phase C(T2 版本化所有权 → T4 配额/优先级 → T5 背压)。
- 全部包在 `enable_async_spill_replica_after_set` 下新增的独立开关后;异常即关对应开关回退上一阶段。

---