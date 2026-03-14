Day 31 – Dockerfile: Build Your Own Images
Objective
Learn how to create custom Docker images using Dockerfiles and understand Docker build optimization.
Task 1 – Your First Dockerfile
Dockerfile
FROM ubuntu:latest
RUN apt update && apt install -y curl
CMD ["echo", "Hello from my custom image!"]
Build Image
docker build -t my-ubuntu:v1 .
Run Container
docker run my-ubuntu:v1
Output
Hello from my custom image!

Task 2 – Dockerfile Instructions
This Dockerfile demonstrates common instructions.
FROM python:3.14
WORKDIR /app
COPY app.py /app
RUN pip install flask
EXPOSE 8000
CMD ["python", "app.py"]
Explanation:
| Instruction | Purpose                       |
| ----------- | ----------------------------- |
| FROM        | Base image                    |
| RUN         | Execute command during build  |
| COPY        | Copy files from host to image |
| WORKDIR     | Set working directory         |
| EXPOSE      | Document container port       |
| CMD         | Default command               |

Task 3 – CMD vs ENTRYPOINT
CMD Example:
Run:
docker run cmd-demo:v1

Output:
hello
Override command:
docker run cmd-demo:v1 echo "Anand"

Output:
Anand
ENTRYPOINT Example
FROM ubuntu:latest
ENTRYPOINT ["echo"]
Run:
docker run entrypoint-demo:v1 hello

Output:
hello
With arguments:
docker run entrypoint-demo:v1 hello Anand

Output:
hello Anand
Difference:
| Feature   | CMD             | ENTRYPOINT        |
| --------- | --------------- | ----------------- |
| Purpose   | Default command | Fixed command     |
| Override  | Yes             | No                |
| Arguments | Replace command | Append to command |

Task 4 – Build a Simple Web App Image
index.html
<!DOCTYPE html>
<html>
<head>
<title>My Website</title>
</head>
<body>
<h1>Hello from Anand's Docker Website!</h1>
<p>This is my Day 31 Docker task.</p>
</body>
</html>

Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80

Build Image
docker build -t my-website:v1 .
Run Container
docker run -d -p 8080:80 --name my-website-container my-website:v1

Access Website
http://13.60.241.125:8080

Task 5 – .dockerignore
Create .dockerignore file.
node_modules
.git
*.md
.env

Purpose
.dockerignore prevents unnecessary files from being included in the Docker build context.
Benefits:
Faster builds
Smaller image size
Security of sensitive files

Task 6 – Build Optimization
Docker uses caching to speed up builds.

Example:
Step 1/3 : FROM nginx:alpine
 ---> Using cache
If a file changes:
COPY index.html /usr/share/nginx/html/index.html
 ---> Running
Why Layer Order Matters
Docker images are built in layers.
If an early layer changes, all subsequent layers must rebuild.

Best practice:
Put stable commands first
Put frequently changing files last

Example optimized Dockerfile:
FROM nginx:alpine
RUN apk update
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80

Key Learnings
How to create Dockerfiles
Common Dockerfile instructions
Difference between CMD and ENTRYPOINT
Building and running custom Docker images
Using .dockerignore
Docker build cache optimization

Conclusion
Day 31 focused on creating custom Docker images using Dockerfiles and optimizing the build process using caching and proper layer ordering.
