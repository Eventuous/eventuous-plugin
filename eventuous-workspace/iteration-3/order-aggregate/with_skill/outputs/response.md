# Order Aggregate with Eventuous

This guide shows a complete, runnable implementation of an `Order` aggregate in Eventuous, covering:
- Domain events
- State
- Aggregate
- Commands
- Aggregate-based command service
- DI registration with KurrentDB
- HTTP API via minimal API auto-discovery

---

## Project Setup

Create a new ASP.NET Core Web API project and add these NuGet packages:

```xml
<PackageReference Include="Eventuous.AspNetCore" Version="*" />
<PackageReference Include="Eventuous.KurrentDB" Version="*" />
<PackageReference Include="Eventuous.KurrentDB.Subscriptions" Version="*" />
```

---

## 1. Identity

```csharp
// OrderId.cs
namespace Orders.Domain;

public record OrderId(string Value) : Id(Value);
```

---

## 2. Domain Events

Events are immutable records decorated with `[EventType]` for serialization. Group them in a static class with a versioned nested class to support schema evolution.

```csharp
// OrderEvents.cs
namespace Orders.Domain;

using Eventuous;

public static class OrderEvents {
    public static class V1 {
        [EventType("V1.OrderPlaced")]
        public record OrderPlaced(
            string CustomerId,
            string[] ItemSkus,
            decimal TotalAmount,
            string Currency
        );

        [EventType("V1.OrderConfirmed")]
        public record OrderConfirmed(
            string ConfirmedBy,
            DateTimeOffset ConfirmedAt
        );

        [EventType("V1.OrderCancelled")]
        public record OrderCancelled(
            string Reason,
            DateTimeOffset CancelledAt
        );
    }
}
```

> Events use primitive types only — no value objects. This keeps serialization simple and avoids coupling the event schema to the domain model.

---

## 3. State

State is an abstract record reconstructed from events. Register event handlers in the parameterless constructor using `On<TEvent>()`. Handlers are pure static functions that return new state via `with` expressions.

```csharp
// OrderState.cs
namespace Orders.Domain;

using Eventuous;
using static OrderEvents.V1;

public enum OrderStatus { New, Placed, Confirmed, Cancelled }

public record OrderState : State<OrderState> {
    public string      CustomerId  { get; init; } = null!;
    public string[]    ItemSkus    { get; init; } = [];
    public decimal     TotalAmount { get; init; }
    public string      Currency    { get; init; } = null!;
    public OrderStatus Status      { get; init; } = OrderStatus.New;

    public OrderState() {
        On<OrderPlaced>(HandlePlaced);
        On<OrderConfirmed>(HandleConfirmed);
        On<OrderCancelled>(HandleCancelled);
    }

    static OrderState HandlePlaced(OrderState state, OrderPlaced e)
        => state with {
            CustomerId  = e.CustomerId,
            ItemSkus    = e.ItemSkus,
            TotalAmount = e.TotalAmount,
            Currency    = e.Currency,
            Status      = OrderStatus.Placed
        };

    static OrderState HandleConfirmed(OrderState state, OrderConfirmed _)
        => state with { Status = OrderStatus.Confirmed };

    static OrderState HandleCancelled(OrderState state, OrderCancelled _)
        => state with { Status = OrderStatus.Cancelled };
}
```

---

## 4. Aggregate

The aggregate holds business logic and enforces invariants. Call `Apply()` to record an event — this updates `State` and adds the event to pending `Changes`.

```csharp
// Order.cs
namespace Orders.Domain;

using Eventuous;
using static OrderEvents.V1;

public class Order : Aggregate<OrderState> {

    public void Place(string customerId, string[] itemSkus, decimal totalAmount, string currency) {
        EnsureDoesntExist();
        if (itemSkus.Length == 0)
            throw new DomainException("Order must contain at least one item");
        if (totalAmount <= 0)
            throw new DomainException("Total amount must be positive");

        Apply(new OrderPlaced(customerId, itemSkus, totalAmount, currency));
    }

    public void Confirm(string confirmedBy) {
        EnsureExists();
        if (State.Status != OrderStatus.Placed)
            throw new DomainException($"Cannot confirm an order in status {State.Status}");

        Apply(new OrderConfirmed(confirmedBy, DateTimeOffset.UtcNow));
    }

    public void Cancel(string reason) {
        EnsureExists();
        if (State.Status == OrderStatus.Cancelled)
            throw new DomainException("Order is already cancelled");
        if (State.Status == OrderStatus.Confirmed)
            throw new DomainException("Confirmed orders cannot be cancelled");

        Apply(new OrderCancelled(reason, DateTimeOffset.UtcNow));
    }
}
```

---

## 5. Commands

Commands are record types grouped in a static class. The `[HttpCommand]` attribute enables minimal API auto-discovery.

```csharp
// OrderCommands.cs
namespace Orders.Application;

using Eventuous.AspNetCore.Web;
using Orders.Domain;

[HttpCommands<OrderState>]
public static class OrderCommands {
    [HttpCommand(Route = "place")]
    public record PlaceOrder(
        string   OrderId,
        string   CustomerId,
        string[] ItemSkus,
        decimal  TotalAmount,
        string   Currency
    );

    [HttpCommand(Route = "confirm")]
    public record ConfirmOrder(
        string OrderId,
        string ConfirmedBy
    );

    [HttpCommand(Route = "cancel")]
    public record CancelOrder(
        string OrderId,
        string Reason
    );
}
```

---

## 6. Command Service

The aggregate-based command service loads the aggregate, routes commands to aggregate methods, and persists resulting events.

```csharp
// OrderService.cs
namespace Orders.Application;

using Eventuous;
using Orders.Domain;

public class OrderService : CommandService<Order, OrderState, OrderId> {

    public OrderService(IEventStore store) : base(store) {

        On<PlaceOrder>()
            .InState(ExpectedState.New)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Place(
                cmd.CustomerId,
                cmd.ItemSkus,
                cmd.TotalAmount,
                cmd.Currency
            ));

        On<ConfirmOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Confirm(cmd.ConfirmedBy));

        On<CancelOrder>()
            .InState(ExpectedState.Existing)
            .GetId(cmd => new OrderId(cmd.OrderId))
            .Act((order, cmd) => order.Cancel(cmd.Reason));
    }
}
```

---

## 7. DI Registration & HTTP API (Program.cs)

```csharp
// Program.cs
using Eventuous;
using Eventuous.AspNetCore;
using Eventuous.KurrentDB;
using Orders.Application;
using Orders.Domain;

var builder = WebApplication.CreateBuilder(args);

// 1. Register KurrentDB client
builder.Services.AddKurrentDBClient(
    builder.Configuration["KurrentDB:ConnectionString"]!
);

// 2. Register the event store
builder.Services.AddEventStore<KurrentDBEventStore>();

// 3. Register the command service (also registers ICommandService<OrderState>)
builder.Services.AddCommandService<OrderService, OrderState>();

// 4. Add controllers (needed for CommandHttpApiBase; skip if only using minimal API)
builder.Services.AddControllers();

var app = builder.Build();

// 5. Map auto-discovered HTTP commands (from [HttpCommands<OrderState>])
app.MapDiscoveredCommands<OrderState>();

// Optional: add Eventuous diagnostics endpoint
// app.MapEventuousSpyglass();

app.Run();
```

### appsettings.json

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

## 8. HTTP API Endpoints (Generated)

The `[HttpCommands<OrderState>]` and `[HttpCommand(Route = ...)]` attributes generate the following minimal API routes under the `/order` path (derived from `OrderState` → `Order`):

| Method | Path             | Body             |
|--------|------------------|------------------|
| POST   | /order/place     | `PlaceOrder`     |
| POST   | /order/confirm   | `ConfirmOrder`   |
| POST   | /order/cancel    | `CancelOrder`    |

### Example: Place an Order

```http
POST /order/place
Content-Type: application/json

{
  "orderId": "order-42",
  "customerId": "customer-7",
  "itemSkus": ["SKU-001", "SKU-002"],
  "totalAmount": 129.99,
  "currency": "USD"
}
```

### Successful Response

```json
{
  "globalPosition": 0,
  "changes": [
    {
      "eventType": "V1.OrderPlaced",
      "data": {
        "customerId": "customer-7",
        "itemSkus": ["SKU-001", "SKU-002"],
        "totalAmount": 129.99,
        "currency": "USD"
      }
    }
  ],
  "state": {
    "customerId": "customer-7",
    "itemSkus": ["SKU-001", "SKU-002"],
    "totalAmount": 129.99,
    "currency": "USD",
    "status": 1
  }
}
```

---

## Key Points

- **Event types are registered automatically** via source generation — no manual `TypeMap` calls needed.
- **`IAggregateStore` is deprecated** — `CommandService` uses `IEventStore` / `IEventReader` / `IEventWriter` internally.
- **Stream name** follows the default pattern: `Order-order-42`.
- **Optimistic concurrency** is handled automatically by `CommandService` using `OriginalVersion`.
- **`EnsureDoesntExist()`** guards against duplicate `PlaceOrder` commands (idempotency at the aggregate level).
- **`EnsureExists()`** ensures `ConfirmOrder` and `CancelOrder` only act on existing orders.
