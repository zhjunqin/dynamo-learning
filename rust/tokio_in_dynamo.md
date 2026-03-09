# Dynamo 中 Tokio 的使用总结

本文结合 `dynamo` 代码，梳理 Tokio 在仓库中的主要使用场景、常见写法和可直接参考的示例。

---

## 1. 总览

在 `dynamo` 中，Tokio 不是“某个模块顺手用了 async”，而是整个运行时层的基础设施，主要承担这几类职责：

| 类别 | 作用 | 典型位置 |
|------|------|----------|
| 运行时管理 | 创建/复用 Tokio runtime，区分主执行线程池与后台线程池 | `dynamo/lib/runtime/src/runtime.rs`、`dynamo/lib/runtime/src/worker.rs` |
| 后台任务 | 启动 signal handler、watcher、网络服务、状态服务等异步任务 | `dynamo/lib/runtime/src/worker.rs`、`dynamo/lib/runtime/src/utils/typed_prefix_watcher.rs` |
| 取消与优雅关闭 | 用 `CancellationToken`、`tokio::select!` 协调退出、超时和任务结束 | `dynamo/lib/runtime/src/runtime.rs`、`dynamo/lib/runtime/src/worker.rs`、`dynamo/lib/runtime/src/utils/tasks/tracker.rs` |
| 并发控制 | 用 `Semaphore` 控制并发量，用 `spawn_blocking` 隔离阻塞/CPU 密集工作 | `dynamo/lib/runtime/src/runtime.rs`、`dynamo/lib/runtime/src/utils/tasks/tracker.rs` |
| 状态同步与通信 | 用 `mpsc`、`oneshot`、`watch`、`Notify` 在任务间传递消息、状态和事件 | `dynamo/lib/runtime/src/utils/typed_prefix_watcher.rs`、`dynamo/lib/llm/src/discovery/watcher.rs`、`dynamo/lib/runtime/tests/common/mock.rs` |
| 测试与示例 | 用 `#[tokio::test]` 驱动异步单测和流式 pipeline 测试 | `dynamo/lib/runtime/tests/pipeline.rs`、`dynamo/lib/runtime/tests/pool.rs` |

另外，workspace 里统一依赖了：

- `tokio = 1.48.0`，特性为 `full`
- `tokio-util = 0.7.x`
- `tokio-stream = 0.1.x`

因此代码里可以直接使用大多数 Tokio 核心能力，而不需要每个 crate 单独拼 feature。

---

## 2. 在 Dynamo 中的主要使用场景

### 2.1 作为统一异步运行时

`dynamo/lib/runtime/src/runtime.rs` 中的 `Runtime` 封装了 Tokio runtime，并区分：

- `primary`：业务主执行 runtime
- `secondary`：后台任务 runtime，例如 etcd/NATS 等后台工作

这个封装不是简单持有 `tokio::runtime::Runtime`，还额外管理：

- `CancellationToken`
- 优雅关闭 tracker
- 计算池与 `spawn_blocking` 协作
- 在异步上下文中安全关闭 runtime

这意味着在 `dynamo` 里，Tokio runtime 是“平台资源”，不是随手在业务代码里临时 new 一个 runtime。

### 2.2 启动后台任务

仓库中大量后台逻辑通过 `tokio::spawn` 启动，例如：

- `Worker` 启动 signal handler
- etcd prefix watcher 持续接收 watch 事件
- 网络 ingress/egress、系统状态服务、健康检查等后台任务

适合 `tokio::spawn` 的任务通常满足：

- 本身是异步循环
- 需要长期驻留
- 由取消令牌或 channel 事件驱动退出

### 2.3 取消、优雅关闭与超时控制

这是 `dynamo` 中 Tokio 使用最集中的场景之一。

常见组合是：

- `tokio_util::sync::CancellationToken`
- `tokio::select!`
- `tokio::time::sleep` / `timeout`

例如 `Worker::execute_internal()` 会同时等待：

- 应用完成
- 取消信号触发
- 优雅关闭超时

`Runtime::shutdown()` 则分阶段执行：

1. 先取消 endpoint shutdown token，停止接新请求
2. 等待 graceful tracker 中的任务完成
3. 最后再取消主 token，断开后端连接

这类写法非常适合分布式服务，因为它把“停止接收流量”和“真正断链退出”分开了。

### 2.4 任务间通信与状态广播

`dynamo` 里没有把所有状态都塞进 `Arc<Mutex<T>>`，而是按场景选不同 Tokio 同步原语：

- `mpsc`：流式发送消息或事件
- `oneshot`：一次性完成通知
- `watch`：广播“最新状态”
- `Notify`：只关心“有事件发生了”，不关心具体数据

具体对应关系：

- `TypedPrefixWatcher` 用 `watch::channel(HashMap<K, V>)` 广播当前 collated state
- `ModelWatcher` 用 `Notify` 等待“至少有一个模型可用”
- mock network 测试用 `mpsc + oneshot` 模拟控制面/数据面交互

### 2.5 并发限制与背压

`dynamo/lib/runtime/src/utils/tasks/tracker.rs` 中的 `SemaphoreScheduler` 用 `tokio::sync::Semaphore` 限制任务并发数，并在等待 permit 时配合取消令牌：

- 等 permit
- 或收到取消

二者谁先发生，就走哪条路径。

这是 `tokio::select!` 和 `Semaphore` 很典型的配合方式，适合做：

- 限流
- 资源保护
- 避免过多并行任务压垮下游

### 2.6 隔离阻塞或 CPU 密集工作

虽然 `dynamo` 以异步为主，但并不把所有工作都塞进 async 任务。

`runtime.rs` 中会用 `tokio::task::spawn_blocking`：

- 探测 worker thread 数量
- 在线程上初始化 thread-local compute context

这说明仓库里的原则是：

- IO 等待任务放 Tokio async runtime
- 阻塞或 CPU 密集步骤放 `spawn_blocking` 或专门的 compute pool

这能避免把 Tokio worker thread 卡死。

### 2.7 异步测试

大量测试使用 `#[tokio::test]`，例如：

- pipeline 流式输出测试
- soak test
- pool / tracker 测试

这样测试可以直接：

- `await` 异步接口
- 验证 stream 行为
- 配合 `sleep`、`timeout`、`spawn` 测试并发场景

---

## 3. Dynamo 中常见的 Tokio 用法

### 3.1 `tokio::spawn`

适用场景：

- 后台守护任务
- watcher
- 服务端子任务
- 不需要阻塞当前 async 函数的并发工作

仓库里的典型模式：

```rust
tokio::spawn(async move {
    loop {
        // 等待事件、处理状态、响应取消
    }
});
```

注意：

- 被 `spawn` 的 future 必须是 `Send + 'static`
- 若任务需要退出条件，最好结合 `CancellationToken`
- 若任务结果需要回收，不要只 `spawn` 不保存 `JoinHandle`

### 3.2 `tokio::select!`

`select!` 是本仓库里最重要的 Tokio 宏之一，常用于同时等待多个异步事件：

- 取消
- 超时
- channel 收包
- 子任务结束
- 信号

常见写法：

```rust
tokio::select! {
    _ = cancel_token.cancelled() => {
        // 收到取消
    }
    msg = rx.recv() => {
        // 收到消息
    }
    _ = tokio::time::sleep(duration) => {
        // 超时
    }
}
```

### 3.3 `tokio::sync::{mpsc, oneshot, watch, Notify}`

建议按语义选型：

| 原语 | 适合场景 | Dynamo 中示例 |
|------|----------|---------------|
| `mpsc` | 多次消息传递、生产者到消费者 | mock network 控制面和数据面 |
| `oneshot` | 单次完成通知、任务结束通知 | 流结束通知、一次性回执 |
| `watch` | “我只关心最新状态” | etcd prefix collated state |
| `Notify` | “有变化了，快醒一下” | 等待模型出现 |

### 3.4 `tokio::sync::Semaphore`

适合限制并发度，例如：

- 最多同时处理 N 个任务
- 控制某类资源访问上限
- 防止瞬时任务风暴

在 `SemaphoreScheduler` 中，Dynamo 的处理方式是：

- 先检查取消
- 再在 `select!` 中等待 permit 或取消
- 成功拿到 permit 后，把 permit 生命周期绑定到执行阶段

这种写法比“先 await permit，再单独检查取消”更稳妥。

### 3.5 `tokio::task::spawn_blocking`

适用场景：

- 阻塞系统调用
- CPU 密集逻辑
- 需要占用线程较久的同步代码

不建议把这类工作直接放进普通 async task，否则会拖慢整个 runtime。

### 3.6 `tokio::time`

仓库中常见用法：

- `sleep`：模拟延迟、轮询等待、优雅关闭超时
- `timeout`：为流、请求或任务包一层时限

常见场景：

- pipeline 测试里控制 token 逐步输出
- soak test 中限制等待时间
- worker 优雅关闭超时

### 3.7 `#[tokio::test]`

异步测试里最直接的方式就是：

```rust
#[tokio::test]
async fn test_xxx() {
    // 直接 await
}
```

对于 `dynamo` 这种 heavily async 的代码库，这种测试方式比手动创建 runtime 更自然，也更接近真实执行路径。

---

## 4. 推荐的写法模式

### 4.1 后台 watcher 模式

适用于 etcd、配置、服务发现等持续监听场景。

```rust
use tokio::sync::watch;
use tokio_util::sync::CancellationToken;

let (tx, rx) = watch::channel(State::default());

tokio::spawn(async move {
    let mut state = State::default();
    loop {
        tokio::select! {
            _ = cancel_token.cancelled() => break,
            event = events_rx.recv() => {
                let Some(event) = event else { break; };
                state.apply(event);
                let _ = tx.send(state.clone());
            }
        }
    }
});
```

适合 `dynamo/lib/runtime/src/utils/typed_prefix_watcher.rs` 这种“后台维护一份最新状态快照”的场景。

### 4.2 优雅关闭模式

适用于服务主循环、后台 worker、需要区分取消和正常结束的任务。

```rust
tokio::select! {
    _ = cancel_token.cancelled() => {
        tracing::info!("shutdown requested");
    }
    result = task => {
        return result;
    }
    _ = tokio::time::sleep(timeout) => {
        anyhow::bail!("graceful shutdown timed out");
    }
}
```

这个模式在 `dynamo/lib/runtime/src/worker.rs` 中非常典型。

### 4.3 限流模式

适用于“任务很多，但同时只能跑有限个”的场景。

```rust
use tokio::sync::Semaphore;
use tokio_util::sync::CancellationToken;
use std::sync::Arc;

let sem = Arc::new(Semaphore::new(8));
let cancel_token = CancellationToken::new();

tokio::select! {
    permit = sem.clone().acquire_owned() => {
        let _permit = permit?;
        // 执行实际工作
    }
    _ = cancel_token.cancelled() => {
        // 放弃排队
    }
}
```

这和 `SemaphoreScheduler` 的核心思路一致。

### 4.4 阻塞任务隔离模式

```rust
let handle = tokio::task::spawn_blocking(move || {
    heavy_sync_work();
});

handle.await?;
```

当逻辑本质上不是异步 IO，而是同步重计算时，这种写法比硬塞进 async 更合适。

---

## 5. 来自 Dynamo 的真实示例

### 示例 1：Worker 中的信号处理与优雅关闭

来源：`dynamo/lib/runtime/src/worker.rs`

核心点：

- `tokio::spawn` 启动 signal handler
- `oneshot` 用于应用生命周期协作
- `tokio::select!` 同时等待取消、应用结束和超时

可重点看：

- `signal_handler(cancel_token)`
- `Worker::execute_internal()`

### 示例 2：TypedPrefixWatcher 维护最新状态

来源：`dynamo/lib/runtime/src/utils/typed_prefix_watcher.rs`

核心点：

- `watch::channel(HashMap<K, V>)` 广播“当前完整状态”
- `tokio::spawn` 启动后台循环
- `tokio::select!` 在“取消”和“watch 事件”之间切换

这个例子很适合学习：

- 如何把事件流折叠成内存态
- 如何把“增量事件”变成“可订阅的最新快照”

### 示例 3：ModelWatcher 等待模型可用

来源：`dynamo/lib/llm/src/discovery/watcher.rs`

核心点：

- `Notify` 用于“有模型更新时唤醒等待者”
- `wait_for_chat_model()` 通过循环检查 + `notified().await` 等待条件满足

它适合“只关心唤醒，不需要带数据”的场景。

### 示例 4：SemaphoreScheduler 的并发控制

来源：`dynamo/lib/runtime/src/utils/tasks/tracker.rs`

核心点：

- `Semaphore` 做并发上限
- `select!` 让“等待 permit”可以被取消

这是仓库里很标准的 Tokio 并发控制写法。

### 示例 5：异步 pipeline 测试

来源：`dynamo/lib/runtime/tests/pipeline.rs`

核心点：

- `#[tokio::test]`
- `tokio::time::sleep(Duration::from_millis(10)).await`
- 直接测试流式输出是否按预期逐步产生

这是写异步流测试时很值得参考的最小样板。

---

## 6. 如果要在 Dynamo 里继续写 Tokio 代码，建议遵循这些原则

1. 不要随手新建 runtime，优先复用 `dynamo` 已经封装好的 `Runtime` / `Worker`。
2. 异步循环最好总是带退出机制，优先使用 `CancellationToken`。
3. 同时等待多个事件时优先考虑 `tokio::select!`，不要写成串行等待。
4. 有状态广播优先考虑 `watch`，一次性结果用 `oneshot`，流式消息用 `mpsc`。
5. 需要限流时用 `Semaphore`，不要靠裸 `Mutex` 或手写计数器硬控。
6. CPU 密集或阻塞逻辑不要占用 async worker thread，改用 `spawn_blocking` 或 compute pool。
7. 异步单测优先用 `#[tokio::test]`，这样最贴近仓库当前代码风格。

---

## 7. 小结

Tokio 在 `dynamo` 中主要不是“提供 async/await 语法支持”，而是负责：

- 运行时托管
- 后台任务调度
- 取消与优雅关闭
- 通道通信与状态广播
- 并发限制
- 异步测试

如果把 `PyO3` 文档理解为“Python 和 Rust 的桥”，那么 Tokio 在 `dynamo` 里更像“Rust 侧所有异步部件共享的地基”。
