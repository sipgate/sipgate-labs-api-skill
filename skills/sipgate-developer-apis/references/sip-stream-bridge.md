# sipgate SIP Stream Bridge

Official documentation: [sipgate.github.io/sipgate-sip-stream-bridge](https://sipgate.github.io/sipgate-sip-stream-bridge/)

---

## What Is It?

The SIP Stream Bridge is a **Twilio Media Streams compatible translation layer** that routes telephony through sipgate's German infrastructure instead of Twilio — **without changing a single line of application code**.

It acts as middleware: your app speaks Twilio's Media Streams WebSocket protocol, the bridge translates to native SIP, and sipgate handles the actual telephony.

```
Your App ←[Twilio Media Streams WS]→ SIP Stream Bridge ←[SIP]→ sipgate ←→ Phone Network
```

**Key value**: Migrate from Twilio to German infrastructure (DSGVO/GDPR compliant, German phone numbers, data stays in Germany) with zero code changes.

---

## Common Mistakes

- **Missing SIP credentials**: You need a sipgate **trunking account** (not basic/team) from [sipgatetrunking.de](https://sipgatetrunking.de). The SIP credentials come from the trunking device settings.
- **Wrong SDP_CONTACT_IP**: Must be the public IP of the machine running the bridge, not `localhost`.
- **macOS/Windows networking**: `--network host` only works on Linux. On macOS/Windows, map ports explicitly.
- **Not using the Go version**: The Go version is production-ready and recommended. The Node.js version is a reference implementation.

---

## When to Use

Use the SIP Stream Bridge when:
- **Migrating from Twilio** — existing app uses Twilio Media Streams and you want to switch to sipgate/German infrastructure
- **Building new real-time audio apps** — you want the Twilio Media Streams protocol but with German telephony
- **DSGVO/GDPR compliance** — data must stay in Germany, no US infrastructure
- **Custom audio processing** — you need raw audio streams (e.g., for your own STT/TTS, recording, AI processing)

Do NOT use when:
- You need a simple voice bot → use **Flow API** instead (much simpler)
- You need basic call routing/IVR → use **Push API** instead
- You don't need raw audio streams → other APIs are easier

---

## Prerequisites

- **sipgate trunking account** — [sipgatetrunking.de](https://sipgatetrunking.de)
- **SIP credentials** from the trunking device (username, password, domain, registrar)
- **A server** with Docker (Linux recommended for production)
- **Your backend app** that speaks Twilio Media Streams WebSocket protocol

---

## Setup (3 Steps)

### Step 1: Configure Environment

Create a `.env` file with your SIP credentials:

```env
SIP_USER=your-sip-username
SIP_PASSWORD=your-sip-password
SIP_DOMAIN=sipgate.de
SIP_REGISTRAR=sipconnect.sipgate.de
WS_TARGET_URL=ws://your-app:8080/media-stream
SDP_CONTACT_IP=your-public-ip
```

| Variable | Description |
|----------|-------------|
| `SIP_USER` | SIP username from sipgate trunking device |
| `SIP_PASSWORD` | SIP password |
| `SIP_DOMAIN` | SIP domain (typically `sipgate.de`) |
| `SIP_REGISTRAR` | SIP registrar (typically `sipconnect.sipgate.de`) |
| `WS_TARGET_URL` | WebSocket URL of your backend app (receives Twilio Media Streams) |
| `SDP_CONTACT_IP` | Public IP of the bridge server |

### Step 2: Run with Docker

```bash
docker run --env-file .env --network host \
  ghcr.io/sipgate/sipgate-sip-stream-bridge:latest
```

**Docker image tags**:
- `latest` — current stable release
- `v1.2.3` — fixed version (recommended for production)
- `main` — bleeding edge
- `sha-abc1234` — specific commit

### Step 3: Done

Your existing Twilio-compatible backend now receives calls through sipgate's German infrastructure. No code changes needed.

---

## Features

| Feature | Supported |
|---------|-----------|
| Inbound audio streams | Yes |
| Outbound audio streams | Yes |
| DTMF tones | Yes |
| Mark/Clear commands | Yes |
| Auto-reconnection | Yes |
| Concurrent calls | Up to 100 (configurable) |
| Health check | `GET /health` |
| Prometheus metrics | `GET /metrics` |

---

## Implementation Versions

| Version | Language | Use Case | Size |
|---------|----------|----------|------|
| **Go** (recommended) | Go | Production — lightweight, fast startup, multi-week stability | ~10 MB |
| **Node.js** | Node.js | Reference/development — modifiable, identical features | Larger |

Minimal server requirements: **1 vCPU, 512 MB RAM**.

---

## Architecture

The bridge is transparent middleware. Applications cannot distinguish between direct Twilio connectivity and bridged sipgate communication.

```
┌──────────────┐    Twilio Media     ┌──────────────┐    SIP/RTP     ┌──────────┐
│  Your App    │ ←─ Streams WS ────→ │  SIP Stream  │ ←───────────→ │ sipgate  │
│  (unchanged) │    (bidirectional)   │  Bridge      │               │ Trunking │
└──────────────┘                      └──────────────┘               └──────────┘
                                            │
                                      Docker container
                                      (self-hosted)
```

**Protocol flow**:
1. Incoming call arrives at sipgate trunking number
2. sipgate routes SIP INVITE to the bridge
3. Bridge establishes WebSocket to your app using Twilio Media Streams protocol
4. Bidirectional audio streams flow between caller and your app
5. Your app sends/receives audio, DTMF, and control messages as if talking to Twilio

---

## Comparison with Other sipgate APIs

| Aspect | Flow API | SIP Stream Bridge |
|--------|----------|-------------------|
| **Abstraction level** | High (events + actions) | Low (raw audio streams) |
| **Audio access** | No (sipgate handles STT/TTS) | Yes (raw PCM audio) |
| **Protocol** | Custom JSON events | Twilio Media Streams WS |
| **Custom STT/TTS** | No (use sipgate's) | Yes (bring your own) |
| **Setup complexity** | Low (SDK + webhook) | Medium (Docker + SIP trunking) |
| **Best for** | AI voice bots with sipgate's STT/TTS | Custom audio processing, Twilio migration |
| **Account type** | sipgate basic/team | sipgate **trunking** |

---

## Cost

- **Bridge**: Free and open source
- **sipgate Trunking**: Standard German telecom pricing ([sipgatetrunking.de](https://sipgatetrunking.de))
- **Hosting**: Minimal (1 vCPU, 512 MB RAM)

---

## Compatible Example Projects

Since the bridge speaks the Twilio Media Streams protocol, all Twilio Media Streams examples work unchanged. Point `WS_TARGET_URL` at any of these:

| Project | Language | What It Does |
|---------|----------|--------------|
| [OpenAI Realtime Voice Assistant](https://github.com/twilio-samples/speech-assistant-openai-realtime-api-node) | Node.js | AI phone assistant via OpenAI Realtime API |
| [Realtime Transcription](https://github.com/twilio/media-streams/tree/master/node/realtime-transcription) | Node.js | Live speech-to-text |
| [Realtime Transcription](https://github.com/twilio/media-streams/tree/master/python/realtime-transcription) | Python | Live speech-to-text |
| [Realtime Transcription](https://github.com/twilio/media-streams/tree/master/java/realtime-transcription) | Java | Live speech-to-text |
| [Realtime Transcription](https://github.com/twilio/media-streams/tree/master/ruby) | Ruby/Rails | Live speech-to-text |
| [Keyword Detection](https://github.com/twilio/media-streams/tree/master/node/keyword-detection) | Node.js | Detect keywords in live calls |
| [Google Dialogflow](https://github.com/twilio/media-streams/tree/master/node/connect-to-google-dialogflow) | Node.js | Dialogflow conversational AI via phone |
| [Save Audio](https://github.com/twilio/media-streams/tree/master/java/save-audio) | Java | Record calls to WAV files |
| [AWS Transcribe](https://github.com/twilio/media-streams/tree/master/node/amazon-transcribe) | Node.js | Live transcription with AWS Transcribe |

All examples from the [twilio/media-streams](https://github.com/twilio/media-streams) repository are compatible.

---

## Further Reading

- [SIP Stream Bridge Documentation](https://sipgate.github.io/sipgate-sip-stream-bridge/)
- [GitHub Repository](https://github.com/sipgate/sipgate-sip-stream-bridge)
- [sipgate Trunking](https://sipgatetrunking.de)
- [Twilio Media Streams Docs](https://www.twilio.com/docs/voice/media-streams) (protocol reference — compatible with Bridge)
- [Twilio Media Streams Examples](https://github.com/twilio/media-streams) (all work with Bridge)
