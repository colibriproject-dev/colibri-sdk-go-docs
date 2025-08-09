---
sidebar_position: 5
title: Comunicação assíncona
---

Este pacote implementa um sistema de mensageria assíncrona com suporte para múltiplos provedores (AWS SNS/SQS e Google Cloud Pub/Sub), com recursos de produção e consumo de mensagens, observabilidade e tratamento de erros.

## Componentes Principais

### 1. Inicialização

Para inicializar os recursos de mensageria, é necessário adicionar a instrução abaixo na função `main`.

``` go showLineNumbers
// Inicialização do sistema de mensageria
messaging.Initialize()
```

O sistema detecta automaticamente o provedor de nuvem configurado (AWS ou GCP) através das variáveis de ambiente e inicializa a conexão apropriada.

### 2. Produtores (Publishers)

``` go showLineNumbers
// Criação de um produtor
producer := messaging.NewProducer("NOME_DO_TOPICO")

// Publicação de mensagem
err := producer.Publish(ctx, "create", minhaMensagem)
```

Características:
- Suporte a tipagem forte
- Contexto de autenticação automático
- Rastreamento de mensagens via UUID
- Monitoramento integrado

### 3. Consumidores (Consumers)

``` go showLineNumbers
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
    // Processamento da mensagem
    return nil
}

// Registro do consumidor
messaging.NewConsumer(&MeuConsumidor{})
```

### 4. Mensagens

``` go showLineNumbers
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

Abstração dos provedores de nuvem abaixo:
- AWS com SNS/SQS
- Google Cloud com Pub/Sub

### 2. Observabilidade

- Integração com [OpenTelemetry](https://opentelemetry.io/)
- Logging estruturado
- Rastreamento de mensagens

### 3. Resiliência

- Tratamento de erros
- Dead Letter Queue (DLQ)

## Exemplos de Uso

### 1. Publicando uma mensagem

``` go showLineNumbers
type Usuario struct {
    Nome  string `json:"nome"`
    Email string `json:"email"`
}

func PublicarNovoUsuario(ctx context.Context, usuario Usuario) error {
    producer := messaging.NewProducer("USUARIOS_CRIADOS")
    return producer.Publish(ctx, "create", usuario)
}
```

### 2. Consumindo uma mensagem

``` go showLineNumbers
type UsuarioConsumer struct{}

func (p *UsuarioConsumer) QueueName() string {
    return "FILA_USUARIOS_CRIADOS"
}

func (p *UsuarioConsumer) Consume(ctx context.Context, msg *ProviderMessage) error {
    var usuario Usuario
    if err := msg.DecodeAndValidateMessage(&usuario); err != nil {
        return err
    }

    // Processamento do usuário
    // ...

    return nil
}

// Inicialização
messaging.NewConsumer(&UsuarioConsumer{})
```

### 3. Processamento com Contexto de Autenticação

``` go showLineNumbers
type ProcessadorAutenticado struct{}

func (p *ProcessadorAutenticado) QueueName() string {
    return "FILA_AUTENTICADA"
}

func (p *ProcessadorAutenticado) Consume(ctx context.Context, msg *ProviderMessage) error {
    // Contexto de autenticação disponível automaticamente quando fornecido nos metadados da mensagem
    tenantID := msg.AuthContext.TenantID
    userID := msg.AuthContext.UserID
    
    // Processamento com contexto de segurança
    return nil
}
```

## Utilizando RabbitMQ

Para utilizar o RabbitMQ é necessário realizar passar algumas variáveis de ambiente adicionais:

- `COLIBRI_MESSAGING`: Aceita os valores `CLOUD_DEFAULT` ou `RABBITMQ`, por padrão é definido como `CLOUD_DEFAULT`.
- `RABBITMQ_URL`: URL para acessar o serviço do RabbitMQ no formato `amqp://guest:guest@localhost:5672/`

Ao se utilizar o RabbitMQ, filas de `DLQ` são criadas automaticamente para as mensagens com erro sejam encaminhadas para elas.

> Por padrão ao definir o RabbitMQ como broker padrão para utilização, é ignorado os demais serviços como `SNS/SQS` e `PubSub`.

### Definindo um consumidor para filas com RabbitMQ

Para se utilizar o RabbitMQ no consumo de filas é necessário implementar a interface `QueueConsumerConfig` que tem um método `Config() *QueueConfiguration`.

Exemplo de implemantação:

```go showLineNumbers
package consumers

import (
	"context"
	"github.com/colibriproject-dev/colibri-sdk-go/pkg/messaging"
)

type MessageConsumer struct {
}

func (c *MessageConsumer) QueueName() string {
	return "message.created.app1"
}

func (c *MessageConsumer) Consume(ctx context.Context, msg *messaging.ProviderMessage) error {
	var data Message
	if err := msg.DecodeAndValidateMessage(&data); err != nil {
		return err
	}

	// Processamento da mensagem

	return nil
}

func (c *MessageConsumer) Config() *messaging.QueueConfiguration {
	return &messaging.QueueConfiguration{
		TopicName: "message.created",
	}
}

```

Na linha 28 do exemplo acima, é definido o atributo `TopicName` que será utilizado para vincular a `queue` na exchange correta do RabbitMQ.

## Boas Práticas

1. **Nomenclatura de Tópicos/Filas**:
    - Use nomes descritivos
    - Prefixe com o nome do serviço
    - Use maiúsculas e underscore

2. **Tratamento de Erros**:
    - Implemente retry quando apropriado
    - Use DLQ para mensagens com falha
    - Monitore erros de processamento

3. **Validação de Mensagens**:
    - Use para garantir integridade `DecodeAndValidateMessage`
    - Defina estruturas com tags de validação
    - Valide antes de processar

4. **Monitoramento**:
    - Configure alertas para erros
    - Monitore latência de processamento
    - Acompanhe tamanho das filas

5. **Testes**:
    - Use para testes de integração `TestProducer`
    - Simule falhas de processamento
    - Verifique timeout e retry

___
