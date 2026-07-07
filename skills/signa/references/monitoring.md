# Signa monitoring reference

Watches → alerts → webhooks. Live in beta. Requires the `portfolios:manage` scope. Always-current details: `https://docs.signa.so/guides/monitoring/overview.md` (and sibling pages under `guides/monitoring/`).

## Model

| Building block | What it is |
|---|---|
| **Watch** | A saved query Signa evaluates on every ingestion sync |
| **Alert** | Immutable record of one match: trademark + triggering event + severity + opposition deadline where applicable |
| **Webhook** | HMAC-signed `POST` to your URL per alert (Standard Webhooks spec) |

Delivery modes: webhook (push, seconds), polling `GET /v1/alerts` (pull), or webhook + a daily `POST /v1/alerts/lookup` reconciliation pass for workflows that can't tolerate a missed alert.

## Watch types

| `watch_type` | Track | Required field |
|---|---|---|
| `mark` | One specific mark (renewals, status drift) | `filters.trademarkIds` (one ID) |
| `portfolio` | A set of marks | `filters.trademarkIds` (1+) |
| `owner` | A competitor's filings | `filters.ownerId` |
| `class` | New filings in Nice class(es) | `filters.niceClasses` |
| `similarity` | Confusingly-similar new filings | `q` |

## The query DSL — camelCase, `version: "v2"`

```typescript
interface WatchQuery {
  version: "v2";                        // REQUIRED
  q?: string;                           // required for similarity; ≤20 keywords, each ≥3 chars, no stop words
  strategies?: ("exact"|"phonetic"|"fuzzy"|"prefix")[];  // default exact+fuzzy
  filters?: WatchFilters;               // camelCase keys
  trigger_events?: string[];            // non-empty subset of the five events
  min_match_tier?: "exact"|"normalized"|"fuzzy"|"phonetic";  // similarity gating
}
```

**Filter keys are camelCase** — the same vocabulary as REST search, translated: `niceClasses`, `trademarkIds`, `ownerId`, `ownerName`, `attorneyId`, `firmId`, `jurisdictions`, `offices`, `statusStage`, `statusPrimary`, `filingDate`, `markFeatureType`, `goodsServicesText`, `hasProceedings`, `isMadrid`, etc. Snake_case keys (`nice_classes`) are rejected with `400` (the error suggests the camelCase spelling). ID filters accept prefixed IDs or raw UUIDs; office codes are case-insensitive here; `jurisdictions` auto-translates to `offices` at create time.

**Rejected outright:** `sort`, `cursor`, `aggregations`, `highlight`, `function_score`, `script*` anywhere in the query; any `match` key; unknown filter keys; empty `trigger_events`.

**`trigger_events`** — default set when omitted: `trademark.created`, `trademark.updated`, `trademark.status_changed`. Opt-in extras: `trademark.retracted` / `trademark.corrected` (source-feed flips, not legal status). Cancellations arrive as `trademark.status_changed` (stage → `cancelled`, primary → `inactive`) — no special setup.

**`min_match_tier`** (similarity only) — `exact` ⊂ `normalized` (+ phrase/synonym/homoglyph) ⊂ `fuzzy` (+ fuzzy/prefix) ⊂ `phonetic` (broadest). Replaces the legacy `score_threshold`.

**`customer_reference`** — free-text label (≤200 chars) on the watch itself (top level, next to `name`/`query`), echoed frozen onto every alert as `data.customer_reference`. Use it for matter numbers / routing keys.

## Create, preview, bulk

```typescript
// Always preview first — estimates volume with the real evaluator
const preview = await signa.watches.preview({ query, trial_window_days: 30 });
// preview.estimated_match_count; estimate_basis === "candidacy_upper_bound" means upper bound, not exact.
// One preview per org at a time (429 + Retry-After on concurrency); 504 preview_timeout is NOT retryable — narrow the query.

await signa.watches.create({
  name: "Marks similar to ACME",
  watch_type: "similarity",
  customer_reference: "matter-2026-0481",
  query: {
    version: "v2",
    q: "ACME",
    strategies: ["exact", "fuzzy", "phonetic"],
    filters: { niceClasses: [9, 35], jurisdictions: ["US", "EU"] },
    min_match_tier: "fuzzy",
  },
});

await signa.watches.bulk({ watches: [...] });  // ≤100, all-or-nothing validation
```

Watch errors: `400` bad DSL / stop words / missing required field for the type; `409` plan watch limit; `413` query > 256 KB.

## Alerts

`GET /v1/alerts` filters: `watch_id`, `status` (`unacknowledged`/`acknowledged`/`dismissed`), `severity`, `trademark_id`, `event_type`. `PATCH /v1/alerts/{id}` acknowledges/dismisses. `POST /v1/alerts/lookup` bulk-fetches by IDs (idempotency-exempt; returns a bare list envelope with no pagination).

Alert fields to route on: `severity` (`normal`/`high`/`critical`), `must_act_by` (ISO deadline, usually opposition-window close), `opposition_window_status` (`open`/`closing_soon`/`critical`/`closed`/null). Opposition deadlines are computed per-jurisdiction where supported.

## Webhooks

```typescript
const wh = await signa.webhooks.create({
  url: "https://alerts.example.com/signa",
  enabled_events: ["alert.created"],   // the only subscribable event today
});
// wh.secret is returned ONCE — store it immediately.
```

Delivery envelope: `{ type, id, timestamp, data }`. For `alert.created`, `data` is self-contained — the full alert object (watch `{id,name,type}`, `event` with `summary` + `diff`, inline `trademark` snapshot, `deadline`, `customer_reference`) plus always-present legacy flat fields (`alert_id`, `watch_id`, `trademark_id`, `event_type`, `severity`, `must_act_by`). The rich half is best-effort — feature-detect (`if (data.event) {...} else { fetch via data.alert_id }`); the flat fields are guaranteed.

**Verification (Standard Webhooks HMAC-SHA256):**

- Headers: `webhook-id` (stable across retries — use as your idempotency key), `webhook-timestamp` (fresh per retry; reject if > 5 min old), `webhook-signature` (`v1,<base64>`; two space-separated entries during secret rotation), `webhook-attempt` (unsigned counter — never trust it for logic).
- **Verify against the raw request bytes.** The body is canonicalized JSON at signing time; if your framework re-parses/re-stringifies before verification, signatures will fail.
- SDK helper: `import { verifyWebhookSignature } from "@signa-so/sdk"`. Any Standard Webhooks library also works.

Retries: up to 7 attempts with exponential backoff. Repeated failures auto-disable the endpoint. `POST /v1/webhooks/{id}/test` sends a `webhook.test` ping (never retried, never counts toward auto-disable). `POST /v1/webhooks/{id}/rotate-secret` rotates with an overlap window; `GET /v1/webhooks/{id}/deliveries` + `/deliveries/{id}/replay` for inspection and redelivery.

**Receiver checklist:** verify signature on raw bytes → dedupe on `webhook-id` → respond 2xx fast (enqueue, don't process inline) → reconcile daily via `POST /v1/alerts/lookup` if missing an alert is unacceptable.

## Troubleshooting

"Why didn't my alert fire?" — use the watch diagnostics endpoint (`GET /v1/watches/{id}/diagnostics`; see `https://docs.signa.so/guides/monitoring/troubleshooting.md`). Alert latency varies by office sync cadence (daily for most offices, weekly for some) — see `https://docs.signa.so/guides/data-freshness.md`.
