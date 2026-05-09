---
name: gdpr-erasure
description: Use this skill before designing any "delete account" / "erase my data" flow, before approving any PII-bearing schema, and before responding to a real GDPR erasure request. Triggers on every "GDPR", "right to be forgotten", "data deletion" mention.
---

# GDPR Erasure — Freedom's Right-to-be-Forgotten Flow

> Stub. Full skill to be authored when erasure goes from "phase 2 problem" to "phase 1 problem" — likely when a real beta user asks. For phase 1 the prototype is architected for it but doesn't expose the flow yet.

## The contract

- Erasure is a **named, audited action**: `USER_ERASURE_REQUESTED`, `USER_ERASED`. Both are audit-events.
- Erasure deletes:
  - The user's `auth.users` row.
  - All advisor-owned business rows (cascade behavior set in schema).
  - All Storage objects belonging to them (KYC docs, proposal PDFs, profile photos).
- Erasure preserves:
  - `audit_events` rows authored by or about them, with PII fields redacted to `[ERASED]` and `actor_id` set to a tombstone constant.
  - `commissions` rows older than the legal retention window (10 years for Swiss/EU advisor commissions; check with legal before final implementation).
  - Aggregate analytics (anonymized counts) — never per-row.

## The decision tree per table

When you add a table, decide its erasure behavior in the migration. Document in the PR description:

| Behavior | When to use | Example |
|---|---|---|
| Hard delete | Personal data with no legal retention requirement | `messages`, `meeting_notes`, `kyc_documents` |
| Soft delete (already done by `deleted_at`) | The row's continued existence is needed by another agent (advisor, counterparty) but the erased user shouldn't appear | n/a — soft delete alone is not GDPR-compliant erasure |
| Redact in place | Compliance-required retention but PII-bearing | `audit_events`, `proposals` (signed), `commissions` |
| Cascade-erase | The row is meaningless without the user | `portfolio_snapshots`, `proposal_mix`, `meeting_attendees` |

## The redaction routine

```sql
-- For audit_events:
update audit_events
set actor_id = '00000000-0000-0000-0000-000000000000',
    metadata = metadata - 'email' - 'name' - 'phone' - 'document_path'
              || jsonb_build_object('redacted_at', now(), 'redacted_for_erasure', true)
where actor_id = $erased_user_id;
```

Same pattern for proposals and commissions, scrubbing the PII keys and tombstoning identifiers while leaving timestamps + amounts intact for compliance.

## Patterns to author here later

- The full erasure server-action / edge function, with transaction boundaries.
- The "verify before erase" identity check (we don't erase based on an email alone).
- The runbook for legal-mandated erasures (e.g., a regulator says "erase this client's KYC after 7 years"). Different from user-requested.
- Test coverage: a Vitest suite that creates a full advisor + 3 leads + 2 clients + signed proposal + commission, then runs erasure, then asserts what's gone vs what's preserved-with-redaction.

Author in full when phase 2 is on the horizon, or when a real user requests erasure — whichever comes first.
