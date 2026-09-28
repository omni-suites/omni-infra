# Deploy apps stack (behind Traefik)
#
# Public hostnames: ../traefik/dynamic/routes.yml
# This compose only joins `edge` + stable container_name for Traefik backends.
#
# Split roots — infra stays under omni-infra; app code under omni-app:
#
#   /opt/omni-infra/          ← infra only
#     apps/                   ← compose + .env (this folder)
#     traefik/
#     squash-tm/
#     report-portal/
#
#   /opt/omni-app/            ← services only
#     order-service/
#     inventory-service/
#     notification-service/
#     omni-client/
#
# APP_ROOT defaults to ../../omni-app (from apps/ → /opt/omni-app).
# Local monorepo only: set APP_ROOT=../../app in .env
#
# Start (after Traefik):
#   cd /opt/omni-infra/apps && docker compose up -d --build
#
# Note: *.work.gd may hit Let's Encrypt rate limits (DEFAULT CERT).
