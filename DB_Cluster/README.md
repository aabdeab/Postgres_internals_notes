# PostgreSQL Database Cluster Notes

## What is a cluster?
A PostgreSQL cluster is a single PostgreSQL server instance that manages multiple databases.

The cluster is initialized with `initdb`.

When a new cluster is created, PostgreSQL includes these default databases:
- `postgres`
- `template1`
- `template2`

## Cluster data location
`PGDATA` is the environment variable that points to the cluster data directory.

Inside `PGDATA`, you will find:
- `base/`: database-specific files (one subdirectory per database, named by OID)
- `global/`: cluster-wide system catalog files (for example metadata such as `pg_database`)
- configuration files
- WAL and other internal directories

![PGDATA folder layout overview](../images/schema.png)

_Figure: High-level layout of `PGDATA` and default database folders._

## OID basics
An OID (Object Identifier) is a unique numeric identifier assigned to PostgreSQL objects such as databases, tables, and indexes.

Many on-disk folder/file names in `PGDATA` use OIDs.

Example query to list database names and OIDs:

```sql
SELECT datname, oid FROM pg_database;
```

![OID lookup query example](../images/image0.png)

_Figure: Example query to inspect database OIDs._

## Important files and directories in `PGDATA`

![PGDATA items reference table](../images/image.png)

_Figure: Quick reference of common files/directories found in `PGDATA`._

### Top-level files

#### `PG_VERSION`
`PG_VERSION` stores the major version number of PostgreSQL for the entire cluster (for example `16`, `15`, `14`).

Why it matters:
- It is one of the first compatibility guards when PostgreSQL starts.
- All data files under `PGDATA` are expected to be in a format compatible with that major version.
- It prevents accidental startup of a data directory with the wrong server major version.

Operational notes:
- If this version and the binary version do not match, PostgreSQL refuses to start the cluster.
- A major upgrade changes internal on-disk formats, so tools like `pg_upgrade` are used instead of just replacing binaries.

#### `current_logfiles`
`current_logfiles` is a helper file written by the logging collector to indicate which log file(s) are currently active.

Why it matters:
- Useful for troubleshooting because it points directly to the log target currently receiving entries.
- Helps operators and automation avoid guessing the current file name when log rotation is enabled.

When it appears:
- Created only when `logging_collector = on`.
- It can be missing when the collector is disabled.

Container note (keeping your previous context):
- In many Docker/container setups, PostgreSQL logs are sent to standard output/error and consumed by the runtime (`docker logs`).
- In that setup, `current_logfiles` may be absent or less relevant.

#### `pg_hba.conf`
`pg_hba.conf` (Host-Based Authentication) controls who can connect, from where, to which database, as which role, and using which authentication method.

How to read a rule:
- Connection type (`local`, `host`, `hostssl`, `hostnossl`)
- Target database(s)
- Target user(s)
- Client address or network range (for host-based rules)
- Authentication method (`peer`, `md5`, `scram-sha-256`, `trust`, etc.)

Why it matters:
- This file is a primary security boundary for PostgreSQL access.
- A permissive entry can expose the cluster; an overly strict entry can block valid clients.

Operational notes:
- Rules are checked top to bottom; first match wins.
- Syntax or policy changes are typically applied with `SELECT pg_reload_conf();` or a server reload.

#### `pg_ident.conf`
`pg_ident.conf` defines user name maps used with authentication methods that rely on external identities.

What it does:
- Maps an external/system identity (for example OS user) to a PostgreSQL role.
- Works with mappings referenced from `pg_hba.conf`.

Why it matters:
- Enables controlled identity translation without creating one-to-one role names.
- Useful in environments where system/service account names differ from database role names.

#### `postgresql.conf`
`postgresql.conf` is the main configuration file for server behavior.

What is typically configured here:
- Memory and resource settings (`shared_buffers`, `work_mem`, `maintenance_work_mem`)
- WAL and checkpoint behavior (`wal_level`, `checkpoint_timeout`, `max_wal_size`)
- Autovacuum behavior
- Logging configuration
- Planner/runtime settings
- Replication and connection settings

Why it matters:
- Most performance tuning and operational behavior is controlled here.
- Bad values can reduce performance or stability; good values are workload-dependent.

Operational notes:
- Some parameters reload dynamically; others require full restart.
- Includes can split configuration into multiple files for cleaner operations.

#### `postgresql.auto.conf`
`postgresql.auto.conf` stores values written by `ALTER SYSTEM` (PostgreSQL 9.4+).

Why it matters:
- It is PostgreSQL-managed and intended for automated/system-level overrides.
- Settings in this file are loaded in addition to `postgresql.conf` and can override it.

Best practice:
- Avoid manually editing this file unless there is a controlled reason.
- Use `ALTER SYSTEM`, then reload/restart as required by each parameter.

#### `postmaster.opts`
`postmaster.opts` records the exact command-line options used when the PostgreSQL server (postmaster) was last started.

Why it matters:
- Helpful for incident analysis and reproducibility.
- Lets you verify startup-time options that may not be obvious from config files alone.

Typical usage:
- Confirm whether a custom socket directory, data path option, or other launch-time flag was applied.

#### `postmaster.pid` (important runtime file)
`postmaster.pid` is a lock/status file that exists while the server is running.

What it contains (high level):
- Postmaster process ID
- Data directory path
- Startup timestamp
- Port and socket details
- Shared memory identifiers and listener metadata

Why it matters:
- Prevents two PostgreSQL servers from starting on the same data directory.
- Gives immediate runtime metadata for diagnostics.

Caution:
- Do not remove this file manually unless you are absolutely sure no PostgreSQL process is using the data directory.

### Subdirectories

#### `base/`
Contains one subdirectory per database in the cluster.

Key point (keeping your earlier note):
- Folder names are OIDs, not human-readable database names.
- The mapping can be inspected from catalog tables, such as querying `pg_database`.

What is inside each database OID folder:
- Relation files for tables and indexes
- TOAST relation files for large values
- Fork files (main/fsm/vm) that store different storage aspects

#### `global/`
Cluster-wide storage area for shared catalogs and control metadata.

Key point (keeping your earlier note):
- Contains global objects such as information related to `pg_database`.
- Includes important control information (for example `pg_control`) used to track cluster state and WAL/checkpoint consistency.

Why it matters:
- Damage here affects the whole cluster, not just one database.

#### `pg_commit_ts/` (PostgreSQL 9.5+)
Stores commit timestamp data per transaction when commit timestamp tracking is enabled.

Why it matters:
- Supports auditing/debugging use cases that need commit-time visibility.
- Adds overhead in CPU/memory/I/O when enabled, so it is often left off unless required.

#### `pg_clog/` (PostgreSQL 9.6 or earlier)
Stores transaction commit status data in old versions.

Version history:
- Renamed to `pg_xact/` in PostgreSQL 10.

Why it matters:
- This status is critical for MVCC visibility checks and transaction lifecycle decisions.

#### `pg_dynshmem/` (PostgreSQL 9.4+)
Contains files used by PostgreSQL dynamic shared memory.

Why it matters:
- Supports features that allocate shared memory segments dynamically for parallel operations and internal coordination.

#### `pg_logical/` (PostgreSQL 9.4+)
Stores status data used by logical decoding.

Why it matters:
- Required for logical replication and CDC-style workflows.
- Holds metadata needed to decode WAL into logical change streams.

#### `pg_multixact/`
Stores multixact state used mainly for shared row locks.

Why it matters:
- Enables multiple transactions to hold lock state on a tuple.
- Important for concurrency correctness in lock-heavy workloads.

#### `pg_notify/`
Stores LISTEN/NOTIFY status data.

Why it matters:
- Supports asynchronous notification signaling between sessions.

#### `pg_replslot/` (PostgreSQL 9.4+)
Stores replication slot metadata and state.

Why it matters:
- Replication slots prevent required WAL from being removed before consumers read it.
- Mismanaged slots can cause WAL retention growth and disk pressure.

#### `pg_serial/` (PostgreSQL 9.1+)
Stores metadata for committed serializable transactions.

Why it matters:
- Used by Serializable Snapshot Isolation internals to preserve correctness guarantees.

#### `pg_snapshots/` (PostgreSQL 9.2+)
Stores exported snapshots created by `pg_export_snapshot`.

Why it matters:
- Allows other sessions to import and use a synchronized visibility snapshot.
- Useful in advanced consistency workflows and coordinated reads.

#### `pg_stat/`
Stores persistent statistics subsystem files.

Why it matters:
- Statistics influence query planning decisions.
- Persistent stats support stable optimizer behavior across restarts (subject to version behavior).

#### `pg_stat_tmp/`
Stores temporary statistics subsystem files.

Why it matters:
- Used for transient stats activity.
- Temporary-by-design area that complements persistent stats mechanisms.

#### `pg_subtrans/`
Stores subtransaction status data.

Why it matters:
- Needed to resolve parent/child transaction relationships for visibility and rollback behavior.

#### `pg_tblspc/`
Contains symbolic links to tablespace locations outside the main data directory.

Why it matters:
- Enables placing selected objects on different storage tiers/devices.
- Broken links or unavailable mount points can make dependent objects inaccessible.

#### `pg_twophase/`
Stores state files for prepared transactions (two-phase commit / 2PC).

Why it matters:
- Required for distributed transaction coordination patterns that rely on `PREPARE TRANSACTION`.
- Prepared transactions left open too long can hold locks/resources.

#### `pg_wal/` (PostgreSQL 10+)
Stores WAL (Write-Ahead Log) segment files.

Why it matters:
- WAL is central to crash recovery, replication, and durability guarantees.
- WAL volume strongly affects disk sizing, backup strategy, and retention planning.

Version history:
- In PostgreSQL 9.6 and earlier, this directory was named `pg_xlog/`.

#### `pg_xact/` (PostgreSQL 10+)
Stores transaction commit state data.

Why it matters:
- Core MVCC visibility metadata lives here in modern versions.

Version history:
- Renamed from `pg_clog/` in PostgreSQL 10.

#### `pg_xlog/` (PostgreSQL 9.6 or earlier)
Legacy WAL directory name used in older versions.

Version history:
- Renamed to `pg_wal/` in PostgreSQL 10.

#### Version and naming compatibility notes
- PostgreSQL 10+ uses `pg_wal/`; PostgreSQL 9.6 and earlier use `pg_xlog/`.
- PostgreSQL 10+ uses `pg_xact/`; PostgreSQL 9.6 and earlier use `pg_clog/`.
- When reading old documentation, always map old names to new names before applying operational steps.
