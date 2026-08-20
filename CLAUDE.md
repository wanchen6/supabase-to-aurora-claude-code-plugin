# Supabase to Aurora PostgreSQL Migration Plugin v1.0.0

Installed at: `~/.claude/plugins/supabase-to-aurora/`

This plugin automates migrating PostgreSQL databases from Supabase to Amazon Aurora PostgreSQL. It covers the full migration lifecycle — assessment, target provisioning, data migration (offline and online), validation, application cutover, and post-migration optimization.

## MCP Servers (3)

| Server | Purpose |
|--------|---------|
| `supabase` | Source database discovery and assessment (OAuth) |
| `awslabs-postgres-mcp-server` | Aurora PostgreSQL target operations (create cluster, run queries) |
| `aws-pricing-mcp-server` | Cost analysis and pricing comparison |

## Available Skills (1)

| Skill | Description |
|-------|-------------|
| **supabase-to-aurora** | Full migration workflow: assess source, provision Aurora target, migrate data (pg_dump, native replication, or DMS), refactor application code, validate, and cut over |

## Usage

The skill activates automatically when you mention migrating from Supabase to Aurora, or invoke it directly:

```
/supabase-to-aurora:supabase-to-aurora
```

Or simply describe what you want:

- "I want to migrate my Supabase database to Aurora PostgreSQL"
- "Help me move from Supabase to AWS"
- "Migrate my database from Supabase"

## Prerequisites

- AWS CLI configured (`aws sts get-caller-identity` should succeed)
- `uvx` installed (for local MCP servers): `pip install uv` or `brew install uv`
- Supabase project URL (you'll complete OAuth during first use)
