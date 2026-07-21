# Vobiz repository instructions

Use `skills/vobiz-api/SKILL.md` to select the correct Vobiz domain skill. Treat
the skill files and current Vobiz documentation as the source of truth.

- Docs: `https://vobiz.ai/docs/introduction`
- OpenAPI: `https://vobiz.ai/openapi.json`
- MCP: `https://vobiz.ai/docs/mcp`
- REST base URL: `https://api.vobiz.ai/api/v1`

Preserve API path casing and URL-encode E.164 path parameters. Keep credentials
out of source code and examples. Never invent endpoints or fields. Require
explicit confirmation before generating or running code that places calls,
sends messages, purchases or releases numbers, transfers balance, deletes
resources, or produces another billable or externally visible effect.

