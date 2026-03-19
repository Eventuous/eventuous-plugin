# Modeling an Order Aggregate with Eventuous

This guide walks through building an `Order` aggregate that supports placing, confirming, and cancelling orders using the Eventuous library.

## Project Setup

Add the Eventuous core package to your project:

```xml
<PackageReference Include="Eventuous" Version="0.*" />
```

---

## Events

Events are immutable records that capture what happened. Define them first — they are the source of truth.

```csharp
// Events/OrderEvents.cs
using Eventuous;

namespace MyApp.Orders;

public static class OrderEvents
{
    [EventType("V1.OrderPlaced")]
    public record OrderPlaced(
        string OrderId,
        string CustomerId,
        IReadOnlyList<OrderItem> Items,
        Money TotalAmount,
        DateTimeOffset PlacedAt
    );

    [EventType("V1.OrderConfirmed")]
    public record OrderConfirmed(
        string OrderId,
        DateTimeOffset ConfirmedAt
    );

    [EventType("V1.OrderCancelled")]
    public record OrderCancelled(
        string OrderId,
        string Reason,
        DateTimeOffset CancelledAt
    );
}

// Supporting value types
public record OrderItem(string ProductId, string ProductName, int Quantity, Money UnitPrice);

public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
}
```

> The `[EventType]` attribute registers the event type name used during serialization. Eventuous uses this to resolve event types from the stream.

---

## Order ID

Eventuous requires a strongly-typed identity record that inherits from `Id`.

```csharp
// Domain/OrderId.cs
using Eventuous;

namespace MyApp.Orders;

public record OrderId(string Value) : Id(Value);
```

---

## Order Status

An enum to track the lifecycle state of the order:

```csharp
// Domain/OrderStatus.cs
namespace MyApp.Orders;

public enum OrderStatus
{
    None,
    Placed,
    Confirmed,
    Cancelled
}
```

---

## Order State

`State<T>` is an immutable record that is reconstructed by folding events. It contains no business logic — just data derived from events.

```csharp
// Domain/OrderState.cs
using Eventuous;
using static MyApp.Orders.OrderEvents;

namespace MyApp.Orders;

public record OrderState : State<OrderState>
{
    public OrderId Id { get; init; } = null!;
    public string CustomerId { get; init; } = string.Empty;
    public IReadOnlyList<OrderItem> Items { get; init; } = [];
    public Money TotalAmount { get; init; } = Money.Zero("USD");
    public OrderStatus Status { get; init; } = OrderStatus.None;

    // Register event handlers in the constructor
    public OrderState()
    {
        On<OrderPlaced>(Handle);
        On<OrderConfirmed>(Handle);
        On<OrderCancelled>(Handle);
    }

    static OrderState Handle(OrderState state, OrderPlaced e) =>
        state with
        {
            Id = new OrderId(e.OrderId),
            CustomerId = e.CustomerId,
            Items = e.Items,
            TotalAmount = e.TotalAmount,
            Status = OrderStatus.Placed
        };

    static OrderState Handle(OrderState state, OrderConfirmed _) =>
        state with { Status = OrderStatus.Confirmed };

    static OrderState Handle(OrderState state, OrderCancelled _) =>
        state with { Status = OrderStatus.Cancelled };
}
```

---

## Order Aggregate

The aggregate enforces business rules and raises events. It inherits from `Aggregate<TState>`.

```csharp
// Domain/Order.cs
using Eventuous;
using static MyApp.Orders.OrderEvents;

namespace MyApp.Orders;

public class Order : Aggregate<OrderState>
{
    /// <summary>
    /// Place a new order. Can only be called when the order does not yet exist.
    /// </summary>
    public void Place(
        OrderId orderId,
        string customerId,
        IReadOnlyList<OrderItem> items,
        Money totalAmount)
    {
        EnsureDoesntExist();

        if (string.IsNullOrWhiteSpace(customerId))
            throw new DomainException("Customer ID must be provided.");

        if (items == null || items.Count == 0)
            throw new DomainException("Order must contain at least one item.");

        if (totalAmount.Amount <= 0)
            throw new DomainException("Order total must be positive.");

        Apply(new OrderPlaced(
            orderId.Value,
            customerId,
            items,
            totalAmount,
            DateTimeOffset.UtcNow
        ));
    }

    /// <summary>
    /// Confirm a placed order. Can only be confirmed if currently in Placed status.
    /// </summary>
    public void Confirm()
    {
        EnsureExists();

        if (Current.Status != OrderStatus.Placed)
            throw new DomainException($"Cannot confirm an order in '{Current.Status}' status.");

        Apply(new OrderConfirmed(
            Current.Id.Value,
            DateTimeOffset.UtcNow
        ));
    }

    /// <summary>
    /// Cancel an order. Can only cancel if placed or confirmed (not already cancelled).
    /// </summary>
    public void Cancel(string reason)
    {
        EnsureExists();

        if (Current.Status == OrderStatus.Cancelled)
            throw new DomainException("Order is already cancelled.");

        if (string.IsNullOrWhiteSpace(reason))
            throw new DomainException("A cancellation reason must be provided.");

        Apply(new OrderCancelled(
            Current.Id.Value,
            reason,
            DateTimeOffset.UtcNow
        ));
    }
}
```

Key points:
- `EnsureDoesntExist()` — built-in guard that throws if the aggregate already has events (i.e., already created).
- `EnsureExists()` — built-in guard that throws if the aggregate has no events (i.e., not yet created).
- `Current` — the current `OrderState`, derived from applied events.
- `Apply(event)` — records the event into `Changes` and folds it into `Current` via the state's `When` handler.

---

## Command Service

Wire everything together with a `CommandService` that loads the aggregate, executes commands, and persists events.

```csharp
// Application/OrderService.cs
using Eventuous;
using Eventuous.Application;

namespace MyApp.Orders;

public class OrderCommandService : CommandService<Order, OrderState, OrderId>
{
    public OrderCommandService(IEventStore store) : base(store)
    {
        On<PlaceOrder>()
            .InState(ExpectedState.New)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Place(
                new OrderId(cmd.OrderId),
                cmd.CustomerId,
                cmd.Items,
                cmd.TotalAmount
            ));

        On<ConfirmOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Confirm());

        On<CancelOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Cancel(cmd.Reason));
    }
}
```

---

## Commands

Commands are simple records representing the caller's intent:

```csharp
// Application/OrderCommands.cs
namespace MyApp.Orders;

public record PlaceOrder(
    string OrderId,
    string CustomerId,
    IReadOnlyList<OrderItem> Items,
    Money TotalAmount
);

public record ConfirmOrder(string OrderId);

public record CancelOrder(string OrderId, string Reason);
```

---

## Registration (ASP.NET Core / DI)

Register the type map and service in your `Program.cs` or startup:

```csharp
// Program.cs
using Eventuous;
using MyApp.Orders;
using static MyApp.Orders.OrderEvents;

// Register event types for serialization
TypeMap.RegisterKnownEventTypes(typeof(OrderPlaced).Assembly);

var builder = WebApplication.CreateBuilder(args);

// Register the event store (e.g. EventStoreDB)
builder.Services.AddEventStoreClient("esdb://localhost:2113?tls=false");
builder.Services.AddAggregateStore<EsdbEventStore>();

// Register the command service
builder.Services.AddCommandService<OrderCommandService, Order, OrderState, OrderId>();

var app = builder.Build();
// ... map endpoints
app.Run();
```

---

## Sending Commands (Minimal API Example)

```csharp
app.MapPost("/orders", async (PlaceOrder cmd, OrderCommandService svc, CancellationToken ct) =>
{
    var result = await svc.Handle(cmd, ct);
    return result.Success ? Results.Ok(result) : Results.BadRequest(result.ErrorMessage);
});

app.MapPost("/orders/{id}/confirm", async (string id, OrderCommandService svc, CancellationToken ct) =>
{
    var result = await svc.Handle(new ConfirmOrder(id), ct);
    return result.Success ? Results.Ok(result) : Results.BadRequest(result.ErrorMessage);
});

app.MapPost("/orders/{id}/cancel", async (string id, string reason, OrderCommandService svc, CancellationToken ct) =>
{
    var result = await svc.Handle(new CancelOrder(id, reason), ct);
    return result.Success ? Results.Ok(result) : Results.BadRequest(result.ErrorMessage);
});
```

---

## Summary

| Concept | Type | Responsibility |
|---|---|---|
| `OrderEvents.*` | Records | Immutable facts; what happened |
| `OrderId` | `Id` record | Strongly-typed identity |
| `OrderState` | `State<OrderState>` | Current view rebuilt from events; no logic |
| `Order` | `Aggregate<OrderState>` | Business rules; raises events |
| `OrderCommandService` | `CommandService<...>` | Orchestrates load, execute, persist |
| `PlaceOrder` / `ConfirmOrder` / `CancelOrder` | Records | Caller intent (commands) |

### Key Eventuous Patterns Used

- **State folding**: `On<TEvent>(handler)` in `State<T>` constructor maps events to state transitions.
- **Optimistic concurrency**: Built into `IEventStore` — conflicts surface as exceptions without extra code.
- **`[EventType]` attribute**: Registers a stable type name for serialization/deserialization across versions.
- **`ExpectedState.New` / `.Existing`**: Declarative preconditions in `CommandService` — no boilerplate loading/checking needed.
- **`EnsureExists()` / `EnsureDoesntExist()`**: Aggregate-level guards for creation vs. mutation scenarios.
