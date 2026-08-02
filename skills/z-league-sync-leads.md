---
name: Incrementally sync leads
description: Poll the MEGA CRM for new and updated leads using stable keyset cursor pagination.
api: openapi/z-league-crm-lead-openapi.json
operations: [listLeads]
scopes: [public_api:leads:read]
---

# Incrementally sync leads

Keep an external system in sync with a MEGA CRM by polling `listLeads`. Server-to-server only.

## Auth
Every request needs two headers:
- `Authorization: Bearer mega_<token>` (customer-locked PAT with `public_api:leads:read`)
- `x-customer-id: <uuid>` (must match the customer the key is locked to)

## Steps
1. **First page** — call `listLeads` (`GET /api/agents/crm/leads`) with `sort_by=updated_at&sort_dir=asc&limit=100`. For stable incremental polling always use this sort.
2. **Read the page** — process `leads[]`. `total` is the count matching the filter (ignores cursor/offset). Set `include_custom_fields=true` if you need the per-lead `custom_fields` map.
3. **Page forward** — if `next_cursor` is non-null, call again passing it as `cursor` **with the same `sort_by`/`sort_dir`** (a cursor replayed under a different sort returns `400`). Stop when `next_cursor` is `null`.
4. **Next poll** — resume from your last saved `cursor`, or pass `updated_since=<last ISO timestamp>` to fetch only changes.

## Rules
- Respect rate limits: default 60/min + 1000/hour. Back off on `429` using `Retry-After`.
- Filter server-side with `status`, `stageId`, `lead_line` (`buyer`/`seller`), `created_since`, `updated_since`, or `search` rather than scanning everything.
- Errors use `{ "error": { "type", "code", "message" } }` — branch on `type`.
