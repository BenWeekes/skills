# Agora Skills — Eval Cases

Evaluation-driven regression testing. Run these cases after every skill change.

## Evaluation Method

For each case:
1. Send "User Input" to the model with skills loaded
2. Check if model behavior matches "Expected Behavior"
3. Mark PASS / FAIL
4. Failed cases drive skill modifications

---

## 1. Routing Accuracy (R-series)

### R-01: RTC Web

- User Input: "How do I implement a video call on Web?"
- Expected Behavior: Routes to `references/rtc/web.md`, provides initialization and track management guidance
- Pass Criteria: References `AgoraRTC.createClient`; does not route through intake
- Result: ___

### R-02: RTC iOS

- User Input: "RTC on iOS Swift"
- Expected Behavior: Routes to `references/rtc/ios.md`
- Pass Criteria: References `AgoraRtcEngineKit`; not Android or Web SDK
- Result: ___

### R-03: RTC Android

- User Input: "RTC Android Kotlin"
- Expected Behavior: Routes to `references/rtc/android.md`
- Pass Criteria: References `RtcEngine`; Kotlin syntax
- Result: ___

### R-04: RTC React

- User Input: "Agora React hooks"
- Expected Behavior: Routes to `references/rtc/react.md`
- Pass Criteria: References `agora-rtc-react` or `AgoraRTCProvider`
- Result: ___

### R-05: ConvoAI Python

- User Input: "ConvoAI agent in Python"
- Expected Behavior: Routes to `references/conversational-ai/README.md`
- Pass Criteria: References ConvoAI REST API; does not reference RTC SDK directly as the primary API
- Result: ___

### R-06: TEN Framework extension

- User Input: "TEN Framework extension"
- Expected Behavior: Routes to `references/ten-framework/README.md`
- Pass Criteria: References `AsyncExtension` or `ten_runtime`
- Result: ___

### R-07: Server-side token generation

- User Input: "Generate an RTC token in Go"
- Expected Behavior: Routes to `references/server/tokens.md`
- Pass Criteria: Provides token generation guidance; references Go SDK
- Result: ___

### R-08: RTM Web

- User Input: "RTM messaging on Web"
- Expected Behavior: Routes to `references/rtm/web.md`
- Pass Criteria: References `agora-rtm`; not `agora-rtc-sdk-ng`
- Result: ___

---

## 2. Code Generation Quality (C-series)

### C-01: agent_rtc_uid type

- User Input: "Create a ConvoAI agent in Python"
- Expected Behavior: `agent_rtc_uid` is string `"0"`
- Pass Criteria: Not int `0`
- Result: ___

### C-02: remote_rtc_uids type

- User Input: Same as C-01
- Expected Behavior: `remote_rtc_uids` is `["*"]`
- Pass Criteria: Not string `"*"`
- Result: ___

### C-03: Agent name uniqueness

- User Input: Same as C-01
- Expected Behavior: Agent `name` includes a random suffix (e.g., `agent_{uuid[:8]}`)
- Pass Criteria: Not a fixed string like `"my_agent"`
- Result: ___

### C-04: Credentials not hardcoded

- User Input: Same as C-01
- Expected Behavior: AppID, Customer Key/Secret read from environment variables
- Pass Criteria: No hardcoded credential values in generated code
- Result: ___

### C-05: RTC Web autoplay restriction

- User Input: "Play audio track in RTC on Web"
- Expected Behavior: Skill reminds to call `track.play()` after a user gesture
- Pass Criteria: References autoplay restriction; does not generate code that calls `play()` without user interaction context
- Result: ___

### C-06: Next.js SSR pattern

- User Input: "Agora RTC in Next.js"
- Expected Behavior: Uses `"use client"` directive and dynamic import pattern
- Pass Criteria: Generated code matches the Web Framework Notes pattern in the skill; does not use `next/dynamic` with `ssr: false` as the recommended approach
- Result: ___

### C-07: MCP server features

- User Input: "What features could we add to an MCP server with ConvoAI?"
- Expected Behavior: Describes MCP tool patterns (`save_memory`, `search_memory`, etc.); references `server-mcp` recipe
- Pass Criteria: Does not fabricate non-existent Agora APIs
- Result: ___

### C-08: TEN Framework nuclear restart

- User Input: "How do I restart a TEN Framework extension?"
- Expected Behavior: Uses the nuclear restart procedure from `references/ten-framework/operations.md`
- Pass Criteria: Does not recommend `docker restart` alone; references the full restart sequence including `pkill`, lock file removal, and `task run`
- Result: ___

---

## 3. Failure Paths (F-series)

### F-01: Vague request — no product specified

- User Input: "Build me an AI assistant for my app"
- Expected Behavior: Does not generate code immediately; asks clarifying questions or routes through intake (when PRD-05 intake skill is complete)
- Pass Criteria: Does not fabricate an Agora product choice; acknowledges ambiguity
- Result: ___

### F-02: MCP server unreachable

- Scenario: MCP server unreachable
- User Input: "Integrate ConvoAI in Python"
- Expected Behavior: Informs user MCP is unavailable; falls back to local references and provides fallback URL
- Pass Criteria: Does not fabricate REST API parameters; notifies user to verify against official docs
- Result: ___

### F-03: Non-existent "create channel" API

- User Input: "Create an RTC channel on Web"
- Expected Behavior: Explains that channels auto-create when the first user joins via `client.join()`
- Pass Criteria: Does not generate a fabricated "create channel" API call
- Result: ___

---

## Evaluation Log

| Date | Skill Version | Pass | Fail | Failed Cases | Fix Actions |
|------|--------------|------|------|-------------|-------------|
| | | | | | |
