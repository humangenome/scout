# Scout: AI Voice Assistant Powered by Claude

## Vision

Scout is an **AI voice assistant** that replaces Siri, Google Assistant, and Alexa — powered by Claude Code instead of proprietary AI. It controls your smart home, answers questions, manages your schedule, and runs agentic coding tasks — all through natural voice conversation.

Three clients, one server:

| Client | Mode | Voice | Target use |
|--------|------|-------|------------|
| **Terminal CLI** | Text only | No | Continue conversations from your desktop. Free. Like `claude` CLI with session management. |
| **Phone app** | Conversational + voice | Yes (wake word, STT, TTS) | On-the-go assistant. Replaces Siri / Google Assistant. |
| **Tablet app** | Hub + voice + ambient | Yes (always-listening, idle screen, buttons) | Smart home hub. Replaces Alexa / Google Home. |

All three connect to the **same session pool** on the server. Start a conversation on your phone, pick it up on your desktop, see results on the tablet.

**APIs**: OpenAI Whisper (STT), OpenAI TTS (TTS), Claude Code (agentic AI).
**Business model**: Open source core (trust) + managed cloud hosting (revenue) + premium app features (revenue).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Scout Server                             │
│                   (Node.js on your box)                         │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Session Mgr  │  │ Audio Pipeline│  │ PTY Manager           │  │
│  │ • create     │  │ • Whisper STT │  │ • spawn claude CLI    │  │
│  │ • resume     │  │ • OpenAI TTS  │  │ • I/O forwarding      │  │
│  │ • timeout    │  │ • wake word   │  │ • auto-restart        │  │
│  │ • metadata   │  │   (hybrid)    │  │ • resize              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
│         │                 │                      │               │
│         └─────────────────┼──────────────────────┘               │
│                           │                                      │
│                    WebSocket Server                               │
│                    (Express + ws)                                 │
└───────────────────────────┼──────────────────────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              │             │              │
     ┌────────▼───┐  ┌─────▼─────┐  ┌─────▼──────┐
     │ Terminal    │  │ Phone App │  │ Tablet App  │
     │ CLI (text)  │  │ (voice)   │  │ (hub mode)  │
     │             │  │           │  │             │
     │ scout start │  │ "Hey      │  │ Always-on   │
     │ scout ls    │  │  Scout"   │  │ ambient     │
     │ scout resume│  │           │  │ display     │
     └─────────────┘  └───────────┘  └─────────────┘
```

---

## Session Management

Scout owns the session lifecycle. Each session wraps a Claude Code `--resume` session ID with Scout metadata.

### Smart Session Routing (timeout-based)

- **Active session** (idle < timeout): Next query continues the same session.
- **Stale session** (idle > timeout): Next query starts a fresh session. Default timeout: configurable (e.g. 15 min).
- **Manual override**: "Hey scout, new session" / "Hey scout, continue last session" (voice commands) or New/Continue buttons in UI.

### Session Pool

```
Scout Session DB (JSON/SQLite)
├── session-001: { claude_id: "abc-123", name: "Refactor auth", created: ..., last_active: ..., project: "scout" }
├── session-002: { claude_id: "def-456", name: "Weather check", created: ..., last_active: ..., project: null }
└── session-003: { claude_id: "ghi-789", name: "HA automation", created: ..., last_active: ..., project: "homeassistant" }
```

All clients see the same pool. Terminal CLI, phone app, tablet — all can resume any session.

---

## Three Clients

### 1. Terminal CLI (`scout` command)

Thin wrapper around Claude CLI with session management. No voice. Free.

```bash
scout start                    # New session (spawns claude --dangerously-skip-permissions)
scout ls                       # List all sessions (like dev ls)
scout resume 3                 # Resume session 3
scout resume                   # Resume most recent active session
scout stop                     # Stop current session
scout server start|stop|status # Manage the Scout server
```

Feels like native `claude` CLI but with Scout's session management, auto-resume, and multi-device continuity.

### 2. Phone App (native — Android + iOS)

Conversational voice assistant. Replaces Siri / Google Assistant.

- **Hybrid wake word**: On-device detection (WASM/TFLite in client), server-side STT (Whisper)
- **Configurable wake word**: Users set their own wake phrase
- **Visual activation**: Screen indicator when wake word detected (no chime by default)
- **Smart TTS**: Context-aware — speaks responses when input was voice, silent when input was text
- **Smart output**: Plain text is spoken; for code/technical output says "I wrote the code, check the terminal"
- **User-configurable voice**: Pick from OpenAI TTS voices (nova, onyx, echo, alloy, fable, shimmer)
- **Background listening**: Native app can listen for wake word even when backgrounded/screen off
- **Session continuity**: Same session pool as terminal and tablet

### 3. Tablet App (web-based — kiosk/hub mode)

Always-on smart home hub. Replaces Alexa Show / Google Home Hub.

- Everything the phone app has, plus:
- **Always-listening**: Continuous wake word detection
- **Idle screen**: After configurable timeout → ambient display (clock, weather, solar — all from HA REST API, zero token usage)
- **Button bar**: Quick-tap commands from `config/buttons.json` (JSON file config only)
- **xterm.js terminal**: Full Claude Code terminal visible on screen
- **Multi-client shared session**: Multiple browsers can connect to the same PTY — all see and can type
- **Auto-restart**: If Claude process crashes, auto-respawn

---

## Design

**Clean light + dark mode**. Modern, polished consumer feel (like Apple apps). Supports both light and dark themes. Touch-friendly, responsive from phone to tablet to desktop.

---

## Project Structure

```
~/projects/scout/
├── package.json
├── tsconfig.json
├── .env
├── src/                           # Server (TypeScript → dist/)
│   ├── server.ts                  # Express + WebSocket + orchestration
│   ├── pty-manager.ts             # node-pty: spawn claude, I/O, resize, auto-restart
│   ├── session-manager.ts         # Session lifecycle: create, resume, timeout, metadata
│   ├── audio-pipeline.ts          # PCM buffering → WAV → Whisper API → text
│   ├── tts-speaker.ts             # Text → OpenAI TTS → MP3 chunks → client
│   ├── output-parser.ts           # Parse Claude output, extract speakable text, smart detection
│   ├── ambient-data.ts            # HA REST API → weather, solar, time (zero tokens)
│   └── config.ts                  # Env vars, constants
├── cli/                           # Terminal CLI (`scout` command)
│   └── scout.ts                   # CLI entry point: start, ls, resume, stop, server
├── public/                        # Web client (served statically)
│   ├── index.html                 # App shell: header + terminal + footer
│   ├── manifest.json              # PWA manifest
│   ├── scout.css                  # Light/dark theme, responsive, touch-friendly
│   ├── scout.js                   # WebSocket, state, view switching
│   ├── terminal.js                # xterm.js + WebSocket bridge
│   ├── audio.js                   # Mic → AudioWorklet → WebSocket
│   ├── pcm-processor.js           # AudioWorklet: 16kHz mono PCM
│   ├── wake-word.js               # Client-side wake word detection (WASM/TFLite)
│   ├── idle-screen.js             # Ambient display (clock, weather, solar from HA)
│   ├── buttons.js                 # Button bar from config
│   ├── icons/
│   └── sounds/
│       └── done.mp3               # Completion chime
├── app/                           # Native app (React Native / Expo — future)
│   └── ...
├── config/
│   └── buttons.json               # Button definitions (JSON file only)
└── scripts/
    └── setup-tablet.md            # Kiosk setup guide
```

---

## Phased Build Order

### Phase 1: Server + Web Terminal
**Goal**: Scout server with xterm.js in the browser. Type commands, see Claude respond. Multi-client shared session.

- `src/server.ts` — Express serves public/, WebSocket bridges to PTY, supports multiple clients
- `src/pty-manager.ts` — Spawn `claude --dangerously-skip-permissions`, forward I/O, resize, auto-restart on crash
- `src/config.ts` — Env vars, port, project dir
- `public/index.html` — Layout: header bar + terminal + footer bar (scaffold for future phases)
- `public/terminal.js` — xterm.js init, WebSocket, keyboard forwarding
- `public/scout.js` — WebSocket connection manager, view state
- `public/scout.css` — Light/dark theme, responsive (phone → tablet → desktop)

**Test**: Open `http://localhost:3333` in multiple browsers → all see same terminal → type → Claude responds.

### Phase 2: Session Management
**Goal**: Scout CLI + session lifecycle. Create, list, resume, timeout-based routing.

- `src/session-manager.ts` — Session DB (JSON), create/resume/list/timeout logic
- `cli/scout.ts` — `scout start`, `scout ls`, `scout resume`, `scout stop`
- Server WebSocket extended: session-aware routing
- Timeout-based auto-new-session (configurable)
- Manual override: "new session" / "continue" via UI buttons

**Test**: `scout start` → type → wait past timeout → new query starts fresh session → `scout ls` shows both → `scout resume 1` picks up old session.

### Phase 3: Voice Input (Push-to-Talk + Wake Word)
**Goal**: Tap mic or say "hey scout" to speak. Transcription sent to Claude.

- `src/audio-pipeline.ts` — Accumulate PCM, build WAV, Whisper API → text → raw paste to PTY stdin
- `public/audio.js` — getUserMedia, AudioWorklet, stream PCM to server
- `public/pcm-processor.js` — AudioWorklet: 16kHz mono PCM
- `public/wake-word.js` — Client-side wake word detection (hybrid: detect on client, STT on server)
- Configurable wake word
- Visual-only activation (screen indicator, no chime)
- After wake word → buffer audio until silence (VAD) → Whisper → PTY

**Test**: Tap mic → speak → release → text appears → Claude responds. Say "hey scout, what time is it" → visual indicator → transcription → response.

### Phase 4: TTS Voice Output
**Goal**: Claude's responses spoken aloud. Smart detection — only speak conversational text.

- `src/tts-speaker.ts` — Text → OpenAI TTS API → stream MP3 chunks to client
- `src/output-parser.ts` — Detect response completion (2s silence heuristic), extract speakable text, skip ANSI/code/tool calls. For code output: "I wrote the code, check the terminal."
- Context-aware TTS: on when input was voice, off when input was keyboard
- User-configurable voice selection

**Test**: Voice query → Claude responds → hear spoken response. Type query → Claude responds → no audio.

### Phase 5: Button Bar + Idle Screen
**Goal**: Quick-tap buttons. Ambient display on idle.

- `config/buttons.json` — Button definitions (JSON file only)
- `public/buttons.js` — Render buttons, tap sends command to PTY
- `src/ambient-data.ts` — HA REST API → weather, solar, time (zero token usage)
- `public/idle-screen.js` — Ambient display after configurable timeout

**Test**: Tap "Weather" → response. Wait past idle timeout → ambient screen → tap → back to terminal.

### Phase 6: PWA + HTTPS
**Goal**: Installable on phones/tablets. Full-screen app experience.

- `public/manifest.json` — PWA manifest
- `public/icons/` — App icons
- HTTPS: document both Tailscale Funnel and Nginx/Caddy reverse proxy options
- Online only (no offline/service worker caching)

**Test**: Phone → "Add to Home Screen" → launch → full-screen Scout.

### Phase 7: Terminal CLI
**Goal**: `scout` command for desktop/terminal use.

- `cli/scout.ts` — Full CLI: start, ls, resume, stop, server management
- Connects to same session pool as phone/tablet
- npm bin / global install

**Test**: `scout start` → Claude session → `scout ls` → `scout resume` from another terminal.

### Phase 8: Native Companion Apps
**Goal**: Native Android + iOS with background wake word. Priority feature.

- React Native (Expo) — one codebase, both platforms
- Same WebSocket protocol as web
- Picovoice Porcupine or similar for on-device background wake word
- Push notifications when Scout completes long tasks
- Home screen widgets
- Background listening even with screen off

**Test**: Install app → "hey scout" with screen off → wakes up → voice query → spoken response.

---

## WebSocket Protocol

### Client → Server
| type | fields | description |
|------|--------|-------------|
| `terminal-input` | `data: string` | Raw keystrokes |
| `audio-start` | | Begin voice recording |
| `audio-chunk` | `data: string` (base64 PCM) | Audio data (~100ms chunks) |
| `audio-stop` | | End recording, trigger STT |
| `command` | `text: string` | Button command / voice override ("new session") |
| `resize` | `cols, rows` | Terminal resize |
| `session-list` | | Request session list |
| `session-resume` | `id: string` | Resume a specific session |
| `session-new` | | Force new session |

### Server → Client
| type | fields | description |
|------|--------|-------------|
| `terminal-output` | `data: string` | PTY output (ANSI) |
| `status` | `state: string` | listening / transcribing / thinking / speaking / idle |
| `transcription` | `text: string` | Whisper result |
| `tts-audio` | `data: string` (base64 MP3), `done: bool` | TTS audio chunk |
| `ambient` | `{weather, solar, time}` | Ambient data |
| `session-info` | `{id, name, created, last_active}` | Current session info |
| `session-list` | `[{id, name, last_active}]` | All sessions |
| `error` | `message: string` | Error message |

---

## Business Model & Monetization

### The OpenClaw Playbook

[OpenClaw](https://github.com/openclaw/openclaw) exploded in early 2026. 100% open source, BYOK. Creator made $0 directly. YC startups built hosting businesses on top: Clawdhost ($24/mo), SimpleClaw (~$44/mo), xCloud ($24/mo), OpenClaw Cloud ($39/mo). Trust from open source, revenue from convenience.

Scout follows the same model.

### Why Open Source Is Essential

Scout accesses your microphone, voice, personal data, and home. Nobody trusts closed-source with that. Open source means:
- Audit what mic data touches (Whisper API and nowhere else)
- Verify no telemetry, no data collection
- Self-hosters keep everything on their network
- Community builds trust faster than marketing

### Revenue Tiers

| Tier | Price | What they get | Target |
|------|-------|---------------|--------|
| **Scout Free** | $0 | Full server + terminal CLI + web client. Self-hosted. BYOK. | Developers, tinkerers |
| **Scout Cloud** | $20-40/mo | Managed container with Claude Code. Your CLAUDE.md, MCPs, files. One-click setup. BYOK for API keys. | Prosumers who want it in 5 minutes |
| **Scout Pro** (app) | $5-10/mo | Premium native app: background wake word, custom wake words, multi-device sync, premium voices, widgets, themes. | Mobile-first users |
| **Scout Teams** | $50-200/mo | Multiple stations per account. Per-station CLAUDE.md. Dashboards, audit logs. White-label. | Businesses, smart offices |

### Revenue Projections (conservative)

| Milestone | Users | Cloud subs | App subs | Monthly revenue |
|-----------|-------|-----------|----------|----------------|
| 6 months | 500 | 20 @ $30 | 50 @ $7 | ~$950 |
| 12 months | 5,000 | 150 @ $30 | 400 @ $7 | ~$7,300 |
| 24 months | 25,000 | 800 @ $30 | 2,000 @ $7 | ~$38,000 |

### Scout Cloud (main revenue driver)

Each customer gets an isolated container:
```
Customer's Scout Cloud Container
├── Claude Code running in PTY
├── Their CLAUDE.md
├── Their MCP servers
├── Their files and configs
├── Scout server (Node.js)
└── Accessible via app/browser
```

We handle: provisioning, uptime, updates, backups, tunneling.
They handle: their API keys (Anthropic + OpenAI), their config.

---

## Dependencies

```json
{
  "dependencies": {
    "express": "^5",
    "ws": "^8",
    "node-pty": "^1",
    "openai": "^4",
    "dotenv": "^16",
    "strip-ansi": "^7"
  },
  "devDependencies": {
    "typescript": "^5",
    "@types/express": "^5",
    "@types/ws": "^8",
    "nodemon": "^3"
  }
}
```

xterm.js + addons loaded from CDN in the browser.

---

## Estimated API Costs

~30 voice queries/day: **~$2.40/month** (Whisper ~$0.90 + TTS ~$1.50).
Wake word detection is on-device (hybrid approach) — no API cost for listening.

---

## Verification Checklist

1. **Phase 1**: `http://localhost:3333` → multiple browsers → shared terminal → type → Claude responds
2. **Phase 2**: `scout start` / `scout ls` / `scout resume` → timeout creates new session
3. **Phase 3**: "Hey scout, how's the weather" → visual indicator → transcription → response
4. **Phase 4**: Voice query → spoken response. Typed query → no audio.
5. **Phase 5**: Tap "Weather" button → response. Idle timeout → ambient screen.
6. **Phase 6**: Phone → Add to Home Screen → full-screen Scout PWA.
7. **Phase 7**: `scout start` from terminal → resume on phone → same session.
8. **Phase 8**: Native app → "hey scout" with screen off → response.

---

## Go-to-Market

### Phase A: Build First
Product first. README is the landing page. Marketing when there's something to demo.

### Phase B: Community Launch (after Phase 6 — PWA)
- Hacker News, Reddit r/homeassistant, r/selfhosted, r/LocalLLaMA
- "Show HN: Scout — open-source voice assistant powered by Claude, replaces Alexa/Siri"
- Goal: 500 GitHub stars, 100 self-hosted users

### Phase C: Scout Cloud Launch
- Managed hosting, simple onboarding
- Free tier → paid tiers
- Target early adopters from open source community

### Phase D: App Store Launch (Phase 8)
- Native Android + iOS
- Free + Scout Pro subscription
- App Store presence legitimizes for non-technical users
