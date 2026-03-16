# Day 36 – Docker Project: Dockerize a Full Application

## App Choice: Task Manager
A full-stack Task Manager built with Python Flask, PostgreSQL, and Redis.
Chosen because it covers all 3 tiers — web, database, and cache.
## Project Structure
day-36/
├── app/
│   ├── app.py           # Flask application
│   ├── Dockerfile       # Multi-stage build
│   ├── requirements.txt # Python dependencies
│   └── .dockerignore    # Ignore unnecessary files
├── .env                 # Environment variables
├── init.sql             # Database schema + seed data
├── docker-compose.yml   # Full stack orchestration
└── README.md            # How to run the project

## Task 1: The App — Task Manager
### What it does
- Shows all tasks with status (Done/Pending)
- Beautiful dark UI with stats (Total, Completed, Pending)
- Auto-refreshes every 10 seconds
- PostgreSQL stores tasks permanently
- Redis tracks visit counter

### Routes
| Route | Method | Description |
|-------|--------|-------------|
| / | GET | Task Manager UI |
| /health | GET | Check all services |
| /tasks | GET | List all tasks |
| /tasks | POST | Add new task |

### App tested at
http://13.60.63.160:5000 ✅
## Task 2: Dockerfile
- Multi-stage build — builder + final stage
- python:3.12-slim base — small image
- Non-root user appuser — security
- Gunicorn — production WSGI server

## Task 3: Docker Compose
- 3 services: web, db, cache
- Named volume: pg_data
- Networks: backend + frontend
- .env file for config
- Healthchecks on DB + Cache
- depends_on: service_healthy

## Task 4: Docker Hub
- Image: anan623/task-manager:v1
- Link: https://hub.docker.com/r/anan623/task-manager

## Task 5: Fresh Flow Test
docker compose down
docker rmi day-36-web
docker compose up -d
Result: app:ok, db:ok, cache:ok ✅

## Challenges & Solutions
| Challenge | Solution |
|-----------|----------|
| App crash before DB ready | depends_on + service_healthy |
| Large image | Multi-stage + slim base |
| Security | Non-root user appuser |
| Data loss | Named volume pg_data |

## Final Image Size
| Image | Size |
|-------|------|
| day-36-web | ~150 MB ✅ |
| postgres:15-alpine | ~240 MB |
| redis:7-alpine | ~40 MB |

## Key Learnings
1. Multi-stage builds = small + secure images
2. .env files keep secrets safe
3. Healthchecks prevent startup race conditions
4. Named volumes = data persists after restart
5. Non-root user = production best practice
