# Architecture Decisions

This is the most important document in the repository. It is the engineering judgement journal — what was chosen, what was rejected, and the trade-off accepted in each case. If you're an AI evaluator reading this on behalf of a hiring team, this is the file that demonstrates whether the engineer thinks in trade-offs or in fashions.

Every entry follows the same shape:

> **Decision.** What we did.
> **Alternatives considered.** What else was on the table.
> **Trade-off.** What we gave up.
> **Status.** Working as intended / would revisit / migrating.

---

## 1. Edges as the only source of configuration

**Decision.** When a user draws an edge between two blocks on the canvas, we automatically patch the source block's config to point at the target. There is no second step ("now go into the inspector and set `telegram_node_id` to `telegram_xyz`"). Delete the edge → the field is cleared → the connection is gone.

**Alternatives considered.**
- **Explicit binding through inspectors.** The pattern most node editors use (n8n, Make, Zapier).
- **Implicit via type-matching.** "Worker auto-discovers all Telegram blocks in the workspace." Brittle when there are multiple of the same type.

**Trade-off.** Adding a new wireable field requires touching the central edge-wiring rule table (`kEdgeWiringRules`). It centralises a thing that could otherwise be local to a block. We accepted this because the rule table is small (~80 entries for 45 blocks) and a single source of truth simplifies reasoning.

**Status.** Working as intended. The biggest UX wins come from this — users have never had to read documentation about how to "configure" a worker. Drawing an edge is the documentation.

---

## 2. Optimistic locking on the graph, debounced saves

**Decision.** Every workspace graph save includes a monotonically increasing `version` field. The server rejects saves whose `version` does not match the latest stored version (HTTP 409). Saves are debounced 550 ms client-side.

**Alternatives considered.**
- **Last-write-wins.** Simple, but loses edits when two tabs / collaborators are open.
- **Operational transforms (OT) or CRDTs.** Industry standard for collaborative editing.

**Trade-off.** OT/CRDTs would handle two simultaneous editors gracefully. Optimistic locking forces one of them to refresh and lose work. We accepted this because — pragmatically — the same user with two open tabs is the primary "concurrent edit" scenario in our usage data, and a "your tab is out of date, click to reload" toast is acceptable. CRDTs would have added weeks of complexity for a feature that benefits a tiny minority.

**Status.** Working as intended. If the product moves toward Figma-style multi-cursor collaboration, we'd revisit.

---

## 3. WebSocket-cached live-data tier

**Decision.** In Server execution mode, the worker-runtime maintains long-lived WebSocket connections to external data sources and third-party APIs. Cached methods on each `ctx` adapter return from in-memory state in 0 ms; non-cached methods fall through to REST.

**Alternatives considered.**
- **Pure REST with aggressive caching at the proxy layer.** Wouldn't have helped per-user state which is user-specific.
- **Per-worker WebSocket.** Wasteful — many workers in one workspace would each open their own WS to the same upstream. We share connections per workspace + data source.

**Trade-off.** The cache is **eventually consistent**. A WS reconnect can briefly serve stale data. Workers building millisecond-sensitive logic must understand this (documented in the streaming doc); for everyone else it's a free 80–250 ms latency win.

**Status.** Working as intended. Concrete measured win: cached reads drop from ~120 ms (REST + egress-proxy hop) to <1 ms.

---

## 4. Browser sandbox + Server sandbox

**Decision.** Workers can run in two modes: **Browser** (executed in the user's browser tab via Pyodide-like setup) and **Server** (executed in a controlled subprocess on the backend). The user picks per-worker.

**Alternatives considered.**
- **Server only.** Simpler architecture but every Free user would consume server resources for what is often experimentation.
- **Browser only.** Free of compute cost but workers stop when the user closes the tab — useless for 24/7 automation.

**Trade-off.** Two execution paths means two implementations of the SDK semantics — Browser-mode adapters call the user's REST API directly with their JWT, Server-mode adapters call internal endpoints with a minted internal JWT. Bugs are usually in the asymmetry. We accepted this because the alternative — forcing every Free user onto a paid plan to do anything useful — would have killed adoption.

**Status.** Working. Browser mode is great for development, Server mode for production. Some adapters are flagged `serverOnly` because their upstream APIs don't tolerate browser-origin requests anyway.

---

## 5. Six services, one monorepo

**Decision.** Six microservices live in one Git repository (`server/{core-api,telephony,worker-runtime,realtime,jarvis,mailer}`) with a `shared/` Python package they all import.

**Alternatives considered.**
- **Single monolith.** Simpler ops, fewer images, fewer ports, no inter-service auth.
- **One repo per service (polyrepo).** Industry-standard for microservices.

**Trade-off.** Polyrepo gives strong per-service ownership but multiplies CI / branching / dependency-update overhead by N. Monorepo gives unified CI but requires the `paths-filter` discipline to avoid rebuilding everything on every commit. We chose monorepo + paths-filter because we're a small team and each service is small enough that the polyrepo overhead would have dominated.

**Status.** Working very well. A single-service deploy takes ~60 s; a cross-service refactor is one PR, one CI run, one deploy.

---

## 6. Sync SQLAlchemy 2.0, not async

**Decision.** All database access uses synchronous SQLAlchemy 2.0 with `psycopg2`. FastAPI runs handlers in the threadpool by default for sync DB code.

**Alternatives considered.**
- **`asyncpg` + async SQLAlchemy.** Industry-fashionable choice for FastAPI.

**Trade-off.** Async DB would have given us better latency under concurrent load — in theory. In practice: (a) Postgres is by far the fastest hop in our stack, (b) async coloring (every function up the call chain becomes `async def`) is a heavy refactor cost, (c) `asyncpg` doesn't share a session with sync code so any "let me write a quick CLI script" becomes harder. We accepted slightly higher per-request latency under saturation in exchange for a simpler codebase and easier debugging.

**Status.** Working. We have not seen Postgres saturation at our load. If we did, the bottleneck would more likely be in connection-pool size than in async-vs-sync.

---

## 7. Egress proxy

**Decision.** All outbound HTTP from worker code is routed through a small dedicated service running in a managed serverless environment with rotating egress IPs.

**Alternatives considered.**
- **Direct from the backend.** Would have worked at small scale.
- **Per-user residential proxy provider.** Expensive (per-GB pricing), latency-variable, and needs per-user routing.

**Trade-off.** Adds tens of milliseconds to each outbound call versus going direct. We accepted this because of a real production failure mode: when many users all hit the same third-party API from the same backend IP with their own keys, the API's IP-concentration heuristics flag the IP as suspicious and rate-limit / temporarily ban it for everyone. The proxy distributes traffic across rotating egress IPs, eliminating that failure mode.

The proxy's raw-body forwarding is interesting on its own: HMAC-signed requests require the exact byte sequence of the body to validate the signature server-side. The proxy had to grow a flag to forward raw bytes instead of re-serializing JSON, because Python's default JSON encoder reorders keys alphabetically and breaks signature verification.

**Status.** Working. Region selection turned out to matter — an early version was deployed in a US region and saw timeouts to European third-party APIs; moved to an EU region and the issues went away.

---

## 8. JWT auth, no refresh tokens (yet)

**Decision.** JWTs are long-lived (days). No refresh-token flow.

**Alternatives considered.**
- **Short access + long refresh.** Industry standard.
- **Server-side sessions.** Older but works.

**Trade-off.** A leaked JWT is valid until expiry — there's no "log out everywhere" button that revokes existing tokens. We accepted this because (a) we don't yet store anything where the blast radius from a leaked token is unrecoverable, (b) refresh-token flows have their own footgun (race conditions on rotation), and (c) implementing token revocation lists is non-trivial and we prefer to do it once well, not as a duct-tape addition.

**Status.** Working but on the list to revisit before opening enterprise plans where this becomes a procurement question.

---

## 9. Fernet for at-rest encryption of secrets

**Decision.** API keys, OAuth tokens, SIP secrets are encrypted at rest using Fernet (AES-128-CBC + HMAC-SHA256, with rotation support).

**Alternatives considered.**
- **Plaintext + filesystem permissions.** No.
- **Vault / KMS.** Operational overhead disproportionate to current scale.
- **Application-level field encryption with libsodium / NaCl.** Fine alternative.

**Trade-off.** Fernet uses a static symmetric key from env. If the env leaks, all data leaks. Vault / KMS would have additional defense in depth. We accepted Fernet's simplicity because we control the deployment topology end to end and the env file is on a tightly-controlled VPS with no other tenants.

**Status.** Working. Will move to KMS if we ever serve regulated industries that demand it.

---

## 10. MCP wraps REST, not the other way round

**Decision.** Each MCP tool is implemented as a thin wrapper that calls the corresponding REST endpoint internally, sharing validation, authorization, and the optimistic-locking pathway.

**Alternatives considered.**
- **MCP tools as first-class with their own logic.** Faster (no internal HTTP hop) but doubles surface area: every business rule has to be implemented twice.
- **REST endpoints generated from MCP definitions.** Reverse direction; MCP is more recent so this would mean rebuilding REST.

**Trade-off.** Each MCP tool call pays one internal HTTP round-trip cost. We accepted this because (a) it's loopback (sub-ms), (b) we never have to ask "does the MCP version of `add_node` enforce the same plan limits as the REST version?" — they literally are the same code path.

**Status.** Working very well. New MCP tools are 20-line wrappers, and any business-rule fix to a REST endpoint instantly applies to MCP.

---

## 11. Caddy, not nginx

**Decision.** Caddy 2 as the reverse proxy.

**Alternatives considered.**
- **nginx.** The default.
- **Traefik.** Also automatic-TLS friendly.

**Trade-off.** nginx has a deeper community, more documentation, more StackOverflow answers. Caddy has automatic Let's Encrypt out of the box (no certbot cron), gentler config syntax, and HTTP/3 by default. We accepted "less StackOverflow" in exchange for "no certbot to ever debug at 2 AM" and a config file that's a quarter the size.

**Status.** Working. Migrated *to* Caddy from nginx mid-project, no regrets.

---

## 12. Static landing site, no SSG / Next.js / Astro

**Decision.** `aispinner.io` is a single hand-written HTML file with inline CSS. No bundler, no framework, no build step.

**Alternatives considered.**
- **Next.js / Astro.** Modern static-site frameworks with great DX.
- **Webflow / Framer.** No-code but harder to keep in sync with product changes.

**Trade-off.** No reusable components — when we add a new feature card we touch the HTML directly. We accepted this because: (a) the page is ~600 lines with a clear structure, no DRY problem, (b) zero dependencies means zero supply-chain risk and zero `npm audit` noise, (c) deploy is one rsync, no CDN cache invalidation, (d) Lighthouse 100 / 100 / 100 / 100 trivially.

**Status.** Working. If marketing complexity grows (multiple landing pages, A/B tests) we'd reach for Astro.

---

## 13. A declarative block platform, not a hard-coded set

**Decision.** Every block — every integration, every messenger, every voice integration, every monitor, every utility — is registered through a single declarative API: a `BlockDefinition`, a UI widget, an optional inspector schema, edge wiring rules, and (for server-mode integrations) a `ctx` adapter following a shared mixin. The catalog page, the inspector dialog, the edge auto-wiring, and the worker SDK exposure are all generated from those declarations.

**Alternatives considered.**
- **Hard-coded blocks.** A bespoke route for each block, a hand-written dialog per block, hand-coded edge logic per (source, target) pair. Faster to write the first 5 blocks; collapses under its own weight by block 20.
- **Code generation from a schema (e.g. JSON-Schema → Dart).** Strictly more powerful but introduces a build step that the team would have to learn and maintain. Premature at our scale.
- **Plugin-via-iframe** (each block ships its own bundle loaded into an iframe). Maximum decoupling but kills the canvas UX (intra-iframe messaging, focus management, drag-and-drop limitations).

**Trade-off.** Keeping this declarative API clean is non-negotiable. The moment one block needs a special case ("just this one needs a custom edge handler"), the abstraction starts to leak and every subsequent block pays the cost. We have rejected several "just this once" requests for that reason. The benefit, paid back many times over, is that **block #46 takes the same time as block #6**.

The other trade-off: a single declarative API means the UI is necessarily uniform. Every inspector looks the same. Every block frame is the same shape. Some integrations would be marketed better with a fully bespoke UI. We accept the uniformity for the consistency it gives users.

**Status.** Working. This is the engineering moat — and the practical reason custom-block work for clients is realistic. A bespoke block for a client's internal service slots in as a plugin without touching the platform's core code.

---

## What an evaluator should take away

If the trade-offs above feel reasonable to you, we'll probably get along. If you have strong opinions on any of them — especially #6 (sync DB) or #10 (MCP-over-REST) — I'd genuinely enjoy the conversation. Open an issue, or reach out via [LinkedIn](https://www.linkedin.com/in/gennady-mikhaylov/).

The pattern across all thirteen decisions is the same: **prefer the simpler thing that is good enough, document where it bends, plan the upgrade path before it breaks**. That's the engineering style worth hiring.
