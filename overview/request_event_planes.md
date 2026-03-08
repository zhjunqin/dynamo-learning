# Dynamo 中的 Request Plane 与 Event Plane

本文解释 Dynamo 运行时中两个常见概念：**Request Plane（请求平面）** 与 **Event Plane（事件平面）**。它们分别解决“请求如何从路由器/前端到 worker 并返回结果”和“系统内部状态事件如何发布/订阅传播”的问题。

---

## 核心定义（一句话）

- **Request Plane**：用于**点对点请求分发与调用**（典型是 router/frontend → worker 的 RPC/握手/投递），强调低延迟、可寻址（发给哪个实例/endpoint）。
- **Event Plane**：用于**发布-订阅事件流**（异步广播、状态传播、监控/路由信息同步），强调扇出、多订阅者解耦、后台消费。

在 Dynamo 里，两者都由 `DistributedRuntime` 管理，并与 **Discovery（服务发现）** 协作：

- **Request Plane**：worker 启动 server 并将 endpoint 的“可达地址”注册到 discovery；客户端/路由器 watch discovery 拿到实例列表，然后按 transport 发送请求。
- **Event Plane**：发布者/订阅者按 scope+topic 建立 pub/sub；在某些 transport（如 ZMQ direct）下也会借助 discovery 来发现发布者并动态连接。

---

## Request Plane（请求平面）

### 配置项与取值

CLI / 环境变量：

- 参数：`--request-plane`
- 环境变量：`DYN_REQUEST_PLANE`
- 可选值：`tcp`（默认）、`http`、`nats`

（定义在 `dynamo/components/src/dynamo/frontend/frontend_args.py` 与 Rust runtime 的 `RequestPlaneMode`。）

### 这三种模式“怎么工作”

Request Plane 的关键抽象：

- server 侧统一接口：`RequestPlaneServer`（HTTP/TCP/NATS 都实现该 trait）
- client 侧统一接口：`RequestPlaneClient`（HTTP/TCP/NATS 都实现该 trait）
- discovery 中记录的“实例地址”：`TransportType::{Tcp,Http,Nats}`

#### 1) `tcp`（默认）

- **地址形态（写入 discovery）**：`host:port/<instance_id_hex>/<endpoint_name>`
- **含义**：同一 TCP server 可 multiplex 多 endpoint；当同进程多 worker 时，用 `<instance_id>/<endpoint>` 避免碰撞。
- **client 行为**：解析 `host:port/...`，并把路径部分写入 header（例如 `x-endpoint-path`），再直连 TCP 发送请求。

**适用**：同网段/同集群内，追求最低开销与延迟。

#### 2) `http`

- **地址形态（写入 discovery）**：`http://host:port/v1/rpc/<endpoint_name>`（root path 可通过 `DYN_HTTP_RPC_ROOT_PATH` 配置）
- **含义**：请求平面使用 HTTP/2 client/server（实现上是对 address 做 POST）。
- **client 行为**：对 discovery 给出的 URL 做 HTTP POST，payload 为二进制（`application/octet-stream`）。

**适用**：需要更强“基础设施兼容性”（代理/网关/调试/可观测性）时。

#### 3) `nats`（legacy）

- **地址形态（写入 discovery）**：一个 NATS subject（对 endpoint + instance 唯一化）
- **含义**：请求通过 NATS request/reply 在 subject 上投递。
- **client 行为**：`request_with_headers(subject, headers, payload)` 等待 reply。

**适用**：路由器与 worker 之间不方便直连（只要都能连 NATS 即可）、或历史原因需要 NATS request-plane。

### Request Plane 与 Discovery 的关系（注册与发现）

worker 侧：当你在 Rust 侧启动 endpoint（或 Python 侧调用 `Endpoint.serve_endpoint(...)`），运行时会：

1. 启动 request-plane server 并注册 handler；
2. 生成对应 `TransportType`；
3. 将 `DiscoverySpec::Endpoint` 注册到 discovery。

client/router 侧：对同一个 `namespace.component.endpoint` 做 `DiscoveryQuery::Endpoint` 的 `list_and_watch`，持续获得实例列表（包含 `TransportType`），再按 transport 选择 address 并通过 `RequestPlaneClient` 发起请求。

---

## Event Plane（事件平面）

### 配置项与取值

CLI / 环境变量：

- 参数：`--event-plane`
- 环境变量：`DYN_EVENT_PLANE`
- 可选值：`nats`（默认）、`zmq`

另外还有编码与 broker 相关可选项（取决于 transport）：

- `DYN_EVENT_PLANE_CODEC`：事件 envelope/payload 编码（transport 有默认值）
- ZMQ broker 配置：`DYN_ZMQ_BROKER_URL` 或 `DYN_ZMQ_BROKER_ENABLED=true`（并通过 discovery 查找 broker）

### `nats` vs `zmq` 的区别

#### 1) `event_plane=nats`

- **拓扑**：中心 broker（NATS）式 pub/sub
- **寻址**：subject = `scope_prefix + "." + topic`
  - scope_prefix 由 `EventScope` 决定（namespace-scoped 或 component-scoped）
- **优点**：跨机/跨网段更省心（只要都能连 NATS）；多订阅者扇出自然；运维体系成熟（可进一步扩展到持久化/流等能力）。
- **缺点**：依赖 NATS；多一跳 broker 带来额外开销。

#### 2) `event_plane=zmq`

ZMQ 有两种运行形态：

- **Direct 模式（默认）**：
  - 发布者在**自己进程内** bind 一个 PUB socket（通常随机端口 `tcp://0.0.0.0:0`）。
  - 发布者将对外可达 endpoint（如 `tcp://<ip>:<port>`）注册到 discovery（`DiscoverySpec::EventChannel`，transport=`EventTransport::Zmq{endpoint}`）。
  - 订阅者启动 `DynamicSubscriber`：watch discovery 的 `EventChannel` 变化，发现发布者就 connect SUB。
  - **前提**：订阅者网络上能直接连到发布者公布的 endpoint（容器/NAT/防火墙需要小心）。

- **Broker 模式（可选）**：
  - 发布者 connect broker 的 XSUB，订阅者 connect broker 的 XPUB。
  - broker 可通过环境变量显式配置，或通过 discovery 查找已注册的 broker。
  - 在 broker 模式下，发布者通常**不再注册每个 publisher 的 endpoint 到 discovery**（因为订阅者统一连 broker）。

### Event Plane 里“谁是发布者/订阅者”

Event Plane 是通用能力：任何模块都可以成为发布者或订阅者。当前代码中最典型的是 KV Router 生态：

- **KV events（`kv-events`）**
  - 发布者：KV cache / KV router publisher 侧将 `KvCacheEvent` 包装成 `RouterEvent` 并 publish
  - 订阅者：KV router subscriber 侧订阅并将事件喂给 indexer（本地索引/路由状态更新）

- **Active sequences events（`active_sequences_events`）**
  - 发布者：多 worker 序列追踪（用于路由/同步/调度）
  - 订阅者：例如 LoRA load estimator（多 router 模式下事件驱动统计）等

- **KV metrics（`kv_metrics`）**
  - 发布者：负载/忙闲指标发布（namespace-scoped）
  - 订阅者：worker monitor 等，用于 busy/free 检测、指标清理等（订阅可能是可选的）

这些 topic 常量集中在 `dynamo/lib/llm/src/kv_router.rs`。

---

## 常见选型建议

### Request Plane 怎么选？

- **追求性能/同网段直连**：优先 `tcp`
- **更强的基础设施兼容/调试友好**：选 `http`
- **网络环境限制导致直连困难**：考虑 `nats`（但要接受额外组件与延迟；并注意其在代码中被标注为 legacy request plane）

### Event Plane 怎么选？

- **想要跨机省心、统一 broker**：选 `nats`
- **同网段、想走更直连/更轻的 pubsub**：`zmq` direct 可行
- **想用 ZMQ 但 direct 模式网络不可达**：开启 `zmq` broker 模式，让订阅者统一连 broker

---

## 进一步阅读（源码入口）

- Request plane 模式与地址构造：`dynamo/lib/runtime/src/component/endpoint.rs`
- Request plane client/server 抽象与实现：
  - server：`dynamo/lib/runtime/src/pipeline/network/ingress/`
  - client：`dynamo/lib/runtime/src/pipeline/network/egress/`
  - 网络管理器：`dynamo/lib/runtime/src/pipeline/network/manager.rs`
- Event plane 核心：`dynamo/lib/runtime/src/transports/event_plane/`
  - ZMQ direct/broker、discovery registration：`event_plane/mod.rs`
  - ZMQ transport：`event_plane/zmq_transport.rs`
  - 动态订阅者：`event_plane/dynamic_subscriber.rs`
- KV router 事件的发布/订阅（最典型使用方）：
  - topics：`dynamo/lib/llm/src/kv_router.rs`
  - publisher：`dynamo/lib/llm/src/kv_router/publisher.rs`
  - subscriber：`dynamo/lib/llm/src/kv_router/subscriber.rs`

