# Scout

AI voice assistant powered by Claude. Open source, self-hosted, privacy-first.

## Essential Documentation

| Document | Purpose |
|----------|---------|
| `PLAN.md` | Full technical spec — architecture, WebSocket protocol, voice pipeline, session model, build phases |
| `.env.example` | All environment variables with descriptions |
| `config/buttons.json` | Tablet idle-screen button definitions |

**Read PLAN.md before implementing any feature.** It contains the WebSocket message schema, session state machine, multi-client locking model, and phase-by-phase build order.

---

## Architecture

**Server**: Node.js + Express 5 + ws 8 + node-pty + better-sqlite3 + TypeScript
**Web Client**: Vanilla JS, xterm.js (CDN), Web Audio AudioWorklet, Porcupine WASM (CDN)
**Native Client**: React Native (Expo) — future
**APIs**: OpenAI Whisper (STT), OpenAI TTS, Home Assistant REST (ambient data)

**How it works**: Scout runs as an always-on server (Docker or native Linux). Client connects via WSS with token auth. Server spawns `claude` CLI in a PTY. Terminal I/O streams over WebSocket. Voice input goes through Whisper STT, transcription gets pasted to PTY stdin. Claude's text responses get spoken back via OpenAI TTS. SQLite tracks sessions with state machine (idle/busy/cooldown) and optimistic locking for multi-client.

**Deployment**: Docker (recommended) or native Linux. Clients reach the server remotely via Tailscale, Cloudflare Tunnel, or reverse proxy. See PLAN.md Deployment section for full details.

---

## Project Structure

```
scout/
  src/           server.ts, auth.ts, pty-manager.ts, session-manager.ts,
                 audio-pipeline.ts, tts-speaker.ts, output-parser.ts,
                 ambient-data.ts, rate-limiter.ts, config.ts
  cli/           scout.ts (terminal CLI client)
  public/        index.html, scout.css, scout.js, terminal.js,
                 audio.js, pcm-processor.js, wake-word.js,
                 idle-screen.js, buttons.js, manifest.json
  public/icons/  PWA icons
  public/sounds/ (reserved)
  app/           React Native (Expo) — future
  config/        buttons.json
  data/          SQLite database (gitignored)
  scripts/       (reserved)
  Dockerfile
  docker-compose.yml
```

---

## Build & Run

```bash
# Native
npm install
npm run build    # tsc (server + cli)
npm start        # node dist/server.js
npm run dev      # nodemon watch + rebuild
npm run clean    # rm -rf dist

# Docker
docker compose up --build     # build and start
docker compose up -d          # detached (always-on)
docker compose down           # stop
```

Port: `3333` (default)

---

## Environment (.env)

**Required**: `OPENAI_API_KEY`, `PORT`, `SCOUT_TOKEN`

**Optional**: `ANTHROPIC_API_KEY` (bypasses OAuth login — pay-per-use rates), `ALLOWED_ORIGINS`, `HA_URL`, `HA_TOKEN`, `CLAUDE_CMD` (default: `claude`), `CLAUDE_ARGS`, `PROJECT_DIR`, `TTS_VOICE` (default: `nova`), `TTS_MODEL`, `STT_MODEL`, `SESSION_TIMEOUT` (default: 15 min), `AUDIO_BUDGET_MINUTES` (default: 60), `MAX_MONTHLY_SPEND`

---

## Build Phases

Build in order. Each phase is a working milestone.

| Phase | What | Key Files |
|-------|------|-----------|
| 0 | Auth + HTTPS + Deployment | auth.ts, config.ts, Dockerfile, docker-compose.yml |
| 1 | Server + web terminal (single client) | server.ts, pty-manager.ts, session-manager.ts, index.html, terminal.js |
| 2 | Multi-client + sessions | session-manager.ts (locking), scout.js |
| 3 | Voice input (STT + wake word) | audio-pipeline.ts, audio.js, pcm-processor.js, wake-word.js |
| 4 | TTS output | tts-speaker.ts, output-parser.ts |
| 5 | iOS spike | app/ |
| 6 | Buttons + idle screen | idle-screen.js, buttons.js, ambient-data.ts |
| 7 | PWA + CLI | manifest.json, scout.ts |
| 8 | Native apps | app/ |

---

## Rules

### RULE 1: Follow the Build Phases

Do not skip ahead. Phase 0 before Phase 1, etc. Each phase should be fully working before starting the next. See PLAN.md for phase details and verification criteria.

### RULE 2: Security First

- Token auth is **required**. No unauthenticated WebSocket connections, ever.
- Claude safe mode is **default**. `--dangerously-skip-permissions` is opt-in via `CLAUDE_ARGS` env var.
- WSS required in production (getUserMedia needs secure context).
- PTY: drop privileges, enforce resource limits. node-pty is not thread-safe — single-thread ownership only. Docker handles most isolation.
- Never expose `.env` or tokens in client-side code.
- In Docker: always run as non-root `node` user. Use `init: true` for zombie reaping.

### RULE 3: Code Style

- TypeScript strict mode. No `any` unless absolutely necessary.
- Vanilla JS for web client — no frameworks, no bundler.
- Follow existing patterns. Match indentation, naming, structure.
- `//` single-line comments. No JSDoc blocks unless documenting a public API.
- Keep files focused. One module per concern per the structure above.

### RULE 4: Git & Deployment

- Commit and push after every change. Don't let work pile up uncommitted.
- Commit and push working code only. Don't commit broken builds.
- Descriptive commit messages explaining what changed and why.
- No `git clean`, `git reset --hard`, or force pushes without explicit approval.
- Run `npm run build` before committing to verify TypeScript compiles.

### RULE 5: PLAN.md is Source of Truth

PLAN.md contains the authoritative spec for:
- WebSocket message types and fields
- Session state machine and locking model
- Multi-client PTY behavior
- Voice pipeline architecture
- Wake word implementation details

Do not contradict PLAN.md. If something needs to change, update PLAN.md first.

### RULE 6: No Overengineering

- Build the simplest thing that works for the current phase.
- No premature abstractions, feature flags, or "future-proofing."
- Three similar lines of code > a premature helper function.
- Don't add config options for things that can be hardcoded now.
