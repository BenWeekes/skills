---
name: agora
description: Write code using Agora SDKs (agora.io) for real-time communication. Covers RTC (video/voice calling, live streaming), RTM (signaling, messaging, presence), Conversational AI (voice AI agents), and TEN Framework (ten_runtime, ten_ai_base, AsyncExtension, ten_packages, voice-assistant-advanced). Use when the user wants to build real-time audio/video applications, integrate Agora SDKs (Web JS/TS, React, iOS Swift, Android Kotlin/Java, Go, Python), manage channels, tracks, tokens, use RTM for messaging/signaling, build Conversational AI with the agent-toolkit, or build TEN Framework AI agents. Triggers on mentions of Agora, agora.io, RTC, RTM, video calling, voice calling, real-time communication, agora-rtc-sdk-ng, agora-rtc-react, agora-rtm, conversational AI with Agora, Agora token generation, TEN Framework, ten_runtime, ten_ai_base, AsyncExtension, ten_packages, voice-assistant-advanced.
---

# Agora (agora.io)

**Skill version: 1.1.0**

Build real-time communication applications using Agora SDKs across Web, iOS, Android, and server-side platforms.

## Core Concepts

- **App ID**: Project identifier from [Agora Console](https://console.agora.io). Required for all SDK calls.
- **Token**: Time-limited auth key generated server-side from App ID + App Certificate. Never expose App Certificate on client. For testing, disable token authentication in Agora Console and pass `null` as the token.
- **Channel**: Auto-created when first user joins, destroyed when last leaves. Users in same channel can communicate.
- **UID**: Unique user identifier per channel. Pass `null`/`0` for auto-assignment. Duplicate UIDs cause undefined behavior.

## Products

Read the README for the product the user needs. Only load what is needed.

### RTC (Video/Voice SDK)

Real-time audio and video. Users join channels, publish local tracks, subscribe to remote tracks.

**[references/rtc/README.md](references/rtc/README.md)** — Platforms: Web, React, iOS, Android

### RTM (Signaling / Messaging)

Text messaging, signaling, presence, and metadata. Independent from RTC — channel namespaces are separate.

**[references/rtm/README.md](references/rtm/README.md)** — Platforms: Web

### Conversational AI (Voice AI Agents)

REST API-driven voice AI agents. Create agents that join RTC channels and converse with users via speech. Front-end clients connect via RTC+RTM.

**[references/conversational-ai/README.md](references/conversational-ai/README.md)** — REST API, agent config, 5 recipe repos (agent-samples, agent-toolkit, agent-ui-kit, server-custom-llm, server-mcp)

### Server-Side

Token generation and server utilities. Required for production authentication.

**[references/server/README.md](references/server/README.md)** — Token generation for Node.js, Python, Go

### TEN Framework

Open-source graph-based AI agent framework (v0.11+). Modular extensions connected through data flows for voice/video AI pipelines.

**[references/ten-framework/README.md](references/ten-framework/README.md)** — Extensions, graphs, operations

## Web Framework Notes

### Next.js / SSR Frameworks

The Agora Web SDK (`agora-rtc-sdk-ng`) is browser-only and cannot run during server-side rendering. When using Next.js or any SSR framework:

- Mark components using Agora as `"use client"` in Next.js App Router.
- Lazy-load the Agora component to avoid SSR — use dynamic import in a client component:
  ```tsx
  "use client";
  import { useState, useEffect } from "react";
  export default function Page() {
    const [VideoCall, setVideoCall] = useState<React.ComponentType | null>(null);
    useEffect(() => { import("./VideoCall").then(m => setVideoCall(() => m.default)); }, []);
    if (!VideoCall) return <div>Loading...</div>;
    return <VideoCall />;
  }
  ```
- Note: `next/dynamic` with `ssr: false` does NOT work in Server Components in Next.js 14+. Use the client component pattern above instead.
- Requires Node.js >= 20.9.0 for Next.js 16+ (used by agent-samples clients). Next.js 14/15 require Node.js >= 18.
