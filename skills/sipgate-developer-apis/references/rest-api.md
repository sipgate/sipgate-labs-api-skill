# sipgate REST API Reference

Full Swagger documentation: [api.sipgate.com/v2/doc](https://api.sipgate.com/v2/doc)

---

## Common Mistakes

- **Phone number format**: All phone numbers must be in **E.164 format** (e.g., `+4915790123456`). Missing the `+` or country code causes errors.
- **Token expiry**: Access tokens expire after ~5 minutes. Always implement token refresh. See `references/authentication.md`.
- **Pagination**: Large result sets require pagination with `offset` and `limit`. Don't try to fetch everything at once.
- **v3 is experimental**: The v3 API may have breaking changes. Use v2 for all production applications. v3 is only for features not yet available in v2 (e.g., events, labels, groups).
- **Wrong session endpoint**: Use `/sessions/calls` for click-to-dial (not `/calls` which is RTCM).

---

## Base URLs

| Version | URL | Status |
|---------|-----|--------|
| v2 | `https://api.sipgate.com/v2` | Stable |
| v3 | `https://api.sipgate.com/v3` | Experimental |

---

## Authentication

See `references/authentication.md` for full details.

- **OAuth2** (recommended): `Authorization: Bearer <access_token>` — [Auth Docs](https://en.sipgate.io/rest-api/authentication)
- **Personal Access Token** (⚠️ prototyping only): `Authorization: Basic <base64(tokenId:token)>` — [Create PAT](https://app.sipgate.com/personal-access-token)

> **WARNING**: Personal Access Tokens grant full account access. Never commit them to repositories or use in client-side code. Use only for personal scripts, CLI tools, and prototyping. For production applications, always use OAuth2.

OAuth2 Scopes: [Full Scope Reference](https://en.sipgate.io/rest-api/oauth2-scopes)

---

## Communication & Sessions

### Initiate a Call (Click-to-Dial)

`POST /v2/sessions/calls` — Scope: `sessions:calls:write`

```bash
curl -X POST "https://api.sipgate.com/v2/sessions/calls" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "deviceId": "e0",
    "callee": "+4915790123456",
    "callerId": "+4921112345"
  }'
```

Your device rings first. When you pick up, the callee is dialed.

### Send SMS

`POST /v2/sessions/sms` — Scope: `sessions:sms:write`

```bash
curl -X POST "https://api.sipgate.com/v2/sessions/sms" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "smsId": "s0",
    "recipient": "+4915790123456",
    "message": "Hello from sipgate!"
  }'
```

### Send Fax

`POST /v2/sessions/fax` — Scope: `sessions:fax:write`

```bash
curl -X POST "https://api.sipgate.com/v2/sessions/fax" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "faxlineId": "f0",
    "recipient": "+4921112345",
    "filename": "document.pdf",
    "base64Content": "JVBERi0xLjQ..."
  }'
```

---

## Real-Time Call Management (RTCM)

Manipulate currently active calls in real time. All endpoints require scope: `rtcm:write`. [RTCM Overview](https://www.sipgate.io/funktionen/rtcm) | [Swagger Docs](https://api.sipgate.com/v2/doc)

> **Note**: RTCM works on **established** calls only. Use `callId` from `GET /v2/calls` or from [Push API](https://en.sipgate.io/push-api/api-references) webhooks.

### List Active Calls

`GET /v2/calls` — Returns only established calls (not ringing or voicemail recordings).

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/calls"
```

### Start Channel Call (Beta)

`POST /v2/calls` — Start a new call from a device. Differs from `POST /sessions/calls` (click-to-dial): here the target rings directly, no intermediate step.

> **Note**: This endpoint is in beta. Established calls won't appear in history.

```bash
curl -X POST "https://api.sipgate.com/v2/calls" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "deviceId": "e0",
    "targetNumber": "+4915790123456",
    "callerId": "+4921112345",
    "channelId": "optional-channel-uuid"
  }'
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `deviceId` | Yes | Local device ID (e.g., `e0`) |
| `targetNumber` | Yes | Phone number of remote party (E.164) |
| `callerId` | No | Caller ID displayed to remote party |
| `channelId` | No | Channel ID, defaults to device's default channel |

### Manage Active Calls

| Action | Method | Endpoint | Request Body |
|--------|--------|----------|-------------|
| Hang up | `DELETE` | `/v2/calls/{callId}` | — |
| Transfer | `POST` | `/v2/calls/{callId}/transfer` | `{ phoneNumber, attended, callerId? }` |
| Send DTMF | `POST` | `/v2/calls/{callId}/dtmf` | `{ sequence }` |
| Hold/Resume | `PUT` | `/v2/calls/{callId}/hold` | `{ value: true/false }` |
| Mute/Unmute | `PUT` | `/v2/calls/{callId}/muted` | `{ value: true/false }` |
| Record | `PUT` | `/v2/calls/{callId}/recording` | `{ value: true/false, announcement? }` |
| Play announcement | `POST` | `/v2/calls/{callId}/announcements` | `{ url }` |

### Transfer Call

`POST /v2/calls/{callId}/transfer`

```bash
curl -X POST "https://api.sipgate.com/v2/calls/{callId}/transfer" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "phoneNumber": "+4915799912345",
    "attended": false,
    "callerId": "+492110012345"
  }'
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `phoneNumber` | Yes | Transfer target number (E.164) |
| `attended` | Yes | `true` = attended (warm), `false` = blind (cold) transfer |
| `callerId` | No | Override caller ID for the transfer |

### Play Announcement

`POST /v2/calls/{callId}/announcements` — Play an audio file to all participants during an active call.

```bash
curl -X POST "https://api.sipgate.com/v2/calls/{callId}/announcements" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/announcement.wav"
  }'
```

> **Audio format**: Mono 16-bit PCM WAV, 8kHz sampling rate. Convert from MP3: `mpg123 --rate 8000 --mono -w output.wav input.mp3`

### Call Recording

`PUT /v2/calls/{callId}/recording`

```bash
curl -X PUT "https://api.sipgate.com/v2/calls/{callId}/recording" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "value": true,
    "announcement": true
  }'
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `value` | Yes | `true` = start, `false` = stop recording |
| `announcement` | No | Announce recording to all participants. Cannot be `false` for sipgate neo accounts. |

### Hold / Mute

```bash
# Hold
curl -X PUT "https://api.sipgate.com/v2/calls/{callId}/hold" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "value": true }'

# Mute
curl -X PUT "https://api.sipgate.com/v2/calls/{callId}/muted" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "value": true }'
```

> **Hold**: Returns `409 Conflict` if call is not yet established.

### Send DTMF

```bash
curl -X POST "https://api.sipgate.com/v2/calls/{callId}/dtmf" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "sequence": "123456" }'
```

### Common RTCM Pattern: Push API + RTCM

Use Push API webhooks to get the `callId`, then manipulate the call via RTCM:

1. Receive `newCall` webhook → extract `callId`
2. Use `callId` to hold, record, transfer, or play announcements
3. Receive `onHangup` webhook → stop recording, log result

See `references/common-patterns.md` for full cross-API patterns.

---

## History & Events (v3)

### List Events

`GET /v3/events` — Scope: `events:read`

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v3/events?types=CALL&directions=INCOMING&limit=50&offset=0"
```

**Query Parameters:**

| Parameter | Values | Description |
|-----------|--------|-------------|
| `types` | `CALL`, `VOICEMAIL`, `SMS`, `FAX` | Filter by event type |
| `directions` | `INCOMING`, `OUTGOING` | Filter by direction |
| `status` | `SUCCESS`, `FAILURE` | Filter by status |
| `directories` | `INBOX`, `ARCHIVE` | Filter by directory |
| `starred` | `STARRED`, `UNSTARRED` | Filter by starred |
| `read` | `READ`, `UNREAD` | Filter by read state |
| `limit` | 1–1000 (default: 10) | Results per page |
| `offset` | number | Pagination offset |
| `from` | ISO8601 date | Start date filter |
| `to` | ISO8601 date | End date filter |
| `phonenumber` | E.164 number | Filter by phone number |
| `labelIds` | comma-separated IDs | Filter by labels |
| `connectionIds` | comma-separated IDs | Filter by connections |

### Batch Label Operations

- `POST /v3/events/batch-attach-label` — Scope: `events:write`
- `POST /v3/events/batch-detach-label` — Scope: `events:write`

### Transcribe Voicemail

`POST /v3/events/{dataId}/transcribe` — Scope: `voicemails:write`

---

## Account & Users

### Get Account Data

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/account"
```
Scope: `account:read`

### Get Current User Info

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/authorization/userinfo"
```

### Get Account Balance

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/balance"
```
Scope: `balance:read`

### Update User Name

`PUT /v2/users/{userId}/name` — Scope: `users:write`

```json
{
  "firstName": "John",
  "lastName": "Doe"
}
```

---

## Contacts

All contact endpoints: Scopes `contacts:read` / `contacts:write`

| Action | Method | Endpoint |
|--------|--------|----------|
| List contacts | `GET` | `/v2/contacts` |
| Create contact | `POST` | `/v2/contacts` |
| Update contact | `PUT` | `/v2/contacts/{contactId}` |
| Delete contact | `DELETE` | `/v2/contacts/{contactId}` |
| Import CSV | `POST` | `/v2/contacts/csv/prepare` |
| Import vCard | `POST` | `/v2/contacts/import/vcard` |
| Export CSV | `GET` | `/v2/contacts/export/csv` |
| Export vCard | `GET` | `/v2/contacts/export/vcard` |

---

## Devices

### Getting the userId

Many device endpoints require a `userId` parameter (e.g. `w0`). This value is **not** the same as the OIDC `providerAccountId` from the OAuth token (which has a format like `f:2e724444-...:2778579`).

The correct `userId` comes exclusively from:

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/authorization/userinfo"
```

Use the `sub` field from the response (e.g. `w0`) as the `userId` for all device API calls.

### List Devices (v3)

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v3/devices?userId=w0"
```
Scope: `devices:read`

> **Hinweis**: Es gibt auch `GET /v2/{userId}/devices`, aber dieser Endpoint liefert **kein** `sipCredentials`-Feld zurück. Für SIP/WebRTC-Credentials muss zwingend `/v3/devices` verwendet werden.

### SIP Credentials

When fetching REGISTER-type devices via `GET /v3/devices`, the response includes SIP credentials for each register device. This is **not documented in the Swagger** but is essential for SIP/WebRTC integrations.

Response structure for REGISTER devices:
```json
{
  "type": "REGISTER",
  "id": "e0",
  "alias": "My SIP Phone",
  "sipCredentials": {
    "username": "1234567e0",
    "password": "secretpassword",
    "sipServer": "sipgate.de",
    "outboundProxy": "sipconnect.sipgate.de",
    "sipServerWebsocketUrl": "wss://sipgate.de/ws"
  }
}
```

| Field | Description |
|-------|-------------|
| `username` | SIP username for registration |
| `password` | SIP password |
| `sipServer` | SIP server hostname |
| `outboundProxy` | Outbound proxy for SIP traffic |
| `sipServerWebsocketUrl` | WebSocket URL for WebRTC/browser-based SIP |

### Reset SIP Password

`POST /v2/devices/{deviceId}/credentials/password` — Scope: `devices:write`

Resets the SIP password for a register device. Returns the new password:

```bash
curl -X POST "https://api.sipgate.com/v2/devices/e0/credentials/password" \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

Response: `{ "password": "newpassword123" }`

### Create Register Device

`POST /v3/devices/register` — Scope: `devices:write`

Creates a new SIP register device with SIP credentials:

```bash
curl -X POST "https://api.sipgate.com/v3/devices/register" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"alias": "My SIP Phone", "owner": "w0"}'
```

The response includes the full `sipCredentials` object. Use these credentials to register a SIP phone, softphone, or WebRTC client against sipgate's SIP infrastructure.

### Device Operations (v2)

| Action | Method | Endpoint | Scope |
|--------|--------|----------|-------|
| Get device | `GET` | `/v2/devices/{deviceId}` | `devices:read` |
| Update device | `PUT` | `/v2/devices/{deviceId}` | `devices:write` |
| Delete device | `DELETE` | `/v2/devices/{deviceId}` | `devices:write` |
| Get caller ID | `GET` | `/v2/devices/{deviceId}/callerid` | `devices:callerid:read` |
| Set caller ID | `PUT` | `/v2/devices/{deviceId}/callerid` | `devices:callerid:write` |

---

## Voicemails

Scopes: `voicemails:read` / `voicemails:write`

| Action | Method | Endpoint |
|--------|--------|----------|
| List voicemails | `GET` | `/v3/voicemails` |
| Rename voicemail | `PUT` | `/v3/voicemails/{voicemailId}/alias` |
| Set PIN | `PUT` | `/v3/voicemails/{voicemailId}/pin` |
| Enable transcription | `PUT` | `/v3/voicemails/{voicemailId}/transcription` |

Enable automatic transcription:
```json
{ "enabled": true }
```

---

## Blacklist

Scopes: `blacklist:read` / `blacklist:write`

| Action | Method | Endpoint |
|--------|--------|----------|
| Get incoming blacklist | `GET` | `/v2/blacklist/incoming` |
| Add entry | `POST` | `/v2/blacklist/incoming` |
| Remove entry | `DELETE` | `/v2/blacklist/incoming/{phoneNumber}` |

---

## Groups

Scopes: `groups:read` / `groups:write`

| Action | Method | Endpoint |
|--------|--------|----------|
| List groups | `GET` | `/v3/groups` |
| Create group | `POST` | `/v3/groups` |
| Delete group | `DELETE` | `/v3/groups/{groupId}` |
| Update alias | `PUT` | `/v3/groups/{groupId}/alias` |
| Update members | `PUT` | `/v3/groups/{groupId}/members` |

---

## Labels

Scopes: `labels:read` / `labels:write`

| Action | Method | Endpoint |
|--------|--------|----------|
| List labels | `GET` | `/v3/labels` |
| Create label | `POST` | `/v3/labels` |
| Get label | `GET` | `/v3/labels/{labelId}` |
| Update label | `PUT` | `/v3/labels/{labelId}` |
| Delete label | `DELETE` | `/v3/labels/{labelId}` |

---

## Channels

Scope: `channels:read`

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/channels"
```

---

## Forwardings

| Action | Method | Endpoint | Scope |
|--------|--------|----------|-------|
| Get forwardings | `GET` | `/v2/{userId}/phonelines/{phonelineId}/forwardings` | `phonelines:forwardings:read` |
| Update forwardings | `PUT` | `/v2/{userId}/phonelines/{phonelineId}/forwardings` | `phonelines:forwardings:write` |

---

## Routings

| Action | Method | Endpoint |
|--------|--------|----------|
| Create routing | `PUT` | `/v2/routings` |
| Delete routing | `DELETE` | `/v2/routings` |
| Route to channel | `PUT` | `/v2/routings/channel` |

Create routing example:
```json
{
  "e164Numbers": ["+49211123456"],
  "extension": "g3"
}
```

---

## Response Types

### History Entry Types

```
HistoryEntryType:    CALL | VOICEMAIL | SMS | FAX
Direction:           INCOMING | OUTGOING
Status:              SUCCESS | FAILURE
ReadState:           READ | UNREAD
Directory:           INBOX | ARCHIVE
Starred:             STARRED | UNSTARRED
```

### Call Status Types

```
SUCCESS | FAILURE | REJECTED | REJECTED_DND | VOICEMAIL_NO_MESSAGE | BUSY_ON_BUSY | BUSY | MISSED
```

### Fax Status Types

```
PENDING | SENDING | FAILED | SENT | SCHEDULED
```

### Device/Endpoint Types

```
SMS | REGISTER | EXTERNAL | FAX | MOBILE | GROUP | VOICEMAIL | PHONELINE | USER | NUMBER | CONFERENCEROOM | CHANNEL | ACD | DESKTOP_APP | MOBILE_APP
```

---

## Error Handling

Standard error response format:

```json
{
  "name": "PARAMETER_INVALID",
  "message": "Description of the error"
}
```

### Common Error Types

| Error | Description |
|-------|-------------|
| `PARAMETER_INVALID` | Invalid request parameter |
| `REQUEST_NOT_AUTHENTICATED` | Missing or invalid auth token |
| `REQUEST_NOT_AUTHORIZED` | Insufficient permissions/scopes |
| `RATE_LIMIT_EXCEEDED` | Too many requests |
| `RESOURCE_NOT_FOUND` | Requested resource doesn't exist |
| `RESOURCE_ALREADY_EXISTS` | Duplicate resource |
| `INVALID_JSON` | Malformed request body |
| `FORBIDDEN` | Action not allowed |
| `INTERNAL_ERROR` | Server-side error |

---

## Further Reading

- [Swagger Documentation](https://api.sipgate.com/v2/doc)
- [Authentication](https://en.sipgate.io/rest-api/authentication)
- [OAuth2 Scopes](https://en.sipgate.io/rest-api/oauth2-scopes)
- [Building Third-Party Applications](https://en.sipgate.io/rest-api/building-a-third-party-application)
