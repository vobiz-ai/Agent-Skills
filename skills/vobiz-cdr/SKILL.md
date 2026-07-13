---
name: vobiz-cdr
description: Query Vobiz Call Detail Records - list, search, retrieve by call_id, fetch recent, and export CSV. 43 fields per CDR including quality metrics (MOS, jitter, packet loss) and hangup attribution.
---

# Vobiz CDR skill

Use this for **call history, billing reconciliation, quality analysis, or analytics**. A CDR is written after a call completes - it is not available mid-call. Poll the list/recent endpoints, or use trunk webhooks for real-time events.

Base URL: `https://api.vobiz.ai`. Auth headers on every request: `X-Auth-ID` and `X-Auth-Token`.

## Endpoints

| Op | Path | Notes |
|---|---|---|
| List | `GET /api/v1/Account/{auth_id}/cdr` | Filter by from/to number, date range, direction, min_duration; paginated; includes `summary` |
| Search | `GET /api/v1/Account/{auth_id}/cdr/search` | Same filters + adds a `filters` echo of the active filter values |
| Recent | `GET /api/v1/Account/{auth_id}/cdr/recent` | Last 20 by default (`limit` to change). No `pagination`, no `summary`, no `filters` |
| Export | `GET /api/v1/Account/{auth_id}/cdr/export` | Returns `text/csv`, not JSON. Same filters as list (no `page`/`per_page`) |
| Get one | `GET /api/v1/Account/{auth_id}/cdr/{call_id}` | Path param is named `call_id`; pass the call's `uuid` from a list/webhook |

`auth_id` is your master account ID (`MA_XXXXXXXX`). The path segment is `Account` (capital A) followed by lowercase `cdr` - match the casing exactly.

## Response shape

Top-level (list): `{ account_id, count, data, pagination, summary, success }`.
Search adds `filters`. Recent returns only `{ account_id, count, data, success }`. Get-one returns `{ data, success }` where `data` is a single object (not an array).

`pagination`: `{ page, per_page, total, pages, has_next, has_prev }` (NOT `current_page` / `total_records`). `total` is the count across all pages; `pages = ceil(total / per_page)`.

`summary`: `{ totalCalls, answeredCalls, answerRate, avgCallDuration, total_duration_seconds, total_billable_seconds, total_cost, last_call_at }`. Note `avgCallDuration` is a **string** like `"28s"`, not a number. The summary reflects the **filtered** set, not the whole account.

`filters` (search only): `{ call_direction, from_number, hangup_cause, to_number }`. Empty values come back as `""`, not omitted.

Each CDR object has **43 fields**. See the field glossary below.

## Filters that work

Apply to list, search, and export (export ignores `page`/`per_page`):

- `from_number` - originating number; substring or full E.164 (example value `9876543210`)
- `to_number` - destination number; substring or full E.164
- `start_date` - `YYYY-MM-DD`. **Required together with** `end_date`
- `end_date` - `YYYY-MM-DD`. **Required together with** `start_date`
- `call_direction` - enum: `inbound` or `outbound` (no other values)
- `min_duration` - integer seconds; excludes calls shorter than this
- `page` - default `1` (list, search only)
- `per_page` - default `20`, max `100` (list, search only)

Recent takes only `limit` (default `20`).

## Field glossary (CDR object)

Identity & routing: `id` (integer, internal), `uuid` (string, globally unique - use this for get-one), `account_id`, `call_direction` (`inbound`/`outbound`), `caller_id_name`, `caller_id_number` (E.164, NOT `from_number`), `destination_number` (NOT `to_number`), `context`, `region`, `origination_region`, `campaign_id` (nullable), `customer_endpoint` (nullable), `trunk_id` (nullable), `terminated_to` (nullable).

Timing & duration: `duration` (integer seconds, ring + talk), `billsec` (integer billable seconds, talk only), `ring_time`, `answer_time` (nullable - null if unanswered), `start_time`, `end_time`, `progress_time`, `created_at`, `updated_at`. Timestamps are ISO 8601 (`...Z`).

Billing: `cost`, `total_cost` (cost + streaming), `streaming_cost`, `currency` (e.g. `INR`).

Quality metrics: `mos` (1.0-5.0, higher is better; `0` on unanswered/failed), `jitter` (ms), `packet_loss` (%), `codec` (`PCMU`/`PCMA`/`opus`, nullable). All quality fields are nullable and may be `0` for calls that never carried media.

Hangup attribution: `hangup_cause` (string, e.g. `NORMAL_CLEARING`), `hangup_cause_code` (integer, e.g. `4000`), `hangup_cause_name` (e.g. `Normal Hangup`), `hangup_disposition` (`send_bye`/`recv_bye`/`send_cancel`...), `hangup_source` (`Caller`/`Callee`/`Carrier`/...), `failure_code` (nullable), `failure_reason` (nullable).

SIP / network: `bridge_uuid` (nullable), `sip_call_id`, `sip_user_agent` (nullable), `network_addr` (nullable), `carrier_ip` (nullable).

## Recipes

List a date range, page 2, 50/page:
```bash
curl -G "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr" \
  --data-urlencode "start_date=2026-03-01" \
  --data-urlencode "end_date=2026-03-17" \
  --data-urlencode "page=2" --data-urlencode "per_page=50" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN"
```

Search outbound calls from one number, echo the filters:
```bash
curl -G "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr/search" \
  --data-urlencode "call_direction=outbound" \
  --data-urlencode "from_number=919876543210" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN"
```

Export filtered CDRs to a CSV file (no `Accept: application/json`):
```bash
curl -G "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr/export" \
  --data-urlencode "start_date=2026-03-01" --data-urlencode "end_date=2026-03-17" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" \
  -o cdrs.csv
```

Walk every page until `has_next` is false (jq):
```bash
page=1
while :; do
  resp=$(curl -sG "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr" \
    --data-urlencode "page=$page" --data-urlencode "per_page=100" \
    -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN")
  echo "$resp" | jq -c '.data[]'
  [ "$(echo "$resp" | jq '.pagination.has_next')" = "true" ] || break
  page=$((page+1))
done
```

Get a single CDR by its uuid (taken from `data[].uuid`):
```bash
curl "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr/$CALL_UUID" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN"
```

## Pitfalls

- **Casing:** the path is `/api/v1/Account/{auth_id}/cdr` - `Account` is capitalized, `cdr` is lowercase. Lowercasing `Account` or capitalizing `cdr` will 404/401.
- **get-cdr uses the uuid, not the numeric id:** the path param is *named* `call_id`, but pass the call's `uuid` (from `data[].uuid` or a webhook). Feeding back the numeric `data[].id` returns `404`.
- **Export is not JSON:** the export endpoint returns `text/csv`. Don't send `Accept: application/json`; pipe to a file with `curl -o cdrs.csv`. `page`/`per_page` are ignored - filter with `start_date`/`end_date`/`min_duration` to limit rows.
- **Date filters are paired:** `start_date` and `end_date` must be supplied together, both `YYYY-MM-DD`. A lone date may be ignored.
- **Field name drift:** it's `caller_id_number`/`destination_number` on the CDR object, but the *filter params* are `from_number`/`to_number`. Don't filter on `caller_id_number`.
- **`avgCallDuration` is a string** (`"28s"`) - parse it, don't do math on it directly. For numeric aggregates use `total_duration_seconds` / `total_billable_seconds`.
- **Recent has no pagination or summary:** only `account_id`, `count`, `data`, `success`. Use list if you need totals or paging.
- **Empty result set:** a valid request that matches nothing returns `200` with `data: []`, `count: 0`, and a zeroed `summary` (not a 404).
- **Quality fields can be `0` or null** on unanswered/failed calls - treat `mos: 0` as "no media", not "worst quality".
- **Hangup causes:** the full numeric-code reference (4000, 1010, 2030, 3130, etc.) and the textual SIP causes live in the canonical [Hangup Causes](/concepts/hangup-causes) page. The CDR overview at `/cdr.mdx` also carries the textual `hangup_cause` table.
