---
name: vobiz-api
description: Build with the Vobiz REST API and select the correct domain workflow. Use for authentication, API path casing, request construction, errors, phone numbers, voice calls, SIP trunks, applications, endpoints, CDRs, partner accounts, sub-accounts, WhatsApp, VobizXML, AI voice agents, and solution design.
license: MIT
compatibility: Vobiz REST API v1, X-Auth-ID + X-Auth-Token header auth.
metadata:
  author: vobiz
  version: "1.1"
---

# Vobiz API

Use the Vobiz API at `https://api.vobiz.ai/api/v1`. Authenticate server-side requests with `X-Auth-ID` and `X-Auth-Token`. Read credentials from environment variables or a secret manager. Never print, persist, or commit tokens.

## Route the task

Load only the specialized skill that matches the requested operation:

| Task | Skill |
| --- | --- |
| Decide which Vobiz path fits a use case before building | `vobiz-forward-deployed-engineer` |
| Purchase, assign, or release a phone number | `vobiz-phone-numbers` |
| Create or control calls | `vobiz-voice-calls` |
| Build IVR and call-control XML | `vobiz-voice-xml` |
| Stream call audio over WebSocket | `vobiz-audio-streams` |
| Manage SIP trunks, credentials, IP ACLs, or routing | `vobiz-sip-trunking` |
| Configure voice applications or SIP endpoints | `vobiz-applications-endpoints` |
| Query completed call records | `vobiz-cdr` |
| Manage internal sub-accounts and their KYC | `vobiz-sub-accounts` |
| Operate a reseller partner account | `vobiz-partner-api` |
| Send SMS, MMS or RCS, run broadcasts, or manage opt-outs | `vobiz-sms` |
| Send WhatsApp messages or manage channels | `vobiz-whatsapp` |
| Connect an AI voice platform | `vobiz-ai-voice-agents` |
| Migrate a Plivo integration | `vobiz-plivo-migration` |

When the request names a business outcome ("remind patients", "put our agent on a number", "let resellers onboard clients") rather than an endpoint, start with `vobiz-forward-deployed-engineer`. It interviews for direction, who talks, geography, scale, and tenancy, then hands off to the domain skills above.

## Apply common rules

1. Fetch current endpoint details from `https://www.vobiz.ai/docs` or the Vobiz Docs MCP server at `https://www.vobiz.ai/docs/mcp` when a request shape is uncertain.
2. Preserve API path casing exactly. Vobiz uses `/Account/`, `/account/`, and `/accounts/` for different resources.
3. URL-encode E.164 numbers used in paths (`+` becomes `%2B`). Keep the literal `+` in JSON bodies.
4. Treat `401` as an authentication or path-casing problem, `429` as a rate or capacity limit, and `5xx` as an unknown outcome that requires state verification before retrying.
5. Show the target account, resource, and financial effect before a write operation.
6. Require explicit confirmation before purchases, balance transfers, call placement, number release, deletion, or other irreversible operations.
7. Do not retry a timed-out money-moving request until the resulting state or transaction history has been checked.
8. Never write Vobiz IP addresses inline; link to `https://www.vobiz.ai/docs/concepts/ip-whitelisting`.

## Prefer documentation-backed answers

Use endpoint pages with HTTP method badges for request and response contracts. Use overview pages for concepts. If the OpenAPI definition and a narrative example disagree, describe the discrepancy and ask the user before performing a write.
