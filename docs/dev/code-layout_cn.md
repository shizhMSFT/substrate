# 代码布局

本文档说明本仓库的顶层目录结构，以及代码放置位置背后的原理。当你不确定新文件应该放在哪里时，请先阅读本文。

## 顶层目录

```
substrate/
├── cmd/          # Binary entry points (one subdirectory per binary)
├── internal/     # Shared Go packages, not importable outside this module
├── pkg/          # Shared Go packages, intentionally importable by external users
├── docs/         # Design documents and developer guides
├── hack/         # Scripts used during development, CI, and cluster management
├── manifests/    # Kubernetes manifests for deploying Substrate
├── demos/        # Self-contained demo applications
├── benchmarking/ # Load testing tools and workloads
├── tools/        # Standalone Go tools (runnable with `go run ./tools/<name>`)
└── bin/          # Vendored or pinned tool binaries (e.g. protoc)
```

## 决策规则

### `pkg/` 与 `internal/`

**`pkg/`** 用于具有对外 API 契约的 Go 包——即用户或第三方工具可以直接从
`github.com/agent-substrate/substrate/pkg/...` 导入的代码。将某个包放在这里，
意味着承诺其向后兼容性与可发现性。

**`internal/`** 用于在本模块内多个二进制程序之间共享、但不属于任何对外 API 的 Go 包。
Go 工具链会强制保证 `github.com/agent-substrate/substrate` 之外的任何代码都无法导入这些包。

> **谨慎使用 `pkg/`。** 一旦项目达到 GA，`pkg/` 中任何导出的类型、函数或字段都会受到
> 兼容性保证的约束——删除或重命名它就会对外部使用者造成破坏性变更。在其 API 稳定之前，
> 请优先将新代码保留在 `internal/` 中。加入 `pkg/` 的标准是：“我确信外部用户需要导入它，
> 并且我准备好无限期地维护它的 API。”

### `cmd/<binary>/`

`cmd/` 的每个子目录对应一个编译出的二进制程序：

| Directory            | Binary / Purpose                                      |
|----------------------|-------------------------------------------------------|
| `cmd/ateapi`         | Control-plane API server (gRPC)                       |
| `cmd/atecontroller`  | Kubernetes controller for WorkerPool/ActorTemplate    |
| `cmd/atelet`         | Node supervisor (DaemonSet)                           |
| `cmd/atenet`         | Network proxy / Envoy external-processing server      |
| `cmd/ateom-gvisor`   | In-pod gVisor container image entry point             |
| `cmd/ateom-microvm`  | In-pod kata + cloud-hypervisor micro-VM container image entry point |
| `cmd/kubectl-ate`    | `kubectl` plugin for interacting with Substrate       |
| `cmd/podcertcontroller` | Controller that issues pod TLS certificates        |

每个 `cmd/<binary>/` 包含：
- `main.go` —— 入口点，保持精简（参数解析、装配、信号处理）
- `internal/` —— 该二进制程序私有的包；**不**与其他二进制程序共享

如果某个包只被一个二进制程序使用，请将其放在 `cmd/<binary>/internal/` 下。
如果两个或更多二进制程序共享某个包，请将其移动到 `internal/`（如果外部使用者也需要，则移到 `pkg/`）。

### `hack/`

供人使用的脚本：搭建开发环境、运行校验器、生成代码、管理集群。`hack/` 中的任何内容都不会作为
Go 包被导入。这里优先使用 shell 脚本；对于更复杂的自动化，请将 Go 工具放入 `tools/`。

### `tools/`

用 Go 编写的开发/CI 工具。这些是构建期或运维工具，属于本仓库的一部分，但不会被编译进任何
交付的二进制程序中。例如：`tools/setup-gcp` 用于置备 GCP 资源。

每个工具都必须拥有自己专用的 `go.mod`（以及 `go.sum`）。

### `manifests/`

用于部署 Substrate 组件的 Kubernetes YAML。

### `demos/`

展示如何基于 Agent Substrate 进行构建的示例应用。

### `benchmarking/`

负载测试工具与基准测试工作负载。

## 放置清单

在添加新的 Go 包时，请自问：

1. **它是否只被一个二进制程序使用？** → `cmd/<binary>/internal/<pkg>`
2. **它是否被多个二进制程序共享、但不打算供外部导入？** → `internal/<pkg>`
3. **它是否是有意对外公开、供外部用户使用的 API？** → `pkg/<pkg>`
4. **它是否是由 protobuf 生成的包？**
   - 公开的 gRPC API（控制平面）→ `pkg/proto/<name>`
   - 内部的 gRPC API（atelet、ateom）→ `internal/proto/<name>`
5. **它是否是脚本或开发工具？** → `hack/`（shell）或 `tools/<name>`（Go）
