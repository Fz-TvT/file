# YuanRong 后端接入设计说明（修订版）

> 本文档描述 **YuanRong KV 缓存后端接入 MindIE Motor 的方案**。v2 版基于对 yuanrong 侧仓库的事实核查做了修正：`dsc1`、前台 `poll()` 守护、`hostname -I`、per-DP 端口、prometheus `/metrics` 等假设均被推翻。文档以 MemCache 后端为参照讲清"后端如何接入"，再给出 YuanRong 的接入方案与 NodeManager 进程守护设计。

---

## 1. 背景：KV 池化与后端抽象

MindIE Motor 支持 PD 分离架构下的 **KV Cache 池化**：Prefill（P）实例把算好的 KV 块写入共享池，Decode（D）实例从池中拉取。底层存储后端可替换，已有三种：

| 后端 | 池主控 | 节点存储进程 | 引擎侧连接 |
|------|--------|--------------|-----------|
| **Mooncake** | `mooncake_master`（独立 Pod） | — | MooncakeConnectorV1 |
| **MemCache** | MetaService（独立 Pod） | LocalService（NodeManager 管理） | AscendStoreConnector（`backend=memcache`） |
| **YuanRong** | **etcd（外部依赖）** | **`datasystem_worker`（由 `dscli` 拉起）** | **YuanRongConnector**（vLLM `--kv-transfer-config` 配置） |

后端接入链路四块：**配置层 → deployer（K8s 生成）→ 节点生命周期（NodeManager）→ 指标采集**；另有 kv-conductor（Rust）与调度器 KV 亲和链路按后端模式分发。

---

## 2. MemCache 后端接入全解析（参照）

MemCache 是接入最完整的后端，是新增后端的参照模板。注意非开源 memcached，而是昇腾自研组件（`memcache_hybrid` 包），提供 HBM → DRAM → SSD 三层缓存。

### 2.1 用户配置（两处）

引擎侧（vLLM 用 memcache 做 KV 池读写）：

```json
"kv_transfer_config": {
  "kv_connector": "MultiConnector",
  "kv_connector_extra_config": { "connectors": [
    { "kv_connector": "MooncakeConnectorV1", "kv_role": "kv_producer", "kv_port": "30001" },
    { "kv_connector": "AscendStoreConnector", "kv_role": "kv_producer",
      "kv_connector_extra_config": { "backend": "memcache" } }
  ]}
}
```

顶层（启动 KV 存储服务）：

```json
"kv_cache_store_config": {
  "backend": "memcache",
  "local_service_mode": "standalone",
  "target_job_id": "service-a"
}
```

### 2.2 部署与启动

- **deploy.py**：`update_kv_store_enabled_flag()` 探测启用 → `normalize_kv_cache_store_config()`（`backend` 默认 `"memcache"`）→ `generate_yaml_kv_store()` 生成 KV Store Pod（Service `mindie-motor-kvs-master` 50088/50089/50090）→ `create_motor_config_configmap()` 打包 4 个文件（2 脚本 + 2 `.conf`）→ 引擎 Pod 注入 `KVS_MASTER_SERVICE`/`KV_STORE_BACKEND`/`MMC_LOCAL_CONFIG_PATH` 等。
- **KV Store Pod**：`boot.sh → kv_cache_store.sh`（按 `KV_STORE_BACKEND` 分发）→ `memcache.sh` → `memcache_meta_service.py` 调 `MetaService.setup()/main()`，用 `POD_IP` + 端口组 URL 成为池主控。
- **引擎 Pod**：`engine.sh` 调 `sync_mmc_local_config()`（ConfigMap .conf 复制为真实文件、`$KVS_MASTER_SERVICE` 替换 DNS、导出 `MMC_LOCAL_CONFIG_PATH`）；随后 `Daemon` 启动，`services="engine,memcache"` → `registry.discover()` 导入 lifecycle → `@register_service("kv_store", backend="memcache")` 注册 `LocalService`。

### 2.3 运行时数据流

```
P 实例算完 KV → AscendStoreConnector 写入本地 DRAM 池
  → DistributedObjectStore 与 MetaService(tcp://...:50088) 同步元数据
D 实例 → 查 MetaService 定位块 → 按协议从 P 节点拉取 → 跳过重算
```

### 2.4 指标

`metrics_collector.py` 的 `_filter_kvstore_metrics(raw, backend)` 按 backend 分发，memcache 走 `_filter_memcache_metrics()`：重命名 `memcache_* → motor:memcache_*` 并聚合 `kv_store_size/ratio/keys/eviction` 四族。

---

## 3. YuanRong 侧事实核查（已确认，附证据）

> 证据路径均为 yuanrong 仓库（外部），供核对。

### 3.1 CLI 与进程模型

| 事实 | 结论 | 证据 |
|------|------|------|
| CLI 名 | `dscli`（Python，pip 安装），**非 `dsc1`**；worker 二进制为 `datasystem_worker` | cli/start.py:777 |
| `dscli start` 是否前台阻塞 | **否**。`ready 即返回`：`start_worker` 用 `subprocess.Popen(..., start_new_session=True)` 拉起 `datasystem_worker`，循环 poll 直到 `ready_check_path` 文件出现（worker 初始化完成后写出），随即 `run()` 返回 SUCCESS、dscli 进程退出 | cli/start.py:542、cli/start.py:210；worker_oc_server.cpp:2732-2741 |
| worker 是否 daemonize | **否**。直跑 `datasystem_worker` 二进制即前台阻塞，`poll()` 有效 | worker_oc_server.cpp:1970-1975 |
| 存活信号 | worker 周期性写 liveness 文件（"liveness check success"，经 `liveness_check.sh`）；K8s 侧默认就是 exec 文件检查探针 | worker_oc_server.cpp:1970-1975；worker_daemonset.yaml |
| 重启 | 无 `dscli restart`，`restart = stop + start` | vllm_yuanrong_cli.sh:226 |
| 锁残留 | 前一个 worker 可能短暂持有 metadata store 锁，dscli 识别并自动重试启动 | cli/start.py:563 `is_retryable_worker_store_lock_error` |

**推论**：`dscli start` 返回后拿到的 PID 是 dscli 自己（已退出），对 launcher PID 做 `poll()` 守护**必然失效**。守护必须基于文件探活，或绕过 dscli 直跑二进制（此时 `poll()` 有效）。两条路二选一（见 §4 拍板项 1）。

### 3.2 本机 IP

| 事实 | 结论 | 证据 |
|------|------|------|
| IP 发现 | yuanrong 侧**无自动发现机制**，IP 由部署方显式提供；权威来源是 `worker_address`（host:port），写死在 `worker_config.json`（默认 `127.0.0.1:31501`） | worker_config.json |
| 如何改写 | `dscli up` 按集群配置的节点列表 + `WORKER_PORT` 逐节点改写 `worker_address`，**一节点一 worker** | cli/up.py:297-305、cli/up.py:265 |
| 参考部署做法 | `deploy_env.sh` 手填 `HOST_IP`，同一变量同时喂 `dscli --worker_address` 和 vLLM `--host`/`HCCL_IF_IP` | deploy_env.sh |

**推论**：**不要用 `hostname -I`**（多网卡/IPv6/与 RPC 绑定地址不一致都会踩坑）。Motor 从自己发起部署时保存的 `worker_address`/`HOST_IP` 取 IP，与配给 worker 的是同一个值。

### 3.3 etcd 部署

| 事实 | 结论 | 证据 |
|------|------|------|
| etcd 定位 | ETCD 模式下，yuanrong 把 etcd 当**外部依赖**，仓库不提供任何 etcd 清单；「安装并部署 ETCD」是环境前置条件（ETCD 3.5） | dscli.md「环境准备」表 |
| 多端点 | `etcd_address` 支持逗号分隔多端点 | validator.h:223 `ValidateEtcdAddresses` |
| 无 etcd 替代 | 支持 `metastore_address` 模式完全替代 etcd | dscli `up --metastore`，up.py:99 |
| 参考部署 | 参考脚本是**单实例 etcd**（`--name etcd-yuanrong`） | vllm_yuanrong_cli.sh:174-187 |

**推论**：**复用外部 etcd 是最简路径**——`etcd_address` 直接填外部端点，NodeManager 只负责 worker。自部署则需 Motor deployer 自建 StatefulSet（yuanrong 无可复用清单）。

### 3.4 引擎侧连接器

| 事实 | 结论 | 证据 |
|------|------|------|
| connector 名 | vLLM 侧存在可配置的 **`YuanRongConnector`** | deploy_env.sh:60/79/95 |
| 配置方式 | 在 vLLM 启动参数 `--kv-transfer-config` 里声明，含 `"kv_connector":"YuanRongConnector"` 和 `kv_role`（`kv_both`/`kv_producer`/`kv_consumer` 按角色取值） | vllm_yuanrong_cli.sh:213 |

**推论**：connector 名称由 vLLM 启动参数配置，**不经过 Motor**；`kv_transfer_config` 直接直传 vLLM。分工照抄参考部署即可，无新概念。

### 3.5 DP → worker 映射（关键冲突点）

| 事实 | 结论 | 证据 |
|------|------|------|
| 默认拓扑 | **每节点一个 `datasystem_worker`、一个 `worker_address` 端口**（默认 31501，参考脚本 32451），同一 brpc 端口服务该节点所有客户端/DP | cli/up.py:265 |
| 无 per-DP 端口 | 仓库没有 per-DP 端口概念；`oc_worker_worker_direct_port`/`sc_worker_worker_direct_port`/`shared_memory_worker_port` 均非 per-DP | 端口 flags |

**冲突**：Motor 侧 conductor 当前假设 `MatchMode::None` = "subscriber tied to a fixed dp_rank"，即 **per-DP 端口订阅**（[backend.rs:82](motor/kv_conductor/src/backend.rs#L82)）。若按端口映射 DP 会撞车（同节点多 DP 共享一端口）。

**两条出路**：
- **A（推荐）**：Motor 自建「DP → 节点 → worker_address」拓扑（Motor 本就是发起部署的一方，掌握 endpoints/DP 分布），注册时按节点聚合上报、显式带 `dp_rank`；conductor 的 YuanRong 匹配语义需从「端口=DP」改为「按节点聚合 worker + 节点内多 DP」。
- **B**：为每个 DP 各起一个 worker（`dscli start` 支持不同 `--worker_address` 端口并行），端口=DP 成立，但会**重复 etcd heartbeat / location 上报**，非默认形态，开销大。

### 3.6 指标

| 事实 | 结论 | 证据 |
|------|------|------|
| 无 /metrics 端点 | datasystem 自定义指标**没有 Prometheus /metrics 端点**；brpc 内置服务默认关闭 | common_flag_define.cpp:191 `brpc_enable_builtin_services=false` |
| 开启后 | 同一 worker_address 端口暴露 `/health`、`/status`、`/vars`、`/prometheus_metrics`，但仅是 **brpc 层指标**（QPS/延迟/连接数），不含自定义指标 | brpc 内置服务 |
| 自定义指标去向 | 导出到 **`kv_metrics.log`（JSON Lines）** 和 INFO 日志；`log_monitor`/`json_log_monitor` 默认均 true | common_flag_define.cpp:126-127、kv_metrics.cpp |

**推论**：`kv_store_*` 聚合有三种取数方式（需 Motor 确认要哪些指标）：开 brpc 抓 `/prometheus_metrics`（brpc 层）／解析 `kv_metrics.log`（自定义层，默认开）／跳过。

---

## 4. Motor 侧决策（已定案）

> v3 起 4 项拍板全部按推荐方案定案，不再作为待确认项。

| # | 决策项 | 已定方案 | 理由 |
|---|--------|---------|------|
| 1 | **worker 启动与守护方式** | **走 `dscli` + ready/liveness 文件探活** | K8s 侧规范做法就是文件探针；dscli 封装了配置生成/etcd 心跳/锁重试/ready 检查，Motor 不重复造轮子 |
| 2 | **IP 来源** | **部署时保存的 `worker_address`/`HOST_IP`** | 与配给 worker 的值一致，避免多网卡/IPv6 坑；弃用 `hostname -I` |
| 3 | **DP → worker 映射** | **Motor 自建「DP→节点→worker_address」+ conductor 按节点聚合** | 默认拓扑即一节点一 worker，per-DP 起 worker 会重复 heartbeat |
| 4 | **`kv_store_*` 指标来源** | **解析 `kv_metrics.log`（JSON Lines，自定义层）** | 需容量/淘汰等自定义指标；仅 QPS 时可选 brpc，暂不需要可跳过 |

---

## 5. 接入方案（修订）


### 5.1 配置层

`motor/config/node_manager.py` 的 `KVCacheStoreConfig` 增加 YuanRong 专属字段：

```python
# --- YuanRong ---
# dscli 与 datasystem_worker
worker_address: str = ""          # host:port，部署时保存，默认 127.0.0.1:31501
etcd_address: str = ""            # 逗号分隔多端点；或走 metastore_address 模式（二选一）
metastore_address: str = ""
cluster_name: str = ""
worker_config_path: str = ""      # worker_config.json 路径（若 dscli 需）
# 文件探活（拍板项 1 → A）
ready_check_path: str = ""        # dscli start 等待的 ready 文件
liveness_check_path: str = ""     # worker 周期性写的 liveness 文件
liveness_timeout_s: int = 30      # 超过该时长未更新 → 判定僵死
start_timeout_s: int = 120        # dscli start 后等待 ready 的超时
# 重启开关（复用现有语义）
restart_worker: bool = True       # 复用 MOTOR_RESTART_LOCAL_SERVICE
```

`motor/common/utils/env.py` 增加对应读接口（仿 `mmc_*` 写法）。

### 5.2 NodeManager 生命周期与进程守护（本次核心）

新增 `motor/node_manager/core/services/yuanrong/lifecycle.py`。骨架沿用 memcache 的 `@register_service + pull/stop/health_check`，但**守护模型不同**（memcache 是子进程 `poll()`，yuanrong 是文件探活）：

```python
@register_service(SERVICE_KV_STORE, backend="yuanrong",
                  prepare_priority=10, factory=_create_yuanrong_worker)
class YuanRongWorker:
    """拉起并守护 datasystem_worker（经 dscli）。守护基于文件探活。"""

    def should_launch(self) -> bool:
        return self._kv_cfg.enable and self._kv_cfg.backend == "yuanrong"

    def prepare(self, **kwargs) -> None:
        """引擎启动前：校验 dscli/worker_config，记录 worker_address 与探活文件路径。"""
        if not shutil.which("dscli"): raise RuntimeError("dscli not installed")
        # 校验 ready/liveness 探活文件路径可写

    def pull(self) -> None:
        """拉起 worker。dscli start 是 ready 即返回，必须等 ready 文件。"""
        if self.is_alive(): return
        self._clear_ready_file()                 # 防残留旧 ready 文件
        env = os.environ.copy()
        cmd = ["dscli", "start", "--worker_address", self._kv_cfg.worker_address,
               "--etcd_address", self._kv_cfg.etcd_address,      # 或 --metastore
               "--cluster_name", self._kv_cfg.cluster_name, ...]
        subprocess.run(cmd, env=env, timeout=self._kv_cfg.start_timeout_s)
        # dscli 已退出；等 ready_check_path 出现（worker 真正就绪）
        self._wait_for_ready(self._kv_cfg.start_timeout_s)

    def is_alive(self) -> bool:
        """文件探活：liveness 文件新鲜 → 存活。"""
        return _file_mtime_within(self._kv_cfg.liveness_check_path,
                                  self._kv_cfg.liveness_timeout_s)

    def stop(self) -> None:
        """dscli stop 优雅退出；超时兜底（无 restart，restart=stop+start）。"""
        subprocess.run(["dscli", "stop", "--worker_address", self._kv_cfg.worker_address], ...)

    def health_check(self) -> None:      # ← 5s 监视器自动调用
        """liveness 文件超时未更新 → 判定僵死 → stop + start。"""
        if self.is_started() and not self.is_alive():
            logger.warning("yuanrong worker 僵死 (restart=%s)", self.restart_worker)
            self.mark_dead()
            if self.restart_worker:
                self.stop(); self.pull()
```

要点：
- **`pull()` 等 ready 文件**：`dscli start` 返回≠worker 就绪，`ready_check_path` 出现才是（worker_oc_server.cpp:2732-2741）。
- **`health_check()` 查 liveness 文件 mtime**：worker 周期性写 "liveness check success"（worker_oc_server.cpp:1970-1975），超时未更新判定僵死。这与 K8s 侧 exec 文件探针一致。
- **`is_started()`**：因 dscli 退出后拿不到 worker PID，改为「是否已通过 ready/存在 liveness 文件」判断。
- **可选兜底**：按 `worker_address` 从进程表找 `datasystem_worker` PID 做进程级校验，与文件探活双保险。
- 直跑二进制路径（拍板项 1 → B）会显著简化：`Popen(["datasystem_worker", ...])` + `poll()`，但需 Motor 自实现 dscli 的配置/锁重试/ready 逻辑，工作量大，不推荐。

### 5.3 注册与生效机制（改动极小）

`motor/node_manager/core/services/registry.py` 的 `_DEFAULT_MODULE_MAP` 加 1 行：

```python
"yuanrong": ["motor.node_manager.core.services.yuanrong.lifecycle"],
```

`motor/node_manager/core/daemon.py` **零改动**，链路自动生效：

```
kv_cache_store_config.backend = "yuanrong"
  → services = "engine,yuanrong"            # daemon.py 已通用
  → discover() 导入 yuanrong.lifecycle       # 注册表填入 YuanRongWorker
  → _services["kv_store"] 被实例化            # 进入监视器守护名单
  → pull_kv_store() 调 pull()（dscli start + 等 ready）
  → _process_monitor_loop 每 5s 调 health_check()  ← liveness 过期自动 stop+start
```

### 5.4 Deployer 生成

- `examples/deployer/lib/constant.py`：YuanRong 常量（`YUANRONG_STORE_BACKEND`、`ENV_YR_WORKER_ADDRESS`、`ENV_YR_ETCD_ADDRESS`、`ENV_YR_READY_CHECK_PATH`、`ENV_YR_LIVENESS_CHECK_PATH` 等）。
- `examples/deployer/lib/generator/kv_cache_store.py`：
  - `gen_kv_store_env()` 加 `elif backend == "yuanrong":` 分支，注入 `YRC_WORKER_ADDRESS`、`YRC_ETCD_ADDRESS`、探活文件路径等；
  - `resolve_kv_store_target_job_id()` 复用现有跨服务逻辑（**注意**：判断"可复用"时要容忍旧 worker 的 metadata store 锁残留——dscli 会自动重试启动，见 §3.1）。
- `examples/deployer/lib/generator/k8s_utils.py`：ConfigMap 打包 `kv_store_backends.yuanrong.*` 脚本；`build_kv_store_env_items()` 加引擎侧 `YRC_*` 环境变量。
- **etcd 模板：默认不生成**（复用外部 etcd）；仅当选择自部署时才新增 `etcd_template.yaml`（StatefulSet，yuanrong 仓库无可复用清单，参考脚本是单实例 etcd）。

### 5.5 启动脚本

```
examples/deployer/startup/roles/kv_store_backends/yuanrong/
└── yuanrong.sh            # 复用外部 etcd 时仅做 env 准备；自部署时才负责 etcd 初始化
```

- `common.sh`：加 `sync_yuanrong_config()`，把部署时保存的 `worker_address`/`HOST_IP`、etcd 端点、探活文件路径解析/导出，**不拉起 worker**（由 NodeManager 拉）。
- `engine.sh` / `all_combine_in_single_container.sh`：各加 `yuanrong)` 分支，只调 `sync_yuanrong_config()`。
- `kv_cache_store.sh` 已通用，无需改。

### 5.6 指标采集（已定决策 4 → 解析 kv_metrics.log）

`motor/coordinator/metrics/metrics_collector.py`：

- `_filter_kvstore_metrics()` 加 `elif backend == "yuanrong":` 分发；
- 按已定决策 4 实现 `_filter_yuanrong_metrics()`（方案 B 定为最终方案，A/C 否决）：
  - **方案 B（最终）**：解析各节点 `kv_metrics.log`（JSON Lines，默认开启）取自定义指标（容量/key/淘汰），映射到公共 `kv_store_*` 四族。`kv_metrics.log` 在 worker 节点本地，coordinator 经 NodeManager 周期性转发获取（复用 NodeManager 已有的日志/状态上报通道或共享卷）。
  - ~~方案 A~~（否决）：brpc 层指标非 `kv_store_*` 目标。
  - ~~方案 C~~（否决）：暂不实现。

### 5.7 Conductor 侧调整（已定决策 3 → 按节点聚合）

现状假设 per-DP 多端口订阅，与 yuanrong 默认 per-node 单端口拓扑冲突。需调整：

1. **`conductor_api_client._register_yuanrong_dp`**：`medium_endpoints` 的 `_resolve_endpoint_url(pattern, ip, dp_rank)` 目前按 `port + dp_rank` 偏移（[conductor_api_client.py:245](motor/coordinator/api_client/conductor_api_client.py#L245)），对 yuanrong 改为**按节点聚合**：`_build_medium_endpoints` 新增 `node_level` 语义（按 backend mode 自动判断），同一节点所有 DP 上报同一个 `worker_address` 端口（无偏移），`dp_rank` 仍显式携带在 register_data 中。Mooncake/Memcache 的 per-DP 偏移不受影响（`node_level=False`）。
2. **`motor/kv_conductor/src/backend.rs`**：**实现验证无需新增 MatchMode**——现有 `MatchMode::None` + ZMQ 传输层扇出已天然实现按节点聚合：每个 DP 独立创建 ZMQ SUB 连接同一节点级 PUB 端口（[zmq_subscriber.rs:53](motor/kv_conductor/src/zmq_subscriber.rs#L53)），事件发布后传输层扇出到该节点全部 DP，每个 subscriber 以自身 `backend_id`（=注册的 `instance_id`）和自身 `dp_rank` 通过 `MatchMode::None` 应用到**自身**实例（[events/pool.rs:113](motor/kv_conductor/src/events/pool.rs#L113)）。故仅更新 [backend.rs](motor/kv_conductor/src/backend.rs) 与 [events/helpers.rs](motor/kv_conductor/src/events/helpers.rs) 中关于 YuanRong "per-DP 端口 / 固定 dp_rank" 的错误注释为**按节点单端口**语义。
3. **`motor/config/coordinator.py` `KvConductorConfig`**：`npu/cpu/disk_endpoint` 当前是 "Per-DP ... pattern"（`tcp://*:15557+dp_rank`），对 yuanrong 改为**节点级 worker 地址**（不带偏移）：端口解析时按 backend mode 决定是否加 `+dp_rank`，配置格式本身不变（`tcp://*:31501` 即节点 worker 端口）；`replay_endpoint` 维持 per-DP 偏移不变。

> 这是相对 v1「conductor 无需改动」假设的最大修正：per-DP 端口假设不成立，conductor 注册需按节点聚合调整；匹配侧借助 ZMQ PUB-SUB 传输层扇出 + `MatchMode::None` 自身实例语义，无需新增匹配模式。

---

## 6. 端口与环境变量设计

| 用途 | 端口/变量 |
|------|-----------|
| datasystem worker（默认） | 31501（`worker_address`，参考脚本用 32451） |
| etcd client（外部依赖） | 2379（`YRC_ETCD_ADDRESS`，逗号分隔多端点） |
| ready/liveness 探活文件 | `YRC_READY_CHECK_PATH` / `YRC_LIVENESS_CHECK_PATH` |
| 引擎侧注入 | `YRC_WORKER_ADDRESS`、`YRC_ETCD_ADDRESS`、`KV_STORE_BACKEND=yuanrong` |

---

## 8. 风险与待确认项

1. **守护基于文件探活（已定决策 1 → dscli）**：`health_check()` 只查 `liveness_check_path` mtime；进程假死时依赖 worker 自身 liveness 写入频率，需实测标定 `liveness_timeout_s`（默认 30s）。
2. **conductor 按节点聚合（已定决策 3）是唯一的结构性改动**：涉及 Rust `backend.rs` 匹配语义 + api_client 注册 + 配置语义三层，需评估对现有 Mooncake/Memcache 路径的回归影响。
3. **`kv_metrics.log` 的跨节点采集（已定决策 4 → B）**：coordinator 拿不到节点本地日志，需设计通道（NodeManager 转发 / 共享卷），是指标项的主要实现成本。
4. **锁残留**：`kv_store_reusable()` 复用判断要容忍旧 worker 的 metadata store 锁（dscli 自动重试），避免误判为"不可复用"。
5. **etcd 高可用**：复用外部 etcd 无此负担；自部署需 Motor 自建 StatefulSet（yuanrong 无可复用清单），并保证 stable identity。
6. **引擎侧连接器**：`YuanRongConnector` 在 vLLM `--kv-transfer-config` 配置、不经过 Motor，需确认 Motor 侧 `kv_transfer_config` 直传机制与参考部署一致。

---

## 9. 总结

- **接入形态**：仿 memcache 的 NodeManager 服务模式，把 `datasystem_worker` 的拉起与守护收编进 daemon；etcd 作为外部依赖复用（最简）或由 deployer 自建（备选）。
- **进程守护**：worker 挂掉/僵死后由 NodeManager 进程监视器（每 5 秒）通过 **liveness 文件新鲜度** 发现，执行 `stop + start` 重启；`dscli start` 的 ready-即-返回语义靠 `ready_check_path` 兜底。daemon 监视器/注册机制零改动，只需新增 lifecycle 模块 + registry 一行映射。
