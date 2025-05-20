---
sidebar_position: 1
title: Imagem Docker
---

Nesta documentação, vamos aprender como criar e publicar uma API web utilizando containers Docker. Vamos usar como base um exemplo de aplicação chamada `my-awesome-service`.

Para isso crie um arquivo com o nome `Dockerfile` na raiz do projeto. 

## Passo 1: Entendendo o Dockerfile

O Dockerfile é o arquivo que contém as instruções para construir nossa imagem Docker. Vamos analisar o Dockerfile que usaremos como base:

``` dockerfile showLineNumbers
FROM golang:1.24-alpine3.21 AS build

WORKDIR /build

COPY . .

RUN go test ./... -v -failfast || exit 1
RUN CGO_ENABLED=0 GOOS=linux go build -o my-awesome-service ./cmd/cli/main.go

FROM gcr.io/distroless/static

WORKDIR /app
COPY --from=build /build/my-awesome-service /app/

ENTRYPOINT ["/app/my-awesome-service"]
```
Este Dockerfile utiliza o conceito de `multi-stage build`, que consiste em:

1. **Primeiro estágio (build)**:
    - Linha 1: usa a imagem como base `golang:1.24-alpine3.21`.
    - Linha 3: define a pasta de trabalho como `/build`.
    - Linha 5: copia todo o código fonte para dentro do container.
    - Linha 7: executa os testes.
    - Linha 8: compila a aplicação gerando um binário chamado `my-awesome-service`.

2. **Segundo estágio**:
    - Linha 10: usa a imagem que é extremamente leve e segura `gcr.io/distroless/static`
    - Linha 12: define a pasta padrão como `/app`.
    - Linha 13: copia apenas o binário compilado do primeiro estágio.
    - Linha 15: define o ponto de entrada para executar a aplicação.

## Passo 2: Criando o arquivo .dockerignore

O arquivo `.dockerignore` é importante para excluir arquivos desnecessários durante a construção da imagem: 

```ignorelang showLineNumbers
*.log
*.txt
my-awesome-service
logs/
```

Isso impede que arquivos de log, arquivos de texto, o binário já compilado localmente e diretórios de logs sejam incluídos na imagem, tornando-a mais leve e o processo de build mais rápido.

## Passo 3: Estrutura do Projeto

Certifique-se que seu projeto tenha uma estrutura similar a esta:

```text showLineNumbers 
projeto/
├── cmd/
│   └── cli/
│       └── main.go
├── internal/
│   └── (seus pacotes internos)
├── pkg/
│   └── (seus pacotes públicos)
├── Dockerfile
└── .dockerignore
```

## Passo 4: Construindo a Imagem Docker

Para construir a imagem, execute o seguinte comando no diretório raiz do projeto:

```bash showLineNumbers
docker build -t my-awesome-service:latest .
```

## Passo 5: Verificando a Imagem Criada

Verifique se a imagem foi criada corretamente:

```bash showLineNumbers
docker images | grep my-awesome-service
```

## Passo 6: Executando o Container

Para executar sua API em um container, use:

```bash showLineNumbers
docker run --rm \
  -e ENVIRONMENT=production \
  -e APP_TYPE=service \
  -e APP_NAME=my-awesome-project \
  -e CLOUD=gcp \
  -e PORT=8080 \
  -p 8080:8080 \
  my-awesome-service:latest
```

Este comando define algumas variáveis de ambiente necessários para iniciar o `colibri-sdk-go` corretamente e mapeia a porta 8080 do container para a porta 8080 do host, permitindo acesso à API.

## Passo 7: Configuração para Ambientes de Produção

Para ambientes de produção, considere:
- Definir variáveis de ambiente para configuração da aplicação
- Implementar health checks
- Utilizar Kubernetes para orquestração

## Conclusão

Agora você tem uma imagem Docker otimizada para sua API web! 

Esta abordagem usando multi-stage build garante que a imagem final seja pequena e segura, contendo apenas os componentes necessários para executar a aplicação.

O uso do como imagem base para o segundo estágio proporciona uma superfície de ataque reduzida, pois esta imagem contém apenas o essencial para executar a aplicação, sem shell ou utilitários adicionais que poderiam ser explorados por atacantes. `distroless`

___
