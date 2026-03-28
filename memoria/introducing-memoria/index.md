---
title: "Introducing Memoria: The World's First Git for AI Agent Memory"
title_zh: "Memoria 发布：AI Agent 记忆的 Git"
date: "2026-03-19"
tag: "Product"
tag_zh: "产品"
status: "published"
description: "The models are brilliant. The memory is the bottleneck. We built Memoria to fix that — version-controlled, semantically-retrieved, self-governing memory for every AI agent."
description_zh: "模型已经很出色，但记忆是瓶颈。我们构建了 Memoria 来解决这个问题——为每个 AI Agent 提供版本控制、语义检索和自治治理的记忆层。"
---

# Introducing Memoria: The World's First Git for AI Agent Memory

## The Memory Problem No One Has Solved

Open Cursor. Tell it your project uses React, TypeScript, and Zustand with the slice pattern. Watch it build exactly what you want.

Close the tab. Open a new conversation. Ask it to add a user module.

> "Sure! What framework are you using? Redux or MobX for state management?"

You told it twenty minutes ago. It already forgot.

Now open OpenClaw. You've been using it for months — it remembers your preferences, your project context, your workflow patterns. But open `MEMORY.md` and look at what's accumulated: hundreds of entries, growing every session. Each one gets injected into the system prompt on every API call, burning tokens whether or not it's relevant. Your monthly bill climbs. You try to move to a new machine or share your setup with a teammate — the memory is scattered across local Markdown files with no export, no sync, no structure.

These aren't edge cases. This is the daily reality for millions of developers using AI agents in 2026. The models are brilliant. The memory is the bottleneck.

## A Walk Through the Current State of Agent Memory

To understand why we built Memoria, it helps to see where the industry actually stands — across coding agents, OpenClaw, and custom-built agents.

### No memory at all

Most coding agents — Cursor, Claude Code, Kiro — ship with zero persistent memory. Every session starts from a blank slate. The agent doesn't know your tech stack, your code style, your naming conventions, or the architectural decisions you made last week. It can only work with what's in the current conversation window.

The cost is invisible but constant: repeated context-setting, redundant questions, inconsistent outputs across sessions. The agent is smart in the moment but has no continuity. It's a new hire every morning.

### Markdown-based memory

The industry's first response was static files. `.cursorrules` tells Cursor your preferences. `CLAUDE.md` does the same for Claude. Kiro has steering files. OpenClaw built a community-driven rule library so developers can share and reuse configurations.

OpenClaw went further with `MEMORY.md` — the agent can write memories to disk and read them back on the next session. This is genuine persistence. But the implementation has structural problems:

- **Token bloat.** All accumulated memories are loaded into the system prompt on every call. By session 10, you're burning 10,000+ tokens of context before the agent even reads your message. Most of those memories are irrelevant to the current task.
- **No semantic retrieval.** Memories are matched by date and category, not by meaning. The agent can't find "formatting preferences" when you ask it to "format this file" — unless the exact keywords happen to match.
- **No portability.** Memories live in local `.md` files. Moving to a new machine, sharing with a teammate, or syncing across devices requires manual copy-paste.
- **No structure.** Code style, architecture decisions, deploy workflows, personal preferences — all mixed in flat files that grow into an unmaintainable mess.
- **No rollback.** If a bad memory entry corrupts the agent's behavior, you have to manually find and delete it. There's no undo.

### Structured memory frameworks

Mem0, Letta, and Zep represent the next generation. They store memories in real databases with vector embeddings, enabling semantic retrieval. This is a meaningful step forward — the agent can find relevant memories by meaning, not just keywords.

But they all share a critical limitation: **memories are append-or-update records with no version control.**

Mem0 provides a general-purpose memory API across 24+ vector database backends. Letta builds tiered memory into its agent framework. Zep constructs temporal knowledge graphs. Each solves the storage and retrieval problem well.

None of them solve the management problem.

## The Problems That Emerge at Scale

When agents accumulate hundreds or thousands of memories over weeks and months of use, a new class of problems appears — problems that storage and retrieval alone can't fix.

### No way to undo

You're refactoring the authentication module. Over three sessions, the agent updates its understanding of your auth architecture — new file structure, new token flow, new API patterns. The refactor fails. You revert the code with `git checkout`. But the agent's memory still reflects the refactored state. It now gives advice based on an architecture that no longer exists.

In every existing memory system, your only option is to manually find and delete each affected memory entry. If you miss one, the agent's behavior stays corrupted.

### No way to experiment safely

You want to evaluate switching from PostgreSQL to SQLite. You tell the agent. It updates its project memory. You explore the idea for a few sessions, decide it won't work, and abandon it. But the agent now "knows" you use SQLite. The old PostgreSQL memories have been overwritten or contradicted.

There's no way to say "let me try this in isolation and merge it back only if it works." Every change is permanent and global.

### Memory drift and contradiction

Over time, memories accumulate contradictions. You told the agent you use black for formatting in January. In March, you switched to ruff. Both memories exist. Which one wins? In most systems, it depends on retrieval ranking — which means the agent might use black on Monday and ruff on Tuesday, depending on how the query happens to match.

### Memory poisoning

This isn't theoretical. Multiple papers from 2025 (MemoryGraft, MINJA, A-MemGuard) demonstrated that adversaries can inject malicious content into an agent's long-term memory through indirect prompt injection — simply by having the agent read a crafted document. The poisoned memories persist across sessions and gradually alter agent behavior. The user may never notice.

Most memory systems have no recovery mechanism beyond manually inspecting every entry. At scale, that's not feasible.

### The common thread

Every one of these problems has the same root cause: **agent memory has no version control.**

Code had the same problems before Git. You couldn't undo safely. You couldn't experiment in isolation. You couldn't trace when a change was introduced or roll back to a known good state. Git solved this for code. Agent memory needs the same thing.

## Memoria: Version Control for Agent Memory

[Memoria](https://github.com/matrixorigin/memoria) is an open-source memory layer that brings Git's core abstractions to AI agent memory. Built in Rust, shipped as a single binary, backed by [MatrixOne](https://github.com/matrixorigin/matrixone)'s Copy-on-Write database engine.

The key operations:

**Snapshot** — save the current memory state before a risky operation. Zero-copy, millisecond completion, zero additional storage until changes are made.

```
You:    "Take a snapshot before we refactor the auth module"
Agent:  → memory_snapshot(name="pre-auth-refactor")
        ✓ Snapshot created.
```

**Rollback** — restore memory to any previous snapshot. One command, deterministic, complete.

```
You:    "The refactor didn't work. Roll back."
Agent:  → memory_rollback(name="pre-auth-refactor")
        ✓ All memory restored to pre-refactor state.
```

**Branch** — create an isolated memory space for experimentation. Changes on a branch don't affect main.

```
You:    "Let's evaluate switching to SQLite"
Agent:  → memory_branch(name="eval-sqlite")
        → memory_checkout(name="eval-sqlite")
        → memory_store("Project uses SQLite", type="semantic")
        (stored on eval-sqlite only — main is untouched)
```

**Diff** — preview what would change before merging. Like a pull request for your agent's knowledge.

```
You:    "What changed on the sqlite branch?"
Agent:  → memory_diff(source="eval-sqlite")
        + [semantic] Project uses SQLite
        ~ [semantic] Database: PostgreSQL → SQLite
        (2 additions, 1 modification)
```

**Merge** — bring a branch back into main after validation.

```
You:    "The experiment worked. Merge it."
Agent:  → memory_merge(source="eval-sqlite")
        ✓ Merged 3 memories from eval-sqlite.
```

These aren't metaphors. They're real operations backed by MatrixOne's CoW engine, which provides branch isolation and zero-copy snapshots at the database level.

## Why This Architecture Matters

Two lines of research from 2025 validate this approach:

**Git operations improve agent reasoning.** The Git-Context-Controller paper (Wu, 2025) showed that adding commit/branch/merge operations to agent context management improved SWE-Bench task resolution by 13 percentage points, reaching 80.2%. The ablation study confirmed that branch and merge specifically contributed the critical final gains. Agents with branching capability spontaneously developed more structured exploration strategies.

**Memory poisoning is a real threat.** MemoryGraft, MINJA, and A-MemGuard demonstrated practical attacks against agent long-term memory. Snapshot + rollback provides a deterministic recovery path that no other memory system offers.

Memoria extends these principles from ephemeral context to persistent memory — a complementary and necessary layer.

## Beyond Version Control: What Else Memoria Does

Version control is the headline, but Memoria is a complete memory infrastructure:

**Typed memories.** Six categories with different retrieval priorities and lifecycle rules:

| Type | Purpose | Example |
|------|---------|---------|
| `semantic` | Project facts, technical decisions | "This project uses Go 1.22 with modules" |
| `profile` | User preferences | "Always use pytest, never unittest" |
| `procedural` | Workflows, how-to knowledge | "To deploy: make build → kubectl apply" |
| `working` | Current task context | "Currently refactoring the auth module" |
| `tool_result` | Cached tool outputs | Stored command results |
| `episodic` | Session summaries | "Session: DB optimization → added indexes → 93% faster" |

**Semantic retrieval.** Hybrid vector + full-text search. Store "black formatter" and search "formatting tool" — semantic matching finds it. Scored by relevance and recency.

**Self-governance.** Three maintenance tools run periodically:
- `memory_governance` — quarantines low-confidence memories, cleans stale data
- `memory_consolidate` — detects contradictions (like the black vs. ruff problem), fixes orphaned entries
- `memory_reflect` — synthesizes high-level insights from memory clusters via LLM

**Configurable embedding.** OpenAI, SiliconFlow, Ollama, or any OpenAI-compatible endpoint. Local embedding available when building from source.

## How It Compares

| | Memoria | Mem0 | Letta | Zep/Graphiti |
|---|---|---|---|---|
| Version control | Snapshot, branch, merge, rollback, diff | Timestamps only | None | Bitemporal fact tracking |
| Storage | Single database (MatrixOne) | 24+ vector DB backends | PostgreSQL + vector DB | Neo4j |
| Retrieval | Structured+ Vector + Full-text + hybrid | Vector | Vector + structured | Graph traversal |
| Self-governance | Contradiction detection + quarantine + reflect | AUDN conflict resolution | Sleep-time compute | Temporal invalidation |
| Interface | MCP Server + OpenClaw plugin + REST API | SDK/API | Agent framework | SDK/API |
| Target | Any agent (coding, OpenClaw, custom) | AI app developers | Letta framework users | Knowledge graph users |
| License | Apache 2.0 | Partially open | Apache 2.0 | Apache 2.0 |

Different tools for different problems. Mem0 is right if you need 3-line integration across 20+ frameworks. Letta is right if you're building agents from scratch within their framework. Zep is right if you need temporal knowledge graphs.

Memoria is for anyone who wants their agent's memory to be as manageable as their code — with the full safety net of version control.

## Get Started

Memoria ships as a single Rust binary. Setup depends on your agent type.

### Coding Agents (Cursor / Kiro / Claude Code)

```bash
# 1. Install Memoria
curl -sSL https://raw.githubusercontent.com/matrixorigin/Memoria/main/scripts/install.sh | bash

# 2. Start MatrixOne (or use MatrixOne Cloud — free tier, no Docker needed)
docker compose up -d

# 3. Configure your tool
cd your-project
memoria init --tool kiro \
             --embedding-provider openai \
             --embedding-base-url https://api.siliconflow.cn/v1 \
             --embedding-api-key sk-... \
             --embedding-model BAAI/bge-m3 \
             --embedding-dim 1024
```

Replace `kiro` with `cursor` or `claude`. Restart your AI tool. Done.

### OpenClaw

The [openclaw-memoria](https://github.com/matrixorigin/openclaw-memoria) plugin replaces OpenClaw's default file-based memory with the full Memoria backend — semantic retrieval, version control, self-governance.

```bash
curl -fsSL https://raw.githubusercontent.com/matrixorigin/openclaw-memoria/main/scripts/install-openclaw-memoria.sh | \
  env MEMORIA_DB_URL='mysql+pymysql://root:111@127.0.0.1:6001/memoria' \
      MEMORIA_EMBEDDING_PROVIDER='openai' \
      MEMORIA_EMBEDDING_MODEL='text-embedding-3-small' \
      MEMORIA_EMBEDDING_API_KEY='sk-...' \
      MEMORIA_EMBEDDING_DIM='1536' \
      bash -s --
```

### Custom Agents / Any MCP-Compatible Tool

Point any MCP-compatible agent at Memoria:

```json
{
  "mcpServers": {
    "memoria": {
      "command": "memoria",
      "args": ["mcp", "--db-url", "mysql+pymysql://root:111@localhost:6001/memoria", "--user", "alice"]
    }
  }
}
```

Or connect to a deployed Memoria REST API:

```json
{
  "mcpServers": {
    "memoria": {
      "command": "memoria",
      "args": ["mcp", "--api-url", "https://your-server:8100", "--token", "sk-your-key..."]
    }
  }
}
```

## What's Next

Memoria is Apache 2.0 licensed and fully open source. This is an early release — the architecture is solid, the edges are still being refined.

We're also building **Memoria Cloud** — a managed service so you can get version-controlled agent memory without running your own database. Stay tuned.

In the meantime, we'd genuinely appreciate your feedback. Try it, break it, tell us what doesn't work. The GitHub issues tab is open.

- **Memoria:** [github.com/matrixorigin/memoria](https://github.com/matrixorigin/memoria)
- **OpenClaw Plugin:** [github.com/matrixorigin/openclaw-memoria](https://github.com/matrixorigin/openclaw-memoria)
- **MatrixOne:** [github.com/matrixorigin/matrixone](https://github.com/matrixorigin/matrixone)

Code has Git. Agent memory now has Memoria.

---

*Built by [MatrixOrigin](https://github.com/matrixorigin). Open-sourced at GTC 2026.*




