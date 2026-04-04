---
sidebar_position: 1
title: Docker Image
---

In this section, we will learn how to package your application using Docker containers. We will use the *multi-stage build* concept to ensure a final image that is lightweight, secure, and optimized for production.

## The `Dockerfile`

Create a file named `Dockerfile` in the root of your project with the following content:

```dockerfile showLineNumbers
# Stage 1: Compilation
FROM golang:1.24-alpine3.21 AS build

WORKDIR /build

# Copy dependency files first to leverage Docker cache
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the source code
COPY . .

# Run unit tests
RUN go test ./... -v -failfast || exit 1

# Compile the static binary
RUN CGO_ENABLED=0 GOOS=linux go build -o app-bin ./cmd/main.go

# Stage 2: Final Image (Runtime)
FROM gcr.io/distroless/static

WORKDIR /app

# Copy only the compiled binary from the previous stage
COPY --from=build /build/app-bin /app/

# Define the application entry point
ENTRYPOINT ["/app/app-bin"]
```

### Why use *Multi-stage Build*?

1.  **Security**: The final image does not contain source code, compilers, or build tools, reducing the attack surface.
2.  **Size**: We use the `distroless/static` image in the final stage, which is extremely small (contains only the essentials to run Go binaries), resulting in images of just a few MBs.
3.  **Efficiency**: Docker caches layers. By copying `go.mod` and `go.sum` before the rest of the code, dependencies will only be downloaded again if these files change.

## The `.dockerignore` file

To prevent unnecessary files (such as logs, local binaries, or the `.git` folder) from being sent to the Docker build context, create a `.dockerignore` file:

```text
.git
*.log
bin/
vendor/
.env
```

## Building the image

Run the command below in your project's root directory:

```shell
docker build -t my-microservice:latest .
```

## Running the container

To start your application in a container, passing the necessary environment variables for `colibri-sdk-go`:

```shell showLineNumbers
docker run --rm \
  -e ENVIRONMENT=production \
  -e APP_NAME=my-microservice \
  -e APP_TYPE=service \
  -e CLOUD=none \
  -e PORT=8080 \
  -p 8080:8080 \
  my-microservice:latest
```

### Parameter explanation:
*   `--rm`: Automatically removes the container after it exits.
*   `-e`: Defines environment variables.
*   `-p 8080:8080`: Maps port 8080 of your computer to port 8080 of the container.

## Conclusion

Using this approach, you ensure that your Go application runs consistently in any environment (development, staging, or production), with maximum performance and security.

---
