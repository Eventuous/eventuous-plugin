# Setting Up a Functional Command Service in Eventuous

This guide walks through setting up a functional command service for a `BookingState` with `BookRoom` and `CancelBooking` commands. The functional style uses pure functions that receive state and return events — no aggregate class required.

---

## 1. Domain Events

Events are immutable records decorated with `[EventType]`. Group them in a static class with versioned nested classes to support schema evolution. Use primitive types (not value objects) in event records.

```csharp
using Eventuous;

public static class BookingEvents {
    public static class V1 {
        [EventType("V1.RoomBooked")]
        public record RoomBooked(
            string BookingId,
            string GuestId,
            string RoomId,
            DateTime CheckIn,
            DateTime CheckOut,
            float Price,
            string Currency
        );

        [EventType("V1.BookingCancelled")]
        public record BookingCancelled(
            string BookingId,
            string Reason
        );
    }
}
```

---

## 2. State

`BookingState` extends `State<BookingState>` and registers event handlers in its parameterless constructor using `On<TEvent>()`. Handlers are pure functions that return new state via `with` expressions.

```csharp
using Eventuous;
using static BookingEvents;

public record BookingState : State<BookingState> {
    public string GuestId    { get; init; } = null!;
    public string RoomId     { get; init; } = null!;
    public float  Price      { get; init; }
    public string Currency   { get; init; } = null!;
    public bool   IsBooked   { get; init; }
    public bool   IsCancelled { get; init; }

    public BookingState() {
        On<V1.RoomBooked>((state, e) => state with {
            GuestId    = e.GuestId,
            RoomId     = e.RoomId,
            Price      = e.Price,
            Currency   = e.Currency,
            IsBooked   = true
        });

        On<V1.BookingCancelled>((state, _) => state with {
            IsCancelled = true
        });
    }
}
```

---

## 3. Commands

Commands are simple record types. Group them in a static class for organization.

```csharp
public static class BookingCommands {
    public record BookRoom(
        string   BookingId,
        string   GuestId,
        string   RoomId,
        DateTime CheckIn,
        DateTime CheckOut,
        float    Price,
        string   Currency
    );

    public record CancelBooking(
        string BookingId,
        string Reason
    );
}
```

---

## 4. Functional Command Service

The functional command service extends `CommandService<TState>` (not `CommandService<TAggregate, TState, TId>`). Key differences from the aggregate-based approach:

- Use `.GetStream(cmd => ...)` instead of `.GetId(cmd => ...)` — the helper `GetStream(id)` produces a stream name following the default `{StateNameWithoutSuffix}-{id}` convention (e.g., `Booking-abc123`).
- `.Act(cmd => events)` is used for `ExpectedState.New` — it receives only the command and returns the new events.
- `.Act((state, originalEvents, cmd) => events)` is used for `ExpectedState.Existing` — it receives the current state, the original events already in the stream, and the command.

```csharp
using Eventuous;
using static BookingCommands;
using static BookingEvents;

public class BookingService : CommandService<BookingState> {
    public BookingService(IEventStore store) : base(store) {
        On<BookRoom>()
            .InState(ExpectedState.New)
            .GetStream(cmd => GetStream(cmd.BookingId))
            .Act(cmd => [
                new V1.RoomBooked(
                    cmd.BookingId,
                    cmd.GuestId,
                    cmd.RoomId,
                    cmd.CheckIn,
                    cmd.CheckOut,
                    cmd.Price,
                    cmd.Currency
                )
            ]);

        On<CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetStream(cmd => GetStream(cmd.BookingId))
            .Act((state, _, cmd) => {
                if (state.IsCancelled)
                    throw new DomainException("Booking is already cancelled");

                return [new V1.BookingCancelled(cmd.BookingId, cmd.Reason)];
            });
    }
}
```

> **Note on `GetStream`:** The `GetStream(id)` helper method (inherited from `CommandService<TState>`) converts the string id into a `StreamName` using the state type name minus any "State" suffix. For `BookingState`, the stream will be named `Booking-{BookingId}`.

---

## 5. DI Registration

Register the service in `Program.cs`:

```csharp
using Eventuous;
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);

// 1. Configure JSON serialization (recommended)
DefaultEventSerializer.SetDefaultSerializer(
    new DefaultEventSerializer(new JsonSerializerOptions(JsonSerializerDefaults.Web))
);

// 2. Register event types from the assembly
TypeMap.RegisterKnownEventTypes(typeof(BookingEvents).Assembly);

// 3. Register the event store (example: KurrentDB / EventStoreDB)
builder.Services.AddEventStore<EsdbEventStore>();  // replace with your provider

// 4. Register the functional command service
builder.Services.AddCommandService<BookingService, BookingState>();

var app = builder.Build();
app.Run();
```

---

## 6. Handling Commands

Inject `ICommandService<BookingState>` wherever you need to dispatch commands:

```csharp
public class BookingEndpoints {
    public static void Map(WebApplication app) {
        app.MapPost("/bookings/book", async (
            BookRoom cmd,
            ICommandService<BookingState> service,
            CancellationToken ct) =>
        {
            var result = await service.Handle(cmd, ct);
            return result.Match(
                ok    => Results.Ok(ok.State),
                error => Results.Problem(error.ErrorMessage)
            );
        });

        app.MapPost("/bookings/cancel", async (
            CancelBooking cmd,
            ICommandService<BookingState> service,
            CancellationToken ct) =>
        {
            var result = await service.Handle(cmd, ct);
            return result.Match(
                ok    => Results.Ok(ok.State),
                error => Results.Problem(error.ErrorMessage)
            );
        });
    }
}
```

Alternatively, use the auto-discovery approach with `[HttpCommand]` on the command records and `app.MapDiscoveredCommands<BookingState>()`.

---

## 7. Complete File Layout

A typical project layout for this example:

```
src/
  Domain/
    BookingEvents.cs      # Domain events with [EventType]
    BookingState.cs       # State with On<> handlers
  Application/
    BookingCommands.cs    # Command records
    BookingService.cs     # Functional CommandService<BookingState>
  Program.cs              # DI registration and app startup
```

---

## Key Concepts Summary

| Concept | Detail |
|---|---|
| Base class | `CommandService<BookingState>` (functional, no aggregate) |
| New stream | `.InState(ExpectedState.New)` + `.Act(cmd => events)` |
| Existing stream | `.InState(ExpectedState.Existing)` + `.Act((state, originalEvents, cmd) => events)` |
| Stream naming | `GetStream(id)` → `Booking-{id}` (from `BookingState`) |
| DI registration | `services.AddCommandService<BookingService, BookingState>()` |
| Result type | `Result<BookingState>` — pattern match on `Ok` or `Error` |
