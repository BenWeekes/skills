---
name: agora
description: Write code using Agora SDKs (agora.io) for real-time communication. Covers RTC (video/voice calling, live streaming), RTM (signaling, messaging, presence), and Conversational AI (voice AI agents). Use when the user wants to build real-time audio/video applications, integrate Agora SDKs (Web JS/TS, React, iOS Swift, Android Kotlin/Java, Go, Python), manage channels, tracks, tokens, use RTM for messaging/signaling, or build Conversational AI with the agent-toolkit. Triggers on mentions of Agora, agora.io, RTC, RTM, video calling, voice calling, real-time communication, agora-rtc-sdk-ng, agora-rtc-react, agora-rtm, conversational AI with Agora, or Agora token generation.
---

# Agora (agora.io)

Build real-time communication applications using Agora SDKs across Web, iOS, Android, and server-side platforms. Covers RTC (audio/video), RTM (messaging/signaling), and Conversational AI.

## Core Concepts

- **App ID**: Project identifier from [Agora Console](https://console.agora.io). Required for all SDK calls.
- **Token**: Time-limited auth key generated server-side from App ID + App Certificate. Never expose App Certificate on client.
- **Channel**: Auto-created when first user joins, destroyed when last leaves. Users in same channel can communicate.
- **Channel Profile**: `rtc` (communication, all peers equal) or `live` (host/audience roles, higher bitrate).
- **UID**: Unique user identifier per channel. Pass `null`/`0` for auto-assignment. Duplicate UIDs cause undefined behavior.
- **Tracks** (Web): Audio and video are independent track objects, created, published, and subscribed separately.
- **Publish/Subscribe**: Publish sends local tracks to channel; subscribe receives remote user tracks.

## Platform Reference Files

Select the platform-specific guide based on the user's target:

- **Web (JS/TS)**: See [references/web.md](references/web.md) for `agora-rtc-sdk-ng` patterns, track management, event handling
- **Web (React)**: See [references/react.md](references/react.md) for `agora-rtc-react` hooks and component patterns
- **iOS (Swift)**: See [references/ios.md](references/ios.md) for `AgoraRtcEngineKit` integration
- **Android (Kotlin/Java)**: See [references/android.md](references/android.md) for `RtcEngine` integration
- **Server-side**: See [references/server.md](references/server.md) for token generation (Node/Go/Python)
- **RTM (Real-Time Messaging)**: See [references/rtm.md](references/rtm.md) for signaling, chat, presence alongside RTC
- **Conversational AI**: See [references/conversational-ai.md](references/conversational-ai.md) for the agent-toolkit SDK

## Critical Rules (All Platforms)

1. **Register event handlers BEFORE joining** the channel, or miss events for users already present.
2. **Token management is mandatory in production**. Handle `token-privilege-will-expire` (Web) / `onTokenPrivilegeWillExpire` (native) to renew tokens. UID in token must match UID used to join.
3. **HTTPS required** for Web SDK (except `localhost`).
4. **Audio autoplay**: Browsers block audio autoplay. Require user interaction (click/tap) before playing remote audio.
5. **Track cleanup**: Always `stop()` then `close()` local tracks before setting to null. Failure to clean up causes memory leaks and device locks.
6. **`user-published` fires separately** for audio and video. A user publishing both triggers two events.
7. **Never expose App Certificate on client**. Token generation must happen server-side.

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

## API Documentation Links

- Web: https://api-ref.agora.io/en/video-sdk/web/4.x/index.html
- Android: https://api-ref.agora.io/en/video-sdk/android/4.x/API/rtc_api_overview.html
- iOS: https://api-ref.agora.io/en/video-sdk/ios/4.x/API/rtc_api_overview_ng.html
