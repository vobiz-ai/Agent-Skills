---
name: vobiz-phone-numbers
description: Discover, purchase, assign, and release Vobiz phone numbers (DIDs) - including 140-series, 1600-series, mobile, and toll-free numbers across countries.
license: MIT
compatibility: Vobiz REST API v1.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz Phone Numbers skill

Use this when the user wants to **buy a new phone number, attach it to a trunk, hand it to a sub-account, check a number's health, or release it**.

Base URL: `https://api.vobiz.ai/api/v1`. Auth: `X-Auth-ID` + `X-Auth-Token` (your Auth ID / Auth Token) on every request.

## Endpoints

| Op | Method + Path | Notes |
|---|---|---|
| Search inventory | `GET /Account/{auth_id}/inventory/numbers` | Filter by `country`, `search` (substring on E.164), `exclude` (E.164 prefixes incl. country code, comma-separated or repeated), `page`, `per_page` (default 25, max 100) |
| Purchase | `POST /Account/{auth_id}/numbers/purchase-from-inventory` | Body: `{ "e164": "+91...", "currency": "INR" }`. Debits `setup_fee + monthly_fee` |
| List owned | `GET /Account/{auth_id}/numbers` | `page`, `per_page` (default 25), `search` (URL-encode a leading `+`). Returns `items` (status, fees, `application_id`, `trunk_group_id`) + `page`/`per_page`/`total`. An `MA_` listing includes its sub-accounts' numbers; check each item's `account_id` |
| Assign to trunk | `POST /Account/{auth_id}/numbers/{phone_number}/assign` | Body: `{ "trunk_group_id": "<uuid>" }`. Path number must be `%2B`-encoded. `204` on success; `400` if already assigned |
| Unassign from trunk | `DELETE /Account/{auth_id}/numbers/{phone_number}/assign` | Path number must be `%2B`-encoded. `204` |
| Assign to sub-account | `POST /account/{auth_id}/numbers/{e164}/assign-subaccount` | **lowercase `account`**, `%2B`-encoded. Body: `{ "sub_account_id": "SA_..." }`. `200`/`204` |
| Unassign from sub-account | `DELETE /account/{auth_id}/numbers/{e164}/assign-subaccount` | **lowercase `account`**. 15-day cool-off (`409`); admin `?force=true` |
| Number health | `GET /account/{auth_id}/numbers/{e164}/health` | **lowercase `account`**. `granularity` (`daily` default, `hourly`), `days` (default 30). Returns `status`, `is_spam`, `summary`, `snapshots[]` |
| Release (unrent) | `DELETE /Account/{auth_id}/numbers/{e164}` | Charges the account-specific `release_fee`; enters `pending_release` for 24 h, then a 45-day quarantine. `?immediate=true` skips the cooldown and cannot be cancelled |
| Cancel release | `POST /account/{auth_id}/numbers/{e164}/cancel-release` | **lowercase `account`**. Only while `pending_release`; restores `active` and refunds the release fee. `400` once the window has passed |

## Casing - copy verbatim

The number endpoints mix two casings. **Match the spec exactly or you get a 404:**

- Capital **`/Account/`**: inventory, purchase, list, assign/unassign **to trunk**, release.
- Lowercase **`/account/`**: assign/unassign **to a sub-account** (singular `assign-subaccount` segment), **number health**, and **cancel-release**.

## Discover → purchase → assign recipe

```bash
# 1. Find an unassigned IN number (inventory returns status=active, auth_id=NULL only)
curl -s "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/inventory/numbers?country=IN&search=80&per_page=25" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN"

# 2. Purchase by e164 (debits setup_fee + monthly_fee from the balance - run only with
#    the user's own credentials after they confirm the number and price)
curl -s -X POST "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/numbers/purchase-from-inventory" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" -H "Content-Type: application/json" \
  -d '{"e164":"+918065551234","currency":"INR"}'

# 3. Route inbound calls to a trunk (note the %2B-encoded +)
curl -s -X POST "https://api.vobiz.ai/api/v1/Account/$AUTH_ID/numbers/%2B918065551234/assign" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" -H "Content-Type: application/json" \
  -d '{"trunk_group_id":"e3e55a78-1234-5678-90ab-cdef12345678"}'

# 4. (Optional) hand the DID to a sub-account - lowercase /account/, singular segment
curl -s -X POST "https://api.vobiz.ai/api/v1/account/$AUTH_ID/numbers/%2B918065551234/assign-subaccount" \
  -H "X-Auth-ID: $AUTH_ID" -H "X-Auth-Token: $AUTH_TOKEN" -H "Content-Type: application/json" \
  -d '{"sub_account_id":"SA_XXXXXX"}'
```

## Pitfalls

- **No `/numbers/search` endpoint.** Inventory search is `GET /Account/{auth_id}/inventory/numbers?country=IN&search=80`. `search` is a substring match on the E.164 string. `exclude` needs the country code (`exclude=9180`, not `80`) and is ANDed with every other filter, so it overrides `search`.
- **Purchase body is `e164`, not `number_id`** (e.g. `{"e164":"+91..."}`). The `currency` field is optional and defaults to the number's currency, else `USD`.
- **Who pays:** when a sub-account (`SA_`) purchases, the **parent master account (`MA_`)** is debited - not the sub-account.
- **Insufficient balance:** the debit fails, the number stays in inventory, and the purchase returns an error (spec documents `404` for not-in-inventory and `500` for a failed debit; confirm the exact insufficient-balance code with the API before relying on `402`).
- **URL-encode the `+`** as `%2B` in every path that carries an E.164 (`/assign`, `/assign-subaccount`, `/health`, release, `/cancel-release`) and in the list `search` query parameter. Endpoints that take the number in the JSON body (purchase) keep the literal `+`.
- **Sub-account 15-day cool-off:** unassigning a DID that had a call in the last 15 days returns `409 did_cool_off_in_effect` with `cool_off_until` and `cool_off_remaining_seconds`. Never-used DIDs (`last_call_at` is `NULL`) move back instantly.
- **Admin force bypass:** `DELETE .../assign-subaccount?force=true` skips the cool-off but requires an **admin-role account** (`403` otherwise) and writes a `did_assignment_audit` row.
- **Release costs money and detaches routing.** A standard release charges the account-specific `release_fee` (returned in the response), detaches the trunk and voice application, stops the monthly rental, and puts the number in `pending_release` for 24 hours (then a 45-day quarantine before it returns to the pool). Cancel within the window with cancel-release to get the fee refunded - the trunk/application are not re-attached. `?immediate=true` skips the window and cannot be cancelled. Returns `403` if your account doesn't own the number. Don't release prod numbers in test scripts.
- **Purchase creates a recurring monthly charge** (`monthly_fee` every billing cycle, see `next_billing_date`) plus the one-time `setup_fee`.
- **Capabilities** (`voice`/`sms`/`mms`/`fax`) are per-number booleans; Vobiz numbers are voice-first and `sms`/`mms`/`fax` are commonly `false`.

## Number shape

`items[]` from list-numbers: `{id, account_id, e164, country, region, capabilities{voice,sms,mms,fax}, status, provider, setup_fee, monthly_fee, currency, application_id, voice_enabled, tags, purchased_at, is_blocked, is_trial_number, last_billing_date, next_billing_date, minimum_commitment_months, aadhaar_verification_required, aadhaar_verified, source, trunk_group_id, released_at, created_at, updated_at}`.

`status` is one of `active`, `pending_purchase`, `pending_release`, `released`, `blocked`.

## When to search docs

- "How do I attach my number to an application?" → `applications/attach-number`
- "What's the difference between assigning to a trunk vs an app?" → `account-phone-number` overview
- "Country availability" → `account-phone-number/list-inventory-numbers`
- "Hand a number to a sub-account / cool-off" → `account-phone-number/assign-subaccount`, `account-phone-number/unassign-subaccount`
- "Release fee / cooldown / undo a release" → `account-phone-number/unrent-number`, `account-phone-number/cancel-release`
- "Is my caller ID flagged as spam / answer rate" → `account-phone-number/get-number-health`, `best-practices/number-health-guide`
- "140 / 160 / 92 series India numbers" → `faq/number-series` (requested through support)
