# Scout — Voice-First Claude Code Assistant

## What Is Scout

Scout is an open-source, voice-first AI assistant that puts Claude Code behind a conversational interface. Say "hey scout" followed by a query; Scout transcribes it, pipes it to a Claude Code session, displays the response in a terminal, and speaks it back. Runs on any device with a browser — tablets, phones, desktops, Raspberry Pi.

## Architecture

```
Browser/PWA (thin client)              Server (Node.js)
┌──────────────────────┐              ┌────────────────────────────┐
│ xterm.js terminal    │◄────WSS─────►│ Express + WebSocket + PTY  │
│ Mic → AudioWorklet   │  audio/data  │ ├─ OpenAI Whisper (STT)    │
│ Speaker ← TTS audio  │              │ ├─ node-pty → happy (CC)   │
│ Button bar + idle     │              │ ├─ OpenAI TTS              │
│ PWA installable       │              │ └─ HA REST API (ambient)   │
└──────────────────────┘              └────────────────────────────┘
```

## Project Structure

```
scout/
├── package.json, tsconfig.json, .env
├── src/                    # Server (TypeScript → dist/)
│   ├── server.ts           # Express + WebSocket orchestration
│   ├── pty-manager.ts      # node-pty: spawn happy, I/O, resize
│   ├── audio-pipeline.ts   # PCM → WAV → Whisper API → text
│   ├── tts-speaker.ts      # Text → OpenAI TTS → MP3 → client
│   ├── output-parser.ts    # Parse Claude output for speakable text
│   ├── ambient-data.ts     # HA REST API → weather, solar
│   └── config.ts           # Env vars, constants
├── public/                 # Client (static files)
│   ├── index.html          # Responsive app shell
│   ├── scout.js            # WebSocket, state, view switching
│   ├── terminal.js         # xterm.js + WebSocket bridge
│   ├── audio.js            # Mic → AudioWorklet → WebSocket
│   ├── pcm-processor.js    # AudioWorklet: 16kHz mono PCM
│   ├── idle-screen.js      # Ambient display (clock, weather)
│   ├── buttons.js          # Button bar from config
│   ├── scout.css           # Dark theme, responsive
│   ├── manifest.json       # PWA manifest
│   └── sw.js               # Service worker
├── config/
│   └── buttons.json        # Button definitions
└── scripts/
    └── setup-tablet.md     # Kiosk setup guide
```

## Tech Stack

- **Server**: Node.js, Express 5, WebSocket (ws), node-pty, TypeScript
- **Client**: Vanilla JS, xterm.js (CDN), Web Audio API (AudioWorklet)
- **APIs**: OpenAI Whisper (STT), OpenAI TTS (tts-1), HA REST API
- **Claude**: happy --dangerously-skip-permissions (Claude Code via Happy)

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
- `OPENAI_API_KEY` — for Whisper STT and TTS
- `PORT` — server port (default 3333)

Optional:
- `HA_URL` — Home Assistant URL for ambient data
- `HA_TOKEN` — HA long-lived access token
- `CLAUDE_CMD` — Claude binary (default: happy)
- `CLAUDE_ARGS` — Claude flags (default: --dangerously-skip-permissions)
- `PROJECT_DIR` — Working directory for Claude
- `TTS_VOICE` — OpenAI voice (default: nova)

## WebSocket Protocol

### Client → Server
| type | fields | description |
|------|--------|-------------|
| `terminal-input` | `data` | Raw keystrokes |
| `audio-start` | | Begin recording |
| `audio-chunk` | `data` (base64 PCM) | Audio data |
| `audio-stop` | | End recording, trigger STT |
| `command` | `text` | Button command |
| `resize` | `cols, rows` | Terminal resize |

### Server → Client
| type | fields | description |
|------|--------|-------------|
| `terminal-output` | `data` | PTY output (ANSI) |
| `status` | `state` | listening/transcribing/thinking/speaking/idle |
| `transcription` | `text` | Whisper result |
| `tts-audio` | `data` (base64 MP3), `done` | TTS chunk |
| `ambient` | `{weather, solar, time}` | Ambient data |
| `error` | `message` | Error |

## Development Phases

1. **Web Terminal** — xterm.js + PTY via WebSocket (foundation)
2. **Push-to-Talk** — Mic → Whisper API → text → Claude
3. **TTS Output** — Claude response → OpenAI TTS → speaker
4. **Buttons + Idle** — Quick-tap commands, ambient display
5. **PWA** — Installable on phones, offline shell
6. **Wake Word** — "Hey Scout" via openWakeWord (server-side)
7. **dev.sh integration** — `dev scout` starts/stops server
8. **Native Apps** — React Native companion (future)

## Business Model

Open source core (BYOK) + managed cloud hosting (Scout Cloud $20-40/mo) + premium app features (Scout Pro $5-10/mo). Follows the OpenClaw playbook: trust from open source, revenue from convenience.

## Code Conventions

- TypeScript on server, vanilla JS on client
- Dark theme, touch-friendly, mobile-first responsive CSS
- No frameworks on client (keep it light for thin-client devices)
- xterm.js and addons loaded from CDN
- All audio processing via Web Audio API AudioWorklet
