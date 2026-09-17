# PostgreSQL High Availability Cluster — Implementation Report

**Project:** Containerized PostgreSQL HA Cluster (Master-Standby)
**Environment:** Two-node setup on Ubuntu hosts with Docker
**Nodes:**
- **Node 1** — hostname `Forge`, IP `192.168.0.28`
- **Node 2** — hostname `test`, IP `192.168.0.163`

**Document Date:** 2026-09-17
**Status:** Master-Standby cluster **operational and failover-verified** ·

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Objectives](#2-objectives)
3. [Architecture Overview](#3-architecture-overview)
4. [Approach Timeline](#4-approach-timeline)
5. [Phase 1 — Patroni + etcd (`manwilzaki/ha-postgres`)](#5-phase-1--patroni--etcd-manwilzakiha-postgres)
6. [Phase 2 — Repmgr (`soldevelo/postgresql-repmgr`)](#6-phase-2--repmgr-soldevelopostgresql-repmgr)
7. [Phase 3 — Custom Docker Compose (Working Solution)](#7-phase-3--custom-docker-compose-working-solution)
8. [Detailed Technical Walkthrough](#8-detailed-technical-walkthrough)
9. [Test Cases & Results](#9-test-cases--results)
10. [Failover Test — Detailed Walkthrough](#10-failover-test--detailed-walkthrough)
11. [Key Concepts Explained](#11-key-concepts-explained)
12. [Issues Encountered & Resolutions](#12-issues-encountered--resolutions)
13. [Operational Runbook](#13-operational-runbook)
14. [Next Steps — pgactive Multi-Master](#14-next-steps--pgactive-multi-master)
15. [Appendices](#15-appendices)

---

## 1. Executive Summary

This document describes the design, implementation, and validation of a containerized PostgreSQL high-availability cluster across two physical hosts. Two architectural patterns were investigated:

1. **Master-Standby (Primary-Replica)** — one node accepts writes, the other replicates and serves read-only traffic.
2. **Multi-Master (Active-Active)** — both nodes accept writes and synchronize changes bidirectionally.

After evaluating three different toolchains (Patroni+etcd, repmgr, and a hand-rolled Docker Compose setup), the **custom Docker Compose approach using the official `postgres:16-alpine` image** was selected for the master-standby cluster. It provides:

- Native PostgreSQL streaming replication
- Full transparency — every configuration file is version-controlled
- No external dependencies (no etcd, Patroni, or repmgr)
- Reproducible, declarative setup

**Result:**
- Master-standby cluster is **operational**.
- Streaming replication verified with seven passing data-replication test cases.
- **Failover verified end-to-end** — the standby was promoted, accepted writes on a new timeline, and the old primary was successfully re-joined as a standby of the new primary.

**Pending:** The `pgactive` multi-master implementation.

---

## 2. Objectives

| # | Objective | Status |
|---|-----------|--------|
| 1 | Deploy a containerized PostgreSQL HA cluster across two nodes | ✅ Complete |
| 2 | Verify streaming replication with a dummy table | ✅ Complete |
| 3 | Confirm the standby is read-only and rejects writes | ✅ Complete |
| 4 | Evaluate multiple HA toolchains and select the most robust | ✅ Complete |
| 5 | Test failover (promote standby, rejoin old primary) | ✅ Complete |
| 6 | Deploy and validate a `pgactive` multi-master cluster | ⏳ Pending |

---

## 3. Architecture Overview

### 3.1 Master-Standby (Physical Streaming Replication)

```
   ┌─────────────────────┐          ┌─────────────────────┐
   │      Node 1         │          │      Node 2         │
   │  192.168.0.28       │          │  192.168.0.163      │
   │                     │          │                     │
   │  ┌──────────────┐   │   WAL    │  ┌──────────────┐   │
   │  │  PostgreSQL  │───┼─────────▶│  │  PostgreSQL  │   │
   │  │   PRIMARY    │   │  stream  │  │   STANDBY    │   │
   │  │ (read/write) │   │          │  │ (read-only)  │   │
   │  └──────────────┘   │          │  └──────────────┘   │
   │         :5432       │          │         :5433       │
   └─────────────────────┘          └─────────────────────┘
```

**Characteristics:**
- One writer, one or more readers.
- WAL segments are continuously shipped from primary to standby.
- The standby is a byte-for-byte physical copy of the primary.
- DDL (schema changes) replicates automatically — it's part of the WAL stream.
- Failover is a manual (or orchestrated) promotion of the standby.

### 3.2 Multi-Master (Logical Bidirectional Replication)

```
   ┌─────────────────────┐          ┌─────────────────────┐
   │      Node 1         │          │      Node 2         │
   │                     │          │                     │
   │  ┌──────────────┐   │  logical │  ┌──────────────┐   │
   │  │  PostgreSQL  │◀──┼─────────▶│  │  PostgreSQL  │   │
   │  │  + pgactive  │   │  stream  │  │  + pgactive  │   │
   │  │ (read/write) │   │          │  │ (read/write) │   │
   │  └──────────────┘   │          │  └──────────────┘   │
   │         :5432       │          │         :5432       │
   └─────────────────────┘          └─────────────────────┘
```

**Characteristics:**
- Both nodes accept writes.
- Changes are propagated via **logical replication** (row-level events).
- DDL is **not** replicated — schema changes must be applied to every node.
- Write conflicts are possible and must be monitored.
- No single point of failure for writes.

---

## 4. Approach Timeline

| Phase | Toolchain | Outcome | Reason |
|-------|-----------|---------|--------|
| 1 | `manwilzaki/ha-postgres` (Patroni + etcd + HAProxy) | ❌ Abandoned | Repeated failures across networking, bootstrap, and auth layers |
| 2 | `soldevelo/postgresql-repmgr` (Bitnami fork) | ❌ Abandoned | Container ran as UID 1001 with no passwd entry; `repmgr` refused to run |
| 3 | Custom `docker-compose.yml` with `postgres:16-alpine` | ✅ **Working** | Transparent, no external deps, fully under our control |
| 3b | Failover rehearsal on the working cluster | ✅ **Verified** | Promote, write-forward, re-join tested end-to-end |

---

## 5. Phase 1 — Patroni + etcd (`manwilzaki/ha-postgres`)

### 5.1 What We Tried

Deployed the `manwilzaki/ha-postgres` image on both nodes. This image bundles:
- **PostgreSQL** — the database
- **Patroni** — HA manager that handles leader election and failover
- **etcd** — distributed key-value store used by Patroni for consensus
- **HAProxy** — connection router (sends writes to the leader, reads to any node)

Initial deployment used Docker bridge networking with published ports `5000, 5001, 7000, 8008`.

### 5.2 Failure Modes Encountered

| # | Symptom | Root Cause |
|---|---------|-----------|
| 1 | `curl localhost:8008/patroni` → `Connection reset by peer` | Patroni not running; etcd had not formed quorum |
| 2 | `dial tcp 192.168.0.163:2380: connection refused` in etcd log | Peer port 2380 unpublished on bridge network |
| 3 | `bootstrap failed: member has already been bootstrapped` | Stale etcd data persisted across `docker rm` |
| 4 | `no pg_hba.conf entry for host "192.168.0.28"` | Native PostgreSQL on host occupied port 5432 |
| 5 | `password authentication failed for user "replicator"` | Role never created; `pg_hba.conf` used `md5` against SCRAM hash |
| 6 | `could not remove data directory: Resource busy` | PostgreSQL process held the data directory during cleanup |

### 5.3 Why It Was Abandoned

- **etcd in a 2-node topology is inherently fragile.** With only two members, losing one node means losing quorum, which demotes the remaining primary. This defeats the purpose of HA.
- **The image's internal wiring was non-standard.** The supervisor config lacked a `[supervisorctl]` section; log paths differed from documentation; the `replicator` role was not created during bootstrap.
- **Anonymous volumes survived `docker rm`.** Stale etcd state persisted invisibly, causing bootstrap failures on every recreate.
- **The combination of Patroni + etcd + HAProxy is heavyweight** for a simple 2-node setup and introduces many moving parts to debug.

**Conclusion:** Not suitable for this use case. Documented here for completeness and to inform future architecture decisions.

---

## 6. Phase 2 — Repmgr (`soldevelo/postgresql-repmgr`)

### 6.1 What We Tried

Deployed the `soldevelo/postgresql-repmgr` image — a maintained fork of the Bitnami repmgr image. Repmgr is a lighter-weight HA manager than Patroni and does not require etcd.

Configuration used environment variables:
- `REPMGR_PRIMARY_HOST`, `REPMGR_PARTNER_NODES`, `REPMGR_NODE_NAME`, `REPMGR_NODE_ID`, etc.

### 6.2 Failure Modes Encountered

| # | Symptom | Root Cause |
|---|---------|-----------|
| 1 | `The node name does not follow the required format. Valid format: ^.*+-[0-9]+$` | Node name must end in `-<digit>` (e.g. `node-1`, not `node1`) |
| 2 | `The node id is required` | Image does not infer `REPMGR_NODE_ID`; must be set explicitly |
| 3 | `could not get current user name: Success` | Container runs as UID 1001 with no `/etc/passwd` entry |
| 4 | `repmgr: cannot be run as root` | `repmgr` refuses to execute with root privileges |

### 6.3 What Actually Worked

Despite the `repmgr` CLI being unusable inside the container, **the underlying replication worked**:
- Node 1 registered as primary
- Node 2 cloned from primary
- WAL streaming started
- `repmgrd` daemon started on both nodes

The problem was purely with **introspection** — `repmgr cluster show` could not be run to query the cluster state. Verification via `psql` (which does not have the same user constraints) was possible.

### 6.4 Why It Was Abandoned

- **The image's UID/passwd mismatch** is a known quirk of the Bitnami image family, but it made operations unnecessarily difficult.
- **Bitnami's public images were recently moved behind a commercial subscription**, and the `soldevelo` fork, while maintained, is not as widely documented.
- **A custom Compose setup would give us identical functionality with far more transparency.**

---

## 7. Phase 3 — Custom Docker Compose (Working Solution)

### 7.1 Design Goals

- Use the **official `postgres:16-alpine` image** — no third-party forks.
- Native PostgreSQL **streaming replication** — no external HA manager.
- **Declarative configuration** — all settings in version-controlled files.
- **Transparent troubleshooting** — every log line comes from PostgreSQL itself.
- **Reproducible** — `docker compose up` on each node produces the cluster.

### 7.2 File Layout

```
~/pg-ha-cluster/
├── docker-compose.yml          # Primary definition (Node 1)
├── init-primary.sh             # Runs on first start: creates replicator role + slot
├── primary/
│   ├── postgresql.conf         # Primary PostgreSQL config
│   └── pg_hba.conf             # Primary HBA rules
└── standby/
    ├── postgresql.conf         # Standby config (with hba_file directive)
    └── pg_hba.conf             # Standby HBA rules
```

**On Node 2, additionally:**
```
~/pg-ha-cluster/
├── docker-compose.yml          # Standby definition (Node 2)
└── standby-entrypoint.sh       # Wrapper: base backup → exec official entrypoint
```

### 7.3 Key Configuration Decisions

| Decision | Rationale |
|----------|-----------|
| `hba_file = '/etc/postgresql/pg_hba.conf'` | PostgreSQL defaults to reading HBA from the data directory; this directive points it at the mounted file |
| `command: postgres -c config_file=/etc/postgresql/postgresql.conf` | Tells PostgreSQL to load the mounted config instead of the data directory copy |
| Wrapper `entrypoint:` on standby | The official image's entrypoint must run to drop privileges from root to `postgres`; the wrapper does pre-start work then `exec`s it |
| `pg_basebackup -R` flag | Automatically writes `standby.signal` and `postgresql.auto.conf` for streaming replication |
| Named volume `primary_data` / `standby_data` | Persists database files across container restarts |
| Publishing port 5432 on Node 1, 5433 on Node 2 | Avoids conflict on the standby host; allows future promotion without remapping |

---

## 8. Detailed Technical Walkthrough

### 8.1 Prerequisites

**On both nodes:**
- Docker Engine and Docker Compose plugin installed
- Ports 5432 (Node 1) and 5433 (Node 2) free
- Host firewalls allow inter-node traffic on 5432 (if UFW or iptables policies are active)

**Verify port availability:**
```bash
ss -tlnp | grep -E '5432|5433'
```
Expected: no output. If a native PostgreSQL is running, stop it:
```bash
sudo systemctl stop 'postgresql@*'
sudo systemctl stop postgresql
```

### 8.2 Primary Node (Node 1 — 192.168.0.28)

**`docker-compose.yml`:**

```yaml
services:
  postgres-primary:
    image: postgres:16-alpine
    container_name: postgres-primary
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
    ports:
      - "5432:5432"
    volumes:
      - primary_data:/var/lib/postgresql/data
      - ./primary/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./primary/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./init-primary.sh:/docker-entrypoint-initdb.d/init-primary.sh
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - postgres-network

volumes:
  primary_data:

networks:
  postgres-network:
    driver: bridge
```

**`primary/postgresql.conf`:**

```conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 256MB
max_replication_slots = 5
hot_standby = on

listen_addresses = '*'
port = 5432
max_connections = 100

shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
work_mem = 4MB

hba_file = '/etc/postgresql/pg_hba.conf'

logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
```

**`primary/pg_hba.conf`:**

```conf
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
host    replication     replicator      0.0.0.0/0               md5
host    all             all             0.0.0.0/0               md5
```

**`init-primary.sh`:**

```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator123';
    SELECT pg_create_physical_replication_slot('replication_slot');
EOSQL

echo "Primary server initialization completed"
```

Make executable:
```bash
chmod +x init-primary.sh
```

**Start the primary:**
```bash
cd ~/pg-ha-cluster
docker compose up -d
docker compose logs -f postgres-primary
```

Look for: `database system is ready to accept connections`.

**Verify the HBA is loaded:**
```bash
docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT type, database, user_name, address FROM pg_hba_file_rules WHERE type='host';"
```

Should show a row: `host | {replication} | {replicator} | 0.0.0.0`.

### 8.3 Standby Node (Node 2 — 192.168.0.163)

**`docker-compose.yml`:**

```yaml
services:
  postgres-standby:
    image: postgres:16-alpine
    container_name: postgres-standby
    restart: unless-stopped
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: mydb
    ports:
      - "5433:5432"
    volumes:
      - standby_data:/var/lib/postgresql/data
      - ./standby/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./standby/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./standby-entrypoint.sh:/custom-entrypoint.sh:ro
    entrypoint: ["/bin/bash", "/custom-entrypoint.sh"]
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - postgres-network

volumes:
  standby_data:

networks:
  postgres-network:
    driver: bridge
```

**`standby-entrypoint.sh` (final, corrected version):**

The wait-loop must live **inside** the `if [ ! -f PG_VERSION ]` block. If it's outside, an already-initialized container will block forever waiting for a primary that may no longer exist.

```bash
#!/bin/bash
set -e

if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
  echo "Data directory empty; preparing for base backup..."

  until pg_isready -h <PRIMARY_HOST> -p <PRIMARY_PORT> -U admin -q; do
    echo "Waiting for primary at <PRIMARY_HOST>:<PRIMARY_PORT>..."
    sleep 2
  done

  mkdir -p /var/lib/postgresql/data
  chown -R postgres:postgres /var/lib/postgresql
  chmod 700 /var/lib/postgresql/data

  echo "Starting base backup..."
  gosu postgres env PGPASSWORD=replicator123 \
    pg_basebackup \
      -h <PRIMARY_HOST> -p <PRIMARY_PORT> \
      -U replicator \
      -D /var/lib/postgresql/data \
      -Fp -Xs -R -P -v
else
  echo "Data directory already initialized; skipping base backup."
fi

exec /usr/local/bin/docker-entrypoint.sh "$@"
```

Make executable:
```bash
chmod +x standby-entrypoint.sh
```

**`standby/postgresql.conf`:**

```conf
hot_standby = on
listen_addresses = '*'
port = 5432
max_connections = 100
shared_buffers = 256MB
hba_file = '/etc/postgresql/pg_hba.conf'
```

**`standby/pg_hba.conf`:**

```conf
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
host    all             all             0.0.0.0/0               md5
host    replication     replicator      0.0.0.0/0               md5
```

> The last line (`host replication ...`) must be present on the **standby** when it becomes a primary, so that other nodes can stream from it.

**Start the standby:**
```bash
cd ~/pg-ha-cluster
docker compose up -d
docker compose logs -f postgres-standby
```

Look for:
- `pg_basebackup: base backup completed`
- `entering standby mode`
- `started streaming WAL from primary`
- `database system is ready to accept read-only connections`

### 8.4 Verification

**On the primary:**
```bash
docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

Expected:
```
 client_addr  |   state   | sync_state
---------------+-----------+------------
 192.168.0.163 | streaming | async
```

**On the standby:**
```bash
docker exec -i postgres-standby psql -U admin -d mydb -c "SELECT 1;"
```

### 8.5 Dummy Table Test

**Create on primary:**
```bash
docker exec -i postgres-primary psql -U admin -d mydb <<'EOF'
CREATE TABLE dummy_ha (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    value NUMERIC,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
INSERT INTO dummy_ha (name, value) VALUES
    ('alpha', 100.50), ('beta', 200.75), ('gamma', 300.25);
EOF
```

**Read on standby:**
```bash
docker exec -i postgres-standby psql -U admin -d mydb -c "SELECT * FROM dummy_ha;"
```

**Write-forward test:**
```bash
# On primary
docker exec -i postgres-primary psql -U admin -d mydb -c \
  "INSERT INTO dummy_ha (name, value) VALUES ('delta', 400.00);"

# On standby
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT count(*) FROM dummy_ha;"
```
Expected: `4`.

**Read-only enforcement:**
```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "INSERT INTO dummy_ha (name, value) VALUES ('should-fail', 0);"
```
Expected: `ERROR: cannot execute INSERT in a read-only transaction`.

---

## 9. Test Cases & Results

### 9.1 Infrastructure & Connectivity

| ID | Test Case | Expected | Actual | Status |
|----|-----------|----------|--------|--------|
| INF-01 | Primary container healthy | `Up (healthy)` | Confirmed | ✅ |
| INF-02 | Standby container healthy | `Up` | Confirmed | ✅ |
| INF-03 | Primary publishes port 5432 | `0.0.0.0:5432->5432/tcp` | Confirmed | ✅ |
| INF-04 | Standby publishes port 5433 | `0.0.0.0:5433->5432/tcp` | Confirmed | ✅ |
| INF-05 | Host→container connectivity (Node 1) | `SELECT 1` returns row | Confirmed | ✅ |
| INF-06 | Node 2→Node 1:5432 reachable | TCP connection succeeds | Confirmed | ✅ |
| INF-07 | `pg_hba.conf` allows replication from Node 2 | Entry present in `pg_hba_file_rules` | Confirmed | ✅ |
| INF-08 | iptables `DOCKER` chain clean | Only `ACCEPT` rules | After fix | ✅ |

### 9.2 Replication Setup

| ID | Test Case | Expected | Actual | Status |
|----|-----------|----------|--------|--------|
| REP-01 | Replication slot created | `replication_slot` exists | Confirmed | ✅ |
| REP-02 | `pg_basebackup` completes | `base backup completed` | Confirmed | ✅ |
| REP-03 | Standby enters recovery | `entering standby mode` | Confirmed | ✅ |
| REP-04 | WAL streaming starts | `started streaming WAL` | Confirmed | ✅ |
| REP-05 | Primary sees standby | `state = streaming` | `192.168.0.163 / streaming` | ✅ |
| REP-06 | Standby reachable for reads | `SELECT 1` succeeds | Confirmed | ✅ |

### 9.3 Data Replication

| ID | Test Case | Expected | Actual | Status |
|----|-----------|----------|--------|--------|
| DAT-01 | Create table on primary | `CREATE TABLE` | Confirmed | ✅ |
| DAT-02 | Insert 3 rows on primary | `INSERT 0 3` | Confirmed | ✅ |
| DAT-03 | Read 3 rows on primary | 3 rows | Confirmed | ✅ |
| DAT-04 | Read 3 rows on standby | 3 rows, identical timestamps | Confirmed | ✅ |
| DAT-05 | Insert additional row on primary | `INSERT 0 1` | Confirmed | ✅ |
| DAT-06 | New row visible on standby | `count = 4` | Confirmed | ✅ |
| DAT-07 | Standby rejects writes | `read-only transaction` error | Confirmed | ✅ |

### 9.4 Failover (Complete)

| ID | Test Case | Expected | Actual | Status |
|----|-----------|----------|--------|--------|
| FAI-01 | Promote standby | Primary role | `pg_ctl promote` succeeded; timeline bumped to 2 | ✅ |
| FAI-02 | Promoted node accepts writes | `INSERT` succeeds | `INSERT 0 1` for `post-failover` | ✅ |
| FAI-03 | `pg_is_in_recovery()` returns `f` | `f` | `f` | ✅ |
| FAI-04 | Timeline advances | `2` | `2` | ✅ |
| FAI-05 | Old primary rejoined as standby | Base backup + streaming | `pg_basebackup` completed; `started streaming WAL from primary at timeline 2` | ✅ |
| FAI-06 | Replication direction reversed | Node 2 → Node 1 | `192.168.0.28 / streaming / async` on Node 2 | ✅ |
| FAI-07 | New standby read-only | `INSERT` rejected | `cannot execute INSERT in a read-only transaction` | ✅ |
| FAI-08 | Data integrity post-failover | All 6 rows on both nodes | Confirmed (6 rows on both) | ✅ |

### 9.5 Multi-Master (Pending)

| ID | Test Case | Expected | Actual | Status |
|----|-----------|----------|--------|--------|
| MMA-01 | Custom image builds with pgactive | Image builds | Not run | ⏳ |
| MMA-02 | `shared_preload_libraries` includes `pgactive` | Confirmed via `SHOW` | Not run | ⏳ |
| MMA-03 | `CREATE EXTENSION pgactive` on both nodes | Success | Not run | ⏳ |
| MMA-04 | `pgactive_create_group()` on Node 1 | Group created | Not run | ⏳ |
| MMA-05 | `pgactive_join_group()` on Node 2 | Node joins, `ready` | Not run | ⏳ |
| MMA-06 | `pgactive_nodes` shows 2 ready rows | 2 rows, `ready` | Not run | ⏳ |
| MMA-07 | DDL applied to both nodes | Table exists on both | Not run | ⏳ |
| MMA-08 | Insert on Node 1 → visible on Node 2 | Row replicates | Not run | ⏳ |
| MMA-09 | Insert on Node 2 → visible on Node 1 | Bidirectional | Not run | ⏳ |
| MMA-10 | Conflict history query | Empty (no conflicts) | Not run | ⏳ |
| MMA-11 | Replication stats show commits | Non-zero counters | Not run | ⏳ |

---

## 10. Failover Test — Detailed Walkthrough

The failover test was carried out in four phases, with verification at each stage. The initial state was Node 1 as primary (timeline 1) and Node 2 as standby, with 4 rows in `dummy_ha`.

### 10.1 Phase A — Pre-Flight Verification

**On Node 1 (primary):**

```bash
docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT client_addr, state, sync_state FROM pg_stat_replication;"

docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT timeline_id FROM pg_control_checkpoint();"

docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT * FROM dummy_ha ORDER BY id;"
```

**Result:**
- `pg_stat_replication`: `192.168.0.163 / streaming / async`
- `timeline_id = 1`
- `dummy_ha`: 4 rows (alpha, beta, gamma, delta)

**On Node 2 (standby):**

```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT pg_is_in_recovery();"

docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT count(*) FROM dummy_ha;"
```

**Result:**
- `pg_is_in_recovery = t`
- `count = 4`

✅ Pre-flight state clean.

### 10.2 Phase B — Promote Node 2

**Step 1 — Stop Node 1 (simulate primary failure):**

```bash
cd ~/pg-ha-cluster
docker compose stop postgres-primary
```

Verified: `docker ps --filter name=postgres-primary` returned no running container.

**Step 2 — Promote Node 2:**

The official `postgres` image runs as root by default for `docker exec`, and `pg_ctl` refuses to run as root. The promotion must be executed as the `postgres` user:

```bash
docker exec -u postgres -i postgres-standby pg_ctl promote -D /var/lib/postgresql/data
```

Output:
```
waiting for server to promote.... done
server promoted
```

**Step 3 — Observe promotion in logs:**

```
LOG:  received promote request
LOG:  redo done at 0/502A078 system usage: CPU: user: 0.34 s, system: 1.13 s, elapsed: 143370.50 s
LOG:  last completed transaction was at log time 2026-09-15 12:36:57.861838+00
LOG:  selected new timeline ID: 2
LOG:  archive recovery complete
LOG:  database system is ready to accept connections
```

**Step 4 — Verify promotion:**

```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT pg_is_in_recovery();"
# f

docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT timeline_id FROM pg_control_checkpoint();"
# 2
```

**Step 5 — Write to new primary:**

```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "INSERT INTO dummy_ha (name, value) VALUES ('post-failover', 500.00);"
# INSERT 0 1
```

5 rows total, with `post-failover` as the last row.

✅ Node 2 is now the sole primary. Writes succeed. Node 1 is offline.

### 10.3 Phase C — Rejoin Node 1 as a Standby of Node 2

**Step 1 — Prepare Node 2's HBA to accept replication:**

Append a replication rule to `~/pg-ha-cluster/standby/pg_hba.conf` on Node 2:

```
host replication replicator 0.0.0.0/0 md5
```

Restart Node 2:

```bash
docker compose restart postgres-standby
```

Verify:

```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT type, database, user_name, address FROM pg_hba_file_rules WHERE type='host';"
```

Expected: 4 rows including `host | {replication} | {replicator} | 0.0.0.0`.

**Step 2 — Test replication connectivity from Node 1:**

```bash
timeout 3 bash -c '</dev/tcp/192.168.0.163/5433' && echo "5433 OPEN" || echo "5433 CLOSED"
# 5433 OPEN

PGPASSWORD=replicator123 psql -h 192.168.0.163 -p 5433 -U replicator -d postgres \
  -c "SELECT current_user;"
# current_user
# --------------
#  replicator
```

**Step 3 — Rewrite Node 1 as a standby:**

- Created `~/pg-ha-cluster/standby-entrypoint.sh` pointing at `192.168.0.163:5433` (corrected version, with the wait-loop inside the `if`).
- Created `~/pg-ha-cluster/standby/postgresql.conf` and `standby/pg_hba.conf`.
- Backed up the primary compose file to `docker-compose.yml.primary-backup`.
- Wrote a new `docker-compose.yml` with the standby service (`postgres-standby-node1`) using the wrapper entrypoint.

**Step 4 — Wipe Node 1's old primary volume:**

```bash
docker compose down --remove-orphans
docker volume rm pg-ha-cluster_primary_data
docker volume rm pg-ha-cluster_standby_data
```

**Step 5 — Start Node 1 as standby:**

```bash
docker compose up -d
docker compose logs -f postgres-standby-node1
```

Log sequence:
```
Data directory empty; preparing for base backup from Node 2...
Starting base backup from primary (Node 2)...
pg_basebackup: write-ahead log start point: 0/6000028 on timeline 2
pg_basebackup: base backup completed
PostgreSQL Database directory appears to contain a database; Skipping initialization
LOG:  entering standby mode
LOG:  starting backup recovery with redo LSN 0/6000028, checkpoint LSN 0/6000060, on timeline ID 2
LOG:  database system is ready to accept read-only connections
LOG:  started streaming WAL from primary at 0/7000000 on timeline 2
```

✅ Node 1 has re-joined as a standby of Node 2, streaming on timeline 2.

### 10.4 Phase D — Final Verification

**On Node 2 (new primary):**

```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

Result:
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.28 | streaming | async
```

**On Node 1 (new standby):**

```bash
docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT pg_is_in_recovery();"
# t

docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT timeline_id FROM pg_control_checkpoint();"
# 2

docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT count(*) FROM dummy_ha;"
# 5
```

**Write-forward test:**

```bash
# On Node 2 (new primary)
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "INSERT INTO dummy_ha (name, value) VALUES ('after-rejoin', 600.00);"
# INSERT 0 1

# On Node 1 (new standby)
docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT count(*) FROM dummy_ha;"
# 6
```

**Read-only enforcement on new standby:**

```bash
docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "INSERT INTO dummy_ha (name, value) VALUES ('should-fail', 0);"
# ERROR:  cannot execute INSERT in a read-only transaction
```

**Final data snapshot on Node 2:**

```
 id |     name      | value  |          created_at
----+---------------+--------+-------------------------------
  1 | alpha         | 100.50 | 2026-09-15 12:35:49.661865+00
  2 | beta          | 200.75 | 2026-09-15 12:35:49.661865+00
  3 | gamma         | 300.25 | 2026-09-15 12:35:49.661865+00
  4 | delta         | 400.00 | 2026-09-15 12:36:57.857364+00
 34 | post-failover | 500.00 | 2026-09-17 04:23:30.459997+00
 35 | after-rejoin  | 600.00 | 2026-09-17 05:07:00.469961+00
(6 rows)
```

✅ All checks passed. Failover and rejoin complete.

### 10.5 Summary Table

| Stage | Primary | Standby | Timeline | Direction | Rows |
|-------|---------|---------|----------|-----------|------|
| Initial | Node 1 | Node 2 | 1 | 1 → 2 | 4 |
| After promotion | Node 2 | — | 2 | — | 5 |
| After rejoin | Node 2 | Node 1 | 2 | 2 → 1 | 5 |
| After write-forward | Node 2 | Node 1 | 2 | 2 → 1 | 6 |

### 10.6 Observed Behaviour — Sequence ID Jump

The `id` values in `dummy_ha` jumped from `4` to `34` then `35` across the failover. This is **expected behaviour**:

- PostgreSQL sequences use a **cache** (default 32) on the primary to reduce WAL writes.
- The old primary (Node 1) had cached IDs 5–36 in memory at the time of failover.
- Those cached values were never flushed to WAL, so the standby never saw them.
- After promotion, Node 2's sequence resumed from its own cached position — producing 34, 35, etc.

**This is not a data integrity issue**, but it means IDs are not strictly contiguous across failovers. Mitigations:

1. Set `CACHE 1` on sequences (performance cost, strict ordering).
2. Use UUIDs or application-managed IDs.

Neither is required for correctness; noted for the runbook.

### 10.7 Known Limitation — No Automatic Failover

A two-node topology has **no quorum witness**. Automatic failover cannot be made reliable without a third node or a witness service (etcd, Consul, or a lightweight witness). Production deployments requiring automatic failover should:

- Add a third node (1 primary + 2 standbys), or
- Add a witness node, or
- Use an HA manager (Patroni with a 3-node etcd, or repmgr with a witness).

---

## 11. Key Concepts Explained

### 11.1 Streaming Replication

PostgreSQL records every change in a **Write-Ahead Log (WAL)**. In streaming replication, the primary continuously ships WAL records to the standby over a TCP connection. The standby applies these records to its own copy of the data files, keeping it byte-identical to the primary.

**Timeline:**
1. Standby connects to primary using credentials in `primary_conninfo`.
2. Primary starts a **WAL sender** process for that connection.
3. Standby's **WAL receiver** streams records and writes them to disk.
4. Standby's **startup process** applies records continuously.
5. The standby is always in **recovery mode** — it accepts only read queries.

### 11.2 Replication Slots

A replication slot is a mechanism that prevents the primary from discarding WAL segments that a standby hasn't yet received. Without a slot, the primary might recycle WAL that a temporarily-disconnected standby needs, causing the standby to fall behind irrecoverably.

`init-primary.sh` creates a physical slot named `replication_slot`:
```sql
SELECT pg_create_physical_replication_slot('replication_slot');
```

### 11.3 `pg_hba.conf` and `hba_file`

`pg_hba.conf` controls client authentication. PostgreSQL reads it from the data directory by default. Since we want to keep configuration in version-controlled files outside the container, we:
1. Mount the file at `/etc/postgresql/pg_hba.conf`.
2. Add `hba_file = '/etc/postgresql/pg_hba.conf'` to `postgresql.conf` so PostgreSQL reads the mounted file.

Without this directive, PostgreSQL silently reads the default file inside the data directory, and any replication rule you add to the mounted file has no effect.

### 11.4 Privilege Drop in the Official Image

The official `postgres` image's entrypoint runs as **root** initially, performs setup, then uses `gosu` to drop privileges to the `postgres` user before starting the server. If you override `command:` with a raw `bash -c "postgres ..."`, that drop never happens, and PostgreSQL refuses to start with:
```
"root" execution of the PostgreSQL server is not permitted.
```
The correct pattern is to provide a **wrapper entrypoint** that does pre-start work and then `exec`s the official entrypoint.

Similarly, `pg_ctl promote` and other administrative commands must be run with `-u postgres` because `pg_ctl` refuses to execute as root.

### 11.5 PostgreSQL Timelines

Every promotion creates a new **timeline** — an ordered sequence of WAL records starting from the promotion point. Timelines ensure that after a failover, the new primary's WAL history diverges cleanly from the old primary's, so WAL from the old branch can never be mistakenly applied.

- A freshly initialized cluster starts at **timeline 1**.
- The first promotion bumps to **timeline 2**.
- Each subsequent promotion increments further.

The standby that rejoins a promoted primary must be re-based from the new timeline's history, which is why the old primary's data directory must be wiped before rejoining — it belongs to the old timeline branch and cannot be reconciled.

### 11.6 Docker Bridge Networking + Published Ports

When you specify `ports: - "5432:5432"`:
1. Docker gives the container an internal IP (e.g. `172.18.0.2`).
2. Docker starts a `docker-proxy` process listening on `0.0.0.0:5432` on the host.
3. Docker inserts iptables `DNAT` rules in the `nat` table that rewrite inbound packets to the container's IP.
4. Docker inserts `ACCEPT` rules in the `DOCKER` iptables chain for the published port.

**You never need to reference the container's internal IP from outside the host.** Remote clients connect to the host IP and port; Docker handles the translation.

### 11.7 The `iptables DOCKER` Chain

Docker maintains its own iptables chain called `DOCKER`. Rules here are executed for packets destined to published container ports. Docker inserts `ACCEPT` rules automatically when containers start.

**Any additional rules you add manually** (like blanket `DROP`s for security hardening) will sit alongside Docker's rules. If they end up *above* Docker's `ACCEPT`, they'll silently drop traffic to your containers.

**Lesson:** Never insert `DROP` rules into the `DOCKER` chain. Use the `DOCKER-USER` chain instead, which is designed for user-defined policies and is evaluated *before* `DOCKER`.

---

## 12. Issues Encountered & Resolutions

### 12.1 Phase 1 & 2 (Abandoned Toolchains)

| # | Issue | Root Cause | Resolution |
|---|-------|-----------|------------|
| 1 | Patroni REST API unreachable | etcd had not formed quorum | Abandoned toolchain |
| 2 | etcd peer unreachable across bridge | Port 2380 unpublished | Switched to `--network host` |
| 3 | Stale etcd bootstrap | Anonymous volume survived `docker rm` | Manual `docker volume rm` |
| 4 | Missing `replicator` role | Patroni bootstrap did not create it | Manual `CREATE ROLE` |
| 5 | `md5` vs `scram-sha-256` mismatch | HBA method incompatible with password hash | Manual HBA edit |
| 6 | Native PostgreSQL on host held port 5432 | System PostgreSQL running | `systemctl stop postgresql@*` |
| 7 | Repmgr node name format rejected | Requires `<name>-<digit>` | Renamed to `node-1` / `node-2` |
| 8 | `repmgr` CLI unusable in container | UID 1001 has no passwd entry; root rejected | Abandoned toolchain |

### 12.2 Phase 3 (Working Solution)

| # | Issue | Root Cause | Resolution |
|---|-------|-----------|------------|
| 9 | `docker port` returned nothing | Container created before `ports:` was added | `docker compose down && up -d` |
| 10 | `Connection refused` despite healthy container | Rogue `DROP` rules in `DOCKER` chain | `iptables -D DOCKER <n>` |
| 11 | Deleted wrong iptables rule | Incorrect rule number provided | Recreated container to re-insert ACCEPT |
| 12 | `pg_basebackup: no pg_hba.conf entry` | PostgreSQL read HBA from data dir | Added `hba_file` directive |
| 13 | `"root" execution ... not permitted` | Custom `command:` bypassed entrypoint | Wrapper `entrypoint:` that `exec`s original |
| 14 | `cannot attach stdin to a TTY` with heredoc | `-t` conflicts with piped stdin | Use `docker exec -i` |
| 15 | Config files unreadable by container | Host file permissions | `chmod 644` on mounted `.conf` files |

### 12.3 Failover Phase

| # | Issue | Root Cause | Resolution |
|---|-------|-----------|------------|
| 16 | `pg_ctl: cannot be run as root` | Docker exec default user is root | `docker exec -u postgres -i ... pg_ctl promote` |
| 17 | Orphan container held port 5432 | `docker compose down` doesn't remove services not in current compose file | `docker compose down --remove-orphans` |
| 18 | Orphan volume could not be removed | Attached to a stopped-but-not-removed container | `docker rm <container>` before `docker volume rm` |
| 19 | Wrapper entrypoint blocked on restart | Wait-loop was outside the `if [ ! -f PG_VERSION ]` block | Move the wait-loop inside the `if` |
| 20 | Standby never re-joined as primary after role flip | Wrapper kept waiting for the old primary IP | Rewrite wrapper to target the new primary |

### 12.4 Recurring / Unresolved

| # | Issue | Status |
|---|-------|--------|
| 21 | Rogue `DROP` rules reappear in `DOCKER` chain | Source script still unidentified; mitigated by manual cleanup |
| 22 | Sequence ID gaps after failover | Inherent to physical replication; documented; use `CACHE 1` or UUIDs if strict ordering needed |

---

## 13. Operational Runbook

### 13.1 Daily Operations

**Check cluster health (on primary):**
```bash
docker exec -i postgres-primary psql -U admin -d mydb -c \
  "SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"
```

**Check replication lag (on standby):**
```bash
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;"
```

**View container logs:**
```bash
docker logs postgres-primary --tail 100
docker logs postgres-standby --tail 100
```

### 13.2 Restart a Node

**Restart standby (safe):**
```bash
cd ~/pg-ha-cluster
docker compose restart postgres-standby
```

**Restart primary (brief write outage):**
```bash
cd ~/pg-ha-cluster
docker compose restart postgres-primary
```

### 13.3 Failover Procedure (Tested)

**Phase 1 — Promote the standby:**

```bash
# Stop the current primary (simulates failure or planned maintenance)
# On Node 1 (current primary):
cd ~/pg-ha-cluster
docker compose stop postgres-primary

# Promote Node 2 (current standby)
# On Node 2:
docker exec -u postgres -i postgres-standby pg_ctl promote -D /var/lib/postgresql/data

# Verify promotion
docker exec -i postgres-standby psql -U admin -d mydb -c "SELECT pg_is_in_recovery();"
# expected: f
docker exec -i postgres-standby psql -U admin -d mydb -c "SELECT timeline_id FROM pg_control_checkpoint();"
# expected: 2 (or higher)
```

**Phase 2 — Add replication HBA rule to the new primary:**

```bash
# On Node 2:
echo "host replication replicator 0.0.0.0/0 md5" >> ~/pg-ha-cluster/standby/pg_hba.conf
docker compose restart postgres-standby
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT type, database, user_name, address FROM pg_hba_file_rules WHERE type='host';"
```

**Phase 3 — Convert old primary into a standby of the new primary:**

```bash
# On Node 1 (old primary):
cd ~/pg-ha-cluster
cp docker-compose.yml docker-compose.yml.primary-backup

# Write a new compose file with the standby service (see §7 for template)
# Ensure standby-entrypoint.sh targets 192.168.0.163:5433

docker compose down --remove-orphans
docker volume rm pg-ha-cluster_primary_data
docker volume rm pg-ha-cluster_standby_data
docker compose up -d
docker compose logs -f postgres-standby-node1
```

Look for: `entering standby mode`, `started streaming WAL from primary at timeline 2`, `database system is ready to accept read-only connections`.

**Phase 4 — Verify:**

```bash
# On Node 2 (new primary):
docker exec -i postgres-standby psql -U admin -d mydb -c \
  "SELECT client_addr, state FROM pg_stat_replication;"
# expected: 192.168.0.28 / streaming

# On Node 1 (new standby):
docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT pg_is_in_recovery();"    # t
docker exec -i postgres-standby-node1 psql -U admin -d mydb -c \
  "SELECT timeline_id FROM pg_control_checkpoint();"    # 2
```

### 13.4 Full Cluster Rebuild

**On both nodes:**
```bash
docker compose down --remove-orphans
docker volume rm pg-ha-cluster_primary_data pg-ha-cluster_standby_data 2>/dev/null
```

**On Node 1 (primary):**
```bash
docker compose up -d    # using the primary compose file
```

Wait for `database system is ready to accept connections`.

**On Node 2 (standby):**
```bash
docker compose up -d    # using the standby compose file
docker compose logs -f postgres-standby
```

### 13.5 Backup

**Physical backup (on primary):**
```bash
docker exec -i postgres-primary pg_basebackup \
  -D /tmp/backup -Ft -z -P -U replicator
docker cp postgres-primary:/tmp/backup ./backup-$(date +%F).tar.gz
```

**Logical backup (any node):**
```bash
docker exec -i postgres-primary pg_dump -U admin -d mydb -Fc > mydb-$(date +%F).dump
```

### 13.6 Best Practices Established

1. **Always use `docker compose down --remove-orphans`** when switching compose file versions (e.g. between primary and standby roles). Otherwise old containers linger and hold ports.
2. **Always run `pg_ctl` with `-u postgres`** inside the official image. The default docker exec user is `root`, which `pg_ctl` rejects.
3. **The wrapper entrypoint's wait-loop must be inside the `if [ ! -f PG_VERSION ]` block.** An already-initialized container must not wait for a primary.
4. **Never insert `DROP` rules into the `DOCKER` iptables chain.** Use `DOCKER-USER` for custom policies.
5. **Mounted config files must be `chmod 644`** and referenced explicitly via `hba_file` / `config_file` directives.
6. **Named volumes must be removed explicitly by name**, not via `docker volume prune` (which only removes truly dangling volumes).

---

## 14. Next Steps — pgactive Multi-Master

### 14.1 What Needs to Happen

1. **Build a custom Docker image** based on `postgres:17-bookworm` that compiles and installs the `pgactive` extension (v2.1.7).
2. **Deploy one container per node** with `shared_preload_libraries = 'pgactive'` and logical replication settings.
3. **Create a pgactive group** on Node 1 and join Node 2.
4. **Test bidirectional writes** with a dummy table.
5. **Monitor conflict history** via `pgactive_conflict_history`.

### 14.2 Dockerfile Sketch

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
COPY --from=builder /usr/lib/postgresql/17/lib/pgactive.so /usr/lib/postgresql/17/lib/
COPY --from=builder /usr/share/postgresql/17/extension/pgactive* /usr/share/postgresql/17/extension/
```

### 14.3 Configuration Additions

To the `postgresql.conf` used by each multi-master node:
```conf
wal_level = logical
max_worker_processes = 64
max_replication_slots = 32
max_wal_senders = 32
shared_preload_libraries = 'pgactive'
track_commit_timestamp = on
max_parallel_workers = 16
max_logical_replication_workers = 32
max_sync_workers_per_subscription = 8
pgactive.max_nodes = 16
```

### 14.4 Key Operational Caveat

**DDL is not replicated by pgactive.** Any `CREATE TABLE`, `ALTER TABLE`, or index change must be applied to **every node**. Plan your migrations accordingly.

### 14.5 Open Questions

- Conflict resolution strategy for concurrent writes to the same row.
- Whether `pgactive` is production-grade for the intended workload.
- Monitoring hooks for `pgactive_conflict_history` and `pgactive_stats`.

---

## 15. Appendices

### Appendix A — Complete Command Reference

**Container lifecycle:**
```bash
docker compose up -d                          # Start in background
docker compose down                           # Stop and remove
docker compose down --remove-orphans          # Also remove orphan containers
docker compose logs -f <service>              # Stream logs
docker compose restart <service>              # Restart
docker volume rm <volume>                     # Delete a volume
```

**Database access:**
```bash
docker exec -i postgres-primary psql -U admin -d mydb -c "SQL..."
docker exec -i postgres-standby psql -U admin -d mydb -c "SQL..."
docker exec -i postgres-primary psql -U admin -d mydb <<'EOF' ... EOF
```

**Administrative operations (must use `-u postgres`):**
```bash
docker exec -u postgres -i postgres-standby pg_ctl promote -D /var/lib/postgresql/data
docker exec -u postgres -i postgres-standby pg_ctl status -D /var/lib/postgresql/data
```

**Replication diagnostics:**
```sql
-- On primary
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
SELECT timeline_id FROM pg_control_checkpoint();

-- On standby
SELECT pg_is_in_recovery();
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;
SELECT * FROM pg_stat_wal_receiver;
SELECT timeline_id FROM pg_control_checkpoint();

-- Replication slots
SELECT * FROM pg_replication_slots;
```

**Networking diagnostics:**
```bash
ss -tlnp | grep 5432
docker port postgres-primary
docker inspect postgres-primary --format '{{json .NetworkSettings.Ports}}'
sudo iptables -L DOCKER -n
```

### Appendix B — Failure Taxonomy

| Layer | Typical Symptom | Where to Look |
|-------|-----------------|---------------|
| Networking | `connection refused`, `no response` | `ss -tlnp`, `iptables -L`, `ip route` |
| Port publishing | `docker port` empty | `docker-compose.yml` `ports:` block |
| Authentication | `no pg_hba.conf entry`, `password authentication failed` | `pg_hba_file_rules`, `pg_authid` |
| Config loading | Mounted file ignored | `SHOW hba_file`, `SHOW config_file` |
| Privilege | `"root" execution ... not permitted` | Container `entrypoint`, `user:` |
| Replication | `pg_stat_replication` empty | `primary_conninfo`, network, HBA |
| Failover | Standby refuses to promote | `pg_ctl promote`, `pg_is_in_recovery()` |
| Timeline | Standby won't rejoin after failover | Timeline mismatch — wipe and re-base |
| Storage | `Resource busy`, `could not remove` | Stopped processes, volume state |

### Appendix C — Glossary

| Term | Definition |
|------|-----------|
| **WAL** | Write-Ahead Log; PostgreSQL's durable change log |
| **Streaming replication** | Continuous WAL shipping over TCP from primary to standby |
| **Physical replication** | Byte-for-byte copy of the data directory; standby is a clone |
| **Logical replication** | Row-level change events; replicas can differ in structure |
| **Replication slot** | Bookmark that prevents WAL recycling before a standby consumes it |
| **Base backup** | Full snapshot of the primary used to initialize a standby (`pg_basebackup`) |
| **`pg_hba.conf`** | Host-Based Authentication; controls who can connect from where |
| **`pg_hba_file_rules`** | System view that shows the effective rules from `pg_hba.conf` |
| **`standby.signal`** | File that tells PostgreSQL to start in recovery mode |
| **`pg_basebackup -R`** | Flag that auto-generates `standby.signal` and `primary_conninfo` |
| **`gosu`** | The tool the official image uses to drop privileges |
| **Promotion** | Converting a standby into a primary (`pg_ctl promote`) |
| **Timeline** | An ordered sequence of WAL records; each promotion creates a new one |
| **Quorum** | Majority agreement needed by distributed consensus systems (etcd, Consul) |
| **Witness node** | A third node that doesn't store data but participates in quorum |
| **Orphan container** | A container whose service was removed from the compose file but is still running |
| **pgactive** | AWS logical replication extension for PostgreSQL multi-master |

---

**Document End.**
