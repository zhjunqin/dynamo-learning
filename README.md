# dynamo-learning

本仓库用于学习/梳理 **NVIDIA Dynamo** 相关代码与运行机制，并沉淀一些面向本项目的说明文档。

## 文档导航

### 构建与本地开发

- [Dynamo 构建与开发指南](build/build_dev_guide.md)
- [容器构建与运行（上游 Dynamo）](dynamo/container/README.md)
- [测试与本地开发流程（上游 Dynamo）](dynamo/tests/README.md)

### 架构概念与源码逻辑（本仓库整理）

- [Dynamo 源码结构与运行逻辑](overview/DYNAMO_SOURCE_STRUCTURE.md)
- [Frontend 启动代码执行路径](overview/code_logic/frontend_startup.md)
- [Request Plane 与 Event Plane 概念与源码入口](overview/request_event_planes.md)
- [Python → Rust 绑定详解（PyO3 / maturin）](overview/rust/python_rust_bindings.md)

### 上游 Dynamo 文档入口（从这里开始深挖）

- [Dynamo 文档站点构建与发布流程（Fern）](dynamo/docs/README.md)
- [Kubernetes 文档入口](dynamo/docs/kubernetes/README.md)
- [组件文档：Frontend](dynamo/docs/components/frontend/README.md)
- [组件文档：Router](dynamo/docs/components/router/README.md)
- [后端文档：vLLM](dynamo/docs/backends/vllm/README.md)
- [后端文档：SGLang](dynamo/docs/backends/sglang/README.md)
- [后端文档：TensorRT-LLM](dynamo/docs/backends/trtllm/README.md)

### 示例与生产配方（上游 Dynamo）

- [Examples 总览](dynamo/examples/README.md)
- [Kubernetes Recipes 总览](dynamo/recipes/README.md)