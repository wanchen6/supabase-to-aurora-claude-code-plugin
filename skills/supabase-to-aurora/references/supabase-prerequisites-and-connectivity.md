# Prerequisites and Connectivity — Supabase to Aurora PostgreSQL

## Purpose

Single source of truth for all connectivity prerequisites: Supabase source configuration, Aurora target access, EC2 bastion setup, MCP server setup, and known limitations. Complete these before starting any migration phase.

## Supabase Source Prerequisites

### Write Access Verification

Before configuring Supabase for migration, determine whether write access to the source is actually needed for the current phase:

- **Assessment phase**: Read-only access is sufficient. The Supabase MCP server in read-only mode can discover schemas, tables, extensions, RLS policies, and run all assessment queries. No write access needed.
- **Mobilize phase (schema migration)**: Read-only access to Supabase is sufficient — schema export uses `pg_dump` (read-only) and schema import targets Aurora. Write access to Supabase is only needed if creating publications/replication slots for online migration.
- **Migrate phase (online migration)**: Write access to Supabase is required to create publications and replication slots. The agent provides exact commands for the user to run manually.

**Default to read-only access** until a specific operation requires writes. This minimizes risk to the production source database.

### Instance Requirements

- **Instance size**: XL or greater recommended for logical replication performance (native or DMS). Note: logical replication is available on all Supabase plans including Free, but XL provides better performance for production migrations.
- **IPv4 add-on**: NOT required if VPC supports IPv6 — use DMS dual-stack mode instead (free). Only use IPv4 add-on if it is already enabled on the Supabase project. See "IPv6 and Dual-Stack Connectivity" below for details.
- **External connectivity decision tree**:
  1. Check if the user's VPC has IPv6 CIDR, subnets with IPv6 allocation, and `::/0` route to IGW
  2. If YES → use DMS dual-stack mode (`--network-type dual`) — no Supabase add-on needed
  3. If NO → check if Supabase IPv4 add-on is already enabled
  4. If IPv4 already enabled → use it (no VPC changes needed)
  5. If neither → enable IPv6 on the VPC (preferred) or enable IPv4 add-on on Supabase (simpler but costs extra)
- **PostgreSQL user**: Use the `postgres` user for all replication and export operations

### Replication Parameters (Supabase CLI)

Configure via the Supabase CLI before starting any online migration:

```bash
# Parameters that require a Supabase instance restart
supabase --experimental --project-ref <project-ref> postgres-config update \
  --config max_replication_slots=10

supabase --experimental --project-ref <project-ref> postgres-config update \
  --config max_wal_senders=10

# Parameters that do NOT require restart
supabase --experimental --project-ref <project-ref> postgres-config update \
  --config wal_sender_timeout=0

supabase --experimental --project-ref <project-ref> postgres-config update \
  --config max_slot_wal_keep_size=4GB

supabase --experimental --project-ref <project-ref> postgres-config update \
  --config max_wal_size=2GB
```

| Parameter | Requires Restart | Purpose |
| --------- | ---------------- | ------- |
| `max_replication_slots` | Yes | Number of replication slots available |
| `max_wal_senders` | Yes | Number of concurrent WAL sender processes |
| `wal_sender_timeout` | No | Timeout for idle replication connections (0 = disabled) |
| `max_slot_wal_keep_size` | No | Maximum WAL retained per slot before forced cleanup |
| `max_wal_size` | No | WAL size before checkpoint is triggered |

Verify configuration was applied:

```sql
SELECT name, setting, context, pending_restart
FROM pg_settings
WHERE name IN ('max_replication_slots', 'max_wal_senders', 'wal_sender_timeout',
               'max_slot_wal_keep_size', 'max_wal_size');
```

### Supabase Connection Details

Gather these before starting:

- **Host**: `db.<project-ref>.supabase.co`
- **Port**: `5432`
- **User**: `postgres`
- **Password**: Supabase database password (from project settings)
- **Database**: `postgres` (default Supabase database name)
- **SSL**: Required (`sslmode=require`)

## Aurora Target Prerequisites

### Security Group Configuration

The Aurora security group must allow inbound TCP 5432 from:

1. **Application host** (EC2, ECS, Lambda VPC) — for application connections
2. **CloudShell VPC environment** — for schema migration and ad-hoc psql operations (recommended over EC2 bastion)
3. **Supabase IP range** — for native logical replication (if using that path)
4. **DMS replication instance subnet** — for DMS CDC (if using that path)
5. **EC2 bastion** — for manual psql operations (alternative to CloudShell)

### CloudShell VPC Environment (Alternative for Manual Operations)

AWS CloudShell provides free, browser-based `psql` access to private Aurora. There are two ways to use it:

**Option A: Integrated CloudShell from RDS Console (simplest for manual operations)**
1. AWS Console → RDS → select your Aurora cluster → **Connectivity & security** tab
2. Click the **Connect** button — the console provides ready-made connection commands and integrated CloudShell access
3. CloudShell opens pre-configured with the correct endpoint and connection details
4. `psql` is pre-installed — run schema migration or ad-hoc commands directly

Reference: [Amazon RDS enhanced console experience for database connectivity](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-rds-provides-enhanced-console-experience/) (Feb 2026, available for Aurora PostgreSQL across all commercial regions)

**Option B: CloudShell VPC Environment (for custom VPC configurations)**
1. AWS Console → CloudShell → click "+" → **Create VPC environment**
2. Select Aurora's VPC, a **private subnet** (same as Aurora), and Aurora's **security group**
3. The CloudShell environment inherits VPC networking and can reach private Aurora directly

Reference: [Connect to Private Amazon RDS PostgreSQL Database Using AWS CloudShell](https://docs.aws.amazon.com/hands-on/latest/connect-to-private-amazon-rds-for-postgresql-from-aws-cloudshell/connect-to-private-amazon-rds-postgresql-database-using-aws-cloudshell.html)

**Note**: For schema migration execution methods (Data API, CloudShell, EC2 bastion), see `02-mobilize-phase.md` → "Execution methods for Method B".

```bash
# Allow EC2 bastion to connect to Aurora
aws ec2 authorize-security-group-ingress \
  --group-id <aurora-sg-id> \
  --protocol tcp --port 5432 \
  --source-group <ec2-sg-id> \
  --region <aws-region>
```

### Aurora Parameter Group

For online migration targets, configure:

- `rds.logical_replication = 1` (if Aurora will also act as a replication source later)
- `max_replication_slots = 10`
- `max_wal_senders = 10`
- `shared_preload_libraries = 'pg_stat_statements'`

## EC2 Bastion Setup

If using an EC2 instance as a bastion for psql operations, schema migration, or pg_dump/pg_restore:

### IAM Policy

Attach `SecretsManagerReadWrite` to the EC2 instance role so it can retrieve the Aurora master password:

```bash
aws iam attach-role-policy \
  --role-name <ec2-instance-role-name> \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite
```

### PostgreSQL Client

Install the PostgreSQL client tools on the EC2 instance:

```bash
# Amazon Linux 2023 / AL2
sudo yum install -y postgresql17

# Ubuntu/Debian
sudo apt-get install -y postgresql-client-17
```

### Connection Pattern

Retrieve the Aurora password from Secrets Manager and connect:

```bash
export PGPASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id "<rds-secret-id>" --region <aws-region> \
  --query SecretString --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

psql "host=<aurora-endpoint> port=5432 dbname=<database-name> user=postgres sslmode=require"
```

## DMS Connectivity Prerequisites

If using AWS DMS for migration:

1. **DMS replication instance** must be in a VPC subnet that can reach both Supabase (internet) and Aurora (private)
2. **Source endpoint** (Supabase): uses public internet — ensure the DMS subnet has a NAT gateway or internet gateway
3. **Target endpoint** (Aurora): uses private networking within the VPC
4. **Security group** for the DMS replication instance must allow outbound to Supabase on port 5432 and inbound/outbound to Aurora on port 5432
5. **SSL**: Always use `SslMode: require` on BOTH source and target endpoints
6. **Test connections** before creating migration tasks

### DMS Replication Instance — Network Configuration

**CRITICAL**: The DMS replication instance network settings depend on whether the source is external (internet) or internal (VPC). Getting this wrong causes connection timeouts that are hard to diagnose.

```
Is the source database external (outside AWS VPC)?
├── YES (Supabase, Heroku, Neon, GCP, Azure, on-premises via internet)
│   └── DMS replication instance MUST have outbound internet access:
│       ├── Option A (simplest): --publicly-accessible flag during creation
│       │   Instance gets a public IP and can route to external sources directly
│       └── Option B: Private instance in a subnet with NAT gateway
│           Instance routes outbound traffic through NAT (more complex, no public IP)
│       ★ IMPORTANT: --publicly-accessible CANNOT be changed after creation.
│         If you create without it, you must delete and recreate the instance.
└── NO (source is in the same VPC or peered VPC)
    └── --no-publicly-accessible (default, more secure)
```

**For Supabase migrations, always use `--publicly-accessible`** because Supabase is an external source. The DMS instance also needs `--network-type dual` for IPv6 connectivity to Supabase.

**Complete DMS instance creation command for Supabase migrations:**

```bash
aws dms create-replication-instance \
  --replication-instance-identifier <name> \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 20 \
  --no-multi-az \
  --publicly-accessible \
  --network-type dual \
  --replication-subnet-group-identifier <subnet-group> \
  --region <region>
```

### DMS Endpoint SSL Configuration

**Before creating any DMS endpoint, resolve the correct Aurora secret.** Do NOT scan Secrets Manager for a likely secret — get it directly from the cluster:

```bash
# Step 1: Get the cluster's own managed secret ARN and master username
aws rds describe-db-clusters \
  --db-cluster-identifier <cluster-id> \
  --query 'DBClusters[0].{User:MasterUsername,SecretArn:MasterUserSecret.SecretArn}'

# Step 2: Verify the secret content matches
aws secretsmanager get-secret-value --secret-id <SecretArn> \
  --query 'SecretString' --output text | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('user='+d['username'])"
```

Only proceed to create the DMS target endpoint after confirming the username matches `MasterUsername`. Using the wrong secret is a common source of `password authentication failed` errors during DMS endpoint testing.

**Secrets Manager flags**: Do NOT use `--secrets-manager-secret-id` / `--secrets-manager-access-role-arn` as top-level CLI flags — they are not supported that way. These must be passed inside `--postgre-sql-settings` as `SecretsManagerSecretId` and `SecretsManagerAccessRoleArn`.

**Always use SSL for both source and target endpoints:**

- Source endpoint (Supabase): `"SslMode": "require"` — Supabase requires SSL
- Target endpoint (Aurora): `"SslMode": "require"` — Aurora supports SSL and it should always be enforced

Include `SslMode` in both endpoint JSON configurations.

### DMS FK Handling — MANDATORY for Schemas with Foreign Keys

**CRITICAL**: If the target schema has foreign keys, you MUST configure FK bypass BEFORE starting the DMS task. Failure to do this causes `Table error` on child tables during full load because DMS loads tables in parallel and child rows may arrive before parent rows.

**Recommended approach: `session_replication_role = 'replica'` via target endpoint ECA (Extra Connection Attributes)**

Add this to the target endpoint's `PostgreSQLSettings` when creating the endpoint:

```json
{
  "PostgreSQLSettings": {
    "AfterConnectScript": "SET session_replication_role = 'replica';"
  }
}
```

This bypasses FK checks, triggers, and rules for the entire DMS session (both full load and CDC phases). Remove the `AfterConnectScript` only after migration cutover is complete and the application is writing directly to Aurora.

**Do NOT drop and recreate FKs** — this is fragile, especially during CDC where ongoing changes depend on FK relationships. The `session_replication_role` approach is simpler, safer, and recommended by the power's Decision 6 in `02-mobilize-phase.md`.

### IPv6 and Dual-Stack Connectivity (Preferred for Supabase)

Supabase database hosts (`db.<project-ref>.supabase.co`) resolve to IPv6 addresses by default. **DMS dual-stack mode is the preferred connectivity method** because it avoids the Supabase IPv4 add-on cost and works with Supabase's native IPv6 endpoints.

**Decision tree for Supabase external connectivity:**

```
Does the user's VPC have IPv6 enabled?
├── Check: aws ec2 describe-vpcs --query "Vpcs[0].Ipv6CidrBlockAssociationSet"
├── YES (IPv6 CIDR assigned) → Check subnets and routes
│   ├── Subnets have IPv6 CIDR? → Check: aws ec2 describe-subnets --query "Subnets[*].Ipv6CidrBlockAssociationSet"
│   ├── Route table has ::/0 → IGW? → Check: aws ec2 describe-route-tables --query "RouteTables[*].Routes[?DestinationIpv6CidrBlock=='::/0']"
│   ├── All YES → Use DMS dual-stack mode (--network-type dual) ✓ FREE
│   └── Missing pieces → Add IPv6 CIDR to subnets / add ::/0 route (simple VPC config)
└── NO (no IPv6 CIDR)
    ├── Is Supabase IPv4 add-on already enabled?
    │   ├── YES → Use IPv4 connectivity (no changes needed)
    │   └── NO → Prefer enabling IPv6 on VPC (free) over IPv4 add-on (paid)
    └── If VPC IPv6 setup is not feasible → Enable Supabase IPv4 add-on as last resort
```

**DMS replication instance — always create with dual-stack mode** when connecting to Supabase:

```bash
aws dms create-replication-instance \
  --replication-instance-identifier <name> \
  --replication-instance-class dms.r5.large \
  --network-type dual \
  --region <region>
```

The `--network-type dual` flag enables both IPv4 and IPv6 connectivity. For an existing replication instance, you can modify it to use Dual-stack mode:

```bash
aws dms modify-replication-instance \
  --replication-instance-arn <instance-arn> \
  --network-type dual \
  --region <region>
```

If the DMS instance cannot resolve the Supabase host, verify:
- The VPC has an IPv6 CIDR block assigned
- The subnet has IPv6 CIDR allocation
- The route table has routes for IPv6 traffic (`::/0` via internet gateway or egress-only internet gateway)

### Outbound IPv6 — Aurora Confirmed Limitation

**Aurora PostgreSQL CANNOT make outbound IPv6 connections** (via EIGW or IGW from private subnets).

**Evidence (tested May 2026):** VPC flow logs show Aurora sends TCP SYN packets but no SYN-ACK returns, while an EC2 instance in the same subnet/route table/SG connects successfully. Tested with both EIGW and IGW routes — neither works for Aurora outbound IPv6.

**What works and what doesn't (IPv6-only Supabase):**

| Method | Works? | Why |
|--------|--------|-----|
| `CREATE SUBSCRIPTION` from Aurora | ❌ | Aurora can't reach IPv6 endpoints |
| DMS (publicly-accessible, dual-stack) | ✅ | DMS has its own public IP |
| EC2 bastion with IPv6 for pg_dump | ✅ | EC2 outbound IPv6 works normally |

**Agent decision logic for IPv6-only sources:**
1. User wants near-zero downtime → recommend DMS (publicly-accessible, dual-stack, ~$3/week)
2. User accepts seconds of downtime → recommend pg_dump via bastion (cheapest, ~$0.01)
3. User already has IPv4 add-on → recommend native replication (simplest online option)

**Note for EC2 bastions in private subnets**: If the bastion needs to reach Supabase's IPv6 endpoint, it needs either a public IPv6 address (public subnet + IGW) or an Egress-Only Internet Gateway (EIGW) route (`::/0 → eigw-xxx`). EIGW is free — create with `aws ec2 create-egress-only-internet-gateway --vpc-id <vpc-id>` and add the route to the subnet's route table.

Reference: [Enable outbound IPv6 traffic using an egress-only internet gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)

**Aurora dual-stack mode** (for inbound IPv6, not outbound): Enable if needed for DMS target connectivity:

```bash
aws rds modify-db-cluster \
  --db-cluster-identifier <cluster-id> \
  --network-type DUAL \
  --region <region>
```

**Note**: You cannot use RDS Proxy with dual-stack mode DB clusters.

Test connections before creating migration tasks:

```bash
aws dms test-connection \
  --replication-instance-arn <replication-instance-arn> \
  --endpoint-arn <source-endpoint-arn> \
  --region <aws-region>

aws dms test-connection \
  --replication-instance-arn <replication-instance-arn> \
  --endpoint-arn <target-endpoint-arn> \
  --region <aws-region>
```

## MCP Server Setup

**Read-only access pattern**: The agent uses the Supabase MCP server in read-only mode for all discovery, assessment, and verification operations. For any write operations on Supabase (creating publications, replication slots, schema changes, CLI config updates), the agent provides the exact command and asks the user to execute it manually. This ensures the agent never modifies the production source database without explicit user action.

### Supabase MCP Server

The Supabase MCP server connects to your Supabase project for database discovery and assessment. It uses **OAuth authentication** — when you first connect, it opens a browser for Supabase login and project selection. No hardcoded project reference is needed.

**Configuration** (in `mcp.json`):

```json
{
  "supabase": {
    "type": "http",
    "url": "https://mcp.supabase.com/mcp",
    "disabled": false,
    "autoApprove": []
  }
}
```

On first use, the MCP server will prompt you to authenticate via your browser and select which Supabase project to connect to. To restrict to read-only mode or scope to a specific project without OAuth prompts, you can append query parameters: `https://mcp.supabase.com/mcp?read_only=true&project_ref=<project-ref>`

#### Auto-Fix: Wrong Project Connected

When the Supabase MCP returns data from a different project than the user's target (common with multiple orgs), the agent fixes this automatically:

1. Extract the correct `project_ref` from the user's Supabase URL (e.g., `iubiqvnnusijoogaauqf` from `https://iubiqvnnusijoogaauqf.supabase.co`)
2. Edit `~/.kiro/settings/mcp.json` — find the power's Supabase MCP server entry (under the `powers` block) and update the URL to `https://mcp.supabase.com/mcp?project_ref=<project-ref>`. Do NOT create a new entry in workspace `.kiro/settings/mcp.json` — that creates a duplicate server that lacks the OAuth token and causes permission errors.
3. Wait for the MCP server to reconnect (Kiro reconnects automatically on config save)
4. Retry `list_tables` to verify it now returns the correct project
5. If it returns an auth/permission error: the server needs to authenticate with this project for the first time. Tell the user: "Please complete the OAuth flow when prompted in your browser to grant access to this project."

**Available tools (read-only when `read_only=true` is set)**:

| Tool | Purpose |
| ---- | ------- |
| `list_tables` | List all tables within specified schemas |
| `list_extensions` | List all installed extensions |
| `list_migrations` | List all migrations in the database |
| `execute_sql` | Execute read-only SQL queries (SELECT, EXPLAIN, etc.) |
| `search_docs` | Search Supabase documentation |
| `get_logs` | Get logs by service type (postgres, api, auth, etc.) |
| `get_advisors` | Get advisory notices (security, performance) |
| `generate_typescript_types` | Generate TypeScript types from database schema |

**Mutating tools** (disabled when `read_only=true` is set):

- `apply_migration` — DDL/schema changes
- `create_project`, `pause_project`, `restore_project`
- `deploy_edge_function`
- `create_branch`, `delete_branch`, `merge_branch`, `reset_branch`, `rebase_branch`
- `update_storage_config`

**Feature groups**: By default, all feature groups are enabled via OAuth. To restrict, append `&features=database,docs,debugging` to the URL. Available groups: `account`, `docs`, `database`, `debugging`, `development`, `functions`, `storage`, `branching`.

### Operations Requiring Manual User Action

Because the Supabase MCP server runs in read-only mode, the following operations cannot be performed through MCP and must be done manually by the user. The agent will provide the exact commands and ask the user to execute them.

**Supabase CLI operations** (run from user's terminal):

| Operation | Command | When Needed |
| --------- | ------- | ----------- |
| Configure replication parameters | `supabase --experimental --project-ref <ref> postgres-config update --config <param>=<value>` | Before online migration |
| Restart instance (after parameter changes) | Via Supabase Dashboard → Settings → Infrastructure | After changing `max_replication_slots` or `max_wal_senders` |

**SQL operations requiring write access** (run via Supabase SQL Editor or psql):

| Operation | SQL | When Needed |
| --------- | --- | ----------- |
| Create publication | `CREATE PUBLICATION supabase_to_aurora;` | Native replication setup |
| Add tables to publication | `ALTER PUBLICATION supabase_to_aurora ADD TABLE ...;` | Native replication setup |
| Create replication slot | `SELECT pg_create_logical_replication_slot('supabase_to_aurora_slot', 'pgoutput');` | Native replication setup |
| Drop replication slot (cleanup) | `SELECT pg_drop_replication_slot('supabase_to_aurora_slot');` | Post-migration cleanup |
| Drop publication (cleanup) | `DROP PUBLICATION supabase_to_aurora;` | Post-migration cleanup |
| Apply schema migrations | `CREATE TABLE`, `ALTER TABLE`, etc. | Schema changes on Supabase |

**How the agent handles read-only limitations**:
1. The agent generates the exact SQL or CLI command needed
2. The agent presents the command with an explanation of what it does and why
3. The agent asks the user to execute the command manually
4. The agent asks the user to confirm completion before proceeding
5. The agent verifies the result using read-only queries (e.g., checking `pg_publication`, `pg_replication_slots`)

### Aurora PostgreSQL MCP Server (pgwire + Secrets Manager)

The Aurora PostgreSQL MCP server connects to your Aurora cluster for all target database operations. It uses **pgwire** (PostgreSQL wire protocol) with **AWS Secrets Manager** for credential management.

**MANDATORY PREREQUISITE**: The agent MUST verify the Aurora PostgreSQL MCP server is properly configured before starting the Mobilize phase. If `AWS_PROFILE` or `AWS_REGION` are still set to placeholder values (`<your-aws-profile>`, `<your-aws-region>`), the MCP server will fail to connect. The agent should:

1. At the start of the migration (during or after the Assess phase), check if the Aurora MCP server is connected by calling `get_database_connection_info` or `is_database_connected`
2. If the server fails with a profile error, ask the user to update the MCP configuration:
   ```text
   "The Aurora PostgreSQL MCP server needs your AWS profile and region configured.
   Please update the MCP server settings:
   - Set AWS_PROFILE to your AWS CLI profile name (e.g., 'default')
   - Set AWS_REGION to your target region (e.g., 'us-east-1')
   You can do this in the Kiro MCP Servers panel or by editing .kiro/settings/mcp.json."
   ```
3. Do NOT proceed with provisioning until the MCP server is confirmed working

**Configuration** (in `mcp.json` — uses default AWS credential chain, no placeholders):

```json
{
  "awslabs.postgres-mcp-server": {
    "command": "uvx",
    "args": [
      "awslabs.postgres-mcp-server@latest",
      "--allow_write_query"
    ],
    "env": {
      "FASTMCP_LOG_LEVEL": "ERROR"
    },
    "disabled": false,
    "autoApprove": []
  }
}
```

**Note**: The server uses the standard AWS credential chain (environment variables → `~/.aws/config` default profile → instance profile). If the user needs a non-default profile or region, they can add `"AWS_PROFILE": "profile-name"` and `"AWS_REGION": "region"` to the `env` block. Do NOT use placeholders like `<your-aws-profile>` — they are passed literally and cause authentication failures.

**Connection setup** — After the MCP server starts and is configured, connect to Aurora:

```
Connect to database named <database-name> with database endpoint as <aurora-endpoint> with database_type as APG, using pgwire as connection method in <aws-region> region
```

**Prerequisites for pgwire connection**:
- `AWS_PROFILE` and `AWS_REGION` must be set to actual values (not placeholders)
- AWS credentials (via `AWS_PROFILE`) must have permission to access AWS Secrets Manager
- The Aurora cluster endpoint must be reachable from the MCP server host (direct TCP 5432 connectivity)
- `--allow_write_query` is included because migration requires write operations (schema creation, data loading, subscription setup)
- **IMPORTANT**: If the MCP server runs on the user's local machine (outside VPC), `pgwire` will NOT work for a private Aurora cluster. Use `rdsapi` connection method instead (see below).

For the connection method decision tree (rdsapi vs pgwire vs pgwire_iam) and MCP server known limitations, see `postgresql-migration-overview.md` → "Aurora PostgreSQL MCP — Connection Methods".

**Agent decision logic for Aurora MCP connection method:**
1. Is the MCP server running on the user's local machine? → Use `rdsapi`
2. Is the MCP server running inside the VPC (EC2, Cloud9)? → Use `pgwire`
3. Does the operation require > 45 seconds (e.g., `CREATE SUBSCRIPTION` with `copy_data = true`, large schema migrations)? → Cannot use `rdsapi`; direct user to CloudShell or use EC2 bastion

**Security best practices** (from Aurora PostgreSQL MCP documentation):
- Use `AWS_PROFILE` to specify the correct AWS profile — the MCP server creates a boto3 session using this profile
- AWS IAM credentials remain on your local machine and are used only for accessing AWS services
- Store Aurora master credentials in AWS Secrets Manager (not in environment variables or config files)
- Use least-privilege IAM policies — the profile needs `secretsmanager:GetSecretValue` and `rds:DescribeDBClusters` at minimum
- Ensure the Aurora cluster has encryption at rest enabled (KMS)
- Enforce SSL/TLS for all database connections (`sslmode=require`)

## Connectivity Verification Checklist

Before proceeding to migration execution, verify:

- [ ] Supabase MCP server connected (via OAuth) and can list tables
- [ ] Supabase instance is XL or greater
- [ ] Supabase external connectivity verified (DMS dual-stack mode preferred; IPv4 add-on only if already enabled)
- [ ] Supabase replication parameters are configured and verified
- [ ] Aurora PostgreSQL MCP server connected (uses default AWS credential chain; add AWS_PROFILE/AWS_REGION to env only if non-default profile needed)
- [ ] Aurora cluster is provisioned and accessible
- [ ] Aurora security group allows inbound from all required sources
- [ ] EC2 bastion can connect to Aurora via psql (if using bastion)
- [ ] EC2 bastion can connect to Supabase via psql (if using bastion for pg_dump)
- [ ] DMS endpoint connections tested successfully (if using DMS)
- [ ] AWS Secrets Manager contains Aurora credentials
