---
name: Push a lead safely (idempotent)
description: Create or merge a single lead in a MEGA CRM without creating duplicates on retry.
api: openapi/z-league-crm-lead-openapi.json
operations: [createLead]
scopes: [public_api:leads:write]
---

# Push a lead safely (idempotent)

Create one lead via `createLead` (`POST /api/agents/crm/leads`). Server-to-server only.

## Auth
- `Authorization: Bearer mega_<token>` (PAT with `public_api:leads:write`)
- `x-customer-id: <uuid>`

## Steps
1. **Build the body** — provide at least one of `contact_name`, `contact_phone`, `contact_email`. Optionally set `source_platform`, `source_type`, `stage_slug`, and a `custom_fields` map (scalar values).
2. **Set an idempotency key** — send an `Idempotency-Key` header (a fresh UUID per logical create). Replaying the same key returns the original stored response instead of double-creating; reusing it with a *different* body returns `409`.
3. **Call createLead** — `POST /api/agents/crm/leads`. Returns `201` with `{ "lead": { ... } }`.
4. **Understand merges** — MEGA de-duplicates on `email` and `phone-last-10`. A matching lead is **merged**, not duplicated, so a `201` may return an existing (updated) lead.

## Rules
- Never call from a browser/mobile client — the key is a secret.
- Handle `400` (missing required contact field), `403` (missing scope / wrong customer), `429` (honor `Retry-After`).
- Only genuine new inserts fire the `lead.created` webhook; merges do not.
