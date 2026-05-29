# Database-BenchMarking-Optimization-Framework
Persistent SQL Server performance monitoring built on Extended Events - captures slow queries, waits, locks, and memory pressure into tables with incremental, idempotent loading.
The collector reads the rolling `.xel` file from where it last stopped, using a `(file_name, file_offset, buffer_sequence)` watermark — so no event is loaded twice and no event is missed.

## Prerequisites

- SQL Server 2016 or later (uses `query_memory_grant_usage`, `STRING_AGG`-style patterns optional)
- `VIEW SERVER STATE` permission for the account running the load proc
- A writable folder for `.xel` files (default: `C:\DatabaseHealth\`)
- A monitoring database to host the `XE_*` tables (can be any DB you control)


## Installation

### 1. Create the XE session

### 2. Create the tables and watermark

This creates 'XE_QueryPerformanceMetrics', 'XE_WaitEvents', 'XE_LockContentions', 'XE_MemorySpills', 'XE_MemoryGrants', 'XE_LoadWatermark', and 'XE_DailyMetricsAggregated'.

### 3. Create the stored procedures

Optimization_LoadXEQueryPerformance.sql      -- incremental load
Optimization_AggregateDailyMetrics.sql       -- daily roll-up
Optimization_CollectXEMetricsScheduled.sql          -- 30-day retention

### 4. Schedule via SQL Agent

| Job step | Frequency | Procedure |
|---|---|---|
| Load XE data | Every 5–15 min | Optimization_LoadXEQueryPerformance |
| Daily aggregation | Once at 00:30 UTC | Optimization_AggregateDailyMetrics |
| Cleanup + reload | Once at 01:00 UTC | Optimization_CollectXEMetricsScheduled |

---

## Usage

### Run the full dashboard

Full_Dashboard.sql

Returns 8 result sets covering:
1. Real-time overview (last hour)
2. Slow query detective work (top 50, recurring, percentiles by DB)
3. Wait analysis (top types, categorized, by database)
4. Lock contention (deadlocks vs timeouts, recent events)
5. Memory (spill summary, over- and under-granted queries, spill↔grant correlation)
6. Trend analysis (hourly slow-query trend, multi-signal correlation)
7. Plan regression detection (last-hour vs 7-day baseline)
8. Health-check infrastructure (watermark age, row counts, session status)

Scope to one database by setting '@DatabaseName' at the top of the script. Leave 'NULL' for server-wide.


## Configuration

### XE session thresholds

Defined in `01_create_event_session.sql`. Defaults:

| Event | Threshold | Notes |
|---|---|---|
| `rpc_completed` | duration > 1000 µs, database_id > 4 | Excludes system DBs |
| `sql_batch_completed` | duration > 5000 µs | |
| `sp_statement_completed` | duration > 1000 µs | |
| `wait_info` | opcode = 1 AND duration > 1000 ms | `opcode = 1` = End only; without this, volume is unmanageable |
| Others | unfiltered | |

**Units to remember**: completed-query `duration` is in **microseconds** (load proc divides by 1000); `wait_info.duration` is in **milliseconds** straight through. The schema stores everything as ms, but the conversion happens in the load.

### File location

Change 'C:\DatabaseHealth\' in 'CreateEventSession' and 'Optimization_LoadXEQueryPerformance.sql' to point wherever you want the '.xel' files. The folder must exist and be writable by the SQL Server service account.

### Retention

Default is 30 days, defined in `Optimization_CollectXEMetricsScheduled`. The '.xel' files themselves are bounded by 'max_file_size = 100' MB × 'max_rollover_files = 10' ≈ 1 GB on disk.


## How the incremental load works (the part worth understanding)

'sys.fn_xe_file_target_read_file' is deterministic — given a '(file_name, offset)' it always returns events from there in the same order. Many events share a 'file_offset' (it's the page offset, not per-event), so we add 'ROW_NUMBER() OVER (PARTITION BY file_name, file_offset)' to get a stable third coordinate.

The watermark stores '(LastFileName, LastFileOffset, LastBufferSeq)'. Each run:

1. Reads from the watermark forward
2. Filters to events strictly greater than the triple
3. Inserts into all five target tables in one transaction
4. Advances the watermark to the last event read

If anything fails, the transaction rolls back and the watermark stays put. Next run reads the same window again with nothing committed to collide with. No unique constraints needed; correctness comes from the watermark, not from event content (timestamps aren't unique either, which is why naive dedup fails).

Rollover safety: if the `.xel` file the watermark points at has been rolled off disk, the load detects it and resets to the start of available data rather than silently doing nothing.

