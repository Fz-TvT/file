---
name: kv-cache-block-hash-key
description: KV Cache 前缀缓存中 Block Hash Key 的构成、长度、限制和查看方式
metadata: 
  node_type: memory
  type: reference
  originSessionId: 417e3eeb-28af-45ac-8b41-8ac1fc1f444a
---

# KV Cache Block Hash Key 详解

> vLLM 中用于前缀缓存（Prefix Caching/Prefill Cache）的核心 Key 结构。

---

## 1. Key 的类型体系

定义于 [`vllm/v1/core/kv_cache_utils.py`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py) (第 41-54 行)：

| 类型 | 说明 | 长度 |
|------|------|:----:|
| `BlockHash` | 单个 block 的原始 hash 值（`bytes` 的 NewType） | 16 或 32 bytes |
| `BlockHashWithGroupId` | BlockHash + KV cache group id，供内部缓存使用 | 20 或 36 bytes |
| `ExternalBlockHash` | KV events 使用，可为 `bytes` 或 `int` | 8 bytes（truncated to 64-bit） |

还有同构的 **`OffloadKey`**（[`vllm/v1/kv_offload/base.py`](/Users/fiera/vllm/vllm/v1/kv_offload/base.py) 第 28-48 行）——结构与 `BlockHashWithGroupId` 完全一致，用于磁盘 offload。

---

## 2. Key 的组装方式

### 2.1 `BlockHashWithGroupId` 打包

```python
# kv_cache_utils.py:57-66
def make_block_hash_with_group_id(block_hash: BlockHash, group_id: int) -> BlockHashWithGroupId:
    return BlockHashWithGroupId(block_hash + group_id.to_bytes(4, "big", signed=False))
```

**布局**：`[BlockHash bytes][4 bytes group_id (big-endian)]`

**解包**（第 69-76 行）：
- `get_block_hash(key)` → `key[:-4]` 提取 BlockHash
- `get_group_id(key)` → `int.from_bytes(key[-4:], "big")` 提取 group id

### 2.2 `BlockHash` 的计算链条

核心函数 `hash_block_tokens`（[`kv_cache_utils.py:564-591`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py#L564-L591)）：

```python
hash_input = (parent_block_hash, curr_block_token_ids_tuple, extra_keys)
block_hash = hash_function(hash_input)
```

输入内容：
| 分量 | 说明 |
|------|------|
| `parent_block_hash` | 上一个 block 的 hash（首 block 为 `NONE_HASH`） |
| `curr_block_token_ids` | 当前 block 内的 token id 序列 |
| `extra_keys`（可选）| LoRA name、Multimodal 标识符+offset、Cache salt、Prompt embedding hash |

形成 **Merkle-Damgård 链式哈希**——每个 block 的 hash 依赖其父 block，构成前缀树。

### 2.3 `NONE_HASH` 种子（第 87-114 行）

首 block 的父 hash 种子：

- `PYTHONHASHSEED` **未设置** → `os.urandom(32)`（32 随机 bytes）
- `PYTHONHASHSEED` **已设置** → `hash_fn(hash_seed)`（hash 函数输出）

---

## 3. Hash 算法与 Key 长度

配置于 [`vllm/config/cache.py`](/Users/fiera/vllm/vllm/config/cache.py) 第 38+ 行：

| Hash 算法 | BlockHash 长度 | BlockHashWithGroupId 长度 | 说明 |
|-----------|:-------------:|:------------------------:|------|
| `sha256`（**默认**） | **32 bytes** | **36 bytes** | SHA-256 of pickle 序列化输入 |
| `sha256_cbor` | 32 bytes | 36 bytes | SHA-256 of CBOR（跨语言可复现） |
| `xxhash` | 16 bytes | 20 bytes | xxHash3_128 of pickle 序列化输入 |
| `xxhash_cbor` | 16 bytes | 20 bytes | xxHash3_128 of CBOR（跨语言可复现） |

内存中始终为平坦的 `bytes`，最大 **36 bytes**。

---

## 4. 限制与约束

### 硬限制

**没有硬编码的 key 长度上限。** 长度完全由所选 hash 算法隐式决定。

### 碰撞处理

当两个不同的 block 产生相同的 hash key 时（[`vllm/v1/core/block_pool.py:34-127`](/Users/fiera/vllm/vllm/v1/core/block_pool.py#L34-L127)）：

```python
_cache: dict[BlockHashWithGroupId, KVCacheBlock | dict[int, KVCacheBlock]]
```

自动从单个 `KVCacheBlock` 升级为 `dict[int, KVCacheBlock]` 容纳碰撞。SHA-256 下碰撞概率极低。

### KV Events 截断

当 `VLLM_KV_EVENTS_USE_INT_BLOCK_HASHES` 启用（默认 `True`，[`vllm/envs.py:239`](/Users/fiera/vllm/vllm/v1/kv_offload/file_mapper.py)）：

```python
return int.from_bytes(hash_bytes, byteorder="big") & ((1 << 64) - 1)
```

hash 被截断为 **64 bits（8 bytes）** 用于外部事件系统。

### 磁盘文件路径

（[`vllm/v1/kv_offload/file_mapper.py`](/Users/fiera/vllm/vllm/v1/kv_offload/file_mapper.py)）：

```
<base>_r<rank>/<hhh>/<hh>_g<group_idx>/<hash_hex>.bin
  ├── subfolder1: hash hex 前 3 字符
  ├── subfolder2: hash hex 第 3-5 字符
  └── g<group_idx>: KV cache group index
```

hash hex 长度：SHA-256 → 64 字符，xxHash → 32 字符。

---

## 5. 相关配置参数

| 参数 | 默认值 | 文件 | 说明 |
|------|--------|------|------|
| `block_size` | 16 tokens | `cache.py:46` | 物理 KV cache block 大小 |
| `hash_block_size` | 等于 `block_size` | `cache.py:55` | hash 粒度（可更细，用于混合模型） |
| `mamba_block_size` | 8 的倍数 | `cache.py` | Mamba 层的 block 大小 |
| `VLLM_KV_EVENTS_USE_INT_BLOCK_HASHES` | `True` | `envs.py:239` | 是否将 hash 截断为 64-bit int |

---

## 6. 快速查看入口

| 想看什么 | 文件 | 行号 |
|----------|------|:----:|
| Key 类型定义和组装 | [`vllm/v1/core/kv_cache_utils.py`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py) | 41-76 |
| Block hash 计算逻辑 | [`vllm/v1/core/kv_cache_utils.py`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py) | 564-591 |
| `NONE_HASH` 种子 | [`vllm/v1/core/kv_cache_utils.py`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py) | 87-114 |
| `hash_block_tokens` 中 `extra_keys` 组装 | [`vllm/v1/core/kv_cache_utils.py`](/Users/fiera/vllm/vllm/v1/core/kv_cache_utils.py) | 526-561 |
| Hash 算法选择 | [`vllm/config/cache.py`](/Users/fiera/vllm/vllm/config/cache.py) | 38+ |
| 缓存存储和碰撞处理 | [`vllm/v1/core/block_pool.py`](/Users/fiera/vllm/vllm/v1/core/block_pool.py) | 34-127 |
| OffloadKey（磁盘 offload） | [`vllm/v1/kv_offload/base.py`](/Users/fiera/vllm/vllm/v1/kv_offload/base.py) | 28-48 |
| 磁盘文件路径编码 | [`vllm/v1/kv_offload/file_mapper.py`](/Users/fiera/vllm/vllm/v1/kv_offload/file_mapper.py) | - |


# 数据系统 Key 约束说明

## 合法字符集

KV 和 Object 类 key 只支持以下字符：

```
a-z  A-Z  0-9  -  _  !  @  #  %  ^  *  (  )  +  =  :  ;
```

对应正则表达式（[validator.h](../../../src/datasystem/common/util/validator.h#L61)）：

```
^[a-zA-Z0-9\-_!@#%^*()+=:;]*$
```

## 明确不支持的字符

| 类别 | 不支持字符 |
|------|-----------|
| **英文点号** | `.` |
| **斜杠** | `/` |
| **美元符** | `$` |
| **波浪号** | `~` |
| **与符号** | `&` |
| **问号** | `?` |
| **逗号** | `,` |
| **引号** | `'` `"` `` ` `` |
| **空格/空白** | 所有空白字符（含 ` `、`\t`、`\n` 等） |
| **括号/花括号/方括号** | `<` `>` `{` `}` `[` `]` |
| **管道符/反斜杠** | `\|` `\` |
| **中文/Unicode** | 所有非 ASCII 字符 |
| **空字节/控制字符** | `\0` 及 ASCII 0x00–0x1F |

## 长度与数量限制

| 限制项 | 上限 | 说明 |
|--------|------|------|
| **单个 key 最大长度** | **255 字节** | `value.size() <= UINT8_MAX`（`UINT8_MAX = 255`），由 `Validator::IsIdFormat()` 强制校验 |
| **单次批量操作最大 key 数** | **10,000 个** | `OBJECT_KEYS_MAX_SIZE_LIMIT`，由 `Validator::IsBatchSizeUnderLimit()` 校验 |
| **空 key** | 允许 | 空字符串通过校验，实际调用视具体接口行为 |

## 校验位置

- **C++ 层**：`Validator::IsIdFormat()` 做实际正则匹配校验（[validator.h](../../../src/datasystem/common/util/validator.h#L586-L607)）
- **Python SDK**：`yr.datasystem.util.Validator` **不做 key 字符校验**，仅做参数类型检查；底层调用 C++ pybind 接口，校验失败抛出 `RuntimeError`
- **设备对象（Device Object）key**：另用 `client_device_object_manager.h` 中的规则，字符集略宽松（额外允许 `.` `~` `$` `&`），长度限制也是 255 字节（`less than 256`）

## 验证示例

```python
from yr.datasystem.hetero_client import HeteroClient

client = HeteroClient("127.0.0.1", 31501)
client.init()

# 合法 key
client.delete(["hello", "test_key", "foo-bar!@#"])

# 非法 key（含不合法字符）=> 抛出 RuntimeError
client.delete(["invalid.key"])     # '.' not allowed
client.delete(["slash/test"])      # '/' not allowed
client.delete(["$money"])          # '$' not allowed
client.delete(["中文key"])         # Unicode not allowed
```
