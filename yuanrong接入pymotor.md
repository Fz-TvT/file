# YuanRong 后端接入设计说明（MindIE Motor）

> 本文档描述 **YuanRong KV 缓存后端接入 MindIE Motor 的方案**。先以 MemCache 后端为参照讲清"后端是如何接入的"，再给出 YuanRong 的接入方案与 **NodeManager 进程守护**修订。表述均来自仓库内代码与部署脚本的实现，未实现部分会明确标注为 TODO。

---

## 1. 背景：KV 池化与后端抽象

MindIE Motor 支持 PD 分离架构下的 **KV Cache 池化**：Prefill（P）实例把算好的 KV 块写入共享池，Decode（D）实例从池中拉取，跳过重复计算。池化的底层存储后端是可替换的，目前已有三种：

| 后端 | 池主控 | 节点存储进程 | 引擎侧连接 |
|------|--------|--------------|-----------|
| **Mooncake** | `mooncake_master`（独立 Pod） | — | MooncakeConnectorV1 |
| **MemCache** | MetaService（独立 Pod） | LocalService（NodeManager 管理） | AscendStoreConnector（`backend=memcache`） |
| **YuanRong** | **etcd 集群** | **datasystem worker（`dsc1`）** | vLLM 原生 yuanrong backend（patch 进 vLLM） |

后端接入 Motor 的链路包括四块：**配置层 → deployer（K8s 生成）→ 节点生命周期（NodeManager）→ 指标采集**。此外 kv-conductor（Rust）与调度器的 KV 亲和链路也按后端模式分发。

---

## 2. MemCache 后端接入全解析（参照）

MemCache 是接入最完整的后端，是新增后端的参照模板。注意此 MemCache **不是**开源 memcached，而是昇腾自研分布式对象存储组件（`memcache_hybrid` 包，预装在 Motor 镜像），提供 HBM/NPU → DRAM/CPU → SSD/Disk 三层缓存。

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

### 2.2 部署流程（deploy.py）

1. `update_kv_store_enabled_flag()` 探测是否启用 KV store。
2. `normalize_kv_cache_store_config()` 归一化，`backend` 默认 `"memcache"`。
3. `generate_yaml_kv_store()` 生成 **KV Store Pod**（Deployment `mindie-motor-kv-store` 1 副本 + Service `mindie-motor-kvs-master` 暴露 50088/50089/50090）。
4. `create_motor_config_configmap()` 把 memcache 的 4 个文件打进 `motor-config` ConfigMap（2 个启动脚本 + 2 个 `.conf`）。
5. 引擎 Pod 注入环境变量：`KVS_MASTER_SERVICE`、`KV_STORE_BACKEND=memcache`、`MMC_LOCAL_CONFIG_PATH`、`MMC_LOCAL_SERVICE_MODE`。

### 2.3 Pod 启动

**KV Store Pod**：`boot.sh → kv_cache_store.sh`（按 `KV_STORE_BACKEND` 分发）→ `memcache.sh` → `memcache_meta_service.py` 调用 `MetaService.setup()/main()`，用 `POD_IP` + 端口组 URL，成为池主控。

**引擎 Pod**：`engine.sh` 检测到 memcache 后调 `sync_mmc_local_config()`：

- 把 `.conf` 从 ConfigMap 复制为真实文件（memcache 拒绝符号链接）；
- 用 `$KVS_MASTER_SERVICE` 替换硬编码 DNS（支持 IPv6 括号包裹）；
- 导出 `MMC_LOCAL_CONFIG_PATH` 指向 `mmc-local-inprocess.conf`。

随后 `Daemon` 启动：

```python
services = "engine,memcache"           # 由 kv_cache_store_config.mode 决定
registry.discover(services=services)   # 导入 memcache lifecycle 模块
```

`@register_service("kv_store", backend="memcache")` 注册 `LocalService`；`pull_engine()` 第一阶段 `prepare()`（设环境变量、A2/A5 强制 inprocess），第二阶段启动 vLLM，最后 `pull_kv_store()` 按 `standalone`/`inprocess` 拉起 LocalService（standalone 时用子进程跑 `motor.node_manager.core.services.memcache.worker` 调 `DistributedObjectStore.init(0)`）。

### 2.4 运行时数据流

```
P 实例算完 KV → AscendStoreConnector 写入本地 DRAM 池
  → DistributedObjectStore 与 MetaService(tcp://...:50088) 同步元数据
  → MetaService 记录 KV 块位置(节点/DP rank)
D 实例 → 查 MetaService 定位块 → 按协议(RDMA/SDMA/URMA)从 P 节点拉取 → 跳过重算
```

### 2.5 可选增强与监控

- **kv-conductor**：单独 Pod，注册池（`memcache-pool`）+ 各 DP 的 HBM endpoint，维护 RadixTree 索引，供 `kv_cache_affinity` 调度。
- **指标**：`metrics_collector.py` 的 `_filter_kvstore_metrics(raw, backend)` 按 backend 分发到 `_filter_memcache_metrics()`，把 `memcache_*` 重命名为 `motor:memcache_*` 并聚合出 `kv_store_size/ratio/keys/eviction` 四族指标。

---

## 3. YuanRong 接入方案

### 3.1 现状盘点

#### 已完成（conductor/调度侧，无需改动）

| 模块 | 现状 |
|------|------|
| `motor/kv_conductor/src/backend.rs`（Rust） | `Backend::YuanRong` 已实现：per-node 多端口、`MatchMode::None`（端口号=DP）、per-DP 多介质订阅 |
| `motor/coordinator/api_client/conductor_api_client.py` | `_register_yuanrong_dp()` 已实现，按 DP 上报 `npu/cpu/disk` 三介质 endpoint |
| `KvConductorConfig` | `npu_endpoint`/`cpu_endpoint`/`disk_endpoint` 已支持 |
| `motor/coordinator/scheduler/policy/kv_cache_affinity.py` | `w_npu/w_cpu/w_disk` 三介质权重 + `npu/cpu/disk_blocks` 匹配已实现 |
| 测试 | `test_yuanrong_registration_dispatches_per_dp` 已有 |

#### 未完成（本次方案要做的全部）

**Motor 的 KV 池化链路完全没接 YuanRong**：`kv_cache_store_config.backend="yuanrong"` 从头到尾不通。`docs/zh/user_guide/features/kv_cache_store/README.md` 明确标注 `Yuanrong | TODO：后续版本支持`。参考案例（ModelArts）是**旁路部署**（手动传 etcd YAML + 脚本拉起 `dsc1`），完全绕开 Motor 的 deployer/生命周期/指标框架。

### 3.2 总体架构

#### 组件对照

| 组件 | Memcache | Mooncake | YuanRong |
|------|----------|----------|----------|
| 池主控 | MetaService（独立 Pod） | `mooncake_master`（独立 Pod） | **etcd 集群**（3 副本/复用已有） |
| 节点存储进程 | LocalService（NodeManager 管理） | — | **datasystem worker（`dsc1`）** |
| 引擎侧连接 | AscendStoreConnector | MooncakeConnectorV1 | vLLM 原生 yuanrong backend（patch 进 vLLM） |
| 事件发布 | ZMQ/HTTP | ZMQ（master PUB） | **每节点独立端口**（如 18481，端口=DP） |

#### 目标部署拓扑

```
┌─────────────────────┐     ┌─────────────────────────────┐
│  etcd 集群 (StatefulSet) │     │  引擎节点 (每节点)            │
│  3 副本 / 或复用外部 etcd │◄────│  引擎 Pod 内:               │
│  Service:              │     │    ├─ vLLM (patch yuanrong)  │
│   yrc-etcd-client:2379 │     │    └─ datasystem worker      │
└─────────────────────┘     │        dsc1 start --worker_     │
                            │        address HOST_IP:18481    │
                            │        --etcd_address ...       │
                            └─────────────────────────────┘
                                        │  per-DP 独立端口
                                        ▼
                              kv-conductor (已支持 YuanRong)
```

#### 职责分工

| 组件 | 谁管 | 为什么 |
|------|------|--------|
| **etcd 集群**（池主控） | **K8s**（StatefulSet 自带 `etcd-0/1/2` 重启恢复） | 独立 Pod，K8s 原生守护 |
| **datasystem worker**（`dsc1`，每节点） | **NodeManager** | 跑在引擎节点上、K8s 不直接管理，由 daemon 拉起 + 5s 健康检查自动重启 |

### 3.3 核心设计决策

**沿用 memcache 的 NodeManager 服务模式，不引入独立的"脚本拉起"路径。** 理由：

1. **YuanRong datasystem worker 跑在引擎节点上**，K8s 不直接管理，必须有一个节点级守护者——NodeManager 的 daemon 恰好就是（其 `_process_monitor_loop` 每 5 秒遍历所有注册服务的 `health_check()`，memcache 的 `health_check()` 就是"进程挂了 → `pull()` 重启"）。
2. **daemon 的注册机制天然可插拔**：`services = f"engine,{kv_cfg.backend}"` 对 `"yuanrong"` 自然生效，只需在 registry 的 `_DEFAULT_MODULE_MAP` 加映射。
3. mooncake 之所以 NodeManager 不管，是因为 `"mooncake"` 不在模块映射里、`discover()` 无模块可导入——这恰恰证明"加映射 → 被守护"是现成的开关。

> 注：此处与原旁路脚本（ModelArts 案例中 `start_yr_worker.sh` 的 while-true 等待与手动拉起）不同——拉起与守护职责整体移交 NodeManager，脚本只负责配置准备。

### 3.4 分模块方案

#### ① 配置层

`motor/config/node_manager.py` 的 `KVCacheStoreConfig` 增加 YuanRong 专属字段：

```python
# --- YuanRong ---
etcd_endpoints: str = ""          # --etcd_address（可为逗号分隔）
worker_port: int = 18481
cluster_name: str = ""
shared_memory_size_mb: int = 512000
node_timeout_s: int = 300
node_dead_timeout_s: int = 600
liveness_check_path: str = "/workspace/liveness"
log_dir: str = ""
restart_worker: bool = True       # 复用 MOTOR_RESTART_LOCAL_SERVICE 语义
```

`motor/common/utils/env.py` 增加对应环境变量读接口（仿 `mmc_*` 写法）。

#### ② NodeManager 生命周期★ 本次核心

新增 `motor/node_manager/core/services/yuanrong/lifecycle.py`，完全复刻 memcache `LocalService` 骨架（`@register_service` + `pull/stop/health_check`），但**去掉** inprocess/standalone 双模式和 conf 准备，只保留"子进程守护"：

```python
@register_service(
    SERVICE_KV_STORE,
    backend="yuanrong",
    prepare_priority=10,          # 引擎启动前先 prepare
    factory=_create_yuanrong_worker,
)
class YuanRongWorker:
    """Manage the yuanrong datasystem worker (dsc1) as a subprocess."""

    @property
    def _can_launch(self) -> bool:
        return self._kv_cfg.enable and self._kv_cfg.backend == "yuanrong"

    def should_launch(self) -> bool:
        return self._can_launch

    def prepare(self, **kwargs) -> None:
        """引擎启动前：解析 HOST_IP，校验 dsc1/etcd 可达，导出 env。"""
        if not self._can_launch: return
        self._host_ip = _resolve_host_ip()          # 复用 daemon 本机 IP 或 hostname -I
        if not shutil.which("dsc1"): raise ...      # 启动前快速失败

    def pull(self) -> None:
        """拉起 dsc1 子进程（memcache pull 同款）。"""
        if not self._can_launch or self.is_alive(): return
        env = os.environ.copy()
        env.update({"HOST_IP": self._host_ip, "ETCD_K8S_SERVICE": self._kv_cfg.etcd_endpoints})
        cmd = ["dsc1", "start", "-t", "600", "-w",
               "--worker_address", f"{self._host_ip}:{self._kv_cfg.worker_port}",
               "--etcd_address", self._kv_cfg.etcd_endpoints,
               "--cluster_name", self._kv_cfg.cluster_name,
               "--shared_memory_size_mb", str(self._kv_cfg.shared_memory_size_mb),
               "--node_timeout_s", str(self._kv_cfg.node_timeout_s),
               "--node_dead_timeout_s", str(self._kv_cfg.node_dead_timeout_s),
               "--liveness_check_path", self._kv_cfg.liveness_check_path,
               "--log_dir", self._kv_cfg.log_dir]
        self._proc = subprocess.Popen(cmd, shell=False, env=env)
        if self._proc.poll() is not None:
            raise RuntimeError("dsc1 exited immediately: %s" % self._proc.returncode)

    def stop(self) -> None:          # 幂等，SIGKILL，仿 memcache
        ...
    def is_started(self) -> bool:    return self._proc is not None
    def is_alive(self) -> bool:      return self._proc.poll() is None

    def health_check(self) -> None:  # ← 5s 监视器自动调用
        """进程挂了就拉起（DaemonService protocol）。"""
        if self.is_started() and not self.is_alive():
            logger.warning("yuanrong worker died (restart=%s)", self.restart_worker)
            self._proc = None
            if self.restart_worker:
                self.pull()          # 自动重启
```

要点：`dsc1` 是 CLI 命令，直接 `subprocess.Popen`，**不需要** memcache 那种 `worker.py` 模块入口。

#### ③ 注册与生效机制

`motor/node_manager/core/services/registry.py` 的 `_DEFAULT_MODULE_MAP` 新增映射：

```python
"yuanrong": ["motor.node_manager.core.services.yuanrong.lifecycle"],
```

`motor/node_manager/core/daemon.py` **零改动**，链路自动生效：

```
kv_cache_store_config.backend = "yuanrong"
  → services = "engine,yuanrong"            # daemon.py 已通用
  → discover() 导入 yuanrong.lifecycle       # 注册表填入 YuanRongWorker
  → _services["kv_store"] 被实例化            # 进入监视器守护名单
  → pull_kv_store() 调 pull() 拉起 dsc1
  → _process_monitor_loop 每 5s 调 health_check()  ← 挂掉自动重启
```

#### ④ Deployer 生成

- `examples/deployer/lib/constant.py`：YuanRong 相关常量（`YUANRONG_STORE_BACKEND`、`DEFAULT_ETCD_PORT`、`ENV_ETCD_ENDPOINTS` 等）。
- `examples/deployer/lib/generator/kv_cache_store.py`：
  - `gen_kv_store_env()` 加 `elif backend == "yuanrong":` 分支，注入 `YRC_ETCD_ENDPOINTS`、`YRC_WORKER_PORT`；
  - `generate_yaml_kv_store()` 加 etcd 端口同步逻辑；
  - `resolve_kv_store_target_job_id()` 复用现有跨服务逻辑（检测已有 etcd Service + Running Pod → 复用）。
- `examples/deployer/lib/generator/k8s_utils.py`：ConfigMap 打包 `kv_store_backends.yuanrong.*` 脚本；`build_kv_store_env_items()` 加引擎侧 `YRC_*` 环境变量。
- `examples/deployer/yaml_template/etcd_template.yaml`（新增）：StatefulSet 3 副本 + `etcd-client`/`etcd-headless` 两个 Service。

#### ⑤ 启动脚本

```
examples/deployer/startup/roles/kv_store_backends/yuanrong/
└── yuanrong.sh                # 只负责 etcd（自部署时）；datasystem worker 不在 KV Store Pod
```

- `common.sh`：加 `sync_yuanrong_config()`，负责把 `HOST_IP`、`YRC_ETCD_ENDPOINTS` 解析/格式化并导出，**不再拉起 dsc1**。
- `engine.sh` / `all_combine_in_single_container.sh`：各加 `yuanrong)` 分支，只调 `sync_yuanrong_config()`。
- `kv_cache_store.sh` 已通用（按 `$KV_STORE_BACKEND` 找脚本），无需改。

#### ⑥ 指标采集

`motor/coordinator/metrics/metrics_collector.py`：

- `_filter_kvstore_metrics()` 加 `elif backend == "yuanrong": return _filter_yuanrong_metrics(raw)`；
- 新增 `_filter_yuanrong_metrics()`：解析 etcd `/metrics`（Prometheus 原生 `etcd_server_*`）与 datasystem 指标，映射到公共 `kv_store_*` 四族（复用 `_emit_kv_store_prometheus()`）；
- 端口选择逻辑（当前硬编码 `mooncake→50088 / 其他→50090`）加 YuanRong 分支。

#### ⑦ Conductor 侧

注册、查询、亲和度量的三介质链路均已实现，集成后跑通端到端验证（注册上报 `medium_endpoints` → conductor 建立 per-DP 多端口订阅 → 调度命中 `npu/cpu/disk_blocks`）。

### 3.5 端口与环境变量设计

| 用途 | 端口/变量 |
|------|-----------|
| etcd client | 2379（`YRC_ETCD_ENDPOINTS`） |
| etcd peer | 2380 |
| datasystem worker | 18481（`YRC_WORKER_PORT`，**端口号=DP**，conductor 据此匹配） |
| 引擎侧注入 | `YRC_ETCD_ENDPOINTS`、`YRC_WORKER_PORT`、`KV_STORE_BACKEND=yuanrong` |

### 3.6 健康检查增强（进程级之上，可选）

`dsc1` 启动时带 `--liveness_check_path`（worker 自己写存活文件）。`health_check()` 可增强为双检测：

```python
def health_check(self):
    # ① 进程级：poll() 死了 → 重启
    # ② 存活文件：liveness 文件超过 N 秒未更新 → 判定僵死 → kill + pull()
```

进程活着但 datasystem 假死时，仅靠 `poll()` 兜不住。第一版可先做 ①，②作为快速迭代项。


### 3.7 风险与待确认项

1. **`dsc1` 是否前台阻塞**：参考脚本直接前台跑，假定前台阻塞。若 `dsc1 start` 会自行 daemonize 脱离，`poll()` 守护会失效，必须改用 `--liveness_check_path` 文件检测（即增强项 ②）。
2. **本机 IP 获取**：参考案例用 `hostname -I`，Motor 侧是否有更可靠的 daemon 本机 IP 来源（如 engine 的 `endpoints` 信息）需对齐。
3. **etcd 由谁部署**：支持"复用外部 etcd"时，`etcd_endpoints` 直接填外部地址，NodeManager 只管 worker（最简）；自部署走 deployer 生成 etcd StatefulSet。
4. **重启开关粒度**：建议复用 `MOTOR_RESTART_LOCAL_SERVICE` 控制 `restart_worker`，少引入一个概念。
5. **引擎侧连接器**：ModelArts 案例把 yuanrong backend patch 进 vLLM，不经过 Motor 的 `kv_transfer_config`。需确认 vLLM 侧是否存在可配置的 connector 名称，或继续沿用"vLLM 预打 patch + Motor 只管 datasystem 生命周期"的分工。
6. **`dsc1` 的 per-DP 端口**：conductor 的 `MatchMode::None` 假设"端口号=DP 唯一标识"，需确认 datasystem worker 是否为每个 DP 暴露独立端口。
7. **etcd 高可用与故障恢复**：etcd 是 3 副本 StatefulSet，Pod 重建需保持 stable identity（`etcd-0/1/2`），与现有 `kv_store_reusable()` 检测逻辑要兼容。
8. **指标格式**：datasystem 是否暴露 Prometheus `/metrics` 未确认。若只暴露内部接口，`kv_store_*` 聚合可能改从 etcd 侧采集或跳过。

---

## 4. 总结

- **接入形态**：仿 memcache 的 NodeManager 服务模式，把 etcd + `dsc1` datasystem 的部署收编进 Motor 的 deployer 框架，同时复用已完成的 conductor/调度三介质链路。
- **进程守护**：datasystem worker 挂掉后，由 NodeManager 的进程监视器（每 5 秒）通过 `health_check()` 发现并自动 `pull()` 重启；daemon 监视器、注册机制、`services` 计算全部零改动，只需新增一个 lifecycle 模块 + registry 映射。
- **首要风险**：`dsc1` 是否前台阻塞（决定守护基于 `poll()` 还是 liveness 文件）。
