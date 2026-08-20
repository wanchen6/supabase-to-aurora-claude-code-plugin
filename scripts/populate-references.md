# Populating Reference Files

The reference files in `skills/supabase-to-aurora/references/` must contain the exact
content from the Kiro power's steering files. To populate them:

## Option 1: Using Kiro (recommended)

In a Kiro session with the `supabase-to-aurora-postgresql` power activated, run:

```
For each steering file listed below, read it from the power using `readSteering` 
and write the exact content (minus the YAML frontmatter) to the corresponding file 
in `skills/supabase-to-aurora/references/`:

1. 01-assess-phase.md
2. 02-mobilize-phase.md
3. 03-migrate-phase.md
4. 04-modernize-phase.md
5. agent-behavioral-rules.md (already done)
6. aurora-capabilities-reference.md
7. aurora-pg-best-practices.md
8. auth-comparison-supabase-vs-aurora.md
9. database-operations-reference.md
10. postgresql-migration-overview.md
11. supabase-migration-feature-mapping.md
12. supabase-migration-strategy-reference.md
13. supabase-prerequisites-and-connectivity.md
14. troubleshooting.md
```

## Option 2: From the Kiro power source

If you have access to the power's source directory, copy the steering files directly:

```bash
POWER_DIR="<path-to-kiro-power-steering-files>"
REFS="skills/supabase-to-aurora/references"

for f in 01-assess-phase.md 02-mobilize-phase.md 03-migrate-phase.md \
         04-modernize-phase.md agent-behavioral-rules.md \
         aurora-capabilities-reference.md aurora-pg-best-practices.md \
         auth-comparison-supabase-vs-aurora.md database-operations-reference.md \
         postgresql-migration-overview.md supabase-migration-feature-mapping.md \
         supabase-migration-strategy-reference.md \
         supabase-prerequisites-and-connectivity.md troubleshooting.md; do
  # Strip YAML frontmatter (between first two --- lines) if present
  sed '1{/^---$/!q;};1,/^---$/d' "$POWER_DIR/$f" > "$REFS/$f"
done
```
