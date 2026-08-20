# Troubleshooting — PostgreSQL Migration

## Overview

Common errors and resolutions encountered during PostgreSQL migrations to Aurora/RDS, organized by category. Reference this file when encountering errors during any migration phase.

## Generic Database Connection Errors

### SSL/TLS Connection Failures

**Symptom**: `SSL connection is required` or `SSL error: certificate verify failed` when connecting to Aurora.

**Root Cause**: Aurora enforces SSL by default via the `rds.force_ssl` parameter. Clients must use `sslmode=require` (or stricter) and require the RDS CA certificate bundle.

**SSL Modes Summary**:

| Mode | Encrypted | CA Verified | Hostname Verified |
| ---- | --------- | ----------- | ----------------- |
| `disable` | ❌ | ❌ | ❌ |
| `allow` | maybe | ❌ | ❌ |
| `prefer` | maybe | ❌ | ❌ |
| `require` | ✅ | ❌ | ❌ |
| `verify-ca` | ✅ | ✅ | ❌ |
| `verify-full` | ✅ | ✅ | ✅ |

**Resolution**:
1. Verify the Aurora parameter group has `rds.force_ssl = 1` (this is the default and recommended setting — do NOT disable it)
2. Update the connection string to include `sslmode=require` at minimum:
   ```
   psql "host=<aurora-endpoint> port=5432 dbname=<db> user=postgres sslmode=require"
   ```
3. For `sslmode=verify-ca` or `sslmode=verify-full` (recommended for production), download the RDS CA bundle and set `sslrootcert`:
   ```bash
   wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
   psql "host=<aurora-endpoint> port=5432 dbname=<db> user=postgres sslmode=verify-full sslrootcert=global-bundle.pem"
   ```
4. For application connection pools, set the SSL option in the pool config:
   ```javascript
   // Node.js (pg) — sslmode=require equivalent
   const pool = new Pool({ ssl: { rejectUnauthorized: false } })

   // Node.js (pg) — sslmode=verify-full equivalent
   const pool = new Pool({ ssl: { rejectUnauthorized: true, ca: fs.readFileSync('global-bundle.pem') } })
   ```

### pg_hba.conf / Authentication Errors

**Symptom**: `FATAL: no pg_hba.conf entry for host` or `FATAL: password authentication failed`.

**Root Cause**: The client IP is not allowed to connect, or the credentials are incorrect.

**Resolution**:
1. On Aurora/RDS, `pg_hba.conf` is managed via security groups — verify the security group allows inbound TCP 5432 from the client IP or security group:
   ```bash
   aws ec2 describe-security-groups --group-ids <aurora-sg-id> --region <region> \
     --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`]"
   ```
2. Verify the password is correct — retrieve from Secrets Manager:
   ```bash
   aws secretsmanager get-secret-value --secret-id <secret-id> --region <region> \
     --query SecretString --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])"
   ```
3. Verify the username exists on the target: `SELECT usename FROM pg_user;`
4. For source databases (Supabase, Heroku, etc.), check the platform dashboard for the correct connection credentials

### Connection Timeout

**Symptom**: `connection timed out` or `could not connect to server: Operation timed out`.

**Root Cause**: Network path is blocked — security group, NACL, routing, or the endpoint is unreachable.

**Resolution**:
1. Verify the endpoint is correct (cluster endpoint for writes, reader endpoint for reads)
2. Check security group inbound rules allow TCP 5432 from the source
3. Check VPC route tables — if connecting from outside the VPC, ensure a NAT gateway or internet gateway is configured
4. Test basic connectivity: `nc -zv <endpoint> 5432` (should show "Connection succeeded")
5. If connecting via SSM to an EC2 bastion, verify the bastion's security group allows outbound to Aurora
6. For cross-VPC or cross-account connections, verify VPC peering or Transit Gateway routes


### IPv6 / Dual-Stack Connectivity

**Symptom**: DMS replication instance cannot connect to a source database that resolves to an IPv6 address (AAAA record only), or connections time out despite correct security group rules.

**Root Cause**: Some managed database providers (e.g., Supabase) expose endpoints that resolve to IPv6 addresses only. By default, DMS replication instances and Aurora clusters use IPv4-only networking. If the source endpoint is IPv6-only, the DMS replication instance cannot reach it unless Dual-stack mode is enabled.

**How to check**: Verify whether the source host resolves to IPv6:
```bash
# Check DNS resolution
dig AAAA db.<project-ref>.supabase.co
nslookup -type=AAAA db.<project-ref>.supabase.co

# If you see an AAAA record (e.g., 2a05:d014:...), the host is IPv6
```

**Resolution**: Follow the complete IPv6 and dual-stack setup steps in `supabase-prerequisites-and-connectivity.md` → "IPv6 and Dual-Stack Connectivity (Preferred for Supabase)" and → "Aurora Outbound IPv6 — Confirmed Limitation".

## DMS-Specific Errors

### Reading DMS Task Logs

DMS task logs are available in CloudWatch Logs under the log group `/aws/dms/tasks/<task-id>`. Key log signatures:

- `]E:` — Error messages (critical, usually cause task failure)
- `]W:` — Warning messages (may indicate data issues but task continues)
- `]I:` — Informational messages (normal operation)

**Troubleshooting workflow**:
1. Check the DMS task status in the console or via CLI:
   ```bash
   aws dms describe-replication-tasks \
     --filters Name=replication-task-id,Values=<task-id> \
     --region <region> \
     --query "ReplicationTasks[0].{Status:Status,StopReason:StopReason,LastError:ReplicationTaskStats}"
   ```
2. Search CloudWatch Logs for errors:
   ```bash
   aws logs filter-log-events \
     --log-group-name /aws/dms/tasks/<task-id> \
     --filter-pattern "]E:" \
     --region <region> \
     --query "events[*].message" --output text
   ```
3. Correlate DMS errors with target database logs by timestamp — Aurora PostgreSQL logs are in CloudWatch under `/aws/rds/cluster/<cluster-id>/postgresql`
4. Common DMS error patterns:
   - `Cannot find or open table` — table doesn't exist on target; check schema migration
   - `No primary key` — CDC cannot replicate UPDATE/DELETE without a PK; add PK or use full load only
   - `Replication slot does not exist` — slot was dropped or never created; recreate on source
   - `ENUM type does not exist` — create ENUM types on target before starting DMS task

### DMS Endpoint Connection Failures

**Symptom**: `test-connection` returns `failed` status.

**Resolution**:
1. For source endpoints (Supabase, Heroku, etc.): verify the DMS replication instance subnet has internet access (NAT gateway) since source databases are on the public internet
2. For target endpoints (Aurora): verify the DMS replication instance security group can reach Aurora on port 5432
3. Verify credentials are correct — test with psql from an EC2 instance in the same VPC
4. Check that the DMS replication instance is in the correct VPC and subnet group

### DMS Full Load + CDC Foreign Key Handling

**Symptom**: DMS full load fails with foreign key constraint violations, or `TRUNCATE TABLE` / `DROP TABLE` fails with `cannot truncate a table referenced in a foreign key constraint`.

**Root Cause**: DMS loads each table one at a time during full load, but table order is not guaranteed. If a child table's rows arrive before the parent table's rows, foreign key constraints cause the task to fail. PostgreSQL also has a failsafe mechanism that prevents truncation even when `session_replication_role` is set to `replica`.

Reference: [AWS DMS — Using PostgreSQL as a target](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.PostgreSQL.html)

**Resolution — Method 1: Use `session_replication_role` (recommended)**

Set the `session_replication_role` parameter to `replica` on the DMS target endpoint. This deactivates foreign key triggers during full load so DMS can load tables in any order without FK violations.

You can set this via the DMS endpoint Extra Connection Attribute (ECA):
```
AfterConnectScript=SET session_replication_role = 'replica'
```

Or set it via the Aurora/RDS custom parameter group before starting the DMS task:

1. In the RDS console (or via CLI), modify the custom parameter group for your Aurora cluster
2. Set `session_replication_role` = `replica`
3. Apply the change (no reboot required — this is a dynamic parameter)

```bash
# Via AWS CLI
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=replica,ApplyMethod=immediate"
```

**Note**: `ALTER SYSTEM` is not allowed on RDS/Aurora — you must use the parameter group.

After migration cutover is complete and the application is writing directly to Aurora, re-enable FK enforcement:

```bash
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=origin,ApplyMethod=immediate"
```

Then verify FK integrity:
```sql
-- Check for orphaned rows (example for a specific FK relationship)
SELECT child.id FROM child_table child
LEFT JOIN parent_table parent ON child.parent_id = parent.id
WHERE parent.id IS NULL;
```

**Resolution — Method 2: Use `DO_NOTHING` target table preparation mode**

If you set the DMS task's target table preparation mode to `DO_NOTHING`, DMS will not attempt to `DROP` or `TRUNCATE` target tables before loading. This avoids the PostgreSQL failsafe that blocks truncation of tables referenced by foreign keys — even when `session_replication_role` is set to `replica`.

Use `DO_NOTHING` when:
- Tables already exist on the target with the correct schema
- You want to avoid any risk of FK-related DROP/TRUNCATE failures
- You are reloading specific tables into an existing target

```json
"FullLoadSettings": {
  "TargetTablePrepMode": "DO_NOTHING"
}
```

**Important**: If using `DO_NOTHING`, ensure the target tables are empty before starting the full load (or handle duplicates in your application logic).

### DMS Truncate/Drop Cascade for Reload Scenarios

**Symptom**: DMS task with `DROP_AND_CREATE` or `TRUNCATE_BEFORE_LOAD` target table prep mode fails because PostgreSQL cannot drop or truncate a table that is referenced by foreign key constraints from other tables.

**Root Cause**: PostgreSQL prevents `TRUNCATE` and `DROP` on tables that have dependent foreign keys from other tables. Even with `session_replication_role = 'replica'`, the failsafe mechanism blocks truncation of FK-referenced tables.

**Resolution**:

If you need to reload tables from scratch (e.g., after a failed DMS task), manually truncate with `CASCADE` before restarting the DMS task. `TRUNCATE ... CASCADE` automatically truncates all dependent (child) tables as well.

```sql
-- Truncate the parent table and all tables that reference it via FK
TRUNCATE TABLE parent_table CASCADE;
```

**Critical**: When you use `CASCADE`, all related tables in the FK chain are also truncated. You MUST reload ALL cascaded tables — not just the one you intended to truncate. Otherwise those dependent tables will be empty.

Steps for a clean reload:
1. Identify all tables that will be affected by the cascade:
   ```sql
   -- Find all tables that reference a given table via FK
   SELECT DISTINCT tc.table_schema, tc.table_name
   FROM information_schema.table_constraints tc
   JOIN information_schema.constraint_column_usage ccu
     ON tc.constraint_name = ccu.constraint_name
   WHERE tc.constraint_type = 'FOREIGN KEY'
     AND ccu.table_name = 'parent_table';
   ```
2. Run `TRUNCATE ... CASCADE` on the target:
   ```sql
   TRUNCATE TABLE parent_table CASCADE;
   ```
3. Ensure the DMS task table mappings include ALL cascaded tables so they are all reloaded
4. Restart the DMS task with `TargetTablePrepMode: DO_NOTHING` (since you already truncated manually)

Alternatively, disable FK checks via the parameter group, truncate individually, then re-enable:

```bash
# Disable FK triggers via parameter group
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=replica,ApplyMethod=immediate"
```

**Note**: `ALTER SYSTEM` is not allowed on RDS/Aurora — you must use the parameter group.

Then truncate tables in reverse dependency order (children before parents):

```sql
TRUNCATE TABLE child_table;
TRUNCATE TABLE parent_table;
```

After truncation, re-enable FK enforcement:

```bash
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=origin,ApplyMethod=immediate"
```

This avoids the cascade but requires you to truncate tables in reverse dependency order (children before parents).

## Aurora Data API Errors

### UUID Type Mismatch

**Symptom**: `operator does not exist: uuid = text` when using Aurora Data API with UUID columns.

**Root Cause**: The RDS Data API treats all string parameters as `text` type by default. When comparing against a `uuid` column, PostgreSQL cannot implicitly cast `text` to `uuid` in the `=` operator.

**Resolution**:

Option A — Add `typeHint: "UUID"` to the SqlParameter:
```json
{
  "name": "id",
  "value": { "stringValue": "550e8400-e29b-41d4-a716-446655440000" },
  "typeHint": "UUID"
}
```

Option B — Cast to `::text` in the SQL query:
```sql
SELECT * FROM users WHERE id::text = :id
```

Option C — Cast the parameter to `::uuid` in the SQL query:
```sql
SELECT * FROM users WHERE id = :id::uuid
```

Option A is preferred because it preserves index usage on the `uuid` column. Options B and C may prevent index scans depending on the query planner.

### Data API Timeout (45 seconds)

**Symptom**: `Communications link failure` or timeout after ~45 seconds.

**Root Cause**: Aurora Data API has a hard 45-second execution timeout per statement.

**Resolution**:
- For long-running queries, use a direct psql connection (via bastion/SSM) instead of Data API
- For large data operations, batch into smaller transactions
- Data API is best suited for small, quick operations (CRUD, validation queries, ad-hoc checks)

## Deployment Platform Errors

### Vercel OIDC / IAM Authentication Failures

**Symptom**: `InvalidIdentityTokenException` or `OpenIDConnect provider's HTTPS certificate doesn't match configured thumbprint` when using Vercel OIDC with AWS.

**Root Cause**: The AWS IAM OIDC provider for Vercel is not configured correctly, or the trust policy doesn't match the Vercel project/team.

**Resolution**:
1. Create the OIDC provider in IAM (if not already done):
   ```bash
   aws iam create-open-id-connect-provider \
     --url https://oidc.vercel.com \
     --client-id-list https://vercel.com/<team-slug> \
     --thumbprint-list <thumbprint>
   ```
2. Get the correct thumbprint — this changes periodically. Use the AWS CLI to fetch it:
   ```bash
   # Get the OIDC provider thumbprint
   openssl s_client -servername oidc.vercel.com -showcerts -connect oidc.vercel.com:443 </dev/null 2>/dev/null | \
     openssl x509 -fingerprint -noout | sed 's/://g' | cut -d= -f2 | tr '[:upper:]' '[:lower:]'
   ```
3. Verify the IAM role trust policy matches the Vercel project:
   ```json
   {
     "Effect": "Allow",
     "Principal": { "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.vercel.com" },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": {
         "oidc.vercel.com:aud": "https://vercel.com/<team-slug>",
         "oidc.vercel.com:sub": "owner:<team-slug>:project:<project-name>:environment:production"
       }
     }
   }
   ```
4. If the thumbprint error persists, update the OIDC provider with the latest thumbprint:
   ```bash
   aws iam update-open-id-connect-provider-thumbprint \
     --open-id-connect-provider-arn arn:aws:iam::<account-id>:oidc-provider/oidc.vercel.com \
     --thumbprint-list <new-thumbprint>
   ```

### Vercel / Serverless Platform Cannot Connect to Private Aurora

**Symptom**: Application deployed on Vercel (or similar serverless platform) cannot connect to Aurora in a private VPC.

**Root Cause**: Vercel functions run outside your VPC and cannot reach private Aurora endpoints.

**Resolution options**:
1. **Aurora Data API** — no VPC connectivity needed; access Aurora via HTTPS. Best for simple CRUD operations (< 45s execution time). Requires enabling Data API on the Aurora cluster.
2. **Aurora public access** — enable public accessibility on the Aurora instance and restrict via security group to known IPs. Less secure but simplest.
3. **Vercel Secure Compute** (Enterprise plan) — provides VPC peering from Vercel to your AWS VPC. Most secure but requires Vercel Enterprise.
4. **AWS Amplify Hosting** — alternative to Vercel that runs inside your AWS account. SSR uses Lambda functions that can be configured with VPC access to reach private Aurora. Also supports IAM compute roles and VPC endpoints for private connectivity.
5. **PrivateLink** — create a VPC endpoint service for Aurora and connect from Vercel via PrivateLink (requires Vercel Enterprise).

For the complete OIDC setup, IAM role configuration, and Aurora Data API client implementation, see `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity".
