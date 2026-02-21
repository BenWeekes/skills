# Test: MLLM/Gemini Live Setup

## Prompt

How do I set up agent-samples with MLLM/Gemini Live for the voice client?

## Skill Files Loaded

1. `~/.claude/skills/agora/SKILL.md` -- main skill entry point, product index
2. `~/.claude/skills/agora/references/conversational-ai/README.md` -- ConvoAI architecture, REST API overview, topic index
3. `~/.claude/skills/agora/references/conversational-ai/recipes.md` -- 5 recipe repos, agent-samples quick start, AGENT.md link
4. `~/.claude/skills/agora/references/conversational-ai/agent-config.md` -- full `properties` schema including MLLM section, advanced features, turn detection, TTS/ASR/LLM config
5. `~/.claude/skills/agora/references/conversational-ai/rest-api.md` -- REST endpoint details, join payload structure, error handling
6. `~/.claude/skills/agora/references/conversational-ai/web-client.md` -- agent-toolkit SDK, React hooks, transcript handling
7. **External URL:** `https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md` -- profile-based config system, MLLM/Gemini Live environment variables, debugging guide, production deployment

## Plan

### Overview

The goal is to configure the `agent-samples` repository to run in MLLM (Multimodal LLM) mode using Gemini Live via Google Vertex AI, connected to the react-voice-client. In MLLM mode, Gemini Live handles end-to-end voice processing natively -- ASR, LLM reasoning, and TTS are all bypassed in favor of the multimodal model's native audio capabilities.

### Prerequisites

1. **Agora Account**: An active Agora Console account at https://console.agora.io with:
   - An App ID (project identifier)
   - Customer ID and Customer Secret (from Developer Toolkit > RESTful API) for HTTP Basic Auth
   - Optionally, an App Certificate if token authentication is enabled

2. **Google Cloud Platform (GCP)**:
   - A GCP project with the Vertex AI API enabled
   - A service account with Vertex AI permissions
   - The service account JSON key file downloaded locally

3. **Software**:
   - Python 3.x with pip
   - Node.js 18+ with npm
   - Git

### Step 1: Clone the Repository

```bash
git clone https://github.com/AgoraIO-Conversational-AI/agent-samples.git
cd agent-samples
```

### Step 2: Prepare GCP Credentials

The MLLM configuration requires the GCP service account JSON key to be base64-encoded. Generate it:

```bash
# Base64-encode your service account key (no newlines)
cat /path/to/your-service-account-key.json | base64 | tr -d '\n'
```

Save this base64 string -- you will use it as the `VOICE_MLLM_ADC_CREDENTIALS_STRING` value.

### Step 3: Configure the Backend `.env` File

The agent-samples backend uses a **profile-based configuration system**. All environment variables are prefixed with the profile name. For the voice client, the profile is `VOICE`.

```bash
cd simple-backend
cp .env.example .env
```

Edit the `.env` file with the following variables. The critical MLLM-specific variables are marked:

```env
# === Agora Credentials (shared across profiles) ===
AGORA_APP_ID=your_agora_app_id
AGORA_APP_CERTIFICATE=your_agora_app_certificate
AGORA_CUSTOMER_ID=your_agora_customer_id
AGORA_CUSTOMER_SECRET=your_agora_customer_secret

# === VOICE Profile: Gemini Live MLLM Mode ===

# Enable MLLM mode (critical -- disables the ASR/LLM/TTS pipeline)
VOICE_ENABLE_MLLM=true

# MLLM vendor and model
VOICE_MLLM_VENDOR=vertexai
VOICE_MLLM_MODEL=gemini-live-2.5-flash-preview-native-audio-09-2025

# GCP credentials (base64-encoded service account JSON from Step 2)
VOICE_MLLM_ADC_CREDENTIALS_STRING=<paste-base64-encoded-json-here>

# GCP project and region
# IMPORTANT: Use "location", NOT "region". The field name must be LOCATION.
VOICE_MLLM_PROJECT_ID=your-gcp-project-id
VOICE_MLLM_LOCATION=us-central1

# Gemini Live voice selection
# Options: Aoede, Puck, Charon, Kore, Fenrir, Leda, Orus, Zephyr
VOICE_MLLM_VOICE=Charon

# Enable transcription (so transcripts appear in the voice client)
VOICE_MLLM_TRANSCRIBE_AGENT=true
VOICE_MLLM_TRANSCRIBE_USER=true

# ASR settings (still needed for certain fallback/VAD paths)
VOICE_ASR_VENDOR=ares
VOICE_ASR_LANGUAGE=en-US

# VAD / Turn detection
VOICE_VAD_SILENCE_DURATION_MS=300
VOICE_ENABLE_AIVAD=true
```

**Key pitfalls to avoid:**
- Do NOT use `VOICE_MLLM_REGION` -- the backend expects `VOICE_MLLM_LOCATION`. Using `region` instead of `location` will cause `null` to be sent in the API request, resulting in a 400 error.
- The `VOICE_MLLM_ADC_CREDENTIALS_STRING` must be the raw base64-encoded content of the entire service account JSON file, not a file path.
- `VOICE_ENABLE_MLLM=true` is mandatory. Without it, the backend will use the standard ASR/LLM/TTS pipeline and ignore all `mllm` settings.

### Step 4: Understand What the Backend Sends

When the voice client calls `http://localhost:8082/start-agent?channel=test&profile=VOICE`, the backend constructs a join payload like this and sends it to the Agora ConvoAI REST API:

```json
{
  "name": "unique-agent-name",
  "properties": {
    "channel": "test",
    "token": "<generated-token>",
    "agent_rtc_uid": "0",
    "remote_rtc_uids": ["<user-uid>"],
    "advanced_features": {
      "enable_mllm": true
    },
    "mllm": {
      "vendor": "vertexai",
      "style": "openai",
      "greeting_message": "Hey there!",
      "input_modalities": ["audio"],
      "output_modalities": ["audio"],
      "params": {
        "model": "gemini-live-2.5-flash-preview-native-audio-09-2025",
        "adc_credentials_string": "<base64-encoded-service-account-json>",
        "project_id": "your-gcp-project-id",
        "location": "us-central1",
        "voice": "Charon",
        "instructions": "You are a friendly assistant.",
        "transcribe_agent": true,
        "transcribe_user": true
      }
    },
    "turn_detection": {
      "type": "server_vad"
    }
  }
}
```

Note that `llm`, `tts`, and `asr` configuration blocks are NOT used in MLLM mode. Gemini Live handles the full audio-in, audio-out pipeline natively.

### Step 5: Start the Backend

```bash
cd agent-samples/simple-backend
pip3 install -r requirements-local.txt
python3 local_server.py
```

The backend starts on port 8082. Verify it is running:

```bash
curl http://localhost:8082/health
```

### Step 6: Start the React Voice Client

In a separate terminal:

```bash
cd agent-samples/react-voice-client
npm install --legacy-peer-deps
npm run dev
```

The voice client starts on port 8083. Open `http://localhost:8083` in your browser.

### Step 7: Test the Connection

1. Open the voice client at `http://localhost:8083`
2. The client will prompt you to allow microphone access -- grant it
3. Click the connect/start button
4. The client sends a request to the backend with `profile=VOICE`
5. The backend creates an agent via the Agora ConvoAI REST API with MLLM config
6. The Gemini Live agent joins the RTC channel and begins listening
7. Speak into your microphone -- you should hear the agent respond with Gemini Live's native audio voice

### Step 8: Debugging Common Failures

**Agent returns 400 error (visible in backend logs):**
- Check that `VOICE_MLLM_LOCATION` is set to a valid GCP region (e.g., `us-central1`), not null
- Verify `VOICE_ENABLE_MLLM=true` is in the `.env` file
- Confirm the base64-encoded GCP service account JSON is valid: `echo "<base64-string>" | base64 -d` should output valid JSON
- Make sure the GCP project has Vertex AI API enabled

**RTM error `-11033: user offline`:**
- This typically means the agent failed to start (often a 400 from the ConvoAI API)
- Check backend logs for the actual HTTP response from Agora
- The root cause is usually one of the credential/config issues above

**No transcripts appearing in the voice client:**
- Verify `VOICE_MLLM_TRANSCRIBE_AGENT=true` and `VOICE_MLLM_TRANSCRIBE_USER=true`
- If using RTM for data delivery, ensure `advanced_features.enable_rtm: true` and `parameters.data_channel: "rtm"` are set

**Agent speaks but voice sounds wrong:**
- Try a different voice: `Aoede`, `Puck`, `Charon`, `Kore`, `Fenrir`, `Leda`, `Orus`, `Zephyr`
- Update `VOICE_MLLM_VOICE` in the `.env` file

### Architecture Summary

```
Browser (react-voice-client, port 8083)
    |
    v
Backend (simple-backend, port 8082)
    |  POST /start-agent?profile=VOICE
    v
Agora ConvoAI REST API (POST /join with MLLM config)
    |
    v
Gemini Live Agent (joins RTC channel)
    |  Native audio-in / audio-out
    v
User hears agent via RTC audio
Transcripts arrive via RTC data channel or RTM
```

In MLLM mode, the traditional ASR -> LLM -> TTS pipeline is entirely replaced by Gemini Live's native multimodal audio processing. The model receives raw audio input and produces audio output directly, resulting in lower latency and more natural conversation.

## Assessment

**Rating: 4/5**

**What the skill did well:**
- The `agent-config.md` file provided a complete and accurate MLLM configuration schema with all required fields, types, a full example JSON payload, and debugging tips for common 400 errors
- The `recipes.md` file clearly pointed to the agent-samples repo with a quick start guide and, critically, linked to the external AGENT.md
- The external AGENT.md (fetched from GitHub) provided the profile-based environment variable naming convention (`VOICE_MLLM_*` prefix pattern), which is essential for configuring the backend correctly and would be very difficult to guess otherwise
- The `agent-config.md` explicitly warned about the `location` vs `region` field name pitfall, which is a known source of 400 errors
- The available Gemini Live voice names were enumerated (Aoede, Puck, Charon, etc.)
- The `web-client.md` provided transcript handling details that explain how the voice client receives and renders agent transcripts

**What was missing or could be improved:**
- The skill does not include the actual `.env.example` file contents from agent-samples, so a developer has to either fetch it from the repo or guess the full set of base variables (the AGENT.md external fetch partially compensated for this)
- The system prompt / instructions configuration for MLLM mode (`params.instructions`) is mentioned in the schema but there is no guidance on best practices for Gemini Live prompting vs. standard LLM prompting
- The skill mentions `VOICE_ENABLE_AIVAD=true` and `VOICE_VAD_SILENCE_DURATION_MS` as relevant but does not explain how turn detection interacts with MLLM mode specifically -- Gemini Live has its own server VAD, so it is unclear whether the Agora-side VAD settings are redundant, complementary, or required
- The model ID (`gemini-live-2.5-flash-preview-native-audio-09-2025`) appears to be a preview/versioned identifier that may change over time; there is no guidance on how to find the current model ID
- No mention of GCP quota or rate limits for Gemini Live via Vertex AI, which can be a blocker for new projects
