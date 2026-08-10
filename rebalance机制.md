# rebalance 功能

## 一、两条机制

**机制 A：master 调度的内存 rebalance（`FLAGS_enable_memory_rebalance`）**
Worker 每 ~30s 向 master 上报资源（node_selector.cpp:289-319 `ReportResource`）→ master 的 MemoryRebalanceScheduler 决定"要不要、让谁迁给谁"，把**目标 worker 直接写死在任务里**发给源 worker → 源 worker 用 RebalanceExecutor 从淘汰链表里挑旧对象迁过去。

**机制 B：worker 侧写路径的节点选择（`NodeSelector::SelectNode`）**
DataMigrator 在"向另一 worker 迁移/重定向"时用它选目标（data_migrator.cpp:602）。rank 列表来自 master 回填的所有 worker 可用内存（node_selector.cpp:349-362）。

---

## 二、日志

**master 侧**（统一前缀 `[MemoryRebalance]`，memory_rebalance_scheduler.cpp）：

- `[MemoryRebalance] select source=...(xx%) target=...(yy% -> zz%), max_bytes=...` — 每次选出任务的决策点（L622）
- `[MemoryRebalance] assign task <id> source=... target=... max_bytes=... timeout_ms=... deadline_ms=...` — 任务下发（L132）
- `[MemoryRebalance] finish task ... status=... migrated_bytes=... migrated_objects=... failed_objects=... reason=...` — 任务结束（L203）
- `[MemoryRebalance] expire task ...` / `hold in-flight target=...` / `release held in-flight ...` — 超时/在途占用（L227、L365、L431）

**worker 侧**（rebalance_executor.cpp + data_migrator）：

- `Start rebalance task <id>, assigned cluster master: ...`（L393）
- `Finish rebalance task <id>, status: %d, maxBytes: ..., migratedBytes: ..., migratedObjects: ..., failedObjects: ..., costMs: ..., reason: ...`（L420）— 这是判断迁移量最直接的日志
- `Ignore duplicated rebalance task ...` / `Reject rebalance task ... because task ... is still running`
- 数据面：`[Migrate Data] Processing data migrate begin, migrate type: ..., object size: ..., task id: ...`（data_migrator.cpp:148）和 `[Migrate Data Process] ... objects finished ...`（L111）

| 文件 | 包含的日志 |
| --- | --- |
| `datasystem_worker.INFO.<ts>.log` | **全部日志（含 INFO）** —— `[MemoryRebalance] select/assign/finish`、`Start/Finish rebalance task`、`[Migrate Data]` 都在这 |
| `datasystem_worker.WARNING.<ts>.log` | WARNING+ERROR —— `[MemoryRebalance] expire task`、`release held in-flight`、`Reject rebalance task ... is still running` |
| `datasystem_worker.ERROR.<ts>.log` | ERROR —— 迁移失败、`Report rebalance result ultimately failed` 等 |

---

## 三、选"空闲 worker"的迁移策略

### 机制 A（master 调度的 rebalance）

**源（把数据迁出去的 worker）要同时满足**（memory_rebalance_scheduler.cpp:501-510）：

- 状态正常（`isReady` + 拓扑中 ACTIVE），`memoryLimit > 0`
- 内存用量率 ≥ 70%（`rebalance_source_usage_percent`）
- 当前**没有正在执行的迁出任务**
- 当前**没有正在接收别家迁过来的数据**（怕两头一起动，且它上报的内存还没包含刚收进来的数据，数据不准）

**目标（接收数据的 worker）候选**：任何状态正常、不在冷却期的 worker。

**配对规则**（memory_rebalance_scheduler.cpp:552-554）：

- 源和目标**用量差 ≥ 20%** 才配对（`rebalance_usage_gap_percent`），差得不够多就不迁
- 一次迁多少 = 取三者最小（L631-640）：
    - 源和目标用量差的一半（目标是把两人"拉平"到中间点）
    - 目标还剩下的可接收空间（**扣掉已经派给它、但还没迁移完成的数据量**）
    - 1GB 上限

**排序选优**（L602-616）：满足条件的配对里，依次按——

1. 用量差**最大**的优先（差距最大的先迁）
2. 迁完后目标预计用量**最小**的优先（别把目标撑高）
3. 目标剩余可接收空间**最大**的优先
4. nodeId 兜底

即：**优先挑"差距最大、又不会把目标撑爆"的那一对**。

### 机制 B（worker 侧写路径选节点）

`NodeSelector` 拿着一张"所有 worker 剩余可接收空间"的排名表（按剩余空间从大到小，状态正常的排前面），选目标时（node_selector.cpp:111-166）：

- 从排名里找**剩余空间 > 要迁的数据量**的前 5 个，**随机抽一个**
- 如果所有 worker 剩余空间都不够装，就退而选**剩余空间最大的那个**，先试再说（装不下再报空间不足）

---

## 四、能否指定某几个 worker 优先迁移

**结论：接口留了钩子，但当前不可配，生产路径上没生效。**

- `SelectionStrategy::SelectNode(originAddr, preferNode, ...)` 和 `NodeSelector::SelectNode(..., preferNode, ...)` 都有 `preferNode` 参数，`ScaleDownNodeSelector` 也明确优先用它（scale_down_node_selector.cpp:104-111），`NodeSelector` 中 prefer 节点只要 ready 且可用内存够就优先选中（node_selector.cpp:128-133）。
- 但全代码库唯一调用点传的是空串：`strategy->SelectNode(selectionOrigin, "", totalSize, nextWorker)`（data_migrator.cpp:602）。**没有任何配置项/flag 能指定优先 worker**（grep 过 flags 和配置，没有 prefer 相关项）。
- master 调度器**完全没有"优先目标"概念**，纯按用量率。

---

## 五、如果所有 worker 都满了，怎么选目标

- **机制 A（master rebalance）：干脆不迁。** 一旦没有任何配对满足"用量差 ≥ 20%"，`CollectCandidatePairsLocked` 会把所有对过滤掉，`TryBuildTaskLocked` 返回 `K_NOT_FOUND`，调度器不发任务（memory_rebalance_scheduler.cpp:552-554）。rebalance 只是均衡器，不是 OOM 逃生通道——大家都很满时它什么都不做，源 worker 只能靠 eviction / spill 到盘来释放。
- **机制 B（NodeSelector 写路径）：尽力而为。**
    - 没有 worker 有足够空间，但存在 ready 节点 → 回退选**可用内存最大的那个**当 `backupNode`（node_selector.cpp:146-159），写进去再失败则返回 `K_NO_SPACE`。
    - 全部可用 ≤ 1MB → 直接 `K_NO_SPACE`。
    - `rankList_` 为空 → 走拓扑环 `FindNextCommittedMember` 轮询取下一个节点（node_selector.cpp:168-191）。
