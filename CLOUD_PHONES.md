# Cloud Phone Options

Comparison of cloud Android services for running a 24/7 virtual phone controlled by Phone Butler.

## Key Requirements

- Always-on (24/7, not session-based)
- Install arbitrary APKs (banking, WhatsApp, Uber, etc.)
- ADB access for automation
- Pass SafetyNet/Play Integrity (for banking apps)
- Affordable ($5-30/mo range)

## Enterprise / Developer Cloud Android

| Service | Price/mo | APKs | ADB | 24/7 | SafetyNet | Notes |
|---------|----------|------|-----|------|-----------|-------|
| **Genymotion Cloud** | $25-100 | Yes | Yes | Yes | No | Best dev ADB support. AWS/GCP/Azure. |
| **Corellium** | $99-300 | Yes | Yes + SSH | Yes | Sometimes | ARM-native VMs. Closest to real hardware. |
| **Google Cloud Emulator** | $50-150 | Yes | Yes | Yes | No | GCE VM + emulator. No GPU, slow. |
| **AWS Device Farm** | $0.17/min | Yes | Yes | No (150min max) | No | Testing only. $7,344/mo for 24/7. |
| **BrowserStack** | $40-400 | Limited | No | No (2hr max) | Yes (real devices) | Real phones, session-based. |
| **Appetize.io** | $40-200 | Yes | No | No (2hr max) | No | Browser-streamed emulator. |
| **Anbox Cloud** | $50-100+ | Yes | Yes | Yes | No | LXD containers. Self-hostable. |

## Consumer Cloud Phones (Asia-market)

| Service | Price/mo | APKs | ADB | 24/7 | SafetyNet | Notes |
|---------|----------|------|-----|------|-----------|-------|
| **Redfinger** | $3-10 | Yes | No (proprietary) | Yes | Sometimes (ARM) | Cheapest. Servers in Asia/US/EU. Has Google Play. |
| **NBCloud (Now.gg)** | $5-15 | Yes | No | Yes | Sometimes | Gaming-focused. Asia. |
| **Huawei Cloud Phone** | $10-30 | Yes | Limited | Yes | No (HMS only) | ARM Kunpeng. China mainland. |
| **Baidu Cloud Phone** | $5-20 | Yes | API available | Yes | No | China only. |

## Self-Hosted (Cheapest)

| Approach | Cost | ADB | 24/7 | SafetyNet | Notes |
|----------|------|-----|------|-----------|-------|
| **Redroid on ARM VPS** | $0-5/mo | Yes | Yes | No (Magisk possible) | **Best self-hosted option.** Docker-based Android. |
| **Waydroid on Linux** | $5-20/mo | Yes | Yes | No | LXC container. Needs kernel binder support. |
| **Android-x86 VM** | $5-20/mo | Yes | Yes | No | x86 translation issues. |
| **Docker-Android (budtmo)** | $5-20/mo | Yes | Yes | No | Docker + noVNC. x86 only. |
| **Cuttlefish** | $20-50/mo | Yes | Yes | No | Google's reference device. Needs 8GB+ RAM. |

## Best Option: Redroid on Oracle ARM VM (FREE)

You already have an Oracle Cloud ARM VM (4 CPU, 24GB RAM). **Redroid runs Android 12 in Docker on ARM — full ADB, 24/7, $0 cost.**

```bash
# Run Android 12 on ARM in Docker
docker run -d --name redroid \
  --privileged \
  -p 5555:5555 \
  redroid/redroid:12.0.0-arm64 \
  androidboot.redroid_gpu_mode=guest

# Connect via ADB (from laptop via Tailscale)
adb connect <oracle-vm-tailscale-ip>:5555

# Install apps
adb install app.apk

# Use mcp-android-adb tools directly
# All tools work: screenshot, tap, swipe, launch_app, etc.
```

**What works:** Uber, DiDi, Rappi, AliExpress, Spotify, Telegram, most non-banking apps.

**What doesn't work (without hacks):** Banking apps (Klar, BBVA) — they check Play Integrity. WhatsApp needs a phone number verification SMS.

**Google Play:** Install via OpenGApps or MindTheGapps ARM64 package.

## Play Integrity / SafetyNet Deep Dive

Banking apps require device attestation. Here's what each level means:

| Level | What | Who passes |
|-------|------|-----------|
| **MEETS_STRONG_INTEGRITY** | Locked bootloader, verified boot | Only real physical devices |
| **MEETS_DEVICE_INTEGRITY** | Unlocked OK, no root | BrowserStack (real), Corellium (sometimes), Redfinger (sometimes) |
| **MEETS_BASIC_INTEGRITY** | Can be rooted but not tampered | Some cloud phones with Magisk + Play Integrity Fix |

**Mexican banking apps:** Klar and BBVA typically require at least MEETS_DEVICE_INTEGRITY. Workaround: Magisk + Zygisk + Play Integrity Fix module — works on some ARM cloud phones but it's a cat-and-mouse game with Google.

## Recommendation: Hybrid Approach

| Use Case | Solution | Cost |
|----------|----------|------|
| Non-banking apps (Uber, Rappi, AliExpress, Spotify, Telegram) | **Redroid on Oracle ARM VM** | Free |
| Banking (Klar, BBVA) + WhatsApp | **Physical phone** (used Pixel/Poco, plugged in at home) | $50-100 one-time |
| VoIP bridge | Twilio | ~$5/mo |
| AI brain | Claude API | ~$3-5/mo |
| **Total** | | **~$8-10/mo + $50-100 one-time** |

## For WhatsApp Specifically

WhatsApp doesn't require Play Integrity but needs SMS verification:
- **Virtual numbers:** TextNow (free US), Google Voice ($0), Hushed (~$5/mo)
- **Better option:** WhatsApp Business API for pure messaging automation
- **Simplest:** Physical phone with real SIM — just leave it at home

## TODO

- [ ] Test Redroid on Oracle ARM VM
- [ ] Install Google Play on Redroid via MindTheGapps
- [ ] Test Magisk + Play Integrity Fix on Redroid
- [ ] Test which Mexican banking apps enforce which SafetyNet level
- [ ] Test Redfinger with Mexican apps from a US/EU server
- [ ] Benchmark Redroid resource usage on Oracle VM (already running other services)
