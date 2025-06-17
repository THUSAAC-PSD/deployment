# THUSAA Deployment

This repository manages the deployment of all THUSAA projects using a Docker-based infrastructure. It utilizes an Nginx reverse proxy with `docker-gen` for automatic service discovery and configuration, and `Certbot` for SSL certificate management.

## Core Concepts

The architecture consists of three main components run from the root `docker-compose.yml`:

1.  **Nginx Proxy (`nginx-proxy`):** The public-facing web server that handles all incoming traffic on ports 80 and 443.
2.  **Service Discovery (`docker-gen`):** A utility that monitors Docker for running containers. When a container starts, `docker-gen` uses a template (`proxy/nginx.tmpl`) to automatically generate a new Nginx configuration for it and reloads Nginx.
3.  **Certbot (`certbot`):** Manages SSL certificates, primarily for renewal. Initial certificate acquisition for new domains may require a manual step.

All individual projects (like `algorithmia`) are managed by their own Docker Compose files located in the `projects/` directory.

## Prerequisites

- Docker
- Docker Compose

## First-Time Setup

1.  **Create the Shared Docker Network:**
    This network allows the proxy to communicate with the project containers. You only need to do this once.
    ```bash
    docker network create thusaa-nginx-proxy-net
    ```

2.  **Start the Proxy Services:**
    From the root of this repository, run:
    ```bash
    docker-compose up -d
    ```
    This will start the Nginx proxy, the `docker-gen` container, and the `certbot` service for certificate renewals.

## How to Deploy a New Project

1.  **Containerize the Project:** Ensure your project has a `Dockerfile` that builds a runnable image of the application.

2.  **Create Project Directory and Compose File:**
    Create a new directory for your project within `projects/` (e.g., `projects/my-new-app/`).
    Inside this directory, add a `docker-compose.yml` file (e.g., `projects/my-new-app/docker-compose.yml`).

3.  **Configure the Project:**
    For each service in your project's `docker-compose.yml` that needs to be accessible from the internet, you must:
    *   Create `.proxy.[service-name].env` files (e.g., `projects/my-new-app/.proxy.frontend.env`) to store Nginx-related environment variables.
        *   `VIRTUAL_HOST`: The domain name for this service (e.g., `app.yourdomain.com`).
        *   `VIRTUAL_PORT`: The internal port the container listens on (e.g., `80` or `8000`).
        *   `VIRTUAL_PATH` (Optional): If the service should only handle a specific path (e.g., `/api/`). If omitted, defaults to `/`.
        *   `LETSENCRYPT_HOST` (Optional): The domain name for which an SSL certificate should be used/obtained. Typically the same as `VIRTUAL_HOST`. Certs must be named `VIRTUAL_HOST.crt` and `VIRTUAL_HOST.key` and placed in `./proxy/certs` (managed by Certbot).
    *   In your project's `docker-compose.yml`, use `env_file` to load these proxy configuration files for the respective services.
    *   Connect the service to the external proxy network `thusaa-nginx-proxy-net`.
    *   You may also have a general `.env` file for application-specific environment variables.

    **Example Project Structure (`projects/my-web-app/`):**

    *   `projects/my-web-app/docker-compose.yml`:
        ```yaml
        services:
          frontend:
            image: my-frontend-image:latest # Replace with your actual frontend image
            restart: unless-stopped
            env_file:
              - .env # For application-specific variables
              - .proxy.frontend.env # For Nginx proxy variables
            networks:
              - shared_proxy_net # Connects to the external proxy network

          backend:
            image: my-backend-image:latest # Replace with your actual backend image
            restart: unless-stopped
            env_file:
              - .env # For application-specific variables
              - .proxy.backend.env # For Nginx proxy variables
            networks:
              - shared_proxy_net # Connects to the external proxy network
            # Add other configurations like ports (not exposed directly), volumes, etc.

        networks:
          shared_proxy_net: # Alias used by services in this compose file
            external: true
            name: thusaa-nginx-proxy-net # Must match the actual created network name
        ```

    *   `projects/my-web-app/.proxy.frontend.env`:
        ```env
        VIRTUAL_HOST=my-app.yourdomain.com
        VIRTUAL_PORT=80
        LETSENCRYPT_HOST=my-app.yourdomain.com
        ```

    *   `projects/my-web-app/.proxy.backend.env`:
        ```env
        VIRTUAL_HOST=my-app.yourdomain.com
        VIRTUAL_PORT=8080
        VIRTUAL_PATH=/api/
        LETSENCRYPT_HOST=my-app.yourdomain.com
        ```

    *   `projects/my-web-app/.env` (Example for application variables):
        ```env
        DATABASE_URL=postgres://user:pass@host:port/db
        API_KEY=your_api_key_here
        ```

4.  **Launch the Project:**
    Ensure you have created the necessary `.env` and `.proxy.*.env` files for your project.
    Navigate to your project's directory (e.g., `projects/my-web-app/`) and run its compose file:
    ```bash
    cd projects/my-web-app/
    docker-compose up -d
    ```
    `docker-gen` will automatically detect the new container(s) and configure Nginx to route traffic accordingly. If `LETSENCRYPT_HOST` is set and valid certificates (named `VIRTUAL_HOST.crt/key`) exist in the proxy's certificate directory, SSL will be enabled.

## Managing Deployments

-   **To stop a project:** Navigate to the project's directory (e.g., `projects/my-web-app/`) and run:
    ```bash
    docker-compose down
    ```
-   **To update a project:**
    1.  Pull the latest changes in your project's code.
    2.  Rebuild your Docker image(s) if necessary (e.g., `docker build -t my-frontend-image:latest ./frontend_code_dir`).
    3.  Navigate to the project's directory and run:
        ```bash
        docker-compose up -d --no-deps [service-name] # To update specific services
        # OR to pull new images and recreate services
        docker-compose pull # If images are hosted on a registry
        docker-compose up -d --force-recreate --no-deps # To recreate all services in the project
        ```
    The `--no-deps` flag prevents it from affecting other potentially shared dependency containers if not desired. Use `--force-recreate` to ensure containers are updated with new images or configurations.

## SSL Certificates (Let's Encrypt)

-   The `certbot` service in the root `docker-compose.yml` is configured to automatically renew existing SSL certificates.
-   For **new domains** specified in `LETSENCRYPT_HOST` / `VIRTUAL_HOST`, you may need to manually obtain the initial certificate. This typically involves running a `certbot certonly` command. Ensure the certificates (`fullchain.pem` and `privkey.pem`) are placed in `./proxy/certs` and named `VIRTUAL_HOST.crt` and `VIRTUAL_HOST.key` respectively. For example, `certbot` might generate `yourdomain.com/fullchain.pem`; you would copy/symlink it to `./proxy/certs/yourdomain.com.crt`.
-   The Nginx template (`proxy/nginx.tmpl`) will automatically enable SSL for services if it finds corresponding `.crt` and `.key` files for the `VIRTUAL_HOST`.
