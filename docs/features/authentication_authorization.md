---
sidebar_position: 8
title: Autenticação e Autorização
---

O pacote de contexto de autenticação fornece uma estrutura simples e poderosa para gerenciar informações do usuário autenticado em toda a aplicação. Utilizando o mecanismo de contexto nativo do Go, ele permite armazenar e recuperar detalhes do usuário de forma segura, sendo ideal para aplicações *multi-tenant*.

## Componentes Principais

### 1. `AuthenticationContext`

A estrutura central que mantém os dados de autenticação:

```go showLineNumbers
type AuthenticationContext struct {
    TenantID string `json:"tenantId"`
    UserID   string `json:"userId"`
}
```

### 2. Interface `IAuthenticationContext`

Define o contrato para acesso padronizado às informações:

```go showLineNumbers
type IAuthenticationContext interface {
    GetUserID() string
    GetTenantID() string
}
```

### 3. Estrutura de `User`

Representa o perfil completo do usuário no sistema:

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

## Funcionalidades Principais

### 1. Criação do Contexto

```go showLineNumbers
authContext := security.NewAuthenticationContext("tenant123", "user456")
```

### 2. Armazenamento no Contexto Go

Adicione as informações de autenticação ao contexto da requisição:

```go showLineNumbers
ctx = authContext.SetInContext(context.Background())
```

### 3. Recuperação de Dados

Recupere as informações de qualquer lugar que tenha acesso ao contexto:

```go showLineNumbers
authContext := security.GetAuthenticationContext(ctx)
if authContext != nil {
    tenantID := authContext.GetTenantID()
    userID := authContext.GetUserID()
}
```

### 4. Validação

Verifique se o contexto possui dados válidos:

```go showLineNumbers
if authContext.Valid() {
    // Prosseguir com a operação
}
```

## Exemplos de Uso

### 1. Acesso a Dados *Multi-tenant*

Garanta o isolamento de dados utilizando o `TenantID` recuperado do contexto.

```go showLineNumbers
func (r *Repository) BuscarDados(ctx context.Context) (*Dados, error) {
    authContext := security.GetAuthenticationContext(ctx)
    if authContext == nil {
        return nil, errors.New("usuário não autenticado")
    }
    
    // Filtro obrigatório por TenantID para segurança
    return r.db.Query(
        "SELECT * FROM tabela WHERE tenant_id = ? AND user_id = ?",
        authContext.GetTenantID(),
        authContext.GetUserID(),
    )
}
```

### 2. Propagação entre Camadas

O contexto deve ser passado adiante para manter a cadeia de autenticação íntegra.

```go showLineNumbers
func (s *Service) Processar(ctx context.Context, dados *Dados) error {
    // O contexto já carrega as informações de segurança.
    // Apenas repasse para a camada de persistência.
    return s.repository.Salvar(ctx, dados)
}
```

## Boas Práticas

1.  **Propagação**: Sempre passe o `context.Context` como primeiro argumento de suas funções para não perder o rastro da autenticação.
2.  **Validação Rígida**: Sempre verifique se `authContext != nil` e utilize `Valid()` antes de realizar operações críticas.
3.  **Segurança de Dados**: O contexto de autenticação deve ser criado apenas **após** a validação bem-sucedida de credenciais (como um token JWT).
4.  **Isolamento**: Em sistemas *multi-tenant*, nunca realize consultas ao banco de dados sem incluir o `TenantID` na cláusula `WHERE`.

___
