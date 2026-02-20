# TEN Framework — Extension Development

## Extension Directory Structure

```
ten_packages/extension/my_extension_python/
├── __init__.py           # Empty or package init
├── addon.py              # Extension registration
├── extension.py          # Main extension logic
├── manifest.json         # Extension metadata
├── property.json         # Default properties
└── requirements.txt      # Python dependencies
```

## Required Files

### addon.py

Registers the extension with TEN runtime:

```python
from ten_runtime import Addon, register_addon_as_extension
from .extension import MyExtension

@register_addon_as_extension("my_extension_python")
class MyExtensionAddon(Addon):
    def on_create_instance(self, ten_env, name, context):
        return MyExtension(name)
```

### extension.py — Basic AsyncExtension

Full pattern with all lifecycle methods:

```python
from ten_runtime import AsyncExtension, AsyncTenEnv, Cmd, Data, AudioFrame

class MyExtension(AsyncExtension):
    async def on_start(self, ten_env: AsyncTenEnv) -> None:
        # CRITICAL: Property getters return tuples (value, error)
        api_key_result = await ten_env.get_property_string("api_key")
        self.api_key = api_key_result[0] if isinstance(api_key_result, tuple) else api_key_result

        ten_env.log_info("Extension started")
        ten_env.on_start_done()

    async def on_stop(self, ten_env: AsyncTenEnv) -> None:
        # Cleanup resources here
        ten_env.log_info("Extension stopped")
        ten_env.on_stop_done()

    async def on_cmd(self, ten_env: AsyncTenEnv, cmd: Cmd) -> None:
        cmd_name = cmd.get_name()
        ten_env.log_info(f"Received command: {cmd_name}")
        cmd_result = Cmd.create("cmd_result")
        await ten_env.return_result(cmd_result, cmd)

    async def on_data(self, ten_env: AsyncTenEnv, data: Data) -> None:
        data_name = data.get_name()
        ten_env.log_info(f"Received data: {data_name}")

    async def on_audio_frame(self, ten_env: AsyncTenEnv, audio_frame: AudioFrame) -> None:
        pcm_data = audio_frame.get_buf()
        # Process audio...
```

### extension.py — LLM Tool Extension

For extensions that provide tools to an LLM node:

```python
from ten_ai_base.llm_tool import AsyncLLMToolBaseExtension
from ten_ai_base.types import LLMToolMetadata, LLMToolResult

class MyToolExtension(AsyncLLMToolBaseExtension):
    def get_tool_metadata(self, ten_env) -> list[LLMToolMetadata]:
        return [
            LLMToolMetadata(
                name="my_tool",
                description="Tool description for LLM",
                parameters=[
                    {"name": "param1", "type": "string", "description": "Parameter description"}
                ]
            )
        ]

    async def run_tool(self, ten_env, name: str, args: dict) -> LLMToolResult:
        ten_env.log_info(f"Tool called: {name} with args: {args}")
        return LLMToolResult(type="text", content="Tool result")
```

### manifest.json

Extension metadata and dependencies:

```json
{
  "type": "extension",
  "name": "my_extension_python",
  "version": "0.1.0",
  "dependencies": [],
  "api": {}
}
```

### property.json

Default property values for the extension:

```json
{
  "api_key": "",
  "threshold": 0.5
}
```

### requirements.txt

Python dependencies installed inside the container:

```
requests>=2.28.0
```

## Critical Patterns

### NEVER Use Signal Handlers

Signal handlers only work in the main thread. TEN extensions run in worker threads.

```python
# ❌ WRONG - Will crash with ValueError
signal.signal(signal.SIGTERM, self._cleanup)
atexit.register(self._emergency_cleanup)

# ✅ CORRECT - Use lifecycle methods
async def on_stop(self, ten_env):
    # Cleanup here - always called before termination
    if self.websocket:
        await self.websocket.close()
```

### Property Loading Returns Tuples

ALL property getters return `(value, error_or_none)`:

```python
# ❌ WRONG - Will cause TypeError in comparisons
self.threshold = await ten_env.get_property_float("threshold")

# ✅ CORRECT - Extract first element
threshold_result = await ten_env.get_property_float("threshold")
self.threshold = threshold_result[0] if isinstance(threshold_result, tuple) else threshold_result
```

### Use ten_runtime, Not ten

```python
# ✅ CORRECT (v0.11+)
from ten_runtime import AsyncExtension, AsyncTenEnv

# ❌ WRONG (v0.8.x - old API)
from ten import AsyncExtension
```

## Base Classes Location

Base classes are at:
```
agents/ten_packages/system/ten_ai_base/interface/ten_ai_base/
```

API interface definitions:
```
agents/ten_packages/system/ten_ai_base/api/*.json
```

## Official Documentation

- https://theten.ai/docs — TEN Framework docs
- https://github.com/TEN-framework/ten-framework — GitHub repository
