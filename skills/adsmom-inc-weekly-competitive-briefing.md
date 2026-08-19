---
generated: '2026-08-13'
method: generated
name: Build a weekly competitive briefing
description: Assemble a scheduled Meta competitive briefing from Adsmom's AI advertiser summary, the weekly report, activity counts, share-of-voice concentration and top ads by reach.
api: openapi/adsmom-inc-openapi.json
operations: [getMetaAdvertiserSummary, getMetaAdvertiserWeeklyReport, getMetaActivity, getMetaShareOfVoice, getMetaTopAds, getMetaReach, getMetaReachRegions, getMetaTargeting, getUsage]
source: >-
  Grounded in openapi/adsmom-inc-openapi.json (OpenAPI 3.0.0, harvested verbatim
  from https://api.adsmom.com/api/v1/openapi.json). Every operationId is verified
  verbatim in the spec. Schema-stability warnings are quoted from the spec's own
  field descriptions.
---

# Build a weekly competitive briefing

The scheduled-job flow Adsmom's REST API exists for: once a week, per tracked advertiser, produce a readable competitive brief. This is the flow with the most AI-generated content in it, and therefore the one that needs the most defensive handling.

## Auth
- `Authorization: Bearer <JWT>` (scope `api:read`). Base URL `https://api.adsmom.com`.

## Steps

1. **Check credits first** — `getUsage` (`GET /api/v1/usage`). The AI operations in this flow consume credits, and `credits.limit` is plan-bound (Basic 250/mo, Business 1 000/mo, Custom 10 000/mo). `credits.period_start`/`period_end` give you the billing window, so a weekly job can pace itself against the period rather than the calendar.

2. **Pull the AI advertiser summary** — `getMetaAdvertiserSummary` (`GET /api/v1/insights/meta/advertisers/{id_token}/summary`). Returns an `AdvertiserInsight`: `status`, `generated_at`, `total_ads_analyzed`, plus `page_summary`, `ad_analysis`, `initial_assessment` and `recommendations`. **Check `status` before rendering** — the spec describes it as "Generation status (e.g. completed, pending)". A `pending` summary is not a failure; re-poll.

3. **Pull the weekly report** — `getMetaAdvertiserWeeklyReport` (`GET /api/v1/insights/meta/advertisers/{id_token}/reports/weekly`), with the `week_start` query parameter to select the week. Returns `week_start`, `week_end`, `created_at`, `report`, `metrics` and `previous_metrics` — `previous_metrics` is what lets you write week-over-week deltas without keeping your own history.

4. **Add the hard numbers.** These are deterministic chart operations, not AI, and they take `advertiser_ids`, `date_from`/`date_to` and `granularity`:
   - `getMetaActivity` (`GET /api/v1/analytics/meta/activity`) — new + active ad counts over time. The launch/pause signal.
   - `getMetaShareOfVoice` (`GET /api/v1/analytics/meta/share-of-voice`) — reach concentration per advertiser, expressed as a Lorenz curve with a Gini coefficient (`ConcentrationPoint`). Read it as "how much of this advertiser's reach comes from how few ads", not as market share.
   - `getMetaTopAds` (`GET /api/v1/analytics/meta/top-ads`) — top ads by reach, for the creative exhibit.
   - `getMetaReach` and `getMetaReachRegions` — reach over time and by region.
   - `getMetaTargeting` — targeted vs excluded regions.

5. **Assemble.** Lead with the deterministic numbers from step 4, use the AI text from steps 2–3 as commentary, and attribute it as AI-generated in the output.

## Rules
- **The AI fields are untyped by design.** `page_summary`, `ad_analysis`, `initial_assessment`, `recommendations` and `report` are all declared as `object` with **no properties** and the spec's own caveat: "AI-generated; structure may evolve." Do not bind a typed client to them, do not assume a key is present, and do not let a schema change break the whole briefing — render defensively.
- **Analytics results are read models, not resources.** The 23 `*Chart` schemas have no id and cannot be fetched again by reference. Persist the values if you need them later.
- **Share of voice is concentration, not market share.** Lorenz/Gini describes distribution within the set of advertisers you queried — change `advertiser_ids` and the number changes.
- **The set is what you track.** Every Analytics operation is scoped to tracked advertisers; an untracked competitor is invisible, not zero. Run `skills/adsmom-inc-start-tracking-a-competitor.md` first.

## Scheduling
- No webhooks and no AsyncAPI — Adsmom publishes no event surface for the REST API, so a weekly briefing is a **poll**, not a subscription. Align the job to `week_start` boundaries and re-poll step 2 while `status` is pending.
- No `Retry-After` header and no declared `429`. Back off on your own schedule and read `getUsage` between runs.

## Errors
- `401` → RFC 9457 problem+json carrying `request_id`. No `4xx`/`5xx` is declared on any operation; treat every non-2xx as unmodelled and log `x-request-id`. See `errors/adsmom-inc-problem-types.yml`.
