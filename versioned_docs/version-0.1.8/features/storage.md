---
sidebar_position: 6
title: Storage
---

Este pacote fornece uma abstração para operações de armazenamento em nuvem, suportando tanto AWS S3 quanto Google Cloud Storage (GCS). Oferece operações básicas de manipulação de arquivos com monitoramento integrado.

## Componentes Principais

### 1. Inicialização

Para configurarmos a aplicação para utilizar a funcionalidade de `storage`, a instrução abaixo deve ser incluída na função `main`.

``` go showLineNumbers
// Inicialização do storage
storage.Initialize()
```

O sistema detecta automaticamente o provedor de nuvem configurado (AWS ou GCP) e inicializa a conexão apropriada.

### 2. Operações Principais

#### Upload de Arquivo

``` go showLineNumbers
location, err := storage.UploadFile(
    ctx,
    "nome-do-bucket",
    "caminho/arquivo.txt",
    arquivoMultipart,
)
```

#### Download de Arquivo

``` go showLineNumbers
arquivo, err := storage.DownloadFile(
    ctx,
    "nome-do-bucket",
    "caminho/arquivo.txt",
)
```

#### Remoção de Arquivo

``` go showLineNumbers
err := storage.DeleteFile(
    ctx,
    "nome-do-bucket",
    "caminho/arquivo.txt",
)
```

## Recursos Avançados

### 1. Abstração do provedor de nuvem

- AWS S3
- Google Cloud Storage

### 2. Observabilidade
- 
- Integração com [OpenTelemetry](https://opentelemetry.io/)
- Monitoramento de transações
- Logging estruturado

## Exemplos de Uso

### 1. Upload de Arquivo

``` go showLineNumbers
func UploadProfilePicture(ctx context.Context, userID string, file *multipart.File) (string, error) {
    bucket := "profile-pictures"
    key := fmt.Sprintf("users/%s/profile.jpg", userID)
    
    location, err := storage.UploadFile(ctx, bucket, key, file)
    if err != nil {
        log.Error().Err(err).Msg("falha ao fazer upload da foto")
        return "", err
    }
    
    return location, nil
}
```

### 2. Download de Arquivo

``` go showLineNumbers
func ProcessarArquivo(ctx context.Context, bucket, key string) error {
    // Download do arquivo
    arquivo, err := storage.DownloadFile(ctx, bucket, key)
    if err != nil {
        return fmt.Errorf("erro no download: %w", err)
    }
    defer arquivo.Close()
    
    // Processamento do arquivo
    dados, err := io.ReadAll(arquivo)
    if err != nil {
        return fmt.Errorf("erro na leitura: %w", err)
    }
    
    // ... processamento adicional ...
    
    return nil
}
```

### 3. Remoção de Arquivo

``` go showLineNumbers
func ManipularArquivoTemporario(ctx context.Context, bucket, key string) error {
    // Download para arquivo temporário
    fileToDelete, err := storage.DownloadFile(ctx, bucket, key)
    if err != nil {
        return err
    }
    defer os.Remove(fileToDelete.Name()) // Remove arquivo do storage
    defer fileToDelete.Close()

    // ...
    
    return nil
}
```

## Boas Práticas

1. **Gerenciamento de Recursos**:
    - Sempre feche arquivos após uso
    - Limpe arquivos temporários
    - Use defer para garantir limpeza

2. **Estrutura de Buckets**:
    - Use nomes descritivos
    - Organize em hierarquia lógica
    - Mantenha consistência na nomenclatura

3. **Tratamento de Erros**:
    - Verifique erros de todas as operações
    - Log apropriado de falhas
    - Implemente retry quando necessário

4. **Segurança**:
    - Valide tipos de arquivo
    - Implemente limites de tamanho
    - Use políticas de acesso adequadas

5. **Performance**:
    - Use streaming para arquivos grandes
    - Monitore tempos de operação

6. **Monitoramento**:
    - Configure alertas para falhas
    - Monitore uso de espaço
    - Acompanhe custos de armazenamento

___
