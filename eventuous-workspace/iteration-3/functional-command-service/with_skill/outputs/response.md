# Setting Up a Functional Command Service in Eventuous

This guide shows how to set up a functional command service for a `BookingState` that handles `BookRoom` and `CancelBooking` commands, with KurrentDB as the event store and an HTTP API.

## Project Structure

```
Bookings/
├── Bookings.csproj
├── Program.cs
├── appsettings.json
├── Domain/
│   ├── BookingEvents.cs
│   ├── BookingState.cs
│   └── BookingCommands.cs
└── Application/
    └── BookingsCommandService.cs
```

## NuGet Packages

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>preview</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Eventuous.KurrentDB" Version="0.*" />
    <PackageReference Include="Eventuous.AspNetCore" Version="0.*" />
    <PackageReference Include="Eventuous.Extensions.DependencyInjection" Version="0.*" />
  </ItemGroup>
</Project>
```

## Domain Events

Events are immutable records decorated with `[EventType]`. Group them in a static class for organization and schema evolution support. Use primitive types in events, not value objects.

```csharp
// Domain/BookingEvents.cs
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

The `[EventType]` attribute registers these events automatically via source generation — no manual `TypeMap` calls are needed.

## Booking State

`State<T>` is an abstract record reconstructed from events. Register event handlers in the parameterless constructor using `On<TEvent>()`. Handlers are pure functions that return new state via `with` expressions.

```csharp
// Domain/BookingState.cs
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
    public bool     Cancelled  { get; init; }

    public BookingState() {
        On<V1.RoomBooked>(HandleBooked);
        On<V1.BookingCancelled>(HandleCancelled);
    }

    static BookingState HandleBooked(BookingState state, V1.RoomBooked e)
        => state with {
            BookingId = e.BookingId,
            GuestId   = e.GuestId,
            RoomId    = e.RoomId,
            CheckIn   = e.CheckIn,
            CheckOut  = e.CheckOut,
            Price     = e.Price,
            Currency  = e.Currency
        };

    static BookingState HandleCancelled(BookingState state, V1.BookingCancelled e)
        => state with { Cancelled = true };
}
```

## Commands

Commands are simple record types. Annotate with `[HttpCommand]` to enable minimal API auto-discovery.

```csharp
// Domain/BookingCommands.cs
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

## Functional Command Service

The functional command service extends `CommandService<TState>`. It uses `GetStream` (not `GetId`) and `Act` returns events rather than calling aggregate methods.

Key differences from aggregate-based services:
- No aggregate class needed — pure functions operate directly on state
- `Act` for new streams takes only the command and returns events
- `Act` for existing streams receives `(state, originalEvents, command)` and returns events
- `GetStream(id)` helper uses the pattern `{StateNameWithoutSuffix}-{id}` (e.g., `Booking-booking-123`)

```csharp
// Application/BookingsCommandService.cs
using Eventuous;
using static BookingEvents;
using static BookingCommands;

public class BookingsCommandService : CommandService<BookingState> {
    public BookingsCommandService(IEventStore store) : base(store) {
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
            .Act((state, originalEvents, cmd) => {
                if (state.Cancelled)
                    throw new DomainException("Booking is already cancelled");

                return [new V1.BookingCancelled(cmd.BookingId, cmd.Reason)];
            });
    }
}
```

### Fluent Builder Chain (Functional)

1. `On<TCommand>()` — register handler for this command type
2. `.InState(ExpectedState)` — `New` (stream must not exist), `Existing` (stream must exist), or `Any`
3. `.GetStream(cmd => ...)` — derive the stream name from the command; use the `GetStream(id)` helper for default naming
4. `.Act(cmd => events)` — for new streams; receives only the command, returns `IEnumerable<object>`
5. `.Act((state, originalEvents, cmd) => events)` — for existing streams; receives current state, all original events, and the command

## DI Registration and HTTP API (Program.cs)

```csharp
// Program.cs
using Eventuous;
using Eventuous.KurrentDB;
using Eventuous.AspNetCore.Web;

var builder = WebApplication.CreateBuilder(args);

// 1. Register KurrentDB client (reads connection string from configuration)
builder.Services.AddKurrentDBClient(
    builder.Configuration["KurrentDB:ConnectionString"]!
);

// 2. Register the event store (registers IEventStore, IEventReader, IEventWriter)
builder.Services.AddEventStore<KurrentDBEventStore>();

// 3. Register the functional command service
builder.Services.AddCommandService<BookingsCommandService, BookingState>();

// 4. Add controllers or minimal API support
builder.Services.AddControllers();

var app = builder.Build();

// 5. Map all commands discovered via [HttpCommands<BookingState>] attribute
//    BookRoom  -> POST /booking/book
//    CancelBooking -> POST /booking/cancel
app.MapDiscoveredCommands<BookingState>(assemblies: typeof(BookingCommands).Assembly);

app.Run();
```

## Configuration

```json
// appsettings.json
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

## HTTP API

With `[HttpCommands<BookingState>]` and `[HttpCommand(Route = ...)]` on the command records, `MapDiscoveredCommands` automatically registers these endpoints:

| Method | Route             | Command         | Expected State |
|--------|-------------------|-----------------|----------------|
| POST   | `/booking/book`   | `BookRoom`      | New            |
| POST   | `/booking/cancel` | `CancelBooking` | Existing       |

### Example Requests

**Book a room:**

```http
POST /booking/book
Content-Type: application/json

{
  "bookingId": "booking-123",
  "guestId": "guest-456",
  "roomId": "room-101",
  "checkIn": "2026-04-01T00:00:00Z",
  "checkOut": "2026-04-05T00:00:00Z",
  "price": 250.00,
  "currency": "USD"
}
```

**Cancel a booking:**

```http
POST /booking/cancel
Content-Type: application/json

{
  "bookingId": "booking-123",
  "reason": "Guest requested cancellation"
}
```

### Response

Both endpoints return `Result<BookingState>.Ok` on success:

```json
{
  "state": {
    "bookingId": "booking-123",
    "guestId": "guest-456",
    "roomId": "room-101",
    "checkIn": "2026-04-01T00:00:00Z",
    "checkOut": "2026-04-05T00:00:00Z",
    "price": 250.00,
    "currency": "USD",
    "cancelled": false
  },
  "changes": [...],
  "globalPosition": 42
}
```

On error (e.g., booking already cancelled), a `400 Bad Request` or `404 Not Found` is returned with the error message.

## Alternative: Controller-Based HTTP API

If you prefer a controller over minimal APIs:

```csharp
using Eventuous;
using Eventuous.AspNetCore.Web;
using Microsoft.AspNetCore.Mvc;
using static BookingCommands;

[Route("/booking")]
public class BookingCommandApi(ICommandService<BookingState> service)
    : CommandHttpApiBase<BookingState>(service) {

    [HttpPost("book")]
    public Task<ActionResult<Result<BookingState>.Ok>> BookRoom(
        [FromBody] BookRoom cmd, CancellationToken ct) => Handle(cmd, ct);

    [HttpPost("cancel")]
    public Task<ActionResult<Result<BookingState>.Ok>> CancelBooking(
        [FromBody] CancelBooking cmd, CancellationToken ct) => Handle(cmd, ct);
}
```

Add `builder.Services.AddControllers()` and `app.MapControllers()` in `Program.cs` for this approach.

## Stream Naming

The functional service's `GetStream(id)` helper uses the pattern `{StateNameWithoutSuffix}-{id}`. With `BookingState`, the "State" suffix is stripped, so:

- `GetStream("booking-123")` produces the stream name `Booking-booking-123`

To customize this, register a `StreamNameMap`:

```csharp
var map = new StreamNameMap();
map.Register<string>(id => new StreamName($"bookings:{id}"));
// Pass to command service constructor: base(store, streamNameMap: map)
```

## Key Design Points

- **No aggregate class needed**: The functional style keeps business logic in pure functions within the command service itself, not in a separate aggregate class.
- **State is rebuilt from events**: On each command, Eventuous reads the stream from KurrentDB, replays all events through `BookingState`, and passes the resulting state to your `Act` handler.
- **Optimistic concurrency**: `ExpectedState.New` fails if the stream already exists; `ExpectedState.Existing` fails if it does not. This prevents double-booking and handles concurrent modifications.
- **Event type registration**: The `[EventType]` attribute on event records triggers source generation at compile time — no `TypeMap.RegisterKnownEventTypes()` call is needed anywhere.
- **`IAggregateStore` is deprecated**: Use `IEventReader`/`IEventWriter` extension methods (or `IEventStore`) instead.
