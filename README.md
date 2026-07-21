# Vobiz agent skills

Reusable skills that teach coding agents how to build with the Vobiz REST API, Voice API, VobizXML, SIP trunking, phone numbers, partner accounts, sub-accounts, WhatsApp, and AI voice integrations.

## Install

### Claude plugin

Add the official Vobiz marketplace and install the plugin in Claude Code:

```bash
claude plugin marketplace add vobiz-ai/Agent-Skills
claude plugin install vobiz@vobiz-plugins
```

The plugin includes all Vobiz skills in this repository and connects Claude to
the public Vobiz documentation MCP server for current documentation search.
After installation, run `/reload-plugins` in Claude Code.

You can also add `https://github.com/vobiz-ai/Agent-Skills` through your Claude
organization's **Settings → Plugins → Add plugins** control.

### Other coding agents

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

## Agent configuration

This repository includes public, repository-scoped instructions for major
coding agents:

| Agent | Configuration |
| --- | --- |
| Codex and compatible agents | [`AGENTS.md`](./AGENTS.md) |
| Claude Code | [`.claude/rules/vobiz.md`](./.claude/rules/vobiz.md) |
| Cursor | [`.cursor/rules/vobiz.mdc`](./.cursor/rules/vobiz.mdc) |
| Windsurf | [`.windsurf/rules/vobiz.md`](./.windsurf/rules/vobiz.md) |
| GitHub Copilot | [`.github/copilot-instructions.md`](./.github/copilot-instructions.md) |

These files route agents to the maintained `skills/` content, canonical Vobiz
documentation, OpenAPI definition, and MCP server. They also apply credential
and confirmation safeguards to billable or externally visible operations.

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

## Developer resources

| Resource | URL |
| --- | --- |
| API documentation | [vobiz.ai/docs/introduction](https://vobiz.ai/docs/introduction) |
| OpenAPI specification | [vobiz.ai/openapi.json](https://vobiz.ai/openapi.json) |
| Documentation MCP server | [vobiz.ai/docs/mcp](https://vobiz.ai/docs/mcp) |
| Developer Console | [console.vobiz.ai](https://console.vobiz.ai) |
| LLM discovery index | [vobiz.ai/llms.txt](https://vobiz.ai/llms.txt) |
| Agent orientation | [vobiz.ai/agents.md](https://vobiz.ai/agents.md) |
| Integration guides | [vobiz.ai/docs/integrations](https://vobiz.ai/docs/integrations) |
