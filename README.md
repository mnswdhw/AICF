# AI Changefeed (AICF)

**A minimal, web-native protocol for exposing content changes to AI systems.**

> 🚀 **Current Version:** [v0.1 (Draft)](spec/AICF-v0.1.md)  
> ⚠️ **Status:** Early draft - feedback welcome! 

## What is AICF?

AICF enables publishers to expose **what changed** on their sites through a simple append-only feed, so AI systems (crawlers, RAG, agents) can efficiently update their knowledge without expensive full re-crawls.

**Key Features:**
- 🎯 **Section-level updates** via anchors (no need to re-process entire pages)
- 📝 **Simple NDJSON format** (one event per line)
- 🔍 **Standard discovery** via `.well-known/ai-changefeed`
- 🚫 **No complex dependencies** (just HTTP + JSON)
- 🤝 **Complements existing standards** (sitemaps, RSS, MCP)

## Quick Example

Publishers expose changes:
```json
{"id":"01J3Z6B1...","action":"update","url":"https://docs.example/pricing#tiers","time":"2025-08-21T10:15:00Z","anchor":"tiers"}
```

AI systems update only what changed - no more daily full crawls!
Get Started

* 📖 Read the Specification
* 💬 Discuss & Contribute
* 🌟 Star this repo to follow updates

Integration with MCP
AICF feeds work great as data sources for Anthropic's Model Context Protocol (MCP) servers, enabling real-time context updates without full re-indexing.
