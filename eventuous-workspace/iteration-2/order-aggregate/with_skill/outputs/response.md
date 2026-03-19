# Order Aggregate with Eventuous

This guide shows how to model an `Order` aggregate with Eventuous, covering:
- Domain events
- Aggregate identity
- State
- Aggregate with business logic
- Commands
- Command service
- DI registration
- HTTP API

---

## Project Setup

Install the required NuGet packages:

```xml
<PackageReference Include="Eventuous" Version="*" />
<PackageReference Include="Eventuous.Application" Version="*" />
<PackageReference Include="Eventuous.KurrentDB" Version="*" />
<PackageReference Include="Eventuous.AspNetCore" Version="*" />
<PackageReference Include="Eventuous.AspNetCore.Web" Version="*" />
```

---

## Domain Events

Events are immutable records decorated with `[EventType]` for serialization. Group them in a static class with versioned nested classes to allow future schema evolution. Use primitive types only — no value objects in events.

```csharp
// Domain/OrderEvents.cs
using Eventuous;

public static class OrderEvents {
    public static class V1 {
        [EventType("V1.OrderPlaced")]
        public record OrderPlaced(
            string CustomerId,
            string[] ItemIds,
            float   TotalAmount,
            string  Currency
        );

        [EventType("V1.OrderConfirmed")]
        public record OrderConfirmed(string ConfirmedBy);

        [EventType("V1.OrderCancelled")]
        public record OrderCancelled(string Reason);
    }
}
```

---

## Aggregate Identity

Strongly-typed IDs extend the abstract `Id` record. The base class validates non-empty strings and provides implicit string conversion.

```csharp
// Domain/OrderId.cs
using Eventuous;

public record OrderId(string Value) : Id(Value);
```

---

## State

State is an abstract record reconstructed by folding events. Register event handlers in the parameterless constructor using `On<TEvent>()`. Handlers are pure functions that return new state via `with` expressions.

`OrderStatus` tracks the lifecycle so the aggregate can guard against invalid transitions.

```csharp
// Domain/OrderState.cs
using Eventuous;
using static OrderEvents.V1;

public enum OrderStatus { New, Placed, Confirmed, Cancelled }

public record OrderState : State<OrderState> {
    public string      CustomerId { get; init; } = null!;
    public string[]    ItemIds    { get; init; } = [];
    public float       Amount     { get; init; }
    public string      Currency   { get; init; } = null!;
    public OrderStatus Status     { get; init; } = OrderStatus.New;

    public OrderState() {
        On<OrderPlaced>((state, e) => state with {
            CustomerId = e.CustomerId,
            ItemIds    = e.ItemIds,
            Amount     = e.TotalAmount,
            Currency   = e.Currency,
            Status     = OrderStatus.Placed
        });

        On<OrderConfirmed>((state, _) => state with { Status = OrderStatus.Confirmed });

        On<OrderCancelled>((state, _) => state with { Status = OrderStatus.Cancelled });
    }
}
```

---

## Aggregate

The aggregate enforces business invariants and records events via `Apply<TEvent>()`. `EnsureDoesntExist()` guards against re-creating an existing stream; `EnsureExists()` guards against operating on a stream that has never been created.

```csharp
// Domain/Order.cs
using Eventuous;
using static OrderEvents.V1;

public class Order : Aggregate<OrderState> {

    /// <summary>Place a new order. The aggregate must not exist yet.</summary>
    public void Place(string customerId, string[] itemIds, float totalAmount, string currency) {
        EnsureDoesntExist();
        if (itemIds is not { Length: > 0 })
            throw new DomainException("An order must contain at least one item.");
        if (totalAmount <= 0)
            throw new DomainException("Order total must be greater than zero.");

        Apply(new OrderPlaced(customerId, itemIds, totalAmount, currency));
    }

    /// <summary>Confirm a placed order.</summary>
    public void Confirm(string confirmedBy) {
        EnsureExists();
        if (State.Status == OrderStatus.Cancelled)
            throw new DomainException("Cannot confirm a cancelled order.");
        if (State.Status == OrderStatus.Confirmed)
            throw new DomainException("Order is already confirmed.");

        Apply(new OrderConfirmed(confirmedBy));
    }

    /// <summary>Cancel an order that has not already been confirmed or cancelled.</summary>
    public void Cancel(string reason) {
        EnsureExists();
        if (State.Status == OrderStatus.Confirmed)
            throw new DomainException("Cannot cancel a confirmed order.");
        if (State.Status == OrderStatus.Cancelled)
            throw new DomainException("Order is already cancelled.");

        Apply(new OrderCancelled(reason));
    }
}
```

---

## Commands

Commands are simple records. Grouping them in a static class and annotating with `[HttpCommands]` enables Minimal API auto-discovery.

```csharp
// Application/OrderCommands.cs
using Eventuous.AspNetCore.Web;

[HttpCommands<OrderState>]
public static class OrderCommands {
    [HttpCommand(Route = "place")]
    public record PlaceOrder(
        string   OrderId,
        string   CustomerId,
        string[] ItemIds,
        float    TotalAmount,
        string   Currency
    );

    [HttpCommand(Route = "confirm")]
    public record ConfirmOrder(string OrderId, string ConfirmedBy);

    [HttpCommand(Route = "cancel")]
    public record CancelOrder(string OrderId, string Reason);
}
```

---

## Command Service

The command service wires commands to aggregate methods. It extends `CommandService<TAggregate, TState, TId>` and registers handlers in the constructor using the fluent builder chain.

- `InState(ExpectedState.New)` — stream must not exist yet (for `PlaceOrder`)
- `InState(ExpectedState.Existing)` — stream must already exist (for `ConfirmOrder` and `CancelOrder`)
- `GetId(cmd => ...)` — extracts the aggregate ID from the command
- `Act((order, cmd) => ...)` — calls the aggregate method

```csharp
// Application/OrderCommandService.cs
using Eventuous;
using Eventuous.Application;

public class OrderCommandService : CommandService<Order, OrderState, OrderId> {
    public OrderCommandService(IEventStore store) : base(store) {

        On<OrderCommands.PlaceOrder>()
            .InState(ExpectedState.New)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Place(
                cmd.CustomerId,
                cmd.ItemIds,
                cmd.TotalAmount,
                cmd.Currency
            ));

        On<OrderCommands.ConfirmOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Confirm(cmd.ConfirmedBy));

        On<OrderCommands.CancelOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Cancel(cmd.Reason));
    }
}
```

---

## DI Registration and HTTP API (Program.cs)

```csharp
// Program.cs
using Eventuous;
using Eventuous.KurrentDB;

var builder = WebApplication.CreateBuilder(args);

// 1. Register KurrentDB client
builder.Services.AddKurrentDBClient(
    builder.Configuration["KurrentDB:ConnectionString"]!
);

// 2. Register the event store backed by KurrentDB
builder.Services.AddEventStore<KurrentDBEventStore>();

// 3. Register the command service
//    AddCommandService wires up ICommandService<OrderState> and the concrete type
builder.Services.AddCommandService<OrderCommandService, OrderState>();

// 4. (Optional) Add OpenTelemetry diagnostics
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddEventuousTracing())
    .WithMetrics(m => m.AddEventuous());

// 5. Add controllers if using a controller-based API
builder.Services.AddControllers();

var app = builder.Build();

// 6. Map all commands discovered via [HttpCommands<OrderState>] / [HttpCommand] attributes
//    This creates POST endpoints at /order/place, /order/confirm, /order/cancel
app.MapDiscoveredCommands<OrderState>(routePrefix: "order");

app.MapControllers();

// (Optional) Eventuous diagnostic endpoint
app.MapEventuousSpyglass();

app.Run();
```

### appsettings.json

```json
{
  "KurrentDB": {
    "ConnectionString": "kurrentdb://localhost:2113?tls=false"
  }
}
```

---

## Alternative: Controller-Based HTTP API

If you prefer a traditional MVC controller, extend `CommandHttpApiBase<TState>`:

```csharp
// Api/OrderController.cs
using Eventuous.AspNetCore.Web;
using Microsoft.AspNetCore.Mvc;

[Route("/order")]
public class OrderController(ICommandService<OrderState> service)
    : CommandHttpApiBase<OrderState>(service) {

    [HttpPost("place")]
    public Task<ActionResult<Result<OrderState>.Ok>> Place(
        [FromBody] OrderCommands.PlaceOrder cmd, CancellationToken ct)
        => Handle(cmd, ct);

    [HttpPost("confirm")]
    public Task<ActionResult<Result<OrderState>.Ok>> Confirm(
        [FromBody] OrderCommands.ConfirmOrder cmd, CancellationToken ct)
        => Handle(cmd, ct);

    [HttpPost("cancel")]
    public Task<ActionResult<Result<OrderState>.Ok>> Cancel(
        [FromBody] OrderCommands.CancelOrder cmd, CancellationToken ct)
        => Handle(cmd, ct);
}
```

With this approach, replace `app.MapDiscoveredCommands<OrderState>()` with standard controller mapping:

```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

---

## How It All Fits Together

1. A POST to `/order/place` deserializes the JSON body into `PlaceOrder`.
2. `ICommandService<OrderState>` routes it to `OrderCommandService`.
3. The service sees `ExpectedState.New`, creates a fresh `Order` aggregate, and calls `order.Place(...)`.
4. `Place()` enforces invariants and calls `Apply(new OrderPlaced(...))`.
5. `Apply` updates `State` via `OrderState.On<OrderPlaced>` and adds the event to `Changes`.
6. The service persists `Changes` to KurrentDB under stream `Order-{orderId}`.
7. The endpoint returns `Result<OrderState>.Ok` containing the final state and the stored events.

### Stream Naming

By default, stream names follow the pattern `{AggregateType}-{AggregateId}`. For an `OrderId("order-42")` the stream name will be `Order-order-42`.

### Event Type Registration

Events decorated with `[EventType]` are automatically registered in `TypeMap` by the Eventuous source generator at compile time. No manual `TypeMap.RegisterKnownEventTypes()` calls are needed.
