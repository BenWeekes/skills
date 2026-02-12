# Agora RTM (Real-Time Messaging / Signaling)

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
- [RTM v2 (Web)](#rtm-v2-web)
- [RTM v1 (Legacy Web)](#rtm-v1-legacy-web)
- [Common Use Cases with RTC](#common-use-cases-with-rtc)

## Overview

RTM provides signaling, text messaging, presence, and metadata capabilities alongside RTC audio/video. RTC and RTM are **independent systems** — RTC channels and RTM channels are separate namespaces.

**When to use RTM alongside RTC:**
- Text chat during video calls
- Signaling (call invitations, control messages)
- User presence/status tracking
- Custom data exchange (VAD signals, resolution requests)
- Sending text messages to AI agents (Conversational AI)

Package: `agora-rtm` (v2.x) on npm.

## Installation

```bash
npm install agora-rtm
```

## RTM v2 (Web)

### Initialization and Login

```typescript
import AgoraRTM from "agora-rtm"

const rtmClient = new AgoraRTM.RTM("your-app-id", "user-id-string", {
  logLevel: "debug", // "debug" | "info" | "warn" | "error"
})

// Login (uses App ID for authentication, or token for production)
await rtmClient.login()
// With token: await rtmClient.login({ token: "your-rtm-token" })
```

### Channel Subscription

```typescript
// Subscribe to a channel to receive messages and presence
await rtmClient.subscribe("channel-name", {
  withMessage: true,
  withPresence: true,
})

// Unsubscribe
await rtmClient.unsubscribe("channel-name")
```

### Sending Messages

```typescript
// Publish to channel (all subscribers receive)
await rtmClient.publish("channel-name", "Hello everyone!", {
  customType: "chat.message",
})

// Publish JSON data
await rtmClient.publish("channel-name", JSON.stringify({
  type: "control",
  action: "mute-all",
}), {
  customType: "control.message",
})

// Peer-to-peer (publish to user topic)
await rtmClient.publish("target-user-id", JSON.stringify({
  message: "Hello!",
  priority: "APPEND",
}), {
  customType: "user.transcription",
  channelType: "USER",
})
```

### Receiving Messages

```typescript
rtmClient.addEventListener("message", (event) => {
  console.log("Message from:", event.publisher)
  console.log("Channel:", event.channelName)
  console.log("Content:", event.message)      // string or Uint8Array
  console.log("Type:", event.customType)

  // Parse if JSON
  if (typeof event.message === "string") {
    try {
      const data = JSON.parse(event.message)
    } catch {}
  }
})
```

### Presence Events

```typescript
rtmClient.addEventListener("presence", (event) => {
  // event.eventType: "SNAPSHOT" | "INTERVAL" | "JOIN" | "LEAVE" | "TIMEOUT" | "STATE_CHANGED"
  console.log("Presence:", event.eventType, event)
})
```

### Connection Status

```typescript
rtmClient.addEventListener("status", (event) => {
  // event.state: "CONNECTED" | "CONNECTING" | "RECONNECTING" | "DISCONNECTED"
  console.log("RTM status:", event.state)
})
```

### Cleanup

```typescript
async function cleanupRTM() {
  await rtmClient.unsubscribe("channel-name")
  await rtmClient.logout()
}
```

## RTM v1 (Legacy Web)

Used in the multigrid project. Still functional but v2 is recommended for new projects.

```typescript
import AgoraRTM from "agora-rtm-sdk"

// Initialize
const rtmClient = AgoraRTM.createInstance("your-app-id", {
  logFilter: AgoraRTM.LOG_FILTER_OFF,
})

// Login
await rtmClient.login({ token: null, uid: "user-id" })

// Create and join channel
const rtmChannel = rtmClient.createChannel("channel-name")
await rtmChannel.join()

// Listen for channel messages
rtmChannel.on("ChannelMessage", ({ text }, senderId) => {
  console.log(`${senderId}: ${text}`)
})

// Listen for peer messages
rtmClient.on("MessageFromPeer", ({ text }, senderId) => {
  console.log(`Peer ${senderId}: ${text}`)
})

// Send channel message
await rtmChannel.sendMessage({ text: "Hello" })

// Send peer message
await rtmClient.sendMessageToPeer({ text: "Hello" }, "target-user-id")

// Cleanup
await rtmChannel.leave()
await rtmClient.logout()
```

## Common Use Cases with RTC

### Text Chat During Video Call

```typescript
// RTM for chat alongside RTC video
rtmClient.addEventListener("message", (event) => {
  if (event.customType === "chat.message") {
    displayChatMessage(event.publisher, event.message)
  }
})

function sendChatMessage(text: string) {
  rtmClient.publish(channelName, text, { customType: "chat.message" })
}
```

### Voice Activity Detection (VAD) Signaling

```typescript
// Notify other users when speaking (for prioritized video)
function notifyVAD(uid: number) {
  rtmClient.publish(channelName, JSON.stringify({
    type: "VAD",
    uid: uid,
  }), { customType: "signaling.vad" })
}
```

### Control Messages

```typescript
// Request resolution change, stop screen share, etc.
rtmClient.publish(channelName, JSON.stringify({
  type: "INCREASE_RESOLUTION",
  targetUid: uid,
}), { customType: "signaling.control" })
```

### Sending Text to Conversational AI Agent

```typescript
// Send a text message to an AI agent via RTM
async function sendMessageToAgent(message: string, agentUid: string) {
  const publishTarget = `${agentUid}-${channel}`

  await rtmClient.publish(publishTarget, JSON.stringify({
    message: message.trim(),
    priority: "APPEND", // or "REPLACE"
  }), {
    customType: "user.transcription",
    channelType: "USER",
  })
}
```

## Official Documentation

For APIs or features not covered above:
- API Reference: https://api-ref.agora.io/en/signaling-sdk/web/2.x/index.html
- Guides: https://docs.agora.io/en/signaling/overview/product-overview
