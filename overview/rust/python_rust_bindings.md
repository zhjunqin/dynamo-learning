# Dynamo 中 Python → Rust 绑定详解

本文结合 Dynamo 代码，说明 Python 与 Rust 的绑定机制、编写规则和典型模式。

---

## 1. 技术栈与构建

### 1.1 核心组件

| 组件 | 作用 |
|------|------|
| **PyO3** | Rust 与 CPython C API 的绑定库，提供 `#[pyclass]`、`#[pymethods]`、`#[pyfunction]`、`#[pymodule]` 等宏 |
| **maturin** | 用 Cargo 构建 Python 扩展的构建后端，产出 `.so` 供 Python 导入 |
| **pyo3-async-runtimes** | 在 PyO3 中桥接 Rust async 与 Python asyncio（如 `future_into_py`、TaskLocals） |
| **pythonize** | Python 对象 ↔ `serde_json::Value` 的序列化/反序列化 |

### 1.2 构建配置

- **Cargo.toml**（`lib/bindings/python/Cargo.toml`）：
  - `[lib]` 中 `name = "_core"`，与 Python 导入名对应。
  - `crate-type = ["cdylib", "rlib"]`：cdylib 生成供 Python 加载的共享库。
  - PyO3 使用 `extension-module`、`abi3-py310`（稳定 ABI，最低 Python 3.10）。

- **pyproject.toml**（`lib/bindings/python/pyproject.toml`）：
  - `[tool.maturin]` 中 `module-name = "dynamo._core"`：安装后 Python 中为 `dynamo._core`。
  - `python-packages = ["dynamo"]`：同时打包纯 Python 包。

- **Python 侧**：通过 `from dynamo._core import ...` 使用；上层包（如 `dynamo.runtime`、`dynamo.llm`）从 `dynamo._core` 再导出，对外暴露稳定 API。

**Python 侧调用示例**（导入与版本）：

```python
# 直接使用扩展模块（版本、底层函数），仅 dynamo._core 有，未在上层重新导出
from dynamo._core import __version__, log_message, get_tool_parser_names

assert __version__[0].isdigit()  # 如 test_bindings_install.py

# 推荐：通过上层包使用，API 稳定
# 以下由 dynamo.llm 重新导出 ← lib/bindings/python/src/dynamo/llm/__init__.py
from dynamo.llm import register_model, make_engine, EngineType, EntrypointArgs
# 以下由 dynamo.runtime 重新导出 ← lib/bindings/python/src/dynamo/runtime/__init__.py
from dynamo.runtime import DistributedRuntime, Endpoint, dynamo_worker
```

**Python 侧总结**

| 包 | 文件路径 | 再导出的内容 |
|----|----------|----------------|
| `dynamo.runtime` | `lib/bindings/python/src/dynamo/runtime/__init__.py` | Client, Context, DistributedRuntime, Endpoint |
| `dynamo.llm` | `lib/bindings/python/src/dynamo/llm/__init__.py` | 引擎/模型/路由相关类、枚举和函数（见上） |
| `dynamo._internal` | `lib/bindings/python/src/dynamo/_internal/__init__.py` | ModelDeploymentCard |

`dynamo._core` 本身是 maturin 从 Rust 编译出的扩展模块（如 `_core.cpython-xxx.so`），安装后位于 `dynamo/_core.*.so`，上述三个 `__init__.py` 通过 `from dynamo._core import ...` 再导出，形成对外使用的稳定 API。

---

## 2. 模块入口：`#[pymodule]`

入口函数名必须与 Cargo.toml 里 `lib.name` 一致（这里是 `_core`），这样 Python 才能正确导入扩展模块。

```rust
// rust/lib.rs
#[pymodule]
fn _core(m: &Bound<'_, PyModule>) -> PyResult<()> {
    // 1. 函数
    m.add_function(wrap_pyfunction!(register_model, m)?)?;
    m.add_function(wrap_pyfunction!(llm::entrypoint::make_engine, m)?)?;

    // 2. 类
    m.add_class::<DistributedRuntime>()?;
    m.add_class::<llm::entrypoint::EntrypointArgs>()?;

    // 3. 子模块（如 prometheus_metrics）
    let prometheus_metrics = PyModule::new(m.py(), "prometheus_metrics")?;
    prometheus_metrics::add_to_module(&prometheus_metrics)?;
    m.add_submodule(&prometheus_metrics)?;

    m.add("__version__", env!("CARGO_PKG_VERSION"))?;
    Ok(())
}
```

**规则要点**：

- 所有要暴露给 Python 的类型和函数都必须在 `_core` 里 `add_class` / `add_function` / `add_submodule` 一次。
- 子模块可封装自己的 `add_to_module(m)`，在 `_core` 里创建 `PyModule` 再 `add_submodule`。

**Python 侧调用示例**（导入即使用，无需显式“打开”模块）：

```python
# 扩展模块由 maturin 安装为 dynamo._core，导入后其属性即 Rust 注册的类/函数
from dynamo._core import __version__
from dynamo.llm import register_model, KvRouterConfig  # 来自 dynamo.llm 对 _core 的再导出
from dynamo.runtime import DistributedRuntime, Endpoint  # 来自 dynamo.runtime 对 _core 的再导出
```

---

## 3. 暴露给 Python 的三种形态

### 3.1 自由函数：`#[pyfunction]`

用于不依赖某类实例的全局函数。

**签名与默认参数**：用 `#[pyo3(signature = (...))]` 写 Python 侧签名和默认值；Rust 参数需与之一一对应，可选参数用 `Option<T>`。

```rust
// rust/lib.rs
#[pyfunction]
#[pyo3(signature = (remote_name, ignore_weights=false))]
fn fetch_model<'p>(
    py: Python<'p>,
    remote_name: &str,
    ignore_weights: bool,
) -> PyResult<Bound<'p, PyAny>> {
    let repo = remote_name.to_string();
    pyo3_async_runtimes::tokio::future_into_py(py, async move {
        LocalModel::fetch(&repo, ignore_weights)
            .await
            .map_err(to_pyerr)
    })
}
```

**规则**：

- 第一个参数可以是 `Python<'p>`，用于拿到 GIL 或做 `future_into_py`。
- 返回 `PyResult<T>`，错误用 `PyErr`（如 `PyValueError::new_err(...)`）。
- 若函数内部是 async，用 `pyo3_async_runtimes::tokio::future_into_py(py, async move { ... })` 把 `Future` 转成 Python 的 awaitable，返回 `Bound<'p, PyAny>`。

**文档签名**（可选）：`#[pyo3(text_signature = "(level, message, module, file, line)")]` 给 `inspect.signature` 用。

**Python 侧调用示例**（自由函数）：

```python
# 同步函数：lora_name_to_id（lib/bindings/python/tests/test_lora_utils.py）
from dynamo.llm import lora_name_to_id
id1 = lora_name_to_id("lora_adapter_1")
id2 = lora_name_to_id("lora_adapter_2")
assert id1 != id2 and 1 <= id1 <= 0x7FFFFFFF

# 带默认参数：log_message（lib/bindings/python/src/dynamo/runtime/logging.py）
from dynamo._core import log_message
log_message(
    record.levelname.lower(),
    log_entry,
    module_path,
    record.pathname,
    record.lineno,
)

# 异步函数：fetch_model，需 await（components/src/dynamo/vllm/main.py）
from dynamo.llm import fetch_model
local_path = await fetch_model(config.model)  # 可选 ignore_weights=False

# 多参数/可选参数：compute_block_hash_for_seq（lib/bindings/python/tests/test_mm_kv_router.py）
from dynamo.llm import compute_block_hash_for_seq
hashes = compute_block_hash_for_seq(tokens, block_size, block_mm_infos=None, lora_name=None)
```

---

### 3.2 类：`#[pyclass]` + `#[pymethods]`

对应 Python 的 class；Rust 里用「内部存 Rust 类型 + 方法转发」的 wrapper 模式。

#### 简单数据类 / 配置类

```rust
// rust/llm/entrypoint.rs
#[pyclass]
#[derive(Default, Clone, Debug, Copy)]
pub struct KvRouterConfig {
    inner: RsKvRouterConfig,
}

#[pymethods]
impl KvRouterConfig {
    #[new]
    #[pyo3(signature = (overlap_score_weight=1.0, router_temperature=0.0, ...))]
    fn new(
        overlap_score_weight: f64,
        router_temperature: f64,
        // ...
    ) -> Self {
        KvRouterConfig {
            inner: RsKvRouterConfig { ... },
        }
    }
}
```

- `#[new]`：Python 的 `__init__`，对应构造函数。
- 若需在 Rust 其它地方用「真正的类型」，可再实现 `From<PyType> for RsType`，在绑定层只暴露 Py 类型，内部转成 `RsKvRouterConfig`。

#### 带 getter/setter 的类

```rust
#[pyclass]
pub struct RouterConfig {
    #[pyo3(get, set)]
    pub router_mode: RouterMode,
    #[pyo3(get, set)]
    pub kv_router_config: KvRouterConfig,
    active_decode_blocks_threshold: Option<f64>,
    // 无 get/set 的字段在 Python 侧不可见
}
```

- `#[pyo3(get)]` / `#[pyo3(get, set)]` 会生成 Python 的 property。

#### 枚举（暴露给 Python 的枚举）

```rust
#[pyclass(eq, eq_int)]
#[derive(Clone, Debug, PartialEq)]
#[repr(i32)]
pub enum EngineType {
    Echo = 1,
    Dynamic = 2,
    Mocker = 3,
}
```

- `eq_int` 允许与整数比较；`repr(i32)` 保证和 C/Python 互操作时的数值一致。

#### 类属性（常量）

```rust
#[pymethods]
impl ModelType {
    #[classattr]
    const Chat: Self = ModelType { inner: llm_rs::model_type::ModelType::Chat };
    #[classattr]
    const Prefill: Self = ...;
}
```

#### 方法命名与重载

- 用 `#[pyo3(signature = (...))]` 写默认参数。
- 用 `#[pyo3(name = "to_capsule")]` 等改 Python 侧方法名；用 `#[pyo3(name = "__repr__")]` 实现双下划线方法。

**规则小结**：

- Py 类通常持有一个 `inner: RsType`，所有逻辑在 Rust 里实现，Py 只做薄封装。
- 需要从 Py 传到纯 Rust 时，在绑定层实现 `From<PyType> for RsType`。
- 若类型要在多个 crate 间共享（如 `DistributedRuntime`），保持只有一个 `#[pyclass]` 定义，避免 PyO3 类型不匹配。

**Python 侧调用示例**（类与枚举）：

```python
# 创建 Runtime、Endpoint（lib/bindings/python/examples/cli/cli.py）
from dynamo.runtime import DistributedRuntime
loop = asyncio.get_running_loop()
runtime = DistributedRuntime(loop, "etcd", "nats")
endpoint = runtime.endpoint("test.backend.generate")

# 枚举：EngineType、ModelType、ModelInput、RouterMode（tests/serve/launch/template_verifier.py）
from dynamo.llm import ModelInput, ModelType, register_model
await register_model(
    ModelInput.Tokens,
    ModelType.Chat,
    endpoint,
    model_name,
    model_name=model_name,
    custom_template_path=str(template_path),
)

# 配置类：KvRouterConfig、RouterConfig（tests/router/common.py）
from dynamo.llm import KvRouterConfig, KvRouter
kv_router_config = KvRouterConfig()  # 全默认
# 或带参数
kv_router_config = KvRouterConfig(
    router_snapshot_threshold=20,
    durable_kv_events=durable_kv_events,
)

# EntrypointArgs：构造并传给 make_engine（components/src/dynamo/frontend/main.py）
from dynamo.llm import EngineType, EntrypointArgs, make_engine, run_input
e = EntrypointArgs(EngineType.Dynamic, **kwargs)
engine = await make_engine(runtime, e)

# 类属性：ModelType.Chat、ModelType.Prefill 等（见 dynamo.llm 导出）
assert ModelType.Chat.supports_chat()
```

---

### 3.3 异步方法

Dynamo 里大量是「Python 调 Rust async」：在持有 GIL 的上下文中用 `pyo3_async_runtimes::tokio::future_into_py` 把 Rust 的 `Future` 转成 Python 的 coroutine，这样 Python 端可以 `await`。

```rust
// 示例：rust/lib.rs 中的 register_model
pyo3_async_runtimes::tokio::future_into_py(py, async move {
    LocalModel::fetch(&source_path, false).await.map_err(to_pyerr)?;
    // ...
    local_model.attach(&endpoint.inner, ...).await.map_err(to_pyerr)?;
    Ok(())
})
```

**规则**：

- 在 `async move { }` 里不要长期持有 `Python` 或 `Bound`，否则会占着 GIL 阻塞事件循环。
- 错误统一用项目里的 `to_pyerr(E)` 转成 `PyErr`，保持错误在 Python 侧一致。

**Python 侧调用示例**（await 绑定返回的协程）：

```python
# register_model、make_engine、run_input 均返回 awaitable（tests/serve/launch/template_verifier.py）
await register_model(
    ModelInput.Tokens,
    ModelType.Chat,
    endpoint,
    model_name,
    model_name=model_name,
    custom_template_path=str(template_path),
)
await endpoint.serve_endpoint(handler.generate)

# cli 示例：make_engine + run_input（lib/bindings/python/examples/cli/cli.py）
e = EntrypointArgs(engine_type, **entrypoint_kwargs)
engine = await make_engine(runtime, e)
await run_input(runtime, args["in_mode"], engine)

# endpoint.client()、client.generate/round_robin/direct（lib/bindings/python/examples/hello_world/client.py）
client = await endpoint.client()
await client.wait_for_instances()
stream = await client.generate("hello world")
async for char in stream:
    print(char)

# round_robin / random（lib/bindings/python/examples/pipeline/frontend.py）
async for output in await self.next.round_robin(request):
    yield output.get("data")
```

---

## 4. Python 回调到 Rust（callback 与 TaskLocals）

Rust 侧有时需要「在异步任务里调用 Python 的 async 函数」。Dynamo 的做法是：

1. 在**注册时**（Python 事件循环已跑）用 `pyo3_async_runtimes::tokio::get_current_locals(py)` 拿到当前 TaskLocals。
2. 把「Python 可调用对象 + TaskLocals」一起存起来（例如放在 `PyEngineFactory` 或 `EngineRouteCallback` 的闭包里）。
3. 在 Rust 的 async 路径里需要调 Python 时：
   - 用 `Python::with_gil` 调 Python，得到 coroutine；
   - 用 `pyo3_async_runtimes::into_future_with_locals(&locals, coroutine)` 把 coroutine 转成 Rust 的 `Future`；
   - 在 Rust 里 `.await` 这个 future（此时会释放 GIL），最后再把结果转回 Rust 类型。

这样既不会在错误的线程/事件循环上跑 Python，也不会长期占 GIL。

**示例（chat_engine_factory）**：

```rust
// rust/llm/entrypoint.rs
struct PyEngineFactory {
    callback: Arc<PyObject>,
    locals: Arc<TaskLocals>,
}

// 注册时捕获 TaskLocals
let chat_engine_factory = chat_engine_factory.map(|callback| {
    let locals = pyo3_async_runtimes::tokio::get_current_locals(py).map_err(|e| ...)?;
    Ok(PyEngineFactory { callback: Arc::new(callback), locals: Arc::new(locals) })
}).transpose()?;

// 在 Rust 里调用 Python async 时
let py_future = Python::with_gil(|py| {
    let coroutine = callback.call1(py, (py_instance_id, py_card_obj))?;
    pyo3_async_runtimes::into_future_with_locals(&locals, coroutine.into_bound(py))
})?;
let py_result = py_future.await?;
```

**规则**：

- 凡「Rust 在任意时刻可能调用的 Python async」都要在**注册时**绑好 TaskLocals，不能假设调用时一定在 Python 主线程。
- 用 `call1`/`call_method` 等传参时，类型需实现 PyO3 的 `IntoPy`/可从 Python 提取；复杂结构可用 `pythonize`/`depythonize` 过一遍 `serde_json::Value`。

**Python 侧调用示例**（向 Rust 注册 Python 回调）：

```python
# 1. register_engine_route：注册 async (body: dict) -> dict，供 HTTP /engine/<route_name> 调用
# components/src/dynamo/vllm/main.py
runtime.register_engine_route("sleep", handler.sleep)
runtime.register_engine_route("wake_up", handler.wake_up)

# components/src/dynamo/sglang/request_handlers/handler_base.py
runtime.register_engine_route("start_profile", self.start_profile)
runtime.register_engine_route("stop_profile", self.stop_profile)

# 2. chat_engine_factory：传给 EntrypointArgs，Rust 在发现模型时调用，返回 PythonAsyncEngine
# components/src/dynamo/frontend/main.py
chat_engine_factory = setup_engine_factory(...).chat_engine_factory
kwargs["chat_engine_factory"] = chat_engine_factory
e = EntrypointArgs(EngineType.Dynamic, **kwargs)

# 回调签名（components/src/dynamo/frontend/vllm_processor.py）
from dynamo._internal import ModelDeploymentCard
from dynamo.llm import ModelCardInstanceId, PythonAsyncEngine

async def chat_engine_factory(
    self,
    instance_id: ModelCardInstanceId,
    mdc: ModelDeploymentCard,
) -> PythonAsyncEngine:
    model_type = mdc.model_type()
    source_path = mdc.source_path()
    # ... 用 mdc 构造并返回 PythonAsyncEngine
    return engine
```

---

## 5. 类型与错误转换

### 5.1 Rust 错误 → Python 异常

项目里统一用：

```rust
pub fn to_pyerr<E>(err: E) -> PyErr
where
    E: Display,
{
    PyException::new_err(format!("{}", err))
}
```

在 `async` 里：`.map_err(to_pyerr)?`；在需要更具体异常时用 `PyValueError::new_err(...)`、`PyRuntimeError::new_err(...)` 等。

### 5.2 Python 对象 ↔ Rust 数据

- **简单类型**：PyO3 自动在 `&str`、`String`、`i64`、`bool`、`Option<T>` 等之间转换。
- **复杂结构**：用 `pythonize::depythonize(dict)` 得到 `serde_json::Value` 或具体 Rust 类型；用 `pythonize::pythonize(py, &value)` 得到 `PyObject`，这样 Python 端收到的是 dict/list 等。

### 5.3 在绑定层隐藏内部类型

- 只给 Python 用的类型可标 `pub(crate)`，避免泄漏到其它 crate。
- 内部 Rust 类型命名为 `RsXxx`（如 `RsRouterConfig`），绑定层类型命名为 `Xxx`（如 `RouterConfig`），在绑定层实现 `From<Xxx> for RsXxx`，这样纯 Rust 代码只依赖 `RsXxx`。

**Python 侧调用示例**（错误与复杂类型）：

```python
# 错误：Rust 里 to_pyerr / PyValueError 等在 Python 侧变为普通异常
try:
    await register_model(...)
except Exception as e:
    print(e)  # 即 Rust 里 format!("{}", err) 的字符串

# 复杂数据：Client.generate / round_robin 等接收 dict（Python 侧），Rust 用 pythonize 转成 Value
# lib/bindings/python/examples/typed/client.py
stream = await client.generate(Request(data="hello world").model_dump_json())

# 内部类型：ModelDeploymentCard 仅从 dynamo._internal 导出，供 chat_engine_factory 等使用
# components/src/dynamo/frontend/vllm_processor.py
from dynamo._internal import ModelDeploymentCard
source_path = mdc.source_path()
model_type = mdc.model_type()
```

---

## 6. 文件与分层约定（结合 Dynamo）

- **入口**：`rust/lib.rs` — 只做 `_core` 的 `#[pymodule]`、`add_*` 和少量直接在根模块的 `#[pyclass]`/`#[pyfunction]`（如 `DistributedRuntime`、`register_model`）。
- **按领域分子模块**：如 `rust/llm/entrypoint.rs`、`rust/llm/kv.rs`、`rust/engine.rs`、`rust/planner.rs`，各自定义本领域的 Py 类和函数，在 `lib.rs` 里 `add_class`/`add_function`。
- **子模块入口**：像 `prometheus_metrics` 那样在 `lib.rs` 里 `PyModule::new` + `add_to_module` + `add_submodule`，保持命名空间清晰。
- **Python 包**：`dynamo._core` 是扩展模块；`dynamo.runtime`、`dynamo.llm` 等从 `dynamo._core` 做 `from dynamo._core import ...` 再导出，对外只暴露稳定 API；内部实现细节可放在 `dynamo._internal`。

**Python 侧调用示例**（分层与子模块）：

```python
# 推荐入口：dynamo.runtime / dynamo.llm（稳定 API）
from dynamo.runtime import DistributedRuntime, dynamo_worker, Endpoint, Client, Context
from dynamo.llm import (
    register_model, make_engine, fetch_model,
    EngineType, EntrypointArgs, KvRouterConfig, RouterConfig, RouterMode,
    ModelType, ModelInput, PythonAsyncEngine, WorkerMetricsPublisher,
)

# 内部/实现用：dynamo._internal（如 ModelDeploymentCard）
from dynamo._internal import ModelDeploymentCard

# 底层/测试用：直接 dynamo._core（版本、parser 名等）
from dynamo._core import __version__, get_tool_parser_names, get_reasoning_parser_names

# WorkerMetricsPublisher（绑定类，由组件封装使用）（components/src/dynamo/vllm/publisher.py）
from dynamo.llm import WorkerMetricsPublisher
from dynamo.runtime import Endpoint
pub = WorkerMetricsPublisher()
await pub.create_endpoint(endpoint)
pub.publish(dp_rank=0, active_decode_blocks=42)
```

---

## 7. 小结：编写规则速查

| 需求 | 做法 |
|------|------|
| 暴露一个函数 | `#[pyfunction]` + `#[pyo3(signature = (...))]`，在 `_core` 里 `m.add_function(wrap_pyfunction!(fn_name, m)?)` |
| 暴露一个类 | `#[pyclass]` 在 struct/enum 上，`#[pymethods] impl` 里写 `#[new]`、方法、`#[classattr]` 等，再 `m.add_class::<T>()` |
| 默认参数 | 在 `#[pyo3(signature = (a, b=1, c=None))]` 里写，Rust 用 `Option` 等对应 |
| 异步函数/方法 | 用 `pyo3_async_runtimes::tokio::future_into_py(py, async move { ... })` 返回 awaitable |
| Python 回调到 Rust async | 注册时 `get_current_locals(py)` 存起来，调用时 `into_future_with_locals` 把 coroutine 转成 Future 再 await |
| 错误 | 用 `to_pyerr` 或 `PyValueError`/`PyRuntimeError` 等 |
| 复杂数据 | `pythonize`/`depythonize` 经 `serde_json::Value` 或具体类型 |
| 内部 Rust 类型 | 绑定层用 wrapper 持 `inner: RsType`，实现 `From<Wrapper> for RsType` 供纯 Rust 使用 |

按上述规则，即可在 Dynamo 中一致地扩展和维护 Python → Rust 的绑定。
