---
name: vobiz-reports
description: Generate customer reports from Vobiz account data and save each one as a dated Markdown file - connected, not-connected and error calls, answer and connect rates by hour, number, destination or campaign, hangup-cause breakdowns, per-number performance and spam flags, call quality, spend by day and by charge type, balance and runway, campaign outcomes, SMS and WhatsApp delivery, sub-account usage. Use this whenever someone asks "how many calls", "what is my answer rate", "which number performs best or worst", "what is happening with my calls", "what did I spend", or wants any report, dashboard, summary or export from Vobiz, even when they do not say "report".
license: MIT
compatibility: Vobiz REST API v1 (X-Auth-ID + X-Auth-Token), the Vobiz MCP Server, or the vobiz CLI. Needs a shell with curl, jq and python3 for local aggregation.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz reports skill

Use this for **any question that is answered by counting, rating or costing what happened on a Vobiz account**: calls for a day, answer rate, which caller ID connects best, why calls fail, what was spent, how a campaign went, whether a number is flagged as spam. The skill turns the question into a defined report, pulls the data through whichever Vobiz route is available, computes the numbers the same way every time, and saves the result as a dated Markdown file so the next question, or the `vobiz-campaign-planner` skill, can build on it.

Base URL: `https://api.vobiz.ai`. Auth headers on every request: `X-Auth-ID` and `X-Auth-Token`. Every report in this skill is read-only; nothing here spends money or places a call.

## Pick the data route

Use the first route that is available, in this order. They return the same fields.

| Route | How to tell it is available | Best for |
|---|---|---|
| Vobiz MCP Server | tools such as `list_call_records`, `get_number_health`, `list_transactions`, `get_account_balance` are present | Headline numbers and reports up to a few thousand rows, inside Claude, ChatGPT, Cursor and other MCP clients |
| `vobiz` CLI | `vobiz --version` works and `vobiz account show` returns the account | Bulk pulls: `vobiz cdr export --all`, `vobiz account transactions list --all`, CSV on disk |
| REST API with curl | `$AUTH_ID` and `$AUTH_TOKEN` are set | Everything else, and the only route for the Campaign Manager and partner endpoints |

Credentials come from the MCP connection, the CLI profile, or environment variables. Never print, persist or commit a token, and never write one into a report file. The account ID (`MA_XXXXXXXX` or `SA_XXXXXX`) may appear in reports; the token may not.

## Define the outcomes once

Every report classifies calls the same way, so numbers agree across reports and with the Console. A call is one CDR row.

| Outcome | Rule | Why |
|---|---|---|
| **Connected** | `billsec > 0` | Matches `summary.answeredCalls` in the CDR API and the Campaign Manager's `answered` status |
| **Not connected** | `billsec == 0` and `hangup_cause` in `NO_ANSWER`, `USER_BUSY`, `ORIGINATOR_CANCEL`, `CALL_REJECTED`, `REJECTED`, `NORMAL_CLEARING`, or `hangup_cause_code` is `9100` (machine detected) | The recipient or the caller ended it; the route worked |
| **Error** | `billsec == 0` and anything else: `INVALID_NUMBER`, `UNALLOCATED_NUMBER`, `SERVICE_UNAVAILABLE`, `SERVER_ERROR`, `MEDIA_TIMEOUT`, `PROTOCOL_ERROR`, `NETWORK_OUT_OF_ORDER`, `DESTINATION_OUT_OF_ORDER`, `NORMAL_TEMPORARY_FAILURE`, `SWITCH_CONGESTION`, `UNKNOWN` | Something between the account and the destination failed; these need a fix, not a retry |

Inside **Error**, group by `hangup_cause_code` range so the reader knows who to call: `1010` out of credits and `1020` dial limit (account), `2000-2070` destination barred or invalid (list hygiene), `3070-3140` carrier including `3130` spam block (number reputation), `5000-5030` platform, `6000-6020` timeouts, `7xxx` answer/hangup URL failures and `8xxx` XML errors (the customer's own webhook or XML). The code table is at https://www.vobiz.ai/docs/concepts/hangup-causes.

Derived metrics, always computed from these buckets:

- **Answer rate** = connected / total calls (the API's `answerRate` uses the same definition).
- **Connect rate** = connected / (total - error). Use this to judge a list or a caller ID, because errors are not the recipient's doing.
- **Error rate** = error / total.
- **Short-call share** = connected calls with `billsec < 10` / connected. A high share with a good answer rate usually means the audience hangs up on the opening line.
- **Cost per connected call** = sum of `total_cost` / connected. **Cost per minute** = sum of `total_cost` / (sum of `billsec` / 60).
- **Quality**: average `mos` over connected calls only (`mos` is `0` when no media flowed), plus the share of connected calls with `mos < 3.5`.

## Report catalogue

| Report | Answers | Primary source | Where the maths happens |
|---|---|---|---|
| `connectivity` | Calls, connected, not connected, error, answer and connect rate for a period; by day, by hour, by direction | CDR list `summary` for totals; CDR export for the cuts | Server for totals, local for buckets |
| `hangup-causes` | Why calls fail; top causes and codes, who is responsible, trend against the previous period | CDR export | Local |
| `number-performance` | Best and worst caller IDs: calls, answer rate, failed, spam flag, usage status, idle numbers still paying rent, next billing date | Number list + number health per number; CDR export grouped by `caller_id_number` for the long tail | Server per number, local for ranking |
| `destination` | Answer rate and error rate by destination country or prefix, which lists are dead | CDR export grouped by the leading digits of `destination_number` | Local |
| `quality` | MOS, jitter, packet loss by day, number, region and codec; worst calls | CDR export, connected calls only | Local |
| `spend` | Spend by day and by `reference_type` (calls, DID rental, recording, streaming, transcription), credits vs debits, pending transactions | Transactions `summary` and `by_reference_type`; rows for the daily cut | Server for totals, local for days |
| `balance` | Available balance, reserved funds, credit limit, average daily burn, days of runway, low-balance threshold | Balance endpoint + the last 7 and 30 days of `spend` | Local |
| `campaign` | Per campaign: contacts by status, attempts per contact, connect rate, retries, cost from CDRs, pause history, pool number eligibility; comparison across campaigns | Campaign Manager campaign, contacts, calls and pool endpoints; CDR filtered by `campaign_id` | Local |
| `messaging` | SMS, MMS, RCS delivered and failed by day and channel, broadcast funnels, WhatsApp campaign sent → delivered → read, opt-outs | SMS messages list, broadcasts (`stats`), WhatsApp campaigns | Server for broadcasts, local for messages |
| `sub-accounts` | Calls, minutes and cost per sub-account or per partner customer | CDR with each sub-account's own `SA_` credentials, or the partner CDR and transactions endpoints | Local |
| `capacity` | Concurrency and CPS in use against the purchased limits, peak hour | Concurrency endpoint, `/auth/me`, CDR by hour | Local |

When the question does not match a row, build the closest report and add the extra cut; do not invent a new report type.

## Workflow

1. **Scope the question.** Fix the account, the period, the timezone and the cut. "Yesterday" means the account's local day. If the person gave no period, use the last 7 full days and say so.
2. **Get the account context first.** `GET /api/v1/auth/me` returns `timezone`, `pricing_tier` (rate per minute, billing increment, minimum duration) and the purchased limits. Reports quote rates from here, never from memory.
3. **Pull totals from the server before pulling rows.** CDR list and transactions list both return a `summary` over the whole filtered set regardless of paging, so `per_page=1` gives the headline numbers in one call. Only export or page rows when the cut needs them (by hour, by number, by cause).
4. **Compute locally with a script, not by eye.** Save the raw CSV, run the aggregation (see the recipe), and keep the script output. If the pull hit a cap or a date range was trimmed, record it.
5. **Write the report file** using the convention below, then answer in chat with the headline numbers, the top findings and the file path. The chat answer must stand on its own; the file holds the tables.
6. **Append one line to the index** so later questions and the campaign planner can find the report without regenerating it.

## Dates and timezones

- CDR `start_date` and `end_date` are `YYYY-MM-DD` and must be sent together. CDR timestamps are ISO 8601 in UTC (`...Z`). To report an IST day, request the UTC day before and the UTC day itself, then keep rows whose `start_time` converted to `Asia/Kolkata` falls inside the local day.
- Transactions `from_date` and `to_date` accept a full offset timestamp. Send `2026-10-21T00:00:00+05:30` style values and URL-encode the `+`; a bare date resolves in UTC.
- SMS `startDate` and `endDate` are RFC 3339 and the end is exclusive.
- Campaign windows, daily limits and the Console's campaign call logs use the campaign's own `timezone`.
- State the timezone in every report header and in the chat answer.

## Pull sizes and caps

| Data | Per page | Notes |
|---|---|---|
| CDR list and search | 100 | Follow `pagination.has_next`. The MCP tool caps a range at 92 days |
| CDR export | all rows | `text/csv`; always send both dates. Prefer this over paging for more than about 1,000 rows |
| Transactions | 1,000 | A value above 1,000 silently falls back to 50 |
| Number health | one number per call | `days` 1-90, `granularity` daily or hourly; loop over the number list |
| Campaign calls | 200 | `page` and `per_page` |
| SMS messages | 100 | `limit` and `offset`; no summary, compute delivery rate locally |
| Broadcasts | 500 | `stats` is recounted on every read |

Rate limits are per parent account and not published. Walk pages sequentially, back off on `429`, and honour `Retry-After` when present.

## Report file convention

Reports live in `vobiz-reports/` under the current working directory, or in `$VOBIZ_REPORTS_DIR` when set. Raw pulls go in `vobiz-reports/data/`. Never put a token in either.

File name: `YYYY-MM-DD_<account>_<report>_<period>.md`, where the first date is the generation date and the period is `YYYY-MM-DD` for a single day or `YYYY-MM-DD_to_YYYY-MM-DD` for a range. Add a suffix for a cut, for example `_by-number` or `_campaign-<id>`.

```markdown
---
report: connectivity
account: MA_XXXXXXXX
period_start: 2026-10-21T00:00:00+05:30
period_end: 2026-10-21T23:59:59+05:30
timezone: Asia/Kolkata
generated_at: 2026-10-22T09:14:00+05:30
source: api cdr/export
filters: { call_direction: outbound }
rows: 4821
complete: true
currency: INR
---

# Outbound connectivity, 21 Oct 2026

## Summary
Four to six headline numbers, one per line: total, connected, answer rate, connect rate, error rate, cost.

## Findings
Ranked, each with the number that supports it and the period it is compared against.

## Tables
By hour, by caller ID, by cause. Rates as percentages with one decimal; money with the currency code.

## Recommendations
One action per finding, in the order the reader should take them. Mark anything that spends money or changes configuration as needing their confirmation.

## Data notes
What was pulled, which filters, caps hit, how outcomes were classified, what was excluded.

## Files
Relative paths to the CSV pulls this report was computed from.
```

`complete: false` whenever a cap trimmed the data or a sub-pull failed; say which one in Data notes. Index file `vobiz-reports/index.md`, newest first, one line per report:

```
- 2026-10-22 | connectivity | MA_XXXXXXXX | 2026-10-21 | outbound | 2026-10-22_MA_XXXXXXXX_connectivity_2026-10-21.md
```

Before generating, check the index. Reuse a report for the same account, type, period and filters that is less than a day old unless the person asks for a refresh.

## Recipes

Headline numbers for a range in one call (the summary ignores paging):
```bash
curl -sG "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr" \
  --data-urlencode "start_date=2026-10-15" --data-urlencode "end_date=2026-10-21" \
  --data-urlencode "call_direction=outbound" --data-urlencode "per_page=1" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" | jq '.summary'
```

Export the rows for local cuts:
```bash
mkdir -p vobiz-reports/data
curl -sG "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/cdr/export" \
  --data-urlencode "start_date=2026-10-20" --data-urlencode "end_date=2026-10-21" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" \
  -o vobiz-reports/data/2026-10-22_${AUTH_ID}_cdr_2026-10-20_to_2026-10-21.csv
```
With the CLI: `vobiz cdr export --from 2026-10-20 --to 2026-10-21 --direction outbound --out <same path>`.

Classify and bucket locally (python3, standard library only; adapt the keys to the cut you need):
```python
import csv, collections, datetime, zoneinfo
TZ = zoneinfo.ZoneInfo("Asia/Kolkata")
NOT_CONNECTED = {"NO_ANSWER","USER_BUSY","ORIGINATOR_CANCEL","CALL_REJECTED","REJECTED","NORMAL_CLEARING"}
def outcome(r):
    if int(r["billsec"] or 0) > 0: return "connected"
    if r["hangup_cause"] in NOT_CONNECTED or r.get("hangup_cause_code") == "9100": return "not_connected"
    return "error"
by = collections.defaultdict(lambda: collections.Counter())
cost = collections.defaultdict(float)
for r in csv.DictReader(open("cdr.csv")):
    t = datetime.datetime.fromisoformat(r["start_time"].replace("Z","+00:00")).astimezone(TZ)
    if t.date() != datetime.date(2026,10,21): continue          # keep the local day only
    key = t.strftime("%H:00")                                   # or r["caller_id_number"], r["hangup_cause"], r["destination_number"][:5]
    by[key][outcome(r)] += 1; by[key]["total"] += 1
    cost[key] += float(r["total_cost"] or 0)
for k in sorted(by):
    c = by[k]; nonerr = c["total"] - c["error"]
    print(k, c["total"], c["connected"], f'{100*c["connected"]/c["total"]:.1f}%', f'{100*c["connected"]/nonerr:.1f}%' if nonerr else "-", f'{cost[k]:.2f}')
```

Per-number performance (loop the owned numbers through the health endpoint):
```bash
curl -sG "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/numbers" --data-urlencode "per_page=100" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" | jq -r '.items[].e164' | while read n; do
  curl -sG "https://api.vobiz.ai/api/v1/account/$AUTH_ID/numbers/$(printf %s "$n" | sed 's/+/%2B/')/health" \
    --data-urlencode "days=30" -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" \
    | jq -c '{e164, is_spam, usage_status, calls: .summary.total_calls, answered: .summary.answered_calls, answer_rate: .summary.answer_rate, minutes: .summary.total_minutes}'
done
```
Numbers with zero calls in the window are **idle**: list them with `monthly_fee`, `currency` and `next_billing_date` from the number list so the reader can decide whether to keep paying.

Spend by charge type and the daily cut:
```bash
curl -sG "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/transactions" \
  --data-urlencode "from_date=2026-10-01T00:00:00+05:30" --data-urlencode "to_date=2026-10-21T23:59:59+05:30" \
  --data-urlencode "per_page=1" -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" | jq '.summary'
```
Then pull rows with `per_page=1000` and `type=debit` for the by-day table. `reference` values like `cdr:<uuid>` join a charge to its CDR.

Balance and runway: `GET /api/v1/Account/$AUTH_ID/balance/INR` gives `available_balance`, `reserved_funds`, `credit_limit`, `is_postpaid` and `low_balance_threshold`. Runway in days = available balance / (last 7 days of debits / 7). Flag runway under 7 days at the top of the Summary.

Campaign report: `GET /api/v1/account/$AUTH_ID/campaigns/<id>` for status, health and counters, `.../contacts?status=<s>` with `limit=1` per status for the funnel (`pending`, `dialling`, `answered`, `no_answer`, `busy`, `failed`, `retry_pending`, `expired`), `.../calls` for attempts, `.../pool` for number eligibility, and the CDR export with `campaign_id=<id>` for cost and quality. The results CSV at `.../results` has `attempt_number` and `answered_at` per contact.

## Pitfalls

- **Two answer rates can both be right.** The API's `answerRate` is over all calls; the connect rate excludes errors. Show both and label them, or a 60% answer rate with a 40% error rate looks like a list problem when it is a routing problem.
- **A day in IST is two UTC dates.** Pulling `start_date=end_date=2026-10-21` returns the UTC day and undercounts the evening. Pull both dates and filter locally.
- **`summary` ignores paging but respects filters.** It is the fastest total you can get; it is not an account-wide total unless you sent no filters.
- **Transactions for an `SA_` account return the parent's ledger.** Per-sub-account spend comes from CDR `total_cost` pulled with that sub-account's credentials, not from transactions.
- **`mos: 0` means no media,** not bad quality. Average `mos` over connected calls only.
- **`avgCallDuration` is a string** like `"28s"`. Use `total_billable_seconds / answeredCalls` instead.
- **Campaign outcome counters are not fully documented.** Build the campaign funnel from the contacts endpoint totals per status, which are, and treat the counters on the campaign object as a cross-check.
- **Number health is one call per number.** For hundreds of numbers, rank from the CDR export grouped by `caller_id_number` first and call health only for the top and bottom ten plus any number the reader names.
- **A zeroed `summary` next to a non-empty `data` array** is a degraded response. Retry once; if it persists, compute totals locally and set `complete: false`.
- **Recordings `add_time` carries a `+05:30` offset** while CDRs are UTC. Convert before joining on time.
- **Never let a report recommend a write without a confirmation gate.** Releasing an idle number, buying a number, or topping up are the reader's decisions; the report states the cost and the effect and stops there.

## When to search docs

- CDR fields, filters and the export: https://www.vobiz.ai/docs/cdr
- Hangup codes and causes: https://www.vobiz.ai/docs/concepts/hangup-causes
- Number health: https://www.vobiz.ai/docs/account-phone-number/get-number-health
- Transactions and date handling: https://www.vobiz.ai/docs/account/transactions
- Campaign Manager contacts, calls, results and pause reasons: https://www.vobiz.ai/docs/campaign-manager/overview
- Partner per-customer CDRs and transactions: https://www.vobiz.ai/docs/partner/api/cdrs
- Rate limits: https://www.vobiz.ai/docs/rate-limits
- Vobiz MCP Server tools: https://www.vobiz.ai/docs/resources/vobiz-mcp
