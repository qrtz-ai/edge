# Edge architecture (proposal, awaiting review)

**Edge is only a transport.** It moves an opaque message from A to B and holds it until B reads it. There is no logic, no LLM calls, no parsing of message bodies, no registry, no search, and no orchestration.

## The pick

| Part | Choice |
|---|---|
| Moving parts | **1 Worker + 1 Durable Object class (one instance per inbox, SQLite-backed)** |
| Runtime deps | **0** (Worker hand-rolls the ~4 MCP JSON-RPC methods it needs) |
| Endpoints | `POST /mcp` (MCP, Streamable HTTP, JSON responses); `POST /send` and `GET /read?ack=` (REST for GPT Actions and curl); `POST /a2a` (A2A shim); static `/openapi.json` and `/.well-known/agent-card.json` |
| Tools | **2:** `send(to, body)` and `read(ack?)` |
| Wire format | `{id, from, to, body, ts}`; `body` is an opaque string, stored and returned byte-for-byte |
| Delivery | At-least-once. TTL is 7 days, enforced by filtering on read and deleting lazily |
| Auth | Per-inbox secret token (the capability). See the open questions |
| Budget | Worker ≤ ~180 LOC (REST adds ~30), `openapi.json` ~60 lines, tests ≤ ~120 LOC. Runtime deps: 0 |

## Storage: what's the minimum?

| Option | Cost | Verdict |
|---|---|---|
| Stateless relay (no storage) | Recipient must be online. Consumer ChatGPT/Claude can't receive pushes, so messages would be lost | ✗ Can't work |
| Workers KV | Eventually consistent: writes "may take up to 60 seconds or more to be visible in other global network locations" ([Cloudflare KV](https://developers.cloudflare.com/kv/concepts/how-kv-works/)). Listing and ack races between POPs | ✗ A reply could be invisible for a minute |
| Single global DO | Simplest code, strongly consistent; one hot object and a single point of contention | OK for a PoC, but no simpler than one per inbox |
| **One DO per inbox** | Strongly consistent; `idFromName(inbox)` is the only routing needed. SQLite-backed DOs run on the Workers Free plan ([DO pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/)) | **✓ Pick.** It's the same code as a single DO, with isolation for free |

## Wire: A2A is a shim, not the core

The native wire is the 5-field envelope above. A2A is translated only at `/a2a`: `message/send` (or v1.0 `SendMessage`) text parts are joined into `body`, and `metadata.edge.{from,to}` become the addresses. The Agent Card is a static JSON. Edge does **not** implement A2A tasks, streaming, push notifications or artifacts. That keeps A2A compatibility without A2A weight.

## Tools: 3 → 2

`ack` folds into `read`. `read({ack: [ids]})` deletes the listed ids and then returns what's left, so a client acks the previous batch on its next poll. This stays at-least-once (anything not acked is redelivered) with one fewer tool. Reading and deleting in one step (at-most-once) was rejected because a dropped response would lose messages.

## Dependencies and cold start

- `@modelcontextprotocol/sdk` is **6.3 MB installed with 17 direct deps** (measured locally with `npm i`, v1.x). The Worker only needs `initialize`, `tools/list`, `tools/call` and `ping` over JSON responses, which Streamable HTTP allows. Hand-rolling them is about 40 LOC, so **the Worker ships 0 runtime deps**.
- Workers limits: 64 MiB max size and 1 s startup time ([limits](https://developers.cloudflare.com/workers/platform/limits/)). A dependency-free ~180-line Worker is a few KB, far under both.
- Dev-only deps: `typescript`, `vitest` and `@cloudflare/workers-types`.

## Cut, or out of scope

Cut: the separate `ack` tool, the Claude-side stdio wrapper (see Q2), JSON-RPC batch handling, the `/` info page, the `agent.json` alias, the MCP SDK in the Worker. (REST/OpenAPI was restored for GPT Actions and curl. All three surfaces share one ~10-line handler.)
Out of scope, forever or until a later phase: registries/discovery, contacts, shared datasets, search, embeddings, orchestration, LLM calls, message transforms, streaming/SSE, push webhooks, and A2A tasks.

## Open questions for Jared

1. **Auth shape.** ChatGPT's custom MCP auth is either none or OAuth 2.1 per the MCP authorization spec ([OpenAI auth](https://developers.openai.com/plugins/build/auth)). A plain static bearer header isn't a documented ChatGPT option. The cheapest option is a **capability URL**, `/mcp/<inbox-secret>`, with "no auth" set in ChatGPT. The secret is the inbox. That's zero extra code, but anyone holding the URL owns the inbox. OAuth is real auth but adds an authorization server, which is far more than the transport itself. Is a capability URL OK for the non-public PoC?
2. **Claude side.** Claude custom connectors also take a remote MCP URL ([Claude help](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)), so Claude can use the same `/mcp` directly, with no wrapper and no SDK. Drop the planned Claude-side A2A wrapper?
3. **TTL.** Is 7 days right, and should there be a per-inbox cap (for example 1,000 messages)?
4. **Message size cap.** 16 KB?

## REST surface (approved: "It needs to work w everything")

- `POST /send` with body `{to, body}` returns `{id, ts}`.
- `GET /read?ack=id1,id2` deletes the acked ids, then returns `{messages:[{id,from,to,body,ts}]}`.
- Static `/openapi.json` (OpenAPI 3.1) for Custom GPT Actions.
- These are the same two operations as the MCP tools and the same envelope, and they share the same handler.

**Auth mapping.** GPT Actions support **None, API Key or OAuth** ([OpenAI Actions auth](https://developers.openai.com/api/docs/actions/authentication)). Edge accepts the inbox secret either as a path segment (`/i/<secret>/send`, which works with "None") or as `Authorization: Bearer <secret>` (which works with "API Key → Bearer"). Prefer the bearer form wherever a client supports headers. It's still one secret per inbox, and the Worker hashes it to the DO name.

## Cost

One request is one Worker invocation plus one DO request. SQLite-backed DOs are on the Workers Free plan ([DO pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/)). The PoC volume should be $0.
