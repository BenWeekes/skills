# Agora Server-Side: Token Generation

## Table of Contents
- [Overview](#overview)
- [Node.js Token Server](#nodejs-token-server)
- [Python Token Generation](#python-token-generation)
- [Go Token Generation](#go-token-generation)
- [Token Types](#token-types)
- [Security Best Practices](#security-best-practices)

## Overview

Tokens authenticate users before they can join channels. **AccessToken2** is the current token format (replaces legacy AccessToken). Token libraries are available for **Node.js**, **Python**, **Go**, **PHP**, **Java**, and **C++**.

Generated server-side from:
- **App ID**: Your project identifier
- **App Certificate**: Secret key (NEVER expose on client)
- **Channel name**: The channel the token grants access to
- **UID**: The user ID the token is valid for (must match join UID)
- **Expiration**: Token validity period (max 24 hours)

The `onTokenPrivilegeWillExpire` callback fires **30 seconds** before token expiration — use it to fetch and renew the token before disconnection.

## Node.js Token Server

### Installation

```bash
npm install agora-token
```

### Express Token Server

```typescript
import express from "express"
import { RtcTokenBuilder, RtcRole } from "agora-token"

const app = express()
const APP_ID = process.env.AGORA_APP_ID!
const APP_CERTIFICATE = process.env.AGORA_APP_CERTIFICATE!

app.get("/api/token", (req, res) => {
  const { channel, uid, role } = req.query

  if (!channel) {
    return res.status(400).json({ error: "channel is required" })
  }

  const uidNum = parseInt(uid as string) || 0
  const tokenRole = role === "audience" ? RtcRole.SUBSCRIBER : RtcRole.PUBLISHER
  const expirationInSeconds = 3600 // 1 hour
  const currentTimestamp = Math.floor(Date.now() / 1000)
  const privilegeExpiredTs = currentTimestamp + expirationInSeconds

  const token = RtcTokenBuilder.buildTokenWithUid(
    APP_ID,
    APP_CERTIFICATE,
    channel as string,
    uidNum,
    tokenRole,
    privilegeExpiredTs
  )

  res.json({ token })
})

app.listen(3001, () => console.log("Token server on :3001"))
```

### String UID Support

```typescript
import { RtcTokenBuilder } from "agora-token"

// For string UIDs (used in some applications)
const token = RtcTokenBuilder.buildTokenWithAccount(
  APP_ID,
  APP_CERTIFICATE,
  channel,
  userAccount, // string
  role,
  privilegeExpiredTs
)
```

## Python Token Generation

### Installation

```bash
pip install agora-token-builder
```

### Flask Token Server

```python
from flask import Flask, request, jsonify
from agora_token_builder import RtcTokenBuilder
import time

app = Flask(__name__)
APP_ID = "your-app-id"
APP_CERTIFICATE = "your-app-certificate"

@app.route("/api/token")
def get_token():
    channel = request.args.get("channel")
    uid = int(request.args.get("uid", 0))
    role = 1  # 1 = publisher, 2 = subscriber

    if not channel:
        return jsonify({"error": "channel required"}), 400

    expiration_in_seconds = 3600
    current_timestamp = int(time.time())
    privilege_expired_ts = current_timestamp + expiration_in_seconds

    token = RtcTokenBuilder.buildTokenWithUid(
        APP_ID, APP_CERTIFICATE, channel, uid, role, privilege_expired_ts
    )

    return jsonify({"token": token})

if __name__ == "__main__":
    app.run(port=3001)
```

## Go Token Generation

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strconv"
    "time"

    rtctokenbuilder "github.com/AgoraIO/Tools/DynamicKey/AgoraDynamicKey/go/src/rtctokenbuilder2"
)

const (
    appID          = "your-app-id"
    appCertificate = "your-app-certificate"
)

func tokenHandler(w http.ResponseWriter, r *http.Request) {
    channel := r.URL.Query().Get("channel")
    uidStr := r.URL.Query().Get("uid")

    if channel == "" {
        http.Error(w, "channel required", http.StatusBadRequest)
        return
    }

    uid, _ := strconv.ParseUint(uidStr, 10, 32)
    expiration := uint32(3600)

    token, err := rtctokenbuilder.BuildTokenWithUid(
        appID, appCertificate, channel, uint32(uid),
        rtctokenbuilder.RolePublisher, expiration, expiration,
    )
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintf(w, `{"token":"%s"}`, token)
}

func main() {
    http.HandleFunc("/api/token", tokenHandler)
    log.Fatal(http.ListenAndServe(":3001", nil))
}
```

## Token Types

### RTC Token
Authenticates users for audio/video channels.

```
RtcTokenBuilder.buildTokenWithUid(appId, cert, channel, uid, role, expireTs)
```

Roles:
- `RtcRole.PUBLISHER` (1): Can publish and subscribe (host)
- `RtcRole.SUBSCRIBER` (2): Can only subscribe (audience)

### RTM Token
Authenticates users for Real-Time Messaging.

```
RtmTokenBuilder.buildToken(appId, cert, userId, role, expireTs)
```

### Combined Token (Token 007)
Newer token format that bundles multiple service privileges:

```typescript
import { AccessToken2, ServiceRtc, ServiceRtm } from "agora-token"

const token = new AccessToken2(APP_ID, APP_CERTIFICATE, issueTs, expire)

// Add RTC service
const rtcService = new ServiceRtc(channel, uid)
rtcService.addPrivilege(ServiceRtc.kPrivilegeJoinChannel, expire)
rtcService.addPrivilege(ServiceRtc.kPrivilegePublishAudioStream, expire)
rtcService.addPrivilege(ServiceRtc.kPrivilegePublishVideoStream, expire)
token.addService(rtcService)

// Add RTM service (optional)
const rtmService = new ServiceRtm(userId)
rtmService.addPrivilege(ServiceRtm.kPrivilegeLogin, expire)
token.addService(rtmService)

const tokenString = token.build()
```

## Security Best Practices

1. **Never expose App Certificate** in client code, environment variables on client, or public repositories
2. **Always validate requests** to your token server (authenticate the user making the request)
3. **Set reasonable expiration** — 1 hour is typical; max 24 hours
4. **Use HTTPS** for your token server endpoint
5. **Rate limit** token generation to prevent abuse
6. **UID must match** — the UID in the token must match the UID used to join the channel
7. **Store secrets securely** — use environment variables or secret managers, not code
8. **Use subscriber role tokens for audience** — generate tokens with `RtcRole.SUBSCRIBER` for audience-only users to prevent stream bombing (unauthorized publishing)

## Official Documentation

For APIs or features not covered above:
- Authentication Guide: https://docs.agora.io/en/video-calling/token-authentication/authentication-workflow
- Deploy a Token Server: https://docs.agora.io/en/video-calling/token-authentication/deploy-token-server
