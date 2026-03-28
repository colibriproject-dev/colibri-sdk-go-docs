---
sidebar_position: 5
title: Comunicação Assíncrona
---

Este pacote implementa um sistema de mensageria assíncrona com suporte para múltiplos provedores (AWS SNS/SQS, Google Cloud Pub/Sub e RabbitMQ), abrangendo recursos de produção e consumo de mensagens, observabilidade e tratamento de erros.

> **Importante**: Antes de utilizar o mecanismo de comunicação assíncrona, é necessário que toda a estrutura de tópicos e filas tenha sido criada previamente no provedor escolhido.

## Configuração

O sistema detecta automaticamente o provedor de nuvem configurado através das variáveis de ambiente.

### AWS / GCP (Padrão)
Por padrão, a variável `COLIBRI_MESSAGING` é definida como `CLOUD_DEFAULT`.

### RabbitMQ
Para utilizar o RabbitMQ, configure as seguintes variáveis:
- `COLIBRI_MESSAGING`: Defina como `RABBITMQ`.
- `RABBITMQ_URL`: URL de acesso ao serviço (ex: `amqp://guest:guest@localhost:5672/`).

*Nota: Ao utilizar RabbitMQ, os serviços de nuvem (SNS/SQS e Pub/Sub) são ignorados.*

## Inicialização

Para habilitar os recursos de mensageria, adicione a inicialização na função `main.go`:

```go showLineNumbers
// Inicialização do sistema de mensageria
messaging.Initialize()
```

## Componentes Principais

### 1. Produtores (Publishers)

Utilizados para enviar mensagens para um tópico específico.

```go showLineNumbers
// Criação de um produtor
producer := messaging.NewProducer("NOME_DO_TOPICO")

// Publicação de mensagem
// O segundo parâmetro "action" ajuda a identificar o propósito da mensagem
err := producer.Publish(ctx, "create", minhaMensagem)
```

Características:
- Suporte a tipagem forte.
- Propagação automática do contexto de autenticação.
- Rastreamento de mensagens via UUID.
- Monitoramento integrado.

### 2. Consumidores (Consumers)

Para consumir mensagens, implemente a interface de consumidor e registre-a no sistema.

```go showLineNumbers
// Implementação do consumidor
type MeuConsumidor struct{}

func (c *MeuConsumidor) QueueName() string {
    return "NOME_DA_FILA"
}

func (c *MeuConsumidor) Consume(ctx context.Context, msg *ProviderMessage) error {
    var dados MinhaEstrutura
    if err := msg.DecodeAndValidateMessage(&dados); err != nil {
        return err
    }
    // Lógica de processamento da mensagem
    return nil
}

// Registro do consumidor
messaging.NewConsumer(&MeuConsumidor{})
```

### 3. Estrutura da Mensagem (`ProviderMessage`)

As mensagens recebidas pelo consumidor seguem a estrutura abaixo:

```go showLineNumbers
type ProviderMessage struct {
    ID          uuid.UUID
    Origin      string
    Action      string
    Message     any
    AuthContext *security.AuthenticationContext
}
```

## Recursos Avançados

### 1. Suporte Multi-Cloud
Abstração completa para:
- **AWS**: SNS para tópicos e SQS para filas.
- **Google Cloud**: Pub/Sub para tópicos e assinaturas.
- **RabbitMQ**: Exchanges e Queues.

### 2. Observabilidade e Resiliência
- Integração nativa com [OpenTelemetry](https://opentelemetry.io/).
- *Logging* estruturado e rastreamento de mensagens.
- Suporte a *Dead Letter Queue* (DLQ) para tratamento de falhas.

## Exemplos de Uso

### 1. Publicando um Novo Usuário

```go showLineNumbers
type Usuario struct {
    Nome  string `json:"nome"`
    Email string `json:"email"`
}

func PublicarNovoUsuario(ctx context.Context, usuario Usuario) error {
    producer := messaging.NewProducer("USUARIOS_CRIADOS")
    return producer.Publish(ctx, "create", usuario)
}
```

### 2. Processamento com Contexto de Autenticação

```go showLineNumbers
func (p *MeuConsumidor) Consume(ctx context.Context, msg *ProviderMessage) error {
    // O contexto de autenticação é preenchido automaticamente
    // se fornecido nos metadados da mensagem original.
    tenantID := msg.AuthContext.TenantID
    userID := msg.AuthContext.UserID
    
    // Processamento com isolamento de dados ou permissões específicas
    return nil
}
```

## Boas Práticas

1.  **Nomenclatura**: Use nomes descritivos em maiúsculas com *underscore* (ex: `PEDIDOS_PROCESSADOS`), preferencialmente prefixados pelo nome do serviço de origem.
2.  **Idempotência**: Garanta que o processamento da mensagem seja idempotente para evitar efeitos colaterais em caso de reprocessamento.
3.  **Validação**: Utilize sempre `msg.DecodeAndValidateMessage` para garantir que o *payload* recebido está conforme o esperado.
4.  **Monitoramento**: Acompanhe o tamanho das filas e a taxa de erros para identificar gargalos ou falhas na lógica de consumo.

___
