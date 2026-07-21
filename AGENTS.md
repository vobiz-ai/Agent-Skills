# Vobiz agent instructions

This repository publishes reusable agent skills for the Vobiz programmable
telephony platform. Use the instructions below when you answer questions or
change files in this repository.

## Use the maintained skills

Start with `skills/vobiz-api/SKILL.md`, then load only the specialized skill
that matches the task. Do not duplicate full API guidance in platform-specific
configuration files.

Available domains include phone numbers, voice calls, VobizXML, audio streams,
SIP trunking, applications and endpoints, CDRs, sub-accounts, the Partner API,
WhatsApp, AI voice agents, and Plivo migration.

## Use canonical developer resources

- Documentation: `https://vobiz.ai/docs/introduction`
- OpenAPI: `https://vobiz.ai/openapi.json`
- Documentation MCP server: `https://vobiz.ai/docs/mcp`
- Agent Skills: `https://github.com/vobiz-ai/Agent-Skills`
- Agent discovery index: `https://vobiz.ai/agents.md`
- LLM discovery index: `https://vobiz.ai/llms.txt`

Fetch current documentation when an endpoint, request field, response, price,
limit, or product behavior may have changed. Never invent an API path or field.

## Follow Vobiz conventions

- Use the REST base URL `https://api.vobiz.ai/api/v1`.
- Authenticate server-side requests with `X-Auth-ID` and `X-Auth-Token` unless
  the referenced endpoint explicitly documents another supported method.
- Read credentials from environment variables or a secret manager.
- Never print, persist, commit, or place credentials in examples.
- Preserve API path casing exactly. `/Account/`, `/account/`, and `/accounts/`
  are not interchangeable.
- URL-encode E.164 numbers used in URL paths. Encode `+` as `%2B`.
- Keep the literal `+` in JSON request bodies.

## Protect users from unintended actions

Show the target account, resource, and expected effect before a write. Require
explicit confirmation before purchasing or releasing numbers, placing calls,
sending messages, transferring balance, deleting resources, or performing
another action that can spend money or affect an external recipient.

Do not retry a timed-out money-moving or externally visible request until you
have checked the resulting resource state or transaction history.

## Repository standards

- Keep each `SKILL.md` focused on one domain.
- Use active voice and concise instructions.
- Link to canonical Vobiz documentation instead of copying long API schemas.
- Keep examples free of production identifiers, phone numbers, and secrets.
- Update `README.md` when adding, renaming, or removing a skill.
- Run `npx --yes skills add . --list` before opening a pull request.

