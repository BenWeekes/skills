# Agora Server-Side

Server-side utilities for Agora — primarily token generation for secure authentication.

## When Tokens Are Needed

- **Production**: Always. Tokens authenticate users before they join channels.
- **Testing/Development**: Optional. Disable token authentication in [Agora Console](https://console.agora.io) and pass `null` as token.
- **Never expose App Certificate on client**. Token generation must happen server-side.

## Token Types

- **RTC Token**: Grants access to join a specific RTC channel with a specific UID. Required for Video/Voice SDK.
- **RTM Token**: Grants access to RTM services for a specific user ID.
- **Token 007**: Current token format (replaces legacy Token 006). Supports privilege expiration per service.

## Reference Files

- **[tokens.md](tokens.md)** — Token generation for Node.js, Python, and Go. Express server example, security best practices.

## Official Documentation

- Authentication Guide: https://docs.agora.io/en/video-calling/token-authentication/authentication-workflow
