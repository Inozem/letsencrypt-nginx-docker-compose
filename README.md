# Automated Nginx Deployment with Let's Encrypt for Azure Realtime STT

## Overview

This solution is a Docker Compose configuration that automates the deployment of Nginx with SSL certificates via Let's Encrypt and WebSocket setup for integration with Azure Realtime STT (Speech-to-Text). The solution also includes a mechanism for automatic SSL certificate renewal and Nginx reload.


## Components

1. **Nginx** — A web server configured to work with HTTPS and WebSocket support required for Azure Realtime STT.
2. **Let's Encrypt (Certbot)** — A service for automatically generating and renewing SSL certificates.
3. **Docker Compose** — A tool for managing containers, providing an easy way to deploy and manage Nginx, Certbot, and your project.


## Installation Steps

1. **Clone the repository**
   ```bash
   git clone git@github.com:Inozem/letsencrypt-nginx-docker-compose.git
   cd letsencrypt-nginx-docker-compose
   ```

2. **Insert real data**
    Before running Docker Compose, replace placeholders with your actual information:
    - Update your Nginx configuration file (e.g., in `./nginx/nginx.http.conf` and `./nginx/nginx.https.conf`) by replacing `$domain` with your actual domain:
        ```nginx
        server_name $domain;
        ssl_certificate /etc/letsencrypt/live/$domain/fullchain.pem; 
        ssl_certificate_key /etc/letsencrypt/live/$domain/privkey.pem; 
        ```

    - In the `docker-compose.yml` file, update the `certbot` command with your real email and domain:
        ```yaml
        command: >- 
                 certonly --webroot --webroot-path=/var/www/certbot
                 --email ${EMAIL} --agree-tos --no-eff-email
                 -d ${DOMAIN} --non-interactive --quiet
        ```
        > **Important:** Replace `${EMAIL}` and `${DOMAIN}` with your real email address and domain name.
    
    - Add your application containers to the `docker-compose.yml` under the `services` section. Here’s an example of how to define an application container:
        ```yaml
          web:
            build: .
            expose:
              - "5000"
            volumes:
              - .:/application
        ```

        This configuration builds your application from the current directory and exposes it on port 5000.

    - **Add a `Dockerfile` for your application.** Below is an example `Dockerfile` for a Flask application with Azure Realtime STT support:
        ```dockerfile
        FROM python:3.10-slim
        RUN apt-get update && apt-get install -y \
            build-essential \
            ca-certificates \
            libasound2-dev \
            wget \
            && wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.0g-2ubuntu4_amd64.deb \
            && dpkg -i libssl1.1_1.1.0g-2ubuntu4_amd64.deb \
            && apt-get install -y libssl-dev \
            && apt-get clean \
            && rm -rf /var/lib/apt/lists/* \
            && rm -f libssl1.1_1.1.0g-2ubuntu4_amd64.deb
        COPY application/ /application
        WORKDIR /application
        RUN pip install --no-cache-dir -r /application/requirements.txt
        CMD ["gunicorn", "-c", "/config/gunicorn_config.py", "application.wsgi:application"]
        ```

        This `Dockerfile` installs the necessary dependencies and sets up the Flask application to run using `gunicorn` with the configuration provided.

    - **Add a `gunicorn_config.py` file** to configure `gunicorn`. Here’s an example:
        ```python
        bind = "0.0.0.0:5000"
        workers = 1
        timeout = 300
        websocket_timeout = 300
        worker_connections = 1000
        limit_request_line = 4094
        limit_request_fields = 100
        limit_request_field_size = 8190
        worker_class = "gevent"
        ```

        This configuration optimizes `gunicorn` for handling WebSocket connections and long requests typical of Azure Realtime STT applications.

3. **Start the containers**
    Once the real data is inserted, the Nginx settings are reviewed and modified, and your project container is added, run the following Docker Compose command to start the Nginx, Certbot, and your project containers:
    ```bash
    docker-compose up -d
    ```


## SSL Certificate Renewal and Nginx Reload

To automate SSL certificate renewal and Nginx reload every 2 months at 3 AM, follow these steps:

1. **Open the cron editor**:
   ```bash
   crontab -e
   ```

2. **Add the following cron job** to renew SSL certificates and reload Nginx every 2 months at 3 AM:
   ```bash
   0 3 1 */2 * docker-compose -f /path/to/your/docker-compose.yml run --rm certbot certonly --reinstall --webroot --webroot-path=/var/www/certbot --email ${EMAIL} --agree-tos --no-eff-email -d ${DOMAIN} --force-renewal && docker-compose -f /path/to/your/docker-compose.yml restart nginx_https
   ```
   > **Important:** Replace `${EMAIL}` and `${DOMAIN}` with your real email address and domain name. Replace `/path/to/your/docker-compose.yml` with the actual path to your `docker-compose.yml` file.


## How It Works

### Docker Compose

This setup uses Docker Compose to manage three main services: `nginx_http`, `certbot`, and `nginx_https`. Here’s a step-by-step explanation of how it works:

1. **nginx_http**:
   - This is the first service that runs when the containers are started. It uses the `nginx` web server (specifically the lightweight Alpine version) to handle HTTP (port 80) requests.
   - This service is responsible for serving a simple web page and redirecting traffic to HTTPS once the SSL certificate is available.
   - It mounts two volumes:
     - `./nginx/nginx.http.conf` – a configuration file for Nginx.
     - `certbot_data` – a shared volume used by Certbot to store temporary files during certificate generation.

2. **certbot**:
   - Certbot is a tool provided by Let’s Encrypt to generate SSL certificates. In this setup, it runs in a separate container.
   - The command used (`certonly --webroot ...`) tells Certbot to generate the SSL certificate using a webroot method, which means it will place verification files in the `certbot_data` volume to prove ownership of the domain.
   - Certbot uses your email (`${EMAIL}`) and domain (`${DOMAIN}`) from environment variables to register and request the certificate.
   - Once Certbot successfully generates the SSL certificate, the certificate files are stored in the `./letsencrypt` directory, which is mounted as a volume.
   - This service depends on the `nginx_http` service to be running since it needs to use the HTTP server for verification.

3. **nginx_https**:
   - Once Certbot successfully obtains the SSL certificate, the `nginx_https` service is started.
   - This service also uses the `nginx` web server, but this time, it’s configured to serve content over HTTPS (port 443).
   - The `nginx_https` service uses the SSL certificate files stored in the `./letsencrypt` directory to enable secure HTTPS connections.
   - It depends on the `certbot` service, meaning it will only start after Certbot successfully generates the certificate.
   - Like the `nginx_http` service, it mounts the necessary configuration and certificate files using volumes.

4. **Volumes**:
   - `certbot_data` is a shared volume used by both the `nginx_http` and `certbot` containers. It allows Certbot to place its verification files in a location accessible by the Nginx HTTP server during the SSL verification process.
   - The `./letsencrypt` directory stores the SSL certificates generated by Certbot, which are shared between the `certbot` and `nginx_https` services.


### SSL Certificate Renewal and Nginx Reload

In addition to obtaining SSL certificates, this setup also supports automated certificate renewal and Nginx reload. Every 2 months at 3 AM, a cron job is executed to renew the SSL certificates and restart Nginx if the certificates are updated.

1. **Cron Job for SSL Certificate Renewal**:
   - A cron job is scheduled to run every 2 months to renew the SSL certificates:
     ```bash
     0 3 1 */2 * docker-compose -f /path/to/your/docker-compose.yml run --rm certbot certonly --reinstall --webroot --webroot-path=/var/www/certbot --email ${EMAIL} --agree-tos --no-eff-email -d ${DOMAIN} --force-renewal && docker-compose -f /path/to/your/docker-compose.yml restart nginx_https
     ```

   - The cron job runs the `certbot certonly` command with the `--force-renewal` flag to ensure the SSL certificates are renewed if needed.
   - After renewing the certificates, Nginx is restarted using `docker-compose restart nginx_https` to apply the new certificates.

2. **How it Works**:
   - The cron job is scheduled to run at 3:00 AM on the first day of every second month.
   - It automatically checks if the SSL certificate needs renewal, renews it if necessary, and then reloads Nginx to apply the renewed certificate.
   - This ensures that your website always uses up-to-date SSL certificates without any manual intervention.

### Summary of the Process:
- Initially, `nginx_http` serves content over HTTP.
- Certbot runs and uses `nginx_http` to verify domain ownership and generate SSL certificates.
- Once Certbot successfully generates the SSL certificates, the `nginx_https` service starts and begins serving traffic securely over HTTPS.
- The cron job ensures that SSL certificates are automatically renewed every 2 months, and Nginx is reloaded to apply the updated certificates.

This automated process ensures your web server is always running with secure HTTPS and up-to-date SSL certificates, with the help of Let’s Encrypt and Docker Compose.


## Conclusion
This solution provides a simple and effective way to deploy Nginx with HTTPS and WebSocket support for Azure Realtime STT, along with easy integration of your project. With Docker Compose, you can easily renew SSL certificates and automatically reload Nginx.
