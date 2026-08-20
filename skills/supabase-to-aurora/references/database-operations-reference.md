# Database Operations Reference — PostgreSQL

General-purpose queries for day-to-day database operations on PostgreSQL (Aurora, RDS, or any standard PostgreSQL instance). These queries work with any schema — no application-specific assumptions.

## Schema Exploration

### List All Tables with Sizes

```sql
SELECT schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
  pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
```

### List All Tables with Row Counts

```sql
SELECT schemaname, relname AS tablename, n_live_tup AS estimated_rows,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
```

### Database Size Summary

```sql
SELECT pg_size_pretty(pg_database_size(current_database())) AS database_size;

-- Size by schema
SELECT schemaname,
  COUNT(*) AS table_count,
  pg_size_pretty(SUM(pg_total_relation_size(schemaname || '.' || tablename))) AS total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY SUM(pg_total_relation_size(schemaname || '.' || tablename)) DESC;
```

### List Columns for a Table

```sql
SELECT column_name, data_type, is_nullable, column_default,
  character_maximum_length, numeric_precision
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = '<table_name>'
ORDER BY ordinal_position;
```

### List All ENUM Types

```sql
SELECT t.typname AS enum_name,
  string_agg(e.enumlabel, ', ' ORDER BY e.enumsortorder) AS values
FROM pg_type t
JOIN pg_enum e ON t.oid = e.enumtypid
GROUP BY t.typname
ORDER BY t.typname;
```

### List All Extensions

```sql
SELECT extname, extversion FROM pg_extension ORDER BY extname;
```

## Index Operations

### List All Indexes with Usage Stats

```sql
SELECT schemaname, tablename, indexname, idx_scan,
  idx_tup_read, idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### Find Unused Indexes (Candidates for Removal)

```sql
SELECT schemaname, relname AS tablename, indexrelname AS indexname,
  idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Find Missing Indexes (High Sequential Scan Tables)

```sql
SELECT schemaname, relname AS tablename, seq_scan, seq_tup_read, idx_scan,
  CASE WHEN seq_scan > 0 THEN seq_tup_read / seq_scan ELSE 0 END AS avg_rows_per_scan,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND pg_total_relation_size(schemaname || '.' || relname) > 1048576
ORDER BY seq_tup_read DESC
LIMIT 20;
```

### Duplicate or Overlapping Indexes

```sql
SELECT a.indrelid::regclass AS table_name,
  a.indexrelid::regclass AS index_a,
  b.indexrelid::regclass AS index_b
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
  AND a.indexrelid != b.indexrelid
  AND a.indkey::text LIKE b.indkey::text || '%'
WHERE a.indrelid::regclass::text NOT LIKE 'pg_%';
```

## Constraint and Relationship Inspection

### List All Foreign Keys

```sql
SELECT tc.table_schema, tc.table_name, kcu.column_name,
  ccu.table_schema AS foreign_schema, ccu.table_name AS foreign_table,
  ccu.column_name AS foreign_column, tc.constraint_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name AND tc.table_schema = kcu.table_schema
JOIN information_schema.constraint_column_usage ccu
  ON ccu.constraint_name = tc.constraint_name AND ccu.table_schema = tc.table_schema
WHERE tc.constraint_type = 'FOREIGN KEY'
ORDER BY tc.table_schema, tc.table_name;
```

### List All Check Constraints

```sql
SELECT tc.table_schema, tc.table_name, tc.constraint_name, cc.check_clause
FROM information_schema.table_constraints tc
JOIN information_schema.check_constraints cc
  ON tc.constraint_name = cc.constraint_name AND tc.constraint_schema = cc.constraint_schema
WHERE tc.constraint_type = 'CHECK'
  AND tc.table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY tc.table_schema, tc.table_name;
```

### Tables Without Primary Keys

```sql
SELECT t.schemaname, t.relname AS tablename
FROM pg_stat_user_tables t
LEFT JOIN pg_constraint c
  ON c.conrelid = (t.schemaname || '.' || t.relname)::regclass AND c.contype = 'p'
WHERE c.conname IS NULL
ORDER BY t.schemaname, t.relname;
```

### List All Sequences

```sql
SELECT schemaname, sequencename, last_value, start_value, increment_by, max_value
FROM pg_sequences
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, sequencename;
```

## RLS Policy Inspection

### List All RLS Policies

```sql
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual, with_check
FROM pg_policies
ORDER BY schemaname, tablename, policyname;
```

### Tables with RLS Enabled

```sql
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE rowsecurity = true
ORDER BY schemaname, tablename;
```

## Active Session and Connection Monitoring

### Current Active Queries

```sql
SELECT pid, usename, application_name, client_addr, state,
  now() - query_start AS query_duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND pid != pg_backend_pid()
ORDER BY query_start;
```

### Connection Summary by State

```sql
SELECT state, usename, COUNT(*) AS connections
FROM pg_stat_activity
GROUP BY state, usename
ORDER BY connections DESC;
```

### Long-Running Queries (> 1 minute)

```sql
SELECT pid, usename, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '1 minute'
ORDER BY query_start;
```

### Blocked Queries (Lock Waits)

```sql
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
  blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
  AND kl.database IS NOT DISTINCT FROM bl.database
  AND kl.relation IS NOT DISTINCT FROM bl.relation
  AND kl.page IS NOT DISTINCT FROM bl.page
  AND kl.tuple IS NOT DISTINCT FROM bl.tuple
  AND kl.transactionid IS NOT DISTINCT FROM bl.transactionid
  AND kl.classid IS NOT DISTINCT FROM bl.classid
  AND kl.objid IS NOT DISTINCT FROM bl.objid
  AND kl.objsubid IS NOT DISTINCT FROM bl.objsubid
  AND kl.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted;
```

## Troubleshooting

### Table Bloat Detection

```sql
SELECT schemaname, relname AS tablename,
  n_dead_tup, n_live_tup,
  round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_autovacuum, last_autoanalyze,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Cache Hit Ratio

```sql
SELECT
  sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS table_cache_hit_ratio,
  sum(idx_blks_hit) / NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0) AS index_cache_hit_ratio
FROM pg_statio_user_tables;
```

### Top Queries by Total Time (requires pg_stat_statements)

```sql
SELECT query, calls, total_exec_time, mean_exec_time, rows,
  round(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Replication Status (if applicable)

```sql
-- Check outgoing replication (source side)
SELECT pid, usename, application_name, client_addr, state,
  sent_lsn, write_lsn, flush_lsn, replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Check incoming subscription (target side)
SELECT subname, subenabled, subslotname
FROM pg_stat_subscription;

-- Check replication slot status
SELECT slot_name, active,
  pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_pretty
FROM pg_replication_slots;
```

### Vacuum and Analyze Status

```sql
SELECT schemaname, relname, last_vacuum, last_autovacuum,
  last_analyze, last_autoanalyze, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST
LIMIT 20;
```

## Maintenance Operations

### Gather Statistics (After Migration or Bulk Load)

```sql
ANALYZE VERBOSE;
```

### Reset Sequences After Migration

Native logical replication does not replicate sequences. After cutover, reset each sequence:

```sql
-- Generate reset commands for all serial/identity columns
SELECT 'SELECT setval(pg_get_serial_sequence(''' || schemaname || '.' || tablename || ''', ''' || column_name || '''), COALESCE((SELECT MAX(' || column_name || ') FROM ' || schemaname || '.' || tablename || '), 1));'
FROM information_schema.columns c
JOIN pg_tables t ON c.table_schema = t.schemaname AND c.table_name = t.tablename
WHERE c.column_default LIKE 'nextval%'
  AND c.table_schema NOT IN ('pg_catalog', 'information_schema');
```

### Terminate a Specific Backend

```sql
-- Graceful cancel (query only)
SELECT pg_cancel_backend(<pid>);

-- Force terminate (entire connection) — use with caution
SELECT pg_terminate_backend(<pid>);
```
