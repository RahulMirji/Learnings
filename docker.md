# 🐳 Docker Basics – Detailed Guide

## 📌 What is Docker?

Docker is a platform that allows developers to **build, package, and run applications inside containers**.

A container includes:

* Application code
* Runtime (Node, Python, etc.)
* Libraries and dependencies
* System tools

👉 This ensures the application runs the same way on any machine.

---

## ⚠️ Problem Docker Solves

Without Docker:

* App works on your machine ❌ fails on another
* Dependency conflicts
* Different OS environments

With Docker:

* Same environment everywhere ✅
* Easy deployment
* Faster setup

---

## 🧠 Core Concepts

### 1️⃣ Docker Image

A Docker Image is a **read-only template** used to create containers.

👉 Think of it like a *blueprint* or *class*

#### Example:

```bash
docker pull node
```

This pulls a Node.js image.

---

### 2️⃣ Docker Container

A Container is a **running instance of an image**.

👉 Think of it like an *object created from a class*

#### Example:

```bash
docker run node
```

This creates and runs a container.

---

### 3️⃣ Dockerfile

A Dockerfile is a **script containing instructions** to build an image.

#### Example:

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
```

Explanation:

* `FROM` → Base image
* `WORKDIR` → Working directory
* `COPY` → Copy files
* `RUN` → Execute commands
* `CMD` → Default command when container runs

---

### 4️⃣ Docker Hub

Docker Hub is a **cloud registry** where images are stored and shared.

👉 Similar to GitHub but for Docker images

#### Example:

```bash
docker pull postgres
```

---

## 🔄 Lifecycle: Image → Container

1. Build image using Dockerfile
2. Store image locally or on Docker Hub
3. Run container from image
4. Container executes and may stop or continue

---

## ⚙️ Basic Commands

### Check Docker Version

```bash
docker --version
```

### Pull Image

```bash
docker pull ubuntu
```

### Run Container

```bash
docker run ubuntu
```

### Interactive Mode

```bash
docker run -it ubuntu bash
```

### List Running Containers

```bash
docker ps
```

### List All Containers

```bash
docker ps -a
```

### Build Image

```bash
docker build -t myapp .
```

### Run App with Port Mapping

```bash
docker run -p 3000:3000 myapp
```

---

## 🔗 Port Mapping Explained

```bash
docker run -p 3000:3000 myapp
```

* First `3000` → Host machine port
* Second `3000` → Container port

👉 Access app at: `http://localhost:3000`

---

## 🧪 Example: Node.js App in Docker

### Step 1: Create app.js

```javascript
const http = require('http');

http.createServer((req, res) => {
  res.write("Hello from Docker Node App");
  res.end();
}).listen(3000);
```

---

### Step 2: Create Dockerfile

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
CMD ["node", "app.js"]
```

---

### Step 3: Build Image

```bash
docker build -t node-app .
```

---

### Step 4: Run Container

```bash
docker run -p 3000:3000 node-app
```

👉 Open browser → [http://localhost:3000](http://localhost:3000)

---

## 🔄 Container States

| State   | Meaning                           |
| ------- | --------------------------------- |
| Created | Container created but not started |
| Running | Container is executing            |
| Exited  | Container has stopped             |
| Paused  | Execution temporarily halted      |

---

## ⚡ Detached Mode

Run container in background:

```bash
docker run -d nginx
```

Stop container:

```bash
docker stop <container_id>
```

---

## 🧾 Logs & Debugging

View logs:

```bash
docker logs <container_name>
```

Inspect container:

```bash
docker inspect <container_id>
```

---

## 🧩 Docker vs Virtual Machine

| Feature     | Docker        | VM      |
| ----------- | ------------- | ------- |
| Size        | Small         | Large   |
| Boot Time   | Seconds       | Minutes |
| OS          | Shared Kernel | Full OS |
| Performance | High          | Lower   |

---

## 📦 [NEW & ADVANCED] Docker Volumes (Data Persistence)

By default, data inside a container is lost when the container is deleted. **Volumes** solve this by storing data outside the container.

### Types of Storage:
1. **Volumes:** Managed by Docker (Recommended).
2. **Bind Mounts:** Maps a specific path on the host machine to the container.

#### Creating a Volume:
```bash
docker volume create my-data
```

#### Running a Container with a Volume:
```bash
docker run -v my-data:/var/lib/postgresql/data postgres
```

---

## 🌐 [NEW & ADVANCED] Docker Networking

Containers need to communicate with each other. Docker networks make this possible securely.

### Default Networks:
* **bridge:** Default network for standalone containers.
* **host:** Removes network isolation between the container and the Docker host.
* **none:** Disables networking entirely.

#### Creating a Custom Network:
```bash
docker network create my-network
```

#### Running a Container on a Network:
```bash
docker run --network my-network myapp
```

---

## 🐙 [NEW & ADVANCED] Docker Compose

Docker Compose is a tool for defining and running **multi-container Docker applications**. It uses a `docker-compose.yml` file.

### Example `docker-compose.yml`:
```yaml
version: '3.8'
services:
  web:
    image: node-app
    ports:
      - "3000:3000"
    depends_on:
      - db
  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: secretpassword
    volumes:
      - pg-data:/var/lib/postgresql/data

volumes:
  pg-data:
```

### Key Compose Commands:
* `docker-compose up -d` → Start all services in the background.
* `docker-compose down` → Stop and remove all services.

---

## 🏗️ [NEW & ADVANCED] Dockerfiles: Multi-Stage Builds

Multi-stage builds help create **smaller, optimized, and more secure** production images by separating the build environment from the runtime environment.

### Example (React/Node.js to Nginx):
```Dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
👉 The final image only contains the compiled static files and Nginx, making it extremely lightweight.

---

## 🚀 Why Docker is Important

* Consistent environments
* Faster deployment
* Microservices architecture
* Cloud & DevOps standard
* Easy scaling

---

## 🎯 Summary

* Image = Blueprint
* Container = Running instance
* Dockerfile = Instructions
* Docker Hub = Image storage

👉 Docker removes: *"It works on my machine" problem*

---

## ✅ Practice Tasks

1. Run `hello-world` container
2. Run `ubuntu` container interactively
3. Build your own Node app image
4. Run container with port mapping
5. Check logs of a container
6. Create a Docker Volume and attach it to a database container
7. Write a `docker-compose.yml` file for a frontend and backend app
8. Optimize a Dockerfile using Multi-Stage Builds

---

🔥 You now understand Docker fundamentals like a developer, not a beginner.
