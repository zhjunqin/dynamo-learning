# Dynamo 构建与开发指南

本文档说明如何在本地构建 vLLM 开发镜像，以及进入容器后如何重新编译修改过的 Rust 代码。

## 一、本地编译：构建 vLLM 开发镜像

开发环境推荐使用 **local-dev** 镜像：容器内以 `dynamo` 用户运行，UID/GID 与宿主机一致，便于挂载工作区并直接编辑代码。

### 1.1 生成 Dockerfile

进入 **dynamo** 目录（即包含 `container/` 的子目录）后执行：

```bash
cd dynamo

# vLLM + local-dev 目标，输出文件名为 container/rendered.Dockerfile
container/render.py --framework=vllm --target=local-dev --output-short-filename
```

可选参数示例：

- **CUDA 版本**：`--cuda-version=12.9` 或 `13.0`（vllm 支持）
- **平台**：`--platform=amd64`（默认）或 `arm64`

### 1.2 构建镜像

在 **dynamo** 目录下执行：

```bash
# 传入当前用户 UID/GID，便于挂载目录时权限一致
docker build --build-arg USER_UID=$(id -u) --build-arg USER_GID=$(id -g) \
  -f container/rendered.Dockerfile -t dynamo:latest-vllm-local-dev .
```

### 1.3 启动开发容器（宿主机改代码、容器内可见并编译）

要让**宿主机上的 dynamo 目录**（例如 `dynamo-learning/dynamo`）里的修改在容器内立即可见，必须**从该 dynamo 目录**启动容器，并加上 `--mount-workspace`。  
`run.sh` 会把「**run.sh 所在目录的上一级**」（即你执行命令时的当前目录）挂载到容器内的 `/workspace`，因此两边是同一份代码。

**步骤：**

1. 在宿主机进入 dynamo 目录（你要改代码的那个路径）：
   ```bash
   cd dynamo-learning/dynamo
   ```

2. 在该目录下启动开发容器：
   ```bash
   container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it \
     -v $HOME/.cache:/home/dynamo/.cache
   ```

3. 进入容器后，当前工作目录为 `/workspace`，对应宿主机上的 `dynamo` 目录。  
   在宿主机用编辑器修改 `dynamo` 下的 Rust/Python 后，容器内会立即看到变更，在容器内执行 `cargo build ...` 等即可重新编译。

**一条命令示例（宿主机）：**

```bash
cd dynamo-learning/dynamo
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it -v $HOME/.cache:/home/dynamo/.cache -v ../../dynamo-learning:/dynamo-learning --user root
```

**以 root 用户进入容器：**

在命令中加上 `--user root`，并将缓存挂到 root 的 home（可选）：

```bash
cd dynamo-learning/dynamo
container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it --user root \
  -v ../../dynamo-learning:/dynamo-learning  -v $HOME/.cache:/root/.cache
```

**dynamo 是独立仓库 vs submodule（容器内 git 是否可用）：**

- **独立仓库**：若你是直接 `git clone` 的 dynamo 仓库，则 `dynamo/.git` 是一个**目录**。只挂载 `dynamo` 到 `/workspace` 时，容器内 `git status` 等即可正常使用，无需改挂载方式。
- **Submodule**：若 dynamo 是父仓库（如 `dynamo-learning`）里的 submodule，则 `dynamo/.git` 是一个**文件**，内容指向 `../.git/modules/dynamo`。在宿主机上能跑 `git status`，是因为父仓库存在；容器里若只挂载 `dynamo`，没有父仓库，就会报 `fatal: not a git repository: /workspace/../.git/modules/dynamo`。此时需要把**父仓库**挂到 `/workspace`，并在容器里把工作目录设到 `dynamo`。

如何判断：在宿主机执行 `ls -la dynamo/.git`，若是**文件**则为 submodule，若为**目录**则为独立仓库。

**Submodule 时的启动方式（容器内要用 git）：**

在**父仓库根目录**执行（路径按你本机调整）：

```bash
cd dynamo-learning
dynamo/container/run.sh --image dynamo:latest-vllm-local-dev --mount-workspace -it --user root \
  -v $HOME/.cache:/root/.cache \
  -v dynamo-learning:/workspace \
  --workdir /workspace/dynamo
```

- `-v .../dynamo-learning:/workspace`：用父仓库覆盖默认的 workspace 挂载，使 `/workspace/.git` 和 `/workspace/.git/modules/dynamo` 存在。
- `--workdir /workspace/dynamo`：进入容器后当前目录为 `dynamo`，可直接 `git status`、`cargo build` 等。

### 1.4 其他构建目标简要说明

| 目标        | 用途           | 说明 |
|-------------|----------------|------|
| `local-dev` | 日常开发（推荐） | 非 root、UID/GID 与宿主机一致，可挂载 workspace |
| `dev`       | 旧版开发镜像    | 以 root 运行，慎用 |
| `runtime`   | 生产/压测      | 非 root，无 Rust 工具链，仅预装 wheel |

更多细节见 [dynamo/container/README.md](dynamo/container/README.md)。

---

## 二、容器内重新编译修改过的 Rust 代码

使用 `--mount-workspace` 时，宿主机上的代码会挂载到容器内的 `/workspace`。在宿主机或容器内修改 Rust 代码后，**无需重新构建 Docker 镜像**，只需在容器内重新编译即可。

### 2.1 仅修改了纯 Rust 代码（不涉及 Python 绑定）

在容器内、项目根目录（如 `/workspace`，即 dynamo 根）执行：

```bash
cargo build --locked --features dynamo-llm/block-manager --workspace
```

若项目启用了其他 feature（如 KVBM），保持与现有用法一致即可。

### 2.2 修改了 Rust 代码且 Python 会调用（如 dynamo._internal、llm 绑定等）

需要先编译 Rust，再刷新 Python 侧的扩展：

```bash
# 1. 编译 Rust
cargo build --locked --features dynamo-llm/block-manager --workspace

# 2. 重新安装 Python 绑定到当前 venv（使 import dynamo 使用最新 Rust 代码）
cd lib/bindings/python && maturin develop --uv && cd -
```

之后在容器内运行 `python -m dynamo.xxx` 或 pytest 时会使用刚编译的 Rust 扩展。

### 2.3 Release 构建与测试（可选）

```bash
# Release 构建（优化更多，编译更慢）
cargo build --locked --release --features dynamo-llm/block-manager --workspace

# 运行 Rust 单元/集成测试
cargo test --locked --all-targets

# 需要 ETCD/NATS 的集成测试
cargo test --features integration
```

### 2.4 环境自检（可选）

编译完成后，可做一次快速自检：

```bash
# local-dev / dev 容器
deploy/sanity_check.py

# 仅检查运行时环境（如 runtime 镜像）
deploy/sanity_check.py --runtime-check-only
```

---

## 三、流程小结

| 步骤       | 位置     | 操作 |
|------------|----------|------|
| 构建镜像   | 宿主机（dynamo 目录） | `render.py` → `docker build` |
| 启动容器   | 宿主机（dynamo 目录） | `container/run.sh --image ... --mount-workspace -it ...` |
| 修改代码   | 宿主机或容器 | 编辑 Rust/Python（挂载后两边同步） |
| 重新编译   | 容器内   | `cargo build ...`，必要时 `maturin develop --uv` |

相关文档：

- [dynamo/container/README.md](dynamo/container/README.md) — 容器构建与 run.sh 详细说明  
- [dynamo/tests/README.md](dynamo/tests/README.md) — 测试与本地开发流程
