# Hybrid PostgreSQL Cluster — Active-Active ⇄ Active-Passive Mode Switching

**Project:** Two-node PostgreSQL 17 cluster with a single-instruction mode toggle between `pgactive` active-active and streaming-replication active-passive
**Environment:** tbNAC NM Ubuntu v3.1.2-2026-09-30 · Docker Compose · Host networking · port 5434
**Reference nodes:** Node 1 `Forge` 192.168.0.28 · Node 2 `Jigar-6` 192.168.0.209
**Database:** `tbnac_core` / `tbnac_core_user` · schema `nac` (39 tables, 32 sequences)
**Companion documents:** Master-Standby Implementation Report · Failure Scenarios Addendum · Multi-Master Bridge Report · tbNAC Host-Networking Report
**Document status:** Design proposal — implementation not yet performed

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement and Motivation](#2-problem-statement-and-motivation)
3. [Prior Art and Feasibility](#3-prior-art-and-feasibility)
4. [Architecture Overview](#4-architecture-overview)
5. [Why This Is Not a Native Toggle](#5-why-this-is-not-a-native-toggle)
6. [The Two-Folder Configuration Model](#6-the-two-folder-configuration-model)
7. [The Mode Switch State Machine](#7-the-mode-switch-state-machine)
8. [Direction 1 — Active-Active to Active-Passive](#8-direction-1--active-active-to-active-passive)
9. [Direction 2 — Active-Passive to Active-Active](#9-direction-2--active-passive-to-active-active)
10. [The Four Gaps and How They Are Closed](#10-the-four-gaps-and-how-they-are-closed)
11. [Sequence Offset Discipline Across Modes](#11-sequence-offset-discipline-across-modes)
12. [The Wrapper Script](#12-the-wrapper-script)
13. [Repository Layout](#13-repository-layout)
14. [What Changes and What Stays the Same](#14-what-changes-and-what-stays-the-same)
15. [Write Quiescence Strategy](#15-write-quiescence-strategy)
16. [Test Plan](#16-test-plan)
17. [Failure Handling During the Switch](#17-failure-handling-during-the-switch)
18. [Operational Runbook](#18-operational-runbook)
19. [Risks, Limitations and Non-Goals](#19-risks-limitations-and-non-goals)
20. [Appendices](#20-appendices)

---

## 1. Executive Summary

This document specifies a two-node PostgreSQL cluster that can operate in **either** of two modes on the **same physical hosts and the same data volumes**:

- **Mode A — Active-Active:** both nodes accept writes; changes propagate bidirectionally via `pgactive` logical replication.
- **Mode B — Active-Passive:** one node accepts writes; the other is a read-only physical streaming standby.

The mode is changed by a **single operator instruction** — a wrapper script command such as `./switch.sh active-passive` — which performs a pre-flight check, quiesces application writes, tears down the current replication topology, reconciles the data directories, swaps the declarative configuration folder, starts the cluster in the new mode, verifies it, and unfreezes writes.

**The key architectural fact:** this is **one cluster with two configurations**, not two clusters. The hosts, IP addresses, port, schema, and data volumes are identical in both modes. Only the replication engine and the configuration files change.

**The key limitation:** the switch is **not online**. It requires a short, bounded write freeze for the duration of the conversion. It does **not** provide automatic failover during the switch, and it does **not** eliminate the split-brain risk inherent to a two-node topology without a quorum witness.

**The key prior art:** the pattern is vendor-documented. Fujitsu Enterprise Postgres documents a physical standby attached to a `pgactive` node; EDB's BDR/PGD — the ancestor of `pgactive` — documents "Physical Standby Nodes" of a BDR node; and the PGCon active-active talk explicitly states that physical standbys are *not* part of the active-active HA set. The hybrid topology is a supported extension of the documented model, not an invention.

---

## 2. Problem Statement and Motivation

The tbNAC deployment (File 4) runs `pgactive` active-active on two nodes. The failure-scenario testing (File 2) established that the active-active pair has **no quorum witness** and therefore cannot provide automatic failover or split-brain prevention. The master-standby build (File 1) established that a physical streaming standby is simple, safe, and provides automatic recovery from transient restarts but requires **manual promotion** for primary failure.

The two modes are complementary:

| Concern | Active-Active handles it well | Active-Passive handles it well |
|---|---|---|
| Write throughput scaling | ✅ Both nodes write | ❌ Single writer |
| Latency to local application tier | ✅ Local writes on both nodes | ✅ Local reads on standby |
| Automatic recovery from transient restarts | Partial | ✅ For both directions |
| Split-brain prevention | ❌ No witness | ✅ Standby is read-only |
| DDL propagation | ❌ Manual per node | ✅ Automatic via WAL |
| Sequence collision risk | Present (mitigated by offsets) | Absent (single writer) |
| Operational complexity | Higher | Lower |

The motivation for mode switching is **operational flexibility**: run active-active during steady-state multi-site operation, and switch to active-passive for maintenance, schema migration, or during a period when one node must be taken out of the write set. The alternative — rebuilding the cluster each time — is costly and error-prone. The two-folder model makes the mode a **declarative configuration choice**, version-controlled and reproducible.

---

## 3. Prior Art and Feasibility

### 3.1 Fujitsu Enterprise Postgres — physical standby of a pgactive node

Fujitsu's documented procedure uses the same primitives this design relies on:

- `synchronized_standby_slots` on the pgactive node, naming the physical replication slot used by the standby.
- `pgactive.pgactive_apply_pause()` / `pgactive_apply_resume()` to freeze logical apply during the base backup.
- `pg_basebackup -R -S <physical_slot>` to initialize the standby.
- The standby starts with `hot_standby = on` and a `primary_conninfo` pointing at the pgactive node.

This confirms that a physical standby of a pgactive node is a **supported configuration**, not a hack.

### 3.2 EDB BDR / PGD — Physical Standby Nodes

The BDR/PGD documentation — which predates pgactive and describes the same replication lineage — documents physical standby nodes attached to BDR nodes, intended to **directly replace a BDR node after a short promotion procedure**. It warns that physical standby nodes have worse operational characteristics than logical standby nodes, and that replication slot propagation to the standby must be verified before relying on failover.

### 3.3 PGCon active-active talk (AWS / Fujitsu)

The clearest architectural statement comes from the PGCon presentation on pgactive:

> "Active-active deployment model • One or more primaries (active) that replicate between each other • **Can also include standbys, but not in 'high availability set'**"

This is the design principle the hybrid model must respect: the physical standby is a **failover target and read-offload node**, not a voting member of the active-active group.

### 3.4 What is *not* prior art

The specific pattern of **flipping the same two nodes between active-active and active-passive via a configuration folder swap** is not documented as a single procedure in any vendor guide. The building blocks are documented; the orchestration is what this design contributes.

### 3.5 Feasibility verdict

| Requirement | Status |
|---|---|
| pgactive supports detach and remove of nodes | ✅ Documented |
| pgactive supports pausing logical apply | ✅ Documented |
| Physical standby of a pgactive node is supported | ✅ Documented |
| pg_rewind works with a pgactive primary as source | ✅ Standard PostgreSQL behaviour |
| Sequence parity can be re-applied programmatically | ✅ Demonstrated in tbNAC build |
| Wrapper script can orchestrate the sequence | ✅ Design contributed here |
| **Single command with zero downtime** | ❌ Not achievable — write freeze required |
| **Automatic failover during the switch** | ❌ Not achievable without a witness |

**Conclusion:** The pattern is achievable as an **operator-initiated, scripted, mode-switching procedure** with a short write freeze. It is not achievable as a zero-downtime runtime toggle.

---

## 4. Architecture Overview

### 4.1 Mode A — Active-Active

```
   ┌───────────────────────────┐          ┌───────────────────────────┐
   │          NODE 1           │          │          NODE 2           │
   │  Forge · 192.168.0.28     │          │  Jigar-6 · 192.168.0.209  │
   │                           │          │                           │
   │  ┌─────────────────────┐  │  pgactive│  ┌─────────────────────┐  │
   │  │ PostgreSQL 17       │  │  logical │  │ PostgreSQL 17       │  │
   │  │ + pgactive 2.1.7    │◀─┼─────────▶│  │ + pgactive 2.1.7    │  │
   │  │ READ + WRITE :5434  │  │  stream  │  │ READ + WRITE :5434  │  │
   │  └─────────────────────┘  │          │  └─────────────────────┘  │
   │                           │          │                           │
   │  host networking          │          │  host networking          │
   └───────────────────────────┘          └───────────────────────────┘

   Sequence discipline:
     Node 1 → odd ids   (INCREMENT BY 2, offset 1)
     Node 2 → even ids  (INCREMENT BY 2, offset 2)
```

**Properties:**
- Both nodes writable. No single point of failure for writes.
- DDL applied manually on each node.
- Sequence collision prevented by odd/even parity.
- No automatic failover; no quorum witness; split-brain possible if both promoted.

### 4.2 Mode B — Active-Passive

```
   ┌───────────────────────────┐          ┌───────────────────────────┐
   │          NODE 1           │          │          NODE 2           │
   │  Forge · 192.168.0.28     │          │  Jigar-6 · 192.168.0.209  │
   │                           │          │                           │
   │  ┌─────────────────────┐  │ physical │  ┌─────────────────────┐  │
   │  │ PostgreSQL 17       │──┼─ WAL ───▶│  │ PostgreSQL 17       │  │
   │  │ PRIMARY             │  │  stream  │  │ STANDBY             │  │
   │  │ READ + WRITE :5434  │  │          │  │ READ-ONLY :5434     │  │
   │  └─────────────────────┘  │          │  └─────────────────────┘  │
   │                           │          │                           │
   │  host networking          │          │  host networking          │
   └───────────────────────────┘          └───────────────────────────┘

   Sequence discipline:
     Node 1 (only writer) → sequential ids, INCREMENT BY 1
     Node 2 (standby)     → read-only, sequences follow via WAL
```

**Properties:**
- Single writer. Standby read-only.
- DDL automatic via WAL.
- No sequence collision possible.
- Manual promotion still required for primary failure — same limitation as File 1.
- Standby is not in any pgactive group; no extension required on standby.

### 4.3 The Mode Switch Control Plane

```
                         ┌─────────────────────────┐
                         │   OPERATOR COMMAND      │
                         │   ./switch.sh <mode>    │
                         └────────────┬────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │   PRE-FLIGHT CHECKS     │
                         │   · both nodes reachable│
                         │   · replication lag = 0 │
                         │   · no open conflicts   │
                         │   · schema identical    │
                         └────────────┬────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                 FAIL               PASS               FAIL
                    │                 │                 │
                    ▼                 ▼                 ▼
              ┌──────────┐   ┌──────────────┐   ┌──────────┐
              │ STOP     │   │ QUIESCE      │   │ ROLLBACK │
              │ no change│   │ writes       │   │ to old   │
              └──────────┘   └──────┬───────┘   │ config   │
                                     │           └──────────┘
                              ┌──────▼───────┐
                              │ CONVERT      │
                              │ topology     │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ VERIFY       │
                              │ new mode     │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ UNFREEZE     │
                              │ writes       │
                              └──────────────┘
```

---

## 5. Why This Is Not a Native Toggle

A reasonable first question is: why not a single PostgreSQL parameter? The answer is that active-active and active-passive are not two settings of the same engine — they are **two different replication engines**:

| Property | Active-Active (`pgactive`) | Active-Passive (Streaming) |
|---|---|---|
| Replication type | Logical (row-level events) | Physical (raw WAL bytes) |
| Writable nodes | Both | Primary only |
| Standby signal | None — node is a normal primary | `standby.signal` must exist |
| Peer connection stored in | FDW servers (`pg_foreign_server`) | `primary_conninfo` in `postgresql.auto.conf` |
| DDL replication | Manual per node | Automatic via WAL |
| Sequence discipline | Odd/even parity required | Single writer, no parity needed |
| Promotion semantics | `pgactive_join_group()` | `pg_ctl promote` |
| Demotion semantics | `pgactive_detach_nodes()` + `pgactive_remove()` | `pg_rewind` + `standby.signal` |

Flipping between the two requires **tearing down one topology and building the other** on the same data directories. No `ALTER SYSTEM` command performs that atomically, and `pgactive` does not expose a "convert to physical standby" function. The switch is a **sequence of operations**, which is why a wrapper script is the right abstraction: it makes the sequence reliable and auditable, but the underlying work is still a multi-step conversion.

---

## 6. The Two-Folder Configuration Model

The core design is a **declarative configuration folder per mode**, with **shared data volumes**.

```
~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30/
│
├── switch.sh                          ← the single instruction
├── switch-state.json                  ← records current mode
│
├── active-active/                     ← MODE A configuration
│   ├── docker-compose.yml             (both nodes, pgactive preload)
│   ├── postgresql.conf                (wal_level=logical, pgactive)
│   └── pg_hba.conf                    (replication + app access)
│
├── active-passive/                    ← MODE B configuration
│   ├── docker-compose.yml             (primary and standby blocks)
│   ├── postgresql.conf                (hot_standby, primary_conninfo)
│   └── pg_hba.conf                    (replication from primary)
│
└── shared state (NOT duplicated)
    ├── volume: tbnac_node1_data       ← same disk, both modes
    └── volume: tbnac_node2_data       ← same disk, both modes
```

**Why this works:**

- The two folders are **version-controlled text files**. There is no hidden runtime state that has to be re-created by hand.
- The data volumes are **named Docker volumes** and persist across `docker compose down`. They are not removed when switching modes.
- The `switch.sh` script copies the selected folder's files into the working directory and runs `docker compose up -d` there. The data volumes are referenced by name, so they are reused.
- The current mode is recorded in `switch-state.json` so a restart after a switch does not accidentally bring up the wrong topology.

**What the folders differ in:**

| File | Active-Active | Active-Passive |
|---|---|---|
| `docker-compose.yml` | Both nodes with `pgactive` image | Primary + standby blocks; standby no `pgactive` |
| `postgresql.conf` | `shared_preload_libraries='pgactive'`, `output_plugin_libraries`, `wal_level=logical` | `hot_standby=on`, `primary_conninfo`, no `pgactive` preload |
| `pg_hba.conf` | Full app + replication rules | Replication from primary only; app read-only rules |

**What the folders do *not* differ in:**

- The port (`5434`), the database (`tbnac_core`), the user (`tbnac_core_user`), the schema (`nac`), the hosts and IPs, and the data volumes are **identical in both folders**. Only replication configuration changes.

---

## 7. The Mode Switch State Machine

The switch is a state machine with **eight states** and **two terminal outcomes** (complete or rollback). The wrapper script implements this state machine.

```
┌──────────────┐
│ 0. IDLE      │  current mode recorded in switch-state.json
└──────┬───────┘
       │ operator issues ./switch.sh <target-mode>
       ▼
┌──────────────┐
│ 1. PRE-FLIGHT│  verify reachability, lag=0, no conflicts,
│              │  schema parity, target mode ≠ current mode
└──────┬───────┘
       │ pass
       ▼
┌──────────────┐
│ 2. QUIESCE   │  stop app containers or set read-only;
│              │  wait for active transactions to drain
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 3. DETACH    │  pgactive_apply_pause, detach nodes,
│              │  pgactive_remove (for AA→AP)
│              │  OR pg_ctl promote (for AP→AA)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 4. RECONCILE │  pg_rewind (AA→AP) or reinstall extension
│              │  (AP→AA); re-apply sequence parity
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 5. SWAP      │  copy target folder config into working dir
│              │  update switch-state.json
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 6. START     │  docker compose up -d
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 7. VERIFY    │  new mode health checks
│              │  (pg_stat_replication or pgactive_nodes)
└──────┬───────┘
       │
   ┌───┴───┐
   │       │
 PASS    FAIL
   │       │
   ▼       ▼
┌──────┐ ┌──────────┐
│8.UN- │ │ ROLLBACK │
│FREEZE│ │ swap back│
└──────┘ │ restart  │
         └──────────┘
```

**Pre-flight refusal is a design principle.** The script must not begin a conversion it cannot finish. If any pre-flight check fails, the script exits without touching the cluster.

**Rollback is limited.** If verification fails, the script swaps back to the previous config folder and restarts. This is not always clean — a partial `pg_rewind` or a half-formed `pgactive` group may need manual cleanup. The runbook (§18) documents the recovery for each failure point.

---

## 8. Direction 1 — Active-Active to Active-Passive

**Goal:** Node 1 remains the writer; Node 2 becomes a physical streaming standby of Node 1.

**Precondition:** cluster is healthy in active-active mode, both nodes `r` in `pgactive_nodes`, zero lag on both slots.

### 8.1 Quiesce writes

Stop the application tier on both nodes to prevent new writes:

```bash
# Both nodes
docker compose stop tbnac-core-backend tbnac-worker
```

Alternatively, set the database read-only:

```sql
ALTER DATABASE tbnac_core SET default_transaction_read_only = on;
```

Wait for all active transactions to complete:

```sql
SELECT count(*) FROM pg_stat_activity 
WHERE state = 'active' AND backend_type = 'client backend';
-- Must return 0 before proceeding
```

### 8.2 Drain Node 2's writes to Node 1

Before detaching Node 2, confirm that all of Node 2's local writes have replicated to Node 1. On Node 1, inspect the slot for Node 2:

```sql
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes
FROM pg_replication_slots
WHERE slot_name LIKE 'pgactive%';
```

Wait until `lag_bytes` is **0** for the slot corresponding to Node 2. This is the critical step that prevents data loss on detach.

### 8.3 Pause and detach the pgactive group

On **Node 2** (the node becoming the standby):

```sql
-- Stop applying logical changes locally
SELECT pgactive.pgactive_apply_pause();

-- Detach from the group
SELECT pgactive.pgactive_detach_nodes(ARRAY['tbnac-node1']);

-- Remove pgactive state from this node
SELECT pgactive.pgactive_remove();
```

On **Node 1**, detach Node 2 from the group:

```sql
SELECT pgactive.pgactive_detach_nodes(ARRAY['tbnac-node2']);
```

On **Node 1**, create the physical replication slot that Node 2 will use as a standby:

```sql
SELECT pg_create_physical_replication_slot('standby_node2_slot');
```

### 8.4 Stop Node 2

```bash
# On Node 2
docker compose stop postgres
```

Confirm no `postmaster.pid` remains in the data directory:

```bash
docker run --rm -v tbnac_node2_data:/data alpine ls /data/postmaster.pid
# Should report "No such file"
```

### 8.5 Reconcile Node 2's data directory to Node 1's timeline

Node 2 was a primary with its own timeline. To become a standby of Node 1, it must be rewound to Node 1's timeline branch.

```bash
# On Node 2
docker run --rm \
  -v tbnac_node2_data:/var/lib/postgresql/data \
  --network host \
  pgactive-postgres:17 \
  pg_rewind \
    --target-pgdata=/var/lib/postgresql/data \
    --source-server="host=192.168.0.28 port=5434 user=tbnac_core_user dbname=tbnac_core" \
    --write-recovery-conf \
    --progress
```

`--write-recovery-conf` creates `standby.signal` and appends `primary_conninfo` to `postgresql.auto.conf`. The `primary_conninfo` will point at Node 1 and use `standby_node2_slot`.

**If `pg_rewind` cannot run** (e.g., `wal_log_hints` was off and checksums were not enabled), fall back to a full base backup:

```bash
docker run --rm \
  -v tbnac_node2_data:/var/lib/postgresql/data \
  --network host \
  -e PGPASSWORD=tbnac_core_pass \
  pgactive-postgres:17 \
  pg_basebackup \
    -h 192.168.0.28 -p 5434 -U tbnac_core_user \
    -D /var/lib/postgresql/data \
    -Fp -Xs -R -P -v \
    -S standby_node2_slot
```

Before this fallback, **archive Node 2's data directory** in case the rewind fails and the divergent state is needed for forensics.

### 8.6 Swap to the active-passive configuration folder

```bash
# On both nodes
rm -f docker-compose.yml postgresql.conf pg_hba.conf
cp active-passive/docker-compose.yml .
cp active-passive/postgresql.conf .
cp active-passive/pg_hba.conf .
```

Update `switch-state.json`:

```json
{
  "mode": "active-passive",
  "primary": "node1",
  "standby": "node2",
  "last_switch": "2026-10-20T15:00:00Z"
}
```

### 8.7 Start Node 2 as standby

```bash
# On Node 2
docker compose up -d postgres
docker compose logs -f postgres
```

Look for:

```
entering standby mode
started streaming WAL from primary at ... on timeline ...
database system is ready to accept read-only connections
```

### 8.8 Verify

On **Node 1** (primary):

```sql
SELECT client_addr, state, sync_state, replay_lag
FROM pg_stat_replication;
-- Expected: 192.168.0.209 / streaming / async / small lag
```

On **Node 2** (standby):

```sql
SELECT pg_is_in_recovery();   -- t
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;   -- small
SELECT timeline_id FROM pg_control_checkpoint();   -- matches Node 1
```

### 8.9 Unfreeze writes

```bash
# On both nodes
docker compose start tbnac-core-backend tbnac-worker
```

Or:

```sql
ALTER DATABASE tbnac_core SET default_transaction_read_only = off;
```

**Switch complete.**

---

## 9. Direction 2 — Active-Passive to Active-Active

**Goal:** Node 2 (the standby) becomes a writable primary; both nodes rejoin the pgactive group.

**Precondition:** cluster is healthy in active-passive mode, Node 2 streaming from Node 1, both nodes on the same timeline.

### 9.1 Quiesce writes

Same as §8.1 — stop the application tier on both nodes.

### 9.2 Promote Node 2

```bash
# On Node 2
docker exec -u postgres -i tbnac-postgres \
  pg_ctl promote -D /var/lib/postgresql/data
```

Look for:

```
waiting for server to promote.... done
server promoted
```

Verify:

```sql
SELECT pg_is_in_recovery();   -- f
SELECT timeline_id FROM pg_control_checkpoint();   -- incremented (e.g. 2 or 3)
```

### 9.3 Verify schema parity

DDL was not replicated during active-active mode. If any schema changes were applied during the active-passive phase, they must have been applied to **both** nodes before promotion. Verify now:

```bash
# On both nodes
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "\dt nac.*"
```

Both must show the **same 39 tables with the same columns**. If not, the switch cannot proceed — apply the missing DDL to the deficient node first.

### 9.4 Reinstall pgactive on both nodes

The active-passive folder does not carry `pgactive` configuration, and `pgactive_remove()` was called during the previous switch. Reinstall:

**On both nodes**, edit `postgresql.conf` in the `active-active/` folder to include:

```conf
shared_preload_libraries = 'pgactive'
output_plugin_libraries = 'pgoutput, test_decoding, pgactive'
wal_level = logical
max_worker_processes = 64
max_replication_slots = 32
max_wal_senders = 32
max_logical_replication_workers = 32
max_sync_workers_per_subscription = 8
pgactive.max_nodes = 16
track_commit_timestamp = on
```

Restart both containers:

```bash
docker compose restart postgres
```

Then, **on both nodes**, install the extension and FDW servers:

```sql
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
```

### 9.5 Create the group on Node 1, join Node 2

On **Node 1**:

```sql
SELECT pgactive.pgactive_create_group(
    node_name := 'tbnac-node1',
    node_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint1'
);
SELECT pgactive.pgactive_wait_for_node_ready();
```

On **Node 2**:

```sql
SELECT pgactive.pgactive_join_group(
    node_name := 'tbnac-node2',
    node_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint2',
    join_using_dsn := 'user_mapping=tbnac_core_user pgactive_foreign_server=pgactive_server_endpoint1'
);
SELECT pgactive.pgactive_wait_for_node_ready();
```

### 9.6 Re-apply sequence parity

During active-passive mode, Node 1 was the only writer. Node 2's sequences were replicated via WAL, so they are **identical to Node 1's** at the point of promotion. If both nodes now write independently, they will generate the same next IDs → duplicate key errors.

Apply parity using the tbNAC `pg_sequences` script (File 4 §9.4):

**On Node 1 (odd):**

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
        -- Node 1 = odd
        target := CASE WHEN cur % 2 = 1 THEN cur + 2 ELSE cur + 1 END;
        EXECUTE format('SELECT setval(%L, %s, false)',
                       seq.schemaname || '.' || seq.sequencename, target);
    END LOOP;
END $$;
```

**On Node 2 (even)** — identical except the parity line:

```sql
target := CASE WHEN cur % 2 = 0 THEN cur + 2 ELSE cur + 1 END;
```

### 9.7 Swap to the active-active configuration folder

```bash
# On both nodes
rm -f docker-compose.yml postgresql.conf pg_hba.conf
cp active-active/docker-compose.yml .
cp active-active/postgresql.conf .
cp active-active/pg_hba.conf .
```

Update `switch-state.json`:

```json
{
  "mode": "active-active",
  "nodes": ["node1", "node2"],
  "last_switch": "2026-10-20T18:00:00Z"
}
```

### 9.8 Verify

On either node:

```sql
SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;
-- Expected: tbnac-node1 | r, tbnac-node2 | r
```

Check conflict history:

```sql
SELECT count(*) FROM pgactive.pgactive_conflict_history;
-- Expected: 0 (or only the benign pgactive_nodes entry)
```

### 9.9 Unfreeze writes

```bash
docker compose start tbnac-core-backend tbnac-worker
```

**Switch complete.**

---

## 10. The Four Gaps and How They Are Closed

The user's proposed mechanism — "save data, stop, swap folder, start" — works for the trivial case (restarting the same topology) but fails in four specific places. Each gap has a documented closure:

### Gap 1 — pgactive state survives a container restart

**Problem:** A node that has run `pgactive` carries FDW servers, logical replication slots, and catalog entries. Starting it as a physical standby without cleanup leaves stale state that will cause `pgactive_join_group()` to fail on the return switch.

**Closure:** Explicit `pgactive.pgactive_detach_nodes()` and `pgactive.pgactive_remove()` on the node being converted (steps 8.3 and 9.4).

### Gap 2 — Timeline divergence blocks streaming replication

**Problem:** Both active-active nodes are primaries with their own timelines. Converting one into a standby of the other requires reconciling the timeline branch.

**Closure:** `pg_rewind --write-recovery-conf` on the target node (step 8.5), with `pg_basebackup -R` as fallback if rewind is impossible.

### Gap 3 — Sequence parity is lost on the standby and must be restored

**Problem:** Physical replication carries sequence increments via WAL, so the standby inherits the primary's sequence values. When it becomes a writer again, both nodes generate the same IDs.

**Closure:** Re-run the `pg_sequences` parity script on both nodes after promotion (step 9.6).

### Gap 4 — The standby needs `standby.signal` and `primary_conninfo`

**Problem:** An active-active data directory has no `standby.signal`. Without it, PostgreSQL starts as a primary, not a standby.

**Closure:** `pg_rewind --write-recovery-conf` creates both. If `pg_rewind` is not used, `pg_basebackup -R` does. In either case, the active-passive `postgresql.conf` must **not** override the automatically written `primary_conninfo`.

---

## 11. Sequence Offset Discipline Across Modes

The sequence state is **one of the most consequential things that changes across the switch**, and it is worth treating explicitly.

| Phase | Node 1 sequences | Node 2 sequences | INCREMENT BY |
|---|---|---|---|
| Active-Active steady state | Odd ids | Even ids | 2 (both) |
| After AA→AP switch | Unchanged (still odd) | Follow Node 1 via WAL | 2 (Node 1 only writes) |
| During AP | Node 1 writes | Read-only | 2 (Node 1) |
| After AP→AA switch | Odd | Even | 2 (both) |

**Option A (recommended):** Leave Node 1's sequences at `INCREMENT BY 2` throughout. This produces odd ids in active-passive mode, which is harmless — the single writer simply skips even ids. No change is required at the AA→AP boundary.

**Option B (optional):** When switching AA→AP, reset Node 1's sequences to `INCREMENT BY 1` for a clean sequential id space. This requires re-running the parity script on the return switch to restore `INCREMENT BY 2`. It is cleaner but adds two more script steps.

**The wrapper script should default to Option A**, with Option B available as a `--reset-sequences` flag. Option A minimizes the number of changes per switch and reduces the failure surface.

**Critical:** On the return switch (AP→AA), the parity script **must** be run before writes are unfrozen. If writes start before parity is applied, both nodes will produce the same ids and duplicate key errors will occur. The wrapper must enforce this ordering.

---

## 12. The Wrapper Script

The script is the "single instruction." Its value is **reliability and auditability**, not magic. It does not make the switch instantaneous — it makes it **repeatable and verifiable**.

### 12.1 Interface

```bash
./switch.sh active-passive [--primary=node1|node2] [--reset-sequences] [--dry-run]
./switch.sh active-active  [--primary=node1]        [--dry-run]
./switch.sh status
./switch.sh verify
```

### 12.2 Pre-flight checks (enforced before any change)

| Check | Failure action |
|---|---|
| `switch-state.json` mode ≠ target mode | Exit with message "already in target mode" |
| Both nodes reachable on port 5434 | Exit |
| `pg_stat_replication` lag < 5 MB (AA→AP) | Exit |
| `pgactive_nodes` both `r` (AA→AP) | Exit |
| `pg_is_in_recovery() = f` on primary, `t` on standby (AP→AA) | Exit |
| `pgactive_conflict_history` has no unresolved rows | Exit |
| Schema parity: `\dt nac.*` identical on both nodes | Exit |
| `docker compose ps` shows all app containers healthy | Exit |
| Sufficient disk space on both data volumes | Exit |

### 12.3 Post-conversion checks (before unfreeze)

| Check | Failure action |
|---|---|
| `pg_stat_replication` shows standby (AP mode) | Rollback |
| `pg_is_in_recovery() = t` on standby (AP mode) | Rollback |
| `pgactive_nodes` shows two `r` rows (AA mode) | Rollback |
| `pgactive_conflict_history` empty (AA mode) | Rollback |
| Sequence parity correct on both nodes (AA mode) | Rollback |
| A sample write replicates in < 2s | Rollback |

### 12.4 Rollback behaviour

Rollback is **best-effort**, not guaranteed. It consists of:

1. Stopping the new mode's containers.
2. Copying the previous config folder back.
3. Restarting.
4. Verifying the previous mode's health checks.

If the previous mode cannot be restored (e.g., a partial `pg_rewind` left the data directory in an inconsistent state), the script must **stop and report** rather than attempting further automated recovery. The runbook (§18) documents manual recovery for this case.

### 12.5 Dry-run mode

`--dry-run` executes pre-flight checks and prints the planned sequence of commands **without executing any state-changing step**. This is the mode used in the quarterly rehearsal, so the team can verify the procedure against the current environment without risking the cluster.

### 12.6 State file

```json
{
  "mode": "active-active",
  "primary": null,
  "standby": null,
  "nodes": ["node1", "node2"],
  "last_switch": "2026-10-20T18:00:00Z",
  "last_switch_duration_seconds": 47,
  "last_switch_result": "success"
}
```

The file is written **only after** verification passes. A failed switch leaves the previous value intact, so a subsequent `./switch.sh status` reports the actual mode of the cluster.

---

## 13. Repository Layout

The complete proposed repository:

```
~/tbNAC_NM-UBUNTU-3.1.2-2026-09-30/
│
├── switch.sh                                  ← the operator command
├── switch-state.json                          ← current mode record
├── SWITCH-RUNBOOK.md                          ← link to §18 of this document
│
├── active-active/
│   ├── docker-compose.yml                     ← both nodes, pgactive image
│   ├── postgresql.conf                        ← pgactive preload, output plugins
│   └── pg_hba.conf                            ← replication + app access
│
├── active-passive/
│   ├── docker-compose.yml                     ← primary + standby blocks
│   ├── postgresql.conf                        ← hot_standby, primary_conninfo
│   └── pg_hba.conf                            ← replication from primary
│
├── Dockerfile                                 ← pgactive image (File 4 §5)
│
└── shared volumes (Docker-managed, not in repo)
    ├── tbnac_node1_data/
    └── tbnac_node2_data/
```

---

## 14. What Changes and What Stays the Same

| Element | Active-Active | Active-Passive | Changes on switch? |
|---|---|---|---|
| Physical hosts | Forge / Jigar-6 | Forge / Jigar-6 | ❌ No |
| IP addresses | .28 / .209 | .28 / .209 | ❌ No |
| Port | 5434 | 5434 | ❌ No |
| Database | tbnac_core | tbnac_core | ❌ No |
| Schema | nac (39 tables) | nac (39 tables) | ❌ No |
| Data volumes | tbnac_node1_data, tbnac_node2_data | Same | ❌ No — reused |
| `shared_preload_libraries` | `pgactive` | (none) | ✅ Yes |
| `output_plugin_libraries` | `pgactive` listed | (n/a) | ✅ Yes |
| `hot_standby` | (n/a) | `on` on standby | ✅ Yes |
| `primary_conninfo` | (n/a) | On standby | ✅ Yes |
| `standby.signal` | Absent | Present on standby | ✅ Yes |
| FDW servers | Present | Absent | ✅ Yes |
| Node 2 writability | Read + Write | Read-only | ✅ Yes |
| Sequence INCREMENT BY | 2 (both) | 2 (primary only) | ⚠️ Depends on option |
| DDL propagation | Manual per node | Automatic | ✅ Yes |
| **Write availability** | Continuous | Short freeze during switch | ⚠️ Planned |

---

## 15. Write Quiescence Strategy

The switch cannot run while writes are in flight. The strategy must drain **all** writes to the primary (or to the group, for AA→AP) before any topology change begins.

### 15.1 Recommended: stop the application tier

For tbNAC, the application tier is `tbnac-core-backend` and `tbnac-worker` on each node. Stopping these containers guarantees no new writes and allows in-flight transactions to drain naturally:

```bash
# Both nodes
docker compose stop tbnac-core-backend tbnac-worker
```

Wait for the write transaction count to reach zero:

```sql
SELECT count(*) FROM pg_stat_activity
WHERE state = 'active' AND backend_type = 'client backend';
-- Must be 0
```

### 15.2 Alternative: database read-only mode

If the application tier cannot be stopped (e.g., read-heavy but write-rejecting workloads must stay online), set the database read-only:

```sql
ALTER DATABASE tbnac_core SET default_transaction_read_only = on;
SELECT pg_reload_conf();
```

New sessions will be read-only. Existing write transactions must complete before proceeding.

### 15.3 For AA→AP: drain the peer's writes

Before detaching Node 2, verify that its slot on Node 1 has zero lag:

```sql
-- On Node 1
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes
FROM pg_replication_slots
WHERE slot_name LIKE 'pgactive%';
```

Wait until `lag_bytes = 0` for Node 2's slot. This guarantees that Node 2's local writes have been applied to Node 1 before Node 2 becomes a standby. Without this, writes made on Node 2 shortly before the switch would be discarded when its data directory is rewound.

### 15.4 Unfreeze

```bash
# Both nodes
docker compose start tbnac-core-backend tbnac-worker
```

Or:

```sql
ALTER DATABASE tbnac_core SET default_transaction_read_only = off;
SELECT pg_reload_conf();
```

---

## 16. Test Plan

The switch procedure must be tested end-to-end on the lab environment **before** being run on production. The test plan is structured around the **round trip** — not just the individual directions.

### 16.1 Phase 1 — Baseline (Mode A healthy)

| ID | Test | Expected | Status |
|---|---|---|---|
| BL-01 | Both nodes `r` in `pgactive_nodes` | 2 rows, `r` | ⏳ |
| BL-02 | Bidirectional write works | Insert on Node 1 visible on Node 2 within 2s | ⏳ |
| BL-03 | Zero conflicts | `pgactive_conflict_history` empty | ⏳ |
| BL-04 | Sequence parity | Node 1 odd, Node 2 even | ⏳ |
| BL-05 | `switch-state.json` mode | `active-active` | ⏳ |

### 16.2 Phase 2 — AA → AP conversion

| ID | Test | Expected | Status |
|---|---|---|---|
| AP-01 | Pre-flight passes | All checks green | ⏳ |
| AP-02 | Write freeze applied | `pg_stat_activity` write count = 0 | ⏳ |
| AP-03 | Node 2 slot lag = 0 on Node 1 | Confirmed | ⏳ |
| AP-04 | `pgactive_remove()` succeeds on Node 2 | No error | ⏳ |
| AP-05 | `pg_rewind` succeeds (or fallback base backup) | Exit code 0 | ⏳ |
| AP-06 | Node 2 starts as standby | `pg_is_in_recovery() = t` | ⏳ |
| AP-07 | Streaming starts | `pg_stat_replication` shows Node 2 | ⏳ |
| AP-08 | Standby rejects writes | `ERROR: cannot execute INSERT in a read-only transaction` | ⏳ |
| AP-09 | Replication lag < 1 MB | Confirmed | ⏳ |
| AP-10 | `switch-state.json` mode | `active-passive` | ⏳ |
| AP-11 | **Switch duration** | Record actual seconds | ⏳ |

### 16.3 Phase 3 — Operate in AP mode

| ID | Test | Expected | Status |
|---|---|---|---|
| OP-01 | Write on Node 1 → visible on Node 2 | Confirmed | ⏳ |
| OP-02 | Standby rejects writes | Confirmed | ⏳ |
| OP-03 | DDL on Node 1 propagates to Node 2 | Confirmed | ⏳ |
| OP-04 | Transient restart of Node 2 → auto-recovery | `pg_stat_replication` re-establishes | ⏳ |
| OP-05 | Transient restart of Node 1 → Node 2 stays passive | `pg_is_in_recovery() = t` throughout | ⏳ |

### 16.4 Phase 4 — AP → AA conversion

| ID | Test | Expected | Status |
|---|---|---|---|
| AA-01 | Pre-flight passes | All checks green | ⏳ |
| AA-02 | `pg_ctl promote` on Node 2 | `f`, new timeline | ⏳ |
| AA-03 | Schema parity checked | Identical 39 tables | ⏳ |
| AA-04 | pgactive reinstalled on both nodes | `\dx pgactive` shows 2.1.7 | ⏳ |
| AA-05 | `pgactive_create_group` on Node 1 | Success | ⏳ |
| AA-06 | `pgactive_join_group` on Node 2 | Success | ⏳ |
| AA-07 | Sequence parity re-applied | Node 1 odd, Node 2 even | ⏳ |
| AA-08 | Both nodes `r` | 2 rows, `r` | ⏳ |
| AA-09 | Bidirectional write works | Insert on both visible on both | ⏳ |
| AA-10 | Zero conflicts | `pgactive_conflict_history` empty | ⏳ |
| AA-11 | **Round-trip duration** | Record actual seconds | ⏳ |

### 16.5 Phase 5 — Repeat

| ID | Test | Expected | Status |
|---|---|---|---|
| RP-01 | Run full AA→AP→AA round trip twice | Both succeed | ⏳ |
| RP-02 | Compare second round-trip duration to first | ≤ 20% increase | ⏳ |
| RP-03 | No residual pgactive state after the second AA→AP | `\dx` clean, no orphan slots | ⏳ |

The **round-trip test is the critical one**. A single-direction test can pass while leaving latent state that only surfaces on the return switch. Two full round trips with a state-cleanup check between them is the minimum evidence that the switch is safe to run in production.

---

## 17. Failure Handling During the Switch

The switch is the **most fragile moment** in the cluster's life. If a node fails mid-switch, the cluster can end up in a state that no automated rollback can recover. This section documents the failure modes and the required operator response.

### 17.1 Failure taxonomy

| Failure point | Symptom | Recovery |
|---|---|---|
| Pre-flight | Any check fails | Script exits; no change. Fix root cause, retry. |
| Quiesce | Application containers fail to stop | `docker stop -f`; proceed only if no write transaction is active. |
| Drain (AA→AP) | Node 2's slot lag never reaches 0 | Do **not** proceed. Node 2 has unreplicated writes. Investigate replication first. |
| Detach (AA→AP) | `pgactive_detach_nodes` error | Do not proceed. Restart Node 2 with `pgactive_apply_resume()` and retry. |
| `pg_rewind` | Exit code ≠ 0 | Archive Node 2's data directory; fall back to `pg_basebackup`. |
| Start standby | `standby.signal` missing or wrong conninfo | Check `pg_rewind --write-recovery-conf` output. Manual fix required. |
| Start standby | Streaming does not start | Check HBA, slot, `primary_conninfo`. Node 2 is a failed standby; **roll back to active-active** or **manually promote Node 2 to standalone primary**. |
| Verify (post) | Health checks fail | Rollback if possible; otherwise manual intervention per §18.4. |
| Promote (AP→AA) | `pg_ctl promote` hangs | Check recovery state; may need to wait for WAL replay. Do not kill. |
| Join (AP→AA) | `pgactive_join_group` fails | Check `pgactive_nodes` for stale `state=i`; manually delete and retry. |
| Sequence parity (AP→AA) | Script errors mid-loop | Re-run script (idempotent). **Do not unfreeze writes until it completes.** |

### 17.2 The unrecoverable case: node failure during the switch

If Node 1 fails **during** the AA→AP switch (while Node 2 is being rewound), the cluster has:
- Node 1 offline (the target primary).
- Node 2 mid-rewind (data directory in a partially reconciled state).
- No running writer.

Recovery requires:
1. Bring Node 1 back up.
2. Complete Node 2's rewind (or restart from base backup).
3. Re-run the switch procedure from step 8.3.

If Node 1's data directory is **lost** during the switch, the only data is on Node 2 (mid-rewind) — which may be corrupt. Recovery then requires restoring Node 2 from a backup taken before the switch. **This is why a fresh backup must be taken before any switch is attempted** (see §18.0).

### 17.3 Rollback is not symmetric

Rolling back an AA→AP switch that has completed is **not** simply "swap folder and restart" — Node 2 is now a standby with a different data directory. Rollback requires running the **AP→AA** procedure on the already-converted cluster, which is the same cost as completing the switch. The wrapper script should refuse `--rollback` after verification has passed; the correct action is `./switch.sh active-active`.

---

## 18. Operational Runbook

### 18.0 Pre-switch checklist

Before any switch:

1. **Take a full backup** of both nodes:
   ```bash
   docker exec -i tbnac-postgres pg_dump -U tbnac_core_user -d tbnac_core -p 5434 -Fc > pre-switch-$(date +%F-%H%M).dump
   ```
2. Record current state in `switch-state.json` (already there, but confirm).
3. Confirm no active incidents on either node.
4. Schedule a maintenance window — the switch is not online.
5. Notify application owners of the write freeze duration.

### 18.1 Daily health check (unchanged from File 4)

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;"
```

In AP mode, the equivalent is:

```bash
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"
```

### 18.2 Switch to active-passive

```bash
./switch.sh active-passive --dry-run     # rehearse
./switch.sh active-passive               # execute
```

Watch output for pre-flight pass, quiesce, detach, rewind, swap, verify, unfreeze. **Do not interrupt the script.**

If the script stops before verification, read the last error and consult §17.

If the script stops **after** verification but before unfreeze, the cluster is in the new mode — manually start the app tier:

```bash
docker compose start tbnac-core-backend tbnac-worker
```

### 18.3 Switch to active-active

```bash
./switch.sh active-active --dry-run
./switch.sh active-active
```

Same watching pattern.

### 18.4 Manual recovery — failed switch

If the wrapper cannot complete or roll back:

1. **Stop all containers on both nodes**:
   ```bash
   docker compose down
   ```
2. **Inspect `switch-state.json`** — it will show the last *successfully completed* mode, not the attempted one.
3. **Inspect both data directories** for `standby.signal` and `postmaster.pid`. Determine each node's actual role.
4. **Restore from pre-switch backup** if the data directories are inconsistent:
   ```bash
   docker run --rm -v tbnac_node1_data:/var/lib/postgresql/data \
     -v $(pwd):/backup alpine \
     sh -c 'rm -rf /var/lib/postgresql/data/* && tar xzf /backup/pre-switch-*.tar.gz -C /var/lib/postgresql/data'
   ```
5. **Restart in the last known good mode** by manually copying the corresponding folder's files.
6. **Document the incident** — cause, impact, data integrity, recovery time.

### 18.5 Quarterly rehearsal

Run `./switch.sh <target> --dry-run` once per quarter on a **lab replica** of the production environment. Do not run it on production without a scheduled window. The dry-run verifies that the pre-flight checks still pass against the current configuration — it does not exercise the state-changing steps.

### 18.6 Best practices (consolidated)

1. **Never unfreeze writes before verification passes.** This is the most important rule in the runbook.
2. **Always back up before switching.** The backup is the rollback path of last resort.
3. **Always drain the peer's writes before detaching.** A lost write is a lost write; the slot lag check is not optional.
4. **Always re-apply sequence parity on the AP→AA switch, before unfreeze.**
5. **Always run `--dry-run` first** when the environment has changed.
6. **Never run the switch without the wrapper.** Manual execution is error-prone and leaves no audit trail.
7. **Never assume rollback is free.** Once verification passes, the switch is complete; the reverse direction is a new switch.

---

## 19. Risks, Limitations and Non-Goals

### 19.1 Risks

| Risk | Mitigation |
|---|---|
| Write loss if drain step is skipped | Enforced in wrapper; pre-flight refuses to proceed if slot lag ≠ 0 |
| Data corruption if `pg_rewind` is interrupted | Pre-switch backup; archive divergent data on rewind failure |
| Duplicate key errors after AP→AA if parity is skipped | Wrapper enforces parity before unfreeze; idempotent script |
| Stale pgactive state causing join failure | Explicit `pgactive_remove()` + verification of `\dx` and `pg_replication_slots` |
| Split-brain during the switch | Not solved by this design — see §19.2 |
| Backup/restore window longer than expected | Test on lab first; measure actual durations |

### 19.2 Limitations

1. **The switch requires a write freeze.** It is not online. The duration depends on the size of the data directory (for `pg_basebackup`) and on the time required for the WAL replay (for `pg_rewind`). Both must be measured on the lab before being quoted to management.

2. **The switch does not provide automatic failover.** If the primary fails during the switch, recovery is manual.

3. **The switch does not solve split-brain.** The active-active pair still has no quorum witness. A network partition during active-active mode remains a manual-recovery scenario, as documented in File 2.

4. **The passive standby is not in the pgactive HA set.** This is consistent with the PGCon/Fujitsu guidance: physical standbys are failover targets, not group members.

5. **Rollback is best-effort, not guaranteed.** Once verification passes, the reverse direction is a new switch, not an undo.

### 19.3 Non-goals

- Zero-downtime mode switching.
- Automatic mode switching based on health signals.
- Multi-region active-active with more than two nodes.
- Automatic failover without a quorum witness.
- Replacing a proper HA manager (Patroni, repmgr, or CloudNativePG).

---

## 20. Appendices

### Appendix A — Complete command reference

**Pre-switch checks:**

```bash
# Cluster health (AA mode)
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT node_name, node_status FROM pgactive.pgactive_nodes ORDER BY 1;"

# Replication lag (both modes)
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes
FROM pg_replication_slots WHERE slot_name LIKE 'pgactive%' OR slot_name LIKE 'standby%';"

# Schema parity
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "\dt nac.*"

# Conflict history
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT conflict_id, local_conflict_time, object_name, error_sqlstate
FROM pgactive.pgactive_conflict_history
ORDER BY local_conflict_time DESC LIMIT 20;"
```

**Quiesce and drain:**

```bash
docker compose stop tbnac-core-backend tbnac-worker
docker exec -i tbnac-postgres psql -U tbnac_core_user -d tbnac_core -p 5434 -c "
SELECT count(*) FROM pg_stat_activity
WHERE state = 'active' AND backend_type = 'client backend';"
```

**AA → AP:**

```sql
-- Node 2
SELECT pgactive.pgactive_apply_pause();
SELECT pgactive.pgactive_detach_nodes(ARRAY['tbnac-node1']);
SELECT pgactive.pgactive_remove();

-- Node 1
SELECT pgactive.pgactive_detach_nodes(ARRAY['tbnac-node2']);
SELECT pg_create_physical_replication_slot('standby_node2_slot');
```

```bash
# Node 2
docker compose stop postgres
docker run --rm -v tbnac_node2_data:/var/lib/postgresql/data --network host \
  pgactive-postgres:17 pg_rewind \
    --target-pgdata=/var/lib/postgresql/data \
    --source-server="host=192.168.0.28 port=5434 user=tbnac_core_user dbname=tbnac_core" \
    --write-recovery-conf --progress
cp active-passive/* .
docker compose up -d postgres
```

**AP → AA:**

```bash
# Node 2
docker exec -u postgres -i tbnac-postgres pg_ctl promote -D /var/lib/postgresql/data
```

```sql
-- Both nodes
CREATE EXTENSION IF NOT EXISTS pgactive;
-- recreate FDW servers (see §9.4)

-- Node 1
SELECT pgactive.pgactive_create_group('tbnac-node1', 'user_mapping=... pgactive_foreign_server=...');
SELECT pgactive.pgactive_wait_for_node_ready();

-- Node 2
SELECT pgactive.pgactive_join_group('tbnac-node2', 'user_mapping=...', 'user_mapping=...');
SELECT pgactive.pgactive_wait_for_node_ready();

-- Both nodes: re-apply sequence parity (see §9.6)
```

```bash
cp active-active/* .
docker compose up -d
```

**Unfreeze:**

```bash
docker compose start tbnac-core-backend tbnac-worker
```

### Appendix B — Failure taxonomy for the switch

| Layer | Symptom | Where to look |
|---|---|---|
| Pre-flight | Any check fails | Wrapper output; `switch-state.json` |
| Drain | Slot lag never reaches 0 | `pg_replication_slots` on primary; check network, slot active |
| Detach | `pgactive_detach_nodes` error | `pgactive_nodes` table; slot state |
| Rewind | `pg_rewind` fails | Check `wal_log_hints` and `data_checksums`; use base backup fallback |
| Standby start | No streaming | `standby.signal`, `primary_conninfo`, HBA, slot |
| Verify | Health check fails | Mode-specific check in wrapper |
| Promote | `pg_ctl promote` hangs | `pg_is_in_recovery()`; WAL replay progress |
| Join | `pgactive_join_group` fails | `pgactive_nodes` for stale `state=i`; slot cleanup |
| Parity | Script error | Re-run (idempotent); do not unfreeze |

### Appendix C — Glossary

| Term | Definition |
|---|---|
| **Mode A / Active-Active** | Both nodes writable; changes propagate via pgactive |
| **Mode B / Active-Passive** | One writer, one read-only physical standby |
| **Two-folder model** | Declarative configuration per mode; same data volumes reused |
| **Mode switch** | Operator-initiated conversion from one mode to the other |
| **Write freeze** | Temporary suspension of application writes during the switch |
| **Drain** | Waiting for all writes to replicate before detaching a node |
| **`pgactive_apply_pause()`** | Pauses logical apply on the local node |
| **`pgactive_detach_nodes()`** | Removes a peer from the pgactive group |
| **`pgactive_remove()`** | Removes pgactive state from the local node |
| **`pg_rewind`** | Reconciles a diverged data directory with a new primary |
| **`pg_basebackup -R`** | Full base backup that creates `standby.signal` and `primary_conninfo` |
| **`standby.signal`** | File that tells PostgreSQL to start in recovery mode |
| **`primary_conninfo`** | Connection string a standby uses to reach its primary |
| **Sequence parity** | Odd ids on one node, even on the other; prevents duplicate key errors |
| **`setval(..., false)`** | Sets a sequence so the next `nextval` returns exactly the given value |
| **Pre-flight refusal** | Wrapper exits without changing state if checks fail |
| **Rollback** | Best-effort attempt to restore the previous mode after a failed switch |
| **Round trip** | Full AA→AP→AA sequence; the minimum test for safety |

### Appendix D — Prior art references

| Source | Contribution to this design |
|---|---|
| **Fujitsu Enterprise Postgres documentation** | Physical standby of a pgactive node; `synchronized_standby_slots`; `pgactive_apply_pause` / `pgactive_apply_resume` |
| **EDB BDR / PGD documentation** | Physical Standby Nodes of a BDR node; slot propagation verification; warning about worse operational characteristics |
| **PGCon active-active talk (AWS / Fujitsu)** | "Can also include standbys, but not in 'high availability set'" — confirms standbys are failover targets, not group members |
| **File 1 — Master-Standby Implementation Report** | Physical streaming standby setup; `pg_basebackup -R`; failover procedure; two-node limitation |
| **File 2 — Failure Scenarios Addendum** | Split-brain behaviour; manual recovery runbook; HA manager comparison |
| **File 3 — Multi-Master Bridge Report** | pgactive setup with bridge networking; Dockerfile artifact requirement |
| **File 4 — tbNAC Host-Networking Report** | Production environment; `pg_sequences` parity script; secondary unique constraint warnings; organic conflict behaviour |

### Appendix E — Open questions for the lab

1. **Actual `pg_rewind` duration** on tbNAC's 39-table, ~40K-row dataset. Must be measured before quoting switch time to management.
2. **Actual `pg_basebackup` duration** as the fallback path.
3. **Whether `wal_log_hints` is enabled** on the current build — `pg_rewind` requires either `wal_log_hints = on` or `data_checksums = on`. If neither is enabled, only `pg_basebackup` is available.
4. **Behaviour of the tbNAC application tier** during the write freeze — does the frontend queue writes, or does it error visibly?
5. **Behaviour of the `mac_vendors` seeding logic** after a round trip — will the "already seeded" check hold, or will one node attempt a re-seed?
6. **The secondary unique constraint warnings** (File 4 §12) — do they affect the AA→AP conversion? In AP mode, the standby is read-only, so the constraint is not a conflict target. The warning is relevant only in AA mode.
7. **Whether `synchronized_standby_slots` should be added to the active-active node** when a physical standby exists. This is required if the standby is to be **safely promotable** from a byte-for-byte standpoint. If omitted, a promotion of the standby may produce a divergent timeline. For the **mode switch** (standby → rejoin as AA member), `synchronized_standby_slots` is not strictly required because the node will rejoin the logical group, not be promoted in place. For **emergency failover** of the standby, it is required.

---

**Document End.**
