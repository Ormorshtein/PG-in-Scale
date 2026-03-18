# Patroni HA Test Framework

## What This Project Is

An automated test framework for validating a Patroni + etcd PostgreSQL High Availability deployment across a 3-site witness model (Site A: PG + etcd, Site B: PG + etcd, Site C: etcd witness only). The framework runs 25 tests grouped into 5 categories and produces a structured JSON report with pass/fail results, timing measurements, and data loss quantification.

All 25 tests run fully unattended with no manual interaction. Every test includes automated recovery back to a healthy cluster state before the next test begins.

This is an internal tool for a platform team that manages on-premises data infrastructure for 100+ internal customer teams.

## Architecture Under Test

```
Site A                Site B                Site C
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ PostgreSQL   │     │ PostgreSQL   │     │              │
│ Patroni      │◄───►│ Patroni      │     │              │
│ etcd node    │◄───►│ etcd node    │◄───►│ etcd witness │
└──────────────┘     └──────────────┘     └──────────────┘
```

- Async streaming replication between Site A and Site B.
- Patroni manages automatic failover using etcd as the DCS.
- Site C is an etcd-only witness node for quorum tie-breaking.
- Patroni timing: `ttl: 60`, `loop_wait: 15`, `retry_timeout: 15`.

## Test Runner Placement (CRITICAL)

The machine running this test framework MUST be on a separate management network with independent network paths to all three sites. It MUST NOT be co-located with any site under test. If the test runner shares a network path with Site A and a partition test blocks A↔B, the SSH session to Site B may hang or break.

If a fully independent management network is not available, the `ssh.py` module must be configured to use out-of-band management IPs (e.g., IPMI, iLO, or a dedicated management VLAN) that are not affected by the data-plane iptables/route rules. The `topology.yaml` supports separate `ssh_host` and `pg_host`/`etcd_host` fields for this purpose.

## Tech Stack

- **Language:** Python 3.10+
- **Test framework:** pytest with pytest-timeout
- **SSH:** paramiko (for remote command execution on PG/etcd hosts)
- **HTTP:** httpx (for Patroni REST API and etcd API calls)
- **Config:** YAML config file for environment topology
- **Output:** JSON test report + optional JUnit XML for CI integration

## pyproject.toml

```toml
[project]
name = "patroni-ha-tests"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "pytest>=7.0",
    "pytest-timeout>=2.1",
    "pytest-json-report>=1.5",
    "paramiko>=3.0",
    "httpx>=0.24",
    "psycopg[binary]>=3.1",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
dev = ["ruff", "mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]
timeout = 600
log_cli = true
log_cli_level = "INFO"
markers = [
    "watchdog: tests requiring softdog kernel module",
    "hypervisor: tests requiring VM power control API",
    "meta: framework self-tests (run first)",
]
```

## Project Structure

```
patroni-ha-tests/
├── README.md
├── pyproject.toml
├── config/
│   ├── topology.yaml          # Environment topology (hosts, ports, SSH credentials)
│   └── topology.example.yaml  # Example config with comments
├── src/
│   └── patroni_ha_tests/
│       ├── __init__.py
│       ├── config.py           # Load and validate topology.yaml
│       ├── ssh.py              # SSH connection manager (paramiko wrapper)
│       ├── patroni_client.py   # Patroni REST API client
│       ├── etcd_client.py      # etcd health/status client
│       ├── pg_client.py        # PostgreSQL query helper (psycopg2/3)
│       ├── workload.py         # Write workload generator with sequence tracking
│       ├── network.py          # Network partition helper (iptables + routes) with cleanup guarantees
│       ├── timing.py           # RTO/RPO measurement utilities
│       ├── recovery.py         # Automated cluster recovery between tests
│       └── report.py           # Test result aggregation and JSON/JUnit output
├── tests/
│   ├── conftest.py             # pytest fixtures (SSH sessions, clients, cleanup)
│   ├── test_00_meta/
│   │   └── test_partition_ttl_safety.py
│   ├── test_01_graceful/
│   │   ├── test_planned_switchover.py
│   │   ├── test_hybrid_sync_switchover.py
│   │   └── test_switchover_switchback.py
│   ├── test_02_unplanned/
│   │   ├── test_kill_primary_process.py
│   │   ├── test_hard_poweroff.py
│   │   ├── test_write_load_during_kill.py
│   │   ├── test_kill_replica_process.py
│   │   └── test_hard_poweroff_replica.py
│   ├── test_03_partition/
│   │   ├── test_partition_ab_witness_up.py
│   │   ├── test_site_a_isolated.py
│   │   ├── test_witness_unreachable_from_b.py
│   │   ├── test_witness_down_then_partition.py
│   │   ├── test_partition_via_host_routes.py
│   │   ├── test_full_mesh_partition.py
│   │   └── test_asymmetric_partition.py
│   └── test_04_edge_cases/
│       ├── test_failover_blocked_by_lag.py
│       ├── test_pg_rewind_divergence.py
│       ├── test_etcd_leader_change_mid_failover.py
│       ├── test_pause_mode_split_brain.py
│       ├── test_cascading_pg_etcd_failure.py
│       └── test_lagging_replica_promoted_after_primary_dies.py
│   └── test_05_watchdog/
│       ├── test_watchdog_graceful_demotion.py
│       ├── test_watchdog_fires_on_stuck_patroni.py
│       ├── test_watchdog_slow_pg_shutdown.py
│       └── test_watchdog_required_mode_no_device.py
└── scripts/
    ├── run_all.sh              # Run full suite with report generation
    ├── reset_environment.sh    # Reset cluster to known-good state between tests
    └── partition_watchdog.sh   # Safety watchdog that auto-cleans stale partitions (runs on each host via cron)
```

## Configuration File: topology.yaml

```yaml
sites:
  site_a:
    # ssh_host is used for test runner SSH access (management plane).
    # pg_host/etcd_host are used for data plane — these are the IPs that
    # partition tests will block. If ssh_host differs from pg_host,
    # SSH remains reachable during partition tests.
    ssh_host: "10.0.1.10"        # management IP (or same as pg_host if no separate mgmt network)
    pg_host: "10.0.1.10"
    pg_port: 5432
    patroni_port: 8008
    etcd_host: "10.0.1.10"
    etcd_client_port: 2379
    etcd_peer_port: 2380
    ssh_user: "admin"
    ssh_key_path: "~/.ssh/id_rsa"
  site_b:
    ssh_host: "10.0.2.10"
    pg_host: "10.0.2.10"
    pg_port: 5432
    patroni_port: 8008
    etcd_host: "10.0.2.10"
    etcd_client_port: 2379
    etcd_peer_port: 2380
    ssh_user: "admin"
    ssh_key_path: "~/.ssh/id_rsa"
  site_c:
    ssh_host: "10.0.3.10"
    etcd_host: "10.0.3.10"
    etcd_client_port: 2379
    etcd_peer_port: 2380
    ssh_user: "admin"
    ssh_key_path: "~/.ssh/id_rsa"

patroni:
  cluster_name: "pg-ha-cluster"
  config_path: "/etc/patroni.yml"
  ttl: 60
  loop_wait: 15
  retry_timeout: 15
  maximum_lag_on_failover: 1048576  # 1MB
  pg_data_dir: "/var/lib/postgresql/16/main"  # or read dynamically from patroni.yml

# Watchdog configuration for tests 22-25.
# These tests require the softdog kernel module loaded on both PG hosts.
# If not configured, tests 22-25 will be skipped.
watchdog:
  enabled: true
  device: "/dev/watchdog"
  # The Patroni watchdog mode currently configured on the cluster.
  # Tests will temporarily change this and restore it afterward.
  current_mode: "automatic"  # or "required", "off"
  # softdog module loaded with soft_noboot=1 for testing?
  # If true, the watchdog logs instead of rebooting — needed for tests
  # that intentionally trigger the watchdog (test 23).
  # If false (production softdog), test 23 will actually reboot the machine.
  soft_noboot: true

postgres:
  superuser: "postgres"
  superuser_password: "${PG_SUPERUSER_PASSWORD}"  # env var reference
  test_database: "ha_test_db"

# Optional: hypervisor API for VM power-off tests (tests 5, 17)
# If not configured, tests 5 and 17 will be skipped.
hypervisor:
  type: "libvirt"  # or "vsphere", "cloud"
  # libvirt:
  #   uri: "qemu+ssh://admin@hypervisor/system"
  #   vm_names:
  #     site_a: "pg-site-a"
  #     site_b: "pg-site-b"

# Ports to block during partition tests. Only these ports are blocked —
# SSH (22) and other management traffic remain open.
# If ssh_host == pg_host (no separate management network), add port 22
# to this list AND ensure the test runner can tolerate SSH loss to
# partitioned sites (use fresh connections with aggressive timeouts).
partition_ports:
  - 5432   # PostgreSQL
  - 2379   # etcd client
  - 2380   # etcd peer
  - 8008   # Patroni REST API

timeouts:
  failover_max_wait: 120       # seconds to wait for failover to complete
  replication_sync_wait: 30    # seconds to wait for replication to catch up
  partition_hold_time: 120     # seconds to hold a partition before checking state
  pg_rewind_max_wait: 180      # seconds to wait for pg_rewind to complete
  ssh_connect_timeout: 10      # seconds for SSH connection attempts
  ssh_command_timeout: 30      # seconds for SSH command execution
```

## Module Specifications

### src/patroni_ha_tests/config.py

Load `topology.yaml`, resolve environment variable references (${VAR} syntax), and validate that all required fields are present. Expose a `Config` dataclass with typed access to all fields. Raise clear errors if required fields are missing or if env vars are unset. If `ssh_host` is not specified for a site, default it to `pg_host`.

The config file path comes from the `--config` pytest CLI arg. Register it in `conftest.py`:
```python
def pytest_addoption(parser):
    parser.addoption("--config", action="store", default="config/topology.yaml",
                     help="Path to topology.yaml")
```

#### Key dataclass definitions:

```python
@dataclass
class SiteConfig:
    ssh_host: str
    pg_host: str
    pg_port: int
    patroni_port: int
    etcd_host: str
    etcd_client_port: int
    etcd_peer_port: int
    ssh_user: str
    ssh_key_path: str

@dataclass
class PatroniConfig:
    cluster_name: str
    config_path: str
    ttl: int
    loop_wait: int
    retry_timeout: int
    maximum_lag_on_failover: int
    pg_data_dir: str

@dataclass
class WatchdogConfig:
    enabled: bool
    device: str
    current_mode: str  # "automatic", "required", "off"
    soft_noboot: bool

@dataclass
class TimeoutsConfig:
    failover_max_wait: int
    replication_sync_wait: int
    partition_hold_time: int
    pg_rewind_max_wait: int
    ssh_connect_timeout: int
    ssh_command_timeout: int

@dataclass
class Config:
    sites: dict[str, SiteConfig]  # keys: "site_a", "site_b", "site_c"
    patroni: PatroniConfig
    postgres: dict  # superuser, superuser_password, test_database
    watchdog: Optional[WatchdogConfig]
    hypervisor: Optional[dict]
    partition_ports: list[int]
    timeouts: TimeoutsConfig
    
    def pg_sites(self) -> list[str]:
        """Return site names that have PostgreSQL (site_a, site_b)."""
        return [name for name, site in self.sites.items() if hasattr(site, 'pg_port')]
```

### src/patroni_ha_tests/ssh.py

Paramiko-based SSH connection manager.

```python
@dataclass
class CommandResult:
    stdout: str
    stderr: str
    exit_code: int
    duration_seconds: float

@dataclass
class BackgroundProcess:
    site: str
    command: str
    channel: paramiko.Channel  # the underlying SSH channel
    
    def is_alive(self) -> bool: ...
    def kill(self) -> None: ...
    def wait(self, timeout: int = 30) -> CommandResult: ...
```

- `SSHManager` class that holds connections to all 3 sites.
- Constructor: `SSHManager(config: Config)` — initializes connections to all sites using `ssh_host`, `ssh_user`, `ssh_key_path` from config.
- Connects using `ssh_host` (management IP), NOT `pg_host`. This is critical — during partition tests, `pg_host` may be unreachable but `ssh_host` must remain accessible.
- `run(site: str, command: str, sudo: bool = False, timeout: int = None) -> CommandResult` — execute a command, return stdout/stderr/exit_code. Default timeout from config (`ssh_command_timeout`).
- `run_bg(site: str, command: str) -> BackgroundProcess` — start a background process, return a handle that can be used to check status or kill it.
- Connection pooling: reuse SSH connections across commands within a test. Close all on teardown.
- **Resilient connections:** If a command fails with `socket.timeout`, `paramiko.SSHException`, or `EOFError`, the manager should automatically drop the stale connection, establish a new one (with `ssh_connect_timeout`), and retry once. Only raise after the retry fails.
- `reconnect(site: str)` — force-drop and re-establish the SSH connection to a specific site. Tests can call this after applying/removing partitions if `ssh_host == pg_host`.

### src/patroni_ha_tests/patroni_client.py

HTTP client for Patroni's REST API, with SSH fallback for `patronictl` commands.

Constructor: `PatroniClient(config: Config, ssh: SSHManager)` — needs both because some operations use the REST API (read-only queries) and some use `patronictl` via SSH (state-changing operations).

- **REST API calls** use `httpx` directly from the test runner to `http://{pg_host}:{patroni_port}/...`. These are read-only and fast.
- **patronictl calls** use SSH to execute commands on the node. These are used for switchover, pause, resume, and config changes because `patronictl` handles interactive prompts and DCS coordination.

```python
@dataclass
class ClusterMember:
    name: str
    site: str         # "site_a" or "site_b"
    role: str         # "primary" or "replica"
    state: str        # "running", "stopped", "unknown"
    lag_bytes: int
    timeline: int

@dataclass
class ClusterStatus:
    members: list[ClusterMember]
    primary: Optional[ClusterMember]
    replicas: list[ClusterMember]
    patroni_version: str

@dataclass
class SwitchoverResult:
    success: bool
    message: str         # raw patronictl output
    duration_seconds: float
```

- `get_cluster_status(site: str) -> ClusterStatus` — GET /cluster, parse into ClusterStatus.
- `get_node_role(site: str) -> str` — GET /primary returns 200 (primary) or 503 (replica). GET /replica returns 200 (replica) or 503 (primary). Return "primary" or "replica". Uses httpx with a 5-second timeout. Returns "unreachable" if the request fails.
- `wait_for_role(site: str, role: str, timeout: int) -> bool` — poll `get_node_role()` every 2 seconds until the node has the expected role or timeout.
- `is_healthy(site: str) -> bool` — GET /health returns 200.
- `discover_primary() -> str` — query both PG sites via `get_node_role()`, return the site name that is currently primary. Raises `NoPrimaryError` if neither is primary. This is the canonical way to determine the current primary — no test should assume which site is primary.
- `discover_replica() -> str` — return the site name that is currently replica. Raises `NoReplicaError` if neither is replica.
- `trigger_switchover(source_site: str, target_site: str) -> SwitchoverResult` — via SSH, execute: `patronictl -c {config_path} switchover {cluster_name} --primary {source_host} --candidate {target_host} --force`. The `--force` flag avoids interactive prompts. Parse stdout for success/failure.
- `pause(site: str)` / `resume(site: str)` — via SSH, execute `patronictl -c {config_path} pause` / `resume`.
- `edit_config(site: str, key: str, value: str)` — via SSH, execute `patronictl -c {config_path} edit-config -s '{key}={value}' --force`.

### src/patroni_ha_tests/etcd_client.py

HTTP client for etcd's REST API.

Constructor: `EtcdClient(config: Config)` — stores the etcd endpoints for all 3 sites.

All queries use `httpx` from the test runner directly to `http://{etcd_host}:{etcd_client_port}/...`. For etcd v3 API, use the `/v3/` endpoints. For etcd v2 (if running etcd 2.x), use `/v2/keys/...`. Auto-detect the API version by trying `/version` first.

- `get_cluster_health() -> dict[str, bool]` — query each etcd node's `/health` endpoint. Return `{"site_a": True, "site_b": True, "site_c": False}`.
- `get_leader() -> str` — determine which etcd node is the current Raft leader. Use `/v3/maintenance/status` (v3) or parse the `X-Raft-Leader` header (v2). Return the site name.
- `get_member_list() -> list[dict]` — `/v3/cluster/member/list` or `etcdctl member list` via SSH. Return parsed member info.
- `is_quorum_healthy() -> bool` — True if >= 2 of 3 nodes return healthy from `get_cluster_health()`.
- All HTTP calls use a 5-second timeout. Catch `httpx.ConnectError` and `httpx.TimeoutException` — return unhealthy/unreachable, don't crash.

### src/patroni_ha_tests/pg_client.py

Thin wrapper around psycopg (v3) for test-specific queries.

Constructor: `PGClient(config: Config, ssh: SSHManager)` — stores the config for connection params. Also keeps an SSH reference for `can_write()` which needs to work during partitions.

**Connection strategy:** Direct connections from the test runner use `pg_host:{pg_port}`. During partition tests, the test runner may not be able to reach a partitioned node's PG port. For operations that need to run on a partitioned node (like `can_write()`), use SSH to execute `psql` on the host itself rather than connecting remotely.

- `connect(site: str) -> psycopg.Connection` — connect to PG on the specified site via `pg_host`. Use `connect_timeout=5`. Caller is responsible for closing.
- `connect_via_ssh(site: str, sql: str) -> str` — execute a SQL command on the site by SSHing to the host and running `psql -U {superuser} -d {test_database} -t -A -c "{sql}"`. Returns stdout. Use this when the test runner can't reach PG directly (during partitions).
- `insert_marker(conn, marker_id: str) -> datetime` — insert a row into `ha_test_markers(id TEXT, ts TIMESTAMPTZ, seq BIGSERIAL)`. Return the timestamp.
- `get_marker(conn, marker_id: str) -> Optional[Row]` — retrieve a marker row.
- `get_wal_lsn(conn) -> str` — `SELECT pg_current_wal_lsn()` (primary) or `SELECT pg_last_wal_receive_lsn()` (replica).
- `get_timeline_id(conn) -> int` — from `pg_control_checkpoint()`.
- `is_in_recovery(conn) -> bool` — `SELECT pg_is_in_recovery()`.
- `pause_wal_replay(conn)` / `resume_wal_replay(conn)` — for test 11.
- `can_write(site: str) -> bool` — uses `connect_via_ssh()` to run: `BEGIN; CREATE TEMP TABLE _ha_test_probe(x int); ROLLBACK;`. Returns True if it succeeds, False if it fails with a read-only error. Uses SSH because the test runner may not be able to reach the node's PG port during a partition, but SSH (via `ssh_host`) remains available.
- `setup_test_schema(conn)` — `CREATE TABLE IF NOT EXISTS ha_test_markers(id TEXT PRIMARY KEY, ts TIMESTAMPTZ DEFAULT now(), seq BIGSERIAL)` and `CREATE TABLE IF NOT EXISTS ha_test_workload(seq BIGINT PRIMARY KEY, ts TIMESTAMPTZ DEFAULT now(), payload TEXT)`.
- `get_all_workload_seqs(conn) -> set[int]` — `SELECT seq FROM ha_test_workload ORDER BY seq`. Returns the set of all seq values present. Used by `timing.measure_data_loss()`.

### src/patroni_ha_tests/workload.py

Write workload generator for measuring data loss during failover.

Constructor: `WorkloadGenerator(config: Config, target_site: str)` — connects directly to `pg_host:{pg_port}` of the specified site (the current primary). Does NOT connect via a load balancer — the workload must target the primary directly so that connection failures during failover are deterministic and measurable.

```python
@dataclass
class WorkloadStats:
    committed_seqs: frozenset[int]  # immutable snapshot of committed seqs at time of call
    last_committed_seq: int
    kill_seq: Optional[int]
    total_attempted: int
    total_committed: int
    total_failed: int
    actual_tps: float              # measured TPS over the run duration
    run_duration_seconds: float
```

- `WorkloadGenerator` class that runs INSERT statements in a loop with a monotonic sequence counter.
- Each INSERT: `INSERT INTO ha_test_workload(seq, ts, payload) VALUES (%s, now(), %s)` where `seq` is the counter and `payload` is a small random string.
- The generator tracks:
  - `committed_seqs: set[int]` — the full set of seq values that received a successful commit acknowledgment from PG. This is the source of truth, not just MAX(seq).
  - `last_committed_seq: int` — the highest seq in `committed_seqs` (convenience accessor).
  - `total_attempted: int`, `total_committed: int`, `total_failed: int`.
  - `kill_seq: Optional[int]` — the seq counter value at the moment `record_kill_point()` is called. Tests call this immediately before killing the primary.
- `committed_seqs` MUST be a thread-safe set (use `threading.Lock`). Every successful commit appends to this set; every failed commit is logged but not added.
- Runs in a separate thread. Exposes `start()`, `stop()`, `record_kill_point()`, `get_stats() -> WorkloadStats`.
- On connection failure (during failover), the generator should retry with backoff (initial 0.5s, max 5s, up to 30 retries), not crash. It logs each failed attempt with the seq number.
- Configurable TPS target (default: 100). Uses `time.sleep()` to throttle.
- **Data loss measurement** (in `timing.py`): after failover, query the new primary for `SELECT seq FROM ha_test_workload ORDER BY seq` and compare against `committed_seqs`. The difference is the precise set of lost transactions — not just a count but the actual seq values. This catches gaps (seq 98 committed, 99 lost, 100 committed) that a simple MAX comparison would miss.

### src/patroni_ha_tests/network.py

Network partition management via iptables or host routes. This is the most safety-critical module.

- `NetworkPartition` context manager class.
- Usage: `with NetworkPartition(ssh, config, block_pairs=[("site_a", "site_b")], method="iptables") as partition:` — applies blocking rules on both sides, yields, then removes rules in the finally block regardless of test outcome.
- **Supports two partition methods** (configurable per test):
  - `method="iptables"` (default): Uses port-specific blocking rules. For each port in `config.partition_ports`: `iptables -A INPUT -s <target_ip> -p tcp --dport <port> -j DROP -m comment --comment "patroni-ha-test"` and the corresponding OUTPUT rule. This blocks only the service ports, leaving SSH and other management traffic open.
  - `method="route"`: Uses `ip route add blackhole <target_ip>/32` on both sides. NOTE: blackhole routes block ALL traffic to the IP including SSH. Only use this method if `ssh_host != pg_host` (separate management network). If `ssh_host == pg_host`, the framework must reject `method="route"` with a clear error.
- `block(source_site: str, target_site: str)` — apply the blocking rules using the configured method on both sides. Write a timestamp file (`/tmp/patroni-ha-test-partition-timestamp`) on each affected host for the watchdog.
- `unblock(source_site: str, target_site: str)` — remove the specific rules. For iptables: use `-D` not `-F`. For routes: `ip route del blackhole <target_ip>/32`. Remove the timestamp file.
- `verify_partition(source_site: str, target_site: str) -> bool` — after applying a partition, verify it is effective by attempting a TCP connect from source to target on a service port (e.g., 5432). Returns True if the connection fails (partition working). Tests MUST call this after applying a partition and before proceeding.
- **Critical safety features:**
  - Always use the context manager pattern. Never expose raw block/unblock without cleanup guarantees.
  - On `__exit__`, remove ALL rules that were added, even if the test raised an exception.
  - `force_cleanup()` — called by the network fixture teardown. Removes all rules this instance ever applied, ignoring errors.
  - `verify_all_clean(ssh, config)` class method — SSHes to every host and checks that no test-related iptables rules or blackhole routes remain. Call in the network fixture teardown after all force_cleanup calls.
  - For iptables: cleanup uses `iptables -S | grep "patroni-ha-test"` to find test rules.
  - For routes: cleanup uses `ip route show type blackhole` to find test routes.
  - **Partition watchdog (separate process):** Deploy `scripts/partition_watchdog.sh` on each host via cron. It checks for stale rules older than 10 minutes and removes them. This is the safety net if the test runner crashes.
- Log every command executed for debugging.

### src/patroni_ha_tests/timing.py

```python
@dataclass
class RTOResult:
    new_primary_site: str      # which site became primary
    elapsed_seconds: float     # time from start of measurement to primary detected
    succeeded: bool            # True if primary was found within timeout

@dataclass
class DataLossResult:
    lost_count: int                # number of lost transactions
    lost_seqs: list[int]           # the specific seq values that were lost
    max_committed_seq: int         # highest seq the workload committed
    max_seq_on_new_primary: int    # highest seq found on the new primary
    has_gaps: bool                 # True if losses are non-contiguous
    estimated_bytes_lost: int      # rough estimate: lost_count * avg_row_size
```

- `Timer` context manager: `with Timer() as t:` — captures start/end time with `time.monotonic()`. Exposes `t.elapsed` after exit.
- `measure_failover_rto(patroni_client, timeout) -> RTOResult` — polls `patroni_client.get_node_role()` on BOTH PG sites every 2 seconds until one returns "primary". Does NOT assume which site will become primary. Returns the site that became primary, elapsed time, and whether it succeeded within timeout.
- `measure_data_loss(pg_client, workload: WorkloadGenerator, new_primary_site: str) -> DataLossResult` — calls `pg_client.get_all_workload_seqs()` on the new primary, compares against `workload.get_stats().committed_seqs`. The difference (seqs in committed_seqs but not on the new primary) is the precise set of lost transactions.
- `poll_until(fn: Callable[[], bool], timeout: int, interval: float = 2.0, description: str = "") -> bool` — generic polling helper. Calls `fn()` every `interval` seconds until it returns True or `timeout` is reached. Logs each poll attempt with `description`. Returns True if fn() returned True within timeout, False otherwise. ALL polling in the framework should use this function — do not write custom while loops.

### src/patroni_ha_tests/recovery.py

Every test must return the cluster to a healthy state before the next test begins. No manual intervention allowed.

- `recover_cluster(max_wait: int = 300) -> RecoveryResult` — the master recovery function. Performs the following steps in order:
  1. **Clean network state:** Remove all test-related iptables rules, blackhole routes, and tc qdiscs on all 3 sites.
  2. **Resume WAL replay:** Run `SELECT pg_wal_replay_resume()` on both PG sites (safe no-op on primary).
  3. **Unfreeze Patroni:** Send `SIGCONT` to any stopped Patroni processes on both PG sites: `kill -SIGCONT $(pgrep -f patroni) 2>/dev/null`. Safe no-op if Patroni isn't stopped.
  4. **Resume Patroni:** If paused, run `patronictl resume` on both PG sites.
  5. **Restore watchdog state:** If `config.watchdog.enabled`:
     - Restore `/dev/watchdog` permissions: `chmod 660 /dev/watchdog && chown postgres /dev/watchdog` on both PG sites.
     - Restore watchdog mode to `config.watchdog.current_mode`: `patronictl edit-config -s 'watchdog.mode={current_mode}' --force`.
     - Restore `master_stop_timeout` if it was changed: read the current value from `patronictl show-config`, compare with default, reset if different.
  6. **Restart etcd:** Check each etcd node's health. Restart any that are down via `systemctl start etcd`.
  7. **Wait for etcd quorum:** Poll until `etcd_client.is_quorum_healthy()` returns True (timeout: 60s).
  8. **Restart Patroni:** Check Patroni health on both PG sites. Restart any that are down via `systemctl start patroni`.
  9. **Wait for convergence:** Poll until exactly one primary and one replica exist (timeout: 120s).
  10. **Handle stuck DCS state:** If after 120s there is no primary, or there are two primaries:
     - Two primaries: stop Patroni on the node with the higher replication lag, stop its PG, run `patronictl reinit` on that node, restart Patroni. It will re-initialize from the real primary via pg_basebackup.
     - No primary: check if Patroni is stuck. Run `patronictl reinit <cluster> <stale-node>` on the node that should be the replica, then restart Patroni on both nodes.
  9. **Wait for replication:** Poll until replication lag drops below 1000 bytes (timeout: 60s).
  10. **Truncate test tables:** `TRUNCATE ha_test_markers, ha_test_workload` on the primary.
  11. **Final health check:** Verify one primary, one replica, etcd quorum, replication flowing.
- Returns `RecoveryResult(success: bool, steps_taken: list[str], duration_seconds: float, error: Optional[str])`.
- If recovery fails after `max_wait`, log the full state of all components and raise `ClusterRecoveryError` with diagnostic details. The test suite should abort at this point.

### src/patroni_ha_tests/report.py

- Collect results from all tests into a structured report.
- `TestResult` dataclass: test_number, test_name, category, passed (bool), rto_seconds (optional), data_loss_transactions (optional), data_loss_bytes (optional), lost_seq_values (optional list[int]), duration_seconds, notes, error (optional).
- `generate_json_report(results: list[TestResult], output_path: str)` — write a JSON report.
- `generate_junit_xml(results: list[TestResult], output_path: str)` — write JUnit XML for CI integration.
- Include a summary section: total passed, total failed, total skipped, overall verdict.

## Test Specifications

### Design Rules for All Tests

1. **No site assumptions.** Every test must call `patroni.discover_primary()` and `patroni.discover_replica()` at the start. No test may assume Site A is primary. All subsequent logic must use the dynamically discovered roles.
2. **Automated recovery.** Every test calls `recovery.recover_cluster()` in its teardown. If recovery fails, the test is marked as ERROR (not FAIL) and the suite aborts.
3. **Partition verification.** After applying any network partition, the test must call `partition.verify_partition()` to confirm the partition is effective before proceeding.
4. **No manual steps.** If a test creates a state that cannot be automatically recovered (e.g., a diverged cluster that pg_rewind cannot fix), the recovery module handles it — falling back to `patronictl reinit` or `pg_basebackup` as needed.
5. **Data loss precision.** Tests that measure data loss must use the full `committed_seqs` comparison, not just MAX(seq).
6. **Use `poll_until()` for all polling.** Never write custom `while True` polling loops. Always use `timing.poll_until(fn, timeout, interval, description)`. This ensures consistent logging, timeouts, and intervals across all tests.

Each test file should follow this structure:

```python
"""
Test N: <name>
Category: <graceful|unplanned|partition|edge_case>
Objective: <one line>
"""

import pytest
from patroni_ha_tests.report import TestResult

class TestNameHere:
    """Docstring with full test description."""

    @pytest.fixture(autouse=True)
    def setup_and_teardown(self, cluster_healthy, recovery):
        """Ensure cluster is healthy before test. Recover after."""
        yield
        recovery.recover_cluster()

    def test_main_scenario(self, ssh, patroni, pg, etcd, workload, timing, network, report):
        # Step 1: discover current roles — NEVER assume which site is primary
        primary_site = patroni.discover_primary()
        replica_site = patroni.discover_replica()

        # ... test implementation using primary_site/replica_site ...

        report.add(TestResult(
            test_number=N,
            test_name="...",
            category="...",
            passed=True/False,
            rto_seconds=...,
            data_loss_transactions=...,
            lost_seq_values=[...],
            duration_seconds=...,
            notes="..."
        ))
```

### conftest.py Fixtures

```python
import pytest
from patroni_ha_tests.config import Config, load_config
from patroni_ha_tests.ssh import SSHManager
from patroni_ha_tests.patroni_client import PatroniClient
from patroni_ha_tests.pg_client import PGClient
from patroni_ha_tests.etcd_client import EtcdClient
from patroni_ha_tests.workload import WorkloadGenerator
from patroni_ha_tests.network import NetworkPartition
from patroni_ha_tests.recovery import ClusterRecovery
from patroni_ha_tests.report import TestReport

def pytest_addoption(parser):
    parser.addoption("--config", action="store", default="config/topology.yaml",
                     help="Path to topology.yaml")

@pytest.fixture(scope="session")
def config(request):
    """Load topology.yaml from the --config CLI arg."""
    config_path = request.config.getoption("--config")
    return load_config(config_path)

@pytest.fixture(scope="session")
def ssh(config):
    """SSHManager connected to all sites via ssh_host. Closed after all tests."""
    mgr = SSHManager(config)
    yield mgr
    mgr.close_all()

@pytest.fixture(scope="session")
def patroni(config, ssh):
    """PatroniClient instance. Needs both config (for REST API URLs) and ssh (for patronictl)."""
    return PatroniClient(config, ssh)

@pytest.fixture(scope="session")
def pg(config, ssh):
    """PGClient instance. Runs setup_test_schema on first use."""
    client = PGClient(config, ssh)
    # Create test schema on whatever is currently the primary
    # This runs once at session start
    primary_site = None
    for site in config.pg_sites():
        try:
            conn = client.connect(site)
            if not client.is_in_recovery(conn):
                client.setup_test_schema(conn)
                primary_site = site
                conn.close()
                break
            conn.close()
        except Exception:
            continue
    if primary_site is None:
        pytest.fail("Could not connect to any primary to set up test schema")
    return client

@pytest.fixture(scope="session")
def etcd(config):
    """EtcdClient instance."""
    return EtcdClient(config)

@pytest.fixture(scope="function")
def workload(config, patroni):
    """Fresh WorkloadGenerator for each test. Stopped and cleaned on teardown."""
    # Discover current primary at workload creation time
    primary_site = patroni.discover_primary()
    gen = WorkloadGenerator(config, target_site=primary_site)
    yield gen
    gen.stop()  # always stop, even if test didn't call stop

@pytest.fixture(scope="session")
def report(config):
    """TestReport collector. Writes JSON + JUnit XML after all tests complete."""
    r = TestReport()
    yield r
    r.generate_json_report("reports/results.json")
    r.generate_junit_xml("reports/junit.xml")

@pytest.fixture(scope="function")
def network(ssh, config):
    """NetworkPartition factory. Verifies clean state on teardown."""
    partitions_created = []
    def create_partition(**kwargs):
        p = NetworkPartition(ssh, config, **kwargs)
        partitions_created.append(p)
        return p
    yield create_partition
    # Teardown: force-clean any partitions that weren't properly closed
    for p in partitions_created:
        p.force_cleanup()
    NetworkPartition.verify_all_clean(ssh, config)

@pytest.fixture(scope="function")
def recovery(ssh, patroni, etcd, pg, config):
    """Recovery module instance."""
    return ClusterRecovery(ssh, patroni, etcd, pg, config)

@pytest.fixture(scope="function")
def cluster_healthy(patroni, etcd, recovery):
    """
    Pre-test assertion: cluster must be healthy before each test starts.
    Checks: one primary, one replica, etcd quorum intact, replication lag < 1000 bytes.
    If not healthy, run recover_cluster() with a 3-minute timeout.
    If still not healthy after recovery, fail the test with full diagnostic output.
    """
    try:
        primary = patroni.discover_primary()
        replica = patroni.discover_replica()
        assert etcd.is_quorum_healthy(), "etcd quorum not healthy"
        status = patroni.get_cluster_status(primary)
        for r in status.replicas:
            assert r.lag_bytes < 1000, f"Replica {r.name} lag too high: {r.lag_bytes}"
    except (AssertionError, Exception) as e:
        logging.warning(f"Cluster not healthy pre-test: {e}. Running recovery...")
        result = recovery.recover_cluster(max_wait=180)
        if not result.success:
            pytest.fail(f"Cluster recovery failed: {result.error}")
```

### Test Details

Below is what each test does. The test steps are precise — implement them exactly as described.

**Test 0: Partition Watchdog Safety Test (Meta-Test)**
- Validates that the partition TTL/watchdog cleanup mechanism works.
1. Discover primary and replica.
2. Create a partition object directly (not as a context manager): `p = NetworkPartition(ssh, config, block_pairs=[(primary_site, replica_site)])`. Call `p.block()` manually — this applies the iptables rules and writes the timestamp file.
3. Override the timestamp file to be 11 minutes in the past: `ssh.run(site, f"echo {int(time.time()) - 660} > /tmp/patroni-ha-test-partition-timestamp")`. This makes the watchdog think the rules are stale.
4. Do NOT call `p.unblock()` or `p.__exit__()` — the rules remain in place, simulating a crashed test runner.
5. Wait 70 seconds (the cron-based watchdog runs every 60 seconds).
6. SSH to both hosts. Check: the iptables rules should have been auto-removed by the watchdog cron job.
7. Also verify: `NetworkPartition.verify_all_clean(ssh, config)` returns True.
8. Pass if: rules were cleaned up automatically without any explicit unblock call.
- This test MUST run first in the suite (use `@pytest.mark.meta` and configure pytest ordering). If it fails, no partition tests are safe to run.

**Test 1: Planned Switchover — Healthy Cluster**
1. Discover current primary and replica via `patroni.discover_primary()`.
2. Insert marker row `T1-{uuid}` on primary.
3. Wait for marker to appear on replica (poll with 1s interval, 30s timeout).
4. Execute `patronictl switchover` from primary to replica.
5. Wait for the old replica to become primary (poll Patroni REST, 120s timeout).
6. Verify marker `T1` exists on new primary.
7. Verify old primary is now replica.
8. Insert marker `T1-verify` on new primary. Verify it replicates to old primary (now replica).
9. Pass if: all markers present, roles correct, replication flowing in new direction.

**Test 2: Hybrid Sync Switchover**
1. Discover primary and replica.
2. Start workload generator at 100 TPS.
3. Run for 10 seconds to establish a baseline.
4. Execute `patronictl edit-config -s 'synchronous_mode=true'`.
5. Poll `patronictl list` until the replica shows `Sync Standby` state (timeout: 60s).
6. Call `workload.record_kill_point()` to snapshot the current committed state.
7. Execute planned switchover.
8. Wait for new primary to accept writes.
9. Execute `patronictl edit-config -s 'synchronous_mode=false'`.
10. Stop workload generator.
11. Measure data loss using `timing.measure_data_loss()`.
12. Pass if: zero committed transactions lost. Every seq in `committed_seqs` up to `kill_seq` exists on the new primary.

**Test 3: Switchover and Immediate Switchback**
1. Discover primary and replica.
2. For i in range(3):
   a. Switchover current_primary → current_replica.
   b. Wait for roles to swap.
   c. Insert marker `T3-{i}` on new primary.
   d. Verify marker replicates to new replica.
   e. Check Patroni logs (via SSH grep) for "pg_rewind" — it should appear, NOT "pg_basebackup".
   f. Update primary/replica variables for the next iteration.
3. Pass if: all 3 round-trips succeed, all markers present, pg_rewind used (not pg_basebackup).

**Test 4: Kill Primary Process**
1. Discover primary and replica.
2. Start workload generator at 100 TPS.
3. Run for 5 seconds.
4. Call `workload.record_kill_point()`.
5. Via SSH to primary: `kill -9 $(head -1 <pg_data_dir>/postmaster.pid)` (use `config.patroni.pg_data_dir`).
6. Start RTO timer.
7. Wait for any site to become primary via `timing.measure_failover_rto()` (120s timeout).
8. Stop RTO timer.
9. Stop workload generator.
10. Measure data loss via `timing.measure_data_loss()`.
11. Restart Patroni on old primary (via SSH: `systemctl start patroni`).
12. Wait for old primary to rejoin as replica (poll Patroni REST, 180s timeout).
13. Pass if: failover completed within TTL (60s) + margin, data loss within `maximum_lag_on_failover`, old primary rejoined.

**Test 5: Hard Power-Off Primary Host**
- Skip if `hypervisor` section is not configured in topology.yaml.
1. Discover primary and replica.
2. Same as Test 4 steps 2-4.
3. Power off primary VM via hypervisor API.
4. Same as Test 4 steps 6-10.
5. Power on the VM via hypervisor API.
6. Wait for Patroni to start and the node to rejoin as replica (longer timeout: 300s, since VM boot takes time).
7. Pass criteria: same as Test 4.
- Note: Set pytest timeout to 600s.

**Test 6: Primary Under Write Load During Kill**
1. Discover primary and replica.
2. Run 3 iterations at different TPS targets: [10, 500, 0] where 0 = unlimited/max throughput.
3. For each TPS:
   a. Start workload generator at target TPS.
   b. Run for 60 seconds to establish steady state.
   c. Call `workload.record_kill_point()`.
   d. Kill primary process (same as Test 4 step 5).
   e. Wait for failover via `timing.measure_failover_rto()`.
   f. Measure data loss via `timing.measure_data_loss()`.
   g. Record: `{tps_target, actual_tps, lost_transactions, lost_seq_values, lost_bytes_estimate}`.
   h. Run `recovery.recover_cluster()` to reset between iterations.
4. Output: a data loss profile table mapping TPS to observed loss.
5. Pass if: all 3 iterations complete and data loss is within `maximum_lag_on_failover` for each.

**Test 7: Partition Between PG Sites, Witness Reachable**
1. Discover primary and replica. Record which site is primary before the partition.
2. Start workload generator at 50 TPS.
3. Apply partition between the two PG sites (NOT the witness): `NetworkPartition(block_pairs=[(primary_site, replica_site)])`.
4. Call `partition.verify_partition()` — confirm the two PG sites cannot communicate on service ports.
5. Wait for `partition_hold_time` (120s).
6. Check both sites' Patroni REST: exactly one should respond 200 to /primary.
7. The site that held the etcd leader lock before the partition should remain primary.
8. Use `pg.can_write(primary_site)` — should return True.
9. Use `pg.can_write(other_site)` — should return False.
10. Remove partition.
11. Wait for the demoted node to rejoin as replica.
12. Pass if: exactly one primary at all times, no split brain, demoted side refused writes.

**Test 8: Current Primary Fully Isolated**
1. Discover primary and replica.
2. Apply partition isolating the primary from both other sites: `NetworkPartition(block_pairs=[(primary_site, replica_site), (primary_site, "site_c")])`.
3. Call `partition.verify_partition()` for both pairs.
4. Wait 120s.
5. Check replica site: should now be primary (replica_site + site_c have quorum).
6. Check old primary site: should be demoted (only 1/3 etcd nodes, no quorum).
7. `pg.can_write(replica_site)` — True. `pg.can_write(primary_site)` — False.
8. Remove partition.
9. Wait for old primary to rejoin via pg_rewind.
10. Pass if: old replica promoted, old primary demoted, old primary rejoined after healing.

**Test 9: Witness Unreachable from One PG Site**
1. Discover primary and replica.
2. Apply partition between the replica and the witness: `NetworkPartition(block_pairs=[(replica_site, "site_c")])`.
3. Call `partition.verify_partition()`.
4. Wait 120s.
5. Check: primary should be unchanged. Replication should continue. No failover should occur.
6. Insert marker on primary, verify it replicates to replica.
7. Remove partition.
8. Pass if: no state change, no failover, cluster operated normally throughout.

**Test 10: Witness Down Then PG-Site Partition**
1. Discover primary and replica.
2. Kill etcd on Site C: via SSH `systemctl stop etcd`.
3. Wait 15s. Verify cluster still works (primary_site + replica_site etcd = 2/3 quorum).
4. Apply partition between the two PG sites: `NetworkPartition(block_pairs=[(primary_site, replica_site)])`.
5. Call `partition.verify_partition()`.
6. Wait 120s.
7. Check BOTH PG sites: neither should be primary. Both Patroni instances should have demoted.
8. `pg.can_write(primary_site)` — False. `pg.can_write(replica_site)` — False.
9. Restart etcd on Site C: `systemctl start etcd`.
10. Remove partition.
11. Wait for etcd quorum to reform and Patroni to elect a primary.
12. Pass if: during double failure, no primary existed. After recovery, exactly one primary.

**Test 11: Failover Blocked by Lag Threshold**
1. Discover primary and replica.
2. On the replica, pause WAL replay: `SELECT pg_wal_replay_pause()`.
3. Start workload at 200 TPS on primary for 30 seconds to build up replay lag.
4. Verify lag exceeds `maximum_lag_on_failover` (check via `patronictl list`).
5. Call `workload.record_kill_point()`. Kill primary process.
6. Wait 120s.
7. Check: replica should NOT have promoted. Parse Patroni logs on replica for "no suitable candidate" or equivalent message.
8. **Automated recovery from this state:**
   a. On replica: `SELECT pg_wal_replay_resume()`.
   b. Wait for replay to catch up (the WAL is already received, just not applied). Timeout: 60s.
   c. After lag drops below threshold, Patroni should auto-promote the replica (since the primary is still dead). Wait for promotion (timeout: 120s).
   d. If Patroni doesn't auto-promote (it may need a fresh primary failure detection cycle), restart Patroni on the replica to trigger re-evaluation.
   e. Once a primary exists, restart Patroni on the old primary to trigger pg_rewind rejoin.
9. Pass if: no promotion occurred while lag exceeded threshold, AND the cluster recovered to healthy after replay resumed.

**Test 12: pg_rewind After Timeline Divergence**
1. Discover primary and replica.
2. Start workload at 50 TPS on primary.
3. Apply partition between the two PG sites: `NetworkPartition(block_pairs=[(primary_site, replica_site)])`.
4. **Verify partition is effective:** call `partition.verify_partition()`.
5. Insert 5 marker rows on the current primary that will NOT replicate (partition is active).
6. Record these marker IDs.
7. **Wait for the replica to promote.** The replica can reach the witness, so replica + witness = 2/3 quorum. The old primary's lock will expire after TTL. Poll `patroni.get_node_role(replica_site)` until it becomes "primary" (timeout: TTL + 30s = 90s). If the replica doesn't promote within this window, fail the test — the partition didn't achieve the expected failover.
8. After replica promotes: the old primary is now on an old timeline with diverged commits.
9. Remove partition.
10. Watch Patroni on old primary: it should detect divergence and run pg_rewind. Wait for old primary to rejoin as replica (timeout: `pg_rewind_max_wait` = 180s).
11. Query old primary (now replica) for the diverged marker rows — they should be GONE.
12. Check Patroni logs on old primary for "pg_rewind" execution.
13. Pass if: old primary rejoined via pg_rewind (not pg_basebackup), diverged rows are gone, logs confirm pg_rewind.

**Test 13: etcd Leader Change Mid-PG-Failover**
1. Discover primary and replica.
2. Identify the current etcd leader via `etcd.get_leader()`.
3. Start workload at 50 TPS.
4. Call `workload.record_kill_point()`. Kill PG primary process.
5. Within 2 seconds (use `time.sleep(1)` then execute), kill the etcd leader process via SSH: `kill -9 <etcd_pid>`.
6. Wait 120s.
7. Check: etcd should have elected a new leader from the remaining 2/3 nodes.
8. Verify: exactly one PG primary exists, OR no primary exists (if failover was aborted). Split brain is the only failure condition.
9. Restart the killed etcd node.
10. If PG failover didn't complete, wait for it to complete on the next Patroni loop.
11. Pass if: no split brain. Failover either completed or aborted cleanly.

**Test 14: Pause Mode Split Brain (Risk Documentation Test)**
- This test intentionally creates a split brain to prove the risk is real. The expected outcome is that split brain OCCURS.
1. Discover primary and replica.
2. Execute `patronictl pause`.
3. Apply partition: `NetworkPartition(block_pairs=[(primary_site, replica_site)])`.
4. Call `partition.verify_partition()`.
5. Via SSH to replica: `pg_ctl promote -D <pg_data_dir>` (manually promote the replica while Patroni is paused).
6. Wait 30s.
7. `pg.can_write(primary_site)` — should return True (old primary still thinks it's primary).
8. `pg.can_write(replica_site)` — should return True (manually promoted).
9. Both sides accepting writes = split brain confirmed. Record this as evidence.
10. Remove partition.
11. Execute `patronictl resume`.
12. **Automated recovery from split brain:**
    a. Wait 30s for Patroni to detect the inconsistency.
    b. Check if Patroni auto-resolves (it should detect two primaries and demote one).
    c. If after 60s there are still two primaries (Patroni may be confused because the manual promote bypassed DCS):
       - Stop Patroni on the replica_site: `systemctl stop patroni`.
       - Stop PG on replica_site: via SSH `pg_ctl stop -D <pg_data_dir>`.
       - Remove the replica_site's Patroni DCS key: `patronictl -c <config> remove <cluster-name> <replica-node-name>`.
       - Restart Patroni on replica_site. It will re-initialize from the primary via pg_basebackup.
    d. Wait for cluster to converge to one primary + one replica.
13. Pass if: split brain was demonstrated, AND automated recovery succeeded.

**Test 15: Cascading PG + etcd Failure**
1. Discover primary and replica.
2. Start workload at 50 TPS.
3. Call `workload.record_kill_point()`.
4. Simultaneously kill PG process AND etcd process on the primary site.
5. Wait for failover via `timing.measure_failover_rto()`.
6. Check: etcd on replica_site + site_c = 2/3 quorum (still healthy). Replica should promote.
7. Measure data loss.
8. Restart PG and etcd on old primary site.
9. Wait for both to rejoin their respective clusters.
10. Pass if: failover succeeded, etcd quorum maintained, old primary fully recovers.

**Test 16: Kill Replica Process**
1. Discover primary and replica.
2. Start workload at 100 TPS on the primary.
3. Via SSH to replica: `kill -9 $(head -1 <pg_data_dir>/postmaster.pid)`.
4. Wait 30s. Continue writing to primary throughout.
5. Check: primary should remain primary. No role change. Writes continue uninterrupted.
6. Check Patroni REST on primary: cluster status should show replica as down/unknown.
7. Restart Patroni on the replica (via SSH: `systemctl start patroni`).
8. Wait for replica to rejoin and begin streaming replication (poll Patroni REST, 180s timeout).
9. Verify: replica catches up (lag drops below 1000 bytes). All marker rows written during the outage are present on the replica.
10. Stop workload.
11. Pass if: primary was never affected, no failover occurred, replica rejoined and caught up with all data intact.

**Test 17: Hard Power-Off Replica Host**
- Skip if `hypervisor` section is not configured in topology.yaml.
1. Discover primary and replica.
2. Start workload at 100 TPS on the primary.
3. Power off the replica VM via hypervisor API.
4. Wait 60s. Continue writing to primary.
5. Check: primary should remain primary. Writes continue.
6. Insert a marker `T17-during-outage` on primary.
7. Power on the replica VM via hypervisor API.
8. Wait for Patroni to start and the replica to rejoin (300s timeout — VM boot time).
9. Wait for replication to catch up.
10. Verify marker `T17-during-outage` exists on the replica.
11. Stop workload. Verify all workload rows are present on replica.
12. Pass if: primary unaffected, no failover, replica recovered all data including writes made during its outage.
- Note: Set pytest timeout to 600s.

**Test 18: Network Partition via Host Routes (Blackhole)**
- Skip if `ssh_host == pg_host` for any PG site (blackhole routes would block SSH).
1. Discover primary and replica.
2. Start workload at 50 TPS.
3. Apply partition using route method: `NetworkPartition(block_pairs=[(primary_site, replica_site)], method="route")`.
4. Call `partition.verify_partition()`.
5. Wait 120s.
6. Check: which site is primary? The etcd lock holder should remain primary.
7. `pg.can_write(primary)` — True. `pg.can_write(other)` — False.
8. Remove partition.
9. Wait for demoted node to rejoin.
10. Compare behavior with Test 7 results: record whether detection was faster/slower, whether etcd leader election behavior differed, and any differences in Patroni log messages.
11. Pass if: same correctness as Test 7 (one primary, no split brain, clean rejoin). The deliverable is also a comparison note.

**Test 19: Full Mesh Partition — Every Site Isolated**
1. Discover primary and replica.
2. Apply partition: `NetworkPartition(block_pairs=[(primary_site, replica_site), (primary_site, "site_c"), (replica_site, "site_c")])`.
3. Call `partition.verify_partition()` for all three pairs.
4. Wait 120s.
5. Each etcd node can only see itself (1/3). No quorum anywhere.
6. Verify: Patroni on BOTH PG sites should demote. No primary should exist.
7. `pg.can_write(primary_site)` — False. `pg.can_write(replica_site)` — False.
8. Remove partition (all three pairs).
9. Wait for etcd to re-form quorum.
10. Wait for Patroni to elect a primary.
11. Verify: exactly one primary, one replica, healthy cluster.
12. Pass if: during total partition, no primary existed. After recovery, cluster converges.

**Test 20: Asymmetric Partition (Route + iptables Combined)**
- Skip if `ssh_host == pg_host` for either PG site.
1. Discover primary and replica.
2. On primary site: `ip route add blackhole <replica_pg_host>/32` (primary→replica gets immediate errors).
3. On replica site: apply iptables DROP rules for primary's IP on service ports (replica→primary packets silently dropped).
4. Call `partition.verify_partition()` from both directions.
5. Wait 120s.
6. Verify: one primary, no split brain. Check Patroni logs on both sides for different log patterns.
7. Clean up both sides (route on primary, iptables on replica).
8. Wait for cluster recovery.
9. Pass if: one primary, no split brain. Deliverable includes a log comparison.

**Test 21: Lagging Replica Promoted After Primary Dies (Stale Replica Failover)**
- Uses `tc netem` to throttle replication bandwidth for deterministic lag control.
- **Run twice as sub-tests:** once forcing lag UNDER the threshold (should promote), once OVER (should block).

**Sub-test 21a: Small gap (should promote)**
1. Discover primary and replica.
2. Start workload at 100 TPS on primary.
3. Stop Patroni on replica: `systemctl stop patroni`.
4. Continue writing to primary for 5 seconds (small gap, should be well under 1MB).
5. Start Patroni on replica: `systemctl start patroni`.
6. **Throttle replication bandwidth** on the replica to slow catch-up: via SSH to replica, run `tc qdisc add dev <iface> root netem rate 10kbit` (auto-detect interface via `ip route get <primary_pg_host>`).
7. Wait 5 seconds for Patroni to register the replica as connected but lagging.
8. Call `workload.record_kill_point()`. Kill primary process.
9. Wait 120s. Patroni should evaluate the replica's lag. Since the gap is small (well under 1MB), the replica should be promoted.
10. Remove the `tc` throttle: `tc qdisc del dev <iface> root`.
11. Measure data loss via `timing.measure_data_loss()`.
12. Pass if: replica was promoted, data loss bounded by the gap size.

**Sub-test 21b: Large gap (should block)**
1. Same setup as 21a, but write for 120 seconds while replica is stopped (much more data, should exceed 1MB threshold).
2. Start Patroni on replica.
3. Apply same `tc` throttle.
4. Wait 5 seconds.
5. Kill primary.
6. Wait 120s. Patroni should BLOCK failover because the replica's lag exceeds `maximum_lag_on_failover`.
7. Parse Patroni logs for the blocking message.
8. Remove `tc` throttle. Wait for replica to catch up.
9. After lag drops below threshold, Patroni should auto-promote. If not, restart Patroni on replica to trigger re-evaluation.
10. Restart old primary. Wait for recovery.
11. Pass if: failover was initially blocked, then succeeded after lag dropped.

**IMPORTANT for Test 21 implementation:** Determine what Patroni uses for `maximum_lag_on_failover` — is it receive lag (bytes received from primary but not yet applied) or replay lag (bytes not yet received)? If Patroni checks receive lag, throttling the network via `tc netem` is correct. If it checks replay lag, use `pg_wal_replay_pause()` instead (like Test 11). Check the Patroni source code: look at how `maximum_lag_on_failover` is evaluated in `patroni/ha.py`. Document the finding in the test's docstring.

### Watchdog Tests (22-25)

These tests validate Patroni's Linux kernel watchdog integration. They require the `softdog` kernel module loaded on both PG hosts with the Patroni user having write access to `/dev/watchdog`.

**CRITICAL prerequisite:** For tests that intentionally trigger the watchdog (Test 23, 24), the softdog module MUST be loaded with `soft_noboot=1`:
```bash
modprobe softdog soft_noboot=1
```
This makes the watchdog log to the kernel ring buffer (`dmesg`) instead of actually rebooting the machine. Without `soft_noboot=1`, Test 23 will hard-reboot your primary server. The test framework checks `config.watchdog.soft_noboot` and refuses to run tests 23/24 if it's set to `false`.

For tests that verify the watchdog does NOT fire (Test 22, 25), `soft_noboot` doesn't matter — the watchdog shouldn't trigger at all.

**Test 22: Watchdog Does Not Fire During Graceful Demotion**
- Verifies that when Patroni loses etcd connectivity but is still alive and functioning, it gracefully demotes PG and disables the watchdog before the watchdog timer expires. No reboot occurs.
- Skip if `config.watchdog.enabled` is false.
1. Discover primary and replica.
2. Verify the watchdog is active on the primary: via SSH check that Patroni logs show "watchdog activated" or check `/dev/watchdog` is open (via `lsof /dev/watchdog`).
3. Record the system uptime on the primary: `cat /proc/uptime`.
4. **Record dmesg baseline:** via SSH, run `dmesg --time-format iso | tail -1` and save the timestamp. After the test, only check dmesg entries AFTER this timestamp to avoid false positives from previous runs.
5. Start workload at 50 TPS on the primary.
5. Apply partition isolating the primary from BOTH etcd peers: `NetworkPartition(block_pairs=[(primary_site, replica_site), (primary_site, "site_c")])`. This cuts the primary off from etcd quorum — it cannot renew its leader lock.
6. Call `partition.verify_partition()`.
7. Now the race begins: Patroni has `ttl - safety_margin` seconds (55s with your config) to demote PG before the watchdog fires.
8. Wait 90 seconds (well past the watchdog timeout).
9. Check system uptime on the primary again: `cat /proc/uptime`. Compare with step 3.
10. **The machine should NOT have rebooted.** Uptime should be continuous (current uptime > previous uptime, no reset).
11. Check `dmesg` on the primary for watchdog trigger messages — there should be NONE.
12. Check Patroni logs on the primary: should show "leader lock lost" → "demoting PostgreSQL" → watchdog disabled. The graceful path completed before the watchdog timeout.
13. Verify PG on the primary is stopped or in recovery mode (not accepting writes).
14. Remove partition.
15. Wait for cluster recovery.
16. Pass if: no reboot occurred, Patroni gracefully demoted PG, watchdog was disabled cleanly, `dmesg` shows no watchdog timeout events.
- **Deliverable:** Record the exact time between lock renewal failure and PG demotion completion. This is the "grace period" — the time Patroni took to handle the situation. Document: `watchdog_timeout - grace_period = safety_margin_actual`. If this margin is thin (< 5 seconds), the config needs tuning.

**Test 23: Watchdog Fires When Patroni Is Stuck (SIGSTOP)**
- Verifies that the watchdog fires when Patroni cannot execute its demotion logic. Uses `SIGSTOP` to freeze the Patroni process, simulating a stuck/hung process.
- Skip if `config.watchdog.enabled` is false.
- Skip if `config.watchdog.soft_noboot` is false (would cause actual reboot).
1. Discover primary and replica.
2. Verify the watchdog is active on the primary.
3. Start workload at 50 TPS.
4. Freeze the Patroni process on the primary: `kill -SIGSTOP $(pgrep -f patroni)`. This suspends Patroni — it's alive but cannot execute any code. It can't demote PG, can't send watchdog keepalives, can't do anything.
5. PG is still running and accepting writes (Patroni being stopped doesn't stop PG).
6. Wait for `ttl + 30 seconds` (90s with your config). The watchdog timeout (55s) should have expired.
7. Check `dmesg` on the primary for watchdog trigger messages. With `soft_noboot=1`, you should see a log entry like "SoftDog: Triggered - Reboot not needed (soft_noboot)" or similar.
8. **This is the proof:** the watchdog detected that keepalives stopped and would have rebooted the machine in production.
9. Meanwhile, check the replica: it should have promoted (the DCS lock expired at 60s, the replica acquired it).
10. Unfreeze Patroni on the primary: `kill -SIGCONT $(pgrep -f patroni)`.
11. Patroni resumes, discovers it's no longer the leader, and demotes PG.
12. Wait for the primary to rejoin as a replica.
13. Pass if: `dmesg` shows watchdog trigger, replica promoted, and the old primary rejoined after being unfrozen.
- **Note:** In production (without `soft_noboot`), the machine would have rebooted at step 7. With `soft_noboot=1`, we can observe the watchdog trigger without losing the machine, which is why this flag is required for this test.

**Test 24: Watchdog Race — Slow PG Shutdown**
- Tests the race condition documented in Patroni's docs: what happens when Patroni starts demoting PG after losing the lock, but PG shutdown takes too long and the watchdog fires before demotion completes.
- Skip if `config.watchdog.enabled` is false.
- Skip if `config.watchdog.soft_noboot` is false.
1. Discover primary and replica.
2. Verify the watchdog is active on the primary.
3. **Create conditions for a slow PG shutdown:** Start a long-running transaction on the primary that will delay shutdown. Via psql: `BEGIN; SELECT pg_advisory_lock(1); SELECT pg_sleep(300);` — this holds a lock and sleeps for 5 minutes. When PG tries to shut down, it will wait for this session to terminate (up to the configured `shutdown_timeout` or `stop_timeout` in Patroni).
4. Also increase PG's `checkpoint_timeout` or dirty a lot of shared buffers to make the checkpoint during shutdown slower. Alternative: set Patroni's `master_stop_timeout` to a very high value (e.g., 300 seconds) so Patroni waits a long time for PG to stop before force-killing it.
5. Apply partition isolating the primary from etcd (same as Test 22 step 5).
6. Call `partition.verify_partition()`.
7. Patroni will detect the lock loss and attempt to demote PG. But PG shutdown will be slow because of the long-running transaction.
8. Wait 90 seconds.
9. Check `dmesg` for watchdog trigger messages.
10. **Expected outcome depends on the timing:**
    - If Patroni's `master_stop_timeout` is shorter than the watchdog timeout (55s), Patroni will `kill -9` PG before the watchdog fires. Watchdog does NOT trigger. This is the "safe" config.
    - If `master_stop_timeout` is longer than the watchdog timeout, PG is still shutting down when the watchdog fires. Watchdog DOES trigger (visible in `dmesg` with `soft_noboot=1`).
11. Record which outcome occurred and the exact timing.
12. Clean up: terminate the long-running psql session (if PG is still up), remove the partition, unfreeze anything stuck.
13. Wait for cluster recovery.
14. Pass if: the test completed and the outcome matches the expected behavior based on `master_stop_timeout` vs. watchdog timeout. Document the race timeline.
- **Deliverable:** A timeline showing: lock lost at T=0, Patroni starts shutdown at T=X, PG shutdown stalls at T=Y, watchdog would fire at T=55. This documents the exact config relationship between `master_stop_timeout` and the watchdog timeout that determines whether the graceful path or the watchdog path wins.
- **Config recommendation produced by this test:** `master_stop_timeout` MUST be less than `ttl - safety_margin - loop_wait` (i.e., less than 40s with your config). If it's not, the watchdog can fire during what is otherwise a graceful (but slow) demotion.

**Test 25: Watchdog Required Mode — Node Refuses Primary Without Device**
- Verifies that with `watchdog.mode: required`, a node refuses to become primary if `/dev/watchdog` is not accessible.
- Skip if `config.watchdog.enabled` is false.
1. Discover primary and replica.
2. On the replica, remove access to the watchdog device: `chmod 000 /dev/watchdog` (or rename it: `mv /dev/watchdog /dev/watchdog.bak`).
3. Temporarily set Patroni's watchdog mode to `required` on the replica: `patronictl edit-config -s 'watchdog.mode=required'` (or edit the local config and restart Patroni).
4. Trigger a switchover: try to make the replica (which now has no watchdog access) become primary.
5. **Expected:** The switchover should fail. The replica should refuse to accept the primary role because watchdog activation will fail and mode is `required`.
6. Check Patroni logs on the replica for a message about watchdog activation failure.
7. Verify: the original primary is still primary. The switchover was rejected.
8. **Restore:**
   a. Restore watchdog device access: `chmod 660 /dev/watchdog` and `chown postgres /dev/watchdog` (or `mv /dev/watchdog.bak /dev/watchdog`).
   b. Restore watchdog mode to original: `patronictl edit-config -s 'watchdog.mode=automatic'` (or whatever `config.watchdog.current_mode` was).
9. Verify cluster is healthy with original roles.
10. Pass if: the replica refused to promote, citing watchdog activation failure. This proves that `required` mode prevents a node from becoming primary without the safety net.

## Environment Reset Script (scripts/reset_environment.sh)

Between tests, the cluster may be in a degraded state. The reset script should:
1. Resume Patroni if paused (`patronictl resume` on both PG sites).
2. Unfreeze any stopped Patroni processes: `kill -SIGCONT $(pgrep -f patroni) 2>/dev/null` on both PG sites.
3. Remove any stale iptables rules (find rules with `iptables -S | grep "patroni-ha-test"`, delete each with `iptables -D`).
4. Remove any stale blackhole routes (`ip route show type blackhole` — delete any that match test host IPs).
5. Remove any `tc` qdiscs added by tests (`tc qdisc del dev <iface> root` on all hosts — safe to run even if no qdisc exists).
6. Resume WAL replay if paused (`SELECT pg_wal_replay_resume()` on both PG sites — safe no-op on primary).
7. Restore `/dev/watchdog` permissions if changed: `chmod 660 /dev/watchdog && chown postgres /dev/watchdog` on both PG sites (safe if watchdog isn't configured).
8. Restart etcd on all 3 sites if any are down.
9. Restart Patroni on both PG sites if either is down.
10. Wait for etcd quorum.
11. Wait for Patroni to converge (one primary, one replica).
12. If no primary after 120s, attempt `patronictl reinit` on the stale node.
13. Wait for replication lag to drop below 1000 bytes.
14. Truncate test tables (`ha_test_markers`, `ha_test_workload`).
15. Exit 0 if healthy, exit 1 if recovery failed after 5 minutes.

This logic is also implemented in `recovery.py` and callable as a pytest fixture, but having a standalone script is useful for manual recovery if the test suite itself crashes.

## Partition Watchdog Script (scripts/partition_watchdog.sh)

Deploy on each host (Site A, B, C) via cron (`* * * * *` — runs every minute). This is the nuclear safety net for leaked partitions.

```bash
#!/bin/bash
# partition_watchdog.sh — remove stale test partitions
# Deploy via cron: * * * * * /usr/local/bin/partition_watchdog.sh >> /var/log/partition-watchdog.log 2>&1

TIMESTAMP_FILE="/tmp/patroni-ha-test-partition-timestamp"
MAX_AGE_SECONDS=600  # 10 minutes

# Clean stale iptables rules
STALE_RULES=$(iptables -S | grep "patroni-ha-test")
if [ -n "$STALE_RULES" ]; then
    if [ -f "$TIMESTAMP_FILE" ]; then
        AGE=$(( $(date +%s) - $(cat "$TIMESTAMP_FILE") ))
        if [ "$AGE" -gt "$MAX_AGE_SECONDS" ]; then
            echo "$(date): Removing stale iptables rules (age: ${AGE}s)"
            echo "$STALE_RULES" | while read -r rule; do
                DELETE_RULE=$(echo "$rule" | sed 's/^-A/-D/')
                iptables $DELETE_RULE
            done
            rm -f "$TIMESTAMP_FILE"
        fi
    else
        # No timestamp file but rules exist — remove immediately (orphaned)
        echo "$(date): Removing orphaned iptables rules (no timestamp file)"
        echo "$STALE_RULES" | while read -r rule; do
            DELETE_RULE=$(echo "$rule" | sed 's/^-A/-D/')
            iptables $DELETE_RULE
        done
    fi
fi

# Clean stale blackhole routes
BLACKHOLE_ROUTES=$(ip route show type blackhole 2>/dev/null)
if [ -n "$BLACKHOLE_ROUTES" ]; then
    echo "$(date): Removing blackhole routes: $BLACKHOLE_ROUTES"
    echo "$BLACKHOLE_ROUTES" | while read -r route; do
        ip route del $route
    done
fi

# Clean stale tc qdiscs (netem)
TC_QDISCS=$(tc qdisc show 2>/dev/null | grep netem)
if [ -n "$TC_QDISCS" ]; then
    echo "$(date): Removing tc netem qdiscs"
    for iface in $(echo "$TC_QDISCS" | awk '{print $5}'); do
        tc qdisc del dev "$iface" root 2>/dev/null
    done
fi
```

## Running the Tests

```bash
# Full suite (runs all 25 tests + meta-test, fully unattended)
pytest tests/ -v --tb=short --timeout=600 --config=config/topology.yaml

# Single category
pytest tests/test_01_graceful/ -v

# Single test
pytest tests/test_03_partition/test_partition_ab_witness_up.py -v

# Generate report
pytest tests/ -v --json-report=reports/results.json --junitxml=reports/junit.xml

# Abort on first failure (useful for debugging)
pytest tests/ -v -x --timeout=600 --config=config/topology.yaml
```

## Implementation Notes

- **Port-specific iptables rules:** Partition tests block only the service ports listed in `config.partition_ports` (default: 5432, 2379, 2380, 8008). This leaves SSH (port 22) open so the test runner can always reach all hosts. The rule format is: `iptables -A INPUT -s <ip> -p tcp --dport <port> -j DROP -m comment --comment "patroni-ha-test"` (one rule per port per direction). The OUTPUT chain uses `--dport` as well for outgoing connections to the target.
- **iptables comment tagging:** Every iptables rule added by the framework MUST include `-m comment --comment "patroni-ha-test"`. Cleanup uses `iptables -S | grep "patroni-ha-test"` to find and remove all test-related rules.
- **Blackhole route safety:** `method="route"` blocks ALL traffic to the target IP including SSH. The framework MUST reject this method if `ssh_host == pg_host` for the affected sites. Always prefer `method="iptables"` unless the test specifically needs immediate-error (ENETUNREACH) behavior.
- **Partition verification:** After every `partition.block()` call, the test must call `partition.verify_partition()` before proceeding. This confirms the partition is effective by attempting a TCP connect on a service port.
- **Partition watchdog:** Deploy `scripts/partition_watchdog.sh` on all 3 hosts via cron BEFORE running the test suite. This is non-negotiable — it's the safety net for leaked partitions. Test 0 validates it works.
- **tc netem cleanup:** Test 21 uses `tc qdisc add dev <iface> root netem rate 10kbit` to throttle replication. Auto-detect the interface: `ip route get <target_ip> | grep -oP 'dev \K\S+'`. Cleanup: `tc qdisc del dev <iface> root`. The watchdog also cleans stale netem qdiscs.
- **PG data directory path:** Read from `config.patroni.pg_data_dir`. If not set, parse the Patroni config on the host via SSH: `grep -A5 'postgresql:' /etc/patroni.yml | grep 'data_dir'`.
- **Patroni management method:** The framework assumes Patroni runs as a systemd service (`systemctl start/stop patroni`). If it runs differently (Docker, supervisor, etc.), make the service manager configurable via a `patroni.service_manager` config field.
- **No site assumptions:** Every test discovers the current primary/replica dynamically. No test hardcodes "Site A is primary."
- **Test isolation:** Each test is independent. `cluster_healthy` verifies health before each test. `recovery.recover_cluster()` runs in teardown after each test.
- **Timeouts:** All waits have explicit timeouts. Never poll infinitely.
- **Logging:** Use Python's `logging` module. Log every SSH command, every HTTP request to Patroni/etcd, and every state change. DEBUG for troubleshooting, INFO for normal runs.
- **Thread safety:** The workload generator runs in a separate thread. `committed_seqs` uses a `threading.Lock`. All stats accessors are thread-safe.
- **Error handling in network.py:** If `iptables -D` fails during cleanup (e.g., rule already removed), log a warning but don't raise. `verify_all_clean()` is the final safety check.
- **Data loss precision:** Tests use full `committed_seqs` set comparison, not just `MAX(seq)`. This catches gaps where a middle seq was lost but higher seqs survived.
- **Watchdog tests prerequisites:** Tests 22-25 require `softdog` kernel module loaded on both PG hosts. Tests 23-24 (which intentionally trigger the watchdog) require `soft_noboot=1` — without it, the machine will actually reboot. The framework checks this at test startup and skips with a clear message if the prerequisite isn't met. To set up: `modprobe softdog soft_noboot=1 && chown postgres /dev/watchdog`. To verify: `cat /sys/module/softdog/parameters/soft_noboot` should return `1`.
- **Watchdog test isolation:** Tests 24 and 25 modify Patroni config (`master_stop_timeout`, `watchdog.mode`) and system state (`/dev/watchdog` permissions). The teardown MUST restore original values. The recovery module should verify watchdog config is restored as part of its health check.

## Definition of Done

- All 25 tests + 1 meta-test implemented and running fully unattended against a test environment.
- Watchdog tests (22-25) require `softdog` module with `soft_noboot=1` on PG hosts. Tests skip cleanly if watchdog is not configured.
- No test requires manual intervention for recovery. Every test returns the cluster to a healthy state automatically.
- JSON report includes: test name, pass/fail, RTO (where applicable), data loss details (count, specific lost seqs, bytes), duration, notes.
- `scripts/reset_environment.sh` successfully recovers the cluster from any state left by any test.
- `scripts/partition_watchdog.sh` deployed on all 3 hosts and verified working (Test 0).
- No iptables rules, blackhole routes, or tc qdiscs leaked after any test run (verified by `network.verify_all_clean()`).
- README updated with actual test environment details and any deviations from this spec.
