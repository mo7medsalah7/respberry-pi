# Docker Folder Contents

## Table of Contents

- [Initializing the Docker Container](#inityaml)
- [Services](#Services)
  - [vscode ](#1-vscode-service)
    - [Dockerfile](#this-is-the-dockerfile-for-the-vscode-service)
  - [nginx Folder](#2-nginx-service)
    - [nginx.conf](#nginxconf-file)
    - [nginx.env](#nginxenv-file)
  
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

### This is the `Dockerfile` for the vscode service

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

## 2. Nginx Service.

### **nginx.conf** file

```
# /etc/nginx/nginx.conf


user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

### **nginx.env** file
```
# nginx/nginx.env

DOMAIN=yourdomain
```
## Grafana



