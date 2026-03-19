# Minimal Eventuous DI Setup for ASP.NET Core

This guide covers registering event types and wiring up DI for an ASP.NET Core app with an event store and subscriptions.

## NuGet Packages

```xml
<PackageReference Include="Eventuous.AspNetCore" Version="0.*" />
<PackageReference Include="Eventuous.EventStore" Version="0.*" />
<!-- Or for PostgreSQL: -->
<!-- <PackageReference Include="Eventuous.Postgresql" Version="0.*" /> -->
<PackageReference Include="Eventuous.EventStore.Subscriptions" Version="0.*" />
```

## 1. Define Your Domain Events

```csharp
// Events/BookingEvents.cs
namespace MyApp.Events;

[EventType("V1.BookingInitiated")]
public record BookingInitiated(string BookingId, string RoomId, decimal Price);

[EventType("V1.BookingConfirmed")]
public record BookingConfirmed(string BookingId);

[EventType("V1.BookingCancelled")]
public record BookingCancelled(string BookingId, string Reason);
```

The `[EventType]` attribute registers the event with a stable string name used for serialization. This is important so that renaming a C# class does not break stored events.

## 2. Define Your Aggregate and State

```csharp
// Domain/Booking.cs
namespace MyApp.Domain;

public class Booking : Aggregate<BookingState> {
    public void Initiate(string bookingId, string roomId, decimal price) {
        EnsureDoesntExist();
        Apply(new BookingInitiated(bookingId, roomId, price));
    }

    public void Confirm() {
        EnsureExists();
        if (CurrentState.IsConfirmed)
            throw new InvalidOperationException("Already confirmed");
        Apply(new BookingConfirmed(CurrentState.BookingId));
    }

    public void Cancel(string reason) {
        EnsureExists();
        Apply(new BookingCancelled(CurrentState.BookingId, reason));
    }
}

public record BookingState : State<BookingState> {
    public string BookingId  { get; init; } = "";
    public string RoomId     { get; init; } = "";
    public decimal Price     { get; init; }
    public bool IsConfirmed  { get; init; }

    public override BookingState When(object @event) => @event switch {
        BookingInitiated e  => this with { BookingId = e.BookingId, RoomId = e.RoomId, Price = e.Price },
        BookingConfirmed _  => this with { IsConfirmed = true },
        BookingCancelled _  => this,
        _                   => this
    };
}

public record BookingId(string Value) : Id(Value);
```

## 3. Define a Command Service

```csharp
// Application/BookingService.cs
namespace MyApp.Application;

public class BookingService : CommandService<Booking, BookingState, BookingId> {
    public BookingService(IEventStore store) : base(store) {
        On<InitiateBooking>()
            .InState(ExpectedState.New)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Initiate(cmd.BookingId, cmd.RoomId, cmd.Price));

        On<ConfirmBooking>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Confirm());

        On<CancelBooking>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new BookingId(cmd.BookingId))
            .Act((booking, cmd) => booking.Cancel(cmd.Reason));
    }
}

// Commands
public record InitiateBooking(string BookingId, string RoomId, decimal Price);
public record ConfirmBooking(string BookingId);
public record CancelBooking(string BookingId, string Reason);
```

## 4. Define a Subscription Event Handler

```csharp
// Projections/BookingProjection.cs
namespace MyApp.Projections;

public class BookingProjection : IEventHandler {
    readonly ILogger<BookingProjection> _logger;

    public BookingProjection(ILogger<BookingProjection> logger) {
        _logger = logger;
    }

    public string DiagnosticName => "BookingProjection";

    public async ValueTask<EventHandlingStatus> HandleEvent(IMessageConsumeContext context) {
        switch (context.Message) {
            case BookingInitiated e:
                _logger.LogInformation("Booking initiated: {BookingId}", e.BookingId);
                // Write to your read model here
                return EventHandlingStatus.Success;

            case BookingConfirmed e:
                _logger.LogInformation("Booking confirmed: {BookingId}", e.BookingId);
                return EventHandlingStatus.Success;

            default:
                return EventHandlingStatus.Ignored;
        }
    }
}
```

## 5. Wire Up DI in Program.cs (EventStoreDB)

```csharp
// Program.cs
using EventStore.Client;
using Eventuous;
using Eventuous.AspNetCore;
using Eventuous.EventStore;
using Eventuous.EventStore.Subscriptions;

var builder = WebApplication.CreateBuilder(args);

// ── 1. Register event types from all assemblies that use [EventType] ──────────
// TypeMap auto-discovers types decorated with [EventType] during startup.
// You can also register manually:
TypeMap.RegisterKnownEventTypes(typeof(BookingInitiated).Assembly);

// ── 2. EventStoreDB client ────────────────────────────────────────────────────
builder.Services.AddEventStoreClient(
    builder.Configuration["EventStore:ConnectionString"]!
    // e.g. "esdb://localhost:2113?tls=false"
);

// ── 3. Eventuous core: event store + serializer ───────────────────────────────
builder.Services.AddEventuous(eb => eb
    .UseEventStore()           // registers IEventStore backed by EventStoreDB
    .AddEventSerializer()      // registers the default JSON serializer
);

// ── 4. Command services ───────────────────────────────────────────────────────
builder.Services.AddCommandService<BookingService>();

// ── 5. Subscriptions ─────────────────────────────────────────────────────────
builder.Services.AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(
    "BookingsSubscription",
    b => b
        .Configure(opts => opts.ConcurrencyLimit = 2)
        .AddEventHandler<BookingProjection>()
);

// ── 6. Standard ASP.NET Core services ────────────────────────────────────────
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

// ── 7. Eventuous middleware (exposes /health and diagnostic endpoints) ────────
app.UseEventuous();

app.MapControllers();
app.Run();
```

## 6. Command HTTP Endpoint (minimal API)

```csharp
// Controllers/BookingController.cs
namespace MyApp.Controllers;

[ApiController]
[Route("bookings")]
public class BookingController : ControllerBase {
    readonly BookingService _service;

    public BookingController(BookingService service) => _service = service;

    [HttpPost]
    public async Task<IActionResult> Initiate([FromBody] InitiateBooking cmd, CancellationToken ct) {
        var result = await _service.Handle(cmd, ct);
        return result.Success ? Ok(result) : BadRequest(result.ErrorMessage);
    }

    [HttpPost("{id}/confirm")]
    public async Task<IActionResult> Confirm(string id, CancellationToken ct) {
        var result = await _service.Handle(new ConfirmBooking(id), ct);
        return result.Success ? Ok(result) : BadRequest(result.ErrorMessage);
    }
}
```

## Key Concepts

### Event Type Registration

`TypeMap.RegisterKnownEventTypes(assembly)` scans the assembly for all types decorated with `[EventType("name")]` and registers them. This mapping is required for serialization/deserialization — stored events use the string name, not the CLR type name.

You can also register types manually:

```csharp
TypeMap.AddType<BookingInitiated>("V1.BookingInitiated");
```

### Stream Naming

By default, Eventuous uses the pattern `{AggregateTypeName}-{AggregateId}` for stream names. For a `Booking` aggregate with id `abc-123`, the stream will be `Booking-abc-123`. You can override this via `StreamNameMap`.

### Subscription Checkpoints

`AllStreamSubscription` reads all events from the `$all` stream in EventStoreDB. For production, attach a checkpoint store so the subscription resumes after restart:

```csharp
builder.Services.AddSubscription<AllStreamSubscription, AllStreamSubscriptionOptions>(
    "BookingsSubscription",
    b => b
        .UseCheckpointStore<MongoCheckpointStore>()  // or EsdbCheckpointStore, PostgresCheckpointStore
        .AddEventHandler<BookingProjection>()
);
```

### PostgreSQL Alternative

Swap out the EventStoreDB packages for PostgreSQL:

```csharp
// NuGet: Eventuous.Postgresql
builder.Services.AddEventuous(eb => eb
    .UsePostgresEventStore()
    .AddEventSerializer()
);
builder.Services.AddNpgsqlDataSource(connectionString);
```

## appsettings.json

```json
{
  "EventStore": {
    "ConnectionString": "esdb://localhost:2113?tls=false"
  }
}
```

## Summary

The minimal steps are:

1. Decorate events with `[EventType("stable-name")]`
2. Call `TypeMap.RegisterKnownEventTypes(assembly)` before the app starts
3. Register the event store client and call `AddEventuous(eb => eb.UseEventStore()...)`
4. Register command services with `AddCommandService<TService>()`
5. Register subscriptions with `AddSubscription<TSubscription, TOptions>(...)` and attach handlers via `.AddEventHandler<THandler>()`
6. Call `app.UseEventuous()` for middleware
