# Agent Substrate 中的 Actor 可观测性

Agent Substrate 将 actor 作为虚拟的长生命周期实体来管理，它们可以在空闲时被挂起（suspend），并随时间推移在不同的 Kubernetes worker pod 上恢复（resume）。

本指南说明 Agent Substrate 如何在这些挂起/恢复周期中实现可观测性，让你能够监控日志、指标和追踪，就好像一个 actor 一直在单台专用机器上持续运行一样。

## 可观测性模型

为了让底层基础设施的迁移对用户透明，Agent Substrate 建立了一套标准化的元数据模型，用于在各个 worker pod 之间标识 actor：
* `ate.dev/actor_id`：actor 的唯一标识符（例如 `my-counter-1` 或 `test`）。
* `ate.dev/actor_template_name`：actor 所属 ActorTemplate 的名称（例如 `counter`）。
* `ate.dev/actor_template_namespace`：actor 所属 ActorTemplate 的 Kubernetes 命名空间（例如 `ate-demo-counter`）。
* `ate.dev/container_name`：actor 中产生该日志行的容器名称（例如 `counter`），以便多容器 actor 的日志能够按容器进行解复用（demultiplex）。

目前，Agent Substrate 会自动包装容器输出，并将这些元数据标签注入到**容器日志**中。对于指标和分布式追踪，Agent Substrate 提供了基础的系统遥测和按需请求追踪，并在路线图中计划全面集成 actor 级别的关联。

---

## 1. 日志（Logging）

Agent Substrate 捕获容器的标准输出/标准错误，将它们包装成结构化的 JSON 日志条目，并注入 `ate.dev` 元数据标签。

### 通过 CLI 检查活跃 Actor
若要快速、按需地调试一个活跃的 actor，可使用 Agent Substrate CLI：

```bash
kubectl ate logs actors <actor_id> [--follow / -f]
```

> **注意：** 默认情况下，`kubectl ate logs` 会查询 actor *当前*运行所在的 worker pod 的 Kubernetes API。它专为即时检查活跃 actor 而设计。若要查看跨越以往 worker pod 和挂起周期的历史日志，请使用集中式日志后端。

#### 示例 1：Actor 当前未运行
如果某个 actor 处于挂起状态或未被分配到任何 worker pod，CLI 会立即告知你：

```bash
$ kubectl ate logs actors test
Error: actor test is not currently running on any worker pod
```

#### 示例 2：默认的干净 JSON Lines 输出
当一个活跃的 actor 被分配到某个 worker pod 时，CLI 会输出干净、统一、去除了 Substrate 元数据的 JSON 行，完全匹配标准 `kubectl logs` 的行为：

```bash
$ kubectl ate logs actors test
{"time":"2026-05-22T21:49:15.23700774Z","message":"Actor started"}
{"time":"2026-05-22T21:49:15.23700774Z","level":"INFO","msg":"Starting counter server on port 80"}
{"time":"2026-05-22T21:49:15.255765354Z","count":0,"fshash":"mCY7G4S318ztOUojPTF2NA/W+ZSmWyr+T5K3udFuP50","level":"INFO","msg":"Count"}
{"time":"2026-05-22T21:49:25.263744806Z","count":1,"fshash":"mCY7G4S318ztOUojPTF2NA/W+ZSmWyr+T5K3udFuP50","level":"INFO","msg":"Count"}
```

#### 示例 3：流式/实时日志（`--follow` 或 `-f`）
若要实时流式查看 actor 日志，可附加 `--follow`（或 `-f`）标志。CLI 完全具备 actor 感知能力，当 actor 被挂起或迁移到不同的 worker pod 时，会自动恢复日志流：

```bash
$ kubectl ate logs actors test -f
Actor is currently running on pod ate-demo-counter/counter-deployment-d8f99-m7d96
{"time":"2026-05-22T21:49:15.255765354Z","count":0,"fshash":"mCY7...","level":"INFO","msg":"Count"}
{"time":"2026-05-22T21:49:25.263744806Z","count":1,"fshash":"mCY7...","level":"INFO","msg":"Count"}
Actor is currently running on pod ate-demo-counter/counter-deployment-ab123-x4y5z
{"time":"2026-05-22T21:50:02.123456789Z","count":2,"fshash":"mCY7...","level":"INFO","msg":"Count"}
```


---

### 集中式日志后端（多维度聚合）
若要查看 actor 跨越以往和当前 worker pod 的连续日志历史，你可以将 Agent Substrate 与任意支持结构化 JSON 索引的集中式日志后端（如 Grafana 或 Google Cloud Logging）集成。

由于日志管道会索引核心元数据标签，你可以使用日志平台的查询语言从多个维度查询日志（下面的示例使用 Google Cloud Log Explorer 语法）：

#### 1. 以 Actor 为中心的视图
无论一个 actor 在多少个 worker pod 之间迁移过或被挂起/恢复过多少次，都可以追踪其统一、连续的生命周期：

```text
labels.actor_id="test"
```

#### 2. 以模板为中心的视图
监控或调试由某个特定模板创建的所有 actor 实例（例如分析所有 counter actor 的整体行为或错误率）：

```text
labels.actor_template="counter"
```

#### 3. 以 Pod 为中心的视图
检查物理 worker pod 的聚合流，查看多路复用在一起的所有共置 actor（对于排查 pod 级别的资源耗尽或"吵闹邻居"问题很有用）：

```text
resource.labels.pod_name="counter-deployment-c995fdf4c-m7d96"
```

---

## 2. 指标（Metrics）

Agent Substrate 发出基础的 OpenTelemetry 系统与服务器指标，用于监控控制平面服务的整体健康状况和性能。下面的每个指标都由某个服务二进制文件通过 OTLP 发出，并且**独立于部署环境**——Kind 开发集群获得的仪表（instrument）与生产环境相同；只是后端不同（参见[遥测数据的去向](#4-遥测数据的去向)）。

| 指标 | 发出方 | 类型 | 度量内容 |
|--------|------------|------|----------|
| `rpc.server.call.duration` | ateapi 和 atelet（gRPC 服务器，通过 `otelgrpc`） | histogram | 每方法（per-method）的 gRPC 延迟、请求速率和错误（标签 `rpc.method`、`rpc.response.status_code`） |
| `atenet.router.route.duration` | atenet-router | histogram | Substrate 端到端——从 Envoy 接收请求到 Envoy 将其转发给已解析的 worker，不包括 actor 计算和响应 |
| `atelet.snapshot.size` | atelet | histogram | 检查点（checkpoint）期间写入的每个 gVisor 快照镜像的未压缩字节大小（标签 `kind`、`actor_template_name`） |

上表列出的是 OpenTelemetry 仪表名称。一个名称在查询中如何呈现取决于后端（Cloud Monitoring (GMP) / Kind collector）。

### 使用 Prometheus 的本地指标（Kind 集群）

对于 `kind` 集群内的本地开发，Agent Substrate 会自动在 `otel-system` 命名空间中预置一个 Prometheus 服务器。

在本地探索指标：

1. **暴露 Prometheus UI**，通过端口转发：
   ```bash
   kubectl port-forward -n otel-system svc/prometheus 9090:9090
   ```

2. **在 Web 浏览器中打开 Prometheus UI**：
   [http://localhost:9090](http://localhost:9090)

3. **查询指标**：运行 `up` 以确认每个组件都被抓取（每个目标一条序列，值为 `1`），然后通过表达式浏览器的自动补全功能探索 `rpc_*` 序列。**Status > Targets** 会列出已发现的 pod。

> **注意：** 存储是临时性的（`emptyDir`），因此当 Prometheus pod 重启时，指标会丢失。

> **路线图说明（Actor 级别指标）：** 一个全面的指标路线图正在积极开发中，以同时支持系统运维人员和工作负载分析。计划中的 OpenTelemetry 埋点聚焦于控制平面延迟、状态快照性能、机群利用密度，以及用标准化的 actor 标签丰富指标，从而在 pod 迁移过程中实现无缝聚合。

---

## 3. 追踪（Tracing）

分布式追踪跟踪请求在通过 Agent Substrate 网关、路由器、worker pod 和外部服务时的端到端流程。

目前，Agent Substrate 支持按需请求追踪。当由客户端发起时（例如通过 `--trace` 标志），Agent Substrate 利用 OpenTelemetry (OTel) 在整个调用栈中进行上下文传播。每个被追踪的请求都会生成一个唯一的追踪哈希/ID，你可以用它在 Google Cloud Trace 或 Jaeger 中检查详细的请求生命周期和 span 层级结构。

### 使用 Jaeger 的本地追踪（Kind 集群）

对于 `kind` 集群内的本地开发，Agent Substrate 会自动预置一个本地 OpenTelemetry Collector 和 Jaeger 实例。

在本地可视化追踪：

1. **暴露 Jaeger 查询 UI**，通过端口转发：
   ```bash
   kubectl port-forward -n otel-system svc/jaeger 16686:16686
   ```

2. **在 Web 浏览器中打开 Jaeger UI**：
   [http://localhost:16686](http://localhost:16686)

3. **生成追踪**：使用 `--trace` 标志运行一条 CLI 命令或 API 调用，例如：
   ```bash
   kubectl ate get actor --trace
   # 或
   kubectl ate suspend actor <actor-id> --trace
   ```

4. **搜索并检查**：从 CLI 输出中复制打印出的 Trace ID，粘贴到 Jaeger 搜索框（右上角），或在 **Service** 下拉菜单中选择 `ateapi` 或 `atelet` 并点击 **Find Traces**，以检查详细的调用栈、DB 事务、状态更新和 worker pod 交接。

> **开发者指南：** 有关在服务器或客户端中配置 OpenTelemetry tracer provider、中间件和 exporter 的详细说明，请参阅[追踪最佳实践](dev/best-practices/tracing_cn.md)指南。

---

## 4. 遥测数据的去向

遥测数据在所有环境中都以相同的方式发出；本地 Kind 集群与 Google Cloud (GKE) 部署之间只是后端不同。下面云端的后端都是 **GCP 服务**。

| | Kind | GKE (Google Cloud) |
|---|---|---|
| 路径 | service → 集群内 `opentelemetry-collector` | service → Google Managed Prometheus (GMP) |
| 指标 | 位于 `:8889` 的 collector Prometheus exporter | Google Cloud Monitoring |
| 追踪 | Jaeger UI | Google Cloud Trace |
| 仪表盘 | 不支持 | Google Cloud Monitoring（参见[仪表盘](#5-仪表盘)） |

> 在 Kind 中，只有 `ateapi` 和 `atelet` 指向集群内的 collector；`atenet-router` 仍然指向 GKE collector 端点，因此 `atenet.router.route.duration` 虽被发出，但不会在本地被收集。

---

## 5. 仪表盘（Dashboards）

> **GCP 专属。** 这些是 **Google Cloud Monitoring** 仪表盘；它们仅适用于 GKE / Google Cloud 部署。Kind 上不支持仪表盘——本地开发请使用[指标](#2-指标metrics)中的 Prometheus UI。

仪表盘定义位于 [`tools/setup-gcp/dashboards/`](../tools/setup-gcp/dashboards/)（其 README 中有每个仪表盘的详细分解）。它们作为 **GCP 设置的一部分**被创建和更新：`tools/setup-gcp` 会幂等地应用每个仪表盘（按显示名称匹配和更新），因此重新运行是安全的。

```sh
go run ./tools/setup-gcp create dashboards   # 也是以下命令的一部分：bootstrap
```
