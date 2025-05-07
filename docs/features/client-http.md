---
sidebar_position: 2
title: Cliente HTTP
---

O `RestClient` é uma estrutura em Go que fornece uma interface robusta para fazer requisições HTTP. Ele inclui recursos como circuit breaker, retry de requisições, cache, e suporte para diferentes tipos de requisições HTTP incluindo multipart/form-data.

## Estrutura
``` go
type RestClient struct {
    name       string                         // Nome do cliente REST
    baseURL    string                         // URL base para todas as requisições
    retries    uint8                          // Número de tentativas de retry
    retrySleep uint                           // Tempo de espera entre retries (em segundos)
    client     *http.Client                   // Cliente HTTP
    cb         *circuitbreaker.CircuitBreaker // Circuit breaker
}
```
## Configuração
Para criar uma instância do `RestClient`, utilize a função `NewRestClient`, passando as opções como no exemplo abaixo: 

``` go showLineNumbers
config := &RestClientConfig{
    Name:                "meu-cliente",
    BaseURL:             "http://api.exemplo.com",
    Timeout:             30,
    Retries:             3,
    RetrySleepInSeconds: 2
}

client := NewRestClient(config)
```

## Principais Funcionalidades

### 1. Requisições HTTP

O cliente suporta todos os métodos HTTP padrão (GET, POST, PUT, PATCH, DELETE, etc) através da estrutura : `Request`

``` go showLineNumbers
// Exemplo de GET
response := Request[MinhaResponseStruct, ErrorStruct]{
    Ctx:        context.Background(),
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/usuarios/1",
}.Call()

// Exemplo de POST com corpo
novoUsuario := Usuario{Nome: "João", Email: "joao@exemplo.com"}
response := Request[UsuarioResponse, ErrorResponse]{
    Ctx:        context.Background(),
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/usuarios",
    Body:       &novoUsuario,
}.Call()
```
### 2. Upload de Arquivos (Multipart)

Suporte para envio de arquivos usando `multipart/form-data`:

``` go showLineNumbers
arquivo := bytes.NewBufferString("conteúdo do arquivo")
response := Request[RespostaUpload, ErroUpload]{
    Ctx:        context.Background(),
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/upload",
    MultipartFields: map[string]any{
        "arquivo": MultipartFile{
            FileName:    "documento.txt",
            File:        arquivo,
            ContentType: "text/plain",
        },
        "descricao": "Meu documento",
    },
}.Call()
```

### 3. Cache

Suporte para [cache](./cache.md) nas requisições.

``` go showLineNumbers
response := Request[DadosUsuario, ErroResponse]{
    Ctx:        context.Background(),
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/usuarios/1",
    Cache:      cacheDB.NewCache[DadosUsuario]("keyName", 10 * time.Second), // cria uma chave com 10s de TTL.
}.Call()
```

### 4. Tratamento de Respostas

O cliente fornece uma interface que facilita o tratamento das respostas: `ResponseData`.

``` go showLineNumbers
if response.HasSuccess() {
    dados := response.SuccessBody()
    // processar dados de sucesso
} else if response.HasError() {
    erro := response.ErrorBody()
    // tratar erro
}

// Verificar tipo de resposta
if response.IsSuccessfulResponse() {
    // Status 2xx
} else if response.IsClientErrorResponse() {
    // Status 4xx
} else if response.IsServerErrorResponse() {
    // Status 5xx
}
```
## Recursos de Resiliência

### Circuit Breaker

O cliente implementa um circuit breaker que:
- Abre após 5 falhas consecutivas
- Permanece aberto por 10 segundos

### Retry

Sistema de retry configurável que:
- Permite definir número máximo de tentativas
- Configura tempo de espera entre tentativas

## Tipos de Dados

O cliente utiliza interfaces genéricas para dados de requisição e resposta:
- `RequestData`: Interface para dados de requisição.
- `ResponseSuccessData`: Interface para dados de resposta de sucesso.
- `ResponseErrorData`: Interface para dados de resposta de erro.

Este design permite forte tipagem e flexibilidade ao mesmo tempo.
