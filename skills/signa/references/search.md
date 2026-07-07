# Signa search reference

Full filter vocabulary and patterns for `GET/POST /v1/trademarks`. For the authoritative, always-current schema fetch `https://docs.signa.so/guides/search.md` and `https://api.signa.so/v1/openapi.json`.

## GET vs POST

Both hit the same service and return the same shape. GET takes flat query params; POST takes a JSON body with filters nested under `filters`. Aggregations (`options.aggregations`, `options.aggregations_only`) are POST-only. Everything else — strategies, pagination, `include_total`, `sort`, projections — works on both.

```json
POST /v1/trademarks
{
  "query": "signa",
  "strategies": ["exact", "phonetic", "fuzzy", "prefix"],
  "filters": {
    "offices": ["uspto", "euipo"],
    "nice_classes": [9, 42],
    "status_stage": ["registered"],
    "filing_date": { "gte": "2020-01-01", "lt": "2026-01-01" }
  },
  "options": { "aggregations": ["office_code", "status_stage"], "include_total": true },
  "sort": "-filing_date",
  "limit": 50
}
```

On GET the same query is: `q=signa&strategies=exact,phonetic,fuzzy,prefix&offices=uspto,euipo&nice_classes=9,42&status_stage=registered&filing_date_gte=2020-01-01&filing_date_lt=2026-01-01&sort=-filing_date&limit=50`.

## Strategies

| Strategy | Behavior |
|---|---|
| `exact` | Case-insensitive exact keyword match on the whole mark text |
| `fuzzy` | Typo tolerance; edit distance is always AUTO (no fuzziness knob) |
| `phonetic` | Sound-alikes regardless of spelling (SIGNA / CYGNA / SYNNA) |
| `prefix` | Marks starting with the query text |

Default when omitted: `exact,fuzzy`. Clearance searches should pass all four.

## Filters

**Status** — `status_primary` (`pending` \| `active` \| `inactive` \| `unknown`), `status_stage` (18 stages: `filed`, `examining`, `pending_publication`, `published`, `opposition_period`, `pending_opposition`, `pending_cancellation`, `pending_issuance`, `registered`, `allowed`, `abandoned`, `withdrawn`, `surrendered`, `refused`, `cancelled`, `invalidated`, `expired`, `unknown`), `status_reason`, `challenge_states`.

**Geography** — `offices` (lowercase codes; `GET /v1/offices` lists live ones), `jurisdictions` (uppercase ISO), `owner_country`.

**Classification** — `nice_classes` (1–45), `vienna_codes`, `goods_services_text` (text match within G&S).

**Identifiers** — `application_number` + `office` (required together), `registration_number` + `office`, `ir_number` (globally unique, no office needed).

**Dates** (all `_gte`/`_lt`) — `filing_date`, `registration_date`, `expiry_date`, `renewal_due_date`, `publication_date`, `termination_date`, `updated_at`. Also `next_deadline_before`.

**Mark shape** — `mark_feature_type` (word, figurative, …), `mark_legal_category`, `right_kind`, `scope_kind`, `filing_route` (`direct_national`, `direct_regional`, `madrid_ir`, `madrid_designation`, `transformation`, `divisional`).

**Parties** — `owner_id` (`own_*`), `owner_name` (fuzzy), `attorney_id`, `firm_id`, `entity_id` (`ent_*` resolved entity — matches across all member owners), `entity_group` (GLEIF corporate family walk).

**Booleans** — `has_media`, `has_proceedings`, `is_madrid`, `is_retracted`, `is_series_mark`.

**Madrid grain** — `international_registrations=grouped` (default; one row per mark, `coverage` rollup, `mark_count` total) or `expanded` (one row per territorial-protection leg, `territorial_protection_count` total). Some combinations (Vienna/G&S text filters, aggregations, custom sort, attorney-firm-name filter) can't be served grouped — the response falls back to `expanded` and reports it in `search_meta.fallback_reason`. Check `search_meta.international_registrations` for the mode actually served. (`expand_ir` is the removed predecessor — never use it.)

## Projections

- `fields=` — sparse top-level field selection (`fields=mark_text,status,owners`); `id` and `object` always kept; no nested paths.
- `include=` — whitelisted extras. On list/search rows: `full_goods_services` (untruncated `classifications[].goods_services_text`; rows otherwise truncate and set `goods_services_text_truncated: true`). On detail: `office_extensions` (raw office-specific blob), `history` (embeds up to 50 events).

## Aggregations (POST-only)

Fields: `office_code`, `jurisdiction_code`, `nice_classes`, `status_stage`, `filing_year`, `mark_feature_type`, `mark_legal_category`, `filing_route`, `right_kind`, `scope_kind`. Use `"aggregations_only": true` for counts without documents (fast market-structure queries).

## Scoring & sorting

Relevance-sorted text searches include `relevance_score` (0–100 normalized) and `match_explanation`. Supplying an explicit `sort` drops those fields (absent, not null). Default sort for non-text queries: `-filing_date`. Max 3 sort fields; `id` is the implicit tiebreaker.

## Suggest

- `GET /v1/trademarks/suggest?q=sig&limit=10` — mark-text typeahead (q ≥ 2 chars; filters: `jurisdictions`, `nice_classes`, `status_stage`).
- `GET /v1/suggest?q=nik&entity_type=owner` — cross-entity typeahead (`trademark` \| `owner` \| `attorney` \| `firm` \| `entity`, q ≥ 3 chars).

## Screening

`GET /v1/screening?q=NAME` — synchronous knockout check returning a `screening` verdict block (`clear` \| `caution` \| `high_risk`) plus `data[]` of hits with risk bands (`high`/`medium`/`low`, never numeric scores). Params: `q` (2–200 chars), `nice_classes`, `jurisdictions`/`offices`, `include=live|all` (live-only vs including dead marks), `sensitivity=strict|standard|broad`, `limit` (1–50). Treat output as risk triage, not legal advice.

## Recipes

**Comprehensive clearance sweep:**

```typescript
const hits = await signa.trademarks.search({
  query: candidateName,
  strategies: ["exact", "phonetic", "fuzzy", "prefix"],
  filters: { nice_classes: relevantClasses, status_primary: "active" },
  options: { include_total: true },
});
```

Then widen: drop `status_primary` to catch recently-dead blocking marks (they can be revived or evidence prior use), and re-run per key jurisdiction.

**Market scan without documents:**

```json
{ "query": "vegan leather", "filters": { "nice_classes": [18] },
  "options": { "aggregations": ["office_code", "filing_year"], "aggregations_only": true } }
```

**Incremental sync:** page with `updated_at_gte=<last checkpoint>&sort=updated_at`, checkpoint the max `updated_at` you've processed, restart from the checkpoint if the cursor expires (24h).
