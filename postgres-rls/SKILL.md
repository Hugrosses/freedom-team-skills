---
name: postgres-rls
description: Use this skill before writing any RLS policy, before approving any schema PR, and before merging any change to `auth.users` or any tenant-bearing table. Triggers on every mention of "RLS", "policy", "tenant", "service role" in tickets.
---

# Postgres RLS — Freedom's "Every Table, No Exceptions" Rule

> Stub. Full skill to be authored when Kai picks up the first ticket that needs it (likely IAF-11 — leads schema). For now, this stub captures the non-negotiables.

## The rule

**Every table has RLS enabled before it accepts its first row.** Including tables that "feel internal." That is the table that gets exfiltrated. There are no exceptions.

```sql
alter table <name> enable row level security;
```

If you forget this, you'll see your data through the anon key. That's the bug.

## The four tenancy patterns

Most Freedom tables fall into one of four shapes:

1. **Advisor-owned** (`leads`, `clients`, `meetings`, `proposals`, `commissions`): policies read `advisor_id = auth.uid()`. Service role bypasses everything.
2. **Client-readable, advisor-writable** (`messages`, `documents`): two-policy pattern; the client side authenticates via the portal session that resolves to the client's user_id.
3. **Service-role-only** (`audit_events`, integration webhook payload tables): no policy permits authenticated reads/writes; only service role.
4. **Public read, service-role write** (`products`, `templates` seed data): single permissive `select` policy `using (true)`; mutations only via service role.

Anti-pattern: **never** ship `using (true)` for `select` AND for mutations. That's open access in disguise.

## Service role usage

- `service_role` bypasses RLS by design. Only use it from server-side code that has already authenticated the actor through other means.
- Server actions: don't reach for `service_role` because RLS is "in the way." If the policy is in the way, the policy is wrong; fix the policy.
- Tests: factories may use `service_role` for setup/teardown, but business-logic assertions run with a real user role.

## Patterns to author here later

- The advisor-owned policy template (insert/update/select/delete) with audit-hook expectations.
- The portal session pattern (how clients authenticate and what their `auth.uid()` maps to).
- The "policy + index" coupling — every RLS predicate needs a supporting index, otherwise queries become full scans.
- Anti-patterns observed in the Freedom-shape: cross-tenant leaks via `lead_id` joins, `auth.uid()` in jsonb predicates, etc.

Author this in full once Kai has shipped IAF-11 (leads RLS) and IAF-15 (meetings RLS). The patterns from those two tickets inform the rest.
