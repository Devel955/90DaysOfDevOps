# Day 34 – Docker Compose: Real-World Multi-Container Apps

## Overview
Today I built a production-style 3-tier application stack using Docker Compose.
The stack includes a Flask web app, PostgreSQL database, and Redis cache — all
orchestrated with healthchecks, restart policies, named volumes, and custom networks.

## Project Structure
```
day-34/
├── web/
│   ├── app.py            # Flask application
│   ├── Dockerfile        # Custom image build
│   └── requirements.txt  # Python dependencies
├── docker-compose.yml    # Full stack orchestration
├── init.sql              # Database schema
└── day-34-compose-advanced.md
```

## Task 1: 3-Service Stack

Built a 3-container stack:
- **Web**: Python Flask app with `/`, `/health`, `/visits` endpoints
- **Database**: PostgreSQL 15
- **Cache**: Redis 7

The Flask app connects to both PostgreSQL and Redis. Visit counter is stored
in Redis and page shows live connection status of all services.

**Start the stack:**
```bash
docker compose up --build
```

**Test endpoints:**
```bash
curl http://localhost:5000/         # Hello message
curl http://localhost:5000/health   # Service status
curl http://localhost:5000/visits   # Redis visit counter
```

---

## Task 2: depends_on & Healthchecks

Added healthchecks to `db` and `cache` services:
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 10s
```

Used `condition: service_healthy` so web app waits for DB to be truly ready:
```yaml
depends_on:
  db:
    condition: service_healthy
  cache:
    condition: service_healthy
```

**Result:** Web container starts only after DB and Cache pass healthchecks.
```bash
docker compose ps
# db and cache show "healthy" before web starts
```

---

## Task 3: Restart Policies

### restart: always (used on db)
- Restarts container on any exit
- Does NOT restart on `docker stop` or `docker kill`

### restart: on-failure (used on web and cache)
- Restarts only when container exits with non-zero code
- Does not restart on clean stop

### restart: unless-stopped
- Always restarts EXCEPT when manually stopped by user

### Test performed:
```bash
docker kill day34_db    # Force kill DB
docker ps               # DB restarts automatically
```

### When to use each:
| Policy | Use Case |
|--------|----------|
| `always` | Critical services like DB, always need to be up |
| `on-failure` | Apps that should only restart on crashes |
| `unless-stopped` | Services you want full manual control over |

---

## Task 4: Custom Dockerfile in Compose

Instead of using a pre-built image, used `build:` directive:
```yaml
web:
  build: ./web
```

**Dockerfile highlights:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN apt-get update && apt-get install -y libpq5
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

**Rebuild with one command:**
```bash
docker compose up --build -d
```

---

## Task 5: Named Networks & Volumes

### Named Volumes
```yaml
volumes:
  pg_data:
    labels:
      com.day34.description: "PostgreSQL persistent data"
```
Data survives container restarts and re-creations.

### Named Networks
```yaml
networks:
  backend:
    driver: bridge   # DB, Cache, Web communicate here
  frontend:
    driver: bridge   # Web app exposed to outside
```

### Labels added to all services:
```yaml
labels:
  com.day34.service: "database"
  com.day34.tier: "data"
```

---

## Task 6: Scaling (Bonus)
```bash
docker compose up -d --scale web=3
```

**Error received:**
```
WARNING: The "web" service is using the custom container name "day34_web".
Docker requires each container to have a unique name.
```

### Why scaling fails:
1. **Fixed container_name** — Each container must have a unique name
2. **Port binding** — `5000:5000` can only bind to one container at a time.
   3 containers cannot all use port 5000 on the host.

### Production Solution:
Use a **Load Balancer** (Nginx or Traefik) in front of the web containers:
- Remove `container_name` from web service
- Remove fixed port mapping from web service
- Let Load Balancer distribute traffic across replicas

---

## Screenshots

### Docker 3-Tier Demo UI
![UI Screenshot](screenshots/ui.png)

### docker ps output
![Docker PS](screenshots/docker-ps.png)

---

## Key Learnings

1. `depends_on` with `condition: service_healthy` is much more reliable than
   just `condition: service_started`
2. Healthchecks are essential for production — without them, app can start
   before DB is ready and crash
3. Named volumes ensure data persistence across container lifecycles
4. Simple port-based scaling doesn't work — need a reverse proxy/load balancer
5. Labels help organize and filter containers in large deployments

---

## Commands Reference
```bash
# Start everything
docker compose up --build -d

# Check status
docker compose ps

# View logs
docker compose logs -f web

# Stop everything
docker compose down

# Stop and remove volumes
docker compose down -v

# Rebuild after code change
docker compose up --build -d

# Scale (requires load balancer setup)
docker compose up --scale web=3
```
