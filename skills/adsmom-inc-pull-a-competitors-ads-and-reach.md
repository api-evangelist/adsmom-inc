---
generated: '2026-08-13'
method: generated
name: Pull a competitor's ads and their reach
description: Page through a tracked competitor's Meta ads, hydrate the ones that matter in batches of 25, and pull daily reach timeseries — while handling signed creative URLs that expire in about ten minutes.
api: openapi/adsmom-inc-openapi.json
operations: [listMetaAds, getMetaAd, batchMetaAdReach, getMetaAdReach, listTiktokAds, listTiktokAdSnapshots, getTiktokAdAtSnapshot]
source: >-
  Grounded in openapi/adsmom-inc-openapi.json (OpenAPI 3.0.0, harvested verbatim
  from https://api.adsmom.com/api/v1/openapi.json). Every operationId, parameter
  name, enum value and cap below is verified verbatim in the spec.
---

# Pull a competitor's ads and their reach

The Explore surface is where the 200M+ ad index is actually queried. This flow gets from "an advertiser I track" to "a set of ads with creative and a reach curve".

## Auth
- `Authorization: Bearer <JWT>` (scope `api:read`). Base URL `https://api.adsmom.com`.

## Steps

1. **List the ads** — `listMetaAds` (`GET /api/v1/explore/meta/ads`). Real parameters, verbatim from the spec:
   - `advertiser_id` — filter to one advertiser by **id token**
   - `query` — free-text search across creative text
   - `active` — `true` | `false`, active vs ended
   - `started_after` — created on/after (ISO 8601); `updated_after` — ingested on/after
   - `format` — `video` | `image`
   - `sort` — `newest` | `oldest` | `reach_desc`
   - `cursor` — opaque pagination cursor; `limit` — default **25**, maximum **25**
   Returns a bare JSON **array** of `MetaAdSummary`.

2. **Page carefully.** The response is an array with no envelope, so **no `next_cursor` is returned or declared anywhere**. Cursor pagination is an input with no documented output (`conventions/adsmom-inc-conventions.yml`). Until Adsmom documents it, page defensively: request `limit=25`, and if you receive a full page, treat the set as possibly incomplete rather than assuming you have everything. Do not fabricate a cursor.

3. **Batch-hydrate rather than looping single reads** — `listMetaAds` also accepts `ids`: "Batch-hydrate ad id tokens (comma-separated, max 25). When set, filters and pagination are ignored." Twenty-five ids per call, one request. Use `getMetaAd` (`GET /api/v1/explore/meta/ads/{id_token}`) only when you need one ad's full detail.

4. **Ask for targeting explicitly** — on `getMetaAd`, `targeting` is populated "Only with `include=full_targeting`; null otherwise". If you do not pass it, the field is `null` and that is not an error.

5. **Pull reach** — `batchMetaAdReach` (`GET /api/v1/explore/meta/ads/reach`) for many ads in one call (the sync path), or `getMetaAdReach` (`GET /api/v1/explore/meta/ads/{id_token}/reach`) for one ad's daily timeseries. Both return `MetaReachPoint` series.

6. **TikTok adds time travel.** `listTiktokAds` mirrors `listMetaAds`, and TikTok ads additionally support `listTiktokAdSnapshots` (`GET /api/v1/explore/tiktok/ads/{id_token}/snapshots`) and `getTiktokAdAtSnapshot` (`.../snapshots/{snapshot_uuid}`) — point-in-time captures with `captured_at`, `reach_upper_bound`, `reach_lower_bound` and `region_reach`. LinkedIn has the same pair. **Meta and Google do not** — they expose current state only.

## Rules
- **Creative URLs expire.** `media_url` and `thumbnail_url` are signed and short-lived (~10 minutes) — the spec says "Fetch promptly or re-request." Never store these URLs in a database, a report, or a message to another system. Persist the ad `id_token` and re-hydrate when you need the asset.
- **`reach_total` is Meta's number, not Adsmom's** — "Lifetime breakdown reach from the Meta Ad Library."
- **`end_date` is null while the ad is still running** — "Delivery stop date; null while active."
- **LinkedIn reports impression brackets, not values** (`LinkedinImpressionsPoint`); do not sum them as if they were counts.
- **Google has no per-ad reach path.** Its impression data lives in Analytics (`getGoogleActivity`).

## Budget
- No `RateLimit-*` headers exist. Read `rate_limit.requests_per_minute` and `credits.remaining` from `getUsage` before a large crawl — see `rate-limits/adsmom-inc-rate-limits.yml`. Batch-hydration (25 ids per request) is the cheapest way to cover a lot of ads.

## Errors
- `401` → RFC 9457 problem+json with `request_id`. No `4xx` is declared on any operation; a bad `format`/`sort` enum value or a `limit` above 25 will fail in a way the contract does not describe. See `errors/adsmom-inc-problem-types.yml`.
