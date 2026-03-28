---
sidebar_position: 2
title: HTTP Client
---

The `RestClient` provides a robust and simplified interface for making HTTP requests in Go. It encapsulates essential features for modern microservices, such as *circuit breaker*, retry strategies, cache support, and handling of different content types, including `multipart/form-data`.

## Main Structure

Below is the definition of the `RestClient` structure:

```go showLineNumbers
type RestClient struct {
    name       string                         // Client identifier name
    baseURL    string                         // Base URL for all requests
    retries    uint8                          // Maximum number of retries
    retrySleep uint                           // Wait time between retries (seconds)
    client     *http.Client                   // Native Go HTTP client
    cb         *circuitbreaker.CircuitBreaker // Circuit Breaker instance
}
```

## Configuration and Installation

To create an instance of `RestClient`, use the `NewRestClient` function with a configuration structure:

```go showLineNumbers
config := &RestClientConfig{
    Name:                "my-service-client",
    BaseURL:             "http://api.example.com",
    Timeout:             30,
    Retries:             3,
    RetrySleepInSeconds: 2,
    ProxyURL:            "http://proxy.example.com:8080", // Optional
}

client := NewRestClient(config)
```

## Core Features

### 1. Standardized Requests

The client uses generic types to facilitate automatic mapping of success and error responses.

```go showLineNumbers
// GET request example
response := Request[MyResponse, ErrorResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/users/1",
}.Call()

// POST request example with body (JSON)
newUser := User{Name: "John", Email: "john@example.com"}
response := Request[UserResponse, ErrorResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/users",
    Body:       &newUser,
}.Call()
```

### 2. File Upload (`Multipart`)

Native support for sending files and form fields:

```go showLineNumbers
file := bytes.NewBufferString("file content")
response := Request[UploadResponse, UploadError]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/upload",
    MultipartFields: map[string]any{
        "file": MultipartFile{
            FileName:    "document.txt",
            File:        file,
            ContentType: "text/plain",
        },
        "category": "personal_documents",
    },
}.Call()
```

### 3. Cache Integration

It is possible to enable [cache](./cache.md) directly in the request definition to avoid repetitive calls.

```go showLineNumbers
response := Request[UserData, ErrorResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/users/1",
    Cache:      cacheDB.NewCache[UserData]("user_1_key", 10 * time.Minute),
}.Call()
```

### 4. Response Handling

The `ResponseData` structure simplifies checking the operation result:

```go showLineNumbers
if response.HasSuccess() {
    data := response.SuccessBody()
    // Success logic (Status 2xx)
} else if response.HasError() {
    err := response.ErrorBody()
    // Mapped error logic
}

// Status Code verification helpers
if response.IsSuccessfulResponse() {
    // 2xx
} else if response.IsClientErrorResponse() {
    // 4xx
} else if response.IsServerErrorResponse() {
    // 5xx
}
```

## Resilience and Fault Tolerance

### *Circuit Breaker*
The client monitors the health of the target service. By default:
- The circuit **opens** after 5 consecutive failures.
- It remains open for 10 seconds before attempting a new connection (*half-open*).

### *Retry*
Allows configuring behavior in case of temporary network failures or momentary unavailability, defining the number of attempts and the interval between them.

---
