---
name: vobiz-sip-trunking
description: Manage Vobiz SIP trunks, IP ACLs, SIP credentials, and origination URIs for BYOC and outbound routing.
license: MIT
compatibility: Vobiz REST API v1.
metadata:
  author: vobiz
  version: "1.1"
---

# Vobiz SIP trunking skill

Use this when the user is **provisioning their own SIP infrastructure** - connecting an on-prem PBX, a BYOC carrier, or restricting trunk access by IP.

Base URL: `https://api.vobiz.ai/api/v1`. Auth on every request: `X-Auth-ID` and `X-Auth-Token` headers (your Auth ID / Auth Token). JSON bodies use `Content-Type: application/json`.

## Resources

- **Trunks** - the parent SIP container. `POST /Account/{auth_id}/trunks`, `GET` (list and `/{trunk_id}`), `PUT /{trunk_id}`, `DELETE /{trunk_id}`. Creating a trunk auto-generates a SIP domain `{trunk_id}.sip.vobiz.ai`, returned as `trunk_domain`.
- **Credentials** - username/password for SIP digest auth. `POST /Account/{auth_id}/credentials`, list at `GET /Account/{auth_id}/trunks/credentials`, `PUT`/`DELETE /Account/{auth_id}/credentials/{credential_id}`.
- **IP ACL** - IPv4 address/CIDR allowlist for inbound SIP traffic. `POST /Account/{auth_id}/ip-acl`, list at `GET /Account/{auth_id}/trunks/ip-acl`, `PUT`/`DELETE /Account/{auth_id}/ip-acl/{ip_acl_id}`.
- **Origination URIs** - outbound SIP routing destinations with priority/weight failover. `POST /Account/{auth_id}/origination-uris`, list at `GET /Account/{auth_id}/trunks/origination-uris`, `PUT`/`DELETE /Account/{auth_id}/origination-uris/{uri_id}`.
- **Number assignment** - route a DID to a trunk. `POST /Account/{auth_id}/numbers/{phone_number}/assign` with body `{ "trunk_group_id": "<trunk_id>" }`; `DELETE` on the same path unassigns. Both return `204`.

## The four auth/routing layers

A trunk can combine four independent controls. Know which one the user needs before reaching for an endpoint:

1. **Credentials** (SIP digest) - username/password. Use for dynamic IPs, softphones, remote workers, AI agents. Works behind NAT. The SIP client must present the `realm` returned on the credential object alongside the username and password.
2. **IP ACL** - allowlist a source IPv4 address or CIDR. Use for static-IP PBXs, SBCs, and carrier interconnects. No per-client config, faster (no auth exchange), but public-IP only.
3. **Origination URI** - **outbound** termination targets. `priority` (lower wins, tried first) selects the route; `weight` splits traffic across same-priority URIs (weight 20 gets 2x the traffic of weight 10).
4. **Webhooks (callbacks)** - **informational only**. Set `webhook_url` (public http/https) and `webhook_method` (`POST` default, or `GET`) on the trunk. Events: `CallInitiated` (with `Allowed`/`Reason`), `Hangup` (with `Billsec`/`MOS`/`Cost`), and `recording.completed` (only when recording is enabled on the trunk). Fail-open and non-blocking; a webhook response cannot accept or reject a call.

Credentials and IP ACL can be combined for defense-in-depth (require both a whitelisted IP AND valid digest auth).

## Trunk lifecycle

1. `POST /Account/{auth_id}/trunks`. Per the OpenAPI contract only `name` is required; optional fields include `trunk_direction` (`inbound` or `outbound`), `transport` (`udp`/`tcp`/`tls`), `secure` (SRTP media; signalling encryption is set by `transport`), `concurrent_calls_limit`, `cps_limit`, `webhook_url`, `webhook_method`, `recording`, `recording_webhook_enabled`, `credential_uuid`, `ipacl_uuid`, `primary_uri_uuid`, `fallback_uri_uuid`. Returns `trunk_id` and auto `trunk_domain`. See the note under Pitfalls about the create page's three-field form.
2. (optional) `POST /credentials` and/or `POST /ip-acl` for inbound auth.
3. (optional) `POST /origination-uris` for outbound BYOC routing (set `priority`; `weight` is Console-managed after creation).
4. `POST /Account/{auth_id}/numbers/{number}/assign` to route a DID inbound.
5. `PUT /Account/{auth_id}/trunks/{trunk_id}` to update. The OpenAPI contract lists every field as optional (same field set as create, with `trunk_status` `enabled`/`disabled`); the update page body shows a three-field replace (`name`, `max_concurrent_calls`, `enabled`). Fetch the trunk first and resend values you want to keep.
6. `DELETE /Account/{auth_id}/trunks/{trunk_id}` to remove (or disable it via `PUT` to keep config).

## End-to-end BYOC recipe

```bash
AUTH=MA_XXXXXXXX
BASE=https://api.vobiz.ai/api/v1/Account/$AUTH
H=(-H "X-Auth-ID: $AUTH" -H "X-Auth-Token: $TOKEN" -H "Content-Type: application/json")

# 1. Create the trunk (OpenAPI contract: only name is required)
TRUNK=$(curl -s -X POST "$BASE/trunks" "${H[@]}" \
  -d '{"name":"Acme BYOC","trunk_direction":"outbound","transport":"udp","concurrent_calls_limit":10}' \
  | jq -r .trunk_id)
# trunk_domain is now $TRUNK.sip.vobiz.ai

# 2a. Inbound auth via IP ACL (name + ip_address required; CIDR allowed)
curl -s -X POST "$BASE/ip-acl" "${H[@]}" \
  -d '{"name":"Prod SBC","ip_address":"10.20.30.0/24"}'

# 2b. ...or via digest credentials (username + password required; save the password now)
curl -s -X POST "$BASE/credentials" "${H[@]}" \
  -d '{"username":"acme_sip_user_01","password":"S3cure-Pass-!2025"}'

# 3. Outbound routing: primary + failover (name + sip_uri + priority required)
curl -s -X POST "$BASE/origination-uris" "${H[@]}" \
  -d '{"name":"Primary SBC","sip_uri":"sip:sbc1.example.com:5060","priority":1}'
curl -s -X POST "$BASE/origination-uris" "${H[@]}" \
  -d '{"name":"Backup SBC","sip_uri":"sip:sbc2.example.com:5060","priority":2}'

# 4. Route an inbound DID to the trunk (URL-encode + as %2B)
curl -s -X POST "$BASE/numbers/%2B919876543210/assign" "${H[@]}" \
  -d "{\"trunk_group_id\":\"$TRUNK\"}"
```

## Webhook verification

Trunk webhooks (callbacks) are not signed: there is no signing secret and no signature header. To verify a request came from Vobiz, allow-list the Vobiz callback IPs listed on [IP Whitelisting](https://vobiz.ai/docs/concepts/ip-whitelisting#callbacks), and use an unguessable path or token in the webhook URL.

Headers sent: `Content-Type: application/json`, `User-Agent: Vobiz/1.0`, `X-Vobiz-Event` (`CallInitiated`, `Hangup`, or `recording.completed`), and `X-Vobiz-Request-ID`. Link `CallInitiated` and `Hangup` for the same call by `SIPCallID` - `CallUUID` in `Hangup` is Vobiz's internal ID and can differ. `X-Vobiz-Request-ID` is unique per delivery; use it only to de-duplicate retries. `recording.completed` uses snake_case fields (`sip_call_id`, `recording_url`, ...).

`webhook_url` is SSRF-validated: it must be a public http/https URL. Localhost, private (RFC 1918), and link-local/cloud-metadata addresses are rejected with `invalid webhook_url`.

## Pitfalls

- **Path casing:** Trunk, credential, IP ACL, origination-URI, and number-to-trunk endpoints all use **uppercase** `Account` (`/api/v1/Account/...`). The lowercase `/api/v1/account/...` form is used only by a few number endpoints (sub-account assign/unassign, number health, cancel-release) - do not copy that casing onto trunk routes.
- **List paths differ from create paths.** You create at `/Account/{auth_id}/credentials` (and `/ip-acl`, `/origination-uris`) but list at `/Account/{auth_id}/trunks/credentials` (and `/trunks/ip-acl`, `/trunks/origination-uris`). The `trunks/` segment appears only on the list routes. Update/delete use the create path plus `/{id}`.
- **Create vs response field names diverge.** Origination URI create body uses `name` + `sip_uri`; the response returns them as `description` + `uri`. IP ACL create uses `name` + `ip_address`; the response returns `description` + `ip_address`. Credential responses echo `username` and add `realm`. Read the response schema, not the request.
- **Trunk create/update field names: two forms in the docs.** The OpenAPI contract (and the API playground on the create page) uses `trunk_direction` (`inbound`/`outbound`), `concurrent_calls_limit`, `cps_limit`, `trunk_status` (`enabled`/`disabled`), with only `name` required. The body text of `trunks/create-trunk` and `trunks/update-trunk` instead shows `name` + `trunk_type` (`INBOUND`/`OUTBOUND`) + `max_concurrent_calls` (and `enabled` on update) as the required set. The response always reports `trunk_direction`, `concurrent_calls_limit`, `cps_limit`, and `trunk_status`. If a request is rejected, check the other form and the page.
- **Required create fields are minimal.** IP ACL: `name` + `ip_address`. Credential: `username` + `password` (duplicate `username` returns `409`). Origination URI: `name` + `sip_uri` + `priority`; `weight` (default `10`) and `transport` are set after creation in the Console. Trunk: `name` (see above).
- **Origination URI format.** The API pages (`trunks/origination-uri/create-origination-uri`, `origination-uri-object`) show `sip:` examples (`sip:host`, `sip:user@host:port`); some vendor integration pages ask for the hostname or `host:port` only in the Console URI field. Follow the create page for API calls.
- **Origination URI update is a replace of `name` + `priority` only.** The PUT body does not take `uri`, `weight`, or `enabled`. To change the destination, create a new URI and delete the old one; to rebalance `weight` or toggle `enabled`, use the Console.
- **IP ACL accepts CIDR**, not just single IPs (e.g. `10.20.30.0/24`). IPv4 only; IPv6 is rejected. The field is `ip_address` (not `cidr_ip` as one spec example shows). The create page states that RFC 5737 documentation prefixes (such as `203.0.113.0/24`) are rejected; use a real public or RFC 1918 (`10.0.0.0/24`) range. Update is a replace of `name` + `ip_address`; the `enabled` toggle is Console-managed.
- **Write-only secrets.** Credential `password` is never returned (shows as `<redacted>`). Save it at creation; `username` is immutable - rotate by creating a new credential and deleting the old one. The OpenAPI contract for credential update takes `password`; the update page also documents `enabled` and `description`.
- **Disable vs delete.** Prefer disabling a trunk via `PUT` (`trunk_status: disabled` per the OpenAPI contract; `enabled: false` on the update page) over `DELETE` to preserve config. `DELETE` on a trunk cascades to its credentials, IP ACLs, and origination URIs, stops active calls, and can return **409** if calls are active. Deleting a credential can also return `409` while it is in use by active calls.
- **Number assignment body field is `trunk_group_id`** (the trunk's UUID), not `trunk_id`. URL-encode the phone number with `%2B` for `+` or you get a 404. Assigning a number that is already on a trunk returns `400` - unassign it first.
- **`cps_limit` vs `concurrent_calls_limit`:** CPS caps new call attempts per second (flood protection); concurrent caps simultaneous live calls (capacity/cost). Defaults seen in responses: `cps_limit: 2`, `concurrent_calls_limit: 10`. The OpenAPI contract notes these values are stored on the trunk while the enforced limits are account-level.

## When to search docs

- "How do I attach a number to a trunk?" → `trunks/assign-number`
- "What IP ranges / CIDR work?" → `trunks/ip-acl/create-ip-acl`
- "Outbound routing with priority + fallback / load balancing" → `trunks/origination-uri/create-origination-uri`
- "Credentials vs IP ACL - which auth?" → `trunks/ip-acl` (comparison table)
- "Trunk webhook events / payloads / verification" → `trunks/webhook`
- "Which Vobiz IPs to allow for SIP, RTP, and callbacks?" → `concepts/ip-whitelisting`
