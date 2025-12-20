# Laravel Production Stack

A Docker stack optimized for **Production Deployment** of Laravel applications.

This stack utilizes the **Nginx Sidecar + PHP-FPM** architecture. This ensures better performance, enhanced security, and native integration with the Traefik Reverse Proxy.

## Key Features

| Feature | Laravel Sail (Standard) | This Stack (Production) |
| :--- | :--- | :--- |
| **Web Server** | Built-in Server / Supervisord | **Nginx + PHP-FPM** (Industry Standard) |
| **SSL/HTTPS** | Manual configuration | **Automatic** via Traefik (Labels included) |
| **Dependencies** | Installs everything (Dev + Prod) | Installs only `no-dev` (Lightweight & Secure) |
| **Networking** | Simple local ports | Isolated internal network for DB & Cache |

## Architecture

* **App (PHP-FPM):** Container holding the application code and dependencies.
* **Web (Nginx):** Web server that serves static files and forwards PHP requests to the `app` container.
* **Database (PostgreSQL):** Persistent relational database.
* **Cache (Redis):** Handles application caching, queues, and sessions.

## How to Use

### 1. Configure Environment Variables
Copy the example file and configure your secrets and domain.

```bash
cp .env.example .env
```

> **Note:** The DOMAIN_NAME variable defines the host where Traefik will expose your application (e.g., app.yourdomain.com).

### 2. Start the Application
Since this is a production stack, if you aren't on Portainer, it is recommended to build the images to ensure the latest code is used:

```bash
docker compose up -d --build
```

### 3. Initial Setup (First Run Only)
Because the environment runs in production mode (APP_ENV=production), development dependencies are not installed. You need to manually trigger the setup commands inside the container:

```bash
# 1. Install production dependencies
docker compose exec app composer install --optimize-autoloader --no-dev

# 2. Generate Application Key
docker compose exec app php artisan key:generate

# 3. Run Database Migrations
docker compose exec app php artisan migrate --force

# 4. Optimize Caches
docker compose exec app php artisan config:cache
docker compose exec app php artisan route:cache
docker compose exec app php artisan view:cache
```

### Using Portainer (GUI)
If you are managing this stack via Portainer, you don't need SSH access. You can run the setup commands directly from your browser:

1. Open your Stack list and click on the app container (e.g., laravel_prod_app-1).
2. Click the Console (>_) icon.
3. Crucial Step: In the "User" field, type www-data (this ensures files created have the correct permissions).
4. Select /bin/bash and click Connect.

Paste the commands below into the terminal window _(line per line)_:

```bash
composer install --optimize-autoloader --no-dev
php artisan key:generate
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
```

## Traefik Integration
The docker-compose.yml file includes the necessary labels for automatic service discovery. No manual Nginx configuration is required on the host.

Configuration Snippet:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.laravel-app.rule=Host(`${DOMAIN_NAME}`)"
  - "traefik.http.routers.laravel-app.entrypoints=websecure"
  - "traefik.http.routers.laravel-app.tls.certresolver=myresolver"
```

Once the container starts, Traefik will detect the service, generate a Let's Encrypt SSL certificate, and start routing traffic immediately.