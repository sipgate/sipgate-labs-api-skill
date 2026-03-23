---
name: sipgate-developer-apis
description: "Comprehensive guide for sipgate API integrations. Use when building voice bots, sending SMS/fax, handling telephony webhooks, implementing 'Login with sipgate', raw audio streaming, Twilio migration, or integrating with sipgate's Flow API, REST API, Push API, or SIP Stream Bridge. Covers authentication (OAuth2, PAT), use cases, and cross-API patterns. Keywords: sipgate, telephony, voice bot, AI receptionist, webhooks, PBX, SMS, fax, click-to-call, IVR, DTMF, SIP, VoIP, call routing, call screening, sipgate.io, sipgateio, Twilio, media streams, audio streaming, SIP trunking, WebRTC."
---

# sipgate Developer APIs

This skill helps developers choose and integrate the right sipgate API for their use case.

## APIs Overview

| API | Purpose | Documentation |
|-----|---------|---------------|
| **Flow API** | Real-time AI voice applications (bots, receptionists) | [LLM Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) |
| **REST API** | Account management, SMS, fax, calls, contacts, history | [Swagger Docs](https://api.sipgate.com/v2/doc) |
| **Push API** | Telephony event webhooks (call routing, IVR, logging) | [API Reference](https://en.sipgate.io/push-api/api-references) |
| **SIP Stream Bridge** | Raw audio streaming, Twilio migration, custom STT/TTS | [Documentation](https://sipgate.github.io/sipgate-sip-stream-bridge/) |

## When to Use This Skill

- Building any integration with sipgate APIs
- Choosing between Flow, REST, and Push APIs
- Implementing "Login with sipgate" (OAuth2)
- Setting up authentication (OAuth2, PAT, shared secret)
- Building voice bots, IVR menus, SMS/fax sending, call management
- Raw audio streaming or migrating from Twilio (SIP Stream Bridge)
- Combining multiple sipgate APIs

---

## MANDATORY RULES

Follow these rules in ALL responses involving sipgate APIs:

### Phone Numbers
All phone numbers must be in **E.164 format**: `+{country}{number}` (e.g., `+4915790123456`). [Reference](https://en.wikipedia.org/wiki/E.164)

### Personal Access Token Warning
**EVERY TIME** Personal Access Tokens (PATs) are mentioned, discussed, or used in code, include this warning:

> **WARNING**: Personal Access Tokens grant full account access. Never commit them to repositories or use in client-side code. Use only for personal scripts, CLI tools, and prototyping. For production applications, always use OAuth2. [PAT Settings](https://app.sipgate.com/personal-access-token)

### Flow API: Integration Method Hierarchy
When working with the Flow API, **ALWAYS** recommend in this order:
1. **SDK** ([`@sipgate/ai-flow-sdk`](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)) — always the first suggestion
2. **WebSocket** — if SDK doesn't fit the user's stack
3. **HTTP Webhooks** — only as last resort

### Outbound Calls (Flow API)
Outbound calls via the Flow API are **only available on explicit request** to sipgate. Do NOT proactively suggest outbound call use cases or campaigns.

### Language-Agnostic
Provide **cURL/HTTP examples** by default, not language-specific code. The one exception: Flow SDK examples may use TypeScript (the SDK only exists in TS). When the user works in a specific language, adapt accordingly.

### Documentation Links
**ALWAYS** enrich concepts, endpoints, and auth methods with links to official documentation. Never explain a concept without linking to its docs.

---

## Router: Which Reference to Load

### Starting a new project?
→ Read `references/getting-started.md` first (account setup, OAuth2, "Login with sipgate")

### Not sure which API to use?
→ Read `references/choosing-an-api.md` (decision matrix) and `references/use-cases.md` (detailed use cases)

### Working with a specific API?

**Flow API** (voice bots, AI receptionists):
1. Read `references/flow-api.md` for quick reference and SDK setup
2. **WebFetch** the full reference: `https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt`

**REST API** (SMS, fax, calls, contacts, history):
→ Read `references/rest-api.md`

**RTCM** (hold, mute, transfer, record, play announcements during active calls):
→ Read `references/rest-api.md` — section "Real-Time Call Management (RTCM)"
→ Combine with Push API webhooks for event-driven call manipulation

**Push API** (webhooks, IVR, call routing):
→ Read `references/push-api.md`

**SIP Stream Bridge** (raw audio streams, Twilio migration, custom STT/TTS):
→ Read `references/sip-stream-bridge.md`
→ Requires sipgate **trunking** account ([sipgatetrunking.de](https://sipgatetrunking.de)), not basic/team

### Authentication questions?
→ Read `references/authentication.md`

### Combining multiple APIs?
→ Read `references/common-patterns.md` (cross-API patterns with sequence diagrams)

---

## Quick Reference: Common Tasks

| Task | API | Endpoint / Method | Reference |
|------|-----|-------------------|-----------|
| Send SMS | REST | `POST /sessions/sms` | `references/rest-api.md` |
| Send fax | REST | `POST /sessions/fax` | `references/rest-api.md` |
| Click-to-call | REST | `POST /sessions/calls` | `references/rest-api.md` |
| Build voice bot | Flow | SDK event handlers | `references/flow-api.md` |
| IVR menu | Push | Gather + Dial XML | `references/push-api.md` |
| Call history | REST | `GET /v3/events` | `references/rest-api.md` |
| Manage contacts | REST | `/contacts` CRUD | `references/rest-api.md` |
| OAuth2 login | REST | Authorization Code Flow | `references/authentication.md` |
| Call screening | Push | newCall → Reject XML | `references/push-api.md` |
| Voicemail transcription | REST | `POST /v3/events/{id}/transcribe` | `references/rest-api.md` |
| **Hold/mute/transfer active call** | **REST (RTCM)** | `/v2/calls/{callId}/*` | `references/rest-api.md` |
| **Play announcement in call** | **REST (RTCM)** | `POST /v2/calls/{callId}/announcements` | `references/rest-api.md` |
| **Record active call** | **REST (RTCM)** | `PUT /v2/calls/{callId}/recording` | `references/rest-api.md` |
| Raw audio streaming | SIP Stream Bridge | Twilio Media Streams WS | `references/sip-stream-bridge.md` |
| Twilio migration | SIP Stream Bridge | Drop-in replacement | `references/sip-stream-bridge.md` |
| Custom STT/TTS pipeline | SIP Stream Bridge | Raw PCM audio access | `references/sip-stream-bridge.md` |

---

## Key Links

- [sipgate REST API Swagger](https://api.sipgate.com/v2/doc)
- [sipgate.io Developer Portal](https://en.sipgate.io)
- [Flow API LLM Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt)
- [Push API Reference](https://en.sipgate.io/push-api/api-references)
- [OAuth2 Authentication](https://en.sipgate.io/rest-api/authentication)
- [OAuth2 Scopes](https://en.sipgate.io/rest-api/oauth2-scopes)
- [Building Third-Party Apps](https://en.sipgate.io/rest-api/building-a-third-party-application)
- [Flow SDK (npm)](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)
- [PAT Settings](https://app.sipgate.com/personal-access-token)
- [OAuth Example: Node.js](https://github.com/sipgate-io/sipgateio-oauth-node)
- [OAuth Example: Java](https://github.com/sipgate-io/sipgateio-oauth-java)
- [OAuth Example: Python](https://github.com/sipgate-io/sipgateio-oauth-python)
- [SIP Stream Bridge](https://sipgate.github.io/sipgate-sip-stream-bridge/)
- [SIP Stream Bridge GitHub](https://github.com/sipgate/sipgate-sip-stream-bridge)
- [sipgate Trunking](https://sipgatetrunking.de)
