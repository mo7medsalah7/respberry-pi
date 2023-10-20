# Docker Folder Contents

This markdown file provides an overview of the contents of the Docker folder in your repository.

## Table of Contents

- [Initializing the Docker Container](#inityaml)
- [Services](#Services)
  - [vscode ](#1-vscode-service)
  - [grafana Folder](#grafana-folder)
  - [nginx Folder](#nginx-folder)
  
---

## init.yaml
The **`init.yaml`** file is a configuration file used for initializing the Docker container.

This **init.yaml** file provides the necessary configuration for the certbot service in the Docker container, enabling it to access Let's Encrypt certificates and data from the host system.

- image: **certbot/certbot**: Specifies the Docker image to be used, in this case, *certbot/certbot*.

- network_mode: **host**: Sets the container's network mode to "host," allowing the container to use the host network stack.

- volumes: Mounts directories or files from the host system into the container.

## Services
### docker-compose.yaml 

Docker Compose is a tool that helps you define and run multi-container Docker applications. With Compose, you use a YAML file to configure your application's services. Then, with a single command, you create and start all the services from your configuration.


- Run the following command, This will create and start all of the services defined in the docker-compose.yaml file.

`docker-compose up -d`

## 1. vscode Service:

* You can access it independently port *8080* on your host [localhost:8080](http://localhost:8080).

* `build`: Specifies the build context and Dockerfile location for the service.

* `ports`: Maps the host port 8080 to the container port 8080.

* `volumes`: Mounts the `../notebooks` directory on the host to `/home/coder/project` inside the container. Volume used to access the `notebooks` folder.

* `environment`: Sets the environment variable 
             - :lock: **PASSWORD** to **yourpassword**.

* `devices`: Maps host devices /dev/i2c-1, /dev/ttyUSB1, and /dev/ttyUSB2 to the corresponding devices inside the container.

   - *I2C and Serial Interfaces for respectively BME680 sensor and QUECTEL EG25-G 4G HAT cellular device.*

## 2. Questdb Service.

QuestDB is a high-performance, open-source time series database. It is designed to handle large volumes of time-stamped data efficiently. QuestDB is often used for applications such as real-time monitoring, analytics, and logging.

The `questdb` service in your Docker Compose file defines a QuestDB container. The following sections explain the different components of the service:

* `image` Specifies the QuestDB Docker image to use. In this case, the `questdb/questdb:7.3` image is used.
* `ports` Maps the container's ports to the host's ports. This allows you to access the QuestDB database from the host machine.
* `volumes` Mounts a host directory to a directory inside the container. This allows you to store QuestDB data on the host machine.
* `environment` Sets environment variables for QuestDB. In this case, the `QDB_PG_USER` and `QDB_PG_PASSWORD` environment variables are set.


## Grafana



