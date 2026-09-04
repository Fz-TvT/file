# Clock 淘汰算法：并发场景下的若干问题

## 1. 顺序是确定的：年龄为主，counter 只做"延迟"

主序是 list 位置（入列顺序）。时钟指针 `oldest_` 从最老节点出发，每次 `FindEvictCandidate` 只会前移一格：

- `curCounter == 0` → 选中，指针留在原地（等 `Erase` 时再前移）；
- `curCounter > 0` → 减 1，指针前移，节点保留。

所以 counter 相同（平局）时，靠指针位置裁决：靠前的（更老的）先被选中，不会出现随机性。比如 `[A(1) B(1)]`：第一圈两个都减成 0，第二圈先选 A 再选 B，顺序稳定。

## 2. 并发时 counter 相同会产生的几个问题

> 这里"并发"指的是：内存淘汰本身是单线程（`isDone_` 闩锁保证同一时刻只有一个 `EvictionTask`，见 `Evict()`），但请求线程（发布、命中、删除）会并发地 `Add`/`Erase`/`OnCacheHit` 修改同一个列表。

### ① counter 粒度太粗，相同值分不清真实热度

counter 只取 0/1/2（失败回插才到 5），`maxCounter` 封顶。所以一个每秒命中 100 次的对象和一个每分钟命中 1 次的对象，只要都顶到 `maxCounter`（Q2），在算法眼里完全等价，顺序只能退回到"位置"决定——热的旧的很可能先于冷的新的被淘汰，这是"LRU 不生效"的根源之一。

### ② 初始 counter 被引用计数量化成两档

`ComputeClockAddCounter`（`eviction_strategy.cpp:30-34`）只有 `GetRefWorkerCount == 0` → Q1、`≥1` → Q2 两档。有 1 个远端 worker 引用和有 100 个远端 worker 引用的对象，初始保护完全相同。并发下多个 worker 引用同一对象，不会给它任何额外保护。

### ③ 选中与命中的 TOCTOU：刚被访问的对象可能被淘汰

`FindEvictCandidate` 在 list 写锁下读 `curCounter==0` 选中；命中路径在锁外用 per-key CAS 自增 counter（`eviction_list.cpp:133-144`），两者没有原子性：

并发命中恰好发生在选中瞬间 → 命中先读 0、选中也读 0 → 选中成立，命中那次增的自增就作废了，一个刚被 Get 的对象照样被淘汰。

Clock 模式没有 `lastAccessMs`，无法像 heat 策略那样用"最近访问保护窗口"挡住这个窗口（heat 的 `IsHeatCandidateProtected` 就是干这个的）。

### ④ list 写锁放大扫描延迟

`FindEvictCandidate` 持有 list 写锁扫描（`eviction_list.cpp:407`），而新对象插入 `emplace_back` 和 `Erase` 也要同一把写锁。最坏情况（所有节点 counter>0）下一次选中要绕最多 2 圈才能找到 0，持锁时间 O(整表)，高淘汰压力下会短暂阻塞并发的发布和删除。命中（已有节点）走 CAS 不碰写锁，不受影响。

### ⑤ 相等 counter 下"年龄优先"替代"最近访问优先"

两个 counter 相同的节点，选谁只看谁更老，不看谁更久没被访问。严格 LRU 该淘汰"最后访问时间最久"的 B，clock 却淘汰"更老但刚被访问过"的 A——这是时钟算法对严格 LRU 的固有偏差，不是 bug，是设计取舍。
