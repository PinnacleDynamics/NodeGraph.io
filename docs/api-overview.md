# API Overview

A summary of the public surfaces a client (web, mobile, AI assistant, automation script) can talk to.

The full reference for each surface lives at [docs.aispinner.io](https://docs.aispinner.io). This page exists so an evaluator can see the shape and depth of the API without leaving the repo.

| Surface | Where | Auth |
|---|---|---|
| REST | `https://api.aispinner.io` | JWT Bearer |
| WebSocket events | `wss://api.aispinner.io/ws/events/{ws_id}?token=<JWT>` | JWT in query string |
| MCP server | `https://api.aispinner.io/mcp-api/mcp` | JWT Bearer (FastMCP) |

---

## 1. REST

### Authentication
- `POST /login` → `TokenOut`
- `POST /register/start` → email confirmation code sent
- `POST /register/confirm` → `TokenOut`
- `POST /password/reset/start` / `/confirm`
- `POST /change-password`
- `GET /me`

The login → register flow uses email-code confirmation with a short-lived code stored in Redis. JWTs include `uid`, `role`, `plan`, and `country`. Country is derived once on first login from the client's IP via DB-IP's free `mmdb` and stored on the user row — used for plan eligibility and to surface country-specific restrictions in the catalog UI.

### Workspaces
- `GET /workspaces` — list
- `GET /workspaces/me` — current workspace + graph in one call (used by the canvas on first paint)
- `POST /workspaces`
- `GET /workspaces/{id}`
- `PUT /workspaces/{id}` — rename
- `GET /workspaces/{id}/graph`
- `PUT /workspaces/{id}/graph` — **optimistic locking** via `version`. Mismatch → `409 Conflict`.

### Node configs (sidecar)
- `GET /workspaces/{ws_id}/nodes/{node_id}/config`
- `PUT /workspaces/{ws_id}/nodes/{node_id}/config`

A separate table from `graph_json` so per-node changes (e.g. PBX operator list edits) don't bump the graph version.

### Workers
- `POST /workers/deploy` — body includes code, tick interval, max errors, auto-heal flag, connected blocks summary
- `POST /workers/{ws_id}/{node_id}/start` / `/stop`
- `GET /workers/{ws_id}/{node_id}/status`
- `GET /workers/{ws_id}/{node_id}/logs?limit=100&level=info`
- `PATCH /workers/{ws_id}/{node_id}/settings` — change `tick_interval`, `max_errors`, `auto_heal` without redeploy

### Telephony
- `POST /telephony/dialer/start`
- `GET /telephony/runs?workspace_id=&node_id=&limit=`
- `GET /telephony/runs/resumable`
- `PATCH /telephony/runs/{call_id}/name`
- `GET /telephony/calls/{call_id}/events`
- `GET /telephony/calls/{call_id}/audio?token=<JWT>` — MP3, token in query string so `<audio>` players can use the URL directly
- `GET /telephony/journal` — all-nodes journal with optional filters
- `GET /telephony/journal/{node_id}` — per-node journal
- `GET /telephony/journal/call/{call_id}` — single call detail
- `GET /telephony/settings` / `PUT /telephony/settings`

#### Delta pagination

The journal supports `?since_id=<max_id>`. Combined with the WebSocket `journal.new` event, the frontend keeps the journal current with a single full load + cheap incremental refreshes.

```
1. UI mounts → GET /telephony/journal/{node_id} → returns calls, max_id=12345
2. WS pushes journal.new
3. UI calls GET /telephony/journal/{node_id}?since_id=12345 → returns only newer rows + new max_id
4. Repeat
```

Cache for full loads only (never deltas): `journal_cache:{md5(...)}` with 300 s TTL. Invalidated on every call status change and on auto-resolve of stale campaigns.

### Phone provider configuration

Per-node carrier configuration for `ai.phone_number` blocks.

- `GET /workspaces/{ws_id}/nodes/{node_id}/phone-provider`
- `PUT /workspaces/{ws_id}/nodes/{node_id}/phone-provider` — Twilio (`twilio_sid`, `twilio_token`, `twilio_phone`) or SIP (`sip_server`, `sip_user`, `sip_password`, `sip_port`); secrets are encrypted at rest with Fernet
- `DELETE /workspaces/{ws_id}/nodes/{node_id}/phone-provider`
- `POST /workspaces/{ws_id}/nodes/{node_id}/phone-provider/import` — pulls existing SIP/Twilio configuration from the user's linked ElevenLabs account

### ElevenLabs integration
- `GET /integrations/elevenlabs/keys` / `POST` / `DELETE`
- `GET /integrations/sip/phone-numbers` — every phone number visible in the user's ElevenLabs account
- `GET /agents/elevenlabs/list`
- `POST /agents/adopt` — pull an existing ElevenLabs agent's full config locally without creating a new agent
- `POST /integrations/elevenlabs/agents/test`

### Errors

```json
{ "detail": "..." }
```

| Status | Meaning |
|---|---|
| 401 | Missing / invalid JWT |
| 403 | Plan-level forbidden (e.g. server-only block on Free plan) |
| 404 | Resource not found *or* not owned by the requesting user — returned identically to avoid existence leaks |
| 409 | Optimistic-locking version conflict on graph save |
| 422 | Pydantic validation error |
| 429 | Upstream rate-limit hit (e.g. ElevenLabs); the system retries automatically and only surfaces 429 when the user's account quota is exhausted |

---

## 2. WebSocket events

Single connection per workspace. Multi-listener pattern on the client: each block registers callbacks by `nodeId` so several blocks listen simultaneously without re-opening sockets.

```
wss://api.aispinner.io/ws/events/{workspace_id}?token=<JWT>
```

### Event types

| Type | Publisher | Frontend reaction |
|---|---|---|
| `journal.new` | telephony | Journal block fetches delta via REST |
| `call.done` / `call.transcript` | telephony | PBX Monitor updates a row |
| `batch.status` | telephony | Campaign progress bar tick |
| `campaign.started` / `campaign.stopped` | telephony | Refresh campaign list |
| `graph.version` | core-api | Refresh graph if a different tab / collaborator saved |
| `subscribed` / `pong` | realtime | Connection management |

Plus pass-through telephony events (DTMF, voicemail-detected, interruption signals) — not enumerated, forwarded as-is.

### Subscription model

The connection is **auto-subscribed to the workspace channel**. Clients can opt into additional channels:

```json
{ "action": "subscribe", "channel": "calls:pbx_xyz" }
```

The server replies `{"type": "subscribed", "channel": "..."}`.

### Reconnect

Clients implement exponential backoff (1 s, 2 s, 4 s … max 30 s). After reconnecting they fetch the latest graph version (in case events were missed) and journal deltas (`since_id=<last_seen>`).

---

## 3. MCP server

A FastMCP server with `stateless_http=True` so each tool call is independent — no per-session memory. Same JWT as REST.

13 tools exposed (see [integrations.md → MCP server](integrations.md#3-mcp-server--13-tools) for the full list).

### Why MCP

When integrating an AI assistant with a SaaS, there are roughly three options:

1. **Custom function-calling adapter per assistant** — N adapters for N assistants
2. **A "build a tool with our REST" doc and let users wire it themselves** — friction, error-prone
3. **MCP** — one server, every MCP-compatible assistant can use it immediately

Choosing #3 was a bet that MCP would become the standard. As of 2026 it has — Claude Desktop, Cursor, ChatGPT (custom GPTs), and several assistants ship MCP support natively. The same JWT works for the user's REST calls, app session, and AI assistant. No separate auth flow.

### How tools stay honest

Each MCP tool is a thin wrapper over the corresponding REST endpoint. They share validation, authorization, optimistic locking, and audit logging. Adding a new MCP tool means picking which REST endpoint to expose — never duplicating business logic. This was an explicit design choice — see [decisions.md → "MCP wraps REST, not the other way round"](decisions.md#10-mcp-wraps-rest-not-the-other-way-round).

---

## 4. Auth quick reference

```
Browser / app → JWT in Authorization header
                or in query string for `<audio>` and WS upgrades

Worker code → never sees user-facing JWTs
              ctx.* methods are authenticated transparently by the SDK
              with short-lived per-session tokens scoped to the user

AI assistant → JWT Bearer to /mcp-api/mcp
                same JWT, same authz, same audit trail as REST
```

The principle: **one auth mechanism, four entry points, zero special cases**.
