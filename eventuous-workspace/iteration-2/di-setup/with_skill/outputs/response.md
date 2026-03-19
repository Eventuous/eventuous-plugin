# Minimal Eventuous Setup: ASP.NET Core + KurrentDB

This guide shows the minimal `Program.cs` and `appsettings.json` for an ASP.NET Core app using Eventuous with KurrentDB as the event store and subscriptions.

---

## NuGet Packages

```xml
<PackageReference Include="Eventuous.KurrentDB" />
<PackageReference Include="Eventuous.Extensions.DependencyInjection" />
<PackageReference Include="Eventuous.AspNetCore" />
```

For subscriptions with a checkpoint store (required for catch-up subscriptions), add one of:

```xml
<!-- Option A: MongoDB checkpoint store -->
<PackageReference Include="Eventuous.Projections.MongoDB" />

<!-- Option B: PostgreSQL checkpoint store -->
<PackageReference Include="Eventuous.Postgresql" />
```

---

## Domain Events

Events are immutable records decorated with `[EventType]`. The source generator automatically registers them in `TypeMap` — **no manual `TypeMap.RegisterKnownEventTypes()` call is needed**.

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
            DateTime CheckInDate,
            DateTime CheckOutDate,
            float    Price,
            string   Currency
        );

        [EventType("V1.BookingCancelled")]
        public record BookingCancelled(string Reason);

        [EventType("V1.PaymentRecorded")]
        public record PaymentRecorded(string PaymentId, float Amount, string Currency);
    }
}
```

---

## Aggregate

```csharp
// Domain/Booking.cs
using Eventuous;
using static BookingEvents.V1;

public record BookingId(string Value) : Id(Value);

public record BookingState : State<BookingState> {
    public string   GuestId    { get; init; } = null!;
    public string   RoomId     { get; init; } = null!;
    public float    Price      { get; init; }
    public string   Currency   { get; init; } = null!;
    public bool     Cancelled  { get; init; }
    public float    AmountPaid { get; init; }

    public BookingState() {
        On<RoomBooked>((state, e) => state with {
            GuestId  = e.GuestId,
            RoomId   = e.RoomId,
            Price    = e.Price,
            Currency = e.Currency
        });
        On<BookingCancelled>((state, _) => state with { Cancelled = true });
        On<PaymentRecorded>((state, e) => state with { AmountPaid = state.AmountPaid + e.Amount });
    }
}

public class Booking : Aggregate<BookingState> {
    public void Book(string guestId, string roomId, DateTime checkIn, DateTime checkOut, float price, string currency) {
        EnsureDoesntExist();
        Apply(new RoomBooked(State.Id?.Value ?? "", guestId, roomId, checkIn, checkOut, price, currency));
    }

    public void Cancel(string reason) {
        EnsureExists();
        Apply(new BookingCancelled(reason));
    }

    public void RecordPayment(string paymentId, float amount, string currency) {
        EnsureExists();
        Apply(new PaymentRecorded(paymentId, amount, currency));
    }
}
```

---

## Commands

```csharp
// Application/BookingCommands.cs
public static class BookingCommands {
    public record BookRoom(
        string   BookingId,
        string   GuestId,
        string   RoomId,
        DateTime CheckInDate,
        DateTime CheckOutDate,
        float    Price,
        string   Currency
    );

    public record CancelBooking(string BookingId, string Reason);

    public record RecordPayment(string BookingId, string PaymentId, float Amount, string Currency);
}
```

---

## Command Service

```csharp
// Application/BookingsCommandService.cs
using Eventuous;
using static BookingCommands;

public class BookingsCommandService : CommandService<Booking, BookingState, BookingId> {
    public BookingsCommandService(IEventStore store) : base(store) {
        On<BookRoom>()
            .InState(ExpectedState.New)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Book(
                cmd.GuestId,
                cmd.RoomId,
                cmd.CheckInDate,
                cmd.CheckOutDate,
                cmd.Price,
                cmd.Currency
            ));

        On<CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Cancel(cmd.Reason));

        On<RecordPayment>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.RecordPayment(cmd.PaymentId, cmd.Amount, cmd.Currency));
    }
}
```

---

## Event Handler (for Subscriptions)

```csharp
// Application/BookingEventHandler.cs
using Eventuous.Subscriptions;
using EventHandler = Eventuous.Subscriptions.EventHandler;
using static BookingEvents.V1;

public class BookingEventHandler : EventHandler {
    readonly ILogger<BookingEventHandler> _logger;

    public BookingEventHandler(ILogger<BookingEventHandler> logger) {
        _logger = logger;

        On<RoomBooked>(ctx => {
            _logger.LogInformation(
                "Room {RoomId} booked for guest {GuestId} from {CheckIn} to {CheckOut}",
                ctx.Message.RoomId,
                ctx.Message.GuestId,
                ctx.Message.CheckInDate,
                ctx.Message.CheckOutDate
            );
            return ValueTask.CompletedTask;
        });

        On<PaymentRecorded>(ctx => {
            _logger.LogInformation(
                "Payment {PaymentId} of {Amount} {Currency} recorded",
                ctx.Message.PaymentId,
                ctx.Message.Amount,
                ctx.Message.Currency
            );
            return ValueTask.CompletedTask;
        });
    }
}
```

---

## Program.cs

```csharp
using Eventuous;
using Eventuous.Diagnostics.OpenTelemetry;
using Eventuous.KurrentDB;
using Eventuous.KurrentDB.Subscriptions;

var builder = WebApplication.CreateBuilder(args);

// ── Controllers / API ─────────────────────────────────────────────────────────
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ── KurrentDB client ──────────────────────────────────────────────────────────
// Registers KurrentDBClient as a singleton; used by event store and subscriptions.
builder.Services.AddKurrentDBClient(
    builder.Configuration["EventStore:ConnectionString"]!
);

// ── Event store ───────────────────────────────────────────────────────────────
// Registers IEventStore, IEventReader, and IEventWriter.
// No manual TypeMap calls needed — [EventType] attributes are source-generated.
builder.Services.AddEventStore<KurrentDBEventStore>();

// ── Command service ───────────────────────────────────────────────────────────
// Registers BookingsCommandService as ICommandService<BookingState>.
builder.Services.AddCommandService<BookingsCommandService, BookingState>();

// ── Catch-up subscription: $all stream ────────────────────────────────────────
// Reads every event across all streams from the beginning (or last checkpoint).
// Requires an external checkpoint store — here we use an in-memory store for
// simplicity. Replace with MongoCheckpointStore, PostgresCheckpointStore, etc.
// for production use.
//
// For a real checkpoint store, add the relevant NuGet package and replace:
//   .UseCheckpointStore<NoOpCheckpointStore>()
// with:
//   .UseCheckpointStore<MongoCheckpointStore>()   // Eventuous.Projections.MongoDB
//   .UseCheckpointStore<PostgresCheckpointStore>() // Eventuous.Postgresql
builder.Services.AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(
    "BookingReadModels",
    b => b
        .UseCheckpointStore<NoOpCheckpointStore>()  // replace with real store in production
        .AddEventHandler<BookingEventHandler>()
        .WithPartitioningByStream(2)                // parallel processing by stream
);

// ── Persistent subscription (no checkpoint store needed) ─────────────────────
// KurrentDB manages the position server-side. Auto-created if it doesn't exist.
builder.Services.AddSubscription<AllPersistentSubscription, AllPersistentSubscriptionOptions>(
    "BookingIntegration",
    b => b
        .AddEventHandler<BookingEventHandler>()
);

// ── OpenTelemetry (optional but recommended) ──────────────────────────────────
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddEventuousTracing()
    )
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddEventuous()
        .AddEventuousSubscriptions()
    );

// ─────────────────────────────────────────────────────────────────────────────

var app = builder.Build();

app.UseSwagger().UseSwaggerUI();
app.MapControllers();

// Eventuous diagnostic endpoint — shows subscription health, checkpoint info, etc.
app.MapEventuousSpyglass();

app.Run();
```

---

## appsettings.json

```json
{
  "EventStore": {
    "ConnectionString": "esdb://localhost:2113?tls=false"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Grpc": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

For a TLS-secured KurrentDB instance (e.g., in staging/production):

```json
{
  "EventStore": {
    "ConnectionString": "esdb://admin:changeit@my-kurrentdb-host:2113"
  }
}
```

---

## Key Points

### Event Type Registration

Events decorated with `[EventType("...")]` are automatically discovered and registered by a compile-time source generator. You do **not** need:

```csharp
// NOT needed — handled automatically
TypeMap.RegisterKnownEventTypes();
DefaultEventSerializer.SetDefaultSerializer(...);
```

### Checkpoint Store Selection

`AllStreamSubscription` and `StreamSubscription` are **catch-up subscriptions** — they read from a position and need a checkpoint store to survive restarts:

| Package | Checkpoint Store Class |
|---|---|
| `Eventuous.Projections.MongoDB` | `MongoCheckpointStore` |
| `Eventuous.Postgresql` | `PostgresCheckpointStore` |
| `Eventuous.SqlServer` | `SqlServerCheckpointStore` |

**Persistent subscriptions** (`AllPersistentSubscription`, `StreamPersistentSubscription`) store their position in KurrentDB itself — no external checkpoint store is needed.

### Subscription Type Summary

| Type | Stream | Checkpoint | Use Case |
|---|---|---|---|
| `AllStreamSubscription` | `$all` (all streams) | External store required | Cross-aggregate projections |
| `StreamSubscription` | Single named stream | External store required | Stream-specific handlers |
| `AllPersistentSubscription` | `$all` (all streams) | KurrentDB (server-side) | Integration, load-balanced consumers |
| `StreamPersistentSubscription` | Single named stream | KurrentDB (server-side) | Integration, load-balanced consumers |

### Deprecated API

`IAggregateStore` is deprecated. Use `IEventReader.LoadAggregate<>()` and `IEventWriter.StoreAggregate<>()` extension methods, or use `IEventStore` directly via `CommandService`.

### Connection String Format

Eventuous KurrentDB uses the `esdb://` scheme (not `kurrentdb://`):

```
esdb://localhost:2113?tls=false          # local dev, no TLS
esdb://admin:changeit@host:2113          # with credentials
esdb://host1:2113,host2:2113?tls=true   # cluster
```
