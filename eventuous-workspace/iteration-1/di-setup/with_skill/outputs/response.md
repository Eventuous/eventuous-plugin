# Minimal Eventuous Setup: Event Type Registration & DI for ASP.NET Core

This guide shows the minimal setup for an ASP.NET Core app using Eventuous with KurrentDB (the recommended default event store) and a catch-up subscription.

## NuGet Packages

```xml
<PackageReference Include="Eventuous.KurrentDB" />
<PackageReference Include="Eventuous.Extensions.DependencyInjection" />
<!-- For checkpoint store (pick one): -->
<PackageReference Include="Eventuous.Projections.MongoDB" />
```

## 1. Define Your Domain Events

Decorate events with `[EventType]` — this attribute drives automatic type map registration.

```csharp
// Events/BookingEvents.cs
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
        public record BookingCancelled(string BookingId, string Reason);
    }
}
```

## 2. Register Event Types

Call this once at startup, before any serialization or subscription work happens. The easiest approach scans all loaded assemblies for `[EventType]`-decorated types:

```csharp
// Scans all loaded assemblies
TypeMap.RegisterKnownEventTypes();

// Or scope to a specific assembly (more predictable, recommended)
TypeMap.RegisterKnownEventTypes(typeof(BookingEvents).Assembly);
```

## 3. Complete Program.cs

```csharp
// Program.cs
using Eventuous;
using Eventuous.KurrentDB;
using Eventuous.KurrentDB.Subscriptions;
using Eventuous.Projections.MongoDB;

var builder = WebApplication.CreateBuilder(args);

// --- Event type registration (must happen before anything serializes events) ---
TypeMap.RegisterKnownEventTypes(typeof(BookingEvents).Assembly);

// --- Optional: configure the default serializer ---
DefaultEventSerializer.SetDefaultSerializer(
    new DefaultEventSerializer(new JsonSerializerOptions(JsonSerializerDefaults.Web))
);

// --- Event store ---
// Register the KurrentDB gRPC client
builder.Services.AddKurrentDBClient(
    builder.Configuration["EventStore:ConnectionString"]!
);
// Register IEventStore / IEventReader / IEventWriter
builder.Services.AddEventStore<KurrentDBEventStore>();

// --- Command service ---
builder.Services.AddCommandService<BookingsCommandService, BookingState>();

// --- Subscription with a catch-up subscription to $all ---
// AllStreamSubscription requires a checkpoint store; using MongoDB here.
builder.Services.AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(
    "BookingsProjections",
    b => b
        .UseCheckpointStore<MongoCheckpointStore>()
        .AddEventHandler<BookingReadModelHandler>()
);

// --- ASP.NET Core pipeline ---
builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```

## 4. Example Command Service

```csharp
// Application/BookingsCommandService.cs
using Eventuous;

public class BookingsCommandService : CommandService<Booking, BookingState, BookingId> {
    public BookingsCommandService(IEventStore store) : base(store) {
        On<BookingCommands.BookRoom>()
            .InState(ExpectedState.New)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.BookRoom(
                cmd.GuestId,
                new RoomId(cmd.RoomId),
                new StayPeriod(cmd.CheckIn, cmd.CheckOut),
                new Money(cmd.Price, cmd.Currency)
            ));

        On<BookingCommands.CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Cancel(cmd.Reason));
    }
}
```

## 5. Example Event Handler (Subscription Consumer)

```csharp
// Application/BookingReadModelHandler.cs
using Eventuous.Subscriptions.Context;
using static BookingEvents.V1;

public class BookingReadModelHandler : EventHandler {
    // Inject any read-model store here, e.g. IMongoDatabase
    public BookingReadModelHandler() {
        On<RoomBooked>(async ctx => {
            var evt = ctx.Message;
            // Write to your read model store
            // e.g. await _collection.InsertOneAsync(new BookingDocument { ... });
        });

        On<BookingCancelled>(async ctx => {
            // Update read model
        });
    }
}
```

## 6. appsettings.json

```json
{
  "EventStore": {
    "ConnectionString": "esdb://localhost:2113?tls=false"
  }
}
```

## Key Points

| Concern | How |
|---|---|
| **Event type registration** | `TypeMap.RegisterKnownEventTypes(assembly)` at startup, driven by `[EventType]` attributes |
| **Event store** | `AddKurrentDBClient(connStr)` then `AddEventStore<KurrentDBEventStore>()` |
| **Command service** | `AddCommandService<TService, TState>()` — registers as `ICommandService<TState>` |
| **Catch-up subscription** | `AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(...)` — needs a checkpoint store |
| **Persistent subscription** | `AddSubscription<StreamPersistentSubscription, StreamPersistentSubscriptionOptions>(...)` — no checkpoint store needed, KurrentDB manages position |
| **Serializer** | Defaults to `System.Text.Json`; customize with `DefaultEventSerializer.SetDefaultSerializer(...)` |

### When to use `AllStreamSubscription` vs `StreamPersistentSubscription`

- **`AllStreamSubscription`**: catch-up style, reads from `$all`, restartable, requires an external checkpoint store (MongoDB, PostgreSQL, etc.). Use for projections.
- **`StreamPersistentSubscription`**: server-managed checkpoint, competes for messages with other consumers in the same group. Use for integration handlers where at-least-once delivery and competing consumers are needed.
