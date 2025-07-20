# FastAPI Project - Local Development Setup with HTTPS

This document guides you through setting up the project on your local machine for development. The goal is to run the entire stack using Docker Compose and have all services accessible via secure `https://` URLs (e.g., `https://dashboard.fastapi-project.local`) without any browser warnings.

This setup uses **Traefik** as a reverse proxy and **`mkcert`** to generate locally-trusted SSL certificates.

### **Prerequisites**

Before starting, ensure you have the following installed and working:

*   **Docker & Docker Compose:** To build and run the containerized services.
*   **[mkcert](https://github.com/FiloSottile/mkcert#installation):** A simple tool for making locally-trusted development certificates.
*   **Administrative/Sudo Access:** Required to edit your `hosts` file and install the `mkcert` root certificate.

---

### **Step 1: Initial Host and Certificate Configuration**

You only need to perform these steps once per development machine.

1.  **Configure Your `hosts` File for Local DNS:**
    This maps the project's domains to your local machine. You will need administrator privileges to edit this file.

    *   **Linux/macOS:** `/etc/hosts`
    *   **Windows:** `C:\Windows\System32\drivers\etc\hosts`

    Add the following lines to the file and save it:
    ```
    127.0.0.1 traefik.fastapi-project.local
    127.0.0.1 dashboard.fastapi-project.local
    127.0.0.1 api.fastapi-project.local
    127.0.0.1 adminer.fastapi-project.local
    127.0.0.1 dashboard.staging.fastapi-project.local
    127.0.0.1 api.staging.fastapi-project.local
    127.0.0.1 adminer.staging.fastapi-project.local
    ```

2.  **Create and Install a Local Certificate Authority (CA):**
    This command installs a local root CA in your system's trust stores, which makes your browser trust the certificates you are about to create.
    ```bash
    # This may prompt for your administrator password
    mkcert -install
    ```

3.  **Create the Shared Docker Network:**
    Our services communicate over a custom Docker network. Create it by running:
    ```bash
    docker network create traefik-public
    ```

---

### **Step 2: Project-Specific Setup**

Perform these steps inside your cloned project directory.

1.  **Generate SSL Certificates for Project Domains:**
    This command creates the `cert.pem` and `key.pem` files that Traefik will serve.

    First, create the directory:
    ```bash
    mkdir -p traefik/certs
    ```
    Then, generate the certificates:
    ```bash
    mkcert -cert-file traefik/certs/local-cert.pem -key-file traefik/certs/local-key.pem \
      traefik.fastapi-project.local \
      dashboard.fastapi-project.local \
      api.fastapi-project.local \
      adminer.fastapi-project.local \
      dashboard.staging.fastapi-project.local \
      api.staging.fastapi-project.local \
      adminer.staging.fastapi-project.local
    ```

2.  **Prepare Environment Files:**
    This project uses two environment files.

    *   **Main App Config (`.env`):** Copy the example file. No changes are needed for a basic local setup.
        ```bash
        cp .env.example .env
        ```
    *   **Traefik Config (`.env.traefik`):** Create this file for Traefik's credentials. **The `$$` to escape the password hash is critical.**
        ```bash
        cat <<EOF > .env.traefik
        DOMAIN=fastapi-project.local
        USERNAME=admin
        PASSWORD=acs@1234
        HASHED_PASSWORD=\$\$apr1\$\$X3.oP6FF\$\$ghWQhDA64girVBrbmfIPk.
        EMAIL=admin@fastapi.dev
        EOF
        ```

3.  **Configure Docker Compose Files:**
    You must modify the Docker Compose files to use your local certificates instead of the default Let's Encrypt setup.

    *   **Create `traefik/dynamic_conf.yml`:** This file tells Traefik where to find your certificates *inside its container*.
        ```bash
        cat <<EOF > traefik/dynamic_conf.yml
        tls:
          certificates:
            - certFile: /certs/local-cert.pem
              keyFile: /certs/local-key.pem
        EOF
        ```
    *   **Modify `docker-compose.traefik.yml`:** Replace its entire content with the configuration for local development. This new version mounts your local certs and tells Traefik to read the `dynamic_conf.yml` file. (See Appendix for the full file).

    *   **Modify `docker-compose.yml`:** In this file, you must **remove the Let's Encrypt certificate resolver** from each service exposed to Traefik. For the `adminer`, `backend`, and `frontend` services, find and **delete the following line** from their `labels` section:
        ```yaml
        - traefik.http.routers.${STACK_NAME?Variable not set}-...-https.tls.certresolver=le
        ```

### **Step 3: Launch the Application**

With all the configuration in place, you can now launch the entire stack with a single command. This command provides both environment files to Docker Compose.

```bash
docker-compose --env-file .env --env-file .env.traefik -f docker-compose.yml -f docker-compose.traefik.yml up -d --build
```

### **Step 4: Verify Your Setup**

Once the containers are running, open your browser and navigate to the following URLs. Each should load with a secure lock icon and no warnings.

*   **Application Frontend:** `https://dashboard.fastapi-project.local`
*   **Backend API Docs:** `https://api.fastapi-project.local/docs`
*   **Database Manager:** `https://adminer.fastapi-project.local`
*   **Traefik Dashboard:** `https://traefik.fastapi-project.local`
    *   *(Login with `admin` / `acs@1234`)*

---
### **Appendix: Final `docker-compose.traefik.yml` for Local Development**

Replace the entire contents of your `docker-compose.traefik.yml` with this:

```yaml
services:
  traefik:
    image: traefik:3.0
    ports:
      - 80:80
      - 443:443
    restart: always
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.http.services.traefik-dashboard.loadbalancer.server.port=8080
      - traefik.http.routers.traefik-dashboard-http.entrypoints=http
      - traefik.http.routers.traefik-dashboard-http.rule=Host(`traefik.${DOMAIN?Variable not set}`)
      - traefik.http.routers.traefik-dashboard-https.entrypoints=https
      - traefik.http.routers.traefik-dashboard-https.rule=Host(`traefik.${DOMAIN?Variable not set}`)
      - traefik.http.routers.traefik-dashboard-https.tls=true
      - traefik.http.routers.traefik-dashboard-https.service=api@internal
      - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
      - traefik.http.routers.traefik-dashboard-http.middlewares=https-redirect
      - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
      - traefik.http.routers.traefik-dashboard-https.middlewares=admin-auth
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/certs:/certs:ro
      - ./traefik/dynamic_conf.yml:/etc/traefik/dynamic_conf.yml:ro
    command:
      - --providers.docker
      - --providers.docker.exposedbydefault=false
      - --providers.file.filename=/etc/traefik/dynamic_conf.yml
      - --entrypoints.http.address=:80
      - --entrypoints.https.address=:443
      - --accesslog
      - --log
      - --api
    networks:
      - traefik-public

volumes:
  traefik-public-certificates:

networks:
  traefik-public:
    external: true
```