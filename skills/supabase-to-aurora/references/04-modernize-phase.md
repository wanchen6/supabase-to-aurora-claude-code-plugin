# Modernize Phase — PostgreSQL Migration

## Overview

The Modernize phase optimizes the Aurora PostgreSQL (or RDS PostgreSQL) environment after migration, configures high availability and disaster recovery, implements cost optimization strategies, and sets up production-grade monitoring. Platform-specific feature replacements (e.g., managed auth, storage, realtime) are covered in separate platform-specific steering files.

## Migration Context

- Source: PostgreSQL databases hosted on 3rd-party platforms (Supabase, Heroku, Neon, GCP Cloud SQL, Azure Database, on-premises, etc.)
- Target: AWS RDS PostgreSQL or Amazon Aurora PostgreSQL
- Approach: Database-focused migration with application connection update or refactoring as needed
- Focus: Optimize the target environment, implement HA/DR, reduce costs, and replace platform-specific features

## Outline

1. [Modernize Decision Tree](#modernize-decision-tree)
2. [Aurora PostgreSQL Optimization](#aurora-postgresql-optimization)
3. [High Availability and Disaster Recovery](#high-availability-and-disaster-recovery)
4. [Cost Optimization](#cost-optimization)
5. [Security Enhancements](#security-enhancements)
6. [Operational Excellence](#operational-excellence)
7. [Platform-Specific Feature Replacement](#platform-specific-feature-replacement)
8. [Monitoring Setup](#monitoring-setup)
9. [Post-Migration Checklist](#post-migration-checklist)
10. [Common Modernization Paths](#common-modernization-paths)

## Modernize Decision Tree

Use this decision tree to prioritize and sequence the modernization activities after migration cutover. Not all decisions apply to every migration — skip those that don't match your environment.

### Decision 1: Modernization Priority Path

```
What is the most urgent post-migration concern?
├── Query performance is degraded vs source → Start with Database Performance Path
│   └── ANALYZE VERBOSE → pg_stat_statements → slow query tuning → index optimization
├── HA/DR is not yet configured → Start with High Availability Path
│   └── Read replicas → failover testing → backup retention → (optional) Global Database
├── Cost needs to be justified or reduced → Start with Cost Optimization Path
│   └── Right-size instances → Graviton evaluation → Serverless v2 → Reserved Instances
└── All stable → Work through all paths in parallel (performance, HA, cost, security)
```

### Decision 2: Instance Right-Sizing

```
Has the workload been running on Aurora for 2+ weeks?
├── NO → Keep current instance size, monitor CloudWatch metrics, revisit after 2 weeks
└── YES → Evaluate CloudWatch metrics:
    ├── Average CPU < 40% and FreeableMemory > 2 GB?
    │   └── YES → Downsize one instance class (e.g., db.r7g.xlarge → db.r7g.large)
    ├── CPU spikes > 80% frequently but average is low?
    │   └── YES → Consider Aurora Serverless v2 (auto-scales with demand)
    ├── Steady CPU 40–70%?
    │   └── YES → Current size is appropriate, no change needed
    └── CPU consistently > 70%?
        └── YES → Upsize one instance class or add read replicas to offload reads
```

### Decision 3: Graviton Migration

```
Is the cluster running on Intel instances (db.r6i, db.r5)?
├── NO (already on Graviton db.r7g / db.r6g) → No action needed
└── YES
    ├── Is this a production cluster?
    │   ├── YES → Test on a non-production cluster first, then modify instance class
    │   └── NO → Modify instance class directly to db.r7g equivalent
    └── Expected savings: up to 20% better price-performance vs Graviton2 (R7g); R8g (Graviton4) also available
```

### Decision 4: Connection Pooling Strategy

```
How does the application connect to Aurora?
├── AWS Lambda or serverless functions → RDS Proxy (recommended, handles connection reuse)
├── High-concurrency application (> 100 concurrent connections) → RDS Proxy
├── Moderate concurrency with existing app-level pooling (PgBouncer, HikariCP, etc.)
│   └── App-level pooling is sufficient, RDS Proxy optional
└── Low concurrency (< 50 connections) → Direct connection is fine, no pooler needed
```

### Decision 5: Read Replica Configuration

```
Does the workload have significant read traffic?
├── NO (write-heavy or balanced) → One read replica for HA failover only
└── YES (read-heavy)
    ├── Read traffic is from the same region?
    │   └── YES → Add 1–2 read replicas in different AZs, use reader endpoint
    └── Read traffic is from multiple regions?
        └── YES → Consider Aurora Global Database (cross-region read replicas, < 1s lag)
```

### Decision 6: Backup and DR Strategy

```
What is the RPO/RTO requirement?
├── RPO < 1 minute, RTO < 1 minute → Aurora Global Database (cross-region, managed failover)
├── RPO < 5 minutes, RTO < 30 seconds → Aurora with read replica in different AZ (default HA)
├── RPO < 1 hour, RTO < 1 hour → Aurora with PITR (point-in-time recovery, enabled by default)
└── RPO < 24 hours, RTO < 4 hours → Daily snapshots + cross-region snapshot copy
    └── Set backup retention:
        ├── Non-production → 1–3 days
        ├── Production → 7–14 days
        └── Compliance/regulated → 35 days (maximum)
```

### Decision 7: Security Enhancements

```
Which security enhancements are needed?
├── Database authentication method?
│   ├── Application uses IAM roles (ECS, Lambda, EC2) → Enable IAM database authentication
│   ├── Application uses static credentials → Use Secrets Manager with automatic rotation
│   └── Both → IAM auth for AWS services, Secrets Manager for external clients
├── Audit logging required?
│   ├── YES → Enable pgAudit extension (add to shared_preload_libraries, requires reboot)
│   └── NO → Standard PostgreSQL logging is sufficient
├── Threat detection needed?
│   ├── YES → Enable GuardDuty RDS Protection
│   └── NO → Skip (can enable later)
└── Encryption?
    ├── At rest → Enabled by default (AWS-managed KMS); use customer-managed KMS for compliance
    └── In transit → Set rds.force_ssl = 1 in parameter group
```

### Decision 8: Platform Feature Replacement Priority

```
Did the source platform provide managed features beyond the database?
├── NO (database only) → Skip this decision
└── YES → Prioritize replacements by user impact:
    ├── Authentication (users can't log in without it)?
    │   └── HIGH priority → Amazon Cognito or Auth0 (implement before or during cutover)
    ├── File/object storage (app stores files via platform)?
    │   └── MEDIUM priority → Amazon S3 + CloudFront (can run in parallel with migration)
    ├── Realtime subscriptions (live updates in the UI)?
    │   └── MEDIUM priority → AWS AppSync or API Gateway WebSocket
    ├── Serverless functions (platform edge functions)?
    │   └── LOW priority → AWS Lambda (can migrate incrementally post-cutover)
    └── Auto-generated REST API?
        └── LOW priority → API Gateway + Lambda or direct DB queries
```

Refer to the platform-specific steering file for detailed replacement steps.

### Decision 9: Monitoring Depth

```
What level of monitoring is needed?
├── Basic (non-production) → CloudWatch default metrics + 1–2 key alarms (CPU, connections)
├── Standard (production) → CloudWatch alarms (see Monitoring Setup table) + Enhanced Monitoring
├── Advanced (business-critical production)
│   └── All of Standard, plus:
│       ├── Performance Insights enabled
│       ├── pg_stat_statements for query-level analysis
│       ├── Slow query logging (log_min_duration_statement via parameter group)
│       ├── pgAudit for audit trail
│       └── Custom CloudWatch dashboards for application-specific metrics
└── All environments:
    └── Enable EventBridge rules for RDS events (failover, maintenance, snapshot)
```

### Decision 10: Autovacuum Tuning

```
Are there high-churn tables (frequent INSERT/UPDATE/DELETE)?
├── NO → Default autovacuum settings are fine
└── YES
    ├── Tables with > 10% dead tuples?
    │   └── YES → Lower autovacuum_vacuum_scale_factor to 0.05 (from default 0.2)
    ├── Large tables (> 10 GB) with moderate churn?
    │   └── YES → Lower autovacuum_vacuum_threshold and increase autovacuum_work_mem
    └── Tables with frequent bulk loads?
        └── YES → Lower autovacuum_analyze_scale_factor to 0.05 for faster statistics refresh
```

### Modernize Decision Summary Template

After working through the decisions above, document the results:

| Decision | Choice | Notes |
| -------- | ------ | ----- |
| Priority path | Performance / HA / Cost / All parallel | |
| Instance right-sizing | Keep current / Downsize / Upsize / Serverless v2 | Target instance: |
| Graviton migration | Already Graviton / Migrate to db.r7g / N/A | |
| Connection pooling | RDS Proxy / App-level pooling / Direct connection | |
| Read replicas | HA only (1 replica) / Read offload (2+ replicas) / Global Database | |
| Backup and DR | Default HA / PITR / Cross-region snapshots / Global Database | Retention: days |
| Security enhancements | IAM auth / Secrets Manager / pgAudit / GuardDuty / force_ssl | |
| Platform feature replacements | Auth / Storage / Realtime / Functions / API / None | |
| Monitoring depth | Basic / Standard / Advanced | |
| Autovacuum tuning | Default / Custom per-table settings | Tables to tune: |

## Aurora PostgreSQL Optimization

### Gather Fresh Statistics

Immediately after migration, gather statistics so the query planner makes good decisions:

```sql
ANALYZE VERBOSE;
```

For large databases, analyze the most critical tables first:

```sql
-- Analyze tables by size (largest first)
SELECT 'ANALYZE VERBOSE ' || schemaname || '.' || relname || ';'
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || relname) DESC;
```

### Enable Performance Insights

Enable Performance Insights on the Aurora cluster for query-level performance monitoring:
- Top SQL by wait events, calls, and duration
- Database load analysis
- Counter metrics for connections, transactions, and I/O

### Enable pg_stat_statements

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Index Optimization

Review and optimize indexes post-migration:

```sql
-- Find missing indexes (high sequential scan tables)
SELECT schemaname, relname, seq_scan, seq_tup_read, idx_scan,
  seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_tup,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC LIMIT 20;

-- Find unused indexes (candidates for removal)
SELECT schemaname, relname AS tablename, indexrelname AS indexname, idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Connection Management

Configure connection pooling for Aurora:
- Use the Aurora writer endpoint for writes, reader endpoint for reads
- Set DNS TTL < 30 seconds for fast failover
- Consider RDS Proxy for high-concurrency workloads or Lambda
- Review application connection pool settings (max connections, idle timeout)

See [aurora-pg-best-practices.md](aurora-pg-best-practices.md) for detailed connection pooling guidance by framework.

### Autovacuum Tuning

For high-churn tables, tune autovacuum:

```sql
-- Identify tables with high dead tuple counts
SELECT schemaname, relname, n_dead_tup, n_live_tup,
  round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC LIMIT 20;

-- Tune autovacuum for high-churn tables
ALTER TABLE high_churn_table SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.05
);
```

### Slow Query Logging

Set `log_min_duration_statement` via the Aurora custom parameter group (not `ALTER SYSTEM`, which is not allowed on RDS/Aurora):

```bash
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate"
```

This logs all queries taking longer than 1 second. Verify with:

```sql
SHOW log_min_duration_statement;
```

## High Availability and Disaster Recovery

### Aurora HA Features

Aurora PostgreSQL provides built-in HA:
- Storage replication across 3 AZs (6 copies of data)
- Automatic failover to read replica (typically < 30 seconds)
- Continuous backup to S3

### Configure Read Replicas

Add at least one read replica in a different AZ for HA:
- Serves as failover target
- Can offload read traffic from the writer
- Use the Aurora reader endpoint for read queries

### Backup Configuration

- Set backup retention to 7+ days for production
- Point-in-time recovery (PITR) is enabled by default — configure backup retention window as needed
- Test restore procedures periodically

### Cross-Region Disaster Recovery

For critical workloads, consider:
- Aurora Global Database for cross-region replication (< 1 second lag)
- Cross-region read replicas for DR
- Automated failover with Route 53 health checks

## Cost Optimization

Use the AWS Pricing MCP server to analyze and optimize costs. Query actual pricing — do NOT estimate from memory.

### Right-Sizing Instances

Query the AWS Pricing MCP server for current pricing:

```
Tool: awslabs.aws-pricing-mcp-server
Query: "Aurora PostgreSQL db.r7g.large on-demand pricing <region>"
```

Evaluate current utilization:
- If average CPU < 40%, consider downsizing
- If CPU spikes are infrequent, consider Serverless v2
- Monitor `CPUUtilization`, `DatabaseConnections`, `FreeableMemory` in CloudWatch

### Graviton-Based Instances

Graviton instances offer better price-performance than Intel equivalents:
- R7g (Graviton3): up to 20% price-performance improvement over Graviton2 — db.r7g.large, db.r7g.xlarge, db.r7g.2xlarge, etc.
- R8g (Graviton4): up to 29% price-performance improvement over Graviton3 — available in select regions
- Same Aurora PostgreSQL compatibility, no application changes required

Query the pricing comparison:

```
Tool: awslabs.aws-pricing-mcp-server
Query: "Aurora PostgreSQL db.r6i.xlarge on-demand pricing <region>"
Then: "Aurora PostgreSQL db.r7g.xlarge on-demand pricing <region>"
Compare the two results to calculate savings percentage.
```

### Aurora Serverless v2

Consider Serverless v2 for variable workloads:
- Scales automatically from 0–256 ACUs
- Pay per ACU-hour (no idle instance cost at minimum)
- Best for: dev/test, variable traffic, new applications with unknown load patterns
- Not ideal for: steady high-throughput production (provisioned is cheaper)

```
Tool: awslabs.aws-pricing-mcp-server
Query: "Aurora Serverless v2 PostgreSQL ACU pricing <region>"
```

### Reserved Instances

For steady-state production workloads (savings depend on instance class and payment option):
- 1-Year RI: Up to ~30% savings (No Upfront) over on-demand
- 3-Year RI: Up to ~60% savings (Partial Upfront) or Up to ~63% savings (All Upfront) over on-demand
- Evaluate after 1–2 months of stable usage patterns
- See [Amazon RDS Reserved Instances](https://aws.amazon.com/rds/reserved-instances/) for current pricing

### Storage Cost Optimization

Aurora storage auto-scales and is billed per GB-month:
- No pre-provisioning needed
- Storage only grows (does not auto-shrink)
- To reclaim space: export, recreate cluster, reimport (rare, for major cleanups)
- Monitor `VolumeBytesUsed` in CloudWatch

### Cost Comparison: Source Platform vs Aurora (Multi-Option)

During the Assess phase, a multi-option cost comparison was generated covering: On-Demand, 1-Year RI, 3-Year RI, DB Savings Plan, Serverless v2, and Serverless v2 + Savings Plan. Refer to `01-assess-phase.md` → "Cost Analysis (Pre-Migration)" for the full comparison methodology and table templates.

After 2–4 weeks of actual Aurora usage, re-run the comparison with real data:

| Cost Component | Source Platform (actual) | Aurora (actual, current option) | Aurora (alternative options) |
| -------------- | ----------------------- | ------------------------------- | ---------------------------- |
| Compute | $ (from billing history) | $ (CloudWatch + billing) | Re-query pricing MCP |
| Storage | $ | $ (VolumeBytesUsed metric) | Same across options |
| I/O | $ | $ (VolumeReadIOPs + VolumeWriteIOPs) | Same across options |
| Backup | $ | $ (TotalBackupStorageBilled) | Same across options |
| Data Transfer | $ | $ (actual egress) | Same across options |
| Connection Pooling | $ | $ (RDS Proxy, if used) | Same across options |
| **Total Monthly** | **$** | **$** | **$ (range)** |

Use AWS Pricing MCP to refresh Aurora pricing for the specific region and instance type. Obtain source platform costs from the user or the platform's billing dashboard.

### Post-Migration Cost Re-evaluation

After 2–4 weeks of production traffic on Aurora, re-evaluate the pricing option selected during the Assess phase:

1. **Collect actual usage metrics** from CloudWatch:
   - Average and peak CPU utilization (`CPUUtilization`)
   - Average and peak ACU utilization (Serverless v2: `ServerlessDatabaseCapacity`)
   - Storage used (`VolumeBytesUsed`)
   - I/O operations (`VolumeReadIOPs`, `VolumeWriteIOPs`)
   - Connection count (`DatabaseConnections`)

2. **Re-run the multi-option pricing comparison** using actual metrics instead of estimates:

   ```
   Tool: awslabs.aws-pricing-mcp-server
   Queries (same as Assess phase — see 01-assess-phase.md Step 2):
   → On-demand, 1yr RI, 3yr RI, Serverless v2, DB Savings Plan pricing for <region>
   ```

3. **Compare actual cost vs Assess phase estimate**:
   - If actual cost is within 10% of estimate → original option is validated
   - If actual cost is > 20% higher → consider switching options (e.g., On-Demand → RI, or Provisioned → Serverless v2)
   - If actual cost is > 20% lower → consider downsizing or switching to a cheaper option

4. **Present updated recommendation** to the user with the refreshed comparison table and a clear recommendation on whether to stay with the current option or switch.

5. **Schedule quarterly re-evaluations** — workload patterns change over time. Set a reminder to re-run this analysis every 3 months.

## Security Enhancements

### IAM Database Authentication

Enable IAM authentication for Aurora to eliminate password management:

- Applications authenticate using IAM credentials instead of database passwords
- Combine with RDS Proxy for connection pooling with IAM auth
- Use AWS Knowledge MCP `search_documentation` for setup steps

### Encryption

- At-rest: Aurora encrypts storage by default with AWS-managed KMS keys. For compliance, use customer-managed KMS keys
- In-transit: Enforce SSL/TLS via the `rds.force_ssl = 1` parameter in the cluster parameter group
- Secrets Manager: rotate database credentials automatically with Secrets Manager rotation

### Database Activity Monitoring

```sql
-- Enable pgAudit for database activity logging
-- NOTE: pgaudit must first be added to shared_preload_libraries in the
-- cluster parameter group (requires a reboot) before CREATE EXTENSION will work.
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure audit logging (via parameter group)
-- pgaudit.log = 'ddl,role,write'  -- log DDL, role changes, and write operations
-- pgaudit.log_catalog = off       -- reduce noise from catalog queries
```

pgAudit logs are sent to CloudWatch Logs for centralized analysis and alerting.

### Additional Security Measures

- Configure VPC endpoints for AWS services (Secrets Manager, S3) to avoid internet traffic
- Enable AWS GuardDuty RDS Protection for threat detection on Aurora
- Review and restrict database roles — follow least-privilege principle
- Disable public accessibility on the Aurora cluster

## Operational Excellence

### Automated Failover Testing

Schedule periodic failover tests to validate HA configuration:

```bash
# Trigger a manual failover (Aurora)
aws rds failover-db-cluster --db-cluster-identifier <cluster-id> --region <aws-region>
```

Monitor failover time and application recovery. Target: typically < 30 seconds for Aurora (actual time depends on workload and cluster configuration).

### Automated Backup Validation

- Periodically restore from a snapshot to a test cluster and run validation queries
- Verify point-in-time recovery (PITR) works within the backup retention window
- Document restore procedures and expected recovery times (RTO)

### Self-Healing Automation

- Use CloudWatch Alarms + Lambda for automated responses:
  - High CPU → trigger investigation or scale-up notification
  - Connection count near max → alert and optionally restart idle connections
  - Replication lag on read replicas → alert for investigation
- Use EventBridge rules to capture RDS events (failover, maintenance, snapshot completion)

### Runbook Development

Create operational runbooks for:
- Routine maintenance (parameter changes, minor version upgrades)
- Failover and recovery procedures
- Performance troubleshooting (slow queries, high CPU, lock contention)
- Scaling procedures (vertical scaling, adding/removing read replicas)
- Incident response (connection storms, disk full, replication lag)

## Platform-Specific Feature Replacement

If the source platform provides managed features beyond the database (authentication, file storage, realtime subscriptions, serverless functions, auto-generated APIs), these need AWS-native replacements.

### Application Data Layer Refactoring

If the application uses a platform-specific client SDK (e.g., Supabase client SDK) for database access instead of a direct PostgreSQL connection, the data access layer must be refactored as part of the migration. This is typically handled during the cutover step (see `03-migrate-phase.md`, Step 7a), but any remaining cleanup or optimization of the refactored code can be done during the Modernize phase.

Post-cutover data layer improvements:
- Replace N+1 query patterns with JOINs or JSON aggregation
- Add connection pooling via RDS Proxy for production workloads
- Implement query result caching where appropriate
- Add database query logging and performance monitoring

For detailed SDK-to-SQL pattern mapping and refactoring guidance, refer to the platform-specific steering file (e.g., `supabase-migration-feature-mapping.md`, section "Application Code Refactoring").

Common replacement patterns:

For detailed replacement steps by feature (Auth→Cognito, Storage→S3, Realtime→AppSync, Edge Functions→Lambda), see `supabase-migration-feature-mapping.md`.

Detailed migration steps for platform-specific features are covered in the platform-specific steering files (e.g., `supabase-migration-feature-mapping.md`). Consult the relevant file for your source platform.

## Monitoring Setup

### CloudWatch Alarms

Configure these alarms for production:

| Metric | Threshold | Action |
|--------|-----------|--------|
| CPUUtilization | > 80% for 5 min | Scale up or investigate |
| DatabaseConnections | > 80% of max | Review connection pooling |
| FreeableMemory | < 500 MB | Scale up instance |
| ReadLatency | > 20 ms | Check queries, indexes |
| WriteLatency | > 20 ms | Check queries, locks |
| BufferCacheHitRatio | < 95% | Increase instance size |
| DiskQueueDepth | > 10 | I/O bottleneck |

### Enhanced Monitoring

Enable Enhanced Monitoring for OS-level metrics:
- CPU breakdown (user, system, wait)
- Memory usage details
- Disk I/O and throughput
- Network throughput

## Post-Migration Checklist

See `aurora-pg-best-practices.md` → "Post-Migration Checklist" for the complete list.

## Deliverables

- Aurora cluster optimized (statistics, indexes, autovacuum, connection pooling)
- High availability configured (read replicas, backup retention, failover tested)
- Security enhancements applied (IAM auth, encryption, pgAudit, GuardDuty)
- Cost analysis completed with source-vs-Aurora comparison
- Platform-specific feature replacements planned or implemented
- Monitoring and alerting configured (CloudWatch, Performance Insights)
- Operational runbooks created (maintenance, failover, troubleshooting, scaling)
- Post-migration checklist completed

## Common Modernization Paths

### Database Performance Path

1. Gather fresh statistics (`ANALYZE VERBOSE`)
2. Enable pg_stat_statements and Performance Insights
3. Identify and optimize top slow queries
4. Add missing indexes, remove unused indexes
5. Tune autovacuum for high-churn tables
6. Implement RDS Proxy for connection pooling
7. Add read replicas for read-heavy workloads

### High Availability Path

1. Ensure at least one read replica in a different AZ
2. Test manual failover and measure recovery time
3. Set backup retention to 7+ days
4. Verify point-in-time recovery (PITR) is enabled — it is on by default with configurable backup retention
5. For critical workloads: implement Aurora Global Database for cross-region DR
6. Configure Route 53 health checks for automated DNS failover

### Cost Optimization Path

1. Monitor actual usage for 2–4 weeks post-migration
2. Re-run the multi-option pricing comparison from the Assess phase using actual metrics (see "Post-Migration Cost Re-evaluation" above)
3. Right-size instances based on CloudWatch metrics (CPU, memory, connections)
4. Evaluate Graviton instances (db.r7g or db.r8g) for up to 20–29% price-performance improvement
5. Consider Aurora Serverless v2 for dev/test or variable workloads
6. Purchase Reserved Instances for steady-state production (Up to ~30–63% savings depending on term and payment option; see [Amazon RDS Reserved Instances](https://aws.amazon.com/rds/reserved-instances/))
7. Archive old data to S3 using Aurora S3 integration (if applicable)
8. Schedule quarterly cost re-evaluations

## Success Criteria

- Query performance meets or exceeds source baseline
- High availability validated (failover tested, recovery < 30 seconds)
- Monitoring and alerting operational with appropriate thresholds
- Security enhancements applied (encryption, IAM auth, audit logging)
- Cost analysis completed with clear comparison to source platform
- Operational runbooks created and reviewed
- Team trained on Aurora operations and troubleshooting
- Post-migration checklist fully completed

## Timeline

- Initial optimization: 2–4 weeks after migration cutover
- Ongoing optimization: continuous, with quarterly reviews
- Full modernization (HA, security, cost optimization): 4–12 weeks
