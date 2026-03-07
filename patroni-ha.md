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
| Cost of third site | Full Postgres node | Tiny etcd witness only |
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
