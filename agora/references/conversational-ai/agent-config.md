# Conversational AI Agent Configuration

The `properties` object in the [join request body](rest-api.md#start-agent-join) configures the agent's behavior. This file documents all configuration sections.

## Table of Contents
- [Channel Configuration](#channel-configuration)
- [LLM Configuration](#llm-configuration)
- [TTS Configuration](#tts-configuration)
- [ASR Configuration](#asr-configuration)
- [Turn Detection](#turn-detection)
- [Advanced Features](#advanced-features)
- [Parameters](#parameters)
- [RTC Encryption](#rtc-encryption)
- [Filler Words](#filler-words)
- [Other Settings](#other-settings)

## Channel Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `channel` | string | Yes | — | RTC channel name to join |
| `token` | string | Yes | — | RTC authentication token |
| `agent_rtc_uid` | string | Yes | — | Agent's UID in the channel (`"0"` for auto-assign) |
| `remote_rtc_uids` | string[] | Yes | — | User UIDs to subscribe to (currently supports 1 user) |
| `enable_string_uid` | boolean | No | `false` | Enable string UID format |
| `idle_timeout` | integer | No | `30` | Seconds before auto-exit when subscribed users leave |

## LLM Configuration

`properties.llm` — configures the Large Language Model.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `url` | string | Yes | — | LLM API endpoint (OpenAI-compatible) |
| `api_key` | string | No | — | API key for the LLM service |
| `system_messages` | array | No | — | System prompt messages: `[{ "role": "system", "content": "..." }]` |
| `params` | object | Yes | — | Model parameters (vendor-specific) |
| `params.model` | string | — | — | Model name (e.g., `"gpt-4o"`, `"claude-sonnet-4-5-20250929"`) |
| `params.max_tokens` | integer | — | — | Max tokens in response |
| `max_history` | integer | No | `32` | Conversation turns cached (range: 1-1024) |
| `greeting_message` | string | No | — | Initial greeting spoken when user joins |
| `failure_message` | string | No | — | Fallback message when LLM fails |
| `input_modalities` | string[] | No | `["text"]` | Input types: `"text"` or `["text", "image"]` |
| `output_modalities` | string[] | No | `["text"]` | Output format |
| `vendor` | string | No | — | Provider type: `"custom"`, `"azure"` |
| `style` | string | No | `"openai"` | API format: `"openai"` or `"anthropic"` |
| `greeting_configs` | object | No | — | Greeting broadcast mode: `{ "mode": "single_every" }` or `"single_first"` |
| `template_variables` | object | No | — | Dynamic text substitution in system messages |
| `mcp_servers` | array | No | — | Model Context Protocol server connections |

### Example: OpenAI

```json
{
  "url": "https://api.openai.com/v1/chat/completions",
  "api_key": "sk-...",
  "system_messages": [
    { "role": "system", "content": "You are a helpful voice assistant. Keep responses concise." }
  ],
  "params": { "model": "gpt-4o", "max_tokens": 1024 },
  "max_history": 32,
  "greeting_message": "Hello! How can I help you today?"
}
```

### Example: Custom / Self-Hosted LLM

```json
{
  "url": "https://your-server.com/v1/chat/completions",
  "api_key": "your-key",
  "vendor": "custom",
  "system_messages": [
    { "role": "system", "content": "You are a customer service agent." }
  ],
  "params": { "model": "your-model", "max_tokens": 512 }
}
```

## TTS Configuration

`properties.tts` — configures Text-to-Speech.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vendor` | string | Yes | Provider: `"microsoft"`, `"elevenlabs"`, `"openai"`, `"google"`, `"amazon"` |
| `params` | object | Yes | Vendor-specific parameters (see below) |
| `skip_patterns` | integer[] | No | Skip bracketed content types (1-5) |

### Microsoft TTS Params

```json
{
  "vendor": "microsoft",
  "params": {
    "key": "your-azure-speech-key",
    "region": "eastus",
    "voice_name": "en-US-AndrewMultilingualNeural",
    "speed": 1.0,
    "volume": 100,
    "sample_rate": 24000
  }
}
```

### ElevenLabs TTS Params

```json
{
  "vendor": "elevenlabs",
  "params": {
    "key": "your-elevenlabs-key",
    "voice_id": "voice-id",
    "model_id": "eleven_turbo_v2_5"
  }
}
```

### OpenAI TTS Params

```json
{
  "vendor": "openai",
  "params": {
    "key": "sk-...",
    "model": "tts-1",
    "voice": "alloy"
  }
}
```

## ASR Configuration

`properties.asr` — configures Automatic Speech Recognition.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `language` | string | No | `"en-US"` | BCP-47 language code |
| `vendor` | string | No | `"ares"` | Provider: `"ares"`, `"microsoft"`, `"deepgram"`, `"openai"`, `"google"`, `"amazon"` |
| `params` | object | No | — | Vendor-specific configuration |

## Turn Detection

`properties.turn_detection` — configures when the user starts/stops speaking and interruption behavior.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | string | `"default"` | Detection mechanism |
| `config.speech_threshold` | number | `0.5` | VAD sensitivity (0.0-1.0, higher = less sensitive) |
| `config.start_of_speech.mode` | string | — | Start-of-speech detection: `"vad"`, `"keywords"` (Beta), `"disabled"` |
| `config.end_of_speech.mode` | string | — | End-of-speech detection: `"vad"`, `"semantic"` |

### Example: Semantic End-of-Speech

```json
{
  "mode": "default",
  "config": {
    "speech_threshold": 0.5,
    "start_of_speech": { "mode": "vad" },
    "end_of_speech": { "mode": "semantic" }
  }
}
```

## Advanced Features

`properties.advanced_features` — enable optional capabilities.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enable_aivad` | boolean | `false` | **Deprecated** — use semantic turn detection instead |
| `enable_mllm` | boolean | `false` | Multimodal LLM mode (disables ASR/LLM/TTS pipeline) |
| `enable_rtm` | boolean | `false` | Enable RTM signaling integration |
| `enable_sal` | boolean | `false` | Selective attention locking (voiceprint) |
| `enable_tools` | boolean | `false` | Enable LLM tool/function calling |

When `enable_mllm` is true, configure `properties.mllm` instead of `llm`/`tts`/`asr`. MLLM handles end-to-end voice processing directly (ASR, LLM, and TTS are all bypassed).

## MLLM Configuration (Gemini Live / Multimodal)

`properties.mllm` — configures the multimodal LLM when `enable_mllm` is true.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vendor` | string | Yes | `"vertexai"` for Google Gemini Live |
| `style` | string | Yes | `"openai"` (message format) |
| `greeting_message` | string | No | Initial agent message on connect |
| `input_modalities` | array | No | `["audio"]` (default) or `["audio", "text"]` |
| `output_modalities` | array | No | `["audio"]` (default) or `["text", "audio"]` |
| `params.model` | string | Yes | Model ID, e.g. `"gemini-live-2.5-flash-preview-native-audio-09-2025"` |
| `params.adc_credentials_string` | string | Yes | Base64-encoded GCP service account JSON |
| `params.project_id` | string | Yes | Google Cloud project ID |
| `params.location` | string | Yes | GCP region, e.g. `"us-central1"` (**not** `region`) |
| `params.voice` | string | No | Voice name: `Aoede`, `Puck`, `Charon`, `Kore`, `Fenrir`, `Leda`, `Orus`, `Zephyr` |
| `params.instructions` | string | No | System prompt |
| `params.transcribe_agent` | boolean | No | Enable agent speech-to-text transcription |
| `params.transcribe_user` | boolean | No | Enable user speech-to-text transcription |
| `params.messages` | array | No | Conversation history for context |

**Example MLLM join payload:**

```json
{
  "properties": {
    "advanced_features": { "enable_mllm": true },
    "mllm": {
      "vendor": "vertexai",
      "style": "openai",
      "greeting_message": "Hey there!",
      "input_modalities": ["audio"],
      "output_modalities": ["audio"],
      "params": {
        "model": "gemini-live-2.5-flash-preview-native-audio-09-2025",
        "adc_credentials_string": "<base64-encoded-service-account-json>",
        "project_id": "my-gcp-project",
        "location": "us-central1",
        "voice": "Charon",
        "instructions": "You are a friendly assistant. Keep responses conversational.",
        "transcribe_agent": true,
        "transcribe_user": true
      }
    },
    "turn_detection": { "type": "server_vad" }
  }
}
```

**Debugging MLLM failures:** If the agent returns 400, check: (1) `location` is not null (use `"us-central1"`, not `region`), (2) `adc_credentials_string` is valid base64 GCP JSON, (3) `enable_mllm: true` is set in `advanced_features`.

See also: https://docs.agora.io/en/conversational-ai/models/mllm/gemini

## Parameters

`properties.parameters` — additional behavior settings.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `silence_config.timeout_ms` | integer | `0` | Max silence duration before action (ms) |
| `silence_config.action` | string | `"speak"` | `"speak"` or `"think"` on silence timeout |
| `silence_config.content` | string | — | Prompt sent to LLM on silence |
| `farewell_config.graceful_enabled` | boolean | `false` | Wait for IDLE state before leaving |
| `farewell_config.graceful_timeout_seconds` | integer | `30` | Graceful leave timeout (0-120) |
| `data_channel` | string | `"datastream"` | Transcript delivery: `"rtm"` or `"datastream"` |
| `enable_metrics` | boolean | `false` | Receive performance metrics |
| `enable_error_message` | boolean | `false` | Receive error events |

## RTC Encryption

`properties.rtc` — media encryption settings.

| Field | Type | Description |
|-------|------|-------------|
| `encryption_key` | string | Media encryption key |
| `encryption_salt` | string | Base64-encoded 32-byte salt |
| `encryption_mode` | integer | Mode 1-8 (AES variants, SM4) |

## Filler Words

`properties.filler_words` — insert phrases while the agent is thinking.

```json
{
  "enable": true,
  "trigger": {
    "mode": "fixed_time",
    "response_wait_ms": 2000
  },
  "content": {
    "mode": "static",
    "phrases": ["Let me think about that...", "One moment please..."],
    "selection_rule": "random"
  }
}
```

## Other Settings

| Field | Type | Description |
|-------|------|-------------|
| `properties.geofence` | object | Geographic region restrictions: `GLOBAL`, `NORTH_AMERICA`, `EUROPE`, `ASIA`, `INDIA`, `JAPAN` |
| `properties.labels` | object | Custom metadata key-value pairs for business logic |
| `properties.avatar` | object | AI avatar config (see Avatar section below) |
| `properties.sal` | object | Speaker attention locking: `{ "sal_mode": "locking", "sample_urls": {...} }` |

## Function Calling (Tools)

When `enable_tools` is true, add tool definitions in the LLM config. The schema follows the OpenAI function calling format:

```json
{
  "llm": {
    "url": "https://api.openai.com/v1/chat/completions",
    "api_key": "sk-...",
    "params": {
      "model": "gpt-4o",
      "tools": [
        {
          "type": "function",
          "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
              "type": "object",
              "properties": {
                "location": { "type": "string", "description": "City name" }
              },
              "required": ["location"]
            }
          }
        }
      ]
    }
  },
  "advanced_features": { "enable_tools": true }
}
```

The agent handles the tool-call loop automatically — it calls the LLM, executes tool calls, and feeds results back.

## Avatar Configuration

`properties.avatar` — configure an AI avatar that generates video:

```json
{
  "avatar": {
    "enable": true,
    "vendor": "heygen",
    "params": {
      "avatar_id": "Graham_Chair_Sitting_public",
      "api_key": "<heygen-api-key>"
    }
  }
}
```

When enabled, the agent publishes a video track (the avatar) alongside its audio track. The client subscribes to both audio and video from the agent's RTC UID. Available vendors: `heygen`, `anam`.

**HeyGen params**: `avatar_id` (required), `api_key` (required). See HeyGen's API docs for available avatar IDs.

**Anam params**: `avatar_id` (required), `api_key` (required), plus optional persona and streaming config.

## Deprecated Fields (v2.4)

The following top-level `turn_detection` fields are deprecated. Use the nested `turn_detection.config` object instead:

| Deprecated Field | Replacement |
|-----------------|-------------|
| `turn_detection.interrupt_mode` | `turn_detection.config.start_of_speech.mode` |
| `turn_detection.interrupt_keywords` | `turn_detection.config.start_of_speech.keywords` |
| `turn_detection.interrupt_duration_ms` | `turn_detection.config.start_of_speech.duration_ms` |
| `turn_detection.prefix_padding_ms` | `turn_detection.config.prefix_padding_ms` |
| `turn_detection.silence_duration_ms` | `turn_detection.config.end_of_speech.silence_duration_ms` |
| `turn_detection.threshold` | `turn_detection.config.speech_threshold` |
| `advanced_features.enable_aivad` | Use `turn_detection.config.end_of_speech.mode: "semantic"` |

## Custom LLM Interruptable Metadata

When using a custom LLM endpoint, the first SSE chunk can include metadata to control whether the current response is interruptable:

```json
{"object": "chat.completion.custom_metadata", "metadata": {"interruptable": true}}
```

Set `interruptable: false` to prevent user speech from interrupting critical responses (e.g., compliance disclaimers). Subsequent chunks use the standard `chat.completion.chunk` format.

## Official Documentation

- Join endpoint (full schema): https://docs.agora.io/en/conversational-ai/rest-api/join
- Custom LLM guide: https://docs.agora.io/en/conversational-ai/develop/custom-llm
- Gemini Live MLLM: https://docs.agora.io/en/conversational-ai/models/mllm/gemini
- Google Vertex AI LLM: https://docs.agora.io/en/conversational-ai/models/llm/google-vertex-ai
- Release notes (new params): https://docs.agora.io/en/conversational-ai/overview/release-notes
