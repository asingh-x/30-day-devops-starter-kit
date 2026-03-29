# Day 9 — Docker Networking and Volumes

---

## Networking

By default, containers are isolated. Docker networking lets them communicate.

### Network Types

| Network | When to use |
|---------|-------------|
| `bridge` | Default. Containers on same host can communicate |
| `host` | Container shares host network. No isolation. Max performance |
| `none` | No networking. Fully isolated container |
| `overlay` | Multi-host networking (used in Docker Swarm/K8s) |

### Bridge Network (Default)

```bash
# Every container gets an IP on the default bridge (172.17.0.x)
docker run -d --name app nginx
docker inspect app | grep IPAddress
# "IPAddress": "172.17.0.2"
```

**Problem with default bridge:** containers cannot reach each other by name — only by IP.

### Custom Bridge Network (Recommended)

On a custom bridge, containers can reach each other by **container name**.

```bash
# Create a custom network
docker network create mynetwork

# Run containers on it
docker run -d --name web --network mynetwork nginx
docker run -d --name db --network mynetwork postgres

# Now 'web' can reach 'db' by name
docker exec web ping db         # Works!
docker exec web nc -zv db 5432    # Test TCP connectivity to the database port (databases don't speak HTTP, so curl won't work here)
```

**Always use custom networks** for containers that need to talk to each other.

### Network Commands

```bash
docker network ls                    # List networks
docker network create mynetwork      # Create a bridge network
docker network rm mynetwork          # Delete a network
docker network inspect mynetwork     # Inspect (see connected containers, subnet)
docker network connect mynetwork container1    # Connect running container to network
docker network disconnect mynetwork container1 # Disconnect
```

### Host Network

```bash
docker run -d --network host nginx
# nginx now listens directly on host port 80 — no port mapping needed
# No network isolation — use only when performance is critical
```

---

## Volumes — Persistent Storage

Containers are **stateless by default** — when you delete a container, all its data is gone.

Volumes solve this by storing data **outside the container**.

### Three Ways to Mount Storage

#### 1. Named Volumes (Recommended)

Managed by Docker. Data persists even after container is deleted.

```bash
# Create a named volume
docker volume create mydata

# Mount it
docker run -d \
  --name db \
  -v mydata:/var/lib/postgresql/data \
  postgres

# The data in /var/lib/postgresql/data is stored in mydata volume
# Delete and recreate the container — data is still there
docker rm -f db
docker run -d --name db -v mydata:/var/lib/postgresql/data postgres
```

Volume commands:
```bash
docker volume ls              # List volumes
docker volume inspect mydata  # See where data is stored on host
docker volume rm mydata       # Delete volume (and its data)
docker volume prune           # Delete all unused volumes
```

#### 2. Bind Mounts

Mount a specific directory from your host into the container. Changes in either direction are reflected immediately.

```bash
# Mount current directory into container
docker run -d \
  -v /home/ankit/myapp:/app \    # host:container
  -p 5000:5000 \
  myapp

# Great for development — edit code on host, changes visible in container
docker run -d \
  -v $(pwd):/app \               # $(pwd) = current directory
  -p 3000:3000 \
  node-dev
```

**Use case:** Development only. In production, use named volumes or bake files into the image.

#### 3. tmpfs (In-Memory)

Data stored in RAM — never written to disk. Gone when container stops.

```bash
docker run -d \
  --tmpfs /tmp \
  --tmpfs /run \
  nginx
```

Use for sensitive data that should never touch disk.

---

## Backup and Restore Volumes

```bash
# Backup: dump volume contents to a tar file
docker run --rm \
  -v mydata:/source \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /source .

# Restore
docker run --rm \
  -v mydata:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /target
```

---

## Common Patterns

### Stateful Service (Database)

```bash
docker run -d \
  --name postgres \
  --network mynetwork \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=appdb \
  --restart unless-stopped \
  postgres:16
```

### Stateless App (Web Server)

```bash
docker run -d \
  --name webapp \
  --network mynetwork \
  -p 8080:8080 \
  -e DATABASE_URL=postgresql://postgres:secret@postgres/appdb \
  --restart unless-stopped \
  myapp:latest
```

Now `webapp` can reach `postgres` by name, and postgres data survives container restarts.

---

## Exercises

1. Create a custom Docker network called `devnet`.
2. Run a PostgreSQL container on `devnet`. Run an `alpine` container on the same network and `ping` postgres by name.
3. Create a named volume `pgdata`. Run postgres with that volume. Create a table inside the database. Delete the container. Re-create it with the same volume — verify the table still exists.
4. Run nginx with a bind mount: mount a local `html/` directory to `/usr/share/nginx/html`. Edit a file on your host and reload the browser to see the change instantly.
5. List all volumes and networks on your machine. Remove ones you created for practice.

---

## Key Takeaways

- Use **custom bridge networks** so containers can find each other by name
- Use **named volumes** for data that must persist (databases, uploads)
- Use **bind mounts** during development (live code reload)
- `docker network create mynet` → run containers `--network mynet`
- Always use `--restart unless-stopped` for services that should survive reboots
