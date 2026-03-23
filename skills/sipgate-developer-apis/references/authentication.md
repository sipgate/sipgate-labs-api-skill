# sipgate API Authentication

## Overview

| API | Auth Method | Header | Documentation |
|-----|-------------|--------|---------------|
| REST API | OAuth2 (recommended) | `Authorization: Bearer <token>` | [Auth Docs](https://en.sipgate.io/rest-api/authentication) |
| REST API | Personal Access Token | `Authorization: Basic <base64>` | [PAT Settings](https://app.sipgate.com/personal-access-token) |
| Flow API | Shared Secret | `X-API-TOKEN: <secret>` | [Flow Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) |
| Push API | URL Registration | N/A (webhook URL configured in UI) | [Push API Docs](https://en.sipgate.io/push-api/api-references) |

---

## Common Mistakes

- **Using expired tokens**: Access tokens expire after ~5 minutes. Always implement refresh logic.
- **Requesting too many scopes**: Only request scopes you actually need. [Scope Reference](https://en.sipgate.io/rest-api/oauth2-scopes)
- **PAT in client-side code**: Personal Access Tokens must NEVER be used in frontend/client-side code.
- **Missing Content-Type**: Token endpoint requires `Content-Type: application/x-www-form-urlencoded`.
- **Not validating Flow shared secret**: Always check the `X-API-TOKEN` header in Flow API requests.

---

## OAuth2 (Recommended for REST API)

The standard authentication method for production applications. Uses the [Authorization Code Flow](https://en.sipgate.io/rest-api/authentication).

### Endpoints

| Purpose | URL |
|---------|-----|
| Authorization | `https://login.sipgate.com/auth/realms/third-party/protocol/openid-connect/auth` |
| Token Exchange | `https://api.sipgate.com/login/third-party/protocol/openid-connect/token` |
| Userinfo | `https://api.sipgate.com/v2/authorization/userinfo` |

### Token Exchange (cURL)

```bash
curl -X POST "https://api.sipgate.com/login/third-party/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "code=AUTHORIZATION_CODE" \
  -d "redirect_uri=YOUR_REDIRECT_URI" \
  -d "grant_type=authorization_code"
```

### Token Refresh (cURL)

```bash
curl -X POST "https://api.sipgate.com/login/third-party/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "refresh_token=YOUR_REFRESH_TOKEN" \
  -d "grant_type=refresh_token"
```

### Using the Access Token

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/account"
```

### OAuth2 Scopes

Full list of available scopes: [OAuth2 Scopes Reference](https://en.sipgate.io/rest-api/oauth2-scopes)

Common scopes:

| Scope | Description |
|-------|-------------|
| `account:read` | Read account details |
| `sessions:calls:write` | Initiate calls |
| `sessions:sms:write` | Send SMS |
| `sessions:fax:write` | Send faxes |
| `history:read` | Read call/SMS/fax history |
| `contacts:read` | Read contacts |
| `contacts:write` | Manage contacts |
| `devices:read` | Read device settings |
| `voicemails:read` | Read voicemails |
| `rtcm:write` | Real-time call management |

### Language-Specific Examples

- **Node.js**: [sipgateio-oauth-node](https://github.com/sipgate-io/sipgateio-oauth-node)
- **Java**: [sipgateio-oauth-java](https://github.com/sipgate-io/sipgateio-oauth-java)
- **Python**: [sipgateio-oauth-python](https://github.com/sipgate-io/sipgateio-oauth-python)

---

## Personal Access Token (PAT)

> **WARNING**: Personal Access Tokens grant full account access. Never commit them to repositories or use in client-side code. Use only for personal scripts, CLI tools, and prototyping. For production applications, always use OAuth2.

### Create a Token

1. Go to [Personal Access Token Settings](https://app.sipgate.com/personal-access-token)
2. Select required scopes and create the token
3. Copy **Token ID** and **Token** — shown only once!

### Usage (cURL)

```bash
# base64 encode "tokenId:token"
TOKEN=$(echo -n "token-XXXXXX:YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY" | base64)

curl -H "Authorization: Basic $TOKEN" \
  "https://api.sipgate.com/v2/account"
```

---

## Flow API Shared Secret

The Flow API uses a shared secret for request authentication. Your application receives an `X-API-TOKEN` header with each request from sipgate. [Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt)

### Validation

When receiving a Flow API request, always validate the shared secret:

```
# Pseudocode — implement in your language of choice
if request.headers["X-API-TOKEN"] != YOUR_CONFIGURED_SECRET:
    return 401 Unauthorized
```

The shared secret is configured when setting up your Flow API application in sipgate.

---

## Push API Authentication

The Push API does not use traditional API authentication. Instead, you configure a webhook URL in the sipgate web interface, and sipgate sends POST requests to that URL. [Push API Setup](https://en.sipgate.io/push-api/api-references)

### Security Recommendations

- Use HTTPS for your webhook URL
- Validate the source IP (sipgate's IP ranges)
- Use a secret path segment in your webhook URL (e.g., `/webhook/a8f3b2c1d4e5`)
- Respond quickly (sipgate has a timeout for webhook responses)

---

## Further Reading

- [Authentication Overview](https://en.sipgate.io/rest-api/authentication)
- [Building Third-Party Applications](https://en.sipgate.io/rest-api/building-a-third-party-application)
- [Managing Third-Party Clients](https://en.sipgate.io/rest-api/managing-third-party-clients)
- [OAuth2 Scopes Reference](https://en.sipgate.io/rest-api/oauth2-scopes)
