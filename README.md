# Agora Skills

Structured reference knowledge for [Agora](https://www.agora.io) (agora.io) real-time communication SDKs, designed for AI coding assistants. Covers RTC (video/voice), RTM (signaling/messaging), Conversational AI (voice AI agents), and server-side token generation.

## Quick Install — Claude Code

Run these two commands inside Claude Code:

```
/plugin marketplace add BenWeekes/skills
/plugin install agora@agora-skills
```

Then **restart Claude Code** (exit and reopen). The skill will appear in `/skills` and activate automatically when you mention Agora, RTC, RTM, video calling, conversational AI, etc.

## Other Installation Methods

### Claude Code — Manual Symlink

```bash
git clone https://github.com/BenWeekes/skills.git ~/agora-skills
mkdir -p ~/.claude/skills
ln -s ~/agora-skills/agora/skills/agora ~/.claude/skills/agora
```

Restart Claude Code after creating the symlink.

### Claude Code — Project-Level

Copy the skill into your project so all team members get it automatically:

```bash
mkdir -p .claude/skills
cp -r /path/to/skills/agora/skills/agora .claude/skills/agora
```

### Other AI Assistants

The skill files are plain markdown — any AI coding assistant that supports custom knowledge or context files can use them. Point your tool at `agora/skills/agora/` or individual files:

- **Cursor** — Add files to `.cursor/rules/` or reference in project settings
- **Windsurf** — Add files to your Cascade context
- **Copilot** — Reference via `@workspace` or include in `.github/copilot-instructions.md`
- **Any tool with custom context** — Feed `SKILL.md` as the entry point; it links to everything else

Each leaf file (Layer 4) is self-contained with full code examples and doc URLs, so you can also load individual files directly.

---

## What This Is

This repo contains markdown skill files that give AI coding assistants deep knowledge of Agora's platform. When a developer asks for help with Agora, the assistant loads the relevant reference material — from high-level product overviews down to platform-specific code examples and API details.

**Products covered:**
- **RTC (Video/Voice SDK)** — Web, React, iOS (Swift), Android (Kotlin/Java)
- **RTM (Signaling)** — Web (JS/TS) messaging, presence, metadata, stream channels
- **Conversational AI** — REST API, agent configuration (LLM/TTS/ASR/MLLM), web client SDK
- **Server-Side** — Token generation for Node.js, Python, Go
- **TEN Framework** — Extension development, graph configuration, Docker operations, debugging

## Design — 4-Layer Progressive Disclosure

LLM context windows are finite. Load the minimum needed, go deeper only when required.

| Layer | What | Size | When Loaded |
|-------|------|------|-------------|
| **1 — Description** | Trigger keywords in `SKILL.md` frontmatter | ~100 words | Always (skill index) |
| **2 — SKILL.md body** | Core concepts, product index, framework notes | ~65 lines | On activation |
| **3 — Product README** | Overview, critical rules, topic links | 23–85 lines | Per product |
| **4 — Topic files** | Implementation detail, code examples, API reference | 237–500 lines | Per topic |

Navigation: `SKILL.md` → product `README.md` → topic file (e.g., `web.md`, `rest-api.md`). Every leaf file ends with official Agora doc URLs as fallback.

## File Structure

```
agora/                                                        Plugin root
├── .claude-plugin/
│   └── plugin.json                                          Plugin manifest
└── skills/
    └── agora/                                               Skill root
        ├── SKILL.md                            (65 lines)   Entry point, product index
        └── references/
            ├── rtc/                                          RTC (Video/Voice SDK)
            │   ├── README.md                   (85 lines)   Critical rules, encoder profiles, cross-platform notes
            │   ├── web.md                     (498 lines)   agora-rtc-sdk-ng: client, tracks, events, screen share
            │   ├── react.md                   (295 lines)   agora-rtc-react: hooks, custom patterns
            │   ├── ios.md                     (301 lines)   AgoraRtcEngineKit (Swift): setup, delegation
            │   └── android.md                 (340 lines)   RtcEngine (Kotlin/Java): setup, callbacks
            ├── rtm/                                          RTM (Signaling / Messaging)
            │   ├── README.md                   (25 lines)   Key concepts, platform links
            │   └── web.md                     (375 lines)   agora-rtm v2: messaging, presence, stream channels
            ├── conversational-ai/                            Conversational AI (Voice AI Agents)
            │   ├── README.md                   (67 lines)   Architecture, endpoints, auth, lifecycle
            │   ├── rest-api.md                (372 lines)   Full REST API: request/response, rate limits
            │   ├── agent-config.md            (375 lines)   Properties: LLM, TTS, ASR, MLLM/Gemini, tools, avatars
            │   └── web-client.md              (426 lines)   agent-toolkit SDK, React hooks, agent states, events
            └── server/                                       Server-Side (Tokens)
                ├── README.md                   (20 lines)   Token types, when tokens are needed
                └── tokens.md                  (238 lines)   Token generation (Node/Python/Go), security practices
    └── ten/                                                 TEN Framework Skill
        ├── SKILL.md                            (76 lines)   Entry point, concepts, critical rules
        └── references/
            ├── extensions.md                  (181 lines)   Extension development, templates, patterns
            ├── graphs.md                      (170 lines)   Graph config, connections, API endpoints
            └── operations.md                  (366 lines)   Docker, startup, logs, deployment, remote access, troubleshooting
```

18 skill files, ~4270 lines total.

## Maintaining and Extending

### Adding a New Product

1. Create `references/{product}/README.md` (Layer 3)
2. Add an entry to the **Products** section of `SKILL.md`
3. Create topic files as needed (Layer 4)

### Adding a New Platform

1. Create `references/{product}/{platform}.md` (Layer 4)
2. Add a link in the product's `README.md`

### Updating Content

- Edit the specific Layer 4 file.
- Keep the **Official Documentation** URLs at the bottom of every leaf file.
- Keep content factual and code-example-driven.

### Verifying URLs

```bash
grep -roh 'https://[^ )]*' agora/ | sort -u | while read url; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -L --max-time 10 "$url")
  echo "$code $url"
done
```

### Agora Release Notes

- Conversational AI: https://docs.agora.io/en/conversational-ai/overview/release-notes
- Video SDK: https://docs.agora.io/en/video-calling/overview/release-notes
- Signaling (RTM): https://docs.agora.io/en/signaling/overview/release-notes
