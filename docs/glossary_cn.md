# Agent Substrate 术语表

本文档定义了 Agent Substrate 中使用的核心术语。

关于各部分如何协同工作,参见 [Architecture](architecture_cn.md) 和
[API Guide](api-guide_cn.md)。

## 资源（声明式,Kubernetes CRD）

- **ActorTemplate**：actor“类”的定义：容器镜像以及快照配置。创建
  `ActorTemplate` 会触发 [Golden Snapshot（黄金快照）](#快照snapshots) 的创建。
  它被视为不可变的：为新版本创建一个新模板,而不是编辑现有模板。它
  类似于 Pod 模板,但面向的是可检查点保存的工作负载。

- **WorkerPool**：声明热备计算容量,即一组预先启动的 worker
  pod。它由 [atecontroller](#组件) 协调为一个 Kubernetes `Deployment`。

## 记录（动态状态,存于控制平面存储中）

它们不是 Kubernetes 对象;由于变化过于频繁,不适合 etcd,因此它们
存在于控制平面数据库中。

- **Actor**：从 `ActorTemplate` 派生的单个实例,由一个
  DNS-1123 actor ID 标识。它是被挂起和恢复的单位,并在其生命周期内
  在各 worker 之间移动。Actor 记录会跟踪其生命周期
  状态和快照引用。

- **Worker**：代表 `WorkerPool` 中一个 worker pod 的记录。一个 Worker
  同一时刻最多托管一个 Actor;随着时间推移,许多 Actor 会在一个池中被多路复用。

## 组件

- **ate-api-server**（二进制文件 `ateapi`）：控制平面。它负责 Actor
  生命周期,将 Actor 调度到 Worker 上,并协调它们的快照,
  这一切都由状态存储支撑。`kubectl-ate` CLI 与它通信。

- **atecontroller**：协调 CRD 的 Kubernetes 控制器（例如,
  它将 `WorkerPool` 变成一个 `Deployment`）。

- **atelet**：节点级的监管进程,以 DaemonSet 方式运行。它拉取镜像、
  组装 OCI bundle、通过 ateom 在节点上驱动沙箱生命周期,
  并在快照存储之间流式传输快照。

- **ateom**：运行在每个 worker pod 内部的协调器,代表 atelet 驱动
  沙箱运行时。这将物理 pod 生命周期与被沙箱隔离的 agent 进程解耦。

- **atenet**：网络栈。它提供一个用于 actor 解析的 DNS 服务器,
  以及一个按需恢复挂起 Actor 并将流量路由到正确 worker pod 的 router。

- **podcertcontroller**：签发短期的 pod 证书,供组件
  用作其 TLS 身份,以相互认证连接（双向 TLS）。

- **kubectl-ate**：用于管理 Actor 生命周期和
  列出 Worker 的 `kubectl` 插件 CLI。

## 生命周期

- **Suspend（挂起）**：通过将正在运行的 Actor 检查点保存到快照并
  释放其 Worker 来休眠它。被请求的快照会上传到外部存储。

- **Pause（暂停）**：正在运行的 Actor 的短期检查点。快照文件保留
  在节点 VM 上,随后的 Resume 会优先调度到持久化了这些快照的节点 VM 上。

- **Resume（恢复）**：通过将挂起/暂停的 Actor 恢复到 Worker 上来激活它。
  常见路径是从快照恢复,而不是冷启动。

## 卷（Volumes）

- **DurableDir 卷**：一个挂载到一个或多个容器中的目录,
  其内容由 [`Data` 快照范围](#快照snapshots) 保留,
  因此可独立于进程内存或其他 rootfs 写入而跨 Suspend/Resume 存续。
  单个 `ActorTemplate` 可以声明多个 `DurableDir` 卷,
  且同一个卷可以挂载到多个容器中（可能位于不同路径）。这是
  per-Actor 的应用数据界面。

## 快照（Snapshots）

- **快照范围（Snapshot scope）**：`ActorTemplate` 的 `SnapshotsConfig` 在
  给定快照中包含哪些内容。目前存在两种范围：
  - **`Full`**：进程内存加上 OCI 镜像之上的 rootfs 增量
    （其中也包括任何已附加的 `DurableDir` 卷,
    因为它们位于 rootfs 内部）。用于捕获热恢复所需的一切内容。
  - **`Data`**：仅包含支持快照的已附加卷的内容——
    目前是 `DurableDir` 卷。进程内存和 rootfs 的
    其余部分会被丢弃;恢复时,Actor 会从 OCI 镜像冷启动,
    并恢复 `DurableDir` 内容。用于以低成本持久化
    应用数据,而无需完整内存镜像的开销。

  通过 `onPause` 和 `onCommit` 按触发器配置：`onPause` 选择
  在 [Pause](#生命周期) 期间捕获什么（保留在节点上）,而
  `onCommit` 选择在 [Suspend](#生命周期) 期间捕获什么
  （上传到快照存储）。`onCommit` 必须是 `onPause` 的子集。

- **Golden Snapshot（黄金快照）**：在创建 `ActorTemplate` 时,
  从工作负载的一次临时“黄金”启动中一次性捕获的初始检查点。
  默认情况下,该模板的 Actor 首先从这个共享快照恢复。

- **Last Snapshot（最新快照）**：最近的 per-Actor 快照,在 Suspend 时写入,
  用于在下次 Resume 时恢复该特定 Actor。

- **快照存储（Snapshot storage）**：持久化快照的对象存储（GCS 或 S3）,
  使 Actor 状态在整个集群中持久且可移植。

## 网络

- **Uniform DNS Mesh（统一 DNS 网格）**：每个 Actor 都可通过一个统一地址访问,
  即 `<actor-id>.actors.resources.substrate.ate.dev`,由 atenet 解析。发往
  该名称的流量会被自动路由（并在需要时恢复 Actor）。
