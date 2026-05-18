# Migrations: forward-only, source-of-truth in repo, applied via CLI

Migration files live in `migrations/NNNN_slug.sql` under the application repo — they are the source of truth. The `_ntts.schema_migrations` table tracks applied state only. There are no down-migrations: a wrong migration is corrected by writing a new forward migration. Migrations are applied via `ntts db migrate` (CLI) or the equivalent Studio button that calls the same control-plane API.

## Considered Options

- **Up/down pairs (the original PRD wording).** Rejected: down-migrations destroy data, are rarely correct in production, and encourage Operators to think "I can always roll back" — a false comfort.
- **State-driven (declare desired schema; tool computes diff).** Rejected: opaque, hard to review in PRs, mismatch with Supabase ecosystem expectations.
- **Forward-only files + state table (chosen).** Linear, audit-friendly, matches Supabase's actual default, leaves room for explicit "fix-forward" repair migrations.

## Mechanism

- Files in `migrations/NNNN_slug.sql`, sortable by `NNNN`.
- Each Migration runs in a transaction by default. The `-- ntts:no-transaction` header pragma opts out (for `CREATE INDEX CONCURRENTLY` and similar).
- F-FN-12 schema-compat-check runs **inside** the migration transaction. Compat failure → automatic rollback + "blocked by Functions: [list]" report.
- After successful COMMIT, control-plane fires `NOTIFY pgrst, 'reload schema'`, regenerates types into `_ntts.generated_types`, fires `NOTIFY ntts_types_change`.
- File-hash drift detection: if an already-applied migration's file content diverges from `schema_migrations.hash`, CLI warns at next run.

## Consequences

- Operators who need "rollback" write a new migration. This is more work but produces a real, auditable history.
- Schema-drift (manual `ALTER TABLE` in `psql`) is detectable via `ntts db status` comparing a structural hash, but not preventable. The appliance trust model accepts that an Operator-with-shell can do anything; the tool surfaces the drift instead of hiding it.
- The 1-click "sync affected Functions" UX is a coordinator (open Functions in editor + re-deploy queue), not a code-rewriter. Source code changes are always human-authored.
