# Day 8 — Writing Dockerfiles

## What is a Dockerfile?

A Dockerfile is a text file with instructions to build a Docker image. Each instruction creates a new layer.

```dockerfile
# Example: Python Flask app
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

Build and run:
```bash
docker build -t myapp:latest .
docker run -d -p 5000:5000 myapp:latest
```

---

## Dockerfile Instructions

### FROM — Base Image

Every Dockerfile starts with `FROM`. This is the starting point.

```dockerfile
FROM ubuntu:22.04
FROM python:3.11-slim          # Slim = smaller, fewer pre-installed packages
FROM node:20-alpine            # Alpine = even smaller (~5MB base)
FROM scratch                   # Empty image — for statically compiled binaries
```

Choose the smallest base image that meets your needs.

### WORKDIR — Working Directory

Sets the working directory for all subsequent instructions.

```dockerfile
WORKDIR /app                   # Creates /app if it doesn't exist
```

Always use `WORKDIR` instead of `RUN cd /app`. It's cleaner and explicit.

### COPY vs ADD

```dockerfile
COPY requirements.txt .        # Copy file to current WORKDIR
COPY src/ ./src/               # Copy directory
COPY . .                       # Copy everything (filtered by .dockerignore)

ADD archive.tar.gz /app/       # ADD can auto-extract archives
ADD https://... /app/file      # ADD can download URLs (avoid this — use RUN curl)
```

**Rule:** Use `COPY` unless you specifically need `ADD`'s archive extraction.

### RUN — Execute Commands

```dockerfile
RUN apt update && apt install -y curl nginx \
    && rm -rf /var/lib/apt/lists/*      # Clean up in same layer!

RUN pip install --no-cache-dir -r requirements.txt

RUN npm ci --omit=dev
```

**Important:** Each `RUN` creates a new layer. Combine related commands with `&&` to keep layers small.

### ENV — Environment Variables

```dockerfile
ENV APP_ENV=production
ENV PORT=8080
ENV DATABASE_URL=postgresql://localhost/mydb

# Reference in later instructions
WORKDIR /app/$APP_ENV
```

### ARG — Build-time Variables

Unlike `ENV`, `ARG` is only available during the build — not in the running container.

```dockerfile
ARG APP_VERSION=1.0.0
RUN echo "Building version $APP_VERSION"

# Pass at build time
docker build --build-arg APP_VERSION=2.0.0 .
```

### EXPOSE — Document Ports

```dockerfile
EXPOSE 8080          # Documents that the app listens on 8080
EXPOSE 5432          # Does NOT actually open the port — just documentation
```

You still need `-p 8080:8080` when running. `EXPOSE` is documentation only.

### CMD vs ENTRYPOINT

Both define what runs when the container starts. Understanding the difference matters.

**CMD** — default command, can be overridden:
```dockerfile
CMD ["python", "app.py"]

# Can be overridden at runtime:
docker run myapp python other_script.py
```

**ENTRYPOINT** — always runs, arguments are appended:
```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]              # Default argument to entrypoint

docker run myapp            # runs: python app.py
docker run myapp other.py   # runs: python other.py
```

**Shell form vs Exec form:**
```dockerfile
# Shell form — runs in /bin/sh -c, signals not passed correctly
CMD python app.py

# Exec form — recommended, PID 1, signals handled correctly
CMD ["python", "app.py"]
```

Always use **exec form** (`["command", "arg"]`) for CMD and ENTRYPOINT.

### USER — Run as Non-Root

```dockerfile
RUN useradd --system --no-create-home appuser
USER appuser
```

Never run containers as root in production. It's a major security risk.

### HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
# Note: curl must be installed in the image. Add 'RUN apt-get install -y curl' or use wget as an alternative.
```

Docker will mark the container as unhealthy if the check fails repeatedly.

---

## Layer Caching — The Most Important Optimization

Docker caches each layer. If nothing in a layer changed, Docker reuses the cached version.

**Bad — busts cache every time code changes:**
```dockerfile
FROM python:3.11-slim
COPY . .                               # Cache busted if ANY file changes
RUN pip install -r requirements.txt    # Re-runs every time!
```

**Good — dependencies cached separately from code:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .               # Only invalidated if requirements change
RUN pip install -r requirements.txt   # Cached unless requirements.txt changes
COPY . .                              # Copy code last
```

**Rule:** Copy files that change rarely (requirements, package.json) before files that change often (your source code).

---

## .dockerignore

Like `.gitignore` but for Docker builds. Prevents unnecessary files from being sent to the Docker daemon.

```dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env
.env.local
*.log
dist/
build/
README.md
.DS_Store
```

Without `.dockerignore`, `COPY . .` sends your entire directory — including `node_modules` (hundreds of MB).

---

## Multi-Stage Builds

Build in one stage, copy only the final artifact to a clean image.

**Example: Go application**

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Stage 2: Run (tiny final image)
FROM alpine:3.19
# Pin to a specific version in production; latest works for learning
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

Result: the final image is ~10MB instead of ~800MB (the Go toolchain is not included).

**Example: Node.js application**

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Real Examples

### Python Flask App

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --system --no-create-home appuser
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s CMD curl -f http://localhost:5000/health || exit 1
# Note: curl must be installed in the image. Add 'RUN apt-get install -y curl' or use wget as an alternative.

CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

### Node.js App

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

CMD ["node", "server.js"]
```

---

## Exercises

1. Write a Dockerfile for a Python script that prints "Hello from Docker" and exits.
2. Write a Dockerfile for a Flask app that returns the hostname on `/`. Test it locally.
3. Add a `.dockerignore` to your project. What did you exclude and why?
4. Optimize a Dockerfile with bad layer ordering — move `COPY requirements.txt` before `COPY . .`.
5. Write a multi-stage Dockerfile for a Go or Node.js app. Compare the final image size with a single-stage build.
6. Add a `HEALTHCHECK` to your Flask app Dockerfile.

---

## Key Takeaways

- Order matters: copy rarely-changed files before frequently-changed files
- Always use exec form for CMD/ENTRYPOINT: `["command", "arg"]`
- Never run as root — add a non-root user
- Use `.dockerignore` to speed up builds
- Multi-stage builds dramatically reduce final image size
- `EXPOSE` is documentation — you still need `-p` to actually expose ports
