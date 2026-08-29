---
name: tailor-safe-schema-migration
description: Change a TailorDB schema and deploy it without losing data, using Tailor's published migration commands and failure-recovery semantics.
api: Tailor Platform
operations:
  - ComposeTailorDBSDL
  - CreateTailorDBType
  - UpdateTailorDBType
  - GetTailorDBType
  - ListTailorDBTypes
  - GetApplicationSchemaHealth
generated: '2026-08-29'
method: generated
source: https://docs.tailor.tech/sdk/services/tailordb-migration
---

# Change a TailorDB schema safely

TailorDB generates the whole GraphQL API from the schema, so a schema change is an API
change. This flow is the published safe path.

## Steps

1. **Edit the table definition** in your project's `db/` directory
   (`db.table("Task", { … })`).

2. **Generate the migration.**
   `tailor tailordb migration generate` — detects the difference between the local
   tables and the previous migration snapshot.

3. **Validate without deploying.**
   `tailor tailordb migration validate` — runs the full migration-history check,
   flags unreviewed generated scripts, and detects schema drift both ways (local vs
   snapshot, remote vs checkpoint). It runs the same checks `deploy` runs and exits
   non-zero on issues. **This is the rehearsal step. Do not skip it.**

4. **Test against real-shaped data.**
   `tailor tailordb migration test` — runs pending migrations with seed fixtures or
   cloned data in a **temporary** workspace.

5. **Check status.**
   `tailor tailordb migration status` — shows applied and pending migrations per
   namespace.

6. **Deploy.**
   `npm run deploy -- --workspace-id <id>`.

7. **Verify.** Call `GetApplicationSchemaHealth`, then open the GraphQL Playground and
   run a query against the changed type.

## What happens when it fails

Tailor's documented failure recovery, verbatim in substance:

- The transaction rolls back for that migration's script; database changes it made are
  undone.
- Pre-migration schema changes roll back to the prior checkpoint — pre-existing tables
  are restored to their previous shape, newly introduced tables are dropped.
- The workspace is left at its prior checkpoint and prior schema, **not half-applied**.
- The `apply` aborts and the checkpoint label is not bumped.

## What is NOT reversible

- A migration that **succeeded**. There is no operator-initiated rollback.
- Reverting the config commit and redeploying rolls back configuration only — the docs
  are explicit: *"Schema and data are not rolled back."*
- `tailor tailordb truncate` / the `TruncateTailorDBType` RPC. No undo exists.

Take a data copy first if the migration is destructive. `CloneApplicationData` copies
application data into another application.

## Concurrency warning

There is no migration locking. Do not run migrations in parallel against the same
workspace — serialize deploys per environment.
