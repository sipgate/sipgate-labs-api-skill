# Choosing the Right sipgate API

## Common Mistakes

- **Using REST API for conversational voice interaction**: REST API can initiate calls and manipulate active calls via [RTCM](https://api.sipgate.com/v2/doc) (hold, mute, transfer, record, play announcements), but it cannot handle conversations (speech recognition, TTS). Use Flow API for AI voice bots or Push API for IVR/routing.
- **Using Push API when Flow API would be simpler**: For AI-powered voice bots, Flow API + SDK handles STT/TTS for you. Push API only does basic routing via XML.
- **Using SIP Stream Bridge when Flow API suffices**: If sipgate's built-in STT/TTS are good enough, Flow API is much simpler. Only use the Bridge when you need raw audio or custom STT/TTS providers.
- **Forgetting that SIP Stream Bridge needs trunking**: The Bridge requires a sipgate **trunking** account ([sipgatetrunking.de](https://sipgatetrunking.de)), not basic/team.

## API Overview

| API | Purpose | Best For | Docs |
|-----|---------|----------|------|
| **Flow API** | Real-time voice AI | Voice bots, AI receptionists, conversational IVR | [Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) |
| **REST API** | Account & communication management + RTCM | SMS, fax, click-to-call, contacts, history, **active call control** (hold, mute, transfer, record, announcements) | [Swagger](https://api.sipgate.com/v2/doc) |
| **Push API** | Telephony event webhooks | IVR menus, call routing, call logging, screening | [Docs](https://en.sipgate.io/push-api/api-references) |
| **SIP Stream Bridge** | Raw audio streams, Twilio compat | Twilio migration, custom STT/TTS, audio processing | [Docs](https://sipgate.github.io/sipgate-sip-stream-bridge/) |

---

## Decision Matrix

| Use Case | API | Why |
|----------|-----|-----|
| AI voice bot / receptionist | Flow API (SDK) | Real-time speech-to-text + TTS, event-driven conversation |
| IVR menu (press 1 for...) | Push API | Gather DTMF digits, route calls via XML |
| Click-to-call from web app | REST API | Simple POST to initiate call |
| Send SMS | REST API | Single POST request |
| Send fax | REST API | Single POST with base64 PDF |
| Log calls to CRM | Push API + REST API | Push for real-time events, REST for contact lookup |
| Block unwanted callers | Push API | Reject in newCall handler |
| Missed call notification | Push API + REST API | Push detects miss, REST sends SMS |
| Sync contacts | REST API | CRUD operations on contacts |
| Voicemail transcription | REST API | Fetch + transcribe voicemails |
| Call analytics dashboard | REST API (v3) | Query historical events |
| "Login with sipgate" | REST API (OAuth2) | Standard OAuth2/OpenID Connect |
| Conference calls | REST API | Manage conference rooms |
| Custom phonebook | REST API + OAuth2 | Contacts API + authentication |
| Twilio migration | SIP Stream Bridge | Zero-code-change drop-in, German infrastructure |
| Custom STT/TTS pipeline | SIP Stream Bridge | Raw audio access, bring your own AI |
| Real-time audio analysis | SIP Stream Bridge | Raw PCM streams via Twilio Media Streams WS |

---

## Flow API: Integration Method Preference

When using the Flow API, always prefer in this order:

1. **SDK** ([`@sipgate/ai-flow-sdk`](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)) — always the first choice. Type-safe, abstracts complexity.
2. **WebSocket** — if SDK doesn't fit your stack. Bidirectional, lower latency, supports ALL events including `user_speech_started`.
3. **HTTP Webhooks** — only as fallback. Some events are not available over HTTP.

> **Outbound calls** via Flow API are only available on explicit request to sipgate. Do not suggest this as a standard feature.

---

## Combining APIs

Many real-world applications combine multiple sipgate APIs:

### Push API + REST API
- **Call screening**: Push API receives `newCall` → REST API looks up contact → decide to accept or reject
- **Call logging**: Push API events → store in your system → REST API for additional data
- **Missed call SMS**: Push API `onHangup` → REST API `POST /sessions/sms`

### Flow API + REST API
- **AI receptionist**: Flow API handles conversation → REST API for contact/calendar lookup
- **Smart transfer**: Flow API understands caller intent → REST API to find right department

### Push API → Flow API
- **IVR-to-AI handoff**: Push API collects initial DTMF choice → transfer to Flow API for AI-powered conversation

### SIP Stream Bridge
- **Lowest level**: Direct access to raw audio streams via Twilio Media Streams WebSocket protocol
- Requires sipgate **trunking** account ([sipgatetrunking.de](https://sipgatetrunking.de))
- Use when you need raw audio, custom STT/TTS, or are migrating from Twilio
- See `references/sip-stream-bridge.md`

See `references/common-patterns.md` for detailed implementation patterns with sequence diagrams.

---

## Authentication by API

| API | Auth Method | Details |
|-----|-------------|---------|
| REST API | OAuth2 or PAT | [Authentication Guide](https://en.sipgate.io/rest-api/authentication) |
| Flow API | Shared Secret | `X-API-TOKEN` header validation |
| Push API | Webhook URL | Configure in sipgate settings |
| SIP Stream Bridge | SIP credentials | Trunking device credentials |

See `references/authentication.md` for full details.
