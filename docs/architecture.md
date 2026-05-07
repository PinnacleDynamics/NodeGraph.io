# Architecture

This document describes the production architecture of AiSpinner — the topology, the services, the data flow, and the deploy pipeline. It is written for an engineer or an AI evaluator deciding whether the project demonstrates production-grade thinking.

For the trade-offs that drove these choices, see [decisions.md](decisions.md).

---

## 1. Topology

A single VPS hosts the production stack. **12 cores, 48 GB RAM, 250 GB NVMe.** No managed services on the critical path; every component is operationally legible.

```mermaid
flowchart TB
    Internet((Internet))

    subgraph Server["VPS · 12 cores · 48 GB RAM"]
        Caddy[Caddy 2<br/>automatic TLS · 4 subdomains<br/>gzip + zstd]

        subgraph Apps["Application containers"]
            CoreAPI[core-api<br/>:8000 · 8 uvicorn workers]
            Telephony[telephony<br/>:8001 · voice + dialer workers]
            WorkerRT[worker-runtime<br/>:8002 · sandbox + adapters]
            Realtime[realtime<br/>:8003 · WS hub]
            Jarvis[jarvis<br/>:8004 · AI assistant]
            Mailer[mailer<br/>:5055 · Node.js email]
        end

        subgraph Data["Data layer"]
            PG[(PostgreSQL 16<br/>shared_buffers 12 GB)]
            Redis[(Redis 7<br/>maxmemory 4 GB · allkeys-lru)]
        end
    end

    subgraph External["Off-server"]
        CloudRun[Cloud Run egress proxy<br/>europe-west1 · rotating IPs]
        GHCR[(ghcr.io<br/>image registry)]
        GHA[GitHub Actions<br/>CI · paths-filter]
    end

    Internet --> Caddy
    Caddy --> CoreAPI
    Caddy --> Telephony
    Caddy --> WorkerRT
    Caddy --> Realtime
    Caddy --> Jarvis

    CoreAPI --> PG
    Telephony --> PG
    WorkerRT --> PG
    CoreAPI --> Redis
    Telephony --> Redis
    WorkerRT --> Redis
    Realtime --> Redis

    WorkerRT --> CloudRun
    Telephony --> CloudRun

    GHA --> GHCR --> Apps
```

**Reverse proxy.** Caddy 2 fronts everything. Four subdomains are served from the same instance:

| Subdomain | Backed by |
|---|---|
| `aispinner.io` | static landing site (HTML/CSS) |
| `app.aispinner.io` | Flutter Web build with SPA fallback (`try_files {path} /index.html`) |
| `api.aispinner.io` | core-api with WebSocket upgrade for `/ws/events/*` |
| `docs.aispinner.io` | static VitePress build |

Caddy handles automatic Let's Encrypt renewal, gzip + zstd encoding, and host-specific cache-control rules (e.g. no-cache for the Flutter bootstrap files, long cache for hashed asset bundles).

---

## 2. Microservices

Six processes, all built from the same monorepo. Each has a small, clear responsibility.

| Service | Port | Responsibility |
|---|---|---|
| **core-api** | 8000 | Authentication (JWT, bcrypt), workspaces, graph CRUD with optimistic locking, node configs, integrations (encrypted API keys), MCP server with 13 tools, Jarvis chat |
| **telephony** | 8001 | ElevenLabs Conversational AI, Asterisk ARI bridge, SIP/Twilio phone-number import, chunked campaign auto-dialer, journal with transcripts and audio, post-call webhooks, voice translator (OpenAI Realtime) |
| **worker-runtime** | 8002 | Python sandbox executor with restricted builtins, 16 `ctx.*` adapters (state, log, monitor, http, 10 exchanges, telegram, files, llm), WebSocket cache to 6 exchanges, log batcher, monitor event stream |
| **realtime** | 8003 | WebSocket hub. Single connection per workspace fans out 7 event types (journal.new, call.done, batch.status, graph.version, etc.) to all subscribed clients |
| **jarvis** | 8004 | AI assistant chat. Uses the same ai_tools registry as MCP — keeps the assistant's capabilities and the MCP surface in lock-step. |
| **mailer** | 5055 | Transactional email (verification codes, password reset). Node.js — historical artifact from before the Python stack consolidated. |

A `shared/` Python package is imported by every Python service: ORM models, DB session factory, Redis client, Fernet crypto, Pydantic schemas. This keeps types, table definitions, and connection pools identical across services without copy-paste drift.

### Why six services and not one monolith?

Each service has independent CI / deploy tempo. A bug fix in `worker-runtime` rebuilds and pushes only one image. A schema migration in `core-api` doesn't restart the telephony workers. The CI workflow uses GitHub Actions `paths-filter` to compute the changed services from the diff and rebuild only those. See [decisions.md → "Six services, one monorepo"](decisions.md#5-six-services-one-monorepo).

---

## 3. Data layer

### PostgreSQL 16

Tuned for the workload (read-heavy graph + write-heavy telephony events):

| Knob | Value | Rationale |
|---|---|---|
| `shared_buffers` | 12 GB | 25 % of RAM — the standard PG starting point on 48 GB |
| `effective_cache_size` | 36 GB | 75 % of RAM — informs the planner about OS page cache |
| `work_mem` | 64 MB | Sortable journal queries with multiple status / date filters |
| `shm_size` (Docker) | 16 GB | Required for the increased shared_buffers + work_mem combo |

**Hot tables.**

| Table | What it stores | Notes |
|---|---|---|
| `users` | accounts, hashed password, country code | country derived once on first login from IP |
| `workspaces` | workspace metadata | one workspace per user on Free, unlimited on Max |
| `workspace_documents` | full graph JSON | optimistic locking via monotonically increasing `version` |
| `workspace_node_configs` | per-node JSON config (e.g. PBX settings) | sidecar to graph_json so node-level edits don't bump graph version |
| `phone_provider_configs` | encrypted Twilio / SIP credentials per node | Fernet encryption at rest |
| `telephony_runs` | every call + every campaign | parent rows are campaigns, child rows are individual calls |
| `ari_providers` / `elevenlabs_api_keys` | encrypted integration credentials | `key_hash` (SHA-256) is unique per user to prevent dup-add |

The graph_json column is JSONB. We rebuild it whole on every save (instead of patching). Saves are debounced 550 ms client-side and version-checked server-side — see decisions.md.

### Redis 7

Used as four distinct things, segmented by key prefix:

1. **Worker state** — `state:{ws_id}:{node_id}:*` — the persistent KV store backing `ctx.state.set/get`
2. **Caches** — `journal_cache:{md5}`, `pbx_op:*`, `pbx_op_ws:*` — short TTL (60–300 s) for hot reads
3. **Pub/sub** — `events:ws:{ws_id}` — the channel that the realtime hub fans out to WS clients
4. **Coordination** — `campaign:{call_id}:signal`, `campaign:{call_id}:running` — TTL'd flags that the dialer reads each chunk to support pause / resume / stop

`maxmemory 4 GB` with `allkeys-lru` ensures we never run out — when memory pressure hits, the LRU caches evict first; state keys, being touched constantly, stay hot.

---

## 4. Real-time layer

The realtime service is a thin WebSocket hub. It does **two** things:

1. Holds an open WS per connected workspace client at `/ws/events/{workspace_id}?token=<JWT>`.
2. Subscribes to a Redis pub/sub channel per workspace and forwards every message to that workspace's WS clients.

Other services (telephony, worker-runtime, core-api) **publish** events to Redis. They never know who's subscribed. The realtime service has no knowledge of business logic. This decoupling is the entire architectural insight here.

### Event types

| Type | Publisher | Use |
|---|---|---|
| `journal.new` | telephony (after a call resolves) | Journal block appends incrementally instead of polling |
| `call.done` / `call.transcript` | telephony | Live status updates during a campaign |
| `batch.status` | telephony | ElevenLabs batch progress (X/Y completed) |
| `campaign.started` / `campaign.stopped` | telephony | Campaign lifecycle |
| `graph.version` | core-api (after PUT /workspaces/:id/graph) | Multi-tab / multi-collaborator sync |
| `subscribed` / `pong` | realtime | Connection management |

Plus pass-through telephony events (DTMF, voicemail-detected, interruption signals) that aren't enumerated explicitly — `realtime` forwards anything published to the workspace channel.

### Why not server-sent events (SSE) or HTTP long-poll?

WebSocket gives us bidirectional in case we ever want client → server pushes (we don't yet, but it's free). It also stays open through corporate proxies that buffer SSE. Long-poll is a no-go — we have several thousand active workspaces simultaneously and connection churn would dominate.

---

## 5. Worker execution model

The worker-runtime service runs user-supplied Python in a restricted sandbox. This is the most security-sensitive component in the system.

```mermaid
sequenceDiagram
    participant U as User (browser)
    participant CO as core-api
    participant DB as PostgreSQL
    participant WR as worker-runtime
    participant RD as Redis
    participant PR as Cloud Run proxy
    participant EX as Exchange

    U->>CO: POST /workers/deploy<br/>(code, tick_interval, connected_blocks)
    CO->>DB: persist code + config
    CO->>WR: signal deploy

    U->>CO: POST /workers/{ws}/{node}/start
    CO->>WR: signal start

    loop Every tick_interval seconds
        WR->>WR: exec(code, sandbox_globals)
        WR->>RD: ctx.state.get/set
        WR->>RD: ctx.log.info (batched)
        Note over WR,EX: Exchange call paths
        WR->>PR: ctx.binance.get_ticker
        PR->>EX: HTTPS request
        EX-->>PR: response
        PR-->>WR: response
        WR->>RD: ctx.monitor.render → stream
    end

    Note over WR,RD: Auto-heal: 5 consecutive errors<br/>→ pause, optionally ask Claude to fix
```

### Sandbox boundaries

- `__import__` is replaced with a custom version that allows only a whitelist of stdlib modules plus `numpy` / `pandas`.
- `open`, `socket`, `urllib`, `subprocess`, `os`, `sys` are not in the builtins dict at all — `import os` raises an `ImportError` at import time.
- The `ctx` object is the only sanctioned escape hatch. Each adapter goes through a single `_ApiMixin` that requires a JWT and routes through the trusted backend.
- Each tick is wrapped in a 60 s timeout. Exceeding it raises `TickTimeoutError` and counts toward the auto-heal threshold.
- All log writes go through a batched writer that truncates and rate-limits per tick — no log-bomb DoS vector.

See [integrations.md](integrations.md) for the full `ctx.*` adapter list.

### State persistence

`ctx.state.get/set` writes to Redis. On worker startup, the runtime hydrates state from the last Postgres snapshot (saved on previous shutdown). On clean shutdown, state is snapshotted back to Postgres. This gives the worker:

- **Hot reads/writes** during execution (Redis at single-digit ms)
- **Durability** across container restarts (Postgres)
- **Survives full server reboots** without data loss

---

## 6. Egress proxy (Cloud Run)

A separate FastAPI service running on Cloud Run in `europe-west1`. Every outbound call from worker code (`ctx.http`, `ctx.binance`, `ctx.bybit`, etc.) is routed through it.

**Problem it solves.** When 100+ users share the same VPS IP and all hit Bybit's API with their own keys, the exchange's IP-concentration heuristics flag the IP as suspicious and rate-limit or temporarily block the entire IP. Without the proxy, one user's misbehaving worker degrades the platform for everyone.

**How it solves it.** Cloud Run instances scale across rotating egress IPs. The proxy passes the request through with HMAC-signed bodies preserved byte-for-byte (a `raw_body=True` flag forwards the raw bytes instead of re-serializing — critical for Bybit / OKX HMAC).

This is a **trade-off**, not a free win. Documented in [decisions.md → "Cloud Run egress proxy"](decisions.md#7-cloud-run-egress-proxy).

---

## 7. Authentication & secrets

| Concern | Mechanism |
|---|---|
| User passwords | bcrypt hash, never stored in plaintext |
| Session tokens | JWT (python-jose) with `uid`, `role`, `plan`; signed with HS256 |
| API keys / wallet creds / SIP secrets | Fernet authenticated-encryption at rest. The encryption key lives in env, never in DB. Decrypted just-in-time by the service that needs to make the upstream call |
| Browser→server | HTTPS only; Caddy auto-renews Let's Encrypt |
| Internal tokens for worker → backend | Short-lived JWTs minted by `create_internal_token(user_id)` per worker session. Workers never see user-facing JWTs |
| MCP endpoint | Standard JWT Bearer. Same auth surface as REST. Stateless (`stateless_http=True` in FastMCP) |

Secrets are never logged. The CI does not have access to production secrets — they live only on the VPS in a `.env` file outside the repo.

---

## 8. Frontend

Flutter Web in dark mode by default, served as a single-page app. Custom 20 000 × 20 000 px canvas built on top of `CustomPainter` for the edge rendering.

| Concern | Solution |
|---|---|
| State | Provider + `ChangeNotifier` (no Bloc / Riverpod / Redux — kept the dependency graph small) |
| Routing | GoRouter with auth-aware redirects |
| HTTP | Single `ApiClient` that injects Bearer token, retries idempotent requests once on 401 |
| Real-time | Single `EventsClient` per workspace, multi-listener pattern (each block registers callbacks by `nodeId`) — see [api-overview.md](api-overview.md) |
| Secure storage | `flutter_secure_storage` for the JWT |
| Builds | Flutter web release; assets cached aggressively, bootstrap files set to `no-cache` so deploys propagate immediately |

The frontend treats edges as first-class. When the user draws a Worker → Bybit edge, the frontend automatically patches the worker's `cfg.trading_node_id` field — no inspector dialog required. This is enforced by a small declarative rule table (`kEdgeWiringRules`).

---

## 9. CI / CD

```mermaid
flowchart LR
    Dev[git push origin main]
    GHA[GitHub Actions<br/>paths-filter computes<br/>SERVICES_TO_RESTART]

    subgraph Build["Build matrix · only changed services"]
        B1[core-api]
        B2[telephony]
        B3[worker-runtime]
        B4[realtime]
        B5[jarvis]
        B6[mailer]
    end

    GHCR[(ghcr.io)]
    SSH[SSH to VPS]
    Compose[docker compose pull && up -d]

    Dev --> GHA
    GHA --> Build
    Build --> GHCR
    GHCR --> SSH --> Compose
```

The deploy compose file lives at `/srv/deploy/docker-compose.yml` on the VPS. It pulls images from `ghcr.io/pinnacledynamics/aispinner/*`. CI computes which services changed using GitHub's `dorny/paths-filter` action, builds only those, and re-creates only those containers. A single-service deploy takes ~60 s end-to-end.

---

## 10. Observability

- **Plausible analytics** on landing & docs — privacy-first, cookieless, no third-party trackers
- **Structured logs** from every service to stdout, captured by Docker's json-file driver
- **Worker logs** — batched per-tick by the worker SDK, sent to `core-api` via internal endpoint, surfaced in the UI and queryable via `GET /workers/{ws}/{node}/logs`
- **Monitor stream** — `Redis stream monitor:events:{ws_id}:{node_id}` keeps the last N monitor renders for the frontend to lazy-load on Monitor block expansion
- **Health** — Caddy logs each upstream response code; container restarts are logged by docker

---

## What this repository does not show

The architecture is, of course, easier to describe than to build. If you're evaluating it for a hire — or for collaboration — see [CONTRIBUTING.md](../CONTRIBUTING.md) for how to get a private code walkthrough. The interesting bits are:

- The edge-wiring engine that propagates config changes across the graph in O(1) per edge
- The voice-translator state machine bridging two phone calls with two parallel OpenAI Realtime sessions and a shared glossary
- The dialer's chunked-campaign coordinator that survives container restarts via Redis-backed signals
- The Lightstreamer client wrapper for IG Markets — translating Lightstreamer's subscription model into the same `ctx.ig.*` API as the WebSocket exchanges

Each of these is between 500 and 1 500 lines of code. None of them is described in detail here, by design.
