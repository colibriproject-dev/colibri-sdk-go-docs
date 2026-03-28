---
sidebar_position: 1
title: Imagem Docker
---

Nesta seção, vamos aprender como empacotar sua aplicação utilizando containers Docker. Utilizaremos o conceito de *multi-stage build* para garantir uma imagem final leve, segura e otimizada para produção.

## O arquivo `Dockerfile`

Crie um arquivo chamado `Dockerfile` na raiz do seu projeto com o seguinte conteúdo:

```dockerfile showLineNumbers
# Estágio 1: Compilação
FROM golang:1.24-alpine3.21 AS build

WORKDIR /build

# Copia os arquivos de dependências primeiro para aproveitar o cache do Docker
COPY go.mod go.sum ./
RUN go mod download

# Copia o restante do código fonte
COPY . .

# Executa os testes unitários
RUN go test ./... -v -failfast || exit 1

# Compila o binário estático
RUN CGO_ENABLED=0 GOOS=linux go build -o app-bin ./cmd/main.go

# Estágio 2: Imagem Final (Runtime)
FROM gcr.io/distroless/static

WORKDIR /app

# Copia apenas o binário compilado do estágio anterior
COPY --from=build /build/app-bin /app/

# Define o ponto de entrada da aplicação
ENTRYPOINT ["/app/app-bin"]
```

### Por que usar *Multi-stage Build*?

1.  **Segurança**: A imagem final não contém o código fonte, compiladores ou ferramentas de *build*, reduzindo a superfície de ataque.
2.  **Tamanho**: Utilizamos a imagem `distroless/static` no estágio final, que é extremamente pequena (contém apenas o essencial para executar binários Go), resultando em imagens de poucos MBs.
3.  **Eficiência**: O Docker faz o *cache* das camadas. Ao copiar `go.mod` e `go.sum` antes do restante do código, as dependências só serão baixadas novamente se houver alteração nestes arquivos.

## O arquivo `.dockerignore`

Para evitar que arquivos desnecessários (como logs, binários locais ou a pasta `.git`) sejam enviados para o contexto de *build* do Docker, crie um arquivo `.dockerignore`:

```text
.git
*.log
bin/
vendor/
.env
```

## Construindo a imagem

Execute o comando abaixo no diretório raiz do seu projeto:

```shell
docker build -t meu-microsservico:latest .
```

## Executando o container

Para iniciar sua aplicação em um container, passando as variáveis de ambiente necessárias para o `colibri-sdk-go`:

```shell showLineNumbers
docker run --rm \
  -e ENVIRONMENT=production \
  -e APP_NAME=meu-microsservico \
  -e APP_TYPE=service \
  -e CLOUD=none \
  -e PORT=8080 \
  -p 8080:8080 \
  meu-microsservico:latest
```

### Explicação dos parâmetros:
*   `--rm`: Remove o container automaticamente após o encerramento.
*   `-e`: Define variáveis de ambiente.
*   `-p 8080:8080`: Mapeia a porta 8080 do seu computador para a porta 8080 do container.

## Conclusão

Utilizando esta abordagem, você garante que sua aplicação Go rode de forma consistente em qualquer ambiente (desenvolvimento, homologação ou produção), com o máximo de performance e segurança.

___
