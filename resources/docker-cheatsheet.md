# Docker Cheatsheet

A quick-reference guide for Docker commands used in DevOps workflows.

---

## Image Commands

```bash
# Pull images
docker pull nginx                          # Pull latest tag
docker pull nginx:1.25-alpine              # Pull specific tag

# Build images
docker build -t myapp:1.0 .               # Build from Dockerfile in current dir
docker build -t myapp:1.0 -f Dockerfile.prod .  # Specify Dockerfile
docker build --no-cache -t myapp:1.0 .    # Build without cache

# Tag images
docker tag myapp:1.0 myregistry/myapp:1.0  # Tag for a registry
docker tag myapp:1.0 myapp:latest          # Add latest tag

# Login / Logout
docker login                          # Login to Docker Hub (prompts for username/password)
docker login registry.example.com    # Login to a private registry
docker logout                         # Logout when done

# Push images
docker push myregistry/myapp:1.0           # Push to registry

# List and remove images
docker images                              # List all local images
docker images -a                           # Include intermediate images
docker rmi nginx:latest                    # Remove an image
docker rmi $(docker images -q)             # Remove all images
```

---

## Container Commands

```bash
# Run containers
docker run nginx                                      # Run in foreground
docker run -d nginx                                   # Run in background (detached)
docker run -d -p 8080:80 --name webserver nginx       # Named, port-mapped container
docker run -it ubuntu:22.04 bash                      # Interactive shell
docker run --rm ubuntu:22.04 echo "hello"             # Remove after exit

# List containers
docker ps                                             # Running containers
docker ps -a                                          # All containers (including stopped)
docker ps -q                                          # Only container IDs

# Start, stop, restart
docker stop webserver                                 # Graceful stop (SIGTERM)
docker stop $(docker ps -q)                           # Stop all running containers
docker kill webserver                                 # Force stop (SIGKILL)
docker start webserver                                # Start a stopped container
docker restart webserver                              # Restart container

# Remove containers
docker rm webserver                                   # Remove stopped container
docker rm -f webserver                                # Force remove running container
# WARNING: This removes ALL containers including running ones. Use with caution.
# Safer alternative: docker container prune (only removes stopped containers)
docker rm $(docker ps -aq)                            # Remove all containers

# Execute commands inside running container
docker exec -it webserver bash                        # Interactive shell in container
docker exec webserver ls /etc/nginx                   # Run single command
docker exec -u root webserver bash                    # Run as specific user

# Logs
docker logs webserver                                 # View all logs
docker logs -f webserver                              # Follow logs (tail)
docker logs --tail 100 webserver                      # Last 100 lines
docker logs --since 1h webserver                      # Logs from last 1 hour
docker logs --timestamps webserver                    # Include timestamps

# Inspect
docker inspect webserver                              # Full container metadata (JSON)
docker inspect --format '{{.NetworkSettings.IPAddress}}' webserver  # Extract field
```

---

## Common docker run Flags

| Flag                      | Description                                              | Example                              |
|---------------------------|----------------------------------------------------------|--------------------------------------|
| `-d`                      | Detached (background) mode                               | `docker run -d nginx`                |
| `-p host:container`       | Map host port to container port                          | `-p 8080:80`                         |
| `-v host:container`       | Mount host path as volume                                | `-v /data:/app/data`                 |
| `-v name:container`       | Mount named volume                                       | `-v mydata:/app/data`                |
| `--name`                  | Assign a name to the container                           | `--name webserver`                   |
| `-e KEY=VALUE`            | Set environment variable                                 | `-e DB_HOST=localhost`               |
| `--env-file`              | Load environment variables from a file                   | `--env-file .env`                    |
| `--network`               | Connect to a specific network                            | `--network mynet`                    |
| `--restart`               | Restart policy                                           | `--restart unless-stopped`           |
| `--rm`                    | Remove container when it exits                           | `docker run --rm ubuntu echo hi`     |
| `-it`                     | Interactive + TTY (for shells)                           | `-it ubuntu bash`                    |
| `-u user`                 | Run as specific user                                     | `-u 1000:1000`                       |
| `--memory`                | Memory limit                                             | `--memory 512m`                      |
| `--cpus`                  | CPU limit                                                | `--cpus 1.5`                         |
| `--hostname`              | Set container hostname                                   | `--hostname mycontainer`             |

### Restart Policies

| Policy              | Behavior                                              |
|---------------------|-------------------------------------------------------|
| `no`                | Never restart (default)                               |
| `on-failure`        | Restart only on non-zero exit code                    |
| `always`            | Always restart, even if manually stopped              |
| `unless-stopped`    | Always restart unless explicitly stopped by user      |

---

## Dockerfile Instructions Reference

```dockerfile
# Base image — always the first instruction
FROM ubuntu:22.04
FROM node:20-alpine AS builder        # Multi-stage build alias

# Set working directory (creates it if it doesn't exist)
WORKDIR /app

# Copy files from build context into image
COPY . .                              # Copy everything
COPY package*.json ./                 # Copy specific files
COPY --from=builder /app/dist ./dist  # Copy from another stage

# Run commands during build (creates a new layer)
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*    # Clean up in same layer

# Set environment variables (available at build and runtime)
ENV NODE_ENV=production
ENV PORT=3000

# Build-time variables (not available at runtime unless set via ENV)
ARG VERSION=1.0
ARG BUILD_DATE

# Expose port (documentation only — does not publish)
EXPOSE 8080

# Default command — overridden by docker run arguments
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]

# Entry point — always runs, CMD becomes default args
ENTRYPOINT ["python3", "app.py"]
ENTRYPOINT ["/docker-entrypoint.sh"]  # Common pattern with shell scripts

# Set user for subsequent RUN, CMD, ENTRYPOINT instructions
USER node
USER 1001:1001

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Add metadata labels
LABEL maintainer="team@example.com"
LABEL version="1.0"

# Mount point hint (documentation + anonymous volume)
VOLUME ["/data"]
```

### CMD vs ENTRYPOINT

| Aspect             | CMD                             | ENTRYPOINT                        |
|--------------------|---------------------------------|-----------------------------------|
| Purpose            | Default command / arguments     | Fixed executable that always runs |
| Overridden by      | `docker run image <new-cmd>`    | `docker run --entrypoint`         |
| With each other    | CMD becomes default args to ENTRYPOINT | Combine for flexible defaults |

```dockerfile
# Example: ENTRYPOINT + CMD combination
ENTRYPOINT ["python3", "app.py"]
CMD ["--port", "8080"]
# docker run myapp               -> python3 app.py --port 8080
# docker run myapp --port 9090   -> python3 app.py --port 9090
```

---

## Networking Commands

```bash
# List networks
docker network ls

# Create a network
docker network create mynet
docker network create --driver bridge mynet
docker network create --subnet 172.20.0.0/16 mynet

# Connect containers
docker network connect mynet webserver
docker network disconnect mynet webserver

# Run container on a network
docker run -d --network mynet --name app myimage

# Inspect network
docker network inspect mynet

# Remove network
docker network rm mynet

# Container DNS: containers on the same bridge network can reach each other by name
# Example: app container can connect to db container at hostname "db"
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name app myapp   # can reach db at hostname "db"
```

---

## Volume Commands

```bash
# List volumes
docker volume ls

# Create a named volume
docker volume create mydata

# Inspect a volume
docker volume inspect mydata

# Mount volumes
docker run -d -v mydata:/app/data myimage          # Named volume
docker run -d -v /host/path:/container/path myimage # Bind mount
docker run -d -v /host/path:/container/path:ro myimage  # Read-only

# Remove volumes
docker volume rm mydata
docker volume prune                                # Remove all unused volumes
```

---

## Docker Compose Commands

```bash
# Start services
docker compose up                                  # Start and attach
docker compose up -d                               # Start in background
docker compose up --build                          # Rebuild images before starting

# Stop services
docker compose down                                # Stop and remove containers
docker compose down -v                             # Also remove volumes
docker compose stop                                # Stop without removing

# View status and logs
docker compose ps                                  # List services
docker compose logs                                # All service logs
docker compose logs -f app                         # Follow logs for one service
docker compose logs --tail 50

# Run commands
docker compose exec app bash                       # Shell in running service
docker compose run --rm app python manage.py migrate  # One-off command

# Scale services
docker compose up -d --scale worker=3             # Run 3 instances of worker

# Build only
docker compose build
docker compose build --no-cache app

# Pull images
docker compose pull
```

### docker-compose.yml Quick Reference

```yaml
version: "3.9"

services:
  app:
    build: .
    image: myapp:1.0
    container_name: myapp
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    env_file:
      - .env
    volumes:
      - ./src:/app/src
      - uploads:/app/uploads
    depends_on:
      - db
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  pgdata:
  uploads:

networks:
  backend:
```

---

## Cleanup Commands

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune                    # Dangling images only
docker image prune -a                 # All unused images

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Remove everything unused at once
docker system prune
docker system prune -a                # Include all unused images
docker system prune -a --volumes      # Also remove volumes (destructive!)

# Check disk usage
docker system df                      # Summary of disk usage
docker system df -v                   # Verbose with details
```

---

## Debugging Cheatsheet

```bash
# Container keeps crashing — check exit code and logs
docker ps -a                                       # See exit status
docker logs --tail 100 mycontainer                 # Last 100 lines of logs
docker logs --since 5m mycontainer                 # Logs from last 5 minutes

# Get a shell in a running container
docker exec -it mycontainer bash
docker exec -it mycontainer sh                     # If bash is not installed

# Get a shell in a crashed container by overriding entrypoint
docker run -it --entrypoint bash myimage

# Inspect container configuration
docker inspect mycontainer                          # Full JSON metadata
docker inspect --format '{{.State.ExitCode}}' mycontainer
docker inspect --format '{{.HostConfig.Binds}}' mycontainer

# Real-time resource usage
docker stats                                        # All containers
docker stats mycontainer                            # Specific container

# Processes inside a container
docker top mycontainer                              # List processes

# Copy files between host and container
docker cp mycontainer:/app/app.log ./app.log        # Container to host
docker cp ./config.json mycontainer:/app/config.json # Host to container

# Check container network
docker exec mycontainer curl -s http://localhost:8080/health
docker exec mycontainer cat /etc/hosts
docker exec mycontainer env                         # List environment variables

# Build debug — check intermediate layer
docker build --target builder -t debug-image .      # Stop at specific stage
docker run -it debug-image sh                       # Inspect that stage
```
