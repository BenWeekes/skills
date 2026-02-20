---
name: ten
description: Build and operate TEN Framework AI agents. Covers extension development (AsyncExtension, AsyncLLMToolBaseExtension), graph configuration (property.json, connections), Docker operations, and debugging. Use when the user wants to create TEN extensions, configure graphs, manage the ten_runtime environment, or troubleshoot the voice-assistant-advanced example. Triggers on mentions of TEN Framework, ten_runtime, ten_ai_base, AsyncExtension, extensions, graphs, voice-assistant-advanced, Taskfile, ten_packages, TEN agent, graph-based AI agent.
---

# TEN Framework

**Skill version: 1.0.0**

Build modular AI agents using the TEN (Transformative Extensions Network) graph-based framework (v0.11+). Extensions connect through defined data flows to form agent pipelines.

## Key Concepts

- **Extensions**: Modular components that process data — speech-to-text, LLM, TTS, custom analyzers. Built on `AsyncExtension` or `AsyncLLMToolBaseExtension` from `ten_runtime` / `ten_ai_base`.
- **Graphs**: Configurations in `property.json` defining which extensions run and how they connect. Each graph is a named pipeline.
- **Connections**: Data flows between extensions — `cmd` (commands), `data` (messages), `audio_frame` (PCM audio), `video_frame` (video streams).
- **Property Files**: JSON configs with env var substitution — `${env:VAR_NAME}` (required, error if missing) or `${env:VAR_NAME|}` (optional, empty string if missing).

## Repository Structure

```
ai_agents/
├── .env                          # ONLY env file used
├── agents/examples/
│   └── voice-assistant-advanced/
│       ├── Taskfile.yaml         # Build/run automation
│       └── tenapp/
│           ├── property.json     # Graph definitions
│           ├── rebuild_property.py  # Generates property.json
│           └── manifest.json     # Extension dependencies
├── agents/ten_packages/extension/ # Extension source code
├── playground/                   # Next.js frontend UI
└── server/                       # Go API server
```

## Critical Rules

1. **Always use `sudo` with Docker commands** — `sudo docker exec ten_agent_dev ...`
2. **Nuclear restart after adding/removing graphs** — frontend caches the graph list
3. **Always use `task run`** — never `./bin/api` or `./bin/main` directly (sets PYTHONPATH)
4. **Python deps don't persist** after container restart — reinstall with `install_python_deps.sh`
5. **Keep `property.json` and `rebuild_property.py` in sync** — update rebuild script first, run it, then restart

## When to Restart What

| Changed | Container Restart? | Nuclear Restart? | Notes |
|---------|-------------------|------------------|-------|
| **property.json** (add/remove graphs) | No | **YES** | Frontend caches graph list |
| **property.json** (config only, prompts, params) | No | **No** | Loaded per NEW session — just reconnect |
| **rebuild_property.py** (then run script) | No | Only if graphs added/removed | Run script first to regenerate property.json |
| **Python extension code** (.py files) | No | **No** | Loaded per NEW session — just reconnect |
| **.env file** | **YES** | Then yes | Must restart container first |

**NO RESTART NEEDED**: Editing prompts, params in property.json, or Python extension code. Changes apply to new sessions automatically.

## Topics

Read the reference for what the user needs. Only load what is required.

### Extension Development

Creating and modifying TEN extensions — file structure, templates, critical patterns, base classes.

**[references/extensions.md](references/extensions.md)**

### Graph Configuration

Defining graphs, connections, property injection, API endpoints, and testing workflows.

**[references/graphs.md](references/graphs.md)**

### Operations

Environment setup, Docker, startup, nuclear restart, logs, remote access, troubleshooting.

**[references/operations.md](references/operations.md)**
