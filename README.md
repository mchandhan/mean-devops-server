---

# MEAN Stack DevOps Deployment

### Docker | Nginx | GitHub Actions | AWS EC2

This repository contains a **production-ready MEAN stack deployment** architecture. It leverages containerization for environment consistency and an automated CI/CD pipeline to ensure seamless delivery from code commit to a live AWS environment.

---

## Architecture Overview

The system is decoupled into four primary services orchestrated via **Docker Compose**:

1. **Frontend:** Angular 15 application compiled into static files and served via Nginx.
2. **Backend:** Node.js & Express REST API handling business logic.
3. **Database:** MongoDB 6 with a persistent Docker volume for data durability.
4. **Reverse Proxy:** A dedicated Nginx container managing traffic routing:
* `http://<IP>/` → Frontend Service
* `http://<IP>/api/` → Backend Service



---

## Step-by-Step Setup & Deployment

### 1. Prerequisites

* AWS EC2 Instance (Ubuntu 22.04 recommended).
* Docker & Docker Compose installed on the instance.
* Docker Hub account for image hosting.

### 2. Infrastructure Setup (EC2)

Connect to your instance and run:

```bash
# Update and Install Docker
sudo apt update && sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER && newgrp docker

```

### 3. CI/CD Secrets Configuration

In your GitHub Repository, navigate to **Settings > Secrets and variables > Actions** and add:

* `DOCKER_USERNAME`: Your Docker Hub ID.
* `DOCKER_PASSWORD`: Your Docker Hub Access Token.
* `EC2_SSH_KEY`: Your private `.pem` key.
* `EC2_HOST`: Public IP of your EC2 instance.

---

## Deployment Showcases

### CI/CD Pipeline (GitHub Actions)

The pipeline automatically triggers on every push to `main`, running tests, building images, and deploying via SSH.

```
name: MEAN Stack CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # -----------------------------
      # Checkout Code
      # -----------------------------
      - name: Checkout Repository
        uses: actions/checkout@v3

      # -----------------------------
      # Login to Docker Hub
      # -----------------------------
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # -----------------------------
      # Build Backend Image
      # -----------------------------
      - name: Build Backend Docker Image
        run: |
          docker build -t chandhanm/mean-backend:latest ./backend
      # -----------------------------
      # Push Backend Image
      # -----------------------------
      - name: Push Backend Docker Image
        run: |
          docker push chandhanm/mean-backend:latest
      # -----------------------------
      # Build Frontend Image
      # -----------------------------
      - name: Build Frontend Docker Image
        run: |
          docker build -t chandhanm/mean-frontend:latest ./frontend
      # -----------------------------
      # Push Frontend Image
      # -----------------------------
      - name: Push Frontend Docker Image
        run: |
          docker push chandhanm/mean-frontend:latest
      # -----------------------------
      # Deploy to EC2
      # -----------------------------
      - name: Deploy to EC2 Server
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd mean-devops-server
            docker compose pull
            docker compose down
            docker compose up -d
```

###  Docker Hub Registry

Standardized images are tagged and pushed, ensuring the exact same code runs in dev and production.

```
https://hub.docker.com/repository/docker/chandhanm/mean-frontend/general

https://hub.docker.com/repository/docker/chandhanm/mean-backend/general
```

### Live Application UI

The final deployment successfully communicating with the database through the proxy.

```
server {
    listen 80;

    # Frontend
    location / {
        proxy_pass http://frontend:80;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }

    # Backend API
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```

## 📂 Project Structure

```text
mean-devops-server/
├── .github/workflows/    # CI/CD Pipeline (YAML)
├── backend/              # Node/Express Source & Dockerfile
├── frontend/             # Angular Source & Dockerfile
├── nginx/                # Reverse Proxy Config
│   └── default.conf
├── docker-compose.yml    # Main Orchestration File
└── README.md             # Documentation

```

## Docker Compose Snippet

```
version: "3.8"

services:
  mongodb:
    image: mongo:6
    container_name: mongodb
    restart: always
    volumes:
      - mongo-data:/data/db
    networks:
      - mean-network

  backend:
    image: chandhanm/mean-backend:latest
    container_name: backend
    restart: always
    ports:
      - "8080:8080"
    depends_on:
      - mongodb
    environment:
      - MONGO_URL=mongodb://mongodb:27017/dd_db
    networks:
      - mean-network

  frontend:
    image: chandhanm/mean-frontend:latest
    container_name: frontend
    restart: always
    depends_on:
      - backend
    networks:
      - mean-network

  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend
    networks:
      - mean-network

volumes:
  mongo-data:

networks:
  mean-network:
    driver: bridge
```

## Useful DevOps Commands

| Action | Command |
| --- | --- |
| **Check Logs** | `docker-compose logs -f backend` |
| **Force Rebuild** | `docker-compose up -d --build` |
| **Check DB Volume** | `docker volume inspect mean-devops-server_mongo-data` |
| **Stop All** | `docker-compose down` |

---

## Production Features

* **Persistent Data:** MongoDB data persists even if the container is removed.
* **Security:** Only Port 80 is exposed; the DB and API remain on a private Docker network.
* **Automation:** Full "Push-to-Deploy" capability.
* **Efficiency:** Nginx serves static Angular files for high performance.

---
