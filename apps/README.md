# Deploy apps stack (behind Traefik)
#
# Public hostnames: ../traefik/dynamic/routes.yml
# App images: ${GCP_AR_REPO}/<service>:staging (CI push from each app repo)
#
#   /opt/omni-infra/apps/     ← this compose + .env (set GCP_AR_REPO)
#   Traefik must be up first (network `edge`)
#
# Start:
#   cd /opt/omni-infra/apps && docker compose pull && docker compose up -d
#
# Per-service deploy: each app's .github/workflows/deploy-staging.yaml
# pulls/restarts only that service on the self-hosted staging runner.
