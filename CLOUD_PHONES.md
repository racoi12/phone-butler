# Cloud Phone Options

Comparison of cloud Android services for running a 24/7 virtual phone.

## Key Requirements

- Always-on (24/7, not session-based)
- Install arbitrary APKs (banking, WhatsApp, Uber, etc.)
- ADB access or equivalent API
- Pass SafetyNet/Play Integrity (for banking apps)
- Affordable ($10-30/mo range)

## Comparison

| Service | Type | 24/7 | APKs | ADB | SafetyNet | Price/mo | Notes |
|---------|------|------|------|-----|-----------|----------|-------|
| **Redfinger** | Cloud phone | Yes | Yes | No (own API) | Partial | $4-10 | Chinese service, popular in Asia, cheapest option, latency to MX servers |
| **Genymotion Cloud** | Cloud emulator | Yes (SaaS) | Yes | Yes | No | $25+ | Best ADB support, runs on AWS/GCP/Azure, no SafetyNet |
| **Corellium** | ARM virtual device | Yes | Yes | Yes | Maybe | $100+ | Enterprise-grade, real ARM, best compatibility, expensive |
| **AWS Device Farm** | Real devices | No (sessions) | Yes | Yes | Yes | $0.17/min | Real phones but session-based, not 24/7 |
| **BrowserStack** | Real devices | No (sessions) | Yes | Limited | Yes | $29+ | Testing focus, not designed for always-on |
| **Appetize.io** | Browser emulator | Session | Limited | No | No | $40+ | Web-based, good for demos, not for real apps |
| **Anbox Cloud** | Container | Yes | Yes | Yes | No | Self-host | Ubuntu-based, needs ARM server, no Google Play |
| **Waydroid** | Container (Linux) | Yes | Yes | ADB-like | No | Free | Runs on Linux, needs Wayland, no SafetyNet |
| **Android-x86 VM** | VM | Yes | Yes | Yes | No | Free | KVM/QEMU, x86 translation issues, no SafetyNet |
| **Google Cloud Android** | Emulator | Session | Yes | Yes | No | $50+ | For CI/CD, not designed for always-on |

## Best Options by Use Case

### For banking apps (needs SafetyNet)
**Physical phone is the only reliable option.** A $100 Poco M6 plugged into a Raspberry Pi or always-on PC is the cheapest path. Cloud services that use emulators will fail SafetyNet checks.

### For non-sensitive apps (Uber, Rappi, Spotify, AliExpress)
**Redfinger** ($4-10/mo) or **self-hosted Waydroid/Android-x86** (free) work well. These apps usually don't check for emulators.

### For WhatsApp
**Physical phone with SIM card required.** WhatsApp verifies the phone number via SMS and checks device integrity. WhatsApp Web/API might be an alternative for messaging only.

### Best hybrid approach
- **Physical phone** ($100 one-time): Banking, WhatsApp, calls — things that need real hardware + SIM
- **Cloud/VM** ($0-10/mo): Uber, Rappi, AliExpress, Spotify — things that don't need SafetyNet

## Self-Hosted Options (Cheapest)

### Waydroid on VPS
- Run Android in a Linux container on any VPS
- Needs a VPS with KVM support (~$5-10/mo on Hetzner, OVH)
- No Google Play Services (can sideload via MicroG)
- No SafetyNet
- Good for non-Google-dependent apps

### Android-x86 in QEMU
- Run full Android in a VM on your Oracle Cloud free tier ARM instance
- x86 translation can cause app crashes
- No SafetyNet

### Cuttlefish (Google's virtual device)
- Official Google Android Virtual Device
- Requires powerful server (8GB+ RAM)
- ADB access built in
- No SafetyNet but better compatibility than x86

## Recommendation

**For Phase 1-3 (notification monitoring, read-only actions):**
Use your existing Moto G84 or a dedicated cheap phone. The always-on PC you already have is sufficient.

**For Phase 4-5 (full automation, disconnected days):**
- **Budget option:** Raspberry Pi 4 + cheap Android phone (~$150 total, one-time)
- **Cloud option:** Redfinger ($4-10/mo) for non-banking apps + physical phone for banking
- **VPS option:** Hetzner ARM VPS ($5/mo) + Waydroid for non-banking apps

## TODO
- [ ] Test Redfinger with Mexican apps (Klar, BBVA, Rappi, Uber, DiDi)
- [ ] Test Waydroid on Oracle Cloud ARM instance
- [ ] Benchmark Cuttlefish on ARM VPS
- [ ] Test which Mexican banking apps actually enforce SafetyNet
