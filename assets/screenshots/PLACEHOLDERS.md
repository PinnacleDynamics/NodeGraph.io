# Screenshots — what to capture

The README references the following screenshots. Recommended size **1280×720** (or the same aspect — 16:9). PNG. Replace the empty placeholders in this folder.

| Filename | What to capture | Where in the README |
|---|---|---|
| `canvas-overview.png` | A workspace canvas with **5–6 blocks connected with edges**: Worker → external API, Worker → Telegram, Worker → Monitor, AI Agent → Phone Number. Showcase the visual editor at its busiest. | "What it is" section |
| `edge-wiring.png` | A **close-up of an edge being drawn** between two blocks. Optionally show the inspector dialog open with the auto-patched config field highlighted. | "Edges replace configuration" feature |
| `worker-code.png` | The **Worker block fully expanded** with: Code tab open showing real Python, Deploy / Start buttons visible, status dot showing "running". | "24/7 sandboxed Python workers" feature |
| `monitor.png` | The **Monitor block rendering live data** — at minimum show: 2-3 metric cards, 1 status indicator, 1 progress bar, 1 small table. Best if it's clearly tied to a Worker on the same canvas. | "Sub-millisecond live data" feature |
| `pbx-journal.png` | **Two blocks side by side**: PBX block with a campaign running (parallel calls visible), and the Journal block with a few completed calls expanded showing transcripts. | "Voice agents and PBX" feature |

## Optional bonus screenshots (not referenced yet, can add to docs later)

| Filename | Suggested capture |
|---|---|
| `voice-translator.png` | Voice Translator block with two phone number connections + active session indicator |
| `mcp-claude.png` | Claude Desktop UI calling an AiSpinner MCP tool (e.g. `add_node` or `get_workspace_graph`) |
| `streaming-monitor.png` | Worker Monitor showing live data from 3+ external APIs side by side |
| `landing-hero.png` | Top of aispinner.io showing the canvas demo |
| `pricing.png` | The pricing section with all 3 plans visible |

## Tips

- Use the **dark theme** (it's the default, and matches the rest of the site).
- **Hide your real email** — sign in as a fake demo user or scrub the avatar / sidebar.
- **No real API keys visible** — even partially. Cover the integrations sidebar if needed.
- For canvas screenshots — **zoom out** to fit several blocks, then crop tight to the content.
- For editor screenshots — make sure the code is **production-quality real code**, not "Hello world".

Replace this `PLACEHOLDERS.md` with images using the same filenames, and the README will pick them up automatically. You can leave this file in place — GitHub will simply ignore it once images exist.
