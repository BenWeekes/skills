---
name: agora-recipe-thymia
description: |
  Recipe for integrating Thymia voice biomarker analysis with Agora Conversational AI.
  Use when the user asks about voice biomarkers, wellness analysis, Thymia profiles,
  or ThymiaPanel components alongside ConvoAI.
license: MIT
metadata:
  author: agora
  version: "1.0.0"
---

# Recipe: Thymia Voice Biomarker Integration

Thymia provides voice biomarker analysis (emotion, wellness, safety) via a custom LLM server
module integrated with Agora Conversational AI.

## Architecture

```text
User voice → ConvoAI → server-custom-llm (Node.js, Thymia module)
                              ↓
                     Thymia biomarker analysis
                              ↓
                    ThymiaPanel (agent-ui-kit) ← RTM data channel
```

## Prerequisites

- Agora ConvoAI set up with `server-custom-llm` (Node.js implementation)
- `agent-samples` with React clients
- Thymia API credentials

## Setup

### Backend (agent-samples)

Enable Thymia profiles in `.env`:

```bash
NEXT_PUBLIC_ENABLE_THYMIA=true
NEXT_PUBLIC_DEFAULT_PROFILE=THYMIA       # or THYMIA_VIDEO for video+avatar
```

Thymia profiles (`THYMIA`, `THYMIA_VIDEO`) are built into agent-samples — no separate
client directories needed.

### Custom LLM Server (Node.js)

Thymia analysis runs as a pluggable module in `server-custom-llm`:

```bash
# In server-custom-llm/node/
# Configure Thymia module credentials in .env
THYMIA_API_KEY=...
```

> **[recipes/thymia.md](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/recipes/thymia.md)** — Full setup guide including module config and env vars

### Frontend (agent-ui-kit)

`ThymiaPanel` displays biomarker results. Mount alongside the voice UI:

```tsx
import { ThymiaPanel, useThymia } from '@agora/agent-ui-kit';

const { biomarkers, wellness, clinical, safety, progress } = useThymia();

<ThymiaPanel />
```

- `useThymia` — returns biomarker data: `biomarkers`, `wellness`, `clinical`, `safety`, `progress`
- `ThymiaPanel` — pre-built display component, receives data via RTM channel from the custom LLM module

> **[README — Thymia Components](https://github.com/AgoraIO-Conversational-AI/agent-ui-kit#thymia-components)**

## URL Params

- `?profile=THYMIA` — override profile selection at runtime
- `?autoconnect=true` — auto-start conversation on page load

## Reference

- [agent-samples recipes/thymia.md](https://github.com/AgoraIO-Conversational-AI/agent-samples/blob/main/recipes/thymia.md)
- [server-custom-llm Node.js integrations](https://github.com/AgoraIO-Conversational-AI/server-custom-llm/blob/main/node/integrations/README.md)
- [agent-ui-kit Thymia components](https://github.com/AgoraIO-Conversational-AI/agent-ui-kit#thymia-components)
