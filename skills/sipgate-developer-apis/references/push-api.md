# sipgate Push API Reference

Official documentation: [Push API Reference](https://en.sipgate.io/push-api/api-references)

---

## Common Mistakes

- **Wrong Content-Type**: Responses must have `Content-Type: application/xml`. If missing, sipgate ignores your response.
- **Not parsing URL-encoded body**: sipgate sends event data as `application/x-www-form-urlencoded`, not JSON.
- **Webhook timeout**: Your server must respond quickly. Slow responses cause sipgate to skip your actions.
- **HTTP instead of HTTPS**: Always use HTTPS for production webhook URLs.
- **Missing callback URLs**: Actions like `<Gather>` require an `onData` callback URL. Without it, collected digits are lost.
- **Wrong audio format for `<Play>`**: Must be 8kHz mono 16-bit PCM WAV.

---

## Setup

1. Log in to your sipgate account
2. Go to [sipgate.io Webhooks](https://app.sipgate.com/io/hooks)
3. Configure your incoming/outgoing webhook URLs (must be publicly accessible)
4. Under **Quellen**, select which phone numbers/extensions trigger webhooks

Details: [Push API Setup](https://en.sipgate.io/push-api/api-references)

---

## How It Works

When a call involves your sipgate account, sipgate sends HTTP POST requests to your webhook URL with call details. Your server responds with XML containing the desired actions.

```
Caller ──→ sipgate ──[POST webhook]──→ Your Server
                    ←──[XML response]──
```

---

## Event Types

### newCall

Triggered when a new call starts (incoming or outgoing).

**Parameters** (sent as URL-encoded form data):

| Parameter | Description | Example |
|-----------|-------------|---------|
| `event` | Always `newCall` | `newCall` |
| `direction` | `in` or `out` | `in` |
| `from` | Caller number (E.164) | `+4915790123456` |
| `to` | Called number (E.164) | `+4921112345` |
| `callId` | Unique call identifier | `12345ABC` |
| `user[]` | sipgate user name(s) | `John Doe` |
| `userId[]` | sipgate user ID(s) | `w0` |
| `fullUserId[]` | Full user ID(s) | `1234567w0` |
| `originalCallId` | Original call ID (if forwarded) | `12345ABC` |

**Testing with cURL:**

```bash
curl -X POST "http://localhost:3000/webhook" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "event=newCall&direction=in&from=%2B4915790123456&to=%2B4921112345&callId=12345ABC&user[]=John+Doe&userId[]=w0"
```

### onAnswer

Triggered when a call is answered.

| Parameter | Description |
|-----------|-------------|
| `event` | Always `onAnswer` |
| `callId` | Call identifier |
| `user` | User who answered |
| `userId` | User ID |
| `fullUserId` | Full user ID |
| `from` | Caller number |
| `to` | Called number |
| `direction` | `in` or `out` |
| `diverting` | Number that diverted the call (if applicable) |
| `answeringNumber` | Number that answered |

### onHangup

Triggered when a call ends.

| Parameter | Description |
|-----------|-------------|
| `event` | Always `onHangup` |
| `callId` | Call identifier |
| `cause` | Hangup reason (e.g., `normalClearing`, `busy`, `cancel`, `noAnswer`) |
| `from` | Caller number |
| `to` | Called number |
| `direction` | `in` or `out` |
| `answeringNumber` | Number that answered (if call was answered) |

### onData

Triggered after a `<Gather>` action completes (user pressed digits).

| Parameter | Description |
|-----------|-------------|
| `event` | Always `onData` |
| `callId` | Call identifier |
| `dtmf` | The digits the caller pressed |

---

## XML Response Actions

Your webhook must respond with XML. An empty response lets the call proceed normally:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response />
```

### Dial

Redirect the call to one or more targets. Optionally set a custom caller ID.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Dial callerId="+4921112345" anonymous="false">
    <Number>+4915790123456</Number>
  </Dial>
</Response>
```

**Multiple targets** (first to pick up gets connected):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Dial>
    <Number>+4915790123456</Number>
    <Number>+4915790654321</Number>
  </Dial>
</Response>
```

**Attributes:**
- `callerId` — number shown to the callee
- `anonymous` — `true` to hide caller ID

### Play

Play an audio file to the caller. File must be **8kHz mono 16-bit PCM WAV**.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Play>
    <Url>https://example.com/greeting.wav</Url>
  </Play>
</Response>
```

### Gather

Collect DTMF digits (keypad input) from the caller.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Gather onData="https://example.com/webhook/dtmf" maxDigits="1" timeout="5000">
    <Play>
      <Url>https://example.com/menu.wav</Url>
    </Play>
  </Gather>
</Response>
```

**Attributes:**
- `onData` — callback URL receiving the collected digits (required)
- `maxDigits` — maximum digits to collect before sending
- `timeout` — milliseconds to wait for input

The `onData` callback receives a POST with the `dtmf` parameter containing the pressed digits.

### Reject

Reject the call with a reason.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Reject reason="busy" />
</Response>
```

**Reasons:**
- `busy` — caller hears busy signal
- `rejected` — call is rejected

### Hangup

Hang up the call.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Hangup />
</Response>
```

---

## RTCM Integration

You can use the `callId` from Push API webhooks to manipulate active calls via the REST API's [Real-Time Call Management](https://api.sipgate.com/v2/doc) endpoints:

- `PUT /calls/{callId}/hold` — hold/resume
- `PUT /calls/{callId}/muted` — mute/unmute
- `POST /calls/{callId}/transfer` — transfer
- `POST /calls/{callId}/dtmf` — send DTMF
- `DELETE /calls/{callId}` — hang up

---

## Example: Simple IVR Menu

This pseudocode shows a complete IVR flow:

```
# On newCall event:
→ Respond with Gather (play menu audio, collect 1 digit)

# On onData callback:
→ If dtmf == "1": Dial sales number
→ If dtmf == "2": Dial support number
→ Otherwise: Play error message, Gather again
```

**newCall response:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Gather onData="https://example.com/webhook/menu" maxDigits="1" timeout="10000">
    <Play>
      <Url>https://example.com/menu-prompt.wav</Url>
    </Play>
  </Gather>
</Response>
```

**onData response (digits received):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Dial>
    <Number>+49211SALES</Number>
  </Dial>
</Response>
```

---

## Further Reading

- [Push API Documentation](https://en.sipgate.io/push-api/api-references)
- [sipgate.io Developer Portal](https://en.sipgate.io)
- [REST API RTCM Endpoints](https://api.sipgate.com/v2/doc)
