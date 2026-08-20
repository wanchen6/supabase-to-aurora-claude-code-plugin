# Migration Strategy Reference — Supabase to Aurora PostgreSQL

Quick-reference guide for migration approach selection, validation strategy, step-by-step flow summaries, and cutover planning. Use this as a decision-making companion alongside the detailed phase files (01–04).

## Migration Context Template

Capture these details before starting any migration:

```
Source:      Supabase (<plan>) — <project-ref> — PG <version> — <size>
Target:      Aurora PostgreSQL <version> — <cluster-id> — <region>
Database:    <database-name>
Path:        <offline | native-replication | dms | hybrid>
Supabase Features: <RLS | Auth | Storage | Realtime | Edge Functions | PostgREST>
Validation:  <spot-check | checksums | dms-online>
Application: <framework> on <hosting>
```

## Approach Comparison

| Approach | Best For | Downtime | Tools |
| -------- | -------- | -------- | ----- |
| pg_dump/pg_restore | < 100 GB, simple schemas, non-production | Hours (full downtime) | pg_dump, pg_restore |
| Native Logical Replication (preferred) | Production, zero-downtime, tables with PKs | Minutes (cutover only) | Built-in PG 10+ logical replication |
| AWS DMS with CDC (fallback) | Tables where native replication fails, need validation | Minutes (cutover only) | AWS DMS (serverless), Secrets Manager |
| Hybrid (native + DMS + offline) | Mixed schemas, some tables without PKs | Minutes (cutover) + brief window for PK-less tables | All of the above |

## Approach Decision Tree

```
Is downtime acceptable?
├── YES → Offline (pg_dump/pg_restore)
│         Simple, reliable, works for any schema
│         Best for: < 100 GB or non-production
│
└── NO → Online (CDC) required
          │
          Do all tables have primary keys?
          ├── YES → Try Native Logical Replication first
          │         If any table fails → use DMS for that table (hybrid)
          │
          └── NO → Hybrid approach:
                   ├── Tables WITH PKs → Native Replication or DMS
                   └── Tables WITHOUT PKs → Offline (pg_dump per table)
```

**Rule of thumb for online migration path selection:**
- **Source has IPv4** (A record resolves) → Use native PostgreSQL logical replication (preferred — simpler, lower latency, no DMS infrastructure)
- **Source is IPv6-only** (only AAAA record) → Use DMS with CDC (Aurora cannot do outbound IPv6; DMS has its own public IP and works)
- **Fallback for either**: DMS for tables where native replication fails (missing PK, unsupported type, or when built-in validation is needed)

**IMPORTANT — IPv6-only sources (e.g., Supabase without IPv4 add-on):** Native logical replication does NOT work when the source is IPv6-only because Aurora cannot make outbound IPv6 connections (confirmed limitation — see `supabase-prerequisites-and-connectivity.md` → "Aurora Outbound IPv6 — Confirmed Limitation"). In this case:

| Approach | Cost/week | Complexity | Downtime |
|----------|-----------|------------|----------|
| DMS (dms.t3.micro, publicly-accessible, dual-stack) | ~$3 | Medium | Near-zero (CDC) |
| Native replication + NAT gateway + Supabase IPv4 add-on | ~$9 | Low | Near-zero (CDC) |
| pg_dump at cutover (bastion with IPv6) | ~$0.01 | Lowest | Seconds |

## Validation Strategy by Database Size

```
Database size?
├── < 10 GB   → Offline: row counts + spot checks on 5–10 key tables
├── 10–100 GB → Offline: row counts + sample MD5 checksums on all tables
└── > 100 GB  → DMS online validation (continuous, row-by-row)
                 Creates awsdms_validation_failures_v1 table on target
                 Monitor via CloudWatch metrics
```

## Flow Summary: Offline (pg_dump/pg_restore)

1. Put application in maintenance mode — stop writes
2. Export from Supabase:
   ```bash
   pg_dump -h db.<project-ref>.supabase.co -U postgres -d postgres \
     -N auth -N storage -N realtime -N supabase_functions -N extensions \
     --no-owner --no-acl -F c -f supabase_dump.backup
   ```
3. Transfer dump to Aurora-accessible host (EC2 bastion, S3, EFS)
4. Restore to Aurora:
   ```bash
   pg_restore --no-owner --no-acl \
     -h <aurora-endpoint> -U postgres -d <database-name> \
     -j 4 supabase_dump.backup
   ```
5. Run `ANALYZE VERBOSE;`
6. Validate row counts (source vs target)
7. Proceed to cutover

## Flow Summary: Native Logical Replication (Preferred Online Path)

### Prerequisites
- Supabase instance XL or greater
- External connectivity: Aurora CANNOT make outbound IPv6 connections (confirmed limitation). For online migration from IPv6-only Supabase, use DMS (publicly-accessible, dual-stack) or pg_dump via bastion. Native replication requires Supabase IPv4 add-on + NAT gateway. See `supabase-prerequisites-and-connectivity.md` → "Aurora Outbound IPv6 — Confirmed Limitation".
- Replication parameters configured via Supabase CLI:
  ```bash
  supabase --experimental --project-ref <ref> postgres-config update \
    --config max_replication_slots=10 \
    --config max_wal_senders=10 \
    --config wal_sender_timeout=0 \
    --config max_slot_wal_keep_size=4GB \
    --config max_wal_size=2GB
  ```

### Steps
1. **On Supabase** — Create publication and replication slot:
   ```sql
   CREATE PUBLICATION supabase_to_aurora;
   ALTER PUBLICATION supabase_to_aurora SET TABLE ALL IN SCHEMA public;
   SELECT pg_create_logical_replication_slot('supabase_to_aurora_slot', 'pgoutput');
   ```
2. **On Aurora** — Create subscription:
   ```sql
   CREATE SUBSCRIPTION supabase_sub
   CONNECTION 'host=db.<ref>.supabase.co user=postgres password=<pw> dbname=postgres'
   PUBLICATION supabase_to_aurora
   WITH (copy_data = true, create_slot = false, slot_name = 'supabase_to_aurora_slot');
   ```
3. Monitor initial data copy and replication lag
4. Wait for lag < 1 MB before cutover
5. Proceed to cutover

### When Native Replication Won't Work
- Tables without primary keys (UPDATE/DELETE won't replicate)
- Unsupported data types for logical replication
- DDL changes (not replicated)
- **Fallback**: Use DMS for those specific tables


## Flow Summary: AWS DMS with CDC (Fallback Online Path)

### Supabase-Specific DMS Configuration
- Source endpoint: use `postgres` user
- Plugin: `test_decoding` (NOT pglogical — Supabase doesn't support pglogical)
- Extra connection attributes: `pluginName=test_decoding;heartbeatEnable=true;heartbeatFrequency=5;mapBooleanAsBoolean=true`
- Increase `wal_sender_timeout` via Supabase CLI before starting

### Steps
1. Create Secrets Manager entries for source and target credentials
2. Create DMS subnet group, replication instance, and endpoints
3. Test endpoint connections
4. Create replication task with `full-load-and-cdc` migration type
5. Enable `ValidationSettings.EnableValidation = true` for online validation
6. Start the task and monitor via CloudWatch:
   - `CDCLatencySource`, `CDCLatencyTarget`, `CDCIncomingChanges`
   - `ValidationSucceededRecordCount`, `ValidationFailedOverallCount`
7. Wait for CDC lag < 5 seconds and all tables validated
8. Proceed to cutover

### Key DMS Limitations (PostgreSQL Source)
- Tables MUST have primary key for CDC DELETE/UPDATE
- TRUNCATE not supported in CDC
- ENUM types migrate as STRING(64) — create native ENUM types on target first if needed
- INET maps to STRING(50), MACADDR to STRING(18), TSVECTOR to CLOB, TSQUERY to CLOB — all lose native type semantics
- NUMERIC without precision defaults to NUMERIC(28,6)
- Large transactions may cause DMS to spill cached changes from memory to disk, increasing replication instance storage usage

## Flow Summary: Hybrid Approach

For databases with mixed requirements:

| Table Category | Migration Path |
| -------------- | -------------- |
| Tables with PKs, standard types | Native logical replication (preferred) |
| Tables with PKs needing validation | DMS with CDC + validation |
| Tables without PKs | Offline (pg_dump per table during brief maintenance window) |

**Execution order**:
1. Start native replication for the primary table set
2. Start DMS for tables needing validation or where native replication failed
3. After both are caught up, take a brief window for PK-less tables (pg_dump/pg_restore per table)
4. Proceed to cutover

## Post-Load Steps (All Approaches)

After data load completes — regardless of migration path:

1. Create indexes on target (if deferred during load for performance)
2. Create triggers on target (if deferred)
3. Gather statistics: `ANALYZE VERBOSE;`
4. Validate row counts and object counts between source and target
5. Verify enum types and custom types are present on target
6. Verify functions and stored procedures are in place
7. Test application read and write operations

## Connection Configuration Patterns

### Pattern 1: Environment Variables (PG* vars)

```env
PGHOST=<aurora-cluster-endpoint>
PGPORT=5432
PGDATABASE=<database-name>
PGUSER=<username>
PGPASSWORD=<password-from-secrets-manager>
PGSSLMODE=require
```

### Pattern 2: DATABASE_URL (Single Connection String)

```env
DATABASE_URL=postgresql://<username>:<password>@<aurora-endpoint>:5432/<database-name>?sslmode=require
```

### Pattern 3: MCP Server Connection (Kiro Power)

Connect at runtime via Aurora PostgreSQL MCP:

```
region: "<aws-region>"
database_type: "APG"
connection_method: "pgwire"
db_endpoint: "<aurora-endpoint>"
port: 5432
database: "<database-name>"
```

Uses AWS Secrets Manager for credentials — no password in config.

## Cutover Checklist

### Pre-Cutover
- [ ] Replication lag near zero (native: < 1 MB, DMS: CDCLatencyTarget < 5s)
- [ ] Row counts match between source and target
- [ ] DMS validation passed (if applicable)
- [ ] Application maintenance window scheduled
- [ ] Rollback plan documented

### Cutover Execution
1. Stop application writes (maintenance mode)
2. Wait for replication to fully drain
3. Final row count comparison
4. **Reset sequences on Aurora** (native replication does NOT replicate sequences):
   ```sql
   SELECT setval('public.<table>_id_seq', (SELECT MAX(id) FROM public.<table>));
   ```
5. Update application connection config to Aurora endpoint:
   ```
   DATABASE_URL=postgresql://postgres:<pw>@<aurora-endpoint>:5432/<db>?sslmode=require
   ```
6. Restart application
7. Verify application health (logs, read/write ops, critical flows)

### Post-Cutover
- [ ] Monitor for 24–48 hours (error rates, latency, connections, CPU/memory)
- [ ] Stop replication after confirming everything works:
  - Aurora: `DROP SUBSCRIPTION supabase_sub;`
  - Supabase: `DROP PUBLICATION supabase_to_aurora;` and `SELECT pg_drop_replication_slot('supabase_to_aurora_slot');`
  - DMS: stop and delete replication task
- [ ] Confirm no orphaned replication slots on Supabase
- [ ] Run `ANALYZE VERBOSE;` on Aurora

## Data Validation Checklist (Post-Migration)

After migration, verify all of the following:

- [ ] Table counts match between Supabase source and Aurora target
- [ ] Row counts match for each table
- [ ] Enum types and custom types are present on target
- [ ] Indexes are created and being used (`pg_stat_user_indexes`)
- [ ] Triggers and functions are in place
- [ ] Sequences are reset to correct values (native replication does NOT replicate sequences)
- [ ] RLS policies are applied (if migrated)
- [ ] Application can read and write successfully
- [ ] No orphaned replication slots on Supabase (`pg_replication_slots`)

For Supabase schema exclusions and internal roles to remove, see `supabase-migration-feature-mapping.md` → "Supabase Schema Exclusions".
