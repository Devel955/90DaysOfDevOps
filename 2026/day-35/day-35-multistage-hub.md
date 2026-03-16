Day 35 – Multi-Stage Builds & Docker Hub
Goal: Build optimized Docker images using multi-stage builds and distribute them via Docker Hub.

📁 Project Structure:
day-35/
├── app/
│   ├── app.py                # Simple Python app
│   ├── Dockerfile.single     # Task 1 – Single stage build
│   ├── Dockerfile.multi      # Task 2 – Multi-stage build
│   └── Dockerfile.best       # Task 5 – Best practices
└── day-35-multistage-hub.md  # This file

Task 1 – The Problem with Large Images
App
A simple Python "Hello World" app used to demonstrate image size differences.
print("Hello from Docker!")
print("This is a single-stage build.")

Dockerfile.single:
FROM python:3.12
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]

Build & Size:
docker build -f app/Dockerfile.single -t hello-single app/
docker images hello-single

Image         Size
hello-single1.11 GB ❌

Why so large?
The python:3.12 base image includes the full Python compiler, pip, build tools, and system libraries — most of which are only needed to build the app, not to run it.
Task 2 – Multi-Stage Build
Dockerfile.multi
# Stage 1 — Builder (full Python)
FROM python:3.12 AS builder
WORKDIR /app
COPY app.py .

# Stage 2 — Final (minimal image)
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/app.py .
CMD ["python", "app.py"]

Build & Size:
docker build -f app/Dockerfile.multi -t hello-multi app/
docker images hello-multi

Image         Size
hello-single 1.11 GB ❌
hello-multi  119 MB ✅

📉 89% size reduction!:
Why is multi-stage smaller?
Stage        What happens
Stage 1     (builder)Full Python installs dependencies and builds the app
Stage 2     (final)Only copies the built app.py — nothing else
All build tools, compiler, pip cache, and unused libraries are left behind in Stage 1 and never included in the final image.

Task 3 – Push to Docker Hub
Login:
docker login
# Enter Docker Hub username and password
Tag the image:
docker tag hello-multi anan623/hello-multi:v1
Push:
docker push anan623/hello-multi:v1

Verify — Pull it back
bashdocker pull anan623/hello-multi:v1
docker run anan623/hello-multi:v1
🔗 Docker Hub Repository
https://hub.docker.com/r/anan623/hello-multi


Task 4 – Docker Hub Repository
Versioning with Tags
bash# Push a new version
docker tag hello-multi anan623/hello-multi:v2
docker push anan623/hello-multi:v2

# Pull specific version
docker pull anan623/hello-multi:v1   # pulls v1 only
docker pull anan623/hello-multi:v2   # pulls v2 only

# Pull latest
docker pull anan623/hello-multi      # pulls latest tag
Key Concepts
TagMeaningv1, v2Specific versions — stable, reproduciblelatestMost recent push — can change anytime
✅ Best practice: Always use specific tags in production (v1, v2). Never rely on latest in production.


Task 5 – Image Best Practices
Dockerfile.best
dockerfile# Best practices Dockerfile
FROM python:3.12-slim
WORKDIR /app
# Non-root user for security
RUN adduser --disabled-password --gecos "" appuser
COPY app.py .
# Never run as root
USER appuser

CMD ["python", "app.py"]
Build
bashdocker build -f app/Dockerfile.best -t hello-best app/
Best Practices Applied
PracticeWhypython:3.12-slim instead of python:3.1210x smaller base imageUSER appuser (non-root)Security — limits damage if container is compromisedSpecific tag python:3.12 not python:latestReproducible builds — latest can break anytimeCOPY only what's neededSmaller image, faster builds

📊 Final Size Comparison
ImageBase ImageSizeNoteshello-singlepython:3.121.11 GB ❌Full build tools includedhello-multipython:3.12-slim119 MB ✅Multi-stage, only app copiedhello-bestpython:3.12-slim119 MB ✅Multi-stage + non-root user
Result: 89% size reduction from single-stage to multi-stage build.

🔑 Key Learnings
Single-stage builds are simple but include everything — build tools, compilers, caches. Result: huge images.
Multi-stage builds separate the build environment from the runtime environment. Only the final artifact is shipped.
Docker Hub is a public registry to store and share images. Tag with version numbers for proper release management.
Never run containers as root — always add a non-root user.
Use specific base image tags — python:3.12 not python:latest for reproducible builds.
slim > full for base images — python:3.12-slim is 90% smaller than python:3.12.

📋 All Commands Reference:
# Build images
docker build -f app/Dockerfile.single -t hello-single app/
docker build -f app/Dockerfile.multi  -t hello-multi  app/
docker build -f app/Dockerfile.best   -t hello-best   app/

# Check sizes
docker images | grep hello

# Tag and push to Docker Hub
docker tag hello-multi anan623/hello-multi:v1
docker push anan623/hello-multi:v1

# Pull and run from Docker Hub
docker pull anan623/hello-multi:v1
docker run anan623/hello-multi:v1

# Remove local image and pull fresh
docker rmi anan623/hello-multi:v1
docker pull anan623/hello-multi:v1

Day 35 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham




