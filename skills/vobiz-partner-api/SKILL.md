---
name: vobiz-partner-api
description: Vobiz Partner / reseller API - create customer sub-accounts, transfer balance, run KYC sessions (PAN+Aadhaar OTP, no documents), and query per-customer CDRs/transactions/numbers.
---

# Vobiz Partner API skill

Use this for **white-label / reseller workflows** where one master partner account owns multiple customer sub-accounts. Auth uses the **same** `X-Auth-ID` / `X-Auth-Token` headers as the main API - there is **no separate Bearer JWT**. Just use your partner-tier credentials.

## Base path

`https://api.vobiz.ai/api/v1/partner/*`

(Routes do NOT live under `/partner/v1/` - that path returns 404.)

## Partner profile + dashboard

- `GET /api/v1/partner/me` - your partner profile
- `GET /api/v1/partner/dashboard` - aggregated KPIs (total customers, balance held, MTD revenue)

Auth model in one line: **partner creds (`PA_…`) drive `/api/v1/partner/*`; the customer's own creds (`MA_…`) drive `/api/v1/Account/{customer_auth_id}/*` voice ops.** There is no Bearer JWT for partner server-to-server calls (JWT login is for interactive sessions only).

## Customer accounts

- `GET /api/v1/partner/accounts` - list customers. Pagination params are `page` + `per_page` (default 20); the **response** echoes `per_page` back as `size`. `search` substring-matches name or email.
- `POST /api/v1/partner/accounts` - create customer. Required: `name, email, phone, password, country`. Optional (guaranteed): `company`. Returns `201` with the customer's own `auth_id` + `auth_token` - **the `auth_token` is shown only once**; store it encrypted, it's the customer's API credential. `email` must be platform-unique; `password` needs 8+ chars with a number and special char.
- `GET /api/v1/partner/accounts/{customer_auth_id}` - full customer profile (re-fetches `auth_token`).
- `GET /api/v1/partner/accounts/{customer_auth_id}/balance` - real-time wallet (`cash_credits`).

## Balance transfer

- `POST /api/v1/partner/accounts/{customer_auth_id}/transfer-balance` - atomic debit/credit. Body: `{amount, currency, description?}`. `currency` must equal your partner currency (no FX conversion). **Permanent - cannot be reversed; no idempotency key** (poll customer transactions before retrying a timed-out request). `400` on insufficient partner balance (`current_balance`/`requested_amount` in payload) or currency mismatch; `404` if the customer isn't yours.

## KYC sessions (eKYC, no document uploads)

eKYC pulls data from govt APIs - no uploads. Individual (PAN 4th char = `P`): PAN + DOB → Aadhaar via DigiLocker (Govt OTP). Company (PAN 4th char = `C`): PAN + entity name → GSTIN. The PAN determines the path; the partner does not pick it.

- `POST /api/v1/partner/kyc-sessions` - body: `{account_auth_id, flow_type?, customer_email?, redirect_url?, webhook_url?, expires_in_days?, reminder_schedule?, metadata?}`.
  - `flow_type: "email"` (default) requires `customer_email`; session starts at **`email_sent`**.
  - `flow_type: "redirect"` requires `redirect_url`; response carries `widget_url` (token `kst_<64hex>`), session starts at **`link_ready`**. After verify, browser returns to `redirect_url?session_id=&status=&auth_id=` - treat those params as a hint only, confirm server-side.
  - `expires_in_days` default **7**. `metadata` is echoed back on GET + webhooks.
- `GET /api/v1/partner/kyc-sessions` - list (filter `status`/`account_auth_id`; paginated with `page`/`size`).
- `GET /api/v1/partner/kyc-sessions/{session_id}` - one session. Note: GET returns the id under `id`, POST returned it under `session_id` (same value).
- `POST /api/v1/partner/kyc-sessions/{session_id}/resend` - resend email. Email-flow only (`400` on redirect flow). The 30-min cooldown starts at session **creation**, so an immediate resend returns `429` with remaining seconds.
- `DELETE /api/v1/partner/kyc-sessions/{session_id}` - revoke (optional `{reason}` body). `409` if already terminal.

Webhook events on `webhook_url`: `kyc.completed`, `kyc.failed`, `kyc.session_revoked`. Statuses: `email_sent | link_ready → opened → in_progress → kyc_completed | revoked`. `kyc_type` is `null` until PAN is submitted.

## Per-customer queries

- `GET /api/v1/partner/accounts/{customer_auth_id}/transactions` - pagination `page`/`per_page`; date filters `from_date`/`to_date`; `transaction_type` ∈ `recharge|debit|refund|transfer`. Each row's `type` field is the ledger direction `credit`/`debit` (different vocabulary from the filter). Includes a `summary` block.
- `GET /api/v1/partner/accounts/{customer_auth_id}/cdrs` - filters `start_date`, `end_date`, `call_direction` (`inbound|outbound`), `status` (`answered|failed|busy|no_answer`), `min_duration`, `hangup_cause`. Response wraps rows in `data[]` + `pagination` + `summary`. Bill on `billsec` (billable seconds) or `summary.total_cost`.
- `GET /api/v1/partner/accounts/{customer_auth_id}/numbers` - pagination `page`/`per_page`; `search` (E.164 substring). Read-only; assign/release via Console.

## Customer-side endpoints

Same auth scheme, but uses the **customer's** `auth_id`/`auth_token` against `/api/v1/Account/{customer_auth_id}/*`. The customer can call these themselves, or the partner can call them on the customer's behalf using the customer's stored `auth_token`.

## Pitfalls

- **Base path is `/api/v1/partner`, not `/partner/v1`.** The latter 404s. Full base: `https://api.vobiz.ai/api/v1/partner`.
- **`balance` is a string with 5 decimals** (`"23906.83000"`). Cast to float before comparing. On `GET /dashboard` the partner `balance` is `null` - use `total_balance` there.
- **Pagination keys differ by endpoint.** Accounts/transactions/cdrs/numbers use `per_page`; KYC sessions list uses `size`. The accounts list *accepts* `per_page` but *returns* it as `size`.
- **Balance transfer is irreversible and has no idempotency key.** Verify `customer_auth_id`, `amount`, and `currency` (must match partner currency) before firing. If a request times out, poll customer transactions before retrying to avoid a double credit.
- **Only one active KYC session per `account_auth_id`.** Creating a new one silently auto-revokes the previous - always operate on the most recent `session_id`, or you'll get `409`s.
- **KYC resend cooldown starts at creation**, so an immediate resend returns `429`. Resend is email-flow only.
- **Redirect-flow return params can be spoofed** - confirm via the `kyc.completed` webhook or a server-side `GET /kyc-sessions/{id}` before unlocking.
- **Never expose tokens client-side.** The partner token and customer `auth_token`s are long-lived money-moving secrets. Keep them server-side; rotate from the Console if leaked.
- **Capability flags gate operations.** `can_create_accounts` / `can_transfer_balance` / `can_view_cdrs` being `false` returns `403`. Check `GET /me`.

## Complete onboarding recipe

Run in order; each step depends on the previous:

1. **Preflight** - `GET /me`; confirm `can_create_accounts` and `can_transfer_balance` are `true` and `float(balance) >= amount`.
2. **Create customer** - `POST /accounts` with `{name,email,phone,password,country,company?}`. Capture `auth_id` + `auth_token` (store the token encrypted - shown once).
3. **Fund wallet** - `POST /accounts/{auth_id}/transfer-balance` with `{amount, currency}` (currency = your partner currency). Assert response `partner_balance_after >= 0` and `status == "completed"`.
4. **Start KYC** - `POST /kyc-sessions` with `{account_auth_id: auth_id, flow_type, webhook_url, expires_in_days:14, ...}` (email flow needs `customer_email`; redirect flow needs `redirect_url`). Store `session_id`.
5. **Wait for completion** - receive `kyc.completed` on `webhook_url` (verify server-side, see below), or poll `GET /kyc-sessions/{session_id}` until `status == "kyc_completed"`.
6. **Provision usage** - assign a DID via Console, then hand the customer their `auth_id`/`auth_token` to drive voice ops on `/api/v1/Account/{auth_id}/*`.
7. **Monitor** - per-customer `cdrs` / `transactions` for billing; recharge before `cash_credits` hits zero.

## Webhook verification snippet

Re-fetch the session before trusting the POST body; respond `200` fast; stay idempotent (retries on non-2xx).

```python
event = request.json()
session = httpx.get(f"{BASE}/kyc-sessions/{event['session_id']}", headers=PARTNER_HEADERS).json()
if session["status"] == "kyc_completed":
    activate_customer(session["account_auth_id"])  # idempotent
return ("", 200)
```

## When to search docs

- Full reseller onboarding flow → `partner/flow`
- Customer creation → `partner/api/customers`
- KYC end-to-end → `partner/flow` (search "kyc")
- Transaction reconciliation → `partner/api/transactions`
