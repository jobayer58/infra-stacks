# Docker Infrastructure

These are my Docker stacks, designed to be deployed on `bare metal` or Portainer. The current stacks include:

* Traefik
* Infrastructure with Postgres + Redis
* Chatwoot
* Evolution API
* n8n

## Variables

Below is the list of variables required to configure each stack in Portainer. Replace the example values with your actual data.

### Traefik (Proxy & SSL)

| Variable | Example | Description |
| --- | --- | --- |
| `ACME_EMAIL` | `your@email.com` | Email for registering Let's Encrypt certificates. |
| `TRAEFIK_DOMAIN` | `traefik.rafhael.com.br` | Access domain for the Dashboard. |
| `CF_API_EMAIL` | `your@email.com` | *(Optional)* Only if using Cloudflare DNS Challenge. |
| `HOST_DATA_PATH` | `/home/rafhael/traefik_data` | Folder where `acme.json` will be saved. |

---

### Infrastructure (Postgres & Redis)

| Variable | Example | Description |
| --- | --- | --- |
| `POSTGRES_USER` | `postgres` | Master database user. |
| `POSTGRES_PASSWORD` | `YourStrongPassword123` | Master database password. |
| `POSTGRES_DB` | `postgres` | Default initial database. |
| `REDIS_PASSWORD` | `YourRedisPassword123` | Redis access password. |
| `HOST_DATA_PATH` | `/home/rafhael/infra_data` | Folder where DB and Redis data will be saved. |

---

### Chatwoot

| Variable | Example | Description |
| --- | --- | --- |
| `CHATWOOT_DOMAIN` | `chat.ravimedia.com.br` | Main Chatwoot domain. |
| `HOST_DATA_PATH` | `/home/rafhael/chatwoot_data` | Where to save uploads/files. |
| `SECRET_KEY_BASE` | `(generate a long random string)` | Rails security key. |
| `POSTGRES_HOST` | `postgres` | Database container name. |
| `POSTGRES_PASSWORD` | `YourStrongPassword123` | Same password as Infrastructure. |
| `REDIS_HOST` | `redis` | Redis container name. |
| `REDIS_PASSWORD` | `YourRedisPassword123` | Same password as Infrastructure. |
| `REDIS_URL` | `redis://:YourRedisPassword123@redis:6379` | Full connection URL. |
| `MAILER_SENDER_EMAIL` | `Chatwoot <no-reply@your.com>` | Name/Email for sending emails. |
| `SMTP_ADDRESS` | `smtp.resend.com` | SMTP Host. |
| `SMTP_USERNAME` | `resend` | SMTP User. |
| `SMTP_PASSWORD` | `re_123456` | SMTP Password. |

**Note:** Remember to point `POSTGRES_HOST` and `REDIS_HOST` to the container names of the Infra stack (e.g., `postgres` and `redis`) if they are on the same `proxy` network.

---

### Evolution API

| Variable | Example | Description |
| --- | --- | --- |
| `EVOLUTION_DOMAIN` | `m.rafhael.com.br` | API domain. |
| `HOST_DATA_PATH` | `/home/rafhael/evolution_data` | Where to save WhatsApp instances. |
| `AUTHENTICATION_API_KEY` | `YourSecretApiKey` | Global key to control the API. |
| `DATABASE_CONNECTION_URI` | `postgresql://postgres:Password@postgres:5432/evolution` | Database connection (Create the `evolution` DB beforehand). |

---

### N8N

| Variable | Example | Description |
| --- | --- | --- |
| `N8N_DOMAIN` | `n8n.ravimedia.com.br` | N8N Domain. |
| `HOST_DATA_PATH` | `/home/rafhael/n8n_data` | Where to save workflows and credentials. |
| `GENERIC_TIMEZONE` | `America/Sao_Paulo` | Timezone. |