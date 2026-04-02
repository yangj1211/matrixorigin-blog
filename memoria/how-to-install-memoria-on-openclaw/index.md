---
title: "Install Memoria on OpenClaw in 3 Minutes"
title_zh: "1 分钟在 OpenClaw 上安装 Memoria"
date: "2026-04-01"
tag: "Tutorial"
tag_zh: "教程"
status: "published"
description: "A step-by-step guide to replacing OpenClaw's default file-based memory with Memoria — semantic retrieval, version control, and cross-agent memory sharing."
description_zh: "一篇安装教程：将 OpenClaw 默认的文件记忆替换为 Memoria，获得语义检索、版本控制和跨 Agent 记忆共享能力。"
---

# Install Memoria on OpenClaw in 1 Minutes

Have you noticed that after using OpenClaw for a while, it starts to feel a little forgetful — or more precisely, like it's remembering things wrong?

OpenClaw's memory system is built on plain Markdown files: long-term memory lives in `MEMORY.md`, which gets loaded into context at the start of every private session. It's transparent and easy to understand, and early on it works without friction. But as usage grows, cracks start to show:

- **Files bloat and content gets silently truncated**: `MEMORY.md` and other memory files have character limits. Once exceeded, content is cut without any warning — the agent doesn't throw an error, it just forgets.
- **Retrieval quality degrades**: OpenClaw's default hybrid search (vector + keyword) works well when memory is small, but struggles with relational reasoning as entries accumulate. Write "Alice manages the auth team" on Monday, then ask "who handles permission issues?" on Friday — the system surfaces chunks about Alice and chunks about auth, but can't connect them.
- **Poor cross-session continuity**: Context compaction summarizes older context to save tokens. Memory file contents injected into the context window can get rewritten or dropped in the process.

This isn't a bug in OpenClaw. It's the ceiling that file-based memory inevitably hits at scale.

## What Memoria Fixes

[Memoria](https://github.com/matrixorigin/memoria) is a persistent memory layer designed for AI agents. It sits on top of OpenClaw's existing file system and provides stronger retrieval and management capabilities.

**More accurate semantic retrieval.** Memoria doesn't rely on simple vector similarity matching — it understands relationships between memory entries. Cross-entry reasoning like connecting "Alice manages the auth team" to "who handles permission issues?" returns structured context, not a pile of loosely related chunks.

**Memory that survives compaction.** Memoria stores memories outside the context window entirely. Session restarts and compaction events leave memory intact. Relevant entries are injected fresh each turn, unconstrained by token limits.

**Version control: traceable and reversible.** Memoria supports snapshots, branches, and rollbacks. If a memory update causes unexpected agent behavior, you can roll back to the last known-good state without manually digging through files.

**Cross-agent sharing.** The same memory store can be accessed by multiple agent instances. No more starting from scratch every time you open a new conversation.

The free tier includes **20,000 memory entries** and **2,000 retrieval API calls per month** — plenty for everyday personal use.

## Installation

First, grab your Memoria API key. Go to the [Memoria Dashboard](https://thememoria.ai/dashboard), sign up with GitHub, Google, or email, and copy your key (format: `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`).

![Get your API key from the Memoria Dashboard](./images/api_key.png)

Then confirm OpenClaw is running:

```bash
openclaw status
```

Pick either method below — both get you to the same place.

### Option 1: Command Line

**Step 1: Install the plugin**

```bash
openclaw plugins install @matrixorigin/thememoria
```

**Step 2: Configure and enable**

```bash
openclaw memoria setup \
  --mode cloud \
  --api-url https://api.thememoria.ai \
  --api-key sk-YOUR_API_KEY
```

**Step 3: Verify**

```bash
openclaw memoria health
```

You should see `"status": "ok"`. That's it.

### Option 2: Chat Install

Prefer to stay in the chat interface? Send the following prompt to OpenClaw directly. The agent will execute each step in order and report back at every stage:

```
Install the Memoria memory plugin for my OpenClaw in cloud mode.
Credentials (pre-filled from my Memoria account):
- API URL: https://api.thememoria.ai
- API Key: sk-YOUR_API_KEY
Run these steps in order. Stop and report if any step fails.
1) Install plugin:
   openclaw plugins install @matrixorigin/thememoria
2) Setup cloud backend (this also enables the plugin):
   openclaw memoria setup --mode cloud --api-url https://api.thememoria.ai --api-key sk-YOUR_API_KEY
3) Verify:
   openclaw memoria health
   Expected: "status": "ok"
4) After all steps pass, tell the user:
   "Memoria is installed and healthy. To use memory tools (memory_store, memory_search, etc.), start a new conversation by typing /new — the tools won't appear in this conversation."
Rules:
- Show every command you run and its full raw output
- Do not summarize or hide errors
- If a step fails, classify the error (network / auth / config / missing-binary) and suggest the exact fix command
- Do not skip steps or reorder them
- Do NOT use `openclaw memory` commands — those are built-in file memory, not Memoria. The plugin uses `openclaw memoria`
- Do NOT attempt to use memory_store or other memory tools in this conversation
```

Replace `sk-YOUR_API_KEY` with your actual key before sending. If any step fails, the agent will classify the error and suggest the exact fix — no manual debugging needed.

## Confirm Memoria Is Working

In any OpenClaw conversation, type:

```
List my memoria memories
```

If Memoria is connected correctly, the agent will call the memory tool and return your current memory count (empty on first use is normal). Starting from your next conversation, Memoria is in charge of your memory.