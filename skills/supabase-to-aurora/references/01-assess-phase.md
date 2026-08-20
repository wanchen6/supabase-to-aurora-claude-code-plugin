# Assess Phase — PostgreSQL Migration

## Overview

The Assess phase discovers the current source database environment, evaluates compatibility with the target PostgreSQL instance (RDS or Aurora PostgreSQL), identifies platform-specific features requiring replacement, and determines the optimal migration approach.

## Migration Context

- Source: PostgreSQL databases hosted on 3rd-party platforms (Supabase, Heroku, Neon, GCP Cloud SQL, Azure Database, on-premises, etc.)
- Target: AWS RDS PostgreSQL or Amazon Aurora PostgreSQL
- Approach: Database-focused migration with application connection update or refactoring as needed
- Focus: Discover the source environment, assess compatibility, and select the migration strategy

## Outline

1. [Migration Decision Tree](#migration-decision-tree)
2. [Source Database Discovery](#source-database-discovery)
3. [Schema Compatibility Assessment](#schema-compatibility-assessment)
4. [Migration Approach Recommendation](#migration-approach-recommendation)
5. [Application-Database Mapping](#application-database-mapping)
6. [Performance Baseline & Sizing](#performance-baseline--sizing)
7. [Network & Security Assessment](#network--security-assessment)
8. [Deployment Platform Assessment](#deployment-platform-assessment)
9. [Parallel Execution Planning](#parallel-execution-planning)
10. [DMS Resource Planning](#dms-resource-planning)
11. [Risk Assessment](#risk-assessment)
12. [Cost Analysis (Pre-Migration)](#cost-analysis-pre-migration)

## Migration Decision Tree

Use this decision tree to guide the key decisions during the Assess phase. Each decision feeds into subsequent ones — work through them in order.

### Decision 1: Target Engine — RDS PostgreSQL vs Aurora PostgreSQL

```
Is the database < 10 GB with simple, predictable workload?
├── YES → RDS PostgreSQL (lower cost for small, steady workloads)
└── NO
    ├── Is the workload highly variable or unpredictable?
    │   └── YES → Aurora Serverless v2 (auto-scales 0–256 ACU)
    └── NO
        ├── Need built-in HA with typically < 60s failover (often < 30s with replicas)?
        │   └── YES → Aurora PostgreSQL (storage-level replication, up to 15 read replicas)
        └── NO → Aurora PostgreSQL (recommended default for production migrations)
```

Default recommendation: Aurora PostgreSQL — better scalability, faster failover, storage auto-scaling.

### Decision 2: Application Refactoring Scope

```
Does the application use a platform-specific SDK (e.g., Supabase JS, Heroku Connect)?
├── YES → SDK refactoring required
│   ├── Does the app use the SDK for data access (CRUD queries)?
│   │   └── YES → Replace SDK with standard PostgreSQL driver (pg, Prisma, Drizzle, etc.)
│   ├── Does the app use platform auth (e.g., Supabase Auth)?
│   │   └── YES → Replace with Amazon Cognito, Auth0, or custom JWT
│   ├── Does the app use platform storage (e.g., Supabase Storage)?
│   │   └── YES → Replace with Amazon S3
│   └── Does the app use platform realtime (e.g., Supabase Realtime)?
│       └── YES → Replace with AWS AppSync or WebSocket API Gateway
└── NO → Connection string update only (minimal application changes)
```

### Decision 3: Deployment Platform Connectivity

```
Where is the application hosted?
├── EC2 / ECS / EKS / Lambda (in VPC) → Direct private connection to Aurora ✓
├── AWS Amplify Hosting → VPC connectivity supported ✓
├── Vercel Enterprise → Secure Compute with PrivateLink ✓
├── Vercel Free/Pro, Netlify, Railway, Render → Cannot reach private Aurora
│   ├── Option A (preferred): Use Aurora Data API (HTTPS, no VPC needed)
│   ├── Option B: Migrate hosting to Amplify or ECS/Fargate
│   └── Option C: Upgrade to platform enterprise tier (if available)
└── On-premises / self-hosted → VPN or Direct Connect to AWS VPC
```

NEVER recommend making Aurora publicly accessible. If the platform can't reach private Aurora, use Data API or migrate hosting.

### Decision 4: Migration Approach

```
Is downtime acceptable during migration?
├── YES (maintenance window OK)
│   └── Database size?
│       ├── < 100 GB → Offline (pg_dump/pg_restore) — simplest, most reliable
│       └── ≥ 100 GB → Offline is possible but slow; consider online for shorter window
└── NO (zero or near-zero downtime required)
    └── Do ALL tables have primary keys?
        ├── YES → Native logical replication (preferred) or DMS with CDC
        │   ├── Need built-in row-by-row validation? → DMS with CDC + validation
        │   ├── Source supports logical replication? → Native logical replication (lower latency)
        │   └── Source doesn't support logical replication? → DMS with CDC
        └── NO (some tables lack PKs)
            └── Hybrid approach:
                ├── Tables WITH PKs → Native replication or DMS CDC
                └── Tables WITHOUT PKs → Offline (pg_dump per table) during cutover window
```

### Decision 5: DMS Involvement

```
Is DMS part of the migration strategy? (from Decision 4)
├── YES — DMS for full migration (full load + CDC)
│   └── Plan DMS resources: replication instance, endpoints, task, subnet group
├── YES — DMS for validation only (data migrated via native replication or pg_dump)
│   └── Create DMS validation-only task (TargetTablePrepMode: DO_NOTHING)
├── YES — DMS for specific tables only (hybrid approach)
│   └── Create DMS task with table-mapping rules for the subset of tables
└── NO — Not using DMS
    └── Plan offline validation: row counts, checksums, sample comparisons
```

### Decision 6: Instance Sizing

```
Is the workload pattern predictable and steady?
├── YES → Provisioned instance
│   ├── CPU avg < 30% on source → Start one size smaller, monitor and adjust
│   ├── CPU avg 30–60% → Match equivalent vCPU count
│   └── CPU avg > 60% → Size up; consider Graviton (db.r7g) for better price-performance
└── NO (variable, spiky, or unpredictable)
    └── Aurora Serverless v2 (auto-scales 0–256 ACU)
        └── Set min ACU based on baseline load, max ACU based on peak
```

### Decision 7: RLS Policy Handling

```
Does the source database use Row-Level Security (RLS)?
├── NO → No action needed
└── YES
    ├── Is the application moving to direct PostgreSQL connections?
    │   └── YES → Keep RLS on Aurora (natively supported) — migrate policies as-is
    └── Is the application using Aurora Data API or an API layer?
        └── YES → Consider moving authorization to the application/API layer
            └── RLS on Aurora still works but may be redundant if the API enforces access control
```

### Decision 8: Cutover Strategy

```
Was an online migration approach selected? (from Decision 4)
├── YES (CDC is running)
│   ├── Can you do a rolling deployment? (preview/staging → promote to production)
│   │   └── YES → Rolling cutover (Option A in Migrate phase) — lowest risk
│   └── NO → Merge-and-deploy cutover (Option B in Migrate phase)
└── NO (offline migration)
    └── Schedule maintenance window
        ├── Stop writes → dump → restore → switch connection → verify
        └── Window size depends on database size (estimate from pg_dump timing)
```

### Decision 9: Rollback Strategy

```
Is this a production-critical database?
├── YES
│   ├── Can you provision a fall-forward replica (A')? → Strategy 2: Fall Forward (recommended)
│   ├── Is the system multi-tenant / partitioned? → Strategy 3: Dual Write
│   └── Neither practical? → Strategy 4: DMS Bidirectional Replication
└── NO (dev/staging/non-critical)
    └── Strategy 1: Basic Fallback (revert connection string, accept post-cutover data loss)
```

### Decision 10: Cost Estimation Scope

```
Does the source platform bundle non-database features (auth, storage, realtime, functions)?
├── NO (database only) → Compare database costs only
│   └── Source compute + storage vs Aurora compute + storage + I/O + backup
└── YES (bundled platform features)
    ├── Are you replacing those features with AWS services?
    │   ├── YES → Include replacement service costs in the comparison
    │   │   └── Auth → Cognito, Storage → S3, Realtime → AppSync, Functions → Lambda
    │   └── NO (keeping some on the source platform or using third-party)
    │       └── Include only the services you're moving to AWS
    └── Cost comparison must show:
        ├── Current total platform cost (all-in)
        ├── Aurora database cost (compute + storage + I/O + backup + transfer)
        ├── AWS replacement service costs (if applicable)
        └── Net savings or increase with notes on RI/Graviton/Serverless v2 potential
```

Use the AWS Pricing MCP server to query actual pricing — do NOT estimate from memory.

### Decision Summary Template

After working through the decisions above, document the results:

| Decision | Choice | Justification |
| -------- | ------ | ------------- |
| Target engine | Aurora PostgreSQL / RDS PostgreSQL / Aurora Serverless v2 | |
| App refactoring scope | Connection string only / SDK refactoring required | |
| Connectivity method | Direct VPC / Aurora Data API / VPN | |
| Migration approach | Offline / Native replication / DMS CDC / Hybrid | |
| DMS involvement | Full migration / Validation only / Specific tables / None | |
| Instance sizing | Provisioned (instance type) / Serverless v2 (min–max ACU) | |
| RLS handling | Keep on Aurora / Move to application layer / N/A | |
| Cutover strategy | Rolling deployment / Merge and deploy / Maintenance window | |
| Rollback strategy | Basic fallback / Fall forward / Dual write / Bidirectional | |
| Cost estimation scope | Database only / Database + replacement services | |

Present this summary to the customer for review before proceeding to the Mobilize phase.

## Source Database Discovery

### Database Inventory

Connect to the source database and run these discovery queries:

```sql
-- Database size
SELECT pg_size_pretty(pg_database_size(current_database())) AS db_size;

-- PostgreSQL version
SELECT version();

-- Table count and sizes by schema
SELECT schemaname, COUNT(*) AS table_count,
  pg_size_pretty(SUM(pg_total_relation_size(schemaname || '.' || tablename))) AS total_size
FROM pg_stat_user_tables
GROUP BY schemaname
ORDER BY SUM(pg_total_relation_size(schemaname || '.' || tablename)) DESC;

-- Detailed table inventory with row counts
SELECT schemaname, relname, n_live_tup,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || relname) DESC;

-- Tables WITHOUT primary keys (critical for CDC planning)
SELECT t.schemaname, t.relname
FROM pg_stat_user_tables t
LEFT JOIN pg_indexes i ON t.schemaname = i.schemaname AND t.relname = i.tablename
  AND i.indexname LIKE '%_pkey'
WHERE i.indexname IS NULL
ORDER BY t.schemaname, t.relname;
```

### Extensions Audit

```sql
-- Installed extensions
SELECT extname, extversion FROM pg_extension ORDER BY extname;
```

Classify each extension into:

- **Compatible with target**: pgcrypto, uuid-ossp, pg_stat_statements, postgis, pg_trgm, btree_gist, btree_gin, hstore, citext, intarray, ltree, tablefunc, unaccent, fuzzystrmatch, pg_buffercache
- **Platform-specific (may not be available on target)**: varies by source platform — check target documentation
- Use AWS Knowledge MCP `search_documentation` to verify extension availability on Aurora/RDS

### Platform-Specific Feature Assessment

Identify features tied to the source platform that need replacement on AWS. Common examples:

| Source Platform | Feature | AWS Replacement |
| --------------- | ------- | --------------- |
| Supabase | Auth (`auth.*` schema) | Amazon Cognito, Auth0, custom JWT |
| Supabase | Storage (`storage.*` schema) | Amazon S3 |
| Supabase | Realtime subscriptions | AWS AppSync, WebSocket API Gateway |
| Supabase | Edge Functions | AWS Lambda, Lambda@Edge |
| Supabase | PostgREST API | API Gateway + Lambda, direct DB connection |
| Heroku | Heroku Connect (Salesforce sync) | AWS AppFlow, custom integration |
| Heroku | Heroku Data Clips | Amazon QuickSight, custom reporting |
| Neon | Branching | Aurora cloning, snapshots |
| Neon | Autoscaling to zero | Aurora Serverless v2 (scales to 0.5 ACU minimum) |
| GCP Cloud SQL | Cloud SQL Auth Proxy | RDS Proxy, IAM database authentication |
| Azure Database | Azure AD authentication | IAM database authentication |
| On-premises | Custom HA/failover | Aurora built-in HA, Multi-AZ |

For each platform-specific feature in use, document:
1. What the feature does
2. Whether the application depends on it
3. The recommended AWS replacement
4. Migration effort estimate

## Schema Compatibility Assessment

### Object Inventory

```sql
-- Complete object count (adjust excluded schemas for your source platform)
SELECT 'tables' AS object_type, COUNT(*) FROM information_schema.tables
  WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
UNION ALL
SELECT 'views', COUNT(*) FROM information_schema.views
  WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
UNION ALL
SELECT 'functions', COUNT(*) FROM information_schema.routines
  WHERE routine_schema NOT IN ('pg_catalog', 'information_schema')
UNION ALL
SELECT 'indexes', COUNT(*) FROM pg_indexes
  WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
UNION ALL
SELECT 'triggers', COUNT(*) FROM information_schema.triggers
  WHERE trigger_schema NOT IN ('pg_catalog', 'information_schema')
UNION ALL
SELECT 'sequences', COUNT(*) FROM information_schema.sequences
  WHERE sequence_schema NOT IN ('pg_catalog', 'information_schema');
```

### Data Type Compatibility

```sql
-- Data types in use (check for DMS-incompatible types)
SELECT DISTINCT data_type, udt_name
FROM information_schema.columns
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY data_type;
```

Flag these types for DMS limitations:

- ENUM — DMS maps to STRING(64); create ENUM type manually on target if native ENUM is needed
- INET, MACADDR — DMS converts to STRING/CLOB (loses native type semantics)
- TSVECTOR, TSQUERY — DMS converts to STRING/CLOB (loses full-text search functionality)
- NUMERIC without precision — DMS defaults to NUMERIC(28,6)
- ARRAY columns — require primary key for DMS CDC

### Foreign Key Dependencies

```sql
SELECT tc.table_schema, tc.table_name, kcu.column_name,
  ccu.table_schema AS foreign_table_schema, ccu.table_name AS foreign_table_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
ORDER BY tc.table_schema, tc.table_name;
```

### RLS Policies Assessment

```sql
-- All RLS policies
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual, with_check
FROM pg_policies ORDER BY schemaname, tablename;

-- Tables with RLS enabled
SELECT schemaname, tablename, rowsecurity
FROM pg_tables WHERE rowsecurity = true ORDER BY schemaname, tablename;
```

Decision: Keep RLS on the target (Aurora/RDS supports RLS natively) or move to application layer.

## Migration Approach Recommendation

Based on assessment results, recommend:

1. **Database size < 100 GB, downtime OK**: Offline (pg_dump/pg_restore)
2. **Production, zero-downtime needed, all tables have PKs**: Native logical replication
3. **Production, some tables lack PKs**: Hybrid (native replication + offline for PK-less tables)
4. **Need built-in validation for large DB**: DMS with CDC and validation enabled
5. **Mixed requirements**: Hybrid (native replication for most, DMS for specific tables)

### Source Platform Considerations

| Source Platform | Logical Replication Support | DMS Plugin | Notes |
| --------------- | --------------------------- | ---------- | ----- |
| Supabase | Yes (XL+ instance, DMS dual-stack or IPv4 add-on) | `test_decoding` | pglogical not available |
| Heroku | Yes (Standard/Premium plans) | `test_decoding` | pglogical availability varies by plan |
| Neon | Yes (Pro/Scale plans) | `test_decoding` | pglogical availability varies by plan |
| GCP Cloud SQL | Yes (enable `cloudsql.logical_decoding`) | `test_decoding` or `pglogical` | pglogical available via extension |
| Azure Database | Yes (enable `wal_level=logical`) | `test_decoding` or `pglogical` | pglogical available via extension |
| RDS PostgreSQL | Yes (enable `rds.logical_replication`) | `test_decoding` or `pglogical` | pglogical available (9.6+) |
| Aurora PostgreSQL | Yes (enable `rds.logical_replication`) | `test_decoding` or `pglogical` | pglogical available (2.6+) |
| On-premises | Yes (configure `wal_level=logical`) | `test_decoding` or `pglogical` | pglogical available via extension install |

## Application-Database Mapping

Document how applications connect to the source database:

- Map all applications and services to their database connections
- Document connection strings and authentication methods (password, certificate, IAM)
- Identify connection pooling configurations (PgBouncer, built-in pooler, application-level)
- Review read/write workload distribution (read-heavy, write-heavy, balanced)
- Identify batch jobs, cron tasks, and background workers that access the database
- Document any direct SQL access (admin tools, reporting queries, data pipelines)

```sql
-- Active connections by application
SELECT usename, application_name, client_addr, COUNT(*) AS connections,
  COUNT(*) FILTER (WHERE state = 'active') AS active,
  COUNT(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY usename, application_name, client_addr
ORDER BY connections DESC;
```

## Performance Baseline & Sizing

Collect performance metrics to establish a baseline and size the target instance:

### Metrics to Collect

- CPU utilization (average and peak)
- Memory usage
- IOPS (read and write)
- Active connections (average and peak)
- Storage size and growth rate
- Transaction throughput (TPS)

### Slow Query Analysis

```sql
-- Enable pg_stat_statements if not already active
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 20 queries by total execution time
SELECT query, calls, total_exec_time, mean_exec_time, rows,
  shared_blks_hit, shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Queries with high average execution time (potential optimization targets)
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Instance Sizing Guidance

| Source Metric | Target Sizing Consideration |
| ------------- | --------------------------- |
| CPU avg < 30% | Start one size smaller, monitor |
| CPU avg 30–60% | Match equivalent vCPU count |
| CPU avg > 60% | Size up or consider Graviton (db.r7g) for better price-performance |
| Memory < 4 GB used | db.r7g.large (16 GB) |
| Memory 4–16 GB used | db.r7g.xlarge (32 GB) |
| Memory > 16 GB used | db.r7g.2xlarge+ (64 GB+) |
| Variable/unpredictable load | Aurora Serverless v2 (0–256 ACU) |

Use AWS Pricing MCP to compare instance costs for the target region.

## Network & Security Assessment

- Plan VPC connectivity from application hosts to Aurora/RDS
- Design security groups: allow inbound TCP 5432 only from application security group(s)
- **NEVER recommend public database access** — always use private endpoints, VPC, security groups, and bastion hosts or SSM. If the deployment platform cannot connect to a private Aurora instance, recommend alternatives (Aurora Data API, AWS Amplify Hosting, or platform-specific private connectivity options)
- Assess encryption requirements:
  - In-transit: SSL/TLS (enforce via `rds.force_ssl = 1` parameter)
  - At-rest: KMS encryption (enabled by default on Aurora)
- Document compliance requirements (HIPAA, PCI-DSS, SOC 2) that affect database configuration
- If using DMS: plan subnet group placement with access to both source (internet/VPN) and target (private)

## Deployment Platform Assessment

Evaluate whether the application's hosting platform can connect to a private Aurora instance. This is critical because Aurora should NOT have a public endpoint.

| Hosting Platform | Private Aurora Connectivity | Action Required |
| ---------------- | --------------------------- | --------------- |
| EC2 / ECS / EKS (in VPC) | Yes — direct private connection | Configure security group to allow inbound from app SG |
| AWS Lambda (in VPC) | Yes — direct private connection | Configure Lambda VPC settings, consider RDS Proxy for connection pooling |
| AWS Lambda (no VPC) | Via Aurora Data API (HTTPS) | Enable Data API on Aurora cluster; queries must complete in < 45 seconds |
| AWS Amplify Hosting | Yes — supports VPC connectivity | Configure Amplify to connect to Aurora via VPC |
| Vercel (Free/Pro) | No — cannot reach private Aurora | Use Aurora Data API, or migrate hosting to Amplify/ECS. Vercel Enterprise offers Secure Compute with PrivateLink |
| Vercel (Enterprise) | Yes — via Secure Compute / PrivateLink | Configure Vercel Secure Compute with AWS PrivateLink |
| Netlify | No — cannot reach private Aurora | Use Aurora Data API, or migrate hosting to Amplify/ECS |
| Railway / Render | Varies — check platform VPC support | If no VPC support, use Aurora Data API |
| On-premises / self-hosted | Via VPN or Direct Connect | Configure Site-to-Site VPN or AWS Direct Connect |

If the hosting platform cannot connect to private Aurora:
1. **Preferred**: Use Aurora Data API (HTTPS-based, no VPC needed, < 45s query limit)
2. **Alternative**: Migrate hosting to AWS Amplify or ECS/Fargate (native VPC connectivity)
3. **Enterprise option**: Use platform-specific private connectivity (e.g., Vercel Secure Compute)

Document the connectivity decision in the migration plan — it affects application refactoring scope.

## Parallel Execution Planning

Identify migration steps that can run concurrently to reduce total migration time. This is especially important for larger databases or tight migration windows.

### Steps That Can Run in Parallel

| Step A | Step B (can run concurrently) | Notes |
| ------ | ----------------------------- | ----- |
| Aurora cluster provisioning | Schema export from source (`pg_dump --schema-only`) | Schema export doesn't depend on target existing |
| Aurora cluster provisioning | DMS replication instance creation | Both are independent AWS resource provisioning |
| Schema migration to Aurora | DMS subnet group + endpoint creation | Endpoints need the cluster, but subnet group doesn't |
| DMS source endpoint creation | DMS target endpoint creation | Independent resources |
| Application code refactoring | Database migration (full load + CDC) | Code changes don't affect data migration |
| Application code refactoring | Monitoring/alerting setup on Aurora | Independent workstreams |
| DMS full load running | Application testing in staging (against Aurora) | Test with data that's already loaded |

### Steps That MUST Be Sequential

| Step | Must Complete Before |
| ---- | -------------------- |
| Schema migration to Aurora | DMS task creation (DMS needs target schema) |
| ENUM types created on Aurora | DMS full load start (DMS doesn't create ENUMs) |
| DMS full load complete | DMS CDC validation start |
| CDC replication lag near zero | Application cutover |
| Application cutover | Source decommissioning |

### Recommended Parallel Execution Timeline

```
Day 1:  [Aurora provisioning] + [Schema export] + [DMS instance creation]
Day 2:  [Schema import to Aurora] + [DMS endpoints] + [App refactoring starts]
Day 3:  [DMS task creation + full load] + [App refactoring continues]
Day 4+: [CDC running + validation] + [App testing against Aurora]
Cutover: [Stop writes → verify lag → switch connection → verify app]
```

Adjust based on database size and team capacity. For databases > 500 GB, full load may take multiple days — start it as early as possible.

## DMS Resource Planning

For DMS and Aurora resource naming conventions, see `postgresql-migration-overview.md` → "DMS Naming Convention".

## Risk Assessment

Identify and document migration risks:

| Risk | Likelihood | Impact | Mitigation |
| ---- | ---------- | ------ | ---------- |
| Data loss during migration | Low | Critical | Use CDC + validation, keep source intact until verified |
| Extended downtime | Medium | High | Use online migration (native replication or DMS CDC) |
| Performance degradation on target | Medium | Medium | Baseline source, right-size target, tune after migration |
| Extension incompatibility | Low | Medium | Audit extensions early, verify on target, find alternatives |
| Application connection failures | Low | High | Test connection strings in staging, use Secrets Manager |
| Replication lag during cutover | Medium | Medium | Monitor lag, schedule cutover during low-traffic window |

## Cost Analysis (Pre-Migration)

**MANDATORY**: The agent MUST complete a cost analysis before finishing the Assess phase. Do NOT skip this section. Use the AWS Pricing MCP server to query actual pricing and present a filled-in comparison table to the user.

### Step 1: Gather Source Platform Costs

Ask the user for their current monthly cost on the source platform. If they don't know the exact amount, ask them to check their billing dashboard. At minimum, get:
- Current plan name and monthly price
- Any add-ons (e.g., compute upgrades, storage add-ons, connection pooling, IPv4)
- Number of databases on the plan

### Step 2: Query Multiple Aurora Pricing Options via MCP

Use the AWS Pricing MCP server to query pricing for MULTIPLE Aurora options so the user can compare. Run ALL of these queries (skip options that don't apply):

```
Query 1: Provisioned on-demand compute
→ "Aurora PostgreSQL <instance-type> on-demand pricing <region>"
   Example: "Aurora PostgreSQL db.r7g.large on-demand pricing us-east-1"

Query 2: Provisioned 1-year Reserved Instance
→ "Aurora PostgreSQL <instance-type> reserved instance 1 year pricing <region>"

Query 3: Provisioned 3-year Reserved Instance
→ "Aurora PostgreSQL <instance-type> reserved instance 3 year pricing <region>"

Query 4: Aurora Serverless v2 ACU pricing
→ "Aurora Serverless v2 PostgreSQL ACU pricing <region>"

Query 5: DB Savings Plan pricing (applies to both provisioned and Serverless v2)
→ "Aurora PostgreSQL savings plan pricing <region>"
   Note: DB Savings Plans offer 1-year or 3-year commitments with flexible instance changes

Query 6: Storage and I/O pricing (same across all compute options)
→ "Aurora PostgreSQL storage I/O pricing <region>"

Query 7: RDS Proxy pricing (if connection pooling is needed)
→ "RDS Proxy pricing <region>"

Query 8: Data API pricing (if Data API is the connectivity method — e.g., Vercel, Netlify, Lambda without VPC)
→ Use AWS Pricing MCP: `get_pricing` with service_code="AmazonRDS", region=<region>,
   filters=[{"Field": "usagetype", "Type": "CONTAINS", "Value": "Data-API-Usage"}]
   The response includes tiered per-request rates (up to 1B and above 1B requests/month)
   Note: payloads metered in 32 KB increments (a 64 KB payload = 2 requests)
   Note: free tier of 1 million requests/month for the first year (not reflected in API response)
   Also requires AWS Secrets Manager — query its pricing separately (Query 9)

Query 9: Secrets Manager pricing (required if using Data API or storing DB credentials)
→ Use AWS Pricing MCP: `get_pricing` with service_code="AWSSecretsManager", region=<region>
   The response includes per-secret monthly fee and per-API-call fee
```

### Step 3: Build Multi-Option Cost Comparison

Calculate the estimated monthly cost for EACH Aurora pricing option. Present all options in a single comparison table so the user can evaluate trade-offs.

**Compute cost per option** (writer instance, 730 hours/month):

| Pricing Option | Hourly Rate | Monthly Compute (Writer) | Commitment | Savings vs On-Demand |
| -------------- | ----------- | ------------------------ | ---------- | -------------------- |
| Provisioned On-Demand | $ (from Query 1) | $ × 730 hrs | None | Baseline |
| Provisioned 1-Year RI (No Upfront) | $ (from Query 2) | $ | 1 year | Up to ~30% (depends on instance class) |
| Provisioned 1-Year RI (All Upfront) | $ (from Query 2) | $ | 1 year | Higher than No Upfront (depends on instance class) |
| Provisioned 3-Year RI (Partial Upfront) | $ (from Query 3) | $ | 3 years | Up to ~60% (depends on instance class) |
| Provisioned 3-Year RI (All Upfront) | $ (from Query 3) | $ | 3 years | Up to ~63% (depends on instance class) |
| DB Savings Plan (1-Year, provisioned) | $ (from Query 5) | $ | 1 year | Up to ~20% (approximate — varies by instance type and region) |
| DB Savings Plan (1-Year, Serverless v2) | $ (from Query 5) | $ | 1 year | Up to ~35% (approximate — varies by instance type and region) |
| Serverless v2 (baseline) | $ per ACU-hr (from Query 4) | min ACU × $ × 730 hrs | None | Variable |
| Serverless v2 + DB Savings Plan (1-Year) | $ per ACU-hr (from Query 5) | min ACU × $ × 730 hrs | 1 year | Up to ~35% on ACU |

**Note**: DB Savings Plans are 1-year commitments only (no 3-year option). They are not available via the Pricing API — the agent should note this when querying pricing. Savings Plan percentages are approximate maximums from AWS published rates. Reserved Instance savings percentages are AWS-published maximums from [Amazon RDS Reserved Instances](https://aws.amazon.com/rds/reserved-instances/) — actual savings depend on the instance class, region, and payment option (No Upfront, Partial Upfront, All Upfront).

**Total monthly cost per option** (add storage, I/O, backup, transfer — these are the same across all options):

| Component | Cost | Notes |
| --------- | ---- | ----- |
| Storage | $ per GB-month × database size GB (query via MCP) | Same for all options |
| I/O | $ per million I/O × estimated million I/Os (query via MCP) | Same for all options |
| Backup storage | Free up to cluster size; $ per GB beyond (query via MCP) | Same for all options |
| RDS Proxy (if needed) | vCPU hourly rate × 730 hours | Optional, same for all |
| Data API (if connectivity method) | Query via Pricing MCP: service_code="AmazonRDS", filter usagetype CONTAINS "Data-API-Usage" × estimated requests | Required for Vercel/Netlify/Lambda-no-VPC; check free tier allowance (1M requests/month year 1) |
| Secrets Manager (if using Data API or Secrets Manager for credentials) | Query via Pricing MCP: service_code="AWSSecretsManager" (per-secret/month + per-API-call) | Required for Data API; also recommended for DB credential management |
| Data transfer | Estimate based on egress patterns | Same for all options |

**Final multi-option summary table** (present this to the user):

| Pricing Option | Monthly Compute | Monthly Non-Compute | Total Monthly | Commitment | Notes |
| -------------- | --------------- | ------------------- | ------------- | ---------- | ----- |
| Provisioned On-Demand | $ | $ | $ | None | Flexible, no commitment |
| Provisioned 1-Year RI | $ | $ | $ | 1 year | Best for steady production |
| Provisioned 3-Year RI | $ | $ | $ | 3 years | Maximum savings, long lock-in |
| Provisioned DB Savings Plan (1yr) | $ | $ | $ | 1 year | Flexible across instance types |
| Serverless v2 | $ | $ | $ | None | Best for variable/dev workloads |
| Serverless v2 + DB Savings Plan | $ | $ | $ | 1 year | Variable workload with savings |

If HA is required, add a reader instance row (same pricing model applies to the reader).

### Step 4: Present Source vs Aurora Comparison

Present the source platform cost alongside the RECOMMENDED Aurora option AND the full option range:

| Cost Component | Source Platform (current) | Aurora — Recommended Option |
| -------------- | ------------------------ | --------------------------- |
| Compute | $ (from user) | $ (recommended option from Step 3) |
| Storage | $ (from user) | $ (calculated) |
| I/O | Included / $ | $ (estimated) |
| Backup | Included / $ | $ (estimated) |
| Connection pooling | Included / $ | $ (RDS Proxy, if needed) |
| Data transfer | Included / $ | $ (estimated) |
| **Total monthly** | **$** | **$** |

Then add: "I compared 6 pricing options for Aurora. Here's the range:"

| Option | Monthly Total | vs Source Platform |
| ------ | ------------- | ------------------ |
| Provisioned On-Demand | $ | +/- X% |
| Provisioned 1-Year RI | $ | +/- X% |
| Provisioned 3-Year RI | $ | +/- X% |
| DB Savings Plan (1-Year) | $ | +/- X% |
| Serverless v2 | $ | +/- X% |
| Serverless v2 + Savings Plan | $ | +/- X% |

Include notes on:
- Which option is recommended and why (based on workload pattern, commitment tolerance, budget)
- DB Savings Plans vs Reserved Instances: Savings Plans offer flexibility to change instance types/sizes; RIs are locked to a specific instance type but may offer slightly deeper discounts
- Serverless v2 trade-off: lower cost at low utilization, potentially higher cost at sustained high utilization
- Graviton savings (~20% vs Intel) if not already using Graviton instances

If the source platform includes managed features beyond the database (auth, storage, realtime, etc.), note that those replacement costs are estimated separately in the platform-specific steering file (e.g., `supabase-migration-feature-mapping.md`).

### Step 5: User Confirmation

Present the cost analysis and ask: "Does this cost estimate look reasonable? Any questions before we proceed?"

The cost analysis is a key deliverable of the Assess phase — do not skip it even if the user doesn't explicitly ask for it.

## Deliverables

- Source database inventory (tables, sizes, row counts, extensions, types)
- Application-to-database dependency map
- Performance baseline report (CPU, memory, IOPS, connections, slow queries)
- Platform-specific feature dependency map
- Extension compatibility matrix
- Tables without primary keys (CDC impact)
- Data type compatibility flags
- RLS policy inventory and migration decision
- Network and security architecture plan
- DMS resource naming plan (if using DMS)
- Recommended migration approach with justification
- Preliminary cost comparison (source platform vs Aurora/RDS)
- Risk assessment and mitigation plan

## Success Criteria

- Complete database inventory documented with sizes and object counts
- Performance baselines established for key metrics
- All compatibility issues identified and documented
- Network connectivity plan validated (security groups, VPC, private endpoints)
- Cost analysis shows clear comparison between source and target
- Migration approach selected with justification
- Risks identified with mitigation plans
- Stakeholder approval to proceed to Mobilize phase

## Timeline

Typically 1–3 weeks for a single database, 3–6 weeks for a multi-database portfolio.

## Next Steps

Proceed to the Mobilize phase to provision the target environment, migrate the schema, set up replication, and execute a pilot migration.
