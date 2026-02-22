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

## Reference Files

Each file maps to one repo in [AgoraIO-Conversational-AI](https://github.com/AgoraIO-Conversational-AI):

- **[agent-samples.md](agent-samples.md)** — Backend (simple-backend), React clients, profile system, MLLM/Gemini config, deployment
- **[agent-toolkit.md](agent-toolkit.md)** — `@agora/conversational-ai` SDK: ConversationalAIAPI, RTCHelper, RTMHelper, transcript handling
- **[agent-ui-kit.md](agent-ui-kit.md)** — `@agora/agent-ui-kit` React components: voice, chat, video, settings
- **[server-custom-llm.md](server-custom-llm.md)** — Custom LLM proxy: RAG, tool calling, conversation memory
- **[server-mcp.md](server-mcp.md)** — MCP memory server: persistent per-user memory via tool calling

## REST API Reference

Full request/response details for all endpoints:

- **[Start Agent (Join)](https://docs.agora.io/en/conversational-ai/rest-api/agent/join)** — POST /join: start agent with LLM/TTS/ASR config
- **[Stop Agent (Leave)](https://docs.agora.io/en/conversational-ai/rest-api/agent/leave)** — POST /leave: stop agent
- **[Update Agent](https://docs.agora.io/en/conversational-ai/rest-api/agent/update)** — POST /update: update token, LLM config
- **[Query Agent Status](https://docs.agora.io/en/conversational-ai/rest-api/agent/query)** — GET /agents/{id}: query status
- **[List Agents](https://docs.agora.io/en/conversational-ai/rest-api/agent/list)** — GET /agents: list with filters
- **[Broadcast Message (Speak)](https://docs.agora.io/en/conversational-ai/rest-api/agent/speak)** — POST /speak: broadcast TTS
- **[Interrupt Agent](https://docs.agora.io/en/conversational-ai/rest-api/agent/interrupt)** — POST /interrupt
- **[Conversation History](https://docs.agora.io/en/conversational-ai/rest-api/agent/history)** — GET /history

## Agent Configuration (join payload `properties` object)

- **[Custom LLM Guide](https://docs.agora.io/en/conversational-ai/develop/custom-llm)** — LLM vendor, model, url, api_key, system prompt, greeting, style; TTS vendor, model, voice settings; ASR vendor, language, model
- **[Gemini Live MLLM](https://docs.agora.io/en/conversational-ai/models/mllm/gemini)** — Multimodal: vendor, model, credentials, location
- **[Join Endpoint (full schema)](https://docs.agora.io/en/conversational-ai/rest-api/agent/join)** — Complete properties schema: channel, token, turn detection, VAD, tools, avatars, encryption, filler words
- **[Release Notes](https://docs.agora.io/en/conversational-ai/overview/release-notes)** — New parameters and features

## Gotchas & Quirks

Things the official docs don't emphasize that cause frequent mistakes:

- **`/update` overwrites `params` entirely** — sending `{ "llm": { "params": { "max_tokens": 2048 } } }` erases `model` and everything else in `params`. Always send the full object.
- **`/speak` priority enum** — `"INTERRUPT"` (immediate, default), `"APPEND"` (queued after current speech), `"IGNORE"` (skip if agent is busy). `interruptable: false` prevents users from cutting in.
- **20 PCU default limit** — max 20 concurrent agents per App ID. Exceeding returns error on `/join`. Contact Agora support to increase.
- **Event notifications require two flags** — `advanced_features.enable_rtm: true` AND `parameters.data_channel: "rtm"` in the join config. Without both, `onAgentStateChanged`/`onAgentMetrics`/`onAgentError` won't fire. Additionally: `parameters.enable_metrics: true` for metrics, `parameters.enable_error_message: true` for errors.
- **Custom LLM interruptable metadata** — the first SSE chunk can be `{"object": "chat.completion.custom_metadata", "metadata": {"interruptable": false}}` to prevent user speech from interrupting critical responses (e.g., compliance disclaimers). Subsequent chunks use standard `chat.completion.chunk` format.
- **Error response format** — non-200 responses return `{ "detail": "...", "reason": "..." }`.
- **MLLM `location` not `region`** — use `params.location: "us-central1"`, not `region`. The field name is `location` at every level (join payload and backend env vars).
