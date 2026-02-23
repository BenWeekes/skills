# Agent Samples

Backend + frontend clients for Agora Conversational AI.

**Repo:** https://github.com/AgoraIO-Conversational-AI/agent-samples
**Coding Guide:** https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md

## Quick Start

> **[AGENT.md — Local Development Quick Start](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#local-development-quick-start)** — Prerequisites, backend setup, frontend setup
> **[README — Backend Sample](https://github.com/AgoraIO-Conversational-AI/agent-samples#backend-sample)** — Full setup overview

## Backend (simple-backend/)

- Python Flask server on port 8082
- Profile-based config system (`<PROFILE>_<VARIABLE>`)
- Agent lifecycle management, token generation

> **[AGENT.md — Backend Configuration](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#backend-configuration)** — Profile system, variable naming, active profiles

## Profile System

- Default profiles: `VOICE` (Rime TTS + OpenAI), `VIDEO` (ElevenLabs + GPT-4o + HeyGen)
- Profile names are case-insensitive
- Client sends `profile=VOICE` → backend loads all `VOICE_*` env vars

> **[AGENT.md — Profile System Mechanics](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#profile-system-mechanics)**

## MLLM / Gemini Live Configuration

- Required vars: `VOICE_ENABLE_MLLM`, `VOICE_MLLM_VENDOR`, `VOICE_MLLM_MODEL`, `VOICE_MLLM_LOCATION` (NOT REGION!)
- `MLLM_LOCATION` not `MLLM_REGION` — the backend expects LOCATION

> **[AGENT.md — Required MLLM Variables](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#required-mllm-variables-for-gemini-live)**
> **[AGENT.md — Configuration Translation Guide](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#configuration-translation-guide)**

## React Voice Client (react-voice-client/)

- Next.js voice AI client, port 8083
- Uses `@agora/conversational-ai` + `@agora/agent-ui-kit`

## React Video Client (react-video-client-avatar/)

- Next.js video+avatar client, port 8084
- HeyGen/Anam avatar integration

## Prerequisites

- **Node.js >= 20.9.0** — required by Next.js 16 (used by both React clients)
- **Python 3.x** — required by simple-backend

## Simple HTML Clients

- `simple-voice-client-no-backend/` — standalone, no backend needed
- `simple-voice-client-with-backend/` — uses simple-backend

## Debugging Agent Failures

- RTM error `-11033: user offline` → agent failed to create (400 from Agora API)
- Check backend logs for `Response status: 400`
- Common: missing `location` field in MLLM config

> **[AGENT.md — Debugging Agent Creation Failures](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#debugging-agent-creation-failures)**

## Production Deployment (EC2 + nginx)

> **[AGENT.md — Production Deployment](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md#production-deployment-ec2--nginx-on-port-443)** — nginx config, PM2 ecosystem, basePath, gotchas

## Companion Servers

- **server-custom-llm** → see [server-custom-llm.md](server-custom-llm.md)
- **server-mcp** → see [server-mcp.md](server-mcp.md)

## Port Reference

| Server | Port |
|--------|------|
| simple-backend | 8082 |
| react-voice-client | 8083 |
| react-video-client-avatar | 8084 |
