---
generated: '2026-08-13'
method: generated
name: Analyze TikTok creative strategy
description: Read a tracked competitor's TikTok creative mix, optimal ad runtime and regional targeting overlap, then drill from the aggregate breakdown down to the individual ads behind it.
api: openapi/adsmom-inc-openapi.json
operations: [getTiktokCreative, getTiktokCreativeDimension, getTiktokRuntime, getTiktokTargeting, getTiktokShareOfVoice, getTiktokActivity, listTiktokAds, getTiktokAd, listTiktokAdSnapshots]
source: >-
  Grounded in openapi/adsmom-inc-openapi.json (OpenAPI 3.0.0, harvested verbatim
  from https://api.adsmom.com/api/v1/openapi.json). Every operationId is verified
  verbatim in the spec. TikTok carries the richest analytics surface of the four
  platforms — 9 operations, versus 7 for Meta and Google and 0 for LinkedIn.
---

# Analyze TikTok creative strategy

TikTok is the only platform in the Adsmom API with a creative-mix breakdown *and* a runtime distribution *and* snapshots. That combination answers the question a growth team actually asks: what kind of creative is this competitor making, how long do they let it run, and where.

## Auth
- `Authorization: Bearer <JWT>` (scope `api:read`). Base URL `https://api.adsmom.com`.

## Steps

1. **Get the creative mix** — `getTiktokCreative` (`GET /api/v1/analytics/tiktok/creative`). Returns the mix across hook, format and audio — the three dimensions the spec names — as `CreativeMixEntry` series.

2. **Drill into one dimension** — `getTiktokCreativeDimension` (`GET /api/v1/analytics/tiktok/creative/{dimension}`). `{dimension}` is a path parameter with no enum declared in the spec; the dimensions named in `getTiktokCreative`'s own summary are hook, format and audio. Because the values are not enumerated, treat an unexpected `{dimension}` as unmodelled rather than assuming it will 400 predictably.

3. **Read the runtime distribution** — `getTiktokRuntime` (`GET /api/v1/analytics/tiktok/runtime`). Returns `RuntimeBucket` series plus `TiktokOptimalRuntimeEntry` — Adsmom's own read on the optimal runtime. This is the "how long do they let a creative live" number. Google has the same pair (`getGoogleRuntime`); Meta and LinkedIn do not.

4. **Read targeting overlap** — `getTiktokTargeting` (`GET /api/v1/analytics/tiktok/targeting`) for regional targeting overlap across the advertisers you queried, and `getTiktokRegions` for available regions by ad count.

5. **Frame it against the set** — `getTiktokShareOfVoice` (`GET /api/v1/analytics/tiktok/share-of-voice`) for share of voice across the queried advertisers, and `getTiktokActivity` (`GET /api/v1/analytics/tiktok/activity`) for new + active ad counts over time.

6. **Drill from aggregate to instance** — the Analytics operations return charts with no ad ids. To see the actual creative behind a bucket, go to `listTiktokAds` (`GET /api/v1/explore/tiktok/ads`) with `format`, `sort=reach_desc` and `advertiser_id`, then `getTiktokAd` for detail. There is **no** identifier linking a chart bucket to the ads that produced it — the join is yours to make on the filter parameters, and it is approximate.

7. **Check whether creative changed mid-flight** — `listTiktokAdSnapshots` (`GET /api/v1/explore/tiktok/ads/{id_token}/snapshots`) then `getTiktokAdAtSnapshot` (`.../snapshots/{snapshot_uuid}`). Each `TiktokAdSnapshot` carries `captured_at`, `is_active`, `last_shown_date`, `reach_upper_bound`, `reach_lower_bound` and `region_reach`. This is the only way to see an ad's earlier state.

## Rules
- **Reach is a bracket on TikTok.** `TiktokAdSnapshot` gives `reach_upper_bound` and `reach_lower_bound`, not a value. Report a range; never average the bounds and present the result as a measurement.
- **Every Analytics result is scoped to the advertisers you pass** via `advertiser_ids`, and only tracked advertisers are eligible. Widening or narrowing the set changes share of voice and targeting overlap.
- **`{dimension}` is unenumerated.** Read the dimension names out of the `getTiktokCreative` response rather than hard-coding a list.
- **Charts are read models.** The `*Chart` schemas have no id and are not re-fetchable by reference (`data-model/adsmom-inc-data-model.yml`).
- **Signed media expires in ~10 minutes** on any ad detail you hydrate in step 6.

## Errors
- `401` → RFC 9457 `application/problem+json` with `type`/`title`/`status`/`detail`/`instance`/`request_id`. No `4xx` is declared anywhere in the spec, so a bad `{dimension}` or an untracked `advertiser_ids` value fails in an undocumented way. Log `x-request-id`. See `errors/adsmom-inc-problem-types.yml`.
