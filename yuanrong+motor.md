# Yuanrong KV 池化后端部署手册
## 1. 方案概述

Yuanrong 基于 [openYuanrong datasystem](https://gitcode.com/openeuler/yuanrong-datasystem) 提供分布式 KV 池化能力，采用「**外部 ETCD 元数据服务 + 每 P/D 节点一个 Worker**」的架构：

```
                    ┌──────────────────┐
                    │   外部 ETCD 集群  │  ← 需预先部署（Worker 注册/发现/KV 元数据）
                    └────────▲─────────┘
                             │
     ┌───────────────────────┼───────────────────────┐
     │                       │                       │
┌────┴─────┐            ┌────┴─────┐            ┌────┴─────┐
│ P 节点    │            │ D 节点    │            │ P/D 节点  │
│ Engine   │            │ Engine   │            │ Engine   │
│ Worker ← │ NodeManager│ Worker ← │ NodeManager│ Worker ← │ NodeManager
└──────────┘  自动拉起    └──────────┘  自动拉起    └──────────┘
```


Worker 生命周期由 NodeManager 全权托管：

- `prepare()`：引擎拉起前启动 Worker，等待端口就绪（最长 30s）
- `health_check()`：每 5s 端口探测（`POD_IP:worker_port`，1s 超时）；Worker 死亡后自动重启，指数退避（$10s×2^{(n-1)}$，封顶 5min）
- `stop()`：NodeManager 退出时终止 Worker

## 2. 前置条件

1.按照MindIE-Motor完成 镜像、k8s等使用Motor的前置配置
2. 安装etcd且 P/D 节点网络可达。
```json
# 以 Linux amd64 为例安装etcd
ETCD_VER=v3.5.0
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xvf etcd-${ETCD_VER}-linux-amd64.tar.gz
cd etcd-${ETCD_VER}-linux-amd64
sudo cp etcd etcdctl /usr/local/bin/
```

3. **已安装 yuanrong 安装包**（提供 `dscli` 或 `dsc1` 命令行）。

## 3. 部署步骤

### 3.1 安装 yuanrong 安装包（制作镜像时）

在 P/D 节点motor镜像内安装 `openyuanrong_datasystem` wheel，保证motor镜像内可执行 `dscli` 或 `dsc1`：

```bash
# <arch> 为 aarch64（ARM64）或 x86_64，如 manylinux_2_35_aarch64
pip install /workspace/openyuanrong_datasystem-0.8.2-cp311-cp311-manylinux_2_35_<arch>.whl --force-reinstall -v
```

[下载地址](https://gitcode.com/openeuler/yuanrong-datasystem/releases)


### 3.2 修改 user_config.json

#### （1）P 实例的 kv_transfer_config

`AscendStoreConnector` 的 `backend` 改为 `"yuanrong"`（P/D 两处都要改）：

```json
"motor_engine_prefill_config": {
  "engine_type": "vllm",
  "engine_config": {
    "kv_transfer_config": {
      "kv_connector": "MultiConnector",
      "kv_role": "kv_producer",
      "kv_connector_extra_config": {
        "connectors": [
          {
            "kv_connector": "MooncakeConnectorV1",
            "kv_role": "kv_producer",
            "kv_port": "30001"
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
D 实例的 kv_transfer_config
```json
"motor_engine_decode_config": {
  "engine_type": "vllm",
  "engine_config": {
    "kv_transfer_config": {
      "kv_connector": "MultiConnector",
      "kv_role": "kv_consumer",
      "kv_connector_extra_config": {
        "connectors": [
          {
            "kv_connector": "MooncakeConnectorV1",
            "kv_role": "kv_consumer",
            "kv_port": "30002"
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
> 标准 attention 模型用 `MooncakeConnectorV1`；DeepSeek V4 等混合 attention 模型用 `MooncakeHybridConnector`，选型规则与 MemCache/Mooncake 一致。

#### （2）全局 kv_cache_store_config

```json
"kv_cache_store_config": {
  "backend": "yuanrong",
  "worker_port": 18481,
  "worker_args": [
    "dscli", "start", "-w",
    "--worker_address", "{worker_address}",
    "--etcd_address", "10.0.0.100:2379",
    "--shared_memory_size_mb", "51200",
    "--node_timeout_s", "30",
    "--node_dead_timeout_s", "60",
  ]
}
```
#### （3）worker_args 详解

`worker_args` 是 Worker 启动命令的**来源**,把普通的worker启动命令按照下面字符加逗号的方式修改然后填入即可

- **可省略项**：`--worker_address` 可以不写——dscli/dsc1 会自动读取环境变量 `DS_WORKER_ADDR` / `ETCD_ADDRESS`（deployer 已注入）。建议不写，会自动读取pod的IP地址，否则worker ip需要跟真实pod ip对应。
- **dsc1 风格**：
  ```json
  "worker_args": [
    "dsc1", "start", "-t", "600", "-w",
    "--worker_address", "{worker_address}",
    "--etcd_address", "10.0.0.100:2379",
    "--cluster_name", "etcd-cluster-1",
    "--shared_memory_size_mb", "51200",
    "--node_timeout_s", "300",
    "--node_dead_timeout_s", "600",
  ]
  ```

### 3.3 执行部署

在 k8s 管理节点执行（按照Motor服务启动方式启动即可）：

```bash
cd examples/deployer
python deploy.py --config_dir ../infer_engines/vllm
```
