# Enterprise-Grade Home Infrastructure & Security Lab

## 📖 Project Overview
This repository documents the architecture, deployment, and configuration of a resilient, automated, and security-hardened self-hosted infrastructure platform. The project integrates hardware virtualization with container orchestration to create a robust environment for self-hosted private services.

The primary objective was to architect a secure, isolated environment supporting automated data processing pipelines and to implement Enterprise-class solutions for Identity and Access Management (IAM) and network security. This environment serves as a continuous practical testing ground for cybersecurity concepts, network micro-segmentation, and incident response simulations.

## 🛠️ Tech Stack / Technologies Used

**Infrastructure & Orchestration:**
<br>
<img src="https://img.shields.io/badge/Proxmox_VE-E57000?style=for-the-badge&logo=proxmox&logoColor=white" alt="Proxmox" />
<img src="https://img.shields.io/badge/Ubuntu_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu" />
<img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
<img src="https://img.shields.io/badge/Portainer-13BEF9?style=for-the-badge&logo=portainer&logoColor=white" alt="Portainer" />

**Security, Proxy & IAM:**
<br>
<img src="https://img.shields.io/badge/Cloudflare_Zero_Trust-F38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Cloudflare" />
<img src="https://img.shields.io/badge/Nginx_Proxy_Manager-009639?style=for-the-badge&logo=nginx&logoColor=white" alt="NPM" />
<img src="https://img.shields.io/badge/Authelia-43853D?style=for-the-badge&logo=shield&logoColor=white" alt="Authelia" />
<img src="https://img.shields.io/badge/Tailscale-FFFFFF?style=for-the-badge&logo=tailscale&logoColor=black" alt="Tailscale" />

**Core Applications & Pipelines:**
<br>
<img src="https://img.shields.io/badge/Nextcloud-0082C9?style=for-the-badge&logo=nextcloud&logoColor=white" alt="Nextcloud" />
<img src="https://img.shields.io/badge/Jellyfin-00A4DC?style=for-the-badge&logo=jellyfin&logoColor=white" alt="Jellyfin" />
<img src="https://img.shields.io/badge/Automated_Data_Pipeline-4A4A55?style=for-the-badge&logo=databricks&logoColor=white" alt="Pipeline" />

## 🏗️ Architecture & Topology

The core architecture leverages Virtual Machines (VMs) managed by the **Proxmox VE** hypervisor. To maintain a clear separation between the virtualization layer and the containerization layer, the **Docker Engine** is deployed within a dedicated, hardened Ubuntu LTS VM provisioned on Proxmox.

Deployment and configuration are managed declaratively using `docker compose` files stored in this repository **(Infrastructure as Code)**. **Portainer** is utilized as an observability and operational convenience layer, providing a real-time GUI for log streaming and resource monitoring without overriding the Git-backed compose state.

Network micro-segmentation is enforced at the Docker bridge level using distinct, non-routable networks (`proxy` for frontend traffic, `monitoring` for observability telemetry). To prevent Docker's default behavior of bypassing host-level firewall rules (UFW) via direct `iptables` chain manipulation, critical internal services are strictly bound to loopback interfaces (`127.0.0.1`) or confined entirely within non-routable container networks — exposing only the designated Cloudflare Tunnel entrypoint to the outside world.

The architecture separates **control plane components** (Proxmox hypervisor, IAM stack, orchestration tooling) from **data plane services** (applications and automation pipelines), minimizing blast radius in the event of a service-level compromise.

**Traffic Flow (Logical Topology):**
```text
[External User]
      │ HTTPS
      ▼
[Cloudflare Edge – WAF & DDoS Mitigation]
      │ Cloudflare Tunnel (outbound-initiated, no open inbound ports)
      ▼
[Nginx Proxy Manager – TLS Termination & Routing]
      │ auth_request directive
      ▼
[Authelia – 2FA/SSO Verification] ←→ [LLDAP – Identity Store]
      │ (pass, if authenticated)
      ▼
[Docker Internal Bridge Network]
      ├──> Nextcloud
      ├──> self-hosted media streaming service (Jellyfin)
      └──> Automated Data Processing Pipeline
```

## 🔐 Security, Hardening & Access Control

Security is embedded into the core of this infrastructure, adhering to a strict **defense-in-depth approach with network micro-segmentation**:

*   **Hypervisor Isolation:** Access to the Proxmox VE management interface is restricted exclusively via Tailscale VPN. SSH access to the host is further hardened with password authentication disabled, key-based authentication enforced, and `fail2ban` monitoring the SSH service — all bound to the Tailscale network interface only.
*   **Reverse Proxy & TLS Termination:** All inbound traffic is routed through Cloudflare Tunnels to Nginx Proxy Manager. The Cloudflare Tunnel is treated as an external access facilitator, not a security boundary — additional controls including service-level authentication and internal network isolation are enforced independently to avoid single-provider trust dependency. The Nginx Proxy Manager administrative interface is bound exclusively to the host's loopback interface, requiring an explicit SSH tunnel (`ssh -L 81:localhost:81 user@vm`) for remote administration. Nginx is additionally hardened with strict HTTP security headers (HSTS, X-Frame-Options, X-Content-Type-Options) and request rate limiting to reduce attack surface at the ingress layer.
*   **Identity and Access Management (IAM):** Implemented **Authelia** as an Identity Provider paired with **LLDAP** (chosen for its lightweight memory footprint compared to Keycloak, optimized for low-resource environments). Access requires Mandatory 2FA (TOTP). Nginx utilizes the `auth_request` directive to reject unauthenticated HTTP requests before they reach application containers. Authelia is configured with a `strict` same-site cookie policy (fully blocking cross-site request forgery vectors, suitable for a single-domain `*.yourdomain.com` topology) and a session inactivity timeout, enforcing re-authentication after periods of inactivity.
*   **OS-Level Hardening:** The underlying Ubuntu VM is hardened using `ufw` configured with default-deny policies, `fail2ban` for SSH brute-force mitigation, and `unattended-upgrades` for automated security patching.
*   **Service-to-Service Trust:** Inter-container communication is currently trusted within dedicated internal Docker networks. Enforcing mTLS and service identity validation to implement Zero Trust principles at the service mesh level is a planned improvement.

## ⚙️ Deployment & Automation (Data Pipeline)

The infrastructure features a fully automated data ingestion and media management pipeline, demonstrating proficiency in integrating API-driven microservices. The pipeline orchestrates content discovery, automated quality-profile matching, controlled asset retrieval, post-processing (normalization, cataloguing), and seamless integration with the media server for on-demand streaming.

All services are deployed using the **Infrastructure as Code (IaC)** methodology. The `/compose-files` directory in this repository contains the `docker-compose.yml` blueprints utilized for deployment. While secrets are temporarily managed via environment variables for initial lab bootstrapping, the architecture is specifically decoupled to allow seamless migration to an externalized key management system. The next iteration targets **SOPS (Mozilla SOPS) with Age encryption** for storing encrypted secrets directly in the repository, providing a practical intermediate step before a full HashiCorp Vault deployment.

## 🔄 Backup & Disaster Recovery (DR)

To ensure data integrity and service recoverability, a multi-tiered backup strategy is implemented:
1.  **Proxmox Level:** Automated, scheduled snapshots of the entire Ubuntu VM to an external, physically separated drive.
2.  **Offsite Replication (3-2-1 Rule):** Critical application data (Nextcloud, configuration files) is replicated to an offsite location via encrypted rclone sync, adhering to the 3-2-1 backup principle (3 copies, 2 media types, 1 offsite).
3.  **Pre-Deployment Safeguards:** Point-in-time VM snapshots are executed prior to major state mutations or core stack upgrades, effectively providing an instant rollback mechanism to minimize Mean Time To Recovery (MTTR) during deployment regressions.
4.  **DR Testing:** Backup integrity is periodically validated through restoration drills, confirming recoverability and verifying that the documented Recovery Time Objective (RTO) is achievable under real failure conditions.

## 🎯 Threat Model

The security architecture was designed against the following threat scenarios:

| Threat Actor | Vector | Mitigation |
| :--- | :--- | :--- |
| External attacker | Public-facing service enumeration | Cloudflare WAF, no open inbound ports; origin services accessible only via authenticated reverse proxy layer |
| Credential compromise | Brute force / stuffing | `fail2ban`, TOTP mandatory, session timeouts |
| Lateral movement | Compromised container escaping to adjacent services | Docker network segmentation combined with host-level firewall policies (not treated as a strong security boundary); loopback-bound admin panels |
| Insider / misconfiguration | Accidental secret exposure | `.gitignore`, no hardcoded credentials in IaC files |
| Supply chain | Vulnerable base image | Trivy scanning (planned), minimal base images, and regular image updates |

## 🛡️ Security Use Cases (Hands-On)

This environment is actively used to simulate and analyze real-world security scenarios:

*   Reverse proxy misconfiguration leading to authentication bypass attempts
*   Session handling and cookie policy validation (SameSite, secure flags)
*   Brute-force and credential stuffing detection (`fail2ban` + IAM layer correlation)
*   Container permission misconfigurations and volume exposure risks
*   Service exposure validation — ensuring no unintended public endpoints survive a configuration change

## ⚠️ Challenges & Lessons Learned

Building this environment provided extensive hands-on experience in troubleshooting complex network and application layer issues:

*   **Troubleshooting Docker DNS Resolution & Reverse Proxy:** Resolved persistent HTTP 500/502 Gateway error loops between Nginx Proxy Manager and Authelia. The solution required enforcing strict internal Docker networking, overriding the default Docker DNS resolver (`127.0.0.11` resolution inconsistencies under specific network configurations) with manual variable assignments, and explicitly setting `X-Forwarded-Proto https` headers to satisfy Authelia's strict TLS security requirements behind a Cloudflare Tunnel. **Key takeaway:** Docker inter-container DNS resolution should always be validated explicitly during initial stack deployment, particularly in multi-proxy architectures.
*   **Permissions & Container Volumes:** Debugged path mapping discrepancies where internal automated services failed to locate ingested data. This required aligning PUID/PGID environment variables and creating unified, absolute path mappings across multiple isolated containers to prevent permission-denied errors. **Key takeaway:** Standardizing container execution contexts using uniform PUID/PGID and centralized volume management is critical for scalable, permission-agnostic data pipelines.

## 🔍 Live Verification (Read-Only)

To demonstrate the IAM and reverse proxy security layers in a real-world scenario, a read-only demo endpoint is available upon request for technical reviewers. The demo environment runs in a dedicated Docker network namespace fully isolated from production data (Nextcloud, personal files) — no production credentials or data are accessible through the demo stack.

| Endpoint | Description | Security Context |
| :--- | :--- | :--- |
| `https://demo-iam.YOURDOMAIN.com` | Authelia Login Portal | Enforced 2FA, routed via Cloudflare |
| `https://demo-app.YOURDOMAIN.com` | Protected Test Application | Returns `401` unless valid Authelia session exists |

*Demo credentials (read-only user + TOTP seed) available via private message to verified hiring managers.*


## 📊 Observability & Telemetry

A dedicated monitoring stack (`compose-files/monitoring-stack.yml`) provides full visibility into the infrastructure's operational health. **Prometheus** scrapes host and container metrics via **Node Exporter**, while **Loki** aggregates structured logs shipped by **Promtail**. **Grafana** dashboards visualize resource utilization, authentication latency (Authelia metrics), and filesystem usage, enabling proactive anomaly detection and capacity planning.

The monitoring stack also serves as a security observability layer: centralized, structured logs enable SIEM-like correlation of authentication events, failed access attempts, and anomalous resource consumption patterns — supporting incident investigation workflows.

## 🚀 Future Improvements

The infrastructure is an actively evolving environment. Near-term objectives focused on enhancing security incident detection and operational maturity include:

*   **Host Intrusion Detection System (HIDS):** Deployment of Wazuh agents to monitor security events, perform anomaly detection, and implement File Integrity Monitoring (FIM) across the infrastructure.
*   **Alerting & Incident Response:** Integration of **Alertmanager** with Prometheus to deliver real-time critical alerts (disk failure, authentication anomalies, container crashes) via a webhook to a private Discord/Telegram channel — ensuring incidents are surfaced proactively rather than discovered during manual dashboard reviews.
*   **CI/CD Security Gates:** Implementation of a GitHub Actions pipeline to enforce automated security gates on every commit: YAML linting (`yamllint`), container image vulnerability scanning (Trivy), and secret detection — ensuring no insecure configuration reaches the deployment branch.
*   **Secrets Management:** Migration from `.env` file-based secrets to **SOPS + Age** for encrypted-at-rest secrets stored in the repository, as an intermediate step toward a full HashiCorp Vault deployment.
