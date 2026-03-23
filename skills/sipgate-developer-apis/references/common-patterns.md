# Cross-API Integration Patterns

These patterns show how to combine sipgate APIs for common real-world scenarios. Each includes a sequence diagram and implementation guidance.

---

## Pattern 1: Call Screening + Contact Lookup (Push → REST)

**Scenario**: When a call comes in, look up the caller in your contacts. If known, let through. If unknown or blacklisted, reject.

```
Caller ──→ sipgate ──[newCall]──→ Your Server
                                      │
                                      ├─→ REST API: GET /contacts?search=from
                                      │
                                      ├─→ Known? → <Response /> (let through)
                                      └─→ Unknown? → <Reject reason="busy" />
```

**Implementation**:
1. Receive `newCall` webhook with `from` number
2. Look up caller via REST API `GET /contacts` ([Docs](https://api.sipgate.com/v2/doc))
3. Decision logic:
   - Contact found → respond with empty `<Response />`
   - Blacklisted → respond with `<Reject reason="busy" />`
   - Unknown → optionally play a message, then forward

**Auth**: Push API (webhook URL) + REST API (OAuth2 with `contacts:read` scope)

---

## Pattern 2: AI Receptionist (Flow → REST)

**Scenario**: An AI voice bot greets callers, understands their intent, looks up information via REST API, and either answers questions or transfers to the right person.

```
Caller ──→ sipgate ──[session_start]──→ Flow App
                                            │
           sipgate ←──[speak: greeting]─────┘

Caller speaks ──→ [user_speak] ──→ Flow App
                                       │
                                       ├─→ LLM: Understand intent
                                       ├─→ REST API: GET /contacts (lookup)
                                       │
                                       ├─→ Answer? → [speak: response]
                                       └─→ Transfer? → [transfer: +49...]
```

**Implementation**:
1. Use [Flow SDK](https://www.npmjs.com/package/@sipgate/ai-flow-sdk) (always prefer SDK)
2. On `session_start`: greet the caller
3. On `user_speak`: send text to your LLM for intent classification
4. If caller asks about a contact: REST API `GET /contacts` for lookup
5. Respond with information or `transfer` action

**Auth**: Flow API (shared secret) + REST API (OAuth2 recommended; PAT only for prototyping — PATs grant full account access)

---

## Pattern 3: Click-to-Call + Logging (REST → Push → REST)

**Scenario**: User clicks "Call" in your web app. The call is initiated via REST API, and call events are logged via Push API webhooks.

```
Web App ──[POST /sessions/calls]──→ REST API ──→ sipgate
                                                     │
User's phone rings ←────────────────────────────────┘
User picks up → sipgate dials callee
                                                     │
sipgate ──[newCall]──→ Your Server (log start)
sipgate ──[onAnswer]──→ Your Server (log answer)
sipgate ──[onHangup]──→ Your Server (log end + duration)
                            │
                            └─→ REST API: GET /contacts (enrich log)
```

**Implementation**:
1. Web app calls REST API `POST /sessions/calls` ([Docs](https://api.sipgate.com/v2/doc))
2. Push API webhooks receive `newCall`, `onAnswer`, `onHangup` events
3. Log each event with timestamps in your system
4. On `onHangup`: calculate duration, optionally enrich with contact info via REST API
5. Store complete call record in your CRM/database

**Auth**: REST API (OAuth2 with `sessions:calls:write`, `contacts:read`) + Push API (webhook)

---

## Pattern 4: IVR-to-AI Handoff (Push → Flow)

**Scenario**: A Push API IVR menu collects the caller's initial choice, then transfers to a Flow API AI agent for deeper conversation.

```
Caller ──→ sipgate ──[newCall]──→ Push Webhook
                                      │
           sipgate ←──[Gather]────────┘

Caller presses "2" ──→ [onData: dtmf=2] ──→ Push Webhook
                                                   │
              sipgate ←──[Dial: flow-number]───────┘

sipgate ──[session_start]──→ Flow App (AI takes over)
```

**Implementation**:
1. Push API webhook receives `newCall`
2. Respond with `<Gather>` to play menu and collect DTMF ([Push API Docs](https://en.sipgate.io/push-api/api-references))
3. On `onData` callback, route based on digit:
   - "1" → `<Dial>` to sales
   - "2" → `<Dial>` to Flow API number (AI agent)
   - "0" → `<Dial>` to operator
4. Flow API handles AI conversation from `session_start` ([Flow SDK](https://www.npmjs.com/package/@sipgate/ai-flow-sdk))

**Auth**: Push API (webhook) + Flow API (shared secret)

---

## Pattern 5: SMS on Missed Call (Push → REST)

**Scenario**: When a call is missed (not answered), automatically send an SMS to the caller apologizing and offering a callback.

```
Caller ──→ sipgate ──[newCall]──→ Your Server (note caller)

Call not answered...

sipgate ──[onHangup: cause=noAnswer]──→ Your Server
                                            │
                                            ├─→ Check: cause == missed?
                                            │
                                            └─→ REST API: POST /sessions/sms
                                                 "Sorry we missed your call..."
```

**Implementation**:
1. Push API webhook receives `onHangup` event
2. Check the `cause` parameter:
   - `noAnswer`, `cancel` → missed call
   - `normalClearing` → normal hangup (not missed)
3. If missed, send SMS via REST API:
   ```bash
   curl -X POST "https://api.sipgate.com/v2/sessions/sms" \
     -H "Authorization: Bearer ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "smsId": "s0",
       "recipient": "+4915790123456",
       "message": "Sorry we missed your call. We will call you back shortly."
     }'
   ```
4. Optionally create a callback task in your system

**Auth**: Push API (webhook) + REST API (OAuth2 with `sessions:sms:write`)

---

## Pattern 6: AI Call Center with Quality Assurance (Push → REST/RTCM → Flow → SIP Stream Bridge)

**Scenario**: A full AI-powered customer service center combining all four sipgate APIs. Each API has a clearly distinct, non-interchangeable role.

> **Note**: This pattern requires both a sipgate **basic/team** account (for Flow, REST, Push) and a sipgate **trunking** account ([sipgatetrunking.de](https://sipgatetrunking.de)) for the SIP Stream Bridge.

```
Caller ──→ sipgate ──[newCall]──→ Push API (Routing)
                                       │
                                       ├─→ REST API: GET /contacts (Caller Lookup)
                                       ├─→ Business Hours Check
                                       │
                                       ├─→ Outside? → <Reject> + REST: POST /sessions/sms
                                       └─→ Inside? → <Gather> DTMF Menu
                                                              │
                                                              ├─→ "1" (General) → <Dial> Flow Number
                                                              └─→ "2" (Agent) → <Dial> Agent Group

                    ┌──────────────────────────────────────────┘
                    ▼
sipgate ──[session_start]──→ Flow API (AI Pre-Qualification)
                                  │
                                  ├─→ Bot greets, asks about request
                                  ├─→ REST API: GET /contacts (load context)
                                  ├─→ Bot collects: customer ID, request, urgency
                                  │
                                  ├─→ Self-resolved? → [speak: answer] → [hangup]
                                  └─→ Agent needed? → [transfer: +49... Agent Group]
                                                           │
                    ┌──────────────────────────────────────┘
                    ▼
Agent conversation running via sipgate
    │
    ├─→ REST/RTCM: PUT /calls/{id}/recording (start recording)
    ├─→ REST/RTCM: POST /calls/{id}/announcements ("This call is being recorded")
    │
    └─→ SIP Stream Bridge (parallel audio stream)
              │
              ├─→ Custom STT → live transcription for agent dashboard
              ├─→ Sentiment analysis (caller mood)
              └─→ Keyword detection ("cancellation", "lawyer", "complaint")
                        │
                        └─→ Alert to team lead on critical keywords

                    ┌──────────────────────────────────────┘
                    ▼
Call ends
    │
    ├─→ Push API: onHangup → log duration + status
    ├─→ REST API: POST /sessions/sms (summary to customer)
    ├─→ REST API: POST /v3/events/batch-attach-label (label call)
    └─→ Transcript + sentiment report → CRM/database
```

**API Roles**:

| API | Role | Why this API? |
|-----|------|---------------|
| **Push API** | Inbound routing, DTMF menu, event logging | Only API that delivers incoming call event webhooks + allows XML-based routing |
| **REST API + RTCM** | Contact lookup, hold/transfer/record/announcements, SMS, labeling | Only API for contact management, active call manipulation, and SMS sending |
| **Flow API** | AI pre-qualification via speech | Only API with built-in STT/TTS for natural speech conversation |
| **SIP Stream Bridge** | Real-time audio for custom analysis pipeline | Only way to access raw audio with custom STT/sentiment/keyword detection |

**Implementation**:
1. **Push API Server**: Configure webhook URL in [sipgate.io Webhooks](https://app.sipgate.com/io/hooks), implement business hours logic + `<Gather>` DTMF menu ([Push API Docs](https://en.sipgate.io/push-api/api-references))
2. **Flow API App**: Install [Flow SDK](https://www.npmjs.com/package/@sipgate/ai-flow-sdk), build AI bot with LLM integration for pre-qualification, `transfer` action when needed
3. **REST/RTCM Integration**: OAuth2 authentication with scopes `contacts:read`, `rtcm:write`, `sessions:sms:write`, `events:write` ([Swagger Docs](https://api.sipgate.com/v2/doc))
4. **SIP Stream Bridge**: Deploy Docker container with SIP credentials from trunking account, WebSocket target pointing to your analysis pipeline ([Bridge Docs](https://sipgate.github.io/sipgate-sip-stream-bridge/))
5. **Agent Dashboard**: Receives live transcript + sentiment from Bridge stream, displays AI summary from Flow phase, Push events for call status

**Auth**:
- Push API: Webhook URL
- REST API: OAuth2 (`contacts:read`, `rtcm:write`, `sessions:sms:write`, `events:read`, `events:write`, `labels:read`)
- Flow API: Shared Secret (X-API-TOKEN)
- SIP Stream Bridge: SIP credentials (trunking account)

---

## Further Reading

- [Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) — always WebFetch for full details
- [REST API Swagger](https://api.sipgate.com/v2/doc)
- [Push API Documentation](https://en.sipgate.io/push-api/api-references)
- [Flow SDK on npm](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)
