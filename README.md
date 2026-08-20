# Supabase to Aurora PostgreSQL Migration — Claude Code Plugin

Migrate PostgreSQL databases from Supabase to Amazon Aurora PostgreSQL with guided automation. This plugin bundles migration skills, reference documentation, and MCP server connections into a single installable package for Claude Code.

## What's Included

- **Skill**: Full migration workflow (assess → mobilize → migrate → modernize)
- **MCP Servers**: Supabase (OAuth), Aurora PostgreSQL (pgwire/Data API), AWS Pricing
- **14 Reference Documents**: Decision trees, phase guides, troubleshooting, best practices

## Prerequisites

- **AWS CLI** configured and authenticated (`aws sts get-caller-identity` should succeed)
- **uv** installed (for local MCP servers): `pip install uv` or `brew install uv`
- **Supabase project** — you'll need the project URL; OAuth completes during first use

## Installation

### Option 1: Git Clone (recommended)

```bash
git clone https://github.com/<your-org>/supabase-to-aurora-plugin.git \
  ~/.claude/plugins/supabase-to-aurora
```

### Option 2: Manual Copy

```bash
mkdir -p ~/.claude/plugins/supabase-to-aurora
cp -r . ~/.claude/plugins/supabase-to-aurora/
```

### Option 3: Install Script

```bash
curl -fsSL https://raw.githubusercontent.com/<your-org>/supabase-to-aurora-plugin/main/install.sh | bash
```

## Post-Installation

1. **Restart Claude Code** — plugins load at startup
2. **Verify installation** — ask Claude: "What plugins are installed?"
3. **Complete Supabase OAuth** — on first use, the Supabase MCP server opens a browser for authentication
4. **Verify AWS credentials** — ensure your default AWS profile has access to RDS, Secrets Manager, and Pricing APIs

## Usage

The skill activates automatically when you mention migrating from Supabase to Aurora. You can also invoke it directly:

```
/supabase-to-aurora:supabase-to-aurora
```

Or describe what you want:

- "I want to migrate my Supabase database to Aurora PostgreSQL"
- "Help me move from Supabase to AWS"
- "Migrate my database from Supabase"

## Migration Paths Supported

| Path | When | Downtime |
|------|------|----------|
| pg_dump/pg_restore | < 100 GB, downtime OK | Hours |
| Native logical replication | Zero-downtime, tables have PKs | Minutes (cutover only) |
| AWS DMS with CDC | Validation needed, or native replication fails | Minutes (cutover only) |
| Hybrid | Mixed requirements | Minutes + brief window |

## Plugin Structure

```
supabase-to-aurora/
├── .claude-plugin/
│   └── plugin.json              # Metadata + MCP server declarations
├── skills/
│   └── supabase-to-aurora/
│       ├── SKILL.md             # Main migration workflow
│       └── references/          # 14 detailed reference documents
│           ├── 01-assess-phase.md
│           ├── 02-mobilize-phase.md
│           ├── 03-migrate-phase.md
│           ├── 04-modernize-phase.md
│           ├── agent-behavioral-rules.md
│           ├── aurora-capabilities-reference.md
│           ├── aurora-pg-best-practices.md
│           ├── auth-comparison-supabase-vs-aurora.md
│           ├── database-operations-reference.md
│           ├── postgresql-migration-overview.md
│           ├── supabase-migration-feature-mapping.md
│           ├── supabase-migration-strategy-reference.md
│           ├── supabase-prerequisites-and-connectivity.md
│           └── troubleshooting.md
├── CLAUDE.md                    # Plugin manifest (loaded every session)
└── README.md                    # This file
```

## MCP Server Configuration

The plugin declares three MCP servers in `plugin.json`. They start automatically when the plugin loads:

| Server | Type | Purpose |
|--------|------|---------|
| `supabase` | Remote (HTTP/OAuth) | Source database discovery and assessment |
| `awslabs-postgres-mcp-server` | Local (uvx) | Aurora target operations |
| `aws-pricing-mcp-server` | Local (uvx) | Cost analysis and pricing comparison |

### AWS Profile Configuration

The Aurora and Pricing MCP servers use the default AWS credential chain. If you need a non-default profile, add environment variables to the MCP server config in `.claude-plugin/plugin.json`:

```json
"env": {
  "AWS_PROFILE": "your-profile-name",
  "AWS_REGION": "us-east-1"
}
```

## Updating

```bash
cd ~/.claude/plugins/supabase-to-aurora
git pull
```

Restart Claude Code after updating.

## Uninstalling

```bash
rm -rf ~/.claude/plugins/supabase-to-aurora
```

No registry or config to clean — the directory IS the installation.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Plugin not recognized | Missing plugin.json | Verify `.claude-plugin/plugin.json` exists |
| Skill never activates | Description keywords not matching | Try invoking directly with `/supabase-to-aurora:supabase-to-aurora` |
| Supabase MCP fails | OAuth not completed | Complete the browser auth flow when prompted |
| Aurora MCP fails | AWS credentials not configured | Run `aws sts get-caller-identity` to verify |
| `uvx` not found | uv not installed | Run `pip install uv` or `brew install uv` |

## License

MIT
