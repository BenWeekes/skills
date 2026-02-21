# Test: MCP and Custom LLM Features for ConvoAI

## Prompt

What features could we add to an MCP server or custom LLM easily, and how do we use those with Agora Conversational AI?

## Skill Files Loaded

1. `~/.claude/skills/agora/SKILL.md` -- main skill entry point
2. `~/.claude/skills/agora/references/conversational-ai/README.md` -- ConvoAI architecture and REST API overview
3. `~/.claude/skills/agora/references/conversational-ai/recipes.md` -- 5 official recipe repos (agent-samples, server-custom-llm, server-mcp, agent-toolkit, agent-ui-kit)
4. `~/.claude/skills/agora/references/conversational-ai/agent-config.md` -- full agent properties schema including `mcp_servers`, `vendor: "custom"`, tool calling, `template_variables`, `custom_metadata`
5. `~/.claude/skills/agora/references/conversational-ai/rest-api.md` -- all REST endpoints (join, leave, update, speak, interrupt, history)
6. `~/.claude/skills/agora/references/conversational-ai/web-client.md` -- `@agora/conversational-ai` agent-toolkit SDK
7. External: `https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/AGENT.md` -- profile-based config, MLLM variables, LLM config fields, companion server integration, debugging
8. External: `https://docs.agora.io/en/conversational-ai/develop/custom-llm` -- custom LLM proxy specification (request/response format, SSE streaming, tool calling flow, `custom_metadata` for interruptable control)
9. External: GitHub API `repos/AgoraIO-Conversational-AI/server-mcp/readme` -- MCP Memory Server README (tools, URL scheme, integration, architecture)
10. External: GitHub API `repos/AgoraIO-Conversational-AI/server-custom-llm/readme` -- Custom LLM Server README (endpoints, features, tool execution, conversation memory, RAG, RTM integration)

## Plan

### 1. What Features an MCP Server Can Provide

The MCP (Model Context Protocol) server is a standalone service that exposes tools to the ConvoAI agent via the Streamable HTTP transport. The agent calls these tools during conversation just like it would call any LLM function -- the Agora ConvoAI Engine handles the MCP protocol lifecycle (initialize, tools/list, tools/call) automatically when `mcp_servers` is configured in the agent's LLM config.

#### Existing MCP Memory Server Tools

The `server-mcp` recipe already provides 6 tools via SQLite + FTS5:

| Tool | What It Does |
|------|-------------|
| `save_memory` | Persist text with optional category and metadata, scoped per user |
| `search_memory` | Full-text search with BM25 ranking across stored memories |
| `list_memories` | List all or filtered memories, newest first |
| `delete_memory` | Remove a memory by ID (user-scoped) |
| `compact_memories` | Return all memories with LLM hint to consolidate/summarize them |
| `log_message` | Record a message with role/speaker for audit trails |

#### New MCP Tools That Would Be Easy to Add

**A. User Preferences Store**
- `save_preference(key, value)` -- store user preferences (language, tone, topics of interest)
- `get_preference(key)` -- retrieve a preference
- `list_preferences()` -- list all preferences for the current user
- Implementation: simple key-value table in the existing SQLite database, partitioned by user_id from the URL path (`/mcp/{user_id}`)

**B. Task/Reminder Manager**
- `create_task(description, due_date, priority)` -- create a task or reminder
- `list_tasks(status, priority)` -- list pending tasks
- `complete_task(task_id)` -- mark a task as done
- `search_tasks(query)` -- FTS5 search across task descriptions
- Implementation: new SQLite table with status and priority fields; the agent can proactively remind users about pending tasks using the silence_config feature

**C. Knowledge Base / FAQ Tool**
- `search_knowledge(query, category)` -- search a pre-loaded knowledge base
- `get_article(article_id)` -- retrieve a specific article
- Implementation: load FAQ or product docs into a SQLite FTS5 table at server startup; the agent can look up answers during conversation

**D. Conversation Summary**
- `save_summary(session_id, summary)` -- store an LLM-generated conversation summary
- `get_summaries(limit)` -- retrieve past conversation summaries
- Implementation: leverages `compact_memories` pattern -- the LLM generates a summary at end of conversation, saves it via MCP tool

**E. Calendar / Scheduling**
- `check_availability(date, time_range)` -- query a calendar API
- `book_appointment(date, time, description)` -- create a calendar event
- Implementation: thin wrapper around Google Calendar or Cal.com API, exposed as MCP tools

**F. CRM Contact Lookup**
- `lookup_contact(name_or_email)` -- search a contacts database
- `update_contact(contact_id, fields)` -- update contact info
- Implementation: SQLite table or proxy to an external CRM API

### 2. What Features a Custom LLM Proxy Can Provide

The custom LLM server sits between the Agora ConvoAI Engine and the LLM provider. The Engine calls your server at the configured `llm.url` endpoint instead of calling OpenAI directly. Your server must return OpenAI-compatible SSE streaming responses.

#### Existing Custom LLM Server Features

The `server-custom-llm` recipe already provides:

- **`/chat/completions`** -- standard LLM proxy with multi-pass tool execution (up to 5 passes)
- **`/rag/chat/completions`** -- RAG-enhanced endpoint that retrieves context before calling the LLM
- **`/audio/chat/completions`** -- multimodal audio response endpoint
- **Conversation memory** -- per-channel message storage (trimmed at 100 messages, keeping 75)
- **Server-side tool execution** -- `get_weather` and `calculate` as examples
- **RTM text messaging** -- Node.js implementation subscribes to RTM for text input alongside voice

#### New Custom LLM Features That Would Be Easy to Add

**A. Enhanced RAG with Vector Search**
- Replace the keyword-based retrieval with an embedding-based vector store (e.g., ChromaDB, FAISS, or pgvector)
- On each user message, compute embeddings, retrieve top-k relevant documents, inject as context before the LLM call
- Implementation: add an embedding call (OpenAI `text-embedding-3-small` or local model) and a vector store in the existing `/rag/chat/completions` endpoint

**B. Conversation History Summarization**
- Before trimming at 100 messages, summarize older messages using the LLM
- Inject the summary as a system message so the agent maintains long-term context without exceeding token limits
- Implementation: add a summarization pass in the conversation memory module when messages exceed a threshold

**C. Guardrails / Content Filtering**
- Inspect user messages before forwarding to the LLM; block or flag inappropriate content
- Inspect LLM responses before streaming back; redact sensitive information (PII, credit card numbers)
- Use the `custom_metadata` first-chunk mechanism to mark compliance responses as non-interruptable:
  ```json
  {"object": "chat.completion.custom_metadata", "metadata": {"interruptable": false}}
  ```
- Implementation: regex-based or LLM-based content classifier in the proxy pipeline

**D. Dynamic System Prompt Injection**
- Based on user identity, time of day, or conversation context, dynamically modify the system prompt
- Example: inject different personas, language preferences, or business rules based on the user's profile
- Implementation: modify the system messages array in the request before forwarding to the LLM

**E. Multi-Provider Failover**
- Route requests to different LLM providers based on availability, latency, or cost
- Primary: GPT-4o; Fallback: Claude; Budget mode: GPT-4o-mini
- Implementation: try-catch around the primary LLM call, fall back to secondary on failure

**F. Analytics and Logging**
- Log every request/response pair with timestamps, token counts, latency metrics
- Track which tools are called most frequently, average response times, error rates
- Implementation: async logging to a database or analytics service in the proxy pipeline

**G. Structured Output Enforcement**
- Use `response_format` with JSON schema to ensure the LLM returns structured data when needed (e.g., for tool calls or form filling)
- Parse and validate the response before streaming back
- Implementation: add schema validation middleware in the proxy

**H. Custom Tool Execution with External APIs**
- Weather, stock prices, flight status, order tracking -- any external API can be wrapped as a tool
- The custom LLM server already handles multi-pass tool execution; just add new tool definitions and handlers
- Implementation: add tool schema to `tools.py/js/go` and implement the handler function with signature `(appId, userId, channel, args) -> string`

### 3. How to Configure ConvoAI Agents to Use MCP Servers and Custom LLM Endpoints

#### Configuring MCP Servers

MCP servers are configured in the `properties.llm.mcp_servers` array in the agent join payload. The Agora ConvoAI Engine handles the full MCP lifecycle automatically.

```json
{
  "name": "agent-with-mcp",
  "properties": {
    "channel": "my-channel",
    "token": "007eJx...",
    "agent_rtc_uid": "0",
    "remote_rtc_uids": ["12345"],
    "advanced_features": {
      "enable_tools": true
    },
    "llm": {
      "url": "https://api.openai.com/v1/chat/completions",
      "api_key": "sk-...",
      "system_messages": [
        {
          "role": "system",
          "content": "You are a helpful assistant with memory. Use save_memory to remember important things the user tells you. Use search_memory to recall past conversations."
        }
      ],
      "params": {
        "model": "gpt-4o",
        "max_tokens": 1024
      },
      "max_history": 32,
      "greeting_message": "Hello! I remember our past conversations. How can I help?",
      "mcp_servers": [
        {
          "url": "https://your-tunnel.trycloudflare.com/mcp/{user_id}"
        }
      ]
    },
    "tts": {
      "vendor": "microsoft",
      "params": {
        "key": "your-azure-key",
        "region": "eastus",
        "voice_name": "en-US-AndrewMultilingualNeural"
      }
    }
  }
}
```

Key points:
- The `{user_id}` in the MCP URL is the user's UID, providing per-user memory isolation
- `enable_tools: true` must be set in `advanced_features` for tool calling to work
- The MCP server must be publicly accessible (use cloudflared tunnel for local development)
- The LLM system prompt should instruct the agent on when and how to use the memory tools

#### Configuring a Custom LLM Endpoint

To use a custom LLM proxy, point `properties.llm.url` to your server and set `vendor: "custom"`:

```json
{
  "name": "agent-with-custom-llm",
  "properties": {
    "channel": "my-channel",
    "token": "007eJx...",
    "agent_rtc_uid": "0",
    "remote_rtc_uids": ["12345"],
    "advanced_features": {
      "enable_tools": true
    },
    "llm": {
      "url": "https://your-tunnel.trycloudflare.com/chat/completions",
      "api_key": "optional-passthrough-key",
      "vendor": "custom",
      "system_messages": [
        {
          "role": "system",
          "content": "You are a customer support agent for Acme Corp."
        }
      ],
      "params": {
        "model": "gpt-4o",
        "max_tokens": 1024
      },
      "max_history": 32,
      "greeting_message": "Welcome to Acme support! How can I help?"
    },
    "tts": {
      "vendor": "elevenlabs",
      "params": {
        "key": "your-elevenlabs-key",
        "voice_id": "your-voice-id",
        "model_id": "eleven_turbo_v2_5"
      }
    }
  }
}
```

Key points:
- Setting `vendor: "custom"` causes the Engine to include `turn_id` and `timestamp` in requests to your server
- The Engine also sends a `context` object with `appId`, `userId`, and `channel` -- use these for per-user conversation memory
- Your server must return OpenAI-compatible SSE streaming responses
- For RAG, use the `/rag/chat/completions` endpoint path instead

#### Using Both MCP and Custom LLM Together

You can combine both approaches. Use the custom LLM for request/response pipeline control (RAG, guardrails, analytics) and MCP for persistent tool-based capabilities (memory, tasks):

```json
{
  "name": "agent-full-stack",
  "properties": {
    "channel": "my-channel",
    "token": "007eJx...",
    "agent_rtc_uid": "0",
    "remote_rtc_uids": ["12345"],
    "advanced_features": {
      "enable_tools": true
    },
    "llm": {
      "url": "https://custom-llm-tunnel.trycloudflare.com/rag/chat/completions",
      "api_key": "your-key",
      "vendor": "custom",
      "system_messages": [
        {
          "role": "system",
          "content": "You are a knowledgeable assistant. Use your memory tools to remember user preferences and past conversations. Use RAG context provided to answer product questions accurately."
        }
      ],
      "params": {
        "model": "gpt-4o",
        "max_tokens": 1024,
        "tools": [
          {
            "type": "function",
            "function": {
              "name": "get_order_status",
              "description": "Look up the status of a customer order",
              "parameters": {
                "type": "object",
                "properties": {
                  "order_id": { "type": "string", "description": "The order ID" }
                },
                "required": ["order_id"]
              }
            }
          }
        ]
      },
      "max_history": 32,
      "greeting_message": "Hi! I'm your support assistant. I have access to your order history and our knowledge base.",
      "mcp_servers": [
        {
          "url": "https://mcp-tunnel.trycloudflare.com/mcp/{user_id}"
        }
      ]
    },
    "tts": {
      "vendor": "microsoft",
      "params": {
        "key": "your-azure-key",
        "region": "eastus",
        "voice_name": "en-US-AndrewMultilingualNeural"
      }
    },
    "parameters": {
      "silence_config": {
        "timeout_ms": 10000,
        "action": "speak",
        "content": "Check if the user needs anything else"
      }
    }
  }
}
```

### 4. Concrete Feature Ideas That Would Be Easy to Implement

Here are 10 features ranked by implementation effort (easiest first):

#### Tier 1: Afternoon Project (1-3 hours)

**1. Persistent User Preferences (MCP)**
- Add `save_preference` and `get_preference` tools to the existing MCP server
- One new SQLite table, two new tool handlers
- System prompt: "When the user states a preference, save it. Check preferences at conversation start."

**2. Custom Greeting Based on History (Custom LLM)**
- In the custom LLM proxy, check conversation memory on the first message
- If returning user, inject context: "This user has spoken with you 3 times before. They prefer concise answers."
- Modify system messages dynamically before forwarding to LLM

**3. Non-Interruptable Compliance Messages (Custom LLM)**
- Add a keyword/intent detector in the proxy pipeline
- When the conversation touches compliance topics (pricing, legal, terms), send the `custom_metadata` first chunk with `interruptable: false`
- Implementation: regex or simple classifier before streaming

**4. Conversation Analytics Logger (Custom LLM)**
- Add async logging middleware to the proxy
- Log: timestamp, user_id, channel, message_role, token_count, latency_ms, tool_calls
- Write to SQLite or send to an analytics service
- Zero impact on response latency (async fire-and-forget)

#### Tier 2: Day Project (4-8 hours)

**5. FAQ Knowledge Base (MCP or Custom LLM)**
- Pre-load product documentation into SQLite FTS5 table
- Expose as `search_knowledge` MCP tool, or use in the RAG endpoint
- BM25 ranking provides good relevance without embeddings

**6. Order/Appointment Lookup (Custom LLM Tool)**
- Add `get_order_status` and `book_appointment` tool definitions
- Implement handlers that call your backend API
- Multi-pass tool execution is already built into server-custom-llm

**7. Conversation Summarization (Custom LLM)**
- Before trimming conversation history, send older messages to the LLM for summarization
- Store the summary as a system message for future turns
- Gives the agent long-term conversation awareness without token overflow

#### Tier 3: Weekend Project (1-2 days)

**8. Vector RAG with Embeddings (Custom LLM)**
- Add ChromaDB or FAISS to the custom LLM server
- Compute embeddings for user queries, retrieve top-k relevant documents
- Inject retrieved context into the `/rag/chat/completions` pipeline
- Significantly better retrieval quality than keyword-based BM25

**9. Multi-Agent Orchestration (Custom LLM)**
- Custom LLM proxy routes requests to different specialized LLMs based on intent
- "I need help with billing" -> billing-specialized prompt + tools
- "Tell me about the product" -> product knowledge prompt + RAG
- Implementation: intent classifier in the proxy, conditional prompt/tool selection

**10. Voice-Activated Workflows (MCP + Custom LLM)**
- Combine task management (MCP) with workflow execution (Custom LLM tools)
- User says "Schedule a follow-up call for next Tuesday" -> create_task (MCP) + book_appointment (Custom LLM tool) + send confirmation email (Custom LLM tool)
- Multi-pass tool execution chains these steps automatically

### 5. How to Wire It All Together

#### Development Setup

```
Terminal 1: MCP Memory Server
  cd server-mcp/python
  python3 mcp_server.py --port 8090

Terminal 2: Cloudflare Tunnel for MCP
  cloudflared tunnel --url http://localhost:8090
  # Note the URL: https://xxx.trycloudflare.com

Terminal 3: Custom LLM Server
  cd server-custom-llm/python
  export LLM_API_KEY=sk-your-openai-key
  python3 custom_llm.py  # Port 8100

Terminal 4: Cloudflare Tunnel for Custom LLM
  cloudflared tunnel --url http://localhost:8100
  # Note the URL: https://yyy.trycloudflare.com

Terminal 5: Agent Samples Backend
  cd agent-samples/simple-backend
  # Set .env with Agora credentials + tunnel URLs:
  #   MCP_MEMORY_URL=https://xxx.trycloudflare.com/mcp/{user_id}
  #   LLM_URL=https://yyy.trycloudflare.com/chat/completions
  #   LLM_VENDOR=custom
  python3 local_server.py  # Port 8082

Terminal 6: React Voice Client
  cd agent-samples/react-voice-client
  npm install --legacy-peer-deps && npm run dev  # Port 8083
```

#### Production Deployment

1. Deploy MCP server and Custom LLM server to your cloud (EC2, Cloud Run, etc.)
2. Use real domain names with TLS instead of cloudflared tunnels
3. Configure agent-samples backend `.env` with production URLs
4. Set `idle_timeout` appropriately to avoid orphaned agents
5. Monitor with `enable_metrics: true` and `enable_error_message: true`

#### End-to-End Flow

```
User speaks into browser
  -> RTC audio to Agora ConvoAI Engine
    -> ASR converts speech to text
      -> Engine sends text to Custom LLM Server (/chat/completions or /rag/chat/completions)
        -> Custom LLM loads conversation history
        -> Custom LLM injects RAG context (if /rag endpoint)
        -> Custom LLM forwards to LLM provider (OpenAI, etc.)
        -> LLM may return tool_calls
          -> Custom LLM executes tools locally (get_weather, get_order_status, etc.)
          -> OR Engine calls MCP server tools (save_memory, search_memory, etc.)
        -> LLM returns final text response
        -> Custom LLM saves to conversation memory
        -> Custom LLM streams SSE response back to Engine
      -> Engine sends text to TTS
    -> TTS converts to speech
  -> RTC audio back to user's browser
  -> Transcripts arrive via RTC data channel or RTM
```

#### Adding a New MCP Tool (Step by Step)

1. Open `server-mcp/python/mcp_server.py`
2. Add a new tool definition in the `tools/list` handler:
   ```python
   {
     "name": "save_preference",
     "description": "Save a user preference (e.g., language, tone, topic interest)",
     "inputSchema": {
       "type": "object",
       "properties": {
         "key": {"type": "string", "description": "Preference key"},
         "value": {"type": "string", "description": "Preference value"}
       },
       "required": ["key", "value"]
     }
   }
   ```
3. Add a handler in the `tools/call` dispatcher
4. Add a SQLite table for preferences (partitioned by user_id)
5. Update the agent system prompt to instruct the LLM to use the new tool

#### Adding a New Custom LLM Tool (Step by Step)

1. Open `server-custom-llm/python/tools.py`
2. Add the OpenAI-compatible function schema:
   ```python
   {
     "type": "function",
     "function": {
       "name": "get_order_status",
       "description": "Look up the status of a customer order by order ID",
       "parameters": {
         "type": "object",
         "properties": {
           "order_id": {"type": "string", "description": "The order ID to look up"}
         },
         "required": ["order_id"]
       }
     }
   }
   ```
3. Implement the handler:
   ```python
   def get_order_status(app_id, user_id, channel, args):
       order_id = args.get("order_id")
       # Call your order API
       return json.dumps({"order_id": order_id, "status": "shipped", "eta": "2026-02-25"})
   ```
4. Register it in the tool map
5. The multi-pass execution loop already handles calling the tool and feeding results back to the LLM

### Summary Table

| Feature | Where to Build | Effort | Value |
|---------|---------------|--------|-------|
| User preferences | MCP server | 1-2 hrs | High -- personalization |
| Custom greeting from history | Custom LLM | 1-2 hrs | Medium -- returning user experience |
| Non-interruptable compliance | Custom LLM | 1-2 hrs | High -- regulatory requirements |
| Analytics logger | Custom LLM | 2-3 hrs | Medium -- operational visibility |
| FAQ knowledge base | MCP or Custom LLM | 4-6 hrs | High -- reduces LLM hallucination |
| Order/appointment lookup | Custom LLM tools | 4-6 hrs | High -- real business actions |
| Conversation summarization | Custom LLM | 4-8 hrs | Medium -- long conversations |
| Vector RAG | Custom LLM | 1-2 days | High -- better retrieval quality |
| Multi-agent orchestration | Custom LLM | 1-2 days | High -- specialized handling |
| Voice-activated workflows | MCP + Custom LLM | 1-2 days | High -- complex automations |

## Assessment

**Rating: 4 / 5**

### What the Skill Covered Well

- **MCP server architecture**: The skill clearly documents the `server-mcp` recipe with its 6 tools, URL scheme (`/mcp/{user_id}` for per-user isolation), SQLite + FTS5 architecture, and port assignments. The `mcp_servers` field in the LLM config is documented with its type (array) and purpose.
- **Custom LLM proxy**: The skill provides excellent coverage of the `server-custom-llm` recipe -- three endpoints (`/chat/completions`, `/rag/chat/completions`, `/audio/chat/completions`), multi-pass tool execution, conversation memory with per-channel storage, and the SSE streaming requirement.
- **Agent configuration**: The `agent-config.md` file documents every relevant field: `llm.url`, `vendor: "custom"`, `mcp_servers`, `enable_tools`, `template_variables`, plus the `custom_metadata` interruptable control mechanism.
- **Wiring it together**: The `recipes.md` architecture diagram showing how all 5 repos connect, plus the AGENT.md external guide with profile-based configuration and companion server patterns, made it straightforward to understand the full deployment topology.
- **Concrete examples**: The skill provides working JSON payloads for agent join requests, custom LLM configuration, and tool definitions that serve as templates for new features.

### What Was Missing or Could Be Improved

- **`mcp_servers` detailed schema**: The skill documents that `mcp_servers` is an array field in the LLM config, but does not provide the full schema for each server entry (e.g., what fields beyond `url` are supported -- authentication headers? timeout settings? tool filtering?). The MCP server README provides the server-side URL scheme but not the client-side configuration format that goes into the join payload. This required inferring from the server-mcp README and the Agora docs.
- **MCP + Custom LLM interaction**: The skill does not explicitly document whether MCP tool calls and custom LLM tool calls can coexist in the same agent, or whether there are priority/ordering semantics. The plan above assumes they can be combined based on the architecture, but explicit documentation would help.
- **Custom LLM request `context` field**: The server-custom-llm README documents that the Engine sends a `context` object with `appId`, `userId`, and `channel`, but this is not mentioned in the skill's agent-config.md or rest-api.md. This is important for implementing per-user conversation memory in the custom LLM.
- **Rate limits on tool calls**: No documentation on whether there are limits on the number of MCP tool calls per turn, or latency budgets for tool execution before the Engine times out.
