# Auth Reference — Supabase vs Aurora PostgreSQL

Use this file when evaluating: (1) which Supabase auth patterns are safe to keep vs must be replaced, (2) Aurora database connection auth methods, (3) security risks of Supabase key patterns on Aurora.

For feature-by-feature migration steps (GoTrue → Cognito, Storage → S3, Realtime → AppSync, RLS policy rewriting), see `supabase-migration-feature-mapping.md`.

---

## Auth Architecture Difference

**Supabase**: Auth is a platform service (GoTrue). It issues JWTs, sets PostgreSQL session roles, and enables `auth.uid()` in RLS policies via PostgREST middleware.

**Aurora PostgreSQL**: No built-in auth service. App authenticates to PostgreSQL directly via password (Secrets Manager) or IAM token. User identity for RLS must be set explicitly as a session variable by the application.

### RLS: Supabase → Aurora Pattern Mapping

| Supabase pattern | Aurora equivalent |
| ---------------- | ----------------- |
| `auth.uid()` in policy | `current_setting('app.current_user_id', true)::uuid` |
| `auth.jwt()->>'role'` | `current_setting('app.current_role', true)` |
| Role `authenticated` | App sets session var after verifying Cognito/custom JWT |
| Role `anon` | N/A — Aurora is not publicly accessible; no PostgREST layer |
| `service_role` bypass | N/A — use least-privilege DB user |

**MANDATORY**: `auth.uid()` does not exist on Aurora. RLS policies using it fail silently (return no rows). Audit and rewrite before cutover:

```sql
SELECT schemaname, tablename, policyname, qual, with_check
FROM pg_policies
WHERE qual::text LIKE '%auth.uid()%'
   OR qual::text LIKE '%auth.jwt()%'
   OR with_check::text LIKE '%auth.uid()%';
```

Set session variable before each request (application server, after verifying JWT):

```sql
SET LOCAL app.current_user_id = '<verified-user-id>';
```

---

## When Supabase-Style Auth Should NOT Be Used on Aurora

### `service_role` key — never use equivalent

Bypasses ALL RLS. The Aurora equivalent would be using the `postgres` master user in application code — equally dangerous. Use a dedicated least-privilege DB user retrieved from Secrets Manager instead.

### `anon` key — not applicable

The `anon` key exists to identify callers to PostgREST (Supabase's public HTTP API). Aurora lives in a private VPC with no public API layer. Server-side apps connect via PostgreSQL wire protocol — no key needed.

### Static AWS credentials in Vercel

**NEVER** set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in Vercel env vars. Use Vercel OIDC federation — auto-rotating, scoped credentials per function invocation. See `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity".

---

## Aurora Database Authentication Methods

| Method | When | How |
| ------ | ---- | --- |
| Password (Secrets Manager) | Default | Retrieve at runtime: `aws secretsmanager get-secret-value` |
| IAM database auth | EC2/Lambda with IAM roles | `aws rds generate-db-auth-token`; grant `rds_iam` to DB user |
| Vercel OIDC + Data API | Vercel deployments | OIDC federation; no static credentials |

**Before creating any DMS endpoint**, resolve the correct secret from the cluster — do NOT scan Secrets Manager:

```bash
aws rds describe-db-clusters \
  --db-cluster-identifier <cluster-id> \
  --query 'DBClusters[0].{User:MasterUsername,SecretArn:MasterUserSecret.SecretArn}'
```

Confirm username matches before building the endpoint.

---

## Agent Decision Rules

1. App uses GoTrue sign-in → replace with Cognito. See `supabase-migration-feature-mapping.md` → "Supabase Auth → Amazon Cognito".
2. App has RLS policies → audit for `auth.uid()`. Rewrite to `current_setting('app.current_user_id')`. Application sets session var per request.
3. App is server-rendered (Next.js SSR, Express) → no `anon` key needed. Server connects to Aurora directly.
4. App is Vercel-hosted → Vercel OIDC + Aurora Data API. Never static keys.
5. App is database-only (no user accounts) → no auth service replacement needed. Update connection string only.
6. DMS target endpoint → always resolve secret from `describe-db-clusters`, not from a Secrets Manager list scan.
