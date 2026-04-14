# Docker - Complete Reference Guide

Docker is an open-source platform that uses OS-level virtualization (containerization) to build, deploy, and run applications efficiently. It was first released in March 2013 by Solomon Hykes and Sebastien Paul. Docker is written in the **Go** language and runs natively on Linux distributions.

---

## What Problem Does Docker Solve?

Before Docker, developers often faced the issue where code ran correctly on their machine but failed on the user's system. Docker solves this by ensuring consistency across different environments.

---

## Docker Architecture

Docker uses a client-server architecture with the following main components:

| Component | Description |
|-----------|-------------|
| **Docker Daemon** | Runs on the host OS, manages containers and Docker services |
| **Docker Client** | Allows users to interact with Docker daemon using commands & REST API |
| **Docker Host** | Provides environment to run applications (contains daemon, images, containers, networks, storage) |
| **Docker Registry** | Stores Docker images (Public: Docker Hub, Private: Enterprise registry) |

---

## Key Concepts: Image vs Container

> **Image**: A read-only binary template with all dependencies and configurations to run a program. (Not-runnable state)

> **Container**: A runnable instance of an image. When an image is running, it becomes a container.

---

## Advantages of Docker

- No pre-allocation of RAM
- Improves CI/CD efficiency - same image across all deployment stages
- Cost-effective and lightweight
- Runs on physical hardware, virtual hardware, or cloud
- Images are reusable
- Very fast container creation

## Disadvantages of Docker

- Not ideal for applications requiring rich GUI
- Difficult to manage large numbers of containers
- No cross-platform compatibility (Windows container won't run on Linux)
- No built-in solution for data recovery & backup

---

## Docker Commands Cheat Sheet

### Image Management

| Command | Purpose |
|---------|---------|
| `docker images` | List all images on local machine |
| `docker search <image_name>` | Search for images on Docker Hub |
| `docker pull <image_name>` | Download image from Docker Hub to local |
| `docker rmi <image_name>` | Delete an image |

### Container Management

| Command | Purpose |
|---------|---------|
| `docker run -it --name <name> <image_name> /bin/bash` | Create & run container with name and interactive terminal |
| `docker start <container_name>` | Start a stopped container |
| `docker stop <container_name>` | Stop a running container |
| `docker attach <container_name>` | Attach to a running container |
| `docker ps` | List only running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker rm <container_name>` | Delete a container |
| `exit` | Exit from container shell |

### Service Management

| Command | Purpose |
|---------|---------|
| `service docker status` or `service docker info` | Check Docker service status |
| `service docker start` | Start Docker service |

---

## Dockerfile Creation

A **Dockerfile** is a text file containing instructions to automate Docker image creation.

### Dockerfile Instructions

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image (must be first command) |
| `RUN` | Execute commands (creates a layer) |
| `MAINTAINER` | Author/Owner description |
| `COPY` | Copy files from local system (source → destination) |
| `ADD` | Like COPY, but can download from internet & extract files |
| `EXPOSE` | Expose ports (e.g., 8080 for Tomcat, 80 for Nginx) |
| `WORKDIR` | Set working directory for container |
| `CMD` | Execute commands during container creation |
| `ENTRYPOINT` | Similar to CMD but higher priority |
| `ENV` | Set environment variables |
| `ARG` | Define parameters (not accessible after container runs) |

### Dockerfile Creation Steps

1. Create a file named `Dockerfile`
2. Add instructions to the Dockerfile
3. Build the Dockerfile to create an image
4. Run the image to create a container

### Basic Example

```dockerfile
# Dockerfile
FROM ubuntu
RUN echo "Creating our Image" > /tmp/testfile
```

### Build and Run Commands

| Command | Purpose |
|---------|---------|
| `docker build -t <image_name> .` | Build image from Dockerfile (-t for tag, . for current directory) |
| `docker run -it --name <name> <image_name> /bin/bash` | Create container from image |

### Advanced Dockerfile Example

```dockerfile
FROM ubuntu
WORKDIR /tmp
RUN echo "Hello World" > /tmp/testfile
ENV myname DevOps_Brother
COPY testfile /tmp
ADD test.tar.gz /tmp
```

### Commands for Container-to-Image Creation

| Command | Purpose |
|---------|---------|
| `docker diff <container_name>` | See changes (D=Deletion, C=Change, A=Append) |
| `docker commit <container_name> <new_image_name>` | Create image from container |

---

## Docker Volumes

Volumes decouple storage from containers. Benefits include:
- Share volumes among different containers
- Attach volumes to containers
- Volume persists even after container deletion

### Volume Commands

| Command | Purpose |
|---------|---------|
| `docker run -it --name <name> -v /volume_name> <os> /bin/bash` | Create volume |
| `docker run -it --name <name> -privileged=true --volumes-from <source_container> <os> /bin/bash` | Share volume between containers |
| `docker run -it --name <name> -v /host/path:/container/path -privileged=true <os> /bin/bash` | Host to Container volume mapping |

### Volume in Dockerfile

```dockerfile
FROM ubuntu
VOLUME /myvolume1
```

---

## Important Docker Concepts

### docker attach vs docker exec

| Command | Behavior |
|---------|----------|
| `docker exec` | Creates a **new process** in container's environment |
| `docker attach` | Connects to the **main process** inside container (I/O only) |

### expose vs publish (-p)

| Configuration | Accessibility |
|---------------|---------------|
| No expose, no -p | Only inside container itself |
| Only expose | Inside other Docker containers (inter-container communication) |
| expose + -p | Anywhere (outside Docker as well) |

> **Note**: Using `-p` without expose does an implicit expose.

---

## Quick Reference: Docker Workflow

```bash
# 1. Pull an image
docker pull ubuntu

# 2. Run a container
docker run -it --name mycontainer ubuntu /bin/bash

# 3. Make changes inside container
apt-get update && apt-get install nginx

# 4. Exit container
exit

# 5. Commit changes to new image
docker commit mycontainer mycustomimage

# 6. Run new container from custom image
docker run -it --name newcontainer mycustomimage /bin/bash

# 7. Clean up
docker stop mycontainer
docker rm mycontainer
docker rmi ubuntu
```

---

> **Note**: Docker is most suitable when development and testing OS are the same. For different OS environments, Virtual Machines may be better suited.

*Reference: Docker Short Notes for DevOps Engineers*
