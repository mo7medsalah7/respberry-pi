# Docker Folder Contents

## Table of Contents

- [Initializing the Docker Container](#inityaml)
- [Docker Compose](#docker-compose)
- [Services](#Services)
  - [vscode folder](#1-vscode-service)
    - [Dockerfile](#this-is-the-dockerfile-for-the-vscode-service)
    - [requirements.txt](#requirementstxt)
  - [nginx folder](#2-nginx-service)
    - [nginx.conf](#nginxconf-file)
    - [nginx.env](#nginxenv-file)
    - [default conf template](#default.conf.template)
  
---

## init.yaml

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

## docker-compose.yaml

```
version: "3"

services:
  vscode:
    build:
      context: vscode
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    volumes:
      # volume used to access the `notebooks` folder
      - ../notebooks:/home/coder/project

    environment:
      - PASSWORD=yourpassword
    devices:
    # I2C and Serial Interfaces for respectively BME680 sensor and QUECTEL EG25-G 4G HAT cellular device.
      - "/dev/i2c-1:/dev/i2c-1"
      - "/dev/ttyUSB1:/dev/ttyUSB1"
      - "/dev/ttyUSB2:/dev/ttyUSB2"

  grafana:
    image: grafana/grafana:9.5.1
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning/datasources/:/etc/grafana/provisioning/datasources/
      - ./grafana/provisioning/dashboards/:/etc/grafana/provisioning/dashboards/
      - ./grafana/dashboards:/etc/dashboards
    depends_on:
      - questdb
    environment:
      - GRAFANA_QUESTDB_PASSWORD=quest
      - GF_INSTALL_PLUGINS=pr0ps-trackmap-panel

  questdb:
    image: questdb/questdb:7.3
    ports:
      - "9000:9000"
      - "9009:9009"
      - "8812:8812"
      - "9003:9003"
    volumes:
      - questdb-data:/root/.questdb
    environment:
      - QDB_PG_USER=admin
      - QDB_PG_PASSWORD=quest

  nginx:
    image: nginx:stable
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d/default.conf.template:/etc/nginx/conf.d/default.conf.template
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
      - /etc/letsencrypt:/etc/letsencrypt
      - ./nginx/.well-known/:/usr/share/nginx/html/.well-known/
      - ./nginx/certs/dhparam.pem:/etc/ssl/certs/dhparam.pem
      - ./nginx/.htpasswd:/etc/nginx/.htpasswd
    ports:
      - "80:80"
      - "443:443"
    env_file:
      - ./nginx/nginx.env
    command: /bin/bash -c "envsubst '$$DOMAIN' < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && exec nginx -g 'daemon off;'"
    depends_on:
      - vscode
      - grafana
      - questdb

  # Refer to ./init.yaml to for initial SSL setup
  # add the following line in in crontab (5 am, every monday)
  #   0 5 * * 1 docker compose run certbot renew
  certbot:
    image: certbot/certbot
    ports: []
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/lib/letsencrypt:/var/lib/letsencrypt
      - ./nginx/.well-known/:/usr/share/nginx/html/.well-known/


volumes:
  grafana-data:
  questdb-data:
```

## Services

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

### **requirements.txt**

```
# <- Comment or uncomment based on the application

#some-weatherstation-package==x.y.z   <- Comment or uncomment based on the application
bme680==1.1.1   
psycopg2-binary==2.9.7


# some-healthcare-package==x.y.z   <- Comment or uncomment based on the application

# some-GPStracker-package==x.y.z   <- Comment or uncomment based on the application
openrouteservice==2.3.3
pyserial==3.5

# some-MarketData-package==x.y.z   <- Comment or uncomment based on the application
 websocket-client==1.6.3
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

### **conf.d/default.conf.template**

```
# /etc/nginx/conf.d/default.conf.template




# top-level http config for websocket headers
# If Upgrade is defined, Connection = upgrade
# If Upgrade is empty, Connection = close
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# HTTP server to redirect all 80 traffic to SSL/HTTPS
server {
    listen 80;
    server_name vscode.${DOMAIN} questdb.${DOMAIN} grafana.${DOMAIN};

    # Tell all requests to port 80 to be 302 redirected to HTTPS
    location / {
        return 302 https://$host$request_uri;
    }

    location ~ /.well-known {
        root /usr/share/nginx/html;
        allow all;
    }
}

# HTTPS server to handle VS Code
server {
    listen 443 ssl;
    server_name vscode.${DOMAIN};

    ssl_certificate /etc/letsencrypt/live/vscode.${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vscode.${DOMAIN}/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;

    # Managing literal requests to the VS Code front end
    location / {
        proxy_pass http://vscode:8080;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # websocket headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Scheme $scheme;

        proxy_buffering off;
    }

    # Managing requests to verify letsencrypt host
    location ~ /.well-known {
        allow all;
    }
}


# HTTPS server to handle Questdb
server {
    listen 443 ssl;
    server_name questdb.${DOMAIN};

    ssl_certificate /etc/letsencrypt/live/questdb.${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/questdb.${DOMAIN}/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;

    # Managing literal requests to the Questdb front end
    location / {
        proxy_pass http://questdb:9000;
        autoindex on;
        auth_basic "Restricted Content";
        auth_basic_user_file .htpasswd;


        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # websocket headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Scheme $scheme;

        proxy_buffering off;
    }

    # Managing requests to verify letsencrypt host
    location ~ /.well-known {
        allow all;
    }
}


# HTTPS server to handle grafana
server {
    listen 443 ssl;
    server_name grafana.${DOMAIN};

    ssl_certificate /etc/letsencrypt/live/grafana.${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grafana.${DOMAIN}/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;

    # Managing literal requests to the Grafana front end
    location / {
        proxy_pass http://grafana:3000;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # websocket headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Scheme $scheme;

        proxy_buffering off;
    }

    # Managing requests to verify letsencrypt host
    location ~ /.well-known {
        allow all;
    }
}



```



## Grafana



