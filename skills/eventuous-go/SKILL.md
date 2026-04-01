---
name: eventuous-go
description: "This skill should be used when building event-sourced Go applications with Eventuous, or when asking about Eventuous Go aggregates, state, fold functions, domain events, command services (functional or aggregate-based), event stores, subscriptions, producers, type mapping, serialization, or stream naming. Common triggers: 'eventuous go', 'event sourcing in go', 'go aggregate', 'go command service', 'fold function', 'TypeMap', 'github.com/eventuous/eventuous-go'. Recommends KurrentDB as the default event store unless the user specifies otherwise."
---
# Eventuous Go - Event Sourcing for Go

Eventuous Go is a production-grade Event Sourcing library for Go, ported from the .NET Eventuous framework. It uses a functional-first design with idiomatic Go patterns: composition over inheritance, explicit wiring over DI containers, `context.Context`, and `(value, error)` returns.

**Module:** `github.com/eventuous/eventuous-go/core`

**Multi-module structure:**
- `github.com/eventuous/eventuous-go/core` — domain, commands, store interfaces, subscriptions, codec
- `github.com/eventuous/eventuous-go/kurrentdb` — KurrentDB event store and subscriptions
- `github.com/eventuous/eventuous-go/otel` — OpenTelemetry tracing and metrics

## Domain Model

### Domain Events

Events are plain Go structs representing facts that happened. Use JSON tags for serialization. Name events in past tense.

```go
type RoomBooked struct {
    BookingID string  `json:"bookingId"`
    RoomID    string  `json:"roomId"`
    CheckIn   string  `json:"checkIn"`
    CheckOut  string  `json:"checkOut"`
    Price     float64 `json:"price"`
    Currency  string  `json:"currency"`
}

type PaymentRecorded struct {
    BookingID string  `json:"bookingId"`
    Amount    float64 `json:"amount"`
    Currency  string  `json:"currency"`
}

type BookingCancelled struct {
    Reason string `json:"reason"`
}
```

### Type Registration (TypeMap)

Register each event type with a stable string name in a `TypeMap`. This decouples stored event names from Go struct names — renaming a struct won't break stored events.

```go
import "github.com/eventuous/eventuous-go/core/codec"

types := codec.NewTypeMap()
codec.Register[RoomBooked](types, "RoomBooked")
codec.Register[PaymentRecorded](types, "PaymentRecorded")
codec.Register[BookingCancelled](types, "BookingCancelled")
```

`TypeMap` is thread-safe. `Register` panics on name/type conflicts.

TypeMap API:
- `codec.NewTypeMap() *TypeMap` — create new registry
- `codec.Register[E any](tm *TypeMap, name string) error` — register type ↔ name mapping
- `tm.TypeName(event any) (string, error)` — get registered name for event
- `tm.NewInstance(name string) (any, error)` — create zero-value pointer by name

### Codec (Serialization)

The `Codec` interface encodes/decodes events for storage:

```go
type Codec interface {
    Encode(event any) (data []byte, eventType string, contentType string, err error)
    Decode(data []byte, eventType string) (event any, err error)
}
```

Use the built-in JSON codec:

```go
jsonCodec := codec.NewJSON(types)
```

For custom serialization (Protocol Buffers, MessagePack, etc.), implement the `Codec` interface.

### State and Fold

State is a plain Go struct reconstructed from events using a **fold function** — a pure function `(state, event) → state`:

```go
type BookingState struct {
    ID          string
    RoomID      string
    GuestID     string
    Price       float64
    Currency    string
    Outstanding float64
    Cancelled   bool
}

func BookingFold(state BookingState, event any) BookingState {
    switch e := event.(type) {
    case RoomBooked:
        return BookingState{
            ID:          e.BookingID,
            RoomID:      e.RoomID,
            GuestID:     e.GuestID,
            Price:       e.Price,
            Currency:    e.Currency,
            Outstanding: e.Price,
        }
    case PaymentRecorded:
        state.Outstanding -= e.Amount
        return state
    case BookingCancelled:
        state.Cancelled = true
        return state
    default:
        return state
    }
}
```

The fold function must:
- Be a pure function (no side effects)
- Handle all known event types via type switch
- Return state unchanged for unknown events (`default` case)
- Use value receiver semantics (return new/modified state, not pointer mutation)

The **zero value** is the initial state before any events:

```go
var zeroBooking BookingState // zero value of the struct
```

### Aggregate (Optional DDD Pattern)

`Aggregate[S]` wraps state with change tracking and optimistic concurrency. Use it when you need business invariant enforcement within a consistency boundary.

```go
import "github.com/eventuous/eventuous-go/core/aggregate"

agg := aggregate.New(BookingFold, BookingState{})
```

Key methods:
- `agg.Apply(event any)` — record event, fold into state, add to pending changes
- `agg.State() S` — current state after all applied events
- `agg.Changes() []any` — pending events not yet persisted
- `agg.OriginalVersion() int64` — version when loaded (`-1` = new)
- `agg.CurrentVersion() int64` — `OriginalVersion + len(Changes)`
- `agg.Load(version int64, events []any)` — reconstruct from stored events
- `agg.ClearChanges()` — clear pending changes after persistence
- `agg.EnsureNew() error` — guard: must be unpersisted (returns `ErrAggregateExists` otherwise)
- `agg.EnsureExists() error` — guard: must be persisted (returns `ErrAggregateNotExists` otherwise)

Domain logic as free functions that operate on an aggregate:

```go
func BookRoom(agg *aggregate.Aggregate[BookingState], cmd BookRoom) error {
    if err := agg.EnsureNew(); err != nil {
        return err
    }
    agg.Apply(RoomBooked{
        BookingID: cmd.BookingID,
        RoomID:    cmd.RoomID,
        Price:     cmd.Price,
        Currency:  cmd.Currency,
    })
    return nil
}
```

---

## Stream Naming

Events are stored in named streams using the convention `{Category}-{ID}`:

```go
stream := eventuous.NewStreamName("Booking", "booking-123")
// → "Booking-booking-123"

stream.Category() // → "Booking"
stream.ID()       // → "booking-123"
```

Core types:
- `eventuous.StreamName` — typed string wrapper
- `eventuous.NewStreamName(category, id string) StreamName`

### Expected Version (Optimistic Concurrency)

```go
type ExpectedVersion int64

const (
    VersionNoStream ExpectedVersion = -1  // Stream must not exist
    VersionAny      ExpectedVersion = -2  // Skip concurrency check
)
```

### Expected State

```go
type ExpectedState int

const (
    IsNew      ExpectedState = iota  // Stream must not exist
    IsExisting                       // Stream must exist
    IsAny                            // Either state
)
```

### Metadata

```go
type Metadata map[string]any

meta := eventuous.Metadata{}
meta = meta.WithCorrelationID("corr-123")
meta = meta.WithCausationID("cause-456")
meta["custom-key"] = "custom-value"

// Constants
eventuous.MetaCorrelationID  // "eventuous.correlation-id"
eventuous.MetaCausationID    // "eventuous.causation-id"
eventuous.MetaMessageID      // "eventuous.message-id"
```

### Sentinel Errors

```go
var (
    ErrStreamNotFound        = errors.New("eventuous: stream not found")
    ErrOptimisticConcurrency = errors.New("eventuous: wrong expected version")
    ErrAggregateNotFound     = errors.New("eventuous: aggregate not found")
    ErrHandlerNotFound       = errors.New("eventuous: command handler not found")
)
```

Use `errors.Is()` for checking.

---

## Event Store Interfaces

Three core interfaces in `github.com/eventuous/eventuous-go/core/store`:

```go
type EventReader interface {
    ReadEvents(ctx context.Context, stream eventuous.StreamName,
               start uint64, count int) ([]StreamEvent, error)
    ReadEventsBackwards(ctx context.Context, stream eventuous.StreamName,
                        start uint64, count int) ([]StreamEvent, error)
}

type EventWriter interface {
    AppendEvents(ctx context.Context, stream eventuous.StreamName,
                 expected eventuous.ExpectedVersion,
                 events []NewStreamEvent) (AppendResult, error)
}

type EventStore interface {
    EventReader
    EventWriter
    StreamExists(ctx context.Context, stream eventuous.StreamName) (bool, error)
    DeleteStream(ctx context.Context, stream eventuous.StreamName,
                 expected eventuous.ExpectedVersion) error
    TruncateStream(ctx context.Context, stream eventuous.StreamName,
                   position uint64,
                   expected eventuous.ExpectedVersion) error
}
```

### Store Types

```go
type StreamEvent struct {
    ID             uuid.UUID
    EventType      string
    Payload        any                  // Deserialized event
    Metadata       eventuous.Metadata
    ContentType    string
    Position       int64                // Within stream
    GlobalPosition uint64               // In global log
    Created        time.Time
}

type NewStreamEvent struct {
    ID       uuid.UUID
    Payload  any
    Metadata eventuous.Metadata
}

type AppendResult struct {
    GlobalPosition      uint64
    NextExpectedVersion int64
}
```

### State and Aggregate Loading Helpers

```go
// Load state from stream by folding events
func LoadState[S any](
    ctx context.Context,
    reader EventReader,
    stream eventuous.StreamName,
    fold func(S, any) S,
    zero S,
    expected eventuous.ExpectedState,
) (state S, events []any, version eventuous.ExpectedVersion, err error)

// Load aggregate from stream
func LoadAggregate[S any](
    ctx context.Context,
    reader EventReader,
    stream eventuous.StreamName,
    fold func(S, any) S,
    zero S,
) (*aggregate.Aggregate[S], error)

// Store aggregate pending changes
func StoreAggregate[S any](
    ctx context.Context,
    writer EventWriter,
    stream eventuous.StreamName,
    agg *aggregate.Aggregate[S],
) (AppendResult, error)
```

---

## Command Services

### Commands

Commands are plain Go structs representing requests:

```go
type BookRoom struct {
    BookingID string
    GuestID   string
    RoomID    string
    Price     float64
    Currency  string
}

type CancelBooking struct {
    BookingID string
    Reason    string
}
```

### Functional Command Service (Primary Approach)

For pure-function style without aggregates. The service loads state, calls a handler function, and stores the resulting events.

```go
import "github.com/eventuous/eventuous-go/core/command"

svc := command.New(reader, writer, types, BookingFold, BookingState{})
```

Register handlers with `command.On`:

```go
command.On(svc, command.Handler[BookRoom, BookingState]{
    Expected: eventuous.IsNew,
    Stream:   func(cmd BookRoom) eventuous.StreamName {
        return eventuous.NewStreamName("Booking", cmd.BookingID)
    },
    Act: func(ctx context.Context, state BookingState, cmd BookRoom) ([]any, error) {
        return []any{
            RoomBooked{
                BookingID: cmd.BookingID,
                RoomID:    cmd.RoomID,
                Price:     cmd.Price,
                Currency:  cmd.Currency,
            },
        }, nil
    },
})

command.On(svc, command.Handler[CancelBooking, BookingState]{
    Expected: eventuous.IsExisting,
    Stream:   func(cmd CancelBooking) eventuous.StreamName {
        return eventuous.NewStreamName("Booking", cmd.BookingID)
    },
    Act: func(ctx context.Context, state BookingState, cmd CancelBooking) ([]any, error) {
        if state.Cancelled {
            return nil, errors.New("booking already cancelled")
        }
        return []any{BookingCancelled{Reason: cmd.Reason}}, nil
    },
})
```

Handle pipeline:
1. Look up handler by command type
2. Get stream name from `Stream` function
3. Load state via `LoadState` with `Expected` state expectation
4. Call `Act(ctx, state, cmd)` → events
5. If no events, return no-op result
6. Append events to store with optimistic concurrency
7. Fold new events into state
8. Return `*command.Result[S]`

Handle a command:

```go
result, err := svc.Handle(ctx, BookRoom{
    BookingID: "booking-123",
    GuestID:   "guest-456",
    RoomID:    "room-789",
    Price:     100.0,
    Currency:  "USD",
})
```

### Aggregate Command Service (Alternative)

For DDD-style aggregate pattern with `Apply()` calls. Use when business logic benefits from aggregate guards and encapsulation.

```go
aggSvc := command.NewAggregateService(reader, writer, types, BookingFold, BookingState{})

command.OnAggregate(aggSvc, command.AggregateHandler[BookRoom, BookingState]{
    Expected: eventuous.IsNew,
    ID:       func(cmd BookRoom) string { return cmd.BookingID },
    Act: func(ctx context.Context, agg *aggregate.Aggregate[BookingState], cmd BookRoom) error {
        agg.Apply(RoomBooked{
            BookingID: cmd.BookingID,
            RoomID:    cmd.RoomID,
            Price:     cmd.Price,
            Currency:  cmd.Currency,
        })
        return nil
    },
})
```

Key differences from functional service:
- `ID` returns a string ID (stream name built automatically as `{StateType}-{ID}`)
- `Act` receives `*aggregate.Aggregate[S]` and calls `agg.Apply()` to record events
- `Act` returns `error` (not events)
- Guards (`EnsureNew`, `EnsureExists`) are enforced automatically based on `Expected`

### Result Type

```go
type Change struct {
    Event     any
    EventType string
}

type Result[S any] struct {
    State          S
    Changes        []Change
    GlobalPosition uint64
    StreamVersion  int64
}
```

---

## Subscriptions

Subscriptions deliver events to handlers for projections, integrations, and reactions.

### EventHandler Interface

```go
type EventHandler interface {
    HandleEvent(ctx context.Context, msg *ConsumeContext) error
}

// Convenience adapter
type HandlerFunc func(ctx context.Context, msg *ConsumeContext) error

func (f HandlerFunc) HandleEvent(ctx context.Context, msg *ConsumeContext) error
```

### ConsumeContext

```go
type ConsumeContext struct {
    EventID        uuid.UUID
    EventType      string
    Stream         eventuous.StreamName
    Payload        any                    // Deserialized event, nil if unknown type
    Metadata       eventuous.Metadata
    ContentType    string
    Position       uint64                 // Within source stream
    GlobalPosition uint64                 // In global log ($all)
    Sequence       uint64                 // Local sequence in subscription
    Created        time.Time
    SubscriptionID string
}
```

### Subscription Interface

```go
type Subscription interface {
    Start(ctx context.Context) error  // Blocks until context cancelled or fatal error
}
```

### Middleware

Composable middleware wraps handlers:

```go
type Middleware func(EventHandler) EventHandler

// Chain applies middleware in order (first = outermost)
func Chain(handler EventHandler, mw ...Middleware) EventHandler
```

Built-in middleware:

```go
// Log event processing at debug level
subscription.WithLogging(logger *slog.Logger) Middleware

// Process events concurrently up to limit
subscription.WithConcurrency(limit int) Middleware

// Distribute events across N goroutines by partition key
// keyFunc defaults to stream name if nil — preserves ordering per key
subscription.WithPartitioning(count int, keyFunc func(*ConsumeContext) string) Middleware
```

Example with middleware:

```go
handler := subscription.HandlerFunc(func(ctx context.Context, msg *subscription.ConsumeContext) error {
    switch e := msg.Payload.(type) {
    case RoomBooked:
        // project into read model
    case PaymentRecorded:
        // update payment status
    }
    return nil
})

handler = subscription.Chain(handler,
    subscription.WithLogging(slog.Default()),
    subscription.WithPartitioning(4, nil),
)
```

### Checkpoint Store

For catch-up subscriptions that need to resume from where they left off:

```go
type Checkpoint struct {
    ID       string
    Position *uint64  // nil = no checkpoint stored yet
}

type CheckpointStore interface {
    GetCheckpoint(ctx context.Context, id string) (Checkpoint, error)
    StoreCheckpoint(ctx context.Context, checkpoint Checkpoint) error
}
```

### CheckpointCommitter

Batches checkpoint writes with gap detection to handle concurrent/out-of-order processing:

```go
committer := subscription.NewCheckpointCommitter(
    checkpointStore,
    "my-subscription",
    100,                    // batch size (0 = no limit)
    5 * time.Second,        // interval (0 = no time limit)
)

committer.Commit(ctx, position, sequence)  // Record event processed
committer.Flush(ctx)                        // Force immediate commit
committer.Close(ctx)                        // Stop timer and flush
```

---

## Infrastructure-Specific Guides

When working with specific infrastructure, also include the relevant Go skill:

- `eventuous-go-kurrentdb` — KurrentDB event store, catch-up and persistent subscriptions
- `eventuous-go-otel` — OpenTelemetry tracing and metrics for commands and subscriptions
