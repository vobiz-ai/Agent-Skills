---
name: vobiz-partner-api
description: Vobiz Partner / reseller API - create customer sub-accounts, transfer balance, run KYC sessions (PAN+Aadhaar OTP, no documents), and query per-customer CDRs/transactions/numbers.
license: MIT
compatibility: Vobiz REST API v1, partner-tier credentials.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz Partner API skill

Use this for **reseller workflows** where one partner master account owns multiple **partner child accounts** (the docs also call them customer accounts or customer sub-accounts). Each child is a full Vobiz main account (`MA_…`) with its own wallet, KYC, numbers, CPS, and concurrency. Auth uses the **same** `X-Auth-ID` / `X-Auth-Token` headers as the main API - there is **no separate Bearer JWT for server-to-server calls**. Just use your partner-tier credentials. (`POST /login` exchanges email + password for a JWT, but the docs reserve it for interactive sessions; it is doc-only and not in `openapi.yaml`.)

Not sure whether the user needs partner child accounts or sub-accounts? See `concepts/sub-accounts-vs-partner-accounts` and the `vobiz-sub-accounts` skill.

## Base path

`https://api.vobiz.ai/api/v1/partner/*`

(Routes do NOT live under `/partner/v1/` - that path returns 404.) Send `Accept: application/json`, and `Content-Type: application/json` on bodies.

## Partner profile + dashboard

- `GET /api/v1/partner/me` - your partner profile: `id` (UUID), `auth_id` (`PA_…`), `balance` (string, 5 decimals), `currency`, `max_accounts`, `account_count`, `is_verified`, and the capability flags `can_create_accounts`, `can_transfer_balance`, `can_view_cdrs`, `can_create_pricing_tiers`.
- `GET /api/v1/partner/dashboard` - aggregated metrics over a `period`: `accounts {total, active, customers[]}`, `total_balance` (string), `currency`, `calls`, `traffic {inbound, outbound}`, `by_product {sip_trunking, voice_api}`, `time_series[]`. `partner.balance` is `null` here.
- `GET /api/v1/partner/analytics?from_date=&to_date=` - same shape over a custom window (defaults to the last 30 days). Doc-only (`partner/api/analytics`), not in the spec.

Auth model in one line: **partner creds (`PA_…`) drive `/api/v1/partner/*`; the customer's own creds (`MA_…`) drive `/api/v1/Account/{customer_auth_id}/*` voice ops.** Your partner token cannot drive a customer's voice resources. Sending the wrong pair against the wrong base path returns `401`/`403`.

## Customer accounts

- `GET /api/v1/partner/accounts` - list customers. Pagination params are `page` + `per_page` (default 20, max 100); the **response** echoes `per_page` back as `size`. `search` substring-matches name or email. Rows carry `cash_credits`, `kyc_status`, `cps_limit`, `concurrent_calls_limit`, `pricing_tier`.
- `POST /api/v1/partner/accounts` - create customer. Required: `name, email, phone, password, country`. Optional (guaranteed): `company`; other optional fields on `partner/api/customers` are best-effort. Returns `201` with the customer's own `auth_id` + `auth_token` in clear - **store the `auth_token` encrypted server-side immediately**; later reads show it redacted, and `GET /accounts/{customer_auth_id}` is the documented way to retrieve it again. `email` must be platform-unique (`422` / `409`); `password` needs 8+ chars with a number and special char (`422`); `403` when `can_create_accounts` is `false` or you hit `max_accounts`. New customers start `status: active`, `balance: 0.00`, `kyc_status: pending`.
- `GET /api/v1/partner/accounts/{customer_auth_id}` - full customer profile.
- `GET /api/v1/partner/accounts/{customer_auth_id}/balance` - real-time wallet: `{auth_id, name, cash_credits, currency, updated_at}`.

(`GET /accounts/{id}` and `/balance` are doc-only - see `partner/api/customers`.)

## Balance transfer

- `POST /api/v1/partner/accounts/{customer_auth_id}/transfer-balance` - atomic debit/credit, both legs in each ledger. Body: `{amount, currency, description?}`. `currency` must equal your partner currency (no FX conversion). Response `200`: `{transaction_id, from_account, to_account, amount, currency, status: "completed", partner_balance_after, customer_balance_after, timestamp}` - there is no `pending` state. **Permanent - cannot be reversed via API; no idempotency key** (poll customer transactions before retrying a timed-out request). `400` on insufficient partner balance (`error: insufficient_balance` with `current_balance` / `requested_amount` / `currency`), non-positive `amount`, or currency mismatch; `401`/`403` when `can_transfer_balance` is `false`; `404` if the customer isn't yours.

## CPS and concurrency allocation (capacity)

Partner child accounts own their CPS and concurrent-call limits. You move slots between your partner pool and a customer with these routes (documented on `partner/api/capacity`; not yet in `openapi.yaml`). Partner creds, customer `auth_id` in the URL.

| Op | Path | Body / query |
|---|---|---|
| Transfer to customer | `POST /api/v1/partner/accounts/{customer_auth_id}/concurrency/transfer` | `{resource?: "concurrent_calls" (default) \| "cps", slots: 1-10000, description?: ≤500 chars}` |
| Reclaim from customer | `POST /api/v1/partner/accounts/{customer_auth_id}/concurrency/reclaim` | same body |
| View allocation | `GET /api/v1/partner/accounts/{customer_auth_id}/concurrency` | → `partner_concurrent_calls_limit`, `customer_concurrent_calls_limit`, `net_concurrent_calls_allocated`, `partner_cps_limit`, `customer_cps_limit`, `net_cps_allocated` |
| Transfer history | `GET /api/v1/partner/concurrency/transfers?limit=50&offset=0` | `limit` 1-200 (default 50), `offset` default 0, newest first |

Transfer/reclaim response: `{success, slots, resource, from_account, to_account, transfer_reference, partner_remaining, customer_effective, message}`.

Pitfalls:
- The path says `concurrency` but it serves **both** resources - always set `resource` explicitly. CPS and concurrent calls move in **separate** requests.
- A transfer is **additive**: it subtracts `slots` from your limit and adds them to the customer's. It does not *set* the customer's limit to `slots`. Read `GET .../concurrency` first and confirm `resource` + `slots` with the user before firing.
- Reclaim cannot exceed the net capacity you previously allocated for that resource, nor the customer's current limit.
- Requirements: active partner, limit-transfer access enabled for both resources (via support@vobiz.ai), the customer belongs to you and is not itself a partner, and you have enough pool capacity. Both accounts update together; allow up to five minutes if the refreshed limits lag.
- Keep `transfer_reference` and reconcile against `GET /concurrency/transfers`.

## KYC sessions (eKYC, no document uploads)

eKYC pulls data from govt APIs - no uploads. Individual (PAN 4th char = `P`): PAN + DOB → Aadhaar via DigiLocker (Govt OTP). Company (PAN 4th char = `C`): PAN + entity name → GSTIN. The PAN determines the path; the partner does not pick it.

- `POST /api/v1/partner/kyc-sessions` - body: `{account_auth_id, flow_type?, customer_email?, redirect_url?, webhook_url?, expires_in_days?, reminder_schedule?, metadata?}`. Returns `201`.
  - `flow_type: "email"` (default) requires `customer_email`; session starts at **`email_sent`**; response has `email_dispatched_to` (masked).
  - `flow_type: "redirect"` requires `redirect_url`; response carries `widget_url` (token `kst_<64hex>`), session starts at **`link_ready`**. After verify, browser returns to `redirect_url?session_id=&status=&auth_id=` - treat those params as a hint only, confirm server-side.
  - `expires_in_days` default **7**. `reminder_schedule` items: `{trigger: "days_before_expiry", value: <int>}` (email flow). `metadata` is echoed back on GET + webhooks.
  - `422` validation (missing `customer_email`/`redirect_url`, malformed `reminder_schedule`, bad JSON - `details[]` with `loc`/`msg`); `404` if `account_auth_id` isn't yours.
- `GET /api/v1/partner/kyc-sessions` - list (filter `status`/`account_auth_id`; paginated with `page`/`size`, default 20).
- `GET /api/v1/partner/kyc-sessions/{session_id}` - one session: `id`, `status`, `kyc_type`, `expires_at`, `first_opened_at`, `completed_at`, `metadata`, `verified_data`. Note: GET returns the id under `id`, POST returned it under `session_id` (same value).
- `POST /api/v1/partner/kyc-sessions/{session_id}/resend` - resend email. Email-flow only (`400` on redirect flow). `409` if the session is terminal. The 30-min cooldown starts at session **creation**, so an immediate resend returns `429` with remaining seconds in the message.
- `DELETE /api/v1/partner/kyc-sessions/{session_id}` - revoke (optional `{reason}` body). `200 {message, session_id}`; `409` if already terminal.

Webhook events on `webhook_url`: `kyc.completed`, `kyc.failed`, `kyc.session_revoked`. Statuses: `email_sent | link_ready → opened → in_progress → kyc_completed | revoked`. `kyc_type` is `null` until PAN is submitted, then `individual` or `company`. On `kyc_completed` the customer's `kyc_status` flips to `completed`.

## Per-customer queries

- `GET /api/v1/partner/accounts/{customer_auth_id}/transactions` - pagination `page`/`per_page` (max 200; over-limit → `422`); date filters `from_date`/`to_date`; `transaction_type` ∈ `recharge|debit|refund|transfer`. Each row's `type` field is the ledger direction `credit`/`debit` (different vocabulary from the filter). Includes a `summary` block and `total_pages`. Partner transfers land on the customer ledger as `credit` rows with reference `ptc:<partner_id>:<customer_auth_id>:<unix_ts>`.
- `GET /api/v1/partner/accounts/{customer_auth_id}/cdrs` - filters `start_date`, `end_date` (not `from_date`/`to_date`), `call_direction` (`inbound|outbound`), `status` (`answered|failed|busy|no_answer` - underscored), `min_duration`, `hangup_cause`; `page`/`per_page` (max 100). Response wraps rows in `data[]` + `pagination` + `summary`. Bill on `billsec` (billable seconds) or `summary.total_cost`.
- `GET /api/v1/partner/accounts/{customer_auth_id}/numbers` - pagination `page`/`per_page`; `search` (E.164 substring). Response `{items[], page, per_page, total, account_auth_id}`. Read-only; assign/release via Console.
- Global variants `GET /transactions`, `GET /cdrs`, `GET /accounts/{customer_auth_id}/cdrs/{call_uuid}`, `GET /numbers` are doc-only (same filters) - see `partner/api/transactions`, `partner/api/cdrs`, `partner/api/numbers`.

## Customer-side endpoints

Same auth scheme, but uses the **customer's** `auth_id`/`auth_token` against `/api/v1/Account/{customer_auth_id}/*` (inventory, purchase/release numbers, trunks, calls, CDRs). The customer can call these themselves, or the partner can call them on the customer's behalf using the customer's stored `auth_token`.

## Pitfalls

- **Base path is `/api/v1/partner`, not `/partner/v1`.** The latter 404s. Full base: `https://api.vobiz.ai/api/v1/partner`.
- **`balance` is a string with 5 decimals** (`"23906.83000"`). Cast to float before comparing. On `GET /dashboard` the partner `balance` is `null` - use `total_balance` there. Customer wallets are `cash_credits` (also a string).
- **Pagination keys differ by endpoint.** Accounts/transactions/cdrs/numbers use `per_page`; KYC sessions list uses `size`. The accounts list *accepts* `per_page` but *returns* it as `size`.
- **Balance transfer is irreversible and has no idempotency key.** Verify `customer_auth_id`, `amount`, and `currency` (must match partner currency) before firing. If a request times out, poll customer transactions (match `amount` + recent `created_at`) before retrying to avoid a double credit.
- **Capacity transfers are additive, per-resource, and gated by support-enabled access.** Read the current allocation first; confirm `resource` and `slots` before firing.
- **Only one active KYC session per `account_auth_id`.** Creating a new one silently auto-revokes the previous - always operate on the most recent `session_id`, or you'll get `409`s.
- **KYC resend cooldown starts at creation**, so an immediate resend returns `429`. Resend is email-flow only.
- **Redirect-flow return params can be spoofed** - confirm via the `kyc.completed` webhook or a server-side `GET /kyc-sessions/{id}` before unlocking.
- **Never expose tokens client-side.** The partner token and customer `auth_token`s are long-lived money-moving secrets. Keep them server-side; rotate from the Console if leaked.
- **Capability flags gate operations.** `can_create_accounts` / `can_transfer_balance` being `false` returns `403`; `can_view_cdrs` being `false` blocks the CDR endpoints. Check `GET /me`.
- **Two "type" vocabularies on transactions.** Filter with `transaction_type=recharge|debit|refund|transfer`; rows report direction as `type: credit|debit`.

## Complete onboarding recipe

Run in order; each step depends on the previous:

1. **Preflight** - `GET /me`; confirm `can_create_accounts` and `can_transfer_balance` are `true`, `account_count < max_accounts`, and `float(balance) >= amount`.
2. **Create customer** - `POST /accounts` with `{name,email,phone,password,country,company?}`. Capture `auth_id` + `auth_token` (store the token encrypted).
3. **Fund wallet** - `POST /accounts/{auth_id}/transfer-balance` with `{amount, currency}` (currency = your partner currency). Confirm with the user first - irreversible. Assert response `partner_balance_after >= 0` and `status == "completed"`.
4. **Start KYC** - `POST /kyc-sessions` with `{account_auth_id: auth_id, flow_type, webhook_url, expires_in_days:14, ...}` (email flow needs `customer_email`; redirect flow needs `redirect_url`). Store `session_id`.
5. **Wait for completion** - receive `kyc.completed` on `webhook_url` (verify server-side, see below), or poll `GET /kyc-sessions/{session_id}` until `status == "kyc_completed"`.
6. **Allocate capacity (optional)** - `POST /accounts/{auth_id}/concurrency/transfer` once per resource if the customer needs more than the default CPS / concurrent-call limits.
7. **Provision usage** - assign a DID via Console, then hand the customer their `auth_id`/`auth_token` to drive voice ops on `/api/v1/Account/{auth_id}/*`.
8. **Monitor** - per-customer `cdrs` / `transactions` for billing; recharge before `cash_credits` hits zero.

## Webhook verification snippet

Re-fetch the session before trusting the POST body; respond `200` fast; stay idempotent (retries with exponential backoff on non-2xx).

```python
event = request.json()
session = httpx.get(f"{BASE}/kyc-sessions/{event['session_id']}", headers=PARTNER_HEADERS).json()
if session["status"] == "kyc_completed":
    activate_customer(session["account_auth_id"])  # idempotent
return ("", 200)
```

## When to search docs

- Full reseller onboarding flow → `partner/flow`
- Auth scopes, JWT login, auth errors → `partner/api/authentication`
- Customer creation and optional fields → `partner/api/customers`
- Balance transfer errors and checklist → `partner/api/balance`
- CPS / concurrency transfer and reclaim → `partner/api/capacity`
- KYC sessions end-to-end, session object, `verified_data` → `partner/api/kyc-sessions`
- Transaction reconciliation → `partner/api/transactions`
- Sub-account vs partner child account decision → `concepts/sub-accounts-vs-partner-accounts`
