Task 1 – Docker Images
Pull Docker Images
docker pull nginx
docker pull ubuntu
docker pull alpine
These commands download images from Docker Hub to the local system.

List All Images
docker images
Example Output
| Repository | Tag    | Size   |
| ---------- | ------ | ------ |
| nginx      | latest | ~187MB |
| ubuntu     | latest | ~72MB  |
| alpine     | latest | ~7MB   |
Observation
Nginx contains a web server so the size is larger.
Ubuntu includes a full Linux environment.
Alpine is a minimal Linux distribution designed for containers.

Ubuntu vs Alpine
| Feature         | Ubuntu             | Alpine                 |
| --------------- | ------------------ | ---------------------- |
| Size            | ~70MB              | ~7MB                   |
| Package Manager | apt                | apk                    |
| Use Case        | General containers | Lightweight containers |
Why Alpine is smaller
Minimal packages
BusyBox utilities
Designed specifically for containers
Fewer dependencies

Inspect an Image
docker image inspect nginx
Information shown:
Image ID
Creation date
Environment variables
Architecture
Layers
Default command

Remove an Image
docker rmi alpine
This removes an unused image.

Task 2 – Image Layers
View Image Layers
docker image history nginx
| Layer      | Command              | Size |
| ---------- | -------------------- | ---- |
| Layer 1    | CMD                  | 0B   |
| Layer 2    | EXPOSE               | 0B   |
| Layer 3    | COPY                 | 2KB  |
| Layer 4    | RUN install packages | 40MB |
| Base Layer | FROM ubuntu          | 70MB |
Observation
Each line represents a layer.
Some layers show 0B because they only store metadata.

What Are Docker Layers?
Docker images are built from multiple read-only layers.
Each command in a Dockerfile creates a new layer.
Example Dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install nginx
COPY index.html /var/www/html

Why Docker Uses Layer
Caching – Faster image builds
storage efficiency – Images share common layers
Faster downloads – Only missing layers are downloaded

Task 3 – Container Lifecycle
Docker containers move through several states.
Create Container
docker create --name mycontainer nginx

State → Created
Start Container
docker start mycontainer
State → Running
Pause Container
docker pause mycontainer
State → Paused
Unpause Container
docker unpause mycontainer
State → Running
Stop Container
docker stop mycontainer

State → Exited
Restart Container
docker restart mycontainer
Kill Container
docker kill mycontainer
Stops container immediately.

Remove Container
docker rm mycontainer
Check Container Status
docker ps -a
Shows container states.

Task 4 – Working With Running Containers
Run Container in Detached Mode
docker run -d --name nginx-container -p 8080:80 nginx
-d → detached mode
-p → port mapping

View Logs
docker logs nginx-container
Real-Time Logs
docker logs -f nginx-container
Exec Into Container
docker exec -it nginx-container /bin/bash
Explore filesystem:
ls
cd /usr/share/nginx/html
Exit:
exit
Run Command Without Entering Container
docker exec nginx-container ls /
Inspect Container
docker inspect nginx-container

Information available:
Container IP address
Port mappings
Mount points
Network settings

Task 5 – Cleanup
Stop All Running Containers
docker stop $(docker ps -q)
Remove All Stopped Containers
docker container prune
Remove Unused Images
docker image prune -a
Check Docker Disk Usage
docker system df
Hints
docker image history
docker create
docker logs -f
docker inspect
docker system df
docker system prune

Key Learnings
Docker images are templates used to create containers.
Images consist of multiple layers for caching and efficiency.
Containers follow a lifecycle: Created → Running → Paused → Stopped → Removed.
Docker provides powerful commands to inspect, manage, and debug containers.
Cleaning unused resources helps save disk space.
