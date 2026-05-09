---
name: server-actions
description: Use this skill before writing any Next.js Server Action, before any mutation that crosses the client/server boundary, and before adding a new query helper to `packages/db/queries`. Triggers on every "save", "create", "update", "delete" surface in Freedom.
---

# Server Actions — Freedom's Mutation Layer

> Stub. Full skill to be authored when Kai picks up IAF-13 (lead detail's accept-lead action) — the first server action that exercises the full audit + RLS pattern.

## The conventions

- Every mutation that crosses the client/server boundary is a **Server Action**, not an API route. API routes are for inbound webhooks only.
- File location: `apps/web/app/actions/<surface>.ts`. Top of file: `"use server"`.
- Naming: verb-first, action describes the user intent — `acceptLead`, `signProposal`, `archiveClient`. Never `updateLead`/`saveProposal`.
- Inputs are validated with Zod at the action's first line. No exceptions.
- Returns `{ data, error }` shape; never throws on expected failures (validation, RLS denial, business-rule violation). Throw only on truly unexpected errors that Sentry should catch.
- Audit hook on every mutation that touches a compliance-relevant table (see `audit-logging`).

## Pattern skeleton

```ts
"use server";

import { z } from "zod";
import { createServerClient } from "@/packages/db/client";
import { pushAudit } from "@/packages/db/queries/audit";

const InputSchema = z.object({ leadId: z.string().uuid() });

export async function acceptLead(input: z.infer<typeof InputSchema>) {
  const parsed = InputSchema.safeParse(input);
  if (!parsed.success) return { data: null, error: { code: "INVALID_INPUT", details: parsed.error.flatten() } };

  const supabase = createServerClient();
  const { data: lead, error } = await supabase
    .from("leads")
    .update({ stage: "contacted" })
    .eq("id", parsed.data.leadId)
    .select()
    .single();

  if (error) return { data: null, error: { code: "RLS_DENIED_OR_NOT_FOUND", details: error.message } };

  await pushAudit({ action: "LEAD_ACCEPTED", target_kind: "lead", target_id: lead.id });
  return { data: lead, error: null };
}
```

## Patterns to author here later

- Error-shape catalogue (every error code Freedom uses).
- The "fan out to multiple tables in a transaction" pattern (proposal sign → client + commission, IAF-23).
- Idempotency keys for actions that may be triggered twice (webhooks, retried client requests).
- How to test server actions with Vitest + the test-harness factories.
- When to break out an action into an Edge Function instead (long-running, cron, AI-cost-bearing).

Author in full once IAF-13 ships and we have a real reference action to point at.
