---
title: "Why I Rewrote Memoria in Rust"
title_zh: "为什么我用 Rust 重写了 Memoria"
date: "2026-03-25"
tag: "Engineering"
tag_zh: "工程实践"
status: "published"
description: "From Python prototype to Rust production binary — a database kernel engineer's journey rewriting Memoria, and how AI flattened Rust's learning curve."
description_zh: "从 Python 原型到 Rust 生产级二进制——一位数据库内核工程师重写 Memoria 的历程，以及 AI 如何抹平了 Rust 的学习曲线。"
---

# Why I Rewrote Memoria in Rust

*Peng Xu · Tech Lead, MatrixOne*

---

## A Bit About My Background

I've been writing code for close to twenty years, spending nearly the last decade on database kernel development.

In April 2019, I encountered a reverse image search requirement. The initial approach was straightforward — a Python app calling Faiss for vector retrieval, good enough to get the job done. But as I dug deeper, I realized that vector similarity search isn't an application-layer problem; it should be database-level infrastructure. So I designed and implemented the first several versions of Milvus from scratch. If you look at the earliest commits in the Milvus repository, the entire kernel architecture and core implementation were essentially done by me alone. The pre-1.0 Milvus kernel was written entirely in C++ — and not ordinary C++ either. GPU heterogeneous indexing, multi-GPU scheduling, hybrid filtering, dynamic index deletion and updates, all while handling mixed compute-intensive and IO-intensive workloads. At the time, there were virtually no existing reference solutions in the industry for these problems. Milvus later grew into one of the world's most well-known open-source vector databases, but that's a story for another day.

After that, I joined MatrixOne and designed and implemented the entire storage engine. MatrixOne is written in Golang, and my daily work revolves around the storage engine, transaction processing, and query optimization. You could say I'm someone who lives inside compilers and type systems — I expect problems to be caught before the code ever runs.

A quick word about [MatrixOne](https://github.com/matrixorigin/matrixone), since many people may not be familiar with it yet. It's a database kernel project we've been building from scratch for the past five years. It solves two core problems: first, a single storage and compute engine that handles OLTP, OLAP, vector, and full-text workloads simultaneously — no need to shuttle data between different systems. Second, through its cloud-native, storage-compute separation architecture, it delivers **Git for Data** capabilities — you can perform Git-like snapshots, branches, diffs, and merges on terabyte-scale data. Under the hood, this is powered by Copy-on-Write data versioning at the storage engine level: zero-copy branching, instant snapshots, and point-in-time rollback. This is incredibly useful for application development, data engineering, and AI training alike.

Today's story isn't about MatrixOne though — it's about Memoria. But Memoria is built entirely on top of MatrixOne.

```
┌──────────────────────────────────────────────────────────┐
│                  MatrixOne Architecture                    │
│                                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │   OLTP   │ │   OLAP   │ │  Vector  │ │ FullText │    │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘    │
│       └─────────────┴────────────┴─────────────┘          │
│                         │                                 │
│              ┌──────────▼──────────┐                      │
│              │  Unified Compute    │                      │
│              │  Engine             │                      │
│              └──────────┬──────────┘                      │
│                         │                                 │
│              ┌──────────▼──────────┐                      │
│              │  CoW Storage Engine │ ◄── Git for Data     │
│              │                     │                      │
│              │  • Zero-copy branch │                      │
│              │  • Instant snapshot │                      │
│              │  • Point-in-time    │                      │
│              │    rollback         │                      │
│              │  • Diff / Merge     │                      │
│              └──────────┬──────────┘                      │
│                         │                                 │
│              ┌──────────▼──────────┐                      │
│              │  Cloud-Native       │                      │
│              │  Storage-Compute    │                      │
│              │  Separation         │                      │
│              │  (S3 / Object Store)│                      │
│              └─────────────────────┘                      │
└──────────────────────────────────────────────────────────┘
```

## How AI Changed My Worldview, and Why I Started Building Memoria

The database kernel world is deterministic and controllable. But starting in late 2024, AI coding tools began to genuinely change how I work. At first it was just Copilot doing basic completions. Then I noticed AI could do more and more — not just autocomplete, but understand context, refactor code, write tests. As a systems programmer, I started to realize something: **AI isn't just changing how we write code. It's changing what projects are worth building.**

That shift in thinking pushed me directly toward AI-related projects.

MatrixOne natively supports vector indexing, and we'd been thinking about how to make vector search tuning smarter — not having users manually tweak parameters, but letting the system optimize itself based on query patterns and data distribution. This idea required a "memory layer": the system needed to remember past queries, remember which tuning strategies worked, remember user behavior patterns.

That was the origin of [Memoria](https://github.com/matrixorigin/Memoria). It was initially designed for automatic vector search tuning in MatrixOne — a memory system that could persist across sessions, support semantic retrieval, and handle version management.

## From Database Tuning to AI Agent Memory: The Same Problem

At the same time, as someone who uses coding agents every day, I noticed something interesting: **the memory needs of a coding agent and the memory needs of AI-driven database tuning are fundamentally the same problem.** An agent needs to remember your project structure, your coding preferences, where you left off debugging, which approaches have already been tried and failed. A database tuning system needs to remember query patterns, remember which indexing strategies worked. At the abstract level, they're identical — both require "cross-session, searchable, version-controlled persistent memory."

And we already had MatrixOne's Git for Data capabilities in hand. Zero-copy branching means an agent can do speculative reasoning without affecting its main memory. Instant snapshots mean any memory mutation can be rolled back. Vector indexing means semantic retrieval is natively supported. None of this needed to be built from scratch — MatrixOne had already been refined for years.

```
      Database Auto-Tuning                     Coding Agent
    ┌─────────────────┐                  ┌─────────────────┐
    │ Remember queries │                  │ Remember project │
    │ Remember tuning  │                  │ Remember prefs   │
    │   strategies     │                  │ Remember debug   │
    │ Remember data    │                  │   progress       │
    │   distribution   │                  │ Rollback failed  │
    │ Rollback failed  │                  │   approaches     │
    │   attempts       │                  │                  │
    └────────┬────────┘                  └────────┬────────┘
             │                                    │
             │     The same abstract problem       │
             └──────────┬─────────────────────────┘
                        │
                        ▼
          ┌──────────────────────────┐
          │   Persistent Memory Layer │
          │                          │
          │  • Cross-session persist │
          │  • Semantic retrieval    │
          │  • Version control       │
          │    (Git for Data)        │
          └──────────┬───────────────┘
                     │
                     ▼
          ┌──────────────────────────┐
          │       MatrixOne          │
          │  CoW · Vector Index ·    │
          │  Branching               │
          └──────────────────────────┘
```

So Memoria naturally evolved from an internal database tuning tool into a general-purpose memory layer for AI agents.

Looking back, there's a recurring thread throughout my career: **start from a concrete need, discover the general problem, then build infrastructure.** Reverse image search → Milvus. Database tuning → Memoria. The pattern is the same. And Memoria's core — embeddings, semantic retrieval, hybrid indexing — happens to be a natural extension of the domain knowledge I accumulated over years of building Milvus, applied to a new context.

The first version was written in Python. The reason was simple: speed. Python is the default language in the AI ecosystem — LangChain, sentence-transformers, every embedding model SDK is Python-first. As a prototype, Python let us get the full pipeline running in days, from semantic retrieval to Git-style branching.

But as Memoria moved from prototype toward something we actually needed to use internally, the problems with Python started surfacing one after another.

## The Problem with Python: Not Unusable, Just Increasingly Painful

Dynamic typing has a classic pain point: bugs hide until runtime.

You write a function, annotate a parameter as `str`, but Python won't check that at compile time. One day a `None` sneaks in, the program blows up in production, and the stack trace points somewhere you never expected. You add type hints, run mypy, but mypy can't cover every scenario — especially when dynamic reflection and third-party libraries are involved.

This wasn't an isolated incident. I kept running into these issues while developing Memoria:

- An `Optional[str]` not handled correctly, causing the embedding service to intermittently return empty vectors
- Exceptions silently swallowed in async code, because asyncio's error propagation doesn't work the way you'd intuitively expect
- Dependency management was a nightmare — the same `pip install` could produce completely different results on different users' machines
- Distribution was even worse — either make users install a Python environment, or use PyInstaller to produce a package hundreds of megabytes in size

None of these problems are fatal on their own, but together they create constant friction. Every time I wanted to add a new feature, I had to first spend time dealing with these "infrastructure-level" headaches.

I've had a principle since my MatrixOne kernel days: **if it can be rewritten in a compiled language, don't use Python.** The time had finally come to apply that principle to Memoria.

## Why Rust, Not Go?

This is a fair question. I've written Go for years. The entire MatrixOne kernel is in Go. By all accounts, Go should be my comfort zone.

But I've learned lessons about language choices the hard way. When building Milvus, everything before 1.0 was C++. C++ gave me ultimate performance and control, but the cost was enormous — memory safety issues were a constant threat, development velocity was low, and onboarding new contributors was painful. Later, Milvus 2.0 was rewritten in Go, which dramatically improved development efficiency, but in scenarios demanding peak performance, the GC was always a lurking concern. These two experiences gave me a very practical framework for language decisions: **it's not about which language is "best" — it's about which language offers the most reasonable trade-offs for the specific scenario.**

Memoria's use case is different from a database kernel. Memoria is a service that needs to be distributed to and run on users' local machines. That means:

- **Binary size matters.** Users won't install a hundreds-of-megabytes runtime just for a memory server. Rust compiles to self-contained binaries, typically just a few MB.
- **Memory footprint matters.** Memoria runs as a background process alongside the IDE. Go's GC is good, but Rust's zero-cost abstractions and GC-free design mean memory usage can be almost negligible.
- **Performance ceiling matters.** Semantic retrieval involves vector computation; branch merging involves large-scale data diffing. Rust's performance advantage in these scenarios is real.

From C++ to Go to Rust, I've traveled a complete arc. C++ taught me the value of performance and control. Go taught me the value of development velocity and engineering discipline. Rust is the best balance point I've found so far — it delivers C++-level performance with far better memory safety guarantees than C++, and with AI assistance, development velocity is no longer a bottleneck.

```
  Perf  ▲
        │
  Peak  │  ★ C++                              ★ Rust (+ AI)
        │  Milvus 1.0                         Memoria
        │  · Peak performance ✓                · Peak performance ✓
        │  · Memory safety    ✗                · Memory safety    ✓
        │  · Dev velocity     ✗                · Dev velocity     ✓ (AI)
        │
        │                 ★ Go
        │                 MatrixOne
        │                 · Good performance ✓
        │                 · Memory safety    ✓ (GC)
        │                 · Dev velocity     ✓
        │
        └──────────────────────────────────────────────► Safety
```

More importantly, Rust in 2026 is not the same Rust as a few years ago — not because the language changed, but because the way you learn it has.

## AI Has Flattened Rust's Learning Curve

Let me be upfront about something: **when I started the rewrite, I knew almost nothing about Rust.**

I understood the concepts — ownership, borrow checking, lifetimes — but I'd never actually built a complete project in Rust. Two years ago, that would have meant months of grinding through The Rust Programming Language, fighting the compiler, searching Stack Overflow for lifetime errors.

But things are different now.

This February, Anthropic researcher Nicholas Carlini published a blog post about how he used 16 parallel Claude instances to build a Rust-based C compiler from scratch — one capable of compiling the entire Linux 6.9 kernel across x86, ARM, and RISC-V architectures. 100,000 lines of Rust code, roughly 2,000 Claude Code sessions, $20,000 in API costs.

This hit me hard. Not because AI can write compilers — compilers are fundamentally well-defined engineering problems. What truly struck me was: **AI chose Rust as the implementation language, and the resulting code quality was sufficient to compile the Linux kernel.** After reading that blog post, I decided to give it a shot with Memoria.

This case demonstrated two things:

1. AI's understanding of Rust is deep enough to handle ownership, lifetimes, and the trait system — the parts that make Rust "hard"
2. Rust's type system and compiler actually become an advantage for AI — the compiler tells AI exactly what's wrong, and AI can self-correct based on compilation errors

This stands in stark contrast to Python. Python's flexibility is an advantage for humans, but a disadvantage for AI — without a compiler as a safety net, AI-generated Python code is more likely to harbor bugs that only surface at runtime.

Around the same time, Andrej Karpathy mentioned on a podcast that he hadn't hand-written a single line of code since December 2025 — flipping from writing 80% of his code himself to having AI write 80%. That ratio has kept shifting.

My experience was similar. During the AI-assisted Memoria rewrite, I only needed to do three things:

1. **Define the architecture** — tell AI what module structure, data flow, and API design I wanted
2. **Review the code** — check whether AI-generated code matched my intent and had no logical issues
3. **Handle compilation errors** — though most of the time AI could fix them itself based on `cargo build` output

Rust's compiler plays a critical role in this workflow: it's the "quality inspector" between AI and human. Every compilation tells you (and the AI): "here's a problem, here's exactly what it is, here's how to fix it." This feedback loop simply doesn't exist in dynamic languages.

```
  Rust + AI Feedback Loop                 Python + AI Workflow
  =======================                 =======================

  Human: Define arch / requirements       Human: Define arch / requirements
         │                                       │
         ▼                                       ▼
  ┌─────────────┐                         ┌─────────────┐
  │ AI generates │                         │ AI generates │
  │    code      │                         │    code      │
  └──────┬──────┘                         └──────┬──────┘
         │                                       │
         ▼                                       ▼
  ┌─────────────┐    compile error        ┌─────────────┐
  │ cargo build │ ──────────┐              │  Run directly│
  └──────┬──────┘           │              └──────┬──────┘
         │ compiles ok      │                     │
         ▼                  │                     ▼
    ✅ Basically works  AI auto-fixes       ❌ Blows up at
                       (precise errors)       runtime
```

## The Industry Is Moving This Way

I'm not the only one thinking along these lines.

GitHub's Octoverse 2025 report revealed an interesting trend: AI coding tools are creating a "convenience loop" that reshapes developer language choices. TypeScript surpassed both Python and JavaScript on GitHub with 66% growth, becoming the most-used language. One reason: static types provide a safety net for AI-generated code.

The same logic applies to Rust. Stack Overflow's 2024 survey shows Rust maintaining its "most admired" status with an 83% rating. The State of Rust Survey indicates that nearly half of responding enterprises are already using Rust in production. Microsoft has announced plans to migrate C/C++ code to Rust by 2030 using AI-assisted tooling.

Python's share of AI infrastructure-layer code is being eroded by compiled languages. Python remains king for AI research and prototyping, but in systems software that needs to be deployed, distributed, and maintained long-term, compiled languages are reclaiming ground.

This aligns perfectly with what I've seen in the vector database space. Milvus's Python SDK was always the ecosystem entry point, but the kernel was always a compiled language. Qdrant chose Rust from day one. LanceDB is Rust too. In the AI infrastructure arena, "prototype in Python, ship in a compiled language" is no longer a personal preference — it's industry consensus.

For services that need to run persistently inside a user's IDE, zero runtime overhead, memory safety, and single-binary distribution aren't nice-to-haves — they're requirements. Rust is becoming the default choice for these scenarios.

## What the Rewrite Gave Us

After rewriting Memoria in Rust, we got several "free" improvements:

```
                    Python                    Rust
                ┌──────────────┐         ┌──────────────┐
  Binary size   │   ~300 MB    │   →     │    <10 MB    │   ↓ 97%
                ├──────────────┤         ├──────────────┤
  Resident mem  │   200+ MB    │   →     │    ~20 MB    │   ↓ 90%
                ├──────────────┤         ├──────────────┤
  Startup time  │   seconds    │   →     │ milliseconds │   ↓ 99%
                ├──────────────┤         ├──────────────┤
  Dependencies  │ Python + pip │   →     │ single binary│   zero deps
                │ + venv + ... │         │              │
                ├──────────────┤         ├──────────────┤
  Runtime errs  │  anytime     │   →     │ compile-time │   ≈ 0
                └──────────────┘         └──────────────┘
```

- **Binary size**: from hundreds of MB with Python + PyInstaller down to a single executable under 10 MB
- **Memory footprint**: resident memory dropped from 200MB+ to around 20MB
- **Startup time**: from Python interpreter cold-start taking seconds to Rust binary startup in milliseconds
- **Distribution**: users download one binary and it just works — no Python installation, no pip, no virtual environments
- **Reliability**: if it compiles, it basically runs — no more surprise `AttributeError: 'NoneType' object has no attribute 'xxx'` at runtime

These improvements weren't the result of careful optimization. They came for free just by switching languages. That's what I mean by "free upgrades."

## Final Thoughts

I'm not saying Python is bad. Python's position in AI research, data analysis, and rapid prototyping won't be challenged anytime soon. But for systems software that needs to be distributed to users, maintained long-term, and run in resource-constrained environments, Python's weaknesses are becoming increasingly apparent.

And Rust's biggest historical barrier — its steep learning curve — is being rapidly flattened by AI. When AI can handle the details of ownership and lifetimes for you, when the compiler serves as AI's real-time feedback mechanism, "Rust is too hard" stops being a valid objection.

As a database kernel programmer who wrote Go for years, I never imagined I'd rewrite a project in Rust. But AI changed the equation. It let me focus on architecture design and product thinking instead of wrestling with language syntax.

From hand-writing Milvus in C++ in 2019, to building MatrixOne in Go, to rewriting Memoria in Rust today — each language choice was the optimal decision given the constraints of its time. And the arrival of AI has transformed Rust from "want to use but can't afford to" into "no reason not to."

**The system instantly gained memory safety, performance, and distribution size improvements — essentially for free.**

That's why I rewrote Memoria in Rust.

---

*[Memoria](https://github.com/matrixorigin/memoria) is an open-source persistent memory layer for AI agents, supporting Kiro, Cursor, Claude Code, and other mainstream AI coding tools. Powered by [MatrixOne](https://github.com/matrixorigin/matrixone), a cloud-native converged database.*
