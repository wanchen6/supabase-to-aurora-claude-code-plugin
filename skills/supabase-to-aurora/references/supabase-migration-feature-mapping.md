# Supabase to Aurora / RDS PostgreSQL — Feature Mapping and Replacement Guide

## Overview

This file maps Supabase platform features to their AWS replacements when migrating from Supabase to Aurora PostgreSQL or RDS PostgreSQL. It covers schema exclusions, Auth → Cognito, Storage → S3, Realtime → AppSync, Edge Functions → Lambda, PostgREST API replacement, Supabase CLI replication configuration, Supabase MCP tool usage (read-only), DMS plugin configuration, and cost comparison. Use this alongside the general-purpose phase files (01–04) which handle the database migration itself.

## Supabase Schema Exclusions

Supabase creates several internal schemas that should NOT be migrated to Aurora. Exclude these during schema export and data migration:

| Schema | Purpose | Migrate? |
| ------ | ------- | -------- |
| `public` | Application tables | Yes — primary migration target |
| `auth` | Supabase Auth (users, sessions, tokens) | No — replace with Amazon Cognito or custom auth |
| `storage` | Supabase Storage (file/bucket metadata) | No — replace with Amazon S3 |
| `realtime` | Supabase Realtime (subscriptions) | No — replace with AWS AppSync or WebSocket API |
| `extensions` | PostgreSQL extension installations | No — recreate extensions on Aurora |
| `supabase_functions` | Edge Functions metadata | No — replace with AWS Lambda |

When using `pg_dump`, exclude these schemas:

```bash
pg_dump -h db.<project-ref>.supabase.co -U postgres -d postgres \
  --no-owner --no-acl \
  -N auth -N storage -N realtime -N extensions -N supabase_functions \
  -F c -f supabase_dump.backup
```

When configuring DMS table mappings, include only the `public` schema (or other application schemas):

```json
{
  "rules": [{
    "rule-type": "selection",
    "rule-id": "1",
    "rule-name": "select-public-tables",
    "object-locator": { "schema-name": "public", "table-name": "%" },
    "rule-action": "include"
  }]
}
```

## Supabase Extension Compatibility with Aurora PostgreSQL

Supabase installs several extensions. Classify each before migration:

| Extension | Available on Aurora? | Action |
| --------- | -------------------- | ------ |
| `pgcrypto` | Yes | Recreate on Aurora (`CREATE EXTENSION IF NOT EXISTS pgcrypto;`) |
| `uuid-ossp` | Yes | Recreate on Aurora |
| `pg_stat_statements` | Yes | Recreate on Aurora (add to `shared_preload_libraries` via parameter group) |
| `postgis` | Yes | Recreate on Aurora |
| `pg_trgm` | Yes | Recreate on Aurora |
| `hstore` | Yes | Recreate on Aurora |
| `citext` | Yes | Recreate on Aurora |
| `pg_graphql` | No — Supabase-specific | Replace with AWS AppSync or API Gateway + Lambda |
| `pg_net` | No — Supabase-specific | Replace with AWS Lambda (HTTP calls from application layer) |
| `pgjwt` | No — Supabase-specific | Replace with Amazon Cognito JWT validation or application-layer JWT handling |
| `supautils` | No — Supabase-specific | Internal Supabase utility — no replacement needed (not used by application code) |

Use `list_extensions` via Supabase MCP to discover all installed extensions, then cross-reference with this table. For extensions not listed here, use AWS Knowledge MCP `search_documentation` with query `"Aurora PostgreSQL supported extensions"` to verify availability.

## Supabase Feature Audit Queries

Run these queries on Supabase (via Supabase MCP `execute_sql` or psql) to understand which platform features are in use:

```sql
-- Auth users count
SELECT COUNT(*) AS auth_user_count FROM auth.users;

-- Auth providers in use
SELECT DISTINCT provider FROM auth.identities;

-- Storage objects count and total size
SELECT COUNT(*) AS object_count,
  pg_size_pretty(SUM(metadata->>'size')::bigint) AS total_size
FROM storage.objects;

-- Storage buckets
SELECT id, name, public, created_at FROM storage.buckets ORDER BY name;

-- Realtime subscriptions (if table exists)
SELECT COUNT(*) AS subscription_count FROM realtime.subscription;

-- Edge Functions metadata
SELECT id, name, created_at FROM supabase_functions.functions ORDER BY name;
```

### RLS Policies Referencing Supabase Functions

Identify RLS policies that use Supabase-specific functions (these need updating after migration):

```sql
-- Policies referencing auth.uid() or auth.jwt()
SELECT schemaname, tablename, policyname, qual, with_check
FROM pg_policies
WHERE qual::text LIKE '%auth.uid()%'
   OR qual::text LIKE '%auth.jwt()%'
   OR with_check::text LIKE '%auth.uid()%'
   OR with_check::text LIKE '%auth.jwt()%'
ORDER BY schemaname, tablename;
```

These policies will fail on Aurora because `auth.uid()` and `auth.jwt()` are Supabase-specific functions. Options:
1. Replace with application-layer authorization (remove RLS, enforce in app code)
2. Create equivalent functions on Aurora that extract user identity from a session variable (e.g., `current_setting('app.user_id')`)
3. Use Amazon Cognito tokens passed via application and set as session variables

## Supabase Auth → Amazon Cognito (or Alternative)

Supabase Auth (`auth.*` schema) provides user management, JWT tokens, OAuth providers, and magic links.

### Replacement Options

1. **Amazon Cognito** — managed user pools, OAuth/OIDC, MFA, hosted UI
2. **Auth0** — third-party, feature-rich, easy migration
3. **Custom JWT auth** — application-level auth with Aurora as the user store

### Migration Steps (Cognito)

1. Export user data from Supabase:
   ```sql
   SELECT id, email, raw_user_meta_data, created_at
   FROM auth.users
   ORDER BY created_at;
   ```

2. Import users to Cognito using `AdminCreateUser` API or CSV import

3. Handle passwords — passwords cannot be migrated directly:
   - Option A: Force password reset for all users on first login
   - Option B: Use Cognito "migrate user" Lambda trigger — authenticates against old system on first login, then stores in Cognito
   - Option C: Send password reset emails to all users before cutover

4. Configure OAuth providers (Google, GitHub, Apple, etc.) in Cognito to match Supabase Auth providers

5. Update application to use Cognito SDK instead of Supabase Auth SDK

### RLS Policy Updates

If the application uses RLS policies that reference `auth.uid()`:

```sql
-- Create a replacement function on Aurora
CREATE OR REPLACE FUNCTION app_user_id() RETURNS uuid AS $$
  SELECT current_setting('app.user_id', true)::uuid;
$$ LANGUAGE sql STABLE;

-- Update RLS policies to use the new function
-- Example: replace auth.uid() with app_user_id()
ALTER POLICY "Users can view own data" ON public.profiles
  USING (user_id = app_user_id());
```

The application must set the session variable before each request:

```sql
SET LOCAL app.user_id = '<cognito-user-sub>';
```

## Supabase Storage → Amazon S3

Supabase Storage (`storage.*` schema) provides file storage with bucket management.

### Replacement

- **Amazon S3** for file storage
- **CloudFront** for CDN delivery
- Application-level metadata management (or DynamoDB for file metadata)

### Migration Steps

1. Export file metadata from Supabase:
   ```sql
   SELECT id, bucket_id, name, metadata, created_at
   FROM storage.objects
   ORDER BY bucket_id, name;
   ```

2. Download files from Supabase Storage API:
   ```bash
   # For each file, download via Supabase Storage API
   curl -H "Authorization: Bearer <service-role-key>" \
     "https://<project-ref>.supabase.co/storage/v1/object/<bucket>/<path>" \
     -o "<local-path>"
   ```

3. Create matching S3 buckets and upload files:
   ```bash
   aws s3 cp <local-path> s3://<bucket-name>/<path>
   ```

4. Update application to use AWS SDK for S3 operations instead of Supabase Storage SDK

5. Configure CloudFront distribution for public file delivery (if applicable)

## Supabase Realtime → AWS AppSync or WebSocket API

Supabase Realtime provides real-time subscriptions to database changes via WebSocket.

### Replacement Options

1. **AWS AppSync** — managed GraphQL with real-time subscriptions
2. **API Gateway WebSocket API** — custom WebSocket connections
3. **Amazon SNS/SQS** — event-driven notifications (not real-time to client)
4. **Aurora PostgreSQL triggers + Lambda** — trigger Lambda on data changes, push to clients via AppSync or WebSocket

### Migration Approach

1. Identify which tables/events have Realtime subscriptions in the application
2. Choose the replacement based on the subscription pattern:
   - Simple data change notifications → AppSync subscriptions
   - Complex filtering or custom logic → API Gateway WebSocket + Lambda
   - Server-to-server events → SNS/SQS
3. Implement the replacement in the application layer
4. Test real-time behavior matches the original Supabase Realtime experience

## Supabase Edge Functions → AWS Lambda

Supabase Edge Functions run Deno-based serverless functions at the edge.

### Replacement Options

1. **AWS Lambda** — serverless compute, any runtime (Node.js, Python, Go, etc.)
2. **Lambda@Edge** — for CloudFront edge processing
3. **CloudFront Functions** — lightweight edge logic (viewer request/response only)

### Migration Steps

1. List all Edge Functions:
   ```sql
   SELECT id, name, created_at FROM supabase_functions.functions ORDER BY name;
   ```

2. For each function:
   - Review the Deno/TypeScript source code and dependencies
   - Rewrite in the target Lambda runtime (Node.js recommended for Deno→JS migration)
   - Configure API Gateway or direct Lambda invocation
   - Update application to call Lambda endpoints instead of Supabase Edge Function URLs

3. Test each migrated function against the same inputs/outputs as the original

## Supabase PostgREST API Replacement

Supabase auto-generates a REST API from the database schema via PostgREST.

### Replacement Options

1. **Direct database connection** — most applications already have this; simplest option
2. **API Gateway + Lambda** — for a REST API layer over Aurora
3. **AWS AppSync** — for a GraphQL API over Aurora
4. **PostgREST on ECS/Fargate** — if you want to keep the exact same API pattern (self-hosted)

### Decision Guidance

- If the application primarily uses the Supabase client SDK for data access → switch to direct database connection via an ORM or query builder
- If external clients depend on the REST API → build an API Gateway + Lambda layer
- If the team prefers GraphQL → use AppSync with Aurora as the data source

## Supabase CLI Replication Configuration

Supabase requires CLI commands to configure replication parameters. These cannot be done via SQL or the Supabase MCP server.

### Required Configuration

```bash
supabase --experimental --project-ref <project-ref> postgres-config update \
  --config max_replication_slots=10 \
  --config max_wal_senders=10 \
  --config wal_sender_timeout=0 \
  --config max_slot_wal_keep_size=4GB \
  --config max_wal_size=2GB
```

### Prerequisites

- Supabase CLI installed (`npm install -g supabase`)
- Logged in (`supabase login`)
- Instance size XL or greater recommended for replication performance (logical replication is available on all Supabase plans including Free, but XL provides better throughput for production migrations)
- IPv4 add-on NOT required if VPC supports IPv6 (use DMS dual-stack mode instead). Only use IPv4 add-on if already enabled on the Supabase project.

### Verification

After the user runs the CLI commands, verify the configuration via Supabase MCP:

```sql
SHOW max_replication_slots;  -- Should be 10
SHOW max_wal_senders;        -- Should be 10
SHOW wal_sender_timeout;     -- Should be 0
```

## Supabase MCP Tool Usage

For available MCP tools and read-only mode behavior, see `supabase-prerequisites-and-connectivity.md` → "Supabase MCP Server".

## Application Code Refactoring — Supabase SDK to Direct PostgreSQL Connection

When an application uses the Supabase client SDK (`@supabase/supabase-js`) for data access, the data layer must be refactored to use a direct PostgreSQL connection after migrating to Aurora. The Supabase SDK communicates with Supabase's PostgREST API, which is not available on Aurora.

### When This Applies

This section applies when the application:
- Imports `@supabase/supabase-js` or `@supabase/ssr`
- Uses `createClient(SUPABASE_URL, SUPABASE_ANON_KEY)` to create a database client
- Accesses data via `.from("table").select(...)`, `.insert()`, `.update()`, `.delete()` patterns
- Does NOT use a direct PostgreSQL connection string (e.g., `DATABASE_URL`, `pg`, Prisma, Drizzle)

If the application already uses a direct PostgreSQL connection (via `DATABASE_URL` or an ORM), only the connection string needs updating — no code refactoring is needed.

### Agent Workflow for Automated Refactoring

The agent should follow this workflow to refactor the application code:

1. **Discover the data layer files**: Scan the application source for files that import from `@supabase/supabase-js` or the local Supabase client helper. Common locations:
   - `lib/supabase/server.ts` or `lib/supabase/client.ts` — Supabase client factory
   - `lib/data.ts` or `lib/queries.ts` — read queries (SELECT)
   - `app/actions.ts` or `lib/actions.ts` — server actions / mutations (INSERT, UPDATE, DELETE)
   - `api/` route handlers that use the Supabase client

2. **Identify Supabase SDK patterns**: Read each file and catalog the SDK method calls. Map them to the equivalent SQL (see pattern mapping table below).

3. **Choose a replacement driver**: Recommend based on the application's complexity:
   - `pg` (node-postgres) — simplest, raw SQL, good for apps with straightforward queries
   - `drizzle-orm` — type-safe, modern, good for apps that want ORM-like DX without heavy abstraction
   - `prisma` — schema-first, rich ecosystem, good for apps that want full ORM features
   - For most Supabase SDK migrations, `pg` is the simplest drop-in since the SDK patterns map directly to SQL

4. **Generate the refactored code**: For each file, produce the equivalent code using the chosen driver. Present the changes to the user for review.

5. **Update environment variables**: Replace Supabase-specific env vars with PostgreSQL connection parameters.

6. **Install new dependencies** — detect the package manager from the lockfile, then run the install command **before writing any code that imports the new package**. Do not write imports first. See "Package.json Changes" below for the exact commands by package manager.

7. **Test the refactored app**: Guide the user to run the app locally against Aurora and verify all CRUD operations work.

### Supabase SDK → SQL Pattern Mapping

| Supabase SDK Pattern | Equivalent SQL (parameterized) |
| -------------------- | ------------------------------ |
| `.from("table").select("*")` | `SELECT * FROM table` |
| `.from("table").select("col1, col2")` | `SELECT col1, col2 FROM table` |
| `.from("table").select("*, other_table(*)")` | `SELECT t.*, to_jsonb(o.*) FROM table t LEFT JOIN other_table o ON ...` (or separate queries) |
| `.from("table").select("*, join_table(related(*))") ` | Multi-level join — flatten with JOINs or use subqueries |
| `.eq("column", value)` | `WHERE column = $1` |
| `.neq("column", value)` | `WHERE column != $1` |
| `.gt("column", value)` | `WHERE column > $1` |
| `.gte("column", value)` | `WHERE column >= $1` |
| `.lt("column", value)` | `WHERE column < $1` |
| `.lte("column", value)` | `WHERE column <= $1` |
| `.like("column", pattern)` | `WHERE column LIKE $1` |
| `.ilike("column", pattern)` | `WHERE column ILIKE $1` |
| `.is("column", null)` | `WHERE column IS NULL` |
| `.in("column", [values])` | `WHERE column = ANY($1)` |
| `.order("column", { ascending: true })` | `ORDER BY column ASC` |
| `.order("column", { ascending: false })` | `ORDER BY column DESC` |
| `.limit(n)` | `LIMIT n` |
| `.range(start, end)` | `LIMIT (end - start + 1) OFFSET start` |
| `.single()` | Append `LIMIT 1` and return first row |
| `.insert({ col: val })` | `INSERT INTO table (col) VALUES ($1)` |
| `.insert([{...}, {...}])` | `INSERT INTO table (cols) VALUES ($1, $2), ($3, $4)` |
| `.update({ col: val }).eq("id", id)` | `UPDATE table SET col = $1 WHERE id = $2` |
| `.delete().eq("id", id)` | `DELETE FROM table WHERE id = $1` |
| `.upsert({ col: val })` | `INSERT INTO table (cols) VALUES ($1) ON CONFLICT (pk) DO UPDATE SET col = $1` |

### Handling Joins (Supabase Nested Selects)

Supabase SDK supports nested selects that auto-resolve foreign key relationships:

```typescript
// Supabase SDK — nested select via foreign key
const { data } = await supabase
  .from("tasks")
  .select(`*, subtasks (*), task_labels (labels (*))`)
```

This translates to multiple approaches in raw SQL:

**Option A: Separate queries (simplest)**
```typescript
const tasks = await pool.query('SELECT * FROM tasks WHERE project_id = $1 ORDER BY position', [projectId])
for (const task of tasks.rows) {
  const subtasks = await pool.query('SELECT * FROM subtasks WHERE task_id = $1 ORDER BY position', [task.id])
  const labels = await pool.query(
    `SELECT l.* FROM labels l
     JOIN task_labels tl ON tl.label_id = l.id
     WHERE tl.task_id = $1`, [task.id]
  )
  task.subtasks = subtasks.rows
  task.labels = labels.rows
}
```

**Option B: JOIN with post-processing (fewer queries)**
```typescript
const { rows } = await pool.query(`
  SELECT t.*, s.id AS subtask_id, s.title AS subtask_title, s.is_completed,
         l.id AS label_id, l.name AS label_name, l.color AS label_color
  FROM tasks t
  LEFT JOIN subtasks s ON s.task_id = t.id
  LEFT JOIN task_labels tl ON tl.task_id = t.id
  LEFT JOIN labels l ON l.id = tl.label_id
  WHERE t.project_id = $1
  ORDER BY t.position, s.position
`, [projectId])
// Post-process rows to nest subtasks and labels into task objects
```

**Option C: PostgreSQL JSON aggregation (single query, structured output)**
```typescript
const { rows } = await pool.query(`
  SELECT t.*,
    COALESCE((SELECT json_agg(s ORDER BY s.position)
              FROM subtasks s WHERE s.task_id = t.id), '[]') AS subtasks,
    COALESCE((SELECT json_agg(l)
              FROM labels l JOIN task_labels tl ON tl.label_id = l.id
              WHERE tl.task_id = t.id), '[]') AS labels
  FROM tasks t
  WHERE t.project_id = $1
  ORDER BY t.position
`, [projectId])
```

Recommend Option A for simplicity or Option C for performance with larger datasets.

### Connection Helper Replacement

Replace the Supabase client factory with a PostgreSQL connection pool:

**Before (Supabase SDK):**
```typescript
import { createClient as createSupabaseClient } from '@supabase/supabase-js'

export function createClient() {
  return createSupabaseClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

**After (pg — node-postgres):**
```typescript
import { Pool } from 'pg'

const pool = new Pool({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT || '5432'),
  database: process.env.DATABASE_NAME,
  user: process.env.DATABASE_USER,
  password: process.env.DATABASE_PASSWORD,
  ssl: process.env.DATABASE_SSL === 'true' ? { rejectUnauthorized: false } : false,
  max: 10,
})

export { pool }
```

**After (Drizzle ORM):**
```typescript
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'

const pool = new Pool({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT || '5432'),
  database: process.env.DATABASE_NAME,
  user: process.env.DATABASE_USER,
  password: process.env.DATABASE_PASSWORD,
  ssl: process.env.DATABASE_SSL === 'true' ? { rejectUnauthorized: false } : false,
})

export const db = drizzle(pool)
```

### Environment Variable Changes

**For Vercel deployments using Aurora Data API + OIDC (recommended):**

| Before (Supabase) | After (Aurora Data API + OIDC) |
| ------------------ | ------------------------------ |
| `NEXT_PUBLIC_SUPABASE_URL` | (removed) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | (removed) |
| — | `AWS_REGION` (pin to Aurora's region, e.g., `us-east-1`) |
| — | `AWS_ROLE_ARN` (IAM role ARN for OIDC — no secret) |
| — | `AURORA_CLUSTER_ARN` (Aurora cluster ARN — no secret) |
| — | `AURORA_SECRET_ARN` (Secrets Manager ARN — no secret) |
| — | `AURORA_DATABASE` (database name, e.g., `postgres`) |

**IMPORTANT**: No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` needed. Vercel OIDC provides short-lived credentials automatically. See "Vercel + Aurora Connectivity" section for full OIDC setup.

**For direct PostgreSQL connection (EC2, ECS, Lambda in VPC):**

| Before (Supabase) | After (Aurora PostgreSQL) |
| ------------------ | ------------------------ |
| `NEXT_PUBLIC_SUPABASE_URL` | `DATABASE_HOST` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | (removed — use IAM auth or Secrets Manager) |
| — | `DATABASE_PORT` (default: 5432) |
| — | `DATABASE_NAME` |
| — | `DATABASE_USER` |
| — | `DATABASE_SSL` (true for Aurora) |

For direct connections, retrieve the password from Secrets Manager at runtime — never store it as a plaintext environment variable in production.

### Package.json Changes

**MANDATORY**: Never edit `package.json` directly to add or remove dependencies. Always use the package manager CLI — this updates both `package.json` AND the lockfile atomically. Editing `package.json` by hand leaves the lockfile out of sync, causing install failures or mismatched versions on CI/Vercel.

**Before running any install command, detect the package manager from the lockfile**: `pnpm-lock.yaml` → use `pnpm add`, `yarn.lock` → use `yarn add`, otherwise use `npm install`. Always install packages before writing code that imports them — do not write imports first and install later.

After installing, **commit `package.json` and the lockfile together** in the same commit.

```bash
# npm
npm install @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider
# pnpm
pnpm add @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider
# yarn
yarn add @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider

# Optionally remove Supabase SDK (only if no other Supabase features are used)
npm uninstall @supabase/supabase-js @supabase/ssr   # or pnpm remove / yarn remove
```

If using Drizzle:
```bash
npm install drizzle-orm pg && npm install -D @types/pg drizzle-kit
# pnpm: pnpm add drizzle-orm pg && pnpm add -D @types/pg drizzle-kit
```

### Server Actions Refactoring Pattern

Supabase server actions follow a consistent pattern that maps directly to parameterized SQL:

**Before (Supabase SDK):**
```typescript
"use server"
import { createClient } from "@/lib/supabase/server"
import { revalidatePath } from "next/cache"

export async function createProject(formData: FormData) {
  const supabase = createClient()
  const name = formData.get("name") as string
  const description = formData.get("description") as string
  const color = formData.get("color") as string || "#3b82f6"

  const { error } = await supabase.from("projects").insert({
    name, description: description || null, color,
  })

  if (error) { return { error: error.message } }
  revalidatePath("/")
  return { success: true }
}
```

**After (pg — node-postgres):**
```typescript
"use server"
import { pool } from "@/lib/supabase/server"
import { revalidatePath } from "next/cache"

export async function createProject(formData: FormData) {
  const name = formData.get("name") as string
  const description = formData.get("description") as string
  const color = formData.get("color") as string || "#3b82f6"

  try {
    await pool.query(
      'INSERT INTO projects (name, description, color) VALUES ($1, $2, $3)',
      [name, description || null, color]
    )
  } catch (err: any) {
    console.error("Error creating project:", err)
    return { error: err.message }
  }

  revalidatePath("/")
  return { success: true }
}
```

Key differences:
- Replace `createClient()` with the shared `pool` import
- Replace `.from("table").insert({...})` with `pool.query('INSERT ...', [params])`
- Replace `.from("table").update({...}).eq("id", id)` with `pool.query('UPDATE ... WHERE id = $1', [id])`
- Replace `.from("table").delete().eq("id", id)` with `pool.query('DELETE ... WHERE id = $1', [id])`
- Error handling changes from `{ data, error }` destructuring to try/catch
- Always use parameterized queries (`$1`, `$2`, ...) — never string interpolation

### Verification Checklist

After refactoring, verify:

- [ ] App starts without errors (`npm run dev`)
- [ ] All read operations return data (list pages, detail pages)
- [ ] All write operations work (create, update, delete)
- [ ] Joins/nested data loads correctly (e.g., tasks with subtasks and labels)
- [ ] Error handling works (try invalid inputs)
- [ ] No Supabase SDK imports remain (unless other Supabase features are still in use)
- [ ] Environment variables are correctly set for Aurora
- [ ] Connection pooling is configured (max connections, idle timeout)

## Vercel + Aurora Connectivity

Vercel serverless functions run outside your AWS VPC and cannot directly connect to a private Aurora instance. Choose a connectivity strategy based on your Vercel plan and security requirements.

### Connectivity Options

| Option | Vercel Plan | Security | Complexity | Best For |
| ------ | ----------- | -------- | ---------- | -------- |
| Aurora Data API | Any | High (HTTPS only, IAM auth) | Low | Simple CRUD apps, < 45s queries |
| AWS Amplify Hosting | N/A (alternative) | High (IAM roles, VPC-enabled Lambda) | Medium | Apps needing VPC access without Vercel |
| Aurora public access | Any | Lower (IP-restricted) | Low | Dev/test only |
| Vercel Secure Compute + PrivateLink | Enterprise | Highest (VPC peering) | High | Production with strict security |

### Recommended: Aurora Data API + Vercel OIDC

Aurora Data API provides HTTPS access to Aurora without VPC connectivity. Authentication uses **Vercel OIDC federation** with AWS IAM — no static credentials.

**NEVER use static AWS access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) for Vercel deployments.** Use Vercel OIDC federation which provides short-lived, auto-rotating credentials.

**Enable Data API on the Aurora cluster:**

```bash
aws rds enable-http-endpoint --resource-arn arn:aws:rds:<region>:<account>:cluster:<cluster-id> --region <region>
```

#### Vercel OIDC + AWS Setup

**Step 1: Enable OIDC in Vercel project**
- Vercel Dashboard → Project → Settings → Security → Enable "Secure backend access with OIDC federation"

**Step 2: Create AWS OIDC Identity Provider**

First, confirm your Vercel project ID — find it in the Vercel Dashboard → Project → Settings → General → "Project ID". Do NOT infer it from existing IAM resources. The project ID looks like `prj_xxxxxxxxxxxx`.

```bash
aws iam create-open-id-connect-provider \
  --url https://oidc.vercel.com \
  --client-id-list "urn:vercel:project:<YOUR_VERCEL_PROJECT_ID>" \
  --thumbprint-list "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
```

**Step 3: Create IAM Role with trust policy**

Save as `trust-policy.json`. Use `StringLike` with a wildcard on the `sub` claim to allow all Vercel environments (production, preview, development) — not just production:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.vercel.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.vercel.com:aud": "urn:vercel:project:<VERCEL_PROJECT_ID>"
      },
      "StringLike": {
        "oidc.vercel.com:sub": "owner:<VERCEL_TEAM_SLUG>:project:<VERCEL_PROJECT_NAME>:environment:*"
      }
    }
  }]
}
```

**Important**: The `environment:*` wildcard allows production, preview, and development environments. If you restrict to `environment:production` only, preview deployments (including v0 sandbox) will get OIDC auth failures even if the env vars are set.

```bash
aws iam create-role \
  --role-name vercel-aurora-data-api \
  --assume-role-policy-document file://trust-policy.json
```

To update an existing role's trust policy (if you already created it without the wildcard):

```bash
aws iam update-assume-role-policy \
  --role-name vercel-aurora-data-api \
  --policy-document file://trust-policy.json
```

**Step 4: Attach permissions policy**

Save as `permissions-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "rds-data:ExecuteStatement",
      "rds-data:BatchExecuteStatement",
      "rds-data:BeginTransaction",
      "rds-data:CommitTransaction",
      "rds-data:RollbackTransaction",
      "secretsmanager:GetSecretValue"
    ],
    "Resource": [
      "arn:aws:rds:<REGION>:<ACCOUNT_ID>:cluster:<CLUSTER_NAME>",
      "arn:aws:secretsmanager:<REGION>:<ACCOUNT_ID>:secret:<SECRET_NAME>*"
    ]
  }]
}
```

```bash
aws iam put-role-policy \
  --role-name vercel-aurora-data-api \
  --policy-name AuroraDataAPIAccess \
  --policy-document file://permissions-policy.json
```

**Step 5: Set Vercel environment variables (no secrets)**

Set these in **all three environments**: Production, Preview, and Development. In the Vercel Dashboard → Project → Settings → Environment Variables, use the checkboxes to select all three, or use the CLI: `vercel env add <NAME> production preview development`.

| Variable | Value |
| -------- | ----- |
| `AWS_REGION` | Aurora cluster region (e.g., `us-east-1`) — pin this explicitly, Vercel auto-sets it to function region otherwise |
| `AWS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/vercel-aurora-data-api` |
| `AURORA_CLUSTER_ARN` | `arn:aws:rds:<region>:<account>:cluster:<cluster-id>` |
| `AURORA_SECRET_ARN` | ARN of the Secrets Manager secret containing Aurora credentials |
| `AURORA_DATABASE` | Database name on the Aurora cluster |

**Verify all three environments have the variables set** before deploying to a preview branch. Preview deployments use the Preview environment, not Production.

No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` needed in production.

#### Application Code — Aurora Data API Client with OIDC

Install dependencies (detect package manager from lockfile — use `pnpm add` if `pnpm-lock.yaml` exists, `yarn add` if `yarn.lock` exists, otherwise `npm install`):

```bash
# npm
npm install @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider
# pnpm
pnpm add @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider
# yarn
yarn add @aws-sdk/client-rds-data @vercel/oidc-aws-credentials-provider
```

Client implementation:
```typescript
import { RDSDataClient, ExecuteStatementCommand } from "@aws-sdk/client-rds-data"
import { awsCredentialsProvider } from "@vercel/oidc-aws-credentials-provider"

const client = new RDSDataClient({
  region: process.env.AWS_REGION!,
  credentials: awsCredentialsProvider({
    roleArn: process.env.AWS_ROLE_ARN!,
  }),
})

const RESOURCE_ARN = process.env.AURORA_CLUSTER_ARN!
const SECRET_ARN = process.env.AURORA_SECRET_ARN!
const DATABASE = process.env.AURORA_DATABASE || "postgres"

export async function query(sql: string, parameters?: any[]) {
  const command = new ExecuteStatementCommand({
    resourceArn: RESOURCE_ARN,
    secretArn: SECRET_ARN,
    database: DATABASE,
    sql,
    includeResultMetadata: true,
    parameters,
  })
  return client.send(command)
}
```

Reference: [Vercel OIDC + AWS](https://examples.vercel.com/docs/oidc/aws) | [Vercel OIDC AWS Credentials Provider](https://www.npmjs.com/package/@vercel/oidc-aws-credentials-provider)

**Authentication**: Vercel OIDC provides short-lived credentials (minutes) that auto-rotate on every function invocation. No static keys, no manual rotation, full CloudTrail audit trail.

**Limitations**: Data API has a 45-second execution timeout per statement and 1 MB response size limit. For long-running queries or large result sets, use a direct connection via bastion/SSM instead.

**Pricing**: Data API is NOT free. It charges per API request:
- Pricing is tiered by volume (lower rate above 1 billion requests/month)
- Payloads metered in **32 KB increments** — a 64 KB payload counts as 2 requests
- **Free tier**: 1 million API requests/month, aggregated across regions, for the first year
- **Secrets Manager** is required (per-secret monthly fee + per-API-call fee)
- **MANDATORY**: Always query current Data API and Secrets Manager rates via the AWS Pricing MCP before presenting cost estimates:
  - Data API: `get_pricing` with `service_code="AmazonRDS"`, `region=<region>`, `filters=[{"Field": "usagetype", "Type": "CONTAINS", "Value": "Data-API-Usage"}]`
  - Secrets Manager: `get_pricing` with `service_code="AWSSecretsManager"`, `region=<region>`
- Do NOT hardcode prices or assume rates from memory — always query the Pricing MCP for current rates

**IMPORTANT**: Always include Data API request costs and Secrets Manager costs in the cost analysis when Data API is the connectivity method. Do NOT assume Data API is free.

### Aurora Data API — UUID Type Handling

When using the RDS Data API, UUID parameters require special handling. The Data API treats all string parameters as `text` type by default, which causes `operator does not exist: uuid = text` errors when comparing against UUID columns.

**Solution — Add `typeHint: "UUID"` to SqlParameter:**

```json
{
  "name": "id",
  "value": { "stringValue": "550e8400-e29b-41d4-a716-446655440000" },
  "typeHint": "UUID"
}
```

**Alternative — Cast in SQL:**

```sql
-- Cast column to text (may prevent index usage)
SELECT * FROM users WHERE id::text = :id

-- Cast parameter to uuid (preserves index usage)
SELECT * FROM users WHERE id = :id::uuid
```

The `typeHint` approach is preferred because it preserves index usage on UUID columns. Apply this to all queries that filter or join on UUID columns.

See `troubleshooting.md` → "UUID Type Mismatch" for additional details and resolution options.

### Alternative: AWS Amplify Hosting

If Vercel's lack of VPC access is a blocker, consider AWS Amplify Hosting as an alternative deployment platform. Amplify Hosting runs inside your AWS account and supports IAM compute roles for SSR apps, enabling secure access to AWS services from server-side code. It supports Next.js, React, and other frameworks with similar DX to Vercel (git-based deployments, preview environments, custom domains).

**Important**: Amplify Hosting does NOT deploy your application directly into a VPC. Amplify SSR uses Lambda functions behind the scenes. To access private VPC resources (e.g., Aurora in a private subnet), you have two options:

1. **VPC-enabled Lambda compute** — Configure the underlying SSR Lambda functions with VPC access so they can reach private resources like Aurora directly over the private network. This requires configuring security groups and VPC subnets for the Lambda functions.
2. **VPC endpoints** — Use VPC endpoints to establish private connectivity between Amplify's compute and VPC resources without exposing them to the public internet.
3. **Aurora Data API** — Access Aurora via HTTPS using IAM compute roles, without any VPC networking. Simplest option for CRUD workloads under 45 seconds per query.

For Aurora specifically, the Data API approach is simplest. For direct PostgreSQL wire-protocol access to a private Aurora instance, use VPC-enabled Lambda compute.

Reference: [re:Post — Is it possible to deploy AWS Amplify app in specific VPC?](https://repost.aws/questions/QUWYXTWYDEQ-2GrINlS_R2TQ/is-it-possible-to-deploy-aws-amplify-app-in-specific-vpc) | [Amplify Hosting IAM Compute Roles](https://docs.aws.amazon.com/amplify/latest/userguide/amplify-SSR-compute-role.html)

## Supabase DMS Configuration

When using AWS DMS with a Supabase source:

### Source Endpoint Extra Connection Attributes

```
pluginName=test_decoding;
```

Do NOT use `pglogical` — Supabase does not make the `pglogical` extension available. The `test_decoding` plugin is built into PostgreSQL and available on all Supabase instances. See the [AWS blog comparing test_decoding and pglogical](https://aws.amazon.com/blogs/database/comparison-of-test_decoding-and-pglogical-plugins-in-amazon-aurora-postgresql-for-data-migration-using-aws-dms/) for details on plugin differences.

### DMS Endpoint Credentials via Secrets Manager

Store DMS endpoint credentials in AWS Secrets Manager instead of passing plaintext username/password. This is the recommended approach for production migrations.

**Create a secret for the Supabase source endpoint:**

**IMPORTANT**: Do NOT pass passwords as plaintext in CLI commands. Use the secure input pattern (works in both bash and zsh):

```bash
echo -n "Enter Supabase DB password: " && read -s DB_PASS && echo && \
aws secretsmanager create-secret \
  --name dms/supabase-source-endpoint \
  --secret-string "{\"username\":\"postgres\",\"password\":\"$DB_PASS\",\"port\":5432,\"host\":\"db.<project-ref>.supabase.co\"}" \
  --region <region> && \
unset DB_PASS
```

Alternative: Create the secret via the AWS Console (Secrets Manager → Store a new secret → Other type of secret) to avoid CLI entirely.

**Create the DMS endpoint using Secrets Manager:**

```bash
aws dms create-endpoint \
  --endpoint-identifier supabase-source \
  --endpoint-type source \
  --engine-name postgres \
  --database-name postgres \
  --secrets-manager-secret-id arn:aws:secretsmanager:<region>:<account>:secret:dms/supabase-source-endpoint-<suffix> \
  --secrets-manager-access-role-arn arn:aws:iam::<account>:role/<dms-secrets-role> \
  --ssl-mode require \
  --extra-connection-attributes "pluginName=test_decoding;" \
  --region <region>
```

The IAM role (`dms-secrets-role`) needs `secretsmanager:GetSecretValue` permission on the secret ARN. DMS retrieves credentials at runtime — no plaintext passwords in endpoint configuration.

**IMPORTANT — IAM trust policy**: The DMS role must trust BOTH the global and regional DMS service principals:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": ["dms.amazonaws.com", "dms.<region>.amazonaws.com"]},
    "Action": "sts:AssumeRole"
  }]
}
```

**For the Aurora target endpoint** — use the same Secrets Manager pattern, and ALWAYS include:
- `"SslMode": "require"` — enforce SSL for data in transit
- `"AfterConnectScript": "SET session_replication_role = 'replica';"` — bypass FK checks during DMS load (see Decision 6 in `02-mobilize-phase.md`)

Example target endpoint JSON:
```json
{
  "EndpointIdentifier": "dms-ep-target-<name>",
  "EndpointType": "target",
  "EngineName": "postgres",
  "DatabaseName": "<database>",
  "SslMode": "require",
  "PostgreSQLSettings": {
    "SecretsManagerSecretId": "<target-secret-arn>",
    "SecretsManagerAccessRoleArn": "<dms-role-arn>",
    "AfterConnectScript": "SET session_replication_role = 'replica';"
  }
}
```

### Additional DMS Task Settings for Supabase

```json
{
  "SourceEndpoint": {
    "ExtraConnectionAttributes": "pluginName=test_decoding;"
  },
  "TaskSettings": {
    "ChangeProcessingTuning": {
      "HeartbeatEnable": true,
      "HeartbeatFrequency": 5,
      "HeartbeatSchema": "public"
    },
    "TargetMetadata": {
      "MapBooleanAsBoolean": true
    }
  }
}
```

- `HeartbeatEnable` — prevents replication slot from growing during idle periods
- `MapBooleanAsBoolean` — ensures PostgreSQL `boolean` maps correctly (DMS defaults to `varchar(5)`)

## Supabase Cost Comparison

Build a comparison table for the migration business case:

| Cost Component | Supabase (<plan>) | Aurora PostgreSQL |
|---------------|-------------------|-------------------|
| Compute | $<supabase-monthly> | $<aurora-instance-monthly> |
| Storage | Included / $<extra> | $<per-gb-month> × <size-gb> |
| I/O | Included | $<per-million-io> × <estimated-io> |
| Backup | Included | $<backup-storage> |
| Data Transfer | Included | $<egress-estimate> |
| Data API (if used) | N/A (PostgREST included) | Query via Pricing MCP: service_code="AmazonRDS", filter usagetype CONTAINS "Data-API-Usage" × <estimated-requests> (free tier: 1M/month year 1) |
| Secrets Manager (if used) | N/A | Query via Pricing MCP: service_code="AWSSecretsManager" (per-secret/month + per-API-call) |
| Auth (users) | Included | Cognito: $<per-mau> × <mau> |
| Storage (files) | Included | S3: $<per-gb> × <size-gb> |
| Realtime | Included | AppSync: $<per-connection-minute> |
| Edge Functions | Included | Lambda: $<per-invocation> × <invocations> |
| Total Monthly | $<total> | $<total> |

Use AWS Pricing MCP to fill in Aurora, Cognito, S3, AppSync, and Lambda costs for the specific region. Obtain Supabase costs from the user's billing dashboard or the [Supabase pricing page](https://supabase.com/pricing).

### Key Cost Considerations

- Supabase bundles compute, storage, auth, and realtime into a single plan — Aurora pricing is per-service
- Aurora may be cheaper for compute-heavy workloads but more expensive when factoring in all replacement services
- Graviton instances (db.r7g) and Reserved Instances can significantly reduce Aurora compute costs
- Consider Serverless v2 for variable workloads to avoid paying for idle capacity
- **Data API is NOT free** — when using Data API (e.g., for Vercel/Netlify connectivity), query current rates via Pricing MCP (`service_code="AmazonRDS"`, filter `usagetype CONTAINS "Data-API-Usage"` for Data API; `service_code="AWSSecretsManager"` for Secrets Manager) and include both in the comparison. For low-traffic apps the cost is minimal, but for high-traffic apps it can be significant
- Factor in operational overhead — Supabase is fully managed; Aurora requires more operational investment
