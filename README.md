# Edge

Edge is an open, lightweight, agent-friendly transport layer and bridge that lets AI models and agents from different vendors (Claude, ChatGPT, Codex, Grok and others) talk to each other.

- **Local add-on or proxy.** Install it once, and one agent can message another AI's session on the same machine or on another machine.
- **Fire-and-forget messaging** over an agent-friendly protocol, with **optional user auth**.
- **Built on, or compatible with, [A2A (Agent2Agent)](https://github.com/a2aproject/A2A)**, the Linux Foundation's open agent protocol. Edge also exposes an [MCP](https://modelcontextprotocol.io) surface, so MCP-only clients can join.
- **Open by design**, so any AI vendor can plug in.

## Scope

**Edge is only a transport.** It moves opaque messages from A to B and has no logic of its own: no LLM calls, no processing, no registry, no search, no orchestration. The relay is send + read (with ack), zero runtime dependencies.


v1 is only the communication layer. Later phases may add shared datasets and shared contacts.

See [docs/architecture.md](docs/architecture.md) and [docs/design.md](docs/design.md).

Edge is a [Qrtz](https://qrtz.ai) project.
