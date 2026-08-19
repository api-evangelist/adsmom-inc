---
generated: '2026-08-13'
method: generated
name: Start tracking a competitor
description: Add a competitor's Meta or TikTok advertiser to the Adsmom tracked set, confirm the ingest crawl has completed, and check the tracked-advertiser budget before spending a slot.
api: openapi/adsmom-inc-openapi.json
operations: [getUsage, trackMetaAdvertiser, listMetaAdvertisers, getMetaAdvertiser, trackTiktokAdvertiser, untrackMetaAdvertiser]
source: >-
  Grounded in openapi/adsmom-inc-openapi.json (OpenAPI 3.0.0, harvested verbatim
  from https://api.adsmom.com/api/v1/openapi.json). Every operationId was
  verified verbatim in the spec. Auth per authentication/adsmom-inc-authentication.yml,
  conventions per conventions/adsmom-inc-conventions.yml, limits per
  rate-limits/adsmom-inc-rate-limits.yml.
---

# Start tracking a competitor

Nothing in Adsmom's Explore, Insights or Analytics surface returns data for an advertiser you are not tracking. Tracking is the gate, tracked slots are a paid entitlement, and ingest is asynchronous — so this is the first flow any integration runs.

## Auth
- `Authorization: Bearer <JWT>`, token from `https://app.adsmom.com/oauth/token` (`client_credentials` for server-to-server). See `authentication/adsmom-inc-authentication.yml`.
- Base URL `https://api.adsmom.com`. The published spec declares `servers: [{url: "/"}]` and names no host — use the base above, recorded in `overlays/adsmom-inc-openapi-overlay.yaml`.

## Steps

1. **Check the budget before you spend a slot** — `getUsage` (`GET /api/v1/usage`). Returns `plan`, `tracked_advertisers` (how many are active now), `credits` (`used`/`limit`/`remaining`/`period_start`/`period_end`) and `rate_limit.requests_per_minute`. Tracked-advertiser ceilings are per plan (Basic 5, Business 50, Custom 200+ — `plans/adsmom-inc-plans.yml`). There is no `RateLimit-*` response header on this API, so this operation is the only way to read your own limits.

2. **Check whether you already track them** — `listMetaAdvertisers` (`GET /api/v1/insights/meta/advertisers`). Match on `name` or `external_id` (the platform-native page id, "when resolved"). Tracking a duplicate wastes a slot and there is no idempotency mechanism to protect you — see step 5.

3. **Track the advertiser** — `trackMetaAdvertiser` (`POST /api/v1/insights/meta/advertisers`). The body is a `MetaTrackBody` with a single required field, `query`: "Facebook page name, handle, or numeric page id". Prefer the numeric page id when you have it; a name is ambiguous and Adsmom resolves it server-side. Returns `201` with a `TrackedAdvertiser`. For TikTok use `trackTiktokAdvertiser` (`POST /api/v1/insights/tiktok/advertisers`) — same shape, `TiktokTrackBody`.

4. **Wait for ingest, then confirm** — `getMetaAdvertiser` (`GET /api/v1/insights/meta/advertisers/{id_token}`). Poll until `pending_ingest` is `false`; the spec says it is "True when the ingest crawl has not completed yet". Until then, `listMetaAds` for this advertiser will be thin or empty and analytics will be misleading. Use the `id_token` (slash-free) from step 3 in the path — **not** `gid`, which contains a slash.

5. **Undo, if you mis-tracked** — `untrackMetaAdvertiser` (`DELETE /api/v1/insights/meta/advertisers/{id_token}`), returns `204`.

## Rules
- **No idempotency.** There is no `Idempotency-Key` header on any of the 78 operations (`conventions/adsmom-inc-conventions.yml`). A retried `trackMetaAdvertiser` after a network timeout has no documented dedupe behaviour — always re-run step 2 before retrying step 3.
- **Two ids per object.** `gid` is canonical but contains a slash; `id_token` is the slash-free form and is the only thing that goes in a path segment. See `data-model/adsmom-inc-data-model.yml`.
- **Slots are concurrent, not cumulative.** Untracking frees the slot; it does not refund AI credits already spent.

## Errors
- `401` returns RFC 9457 `application/problem+json` with `type`, `title`, `status`, `detail` (e.g. `missing_token`), `instance` and a `request_id` that matches the `x-request-id` header. Quote `request_id` in any support conversation.
- **The spec declares no error responses at all** — only `200`/`201`/`204`. A `403` (missing `api:write` scope), a failed resolution of `query`, or a tracked-advertiser cap breach are all undeclared. Handle non-2xx defensively and branch on `content-type`: routed errors are `application/problem+json`, unrouted paths fall through to an Express `{"message","error","statusCode"}` body. See `errors/adsmom-inc-problem-types.yml`.
