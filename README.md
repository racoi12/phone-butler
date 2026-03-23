# Phone Butler

Personal AI agent that controls your Android phone remotely. Go off-grid with a dumb phone — Phone Butler watches your smart phone, alerts you on what matters, and executes actions on your behalf via voice commands.

## The Problem

You want disconnected days — no doom scrolling, no constant notifications. But you still need to:
- Know if the bank flags something urgent
- Get a message if family has an emergency
- Transfer money, call an Uber, or order food when needed
- Read important WhatsApp messages

## The Solution

Leave your Android phone connected to an always-on server. Carry a dumb phone. Phone Butler:

1. **Monitors** your phone's notifications and classifies them by urgency
2. **Alerts** you via Telegram or VoIP call for critical stuff only
3. **Executes** actions on your phone via voice commands ("check my Klar balance", "send mom a WhatsApp", "call an Uber home")
4. **Confirms** dangerous actions (money transfers) before executing

```
You (dumb phone) ←→ Twilio VoIP ←→ AI Agent ←→ Android Phone
                                        ↓
                                   Telegram alerts
```

## Status

**Phase 1: Notification Monitor** ← we are here

| Phase | Status | Description |
|-------|--------|-------------|
| 1 | Planned | Notification monitor + Telegram alerts |
| 2 | Planned | Read-only playbooks (balance, messages, missed calls) |
| 3 | Planned | VoIP bridge (Twilio inbound/outbound calls) |
| 4 | Planned | Write actions (transfers, messages, rides, food) |
| 5 | Planned | Autonomous agent (scheduled tasks, auto-responses) |

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for full system design, component breakdown, and implementation plan.

## Phone Options

See [CLOUD_PHONES.md](CLOUD_PHONES.md) for comparison of physical vs cloud Android options.

**TL;DR:** Physical phone ($100) is best for banking apps (SafetyNet). Cloud phones ($4-10/mo) work for non-banking apps.

## Quick Start

### Prerequisites

- Android phone with USB debugging enabled
- ADB installed (`apt install adb`)
- Python 3.11+
- [mcp-android-adb](https://github.com/racoi12/mcp-android-adb) for phone control
- [scrcpy-launcher](https://github.com/racoi12/scrcpy-launcher) for visual mirroring

### Setup

```bash
git clone https://github.com/racoi12/phone-butler.git
cd phone-butler

# Install dependencies
uv venv && uv pip install -e .

# Configure
cp .env.example .env
# Edit .env with your Telegram bot token, Twilio credentials, etc.

# Edit device config
vim config/devices.yml

# Edit notification rules
vim config/rules.yml
```

### Run notification monitor (Phase 1)

```bash
# Connect phone
adb connect 192.168.100.16:5555

# Start monitor
python daemon.py
```

### Voice commands (Phase 3+)

```
You call → Twilio number → Agent answers

"What's my Klar balance?"
"Read my WhatsApp messages"
"Send mom a message saying I'll arrive at 8"
"Call an Uber to casa"
"Order my usual from Rappi"
"Transfer 500 pesos to mamá on Klar"  ← requires PIN confirmation
```

## App Playbooks

Automation recipes for each app, defined in YAML:

| App | Read Actions | Write Actions | SafetyNet Required |
|-----|-------------|--------------|-------------------|
| WhatsApp | Read messages, list chats | Send message, make call | No |
| Klar | Check balance | Transfer money | Yes |
| BBVA | Check balance | Transfer money | Yes |
| Uber/DiDi | Check ride status | Request ride | No |
| Rappi | Check order status | Reorder favorites | No |
| Phone | Missed calls | Make call | No |
| Spotify | Now playing | Play/pause/skip | No |
| AliExpress | Order status, cart | — | No |

## Security

- Financial actions **always** require voice PIN or Telegram confirmation
- Transfer amount limits and daily caps
- Approved contacts allowlist for transfers and messages
- All actions logged with screenshots
- ADB over Tailscale VPN only
- Twilio webhooks validated with signature

## Related Projects

- [mcp-android-adb](https://github.com/racoi12/mcp-android-adb) — MCP server for Android control via ADB (40+ tools)
- [scrcpy-launcher](https://github.com/racoi12/scrcpy-launcher) — Interactive TUI for scrcpy with device registry and virtual display

## Cost Estimate

| Component | Monthly Cost |
|-----------|-------------|
| Twilio phone number | $1.15 |
| Twilio voice minutes (~50 min) | $0.65 |
| Telegram bot | Free |
| Cloud phone (optional, Redfinger) | $4-10 |
| Claude API (Sonnet, ~1000 requests) | $3-5 |
| **Total (physical phone)** | **~$5/mo** |
| **Total (with cloud phone)** | **~$15/mo** |

Physical phone is a one-time $100 cost.

## License

MIT
