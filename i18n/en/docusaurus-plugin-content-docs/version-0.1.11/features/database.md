---
sidebar_position: 3
title: Database
---

This package provides a robust abstraction layer for SQL database operations, specially optimized for PostgreSQL. It includes features such as transaction management, migrations, paginated queries, caching, and reflection support for object-relational mapping.

## Main Components

### 1. Connection Management

To enable the database in your application, use the initialization in the `main.go` file:

```go showLineNumbers
// Database initialization
sqlDB.Initialize()
```

Features:
*   Automatic connection pool management.
*   Maximum connection configuration.
*   Native integration with the observability system.
*   Graceful shutdown of connections.

Available environment variables:
- `SQL_DB_HOST`: Database host.
- `SQL_DB_PORT`: Database port.
- `SQL_DB_NAME`: Database name.
- `SQL_DB_USER`: Database user.
- `SQL_DB_PASSWORD`: Connection password.
- `SQL_DB_MAX_OPEN_CONNS`: Sets the maximum number of open connections in the pool. *Default*: 10.
- `SQL_DB_MAX_IDLE_CONNS`: Sets the number of idle connections in the pool. *Default*: 3.
- `SQL_DB_SSL_MODE`: Database SSL mode (optional).
- `SQL_DB_MIGRATION`: Enables automatic migration execution (optional).
- `MIGRATION_SOURCE_URL`: Path to the folder where migrations are located (optional).

### 2. Queries

#### Simple Query

Use `NewQuery` to perform raw SQL queries mapped to Go structs.

```go showLineNumbers
// Query returning a single record
user, err := NewQuery[User](ctx, "SELECT id, name FROM users WHERE id = $1", 1).One()

// Query returning multiple records
users, err := NewQuery[User](ctx, "SELECT id, name FROM users WHERE active = true").Many()
```

#### Query with Cache

Sometimes we want to avoid repeated queries to the database. For this, we can use caching to avoid performance bottlenecks.

```go showLineNumbers
cache := cacheDB.NewCache[User]("users_cache_key", time.Hour)
user, err := NewCachedQuery(ctx, cache, 
    "SELECT * FROM users WHERE id = $1", 1).One()
```

### 3. Paginated Queries

A best practice for queries with multiple records is to paginate the results. This avoids overloading the server and the network.

```go showLineNumbers
pageNumber := 1
pageSize := 10
page := types.NewPageRequest(pageNumber, pageSize, []types.Sort{
    {Direction: types.ASC, Field: "name"},
    {Direction: types.DESC, Field: "birth_date"},
})

result, err := NewPageQuery[User](ctx, page, 
    "SELECT * FROM users").Execute()

// result.Content - List of users
// result.TotalElements - Total records
```

### 4. Statements (Insert/Update/Delete)

```go showLineNumbers
err := NewStatement(ctx, 
    "INSERT INTO users (name, email) VALUES ($1, $2)",
    "John Doe", "john@email.com").Execute()
```

### 5. Transactions

Often, we need to perform multiple changes within the same transactional block. For this, we can use the `Transaction` feature.
For more advanced definitions, see the detailed [transactions](./transactions.md) documentation.

```go showLineNumbers
tx := NewTransaction(sql.LevelSerializable)
 
err := tx.Execute(ctx, func(ctx context.Context) error {
    // Insert user
    err := NewStatement(ctx, 
        "INSERT INTO users (name) VALUES ($1)", 
        "John").Execute()
    if err != nil {
        return err // Causes rollback
    }

    // Insert profile
    err = NewStatement(ctx, 
        "INSERT INTO profiles (user_id, type) VALUES ($1, $2)",
        1, "ADMIN").Execute()
    if err != nil {
        return err // Causes rollback
    }

    return nil // Transaction commit
})
```

### 6. Migrations

The system supports automatic database migrations using SQL files.

Configuration:
```env showLineNumbers
SQL_DB_MIGRATION=true
MIGRATION_SOURCE_URL=./migrations
```

File structure:
``` 
migrations/
  ├── 000001_create_users.up.sql
  ├── 000001_create_users.down.sql
  ├── 000002_add_email.up.sql
  └── 000002_add_email.down.sql
```

## Advanced Features

### 1. Object-Relational Mapping

The package uses reflection to automatically map SQL results to Go structures:

```go showLineNumbers
type Profile struct {
    Id   int
    Name string
}

type User struct {
    Id       int
    Name     string
    Birthday time.Time
    Profile  Profile  // Support for nested structures
}

// The query will automatically map to the structure
user, err := NewQuery[User](ctx, 
    "SELECT u.id, u.name, u.birthday, p.id, p.name FROM users u JOIN profiles p ON u.profile_id = p.id WHERE u.id = $1",
    1,
).One()
```

### 2. Observability
- Integration with [OpenTelemetry](https://opentelemetry.io/) for monitoring
- Structured logging of operations
- Transaction tracking

## Usage Examples

### 1. Basic CRUD

```go showLineNumbers
// Create
err := NewStatement(ctx, 
    "INSERT INTO users (name, email) VALUES ($1, $2)",
    "Mary", "mary@email.com").Execute()

// Read
user, err := NewQuery[User](ctx, 
    "SELECT * FROM users WHERE email = $1",
    "mary@email.com").One()

// Update
err = NewStatement(ctx, 
    "UPDATE users SET name = $1 WHERE email = $2",
    "Mary Smith", "mary@email.com").Execute()

// Delete
err = NewStatement(ctx, 
    "DELETE FROM users WHERE email = $1",
    "mary@email.com").Execute()
```

### 2. Complex Query with Join

```go showLineNumbers
type QueryResult struct {
    UserName string
    TotalOrders int
    LastOrder time.Time
}

result, err := NewQuery[QueryResult](ctx, `
    SELECT 
        u.name as user_name,
        COUNT(o.id) as total_orders,
        MAX(o.date) as last_order
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    GROUP BY u.id
`).Many()
```

### 3. Pagination with Filters

```go showLineNumbers
pageNumber := 1
pageSize := 10
page := types.NewPageRequest(pageNumber, pageSize, []types.Sort{{
    Direction: types.DESC,
    Field: "created_at"
}})

results, err := NewPageQuery[Order](
    ctx,
    page,
    `SELECT * FROM orders WHERE status = $1 
     AND created_at >= $2`,
    "PENDING",
    time.Now().AddDate(0, -1, 0),
).Execute()
```

---
