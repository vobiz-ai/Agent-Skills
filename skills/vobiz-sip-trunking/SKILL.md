---
name: vobiz-sip-trunking
description: Create and manage Vobiz SIP trunks, IP ACLs, SIP credentials, origination URIs, transport, TLS/SRTP, concurrency, and CPS limits. Use when connecting a PBX, contact center, carrier, or AI voice platform through BYOC, configuring inbound or outbound routing, load balancing/failover, or troubleshooting SIP authentication.
---

# Vobiz SIP trunking skill

Use this when the user is **provisioning their own SIP infrastructure** - connecting an on-prem PBX, a BYOC carrier, or restricting trunk access by IP.

Base URL: `https://api.vobiz.ai/api/v1`. Auth on every request: `X-Auth-ID` and `X-Auth-Token` headers. JSON bodies use `Content-Type: application/json`.

## Resources

- **Trunks** - the parent SIP container. `POST /Account/{auth_id}/trunks`, `GET`, `PUT`, `DELETE`. Creating a trunk auto-generates a SIP domain `{trunk_id}.sip.vobiz.ai`.
- **Credentials** - username/password for SIP digest auth. `POST /Account/{auth_id}/credentials`, list at `GET /Account/{auth_id}/trunks/credentials`.
- **IP ACL** - IP/CIDR allowlist for inbound SIP traffic. `POST /Account/{auth_id}/ip-acl`, list at `GET /Account/{auth_id}/trunks/ip-acl`.
- **Origination URIs** - outbound SIP routing destinations with priority/weight failover. `POST /Account/{auth_id}/origination-uris`, list at `GET /Account/{auth_id}/trunks/origination-uris`.
- **Number assignment** - route a DID to a trunk. `POST /Account/{auth_id}/numbers/{phone_number}/assign` with body `{ "trunk_group_id": "<trunk_id>" }`.

## The four auth/routing layers

A trunk can combine four independent controls. Know which one the user needs before reaching for an endpoint:

1. **Credentials** (SIP digest) - username/password. Use for dynamic IPs, softphones, remote workers, AI agents. Works behind NAT.
2. **IP ACL** - allowlist a source IP or CIDR. Use for static-IP PBXs, SBCs, and carrier interconnects. No per-client config, faster (no auth exchange), but public-IP only.
3. **Origination URI** - **outbound** termination targets. `priority` (lower wins, tried first) selects the route; `weight` splits traffic across same-priority URIs (weight 20 gets 2x the traffic of weight 10).
4. **Webhook** - **informational only**. `CallInitiated` (with `Allowed`/`Reason`) and `Hangup` (with `Billsec`/`MOS`) notifications. Fail-open and non-blocking; a webhook response cannot accept or reject a call.

Credentials and IP ACL can be combined for defense-in-depth (require both a whitelisted IP AND valid digest auth).

## Trunk lifecycle

1. `POST /Account/{auth_id}/trunks` with `{ name, trunk_type, max_concurrent_calls }` (the three required fields) → returns `trunk_id` and auto `trunk_domain`.
2. (optional) `POST /credentials` and/or `POST /ip-acl` for inbound auth.
3. (optional) `POST /origination-uris` for outbound BYOC routing (set `priority`/`weight`).
4. `POST /Account/{auth_id}/numbers/{number}/assign` to route a DID inbound.
5. `PUT /Account/{auth_id}/trunks/{trunk_id}` to update (`name`, `max_concurrent_calls`, `enabled`).
6. `DELETE /Account/{auth_id}/trunks/{trunk_id}` to remove (or set `enabled: false` to keep config).

## End-to-end BYOC recipe

```bash
AUTH=MA_XXXXXXXX
BASE=https://api.vobiz.ai/api/v1/Account/$AUTH
H=(-H "X-Auth-ID: $AUTH" -H "X-Auth-Token: $TOKEN" -H "Content-Type: application/json")

# 1. Create the trunk (only name, trunk_type, max_concurrent_calls are required)
TRUNK=$(curl -s -X POST "$BASE/trunks" "${H[@]}" \
  -d '{"name":"Acme BYOC","trunk_type":"OUTBOUND","max_concurrent_calls":10}' \
  | jq -r .trunk_id)
# trunk_domain is now $TRUNK.sip.vobiz.ai

# 2a. Inbound auth via IP ACL (name + ip_address required; CIDR allowed)
curl -s -X POST "$BASE/ip-acl" "${H[@]}" \
  -d '{"name":"Prod SBC","ip_address":"10.20.30.0/24"}'

# 2b. ...or via digest credentials (username + password required)
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

## Webhook signature verification

Trunk webhooks send `X-Vobiz-Event` and `X-Vobiz-Request-ID` headers; verify any signature header against the raw body with HMAC-SHA256 before trusting the payload. Compare in constant time.

```python
import hmac, hashlib

def verify(raw_body: bytes, signature_header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_header or "")
```

> The exact signature header name and signing secret are not defined in `openapi.yaml`. Confirm both against the live API / Console before relying on this; treat the snippet as the verification pattern, not a verified header contract.

## Pitfalls

- **Path casing:** Trunk, credential, IP ACL, origination-URI, and number-to-trunk endpoints all use **uppercase** `Account` (`/api/v1/Account/...`). The lowercase `/api/v1/account/...` form is used **only** by the DID sub-account assignment endpoints - do not copy that casing onto trunk routes.
- **List paths differ from create paths.** You create at `/Account/{auth_id}/credentials` (and `/ip-acl`, `/origination-uris`) but list at `/Account/{auth_id}/trunks/credentials` (and `/trunks/ip-acl`, `/trunks/origination-uris`). The `trunks/` segment appears only on the list routes.
- **Create vs response field names diverge.** Origination URI create body uses `sip_uri`; the response returns it as `uri`. IP ACL and credential create bodies require `name`/`username` that are not echoed identically in every response. Read the response schema, not the request.
- **Required create fields are minimal.** Trunk: `name` + `trunk_type` (`INBOUND`/`OUTBOUND`) + `max_concurrent_calls`. IP ACL: `name` + `ip_address`. Credential: `username` + `password`. Origination URI: `name` + `sip_uri` + `priority`. Anything else (transport, secure, weight) is optional/defaulted or Console-managed.
- **IP ACL accepts CIDR**, not just single IPs (e.g. `10.20.30.0/24`). IPv6 is unsupported. The field is `ip_address` (not `cidr_ip`). The API rejects RFC 5737 documentation prefixes (`203.0.113.0/24`); use real or RFC 1918 (`10.0.0.0/24`) ranges.
- **Write-only secrets.** Credential `password` is never returned (shows as `<redacted>`). Save it at creation; `username` is immutable - rotate by creating a new credential and deleting the old one. Update only changes `password`.
- **Disable vs delete.** Prefer `enabled: false` (trunk via `PUT`, credential/IP ACL/URI via `PUT`) over `DELETE` to preserve config. `DELETE` on a trunk cascades to its credentials, IP ACLs, and origination URIs and can return **409** if calls are active.
- **Number assignment body field is `trunk_group_id`** (the trunk's UUID), not `trunk_id`. URL-encode the phone number with `%2B` for `+` or you get a 404.
- **`cps_limit` vs `concurrent_calls_limit`:** CPS caps new call attempts per second (flood protection); concurrent caps simultaneous live calls (capacity/cost). Defaults seen in responses: `cps_limit: 2`, `concurrent_calls_limit: 10`.

## When to search docs

- "How do I attach a number to a trunk?" → `trunks/assign-number`
- "What IP ranges / CIDR work?" → `trunks/ip-acl/create-ip-acl`
- "Outbound routing with priority + fallback / load balancing" → `trunks/origination-uri/create-origination-uri`
- "Credentials vs IP ACL - which auth?" → `trunks/ip-acl` (comparison table)
- "Trunk webhook events / payloads / signature" → `trunks/webhook`
