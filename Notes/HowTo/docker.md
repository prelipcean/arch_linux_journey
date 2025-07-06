# Docker Quick Reference & How-To

This guide covers essential Docker commands, concepts, and troubleshooting for Linux users.

---

## 📦 Installation

- [Install Docker on Mac](https://docs.docker.com/desktop/setup/install/mac-install/)
- [Install Docker on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Install Docker on Ubuntu](https://docs.docker.com/desktop/setup/install/linux/ubuntu/)

### Running Docker Without `sudo`

[Post-installation steps | Docker Docs](https://docs.docker.com/engine/install/linux-postinstall/)

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
# Log out and back in, or:
newgrp docker
docker run hello-world
```

---

## 🛠️ Basic Commands

### Version & Info

```bash
docker --version
docker info
```

### Images

```bash
docker pull nginx
docker images
docker search ubuntu
docker pull ubuntu:22.04
docker image rm [IMAGE_ID]
docker rmi [IMAGE_ID]
docker rmi -f [IMAGE_ID]
docker image rm $(docker images -q)  # Remove all images
```

### Containers

```bash
docker run nginx
docker run -d nginx
docker run -d -p 8080:80 --name web-server nginx
docker ps
docker ps -a
docker stop [CONTAINER]
docker kill [CONTAINER]
docker rm [CONTAINER]
docker rm $(docker ps -aq)           # Remove all containers
docker rm -f $(docker ps -aq)        # Force remove all containers
docker exec -it [CONTAINER] sh       # Interactive shell
docker logs [CONTAINER]
docker logs -f [CONTAINER]           # Live logs
docker stats                         # Resource usage
```

### Filtering & Inspecting

```bash
docker ps --filter name=[NAME]
docker ps -q
docker inspect [CONTAINER]
docker history [IMAGE]
```

---

## 🏗️ Building Images

### Build from Dockerfile

```bash
docker build .
docker build -t my-image .
```

**Example Dockerfile:**
```dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y procps
CMD ["/bin/bash", "-c", "echo 'Container Information:' && uname -a && cat /etc/os-release && lsblk"]
```

### Tag & Push

```bash
docker tag my-image:latest myuser/my-image:v0.0.1
docker push myuser/my-image:v0.0.1
```

---

## 🗂️ Data Management

### Volumes

```bash
docker volume create website-data
docker run -d -p 3000:80 --name website-main -v website-data:/usr/share/nginx/html nginx:1.27.0
docker volume ls
docker volume inspect website-data
docker volume rm website-data
docker volume rm $(docker volume ls -f dangling=true -q)
```

### Bind Mounts

```bash
docker run --rm -d -p 3000:3000 -v ./public:/app/public -v ./src:/app/src react:dev
```

---

## 🌐 Networking

### List & Inspect Networks

```bash
docker network ls
docker network inspect bridge
```

### User-Defined Networks

```bash
docker network create app-net
docker run -d --name webserver --network app-net nginx:1.27.0
docker run -it --network app-net alpine:3.20 sh
# Inside container:
apk add curl
curl webserver
```

### Host Network (Linux Only)

```bash
docker run -d --network host nginx:1.27.0
curl http://localhost
```

### Remove Networks

```bash
docker network rm app-net
docker network prune
```

---

## ⚙️ Environment Variables

### In Dockerfile

```dockerfile
ENV PORT=3000
```

### At Runtime

```bash
docker run -e PORT=5001 -d -p 5001:5001 --name express-app express
docker run --env-file ".env.prod" -d -p 5000:5000 --name express-app express
```

---

## 🧹 Cleanup & Prune

```bash
docker system prune           # Remove unused data (not volumes)
docker system prune --volumes # Also remove unused volumes
```

---

## 🔄 Restart Policies

```bash
docker run -d --name restart_fail --restart on-failure busybox sh -c "sleep 3; exit 1"
docker run -d --name restart_fail --restart on-failure:3 busybox sh -c "sleep 3; exit 1"
docker run -d --name restart_always --restart always busybox sh -c "sleep 3; exit 0"
docker run -d --name restart_us --restart unless-stopped busybox sh -c "sleep 3; exit 0"
```

---

## 🐳 Docker Compose

```bash
docker compose version
docker-compose version
```

---

## 📝 Troubleshooting

- **Port already in use:**
    ```
    bind() to [::]:80 failed (98: Address already in use)
    ```
    - Stop/remove conflicting containers:
      ```bash
      docker rm -f $(docker ps -aq)
      ```

- **Logs:**
    ```bash
    docker logs [container_name_that_failed]
    ```

---

## 🔗 Resources

- [Docker Hub](https://hub.docker.com/)
- [Play with Docker](https://labs.play-with-docker.com/)
- [Node.js Download](https://nodejs.org/en/download)
- [Set up Node.js on WSL 2](https://learn.microsoft.com/en-us/windows/dev-environment/javascript/nodejs-on-wsl)

---
