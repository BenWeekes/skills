# MCP Memory Server

Multi-user MCP server with persistent memory and full-text search for Agora Conversational AI agents.

**Repo:** https://github.com/AgoraIO-Conversational-AI/server-mcp-memory

## Implementations

| Language | Framework | Port |
|----------|-----------|------|
| Python | Starlette | 8090 |
| Node.js | Express | 8091 |
| Go | Gin | 8092 |

## MCP Tools

- `save_memory` — store a memory with category and tags
- `search_memory` — BM25 full-text search
- `list_memories` — list by category
- `delete_memory` — delete by ID
- `compact_memories` — merge related memories
- `log_message` — append to conversation log

> **[README — Tools](https://github.com/AgoraIO-Conversational-AI/server-mcp-memory#mcp-tools)**

## Integration with ConvoAI

Configure `MCP_SERVERS` JSON array in agent-samples `.env`. Uses `build_mcp_servers` function.

> **[README — Integration](https://github.com/AgoraIO-Conversational-AI/server-mcp-memory#integration-with-agora-conversational-ai)**

## Architecture

- SQLite with FTS5 full-text search
- Per-user memory isolation via URL path (`/mcp/{user_id}`)
- MCP Streamable HTTP protocol

> **[README — Architecture](https://github.com/AgoraIO-Conversational-AI/server-mcp-memory#architecture)**
