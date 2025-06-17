# THUSAA Deployment

This repository manages the deployment of all THUSAA projects using a Docker-based infrastructure. It utilizes an Nginx reverse proxy with `docker-gen` for automatic service discovery and configuration.

## Core Concepts

The architecture consists of two main components run from the root `docker-compose.yml`:

1.  **Nginx Proxy (`nginx-proxy`):** The public-facing web server that handles all incoming traffic on ports 80 and 443.
2.  **Service Discovery (`docker-gen`):** A utility that monitors Docker for running containers. When a container starts, `docker-gen` uses a template (`proxy/nginx.tmpl`) to automatically generate a new Nginx configuration for it and reloads Nginx.

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
    This will start the Nginx proxy and the `docker-gen` container.

## How to Deploy a New Project

1.  **Containerize the Project:** Ensure your project has a `Dockerfile` that builds a runnable image of the application.

2.  **Create a Project Compose File:**
    Add a new `[project-name].yml` file inside the `projects/` directory (e.g., `projects/algorithmia.yml`).

3.  **Configure the Project Compose File:**
    Inside `[project-name].yml`, define your application's services. For each service that needs to be accessible from the internet, you must:
    * Add environment variables:
        * `VIRTUAL_HOST`: The domain name for this service (e.g., `app.yourdomain.com`).
        * `VIRTUAL_PORT`: The internal port the container listens on (e.g., `8000`).
        * `VIRTUAL_PATH` (Optional): If the service should only handle a specific path (e.g., `/api/`).
        * `LETSENCRYPT_HOST` (Optional): The domain name for which to obtain an SSL certificate.
    * Connect the service to the external proxy network.

    **Example Service (`projects/my-new-app.yml`):**
    ```yaml
    version: '3.8'
    services:
      my-app:
        image: my-app-image:latest
        restart: unless-stopped
        environment:
          - VIRTUAL_HOST=app.yourdomain.com
          - VIRTUAL_PORT=5000
        networks:
          - proxy-net

    networks:
      proxy-net:
        external:
          name: nginx-proxy-net
    ```

4.  **Launch the Project:**
    Navigate to the `projects/` directory and run your project's compose file:
    ```bash
    cd projects/
    docker-compose -f [project-name].yml up -d
    ```
    `docker-gen` will automatically detect the new container and configure Nginx to route traffic for `app.yourdomain.com` to it.

## Managing Deployments

-   **To stop a project:** `docker-compose -f projects/[project-name].yml down`
-   **To update a project:** Pull the latest changes in your project's code, rebuild the Docker image, and then run `docker-compose -f projects/[project-name].yml up -d --no-deps`. The `--no-deps` flag prevents it from trying to restart dependency containers.
