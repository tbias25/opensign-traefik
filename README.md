# opensign-traefik
This repository provides a complete Docker Compose setup to run [OpenSign](https://www.opensignlabs.com/) behind a [Traefik](https://traefik.io/traefik/) reverse proxy. It enables a simple, secure, and production-ready deployment of OpenSign with automatic routing, HTTPS ([Let's Encrypt](https://letsencrypt.org/)), and configurable domain settings.

## Table of Contents
1. [Introduction](#introduction)
2. [Reverse Proxy](#reverse-proxy)
3. [OpenSign Services](#opensign-services)
    1. [MongoDB](#mongodb)
    2. [OpenSign Server](#opensign-server)
    3. [OpenSign Client](#opensign-client)
4. [Configuration File Description](#configuration-file-description)
    1. [Frontend Configuration](#frontend-configuration)
    2. [Backend Configuration](#backend-configuration)
    3. [Storage Configuration](#storage-configuration)
    4. [Email Configuration](#email-configuration)

## Introduction

I noticed on the [OpenSign Discord](https://discord.com/invite/xe9TDuyAyj) that many users were having trouble hosting OpenSign behind a Traefik reverse proxy. To make the process easier and provide a ready-to-use solution, I created this repository with a complete, working Docker Compose setup.

## Reverse Proxy

This repository sets up a Traefik reverse proxy to securely route incoming traffic to OpenSign and other services.

The `traefik` service is configured with the following important settings:

- Traefik listens on ports 80 (HTTP) and 443 (HTTPS) through:
  ```yaml
  - "--entrypoints.web.address=:80"
  - "--entrypoints.websecure.address=:443"
  ```

- Traefik automatically generates and manages SSL certificates via Let's Encrypt:

  ```yaml
  - "--certificatesresolvers.le.acme.email=example@example.com"
  - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
  - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
  ```

- The Traefik dashboard is exposed securely on a custom subdomain (e.g., traefik.example.com):

  ```yaml
  - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
  - "traefik.http.routers.dashboard.entrypoints=websecure"
  - "traefik.http.routers.dashboard.tls.certresolver=le"
  - "traefik.http.services.dashboard.loadbalancer.server.port=8080"
  ```

- A named Docker network (proxy_network) is created so Traefik can communicate with other services:

  ```yaml
  networks:
    proxy:
      name: proxy_network
      driver: bridge
  ```

- Important security setting:

  ```yaml
  - "--providers.docker.exposedbydefault=false"
  ```
  This setting ensures that Traefik does not expose all Docker containers by default.
  Only containers with the label traefik.enable=true will be routed, improving overall security.

  All services that should be exposed by Traefik must be connected to the proxy_network and have the necessary Traefik labels set.

## OpenSign Services

This Docker Compose setup defines and starts three main services required for OpenSign to work:

### MongoDB

The `mongo` service provides a MongoDB database for OpenSign to store its data.

- Exposes port `27017` locally:
 
  ```yaml
  ports:
    - "27017:27017"
  
- Stores database files in a named Docker volume (data-volume).
 
  ```yaml
    volumes:
        - data-volume:/data/db
  ```

- Connects the container to the `proxy` network.

  ```yaml
    networks:
      - proxy
  ```
  
### OpenSign Server
The `server` service runs the OpenSign backend API.

- Depends on the `mongo` service.

  ```yaml
    depends_on:
      - mongo
  ```

- Uses environment variables from [.env.prod](https://github.com/tbias25/opensign-traefik/blob/main/opensign/.env.prod) for configuration:
  
  ```yaml
  environment:
  - NODE_ENV=production
  ```
- Stores uploaded files handled by OpenSign in a named Docker volume (opensign-files).
 
  ```yaml
  volumes:
      - opensign-files:/usr/src/app/files
  ```

- Exposes itself behind Traefik (e.g. https://opensign.example.com/api), accessible through the web (80 / http) and websecure (443 / https) entrypoints.
  
  ```yaml
  - "traefik.enable=true"
  - "traefik.http.routers.server.rule=Host(`opensign.example.com`) && PathPrefix(`/api`)"
  - "traefik.http.routers.server.entrypoints=web,websecure"
  ```

- Routes HTTP traffic to the internal port `8080`:
  
  ```yaml
  - "traefik.http.services.server.loadbalancer.server.port=8080"
  ```

- Automatic SSL certificate generation via Let's Encrypt
  
  ```yaml
  - "traefik.http.routers.server.tls.certresolver=le"
  ```

- Strips the /api prefix from incoming routes using a Traefik middleware:

  ```yaml
  - "traefik.http.routers.server.middlewares=server-strip-api@docker"
  - "traefik.http.middlewares.server-strip-api.stripprefix.prefixes=/api"
  ```

- Connects the container to the `proxy` network.

  ```yaml
    networks:
      - proxy
  ```

### OpenSign Client
The `client` service runs the OpenSign web interface (frontend).

- Depends on the `server` service.

  ```yaml
    depends_on:
      - server
  ```

- Uses environment variables from [.env.prod](https://github.com/tbias25/opensign-traefik/blob/main/opensign/.env.prod) for configuration:
  
  ```yaml
  environment:
  - NODE_ENV=production
  ```
  
- Exposes itself behind Traefik (e.g. https://opensign.example.com), accessible through the web (80 / http) and websecure (443 / https) entrypoints.
  
  ```yaml
  - "traefik.enable=true"
  - "traefik.http.routers.client.rule=Host(`opensign.example.com`)"
  - "traefik.http.routers.client.entrypoints=web,websecure"
  ```

- Routes HTTP traffic to the internal port `3000`:
  
  ```yaml
  - "traefik.http.services.client.loadbalancer.server.port=3000"
  ```

- Automatic SSL certificate generation via Let's Encrypt
  
  ```yaml
  - "traefik.http.routers.client.tls.certresolver=le"
  ```

- Connects the container to the `proxy` network.

  ```yaml
    networks:
      - proxy
  ```


### Volumes
Two named volumes are used:

- `data-volume`: Stores MongoDB database data.
- `opensign-files`: Stores uploaded files handled by OpenSign.

  ```yaml
    volumes:
      data-volume:
      opensign-files:
  ```


### Networks
All services are connected to a shared external network `proxy_network`, which allows Traefik to manage routing:

```yaml
networks:
  proxy:
    name: proxy_network
    driver: bridge
    external: true
```

## Configuration File Description

This file contains configuration variables for both the frontend and backend of the OpenSign application, as well as storage and email configurations. The values in this file should be customized according to your deployment environment.

### Frontend Configuration

- **PUBLIC_URL**: The URL where the home page of the app will be accessed (e.g., `https://opensign.example.com/`).
- **GENERATE_SOURCEMAP**: Set to `false` if you do not wish to generate a sourcemap for debugging purposes.
- **REACT_APP_SERVERURL**: The URL where the APIs are accessible. For local development, it should be something like `http://localhost:3000/api/app`.
- **REACT_APP_APPID**: A 12-character random app identifier, which should match the `APP_ID` used in the backend API.
- **REACT_APP_GTM**: Google Tag Manager container ID for configuring GTM on all pages of the app.

### Backend Configuration

- **APP_ID**: A 12-character random app identifier. This should match the frontend `REACT_APP_APPID`.
- **appName**: The name of the app that will appear in verification emails.
- **MASTER_KEY**: A random 12-character secret key used to access all data. This key is used in the Parse dashboard for viewing database data.
- **MONGODB_URI**: MongoDB URI for connecting to the backend database (`mongodb://mongo:27017/opensign`).
- **PARSE_MOUNT**: The path on which APIs should be mounted (`/app`). This variable is subject to being hardcoded in future versions.
- **SERVER_URL**: The URL from which the backend APIs are accessible to NodeJS functions. For local development, it should be `http://localhost:3000/api/app`.

### Storage Configuration

- **DO_SPACE**: The name of the DigitalOcean Space or AWS S3 bucket where documents will be uploaded.
- **DO_ENDPOINT**: The endpoint for DigitalOcean Spaces or AWS S3 for document uploads (e.g., `ams3.digitaloceanspaces.com`).
- **DO_BASEURL**: The base URL for accessing the uploaded documents (e.g., `https://DOSPACENAME.ams3.digitaloceanspaces.com`).
- **DO_ACCESS_KEY_ID**: The access key ID for DigitalOcean Spaces or AWS S3.
- **DO_SECRET_ACCESS_KEY**: The secret access key for DigitalOcean Spaces or AWS S3.
- **DO_REGION**: The region of the DigitalOcean Space or AWS S3 (e.g., `us-west`).
- **USE_LOCAL**: Set to `TRUE` if using local storage instead of an S3-compatible service.

### Email Configuration

- **MAILGUN_API_KEY**: The API key for Mailgun.
- **MAILGUN_DOMAIN**: The domain used for sending emails via Mailgun (e.g., `mail.yourdomain.com`).
- **MAILGUN_SENDER**: The sender email address used in Mailgun emails (e.g., `postmaster@mail.yourdomain.com`).
- **SMTP_ENABLE**: Set to `true` to enable SMTP configuration for email sending.
- **SMTP_HOST**: The SMTP server hostname (e.g., `smtp.mailgun.org`).
- **SMTP_PORT**: The SMTP server port (e.g., `587`).
- **SMTP_USER_EMAIL**: The email address for SMTP authentication.
- **SMTP_PASS**: The password for SMTP authentication.

### Document Signing Certificate

- **PFX_BASE64**: The Base64-encoded PFX or P12 document signing certificate file.
- **PASS_PHRASE**: The passphrase for the above PFX or P12 document.

Steps to Generate Self Sign Certificate: 

```bash
  # execute below command and use passphrase 'opensign'
  openssl genrsa -des3 -out ./local_dev.key 2048
  openssl req -key ./local_dev.key -new -x509 -days 365 -out ./local_dev.crt
  openssl pkcs12 -inkey ./local_dev.key -in ./local_dev.crt -export -out ./local_dev.pfx
  openssl base64 -in ./local_dev.pfx -out ./base64_pfx
```



