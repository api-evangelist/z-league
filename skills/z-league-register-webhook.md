---
name: Register and verify a lead.created webhook
description: Subscribe to MEGA lead.created events and verify the HMAC signature on each delivery.
api: openapi/z-league-crm-lead-openapi.json
operations: [createWebhook, listWebhooks, updateWebhook, deleteWebhook]
scopes: [public_api:webhooks:manage]
---

# Register and verify a lead.created webhook

Receive real-time `lead.created` events. Managing webhooks needs `public_api:webhooks:manage`.

## Auth
- `Authorization: Bearer mega_<token>` (PAT with `public_api:webhooks:manage`)
- `x-customer-id: <uuid>`

## Steps
1. **Register** — `createWebhook` (`POST /api/agents/crm/lead-webhooks`) with a **public HTTPS** `url` (SSRF-validated; loopback/internal rejected) and `event_types: ["lead.created"]`. Optionally set `timeout_seconds` (1–60, default 10) and `retry_attempts` (0–10, default 5).
2. **Store the secret** — the `secret` (`mega_whsec_...`) is returned **exactly once**. Save it in a secret manager; you verify signatures with it.
3. **Verify each delivery** — compute `HMAC_SHA256(secret, X-Mega-Timestamp + "." + rawRequestBody)`, prefix `sha256=`, and constant-time compare to `X-Mega-Signature`. Verify against the RAW bytes — do not re-serialize the parsed JSON.
4. **Guard replays** — reject deliveries whose `X-Mega-Timestamp` is outside a ~5 minute window. De-duplicate at-least-once retries on `X-Mega-Delivery`.
5. **Acknowledge** — respond `2xx`. Any non-2xx or timeout is retried (up to `retry_attempts + 1` attempts, exponential backoff capped at 30s).
6. **Manage** — `listWebhooks` (secrets never returned), `updateWebhook` (`rotate_secret: true` mints a fresh secret, returned once), `deleteWebhook` (soft-delete).

## Rules
- Only `lead.created` is emitted, and only on genuine new inserts (not merges; bulk uploads do not fire).
- Redirects are not followed; SSRF-blocked URLs are terminal (not retried).
