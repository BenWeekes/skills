# Agora Conversational AI Engine

REST API-driven voice AI agents. Create agents that join RTC channels and converse with users via speech. Front-end clients connect via RTC+RTM.

## Architecture

```
Your Server (REST API calls)
    ↓ POST /join with config
Agora ConvoAI Engine
    ↓ creates agent
Agent joins RTC channel ←→ Front-end client (RTC + RTM)
    ↓                           ↓
ASR → LLM → TTS             Receives audio + transcripts
```

1. Your server calls the REST API to create an agent with LLM/TTS/ASR config
2. The agent joins an Agora RTC channel and subscribes to the user's audio
3. ASR converts speech to text → LLM generates response → TTS converts to speech
4. The agent publishes audio back to the channel; transcripts arrive via RTC data channel or RTM

## Authentication

All REST API calls use **HTTP Basic Auth**:
- Credentials: Customer ID + Customer Secret from [Agora Console](https://console.agora.io) → Developer Toolkit → RESTful API
- Header: `Authorization: Basic <base64(customerID:customerSecret)>`
- HTTPS required (TLS 1.0/1.1/1.2)

## Base URL

```
https://api.agora.io/api/conversational-ai-agent/v2/projects/{appid}
```

## Agent Lifecycle

| Status | Code | Description |
|--------|------|-------------|
| IDLE | 0 | Ready, not active |
| STARTING | 1 | Initialization in progress |
| RUNNING | 2 | Active, processing audio |
| STOPPING | 3 | Shutdown in progress |
| STOPPED | 4 | Exited channel |
| RECOVERING | 5 | Error recovery |
| FAILED | 6 | Execution failure |

## REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/join` | Start agent — joins channel |
| POST | `/agents/{agentId}/leave` | Stop agent — leaves channel |
| POST | `/agents/{agentId}/update` | Update agent config (token, LLM) |
| GET | `/agents/{agentId}` | Query agent status |
| GET | `/agents` | List agents (with filters) |
| POST | `/agents/{agentId}/speak` | Broadcast TTS message |
| POST | `/agents/{agentId}/interrupt` | Interrupt agent speech |
| GET | `/agents/{agentId}/history` | Get conversation history |

## Topic Reference Files

Read the file matching what the user needs:

- **[rest-api.md](rest-api.md)** — All REST endpoint details: request/response bodies, parameters, examples
- **[agent-config.md](agent-config.md)** — The `properties` object in the join payload: LLM, TTS, ASR, VAD, turn detection, tools
- **[web-client.md](web-client.md)** — `@agora/conversational-ai` agent-toolkit SDK, React hooks, transcript handling

## Official Documentation

- Overview: https://docs.agora.io/en/conversational-ai/overview/product-overview
- REST Quickstart: https://docs.agora.io/en/conversational-ai/get-started/quickstart
- REST API (Join): https://docs.agora.io/en/conversational-ai/rest-api/agent/join
- Event Notifications: https://docs.agora.io/en/conversational-ai/develop/event-notifications
- Web Toolkit API Reference: https://docs.agora.io/en/conversational-ai/reference/web
- Authentication: https://docs.agora.io/en/conversational-ai/rest-api/restful-authentication
