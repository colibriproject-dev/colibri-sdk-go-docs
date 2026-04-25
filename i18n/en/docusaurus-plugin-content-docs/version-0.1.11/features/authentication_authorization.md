---
sidebar_position: 8
title: Authentication and Authorization
---

The authentication context package provides a simple and powerful structure for managing authenticated user information throughout the application. Using Go's native context mechanism, it allows storing and retrieving user details securely, making it ideal for multi-tenant applications.

## Main Components

### 1. `AuthenticationContext`

The core structure that holds authentication data:

```go showLineNumbers
type AuthenticationContext struct {
    TenantID string `json:"tenantId"`
    UserID   string `json:"userId"`
}
```

### 2. `IAuthenticationContext` Interface

Defines the contract for standardized access to information:

```go showLineNumbers
type IAuthenticationContext interface {
    GetUserID() string
    GetTenantID() string
}
```

### 3. `User` Structure

Represents the complete user profile in the system:

```go showLineNumbers
type User struct {
    ID       string
    Email    string
    Phone    string
    Name     string
    TenantID string
    Profile  string
    Profiles []string
    PhotoURL string
}
```

## Core Features

### 1. Context Creation

```go showLineNumbers
authContext := security.NewAuthenticationContext("tenant123", "user456")
```

### 2. Storing in Go Context

Add authentication information to the request context:

```go showLineNumbers
ctx = authContext.SetInContext(context.Background())
```

### 3. Data Retrieval

Retrieve information from anywhere that has access to the context:

```go showLineNumbers
authContext := security.GetAuthenticationContext(ctx)
if authContext != nil {
    tenantID := authContext.GetTenantID()
    userID := authContext.GetUserID()
}
```

### 4. Validation

Check if the context has valid data:

```go showLineNumbers
if authContext.Valid() {
    // Proceed with the operation
}
```

## Usage Examples

### 1. Multi-tenant Data Access

Ensure data isolation using the `TenantID` retrieved from the context.

```go showLineNumbers
func (r *Repository) FetchData(ctx context.Context) (*Data, error) {
    authContext := security.GetAuthenticationContext(ctx)
    if authContext == nil {
        return nil, errors.New("unauthenticated user")
    }
    
    // Mandatory filter by TenantID for security
    return r.db.Query(
        "SELECT * FROM table WHERE tenant_id = ? AND user_id = ?",
        authContext.GetTenantID(),
        authContext.GetUserID(),
    )
}
```

### 2. Layer Propagation

The context must be passed forward to keep the authentication chain intact.

```go showLineNumbers
func (s *Service) Process(ctx context.Context, data *Data) error {
    // The context already carries the security information.
    // Just pass it to the persistence layer.
    return s.repository.Save(ctx, data)
}
```

## Best Practices

1.  **Propagation**: Always pass `context.Context` as the first argument of your functions to not lose the authentication trace.
2.  **Rigid Validation**: Always check if `authContext != nil` and use `Valid()` before performing critical operations.
3.  **Data Security**: The authentication context should be created only **after** successful credential validation (such as a JWT token).
4.  **Isolation**: In multi-tenant systems, never perform database queries without including the `TenantID` in the `WHERE` clause.

---
