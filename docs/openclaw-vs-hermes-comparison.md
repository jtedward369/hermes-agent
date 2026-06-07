# OpenClaw vs. Hermes Agent — Architectural Comparison

> A thorough comparison of two self-hosted AI agent frameworks, focusing on
> core architecture differences and self-evolving capabilities.

**Date:** 2026-05-25
**Sources:** OpenClaw design docs (~5,900 lines), Hermes design doc (~1,484 lines),
plus direct source analysis of both codebases.

---

## Executive Summary

| Dimension | OpenClaw | Hermes |
|-----------|---------|--------|
| Language | TypeScript (Node.js 22+) | Python 3.11+ |
| Codebase size | 365K symbols, 17.5K files | 138K nodes, 3.4K files |
| Architecture | Plugin-first gateway server | Monolithic agent with plugin extensions |
| Self-evolution | Multi-phase dreaming system (automated) | Background review + curator (triggered) |
| Memory model | File-based + SQLite vector index | File-based (MEMORY.md) + external providers |
| Provider abstraction | 50-hook ProviderPlugin trait | 4-method ProviderTransport ABC |
| Platform adapters | 20+ (channel plugins) | 20+ (platform adapters) |
| Personality | SOUL.md + agent identity files | System prompt + MEMORY.md preferences |

---

## 1. Architecture Philosophy

### OpenClaw: "Gateway-First, Plugin-Everything"

OpenClaw is designed as a **local gateway server** that clients connect to. Everything
is a plugin — providers, channels, memory backends, tools. The gateway owns the
lifecycle, and plugins register capabilities through a typed SDK.

```
Clients → Gateway Server → Plugin Registry → Agent Runtime → Provider Plugins
                                          ↓
                              Channel Plugins (messaging platforms)
```

**Key characteristics:**
- Plugin SDK with 50+ hooks for deep customization
- Protocol-level versioning (JSON-RPC v4 over WebSocket)
- Lazy initialization (nothing loads until first use)
- Prompt cache optimization throughout the architecture
- Everything is eventual — hooks, events, async lifecycle

### Hermes: "Agent-First, Direct Execution"

Hermes is designed as a **standalone agent** that can be invoked directly.
The gateway is an add-on that wraps the agent. Extensions are more like
configuration than deep integration.

```
CLI/Gateway → AIAgent.run_conversation() → Transport → Provider
                         ↓
              Tool Registry (auto-discovered)
```

**Key characteristics:**
- Single ~12K LOC file (`run_agent.py`) contains the core loop
- Direct function calls instead of event/hook systems
- Thread-based concurrency (ThreadPoolExecutor for tools)
- Immediate execution — no lazy loading
- Provider adapters are thin format converters (50-200 LOC each)

---

## 2. Self-Evolving Capabilities — The Core Difference

This is where the two systems diverge most significantly.

### OpenClaw: Automated Multi-Phase Dreaming System

OpenClaw has a **fully automated** knowledge consolidation system that runs
without user intervention:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  OPENCLAW SELF-EVOLUTION STACK                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────────────┐ │
│  │ Active Memory │   │ Memory Flush  │   │ Feedback Reflection   │ │
│  │ (proactive    │   │ (before       │   │ (thumbs-down          │ │
│  │  recall)      │   │  compaction)  │   │  learning)            │ │
│  └───────┬───────┘   └───────┬───────┘   └───────────┬───────────┘ │
│          │                    │                        │              │
│          ▼                    ▼                        ▼              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              SHORT-TERM RECALL STORE                            │  │
│  │  (memory/.dreams/short-term-recall.json)                       │  │
│  │  Tracks: recall count, query diversity, recency, scores        │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                              │                                       │
│                    ┌─────────▼─────────┐                            │
│                    │    DREAMING        │  ← Cron-scheduled          │
│                    │                    │                            │
│                    │  Light Phase:      │                            │
│                    │    ingest signals  │                            │
│                    │    dedupe, stage   │                            │
│                    │                    │                            │
│                    │  REM Phase:        │                            │
│                    │    extract themes  │                            │
│                    │    reinforcement   │                            │
│                    │                    │                            │
│                    │  Deep Phase:       │                            │
│                    │    6-signal RANK   │                            │
│                    │    threshold gates │                            │
│                    │    PROMOTE →       │─────→ MEMORY.md            │
│                    │                    │                            │
│                    └─────────┬─────────┘                            │
│                              │                                       │
│                    ┌─────────▼─────────┐                            │
│                    │   DREAM DIARY     │                            │
│                    │   (DREAMS.md)     │  ← Human review surface    │
│                    └───────────────────┘                            │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              COMMITMENTS                                        │  │
│  │  Inferred follow-up obligations from conversation              │  │
│  │  Delivered via heartbeat on future sessions                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              SOUL.md                                            │  │
│  │  Persistent personality/voice (tone, opinions, bluntness)      │  │
│  │  Injected on every session                                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key innovations:**
1. **Automated promotion** — no human intervention needed to move important facts to long-term memory
2. **6-signal weighted scoring** — frequency (0.24), relevance (0.30), diversity (0.15), recency (0.15), consolidation (0.10), conceptual (0.06)
3. **Gate requirements** — must pass minScore + minRecallCount + minUniqueQueries before promotion
4. **Dream Diary** — transparent human-reviewable log of what was considered/promoted
5. **Grounded backfill** — can replay historical notes and evaluate them for promotion
6. **Phase reinforcement** — Light/REM phases boost Deep-phase scores for items they surfaced

### Hermes: Triggered Background Review + Curator

Hermes uses a **triggered** approach — self-improvement happens after each turn
via a background thread, and periodically via a curator:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  HERMES SELF-EVOLUTION STACK                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  BACKGROUND REVIEW (after every turn)                          │  │
│  │                                                                │  │
│  │  spawn_background_review_thread():                             │  │
│  │    1. Fork AIAgent with conversation snapshot                  │  │
│  │    2. Same model/auth (hits prefix cache)                      │  │
│  │    3. Tool whitelist: memory + skill_manage only               │  │
│  │    4. Ask: "should any memory/skill be saved or updated?"      │  │
│  │    5. Agent writes directly to MEMORY.md or skill files        │  │
│  │                                                                │  │
│  │  Two review prompts:                                           │  │
│  │    • Memory review: user persona, preferences, behavior        │  │
│  │    • Skill review: corrections, patterns, class-level skills   │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  CURATOR (periodic, idle-triggered)                             │  │
│  │                                                                │  │
│  │  Interval: 7 days (DEFAULT_INTERVAL_HOURS = 168)               │  │
│  │  Min idle: 2 hours before running                              │  │
│  │  Responsibilities:                                             │  │
│  │    • Auto-transition skill lifecycle states                    │  │
│  │    • Pin / archive / consolidate / patch skills                │  │
│  │    • Never auto-delete (only archive, recoverable)             │  │
│  │    • Only touches agent-created skills                         │  │
│  │                                                                │  │
│  │  Timing constants:                                             │  │
│  │    • DEFAULT_STALE_AFTER_DAYS = 30                             │  │
│  │    • DEFAULT_ARCHIVE_AFTER_DAYS = 90                           │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  MEMORY (file-based, per-turn sync)                            │  │
│  │                                                                │  │
│  │  MEMORY.md: agent's personal notes (env facts, conventions)    │  │
│  │  USER.md: what agent knows about the user (preferences)        │  │
│  │                                                                │  │
│  │  • Injected as frozen snapshot at session start                │  │
│  │  • Mid-session writes update disk, NOT system prompt           │  │
│  │  • Preserves prefix cache for entire session                   │  │
│  │  • Entry delimiter: § (section sign)                           │  │
│  │  • Threat scanning on writes (prompt injection detection)      │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  EXTERNAL MEMORY PROVIDERS (plugin-based, one at a time)       │  │
│  │                                                                │  │
│  │  Lifecycle: initialize → prefetch → sync_turn → shutdown       │  │
│  │  Hooks: on_turn_start, on_session_end, on_pre_compress,        │  │
│  │         on_memory_write, on_delegation                         │  │
│  │                                                                │  │
│  │  Providers: Honcho, Mem0, Supermemory, Hindsight, Holographic, │  │
│  │             ByteRover, RetainDB, OpenViking                    │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  INSIGHTS ENGINE (analytics, not learning)                     │  │
│  │                                                                │  │
│  │  Analyzes historical session data for usage reports:            │  │
│  │    • Token consumption & cost estimates                        │  │
│  │    • Tool usage patterns                                       │  │
│  │    • Activity trends                                           │  │
│  │    • Model/platform breakdowns                                 │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key innovations:**
1. **Per-turn background review** — every conversation turn triggers potential learning
2. **Skill library as learning artifact** — creates class-level skills from corrections
3. **Curator lifecycle management** — skills transition: active → stale → archived
4. **Two-store separation** — MEMORY.md (agent facts) vs. USER.md (user facts)
5. **Prefix cache preservation** — memory writes don't invalidate the system prompt cache
6. **Security scanning** — injection/exfiltration pattern detection on memory writes

---

## 3. Detailed Self-Evolution Comparison

| Feature | OpenClaw | Hermes |
|---------|---------|--------|
| **Learning trigger** | Cron-scheduled (dreaming), automatic flush, thumbs-down | Per-turn background thread, idle-triggered curator |
| **Learning method** | Statistical scoring (6 signals) | LLM-based review (forked agent) |
| **Promotion criteria** | Quantitative gates (score > 0.75, recalls >= 3, unique queries >= 2) | Qualitative (LLM decides if worthy of saving) |
| **Memory structure** | MEMORY.md + daily notes + short-term store + phase signals | MEMORY.md + USER.md (§-delimited entries) |
| **Memory search** | Hybrid vector + BM25 (SQLite FTS5 + sqlite-vec) | External providers (Honcho/Mem0) or basic file read |
| **Skill creation** | N/A (skills are static files) | Agent-created skills with lifecycle (active → stale → archived) |
| **Skill maintenance** | N/A | Curator reviews every 7 days, consolidates/archives |
| **Personality** | SOUL.md (static, injected every session) | System prompt + MEMORY.md preferences |
| **Follow-ups** | Commitments (inferred, heartbeat-delivered) | N/A |
| **Proactive recall** | Active Memory plugin (blocking sub-agent before each reply) | MemoryProvider.prefetch() (background pre-turn recall) |
| **Human review** | DREAMS.md dream diary | Curator report (last_report_path) |
| **Safety** | No memory injection scanning mentioned | Explicit threat pattern scanning on memory writes |
| **Cache strategy** | Active Memory has its own caching (15s TTL, circuit breaker) | Frozen snapshot at session start (prefix cache preserved) |
| **Multi-agent learning** | N/A | on_delegation hook: parent observes subagent work |
| **Consolidation** | Dreaming merges short-term into MEMORY.md | Curator consolidates narrow skills into class-level skills |
| **Transparency** | DREAMS.md shows scoring, candidates, decisions | Curator state file + report path |

---

## 4. Provider Abstraction

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Interface size** | ~50 hooks (ProviderPlugin trait) | 4 core methods (ProviderTransport ABC) |
| **Responsibility** | Transport + auth + schema normalization + replay + streaming + error classification | Format conversion only (messages, tools, kwargs, response) |
| **Auth handling** | Provider plugin owns auth (prepareRuntimeAuth, OAuth flows) | Agent owns auth (credential_pool, base_url rotation) |
| **Error handling** | Provider classifies errors (matchesContextOverflowError, classifyFailoverReason) | Central classifier (agent/error_classifier.py) handles all providers |
| **Streaming** | Provider plugin wraps stream (wrapStreamFn, createStreamFn) | Agent owns streaming (interruptible_streaming_api_call) |
| **Replay policy** | Per-provider (ProviderReplayPolicy with 13 fields) | N/A (agent handles history replay uniformly) |
| **Tool schema** | Provider normalizes tools (normalizeToolSchemas) | Transport converts tools (convert_tools) |
| **Model resolution** | Multi-step resolution (resolve_dynamic_model → normalize → contribute_compat) | Direct model string passed to API |

### Philosophy Difference

**OpenClaw:** "The provider knows best — give it hooks for everything"
- Complex but allows deep provider-specific optimizations
- Each provider can have unique replay rules, tool format, auth flow
- 50+ hooks means total control but high implementation cost

**Hermes:** "The agent knows best — providers just translate formats"
- Simple but limits provider-specific optimization
- All providers share the same error classification, retry, streaming logic
- 4 methods means easy to add new providers (50-200 LOC each)

---

## 5. Memory Architecture

### OpenClaw Memory Stack

```
Layer 1: Active Memory Plugin
  → Blocking sub-agent runs BEFORE each reply
  → Searches memory, injects hidden context
  → Configurable query mode (message/recent/full)
  → Circuit breaker (3 timeouts → 60s cooldown)

Layer 2: Memory Core Plugin
  → memory_search tool (hybrid vector + BM25)
  → memory_get tool (file access)
  → Memory flush before compaction
  → Dreaming (background promotion)

Layer 3: Memory Index (SQLite + Embeddings)
  → SQLite FTS5 for keyword search
  → sqlite-vec for vector similarity
  → 8 embedding providers supported
  → Temporal decay (30-day half-life)
  → MMR diversity filtering

Layer 4: File System
  → MEMORY.md (long-term, loaded every session)
  → memory/YYYY-MM-DD.md (daily notes)
  → DREAMS.md (dream diary)
  → memory/.dreams/ (machine state)
```

### Hermes Memory Stack

```
Layer 1: MemoryProvider Plugin (one external at a time)
  → prefetch(query) before each turn
  → sync_turn(user, assistant) after each turn
  → on_session_end(messages) at session close
  → on_pre_compress(messages) before compression

Layer 2: Built-in Memory Tool
  → MEMORY.md (agent facts, § delimited)
  → USER.md (user facts, § delimited)
  → add/replace/remove/read actions
  → Threat scanning on writes
  → Frozen snapshot at session start

Layer 3: Background Review
  → Forked agent reviews conversation
  → Writes to MEMORY.md/USER.md or skill files
  → Runs after every turn (daemon thread)

Layer 4: Curator
  → Periodic skill library maintenance
  → Consolidation, archival, lifecycle transitions
```

### Key Differences

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| Search technology | Built-in hybrid (vector + BM25) | External provider or none |
| Automation level | Fully automated promotion pipeline | LLM-driven per-turn review |
| Cost | Embedding API calls + cron scheduling | Extra LLM call per turn (background review) |
| Determinism | Quantitative scoring (reproducible) | LLM judgment (non-deterministic) |
| Latency | Active Memory adds 3-15s to first reply | Background review doesn't block replies |
| Storage | Complex (.dreams/ with 5+ JSON files) | Simple (two .md files + optional provider) |

---

## 6. Context Management

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Compression trigger** | Token estimate > soft threshold (configurable) | Token estimate > model context length |
| **Compression method** | CompactionProvider.summarize() (pluggable) | ContextCompressor (auxiliary LLM call) |
| **Summary budget** | Configurable (reserveTokensFloor: 20K) | Proportional: max(2K, min(12K, 20% × compressed)) |
| **Head/tail protection** | Compaction safeguard (preservedMessages) | Head + tail budget (tail = 25% of context) |
| **Tool result pruning** | Context pruning (soft_trim at 0.3, hard_clear at 0.5) | Pre-pass prunes old tool outputs before summarization |
| **Image handling** | IMAGE_CHAR_ESTIMATE = 8,000 | _IMAGE_TOKEN_ESTIMATE = 1,600 tokens |
| **Cache-aware** | Yes (pruning only after cache TTL expires) | Yes (frozen snapshot preserves prefix cache) |
| **Failure handling** | Compaction provider handles errors | 600s cooldown after failure (_SUMMARY_FAILURE_COOLDOWN) |
| **Summary prefix** | Short marker for runtime use | Long instructional prefix (tells agent not to re-execute) |

---

## 7. Gateway & Platform System

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Architecture** | Plugin-based channel system | Direct adapter pattern |
| **Base class** | None (SDK helpers: defineChannelPluginEntry) | BasePlatformAdapter ABC |
| **Registration** | Plugin manifest + registry | PlatformRegistry + config.yaml |
| **Session binding** | Complex (thread-aware routing, conversation binding) | Simple (platform + channel_id + user_id) |
| **Streaming delivery** | Event frames over WebSocket | Platform-specific chunked delivery |
| **Protocol** | JSON-RPC v4 over WebSocket (typed, versioned) | None (direct function calls) |
| **Hook system** | 37 named hooks with categories | HookRegistry (simpler, gateway-only) |
| **Pairing** | Device pairing (crypto, TLS pinning) | N/A |
| **Platforms** | Telegram, Discord, Slack, iMessage, Matrix, MS Teams, WhatsApp, Signal, IRC, Line, Feishu, etc. | Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Email, SMS, Feishu, DingTalk, WeCom, Weixin, QQBot, BlueBubbles, HomeAssistant, Webhook, etc. |
| **China platforms** | Limited | Extensive (DingTalk, WeCom, Weixin, QQBot, Yuanbao) |

---

## 8. Tool System

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Registration** | Plugin SDK (defineChannelPluginEntry, tool factories) | Central registry (tools/registry.py, auto-discover) |
| **Discovery** | Plugin loading + manifest | AST scan for `registry.register()` calls |
| **Execution** | Sandboxed (workspace guards, append-only mode) | Direct execution (approval callbacks) |
| **Safety** | Tool policy system (allow/deny per group/sender) | Tool guardrails (loop detection, warnings, hard stops) |
| **MCP support** | Yes (src/mcp/) | Yes (tools/mcp_tool.py) |
| **Approval** | Plugin-level approval hooks | Callback-based (CLI approval, Discord buttons) |
| **Subagents** | Yes (sessions_spawn, sessions_send) | Yes (delegate_task, kanban workers) |
| **Budget** | N/A | IterationBudget (90 parent, 50 subagent) |
| **Loop detection** | N/A | ToolCallGuardrailController (exact failure, same-tool, no-progress) |

---

## 9. Error Handling & Recovery

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Classification** | Per-provider hooks (classifyFailoverReason) | Central classifier (agent/error_classifier.py) |
| **Taxonomy** | ~10 categories | 19 FailoverReason variants |
| **Recovery actions** | Provider-specific (hooks decide) | Structured hints (retryable, should_compress, should_rotate, should_fallback) |
| **Credential rotation** | Per-provider (prepareRuntimeAuth, OAuth refresh) | CredentialPool (round-robin rotation on rate limit) |
| **Backoff** | Default 250ms base, 1000ms max, 2 attempts | 5s base, 120s max, decorrelated jitter |
| **Context overflow** | CompactionProvider + context pruning | ContextCompressor + pre-pass tool pruning |
| **Provider-specific errors** | Yes (ThinkingSignature, LongContextTier, etc.) | Yes (thinking_signature, llama_cpp_grammar_pattern, oauth_long_context_beta) |

---

## 10. Configuration

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Format** | JSON (openclaw.json) | YAML (config.yaml) + .env |
| **Schema** | TypeBox (runtime-validated) | Loose dict (no formal schema) |
| **Hot reload** | Yes (gateway detects changes) | No (restart required) |
| **Layers** | Runtime override → config → agent config → defaults | Config file → env vars → CLI args |
| **Location** | ~/.openclaw/ | ~/.hermes/ |
| **Profiles** | Per-agent config (agents/<id>/config.json) | HERMES_HOME env var (profile switching) |
| **Secrets** | SecretRef (env/file/exec sources) | .env file (plain text keys) |

---

## 11. Concurrency Model

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Runtime** | Node.js event loop (single-threaded + workers) | Python asyncio + threading |
| **Tool execution** | Spawned subprocess (sandboxed) | ThreadPoolExecutor (in-process) |
| **Background tasks** | Cron service (isolated agent) | Daemon threads (curator, background review) |
| **Multi-session** | Per-connection state in gateway | SessionStore (thread-safe dict) |
| **Cancellation** | AbortSignal / CancellationToken | threading.Event (interrupt flag) |
| **Memory safety** | V8 isolation between turns | Global state (ToolRegistry singleton, check_fn cache) |

---

## 12. Self-Evolution Philosophy Summary

### OpenClaw's Approach: "Sleep on It"

Inspired by neuroscience (dreaming as memory consolidation):
- **Observe** → track recall frequency, query diversity, retrieval quality
- **Score** → 6-signal weighted ranking with quantitative thresholds
- **Promote** → only items passing all gates move to long-term memory
- **Review** → human can inspect DREAMS.md dream diary
- **Forget** → temporal decay ensures stale information loses weight

**Strengths:**
- Deterministic and reproducible
- Low per-turn cost (scoring is pure math)
- No LLM calls for promotion decisions
- Rich provenance (every promoted entry has audit trail)
- Grounded backfill allows re-evaluating historical notes

**Weaknesses:**
- Complex to implement (5+ JSON files, cron scheduling)
- Slow to promote (needs multiple recalls across different queries)
- No skill/behavior learning (only facts)
- Active Memory adds latency to every reply

### Hermes's Approach: "Learn by Doing"

Inspired by human self-reflection (review what happened, extract lessons):
- **Review** → forked agent re-reads conversation after each turn
- **Extract** → LLM identifies memory-worthy facts and skill-worthy patterns
- **Create** → writes directly to MEMORY.md/USER.md or creates new skills
- **Maintain** → curator periodically consolidates and archives stale skills
- **Correct** → skill review catches user corrections as skill updates

**Strengths:**
- Immediate learning (same turn, not days later)
- Can create new SKILLS (not just facts) from corrections
- Simple implementation (fork agent + restricted tool access)
- User corrections directly update behavior (skill_manage)
- Curator prevents skill rot (30-day stale → 90-day archive)

**Weaknesses:**
- Expensive (extra LLM call per turn for background review)
- Non-deterministic (LLM may miss or hallucinate learnings)
- No quantitative quality signal (no recall frequency, no diversity score)
- No vector search on memories (depends on external providers)
- Can be aggressive ("most sessions produce at least one skill update")

---

## 13. Key Insight: What Each System Optimizes For

| Optimization | OpenClaw | Hermes |
|-------------|---------|--------|
| **Primary goal** | High-quality long-term memory | Fast behavioral adaptation |
| **Cost model** | Embed once, score cheaply, promote rarely | Extra LLM call per turn |
| **Latency model** | Active Memory adds upfront cost | Background review is non-blocking |
| **Learning unit** | Facts (sentences in MEMORY.md) | Facts (MEMORY.md) + Skills (files with SKILL.md) |
| **Quality signal** | Recall frequency (how often was it useful?) | LLM judgment (does this seem worth saving?) |
| **Failure mode** | Under-promotes (high threshold, slow) | Over-promotes (aggressive, non-deterministic) |
| **Human trust** | Transparent (DREAMS.md shows all decisions) | Opaque (LLM decides, no scoring trace) |
| **Best for** | Long-running persistent agents (weeks/months) | Active development assistants (corrections → skills) |

---

## 14. Recommendations for a Hybrid System

A best-of-both-worlds reimplementation could combine:

1. **Hermes-style per-turn review** for immediate correction capture
2. **OpenClaw-style quantitative scoring** for promotion gating (prevent over-saving)
3. **Hermes-style skill creation** for behavioral learning (not just facts)
4. **OpenClaw-style vector search** for recall quality (vs. external providers)
5. **OpenClaw-style dreaming** for periodic consolidation of skill library
6. **Hermes-style curator** for skill lifecycle management
7. **Both:** threat scanning on memory writes (Hermes has this, OpenClaw doesn't mention it)
8. **Both:** prefix cache preservation (both optimize for this differently)

### Proposed Architecture

```
Per-Turn:
  Background Review (Hermes-style)
    → Extract corrections and preferences
    → Write to short-term store (not directly to MEMORY.md)
    → Create draft skills from patterns

Periodic (Cron):
  Dreaming Sweep (OpenClaw-style)
    → Score short-term entries using recall signals
    → Promote to MEMORY.md only if score > threshold
    → Consolidate draft skills into class-level skills

  Curator (Hermes-style)
    → Lifecycle transitions on skills
    → Archive unused skills
    → Never auto-delete

Always-On:
  Active Memory (OpenClaw-style)
    → Proactive recall before each reply
    → Hybrid search (vector + keyword)

  Memory Tool (Hermes-style)
    → Direct user-initiated writes
    → Threat scanning
    → Frozen snapshot for cache
```

---
---

# Part II: Comprehensive All-Perspective Comparison

---

## 15. Multi-Agent / Delegation

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Mechanism** | sessions_spawn / sessions_send tools | delegate_task tool (ThreadPoolExecutor) |
| **Child isolation** | Separate session, own transcript | Fresh conversation, own task_id, restricted toolset |
| **Parent visibility** | Parent sees child result via session events | Parent gets summary result only (no child reasoning) |
| **Recursion** | Subagent depth configurable (spawnDepth) | Recursive delegation blocked (delegate_task in BLOCKED list) |
| **Budget** | Independent per-session (no shared limit) | Independent IterationBudget (50 per child vs 90 parent) |
| **Concurrency** | Async (event-driven, non-blocking) | ThreadPoolExecutor (parallel batch mode available) |
| **Blocked tools** | Configurable via inheritedToolDeny | Hardcoded: delegate_task, clarify, memory, send_message |
| **Batch mode** | N/A (spawn multiple sessions) | Explicit parallel batch (concurrent.futures) |
| **Kanban board** | N/A | Yes (plugins/kanban — multi-agent task dispatcher) |
| **Communication** | sessions_send (cross-session messaging) | N/A (children return result, no cross-talk) |

### Hermes Delegation Specifics

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",     # no recursive delegation
    "clarify",           # no user interaction
    "memory",            # no writes to shared MEMORY.md
    "send_message",      # no cross-platform side effects
])
```

---

## 16. Skill System

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Skill format** | Static files (SKILL.md) — loaded from disk | SKILL.md + references/ + templates/ + scripts/ + assets/ |
| **Creation** | Manual (developer creates skill files) | Agent-created via `skill_manage` tool (autonomous) |
| **Discovery** | Plugin SDK registration | Progressive disclosure (metadata → full content → linked files) |
| **Lifecycle** | Static (no state transitions) | Active → Stale (30d) → Archived (90d) |
| **Maintenance** | Manual | Curator (periodic, auto-consolidates, archives stale) |
| **Provenance** | N/A | Tracked (is_agent_created flag distinguishes human vs agent skills) |
| **Security** | N/A | skills_guard scanner (exfiltration, injection, destructive, obfuscation) |
| **Hub** | N/A (no marketplace) | skills_hub (install from remote repositories) |
| **Pinning** | N/A | Pinned skills bypass auto-transitions |
| **Skill categories** | Flat | Hierarchical directories (category/skill-name/SKILL.md) |
| **Dynamic loading** | N/A | Skills injected into system prompt per-session based on relevance |

### Hermes Skill Lifecycle State Machine

```
[CREATED] ──(30 days no use)──→ [STALE] ──(90 days no use)──→ [ARCHIVED]
     ↑                              ↑
     │ (agent creates)              │ (curator reviews)
     │                              │
  [PINNED] ─── bypasses all auto-transitions
```

---

## 17. Security Model

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Tool execution** | Sandboxed (workspace root guards, append-only mode) | Approval-based (dangerous command detection + user confirmation) |
| **Dangerous commands** | N/A (sandbox prevents escape) | Pattern matching (DANGEROUS_PATTERNS list + LLM auto-approval) |
| **Memory writes** | No scanning mentioned | Threat pattern scanning (injection, exfiltration, persistence, obfuscation) |
| **Skill installation** | N/A | skills_guard scanner (safe/caution/dangerous verdicts) |
| **Prompt injection** | N/A | Explicit detection in memory_tool + skills_guard + cron system |
| **Auth model** | Device tokens, TLS pinning, OAuth, password | API keys in .env file |
| **Gateway security** | WebSocket auth (device token, shared secret, bootstrap) | Allowed users list per platform |
| **Credential storage** | SecretRef (env/file/exec — never plaintext in config) | .env file (plaintext) or Bitwarden plugin |
| **Network security** | TLS cert pinning, proxy support, MITM detection | CA bundle env vars (HERMES_CA_BUNDLE, SSL_CERT_FILE) |
| **Zero-width chars** | Not checked | Explicitly blocked in memory writes (_INVISIBLE_CHARS set) |
| **Jailbreak detection** | N/A | Pattern matching (DAN, dev_mode, hypothetical_bypass, etc.) |
| **Approval hooks** | N/A | Plugin hooks (pre_approval_request, post_approval_response) |
| **Smart approval** | N/A | Auxiliary LLM auto-approves low-risk commands |
| **Per-session state** | Connection-scoped auth | Per-session allowlist (permanentAllowList in config.yaml) |

---

## 18. Voice / Speech / Realtime

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Architecture** | Full voice subsystem (Talk sessions) | Tool-based (voice_mode, tts_tool, transcription) |
| **Modes** | realtime, stt-tts, transcription | Voice mode (streaming STT + TTS) |
| **Transports** | WebRTC, provider-websocket, gateway-relay, managed-room | Direct API calls |
| **Brain modes** | agent-consult, direct-tools, none | Agent processes voice like text (unified loop) |
| **Providers** | Multiple (ElevenLabs, Deepgram, Azure Speech) | OpenAI Whisper + TTS, NeuTTS |
| **Protocol** | 17 event types (session.started, turn.started, audio.delta, etc.) | Simple request/response |
| **Client support** | Native (iOS/Android/macOS apps support voice) | CLI only (voice_mode tool) |
| **VAD** | Configurable (vadThreshold, silenceDurationMs, prefixPaddingMs) | N/A |
| **Barge-in** | Supported (supportsBargeIn capability) | N/A |
| **Session management** | Full (create, join, close, resume) | Per-invocation |

---

## 19. Observability & Diagnostics

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Telemetry** | OpenTelemetry (diagnostics-otel plugin) + Prometheus | Langfuse plugin |
| **Logging** | Structured subsystem logging | Profile-scoped logs (agent.log, errors.log, gateway.log) |
| **Health checks** | `openclaw doctor` (memory, plugins, providers, config) | `hermes doctor` (config, credentials, tools) |
| **Usage analytics** | Session-level (tokens, cost per session) | InsightsEngine (token consumption, cost, tool patterns, trends) |
| **Diagnostics** | Event loop health monitoring (delayP99Ms, utilization) | N/A |
| **Tracing** | `/verbose on`, `/trace on` in sessions | Logging levels (--verbose, --debug) |
| **Cost tracking** | Provider-level usage tracking | Per-session estimated_cost_usd / actual_cost_usd in SQLite |
| **Status** | Gateway status (all channels, connections, accounts) | `hermes status` (processes, gateway, platforms) |

---

## 20. IDE / Editor Integration

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Protocol** | N/A (client apps connect via WebSocket) | ACP (Agent Control Plane) — VS Code / Zed / JetBrains |
| **Architecture** | Native apps as clients | acp_adapter/server.py (HTTP+JSON-RPC server) |
| **Session model** | Gateway sessions accessible from any client | SessionManager with edit approval workflow |
| **Edit approval** | N/A | EditProposal → user review → apply/reject |
| **LSP** | N/A | Built-in LSP client (agent/lsp/) for code intelligence |
| **Code actions** | N/A | Agent uses LSP for hover, references, diagnostics |

---

## 21. Deployment & Distribution

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Installation** | pnpm install, global CLI | pip/uv install, setup.py |
| **Packaging** | Node.js ESM package | Python wheel + Docker |
| **Docker** | Not emphasized | First-class (Dockerfile, docker-compose.yml) |
| **Self-update** | `openclaw update` (update-cli community, 99 symbols) | `hermes update` (git pull + uv pip install) |
| **Nix** | N/A | flake.nix + flake.lock for reproducible builds |
| **Termux** | N/A | constraints-termux.txt (Android support) |
| **System service** | Gateway as background process | systemd support (gateway + cron) |
| **Profiles** | Per-agent config directories | HERMES_HOME env var (profile switching) |

---

## 22. Client Applications

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **iOS** | Full native app (Swift, SwiftUI) | N/A |
| **Android** | Full native app (Kotlin, Compose) | N/A |
| **macOS** | Full native app (Swift) | N/A |
| **watchOS** | Watch extension (limited) | N/A |
| **Desktop UI** | Control UI (web) | N/A |
| **Terminal UI** | Ink-based TUI (integrated in gateway) | Ink-based TUI (ui-tui/ — separate Node process) |
| **CLI** | `openclaw send`, `openclaw gateway`, etc. | `hermes` CLI (Python, 11K LOC) |
| **Web chat** | Yes (webchat-ui client) | N/A (use API server adapter) |
| **API server** | Gateway IS the API server (WebSocket) | Separate APIServerAdapter (REST + WebSocket) |

---

## 23. Plugin System Deep Dive

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Plugin types** | Provider, Channel, Tool, Memory, Setup | Memory, Model-provider, Platform, Web, Image/Video gen, Observability |
| **Plugin SDK** | 50+ hooks, typed contracts, manifest-based | Simple Python modules (register hooks, provide config) |
| **Discovery** | Manifest (openclaw.plugin.json) | Config.yaml (memory.provider, model-providers list) |
| **Loading** | Lazy (only when needed, registry-based) | Eager (loaded at startup based on config) |
| **Hot reload** | N/A (restart for plugin changes) | N/A (restart for plugin changes) |
| **Marketplace** | ClawHub (planned) | Skills Hub (install from git repos) |
| **Testing** | Contract tests (plugin-sdk-package-contract) | pytest (per-plugin test files) |
| **Dependencies** | Plugin-local package.json | Shared requirements.txt / pyproject.toml |
| **Plugin count** | 129 bundled | ~30 bundled |
| **Configuration** | plugins.entries.<id>.config in openclaw.json | Per-plugin config keys in config.yaml |

---

## 24. Context Engine / RAG

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Architecture** | Context engine plugin slot | Built-in ContextEngine class (agent/context_engine.py) |
| **Implementation** | Plugin-provided (separate package) | Integrated with agent (auto-loads relevant context) |
| **File indexing** | N/A (memory system handles) | Workspace file scanning + relevance scoring |
| **References** | N/A | ContextReference + ContextReferenceResult classes |
| **Interaction with memory** | Separate systems | ContextEngine calls into memory for project-level context |

---

## 25. Batch Processing

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Architecture** | N/A (single-request model) | Dedicated batch_runner.py |
| **Concurrency** | N/A | ThreadPoolExecutor (configurable workers) |
| **Use cases** | N/A | Bulk file processing, parallel evaluations |
| **State** | N/A | Separate from main session DB |

---

## 26. Internationalization

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Docs i18n** | Mintlify i18n pipeline (translation memory, glossary) | README.zh-CN.md (manual translation) |
| **UI i18n** | Full (glossary-driven, multiple locales) | locales/ directory |
| **CJK search** | N/A (memory search uses standard tokenizer) | FTS5 trigram tokenizer for CJK substring matching |
| **Regional platforms** | Some (Feishu) | Extensive (DingTalk, WeCom, Weixin, QQBot, Yuanbao, Feishu) |

---

## 27. Testing Philosophy

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Framework** | Vitest | pytest |
| **Test count** | Not specified | ~17,000 tests |
| **Contract tests** | Yes (plugin-sdk contracts, embedding contracts) | N/A |
| **E2E tests** | scripts/e2e/ | tests/run_agent/ (full conversation loops) |
| **Benchmark** | Vitest benchmarks | N/A |
| **Isolation** | --isolate=false safe (clean mocks per test) | Thread-safe (ToolRegistry singleton with locks) |
| **Worker limit** | Max 16 workers | Standard pytest-xdist |

---

## 28. Unique Features (Only One Has It)

### OpenClaw Only

| Feature | Description |
|---------|-------------|
| **Native mobile apps** | iOS, Android, macOS, watchOS |
| **Device pairing** | Cryptographic device tokens, TLS pinning |
| **Dreaming system** | Automated 3-phase memory consolidation |
| **Commitments** | Inferred follow-up obligations delivered via heartbeat |
| **SOUL.md** | Dedicated personality file with behavioral weight |
| **Protocol versioning** | JSON-RPC v4, client/server feature negotiation |
| **Media generation** | Image, video, music generation as first-class capabilities |
| **Voice/Talk sessions** | Full real-time voice with WebRTC/WebSocket/managed-room |
| **Memory Wiki** | Compiled knowledge vault with claims, evidence, provenance |
| **Prompt cache optimization** | System-wide prefix cache awareness |

### Hermes Only

| Feature | Description |
|---------|-------------|
| **Agent-created skills** | Agent autonomously creates SKILL.md from experience |
| **Curator** | Periodic automated skill lifecycle management |
| **Background review** | Per-turn LLM-driven self-reflection (memory + skill) |
| **Skill security scanner** | Jailbreak, exfiltration, injection detection in skills |
| **Smart approval** | Auxiliary LLM auto-approves safe dangerous commands |
| **User.md separation** | Agent facts vs. user facts in separate files |
| **Kanban board** | Multi-agent task dispatcher with worker subagents |
| **Delegation tool** | Explicit subagent spawning with tool restrictions |
| **Batch runner** | Parallel bulk processing |
| **LSP integration** | Built-in Language Server Protocol client for code intelligence |
| **ACP adapter** | IDE integration (VS Code, Zed, JetBrains) |
| **Context engine** | Built-in RAG for workspace files |
| **Insights engine** | Historical analytics (cost, tokens, trends) |
| **Docker first-class** | Dockerfile + compose for containerized deployment |
| **Nix support** | Reproducible builds via flake.nix |
| **Termux support** | Android terminal support |
| **Credential pool** | Multi-key rotation with rate-limit tracking |
| **CJK search** | Trigram FTS5 for Chinese/Japanese/Korean |
| **Prompt injection scanning** | Memory + skill writes scanned for attacks |
| **Process management** | Background process tool with watch patterns and strike limits |
| **IterationBudget** | Thread-safe per-agent budget with refund for code execution |

---

## 29. Performance & Scalability

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Cold start** | ~240ms (lazy OpenAI SDK import), plugin loading | Faster (Python startup + lazy imports) |
| **Memory footprint** | Higher (Node.js + all plugins loaded) | Lower (Python, only active plugins) |
| **Concurrent sessions** | Gateway handles many (event-driven, async) | ThreadPoolExecutor (bounded parallelism) |
| **Context window** | Up to 1M tokens (model-dependent) | Up to 1M tokens (model-dependent) |
| **Prompt caching** | Deep optimization (Active Memory, context pruning, TTL) | Frozen snapshot (simple but effective) |
| **Tool execution** | Subprocess (isolation, overhead) | In-process threads (fast, no isolation) |
| **Embedding cost** | One-time index build + per-query embedding | External provider (per-query API call) |
| **Background work** | Cron service (separate process) | Daemon threads (in-process) |

---

## 30. Maturity & Community

| Aspect | OpenClaw | Hermes |
|--------|---------|--------|
| **Codebase age** | Mature (detailed plugin SDK, contract tests, protocol versioning) | Mature (~17K tests, extensive platform support) |
| **Documentation** | Mintlify docs site, extensive inline CLAUDE.md files | Docusaurus website, AGENTS.md, release notes |
| **Release cadence** | Versioned (v2026.5.x) | Numbered releases (v0.14.0) |
| **Contribution** | CODEOWNERS, PR review process | CONTRIBUTING.md, open source |
| **Plugin ecosystem** | 129 bundled plugins | ~30 plugins + skills hub |
| **Platform reach** | English-first (i18n pipeline) | Global (extensive China platform support) |
| **Target user** | Power user running a personal AI gateway | Developer/hacker wanting a coding assistant |

---

## 31. Summary: When to Choose Which

| Use Case | Better Choice | Why |
|----------|--------------|-----|
| Personal AI assistant (long-running) | OpenClaw | Dreaming + Active Memory + SOUL.md for persistent personality |
| Coding assistant that learns from corrections | Hermes | Skill creation + curator + background review |
| Multi-platform messaging bot | Either | Both support 20+ platforms |
| Mobile-first experience | OpenClaw | Native iOS/Android/macOS apps |
| IDE-integrated coding agent | Hermes | ACP adapter + LSP client + code context engine |
| China/Asia deployment | Hermes | DingTalk, WeCom, Weixin, QQBot, Yuanbao support |
| Voice/real-time conversation | OpenClaw | Full Talk subsystem with WebRTC |
| Self-hosted with minimal setup | Hermes | pip install + config.yaml (no gateway required) |
| Enterprise security requirements | OpenClaw | Device pairing, TLS pinning, protocol versioning |
| Prompt injection defense | Hermes | Explicit scanning in memory, skills, cron |
| Multi-agent orchestration | Hermes | Kanban board + delegate_task + batch runner |
| Maximum provider compatibility | Either | Both support 40+ providers |
| Reproducible deployment | Hermes | Nix + Docker first-class |
| Plugin development | OpenClaw | 50-hook SDK, contract tests, typed manifest |
