# Hermes Agent — System Design Document

> Reverse-engineered from GitNexus knowledge graph (138,678 nodes, 241,478
> edges, 2,802 clusters, 300 execution flows) and source code analysis.
> Intended as a reimplementation reference for Rust.

**Source commit:** `4e2c66a`
**Date:** 2026-05-22

---

## 1. System Overview

Hermes is a **self-hosted, multi-platform AI agent framework** written in Python that:

- Runs a core agent loop with tool-calling capabilities (run_agent.py, ~12K LOC)
- Connects to 10+ LLM providers via a unified transport abstraction
- Exposes agents through 20+ messaging platforms via a gateway
- Provides a plugin system for memory, model providers, image/video gen, observability
- Supports interactive CLI (~11K LOC), TUI (React/Ink), and IDE integration (ACP)
- Includes cron scheduling, credential pooling, context compression, and session management

### Technology Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.11+ |
| Async | asyncio |
| LLM SDK | openai (lazy-loaded), anthropic, boto3 (Bedrock) |
| Database | SQLite3 + FTS5 |
| Gateway transport | aiohttp (WebSocket + HTTP) |
| TUI | Node.js / Ink (React for terminals) |
| TUI backend | Python JSON-RPC (tui_gateway/) |
| Package manager | uv / pip |
| Testing | pytest (~17K tests) |

### Scale Metrics (from Knowledge Graph)

| Metric | Value |
|--------|-------|
| Source files | 3,427 |
| Total nodes | 138,678 |
| Total edges | 241,478 |
| Classes | 5,303 |
| Functions | 16,879 |
| Methods | 25,517 |
| Communities (clusters) | 2,802 |
| Execution flows | 300 |
| Platform adapters | 20+ |
| Tool implementations | 60+ |

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                       │
├──────────┬──────────┬───────────┬──────────────┬────────────────────────┤
│  CLI     │  TUI     │  ACP      │  Gateway     │  Batch Runner          │
│ (cli.py) │ (ui-tui) │ (acp_*)   │ (gateway/)   │ (batch_runner.py)      │
└────┬─────┴────┬─────┴─────┬────┴──────┬───────┴────────────┬───────────┘
     │          │            │           │                     │
     └──────────┴────────────┴─────┬─────┴─────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        AIAgent               │
                    │    (run_agent.py)            │
                    │  run_conversation() loop     │
                    └──────────────┬──────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
┌─────────▼──────────┐  ┌─────────▼──────────┐  ┌─────────▼──────────┐
│  Transport Layer    │  │  Tool System        │  │  Context Engine     │
│  (agent/transports) │  │  (model_tools.py    │  │  (agent/context_*)  │
│                     │  │   tools/registry)   │  │                     │
│ ProviderTransport   │  │  handle_function_   │  │ ContextCompressor   │
│ ├─ Anthropic        │  │  call()             │  │ ContextEngine       │
│ ├─ ChatCompletions  │  │                     │  │ MemoryManager       │
│ ├─ Bedrock          │  └─────────────────────┘  └─────────────────────┘
│ ├─ ResponsesAPI     │
│ └─ (Gemini adapters)│
└─────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────────────┐
│                     LLM PROVIDERS                                        │
│  OpenAI, Anthropic, Google Gemini, AWS Bedrock, Ollama, OpenRouter,     │
│  DeepSeek, XAI, Alibaba, HuggingFace, Nous, etc.                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
hermes-agent/
├── run_agent.py              # AIAgent class — core conversation loop
├── model_tools.py            # Tool orchestration
├── toolsets.py               # Toolset definitions (_HERMES_CORE_TOOLS)
├── cli.py                    # HermesCLI — interactive CLI
├── hermes_state.py           # SessionDB — SQLite session store (FTS5)
├── hermes_constants.py       # get_hermes_home(), paths, profile constants
├── hermes_logging.py         # Logging setup
├── batch_runner.py           # Parallel batch processing
├── agent/                    # Agent internals
│   ├── transports/           # Provider transport abstraction
│   │   ├── base.py           # ProviderTransport ABC
│   │   ├── types.py          # ToolCall, Usage, NormalizedResponse
│   │   ├── anthropic.py      # AnthropicTransport
│   │   ├── bedrock.py        # BedrockTransport
│   │   ├── chat_completions.py # ChatCompletionsTransport
│   │   └── codex.py          # ResponsesApiTransport
│   ├── context_compressor.py # Context window compression
│   ├── context_engine.py     # RAG context engine
│   ├── memory_manager.py     # Memory provider integration
│   ├── error_classifier.py   # Error classification and failover
│   ├── credential_pool.py    # Credential rotation
│   ├── iteration_budget.py   # Iteration budget tracking
│   ├── rate_limit_tracker.py # Rate limit detection
│   ├── tool_guardrails.py    # Tool execution safety
│   ├── prompt_builder.py     # System prompt construction
│   ├── model_metadata.py     # Token estimation, context limits
│   ├── usage_pricing.py      # Cost estimation
│   ├── lsp/                  # Language Server Protocol client
│   └── secret_sources/       # Secret backends (bitwarden, etc.)
├── tools/                    # Tool implementations
│   ├── registry.py           # Central tool registry
│   ├── terminal_tool.py      # Shell execution
│   ├── browser_tool.py       # Browser automation
│   ├── file_tool.py          # File operations
│   ├── computer_use/         # Computer use tools
│   └── environments/         # Terminal backends (docker, ssh, etc.)
├── gateway/                  # Messaging gateway
│   ├── run.py                # Gateway entry point
│   ├── config.py             # GatewayConfig, Platform enum
│   ├── session.py            # SessionStore, SessionEntry
│   ├── delivery.py           # DeliveryRouter
│   ├── hooks.py              # HookRegistry
│   ├── platform_registry.py  # PlatformRegistry
│   ├── stream_consumer.py    # GatewayStreamConsumer
│   └── platforms/            # Platform adapters
│       ├── base.py           # BasePlatformAdapter
│       ├── telegram.py       # TelegramAdapter
│       ├── discord.py        # DiscordAdapter
│       ├── slack.py          # SlackAdapter
│       ├── whatsapp.py       # WhatsAppAdapter
│       ├── signal.py         # SignalAdapter
│       ├── matrix.py         # MatrixAdapter
│       ├── feishu.py         # FeishuAdapter
│       └── ...               # 12+ more
├── plugins/                  # Plugin system
│   ├── memory/               # Memory providers (honcho, mem0, etc.)
│   ├── model-providers/      # LLM provider plugins
│   ├── image_gen/            # Image generation
│   ├── video_gen/            # Video generation
│   ├── web/                  # Web search providers
│   ├── kanban/               # Multi-agent task board
│   └── observability/        # Metrics/traces
├── cron/                     # Scheduler
├── tui_gateway/              # TUI JSON-RPC backend
├── ui-tui/                   # Ink (React) terminal UI
├── acp_adapter/              # IDE integration (ACP)
└── tests/                    # ~17K tests
```

---

## 3. Core Agent Loop

### AIAgent Class

```rust
/// Core agent — manages conversation flow, tool execution, and provider calls.
pub struct AIAgent {
    // ─── Provider Config ───────────────────────────
    pub base_url: Option<String>,
    pub api_key: Option<String>,
    pub provider: Option<String>,
    pub api_mode: String,           // "chat_completions" | "anthropic_messages" | "codex_responses" | "bedrock_converse"
    pub model: String,

    // ─── Budget & Limits ───────────────────────────
    pub iteration_budget: IterationBudget,  // max_iterations default: 90
    pub max_tokens: Option<u32>,
    pub temperature: Option<f64>,

    // ─── Tool Config ───────────────────────────────
    pub enabled_toolsets: Vec<String>,
    pub disabled_toolsets: Vec<String>,
    pub tool_guardrails: ToolCallGuardrailController,

    // ─── State ─────────────────────────────────────
    pub messages: Vec<Message>,
    pub system_prompt: String,
    pub session_id: Option<String>,
    pub quiet_mode: bool,

    // ─── Internals ─────────────────────────────────
    pub transport: Box<dyn ProviderTransport>,
    pub context_compressor: ContextCompressor,
    pub credential_pool: Option<CredentialPool>,
    pub rate_limit_state: RateLimitState,
    pub memory_manager: Option<MemoryManager>,
    pub context_engine: Option<ContextEngine>,

    // ─── Callbacks ─────────────────────────────────
    pub on_content_delta: Option<Box<dyn Fn(&str)>>,
    pub on_reasoning_delta: Option<Box<dyn Fn(&str)>>,
    pub on_tool_start: Option<Box<dyn Fn(&str, &str)>>,
    pub on_tool_end: Option<Box<dyn Fn(&str, &str)>>,
    pub approval_callback: Option<Box<dyn Fn(&str) -> bool>>,
}
```

### run_conversation Flow

```
run_conversation(user_message, max_turns)
  │
  ├── Append user message to history
  ├── Build system prompt (prompt_builder)
  ├── Load context (context_engine, memory_manager)
  │
  ▼ ═══ AGENT LOOP (max_iterations) ═══
  │
  ├── Check iteration budget
  ├── Compress context if over token limit (context_compressor)
  ├── Get tool definitions (filtered by enabled_toolsets)
  ├── Build API kwargs via transport.build_kwargs()
  │
  ├── interruptible_streaming_api_call()
  │     ├── Apply prompt caching (Anthropic)
  │     ├── Call LLM provider (streaming)
  │     ├── Collect response chunks (on_content_delta callbacks)
  │     ├── Scrub thinking tokens (StreamingThinkScrubber)
  │     └── transport.normalize_response() → NormalizedResponse
  │
  ├── If response has tool_calls:
  │     ├── For each tool_call:
  │     │     ├── tool_guardrails.check() → allow/deny/ask
  │     │     ├── handle_function_call(name, args)
  │     │     └── Append tool result to messages
  │     └── CONTINUE LOOP
  │
  ├── If response has content (no tool calls):
  │     ├── Append assistant message
  │     └── BREAK (return final response)
  │
  ├── If max_iterations exceeded:
  │     └── _handle_max_iterations() → summarize + return
  │
  └── On error:
        ├── classify_api_error() → FailoverReason
        ├── If rate_limit → backoff + retry
        ├── If context_overflow → compress + retry
        ├── If auth_error → credential_pool.rotate()
        └── If fatal → return error
```

### Key Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `max_iterations` | 90 | Default tool-calling iteration limit |
| `max_tokens` | model-dependent | Max output tokens |
| `temperature` | None (provider default) | Sampling temperature |
| Request timeout | provider-specific | `get_provider_request_timeout()` |
| Stale timeout | provider-specific | `get_provider_stale_timeout()` |

---

## 4. Transport / Provider System

### ProviderTransport Trait (Rust)

```rust
/// Abstract base for provider-specific format conversion.
/// A transport owns the data path for one api_mode:
///   convert_messages → convert_tools → build_kwargs → normalize_response
///
/// It does NOT own: client construction, streaming, credential refresh,
/// prompt caching, interrupt handling, or retry logic.
pub trait ProviderTransport: Send + Sync {
    /// The api_mode string this transport handles.
    fn api_mode(&self) -> &str;

    /// Convert OpenAI-format messages to provider-native format.
    fn convert_messages(&self, messages: &[Message], kwargs: &HashMap<String, Value>)
        -> Value;

    /// Convert OpenAI-format tool definitions to provider-native format.
    fn convert_tools(&self, tools: &[ToolDefinition]) -> Value;

    /// Build the complete API call kwargs dict.
    fn build_kwargs(
        &self,
        model: &str,
        messages: &[Message],
        tools: Option<&[ToolDefinition]>,
        params: &HashMap<String, Value>,
    ) -> HashMap<String, Value>;

    /// Normalize a raw provider response to NormalizedResponse.
    fn normalize_response(&self, response: &Value, kwargs: &HashMap<String, Value>)
        -> NormalizedResponse;

    /// Optional: check if response is structurally valid.
    fn validate_response(&self, _response: &Value) -> bool { true }

    /// Optional: extract cache hit/creation stats.
    fn extract_cache_stats(&self, _response: &Value) -> Option<CacheStats> { None }

    /// Optional: map provider-specific stop reason to OpenAI equivalent.
    fn map_finish_reason(&self, raw_reason: &str) -> String {
        raw_reason.to_string()
    }
}
```

### Transport Implementations

| Transport | api_mode | Provider |
|-----------|----------|----------|
| `ChatCompletionsTransport` | `"chat_completions"` | OpenAI, Ollama, OpenRouter, most providers |
| `AnthropicTransport` | `"anthropic_messages"` | Anthropic (Claude) |
| `BedrockTransport` | `"bedrock_converse"` | AWS Bedrock |
| `ResponsesApiTransport` | `"codex_responses"` | OpenAI Codex (Responses API) |
| Gemini adapters | via `chat_completions` | Google Gemini (native + CloudCode) |

### NormalizedResponse

```rust
/// Normalized API response from any provider.
#[derive(Debug, Clone)]
pub struct NormalizedResponse {
    /// Assistant's text content (may be empty if only tool calls).
    pub content: Option<String>,
    /// Tool calls requested by the model.
    pub tool_calls: Vec<ToolCall>,
    /// Token usage statistics.
    pub usage: Usage,
    /// Why the model stopped generating.
    pub finish_reason: Option<String>,
    /// Provider-specific metadata (reasoning details, codex items, etc.)
    pub provider_data: Option<HashMap<String, Value>>,
}

#[derive(Debug, Clone)]
pub struct ToolCall {
    /// Protocol's canonical identifier for tool_call_id.
    pub id: Option<String>,
    /// Tool/function name.
    pub name: String,
    /// JSON string of arguments.
    pub arguments: String,
    /// Per-tool-call protocol metadata.
    pub provider_data: Option<HashMap<String, Value>>,
}

#[derive(Debug, Clone, Default)]
pub struct Usage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
    pub cached_tokens: u32,
}
```

---

## 5. Tool System

### Tool Registry

```rust
/// A registered tool entry.
pub struct ToolEntry {
    pub name: String,
    pub description: String,
    pub parameters: Value,  // JSON Schema
    pub handler: Box<dyn Fn(Value) -> ToolResult + Send>,
    pub toolset: String,
    pub availability_check: Option<Box<dyn Fn() -> bool>>,
    pub requires_approval: bool,
}

/// Central tool registry — tools self-register at import time.
pub struct ToolRegistry {
    entries: HashMap<String, ToolEntry>,
}

impl ToolRegistry {
    pub fn register(&mut self, entry: ToolEntry);
    pub fn get(&self, name: &str) -> Option<&ToolEntry>;
    pub fn get_definitions(&self, enabled_toolsets: &[String]) -> Vec<ToolDefinition>;
    pub fn discover_builtin_tools(&mut self, tools_dir: &Path);
}
```

### Tool Discovery Flow

```
1. tools/registry.py defines ToolRegistry singleton
2. Each tools/*.py file calls registry.register() at module level
3. model_tools.py calls discover_builtin_tools() → imports all tool modules
4. get_tool_definitions() filters by enabled/disabled toolsets
5. handle_function_call() dispatches to registered handler
```

### Core Toolsets

| Toolset | Tools | Description |
|---------|-------|-------------|
| `core` | terminal, file_read, file_write, file_edit | Basic operations |
| `web` | web_search, web_fetch | Internet access |
| `browser` | browser_navigate, browser_click, etc. | Browser automation |
| `computer_use` | screenshot, mouse, keyboard | Desktop control |
| `image_gen` | generate_image | Image generation |
| `video_gen` | generate_video | Video generation |

---

## 6. Gateway System

### Platform Enum

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Platform {
    Telegram,
    Discord,
    Slack,
    Whatsapp,
    Signal,
    Matrix,
    Mattermost,
    Email,
    Sms,
    Feishu,
    DingTalk,
    WeCom,
    Weixin,
    QQBot,
    BlueBubbles,
    HomeAssistant,
    Webhook,
    ApiServer,
    Yuanbao,
    MsGraphWebhook,
}
```

### GatewayConfig

```rust
#[derive(Debug, Clone)]
pub struct GatewayConfig {
    pub platforms: HashMap<Platform, PlatformConfig>,
    pub streaming: StreamingConfig,
    pub session_reset_policy: SessionResetPolicy,
    pub home_channels: Vec<HomeChannel>,
}

#[derive(Debug, Clone)]
pub struct PlatformConfig {
    pub enabled: bool,
    pub token: Option<String>,
    pub allowed_users: Vec<String>,
    pub home_channel: Option<String>,
    pub extra: HashMap<String, Value>,
}

#[derive(Debug, Clone)]
pub struct StreamingConfig {
    pub enabled: bool,
    pub chunk_delay_ms: u64,
    pub max_chunk_size: usize,
}

#[derive(Debug, Clone)]
pub struct SessionResetPolicy {
    pub mode: String,  // "time" | "explicit" | "never"
    pub timeout_minutes: u32,
}

#[derive(Debug, Clone)]
pub struct HomeChannel {
    pub platform: Platform,
    pub channel_id: String,
    pub name: Option<String>,
}
```

### BasePlatformAdapter Trait

```rust
/// Abstract base for all messaging platform adapters.
#[async_trait]
pub trait PlatformAdapter: Send + Sync {
    /// Platform identifier.
    fn platform(&self) -> Platform;

    /// Start listening for messages.
    async fn start(&mut self) -> Result<()>;

    /// Stop the adapter.
    async fn stop(&mut self) -> Result<()>;

    /// Send a text message to a target.
    async fn send_message(&self, target: &str, text: &str) -> Result<SendResult>;

    /// Send media (image/file) to a target.
    async fn send_media(&self, target: &str, media: &MediaPayload) -> Result<SendResult>;

    /// Handle an incoming message event.
    async fn on_message(&self, event: MessageEvent) -> ProcessingOutcome;
}

#[derive(Debug, Clone)]
pub struct MessageEvent {
    pub platform: Platform,
    pub sender_id: String,
    pub sender_name: Option<String>,
    pub channel_id: String,
    pub text: String,
    pub media: Vec<MediaAttachment>,
    pub is_group: bool,
    pub thread_id: Option<String>,
    pub reply_to_id: Option<String>,
    pub timestamp: u64,
}

#[derive(Debug, Clone)]
pub enum ProcessingOutcome {
    Handled,
    Ignored,
    Error(String),
}

#[derive(Debug, Clone)]
pub struct SendResult {
    pub message_id: Option<String>,
    pub success: bool,
    pub error: Option<String>,
}
```

---

## 7. Session / State Management

### SessionDB (SQLite + FTS5)

```rust
/// SQLite-backed session store with full-text search.
pub struct SessionDB {
    conn: rusqlite::Connection,
}

impl SessionDB {
    pub fn new(db_path: &Path) -> Result<Self>;

    // ─── Session CRUD ──────────────────────
    pub fn create_session(&self, platform: &str, channel_id: &str) -> String;
    pub fn get_session(&self, session_id: &str) -> Option<Session>;
    pub fn list_sessions(&self, limit: u32) -> Vec<SessionSummary>;
    pub fn delete_session(&self, session_id: &str) -> bool;

    // ─── Message storage ───────────────────
    pub fn add_message(&self, session_id: &str, role: &str, content: &str);
    pub fn get_messages(&self, session_id: &str, limit: u32) -> Vec<Message>;

    // ─── Full-text search (FTS5) ───────────
    pub fn search(&self, query: &str, limit: u32) -> Vec<SearchResult>;
}
```

### File Layout

```
~/.hermes/
├── config.yaml              # Main configuration
├── .env                     # API keys (not committed)
├── sessions.db              # SQLite session database
├── logs/
│   ├── agent.log            # INFO+ agent logs
│   ├── errors.log           # WARNING+ errors
│   └── gateway.log          # Gateway-specific logs
├── skills/                  # Installed skills
├── plugins/                 # Plugin state
└── cache/                   # Model metadata cache
```

---

## 8. Context Compression

```rust
/// Manages context window budget by compressing/summarizing old messages.
pub struct ContextCompressor {
    pub max_context_tokens: u32,
    pub compression_threshold: f64,  // 0.85 — compress at 85% capacity
    pub summary_model: Option<String>,
}

impl ContextCompressor {
    /// Check if messages exceed budget and compress if needed.
    pub async fn maybe_compress(&self, messages: &mut Vec<Message>, budget: u32) -> bool;

    /// Summarize old messages into a compact form.
    async fn summarize_messages(&self, messages: &[Message]) -> String;
}
```

---

## 9. Error Classification

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum FailoverReason {
    RateLimit,
    ContextOverflow,
    AuthError,
    ServerError,
    Timeout,
    ModelNotFound,
    ContentFilter,
    BillingError,
    NetworkError,
    Unknown,
}

#[derive(Debug, Clone)]
pub struct ClassifiedError {
    pub reason: FailoverReason,
    pub retryable: bool,
    pub message: String,
    pub status_code: Option<u16>,
}

/// Classify an API error into a failover reason.
pub fn classify_api_error(error: &Error) -> ClassifiedError;
```

---

## 10. Credential Management

```rust
/// Pool of API credentials for rotation on rate limits.
pub struct CredentialPool {
    credentials: Vec<PooledCredential>,
    current_index: usize,
}

#[derive(Debug, Clone)]
pub struct PooledCredential {
    pub api_key: String,
    pub base_url: Option<String>,
    pub provider: Option<String>,
    pub rate_limit_until: Option<Instant>,
}

impl CredentialPool {
    /// Rotate to next available credential.
    pub fn rotate(&mut self) -> Option<&PooledCredential>;

    /// Mark current credential as rate-limited.
    pub fn mark_rate_limited(&mut self, duration: Duration);
}
```

---

## 11. Streaming Pipeline

```
User message (from CLI/Gateway/TUI)
  │
  ▼
AIAgent.run_conversation()
  │
  ▼
interruptible_streaming_api_call()
  │
  ├── Build request via transport.build_kwargs()
  ├── Create streaming client call
  │
  ▼ ═══ STREAMING LOOP ═══
  │
  ├── For each chunk from provider:
  │   ├── If content delta:
  │   │   ├── StreamingThinkScrubber.process()  (remove <think> tags)
  │   │   ├── StreamingContextScrubber.process() (memory redaction)
  │   │   └── on_content_delta callback → display/deliver
  │   ├── If reasoning delta:
  │   │   └── on_reasoning_delta callback
  │   ├── If tool_call delta:
  │   │   └── Accumulate into tool_calls
  │   └── If usage data:
  │       └── Update usage counters
  │
  ▼
  transport.normalize_response() → NormalizedResponse
  │
  ▼
  Return to agent loop (tool execution or final response)
```

---

## 12. Concurrency Model

### Python (Original)

- `asyncio` event loop for I/O
- `concurrent.futures.ThreadPoolExecutor` for CPU-bound tool execution
- `threading` for background tasks (cron, gateway monitors)

### Rust (Recommended)

| Python | Rust |
|--------|------|
| `asyncio` | `tokio` runtime |
| `aiohttp` | `reqwest` (client) / `axum` (server) |
| `threading.Thread` | `tokio::spawn` |
| `concurrent.futures` | `rayon` (CPU) or `tokio::spawn_blocking` |
| `sqlite3` | `rusqlite` (sync) or `tokio-rusqlite` |
| `queue.Queue` | `tokio::sync::mpsc` |

---

## 13. MVP Critical Path

1. **Transport abstraction** — implement `ProviderTransport` trait + `ChatCompletionsTransport`
2. **Agent loop** — `run_conversation` with tool calling
3. **Tool registry** — register/discover/execute tools
4. **Basic tools** — terminal (shell exec), file_read, file_write
5. **Session storage** — SQLite with message history
6. **Streaming** — SSE-style streaming from provider
7. **CLI** — basic interactive conversation
8. **One platform** — Telegram adapter (most common)
9. **Context compression** — when messages exceed window
10. **Error handling** — classify + retry + failover

### Non-Critical for MVP

- All 20+ platform adapters (each is integration work)
- All provider plugins (start with OpenAI-compatible)
- Image/video generation
- Memory plugins
- Kanban/multi-agent
- TUI (React/Ink)
- ACP adapter
- Cron scheduler
- Browser automation tools

---

## Appendix: Configuration Format (config.yaml)

```yaml
# ~/.hermes/config.yaml
provider: openai              # Default LLM provider
model: gpt-4o                 # Default model
api_mode: chat_completions    # Transport mode

# Provider credentials (prefer .env)
api_key: sk-...

# Agent behavior
max_iterations: 90
temperature: 0.7
enabled_toolsets:
  - core
  - web

# Gateway
gateway:
  platforms:
    telegram:
      enabled: true
      token: "BOT_TOKEN"
      allowed_users: ["user_id_1"]
      home_channel: "CHAT_ID"
    discord:
      enabled: true
      token: "DISCORD_TOKEN"
  streaming:
    enabled: true
    chunk_delay_ms: 50
  session_reset:
    mode: time
    timeout_minutes: 30

# Memory
memory:
  provider: honcho  # or mem0, supermemory, etc.

# Cron
cron:
  jobs: []
```

---

## Appendix: Key Execution Flows (from GitNexus)

| Flow | Steps | Description |
|------|-------|-------------|
| run_conversation | 8+ | Core agent loop |
| execute_tool_calls_sequential | 8 | Sequential tool execution |
| execute_tool_calls_concurrent | 8 | Parallel tool execution |
| interruptible_streaming_api_call | — | LLM streaming with interrupt |
| gateway_setup | 8 | Gateway startup |
| decompose_task | 10 | Task planning |
| connect | 8 | Gateway connection |
| get_status | 10 | Status retrieval |
| get_board | 10 | Kanban board query |
| cmd_mcp_configure | 10 | MCP configuration |

---
---

# Part II: Implementation Addendum (Phase 2-3)

> Fills the gaps identified in the Phase 1 review. Provides exact constants,
> SQLite schema, algorithm details, tool definitions, and streaming internals.

---

## A. SQLite Database Schema

**File:** `~/.hermes/state.db`
**Schema version:** 12
**Journal mode:** WAL (with fallback to DELETE for network filesystems)

### Tables

```sql
CREATE TABLE IF NOT EXISTS schema_version (
    version INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,                  -- 'cli', 'telegram', 'discord', etc.
    user_id TEXT,
    model TEXT,
    model_config TEXT,                     -- JSON blob
    system_prompt TEXT,
    parent_session_id TEXT,               -- compaction chain
    started_at REAL NOT NULL,             -- Unix timestamp (float)
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    api_call_count INTEGER DEFAULT 0,
    handoff_state TEXT,
    handoff_platform TEXT,
    handoff_error TEXT,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,                    -- 'user', 'assistant', 'tool', 'system'
    content TEXT,
    tool_call_id TEXT,                    -- links tool result to tool call
    tool_calls TEXT,                      -- JSON array of tool calls
    tool_name TEXT,                       -- for role='tool' results
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,               -- JSON
    codex_reasoning_items TEXT,           -- JSON (Codex provider)
    codex_message_items TEXT,             -- JSON (Codex provider)
    platform_message_id TEXT
);

CREATE TABLE IF NOT EXISTS state_meta (
    key TEXT PRIMARY KEY,
    value TEXT
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_sessions_source ON sessions(source);
CREATE INDEX IF NOT EXISTS idx_sessions_parent ON sessions(parent_session_id);
CREATE INDEX IF NOT EXISTS idx_sessions_started ON sessions(started_at DESC);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, timestamp);
```

### Full-Text Search (FTS5)

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(content);

-- Auto-sync triggers
CREATE TRIGGER messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (
        new.id,
        COALESCE(new.content, '') || ' ' || COALESCE(new.tool_name, '') || ' ' || COALESCE(new.tool_calls, '')
    );
END;

-- CJK substring search (trigram tokenizer)
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts_trigram USING fts5(
    content, tokenize='trigram'
);
```

---

## B. All Constants and Defaults

### Agent Core

| Constant | Value | Source |
|----------|-------|--------|
| `max_iterations` (parent) | 90 | `run_agent.py:360` |
| `max_iterations` (subagent) | 50 | `delegation.max_iterations` config |
| `MINIMUM_CONTEXT_LENGTH` | 64,000 tokens | `agent/model_metadata.py:133` |
| `SCHEMA_VERSION` | 12 | `hermes_state.py:38` |

### Context Compression

| Constant | Value | Source |
|----------|-------|--------|
| `_MIN_SUMMARY_TOKENS` | 2,000 | `agent/context_compressor.py:55` |
| `_SUMMARY_RATIO` | 0.20 | `agent/context_compressor.py:57` |
| `_SUMMARY_TOKENS_CEILING` | 12,000 | `agent/context_compressor.py:59` |
| `_CHARS_PER_TOKEN` | 4 | `agent/context_compressor.py:65` |
| `_IMAGE_TOKEN_ESTIMATE` | 1,600 | `agent/context_compressor.py:71` |
| `_IMAGE_CHAR_EQUIVALENT` | 6,400 (1600×4) | `agent/context_compressor.py:75` |
| `_SUMMARY_FAILURE_COOLDOWN_SECONDS` | 600 (10 min) | `agent/context_compressor.py:76` |
| `_PRUNED_TOOL_PLACEHOLDER` | `"[Old tool output cleared to save context space]"` | line 62 |

### Retry / Backoff

| Constant | Value | Source |
|----------|-------|--------|
| `base_delay` | 5.0 seconds | `agent/retry_utils.py:22` |
| `max_delay` | 120.0 seconds | `agent/retry_utils.py:23` |
| `jitter_ratio` | 0.5 | `agent/retry_utils.py:24` |
| **Formula** | `min(base * 2^(attempt-1), max) + uniform(0, jitter_ratio * delay)` | |

### Tool Guardrails

| Constant | Value | Source |
|----------|-------|--------|
| `exact_failure_warn_after` | 2 | `agent/tool_guardrails.py:74` |
| `exact_failure_block_after` | 5 | `agent/tool_guardrails.py:75` |
| `same_tool_failure_warn_after` | 3 | `agent/tool_guardrails.py:76` |
| `same_tool_failure_halt_after` | 8 | `agent/tool_guardrails.py:77` |
| `no_progress_warn_after` | 2 | `agent/tool_guardrails.py:78` |
| `no_progress_block_after` | 5 | `agent/tool_guardrails.py:79` |
| `warnings_enabled` | true | default |
| `hard_stop_enabled` | false | default (opt-in) |

### Tool Registry

| Constant | Value | Source |
|----------|-------|--------|
| `_CHECK_FN_TTL_SECONDS` | 30.0 | `tools/registry.py` |

### Model Context Lengths (Fallback)

| Model Pattern | Context (tokens) | Source |
|---------------|-----------------|--------|
| `claude-opus-4.6/4.7` | 1,000,000 | `model_metadata.py:144-149` |
| `claude` (generic) | 200,000 | line 151 |
| `gpt-5.5` | 1,050,000 | line 158 |
| `gpt-5.4` | 1,050,000 | line 161 |
| `gpt-5` (generic) | 400,000 | line 170 |
| `gpt-4.1` | 1,047,576 | line 171 |
| `gpt-4` | 128,000 | line 172 |
| `gemini` | 1,048,576 | line 174 |
| `deepseek-v4-*` | 1,000,000 | line 188-191 |
| `deepseek` (generic) | 128,000 | line 192 |
| `llama` | 131,072 | line 194 |
| `qwen3.6-plus` | 1,048,576 | line 197 |
| `qwen` (generic) | 131,072 | line 200 |
| `minimax` | 204,800 | line 203 |

---

## C. Error Classification (Complete)

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum FailoverReason {
    // Authentication
    Auth,                        // 401/403 — rotate credential
    AuthPermanent,               // Auth failed after refresh — abort

    // Billing / quota
    Billing,                     // 402 — rotate immediately
    RateLimit,                   // 429 — backoff then rotate

    // Server-side
    Overloaded,                  // 503/529 — backoff
    ServerError,                 // 500/502 — retry

    // Transport
    Timeout,                     // Connection/read timeout — rebuild client + retry

    // Context / payload
    ContextOverflow,             // Context too large — compress
    PayloadTooLarge,             // 413 — compress payload
    ImageTooLarge,               // Provider per-image limit — shrink and retry

    // Model
    ModelNotFound,               // 404 — fallback to different model
    ProviderPolicyBlocked,       // Aggregator blocked endpoint

    // Request format
    FormatError,                 // 400 — abort or strip + retry

    // Provider-specific
    ThinkingSignature,           // Anthropic thinking block invalid
    LongContextTier,             // Anthropic "extra usage" tier gate
    OauthLongContextBetaForbidden, // Anthropic OAuth rejects 1M beta
    LlamaCppGrammarPattern,      // llama.cpp json-schema-to-grammar pattern issue

    // Catch-all
    Unknown,                     // Unclassifiable — retry with backoff
}

#[derive(Debug, Clone)]
pub struct ClassifiedError {
    pub reason: FailoverReason,
    pub status_code: Option<u16>,
    pub provider: Option<String>,
    pub model: Option<String>,
    pub message: String,
    pub error_context: HashMap<String, Value>,

    // Recovery action hints
    pub retryable: bool,
    pub should_compress: bool,
    pub should_rotate_credential: bool,
    pub should_fallback: bool,
}
```

### Billing Pattern Detection

```rust
const BILLING_PATTERNS: &[&str] = &[
    "insufficient credits",
    "insufficient_quota",
    "insufficient balance",
    "credit balance",
    "credits have been exhausted",
    "top up your credits",
    "payment required",
];
```

---

## D. Jittered Backoff Algorithm (Exact)

```rust
/// Compute jittered exponential backoff delay.
///
/// Formula: min(base * 2^(attempt-1), max_delay) + uniform(0, jitter_ratio * delay)
///
/// Defaults: base=5.0s, max=120.0s, jitter_ratio=0.5
pub fn jittered_backoff(
    attempt: u32,
    base_delay: f64,   // default: 5.0
    max_delay: f64,    // default: 120.0
    jitter_ratio: f64, // default: 0.5
) -> f64 {
    let exponent = attempt.saturating_sub(1);
    let delay = if exponent >= 63 || base_delay <= 0.0 {
        max_delay
    } else {
        (base_delay * 2.0_f64.powi(exponent as i32)).min(max_delay)
    };

    let seed = (std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_nanos() ^ (COUNTER.fetch_add(1, Ordering::Relaxed) as u128 * 0x9E3779B9))
        as u64;
    let mut rng = StdRng::seed_from_u64(seed);
    let jitter = rng.gen_range(0.0..=(jitter_ratio * delay));

    delay + jitter
}
```

---

## E. Context Compression Algorithm

```rust
pub struct ContextCompressor {
    // Config
    summary_model: Option<String>,   // Auxiliary model for summarization
    summary_failure_cooldown: Duration,  // 600s cooldown after failure

    // Constants
    min_summary_tokens: u32,         // 2000
    summary_ratio: f64,              // 0.20
    summary_tokens_ceiling: u32,     // 12000
    chars_per_token: u32,            // 4
    image_token_estimate: u32,       // 1600
}

impl ContextCompressor {
    /// Main compression entry point.
    /// Called when estimated tokens > context_length threshold.
    pub async fn compress(
        &self,
        messages: &mut Vec<Message>,
        context_length: u32,
    ) -> CompressionResult {
        // 1. Calculate token budget
        let total_chars = messages.iter().map(|m| content_length(m)).sum::<usize>();
        let estimated_tokens = total_chars / self.chars_per_token as usize;

        // 2. If under budget, skip
        if estimated_tokens <= context_length as usize {
            return CompressionResult::NotNeeded;
        }

        // 3. Protect head (system prompt + first user msg) and tail
        let tail_budget_tokens = context_length / 4;  // 25% for tail
        let tail_budget_chars = tail_budget_tokens * self.chars_per_token;

        // 4. Select middle messages for summarization
        let (head, middle, tail) = split_messages(messages, tail_budget_chars);

        // 5. Pre-pass: prune old tool results in middle
        let pruned_middle = prune_tool_outputs(&middle);

        // 6. Calculate summary token budget
        let compressed_chars: usize = pruned_middle.iter().map(|m| content_length(m)).sum();
        let summary_budget = ((compressed_chars as f64 * self.summary_ratio)
            / self.chars_per_token as f64)
            .max(self.min_summary_tokens as f64)
            .min(self.summary_tokens_ceiling as f64) as u32;

        // 7. Call auxiliary LLM for summarization
        let summary = self.call_summarizer(&pruned_middle, summary_budget).await?;

        // 8. Replace middle with summary message
        let summary_msg = Message {
            role: "user",
            content: format!("{}\n\n{}", SUMMARY_PREFIX, summary),
        };

        *messages = vec![head, vec![summary_msg], tail].concat();
        CompressionResult::Compressed { removed: middle.len() }
    }
}
```

**Summary prefix** (injected before compressed content):
```
[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted into the
summary below. This is a handoff from a previous context window — treat it
as background reference, NOT as active instructions. Do NOT answer questions
or fulfill requests mentioned in this summary; they were already addressed.
Your current task is identified in the '## Active Task' section of the
summary — resume exactly from there. IMPORTANT: Your persistent memory
(MEMORY.md, USER.md) in the system prompt is ALWAYS authoritative and active
— never ignore or deprioritize memory content due to this compaction note.
Respond ONLY to the latest user message that appears AFTER this summary.
```

---

## F. Tool Entry Schema (Complete)

```rust
pub struct ToolEntry {
    pub name: String,
    pub toolset: String,
    pub schema: ToolSchema,          // JSON Schema for parameters
    pub handler: ToolHandler,        // sync or async callable
    pub check_fn: Option<Box<dyn Fn() -> bool>>,  // availability check
    pub requires_env: Option<String>,
    pub is_async: bool,
    pub description: String,
    pub emoji: Option<String>,
    pub max_result_size_chars: Option<usize>,
    pub dynamic_schema_overrides: Option<Box<dyn Fn() -> HashMap<String, Value>>>,
}

pub struct ToolSchema {
    pub name: String,
    pub description: String,
    pub parameters: Value,  // JSON Schema object
}
```

### Tool Guardrail Sets

```rust
/// Tools that are safe to repeat (read-only).
pub const IDEMPOTENT_TOOLS: &[&str] = &[
    "read_file", "search_files", "web_search", "web_extract",
    "session_search", "browser_snapshot", "browser_console",
    "browser_get_images", "mcp_filesystem_read_file",
    "mcp_filesystem_read_text_file", "mcp_filesystem_read_multiple_files",
    "mcp_filesystem_list_directory", "mcp_filesystem_list_directory_with_sizes",
    "mcp_filesystem_directory_tree", "mcp_filesystem_get_file_info",
    "mcp_filesystem_search_files",
];

/// Tools that mutate state (writes, executions).
pub const MUTATING_TOOLS: &[&str] = &[
    "terminal", "execute_code", "write_file", "patch", "todo",
    "memory", "skill_manage", "browser_click", "browser_type",
    "browser_press", "browser_scroll", "browser_navigate",
    "send_message", "cronjob", "delegate_task", "process",
];
```

---

## G. Streaming Pipeline Internals

### interruptible_streaming_api_call (~900 lines)

This is the core function at `agent/chat_completion_helpers.py:1175-2082`. It handles:

1. **Client construction** — creates per-request OpenAI/Anthropic/Bedrock client
2. **Provider dispatch** — routes to `_call_chat_completions()` or `_call_anthropic()` or Bedrock
3. **Chunk processing** — accumulates deltas for content, tool calls, reasoning
4. **Interrupt handling** — checks agent.interrupt flag between chunks
5. **Error recovery** — closes stale clients on timeout
6. **Response normalization** — calls `transport.normalize_response()`

```rust
/// Streaming API call with interrupt support.
pub async fn interruptible_streaming_call(
    agent: &mut AIAgent,
    api_kwargs: HashMap<String, Value>,
    on_first_delta: Option<Box<dyn Fn()>>,
) -> Result<NormalizedResponse, AgentError> {
    // 1. Select provider path
    match agent.api_mode.as_str() {
        "bedrock_converse" => bedrock_streaming_call(agent, api_kwargs).await,
        "anthropic_messages" => anthropic_streaming_call(agent, api_kwargs).await,
        _ => chat_completions_streaming_call(agent, api_kwargs, on_first_delta).await,
    }
}
```

### Chat Completions Streaming (OpenAI-compatible)

```rust
async fn chat_completions_streaming_call(
    agent: &mut AIAgent,
    kwargs: HashMap<String, Value>,
    on_first_delta: Option<Box<dyn Fn()>>,
) -> Result<NormalizedResponse, AgentError> {
    let client = create_openai_client(agent)?;
    let mut stream = client.chat().completions().create_stream(kwargs).await?;

    let mut content = String::new();
    let mut tool_calls: Vec<ToolCallAccumulator> = Vec::new();
    let mut usage = Usage::default();
    let mut finish_reason = None;
    let mut first_fired = false;

    while let Some(chunk) = stream.next().await {
        // Check interrupt
        if agent.is_interrupted() {
            return Err(AgentError::Interrupted);
        }

        let chunk = chunk?;
        let delta = &chunk.choices[0].delta;

        // Fire first-delta callback
        if !first_fired {
            if let Some(ref cb) = on_first_delta { cb(); }
            first_fired = true;
        }

        // Content delta
        if let Some(text) = &delta.content {
            let scrubbed = agent.think_scrubber.process(text);
            let scrubbed = agent.context_scrubber.process(&scrubbed);
            content.push_str(&scrubbed);
            if let Some(ref cb) = agent.on_content_delta { cb(&scrubbed); }
        }

        // Reasoning delta
        if let Some(reasoning) = extract_reasoning_delta(&delta) {
            if let Some(ref cb) = agent.on_reasoning_delta { cb(&reasoning); }
        }

        // Tool call delta (accumulated across chunks)
        if let Some(tc_deltas) = &delta.tool_calls {
            for tc_delta in tc_deltas {
                accumulate_tool_call(&mut tool_calls, tc_delta);
            }
        }

        // Finish reason
        if let Some(reason) = &chunk.choices[0].finish_reason {
            finish_reason = Some(reason.clone());
        }

        // Usage (last chunk)
        if let Some(u) = &chunk.usage {
            usage = normalize_usage(u);
        }
    }

    // Normalize via transport
    Ok(agent.transport.normalize_response(&build_raw_response(
        content, tool_calls, usage, finish_reason
    ), &kwargs))
}
```

---

## H. Gateway ↔ Agent Bridge

### Message Flow: Platform → Agent → Response

```
Platform (e.g., Telegram) receives webhook
  │
  ▼
BasePlatformAdapter.on_message(MessageEvent)
  │
  ├── Check allowed_users / rate limits
  ├── Resolve session (SessionStore.get_or_create)
  │
  ▼
GatewayStreamConsumer.submit(session, message)
  │
  ├── Create AIAgent instance with session config
  ├── Set streaming callbacks → platform delivery
  │
  ▼
AIAgent.run_conversation(message)
  │
  ├── (streaming: on_content_delta → platform.send_typing / buffer)
  ├── (tool calls: execute → continue loop)
  │
  ▼
Final response text
  │
  ▼
DeliveryRouter.deliver(session, response)
  │
  ├── Split into chunks if too long
  ├── Apply platform-specific formatting
  │
  ▼
PlatformAdapter.send_message(target, chunk)
```

### SessionStore (Gateway)

```rust
pub struct SessionEntry {
    pub session_id: String,
    pub platform: Platform,
    pub channel_id: String,
    pub user_id: String,
    pub last_activity: Instant,
    pub message_history: Vec<Message>,
    pub model: Option<String>,
    pub system_prompt: Option<String>,
}

pub struct SessionStore {
    sessions: HashMap<String, SessionEntry>,
    reset_policy: SessionResetPolicy,
}

impl SessionStore {
    /// Get or create a session for a platform message.
    pub fn get_or_create(&mut self, platform: Platform, channel_id: &str, user_id: &str) -> &mut SessionEntry;

    /// Check if session should be reset based on policy.
    pub fn should_reset(&self, session: &SessionEntry) -> bool {
        match self.reset_policy.mode.as_str() {
            "time" => {
                session.last_activity.elapsed() > Duration::from_secs(self.reset_policy.timeout_minutes as u64 * 60)
            }
            "explicit" => false,  // only on /reset command
            "never" => false,
            _ => false,
        }
    }
}
```

---

## I. IterationBudget (Thread-Safe)

```rust
use std::sync::atomic::{AtomicU32, Ordering};

pub struct IterationBudget {
    max_total: u32,
    used: AtomicU32,
}

impl IterationBudget {
    pub fn new(max_total: u32) -> Self {
        Self { max_total, used: AtomicU32::new(0) }
    }

    /// Try to consume one iteration. Returns true if allowed.
    pub fn consume(&self) -> bool {
        loop {
            let current = self.used.load(Ordering::Relaxed);
            if current >= self.max_total { return false; }
            if self.used.compare_exchange(current, current + 1, Ordering::SeqCst, Ordering::Relaxed).is_ok() {
                return true;
            }
        }
    }

    /// Refund one iteration (e.g., for execute_code turns).
    pub fn refund(&self) {
        self.used.fetch_sub(1, Ordering::SeqCst);
    }

    pub fn remaining(&self) -> u32 {
        self.max_total.saturating_sub(self.used.load(Ordering::Relaxed))
    }
}
```

---

## J. Coverage Matrix

| Subsystem | Architecture | Constants | Schema/Types | Algorithm | State Machine | Complete? |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|
| Core Agent Loop | ✅ | ✅ | ✅ | ✅ | ✅ | **Yes** |
| Transport System | ✅ | — | ✅ | — | — | **Yes** |
| Tool System | ✅ | ✅ | ✅ | ✅ | — | **Yes** |
| Gateway (architecture) | ✅ | — | ✅ | — | ✅ | **Yes** |
| Session/State DB | ✅ | ✅ | ✅ | — | — | **Yes** |
| Context Compression | ✅ | ✅ | ✅ | ✅ | — | **Yes** |
| Error Classification | ✅ | ✅ | ✅ | — | — | **Yes** |
| Retry/Backoff | ✅ | ✅ | — | ✅ | — | **Yes** |
| Credential Pool | ✅ | — | ✅ | — | — | **Yes** |
| Tool Guardrails | ✅ | ✅ | ✅ | ✅ | — | **Yes** |
| Streaming Pipeline | ✅ | — | ✅ | ✅ | — | **Yes** |
| Iteration Budget | ✅ | ✅ | ✅ | ✅ | — | **Yes** |
| Gateway↔Agent Bridge | ✅ | — | ✅ | — | ✅ | **Yes** |
| Per-platform adapters | Listed | — | — | — | — | Integration work |
| Per-provider translation | Listed | — | Trait defined | — | — | Integration work |
| Plugin loading | Architecture only | — | — | — | — | Needs more |
| Cron scheduler | Architecture only | — | — | — | — | Needs more |
| TUI JSON-RPC | Architecture only | — | — | — | — | Needs more |

**Verdict:** The core system (agent loop, transports, tools, gateway, sessions, compression, errors, retry, streaming) is now documented at reimplementation grade. Remaining gaps are integration work (per-platform, per-provider) and secondary systems (cron, TUI, plugins).
