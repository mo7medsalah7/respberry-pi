# Docker Folder Contents

## Table of Contents

- [Initializing the Docker Container](#inityaml)
- [Services](#Services)
  - [vscode ](#1-vscode-service)
  - [grafana Folder](#grafana-folder)
  - [nginx Folder](#nginx-folder)
  
---

## init.yaml
The **`init.yaml`** file is a configuration file used for initializing the Docker container.

```version: '3'
services:
  certbot:
    image: certbot/certbot
    network_mode: host
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/lib/letsencrypt:/var/lib/letsencrypt
      - ./nginx/.well-known/:/usr/share/nginx/html/.well-known/
```

## Services
### docker-compose.yaml 

## 1. vscode Service:

```
# Use the specified base image
FROM codercom/code-server:4.16.1

# Switch to root user for permissions
USER root

# Update, and install python3-dev and python3-pip
RUN apt-get update && apt-get install -y python3-dev python3-pip

# Copy the requirements file to the container
COPY requirements.txt /tmp/

# Install Python packages from the requirements file
RUN python3 -m pip install -r /tmp/requirements.txt

# Additional setup (e.g., group and user modifications)
# Replace 995 with the gid you find for the I2C Interface
RUN groupadd -g 995 i2cgroup
RUN usermod -aG i2cgroup coder

# Add the coder user to the dialout group for GPS communication with serial port /dev/ttyUSB*
RUN usermod -aG dialout coder

# Switch to coder user
USER coder

# Install the Visual Studio Code extensions
RUN code-server --install-extension ms-toolsai.jupyter@2023.4.1001091014 \
                --install-extension ms-toolsai.vscode-jupyter-powertoys \
                --install-extension ms-python.python



```

## 2. Questdb Service.

QuestDB is a high-performance, open-source time series database. It is designed to handle large volumes of time-stamped data efficiently. QuestDB is often used for applications such as real-time monitoring, analytics, and logging.

The `questdb` service in your Docker Compose file defines a QuestDB container. The following sections explain the different components of the service:

* `image` Specifies the QuestDB Docker image to use. In this case, the `questdb/questdb:7.3` image is used.
* `ports` Maps the container's ports to the host's ports. This allows you to access the QuestDB database from the host machine.
* `volumes` Mounts a host directory to a directory inside the container. This allows you to store QuestDB data on the host machine.
* `environment` Sets environment variables for QuestDB. In this case, the `QDB_PG_USER` and `QDB_PG_PASSWORD` environment variables are set.


## Grafana



