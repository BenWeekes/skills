# TEN Framework — Graph Configuration

## Graph Structure in property.json

Graphs are defined in `tenapp/property.json` under `ten.predefined_graphs`:

```json
{
  "ten": {
    "predefined_graphs": [{
      "name": "my_graph",
      "auto_start": false,
      "graph": {
        "nodes": [{
          "type": "extension",
          "name": "my_extension",
          "addon": "my_extension_python",
          "extension_group": "default",
          "property": {
            "api_key": "${env:MY_API_KEY|}"
          }
        }],
        "connections": [...]
      }
    }]
  }
}
```

### Node Definition

Each node in `nodes` has:
- **type**: Always `"extension"` for extensions
- **name**: Instance name used in connections
- **addon**: Extension package name (matches directory name in `ten_packages/extension/`)
- **extension_group**: Grouping for execution (typically `"default"`)
- **property**: Configuration values, supports env var substitution

## Connection Types

Connections define data flows between extensions. Each connection specifies a source extension and its outgoing flows:

### cmd — Commands

```json
{
  "extension": "main_control",
  "cmd": [
    {
      "names": ["tool_register"],
      "source": [{"extension": "my_extension"}]
    }
  ]
}
```

### data — Data Messages

```json
{
  "extension": "main_control",
  "data": [
    {
      "name": "asr_result",
      "source": [{"extension": "stt"}]
    }
  ]
}
```

### audio_frame — PCM Audio Streams

```json
{
  "extension": "agora_rtc",
  "audio_frame": [
    {
      "name": "pcm_frame",
      "dest": [
        {"extension": "stt"},
        {"extension": "analyzer"}
      ]
    }
  ]
}
```

### video_frame — Video Streams

```json
{
  "extension": "agora_rtc",
  "video_frame": [
    {
      "name": "video_frame",
      "dest": [{"extension": "vision"}]
    }
  ]
}
```

## Property Environment Variable Substitution

Two forms:
- `${env:VAR_NAME}` — **Required**. Error if environment variable is missing.
- `${env:VAR_NAME|}` — **Optional**. Empty string if missing.

## Keep property.json and rebuild_property.py in Sync

When modifying graphs:

1. Update `rebuild_property.py` first
2. Run the script to regenerate `property.json`
3. Do nuclear restart (if graphs were added/removed)

Or if editing `property.json` directly, also update `rebuild_property.py` to avoid overwriting on next script run.

## Server Architecture — Dynamic Property Injection

The Go server (`server/internal/http_server.go`) auto-injects request parameters into graph nodes at session start.

**Channel Name Auto-Injection**: When `/start` is called with `channel_name`, the server injects this value into **all nodes with a "channel" property**. Both `agora_rtc` and other extensions with a `channel` property receive the dynamic channel value automatically.

### Injected Parameters

| Request Parameter | Injected As |
|-------------------|-------------|
| `channel_name` | `channel` (on all nodes with "channel" property) |
| `RemoteStreamId` | `agora_rtc.remote_stream_id` |
| `BotStreamId` | `agora_rtc.stream_id` |
| `Token` | `agora_rtc.token` and `agora_rtm.token` |

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check |
| `/graphs` | GET | List available graphs |
| `/start` | POST | Start agent session |
| `/stop` | POST | Stop agent session |
| `/list` | GET | List active sessions |
| `/token/generate` | POST | Generate Agora RTC token |

## Testing Workflow

```bash
# 1. Health check
curl -s http://localhost:8080/health

# 2. List graphs
curl -s http://localhost:8080/graphs | jq '.data[].name'

# 3. Start session
curl -X POST http://localhost:8080/start \
  -H "Content-Type: application/json" \
  -d '{"graph_name": "my_graph", "channel_name": "test", "remote_stream_id": 123}'

# 4. List active sessions
curl -s http://localhost:8080/list | jq '.'

# 5. Stop session
curl -X POST http://localhost:8080/stop \
  -H "Content-Type: application/json" \
  -d '{"channel_name": "test"}'
```

## Official Documentation

- https://theten.ai/docs — TEN Framework docs
- https://docs.agora.io/en/ten-framework/overview/product-overview — Agora's TEN docs
