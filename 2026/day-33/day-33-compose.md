Day 33 – Docker Compose: Multi-Container Applications
Overview

Today I learned how to use Docker Compose to manage multi-container applications.
Docker Compose allows us to define and run multiple containers using a single configuration file called docker-compose.yml.
Instead of running many docker run commands manually, Compose simplifies application deployment with a single command.

Task 1 – Install & Verify Docker Compose
Command
docker compose version

Output:
Docker Compose version v2.x.x
This confirms Docker Compose is installed and working correctly.

Task 2 – First Compose File (Nginx)
Create Project Folder
mkdir compose-basics
cd compose-basics
docker-compose.yml
services:
  nginx:
    image: nginx
    container_name: nginx-compose
    ports:
      - "8080:80"

Run Container:
docker compose up -d

Task 3 – WordPress + MySQL Multi-Container Setup
In this task I created a multi-container application using Docker Compose with:
WordPress (application)
MySQL (database)
Both services run on the same default network created by Docker Compose.

docker-compose.yml:
services:
  db:
    image: mysql:5.7
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - mysql_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wordpress-app
    restart: always
    depends_on:
      - db
    ports:
      - "8081:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress

volumes:
  mysql_data:
Run Application:
docker compose up -d

Access WordPress
http://<EC2-PUBLIC-IP>:8081
After installation, I created a sample blog post.
Persistence Verification
I stopped the containers:
docker compose down
Then restarted them:

docker compose up -d
The WordPress site and blog post were still available, confirming that the MySQL named volume preserved the database data.

Task 4 – Docker Compose Commands
Start services
docker compose up -d
View running services
docker compose ps
View logs
docker compose logs
Follow logs in real time
docker compose logs -f
Stop services without removing containers
docker compose stop
Remove containers and networks
docker compose down
Rebuild images
docker compose up --build

Task 5 – Environment Variables
To improve configuration management, environment variables were moved to a .env file.
.env file
MYSQL_ROOT_PASSWORD=rootpass
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
MYSQL_PASSWORD=wppass
docker-compose.yml (variables referenced)
environment:
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}

Docker Compose automatically loads variables from the .env file.
Verification
docker compose config
This confirmed that environment variables were correctly loaded.

Key Learnings
Docker Compose simplifies multi-container application management.
Services communicate using service names as DNS.
Named volumes ensure data persistence.
.env files help manage configuration securely.

Conclusion
Docker Compose is a powerful tool for managing multi-container applications.
It allows developers to define infrastructure as code and run complex environments with a single command.

Tags
Docker
Docker Compose
WordPress
MySQL
DevOps
Linux
