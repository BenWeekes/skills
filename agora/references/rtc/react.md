# Agora RTC React Integration

## Table of Contents
- [agora-rtc-react Package](#agora-rtc-react-package)
- [Basic Setup](#basic-setup)
- [Custom Hooks Pattern (agent-toolkit)](#custom-hooks-pattern-agent-toolkit)
- [useLocalVideo Hook](#uselocalvideo-hook)
- [useRemoteVideo Hook](#useremotevideo-hook)
- [Full React App Example](#full-react-app-example)

## agora-rtc-react Package

```bash
npm install agora-rtc-react
# v2.0.0+ bundles agora-rtc-sdk-ng internally — no separate install needed
```

The `agora-rtc-react` package provides React components and hooks that wrap the Agora Web SDK. For more control, use the raw SDK with custom hooks (see agent-toolkit pattern below).

## Basic Setup

```tsx
import AgoraRTC, { AgoraRTCProvider, useJoin, useLocalMicrophoneTrack, useLocalCameraTrack, usePublish, useRemoteUsers, RemoteUser } from "agora-rtc-react"

const client = AgoraRTC.createClient({ mode: "rtc", codec: "vp8" })

function App() {
  return (
    <AgoraRTCProvider client={client}>
      <VideoCall />
    </AgoraRTCProvider>
  )
}

function VideoCall() {
  const { isConnected } = useJoin({ appid: APP_ID, channel: "test", token: null })
  const { localMicrophoneTrack } = useLocalMicrophoneTrack()
  const { localCameraTrack } = useLocalCameraTrack()
  usePublish([localMicrophoneTrack, localCameraTrack])
  const remoteUsers = useRemoteUsers()

  return (
    <div>
      <div id="local">
        {localCameraTrack && <LocalVideoTrack track={localCameraTrack} play />}
      </div>
      {remoteUsers.map(user => (
        <RemoteUser key={user.uid} user={user} playVideo playAudio />
      ))}
    </div>
  )
}
```

## Custom Hooks Pattern (agent-toolkit)

For production apps, the Agora Conversational AI agent-toolkit provides a pattern using custom hooks with the raw SDK for maximum control.

### RTCHelper Singleton

Wrap the Agora client in a singleton helper for consistent lifecycle management:

```typescript
import AgoraRTC, { IAgoraRTCClient, IMicrophoneAudioTrack, ICameraVideoTrack } from "agora-rtc-sdk-ng"

class RTCHelper {
  private static instance: RTCHelper | null = null
  public client: IAgoraRTCClient | null = null
  public localAudioTrack: IMicrophoneAudioTrack | null = null
  public localVideoTrack: ICameraVideoTrack | null = null

  static getInstance(): RTCHelper {
    if (!RTCHelper.instance) RTCHelper.instance = new RTCHelper()
    return RTCHelper.instance
  }

  async init(config: { appId: string; channel: string; token: string | null; uid: number }) {
    this.client = AgoraRTC.createClient({ mode: "rtc", codec: "vp8" })
    // Store config, setup event listeners...
  }

  async createAudioTrack() {
    this.localAudioTrack = await AgoraRTC.createMicrophoneAudioTrack({
      encoderConfig: "high_quality_stereo",
      AEC: true, ANS: true, AGC: true,
    })
  }

  async join() { await this.client!.join(appId, channel, token, uid) }
  async publish() { await this.client!.publish([this.localAudioTrack!]) }

  async leave() {
    this.localAudioTrack?.stop(); this.localAudioTrack?.close(); this.localAudioTrack = null
    this.localVideoTrack?.stop(); this.localVideoTrack?.close(); this.localVideoTrack = null
    await this.client?.leave()
  }

  destroy() { this.leave(); RTCHelper.instance = null }
}
```

## useLocalVideo Hook

Manages camera track lifecycle, device enumeration, and switching:

```typescript
import { useState, useEffect, useCallback } from "react"
import AgoraRTC, { ICameraVideoTrack } from "agora-rtc-sdk-ng"

interface UseLocalVideoConfig {
  deviceId?: string
  encoderConfig?: string
  startEnabled?: boolean
}

function useLocalVideo(config: UseLocalVideoConfig = {}) {
  const [videoTrack, setVideoTrack] = useState<ICameraVideoTrack | null>(null)
  const [isVideoEnabled, setIsVideoEnabled] = useState(config.startEnabled ?? false)
  const [cameras, setCameras] = useState<MediaDeviceInfo[]>([])
  const [error, setError] = useState<string | null>(null)

  const refreshCameras = useCallback(async () => {
    const devices = await AgoraRTC.getCameras()
    setCameras(devices)
  }, [])

  const enableVideo = useCallback(async () => {
    try {
      const track = await AgoraRTC.createCameraVideoTrack({
        cameraId: config.deviceId,
        encoderConfig: config.encoderConfig || "720p_2",
      })
      setVideoTrack(track)
      setIsVideoEnabled(true)
    } catch (err) {
      setError((err as Error).message)
    }
  }, [config.deviceId, config.encoderConfig])

  const disableVideo = useCallback(async () => {
    if (videoTrack) {
      videoTrack.stop()
      videoTrack.close()
      setVideoTrack(null)
    }
    setIsVideoEnabled(false)
  }, [videoTrack])

  const switchCamera = useCallback(async (deviceId: string) => {
    if (videoTrack) await videoTrack.setDevice(deviceId)
  }, [videoTrack])

  // Cleanup on unmount
  useEffect(() => {
    return () => { videoTrack?.stop(); videoTrack?.close() }
  }, [videoTrack])

  return { videoTrack, isVideoEnabled, cameras, enableVideo, disableVideo, switchCamera, refreshCameras, error }
}
```

## useRemoteVideo Hook

Tracks remote video users and auto-subscribes:

```typescript
import { useState, useEffect } from "react"
import { IAgoraRTCClient, IRemoteVideoTrack } from "agora-rtc-sdk-ng"

interface RemoteVideoUser {
  uid: number | string
  videoTrack: IRemoteVideoTrack | undefined
}

function useRemoteVideo(config: { client: IAgoraRTCClient; autoSubscribe?: boolean }) {
  const [remoteVideoUsers, setRemoteVideoUsers] = useState<Map<number | string, RemoteVideoUser>>(new Map())

  useEffect(() => {
    const { client } = config

    const onPublished = async (user: any, mediaType: string) => {
      if (mediaType !== "video") return
      if (config.autoSubscribe !== false) {
        await client.subscribe(user, "video")
      }
      setRemoteVideoUsers(prev => new Map(prev).set(user.uid, { uid: user.uid, videoTrack: user.videoTrack }))
    }

    const onUnpublished = (user: any, mediaType: string) => {
      if (mediaType !== "video") return
      setRemoteVideoUsers(prev => { const next = new Map(prev); next.delete(user.uid); return next })
    }

    const onLeft = (user: any) => {
      setRemoteVideoUsers(prev => { const next = new Map(prev); next.delete(user.uid); return next })
    }

    client.on("user-published", onPublished)
    client.on("user-unpublished", onUnpublished)
    client.on("user-left", onLeft)

    return () => {
      client.off("user-published", onPublished)
      client.off("user-unpublished", onUnpublished)
      client.off("user-left", onLeft)
    }
  }, [config.client])

  return { remoteVideoUsers: [...remoteVideoUsers.values()], count: remoteVideoUsers.size }
}
```

## Full React App Example

```tsx
import { useEffect, useRef, useState } from "react"
import AgoraRTC, { IAgoraRTCClient, IMicrophoneAudioTrack, ICameraVideoTrack } from "agora-rtc-sdk-ng"

const APP_ID = "your-app-id"

function VideoCall({ channel, token }: { channel: string; token: string | null }) {
  const clientRef = useRef<IAgoraRTCClient | null>(null)
  const [localAudio, setLocalAudio] = useState<IMicrophoneAudioTrack | null>(null)
  const [localVideo, setLocalVideo] = useState<ICameraVideoTrack | null>(null)
  const [remoteUsers, setRemoteUsers] = useState<Map<number, any>>(new Map())
  const [joined, setJoined] = useState(false)

  useEffect(() => {
    const client = AgoraRTC.createClient({ mode: "rtc", codec: "vp8" })
    clientRef.current = client

    // Register events before join
    client.on("user-published", async (user, mediaType) => {
      await client.subscribe(user, mediaType)
      if (mediaType === "video") {
        setRemoteUsers(prev => new Map(prev).set(user.uid as number, user))
      }
      if (mediaType === "audio") user.audioTrack?.play()
    })

    client.on("user-left", (user) => {
      setRemoteUsers(prev => { const m = new Map(prev); m.delete(user.uid as number); return m })
    })

    return () => {
      localAudio?.stop(); localAudio?.close()
      localVideo?.stop(); localVideo?.close()
      client.leave()
    }
  }, [])

  const join = async () => {
    const client = clientRef.current!
    await client.join(APP_ID, channel, token, null)
    const [audio, video] = await AgoraRTC.createMicrophoneAndCameraTracks()
    setLocalAudio(audio)
    setLocalVideo(video)
    await client.publish([audio, video])
    setJoined(true)
  }

  return (
    <div>
      {!joined && <button onClick={join}>Join</button>}
      <LocalPlayer videoTrack={localVideo} />
      {[...remoteUsers.values()].map(user => (
        <RemotePlayer key={user.uid} user={user} />
      ))}
    </div>
  )
}

function LocalPlayer({ videoTrack }: { videoTrack: ICameraVideoTrack | null }) {
  const ref = useRef<HTMLDivElement>(null)
  useEffect(() => {
    if (videoTrack && ref.current) videoTrack.play(ref.current)
  }, [videoTrack])
  return <div ref={ref} style={{ width: 640, height: 480 }} />
}

function RemotePlayer({ user }: { user: any }) {
  const ref = useRef<HTMLDivElement>(null)
  useEffect(() => {
    if (user.videoTrack && ref.current) user.videoTrack.play(ref.current)
  }, [user.videoTrack])
  return <div ref={ref} style={{ width: 640, height: 480 }} />
}
```

## Official Documentation

For APIs or features not covered above:
- Web SDK API Reference: https://api-ref.agora.io/en/video-sdk/web/4.x/index.html
- agora-rtc-react: https://docs.agora.io/en/video-calling/overview/product-overview
- Guides: https://docs.agora.io/en/video-calling/overview/product-overview
