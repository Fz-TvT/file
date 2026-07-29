不同vllm-ascend差异
v0.18.0 — 基线版本

# Key 格式
{model_name}@pcp{pcp}@dcp{dcp}@head_or_tp_rank:{rank}@pp_rank:{rank}@{chunk_hash}

v0.20.2rc1 — 重大重构版本（Breaking Change）

# Key 格式（新增了 3 个字段）
{model_name}@pcp{pcp}@dcp{dcp}@head_or_tp_rank:{rank}@pp_rank:{rank}
@group:{group_id}@cache_role:{role}@cache_family:{family}@{chunk_hash}

HEAD（v0.20+，对应未来的 v0.23.x）— 增量优化

# 同 v0.20.2 格式，但 chunk_hash 生成逻辑变了
关键增量变化：

| 变更 | 版本 | 影响 |
|------|------|------|
| Grouped hash rehashing（commit 8991ca5f） | v0.20.2→HEAD | 当 group_block_size > hash_block_size 时，chunk_hash 改为域分隔重哈希 "vllm-ascend-grouped-block-hash-v1\0" → chunk_hash 变了 |
| Grouped hash lookup 编码对齐（commit 7d28ab4e） | HEAD | 修复调度器（hex 字符串）和工作端（raw bytes）hash 不一致 → chunk_hash 稳定性修复 |
| Layerwise KV Pooling（commit 5e390744） | HEAD | Layer 级 key 的数据结构 |
| pool_scheduler.py 独立生成 key | HEAD | generate_keys() 产生 {model_name}@{chunk_hash}（GVA 路径）；_generate_store_query_keys() 产生 PoolKey |
| GVA layerwise 路径 | HEAD | 基于 GVA 地址而非 key 字符串的传输 |
| Mamba groups | HEAD | group_uses_align_state 影响 head_or_tp_rank 取值 |
| LookupKeyClient/LookupKeyServer | HEAD | ZMQ RPC 独立查找 |


| 字段 | DeepSeek V4 Flash | DeepSeek V4 Pro | GLM 5.1 | Qwen3-8B |
|------|-------------------|-----------------|---------|-----------|
| model_type | deepseek_v4 | deepseek_v4 | 非 deepseek_v4（通常是 glm） | qwen2 等 |
| model_name | DeepSeek-V4-Flash | DeepSeek-V4-Pro | glm-5-1 / glm5-1b 等 | Qwen3-8B |
| compress_ratios | ✅ 有（如 [1,4,4,...]） | ✅ 有（与 Flash 不同） | ✅ 可能有 | ❌ 没有 |
| cache_family | c1, c4, c128… | c1, c4, c128… | 取决于 compress_ratios | default |
| compress 路径 | DSV4 特化（get_dsv4_compress_ratio） | DSV4 特化 | 通用（extract_layer_index） | 无压缩 |
| use_mla | ✅ True | ✅ True | ❌/✅ 取决于版本 | ❌ False |
| num_kv_head | 1（MLA 特性） | 1 | 视配置（通常 >1） | 8（标准 GQA） |
| put_step | tp_size // 1 = 最大 | 同 Flash | 取决于 num_kv_head | tp_size // 8 = 较小 |
| head_or_tp_rank | tp_rank // tp_size → 只有 0 | 同 Flash | 取决于 kv_heads | tp_rank // (tp_size//8) → 范围更小但 >1 |
| cache_role | kv + state（两套 key） | kv + state | 可能只有 kv | 只有 kv |
| group_id 数量 | 可能多个（hybrid 压缩组） | 可能多个 | 可能多个 | 只有 group 0 |
| key 总长度 | 最长（family 字段 + state role + layer_id） | 同 Flash | 中等 | 最短 |


Poolkey差异
| # | 字段 | 含义 | 四个模型之间有差异？ |
|---|------|------|---------------------|
| 1 | model_name | 模型名（路径最后一段） | 不同 — 各模型名称不同 |
| 2 | pcp_rank | 预填充上下文并行 rank | 相同 — 取决于部署配置，与模型无关 |
| 3 | dcp_rank | 解码上下文并行 rank | 相同 — 取决于部署配置，与模型无关 |
| 4 | head_or_tp_rank | KV head 或 TP rank 的分片 | 不同 — DS V4（MLA → 始终 0）；其他模型取决于 num_kv_head |
| 5 | pp_rank | 流水线并行 rank | 相同 — 取决于部署配置，与模型无关 |
| 6 | group | KV cache 组 ID | 不同 — DS V4 可能有多组（0,1,2…）；Qwen3-8B 固定 0 |
| 7 | cache_role | kv 或 state | 不同 — DS V4 有 kv + state；其他只有 kv |
| 8 | cache_family | 压缩家族 | 不同 — DS V4 是 c1/c4/c128；Qwen3-8B 是 default |
| 9 | chunk_hash | Token block 的内容哈希 | 间接不同 — 压缩比不同导致 group_block_size 不同，影响重哈希结果，但同一模型同一内容在相同版本下一致 |