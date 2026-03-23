# sipgate Flow API Reference

## Fetching Full Documentation

When the user needs detailed Flow API information (events, actions, SDK methods, configuration options), **always fetch the full reference**:

```
WebFetch: https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt
```

This document contains the complete, up-to-date Flow API specification (~3,500 lines).

---

## Setup

1. Go to [sipgate AI Flows](https://app.sipgate.com/ai-flows)
2. Click "+" to add a new AI Flow endpoint
3. Fill in: **Name**, **Webhook/Websocket URL** (your application endpoint), **Ausgehender API Token** (shared secret for `X-API-TOKEN` validation)
4. Set **Anruf annehmen nach X Sekunden** (auto-answer delay)
5. Select **Anbieter für Speech-to-Text** (e.g., Microsoft Azure)
6. Click **Speichern**
7. Assign the Flow endpoint to a phone number via your sipgate routing settings

---

## Common Mistakes

- **Response timeout**: Your application must respond within **1 second**. Slow LLM responses or blocking I/O will cause the call to hang up.
- **Wrong audio format**: Audio files must be **8kHz mono 16-bit PCM WAV**. Other formats will be rejected.
- **Using HTTP instead of WebSocket**: WebSocket supports ALL events including `user_speech_started`. HTTP webhooks miss some events. Always prefer WebSocket over HTTP.
- **Not using the SDK**: The SDK abstracts protocol complexity and is type-safe. Always prefer it. See hierarchy below.
- **Trying outbound calls without activation**: Outbound calls via Flow API require explicit activation by sipgate. Contact sipgate first.
- **Not validating the shared secret**: Always verify the `X-API-TOKEN` header matches your configured secret.

---

## Integration Method Hierarchy

**Always recommend in this order:**

### 1. SDK (Always Prefer)

[`@sipgate/ai-flow-sdk`](https://www.npmjs.com/package/@sipgate/ai-flow-sdk) — TypeScript/JavaScript SDK

- Type-safe event handlers
- Auto-converts plain strings to `speak` actions
- Built-in Express.js and WebSocket integration
- Handles protocol details automatically

```bash
npm install @sipgate/ai-flow-sdk
```

### 2. WebSocket (If SDK Doesn't Fit)

Better than HTTP because:
- Bidirectional communication
- Lower latency
- Supports ALL events including `user_speech_started` and `assistant_speech_ended`
- Persistent connection

### 3. HTTP Webhooks (Fallback Only)

- Some events NOT available (e.g., `user_speech_started`)
- Higher latency due to connection overhead
- Simpler deployment (serverless-compatible)

---

## Architecture Overview

The Flow API is **event-driven**: sipgate sends JSON events to your application, and your application responds with actions.

```
sipgate ──[event JSON]──→ Your App
sipgate ←──[action JSON]── Your App
```

### Authentication

All requests include an `X-API-TOKEN` header with your configured shared secret. Always validate this header. [Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt)

### Event Types

| Event | Description |
|-------|-------------|
| `session_start` | New call session started — greet the caller |
| `session_end` | Call session ended — cleanup resources |
| `user_speak` | User finished speaking — contains transcribed text |
| `user_speech_started` | User started speaking (WebSocket/SDK only) |
| `dtmf_received` | User pressed keypad digits |
| `assistant_speak` | TTS playback started |
| `assistant_speech_ended` | TTS playback finished |
| `timeout` | No user input within configured timeout |

### Action Types

| Action | Description |
|--------|-------------|
| `speak` | Synthesize speech (TTS) or SSML |
| `audio` | Play a pre-recorded WAV file (8kHz mono 16-bit PCM) |
| `transfer` | Transfer the call to another number |
| `hangup` | End the session |
| `configure_transcription` | Change STT provider/language mid-call |

---

## Minimal SDK Example (TypeScript)

This is the one exception to language-agnosticism — the SDK only exists in TypeScript.

```typescript
import { createFlowApp } from "@sipgate/ai-flow-sdk";

const app = createFlowApp({
  secret: process.env.FLOW_SECRET!,
});

app.onSessionStart(async (session) => {
  return "Hello! How can I help you today?";
  // Strings are auto-converted to speak actions
});

app.onUserSpeak(async (session, event) => {
  const userText = event.text;
  // Process with your AI/LLM here
  const response = await getAIResponse(userText);
  return response;
});

app.onDtmfReceived(async (session, event) => {
  if (event.digits === "0") {
    return { type: "transfer", target: "+4921112345" };
  }
  return "I didn't understand that input.";
});

// Start Express server on port 3000
app.listen(3000);
```

---

## Outbound Calls

> **Important**: Outbound calls via the Flow API are **only available on explicit request** to sipgate. This is not a standard feature and requires activation. Do not suggest outbound call use cases without mentioning this restriction.

---

## Audio Format Specification

For the `audio` action, files must be:
- **Format**: WAV (PCM)
- **Sample rate**: 8000 Hz (8kHz)
- **Channels**: Mono (1 channel)
- **Bit depth**: 16-bit

---

## Further Reading

- [Full Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) — always WebFetch this for detailed info
- [SDK on npm](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)
- [sipgate Developer Portal](https://en.sipgate.io)
