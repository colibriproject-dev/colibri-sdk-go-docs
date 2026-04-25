---
sidebar_position: 5
title: Asynchronous Communication
---

This package implements an asynchronous messaging system with support for multiple providers (AWS SNS/SQS, Google Cloud Pub/Sub, and RabbitMQ), covering message production and consumption, observability, and error handling features.

> **Important**: Before using the asynchronous communication mechanism, all topic and queue structures must have been previously created in the chosen provider.

## Configuration

The system automatically detects the configured cloud provider via environment variables.

### AWS / GCP (Default)
By default, the `COLIBRI_MESSAGING` variable is set to `CLOUD_DEFAULT`.

### RabbitMQ
To use RabbitMQ, configure the following variables:
- `COLIBRI_MESSAGING`: Set to `RABBITMQ`.
- `RABBITMQ_URL`: Service access URL (e.g., `amqp://guest:guest@localhost:5672/`).

*Note: When using RabbitMQ, cloud services (SNS/SQS and Pub/Sub) are ignored.*

## Initialization

To enable messaging features, add initialization in the `main.go` function:

```go showLineNumbers
// Messaging system initialization
messaging.Initialize()
```

## Main Components

### 1. Publishers (Producers)

Used to send messages to a specific topic.

```go showLineNumbers
// Creating a producer
producer := messaging.NewProducer("TOPIC_NAME")

// Message publication
// The second parameter "action" helps identify the message's purpose
err := producer.Publish(ctx, "create", myMessage)
```

Features:
- Strong typing support.
- Automatic authentication context propagation.
- Message tracking via UUID.
- Integrated monitoring.

### 2. Consumers

To consume messages, implement the consumer interface and register it with the system.

```go showLineNumbers
// Consumer implementation
type MyConsumer struct{}

func (c *MyConsumer) QueueName() string {
    return "QUEUE_NAME"
}

func (c *MyConsumer) Consume(ctx context.Context, msg *ProviderMessage) error {
    var data MyStructure
    if err := msg.DecodeAndValidateMessage(&data); err != nil {
        return err
    }
    // Message processing logic
    return nil
}

// Consumer registration
messaging.NewConsumer(&MyConsumer{})
```

### 3. Message Structure (`ProviderMessage`)

Messages received by the consumer follow the structure below:

```go showLineNumbers
type ProviderMessage struct {
    ID          uuid.UUID
    Origin      string
    Action      string
    Message     any
    AuthContext *security.AuthenticationContext
}
```

## Advanced Features

### 1. Multi-Cloud Support
Complete abstraction for:
- **AWS**: SNS for topics and SQS for queues.
- **Google Cloud**: Pub/Sub for topics and subscriptions.
- **RabbitMQ**: Exchanges and Queues.

### 2. Observability and Resilience
- Native integration with [OpenTelemetry](https://opentelemetry.io/).
- Structured logging and message tracking.
- Dead Letter Queue (DLQ) support for failure handling.

## Usage Examples

### 1. Publishing a New User

```go showLineNumbers
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

func PublishNewUser(ctx context.Context, user User) error {
    producer := messaging.NewProducer("USERS_CREATED")
    return producer.Publish(ctx, "create", user)
}
```

### 2. Processing with Authentication Context

```go showLineNumbers
func (p *MyConsumer) Consume(ctx context.Context, msg *ProviderMessage) error {
    // The authentication context is automatically populated
    // if provided in the original message metadata.
    tenantID := msg.AuthContext.TenantID
    userID := msg.AuthContext.UserID
    
    // Processing with data isolation or specific permissions
    return nil
}
```

## Best Practices

1.  **Naming**: Use descriptive uppercase names with underscores (e.g., `ORDERS_PROCESSED`), preferably prefixed by the origin service name.
2.  **Idempotency**: Ensure message processing is idempotent to avoid side effects in case of reprocessing.
3.  **Validation**: Always use `msg.DecodeAndValidateMessage` to ensure the received payload is as expected.
4.  **Monitoring**: Track queue size and error rate to identify bottlenecks or consumption logic failures.

---
