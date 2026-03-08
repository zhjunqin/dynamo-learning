# Dynamo 源码结构与运行逻辑

本文从最简单示例 `dynamo/examples/backends/vllm/launch/agg.sh` 出发，介绍 Dynamo 的源码结构、整体架构与基本运行逻辑。

---

## 一、示例脚本在做什么：`agg.sh`

```bash
# 1. 后台启动前端（Ingress）
python -m dynamo.frontend &

# 2. 前台启动 vLLM 工作进程（Worker）
DYN_SYSTEM_PORT=${DYN_SYSTEM_PORT:-8081} \
    python -m dynamo.vllm --model "$MODEL" --enforce-eager "${EXTRA_ARGS[@]}"
```

- **dynamo.frontend**：对外提供 HTTP API（默认 `DYN_HTTP_PORT=8000`），做请求接入、预处理、路由。
- **dynamo.vllm**：实际跑 vLLM 推理的 Worker，通过环境变量 `DYN_SYSTEM_PORT`（默认 8081）暴露系统/指标端口。

因此「聚合模式（1 GPU）」= 一个 Frontend 进程 + 一个 vLLM Worker 进程，请求经 Frontend 转发到该 Worker。

---

## 二、源码目录结构（与示例相关的部分）

Dynamo 仓库根目录下主要有：

- **`components/src/dynamo/`**：Python 包主体，包含 frontend、vllm、trtllm、sglang 等组件。
- **`lib/`**：Rust 库与 Python 绑定（runtime、llm、bindings 等）。
- **`examples/`**：示例与启动脚本（如 `examples/backends/vllm/launch/agg.sh`）。

### 2.1 Python 包：`components/src/dynamo/`

安装后的包名是 `dynamo`，由 `pyproject.toml` 的 `[tool.hatch.build.targets.wheel] packages = ["components/src/dynamo"]` 决定。

与 agg.sh 直接相关的子包：

| 路径 | 作用 |
|------|------|
| `frontend/` | HTTP/gRPC 入口、预处理、路由（`python -m dynamo.frontend`） |
| `vllm/` | vLLM 后端封装：启动 vLLM、注册模型、处理 generate/clear 等（`python -m dynamo.vllm`） |
| `common/` | 配置、runtime 创建、优雅退出等公共逻辑 |
| `router/`、`planner/`、`profiler/` 等 | 路由策略、扩缩容规划、性能分析等（示例未显式用到） |

### 2.2 入口与调用链

**Frontend**

- 入口：`components/src/dynamo/frontend/__main__.py` → `dynamo.frontend.main.main()`。
- `frontend/main.py` 中：
  - 解析参数得到 `FrontendConfig`（含 `--http-port`、`--router-mode`、discovery/request_plane 等）。
  - 创建 `DistributedRuntime`（来自 Rust 绑定 `dynamo.runtime`）。
  - 根据 `EngineType.Dynamic` 和参数构建 `EntrypointArgs`，调用 **Rust** 的 `make_engine(runtime, e)` 得到 engine 配置。
  - 调用 **Rust** 的 `run_input(runtime, "http", engine)`，启动 HTTP 服务并连接发现与请求平面。

**vLLM Worker**

- 入口：`components/src/dynamo/vllm/__main__.py` → `dynamo.vllm.main.main()`。
- `vllm/main.py` 中：
  - `parse_args()` 得到 `Config`（Dynamo 运行时参数 + vLLM 的 `AsyncEngineArgs`）。
  - 若非 headless：`create_runtime()` 创建同样的 `DistributedRuntime`（discovery_backend、request_plane、event_plane、use_kv_events）。
  - 聚合单机模式下走 `init()`：创建 vLLM 引擎、`DecodeWorkerHandler`、调用 **Rust** 的 `register_model(...)` 向发现后端注册模型。
  - 在 `runtime.endpoint(...)` 上 `serve_endpoint(handler.generate, ...)`，开始对外提供 generate/clear 等 RPC。

### 2.3 Rust 侧（核心运行时与引擎）

- **`lib/bindings/python/`**：Python 绑定的实现与入口。
  - `rust/llm/entrypoint.rs`：`make_engine`、`run_input`；将 Python 的 `EntrypointArgs` 转成 Rust 的 `LocalModel` + `EngineConfig`（如 `EngineType::Dynamic` 对应动态引擎 + 可选 chat_engine_factory）。
  - `rust/lib.rs`：`register_model`、`unregister_model` 等；`register_model` 内部会通过 `endpoint.inner.drt().discovery()` 调用发现后端（如 etcd）注册 Model Deployment Card。
- **`dynamo.runtime`**：在 `lib/bindings/python/src/dynamo/runtime/__init__.py` 中从 `dynamo._core` 导出 `DistributedRuntime`。Runtime 负责发现、请求平面（TCP/NATS）、事件平面等。
- **`dynamo.llm`**：在 `lib/bindings/python/src/dynamo/llm/__init__.py` 中从 `dynamo._core` 导出 `make_engine`、`run_input`、`register_model` 等，供 frontend 与 vllm 使用。

因此：**请求如何被路由到 Worker** 由 Rust 的 runtime + 发现服务决定；**谁在监听 HTTP** 由 frontend 的 `run_input(..., "http", engine)` 决定；**Worker 如何被发现** 由 vllm 进程里的 `register_model` 写 discovery（如 etcd）决定。

---

## 三、整体架构设计（与示例对应的最小视图）

```
                    ┌─────────────────────────────────────────┐
                    │  User / curl                            │
                    │  POST /v1/chat/completions              │
                    └─────────────────┬───────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  dynamo.frontend (Ingress)                                              │
│  - OpenAI 兼容 HTTP 服务 (DYN_HTTP_PORT, 默认 8000)                      │
│  - 预处理：prompt 模板、tokenize（可选 vLLM chat processor）              │
│  - 路由：round-robin / random / kv / direct（--router-mode）             │
│  - 通过 Discovery（如 etcd）发现后端 Worker                               │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                     discovery (etcd) + request plane (tcp/nats)
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  dynamo.vllm (Worker)                                                   │
│  - 启动 vLLM AsyncLLM 引擎（加载模型、KV cache 等）                       │
│  - register_model() 向 discovery 注册（namespace.component.endpoint）    │
│  - serve_endpoint(generate, clear_kv_blocks, ...) 处理推理与 cache 清理   │
│  - DYN_SYSTEM_PORT 暴露系统/指标（Prometheus 等）                         │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Frontend** 不持有一份「写死的 worker 列表」，而是通过 **Discovery 后端**（默认 etcd）动态发现已注册的模型实例；vLLM 进程在 `register_vllm_model` → `register_model` 时写入 discovery。
- **Request plane**（默认 tcp）决定 Frontend 与 Worker 之间请求如何投递（tcp 直连或经 NATS 等）。
- **Event plane**（zmq/nats）用于 KV 路由、复制等高级特性；单机 agg 示例可不依赖。

---

## 四、基本运行逻辑（按时间顺序）

1. **启动 Frontend**  
   - 解析 `FrontendConfig`（含 http_port、router_mode、discovery_backend、request_plane 等）。  
   - 创建 `DistributedRuntime(loop, discovery_backend, request_plane, enable_nats)`。  
   - `make_engine(runtime, EntrypointArgs(EngineType.Dynamic, http_port=..., router_config=..., ...))` 在 Rust 里构建「动态引擎」配置（含可选 chat_engine_factory）。  
   - `run_input(runtime, "http", engine)` 在 Rust 里启动 HTTP 监听、连接 discovery 与 request plane，并开始监听已注册的模型实例。

2. **启动 vLLM Worker**  
   - 解析 `Config`（Dynamo 的 namespace/component/endpoint、discovery/request/event plane、use_kv_events 等）和 vLLM 的 `AsyncEngineArgs`。  
   - 若未设置 `--headless`：`create_runtime()` 用与 Frontend 一致的 discovery/request/event 配置创建 `DistributedRuntime`。  
   - `setup_vllm_engine(config)` 创建 vLLM 的 `AsyncLLM`、统计/Prometheus 等。  
   - `init()` 中：  
     - 用 `runtime.endpoint(f"{namespace}.{component}.{endpoint}")` 拿到 generate/clear 等 endpoint。  
     - `register_vllm_model(...)` → 在 Python 侧准备 `ModelRuntimeConfig` 等，再调用 **Rust** 的 `register_model(...)`，把模型信息写入 discovery（etcd 等）。  
     - 对 generate/clear 等调用 `endpoint.serve_endpoint(handler.generate, ...)`，开始处理 Frontend 转发的请求。

3. **一次 chat 请求的路径**  
   - 用户请求发到 Frontend 的 `http://localhost:8000/v1/chat/completions`。  
   - Frontend 的 HTTP 层接收请求，做预处理（模板、tokenize 等，若使用 vllm chat processor）。  
   - Frontend 通过 discovery 查到已注册的 vLLM 模型实例，再经 request plane（tcp）把请求发给对应 Worker 的 endpoint。  
   - Worker 的 `DecodeWorkerHandler.generate` 被调用，内部用 vLLM 的 `AsyncLLM` 做推理，结果经 request plane 回到 Frontend，再以 OpenAI 兼容格式流式/非流式返回给用户。

---

## 五、与示例直接相关的环境变量与默认值

| 变量 / 参数 | 使用方 | 含义 / 默认 |
|-------------|--------|-------------|
| `DYN_HTTP_PORT` | frontend | HTTP 服务端口，默认 8000 |
| `DYN_SYSTEM_PORT` | vllm worker | 系统/指标服务端口，默认 8081；frontend 会主动 unset，避免占用 |
| `DYN_DISCOVERY_BACKEND` | 二者 | 发现后端，默认 `etcd`（单机也可用 `file` 等） |
| `DYN_REQUEST_PLANE` | 二者 | 请求传输方式，默认 `tcp` |
| `DYN_EVENT_PLANE` | 二者 | 事件平面，用于 KV 路由等，frontend 会设置 |

agg.sh 里只显式设置了 `DYN_SYSTEM_PORT`（避免 worker 与 frontend 端口冲突），其余用默认即可跑通单机聚合。

---

## 六、小结

- **示例**：`agg.sh` 启动一个 **dynamo.frontend**（Ingress）和一个 **dynamo.vllm**（Worker），构成最小的「单 GPU 聚合服务」。  
- **源码结构**：Python 逻辑在 **components/src/dynamo/**（frontend、vllm、common 等）；核心运行时、发现、引擎创建在 **lib/** 的 Rust 与 **lib/bindings/python** 的绑定中（`make_engine`、`run_input`、`register_model`）。  
- **架构**：Frontend 负责 HTTP、预处理与路由；Worker 负责 vLLM 推理并注册到 discovery；二者通过 **DistributedRuntime** 的 discovery 与 request plane 协同。  
- **运行逻辑**：Frontend 启动 HTTP 并监听 discovery；Worker 启动 vLLM、注册模型、在 endpoint 上 serve；请求由 Frontend 经 discovery 查找 Worker 并经 request plane 转发到 vLLM 完成推理。

若需要展开某一块（例如 discovery 协议、router_mode=kv 的流程、或 Rust 的 `run_input` 内部），可以指定文件或模块继续深入。
