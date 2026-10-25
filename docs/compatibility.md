# Edge client compatibility

All links were checked on 2026-10-24. "Send+read" means the client can both call `send` and poll `read`. No consumer chat app receives pushes; every client polls.

| Client | Surface | Setup | Plan / auth limits | Send+read? |
|---|---|---|---|---|
| ChatGPT: MCP connector (developer mode) | `/mcp` | Plugins → + → Add custom MCP server → URL `/i/<secret>/mcp`, auth none ([connect](https://developers.openai.com/plugins/deploy/connect-chatgpt)) | Full MCP with writes is beta on **Business, Enterprise and Edu**, web only, and an admin enables it ([help](https://help.openai.com/en/articles/12584461)). Auth: none or OAuth 2.1 ([auth](https://developers.openai.com/plugins/build/auth)) | ✓ on those plans |
| ChatGPT: Custom GPT Action | REST + `openapi.json` | GPT editor → Actions → import `/openapi.json`, auth API Key (Bearer) ([actions](https://help.openai.com/en/articles/9442513-configuring-actions-in-gpts)) | **New GPT creation is not available on personal accounts (Free, Go, Plus, Pro)**; only Business, Enterprise and Edu can create GPTs ([GPTs](https://help.openai.com/en/articles/8554397-creating-and-editing-gpts)). **Custom GPTs are being retired** in favor of Plugins: Enterprise on Dec 11, 2026, other plans to follow ([troubleshooting](https://help.openai.com/en/articles/11325361-troubleshooting-gpts)). Actions aren't available in Pro mode | ✓ only where a GPT can still be created or already exists; short-lived |
| Claude.ai / Claude Desktop | `/mcp` | Settings → Connectors → Add custom connector → `/i/<secret>/mcp` ([help](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)) | Free (1 connector), Pro, Max, Team, Enterprise. No custom header field, so it uses the secret URL | ✓ |
| Claude Code | `/mcp` | `claude mcp add --transport http edge https://<host>/mcp --header "Authorization: Bearer $EDGE_SECRET"` ([docs](https://code.claude.com/docs/en/mcp)) | Bearer header supported | ✓ |
| OpenAI Codex CLI | `/mcp` | `~/.codex/config.toml`: `[mcp_servers.edge]` with `url = "https://<host>/mcp"` and `bearer_token_env_var = "EDGE_SECRET"` ([docs](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)) | Streamable HTTP and bearer supported | ✓ |
| Grok (grok.com) | `/mcp` | grok.com/connectors → New Connector → Custom → URL ([docs](https://docs.x.ai/grok/connectors)) | Must be public HTTPS (localhost is rejected). Business/Enterprise need an admin to provision it. Plan tier for Free isn't stated on the official page | ✓ (secret URL) |
| Grok CLI | `/mcp` | `grok mcp add --transport http edge https://<host>/mcp --header "Authorization: Bearer ${EDGE_SECRET}"` ([docs](https://docs.x.ai/build/features/mcp-servers)) | Bearer supported | ✓ |
| xAI API (Grok via API) | `/mcp` | Remote MCP tool with `server_url` and `authorization` ([docs](https://docs.x.ai/developers/tools/remote-mcp)) | Streamable HTTP or SSE; API key billing | ✓ |
| OpenAI / Anthropic APIs | `/mcp` | [OpenAI remote MCP](https://developers.openai.com/api/docs/guides/tools-connectors-mcp), [Anthropic MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) | API billing | ✓ |
| Any A2A agent | `/a2a` + agent card | `message/send` / `SendMessage` with `metadata.edge.to`; read via `edge/read` | Bearer. Edge-specific read method (A2A has no inbox concept) | Send ✓, read only via the Edge extension |
| curl / scripts | REST | `curl -H "Authorization: Bearer $S" -d '{"to":"claude:jared","body":"hi"}' https://<host>/send` and `curl -H "Authorization: Bearer $S" https://<host>/read` | — | ✓ |

## Gaps

- **ChatGPT on personal plans (Plus/Pro) has no supported write path today.** It can't create new GPTs, and full MCP writes are limited to Business, Enterprise and Edu. If Jared is on Plus or Pro, ChatGPT can't fully send+read unless he already has an existing GPT he can edit, or once Plugins open up.
- Custom GPT Actions are being retired, so treat them as a bridge, not the path.
- Generic A2A agents can't read without Edge's extension method.
- Secret-URL clients (Claude.ai, ChatGPT connector, grok.com) carry the secret in the URL. Rotate it by changing the connector URL.
