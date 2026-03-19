# DI Setup for ASP.NET Core with KurrentDB, Command Service, Subscriptions, and HTTP API

This guide shows a complete, runnable `Program.cs` and `appsettings.json` for an Eventuous application using KurrentDB as the event store. The example is a Bookings domain with an aggregate-based command service, an `AllStreamSubscription` catch-up subscription for projections, a persistent subscription for payment integration, and HTTP command mapping via Minimal API auto-discovery.

---

## NuGet Packages

```xml
<PackageReference Include="Eventuous.KurrentDB" />
<PackageReference Include="Eventuous.Extensions.DependencyInjection" />
<PackageReference Include="Eventuous.AspNetCore" />
<PackageReference Include="Eventuous.Diagnostics.OpenTelemetry" />
<PackageReference Include="Eventuous.Projections.MongoDB" />
```

> `AllStreamSubscription` (catch-up) requires an external checkpoint store. This example uses MongoDB (`MongoCheckpointStore`). You can substitute `PostgresCheckpointStore` or `SqlServerCheckpointStore` if preferred. Persistent subscriptions (`StreamPersistentSubscription`, `AllPersistentSubscription`) do not need a checkpoint store — KurrentDB manages the position server-side.

---

## Domain Model (supporting types)

These types are referenced by the `Program.cs` below. They follow Eventuous conventions exactly.

### BookingId.cs

```csharp
namespace Bookings.Domain.Bookings;

public record BookingId(string Value) : Id(Value);
```

### BookingEvents.cs

```csharp
using Eventuous;

namespace Bookings.Domain.Bookings;

public static class BookingEvents {
    public static class V1 {
        [EventType("V1.RoomBooked")]
        public record RoomBooked(
            string GuestId,
            string RoomId,
            DateTime CheckInDate,
            DateTime CheckOutDate,
            float    BookingPrice,
            float    PrepaidAmount,
            string   Currency
        );

        [EventType("V1.PaymentRecorded")]
        public record PaymentRecorded(
            string BookingId,
            string PaymentId,
            float  PaidAmount,
            string Currency
        );

        [EventType("V1.BookingCancelled")]
        public record BookingCancelled(string Reason);
    }
}
```

> Events decorated with `[EventType]` are automatically registered in `TypeMap` via source generation. No manual `TypeMap.RegisterKnownEventTypes()` call is needed.

### BookingState.cs

```csharp
using Eventuous;
using static Bookings.Domain.Bookings.BookingEvents;

namespace Bookings.Domain.Bookings;

public record BookingState : State<BookingState> {
    public string GuestId    { get; init; } = null!;
    public string RoomId     { get; init; } = null!;
    public float  Price      { get; init; }
    public string Currency   { get; init; } = null!;
    public bool   Cancelled  { get; init; }

    public BookingState() {
        On<V1.RoomBooked>((state, e) => state with {
            GuestId   = e.GuestId,
            RoomId    = e.RoomId,
            Price     = e.BookingPrice,
            Currency  = e.Currency
        });

        On<V1.BookingCancelled>((state, _) => state with { Cancelled = true });
    }
}
```

### Booking.cs (Aggregate)

```csharp
using Eventuous;
using static Bookings.Domain.Bookings.BookingEvents;

namespace Bookings.Domain.Bookings;

public class Booking : Aggregate<BookingState> {
    public void BookRoom(string guestId, string roomId, DateTime checkIn, DateTime checkOut, float price, string currency) {
        EnsureDoesntExist();
        Apply(new V1.RoomBooked(guestId, roomId, checkIn, checkOut, price, 0, currency));
    }

    public void RecordPayment(string paymentId, float paidAmount, string currency) {
        EnsureExists();
        Apply(new V1.PaymentRecorded(State.Id?.ToString() ?? "", paymentId, paidAmount, currency));
    }

    public void Cancel(string reason) {
        EnsureExists();
        Apply(new V1.BookingCancelled(reason));
    }
}
```

### Commands.cs

```csharp
using Eventuous.AspNetCore.Web;
using Bookings.Domain.Bookings;

namespace Bookings.Application;

[HttpCommands<BookingState>]
public static class BookingCommands {
    [HttpCommand(Route = "book")]
    public record BookRoom(
        string   BookingId,
        string   GuestId,
        string   RoomId,
        DateTime CheckInDate,
        DateTime CheckOutDate,
        float    BookingPrice,
        string   Currency
    );

    [HttpCommand(Route = "payment")]
    public record RecordPayment(
        string BookingId,
        string PaymentId,
        float  PaidAmount,
        string Currency
    );

    [HttpCommand(Route = "cancel")]
    public record CancelBooking(string BookingId, string Reason);
}
```

> Annotating the static class with `[HttpCommands<BookingState>]` and each command with `[HttpCommand(Route = "...")]` enables `app.MapDiscoveredCommands<BookingState>()` to auto-register all Minimal API endpoints at startup.

### BookingsCommandService.cs

```csharp
using Eventuous;
using Bookings.Domain.Bookings;
using static Bookings.Application.BookingCommands;

namespace Bookings.Application;

public class BookingsCommandService : CommandService<Booking, BookingState, BookingId> {
    public BookingsCommandService(IEventStore store) : base(store) {
        On<BookRoom>()
            .InState(ExpectedState.New)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.BookRoom(
                cmd.GuestId,
                cmd.RoomId,
                cmd.CheckInDate,
                cmd.CheckOutDate,
                cmd.BookingPrice,
                cmd.Currency
            ));

        On<RecordPayment>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.RecordPayment(cmd.PaymentId, cmd.PaidAmount, cmd.Currency));

        On<CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Cancel(cmd.Reason));
    }
}
```

### PaymentsIntegrationHandler.cs (EventHandler for subscription)

```csharp
using Eventuous;
using Eventuous.Subscriptions;
using Bookings.Domain.Bookings;
using static Bookings.Application.BookingCommands;
using EventHandler = Eventuous.Subscriptions.EventHandler;

namespace Bookings.Integration;

// Integration event received from an external payments stream
static class IntegrationEvents {
    [EventType("BookingPaymentRecorded")]
    public record BookingPaymentRecorded(string PaymentId, string BookingId, float Amount, string Currency);
}

public class PaymentsIntegrationHandler : EventHandler {
    public static readonly StreamName Stream = new("PaymentsIntegration");

    readonly ICommandService<BookingState> _commandService;

    public PaymentsIntegrationHandler(ICommandService<BookingState> commandService) {
        _commandService = commandService;
        On<IntegrationEvents.BookingPaymentRecorded>(async ctx =>
            await _commandService.Handle(
                new RecordPayment(ctx.Message.BookingId, ctx.Message.PaymentId, ctx.Message.Amount, ctx.Message.Currency),
                ctx.CancellationToken
            )
        );
    }
}
```

---

## Program.cs

```csharp
using Bookings.Application;
using Bookings.Domain.Bookings;
using Bookings.Integration;
using Eventuous;
using Eventuous.AspNetCore.Web;
using Eventuous.Diagnostics.OpenTelemetry;
using Eventuous.KurrentDB;
using Eventuous.KurrentDB.Subscriptions;
using Eventuous.Projections.MongoDB;
using Eventuous.Spyglass;
using MongoDB.Driver;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// ---------------------------------------------------------------------------
// 1. KurrentDB client — uses kurrentdb:// connection scheme
// ---------------------------------------------------------------------------
builder.Services.AddKurrentDBClient(
    builder.Configuration["KurrentDB:ConnectionString"]!
);

// ---------------------------------------------------------------------------
// 2. Event store — registers IEventStore, IEventReader, IEventWriter
//    KurrentDBEventStore implements all three interfaces.
//    Use IEventReader.LoadAggregate<>() and IEventWriter.StoreAggregate<>()
//    extension methods; IAggregateStore is deprecated.
// ---------------------------------------------------------------------------
builder.Services.AddEventStore<KurrentDBEventStore>();

// ---------------------------------------------------------------------------
// 3. Command service
//    AddCommandService registers the concrete type and ICommandService<TState>
// ---------------------------------------------------------------------------
builder.Services.AddCommandService<BookingsCommandService, BookingState>();

// ---------------------------------------------------------------------------
// 4. MongoDB for checkpoint store and projections
//    AllStreamSubscription (catch-up) requires an external checkpoint store.
// ---------------------------------------------------------------------------
builder.Services.AddSingleton<IMongoDatabase>(_ => {
    var settings = MongoClientSettings.FromConnectionString(
        builder.Configuration["Mongo:ConnectionString"]!
    );
    var client = new MongoClient(settings);
    return client.GetDatabase(builder.Configuration["Mongo:Database"]!);
});

// ---------------------------------------------------------------------------
// 5. AllStreamSubscription — subscribes to $all, used for projections.
//    Requires a checkpoint store. MongoCheckpointStore stores position in MongoDB.
//    WithPartitioningByStream(2) enables parallel processing per stream.
// ---------------------------------------------------------------------------
builder.Services.AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(
    "BookingsProjections",
    b => b
        .UseCheckpointStore<MongoCheckpointStore>()
        // Replace BookingStateProjection with your own IEventHandler implementations:
        // .AddEventHandler<BookingStateProjection>()
        // .AddEventHandler<MyBookingsProjection>()
        .WithPartitioningByStream(2)
);

// ---------------------------------------------------------------------------
// 6. StreamPersistentSubscription — KurrentDB manages the checkpoint server-side.
//    No checkpoint store registration is needed.
//    Used here to pick up payment events from an external stream and issue
//    commands into the Bookings aggregate.
// ---------------------------------------------------------------------------
builder.Services.AddSubscription<StreamPersistentSubscription, StreamPersistentSubscriptionOptions>(
    "PaymentIntegration",
    b => b
        .Configure(opts => opts.StreamName = PaymentsIntegrationHandler.Stream)
        .AddEventHandler<PaymentsIntegrationHandler>()
);

// ---------------------------------------------------------------------------
// 7. HTTP API — controllers or Minimal API
//    Controllers: AddControllers() + app.MapControllers()
//    Minimal API auto-discovery: reads [HttpCommands<TState>] / [HttpCommand] attributes
// ---------------------------------------------------------------------------
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ---------------------------------------------------------------------------
// 8. OpenTelemetry diagnostics (optional but recommended)
// ---------------------------------------------------------------------------
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("bookings"))
        .AddAspNetCoreInstrumentation()
        .AddGrpcClientInstrumentation()
        .AddEventuousTracing()
    )
    .WithMetrics(b => b
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("bookings"))
        .AddAspNetCoreInstrumentation()
        .AddEventuous()
        .AddEventuousSubscriptions()
        .AddPrometheusExporter()
    );

// ---------------------------------------------------------------------------
// Build and configure the pipeline
// ---------------------------------------------------------------------------
var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

// Map controller-based API (e.g., CommandHttpApiBase<TState> controllers)
app.MapControllers();

// Map Minimal API endpoints discovered from [HttpCommands<BookingState>] attributes.
// POST /booking/book        -> BookRoom command
// POST /booking/payment     -> RecordPayment command
// POST /booking/cancel      -> CancelBooking command
app.MapDiscoveredCommands<BookingState>();

// Prometheus scraping endpoint for metrics
app.UseOpenTelemetryPrometheusScrapingEndpoint();

// Eventuous Spyglass — diagnostics/introspection endpoint at /spyglass
app.MapEventuousSpyglass();

app.Run();
```

---

## appsettings.json

```json
{
  "KurrentDB": {
    "ConnectionString": "kurrentdb://localhost:2113?tls=false"
  },
  "Mongo": {
    "ConnectionString": "mongodb://mongoadmin:secret@localhost:27017",
    "Database": "Bookings"
  },
  "AllowedHosts": "*",
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

---

## How It Fits Together

### Registration sequence

| Step | Call | What it registers |
|------|------|-------------------|
| 1 | `AddKurrentDBClient(...)` | `KurrentDBClient` singleton (gRPC client from `KurrentDB.Client`) |
| 2 | `AddEventStore<KurrentDBEventStore>()` | `IEventStore`, `IEventReader`, `IEventWriter` — all backed by `KurrentDBEventStore`, with tracing wrappers |
| 3 | `AddCommandService<BookingsCommandService, BookingState>()` | The concrete service + `ICommandService<BookingState>` |
| 4 | MongoDB client | `IMongoDatabase` for checkpoint store and projections |
| 5 | `AddSubscription<AllStreamSubscription, ...>()` | Catch-up subscription on `$all`; checkpoint stored in MongoDB |
| 6 | `AddSubscription<StreamPersistentSubscription, ...>()` | Persistent subscription on a named stream; KurrentDB owns the checkpoint |

### Checkpoint store notes

- `AllStreamSubscription` and `StreamSubscription` (catch-up) **require** an external checkpoint store. Register one globally with `services.AddCheckpointStore<MongoCheckpointStore>()`, or per-subscription with `.UseCheckpointStore<MongoCheckpointStore>()` as shown above.
- `StreamPersistentSubscription` and `AllPersistentSubscription` **do not** need a checkpoint store.

### HTTP API approaches

Two complementary options can coexist:

**Controller-based** — extend `CommandHttpApiBase<TState>`:

```csharp
[Route("/booking")]
public class BookingCommandApi(ICommandService<BookingState> service)
    : CommandHttpApiBase<BookingState>(service) {

    [HttpPost("book")]
    public Task<ActionResult<Result<BookingState>.Ok>> BookRoom(
        [FromBody] BookRoom cmd, CancellationToken ct) => Handle(cmd, ct);
}
```

**Minimal API auto-discovery** — annotate commands and call `MapDiscoveredCommands`:

```csharp
// On the static command class:
[HttpCommands<BookingState>]
public static class BookingCommands {
    [HttpCommand(Route = "book")]
    public record BookRoom(...);
}

// In Program.cs:
app.MapDiscoveredCommands<BookingState>();
```

Both return `Result<BookingState>.Ok` on success (containing `State` and `Changes`) and a problem details response on failure.

### Event type registration

Events decorated with `[EventType("...")]` are registered in `TypeMap` automatically at compile time via source generation. No call to `TypeMap.RegisterKnownEventTypes()` is needed.

### Connection string format

Always use the `kurrentdb://` scheme with `AddKurrentDBClient`. Do not use `AddEventStoreClient` or the `esdb://` scheme with `KurrentDBEventStore`.

```
kurrentdb://localhost:2113?tls=false          # single node, no TLS (dev)
kurrentdb://user:pass@host1,host2,host3:2113  # cluster with credentials
kurrentdb+discover://host:2113?tls=true       # DNS discovery
```
