---
name: vobiz-campaign-planner
description: Plan and run an outbound voice campaign on Vobiz end to end - size it against concurrency and balance, choose the caller IDs from real number performance, decide whether to buy new numbers, pick the calling window and retry policy from historical answer rates, estimate the cost, clean the contact list, build the campaign through the Campaign Manager API, monitor it, and close it with a results report. Use this whenever someone wants to call a list of people, run a dialler, blast, drive or outreach, asks "when should I run my campaign", "which numbers should I use", "how many numbers do I need", "how much will it cost", or wants a plan before launching. It uses the vobiz-reports skill for the evidence.
license: MIT
compatibility: Vobiz REST API v1 (X-Auth-ID + X-Auth-Token) for the Campaign Manager; the Vobiz MCP Server or vobiz CLI for reports and numbers. Launching, buying numbers and placing calls spend money and need the user's explicit confirmation.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz campaign planner skill

Use this when the goal is **to reach a list of people by voice** and the person needs to know how: how many numbers, which ones, when to dial, how fast, what it costs, and how to keep the campaign healthy. The planner produces a written plan with evidence behind every choice, then builds and monitors the campaign only after the person says go.

Base URL: `https://api.vobiz.ai`. Campaign Manager paths are `/api/v1/account/{auth_id}/...` (lowercase `account`). Auth headers on every request: `X-Auth-ID` and `X-Auth-Token`.

**Boundaries.** Plans, previews, reports and draft campaigns are free and reversible. Starting or scheduling a campaign, purchasing numbers, and buying concurrency or CPS spend money and contact real people: show the account, the numbers, the contact count and the estimated cost, then wait for an explicit yes before each one. Never upload a list the person has not confirmed they may call.

## Gather the brief

Ask one question at a time, and skip anything already answered. Capture the answers in the plan file.

1. **Outcome.** What should happen when someone answers: an AI agent, an IVR, a human transfer, a reminder message? Which answer URL or agent exists already?
2. **Audience.** How many contacts, which countries and regions, where the list came from and whether consent and opt-outs are handled.
3. **Deadline.** By when must everyone be reached, and are there dates or hours to avoid.
4. **Budget and success.** The spend ceiling, and the metric that defines success (connected calls, conversations over 30 seconds, transfers, confirmations).
5. **Constraints.** Caller IDs the brand must or must not show, languages, do-not-call rules, and whether the account is prepaid or postpaid.

## Build the evidence

Before choosing anything, load `vobiz-reports` and check `vobiz-reports/index.md`. Reuse reports under 7 days old; otherwise generate these for the last 30 days, outbound only:

| Report | What the planner takes from it |
|---|---|
| `connectivity` by hour and by weekday | The calling window and the throughput assumption |
| `number-performance` | Which caller IDs connect, which are spam-flagged or idle, how many are usable |
| `destination` | Dead prefixes and countries to scrub, expected connect rate per region |
| `hangup-causes` | Whether the previous campaigns failed on the list, the carrier or the customer's own XML |
| `balance` and `spend` | Available balance, daily burn, runway |
| `capacity` | Concurrency in use against the limit, peak hour |

Also read `GET /api/v1/auth/me` for `pricing_tier` (rate per minute, billing increment, minimum duration), `concurrent_calls_limit`, `cps_limit` and `timezone`, and `GET /api/v1/account/$AUTH_ID/capacity` for `concurrent_free` and `running_campaigns`. A new account with no history uses the defaults in this skill and says so in the plan.

## Size the campaign

Work the arithmetic in the plan so the person can change an input and see the effect.

- **Attempts** = contacts × (1 + expected retry share). Expected retry share comes from the not-connected share in `connectivity` (default 0.6 when there is no history), capped by `retry_attempts`.
- **Handle time** = average `ring_time` + average `billsec` for connected calls, or `ring_time` alone for the rest; take both from the CDR export. Default 45 seconds when unknown.
- **Throughput per hour** = `max_concurrent` × 3600 / handle time. `max_concurrent` can never exceed `concurrent_free`, and should leave headroom for inbound traffic and other campaigns.
- **Hours needed** = attempts / throughput per hour. Compare with the calling window × days until the deadline. If it does not fit, the options are more concurrency (price it with `GET /api/v1/accounts/$AUTH_ID/channel-pricing-preview?resource_type=concurrent_calls&quantity=N`), a longer window, or a smaller list; present all three with cost.
- **Cost estimate** per attempt = connect rate × (expected billsec / 60, rounded up to the billing increment) × rate per minute, plus the non-connected fee when the tier charges one, plus streaming, recording or transcription rates if the flow uses them. Multiply by attempts, add number rentals and any purchased capacity, and compare with `available_balance`. If the estimate exceeds half the balance, recommend a top-up before launch so `balance_insufficient` cannot pause the campaign midway.

## Choose the caller IDs

The strategy follows from the audience and the number inventory:

| Strategy | Use when | Settings |
|---|---|---|
| `fixed` | One trusted, recognised number; small or medium lists | `fixed_caller_id` |
| `pool` | Large lists, or a single number would carry too many calls per day | `pool_rotation_strategy` (`least_used` spreads reputation most evenly), `pool_max_calls_per_day`, `pool_max_calls_per_number`, `pool_cooldown_seconds` |
| `per_contact` | The CSV already says which number should call whom (local presence, per-branch lines) | `from` column in the upload |

Rank the owned numbers from `number-performance`. Keep numbers whose 30-day answer rate is at or above the account median and whose `is_spam` is false. Exclude spam-flagged numbers from any pool and recommend resting them. Prefer numbers whose `region` matches the destinations. Give numbers with zero recent calls a lower daily cap for the first days of the campaign so their reputation builds gradually.

**Pool size** = planned calls per day / per-number daily cap, rounded up. The default cap in the plan is 300 calls per number per day; the person can raise it, and the campaign enforces whatever `pool_max_calls_per_day` is set to.

**Buy numbers when** the usable pool is smaller than the pool size, there is no number in the destination region, or every usable number is already committed to another campaign. Search `GET /api/v1/Account/$AUTH_ID/inventory/numbers?country=IN&per_page=100` (add `search` for a prefix), list the candidates with `setup_fee`, `monthly_fee`, `currency`, `aadhaar_verification_required` and the lead time that implies, total the recurring cost, and ask for confirmation before `POST .../numbers/purchase-from-inventory`. A purchase starts a monthly charge; say so in the plan. Hand the purchase itself to `vobiz-phone-numbers`.

## Choose the window, pace and retries

- **Window.** Take the hours with the highest connect rate from `connectivity` by hour in the contacts' timezone, and set `window_enabled`, `window_start_time`, `window_end_time` and `timezone` on the campaign. Confirm the permitted calling hours and do-not-call scrubbing for each destination country with the person's compliance owner before fixing the window; the planner does not know local law.
- **Pace.** `max_concurrent` from the sizing step. Start a new or unproven list at half the planned concurrency for the first hour, read the connect rate, then raise it.
- **Retries.** Set `retry_attempts` 1-3, `retry_delays` in minutes, and `retry_window_days`. Use longer delays when `no_answer` dominates (people are away) and shorter ones when `USER_BUSY` dominates. Contacts that `failed` on `INVALID_NUMBER`, `UNALLOCATED_NUMBER` or barred-destination codes do not improve on retry; keep them out of the retry budget by scrubbing them from the list first.
- **Schedule.** `scheduled_at` in ISO 8601 for a future launch, otherwise start on confirmation.

## Clean the list

Before upload: normalise every `to` to E.164 with a leading `+`, drop duplicates, drop opt-outs and do-not-call matches, drop numbers that errored on invalid or unallocated codes in the last 90 days (from `hangup-causes`), and cap the list at the sized attempt budget. Report the before and after counts. The upload is `multipart/form-data` with a `file` field; only `to` is required, `from` is used by `per_contact`, `answer_url` and `hangup_url` override the agent per row, and any other column becomes a custom SIP header on that call. The response gives `valid_contacts`, `invalid_rows` and `total_rows`; a valid upload moves the campaign from `draft` to `ready`.

## Build it (after the go-ahead)

1. **Agent**: reuse one from `GET .../campaign/agents` or create one with `name`, `answer_url`, `answer_method`, `hangup_url`, `static_headers`. The answer URL must return VobizXML; test it with a single `vobiz-voice-calls` call before the campaign.
2. **Campaign**: `POST .../campaigns` with the plan's values.
   ```json
   { "name": "Renewal reminders Oct", "agent_id": "<agent id>", "max_concurrent": 10,
     "timezone": "Asia/Kolkata", "caller_id_strategy": "pool",
     "window_enabled": true, "window_start_time": "10:00", "window_end_time": "19:00",
     "retry_attempts": 2, "retry_delays": [60, 240], "retry_window_days": 3,
     "pool_rotation_strategy": "least_used", "pool_max_calls_per_day": 300, "pool_cooldown_seconds": 30,
     "notify_url": "https://example.com/vobiz/campaign-events" }
   ```
3. **Pool**: `POST .../campaigns/{id}/pool` with `numbers: [{ from_number, label }]`. The pool is editable only in `draft` or `ready`.
4. **Upload** the cleaned CSV and check `invalid_rows`.
5. **Preview**: restate contacts, concurrency, window, numbers, the cost estimate and the balance. Then, on an explicit yes, `POST .../campaigns/{id}/start` or set `scheduled_at`.

## Monitor and intervene

Check `GET .../campaigns/{id}` and `.../capacity` every 15 minutes for the first hour, then hourly, and run the `campaign` report daily. Act on what you see:

| Signal | Action |
|---|---|
| `health: stalled` or `pause_reason: stalled` | Check the agent's answer URL and the account's concurrency; resume once calls complete |
| `pause_reason: caller_pool_exhausted` or `caller_pool_daily_limit_reached` | Add numbers to the pool before the next day or raise the daily cap, with the reputation trade-off stated |
| `pause_reason: balance_insufficient` | Top-up needed; the campaign resumes after it |
| `pause_reason: consecutive_failures` | Stop. Read `hangup-causes` for the campaign; a `7xxx` or `8xxx` run means the XML or URL is broken, a `3130` run means a number is spam-blocked and must leave the pool |
| Connect rate below half the plan after 200 attempts | Pause, compare caller IDs and hours in the `campaign` report, adjust the pool or window, resume |
| Short-call share above 50% | The opening line, not the dialling, is the problem; stop and fix the flow |
| `window_closed` | Expected; it resumes when the window opens |

Webhooks on `notify_url` carry `campaign.*` and `contact.*` events signed with `X-Vobiz-Signature`; use them instead of polling when the person has an endpoint.

## Close it

Download `GET .../campaigns/{id}/results` (CSV, works mid-run for a partial view), pull the CDR export with `campaign_id`, and run the `campaign` report. Record in the plan file what the actual connect rate, cost per connected call and best hours were against the estimate, which numbers performed, and what to change next time. Archive with `POST .../campaigns/{id}/archive`; archived campaigns keep contacts, CDRs and stats.

## Plan file convention

Plans live in `vobiz-campaigns/` under the working directory or `$VOBIZ_CAMPAIGNS_DIR`. One file per campaign, updated through its life: `YYYY-MM-DD_<slug>_plan.md`.

```markdown
---
campaign: renewal-reminders-oct
account: MA_XXXXXXXX
status: draft | approved | running | paused | completed
campaign_id: null
created_at: 2026-10-22T10:00:00+05:30
updated_at: 2026-10-22T10:00:00+05:30
contacts: 12000
deadline: 2026-10-31
timezone: Asia/Kolkata
evidence: [vobiz-reports/2026-10-22_MA_XXXXXXXX_connectivity_2026-09-22_to_2026-10-21_by-hour.md, ...]
---

# Renewal reminders, October 2026

## Brief
## Evidence used
## Sizing
## Caller IDs
## Window, pace and retries
## List hygiene
## Cost estimate and balance
## Build steps and confirmations
## Monitoring log
## Results and learnings
```

Each confirmation the person gives, and each pause reason seen, gets a dated line under Build steps or Monitoring log.

## Pitfalls

- **Capacity is shared.** Inbound calls, other campaigns and API calls use the same concurrency. Plan `max_concurrent` from `concurrent_free`, not from the account limit.
- **`cps` on the campaign is legacy** and does not gate capacity; `max_concurrent` does.
- **The window and daily caps follow the campaign `timezone`,** while CDR timestamps are UTC. Convert before comparing a report with the window.
- **A spam-flagged number drags the whole pool.** Remove it the moment `number-performance` or a `3130` run shows it; do not wait for the daily report.
- **Retries multiply cost.** Every retry is a billed attempt when the tier charges non-connected calls. The sizing must include them.
- **Sub-account campaigns bill the parent wallet.** Say which wallet pays in the plan.
- **Path casing.** Campaign Manager uses lowercase `/account/`; CDRs, numbers and transactions use `/Account/`. Copy the paths from the docs page.
- **Nothing launches, buys or dials without a fresh, explicit yes** for that specific action; approval of the plan is not approval to start.

## When to search docs

- Campaign Manager overview and endpoint index: https://www.vobiz.ai/docs/campaign-manager/overview
- Create campaign fields: https://www.vobiz.ai/docs/campaign-manager/campaigns/create-campaign
- Capacity, health and pause reasons: https://www.vobiz.ai/docs/campaign-manager/capacity/account-capacity
- Campaign webhooks: https://www.vobiz.ai/docs/campaign-manager/webhooks
- Number inventory and purchase: https://www.vobiz.ai/docs/account-phone-number/list-inventory-numbers
- Number health: https://www.vobiz.ai/docs/account-phone-number/get-number-health
- Hangup codes: https://www.vobiz.ai/docs/concepts/hangup-causes
- Reports this skill relies on: load `vobiz-reports`
