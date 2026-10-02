# Omni-Suites Application Stack (`infra/apps`)

Central Docker Compose orchestration for the **Omni-Suites** core microservices, databases, and frontend application on the staging virtual machine. This stack operates behind the central [Traefik Edge Reverse Proxy](../traefik) via a dual-network topology.

---

## 🏗️ Architecture & Network Topology

The application stack uses two isolated Docker bridge networks:
1. **`edge` (External)**: Shared with the central Traefik instance. Traefik terminates TLS, handles HTTPS routing, and forwards public requests directly to HTTP-facing backend containers (`omni-client`, `order-service`, `inventory-service`, `notification-service`, `omni-integration`). Databases are **not** attached to `edge`, keeping them inaccessible from the public reverse proxy.
2. **`apps-internal` (Private)**: Internal service-to-service communication and database connectivity. All microservices communicate with their designated persistence stores and peer RPC endpoints over Docker internal DNS.

```mermaid
flowchart TD
    subgraph Public Internet
        Browser["User Browser / Client"]
        Linear["Linear Webhooks"]
        Discord["Discord Interactions API"]
    end

    subgraph "Edge Network (Traefik Reverse Proxy)"
        Traefik["Traefik v3.7 (:80 / :443)"]
    end

    Browser -->|frontend-svc.test-suites-poc.work.gd| Traefik
    Browser -->|order-svc.test-suites-poc.work.gd| Traefik
    Browser -->|inventory-svc.test-suites-poc.work.gd| Traefik
    Browser -->|notification-svc.test-suites-poc.work.gd| Traefik
    Linear -->|hooks.test-suites-poc.work.gd/webhooks/linear| Traefik
    Discord -->|hooks.test-suites-poc.work.gd/api/discord/interactions| Traefik

    subgraph "Apps Stack (infra/apps)"
        subgraph "Application Layer (Edge + Apps-Internal)"
            Client["omni-client (:80)<br/>SPA Static Server"]
            OrderSvc["order-service (:3000)<br/>NestJS"]
            InvSvc["inventory-service (:3001)<br/>NestJS"]
            NotifSvc["notification-service (:3002)<br/>NestJS"]
            IntegrationSvc["omni-integration (:3003)<br/>NestJS"]
        end

        subgraph "Persistence Layer (Apps-Internal Only)"
            PostgresDB[("postgres-db (:5432)<br/>order_db & omni_integration_db")]
            MysqlDB[("mysql-db (:3306)<br/>inventory_db")]
            MongoDB[("mongo-db (:27017)<br/>notification_db (Replica Set rs0)")]
        end
    end

    Traefik -->|HTTP :80| Client
    Traefik -->|HTTP :3000| OrderSvc
    Traefik -->|HTTP :3001| InvSvc
    Traefik -->|HTTP :3002| NotifSvc
    Traefik -->|HTTP :3003| IntegrationSvc

    Client -.->|Browser API Calls via Public URLs| Traefik

    OrderSvc -->|HTTP RPC :3001| InvSvc
    OrderSvc -->|HTTP RPC :3002| NotifSvc
    OrderSvc -->|SQL| PostgresDB
    InvSvc -->|SQL| MysqlDB
    NotifSvc -->|Mongo Protocol| MongoDB
    IntegrationSvc -->|SQL| PostgresDB

    subgraph External Systems
        SquashTM["squash-tm (:8080/squash)<br/>(Edge Network)"]
        GitHub["GitHub Actions API"]
    end

    IntegrationSvc -->|REST API| SquashTM
    IntegrationSvc -->|Workflow Dispatch| GitHub
```

---

## 📦 Service Inventory & Specifications

| Service | Container Name | Image / Source | Internal Port | Networks | Data Volume | Dependencies / Healthchecks |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`postgres-db`** | `postgres-db` | `postgres:15-alpine` | `5432` | `apps-internal` | `postgres-data` | `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}` |
| **`mysql-db`** | `mysql-db` | `mysql:8.0` | `3306` | `apps-internal` | `mysql-data` | `mysqladmin ping -uroot -p${MYSQL_ROOT_PASSWORD}` |
| **`mongo-db`** | `mongo-db` | `mongo:6-jammy` | `27017` | `apps-internal` | `mongo-data` | `mongosh --eval "db.adminCommand('ping')"` (Auto-initializes replica set `rs0`) |
| **`order-service`** | `order-service` | `${GCP_AR_REPO}/order-service:staging` | `3000` | `apps-internal`, `edge` | — | Depends on `postgres-db` (healthy), `inventory-service`, `notification-service`. Runs `prisma migrate deploy` on boot. |
| **`inventory-service`** | `inventory-service` | `${GCP_AR_REPO}/inventory-service:staging` | `3001` | `apps-internal`, `edge` | — | Depends on `mysql-db` (healthy). Runs `prisma migrate deploy` and `seed.cjs` on boot. |
| **`notification-service`** | `notification-service` | `${GCP_AR_REPO}/notification-service:staging` | `3002` | `apps-internal`, `edge` | — | Depends on `mongo-db` (healthy). Runs `prisma db push` on boot. |
| **`omni-client`** | `omni-client` | `${GCP_AR_REPO}/omni-client:staging` | `80` | `edge` | — | Depends on `order-service`, `inventory-service`, `notification-service`. Injects runtime env into `/app/dist/config.js`. |
| **`omni-integration`** | `omni-integration` | `${GCP_AR_REPO}/omni-integration:staging` | `3003` | `apps-internal`, `edge` | — | Depends on `postgres-db` (healthy). Connects to Linear, Squash TM, Discord, and GitHub Actions. |

---

## 🌐 Public Hostname Routing (Traefik Ingress)

All public routing rules are managed externally in [`../traefik/dynamic/routes.yml`](../traefik/dynamic/routes.yml):

| Service | Public Ingress Hostname | Target Backend |
| :--- | :--- | :--- |
| **Frontend Web App** | `https://frontend-svc.test-suites-poc.work.gd` | `http://omni-client:80` |
| **Order API** | `https://order-svc.test-suites-poc.work.gd` | `http://order-service:3000` |
| **Inventory API** | `https://inventory-svc.test-suites-poc.work.gd` | `http://inventory-service:3001` |
| **Notification API** | `https://notification-svc.test-suites-poc.work.gd` | `http://notification-service:3002` |
| **Integration & Webhooks** | `https://hooks.test-suites-poc.work.gd` | `http://omni-integration:3003` |

---

## ⚙️ Environment Configuration (`.env`)

Copy the provided sample file to `.env` on the staging host:
```bash
cp .env.sample .env
```

### Complete Variable Reference

| Variable Group | Variable Name | Required | Description / Default Example |
| :--- | :--- | :---: | :--- |
| **Artifact Registry** | `GCP_AR_REPO` | **Yes** | Prefix for Docker images, e.g. `us-central1-docker.pkg.dev/omni-suites/omni-suites`. |
| **PostgreSQL** | `POSTGRES_USER` | **Yes** | Postgres superuser (e.g. `user`). |
| | `POSTGRES_PASSWORD` | **Yes** | Postgres superuser password. |
| | `POSTGRES_DB` | **Yes** | Default database initialized on container creation (`order_db`). |
| **MySQL** | `MYSQL_ROOT_PASSWORD` | **Yes** | Root password for MySQL 8 instance. |
| | `MYSQL_DATABASE` | **Yes** | Database name initialized for inventory (`inventory_db`). |
| **Database URLs** | `ORDER_DATABASE_URL` | **Yes** | `postgresql://user:password@postgres-db:5432/order_db?schema=public` |
| | `INVENTORY_DATABASE_URL` | **Yes** | `mysql://root:password@mysql-db:3306/inventory_db` |
| | `NOTIFICATION_DATABASE_URL` | **Yes** | `mongodb://mongo-db:27017/notification_db?replicaSet=rs0` |
| | `INTEGRATION_DATABASE_URL` | **Yes** | `postgresql://user:password@postgres-db:5432/omni_integration_db?schema=public` |
| **Inter-Service URLs** | `INVENTORY_SERVICE_URL` | No | Internal Docker URL for Order Service: `http://inventory-service:3001` |
| | `NOTIFICATION_SERVICE_URL` | No | Internal Docker URL for Order Service: `http://notification-service:3002` |
| **Frontend Public Endpoints** | `VITE_API_ORDER_URL` | **Yes** | `https://order-svc.test-suites-poc.work.gd` |
| | `VITE_API_INVENTORY_URL` | **Yes** | `https://inventory-svc.test-suites-poc.work.gd` |
| | `VITE_API_NOTIFICATION_URL` | **Yes** | `https://notification-svc.test-suites-poc.work.gd` |
| **Linear Integration** | `LINEAR_WEBHOOK_SECRET` | **Yes** | Webhook verification secret from Linear team settings. |
| | `LINEAR_TEAM_NAME` | **Yes** | Name of the Linear team (e.g. `omni-suites`). |
| | `LINEAR_TEAM_ID` | No | Optional UUID of the Linear team. |
| | `LINEAR_READY_LABEL` | **Yes** | Label triggering test sync (e.g. `ready-for-tc`). |
| **Squash TM Integration** | `SQUASH_BASE_URL` | **Yes** | Path must include `/squash`: `http://squash-tm:8080/squash` (Docker network) or `https://squash-tcms.test-suites-poc.work.gd/squash`. |
| | `SQUASH_API_TOKEN` | Optional | JWT bearer token for Squash TM REST API. |
| | `SQUASH_USER` / `SQUASH_PASSWORD` | Optional | Basic authentication credentials (if token is not used). |
| | `SQUASH_PROJECT_NAME` | **Yes** | Target project name in Squash TM (e.g. `omni-suites`). |
| | `SQUASH_PROJECT_ID` | Optional | Target project numeric identifier in Squash TM. |
| **Discord Bot** | `DISCORD_APPLICATION_ID` | **Yes** | Discord Bot Application ID. |
| | `DISCORD_PUBLIC_KEY` | **Yes** | Discord Bot Ed25519 Public Key for interaction signature verification. |
| | `DISCORD_BOT_TOKEN` | **Yes** | Discord Bot authentication token. |
| | `DISCORD_GUILD_ID` | **Yes** | Discord Server / Guild ID where slash commands (`/run-tests`, `/run-perf`) are registered. |
| | `DISCORD_TEST_RELEASES_CHANNEL_ID` | **Yes** | Target Discord Channel ID for CI/CD test notifications and report embeds. |
| **GitHub Automation** | `DISCORD_GITHUB_PAT_TOKEN` | **Yes** | GitHub Personal Access Token (Classic with `repo` and `workflow` scopes) allowing Discord bot to trigger `workflow_dispatch`. |

---

## 🗄️ Database Initialization & Multi-Tenancy

### 1. Dual-Database on PostgreSQL (`order_db` & `omni_integration_db`)
The `postgres-db` service automatically creates `order_db` on first boot via the `POSTGRES_DB` environment variable. `omni_integration_db` shares the same PostgreSQL container. If creating the cluster on a brand-new volume, initialize `omni_integration_db` manually:

```bash
docker exec -i postgres-db psql -U user -d postgres -c "CREATE DATABASE omni_integration_db;"
```

*Note: Once created, `omni-integration` will automatically run its Prisma migrations on boot.*

### 2. MongoDB Replica Set (`rs0`)
Prisma's MongoDB connector requires a replica set to support transactions and atomic writes. The `mongo-db` container runs a customized entrypoint that automatically initializes a single-node replica set (`rs0`) if not already initialized:
```bash
mongod --replSet rs0 --bind_ip_all & MONGOD_PID=$!; sleep 4; mongosh --eval 'rs.initiate({_id: "rs0", members: [{_id: 0, host: "mongo-db:27017"}]})' || true; wait $MONGOD_PID
```
The connection string **must** contain `?replicaSet=rs0`.

### 3. MySQL & Inventory Auto-Seeding
When `mysql-db` passes its healthcheck (`mysqladmin ping`), `inventory-service` boots, executes `npx prisma migrate deploy`, and then automatically executes `node prisma/seed.cjs` to populate initial product inventory.

---

## 🚀 Operations & Deployment Guide

### Prerequisites
1. **Traefik must be running first**:
   The `edge` network is created by `infra/traefik/docker-compose.yml`. Ensure Traefik is running:
   ```bash
   cd /opt/omni-infra/traefik
   docker compose up -d
   ```
2. **Authenticate with Google Artifact Registry**:
   Ensure Docker on the VM has credentials to pull staging images from GCP Artifact Registry:
   ```bash
   gcloud auth configure-docker us-central1-docker.pkg.dev
   ```

### Full Stack Lifecycle Commands

```bash
# Navigate to stack directory
cd /opt/omni-infra/apps

# 1. Pull the latest staging images from GCP Artifact Registry
docker compose pull

# 2. Start all services in detached mode
docker compose up -d

# 3. Verify container status and healthchecks
docker compose ps

# 4. View unified logs across all services
docker compose logs -f

# 5. Stop the stack (preserves volumes)
docker compose down
```

### Targeted Single-Service Deployments (CI/CD)

Each microservice repository features a `.github/workflows/deploy-staging.yaml` workflow that updates only that individual service without restarting the entire stack:

```bash
# Example: Deploying an updated order-service
docker compose pull order-service
docker compose up -d --no-deps order-service
```

### Checking Specific Service Logs

```bash
docker compose logs -f order-service
docker compose logs -f omni-integration
docker compose logs -f omni-client
```

---

## 🔍 Troubleshooting & Diagnostics

### Common Failure Modes & Resolutions

| Issue | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`network edge not found`** | Traefik stack has not been initialized yet. | Start Traefik first: `cd /opt/omni-infra/traefik && docker compose up -d`. |
| **`database "omni_integration_db" does not exist`** | PostgreSQL only auto-creates the primary `POSTGRES_DB` (`order_db`). | Execute: `docker exec -i postgres-db psql -U $POSTGRES_USER -d postgres -c "CREATE DATABASE omni_integration_db;"`, then restart `omni-integration`. |
| **`MongoServerError: Not master / replica set not initialized`** | MongoDB started before replica set initiation completed. | Check logs: `docker compose logs mongo-db`. Manually verify: `docker exec -it mongo-db mongosh --eval "rs.status()"`. |
| **Frontend showing `Network Error` calling APIs** | Browser trying to connect to local hostnames instead of public Traefik endpoints. | Ensure `VITE_API_ORDER_URL`, `VITE_API_INVENTORY_URL`, and `VITE_API_NOTIFICATION_URL` in `.env` are set to the full public HTTPS domains and restart `omni-client`: `docker compose up -d --force-recreate omni-client`. |
| **`permission denied while trying to connect to Docker daemon`** | Runner or user not in the `docker` group. | Run `sudo usermod -aG docker $USER` and log back into the shell. |
| **Image pull error from Artifact Registry** | Expired or missing GCP Docker credentials. | Re-authenticate runner: `gcloud auth configure-docker us-central1-docker.pkg.dev` or verify VM service account permissions. |

---

## 🛡️ Security Best Practices
- **Network Isolation:** No database ports (`5432`, `3306`, `27017`) are mapped to the host or the `edge` network. They are accessible strictly over `apps-internal`.
- **Secrets Management:** Never commit `.env` containing production or staging secrets to version control. The `.gitignore` in this directory prevents `.env` commits.
- **Webhook Signature Verification:** `omni-integration` strictly validates `LINEAR_WEBHOOK_SECRET` on `POST /webhooks/linear` and Discord Ed25519 signatures on `POST /api/discord/interactions` before processing payloads.
