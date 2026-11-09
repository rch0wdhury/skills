# Oracle Database Resource Manager (DBRM)

## Overview

Database Resource Manager lets the database decide how CPU, parallel execution servers, PGA, undo, session slots, and (per PDB) disk I/O are shared among competing workloads. Sessions are grouped into **consumer groups**; a **resource plan** of **plan directives** says what each group gets and what happens when a session misbehaves.

Use this skill to: protect OLTP from reporting/ad hoc load on the same instance; cancel or kill runaway SQL after a CPU, I/O, or elapsed-time threshold; cap parallelism, per-session PGA, or concurrent sessions for a class of users; rate-limit a PDB's I/O; cage an instance to a fixed CPU count; or diagnose `resmgr:*` waits.

Scope: everything managed with `DBMS_RESOURCE_MANAGER`, `DBMS_RESOURCE_MANAGER_PRIVS`, `RESOURCE_MANAGER_PLAN`, and the PDB I/O limits `MAX_IOPS`/`MAX_MBPS`. CDB plans that divide resources *between* PDBs are in `db/architecture/multitenant.md`; Exadata storage-side IORM is in `db/architecture/exadata-features.md`.

> **Naming in this file.** Every object prefixed `APP_` is created by the examples. Unprefixed names (`SYS_GROUP`, `OTHER_GROUPS`, `LOW_GROUP`, `DEFAULT_PLAN`, `DEFAULT_MAINTENANCE_PLAN`, `ORA$AUTOTASK`, …) are Oracle-supplied and already exist in every database.

---

## Core Concepts

| Element | What it is |
|---------|------------|
| Consumer group | Named set of sessions with similar needs. A session is in exactly one group at a time. |
| Resource plan | Container of directives. One plan active per instance (per PDB in a CDB). May nest **subplans**. |
| Plan directive | One row of a plan: allocation + limits + runaway actions for one group or subplan. Every plan must have a directive for `OTHER_GROUPS`. |
| Mapping rule | Puts a session into a group at login or at run time based on session attributes. |
| Pending area | Staging area. All DBRM DDL is done inside one, validated, then submitted atomically. |

### Oracle-Supplied Objects

| Name | Role |
|------|------|
| `SYS_GROUP` | Initial group for `SYS` and `SYSTEM`. |
| `DEFAULT_CONSUMER_GROUP` | Initial group for every user not mapped elsewhere. |
| `OTHER_GROUPS` | Mandatory in every plan; receives sessions whose group has no directive in the active plan. |
| `LOW_GROUP` | Low-priority group that `PUBLIC` may switch to. |
| `ORA$AUTOTASK` | Subplan/groups for automated maintenance tasks. Keep it in any plan used during maintenance windows. |
| `DEFAULT_PLAN` | Minimal plan: `SYS_GROUP` first, small allocation for maintenance/diagnostics. Enough to turn on instance caging. |
| `DEFAULT_MAINTENANCE_PLAN` | Activated automatically by the predefined maintenance windows. |
| `MIXED_WORKLOAD_PLAN`, `DSS_PLAN`, `ETL_CRITICAL_PLAN` | Shipped examples (use groups `INTERACTIVE_GROUP`, `BATCH_GROUP`, `DSS_GROUP`, `DSS_CRITICAL_GROUP`, `ETL_GROUP`). Do not reuse these group names for your own objects. |
| `INTERNAL_PLAN`, `INTERNAL_QUIESCE` | Internal; never activate. |

```sql
SELECT plan, status, comments FROM dba_rsrc_plans ORDER BY plan;
SELECT consumer_group, mandatory, comments FROM dba_rsrc_consumer_groups ORDER BY consumer_group;
```

---

## Privileges

`ADMINISTER_RESOURCE_MANAGER` is required for all `DBMS_RESOURCE_MANAGER` calls. It is granted `WITH ADMIN OPTION` to the `DBA` role and can be granted or revoked **only** through `DBMS_RESOURCE_MANAGER_PRIVS` — SQL `GRANT`/`REVOKE` do not work for it.

```sql
BEGIN
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SYSTEM_PRIVILEGE(
    grantee_name => 'APP_WLM_ADMIN', privilege_name => 'ADMINISTER_RESOURCE_MANAGER', admin_option => FALSE);
END;
/
SELECT grantee, privilege, admin_option FROM dba_rsrc_manager_system_privs;
```

A session can only be placed in a group that its user (or a role it holds, or `PUBLIC`) has been granted:

```sql
BEGIN
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP(
    grantee_name => 'APP_REPORT_ROLE', consumer_group => 'APP_REPORT_CG', grant_option => FALSE);
END;
/
SELECT grantee, granted_group, grant_option, initial_group FROM dba_rsrc_consumer_group_privs;
```

---

## Build a Plan: Pending Area → Groups → Plan → Directives → Mappings → Submit → Activate

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();

  -- Consumer groups
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('APP_OLTP_CG',   'Customer-facing OLTP');
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('APP_REPORT_CG', 'BI and scheduled reports');
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('APP_BATCH_CG',  'ETL, RMAN, Data Pump');
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('APP_ADHOC_CG',  'Analyst ad hoc SQL');

  -- Plan
  DBMS_RESOURCE_MANAGER.CREATE_PLAN('APP_DAY_PLAN', 'Business hours: protect OLTP, bound everything else');

  -- Level 1: administrators. Level 1 takes only what it needs; the rest flows down.
  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'SYS_GROUP', comment => 'Admin first',
    mgmt_p1 => 100);

  -- Level 2: application groups (must total <= 100 within the level)
  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'APP_OLTP_CG', comment => 'OLTP',
    mgmt_p2                  => 70,
    parallel_degree_limit_p1 => 1,            -- serial only
    switch_group             => 'CANCEL_SQL',  -- ORA-00040 to the caller
    switch_time              => 30,            -- CPU seconds
    switch_for_call          => TRUE);

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'APP_REPORT_CG', comment => 'Reports',
    mgmt_p2                  => 15,
    utilization_limit        => 50,            -- hard CPU cap even when CPU is idle
    parallel_degree_limit_p1 => 4,
    parallel_server_limit    => 40,            -- % of PX server pool
    parallel_queue_timeout   => 600,
    pq_timeout_action        => 'RUN',         -- run (possibly downgraded) instead of ORA-07454
    session_pga_limit        => 2048);         -- MB; ORA-10260 when exceeded

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'APP_BATCH_CG', comment => 'Batch',
    mgmt_p2                  => 10,
    parallel_degree_limit_p1 => 8,
    undo_pool                => 2097152);      -- KB of undo for the whole group; ORA-30027 when exceeded

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'APP_ADHOC_CG', comment => 'Ad hoc',
    mgmt_p2                  => 5,
    utilization_limit        => 20,
    active_sess_pool_p1      => 10,            -- max concurrently active sessions
    queueing_p1              => 120,           -- seconds to wait for a slot, then ORA-07454
    switch_group             => 'KILL_SESSION',-- ORA-00041 to the caller
    switch_elapsed_time      => 1800,          -- wall-clock seconds; works without SQL Monitoring
    switch_for_call          => TRUE,
    max_idle_time            => 900);          -- seconds idle before termination

  -- Level 3: mandatory catch-all
  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'APP_DAY_PLAN', group_or_subplan => 'OTHER_GROUPS', comment => 'Everything else',
    mgmt_p3 => 100);

  -- Mapping rules
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(DBMS_RESOURCE_MANAGER.SERVICE_NAME,    'APP_OLTP_SVC', 'APP_OLTP_CG');
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(DBMS_RESOURCE_MANAGER.SERVICE_NAME,    'APP_RPT_SVC',  'APP_REPORT_CG');
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(DBMS_RESOURCE_MANAGER.ORACLE_FUNCTION, 'BACKUP',       'APP_BATCH_CG');
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(DBMS_RESOURCE_MANAGER.ORACLE_FUNCTION, 'DATALOAD',     'APP_BATCH_CG');
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(DBMS_RESOURCE_MANAGER.CLIENT_PROGRAM,  'sqldeveloper%','APP_ADHOC_CG');

  DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA();
  DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/

-- Users/roles must be allowed into the groups the rules point at
BEGIN
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP('APP_OLTP_ROLE',   'APP_OLTP_CG',   FALSE);
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP('APP_REPORT_ROLE', 'APP_REPORT_CG', FALSE);
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP('APP_BATCH_ROLE',  'APP_BATCH_CG',  FALSE);
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP('APP_ANALYST_ROLE','APP_ADHOC_CG',  FALSE);
END;
/

ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'APP_DAY_PLAN' SCOPE = BOTH;
```

Validation failures surface as `ORA-29382: validation of pending area failed` followed by the reason — most often `ORA-29377: consumer group OTHER_GROUPS is not part of top-plan`. Fix and re-validate, or abandon with `CLEAR_PENDING_AREA`. `ORA-29371` means you forgot `CREATE_PENDING_AREA`; `ORA-29370` means one is already open.

For a single-level, CPU-only plan, `CREATE_SIMPLE_PLAN` does all of the above in one call (it adds `SYS_GROUP` at level 1 and `OTHER_GROUPS` at level 3 for you):

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_SIMPLE_PLAN(simple_plan => 'APP_SIMPLE_PLAN',
    consumer_group1 => 'APP_OLTP_CG', group1_percent => 75,
    consumer_group2 => 'APP_REPORT_CG', group2_percent => 25);
END;
/
```

---

## Mapping Sessions to Groups

| Class | Attributes | Notes |
|-------|------------|-------|
| Login (evaluated at connect) | `ORACLE_USER`, `SERVICE_NAME`, `CLIENT_OS_USER`, `CLIENT_PROGRAM`, `CLIENT_MACHINE`, `CLIENT_ID` | `CLIENT_ID` is set by `DBMS_SESSION.SET_IDENTIFIER`. |
| Run-time (re-evaluated when the attribute changes) | `MODULE_NAME`, `MODULE_NAME_ACTION`, `SERVICE_MODULE`, `SERVICE_MODULE_ACTION`, `ORACLE_FUNCTION` | Module/action come from `DBMS_APPLICATION_INFO` or driver end-to-end metrics. `ORACLE_FUNCTION` is `BACKUP`, `COPY` (RMAN) or `DATALOAD` (Data Pump). |

Wildcards `%` `_` (escape `\`) are allowed only for `CLIENT_OS_USER`, `CLIENT_PROGRAM`, `CLIENT_MACHINE`, `MODULE_NAME`, `MODULE_NAME_ACTION`, `SERVICE_MODULE`, `SERVICE_MODULE_ACTION`. Delete a rule by mapping it to `NULL`.

When several rules match, `DBA_RSRC_MAPPING_PRIORITY` decides. `EXPLICIT` (manual switch) must be 1. Put the attributes you control (service, module) above the broad ones (user, machine):

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING_PRI(
    explicit => 1, service_module_action => 2, service_module => 3, module_name_action => 4,
    module_name => 5, service_name => 6, oracle_user => 7, client_program => 8,
    client_os_user => 9, client_machine => 10, client_id => 11);
  DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/
SELECT attribute, priority FROM dba_rsrc_mapping_priority ORDER BY priority;
SELECT attribute, value, consumer_group FROM dba_rsrc_group_mappings ORDER BY attribute, value;
```

Manual switches (no pending area needed):

```sql
-- Administrator moves one session, or all sessions of a user
EXEC DBMS_RESOURCE_MANAGER.SWITCH_CONSUMER_GROUP_FOR_SESS(session_id => 1234, session_serial => 56789, consumer_group => 'APP_ADHOC_CG');
EXEC DBMS_RESOURCE_MANAGER.SWITCH_CONSUMER_GROUP_FOR_USER(user => 'ANALYST01', consumer_group => 'APP_ADHOC_CG');

-- A session switches itself (needs the switch privilege on the target group)
DECLARE old_cg VARCHAR2(128);
BEGIN
  DBMS_SESSION.SWITCH_CURRENT_CONSUMER_GROUP('APP_BATCH_CG', old_cg, FALSE);
END;
/
```

---

## Directive Reference

| Resource | Directives | Notes |
|----------|------------|-------|
| CPU share | `MGMT_P1`…`MGMT_P8` | 8 priority levels; a level gets CPU only if higher levels leave some; ≤ 100 per level. Idle CPU is redistributed. (`CPU_Pn` deprecated.) |
| CPU cap | `UTILIZATION_LIMIT` | Hard % ceiling even when CPU is idle. Produces `resmgr:cpu quantum` when it bites. (`MAX_UTILIZATION_LIMIT` deprecated.) |
| Parallel | `PARALLEL_DEGREE_LIMIT_P1`, `PARALLEL_SERVER_LIMIT`, `PARALLEL_QUEUE_TIMEOUT`, `PQ_TIMEOUT_ACTION` (`CANCEL`\|`RUN`), `PARALLEL_STMT_CRITICAL` (`BYPASS_QUEUE`) | DOP cap per statement; % of PX pool per group; seconds in the PX queue; timeout action (default `CANCEL` → ORA-07454). (`PARALLEL_TARGET_PERCENTAGE` deprecated.) |
| PGA | `SESSION_PGA_LIMIT` | MB of untunable PGA per session → `ORA-10260: PGA limit (N MB) exceeded - process terminated`. Work areas that can spill to temp are not counted. |
| Runaway | `SWITCH_GROUP` + `SWITCH_TIME` / `SWITCH_IO_MEGABYTES` / `SWITCH_IO_REQS` / `SWITCH_IO_LOGICAL` / `SWITCH_ELAPSED_TIME` / `SWITCH_ESTIMATE` / `SWITCH_FOR_CALL` | Next section. (`SWITCH_TIME_IN_CALL` deprecated → `SWITCH_TIME` + `SWITCH_FOR_CALL => TRUE`.) |
| Pre-check | `MAX_EST_EXEC_TIME` | Reject before start if optimizer estimate (s) exceeds it → ORA-07455. |
| Concurrency | `ACTIVE_SESS_POOL_P1`, `QUEUEING_P1` | Max active sessions in the group; seconds to queue before ORA-07454. |
| Undo | `UNDO_POOL` | KB of undo for the group's active transactions → ORA-30027. |
| Idle | `MAX_IDLE_TIME`, `MAX_IDLE_BLOCKER_TIME` | **Seconds**. (The PDB-level initialization parameters of the same names are in **minutes**.) Blocker variant fires only if the idle session blocks another. |

Change a live directive with `UPDATE_PLAN_DIRECTIVE` (`new_` prefix on each attribute); changes to the active plan apply on `SUBMIT_PENDING_AREA`, no re-`ALTER SYSTEM`.

---

## Runaway Query Control

`SWITCH_GROUP` may be a consumer group (demote), or one of:

| Value | Effect | Error to caller |
|-------|--------|-----------------|
| `CANCEL_SQL` | Cancel the current call; session survives | `ORA-00040: active time limit exceeded - call aborted` |
| `KILL_SESSION` | Terminate the session | `ORA-00041: active time limit exceeded - session terminated` |
| `LOG_ONLY` | Record in Real-Time SQL Monitoring only | none — use this to dry-run thresholds |

Thresholds: `SWITCH_TIME` (CPU s), `SWITCH_IO_MEGABYTES` / `SWITCH_IO_REQS` (physical I/O), `SWITCH_IO_LOGICAL` (buffer gets), `SWITCH_ELAPSED_TIME` (wall-clock s), `SWITCH_ESTIMATE => TRUE` (act on the optimizer estimate before running).

Two rules that catch people out:

1. **`SWITCH_TIME` and `SWITCH_IO_*` need Real-Time SQL Monitoring.** They are not enforced for `NO_MONITOR` statements or when `CONTROL_MANAGEMENT_PACK_ACCESS = 'NONE'`. `SWITCH_ELAPSED_TIME` is enforced regardless — use it where Diagnostics + Tuning Pack are not licensed.
2. **`SWITCH_FOR_CALL`** — `FALSE` (default) accumulates since session start and resets only when idle; `TRUE` accounts per top-level call and restores the original group afterwards. With connection pools, use `TRUE`.

```sql
-- Review every action Resource Manager took on recent statements
SELECT sql_id, username, rm_consumer_group, rm_last_action, rm_last_action_reason, rm_last_action_time
FROM   v$sql_monitor WHERE rm_last_action IS NOT NULL ORDER BY rm_last_action_time DESC;
```

**SQL Quarantine (19c+).** A statement cancelled by Resource Manager can be blocked from running again with `DBMS_SQLQ` (`CREATE_QUARANTINE_BY_SQL_ID`, `ALTER_QUARANTINE`; view `DBA_SQL_QUARANTINE`). Quarantined statements fail at compile time rather than consuming resources before cancellation. Availability is platform-restricted; check the Licensing Information User Manual for your release.

---

## PDB I/O Rate Limits (MAX_IOPS, MAX_MBPS)

Independent of plans and consumer groups: two PDB initialization parameters throttle a PDB's **physical I/O**. No resource plan needs to be active.

```sql
ALTER SESSION SET CONTAINER = app_pdb1;
ALTER SYSTEM SET MAX_IOPS = 3000 SCOPE = BOTH;   -- I/O requests per second
ALTER SYSTEM SET MAX_MBPS = 200  SCOPE = BOTH;   -- megabytes per second
-- 0 (default) = no limit. Set in CDB$ROOT to give every PDB a default; a PDB value overrides it.
```

- CDB only. `ORA-56739` in a non-CDB; `ORA-56742` on Exadata — there IORM manages I/O instead (see `db/architecture/exadata-features.md`).
- DBWR, control-file, password-file and other critical I/O are **exempt but counted**, so the observed rate can briefly exceed the limit.
- Throttled sessions wait on `resmgr: I/O rate limit`. Values under ~100 IOPS or ~25 MBPS are not recommended.
- RAC: the values may differ per instance.

```sql
-- Size a limit from history: what the PDB actually does, and how much of it was exempt
SELECT p.pdb_name, m.begin_time, m.iops, m.iops_throttle_exempt, m.iombps, m.iombps_throttle_exempt, m.avg_io_throttle
FROM   dba_hist_rsrc_pdb_metric m JOIN cdb_pdbs p ON p.con_id = m.con_id
WHERE  p.pdb_name = 'APP_PDB1' ORDER BY m.begin_time DESC FETCH FIRST 60 ROWS ONLY;
```

---

## Enabling, Pinning, and Disabling Plans

```sql
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'APP_DAY_PLAN'       SCOPE = BOTH;  -- enable / switch (error if plan missing)
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'FORCE:APP_DAY_PLAN' SCOPE = BOTH;  -- pin: Scheduler windows cannot change it
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = ''                   SCOPE = BOTH;  -- disable at instance level

-- PL/SQL equivalent; sid defaults to '*' (all RAC instances)
EXEC DBMS_RESOURCE_MANAGER.SWITCH_PLAN(plan_name => 'APP_NIGHT_PLAN', allow_scheduler_plan_switches => FALSE);

SELECT name, is_top_plan, cpu_managed, instance_caging, parallel_execution_managed FROM v$rsrc_plan;
```

Resource Manager is off by default **except**: during the predefined maintenance windows (which activate `DEFAULT_MAINTENANCE_PLAN`), and whenever `INMEMORY_SIZE > 0`. A Scheduler window with a `resource_plan` attribute switches the plan when it opens and restores the previous one when it closes. To use your own plan during maintenance, either set each window's `resource_plan` to it (and include the `ORA$AUTOTASK` subplan) or pin yours with `FORCE:`. To disable completely, clear `RESOURCE_MANAGER_PLAN` **and** each window's `resource_plan` (`DBA_SCHEDULER_WINDOWS`, `DBMS_SCHEDULER.SET_ATTRIBUTE(..., 'RESOURCE_PLAN', '')`). See `db/features/dbms-scheduler.md`.

RAC: `RESOURCE_MANAGER_PLAN` is instance-specific; run the same plan on every instance (Exadata IORM requires it).

---

## Instance Caging and Multi-Instance Servers

```sql
ALTER SYSTEM SET CPU_COUNT = 8 SCOPE = BOTH;                              -- 1. CPU budget for this instance
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'DEFAULT_PLAN' SCOPE = BOTH;    -- 2. any CPU-managing plan
SELECT instance_caging FROM v$rsrc_plan WHERE is_top_plan = 'TRUE';      -- ON
```

Size either **partitioned** (Σ `CPU_COUNT` ≤ physical CPUs: strict isolation) or **over-provisioned** (Σ > physical: better utilization, weaker guarantee).

**26ai — `RESOURCE_MANAGER_CPU_SCOPE`.** `INSTANCE_ONLY` (default) is the behavior above. `SERVER_WIDE` makes Oracle configure Linux cgroups with CPU shares/limits across *all* instances on the host (`PROCESSOR_GROUP_NAME` is then ignored). Static parameter, same value on all RAC instances, Linux with UEK 5+, supported on Exadata (on-premises, Cloud Service, Cloud@Customer) and Autonomous AI Database (default there); not on other Linux.

---

## Monitoring and Diagnosis

| View | Use |
|------|-----|
| `V$RSRC_PLAN` | Active plan; whether CPU, PX, caging are managed. |
| `V$RSRC_CONSUMER_GROUP` | Per-group cumulative since activation: `CONSUMED_CPU_TIME`, `CPU_WAITS`, `CPU_WAIT_TIME`, `QUEUE_LENGTH`, `SQL_CANCELED`, `ACTIVE_SESS_KILLED`. |
| `V$RSRC_SESSION_INFO` | Per session: current vs original group, `MAPPING_ATTRIBUTE`, `STATE` (`RUNNING`, `WAIT_FOR_CPU`, `QUEUED`, …), `CPU_WAIT_TIME`, `QUEUED_TIME`. |
| `V$RSRCMGRMETRIC[_HISTORY]` | Per-group CPU utilization vs limit, waits, PX queueing — last minute / last 60 minutes. |
| `V$RSRC_PLAN_HISTORY`, `V$RSRC_CONS_GROUP_HISTORY` | Plan switches (incl. `WINDOW_NAME`) and per-activation group stats; join on `SEQUENCE#`. |
| `V$RSRCPDBMETRIC[_HISTORY]`, `V$RSRC_PDB` | Same at PDB level, plus `IOPS`, `IOMBPS`, `*_THROTTLE_EXEMPT`, `AVG_IO_THROTTLE`, SGA/PGA bytes. |
| `DBA_HIST_RSRC_PLAN`, `DBA_HIST_RSRC_CONSUMER_GROUP`, `DBA_HIST_RSRC_PDB_METRIC` | AWR history. |
| `DBA_RSRC_PLANS`, `DBA_RSRC_PLAN_DIRECTIVES`, `DBA_RSRC_CONSUMER_GROUPS`, `DBA_RSRC_GROUP_MAPPINGS`, `DBA_RSRC_MAPPING_PRIORITY`, `DBA_RSRC_CONSUMER_GROUP_PRIVS`, `DBA_RSRC_MANAGER_SYSTEM_PRIVS` | Definitions. |

Wait events: `resmgr:cpu quantum` (held off CPU by share or limit), `resmgr:pq queued` (parallel statement queue), `resmgr:become active` (active-session-pool slot), `resmgr:internal state change` (transient, plan/group switch), `resmgr: I/O rate limit` (`MAX_IOPS`/`MAX_MBPS`).

```sql
-- Who is throttled now, and which rule put them in that group
SELECT s.sid, s.username, s.service_name, s.module, ri.state, cg.name AS consumer_group, ri.mapping_attribute,
       ri.cpu_wait_time/1000 AS cpu_wait_s, ri.queued_time/1000 AS queued_s
FROM   v$rsrc_session_info ri JOIN v$session s ON s.sid = ri.sid
JOIN   v$rsrc_consumer_group cg ON cg.id = ri.current_consumer_group_id
WHERE  ri.state IN ('WAIT_FOR_CPU','QUEUED') ORDER BY ri.cpu_wait_time DESC;

-- Are limits binding? (last hour, per group)
SELECT begin_time, consumer_group_name, ROUND(avg_cpu_utilization,1) cpu_pct, cpu_utilization_limit,
       ROUND(cpu_wait_time/1000,1) cpu_wait_s, avg_running_sessions, avg_waiting_sessions, avg_queued_parallel_stmts
FROM   v$rsrcmgrmetric_history WHERE consumer_group_name NOT LIKE 'ORA$%'
ORDER  BY begin_time DESC, consumer_group_name;

-- Why is a user in the wrong group?
SELECT s.username, s.service_name, s.program, s.module, cg.name current_group, og.name original_group, ri.mapping_attribute
FROM   v$rsrc_session_info ri JOIN v$session s ON s.sid = ri.sid
LEFT JOIN v$rsrc_consumer_group cg ON cg.id = ri.current_consumer_group_id
LEFT JOIN v$rsrc_consumer_group og ON og.id = ri.orig_consumer_group_id
WHERE  s.username = 'ANALYST01';
```

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| OLTP waits on `resmgr:cpu quantum` while CPU is idle | `UTILIZATION_LIMIT` on the OLTP group, or OLTP sessions mapped elsewhere | `V$RSRC_SESSION_INFO.MAPPING_ATTRIBUTE`, `V$RSRCMGRMETRIC` |
| Users land in `OTHER_GROUPS` | Rule exists but no switch privilege; or the group has no directive in the active plan | `DBA_RSRC_CONSUMER_GROUP_PRIVS`, `DBA_RSRC_PLAN_DIRECTIVES` |
| `SWITCH_TIME` never fires | SQL Monitoring off (`NO_MONITOR`, pack access `NONE`) | Use `SWITCH_ELAPSED_TIME` |
| ORA-07454 | `QUEUEING_P1` or `PARALLEL_QUEUE_TIMEOUT` expired | Raise timeout or `PQ_TIMEOUT_ACTION => 'RUN'` |
| ORA-10260 | `SESSION_PGA_LIMIT` exceeded | `V$PROCESS.PGA_USED_MEM`; fix SQL or raise limit |
| Plan changes at 22:00 | Maintenance window carries `DEFAULT_MAINTENANCE_PLAN` | `DBA_SCHEDULER_WINDOWS.RESOURCE_PLAN`; `FORCE:` |
| Nothing is limited | No plan (`V$RSRC_PLAN.CPU_MANAGED = 'OFF'`) | `V$RSRC_PLAN` |
| `resmgr: I/O rate limit` | `MAX_IOPS`/`MAX_MBPS` set too low | `DBA_HIST_RSRC_PDB_METRIC` |

---

## Best Practices

- **Dry-run with `LOG_ONLY`** for a week, read `V$SQL_MONITOR.RM_LAST_ACTION*`, then switch to `CANCEL_SQL`/`KILL_SESSION`.
- **Map by service (and module), not by user.** Shared application schemas make `ORACLE_USER` rules useless; give OLTP and batch their own services.
- **`SWITCH_FOR_CALL => TRUE`** for anything that uses connection pools.
- **Use `UTILIZATION_LIMIT` only on the groups you are protecting *against*.** Shares already redistribute idle CPU; a limit deliberately wastes it.
- **Keep `SYS_GROUP` at level 1 and `OTHER_GROUPS` with a non-zero share** so administrators and unmapped sessions always progress.
- **Grant groups to roles**, and prefix your own plans/groups (`APP_`, `WLM_`, …) so they are never confused with Oracle-supplied ones.
- **Watch `V$RSRCMGRMETRIC_HISTORY`**: `AVG_CPU_UTILIZATION` near `CPU_UTILIZATION_LIMIT` with growing `CPU_WAIT_TIME` means a limit is active — confirm it is intended.

## Common Mistakes

- **No `OTHER_GROUPS` directive** → ORA-29382/ORA-29377 at validation.
- **Mapping without `GRANT_SWITCH_CONSUMER_GROUP`** → sessions silently stay in `DEFAULT_CONSUMER_GROUP`/`OTHER_GROUPS`.
- **Expecting `SWITCH_TIME` without Diagnostics/Tuning Pack** → use `SWITCH_ELAPSED_TIME`.
- **Putting application groups at level 1 next to `SYS_GROUP`** → starves every lower level. Level 1 is for administrators.
- **`UTILIZATION_LIMIT` on the OLTP group** → OLTP waits on `resmgr:cpu quantum` on an idle box.
- **Clearing `RESOURCE_MANAGER_PLAN` and being surprised at 22:00** → maintenance windows re-enable `DEFAULT_MAINTENANCE_PLAN`; use `FORCE:` or edit the windows.
- **Reusing shipped names** (`BATCH_GROUP`, `INTERACTIVE_GROUP`, `DSS_PLAN`, …) for your own objects → confusing diagnostics and collisions with example plans.

---

## Oracle Version Notes (19c vs 26ai)

- **19c baseline.** Everything above — groups, plans, directives, mapping rules, all `SWITCH_*` controls, `SESSION_PGA_LIMIT` and `PQ_TIMEOUT_ACTION` (12.2+), `MAX_IOPS`/`MAX_MBPS` (12.2+), instance caging, all `V$RSRC*` views — exists in 19c.
- **19c:** SQL Quarantine (`DBMS_SQLQ`, `DBA_SQL_QUARANTINE`); platform-restricted.
- **26ai only:** `RESOURCE_MANAGER_CPU_SCOPE = SERVER_WIDE` (cgroup-based inter-instance CPU management; Exadata/Autonomous, UEK 5+). `IORM_LIMIT_POLICY` (`RM_PLAN` default \| `CPU_COUNT_RATIO`) controls how Exadata IORM derives PDB shares and flash utilization limits from the CDB plan or `CPU_COUNT`/`CPU_MIN_COUNT`. **Auto Redo Prioritization** (RU 23.26.3+): Resource Manager automatically throttles redo from batch-behaving sessions under log-buffer pressure (`LOG_REDO_PRIORITIZATION = TRUE`, dynamic, default `FALSE`) or when an async Data Guard standby's transport lag exceeds `REDO_PRIORITIZATION_WHEN_TRANSPORT_LAGS_BY_SEC` on `LOG_ARCHIVE_DEST_n`; no consumer-group setup needed; OLTP-behaving sessions unaffected.
- **Deprecated in both releases** (still accepted): `CPU_Pn`, `MAX_UTILIZATION_LIMIT`, `PARALLEL_TARGET_PERCENTAGE`, `SWITCH_TIME_IN_CALL`, `SET_INITIAL_CONSUMER_GROUP`. Write new code with `MGMT_Pn`, `UTILIZATION_LIMIT`, `PARALLEL_SERVER_LIMIT`, `SWITCH_TIME` + `SWITCH_FOR_CALL`, and mapping rules.

## See Also

- [Multitenant Architecture](../architecture/multitenant.md) — CDB plans (`SHARES`, `UTILIZATION_LIMIT` per PDB), `CPU_COUNT`/`CPU_MIN_COUNT` per PDB
- [Exadata Features](../architecture/exadata-features.md) — IORM: how the DBRM plan becomes the storage-side intradatabase plan; `dbplan`; `IORM_LIMIT_POLICY`
- [DBMS_SCHEDULER](../features/dbms-scheduler.md) — windows that switch plans; job classes that map jobs to consumer groups
- [Wait Events](../performance/wait-events.md) — `resmgr:*` waits in context
- [Memory Tuning](../performance/memory-tuning.md) — PGA sizing behind `SESSION_PGA_LIMIT`

## Sources

- [Oracle Database Administrator's Guide 19c — Managing Resources with Oracle Database Resource Manager](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-resources-with-oracle-database-resource-manager.html)
- [Oracle AI Database Administrator's Guide 26ai — Managing Resources with Oracle AI Database Resource Manager](https://docs.oracle.com/en/database/oracle/oracle-database/26/admin/managing-resources-with-oracle-database-resource-manager.html)
- [PL/SQL Packages and Types Reference 19c — DBMS_RESOURCE_MANAGER](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER.html)
- [PL/SQL Packages and Types Reference 19c — DBMS_RESOURCE_MANAGER_PRIVS](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER_PRIVS.html)
- [PL/SQL Packages and Types Reference 19c — DBMS_SQLQ](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQLQ.html)
- [Database Reference 19c — DBA_RSRC_PLAN_DIRECTIVES](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_RSRC_PLAN_DIRECTIVES.html)
- [Database Reference 19c — MAX_IOPS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/MAX_IOPS.html)
- [AI Database Reference 26ai — DBA_HIST_RSRC_PDB_METRIC](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/DBA_HIST_RSRC_PDB_METRIC.html)
- [AI Database Reference 26ai — RESOURCE_MANAGER_CPU_SCOPE](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/RESOURCE_MANAGER_CPU_SCOPE.html)
- [AI Database Reference 26ai — IORM_LIMIT_POLICY](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/IORM_LIMIT_POLICY.html)
- [Database Error Messages — ORA-00040](https://docs.oracle.com/en/error-help/db/ora-00040/), [ORA-07455](https://docs.oracle.com/en/error-help/db/ora-07455), [ORA-30027](https://docs.oracle.com/en/error-help/db/ora-30027), [ORA-56739](https://docs.oracle.com/en/error-help/db/ora-56739)
- [Oracle MAA Blog — Auto Redo Prioritization](https://blogs.oracle.com/maa/auto-redo-prioritization-helping-manage-redo-generation-spikes)
