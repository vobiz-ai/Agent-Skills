---
name: vobiz-sub-accounts
description: Manage Vobiz sub-accounts for multi-team or multi-environment isolation - separate billing, credentials, and number pools under one parent account, including per-sub-account India KYC (PAN, GST, CIN, DigiLocker) and hosted KYC sessions.
license: MIT
compatibility: Vobiz REST API v1.
metadata:
  author: vobiz
  version: "1.1"
---

# Vobiz Sub-Accounts skill

Use this when the user wants to **isolate environments** (dev/staging/prod), **separate teams** (sales vs support), or **carve their account into business units** with independent credentials, numbers, and CDR streams under the same parent wallet - and, for `customer_use` sub-accounts, **verify each customer's own KYC** before it can place calls.

## Difference from partner child accounts

| | Sub-account | Partner child account |
|---|---|---|
| Parent | Your own main account (`MA_…`) | Partner master account (`PA_…`) |
| Balance | Shared - all usage is charged to the parent wallet | Separate wallet per child, funded by balance transfer (or invoiced by Vobiz directly) |
| CPS / concurrency | One limit on the parent, pooled across every sub-account | Own limit per child, transferred/reclaimed from the partner pool |
| KYC | `personal_use` inherits parent's; `customer_use` does its own | Each child completes its own KYC via a partner KYC session |
| API | `/api/v1/accounts/{auth_id}/sub-accounts/*` | `/api/v1/partner/accounts/*` |

If the user wants customers billed, supported, and capacity-managed as independent Vobiz customers, they need the **Partner API** (see `vobiz-partner-api` skill). Sub-accounts suit one bill, one capacity pool, and your own pricing. Many platforms use both - see `concepts/sub-accounts-vs-partner-accounts`.

## Two credential pairs (the rule that prevents most bugs)

- **Parent `X-Auth-ID` / `X-Auth-Token`** (`MA_…`): used for **all provisioning and administration** - CRUD on sub-accounts, **all KYC calls**, buying/assigning/unassigning numbers. The sub-account is named by `{sub_auth_id}` in the URL.
- **Sub-account `auth_id` / `auth_token`** (`SA_…`): returned **once** at creation; the sub-account's **runtime** key for placing/receiving calls, buying its own numbers, and pulling its own CDRs.

Rule of thumb: setting it up → authenticate as the **parent**; using the service → the sub-account authenticates as **itself**. KYC is **not** an exception - it is a parent-authenticated call.

## CRUD endpoints

Base: `https://api.vobiz.ai/api/v1`. All five authenticate as the parent.

| Op | Path | Notes |
|---|---|---|
| List | `GET /accounts/{auth_id}/sub-accounts/` | Plural, lowercase `accounts`. Query: `page` (default 1), `size` (default 25), `active_only`. Response: `{sub_accounts[], total, page, size}`. |
| Create | `POST /accounts/{auth_id}/sub-accounts/` | Body: `{name, email?, phone?, description?, password?, kyc_mode?, business_type?, enabled?, rate_limit?, permissions?}`. Only `name` is required. Returns `201`. |
| Get | `GET /accounts/{auth_id}/sub-accounts/{sub_auth_id}` | `auth_token` is `<redacted>` here. |
| Update | `PUT /accounts/{auth_id}/sub-accounts/{sub_auth_id}` | Body: `{name?, email?, phone?, description?, rate_limit?, permissions?, enabled?, kyc_mode?}`. Omitted fields stay unchanged. |
| Delete | `DELETE /accounts/{auth_id}/sub-accounts/{sub_auth_id}` | Permanent; revokes credentials. Returns `200 {message}` (or `204`). Parent auth only - you don't need the lost sub-account token. |

### Create fields that matter

- `kyc_mode`: `personal_use` *(default)* inherits parent KYC; `customer_use` requires the sub-account to run its own KYC and **requires `email`** (omit it → `400`).
- `business_type` (enum): `individual`, `proprietorship`, `private_limited`, `llp`, `partnership`, `public_limited`, `trust`, `society`, `huf`, `government`. Drives which documents are required.
- `rate_limit`: defaults to `1000`. `enabled`: defaults to `true`. `permissions`: object with booleans `calls` and `cdr`; omitted → inherits the parent's default access.
- `password`: console login only; the REST credentials (`auth_id`/`auth_token`) are generated regardless.
- Errors: `400` (`customer_use` without `email`), `401` (bad credentials or wrong path casing), `404` (`{auth_id}` is not a main account you own).

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

`sub_account.auth_id` (`SA_…`) is what later calls expect as `{sub_auth_id}`. The numeric `id` is NOT the same. `tokens` (JWT) are only for console sign-in - use `auth_id`/`auth_token` for the REST/telephony API.

## Phone numbers - assign, unassign, or let the sub-account buy

| Op | Path | Auth | Notes |
|---|---|---|---|
| Buy into parent pool | `POST /Account/{auth_id}/numbers/purchase-from-inventory` `{e164}` | Parent | Debits the parent (setup + monthly fee). Capital-A `/Account/`. |
| Assign to sub-account | `POST /account/{auth_id}/numbers/{e164}/assign-subaccount` `{sub_account_id: "SA_…"}` | Parent | **Lowercase `/account/`**, singular `assign-subaccount`. URL-encode `+` as `%2B`. `200`/`204` no body; `404` if number or sub-account not found. |
| Unassign (back to parent pool) | `DELETE /account/{auth_id}/numbers/{e164}/assign-subaccount` | Parent | **15-day cool-off**: a DID with a call in the last 15 days → `409 did_cool_off_in_effect` with `cool_off_until` / `cool_off_remaining_seconds`. Never-used DIDs move immediately. `?force=true` is admin-role only (`403` otherwise). |
| Sub-account buys its own | `POST /Account/{sub_auth_id}/numbers/purchase-from-inventory` | Sub-account | Lands directly on the sub-account; billing still routes to the parent `MA_`. |

Purchasing spends money - confirm the number and fee with the user before firing. See `sub-accounts/onboarding-flow` and `account-phone-number/unassign-subaccount`.

## KYC (customer_use only) - authenticate as the PARENT

All KYC endpoints live at `/api/v1/sub-accounts/{sub_auth_id}/kyc/…` (real) or `/api/v1/sub-accounts/test/{sub_auth_id}/kyc/…` (mock) and authenticate with the **parent's** `X-Auth-ID`/`X-Auth-Token`. The sub-account is in the path.

| Op | Path | Body |
|---|---|---|
| Status | `GET .../kyc/status` | - → `{sub_account_id, kyc_mode, business_type, overall_status, kyc_calls_blocked, verifications:{pan,gst,cin,aadhaar}}`. `403` if the caller isn't the parent; `404` unknown sub-account. |
| PAN | `POST .../kyc/verify-pan` | `{pan}` (exactly 10 chars; `400` otherwise) |
| GST | `POST .../kyc/verify-gst` | `{gstin}` (exactly 15 chars; `400` otherwise) |
| CIN search | `POST .../kyc/cin/search` | `{company_name}` → `{matches:[{cin, company_name, status}]}` |
| CIN confirm | `POST .../kyc/cin/confirm` | `{company_name, selected_cin}` (verbatim from search) - this is the step that persists |
| DigiLocker initiate | `POST .../kyc/digilocker/initiate` | `{redirect_url, oauth_state?}` → `{auth_url, access_request_id}` |
| DigiLocker verify | `POST .../kyc/digilocker/verify` | `{access_request_id, linked_number?}` |
| Hosted session | `POST .../kyc-sessions` | `{account_auth_id, flow_type, customer_email? / redirect_url?, webhook_url?, expires_in_days?}` → `201`; `422` if the flow's required field is missing |

Each verification returns `{verification_type, status, kyc_calls_blocked?}` with `status` ∈ `verified | failed | pending`; `overall_status` and each entry in `verifications` ∈ `not_started | pending | verified | failed`.

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

Magic inputs at `/sub-accounts/test/{sub_auth_id}/kyc/…`: `TESTSUCCESS0001` → verified, `TESTFAIL0001` → failed, `TESTERROR0001` → HTTP 500, `TESTPENDING001` → pending (finalize verified), `TESTPENDING_FAIL` → pending (finalize failed). GST input must still be 15 chars (`TESTSUCCESS0001` already is - pass it as-is, don't append a suffix; pad shorter magic inputs to 15). Mock CIN confirm succeeds when `selected_cin` starts with `U72900KA2024PTC123456`. DigiLocker verify: `MOCK_AR_SUCCESS`/`MOCK_AR_FAIL`. Drive the async path with `POST .../kyc/finalize-pending` `{verification_type: pan|aadhaar|gst|cin, outcome: verified|failed}`. Responses carry `mock: true`. Test mode writes **real** rows - use a throwaway sub-account.

### Hosted sessions + webhooks (callbacks)

`flow_type: "email"` (default) emails a signed link from `kyc@vobiz.ai` hosted at `kyc.vobiz.ai` (needs `customer_email`; session starts `email_sent`); `flow_type: "redirect"` returns a `widget_url` (needs `redirect_url`; session starts `link_ready`). `expires_in_days` defaults to `7`. No `webhook_url` → no callbacks. Webhook events on `webhook_url`: `kyc.initiated`, `kyc.submitted`, `kyc.completed`, `kyc.failed`, `kyc.session_expired`, `kyc.session_revoked`. Payload carries `event`, `session_id`, `account_auth_id`, `kyc_type`, `session_status`, `metadata`.

Verify every KYC webhook - header `X-Vobiz-Signature: sha256=<hex>`, HMAC-SHA256 over the **raw body**, secret = the **parent account's `auth_token`**:

```python
import hmac, hashlib
def verify(raw_body: bytes, header: str, parent_auth_token: str) -> bool:
    expected = "sha256=" + hmac.new(parent_auth_token.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)
```

Return `2xx` to ack; failures are retried with exponential backoff - keep the handler idempotent.

### Voice callbacks on sub-accounts are signed differently

The scheme above is for **KYC session webhooks** only. Answer/hangup/action **call callbacks** for a sub-account use the standard callback signatures (`X-Vobiz-Signature-V2` / `-V3`, base64 HMAC-SHA256 over `url + nonce`, keyed with the **sub-account's** auth token) **plus** `X-Vobiz-Signature-MA-V2` / `X-Vobiz-Signature-MA-V3`, the same computation keyed with the **parent's** auth token - so you can validate with either key. See https://vobiz.ai/docs/concepts/validating-callbacks#sub-account-callbacks.

## Pitfalls

- **Path casing is significant.** Sub-account CRUD + KYC use plural, **lowercase** `accounts` / `sub-accounts` (`/api/v1/accounts/…`, `/api/v1/sub-accounts/…`); assign/unassign use lowercase singular `/api/v1/account/{auth_id}/numbers/…/assign-subaccount`; inventory, purchase, and calls use capital-A `/Account/`. The wrong casing returns 401/404 even with valid credentials.
- **KYC authenticates as the parent**, not the sub-account. The sub-account is the `{sub_auth_id}` path segment, never the `X-Auth-ID`. A caller that isn't the parent → `403`.
- **`customer_use` requires `email`** - at create (`400` without it) and when promoting via `PUT` (the sub-account must already have an `email`; the docs' example sets `email` and `kyc_mode` in the same PUT). Promoting a live sub-account to `customer_use` re-derives `kyc_calls_blocked` and can immediately block its calls; demoting to `personal_use` sets it `false`.
- The numeric `id` is NOT the `auth_id`. Use `auth_id` (`SA_…`) for further API calls and as `{sub_auth_id}`.
- **`auth_token` is shown once** at creation; it is `<redacted>` on every later read. Lose it → rotate from the Console (**Setup → Subaccounts → Reset Credentials / Rotate Token**; the old token stays valid for a grace window). The parent can still list/delete the sub-account without its token.
- A single **`failed`** document keeps `kyc_calls_blocked: true`. Re-run that one verification with corrected input; don't re-run the whole flow.
- **CIN confirm** needs a `selected_cin` taken **verbatim** from a search match - hand-typed CINs won't verify. Search alone persists nothing.
- **DigiLocker** is a browser OAuth round-trip: store `access_request_id` (and your `oauth_state`) before redirecting and only call verify after the customer returns.
- A disabled sub-account stops accepting traffic but **doesn't release its phone numbers**.
- **Unassigning a used DID is blocked for 15 days** after its last call (`409 did_cool_off_in_effect`). Plan number moves between customers around that.
- Hosted-session `expires_in_days` defaults to **7** on both the sub-account and partner endpoints. `reminder_schedule` and `metadata` are documented on the partner endpoint's contract; the sub-account walkthroughs (`sub-accounts/kyc/email-flow-webhook`) also pass them - check that page before relying on them here.

## When to search docs

- Full create → KYC → number → first call walkthrough → `sub-accounts/onboarding-flow`
- KYC overview and document matrix → `sub-accounts/kyc/overview`
- Hosted session, webhook events, signature → `sub-accounts/kyc/email-flow-webhook`, `sub-accounts/kyc/redirect-flow-webhook`
- Test mode → `sub-accounts/kyc/test-mode`
- Sub-account vs partner decision → `concepts/sub-accounts-vs-partner-accounts`
- Console management (rotate token, assign numbers) → `platform/subaccounts`
