# Project Guidelines

## Code Style
- This repo is a Helm chart made of explicit Kubernetes YAML manifests; keep templates simple and declarative.
- Follow existing 2-space YAML indentation and key ordering patterns from `helm/templates/*.yaml`.
- Use `.Values` only where already parameterized (mostly image tags and runtime settings), following `helm/values.yaml`.
- Keep workload labels/service selectors aligned (`app: <name>`), as in `helm/templates/deployment-homeassistant.yaml` and `helm/templates/deployment-mosquitto.yaml`.
- Preserve scheduling conventions like `floor` node affinity and `TZ=Europe/Zurich` where already used.

## Architecture
- Single chart in `helm/` deploys home automation stack components (Home Assistant, ESPHome, Zigbee2MQTT, Mosquitto, Pi-hole, UniFi, ChangeDetection, ingress, cert issuer, DuckDNS cronjobs).
- Routing and TLS are centralized in `helm/templates/ingress.yaml` with cert-manager integration via `helm/templates/clusterissuer-letsencrypt.yaml`.
- External DNS refresh is handled by `helm/templates/cron-duckdns.yaml`.
- Runtime is node-coupled in multiple services via `hostPath`, USB/device mounts, and fixed `NodePort` services.

## Build and Test
- Install cert-manager (from `readme.md`):
  - `helm repo add jetstack https://charts.jetstack.io`
  - `helm repo update`
  - `helm upgrade cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --install --version v1.13.3 --set installCRDs=true`
- Validate chart locally before deploy:
  - `helm lint helm/`
  - `helm template home helm/`
- Deploy chart (local/CI pattern from `readme.md` and `.github/workflows/build_and_deploy.yml`):
  - `helm upgrade home helm/ --install --namespace default --create-namespace --set letsencrypt.email=... --set duckdnstoken=... --set hostnamepantry=... --set hostnamepantryapi=... --set hostnamehomeassistant=... --set esphome.password=...`
- No automated unit/integration test suite is defined in this repository; do not invent test commands.

## Project Conventions
- Keep `helm/values.yaml` focused on image tags; avoid storing operational secrets or hostnames there.
- Pass environment-specific values via `--set` (or equivalent secure CI inputs), matching `.github/workflows/build_and_deploy.yml`.
- Do not refactor resource kinds/names casually; service names are wired into ingress backends (see `helm/templates/ingress.yaml`).
- Keep fixed ports/nodePorts stable unless explicitly requested; downstream home-network config depends on them.

## Integration Points
- Kubernetes integrations: `ingress-nginx` class `nginx`, cert-manager CRDs/issuer, and cluster services.
- External integrations: Let’s Encrypt ACME endpoint, DuckDNS update API, and container images from Docker Hub/GHCR.
- CI integration: GitHub Actions self-hosted runner deploy job in `.github/workflows/build_and_deploy.yml`.

## Security
- Treat all `--set` values for tokens/passwords/hostnames as sensitive; never hardcode or commit real secrets.
- Prefer Kubernetes Secret references for new sensitive configuration instead of plain env/args.
- Be cautious when editing privileged workloads (`privileged: true`, `hostNetwork`, `hostPath`, device mounts), e.g. Zigbee2MQTT/Home Assistant.
- Flag risky defaults when touched (example: `WEBPASSWORD: "password"` in `helm/templates/deployment-pihole.yaml`).