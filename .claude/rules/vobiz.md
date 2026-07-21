# Vobiz development rules

Use the maintained instructions under `skills/` for every Vobiz task. Start
with `skills/vobiz-api/SKILL.md`, then load only the matching domain skill.

- Verify changing behavior against `https://vobiz.ai/docs/introduction` or the
  MCP server at `https://vobiz.ai/docs/mcp`.
- Use `https://api.vobiz.ai/api/v1` as the REST base URL.
- Preserve endpoint path casing exactly.
- Keep `X-Auth-ID` and `X-Auth-Token` in environment variables or a secret
  manager. Never reveal or commit them.
- Encode `+` as `%2B` when an E.164 number appears in a URL path.
- Ask for explicit confirmation before calls, messages, purchases, releases,
  balance transfers, deletion, or another billable or externally visible
  action.
- Do not invent endpoints, request fields, response fields, prices, or limits.

Canonical resources:

- `https://vobiz.ai/openapi.json`
- `https://vobiz.ai/agents.md`
- `https://github.com/vobiz-ai/Agent-Skills`

