# Getting Started with sipgate APIs

## Common Mistakes

- **Wrong Redirect URI**: The redirect URI in your OAuth2 client config must exactly match the one in your code (including trailing slashes). [Docs](https://en.sipgate.io/rest-api/building-a-third-party-application)
- **Missing Scopes**: Request only the scopes you need, but make sure you don't forget any. [Scope Reference](https://en.sipgate.io/rest-api/oauth2-scopes)
- **Not Refreshing Tokens**: Access tokens expire after ~5 minutes. Always implement token refresh logic.
- **HTTP in Production**: Redirect URIs must use HTTPS in production. `http://localhost` is only allowed for development.
- **PAT in Production**: Personal Access Tokens are for prototyping only. Use OAuth2 for production apps. See PAT section below.

---

## 1. sipgate Account & API Access

### Prerequisites
- A sipgate account (basic or team) — [Create account](https://www.sipgate.de)
- For third-party apps: Register an OAuth2 client

### Register an OAuth2 Client

Follow the guide at [Building a Third-Party Application](https://en.sipgate.io/rest-api/building-a-third-party-application):

1. Go to [sipgate Console](https://console.sipgate.com) or your sipgate account settings
2. Register a new OAuth2 client
3. Note your **Client ID** and **Client Secret**
4. Configure your **Redirect URI** (e.g., `http://localhost:3000/callback` for development)
5. Select the required **OAuth2 scopes** — [Scope Reference](https://en.sipgate.io/rest-api/oauth2-scopes)

---

## 2. "Login with sipgate" (OAuth2 / OpenID Connect)

This is the recommended authentication method for all production applications. It follows the standard [OAuth2 Authorization Code Flow](https://en.sipgate.io/rest-api/authentication).

### OAuth2 Endpoints

| Endpoint | URL |
|----------|-----|
| Authorization | `https://login.sipgate.com/auth/realms/third-party/protocol/openid-connect/auth` |
| Token | `https://api.sipgate.com/login/third-party/protocol/openid-connect/token` |
| Userinfo | `https://api.sipgate.com/v2/authorization/userinfo` |

### Step 1: Redirect User to sipgate Login

Construct the authorization URL and redirect the user:

```
https://login.sipgate.com/auth/realms/third-party/protocol/openid-connect/auth
  ?client_id=YOUR_CLIENT_ID
  &redirect_uri=http://localhost:3000/callback
  &scope=account:read sessions:sms:write
  &response_type=code
  &state=RANDOM_STATE_VALUE
```

### Step 2: Receive Authorization Code

After the user logs in and consents, sipgate redirects to your callback URL:

```
http://localhost:3000/callback?code=AUTHORIZATION_CODE&state=RANDOM_STATE_VALUE
```

Always verify the `state` parameter matches what you sent.

### Step 3: Exchange Code for Tokens

```bash
curl -X POST "https://api.sipgate.com/login/third-party/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "code=AUTHORIZATION_CODE" \
  -d "redirect_uri=http://localhost:3000/callback" \
  -d "grant_type=authorization_code"
```

Response:
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "eyJhbGci...",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "account:read sessions:sms:write"
}
```

### Step 4: Use the Access Token

```bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
  "https://api.sipgate.com/v2/authorization/userinfo"
```

### Step 5: Refresh Expired Tokens

Access tokens expire after ~5 minutes. Use the refresh token:

```bash
curl -X POST "https://api.sipgate.com/login/third-party/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "refresh_token=YOUR_REFRESH_TOKEN" \
  -d "grant_type=refresh_token"
```

### Language-Specific Examples

Official OAuth2 example implementations on GitHub:
- **Node.js**: [sipgateio-oauth-node](https://github.com/sipgate-io/sipgateio-oauth-node)
- **Java**: [sipgateio-oauth-java](https://github.com/sipgate-io/sipgateio-oauth-java)
- **Python**: [sipgateio-oauth-python](https://github.com/sipgate-io/sipgateio-oauth-python)

---

## 3. Personal Access Token (Quick Alternative)

> **WARNING**: Personal Access Tokens grant full account access. Never commit them to repositories or use in client-side code. Use only for personal scripts, CLI tools, and prototyping. For production applications, always use OAuth2.

### Create a Personal Access Token

1. Go to [Personal Access Token Settings](https://app.sipgate.com/personal-access-token)
2. Create a new token and select the required scopes
3. Copy the **Token ID** and **Token** — the token is shown only once!

### Usage

PATs use HTTP Basic Authentication with `tokenId` as username and `token` as password:

```bash
# Encode credentials: base64(tokenId:token)
TOKEN=$(echo -n "token-XXXXXX:YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY" | base64)

curl -H "Authorization: Basic $TOKEN" \
  "https://api.sipgate.com/v2/account"
```

### Security Best Practices

- Store tokens in environment variables or `.env` files
- Add `.env` to `.gitignore`
- Never hard-code tokens in source code
- Rotate tokens regularly
- Use minimal required scopes

---

## 4. Choosing Your API

After authentication is set up, choose the right API for your use case:

| Goal | API | Getting Started |
|------|-----|-----------------|
| AI voice bot / real-time voice | Flow API | See `references/flow-api.md` |
| Send SMS, fax, initiate calls | REST API | See `references/rest-api.md` |
| React to incoming/outgoing calls | Push API | See `references/push-api.md` |
| Multiple APIs needed? | See decision matrix | See `references/choosing-an-api.md` |

For detailed use case descriptions, see `references/use-cases.md`.

---

## Further Reading

- [sipgate REST API Documentation](https://api.sipgate.com/v2/doc)
- [sipgate.io Developer Portal](https://en.sipgate.io)
- [Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt)
- [Push API Reference](https://en.sipgate.io/push-api/api-references)
- [OAuth2 Scopes](https://en.sipgate.io/rest-api/oauth2-scopes)
- [Building Third-Party Applications](https://en.sipgate.io/rest-api/building-a-third-party-application)
