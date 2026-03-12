Task 1: What is Docker?
1. What is a Container and Why Do We Need Them?
A container is a lightweight and isolated environment that packages an application along with its dependencies such as libraries, runtime, and configuration files.
Containers ensure that the application runs the same way in every environment such as development, testing, and production.

Why do we need containers?
Containers solve several common problems:
Environment consistency – Application runs the same everywhere
Fast deployment – Containers start in seconds
Portability – Can run on any system with Docker installed
Efficient resource usage – Uses fewer resources compared to virtual machines
Example:
A developer builds an app on their laptop. By using Docker containers, the same application can run without errors on a server or cloud platform.

2. Containers vs Virtual Machines
Containers and Virtual Machines are both used for virtualization, but they work differently.
| Feature        | Containers           | Virtual Machines       |
| -------------- | -------------------- | ---------------------- |
| OS             | Share host OS kernel | Each VM has its own OS |
| Size           | Small (MBs)          | Large (GBs)            |
| Startup Time   | Seconds              | Minutes                |
| Resource Usage | Lightweight          | Heavy                  |
| Performance    | Faster               | Slower                 |
Key Difference
Virtual Machines run a full operating system for each instance.
Containers share the host operating system kernel, making them much faster and lightweight.

3. Docker Architecture
Docker uses a client-server architecture to manage containers.
Main components of Docker architecture:
Docker Client
The Docker client is the interface used by users to interact with Docker.
Example commands:
docker run
docker pull
docker ps
hese commands communicate with the Docker daemon.

Docker Daemon
The Docker daemon (dockerd) runs in the background and manages Docker objects such as:
Images
Containers
Networks
Volumes
It listens for requests from the Docker client and performs the required actions.

Docker Images
A Docker image is a read-only template used to create containers.
Images contain:
Application code
Libraries
Dependencies
Configuration
Example image:
nginx
ubuntu
node

Docker Containers
A container is a running instance of a Docker image.
Example:
docker run nginx
This command creates and starts a container using the nginx image.
Docker Registry
A Docker registry is a place where Docker images are stored.
Popular registry:
Docker Hub
When you run a container, Docker pulls the image from the registry if it is not available locally.

4. Docker Architecture (My Explanation)
The Docker workflow works like this:
The developer runs a command using the Docker client (for example docker run nginx).
The Docker client sends the request to the Docker daemon.
The Docker daemon checks if the image exists locally.
if the image is not available, Docker pulls it from the Docker registry (Docker Hub).
The Docker daemon then creates and runs a container from that image.
Simple Flow
Developer
↓
Docker Client
↓
Docker Daemon
↓
Docker Registry (Docker Hub)
↓
Container Created and Running

Task 2: Install Docker
1. Install Docker on Ubuntu
First, update the system packages.
sudo apt update
Then install Docker using the following command:
sudo apt install docker.io -y
After installation, start the Docker service.
sudo systemctl start docker
Enable Docker to start automatically on system boot.
sudo systemctl enable docker

3. Verify Docker Installation
To verify that Docker is installed correctly, run the following command:
docker --version
Example output:
Docker version 24.x.x, build xxxx
This confirms that Docker has been successfully installed on the system.

3. Run the Hello-World Container
Run the following command:
sudo docker run hello-world
What happens when this command runs:
Docker checks if the hello-world image exists locally.
If it does not exist, Docker pulls the image from Docker Hub.
Docker creates a new container from the image.
The container runs a small program that prints a message.
Example output:
Hello from Docker!
This message shows that your installation appears to be working correctly.

4. What the Output Means
The output explains the following steps:
Docker client contacted the Docker daemon
The daemon pulled the hello-world image from Docker Hub
A new container was created
The container ran the program and displayed the message
After execution, the container stopped.
This confirms that Docker is installed and working properly.

Task 3: Run Real Containers
1. Run an Nginx Container
First pull and run the Nginx container.
docker run -d -p 8080:80 nginx
Explanation
docker run → Create and run a container
-d → Run container in detached mode (background)
-p 8080:80 → Map host port 8080 to container port 80
nginx → Docker image name
Access Nginx in Browser
Open browser and visit:
http://localhost:8080
If using a cloud server:
http://<server-ip>:8080
You should see the Nginx Welcome Page, which confirms that the container is running successfully.

2. Run an Ubuntu Container in Interactive Mode
Run Ubuntu container with terminal access.
docker run -it ubuntu
Explanation
-i → Interactive mode
-t → Allocate terminal
This opens a Linux shell inside the container.
Example commands you can try:
ls
pwd
whoami
apt update
To exit the container:
exit

3. List All Running Containers
To see all currently running containers:
docker ps
Example output shows:
Container ID
Image
Status
Ports
Container Name

4. List All Containers (Including Stopped Ones)
To see all containers including stopped ones:
docker ps -a
This command displays:
Running containers
Stopped containers
Container history

5. Stop a Container
To stop a running container:
docker stop <container_id>
Example:
docker stop 3f4a9c2b1d
Docker will stop the running container.

6. Remove a Container
To remove a container:
docker rm <container_id>
Example:
docker rm 3f4a9c2b1d
This deletes the container from the system.

Conclusion
In this task, we successfully:
Ran an Nginx container and accessed it through the browser
Ran an Ubuntu container in interactive mode to explore it like a Linux machine
Listed running containers using docker ps
Listed all containers using docker ps -a
Stopped and removed containers using docker stop and docker rm
These commands are essential for basic Docker container management.

Task 4: Explore Docker Containers
1. Run a Container in Detached Mode
Command:
docker run -d nginx
Explanation
-d flag container ko detached mode me run karta hai.
Detached mode ka matlab hai container background me run hota hai aur terminal free ho jata hai.
Difference
| Mode                     | Behavior                                    |
| ------------------------ | ------------------------------------------- |
| Interactive Mode (`-it`) | Terminal container ke andar attach hota hai |
| Detached Mode (`-d`)     | Container background me run karta hai       |

2. Give a Container a Custom Name
By default Docker container ko random name deta hai (jaise: crazy_yalow, nice_zhukovsky).
Custom name dene ke liye:
docker run -d --name mynginx nginx
Is command me:
--name mynginx → container ka custom name set karta hai.
Container check karne ke liye:
docker ps

3. Map a Port from Container to Host
Port mapping se container ka port host machine ke port se connect hota hai.
Command:
docker run -d -p 8080:80 nginx
Explanation:
8080 → Host port
80 → Container port
Ab browser me open kar sakte ho:
http://localhost:8080
Ya agar cloud server use kar rahe ho:
http://SERVER-IP:8080

4. Check Logs of a Running Container
Container logs dekhne ke liye command:
docker logs container_name
Example:
docker logs mynginx
Ye container ke application logs show karta hai.

5. Run a Command Inside a Running Container
Running container ke andar command run karne ke liye:
docker exec -it container_name bash
Isse container ke andar Linux shell open ho jata hai.
Example commands inside container:
ls
pwd
whoami
Container se exit karne ke liye:
exit

Conclusion
In this task we explored advanced Docker commands such as:
Running containers in detached mode
Giving containers custom names
Mapping ports between host and container
Viewing container logs
Executing commands inside running containers
These commands are commonly used for container management and debugging in DevOps environment.
