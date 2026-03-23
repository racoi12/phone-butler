# Phone Butler — Architecture

## Overview

Phone Butler is a personal AI agent that controls your Android phone remotely, acting as a voice-activated assistant accessible from any dumb phone via VoIP. It monitors notifications, executes actions on apps, and bridges the gap between disconnected days and digital life.

## System Diagram

```
                          ┌─────────────────────────────────┐
                          │        PHONE BUTLER SERVER      │
                          │     (always-on PC or VPS)       │
                          │                                 │
  ┌──────────┐    VoIP    │  ┌───────────┐  ┌───────────┐  │     ADB/WiFi      ┌──────────────┐
  │  You     │◄──────────►│  │  Voice    │  │  Agent    │  │◄─────────────────►│  Android     │
  │  (dumb   │   Twilio   │  │  Bridge   │  │  Brain    │  │  mcp-android-adb  │  Phone       │
  │  phone)  │            │  │  STT/TTS  │  │  (Claude) │  │                   │  (or cloud)  │
  └──────────┘            │  └─────┬─────┘  └─────┬─────┘  │                   └──────────────┘
                          │        │              │         │
  ┌──────────┐  Telegram  │  ┌─────▼──────────────▼─────┐  │
  │  Alerts  │◄───────────│  │     Notification         │  │
  │  Channel │            │  │     Monitor              │  │
  └──────────┘            │  └──────────────────────────┘  │
                          │                                 │
                          │  ┌──────────────────────────┐  │
                          │  │     App Playbooks         │  │
                          │  │  (banking, uber, rappi,   │  │
                          │  │   whatsapp, calls...)     │  │
                          │  └──────────────────────────┘  │
                          │                                 │
                          │  ┌──────────────────────────┐  │
                          │  │     Action Log +          │  │
                          │  │     Confirmation Queue    │  │
                          │  └──────────────────────────┘  │
                          └─────────────────────────────────┘
```

## Components

### 1. Notification Monitor (`monitor/`)

Polls the Android device for new notifications and classifies them.

```
dumpsys notification → parse → classify urgency → route
```

**Urgency levels:**
- **CRITICAL** → VoIP call to dumb phone (bank fraud, family emergency, security alerts)
- **HIGH** → Telegram message immediately (missed calls, important WhatsApp, delivery updates)
- **MEDIUM** → Telegram digest every 30 min (app updates, promotions worth seeing)
- **LOW** → Ignored (spam, game notifications, marketing)

**Classification:** Uses keyword rules + optional Claude API for ambiguous cases.

**Tech:** Python daemon, polls every 30-60s via ADB `dumpsys notification`.

### 2. Voice Bridge (`voice/`)

Two-way voice communication between you (dumb phone) and the agent.

**Inbound (agent calls you):**
```
Critical notification → Agent composes message → TTS → Twilio call → your dumb phone
```

**Outbound (you call agent):**
```
You call Twilio number → STT transcription → Claude interprets intent → executes on phone → TTS response
```

**Tech:** Twilio (Voice API + Programmable Voice), Deepgram/Whisper for STT, edge TTS for speech.

**Estimated cost:** ~$1/mo Twilio number + $0.013/min calls = $5-15/mo depending on usage.

### 3. Agent Brain (`agent/`)

The AI that interprets commands and plans action sequences.

**Input sources:**
- Voice commands (via Voice Bridge STT)
- Telegram commands (text-based)
- Scheduled tasks (cron)
- Notification triggers (auto-responses)

**Planning:**
```
"Transfer 500 pesos to mom on Klar"
  → Intent: bank_transfer
  → App: Klar
  → Params: {amount: 500, recipient: "mom", currency: "MXN"}
  → Playbook: klar_transfer
  → Steps: [launch_klar, tap_transfer, select_contact, enter_amount, confirm]
  → Confirmation required: YES (financial)
```

**Tech:** Claude API for intent parsing, local playbook execution via mcp-android-adb.

### 4. App Playbooks (`playbooks/`)

YAML-defined sequences of actions for each app/task.

```yaml
# playbooks/klar/check_balance.yml
name: Check Klar Balance
app: com.klar.android
requires_confirmation: false
steps:
  - action: launch_app
    app: klar
  - action: wait
    seconds: 3
  - action: screenshot
    save_as: klar_home
  - action: ocr
    region: balance_area
    extract: balance
  - action: respond
    message: "Your Klar balance is {balance}"
  - action: press_key
    key: home
```

**Each playbook has:**
- Pre-conditions (app installed, logged in)
- Step sequence with error handling
- Verification screenshots at key steps
- Rollback actions if something goes wrong
- Confirmation gates for dangerous actions (payments, messages)

**Priority playbooks:**
| App | Actions | Complexity |
|-----|---------|------------|
| WhatsApp | Read messages, send message, make call | Medium |
| Klar | Check balance, transfer money | High (financial) |
| BBVA | Check balance, transfer | High (financial) |
| Uber/DiDi | Request ride to saved location | Medium |
| Rappi | Reorder from favorites | Medium |
| Phone | Make call, check missed calls | Low |
| Spotify | Play/pause, change playlist | Low |

### 5. Confirmation Queue (`queue/`)

Critical actions require explicit confirmation before execution.

```
Agent: "I'm about to transfer MX$500 to 'Mamá' on Klar. Confirm?"
  → Twilio call with TTS + keypad confirmation (press 1 to confirm)
  → Or Telegram inline button
  → Timeout after 5 min → cancel
```

### 6. Action Log (`log/`)

Every action is logged with screenshots for audit trail.

```json
{
  "timestamp": "2026-03-23T14:30:00Z",
  "trigger": "voice_command",
  "intent": "check_balance",
  "app": "klar",
  "steps_executed": 5,
  "result": "Balance: MX$12,450.00",
  "screenshots": ["klar_home.png", "klar_balance.png"],
  "duration_seconds": 8
}
```

## Phone Options

### Option A: Physical Phone (cheapest, most compatible)

- Dedicated cheap Android phone (Poco M6, ~$100 USD)
- Connected to home server via USB or WiFi ADB
- Always plugged in, screen off
- **Pros:** Real device, passes SafetyNet, all apps work, SIM card for calls/SMS
- **Cons:** Needs physical setup, power/network dependency

### Option B: Cloud Android (no hardware needed)

See [CLOUD_PHONES.md](CLOUD_PHONES.md) for detailed comparison.

- **Cons:** Most don't pass SafetyNet (banking apps fail), more expensive monthly

### Option C: Hybrid

- Cloud Android for non-sensitive apps (Uber, Rappi, Spotify)
- Physical phone for banking/WhatsApp (needs real SIM + SafetyNet)

## Security Considerations

### Critical
- **Financial actions ALWAYS require confirmation** (voice PIN, Telegram button, or keypad)
- **No stored banking passwords** — use biometric/PIN on device
- **Action logs** with screenshots for audit trail
- **Rate limiting** — max N transfers per day
- **Allowlist** — only pre-approved recipients for transfers

### Network
- ADB over Tailscale VPN (not exposed to internet)
- Twilio webhook secured with signature validation
- Telegram bot with restricted user ID

### Phone
- Device encryption enabled
- Auto-lock when not in use by agent
- Separate Google account (not personal)

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Agent brain | Claude API (Sonnet for speed, Opus for complex planning) |
| Phone control | mcp-android-adb (our MCP server) |
| Voice bridge | Twilio Voice API + Deepgram STT + edge-tts |
| Notifications | Telegram Bot API |
| Playbook engine | Python + YAML definitions |
| Scheduling | cron / APScheduler |
| Storage | SQLite (action log, notification history) |
| VPN | Tailscale (secure ADB access) |
| OCR | Tesseract or Claude Vision (for reading screen content) |

## Project Structure

```
phone-butler/
├── agent/                  # AI brain — intent parsing, action planning
│   ├── __init__.py
│   ├── brain.py            # Claude API integration
│   ├── intent.py           # Intent classification
│   └── executor.py         # Playbook executor
├── monitor/                # Notification monitoring daemon
│   ├── __init__.py
│   ├── poller.py           # ADB notification polling
│   ├── classifier.py       # Urgency classification
│   └── router.py           # Route to alerts/voice/action
├── voice/                  # VoIP bridge
│   ├── __init__.py
│   ├── bridge.py           # Twilio webhook handler
│   ├── stt.py              # Speech-to-text
│   └── tts.py              # Text-to-speech
├── playbooks/              # App automation recipes
│   ├── base.py             # Playbook engine
│   ├── whatsapp/
│   ├── klar/
│   ├── uber/
│   ├── rappi/
│   └── phone/
├── alerts/                 # Notification delivery
│   ├── __init__.py
│   ├── telegram.py         # Telegram bot
│   └── call.py             # Twilio outbound calls
├── queue/                  # Confirmation queue
│   ├── __init__.py
│   └── confirmation.py
├── log/                    # Action logging
│   ├── __init__.py
│   └── logger.py
├── config/                 # Configuration
│   ├── devices.yml         # Device registry
│   ├── contacts.yml        # Approved contacts/recipients
│   ├── rules.yml           # Notification classification rules
│   └── playbooks.yml       # Playbook registry
├── server.py               # FastAPI server (webhooks, API)
├── daemon.py               # Main daemon entry point
├── pyproject.toml
├── ARCHITECTURE.md
├── CLOUD_PHONES.md
└── README.md
```

## Implementation Phases

### Phase 1: Notification Monitor + Telegram Alerts
- Poll notifications via ADB
- Classify urgency with keyword rules
- Send alerts to Telegram
- **Deliverable:** You get important notifications on Telegram while away

### Phase 2: App Playbooks (Read-Only)
- WhatsApp: Read unread messages
- Banking: Check balance
- Phone: List missed calls
- OCR/Vision for reading screen content
- **Deliverable:** "What's my Klar balance?" via Telegram

### Phase 3: VoIP Bridge
- Twilio number setup
- Inbound calls → STT → agent
- Outbound calls → TTS for critical alerts
- **Deliverable:** Call your Twilio number, ask questions

### Phase 4: Write Actions
- WhatsApp: Send messages
- Banking: Transfer money (with confirmation)
- Uber/DiDi: Request rides
- Rappi: Reorder favorites
- **Deliverable:** "Send mom a WhatsApp saying I'll be late"

### Phase 5: Autonomous Agent
- Scheduled checks (daily balance, weekly summary)
- Auto-responses (configurable)
- Smart notification handling (auto-dismiss spam, auto-reply to known contacts)
- **Deliverable:** Phone runs itself while you're off-grid
