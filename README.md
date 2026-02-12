# Agora Skill for Claude Code

An [Agora](https://www.agora.io) (agora.io) skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that provides comprehensive real-time communication SDK knowledge across RTC (video/voice), RTM (signaling/messaging), Conversational AI (voice AI agents), and server-side token generation.

## Table of Contents

1. [Overview](#overview)
2. [Design Philosophy — 4-Layer Progressive Disclosure](#design-philosophy--4-layer-progressive-disclosure)
3. [Skill Architecture](#skill-architecture)
4. [For Users — Installing and Using the Skill](#for-users--installing-and-using-the-skill)
5. [For Developers — Maintaining and Extending](#for-developers--maintaining-and-extending)
6. [Packaging and Publishing](#packaging-and-publishing)
7. [Content Maintenance](#content-maintenance)

## Overview

This skill gives Claude deep knowledge of Agora's real-time communication platform. When a user mentions Agora, RTC, RTM, video calling, or conversational AI, Claude progressively loads the relevant reference material — from high-level product overviews down to platform-specific code examples and API details.

The skill covers:
- **RTC (Video/Voice SDK)** — Web, React, iOS (Swift), Android (Kotlin/Java)
- **RTM (Signaling)** — Web (JS/TS) messaging, presence, metadata
- **Conversational AI** — REST API, agent configuration, web client SDK (agent-toolkit)
- **Server-Side** — Token generation for Node.js, Python, Go (+ PHP, Java, C++)

## Design Philosophy — 4-Layer Progressive Disclosure

LLM context windows are finite. Skills should load the minimum context needed to answer a question, then go deeper only when required. This skill uses a 4-layer model:

| Layer | What | Size | When Loaded |
|-------|------|------|-------------|
| **1 — Description** | Trigger keywords in SKILL.md frontmatter | ~100 words | Always in context (skill index) |
| **2 — SKILL.md body** | Core concepts, product index, framework notes | ~65 lines | On skill activation |
| **3 — Product README** | Product overview, critical rules, topic links | 23–82 lines | When a specific product is needed |
| **4 — Topic files** | Deep implementation detail, code examples, API reference | 237–424 lines | When a specific topic is needed |

Every file ends with official Agora documentation URLs as a fallback for information not covered in the skill.

**How Claude navigates**: `SKILL.md` → product `README.md` → topic file (e.g., `web.md`, `rest-api.md`)

## Skill Architecture

```
agora/
├── SKILL.md                                    (64 lines)  Layer 2 — entry point, product index
└── references/
    ├── rtc/                                                 RTC (Video/Voice SDK)
    │   ├── README.md                           (82 lines)  Layer 3 — critical rules, encoder profiles, platform links
    │   ├── web.md                             (422 lines)  Layer 4 — agora-rtc-sdk-ng: client, tracks, events, examples
    │   ├── react.md                           (295 lines)  Layer 4 — agora-rtc-react: hooks, custom patterns
    │   ├── ios.md                             (289 lines)  Layer 4 — AgoraRtcEngineKit (Swift): setup, delegation
    │   └── android.md                         (328 lines)  Layer 4 — RtcEngine (Kotlin/Java): setup, callbacks
    ├── rtm/                                                 RTM (Signaling / Messaging)
    │   ├── README.md                           (29 lines)  Layer 3 — key concepts, platform links
    │   └── web.md                             (250 lines)  Layer 4 — agora-rtm v2: messaging, presence, connection mgmt
    ├── conversational-ai/                                   Conversational AI (Voice AI Agents)
    │   ├── README.md                           (75 lines)  Layer 3 — architecture, endpoints, auth, lifecycle
    │   ├── rest-api.md                        (371 lines)  Layer 4 — full REST API: request/response, rate limits
    │   ├── agent-config.md                    (261 lines)  Layer 4 — properties object: LLM, TTS, ASR, VAD, deprecations
    │   └── web-client.md                      (424 lines)  Layer 4 — agent-toolkit SDK, React hooks, agent states, events
    └── server/                                              Server-Side (Tokens)
        ├── README.md                           (23 lines)  Layer 3 — token types, when tokens are needed
        └── tokens.md                          (237 lines)  Layer 4 — token generation (Node/Python/Go), security practices
```

**14 files, ~3150 lines total.**

## For Users — Installing and Using the Skill

### Personal Install

Symlink the skill into your Claude Code skills directory:

```bash
ln -s /path/to/skills/agora ~/.claude/skills/agora
```

### Plugin Install

If published as a Claude Code plugin, install via the marketplace. See the [Claude Code plugin documentation](https://docs.anthropic.com/en/docs/claude-code/skills) for details.

### Usage

Mention Agora in your conversation or use `/agora` — Claude will progressively load the relevant reference material.

**Trigger keywords**: Agora, agora.io, RTC, RTM, video calling, voice calling, real-time communication, `agora-rtc-sdk-ng`, `agora-rtc-react`, `agora-rtm`, conversational AI (with Agora context), token generation.

**Available products and what triggers each**:

| Product | Triggers |
|---------|----------|
| RTC | Video/voice calls, channels, tracks, streaming, `agora-rtc-sdk-ng` |
| RTM | Messaging, signaling, presence, `agora-rtm` |
| Conversational AI | Voice AI agents, agent-toolkit, ConvoAI REST API |
| Server | Token generation, authentication, App Certificate |

## For Developers — Maintaining and Extending

### Adding a New Product

1. Create `references/{product}/README.md` (Layer 3 — overview, critical rules, topic links)
2. Add an entry to the **Products** section of `SKILL.md`
3. Create topic files as needed (Layer 4)

### Adding a New Platform to an Existing Product

1. Create `references/{product}/{platform}.md` (Layer 4 — implementation detail)
2. Add a link in the product's `README.md` under "Platform Reference Files"

### Updating Content

- Edit the specific Layer 4 file for the content that changed.
- Preserve the **Official Documentation** URLs section at the bottom of every file.
- Keep content factual and code-example-driven — avoid filler.

### Updating Trigger Keywords

Edit the `description` field in `SKILL.md` frontmatter (the YAML between `---` markers). This is what Claude Code uses to decide when to activate the skill.

## Packaging and Publishing

### Claude Code Plugin Structure

```
.claude-plugin/
├── plugin.json           # Plugin manifest (name, version, description)
└── marketplace.json      # Marketplace metadata (optional, for publishing)
skills/
└── agora/
    ├── SKILL.md
    └── references/
        └── ...
```

### Marketplace Publishing

1. Ensure the repo has a `.claude-plugin/plugin.json` manifest.
2. Add `.claude-plugin/marketplace.json` with marketplace metadata.
3. Publish via git repository URL.

See the [Claude Code plugin documentation](https://docs.anthropic.com/en/docs/claude-code/skills) for the full manifest schema and publishing workflow.

## Content Maintenance

### Verify Documentation URLs

Periodically check that official Agora doc URLs still resolve (no 404s or unexpected redirects):

```bash
# Check all URLs in the skill files
grep -roh 'https://[^ )]*' agora/ | sort -u | while read url; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  echo "$status $url"
done
```

### Monitor Agora Release Notes

Keep content in sync with Agora platform updates:
- **Conversational AI**: https://docs.agora.io/en/conversational-ai/overview/release-notes
- **Video SDK**: https://docs.agora.io/en/video-calling/overview/release-notes
- **Signaling (RTM)**: https://docs.agora.io/en/signaling/overview/release-notes

Watch for: new API parameters, deprecations, new products, SDK version bumps, and breaking changes.
