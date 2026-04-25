---
sidebar_position: 4
title: Cache
---

The package offers a robust cache implementation using [Redis](https://redis.io), with support for generic types and configurable TTL (*Time To Live*). It is especially useful for temporarily storing frequently accessed data, reducing the load on other services like the database.

## Configuration

To use the cache, the following environment variables can be configured:

- `CACHE_URI`: Redis connection URI (e.g., `localhost:6379`).
- `CACHE_PASSWORD`: Redis connection password (optional).

## Initialization

To enable cache in your application, include the instruction below in the `main.go` function:

```go showLineNumbers
// Cache initialization
cacheDB.Initialize()
```

Features:
- Automatic connection to [Redis](https://redis.io).
- Integration with [OpenTelemetry](https://opentelemetry.io/) for monitoring.

## Main Components

### 1. Cache Creation

```go showLineNumbers
// Creating a new cache with TTL
cache := NewCache[MyStructure]("cache-name", time.Hour)
```

Parameters:
- `name`: Unique name to identify the cache.
- `ttl`: Time to live for data in the cache (time.Duration).

### 2. Core Operations

#### Storage (Set)

```go showLineNumbers
// Storing a single item
user := User{Id: 1, Name: "John"}
err := cache.Set(ctx, user)

// Storing multiple items
users := []User{
    {Id: 1, Name: "John"},
    {Id: 2, Name: "Mary"},
}
err := cache.Set(ctx, users)
```

#### Retrieval (Get)

```go showLineNumbers
// Retrieving a single item
user, err := cache.One(ctx)

// Retrieving multiple items
users, err := cache.Many(ctx)
```

#### Deletion (Del)

```go showLineNumbers
// Removing data from the cache
err := cache.Del(ctx)
```

## Advanced Features

### 1. Generic Typing

The cache uses Go generic types, allowing for strong typing:

```go showLineNumbers
// Structure definition
type User struct {
    Id   int
    Name string
}

// Typed cache for User
userCache := NewCache[User]("users", time.Hour)

// Typed cache for another type
orderCache := NewCache[Order]("orders", 30 * time.Minute)
```

### 2. Observability

- Monitoring via [OpenTelemetry](https://opentelemetry.io/).
- Structured logging of operations.
- Error tracking.

### 3. Connection Management

- Support for [Redis](https://redis.io) cluster.
- Safe closing of connections.

## Usage Examples

### 1. Cache with Database

The database module already has transparent integration with the cache.

```go showLineNumbers
func FetchUser(ctx context.Context, id int) (*User, error) {
    // Create cache with 1 hour duration
    cache := cacheDB.NewCache[User]("users_cache_key", time.Hour)

    // When executing the query, the internal mechanism of the 'NewCachedQuery' method performs the following steps:
    //  1 - Validates if a record exists in the cache for the given key.
    //  2 - If the record exists, it is returned.
    //  3 - If no record is found in the cache, the query is executed in the database.
    //  4 - The query result is saved in the cache.
    //  5 - The query result is returned.
    return sqlDB.NewCachedQuery(ctx, cache,
      "SELECT * FROM users WHERE id = $1", 1).One()
}
```

### 2. List Cache

```go showLineNumbers
func ListProducts(ctx context.Context) ([]Product, error) {
    cache := NewCache[Product]("products", 15 * time.Minute)
    
    // Try to retrieve list from cache
    products, err := cache.Many(ctx)
    if err == nil && len(products) > 0 {
        return products, nil
    }

    // Retrieve product list from another application layer
    products, err = products.ListProducts()
    if err != nil {
        return nil, err
    }

    // Update cache
    if err := cache.Set(ctx, products); err != nil {
        log.Warn("failed to update cache", err)
    }

    return products, nil
}
```

### 3. Cache Invalidation

```go showLineNumbers
func UpdateProduct(ctx context.Context, product Product) error {
    // Update in database
    if err := db.UpdateProduct(product); err != nil {
        return err
    }
    
    // Invalidate cache
    cache := NewCache[Product]("products", time.Hour)
    if err := cache.Del(ctx); err != nil {
        log.Warn("failed to invalidate cache", err)
    }
    
    return nil
}
```

### 4. Cache with Application Prefix

The cache automatically uses the application name as a prefix:

```go
cache := NewCache[User]("users", time.Hour)
// Internally stores as "MyApp::users"
// `MyApp` is the value defined in the 'APP_NAME' environment variable
```

This design allows:
- Isolation between different applications.
- Avoids naming conflicts.
- Facilitates debugging.

## Best Practices

1. **Define appropriate TTLs**:
    - Frequently updated data: Short TTL.
    - Static data: Long TTL.
    - Avoid zero TTL (no expiration).

2. **Handle failures gracefully**:
    - Always have a fallback when the cache fails.
    - Do not block critical operations due to cache failures.

3. **Monitor usage**:
    - Configure alerts for connection failures.
    - Monitor hit/miss ratio.
    - Observe memory usage.

4. **Maintain consistency**:
    - Invalidate related caches together.
    - Use consistent cache names.
    - Document the TTLs used.

---
