# Conversational AI REST API

All endpoints use the base URL: `https://api.agora.io/api/conversational-ai-agent/v2/projects/{appid}`

All requests require:
- `Authorization: Basic <base64(customerID:customerSecret)>`
- `Content-Type: application/json`

## Table of Contents
- [Start Agent (Join)](#start-agent-join)
- [Stop Agent (Leave)](#stop-agent-leave)
- [Update Agent](#update-agent)
- [Query Agent Status](#query-agent-status)
- [List Agents](#list-agents)
- [Broadcast Message (Speak)](#broadcast-message-speak)
- [Interrupt Agent](#interrupt-agent)
- [Get Conversation History](#get-conversation-history)
- [Agent Status Enum](#agent-status-enum)
- [Error Responses](#error-responses)

## Start Agent (Join)

**POST** `/join`

Creates an agent instance and joins the specified RTC channel.

### Request Body

```json
{
  "name": "unique-agent-name",
  "properties": {
    "channel": "my-channel",
    "token": "007eJx...",
    "agent_rtc_uid": "0",
    "remote_rtc_uids": ["12345"],
    "enable_string_uid": false,
    "idle_timeout": 30,
    "llm": {
      "url": "https://api.openai.com/v1/chat/completions",
      "api_key": "sk-...",
      "system_messages": [
        { "role": "system", "content": "You are a helpful assistant." }
      ],
      "params": {
        "model": "gpt-4o",
        "max_tokens": 1024
      },
      "max_history": 32,
      "greeting_message": "Hello! How can I help you?",
      "failure_message": "Sorry, I'm having trouble right now."
    },
    "tts": {
      "vendor": "microsoft",
      "params": {
        "key": "your-azure-key",
        "region": "eastus",
        "voice_name": "en-US-AndrewMultilingualNeural",
        "speed": 1.0,
        "sample_rate": 24000
      }
    },
    "asr": {
      "language": "en-US"
    }
  }
}
```

### Key Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique agent identifier (cannot be reused) |
| `properties` | object | Yes | Agent configuration — see [agent-config.md](agent-config.md) for full schema |
| `properties.channel` | string | Yes | RTC channel name to join |
| `properties.token` | string | Yes | RTC authentication token |
| `properties.agent_rtc_uid` | string | Yes | Agent's UID (`"0"` for auto-assign) |
| `properties.remote_rtc_uids` | string[] | Yes | User UIDs to subscribe to (currently supports 1 user) |
| `properties.llm` | object | Yes | LLM configuration |
| `properties.tts` | object | Yes | Text-to-speech configuration |

### Response (200)

```json
{
  "agent_id": "1NT29X10YHxxxxxWJOXLYHNYB",
  "create_ts": 1737111452,
  "status": "RUNNING"
}
```

Store `agent_id` for all subsequent API calls.

---

## Stop Agent (Leave)

**POST** `/agents/{agentId}/leave`

Stops the agent and leaves the RTC channel. Responds asynchronously (v2.3+).

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Request Body

None.

### Response (200)

Empty body `{}`.

---

## Update Agent

**POST** `/agents/{agentId}/update`

Updates agent configuration while running. Supports updating token, LLM system messages, and LLM params.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Request Body

```json
{
  "properties": {
    "token": "new-token-value",
    "llm": {
      "system_messages": [
        { "role": "system", "content": "Updated system prompt." }
      ],
      "params": {
        "model": "gpt-4o",
        "max_tokens": 2048
      }
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `properties.token` | string | New RTC token (for token renewal) |
| `properties.llm.system_messages` | array | Updated system prompt messages |
| `properties.llm.params` | object | Updated LLM params (**overwrites** existing config entirely) |
| `properties.mllm.params` | object | Updated MLLM params (if using multimodal mode) |

### Response (200)

```json
{
  "agent_id": "1NT29X10YHxxxxxWJOXLYHNYB",
  "create_ts": 1737123456,
  "status": "RUNNING"
}
```

---

## Query Agent Status

**GET** `/agents/{agentId}`

Returns the current status and metadata of a specific agent.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Response (200)

```json
{
  "agent_id": "1NT29X10YHxxxxxWJOXLYHNYB",
  "start_ts": 1735035893,
  "stop_ts": 1735035900,
  "status": "RUNNING",
  "message": ""
}
```

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | string | Agent identifier |
| `start_ts` | integer | Creation timestamp |
| `stop_ts` | integer | Stop timestamp (if stopped) |
| `status` | string | Current status (see [Agent Status Enum](#agent-status-enum)) |
| `message` | string | Status message (e.g., exit reason) |

---

## List Agents

**GET** `/agents`

Returns agents matching the specified criteria, with pagination.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `channel` | string | — | Filter by channel name |
| `from_time` | number | 2 hours ago | Start timestamp (seconds) |
| `to_time` | number | now | End timestamp (seconds) |
| `state` | string | `2` (RUNNING) | Filter by state code (0-6) |
| `limit` | integer | 20 | Max results per page |
| `cursor` | string | — | Pagination cursor from previous response |

### Response (200)

```json
{
  "data": {
    "count": 1,
    "list": [
      {
        "agent_id": "1234567890ABCDE1CVGZNU80BEIN56XF",
        "start_ts": 1735035893,
        "status": "RUNNING"
      }
    ]
  },
  "meta": {
    "cursor": "",
    "total": 1
  },
  "status": "ok"
}
```

---

## Broadcast Message (Speak)

**POST** `/agents/{agentId}/speak`

Makes the agent speak a custom message via TTS. Not supported with MLLM mode.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Request Body

```json
{
  "text": "Sorry, the conversation content is not compliant.",
  "priority": "INTERRUPT",
  "interruptable": false
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `text` | string | — | Message text (max 512 bytes) |
| `priority` | string | `INTERRUPT` | `INTERRUPT` (immediate), `APPEND` (after current), `IGNORE` (skip if busy) |
| `interruptable` | boolean | `true` | Whether users can interrupt the broadcast by speaking |

### Response (200)

```json
{
  "agent_id": "1NT29XxxxxxxxxELWEHC8OS",
  "channel": "test_channel",
  "start_ts": 1744877089
}
```

---

## Interrupt Agent

**POST** `/agents/{agentId}/interrupt`

Interrupts the agent immediately — stops both speaking and thinking.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Request Body

Empty `{}`.

### Response (200)

Success with agent info.

---

## Get Conversation History

**GET** `/agents/{agentId}/history`

Retrieves the agent's conversation history (short-term memory). Only works while the agent is running.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Agent ID from the join response |

### Response (200)

```json
{
  "agent_id": "xxxx",
  "start_ts": 123,
  "status": "RUNNING",
  "contents": [
    { "role": "user", "content": "hello." },
    { "role": "assistant", "content": "hi, how can I help you?" }
  ]
}
```

Max cached entries controlled by `llm.max_history` (default: 32).

---

## Rate Limits

- **PCU (Peak Concurrent Users)**: Default limit is **20 concurrent agents** per App ID.
- To increase the PCU limit, contact Agora support or your account manager.
- Exceeding the limit returns an error on `/join` calls.

---

## Agent Status Enum

| Status | Code | Description |
|--------|------|-------------|
| IDLE | 0 | Ready, not active |
| STARTING | 1 | Initialization in progress |
| RUNNING | 2 | Active, processing audio |
| STOPPING | 3 | Shutdown in progress |
| STOPPED | 4 | Exited channel |
| RECOVERING | 5 | Error recovery |
| FAILED | 6 | Execution failure |

## Error Responses

Non-200 responses return:

```json
{
  "detail": "error description",
  "reason": "failure reason"
}
```

## Official Documentation

- Start Agent: https://docs.agora.io/en/conversational-ai/rest-api/agent/join
- REST Quickstart: https://docs.agora.io/en/conversational-ai/get-started/quickstart
- Authentication: https://docs.agora.io/en/conversational-ai/rest-api/restful-authentication
