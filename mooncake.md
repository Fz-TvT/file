# 远端 CPU → 本地 GPU 的流程与注册优化

### 一、完整流程

以 **本端（Node A）** 的视角，需要把远端（Node B）CPU 内存的数据拉到本地 GPU 显存：

`┌─────────────────────┐                    ┌──────────────────────┐
│   Node A (本端)      │                    │   Node B (远端)       │
│                      │                    │                      │
│ ① 注册 GPU 显存      │                    │ ③ 远端已注册 CPU 内存  │
│    ibv_reg_mr(gpu)   │                    │    ibv_reg_mr(cpu)   │
│    → 得到 local_rkey │                    │    → 得到 remote_rkey│
│                      │                    │                      │
│ ② openSegment(B)     │── 读 metadata ──→  │ ④ 发布 SegmentDesc   │
│    → 拿到 remote_rkey│                    │    (cpu addr, rkey)  │
│                      │                    │                      │
│ ⑤ submitTransfer(    │                    │                      │
│    OPCODE_READ,      │── RDMA READ ─────→ │  远程 CPU 内存       │
│    dst=local_gpu,    │                    │                      │
│    src=remote_cpu)   │◀── 数据流 ─────── │                      │
│                      │                    │                      │
│ ⑥ 本地 GPU 已收到数据 │                    │                      │
└─────────────────────┘                    └──────────────────────┘`

**关键点**：

- 本端主动发起 **RDMA READ**，从远端 CPU 内存拉到本地 GPU 显存
- 远端只负责注册内存、发布 rkey，不参与数据传输的控制
- GPU 显存要用 RDMA 访问，必须事先在本地 RDMA NIC 上注册（`ibv_reg_mr`）

---

### 二、注册开销来源

在这个场景中，**本端 GPU 内存的注册**是主要开销来源，分两个路径：

| 条件 | 注册方式 | 额外开销 |
| --- | --- | --- |
| **nvidia-peermem 已加载**（默认、TENT 唯一路径） | `ibv_reg_mr(pd, gpu_addr, len, access)` | ≈ CPU 注册，略有 GPU TLB 同步 |
| **nvidia-peermem 未加载**（经典引擎 fallback） | `cuMemGetAddressRange` + `cuMemGetHandleForAddressRange(..DMA_BUF..)` + `ibv_reg_dmabuf_mr()` | ❌ 多 2 次 CUDA API + dmabuf fd 生命周期管理 |

---

### 三、缓解注册开销的机制

针对 **GPU 内存注册**，Mooncake 使用以下缓解手段：

### 1. 引用计数 — 复用已注册 GPU MR

代码在 segment_tracker.cpp:48

这是对**频繁注册**最直接的优化。同一个 GPU 地址+长度被多次调用 `registerLocalMemory` 时，只第一次真正调 `ibv_reg_mr`，后续只递增引用计数。注销同理，减到零才调 `ibv_dereg_mr`。

`第一次调用 registerLocalMemory(gpu_addr, size)  → ref=1, ibv_reg_mr
第二次调用 registerLocalMemory(gpu_addr, size)  → ref=2, 跳过注册
unregister(gpu_addr, size)                      → ref=1, 跳过注销
unregister(gpu_addr, size)                      → ref=0, ibv_dereg_mr`

这在 PyTorch 的缓存分配器场景下尤其关键——同一个 GPU buffer 可能被多个模块反复引用。

### 2. 跨 NIC 并行注册

代码在 buffers.cpp:114

GPU 内存需要在每个 RDMA NIC 上各注册一个 MR。使用 `std::async` 并发完成：

`for (auto& ctx : rdma_contexts) {
    futs.push_back(std::async(std::launch::async, [&]() {
        return ctx->registerMemReg(addr, length, access_flags);
    }));
}`

### 3. Batch 批量注册（一次 metadata 同步）

代码在 transfer_engine_impl.cpp:651

多个 GPU buffer 批量注册时，所有 buffer 打包成一次 metadata server 同步 → 一次 RTT 替代 N 次。

### 4. CUDA Primary Context 保活

代码在 context.cpp:538

注册 GPU 内存前，确保 CUDA primary context 已被激活并设为 current。避免了 `ibv_reg_mr` 因 context 未初始化而触发额外的延迟：

`if (is_device_memory) {
    cuDeviceGet(&dev, device_ordinal);
    cuDevicePrimaryCtxRetain(&ctx, dev);
    cuCtxSetCurrent(ctx);              // 保证 context 在线
}
mr = verbs_.ibv_reg_mr_default(pd_, addr, length, access);
cuDevicePrimaryCtxRelease(dev);`

### 5. GPU 内存不做 Warm-up——但有替代思路

GPU 内存**不执行** MR Warm-up（context.cpp:581 只对 CPU buffer ≥4GiB 生效）。

原因是：nvidia-peermem 在 `ibv_reg_mr` 时会隐式处理页表固定，额外的 reg/dereg 循环收益有限。且 GPU 内存属于稀缺资源，临时注册再注销的开销可能得不偿失。

---

### 四、总结表

| 优化机制 | 效果 | 适用阶段 |
| --- | --- | --- |
| **引用计数** | 消除同 buffer 重复注册，最直接的优化 | register 调用时 |
| **并行注册** | 多 NIC 注册时间 O(N)→O(1) | register 执行时 |
| **Batch 注册** | 多个 buffer 共享 1 次 metadata 同步 | register 批量时 |
| **CUDA Context 保活** | 避免 context 切换额外延迟 | register 准备时 |
| **DMA-BUF fallback** | 无 peermem 时仍可用 GPU RDMA（但更贵） | 环境适配 |

这里的关键设计思路是：**不试图让单次 GPU 注册变快（它确实比 CPU 注册慢），而是通过引用计数 + 并行 + 批量来稀释或避免重复注册**，让实际生效的 `ibv_reg_mr` 调用次数最小化。