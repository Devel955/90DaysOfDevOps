# Day 32 – Docker Volumes & Networking
## Task 1: The Problem
### Goal
Understand why container data disappears after a container is removed.

### Step 1: Run a Postgres container
```bash
docker run -d \
  --name pg-no-volume \
  -e POSTGRES_PASSWORD=admin123 \
  -p 5432:5432 \
  postgres:15
```
### Step 2: Enter the container
```bash
docker exec -it pg-no-volume psql -U postgres
```

### Step 3: Create a database table and insert data
```sql
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  course VARCHAR(50)
);

INSERT INTO students (name, course)
VALUES
  ('Anand', 'DevOps'),
  ('Rahul', 'Docker');

SELECT * FROM students;
```
### Example output
```sql
 id | name  | course
----+-------+--------
  1 | Anand | DevOps
  2 | Rahul | Docker
```

### Step 4: Stop and remove the container
Exit from `psql` first:
```bash
\q
```

Then run:
```bash
docker stop pg-no-volume
docker rm pg-no-volume
```

### Step 5: Run a new Postgres container
```bash
docker run -d \
  --name pg-no-volume-new \
  -e POSTGRES_PASSWORD=admin123 \
  -p 5433:5432 \
  postgres:15
```

### Step 6: Check whether old data still exists
```bash
docker exec -it pg-no-volume-new psql -U postgres
```

Now check:
```sql
\dt
SELECT * FROM students;
```
 inserted rows inside `psql`
3. `docker stop` and `docker rm` output
4. New container running
5. `\dt` or error showing table does not exist in the new container

Task 2: Named Volumes
Goal
Understand how Docker named volumes provide persistent storage for containers.
Step 1 – Create a Named Volume
docker volume create pg-data

Verify:
docker volume ls
Step 2 – Run Postgres Container with Volume
docker run -d \
--name pg-volume \
-e POSTGRES_PASSWORD=admin123 \
-v pg-data:/var/lib/postgresql/data \
-p 5434:5432 \
postgres:15

Explanation:
pg-data → Docker named volume
/var/lib/postgresql/data → Postgres data directory

Step 3 – Create Data
Enter container:
docker exec -it pg-volume psql -U postgres

Create table:
CREATE TABLE employees (
id SERIAL PRIMARY KEY,
name VARCHAR(50),
role VARCHAR(50)
);
INSERT INTO employees (name, role)
VALUES ('Anand','DevOps'), ('Rohit','Cloud Engineer');
SELECT * FROM employees;

Exit:
\q
Step 4 – Stop and Remove Container
docker stop pg-volume
docker rm pg-volume
Step 5 – Run New Container with Same Volume
docker run -d \
--name pg-volume-new \
-e POSTGRES_PASSWORD=admin123 \
-v pg-data:/var/lib/postgresql/data \
-p 5435:5432 \
postgres:15
Step 6 – Verify Data
docker exec -it pg-volume-new psql -U postgres
SELECT * FROM employees;

Data still exists.
Observation
Using a named volume, the database data persisted even after removing the container because the data was stored outside the container filesystem.

Task 3: Bind Mounts
Goal

Understand how bind mounts allow containers to access files from the host machine.
Step 1 – Create Folder and File
mkdir nginx-bind-demo
cd nginx-bind-demo
nano index.html

Example HTML:
<html>
<head>
<meta charset="UTF-8">
<title>Bind Mount Demo</title>
</head>
<body>
<h1>Hello Anand 🚀</h1>
<p>This page is coming from the HOST machine using a Bind Mount.</p>
</body>
</html>
Step 2 – Run Nginx Container with Bind Mount
docker run -d \
--name nginx-bind-demo \
-p 8080:80 \
-v $(pwd):/usr/share/nginx/html \
nginx

Step 3 – Access Web Page
Open browser:
http://<server-ip>:8080
The HTML page should appear.

Step 4 – Edit File on Host
Edit:
nano index.html
Change content and save
Refresh browser → changes appear instantly.

Difference: Named Volume vs Bind Mount:
| Named Volume             | Bind Mount                |
| ------------------------ | ------------------------- |
| Managed by Docker        | Uses host filesystem path |
| Good for databases       | Good for development      |
| Portable                 | Host-dependent            |
| Docker controls location | User controls location    |

Task 4: Docker Networking Basics
List Networks
docker network ls
Example:
bridge
host
none
Inspect Default Bridge
docker network inspect bridge
Run Two Containers
docker run -dit --name container1 ubuntu
docker run -dit --name container2 ubuntu
Test Communication

Inside container1:
docker exec -it container1 bash
apt update
apt install iputils-ping -y
Ping by name:
ping container2

This usually fails on default bridge.
Get Container IP
docker inspect container2
Ping using IP:
ping <container-ip>
This works.

Observation
Containers on the default bridge network can communicate using IP addresses, but not always by container names.

Task 5: Custom Networks
Create Network
docker network create my-app-net
Run Containers on Custom Network
docker run -dit --name app1 --network my-app-net ubuntu
docker run -dit --name app2 --network my-app-net ubuntu
Test Communication

Enter container:
docker exec -it app1 bash
apt update
apt install iputils-ping -y
Ping by name:
ping app2
Ping works successfully.

Explanation
User-defined bridge networks provide built-in DNS resolution, allowing containers to communicate using container names.

Task 6: Put It Together
Goal
Combine custom networks + volumes to simulate an application communicating with a database.

Create Network
docker network create app-db-net
Create Volume
docker volume create pg-app-data
Run Database Container
docker run -d \
--name postgres-db \
--network app-db-net \
-e POSTGRES_PASSWORD=admin123 \
-e POSTGRES_DB=myappdb \
-v pg-app-data:/var/lib/postgresql/data \
postgres:15
Run App Container
docker run -dit \
--name app-client \
--network app-db-net \
ubuntu

Verify Connectivity
Enter app container:
docker exec -it app-client bash
Install ping:
apt update
apt install iputils-ping -y
Ping database container:
ping postgres-db
Connection successful.

Conclusion
Using a custom Docker network, containers can communicate using container names through Docker’s internal DNS. The named volume ensures that the database data remains persistent even if the container is removed.

