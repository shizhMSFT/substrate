# 追踪（Tracing）最佳实践

本文档概述了 Agent Substrate 项目中使用 OpenTelemetry 进行追踪（tracing）的最佳实践。

## 我们为什么需要追踪？

追踪对于调试和性能优化非常重要。它让你能够看到一个请求是如何被处理的，以及它可能在哪里变慢。

## 什么是追踪？

追踪是一种跟踪请求在系统中流动过程的方法。它让你能够看到每一步花费了多长时间，以及瓶颈出现在哪里。

理想情况下，追踪会勾勒出一个请求从客户端到服务器再返回的完整流程，并包含所有被调用的服务。

追踪由 **span** 和 **trace** 组成。span 是单个操作，而 trace 是一组彼此相关的 span 的集合。
span 有开始时间和结束时间，并且可以带有属性（键值对），用于提供有关该操作的附加信息。
trace 有一个 trace ID 和一个 span ID，用于标识该 trace 和 span。

## 追踪的工作原理

追踪数据保存在 Golang 的 context 对象中，从而使状态能够在调用栈中传播。

当发起 HTTP 请求时，追踪数据可能会包含在 HTTP 头中。对于 gRPC，追踪数据则包含在 metadata 对象中。
Otel 中间件会自动处理追踪数据的提取（extraction）和注入（injection）。

服务器有一个 exporter 服务，它会将 span 批量处理并推送到远端 collector 以供分析。

## 实现追踪

### 服务器端

所有服务器都需要初始化一个 OpenTelemetry exporter 和 tracer provider。可参考 `cmd/ateapi/ateapi.go:initTracing()` 中的示例：

```go
func initTracing(ctx context.Context) (*sdktrace.TracerProvider, error) {
	exporter, err := otlptracegrpc.New(ctx,
		// GKE managed traces doesn't support validating the TLS certs of the collector
		otlptracegrpc.WithInsecure(),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create OTLP exporter: %w", err)
	}

	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName("ateapi"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create resource: %w", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		// Only trace on-demand when signaled by the client (e.g. via --trace flag)
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.NeverSample())),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})

	return tp, nil
}
```

在调用 `initTracing()` 函数时，请务必在调用后 `defer tp.Shutdown(ctx)`，以确保在服务器退出时正确关闭 tracer provider：

```go
defer func() {
  if err := tp.Shutdown(ctx); err != nil {
    slog.Error("Failed to shutdown TracerProvider", slog.Any("err", err))
  }
}()
```

请注意以下重要特性：

* 我们不会验证 collector 的 TLS 证书
* 我们向 exporter 提供一个服务名，用于标识是哪个进程在发出这些 span
* 我们仅在客户端需要时按需追踪，这由请求中是否存在追踪 metadata/头来决定
  * 对于生产环境，出于安全目的，我们需要对可以启用追踪的对象/方式进行门控

你的服务器的 YAML manifest 应包含 `OTEL_EXPORTER_OTLP_ENDPOINT` 环境变量，将 exporter 指向 GKE 的托管追踪 collector，例如：

```yaml
      containers:
        - name: ateapi
          image: ko://github.com/agent-substrate/substrate/cmd/ateapi
          ports:
            - "443:443"
          env:
            # Tracing related environment variables
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://opentelemetry-collector.gke-managed-otel.svc.cluster.local:4317"
```

#### gRPC 服务器
在实现 gRPC 服务器时，你应该包含以下中间件来处理追踪：

```go
server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler())
)
```

#### HTTP 服务器
在实现 HTTP 服务器时，你应该用 `otelhttp.NewHandler` 包装根多路复用器（root multiplexer）：

```go
tracedMux := otelhttp.NewHandler(
    mux,
    "/",
)
```

虽然这种模型确保了所有请求都有资格被追踪，但它不会将请求的性质添加到 span 中。因此，你应该在处理器中创建一个 span 来捕获请求的性质：

```go
tracer := otel.Tracer("my-server-name")

func someHandler(w http.ResponseWriter, r *http.Request) {
  ctx, span := tracer.Start(r.Context(), "operationIdentifier")
  defer span.End()
  // ... rest of your handler
}
```

#### 子 Span（Sub-Spans）
如果你希望提供对服务器内部运作的可见性，可以在任意位置创建子 span：

```go
tracer := otel.Tracer("my-package-name")

func someFunc(ctx context.Context) {
  ctx, span := tracer.Start(ctx, "operationIdentifier")
  defer span.End()
}
```

### 客户端

客户端无需实例化 exporter，但它们应提供一个选项，允许在请求中包含追踪 metadata，从而让用户能够发起一次追踪。

#### Golang

与服务器一样，tracer provider 必须被初始化和关闭，但不需要 exporter（注意 sampler 的切换）：

```go
func initTracing(ctx context.Context, enabled bool) (*sdktrace.TracerProvider, error) {
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.UserAgentOriginal("my-client-name"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create resource: %w", err)
	}

	sampler := sdktrace.NeverSample()
	if enabled {
		sampler = sdktrace.AlwaysSample()
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sampler),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})

	return tp, nil
}
```

如果你的服务器同时也是一个客户端，则此步骤是多余的，可以省略。

请注意，我们在这里设置 UserAgentOriginal 属性，是因为我们假定这是一个面向用户的客户端。
如果这是一个系统服务，我们必须改为设置 ServiceName 属性。

##### gRPC 客户端

在使用 gRPC 客户端时，包含 stats handler：

```go
clientConn, err := grpc.NewClient(
    serverAddr,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

##### HTTP 客户端

对于 HTTP 客户端，将 Otel 的 transport 包装器添加到你的 transport 中：

```go
client := &http.Client{
  Transport: otelhttp.NewTransport(http.DefaultTransport),
}
```

#### Python

与 Go 一样，provider 必须被初始化（注意，由于 Python 仅用于负载测试，我们使用基于概率的追踪）：

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.propagate import set_global_textmap, inject
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

def init_tracing(probability: float = 1.0):
  sampler = TraceIdRatioBased(probability)
  resource = Resource(attributes={
      SERVICE_NAME: "my-locust-service"
  })
  provider = TracerProvider(sampler=sampler, resource=resource)

  trace.set_tracer_provider(provider)
  set_global_textmap(TraceContextTextMapPropagator())
```

##### gRPC 客户端

在使用 gRPC 客户端时，只需实例化一个 span 并注入头，将它们作为 metadata 发送：

```python
from opentelemetry import trace
from opentelemetry.propagate import inject

tracer = trace.get_tracer("my-service")

def call_with_trace(stub, method, request):
  with tracer.start_as_current_span("operationIdentifier") as span:
    headers = {}
    inject(headers)
    metadata = list(headers.items())
    response = stub.GetActor(
        ateapi_pb2.GetActorRequest(actor_key="my-actor"),
        metadata=metadata
    )
```

##### HTTP 客户端

对于 HTTP 客户端，实例化一个 span 并将头注入到 HTTP 请求中：

```python
from opentelemetry import trace
from opentelemetry.propagate import inject

tracer = trace.get_tracer("my-service")

def call_with_trace(stub, method, request):
  with tracer.start_as_current_span("operationIdentifier") as span:
    headers = {}
    inject(headers)
    response = requests.get(
        "http://example.com",
        headers=headers
    )
```
