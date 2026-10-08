---
name: fleeex-connect-mcp
description: Connect an AI coding agent (Claude Code, Cursor, VS Code, any MCP client) to the remote fleeex MCP server at https://api.fleeex.dev/mcp and authenticate it, so the agent can create fleeex apps, API keys and webhooks. Use when fleeex MCP tools (list_apps, create_app, create_api_key, set_webhook...) are missing, return UNAUTHORIZED, or the user says "connect the fleeex MCP", "add fleeex to Claude Code", "fleeex mcp login".
---

# Connect the fleeex MCP

The fleeex MCP server lets the **account owner** manage their own fleeex account: apps,
API keys, webhooks, usage, wallet (read only). It speaks Streamable HTTP and OAuth 2.1;
sign-in and consent happen in the fleeex dashboard. No tool moves money.

## 1. Already connected?

Look for the fleeex tools (`list_apps`, `create_app`, ...). If present, call `list_apps`:

- it answers: done;
- `UNAUTHORIZED`, or the client reports the server needs authentication: go to step 3.

## 2. Register the server

**Claude Code with the fleeex plugin**: the plugin already declares the `fleeex` server.
Go to step 3.

**Claude Code without the plugin**:

```bash
claude mcp add --transport http fleeex https://api.fleeex.dev/mcp
```

Add `--scope user` to make it available in every project.

**Cursor** (`~/.cursor/mcp.json` or `.cursor/mcp.json`):

```json
{ "mcpServers": { "fleeex": { "url": "https://api.fleeex.dev/mcp" } } }
```

**VS Code** (`.vscode/mcp.json`):

```json
{ "servers": { "fleeex": { "type": "http", "url": "https://api.fleeex.dev/mcp" } } }
```

**Any other client** works if it supports the Streamable HTTP transport, OAuth discovery
with dynamic client registration, and a loopback redirect (`http://localhost`,
`http://127.0.0.1` or `http://[::1]`, any port). Any other redirect host is refused.

## 3. Authenticate (the user does this)

You cannot complete this step: it needs the user's browser. Tell the user:

1. In Claude Code run `/mcp`, pick `fleeex`, choose **Authenticate** (other clients: their
   "connect"/"login" action for the server).
2. The browser opens `https://app.fleeex.dev/connect`. Sign in with the email code or
   Google (sign up first if there is no fleeex account yet).
3. Check the client name and the redirect address on the consent screen, then approve.

Tokens are stored and refreshed by the client: nothing to copy. Wait for the user to
confirm, then call `list_apps` to verify. A client may need a restart before new tools
show up.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `UNAUTHORIZED` from a tool | Re-run step 3. If it persists: `claude mcp remove fleeex`, then step 2 again. |
| Consent screen refuses the redirect | The client uses a non-loopback redirect: it is not supported. |
| `ACCOUNT_UNAVAILABLE` | The fleeex account is being deleted or was erased: contact `support@fleeex.dev`. |
| Tools never appear | Restart the client; check the server is listed (`claude mcp list`). |

## No MCP at all

Everything the integration needs is also in the dashboard (`https://app.fleeex.dev`):
create the app, mint keys, set the webhook. The user then writes the values into the env
file themselves (see **fleeex-api-keys** for the variable names). Do not ask them to paste
secrets into the chat.
