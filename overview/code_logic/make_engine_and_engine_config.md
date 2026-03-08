# make_engine 完成什么、与 runtime 的关系、engine 里包含与管理什么

本文结合代码说明 **make_engine** 的职责、与 **DistributedRuntime（runtime）** 的关系，以及 **EngineConfig（engine）** 的内容与所管理的资源。代码路径均相对 **dynamo 子模块根目录**（`dynamo/`）。

---

## 1. make_engine 完成什么功能

**位置**：`dynamo/lib/bindings/python/rust/llm/entrypoint.rs` 第 284–331 行。

**签名**（Python 侧）：`async def make_engine(distributed_runtime: DistributedRuntime, args: EntrypointArgs) -> EngineConfig`  
即：输入 **runtime** 和 **EntrypointArgs**，输出一个 **EngineConfig**（在 Python 里常被称作 engine）。

**做的三件事**：

1. **用 EntrypointArgs 填 LocalModelBuilder，必要时拉取模型**  
   - 把 `model_name`、`endpoint_id`、`context_length`、`request_template`、`kv_cache_block_size`、`router_config`、`migration_limit`、`http_host`/`http_port`、TLS、`namespace`/`namespace_prefix`、`runtime_config` 等写入 builder。  
   - 若提供了 `model_path` 且路径不存在，则调用 `LocalModel::fetch(...)` 从 HuggingFace 下载（Mocker 可设 `ignore_weights`）。  
   - 最后 `builder.build().await` 得到 **LocalModel**。

2. **按引擎类型选具体引擎配置（select_engine）**  
   - **Echo**：`RsEngineConfig::InProcessText { model, engine: make_echo_engine() }`。  
   - **Dynamic**（Frontend 默认）：`RsEngineConfig::Dynamic { model, chat_engine_factory }`，不在此处创建实际推理引擎，只保存「模型元数据 + 可选的回调工厂」。  
   - **Mocker**：`make_mocker_engine(drt, endpoint, mocker_args)` 得到真实引擎，再包成 `RsEngineConfig::InProcessTokens { engine, model, is_prefill }`。

3. **返回 EngineConfig**  
   - 将上面得到的 `RsEngineConfig` 包成 Python 可见的 `EngineConfig { inner }` 并返回。  
   - **不**启动 HTTP、不启动 discovery 监听、不持有 runtime 的引用；只是**配置对象**的构造。

因此：**make_engine 的职责 = 根据参数和（可选）模型路径，构造「如何对外提供/如何发现模型」的配置（EngineConfig），并在 Dynamic 模式下为后续「按需创建 chat 引擎」准备好 LocalModel 和可选的 chat_engine_factory。** 它不负责把服务跑起来，跑起来的是 **run_input(runtime, "http", engine)**。

---

## 2. make_engine 与 runtime 之间的关系

- **入参**：`make_engine(distributed_runtime, args)` 会**接收** runtime，但只在一处用到：  
  - **Mocker** 分支里 `make_mocker_engine(distributed_runtime.inner, endpoint, mocker_args)`，用于在 Rust 里创建 mocker 引擎（可能用到 runtime 的 discovery 等）。  
  - **Echo / Dynamic** 分支里，`select_engine` **没有**把 runtime 存进返回的 `EngineConfig`。

- **返回值**：返回的 `EngineConfig` 在 Python/Rust 的 **EngineConfig** 定义里都**不包含** `DistributedRuntime` 字段。  
  - 也就是说，**engine 本身不持有 runtime**。  
  - 二者是**分开传**的：`run_input(runtime, "http", engine)` 同时接收 runtime 和 engine；在 `http::run(drt, engine_config)` 里才把「runtime 的能力」和「engine 的配置」一起用。

- **协作方式**：  
  - **runtime**：提供 discovery、request plane、metrics、cancel token、network manager 等**进程/集群级资源**。  
  - **engine（EngineConfig）**：提供**本「前端/入口」要用的模型元数据、路由配置、HTTP 端口、以及（Dynamic 时）按实例创建 chat 引擎的回调**。  
  - **run_input** 把两者结合：例如在 HTTP 模式下用 runtime 的 discovery 做 list_and_watch，用 engine 的 local_model 的 http_port/router_config 建 HttpService，用 engine 的 chat_engine_factory 在发现新模型实例时创建真实引擎。

所以：**make_engine 与 runtime 的关系是「make_engine 只在使用 runtime 构造 Mocker 时依赖 runtime」；返回的 engine 与 runtime 是并列的两类对象，在 run_input 里才一起使用，而不是 engine 包含或管理 runtime。**

---

## 3. Engine（EngineConfig）里面包含了什么

**Rust 定义**：`dynamo/lib/llm/src/entrypoint.rs` 第 65–95 行。

```rust
pub enum EngineConfig {
    /// Remote networked engines that we discover via etcd
    Dynamic {
        model: Box<LocalModel>,
        chat_engine_factory: Option<ChatEngineFactoryCallback>,
    },
    InProcessText {
        engine: Arc<dyn StreamingEngine>,
        model: Box<LocalModel>,
    },
    InProcessTokens {
        engine: ExecutionContext,  // 实际执行请求的引擎
        model: Box<LocalModel>,
        is_prefill: bool,
    },
}
```

**共同点**：三种变体都包含 **`model: Box<LocalModel>`**。  
**统一访问**：`engine_config.local_model()` 恒返回这个 `LocalModel`（entrypoint.rs 86–94 行）。

因此 **engine 里「包含」的**可以分成两块：

- **LocalModel**（三种变体共有）：模型/部署的元数据与配置，见下一节。  
- **按变体额外有的**：  
  - **Dynamic**：可选的 **chat_engine_factory**（按 `ModelCardInstanceId` + `ModelDeploymentCard` 创建 `OpenAIChatCompletionsStreamingEngine` 的回调）。  
  - **InProcessText**：**engine: Arc<dyn StreamingEngine>**（例如 echo engine）。  
  - **InProcessTokens**：**engine: ExecutionContext**（如 mocker）、**is_prefill: bool**。

**EngineConfig 不包含**：  
- 不包含 `DistributedRuntime`；  
- 不包含 HTTP server、discovery 客户端、NATS 等；这些都在 runtime 侧，由 `run_input` 把 runtime 和 engine_config 一起传给 `http::run` 等。

---

## 4. LocalModel 包含与管理什么

**定义**：`dynamo/lib/llm/src/local_model.rs` 第 321–327 行。

```rust
pub struct LocalModel {
    full_path: PathBuf,           // 本地模型目录路径（可为空，如 frontend-only）
    card: ModelDeploymentCard,    // 模型部署卡：名称、类型、上下文长度、KV block 等
    endpoint_id: EndpointId,
    template: Option<RequestTemplate>,
    http_host: Option<String>,
    http_port: u16,
    http_metrics_port: Option<u16>,
    tls_cert_path: Option<PathBuf>,
    tls_key_path: Option<PathBuf>,
    router_config: RouterConfig,
    runtime_config: ModelRuntimeConfig,
    namespace: Option<String>,
    namespace_prefix: Option<String>,
    migration_limit: u32,
}
```

- **ModelDeploymentCard**（model_card.rs）：`display_name`、slug、model_type、model_input、context_length、kv_cache_block_size、tokenizer 相关、source_path、lora 信息、runtime_config 等；用于 discovery 注册和 HTTP 层识别模型。  
- **RouterConfig**：路由模式（round-robin / random / kv / direct）、KV 路由参数、负载阈值、enforce_disagg 等；在 Dynamic 模式下被 `http::run` 用来建 ModelWatcher 和路由逻辑。  
- **HTTP/TLS**：`http_host`、`http_port`、`http_metrics_port`、`tls_*`；在 `http::run` 里用于 `HttpService::builder().port(...).host(...)` 等。  
- **namespace / namespace_prefix**：发现时的命名空间过滤。  
- **migration_limit**：请求迁移上限，交给 watcher/路由使用。

因此：**LocalModel 管理的是「单模型」的部署与接入配置（卡信息、HTTP、路由、命名空间、迁移限制等）**，不管理进程级或集群级资源；**EngineConfig 通过持有 LocalModel（以及可选的 engine/chat_engine_factory）来「管理」这些配置，并在 run_input 时被用来驱动 HTTP/grpc/text 等输入侧如何建服务和如何按需创建引擎。**

---

## 5. run_input 如何把 runtime 和 engine 用在一起

**入口**：`dynamo/lib/llm/src/entrypoint/input.rs` 第 108–146 行。

- `run_input(drt, in_opt, engine_config)` 根据 `in_opt`（Http / Grpc / Text / Stdin / Batch / Endpoint）调用不同模块的 `run`。  
- 以 **Input::Http** 为例：`http::run(drt, engine_config).await`（input.rs 124–126）。

在 **http::run**（`dynamo/lib/llm/src/entrypoint/input/http.rs`）里：

- 用 **engine_config.local_model()** 取端口、host、TLS、request_template，构建 HttpService。  
- 用 **drt**（runtime）的 `primary_token()`、`get_metrics_registry()`、`discovery()` 注入取消、指标、发现。  
- 对 **EngineConfig::Dynamic**：  
  - 用 **model.router_config()**、**model.migration_limit()**、**model.namespace()/namespace_prefix()** 调用 `run_watcher(drt, ..., router_config, migration_limit, namespace_filter, ..., chat_engine_factory)`。  
  - watcher 用 **drt.discovery().list_and_watch(...)** 监听注册，发现新实例时用 **chat_engine_factory** 创建真实 chat 引擎并加入 HTTP 的 ModelManager。  
- 最后 **http_service.run(drt.primary_token()).await** 真正开始 HTTP 监听并阻塞。

可以看到：**runtime 提供「发现、网络、取消、指标」等能力；engine（EngineConfig）提供「用哪个端口、用什么路由、按什么规则发现模型、以及如何为每个实例创建引擎」**。两者在 run_input → http::run 里配合，而不是 engine 包含或管理 runtime。

---

## 6. 小结表

| 问题 | 结论 |
|------|------|
| **make_engine 完成什么** | 根据 EntrypointArgs 构建 LocalModel（必要时拉取模型），再按引擎类型（Echo/Dynamic/Mocker）得到 RsEngineConfig，包装成 EngineConfig 返回；不启动服务。 |
| **和 runtime 的关系** | make_engine 仅在 Mocker 分支里用 runtime 建 mocker 引擎；返回的 engine 不持有 runtime；两者在 run_input(runtime, input, engine) 里一起使用。 |
| **engine 里面包含什么** | 所有变体都有 **LocalModel**；Dynamic 另有 **chat_engine_factory**；InProcessText 有 **StreamingEngine**；InProcessTokens 有 **ExecutionContext** 和 **is_prefill**。 |
| **engine 管理什么** | 通过 LocalModel 管理「单模型」的部署与接入配置（ModelDeploymentCard、HTTP/TLS、RouterConfig、namespace、migration_limit）；通过 chat_engine_factory 或内联 engine 管理「如何得到可执行的 chat/completions 引擎」。不管理 runtime、discovery、请求平面等进程/集群资源。 |

简记：**make_engine = 造配置（EngineConfig）；engine = 模型配置 + 可选引擎/工厂；runtime = 集群/进程资源。run_input 把 runtime 和 engine 组合起来，真正启动 HTTP/grpc/text 等服务。**
