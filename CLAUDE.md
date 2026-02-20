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
├── Session Manager — create, resume, list, timeout-based routing
├── PTY Manager — spawn claude CLI, I/O, auto-restart
├── Audio Pipeline — Whisper STT, OpenAI TTS
├── Wake Word — hybrid (client detects, server transcribes)
├── Ambient Data — HA REST API (zero tokens)
└── WebSocket Server (Express + ws)
    ├── Terminal CLI (text)
    ├── Phone App (voice)
    └── Tablet App (hub)
```

**Key decisions:**
- Uses raw `claude` CLI directly (NOT Happy). Scout is its own product.
- Session management wraps Claude's `--resume` with Scout metadata (names, timestamps, project).
- Timeout-based session routing: idle > X min = next query starts new session. Configurable.
- Manual overrides: "new session" / "continue" via voice commands + UI buttons.
- Multi-client: multiple browsers can connect to same PTY, all see and type.
- Auto-restart on claude process crash.

## Project Structure

```
scout/
├── package.json, tsconfig.json, .env
├── src/                    # Server (TypeScript → dist/)
│   ├── server.ts           # Express + WebSocket orchestration
│   ├── pty-manager.ts      # node-pty: spawn claude, I/O, resize, auto-restart
│   ├── session-manager.ts  # Session lifecycle, timeout routing, metadata DB
│   ├── audio-pipeline.ts   # PCM → WAV → Whisper API → text (raw paste to stdin)
│   ├── tts-speaker.ts      # Text → OpenAI TTS → MP3 → client
│   ├── output-parser.ts    # Smart detection: speak plain text, skip code/tools
│   ├── ambient-data.ts     # HA REST API → weather, solar (zero token usage)
│   └── config.ts           # Env vars, constants
├── cli/                    # Terminal CLI
│   └── scout.ts            # scout start, ls, resume, stop, server
├── public/                 # Web client (tablet/browser)
│   ├── index.html          # Header + terminal + footer layout
│   ├── scout.css           # Light/dark theme, responsive, consumer-polished
│   ├── scout.js            # WebSocket, state, views
│   ├── terminal.js         # xterm.js + WebSocket bridge
│   ├── audio.js            # Mic → AudioWorklet → WebSocket
│   ├── pcm-processor.js    # AudioWorklet: 16kHz mono PCM
│   ├── wake-word.js        # Client-side wake word (WASM/TFLite)
│   ├── idle-screen.js      # Ambient display
│   ├── buttons.js          # Button bar from config
│   ├── manifest.json       # PWA manifest
│   └── icons/, sounds/
├── app/                    # Native app (React Native / Expo — future)
├── config/
│   └── buttons.json        # Button definitions (JSON file only)
└── scripts/
    └── setup-tablet.md
```

## Tech Stack

- **Server**: Node.js, Express 5, WebSocket (ws), node-pty, TypeScript
- **Terminal CLI**: Node.js, wraps claude CLI with session management
- **Web client**: Vanilla JS, xterm.js (CDN), Web Audio API (AudioWorklet)
- **Native app**: React Native (Expo) — future, priority feature
- **APIs**: OpenAI Whisper (STT), OpenAI TTS, HA REST API
- **Claude**: `claude --dangerously-skip-permissions` (raw CLI, no Happy)
- **Wake word**: Hybrid — client-side detection (WASM), server-side STT. Configurable phrase.
- **Design**: Clean light + dark mode. Modern, polished, consumer-grade (Apple-like).

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

Optional:
- `HA_URL` — Home Assistant URL for ambient data
- `HA_TOKEN` — HA long-lived access token
- `CLAUDE_CMD` — Claude binary (default: claude)
- `CLAUDE_ARGS` — Claude flags (default: --dangerously-skip-permissions)
- `PROJECT_DIR` — Working directory for Claude
- `TTS_VOICE` — OpenAI voice (default: nova, user-configurable)
- `SESSION_TIMEOUT` — Minutes before auto-new-session (default: 15)

## Voice Behavior

- **TTS is context-aware**: Speaks when input was voice, silent when input was keyboard.
- **Smart output detection**: Speaks plain text, skips code/ANSI/tool calls. For code: "I wrote the code, check the terminal."
- **Wake word**: Visual-only activation (screen indicator, no chime). Configurable phrase.
- **Transcription**: Raw paste into Claude stdin. No prefix, no wrapping.

## Code Conventions

- TypeScript on server + CLI, vanilla JS on web client
- Light + dark theme, consumer-polished CSS
- No frameworks on web client (keep lightweight for thin-client devices)
- xterm.js and addons from CDN
- Audio via Web Audio API AudioWorklet
- Session data in JSON (future: SQLite)

## Business Model

Open source (MIT, BYOK) + Scout Cloud ($20-40/mo managed hosting) + Scout Pro ($5-10/mo premium app features). Follows the OpenClaw playbook.
