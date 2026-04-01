---
name: eventuous-go-kurrentdb
description: "This skill should be used when configuring KurrentDB or EventStoreDB integration with Eventuous Go. Covers the KurrentDB event store implementation, catch-up subscriptions ($all and stream), persistent subscriptions, checkpoint management, and Docker setup. Common triggers: 'kurrentdb go', 'go event store', 'go catch-up subscription', 'go persistent subscription', 'kurrentdb.NewStore', 'kurrentdb.NewCatchUp', 'kurrentdb.NewPersistent', 'github.com/eventuous/eventuous-go/kurrentdb'."
---
# Eventuous Go — KurrentDB Integration

Infrastructure-specific guidance for using Eventuous Go with KurrentDB (EventStoreDB) as the event store and subscription source.

**Module:** `github.com/eventuous/eventuous-go/kurrentdb`

**Install:**

```bash
go get github.com/eventuous/eventuous-go/kurrentdb
```

## Running KurrentDB

Docker for local development:

```bash
docker run -d --name kurrentdb \
  -p 2113:2113 \
  -e EVENTSTORE_INSECURE=true \
  -e EVENTSTORE_MEM_DB=true \
  docker.kurrent.io/kurrent/kurrentdb:latest
```

Connection string: `kurrentdb://localhost:2113?tls=false`

## Event Store

`kurrentdb.Store` implements `store.EventStore` (combining `EventReader`, `EventWriter`, and stream management).

```go
import (
    kdb "github.com/kurrent-io/KurrentDB-Client-Go/kurrentdb"
    "github.com/eventuous/eventuous-go/kurrentdb"
    "github.com/eventuous/eventuous-go/core/codec"
)

// Connect to KurrentDB
settings, _ := kdb.ParseConnectionString("kurrentdb://localhost:2113?tls=false")
client, _ := kdb.NewClient(settings)

// Create codec
types := codec.NewTypeMap()
codec.Register[RoomBooked](types, "RoomBooked")
codec.Register[PaymentRecorded](types, "PaymentRecorded")
jsonCodec := codec.NewJSON(types)

// Create event store
eventStore := kurrentdb.NewStore(client, jsonCodec)
```

### Operations

**Append events** (optimistic concurrency):

```go
result, err := eventStore.AppendEvents(ctx,
    eventuous.NewStreamName("Booking", "booking-123"),
    eventuous.VersionNoStream,  // stream must not exist
    []store.NewStreamEvent{
        {ID: uuid.New(), Payload: RoomBooked{...}, Metadata: eventuous.Metadata{}},
    },
)
// result.GlobalPosition, result.NextExpectedVersion
```

**Read events** forward:

```go
events, err := eventStore.ReadEvents(ctx,
    eventuous.NewStreamName("Booking", "booking-123"),
    0,    // start position
    100,  // max count
)
```

**Read events** backward:

```go
events, err := eventStore.ReadEventsBackwards(ctx,
    eventuous.NewStreamName("Booking", "booking-123"),
    ^uint64(0),  // from end
    10,           // last 10 events
)
```

**Stream existence, delete, truncate:**

```go
exists, err := eventStore.StreamExists(ctx, stream)
err := eventStore.DeleteStream(ctx, stream, eventuous.VersionAny)
err := eventStore.TruncateStream(ctx, stream, position, eventuous.VersionAny)
```

### Optimistic Concurrency Mapping

| Eventuous | KurrentDB |
|---|---|
| `VersionNoStream` (-1) | `NoStream` — stream must not exist |
| `VersionAny` (-2) | `Any` — skip concurrency check |
| `N` (>= 0) | `StreamRevision(N)` — exact version match |

On conflict, returns `eventuous.ErrOptimisticConcurrency`.

---

## Catch-Up Subscriptions

`kurrentdb.CatchUp` reads events from a starting position and follows the stream in real-time. Requires a handler; optionally uses a checkpoint store for resume.

```go
import "github.com/eventuous/eventuous-go/kurrentdb"

sub := kurrentdb.NewCatchUp(client, jsonCodec, "BookingsProjection",
    kurrentdb.FromAll(),
    kurrentdb.WithHandler(subscription.HandlerFunc(
        func(ctx context.Context, msg *subscription.ConsumeContext) error {
            switch e := msg.Payload.(type) {
            case RoomBooked:
                // project booking into read model
            case PaymentRecorded:
                // update payment status
            }
            return nil
        },
    )),
    kurrentdb.WithCheckpointStore(myCheckpointStore),
    kurrentdb.WithMiddleware(
        subscription.WithLogging(slog.Default()),
        subscription.WithPartitioning(4, nil),
    ),
)

// Blocks until context cancelled
err := sub.Start(ctx)
```

### Source Options

**Subscribe to $all stream** (default — cross-aggregate projections):

```go
kurrentdb.FromAll()
```

**Subscribe to a specific stream:**

```go
kurrentdb.FromStream(eventuous.NewStreamName("Booking", "booking-123"))
```

### CatchUp Options

| Option | Description |
|---|---|
| `FromAll()` | Subscribe to `$all` stream (default) |
| `FromStream(name)` | Subscribe to a specific stream |
| `WithHandler(h)` | Set event handler (**required**) |
| `WithCheckpointStore(cs)` | Enable checkpoint persistence for resume |
| `WithMiddleware(mw...)` | Add middleware to handler chain |
| `WithResolveLinkTos(bool)` | Resolve link events to targets (default: false) |
| `WithFilter(filter)` | Server-side event filter for `$all` subscriptions |

### Filtering ($all subscriptions)

Use KurrentDB server-side filters to limit events:

```go
kurrentdb.NewCatchUp(client, jsonCodec, "FilteredSub",
    kurrentdb.FromAll(),
    kurrentdb.WithHandler(handler),
    kurrentdb.WithFilter(&kdb.SubscriptionFilter{
        Type:     kdb.EventFilterType,
        Prefixes: []string{"RoomBooked", "PaymentRecorded"},
    }),
)
```

### Behavior Notes

- System events (types starting with `$`) are always skipped
- Events that fail to decode have `Payload` set to `nil` (not skipped)
- If no checkpoint store is configured, subscription starts from the beginning each time
- `Start()` blocks until the context is cancelled or a fatal error occurs

---

## Persistent Subscriptions

`kurrentdb.Persistent` uses server-managed consumer groups. KurrentDB tracks the checkpoint — no external checkpoint store needed. The subscription group is auto-created on first `Start()`.

```go
sub := kurrentdb.NewPersistent(client, jsonCodec, "PaymentIntegration",
    kurrentdb.PersistentFromStream(eventuous.NewStreamName("Payment", "")),
    kurrentdb.PersistentWithHandler(subscription.HandlerFunc(
        func(ctx context.Context, msg *subscription.ConsumeContext) error {
            // process event
            return nil
        },
    )),
    kurrentdb.PersistentWithMiddleware(
        subscription.WithLogging(slog.Default()),
    ),
)

err := sub.Start(ctx)
```

### Persistent Options

| Option | Description |
|---|---|
| `PersistentFromAll()` | Subscribe to `$all` stream (default) |
| `PersistentFromStream(name)` | Subscribe to a specific stream |
| `PersistentWithHandler(h)` | Set event handler (**required**) |
| `PersistentWithMiddleware(mw...)` | Add middleware to handler chain |
| `PersistentWithBufferSize(size)` | Connection buffer size |
| `PersistentWithFilter(filter)` | Server-side filter for `$all` subscriptions |

### ACK/NACK Behavior

- **Success:** event is ACKed automatically
- **Handler error:** event is NACKed with `NackActionRetry`
- **System events:** skipped but ACKed

### Catch-Up vs Persistent

| | Catch-Up | Persistent |
|---|---|---|
| **Checkpoint** | Client-managed (external store) | Server-managed |
| **Consumer groups** | No | Yes |
| **Retry** | Replay from checkpoint | Server-side NACK/retry |
| **Use case** | Projections, read models | Integration, competing consumers |

---

## Complete Example

```go
package main

import (
    "context"
    "log/slog"

    kdb "github.com/kurrent-io/KurrentDB-Client-Go/kurrentdb"
    eventuous "github.com/eventuous/eventuous-go/core"
    "github.com/eventuous/eventuous-go/core/codec"
    "github.com/eventuous/eventuous-go/core/command"
    "github.com/eventuous/eventuous-go/core/store"
    "github.com/eventuous/eventuous-go/core/subscription"
    "github.com/eventuous/eventuous-go/kurrentdb"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 1. Connect to KurrentDB
    settings, _ := kdb.ParseConnectionString("kurrentdb://localhost:2113?tls=false")
    client, _ := kdb.NewClient(settings)

    // 2. Register event types and create codec
    types := codec.NewTypeMap()
    codec.Register[RoomBooked](types, "RoomBooked")
    codec.Register[PaymentRecorded](types, "PaymentRecorded")
    jsonCodec := codec.NewJSON(types)

    // 3. Create event store
    eventStore := kurrentdb.NewStore(client, jsonCodec)

    // 4. Create command service
    svc := command.New[BookingState](eventStore, eventStore, types, BookingFold, BookingState{})

    command.On(svc, command.Handler[BookRoom, BookingState]{
        Expected: eventuous.IsNew,
        Stream:   func(cmd BookRoom) eventuous.StreamName {
            return eventuous.NewStreamName("Booking", cmd.BookingID)
        },
        Act: func(ctx context.Context, state BookingState, cmd BookRoom) ([]any, error) {
            return []any{RoomBooked{
                BookingID: cmd.BookingID,
                RoomID:    cmd.RoomID,
                Price:     cmd.Price,
                Currency:  cmd.Currency,
            }}, nil
        },
    })

    // 5. Handle a command
    result, _ := svc.Handle(ctx, BookRoom{
        BookingID: "booking-1",
        RoomID:    "room-42",
        Price:     200.0,
        Currency:  "EUR",
    })
    slog.Info("booked", "version", result.StreamVersion)

    // 6. Start catch-up subscription
    sub := kurrentdb.NewCatchUp(client, jsonCodec, "BookingsProjection",
        kurrentdb.FromAll(),
        kurrentdb.WithHandler(subscription.HandlerFunc(
            func(ctx context.Context, msg *subscription.ConsumeContext) error {
                slog.Info("event received",
                    "type", msg.EventType,
                    "stream", msg.Stream,
                    "position", msg.GlobalPosition,
                )
                return nil
            },
        )),
        kurrentdb.WithMiddleware(subscription.WithLogging(slog.Default())),
    )

    _ = sub.Start(ctx) // blocks
}
```

## Testing with Testcontainers

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
)

req := testcontainers.ContainerRequest{
    Image:        "docker.kurrent.io/kurrent/kurrentdb:latest",
    ExposedPorts: []string{"2113/tcp"},
    Env: map[string]string{
        "EVENTSTORE_INSECURE": "true",
        "EVENTSTORE_MEM_DB":   "true",
    },
    WaitingFor: wait.ForHTTP("/health/live").WithPort("2113/tcp"),
}
```
