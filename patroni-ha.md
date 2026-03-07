# PostgreSQL High Availability with Patroni

## Overview

This document covers the architecture decisions and operational concepts for running PostgreSQL in a highly available configuration using Patroni, etcd, and streaming replication across multiple data centres.

---

## Architecture: 3-Site Witness Model

The recommended architecture is a **Patroni stretched cluster** with:

- **Site A** — PostgreSQL node + etcd node
- **Site B** — PostgreSQL node + etcd node
- **Site C** — etcd witness node only (no Postgres)

Async streaming replication runs between Site A and Site B. Patroni manages automatic failover and leader election using the etcd DCS (Distributed Configuration Store).

Site C's role is more than passive storage. If Sites A and B lose their direct connection to each other, both can still reach Site C independently. Each can therefore form a 2-of-3 majority with the witness, so neither side is automatically demoted by the partition alone — the side that holds the etcd leader lock continues as primary. Site C acts as the referee that breaks the tie. Without it, a direct A↔B partition would leave both sides unable to resolve a winner.

### Why This Is Superior to a 2-Site (4-vs-3) Split

A 7-node etcd cluster split 4-vs-3 between two sites has a critical flaw: etcd requires a strict majority (`n/2 + 1`). If the 4-node site fails, the 3-node site cannot reach a quorum of 4 and freezes entirely — even though it is healthy.

| Property | 4-3 Split (2 sites) | 1-1-1 Model (3 sites) |
|---|---|---|
| Survives loss of any single site | No | Yes |
| Single point of failure | 4-node site | None |
| Cost of third site | N/A (2 sites only) | Tiny etcd witness only |
| Quorum after any partition | Depends on which side fails | Any two sites form majority |

---

## Standby Cluster vs Stretched Cluster

Understanding the trade-offs between these two approaches explains why the 3-site stretched architecture was chosen.

| Feature | Standby Cluster | Stretched Cluster |
|---|---|---|
| Failover | Manual only | Automatic (Patroni) |
| Cross-site communication | None (replication only) | Shared etcd DCS |
| Split brain risk | Low (no auto-promote) | Managed by Patroni + fencing |
| Recovery after partition | Manual reconfiguration | pg_rewind + auto rejoin |
| Operational complexity | Lower | Higher |

The stretched cluster is preferred when **RTO (Recovery Time Objective)** is important and operator response time cannot be guaranteed. The standby cluster is simpler and safer when full control over switchover timing is required.

---

## How etcd Quorum Works

etcd uses Raft consensus. A cluster of `n` nodes requires `n/2 + 1` nodes to agree before any write (including the Patroni leader lock) is accepted.

| Cluster size | Required quorum | Fault tolerance |
|---|---|---|
| 3 | 2 | 1 node |
| 5 | 3 | 2 nodes |
| 7 | 4 | 3 nodes |

In a WAN partition, the side that can form a quorum continues operating. The side that cannot reach quorum stops accepting writes and loses the leader lock — Patroni on that side will demote its Postgres node.

### Shared etcd Across Multiple Clusters

This same quorum model applies when etcd is shared across multiple Patroni clusters. Sharing a single etcd cluster is supported, but each Patroni cluster must use a unique `scope` name (namespace) to avoid key collisions. This approach increases coupling between clusters — a full etcd outage affects all of them simultaneously.

---

## Patroni Automatic Failover

Patroni monitors the leader lock in etcd continuously. If the primary cannot renew its lock before the TTL expires, Patroni initiates an automatic failover:

1. The primary is demoted (fencing).
2. The most up-to-date replica is elected as the new primary.
3. All other replicas reconfigure to follow the new primary.

### What a Script Cannot Replicate

A hand-written switchover script would need to independently solve every one of the following problems that Patroni handles out of the box:

- **Fencing** — Ensuring the old primary is truly stopped before the replica promotes. Without this, two nodes can both believe they are primary (split brain).
- **Source of truth** — A DCS (etcd) that provides consensus and prevents two primaries from existing simultaneously.
- **Best replica election** — Choosing the replica with the least replication lag to minimise data loss.
- **Catch-up promotion** — Allowing the replica to consume its remaining WAL before promoting, when possible.
- **Node rebuild** — Automatically reconfiguring nodes that rejoin the cluster after a partition (via `pg_basebackup` or `pg_rewind`).
- **Network jitter resilience** — Tolerating brief network blips without triggering false failovers.

### Disabling Automatic Failover

Setting `pause: true` effectively disables fencing and the HA apparatus. Patroni will continue reporting state to the DCS but will not act on failures. This means:

- A network partition can produce a split brain with no automatic recovery.
- Any manually triggered switchover during a partition carries the same risk.
- The entire safety model depends on operator actions being correct and timely.

This mode should only be used for planned maintenance with a full understanding of the current cluster topology.

---

## Preventing Split Brain

Split brain occurs when two nodes simultaneously believe they are the primary and accept writes. This causes **timeline divergence** — the two timelines can never be trivially merged.

Patroni's protections:

1. **Leader lock TTL** — The primary must renew its lock at every `loop_wait` interval. If it fails to renew before `ttl` expires, the lock is released.
2. **Fencing via DCS** — Before promoting, the replica acquires the lock. The old primary cannot renew a lock it no longer holds and must step down.
3. **retry_timeout** — Short DCS blips are retried for up to `retry_timeout` seconds before any action is taken, preventing failovers caused by momentary jitter.

---

## Timing Parameters

Three parameters control how quickly Patroni reacts to failures. They must be tuned together:

| Parameter | Default | Minimum | Purpose |
|---|---|---|---|
| `loop_wait` | 10 s | 1 s | Sleep interval between each Patroni loop iteration |
| `ttl` | 30 s | 20 s | How long before the leader lock expires and failover begins |
| `retry_timeout` | 10 s | 3 s | How long to retry DCS/Postgres operations before acting |

**Rule of thumb:** `ttl` should be at least `2 × loop_wait + retry_timeout` to avoid false failovers.

### Tuning for Network Jitter

In WAN environments with intermittent packet loss, increase all three values to let the cluster ride out blips without unnecessary promotions:

```yaml
# patroni.yml
bootstrap:
  dcs:
    ttl: 60
    loop_wait: 15
    retry_timeout: 15
```

With these values, a network outage shorter than ~60 seconds will not trigger automatic failover.

---

## Manual Switchover

Use a manual switchover when the cluster is healthy but you need to redirect clients — for example, when clients can only reach Site B but the primary is on Site A, and Patroni considers Site A healthy (because it can still talk to the witness).

```bash
patronictl -c /etc/patroni.yml switchover <cluster-name> \
  --primary <site-a-host> \
  --candidate <site-b-host> \
  --scheduled now
```

Patroni performs a graceful shutdown: it checkpoints the primary, waits for the replica to catch up, then promotes. There is no data loss under normal conditions.

---

## Timeline Divergence and pg_rewind

If a **forceful** switchover or failover is executed while a network partition is active, the old primary may have committed transactions that the new primary does not have. This is a **timeline divergence**.

Once the network heals, Patroni automatically resolves this using `pg_rewind`:

1. The old primary connects to the new primary.
2. `pg_rewind` rewinds the old primary's data directory to the divergence point.
3. The old primary replays the new primary's WAL from that point forward.
4. The old primary rejoins as a follower.

Any transactions that were committed only on the old primary (after the divergence) are lost. This is the accepted trade-off of async replication under a network partition.

> **Prerequisites:**
> - `wal_log_hints = on` or data checksums must be enabled in PostgreSQL.
> - `use_pg_rewind: true` must be set in `patroni.yml`:
>
>   ```yaml
>   postgresql:
>     use_pg_rewind: true
>   ```
>
> Without `use_pg_rewind: true`, Patroni falls back to a full `pg_basebackup` to rebuild the diverged node, which is significantly slower.

---

## Q&A

### 1. Cross-DC Support and Latency Concerns

**Q: Is a stretched cluster supported across data centres? What about latency?**

The stretched cluster architecture described in this document is designed for cross-DC deployment. The key constraint is **etcd**, which uses Raft consensus with default heartbeat intervals of 100ms and election timeouts of 1000ms.

**Latency guidelines:**

| RTT between sites | Suitability | Notes |
|---|---|---|
| ≤ 10ms | Ideal | No etcd tuning needed |
| 10–50ms | Workable | Tune `heartbeat-interval` and `election-timeout` |
| > 50ms | Not recommended | etcd leader elections become unstable; consider a standby cluster instead |

Our measured baseline RTT between sites is **~10ms**, which falls in the ideal range.

**What about latency peaks?**

The 3-node etcd topology makes peaks significantly less dangerous:

- etcd needs **2 out of 3** nodes for quorum. A write only needs to reach the **fastest responding peer** — the slow node catches up asynchronously.
- If one link spikes but the other two are healthy, the cluster continues at normal speed.
- Patroni's timing parameters (`ttl: 60`, `retry_timeout: 15`) mean the failover logic ignores anything shorter than ~15 seconds.
- With a tuned etcd `election-timeout` (e.g., 5000ms), a latency spike would need to be sustained for several seconds before triggering a leader election.

Peaks would only matter if **two links spike simultaneously** (unlikely with 3 independent sites) or if a spike lasts long enough to cause an etcd leader election.

### 2. Handling Network Blips and Flapping

**Q: How does the system handle momentary network disconnections? Won't clusters flap between sites?**

Patroni is specifically designed to avoid flapping. Multiple layers prevent a brief network blip from triggering an unnecessary failover:

1. **`retry_timeout`** — Patroni retries DCS and PostgreSQL operations for up to `retry_timeout` seconds before taking any action. Short blips are silently absorbed (see [Timing Parameters](#timing-parameters)).
2. **`ttl` (leader lock expiry)** — The primary's lock must fully expire before any failover begins. With `ttl: 60`, the network must be down for a full minute before the lock is released.
3. **etcd quorum via 3 sites** — A partition between Site A and Site B does not cause flapping because both sites can still reach the witness (Site C). The side holding the etcd leader lock remains primary. There is no back-and-forth — the lock is the single source of truth (see [Architecture: 3-Site Witness Model](#architecture-3-site-witness-model)).
4. **Fencing** — Once a failover does occur, the old primary is forcefully demoted and cannot reclaim the lock until it rejoins as a replica. This prevents the "bouncing" behaviour seen in systems without proper fencing (see [Preventing Split Brain](#preventing-split-brain)).

With the recommended WAN tuning (`ttl: 60`, `loop_wait: 15`, `retry_timeout: 15`), outages shorter than ~60 seconds will not trigger a failover at all.

### 3. Data Loss During Failover

**Q: Do we failover at all costs, regardless of potential data loss?**

No. Patroni provides a configurable safety net: **`maximum_lag_on_failover`**.

This parameter defines the maximum replication lag (in bytes) a replica is allowed to have before it can be considered for promotion. If all replicas exceed this threshold, **Patroni will not failover** — it prefers unavailability over data loss.

```yaml
# patroni.yml
bootstrap:
  dcs:
    maximum_lag_on_failover: 1048576  # 1 MB (default)
```

**How it works in practice:**

| Scenario | Behaviour |
|---|---|
| Replica lag < `maximum_lag_on_failover` | Failover proceeds; replica consumes remaining WAL then promotes |
| Replica lag > `maximum_lag_on_failover` | Failover is blocked; cluster remains without a primary until the issue is resolved |
| Graceful switchover (`patronictl switchover`) | Replica catches up fully before promoting — **zero data loss** |

For unplanned failovers (crashes, unexpected outages), `maximum_lag_on_failover` is the safety net. For planned switchovers, Patroni ensures the replica catches up fully before promoting.

For details on how diverged nodes are recovered, see [Timeline Divergence and pg_rewind](#timeline-divergence-and-pg_rewind).

> **Decision required:** The team should agree on an acceptable `maximum_lag_on_failover` value that balances data safety against availability.

> **NEEDS REVIEW:** The following hybrid approach is proposed but has not yet been validated by the team.

**Hybrid approach: async by default, sync before planned switchover**

Instead of running synchronous replication permanently (which adds ~1x RTT write latency), the idea is to:

1. Run in **async mode** during normal operation for maximum write performance.
2. Before a **planned switchover**, dynamically enable sync replication to guarantee the replica is fully caught up.
3. Perform the switchover with **zero data loss**.
4. Switch back to async mode after the switchover completes.

Patroni supports changing `synchronous_mode` at runtime without a restart:

```bash
# Step 1: Enable sync mode
patronictl -c /etc/patroni.yml edit-config -s 'synchronous_mode=true'
# Step 2: Wait for replica to confirm sync
# Step 3: Switchover
patronictl -c /etc/patroni.yml switchover <cluster-name>
# Step 4: Return to async mode
patronictl -c /etc/patroni.yml edit-config -s 'synchronous_mode=false'
```

This gives the best of both worlds: low-latency writes in steady state, and zero data loss for planned migrations.

### 4. Justification for Patroni as an Additional Tool

**Q: Why maintain another tool? Why not write a migration/failover script and expose it through SRE or an existing platform? Oracle has automatic failover but it was never implemented — there must be a good reason.**

A hand-written script would need to independently solve every problem listed in [What a Script Cannot Replicate](#what-a-script-cannot-replicate): fencing, consensus-based leader election, best replica selection, catch-up promotion, node rebuild, and network jitter resilience. Each of these is a complex distributed systems problem. Combining them into a reliable, production-grade script is effectively building Patroni from scratch.

**Key differences from Oracle's situation:**

- Oracle has **Data Guard Broker** — a built-in HA framework with fencing, role management, and automatic failover. The capability already exists natively; choosing not to enable it is a policy decision.
- PostgreSQL has **no native equivalent**. There is no built-in orchestrator for automatic failover, fencing, or replica promotion coordination. Patroni fills this gap.

**Why not just a script exposed via SRE?**

- A script runs **on demand** — it cannot detect failures or react in real time. Someone must notice the outage, decide to act, and trigger the script. This directly increases RTO.
- A script has **no state** — it cannot know whether the old primary is truly down, whether a replica is caught up, or whether another operator is running a conflicting action at the same time.
- Patroni runs **continuously** as a daemon alongside PostgreSQL, monitoring health and holding a distributed lock. This is fundamentally different from a script that runs once and exits.

Patroni is not "another tool" in the same sense as adding a new monitoring system or deployment platform. It is the **missing HA layer** that PostgreSQL does not ship with — comparable to what Oracle already has built in.
