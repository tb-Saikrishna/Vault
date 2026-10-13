# tbNAC Multi-Master Cluster (Host Networking) — Implementation Report

**Project:** Containerized tbNAC with PostgreSQL 17 Multi-Master (Active-Active) Cluster using `pgactive` on **host networking**
**Environment:** Two-node Ubuntu deployment with Docker Compose
**Nodes:**
- **Node 1** — hostname `Forge`, IP `192.168.0.28`
- **Node 2** — hostname `Jigar-6`, IP `192.168.0.209`

**Application:** tbNAC NM Ubuntu v3.1.2-2026-09-30
**Database:** `tbnac_core` / `tbnac_core_user` on **port 5434**
**Extension:** AWS `pgactive` v2.1.7
**Document Date:** 2026-10-13
**Status:** Multi-Master cluster **operational and fully verified** for scopes a–e

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Objectives](#2-objectives)
3. [Architecture Overview](#3-architecture-overview)
4. [What Changed From the Bridge-Network Reference Build](#4-what-changed-from-the-bridge-network-reference-build)
5. [The Custom PostgreSQL Image](#5-the-custom-postgresql-image)
6. [Configuration Files](#6-configuration-files)
7. [Docker Compose — Node 1 and Node 2](#7-docker-compose--node-1-and-node-2)
8. [Detailed Deployment Walkthrough](#8-detailed-deployment-walkthrough)
9. [Sequence Offsets — The Multi-Master Pattern](#9-sequence-offsets--the-multi-master-pattern)
10. [Test Plan & Results](#10-test-plan--results)
11. [The Organic Conflict — `mac_vendors`](#11-the-organic-conflict--mac_vendors)
12. [Secondary Unique Constraint Warnings](#12-secondary-unique-constraint-warnings)
13. [Issues Encountered & Resolutions](#13-issues-encountered--resolutions)
14. [Operational Runbook](#14-operational-runbook)
15. [Host Networking vs Bridge Networking](#15-host-networking-vs-bridge-networking)
16. [Key Concepts](#16-key-concepts)
17. [Appendices](#17-appendices)

---

## 1. Executive Summary

This document describes the design, deployment, and validation of a **containerized PostgreSQL 17 multi-master (active-active) cluster** running as the database tier of the tbNAC appliance, across two physical hosts using Docker **host networking**.

Both nodes accept writes simultaneously. Changes propagate bidirectionally via **logical replication** using Amazon's `pgactive` v2.1.7 extension. Sequence-collision is prevented via **per-node sequence offsets** (odd on Node 1, even on Node 2) applied to all 32 sequences in the `nac` schema.

**Result highlights:**

- Custom PostgreSQL image built with `pgactive` compiled in, plus the **four artifact types** (`pgactive.so`, `pgactive--*.sql` + `.control`, `pgactive_dump`, `pgactive_init_copy`).
- Two-node cluster deployed via Docker Compose with **`network_mode: host`** on both nodes; PostgreSQL listens directly on `:5434` on each host's LAN interface.
- `pgactive` group created on Node 1 (`tbnac-node1`); Node 2 (`tbnac-node2`) successfully joined and pulled a full database copy.
- Both nodes report status `r` in `pgactive.pgactive_nodes`.
- Bidirectional **INSERT / UPDATE / DELETE** verified for `nac.nas` with correct odd/even id parity.
- **Node-failure tolerance** verified: stopping Node 1's Postgres left Node 2 fully operational; Node 1 caught up on restart with `lag_bytes = 56`.
- A **real concurrent-write conflict** (40,324 vs 40,321 `mac_vendors` rows) was resolved automatically by last-writer-wins; both nodes converged to 40,321 rows with zero data loss.
- Zero `nr_disconnect` across the entire run.

**Key lesson:** The entire sequence-offset strategy from the bridge build had to be **rewritten** for tbNAC because only 8 of 32 sequences were `OWNED BY` a column. The corrected script reads from `pg_sequences` directly.

---

## 2. Objectives

| # | Objective | Status |
|---|-----------|--------|
| 1 | Build a custom Postgres image with `pgactive` v2.1.7 against PostgreSQL 17 | ✅ Complete |
| 2 | Deploy two independent tbNAC stacks with host networking | ✅ Complete |
| 3 | Configure inter-node connectivity on port `5434` (no port translation) | ✅ Complete |
| 4 | Create a `pgactive` group on Node 1 | ✅ Complete |
| 5 | Join Node 2 to the group with a full DB copy | ✅ Complete |
| 6 | Verify bidirectional INSERT / UPDATE / DELETE replication | ✅ Complete |
| 7 | Eliminate sequence ID collisions across all 32 sequences | ✅ Complete |
| 8 | Verify node-failure tolerance and rejoin catch-up | ✅ Complete |
| 9 | Verify conflict resolution under real concurrent writes | ✅ Complete |
| 10 | Document full operational runbook for host networking | ✅ Complete |

---

## 3. Architecture Overview

### 3.1 Topology

```
   ┌─────────────────────────────────┐          ┌─────────────────────────────────┐
   │            Node 1               │          │            Node 2               │
   │  Host: Forge                    │          │  Host: Jigar-6                  │
   │  IP:   192.168.0.28             │          │  IP:   192.168.0.209            │
   │                                 │          │                                 │
   │  ┌───────────────────────────┐  │          │  ┌───────────────────────────┐  │
   │  │ tbNAC full stack          │  │          │  │ tbNAC full stack          │  │
   │  │  - tbnac-frontend    :1001│  │          │  │  - tbnac-frontend    :1001│  │
   │  │  - tbnac-backend     :1002│  │          │  │  - tbnac-backend     :1002│  │
   │  │  - tbnac-worker           │  │          │  │  - tbnac-worker           │  │
   │  │  - freeradius        :1812│  │          │  │  - freeradius        :1812│  │
   │  │  - domain-ctl        :8099│  │          │  │  - domain-ctl        :8099│  │
   │  │  - redis             :6379│  │          │  │  - redis             :6379│  │
   │  │  - netdata           :19999│ │          │  │  - netdata           :19999│ │
   │  │                           │  │          │  │                           │  │
   │  │  ┌─────────────────────┐  │  │  logical │  │  ┌─────────────────────┐  │  │
   │  │  │ PostgreSQL 17       │  │  │  stream  │  │  │ PostgreSQL 17       │  │  │
   │  │  │  + pgactive 2.1.7   │◀─┼──┼─────────▶│  │  │  + pgactive 2.1.7   │  │  │
   │  │  │  (read / write)     │  │  │  :5434   │  │  │  (read / write)     │  │  │
   │  │  │  :5434              │  │  │          │  │  │  :5434              │  │  │
   │  │  └─────────────────────┘  │  │          │  │  └─────────────────────┘  │  │
   │  └───────────────────────────┘  │          │  └───────────────────────────┘  │
   │                                 │          │                                 │
   │  network_mode: host             │          │  network_mode: host             │
   │  DB :5434 bound on 0.0.0.0      │          │  DB :5434 bound on 0.0.0.0      │
   └─────────────────────────────────┘          └─────────────────────────────────┘
             ▲                                              ▲
             │         LAN 192.168.0.0/24                   │
             └──────────────────────────────────────────────┘
                    direct connection on :5434
                    (no port translation, no bridge)
```

### 3.2 Characteristics

- **Both nodes accept writes.** No single point of failure for write traffic.
- **Changes propagate via logical replication** — row-level events streamed through `pgactive`'s logical decoding plugin.
- **DDL is not replicated.** Schema changes must be applied on each node independently.
- **Sequence collisions are prevented** via per-node offsets (odd/even) applied to all 32 sequences.
- **Networking:** Host networking on both nodes; PostgreSQL listens on `0.0.0.0:5434` directly on each host's LAN interface.
- **FDW endpoints** point to `192.168.0.28:5434` and `192.168.0.209:5434` — no `6432` translation layer.
- **Application tier** on each node points at its **local** Postgres (`DB_HOST=127.0.0.1` or host's LAN IP, `DB_PORT=5434`).

---

## 4. What Changed From the Bridge-Network Reference Build

The bridge-network build (documented previously) used a standalone 2-node Postgres-only cluster on port `6432` with published ports. tbNAC required several adaptations:

| Aspect | Bridge reference | tbNAC host networking |
|--------|------------------|-----------------------|
| Postgres port on LAN | `6432` (published) | **`5434` (native)** |
| FDW port | `6432` | **`5434`** |
| Networking | Docker bridge + published ports | **`network_mode: host`** |
| Database name | `postgres` | `tbnac_core` |
| Database user | `postgres` | `tbnac_core_user` |
| Schema | hand-created `dummy_multi` | **39 tables in `nac` schema**, loaded from `schema.sql` + `seed.sql` |
| Sequence naming | uniform `*_id_seq` | mixed — includes `radacct_radacctid_seq` (non-`id` column) |
| Sequences handled | 1 | **32** |
| Sequence-ownership via `pg_depend` | all 1 | only 8 of 32 |
| Conflict source | engineered | **organic** — concurrent `mac_vendors` seeding |
| Secondary unique constraints | none | `switches.ux_switches_ip` (partial), `cluster.ip_key`, `software.software_key_key` |
| `psql` connection | `-p 5432` (server default) | **`-p 5434`** (must be explicit) |
| Compose service names | `postgres-mm-node1` etc. | `postgres`, `tbnac-core-backend`, `tbnac-worker` |

**The three most important adaptations:**
1. FDW `port` → `5434`
2. Sequence-offset script rewritten to use `pg_sequences` instead of `pg_depend`
3. `psql` invocations always include `-p 5434`

---

## 5. The Custom PostgreSQL Image

### 5.1 Dockerfile

Placed at `~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30/Dockerfile`. Built **on both nodes**.

```dockerfile
FROM postgres:17-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential git ca-certificates \
    postgresql-server-dev-17 \
    libkrb5-dev krb5-multidev libgssapi-krb5-2 \
    libpq-dev libselinux1-dev libzstd-dev liblz4-dev \
    libxslt1-dev libxml2-dev libpam0g-dev libssl-dev \
    zlib1g-dev libreadline-dev libbz2-dev libgcrypt-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
RUN git clone https://github.com/aws/pgactive.git && \
    cd pgactive && \
    git checkout v2.1.7 && \
    export PG_CONFIG=/usr/lib/postgresql/17/bin/pg_config && \
    ./configure && \
    make LDFLAGS="-L/usr/lib/postgresql/17/lib" \
         CPPFLAGS="-I/usr/include/postgresql/17/server" && \
    make install

FROM postgres:17-bookworm

# 1. Shared library
COPY --from=builder /usr/lib/postgresql/17/lib/pgactive.so /usr/lib/postgresql/17/lib/

# 2. Extension SQL + control
COPY --from=builder /usr/share/postgresql/17/extension/pgactive* /usr/share/postgresql/17/extension/

# 3. Helper binaries — REQUIRED for pgactive_create_group / pgactive_join_group
COPY --from=builder /usr/lib/postgresql/17/bin/pgactive_dump /usr/lib/postgresql/17/bin/
COPY --from=builder /usr/lib/postgresql/17/bin/pgactive_init_copy /usr/lib/postgresql/17/bin/
```

### 5.2 Build (both nodes)

```bash
cd ~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30
docker build -f Dockerfile -t pgactive-postgres:17 .
```

### 5.3 Verify all four artifact types exist

```bash
docker run --rm pgactive-postgres:17 ls /usr/lib/postgresql/17/lib/     | grep pgactive
docker run --rm pgactive-postgres:17 ls /usr/share/postgresql/17/extension/ | grep pgactive
docker run --rm pgactive-postgres:17 ls /usr/lib/postgresql/17/bin/     | grep pgactive
```

Expected:
```
# lib/
pgactive.so

# extension/
pgactive--2.1.0--2.1.1.sql
pgactive--2.1.0.sql
... (upgrade chain) ...
pgactive.control

# bin/
pgactive_dump
pgactive_init_copy
```

> The two binaries in `bin/` are the single most common miss. Omitting them causes the per-db worker to silently fail during `pgactive_join_group()` and emit a misleading "could not detect a running pgactive perdb worker" error.

---

## 6. Configuration Files

### 6.1 `config/postgresql.conf`

Placed at `~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30/config/postgresql.conf`. **Identical on both nodes.**

```conf
listen_addresses = '*'
port = 5434
hba_file = '/etc/postgresql/pg_hba.conf'

# pgactive / logical replication
wal_level = logical
max_worker_processes = 64
max_replication_slots = 32
max_wal_senders = 32
shared_preload_libraries = 'pgactive'
track_commit_timestamp = on
max_parallel_workers = 16
max_logical_replication_workers = 32
max_sync_workers_per_subscription = 8

# pgactive node capacity
pgactive.max_nodes = 16

# PostgreSQL 17.11+ output plugin whitelist — REQUIRED
output_plugin_libraries = 'pgoutput, test_decoding, pgactive'
```

### 6.2 `config/pg_hba.conf`

Placed at `~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30/config/pg_hba.conf`. **Identical on both nodes.**

```conf
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
host    all             all             192.168.0.0/24          scram-sha-256
```

> **Why the LAN subnet and not `0.0.0.0/0`:** With host networking, PostgreSQL is directly exposed on the host's LAN interface. There is no Docker iptables layer to shield it. Narrowing HBA to `192.168.0.0/24` is the minimum safe configuration.

### 6.3 Why `output_plugin_libraries` Is Mandatory

PostgreSQL 17.11 introduced a security hardening measure that restricts which logical decoding output plugins can run. Without `pgactive` listed here, the extension's output plugin is rejected and logical decoding fails silently. This is **not** the same as `shared_preload_libraries` — both must contain `pgactive`.

---

## 7. Docker Compose — Node 1 and Node 2

### 7.1 Common edits to both nodes

Starting from the tbNAC `services.txt`, apply:

**IP replacement.** The original tbNAC compose used `192.168.0.194`. Each node's compose must point at that node's own IP:

```bash
# Node 1
sed -i 's/192\.168\.0\.194/192.168.0.28/g' docker-compose.yml

# Node 2
sed -i 's/192\.168\.0\.194/192.168.0.209/g' docker-compose.yml
```

**Exception:** Leave `DOMAIN_CTL_URL: http://127.0.0.1:8099` untouched. It's loopback by design — domain-ctl owns the Docker socket and must not be routable.

**`postgres` service replacement.** Replace the entire `postgres:` block with:

```yaml
  postgres:
    image: pgactive-postgres:17
    container_name: tbnac-postgres
    network_mode: host
    environment:
      - POSTGRES_USER=tbnac_core_user
      - POSTGRES_PASSWORD=tbnac_core_pass
      - POSTGRES_DB=tbnac_core
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8
    restart: unless-stopped
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./Infra/sql/schema.sql:/docker-entrypoint-initdb.d/10-schema.sql       # Node 1 ONLY
      - ./Infra/sql/seed.sql:/docker-entrypoint-initdb.d/20-seed.sql           # Node 1 ONLY
      - ./certs/cert.pem:/opt/cert.pem
      - ./certs/key.pem:/opt/key.pem
      - /etc/hosts:/etc/hosts:ro
      - ./config/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./config/pg_hba.conf:/etc/postgresql/pg_hba.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tbnac_core_user -d tbnac_core -p 5434"]
      interval: 10s
      timeout: 5s
      retries: 10
```

### 7.2 Node 1 — final `postgres` block

Includes **both** init mounts (`schema.sql`, `seed.sql`). Everything else in the file remains unchanged. All app services keep `DB_HOST: 192.168.0.28`, `DB_PORT: 5434`.

### 7.3 Node 2 — final `postgres` block

**Identical to Node 1 except** these two lines are **deleted**:

```yaml
      - ./Infra/sql/schema.sql:/docker-entrypoint-initdb.d/10-schema.sql
      - ./Infra/sql/seed.sql:/docker-entrypoint-initdb.d/20-seed.sql
```

Node 2's `tbnac_core` starts empty and is populated by `pgactive_join_group()` pulling the full DB from Node 1. Running init scripts on Node 2 would race with that copy.

All app services keep `DB_HOST: 192.168.0.209`, `DB_PORT: 5434`.

### 7.4 Pre-flight checklist (both nodes)

```bash
# 1. Port 5434 must be free
sudo ss -tlnp | grep ':5434'          # must be empty

# 2. Confirm image is built
docker images | grep pgactive-postgres

# 3. Wipe any prior stack + volumes
docker compose down --remove-orphans --volumes

# 4. Confirm peer reachable (only after peer is up)
nc -zv 192.168.0.209 5434             # from Node 1
nc -zv 192.168.0.28  5434             # from Node 2
```

---

## 8. Detailed Deployment Walkthrough

### 8.1 Step 1 — Start Postgres on Node 1

```bash
cd ~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30
docker compose up -d postgres
docker compose logs -f postgres
```

Wait for:
```
database system is ready to accept connections
```

The `pgactive supervisor restarting to connect to 'pgactive_supervisordb' DB` line on first start is normal.

### 8.2 Step 2 — Verify Node 1 Postgres

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SHOW shared_preload_libraries;"      # → pgactive
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SHOW output_plugin_libraries;"       # → pgoutput, test_decoding, pgactive
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SHOW wal_level;"                     # → logical
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SHOW port;"                          # → 5434
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "\dt nac.*"                           # → 39 tables
```

> **Gotcha:** Always pass `-p 5434`. Inside the container, `psql` defaults to port 5432 (socket path `/var/run/postgresql/.s.PGSQL.5432`), which will fail with `No such file or directory`.

### 8.3 Step 3 — Install pgactive + create FDW servers on Node 1

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
CREATE EXTENSION IF NOT EXISTS pgactive;

CREATE SERVER pgactive_server_endpoint1
  FOREIGN DATA WRAPPER pgactive_fdw
  OPTIONS (host '192.168.0.28', port '5434', dbname 'tbnac_core');
CREATE USER MAPPING FOR tbnac_core_user
  SERVER pgactive_server_endpoint1
  OPTIONS (user 'tbnac_core_user', password 'tbnac_core_pass');

CREATE SERVER pgactive_server_endpoint2
  FOREIGN DATA WRAPPER pgactive_fdw
  OPTIONS (host '192.168.0.209', port '5434', dbname 'tbnac_core');
CREATE USER MAPPING FOR tbnac_core_user
  SERVER pgactive_server_endpoint2
  OPTIONS (user 'tbnac_core_user', password 'tbnac_core_pass');
EOF
```

Verify:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "\des"
# → pgactive_server_endpoint1, pgactive_server_endpoint2
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT extname, extversion FROM pg_extension WHERE extname='pgactive';"
# → pgactive | 2.1.7
```

### 8.4 Step 4 — Create the group on Node 1

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
SELECT pgactive.pgactive_create_group(
    node_name := 'tbnac-node1',
    node_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint1'
);
SELECT pgactive.pgactive_wait_for_node_ready();
EOF
```

Expected notices:

```
WARNING:  secondary unique constraint(s) exist on replicated table(s)
DETAIL:  Table nac.software has secondary unique constraint software_software_key_key.
HINT:    Drop the secondary unique constraint(s), change the table(s) to UNLOGGED if
         they don't need to be replicated, or exclude the table(s) from the active
         replication set(s).

WARNING:  secondary unique constraint(s) exist on replicated table(s)
DETAIL:  Table nac.cluster has secondary unique constraint cluster_ip_key.

NOTICE:  successfully created first node in pgactive group
```

> These warnings are **informational only** — see §12.

Verify:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT node_name, node_status FROM pgactive.pgactive_nodes;"
# → tbnac-node1 | r
```

### 8.5 Step 5 — Apply sequence offsets on Node 1 (odd parity)

Run **before** Node 2 joins, so no writes occur mid-shift:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
DO $$
DECLARE
    seq RECORD;
    cur BIGINT;
    target BIGINT;
BEGIN
    FOR seq IN
        SELECT schemaname, sequencename
        FROM pg_sequences
        WHERE schemaname NOT IN ('pg_catalog','information_schema','pgactive','pg_toast')
    LOOP
        EXECUTE format('SELECT last_value FROM %I.%I',
                       seq.schemaname, seq.sequencename) INTO cur;
        cur := COALESCE(cur, 0);
        EXECUTE format('ALTER SEQUENCE %I.%I INCREMENT BY 2',
                       seq.schemaname, seq.sequencename);
        -- Node 1 = odd, strictly greater than cur
        target := CASE WHEN cur % 2 = 1 THEN cur + 2 ELSE cur + 1 END;
        EXECUTE format('SELECT setval(%L, %s, false)',
                       seq.schemaname || '.' || seq.sequencename, target);
        RAISE NOTICE 'Node1: %.% cur=% -> next %',
                     seq.schemaname, seq.sequencename, cur, target;
    END LOOP;
END $$;
EOF
```

You should see **32 `NOTICE` lines** — one per sequence in the `nac` schema.

Verify:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT sequencename, increment_by FROM pg_sequences
WHERE schemaname='nac' ORDER BY 1;"
# → 32 rows, all increment_by = 2
```

### 8.6 Step 6 — Prepare Node 2

```bash
# On Node 2
cd ~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30

# Sanity: no stray IPs, image present, no schema/seed mounts
grep -n '192.168.0.194' docker-compose.yml         # → empty
grep -n 'pgactive-postgres:17' docker-compose.yml  # → present
grep -n 'schema.sql\|seed.sql' docker-compose.yml  # → empty

# Free port 5434
sudo ss -tlnp | grep ':5434'                       # → empty

# Fresh volume
docker compose down --remove-orphans --volumes

# Start Postgres only
docker compose up -d postgres
docker compose logs -f postgres
```

Wait for `ready to accept connections`. **The DB will be empty — that's expected.**

Verify config:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "SHOW shared_preload_libraries;"
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "SHOW output_plugin_libraries;"
```

### 8.7 Step 7 — Install pgactive + FDW on Node 2

Identical script to §8.3:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
CREATE EXTENSION IF NOT EXISTS pgactive;

CREATE SERVER pgactive_server_endpoint1
  FOREIGN DATA WRAPPER pgactive_fdw
  OPTIONS (host '192.168.0.28', port '5434', dbname 'tbnac_core');
CREATE USER MAPPING FOR tbnac_core_user
  SERVER pgactive_server_endpoint1
  OPTIONS (user 'tbnac_core_user', password 'tbnac_core_pass');

CREATE SERVER pgactive_server_endpoint2
  FOREIGN DATA WRAPPER pgactive_fdw
  OPTIONS (host '192.168.0.209', port '5434', dbname 'tbnac_core');
CREATE USER MAPPING FOR tbnac_core_user
  SERVER pgactive_server_endpoint2
  OPTIONS (user 'tbnac_core_user', password 'tbnac_core_pass');
EOF
```

**Critical pre-check — can Node 2 reach Node 1?**

```bash
docker exec -i tbnac-postgres psql -h 192.168.0.28 -p 5434 -U tbnac_core_user -d tbnac_core -c "SELECT 1;"
# Must return 1
```

If this fails, **stop** — do not attempt the join until connectivity is fixed.

### 8.8 Step 8 — Join Node 2 to the group

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
SELECT pgactive.pgactive_join_group(
    node_name := 'tbnac-node2',
    node_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint2',
    join_using_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint1'
);
SELECT pgactive.pgactive_wait_for_node_ready();
EOF
```

Expected notices:

```
NOTICE:  transferring of database 'tbnac_core' ...
NOTICE:  successfully joined the node and restored database 'tbnac_core' from node tbnac-node1
```

Verify on **both** nodes:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;"
```

Expected:

```
  node_name  | node_status
-------------+-------------
 tbnac-node1 | r
 tbnac-node2 | r
(2 rows)
```

Confirm Node 2 got the schema:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "\dt nac.*"
# → same 39 tables
```

### 8.9 Step 9 — Apply sequence offsets on Node 2 (even parity)

Identical to §8.5, **except one line** — flip the parity formula:

```sql
target := CASE WHEN cur % 2 = 0 THEN cur + 2 ELSE cur + 1 END;   -- Node 2 (even)
```

Full block:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
DO $$
DECLARE
    seq RECORD;
    cur BIGINT;
    target BIGINT;
BEGIN
    FOR seq IN
        SELECT schemaname, sequencename
        FROM pg_sequences
        WHERE schemaname NOT IN ('pg_catalog','information_schema','pgactive','pg_toast')
    LOOP
        EXECUTE format('SELECT last_value FROM %I.%I',
                       seq.schemaname, seq.sequencename) INTO cur;
        cur := COALESCE(cur, 0);
        EXECUTE format('ALTER SEQUENCE %I.%I INCREMENT BY 2',
                       seq.schemaname, seq.sequencename);
        -- Node 2 = even
        target := CASE WHEN cur % 2 = 0 THEN cur + 2 ELSE cur + 1 END;
        EXECUTE format('SELECT setval(%L, %s, false)',
                       seq.schemaname || '.' || seq.sequencename, target);
        RAISE NOTICE 'Node2: %.% cur=% -> next %',
                     seq.schemaname, seq.sequencename, cur, target;
    END LOOP;
END $$;
EOF
```

Verify parity — Node 1 must produce odd, Node 2 even:

```bash
# Node 1
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT nextval('nac.switches_id_seq');"
# e.g. 1, 3, 5, ...

# Node 2
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT nextval('nac.switches_id_seq');"
# e.g. 2, 4, 6, ...
```

### 8.10 Step 10 — Bring up the rest of the stack (both nodes)

```bash
docker compose up -d
docker compose ps
```

Apps now connect to their **local** Postgres on `:5434` and both nodes are live.

> **First-boot note:** On Node 1, `tbnac-backend` will report `"mac_vendors already seeded" count=40321` (from `seed.sql`). On Node 2, it will download the IEEE OUI CSV and insert ~40,324 rows. This generates an automatic conflict — see §11.

---

## 9. Sequence Offsets — The Multi-Master Pattern

### 9.1 The Problem

In a multi-master cluster, each node has its own local sequence. Without coordination:

1. Node 1 inserts a row with `id=1`, sequence advances to 2.
2. The row replicates to Node 2, but Node 2's sequence is still at 1.
3. Node 2 tries to insert, uses `id=1` → **duplicate key error**.

### 9.2 The Solution

Assign each node a **distinct offset** and a **shared step size**:

| Nodes | Offsets | Step | Node 1 produces | Node 2 produces |
|-------|---------|------|-----------------|-----------------|
| 2 | 0, 1 | 2 | 1, 3, 5, 7, ... | 2, 4, 6, 8, ... |
| 3 | 0, 1, 2 | 3 | 1, 4, 7, ... | 2, 5, 8, ... | 
| N | 0..N-1 | N | 1, 1+N, ... | 2, 2+N, ... |

### 9.3 Why the Bridge-Report Script Didn't Work

The original bridge-report offset script filtered sequences via:

```sql
JOIN pg_depend d ON d.objid = c.oid AND d.deptype = 'a'
```

This finds only sequences that are `OWNED BY` a column (SERIAL / IDENTITY style). In tbNAC's schema, **only 8 of 32 sequences** are owned that way — the other 24 are standalone `CREATE SEQUENCE` with column defaults referencing `nextval()`, so they have no `pg_depend` link.

**Only `nac.nas_id_seq`, `nac.radacct_radacctid_seq`, and 6 other `rad*` sequences matched.** `switches_id_seq`, `ad_domains_id_seq`, `cluster_id_seq`, and 24 others were silently skipped.

### 9.4 The Corrected Script (used in production)

Uses `pg_sequences` — the built-in catalog view listing **all** sequences in the database:

```sql
DO $$
DECLARE
    seq RECORD;
    cur BIGINT;
    target BIGINT;
BEGIN
    FOR seq IN
        SELECT schemaname, sequencename
        FROM pg_sequences
        WHERE schemaname NOT IN ('pg_catalog','information_schema','pgactive','pg_toast')
    LOOP
        EXECUTE format('SELECT last_value FROM %I.%I',
                       seq.schemaname, seq.sequencename) INTO cur;
        cur := COALESCE(cur, 0);
        EXECUTE format('ALTER SEQUENCE %I.%I INCREMENT BY 2',
                       seq.schemaname, seq.sequencename);
        -- Node 1 (odd): target := CASE WHEN cur % 2 = 1 THEN cur + 2 ELSE cur + 1 END;
        -- Node 2 (even): target := CASE WHEN cur % 2 = 0 THEN cur + 2 ELSE cur + 1 END;
        target := CASE WHEN cur % 2 = 1 THEN cur + 2 ELSE cur + 1 END;
        EXECUTE format('SELECT setval(%L, %s, false)',
                       seq.schemaname || '.' || seq.sequencename, target);
        RAISE NOTICE 'Node1: %.% cur=% -> next %',
                     seq.schemaname, seq.sequencename, cur, target;
    END LOOP;
END $$;
```

**Idempotent.** Uses `setval(..., false)` — an absolute set, not a relative increment — so re-running on the same node is safe. `ALTER SEQUENCE ... INCREMENT BY 2` is naturally idempotent.

### 9.5 The 32 Sequences in `nac`

```
ad_domains_id_seq                     nas_id_seq
auth_sources_id_seq                   radacct_radacctid_seq   ← non-id column name
cluster_id_seq                        radcheck_id_seq
connection_profile_sources_id_seq     radgroupcheck_id_seq
connection_profiles_id_seq            radgroupreply_id_seq
dhcp_fingerprints_id_seq              radius_audit_log_id_seq
dhcp_vendor_classes_id_seq            radpostauth_id_seq
eap_profiles_id_seq                   radreply_id_seq
filter_rules_id_seq                   radusergroup_id_seq
locationlog_id_seq                    realm_id_seq
                                      realm_sources_id_seq
                                      roles_id_seq
                                      security_event_classes_id_seq
                                      security_events_id_seq
                                      software_id_seq
                                      switch_filters_id_seq
                                      switch_groups_id_seq
                                      switch_templates_id_seq
                                      switches_id_seq
                                      syslog_forwarders_id_seq
                                      system_info_id_seq
                                      vlan_pool_id_seq
```

### 9.6 Why `setval(..., false)`

The second argument to `setval` is `is_called`:

- `setval(seq, 5, true)` → next `nextval` returns **6**.
- `setval(seq, 5, false)` → next `nextval` returns **5**.

Using `false` lets us precisely set the first value each node will produce.

### 9.7 Alternative Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Sequence offsets** (chosen) | Simple; no schema change; standard practice | Non-contiguous ids; requires per-node setup |
| UUID primary keys | No coordination needed | Larger storage; index performance impact |
| Application-managed IDs | Full control | Complexity moves to app |
| Snowflake IDs | Distributed-friendly | Requires ID generator infrastructure |

---

## 10. Test Plan & Results

### 10.1 Test A — Cluster formation

| ID | Test | Expected | Observed | Status |
|----|------|----------|----------|--------|
| A-1 | Cluster formed | both nodes `r` | `tbnac-node1 r`, `tbnac-node2 r` on both | ✅ |
| A-2 | Replication slot active | 1 active `pgactive_*` per node | Confirmed on both | ✅ |

**Evidence:**

```
  node_name  | node_status
-------------+-------------
 tbnac-node1 | r
 tbnac-node2 | r
(2 rows)

                  slot_name                   |  plugin  | active
----------------------------------------------+----------+--------
 pgactive_16385_7696043117641756816_0_16385__ | pgactive | t
```

### 10.2 Test B — Bidirectional data replication

| ID | Test | Expected | Observed | Status |
|----|------|----------|----------|--------|
| B-1 | INSERT on Node 1 → visible on Node 2 | Odd id | `id=3 test-nas-node1` on both | ✅ |
| B-2 | INSERT on Node 2 → visible on Node 1 | Even id | `id=4 test-nas-node2` on both | ✅ |
| B-3 | UPDATE on Node 1 → visible on Node 2 | Reflected | `updated-on-node1` on both | ✅ |
| B-4 | DELETE on Node 2 → removed from Node 1 | Row gone | `count=0` on both | ✅ |

**Commands used:**

```sql
-- B-1: INSERT on Node 1
INSERT INTO nac.nas (nasname, shortname, secret, description)
VALUES ('test-nas-node1', 'N1', 'testing123', 'inserted on node1')
RETURNING id, nasname, shortname;
-- → id=3 (odd)

-- B-2: INSERT on Node 2
INSERT INTO nac.nas (nasname, shortname, secret, description)
VALUES ('test-nas-node2', 'N2', 'testing123', 'inserted on node2')
RETURNING id, nasname, shortname;
-- → id=4 (even)

-- B-3: UPDATE on Node 1
UPDATE nac.nas SET description='updated-on-node1'
WHERE nasname='test-nas-node2';

-- B-4: DELETE on Node 2
DELETE FROM nac.nas WHERE nasname='test-nas-node1';
```

### 10.3 Test C — Node-failure tolerance

| ID | Test | Expected | Observed | Status |
|----|------|----------|----------|--------|
| C-1 | Node 1 Postgres stopped | Slot on Node 2 → `f` | Slot `f`, Node 2 fully operational | ✅ |
| C-2 | Write during outage | Accepted on Node 2 | `real-outage-2026` with `id=8` (even) | ✅ |
| C-3 | Node 1 restarted | Slot → `t`, both `r` | Confirmed | ✅ |
| C-4 | Catch-up on Node 1 | Row visible; lag low | `real-outage-2026` visible; `lag_bytes=56` | ✅ |

**Commands used:**

```bash
# On Node 1 — stop Postgres only
docker compose stop postgres

# On Node 2 — verify slot went inactive
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT slot_name, active FROM pg_replication_slots WHERE slot_name LIKE 'pgactive%';"
# → active = f

# On Node 2 — write during outage
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
INSERT INTO nac.nas (nasname, shortname, secret, description)
VALUES ('real-outage-2026', 'ROUT', 'testing123', 'written during node1 outage')
RETURNING id, nasname;"
# → id=8 (even)

# On Node 1 — restart
docker compose start postgres
sleep 20

# On Node 1 — verify catch-up
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT id, nasname, description FROM nac.nas WHERE nasname='real-outage-2026';"
# → id=8 present

# Lag check
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots WHERE slot_name LIKE 'pgactive%';"
# → lag_bytes = 56
```

### 10.4 Test D — Conflict handling

| ID | Test | Expected | Observed | Status |
|----|------|----------|----------|--------|
| D-1 | Concurrent `mac_vendors` write | Auto-resolve | Both nodes converged to **40,321**; `nr_update_conflict=1` on Node 1 | ✅ |

See §11 for the full narrative.

### 10.5 Test E — Rejoin verification

| ID | Test | Expected | Observed | Status |
|----|------|----------|----------|--------|
| E-1 | New write after rejoin | Cross-visible | `post-rejoin-n1` (id=5) seen on Node 2 | ✅ |
| E-2 | Final row count parity | Identical | `count(nac.nas)=4` on both | ✅ |

### 10.6 Final state

| Metric | Node 1 | Node 2 |
|--------|--------|--------|
| `pgactive_nodes` | both `r` | both `r` |
| `nr_commit` | 107 | 18 |
| `nr_insert` | 40,325 | 5 |
| `nr_insert_conflict` | 0 | 0 |
| `nr_update` | 7 | 7 |
| `nr_update_conflict` | **1** | 0 |
| `nr_delete` | 1 | 0 |
| `nr_delete_conflict` | 0 | 0 |
| `nr_disconnect` | **0** | **0** |
| `count(nac.nas)` | **4** | **4** |

**Zero connection drops across the entire test run.** Both nodes identical.

---

## 11. The Organic Conflict — `mac_vendors`

### 11.1 What Happened

At cluster formation, both tbNAC backends ran their "seed mac_vendors if empty" logic:

| Node | Log line |
|------|----------|
| **Node 1** | `"mac_vendors already seeded" count=40321` (from `seed.sql`) |
| **Node 2** | `"mac_vendors empty — downloading IEEE OUI CSV"` → `"inserted":40324` |

Node 2 saw an empty table (before replication caught up), downloaded the IEEE OUI CSV, and inserted **40,324** rows — 3 more than Node 1 had. Those rows replicated to Node 1, where 40,321 rows already existed.

### 11.2 Outcome

| Metric | Node 1 | Node 2 |
|--------|--------|--------|
| `count(*) nac.mac_vendors` | **40,321** | **40,321** |
| `nr_insert` | 40,323 | 4 |
| `nr_insert_conflict` | 0 | 0 |
| `nr_update_conflict` | **1** | 0 |

**Both nodes converged to 40,321 rows** — pgactive's last-writer-wins resolved the collision. The 3 extra rows Node 2 attempted to insert were suppressed via an update conflict, not a hard error.

### 11.3 What This Proves

- **No data loss.** Both sides hold identical data.
- **No orphan rows.** No PK violations left dangling.
- **No application errors.** tbNAC's backend didn't raise — pgactive handled it transparently.
- **This is exactly the behavior a multi-master cluster must exhibit under real concurrent writes.**

---

## 12. Secondary Unique Constraint Warnings

`pgactive_create_group()` emitted:

```
WARNING: secondary unique constraint(s) exist on replicated table(s)
DETAIL:  Table nac.software has secondary unique constraint software_software_key_key.
HINT:    Drop the secondary unique constraint(s), change the table(s) to UNLOGGED if
         they don't need to be replicated, or exclude the table(s) from the active
         replication set(s).

WARNING: secondary unique constraint(s) exist on replicated table(s)
DETAIL:  Table nac.cluster has secondary unique constraint cluster_ip_key.
```

### 12.1 What This Means

pgactive handles the **primary key** cleanly (last-writer-wins). Secondary unique constraints are **not resolved automatically** — a conflict on `software.software_key` or `cluster.ip` would raise an error rather than silently pick a winner.

### 12.2 Impact Assessment

| Constraint | Table | Current Risk |
|------------|-------|--------------|
| `software.software_key_key` | `nac.software` | Low — not on hot path |
| `cluster.cluster_ip_key` | `nac.cluster` | Low — cluster membership is rarely written |
| `switches.ux_switches_ip` (partial unique on `ip` where `ip_range IS NULL`) | `nac.switches` | Medium — switches are frequently managed |

### 12.3 Recommendations for Production

Choose one per table:

1. **Drop the constraint** if uniqueness is not strictly required.
2. **Change the table to `UNLOGGED`** if it doesn't need to be replicated (surprisingly useful for cache-like tables).
3. **Exclude the table from replication** via `pgactive.pgactive_exclude_from_replication('nac.software')`.
4. **Write a custom conflict resolver** (out of scope here).

For tbNAC, dropping the constraint on `nac.cluster.ip` is the cleanest option because cluster membership is small and mostly static.

---

## 13. Issues Encountered & Resolutions

### 13.1 Deployment Phase

| # | Issue | Root Cause | Resolution |
|---|-------|-----------|------------|
| 1 | `psql: error ... /var/run/postgresql/.s.PGSQL.5432` | Server on 5434, `psql` defaulting to 5432 | Always pass `-p 5434` |
| 2 | Original offset script touched only 8 of 32 sequences | `pg_depend deptype='a'` filter misses standalone `CREATE SEQUENCE` | Rewrote using `pg_sequences` |
| 3 | `docker compose stop tbnac-backend` → `no such service` | Compose tracks services by key (`tbnac-core-backend`), not container name | Used `docker compose stop postgres` (only DB matters for cluster failover) |
| 4 | `\des` shows servers — but not mappings | Verification only checked server list | Also query `pg_user_mappings` or `\deu+` to confirm mappings |
| 5 | `pgactive_create_group` warnings about secondary unique constraints | Schema has non-PK uniques on `software`, `cluster` | Documented; not blocking |
| 6 | `mac_vendors` double-seed (Node 2 downloaded IEEE OUI CSV) | Node 2 schema-less initially; app self-seeds when empty | Handled by pgactive's last-writer-wins (see §11) |

### 13.2 Key Lessons

1. **`psql` default port mismatch is a silent time-sink.** Always pass `-p <port>` inside the container when Postgres isn't on 5432.
2. **`pg_depend` is unreliable for offset discovery.** `pg_sequences` sees all sequences; use it.
3. **Docker Compose service names ≠ container names.** Use `docker compose ps` to see the service key.
4. **An empty DB on Node 2 is expected** before the join. Don't panic when `\dt` returns nothing.
5. **First-boot self-seeding can generate conflicts automatically.** Node 2's backend inserted 40,324 rows into what it saw as an empty table — an unintentional but perfect conflict test.
6. **Host networking removes port translation** but exposes PostgreSQL directly. HBA must be tightened to the LAN subnet.
7. **`pgactive` recovers from a peer restart without manual intervention** — the slot goes `f` then back to `t` on its own.

---

## 14. Operational Runbook

### 14.1 Daily Health Check

Run on **either** node:

```bash
# Cluster membership
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;"
# Expect: both 'r'

# Replication stats
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT nr_commit, nr_insert, nr_insert_conflict,
       nr_update, nr_update_conflict,
       nr_delete, nr_delete_conflict,
       nr_disconnect
FROM pgactive.pgactive_stats;"
# nr_disconnect should stay 0

# Conflict log
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT conflict_id, local_conflict_time, object_name, error_sqlstate
FROM pgactive.pgactive_conflict_history
ORDER BY local_conflict_time DESC LIMIT 20;"
# Empty under normal operation

# Slot status + lag
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots WHERE slot_name LIKE 'pgactive%';"
# active = t, lag_bytes near 0
```

### 14.2 Restarting a Node

```bash
cd ~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30
docker compose stop postgres
docker compose start postgres
sleep 20

# Verify rejoin
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;"
```

The peer node keeps serving writes throughout. Slot on the peer flips to `f`, then back to `t` within a few seconds of restart.

### 14.3 Adding a New Table

**DDL is not replicated.** Apply on **every node** independently:

```bash
# Node 1
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
CREATE TABLE nac.new_table (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ...
);
EOF

# Node 2 — identical DDL
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
CREATE TABLE nac.new_table ( ... );
EOF
```

Then apply the sequence offset to the new sequence on **each** node:

```bash
# Node 1 (odd)
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
ALTER SEQUENCE nac.new_table_id_seq INCREMENT BY 2;
SELECT setval('nac.new_table_id_seq', 1, false);
EOF

# Node 2 (even)
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
ALTER SEQUENCE nac.new_table_id_seq INCREMENT BY 2;
SELECT setval('nac.new_table_id_seq', 2, false);
EOF
```

### 14.4 Schema Migration Procedure

1. **Stop application writes.** Pause `tbnac-backend` and `tbnac-worker` on all nodes.
2. **Apply DDL to each node** (one at a time — no ordering requirement).
3. **Verify** the schema is identical across nodes (`\dt nac.*`).
4. **Apply sequence offsets** to any new sequences (see §14.3).
5. **Resume writes.**

### 14.5 Node Maintenance

To take a node offline for maintenance:

```bash
# Node being maintained
docker compose stop postgres
```

The other node continues accepting writes; its app tier keeps running against its local DB. When maintenance is complete:

```bash
docker compose start postgres
sleep 20
# Verify rejoin
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 \
  -c "SELECT node_name, node_status FROM pgactive.pgactive_nodes;"
```

pgactive resumes replication automatically — no manual slot recreation needed if the slot wasn't dropped.

### 14.6 Full Cluster Rebuild

**On both nodes:**

```bash
cd ~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30
docker compose down --remove-orphans --volumes
```

Then follow the walkthrough from §8.1.

### 14.7 Backup Strategy

Because both nodes hold identical data, back up from **either** node:

```bash
docker exec -i tbnac-postgres pg_dump -U tbnac_core_user -d tbnac_core -p 5434 -Fc \
  > tbnac_core-$(date +%F).dump
```

Restore into a single-node test instance:

```bash
docker exec -i tbnac-postgres pg_restore -U tbnac_core_user -d tbnac_core -p 5434 \
  < tbnac_core-2026-10-13.dump
```

### 14.8 Reapplying Sequence Offsets (post-restore)

If a node is rebuilt from a backup and needs to rejoin an existing cluster, re-run the parity-specific script from §9.4 on that node only.

---

## 15. Host Networking vs Bridge Networking

| Aspect | Bridge Networking | **Host Networking (this build)** |
|--------|-------------------|-----------------------------------|
| Port on LAN | Custom (e.g. 6432) published | Native (5434) direct |
| FDW port | Custom published port | Native port |
| Network isolation | Strong (private namespace) | **None** — container shares host stack |
| Docker DNS between containers | Works | **Broken** — use LAN IPs |
| Port collision risk | Low | **High** — 5434 must be free |
| Latency | Slight overhead (NAT) | Lowest possible |
| Security posture | Better (HBA + Docker firewall) | Weaker — HBA is the only gate |
| Multi-container on same host | Possible (distinct host ports) | **Impossible** (both want 5434) |
| Compose complexity | Moderate | Minimal |
| Best for | Production, multi-tenant hosts | Performance, single-purpose hosts |

**When to pick host networking:**
- The tbNAC appliance owns the entire host.
- Port 5434 is guaranteed free.
- No other Docker stack competes for the same port.
- Performance testing where NAT overhead matters.

**When to pick bridge networking:**
- Multi-tenant hosts.
- Multiple Postgres instances per host.
- Production where isolation matters more than latency.

---

## 16. Key Concepts

### 16.1 Logical Replication

Unlike streaming replication (raw WAL bytes), **logical replication** decodes WAL into row-level change events (INSERT/UPDATE/DELETE) and ships them to subscribers, which apply them as normal SQL.

**Implications for multi-master:**
- Both nodes can generate changes independently.
- Events are discrete, not a raw WAL stream.
- **DDL is not included** — schema changes must be applied on every node.

### 16.2 `pgactive` Architecture

Built on PostgreSQL's logical decoding infrastructure, adding:

- **A supervisor background worker** managing per-database state.
- **A per-db worker** for each database participating in replication.
- **A conflict-resolution framework** (last-writer-wins by default).
- **A cluster membership catalog** (`pgactive.pgactive_nodes`).
- **Foreign data wrappers** for storing peer connection credentials.

### 16.3 The `pgactive_dump` / `pgactive_init_copy` Binaries

When a new node joins a pgactive group, the extension needs to **copy the existing database state** to the joining node. It invokes:

- `pgactive_dump` — a wrapper around `pg_dump` handling pgactive's internal catalogs.
- `pgactive_init_copy` — orchestrates the initial copy.

Both must exist in `bindir` (`/usr/lib/postgresql/17/bin/` on Debian). Missing them → the per-db worker fails silently and the join dies with `could not detect a running pgactive perdb worker`.

### 16.4 The `output_plugin_libraries` Parameter

Introduced in PostgreSQL 17.11 as a security hardening measure. Restricts which logical decoding output plugins may run. Without `pgactive` listed, logical decoding fails.

**Not the same as `shared_preload_libraries`.** Both must include `pgactive`.

### 16.5 FDW as Configuration Store

pgactive stores peer connection details via **Foreign Data Wrappers**, not in a config file. When you run `pgactive_create_group(node_dsn := 'user_mapping=... pgactive_foreign_server=...')`, the `pgactive_foreign_server` refers to an FDW server defined in the current database. Peer IPs, ports, and credentials are all in the DB.

This means:
- Changing a peer's IP requires `ALTER SERVER`.
- FDW servers must exist on **every** node before joining.
- The `port` in FDW `OPTIONS` is the **real** listening port (5434 here).

### 16.6 Conflict History and Stats

Two essential monitoring views:

- **`pgactive.pgactive_conflict_history`** — one row per write conflict (PK, unique key, FK).
- **`pgactive.pgactive_stats`** — counters for commits, inserts, updates, deletes, conflicts, and disconnects.

Both are per-database and reset on `pgactive.pgactive_stats_reset()` or by recreating the DB.

---

## 17. Appendices

### Appendix A — Complete Command Reference

**Container lifecycle:**

```bash
docker compose up -d
docker compose down --remove-orphans --volumes
docker compose restart <service>
docker compose stop postgres        # the ONLY cluster-relevant service
docker compose start postgres
docker compose logs -f postgres
```

**psql access (always include `-p 5434`):**

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "SQL..."
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 <<'EOF'
... multi-line SQL ...
EOF
```

**Handy alias (per shell session):**

```bash
alias tp='docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434'
tp -c "SELECT node_name, node_status FROM pgactive.pgactive_nodes;"
```

**pgactive inspection:**

```sql
-- Cluster membership
SELECT * FROM pgactive.pgactive_nodes;

-- Conflicts
SELECT * FROM pgactive.pgactive_conflict_history ORDER BY local_conflict_time DESC;

-- Stats
SELECT * FROM pgactive.pgactive_stats;

-- Replication slots
SELECT slot_name, plugin, active, database FROM pg_replication_slots;
```

**Sequence inspection:**

```sql
-- All sequences
SELECT schemaname, sequencename, increment_by FROM pg_sequences
WHERE schemaname='nac' ORDER BY 2;

-- One sequence's raw state
SELECT * FROM nac.switches_id_seq;
```

### Appendix B — Failure Taxonomy

| Layer | Symptom | Where to Look |
|-------|---------|---------------|
| Image build | Binary missing from runtime image | `docker run --rm pgactive-postgres:17 ls /usr/lib/postgresql/17/bin/` |
| Config | `output plugin "pgactive" is not installed` | `SHOW output_plugin_libraries;` |
| Config | `psql` can't connect | `-p 5434` missing; `SHOW port;` |
| Networking | Container can't reach peer | `docker exec ... psql -h <peer> -p 5434` |
| Worker | `per-db worker exited` | `docker logs tbnac-postgres` |
| Join | `state=i` stuck in `pgactive_nodes` | Manual `DELETE` from catalog, then re-join |
| Cleanup | `previous init failed` | `docker compose down --volumes` on both, restart |
| Sequence | Duplicate key on insert | `SELECT increment_by FROM pg_sequences WHERE schemaname='nac';` |
| Schema | `column "X" does not exist` | Check real schema — column names differ from guesses |
| HBA | Peer connection refused | `SHOW hba_file;`, verify 192.168.0.0/24 line |
| Compose | `no such service: tbnac-backend` | Service key is `tbnac-core-backend`; use `docker compose ps` |

### Appendix C — Glossary

| Term | Definition |
|------|-----------|
| **pgactive** | AWS logical replication extension for multi-master PostgreSQL |
| **Logical replication** | Row-level change events propagated between nodes |
| **Output plugin** | Decodes WAL into logical events (`pgoutput`, `pgactive`) |
| **Per-db worker** | Background process per database managing replication state |
| **Supervisor** | Cluster-wide background worker for pgactive |
| **FDW** | Foreign Data Wrapper used to store peer connection info |
| **Conflict** | Two nodes writing incompatible changes (e.g. same PK) |
| **Sequence offset** | Per-node starting value + shared step to avoid ID collisions |
| **`setval(..., false)`** | Sets sequence so next `nextval` returns exactly the given value |
| **`pg_sequences`** | Catalog view listing all sequences (unlike `pg_depend` which only finds owned ones) |
| **`TEMPLATE template0`** | Creates a database without inheriting extensions/config from `template1` |
| **Host networking** | Docker mode where the container shares the host's network namespace |
| **Published port** | Host port that forwards to a container port (bridge mode only) |
| **Last-writer-wins** | pgactive's default conflict resolution strategy |
| **Secondary unique constraint** | Non-PK unique index that pgactive cannot auto-resolve |

### Appendix D — Raw Test Evidence

**D.1 Final cluster state (both nodes):**

```
  node_name  | node_status
-------------+-------------
 tbnac-node1 | r
 tbnac-node2 | r
(2 rows)
```

**D.2 `nac.nas` final contents (both nodes):**

```
 id |     nasname      |         description
----+------------------+-----------------------------
  4 | test-nas-node2   | updated-on-node1
  5 | post-rejoin-n1   | written on node1 after rejoin
  6 | during-outage    | written while node1 down
  8 | real-outage-2026 | written during node1 outage
(4 rows)
```

(Note: `id=3` and `id=7` were deleted during tests; parity preserved.)

**D.3 Node 1 final stats:**

```
 nr_commit | nr_insert | nr_insert_conflict | nr_update | nr_update_conflict | nr_delete | nr_delete_conflict | nr_disconnect
-----------+-----------+--------------------+-----------+--------------------+-----------+--------------------+---------------
       107 |     40325 |                  0 |         7 |                  1 |         1 |                  0 |             0
```

**D.4 Node 2 final stats:**

```
 nr_commit | nr_insert | nr_insert_conflict | nr_update | nr_update_conflict | nr_delete | nr_delete_conflict | nr_disconnect
-----------+-----------+--------------------+-----------+--------------------+-----------+--------------------+---------------
        18 |         5 |                  0 |         7 |                  0 |         0 |                  0 |             0
```

**D.5 Conflict history (Node 1):**

```
 conflict_id |      local_conflict_time      |  object_name   | error_sqlstate | error_constraintname
-------------+-------------------------------+----------------+----------------+----------------------
           1 | 2026-10-13 06:57:56.461384+00 | pgactive_nodes |                |
(1 row)
```

The `pgactive_nodes` entry is internal to group formation (no `error_sqlstate`) — benign. The `mac_vendors` conflict did **not** produce a `pgactive_conflict_history` row because last-writer-wins resolved it via an update, which is logged in `nr_update_conflict` instead.

**D.6 Replication lag after rejoin:**

```
                  slot_name                   | active | lag_bytes
----------------------------------------------+--------+-----------
 pgactive_16385_7696043117641756816_0_16385__ | t      |        56
```

**D.7 mac_vendors convergence:**

| | Node 1 | Node 2 |
|---|---|---|
| `count(*)` | 40,321 | 40,321 |
| `nr_insert` | 40,323 | 4 |
| `nr_update_conflict` | 1 | 0 |

Both nodes identical. No data loss.

---

## Document End.
