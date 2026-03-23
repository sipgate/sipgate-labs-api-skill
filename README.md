# sipgate API Skill for Claude Code

A Claude Code plugin that helps developers build integrations with sipgate's telephony APIs.

> **Unofficial** — This plugin is provided by sipgate but is not officially supported. Use at your own discretion.

## What it does

When installed, Claude Code gains deep knowledge of sipgate's four APIs and can help you:

- **Choose the right API** for your use case (Flow, REST, Push, or SIP Stream Bridge)
- **Set up authentication** (OAuth2, Personal Access Tokens, shared secrets)
- **Build integrations** with correct endpoint usage, phone number formatting, and error handling
- **Combine APIs** for complex scenarios (e.g., Push webhooks + REST/RTCM call control)
- **Migrate from Twilio** using the SIP Stream Bridge

## APIs covered

| API | Purpose | Docs |
|-----|---------|------|
| **Flow API** | AI voice bots, receptionists | [Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt) |
| **REST API + RTCM** | SMS, fax, calls, contacts, active call control | [Swagger](https://api.sipgate.com/v2/doc) |
| **Push API** | Telephony webhooks, IVR, call routing | [Docs](https://en.sipgate.io/push-api/api-references) |
| **SIP Stream Bridge** | Raw audio streaming, Twilio migration | [Docs](https://sipgate.github.io/sipgate-sip-stream-bridge/) |

## Installation

In Claude Code, run:

```
/plugin marketplace add sipgate/sipgate-labs-api-skill
/plugin install sipgate-api@sipgate-labs
```

## Manual installation

```bash
git clone https://github.com/sipgate/sipgate-labs-api-skill.git
cp -r sipgate-labs-api-skill/skills/sipgate-developer-apis ~/.claude/skills/
```

## Usage

Once installed, Claude Code automatically activates the skill when you work on sipgate-related tasks. You can also invoke it explicitly:

- Ask Claude to help you build a sipgate integration
- Ask which sipgate API to use for your use case
- Ask for code examples with any sipgate API endpoint

## Links

- [sipgate Developer Portal](https://en.sipgate.io)
- [sipgate REST API](https://api.sipgate.com/v2/doc)
- [Flow API Reference](https://sipgate.github.io/sipgate-ai-flow-api/LLM_REFERENCE.txt)
- [Push API Reference](https://en.sipgate.io/push-api/api-references)
- [SIP Stream Bridge](https://sipgate.github.io/sipgate-sip-stream-bridge/)
- [Flow SDK (npm)](https://www.npmjs.com/package/@sipgate/ai-flow-sdk)
