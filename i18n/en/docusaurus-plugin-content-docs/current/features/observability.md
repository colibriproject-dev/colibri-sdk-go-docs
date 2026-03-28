---
sidebar_position: 7
title: Observability
---

The package provides a complete observability layer for applications, integrating metrics, tracing, and logs. The implementation encapsulates [OpenTelemetry](https://opentelemetry.io/), offering features for monitoring transactions, segments, and errors.

## Configuration

Configuration is automatically performed when starting `colibri-sdk-go`. To export data to an OpenTelemetry collector, use the following environment variable:

- `OTEL_EXPORTER_OTLP_ENDPOINT`: OTLP collector endpoint (e.g., `http://localhost:4318`).

## Initialization

When initializing Colibri, the following components are configured:
- [OpenTelemetry](https://opentelemetry.io/) connection.
- Default application metrics.

## Main Components

### 1. Monitoring Transactions

```go showLineNumbers
// Starting a new monitoring transaction
txn, ctx := monitoring.StartTransaction(ctx, "MyOperation")
defer monitoring.EndTransaction(txn)
```

### 3. Segments

```go
// Creating a transaction segment
segment := monitoring.StartTransactionSegment(ctx, "MyOperation", map[string]string{
    "operation": "query",
    "type": "database",
})
defer monitoring.EndTransactionSegment(segment)
```

> Note: For the segment to be correctly related to the transaction, we must pass the `ctx` attribute created in the main transaction.

### 4. Error Tracking

```go showLineNumbers
// Notifying errors
if err != nil {
    monitoring.NoticeError(txn, err)
    return err
}
```

## Usage Examples

### 1. External Service Monitoring

```go showLineNumbers
func (s *Service) CallExternalAPI(ctx context.Context) error {
    segment := monitoring.StartTransactionSegment(ctx, "Payment_API")
    defer monitoring.EndTransactionSegment(segment)
    
    resp, err := http.Get("https://api.external.com/resource")
    if err != nil {
        monitoring.NoticeError(monitoring.GetTransactionInContext(ctx), err)
        return err
    }
    defer resp.Body.Close()

    return nil
}
```

## Best Practices

1. **Transaction Naming**:
    - Use descriptive and consistent names.
    - Follow a naming standard.
    - Avoid dynamic names.

2. **Context Management**:
    - Always propagate the context.
    - Use defer to end transactions.
    - Maintain the correct lifecycle.

3. **Attributes**:
    - Add relevant attributes.
    - Do not add sensitive data.
    - Use consistent keys.

4. **Errors**:
    - Notify relevant errors.
    - Add context to errors.
    - Avoid notifying expected errors.

5. **Performance**:
    - Monitor response times.
    - Observe resource usage.
    - Identify bottlenecks.

6. **Security**:
    - Do not log sensitive data.
    - Respect data protection laws.

---
