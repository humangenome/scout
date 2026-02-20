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
┌─────────────────────────────────────────────────────────────────────┐
│                         Scout Server                                │
│                    (Node.js on your box)                            │
│                                                                     │
│  ┌────────────┐  ┌──────────────┐  ┌───────────────────────────┐   │
│  │ Auth       │  │ Session Mgr  │  │ PTY Manager               │   │
│  │ • token    │  │ • create     │  │ • spawn claude CLI        │   │
│  │ • origin   │  │ • resume     │  │ • I/O forwarding          │   │
│  │ • per-     │  │ • timeout    │  │ • auto-restart            │   │
│  │   client   │  │ • locking    │  │ • privilege isolation     │   │
│  │   identity │  │ • state      │  │ • flow control            │   │
│  └─────┬──────┘  │   machine    │  └────────────┬──────────────┘   │
│        │         └──────┬───────┘               │                  │
│        │                │                       │                  │
│  ┌─────▼────────────────▼───────────────────────▼──────────────┐   │
│  │              WebSocket Server (WSS)                          │   │
│  │   • versioned protocol  • heartbeat  • rate limiting        │   │
│  │   • binary audio frames • message IDs • backpressure        │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                             │                                      │
│  ┌──────────────┐  ┌───────┴────────┐  ┌───────────────────────┐  │
│  │ Audio        │  │ Output Parser  │  │ Ambient Data          │  │
│  │ Pipeline     │  │ • structured   │  │ • HA REST API         │  │
│  │ • Whisper    │  │   JSON output  │  │ • zero token usage    │  │
│  │ • TTS        │  │ • fallback     │  │                       │  │
│  │ • rate limits│  │   heuristic    │  │                       │  │
│  └──────────────┘  └────────────────┘  └───────────────────────┘  │
└─────────────────────────────┬──────────────────────────────────────┘
                              │
                ┌─────────────┼──────────────┐
                │             │              │
       ┌────────▼───┐  ┌─────▼─────┐  ┌─────▼──────┐
       │ Terminal    │  │ Phone App │  │ Tablet App  │
       │ CLI (text)  │  │ (voice)   │  │ (hub mode)  │
       │ auth token  │  │ auth token│  │ auth token  │
       │             │  │           │  │             │
       │ scout start │  │ "Hey      │  │ Always-on   │
       │ scout ls    │  │  Scout"   │  │ ambient     │
       │ scout resume│  │           │  │ display     │
       └─────────────┘  └───────────┘  └─────────────┘
```

---

## Security Model

Scout exposes a PTY (terminal) over WebSocket. Without auth, this is an **unauthenticated remote shell**. Security is not optional — it's Phase 0.

### Authentication & Authorization
- **Token-based auth**: Server generates API tokens. Clients must present a valid token on WebSocket handshake.
- **Origin checking**: Server validates `Origin` header, rejects unknown origins.
- **Per-client identity**: Each connection is identified (client ID, device type). Logged for audit.
- **WSS (TLS) required**: All connections over HTTPS/WSS. No plaintext WebSocket in production.
- **CLAUDE_ARGS default**: Safe mode by default. `--dangerously-skip-permissions` requires explicit opt-in in config.

### PTY Isolation
- PTY processes run in a restricted environment (drop privileges, resource limits).
- Single-thread ownership of PTY I/O (no concurrent writes from node-pty, which is not thread-safe).
- Flow control: backpressure handling for PTY output to prevent memory exhaustion.

### Rate Limiting & Budgets
- Per-client rate limits on WebSocket messages.
- Per-session audio budgets (max minutes/day for STT/TTS to prevent runaway API costs).
- Cost kill-switch: configurable max monthly spend, server pauses voice features when exceeded.

---

## Session Management

Scout owns the session lifecycle. Each session wraps a Claude Code `--resume` session ID with Scout metadata. Stored in **SQLite** (WAL mode, transactions, indexed on `last_active`).

### Session State Machine

```
        ┌─────────┐
        │  idle    │ ← no active query
        └────┬────┘
             │ query arrives
             ▼
        ┌─────────┐
        │  busy    │ ← Claude is processing, PTY active
        └────┬────┘
             │ response complete
             ▼
        ┌──────────┐
        │ cooldown  │ ← brief window before going idle (prevents race)
        └────┬─────┘
             │ cooldown expires
             ▼
        ┌─────────┐
        │  idle    │ ← if idle > timeout, next query starts new session
        └─────────┘
```

### Smart Session Routing (timeout-based)

- **Active session** (idle < timeout): Next query continues the same session.
- **Stale session** (idle > timeout): Next query starts a fresh session. Default timeout configurable (e.g. 15 min).
- **Busy session**: Query queued or rejected — cannot fork a session mid-response.
- **Manual override**: "Hey scout, new session" / "Hey scout, continue last session" (voice commands) or New/Continue buttons in UI.

### Session Locking

- Each session has an **owner lock** (optimistic, version-numbered).
- When a client sends input, it acquires the lock. Other clients become **read-only observers** until the response completes.
- Prevents two clients from racing at the timeout boundary and splitting a conversation.
- Lock auto-releases after response + cooldown.

### Session Pool (SQLite)

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  claude_id TEXT NOT NULL,
  name TEXT,
  project TEXT,
  state TEXT DEFAULT 'idle',      -- idle | busy | cooldown
  lock_owner TEXT,                 -- client_id holding the lock
  lock_version INTEGER DEFAULT 0,
  created_at DATETIME,
  last_active DATETIME
);
CREATE INDEX idx_sessions_last_active ON sessions(last_active);
```

All clients see the same pool. Terminal CLI, phone app, tablet — all can resume any session.

---

## Multi-Client Shared PTY

Multiple clients can connect to the same terminal session. This needs explicit rules:

### Collaboration Model
- **One writer, many readers**: When a client sends input, they hold the input lock. Other clients see output in real-time but cannot type until the lock releases.
- **Read-only observers**: Clients without the lock see a "watching" indicator. They can request the lock when it's free.
- **Resize authority**: Only the client that most recently acquired the input lock controls terminal size. Observers adapt.
- **Disconnect handling**: If the lock-holding client disconnects, lock auto-releases after 5s.

---

## Three Clients

### 1. Terminal CLI (`scout` command)

Thin wrapper around Claude CLI with session management. No voice. Free.

```bash
scout start                    # New session
scout ls                       # List all sessions
scout resume 3                 # Resume session 3
scout resume                   # Resume most recent active session
scout stop                     # Stop current session
scout server start|stop|status # Manage the Scout server
```

Feels like native `claude` CLI but with Scout's session management, auto-resume, and multi-device continuity.

### 2. Phone App (native — Android + iOS)

Conversational voice assistant. Replaces Siri / Google Assistant.

- **Hybrid wake word**: On-device detection (Picovoice Porcupine SDK), server-side STT (Whisper). Audio only sent to server after wake word triggers — zero bandwidth while listening.
- **Curated wake words**: Ship with pre-trained models: "Hey Scout", "Scout", "OK Scout". Custom wake words require generated `.ppn` model files per platform — available as Scout Pro feature.
- **Visual activation**: Screen indicator when wake word detected (no chime by default)
- **Smart TTS**: Context-aware — speaks responses when input was voice, silent when input was text
- **Smart output**: Plain text is spoken; for code/technical output says "I wrote the code, check the terminal"
- **User-configurable voice**: Pick from OpenAI TTS voices (nova, onyx, echo, alloy, fable, shimmer)
- **Background listening (Android)**: Native foreground service for background wake word — works reliably
- **Background listening (iOS)**: Limited by Apple guidelines (2.5.4). Foreground wake word guaranteed. Background wake word attempted via audio background mode, but App Store approval is not guaranteed. **Fallback**: push-to-talk, Siri Shortcut integration, or foreground-only wake word.
- **Session continuity**: Same session pool as terminal and tablet

### 3. Tablet App (web-based — kiosk/hub mode)

Always-on smart home hub. Replaces Alexa Show / Google Home Hub.

- Everything the phone app has, plus:
- **Always-listening**: Continuous wake word detection (foreground — browser tab stays open in kiosk mode)
- **Idle screen**: After configurable timeout → ambient display (clock, weather, solar — all from HA REST API, zero token usage)
- **Button bar**: Quick-tap commands from `config/buttons.json` (JSON file config only)
- **xterm.js terminal**: Full Claude Code terminal visible on screen
- **Multi-client shared session**: One writer + many readers (see collaboration model above)
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
├── tsconfig.cli.json                  # Separate build for CLI
├── .env
├── src/                               # Server (TypeScript → dist/)
│   ├── server.ts                      # Express + WSS + orchestration
│   ├── auth.ts                        # Token auth, origin checking, client identity
│   ├── pty-manager.ts                 # node-pty: spawn claude, I/O, flow control, isolation
│   ├── session-manager.ts             # SQLite session DB, state machine, locking
│   ├── audio-pipeline.ts              # PCM → WAV → Whisper API → text
│   ├── tts-speaker.ts                 # Text → OpenAI TTS → MP3 → client
│   ├── output-parser.ts               # Structured JSON output + fallback heuristic
│   ├── ambient-data.ts                # HA REST API → weather, solar (zero tokens)
│   ├── rate-limiter.ts                # Per-client rate limits, audio budgets
│   └── config.ts                      # Env vars, constants
├── cli/                               # Terminal CLI (`scout` command)
│   └── scout.ts                       # CLI: start, ls, resume, stop, server
├── public/                            # Web client (served statically)
│   ├── index.html                     # Header + terminal + footer layout
│   ├── manifest.json                  # PWA manifest
│   ├── scout.css                      # Light/dark theme, responsive
│   ├── scout.js                       # WebSocket (WSS), state, views
│   ├── terminal.js                    # xterm.js + WebSocket bridge
│   ├── audio.js                       # Mic → AudioWorklet → WebSocket
│   ├── pcm-processor.js              # AudioWorklet: 16kHz mono PCM
│   ├── wake-word.js                   # Client-side Porcupine wake word
│   ├── idle-screen.js                 # Ambient display
│   ├── buttons.js                     # Button bar from config
│   ├── icons/
│   └── sounds/
│       └── done.mp3
├── app/                               # Native app (React Native / Expo)
│   └── ...
├── config/
│   └── buttons.json                   # Button definitions (JSON file only)
└── scripts/
    └── setup-tablet.md
```

---

## Phased Build Order

### Phase 0: Auth + HTTPS + Security Foundation
**Goal**: Secure the server before exposing any functionality. No unauthenticated remote shells.

- `src/auth.ts` — Token generation/validation, origin checking, per-client identity
- `src/config.ts` — Env vars including `SCOUT_TOKEN`, `ALLOWED_ORIGINS`, `CLAUDE_ARGS` (default: safe mode)
- HTTPS setup: document Tailscale Funnel (zero-config) and Nginx/Caddy + Let's Encrypt (standard). **Required before voice features** — `getUserMedia` needs secure context.
- WSS-only WebSocket (reject `ws://` in production)
- Threat model document

**Test**: Connect without token → rejected. Connect with valid token over WSS → accepted. Connect from unknown origin → rejected.

### Phase 1: Server + Web Terminal (single client)
**Goal**: Scout server with xterm.js in the browser. Single authenticated client types commands, sees Claude respond.

- `src/server.ts` — Express serves public/ over HTTPS, WSS with auth handshake
- `src/pty-manager.ts` — Spawn `claude` CLI (safe mode default), I/O forwarding, flow control, auto-restart on crash. Single-thread PTY ownership.
- `src/session-manager.ts` — SQLite session DB, create/resume, basic state tracking
- `public/index.html` — Layout: header bar + terminal + footer bar
- `public/terminal.js` — xterm.js init, WSS with auth token, keyboard forwarding
- `public/scout.js` — WSS connection manager, auth, view state
- `public/scout.css` — Light/dark theme, responsive (phone → tablet → desktop)

**Test**: Open `https://scout:3333` with valid token → xterm.js → type → Claude responds. No token → rejected.

### Phase 2: Multi-Client + Session Management
**Goal**: Multiple clients share a terminal. Session lifecycle with timeout routing and locking.

- `src/session-manager.ts` — Full state machine (idle/busy/cooldown), locking, timeout-based routing
- `src/pty-manager.ts` — Extended for multi-client: one writer + many readers, lock semantics, resize authority
- Server WSS: session-aware routing, session-list/resume/new messages
- UI: session list, New Session / Continue buttons, "watching" indicator for observers
- Manual override: voice commands ("new session" / "continue") + UI buttons

**Test**: Two browsers connect → both see output → only lock holder can type → timeout creates new session → `session-list` shows both.

### Phase 3: Voice Input (requires HTTPS from Phase 0)
**Goal**: Tap mic or say "hey scout" to speak. Transcription sent to Claude as raw text.

- `src/audio-pipeline.ts` — Accumulate PCM, build WAV, Whisper API → text → paste to PTY stdin
- `src/rate-limiter.ts` — Per-client rate limits, audio budget (max minutes/day)
- `public/audio.js` — getUserMedia (requires secure context ✓), AudioWorklet, binary frames to server
- `public/pcm-processor.js` — AudioWorklet: 16kHz mono PCM
- `public/wake-word.js` — Client-side Porcupine wake word (WASM). Curated keywords: "Hey Scout", "Scout", "OK Scout".
- Visual-only activation (screen indicator, no chime)
- After wake word → buffer audio until silence (VAD) → Whisper → PTY

**Test**: Tap mic → speak → release → text appears → Claude responds. Say "hey scout, what time is it" → visual indicator → transcription → response.

### Phase 4: TTS Voice Output
**Goal**: Claude's responses spoken aloud. Smart detection — only speak conversational text.

- `src/tts-speaker.ts` — Text → OpenAI TTS API → binary MP3 frames to client
- `src/output-parser.ts` — Primary: use `claude --output-format json` for structured response boundaries. Fallback: 2s silence heuristic + ANSI stripping. Smart detection: speak plain text, skip code/tool calls, for technical output say "I wrote the code, check the terminal."
- Context-aware TTS: on when input was voice, off when input was keyboard
- User-configurable voice selection
- Audio budget enforcement (rate-limiter)

**Test**: Voice query → Claude responds → hear spoken response. Type query → Claude responds → no audio. Exceed budget → TTS paused, text-only.

### Phase 5: iOS Feasibility Spike
**Goal**: Validate background wake word on iOS before committing to native app architecture.

- Build minimal React Native (Expo) app with Picovoice Porcupine
- Test foreground wake word (guaranteed to work)
- Test background audio mode wake word (may be rejected by Apple)
- Test Siri Shortcut integration as fallback
- Document findings: what works, what doesn't, App Store review risk
- Decision point: full native app scope vs PWA + Siri Shortcut

**Test**: Foreground "hey scout" works. Background behavior documented with iOS constraints.

### Phase 6: Button Bar + Idle Screen
**Goal**: Quick-tap buttons. Ambient display on idle.

- `config/buttons.json` — Button definitions (JSON file only)
- `public/buttons.js` — Render buttons, tap sends command to PTY
- `src/ambient-data.ts` — HA REST API → weather, solar, time (zero token usage)
- `public/idle-screen.js` — Ambient display after configurable timeout

**Test**: Tap "Weather" → response. Wait past idle timeout → ambient screen → tap → back to terminal.

### Phase 7: PWA + Terminal CLI
**Goal**: Installable web app + `scout` command for desktop.

- `public/manifest.json` — PWA manifest (online only, no service worker caching)
- `public/icons/` — App icons (192x192, 512x512)
- `cli/scout.ts` — Full CLI: start, ls, resume, stop, server management
- `tsconfig.cli.json` — Separate TypeScript build for CLI
- `package.json` bin entry for `scout` command
- npm global install support

**Test**: Phone → "Add to Home Screen" → full-screen Scout. `scout start` → session → `scout ls` → `scout resume` from another terminal.

### Phase 8: Native Companion Apps
**Goal**: Native Android + iOS apps. Background wake word on Android (guaranteed). iOS per feasibility spike results.

- React Native (Expo) — one codebase, both platforms
- Same WSS protocol + auth as web
- Picovoice Porcupine for on-device wake word
- Android: foreground service for reliable background listening
- iOS: per Phase 5 findings — foreground wake word guaranteed, background best-effort
- Push notifications when Scout completes long tasks
- Home screen widgets

**Test**: Android → "hey scout" with screen off → response. iOS → foreground wake word works, background per findings.

---

## WebSocket Protocol (v1)

All messages are JSON. Audio data uses binary WebSocket frames (not base64) for efficiency.

### Handshake
Client connects to `wss://host:port/ws?token=<auth_token>&client_id=<uuid>&client_type=<terminal|phone|tablet>`

Server validates token and origin, then sends `{ type: "connected", protocol_version: 1, session: {...} }`.

### Client → Server
| type | fields | description |
|------|--------|-------------|
| `terminal-input` | `data: string, request_id: string` | Raw keystrokes (only if holding input lock) |
| `audio-start` | `request_id: string` | Begin voice recording |
| `audio-chunk` | *(binary frame)* | Raw PCM audio data (~100ms chunks) |
| `audio-stop` | `request_id: string` | End recording, trigger STT |
| `command` | `text: string, request_id: string` | Button command / voice override |
| `resize` | `cols: number, rows: number` | Terminal resize (only if holding lock) |
| `session-list` | `request_id: string` | Request session list |
| `session-resume` | `id: string, request_id: string` | Resume specific session |
| `session-new` | `request_id: string` | Force new session |
| `lock-request` | | Request input lock |
| `lock-release` | | Release input lock |
| `ping` | | Heartbeat |

### Server → Client
| type | fields | description |
|------|--------|-------------|
| `connected` | `protocol_version, session, client_id` | Handshake success |
| `terminal-output` | `data: string` | PTY output (ANSI) |
| `status` | `state: string` | listening / transcribing / thinking / speaking / idle |
| `transcription` | `text: string, request_id: string` | Whisper result |
| `tts-audio` | *(binary frame)* | MP3 audio chunk |
| `tts-done` | `request_id: string` | TTS playback complete |
| `ambient` | `{weather, solar, time}` | Ambient data |
| `session-info` | `{id, name, state, lock_owner, created, last_active}` | Current session |
| `session-list` | `[{id, name, state, last_active}]` | All sessions |
| `lock-granted` | `client_id: string` | Input lock acquired |
| `lock-denied` | `holder: string` | Lock held by another client |
| `lock-released` | | Lock released (broadcast) |
| `error` | `code: string, message: string, request_id?: string` | Typed error |
| `pong` | | Heartbeat response |

### Error Codes
| code | meaning |
|------|---------|
| `auth_failed` | Invalid or missing token |
| `session_busy` | Session is processing, cannot accept input |
| `lock_held` | Input lock held by another client |
| `rate_limited` | Too many requests |
| `budget_exceeded` | Audio budget (STT/TTS minutes) exceeded |
| `internal_error` | Server error |

---

## Business Model & Monetization

### The OpenClaw Playbook

[OpenClaw](https://github.com/openclaw/openclaw) exploded in early 2026. 100% open source, BYOK. Creator made $0 directly. YC startups built hosting businesses on top: Clawdhost ($24/mo), SimpleClaw (~$44/mo), xCloud ($24/mo), OpenClaw Cloud ($39/mo). Trust from open source, revenue from convenience.

Scout follows the same model. **Caveat**: hosting-on-top-of-open-source has low moat and margin pressure. The real defensibility comes from the native apps (App Store presence, background wake word, polish) and the brand.

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
| **Scout Pro** (app) | $5-10/mo | Premium native app: background wake word, custom wake word models, multi-device sync, premium voices, widgets, themes. Note: iOS requires IAP for in-app feature unlocks. | Mobile-first users |
| **Scout Teams** | $50-200/mo | Multiple stations per account. Per-station CLAUDE.md. Dashboards, audit logs. White-label. Requires auth/RBAC foundation (built into core). | Businesses, smart offices |

### Revenue Projections (conservative)

| Milestone | Users | Cloud subs | App subs | Monthly revenue |
|-----------|-------|-----------|----------|----------------|
| 6 months | 500 | 20 @ $30 | 50 @ $7 | ~$950 |
| 12 months | 5,000 | 150 @ $30 | 400 @ $7 | ~$7,300 |
| 24 months | 25,000 | 800 @ $30 | 2,000 @ $7 | ~$38,000 |

**Unit economics TBD**: Need to model support minutes/user, container infra overhead, churn rate, and customer acquisition cost before committing to pricing.

### Scout Cloud (main revenue driver)

Each customer gets an isolated container:
```
Customer's Scout Cloud Container
├── Claude Code running in PTY (privilege-isolated)
├── Their CLAUDE.md
├── Their MCP servers
├── Their files and configs
├── Scout server (Node.js)
└── Accessible via authenticated app/browser
```

We handle: provisioning, uptime, updates, backups, tunneling, auth.
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
    "strip-ansi": "^7",
    "better-sqlite3": "^11"
  },
  "devDependencies": {
    "typescript": "^5",
    "@types/express": "^5",
    "@types/ws": "^8",
    "@types/better-sqlite3": "^7",
    "nodemon": "^3"
  }
}
```

xterm.js + addons loaded from CDN in the browser.
Porcupine Web SDK loaded from CDN or bundled for wake word.

---

## Estimated API Costs

~30 voice queries/day: **~$2.40/month** (Whisper ~$0.90 + TTS ~$1.50).
Wake word detection is on-device (Porcupine) — no API cost for listening.
Configurable audio budget prevents runaway costs.

---

## Known Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| iOS background wake word rejected by App Store | High | Phase 5 feasibility spike. Fallback: push-to-talk, Siri Shortcut, foreground-only. |
| node-pty runs at parent privilege | High | Container/namespace isolation, drop privileges, resource limits. |
| Multi-client PTY race conditions | High | Input lock with optimistic versioning, session state machine. |
| Hosting business low moat | Medium | Differentiate via native apps, brand, polish. Hosting is early revenue, not endgame. |
| Apple IAP requirement for app features | Medium | iOS monetization via IAP. Android can use direct billing. Web entitlements for non-app features. |
| Custom wake words are not a text field | Medium | Ship curated keywords (pre-trained .ppn models). Custom words via Porcupine Console as Pro feature. |
| Claude CLI output parsing is brittle | Medium | Primary: `--output-format json`. Fallback: silence heuristic. |

---

## Verification Checklist

0. **Phase 0**: Connect without token → rejected. Connect with valid token over WSS → accepted.
1. **Phase 1**: `https://scout:3333` → authenticated → xterm.js → type → Claude responds.
2. **Phase 2**: Two browsers → shared terminal → lock holder types → observer watches → timeout creates new session.
3. **Phase 3**: "Hey scout, how's the weather" → visual indicator → transcription → response. Exceed audio budget → paused.
4. **Phase 4**: Voice query → spoken response. Typed query → no audio.
5. **Phase 5**: iOS spike → foreground wake word works → background behavior documented.
6. **Phase 6**: Tap "Weather" button → response. Idle timeout → ambient screen.
7. **Phase 7**: Phone → Add to Home Screen → full-screen. `scout start` → resume on phone → same session.
8. **Phase 8**: Android → "hey scout" with screen off → response. iOS → per findings.

---

## Go-to-Market

### Phase A: Build First
Product first. README is the landing page. Marketing when there's something to demo.

### Phase B: Community Launch (after Phase 7 — PWA + CLI)
- Hacker News, Reddit r/homeassistant, r/selfhosted, r/LocalLLaMA
- "Show HN: Scout — open-source voice assistant powered by Claude, replaces Alexa/Siri"
- Goal: 500 GitHub stars, 100 self-hosted users

### Phase C: Scout Cloud Launch
- Managed hosting, simple onboarding
- Free tier → paid tiers
- Target early adopters from open source community

### Phase D: App Store Launch (Phase 8)
- Native Android + iOS
- Free + Scout Pro subscription (IAP on iOS)
- App Store presence legitimizes for non-technical users
