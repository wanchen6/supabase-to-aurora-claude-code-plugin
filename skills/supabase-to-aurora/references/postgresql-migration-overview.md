# Migration Overview — PostgreSQL to AWS RDS/Aurora PostgreSQL

## Purpose

Central index for migrating a PostgreSQL database from a third-party cloud platform (Supabase, Heroku, Neon, GCP Cloud SQL, Azure Database for PostgreSQL) or on-premises to Amazon Aurora PostgreSQL or Amazon RDS for PostgreSQL. Links to each phase, summarizes the migration lifecycle, and provides quick-reference tables for naming conventions, AWS services, and security practices.

## Migration Phases

1. **[Assess Phase](01-assess-phase.md)** — Database discovery, compatibility assessment, platform-specific feature audit, and cost analysis
2. **[Mobilize Phase](02-mobilize-phase.md)** — Target provisioning, network assessment, schema migration decisions, DMS infrastructure (instance, endpoints), native replication configuration, pilot migrations. **Deliverable**: all infrastructure ready, connections tested, approach validated.
3. **[Migrate Phase](03-migrate-phase.md)** — DMS task execution, full load + CDC monitoring, application code refactoring, cutover, rollback, validation, post-migration activities. **Deliverable**: data migrated, app switched to Aurora, source decommissioned.
4. **[Modernize Phase](04-modernize-phase.md)** — Aurora optimization, HA/DR, cost optimization, and Graviton/Serverless adoption

## Supporting References — PostgreSQL Target

| File | Purpose |
| ---- | ------- |
| [database-operations-reference.md](database-operations-reference.md) | General-purpose PostgreSQL query library: schema exploration, index analysis, session monitoring, maintenance operations |
| [aurora-pg-best-practices.md](aurora-pg-best-practices.md) | Post-migration Aurora best practices: schema design, indexing, monitoring, connection management |

## Supporting References — Supabase Source

| File | Purpose |
| ---- | ------- |
| [supabase-migration-strategy-reference.md](supabase-migration-strategy-reference.md) | Supabase migration: approach decision tree, validation strategy, flow summaries, cutover checklist |
| [supabase-prerequisites-and-connectivity.md](supabase-prerequisites-and-connectivity.md) | Supabase migration: source config, Aurora target, EC2 bastion, MCP setup |
| [supabase-migration-feature-mapping.md](supabase-migration-feature-mapping.md) | Supabase migration feature mapping: Auth→Cognito, Storage→S3, Realtime→AppSync, Edge Functions→Lambda, schema exclusions, MCP usage |

## Source Platform Comparison

| Platform | Logical Replication | DMS Plugin | Platform-Specific Schemas | Notes |
| -------- | ------------------- | ---------- | ------------------------- | ----- |
| Supabase | Supported (XL+ plan, DMS dual-stack or IPv4 add-on) | `test_decoding` | `auth`, `storage`, `realtime`, `extensions`, `supabase_functions` | Requires CLI config for replication params |
| Heroku | Supported (Standard+) | `test_decoding` | None (all in `public`) | Connection via Heroku CLI credentials |
| Neon | Supported | `test_decoding` / `pglogical` | `neon` | Branching not available on Aurora |
| GCP Cloud SQL | Supported (flags required) | `test_decoding` / `pglogical` | None | Enable `cloudsql.logical_decoding` flag |
| Azure Database | Supported (config required) | `test_decoding` / `pglogical` | None | Set `wal_level = logical` via portal |
| On-premises | Supported (config required) | `test_decoding` / `pglogical` | Varies | Full control over `postgresql.conf` |

## Quick Reference

### Assess (3–6 weeks)

Discover source databases, audit extensions and platform-specific features, assess Aurora/RDS compatibility, establish performance baselines, and build the migration business case.

### Mobilize (6–10 weeks)

Provision Aurora cluster, configure source replication parameters, set up native logical replication and/or DMS, migrate schema, and execute 2–5 pilot table migrations.

### Migrate (8–24 weeks)

Execute wave-based migration using the chosen path (offline, native replication, DMS, or hybrid). Validate data integrity continuously. Cut over application connections.

### Modernize (Ongoing)

Optimize with Aurora features (Performance Insights, Graviton instances, Serverless v2), implement HA/DR, tune autovacuum, replace platform-specific features with AWS-native equivalents, and reduce costs.

## Migration Approaches

| Approach | Downtime | Best For | Details |
| -------- | -------- | -------- | ------- |
| pg_dump/pg_restore | Yes | < 100 GB or non-production | Simple, reliable, works for any schema |
| Native logical replication | Near-zero | Production, all tables have PKs | Preferred online path, built-in PostgreSQL feature |
| AWS DMS with CDC | Near-zero | Tables needing validation or where native replication fails | Fallback online path, built-in row-by-row validation |
| Hybrid | Brief window | Mixed requirements | Native replication + DMS + offline for PK-less tables |

See [supabase-migration-strategy-reference.md](supabase-migration-strategy-reference.md) for the full decision tree and flow summaries.

## MCP Servers

For the full MCP server list, configuration, and authentication details, see `POWER.md` → "MCP Servers".

## Aurora PostgreSQL MCP — Connection Methods

Choose the connection method based on where the MCP server is running:

| Method | When to Use | Prompt Example |
| ------ | ----------- | -------------- |
| `rdsapi` (recommended for local MCP) | MCP server runs outside VPC (user's local machine, Kiro IDE) | "Connect to database named X in Aurora PostgreSQL cluster 'Y' with database_type as APG, using rdsapi as connection method in Z region" |
| `pgwire` | MCP server runs inside VPC (EC2, Cloud9, CloudShell) | "Connect to database named X with database endpoint as Y with database_type as APG, using pgwire as connection method in Z region" |
| `pgwire_iam` | Same as pgwire + IAM authentication enabled on cluster | Same as pgwire but uses IAM token instead of password |

**`rdsapi` limitations** (inherits Aurora Data API constraints):
- 45-second execution timeout per statement
- 1 MiB response size limit
- For operations exceeding these limits (e.g., `CREATE SUBSCRIPTION` with `copy_data = true`, large schema migrations), use CloudShell or EC2 bastion + SSM instead

For MCP server configuration (JSON, env vars, OAuth setup), see `supabase-prerequisites-and-connectivity.md` → "MCP Server Setup".

## MCP Server Known Limitations

| Blocked Pattern | Workaround |
| --------------- | ---------- |
| `CREATE FUNCTION` with `$$` or single-quote body delimiters | Write to a `.sql` file, run via `psql -f` |
| `DROP TABLE`, `DROP DATABASE`, `DROP SCHEMA` | Run via `psql` directly |
| Multi-statement queries separated by `;` | Use single statements per MCP call |

## AWS Services Used

- Amazon Aurora PostgreSQL — target database
- AWS Database Migration Service (DMS) — online migration with CDC and validation
- AWS Secrets Manager — credential storage and rotation
- Amazon CloudWatch — monitoring, alarms, DMS validation metrics
- AWS Systems Manager (SSM) — EC2 remote command execution
- Amazon RDS Proxy — connection pooling for high-concurrency or Lambda workloads

## DMS Naming Convention

All DMS resources follow a consistent naming pattern:

| Resource | Naming Pattern |
| -------- | -------------- |
| Secrets Manager (source) | `dms-source-<project-name>` |
| Secrets Manager (target) | `dms-target-<project-name>` |
| Subnet group | `dms-subnet-group-<project-name>` |
| Replication instance | `dms-instance-<project-name>` |
| Source endpoint | `dms-source-<project-name>` |
| Target endpoint | `dms-target-<project-name>` |
| Migration project | `dms-mp-<project-name>` |

## Aurora Naming Convention

| Resource | Naming Pattern |
| -------- | -------------- |
| Cluster identifier | `<project-name>-cluster` |
| Custom parameter group | `<project-name>-pg` |
| Custom cluster parameter group | `<project-name>-cluster-pg` |

## Security Best Practices

- Never expose Aurora instances to the public internet
- Always connect from application hosts via private subnets
- Use security groups to restrict access to specific source IPs/security groups
- Enable encryption at rest (KMS) and in transit (SSL/TLS)
- Implement IAM database authentication where supported
- Rotate credentials using AWS Secrets Manager
- Enable audit logging with the pgAudit extension
- Use least-privilege database roles — avoid using the master user for application connections

## Agent Behavioral Rules

See `agent-behavioral-rules.md` for mandatory security, user interaction, MCP usage, and phase execution guardrails.
