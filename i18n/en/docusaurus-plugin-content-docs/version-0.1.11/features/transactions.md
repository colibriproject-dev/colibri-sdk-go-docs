---
sidebar_position: 9
title: Transaction Control
---

This package provides a robust abstraction for working with SQL transactions in Go, allowing the safe execution of multiple database operations as a single atomic unit. The system ensures data integrity through a commit mechanism only if all operations are successful, or an automatic rollback in case of any failure.

## Main Components

### `Transaction` Interface

The `Transaction` interface defines the contract for executing transactional blocks:

```go showLineNumbers
type Transaction interface {
    Execute(context.Context, func(ctx context.Context) error) error
}
```

This interface defines the basic contract for all transaction implementations.

## Core Features

### 1. Creating Transactions

You can create a transaction with the default isolation level or specify a custom level:

```go showLineNumbers
// Create a transaction with the default isolation level (Read Committed in PostgreSQL)
transaction := sqlDB.NewTransaction()

// Create a transaction with Serializable isolation level
transaction := sqlDB.NewTransaction(sql.LevelSerializable)
```

### 2. Executing Transactional Blocks

Execution occurs within an anonymous function (callback). If the function returns `nil`, the transaction is committed; if it returns an error, the transaction is rolled back.

```go showLineNumbers
err := transaction.Execute(ctx, func(ctx context.Context) error {
    // All operations inside here must use the 'ctx' provided by the callback
    // to ensure they inherit the same transaction.
    return nil 
})
```

## Supported Isolation Levels

The framework supports all standard isolation levels from the `database/sql` package:
- `sql.LevelDefault`
- `sql.LevelReadUncommitted`
- `sql.LevelReadCommitted`
- `sql.LevelRepeatableRead`
- `sql.LevelSerializable`
*(Other levels like `Snapshot` and `Linearizable` depend on the support of the database driver used).*

## Usage Examples

### 1. Simple Transaction

```go showLineNumbers
func CreateUser(ctx context.Context, user User) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Insert user
        insertUser := "INSERT INTO users (name, email) VALUES ($1, $2)"
        stmtUser := sqlDB.NewStatement(ctx, insertUser, user.Name, user.Email)
        if err := stmtUser.Execute(); err != nil {
            return err
        }
        
        // Register creation log
        insertLog := "INSERT INTO user_logs (action, user_email) VALUES ($1, $2)"
        stmtLog := sqlDB.NewStatement(ctx, insertLog, "CREATE", user.Email)
        if err := stmtLog.Execute(); err != nil {
            return err
        }
        
        return nil
    })
}
```

### 2. Bank Transfer (Atomic Operation)

```go showLineNumbers
func TransferFunds(ctx context.Context, fromAccount, toAccount string, amount float64) error {
    transaction := sqlDB.NewTransaction(sql.LevelSerializable)
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Debit from origin account
        debitQuery := "UPDATE accounts SET balance = balance - $1 WHERE account_number = $2"
        debitStmt := sqlDB.NewStatement(ctx, debitQuery, amount, fromAccount)
        if err := debitStmt.Execute(); err != nil {
            return err
        }
        
        // Check if account has sufficient balance
        checkQuery := "SELECT balance FROM accounts WHERE account_number = $1 AND balance >= 0"
        result, err := sqlDB.NewQuery[struct{ Balance float64 }](ctx, checkQuery, fromAccount).One()
        if err != nil {
            return err
        }
        if result == nil {
            return fmt.Errorf("insufficient balance in account %s", fromAccount)
        }
        
        // Credit to destination account
        creditQuery := "UPDATE accounts SET balance = balance + $1 WHERE account_number = $2"
        creditStmt := sqlDB.NewStatement(ctx, creditQuery, amount, toAccount)
        if err := creditStmt.Execute(); err != nil {
            return err
        }
        
        // Register the transaction
        logQuery := "INSERT INTO transfers (from_account, to_account, amount) VALUES ($1, $2, $3)"
        logStmt := sqlDB.NewStatement(ctx, logQuery, fromAccount, toAccount, amount)
        if err := logStmt.Execute(); err != nil {
            return err
        }
        
        return nil
    })
}
```

### 3. Implementing Atomic CRUD Operations

```go showLineNumbers
func CreateProduct(ctx context.Context, product Product, categories []string) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Insert the product
        insertProduct := "INSERT INTO products (name, price, stock) VALUES ($1, $2, $3) RETURNING id"
        var productID int
        err := sqlDB.NewStatement(ctx, insertProduct, product.Name, product.Price, product.Stock).QueryRow(&productID)
        if err != nil {
            return fmt.Errorf("failed to insert product: %w", err)
        }
        
        // Associate categories with the product
        for _, category := range categories {
            insertCategory := "INSERT INTO product_categories (product_id, category) VALUES ($1, $2)"
            stmt := sqlDB.NewStatement(ctx, insertCategory, productID, category)
            if err := stmt.Execute(); err != nil {
                return fmt.Errorf("failed to associate category %s: %w", category, err)
            }
        }
        
        // Update stock
        updateInventory := "INSERT INTO inventory_changes (product_id, quantity, reason) VALUES ($1, $2, $3)"
        inventoryStmt := sqlDB.NewStatement(ctx, updateInventory, productID, product.Stock, "Initial stock")
        if err := inventoryStmt.Execute(); err != nil {
            return fmt.Errorf("failed to update inventory: %w", err)
        }
        
        return nil
    })
}
```

### 4. Error Handling in Transactions

```go showLineNumbers
func ProcessBatchUpdate(ctx context.Context, items []UpdateItem) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        for i, item := range items {
            updateQuery := "UPDATE items SET status = $1 WHERE id = $2"
            stmt := sqlDB.NewStatement(ctx, updateQuery, item.Status, item.ID)
            if err := stmt.Execute(); err != nil {
                return fmt.Errorf("failed on item %d (%s): %w", i, item.ID, err)
            }
            
            // Operation log
            logQuery := "INSERT INTO update_logs (item_id, old_status, new_status) VALUES ($1, $2, $3)"
            logStmt := sqlDB.NewStatement(ctx, logQuery, item.ID, item.OldStatus, item.Status)
            if err := logStmt.Execute(); err != nil {
                return fmt.Errorf("failed to register log for item %d: %w", i, err)
            }
        }
        
        return nil
    })
}
```

## Best Practices

### 1. Resource Management
- The implementation automatically closes the transaction channel using `defer close(transactionChannel)`.
- Rollback operations are automatically executed in case of error.

### 2. Isolation Levels
- Choose the appropriate isolation level for your needs:
    - `LevelDefault`: For most cases. 
    - `LevelSerializable`: For operations requiring strict consistency.
    - `LevelReadCommitted`: For read operations that do not require full isolation.
    - `LevelSerializable`: This is the most rigorous isolation level available in SQL transactions, ensuring maximum data consistency.

### 3. Error Handling
- Rollback errors are logged and combined with the original error.
- Commit errors are logged and returned.
- The system performs proper error logging.

### 4. Context
- The context is enriched with the SQL transaction using `context.WithValue`.
- The transaction is available in sub-operations through the context.

## Limitations
2. **Distributed Transactions**: This package does not support distributed transactions across multiple databases.
3. **Timeout**: There is no explicit timeout implementation in transactions, depending on the provided context.

---
