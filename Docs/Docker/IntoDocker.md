# Docker Overview

Docker is an open-source platform used to **build, deploy, and run applications** efficiently.

It is written in **Go** and uses **OS-level virtualization**, also known as **containerization**, to package applications and their dependencies into lightweight containers.

---

## What is Docker?

- Docker helps developers create consistent environments.
- It ensures that applications run the same way on different systems.
- It simplifies software delivery and deployment.

---

## Key Features

- **Containerization**: Packages applications with all dependencies.
- **Lightweight**: Uses fewer resources than traditional virtual machines.
- **Portable**: Runs consistently across systems.
- **Fast Deployment**: Quick startup and execution.

---

## Advantages of Docker

### 1. No Pre-allocation of RAM
Docker containers use system resources only when needed, making them efficient.

### 2. Improved CI/CD Efficiency
Docker allows you to:
- Build a container image once
- Use the same image across:
  - Development
  - Testing
  - Staging
  - Production

This improves consistency and reduces deployment issues.

### 3. Cost Effective
- Reduces infrastructure costs
- Optimizes hardware usage

### 4. Lightweight
- Faster startup time
- Less overhead than virtual machines


## Architecture of Docker

<img width="1182" height="689" alt="image" src="https://github.com/user-attachments/assets/d739c513-4f91-4429-8346-bb120773dd62" />

## Components of Docker

### 1. Docker Daemon
- It is responsible for running containers, and managing docker services.
### 2. Docker Client
- Docker client uses commands and Rest API to communicate with the docker
daemon.
### 3. Docker Host
- Host is used to providing an environment to execute and run
applications. It contains the docker daemon, images, containers, networks,
and storage.
### 4. Docker Hub/Registry
- Docker registry manages and stores the docker images.
- ### (i) Pubic
- ### (ii) Private

---

## Docker Container
- The container holds the entire package that is needed to run the application.
or,
- In other words, we can say that the image is a template and the container is
a copy of that template.
- container is like virtualization when they run on the Docker engine.
- Images become containers when they run on the docker engine.



--

## Docker Commands


### 1. View All Downloaded Images
```bash
docker images
```
### 2. To find out images in the docker hub.
```bash
docker search <image_name>
```
### 3. To download an image from docker-hub to a local machine.
```bash
docker pull <image_name>
```
### 4. To run a container and give it a name
```bash
docker run --name <container_name> <image_name>
### 4. To run a container and give it a name
```bash
docker run --name <container_name> <image_name>
```
### 4. To run a container and give it a name
```bash
docker run --name <container_name> <image_name>
```
### 5. To check whether the Docker service is running
```bash
systemctl status docker
```
### 6. To start the Docker service
```bash
sudo systemctl start docker
```
### 7. To start a stopped container
```bash
docker start <container_name>
```
### 8. To stop a running container
```bash
docker stop <container_name>
```
### 9. To go inside a running container
```bash
docker exec -it <container_name> /bin/bash
```
### 10. To see all containers
```bash
docker ps -a
```
## 11. To see only running container
```bash
docker ps
```
## 12. To remove (delete) a container
```bash
docker rm <container_name>
```
## 13. To exit from a Docker container
```bash
exit
```
## 14. To delete image
```bash
docker rmi <image_name>
```







