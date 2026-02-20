# ConvoAI Recipe Repos

Five official repos that form the Agora Conversational AI developer recipe set. Use together or independently.

## Architecture

```
Browser                          Your Servers                    Agora Cloud
┌─────────────────────┐    ┌──────────────────────┐    ┌──────────────────┐
│ agent-toolkit (SDK) │───▶│ agent-samples        │───▶│ Agora ConvoAI    │
│ agent-ui-kit (UI)   │    │ (backend, port 8082) │    │ REST API         │
└─────────────────────┘    └──────┬───────┬───────┘    └──────────────────┘
                                  │       │
                           ┌──────▼──┐ ┌──▼────────────┐
                           │ Custom  │ │ MCP Memory     │
                           │ LLM     │ │ Server         │
                           │ 8100-02 │ │ port 8090      │
                           └─────────┘ └────────────────┘
```

## Repos

### agent-samples

Backend + frontend clients for Agora Conversational AI.

- **Backend** (`simple-backend/`): Python Flask server. Manages agent lifecycle, token generation, profile-based config.
- **React voice client** (`react-voice-client/`): Next.js voice AI client with real-time transcription. Port 8083.
- **React video client** (`react-video-client-avatar/`): Next.js video+avatar client. Port 8084.
- **Simple HTML clients**: Standalone HTML/JS clients for quick testing.

**Quick start:**
```bash
cd agent-samples/simple-backend
cp .env.example .env   # Fill in Agora + LLM + TTS credentials
pip3 install -r requirements-local.txt
python3 local_server.py  # Starts on port 8082

# In another terminal:
cd agent-samples/react-voice-client
npm install --legacy-peer-deps && npm run dev  # Port 8083
```

**Repo:** https://github.com/AgoraIO-Conversational-AI/agent-samples

---

### server-custom-llm

Custom LLM proxy server — intercepts LLM requests for RAG, tool calling, conversation memory, and custom prompts. Implementations in Python, Node.js, and Go.

- **Endpoints:** `/chat/completions`, `/rag/chat/completions`, `/audio/chat/completions`
- **Ports:** Python 8100, Node.js 8101, Go 8102
- **Features:** Multi-pass tool execution, streaming SSE, conversation history, RAG retrieval

**Quick start:**
```bash
cd server-custom-llm/python
export LLM_API_KEY=your-openai-key
pip3 install -r requirements.txt
python3 custom_llm.py  # Port 8100
```

**Repo:** https://github.com/AgoraIO-Conversational-AI/server-custom-llm

---

### server-mcp

MCP memory server — gives agents persistent per-user memory via tool calling (MCP protocol). SQLite + FTS5 full-text search.

- **Port:** 8090 (Python)
- **Tools:** save_memory, search_memory, list_memories, delete_memory, compact_memories, log_message
- **Features:** Sub-100ms tool calls, multi-user isolation, BM25 search ranking

**Quick start:**
```bash
cd server-mcp/python
pip3 install -r requirements.txt
python3 mcp_server.py  # Port 8090
```

**Repo:** https://github.com/AgoraIO-Conversational-AI/server-mcp

---

### agent-toolkit

TypeScript SDK package (`@agora/conversational-ai`) — framework-agnostic core for building ConvoAI clients.

- **Core classes:** `ConversationalAIAPI`, `RTCHelper`, `RTMHelper`, `SubRenderController`
- **React hooks:** `useLocalVideo`, `useRemoteVideo`
- **Features:** Zero runtime dependencies, tree-shakeable, type-safe event system, message deduplication, PTS sync

**Install:**
```bash
npm install @agora/conversational-ai
```

**Repo:** https://github.com/AgoraIO-Conversational-AI/agent-toolkit

---

### agent-ui-kit

React UI component library (`@agora/agent-ui-kit`) — pre-built components for ConvoAI interfaces.

- **Voice:** MicButton, AgentVisualizer, AudioVisualizer, LiveWaveform
- **Chat:** Conversation, Message, ConvoTextStream
- **Video:** AvatarVideoDisplay, LocalVideoPreview, CameraSelector
- **Layout:** VideoGrid, MobileTabs
- **Primitives:** Button, Card, Dialog, DropdownMenu, and more

**Install:**
```bash
npm install @agora/agent-ui-kit
```

**Repo:** https://github.com/AgoraIO-Conversational-AI/agent-ui-kit

---

## Port Reference

| Server | Language | Port |
|--------|----------|------|
| agent-samples backend | Python | 8082 |
| react-voice-client | Next.js | 8083 |
| react-video-client-avatar | Next.js | 8084 |
| MCP Memory | Python | 8090 |
| Custom LLM | Python | 8100 |
| Custom LLM | Node.js | 8101 |
| Custom LLM | Go | 8102 |
