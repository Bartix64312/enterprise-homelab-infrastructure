## Infrastructure Visualization & Dashboards

The following captures demonstrate the live operational state of the environment, managed via Portainer and protected by the SSO layer.

### 1. Identity & Access Management (Authelia)
Entry point for all protected internal services. Enforces 2FA and queries the LLDAP backend.

![Authelia Login Portal](./images/autheliaauth.png)

### 2. Edge Routing (Nginx Proxy Manager)
Internal TLS termination and reverse proxy routing. Services are strictly mapped within Docker networks without exposing host ports.

![Nginx Proxy Manager Dashboard](./images/nginx.png)

### 3. Container Orchestration (Portainer)
Live view of the containerized workloads. The infrastructure is logically separated into distinct operational stacks.

![Portainer Container List](./images/portainer.jpg)

## Deployed Stacks & Services

Based on the current operational state, the infrastructure is segmented into the following functional stacks managed via Docker Compose:

* **SSO System (`sso-system`):** * `sso_authelia_frontend` (Access Control)
    * `sso_lldap_backend` (Identity Provider)
    * `sso_redis_cache` (Session Management)
* **Edge & Proxy:** * `nginx-proxy-manager` (Routing)
    * `cloudflared-tunnel` (Secure Ingress)
    * `adguard-home` (DNS sinkhole & internal resolution)
* **Monitoring & Observability (`monitoring-logs`):** * `prometheus` & `node-exporter` (Metrics collection)
    * `promtail` & `loki` (Log aggregation)
    * `grafana` (Visualization)
    * `uptime-kuma` (Service health monitoring)
* **Core Applications:**
    * `nextcloud-aio` (Self-hosted productivity suite, deployed via All-in-One master container)
    * `homeassistant` (IoT/Smart Home control plane)
* **Automated Media Pipeline (`arr-stack`):**
    * `prowlarr`, `radarr`, `sonarr` (Content management automation)
    * `transmission` (Data retrieval)
    * `flaresolverr` (Proxy clearance)
    * `jellyfin` (Media streaming output)