# Stacks Catalog

Here you will find all the independent services available in this repository. Each stack is self-contained and includes its own configuration guide.

| Stack | Category | Description |
| :--- | :--- | :--- |
| **[Monitoring](./monitoring)** | Observability | Prometheus, Grafana, Node Exporter & cAdvisor. |
| **[Traefik](./traefik)** | Networking | Cloud-native reverse proxy and load balancer with auto SSL. |
| **[n8n](./n8n)** | Automation | Workflow automation tool (fair-code licensed). |
| **[Evolution API](./evolution-api)** | Messaging | WhatsApp API for managing messages and bots. |
| **[Chatwoot](./chatwoot)** | Support | Open-source customer engagement suite. |
| **[Affine](./affine)** | Productivity | Next-gen knowledge base (Notion/Miro alternative). |
| **[Infra DB](./infra-db)** | Database | Shared PostgreSQL & Redis containers. |
| **[Laravel](./laravel)** | Web | Laravel Production Setup |

## ⚡ Quick Tips
- **Dependencies:** Some stacks _(like Chatwoot and n8n)_ depend on **Infra DB**. Deploy that first.
- **Databases:** You must manually create the specific databases for each service.
  ```bash
  # Example: Creating the database for n8n for user name postgres
  docker exec -it infra-db-postgres-1 createdb -U postgres n8n_db
  ```
- **Networking:** All stacks expect a `proxy` external network to exist for Traefik routing.