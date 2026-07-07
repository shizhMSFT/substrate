# Agent Substrate — Consumer/Caller Interaction Guide

> How to use Substrate from the outside. Covers the full lifecycle from deployment to traffic routing.
>
> Date: 2026-07-07

---

## Overview: The 4 Steps

Your intuition is correct. As a consumer of Substrate, the interaction is:

```
1. Install Substrate on a K8s cluster
2. Define workload infrastructure (CRDs: WorkerPool + ActorTemplate)
3. Create an atespace, then actor instances in it (via CLI or gRPC API)
4. Send HTTP traffic → auto-activates suspended actors
```

Plus optional lifecycle management (suspend, resume, pause, delete, observe).

> **Atespace:** actors live inside an **atespace** — the isolation boundary an
> actor belongs to. An actor is identified by the pair `(atespace, actor-id)`;
> an actor id is only unique within its atespace. The atespace must exist before
> you can create actors in it, and it becomes part of the actor's routing DNS
> name and its storage key (`actor:<atespace>:<actor-id>`).

---

## Step 1: Install Substrate

This is a one-time cluster setup. You deploy the Substrate "system" into an `ate-system` namespace.

```bash
# Option A: Local development (KinD)
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system

# Option B: GKE
source .ate-dev-env.sh
go run ./tools/setup-gcp --all        # Creates GKE cluster, Redis, GCS, IAM
./hack/install-ate.sh --deploy-ate-system
```

**What gets deployed:**
| Component | K8s Resource | Purpose |
|-----------|-------------|---------|
| `ate-api-server` | Deployment | Control plane gRPC API |
| `atelet` | DaemonSet | Node-level supervisor |
| `atenet-router` | Deployment | HTTP traffic entry point (Envoy) |
| `atenet-dns` | Deployment | DNS resolution for actors |
| `atecontroller` | Deployment | CRD reconciler |
| `podcertcontroller` | Deployment | Pod TLS certificates |
| Valkey (Redis) | StatefulSet | State store |
| CRDs | `WorkerPool`, `ActorTemplate` | API extensions |

After this, the system is idle and waiting for you to define workloads.

---

## Step 2: Define Your Workload (CRDs)

You declare **two resources** via standard `kubectl apply`:

### 2a. WorkerPool — Physical Compute Capacity

```yaml
apiVersion: ate.dev/v1alpha1
kind: WorkerPool
metadata:
  name: my-pool
  namespace: my-app
  labels:
    workload: my-agent        # ActorTemplates select pools by these labels
spec:
  replicas: 5                  # Number of warm, standby pods
  ateomImage: ko://github.com/agent-substrate/substrate/cmd/ateom-gvisor
  # sandboxClass: gvisor       # optional; gvisor (default) or microvm
```

**What happens:** The controller creates a Deployment with `replicas` pods. Each pod runs `ateom-gvisor` (privileged) and sits idle, waiting to host actors. Pools are selected by `ActorTemplate.spec.workerSelector` matching the pool's **labels** (there is no direct pool reference on the template).

### 2b. ActorTemplate — Workload Blueprint

```yaml
apiVersion: ate.dev/v1alpha1
kind: ActorTemplate
metadata:
  name: my-agent
  namespace: my-app
spec:
  # Root sandbox container (must be pinned @sha256)
  pauseImage: "registry.k8s.io/pause:3.10.2@sha256:f548e0e8e3dc..."

  # Your actual workload containers
  containers:
  - name: agent
    image: gcr.io/my-project/my-agent@sha256:abc123...  # MUST be pinned
    command: ["/app/server"]
    readyz:                    # readiness probe against the container's interior IP
      httpGet:
        path: /readyz
        port: 80
    volumeMounts:              # optional; mount durable volumes
    - name: data
      mountPath: /home/agent

  # sandboxClass: gvisor       # optional; gvisor (default) or microvm.
                               # Must match the eligible WorkerPools' class.

  # Which worker pools may host this template's actors (by pool label).
  workerSelector:
    matchLabels:
      workload: my-agent

  # Where to store memory + disk snapshots, and what to capture.
  snapshotsConfig:
    location: gs://my-bucket/snapshots/my-agent/
    onPause: Full              # Full = process memory + rootfs delta; Data = volumes only
    onCommit: Data             # must be a subset of onPause

  # Optional durable volumes (survive across snapshots).
  volumes:
  - name: data
    durableDir: {}
```

> **runsc / sandbox binaries are no longer in the `ActorTemplate`.** The gVisor
> `runsc` binary (and micro-VM assets) are fetched from a cluster-scoped
> `SandboxConfig` object, selected by `sandboxClass`. The base install ships a
> default gVisor `SandboxConfig`; you do not declare `runsc` per template.
>
> **The `ActorTemplate` spec is immutable.** To change the image or config,
> create a new template (a new revision), rather than patching in place.

**What happens automatically:**
1. Controller boots a "golden" actor from your image
2. Waits 20 seconds for initialization
3. Suspends it, creating a **Golden Snapshot** (Version 0)
4. Template enters `Ready` phase

**Wait for readiness:**
```bash
kubectl wait --for=condition=Ready actortemplate/my-agent -n my-app --timeout=5m
```

### Key constraints for your container image:
- **Must listen on a port** (typically :80) for HTTP traffic
- **Must be pinned** with `@sha256:...` digest (mutable tags invalidate snapshots)
- **Expensive initialization goes in startup** — it's captured in the golden snapshot and never repeated
- **No special SDK required** — any standard HTTP server works

---

## Step 3: Create Actor Instances

Actors are lightweight logical instances. You can create millions of them — they don't consume compute until activated. Every actor lives in an **atespace**, which must exist first.

### Create the atespace (once per isolation boundary):
```bash
kubectl ate create atespace my-space
```

### Via CLI:
```bash
# -a/--atespace is required and the atespace must already exist
kubectl ate create actor my-session-1 --template my-app/my-agent -a my-space
kubectl ate create actor my-session-2 --template my-app/my-agent -a my-space
kubectl ate create actor user-abc-workspace --template my-app/my-agent -a my-space
```

### Via gRPC API (programmatic):
```protobuf
// Service: ateapi.Control
rpc CreateAtespace(CreateAtespaceRequest) returns (CreateAtespaceResponse);
rpc CreateActor(CreateActorRequest) returns (CreateActorResponse);
```

```go
resp, err := client.CreateActor(ctx, &ateapipb.CreateActorRequest{
    ActorRef:               &ateapipb.ActorRef{Atespace: "my-space", Name: "my-session-1"},
    ActorTemplateNamespace: "my-app",
    ActorTemplateName:      "my-agent",
})
// resp.Actor.Status == STATUS_SUSPENDED
```

**Actor ID rules:**
- 1–63 characters
- Lowercase alphanumeric + hyphens only
- Must start and end with alphanumeric
- Must be unique **within its atespace** (the `(atespace, actor-id)` pair is the identity)

**Atespace rules:** an atespace name is a DNS-1123 label (same rules as an actor id). Creating an actor in a non-existent atespace fails with `FailedPrecondition`. Deleting an atespace only succeeds when it is empty.

**After creation:** The actor exists in the state store as `STATUS_SUSPENDED`. Zero compute consumed.

---

## Step 4: Send HTTP Traffic (Auto-Activation)

This is the magic. You just send an HTTP request with a special `Host` header, and Substrate handles everything:

### The Contract:

```
Host: <actor-id>.<atespace>.actors.resources.substrate.ate.dev
```

The atespace is part of the name because an actor id is only unique within its atespace.

### Example:

```bash
# Port-forward the router (for local dev)
kubectl port-forward -n ate-system svc/atenet-router 8000:80

# Send request — actor auto-resumes from snapshot
curl -X POST \
  -H "Host: my-session-1.my-space.actors.resources.substrate.ate.dev" \
  -d '{"message": "hello"}' \
  http://localhost:8000/
```

### What happens behind the scenes:

```
1. DNS resolves *.actors.resources.substrate.ate.dev → atenet-router Service IP
2. Envoy receives the request
3. ExtProc extracts "my-space" (atespace) and "my-session-1" (actor id) from the Host header
4. Router calls ateapi.ResumeActor({atespace: "my-space", name: "my-session-1"})
5. ateapi picks a free worker pod, downloads snapshot, restores process
6. Actor is now RUNNING with its original memory + disk state
7. Request is forwarded to the actor's port 80
8. Actor processes request and responds
9. Response flows back to the caller
```

**Latency:** The entire resume + forward takes ~100ms–2s depending on snapshot size.

**Subsequent requests:** If the actor is already `RUNNING`, routing is instant (no resume needed).

### HTTPS variant:
```bash
curl -X POST \
  -H "Host: my-session-1.my-space.actors.resources.substrate.ate.dev" \
  https://localhost:8443/   # port 8443 for TLS
```

---

## Step 5 (Optional): Lifecycle Management

All actor lifecycle commands require `-a/--atespace` because an actor is addressed by `(atespace, id)`.

### Explicit Suspend
```bash
kubectl ate suspend actor my-session-1 -a my-space
```
Checkpoints memory + disk → uploads to object storage → frees the worker pod.

### Pause (keep snapshot on the node)
```bash
kubectl ate pause actor my-session-1 -a my-space
```
Unlike suspend, pause keeps the actor's snapshot **local on the node VM** (`STATUS_PAUSED`) for a faster resume onto the same node, without uploading to object storage.

### Explicit Resume (without HTTP trigger)
```bash
kubectl ate resume actor my-session-1 -a my-space
```

### Self-Suspension (from inside the actor)
Your actor code can call the Substrate API directly to suspend itself when idle:

```go
// Inside your actor process
conn, _ := grpc.Dial("api.ate-system.svc.cluster.local:443", grpc.WithTransportCredentials(...))
client := ateapipb.NewControlClient(conn)
client.SuspendActor(ctx, &ateapipb.SuspendActorRequest{
    ActorRef: &ateapipb.ActorRef{Atespace: "my-space", Name: "my-session-1"},
})
// Process freezes here — next resume restores from this exact point
```

### Query State
```bash
kubectl ate get actors -a my-space     # List actors in one atespace
kubectl ate get actors -A              # List actors across all atespaces
kubectl ate get actor my-session-1 -a my-space  # Get specific
kubectl ate get atespaces              # List atespaces
kubectl ate get workers                # See worker pool status
```

### Delete (must be suspended first)
```bash
kubectl ate delete actor my-session-1 -a my-space
kubectl ate delete atespace my-space   # only when empty
```

### Observe
```bash
kubectl ate logs actors my-session-1 -a my-space      # View current logs
kubectl ate logs actors my-session-1 -a my-space -f   # Stream (follows across pod migrations)
```

---

## Complete gRPC API Contract

The public API is defined in `pkg/proto/ateapipb/ateapi.proto`:

```protobuf
service Control {
  rpc CreateActor(CreateActorRequest) returns (CreateActorResponse);
  rpc UpdateActor(UpdateActorRequest) returns (UpdateActorResponse);
  rpc ResumeActor(ResumeActorRequest) returns (ResumeActorResponse);
  rpc SuspendActor(SuspendActorRequest) returns (SuspendActorResponse);
  rpc PauseActor(PauseActorRequest) returns (PauseActorResponse);
  rpc DeleteActor(DeleteActorRequest) returns (DeleteActorResponse);
  rpc GetActor(GetActorRequest) returns (GetActorResponse);
  rpc ListActors(ListActorsRequest) returns (ListActorsResponse);
  rpc ListWorkers(ListWorkersRequest) returns (ListWorkersResponse);

  // Atespaces (the isolation boundary actors live in)
  rpc CreateAtespace(CreateAtespaceRequest) returns (CreateAtespaceResponse);
  rpc GetAtespace(GetAtespaceRequest) returns (GetAtespaceResponse);
  rpc ListAtespaces(ListAtespacesRequest) returns (ListAtespacesResponse);
  rpc DeleteAtespace(DeleteAtespaceRequest) returns (DeleteAtespaceResponse);

  rpc DebugClear(DebugClearRequest) returns (DebugClearResponse); // wipe DB (testing)
}

service SessionIdentity {
  rpc MintJWT(MintJWTRequest) returns (MintJWTResponse);    // Get actor identity JWT
  rpc MintCert(MintCertRequest) returns (MintCertResponse); // Get actor identity mTLS cert
}
```

All actor-scoped RPCs (`Get/Create/Update/Suspend/Pause/Resume/Delete`) identify the actor with an `ActorRef { atespace, name }`.

### Actor States (from caller's perspective):

| Status | Meaning | Can receive traffic? |
|--------|---------|---------------------|
| `SUSPENDED` | Idle, snapshot in object storage, no compute | No (auto-resumes on request) |
| `RESUMING` | Being restored onto a worker | No (request is held/queued) |
| `RUNNING` | Active on a worker pod | **Yes** |
| `SUSPENDING` | Being checkpointed to object storage | Depends on timing |
| `PAUSED` | Idle, snapshot kept **local on the node VM** | No (resumes from the local snapshot) |
| `PAUSING` | Being checkpointed to a local (node-VM) snapshot | Depends on timing |

---

## HTTP Routing Contract Detail

### Request format:
```
Method: Any (GET, POST, PUT, DELETE, etc.)
URL: http(s)://<router-endpoint>/<any-path>
Host: <actor-id>.<atespace>.actors.resources.substrate.ate.dev
Body: Anything your actor expects
Headers: Anything — all forwarded to the actor
```

### What your actor sees:
- A normal HTTP request on port 80 (or whatever port you configured)
- Original path, query params, body, and headers are preserved
- The `Host` header is rewritten to the worker pod IP (internal routing detail)

### Response:
- Whatever your actor's HTTP server responds with
- All response headers and body flow back to the original caller

### Error responses from Substrate (before reaching your actor):

| HTTP Status | Meaning |
|-------------|---------|
| 404 | Actor (or its atespace) not found, or the Host header is not a valid actor DNS name |
| 503 | No free workers available |
| 500 | Internal restore failure |
| 504 | Resume timed out |

---

## Programmatic Client (Go)

If you want to manage actors programmatically (not just via HTTP traffic):

```go
import (
    "github.com/agent-substrate/substrate/internal/ateclient"
    "github.com/agent-substrate/substrate/pkg/proto/ateapipb"
)

// Connect (auto-discovers via K8s port-forward)
client, err := ateclient.NewClient(ctx, "", "", "", false)
defer client.Close()

// Create the atespace (once), then the actor in it
client.CreateAtespace(ctx, &ateapipb.CreateAtespaceRequest{Name: "my-space"})
client.CreateActor(ctx, &ateapipb.CreateActorRequest{
    ActorRef:               &ateapipb.ActorRef{Atespace: "my-space", Name: "agent-42"},
    ActorTemplateNamespace: "my-app",
    ActorTemplateName:      "my-agent",
})

ref := &ateapipb.ActorRef{Atespace: "my-space", Name: "agent-42"}

// Resume (explicit, not needed if using HTTP auto-resume)
client.ResumeActor(ctx, &ateapipb.ResumeActorRequest{ActorRef: ref})

// Suspend
client.SuspendActor(ctx, &ateapipb.SuspendActorRequest{ActorRef: ref})

// Query
resp, _ := client.GetActor(ctx, &ateapipb.GetActorRequest{ActorRef: ref})
fmt.Println(resp.Actor.Status)  // STATUS_RUNNING, STATUS_SUSPENDED, etc.

// Delete
client.DeleteActor(ctx, &ateapipb.DeleteActorRequest{ActorRef: ref})
```

---

## Session Identity (Advanced)

Actors can obtain stable identity credentials that persist across pod migrations:

### JWT:
```go
// From inside your actor process:
resp, _ := identityClient.MintJWT(ctx, &ateapipb.MintJWTRequest{
    AppId:     "my-app",
    UserId:    "user-123",
    SessionId: "session-456",
    Audience:  []string{"https://my-downstream-service.example.com"},
})
// resp.SessionJwt is an OIDC-compatible JWT with claims:
//   sub: "apps/my-app/users/user-123/sessions/session-456"
//   aud: ["https://my-downstream-service.example.com"]
//   ate.dev: {appID, userID, sessionID}
```

### mTLS Certificate:
```go
resp, _ := identityClient.MintCert(ctx, &ateapipb.MintCertRequest{
    AppId:                      "my-app",
    UserId:                     "user-123",
    SessionId:                  "session-456",
    CertificateSigningRequest:  csrDERBytes,
})
// resp.SessionCertificates contains DER-encoded cert chain
// URI SAN: spiffe://substrate-session.local/app/my-app/user/user-123/session/session-456
```

---

## End-to-End Example: Minimal Actor

Here's what a minimal actor looks like (the counter demo):

```go
package main

import (
    "fmt"
    "net/http"
    "sync/atomic"
)

var count uint64

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        n := atomic.AddUint64(&count, 1)
        fmt.Fprintf(w, "count: %d\n", n)
    })
    http.ListenAndServe(":80", nil)
}
```

That's it. No SDK. No special imports. Just a standard HTTP server. The `count` variable persists in memory across suspends/resumes because the entire process state is checkpointed.

---

## Summary: Caller's Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│  As a consumer, you interact with 3 interfaces:                 │
│                                                                 │
│  1. kubectl apply    → CRDs (WorkerPool, ActorTemplate)         │
│     "Here's my workload definition and how much compute I want" │
│                                                                 │
│  2. kubectl ate / gRPC API → Actor lifecycle                    │
│     "Create/suspend/delete logical actor instances"             │
│                                                                 │
│  3. HTTP requests   → Host: <id>.<atespace>.actors.resources... │
│     "Just send traffic — Substrate handles the rest"            │
│                                                                 │
│  Everything else (scheduling, snapshotting, restore, routing)   │
│  is invisible to you.                                           │
└─────────────────────────────────────────────────────────────────┘
```
