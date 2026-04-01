---
name: eventuous-dotnet-sqlserver
description: "This skill should be used when configuring SQL Server (MSSQL) integration with Eventuous. Covers SqlServerStore event store, SqlServerAllStreamSubscription and SqlServerStreamSubscription, SqlServerCheckpointStore, and SqlServerProjector for read model projections. Common triggers: 'SQL Server event store', 'MSSQL with Eventuous', 'SqlServerStore', 'AddEventuousSqlServer', 'SQL Server projections'."
---
# Eventuous SQL Server Integration

NuGet package: `Eventuous.SqlServer`
Namespace: `Eventuous.SqlServer`
Source: `src/SqlServer/src/Eventuous.SqlServer/`

Provides event store, subscriptions, checkpoint store, and projections for SQL Server.

## Setup

### AddEventuousSqlServer

Register the SQL Server event store and schema initializer:

```csharp
// Option 1: Connection string directly
services.AddEventuousSqlServer(
    connectionString: "Server=localhost;Database=mydb;...",
    schema: "eventuous",          // default: "eventuous"
    initializeDatabase: true      // creates schema on startup
);

// Option 2: From IConfiguration (recommended)
services.AddEventuousSqlServer(configuration.GetSection("SqlServer"));
```

The configuration section binds to `SqlServerStoreOptions`. Store the connection string in user secrets, environment variables, or a vault — not in `appsettings.json`:

```json
{
  "SqlServer": {
    "ConnectionString": "Server=localhost;Database=eventuous;User Id=sa;Password=...;TrustServerCertificate=true",
    "Schema": "eventuous"
  }
}
```

This registers:
- `SqlServerStoreOptions` as singleton
- `SqlServerStore` as singleton (the event store)
- `SchemaInitializer` as hosted service (creates tables/stored procs if `InitializeDatabase = true`)
- `SqlServerConnectionOptions` for shared connection info

### SqlServerStoreOptions

```csharp
public record SqlServerStoreOptions {
    public string? ConnectionString   { get; init; }
    public string  Schema             { get; init; } = "eventuous";
    public bool    InitializeDatabase { get; init; }
}
```

## Event Store

`SqlServerStore` extends `SqlEventStoreBase<SqlConnection, SqlTransaction>` and implements `IEventStore`. Default schema is `"eventuous"`.

```csharp
services.AddEventuousSqlServer(connectionString, initializeDatabase: true);
services.AddEventStore<SqlServerStore>();
```

## Schema

The `Schema` class defines stored procedure names and SQL queries scoped to the configured schema name:
- `append_events`, `read_stream_forwards`, `read_stream_backwards`
- `read_all_forwards`, `check_stream`, `truncate_stream`
- Checkpoint queries for the `Checkpoints` table

`SchemaInitializer` is an `IHostedService` that runs embedded SQL scripts to create the schema. It retries up to 10 times with 5-second delays on `SqlException`.

## Subscriptions

### SqlServerAllStreamSubscription

Subscribes to all events across all streams (uses `read_all_forwards` stored procedure).

```csharp
services.AddSubscription<SqlServerAllStreamSubscription, SqlServerAllStreamSubscriptionOptions>(
    "MyAllStreamSub",
    builder => builder
        .Configure(o => {
            o.Schema = "eventuous";
            o.ConnectionString = connectionString;  // optional if AddEventuousSqlServer was called
        })
        .AddEventHandler<MyHandler>()
);
```

### SqlServerStreamSubscription

Subscribes to events in a single named stream.

```csharp
services.AddSubscription<SqlServerStreamSubscription, SqlServerStreamSubscriptionOptions>(
    "MyStreamSub",
    builder => builder
        .Configure(o => {
            o.Stream = new StreamName("MyStream-123");
            o.ConnectionString = connectionString;
        })
        .AddEventHandler<MyHandler>()
);
```

### Options hierarchy

```
SqlSubscriptionOptionsBase          (Schema, MaxPageSize, PollingInterval)
  -> SqlServerSubscriptionBaseOptions  (+ ConnectionString)
       -> SqlServerAllStreamSubscriptionOptions
       -> SqlServerStreamSubscriptionOptions   (+ Stream)
```

Connection string and schema can come from either the subscription options or `SqlServerConnectionOptions` (registered by `AddEventuousSqlServer`).

### Common Subscription Options (SqlSubscriptionOptionsBase)

| Property | Type | Default | Description |
|---|---|---|---|
| `Schema` | string | `"eventuous"` | Database schema name |
| `ConcurrencyLimit` | int | 1 | Number of concurrent message consumers |
| `MaxPageSize` | int | 1024 | Messages fetched per poll |
| `Polling` | PollingOptions | see below | Polling interval configuration |
| `Retry` | RetryOptions | see below | Retry configuration |
| `GapAgeThresholdMs` | int? | 3600000 (1h) | Gaps older than this are skipped |
| `GapSkipTimeoutMs` | int? | 5000 | Max time a gap holds back the subscription |
| `GapHandlingTimeoutMs` | int? | null | When set, creates tombstones for persistent gaps |

**PollingOptions**: `MinIntervalMs` (5), `MaxIntervalMs` (1000), `GrowFactor` (1.5)
**RetryOptions**: `InitialDelayMs` (50)

Configure options inline:

```csharp
services.AddSubscription<SqlServerAllStreamSubscription, SqlServerAllStreamSubscriptionOptions>(
    "FastSub",
    builder => builder
        .Configure(o => {
            o.ConcurrencyLimit = 4;
            o.MaxPageSize = 512;
            o.Polling = new() { MinIntervalMs = 10, MaxIntervalMs = 500 };
        })
        .AddEventHandler<MyHandler>()
);
```

## Checkpoint Store

`SqlServerCheckpointStore` implements `ICheckpointStore`. Stores checkpoints in `{schema}.Checkpoints` table.

```csharp
services.AddSqlServerCheckpointStore();
```

This uses the connection string and schema from `SqlServerConnectionOptions` (falls back to `SqlServerCheckpointStoreOptions` if configured separately).

```csharp
public record SqlServerCheckpointStoreOptions {
    public string? Schema           { get; init; }
    public string? ConnectionString { get; init; }
}
```

## Projections

`SqlServerProjector` is the base class for SQL Server read model projections.

```csharp
public class MyProjection : SqlServerProjector {
    public MyProjection(SqlServerConnectionOptions options) : base(options) {
        On<MyEvent>((connection, ctx) =>
            Project(connection,
                "INSERT INTO MyReadModel (Id, Name) VALUES (@id, @name)",
                new SqlParameter("@id", ctx.Message.Id),
                new SqlParameter("@name", ctx.Message.Name)
            )
        );
    }
}
```

Register with a subscription:
```csharp
services.AddSubscription<SqlServerAllStreamSubscription, SqlServerAllStreamSubscriptionOptions>(
    "MyProjectionSub",
    builder => builder
        .AddEventHandler<MyProjection>()
);
```

## Complete Example

```csharp
services.AddEventuousSqlServer(connectionString, initializeDatabase: true);
services.AddEventStore<SqlServerStore>();
services.AddSqlServerCheckpointStore();

services.AddSubscription<SqlServerAllStreamSubscription, SqlServerAllStreamSubscriptionOptions>(
    "AllEvents",
    builder => builder
        .AddEventHandler<MyHandler>()
);
```

## Source Files

Paths relative to the Eventuous repository root:

- Store: `src/SqlServer/src/Eventuous.SqlServer/SqlServerStore.cs`
- Registration: `src/SqlServer/src/Eventuous.SqlServer/Extensions/RegistrationExtensions.cs`
- Subscriptions: `src/SqlServer/src/Eventuous.SqlServer/Subscriptions/`
- Projections: `src/SqlServer/src/Eventuous.SqlServer/Projections/`
- Checkpoint: `src/SqlServer/src/Eventuous.SqlServer/Subscriptions/SqlServerCheckpointStore.cs`
