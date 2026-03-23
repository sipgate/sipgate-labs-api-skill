# sipgate API Use Cases

This reference describes concrete use cases for sipgate APIs. Each use case explains what is being built, which APIs are needed, and the rough implementation steps.

> **Note**: Outbound call campaigns are intentionally excluded. Outbound calls via the Flow API require explicit activation by sipgate and are ethically sensitive.

---

## 1. AI Voice Bot / Receptionist

**What**: An AI-powered agent answers incoming phone calls, responds to questions using natural language, and transfers calls when needed.

**API**: [Flow API](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) — always use the [SDK (`@sipgate/ai-flow-sdk`)](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)

**Steps**:
1. Set up a Flow API application with shared secret authentication
2. Install the TypeScript SDK: `npm install @sipgate/ai-flow-sdk`
3. Handle `session_start` to greet the caller
4. Process `user_speak` events with your AI/LLM and respond via `speak` actions
5. Use `transfer` action to route to a human agent when needed

**Integration**: Optionally combine with REST API to look up caller info via contacts or history endpoints.

---

## 2. IVR Menu (Voice Menu)

**What**: "Press 1 for sales, press 2 for support..." — a classic interactive voice response menu using DTMF tones.

**API**: [Push API](https://en.sipgate.io/push-api/api-references) with Gather/DTMF actions

**Steps**:
1. Configure a webhook URL in [sipgate.io settings](https://en.sipgate.io/push-api/api-references)
2. On `newCall` event, respond with `<Gather>` XML to collect digits
3. On `onData` callback, read the `dtmf` parameter for the pressed digits
4. Respond with `<Dial>` to route to the appropriate destination
5. Handle timeout/invalid input with a fallback message via `<Play>`

---

## 3. Click-to-Call from Web App

**What**: A user clicks a button in your web application, their desk phone rings, and when answered, it connects to the target number.

**API**: [REST API](https://api.sipgate.com/v2/doc) — `POST /sessions/calls`

**Steps**:
1. Authenticate via OAuth2 (see `references/authentication.md`)
2. Get the user's device ID via `GET /devices` ([Docs](https://api.sipgate.com/v2/doc))
3. Initiate the call:
   ```bash
   curl -X POST "https://api.sipgate.com/v2/sessions/calls" \
     -H "Authorization: Bearer ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"deviceId": "e0", "callee": "+4915790123456", "callerId": "+4921112345"}'
   ```
4. The user's device rings first; once picked up, the callee is dialed

**Scope**: `sessions:calls:write`

---

## 4. SMS Sending

**What**: Send SMS messages for notifications, 2FA codes, appointment reminders, or customer communication.

**API**: [REST API](https://api.sipgate.com/v2/doc) — `POST /sessions/sms`

**Steps**:
1. Authenticate via OAuth2 (recommended) or PAT (see `references/authentication.md`)
   > ⚠️ PATs grant full account access — use OAuth2 for production.
2. Get the SMS-capable device ID via `GET /devices`
3. Send the SMS:
   ```bash
   curl -X POST "https://api.sipgate.com/v2/sessions/sms" \
     -H "Authorization: Bearer ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"smsId": "s0", "recipient": "+4915790123456", "message": "Your code: 123456"}'
   ```

**Scope**: `sessions:sms:write`

> Phone numbers must be in E.164 format (e.g., `+4915790123456`).

---

## 5. Fax Sending

**What**: Send a PDF document as a fax to a fax number.

**API**: [REST API](https://api.sipgate.com/v2/doc) — `POST /sessions/fax`

**Steps**:
1. Authenticate via OAuth2 (recommended) or PAT
   > ⚠️ PATs grant full account access — use OAuth2 for production.
2. Get the fax-capable device ID via `GET /devices`
3. Send the fax with the PDF as base64-encoded content:
   ```bash
   curl -X POST "https://api.sipgate.com/v2/sessions/fax" \
     -H "Authorization: Bearer ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"faxlineId": "f0", "recipient": "+4921112345", "filename": "invoice.pdf", "base64Content": "JVBERi0xLjQ..."}'
   ```

**Scope**: `sessions:fax:write`

---

## 6. CRM Integration / Call Logging

**What**: Automatically log all incoming and outgoing calls in your CRM or custom system, including caller info, duration, and timestamps.

**APIs**:
- [Push API](https://en.sipgate.io/push-api/api-references) — real-time call events
- [REST API](https://api.sipgate.com/v2/doc) — caller info lookup, history

**Steps**:
1. Set up Push API webhooks for `newCall`, `onAnswer`, and `onHangup`
2. On each event, extract `callId`, `from`, `to`, `direction`, `user`
3. Optionally look up the caller via REST API `GET /contacts`
4. Store the call record in your CRM
5. On `onHangup`, update with duration and final status

---

## 7. Call Screening / Blacklist

**What**: Automatically reject or redirect calls from unwanted numbers based on a blocklist.

**API**: [Push API](https://en.sipgate.io/push-api/api-references) — `newCall` event

**Steps**:
1. Configure Push API webhook
2. On `newCall`, check the `from` number against your blocklist
3. If blocked: respond with `<Reject reason="busy" />` XML
4. If allowed: respond with empty XML `<Response />` to let the call through
5. Optionally manage the blocklist via REST API `POST /blacklist/incoming` ([Docs](https://api.sipgate.com/v2/doc))

---

## 8. Missed Call Notification

**What**: Send an SMS or email notification when a call is missed (not answered).

**APIs**:
- [Push API](https://en.sipgate.io/push-api/api-references) — detect missed calls via `onHangup`
- [REST API](https://api.sipgate.com/v2/doc) — send SMS notification

**Steps**:
1. Set up Push API webhook for `onHangup`
2. Check the `cause` parameter — missed calls have specific cause codes
3. Use REST API `POST /sessions/sms` to send notification:
   ```bash
   curl -X POST "https://api.sipgate.com/v2/sessions/sms" \
     -H "Authorization: Bearer ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"smsId": "s0", "recipient": "+49170...", "message": "Missed call from +49211..."}'
   ```

---

## 9. Contact Sync

**What**: Synchronize contacts between sipgate and an external CRM or address book.

**API**: [REST API](https://api.sipgate.com/v2/doc) — Contacts endpoints

**Steps**:
1. Authenticate via OAuth2 with `contacts:read` and `contacts:write` scopes
2. Fetch sipgate contacts: `GET /contacts`
3. Compare with your external system
4. Create new contacts: `POST /contacts`
5. Update existing: `PUT /contacts/{contactId}`
6. Optionally import from CSV/vCard via dedicated endpoints

**Scopes**: `contacts:read`, `contacts:write`

---

## 10. Voicemail-to-Text

**What**: Retrieve voicemail messages and transcribe them to text for notification or archival.

**API**: [REST API](https://api.sipgate.com/v2/doc) — Voicemails + Transcription

**Steps**:
1. Enable voicemail transcription: `PUT /voicemails/{voicemailId}/transcription` with `{"enabled": true}`
2. Fetch voicemail events: `GET /v3/events?types=VOICEMAIL`
3. Read transcription from the event's `transcription` field
4. Alternatively trigger transcription: `POST /v3/events/{dataId}/transcribe`

**Scopes**: `voicemails:read`, `voicemails:write`, `events:read`

---

## 11. Dashboard / Analytics

**What**: Build a dashboard visualizing call statistics — call volume, missed calls, average duration, peak hours.

**API**: [REST API v3](https://api.sipgate.com/v2/doc) — `GET /events`

**Steps**:
1. Authenticate with `events:read` scope
2. Fetch events with filters:
   ```bash
   curl -H "Authorization: Bearer ACCESS_TOKEN" \
     "https://api.sipgate.com/v3/events?types=CALL&from=2024-01-01T00:00:00Z&limit=1000"
   ```
3. Aggregate by direction, status, time period
4. Visualize in your frontend (charts, tables)
5. Use pagination (`offset`, `limit`) for large datasets

---

## 12. "Login with sipgate" Web App

**What**: Use sipgate as an identity provider for your web application via OAuth2/OpenID Connect.

**API**: OAuth2 / OpenID Connect — [Authentication Docs](https://en.sipgate.io/rest-api/authentication)

**Steps**:
1. Register your OAuth2 client — [Guide](https://en.sipgate.io/rest-api/building-a-third-party-application)
2. Implement the Authorization Code Flow (see `references/getting-started.md`)
3. After token exchange, fetch user info: `GET /authorization/userinfo`
4. Create or update the user session in your application
5. Implement token refresh for long-lived sessions

**Example implementations**: [Node.js](https://github.com/sipgate-io/sipgateio-oauth-node), [Java](https://github.com/sipgate-io/sipgateio-oauth-java), [Python](https://github.com/sipgate-io/sipgateio-oauth-python)

---

## 13. Conference Calls

**What**: Set up and manage conference rooms where multiple participants can join a call.

**API**: [REST API](https://api.sipgate.com/v2/doc) — Conference room endpoints

**Steps**:
1. Create a conference room via the REST API
2. Share the conference number with participants
3. Optionally manage participants via RTCM endpoints
4. Monitor active calls: `GET /calls`

---

## 14. Phonebook App

**What**: Build a custom phonebook interface with sipgate contacts, with "Login with sipgate" for authentication.

**APIs**:
- OAuth2 for authentication — [Building Third-Party Apps](https://en.sipgate.io/rest-api/building-a-third-party-application)
- [REST API](https://api.sipgate.com/v2/doc) for contacts

**Steps**:
1. Implement "Login with sipgate" (see Use Case #12)
2. Fetch contacts: `GET /contacts` with `contacts:read` scope
3. Display contacts in your UI with search/filter
4. Allow editing: `PUT /contacts/{contactId}` with `contacts:write` scope
5. Support CSV/vCard import and export via dedicated endpoints

---

## SIP Stream Bridge Use Cases (15–20)

All SIP Stream Bridge use cases require a sipgate **trunking** account from [sipgatetrunking.de](https://sipgatetrunking.de) and the [SIP Stream Bridge](https://sipgate.github.io/sipgate-sip-stream-bridge/) Docker container. The bridge speaks the [Twilio Media Streams](https://www.twilio.com/docs/voice/media-streams) WebSocket protocol — any app built for Twilio Media Streams works unchanged.

### 15. Twilio Migration

**What**: Migrate an existing Twilio Media Streams application to sipgate's German infrastructure without changing any application code. Data stays in Germany (DSGVO/GDPR compliant).

**Steps**:
1. Get sipgate trunking account and SIP credentials
2. Configure `.env` with SIP credentials and your app's WebSocket URL
3. Run via Docker: `docker run --env-file .env --network host ghcr.io/sipgate/sipgate-sip-stream-bridge:latest`
4. Point your phone number to the bridge — existing app works unchanged

This works for ANY Twilio Media Streams application — see examples below.

---

### 16. AI Voice Assistant with OpenAI Realtime API

**What**: Build a phone-based AI assistant using OpenAI's Realtime API for low-latency, natural voice conversations. Caller speaks → audio streams to OpenAI → AI responds in real time.

**Reference project**: [speech-assistant-openai-realtime-api-node](https://github.com/twilio-samples/speech-assistant-openai-realtime-api-node) — originally built for Twilio, works with SIP Stream Bridge without code changes.

**Architecture**:
```
Caller → sipgate → SIP Stream Bridge → [Twilio Media Streams WS] → Your Server → [OpenAI Realtime API WS]
                                                                   ←              ←
```

**Steps**:
1. Clone the OpenAI Realtime API example project
2. Configure with your OpenAI API key and system prompt
3. Set `WS_TARGET_URL` in Bridge to point to the server
4. Caller dials your sipgate number → talks to OpenAI in real time

**Why not Flow API?** Use this when you specifically want OpenAI's Realtime API (or other streaming AI APIs) with direct audio access. Flow API is simpler if sipgate's built-in STT/TTS suffice.

---

### 17. Real-Time Call Transcription

**What**: Transcribe phone calls in real time using your choice of STT provider (Google Speech-to-Text, AWS Transcribe, Deepgram, Whisper, etc.).

**Reference projects** (originally for Twilio, work via Bridge):
- [Node.js realtime transcription](https://github.com/twilio/media-streams/tree/master/node/realtime-transcription)
- [Python realtime transcription](https://github.com/twilio/media-streams/tree/master/python/realtime-transcription)
- [Java realtime transcription](https://github.com/twilio/media-streams/tree/master/java/realtime-transcription)
- [Ruby/Rails transcription](https://github.com/twilio/media-streams/tree/master/ruby)

**Steps**:
1. Build a WebSocket server receiving Twilio Media Streams audio
2. Forward audio chunks to your STT provider
3. Process transcription results (log, display, trigger actions)
4. Run SIP Stream Bridge with `WS_TARGET_URL` pointing to your server

**Use cases**: Live captioning, compliance recording, agent assist, meeting notes.

---

### 18. Real-Time Keyword Detection / Call Monitoring

**What**: Monitor live calls for specific keywords or phrases (e.g., competitor names, compliance terms, escalation triggers) and take automated action.

**Reference project**: [Node.js keyword detection](https://github.com/twilio/media-streams/tree/master/node/keyword-detection)

**Steps**:
1. Stream audio through Bridge to your keyword detection server
2. Run STT and match against keyword dictionary
3. On match: trigger alerts, log events, notify supervisors, or inject audio

**Use cases**: Compliance monitoring, quality assurance, real-time agent coaching, fraud detection.

---

### 19. Conversational AI with Google Dialogflow

**What**: Connect phone calls to Google Dialogflow for intent-based conversational AI — callers interact with a Dialogflow agent via voice.

**Reference project**: [Node.js Dialogflow integration](https://github.com/twilio/media-streams/tree/master/node/connect-to-google-dialogflow)

**Steps**:
1. Create a Dialogflow agent with intents and fulfillments
2. Build a WebSocket server that bridges Media Streams audio ↔ Dialogflow Streaming API
3. Route calls through SIP Stream Bridge to your server
4. Dialogflow processes speech, returns responses as audio

**Why not Flow API?** Use this when you're already invested in Dialogflow's ecosystem or need its specific NLU capabilities.

---

### 20. Call Recording / Audio Archival

**What**: Record phone calls by saving raw audio streams to files or cloud storage for later playback, analysis, or compliance.

**Reference project**: [Java save audio](https://github.com/twilio/media-streams/tree/master/java/save-audio)

**Steps**:
1. Receive audio stream via Twilio Media Streams WebSocket
2. Decode mulaw/PCM audio frames
3. Write to WAV file or stream to cloud storage (S3, GCS, etc.)
4. Optionally process after recording (transcription, sentiment analysis)

**Use cases**: Compliance recording, quality assurance, training data collection, dispute resolution.

---

### When to Use SIP Stream Bridge vs Flow API

| Aspect | Flow API | SIP Stream Bridge |
|--------|----------|-------------------|
| **Abstraction** | High — events + actions | Low — raw audio streams |
| **Own STT/TTS** | No | Yes (any provider) |
| **OpenAI Realtime API** | No | Yes |
| **Twilio code compatible** | No | Yes (zero changes) |
| **Setup complexity** | Low | Medium (Docker + trunking) |
| **Account type** | sipgate basic/team | sipgate **trunking** |
| **Best for** | Quick AI voice bots | Custom audio pipelines, Twilio migration |

> **Note**: Outbound call campaigns are intentionally excluded. Outbound calls via the Flow API require explicit activation by sipgate and are ethically sensitive.

---

## Full-Stack Use Case (alle 4 APIs)

### 21. AI Call Center with Quality Assurance

**What**: A full AI-powered customer service center combining all four sipgate APIs, each in its unique strength: Push API for routing, REST/RTCM for call control, Flow API for AI conversation, SIP Stream Bridge for real-time audio analysis.

**APIs**: All four — [Push API](https://en.sipgate.io/push-api/api-references) + [REST API/RTCM](https://api.sipgate.com/v2/doc) + [Flow API](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) + [SIP Stream Bridge](https://sipgate.github.io/sipgate-sip-stream-bridge/)

> **Accounts**: Requires sipgate **basic/team** (Push, REST, Flow) AND sipgate **trunking** ([sipgatetrunking.de](https://sipgatetrunking.de)) for the SIP Stream Bridge.

**Flow**:
1. **Push API** receives `newCall` → check business hours, look up contact via REST API → play `<Gather>` DTMF menu
2. **Flow API** takes over (via `<Dial>` to Flow number): AI bot pre-qualifies the request, collects customer ID and context via speech
3. **REST/RTCM** on transfer to agent: start recording (`PUT /calls/{id}/recording`), play announcement (`POST /calls/{id}/announcements`)
4. **SIP Stream Bridge** streams audio in parallel to custom pipeline: live transcription, sentiment analysis, keyword detection ("cancellation", "lawyer", "complaint") → alerts to team lead
5. After call ends: Push API `onHangup` → REST API sends SMS summary, labels the call

**Why each API?**:
- **Push**: Only API that delivers incoming call event webhooks + XML-based routing
- **REST/RTCM**: Only API for contact management, active call manipulation (hold/record/announcements) and SMS
- **Flow**: Only API with built-in STT/TTS for natural speech conversation
- **SIP Stream Bridge**: Only way to access raw audio with custom STT/analysis stack

Full sequence diagram and implementation details: see `references/common-patterns.md` → Pattern 6.
