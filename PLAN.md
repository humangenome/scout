# Scout

AI voice assistant powered by Claude. Open source, self-hosted.

## Clients

| Client | Voice | Description |
|--------|-------|-------------|
| Terminal CLI | No | `scout start/ls/resume`. Text-only, free. |
| Phone app | Yes | Native (React Native). Wake word, STT, TTS. |
| Tablet app | Yes | Web kiosk. Always-on, ambient display, buttons. |

All share one session pool on the server.

## Security

- Token auth on WSS handshake. No unauthenticated connections.
- WSS-only in production. getUserMedia requires secure context.
- Origin checking.
- Claude safe mode by default. `--dangerously-skip-permissions` opt-in.
- PTY: drop privileges, resource limits, single-thread ownership (node-pty not thread-safe).
- Per-client rate limits. Audio budget (max minutes/day). Cost kill-switch.

## Sessions

SQLite. State machine: idle, busy, cooldown.

- Timeout routing: idle > N minutes = next query starts new session. Configurable, default 15 min.
- Locking: client acquires input lock on send. Others become read-only. Lock auto-releases after response + cooldown. Optimistic versioning prevents races.
- Manual override: "new session" / "continue" via voice commands or UI buttons.

```sql
sessions(id, claude_id, name, project, state, lock_owner, lock_version, created_at, last_active)
```

## Multi-Client PTY

One writer, many readers. Lock holder types, observers watch. Resize authority follows lock. Lock auto-releases on disconnect (5s grace).

## Wake Word

Picovoice Porcupine, client-side WASM. Curated keywords: "Hey Scout", "Scout", "OK Scout". Custom keywords require .ppn model files (Pro feature).

Visual-only activation (no chime). Audio only sent to server after trigger.

## Voice

- STT: Whisper API. Binary WebSocket frames. Raw transcription pasted to PTY stdin.
- TTS: OpenAI TTS. Binary MP3 frames to client. Context-aware: speaks on voice input, silent on keyboard. User-configurable voice.
- Output parsing: `claude --output-format json` for structured boundaries. Fallback: 2s silence + ANSI strip. Speaks plain text, skips code/tools.

## WebSocket Protocol (v1)

Handshake: `wss://host:port/ws?token=TOKEN&client_id=UUID&client_type=terminal|phone|tablet`

### Client to Server

| type | fields |
|------|--------|
| terminal-input | data, request_id |
| audio-start | request_id |
| audio-chunk | binary frame |
| audio-stop | request_id |
| command | text, request_id |
| resize | cols, rows |
| session-list | request_id |
| session-resume | id, request_id |
| session-new | request_id |
| lock-request | |
| lock-release | |
| ping | |

### Server to Client

| type | fields |
|------|--------|
| connected | protocol_version, session, client_id |
| terminal-output | data |
| status | state (listening/transcribing/thinking/speaking/idle) |
| transcription | text, request_id |
| tts-audio | binary frame |
| tts-done | request_id |
| ambient | weather, solar, time |
| session-info | session data |
| session-list | array of sessions |
| lock-granted | client_id |
| lock-denied | holder |
| lock-released | |
| error | code, message, request_id |
| pong | |

Error codes: auth_failed, session_busy, lock_held, rate_limited, budget_exceeded, internal_error.

## Project Structure

```
scout/
  src/
    server.ts, auth.ts, pty-manager.ts, session-manager.ts,
    audio-pipeline.ts, tts-speaker.ts, output-parser.ts,
    ambient-data.ts, rate-limiter.ts, config.ts
  cli/
    scout.ts
  public/
    index.html, scout.css, scout.js, terminal.js,
    audio.js, pcm-processor.js, wake-word.js,
    idle-screen.js, buttons.js, manifest.json
  app/
    (React Native)
  config/
    buttons.json
```

## Build Phases

**0. Auth + HTTPS**
Token auth, WSS, origin checking, HTTPS (Tailscale Funnel or Nginx/Caddy).

**1. Server + Web Terminal**
Express + WSS + node-pty + xterm.js. Single authenticated client. SQLite sessions. Auto-restart.

**2. Multi-Client + Sessions**
One writer, many readers. State machine + locking. Timeout routing. Session list/resume/new.

**3. Voice Input**
Whisper STT. AudioWorklet to binary frames. Porcupine wake word (WASM). Push-to-talk fallback. Rate limits.

**4. TTS Output**
OpenAI TTS. Context-aware. Structured output parsing. Audio budgets.

**5. iOS Spike**
Minimal React Native + Porcupine. Test foreground (works). Test background (risky, Apple 2.5.4). Decision point.

**6. Buttons + Idle Screen**
Button bar from buttons.json. Ambient display from HA REST API. Configurable idle timeout.

**7. PWA + CLI**
PWA manifest + icons. `scout` CLI. npm bin.

**8. Native Apps**
React Native (Expo). Android: foreground service for background wake word. iOS: per spike. Push notifications.

## Risks

| Risk | Mitigation |
|------|------------|
| iOS background wake word rejected | Phase 5 spike. Fallback: push-to-talk, Siri Shortcut, foreground-only. |
| node-pty privilege escalation | Container isolation, drop privileges, resource limits. |
| Multi-client PTY races | Input lock + optimistic versioning + state machine. |
| Custom wake words not a text field | Ship curated .ppn models. Custom via Porcupine Console. |
| Output parsing brittle | --output-format json primary. Silence heuristic fallback. |
| Runaway API costs | Audio budget, rate limits, cost kill-switch. |

## Dependencies

express 5, ws 8, node-pty 1, openai 4, dotenv 16, strip-ansi 7, better-sqlite3 11

xterm.js + Porcupine from CDN.

## Verification

0. No token = rejected. Valid token + WSS = connected.
1. Browser, terminal, type, Claude responds.
2. Two browsers. Lock holder types. Observer watches. Timeout creates new session.
3. "Hey scout" triggers visual indicator, transcription, response.
4. Voice input = spoken response. Keyboard = silent.
5. iOS foreground works. Background documented.
6. Button tap = response. Idle timeout = ambient screen.
7. PWA install works. `scout start` resumes on phone.
8. Android background wake word works. iOS per findings.
