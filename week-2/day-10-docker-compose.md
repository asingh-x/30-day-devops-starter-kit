# Day 10 — Docker Compose

## The Problem

Running multi-container apps with `docker run` gets painful fast:

```bash
docker network create mynet
docker volume create pgdata
docker run -d --name db --network mynet -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres
docker run -d --name redis --network mynet redis:alpine
docker run -d --name app --network mynet -p 8080:8080 -e DB_HOST=db myapp
```

That's hard to share, hard to remember, and easy to mess up.

Docker Compose solves this with a single YAML file.

---

## The docker-compose.yml File

```yaml
version: "3.9"
# Note: In Docker Compose v2 (default since 2021), the 'version' key is optional.
# version: "3.9" still works but modern compose files can omit it entirely.

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://admin:${POSTGRES_PASSWORD}@db:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  pgdata:
```

```
# Create a .env file with your credentials (never commit this to git)
# POSTGRES_PASSWORD=your-strong-password-here
```

> Add `.env` to your `.gitignore` to prevent credentials being committed to git.

---

## Core Commands

```bash
# Start all services (in background)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services (containers remain)
docker compose stop

# Stop and remove containers, networks
docker compose down

# Stop and remove containers, networks, AND volumes
docker compose down -v

# View logs
docker compose logs               # All services
docker compose logs -f app        # Follow logs for 'app' service
docker compose logs --tail=50 db  # Last 50 lines for 'db'

# Status
docker compose ps

# Run a command in a service container
docker compose exec app bash
docker compose exec db psql -U admin appdb

# Scale a service
docker compose up -d --scale app=3
# Note: Scaling only works if the service has no fixed host port mapping.
# Remove or change host port binding (e.g., "8080:8080") to a range ("8080-8090:8080")
# before scaling, otherwise Docker will fail to bind duplicate ports.
```

---

## Service Configuration

### Build from Dockerfile

```yaml
services:
  app:
    build:
      context: .                  # Dockerfile location
      dockerfile: Dockerfile.prod # Custom Dockerfile name
      args:
        APP_VERSION: "1.0.0"
```

### Environment Variables

```yaml
services:
  app:
    environment:
      # Inline values
      - NODE_ENV=production
      - PORT=3000

      # From host environment
      - SECRET_KEY               # Takes value from your shell
      # Warning: If SECRET_KEY is not set in your shell, it will be passed as an empty string.
      # Use a .env file to define defaults safely.

    env_file:
      - .env                     # Load from file
```

### Health Checks

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

### depends_on with Health

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy   # Wait for db to be healthy

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### Networks

```yaml
services:
  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend           # db only on internal network

networks:
  frontend:
  backend:
    internal: true        # No internet access
```

---

## Real-World Example: Full Stack App

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - certbot_certs:/etc/letsencrypt
    depends_on:
      - app
    restart: unless-stopped

  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://admin:${DB_PASSWORD}@db:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
  certbot_certs:
```

`.env` file (never commit this):
```
DB_PASSWORD=mysecretpassword
```

---

## Development vs Production

```yaml
# docker-compose.yml (base)
services:
  app:
    build: .
    environment:
      - NODE_ENV=production

# docker-compose.dev.yml (overrides for development)
services:
  app:
    volumes:
      - .:/app           # Live code reload
    environment:
      - NODE_ENV=development
    command: npm run dev
```

```bash
# Run with override file
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

---

## Exercises

1. Create a `docker-compose.yml` that runs nginx on port 8080 and a Redis container. Both on a shared network.
2. Add a simple Python Flask app to the compose file that connects to Redis (use Flask + redis-py). The app should count page visits stored in Redis.
3. Add `depends_on` so the app waits for Redis to be ready. Add a health check to Redis.
4. Use an `.env` file for the Redis password and reference it in the compose file.
5. Practice: `docker compose up`, `docker compose logs -f`, `docker compose down -v`.

---

## Key Takeaways

- One YAML file replaces many `docker run` commands
- `docker compose up -d --build` is your main command
- Use `depends_on` with `condition: service_healthy` — not just `depends_on` alone
- Store secrets in `.env` files, never hardcode them in `docker-compose.yml`
- `docker compose down -v` removes everything including volumes — don't run in production without thinking
