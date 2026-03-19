# Setting Up a Functional Command Service in Eventuous

A **functional command service** in Eventuous is a stateless, pure-function approach to handling commands. Instead of loading an aggregate object, it works directly with a `State<T>` record and plain functions that take `(state, command)` and return events. There are no aggregate instances — just state + command → events.

## Prerequisites

Install the NuGet package:

```bash
dotnet add package Eventuous
```

For dependency injection and HTTP gateway support, also add:

```bash
dotnet add package Eventuous.AspNetCore
```

---

## Step 1: Define Your Events

Events are the source of truth. Define them as records.

```csharp
namespace Bookings.Domain;

// Raised when a room is successfully booked
public record RoomBooked(
    string BookingId,
    string RoomId,
    DateOnly CheckIn,
    DateOnly CheckOut,
    decimal Price
);

// Raised when a booking is cancelled
public record BookingCancelled(
    string BookingId,
    string Reason
);
```

---

## Step 2: Define Your State

`State<T>` is an immutable record that is rebuilt by folding events. Implement `When` to apply each event type.

```csharp
using Eventuous;

namespace Bookings.Domain;

public enum BookingStatus { New, Active, Cancelled }

public record BookingState : State<BookingState>
{
    public string        BookingId { get; init; } = "";
    public string        RoomId    { get; init; } = "";
    public DateOnly      CheckIn   { get; init; }
    public DateOnly      CheckOut  { get; init; }
    public decimal       Price     { get; init; }
    public BookingStatus Status    { get; init; } = BookingStatus.New;

    // The fold function: apply an event and return the new state
    public override BookingState When(object @event)
        => @event switch
        {
            RoomBooked e => this with
            {
                BookingId = e.BookingId,
                RoomId    = e.RoomId,
                CheckIn   = e.CheckIn,
                CheckOut  = e.CheckOut,
                Price     = e.Price,
                Status    = BookingStatus.Active
            },
            BookingCancelled => this with { Status = BookingStatus.Cancelled },
            _                => this   // unknown events are ignored
        };
}
```

---

## Step 3: Define Your Commands

Commands are plain records. No base class is required.

```csharp
namespace Bookings.Domain;

public record BookRoom(
    string  BookingId,
    string  RoomId,
    DateOnly CheckIn,
    DateOnly CheckOut,
    decimal  Price
);

public record CancelBooking(
    string BookingId,
    string Reason
);
```

---

## Step 4: Register Event Types

Events must be registered in `TypeMap` so Eventuous can serialize/deserialize them by name.

```csharp
using Eventuous;

TypeMap.RegisterKnownEventTypes(typeof(RoomBooked).Assembly);
// or register individually:
// TypeMap.AddType<RoomBooked>("V1.RoomBooked");
// TypeMap.AddType<BookingCancelled>("V1.BookingCancelled");
```

---

## Step 5: Create the Functional Command Service

Extend `CommandService<TState>` and register handlers in the constructor using the fluent `On<TCommand>` API.

```csharp
using Eventuous;

namespace Bookings.Application;

public class BookingService : CommandService<BookingState>
{
    public BookingService(IEventStore store) : base(store)
    {
        // Handle BookRoom
        // - "New" means the stream must NOT exist yet
        On<BookRoom>()
            .InState(ExpectedState.New)
            .GetStream(cmd => new StreamName($"Booking-{cmd.BookingId}"))
            .Act((state, cmd) =>
            [
                new RoomBooked(
                    cmd.BookingId,
                    cmd.RoomId,
                    cmd.CheckIn,
                    cmd.CheckOut,
                    cmd.Price
                )
            ]);

        // Handle CancelBooking
        // - "Existing" means the stream MUST already exist
        On<CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetStream(cmd => new StreamName($"Booking-{cmd.BookingId}"))
            .Act((state, cmd) =>
            {
                if (state.Status == BookingStatus.Cancelled)
                    throw new DomainException("Booking is already cancelled.");

                return [new BookingCancelled(cmd.BookingId, cmd.Reason)];
            });
    }
}
```

### Key concepts in the fluent API

| Method | Purpose |
|---|---|
| `On<TCommand>()` | Starts the handler registration for a command type |
| `.InState(ExpectedState.New)` | Stream must not exist — used for creation commands |
| `.InState(ExpectedState.Existing)` | Stream must exist — used for mutation commands |
| `.InState(ExpectedState.Any)` | Stream may or may not exist — for idempotent/upsert flows |
| `.GetStream(cmd => ...)` | Derives the event stream name from the command |
| `.Act((state, cmd) => [...])` | Pure function: takes current state + command, returns new events |

---

## Step 6: Register with ASP.NET Core DI

```csharp
using Eventuous;
using Eventuous.AspNetCore;
using Eventuous.EventStore; // or Eventuous.PostgreSQL, etc.

var builder = WebApplication.CreateBuilder(args);

// Register event type map (do this before anything else)
TypeMap.RegisterKnownEventTypes(typeof(RoomBooked).Assembly);

// Register your event store (example: in-memory for tests)
builder.Services.AddSingleton<IEventStore, InMemoryEventStore>();

// Register the command service
builder.Services.AddCommandService<BookingService, BookingState>();

var app = builder.Build();

// Map command HTTP endpoints automatically (requires Eventuous.AspNetCore)
app.MapAggregateCommands<BookingService>();

app.Run();
```

---

## Step 7: Using the Service Directly

If you don't need HTTP endpoints, inject and call the service directly:

```csharp
public class BookingController
{
    private readonly BookingService _service;

    public BookingController(BookingService service)
        => _service = service;

    public async Task BookAsync(CancellationToken ct)
    {
        var result = await _service.Handle(
            new BookRoom(
                BookingId: "booking-123",
                RoomId:    "room-42",
                CheckIn:   new DateOnly(2026, 6, 1),
                CheckOut:  new DateOnly(2026, 6, 7),
                Price:     850m
            ),
            ct
        );

        if (!result.Success)
            Console.WriteLine($"Failed: {result.Exception?.Message}");
    }

    public async Task CancelAsync(CancellationToken ct)
    {
        var result = await _service.Handle(
            new CancelBooking("booking-123", "Guest request"),
            ct
        );

        if (!result.Success)
            Console.WriteLine($"Failed: {result.Exception?.Message}");
    }
}
```

---

## Complete File Structure

```
Bookings/
├── Domain/
│   ├── Events.cs          # RoomBooked, BookingCancelled
│   ├── Commands.cs        # BookRoom, CancelBooking
│   └── BookingState.cs    # State<BookingState> with When()
└── Application/
    └── BookingService.cs  # CommandService<BookingState>
```

---

## How It Works Internally

When `Handle(command, ct)` is called, Eventuous:

1. Derives the stream name via your `.GetStream(...)` lambda.
2. Reads all existing events from that stream and folds them through `BookingState.When()` to reconstruct current state.
3. Calls your `.Act(state, command)` lambda to produce new events.
4. Appends the new events to the stream (using optimistic concurrency via the stream version).
5. Returns a `CommandHandlingResult` indicating success or failure.

This is entirely side-effect-free in the domain logic — the store interaction is handled by the infrastructure, not your code.

---

## Differences from Aggregate-Based Services

| Aspect | Aggregate-based | Functional |
|---|---|---|
| Base class | `CommandService<TAggregate, TState, TId>` | `CommandService<TState>` |
| Domain object | `Aggregate<TState>` with methods | Plain `State<TState>` record |
| Handler body | Calls aggregate methods (`aggregate.BookRoom(...)`) | Returns events directly |
| Mutation style | Imperative (OOP) | Functional (pure functions) |
| Testing | Requires aggregate instance | Easier to unit-test pure functions |

Use the functional style when you prefer pure functions and immutable data over OOP aggregates, or when your domain logic naturally maps to "given this state and command, produce these events."
