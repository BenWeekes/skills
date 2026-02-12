# Agora Conversational AI (agent-toolkit)

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [RTCHelper](#rtchelper)
- [RTMHelper](#rtmhelper)
- [ConversationalAIAPI](#conversationalaiapi)
- [Transcript Handling](#transcript-handling)
- [React Hooks](#react-hooks)
- [Full React Example](#full-react-example)

## Overview

The `@agora/conversational-ai` agent-toolkit is a lightweight TypeScript SDK for building voice AI applications using Agora RTC and RTM. It provides:

- **RTCHelper**: Singleton wrapper for Agora RTC client lifecycle
- **RTMHelper**: Simplified RTM messaging with AI agents
- **ConversationalAIAPI**: Orchestrates RTC + RTM with transcript rendering
- **SubRenderController**: Message assembly, deduplication, PTS-synced rendering
- **React hooks**: `useLocalVideo`, `useRemoteVideo`

~30KB raw, ~8KB gzipped. Zero runtime dependencies (peer deps: `agora-rtc-sdk-ng`, `agora-rtm`).

## Installation

```bash
npm install @agora/conversational-ai
# Peer dependencies (installed automatically if using npm 7+):
# agora-rtc-sdk-ng ^4.23.4
# agora-rtm ^2.0.0
```

## Architecture

```
ConversationalAIAPI (orchestrator)
├── RTCHelper (audio/video, stream-messages from agent)
├── RTMHelper (text messaging to/from agent)
└── SubRenderController (transcript assembly & rendering)
```

- **Primary transport**: RTC stream-messages for AI agent transcripts (low-latency)
- **Secondary transport**: RTM for text input to agent and reliable messaging
- Transcripts arrive as chunked, base64-encoded messages via RTC data channel
- SubRenderController handles assembly, deduplication, and PTS-synchronized word-level rendering

## Quick Start

```typescript
import { ConversationalAIAPI, RTCHelper } from "@agora/conversational-ai"

// 1. Initialize RTC
const rtcHelper = RTCHelper.getInstance()
await rtcHelper.init({
  appId: "your-app-id",
  channel: "test-channel",
  token: "your-rtc-token",  // or null for testing
  uid: 12345,
})

// 2. Create audio track and join
await rtcHelper.createAudioTrack({
  encoderConfig: "high_quality_stereo",
  AEC: true, ANS: true, AGC: true,
})
await rtcHelper.join()
await rtcHelper.publish()

// 3. Initialize ConversationalAIAPI with RTM
const api = ConversationalAIAPI.init({
  rtcEngine: rtcHelper.client!,
  rtmConfig: {
    appId: "your-app-id",
    uid: "12345",         // string for RTM
    token: "rtm-token",   // or null
    channel: "test-channel",
  },
  renderMode: "auto",     // "text" | "word" | "chunk" | "auto"
  enableLog: true,
})

// 4. Listen for transcripts
api.on("transcript-updated", (messages) => {
  messages.forEach(msg => {
    console.log(`${msg.uid === 0 ? "Agent" : "User"}: ${msg.text}`)
  })
})

// 5. Send text message to agent
await api.sendMessage("Hello, agent!", "100")

// 6. Cleanup
api.destroy()
```

## RTCHelper

Singleton wrapper around `agora-rtc-sdk-ng` client.

```typescript
import { RTCHelper } from "@agora/conversational-ai"

const rtcHelper = RTCHelper.getInstance()

// Initialize
await rtcHelper.init({
  appId: "app-id",
  channel: "channel",
  token: null,
  uid: 12345,
  shouldSubscribeAudio: (uid) => uid !== 999,  // optional filter
  shouldSubscribeVideo: (uid) => true,          // optional filter
})

// Create tracks
await rtcHelper.createAudioTrack({
  encoderConfig: "high_quality_stereo",
  AEC: true, ANS: true, AGC: true,
})
await rtcHelper.createVideoTrack({
  encoderConfig: "720p_2",
  cameraId: "optional-device-id",
})

// Lifecycle
await rtcHelper.join()
await rtcHelper.publish()
await rtcHelper.unpublish()
await rtcHelper.leave()

// Mute
await rtcHelper.setMuted(true)
const isMuted = rtcHelper.getMuted()

// Video
await rtcHelper.setVideoEnabled(false)
const isVideoOn = rtcHelper.getVideoEnabled()

// Remote users
const remoteUsers = rtcHelper.getRemoteUsers()

// Events
rtcHelper.on("connection-state-changed", (state) => { /* ... */ })
rtcHelper.on("user-published", (user, mediaType) => { /* ... */ })
rtcHelper.on("user-left", (user) => { /* ... */ })
rtcHelper.on("volume-indicator", (volumes) => { /* ... */ })
rtcHelper.on("network-quality", (quality) => { /* ... */ })
rtcHelper.on("stream-message", (uid, data) => { /* ... */ })
rtcHelper.on("error", (error) => { /* ... */ })

// Cleanup
rtcHelper.destroy()
```

## RTMHelper

Simplified RTM client for messaging with AI agents.

```typescript
import { RTMHelper } from "@agora/conversational-ai"

const rtmHelper = RTMHelper.getInstance()

await rtmHelper.init({
  appId: "app-id",
  uid: "12345",      // string UID for RTM
  token: null,
})

await rtmHelper.login()
await rtmHelper.subscribe("channel-name")

// Send message to AI agent
await rtmHelper.sendMessage(
  "Hello agent",
  "100",            // agent UID
  "APPEND"          // or "REPLACE"
)

// Events
rtmHelper.on("message", (event) => { /* ... */ })
rtmHelper.on("presence", (event) => { /* ... */ })
rtmHelper.on("status", (event) => { /* ... */ })

// Cleanup
rtmHelper.destroy()
```

## ConversationalAIAPI

Orchestrates RTCHelper + RTMHelper + SubRenderController.

```typescript
import { ConversationalAIAPI } from "@agora/conversational-ai"

const api = ConversationalAIAPI.init({
  rtcEngine: rtcHelper.client!,  // Required: initialized IAgoraRTCClient
  rtmConfig: {                    // Optional: enables text messaging
    appId: "app-id",
    uid: "12345",
    token: null,
    channel: "channel",
  },
  renderMode: "auto",            // Transcript render mode
  enableLog: false,
})

// Events
api.on("transcript-updated", (messages: TranscriptItem[]) => {
  // messages: [{ uid, text, turn_id, status, timestamp }]
})
api.on("connection-state-changed", (state) => { /* ... */ })
api.on("agent-error", (error) => { /* ... */ })

// Send text to agent
await api.sendMessage("Hello", "100", "APPEND")

// Get current transcript
const transcript = api.getTranscript()
api.clearTranscript()

// Cleanup
api.destroy()
```

## Transcript Handling

AI agent transcripts arrive via RTC stream-messages in chunked format:

```
message_id|part_idx|part_sum|base64_data
```

The SubRenderController handles:
1. **Chunked assembly**: Collects parts, assembles when complete, base64 decodes
2. **Deduplication**: By message_id and turn_id+uid
3. **PTS synchronization**: Word-level rendering synced to audio playback timing
4. **Render modes**:
   - `text`: Full text updates per turn
   - `word`: Individual word rendering with PTS sync
   - `chunk`: Partial text chunks
   - `auto`: Automatically selects based on message format

Message types routed by `object` field:
- `user.transcription` → user speech-to-text
- `assistant.transcription` → agent response text
- `message.interrupt` → interruption signal

## React Hooks

### useLocalVideo

```tsx
import { useLocalVideo } from "@agora/conversational-ai"

function VideoControls() {
  const {
    videoTrack,       // ICameraVideoTrack | null
    isVideoEnabled,   // boolean
    cameras,          // VideoDevice[]
    enableVideo,      // () => Promise<void>
    disableVideo,     // () => Promise<void>
    toggleVideo,      // () => Promise<void>
    switchCamera,     // (deviceId: string) => Promise<void>
    refreshCameras,   // () => Promise<void>
    error,            // string | null
  } = useLocalVideo({
    deviceId: "default",
    encoderConfig: "720p_2",
    startEnabled: false,
  })

  return (
    <div>
      <button onClick={toggleVideo}>{isVideoEnabled ? "Stop" : "Start"} Video</button>
      {videoTrack && <VideoPlayer track={videoTrack} />}
    </div>
  )
}
```

### useRemoteVideo

```tsx
import { useRemoteVideo } from "@agora/conversational-ai"

function RemoteVideos({ client }) {
  const {
    remoteVideoUsers,      // RemoteVideoUser[]
    getVideoTrack,         // (uid) => IRemoteVideoTrack | null
    hasVideo,              // (uid) => boolean
    count,                 // number
  } = useRemoteVideo({
    client,
    autoSubscribe: true,
    userFilter: (uid) => uid !== 999,
  })

  return (
    <div>
      {remoteVideoUsers.map(user => (
        <VideoPlayer key={user.uid} track={user.videoTrack} />
      ))}
    </div>
  )
}
```

## Full React Example

```tsx
import { useEffect, useRef, useState } from "react"
import { ConversationalAIAPI, RTCHelper } from "@agora/conversational-ai"
import type { TranscriptItem, ConnectionState } from "@agora/conversational-ai"

function VoiceAgent({ appId, channel, token, uid }: Props) {
  const [transcript, setTranscript] = useState<TranscriptItem[]>([])
  const [connectionState, setConnectionState] = useState<ConnectionState>("disconnected")
  const [isMuted, setIsMuted] = useState(false)
  const apiRef = useRef<ConversationalAIAPI | null>(null)
  const rtcRef = useRef<RTCHelper | null>(null)

  const connect = async () => {
    // Initialize RTC
    const rtcHelper = RTCHelper.getInstance()
    rtcRef.current = rtcHelper

    await rtcHelper.init({ appId, channel, token, uid })
    await rtcHelper.createAudioTrack({
      encoderConfig: "high_quality_stereo",
      AEC: true, ANS: true, AGC: true,
    })
    await rtcHelper.join()
    await rtcHelper.publish()

    // Initialize API
    const api = ConversationalAIAPI.init({
      rtcEngine: rtcHelper.client!,
      rtmConfig: { appId, uid: String(uid), token: null, channel },
      renderMode: "auto",
    })
    apiRef.current = api

    api.on("transcript-updated", setTranscript)
    api.on("connection-state-changed", setConnectionState)
  }

  const disconnect = () => {
    apiRef.current?.destroy()
    rtcRef.current?.destroy()
    apiRef.current = null
    rtcRef.current = null
    setConnectionState("disconnected")
  }

  const toggleMute = async () => {
    await rtcRef.current?.setMuted(!isMuted)
    setIsMuted(!isMuted)
  }

  const sendText = async (text: string) => {
    await apiRef.current?.sendMessage(text, "100")
  }

  useEffect(() => () => disconnect(), [])

  return (
    <div>
      <p>Status: {connectionState}</p>
      <button onClick={connectionState === "disconnected" ? connect : disconnect}>
        {connectionState === "disconnected" ? "Connect" : "Disconnect"}
      </button>
      <button onClick={toggleMute}>{isMuted ? "Unmute" : "Mute"}</button>

      <div>
        {transcript.map((msg, i) => (
          <p key={i}><b>{msg.uid === 0 ? "Agent" : "You"}:</b> {msg.text}</p>
        ))}
      </div>
    </div>
  )
}
```

## Official Documentation

For APIs or features not covered above:
- Overview: https://docs.agora.io/en/conversational-ai/overview/product-overview
- Agent Toolkit: https://docs.agora.io/en/conversational-ai/develop/agent-toolkit
- RTC Web SDK (used by agent-toolkit): https://api-ref.agora.io/en/video-sdk/web/4.x/index.html
