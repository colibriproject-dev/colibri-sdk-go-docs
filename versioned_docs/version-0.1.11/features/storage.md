---
sidebar_position: 6
title: Storage
---

O pacote de *Storage* fornece uma camada de abstração para operações de armazenamento em nuvem, com suporte nativo para AWS S3 e Google Cloud Storage (GCS). Ele simplifica a manipulação de arquivos, oferecendo uma interface unificada e monitoramento integrado.

## Configuração

O sistema detecta automaticamente o provedor de nuvem configurado (AWS ou GCP) através das variáveis de ambiente padrão de cada provedor. Certifique-se de que as credenciais e permissões adequadas estejam configuradas no ambiente de execução.

## Inicialização

Para habilitar as funcionalidades de *storage*, inclua a inicialização no seu arquivo `main.go`:

```go showLineNumbers
// Inicialização do storage
storage.Initialize()
```

## Operações Principais

### 1. Upload de Arquivo

Realiza o envio de um arquivo para o *bucket* especificado.

```go showLineNumbers
location, err := storage.UploadFile(
    ctx,
    "nome-do-bucket",
    "caminho/destino/arquivo.txt",
    arquivoReader, // io.Reader
)
```

### 2. Download de Arquivo

Recupera um arquivo do armazenamento em nuvem.

```go showLineNumbers
arquivo, err := storage.DownloadFile(
    ctx,
    "nome-do-bucket",
    "caminho/do/arquivo.txt",
)
// Lembre-se de fechar o arquivo após o uso
defer arquivo.Close()
```

### 3. Remoção de Arquivo

Exclui um arquivo permanentemente do *bucket*.

```go showLineNumbers
err := storage.DeleteFile(
    ctx,
    "nome-do-bucket",
    "caminho/do/arquivo.txt",
)
```

## Recursos e Integrações

### Abstração Multi-Cloud
- **AWS S3**: Utiliza o SDK oficial da AWS.
- **Google Cloud Storage**: Utiliza as bibliotecas padrão do GCP.

### Observabilidade
- **OpenTelemetry**: Rastreamento automático de operações de *upload*, *download* e *delete*.
- **Logging**: Registros estruturados para facilitar a depuração de falhas de permissão ou conectividade.

## Exemplos de Uso

### Processamento de Arquivo em Memória

```go showLineNumbers
func ProcessarRelatorio(ctx context.Context, bucket, key string) error {
    arquivo, err := storage.DownloadFile(ctx, bucket, key)
    if err != nil {
        return fmt.Errorf("falha no download: %w", err)
    }
    defer arquivo.Close()

    dados, err := io.ReadAll(arquivo)
    if err != nil {
        return fmt.Errorf("erro na leitura: %w", err)
    }

    // Lógica de processamento...
    return nil
}
```

## Boas Práticas

1.  **Gerenciamento de Recursos**: Sempre utilize `defer arquivo.Close()` ao realizar o download para evitar vazamentos de memória e conexões abertas.
2.  **Segurança**: Nunca exponha chaves de acesso diretamente no código; utilize variáveis de ambiente ou gerenciadores de segredos.
3.  **Idempotência**: Garanta que o sistema trate corretamente tentativas de *upload* para arquivos que já existem, se necessário.
4.  **Streaming**: Para arquivos grandes, prefira trabalhar com *streams* (`io.Reader`/`io.Writer`) em vez de carregar todo o conteúdo na memória.

___
