# Eventuous Plugin for Claude Code

A Claude Code plugin providing skills and an expert agent for building event-sourced .NET applications with Eventuous. This plugin integrates comprehensive support for Eventuous domain modeling, event store implementations, subscriptions, producers, and cross-context event routing directly into Claude Code.

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

| Skill | Description |
|---|---|
| `eventuous` | Core library — domain modeling, command services, event stores, subscriptions, producers |
| `eventuous-kurrentdb` | KurrentDB (EventStoreDB) event store, subscriptions, and producer |
| `eventuous-postgres` | PostgreSQL event store, subscriptions, checkpoint storage, and projections |
| `eventuous-mongodb` | MongoDB checkpoint storage and projections |
| `eventuous-rabbitmq` | RabbitMQ producer, subscriptions, and configuration |
| `eventuous-kafka` | Kafka producer and subscription |
| `eventuous-sqlserver` | SQL Server event store, subscriptions, and checkpoint storage |
| `eventuous-google-pubsub` | Google Cloud Pub/Sub producer and subscription |
| `eventuous-azure-servicebus` | Azure Service Bus producer and subscription |
| `eventuous-gateway` | Cross-context event routing with Gateway |

## Agent

The plugin includes an `eventuous-expert` agent that activates for cross-cutting Eventuous tasks spanning multiple integrations. This agent helps coordinate complex scenarios that involve multiple event store implementations, producers, subscriptions, and architectural decisions.

## How It Works

Skills activate automatically based on what you're working on. When Claude Code detects you're working with Eventuous code, the relevant skills provide contextual guidance, documentation, and implementation patterns tailored to your use case.

## Links

- **Eventuous Documentation**: https://eventuous.dev
- **GitHub Repository**: https://github.com/Eventuous/eventuous
