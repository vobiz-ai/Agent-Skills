---
name: vobiz-sub-accounts
description: Manage Vobiz sub-accounts for multi-team or multi-environment isolation - separate billing, credentials, and number pools under one parent account, including per-sub-account India KYC (PAN, GST, CIN, DigiLocker) and hosted KYC sessions.
---

# Vobiz Sub-Accounts skill

Use this when the user wants to **isolate environments** (dev/staging/prod), **separate teams** (sales vs support), or **carve their account into business units** with independent credentials, numbers, and CDR streams under the same parent invoice — and, for `customer_use` sub-accounts, **verify each customer's own KYC** before it can place calls.

## Difference from Partner sub-accounts

| | Sub-Account | Partner customer account |
|---|---|---|
| Parent | Your own account | Partner account |
| Billing | Rolled into parent invoice | Separate wallet, transferred from partner |
| KYC | `personal_use` inherits parent's; `customer_use` does its own | Requires its own KYC |
| API | `/api/v1/accounts/{auth_id}/sub-accounts/*` | `/api/v1/partner/accounts/*` |

If the user wants to **resell** to external customers, they need the **Partner API** (see `vobiz-partner-api` skill). Sub-accounts are for **internal segmentation** or for vending a `customer_use` tenant that runs its own KYC.

## Two credential pairs (the rule that prevents most bugs)

- **Parent `X-Auth-ID` / `X-Auth-Token`** (`MA_…`): used for **all provisioning and administration** — CRUD on sub-accounts, **all KYC calls**, buying/assigning numbers. The sub-account is named by `{sub_auth_id}` in the URL.
- **Sub-account `auth_id` / `auth_token`** (`SA_…`): returned **once** at creation; the sub-account's **runtime** key for placing/receiving calls and pulling its own CDRs.

Rule of thumb: setting it up → authenticate as the **parent**; using the service → the sub-account authenticates as **itself**. KYC is **not** an exception — it is a parent-authenticated call.

## CRUD endpoints

| Op | Path | Notes |
|---|---|---|
| List | `GET /api/v1/accounts/{auth_id}/sub-accounts/` | Note plural, lowercase `accounts`. Paginated (`page`, `size`). |
| Create | `POST /api/v1/accounts/{auth_id}/sub-accounts/` | Body: `{name, email?, password?, kyc_mode?, business_type?, enabled?, rate_limit?, permissions?}`. Only `name` is required. |
| Get | `GET /api/v1/accounts/{auth_id}/sub-accounts/{sub_auth_id}` | `auth_token` masked here. |
| Update | `PUT /api/v1/accounts/{auth_id}/sub-accounts/{sub_auth_id}` | Body: `{name?, enabled?, kyc_mode?}`. |
| Delete | `DELETE /api/v1/accounts/{auth_id}/sub-accounts/{sub_auth_id}` | Permanent; revokes credentials. |

### Create fields that matter

- `kyc_mode`: `personal_use` *(default)* inherits parent KYC; `customer_use` requires the sub-account to run its own KYC and **requires `email`** (omit it → `400`).
- `business_type` (enum): `individual`, `proprietorship`, `private_limited`, `llp`, `partnership`, `public_limited`, `trust`, `society`, `huf`, `government`. Drives which documents are required.
- `rate_limit`: defaults to `1000`. `enabled`: defaults to `true`.

### Response shape on create

```json
{
  "message": "Sub-account created successfully",
  "sub_account": { "id": "500001", "auth_id": "SA_XXXXXXXX", "auth_token": "shown-once",
                   "parent_auth_id": "MA_XXXXXXXX", "kyc_mode": "customer_use", "kyc_calls_blocked": true },
  "auth_credentials": { "auth_id": "SA_XXXXXXXX", "auth_token": "shown-once" },
  "tokens": { "access_token": "eyJ…", "refresh_token": "eyJ…", "token_type": "bearer", "expires_in": 1800 }
}
```

`sub_account.auth_id` (`SA_…`) is what later calls expect as `{sub_auth_id}`. The numeric `id` is NOT the same. `tokens` (JWT) are only for console sign-in — use `auth_id`/`auth_token` for the REST/telephony API.

## KYC (customer_use only) — authenticate as the PARENT

All KYC endpoints live at `/api/v1/sub-accounts/{sub_auth_id}/kyc/…` (real) or `/api/v1/sub-accounts/test/{sub_auth_id}/kyc/…` (mock) and authenticate with the **parent's** `X-Auth-ID`/`X-Auth-Token`. The sub-account is in the path.

| Op | Path | Body |
|---|---|---|
| Status | `GET .../kyc/status` | — → `{overall_status, kyc_calls_blocked, verifications:{pan,gst,cin,aadhaar}}` |
| PAN | `POST .../kyc/verify-pan` | `{pan}` (exactly 10 chars) |
| GST | `POST .../kyc/verify-gst` | `{gstin}` (exactly 15 chars) |
| CIN search | `POST .../kyc/cin/search` | `{company_name}` → `{matches:[{cin,...}]}` |
| CIN confirm | `POST .../kyc/cin/confirm` | `{company_name, selected_cin}` (verbatim from search) |
| DigiLocker initiate | `POST .../kyc/digilocker/initiate` | `{redirect_url, oauth_state?}` → `{auth_url, access_request_id}` |
| DigiLocker verify | `POST .../kyc/digilocker/verify` | `{access_request_id, linked_number?}` |
| Hosted session | `POST .../kyc-sessions` | `{account_auth_id, flow_type, customer_email? / redirect_url?, webhook_url?, expires_in_days?}` |

### business_type → required documents

| `business_type` | Documents |
|---|---|
| `individual`, `proprietorship`, `huf` | PAN + Aadhaar (DigiLocker) |
| `private_limited`, `public_limited`, `llp`, `partnership` | PAN + GST and/or CIN |
| `trust`, `society`, `government` | PAN + supporting registration (CIN/GST as applicable) |

### Create → KYC → unblock recipe

```bash
# 1. Create a customer_use sub-account (parent auth). Returns kyc_calls_blocked: true.
curl -X POST "$BASE/accounts/$MA/sub-accounts/" -H "X-Auth-ID: $MA" -H "X-Auth-Token: $MA_TOK" \
  -H "Content-Type: application/json" \
  -d '{"name":"Acme","email":"o@acme.com","kyc_mode":"customer_use","business_type":"private_limited"}'

# 2. Verify documents (parent auth, sub in path).
curl -X POST "$BASE/sub-accounts/$SA/kyc/verify-pan" -H "X-Auth-ID: $MA" -H "X-Auth-Token: $MA_TOK" \
  -H "Content-Type: application/json" -d '{"pan":"ABCDE1234F"}'
curl -X POST "$BASE/sub-accounts/$SA/kyc/verify-gst" -H "X-Auth-ID: $MA" -H "X-Auth-Token: $MA_TOK" \
  -H "Content-Type: application/json" -d '{"gstin":"29AAJCN5983D1Z0"}'

# 3. Poll until kyc_calls_blocked is false.
curl "$BASE/sub-accounts/$SA/kyc/status" -H "X-Auth-ID: $MA" -H "X-Auth-Token: $MA_TOK"
# -> { "overall_status":"verified", "kyc_calls_blocked": false, "verifications": {...} }

# 4. Now the sub-account places calls with ITS OWN auth_id/auth_token.
```

### Test mode (no provider, no real docs)

Magic inputs at `/sub-accounts/test/{sub_auth_id}/kyc/…`: `TESTSUCCESS0001` → verified, `TESTFAIL0001` → failed, `TESTERROR0001` → HTTP 500, `TESTPENDING001` → pending (finalize verified), `TESTPENDING_FAIL` → pending (finalize failed). GST input must still be 15 chars (`TESTSUCCESS0001` already is). DigiLocker verify: `MOCK_AR_SUCCESS`/`MOCK_AR_FAIL`. Drive the async path with `POST .../kyc/finalize-pending` `{verification_type, outcome}`. Test mode writes **real** rows — use a throwaway sub-account.

### Hosted sessions + webhooks

`flow_type: "email"` (default) emails a signed link (needs `customer_email`); `flow_type: "redirect"` returns a `widget_url` (needs `redirect_url`). `expires_in_days` defaults to `7`. Webhook events on `webhook_url`: `kyc.initiated`, `kyc.submitted`, `kyc.completed`, `kyc.failed`, `kyc.session_expired`, `kyc.session_revoked`.

Verify every webhook — header `X-Vobiz-Signature: sha256=<hex>`, HMAC-SHA256 over the **raw body**, secret = the **parent account's `auth_token`**:

```python
import hmac, hashlib
def verify(raw_body: bytes, header: str, parent_auth_token: str) -> bool:
    expected = "sha256=" + hmac.new(parent_auth_token.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)
```

Return `2xx` to ack; failures are retried with exponential backoff.

## Pitfalls

- **Path casing is significant.** Sub-account CRUD + KYC use plural, **lowercase** `accounts` / `sub-accounts` (`/api/v1/accounts/…`, `/api/v1/sub-accounts/…`). Singular or capitalized casing returns 401/404 even with valid credentials. (Number inventory/purchase/calls use `/Account/` with a capital A — a different surface.)
- **KYC authenticates as the parent**, not the sub-account. The sub-account is the `{sub_auth_id}` path segment, never the `X-Auth-ID`. Using the sub-account's own key on a KYC call → `403`.
- **`customer_use` requires `email`** at create, and again when promoting via update — otherwise `400`. Promoting a live sub-account to `customer_use` can immediately set `kyc_calls_blocked: true` and block its calls.
- The numeric `id` is NOT the `auth_id`. Use `auth_id` (`SA_…`) for further API calls and as `{sub_auth_id}`.
- **`auth_token` is shown once** at creation; it is `<redacted>` on every later read. Lose it → regenerate.
- A single **`failed`** document keeps `kyc_calls_blocked: true`. Re-run that one verification to clear it; don't re-run the whole flow.
- **CIN confirm** needs a `selected_cin` taken **verbatim** from a search match — hand-typed CINs won't verify.
- **DigiLocker** is a browser OAuth round-trip: store `access_request_id` before redirecting and only call verify after the customer returns.
- A disabled sub-account stops accepting traffic but **doesn't release its phone numbers**.
- Hosted-session `expires_in_days` defaults to **7** for the sub-account endpoint (the partner endpoint also defaults to 7); `reminder_schedule` and `metadata` are partner-endpoint extras.
