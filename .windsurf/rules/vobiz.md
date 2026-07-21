---
trigger: always_on
---

# Vobiz development rules

Use `skills/vobiz-api/SKILL.md` as the router for Vobiz work. Load only the
specialized skill needed for the requested domain.

Use these canonical sources:

- Docs: `https://vobiz.ai/docs/introduction`
- OpenAPI: `https://vobiz.ai/openapi.json`
- MCP: `https://vobiz.ai/docs/mcp`
- Agent guidance: `https://vobiz.ai/agents.md`

Preserve Vobiz endpoint path casing. Keep `X-Auth-ID` and `X-Auth-Token` in
environment variables or a secret manager, and never expose them. Encode `+`
as `%2B` in E.164 URL path parameters. Ask for explicit confirmation before a
call, message, purchase, number release, balance transfer, deletion, or other
billable or externally visible operation. Do not invent API behavior.

