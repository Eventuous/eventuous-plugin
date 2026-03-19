# Order Aggregate with Eventuous

This example models an `Order` aggregate that supports three operations: placing an order, confirming it, and cancelling it.

## Project Setup

Add the Eventuous NuGet packages:

```xml
<PackageReference Include="Eventuous" Version="*" />
```

---

## Domain Events

Events are immutable records decorated with `[EventType]` for serialization. They use primitive types (not value objects) and are grouped in a versioned static class to support schema evolution.

```csharp
// OrderEvents.cs
using Eventuous;

public static class OrderEvents {
    public static class V1 {
        [EventType("V1.OrderPlaced")]
        public record OrderPlaced(
            string OrderId,
            string CustomerId,
            string[] Items,
            float   TotalAmount,
            string  Currency
        );

        [EventType("V1.OrderConfirmed")]
        public record OrderConfirmed(
            string OrderId,
            string ConfirmedAt
        );

        [EventType("V1.OrderCancelled")]
        public record OrderCancelled(
            string OrderId,
            string Reason,
            string CancelledAt
        );
    }
}
```

---

## Identity

A strongly-typed ID extends the abstract `Id` record. The base class validates non-empty strings and provides implicit string conversion.

```csharp
// OrderId.cs
using Eventuous;

public record OrderId(string Value) : Id(Value);
```

---

## State

State is an abstract record reconstructed by folding events. Event handlers are registered in the parameterless constructor using `On<TEvent>()`. Each handler is a pure function that returns new state via `with` expressions.

```csharp
// OrderState.cs
using Eventuous;
using static OrderEvents.V1;

public record OrderState : State<OrderState> {
    public string   CustomerId  { get; init; } = null!;
    public string[] Items       { get; init; } = [];
    public float    TotalAmount { get; init; }
    public string   Currency    { get; init; } = null!;
    public OrderStatus Status   { get; init; } = OrderStatus.None;

    public OrderState() {
        On<OrderPlaced>((state, e) => state with {
            CustomerId  = e.CustomerId,
            Items       = e.Items,
            TotalAmount = e.TotalAmount,
            Currency    = e.Currency,
            Status      = OrderStatus.Placed
        });

        On<OrderConfirmed>((state, _) => state with {
            Status = OrderStatus.Confirmed
        });

        On<OrderCancelled>((state, _) => state with {
            Status = OrderStatus.Cancelled
        });
    }
}

public enum OrderStatus {
    None,
    Placed,
    Confirmed,
    Cancelled
}
```

---

## Aggregate

The aggregate enforces business invariants and records domain events. It extends `Aggregate<TState>` and calls `Apply<TEvent>()` to record each event (which updates state and adds the event to pending changes).

```csharp
// Order.cs
using Eventuous;
using static OrderEvents.V1;

public class Order : Aggregate<OrderState> {
    public void Place(string orderId, string customerId, string[] items, float totalAmount, string currency) {
        EnsureDoesntExist();

        if (items is not { Length: > 0 })
            throw new DomainException("An order must contain at least one item.");

        if (totalAmount <= 0)
            throw new DomainException("Order total amount must be positive.");

        Apply(new OrderPlaced(
            orderId,
            customerId,
            items,
            totalAmount,
            currency
        ));
    }

    public void Confirm() {
        EnsureExists();

        if (State.Status == OrderStatus.Confirmed)
            throw new DomainException("Order is already confirmed.");

        if (State.Status == OrderStatus.Cancelled)
            throw new DomainException("Cannot confirm a cancelled order.");

        Apply(new OrderConfirmed(
            State.GetId<string>(),
            DateTime.UtcNow.ToString("O")
        ));
    }

    public void Cancel(string reason) {
        EnsureExists();

        if (State.Status == OrderStatus.Cancelled)
            throw new DomainException("Order is already cancelled.");

        if (string.IsNullOrWhiteSpace(reason))
            throw new DomainException("A cancellation reason must be provided.");

        Apply(new OrderCancelled(
            State.GetId<string>(),
            reason,
            DateTime.UtcNow.ToString("O")
        ));
    }
}
```

---

## Commands

Commands are record types grouped in a static class.

```csharp
// OrderCommands.cs
public static class OrderCommands {
    public record PlaceOrder(
        string   OrderId,
        string   CustomerId,
        string[] Items,
        float    TotalAmount,
        string   Currency
    );

    public record ConfirmOrder(string OrderId);

    public record CancelOrder(string OrderId, string Reason);
}
```

---

## Command Service

The aggregate-based command service wires commands to aggregate methods. It extends `CommandService<TAggregate, TState, TId>` and registers handlers in the constructor.

```csharp
// OrderCommandService.cs
using Eventuous;
using static OrderCommands;

public class OrderCommandService : CommandService<Order, OrderState, OrderId> {
    public OrderCommandService(IEventStore store) : base(store) {
        On<PlaceOrder>()
            .InState(ExpectedState.New)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Place(
                cmd.OrderId,
                cmd.CustomerId,
                cmd.Items,
                cmd.TotalAmount,
                cmd.Currency
            ));

        On<ConfirmOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, _) => order.Confirm());

        On<CancelOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Cancel(cmd.Reason));
    }
}
```

---

## DI Registration (Program.cs)

```csharp
using Eventuous;
using Eventuous.EventStore; // or whichever event store you use

// 1. Register event types for serialization
TypeMap.RegisterKnownEventTypes(typeof(OrderEvents).Assembly);

// 2. Configure serialization (optional, uses System.Text.Json by default)
DefaultEventSerializer.SetDefaultSerializer(
    new DefaultEventSerializer(new JsonSerializerOptions(JsonSerializerDefaults.Web))
);

var builder = WebApplication.CreateBuilder(args);

// 3. Register event store (KurrentDB/EventStoreDB is the recommended default)
builder.Services.AddEventStoreClient("esdb://localhost:2113?tls=false");
builder.Services.AddAggregateStore<EsdbEventStore>();

// 4. Register the command service
builder.Services.AddCommandService<OrderCommandService, OrderState>();

var app = builder.Build();
app.Run();
```

---

## Usage Example

```csharp
// Inject ICommandService<OrderState> or OrderCommandService
var result = await commandService.Handle(
    new OrderCommands.PlaceOrder(
        OrderId:     "order-001",
        CustomerId:  "customer-42",
        Items:       ["SKU-A", "SKU-B"],
        TotalAmount: 99.99f,
        Currency:    "USD"
    ),
    cancellationToken
);

result.Match(
    ok    => Console.WriteLine($"Order placed. Stream version: {ok.Changes.Count} events."),
    error => Console.WriteLine($"Failed: {error.ErrorMessage}")
);

// Confirm it
await commandService.Handle(new OrderCommands.ConfirmOrder("order-001"), ct);

// Or cancel it
await commandService.Handle(new OrderCommands.CancelOrder("order-001", "Customer requested cancellation"), ct);
```

---

## Key Design Points

- **Events use primitives** — `string`, `float`, `string[]` instead of value objects. This makes serialization straightforward and avoids coupling the event schema to domain types.
- **State is reconstructed by folding events** — `OrderState` is rebuilt entirely from the event stream each time the aggregate is loaded; it holds no logic, only data.
- **Invariants live in the aggregate** — all business rules (e.g., cannot confirm a cancelled order) are enforced in `Order` before `Apply()` is called.
- **`EnsureExists()` / `EnsureDoesntExist()`** — guard helpers provided by `Aggregate<T>` that throw `DomainException` if the aggregate is in the wrong lifecycle state.
- **`[EventType]` drives serialization** — the string name in the attribute is the stable event type name persisted to the store. Rename the C# class freely without breaking stored events.
- **Stream name** — defaults to `Order-order-001` following the `{AggregateType}-{AggregateId}` convention.
