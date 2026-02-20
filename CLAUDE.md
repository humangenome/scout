# Scout

AI voice assistant powered by Claude. Open source, self-hosted, three clients (terminal CLI, phone app, tablet hub), one server.

## Key Decisions

- Raw `claude` CLI. Not Happy. Scout is its own product.
- Auth required. Token on WSS handshake. No unauthenticated shells.
- Safe mode default. `--dangerously-skip-permissions` is opt-in.
- SQLite for sessions. State machine (idle/busy/cooldown) with optimistic locking.
- Multi-client: one writer, many readers. Input lock.
- Wake word: Porcupine (client-side WASM). Curated keywords, not arbitrary text.
- Output parsing: `--output-format json` primary, 2s silence heuristic fallback.
- TTS: context-aware. Voice input = spoken response. Keyboard = silent.
- HTTPS required (getUserMedia needs secure context).

## Structure

```
src/    server.ts, auth.ts, pty-manager.ts, session-manager.ts,
        audio-pipeline.ts, tts-speaker.ts, output-parser.ts,
        ambient-data.ts, rate-limiter.ts, config.ts
cli/    scout.ts
public/ index.html, scout.css, scout.js, terminal.js,
        audio.js, pcm-processor.js, wake-word.js,
        idle-screen.js, buttons.js, manifest.json
app/    React Native (Expo)
config/ buttons.json
```

## Build and Run

```bash
npm install
npm run build    # tsc
npm start        # node dist/server.js (port 3333)
npm run dev      # nodemon watch + rebuild
```

## Environment (.env)

Required: OPENAI_API_KEY, PORT, SCOUT_TOKEN

Optional: ALLOWED_ORIGINS, HA_URL, HA_TOKEN, CLAUDE_CMD (default: claude), CLAUDE_ARGS, PROJECT_DIR, TTS_VOICE (default: nova), SESSION_TIMEOUT (default: 15 min), AUDIO_BUDGET_MINUTES (default: 60), MAX_MONTHLY_SPEND

## Tech

Server: Node.js, Express 5, ws 8, node-pty, better-sqlite3, TypeScript
Client: vanilla JS, xterm.js (CDN), Web Audio AudioWorklet, Porcupine WASM
Native: React Native (Expo)
APIs: OpenAI Whisper, OpenAI TTS, HA REST
Design: light + dark mode, responsive, consumer-polished

## Build Phases

0. Auth + HTTPS
1. Server + web terminal (single client)
2. Multi-client + session management
3. Voice input (STT + wake word)
4. TTS output
5. iOS feasibility spike
6. Buttons + idle screen
7. PWA + CLI
8. Native apps
