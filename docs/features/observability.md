---
sidebar_position: 7
title: Observabilidade
---

Este pacote fornece uma camada completa de observabilidade para aplicações, integrando métricas, tracing e logging. É encapsulada a implementação com [OpenTelemetry](https://opentelemetry.io/), oferecendo recursos para monitoramento de transações, segmentos e erros.

## Componentes Principais

### 1. Inicialização

A configuração é realizada automaticamente ao iniciar o `colibri-sdk-go`.

São configurados os seguintes componentes:
- Conexão [OpenTelemetry](https://opentelemetry.io/)
- Configuração de métricas padrão

#### Utilizando OpenTelemetry

Para utilizar OpenTelemetry, basta adicionar as variáveis de ambiente:

```dotenv showLineNumbers
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

> `http://localhost:4318` é apenas um exemplo, defina com o endereço do seu serviço OpenTelemetry.

### 2. Transações de monitoramento

``` go showLineNumbers
// Iniciando uma nova transação monitoramento
txn, ctx := monitoring.StartTransaction(ctx, "MinhaOperacao")
defer monitoring.EndTransaction(txn)
```

### 3. Segmentos

``` go
// Criando um segmento de transação
segment := monitoring.StartTransactionSegment(ctx, "MinhaOperacao", map[string]string{
    "operacao": "consulta",
    "tipo": "banco",
})
defer monitoring.EndTransactionSegment(segment)
```

> Atenção: Para que o segmento possa ser relacionado corretamente com a transação, devemos passar o atributo `ctx` criado na transação principal.

### 4. Rastreamento de Erros

``` go showLineNumbers
// Notificando erros
if err != nil {
    monitoring.NoticeError(txn, err)
    return err
}
```

## Exemplos de Uso

### 1. Monitoramento de Serviços Externos

``` go showLineNumbers
func (s *Service) ChamarAPIExterna(ctx context.Context) error {
    segment := monitoring.StartTransactionSegment(ctx, "API_Pagamento")
    defer monitoring.EndTransactionSegment(segment)
    
    resp, err := http.Get("https://api.externa.com/recurso")
    if err != nil {
        monitoring.NoticeError(monitoring.GetTransactionInContext(ctx), err)
        return err
    }
    defer resp.Body.Close()

    return nil
}
```

## Boas Práticas

1. **Nomeação de Transações**:
    - Use nomes descritivos e consistentes
    - Siga um padrão de nomenclatura
    - Evite nomes dinâmicos

2. **Gestão de Contexto**:
    - Sempre propague o contexto
    - Use defer para finalizar transações
    - Mantenha o ciclo de vida correto

3. **Atributos**:
    - Adicione atributos relevantes
    - Não adicione dados sensíveis
    - Use chaves consistentes

4. **Erros**:
    - Notifique erros relevantes
    - Adicione contexto aos erros
    - Evite notificar erros esperados

5. **Performance**:
    - Monitore tempos de resposta
    - Observe uso de recursos
    - Identifique gargalos

6. **Segurança**:
    - Não log dados sensíveis
    - Respeite as leis de proteção de dados

___
