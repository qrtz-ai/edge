# Edge prototype plan

Goal of the PoC: prove that **consumer ChatGPT and Claude can exchange messages through Edge**.

## Constraints

- Neither ChatGPT nor Claude speaks A2A natively. Both can call **remote MCP servers**: Claude through custom connectors ([Claude help](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp), available on Free (1 connector), Pro, Max, Team and Enterprise), and ChatGPT through developer mode and MCP apps ([OpenAI help](https://help.openai.com/en/articles/12584461)).
- Remote connectors must reach a **public HTTPS URL** ([Claude: build a connector](https://claude.com/docs/connectors/building)). A localhost-only process can't serve the consumer apps, so **the PoC needs a hosted relay**.
- Consumer chat apps **can't receive pushes**. They act only during a turn, so they **poll an inbox tool** (`edge_inbox`). True push is only possible for API-driven agents and A2A peers.
- Full MCP support in ChatGPT, including write actions, is in beta for **Business, Enterprise and Edu** workspaces. On other plans, custom MCP may be limited to read/fetch, so the fallback is a Custom GPT Action (OpenAPI/REST) or the API.

## (a) Hosting

| Option | Pros | Cons |
|---|---|---|
| Small always-on server (VPS/Fly) | Simple; long-lived connections | Ops and uptime are on us |
| Serverless: Cloudflare Workers + Durable Objects | Public HTTPS by default; one Durable Object per inbox gives strongly consistent per-agent queues; scales to zero; Cloudflare has first-class remote MCP support ([docs](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)) | Workers runtime limits |
| Serverless: Vercel Functions | Easy deploys | Needs external storage (KV/Redis) for queues; no native per-inbox actor |
| Local-first + optional relay | Private; matches the "local add-on" vision | Consumer apps can't reach localhost |

**Pick: Cloudflare Workers + Durable Objects as the relay for the PoC**, with the architecture staying local-first. A local `edge` daemon is the long-term add-on for desktop and CLI agents, and it syncs through the same relay when peers aren't on the same machine.

## (b) Runtime

**Pick: TypeScript.**
- The official A2A SDKs cover JS/TS, Python, Go, Java and .NET ([a2aproject](https://github.com/a2aproject)). The official MCP TypeScript SDK ([modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk)) is the most mature MCP server SDK.
- **One TS codebase serves both surfaces**: A2A (JSON-RPC and Agent Card) for agent peers, and MCP (Streamable HTTP) for ChatGPT and Claude, over a shared message store.
- It runs on Workers, Node or Bun, so the same code can be the hosted relay and the local daemon.

## Message model (v1)

`send(to, body, thread?)` returns an id (fire-and-forget). `inbox(since?)` returns messages. `ack(ids)` marks them read. Each agent gets an address like `claude:jared`. Optional auth: bearer token or OAuth per inbox. Internally, messages map to A2A `Message` objects so A2A peers interoperate.

## (c) PoC demo: ChatGPT ↔ Claude

1. Scaffold a Worker with the MCP TS SDK (Streamable HTTP at `/mcp`) and the A2A JS SDK (Agent Card at `/.well-known/agent-card.json`).
2. Add a Durable Object `Inbox` keyed by address that stores messages and supports `send`, `inbox` and `ack`.
3. Expose MCP tools: `edge_send`, `edge_inbox` and `edge_ack`. Add a REST mirror (`POST /v1/send`, `GET /v1/inbox`) with an OpenAPI spec for Custom GPT Actions.
4. Add auth: a per-address bearer token (or OAuth) so each app can only read its own inbox.
5. Deploy with `wrangler deploy` to a public HTTPS URL (for example `edge.qrtz.ai`).
6. **Claude:** Settings → Connectors → Add custom connector → paste `https://…/mcp` → allow the Edge tools.
7. **ChatGPT:** enable developer mode (workspace settings), then Apps → Create → MCP endpoint `https://…/mcp`. *Fallback:* create a Custom GPT with an Action that imports the OpenAPI spec.
8. In Claude: "Send `hello from Claude` to `chatgpt:jared` via Edge." Claude calls `edge_send`.
9. In ChatGPT: "Check my Edge inbox and reply." ChatGPT calls `edge_inbox`, then `edge_send` to `claude:jared`.
10. In Claude: "Check Edge inbox." Round trip done. Record it as the demo.
11. **API fallback / automation:** a small script uses the OpenAI Responses API and the Anthropic Messages API, both of which can call remote MCP servers ([OpenAI remote MCP](https://platform.openai.com/docs/guides/tools-remote-mcp), [Anthropic MCP connector](https://docs.anthropic.com/en/docs/agents-and-tools/mcp-connector)). It runs both sides in a loop for a hands-free demo, and it's the only part with real push.
12. Stretch: point an A2A client at the Agent Card to show that an A2A agent can also message `claude:jared`.

## Prior art

- [Cross-Claude MCP](https://github.com/rblank9/cross-claude-mcp): a shared message bus for Claude, Gemini (MCP) and ChatGPT (REST).
- [a2abridge](https://github.com/vbcherepanov/a2abridge): makes local coding agents into A2A peers.
- mcp-a2a-bridge (MCP↔A2A translation), [agentgateway](https://github.com/agentgateway/agentgateway) (enterprise agent proxy) and AgentiBridge (Claude session index and dispatch).
- The A2A spec is run by the Linux Foundation, v1.0 is out, it has 150+ supporting orgs and ~26k GitHub stars. OpenAI and Anthropic don't support it natively.

Edge's gap: a simple, install-once, cross-vendor bridge for regular users, built on A2A with an MCP front door.
