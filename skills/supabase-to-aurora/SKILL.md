---
name: Supabase to Aurora Migration
description: >
  Migrates PostgreSQL databases from Supabase to Amazon Aurora PostgreSQL.
  Covers the full migration lifecycle: assessment, target provisioning, data migration
  (offline pg_dump, native logical replication, or AWS DMS with CDC), application code
  refactoring (Supabase SDK to direct PostgreSQL), validation, cutover, and post-migration
  optimization. Supports three migration paths with automatic path selection based on
  database size, downtime tolerance, and table primary key status.
when_to_use: >
  When the user wants to migrate from Supabase to Aurora, move their database from
  Supabase to AWS, migrate from Supabase to PostgreSQL on AWS, or asks about database
  migration from Supabase. Also applies when refactoring Supabase SDK code to use direct
  PostgreSQL connections, setting up Aurora Data API with Vercel OIDC, or replacing
  Supabase Auth/Storage/Realtime with AWS equivalents.
allowed-tools: Bash, mcp__supabase__*, mcp__awslabs-postgres-mcp-server__*, mcp__aws-pricing-mcp-server__*
---

# Supabase to Aurora PostgreSQL Migration

## Overview

This skill automates migrating a PostgreSQL database from Supabase to Amazon Aurora PostgreSQL. It covers the full migration lifecycle — assessment, target provisioning, data migration (offline and online), validation, application cutover, and post-migration optimization.

Three migration paths are supported:

1. **Offline**: pg_dump/pg_restore — simple, requires downtime
2. **Online (preferred)**: PostgreSQL native logical replication — zero-downtime CDC
3. **Online (fallback)**: AWS DMS with CDC and built-in validation — for tables where native replication doesn't work

**Rule of thumb**: Try native PostgreSQL logical replication first. If it doesn't work for a specific table (e.g., missing primary key, unsupported data type), use DMS for that table.

## Migration Paths

**Fast Path** (database < 1 GB AND brief downtime acceptable):
- Use `pg_dump` from source → `pg_restore` to Aurora (seconds of downtime)
- Still follow the branch/validation rules below

**Full Path** (database > 1 GB OR zero-downtime required):
- Follow the full Decision 0–11 workflow in `references/02-mobilize-phase.md`

| Path | When | Details In |
| ---- | ---- | ---------- |
| Offline (pg_dump/pg_restore) | Downtime OK, < 100 GB | `references/03-migrate-phase.md` → Path 1 |
| Native Logical Replication | Zero-downtime, source has IPv4 | `references/03-migrate-phase.md` → Path 2 |
| AWS DMS with CDC | Zero-downtime, IPv6-only source, or need validation | `references/03-migrate-phase.md` → Path 3 |
| Hybrid | Mixed requirements | `references/03-migrate-phase.md` → Path 4 |

**Key constraint**: Aurora cannot make outbound IPv6. See `references/supabase-prerequisites-and-connectivity.md` for connectivity options.

## Reference Files

Before starting each phase, read the relevant reference file:

- **Assessment**: @references/01-assess-phase.md — database discovery, compatibility, cost analysis
- **Mobilize**: @references/02-mobilize-phase.md — target provisioning, schema migration, DMS/replication setup
- **Migrate**: @references/03-migrate-phase.md — data migration execution, app refactoring, cutover
- **Modernize**: @references/04-modernize-phase.md — optimization, HA/DR, cost reduction
- **Behavioral rules**: @references/agent-behavioral-rules.md — mandatory security and interaction guardrails
- **Troubleshooting**: @references/troubleshooting.md — error patterns and resolutions
- **Strategy reference**: @references/supabase-migration-strategy-reference.md — decision trees and flow summaries
- **Prerequisites**: @references/supabase-prerequisites-and-connectivity.md — MCP setup, connectivity, IPv6
- **Feature mapping**: @references/supabase-migration-feature-mapping.md — SDK refactoring, Auth→Cognito, Vercel OIDC
- **Auth comparison**: @references/auth-comparison-supabase-vs-aurora.md — RLS rewriting, key patterns
- **Aurora capabilities**: @references/aurora-capabilities-reference.md — value proposition for cost justification
- **Aurora best practices**: @references/aurora-pg-best-practices.md — connection management, indexing, monitoring
- **DB operations**: @references/database-operations-reference.md — general PostgreSQL queries
- **Overview**: @references/postgresql-migration-overview.md — central index, naming conventions

## Agent Behavioral Rules

**CRITICAL**: Read `references/agent-behavioral-rules.md` before starting any phase. Key rules:

- NEVER ask for AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY — use IAM roles and OIDC
- NEVER display passwords or secrets in terminal output
- NEVER recommend public database access
- Ask user permission before ANY write operation on AWS resources
- Default to read-only Supabase access (provide write commands for user to run manually)
- Read the relevant steering file BEFORE starting each phase
- Create a migration branch before modifying application code
- Always calculate and verify subnet CIDRs before creating subnets

## MCP Servers

| Server | Purpose | Connection |
| ------ | ------- | ---------- |
| `supabase` | Source database discovery and assessment | OAuth — user selects project during setup |
| `awslabs-postgres-mcp-server` | Aurora PostgreSQL target operations | pgwire via AWS Secrets Manager |
| `aws-pricing-mcp-server` | Cost analysis and pricing comparison | stdio via uvx |

## Getting Started — MANDATORY Intake Flow

**Flow**: Step 0 → verify MCP servers | Step 1 → auto-discover source | Step 2 → infer app stack | Step 3 → propose target config | Step 4 → confirm plan with cost estimate → execute.

**CRITICAL: Complete Steps 0–4 before executing any migration. Auto-discover as much as possible via MCP. Only ask the user for information you genuinely cannot infer.**

### Step 0: Verify MCP Server Configuration

Before starting, verify both MCP servers are connected:

1. **Supabase MCP**: Call `list_projects` or `list_tables`. If it fails, prompt the user to complete OAuth in Claude Code's MCP panel.
2. **Aurora PostgreSQL MCP**: Call `get_database_connection_info`. If it fails, the user may need `AWS_PROFILE` and `AWS_REGION` in the MCP server env config.

Do NOT proceed until both servers are confirmed working.

### Step 1: Source Database Details (Supabase)

Ask only for the Supabase project URL if not provided. Then auto-discover via Supabase MCP:
- PG version, size, tables, row counts, indexes, extensions
- RLS policies, FKs, tables without PKs, data types
- Feature usage (auth.users, storage.objects, realtime, edge functions)
- IPv4/IPv6 status: `dig AAAA db.<project-ref>.supabase.co`

Present a single confirmation summary.

### Step 2: Application Details

Infer from context first, then ask only what you can't determine:
- If workspace has `package.json` → read it for framework and dependencies
- If Vercel URL provided → hosting is Vercel, likely Next.js
- If app uses `@supabase/supabase-js` → SDK refactoring needed

State what you've inferred and ask for confirmation — not a questionnaire.

### Step 3: Target Database Details (Aurora PostgreSQL)

Ask the user for:
- AWS Region
- Whether to use an existing Aurora cluster or create new

Propose with reasoning (derived from Step 1):
- Cluster identifier, database name, engine version
- Capacity model (Serverless v2 vs provisioned — see thresholds in `references/02-mobilize-phase.md`)

### Step 4: Confirm Migration Plan

**MANDATORY**: Complete cost analysis using the AWS Pricing MCP before presenting this summary. Query actual Aurora pricing for the proposed configuration.

Present summary:
```
Source: Supabase (<plan>) — <project-ref> — PG <version> — <size>
Target: Aurora PostgreSQL <version> — <cluster-id> — <region>
Migration Path: <offline | online-native | online-dms | hybrid>
Application: <framework> on <hosting>
Cost Estimate: Current $X/mo → Aurora $Y/mo (<option comparison>)
```

Ask: "Does this look correct? Ready to proceed?" Only proceed after explicit confirmation.

## Post-Migration

See `references/03-migrate-phase.md` → "Post-Migration Activities" and `references/04-modernize-phase.md`.
