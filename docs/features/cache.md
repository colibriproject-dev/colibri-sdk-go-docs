---
sidebar_position: 4
title: Cache
---

O pacote oferece uma implementação robusta de cache usando [Redis](https://redis.io), com suporte a tipos genéricos e TTL (Time To Live) configurável. É especialmente útil para armazenar temporariamente dados frequentemente acessados, reduzindo a carga em outros serviços como banco de dados.

## Componentes Principais

### 1. Inicialização

Para configurarmos a aplicação para utilizar cache, a instrução abaixo deve ser incluída na função `main`.

``` go showLineNumbers
// Inicialização do cache
cacheDB.Initialize()
```

Características da inicialização:
- Conexão automática com [Redis](https://redis.io).
- Integração com [OpenTelemetry](https://opentelemetry.io/) para monitoramento.

Variáveis de ambiente necessárias:
- `CACHE_URI`: host do banco de dados.
- `CACHE_PASSWORD`: porta do banco de dados (Opcional).

### 2. Criação de Cache

``` go showLineNumbers
// Criação de um novo cache com TTL
cache := NewCache[MinhaEstrutura]("nome-do-cache", time.Hour)
```

Parâmetros:
- `name`: Nome único para identificar o cache.
- `ttl`: Tempo de vida dos dados no cache (time.Duration).

### 3. Operações Principais

#### Armazenamento (Set)

``` go showLineNumbers
// Armazenando um único item
usuario := Usuario{Id: 1, Nome: "João"}
err := cache.Set(ctx, usuario)

// Armazenando múltiplos itens
usuarios := []Usuario{
    {Id: 1, Nome: "João"},
    {Id: 2, Nome: "Maria"},
}
err := cache.Set(ctx, usuarios)
```

#### Recuperação (Get)

``` go showLineNumbers
// Recuperando um único item
usuario, err := cache.One(ctx)

// Recuperando múltiplos itens
usuarios, err := cache.Many(ctx)
```
#### Deleção (Del)

``` go showLineNumbers
// Removendo dados do cache
err := cache.Del(ctx)
```

## Recursos Avançados

### 1. Tipagem Genérica

O cache utiliza tipos genéricos do Go, permitindo forte tipagem:

``` go showLineNumbers
// Definição de estrutura
type Usuario struct {
    Id   int
    Nome string
}

// Cache tipado para Usuário
userCache := NewCache[Usuario]("usuarios", time.Hour)

// Cache tipado para outro tipo
pedidoCache := NewCache[Pedido]("pedidos", 30 * time.Minute)
```

### 2. Observabilidade

- Monitoramento via [OpenTelemetry](https://opentelemetry.io/).
- Logging estruturado de operações.
- Rastreamento de erros.

### 3. Gerenciamento de Conexão
 
- Suporte a cluster [Redis](https://redis.io).
- Fechamento seguro de conexões.

## Exemplos de Uso

### 1. Cache com Banco de Dados

O módulo de banco de dados já tem uma integração transparente com o cache.

``` go showLineNumbers
func BuscarUsuario(ctx context.Context, id int) (*Usuario, error) {
    // Criar cache com 1 hora de duração
    cache := cacheDB.NewCache[Usuario]("usuarios_cache_key", time.Hour)

    // Ao executar a query, o mecanimos interno do método 'NewCachedQuery' realiza os seguintes passos
    //  1 - Valida se existe um registro no cache para a chave informada.
    //  2 - Caso tenho o registro é retornado
    //  3 - Caso não tenha registro no cache, a consulta é executado no banco de dados
    //  4 - O resultado da consulta é salvo no cache
    //  5 - O resultado da consulta é retornado
    return sqlDB.NewCachedQuery(ctx, cache,
      "SELECT * FROM usuarios WHERE id = $1", 1).One()
}
```

### 2. Cache de Lista

``` go showLineNumbers
func ListarProdutos(ctx context.Context) ([]Produto, error) {
    cache := NewCache[Produto]("produtos", 15 * time.Minute)
    
    // Tentar recuperar lista do cache
    produtos, err := cache.Many(ctx)
    if err == nil && len(produtos) > 0 {
        return produtos, nil
    }

    // Recuperar lista de produtos de outra camada da aplicação
    produtos, err = produtos.ListarProdutos()
    if err != nil {
        return nil, err
    }

    // Atualizar cache
    if err := cache.Set(ctx, produtos); err != nil {
        log.Warn("falha ao atualizar cache", err)
    }

    return produtos, nil
}
```

### 3. Invalidação de Cache

``` go showLineNumbers
func AtualizarProduto(ctx context.Context, produto Produto) error {
    // Atualizar no banco
    if err := db.AtualizarProduto(produto); err != nil {
        return err
    }
    
    // Invalidar cache
    cache := NewCache[Produto]("produtos", time.Hour)
    if err := cache.Del(ctx); err != nil {
        log.Warn("falha ao invalidar cache", err)
    }
    
    return nil
}
```

### 4. Cache com Prefixo de Aplicação

O cache automaticamente utiliza o nome da aplicação como prefixo:

``` go
cache := NewCache[Usuario]("usuarios", time.Hour)
// Internamente armazena como "MinhaApp::usuarios"
// `MinhaApp` é o valor definido na variável de ambiente 'APP_NAME'
```

Este design permite:
- Isolamento entre diferentes aplicações
- Evita conflitos de nomenclatura
- Facilita a depuração

## Boas Práticas

1. **Defina TTLs apropriados**:
    - Dados frequentemente atualizados: TTL curto
    - Dados estáticos: TTL longo
    - Evite TTL zero (sem expiração)

2. **Trate falhas graciosamente**:
    - Sempre tenha um fallback quando o cache falhar
    - Não bloqueie operações críticas por falhas no cache

3. **Monitore o uso**:
    - Configure alertas para falhas de conexão
    - Monitore o hit/miss ratio
    - Observe o uso de memória

4. **Mantenha a consistência**:
    - Invalide caches relacionados juntos
    - Use nomes de cache consistentes
    - Documente os TTLs utilizados

___
