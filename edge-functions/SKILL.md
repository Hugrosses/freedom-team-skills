---
name: edge-functions
description: Use this skill before writing any Supabase Edge Function, any webhook handler, or any cron-triggered job. Triggers on every "webhook", "scheduled job", "background task" in tickets.
---

# Edge Functions — Freedom's Webhooks & Background Work

> Stub. Full skill to be authored when Kai ships IAF-22 (DocuSeal webhook) — the first non-trivial inbound webhook. For now, the rules:

## The non-negotiables

- Inbound webhooks **verify signatures** before doing anything else. No exceptions. If the SDK doesn't expose signature verification, escalate to Aria — we don't trust unsigned payloads.
- Every webhook handler is **idempotent**. Webhook delivery retries; your handler must produce the same end-state on the 5th attempt as on the 1st. Use the provider's idempotency-key header when present; otherwise hash a deterministic subset of the payload.
- Webhooks run as **service role**. They've already authenticated themselves via signature verification.
- Long-running work (>15s wall-clock) does not happen inside an HTTP handler. Enqueue and return 200; let a separate function process.
- AI-cost-bearing endpoints are **rate-limited per user/per IP**. Use Upstash if needed.

## File layout

```
supabase/functions/
├── docuseal-webhook/
│   ├── index.ts
│   └── deno.json
├── stripe-webhook/
│   └── index.ts
├── score-lead/         # AI cost-bearing, rate-limited
│   └── index.ts
└── _shared/
    ├── verify-signature.ts
    └── idempotency.ts
```

## Pattern skeleton

```ts
import { verifyDocuSealSignature } from "../_shared/verify-signature.ts";
import { withIdempotency } from "../_shared/idempotency.ts";

Deno.serve(async (req) => {
  const raw = await req.text();
  const sig = req.headers.get("x-docuseal-signature");
  if (!verifyDocuSealSignature(raw, sig, Deno.env.get("DOCUSEAL_WEBHOOK_SECRET"))) {
    return new Response("invalid signature", { status: 401 });
  }
  const event = JSON.parse(raw);
  return withIdempotency(event.id, async () => {
    // do the work here
    return new Response("ok", { status: 200 });
  });
});
```

## Patterns to author here later

- Signature verification per provider (DocuSeal, Stripe, Cal.com, Resend bounce).
- The idempotency table shape and TTL.
- Cron job patterns (recurring portfolio snapshots, weekly digest emailing).
- AI-cost-bearing endpoints: cost ceiling enforcement and Sentry breadcrumbs.
- Local development — running edge functions against the Supabase CLI.

Author in full when IAF-22 ships.
