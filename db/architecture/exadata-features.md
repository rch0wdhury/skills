# Oracle Exadata Features: Smart Scan, HCC, and Storage Offload

## Overview

Oracle Exadata is an engineered system that co-locates Oracle Database servers with intelligent storage cells running Exadata Storage Server Software (CELLSRV). The fundamental design principle is to move SQL processing — filtering, joining, decompression — down to the storage layer, dramatically reducing the amount of data that must traverse the storage network to the database servers.

Exadata is available as a physical on-premises system (X9M, X10M), as Exadata Cloud Service (ExaCS) on OCI, and as Exadata Cloud at Customer (ExaDB-C@C). The Exadata-specific features described in this guide are available across all deployment models.

---

## 1. Smart Scan (Cell Offload Processing)

Smart Scan is the core Exadata capability. When Oracle determines that a query can benefit from cell-side processing, it sends the predicate, projection, and optional join filter information to each storage cell. Each cell evaluates the data on its local disk and returns only the rows that satisfy the predicates. This is called **cell offload**.

### How Smart Scan Works

Without Smart Scan:
1. Storage cell reads all blocks from disk
2. All blocks transfer across InfiniBand to DB server
3. DB server evaluates WHERE clause and discards most rows
4. DB server returns small result set to the session

With Smart Scan:
1. Storage cell reads all blocks from disk
2. Cell evaluates WHERE clause, projection, and bloom filter in CELLSRV process
3. Only matching rows (or row pieces) transfer across InfiniBand
4. DB server performs any remaining aggregation

The data reduction ratio (bytes sent vs. bytes on disk) is called the **cell offload efficiency**. A 90% offload efficiency means 90% of the data never left the storage layer.

### Prerequisites for Smart Scan

Smart Scan activates only when all of the following conditions are met:

1. **Full segment scan**: The operation is a full table scan or full index fast scan (not a lookup by rowid or index range scan on a selective range).
2. **Direct-path read**: The operation bypasses the buffer cache and reads directly from storage. This occurs when the segment is large enough to trigger direct reads, or when the session uses `/*+ PARALLEL */` or `/*+ FULL */`.
3. **Exadata storage**: The segment's datafiles reside on ASM disk groups backed by Exadata storage cells.

```sql
-- Verify Smart Scan eligibility for a table
SELECT table_name, blocks, num_rows,
       ROUND(blocks * 8192 / 1024 / 1024, 1) AS size_mb
FROM   user_tables
WHERE  table_name = 'SALES';

-- A table must be large enough to trigger direct-path reads
-- The threshold is roughly: segment_size > _small_table_threshold * db_block_size
SELECT name, value
FROM   v$parameter
WHERE  name = '_small_table_threshold';

-- Force Smart Scan for testing (do not use in production)
ALTER SESSION SET "_serial_direct_read" = always;
```

### Monitoring Smart Scan Offload

```sql
-- System-level Smart Scan statistics
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell physical IO interconnect bytes',
    'cell physical IO interconnect bytes returned by smart scan',
    'cell scans',
    'cell blocks processed by cache layer',
    'cell blocks processed by data layer',
    'cell blocks processed by txn layer'
)
ORDER  BY name;

-- Calculate offload efficiency (percentage of bytes filtered out)
SELECT ROUND(
    (1 - (bytes_returned / NULLIF(bytes_read, 0))) * 100,
    2
) AS offload_efficiency_pct
FROM (
    SELECT
        SUM(CASE WHEN name = 'cell physical IO interconnect bytes'
                 THEN value ELSE 0 END) AS bytes_returned,
        SUM(CASE WHEN name = 'cell physical IO interconnect bytes returned by smart scan'
                 THEN value ELSE 0 END) AS bytes_scan_returned
    FROM v$sysstat
    WHERE name IN (
        'cell physical IO interconnect bytes',
        'cell physical IO interconnect bytes returned by smart scan'
    )
);

-- Session-level: how much was offloaded in the current session
SELECT name, value
FROM   v$mystat m JOIN v$statname n ON m.statistic# = n.statistic#
WHERE  n.name LIKE 'cell%'
ORDER  BY value DESC;

-- SQL-level: check cell_offload_efficiency from AWR
SELECT sql_id,
       ROUND(AVG(cell_offload_efficiency), 2) AS avg_offload_pct,
       SUM(executions) AS total_execs,
       ROUND(SUM(elapsed_time) / 1e6, 1) AS total_elapsed_sec
FROM   dba_hist_sqlstat
WHERE  cell_offload_efficiency > 0
  AND  snap_id >= (SELECT MAX(snap_id) - 24 FROM dba_hist_snapshot)
GROUP  BY sql_id
ORDER  BY total_elapsed_sec DESC
FETCH  FIRST 20 ROWS ONLY;
```

---

## 2. Storage Indexes

Storage Indexes are a purely in-memory optimization maintained automatically by CELLSRV. For each 1 MB region of disk, the storage index tracks the minimum and maximum value for up to 8 columns. When a query has a WHERE clause predicate, the storage cell checks whether the predicate can possibly match any row in each 1 MB region. If not, the entire region is skipped without any disk I/O.

Storage Indexes are:
- **Automatic**: no DDL or configuration required
- **In-memory only**: stored in storage cell memory (DRAM), not on disk
- **Lost on cell restart**: rebuilt over time as data is accessed

### When Storage Indexes Are Most Effective

Storage Indexes are most effective when:
- Data has **natural clustering** by the predicate column (e.g., `ORDER_DATE` in an orders table inserted in chronological order)
- Queries filter on **high-cardinality range predicates** (`WHERE order_date BETWEEN ... AND ...`)
- The table is **not randomly distributed** across the storage cells

```sql
-- Storage Index statistics (per storage cell, from CellCLI or V$ views)
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell blocks helped by minmax predicate',   -- blocks eliminated by storage index
    'cell blocks helped by bloom filter'        -- blocks eliminated by bloom filter join
)
ORDER  BY name;

-- Identify queries that would benefit from storage index clustering
-- Look for full scans with selective date/numeric predicates
SELECT sql_id, sql_text, executions,
       rows_processed / NULLIF(executions, 0) AS avg_rows_per_exec,
       buffer_gets / NULLIF(executions, 0) AS avg_bufgets_per_exec
FROM   v$sql
WHERE  sql_text LIKE '%ORDER_DATE%'
  AND  operation_type = 'TABLE ACCESS FULL'
  AND  executions > 10
ORDER  BY buffer_gets DESC
FETCH  FIRST 10 ROWS ONLY;
```

---

## 3. Hybrid Columnar Compression (HCC)

HCC is an Exadata-specific compression technology (also available on OCI Object Storage and ZFS Storage Appliance) that groups column values from multiple rows into **compression units** and compresses them together. Because column data tends to have low cardinality (many repeated values), HCC achieves compression ratios typically 10x–50x better than row-based compression.

### HCC Compression Levels

| Compression Level | Target Use Case | Typical Ratio | Query Overhead |
|---|---|---|---|
| `QUERY LOW` | Active query tables | 6x–10x | Low |
| `QUERY HIGH` | Active query tables | 10x–15x | Low-Medium |
| `ARCHIVE LOW` | Infrequently queried archives | 15x–25x | Medium |
| `ARCHIVE HIGH` | Cold archives | 25x–50x | Higher |

`QUERY LOW` and `QUERY HIGH` are suitable for tables queried regularly. `ARCHIVE` levels are for data that is loaded once and rarely queried.

### Applying HCC

```sql
-- Create a table with HCC compression
CREATE TABLE sales_archive (
    sale_id       NUMBER        NOT NULL,
    sale_date     DATE          NOT NULL,
    product_id    NUMBER        NOT NULL,
    customer_id   NUMBER        NOT NULL,
    region_id     NUMBER(2)     NOT NULL,
    amount        NUMBER(12,2)  NOT NULL
)
COMPRESS FOR QUERY HIGH;

-- Alter an existing table to use HCC
-- Note: this only affects NEW data loaded after the ALTER
ALTER TABLE sales_history COMPRESS FOR QUERY HIGH;

-- To recompress existing rows, move the table
ALTER TABLE sales_history MOVE COMPRESS FOR QUERY HIGH ONLINE;

-- Or use CTAS (Create Table As Select) to recompress fully
CREATE TABLE sales_history_new
COMPRESS FOR QUERY HIGH
AS SELECT * FROM sales_history;

-- Check compression on individual table partitions
SELECT table_name, partition_name, compression, compress_for,
       blocks, num_rows
FROM   user_tab_partitions
WHERE  table_name = 'SALES_HISTORY'
ORDER  BY partition_position;
```

### HCC and DML

HCC imposes important restrictions on DML operations:

- **INSERT ... AS SELECT (direct path)**: Fully supported; creates HCC compression units
- **INSERT (conventional)**: Rows are stored in a small row-format area (OLTP compression), not HCC
- **UPDATE**: Updated rows are migrated out of the HCC compression unit into OLTP format (row migration)
- **DELETE**: Deleted rows leave "holes" in compression units; does not cause migration but wastes space

For this reason, HCC is appropriate for append-only or archive tables. Mixed read/write workloads should use Advanced Row Compression instead.

```sql
-- Verify HCC compression units for a table (requires SYSDBA or specific privileges)
SELECT table_name, compression, compress_for,
       ROUND(blocks * 8192 / 1024 / 1024, 1) AS size_mb,
       num_rows
FROM   dba_tables
WHERE  table_name = 'SALES_ARCHIVE';

-- Check for row migration caused by updates in HCC tables
SELECT table_name, chain_cnt, num_rows,
       ROUND(chain_cnt / NULLIF(num_rows, 0) * 100, 2) AS migration_pct
FROM   dba_tables
WHERE  compress_for IN ('QUERY LOW', 'QUERY HIGH', 'ARCHIVE LOW', 'ARCHIVE HIGH')
  AND  chain_cnt > 0;
```

---

## 4. I/O Resource Manager (IORM)

IORM runs inside CELLSRV on every storage server and arbitrates I/O among databases, PDBs, and consumer groups. It is the storage-side half of Resource Manager; the database-side half (consumer groups, plans, directives, mapping rules) is covered in `db/features/resource-manager.md`. On Exadata the PDB I/O limits `MAX_IOPS`/`MAX_MBPS` cannot be set (`ORA-56742`) — IORM replaces them.

### How the Plans Fit Together

| Plan | Configured where | Governs I/O among |
|------|------------------|-------------------|
| Database resource plan (*intradatabase*) | In the database: `DBMS_RESOURCE_MANAGER` + `RESOURCE_MANAGER_PLAN` | **Consumer groups** of one database. The active plan is sent to the cells and IORM uses its `MGMT_P1`…`MGMT_P8` shares. No plan enabled → the whole database is treated as one workload. |
| CDB plan | In the CDB: `CREATE_CDB_PLAN`, `CREATE_CDB_PLAN_DIRECTIVE` | **PDBs**: `SHARES` and `UTILIZATION_LIMIT` become the PDBs' IORM shares and flash utilization limits (see `db/architecture/multitenant.md`). |
| `dbplan` (*interdatabase*) | On every cell: `ALTER IORMPLAN dbplan=…` | **Databases** (by `DB_UNIQUE_NAME`) and performance profiles. |
| `clusterplan` | On every cell: `ALTER IORMPLAN clusterplan=…` | **ASM clusters** sharing the cells. Exadata System Software 21.2.0+. |
| `catplan` | — | Deprecated in Exadata System Software 21.2.0, **removed in 25.1.0**. Do not create new ones. |

### IORM Objective

| `objective` | Behavior |
|-------------|----------|
| `auto` | IORM picks latency vs throughput from the observed workload. Recommended; default for new deployments since Exadata System Software 21.2.0. |
| `low_latency` | Favor small-I/O latency (OLTP); throttles large I/O hardest. |
| `balanced` | Middle ground for mixed workloads. |
| `high_throughput` | Favor scan throughput (DW); OLTP latency may suffer. |
| `basic` | Only caps small-I/O latency; disables prioritization. Default in 20.1.0 and earlier. User-defined `dbplan`/`clusterplan` are fully effective only when the objective is **not** `basic`. |

### Database-Side Prerequisites

1. A resource plan is enabled (`RESOURCE_MANAGER_PLAN`) with `MGMT_P[1-8]` directives for every consumer group, and the **same plan on all RAC instances**.
2. For PDBs, a CDB plan with `SHARES`/`UTILIZATION_LIMIT`. **26ai:** `IORM_LIMIT_POLICY` (CDB-level, `ALTER SYSTEM`) decides how PDB shares/limits are derived:
   - `RM_PLAN` (default) — from the user CDB plan; if none is set (`DEFAULT_CDB_PLAN`), equal shares (or shares from `CPU_MIN_COUNT`/`CPU_COUNT`) and no limits.
   - `CPU_COUNT_RATIO` — shares as above; each PDB's utilization limit = PDB `CPU_COUNT` ÷ CDB `CPU_COUNT`.
   IORM utilization limits apply to **flash** IOPS/throughput only, not hard disk or XRMEM, and do not cap flash cache space.

```sql
-- A session mapped to a group that is not in the active plan is silently moved to OTHER_GROUPS
-- (and therefore gets OTHER_GROUPS' I/O share on the cells). Verify mapping before blaming IORM:
SELECT s.username, s.service_name, cg.name AS mapped_group, og.name AS original_group
FROM   v$rsrc_session_info ri
JOIN   v$session s ON s.sid = ri.sid
JOIN   v$rsrc_consumer_group cg ON cg.id = ri.current_consumer_group_id
JOIN   v$rsrc_consumer_group og ON og.id = ri.orig_consumer_group_id
WHERE  ri.current_consumer_group_id <> ri.orig_consumer_group_id;
```

### Cell-Side Configuration (CellCLI)

IORM is configured **individually on every storage server**; run the same commands on all cells (`dcli -g cell_group -l celladmin cellcli -e "..."`).

```text
CellCLI> ALTER IORMPLAN objective=auto

CellCLI> ALTER IORMPLAN dbplan=((name=prod,    share=8, flashcachemin=400G),           -
                                (name=rpt,     share=4, limit=50),                     -
                                (name=dev,     share=1, limit=25, flashcache=off, flashlog=off), -
                                (name=stby_db, share=2, role=standby),                 -
                                (name=DEFAULT, share=1))

CellCLI> LIST IORMPLAN DETAIL
CellCLI> ALTER IORMPLAN dbplan=""          -- reset
```

`dbplan` directive attributes: `name` (`DB_UNIQUE_NAME`, a profile name, or `DEFAULT` for every database not listed); `share` (relative weight) **or** `level`+`allocation` (legacy percentage method — do not mix the two in one plan); `limit` (maximum flash I/O utilization, %); `flashcache`/`flashlog` (`on`|`off`); `flashcachemin` (guaranteed flash cache space), `flashcachelimit` (soft cap), `flashcachesize` (hard cap, 19.2.0+); `xrmemcache`/`xrmemlog` (23.1.0+; `pmemcache`/`pmemlog` before); `role` (`primary`|`standby`, Data Guard role-based directives); `asmcluster` (disambiguate same-named databases, 19.1.0+); `type=profile` (directive applies to every database whose `DB_PERFORMANCE_PROFILE` initialization parameter matches `name`).

### Monitoring IORM

```text
CellCLI> LIST METRICCURRENT DB_IO_UTIL_LG, DB_IO_UTIL_SM ATTRIBUTES name, metricObjectName, metricValue
CellCLI> LIST METRICCURRENT DB_IO_WT_SM_RQ, DB_IO_WT_LG_RQ ATTRIBUTES name, metricObjectName, metricValue
CellCLI> LIST METRICCURRENT WHERE objectType = 'IORM_CONSUMER_GROUP' AND metricValue != 0 -
         ATTRIBUTES name, metricObjectName, metricValue, collectionTime
```

Metric families use the prefixes `DB_` (database), `PDB_` (pluggable database), `CG_` (consumer group), with `_FD_` for flash disks and `_FC_` for flash cache. The two to watch: `*_IO_UTIL_SM`/`*_IO_UTIL_LG` (share of the storage server consumed by small/large I/O) and `*_IO_WT_SM_RQ`/`*_IO_WT_LG_RQ` (average time IORM queued each request). Non-zero `*_WT_*` means IORM is actively throttling that object; high `cell single block physical read` latency in the database with low cell-side waits points at hardware, not IORM. On the database side, `V$RSRC_CONSUMER_GROUP` and `V$RSRCPDBMETRIC` report the same workloads from the instance's view.

---

## 5. Exadata-Specific SQL Hints

Several SQL hints interact directly with Exadata offload capabilities.

### Key Hints

| Hint | Effect |
|---|---|
| `/*+ FULL(table) */` | Forces a full table scan, enabling Smart Scan |
| `/*+ PARALLEL(table, degree) */` | Enables parallel query, which forces direct-path reads and Smart Scan |
| `/*+ NO_PARALLEL(table) */` | Disables parallel query for the table |
| `/*+ CELL_FLASH_CACHE(KEEP) */` | Pins the object in Exadata Smart Flash Cache |
| `/*+ CELL_FLASH_CACHE(NONE) */` | Prevents the object from consuming Smart Flash Cache |
| `/*+ RESULT_CACHE */` | Caches results in the SQL result cache (reduces repeated Smart Scan overhead) |
| `/*+ CACHE(table) */` | Caches blocks in the database buffer cache (disables direct-path) |
| `/*+ NO_CACHE(table) */` | Prevents buffer cache caching (maintains direct-path and Smart Scan) |
| `/*+ VECTOR_TRANSFORM */` | Enables vector transformation for In-Memory aggregation (complements Smart Scan) |

```sql
-- Force Smart Scan with a parallel hint
SELECT /*+ FULL(s) PARALLEL(s, 8) */
       region_id,
       COUNT(*),
       SUM(amount)
FROM   sales s
WHERE  sale_date >= DATE '2025-01-01'
  AND  sale_date <  DATE '2026-01-01'
GROUP  BY region_id;

-- Pin a hot lookup table in Smart Flash Cache to prevent it from
-- being evicted by large Smart Scan operations
ALTER TABLE country_codes STORAGE (CELL_FLASH_CACHE KEEP);

-- Prevent a very large cold table from polluting the Smart Flash Cache
ALTER TABLE sales_archive_2010 STORAGE (CELL_FLASH_CACHE NONE);

-- Check Smart Flash Cache efficiency
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell flash cache read hits',
    'physical read requests optimized'
)
ORDER  BY name;
```

---

## 6. Monitoring Exadata Offload Efficiency

### Key V$ Views for Exadata Monitoring

```sql
-- Overall cell offload efficiency by SQL statement (current)
SELECT sql_id,
       child_number,
       io_cell_offload_eligible_bytes / 1024 / 1024 AS eligible_mb,
       io_cell_offload_returned_bytes  / 1024 / 1024 AS returned_mb,
       ROUND(
           (1 - io_cell_offload_returned_bytes /
                NULLIF(io_cell_offload_eligible_bytes, 0)) * 100,
           2
       ) AS offload_efficiency_pct,
       io_cell_uncompressed_bytes / 1024 / 1024 AS uncompressed_mb
FROM   v$sql
WHERE  io_cell_offload_eligible_bytes > 0
ORDER  BY io_cell_offload_eligible_bytes DESC
FETCH  FIRST 20 ROWS ONLY;

-- Historical offload efficiency from AWR
SELECT snap_id,
       sql_id,
       ROUND(AVG(cell_offload_efficiency), 2) AS avg_offload_pct,
       SUM(io_cell_offload_eligible_bytes) / 1024 / 1024 / 1024 AS eligible_gb,
       SUM(io_cell_offload_returned_bytes) / 1024 / 1024 / 1024 AS returned_gb
FROM   dba_hist_sqlstat
WHERE  cell_offload_efficiency > 0
  AND  snap_id >= (SELECT MAX(snap_id) - 24 FROM dba_hist_snapshot)
GROUP  BY snap_id, sql_id
ORDER  BY eligible_gb DESC
FETCH  FIRST 20 ROWS ONLY;

-- Cell metric statistics (Exadata storage cell performance)
SELECT metric_name,
       AVG(average_value) AS avg_value,
       MAX(maximum_value) AS max_value,
       MIN(minimum_value) AS min_value
FROM   v$cell_metric_desc cmd
JOIN   v$cell_metric      cm ON cmd.metric_id = cm.metric_id
GROUP  BY metric_name
ORDER  BY metric_name;

-- Check Smart Flash Cache hit rate
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell flash cache read hits',
    'physical read total IO requests',
    'physical read requests optimized'
)
ORDER  BY name;

-- Compute Smart Flash Cache hit %
SELECT ROUND(
    SUM(CASE WHEN name = 'cell flash cache read hits' THEN value END) /
    NULLIF(SUM(CASE WHEN name = 'physical read total IO requests' THEN value END), 0) * 100,
    2
) AS flash_cache_hit_pct
FROM   v$sysstat
WHERE  name IN ('cell flash cache read hits', 'physical read total IO requests');
```

### EXPLAIN PLAN and Exadata Offload Indication

```sql
-- Check whether the optimizer plans to use cell offload
EXPLAIN PLAN FOR
SELECT region_id, COUNT(*), SUM(amount)
FROM   sales
WHERE  sale_date >= DATE '2025-01-01'
GROUP  BY region_id;

-- Look for "TABLE ACCESS FULL" with "Batched Disk Reads" -- indicates direct path
-- The key phrase in the plan is "Batched Disk Reads" or "storage" predicate offload
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => '+PREDICATE'));

-- Use cell_offload_plan_display to show offload predicates in execution plan
ALTER SESSION SET "_cell_offload_plan_display" = ALWAYS;

EXPLAIN PLAN FOR
SELECT region_id, SUM(amount)
FROM   sales
WHERE  sale_date >= DATE '2025-01-01'
  AND  amount > 1000
GROUP  BY region_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Plan will show "Storage Filter" and "Storage Project" annotations
-- indicating which predicates and columns are offloaded to the cell
```

---

## 7. Cell Offload Metrics Reference

| Statistic Name | Description |
|---|---|
| `cell physical IO interconnect bytes` | Total bytes sent from storage cells to DB servers |
| `cell physical IO interconnect bytes returned by smart scan` | Bytes returned specifically by Smart Scan operations |
| `cell scans` | Number of Smart Scan operations initiated |
| `cell blocks processed by cache layer` | Blocks processed by the cell caching layer |
| `cell blocks helped by minmax predicate` | Blocks eliminated by Storage Indexes |
| `cell blocks helped by bloom filter` | Blocks eliminated by Bloom filter join offload |
| `cell flash cache read hits` | Reads served from Smart Flash Cache without HDD access |
| `physical read requests optimized` | Physical reads served by Smart Flash Cache |
| `IO megabytes read total` | Total MB read from Exadata storage cells |
| `IO megabytes written total` | Total MB written to Exadata storage cells |

---

## 8. Best Practices

- **Load data into HCC tables using direct-path INSERT.** Conventional insert defeats HCC by writing rows in row format. Always use `INSERT /*+ APPEND */` or `INSERT ... AS SELECT` for large loads.
- **Size the private InfiniBand network appropriately.** The Exadata InfiniBand (or RoCE in X9M) network carries both Smart Scan results and Cache Fusion traffic (in RAC). Monitor for saturation with `V$CELL_METRIC`.
- **Use IORM to protect OLTP latency from large analytics queries.** Without IORM, a parallel Smart Scan consuming all cell I/O bandwidth causes OLTP queries to queue behind it.
- **Keep statistics current on Exadata.** The CBO must know that a full table scan is appropriate to generate a plan that triggers Smart Scan. Stale statistics can cause the optimizer to choose index lookups that bypass Smart Scan.
- **Do not force index usage on large Exadata tables.** Index range scans use small I/Os and bypass Smart Scan. On Exadata, full scans with Smart Scan often outperform selective index scans for range predicates touching more than 5–10% of rows.
- **Use `QUERY HIGH` for reporting tables loaded daily and `ARCHIVE HIGH` only for data that will never be updated.** Mixing ARCHIVE compression with UPDATE-heavy workflows causes severe row migration and performance degradation.

---

## 9. Common Mistakes and How to Avoid Them

### Mistake 1: Expecting Smart Scan on Small Tables

Smart Scan only activates for large segments using direct-path reads. If a table fits in the buffer cache, Oracle will use cached reads and Smart Scan will not activate. The threshold is governed by `_small_table_threshold`.

```sql
-- Check threshold (in database blocks)
SELECT name, value FROM v$parameter WHERE name = '_small_table_threshold';
-- Multiply by block_size to get byte threshold
-- Tables below this size will use buffer cache reads, not Smart Scan
```

### Mistake 2: Using HCC on OLTP Tables

HCC is fundamentally incompatible with frequent DML. Each UPDATE on an HCC row causes the row to migrate out of the compression unit into OLTP format. Over time, the table becomes a mixture of compressed and uncompressed rows, wasting space and slowing reads.

**Fix:** Use `COMPRESS FOR OLTP` (Advanced Row Compression) for OLTP tables. Reserve HCC for append-only or archive tables.

### Mistake 3: Disabling Cell Offload System-Wide

Setting `cell_offload_processing = FALSE` at the system level disables Smart Scan globally. This is sometimes done as a workaround for a specific query bug, but the parameter is then forgotten, leaving the Exadata running as a normal database server.

```sql
-- Check if cell offload is enabled
SELECT name, value FROM v$parameter WHERE name = 'cell_offload_processing';
-- Value should be TRUE for Exadata

-- If set to FALSE for a session-level workaround, never persist it system-wide
-- ALTER SESSION SET cell_offload_processing = FALSE;  -- session only, acceptable
-- ALTER SYSTEM  SET cell_offload_processing = FALSE;  -- NEVER do this
```

### Mistake 4: Ignoring Storage Index Invalidation After Bulk Loads

Storage Indexes are rebuilt in memory as data is accessed. After a large bulk load or partition exchange, the Storage Index for the affected regions is cold. The first few executions of queries against newly loaded data will not benefit from Storage Index skipping. This is expected behavior, not a performance regression.

### Mistake 5: Running Exadata Diagnostics Only at the Database Layer

Exadata performance problems often originate in the storage cells (CELLSRV process, Flash Cache, disk I/O). Cell-level diagnostics require `CellCLI` access or Exadata Metrics available through OCI console. Always check cell-level metrics alongside `V$SYSSTAT` before concluding a performance issue is query-related.

---


## Oracle Version Notes (19c vs 26ai)

- Baseline guidance in this file is valid for Oracle Database 19c unless a newer minimum version is explicitly called out.
- Features marked as 21c, 23c, or 23ai should be treated as Oracle Database 26ai-capable features; keep 19c-compatible alternatives for mixed-version estates.
- For dual-support environments, test syntax and package behavior in both 19c and 26ai because defaults and deprecations can differ by release update.
- **IORM (26ai):** `IORM_LIMIT_POLICY` (`RM_PLAN` | `CPU_COUNT_RATIO`) is new in Oracle AI Database 26ai and controls how PDB IORM shares and flash utilization limits are derived. IORM behavior also depends on the Exadata System Software release, not only the database release: `objective=auto` is the default from 21.2.0, `clusterplan` exists from 21.2.0, and `catplan` is removed in 25.1.0.

## Sources

- [Oracle Exadata Database Machine System Overview](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/) — Smart Scan, Storage Indexes, HCC, IORM
- [Oracle Database SQL Language Reference 19c — COMPRESS clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html) — HCC compression levels
- [DBMS_RESOURCE_MANAGER (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER.html) — database resource plans and CDB plan directives used by IORM
- [Oracle Exadata System Software User's Guide — Managing I/O Resources](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/exadata-storage-server-iorm.html) — IORM concepts, plans, objective
- [Oracle Exadata System Software User's Guide — ALTER IORMPLAN](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/alter-iormplan.html) — `dbplan`/`clusterplan` directive syntax and attributes
- [Oracle Exadata System Software User's Guide — Verifying the Configuration of I/O Resource Management](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/verifying-configuration-i-o-resource-management.html) — database-side prerequisites, metrics
- [Oracle AI Database Reference 26ai — IORM_LIMIT_POLICY](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/IORM_LIMIT_POLICY.html)
- [Database Error Messages — ORA-56742](https://docs.oracle.com/en/error-help/db/oci-56742) — `MAX_IOPS`/`MAX_MBPS` not usable on Exadata
