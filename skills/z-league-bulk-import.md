---
name: Bulk-import leads
description: Import up to 500 leads into a MEGA CRM in one request, with a dry-run preview and per-row results.
api: openapi/z-league-crm-lead-openapi.json
operations: [bulkImportLeads]
scopes: [public_api:leads:write]
---

# Bulk-import leads

Import many leads at once via `bulkImportLeads` (`POST /api/agents/crm/leads/bulk`). Server-to-server only.

## Auth
- `Authorization: Bearer mega_<token>` (PAT with `public_api:leads:write`)
- `x-customer-id: <uuid>`

## Steps
1. **Assemble rows** — up to **500** in `leads[]` (over the cap returns `400`). Each row needs at least one of `contact_name`/`contact_phone`/`contact_email`; optionally `row_ref` (echoed back), `owner_id` (must belong to your customer), `lead_line`, `stage_slug`, `custom_fields`.
2. **Preview first** — set `dry_run: true` to validate and preview results without writing. (`dry_run` requests are not idempotency-cached.)
3. **Choose merge behavior** — `on_duplicate: update` (default, merges) or `skip` (leave the existing lead untouched).
4. **Send** — include an `Idempotency-Key` so a retry is safe. Returns `200` with a `summary` (`total/created/updated/skipped/failed`), per-row `results[]` (`status`, `lead_id`, `matched_by`), and `unknown_custom_fields`.
5. **Reconcile** — a bad row is marked `failed` and never aborts the batch; check each `row_ref`. Unknown custom-field slugs are ignored and surfaced (not silently dropped).

## Rules
- The bulk endpoint has its own stricter limit: **10/min**. Back off on `429`.
- Bulk-uploaded leads are inert (no downstream automation) and do not fire the `lead.created` webhook.
