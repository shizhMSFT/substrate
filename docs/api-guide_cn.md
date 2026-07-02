# Substrate API 指南：WorkerPool 与 ActorTemplate

本指南介绍如何配置 Substrate 资源以部署高密度、有状态的 agent。

## 1. WorkerPool：物理容量

`WorkerPool` 定义了物理“热备”计算容量的池。它管理一组处于待命状态的 pod（herder），这些 pod 已准备好接收并执行 actor 状态。

### 规格（`WorkerPoolSpec`）

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `replicas` | `int32` | **必填。** 集群中需要维持的物理待命 pod 数量。 |
| `ateomImage` | `string` | **必填。** `ateom` herder 进程使用的容器镜像（例如 `ko://github.com/agent-substrate/substrate/cmd/ateom-gvisor`）。 |
| `sandboxClass` | `string` | 可选。该池的沙箱运行时家族：`gvisor`（默认）或 `microvm`。它决定了 worker pod 的形态（例如 KVM 设备挂载、节点放置）以及哪些 `SandboxConfig` 符合条件。 |
| `sandboxConfigName` | `string` | 可选。提供沙箱二进制文件的集群范围 [`SandboxConfig`](#3-sandboxconfig沙箱二进制文件) 的名称。如果为空，则使用该池 `sandboxClass` 对应的集群默认 `SandboxConfig`。 |
| `template` | `WorkerPoolPodTemplate` | **可选。** worker pod 的 pod 调度与资源设置。 |

#### `WorkerPoolPodTemplate`（`spec.template`）

| 字段 | 类型 | Pod 映射 |
| :--- | :--- | :--- |
| `nodeSelector` | `map[string]string` | `spec.nodeSelector` |
| `tolerations` | `[]Toleration` | `spec.tolerations`（最多 16 个） |
| `priorityClassName` | `string` | `spec.priorityClassName` |
| `nodeAffinity` | `NodeAffinity` | `spec.affinity.nodeAffinity` |
| `resources` | `ResourceRequirements` | `spec.containers[].resources` |

### 示例

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: agent-pool
  namespace: ate-demo
  labels:
    workload: secret-agent
spec:
  replicas: 10
  ateomImage: ko://github.com/agent-substrate/substrate/cmd/ateom-gvisor
  # sandboxClass defaults to gvisor; the pool resolves to the cluster's default
  # gvisor SandboxConfig unless sandboxConfigName is set.
```

### GPU 节点调度示例

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: gpu-pool
  namespace: ate-demo
spec:
  replicas: 5
  ateomImage: ko://github.com/agent-substrate/substrate/cmd/ateom-gvisor
  template:
    nodeSelector:
      cloud.google.com/gke-accelerator: nvidia-tesla-t4
    tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
    priorityClassName: substrate-workers
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values: [substrate]
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: "1"
        memory: 2Gi
```

---

## 2. ActorTemplate：工作负载蓝图

`ActorTemplate` 定义了某一特定类型 agent 的代码、环境以及状态管理策略。它用于生成“黄金快照”（Golden Snapshot），该类型的所有 actor 都从此快照派生。

### 规格（`ActorTemplateSpec`）

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `containers` | `[]Container` | **必填。** 工作负载定义（镜像、命令、环境变量、端口）。每个容器还可以声明一个可选的 `readyz` HTTP 探针——参见 [容器就绪探针](#容器就绪探针readyz)。 |
| `sandboxClass` | `string` | 可选。该模板的 actor 所需的沙箱运行时家族：`gvisor`（默认）或 `microvm`。只有 `sandboxClass` 匹配的 `WorkerPool` 才符合条件。 |
| `workerSelector` | `*LabelSelector` | 可选。通过匹配每个池的标签，限定此模板的 actor 可以使用哪些 `WorkerPool`。如果未设置，则所有池均符合条件（受 actor 自身 `worker_selector` 的约束）。 |
| `snapshotsConfig` | `SnapshotsConfig` | **必填。** 存储内存快照的 GCS 存储桶和文件夹。 |
| `pauseImage` | `string` | **必填。** 用于沙箱根的镜像（例如 `gcr.io/gke-release/pause`）。 |

沙箱二进制文件（例如 gVisor 的 `runsc` 二进制文件）**不再在 `ActorTemplate` 上配置**。它们从被引用的 `WorkerPool` 的 [`SandboxConfig`](#3-sandboxconfig沙箱二进制文件) 中解析——按名称（`workerPool.spec.sandboxConfigName`），或默认使用该池 `sandboxClass` 对应的集群默认 `SandboxConfig`。

由于快照无法跨沙箱运行时恢复，`sandboxClass` 是一个**硬性调度门槛**：actor 只会被放置在类别匹配的 `WorkerPool` 上。它与 `workerSelector`（以及 actor 自身的 `worker_selector`）进行 AND 运算，只能进一步缩小符合条件的池范围。它默认为 `gvisor`，并且与规格的其余部分一样是不可变的，因此每个模板的类别在创建时即已固定。

容器环境变量支持字面量 `value` 条目以及 `valueFrom.secretKeyRef`。当工作负载规格被具体化时，Secret 引用由 `ate-api-server` 从 `ActorTemplate` 命名空间中解析。对于黄金 actor，解析后的值会被捕获到黄金快照中，未来的 actor 会继承这些值，直到黄金快照被重新创建为止。对于绕过黄金快照、从当前模板规格启动的 actor，解析后的值会被发送给 atelet，但不会序列化到公开的 Actor API 中。目前尚不支持其他 Kubernetes `valueFrom` 来源。Secret 变更不会自动重启 actor 或使快照失效；轮换 Secret 需要显式的 actor 或模板生命周期操作。

### 工作负载连通性（Uniform DNS）
Substrate 已标准化采用 **Uniform DNS Mesh（统一 DNS 网格）**。你不再需要定义 `SessionDiscovery` 规则。从模板创建的每个 actor 都会通过其唯一 ID 自动经由 **Substrate Router** 可达：

**格式：** `<actor-id>.actors.resources.substrate.ate.dev`

### Actor 身份标识
Substrate 会在每个 actor 容器中以只读方式绑定挂载一个 per-actor 身份目录，位于 **`/run/ate`**。actor 无需解析 `Host` 头部即可获知自身 ID，只需读取其中的文件 **`/run/ate/actor-id`**，该文件包含原始的 actor ID 且末尾无换行符。随着时间推移，该目录中可能会出现更多身份和配置数据。

请实时读取它，而不要在进程启动时缓存。它以 per-actor 绑定挂载的形式提供，而非环境变量，正是为了确保在从黄金快照恢复后它仍携带正确的 ID——环境变量（或烤入镜像中的文件）会被冻结为*黄金* actor 的 ID，因为它存在于被检查点保存的进程内存中，从而对该模板的每个 actor 都完全相同。

### 容器就绪探针（`readyz`）

`containers` 中的每个条目都可以声明一个可选的 **HTTP 就绪探针**，从而让平台只在工作负载真正开始处理流量后才将 actor 视为“已启动”。这类似于 Kubernetes Pod 容器上的 `readinessProbe.httpGet`，但该门槛是在 ateom（pod 内的沙箱驱动）内部强制执行的，而不是由 kubelet 执行。

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `readyz.httpGet.path` | `string` | 可选。要 GET 的 URL 路径。默认为 `/readyz`。必须以 `/` 开头，且只能包含 RFC 3986 路径字符（不能有查询字符串 `?` 或片段 `#`）。 |
| `readyz.httpGet.port` | `int32` | **必填。** 要探测的容器上的 TCP 端口（`1..65535`）。 |

其行为方式：

- **探针在哪里运行。** ateom（gVisor 或 microvm）在 actor 的内部 IP（当前为 `169.254.17.2`）访问容器——一次网络跳转，无需 DNS，不涉及 router。
- **阻塞直至就绪语义。** `RunWorkload`（冷启动）和 `RestoreWorkload`（从快照恢复）只有在每个带有 `readyz` 块的容器都返回 HTTP 200 后才会成功返回。失败会以 Run/Restore 错误的形式暴露，并由控制平面重试；整体等待受到内部 30 秒截止时间的限制。
- **激进轮询。** 轮询循环针对个位数毫秒级的检测延迟进行了调优：使用带 keep-alive 的 HTTP 客户端，间隔约 500µs，每次请求超时 250ms。当工作负载仍在启动时，内核 `RST` 会在微秒级返回，因此循环几乎不会被阻塞；一旦监听器就绪，下一次尝试便以 veth 本地延迟完成。
- **黄金快照预热快捷路径。** 当模板中的**每个**容器都声明了 `readyz` 时，actor template 控制器会跳过其默认的约 20 秒“给工作负载留出稳定时间”的延迟，直接拍摄黄金快照——因为 `ResumeActor` 已经阻塞到工作负载报告 200 为止,因此可以确定工作负载已经初始化完毕。任何容器省略 `readyz` 的模板则保留 20 秒预热作为安全网。
- **快照/恢复交互。** TCP 监听器是被检查点保存的 RAM 的一部分,因此在恢复时 `readyz` 通常在第一次尝试就返回 200,没有可观察到的延迟代价。

如果某个容器省略了 `readyz`,则保留先前的“已启动 == 就绪”行为——平台会在 `runsc start` / `vm.boot` 返回后立即将容器视为就绪。

### 示例

```yaml
apiVersion: ate.dev/v1alpha1
kind: ActorTemplate
metadata:
  name: secret-agent
  namespace: ate-demo
spec:
  # No sandbox/runsc config here — the binaries come from the WorkerPool's
  # SandboxConfig (see section 3).
  pauseImage: "gcr.io/gke-release/pause@sha256:bcbd57ba5653580ec647b16d8163cdd1112df3609129b01f912a8032e48265da"
  containers:
  - name: agent
    image: gcr.io/my-project/my-agent:latest
    command: ["/app/server"]
    ports:
    - containerPort: 80
    # Optional: gate Run/Restore on the agent's HTTP readiness endpoint.
    # See "Container Readiness Probe (readyz)" above.
    readyz:
      httpGet:
        path: /readyz
        port: 80
  # sandboxClass defaults to gvisor; set to microvm to require micro-VM pools.
  sandboxClass: gvisor
  workerSelector:
    matchLabels:
      workload: secret-agent
  snapshotsConfig:
    location: gs://my-bucket/snapshots/secret-agent/
```

---

## 3. SandboxConfig：沙箱二进制文件

`SandboxConfig` 是一种**集群范围**的资源，它将沙箱二进制文件（gVisor 的 `runsc` 二进制文件，或 micro-VM 的内核/固件/配置）与 `ActorTemplate` 解耦。`WorkerPool` 从 `SandboxConfig` 解析其二进制文件——要么是 `spec.sandboxConfigName` 指定的那个，要么是该池 `sandboxClass` 对应的集群默认值。

这意味着单个由集群管理的配置可以为许多模板固定沙箱运行时版本：由于版本记录在每个快照的清单中,快照仍可恢复,而运维人员可以在一处升级运行时。

### 规格（`SandboxConfigSpec`）

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `sandboxClass` | `string` | **必填。** 该配置适用的运行时家族：`gvisor`（默认）或 `microvm`。`WorkerPool` 只使用 `sandboxClass` 与自身匹配的 `SandboxConfig`。 |
| `default` | `bool` | 可选。将其标记为其 `sandboxClass` 的集群默认值。未设置 `sandboxConfigName` 的 `WorkerPool` 会解析到其类别的默认值。每个类别最多一个默认值。 |
| `assets` | `map[arch]map[name]AssetFile` | 可选。atelet 拉取的内容寻址文件，先按架构（`amd64`、`arm64`）再按 asset 名称作为键。gVisor 需要一个 `runsc` asset；micro-VM 后端需要多个。每个 `AssetFile` 是一个 `{ url, sha256 }` 对。 |

平台安装时会附带一个默认的集群范围 gVisor `SandboxConfig`（`gvisor-default`），因此 gVisor 池开箱即用。

### 示例

```yaml
apiVersion: ate.dev/v1alpha1
kind: SandboxConfig
metadata:
  name: gvisor-default
spec:
  sandboxClass: gvisor
  default: true
  assets:
    amd64:
      runsc:
        url: "gs://gvisor/releases/nightly/2026-05-19/x86_64/runsc"
        sha256: "a397be1abc2420d26bce6c70e6e2ff96c73aaaab929756c56f5e2089ea842b63"
    arm64:
      runsc:
        url: "gs://gvisor/releases/nightly/2026-05-19/aarch64/runsc"
        sha256: "1ba2366ae2efceba166046f51a4104f9261c9cb72c6db8f5b3fe2dc57dea86b9"
```

### Micro-VM SandboxConfig

`microvm` 类型的 `SandboxConfig` 提供 [Kata Containers](https://katacontainers.io/) + [Cloud Hypervisor](https://www.cloudhypervisor.org/) 工具链,而不是 `runsc`。每个架构都必须定义完整的 asset 集合——`kata-shim`、`cloud-hypervisor`、`virtiofsd`、`kata-kernel`、`kata-image` 和 `kata-config`——`ValidatingAdmissionPolicy` 会在 apply 时强制执行这一要求。micro-VM 池的 worker pod 需要 `/dev/kvm` 以及具备嵌套虚拟化能力、标记为 `ate.dev/sandboxClass=microvm` 的节点（控制器会自动添加设备挂载和节点放置）。

参见 [`hack/microvm-assets/`](../hack/microvm-assets/) 中用于组装和暂存这些 asset 的脚本,以及一个完整的 counter 演示（`demos/counter/counter-microvm.yaml.tmpl`）,它可以在 worker pod 之间挂起和恢复一个内存中的计数器。

---

## 4. 运行工作流

### 黄金快照
当创建 `ActorTemplate` 时：
1.  Substrate 启动一个临时的 **Golden Pod（黄金 Pod）**。
2.  它按模板中的定义执行你的工作负载容器。
3.  一旦进程初始化完成，gVisor 会拍摄一个 **Golden Snapshot（黄金快照，版本 0）**。
4.  模板进入 `Ready` 阶段。

### 恢复生命周期
一旦模板处于 `Ready` 状态，在逻辑上创建一个 actor（通过 `kubectl-ate create actor`）便可以让它在被引用的 `WorkerPool` 中任何空闲 worker 上瞬间恢复。Substrate 绕过标准的容器启动过程,直接从其上次保存的状态恢复进程。

---

## 5. 最佳实践
*   **启动逻辑：** 将开销大的初始化（加载大型模型、建立基线连接）放在应用程序的入口点。这些将被捕获到黄金快照中,无需在每次恢复时重复执行。
*   **对称性：** 确保你的 `ActorTemplate` 和 `WorkerPool` 位于同一命名空间中,或拥有适当的 RBAC 权限以相互引用。
*   **版本管理：** 更新代码时,创建一个新的 `ActorTemplate`（例如 `v2`）。Substrate 将每个模板视为一个不可变的状态根。

---

## 6. 控制平面 gRPC API

Substrate 控制平面（`ate-api-server`）暴露一个 gRPC 接口来管理 actor 和 worker。这是 `kubectl-ate` CLI 和更高层框架使用的主要 API。

### 服务：`ateapi.Control`

#### `CreateActor`
在系统中注册一个新的逻辑 actor。
*   **请求：** `CreateActorRequest`
    *   `actor_id`：唯一标识符（DNS-1123 label）。
    *   `actor_template_namespace`：`ActorTemplate` 的命名空间。
    *   `actor_template_name`：`ActorTemplate` 的名称。
*   **响应：** `CreateActorResponse`,包含已初始化的 `Actor` 对象。

#### `ResumeActor`
通过将挂起的 actor 恢复到物理 worker 上来激活它。
*   **请求：** `ResumeActorRequest`
    *   `actor_id`：要恢复的 actor 的 ID。
    *   `boot`：（可选）如果为 `true`,则绕过快照并执行冷启动。
*   **响应：** `ResumeActorResponse`,包含更新后的 `Actor` 对象（含物理 `worker_ip`）。

#### `SuspendActor`
休眠一个正在运行的 actor,将其当前的 RAM 和磁盘状态捕获到快照中。
*   **请求：** `SuspendActorRequest`
    *   `actor_id`：要挂起的 actor 的 ID。
*   **响应：** `SuspendActorResponse`,包含处于 `STATUS_SUSPENDED` 状态的 `Actor` 对象。

#### `DeleteActor`
从注册表中移除一个 actor。
*   **约束：** 只有处于 `STATUS_SUSPENDED` 状态的 actor 才能被删除。
*   **请求：** `DeleteActorRequest`
*   **响应：** `DeleteActorResponse`（空）。

#### `GetActor` / `ListActors`
查询逻辑 actor 的状态。
*   **GetActor：** 按 ID 检索单个 actor。
*   **ListActors：** 列出数据库中当前跟踪的所有 actor。

#### `ListWorkers`
查询物理资源池。
*   **请求：** `ListWorkersRequest`
*   **响应：** `ListWorkersResponse`,包含 `Worker` 对象（Pod）列表及其当前分配状态。

---

## 7. 进阶：Session Identity（会话身份）

工作负载可以用其临时的 Kubernetes 凭证交换稳定的 **Session Identity（会话身份）** 凭证,即使进程在不同物理 worker 之间迁移,这些凭证也能持续有效。

### 服务：`ateapi.SessionIdentity`
*   **`MintJWT`：** 生成一个用于标识 Substrate Actor 的、与 OIDC 兼容的 JWT。
*   **`MintCert`：** 对证书签名请求（CSR）进行签名,为 actor 提供 mTLS 身份。

---

## 8. 框架与生态系统集成

Agent Substrate 旨在成为任何 agent 框架的基础执行层。

### Agent Development Kit (ADK)
Substrate 为兼容 ADK 的身份提供原生支持。工作负载可以使用 `SessionIdentity` 服务来铸造与 ADK 安全模型一致的 JWT,从而确保与 ADK 管理的工具和内存无缝集成。

### LangChain
Substrate 是有状态 LangChain agent 的理想运行时。通过将 LangChain agent 定义为 `ActorTemplate`,你可以在多次休眠间将 agent 内部的“思维过程”和对话历史保留在内存中,同时对其工具执行进行沙箱隔离以保证安全。

### Claude Code 与 CodeX
对于面向开发者的 agent,Substrate 支持编码环境的大规模多路复用。每位开发者都可以拥有一个专属的、持久化的终端会话（Actor）,它会保留文件系统增量,而集群只为活跃用户运行物理 pod。
