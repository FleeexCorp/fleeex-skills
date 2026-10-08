# fleeex skills

Agent skills that integrate [fleeex](https://www.fleeex.dev) into your app for you: they
connect the fleeex MCP, create your app and its API keys, wire `@fleeex/sdk` into your
backend, register your webhook and test everything in sandbox.

fleeex is a prepaid AI wallet owned by your users plus an OpenAI-compatible LLM proxy:
your app calls models through fleeex, and each user's own wallet pays for their usage.

## Install

### Claude Code (plugin, recommended)

```bash
/plugin marketplace add FleeexCorp/fleeex-skills
/plugin install fleeex@fleeex
```

The plugin also declares the fleeex MCP server (`https://api.fleeex.dev/mcp`). Run `/mcp`,
pick `fleeex`, and sign in once in the browser.

### Other agents (Cursor, Codex, Copilot, ...)

```bash
npx skills add FleeexCorp/fleeex-skills
```

Then connect the MCP server to your client: see
[`skills/fleeex-connect-mcp`](skills/fleeex-connect-mcp/SKILL.md).

### Manually

Copy the folders under [`skills/`](skills/) into your agent's skills directory (for
Claude Code: `.claude/skills/` or `~/.claude/skills/`).

## Use

Ask your agent:

> Integrate fleeex into this app.

| Skill | Does |
|---|---|
| [`fleeex-setup`](skills/fleeex-setup/SKILL.md) | The whole integration, end to end, using the skills below. |
| [`fleeex-connect-mcp`](skills/fleeex-connect-mcp/SKILL.md) | Connects and authenticates the fleeex MCP server. |
| [`fleeex-api-keys`](skills/fleeex-api-keys/SKILL.md) | Creates the app, live and sandbox keys, stores them in your gitignored env file; rotation and revocation. |
| [`fleeex-sdk`](skills/fleeex-sdk/SKILL.md) | Routes your LLM calls through fleeex (`@fleeex/sdk`, OpenAI SDK, Vercel AI SDK, LangChain, Python), handles `402` top-ups and the connect-wallet gate. |
| [`fleeex-webhooks`](skills/fleeex-webhooks/SKILL.md) | Builds a verified, idempotent receiver and registers the subscription. |
| [`fleeex-sandbox`](skills/fleeex-sandbox/SKILL.md) | Tests for free and forces the `402` paths. |

## Guarantees

- API keys and webhook secrets go straight into a gitignored env file, never into chat,
  client code or commits.
- Destructive actions (revoking a key, rotating a webhook secret) always ask you first.
- No skill and no MCP tool moves money.

## Links

- Docs: https://docs.fleeex.dev
- Dashboard: https://app.fleeex.dev
- SDK: [`@fleeex/sdk`](https://www.npmjs.com/package/@fleeex/sdk)
- Support: support@fleeex.dev

## License

MIT
