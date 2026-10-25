# Edge prototype plan (connector-first)

Edge is only a transport. See [architecture.md](architecture.md) for the lean design and its open questions. No code has been written against this design yet; it's awaiting review.

## ChatGPT connector requirements (verified)

From OpenAI's docs: [Build an MCP server](https://developers.openai.com/plugins/build/mcp-server), [Connect and test](https://developers.openai.com/plugins/deploy/connect-chatgpt), [Authentication](https://developers.openai.com/plugins/build/auth) and [Developer mode help](https://help.openai.com/en/articles/12584461).
- **Remote MCP, not OpenAPI.** It needs a public HTTPS endpoint (or OpenAI's Secure MCP Tunnel) that supports **Streamable HTTP**, typically at `/mcp`.
- Each tool needs a name, title, description, explicit input schema and accurate `readOnlyHint`/`destructiveHint`/`openWorldHint` annotations. Results return `structuredContent` and `content`.
- Auth is either none (anonymous) or OAuth 2.1 per the MCP authorization spec (protected-resource metadata, PKCE, and CIMD, DCR or predefined clients).
- Full MCP with write actions is in beta for **ChatGPT Business, Enterprise and Edu**, on web. A workspace admin must enable developer mode.
- **Connectors are per-user.** That's fine because this is a non-public prototype; no plugin submission is planned.

Claude: custom connectors take a remote MCP URL on Free (1 connector), Pro, Max, Team and Enterprise ([Claude help](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)).

## Other clients

See [compatibility.md](compatibility.md) for the full per-client matrix: ChatGPT (connector and GPT Action), Claude.ai/Desktop/Code, Codex CLI, Grok (web, CLI and API), A2A and curl. REST plus a static `openapi.json` serve GPT Actions and curl.

## Hosting

Cloudflare Workers with one Durable Object per inbox. See architecture.md for why not KV, a stateless relay or a single DO.

## Demo: ChatGPT → Claude → ChatGPT

1. `npx wrangler dev`, then test `/mcp` with `npx @modelcontextprotocol/inspector` (Streamable HTTP).
2. Jared deploys (`wrangler login`, `wrangler deploy`) to get a public HTTPS URL.
3. ChatGPT (Business/Enterprise/Edu, developer mode on): Plugins → **+** → **Add custom MCP server** → name "Edge" → URL `https://<worker>/mcp/<chatgpt-inbox-secret>` → auth: none → accept the risk warning → **Create** → review the discovered `send` and `read` tools.
4. Claude: Settings → Connectors → Add custom connector → `https://<worker>/mcp/<claude-inbox-secret>`.
5. In ChatGPT, type `@Edge`, then: "send `hello` to claude".
6. In Claude: "read my Edge inbox and reply." Claude calls `read`, then `send`.
7. In ChatGPT: "read my Edge inbox." The reply is there. Consumer apps can't receive pushes, so each side polls `read`.
8. If tool metadata changes: open the connection in ChatGPT Plugins → **Refresh**.

## Prior art

[Cross-Claude MCP](https://github.com/rblank9/cross-claude-mcp), [a2abridge](https://github.com/vbcherepanov/a2abridge), mcp-a2a-bridge, [agentgateway](https://github.com/agentgateway/agentgateway) and AgentiBridge. A2A is run by the Linux Foundation and is at v1.0; OpenAI and Anthropic don't support it natively. API fallbacks: [OpenAI remote MCP](https://developers.openai.com/api/docs/guides/tools-connectors-mcp) and the [Anthropic MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector).
