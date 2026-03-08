# Frontend 启动代码执行路径

本文描述执行 `python -m dynamo.frontend` 后，从进程入口到 HTTP 服务开始监听的完整代码路径。路径中的文件均相对于 **dynamo 子模块根目录**（即仓库中的 `dynamo/` 目录）。

---

## 1. 总览（调用链）

```
python -m dynamo.frontend
  → frontend/__main__.py  main()
  → frontend/main.py      main() → uvloop.run(async_main())
  → frontend/main.py      async_main()
       ├─ parse_args() → FrontendArgGroup / FrontendConfig
       ├─ DistributedRuntime(...)
       ├─ make_engine(runtime, EntrypointArgs(...))   [Python 调用 Rust]
       └─ run_input(runtime, "http", engine)          [Python 调用 Rust]
            → Rust: entrypoint::input::run_input(Input::Http)
            → Rust: entrypoint::input::http::run(...)
                 ├─ HttpService::builder().build()
                 ├─ run_watcher(...)   // 启动 discovery 监听与模型注册
                 └─ http_service.run() // 开始 HTTP 监听，阻塞直到退出
```

---

## 2. Python 侧（按执行顺序）

### 2.1 入口与 main

| 步骤 | 文件路径 | 行号 | 说明 |
|------|----------|------|------|
| 1 | `dynamo/components/src/dynamo/frontend/__main__.py` | 4–7 | `from dynamo.frontend.main import main`；`if __name__ == "__main__": main()` |
| 2 | `dynamo/components/src/dynamo/frontend/main.py` | 259–261 | `def main(): uvloop.run(async_main())` |

### 2.2 async_main() 内逻辑

| 步骤 | 文件路径 | 行号 | 说明 |
|------|----------|------|------|
| 3 | `dynamo/components/src/dynamo/frontend/main.py` | 118–131 | 清除 `DYN_SYSTEM_PORT`，`config, vllm_flags = parse_args()`，`dump_config`，设置 `DYN_EVENT_PLANE` |
| 4 | `dynamo/components/src/dynamo/frontend/main.py` | 169–172 | `loop = asyncio.get_running_loop()`；`runtime = DistributedRuntime(loop, config.discovery_backend, config.request_plane, enable_nats)` |
| 5 | `dynamo/components/src/dynamo/frontend/main.py` | 174–178 | 注册 SIGTERM/SIGINT → `graceful_shutdown(runtime)` |
| 6 | `dynamo/components/src/dynamo/frontend/main.py` | 179–199 | 根据 `config.router_mode` 设置 `RouterMode` 与 `RouterConfig` |
| 7 | `dynamo/components/src/dynamo/frontend/main.py` | 201–224 | 构造 `kwargs`（http_host, http_port, router_config, migration_limit 等），可选 chat_engine_factory |
| 8 | `dynamo/components/src/dynamo/frontend/main.py` | 226–246 | `e = EntrypointArgs(EngineType.Dynamic, **kwargs)`；`engine = await make_engine(runtime, e)` |
| 9 | `dynamo/components/src/dynamo/frontend/main.py` | 241–246 | 默认分支：`await run_input(runtime, "http", engine)`（非 interactive 且非 grpc 时） |

**parse_args 使用的参数定义：**

- `dynamo/components/src/dynamo/frontend/frontend_args.py`：`FrontendArgGroup().add_arguments(parser)`（约 98 行起），包含 `--http-port`（DYN_HTTP_PORT，默认 8000）、`--http-host`、`--router-mode`、`--discovery-backend`、`--request-plane` 等。

**DistributedRuntime / make_engine / run_input 来源：**

- `dynamo.runtime`：在 `dynamo/lib/bindings/python/src/dynamo/runtime/__init__.py` 中从 `dynamo._core` 导出 `DistributedRuntime`。
- `make_engine`、`run_input`：在 `dynamo/lib/bindings/python/src/dynamo/llm/__init__.py` 中从 `dynamo._core` 导出（底层为 Rust 实现）。

---

## 3. Rust 绑定层（Python → Rust）

| 步骤 | 文件路径 | 行号 | 说明 |
|------|----------|------|------|
| 10 | `dynamo/lib/bindings/python/rust/llm/entrypoint.rs` | 284–330 | `make_engine(distributed_runtime, args)`：构建 `LocalModelBuilder`，必要时拉取模型，调用 `select_engine(...)` 得到 `RsEngineConfig`，返回 `EngineConfig { inner }` |
| 11 | `dynamo/lib/bindings/python/rust/llm/entrypoint.rs` | 389–438 | `run_input(distributed_runtime, input, engine_config)`：将 `input` 解析为 `Input` 枚举；`future_into_py` 内调用 `dynamo_llm::entrypoint::input::run_input(drt, input_enum, engine_config.inner)` |

对 **EngineType::Dynamic**（Frontend 默认）：

- `select_engine`（约 389–438 行）返回 `RsEngineConfig::Dynamic { model, chat_engine_factory }`，不在此处启动 HTTP，只构建引擎配置。

---

## 4. Rust 核心：run_input 与 HTTP 分支

| 步骤 | 文件路径 | 行号 | 说明 |
|------|----------|------|------|
| 12 | `dynamo/lib/llm/src/entrypoint/input.rs` | 108–146 | `run_input(drt, in_opt, engine_config)`：根据 `in_opt` 分发；`Input::Http` 时调用 `http::run(drt, engine_config).await` |
| 13 | `dynamo/lib/llm/src/entrypoint/input.rs` | 61–64 | 字符串 `"http"` 解析为 `Input::Http`（`TryFrom<&str>`） |

---

## 5. Rust：HTTP 服务构建与运行

| 步骤 | 文件路径 | 行号 | 说明 |
|------|----------|------|------|
| 14 | `dynamo/lib/llm/src/entrypoint/input/http.rs` | 22–47 | `http::run(distributed_runtime, engine_config)`：从 `engine_config.local_model()` 取端口、TLS、host；`HttpService::builder().port(...).host(...).build()` 等 |
| 15 | `dynamo/lib/llm/src/entrypoint/input/http.rs` | 51–64 | 注入 `drt_metrics`、`drt_discovery`；对 `EngineConfig::Dynamic` 再注入 `discovery` |
| 16 | `dynamo/lib/llm/src/entrypoint/input/http.rs` | 67–97 | **EngineConfig::Dynamic**：`http_service_builder.build()` 得到 `HttpService`；调用 `run_watcher(...)` 启动 discovery 监听与模型注册（`ModelWatcher`、`list_and_watch`），将发现的模型加入 HTTP 服务的 ModelManager |
| 17 | `dynamo/lib/llm/src/entrypoint/input/http.rs` | 151–154 | `http_service.run(distributed_runtime.primary_token()).await`：**在此处开始 HTTP 监听**，阻塞直到收到关闭信号；随后 `distributed_runtime.shutdown()` |

**run_watcher（约 162–207 行）：**

- 创建 `ModelWatcher`，通过 `runtime.discovery().list_and_watch(...)` 监听模型注册/注销。
- 将发现的模型通过 `watch_obj.watch(discovery_stream, namespace_filter)` 注册到 HTTP 服务的 `ModelManager`，并更新可用的 endpoint 类型（chat/completions 等）。

---

## 6. 小结表（按执行顺序）

| 顺序 | 语言 | 文件（相对 dynamo/） | 关键函数/行为 |
|------|------|----------------------|----------------|
| 1 | Python | `components/src/dynamo/frontend/__main__.py` | 入口，调用 `main()` |
| 2 | Python | `components/src/dynamo/frontend/main.py` | `main()` → `uvloop.run(async_main())` |
| 3 | Python | `components/src/dynamo/frontend/main.py` | `async_main()`：parse_args、创建 Runtime、构建 EntrypointArgs、`make_engine`、`run_input(runtime, "http", engine)` |
| 4 | Rust (binding) | `lib/bindings/python/rust/llm/entrypoint.rs` | `make_engine` 构建引擎配置；`run_input` 调用 dynamo_llm::entrypoint::input::run_input |
| 5 | Rust | `lib/llm/src/entrypoint/input.rs` | `run_input` 分发到 `Input::Http` → `http::run` |
| 6 | Rust | `lib/llm/src/entrypoint/input/http.rs` | `http::run`：构建 HttpService、`run_watcher`（discovery 监听）、`http_service.run()` 开始监听 |

**HTTP 实际监听**发生在 **`dynamo/lib/llm/src/entrypoint/input/http.rs`** 中的 **`http_service.run(...)`**（约第 151 行），端口与 host 来自 Frontend 的 `--http-port` / `--http-host`（或环境变量 `DYN_HTTP_PORT` / `DYN_HTTP_HOST`），在 Python 侧通过 `EntrypointArgs` 传入并由 `LocalModel` 提供给 Rust 的 `HttpService::builder()`。
