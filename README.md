
# Copi-Hosting Panel 
A multi-tenant hosting system that is lean and consistently functional on Rocky Linux. No bloated control interface; only a small UI, clean automation, and containers. Swappable components, opinionated defaults.


## The Goals Strived To Be Accomplished
- **Multi-tenant**: strict segregation, quotas for each tenant, and a basic lifecycle (provision → scale → suspend → archive → delete).
- **Stacks that matter**: WordPress/PHP-FPM, Node.js, Python (ASGI/Wsgi), CFML (Lucee/Adobe), static sites, and databases (PostgreSQL/MariaDB/SQLite). Redis is intentionally excluded to avoid enterprise licensing and security pitfalls.
- **Security first**: SELinux enforcing, firewalld, least privilege, read-only root FS where possible.
- **Batteries included**: automatic HTTPS, backups (S3-compatible), metrics/logs, alerts.
- **DNS externalized**: tenants manage DNS records (A, AAAA, CNAME, SPF, DKIM, DMARC) at providers like GoDaddy, DreamHost, or Cloudflare. The panel is built to manage TLS and routing at the application level; in the future, we will include API connectors to registers.
- **API + CLI**: All data in a coding format. Base API with minimal web UI.
- **Minimalist UI/UX**: **Feather Icons**, [Heroicons](https://heroicons.com/), or [Lucide](https://lucide.dev/) are just a few examples of free and open-source icon sets that can help with a minimalist, contemporary design that aims to avoid bloat.


## High-Level Architecture
```
┌──────────────────────────────────────────────────────────────────┐
│ Rocky Linux Host │
│ SELinux (enforcing) · firewalld · systemd · cgroups v2 │
│ │
│ ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐ │
│ │ Control API │ │ Worker │ │ Observability │ │
│ │ (FastAPI) │ ←→ │ Executor │ ←→ │ (Prom/Graf/Loki) │ │
│ └──────────────┘ │ (Ansible + │ └────────────────────┘ │
│ ↑ │ Podman) │ ↑ │
│ Web UI │ └──────────────┘ │ │
│ (Next.js)│ ↑ │ │
│ ↓ │ │ │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │ Reverse Proxy + TLS (Caddy) │ │
│ └──────────────────────────────────────────────────────────────┘ │
│ ↑ ↑ ↑ ↑ │
│ Tenant A Tenant B Tenant C Tenant N │
│ (Podman pods: (pods: app+db) (static + sqlite) (… ) │
│ app+db+cache) │ │
└──────────────────────────────────────────────────────────────────┘
```


**Why these choices?**
- **Podman** (rootless where possible) keeps us close to RHEL/Rocky best practices and SELinux friendliness.
- **Caddy** for dead-simple HTTPS (Let’s Encrypt) and on-the-fly vhost config via API.
- **FastAPI** unchanged endpoints are exposed via the control plane, while the heavy lifting is done by **Ansible**.
- **Prometheus/Grafana/Loki** fill in dashboards, logs, and metrics. Measurement advisor for containers.
- **restic** for quick, deduped backups to S3 (Contabo, MinIO, Backblaze, etc.).
- **SQLite** available for smaller tenants or applications that do not necessitate a complete RDBMS.

## Isolation & Resource Model
- **Namespace isolation** via Podman pods per tenant: `net`, `ipc`, `pid`, `uts`.
- **cgroups v2 quotas** separate systemd slice for each tenant (CPU, memory, pids, IO): 'tenant-<id>.slice'.
- **Network isolation**: Only the reverse proxy provides a north-south path in the per-tenant Podman network.
- **File system**: bind-mount read-only images; write paths mapped to volumes under `/srv/tenants/<id>/volumes/`.


## Tenant Manifest (source of truth)
```yaml
# /srv/copilot/tenants/acme-001/manifest.yml
tenant_id: acme-001
primary_domain: app.acme.tld
aliases: [www.acme.tld]
stack: wordpress # wordpress|php|node|python|cfml|static|custom
size: small # small|medium|large (maps to cgroup quotas)
components:
-- db: mariadb # mariadb|postgres|sqlite|none
-- engine: lucee # lucee|adobe (for cfml)
-- runtime: native # native|container
env:
-- WP_ENV: production
-- DB_NAME: acme
-- DB_USER: acme
-- DB_PASS: ${SECRET_DB_PASS}
-- DB_HOST: db
-- storage:
-- app: 10Gi
-- db: 20Gi
backups:
-- s3:
-- endpoint: https://s3.contabo.com
-- bucket: copilot-backups
-- prefix: acme-001
-- access_key: ${S3_KEY}
-- secret_key: ${S3_SECRET}
```

## Web UI / UX
- **Next.js + Tailwind** for front-end simplicity.
- Icons: Feather, Heroicons, or Lucide (all free & open source).
- Pages: Tenants list, Tenant detail, Manifest editor, Events log.
- Focus: **minimal clicks, fast load, clean typography, responsive layout**.

## DNS Model
- Tenants provide their domain.
- TLS handled by **Caddy** (Let’s Encrypt HTTP-01 or DNS-01 if API keys supplied).
- Tenants must configure A/CNAME/AAAA/SPF/DKIM/DMARC themselves at their registrar (GoDaddy, DreamHost, Cloudflare, etc.). We don’t run authoritative DNS servers, reducing scope and complexity.


## Reverse Proxy Flow (Public vs Local)
```
[ Client Browser ]
|
DNS: app.example.com
|
resolves to Public IP
v
┌──────────────────────────┐
│ Rocky Linux Server │
│ Public iface: 203.0.113.42
│ │
│ ┌───────────────┐ │
│ │ Caddy │◄─────┘
│ │ :80 / :443 │
│ │ ACME TLS │
│ │ vhost router │
│ └─────▲─────────┘
│ │ reverse_proxy 127.0.0.1:18081
│ ┌─────┴─────────┐
│ │ Tenant nginx │ (Podman container, localhost only)
│ │ binds → 18081 │
│ └─────▲─────────┘
│ │ proxy_pass
│ ┌─────┴─────────┐
│ │ App Runtime │ (WordPress PHP-FPM, Lucee CFML, Node, etc.)
│ │ (private) │
│ └───────────────┘
│
└──────────────────────────┘
```
