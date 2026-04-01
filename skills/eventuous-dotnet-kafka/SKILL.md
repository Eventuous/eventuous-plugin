---
name: eventuous-dotnet-kafka
description: "This skill should be used when configuring Kafka integration with Eventuous. Covers KafkaBasicProducer for publishing to Kafka topics, Confluent.Kafka configuration, partition keys, message headers, and produce options. Common triggers: 'publish events to Kafka', 'Kafka producer', 'Kafka topic', 'KafkaBasicProducer', 'Confluent.Kafka with Eventuous'."
---
# Eventuous Kafka Integration

NuGet package: `Eventuous.Kafka`
Namespace: `Eventuous.Kafka.Producers`, `Eventuous.Kafka.Subscriptions`
Source: `src/Kafka/src/Eventuous.Kafka/`

Uses Confluent.Kafka under the hood. Produces byte[] payloads with type info in Kafka headers (no schema registry).

## Producer

`KafkaBasicProducer` extends `BaseProducer<KafkaProduceOptions>` and implements `IHostedProducer`, `IAsyncDisposable`.

```csharp
// Constructor takes KafkaProducerOptions (wraps Confluent ProducerConfig)
public record KafkaProducerOptions(ProducerConfig ProducerConfig);

// Per-message produce options (partition key)
public record KafkaProduceOptions(string PartitionKey);
```

### Instantiation

```csharp
var options = new KafkaProducerOptions(new ProducerConfig {
    BootstrapServers = "localhost:9092"
});
await using var producer = new KafkaBasicProducer(options);
await producer.StartAsync(cancellationToken);

// Produce with partition key
await producer.Produce(
    new StreamName("my-topic"),
    events,
    new Metadata(),
    new KafkaProduceOptions("my-partition-key"),
    cancellationToken: ct
);
```

### DI Registration

```csharp
services.AddProducer<KafkaBasicProducer>(sp =>
    new KafkaBasicProducer(
        new KafkaProducerOptions(new ProducerConfig {
            BootstrapServers = "localhost:9092"
        })
    )
);
```

### How it works

- Serializes events using `IEventSerializer`, sends as `byte[]` values
- Stores event type in `message-type` header and content type in `content-type` header (configurable via `KafkaHeaderKeys`)
- When `PartitionKey` is provided, uses a keyed producer (`IProducer<string, byte[]>`); otherwise uses `IProducer<Null, byte[]>`
- Metadata entries are converted to Kafka headers via `MetadataExtensions.AsKafkaHeaders()`
- Supports delivery acknowledgement callbacks (`OnAck`/`OnFail`)
- `StopAsync` flushes pending messages before stopping

### Header Keys

```csharp
public static class KafkaHeaderKeys {
    public static string MessageTypeHeader { get; set; } = "message-type";
    public static string ContentTypeHeader { get; set; } = "content-type";
}
```

## Gateway Integration

To forward events from an event store subscription to Kafka topics, use the Eventuous Gateway. The stream name becomes the Kafka topic name. See the `eventuous-dotnet-gateway` skill for full gateway configuration details.

```csharp
builder.Services.AddGateway<KurrentDBAllStreamSubscription, KafkaProduceOptions>(
    "events-to-kafka",
    GatewaySubscription<KurrentDBAllStreamSubscription>.Create("kafka-gateway-sub"),
    RouteAndTransform.Empty
);
```

## Key Behaviors

- **Topic mapping**: The stream name passed to `Produce` becomes the Kafka topic name
- **Partition key**: When using `KafkaProduceOptions` with a `Key`, messages with the same key go to the same partition (ordering guarantee)
- **Headers**: Each message includes `message-type` and `content-type` headers, plus all metadata entries as additional headers
- **Flush on stop**: The producer flushes pending messages when the hosted service stops
- **Keyed vs unkeyed**: `KafkaBasicProducer` has two internal producers — one for keyed messages (with partition key) and one for unkeyed (round-robin)

## Subscription

`KafkaBasicSubscription` extends `EventSubscription<KafkaSubscriptionOptions>`. Note: the subscription is currently a stub (throws `NotImplementedException`).

```csharp
public record KafkaSubscriptionOptions : SubscriptionOptions {
    public ConsumerConfig ConsumerConfig { get; init; } = null!;
}
```

> **Note:** The Kafka subscription is not yet implemented. For consuming Kafka messages in Eventuous event handlers, use `Confluent.Kafka`'s `ConsumerBuilder` directly and route messages through your application layer. This limitation is tracked for a future release.

## Tracing

Built-in OpenTelemetry tracing with:
- `MessagingSystem = "kafka"`
- `DestinationKind = "topic"`
- `ProduceOperation = "produce"`
