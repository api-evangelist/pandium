---
name: pandium-trigger-and-monitor-sync
description: Trigger an integration sync for a Pandium tenant and monitor it to completion, reading the run log on failure.
api: Pandium Runs API
generated: '2026-09-03'
method: generated
source: openapi/pandium-runs-api-openapi.yml + https://docs.pandium.com/reference/pandium-api
operations:
- triggerTenantSync
- get_run_status_from_trigger_v2_runs_triggers__trigger_id__status_get
- listTenantRuns
- get_run_log_v2_runs__run_id__log_get
---

# Trigger and monitor a Pandium integration sync

Run a sync for a tenant (a customer's installed copy of an integration) and follow it to a
terminal state.

## Prerequisites

- A Pandium API key, generated in the Integration Hub under Settings > API Access. Keys are only
  viewable at creation. Send it on every request in the `X-API-KEY` header.
- Base URL: `https://api.pandium.io` (production) or `https://api.sandbox.pandium.com` (sandbox).
- The `tenant_id` of the tenant to sync (list tenants with `GET /v2/tenants` if needed).

## Steps

1. **Trigger the sync** — `POST /v2/tenants/{tenant_id}/sync` (`triggerTenantSync`). The `mode`
   query parameter selects the run mode (e.g. `init` vs a normal sync). The response is a
   trigger object; keep its `trigger_id`.
2. **Poll the trigger status** — `GET /v2/runs/triggers/{trigger_id}/status`
   (`get_run_status_from_trigger_v2_runs_triggers__trigger_id__status_get`). Poll until the run
   reaches a terminal status. Sync triggers are debounced per tenant, so a re-trigger while a run
   is pending may not start a second run.
3. **On failure, read the log** — find the run via `GET /v2/tenants/{tenant_id}/runs`
   (`listTenantRuns`, paginated with `limit`/`skip`/`sort_by`), then fetch
   `GET /v2/runs/{run_id}/log` (`get_run_log_v2_runs__run_id__log_get`) for the integration's
   stderr log output.

## Rules and cautions

- **No idempotency mechanism exists** on this API: a repeated `POST .../sync` after a network
  timeout may enqueue another run (subject only to per-tenant debouncing). Check trigger status
  before retrying.
- **A triggered run cannot be cancelled** through the public API — do not trigger syncs
  speculatively.
- Errors come back as a custom `{"message": "..."}` envelope; validation failures are 422 with a
  FastAPI `detail[]` shape; a missing/invalid key returns 403 `{"detail": "Not authorized"}`.
- No rate limits are documented; be conservative when polling (a few seconds between status
  checks is plenty for integration runs).
- Alternatively, `run_failed` webhook notifications (configured in Administrator Settings) push
  failure events with direct `api_link`/`admin_link` URLs instead of polling.
