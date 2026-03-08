# DistributedRuntime 包含什么、管理什么资源

本文结合代码说明 **DistributedRuntime（DRT）** 的组成与所管理的资源。Rust 定义在 `dynamo/lib/runtime/src/distributed.rs`，Python 侧为封装层。

---

## 1. 定义与位置

- **Rust 结构体**：`dynamo/lib/runtime/src/distributed.rs` 第 41–78 行  
- **Python 绑定**：`dynamo/lib/bindings/python/rust/lib.rs` 第 443–446 行（`inner: rs::DistributedRuntime`），构造在第 548–616 行  

注释（distributed.rs 37–39 行）概括了其角色：

> Distributed [Runtime] which provides access to **shared resources across the cluster**, this includes **communication protocols and transports**.

---

## 2. 字段一览（Rust 结构体）

以下为 `DistributedRuntime` 的成员及其含义（均来自 `distributed.rs`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `runtime` | `Runtime` | 本地运行时：Tokio runtime、取消令牌、计算池等 |
| `nats_client` | `Option<transports::nats::Client>` | NATS 客户端（可选），用于 KV 路由事件、NATS 请求平面等 |
| `network_manager` | `Arc<NetworkManager>` | 请求平面网络：TCP/HTTP/NATS 服务端与客户端创建与配置 |
| `tcp_server` | `Arc<OnceCell<Arc<tcp::server::TcpStreamServer>>>` | TCP RPC 服务端，懒初始化 |
| `system_status_server` | `Arc<OnceLock<Arc<SystemStatusServerInfo>>>` | 系统状态 HTTP 服务（健康、指标、/metadata），按配置可选启动 |
| `request_plane` | `RequestPlaneMode` | 请求平面模式：Tcp / Http / Nats |
| `discovery_client` | `Arc<dyn Discovery>` | 服务发现：注册/注销、list、list_and_watch |
| `discovery_metadata` | `Option<Arc<RwLock<DiscoveryMetadata>>>` | 发现元数据（Kubernetes 等用），可暴露给 system status /metadata |
| `component_registry` | `component::Registry` | 组件注册表：复用同一组件的 endpoint watcher、NATS 服务等 |
| `instance_sources` | `Arc<Mutex<InstanceMap>>` | 各 endpoint 的实例源（watch receiver） |
| `system_health` | `Arc<Mutex<SystemHealth>>` | 健康状态、uptime、endpoint 健康、健康检查目标 |
| `local_endpoint_registry` | `LocalEndpointRegistry` | 本进程内 endpoint 调用注册（同进程直连） |
| `metrics_registry` | `MetricsRegistry` | Prometheus 指标注册表（DRT 为根层级） |
| `engine_routes` | `EngineRouteRegistry` | /engine/* 路由回调注册（如 sleep、wake_up、start_profile） |

---

## 3. 各资源在做什么

### 3.1 Runtime（本地运行时）

- **定义**：`dynamo/lib/runtime/src/runtime.rs` 中 `struct Runtime`。  
- **持有**：  
  - 主/从 Tokio runtime（`primary` / `secondary`），用于驱动异步任务与后台 etcd/NATS 等。  
  - `cancellation_token`（主取消令牌）、`endpoint_shutdown_token`（子令牌）。  
  - `graceful_shutdown_tracker`、可选 `compute_pool`、`block_in_place_permits`。  
- **在 DRT 中的用法**：  
  - `primary_token()` / `child_token()` 用于关闭整个 DRT 或子任务。  
  - `runtime.secondary()` 用于在绑定层里执行 `DistributedRuntime::new` 等异步构造。  
  - `shutdown()` 时（distributed.rs 324–327）会调用 `runtime.shutdown()` 和 `discovery_client.shutdown()`。

### 3.2 Discovery（服务发现）

- **接口**：`dynamo/lib/runtime/src/discovery/mod.rs` 中 `trait Discovery`（约 688–715 行）：  
  - `instance_id()`：本实例 ID。  
  - `register(spec)` / `unregister(instance)`：注册/注销。  
  - `list(query)`：一次快照。  
  - `list_and_watch(query, cancel_token)`：持续监听变更流。  
  - `shutdown()`：清理。  
- **实现**：  
  - **Kubernetes**：`KubeDiscoveryClient`（distributed.rs 131–145）。  
  - **KV 存储**：`KVStoreDiscovery`，底层为 etcd / file / memory（distributed.rs 146–166）。  
- **谁用**：  
  - Frontend HTTP 的 `run_watcher` 通过 `list_and_watch` 监听模型注册，并更新 HTTP 的 ModelManager。  
  - Worker 侧通过 Python 的 `register_model` → Rust `register_model` 调用 `endpoint.inner.drt().discovery().register(spec)` 注册模型。  

因此 DRT **管理**的是「发现后端」的客户端与连接；**不直接**存模型列表，而是提供发现能力给上层使用。

### 3.3 NetworkManager（请求平面）

- **定义**：`dynamo/lib/runtime/src/pipeline/network/manager.rs`。  
- **职责**（见该文件注释）：  
  - 统一读取网络相关环境变量（如 `DYN_TCP_RPC_PORT`、`DYN_HTTP_RPC_*`）。  
  - 根据 `RequestPlaneMode` 创建/提供 **RequestPlaneServer** 与 **RequestPlaneClient**。  
  - 管理全局共享的 TCP server（多 worker 同进程时共用一个端口）。  
- **创建**：distributed.rs 172–177：  
  - `NetworkManager::new(runtime.child_token(), nats_client, component_registry, request_plane)`。  
- **暴露**：`network_manager()`、`request_plane_server()`（等价于 `network_manager().server().await`）。  

DRT 通过 **NetworkManager** 管理「请求如何从 Frontend 发到 Worker」所用的传输层（TCP/HTTP/NATS），而不是自己直接握有 socket。

### 3.4 TCP Server

- **类型**：`Arc<OnceCell<Arc<tcp::server::TcpStreamServer>>>`，懒初始化。  
- **获取**：`tcp_server()`（distributed.rs 338–348）在首次调用时 `TcpStreamServer::new(options).await`。  
- **用途**：请求平面为 TCP 时，作为 RPC 服务端接收请求；与 NetworkManager 配合，由 NetworkManager 决定何时/如何创建或复用 server。

### 3.5 System Status Server

- **类型**：`Arc<OnceLock<Arc<SystemStatusServerInfo>>>`，在 `DistributedRuntime::new` 里按配置可选启动（distributed.rs 215–255）。  
- **条件**：`config.system_server_enabled()`（通常由 `DYN_SYSTEM_PORT` 等控制）。  
- **作用**：提供系统状态 HTTP 服务（健康、Prometheus 指标、/metadata 等）。Worker 设置 `DYN_SYSTEM_PORT` 时由该服务暴露指标与健康；Frontend 会主动 unset 该变量，不启动或复用该端口。

### 3.6 Component Registry

- **定义**：`component::Registry`，内部为 `Arc<Mutex<RegistryInner>>`（`dynamo/lib/runtime/src/component/registry.rs`）。  
- **用途**（distributed.rs 58–62 注释）：  
  - 同一组件的多个实例（如多个访问同一远程 component 的 client）**共享**同一份 endpoint watcher 等资源。  
  - 减少对 etcd 等后端的 watch 数量。  
- **NATS 相关**：`register_nats_service(component)`（distributed.rs 341–405）会把 NATS 服务按 component 名注册到 registry，避免同一 component 重复建 NATS 服务。

### 3.7 Instance Sources

- **类型**：`Arc<Mutex<InstanceMap>>`，`InstanceMap = HashMap<Endpoint, Weak<Receiver<Vec<Instance>>>>`。  
- **含义**：按 endpoint 保存「实例列表」的 watch receiver 的弱引用。用于在需要时拿到某 endpoint 的当前实例列表（供路由、健康检查等使用）。

### 3.8 System Health

- **定义**：`dynamo/lib/runtime/src/system_health.rs` 中 `struct SystemHealth`。  
- **内容**：  
  - 系统级健康状态、各 endpoint 健康状态、健康检查目标、notifier。  
  - 启动时间与 **uptime**；在 DRT 构造时初始化 uptime gauge（distributed.rs 196–212），并注册到 `metrics_registry` 的 update callback，在每次 Prometheus scrape 前刷新。  
- **暴露**：`system_health()` 返回 `Arc<Mutex<SystemHealth>>`。

### 3.9 Local Endpoint Registry

- **用途**：本进程内 **in-process** 的 endpoint 调用注册。当调用方与被调方在同一进程时，可走本地注册表直调，而不经网络。  
- **暴露**：`local_endpoint_registry()`。

### 3.10 Metrics Registry

- **类型**：`MetricsRegistry`（distributed.rs 189）。  
- **层级**：DRT 实现 `MetricsHierarchy`，`get_metrics_registry()` 返回自己的 `metrics_registry`，作为根。  
- **使用**：Frontend HTTP 在构建 HttpService 时通过 `drt_metrics(Some(distributed_runtime.get_metrics_registry().clone()))`（lib/llm 的 http.rs）注入，用于暴露组件级、路由级等指标。

### 3.11 Engine Routes

- **类型**：`engine_routes::EngineRouteRegistry`。  
- **用途**：注册 **/engine/{route_name}** 的回调（如 `sleep`、`wake_up`、`start_profile`）。这些路由由 system status server 暴露，请求进来后根据 route 名分发到对应回调。  
- **Python**：`runtime.register_engine_route(route_name, callback)`（_core.pyi 94–117）即写此注册表。

### 3.12 NATS Client（可选）

- **创建**：若 `DistributedConfig.nats_config` 为 `Some`，则 `nats_config.connect().await`（distributed.rs 104–107）。  
- **用途**：  
  - 请求平面为 NATS 时的 RPC。  
  - KV 路由相关：事件发布/订阅、replica sync、worker 查询（`kv_router_nats_publish`、`kv_router_nats_subscribe`、`kv_router_nats_request`，distributed.rs 394–431）。  
- **未启用时**：`enable_nats=false` 或未配置 NATS 时，`nats_client` 为 `None`，上述 KV 路由发布会静默跳过，订阅/请求会报错（需要 NATS）。

---

## 4. 配置来源：DistributedConfig

DRT 的「用什么发现、用什么传输、要不要 NATS」由 **DistributedConfig** 决定（distributed.rs 409–424）：

- **discovery_backend**：`DiscoveryBackend::Kubernetes` 或 `KvStore(etcd|file|mem)`。  
- **nats_config**：`Option<nats::ClientOptions>`。  
- **request_plane**：`RequestPlaneMode::Tcp | Http | Nats`。  

Python 侧（lib.rs 548–616）：  
- 从参数得到 `discovery_backend`（字符串）、`request_plane`（字符串）、`enable_nats`（可选）。  
- 若已有 Worker 的 Tokio runtime 则复用，否则 `Worker::from_settings()` 新建。  
- 组装 `DistributedConfig` 后调用 `rs::DistributedRuntime::new(runtime, runtime_config)`。  

环境变量（如 `DYN_DISCOVERY_BACKEND`、`DYN_REQUEST_PLANE`、NATS 相关）在 Rust 的 `DistributedConfig::from_settings()` 中读取（distributed.rs 416–423），Python 构造 DRT 时也可通过参数覆盖。

---

## 5. 小结表：DRT 包含什么、管理什么

| 资源/能力 | 在 DRT 中的形式 | 管理的内容 |
|-----------|-----------------|-------------|
| 本地执行与生命周期 | `runtime: Runtime` | Tokio runtime、取消令牌、优雅关闭、计算池 |
| 服务发现 | `discovery_client: Arc<dyn Discovery>` | 连接与配置（etcd/K8s/file/mem）；不直接存模型列表 |
| 请求平面 | `network_manager` + `request_plane` + `tcp_server` | TCP/HTTP/NATS 服务端与客户端、端口与配置 |
| 可选 NATS | `nats_client: Option<...>` | KV 事件、NATS 请求平面、worker 查询 |
| 系统状态与健康 | `system_status_server` + `system_health` | HTTP 健康/指标/metadata、uptime、endpoint 健康 |
| 组件复用 | `component_registry` | 同组件共享 watcher、NATS 服务等 |
| 实例视图 | `instance_sources` | 各 endpoint 的实例列表 watch |
| 本进程调用 | `local_endpoint_registry` | 同进程 endpoint 直连 |
| 指标 | `metrics_registry` | 根级 Prometheus 注册表，供 Frontend/系统状态使用 |
| 扩展路由 | `engine_routes` | /engine/* 回调（sleep、wake_up 等） |

整体上，**DistributedRuntime** 是「分布式运行时」的入口：不直接执行业务逻辑，而是**持有并管理**发现、网络、健康、指标、生命周期等共享资源，供 Frontend、Worker、Router 等组件通过同一 DRT 协同工作。
