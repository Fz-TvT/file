# Yuanrong 后端部署指导（managed / external 两种配置方式）

> 面向场景：PD 分离推理（Prefill / Decode 分机部署）接入 Yuanrong 分布式 KV 池化。
> 本文同时覆盖两种 Worker 部署方式：**managed**（NodeManager 自动拉起）与 **external**（Worker 由你单独部署）。

---

## 目录

- [1. 两种方式对比](#1-两种方式对比)
- [2. 前置条件](#2-前置条件)
- [3. 方式一：managed（NodeManager 自动拉起 Worker）](#3-方式一managednodeManager-自动拉起-worker)
- [4. 方式二：external（单独部署 Worker）](#4-方式二external单独部署-worker)
- [5. P/D 引擎配置（两种方式相同）](#5-pd-引擎配置两种方式相同)
- [6. 部署步骤](#6-部署步骤)
- [7. 验证方法](#7-验证方法)
- [8. 常见问题](#8-常见问题)

---

## 1. 两种方式对比

| 维度 | managed（默认） | external（单独部署） |
|------|-----------------|---------------------|
| Worker 由谁启动 | NodeManager 自动拉起（对标 MemCache LocalService） | **你**手动启动（systemd / 容器 / 脚本） |
| Worker 由谁监控 | NodeManager health_check 自动重启（退避 10s×2ⁿ，上限 300s） | **你**的运维体系 |
| 配置项 | `etcd_address`（必填）、`dram_size` / `shared_memory_size_mb`、`worker_port` | `worker_address_map`（每 P/D 节点一条） |
| deploy.py 是否创建中央 kv_store | 不创建（ETCD 外部、Worker 每节点一个） | 同左 |
| 适用场景 | 快速上手、随引擎一起部署 | Worker 独立升级 / 独立容量规划 / 已有进程管理体系 |

> 两种方式下，**deploy.py 都不会创建中央 `kv_store` Deployment**，这是 Yuanrong 与 MemCache / Mooncake 的关键区别。

---

## 2. 前置条件

### 2.1 部署 ETCD

- 至少 1 个 ETCD 端点，集群内所有节点可访问（生产建议 3 节点高可用）。
- 参考 [主备倒换特性](../../fault_tolerance/standby.md) 中的 ETCD 部署说明。

### 2.2 安装 yuanrong 安装包

在镜像 / 节点上安装（保证容器内可执行 `dscli` 或 `dsc1`）：

```bash
# <arch> 为 aarch64 或 x86_64，例如 manylinux_2_35_aarch64
pip install /home/yuanrong/openyuanrong_datasystem-0.8.2-cp311-cp311-manylinux_2_35_<arch>.whl
```

- 下载地址见 [Yuanrong releases](https://gitcode.com/openeuler/yuanrong-datasystem/releases)
- **版本要求**：需支持 `--liveness_check_path` 参数（0.8.2 及配套版本已支持）。不支持时 Worker 会因未知参数立即退出，请更换版本。

---

## 3. 方式一：managed（NodeManager 自动拉起 Worker）

### 3.1 配置 `kv_cache_store_config`

在 `user_config.json` 的全局配置中新增：

```json
"kv_cache_store_config": {
  "backend": "yuanrong",
  "etcd_address": "<ETCD_IP>:2379",
  "dram_size": "50GB",
  "worker_port": 18481
}
```

参数说明：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `backend` | string | —（必填） | 固定为 `yuanrong`，需与引擎 `AscendStoreConnector` 中的 `backend` 一致 |
| `etcd_address` | string | 无（**必填**） | ETCD 地址，格式 `host:2379`；也可用环境变量 `ETCD_ADDRESS` / `ETCD_HOST` 注入 |
| `dram_size` | string | `"50GB"` | 每节点 Worker 共享内存大小；也支持 `1024MB` 写法 |
| `shared_memory_size_mb` | int | `51200` | 若同时配置 `dram_size`，以本字段为准（优先级更高） |
| `worker_port` | int | `18481` | 本节点 Worker 监听端口，NodeManager 按 `POD_IP:worker_port` 推导 `DS_WORKER_ADDR` |

> **不需要**配置 `worker_mode`（默认就是 `managed`）、`worker_address_map`。

### 3.2 部署后会发生什么

1. NodeManager 发现 `KV_STORE_BACKEND=yuanrong`，加载 `YuanrongService`。
2. 引擎拉起前的 `prepare()` 阶段，自动执行 `dscli`/`dsc1 start` 启动本节点 Worker，并做 30s 端口就绪检测。
3. Worker 异常退出时，NodeManager 按 `MOTOR_RESTART_LOCAL_SERVICE` 策略自动重启（指数退避：`10s × 2ⁿ⁻¹`，封顶 300s）。
4. 每个 P/D 节点各有一个 Worker，全部注册到同一个 ETCD，组成分布式 KV 池。

---

## 4. 方式二：external（单独部署 Worker）

### 4.1 每台 P/D 节点手动启动 Worker

在**每台**要跑 P/D 引擎的机器上执行（4 机场景即执行 4 次，替换变量）：

```bash
# ====== 变量定义（每台机器按实际替换）=====
HOST_IP="<本机IP>"
WORKER_PORT=18481
ETCD="<ETCD_IP>:2379"
SHM_MB=51200            # 共享内存大小，对应 dram_size 50GB
WORK_DIR="/root/yuanrong/worker"
# ========================================

mkdir -p "$WORK_DIR"

# dscli 风格（docs 默认）：
dscli start \
  -d "$WORK_DIR" \
  -w \
  --v 1 \
  --log_only_write_info_file false \
  --kv_events_config "{\"bind_endpoint\":\"tcp://0.0.0.0:7558\",\"backend_id\":\"${HOST_IP}:${WORKER_PORT}\"}" \
  --worker_address "${HOST_IP}:${WORKER_PORT}" \
  --etcd_address "$ETCD" \
  --shared_memory_size_mb "$SHM_MB" \
  --node_timeout_s 30 \
  --node_dead_timeout_s 60 \
  --liveness_check_path /workspace/liveness
```

> 若系统中命令是 `dsc1`（ModelArts 风格），用：
> ```bash
> dsc1 start -t 600 -w \
>   --worker_address "${HOST_IP}:${WORKER_PORT}" \
>   --etcd_address "$ETCD" \
>   --shared_memory_size_mb "$SHM_MB" \
>   --node_timeout_s 30 \
>   --node_dead_timeout_s 60 \
>   --liveness_check_path /workspace/liveness
> ```

> ⚠️ `dscli`/`dsc1` 前台阻塞运行，退出即停止。生产建议用 systemd / 容器托管（见 4.3）。

### 4.2 配置 `kv_cache_store_config`

```json
"kv_cache_store_config": {
  "backend": "yuanrong",
  "worker_mode": "external",
  "worker_address_map": {
    "10.0.0.1": "10.0.0.1:18481",
    "10.0.0.2": "10.0.0.2:18481",
    "10.0.0.3": "10.0.0.3:18481",
    "10.0.0.4": "10.0.0.4:18481"
  }
}
```

要点：

- **`worker_mode: "external"`** — 通知 NodeManager 不启动、不监控 Worker，只解析地址。
- **`worker_address_map`** — 每台 P/D 节点一条。键建议用**节点 IP**（NodeManager 按 `NODE_NAME` → `HOST_IP` → `POD_IP` → hostname 顺序查表，用 IP 做键时 `HOST_IP`/`POD_IP` 都能命中）；值用 `host:port`，缺省端口时补 `worker_port`。
- **不填** `etcd_address`、`dram_size` / `shared_memory_size_mb` — 这些是 Worker 自己的事，NodeManager 不转发也不校验（填了 `etcd_address` 会被忽略并告警）。

> [!WARNING]
> **预置的 `DS_WORKER_ADDR` 会静默压过 `worker_address_map`。** 若启动流程自己 export 了 `DS_WORKER_ADDR`（如 ModelArts 的 `start_motor.sh`），映射表将不被读取且无告警。排查见 [常见问题 §8.7](#87-方式二下-worker_address_map-好像没生效连的还是本机-podip18481)。

### 4.3 生产化：systemd 托管 Worker（可选）

```ini
[Unit]
Description=Yuanrong Worker
After=network.target

[Service]
ExecStart=/usr/local/bin/dscli start \
  -d /var/lib/yuanrong \
  -w \
  --worker_address 10.0.0.1:18481 \
  --etcd_address 10.0.0.100:2379 \
  --shared_memory_size_mb 51200 \
  --node_timeout_s 30
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 5. P/D 引擎配置（两种方式相同）

两种方式下 P/D 引擎配置**完全一致**——区别只在 `kv_cache_store_config` 那一层。

### 5.1 `motor_deploy_config`（节点规模）

```json
"motor_deploy_config": {
  "p_instances_num": 1,
  "d_instances_num": 3,
  "single_p_instance_pod_num": 1,
  "single_d_instance_pod_num": 1,
  "p_pod_npu_num": 8,
  "d_pod_npu_num": 8,
  "image_name": "<你的镜像>",
  "job_id": "mindie-motor",
  "hardware_type": "800I_A3",
  "weight_mount_path": "/mnt/weight/"
}
```

> 4 机 × 8 卡示例：Node1 跑 Prefill（`p_instances_num=1`），Node2~4 跑 Decode（`d_instances_num=3`）。Worker 数量 = 节点数 = 4，各节点一个。

### 5.2 Prefill 引擎（P 节点）— `motor_engine_prefill_config`

```json
"motor_engine_prefill_config": {
  "engine_type": "vllm",
  "motor_nodemanger_config": {},
  "engine_config": {
    "served_model_name": "qwen3-8B",
    "model": "/mnt/weight/qwen3_8B",
    "gpu_memory_utilization": 0.9,
    "data_parallel_size": 1,
    "tensor_parallel_size": 8,
    "pipeline_parallel_size": 1,
    "data_parallel_rpc_port": 9000,
    "enforce-eager": true,
    "max_model_len": 2048,
    "kv_transfer_config": {
      "kv_connector": "MultiConnector",
      "kv_buffer_device": "npu",
      "kv_role": "kv_producer",
      "kv_parallel_size": 1,
      "kv_port": "30001",
      "engine_id": "0",
      "kv_rank": 0,
      "kv_connector_extra_config": {
        "connectors": [
          {
            "kv_connector": "MooncakeHybridConnector",
            "kv_role": "kv_producer",
            "kv_port": "30001",
            "kv_connector_extra_config": {}
          },
          {
            "kv_connector": "AscendStoreConnector",
            "kv_role": "kv_producer",
            "kv_connector_extra_config": {
              "backend": "yuanrong"
            }
          }
        ]
      }
    }
  }
}
```

### 5.3 Decode 引擎（D 节点）— `motor_engine_decode_config`

结构与 Prefill 相同，仅三处差异（见下表）：

```json
"motor_engine_decode_config": {
  "engine_type": "vllm",
  "motor_nodemanger_config": {},
  "engine_config": {
    "served_model_name": "qwen3-8B",
    "model": "/mnt/weight/qwen3_8B",
    "gpu_memory_utilization": 0.9,
    "data_parallel_size": 1,
    "tensor_parallel_size": 8,
    "pipeline_parallel_size": 1,
    "data_parallel_rpc_port": 9000,
    "enforce-eager": true,
    "max_model_len": 2048,
    "kv_transfer_config": {
      "kv_connector": "MultiConnector",
      "kv_buffer_device": "npu",
      "kv_role": "kv_consumer",
      "kv_parallel_size": 1,
      "kv_port": "30002",
      "engine_id": "0",
      "kv_rank": 0,
      "kv_connector_extra_config": {
        "connectors": [
          {
            "kv_connector": "MooncakeHybridConnector",
            "kv_role": "kv_consumer",
            "kv_port": "30002",
            "kv_connector_extra_config": {}
          },
          {
            "kv_connector": "AscendStoreConnector",
            "kv_role": "kv_consumer",
            "kv_connector_extra_config": {
              "backend": "yuanrong"
            }
          }
        ]
      }
    }
  }
}
```

### 5.4 P/D 引擎配置差异速查

| 字段 | P（prefill_config） | D（decode_config） |
|------|---------------------|--------------------|
| 顶层 `kv_role` | `kv_producer` | `kv_consumer` |
| `MooncakeConnectorV1.kv_role` | `kv_producer` | `kv_consumer` |
| `MooncakeConnectorV1.kv_port` | `30001` | `30002` |
| `AscendStoreConnector.backend` | `yuanrong` | `yuanrong`（读写同一个池） |

> `lookup_rpc_port` 无需填写，Motor 按 DP 实例自动适配。P/D 同机共存时 `kv_port` 必须不同。

---

## 6. 部署步骤

```bash
cd examples/deployer
python deploy.py --config_dir ../infer_engines/vllm
```

**managed 模式**内置流程：

1. `deploy.py` → `normalize_kv_cache_store_config` → 将 `dram_size` 换算为 `shared_memory_size_mb`。
2. `gen_kv_store_env` → 注入 `ETCD_ADDRESS`/`ETCD_HOST`、`YUANRONG_WORKER_PORT`、`YUANRONG_SHM_SIZE_MB`、`DS_H2D/D2H_MEMCPY_POLICY=direct`。
3. `setup_yuanrong_env`（`common.sh`）→ 设置客户端日志目录；managed 模式按 `POD_IP:worker_port` 推导 `DS_WORKER_ADDR`。
4. NodeManager `YuanrongService.prepare()` → 自动拉起 Worker 并等待端口就绪。

**external 模式**区别：环境变量只有 `YUANRONG_WORKER_MODE=external` + 两个 memcpy policy；`DS_WORKER_ADDR` 由 NodeManager 从 `worker_address_map` 解析。

---

## 7. 验证方法

### 7.1 external 模式 — 检查生成的引擎 Pod YAML

yuanrong 相关环境变量应**只有**：

```text
YUANRONG_WORKER_MODE=external
DS_H2D_MEMCPY_POLICY=direct
DS_D2H_MEMCPY_POLICY=direct
```

如果还出现 `ETCD_ADDRESS` / `ETCD_HOST` / `YUANRONG_WORKER_PORT` / `YUANRONG_SHM_SIZE_MB`，说明 `worker_mode` 未生效，仍是 managed 模式。

### 7.2 检查 NodeManager 日志

- **managed**：`Yuanrong Worker is ready at <IP>:<port>`
- **external**：`Yuanrong worker_mode=external: client env exported (DS_WORKER_ADDR=<映射值>)`，且**不应出现** `Starting Yuanrong Worker`

### 7.3 检查引擎进程环境变量

```text
DS_WORKER_ADDR=<本机IP>:18481     # external：等于映射值
```

### 7.4 检查 Worker 进程（external）

```bash
ps aux | grep dscli     # 每台机器应有 Worker 进程
ss -tlnp | grep 18481   # 端口在监听
```

---

## 8. 常见问题

### 8.1 Worker 启动失败：找不到 dscli/dsc1

镜像内未安装 `openyuanrong_datasystem` wheel，或 `PATH` 中找不到命令。

### 8.2 Worker 启动失败：缺少 etcd_address

managed 模式未配置 `etcd_address`（或未注入 `ETCD_ADDRESS`/`ETCD_HOST`）。`external` 模式不校验此项。

### 8.3 P/D 无法连接缓存池

- managed：确认每个 P/D 节点 Worker 都已拉起（日志 `Yuanrong Worker is ready`），`DS_WORKER_ADDR` 指向可达地址。
- external：确认映射值正确、Worker 自身在运行。

### 8.4 Worker 启动失败：报未知参数（Unknown flag）

安装的版本不支持 `--liveness_check_path`，请更换 0.8.2 或配套版本。

### 8.5 external 下 NodeManager 报 “requires this node's Worker address”

该节点在 `worker_address_map` 里无匹配项。错误信息会列出已尝试的身份键与可用映射键；补齐对应条目，或注入 `DS_WORKER_ADDR`（优先级最高）。

### 8.6 external 下日志没有 “Starting Yuanrong Worker”

预期行为——external 模式不启动 Worker，只导出 `DS_WORKER_ADDR`。请确认引擎的 `DS_WORKER_ADDR` 等于映射值，并独立确认 Worker 在运行。

### 8.7 external 下 `worker_address_map` 好像没生效，连的还是本机 `POD_IP:18481`

大概率有别的启动流程预置了 `DS_WORKER_ADDR`（如 ModelArts `start_motor.sh` 的 `export DS_WORKER_ADDR="${POD_IP}:18481"`），它静默压过映射表。删掉该行或改成正确地址。

### 8.8 容器部署（docker 路径）

`lib/docker_utils.py` 已支持 yuanrong 后端。部署前确保 ETCD 在容器网络内可达，并给容器传 `--shm-size`（由 `dram_size`/`shared_memory_size_mb` 决定）以满足共享内存需求。

---

## 附：关键默认值（与 NodeManager 一致）

| 项 | 值 |
|----|----|
| Worker 端口 | `18481` |
| 共享内存 | `51200` MB（= 50GB） |
| ETCD 端口 | `2379` |
| 节点超时 | `node_timeout_s=30`、`node_dead_timeout_s=60` |
| KV 事件端口 | `7558` |
| 健康检查路径 | `/workspace/liveness` |
| 重启退避 | `10s × 2ⁿ⁻¹`，封顶 300s |
| 客户端日志目录 | `<日志根>/yuanrong/<ROLE>/client` |