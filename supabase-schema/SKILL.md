---
name: supabase-schema
description: Use this skill before writing any migration, before adding a column, and before approving any PR that changes the schema. Triggers on every schema decision in the Freedom Portal repo. The conventions here are not negotiable — re-litigating them per ticket wastes everyone's time.
---

# Supabase Schema — How Freedom's Database Is Shaped

This is the source of truth for *how* we write Postgres at Freedom. Every table, column, type, default, and index follows the rules below. If you find yourself wanting to break a rule, write an ADR before the migration.

The single most important fact: **migrations are forever.** A wrong column name lives in code, queries, types, dashboards, audit logs, and client backups for years. The cost of getting the schema right on day one is trivial compared to the cost of fixing it later. Slow down.

---

## Naming

### Tables

- `snake_case`, **plural** — `leads`, `proposals`, `audit_events`, `meeting_attendees`.
- No prefixes (`tbl_`, `freedom_`). The schema is the namespace.
- No abbreviations except universally-known ones (`url`, `id`, `pdf`).
- If the table is a join table, name it for what it represents, not the participants — `meeting_attendees` (not `meetings_users`), `proposal_mix` (not `proposals_products`).

### Columns

- `snake_case`. Plural only when the column genuinely holds an array or jsonb of items (`features text[]`, `holdings jsonb`).
- Foreign keys end in `_id` — `advisor_id`, `lead_id`. Never `advisor` or `advisorId`.
- Booleans phrased so that `true` is the affirmative state — `is_signed`, `is_esg`, `is_tax_deductible`. Avoid double-negatives like `is_not_archived`.
- Timestamps end in `_at` — `created_at`, `signed_at`, `deleted_at`. Always `timestamptz`.
- Counts end in `_count`. Amounts end in `_chf`/`_eur` when currency is fixed, otherwise `amount` + a separate `currency` column.

### Enums

- Postgres `enum` types, named singular: `lead_stage`, `proposal_status`, `actor_type`.
- Values are lowercase `snake_case`: `won`, `signed`, `proposal_signed`. Never UPPERCASE — that's reserved for audit-event `action` values which are not enums.
- Adding an enum value is a normal migration. Removing one requires an ADR (it's effectively a data-loss change for any row holding that value).

### Indexes & constraints

- Unique constraints get explicit names: `uq_advisors_email`, `uq_proposals_id_year_seq`.
- Indexes get explicit names: `ix_leads_advisor_id_score` (suffix is the columns in order).
- Foreign keys get explicit names: `fk_leads_advisor_id`.
- Check constraints get explicit names: `ck_proposal_mix_weight_range`.

This matters because Postgres' default names are unstable and break diffs.

---

## Types

### IDs

- **UUID v7** for every primary key. `uuid_generate_v7()` (after enabling `pgcrypto` and the v7 helper). Reasons:
  - Sortable by creation time → cheap pagination.
  - Globally unique → safe to expose to clients.
  - Doesn't leak row counts the way bigint sequences do.
- `uuid`, **never** `bigint`/`serial` for primary keys.
- Internal sequences (e.g. invoice numbers, proposal numbers) live in their own column alongside the UUID, format `P-YYYY-NNN`.

### Money

- `numeric(18,2)` for amounts. Never `float`/`real`/`double precision`.
- Currency in a separate `text` or enum column. Never inferred from context.
- Display formatting is the UI's job; never store formatted strings.

### Timestamps

- `timestamptz` (not `timestamp`). Always.
- Default to `now()` for `created_at` / `updated_at`.
- `updated_at` is maintained by a trigger (see "Triggers" below).
- `deleted_at` is `null` for live rows; setting it = soft delete.

### Strings

- `text`. Avoid `varchar(n)` — Postgres gains nothing from the length cap, you just get rigid migrations later.
- For known-finite sets, use enums.
- Trim user input at the application layer; don't rely on triggers.

### Booleans

- `boolean`. Defaulted to `false` unless there's a clear reason otherwise.

### JSON

- `jsonb`, never `json`.
- Reserve for genuinely shapeless data: prompt outputs, integration payloads, audit metadata. If a field appears in 50% of rows, lift it to a real column.
- Do not query inside `jsonb` in hot paths. If you find yourself doing it often, that field should be a column.

---

## Required columns on every table

Every table — except `audit_events` — has these columns. No exceptions:

```sql
id          uuid primary key default uuid_generate_v7(),
created_at  timestamptz not null default now(),
updated_at  timestamptz not null default now()
```

User-facing tables also have:

```sql
deleted_at  timestamptz
```

A row with `deleted_at != null` is dead to most queries. Use a partial index:

```sql
create index ix_leads_alive on leads (advisor_id, score desc) where deleted_at is null;
```

Most query helpers should have a `where deleted_at is null` baked in unless they explicitly want to look at the graveyard.

---

## `updated_at` trigger

Every table with `updated_at` gets this trigger:

```sql
create or replace function set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger set_updated_at_<table> before update on <table>
  for each row execute function set_updated_at();
```

Don't trust application code to remember.

---

## Foreign keys

Always explicit. Always with an action:

```sql
advisor_id uuid not null references auth.users(id) on delete restrict,
lead_id    uuid           references leads(id)    on delete set null,
proposal_id uuid not null references proposals(id) on delete cascade
```

Choose the action deliberately:

- `restrict` — the parent cannot be deleted while children exist. Default for advisor → anything.
- `cascade` — children die with the parent. Use when children are conceptually parts of the parent (e.g., `proposal_mix` rows of a `proposal`).
- `set null` — the relationship is "soft." Use when a child can outlive the parent (e.g., a meeting whose lead got archived).
- `no action` — never. Be explicit.

Add the foreign key in the same migration as the table. Don't ship a column called `lead_id` without the FK.

---

## Soft delete vs hard delete

The default is **soft delete** (`deleted_at = now()`).

Hard delete only when:
1. **GDPR erasure** — see `gdpr-erasure` skill. The `audit_events` row survives with PII redacted.
2. **Test fixture cleanup** — service-role only, never in app code.
3. **Compliance-mandated short retention** (none yet for the prototype).

Never hard-delete from app code. Period.

---

## Indexes

### When to add an index

- Every foreign key column gets an index. Always.
- Columns used in `WHERE` filters of hot queries.
- Columns used in `ORDER BY` of paginated queries.
- Composite indexes match query order: `ix_leads_advisor_score (advisor_id, score desc)` for "list this advisor's leads by score."

### When NOT to add an index

- Tiny tables (<1 000 rows). The planner ignores them anyway.
- Columns with low cardinality (booleans, simple enums) unless paired in a composite.
- Columns you "might query someday." Don't pre-optimize.

### Partial indexes

Use partial indexes for the live-row case (you'll do this often):

```sql
create index ix_proposals_alive on proposals (advisor_id, status, created_at desc)
  where deleted_at is null;
```

This keeps the index small and fast.

---

## Audit hook contract

If a table is **compliance-relevant**, every mutation must emit an `audit_events` row.

Compliance-relevant tables in Freedom (as of phase 1):

- `auth.users` — every sign-in, sign-out, password-related action
- `leads`, `clients`
- `proposals`, `proposal_mix`
- `meetings`, `meeting_notes`
- `documents`
- `commissions`
- `kyc_documents` (when added)
- `signatures` (when added)

The audit hook lives in `packages/db/queries/audit.ts` and is called from server actions, **not** from triggers. Triggers can't easily capture actor_id without ugly workarounds; server actions know the actor.

Audit-event `action` values are SCREAMING_SNAKE_CASE, past-tense: `LEAD_ACCEPTED`, `PROPOSAL_SIGNED`, `MEETING_ENDED`. Aria curates the canonical action vocabulary in the `audit-logging` skill.

---

## Multi-tenancy boundary

Every business table has a `advisor_id` column (or chains transitively to one through a fk). This is the tenancy boundary. RLS policies almost always read `auth.uid()` against this column. See `postgres-rls`.

There is **no shared "freedom-wide" data** that an advisor can edit — only the seed data (`products`, `templates`) which is service-role-managed and read-only for advisors.

If you find yourself adding a row that doesn't belong to a single advisor, stop and bring it to Aria. We probably need an ADR.

---

## Migrations

### `supabase/config.toml` — verify against the installed CLI before pushing

`supabase db push` validates `supabase/config.toml` against the schema of the **CLI version actually installed**, not against whatever a tutorial or an older project used. A stale key fails the push before a single migration runs. Don't hand-write `config.toml` from memory — generate it with `supabase init` on the current CLI, then diff against what you intended.

Known stale keys / values that have already bitten us (IAF-5, 2026-05) — do **not** carry these forward:

| Stale | Status | Correct |
|---|---|---|
| `auth.refresh_token_rotation_enabled = true` | Key not recognized by current CLI | Remove it. Rotation behavior is managed in the Supabase dashboard / newer config keys. |
| `auth.refresh_token_reuse_interval = 10` | Key not recognized by current CLI | Remove it. |
| `db.major_version = 15` | Postgres 15 is no longer the default | `db.major_version = 17` |

Before any migration ticket: run `supabase --version`, then `supabase db push --dry-run` (or the lint the CLI offers) and confirm `config.toml` parses clean. If a key you need genuinely isn't supported, that's a question for Aria — don't silently drop a security-relevant setting; confirm where its behavior moved.

### Naming & location

- `supabase/migrations/YYYYMMDDHHMMSS_<slug>.sql`
- One feature per migration. Don't pile schema changes from three tickets into one file.
- The slug describes the change: `20260508142000_create_leads.sql`, `20260512091500_add_score_to_leads.sql`.

### Reversibility

Every migration includes a `down` block as a comment at the top, even if you'll never run it:

```sql
-- DOWN:
--   drop table if exists leads cascade;
--   drop type if exists lead_stage;
```

This forces you to think about reversal. Most data-loss migrations get an ADR exactly because the down isn't truly reversible.

### Testing on a copy of staging

Before any non-trivial migration:

1. Snapshot staging with `supabase db dump`.
2. Restore the dump into a scratch project.
3. Run the migration against the scratch.
4. Verify row counts, run a representative query, check RLS still behaves.
5. Only then push the migration to staging.

For trivial migrations (adding a column with a default, creating an index `concurrently`), skip step 1-4. Use judgment; if in doubt, test.

### `concurrently`

Index creation and `validate constraint` operations should be `concurrently` whenever possible. They take longer but don't block writes. The migration runner has to handle this with care because `concurrently` commands can't run inside transactions.

### Backfills

If a migration adds a column that needs a default value computed from existing data, do it in two steps:

1. Migration A: add the column nullable, no default.
2. Backfill script (run separately, in batches if the table is large) sets the value.
3. Migration B: add the `not null` constraint and any necessary default.

Never write a single migration that locks a large table while computing values.

---

## Query-helper types — `exactOptionalPropertyTypes` and the conditional-spread pattern

`tsconfig.base.json` has `exactOptionalPropertyTypes: true`. That flag makes `foo?: T` reject **explicit `undefined`** — you can omit the key, but you can't assign `undefined` to it. The distinction matters at the DB boundary: an absent key falls through to the column default, an explicit `undefined` would not. The flag exists to keep those two states unconfused; do not turn it off.

This bites whenever a forwarded `input.foo` is *itself* optional and gets dropped into a payload literal. The canonical fix — used in `packages/db/queries/leads.ts:164` (`archiveLead` writing the `lead_events.payload` jsonb) — is to spread the property in conditionally:

```ts
// Type error under exactOptionalPropertyTypes:
payload: { type: "archived", reason: input.reason }

// Canonical pattern:
payload: {
  type: "archived",
  ...(input.reason !== undefined && { reason: input.reason }),
}
```

### `T | null` vs `foo?: T` — they are different things

This is the most common confusion and worth its own rule. Nullable columns and optional properties look similar in the type system but mean different things at the DB:

| Target shape | Meaning | Tool to use |
|---|---|---|
| `foo?: T` | Truly optional. Omitting and `undefined` are distinguishable; omitting lets the column default apply. | Conditional spread. |
| `foo: T \| null` | Nullable. The column accepts `null` as a real value; "missing" is not a state. | `input.foo ?? null`. |

`?? null` is the right tool for nullable columns and nullable jsonb. Conditional-spread on a `T | null` target is wrong — it would silently omit the key when you wanted an explicit `null`. Don't mix them.

Examples from `packages/db`:

- `lead_events.payload` discriminated union, `reason?: string` branch → **conditional-spread** (`leads.ts:164`).
- `lead_events.actor_id` is `text | null`, `audit_events.metadata`, `audit_events.ip`, `audit_events.user_agent` → **`?? null`**.

### What we don't do

- **Widen the type** (`reason: string | undefined` in the target shape). Defeats the flag and lets a real bug recur.
- **Cast with `as`.** Silently moves the bug to runtime; `audit_events` row ends up with an unintended shape.
- **Disable `exactOptionalPropertyTypes`.** It has already caught real bugs (IAF-2, IAF-40). It is locked per `tech-stack-canon`; relaxing it needs an ADR.

This rule applies to every Insert/Update literal, discriminated-union payload, and forwarded-input shape in `packages/db`. Audit was IAF-40 (2026-05-14); `leads.ts:164` is the reference example.

---

## Anti-patterns we don't do

These are real patterns I've watched ship to Freedom-shaped systems and regret:

- **`varchar(255)`.** Use `text`.
- **`bigint` PKs.** Use UUID v7.
- **Boolean flags lumped into a `status` integer.** Use a real enum.
- **"Soft enum" via `text` + check constraint.** Use a Postgres `enum`. The check constraint can't be migrated cleanly.
- **`deleted_at` without a partial index.** All your queries silently scan dead rows.
- **`updated_at` only updated from app code.** Trust the trigger.
- **Audit logging via Postgres triggers.** Move it to server actions where actor_id is known.
- **Storing currency as a column-name suffix** (`amount_eur`, `amount_chf`) when the currency is variable. Use a real `currency` column.
- **JSON blob columns that gain real keys over time.** Refactor to columns once a key is universal.
- **Cascading deletes from `auth.users`.** That's a footgun in a multi-tenant system. Use `restrict` and handle deletion via the GDPR erasure flow.

---

## Worked example — `leads` table

The full shape of `leads` for IAF-11. Use as a reference for similar tables:

```sql
-- Migration: 20260508120000_create_leads.sql
-- DOWN:
--   drop trigger if exists set_updated_at_leads on leads;
--   drop index if exists ix_leads_alive;
--   drop table if exists leads cascade;
--   drop type if exists lead_stage;
--   drop type if exists lead_tag;
--   drop type if exists lead_risk;

create type lead_stage as enum ('new','contacted','meeting','proposal','won','lost');
create type lead_tag   as enum ('hot','warm','cold');
create type lead_risk  as enum ('conservative','moderate','aggressive');

create table leads (
  id           uuid primary key default uuid_generate_v7(),
  advisor_id   uuid not null references auth.users(id) on delete restrict,
  name         text not null,
  email        text,
  phone        text,
  age          int,
  city         text,
  country      text,
  job          text,
  aum_chf      numeric(18,2) not null default 0,
  goals        text,
  risk_profile lead_risk,
  score        int check (score between 0 and 100),
  stage        lead_stage not null default 'new',
  tag          lead_tag,
  note         text,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  deleted_at   timestamptz
);

create index ix_leads_alive on leads (advisor_id, score desc) where deleted_at is null;
create index ix_leads_stage on leads (advisor_id, stage)       where deleted_at is null;

create trigger set_updated_at_leads before update on leads
  for each row execute function set_updated_at();

-- RLS policies live in this same file; see postgres-rls skill for the patterns.
```

The columns map to the mockup's seed leads. The two indexes cover the two most common queries: "list my leads sorted by score" and "kanban by stage."

---

## What "done" looks like for a schema ticket

Every schema PR description must contain:

1. The migration filename and a one-paragraph summary of what's added.
2. The RLS posture comment (see `postgres-rls`).
3. The list of audit-event `action` values emitted by mutations on this table.
4. Test artifact: either a Vitest run hitting the new table through the helpers, or a screenshot of the schema diff applied to staging.

If any of those four are missing, the PR isn't ready to merge.
