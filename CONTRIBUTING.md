# Working with me

Hi — I'm **Gennady Mikhaylov**, the engineer behind AiSpinner. This document is the answer to "how do I get in touch / hire / collaborate?"

## What I'm looking for

**Open to remote.** I'm available for:

- Senior / Staff engineering roles — backend, full-stack, real-time systems, voice AI / developer tooling
- Contracts and consulting — same areas, plus architecture reviews and tech audits
- Technical co-founder / founding engineer roles for the right product
- **Custom AiSpinner integrations** — see below

I'm comfortable taking ownership of an entire stack end-to-end (front to back, app to infra, design to deploy) and equally comfortable embedding into an existing team and following its conventions.

## Custom integration projects

The platform's block architecture is **declarative on purpose** — every block is registered through a single API (see [decisions.md → "A declarative block platform"](docs/decisions.md#13-a-declarative-block-platform-not-a-hard-coded-set)). That makes it realistic to build bespoke blocks for a specific client without forking the product.

Common shapes of work:

- **Private vendor block** — your internal service / SDK / database is wrapped as an AiSpinner block, available only to your team's workspaces
- **White-label deployment** — AiSpinner runs under your branding, with a curated subset of integrations and your own private blocks
- **Domain-specific verticals** — pre-built workspaces with the right blocks already wired up (e.g. a customer-support-voice template, a market-monitoring template)
- **Architecture / migration consulting** — I'll write the same kind of [decisions document](docs/decisions.md) for your platform that I wrote for mine, with concrete recommendations

If any of those fit what you're looking for, reach out via LinkedIn and we'll discuss scope.

## What I bring

A working product on a single VPS, serving four subdomains, six microservices, and 45 integration blocks. Things I have shipped and operated in production:

- **Real-time systems.** WebSocket hubs, sub-millisecond live data caches over long-lived third-party API connections, voice translation between two phone calls
- **Voice / telephony.** ElevenLabs Conversational AI, Asterisk ARI, Twilio, SIP, WebRTC, OpenAI Realtime, chunked auto-dialer with pause/resume
- **External API integrations.** Direct integrations with third-party APIs and data sources, with HMAC signing, an IP-rotation egress proxy that protects against third-party API IP-concentration rate limits, and WebSocket streaming
- **Sandboxed code execution.** Restricted Python sandbox with a uniform `_ApiMixin` adapter pattern (messengers, HTTP, LLM, state, files, monitor) — adding an integration is a small diff — plus persistent state, auto-heal, batched logs
- **AI integrations.** Multi-LLM routing across Claude / GPT / Grok / DeepSeek / Groq, MCP server with 13 tools, AI-assisted code writing, Claude Agent SDK
- **Frontend engineering.** Custom 20 000 × 20 000 px node canvas in Flutter Web, edge-wiring engine, declarative inspector schemas, real-time monitor widgets
- **DevOps.** GitHub Actions with paths-filter, ghcr.io, Docker Compose, Caddy, Postgres tuning, Redis tuning, Cloud Run egress

The architecture documents in this repo describe the technical work in concrete detail. The trade-off journal in [`docs/decisions.md`](docs/decisions.md) is where I'd rather be evaluated than a CV.

## How to reach me

- **LinkedIn** — [linkedin.com/in/gennady-mikhaylov](https://www.linkedin.com/in/gennady-mikhaylov/) (preferred)
- **GitHub Issues on this repo** — for technical questions, decision discussions, or "hey can you elaborate on X"
- **Private code walkthrough** — happy to do these as part of a hiring loop. Reach out via LinkedIn and we'll arrange it.

## What "contributing" actually means here

The product code is closed-source. This repository accepts:

- **Issues** for technical discussion, questions, suggestions, errata in the docs
- **Pull requests** that fix typos, improve documentation clarity, or add useful diagrams

The repository does **not** accept code contributions to the product itself, because the product code is not in this repository.

If you want to build something *like* AiSpinner — go ahead, the documentation here should be useful. If you want to build something *with* AiSpinner — open the [app](https://app.aispinner.io) and start drawing edges.

— Gennady
