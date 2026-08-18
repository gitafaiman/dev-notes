# Supabase Migration Workflow

How a schema change moves from a feature branch to prod in this repo. Written against the actual setup in `essgourmet`: two hosted Supabase projects (dev + prod), the CLI installed as a local devDependency, and hand-maintained TypeScript types (no `supabase gen types`).

## The two projects

| Project | ref | Name | Pointed to by |
|---|---|---|---|
| Dev | `bguofrerlufptqwucwqt` | ESS Gourmet Dev | `.env.development.local` (used by `next dev`) |
| Prod | `iclvauzsmdleqcysnali` | Ess Gourmet Catering | `.env.local` (used by `next build`/`next start`) |

The Supabase CLI only ever talks to one linked project at a time, tracked in `supabase/.temp/linked-project.json`. That file is local to your machine, persists between sessions, and is not something the app reads — only the CLI. There's no ambient indicator of which one you're linked to beyond checking that file, so the #1 rule of this whole workflow is: **know which project you're linked to before you run `db push`**.

The CLI itself is a project devDependency, not a global install, so every command below is prefixed `npx`.

## Step by step

### 1. Branch

```bash
git checkout -b feat/your-change main
```

Nothing schema-specific here — just don't write migrations directly on `main`.

### 2. Write the migration

Add a new file to `supabase/migrations/`, named `<UTC timestamp>_<description>.sql`:

```bash
date -u +%Y%m%d%H%M%S
# 20260818130905
```

```
supabase/migrations/20260818130905_wholesale_order_rush_fee.sql
```

The timestamp is the ordering key `db push` applies migrations in, across every branch that eventually merges to `main` — so if your branch sits open for a while, regenerate the timestamp right before you merge rather than reusing whatever you picked on day one.

Every migration in this repo follows the same shape — a header comment explaining why, wrapped in an explicit transaction:

```sql
-- ============================================================================
-- <table>: <what this migration does>
-- ============================================================================
-- <why -- the reasoning a future reader won't get from the diff alone>
-- ============================================================================
BEGIN;

ALTER TABLE public.wholesale_orders
  ADD COLUMN rush_fee NUMERIC CHECK (rush_fee IS NULL OR rush_fee >= 0);

COMMIT;
```

A few rules that have already bitten this codebase once each:

- **New table → RLS on, policies written immediately.** Every existing table enables `ROW LEVEL SECURITY` and adds `org_id`-scoped policies in the same migration that creates it. A table without RLS is a bug, not a follow-up.
- **Never drop a column before its replacement is backfilled.** When retiring a column in favor of a new table/column, order the migration *create new home for the data → backfill into it → drop the old column*, all three in one migration if the table is live. This repo shipped the drop-before-backfill version once and had to correct it in a follow-up commit.
- **Additive over destructive.** Prefer a new nullable column to rewriting an existing one. If a rename or type change is unavoidable, do it in stages across migrations rather than one irreversible `ALTER`.
- If PostgREST doesn't seem to see your change, that's a known intermittent issue in this project (see `20260813235651_reload_postgrest_schema_cache.sql`) — a follow-up migration with just `NOTIFY pgrst, 'reload schema';` fixes it.

### 3. Update the application code to match

This repo hand-maintains `lib/supabase/types.ts` — there's no `supabase gen types typescript` step. After writing the migration, update by hand, in this order:

1. `lib/supabase/types.ts` — the `Database["public"]["Tables"][...]` Row/Insert/Update shapes, and the app-facing type (e.g. `WholesaleOrder`) if the new column should be exposed.
2. `lib/supabase/mappers.ts` — map the new snake_case column onto the camelCase app type.
3. The `lib/actions/*.ts` server action(s) that read/write that table — add the column to any explicit `select()` string and to insert/update payloads.
4. Anywhere else that hand-lists columns (data loaders under `lib/data/`, bootstrap loaders) — grep the column name from step 2 across the repo; it's easy to update the "main" action file and miss a second `select` elsewhere.

### 4. Apply the migration to dev and verify

```bash
npx supabase link --project-ref bguofrerlufptqwucwqt   # prompts for the dev DB password
npx supabase migration list                              # compare local vs. remote history
npx supabase db push --dry-run                            # preview what would run
npx supabase db push                                      # apply it
```

Then run the app against dev (`next dev` reads `.env.development.local` automatically) and actually exercise the changed flow — don't just trust that the push succeeded.

### 5. Commit migration + code together

```bash
git add supabase/migrations/ lib/ components/ ...
git commit -m "feat: add rush fee to wholesale orders"
```

The SQL and the TypeScript that depends on it are one change, not two — a PR that adds a column without the code that reads it (or vice versa) is a broken intermediate state if anyone else's branch merges in between.

### 6. Open a PR, get it reviewed, merge to main

Same as any other change. Nothing migration-specific here beyond: the reviewer should be able to see the migration file's header comment and understand *why* without reading the rest of the diff.

### 7. Apply to prod

Only after the PR is merged — never push a migration to prod from a feature branch, even if it's "just" been tested on dev.

```bash
npx supabase link --project-ref iclvauzsmdleqcysnali   # prompts for the prod DB password
npx supabase migration list                              # confirm what's actually pending on prod
npx supabase db push --dry-run                            # preview
npx supabase db push                                      # apply
```

Verify against the real app afterward (or have whoever deploys do so) — a migration that ran clean on dev can still fail on prod if prod's data doesn't match the assumptions the migration made (a backfill query is the usual culprit).

### 8. Relink back to dev

```bash
npx supabase link --project-ref bguofrerlufptqwucwqt
```

Do this immediately, not "next time you need it" — leaving the CLI linked to prod is exactly how a routine `db push` during the next feature ends up hitting prod by accident. Dev is the resting state; prod is somewhere you visit deliberately and leave.

## Quick reference

| Command | What it does |
|---|---|
| `npx supabase link --project-ref <ref>` | Point the CLI at a project (asks for its DB password) |
| `npx supabase migration list` | Show which migrations are applied locally vs. on the linked project |
| `npx supabase db push --dry-run` | Preview pending migrations without applying |
| `npx supabase db push` | Apply all migrations not yet in the linked project's history |
| `date -u +%Y%m%d%H%M%S` | Generate a migration filename timestamp |

## Checklist

- [ ] Migration wrapped in `BEGIN;` / `COMMIT;`, with a header comment explaining why
- [ ] New tables have RLS enabled and `org_id`-scoped policies
- [ ] Column drops are preceded by a backfill into their replacement, same migration
- [ ] `lib/supabase/types.ts`, `mappers.ts`, and every relevant `select()` updated to match
- [ ] Verified against dev before merging
- [ ] Migration + code changes committed together, reviewed as one PR
- [ ] Applied to prod only after merge, from `main`
- [ ] CLI relinked back to dev immediately after the prod push

## Related

- `supabase/pgrst303-jwt-issued-at-future.md`
