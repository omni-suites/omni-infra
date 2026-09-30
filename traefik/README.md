# Central Traefik Edge Proxy

Central reverse proxy for the **omni-suites** infrastructure on the VM. It provides automatic TLS termination (Let's Encrypt), HTTP-to-HTTPS redirection, and routes incoming traffic to internal container backends over the shared `edge` Docker network.

---

## 🌐 Configured Domain Routes

All public hostnames are mapped in [`dynamic/routes.yml`](./dynamic/routes.yml):

| Service | Public Hostname | Backend Container | Port |
| :--- | :--- | :--- | :--- |
| **Frontend UI** | `frontend-svc.test-suites-poc.work.gd` | `omni-client` | `80` |
| **Order Service** | `order-svc.test-suites-poc.work.gd` | `order-service` | `3000` |
| **Inventory Service** | `inventory-svc.test-suites-poc.work.gd` | `inventory-service` | `3001` |
| **Notification Service** | `notification-svc.test-suites-poc.work.gd` | `notification-service` | `3002` |
| **Omni Integration (Webhooks)** | `hooks.test-suites-poc.work.gd` | `omni-integration` | `3003` |
| **Squash TM** | `squash-tcms.test-suites-poc.work.gd` | `squash-tm` | `8080` (path `/squash`) |
| **ReportPortal** | `report-portal.test-suites-poc.work.gd` | `reportportal-gateway` | `8080` |

---

## 🚀 VM Bring-Up Guide

Traefik must be started **first** before launching any other compose stacks so that the external `edge` network is available.

### 1. Configure Environment
```bash
cd /opt/omni-infra/traefik
cp .env.sample .env
```
Edit `.env` and set your Let's Encrypt email:
```bash
ACME_EMAIL=your-email@example.com
```

### 2. Start Traefik
```bash
docker compose up -d
```
*(This automatically creates the external `edge` network and starts listening on ports `80` and `443`)*

### 3. Start Backend Services
Start the remaining stacks in any order. Each service joins the `edge` network:
```bash
# Applications (Order, Inventory, Notification, Omni-client, Omni-integration)
cd /opt/omni-infra/apps && docker compose up -d

# Squash TM
cd /opt/omni-infra/squash-tm && docker compose up -d

# ReportPortal
cd /opt/omni-infra/report-portal && docker compose up -d
```

---

## 🔒 Security & Architecture Details

* **Firewall Requirements:** Only open ports `22` (SSH), `80` (HTTP), and `443` (HTTPS) to the public internet. All database and service ports remain protected inside internal Docker networks.
* **SSL / TLS Certificates:** Uses Let's Encrypt with HTTP-01 challenge over port `80`. Certificates are stored persistently in the `letsencrypt` Docker volume (`/letsencrypt/acme.json`).
* **Dynamic Routing:** Routing rules are loaded from [`dynamic/routes.yml`](./dynamic/routes.yml) with hot-reloading (`--providers.file.watch=true`). You can edit hostnames and routing rules without restarting Traefik.
