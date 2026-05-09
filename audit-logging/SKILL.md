---
name: audit-logging
description: Use this skill before adding any audit-event action name, before approving any compliance-relevant mutation, and before designing any export of audit data. Triggers on every reference to "audit", "compliance", "MiFID", "GDPR" in tickets.
---

# Audit Logging — Freedom's Compliance Trail

> Stub. Full skill to be authored when Kai ships IAF-29 (the audit log surface itself). For now, this captures the contract.

## The non-negotiables

- Every mutation that touches a **compliance-relevant table** (see `supabase-schema` for the list) emits an `audit_events` row.
- The hook is `pushAudit({ action, target_kind, target_id, metadata? })` from `packages/db/queries/audit.ts`. Server actions call it; triggers do not.
- Action names are SCREAMING_SNAKE_CASE, past-tense: `LEAD_ACCEPTED`, `PROPOSAL_SIGNED`, `MEETING_ENDED`, `KYC_VERIFIED`.
- `metadata` is a `jsonb` blob; **no PII** allowed in it. The lint rule (and human reviewers) enforce this.
- `audit_events` rows are **never deleted**. Even GDPR erasure preserves the row, redacting PII fields in `metadata` and clearing `actor_id` to a tombstone.

## The canonical action vocabulary (phase 1)

Maintained here as the single source of truth. Add new actions only when you ship the corresponding mutation; don't pre-declare actions that don't yet exist.

```
auth:        SIGNED_IN, SIGNED_OUT, MAGIC_LINK_SENT, MAGIC_LINK_USED
advisor:     ADVISOR_PROFILE_UPDATED, ADVISOR_AVAILABILITY_UPDATED
kyc:         KYC_DOCUMENT_UPLOADED, KYC_VERIFIED (system actor in P0 stub)
lead:        LEAD_CREATED, LEAD_SCORED, LEAD_ACCEPTED, LEAD_STAGE_CHANGED, LEAD_ARCHIVED, LEAD_CONVERTED
meeting:     MEETING_SCHEDULED, MEETING_STARTED, MEETING_ENDED, MEETING_CANCELLED, NOTE_ADDED
proposal:    PROPOSAL_DRAFTED, PROPOSAL_SENT, PROPOSAL_SIGNED, PROPOSAL_DECLINED, DOCUSEAL_DISPATCH
client:      CLIENT_CREATED, DOC_UPLOADED, DOC_ACCESSED, MESSAGE_SENT, PORTFOLIO_SNAPSHOT_TAKEN
commission:  COMMISSION_BOOKED, COMMISSION_PAID, COMMISSION_REVERSED
billing:     SUBSCRIPTION_CREATED, SUBSCRIPTION_UPDATED, INVOICE_PAID, INVOICE_FAILED
compliance:  COMPLIANCE_REVIEW_RAN
```

If you find yourself wanting an action not on this list, propose adding it in your PR description. Aria curates.

## What `metadata` may contain

- Identifiers (uuid, integration ids).
- Counts, amounts (no monetary identifiers like account numbers).
- Status changes (from → to).
- Reason codes for failures.

What it must NOT contain:

- Email addresses, phone numbers, full names.
- Document content, file contents.
- Free-text user input.
- IP addresses (those go in their dedicated `ip` column, scrubbed on erasure).

## Patterns to author here later

- The lint rule that enforces `pushAudit` on compliance-relevant mutations.
- The CSV export contract (column order, escaping, time-zone handling).
- The redaction routine for GDPR erasure (which fields, what tombstone).
- The dashboard query patterns for "events involving client X" / "events on date Y" / "all PROPOSAL_SIGNED in 2026-Q2."

Author in full when IAF-29 ships.
