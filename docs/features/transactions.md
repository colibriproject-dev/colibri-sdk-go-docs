---
sidebar_position: 9
title: Controle de transações
---

Este pacote fornece uma abstração para trabalhar com transações SQL em Go, permitindo a execução segura de múltiplas operações de banco de dados como uma única unidade atômica. O sistema garante que todas as operações sejam confirmadas (commit) apenas se todas forem bem-sucedidas, ou que nenhuma seja aplicada (rollback) em caso de falha.

## Componentes Principais

### Interface Transaction

``` go showLineNumbers
type Transaction interface {
    Execute(context.Context, func(ctx context.Context) error) error
}
```

Esta interface define o contrato básico para todas as implementações de transação.

## Funcionalidades Principais

### 1. Criação de Transações

``` go showLineNumbers
// Criar uma transação com nível de isolamento padrão
transaction := sqlDB.NewTransaction()

// Criar uma transação com nível de isolamento específico
transaction := sqlDB.NewTransaction(sql.LevelSerializable)
```

### 2. Execução de Transações

``` go showLineNumbers
err := transaction.Execute(ctx, func(ctx context.Context) error {
    // Operações de banco de dados aqui
    // Retornar erro se alguma operação falhar
    return nil
})
```

### 3. Níveis de Isolamento

O sistema suporta todos os níveis de isolamento padrão SQL:
- `sql.LevelDefault`
- `sql.LevelReadUncommitted`
- `sql.LevelReadCommitted`
- `sql.LevelWriteCommitted`
- `sql.LevelRepeatableRead`
- `sql.LevelSnapshot`
- `sql.LevelSerializable`
- `sql.LevelLinearizable`

## Exemplos de Uso

### 1. Transação Simples

``` go showLineNumbers
func CreateUser(ctx context.Context, user User) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Inserir usuário
        insertUser := "INSERT INTO users (name, email) VALUES ($1, $2)"
        stmtUser := sqlDB.NewStatement(ctx, insertUser, user.Name, user.Email)
        if err := stmtUser.Execute(); err != nil {
            return err
        }
        
        // Registrar log de criação
        insertLog := "INSERT INTO user_logs (action, user_email) VALUES ($1, $2)"
        stmtLog := sqlDB.NewStatement(ctx, insertLog, "CREATE", user.Email)
        if err := stmtLog.Execute(); err != nil {
            return err
        }
        
        return nil
    })
}
```

### 2. Transferência Bancária (Operação Atômica)

``` go showLineNumbers
func TransferFunds(ctx context.Context, fromAccount, toAccount string, amount float64) error {
    transaction := sqlDB.NewTransaction(sql.LevelSerializable)
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Debitar da conta de origem
        debitQuery := "UPDATE accounts SET balance = balance - $1 WHERE account_number = $2"
        debitStmt := sqlDB.NewStatement(ctx, debitQuery, amount, fromAccount)
        if err := debitStmt.Execute(); err != nil {
            return err
        }
        
        // Verificar se a conta tem saldo suficiente
        checkQuery := "SELECT balance FROM accounts WHERE account_number = $1 AND balance >= 0"
        result, err := sqlDB.NewQuery[struct{ Balance float64 }](ctx, checkQuery, fromAccount).One()
        if err != nil {
            return err
        }
        if result == nil {
            return fmt.Errorf("saldo insuficiente na conta %s", fromAccount)
        }
        
        // Creditar na conta de destino
        creditQuery := "UPDATE accounts SET balance = balance + $1 WHERE account_number = $2"
        creditStmt := sqlDB.NewStatement(ctx, creditQuery, amount, toAccount)
        if err := creditStmt.Execute(); err != nil {
            return err
        }
        
        // Registrar a transação
        logQuery := "INSERT INTO transfers (from_account, to_account, amount) VALUES ($1, $2, $3)"
        logStmt := sqlDB.NewStatement(ctx, logQuery, fromAccount, toAccount, amount)
        if err := logStmt.Execute(); err != nil {
            return err
        }
        
        return nil
    })
}
```

### 3. Implementação de Operações CRUD Atômicas

``` go showLineNumbers
func CreateProduct(ctx context.Context, product Product, categories []string) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        // Inserir o produto
        insertProduct := "INSERT INTO products (name, price, stock) VALUES ($1, $2, $3) RETURNING id"
        var productID int
        err := sqlDB.NewStatement(ctx, insertProduct, product.Name, product.Price, product.Stock).QueryRow(&productID)
        if err != nil {
            return fmt.Errorf("falha ao inserir produto: %w", err)
        }
        
        // Associar categorias ao produto
        for _, category := range categories {
            insertCategory := "INSERT INTO product_categories (product_id, category) VALUES ($1, $2)"
            stmt := sqlDB.NewStatement(ctx, insertCategory, productID, category)
            if err := stmt.Execute(); err != nil {
                return fmt.Errorf("falha ao associar categoria %s: %w", category, err)
            }
        }
        
        // Atualizar estoque
        updateInventory := "INSERT INTO inventory_changes (product_id, quantity, reason) VALUES ($1, $2, $3)"
        inventoryStmt := sqlDB.NewStatement(ctx, updateInventory, productID, product.Stock, "Estoque inicial")
        if err := inventoryStmt.Execute(); err != nil {
            return fmt.Errorf("falha ao atualizar inventário: %w", err)
        }
        
        return nil
    })
}
```

### 4. Manipulação de Erros em Transações

``` go showLineNumbers
func ProcessBatchUpdate(ctx context.Context, items []UpdateItem) error {
    transaction := sqlDB.NewTransaction()
    
    return transaction.Execute(ctx, func(ctx context.Context) error {
        for i, item := range items {
            updateQuery := "UPDATE items SET status = $1 WHERE id = $2"
            stmt := sqlDB.NewStatement(ctx, updateQuery, item.Status, item.ID)
            if err := stmt.Execute(); err != nil {
                return fmt.Errorf("falha no item %d (%s): %w", i, item.ID, err)
            }
            
            // Log da operação
            logQuery := "INSERT INTO update_logs (item_id, old_status, new_status) VALUES ($1, $2, $3)"
            logStmt := sqlDB.NewStatement(ctx, logQuery, item.ID, item.OldStatus, item.Status)
            if err := logStmt.Execute(); err != nil {
                return fmt.Errorf("falha ao registrar log para item %d: %w", i, err)
            }
        }
        
        return nil
    })
}
```

## Boas Práticas

### 1. Gerenciamento de Recursos
- A implementação fecha automaticamente o canal de transação usando `defer close(transactionChannel)`
- Operações de rollback são executadas automaticamente em caso de erro

### 2. Níveis de Isolamento
- Escolha o nível de isolamento apropriado para suas necessidades:
    - `LevelDefault`: Para a maioria dos casos. 
    - `LevelSerializable`: Para operações que exigem consistência rigorosa.
    - `LevelReadCommitted`: Para operações de leitura que não exigem isolamento total.
    - `LevelSerializable`: É o nível de isolamento mais rigoroso disponível em transações SQL, garantindo a máxima consistência de dados.

### 3. Tratamento de Erros
- Erros de rollback são registrados e combinados com o erro original
- Erros de commit são registrados e retornados
- O sistema faz logging adequado de erros

### 4. Contexto
- O contexto é enriquecido com a transação SQL usando `context.WithValue`
- A transação está disponível em sub-operações através do contexto

## Limitações
2. **Transações Distribuídas**: Este pacote não suporta transações distribuídas entre múltiplos bancos de dados.
3. **Tempo Limite**: Não há implementação explícita de timeout nas transações, dependendo do contexto fornecido.

___
