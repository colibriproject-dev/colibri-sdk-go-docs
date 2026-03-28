---
sidebar_position: 3
title: Banco de dados
---

Este pacote fornece uma camada de abstração robusta para operações com banco de dados SQL, especialmente otimizado para PostgreSQL. Inclui funcionalidades como gerenciamento de transações, migrações, consultas paginadas, cache e suporte a reflection para mapeamento objeto-relacional.

## Componentes Principais

### 1. Gerenciamento de Conexão

Para ativar o banco de dados em sua aplicação, utilize a inicialização no arquivo `main.go`:

```go showLineNumbers
// Inicialização do banco de dados
sqlDB.Initialize()
```

Características:
*   Gerenciamento automático de *pool* de conexões.
*   Configuração de número máximo de conexões.
*   Integração nativa com o sistema de observabilidade.
*   Encerramento seguro de conexões (*graceful shutdown*).

Variáveis de ambiente disponíveis:
- `SQL_DB_HOST`: Host do banco de dados.
- `SQL_DB_PORT`: Porta do banco de dados.
- `SQL_DB_NAME`: Nome do banco de dados.
- `SQL_DB_USER`: Usuário do banco de dados.
- `SQL_DB_PASSWORD`: Senha de conexão.
- `SQL_DB_MAX_OPEN_CONNS`: Define o máximo de conexões abertas no pool. *Default*: 10.
- `SQL_DB_MAX_IDLE_CONNS`: Define a quantidade de conexões inativas (*idle*) no pool. *Default*: 3.
- `SQL_DB_SSL_MODE`: Modo SSL do banco de dados (opcional).
- `SQL_DB_MIGRATION`: Ativa a execução automática de migrações (opcional).
- `MIGRATION_SOURCE_URL`: Caminho da pasta onde se encontram as migrações (opcional).

### 2. Consultas (Query)

#### Consulta Simples

Utilize `NewQuery` para realizar consultas SQL puras mapeadas para *structs* Go.

```go showLineNumbers
// Consulta que retorna um único registro
usuario, err := NewQuery[Usuario](ctx, "SELECT id, name FROM users WHERE id = $1", 1).One()

// Consulta que retorna múltiplos registros
usuarios, err := NewQuery[Usuario](ctx, "SELECT id, name FROM users WHERE active = true").Many()
```

#### Consulta com Cache

Eventualmente queremos evitar consultas repetidas ao banco de dados. Para isso, podemos utilizar o cache para evitar gargalos de desempenho.

``` go showLineNumbers
cache := cacheDB.NewCache[Usuario]("usuarios_cache_key", time.Hour)
usuario, err := NewCachedQuery(ctx, cache, 
    "SELECT * FROM usuarios WHERE id = $1", 1).One()
```

### 3. Consultas Paginadas

Uma boa prática para consultas com múltiplos registros é paginar os resultados. Com isso, evitamos sobrecarregar o servidor e a rede.

``` go showLineNumbers
pageNumber := 1
pageSize := 10
page := types.NewPageRequest(pageNumber, pageSize, []types.Sort{
    {Direction: types.ASC, Field: "nome"},
    {Direction: types.DESC, Field: "data_nascimento"},
})

resultado, err := NewPageQuery[Usuario](ctx, page, 
    "SELECT * FROM usuarios").Execute()

// resultado.Content - Lista de usuários
// resultado.TotalElements - Total de registros
```

### 4. Statements (Inserção/Atualização/Deleção)

``` go showLineNumbers
err := NewStatement(ctx, 
    "INSERT INTO usuarios (nome, email) VALUES ($1, $2)",
    "João Silva", "joao@email.com").Execute()
```

### 5. Transações

Muitas vezes, necessitamos realizar múltiplas alterações em um mesmo bloco transacional. Para isso, podemos utilizar a funcionalidade de `Transaction`.
Para definições mais avançadas, consulte a documentação detalhada de [transações](./transactions.md).

``` go showLineNumbers
tx := NewTransaction(sql.LevelSerializable)
 
err := tx.Execute(ctx, func(ctx context.Context) error {
    // Inserir usuário
    err := NewStatement(ctx, 
        "INSERT INTO usuarios (nome) VALUES ($1)", 
        "João").Execute()
    if err != nil {
        return err // Causa rollback
    }

    // Inserir perfil
    err = NewStatement(ctx, 
        "INSERT INTO perfis (usuario_id, tipo) VALUES ($1, $2)",
        1, "ADMIN").Execute()
    if err != nil {
        return err // Causa rollback
    }

    return nil // Commit da transação
})
```

### 6. Migrações

O sistema suporta migrações automáticas de banco de dados usando arquivos SQL.

Configuração:
``` env showLineNumbers
SQL_DB_MIGRATION=true
MIGRATION_SOURCE_URL=./migrations
```

Estrutura de arquivos:
``` 
migrations/
  ├── 000001_create_users.up.sql
  ├── 000001_create_users.down.sql
  ├── 000002_add_email.up.sql
  └── 000002_add_email.down.sql
```

## Recursos Avançados

### 1. Mapeamento Objeto-Relacional

O pacote utiliza reflection para mapear automaticamente resultados SQL para estruturas Go:

``` go showLineNumbers
type Profile struct {
    Id   int
    Name string
}

type User struct {
    Id       int
    Name     string
    Birthday time.Time
    Profile  Profile  // Suporte a estruturas aninhadas
}

// A consulta mapeará automaticamente para a estrutura
usuario, err := NewQuery[User](ctx, 
    "SELECT u.id, u.name, u.birthday, p.id, p.name FROM users u JOIN profiles p ON u.profile_id = p.id WHERE u.id = $1",
    1,
).One()
```

### 2. Observabilidade
- Integração com [OpenTelemetry](https://opentelemetry.io/) para monitoramento
- Logging estruturado de operações
- Rastreamento de transações

## Exemplos de Uso

### 1. CRUD Básico

``` go showLineNumbers
// Create
err := NewStatement(ctx, 
    "INSERT INTO usuarios (nome, email) VALUES ($1, $2)",
    "Maria", "maria@email.com").Execute()

// Read
usuario, err := NewQuery[Usuario](ctx, 
    "SELECT * FROM usuarios WHERE email = $1",
    "maria@email.com").One()

// Update
err = NewStatement(ctx, 
    "UPDATE usuarios SET nome = $1 WHERE email = $2",
    "Maria Silva", "maria@email.com").Execute()

// Delete
err = NewStatement(ctx, 
    "DELETE FROM usuarios WHERE email = $1",
    "maria@email.com").Execute()
```

### 2. Consulta Complexa com Join

``` go showLineNumbers
type ResultadoConsulta struct {
    NomeUsuario string
    TotalPedidos int
    UltimoPedido time.Time
}

resultado, err := NewQuery[ResultadoConsulta](ctx, `
    SELECT 
        u.nome as nome_usuario,
        COUNT(p.id) as total_pedidos,
        MAX(p.data) as ultimo_pedido
    FROM usuarios u
    LEFT JOIN pedidos p ON p.usuario_id = u.id
    GROUP BY u.id
`).Many()
```

### 3. Paginação com Filtros

``` go showLineNUmbers
pageNumber := 1
pageSize := 10
page := types.NewPageRequest(pageNumber, pageSize, []types.Sort{{
    Direction: types.DESC,
    Field: "data_criacao"
}})

resultados, err := NewPageQuery[Pedido](
    ctx,
    page,
    `SELECT * FROM pedidos WHERE status = $1 
     AND data_criacao >= $2`,
    "PENDENTE",
    time.Now().AddDate(0, -1, 0),
).Execute()
```

___
