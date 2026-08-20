---
inclusion: manual
---

# Agent Behavioral Rules — Supabase to Aurora PostgreSQL Migration

These rules apply to ALL phases of the migration. Violating any of these rules degrades the user experience and can cause security incidents.

## Security & Credentials

1. **NEVER display passwords or secrets in terminal output.** Retrieve credentials from Secrets Manager inline. See `supabase-prerequisites-and-connectivity.md` for patterns.
2. **NEVER ask the user to paste secrets into chat.** Use `echo -n` + `read -s` + `aws secretsmanager create-secret` pattern (works in bash and zsh). Or direct user to AWS Console. See `supabase-prerequisites-and-connectivity.md` for exact commands.
3. **NEVER ask for AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY.** Use IAM roles and AWS profiles.
4. **NEVER use static AWS credentials for deployment platforms (Vercel, Netlify, Amplify).** Always use OIDC federation for short-lived, auto-rotating credentials. See `supabase-migration-feature-mapping.md` → "Vercel OIDC Setup" for the complete setup. Static keys are a security risk — they are long-lived, can be leaked, and lack automatic rotation.
5. **NEVER recommend public database access.** Use private endpoints + Data API (for Vercel) or VPC connectivity. Data API has per-request pricing — query via Pricing MCP before presenting costs.
6. **Use Secrets Manager for all DMS endpoint credentials.** See `supabase-migration-feature-mapping.md` for configuration.

## User Interaction

1. **Ask user permission before ANY write operation.** Before creating, modifying, or deleting any AWS resource or database object, explain what you're about to do and why, then wait for confirmation. Example:

   ```text
   "I need to create a DMS replication instance (dms.r5.large, 100 GB storage) in your VPC
   to migrate data from Supabase to Aurora. This will cost approximately $X/hour while running.
   Shall I proceed?"
   ```

2. **Default to read-only Supabase access.** The Supabase MCP server should run in read-only mode. For any write operations on Supabase (creating publications, replication slots, schema changes), provide the exact SQL or CLI command for the user to run manually, explain what it does, and verify the result with a read-only query after the user confirms completion.

## MCP Server Usage

1. **Auto-authenticate the Supabase MCP server immediately.** At the start of the migration, attempt to connect to the Supabase MCP server and run a basic discovery query (e.g., `SELECT version()`) to confirm authentication is working. Do not wait for the user to manually trigger authentication — prompt them to complete the OAuth flow if needed.

2. **Use AWS Knowledge MCP to check official documentation before making assumptions.** When unsure about Aurora/RDS feature support, extension availability, parameter defaults, or DMS behavior, search the documentation first:

   ```text
   Use aws-knowledge-mcp search_documentation with query "Aurora PostgreSQL <topic>"
   ```

3. **Use web search tools to confirm from official guides.** When AWS Knowledge MCP doesn't have the answer, use `remote_web_search` and `webFetch` to check official AWS documentation, blog posts, and re:Post articles before providing guidance.

## Command Execution

1. **Write large JSON, SQL, or configuration to files** instead of passing inline in shell commands (avoids quoting issues with DMS task settings, SSM payloads, etc.).
2. **Use base64 encoding for SSM commands** that contain complex scripts — avoids shell escaping issues.

## Phase Execution — MANDATORY Checkpoints

These rules prevent the agent from skipping guidance that exists in the steering files.

1. **Read the relevant steering file BEFORE starting each phase.** Specifically:
   - Before Assess: read `01-assess-phase.md` (focus on: Migration Decision Tree, Cost Analysis)
   - Before Mobilize: read `02-mobilize-phase.md` → Decision 0 (Network Assessment) FIRST, then Decisions 1-11 as needed. Also read `supabase-migration-strategy-reference.md` for path selection.
   - Before Migrate: read `03-migrate-phase.md` (focus on: the selected Path, Cutover Procedure)
   - Before troubleshooting: read `troubleshooting.md` (match error to known pattern)
   - For Supabase connectivity: read `supabase-prerequisites-and-connectivity.md` → "Aurora Outbound IPv6" section
   - Before Modernize: read `04-modernize-phase.md`
   - Before troubleshooting any error: read `troubleshooting.md`

2. **Present the recommended approach BEFORE accepting user overrides.** If the user requests a specific migration path (e.g., "use DMS"), check the strategy reference first. If the power recommends a different path, explain why and confirm:

   ```text
   "The recommended approach for your case is [X] because [reasons].
   You asked for [Y] — that's the fallback option, typically used when [conditions].
   Would you like to proceed with [X] (recommended) or [Y] (your preference)?"
   ```

   Only proceed with the non-recommended path after the user explicitly confirms.

3. **Follow the migration path decision tree — do not skip it.** The decision tree in `supabase-migration-strategy-reference.md` says: "Try native PostgreSQL logical replication first. Use DMS as a fallback." This is not a suggestion — it is the default path. DMS should only be used when native replication fails or when built-in validation is needed for large databases (> 100 GB).

4. **When errors occur, check logs BEFORE guessing.** Do not assume the root cause based on table names or patterns. Follow the troubleshooting workflow in `troubleshooting.md`:
   - Step 1: Check DMS task status and stop reason
   - Step 2: Search CloudWatch Logs for `]E:` errors
   - Step 3: Correlate with Aurora PostgreSQL logs
   - Step 4: Match the error to a known pattern in `troubleshooting.md`

   Only after reading the actual error message should you propose a fix.

5. **When creating AWS resources, read the prerequisites steering file for the correct settings.** Do not rely on general knowledge for DMS instance configuration, endpoint settings, or security group rules. The `supabase-prerequisites-and-connectivity.md` file has the exact commands and settings for each resource. Key examples:
   - DMS replication instance: must be `--publicly-accessible` for external sources (Supabase)
   - DMS target endpoint: must include `AfterConnectScript` for FK bypass when schema has FKs
   - Both endpoints: must use `SslMode: require`

6. **ALWAYS run a network assessment before creating Aurora, DMS, or any VPC resources.** Follow Decision 0 in `02-mobilize-phase.md`. This includes: detecting source IPv6/IPv4 status, assessing VPC subnets, calculating CIDR blocks, and presenting the plan to the user for approval. Never provision infrastructure without this assessment.

7. **Aurora CANNOT make outbound IPv6 connections.** Do not attempt EIGW-based native replication from IPv6-only sources. This is a confirmed limitation (VPC flow logs show SYN packets leave Aurora's ENI but no SYN-ACK returns, while EC2 in the same subnet works). For online migration from IPv6-only sources, use DMS (publicly-accessible, dual-stack) or a bastion for pg_dump.

8. **ALWAYS create a migration branch before modifying application code.** Never modify the main/production branch directly during migration. Deploy the migration branch to a preview/staging environment for testing before cutover.

   **CHECKPOINT — Before writing ANY application code:**
   1. Read `03-migrate-phase.md` → "Application Code Refactoring & Testing" section
   2. Read `supabase-migration-feature-mapping.md` → "Application Code Refactoring" and "Vercel + Aurora Connectivity" sections
   3. Create the migration branch (`git checkout -b migration/aurora-refactor`)
   4. Only THEN begin modifying application files

9. **ALWAYS calculate and verify subnet CIDRs before creating subnets.** List all existing subnets, identify used CIDR ranges, calculate available blocks, and present proposed CIDRs to the user and must ask for user's consent. Never create subnets without verifying no conflicts exist.

10. **When DMS is needed, create all resources with correct settings on the first attempt.** Read the DMS Creation Checklist in `02-mobilize-phase.md` before creating any DMS resource. Key settings that cannot be changed after creation: `--publicly-accessible`, `--network-type`. Settings that cause FK errors if missing: `AfterConnectScript` on target endpoint. IAM trust policy must include both `dms.amazonaws.com` and `dms.<region>.amazonaws.com`.
