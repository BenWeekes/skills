---
name: agora
description: Write code using Agora SDKs (agora.io) for real-time communication. Covers RTC (video/voice calling, live streaming), RTM (signaling, messaging, presence), and Conversational AI (voice AI agents). Use when the user wants to build real-time audio/video applications, integrate Agora SDKs (Web JS/TS, React, iOS Swift, Android Kotlin/Java, Go, Python), manage channels, tracks, tokens, use RTM for messaging/signaling, or build Conversational AI with the agent-toolkit. Triggers on mentions of Agora, agora.io, RTC, RTM, video calling, voice calling, real-time communication, agora-rtc-sdk-ng, agora-rtc-react, agora-rtm, conversational AI with Agora, or Agora token generation.
---

# Agora (agora.io)

Build real-time communication applications using Agora SDKs across Web, iOS, Android, and server-side platforms. Covers RTC (audio/video), RTM (messaging/signaling), and Conversational AI.

## Core Concepts

- **App ID**: Project identifier from [Agora Console](https://console.agora.io). Required for all SDK calls.
- **Token**: Time-limited auth key generated server-side from App ID + App Certificate. Never expose App Certificate on client. For testing, disable token authentication in Agora Console and pass `null` as the token.
- **Channel**: Auto-created when first user joins, destroyed when last leaves. Users in same channel can communicate.
- **Channel Profile**: `rtc` (communication, all peers equal) or `live` (host/audience roles, higher bitrate).
- **UID**: Unique user identifier per channel. Pass `null`/`0` for auto-assignment. Duplicate UIDs cause undefined behavior.
- **Tracks** (Web): Audio and video are independent track objects, created, published, and subscribed separately.
- **Publish/Subscribe**: Publish sends local tracks to channel; subscribe receives remote user tracks.

## Reference Files

Load the reference file matching the user's product and platform. Only load what is needed.

### RTC (Video/Voice SDK)

- **Web (JS/TS)**: [references/rtc-web.md](references/rtc-web.md) — `agora-rtc-sdk-ng` patterns, track management, event handling
- **Web (React)**: [references/rtc-react.md](references/rtc-react.md) — `agora-rtc-react` hooks, custom hooks, component patterns
- **iOS (Swift)**: [references/rtc-ios.md](references/rtc-ios.md) — `AgoraRtcEngineKit` integration
- **Android (Kotlin/Java)**: [references/rtc-android.md](references/rtc-android.md) — `RtcEngine` integration

### RTM (Signaling / Messaging)

- **Web (JS/TS)**: [references/rtm-web.md](references/rtm-web.md) — signaling, chat, presence, v2 API

### Conversational AI (Voice AI Agents)

- **Web (JS/TS + React)**: [references/conversational-ai-web.md](references/conversational-ai-web.md) — agent-toolkit SDK, transcript handling

### Server-Side

- **Token Generation (Node/Python/Go)**: [references/server-tokens.md](references/server-tokens.md) — RTC/RTM token generation, Token 007

## Critical Rules (All Platforms)

1. **Register event handlers BEFORE joining** the channel, or miss events for users already present.
2. **Token management is mandatory in production**. Handle `token-privilege-will-expire` (Web) / `onTokenPrivilegeWillExpire` (native) to renew tokens. UID in token must match UID used to join.
3. **HTTPS required** for Web SDK (except `localhost`).
4. **Audio autoplay**: Browsers block audio autoplay. Require user interaction (click/tap) before playing remote audio.
5. **Track cleanup**: Always `stop()` then `close()` local tracks before setting to null. Failure to clean up causes memory leaks and device locks.
6. **`user-published` fires separately** for audio and video. A user publishing both triggers two events.
7. **Never expose App Certificate on client**. Token generation must happen server-side.

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
- Requires Node.js >= 18 for Next.js 14+ and the Agora SDK.

## Common Patterns

### Video Encoder Profiles (Web)

```
"120p_1"  → 160×120,   15fps, 65kbps
"180p_1"  → 320×180,   15fps, 140kbps
"360p_1"  → 640×360,   15fps, 400kbps
"480p_1"  → 640×480,   15fps, 500kbps
"720p_1"  → 1280×720,  15fps, 1130kbps
"720p_2"  → 1280×720,  30fps, 2000kbps
"1080p_1" → 1920×1080, 15fps, 2080kbps
"1080p_2" → 1920×1080, 30fps, 3000kbps
```

Or use custom config: `{ width: 640, height: 360, frameRate: 24, bitrateMin: 400, bitrateMax: 1000 }`

### Audio Encoder Profiles (Web)

```
"speech_low_quality"      → mono, 16kHz, 24kbps
"speech_standard"         → mono, 32kHz, 24kbps
"high_quality"            → mono, 48kHz, 40kbps
"high_quality_stereo"     → stereo, 48kHz, 128kbps
```

### Dual Stream (Simulcast)

Enable to send high+low quality streams simultaneously. Subscribers choose which to receive, enabling large-scale calls:

```javascript
// Web: Enable after joining
await client.enableDualStream();
client.setLowStreamParameter({ width: 160, height: 90, framerate: 24, bitrate: 200 });
// Switch remote user to low stream
client.setRemoteVideoStreamType(uid, 1); // 0=high, 1=low
```

### Screen Sharing (Web)

```javascript
const screenTrack = await AgoraRTC.createScreenVideoTrack({
  optimizationMode: "detail", // or "motion" for video content
  encoderConfig: { width: 1280, height: 720, frameRate: 15 }
}, "auto"); // "auto" returns [videoTrack, audioTrack] if audio available
```

Screen share typically uses a separate client instance to avoid replacing the camera track.

## Official Documentation (Fallback)

If the reference files do not cover what is needed, fetch from these official Agora docs:

### RTC (Video/Voice SDK)
- Web API Reference: https://api-ref.agora.io/en/video-sdk/web/4.x/index.html
- Android API Reference: https://api-ref.agora.io/en/video-sdk/android/4.x/API/rtc_api_overview.html
- iOS API Reference: https://api-ref.agora.io/en/video-sdk/ios/4.x/API/rtc_api_overview_ng.html
- Flutter API Reference: https://api-ref.agora.io/en/video-sdk/flutter/6.x/API/rtc_api_overview.html
- Guides: https://docs.agora.io/en/video-calling/overview/product-overview

### RTM (Signaling)
- Web API Reference: https://api-ref.agora.io/en/signaling-sdk/web/2.x/index.html
- Guides: https://docs.agora.io/en/signaling/overview/product-overview

### Conversational AI
- Overview: https://docs.agora.io/en/conversational-ai/overview/product-overview
- Agent Toolkit: https://docs.agora.io/en/conversational-ai/develop/agent-toolkit

### Tokens
- Authentication Guide: https://docs.agora.io/en/video-calling/get-started/authentication-workflow
