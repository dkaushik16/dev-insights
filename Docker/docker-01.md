# Docker

## Overview

- An open platform for developing, shipping, and running applications.
- Provides the ability to package and run an application in a loosely isolated environment called a container.
- Docker provides tooling and a platform to manage the lifecycle of your containers
- Containers are - portable and lightweight

## Port mapping

- docker run -p host_port : container_port IMAGE_NAME

## Docker vs Virtual Machine

- hardware → host OS kernel → application layer
- docker virtualize only application layer and uses the kernel of host OS. → lightweight
- VM virtualize the application as well as the kernel layer. → heavier and complex
- Docker is compatible to Linux only while VM is compatible with all Operation systems as it has its own virtualized kernel layer.

## NETWORK

- Docker networking enables communication between containers, the host machine, and external systems through virtual networks such as Bridge ( default ) , Host, Overlay, and None.

### 1. Bridge Network (Default)

Bridge is the default Docker network for containers running on the same host machine. Containers connected to the same bridge network can communicate with each other using their container names, while remaining isolated from containers on other networks.

### 2. Host Network

Host networking removes network isolation between the container and the host machine. The container uses the host's network directly, which can improve performance but reduces isolation and may lead to port conflicts.

### 3. None Network

The none network completely disables networking for a container. The container cannot communicate with other containers, the host, or external networks unless networking is manually configured later.

### 4. Overlay Network

Overlay networking is used in Docker Swarm to connect containers running across multiple Docker hosts. It creates a virtual network spanning several machines, allowing distributed services to communicate securely as if they were on the same network.

## DOCKER VOLUME

- A **volume** is a storage area managed by Docker that keeps data **even if a container is deleted**.
- without volume→ data stored in container → if container is removed then data is lost.
- with volume → data stored outside the container→ persists across container restarts and recreations.

### Volume mounting

- attaching a volume (or a host folder) to a directory inside a container so the container can read and write data there.

## DOCKER COMPOSE

- Docker Compose is a tool that uses a YAML file to define and manage multiple containers, networks, and volumes as a single application. It allows all services to be started, stopped, and configured together using commands like ‘docker compose up’  and  ‘docker compose down’