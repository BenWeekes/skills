# Test: Create TEN Extension

## Prompt

How do I create a TEN extension that analyzes audio sentiment?

## Skill Files Loaded

1. `~/.claude/skills/ten/SKILL.md` -- Top-level skill file covering key concepts (extensions, graphs, connections, property files), repository structure, critical rules, restart guidance, and topic index pointing to the three reference files.
2. `~/.claude/skills/ten/references/extensions.md` -- Extension development reference covering directory structure, required files (addon.py, extension.py, manifest.json, property.json, requirements.txt), AsyncExtension and AsyncLLMToolBaseExtension templates, critical patterns (no signal handlers, tuple property loading, ten_runtime imports).
3. `~/.claude/skills/ten/references/graphs.md` -- Graph configuration reference covering property.json structure, node definitions, connection types (cmd, data, audio_frame, video_frame), env var substitution, server-side dynamic property injection, API endpoints, and testing workflow.
4. `~/.claude/skills/ten/references/operations.md` -- Operations reference covering environment setup, Docker usage (always sudo), task commands, nuclear restart procedure, log locations, remote access, production deployment, and common error fixes.

## Plan

### Overview

Build a custom TEN extension called `audio_sentiment_python` that receives PCM audio frames from the Agora RTC extension, buffers them, sends audio chunks to a sentiment analysis API (e.g., Hume AI or a custom model), and outputs sentiment results as `data` messages that downstream extensions (such as an LLM) can consume.

### 1. File Structure

Create the extension directory at:

```
agents/ten_packages/extension/audio_sentiment_python/
├── __init__.py           # Empty package init
├── addon.py              # Extension registration
├── extension.py          # Main extension logic
├── manifest.json         # Extension metadata
├── property.json         # Default properties
└── requirements.txt      # Python dependencies
```

### 2. addon.py -- Extension Registration

```python
from ten_runtime import Addon, register_addon_as_extension
from .extension import AudioSentimentExtension

@register_addon_as_extension("audio_sentiment_python")
class AudioSentimentAddon(Addon):
    def on_create_instance(self, ten_env, name, context):
        return AudioSentimentExtension(name)
```

### 3. extension.py -- Main Extension Logic

```python
import asyncio
import aiohttp
import base64
import json
from ten_runtime import AsyncExtension, AsyncTenEnv, Cmd, Data, AudioFrame

class AudioSentimentExtension(AsyncExtension):
    """
    Receives PCM audio frames, buffers them, and periodically sends
    buffered audio to a sentiment analysis API. Emits sentiment results
    as data messages.
    """

    def __init__(self, name: str):
        super().__init__(name)
        self.api_key = ""
        self.api_url = ""
        self.buffer_duration_ms = 3000  # Analyze every 3 seconds
        self.sample_rate = 16000
        self.channels = 1
        self.bytes_per_sample = 2  # 16-bit PCM
        self.audio_buffer = bytearray()
        self.buffer_size_target = 0
        self.running = False
        self._analysis_task = None

    async def on_start(self, ten_env: AsyncTenEnv) -> None:
        # CRITICAL: Property getters return tuples (value, error)
        api_key_result = await ten_env.get_property_string("api_key")
        self.api_key = (
            api_key_result[0]
            if isinstance(api_key_result, tuple)
            else api_key_result
        )

        api_url_result = await ten_env.get_property_string("api_url")
        self.api_url = (
            api_url_result[0]
            if isinstance(api_url_result, tuple)
            else api_url_result
        )

        buffer_ms_result = await ten_env.get_property_int(
            "buffer_duration_ms"
        )
        self.buffer_duration_ms = (
            buffer_ms_result[0]
            if isinstance(buffer_ms_result, tuple)
            else buffer_ms_result
        )

        sample_rate_result = await ten_env.get_property_int("sample_rate")
        self.sample_rate = (
            sample_rate_result[0]
            if isinstance(sample_rate_result, tuple)
            else sample_rate_result
        )

        # Calculate how many bytes we need before sending for analysis
        self.buffer_size_target = (
            self.sample_rate
            * self.channels
            * self.bytes_per_sample
            * self.buffer_duration_ms
            // 1000
        )

        self.running = True

        ten_env.log_info(
            f"AudioSentiment started: api_url={self.api_url}, "
            f"buffer_duration={self.buffer_duration_ms}ms, "
            f"sample_rate={self.sample_rate}, "
            f"buffer_target={self.buffer_size_target} bytes"
        )
        ten_env.on_start_done()

    async def on_stop(self, ten_env: AsyncTenEnv) -> None:
        # CRITICAL: Use lifecycle methods for cleanup, NOT signal handlers
        self.running = False
        if self._analysis_task and not self._analysis_task.done():
            self._analysis_task.cancel()
        self.audio_buffer.clear()
        ten_env.log_info("AudioSentiment stopped")
        ten_env.on_stop_done()

    async def on_audio_frame(
        self, ten_env: AsyncTenEnv, audio_frame: AudioFrame
    ) -> None:
        """Receive PCM audio frames and buffer them."""
        pcm_data = audio_frame.get_buf()
        self.audio_buffer.extend(pcm_data)

        # When buffer reaches target size, analyze it
        if len(self.audio_buffer) >= self.buffer_size_target:
            chunk = bytes(self.audio_buffer[: self.buffer_size_target])
            self.audio_buffer = self.audio_buffer[
                self.buffer_size_target :
            ]

            # Run analysis without blocking audio frame processing
            self._analysis_task = asyncio.create_task(
                self._analyze_and_emit(ten_env, chunk)
            )

    async def _analyze_and_emit(
        self, ten_env: AsyncTenEnv, audio_chunk: bytes
    ) -> None:
        """Send audio to sentiment API and emit results as data."""
        try:
            sentiment_result = await self._call_sentiment_api(
                audio_chunk
            )

            if sentiment_result:
                # Emit sentiment as a data message
                data = Data.create("sentiment_result")
                data.set_property_string(
                    "sentiment", sentiment_result.get("sentiment", "neutral")
                )
                data.set_property_float(
                    "confidence",
                    sentiment_result.get("confidence", 0.0),
                )
                data.set_property_string(
                    "emotions",
                    json.dumps(
                        sentiment_result.get("emotions", {})
                    ),
                )
                await ten_env.send_data(data)

                ten_env.log_info(
                    f"Sentiment: {sentiment_result.get('sentiment')} "
                    f"(confidence: {sentiment_result.get('confidence', 0):.2f})"
                )
        except Exception as e:
            ten_env.log_error(f"Sentiment analysis failed: {e}")

    async def _call_sentiment_api(
        self, audio_chunk: bytes
    ) -> dict | None:
        """Call external sentiment analysis API with audio data."""
        audio_b64 = base64.b64encode(audio_chunk).decode("utf-8")

        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }

        payload = {
            "audio": audio_b64,
            "encoding": "pcm_s16le",
            "sample_rate": self.sample_rate,
            "channels": self.channels,
        }

        async with aiohttp.ClientSession() as session:
            async with session.post(
                self.api_url,
                headers=headers,
                json=payload,
                timeout=aiohttp.ClientTimeout(total=10),
            ) as response:
                if response.status == 200:
                    return await response.json()
                else:
                    error_text = await response.text()
                    raise RuntimeError(
                        f"API returned {response.status}: {error_text}"
                    )

    async def on_cmd(
        self, ten_env: AsyncTenEnv, cmd: Cmd
    ) -> None:
        cmd_name = cmd.get_name()
        ten_env.log_info(f"Received command: {cmd_name}")
        cmd_result = Cmd.create("cmd_result")
        await ten_env.return_result(cmd_result, cmd)

    async def on_data(
        self, ten_env: AsyncTenEnv, data: Data
    ) -> None:
        data_name = data.get_name()
        ten_env.log_info(f"Received data: {data_name}")
```

### 4. manifest.json

```json
{
  "type": "extension",
  "name": "audio_sentiment_python",
  "version": "0.1.0",
  "dependencies": [],
  "api": {}
}
```

### 5. property.json (Extension Defaults)

```json
{
  "api_key": "",
  "api_url": "https://api.hume.ai/v0/batch/jobs",
  "buffer_duration_ms": 3000,
  "sample_rate": 16000
}
```

### 6. requirements.txt

```
aiohttp>=3.9.0
```

### 7. __init__.py

```python
# Empty package init
```

### 8. Register the Extension in the App Manifest

Add the extension to `tenapp/manifest.json` in the dependencies list so the TEN runtime knows about it:

```json
{
  "dependencies": [
    ...existing dependencies...,
    {
      "type": "extension",
      "name": "audio_sentiment_python",
      "version": "0.1.0"
    }
  ]
}
```

### 9. Graph Configuration

Add a new graph (or modify an existing one) in `tenapp/property.json` under `ten.predefined_graphs`. This graph wires audio from Agora RTC into the sentiment analyzer, and sends sentiment data to the LLM for context-aware responses:

```json
{
  "name": "voice_with_sentiment",
  "auto_start": false,
  "graph": {
    "nodes": [
      {
        "type": "extension",
        "name": "agora_rtc",
        "addon": "agora_rtc",
        "extension_group": "default",
        "property": {
          "app_id": "${env:AGORA_APP_ID}",
          "token": "",
          "channel": "",
          "stream_id": 1234,
          "remote_stream_id": 0,
          "subscribe_audio": true,
          "publish_audio": true
        }
      },
      {
        "type": "extension",
        "name": "stt",
        "addon": "deepgram_asr_python",
        "extension_group": "default",
        "property": {
          "api_key": "${env:DEEPGRAM_API_KEY|}",
          "language": "en",
          "model": "nova-2"
        }
      },
      {
        "type": "extension",
        "name": "audio_sentiment",
        "addon": "audio_sentiment_python",
        "extension_group": "default",
        "property": {
          "api_key": "${env:SENTIMENT_API_KEY|}",
          "api_url": "https://api.hume.ai/v0/batch/jobs",
          "buffer_duration_ms": 3000,
          "sample_rate": 16000
        }
      },
      {
        "type": "extension",
        "name": "llm",
        "addon": "openai_chatgpt_python",
        "extension_group": "default",
        "property": {
          "api_key": "${env:OPENAI_API_KEY|}",
          "model": "gpt-4o",
          "prompt": "You are a helpful assistant. You will also receive sentiment analysis data about the user's voice. Use this to adapt your tone and responses. If the user sounds frustrated, be more empathetic. If they sound happy, match their energy."
        }
      },
      {
        "type": "extension",
        "name": "tts",
        "addon": "elevenlabs_tts_python",
        "extension_group": "default",
        "property": {
          "api_key": "${env:ELEVENLABS_TTS_KEY|}",
          "voice_id": "default"
        }
      }
    ],
    "connections": [
      {
        "extension": "agora_rtc",
        "audio_frame": [
          {
            "name": "pcm_frame",
            "dest": [
              {"extension": "stt"},
              {"extension": "audio_sentiment"}
            ]
          }
        ]
      },
      {
        "extension": "stt",
        "data": [
          {
            "name": "asr_result",
            "dest": [{"extension": "llm"}]
          }
        ]
      },
      {
        "extension": "audio_sentiment",
        "data": [
          {
            "name": "sentiment_result",
            "dest": [{"extension": "llm"}]
          }
        ]
      },
      {
        "extension": "llm",
        "data": [
          {
            "name": "text_data",
            "dest": [{"extension": "tts"}]
          }
        ]
      },
      {
        "extension": "tts",
        "audio_frame": [
          {
            "name": "pcm_frame",
            "dest": [{"extension": "agora_rtc"}]
          }
        ]
      }
    ]
  }
}
```

Key design points for the graph:
- **audio_frame fan-out**: The `agora_rtc` node sends `pcm_frame` to both `stt` and `audio_sentiment` -- the graph supports multiple destinations per output.
- **Parallel data paths**: STT produces `asr_result` and sentiment analyzer produces `sentiment_result`, both flowing into the LLM. The LLM receives both transcript text and sentiment context.
- **Standard pipeline**: LLM output goes to TTS, which sends synthesized audio back to `agora_rtc`.

### 10. Environment Variables

Add to `ai_agents/.env`:

```bash
SENTIMENT_API_KEY=your_sentiment_api_key_here
```

### 11. Install Dependencies

After creating the extension, install its Python dependencies inside the container:

```bash
sudo docker exec ten_agent_dev bash -c \
  "cd /app/agents/examples/voice-assistant-advanced/tenapp && \
   bash scripts/install_python_deps.sh"
```

### 12. Keep rebuild_property.py in Sync

If you modified `property.json` directly, also update `rebuild_property.py` to include the new `voice_with_sentiment` graph. Otherwise the next time `rebuild_property.py` runs, it will overwrite your changes.

### 13. Nuclear Restart

Since a new graph was added, a nuclear restart is required (frontend caches the graph list):

```bash
# Nuclear restart
sudo docker exec ten_agent_dev bash -c "pkill -9 -f 'bin/api'; pkill -9 node; pkill -9 bun"
sudo docker exec ten_agent_dev bash -c "rm -f /app/playground/.next/dev/lock"
sleep 2
sudo docker exec -d ten_agent_dev bash -c \
  "cd /app/agents/examples/voice-assistant-advanced && \
   task run > /tmp/task_run.log 2>&1"
sleep 12
curl -s http://localhost:8080/health && echo " API OK"
curl -s http://localhost:8080/graphs | jq -r '.data[].name'
```

Verify that `voice_with_sentiment` appears in the graph list.

### 14. Testing

#### Step 1: Verify the graph is registered

```bash
curl -s http://localhost:8080/graphs | jq '.data[].name'
# Should include "voice_with_sentiment"
```

#### Step 2: Start a session

```bash
curl -X POST http://localhost:8080/start \
  -H "Content-Type: application/json" \
  -d '{"graph_name": "voice_with_sentiment", "channel_name": "sentiment_test", "remote_stream_id": 123}'
```

#### Step 3: Monitor logs

```bash
# Watch for sentiment analysis output
sudo docker exec ten_agent_dev tail -f /tmp/task_run.log | grep --line-buffered "Sentiment"
```

Look for log lines like:
```
Sentiment: positive (confidence: 0.85)
```

#### Step 4: Test with the playground UI

1. Open http://localhost:3000
2. Select the `voice_with_sentiment` graph
3. Speak into your microphone
4. Observe that the agent responds with sentiment-aware language

#### Step 5: Stop the session

```bash
curl -X POST http://localhost:8080/stop \
  -H "Content-Type: application/json" \
  -d '{"channel_name": "sentiment_test"}'
```

### 15. Debugging Tips

- **Check logs**: All extension logs go to `/tmp/task_run.log`. Filter with `grep -E "(AudioSentiment|Sentiment)"`.
- **Check errors**: `sudo docker exec ten_agent_dev tail -200 /tmp/task_run.log | grep -E "(ERROR|Traceback)"`
- **Property not loading?**: Remember that property getters return tuples. If you see TypeError in logs, you may have forgotten to extract the first element.
- **No audio frames arriving?**: Verify the `audio_frame` connection in the graph has the correct `name` ("pcm_frame") and that `subscribe_audio` is `true` on the `agora_rtc` node.
- **Python changes not taking effect?**: Python extension code is loaded per new session. Just disconnect and reconnect -- no restart needed.
- **Dependencies missing after container restart?**: Python deps do not persist. Re-run `install_python_deps.sh`.

### Alternative: LLM Tool Extension Approach

If you want the sentiment analyzer to be available as a tool the LLM can call on demand (rather than continuously streaming), use `AsyncLLMToolBaseExtension`:

```python
from ten_ai_base.llm_tool import AsyncLLMToolBaseExtension
from ten_ai_base.types import LLMToolMetadata, LLMToolResult

class AudioSentimentToolExtension(AsyncLLMToolBaseExtension):
    def get_tool_metadata(self, ten_env) -> list[LLMToolMetadata]:
        return [
            LLMToolMetadata(
                name="analyze_voice_sentiment",
                description="Analyze the sentiment and emotions in the user's recent speech audio. Returns dominant emotion, sentiment polarity, and confidence score.",
                parameters=[]
            )
        ]

    async def run_tool(self, ten_env, name: str, args: dict) -> LLMToolResult:
        # Analyze the most recently buffered audio
        if self.latest_sentiment:
            return LLMToolResult(
                type="text",
                content=f"Sentiment: {self.latest_sentiment['sentiment']}, "
                        f"Confidence: {self.latest_sentiment['confidence']}, "
                        f"Emotions: {self.latest_sentiment['emotions']}"
            )
        return LLMToolResult(
            type="text",
            content="No audio sentiment data available yet."
        )
```

For the tool approach, the graph connection uses `cmd` instead of `data`:

```json
{
  "extension": "llm",
  "cmd": [
    {
      "names": ["tool_register"],
      "source": [{"extension": "audio_sentiment"}]
    }
  ]
}
```

## Assessment

**Rating: 4/5**

### What the skill covered well

1. **Extension file structure**: The skill clearly documented all six required files (addon.py, extension.py, manifest.json, property.json, requirements.txt, __init__.py) with templates for each. This made scaffolding the extension straightforward.

2. **AsyncExtension template**: The full lifecycle pattern (on_start, on_stop, on_cmd, on_data, on_audio_frame) was provided with working code. The `on_audio_frame` handler showing `audio_frame.get_buf()` for PCM data was exactly what was needed.

3. **Critical patterns**: The skill explicitly warned about three common pitfalls: (a) never use signal handlers (use on_stop for cleanup), (b) property getters return tuples, and (c) use `ten_runtime` not `ten`. Without these warnings, the extension would likely crash at runtime.

4. **Graph configuration**: The connection types were well-documented with JSON examples. The `audio_frame` fan-out pattern (one source to multiple destinations) was shown, which is essential for this use case where RTC audio feeds both STT and sentiment analysis in parallel.

5. **Operations workflow**: The testing flow (health check, list graphs, start session, monitor logs, stop session) and the nuclear restart procedure were complete and copy-paste ready. The restart decision table made it clear that adding a graph requires nuclear restart.

6. **Both base classes**: Covering both `AsyncExtension` (continuous processing) and `AsyncLLMToolBaseExtension` (on-demand tool) gave flexibility in how to architect the solution.

### What was missing or could be improved

1. **Data message API**: The skill showed how to receive data (`on_data`) and send commands, but did not provide a complete example of creating and sending `Data` objects (e.g., `Data.create("name")`, `data.set_property_string(...)`, `ten_env.send_data(data)`). I had to infer the Data creation API by analogy with `Cmd.create()`. This is a gap since data-producing extensions are a core pattern.

2. **AudioFrame metadata**: The skill showed `audio_frame.get_buf()` but did not document how to read audio metadata from the frame (sample rate, channels, format, timestamp). For a real sentiment extension, knowing the sample rate from the incoming frame (rather than hardcoding it) would be important.

3. **Async patterns in extensions**: No guidance on using `asyncio.create_task()` inside extension handlers, or whether the TEN event loop supports it. I assumed it does since the extension is `AsyncExtension`, but explicit guidance on concurrency patterns within extensions would help.

4. **No real extension examples for audio processing**: The templates are generic. A worked example of an extension that actually processes audio frames (even something simple like a volume meter) would have been much more instructive than having to build the buffering/analysis pattern from scratch.

5. **Dependency installation specifics**: The skill says to run `install_python_deps.sh` but does not explain how it discovers `requirements.txt` files from extensions. It is unclear whether the script automatically finds new extensions or whether additional registration is needed.
