# Functional Command Service Setup in Eventuous

This guide shows a complete functional command service setup for a `BookingState` with `BookRoom` and `CancelBooking` commands, including DI registration and an HTTP API.

---

## Overview

The **functional command service** (`CommandService<TState>`) is a pure-function style approach — no aggregate instances. Handlers receive state and return events rather than calling methods on an aggregate object.

---

## 1. Domain Events

Events are immutable records decorated with `[EventType]` for serialization. The source generator automatically registers them in `TypeMap` — no manual registration needed.

```csharp
// BookingEvents.cs
using Eventuous;

public static class BookingEvents {
    public static class V1 {
        [EventType("V1.RoomBooked")]
        public record RoomBooked(
            string   BookingId,
            string   GuestId,
            string   RoomId,
            DateTime CheckIn,
            DateTime CheckOut,
            float    Price,
            string   Currency
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

State is an immutable record rebuilt from events. Register event handlers in the parameterless constructor using `On<TEvent>()`. Handlers are pure functions returning new state via `with` expressions.

```csharp
// BookingState.cs
using Eventuous;
using static BookingEvents;

public record BookingState : State<BookingState> {
    public string   BookingId  { get; init; } = null!;
    public string   GuestId    { get; init; } = null!;
    public string   RoomId     { get; init; } = null!;
    public DateTime CheckIn    { get; init; }
    public DateTime CheckOut   { get; init; }
    public float    Price      { get; init; }
    public string   Currency   { get; init; } = null!;
    public bool     IsCancelled { get; init; }

    public BookingState() {
        On<V1.RoomBooked>((state, e) => state with {
            BookingId  = e.BookingId,
            GuestId    = e.GuestId,
            RoomId     = e.RoomId,
            CheckIn    = e.CheckIn,
            CheckOut   = e.CheckOut,
            Price      = e.Price,
            Currency   = e.Currency
        });

        On<V1.BookingCancelled>((state, e) => state with {
            IsCancelled = true
        });
    }
}
```

---

## 3. Commands

Commands are record types. Annotate them with `[HttpCommand]` (or `[HttpCommands]` on the container class) for minimal API auto-discovery.

```csharp
// BookingCommands.cs
using Eventuous.AspNetCore.Web;

[HttpCommands<BookingState>]
public static class BookingCommands {
    [HttpCommand(Route = "book")]
    public record BookRoom(
        string   BookingId,
        string   GuestId,
        string   RoomId,
        DateTime CheckIn,
        DateTime CheckOut,
        float    Price,
        string   Currency
    );

    [HttpCommand(Route = "cancel")]
    public record CancelBooking(
        string BookingId,
        string Reason
    );
}
```

---

## 4. Functional Command Service

Extend `CommandService<TState>`. Use `GetStream` to derive the stream name and `Act` to return new events. The `Act` overload with `(state, originalEvents, cmd)` receives the current state for existing streams.

```csharp
// BookingService.cs
using Eventuous;
using static BookingCommands;
using static BookingEvents;

public class BookingService : CommandService<BookingState> {
    public BookingService(IEventStore store) : base(store) {
        // BookRoom creates a new booking — ExpectedState.New
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

        // CancelBooking requires an existing booking — ExpectedState.Existing
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

**Key points:**
- `GetStream(cmd.BookingId)` uses the default naming convention: `Booking-{id}` (strips the `State` suffix from `BookingState`).
- `ExpectedState.New` — stream must not exist (creates it).
- `ExpectedState.Existing` — stream must exist (updates it).
- `ExpectedState.Any` — works for both new and existing.
- The three-argument `Act` overload `(state, originalEvents, cmd)` receives the rebuilt state and the original persisted events, useful for business rule checks before returning new events.

---

## 5. DI Registration (`Program.cs`)

```csharp
// Program.cs
using Eventuous.KurrentDB;
using Eventuous.AspNetCore.Web;

var builder = WebApplication.CreateBuilder(args);

// 1. Register KurrentDB client
builder.Services.AddKurrentDBClient(
    builder.Configuration["KurrentDB:ConnectionString"]!
);

// 2. Register event store
builder.Services.AddEventStore<KurrentDBEventStore>();

// 3. Register the functional command service
builder.Services.AddCommandService<BookingService, BookingState>();

// 4. (Optional) Add OpenTelemetry diagnostics
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b.AddEventuousTracing())
    .WithMetrics(b => b.AddEventuous());

// 5. Add controllers (only if using controller-based API)
builder.Services.AddControllers();

var app = builder.Build();

// Map auto-discovered HTTP commands (minimal API style)
app.MapDiscoveredCommands<BookingState>();

// Or map controllers (if using controller-based API):
// app.MapControllers();

app.Run();
```

---

## 6. Configuration (`appsettings.json`)

```json
{
  "KurrentDB": {
    "ConnectionString": "kurrentdb://localhost:2113?tls=false"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

---

## 7. HTTP API Options

### Option A: Minimal API (auto-discovered, recommended)

The `[HttpCommands<BookingState>]` / `[HttpCommand(Route = "...")]` annotations from step 3 combined with `app.MapDiscoveredCommands<BookingState>()` automatically wire up:

- `POST /booking/book` → `BookRoom`
- `POST /booking/cancel` → `CancelBooking`

No additional controller code needed.

### Option B: Controller-Based API

```csharp
// BookingCommandApi.cs
using Eventuous.AspNetCore.Web;
using Microsoft.AspNetCore.Mvc;
using static BookingCommands;

[Route("/booking")]
public class BookingCommandApi(ICommandService<BookingState> service)
    : CommandHttpApiBase<BookingState>(service) {

    [HttpPost("book")]
    public Task<ActionResult<Result<BookingState>.Ok>> Book(
        [FromBody] BookRoom cmd, CancellationToken ct) => Handle(cmd, ct);

    [HttpPost("cancel")]
    public Task<ActionResult<Result<BookingState>.Ok>> Cancel(
        [FromBody] CancelBooking cmd, CancellationToken ct) => Handle(cmd, ct);
}
```

Register controllers in DI and map them:

```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

---

## 8. Handling the Result

The command service returns `Result<BookingState>`. You can handle it manually if needed:

```csharp
var result = await bookingService.Handle(command, cancellationToken);

result.Match(
    ok => {
        Console.WriteLine($"State: {ok.State}");
        Console.WriteLine($"Changes: {ok.Changes.Count} event(s)");
        Console.WriteLine($"Global position: {ok.GlobalPosition}");
    },
    error => {
        Console.WriteLine($"Error: {error.ErrorMessage}");
        // error.Exception contains the original exception
    }
);
```

When using `CommandHttpApiBase` or `MapDiscoveredCommands`, the HTTP response is handled automatically — returning `200 OK` with the result body or an appropriate error status.

---

## Project File (NuGet Packages)

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>preview</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Eventuous.AspNetCore.Web" Version="*" />
    <PackageReference Include="Eventuous.KurrentDB" Version="*" />
  </ItemGroup>
</Project>
```

`Eventuous.AspNetCore.Web` transitively includes the core domain and command service packages. `Eventuous.KurrentDB` provides the event store implementation.

---

## Summary

| File | Purpose |
|---|---|
| `BookingEvents.cs` | Domain events with `[EventType]` for auto-registration |
| `BookingState.cs` | Immutable state record rebuilt from events |
| `BookingCommands.cs` | Command records with `[HttpCommand]` for minimal API |
| `BookingService.cs` | Functional command service extending `CommandService<BookingState>` |
| `Program.cs` | DI registration and HTTP mapping |

The functional style is ideal when you prefer pure functions over object-oriented aggregates, or when the domain logic is straightforward enough that a full aggregate class would be overhead.
