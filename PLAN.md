# Scout: Voice-First Claude Code Assistant

## Vision

Scout puts Claude Code behind a conversational voice interface. Say "hey scout" followed by a query; Scout transcribes it, pipes it to a Claude Code session, displays the response in a terminal, and speaks it back. Custom buttons provide quick-tap commands. An idle screen shows ambient info.

Scout is a **consumer software stack**, not a hardware project. It runs on any device with a browser — tablets in kiosk mode, Raspberry Pi, laptops, phones. Companion Android/iOS apps provide a native mobile experience.

**API choices**: OpenAI Whisper API (STT), OpenAI TTS API (TTS).
**Business model**: Open source core (trust) + managed cloud hosting (revenue) + premium app features (revenue).

---

## Business Model & Monetization

### The OpenClaw Playbook

[OpenClaw](https://github.com/openclaw/openclaw) is a viral open-source AI agent that exploded in early 2026. Key lessons:
- **100% open source, BYOK** (bring your own API key) — builds massive trust
- Creator Peter Steinberger made zero money directly from the open source project
- **YC startups built hosting businesses on top**: [Clawdhost](https://clawdhost.net) ($24/mo), [SimpleClaw](https://simpleclaw.com) (~$44/mo), [xCloud](https://xcloud.host) ($24/mo), [OpenClaw Cloud](https://www.getopenclaw.ai/pricing) ($39/mo)
- Users choose: self-host for free, or pay for convenience
- Trust comes from: code is open, your data stays local, you can audit everything
- The ecosystem is now worth millions — OpenAI acquired Steinberger and put OpenClaw under a foundation

**Scout follows the same model** — open source builds the community and trust, managed hosting captures the revenue from users who don't want to run servers.

### Why Open Source Is Essential

Scout has access to your microphone, your voice, your personal data, and your home. Nobody will trust a closed-source product with that. Open source means:
- Anyone can audit what the mic data touches (it goes to OpenAI Whisper API and nowhere else)
- Anyone can verify there's no telemetry, no data collection, no spying
- Self-hosters keep everything on their own network
- Security researchers can find and report vulnerabilities
- The community builds trust faster than any marketing budget

### Revenue Tiers

| Tier | Price | What they get | Target |
|------|-------|---------------|--------|
| **Scout Free** | $0 | Full open source server + web client. Self-hosted. BYOK (Anthropic + OpenAI keys). | Developers, privacy maximalists, tinkerers |
| **Scout Cloud** | $20-40/mo | Managed cloud container with Claude Code pre-installed. Your own isolated instance with CLAUDE.md, MCPs, files. One-click setup, no server needed. BYOK for API keys. | Prosumers who want it working in 5 minutes |
| **Scout Pro** (app) | $5-10/mo | Premium native app features: background wake word, custom wake words, multi-device sync, premium TTS voices, home screen widgets, themes. | Mobile-first users |
| **Scout Teams** | $50-200/mo | Multiple Scout stations per account. Custom CLAUDE.md per station. Usage dashboards, audit logs. White-label option. | Small businesses, smart offices |

### Revenue Projections (conservative)

| Milestone | Users | Cloud subs | App subs | Monthly revenue |
|-----------|-------|-----------|----------|----------------|
| 6 months | 500 | 20 @ $30 | 50 @ $7 | ~$950 |
| 12 months | 5,000 | 150 @ $30 | 400 @ $7 | ~$7,300 |
| 24 months | 25,000 | 800 @ $30 | 2,000 @ $7 | ~$38,000 |

### How Scout Cloud Works (main revenue driver)

Each paying customer gets their own isolated container:
```
Customer's Scout Cloud Container
├── Claude Code (happy) running in PTY
├── Their CLAUDE.md (customized for their home/workflow)
├── Their MCP servers (calendar, HA, custom tools)
├── Their files and configs
├── Scout server (Node.js)
└── Accessible via their Scout app/browser
```

We handle: provisioning, uptime, updates, backups, secure tunneling.
They handle: their API keys (Anthropic + OpenAI), their CLAUDE.md config.

### Comparable Pricing

- OpenClaw Cloud: from $39/mo
- Clawdhost: $24/mo
- Mycroft Enterprise: $1,500/mo per server
- Home Assistant Cloud (Nabu Casa): $7.50/mo

---

## Supported Platforms

Scout is a web app served from a Node.js server. Any device with a modern browser can use it.

| Platform | How it runs | Notes |
|----------|------------|-------|
| **Tablet (kiosk)** | Fully Kiosk Browser or Chrome, always-on | Primary target |
| **Raspberry Pi** | Chromium kiosk mode | Dedicated display with touchscreen |
| **Phone (Android)** | PWA → future native app | Companion for on-the-go |
| **Phone (iOS)** | PWA → future native app | iOS mic/audio constraints |
| **Desktop browser** | Chrome/Firefox/Safari | Dev/testing or desktop use |

---

## Architecture

```
Any Device (thin client)                   Server (Node.js)
┌────────────────────────┐                ┌─────────────────────────────┐
│ Browser / PWA / App    │                │ scout/dist/server.js        │
│ https://scout:3333     │     WSS        │                             │
│                        │◄──────────────►│ Express + WebSocket + PTY   │
│                        │                │                             │
│ Mic → audio.js         │  audio chunks  │ ├─ OpenAI Whisper API (STT) │
│   → WebSocket stream   ────────────────►│ ├─ node-pty → happy          │
│                        │                │ │   --dangerously-skip-perms │
│                        │  terminal data │ │   (Claude Code w/ MCPs)    │
│ xterm.js terminal     ◄────────────────│ ├─ OpenAI TTS API           │
│                        │                │ └─ HA REST API (ambient)    │
│ Speaker ← TTS audio   ◄────────────────│                             │
│                        │                │ CLAUDE.md + MCPs loaded     │
│ Touch: buttons, typing │                │                             │
└────────────────────────┘                └─────────────────────────────┘
```

---

## Project Structure

```
~/projects/scout/
├── package.json
├── tsconfig.json
├── .env                           # OPENAI_API_KEY, HA_URL, HA_TOKEN, PORT
├── src/                           # Server (TypeScript → dist/)
│   ├── server.ts                  # Express + WebSocket + orchestration
│   ├── pty-manager.ts             # node-pty: spawn happy, I/O, resize, restart
│   ├── audio-pipeline.ts          # PCM buffering → WAV → Whisper API → text
│   ├── tts-speaker.ts             # Text → OpenAI TTS → MP3 chunks → client
│   ├── output-parser.ts           # Parse Claude output, extract speakable text
│   ├── ambient-data.ts            # HA REST API → weather, solar, time
│   └── config.ts                  # Env vars, constants
├── public/                        # Client (served statically)
│   ├── index.html                 # App shell: responsive, mobile-first
│   ├── manifest.json              # PWA manifest
│   ├── sw.js                      # Service worker
│   ├── scout.css                  # Dark theme, touch-friendly, responsive
│   ├── scout.js                   # Main: WebSocket, view switching, state
│   ├── terminal.js                # xterm.js setup + WebSocket bridge
│   ├── audio.js                   # Mic capture → AudioWorklet → WebSocket
│   ├── pcm-processor.js           # AudioWorklet: 16kHz mono PCM extraction
│   ├── idle-screen.js             # Clock + weather + solar ambient display
│   ├── buttons.js                 # Render button bar from config
│   ├── icons/                     # PWA icons (192x192, 512x512)
│   └── sounds/
│       ├── wake.mp3               # "Listening" chime
│       └── done.mp3               # "Done" chime
├── config/
│   └── buttons.json               # Custom button definitions
└── scripts/
    └── setup-tablet.md            # Tablet/kiosk setup instructions
```

---

## Phased Build Order

### Phase 1: Web Terminal
**Goal**: Browser shows xterm.js with a live Claude Code session. Type commands, see responses.

Files:
- `src/server.ts` — Express serves public/, WebSocket bridges to PTY
- `src/pty-manager.ts` — Spawn `happy --dangerously-skip-permissions`, forward I/O, resize, auto-restart
- `src/config.ts` — Env vars, port, project dir
- `public/index.html` — Responsive layout: terminal + status bar
- `public/terminal.js` — xterm.js init, WebSocket, keyboard forwarding
- `public/scout.js` — WebSocket connection manager, view state
- `public/scout.css` — Dark theme, responsive

**Test**: Open `http://localhost:3333` → xterm.js → type → Claude responds.

### Phase 2: Push-to-Talk Voice Input
**Goal**: Tap mic button, speak, release. Transcription typed into Claude.

Files:
- `src/audio-pipeline.ts` — Accumulate PCM, build WAV, send to Whisper API, return text
- `public/audio.js` — getUserMedia, AudioWorklet, stream PCM to server
- `public/pcm-processor.js` — AudioWorklet: extract 16kHz mono PCM

**Flow**: Tap mic → browser captures audio → streams base64 PCM over WebSocket → release → server builds WAV → Whisper API → text → PTY stdin → Claude responds.

### Phase 3: TTS Voice Output
**Goal**: Claude's text responses spoken aloud through device speakers.

Files:
- `src/tts-speaker.ts` — Text → OpenAI TTS API (tts-1, voice "nova") → stream MP3 to client
- `src/output-parser.ts` — Detect response completion, extract conversational text (skip ANSI, tool calls, code blocks)

**Heuristic**: 2+ seconds PTY silence after output → grab last plain text block → strip ANSI → TTS.

### Phase 4: Button Bar + Idle Screen
**Goal**: Quick-tap command buttons. Ambient display when idle.

Files:
- `config/buttons.json` — Button definitions (label, icon, command, color)
- `public/buttons.js` — Render buttons, on tap send command to PTY
- `src/ambient-data.ts` — Fetch HA sensor data via REST API
- `public/idle-screen.js` — Clock + weather + solar, auto-show after 60s idle

Initial buttons: Weather, Solar, Calendar, HA Status, Lights, Stop (Ctrl+C).

### Phase 5: PWA
**Goal**: Install Scout to phone home screen. Native app feel.

Files:
- `public/manifest.json` — App name, icons, theme, display: standalone
- `public/sw.js` — Cache app shell, pass-through WebSocket
- `public/icons/` — 192x192, 512x512
- HTTPS via reverse proxy or Tailscale Funnel

### Phase 6: "Hey Scout" Wake Word
**Goal**: Always-listening, activates on "hey scout", no button press.

Approach — **server-side openWakeWord**:
- Browser continuously streams low-bandwidth audio to server
- Server runs openWakeWord Python subprocess for "hey scout"
- On detection → chime → buffer until silence (VAD) → Whisper → PTY
- Train custom TFLite model via openWakeWord synthetic data pipeline

### Phase 7: dev.sh Integration
**Goal**: `dev scout` starts/stops the Scout server.

Already partially done — `dev sc` creates Claude sessions. Add:
```bash
scout)
  local scout_dir="$HOME/projects/scout"
  if [ "$2" = "stop" ]; then
    pkill -f "node.*scout.*server" && echo "Scout stopped"
  elif [ "$2" = "status" ]; then
    pgrep -a -f "node.*scout.*server" || echo "Scout not running"
  else
    cd "$scout_dir" && node dist/server.js
  fi ;;
```

### Phase 8: Native Companion Apps (future)
**Goal**: Native Android + iOS apps with background wake word.

Approach — **React Native (Expo)**:
- Same WebSocket protocol as web
- Picovoice Porcupine for on-device wake word
- Push notifications
- One codebase, both platforms

---

## WebSocket Protocol

### Client → Server
| type | fields | description |
|------|--------|-------------|
| `terminal-input` | `data: string` | Raw keystrokes |
| `audio-start` | | Begin voice recording |
| `audio-chunk` | `data: string` (base64 PCM) | Audio data (~100ms chunks) |
| `audio-stop` | | End recording, trigger STT |
| `command` | `text: string` | Button command text |
| `resize` | `cols, rows` | Terminal resize |

### Server → Client
| type | fields | description |
|------|--------|-------------|
| `terminal-output` | `data: string` | PTY output (ANSI) |
| `status` | `state: string` | listening / transcribing / thinking / speaking / idle |
| `transcription` | `text: string` | Whisper result |
| `tts-audio` | `data: string` (base64 MP3), `done: bool` | TTS audio chunk |
| `ambient` | `{weather, solar, time}` | Ambient data |
| `error` | `message: string` | Error message |

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
With always-on wake word streaming (Phase 6): **~$6/month**.

---

## Verification Checklist

1. **Phase 1**: Open `http://localhost:3333` → xterm.js → type → Claude responds
2. **Phase 2**: Tap mic → speak → release → text in terminal → Claude responds
3. **Phase 3**: Same + hear response through speakers
4. **Phase 4**: Tap "Weather" → response. Wait 60s → idle screen → tap → terminal
5. **Phase 5**: Phone → "Add to Home Screen" → launch from icon → full-screen
6. **Phase 6**: Say "hey scout, how's solar" → chime → transcription → response + TTS
7. **Phase 7**: `dev scout` starts → `dev scout stop` stops
8. **Phase 8**: Native app with background wake word

---

## Go-to-Market Strategy

### Phase A: Build in Public (during Phases 1-6)
- Open source on GitHub from day one
- Build updates on X/Twitter
- Demo videos of voice interactions
- Target: HA community, Claude Code users, privacy-focused AI enthusiasts

### Phase B: Community Launch (after Phase 5)
- Hacker News, Reddit r/homeassistant, r/selfhosted, r/LocalLLaMA
- "Show HN: Scout — open-source voice assistant that puts Claude Code on any tablet"
- Goal: 500 GitHub stars, 100 self-hosted users

### Phase C: Scout Cloud Launch (after community validation)
- Managed hosting, simple onboarding
- Free tier → paid tiers
- Target early adopters from open source community

### Phase D: App Store Launch (Phase 8)
- Native Android + iOS
- Free + Scout Pro subscription
- App Store presence for non-technical users
