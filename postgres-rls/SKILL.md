---
name: postgres-rls
description: Use this skill before writing any RLS policy, before approving any schema PR, and before merging any change that touches tenancy (advisor_id, client_id, user_id mapping). Triggers on every mention of "RLS", "policy", "tenant", "service role", "auth.uid()" in tickets and PRs.
---

# Postgres RLS — Freedom's "Every Table, No Exceptions" Rule

This is the source of truth for *how* we keep advisors' data isolated from each other in Postgres. Get RLS wrong and one advisor sees another advisor's clients, KYC documents, audit trail. There is no "fix it next sprint" for that. Get it right on day one.

If you find yourself wanting to ship a table without RLS, or with a `using (true)` policy "just for now," stop. Bring it to Aria. There is no exception that this skill doesn't already cover.

---

## The non-negotiables

1. **RLS is enabled before the first row is inserted.** Same migration as the `create table`. Not the next migration. Same one.
2. **Every table has at least one policy.** A table with RLS enabled and no policies is invisible to authenticated users — that's correct for service-role-only tables, but you have to *decide* that's the case, not arrive there by forgetting.
3. **Service role never appears in app code that handles user requests.** Reach for service-role only in webhooks, cron, AI endpoints, and tests. If a server action "needs" service role to work, the policy is wrong; fix the policy.
4. **Every RLS predicate that filters by an indexed column references that index.** Without it, every query becomes a sequential scan over every advisor's data. That's both slow and a tail-latency information leak.
5. **Tests run as a real authenticated user, not as service role.** A passing test under service role tells you nothing about whether the policy isolates tenants.

---

## Enabling RLS

For every business table:

```sql
alter table leads enable row level security;
```

That's the line. Without it, your policies are decoration — they don't filter anything.

You will sometimes also see `force row level security`. The difference:

- `enable` — RLS applies to all roles **except** the table owner and roles with the `BYPASSRLS` attribute (in Supabase: `service_role`).
- `force` — RLS applies even to the table owner.

In Supabase you almost never need `force`. Application code uses `anon` or `authenticated`; webhooks/cron use `service_role` (which is *meant* to bypass). The table-owner role is a Postgres internal we don't connect with from app code. Reach for `force` only on tables where even the owner shouldn't accidentally read rows in a SQL console (e.g., a table holding payment-method tokens — phase 2 concern).

---

## Policy syntax — the shape

```sql
create policy <policy_name> on <table>
  as <permissive | restrictive>     -- default: permissive
  for <select | insert | update | delete | all>
  to <role>                         -- default: public; in Supabase you almost always want authenticated
  using (<read_or_old_row_predicate>)
  with check (<new_row_predicate>);
```

- `using` filters which rows are *visible* to `select`/`update`/`delete`.
- `with check` filters which rows are *acceptable* on `insert`/`update`. Without it, a malicious `update` could move a row out of the user's tenant.
- `for select` policies need only `using`. `for insert` policies need only `with check`. `for update` policies need both. `for delete` policies need only `using`.
- Use `for all` only when both predicates are identical. Otherwise write four explicit policies — clearer at a glance.
- Permissive policies are OR-combined (any matching policy permits the row). Restrictive policies are AND-combined (all restrictive policies must pass). For Freedom, use permissive everywhere — restrictive policies are an over-engineered tool we don't need yet.

---

## The `(select auth.uid())` pattern

Always wrap `auth.uid()` in a subselect:

```sql
-- BAD: re-evaluated per row, no plan caching
using (advisor_id = auth.uid())

-- GOOD: cached once per query, recommended by Supabase
using (advisor_id = (select auth.uid()))
```

For a table with a few hundred rows this doesn't matter. For `audit_events` or `meeting_notes` after six months in production, it does. Make it muscle memory.

The same pattern applies to `auth.jwt()` if you read JWT claims (rare in Freedom).

---

## The four tenancy patterns

Every Freedom table fits one of these four shapes. Decide which when you write the migration; the policies follow mechanically.

### Pattern A — Advisor-owned

Use for: `leads`, `meetings`, `meeting_notes`, `proposals`, `proposal_mix`, `clients`, `commissions`, `documents`, `kyc_documents`, `lead_events`, `commission_events`.

Every row carries an `advisor_id` that must equal the calling user's `auth.uid()`. Service role bypasses (for AI scoring, webhooks, cron).

```sql
-- Worked example: leads
alter table leads enable row level security;

create policy leads_advisor_select on leads
  for select to authenticated
  using (advisor_id = (select auth.uid()) and deleted_at is null);

create policy leads_advisor_insert on leads
  for insert to authenticated
  with check (advisor_id = (select auth.uid()));

create policy leads_advisor_update on leads
  for update to authenticated
  using (advisor_id = (select auth.uid()))
  with check (advisor_id = (select auth.uid()));

-- Soft delete only — no hard-delete policy on advisor-owned tables.
-- The "delete" path is an update setting deleted_at = now().
```

Notes:

- The `select` policy includes `deleted_at is null`. Soft-deleted rows are invisible to the advisor. If you need to see them (admin tooling, GDPR audit), come through service role.
- The `with check` on `update` ensures an advisor can't transfer a row to another advisor by changing `advisor_id`. Without it, that's a real attack surface.
- Don't grant `delete`. Soft-delete via update.

### Pattern B — Public read, service-role write

Use for: `products`, `templates`. These are seeded reference data; advisors read but never write.

```sql
alter table products enable row level security;

create policy products_public_read on products
  for select to authenticated
  using (true);

-- No insert/update/delete policies for authenticated.
-- Service role bypasses RLS by default and writes via seed scripts.
```

Notes:

- Catalog updates happen through `supabase/seed.sql` running with service role on deploy.
- Don't grant `to anon` unless the data is truly public (the marketing site reading product names — phase 2 concern). For now, `to authenticated` is enough.

### Pattern C — Service-role-only writes, advisor-scoped reads (the audit trail special case)

Use for: `audit_events`. Append-only. Advisors see their slice; system writes via SECURITY DEFINER function.

```sql
alter table audit_events enable row level security;

-- Read: advisor sees rows where they were the actor or where the target chains to them.
create policy audit_advisor_read on audit_events
  for select to authenticated
  using (
    actor_id = (select auth.uid())
    or exists (
      select 1 from leads l where l.id = audit_events.target_id
        and l.advisor_id = (select auth.uid())
    )
    or exists (
      select 1 from clients c where c.id = audit_events.target_id
        and c.advisor_id = (select auth.uid())
    )
    or exists (
      select 1 from proposals p where p.id = audit_events.target_id
        and p.advisor_id = (select auth.uid())
    )
  );

-- No insert policy for authenticated. Inserts go through pushAuditEvent() RPC,
-- which is a SECURITY DEFINER function that validates actor_id = auth.uid()
-- and inserts with the postgres role's elevated permissions.
-- See the audit-logging skill for the function definition.

-- No update or delete policies. Audit is append-only. Forever.
```

Why a SECURITY DEFINER function instead of a permissive insert policy?

- A permissive `for insert to authenticated with check (actor_id = (select auth.uid()))` works, but it lets an authenticated user write *any* `target_kind` and `target_id` they want, even ones unrelated to them. That dilutes the trail.
- A SECURITY DEFINER RPC validates the (actor, target) relationship before inserting, so the audit trail can't be forged with garbage entries.

For phase 1, the simpler permissive insert policy is acceptable as long as Mira's UI never lets a user choose `target_id` directly. As soon as we expose any "log a custom event" surface (we won't), upgrade to the RPC.

### Pattern D — Client-readable, advisor-writable (messages and the client portal)

Use for: `messages`, and the client-portal slice of `documents` + `portfolio_snapshots`.

This pattern requires a `clients.user_id` column linking the row to a `auth.users` id, so the client's `auth.uid()` resolves to a real `clients` row. The current `clients` schema (IAF-24 as drafted) doesn't have it yet — **add `user_id uuid references auth.users(id) unique` when you implement IAF-24**, or this pattern can't be enforced safely.

Sketch (full implementation in the IAF-37+ tickets, but document the shape now so the schema decisions in IAF-24 don't paint us into a corner):

```sql
-- messages: advisor writes, both parties read
alter table messages enable row level security;

create policy messages_advisor_write on messages
  for insert to authenticated
  with check (
    sender_id = (select auth.uid())
    and exists (
      select 1 from clients c
      where c.id = messages.client_id
        and c.advisor_id = (select auth.uid())
    )
  );

create policy messages_participants_read on messages
  for select to authenticated
  using (
    -- advisor on the conversation
    exists (
      select 1 from clients c
      where c.id = messages.client_id
        and c.advisor_id = (select auth.uid())
    )
    -- client on the conversation
    or exists (
      select 1 from clients c
      where c.id = messages.client_id
        and c.user_id = (select auth.uid())
    )
  );
```

For the client portal, `auth.uid()` is the client's user id. The advisor and the client are both authenticated; the policy distinguishes them by which `clients` row they map to.

Defer detailed client-side update/delete policies to IAF-37 — keep them out of P0 schemas.

---

## The "policy + index" coupling

Every RLS predicate that filters rows must hit an index. Without it, RLS turns every query into a per-row predicate evaluation across the whole table.

Concretely: if your policy says `using (advisor_id = (select auth.uid()))`, the table needs an index whose **leftmost column** is `advisor_id`. The composite index `ix_leads_alive (advisor_id, score desc) where deleted_at is null` from `supabase-schema` satisfies this because the leftmost column is `advisor_id`.

Anti-pattern: an index on `(score desc, advisor_id)` looks similar but doesn't help RLS. The leftmost column matters.

A quick rule: **for every advisor-owned table, the first index you create should have `advisor_id` as its first column.** Any other index is a follow-up.

For the audit_events `exists (select 1 from leads ...)` style policy, every joined table also needs `(advisor_id, id)` as a covering index. Otherwise the policy turns into a nested loop. That's fine while seeded data is small; revisit when a table crosses 10k rows.

---

## Testing RLS

A Vitest suite that covers RLS for a table looks like this (factories from `packages/db/test-harness/`):

```ts
import { describe, it, expect } from "vitest";
import { createTestAdvisor, createTestLead, asAdvisor } from "@/packages/db/test-harness";

describe("leads RLS", () => {
  it("an advisor sees only their own leads", async () => {
    const alice = await createTestAdvisor();
    const bob = await createTestAdvisor();
    await createTestLead(alice, { name: "Alice's lead" });
    await createTestLead(bob, { name: "Bob's lead" });

    const { data } = await asAdvisor(alice).from("leads").select("name");

    expect(data?.map(r => r.name)).toEqual(["Alice's lead"]);
  });

  it("an advisor cannot transfer a lead to another advisor", async () => {
    const alice = await createTestAdvisor();
    const bob = await createTestAdvisor();
    const lead = await createTestLead(alice);

    const { error } = await asAdvisor(alice)
      .from("leads")
      .update({ advisor_id: bob.id })
      .eq("id", lead.id);

    // RLS check fails because the new row would not match the policy
    expect(error).toBeTruthy();
  });

  it("an advisor cannot read soft-deleted leads", async () => {
    const alice = await createTestAdvisor();
    const lead = await createTestLead(alice);
    await asAdvisor(alice).from("leads").update({ deleted_at: new Date() }).eq("id", lead.id);

    const { data } = await asAdvisor(alice).from("leads").select("id");
    expect(data?.find(r => r.id === lead.id)).toBeUndefined();
  });
});
```

Required test cases for every advisor-owned table:

1. **Cross-tenant read isolation.** Two advisors, A's row, B reads `select` — gets nothing.
2. **Tenant-transfer prevention.** A creates a row, A tries to update `advisor_id` to B's id — fails.
3. **Deleted-row invisibility** (if soft-delete applies).
4. **Authenticated-but-no-row.** A new advisor with zero rows reads — gets empty array, not error.

Cross-tenant write isolation is implied by the `with check`, but write a test for `insert` too if the table accepts client input.

---

## Anti-patterns we don't ship

- **`using (true)` "for now."** Once it ships, it never gets replaced. There is no "later." If you can't write the predicate, you don't have a clear enough mental model of who reads this table — bring it to Aria.
- **Service role in server-action code.** Every time someone "just needs to bypass RLS for a moment," it's a sign the policy is wrong. Fix the policy.
- **`auth.uid()` without `(select ...)`.** Catastrophic perf cost on tables that grow.
- **Forgetting `with check` on update policies.** Open tenant-transfer attack.
- **Cross-tenant joins via foreign keys without checking the join target's RLS.** If a policy on `audit_events` joins to `leads`, and `leads` has its own policy, the planner is supposed to apply both — but only if the policy on `leads` allows the row at all. Sanity-check by running the query as the test user and confirming the expected row count.
- **`for all` policy where you actually need different `select` and `update` predicates.** Write four explicit policies.
- **Granting `to anon` on a table that contains any user-specific data.** `anon` is the unauthenticated role. `to authenticated` is what you almost always want.
- **Trying to hide data via column-level grants** instead of row-level policies. Postgres supports column-level privileges, but the right tool here is RLS.
- **Auditing access by adding a `read_at timestamptz` column** that gets updated on every read. A: that's what the audit log is for. B: it requires a trigger on `select` that destroys query plans.

---

## Migration discipline for RLS

- Policies live in the **same migration file** as the table they protect. Never in a follow-up. A reviewer should be able to read one file and know the security posture.
- Renaming a policy (`alter policy`) is normally fine, but if you change the predicate, that's a *new* policy semantically. Drop and recreate, and consider whether existing rows need backfill.
- Dropping a policy without replacing it is a security regression. A migration that drops policies needs an ADR.
- When you `alter table` to add a column, ask whether existing policies are still correct. A new column might leak via `select *`. Usually fine for advisor-owned tables (the row is theirs anyway), but always check.
- The first thing in a `down` block (which is documentation only — see `supabase-schema`) is "re-enable RLS, recreate policies." This is the easiest part to forget.

---

## How to review an RLS PR

When Aria asks you to review another agent's schema PR, walk through:

1. Is RLS enabled on every new table? (`alter table ... enable row level security`)
2. Does each table have at least one policy?
3. For every advisor-owned table: do `select`, `insert`, `update` policies all exist, and does `update` have both `using` and `with check`?
4. Is the policy predicate indexed (leftmost column)?
5. Is `auth.uid()` wrapped in `(select ...)`?
6. Is there a Vitest test that covers the four required cases above?
7. Are policies in the same migration as the table?
8. If the table is compliance-relevant, does the PR description list audit-event actions?

If any answer is no, request changes. Don't approve.
