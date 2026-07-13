# Vobiz agent skills

Reusable skills that teach coding agents how to build with the Vobiz REST API, Voice API, VobizXML, SIP trunking, phone numbers, partner accounts, sub-accounts, WhatsApp, and AI voice integrations.

## Install

List the available skills:

```bash
npx skills add vobiz-ai/Agent-Skills --list
```

Install all Vobiz skills:

```bash
npx skills add vobiz-ai/Agent-Skills --all
```

Install the core API skills:

```bash
npx skills add vobiz-ai/Agent-Skills \
  --skill vobiz-api \
  --skill vobiz-sip-trunking \
  --skill vobiz-voice-calls \
  --skill vobiz-partner-api \
  --skill vobiz-sub-accounts
```

Install globally for a specific agent:

```bash
npx skills add vobiz-ai/Agent-Skills \
  --skill vobiz-api \
  --agent codex \
  --global
```

## Available skills

| Skill | Scope |
| --- | --- |
| `vobiz-api` | Authentication, API conventions, error handling, and routing to specialized skills |
| `vobiz-phone-numbers` | Number discovery, purchase, assignment, and release |
| `vobiz-voice-calls` | Outbound and live-call control |
| `vobiz-voice-xml` | VobizXML call flows |
| `vobiz-audio-streams` | Bidirectional WebSocket audio |
| `vobiz-sip-trunking` | Trunks, credentials, IP ACLs, and origination URIs |
| `vobiz-applications-endpoints` | Voice applications and SIP endpoints |
| `vobiz-cdr` | Call Detail Records and quality data |
| `vobiz-sub-accounts` | Internal tenant isolation and sub-account KYC |
| `vobiz-partner-api` | Reseller customers, wallet transfers, KYC, and reporting |
| `vobiz-whatsapp` | WhatsApp messaging, templates, channels, and webhooks |
| `vobiz-ai-voice-agents` | Vapi, Retell, ElevenLabs, LiveKit, Pipecat, and related integrations |
| `vobiz-plivo-migration` | Plivo-to-Vobiz migration |

Skills provide instructions, not credentials. Keep `X-Auth-ID` and `X-Auth-Token` in environment variables or a secret manager.

For current API documentation, use [docs.vobiz.ai](https://docs.vobiz.ai) and the hosted MCP server at `https://docs.vobiz.ai/mcp`.
