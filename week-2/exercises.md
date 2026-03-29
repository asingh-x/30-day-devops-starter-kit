# Week 2 Mini Project — Containerize a Python Web App

## Objective

Build, containerize, and publish a simple Python web application that returns the server hostname. This simulates a real-world pattern used to verify load balancing.

---

## Application Code

Create this Python app (`app.py`):

```python
from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route("/")
def home():
    return {
        "hostname": socket.gethostname(),
        "env": os.environ.get("APP_ENV", "unknown"),
        "version": os.environ.get("APP_VERSION", "1.0.0")
    }

@app.route("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

`requirements.txt`:
```
flask==3.0.0
gunicorn==21.2.0
```

---

## Tasks

### Task 1 — Write the Dockerfile

Requirements:
- Use `python:3.11-slim` as the base
- Install dependencies before copying source code (for caching)
- Run as a non-root user
- Use gunicorn (not the Flask dev server) for the `CMD`
- Expose port 5000
- Add a health check

Gunicorn command: `gunicorn --bind 0.0.0.0:5000 app:app`

### Task 2 — Build and Test Locally

```bash
docker build -t hostname-app:latest .
docker run -d -p 5000:5000 --name hostname-app \
  -e APP_ENV=development \
  hostname-app:latest

curl http://localhost:5000/
curl http://localhost:5000/health
```

Expected output:
```json
{"hostname": "a1b2c3d4e5f6", "env": "development", "version": "1.0.0"}
```

### Task 3 — Run Multiple Instances

Run 3 containers and notice each returns a different hostname:

```bash
docker run -d -p 5001:5000 --name app1 hostname-app
docker run -d -p 5002:5000 --name app2 hostname-app
docker run -d -p 5003:5000 --name app3 hostname-app

curl http://localhost:5001/
curl http://localhost:5002/
curl http://localhost:5003/
```

### Task 4 — Add nginx Load Balancer with Docker Compose

Create a `docker-compose.yml` with:
- 3 app instances
- nginx as a reverse proxy / load balancer on port 80
- Custom bridge network

`nginx.conf`:
```nginx
upstream app {
    server app1:5000;
    server app2:5000;
    server app3:5000;
}

server {
    listen 80;
    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Run `curl http://localhost/` multiple times — you should see different hostnames (round-robin load balancing).

### Task 5 — Push to Docker Hub

```bash
docker tag hostname-app:latest yourusername/hostname-app:v1.0.0
docker push yourusername/hostname-app:v1.0.0
```

---

## Checklist

- [ ] Dockerfile uses non-root user
- [ ] Dependencies installed before source code is copied
- [ ] Health check added
- [ ] App works correctly with `curl`
- [ ] All 3 instances return different hostnames
- [ ] Docker Compose file works with `docker compose up`
- [ ] Image pushed to Docker Hub
- [ ] `.dockerignore` created
- [ ] Code committed to GitHub with proper PR

---

## Bonus

- [ ] Add a `/metrics` endpoint that returns request count (store in memory)
- [ ] Write a `docker-compose.prod.yml` that uses `gunicorn` with 4 workers
- [ ] Tag your Docker image with the Git commit SHA: `docker build -t app:$(git rev-parse --short HEAD) .`
