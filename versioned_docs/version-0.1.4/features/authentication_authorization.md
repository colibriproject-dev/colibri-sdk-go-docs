---
sidebar_position: 8
title: Autenticação e autorização
---

O pacote de contexto de autenticação fornece uma estrutura simples mas poderosa para gerenciar a informação do usuário autenticado através da aplicação. Ele utiliza o mecanismo de contexto do Go para armazenar e recuperar detalhes do usuário, sendo especialmente útil em aplicações multi-tenant.

## Componentes Principais

### 1. AuthenticationContext

A estrutura central que mantém as informações de autenticação:

``` go showLineNumbers
type AuthenticationContext struct {
    TenantID string `json:"tenantId"`
    UserID   string `json:"userId"`
}
```

### 2. Interface IAuthenticationContext

Define o contrato para acessar informações de autenticação:

``` go showLineNumbers
type IAuthenticationContext interface {
    GetUserID() string
    GetTenantID() string
}
```

### 3. Estrutura de Usuário

``` go showLineNumbers
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

### 1. Criação do Contexto de Autenticação

``` go showLineNUmbers
authContext := security.NewAuthenticationContext("tenant123", "user456")
```

### 2. Armazenamento no Contexto

``` go showLineNUmbers
// Adiciona as informações de autenticação ao contexto
ctx = authContext.SetInContext(context.Background())
```

### 3. Recuperação do Contexto

``` go showLineNUmbers
// Recupera as informações de autenticação do contexto
authContext := security.GetAuthenticationContext(ctx)
if authContext != nil {
    tenantID := authContext.GetTenantID()
    userID := authContext.GetUserID()
}
```

### 4. Validação do Contexto

``` go showLineNUmbers
// Verifica se o contexto de autenticação é válido
if authContext.Valid() {
    // Contexto válido, prosseguir com operação
}
```

## Exemplos de Uso

### 1. Middleware HTTP de Autenticação
``` go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extrair e validar token de autenticação (exemplo simplificado)
        token := r.Header.Get("Authorization")
        
        // Processar token e extrair identificadores
        tenantID := "tenant-extraído-do-token"
        userID := "user-extraído-do-token"
        
        // Criar contexto de autenticação
        authContext := security.NewAuthenticationContext(tenantID, userID)
        
        // Verificar validade
        if !authContext.Valid() {
            http.Error(w, "Credenciais inválidas", http.StatusUnauthorized)
            return
        }
        
        // Adicionar ao contexto e prosseguir
        ctx := authContext.SetInContext(r.Context())
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 2. Acesso a Dados Multi-tenant

``` go showLineNUmbers
func (r *Repository) BuscarDadosDoUsuário(ctx context.Context) (*DadosUsuário, error) {
    // Recuperar contexto de autenticação
    authContext := security.GetAuthenticationContext(ctx)
    if authContext == nil {
        return nil, errors.New("usuário não autenticado")
    }
    
    // Utilizar informações para buscar dados específicos do tenant/usuário
    return r.db.Query(
        "SELECT * FROM dados WHERE tenant_id = ? AND user_id = ?",
        authContext.GetTenantID(),
        authContext.GetUserID(),
    )
}
```

### 3. Propagação em Serviços

``` go showLineNUmbers
type Service struct {
    repository Repository
}

func (s *Service) ProcessarOperação(ctx context.Context, dados *Dados) error {
    // O contexto já contém as informações de autenticação
    // Apenas passar adiante para manter a cadeia de autenticação
    return s.repository.SalvarDados(ctx, dados)
}
```

## Boas Práticas

1. **Propagação do Contexto**:
    - Sempre propague o contexto através das chamadas de função
    - Não crie novos contextos sem necessidade
    - Mantenha a cadeia de informações de autenticação

2. **Validação**:
    - Verifique `authContext != nil` antes de acessar suas propriedades
    - Use para garantir que os IDs são válidos `Valid()`
    - Nunca assuma que o contexto está presente

3. **Segurança**:
    - O contexto de autenticação deve ser criado apenas após validação de credenciais
    - Não armazene informações sensíveis (senhas, tokens) no contexto
    - Limite as informações ao mínimo necessário

4. **Isolamento Multi-tenant**:
    - Use sempre o TenantID em consultas de banco de dados
    - Aplique filtros de tenant em todas as operações de dados
    - Mantenha isolamento rígido entre dados de diferentes tenants

___
