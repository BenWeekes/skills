# Agora Server-Side: Token Generation

## Overview

Tokens authenticate users before joining channels. Generated server-side from App ID + App Certificate + channel + UID + expiration.

## Quick Reference

- `onTokenPrivilegeWillExpire` fires **30 seconds** before expiry — renew proactively
- UID in token must match UID used to join
- Max token validity: 24 hours
- Use `RtcRole.SUBSCRIBER` for audience-only users (prevents stream bombing)

## Token Generation Guides

- **[Deploy a Token Server](https://docs.agora.io/en/video-calling/token-authentication/deploy-token-server)** — Express/Flask/Go server examples
- **[Use Tokens](https://docs.agora.io/en/video-calling/token-authentication/authentication-workflow)** — When and how tokens are used

## Token Libraries

| Language | Package |
|----------|---------|
| Node.js | `agora-token` |
| Python | `agora-token-builder` |
| Go | `github.com/AgoraIO/Tools/DynamicKey/AgoraDynamicKey/go/src/rtctokenbuilder2` |
| Java, PHP, C++ | Available in [AgoraIO/Tools](https://github.com/AgoraIO/Tools/tree/master/DynamicKey/AgoraDynamicKey) |

## Token Types

- **RTC Token** — channel access for Video/Voice SDK
- **RTM Token** — access for Real-Time Messaging
- **Combined Token (Token 007)** — bundles RTC + RTM privileges in one token

> **[Token 007 Guide](https://docs.agora.io/en/video-calling/token-authentication/deploy-token-server)** — AccessToken2 with multi-service privileges
