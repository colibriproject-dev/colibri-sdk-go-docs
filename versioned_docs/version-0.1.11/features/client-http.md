---
sidebar_position: 2
title: Cliente HTTP
---

O `RestClient` fornece uma interface robusta e simplificada para a realização de requisições HTTP em Go. Ele encapsula funcionalidades essenciais para microsserviços modernos, como *circuit breaker*, estratégias de *retry*, suporte a *cache* e manuseio de diferentes tipos de conteúdo, incluindo `multipart/form-data`.

## Estrutura Principal

Abaixo, a definição da estrutura do `RestClient`:

```go showLineNumbers
type RestClient struct {
    name       string                         // Nome identificador do cliente
    baseURL    string                         // URL base para todas as requisições
    retries    uint8                          // Número máximo de tentativas (retry)
    retrySleep uint                           // Tempo de espera entre tentativas (segundos)
    client     *http.Client                   // Cliente HTTP nativo do Go
    cb         *circuitbreaker.CircuitBreaker // Instância de Circuit Breaker
}
```

## Configuração e Instalação

Para criar uma instância do `RestClient`, utilize a função `NewRestClient` com uma estrutura de configuração:

```go showLineNumbers
config := &RestClientConfig{
    Name:                "meu-servico-cliente",
    BaseURL:             "http://api.exemplo.com",
    Timeout:             30,
    Retries:             3,
    RetrySleepInSeconds: 2,
    ProxyURL:            "http://proxy.exemplo.com:8080", // Opcional
}

client := NewRestClient(config)
```

## Funcionalidades Principais

### 1. Requisições Padronizadas

O cliente utiliza tipos genéricos para facilitar o mapeamento automático de respostas de sucesso e erro.

```go showLineNumbers
// Exemplo de requisição GET
response := Request[MinhaResponse, ErroResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/usuarios/1",
}.Call()

// Exemplo de requisição POST com corpo (JSON)
novoUsuario := Usuario{Nome: "João", Email: "joao@exemplo.com"}
response := Request[UsuarioResponse, ErroResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/usuarios",
    Body:       &novoUsuario,
}.Call()
```

### 2. Upload de Arquivos (`Multipart`)

Suporte nativo para envio de arquivos e campos de formulário:

```go showLineNumbers
arquivo := bytes.NewBufferString("conteúdo do arquivo")
response := Request[RespostaUpload, ErroUpload]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodPost,
    Path:       "/upload",
    MultipartFields: map[string]any{
        "arquivo": MultipartFile{
            FileName:    "documento.txt",
            File:        arquivo,
            ContentType: "text/plain",
        },
        "categoria": "documentos_pessoais",
    },
}.Call()
```

### 3. Integração com Cache

É possível habilitar o [cache](./cache.md) diretamente na definição da requisição para evitar chamadas repetitivas.

```go showLineNumbers
response := Request[DadosUsuario, ErroResponse]{
    Ctx:        ctx,
    Client:     client,
    HttpMethod: http.MethodGet,
    Path:       "/usuarios/1",
    Cache:      cacheDB.NewCache[DadosUsuario]("user_1_key", 10 * time.Minute),
}.Call()
```

### 4. Tratamento de Respostas

A estrutura `ResponseData` simplifica a verificação do resultado da operação:

```go showLineNumbers
if response.HasSuccess() {
    dados := response.SuccessBody()
    // Lógica para sucesso (Status 2xx)
} else if response.HasError() {
    erro := response.ErrorBody()
    // Lógica para erro mapeado
}

// Auxiliares de verificação de Status Code
if response.IsSuccessfulResponse() {
    // 2xx
} else if response.IsClientErrorResponse() {
    // 4xx
} else if response.IsServerErrorResponse() {
    // 5xx
}
```

## Resiliência e Tolerância a Falhas

### *Circuit Breaker*
O cliente monitora a saúde do serviço de destino. Por padrão:
- O circuito **abre** após 5 falhas consecutivas.
- Permanece aberto por 10 segundos antes de tentar uma nova conexão (*half-open*).

### *Retry* (Re-tentativa)
Permite configurar o comportamento em caso de falhas temporárias de rede ou indisponibilidade momentânea, definindo o número de tentativas e o intervalo entre elas.

___
