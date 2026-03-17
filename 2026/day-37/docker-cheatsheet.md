# 🐳 Docker Cheat Sheet — Days 29–36

## 🔵 Container Commands
| Command | What it does |
|---------|-------------|
| `docker run nginx` | Run a container |
| `docker run -d nginx` | Run in background (detached) |
| `docker run -it ubuntu bash` | Run interactively |
| `docker run -p 8080:80 nginx` | Map host:container port |
| `docker run --name myapp nginx` | Give container a name |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker stop myapp` | Stop a container |
| `docker rm myapp` | Remove a stopped container |
| `docker rm -f myapp` | Force remove running container |
| `docker exec -it myapp bash` | Enter a running container |
| `docker logs myapp` | View container logs |
| `docker logs -f myapp` | Follow live logs |
| `docker inspect myapp` | Detailed container info |

---

## 🟢 Image Commands
| Command | What it does |
|---------|-------------|
| `docker build -t myapp:v1 .` | Build image from Dockerfile |
| `docker build -f Dockerfile.multi -t myapp .` | Build from specific Dockerfile |
| `docker images` | List all images |
| `docker pull python:3.12` | Pull image from Docker Hub |
| `docker push anan623/myapp:v1` | Push image to Docker Hub |
| `docker tag myapp anan623/myapp:v1` | Tag an image |
| `docker rmi myapp` | Remove an image |
| `docker image inspect myapp` | Detailed image info |
| `docker history myapp` | Show image layers |

---

## 🟡 Volume Commands
| Command | What it does |
|---------|-------------|
| `docker volume create mydata` | Create a named volume |
| `docker volume ls` | List all volumes |
| `docker volume inspect mydata` | Volume details |
| `docker volume rm mydata` | Remove a volume |
| `docker run -v mydata:/app/data nginx` | Mount named volume |
| `docker run -v $(pwd):/app nginx` | Bind mount current folder |

---

## 🟣 Network Commands
| Command | What it does |
|---------|-------------|
| `docker network create mynet` | Create custom network |
| `docker network ls` | List all networks |
| `docker network inspect mynet` | Network details |
| `docker network connect mynet myapp` | Connect container to network |
| `docker run --network mynet nginx` | Run on specific network |

---

## 🔴 Compose Commands
| Command | What it does |
|---------|-------------|
| `docker compose up` | Start all services |
| `docker compose up -d` | Start in background |
| `docker compose up --build` | Rebuild + start |
| `docker compose down` | Stop + remove containers |
| `docker compose down -v` | Also delete volumes |
| `docker compose ps` | List service status |
| `docker compose logs -f web` | Follow service logs |
| `docker compose exec web bash` | Enter a service container |
| `docker compose build` | Build all images |
| `docker compose restart web` | Restart one service |

---

## 🧹 Cleanup Commands
| Command | What it does |
|---------|-------------|
| `docker system df` | Show disk usage |
| `docker system prune` | Remove all unused resources |
| `docker system prune -a` | Remove everything unused |
| `docker container prune` | Remove stopped containers |
| `docker image prune` | Remove dangling images |
| `docker volume prune` | Remove unused volumes |

---

## 📄 Dockerfile Instructions
| Instruction | What it does |
|-------------|-------------|
| `FROM python:3.12-slim` | Base image |
| `WORKDIR /app` | Set working directory |
| `COPY . .` | Copy files into image |
| `ADD file.tar.gz /app` | Copy + auto-extract archives |
| `RUN pip install flask` | Run command during build |
| `EXPOSE 5000` | Document port (doesn't publish) |
| `ENV NAME=value` | Set environment variable |
| `CMD ["python", "app.py"]` | Default command (overridable) |
| `ENTRYPOINT ["python"]` | Fixed command (not overridable) |
| `USER appuser` | Run as non-root user |
| `VOLUME /data` | Create mount point |
| `ARG VERSION=1.0` | Build-time variable |

---

## 🔑 Key Concepts

### Image vs Container
- **Image** = Blueprint (read-only template)
- **Container** = Running instance of an image

### CMD vs ENTRYPOINT
- **CMD** = Default command, can be overridden at runtime
- **ENTRYPOINT** = Fixed command, always runs

### Volumes vs Bind Mounts
- **Named Volume** = Docker manages it, data persists
- **Bind Mount** = You control the path, links to host folder

### -p 8080:80 means
- `8080` = Host port (your machine)
- `80` = Container port
- Traffic on host:8080 → container:80

### docker compose down vs down -v
- `down` = Stop + remove containers + networks
- `down -v` = Also deletes named volumes (DATA LOST!)

### Multi-stage builds
- Stage 1 = Build the app (heavy tools)
- Stage 2 = Copy only the artifact (small + clean)
- Result = Much smaller final image

### How containers communicate
- Same custom network = use service name as hostname
- Example: web container reaches db as `db:5432`

---

## 🏆 Real Commands Used in Days 29–36

```bash
# Day 34 — 3-service stack
docker compose up --build
curl localhost:5000/health

# Day 35 — Multi-stage + Docker Hub
docker build -f Dockerfile.multi -t hello-multi app/
docker tag hello-multi anan623/hello-multi:v1
docker push anan623/hello-multi:v1

# Day 36 — Full project
docker compose down
docker rmi day-36-web
docker compose up -d
```

---

*Docker Revision — Day 37 | #90DaysOfDevOps | #TrainWithShubham*
