# Scout — AI Voice Assistant Powered by Claude

## What Is Scout

Scout is an open-source AI voice assistant that replaces Siri, Google Assistant, and Alexa — powered by Claude Code. It controls your smart home, answers questions, manages your schedule, and runs agentic coding tasks through natural voice conversation.

Three clients, one server:
- **Terminal CLI** (`scout` command) — text only, no voice, free. Continue conversations from your desktop.
- **Phone app** (native) — conversational voice with "hey scout" wake word. Replaces Siri/Google Assistant.
- **Tablet app** (web kiosk) — always-on hub with voice, buttons, ambient display. Replaces Alexa/Google Home.

All three connect to the same session pool. Start a conversation on your phone, pick it up on your desktop.

## Architecture

```
Scout Server (Node.js)
├── Auth — token auth, origin checking, per-client identity, WSS only
├── Session Manager — SQLite DB, state machine (idle/busy/cooldown), locking
├── PTY Manager — spawn claude CLI, I/O, flow control, privilege isolation, auto-restart
├── Audio Pipeline — Whisper STT, OpenAI TTS, rate limits, audio budgets
├── Output Parser — structured JSON output (primary) + silence heuristic (fallback)
├── Ambient Data — HA REST API (zero tokens)
└── WebSocket Server (WSS, versioned protocol, heartbeat, binary audio frames)
    ├── Terminal CLI (text, auth token)
    ├── Phone App (voice, auth token)
    └── Tablet App (hub, auth token)
```

**Key decisions:**
- Uses raw `claude` CLI directly (NOT Happy). Scout is its own product.
- **Auth is Phase 0** — token-based, WSS-only, origin checking. No unauthenticated remote shells.
- Session management: SQLite, state machine with locking, timeout-based routing. Prevents race conditions.
- Multi-client: one writer + many readers. Input lock with optimistic versioning.
- Auto-restart on claude process crash. Single-thread PTY ownership (node-pty is not thread-safe).
- Wake word: Picovoice Porcupine (client-side detection), Whisper (server-side STT). Curated keywords ship pre-trained; custom words are a Pro feature.
- Output parsing: `--output-format json` for structured boundaries, 2s silence heuristic as fallback.
- HTTPS required — `getUserMedia` needs secure context.

## Project Structure

```
scout/
├── package.json, tsconfig.json, tsconfig.cli.json, .env
├── src/                         # Server (TypeScript → dist/)
│   ├── server.ts                # Express + WSS + orchestration
│   ├── auth.ts                  # Token auth, origin check, client identity
│   ├── pty-manager.ts           # node-pty: claude CLI, flow control, isolation
│   ├── session-manager.ts       # SQLite, state machine, locking, timeout
│   ├── audio-pipeline.ts        # PCM → WAV → Whisper → text
│   ├── tts-speaker.ts           # Text → TTS → binary MP3 frames
│   ├── output-parser.ts         # Structured JSON + fallback heuristic
│   ├── ambient-data.ts          # HA REST API (zero tokens)
│   ├── rate-limiter.ts          # Per-client limits, audio budgets
│   └── config.ts                # Env vars, constants
├── cli/                         # Terminal CLI
│   └── scout.ts                 # scout start, ls, resume, stop, server
├── public/                      # Web client (tablet/browser)
│   ├── index.html               # Header + terminal + footer
│   ├── scout.css                # Light/dark, responsive, consumer-polished
│   ├── scout.js                 # WSS + auth, state, views
│   ├── terminal.js              # xterm.js + WSS bridge
│   ├── audio.js                 # Mic → AudioWorklet → binary frames
│   ├── pcm-processor.js         # AudioWorklet: 16kHz mono PCM
│   ├── wake-word.js             # Porcupine WASM wake word
│   ├── idle-screen.js           # Ambient display
│   ├── buttons.js               # Button bar from config
│   ├── manifest.json            # PWA manifest
│   └── icons/, sounds/
├── app/                         # Native app (React Native / Expo)
├── config/
│   └── buttons.json             # Button definitions (JSON only)
└── scripts/
    └── setup-tablet.md
```

## Tech Stack

- **Server**: Node.js, Express 5, WSS (ws), node-pty, better-sqlite3, TypeScript
- **Terminal CLI**: Node.js, wraps claude CLI with session management
- **Web client**: Vanilla JS, xterm.js (CDN), Web Audio API (AudioWorklet), Porcupine WASM
- **Native app**: React Native (Expo) — priority feature
- **APIs**: OpenAI Whisper (STT), OpenAI TTS, HA REST API
- **Claude**: `claude` CLI (raw, no Happy). Safe mode default, `--dangerously-skip-permissions` opt-in.
- **Wake word**: Picovoice Porcupine (client-side). Curated: "Hey Scout", "Scout", "OK Scout".
- **Design**: Clean light + dark mode. Consumer-polished (Apple-like).

## Build & Run

```bash
cd ~/projects/scout
npm install
npm run build          # tsc: src/ → dist/
npm start              # node dist/server.js (port 3333)
npm run dev            # nodemon: watch + rebuild
```

## Environment Variables (.env)

Required:
- `OPENAI_API_KEY` — Whisper STT and TTS
- `PORT` — server port (default 3333)
- `SCOUT_TOKEN` — auth token for client connections

Optional:
- `ALLOWED_ORIGINS` — comma-separated allowed origins (default: localhost)
- `HA_URL` — Home Assistant URL for ambient data
- `HA_TOKEN` — HA long-lived access token
- `CLAUDE_CMD` — Claude binary (default: claude)
- `CLAUDE_ARGS` — Claude flags (default: none. `--dangerously-skip-permissions` is opt-in)
- `PROJECT_DIR` — Working directory for Claude
- `TTS_VOICE` — OpenAI voice (default: nova, user-configurable)
- `SESSION_TIMEOUT` — Minutes before auto-new-session (default: 15)
- `AUDIO_BUDGET_MINUTES` — Max STT+TTS minutes per day (default: 60)
- `MAX_MONTHLY_SPEND` — Cost kill-switch for voice APIs (default: unlimited)

## Voice Behavior

- **TTS is context-aware**: Speaks when input was voice, silent when input was keyboard.
- **Smart output detection**: Primary: structured JSON output from `--output-format json`. Fallback: 2s silence heuristic + ANSI stripping. Speaks plain text, skips code/tools. For code: "I wrote the code, check the terminal."
- **Wake word**: Porcupine (client-side, WASM). Visual-only activation. Curated keywords.
- **Transcription**: Raw paste into Claude stdin. No prefix, no wrapping.
- **Rate limiting**: Per-client limits, configurable audio budget, cost kill-switch.

## Security

- **All connections authenticated** — token required on WSS handshake
- **WSS only in production** — no plaintext WebSocket
- **Origin checking** — rejects unknown origins
- **PTY isolation** — drop privileges, resource limits, single-thread ownership
- **Safe mode default** — `--dangerously-skip-permissions` is opt-in, not default
- **Rate limiting** — prevents abuse and runaway API costs

## Code Conventions

- TypeScript on server + CLI, vanilla JS on web client
- Light + dark theme, consumer-polished CSS
- No frameworks on web client (lightweight for thin-client devices)
- xterm.js and addons from CDN
- Audio via Web Audio API AudioWorklet + binary WebSocket frames
- Session data in SQLite (WAL mode)
- Versioned WebSocket protocol (v1)

## Business Model

Open source (MIT, BYOK) + Scout Cloud ($20-40/mo managed hosting) + Scout Pro ($5-10/mo premium app features via IAP on iOS). Follows the OpenClaw playbook. Real defensibility from native apps and brand, not hosting margin.
