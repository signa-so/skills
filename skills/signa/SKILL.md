---
name: signa
description: Build with the Signa API — trademark search, clearance screening, monitoring (watches, alerts, webhooks), owner/attorney/firm data, and deadline computation across trademark offices worldwide. Use this skill whenever the user is working with Signa, api.signa.so, docs.signa.so, the @signa-so/sdk package, sig_ API keys, or building anything involving trademark search, brand clearance, trademark watching/monitoring, opposition deadlines, or Nice classifications against the Signa API.
license: MIT
metadata:
  author: Signa
  homepage: https://docs.signa.so
  source: https://github.com/signa-so/skills
---

# Signa API

Signa is a REST API for global trademark intelligence: search trademarks across the world's trademark offices (USPTO, EUIPO, WIPO/Madrid, and more — `GET /v1/offices` returns current live coverage), retrieve full records with owners/attorneys/history, screen names for clearance risk, and monitor filings with watches, alerts, and webhooks.

- **Base URL:** `https://api.signa.so/v1`
- **Auth:** `Authorization: Bearer sig_...` (keys are `sig_` + 48 hex chars; get one at [app.signa.so](https://app.signa.so))
- **Docs:** https://docs.signa.so — **every page is fetchable as raw markdown** by appending `.md` to its path
- **OpenAPI:** `https://api.signa.so/v1/openapi.json` (public, no auth)
- **TypeScript SDK:** `npm install @signa-so/sdk`

## Always check the live docs

The API is in beta and evolves quickly. This skill covers the stable conventions; for exact parameter lists, new endpoints, or anything not covered here, fetch the live docs — they are the source of truth:

1. Fetch `https://docs.signa.so/llms.txt` — the full index of every docs page with one-line summaries.
2. Fetch any page as markdown by appending `.md`, e.g. `https://docs.signa.so/guides/search.md`, `https://docs.signa.so/api-reference/errors.md`.
3. For exact request/response schemas, fetch `https://api.signa.so/v1/openapi.json`.

Do this proactively when writing code against an endpoint whose parameters you're not certain about. Guessing parameter names is the #1 source of `validation_error` responses.

## First call

```bash
curl -G "https://api.signa.so/v1/trademarks" \
  -H "Authorization: Bearer $SIGNA_API_KEY" \
  --data-urlencode "q=acme" \
  --data-urlencode "offices=uspto,euipo"
```

```typescript
import { Signa } from "@signa-so/sdk";

const signa = new Signa({ api_key: process.env.SIGNA_API_KEY });

const results = await signa.trademarks.search({
  query: "acme",
  strategies: ["exact", "phonetic", "fuzzy"],
  filters: { offices: ["uspto", "euipo"], nice_classes: [9, 42] },
});
```

Keep `sig_*` keys server-side only — never in client-side JavaScript, mobile apps, or anything a user can inspect. Proxy through your backend if a browser needs Signa data.

## Request conventions (get these right)

These conventions are strict — violations return `400 validation_error`:

- **Query params are `snake_case`**: `status_stage=registered`, `nice_classes=9,42`.
- **Arrays are comma-separated**: `offices=uspto,euipo`. Bracketed keys (`offices[]=uspto`) are rejected.
- **Date ranges use flat underscore operators**: `filing_date_gte=2020-01-01&filing_date_lt=2025-01-01` (`gte` inclusive, `lt` exclusive). No bracket syntax.
- **Booleans are literal `true`/`false`** — `1`, `yes`, `TRUE` are all rejected.
- **Office codes are lowercase** (`uspto`), **jurisdiction codes are uppercase ISO** (`US`, `EU`).
- **`GET /v1/trademarks` requires at least one filter** — unscoped list queries are rejected.
- **`application_number` / `registration_number` filters require `office`** alongside them (`ir_number` is globally unique and doesn't).
- **On `POST /v1/trademarks`, filters must be nested under `"filters"`** — passing `offices`, `nice_classes`, etc. at the top level fails with `unrecognized_keys`. There is no `search_type` or `type` field; strategy selection is the `strategies` array.

## Response envelope

Single resources come back at the top level with `object` and `request_id`:

```json
{ "id": "tm_abc123", "object": "trademark", "mark_text": "ACME", "request_id": "req_..." }
```

Lists use one standard envelope — `has_more` is top-level, not inside `pagination`:

```json
{
  "object": "list",
  "data": [ ... ],
  "has_more": true,
  "pagination": { "cursor": "eyJ..." },
  "request_id": "req_..."
}
```

- **Pagination is cursor-based only.** Pass `cursor` from the previous response; `limit` is 1–100 (default 20). Cursors expire after 24h and are bound to the issuing endpoint + query — don't hand-craft or reuse them across queries.
- `?include_total=true` adds `pagination.total_count` (+ `total_count_approximate`; counts cap at 10k).
- Sort with prefix syntax: `?sort=-filing_date` (descending).
- Every response has a top-level `request_id` — surface it in logs and support requests.

## IDs

Public IDs are prefixed: `tm_` trademark, `own_` owner, `att_` attorney, `firm_` firm, `ent_` resolved entity, `prc_` proceeding, `ptf_` portfolio, `wat_` watch, `alt_` alert, `whk_` webhook, `evt_` event, `key_` API key, `req_` request. Using the wrong type returns `400 id_type_mismatch`.

Owners and entities can be merged by entity resolution: a `410 entity_merged` response carries `merged_into` with the canonical ID — follow it and update cached references.

## Errors

All errors wrap a structured object under `"error"` with a stable snake_case `type` slug, plus top-level `request_id`:

```json
{
  "error": {
    "type": "rate_limited", "title": "Rate limit exceeded", "status": 429,
    "detail": "...", "suggestion": "...", "retryable": true, "retry_after": 30
  },
  "request_id": "req_abc123"
}
```

Switch on `error.type`, honor `retryable` and `retry_after`. Key slugs: `validation_error`, `unauthorized`, `forbidden`, `plan_upgrade_required`, `not_found`, `id_type_mismatch`, `conflict`, `idempotency_processing` (retryable), `entity_merged` (410), `entity_too_large` (422), `cursor_expired`, `rate_limited`, `internal_error`, `service_unavailable`. Full catalog: `https://docs.signa.so/api-reference/errors.md`.

## Rate limits & quotas

Two independent controls per organization (all keys share them):

- **Monthly quota pools** (beta): search 100k, read 500k; reference/utility/monitoring unmetered. Daily sub-cap of 10% of monthly, reset UTC midnight.
- **Per-minute rate limits** (beta): search 1,000/min, read 10,000/min, monitoring 100/min, others 1,000/min.

Responses carry IETF headers: `RateLimit-Policy`, `RateLimit: remaining=N, reset=S`, and `Retry-After` on 429. Back off when `remaining` gets low; on 429 wait `retry_after` then retry. For bulk lookups use `POST /v1/trademarks/batch` (up to 100 IDs = one request) instead of N individual GETs. Check usage: `GET /v1/organization/usage`.

## Idempotency

Mutating requests (`PATCH`, `DELETE`, non-exempt `POST`) **require** an `Idempotency-Key` header (1–255 chars of `[A-Za-z0-9_-]`). Same key + same body within 24h replays the cached 2xx response (`Idempotent-Replayed: true`); same key + different body → `409 conflict`.

Read-shaped POSTs are exempt (no key needed): `POST /v1/trademarks` (search), `/v1/trademarks/batch`, suggest endpoints, `/v1/watches/preview`, `/v1/alerts/lookup`.

## Endpoint map

| Task | Endpoint(s) |
|---|---|
| Search trademarks | `GET/POST /v1/trademarks` (same surface; POST for complex filters + aggregations) |
| Get one trademark (full detail) | `GET /v1/trademarks/{id}` — supports ETag/`If-None-Match`, `fields=`, `include=` |
| Bulk lookup | `POST /v1/trademarks/batch` (≤100 by `ids` or office+number `identifiers`) |
| Record history / diffs / related marks | `GET /v1/trademarks/{id}/history`, `/changes`, `/related` |
| Madrid coverage per designation | `GET /v1/trademarks/{id}/coverage` |
| Proceedings (oppositions etc.) | `GET /v1/trademarks/{id}/proceedings`, `GET /v1/proceedings` |
| Autocomplete | `GET /v1/trademarks/suggest?q=...`, cross-entity `GET /v1/suggest` |
| Clearance screening (risk verdict) | `GET /v1/screening?q=...` — `clear`/`caution`/`high_risk` verdict + risk-banded hits |
| Owners / attorneys / firms | `GET /v1/owners[/{id}]`, `/v1/attorneys[/{id}]`, `/v1/firms[/{id}]`, each with `/{id}/trademarks` |
| Resolved corporate entities | `GET /v1/entities/{id}`, `/v1/entities/{id}/trademarks`; search filters `entity_id`, `entity_group` |
| Monitoring | `/v1/watches` (+`/preview`, `/bulk`), `/v1/alerts` (+`/lookup`), `/v1/webhooks` (+`/test`, `/deliveries`) |
| Compute deadlines (no persistence) | `POST /v1/deadlines/compute`, `POST /v1/oppositions/compute` |
| Diff your data vs the register | `POST /v1/reconcile` |
| Reference data (unmetered) | `GET /v1/offices`, `/jurisdictions`, `/classifications`, `/design-codes`, `/event-types`, `/deadline-rules`, `/opposition-rules` |
| Account | `GET /v1/organization/me`, `/usage`, `/logs`; API key CRUD under `/v1/organization/api-keys` |

Scopes: catalog reads need `trademarks:read`; watches/alerts/webhooks/portfolios need `portfolios:manage`; key management needs `api-keys:manage`; usage/logs need `billing:read`. A `403 forbidden` means the key lacks the scope.

## Search essentials

Four strategies: `exact`, `fuzzy` (typo-tolerant, AUTO edit distance — there is no user-set fuzziness knob), `phonetic` (sound-alikes), `prefix`. Default is `exact,fuzzy`. **For comprehensive clearance searches, pass all four explicitly.** Results carry `relevance_score` (0–100, only present on relevance-sorted text search).

Aggregations (facet counts by `office_code`, `status_stage`, `nice_classes`, `filing_year`, etc.) are POST-only, under `options.aggregations`; add `"aggregations_only": true` to skip result documents.

Madrid grain: `international_registrations=grouped` (default; one row per mark with a `coverage` rollup) or `expanded` (one row per territorial-protection leg). The legacy `expand_ir` parameter is removed. Rows carry `owners[]` (the old `owner_name` field is gone).

For the full filter vocabulary (statuses, dates, classifications, entity filters, projections) read `references/search.md`, and for exact schemas fetch `https://docs.signa.so/guides/search.md`.

## Monitoring essentials

Watches (saved queries run on every ingestion sync) → alerts (immutable match records with severity + opposition deadline) → webhooks (Standard Webhooks HMAC-signed push). Five watch types: `mark`, `portfolio`, `owner`, `class`, `similarity`.

The single biggest gotcha: **watch-DSL filter keys are camelCase** (`niceClasses`, `trademarkIds`, `ownerId`) while REST search params are snake_case — the same vocabulary, different casing. The query body requires `version: "v2"`. Unknown or snake_case keys are rejected with `400`.

Always `POST /v1/watches/preview` before creating a watch to estimate alert volume. For the full DSL, webhook payloads, signature verification, and delivery-handling patterns read `references/monitoring.md`.

## Common workflows

**Clearance check** — knock out a candidate name fast: `GET /v1/screening?q=NAME&nice_classes=9,42` for the verdict; then a full four-strategy search on `/v1/trademarks` for the underlying record set. Screening output is not legal advice; present it as risk triage.

**Portfolio pull** — resolve the owner (`GET /v1/owners?q=...` or `/v1/suggest`), then `GET /v1/owners/{id}/trademarks` with auto-pagination. For corporate families (subsidiaries under one parent), use `entity_id`/`entity_group` filters on search — expect `422 entity_too_large` on giant groups; narrow with additional filters.

**Stay in sync** — poll search with `updated_at_gte=<checkpoint>`, or use watches + webhooks for push. Detail endpoints are immediately consistent; search lags writes by up to ~30s.

**Renewal/deadline tracking** — trademark records carry deadline fields; `POST /v1/deadlines/compute` computes maintenance deadlines (e.g. US §8/§9) for marks without persisting anything; `GET /v1/deadline-rules` lists per-jurisdiction rules.

## SDK notes

```typescript
import { Signa } from "@signa-so/sdk";
const signa = new Signa({ api_key: process.env.SIGNA_API_KEY }); // base_url, timeout, max_retries options
```

- Resource namespaces: `trademarks`, `owners`, `entities`, `attorneys`, `firms`, `proceedings`, `suggest`, `references`, `deadlines`, `oppositions`, `reconcile`, `portfolios`, `watches`, `alerts`, `webhooks`, `events`, `organization`.
- Auto-pagination: `for await (const tm of signa.owners.trademarks('own_...')) { ... }` — or manual paging via `.data` / `.has_more` / `.next_cursor`, or `.toArray()`.
- Built-in retry with exponential backoff for 429 (honors `Retry-After`), 5xx, and network errors.
- Typed errors for `instanceof`: `Signa.NotFoundError`, `Signa.RateLimitError` (has `retry_after`), `Signa.BadRequestError`, `Signa.AuthenticationError`, `Signa.PermissionError`, `Signa.ConflictError`, `Signa.InternalServerError`, plus `ConnectionError`/`TimeoutError`. Every API error exposes `.status`, `.error.type`, `.request_id`.
- Webhook verification helper: `verifyWebhookSignature` (Standard Webhooks).

Python and other languages: use plain HTTP (`requests`/`httpx`) with the conventions above — the API is a straightforward JSON REST surface.

## Gotchas checklist

- POST search filters nested under `filters`; GET search needs ≥1 filter.
- Comma-separated arrays, not `[]` brackets; literal `true`/`false` booleans; `_gte`/`_lt` date ranges.
- Watch DSL is camelCase + `version: "v2"`; REST search is snake_case.
- `Idempotency-Key` required on mutations, exempt on read-shaped POSTs.
- Cursors expire in 24h and are query-bound; checkpoint long syncs.
- `410 entity_merged` → follow `merged_into`; `422 entity_too_large` → narrow the query.
- Batch endpoint over N single GETs for bulk lookups.
- Reference-data endpoints are unmetered — don't cache-bust them needlessly, but feel free to call them.
- ETag/`If-None-Match` on detail endpoints saves quota on repeat reads (`304`).
- When in doubt about a parameter, fetch the live docs page (`.md` suffix) instead of guessing.
