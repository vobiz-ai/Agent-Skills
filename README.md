# Vobiz agent skills

Reusable skills that teach coding agents how to build with the Vobiz REST API, Voice API, VobizXML, SIP trunking, phone numbers, partner accounts, sub-accounts, WhatsApp, and AI voice integrations.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Skills](https://img.shields.io/badge/skills-17-E83C00.svg)](./skills)
[![Docs](https://img.shields.io/badge/docs-vobiz.ai-0b7285.svg)](https://www.vobiz.ai/docs/introduction)

## Overview

Coding agents are good at writing code and poor at guessing API contracts. Ask a
general-purpose agent to "place a Vobiz call" and it will usually produce
something that looks right — a plausible base URL, a plausible auth header, a
plausible XML verb — and fails on the first request. Telephony makes that worse
than usual, because the failure modes are expensive: a wrong number format
buys the wrong DID, a fire-and-forget retry places a second billable call, and
a leaked auth token is a live account.

This repository closes that gap. It packages the Vobiz platform as a set of
seventeen focused skills — short, verified instruction files that an agent loads
on demand. Each skill covers one domain, states the exact base URL, path
casing, auth headers, request fields, and error behaviour for that domain, and
tells the agent when to hand off to a different skill. A router skill
(`vobiz-api`) sits in front and picks the right one, so the agent loads what the
task needs instead of the whole platform.

The skills are consumed as an agent plugin rather than as documentation you read
yourself. Install them into Codex or Claude Code and the plugin also wires up the
public Vobiz Docs MCP server, so an agent can search current docs
instead of relying on whatever it memorised. Alongside the skills, the repository
carries repository-scoped instruction files for Cursor, Windsurf, and GitHub
Copilot, so the same conventions and safeguards apply whichever agent a developer
happens to be running.

It is aimed at developers building on Vobiz with an AI assistant in the loop, and
at platform teams who want their agents to follow one consistent set of Vobiz
conventions across a codebase. What you get at the end is an agent that reaches
for the correct endpoint the first time, keeps `X-Auth-ID` and `X-Auth-Token` out
of the code it writes, and asks before it does anything billable or externally
visible.

## What you can build with it

- **Outbound voice integrations** — an agent scaffolds a call flow that creates a
  call, serves VobizXML from an `answer_url`, and handles the hangup callback.
- **AI voice agents on real phone numbers** — wire Vapi, Retell, ElevenLabs,
  LiveKit, Pipecat, Bolna, Ultravox, OpenAI Realtime, or Deepgram to a Vobiz
  number, over a dashboard answer URL, a vendor API, or a self-hosted runtime.
- **BYOC and PBX connectivity** — provision trunks, SIP credentials, IP ACLs, and
  origination URIs, then route a DID to the trunk.
- **IVR, voicemail, queues, and conferences** — build declarative call flows in
  VobizXML with the correct verb and attribute names.
- **Real-time audio pipelines** — bidirectional WebSocket streaming for custom
  STT/TTS, live transcription, or barge-in handling.
- **WhatsApp messaging and support flows** — templates, campaigns, inbound
  webhooks, and multi-agent inbox work through the messaging API.
- **Reseller and multi-tenant platforms** — partner customer accounts, wallet
  transfers, KYC sessions, and per-customer CDR reporting.
- **Plivo migrations** — a guided, minimal-diff port of an existing Plivo voice
  app, including the XML verb rename and the webhook signature header change.

## What's included

Seventeen skills live under [`skills/`](./skills), one directory per domain, each
containing a single `SKILL.md` with YAML frontmatter (`name`, `description`) that
agent runtimes use for discovery.

| Skill | Scope | What it covers |
| --- | --- | --- |
| [`vobiz-api`](./skills/vobiz-api/SKILL.md) | Router and conventions | Base URL `https://api.vobiz.ai/api/v1`, `X-Auth-ID`/`X-Auth-Token` auth, path casing, request construction, error handling, and the routing table to every other skill |
| [`vobiz-phone-numbers`](./skills/vobiz-phone-numbers/SKILL.md) | Numbers (DIDs) | Inventory search, purchase from inventory, listing owned numbers, trunk assignment and unassignment, release; `%2B` path encoding for E.164 |
| [`vobiz-voice-calls`](./skills/vobiz-voice-calls/SKILL.md) | Call control | Create outbound calls, list and fetch live calls, hangup, mid-call actions (play, speak, DTMF, record, stream), and answering-machine detection |
| [`vobiz-voice-xml`](./skills/vobiz-voice-xml/SKILL.md) | VobizXML | How Vobiz fetches XML from `answer_url`, and the core verbs — play, speak, gather, dial, record, stream, conference, redirect, hangup |
| [`vobiz-audio-streams`](./skills/vobiz-audio-streams/SKILL.md) | Live audio | `<Stream bidirectional="true">` setup, the `start`/`media`/`playAudio`/`checkpoint`/`clearAudio`/`stop` event contract, frame timing, and close semantics |
| [`vobiz-sip-trunking`](./skills/vobiz-sip-trunking/SKILL.md) | SIP infrastructure | Trunks and their auto-generated `{trunk_id}.sip.vobiz.ai` domain, SIP credentials, IP ACLs, origination URIs with priority/weight failover, and the four auth and routing layers |
| [`vobiz-applications-endpoints`](./skills/vobiz-applications-endpoints/SKILL.md) | Applications and endpoints | Applications as webhook containers (`answer_url`, `hangup_url`) versus SIP endpoints for softphones, browser clients, and desk phones — and which one a given scenario needs |
| [`vobiz-cdr`](./skills/vobiz-cdr/SKILL.md) | Call records | List, search, recent, CSV export, and get-by-`call_id`; 43 CDR fields including MOS, jitter, packet loss, and hangup attribution |
| [`vobiz-reports`](./skills/vobiz-reports/SKILL.md) | Reports | Dated Markdown reports from account data: connected, not-connected and error calls, answer and connect rates by hour, number, destination or campaign, number performance and spam flags, spend and balance runway, campaign and messaging outcomes |
| [`vobiz-campaign-planner`](./skills/vobiz-campaign-planner/SKILL.md) | Outbound campaigns | Plans a campaign end to end from the reports: sizing against concurrency and balance, caller-ID selection and whether to buy numbers, calling window and retries, cost estimate, list hygiene, build through the Campaign Manager API behind confirmation gates, monitoring and close-out |
| [`vobiz-sub-accounts`](./skills/vobiz-sub-accounts/SKILL.md) | Internal segmentation | Sub-accounts for dev/staging/prod or per-team isolation, `personal_use` versus `customer_use` KYC, and how they differ from partner customer accounts |
| [`vobiz-partner-api`](./skills/vobiz-partner-api/SKILL.md) | Reseller workflows | Partner profile and dashboard, customer account creation, balance transfers, PAN + Aadhaar OTP KYC sessions, and per-customer CDRs, transactions, and numbers |
| [`vobiz-whatsapp`](./skills/vobiz-whatsapp/SKILL.md) | WhatsApp Business | Sending text, template, and media messages, template management per channel, the multi-agent inbox, signed inbound webhooks, and the three channel onboarding modes |
| [`vobiz-sms`](./skills/vobiz-sms/SKILL.md) | SMS, MMS and RCS | Single and multi-recipient sends, cost preview, idempotency, scheduling, delivery status, two-way conversations, opt-outs, webhooks, broadcasts, RCS senders, and US 10DLC / India DLT compliance |
| [`vobiz-ai-voice-agents`](./skills/vobiz-ai-voice-agents/SKILL.md) | Voice AI integrations | Three deployment models — no-code dashboard, vendor API, and self-hosted runtime — across Vapi, Retell AI, ElevenLabs, LiveKit, Pipecat, Bolna, Ultravox, OpenAI Realtime, Deepgram, and WebRTC |
| [`vobiz-plivo-migration`](./skills/vobiz-plivo-migration/SKILL.md) | Migration | The minimal-diff port from Plivo: client and credential swap, `<GetDigits>`/`<GetInput>` → `<Gather>`, `X-Vobiz-Signature-V2/V3`, and re-pointing `answer_url`/`hangup_url` |
| [`vobiz-forward-deployed-engineer`](./skills/vobiz-forward-deployed-engineer/SKILL.md) | Solution scoping | Interviews the user one question at a time, resolves direction, who talks, AI or human, India compliance, scale, tenancy and channel, then recommends the Vobiz integration path with the exact pages and skills to build it |

Skills provide instructions, not credentials. Keep `X-Auth-ID` and
`X-Auth-Token` in environment variables or a secret manager.

## How it works

```text
                      your prompt
                           │
                           ▼
                  ┌─────────────────┐
                  │    vobiz-api    │  router skill: auth, base URL,
                  │  (always first) │  path casing, error handling
                  └────────┬────────┘
                           │ routes by task
     ┌──────────┬──────────┼──────────┬──────────┬──────────┐
     ▼          ▼          ▼          ▼          ▼          ▼
  numbers    voice-     voice-xml   audio-      sip-      whatsapp
             calls                  streams    trunking      ...
     │          │          │          │          │          │
     └──────────┴──────────┴─────┬────┴──────────┴──────────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │   vobiz-docs MCP server    │  live documentation
                    │  https://www.vobiz.ai/docs/mcp │  search
                    └────────────────────────────┘
                                 │
                                 ▼
                        generated integration
```

1. Your agent runtime discovers the skills from `skills/*/SKILL.md`, reading the
   `name` and `description` frontmatter to decide what is available.
2. When a prompt mentions Vobiz, `vobiz-api` loads first. It establishes the
   base URL, the auth headers, and the non-negotiable conventions.
3. Its routing table sends the agent to exactly one domain skill. Only that
   skill's instructions enter the context, which keeps the guidance specific and
   the context small.
4. If the task touches an endpoint, field, price, or limit that may have moved,
   the agent queries the `vobiz-docs` MCP server declared in
   [`.mcp.json`](./.mcp.json) rather than answering from memory.
5. Before anything billable or externally visible — buying or releasing a number,
   placing a call, sending a message, transferring balance, deleting a resource —
   the agent is instructed to show the target account, resource, and expected
   effect, and to wait for your explicit confirmation.

## Repository layout

| Path | Responsibility |
| --- | --- |
| [`skills/`](./skills) | The seventeen `SKILL.md` files. This is the content everything else points at |
| [`AGENTS.md`](./AGENTS.md) | Repository-scoped instructions for Codex and compatible agents; also the contributor standard for this repo |
| [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) | Claude Code plugin manifest — name, version, author, and the MCP server reference |
| [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json) | Declares the `vobiz-plugins` marketplace containing the `vobiz` plugin |
| [`.codex-plugin/plugin.json`](./.codex-plugin/plugin.json) | Codex plugin manifest, including the `skills`/`mcpServers` paths and store presentation metadata |
| [`.agents/plugins/marketplace.json`](./.agents/plugins/marketplace.json) | Codex marketplace entry targeting the flat repository root |
| [`.mcp.json`](./.mcp.json) | Declares the `vobiz-docs` Streamable HTTP MCP server both plugins load |
| [`server.json`](./server.json) | MCP Registry publication manifest for the public documentation server |
| [`.claude/rules/vobiz.md`](./.claude/rules/vobiz.md) | Claude Code repository rules |
| [`.cursor/rules/vobiz.mdc`](./.cursor/rules/vobiz.mdc) | Cursor repository rules |
| [`.windsurf/rules/vobiz.md`](./.windsurf/rules/vobiz.md) | Windsurf repository rules |
| [`.github/copilot-instructions.md`](./.github/copilot-instructions.md) | GitHub Copilot repository instructions |
| [`.github/workflows/validate.yml`](./.github/workflows/validate.yml) | CI: Skills CLI discovery, JSON manifest parsing, skill count, PNG asset signatures, and manifest path assertions |
| [`assets/`](./assets) | `icon.png` and `logo.png` used by the Codex plugin interface |

## Prerequisites

| Requirement | Notes |
| --- | --- |
| A coding agent | Codex, Claude Code, Cursor, Windsurf, GitHub Copilot, or any agent the Skills CLI supports |
| Node.js 22+ | Needed for `npx skills` and matching the version CI runs |
| A Vobiz account | [Sign up](https://www.vobiz.ai) and collect your `X-Auth-ID` and `X-Auth-Token` from [console.vobiz.ai](https://console.vobiz.ai) |
| Network access to `www.vobiz.ai` | The Vobiz Docs MCP server is a public Streamable HTTP endpoint and needs no credentials |

Installing the skills does not require Vobiz credentials. You only need them when
the agent starts making real API calls on your behalf.

## How to use these skills

### Codex plugin

Add the Vobiz marketplace and install the plugin:

```bash
codex plugin marketplace add vobiz-ai/Agent-Skills --ref main
codex plugin add vobiz@vobiz-plugins
```

Start a new Codex session after installation. The plugin loads all 17 Vobiz
skills and connects to the public Vobiz Docs MCP server.

To verify the marketplace and installation:

```bash
codex plugin marketplace list
codex plugin list
```

### Claude plugin

Add the official Vobiz marketplace and install the plugin in Claude Code:

```bash
claude plugin marketplace add vobiz-ai/Agent-Skills
claude plugin install vobiz@vobiz-plugins
```

The plugin includes all Vobiz skills in this repository and connects Claude to
the public Vobiz Docs MCP server for current documentation search.
After installation, run `/reload-plugins` in Claude Code.

Verify the plugin manifests before contributing changes:

```bash
claude plugin validate . --strict
```

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

### Driving the skills

Ask your agent for the Vobiz workflow you need. For example:

```text
Use Vobiz to build an outbound voice call integration.
Use Vobiz to create a SIP trunk and connect an AI voice agent.
Search the current Vobiz docs for the VobizXML Gather parameters.
```

The core `vobiz-api` skill routes tasks to the relevant domain skill. The
plugin's public documentation MCP connection keeps documentation searches
current. Any billable or externally visible action still requires your
explicit confirmation.

Naming Vobiz explicitly in the prompt is the most reliable way to trigger skill
selection. If the agent starts guessing endpoints, say "use the Vobiz skills"
and it will reload the router.

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

## Developer resources

| Resource | URL |
| --- | --- |
| API documentation | [vobiz.ai/docs/introduction](https://www.vobiz.ai/docs/introduction) |
| OpenAPI specification | [vobiz.ai/openapi.json](https://www.vobiz.ai/openapi.json) |
| Vobiz Docs MCP server | [www.vobiz.ai/docs/mcp](https://www.vobiz.ai/docs/mcp) |
| Developer Console | [console.vobiz.ai](https://console.vobiz.ai) |
| LLM discovery index | [vobiz.ai/llms.txt](https://www.vobiz.ai/llms.txt) |
| Agent orientation | [vobiz.ai/agents.md](https://www.vobiz.ai/agents.md) |
| Integration guides | [vobiz.ai/docs/integrations](https://www.vobiz.ai/docs/integrations) |

## MCP Registry

The repository includes [`server.json`](./server.json) for publishing the
public Vobiz documentation server to the official MCP Registry. The server uses
Streamable HTTP and does not require credentials:

```text
https://www.vobiz.ai/docs/mcp
```

Registry name:

```text
io.github.vobiz-ai/vobiz-docs
```

## Contributing a skill

A skill is a directory under `skills/` containing exactly one `SKILL.md`. The
directory name is the skill name and must match the `name` in the frontmatter.

1. **Create the directory and file.**

   ```bash
   mkdir -p skills/vobiz-<domain>
   $EDITOR skills/vobiz-<domain>/SKILL.md
   ```

2. **Write the frontmatter.** Two fields, both required. The `description` is
   what an agent runtime reads to decide whether to load the skill, so make it
   specific and list the operations it covers.

   ```markdown
   ---
   name: vobiz-<domain>
   description: One sentence stating what this skill does and when to use it, naming the concrete operations it covers.
   ---

   # Vobiz <Domain> skill

   Use this when the user wants to ...
   ```

3. **Follow the repository standards** in [`AGENTS.md`](./AGENTS.md). In short:
   one domain per skill, active voice, exact base URLs and path casing, link to
   canonical documentation rather than pasting long schemas, and no production
   identifiers, phone numbers, or secrets in examples.

4. **Verify everything you assert.** Every endpoint, field, header, and XML
   attribute must be checked against the live documentation or the OpenAPI
   specification before it lands. A confident wrong instruction is worse than
   no instruction, because the agent will act on it.

5. **Register it.** Add a row to the *What's included* table above, and update
   the skill count in [`.github/workflows/validate.yml`](./.github/workflows/validate.yml)
   — CI asserts the exact number of skill directories and will fail otherwise.

6. **Run the checks locally.**

   ```bash
   npx --yes skills add . --list
   claude plugin validate . --strict
   ```

   The first command is what CI runs to confirm discovery. The second validates
   the plugin manifests.

7. **Open a pull request** describing the domain the skill covers and where you
   verified its contents.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Skills do not appear after `codex plugin add` | The session was started before installation | Start a new Codex session, then confirm with `codex plugin list` |
| Skills do not appear in Claude Code | The plugin is installed but not loaded into the running session | Run `/reload-plugins`, or restart Claude Code |
| `npx skills add . --list` returns nothing | Run from outside the repository root, or Node is older than 22 | Run from the repository root and use Node.js 22+, which is what CI uses |
| CI fails with `Expected 17 skills, found 18` | A skill directory was added without updating the count assertion | Update the count in `.github/workflows/validate.yml` in the same pull request |
| CI fails on `assets/icon.png is not a PNG` | An asset was replaced with a different format or corrupted by a transfer | Restore a real PNG; CI checks the `89504e470d0a1a0a` file signature |
| `claude plugin validate . --strict` fails | Malformed JSON or a bad path in one of the manifests | Parse each file in `.claude-plugin/`, `.codex-plugin/`, `.mcp.json`, and `server.json`; CI makes the same checks |
| Agent-generated requests return `401` | `X-Auth-ID`/`X-Auth-Token` are missing from the environment | Export both from [console.vobiz.ai](https://console.vobiz.ai); skills never carry credentials |
| Agent-generated requests return `404` on a valid-looking path | Path casing was changed, or an E.164 number was left unencoded in the path | `/Account/`, `/account/`, and `/accounts/` are not interchangeable; encode `+` as `%2B` in URL paths and keep the literal `+` in JSON bodies |
| Partner endpoints return `404` | The request used `/partner/v1/` | Partner routes live under `https://api.vobiz.ai/api/v1/partner/*` |
| Agent invents endpoints instead of loading a skill | The prompt did not name Vobiz, so the router skill never loaded | Say "use the Vobiz skills" and name the domain, e.g. "use Vobiz SIP trunking" |
| Documentation search returns nothing | The MCP server is not reachable from the agent's environment | Confirm outbound access to `https://www.vobiz.ai/docs/mcp`; it is a public Streamable HTTP endpoint needing no credentials |

## Security notes

- **Skills contain instructions, never credentials.** Nothing in this repository
  holds a Vobiz token, and no skill asks an agent to embed one in generated code.
  `X-Auth-ID` and `X-Auth-Token` belong in environment variables or a secret
  manager, and the agent instructions say so explicitly.
- **Confirmation before consequences.** Every skill inherits the safeguard in
  [`AGENTS.md`](./AGENTS.md): show the target account, resource, and expected
  effect before a write, and require explicit confirmation before purchasing or
  releasing numbers, placing calls, sending messages, transferring balance, or
  deleting resources.
- **No blind retries on money-moving requests.** If a billable or externally
  visible request times out, the instructions require checking the resulting
  resource or transaction state before retrying, so a timeout does not become a
  duplicate charge or a second call to a real person.
- **Public MCP server only.** The `vobiz-docs` server declared in `.mcp.json`
  serves public documentation over Streamable HTTP and takes no credentials.
  Installing the plugin does not grant an agent access to your account.
- **Examples stay clean.** Repository standards forbid production identifiers,
  real phone numbers, and secrets in skill examples. Please hold contributions to
  the same line.
- Found a security issue? Report it privately to
  [piyush@vobiz.ai](mailto:piyush@vobiz.ai) rather than opening a public issue.

## Roadmap

> Planned improvements to this repository. Ideas and pull requests are welcome —
> open an issue to discuss anything here.

- [ ] Replace the hard-coded skill count in `validate.yml` with a manifest-driven
      check, so adding a skill does not require editing CI.
- [ ] Add frontmatter linting to CI — assert that every `SKILL.md` has `name`
      matching its directory and a non-empty `description`.
- [ ] Add a link checker that verifies every documentation URL referenced by a
      skill still resolves.
- [ ] Build an evaluation harness of routing prompts, so changes to `vobiz-api`
      can be regression-tested against the domain skill they should select.
- [ ] Publish `server.json` to the official MCP Registry and keep the version in
      step with the plugin manifests.
- [ ] Introduce versioned releases and a changelog, so installations can pin a
      known-good set of skills instead of tracking `main`.
- [ ] Broaden skill coverage as new Vobiz API surfaces ship, and add worked
      end-to-end recipes that chain several skills together.

## Contributing

Issues and pull requests are welcome. Bug reports, corrections to an endpoint or
field, and new skills are all useful — corrections most of all, since an
inaccurate instruction propagates into every integration an agent writes from it.

Before opening a pull request:

```bash
npx --yes skills add . --list      # skill discovery, same as CI
claude plugin validate . --strict  # plugin manifest validation
```

Read [`AGENTS.md`](./AGENTS.md) for the repository standards, and see
[Contributing a skill](#contributing-a-skill) above if you are adding a new
domain. Please keep credentials, production identifiers, and real phone numbers
out of every example.

## License

Released under the [MIT License](./LICENSE) © Vobiz.

MIT is permissive: you may use, modify, and redistribute this code, including in
closed-source commercial products, provided the copyright notice and licence text
are retained. There is no warranty. If your organisation needs a different
licensing arrangement, contact [piyush@vobiz.ai](mailto:piyush@vobiz.ai).

## Built by Team Vobiz

[Vobiz](https://www.vobiz.ai) is a programmable voice and SIP-trunking platform for
voice APIs, SIP trunking, and AI voice agents. This repository is built and
maintained by the Vobiz team.

**Maintainer:** Piyush Sahoo — [piyush@vobiz.ai](mailto:piyush@vobiz.ai) · [LinkedIn](https://www.linkedin.com/in/piyush-s713/)

Questions, or want to talk through an integration? Open an issue on this repo,
or reach out directly at [piyush@vobiz.ai](mailto:piyush@vobiz.ai).

**Useful links:** [Docs](https://docs.vobiz.ai) · [API reference](https://docs.vobiz.ai/api-reference) · [Sign up](https://www.vobiz.ai)
