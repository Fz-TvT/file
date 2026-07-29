不同vllm-ascend差异
# v0.18.0 — 基线版本

## Key 格式
{model_name}@pcp{pcp}@dcp{dcp}@head_or_tp_rank:{rank}@pp_rank:{rank}@{chunk_hash}

# v0.20.2rc1 — 重大重构版本（Breaking Change）

## Key 格式（新增了 3 个字段）
{model_name}@pcp{pcp}@dcp{dcp}@head_or_tp_rank:{rank}@pp_rank:{rank}
@group:{group_id}@cache_role:{role}@cache_family:{family}@{chunk_hash}

# HEAD（v0.20+，对应未来的 v0.23.x）— 增量优化

## 同 v0.20.2 格式，但 chunk_hash 生成逻辑变了
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


# Key 生成流程对比
## v0.18.0 — 简单流程

process_tokens(token_len, block_hashes, mask_num)
  → 遍历 block_hashes
    → 对每个 hash 调用 _make_key_by_hash(chunk_hash)
      → PoolKey(KeyMetadata(model_name, pcp, dcp, head_or_tp, pp), chunk_hash)
        → to_string()
process_tokens() 只接受 token_len, block_hashes, mask_num 三个参数
无 kv_cache_group_id、cache_role、cache_family 参数
无 rehashing（get_block_hashes() 不存在）
无 grouped hash
无多 group 支持
## v0.20.2 — 引入多组 + 压缩感知 + 重哈希

process_tokens(token_len, block_hashes, mask_num,
               kv_cache_group_id, cache_role, cache_family)
  → 获取 group_block_size
  → 根据 cache_family_ratio 缩放 block_size
  → 调用 get_block_hashes() 重哈希（如果 group_block_size > hash_block_size）
    → 域分隔 SHA-256: "vllm-ascend-grouped-block-hash-v1\0" + framed encoding
  → 遍历重哈希后的 block_hashes
    → 调用 _make_key_by_hash(hash_val, group_id, role, family)
      → PoolKey(KeyMetadata(model_name, ranks..., group_id, role, family), chunk_hash)
        → to_string()
新增路径：

process_tokens_with_block_ids() — 带 block_id 的版本
_generate_store_query_keys() — 调度器查询时枚举所有 ranks 组合
_get_layerwise_gva_hit_tokens() — GVA 路径
##  v0.23.0 — 新增快速路径 + 惰性重哈希

### 路径 A（旧路径保留）：
process_tokens() → _iter_token_chunks() → PoolKey() → to_string()
     与 v0.20.2 逻辑相同，只是重构到 _iter_token_chunks()

### 路径 B（新增快速路径，跳过 PoolKey 分配）：
process_token_key_strings()
  → _get_key_prefix()     # 前缀已缓存: "model_name@pcp0@dcp0@head_or_tp_rank:0@..."
  → _iter_token_chunks()  # 返回 (start, end, hash_val, block_id)
  → 直接拼接: prefix + block_hash_to_str(hash_val)  ← 无 PoolKey 对象

process_token_key_strings_with_block_ids()
  → 同上，额外带 block_id 和分片