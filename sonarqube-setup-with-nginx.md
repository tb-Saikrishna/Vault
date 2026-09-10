# SonarQube Server on Ubuntu — Complete Implementation and Operations Guide

> **Purpose:** This document records the complete implementation used to deploy SonarQube Community Build on the existing Jenkins VM, isolate it from the host PostgreSQL, expose it through Nginx over HTTPS on an internal hostname, and prepare it for Jenkins CI integration. It also documents the cloud migration path so that this on-prem deployment can be lifted to AWS/GCP/Azure with minimal changes.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Original State](#2-original-state)
3. [Requirements and Constraints](#3-requirements-and-constraints)
4. [Design Decisions and Rationale](#4-design-decisions-and-rationale)
5. [Target Architecture](#5-target-architecture)
6. [Installation Phases Overview](#6-installation-phases-overview)
7. [Phase 0 — Preflight Assessment](#7-phase-0--preflight-assessment)
8. [Phase 1 — Disk Safety and Reclaim](#8-phase-1--disk-safety-and-reclaim)
9. [Phase 2 — Directory Layout and Kernel Limits](#9-phase-2--directory-layout-and-kernel-limits)
10. [Phase 3 — Docker Compose Stack](#10-phase-3--docker-compose-stack)
11. [Phase 3A — Offline/Online Image Selection](#11-phase-3a--offlineonline-image-selection)
12. [Phase 3B — Secrets Management](#12-phase-3b--secrets-management)
13. [Phase 3C — First Boot and Health Verification](#13-phase-3c--first-boot-and-health-verification)
14. [Phase 3D — Admin Hardening](#14-phase-3d--admin-hardening)
15. [Phase 3E — Nginx Reverse Proxy and TLS](#15-phase-3e--nginx-reverse-proxy-and-tls)
16. [Phase 3F — DNS Resolution](#16-phase-3f--dns-resolution)
17. [Verification Checklist](#17-verification-checklist)
18. [Operations Runbook](#18-operations-runbook)
19. [Logging and Rotation](#19-logging-and-rotation)
20. [Backup Strategy](#20-backup-strategy)
21. [Restore Procedure](#21-restore-procedure)
22. [Upgrade Procedure](#22-upgrade-procedure)
23. [Rollback Procedure](#23-rollback-procedure)
24. [Troubleshooting Guide](#24-troubleshooting-guide)
25. [Jenkins Integration (Planned)](#25-jenkins-integration-planned)
26. [Cloud Migration Path](#26-cloud-migration-path)
27. [Security Considerations](#27-security-considerations)
28. [File Inventory](#28-file-inventory)
29. [Command Cheat Sheet](#29-command-cheat-sheet)
30. [Mental Model](#30-mental-model)

---

## 1. Executive Summary

A SonarQube Community Build server was deployed on the existing Jenkins VM (`Jenkins-Master`, `192.168.0.17`) using Docker Compose with two containers — SonarQube and PostgreSQL — without disturbing the pre-existing Jenkins service, host PostgreSQL, or Nginx configuration.

**Final architecture:**

```
                    Internal Network
                           |
                           |  HTTPS 443
                           v
              sonarqube.tech-bridge.biz
                           |
                           v
                 +---------------------+
                 |        Nginx        |
                 |  TLS Termination    |
                 |  Host-based routing |
                 +---------------------+
                     |             |
                     |             |  HTTPS 443
                     |             v
                     |     jenkins.tech-bridge.biz
                     |             |
                     |             v
                     |          Jenkins
                     |       127.0.0.1:8080
                     |
                     |  HTTP localhost
                     v
              127.0.0.1:9000
                     |
                     v
              +----------------+
              |   SonarQube    |
              |   container    |
              +----------------+
                     |
                     |  Docker bridge "sonarnet"
                     |  (not exposed to host)
                     v
              +----------------+
              |   PostgreSQL   |
              |   container    |
              |  (no host port)|
              +----------------+
```

**Key security and design properties:**

- SonarQube and PostgreSQL run as Docker containers with pinned image tags.
- PostgreSQL is **never** published to the host or LAN — reachable only on the internal Docker bridge.
- SonarQube is bound to `127.0.0.1:9000` only. Nginx is the sole ingress.
- HTTPS is terminated at Nginx using the existing wildcard certificate `*.tech-bridge.biz`.
- HTTP requests redirect to HTTPS with a 301.
- The pre-existing Jenkins vhost is untouched. SonarQube is a separate Nginx `server` block.
- The host PostgreSQL 16 serving the `cybersio` database is untouched. It runs on `0.0.0.0:5432` and does not conflict with the containerized PostgreSQL.
- Secrets live in `/opt/sonarqube/.env`, mode `0600`, owned `root:root`.
- Kernel limits (`vm.max_map_count`, `fs.file-max`) are persisted via `/etc/sysctl.d/99-sonarqube.conf`.
- Container `ulimits` (`nofile`, `nproc`) match SonarQube’s requirements.
- Anonymous access is disabled and the default `admin` password is changed.
- Container logs are capped (10 MB × 5 files).
- Backups are designed to be shipped off-host by rsync.

---

## 2. Original State

| Property | Value |
|---|---|
| OS | Ubuntu 24.04.5 LTS (Noble) |
| Hostname | `Jenkins-Master` |
| IP | `192.168.0.17` |
| Docker Engine | 29.5.3 |
| Docker Compose | v5.1.4 |
| Disk `/` | 97 GB total, 24 GB used, 69 GB free after reclaim |
| RAM | 19 GiB (16 GiB free, 4 GiB swap unused) |
| vCPU | 20 |
| Jenkins | `jenkins.war` systemd service, `127.0.0.1:8080` |
| Nginx | Installed, terminates TLS on 443, vhost for `jenkins.tech-bridge.biz` |
| Host PostgreSQL | PostgreSQL 16 on `0.0.0.0:5432`, serving `cybersio` database |
| Wildcard cert | `/etc/nginx/ssl/fullchain.pem`, `/etc/nginx/ssl/server.key`, SAN `*.tech-bridge.biz` |
| Internal DNS | `192.168.0.146` |

---

## 3. Requirements and Constraints

From the original brief and subsequent conversation:

1. Use Docker Compose for SonarQube and PostgreSQL.
2. Do not use SonarQube’s embedded database.
3. Use a current supported SonarQube Community Edition / Community Build image, pinned, no `latest`.
4. Use a supported PostgreSQL version compatible with the selected SonarQube version, pinned.
5. Do not publish PostgreSQL to the network.
6. Expose SonarQube via Nginx on an internal hostname, HTTPS, without disrupting Jenkins.
7. Configure kernel and process limits required by SonarQube/Elasticsearch, persisted across reboot.
8. Use non-default, strong database credentials stored safely.
9. Restrict firewall to internal access only.
10. Verify disk safety before install.
11. Configure log rotation and a backup plan off-host.
12. Provide health-check, diagnostic, upgrade, and rollback commands.
13. Ubuntu/Debian-first; ask before doing anything OS-specific.
14. **Do not** use `curl | bash`, `latest` tags, default passwords, unsecured public exposure.
15. Cloud-portable design so on-prem can migrate later.

---

## 4. Design Decisions and Rationale

| Decision | Rationale |
|---|---|
| Docker Compose over native install | Isolates SonarQube from Jenkins and host PostgreSQL; simplified rollback; pinned image tags; cloud-portable. |
| Bind mounts under `/opt/sonarqube` over Docker named volumes | Easier to inspect, back up, rsync, and later move to a dedicated disk or cloud volume. |
| PostgreSQL never published to the host | Eliminates a network attack surface; only the SonarQube container reaches it over an internal bridge. |
| SonarQube bound to `127.0.0.1:9000` | Nginx is the only ingress. Direct network access to SonarQube is impossible. |
| Reuse the existing wildcard certificate | Avoids reissuance, keeps TLS identity consistent with Jenkins, and matches the existing operational model. |
| Separate Nginx vhost file | Guarantees Jenkins routing is not affected by SonarQube configuration. |
| Stricter TLS on the SonarQube vhost | SonarQube rejects TLS 1.0/1.1 without affecting Jenkins’ global policy. |
| `.env` file, mode 0600 | Standard for the official SonarQube Docker image; balances simplicity and security on a single-host deployment. |
| Kernel limits via `/etc/sysctl.d/99-sonarqube.conf` | SonarQube/Elasticsearch require `vm.max_map_count` ≥ 262144. Persist across reboots. |
| Container `ulimits` for `nofile` and `nproc` | Per-container limits do not affect Jenkins. |
| `SONAR_CORE_SERVERBASEURL` env | Ensures webhook URLs and generated links use the external hostname, not `localhost:9000`. |
| Container log rotation via `logging:` | Prevents unbounded JSON log growth on the host disk. |

---

## 5. Target Architecture

### 5.1 Logical layers

```
+---------------------------------------------------------------+
|                    Internal Users / Jenkins                   |
+---------------------------------------------------------------+
                              |
                              | HTTPS 443
                              v
+---------------------------------------------------------------+
|  Nginx (host)                                                 |
|  - TLS termination                                            |
|  - HTTP -> HTTPS redirect                                     |
|  - Host-based routing                                         |
|    server_name jenkins.tech-bridge.biz -> 127.0.0.1:8080      |
|    server_name sonarqube.tech-bridge.biz -> 127.0.0.1:9000    |
+---------------------------------------------------------------+
                              |
                +-------------+-------------+
                |                           |
                v                           v
        +---------------+           +-----------------+
        |   Jenkins     |           |   SonarQube     |
        |  127.0.0.1:8080           |  127.0.0.1:9000 |
        +---------------+           +-----------------+
                                            |
                                            | Docker bridge "sonarnet"
                                            | (internal only)
                                            v
                                    +-----------------+
                                    |   PostgreSQL    |
                                    |  db:5432        |
                                    | (no host port)  |
                                    +-----------------+
```

### 5.2 Storage layout

```
/opt/sonarqube/
├── .env                     (0600 root:root)          secrets
├── .gitignore               (0644 root:root)          .env excluded
├── docker-compose.yaml      (0640 root:root)          stack definition
├── data/                    (0750 1000:1000)          SonarQube data + ES indexes
├── extensions/              (0750 1000:1000)          plugins
├── logs/                    (0750 1000:1000)          SonarQube logs
├── postgres_data/           (0750 999:999)            PostgreSQL data directory
└── postgres_init/           (0750 root:root)          reserved (unused)
```

### 5.3 Network model

| Layer | Address | Notes |
|---|---|---|
| Nginx HTTPS listener | `0.0.0.0:443` | Internal network only |
| Nginx HTTP listener | `0.0.0.0:80` | Redirect to HTTPS |
| SonarQube published port | `127.0.0.1:9000` | Loopback only |
| PostgreSQL published port | none | Internal bridge only |
| Jenkins listener | `127.0.0.1:8080` | Loopback only |
| Host PostgreSQL | `0.0.0.0:5432` | Pre-existing, serves `cybersio` |

---

## 6. Installation Phases Overview

| Phase | Purpose |
|---|---|
| 0 | Preflight assessment: OS, disk, Docker, Nginx, ports |
| 1 | Disk safety verification and reclaim |
| 2 | Directory layout, kernel limits, sysctl persistence |
| 3 | Docker Compose stack, secrets, image pull, first boot |
| 3A | Image selection and pinning |
| 3B | Secrets management |
| 3C | Boot and health verification |
| 3D | Admin hardening (password change, force auth) |
| 3E | Nginx vhost and TLS |
| 3F | DNS resolution |
| 4 | Jenkins integration (planned) |

---

## 7. Phase 0 — Preflight Assessment

Commands run (read-only):

```bash
cat /etc/os-release
df -hT
sudo du -xh /var/lib/docker 2>/dev/null | sort -h | tail -30
docker --version
docker compose version
sudo nginx -T 2>/dev/null | sed -n '1,220p'
```

Findings:

- OS: Ubuntu 24.04.5 LTS.
- Docker 29.5.3, Compose v5.1.4 present.
- Nginx serving `jenkins.tech-bridge.biz` on 443 with `/etc/nginx/ssl/fullchain.pem` and `/etc/nginx/ssl/server.key`, proxying to `127.0.0.1:8080`.
- Global `nginx.conf` allows TLS 1.0–1.3. SonarQube vhost overrides to TLS 1.2–1.3.
- 40 GB free initially — insufficient headroom. Led to Phase 1.

---

## 8. Phase 1 — Disk Safety and Reclaim

Read-only inspection:

```bash
sudo docker ps -a --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Status}}'
sudo docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}'
sudo docker volume ls
df -hT /
```

Findings:

- 10 Docker images, 7 stopped containers (10.49 GB), 22 build cache entries (1.638 GB), 50 named volumes.
- Named volumes preserved; two belonged to a legacy SIEM stack and were left untouched.

Reclaim:

```bash
sudo docker container prune
sudo docker image prune
sudo docker builder prune
# Or, more aggressively, once confirmed safe:
sudo docker system prune -f
```

Result: **12.13 GB reclaimed.** Free space on `/` rose from 40 GB to **69 GB**. Well above the 45 GB safety threshold for SonarQube + PostgreSQL + Elasticsearch headroom.

Post-reclaim state:

```bash
df -hT /              # 97G total, 24G used, 69G free
sudo docker system df # images/containers/build cache cleared
sudo docker volume ls # 50 named volumes preserved
```

---

## 9. Phase 2 — Directory Layout and Kernel Limits

### 9.1 Kernel limits

The kernel already reported:

```
vm.max_map_count = 1048576
fs.file-max = 9223372036854775807
```

SonarQube/Elasticsearch require `vm.max_map_count` ≥ 262144. The current value is well above. The task was to **persist** it.

Create `/etc/sysctl.d/99-sonarqube.conf`:

```
# SonarQube / Elasticsearch requirements
vm.max_map_count = 1048576
fs.file-max = 9223372036854775807
```

Apply:

```bash
sudo sysctl --system
sudo sysctl vm.max_map_count fs.file-max
```

### 9.2 Directory layout

```bash
sudo install -d -o root -g root -m 0750 /opt/sonarqube

sudo install -d -o 1000 -g 1000 -m 0750 /opt/sonarqube/data
sudo install -d -o 1000 -g 1000 -m 0750 /opt/sonarqube/extensions
sudo install -d -o 1000 -g 1000 -m 0750 /opt/sonarqube/logs

sudo install -d -o 999 -g 999 -m 0750 /opt/sonarqube/postgres_data
sudo install -d -o root -g root -m 0750 /opt/sonarqube/postgres_init
```

UID `1000` matches the `sonarqube` user inside the official image. UID `999` matches the `postgres` user inside the official PostgreSQL image. Bind mounts must be writable by these UIDs or both containers will crash on first boot.

---

## 10. Phase 3 — Docker Compose Stack

### 10.1 Final `docker-compose.yaml`

Located at `/opt/sonarqube/docker-compose.yaml` (mode `0640 root:root`):

```yaml
services:
  db:
    image: postgres:16.14-bookworm
    container_name: sonarqube-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - /opt/sonarqube/postgres_data:/var/lib/postgresql/data
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
    # No ports: -> PostgreSQL is NOT reachable from the host or LAN.

  sonarqube:
    image: sonarqube:26.8.0.126808-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      SONAR_JDBC_URL: ${SONAR_JDBC_URL}
      SONAR_JDBC_USERNAME: ${SONAR_JDBC_USERNAME}
      SONAR_JDBC_PASSWORD: ${SONAR_JDBC_PASSWORD}
      SONAR_CORE_SERVERBASEURL: https://sonarqube.tech-bridge.biz
      SONAR_WEB_JAVAOPTS: "-Xmx1g -Xms512m"
      SONAR_CE_JAVAOPTS: "-Xmx1g -Xms512m"
      SONAR_SEARCH_JAVAOPTS: "-Xmx1g -Xms1g"
    volumes:
      - /opt/sonarqube/data:/opt/sonarqube/data
      - /opt/sonarqube/extensions:/opt/sonarqube/extensions
      - /opt/sonarqube/logs:/opt/sonarqube/logs
    ports:
      - "127.0.0.1:9000:9000"
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc:
        soft: 8192
        hard: 8192
    networks:
      - sonarnet
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

networks:
  sonarnet:
    driver: bridge
```

### 10.2 Why each piece exists

| Block | Reason |
|---|---|
| `image:` with pinned tags | Reproducibility; no `latest` drift. |
| `container_name:` | Stable names for `docker exec`, logs, and scripts. |
| `restart: unless-stopped` | Auto-recovery on reboot unless explicitly stopped. |
| `depends_on: db: condition: service_healthy` | SonarQube waits for PostgreSQL readiness before starting. |
| `healthcheck` on `db` | `pg_isready` ensures SonarQube does not start against an unready DB. |
| `volumes` as bind mounts | Data survives container recreation and is easy to back up. |
| `SONAR_JDBC_*` env vars | SonarQube reads these at boot to connect to PostgreSQL. |
| `SONAR_CORE_SERVERBASEURL` | Correct external URL for webhooks, links, notifications. |
| `*_JAVAOPTS` | Explicit heap sizes to prevent JVM from over-allocating on a 19 GiB VM. |
| `ports: 127.0.0.1:9000:9000` | Loopback only; Nginx is the sole ingress. |
| `ulimits` | Matches SonarQube documentation for `nofile` and `nproc`. |
| `networks: sonarnet` | Internal bridge; PostgreSQL is unreachable from the host. |
| `logging:` caps | Prevents JSON log files from growing unbounded on disk. |

---

## 11. Phase 3A — Offline/Online Image Selection

### 11.1 Tags chosen

| Component | Tag | Reason |
|---|---|---|
| SonarQube | `sonarqube:26.8.0.126808-community` | Current Community Build, pinned to a release. |
| PostgreSQL | `postgres:16.14-bookworm` | Pinned minor; within SonarQube 26.x supported range (14–18). |

**Never** use:
- `sonarqube:latest`
- `sonarqube:community` (floating)
- `postgres:16` (floating minor)
- `postgres:latest`

### 11.2 Pull

```bash
sudo docker pull sonarqube:26.8.0.126808-community
sudo docker pull postgres:16.14-bookworm
sudo docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}' | grep -E 'sonarqube|postgres'
```

### 11.3 Offline tarball path (for fully air-gapped environments)

On a machine with internet:

```bash
docker pull sonarqube:26.8.0.126808-community
docker pull postgres:16.14-bookworm
docker save sonarqube:26.8.0.126808-community -o sonarqube-26.8.0.tar
docker save postgres:16.14-bookworm -o postgres-16.14.tar
sha256sum sonarqube-26.8.0.tar postgres-16.14.tar
```

Transfer, then on the air-gapped VM:

```bash
sha256sum sonarqube-26.8.0.tar postgres-16.14.tar   # verify against source
sudo docker load -i sonarqube-26.8.0.tar
sudo docker load -i postgres-16.14.tar
sudo docker images | grep -E 'sonarqube|postgres'
```

**Never** `curl | bash`. **Never** import an image whose SHA256 you have not verified against a source you trust.

---

## 12. Phase 3B — Secrets Management

### 12.1 `.env` file

Location: `/opt/sonarqube/.env`, mode `0600`, owner `root:root`.

```
# PostgreSQL (container-internal only)
POSTGRES_DB=sonarqube
POSTGRES_USER=sonarqube
POSTGRES_PASSWORD=<32+ chars, generated locally>
SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonarqube
SONAR_JDBC_USERNAME=sonarqube
SONAR_JDBC_PASSWORD=<same as POSTGRES_PASSWORD>
```

### 12.2 Rules

- The two passwords **must** match. `POSTGRES_PASSWORD` is what the PostgreSQL container creates the user with; `SONAR_JDBC_PASSWORD` is what SonarQube uses to authenticate.
- Passwords are generated with `openssl rand -base64 32` and stored in the team password manager.
- Never use the `admin` SonarQube password, the Jenkins password, or the `cybersio` DB password here.
- `.env` is excluded by `.gitignore` in `/opt/sonarqube/`.
- Permissions verified with `stat -c '%a %U:%G %n' /opt/sonarqube/.env` → `600 root:root`.
- `.env` is never displayed in shell history, never pasted into chat, never echoed.

### 12.3 Compose invocation

Always pass the env file explicitly:

```bash
cd /opt/sonarqube
sudo docker compose --env-file /opt/sonarqube/.env up -d
```

This avoids relying on the file being auto-discovered and makes the secret source explicit.

### 12.4 Container-visible secrets

The SonarQube image reads `SONAR_JDBC_PASSWORD` from the container environment. On a single-host deployment with root already trusted, this is acceptable. The stronger alternative is Docker Swarm secrets, which require Swarm mode. For a future move to Kubernetes, use Kubernetes Secrets with `envFrom.secretRef`. For cloud, use AWS Secrets Manager / GCP Secret Manager / Azure Key Vault injected at task start.

---

## 13. Phase 3C — First Boot and Health Verification

### 13.1 Boot

```bash
cd /opt/sonarqube
sudo docker compose --env-file /opt/sonarqube/.env up -d
sleep 5
sudo docker compose ps
```

Expected:

```
NAME           IMAGE                               SERVICE     STATUS              PORTS
sonarqube      sonarqube:26.8.0.126808-community   sonarqube   Up                  127.0.0.1:9000->9000/tcp
sonarqube-db   postgres:16.14-bookworm             db          Up (healthy)        5432/tcp
```

Note: `db` shows `5432/tcp` with **no host mapping** — that is correct.

### 13.2 Watch logs

```bash
sudo docker compose logs -f sonarqube
```

Wait for:

```
INFO  app[][o.s.a.SchedulerImpl] SonarQube is operational
```

First boot takes 2–5 minutes. SonarQube will:
1. Connect to PostgreSQL via `jdbc:postgresql://db:5432/sonarqube`.
2. Run schema migrations.
3. Start Elasticsearch.
4. Start Web Server and Compute Engine.

### 13.3 Health API

```bash
curl -sS http://127.0.0.1:9000/api/system/status
```

Expected:

```json
{"id":"...","version":"26.8.0.126808","status":"UP"}
```

### 13.4 Port verification

```bash
sudo ss -tulpn | grep -E ':(80|443|9000|5432|8080)\b'
```

Expected:

- `80`, `443` → nginx
- `9000` → `127.0.0.1` only
- `8080` → `127.0.0.1` only
- `5432` → host PostgreSQL only. No container-side 5432.

### 13.5 Non-root verification

```bash
sudo docker exec sonarqube id
sudo docker exec sonarqube-db id
```

Expected:

- `uid=1000(sonarqube) gid=0(root)` or a non-root UID.
- `uid=999(postgres) gid=999(postgres)`.

Neither should be `uid=0(root)`.

### 13.6 Resource usage

```bash
sudo docker stats --no-stream sonarqube sonarqube-db
sudo docker inspect sonarqube --format '{{.HostConfig.Ulimits}}'
```

Expected:

- SonarQube ~1.5–3 GB resident.
- PostgreSQL under 500 MB.
- `Ulimits` shows `nofile` 65536 and `nproc` 8192.

### 13.7 Base URL check

```bash
sudo docker exec sonarqube printenv SONAR_CORE_SERVERBASEURL
```

Expected:

```
https://sonarqube.tech-bridge.biz
```

---

## 14. Phase 3D — Admin Hardening

### 14.1 Change default admin password

```bash
curl -sS -u admin:admin -X POST \
  "http://127.0.0.1:9000/api/users/change_password" \
  --data-urlencode "login=admin" \
  --data-urlencode "previousPassword=admin" \
  --data-urlencode "password=<NEW_STRONG_PASSWORD>" \
  -o /dev/null -w "HTTP %{http_code}\n"
```

Expected: `HTTP 204`.

Verify:

```bash
curl -sS -u admin:admin -o /dev/null -w "old %{http_code}\n" http://127.0.0.1:9000/api/system/status
curl -sS -u admin:<NEW_STRONG_PASSWORD> -o /dev/null -w "new %{http_code}\n" http://127.0.0.1:9000/api/system/status
```

Expected: old → `401`, new → `200`.

### 14.2 Force authentication (disable anonymous access)

```bash
curl -sS -u admin:<NEW_STRONG_PASSWORD> -X POST \
  "http://127.0.0.1:9000/api/settings/set" \
  --data-urlencode "key=sonar.forceAuthentication" \
  --data-urlencode "value=true" \
  -o /dev/null -w "HTTP %{http_code}\n"
```

Expected: `HTTP 204`. Any unauthenticated request now redirects to `/sessions/new`.

### 14.3 Confirm force-auth from Nginx

```bash
curl -sS -o /dev/null -w "sonarqube root HTTP %{http_code}\n" https://sonarqube.tech-bridge.biz/
```

Expected: `302` or `200` landing on the login page.

---

## 15. Phase 3E — Nginx Reverse Proxy and TLS

### 15.1 DNS prerequisite

`sonarqube.tech-bridge.biz` → `192.168.0.17` (added by IT). Confirmed via:

```bash
getent hosts sonarqube.tech-bridge.biz
```

### 15.2 Certificate

The wildcard certificate at `/etc/nginx/ssl/fullchain.pem` covers `*.tech-bridge.biz`. No reissue needed. Verify:

```bash
sudo openssl x509 -in /etc/nginx/ssl/fullchain.pem -noout -ext subjectAltName
```

Expected: `DNS:*.tech-bridge.biz, DNS:tech-bridge.biz`.

### 15.3 Vhost

Location: `/etc/nginx/sites-available/sonarqube` (mode `0644 root:root`).

```nginx
# SonarQube vhost — internal only.
# TLS terminated here using the shared wildcard cert.
# Upstream: SonarQube container bound to 127.0.0.1:9000.

server {
    listen 80;
    server_name sonarqube.tech-bridge.biz;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name sonarqube.tech-bridge.biz;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;
    ssl_session_tickets off;

    client_max_body_size 100m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    access_log /var/log/nginx/sonarqube.access.log;
    error_log  /var/log/nginx/sonarqube.error.log;

    location / {
        proxy_pass http://127.0.0.1:9000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout    300;
        proxy_send_timeout    300;
        proxy_connect_timeout 30;
    }
}
```

### 15.4 Notes on syntax

- `listen 443 ssl http2;` — the correct syntax for Nginx 1.24 (Ubuntu 24.04). The newer `http2 on;` directive requires Nginx 1.25.1+ and would fail `nginx -t`.
- `ssl_protocols TLSv1.2 TLSv1.3;` — scoped override. The global `nginx.conf` still allows TLS 1.0/1.1 for Jenkins. Flag as a separate hardening item.
- `client_max_body_size 100m` — matches the Jenkins vhost; SonarQube analysis uploads can be large.
- Separate access and error logs prevent SonarQube traffic from mixing with Jenkins logs.
- `X-Forwarded-Proto https` tells SonarQube it is behind a TLS terminator, so generated links use HTTPS.

### 15.5 Enable and validate

```bash
sudo ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
sudo nginx -t
sudo systemctl reload nginx
```

Expected: `syntax is ok`, `test is successful`, then reload silently succeeds.

### 15.6 Verify

```bash
curl -sS -o /dev/null -w "jenkins HTTP %{http_code}\n" https://jenkins.tech-bridge.biz/login
curl -sS -o /dev/null -w "sonarqube HTTP %{http_code}\n" https://sonarqube.tech-bridge.biz/
curl -sS -o /dev/null -w "sonarqube redirect HTTP %{http_code}\n" http://sonarqube.tech-bridge.biz/

openssl s_client -connect sonarqube.tech-bridge.biz:443 -servername sonarqube.tech-bridge.biz </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -ext subjectAltName
```

Expected:

- Jenkins: `200` or `302`.
- SonarQube: `302` (redirect to `/sessions/new`) or `200` (login page).
- HTTP redirect: `301`.
- TLS: SAN includes `*.tech-bridge.biz`, issuer matches the Jenkins cert.

Browser screenshot confirmed the login page renders over HTTPS with a valid lock icon.

---

## 16. Phase 3F — DNS Resolution

The VM was originally unable to resolve external hostnames because `systemd-resolved` had no upstream DNS configured.

Fix:

```bash
sudo tee /etc/systemd/resolved.conf.d/99-internal-dns.conf >/dev/null <<'EOF'
[Resolve]
DNS=192.168.0.146
FallbackDNS=
Domains=~.
EOF

sudo systemctl restart systemd-resolved
```

Verify:

```bash
resolvectl status | sed -n '1,40p'
getent hosts registry-1.docker.io
getent hosts sonarqube.tech-bridge.biz
getent hosts jenkins.tech-bridge.biz
```

Expected:

- `resolvectl` shows `Current DNS Server: 192.168.0.146`.
- All three `getent` commands return IPs. `sonarqube` and `jenkins` resolve to `192.168.0.17`.

**Design note:** `FallbackDNS=` is intentionally empty. In a cybersecurity-company environment, silent fallback to public DNS leaks internal names. Failure of the internal resolver should be loud.

---

## 17. Verification Checklist

Run all of these and confirm each passes.

| # | Check | Command | Expected |
|---|---|---|---|
| 1 | Containers running | `docker compose ps` | Both `Up`, `db` healthy |
| 2 | SonarQube API | `curl -sS http://127.0.0.1:9000/api/system/status` | `"status":"UP"` |
| 3 | 9000 loopback-only | `sudo ss -tulpn \| grep :9000` | `127.0.0.1:9000` |
| 4 | 5432 not container-published | `sudo ss -tulpn \| grep :5432` | host PostgreSQL only |
| 5 | 8080 loopback-only | `sudo ss -tulpn \| grep :8080` | `127.0.0.1:8080` |
| 6 | HTTPS login page | Browser to `https://sonarqube.tech-bridge.biz` | Valid cert, login page |
| 7 | HTTP redirects | `curl -I http://sonarqube.tech-bridge.biz/` | `301` |
| 8 | Jenkins untouched | `curl -I https://jenkins.tech-bridge.biz/login` | `200`/`302` |
| 9 | Admin password changed | Login as `admin` with new password | Succeeds |
| 10 | Anonymous access disabled | Unauthenticated browse | Redirect to `/sessions/new` |
| 11 | Sysctl persisted | `cat /etc/sysctl.d/99-sonarqube.conf` | Both lines present |
| 12 | `.env` perms | `stat -c '%a %U:%G' /opt/sonarqube/.env` | `600 root:root` |
| 13 | Compose perms | `stat -c '%a %U:%G' /opt/sonarqube/docker-compose.yaml` | `640 root:root` |
| 14 | Container user non-root | `docker exec sonarqube id` | non-zero UID |
| 15 | Container logs capped | `docker inspect sonarqube --format '{{.HostConfig.LogConfig}}'` | `json-file` with `max-size` |

---

## 18. Operations Runbook

### 18.1 Start / stop / restart

```bash
cd /opt/sonarqube

# Start
sudo docker compose --env-file /opt/sonarqube/.env up -d

# Stop (preserve data — do NOT add -v)
sudo docker compose down

# Stop and remove containers + network but keep volumes
sudo docker compose down

# Restart a single service
sudo docker compose restart sonarqube

# Restart everything
sudo docker compose restart
```

**Never run `docker compose down -v`.** That would delete the `postgres_data` and `data` bind mounts and destroy the SonarQube installation.

### 18.2 View status

```bash
cd /opt/sonarqube
sudo docker compose ps
sudo docker compose logs --tail=100 sonarqube
sudo docker compose logs --tail=50 db
```

### 18.3 Shell access (for diagnostics only)

```bash
sudo docker exec -it sonarqube bash
sudo docker exec -it sonarqube-db psql -U sonarqube -d sonarqube
```

### 18.4 Resource usage

```bash
sudo docker stats --no-stream
sudo docker system df
df -hT /
```

**Low-water alarm:** if `/` free space drops below 15 GB, stop SonarQube (`docker compose down`) and investigate before restarting.

### 18.5 Health checks

```bash
curl -sS http://127.0.0.1:9000/api/system/status
curl -sS -o /dev/null -w "%{http_code}\n" https://sonarqube.tech-bridge.biz/
sudo docker inspect --format '{{.State.Health.Status}}' sonarqube-db
```

---

## 19. Logging and Rotation

### 19.1 Container logs

Both services use the `json-file` driver with `max-size: 10m` and `max-file: 5`. Maximum disk per container: 50 MB. Total: 100 MB.

### 19.2 SonarQube internal logs

Located on the host at `/opt/sonarqube/logs/`. SonarQube rotates its own `web.log`, `ce.log`, `es.log`, `access.log` internally.

### 19.3 Nginx logs

SonarQube traffic is isolated to:

```
/var/log/nginx/sonarqube.access.log
/var/log/nginx/sonarqube.error.log
```

These are picked up by the existing logrotate configuration for `/var/log/nginx/*.log`.

### 19.4 PostgreSQL logs

Written to the container stdout/stderr, captured by `json-file` driver. Available via:

```bash
sudo docker compose logs db
```

---

## 20. Backup Strategy

### 20.1 What must be backed up

| Item | Path | Frequency | Notes |
|---|---|---|---|
| PostgreSQL logical dump | `/opt/sonarqube/backups/` | Daily | `pg_dump` inside `db` container |
| SonarQube config | `/opt/sonarqube/conf/` | Weekly | mostly static |
| SonarQube plugins | `/opt/sonarqube/extensions/` | Weekly | reproducible from Marketplace |
| SonarQube data | `/opt/sonarqube/data/` | Weekly | Elasticsearch indexes; regenerable but expensive |
| `.env` | `/opt/sonarqube/.env` | On change | store in password manager, not just here |
| `docker-compose.yaml` | `/opt/sonarqube/` | On change | version-control in a private repo |
| Nginx vhost | `/etc/nginx/sites-available/sonarqube` | On change | version-control |
| Cert | `/etc/nginx/ssl/` | On renewal | already handled by IT |

### 20.2 PostgreSQL dump

```bash
sudo docker exec sonarqube-db pg_dump -U sonarqube -Fc sonarqube \
  > /opt/sonarqube/backups/sonarqube-$(date +%F).dump
```

Restore test (monthly, to a throwaway container):

```bash
sudo docker exec -i <test-db> pg_restore -U sonarqube -d sonarqube --clean < sonarqube-<date>.dump
```

### 20.3 Off-host shipping

Backups must not live only on this VM. Use rsync to a NAS or a secondary host:

```bash
rsync -av --delete /opt/sonarqube/backups/ backup@nas:/srv/backups/sonarqube/
```

Wire this into a systemd timer or cron once the destination is confirmed.

### 20.4 Retention

| Tier | Retention |
|---|---|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

Implement rotation on the destination, not on the source.

### 20.5 Restore validation

A backup that has never been restored is not a backup. Monthly: restore the latest dump into a throwaway container on a non-production host, boot SonarQube against it, confirm `/api/system/status` returns `UP`, then destroy the throwaway.

---

## 21. Restore Procedure

Full restore from scratch:

```bash
# 1. Stop the stack
cd /opt/sonarqube
sudo docker compose down

# 2. Wipe PostgreSQL data directory (destructive!)
sudo rm -rf /opt/sonarqube/postgres_data/*
sudo install -d -o 999 -g 999 -m 0750 /opt/sonarqube/postgres_data

# 3. Bring up db only
sudo docker compose --env-file /opt/sonarqube/.env up -d db

# 4. Wait for healthy
sleep 20
sudo docker compose ps

# 5. Restore
sudo docker exec -i sonarqube-db pg_restore -U sonarqube -d sonarqube --clean --if-exists \
  < /path/to/sonarqube-<date>.dump

# 6. Bring up SonarQube
sudo docker compose --env-file /opt/sonarqube/.env up -d sonarqube

# 7. Watch logs
sudo docker compose logs -f sonarqube
```

Validate:

```bash
curl -sS http://127.0.0.1:9000/api/system/status
```

Expected: `"status":"UP"`.

---

## 22. Upgrade Procedure

SonarQube upgrades are not trivial. Follow this sequence.

### 22.1 Before upgrade

1. Read the SonarQube upgrade notes for the target version. Check:
   - PostgreSQL version compatibility matrix.
   - Java version requirements.
   - Elasticsearch index format changes (can require reindexing).
2. Confirm the target Community Build tag exists.
3. Take a full off-host backup (PostgreSQL dump, `conf/`, `extensions/`, `data/`).
4. Note the current tag in `docker-compose.yaml`.

### 22.2 Upgrade

```bash
# 1. Pull new image
sudo docker pull sonarqube:<NEW_TAG>

# 2. Stop stack
cd /opt/sonarqube
sudo docker compose down

# 3. Edit docker-compose.yaml, change sonarqube image tag
sudo nano /opt/sonarqube/docker-compose.yaml

# 4. Start with new image
sudo docker compose --env-file /opt/sonarqube/.env up -d

# 5. Watch logs
sudo docker compose logs -f sonarqube
```

Wait for `SonarQube is operational`.

### 22.3 After upgrade

```bash
curl -sS http://127.0.0.1:9000/api/system/status
curl -sS -o /dev/null -w "%{http_code}\n" https://sonarqube.tech-bridge.biz/
```

Run a smoke analysis from a scanner to confirm end-to-end.

### 22.4 Keep the previous image

Do **not** delete the previous image immediately. Keep it locally for at least one release cycle so rollback is fast.

---

## 23. Rollback Procedure

### 23.1 Rollback a container state change

```bash
cd /opt/sonarqube
sudo docker compose down
# edit docker-compose.yaml back to the previous tag
sudo docker compose --env-file /opt/sonarqube/.env up -d
```

### 23.2 Rollback after a failed upgrade

1. Stop the stack.
2. Wipe `postgres_data/`.
3. Restore the pre-upgrade PostgreSQL dump.
4. Restore `conf/`, `extensions/`, `data/` from the pre-upgrade snapshot.
5. Revert the image tag in `docker-compose.yaml`.
6. Start the stack.

**Note:** Elasticsearch index format may not roll back cleanly. Snapshot `data/` before every upgrade.

### 23.3 Rollback the Nginx vhost

```bash
sudo rm /etc/nginx/sites-enabled/sonarqube
sudo nginx -t
sudo systemctl reload nginx
```

SonarQube containers keep running; local access via `127.0.0.1:9000` remains.

### 23.4 Rollback DNS change

```bash
sudo rm /etc/systemd/resolved.conf.d/99-internal-dns.conf
sudo systemctl restart systemd-resolved
sudo systemctl restart docker
```

Reverts to the pre-change (broken external DNS) state. Only do this if the new DNS config breaks something else.

---

## 24. Troubleshooting Guide

### 24.1 SonarQube container will not start

```bash
sudo docker compose logs --tail=200 sonarqube
```

Common causes:

- `vm.max_map_count too low` → check `/etc/sysctl.d/99-sonarqube.conf`, run `sudo sysctl --system`.
- `cannot write to /opt/sonarqube/data` → check ownership `1000:1000` on `/opt/sonarqube/{data,extensions,logs}`.
- DB connection refused → check `db` is healthy: `docker compose ps`.

### 24.2 SonarQube API returns DOWN

```bash
curl -sS http://127.0.0.1:9000/api/system/status
sudo docker compose logs --tail=120 sonarqube
```

Possible causes:

- Elasticsearch bootstrap failure.
- Schema migration failure.
- Disk full.

### 24.3 Nginx returns 502

```bash
curl -sS http://127.0.0.1:9000/api/system/status
sudo tail -50 /var/log/nginx/sonarqube.error.log
```

If the API check fails, SonarQube itself is the problem. If it succeeds, Nginx cannot reach `127.0.0.1:9000`.

### 24.4 Nginx returns 404

```bash
sudo nginx -T | grep -A3 "server_name sonarqube"
```

Confirm the vhost is loaded. Confirm `sites-enabled/sonarqube` symlink exists.

### 24.5 TLS errors in browser

```bash
openssl s_client -connect sonarqube.tech-bridge.biz:443 -servername sonarqube.tech-bridge.biz </dev/null
```

Check SAN, expiry, issuer. Compare to the Jenkins cert.

### 24.6 DNS does not resolve

```bash
getent hosts sonarqube.tech-bridge.biz
resolvectl status | sed -n '1,40p'
```

If internal DNS does not resolve, check `99-internal-dns.conf` and reachability to `192.168.0.146`.

### 24.7 Disk full

```bash
df -hT /
sudo du -xh /opt/sonarqube | sort -h | tail -20
sudo docker system df
```

Free space with:

```bash
sudo docker builder prune
sudo docker image prune
# Do NOT prune volumes without review.
```

If free < 15 GB, stop SonarQube (`docker compose down`) and investigate.

### 24.8 Cannot log in

Check the API first:

```bash
curl -sS -u admin:<password> -o /dev/null -w "%{http_code}\n" http://127.0.0.1:9000/api/system/status
```

- `200` → password correct; browser autofill or typo.
- `401` → password wrong.
- `403` → account locked from too many failures.

Account recovery requires admin access. Do not wipe data; use DB-level or env-level recovery if needed.

---

## 25. Jenkins Integration (Planned)

This section describes the target integration. Actual implementation happens in Phase 4.

### 25.1 Install the SonarQube Scanner plugin

Jenkins → Manage Jenkins → Plugins → Available → search **SonarQube Scanner** → install → restart.

Alternative if the Update Center is offline: install the `.hpi` from the Jenkins plugin archive.

### 25.2 Create a least-privilege analysis token

**Do not use the admin account or an admin token.**

1. In SonarQube, create a dedicated user, e.g. `jenkins-ci`.
2. Grant that user only the necessary permissions on the projects it analyzes.
3. In SonarQube → My Account → Security → Generate Token, choose **Project Analysis Token** scoped to the target project.
4. Store the token in the password manager.

### 25.3 Store the token in Jenkins

Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials:

- Kind: **Secret text**
- ID: `sonarqube-analysis-token`
- Secret: the token from 25.2
- Description: "SonarQube analysis token for tb-SOAR"

Never hard-code tokens in Jenkinsfiles.

### 25.4 Configure SonarQube server in Jenkins Global Configuration

Jenkins → Manage Jenkins → System → **SonarQube servers**:

- Name: `SonarQube`
- Server URL: `https://sonarqube.tech-bridge.biz`
- Server authentication token: select `sonarqube-analysis-token`
- Check **Enable injection of SonarQube server configuration as build environment variables**.

### 25.5 Webhook from SonarQube to Jenkins

In SonarQube: Administration → Configuration → Webhooks → Create:

- Name: `Jenkins`
- URL: `https://jenkins.tech-bridge.biz/sonarqube-webhook/` (**trailing slash required**)
- Secret: generate a strong secret, store it in Jenkins as a Secret text credential with ID `sonarqube-webhook-secret`.

### 25.6 Pipeline usage pattern

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'sonar-scanner -Dsonar.projectKey=tb-soar-smoke -Dsonar.sources=.'
        }
    }
}

stage('Quality Gate') {
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true,
                              credentialsId: 'sonarqube-webhook-secret'
        }
    }
}
```

**Explain:**

- `withSonarQubeEnv('SonarQube')` injects `SONAR_HOST_URL` and `SONAR_AUTH_TOKEN` from the Jenkins Global Configuration into the build step, so the scanner uploads to SonarQube automatically.
- `waitForQualityGate` pauses the pipeline until SonarQube sends the webhook with the quality-gate result. The `timeout` prevents an indefinite wait if the webhook never arrives.
- `abortPipeline: true` fails the build if the gate fails.

### 25.7 Reachability checks

**Jenkins → SonarQube:**

```bash
# From the Jenkins host:
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer <TOKEN>" \
  https://sonarqube.tech-bridge.biz/api/system/status
```

Expected: `200`.

**SonarQube → Jenkins:**

```bash
# From inside the SonarQube container:
sudo docker exec sonarqube sh -c 'curl -sS -o /dev/null -w "%{http_code}\n" https://jenkins.tech-bridge.biz/sonarqube-webhook/'
```

Expected: `200` or `302`. Not a TLS error.

### 25.8 Smoke-test Jenkinsfile

Use a disposable project key. Do not leak secrets.

```groovy
pipeline {
    agent any
    tools { nodejs 'NodeJS_24.5.0' }
    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('SonarQube Smoke') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -e
                        sonar-scanner \
                          -Dsonar.projectKey=tb-soar-smoke \
                          -Dsonar.projectName="tb-SOAR Smoke Test" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true,
                                      credentialsId: 'sonarqube-webhook-secret'
                }
            }
        }
    }
    post {
        always { cleanWs() }
    }
}
```

---

## 26. Cloud Migration Path

The design was intentionally chosen so that this stack ports cleanly to AWS, GCP, or Azure.

### 26.1 What maps directly

| On-prem component | Cloud equivalent |
|---|---|
| Ubuntu VM | EC2 / GCE / Azure VM, or container service |
| Docker Compose | ECS Task Definitions, EKS/GKE/AKS Pods, or the same Compose on a VM |
| Bind mount `/opt/sonarqube/data` | EBS / Persistent Disk / Azure Disk, or EFS / Filestore / Azure Files |
| `.env` file | AWS Secrets Manager / GCP Secret Manager / Azure Key Vault |
| Nginx vhost + wildcard cert | ALB with ACM cert, or Nginx Ingress with cert-manager |
| Internal DNS `192.168.0.146` | Route 53 Private Zone / Cloud DNS Private Zone / Azure Private DNS |
| `127.0.0.1:9000` binding | Remove the `ports:` block; expose the container port to the task’s internal network only |

### 26.2 Concrete migration patterns

**Option A — Lift and shift to a cloud VM.**

- Provision an Ubuntu VM.
- Install Docker + Compose.
- Copy `/opt/sonarqube` to the VM (data, compose, `.env`).
- Point `.env` and compose at cloud-managed PostgreSQL (RDS/Cloud SQL) if desired.
- Attach the internal DNS name to the VM’s private IP.
- Replace Nginx vhost with either the same Nginx install or an ALB.

**Option B — Containerized on ECS/EKS.**

- Convert the compose file into Kubernetes manifests or an ECS task definition.
- Mount persistent volumes backed by EBS/EFS.
- Pull secrets from Secrets Manager via `envFrom.secretRef` or ECS `secrets:`.
- Replace Nginx vhost with:
  - Kubernetes: Ingress + cert-manager (or ALB Ingress Controller with ACM).
  - ECS: ALB with ACM certificate, listener rule matching `sonarqube.tech-bridge.biz`.
- Hostname resolves through Route 53 Private Hosted Zone.

**Option C — Managed SonarQube (SonarCloud or SonarQube Server on cloud).**

- Move to SonarSource’s cloud offering.
- Keep scanner configuration the same (server URL changes).
- Local secrets become cloud secrets.

### 26.3 Cloud-specific configuration changes

| Item | On-prem | Cloud |
|---|---|---|
| Container port binding | `127.0.0.1:9000:9000` | remove `ports:` and let the service mesh / task network handle ingress |
| TLS cert source | Wildcard from IT, Nginx vhost | ACM / Let’s Encrypt via cert-manager / Key Vault |
| DNS | On-prem internal DNS `192.168.0.146` | Private Hosted Zone (Route 53, Cloud DNS, Private DNS) |
| Secrets | `/opt/sonarqube/.env` | Secrets Manager / Secret Manager / Key Vault |
| Backups | rsync to NAS | RDS automated backups + EBS snapshots + S3 lifecycle |
| Logs | `json-file` + `/opt/sonarqube/logs` | CloudWatch Logs / Cloud Logging / Azure Monitor |
| Sysctl (`vm.max_map_count`) | `/etc/sysctl.d/99-sonarqube.conf` | Fargate handles this; on EKS use a privileged init container or node-level tuning |

### 26.4 What does not need to change

- SonarQube and PostgreSQL versions.
- SonarQube settings, projects, users, tokens.
- Jenkins integration (scanner URL changes but plugin config is identical).
- Quality Gate rules.
- `sonar-project.properties` in the repository.

### 26.5 Testing plan for migration

1. Stand up parallel SonarQube in cloud.
2. Restore PostgreSQL backup into the cloud instance.
3. Verify `/api/system/status` returns `UP`.
4. Run a smoke analysis against a test project key.
5. Compare Quality Gate results to on-prem.
6. Cut over DNS.
7. Decommission on-prem stack after a soak period.

---

## 27. Security Considerations

### 27.1 Achieved

- SonarQube and PostgreSQL are not reachable from the LAN except via Nginx.
- PostgreSQL has no host port at all.
- SonarQube is bound to `127.0.0.1:9000` only.
- HTTPS terminates at Nginx with a valid wildcard certificate.
- HTTP requests are 301-redirected to HTTPS.
- Anonymous access is disabled (`sonar.forceAuthentication=true`).
- Admin password is non-default.
- Secrets live in a `0600 root:root` `.env` file, excluded from Git.
- Container logs are capped at 100 MB total.
- Kernel and container limits match SonarQube documentation.
- `X-Frame-Options`, `X-Content-Type-Options`, HSTS, and Referrer-Policy headers on the vhost.

### 27.2 Outstanding, flagged for separate action

- **Global Nginx TLS policy** still allows TLS 1.0 and 1.1 for the Jenkins vhost. SonarQube overrides to TLS 1.2/1.3. To fix Jenkins too, edit `/etc/nginx/nginx.conf` and set `ssl_protocols TLSv1.2 TLSv1.3;` at the `http` level. Do this as a separate change with a maintenance window.
- **Host PostgreSQL on `0.0.0.0:5432`** is exposed to the LAN. This is a pre-existing security finding unrelated to SonarQube. Recommended fix: bind it to `127.0.0.1` or firewall 5432 to trusted hosts only. Requires application owner approval because it serves `cybersio`.
- **Wildcard certificate sharing.** The wildcard private key at `/etc/nginx/ssl/server.key` is used for both Jenkins and SonarQube. Compromise of that key affects both. Verified `0600 root:root` permissions. Consider per-service certificates in the future.
- **Image tag drift.** Application images built by the CD pipeline use `:latest`. Recommend pinning by digest or SHA-tagged builds.
- **Backups off-host** are designed but not yet wired to a destination. Wire rsync once the NAS/host is confirmed.
- **Rate limiting on Nginx** for SonarQube vhost is not configured. Consider adding `limit_req_zone` for `/api/` if abuse is a concern.

### 27.3 What not to do

- Do not publish 9000 or 5432 to `0.0.0.0`.
- Do not use the admin token in Jenkins jobs.
- Do not paste `.env` contents into chat, tickets, or code.
- Do not run `docker compose down -v`.
- Do not run `docker system prune --volumes` on this host — 50 pre-existing named volumes depend on it.
- Do not `chmod 777` anything under `/opt/sonarqube`.
- Do not install a public SonarQube plugin from an untrusted source.

---

## 28. File Inventory

### 28.1 SonarQube stack

```
/opt/sonarqube/
├── .env
├── .gitignore
├── docker-compose.yaml
├── data/
├── extensions/
├── logs/
├── postgres_data/
└── postgres_init/
```

### 28.2 System configuration

```
/etc/sysctl.d/99-sonarqube.conf
/etc/systemd/resolved.conf.d/99-internal-dns.conf
```

### 28.3 Nginx

```
/etc/nginx/sites-available/sonarqube
/etc/nginx/sites-enabled/sonarqube -> /etc/nginx/sites-available/sonarqube
/etc/nginx/ssl/fullchain.pem
/etc/nginx/ssl/server.key
```

### 28.4 Logs

```
/var/log/nginx/sonarqube.access.log
/var/log/nginx/sonarqube.error.log
```

### 28.5 Backups (on host, then shipped off-host)

```
/opt/sonarqube/backups/sonarqube-<date>.dump
```

---

## 29. Command Cheat Sheet

### Compose

```bash
cd /opt/sonarqube
sudo docker compose ps
sudo docker compose --env-file /opt/sonarqube/.env up -d
sudo docker compose down
sudo docker compose restart sonarqube
sudo docker compose logs -f sonarqube
```

### Health

```bash
curl -sS http://127.0.0.1:9000/api/system/status
curl -sS -o /dev/null -w "%{http_code}\n" https://sonarqube.tech-bridge.biz/
```

### Ports

```bash
sudo ss -tulpn | grep -E ':(80|443|9000|5432|8080)\b'
```

### Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo tail -f /var/log/nginx/sonarqube.error.log
```

### TLS

```bash
openssl s_client -connect sonarqube.tech-bridge.biz:443 -servername sonarqube.tech-bridge.biz </dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

### DNS

```bash
getent hosts sonarqube.tech-bridge.biz
resolvectl status | sed -n '1,40p'
```

### Docker inspect

```bash
sudo docker inspect sonarqube --format '{{.HostConfig.LogConfig}}'
sudo docker inspect sonarqube --format '{{.HostConfig.Ulimits}}'
sudo docker exec sonarqube id
```

### Backup

```bash
sudo docker exec sonarqube-db pg_dump -U sonarqube -Fc sonarqube \
  > /opt/sonarqube/backups/sonarqube-$(date +%F).dump
```

---

## 30. Mental Model

When debugging, work from the outside in:

```
Browser
  ↓
DNS  ── getent hosts sonarqube.tech-bridge.biz
  ↓
TCP  ── ss -tulpn | grep :443
  ↓
TLS  ── openssl s_client
  ↓
Nginx ── sudo nginx -T | grep -A3 sonarqube
  ↓
Reverse proxy ── curl -I https://sonarqube.tech-bridge.biz/
  ↓
SonarQube API ── curl http://127.0.0.1:9000/api/system/status
  ↓
SonarQube app ── docker compose logs sonarqube
  ↓
PostgreSQL ── docker compose logs db
```

Fix the first layer that fails. Do not skip layers.

**One-line interview summary:**

> SonarQube Community Build is deployed on the Jenkins VM via Docker Compose with an isolated PostgreSQL container, bound to `127.0.0.1:9000`, fronted by Nginx that terminates TLS using the existing wildcard certificate on `sonarqube.tech-bridge.biz`, with anonymous access disabled, secrets in a root-only `.env`, kernel and container limits persisted, and a design that lifts cleanly to cloud with only DNS, TLS, and secret-store changes.

---

## End of Document

This document is intended to be the single source of truth for the SonarQube on-prem deployment and its planned cloud migration. Keep it in version control alongside the compose file and Nginx vhost. Update the operations section whenever the deployment topology changes, and update the Jenkins integration section once Phase 4 completes.
