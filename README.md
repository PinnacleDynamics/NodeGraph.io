<p align="center">
  <img src="assets/banner.png" alt="AiSpinner — Visual AI Agent Constructor" width="100%">
</p>

<h1 align="center">AiSpinner</h1>

<p align="center">
  <strong>Visual node-based constructor for AI voice agents, messenger bots, and 24/7 automation workflows.</strong>
</p>

<p align="center">
  <a href="https://aispinner.io">aispinner.io</a> ·
  <a href="https://app.aispinner.io">app.aispinner.io</a> ·
  <a href="https://docs.aispinner.io">docs.aispinner.io</a> ·
  <a href="https://api.aispinner.io">api.aispinner.io</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/blocks-45+-C8A25A?style=flat-square" alt="45+ blocks">
  <img src="https://img.shields.io/badge/messengers-7-C8A25A?style=flat-square" alt="7 messenger blocks">
  <img src="https://img.shields.io/badge/LLM_providers-5-C8A25A?style=flat-square" alt="5 LLM providers">
  <img src="https://img.shields.io/badge/MCP_tools-13-C8A25A?style=flat-square" alt="13 MCP tools">
  <img src="https://img.shields.io/badge/microservices-6-C8A25A?style=flat-square" alt="6 microservices">
  <img src="https://img.shields.io/badge/status-production-1A8754?style=flat-square" alt="status: production">
</p>

---

## What it is

AiSpinner is a **production-deployed** visual platform where users compose AI workflows by dragging blocks onto a canvas and connecting them with edges. Behind the scenes it executes those workflows — making and answering real phone calls, running AI voice agents, driving Telegram / WhatsApp / Discord / Matrix bots, running sandboxed Python 24/7, querying 5 LLM providers, calling any external API or CRM, and pushing real-time monitoring to dashboards.

> ![Canvas screenshot](assets/screenshots/canvas-overview.png)
> _Workspace canvas: Worker block connected to Telegram, an external API, Monitor, and Claude Agent. Drawing the edge automatically wires up the SDK — `ctx.telegram`, `ctx.http`, etc. — inside the worker code._

The repository here is a **public showcase**. The source itself is closed, but every architectural decision, integration, and trade-off is documented in this repo at the depth a senior engineer or a hiring AI assistant needs to evaluate it.

---

## Three things to look at

1. **[`docs/architecture.md`](docs/architecture.md)** — System topology: 6 microservices, data flow, real-time event hub, deploy pipeline, scale considerations.
2. **[`docs/integrations.md`](docs/integrations.md)** — Breadth: every block, every `ctx` adapter, MCP tools, telephony stack, WebSocket streaming tier.
3. **[`docs/decisions.md`](docs/decisions.md)** — The interesting bit: 13 architectural decisions with trade-offs spelled out. Read this if you're evaluating engineering judgement.

[`docs/api-overview.md`](docs/api-overview.md) covers the public REST + WebSocket + MCP surface for completeness.

---

## High-level architecture

```mermaid
flowchart LR
    subgraph Frontend
        FE[Flutter Web · Dart 3.9<br/>Canvas + Provider state]
    end

    subgraph Edge
        CADDY[Caddy 2 · automatic TLS<br/>4 subdomains]
    end

    subgraph Backend [Backend · 6 microservices · Python 3 / FastAPI]
        CORE[core-api<br/>auth · workspaces · MCP · Jarvis]
        TEL[telephony<br/>calls · journal · ARI · ElevenLabs]
        WR[worker-runtime<br/>Python sandbox · 16 ctx adapters]
        RT[realtime<br/>WebSocket hub · pub/sub]
        JV[jarvis<br/>AI assistant]
        ML[mailer<br/>Node.js]
    end

    subgraph Data
        PG[(PostgreSQL 16<br/>graphs · runs · users)]
        RD[(Redis 7<br/>state · cache · pub/sub)]
    end

    subgraph External
        EX[External APIs / data sources<br/>WebSocket + REST]
        MSG[Messengers<br/>Telegram · WhatsApp · Discord · Matrix]
        LLM[5 LLM providers<br/>Claude · GPT · Grok · DeepSeek · Groq]
        VOICE[ElevenLabs · Twilio<br/>SIP · WebRTC]
        PROXY[Managed egress proxy<br/>rotating IPs · EU region]
    end

    FE --> CADDY
    CADDY --> CORE
    CADDY --> TEL
    CADDY --> WR
    CADDY -->|/ws/events/| RT

    CORE --> PG
    TEL --> PG
    WR --> RD
    CORE --> RD
    TEL --> RD
    RT --> RD

    WR --> PROXY --> EX
    WR --> MSG
    WR --> LLM
    TEL --> VOICE
    JV --> LLM

    PG -.->|graph save| RT
    TEL -.->|call events| RT
    WR -.->|monitor stream| RT
    RT -->|push| FE
```

---

## Tech stack

| Layer | Technologies |
|---|---|
| **Frontend** | Flutter / Dart 3.9 · Provider · GoRouter · custom 20 000×20 000 px canvas with edge wiring |
| **Backend** | Python 3 · FastAPI · SQLAlchemy 2.0 (sync, psycopg2) · Pydantic |
| **Data** | PostgreSQL 16 · Redis 7 (LRU eviction policy) |
| **Real-time** | WebSocket hub with pub/sub fan-out · long-lived WS connections to external data sources with a 0 ms read cache |
| **Voice** | ElevenLabs Conversational AI · Asterisk ARI · Twilio · SIP · WebRTC · OpenAI Realtime (translator) |
| **Auth & crypto** | JWT (python-jose) · bcrypt · Fernet authenticated-encryption for stored secrets |
| **Edge / TLS** | Caddy 2 · automatic Let's Encrypt · gzip + zstd encoding |
| **Containers** | Docker · Docker Compose · private image registry |
| **CI / CD** | GitHub Actions with paths-filter — rebuild and redeploy only the services whose code changed |
| **Egress / network** | Managed serverless egress proxy with rotating IPs (EU region) — protects against third-party-API IP-concentration rate limits |
| **Block Platform** | Declarative `BlockDefinition` registry · auto-generated catalog & inspector UIs · rule-based edge wiring · uniform `_ApiMixin` for SDK adapters — adding a new integration is a small diff, not a fork |
| **AI integrations** | Claude / Grok / GPT / DeepSeek / Groq via unified `ctx.llm` router · MCP server with 13 tools (FastMCP) |
| **Observability** | Plausible analytics · structured logs · Redis stream of monitor events |
| **Analytics privacy** | No third-party trackers · cookieless analytics · Fernet at rest for secrets |

---

## Selected features

### Block Platform — extensible by design
Every block on the canvas — AI Agent, Phone Number, PBX, Voice Translator, Worker, the lot — is registered through a single declarative API. The catalog page, the inspector dialogs, the edge auto-wiring, and the SDK adapters are all generated from those declarations. Adding a new integration (a new messenger, a custom enterprise service, a vendor API) is a small diff, not a fork. This is the core engineering moat — and the reason custom-block work for clients is realistic without compromising the platform. See [decisions.md → "A declarative block platform"](docs/decisions.md#13-a-declarative-block-platform-not-a-hard-coded-set).

### Edges replace configuration
Drawing an edge between two blocks automatically wires up their SDK. Worker → Telegram edge ⇒ `ctx.telegram` becomes available inside the worker. Delete the edge ⇒ disconnected. **No manual config needed anywhere in the product.** Detailed write-up in [decisions.md → "Edges as the only source of configuration"](docs/decisions.md#1-edges-as-the-only-source-of-configuration).

> ![Edge wiring screenshot](assets/screenshots/edge-wiring.png)
> _Drawing an edge from Worker to Telegram auto-patches the worker's `cfg.telegram_node_id` field. The worker code now sees `ctx.telegram`._

### 24/7 sandboxed Python workers
Users write Python directly in a Worker block. The runtime executes it in a restricted sandbox: locked-down `__import__`, no `open()` / `socket` / `subprocess`, 60-second tick timeout, persistent state in Redis with periodic Postgres snapshot for durability. Auto-heal can restart on errors and even ask Claude to fix the code.

> ![Worker code editor](assets/screenshots/worker-code.png)
> _Worker block with code editor, deploy button, status, and live logs. Tick model with `setup(ctx)` / `tick(ctx)` / `on_error(ctx, error)`._

### Sub-millisecond live data via WebSocket cache
In Server execution mode, the runtime maintains long-lived WebSocket connections to external data sources and third-party APIs. Workers reading that data get cached state in **0 ms** instead of 80–250 ms via REST, with no rate-limit ceiling. Detailed write-up in [decisions.md → "WebSocket-cached live-data tier"](docs/decisions.md#3-websocket-cached-live-data-tier).

> ![Worker monitor](assets/screenshots/monitor.png)
> _Monitor block rendering live metrics, status indicators, data tables, and a progress bar — pushed by `ctx.monitor.render(...)` from a Server-mode worker._

### Real-time voice translation between two callers
The Voice Translator block bridges two phone lines, transcribes each side, translates via OpenAI Realtime, and speaks the translation back to the other party. A glossary field forwards proper-noun and technical-term hints into both Realtime sessions to prevent mistranslation.

### MCP-first AI control surface
A FastMCP server at `https://api.aispinner.io/mcp-api/mcp` exposes 13 tools — `get_workspace_graph`, `add_node`, `connect_nodes`, `update_node_config`, `start_dialer`, `restart_worker`, etc. — so any MCP-compatible AI (Claude Desktop, Cursor, ChatGPT) can read and modify workspaces directly. JWT-authenticated. Stateless.

### Voice agents and PBX
End-to-end ElevenLabs Conversational AI integration: phone numbers, AI agents, virtual PBX with chunked campaign auto-dialer, journal with audio playback and transcripts, post-call webhooks, operator routing. Custom Voice mode swaps in your own STT / LLM / TTS pipeline (Deepgram + DeepSeek + Cartesia) over Asterisk SIP/ARI.

> ![PBX block + Journal](assets/screenshots/pbx-journal.png)
> _PBX block running a campaign with parallel calls, alongside the standalone Journal block showing call history with transcripts and audio playback._

---

## What's measurable

| Surface | Number |
|---|---|
| Block types | 45+ across 7 sections |
| Messenger blocks | 7 across 4 platforms (Telegram · WhatsApp · Discord · Matrix) |
| LLM providers | 5 (Claude · GPT · Grok · DeepSeek · Groq) |
| Telephony stack | ElevenLabs Conversational AI · Asterisk ARI · Twilio · SIP · WebRTC |
| Worker SDK `ctx.*` adapters | messengers · HTTP · state · files · monitor · LLM · external data |
| MCP tools | 13 |
| Microservices | 6 |
| Subdomains served | 4 — apex, app, api, docs |
| Largest single frontend file | ~2 000 lines (Journal block — CSV export, audio player, campaign grouping, filters) |
| Telephony router | ~3 350 lines (calls, journal, dialer, runs, batch resolve, audio bridging) |
| Worker SDK | ~2 400 lines (16 adapters + sandbox bootstrap + log batcher) |

---

## Why is the source private?

AiSpinner is a closed-source commercial product. The repository here exists so that prospective collaborators, hiring teams, and AI evaluators can understand the architecture and engineering decisions without me handing over the entire production codebase.

If you'd like to:

- **Hire me, or contract me** — see [CONTRIBUTING.md](CONTRIBUTING.md).
- **Discuss a specific decision in depth** — open an issue with the file path / decision number; happy to elaborate publicly.
- **Get a private code walkthrough as part of a hiring loop** — reach out and we'll arrange it.

---

## About

Built by **Gennady Mikhaylov** — backend / full-stack engineer focused on real-time systems, voice AI, workflow automation, and developer-tooling UX.

[LinkedIn → linkedin.com/in/gennady-mikhaylov](https://www.linkedin.com/in/gennady-mikhaylov/)

Open to remote.

### Custom integrations

Because the platform is built around a declarative block registry, **bespoke integrations slot in cleanly**. If you have a service, an internal API, a vendor SDK, or a workflow that should live as a block on an AiSpinner canvas — for your team, your clients, or your product — get in touch via LinkedIn. We do this work on a project basis without forking the product.

---

## License

[Proprietary](LICENSE) — see the LICENSE file. The contents of this repository (architecture documentation, diagrams, and accompanying assets) are made public for evaluation purposes; the underlying product code is not licensed for redistribution.
