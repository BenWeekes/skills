# Test: ConvoAI Recipe Repos Setup

## Prompt

What are the Agora Conversational AI recipe repos and how do I set them all up together?

## Skill Files Loaded

1. `~/.claude/skills/agora/SKILL.md` -- Main skill entry point; identified ConvoAI product and pointed to references
2. `~/.claude/skills/agora/references/conversational-ai/README.md` -- ConvoAI architecture overview, REST API auth, agent lifecycle, and pointers to topic files
3. `~/.claude/skills/agora/references/conversational-ai/recipes.md` -- The primary file: all 5 recipe repos, quick starts, architecture diagram, port reference
4. `~/.claude/skills/agora/references/conversational-ai/agent-config.md` -- Agent join payload configuration (LLM, TTS, ASR, MLLM, tools, MCP servers)
5. `~/.claude/skills/agora/references/conversational-ai/web-client.md` -- agent-toolkit SDK details (RTCHelper, RTMHelper, ConversationalAIAPI, React hooks)
6. `~/.claude/skills/agora/references/conversational-ai/rest-api.md` -- Full REST API endpoint details (join, leave, update, speak, interrupt, history)
7. `~/.claude/skills/agora/references/server/README.md` -- Server-side token generation overview

## Plan

### Overview

Agora Conversational AI has **5 official recipe repos** that form a complete developer recipe set. They can be used independently or together to build a full-stack voice AI application. The architecture looks like this:

```
Browser                          Your Servers                    Agora Cloud
+---------------------+    +----------------------+    +------------------+
| agent-toolkit (SDK) |--->| agent-samples        |--->| Agora ConvoAI    |
| agent-ui-kit (UI)   |    | (backend, port 8082) |    | REST API         |
+---------------------+    +------+-------+-------+    +------------------+
                                  |       |
                           +------v---+ +-v----------------+
                           | Custom   | | MCP Memory       |
                           | LLM      | | Server           |
                           | 8100-02  | | 8090-92          |
                           +----------+ +------------------+
```

### Prerequisites

Before setting up any repos, you need:

1. **Agora Account** -- Sign up at [console.agora.io](https://console.agora.io)
   - Create a project and get your **App ID**
   - Get **Customer ID** and **Customer Secret** from Developer Toolkit > RESTful API (these are for the Basic Auth header used by the ConvoAI REST API)
   - Optionally get your **App Certificate** if you want token-based auth (recommended for production)
   - For development/testing, you can disable token authentication in Agora Console and pass `null` as the token

2. **LLM API Key** -- An OpenAI API key (or compatible provider) for the AI agent's language model

3. **TTS Credentials** -- At least one TTS provider:
   - Microsoft Azure Speech: key + region
   - ElevenLabs: API key + voice ID
   - OpenAI TTS: API key

4. **Runtime Dependencies**:
   - Python 3.8+ with pip
   - Node.js 18+ with npm
   - Go 1.21+ (optional, only if using Go implementations)

---

### Step 1: Clone All Repos

```bash
mkdir agora-convoai && cd agora-convoai

git clone https://github.com/AgoraIO-Conversational-AI/agent-samples.git
git clone https://github.com/AgoraIO-Conversational-AI/server-custom-llm.git
git clone https://github.com/AgoraIO-Conversational-AI/server-mcp.git
git clone https://github.com/AgoraIO-Conversational-AI/agent-toolkit.git
git clone https://github.com/AgoraIO-Conversational-AI/agent-ui-kit.git
```

---

### Step 2: Set Up the Backend -- agent-samples (Port 8082)

This is the core backend. It is a Python Flask server that manages agent lifecycle, generates tokens, and provides profile-based configuration. The backend calls the Agora ConvoAI REST API to start/stop agents.

```bash
cd agent-samples/simple-backend

# Copy and configure environment
cp .env.example .env
```

Edit `.env` with your credentials. The key variables are:

- `AGORA_APP_ID` -- Your Agora App ID
- `AGORA_APP_CERTIFICATE` -- Your Agora App Certificate (for token generation)
- `AGORA_CUSTOMER_ID` -- Customer ID for REST API auth
- `AGORA_CUSTOMER_SECRET` -- Customer Secret for REST API auth
- LLM settings (e.g., `OPENAI_API_KEY`, `LLM_URL`, `LLM_MODEL`)
- TTS settings (e.g., `TTS_VENDOR`, `AZURE_TTS_KEY`, `AZURE_TTS_REGION`, `AZURE_TTS_VOICE`)

```bash
# Install dependencies
pip3 install -r requirements-local.txt

# Start the backend
python3 local_server.py
# Runs on port 8082
```

The backend exposes endpoints that the frontend clients call to start/stop agents. Under the hood, it constructs the ConvoAI REST API `/join` payload with the appropriate `properties` object (LLM config, TTS config, ASR config, channel info, tokens) and sends it to `https://api.agora.io/api/conversational-ai-agent/v2/projects/{appid}/join`.

**Note**: The backend supports profile-based configuration. See the coding guide at `agent-samples/AGENT.md` for details on profile config, MLLM/Gemini Live variables, debugging agent failures, and production deployment on EC2+nginx.

---

### Step 3: Set Up the Custom LLM Server -- server-custom-llm (Ports 8100-8102)

This is an optional but powerful intermediary that sits between the Agora ConvoAI engine and your actual LLM provider. It intercepts LLM requests to add RAG retrieval, tool calling, conversation memory, and custom prompt logic.

Pick one implementation (Python, Node.js, or Go):

**Python (Port 8100):**
```bash
cd server-custom-llm/python
export LLM_API_KEY=your-openai-key
pip3 install -r requirements.txt
python3 custom_llm.py
# Runs on port 8100
```

**Node.js (Port 8101):**
```bash
cd server-custom-llm/node
export LLM_API_KEY=your-openai-key
npm install
node custom_llm.js
# Runs on port 8101
```

**Go (Port 8102):**
```bash
cd server-custom-llm/go
export LLM_API_KEY=your-openai-key
go run .
# Runs on port 8102
```

The custom LLM server exposes OpenAI-compatible endpoints:
- `/chat/completions` -- Standard chat completions with custom logic
- `/rag/chat/completions` -- Chat completions with RAG retrieval
- `/audio/chat/completions` -- Audio-mode chat completions

Features include multi-pass tool execution, streaming SSE responses, and conversation history management.

**Connecting to the backend**: In the agent-samples backend `.env`, set the LLM URL to point to your custom LLM server instead of directly to OpenAI. For example:
```
LLM_URL=http://localhost:8100/chat/completions
LLM_VENDOR=custom
```

This causes the Agora ConvoAI engine to route LLM requests through your custom server, which then forwards them (with modifications) to the actual LLM provider.

The custom LLM server can also send non-interruptable metadata in the first SSE chunk:
```json
{"object": "chat.completion.custom_metadata", "metadata": {"interruptable": true}}
```
Set `interruptable: false` to prevent user speech from interrupting critical responses like compliance disclaimers.

---

### Step 4: Set Up the MCP Memory Server -- server-mcp (Ports 8090-8092)

This gives your AI agent persistent per-user memory via the Model Context Protocol (MCP). The agent can save and recall facts about users across conversations.

Pick one implementation:

**Python (Port 8090):**
```bash
cd server-mcp/python
pip3 install -r requirements.txt
python3 mcp_server.py
# Runs on port 8090
```

**Node.js (Port 8091):**
```bash
cd server-mcp/node
npm install
node mcp_server.js
# Runs on port 8091
```

**Go (Port 8092):**
```bash
cd server-mcp/go
CGO_ENABLED=1 go run -tags sqlite_fts5 .
# Runs on port 8092
```

The MCP server uses SQLite with FTS5 full-text search and provides these tools to the agent:
- `save_memory` -- Store a fact about the user
- `search_memory` -- Search stored memories (BM25 ranking)
- `list_memories` -- List all memories for a user
- `delete_memory` -- Remove a specific memory
- `compact_memories` -- Consolidate memories
- `log_message` -- Log a conversation message

Features: Sub-100ms tool calls, multi-user isolation, BM25 search ranking.

**Connecting to the agent**: The MCP server is referenced in the agent's join configuration via the `properties.llm.mcp_servers` array. You also need `advanced_features.enable_tools: true` to enable function/tool calling. The backend's profile config should include the MCP server URL so the ConvoAI engine knows to connect to it for tool calls.

---

### Step 5: Set Up the Frontend -- React Clients from agent-samples (Ports 8083-8084)

The agent-samples repo includes two Next.js frontend clients:

**React Voice Client (Port 8083):**
```bash
cd agent-samples/react-voice-client
npm install --legacy-peer-deps
npm run dev
# Runs on port 8083
```

**React Video Client with Avatar (Port 8084):**
```bash
cd agent-samples/react-video-client-avatar
npm install --legacy-peer-deps
npm run dev
# Runs on port 8084
```

The voice client provides a real-time voice conversation interface with transcription. The video client adds avatar support (HeyGen or Anam) so the agent has a visual presence.

Both clients connect to the backend on port 8082 to start/stop agents, and then join the Agora RTC channel directly to exchange audio (and video for the avatar client).

**Simple HTML clients**: The agent-samples repo also includes standalone HTML/JS clients for quick testing without a build step.

---

### Step 6: Use the SDK Packages -- agent-toolkit and agent-ui-kit

If you want to build a **custom frontend** rather than using the pre-built clients, use these two npm packages:

**agent-toolkit** (`@agora/conversational-ai`):
```bash
npm install @agora/conversational-ai
# Peer deps: agora-rtc-sdk-ng ^4.23.4, agora-rtm ^2.0.0
# Requires Agora Video SDK v4.5.1+ for full compatibility
```

This is a framework-agnostic TypeScript SDK (~30KB raw, ~8KB gzipped, zero runtime deps) providing:
- `RTCHelper` -- Singleton wrapper for Agora RTC client lifecycle (join, publish, subscribe, mute, events)
- `RTMHelper` -- Simplified RTM messaging with AI agents
- `ConversationalAIAPI` -- Orchestrates RTC + RTM + transcript rendering
- `SubRenderController` -- Message assembly, deduplication, PTS-synchronized word-level rendering
- React hooks: `useLocalVideo`, `useRemoteVideo`

**agent-ui-kit** (`@agora/agent-ui-kit`):
```bash
npm install @agora/agent-ui-kit
```

Pre-built React UI components:
- Voice: MicButton, AgentVisualizer, AudioVisualizer, LiveWaveform
- Chat: Conversation, Message, ConvoTextStream
- Video: AvatarVideoDisplay, LocalVideoPreview, CameraSelector
- Layout: VideoGrid, MobileTabs
- Primitives: Button, Card, Dialog, DropdownMenu, and more

A minimal custom client using these packages looks like:

```typescript
import { ConversationalAIAPI, RTCHelper } from "@agora/conversational-ai"

const rtcHelper = RTCHelper.getInstance()
await rtcHelper.init({ appId: "YOUR_APP_ID", channel: "test-channel", token: null, uid: 12345 })
await rtcHelper.createAudioTrack({ encoderConfig: "high_quality_stereo", AEC: true, ANS: true, AGC: true })
await rtcHelper.join()
await rtcHelper.publish()

const api = ConversationalAIAPI.init({
  rtcEngine: rtcHelper.client!,
  rtmConfig: { appId: "YOUR_APP_ID", uid: "12345", token: null, channel: "test-channel" },
  renderMode: "auto",
})

api.on("transcript-updated", (messages) => {
  messages.forEach(msg => console.log(`${msg.uid === 0 ? "Agent" : "User"}: ${msg.text}`))
})
```

---

### Complete Port Map (All Services Running)

| Server                        | Language | Port |
|-------------------------------|----------|------|
| agent-samples backend         | Python   | 8082 |
| react-voice-client            | Next.js  | 8083 |
| react-video-client-avatar     | Next.js  | 8084 |
| MCP Memory Server             | Python   | 8090 |
| MCP Memory Server             | Node.js  | 8091 |
| MCP Memory Server             | Go       | 8092 |
| Custom LLM Server             | Python   | 8100 |
| Custom LLM Server             | Node.js  | 8101 |
| Custom LLM Server             | Go       | 8102 |

For a typical development setup, you would run:
- 1 backend (port 8082)
- 1 custom LLM server in your preferred language (one of 8100/8101/8102)
- 1 MCP memory server in your preferred language (one of 8090/8091/8092)
- 1 frontend client (port 8083 or 8084)

That is 4 terminal windows / processes minimum.

---

### Data Flow (End to End)

1. **User opens frontend** (port 8083) and clicks "Connect"
2. **Frontend calls backend** (port 8082) to start an agent -- backend generates an RTC token, constructs the join payload, and POSTs to the Agora ConvoAI REST API
3. **Agora ConvoAI engine** creates the agent, which joins the RTC channel
4. **Frontend joins the same RTC channel** using the agent-toolkit SDK and publishes its microphone audio
5. **Agent pipeline**: User speech --> ASR (speech-to-text) --> Custom LLM server (port 8100) --> LLM provider (OpenAI, etc.) --> TTS (text-to-speech) --> Agent publishes audio back to channel
6. **Tool calling**: If `enable_tools` is true, the LLM can invoke tools including the MCP memory server (port 8090) to save/recall user facts
7. **Transcripts** arrive at the frontend via RTC data channel (stream-messages) or RTM, rendered by `SubRenderController`
8. **User clicks "Disconnect"** -- frontend calls backend, which POSTs to `/agents/{agentId}/leave`

---

### Tips

- For development/testing, disable token authentication in Agora Console and pass `null` as the token -- this avoids needing to set up token generation
- The default PCU (Peak Concurrent Users) limit is 20 concurrent agents per App ID; contact Agora support to increase
- Agent idle timeout defaults to 30 seconds -- the agent auto-exits when subscribed users leave the channel
- Use `parameters.data_channel: "rtm"` with `advanced_features.enable_rtm: true` for RTM-based signaling events (agent state changes, metrics, errors)
- For avatar support, configure `properties.avatar` with a HeyGen or Anam vendor and use the video client on port 8084

## Assessment

**Rating: 4/5**

**What the skill covered well:**
- Complete listing of all 5 recipe repos with GitHub URLs, ports, and quick-start commands
- Clear architecture diagram showing how the repos connect
- Detailed port reference table for all services
- Quick start commands for every implementation language (Python, Node.js, Go)
- The agent-config.md file provided excellent detail on how the join payload connects the custom LLM and MCP servers to the agent
- The web-client.md file gave thorough SDK usage patterns for building custom frontends
- The REST API file gave complete endpoint details for understanding the backend's role

**What was missing or could be improved:**
- **Wiring the MCP server to the agent**: The skill mentions `llm.mcp_servers` as a config field but does not provide an example of the exact MCP server configuration object needed in the join payload. A developer would need to consult external docs or the agent-samples AGENT.md coding guide to understand the exact format.
- **Custom LLM wiring details**: While the skill explains the custom LLM endpoints and the `vendor: "custom"` LLM config, it does not provide an explicit example of how the agent-samples backend `.env` should be configured to route through the custom LLM server. The connection between repos is implied but not spelled out step by step.
- **Environment variable reference**: The skill does not list the exact `.env` variables for agent-samples. It says "cp .env.example .env" and "fill in credentials" but does not enumerate the variable names, so the developer must inspect the .env.example file themselves.
- **agent-samples AGENT.md**: The skill references this coding guide with rich content (profile-based config, MLLM variables, EC2+nginx deployment) but does not include it as a loadable skill file -- it is only available if the developer clones the repo.
- **Inter-repo dependency versions**: No guidance on whether specific versions of the repos need to be compatible with each other.
