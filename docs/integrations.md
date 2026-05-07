# Integrations

Breadth catalog: every block, every Worker SDK adapter, the telephony stack, MCP tools, and the WebSocket streaming tier. Use this to evaluate range and depth of integration work.

For the architectural choices behind these, see [architecture.md](architecture.md). For trade-offs, see [decisions.md](decisions.md).

---

## 1. Block catalog (45+ across 9 sections)

A block is the atomic unit on the canvas. Each has a type ID, a default config, a UI body, and edge wiring rules that auto-patch its config when connected to other blocks.

### Telephony — ElevenLabs (4)

| Block | Type ID | Capability |
|---|---|---|
| Phone Number | `ai.phone_number` | SIP trunk or Twilio — inbound & outbound voice |
| AI Agent | `ai.agent` | ElevenLabs conversational voice agent (prompt, voice, model) |
| PBX | `ai.pbx` | Virtual PBX — campaign auto-dialer, parallel calls, operator routing |
| Journal | `ai.journal` | Standalone call history viewer — transcripts, audio, CSV export |

### Telephony — Custom Voice (5, all PRO)

| Block | Type ID | Capability |
|---|---|---|
| Custom Agent | `voice.custom_agent` | Own STT/LLM/TTS pipeline (Deepgram + DeepSeek/OpenAI + Cartesia) |
| Call Guard | `ai.call_guard` | AI call monitor — voicemail / silence / rudeness detection |
| Voice Translator | `voice.translator` | Real-time bidirectional speech translation between two callers via OpenAI Realtime |
| SIP / ARI Trunk | `telephony.trunk` | Asterisk SIP trunk configuration |
| PBX (ARI) | `ais.pbx` | Virtual PBX dialing through Custom Agent + Asterisk |

### AI Agents (5)

| Block | Type ID | Provider |
|---|---|---|
| ChatGPT | `ai.llm_agent` | OpenAI GPT models |
| Claude Agent | `ai.claude_agent` | Anthropic Claude — vision, reasoning, automation |
| Grok Agent | `ai.grok_agent` | xAI Grok — X/Twitter data |
| DeepSeek | `ai.deepseek_agent` | DeepSeek V3 + R1 reasoning |
| Groq | `ai.groq_agent` | Groq — ultra-fast Llama, Mixtral, Gemma inference |

All AI Agent blocks share a hub-style config: they accept connections from messengers, trading, data, files, and more, exposing those capabilities as tools to the underlying LLM.

### Automation (3)

| Block | Type ID | Use |
|---|---|---|
| Worker | `code.worker` | Python — runs in browser sandbox or on the server |
| Scheduler | `automation.scheduler` | Trigger connected blocks on an interval (cron-style) |
| Logic Gate | `automation.logic_gate` | Conditional routing — IF/THEN/ELSE for events |

### Trading — Crypto (8)

| Block | Type ID | Plan | Notes |
|---|---|---|---|
| Bybit | `trading.bybit` | free | Spot + derivatives, 26 methods |
| Binance | `trading.binance` | free | Spot + futures, 31 methods, WebSocket-cached |
| Coinbase | `trading.coinbase` | pro · server-only | Advanced Trade API |
| OKX | `trading.okx` | pro · server-only | Spot, swaps, futures |
| Kraken | `trading.kraken` | pro · server-only | Balance, orders, trading |
| Deribit | `trading.deribit` | pro · server-only | Crypto options, futures, perpetuals — 33 methods |
| Hyperliquid | `trading.hyperliquid` | pro | On-chain perpetuals DEX, EIP-712 wallet signing |
| Bitget | `trading.bitget` | pro | Spot + USDT-FUTURES, copy-trading |

### Trading — Traditional (2)

| Block | Type ID | Plan | Notes |
|---|---|---|---|
| IG Markets | `trading.ig` | pro · server-only | CFDs, forex, indices, commodities — Lightstreamer |
| Interactive Brokers | `trading.ibkr` | free | Stocks, options, futures, forex — Gateway / TWS mode |

### Prediction Markets (2)

| Block | Type ID | Plan |
|---|---|---|
| Polymarket Monitor | `prediction.polymarket_monitor` | free |
| Polymarket Trader | `prediction.polymarket_trader` | pro |

### Data Feeds (3)

| Block | Type ID | What it provides |
|---|---|---|
| CoinGecko | `data.coingecko` | Crypto market data — prices, volumes, trends |
| Twelve Data | `data.twelve_data` | Stocks, forex, indices, commodities |
| Apify | `data.apify` | Universal web scraping — 3 000+ actors |

### Messengers (7)

| Block | Type ID | Subgroup |
|---|---|---|
| Telegram | `messenger.telegram` | Telegram (Bot API) |
| Telegram Bot (AI) | `ai.telegram_bot` | Telegram (LLM-powered responder) |
| Telegram Channel | `messenger.telegram_channel` | Telegram (auto-publishing) |
| Telegram Reader | `messenger.telegram_reader` | Telegram (Telethon — user account) |
| Discord | `messenger.discord` | Discord |
| WhatsApp | `messenger.whatsapp` | WhatsApp Business — Meta Cloud API |
| Matrix Messenger | `messenger.matrix` | Matrix (VoIP + chat) |

### Storage (1)

| Block | Type ID | Capability |
|---|---|---|
| File Explorer | `storage.file_explorer` | Sandboxed file manager — 100 MB per workspace |

### UI / Monitoring (3)

| Block | Type ID | Renders data from |
|---|---|---|
| Worker Monitor | `ui.monitor` | `ctx.monitor.render(...)` widgets |
| PBX Monitor | `ui.pbx_monitor` | PBX call journal + live calls |
| Matrix Monitor | `ui.matrix_monitor` | Matrix call/chat events |

---

## 2. Worker SDK — `ctx.*` adapters

The 16-method object injected into every Python worker. Each adapter requires the corresponding block to be connected via an edge — otherwise calling its methods raises `RuntimeError`.

| Adapter | Methods | What it provides |
|---|---|---|
| `ctx.state` | 5 | Persistent KV store backed by Redis + Postgres snapshot |
| `ctx.log` | 5 | Batched info / warn / error / debug logs |
| `ctx.monitor` | 4 + builders | Stream widgets to a connected Monitor block |
| `ctx.http` | 3 | GET / POST / fetch through Cloud Run egress proxy |
| `ctx.bybit` | 26 | Bybit — spot + derivatives, WebSocket-cached |
| `ctx.binance` | 31 | Binance — spot + futures, WebSocket-cached |
| `ctx.coinbase` | 12 | Coinbase Advanced Trade |
| `ctx.okx` | 15 | OKX — spot, swaps, futures |
| `ctx.kraken` | 12 | Kraken |
| `ctx.deribit` | 33 | Deribit — options, futures, perps, vol indices |
| `ctx.hyperliquid` | — | On-chain perpetuals DEX with EIP-712 signing |
| `ctx.bitget` | — | Bitget — spot + perpetuals |
| `ctx.ibkr` | 7 | Interactive Brokers Web API (Gateway mode) |
| `ctx.ig` | 9 REST + 5 streaming | IG Markets — CFD, forex, indices via Lightstreamer |
| `ctx.polymarket` | 9 | Polymarket — prediction markets |
| `ctx.telegram` | 3 | Send / read / chat info via Telegram Bot |
| `ctx.files` | 4 | Read / write / list / delete in sandboxed FS |
| `ctx.llm` | `.ask` + `.claude` / `.grok` / `.deepseek` / `.groq` + `.ask_with_tools` | Multi-provider LLM routing — pick by name |
| `ctx.connected_blocks` | (read-only) | Map showing which blocks are wired to this worker |

### Multi-LLM routing

```python
def tick(ctx):
    # Use the first connected LLM block
    answer = ctx.llm.ask("Summarize current BTC sentiment.", system="You are a crypto analyst.")

    # Force a specific provider when multiple are connected
    fast = ctx.llm.groq("Classify this text: ...")           # Groq — sub-second
    deep = ctx.llm.claude("Plan a 5-step trading strategy")  # Claude — reasoning
    cheap = ctx.llm.deepseek("Translate to Russian")          # DeepSeek — cost
```

This lets a worker stitch together "fast classifier → deep reasoner → cheap translator" without thinking about API keys, billing, or vendor SDKs.

### WebSocket-cached methods

In Server execution mode, the runtime opens long-lived WS connections to supported exchanges. Cached methods return from in-memory state in **0 ms** instead of 80–250 ms via REST, with no rate-limit ceiling.

| Exchange | Cached methods |
|---|---|
| Bybit | 4/5 — ticker, balance, positions, orders |
| Binance | 3 — ticker, account, positions |
| Deribit | 4/4 — ticker, account summary, positions, orders |
| OKX | ticker, orderbook, positions, orders, balance |
| Coinbase | ticker + orders |
| Kraken | ticker, orderbook, orders, balance |
| IG Markets | Lightstreamer — ACCOUNT (0 ms) ; PRICE/CHART require account-level streaming permissions |

REST fallback is automatic in Browser mode (no WS connection available).

---

## 3. MCP server — 13 tools

A FastMCP server at `https://api.aispinner.io/mcp-api/mcp` lets any MCP-compatible AI (Claude Desktop, Cursor, ChatGPT) read and modify a user's workspaces. Authenticated via the same JWT as the REST API. Stateless (`stateless_http=True`).

| Category | Tool | What it does |
|---|---|---|
| **Read** | `get_workspace_graph` | Full graph JSON for a workspace |
| | `get_node_config` | Per-node config |
| | `list_block_types` | All available block types with their default configs |
| | `get_worker_logs` | Recent logs for a worker |
| | `get_call_history` | Call journal for a workspace / node |
| **Mutate graph** | `add_node` | Add a block of a given type at coords |
| | `remove_node` | Remove a block (and its edges) |
| | `connect_nodes` | Draw an edge — auto-patches edge wiring |
| | `disconnect_nodes` | Remove an edge — undoes wiring |
| | `rename_node` | Rename a block |
| | `update_node_config` | Patch a block's config |
| **Operations** | `restart_worker` | Stop & start a worker |
| | `start_dialer` | Launch a PBX campaign |

All graph mutations go through a single `_modify_graph()` helper that:

1. Reads the latest `workspace_documents` row inside a transaction
2. Applies the change
3. Increments `version`
4. Writes back

This keeps the AI assistant's mutations on the same optimistic-locking pathway as a human user clicking buttons in the canvas — there's no separate code path with weaker invariants.

---

## 4. Telephony stack

```mermaid
flowchart LR
    Caller[Inbound caller] --> Carrier[SIP / Twilio carrier]
    Carrier --> EL[ElevenLabs Conversational AI]
    EL <--> Phone[Phone Number block]
    Phone <--> Agent[AI Agent block]

    PBX[PBX block] -->|chunked campaign| EL
    EL -->|on call end| Webhook[/integrations/sip/post-call/]
    Webhook --> DB[(telephony_runs<br/>+ transcript)]

    Translator[Voice Translator block] -->|2 calls| OAR[OpenAI Realtime ×2]
    OAR -->|translated audio| Translator
    Translator --> Carrier

    DB --> Journal[Journal block]
    DB --> Monitor[PBX Monitor block]
```

### What's interesting under the hood

- **Adopt existing ElevenLabs agents.** `POST /agents/adopt` pulls full configuration (voice, prompt, tools, model, voice settings, temperature, max_tokens, platform tools, safety rules, raw config) into a local AiSpinner record without creating a new agent on ElevenLabs side. Lets users bring already-configured agents onto the canvas non-destructively.

- **Chunked campaign dialer.** A campaign with 1 000 numbers and `max_concurrent: 5` splits into 200 chunks of 5. Each chunk submits a `submit_batch_call()` and waits. Pause/resume/stop is implemented via Redis flags polled at chunk boundaries; the campaign state survives container restarts. After completion: a 60 s wait + 3 retry passes resolves all conversations (transcripts, summaries) before marking the campaign done.

- **Auto-resolve stale campaigns.** When the journal is opened, the backend scans for `batch_running` / `completing` runs whose ElevenLabs batch is actually finished. It resolves one per request (limit to avoid pool exhaustion) and busts the journal cache.

- **Voice Translator.** Bridges two phone lines (often two SignalWire `<Stream>` PSTN sockets). Each side runs its own OpenAI Realtime session. Audio from caller A → ASR → MT (with shared glossary) → TTS → caller B's audio output, and vice versa. The state machine handles connection drops, disagreement on hangup, and end-of-utterance detection.

- **Post-call webhooks.** `ai.postcall_webhook` block receives the full ElevenLabs payload (transcript, summary, tool calls, evaluation criteria) and forwards it to the user's destination URL.

---

## 5. AI integrations

Beyond the LLM agent blocks, AI is woven through the platform:

| Integration | Where | What |
|---|---|---|
| **Jarvis** | Inside the app | Built-in AI assistant that uses the same `ai_tools` registry as MCP |
| **MCP server** | `/mcp-api/mcp` | 13 tools — see above |
| **Worker auto-heal** | `worker-runtime` | When a worker errors out 5 ticks in a row, optionally calls Claude with the error + code, applies the fix, restarts |
| **AI Agent blocks** | LLM blocks | Worker code can call them as tools via `ctx.llm.*` |
| **ElevenLabs Conversational AI** | Telephony | Real-time voice agents over phone |
| **OpenAI Realtime** | Voice Translator | Bidirectional speech translation |
| **Custom Voice pipeline** | `voice.custom_agent` | Deepgram (STT) + DeepSeek/OpenAI (LLM) + Cartesia (TTS) — own STT/LLM/TTS chain over Asterisk SIP/ARI |
| **AI-assisted code writing** | Worker block | Inline "ask AI to write this" while editing worker code |

---

## 6. Frontend integrations

Everything user-facing reflects the same edge-as-config model. A short list:

- **Audio playback** in the Journal block uses an HTML5 `<audio>` element with token-in-URL — no extra API for the player. Frontend tracks `activeSessionId` so opening a second audio kills the first.
- **CSV export** runs entirely in the browser — no server round-trip for exporting a 5 000-call campaign.
- **Worker code editor** is a custom Flutter widget with syntax-highlighting; deploy + start are JWT-authenticated calls to `core-api`.
- **Live monitor** widgets are rendered from a JSON list pushed via WebSocket — `metric`, `status`, `table`, `text`, `progress`, `list`, `log`, `separator`. The renderer is dumb on purpose; all logic lives in the worker.
- **Block inspector** dialogs are generated from a declarative schema (`BlockConfigField[]`) — adding a new config field to a block doesn't require new UI code.

---

## What this catalog implies for the engineering work

- **45 blocks × an average of ~700 lines each** in `packages/blocks/lib/src/blocks/` — the biggest single file is `journal_block_body.dart` at ~2 000 lines covering CSV export, audio player, campaign grouping, filters.
- **3 350 lines** in `telephony/.../telephony_calls.py` — calls, journal, dialer, runs, batch resolve, audio bridging, all in one router because they share a transaction model and a cache-invalidation strategy.
- **2 400 lines** in `worker-runtime/.../worker_sandbox.py` — 16 adapters + sandbox bootstrap + log batcher.
- **600 lines** in `core-api/.../mcp_tools.py` — 13 MCP tools, each one a thin wrapper over the corresponding REST endpoint to keep behaviour in lock-step.

The breadth above is real because there's a small set of tight conventions (edge wiring rules, `_ApiMixin`, shared schemas, `_modify_graph` helper). New integrations slot in by following them, not by spawning bespoke code paths.
