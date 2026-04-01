# Eventuous Plugin for Claude Code

A Claude Code plugin providing skills and an expert agent for building event-sourced applications with Eventuous in **.NET** and **Go**. This plugin integrates comprehensive support for Eventuous domain modeling, event store implementations, subscriptions, producers, and cross-context event routing directly into Claude Code.

## Installation

Add the marketplace and install the plugin with these commands:

```bash
# Add the marketplace (one-time)
/plugin marketplace add Eventuous/eventuous-plugin

# Install the plugin
/plugin install eventuous
```

## Skills

The plugin provides the following skills for Eventuous development:

### .NET Skills

| Skill | Description |
|---|---|
| `eventuous-dotnet` | Core library — domain modeling, command services, event stores, subscriptions, producers |
| `eventuous-dotnet-kurrentdb` | KurrentDB (EventStoreDB) event store, subscriptions, and producer |
| `eventuous-dotnet-postgres` | PostgreSQL event store, subscriptions, checkpoint storage, and projections |
| `eventuous-dotnet-mongodb` | MongoDB checkpoint storage and projections |
| `eventuous-dotnet-rabbitmq` | RabbitMQ producer, subscriptions, and configuration |
| `eventuous-dotnet-kafka` | Kafka producer and subscription |
| `eventuous-dotnet-sqlserver` | SQL Server event store, subscriptions, and checkpoint storage |
| `eventuous-dotnet-google-pubsub` | Google Cloud Pub/Sub producer and subscription |
| `eventuous-dotnet-azure-servicebus` | Azure Service Bus producer and subscription |
| `eventuous-dotnet-gateway` | Cross-context event routing with Gateway |

### Go Skills

| Skill | Description |
|---|---|
| `eventuous-go` | Core library — state/fold, aggregates, commands, subscriptions, codec |
| `eventuous-go-kurrentdb` | KurrentDB event store, catch-up and persistent subscriptions |
| `eventuous-go-otel` | OpenTelemetry tracing and metrics |

## Agent

The plugin includes an `eventuous-expert` agent that activates for cross-cutting Eventuous tasks spanning multiple integrations in both .NET and Go. This agent helps coordinate complex scenarios that involve multiple event store implementations, producers, subscriptions, and architectural decisions.

## How It Works

Skills activate automatically based on what you're working on. When Claude Code detects you're working with Eventuous code, the relevant skills provide contextual guidance, documentation, and implementation patterns tailored to your use case.

## Links

- **Eventuous Documentation**: https://eventuous.dev
- **GitHub Repository**: https://github.com/Eventuous/eventuous
