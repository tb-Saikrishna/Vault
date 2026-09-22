# Master-Standby Failure Scenarios — Test Addendum

# PostgreSQL Master-Standby Cluster — Failure Scenario Test Addendum

**Project:** Containerized PostgreSQL HA Cluster (Master-Standby)
**Environment:** Two-node setup on Ubuntu hosts with Docker Compose
**Nodes:**
- **Node 1** — hostname `Forge`, IP `192.168.0.28` (Primary)
- **Node 2** — hostname `test`, IP `192.168.0.159` (Standby)

**Document Date:** 2026-09-22
**Status:** All three failure scenarios tested and verified
**Companion to:** `PostgreSQL HA Cluster — Implementation Report`

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Test Environment](#2-test-environment)
3. [Scenario A — Standby Bounces](#3-scenario-a--standby-bounces)
4. [Scenario B — Primary Bounces](#4-scenario-b--primary-bounces)
5. [Scenario C — Split-Brain](#5-scenario-c--split-brain)
6. [What an HA Manager Would Do Differently](#6-what-an-ha-manager-would-do-differently)
7. [Comparison Matrix](#7-comparison-matrix)
8. [Architectural Conclusions](#8-architectural-conclusions)
9. [Split-Brain Recovery Runbook](#9-split-brain-recovery-runbook)
10. [Appendices](#10-appendices)

---

## 1. Executive Summary

Three failure scenarios were tested against the two-node master-standby cluster to characterize its behavior under real-world operational disruptions:

| # | Scenario | Data Loss | Manual Intervention | Outcome |
|---|----------|-----------|---------------------|---------|
| **A** | Standby bounces (stop/start Node 2) | ❌ None | ❌ None | Auto-recovery |
| **B** | Primary bounces (stop/start Node 1, no promotion) | ❌ None | ❌ None | Auto-recovery |
| **C** | Split-brain (partition, promote both, reconnect) | ✅ **Yes** | ✅ **Yes** | Manual recovery with data loss on loser |

**Headline conclusions:**

1. **Transient restarts (A & B) are transparent** — the cluster recovers automatically with no data loss. This covers ~90% of real-world operational disruptions (rolling reboots, package upgrades, container restarts).

2. **Network partitions (C) are catastrophic without an HA manager.** Both nodes accept writes independently, silently diverge, and do not reconcile when reconnected. Recovery requires manual intervention and results in **permanent data loss on the losing node**.

3. **The manual-failover design is appropriate for:**
   - Development and test environments
   - Single-region deployments with reliable networking
   - Deployments where manual promotion is acceptable (low RTO requirements)

4. **The design is NOT appropriate for:**
   - Production systems requiring high availability (auto-failover)
   - Multi-datacenter deployments where network partitions are possible
   - Deployments with strict data-loss tolerance

**Recommendation:** For production use, layer an HA manager (Patroni or repmgr) on top of this base — see [§6](#6-what-an-ha-manager-would-do-differently).

---

## 2. Test Environment

### 2.1 Cluster Topology

| Attribute | Node 1 | Node 2 |
|-----------|--------|--------|
| Hostname | `Forge` | `test` |
| IP | `192.168.0.28` | `192.168.0.159` |
| Role | Primary | Standby |
| Container | `postgres-primary` | `postgres-standby` |
| Host port | 5432 | 5433 |
| PostgreSQL | 16.15 (Alpine) | 16.15 (Alpine) |
| Replication type | Physical streaming (WAL) | Physical streaming (WAL) |
| Replication slot | `replication_slot` | — |
| Replication user | `replicator` / `replicator123` | — |
| Application user | `admin` / `admin123` | — |
| Test database | `mydb` | `mydb` |
| Test table | `dummy_ha` | `dummy_ha` |

### 2.2 Baseline State

**Before any scenario test:**

| Attribute | Node 1 | Node 2 |
|-----------|--------|--------|
| `pg_is_in_recovery()` | `f` | `t` |
| Timeline | 1 | 1 |
| `dummy_ha` row count | 7 | 7 |
| `pg_stat_replication` | 1 row (`192.168.0.159`) | — |

---

## 3. Scenario A — Standby Bounces

### 3.1 Test Description

**Question:** When the standby goes down and comes back up, does the primary keep accepting writes? Does the standby catch up automatically?

**Procedure:**
1. Capture pre-flight state.
2. Stop Node 2's container.
3. Observe the primary's view of replication.
4. Write to the primary while the standby is down.
5. Restart the standby.
6. Verify catch-up and resumption of live replication.

### 3.2 Pre-Flight State

**Node 1 (Primary):**
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.159 | streaming | async
(1 row)

 count
-------
     7
(1 row)
```

**Node 2 (Standby):**
```
 pg_is_in_recovery | t
 count             | 7
```

### 3.3 During the Outage

**Command executed on Node 2:**
```bash
docker compose stop postgres-standby
```

**Observation on Node 1 after ~10 seconds:**
```sql
SELECT client_addr, state FROM pg_stat_replication;
```
```
 client_addr | state
-------------+-------
(0 rows)
```

```sql
SELECT slot_name, active FROM pg_replication_slots;
```
```
    slot_name     | active
------------------+--------
 replication_slot | f
(1 row)
```

**Key findings:**
- The primary **immediately notices** the standby is gone — `pg_stat_replication` goes empty.
- The replication slot remains **reserved** but becomes **inactive**. WAL segments are retained for future catch-up.
- **The primary continues accepting writes without any delay or error.**

**Write during outage (on Node 1):**
```sql
INSERT INTO dummy_ha (name, value) VALUES ('while-standby-down', 500.00);
```
```
INSERT 0 1
```
Row count: `7 → 8`.

### 3.4 Standby Restart

**Command executed on Node 2:**
```bash
docker compose start postgres-standby
docker compose logs -f postgres-standby
```

**Log excerpt:**
```
Data directory already initialized; skipping base backup.

PostgreSQL Database directory appears to contain a database; Skipping initialization

LOG:  starting PostgreSQL 16.15 ...
LOG:  database system was shut down in recovery at 2026-09-22 05:34:34 GMT
LOG:  entering standby mode
LOG:  redo starts at 0/2000028
LOG:  consistent recovery state reached at 0/3029EF8
LOG:  database system is ready to accept read-only connections
LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```

**Key findings:**
- The standby **does not re-run `pg_basebackup`** — the entrypoint detects the existing data directory and skips the base backup.
- It **re-enters recovery mode** and immediately begins streaming WAL.
- Total downtime: under 30 seconds (docker stop + docker start).

### 3.5 Post-Restart Verification

**Node 1:**
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.159 | streaming | async
(1 row)
```

**Node 2:**
```
 count
-------
     8
(1 row)
```
The standby caught up, including the row written during the outage.

**Write-forward test (Node 1 → Node 2):**
```sql
-- Node 1
INSERT INTO dummy_ha (name, value) VALUES ('after-standby-return', 600.00);
INSERT 0 1

-- Node 2 (after 2 seconds)
SELECT count(*) FROM dummy_ha;
8 → 9
```

### 3.6 Scenario A — Results

| Assertion | Result |
|-----------|--------|
| Primary detects standby loss | ✅ `pg_stat_replication` becomes empty |
| Replication slot retained (inactive) | ✅ `active = f` |
| Primary continues accepting writes during outage | ✅ No interruption |
| Standby restart skips base backup | ✅ Uses existing data directory |
| Standby auto-reconnects and streams WAL | ✅ Streams from `0/3000000` |
| Standby catches up with zero data loss | ✅ Includes writes made during outage |
| Live replication resumes | ✅ Subsequent writes propagate in <2s |
| Manual intervention required | ❌ **None** |

**Verdict:** ✅ **Passed.** Standby restart is transparent to the application.

---

## 4. Scenario B — Primary Bounces

### 4.1 Test Description

**Question:** When the primary goes down and comes back up, does the standby do anything unexpected? Does the cluster recover automatically?

**Procedure:**
1. Capture pre-flight state.
2. Stop Node 1's container (no promotion).
3. Observe the standby's behavior — does it auto-promote?
4. Attempt a write on the standby (should fail — read-only).
5. Restart Node 1.
6. Verify both nodes recover and replication resumes.

### 4.2 Pre-Flight State

**Node 1 (Primary):**
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.159 | streaming | async
(1 row)

 count: 7
```

**Node 2 (Standby):**
```
 pg_is_in_recovery | t
 count             | 7
```

### 4.3 During the Outage

**Command executed on Node 1:**
```bash
docker compose stop postgres-primary
```

**Observation on Node 2 (after ~30 seconds):**
```
 pg_is_in_recovery
-------------------
 t
(1 row)
```

**Standby log (repeating every 5 seconds):**
```
FATAL:  could not connect to the primary server: connection to server at "192.168.0.28", port 5432 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
LOG:  waiting for WAL to become available at 0/302A6C8
```

**Key findings:**
- The standby **stays in recovery mode** (`t`).
- It does **NOT** auto-promote.
- It **retries every 5 seconds** in a connection loop.
- It **rejects all writes** while the primary is down.

**Write attempt on standby (Node 2):**
```sql
INSERT INTO dummy_ha (name, value) VALUES ('while-primary-down', 700.00);
```
```
ERROR:  cannot execute INSERT in a read-only transaction
```

**Key finding:** The standby **does not accept writes** while the primary is down. This is the critical safety property — it prevents accidental dual-primary.

### 4.4 Primary Restart

**Command executed on Node 1:**
```bash
docker compose start postgres-primary
docker compose logs postgres-primary --tail 30
```

**Log excerpt:**
```
PostgreSQL Database directory appears to contain a database; Skipping initialization

LOG:  starting PostgreSQL 16.15 ...
LOG:  database system was shut down at ...
LOG:  database system is ready to accept connections
```

**Key finding:** The primary **restarts from its existing data directory** — no re-initialization, no data loss.

**Primary's role after restart:**
```
 pg_is_in_recovery | f
 timeline_id       | 1
```

### 4.5 Standby Reconnection

**Observation on Node 2 (~30 seconds after primary restart):**
```
LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```

**Node 2 state:**
```
 pg_is_in_recovery | t
 count             | 7
```

### 4.6 Post-Recovery Verification

**Node 1:**
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.159 | streaming | async
(1 row)
```

**Write-forward test:**
```sql
-- Node 1
INSERT INTO dummy_ha (name, value) VALUES ('after-primary-return', 800.00);
INSERT 0 1

-- Node 2 (after 3 seconds)
SELECT count(*) FROM dummy_ha;
7 → 8
```

### 4.7 Scenario B — Results

| Assertion | Result |
|-----------|--------|
| Standby stays in recovery mode when primary down | ✅ `t` throughout |
| Standby does NOT auto-promote | ✅ No timeline change |
| Standby rejects writes | ✅ `read-only transaction` error |
| Standby logs connection retries every 5s | ✅ Confirmed |
| Primary restarts from existing data directory | ✅ No reinit |
| Primary retains timeline 1 | ✅ Confirmed |
| Standby auto-reconnects and streams | ✅ Within 30s |
| Writes resume and replicate normally | ✅ Confirmed |
| Manual intervention required | ❌ **None** |
| Data loss | ❌ **None** |

**Verdict:** ✅ **Passed.** Primary restart is transparent, and the standby's passive behavior prevents accidental dual-primary.

---

## 5. Scenario C — Split-Brain

### 5.1 Test Description

**Question:** What happens if the network between the nodes is partitioned, both nodes are promoted to primary, writes occur on both, and then the connection is re-established?

**Procedure:**
1. Capture pre-flight state.
2. Simulate a network partition via iptables.
3. Verify replication is broken but containers are alive.
4. Promote Node 2 (creating split-brain).
5. Write to both nodes independently.
6. Remove the partition.
7. Observe that no automatic reconciliation occurs.
8. Recover manually — choose a winner, wipe the loser, re-clone.

**⚠️ Warning:** This test **intentionally causes permanent data loss** on one node. It should only be performed in a test environment or with disposable data.

### 5.2 Pre-Flight State

**Node 1:**
```
 client_addr  |   state
--------------+-----------
 192.168.0.159 | streaming
(1 row)

 count: 8
 timeline_id: 1
```

**Node 2:**
```
 pg_is_in_recovery: t
 count: 8
 timeline_id: 1
```

### 5.3 Creating the Partition

**Command executed on Node 2:**
```bash
sudo iptables -I FORWARD -d 192.168.0.28 -j DROP
sudo iptables -I FORWARD -s 192.168.0.28 -j DROP
```

**Verify partition:**
```bash
docker exec -i postgres-standby psql -h 192.168.0.28 -p 5432 -U admin -c "SELECT 1;"
```
```
psql: error: connection to server at "192.168.0.28", port 5432 failed: Operation timed out
```

**Key finding:** The connection **times out** (not `refused`) — confirming a packet-drop partition, not a stopped service.

### 5.4 Standby Behavior During Partition

**Node 2 log (repeating):**
```
FATAL:  could not connect to the primary server: connection to server at "192.168.0.28", port 5432 failed: Operation timed out
LOG:  waiting for WAL to become available at 0/302ABD8
```

**Node 2 `pg_is_in_recovery`:**
```
 t
```

**Key finding:** The standby stays in recovery mode throughout the partition. **It does not auto-promote.**

### 5.5 Manual Promotion (Creating Split-Brain)

**Command executed on Node 2:**
```bash
docker exec -u postgres -i postgres-standby pg_ctl promote -D /var/lib/postgresql/data
```
```
waiting for server to promote.... done
server promoted
```

**Post-promotion state on Node 2:**
```
 pg_is_in_recovery | f
 timeline_id       | 2
```

### 5.6 Confirming Split-Brain

**Node 1 state (still primary, timeline 1):**
```
 pg_is_in_recovery | f
 timeline_id       | 1
 client_addr       | (0 rows — no standby visible)
```

**Node 2 state (now primary, timeline 2):**
```
 pg_is_in_recovery | f
 timeline_id       | 2
```

**Both nodes are now primaries.** ⚠️

### 5.7 Divergent Writes

**On Node 1:**
```sql
INSERT INTO dummy_ha (name, value) VALUES ('split-write-on-node1', 900.00);
INSERT 0 1
```

**On Node 2:**
```sql
INSERT INTO dummy_ha (name, value) VALUES ('split-write-on-node2', 950.00);
INSERT 0 1
```

**Both writes succeed.** Both nodes now have one row the other doesn't.

### 5.8 Divergence Demonstration

**Node 1 (timeline 1):**
```
 id |         name
----+----------------------
  1 | alpha
  ...
  8 | after-primary-return
  9 | split-write-on-node1
(9 rows)
```

**Node 2 (timeline 2):**
```
 id |         name
----+----------------------
  1 | alpha
  ...
  8 | after-primary-return
 41 | split-write-on-node2
(9 rows)
```

**Both nodes have 9 rows, but they are different rows.** Node 1 has `split-write-on-node1`, Node 2 has `split-write-on-node2`.

### 5.9 Removing the Partition

**Command executed on Node 2:**
```bash
sudo iptables -D FORWARD -d 192.168.0.28 -j DROP
sudo iptables -D FORWARD -s 192.168.0.28 -j DROP
```

### 5.10 Observing the Non-Reconciliation

**After 30 seconds, on Node 1:**
```sql
SELECT client_addr, state FROM pg_stat_replication;
```
```
 client_addr | state
-------------+-------
(0 rows)
```

**On Node 2:**
```
 pg_is_in_recovery
-------------------
 f
(1 row)
```

**Key findings:**
- Node 1 (timeline 1) does **not** see Node 2 (timeline 2) as a standby.
- Node 2 remains a primary — it does **not** auto-demote.
- **No automatic reconciliation occurs.**
- **The split-brain is now permanent until manual intervention.**

### 5.11 Recovery — Choosing a Winner

**Decision:** Node 2 wins (higher timeline: 2 > 1). This is the standard rule.

**Archive loser data (optional but recommended):**

Before wiping Node 1, dump its divergent state:
```bash
docker exec postgres-primary pg_dump -U admin -d mydb -Fc > node1-split-brain-backup.dump
```

**Stop and wipe Node 1:**
```bash
cd ~/pg-ha-cluster
docker compose down --remove-orphans
docker volume rm pg-ha-cluster_primary_data
```

**Rewrite Node 1 as a standby of Node 2:**

- New compose file: service renamed to `postgres-standby-node1`
- New entrypoint: `pg_basebackup -h 192.168.0.159 -p 5433`
- New configs: standby settings, `hot_standby = on`

**Restart Node 1:**
```bash
docker compose up -d
docker compose logs -f postgres-standby-node1
```

**Log excerpt:**
```
Data directory empty; preparing for base backup from Node 2...
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: write-ahead log start point: 0/4000028 on timeline 2
pg_basebackup: base backup completed
LOG:  entering standby mode
LOG:  starting backup recovery with redo LSN 0/4000028 ... on timeline ID 2
LOG:  database system is ready to accept read-only connections
LOG:  started streaming WAL from primary at 0/5000000 on timeline 2
```

### 5.12 Post-Recovery State

**Node 1 (rebuilt standby):**
```
 pg_is_in_recovery | t
 timeline_id       | 2
 row count         | 9
 split-write-on-node1 present? count = 0   ← DATA LOSS CONFIRMED
```

**Node 2 (winner, primary):**
```
 client_addr  |   state   | sync_state
--------------+-----------+------------
 192.168.0.28 | streaming | async
(1 row)

 timeline_id: 2
 row count: 9
 split-write-on-node2 present? YES
```

### 5.13 Scenario C — Results

| Assertion | Result |
|-----------|--------|
| Partition breaks replication | ✅ `Operation timed out` |
| Standby stays in recovery during partition | ✅ No auto-promote |
| Manual `pg_ctl promote` creates split-brain | ✅ Node 2 → timeline 2 |
| Both nodes accept writes independently | ✅ Divergent rows created |
| Data diverges | ✅ Confirmed |
| Reconnect does NOT auto-reconcile | ✅ No self-healing |
| Manual intervention required | ✅ Operator chooses winner |
| Data loss on loser | ✅ `split-write-on-node1` permanently lost |
| Recovery via re-clone | ✅ Node 1 rebuilt as standby of Node 2 |
| Cluster restored | ✅ Streaming replication on timeline 2 |

**Verdict:** ⚠️ **Split-brain is catastrophic without an HA manager.** Data loss is guaranteed on one side.

---

## 6. What an HA Manager Would Do Differently

The two-node manual setup we built is **safe by default, manual on failure**. An HA manager changes this to **automatic on failure, with its own trade-offs**. Here's a scenario-by-scenario comparison.

### 6.1 Scenario A — Standby Bounces

#### Current behavior (manual setup)

- Primary detects loss immediately.
- Primary continues accepting writes.
- Standby auto-reconnects on restart.
- No intervention required.
- **Recovery time: ~10–30 seconds.**

#### With Patroni

- Same outcome — the standby is not the leader, so its loss has no effect on the cluster.
- Patroni's DCS lock is held by the primary; the standby's absence doesn't trigger any election.
- Standby rejoins automatically via `pg_basebackup` or `pg_rewind` (if applicable).
- **Recovery time: ~10–30 seconds.**
- **Net change: essentially none.**

#### With repmgr

- Same outcome — the standby is not the leader, so its loss has no effect.
- `repmgrd` on the standby shuts down cleanly; no failover is triggered.
- Standby rejoins automatically via `repmgr standby clone`.
- **Recovery time: ~10–30 seconds.**
- **Net change: essentially none.**

#### Verdict

> **Both approaches handle this identically.** Scenario A is where the manual setup shines — simple, fast, no external dependencies.

---

### 6.2 Scenario B — Primary Bounces

#### Current behavior (manual setup)

- Standby detects primary loss and **stays passive** (no auto-promote).
- Standby rejects writes.
- Primary restarts from same data directory (no reinit).
- Standby auto-reconnects when primary returns.
- **Recovery time: ~30–60 seconds.**
- **Manual intervention: none.**

#### With Patroni

- **If the primary is down long enough** (`ttl`, default 30 seconds) **and the DCS is still reachable**, Patroni on the standby **promotes it automatically**.
- When the old primary restarts, Patroni detects it as a replica and, with `use_pg_rewind: true`, runs `pg_rewind` to automatically reconcile it with the new primary. It rejoins as a standby.
- **If the primary comes back quickly** (within `ttl`), no promotion occurs, and the outcome matches the manual setup.
- **Recovery time:**
  - Quick restart (<30s): same as manual (~30–60s).
  - Slow restart: **~15–30 seconds** for auto-promotion, plus time for the old primary to rejoin.
- **Net change:** automatic failover for longer outages. But requires **3-node etcd cluster** for DCS quorum — a 2-node etcd cluster loses quorum when the primary disappears, and Patroni cannot make a safe decision.

#### With repmgr

- **`repmgrd` on the standby auto-promotes** when the primary is unreachable for `reconnect_attempts × reconnect_interval`.
- Old primary rejoins via `repmgr node rejoin` (or is auto-rejoined if `repmgrd` is configured with `rejoin` mode).
- **Split-brain protection:** repmgr requires either a **witness node** or a **3rd node** to make a safe promotion decision. Without one, it will refuse to auto-promote (configurable via `failover` mode).
- **Recovery time:**
  - Quick restart: same as manual.
  - Slow restart: **~15–30 seconds** for auto-promotion.
- **Net change:** automatic failover, but requires a witness / 3rd node.

#### Verdict

> **HA managers shine here** — they convert a passive wait into an automatic promotion. But they require additional infrastructure (etcd cluster or witness node) to make safe decisions.
>
> For a **single-region deployment with reliable networking**, the manual setup is often sufficient. For **multi-region** or **high-RTO-tolerance** environments, auto-failover is worth the added complexity.

---

### 6.3 Scenario C — Split-Brain

#### Current behavior (manual setup)

- Partition occurs.
- Both nodes stay in their current roles (primary stays primary, standby stays standby).
- **Manual promotion** creates a second primary.
- Both nodes accept writes independently.
- Reconnect does **nothing** — no reconciliation.
- Recovery requires manual intervention: choose winner, wipe loser, re-clone.
- **Data loss on loser: guaranteed.**
- **Manual intervention required: yes.**

#### With Patroni

- **The DCS is the arbiter.** Both nodes check in with etcd, and the leader lock is a single key in etcd.
- **If the partition is between the two nodes but both can reach etcd** (a common configuration where etcd runs on a 3rd node): the primary retains the lock; the standby sees no leader but **cannot acquire the lock** because the primary still holds it. The standby remains passive.
- **If the partition isolates one node from etcd** (both node-to-node and node-to-etcd broken): that node **demotes itself** after `retry_timeout` and enters a "paused" or "demoted" state. **No split-brain.**
- **When connectivity restores**, the demoted node rejoins via `pg_rewind` (if `use_pg_rewind: true`) or a full re-clone.
- **Split-brain prevention is the primary reason to use Patroni.**
- **Recovery time after partition clears:** ~30–60 seconds.
- **Data loss:** Only if the isolated side accepted writes **before** realizing it lost the DCS. With `failsafe_mode` disabled and short timeouts, this window is small.

#### With repmgr

- **Requires a witness or 3rd node.** Without one, repmgr cannot safely distinguish "primary down" from "network partition," so it will **refuse to auto-promote**, effectively degrading to the manual behavior.
- **With a witness node:** The witness participates in failover decisions. A partitioned node without witness access cannot promote.
- **Split-brain detection:** `repmgrd` uses the `pg_visibility` mechanism to check whether it can still see the primary's writes. If it can't, and the witness agrees, it promotes.
- **Recovery time after partition clears:** similar to Patroni.
- **Data loss:** Small window, similar to Patroni.

#### Verdict

> **HA managers prevent split-brain at the design level**, using a quorum-based decision. This is the single biggest argument in favor of adding them for production.
>
> However, **both Patroni and repmgr require additional nodes or a witness to function safely.** A pure 2-node cluster cannot use either tool to safely prevent split-brain. You need at least a 3-node etcd for Patroni, or a witness node for repmgr.

---

### 6.4 Summary of HA Manager Trade-offs

| Concern | Manual (current) | Patroni | repmgr |
|---------|------------------|---------|--------|
| Extra infrastructure | None | 3-node etcd cluster | Witness or 3rd node |
| Setup complexity | Low | High | Medium |
| Automatic failover | ❌ No | ✅ Yes | ✅ Yes (with witness) |
| Split-brain prevention | ❌ No | ✅ Yes (DCS-based) | ✅ Yes (witness-based) |
| Auto-rejoin after failover | ❌ Manual | ✅ `pg_rewind` | ✅ `repmgr node rejoin` |
| Failure modes introduced | None | etcd quorum, DCS latency, split DCS | Witness failure, config drift |
| Operational overhead | Low | High | Medium |
| RTO on primary failure | Manual (minutes) | ~15–30s | ~15–30s |
| Data loss risk on partition | **High** | Low | Low |
| Best for | Small deployments, single region, low RTO | Multi-region, HA-critical | Mid-size, single region |

### 6.5 When to Add an HA Manager

**Add one if:**
- The application requires automatic failover (RTO < 1 minute without manual intervention).
- The deployment spans multiple datacenters or unreliable networks.
- The cluster runs in a hostile network environment (lossy links, unstable routing).
- Compliance requires demonstrable HA with automatic recovery.

**Stick with manual if:**
- The deployment is single-region with reliable networking.
- Manual failover is acceptable (RTO > 5 minutes is tolerable).
- The operational team prefers simplicity and full visibility.
- The cost of a 3rd node or witness is not justified.

---

## 7. Comparison Matrix

### 7.1 Behavior Summary

| Behavior | Manual (current) | Patroni | repmgr |
|----------|------------------|---------|--------|
| Standby loss detected | Immediate | Immediate | Immediate |
| Primary continues during standby outage | ✅ | ✅ | ✅ |
| Standby auto-reconnects | ✅ | ✅ | ✅ |
| Primary loss detected | Standby notices | DCS notices | Witness notices |
| Primary loss → auto-promote | ❌ | ✅ (with 3-node etcd) | ✅ (with witness) |
| Standby stays passive during partition | ✅ | ✅ (self-demotes if isolated) | ✅ (refuses to promote without witness) |
| Split-brain possible | ✅ Yes | ❌ No (DCS-based) | ❌ No (witness-based) |
| Auto-rejoin demoted primary | ❌ Manual | ✅ `pg_rewind` | ✅ `repmgr node rejoin` |
| Recovery from split-brain | Manual + data loss | Automatic + minimal loss | Automatic + minimal loss |

### 7.2 Recovery Time Matrix

| Scenario | Manual | Patroni | repmgr |
|----------|--------|---------|--------|
| Standby bounce | ~30s (auto) | ~30s (auto) | ~30s (auto) |
| Primary bounce (quick) | ~60s (auto) | ~60s (auto) | ~60s (auto) |
| Primary bounce (slow) | N/A (stays down) | ~30s (auto-promote) | ~30s (auto-promote) |
| Split-brain detection | Never | ~10–30s (DCS loss) | ~30–60s (witness loss) |
| Split-brain recovery | Manual, ~10 min | Automatic, ~60s | Automatic, ~60s |

### 7.3 Data Loss Matrix

| Scenario | Manual | Patroni | repmgr |
|----------|--------|---------|--------|
| Standby bounce | None | None | None |
| Primary bounce | None | None | None |
| Split-brain (unrecoverable on loser) | **Full loss on loser** | Minimal (window < ttl) | Minimal (window < failover delay) |

---

## 8. Architectural Conclusions

### 8.1 What This Cluster Is Good For

- **Test and development environments** requiring HA topology without operational overhead.
- **Single-region deployments** where network partitions are essentially impossible.
- **Applications that can tolerate short manual failover** (RTO measured in minutes).
- **Teams that value operational simplicity** and full transparency over automation.

### 8.2 What This Cluster Is NOT Good For

- **Production systems with strict SLAs** requiring automatic failover.
- **Multi-datacenter deployments** where network partitions are inevitable.
- **Applications with zero data-loss tolerance**.
- **24/7 operations with limited on-call staff** — a network event requires manual investigation.

### 8.3 Recommendations

1. **For the current environment (test/small prod):** Keep the manual setup. Document the recovery procedure (see §9).

2. **For production HA requirements:** Layer Patroni + 3-node etcd on top of this base, or migrate to CloudNativePG (Kubernetes) which handles everything natively.

3. **For mid-tier production:** Consider repmgr with a witness node — simpler than Patroni, but requires the witness.

4. **Regardless of choice:** 
   - **Always monitor** `pg_stat_replication` and `pg_replication_slots`.
   - **Always alert** on replication lag and slot inactivity.
   - **Always test failover** quarterly with a scripted procedure.

### 8.4 Hard-Won Lessons

1. **No HA manager = no automatic failover.** This is not a bug — it's a design choice with clear trade-offs.

2. **2-node topologies cannot support automatic failover safely.** Quorum-based tools (Patroni, repmgr) require a 3rd decision-maker.

3. **Network partitions are the enemy.** They are silent, bidirectional, and create divergent state that only a human can resolve.

4. **Data loss on split-brain is not "maybe" — it's "which side."** The only question is which node's writes get discarded.

5. **Simplicity has a cost, and so does automation.** The manual setup is simpler but requires human intervention for the hard failures. Patroni is more complex but handles those failures automatically — at the cost of etcd dependency and its own failure modes.

---

## 9. Split-Brain Recovery Runbook

This is the standard procedure validated in Scenario C.

### 9.1 Detection

Signs that split-brain has occurred:

- Both nodes report `pg_is_in_recovery = f` (both are primaries).
- Both nodes have **different `timeline_id`** values.
- Neither node shows the other in `pg_stat_replication`.
- Rows may differ between the two nodes.

### 9.2 Assessment

On **both nodes**:

```sql
SELECT pg_is_in_recovery();
SELECT timeline_id FROM pg_control_checkpoint();
SELECT current_setting('cluster_name');  -- if set
SELECT MAX(id), COUNT(*) FROM <critical_table>;
```

Compare timeline IDs.

### 9.3 Choose the Winner

**Rule:** The node with the **highest `timeline_id`** wins.

In Scenario C:
- Node 1: timeline 1 (loser)
- Node 2: timeline 2 (winner)

If timelines are equal (extremely rare), fall back to:
- Most recent `pg_last_xact_replay_timestamp()`
- Or the node with fewer writes (least data to lose)
- Or the node designated as "primary" in the runbook

### 9.4 Archive the Loser's Divergent Data

Before wiping, dump any data that might be needed:

```bash
docker exec <loser-container> pg_dump -U admin -d <db> -Fc > loser-backup-$(date +%F).dump
```

This is a **safety net**, not a merge operation. You cannot easily merge divergent data back into the cluster.

### 9.5 Stop and Wipe the Loser

```bash
cd ~/pg-ha-cluster
docker compose down --remove-orphans
docker volume rm pg-ha-cluster_primary_data
```

### 9.6 Rewrite the Loser as a Standby

- Update `docker-compose.yml` to use standby configuration.
- Update `standby-entrypoint.sh` to point at the winner.
- Update `standby/postgresql.conf` and `pg_hba.conf`.

### 9.7 Start the Loser

```bash
docker compose up -d
docker compose logs -f
```

Expected:
```
pg_basebackup: base backup completed
LOG:  entering standby mode
LOG:  started streaming WAL from primary at ... on timeline 2
```

### 9.8 Verify Recovery

On the **winner**:
```sql
SELECT client_addr, state FROM pg_stat_replication;
-- Expected: 1 row with the loser's IP
```

On the **loser**:
```sql
SELECT pg_is_in_recovery();   -- t
SELECT timeline_id FROM pg_control_checkpoint();  -- matches winner
```

### 9.9 Confirm Data Loss (Documentation)

On the **loser**:
```sql
SELECT COUNT(*) FROM <table> WHERE <condition_that_only_existed_on_loser>;
-- Expected: 0 (the divergent data is gone)
```

Record the actual loss for the incident report.

### 9.10 Post-Incident Actions

1. Document the incident (time, cause, data lost, downtime).
2. Review the partition root cause (network, firewall, etc.).
3. Consider whether an HA manager is warranted.
4. Update the runbook if any step was unclear or missing.

---

## 10. Appendices

### Appendix A — Command Reference

**Replication status (primary):**
```sql
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
SELECT slot_name, active FROM pg_replication_slots;
SELECT timeline_id FROM pg_control_checkpoint();
SELECT pg_current_wal_lsn();
```

**Standby status:**
```sql
SELECT pg_is_in_recovery();
SELECT pg_last_wal_receive_lsn();
SELECT pg_last_wal_replay_lsn();
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;
SELECT timeline_id FROM pg_control_checkpoint();
```

**Promotion (standby → primary):**
```bash
docker exec -u postgres -i <container> pg_ctl promote -D /var/lib/postgresql/data
```

**Force demotion (primary → standby):**
```bash
# Stop PostgreSQL, create standby.signal, restart
docker exec <container> touch /var/lib/postgresql/data/standby.signal
docker restart <container>
```

**Network partition (iptables):**
```bash
sudo iptables -I FORWARD -d <peer-ip> -j DROP
sudo iptables -I FORWARD -s <peer-ip> -j DROP
# To remove:
sudo iptables -D FORWARD -d <peer-ip> -j DROP
sudo iptables -D FORWARD -s <peer-ip> -j DROP
```

### Appendix B — Failure Taxonomy

| Failure Type | Symptom | Auto-Recovers? | Data Loss? |
|--------------|---------|----------------|-----------|
| Standby container restart | `pg_stat_replication` empty briefly | ✅ Yes | ❌ No |
| Primary container restart | Standby retries; primary restarts | ✅ Yes | ❌ No |
| Network partition (short) | Connection timeout; auto-retries | ✅ Yes | ❌ No |
| Network partition (long, no promotion) | Same as above | ✅ Yes | ❌ No |
| Network partition + manual promotion | Split-brain | ❌ No | ✅ Yes |
| Disk failure on primary | Depends — could require rebuild | ❌ No | Depends |
| Disk failure on standby | Standby offline; primary unaffected | ❌ No (manual rebuild) | ❌ No |

### Appendix C — Monitoring Recommendations

| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| `pg_stat_replication` row count | 0 for > 60s | Investigate standby connectivity |
| `pg_replication_slots.active` | `false` for > 5 min | Check standby logs |
| `replay_lag` on standby | > 30s | Check network, load |
| Standby `pg_is_in_recovery` | `false` unexpectedly | Possible split-brain — escalate |
| Timeline mismatch between nodes | Any mismatch | Immediate investigation |

### Appendix D — Glossary

| Term | Definition |
|------|-----------|
| **Split-brain** | Two nodes both acting as primary after a partition |
| **Timeline** | An ordered sequence of WAL records; each promotion creates a new timeline |
| **DCS** | Distributed Configuration Store (etcd, Consul, ZooKeeper) |
| **Quorum** | Majority agreement needed for distributed decisions |
| **Witness node** | A node that doesn't store data but participates in failover decisions |
| **pg_rewind** | A tool to reconcile a diverged primary with a new primary |
| **RTO** | Recovery Time Objective — how fast must service be restored |
| **RPO** | Recovery Point Objective — how much data loss is tolerable |
| **Fencing / STONITH** | Forcibly shutting down a node to prevent split-brain |
| **Failsafe mode** | Patroni feature: keep primary running even if DCS is unreachable |

---

**Document End.**
