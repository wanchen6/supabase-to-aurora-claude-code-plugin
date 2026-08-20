# Aurora PostgreSQL Best Practices

Post-migration best practices for operating on Aurora PostgreSQL. Sourced from the `amazon-aurora-postgresql` Kiro power steering files (`aurora-postgres.md` and `aurora-postgres-mcp.md`).

## MCP Tool Usage Policy

- Default to read-only. Refuse writes unless user authorizes with "RUN IT".
- Dry-run first: request `EXPLAIN` plan before heavy queries (>5s or large scans).
- Bound queries: always include `LIMIT 50` on browsing, narrow with `WHERE` predicates.
- Error handling: surface error message, then propose a fixed query.
- Privacy: redact emails/phones; aggregate where possible. Never echo secrets or full PII.

## SQL Style Guide

- Use ANSI SQL compatible with Aurora PostgreSQL.
- Qualify tables as `schema.table`.
- Use CTEs for clarity; prefer window functions over correlated subqueries.
- Timestamps: treat as UTC unless column or prompt says otherwise.
- Avoid `SELECT *`; target only needed columns; push down filters; avoid functions on indexed columns in WHERE.

## Schema Design Rules

- Normalize to 3NF; denormalize only when proven necessary.
- Use precise types: `INT` over `BIGINT`, `VARCHAR(50)` over `VARCHAR(255)`.
- Apply `NOT NULL` where required; use `CHECK` for limited value sets.
- Use `TIMESTAMPTZ` for all timestamps.
- Include `created_at`/`updated_at` columns.
- Primary keys: `SERIAL`/`BIGSERIAL` default, UUID only when needed.
- Foreign keys: always define FKs, choose `ON DELETE` behavior, index all FK columns.

## Index Strategy

### Always Index
- Foreign keys (not auto-indexed in PostgreSQL)
- `WHERE`, `ORDER BY`, `GROUP BY`, `JOIN` columns

### Index Patterns
- Composite: order by selectivity (most selective first)
- Covering: use `INCLUDE` for SELECT columns
- Partial: for common filters (`WHERE status = 'active'`)
- Expression: for computed columns

### Never Index
- Low-cardinality columns alone (unless in composite)
- Every column (write overhead)
- Redundant indexes (`(a,b)` already covers `(a)`)
- Small tables (< 1000 rows)

### Finding Missing Indexes

See `database-operations-reference.md` → "Find Missing Indexes (High Sequential Scan Tables)".

### Unused Indexes

See `database-operations-reference.md` → "Find Unused Indexes (Candidates for Removal)".

## Query Development

### Always
- `WHERE` with indexed columns
- `LIMIT` for large result sets
- Batch large operations
- Specify column names explicitly
- Use `INSERT ... ON CONFLICT` for upserts
- Wrap multi-statement ops in transactions
- Use `RETURNING` for inserted/updated data

### Never
- Full table scans without `WHERE` on production
- `SELECT *` in application code
- Unbounded queries without `LIMIT`
- Deploy without `EXPLAIN ANALYZE`
- Long transactions (blocks autovacuum)

## Safe Schema Changes

### Low-Risk (Fast/Instant)
- Adding nullable columns
- Adding columns with defaults (PG 11+)
- Changing defaults (metadata only)
- Renaming tables/columns (metadata only)
- `CONCURRENTLY` index operations
- Dropping columns (PG 11+, metadata only)

### Non-Blocking Patterns
```sql
-- Add nullable column (instant)
ALTER TABLE users ADD COLUMN last_login TIMESTAMPTZ;

-- Add column with default (PG 11+, instant)
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- Add check constraint safely
ALTER TABLE users ADD CONSTRAINT check_age CHECK (age >= 18) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT check_age;

-- Create index without blocking writes
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Add foreign key safely
ALTER TABLE orders ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_customer;
```

### Changing Column Type (Shadow Column)
```sql
ALTER TABLE orders ADD COLUMN amount_new DECIMAL(12,2);
UPDATE orders SET amount_new = amount WHERE amount_new IS NULL LIMIT 10000;
-- Repeat in batches, deploy dual-write code, verify, then swap:
BEGIN;
ALTER TABLE orders DROP COLUMN amount;
ALTER TABLE orders RENAME COLUMN amount_new TO amount;
COMMIT;
```

## Connection Best Practices

When connecting to Aurora PostgreSQL during or after migration, choose the right connection method based on the operation. Not every tool is appropriate for every task.

### Connection Method Decision Tree

```
What are you trying to do?
│
├── Run ad-hoc queries for validation, data exploration, or troubleshooting?
│   ├── Simple queries (SELECT, EXPLAIN, ANALYZE) → Aurora PostgreSQL MCP server
│   │   Converts natural language to SQL. Good for quick checks.
│   │   Example: "Show me the top 10 largest tables" → MCP generates and runs the query
│   └── Complex queries or multi-statement scripts → psql via EC2 bastion (SSM)
│
├── Run schema migrations (CREATE TABLE, ALTER TABLE, CREATE INDEX)?
│   └── psql via EC2 bastion (SSM) — write SQL to a .sql file, run via psql -f
│       MCP injection filters block some DDL patterns (CREATE FUNCTION with $$, DROP TABLE)
│
├── Run large data loads (pg_restore, COPY, bulk INSERT)?
│   └── psql or pg_restore via EC2 bastion (SSM) — direct connection, no MCP overhead
│
├── Execute queries from application code (Lambda, ECS, EC2)?
│   ├── Serverless (Lambda) with queries < 45 seconds → Aurora Data API
│   │   No VPC configuration needed. Uses HTTPS. Supports transactions.
│   │   Requires: Aurora Serverless v2 or provisioned with Data API enabled
│   ├── Serverless (Lambda) in VPC → direct psql connection via pg driver
│   │   Requires: Lambda in VPC, security group access to Aurora
│   └── Long-running services (ECS, EC2) → direct psql connection via pg driver + connection pool
│
├── Connect from a platform that can't reach private Aurora (e.g., Vercel free/pro)?
│   ├── Aurora Data API (HTTPS, no VPC needed) — best for simple CRUD
│   ├── AWS Amplify Hosting — can connect to private Aurora via VPC
│   └── Vercel Enterprise with Secure Compute / PrivateLink — if staying on Vercel
│
└── Automate operations from CI/CD or scripts?
    └── AWS CLI + SSM send-command to EC2 bastion
        Write SQL/commands to a file, send via SSM, capture output
```

### Method Details

| Method | Best For | Latency | VPC Required | Max Query Time |
| ------ | -------- | ------- | ------------ | -------------- |
| Aurora PostgreSQL MCP | Ad-hoc queries, validation, exploration | Low | Yes (from MCP host) | No hard limit |
| psql via EC2 bastion | Schema migrations, large scripts, DDL | Low | Yes (EC2 in VPC) | No hard limit |
| SSM send-command | Remote execution from CI/CD or local | Medium (~2-5s overhead) | EC2 in VPC | 3600s default |
| Aurora Data API | Serverless apps, external platforms | Medium (~10-50ms overhead) | No | 45 seconds |
| Direct pg driver | Application runtime connections | Low | Yes | No hard limit |

### Aurora Data API — When to Use

Aurora Data API is an HTTPS-based interface that doesn't require VPC connectivity. It's ideal when:
- The application runs on a platform without VPC access (Vercel, Netlify, external services)
- Using AWS Lambda without VPC configuration
- Query execution time is under 45 seconds
- You want to avoid managing connection pools

Key considerations:
- Requires Aurora Serverless v2 or provisioned cluster with Data API enabled
- Uses `aws rds-data execute-statement` API — not standard PostgreSQL wire protocol
- UUID parameters need `typeHint: "UUID"` in `SqlParameter` or cast to `::text` in SQL (see `troubleshooting.md` for details)
- Supports transactions via `begin-transaction` / `commit-transaction` / `rollback-transaction`
- Maximum result set size: 1 MB
- Not suitable for streaming large result sets or long-running queries

### Aurora PostgreSQL MCP — When to Use

The Aurora PostgreSQL MCP server converts natural language to SQL queries. Use it for:
- Quick validation checks during migration ("Are all rows migrated?", "Show tables with row count mismatches")
- Data exploration ("What are the most common values in the status column?")
- Troubleshooting ("Show me the slowest queries in pg_stat_statements")
- Ad-hoc analysis that doesn't require scripting

Do NOT use MCP for:
- Schema migrations (DDL) — injection filters may block `CREATE FUNCTION`, `DROP TABLE`
- Bulk data operations — use psql or pg_restore directly
- Application runtime queries — use a pg driver with connection pooling

### Fallback: psql via SSM send-command

When you need to run SQL on Aurora from outside the VPC (e.g., from a local machine or CI/CD pipeline), use SSM to execute commands on an EC2 bastion:

```bash
# Write SQL to a file on the bastion, then execute
aws ssm send-command \
  --instance-ids <ec2-instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters commands='[
    "export PGPASSWORD=$(aws secretsmanager get-secret-value --secret-id <secret-id> --region <region> --query SecretString --output text | python3 -c \"import sys,json; print(json.load(sys.stdin)[\\\"password\\\"])\")",
    "psql -h <aurora-endpoint> -U postgres -d <database> -c \"SELECT version();\""
  ]' \
  --region <aws-region>
```

For large SQL scripts, upload the file to the bastion first (via S3 or SCP), then run `psql -f <script.sql>`.

## Connection Management

### Endpoint Routing

- Use the cluster (writer) endpoint for all write operations
- Use the reader endpoint for read-only queries to offload the writer
- Set DNS TTL < 30 seconds — Aurora failover changes the writer endpoint IP, and stale DNS causes connection failures

### Connection Pooling

Aurora's `max_connections` scales with instance size. A general formula:

```
max_connections ≈ LEAST({DBInstanceClassMemory/9531392}, 5000)
```

For common instance classes:
| Instance Class | Approx max_connections |
| -------------- | --------------------- |
| db.r7g.large (16 GB) | ~1,700 |
| db.r7g.xlarge (32 GB) | ~3,400 |
| db.r7g.2xlarge (64 GB) | ~5,000 |
| Serverless v2 (varies) | Scales with ACU |

Pool sizing guidance:
- Set application pool `max` to leave headroom — aim for 80% of `max_connections` across all app instances combined
- Example: 4 app instances × 20 pool connections = 80 total, well within a db.r7g.large limit
- Set `idle_timeout` to 30–60 seconds to reclaim unused connections
- Set `connection_timeout` to 2–5 seconds to fail fast rather than queue indefinitely
- Enable SSL on all pooled connections (`sslmode=require`)

### Connection Pooling by Framework

Adapt these patterns to the application's language and framework:

**Node.js (pg / node-postgres)**:
```javascript
const pool = new Pool({
  host: process.env.PGHOST,       // Aurora cluster endpoint
  max: 20,                         // Pool size per app instance
  idleTimeoutMillis: 30000,        // Close idle connections after 30s
  connectionTimeoutMillis: 3000,   // Fail if no connection in 3s
  ssl: { rejectUnauthorized: true }
});
```

**Python (psycopg2 / SQLAlchemy)**:
```python
engine = create_engine(
    DATABASE_URL,
    pool_size=20,            # Connections per app instance
    max_overflow=5,          # Burst above pool_size
    pool_timeout=3,          # Seconds to wait for a connection
    pool_recycle=1800,       # Recycle connections every 30 min
    connect_args={"sslmode": "require"}
)
```

**Java (HikariCP)**:
```yaml
maximumPoolSize: 20
minimumIdle: 5
idleTimeout: 30000        # 30 seconds
connectionTimeout: 3000   # 3 seconds
maxLifetime: 1800000      # 30 minutes
```

### RDS Proxy

Consider RDS Proxy when:
- Application runs on AWS Lambda (connection churn from cold starts)
- High number of application instances with small individual pools
- Need connection multiplexing to reduce Aurora connection count
- Want automatic failover handling (Proxy pins to the new writer)

RDS Proxy adds a small amount of latency per query (typically low single-digit milliseconds) but significantly reduces connection overhead on Aurora.

### Failover Resilience

- Test failover before production: trigger a manual failover and verify the application reconnects
- Implement connection retry logic with backoff (1–2 retries, 1–3 second delay)
- Watch for `Connection refused` or `Connection reset` errors after failover — these indicate stale DNS or missing retry logic
- Aurora failover typically completes in < 30 seconds (actual time depends on workload and cluster configuration); the application should tolerate brief unavailability

## Monitoring & Maintenance

### Statistics
```sql
VACUUM ANALYZE table_name;  -- After migrations or bulk loads
ANALYZE;                     -- Entire database
```

### Bloat Detection

See `database-operations-reference.md` → "Table Bloat Detection".

### Autovacuum Tuning (High-Churn Tables)
```sql
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

### CloudWatch Alerts to Configure
- CPUUtilization > 80%
- DatabaseConnections approaching max
- FreeableMemory low
- ReadLatency/WriteLatency spikes
- BufferCacheHitRatio < 95%

## Post-Migration Checklist

After migrating from Supabase to Aurora:
1. Run `ANALYZE VERBOSE;` to gather fresh statistics
2. Verify all indexes exist and are being used
3. Enable Performance Insights
4. Configure CloudWatch alarms
5. Enable slow query logging
6. Test failover behavior
7. Set backup retention to 7+ days for production
8. Enable encryption at rest (KMS)
9. Review and tune connection pool settings for Aurora
10. Consider enabling `pg_stat_statements` extension for query analysis
11. Read replicas configured for HA
12. Connection pooling configured and tested
13. Failover tested (manual failover and recovery)
14. Platform-specific feature replacements implemented (if applicable)
15. Cost analysis completed (source platform vs Aurora comparison)
16. Monitoring dashboards created
17. Runbook updated with Aurora operational procedures
