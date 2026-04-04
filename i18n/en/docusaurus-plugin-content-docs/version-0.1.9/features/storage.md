---
sidebar_position: 6
title: Storage
---

The *Storage* package provides an abstraction layer for cloud storage operations, with native support for AWS S3 and Google Cloud Storage (GCS). It simplifies file manipulation, offering a unified interface and integrated monitoring.

## Configuration

The system automatically detects the configured cloud provider (AWS or GCP) via each provider's standard environment variables. Ensure that appropriate credentials and permissions are configured in the execution environment.

## Initialization

To enable storage features, include the initialization in your `main.go` file:

```go showLineNumbers
// Storage initialization
storage.Initialize()
```

## Core Operations

### 1. File Upload

Performs the upload of a file to the specified *bucket*.

```go showLineNumbers
location, err := storage.UploadFile(
    ctx,
    "bucket-name",
    "destination/path/file.txt",
    fileReader, // io.Reader
)
```

### 2. File Download

Retrieves a file from cloud storage.

```go showLineNumbers
file, err := storage.DownloadFile(
    ctx,
    "bucket-name",
    "path/to/file.txt",
)
// Remember to close the file after use
defer file.Close()
```

### 3. File Removal

Permanently deletes a file from the *bucket*.

```go showLineNumbers
err := storage.DeleteFile(
    ctx,
    "bucket-name",
    "path/to/file.txt",
)
```

## Features and Integrations

### Multi-Cloud Abstraction
- **AWS S3**: Uses the official AWS SDK.
- **Google Cloud Storage**: Uses standard GCP libraries.

### Observability
- **OpenTelemetry**: Automatic tracking of *upload*, *download*, and *delete* operations.
- **Logging**: Structured logs to facilitate debugging of permission or connectivity failures.

## Usage Examples

### In-Memory File Processing

```go showLineNumbers
func ProcessReport(ctx context.Context, bucket, key string) error {
    file, err := storage.DownloadFile(ctx, bucket, key)
    if err != nil {
        return fmt.Errorf("download failure: %w", err)
    }
    defer file.Close()

    data, err := io.ReadAll(file)
    if err != nil {
        return fmt.Errorf("read error: %w", err)
    }

    // Processing logic...
    return nil
}
```

## Best Practices

1.  **Resource Management**: Always use `defer file.Close()` when downloading to avoid memory leaks and open connections.
2.  **Security**: Never expose access keys directly in the code; use environment variables or secret managers.
3.  **Idempotency**: Ensure the system correctly handles upload attempts for files that already exist, if necessary.
4.  **Streaming**: For large files, prefer working with streams (`io.Reader`/`io.Writer`) instead of loading the entire content into memory.

---
