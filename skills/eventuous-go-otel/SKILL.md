---
name: eventuous-go-otel
description: "This skill should be used when adding OpenTelemetry observability to Eventuous Go applications. Covers command tracing and metrics, subscription tracing middleware, span and metric naming conventions. Common triggers: 'eventuous go tracing', 'eventuous go metrics', 'go opentelemetry', 'otel.TraceCommands', 'otel.TracingMiddleware', 'github.com/eventuous/eventuous-go/otel'."
---
# Eventuous Go — OpenTelemetry Integration

Observability for Eventuous Go applications using OpenTelemetry tracing and metrics.

**Module:** `github.com/eventuous/eventuous-go/otel`

**Install:**

```bash
go get github.com/eventuous/eventuous-go/otel
```

**Dependencies:** `go.opentelemetry.io/otel` (trace + metric APIs)

## Command Tracing & Metrics

`otel.TraceCommands` wraps a command service (functional or aggregate-based) with OpenTelemetry tracing and metrics.

```go
import (
    "github.com/eventuous/eventuous-go/core/command"
    "github.com/eventuous/eventuous-go/otel"
    "go.opentelemetry.io/otel/trace"
    "go.opentelemetry.io/otel/metric"
)

// Create the base command service
svc := command.New(reader, writer, types, BookingFold, BookingState{})
command.On(svc, command.Handler[BookRoom, BookingState]{...})

// Wrap with tracing and metrics
tracedSvc := otel.TraceCommands(svc, tracer, meter)

// Use tracedSvc.Handle() — same interface as the original service
result, err := tracedSvc.Handle(ctx, BookRoom{...})
```

### What Gets Recorded

**Traces:**
- Span name: `command/{CommandType}` (e.g., `command/BookRoom`)
- Span attribute: `eventuous.command_type` — the command struct type name
- On success: span status set to `Ok`
- On error: span status set to `Error`, error recorded on span

**Metrics:**
- `eventuous.command.duration` (Float64Histogram, seconds) — command handling duration, with attribute `command_type`
- `eventuous.command.errors` (Int64Counter) — error count, with attribute `command_type`

### Works with Both Service Types

`TraceCommands` accepts the `command.CommandHandler[S]` interface, which both `command.Service[S]` and `command.AggregateService[S]` implement:

```go
// Functional service
tracedFunctional := otel.TraceCommands(functionalSvc, tracer, meter)

// Aggregate service
tracedAggregate := otel.TraceCommands(aggregateSvc, tracer, meter)
```

---

## Subscription Tracing Middleware

`otel.TracingMiddleware` creates a subscription middleware that traces event processing.

```go
import (
    "github.com/eventuous/eventuous-go/core/subscription"
    "github.com/eventuous/eventuous-go/otel"
)

handler := subscription.HandlerFunc(func(ctx context.Context, msg *subscription.ConsumeContext) error {
    // process event
    return nil
})

// Add tracing middleware to handler chain
handler = subscription.Chain(handler,
    otel.TracingMiddleware(tracer),
    subscription.WithLogging(slog.Default()),
)
```

### What Gets Recorded

**Traces:**
- Span name: `event/{EventType}` (e.g., `event/RoomBooked`)
- Span attributes:
  - `eventuous.event_type` — event type name
  - `eventuous.stream` — stream name
  - `eventuous.position` — event position in stream
- On success: span status set to `Ok`
- On error: span status set to `Error`, error recorded on span

### Usage with KurrentDB Subscriptions

```go
sub := kurrentdb.NewCatchUp(client, jsonCodec, "BookingsProjection",
    kurrentdb.FromAll(),
    kurrentdb.WithHandler(handler),
    kurrentdb.WithMiddleware(
        otel.TracingMiddleware(tracer),
        subscription.WithLogging(slog.Default()),
        subscription.WithPartitioning(4, nil),
    ),
)
```

Middleware order matters — `Chain` applies outermost first. Place `TracingMiddleware` first so the span covers all inner middleware and the handler.

---

## Complete Setup Example

```go
package main

import (
    "context"
    "log/slog"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"

    eventuous "github.com/eventuous/eventuous-go/core"
    "github.com/eventuous/eventuous-go/core/command"
    "github.com/eventuous/eventuous-go/core/subscription"
    eventuousotel "github.com/eventuous/eventuous-go/otel"
)

func main() {
    ctx := context.Background()

    // 1. Set up OpenTelemetry trace provider
    traceExporter, _ := otlptracegrpc.New(ctx)
    tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(traceExporter))
    defer tp.Shutdown(ctx)
    otel.SetTracerProvider(tp)

    // 2. Set up OpenTelemetry meter provider
    metricExporter, _ := otlpmetricgrpc.New(ctx)
    mp := sdkmetric.NewMeterProvider(sdkmetric.WithReader(
        sdkmetric.NewPeriodicReader(metricExporter),
    ))
    defer mp.Shutdown(ctx)
    otel.SetMeterProvider(mp)

    // 3. Create tracer and meter
    tracer := tp.Tracer("my-service")
    meter := mp.Meter("my-service")

    // 4. Wrap command service with tracing
    svc := command.New(reader, writer, types, BookingFold, BookingState{})
    command.On(svc, command.Handler[BookRoom, BookingState]{...})
    tracedSvc := eventuousotel.TraceCommands(svc, tracer, meter)

    // 5. Create subscription handler with tracing middleware
    handler := subscription.HandlerFunc(func(ctx context.Context, msg *subscription.ConsumeContext) error {
        // handle event
        return nil
    })

    tracedHandler := subscription.Chain(handler,
        eventuousotel.TracingMiddleware(tracer),
        subscription.WithLogging(slog.Default()),
    )

    // Use tracedSvc and tracedHandler in your application
    _ = tracedSvc
    _ = tracedHandler
}
```

## Metric and Span Reference

| Component | Span Name | Metrics |
|---|---|---|
| Command service | `command/{CommandType}` | `eventuous.command.duration` (histogram), `eventuous.command.errors` (counter) |
| Subscription handler | `event/{EventType}` | (none — tracing only) |

| Attribute | Used In | Description |
|---|---|---|
| `eventuous.command_type` | Command spans + metrics | Command struct type name |
| `eventuous.event_type` | Event spans | Event type name |
| `eventuous.stream` | Event spans | Stream name |
| `eventuous.position` | Event spans | Event position |
