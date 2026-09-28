# Central Traefik edge for omni-suites
#
# DNS (A records → this VM) — defined in dynamic/routes.yml:
#   frontend-svc / order-svc / inventory-svc / notification-svc
#   report-portal / squash-tcms
#   *.test-suites-poc.work.gd
#
# Bring-up order on the VM:
#   1. cd infra/traefik && cp .env.sample .env  # set ACME_EMAIL
#      docker compose up -d
#   2. apps / squash-tm / report-portal (join external network `edge`,
#      use container_name matching routes.yml backends)
#   3. Firewall: 22, 80, 443 only
#
# Routing: file provider → ./dynamic/routes.yml (edit hostnames there).
# Backends: http://<container_name>:<port> on network `edge`.
# Certs: Let's Encrypt HTTP-01 on entrypoint `web` (port 80 must stay open).
# Volume: `letsencrypt` for acme.json.
