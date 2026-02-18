# ZeroClaw Architecture Documentation

This document provides a comprehensive, self-contained guide to the ZeroClaw
architecture. It is written for developers who may be new to Rust, async
programming, or agent-based systems. Every diagram is ASCII so you can read
it in any terminal or text editor.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Architecture Diagram](#2-high-level-architecture-diagram)
3. [Trait-Driven Design Pattern](#3-trait-driven-design-pattern)
4. [Module Map](#4-module-map)
5. [Component Deep Dives](#5-component-deep-dives)
   - 5.1 [Agent Orchestration Loop](#51-agent-orchestration-loop)
   - 5.2 [Providers (LLM Backends)](#52-providers-llm-backends)
   - 5.3 [Channels (Messaging Platforms)](#53-channels-messaging-platforms)
   - 5.4 [Tools (Agent Capabilities)](#54-tools-agent-capabilities)
   - 5.5 [Memory System](#55-memory-system)
   - 5.6 [Security Module](#56-security-module)
   - 5.7 [Gateway (HTTP Server)](#57-gateway-http-server)
   - 5.8 [Observability](#58-observability)
   - 5.9 [Runtime Adapters](#59-runtime-adapters)
   - 5.10 [Peripherals (Hardware)](#510-peripherals-hardware)
6. [Communication Flow Diagrams](#6-communication-flow-diagrams)
   - 6.1 [CLI Interactive Chat Flow](#61-cli-interactive-chat-flow)
   - 6.2 [Gateway Webhook Flow](#62-gateway-webhook-flow)
   - 6.3 [Channel Listener Flow](#63-channel-listener-flow)
   - 6.4 [Daemon Mode Flow](#64-daemon-mode-flow)
   - 6.5 [Tool Execution Flow](#65-tool-execution-flow)
   - 6.6 [Memory Recall Flow (Hybrid Search)](#66-memory-recall-flow-hybrid-search)
   - 6.7 [Security Validation Flow](#67-security-validation-flow)
   - 6.8 [Pairing and Authentication Flow](#68-pairing-and-authentication-flow)
7. [Data Flow Between Subsystems](#7-data-flow-between-subsystems)
8. [Configuration System](#8-configuration-system)
9. [Build and Release Architecture](#9-build-and-release-architecture)
10. [Testing Architecture](#10-testing-architecture)

---

## 1. System Overview

ZeroClaw is an **autonomous AI agent runtime** written entirely in Rust. It
connects a large language model (LLM) to the real world through messaging
channels, tools, memory, and hardware peripherals. The design optimises for:

- **Minimal resource usage**: ~3.4 MB binary, <5 MB RAM, <10 ms startup
- **Security by default**: sandboxed execution, encrypted secrets, pairing
- **Full pluggability**: every subsystem is behind a Rust trait

Key numbers at a glance:

```
Binary size ......... ~3.4 MB (release, stripped)
Startup time ........ <10 ms
RAM footprint ....... <5 MB typical
Test count .......... 1,017 unit + integration tests
Providers ........... 23+ LLM backends
Channels ............ 13+ messaging platforms
Tools ............... 27+ capabilities
Memory backends ..... 4 (SQLite, Lucid, Markdown, None)
```

---

## 2. High-Level Architecture Diagram

This diagram shows every major subsystem and how they connect at the
highest level. Data flows from users (left) through channels into the
agent loop (centre), which calls providers, tools, and memory.

```
                          ZeroClaw High-Level Architecture
 ============================================================================

  USERS / EXTERNAL                         CORE ENGINE
  ==================                       ==========================

  +-------------+                          +-------------------------+
  | Terminal    |----(stdin/stdout)-------->|                         |
  | (CLI)       |                          |     Agent Loop          |
  +-------------+                          |     (src/agent/)        |
                                           |                         |
  +-------------+     +----------+         |  1. Receive message     |
  | Telegram   |---->| Channel  |-------->|  2. Load memory context |
  +-------------+    | Listener |         |  3. Build system prompt |
  | Discord    |---->| (async   |-------->|  4. Call LLM provider   |
  +-------------+    |  mpsc    |         |  5. Parse tool calls    |
  | Slack      |---->|  sender) |-------->|  6. Execute tools       |
  +-------------+     +----------+         |  7. Feed results back   |
  | WhatsApp   |                           |  8. Repeat until done   |
  +-------------+                          |  9. Send response       |
  | Matrix     |                           |                         |
  +-------------+                          +-----+-------+-----------+
  | IRC, Email |                                 |       |
  +-------------+                                |       |
                                                 v       v
  +-------------+     +----------+         +-----+---+ +-+----------+
  | HTTP Client|---->| Gateway  |-------->|Provider | |  Tools     |
  | (webhook)  |     | (Axum)   |         |(LLM API)| |(27+ caps) |
  +-------------+     +----------+         +---------+ +-----+------+
                           |                     |           |
                           |               +-----v-----------v------+
  +-------------+          |               |       Memory           |
  | Hardware   |<-------- Peripheral ----->| (SQLite + FTS5 + Vec)  |
  | (STM32,    |          Module           +------------------------+
  |  RPi, ESP) |
  +-------------+                          +------------------------+
                                           |     Security Policy    |
                                           | (sandboxing, rates,    |
                                           |  secrets, pairing)     |
                                           +------------------------+

                                           +------------------------+
                                           |    Observability       |
                                           | (Log, OTel, Prometheus)|
                                           +------------------------+
```

---

## 3. Trait-Driven Design Pattern

ZeroClaw uses Rust **traits** as stable extension points. A trait defines
the interface; concrete structs implement it. A **factory function** in
each module's `mod.rs` picks the right implementation at startup based on
the configuration.

This pattern means you can add a new LLM provider, messaging channel, or
tool without modifying any other module.

```
  TRAIT PATTERN (used by every subsystem)
  ========================================

  +--------------------+        +------------------------+
  |   Trait Definition |        |  Factory Function      |
  |   (traits.rs)      |        |  (mod.rs)              |
  +--------------------+        +------------------------+
  | trait Provider {   |        | fn create_provider(    |
  |   fn chat(...)     |        |     config: &Config    |
  |   fn capabilities()|        | ) -> Box<dyn Provider> |
  | }                  |        | {                      |
  +--------------------+        |   match name {         |
          ^                     |     "openai" => ...    |
          |                     |     "anthropic" => ... |
          |  implements         |     "ollama" => ...    |
          |                     |   }                    |
  +-------+----------+         | }                      |
  | OpenAiProvider    |         +------------------------+
  | AnthropicProvider |
  | OllamaProvider    |
  | GeminiProvider    |
  | ...               |
  +-------------------+
```

### The Eight Core Traits

```
  +-----------------------------------------------------------+
  |                  CORE TRAIT MAP                            |
  +-----------------------------------------------------------+
  |                                                           |
  |  src/providers/traits.rs   -->  Provider                  |
  |    Methods: chat(), chat_with_system(),                   |
  |             chat_with_history(), capabilities(),           |
  |             supports_native_tools(), warmup()              |
  |                                                           |
  |  src/channels/traits.rs    -->  Channel                   |
  |    Methods: name(), send(), listen(),                     |
  |             health_check(), start_typing()                 |
  |                                                           |
  |  src/tools/traits.rs       -->  Tool                      |
  |    Methods: name(), description(),                        |
  |             parameters_schema(), execute()                 |
  |                                                           |
  |  src/memory/traits.rs      -->  Memory                    |
  |    Methods: store(), recall(), get(),                     |
  |             list(), forget(), count()                      |
  |                                                           |
  |  src/observability/traits.rs -> Observer                  |
  |    Methods: on_event()                                    |
  |                                                           |
  |  src/runtime/traits.rs     -->  RuntimeAdapter            |
  |    Methods: execute_shell()                               |
  |                                                           |
  |  src/peripherals/traits.rs -->  Peripheral                |
  |    Methods: name(), init(), tools()                       |
  |                                                           |
  |  src/security/traits.rs    -->  Sandbox                   |
  |    Methods: create(), destroy(), execute()                |
  |                                                           |
  +-----------------------------------------------------------+
```

---

## 4. Module Map

Every file in `src/` and what it does:

```
  src/
  +-- main.rs ........................ CLI entrypoint (clap), command routing
  +-- lib.rs ......................... Library root, module re-exports
  |
  +-- config/
  |   +-- schema.rs .................. All config structs (Config, AutonomyConfig, etc.)
  |   +-- mod.rs ..................... Config loading, merging, env overrides
  |
  +-- agent/
  |   +-- agent.rs ................... Agent struct: holds provider, memory, tools
  |   +-- loop_.rs ................... Main agentic loop (tool call cycle)
  |   +-- dispatcher.rs .............. Routes tool calls to implementations
  |   +-- prompt.rs .................. System prompt assembly (identity + skills + memory)
  |   +-- memory_loader.rs ........... Retrieves relevant memories for context
  |   +-- tests.rs ................... Agent-level tests
  |
  +-- providers/
  |   +-- traits.rs .................. Provider trait + ChatMessage/Response types
  |   +-- mod.rs ..................... Provider factory (create_provider)
  |   +-- openai.rs .................. OpenAI GPT models
  |   +-- anthropic.rs ............... Claude/Anthropic API
  |   +-- ollama.rs .................. Local Ollama server
  |   +-- openrouter.rs .............. OpenRouter (23+ models)
  |   +-- gemini.rs .................. Google Gemini
  |   +-- glm.rs ..................... Zhipu GLM (China)
  |   +-- copilot.rs ................. GitHub Copilot
  |   +-- openai_codex.rs ............ OpenAI subscription auth
  |   +-- compatible.rs .............. Generic OpenAI-compatible endpoints
  |   +-- reliable.rs ................ Retry/circuit-breaker wrapper
  |   +-- router.rs .................. Provider selection & fallback
  |
  +-- channels/
  |   +-- traits.rs .................. Channel trait + message types
  |   +-- mod.rs ..................... Channel factory
  |   +-- cli.rs ..................... Terminal stdin/stdout
  |   +-- telegram.rs ................ Telegram Bot API (polling)
  |   +-- discord.rs ................. Discord WebSocket gateway
  |   +-- slack.rs ................... Slack Bot/Web API
  |   +-- whatsapp.rs ................ WhatsApp Business Cloud API
  |   +-- mattermost.rs .............. Mattermost REST API
  |   +-- matrix.rs .................. Matrix protocol
  |   +-- email_channel.rs ........... Email via SMTP/IMAP
  |   +-- signal.rs .................. Signal protocol bridge
  |   +-- irc.rs ..................... IRC network
  |   +-- imessage.rs ................ Apple iMessage
  |   +-- dingtalk.rs ................ DingTalk (Alibaba)
  |   +-- lark.rs .................... Lark/Feishu (ByteDance)
  |   +-- qq.rs ...................... Tencent QQ
  |
  +-- tools/
  |   +-- traits.rs .................. Tool trait + ToolResult/ToolSpec types
  |   +-- mod.rs ..................... Tool registry factory
  |   +-- shell.rs ................... Shell command execution
  |   +-- file_read.rs ............... Read files (path-safe)
  |   +-- file_write.rs .............. Write files (path-safe)
  |   +-- memory_store.rs ............ Store a memory entry
  |   +-- memory_recall.rs ........... Recall memories (hybrid search)
  |   +-- memory_forget.rs ........... Delete memory entries
  |   +-- browser.rs ................. Browser automation
  |   +-- browser_open.rs ............ Open URL (allowlist-gated)
  |   +-- screenshot.rs .............. Take screenshots
  |   +-- http_request.rs ............ HTTP GET/POST/PUT/DELETE
  |   +-- git_operations.rs .......... Git status/log/commit/push
  |   +-- delegate.rs ................ Delegate to sub-agents
  |   +-- composio.rs ................ 1000+ OAuth apps via Composio
  |   +-- image_info.rs .............. Image metadata analysis
  |   +-- cron_*.rs .................. Scheduled task management
  |   +-- hardware_*.rs .............. Hardware board operations
  |   +-- pushover.rs ................ Push notifications
  |
  +-- memory/
  |   +-- traits.rs .................. Memory trait + MemoryEntry types
  |   +-- mod.rs ..................... Memory backend factory
  |   +-- sqlite.rs .................. SQLite (FTS5 keyword + vector cosine)
  |   +-- lucid.rs ................... Lucid bridge with SQLite fallback
  |   +-- markdown.rs ................ File-based markdown storage
  |   +-- none.rs .................... No-op backend (stateless)
  |   +-- vector.rs .................. Vector similarity (cosine) search
  |   +-- embeddings.rs .............. Embedding providers (OpenAI, noop)
  |   +-- chunker.rs ................. Markdown text chunking
  |   +-- hygiene.rs ................. Memory retention policies
  |   +-- snapshot.rs ................ Memory export/import
  |   +-- response_cache.rs .......... Embedding cache layer
  |
  +-- security/
  |   +-- traits.rs .................. Sandbox trait
  |   +-- policy.rs .................. SecurityPolicy, AutonomyLevel, rate limits
  |   +-- pairing.rs ................. One-time pairing code + bearer token
  |   +-- secrets.rs ................. ChaCha20-Poly1305 encrypted secret store
  |   +-- audit.rs ................... Audit logging
  |   +-- landlock.rs ................ Linux Landlock sandbox
  |   +-- firejail.rs ................ Firejail sandbox
  |   +-- docker.rs .................. Docker sandbox
  |   +-- bubblewrap.rs .............. Bubblewrap sandbox
  |   +-- detect.rs .................. Auto-detect best sandbox
  |
  +-- gateway/
  |   +-- mod.rs ..................... Axum HTTP server (webhooks, pairing)
  |
  +-- runtime/
  |   +-- traits.rs .................. RuntimeAdapter trait
  |   +-- native.rs .................. Native OS shell execution
  |   +-- docker.rs .................. Docker container execution
  |   +-- wasm.rs .................... WASM (planned, errors on use)
  |   +-- mod.rs ..................... Runtime factory
  |
  +-- observability/
  |   +-- traits.rs .................. Observer trait
  |   +-- noop.rs .................... No-op observer
  |   +-- log.rs ..................... Structured tracing logs
  |   +-- verbose.rs ................. Verbose console output
  |   +-- multi.rs ................... Composite (multiple observers)
  |   +-- otel.rs .................... OpenTelemetry OTLP exporter
  |   +-- prometheus.rs .............. Prometheus metrics endpoint
  |   +-- mod.rs ..................... Observer factory
  |
  +-- peripherals/
  |   +-- traits.rs .................. Peripheral trait
  |   +-- rpi.rs ..................... Raspberry Pi GPIO
  |   +-- serial.rs .................. Serial port communication
  |   +-- arduino_*.rs ............... Arduino boards
  |   +-- nucleo_*.rs ................ STM32 Nucleo boards
  |   +-- mod.rs ..................... Peripheral factory
  |
  +-- cron/ .......................... Scheduled task engine
  +-- heartbeat/ ..................... Periodic autonomous tasks
  +-- auth/ .......................... Multi-provider auth profiles
  +-- onboard/ ....................... Interactive setup wizard
  +-- daemon/ ........................ Background service mode
  +-- doctor/ ........................ System diagnostics
  +-- service/ ....................... OS service management
  +-- health/ ........................ Health check system
  +-- cost/ .......................... API cost tracking
  +-- approval/ ...................... Approval workflow for risky actions
  +-- tunnel/ ........................ Tunnel integration (ngrok, etc.)
  +-- skills/ ........................ User-defined skill loading
  +-- integration/ ................... 50+ integration registry
  +-- identity.rs .................... AIEOS identity format support
  +-- migration.rs ................... OpenClaw -> ZeroClaw migration
  +-- util.rs ........................ Common utilities
  +-- rag/ ........................... RAG (PDF extraction, chunking)
```

---

## 5. Component Deep Dives

### 5.1 Agent Orchestration Loop

The agent loop is the brain of ZeroClaw. It sits in `src/agent/loop_.rs`
and implements a standard **ReAct** (Reason + Act) pattern:

```
  AGENT LOOP (src/agent/loop_.rs)
  ================================

  User Message
       |
       v
  +----+----+
  | Load    |  <-- memory_loader.rs: recall relevant memories
  | Context |      prompt.rs: build system prompt from
  +---------+      IDENTITY.md + SOUL.md + skills + memory
       |
       v
  +----+----+
  | Call    |  <-- providers/: send messages + tools to LLM
  | LLM     |      Uses ChatRequest { messages, tools }
  +---------+
       |
       v
  +----+-------+
  | Has tool   |---No---> Return text response to user
  | calls?     |
  +----+-------+
       | Yes
       v
  +----+-------+
  | Execute    |  <-- dispatcher.rs: route each tool call
  | tools      |      to the right Tool implementation
  +----+-------+      (with security policy checks)
       |
       v
  +----+-------+
  | Feed       |  <-- ToolResultMessage appended to history
  | results    |
  | back       |
  +----+-------+
       |
       v
  +----+-------+
  | Iteration  |---Under limit---> Back to "Call LLM"
  | limit?     |
  +----+-------+
       | Over limit
       v
  Return last LLM text
```

**Key file references:**

- `src/agent/loop_.rs:92` - `tools_to_openai_format()` converts tools to
  OpenAI function-calling JSON
- `src/agent/loop_.rs:43` - `scrub_credentials()` redacts secrets from
  tool output before sending to LLM
- `src/agent/loop_.rs:21` - `DEFAULT_MAX_TOOL_ITERATIONS = 10`

### 5.2 Providers (LLM Backends)

**Trait definition:** `src/providers/traits.rs:230`

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String>;

    async fn chat(&self, request: ChatRequest<'_>,
                  model: &str, temperature: f64,
    ) -> anyhow::Result<ChatResponse>;

    fn capabilities(&self) -> ProviderCapabilities;
    fn supports_native_tools(&self) -> bool;
    // ... streaming, warmup, etc.
}
```

**Provider communication:**

```
  PROVIDER DATA FLOW
  ==================

  Agent Loop                     Provider                     External API
  ----------                     --------                     ------------

  ChatRequest {               +--+--------+--+
    messages: [...],          | Provider impl |
    tools: Some([...])  ---->| (e.g. OpenAI) |----HTTPS--->  api.openai.com
  }                           |               |               /v1/chat/completions
                              | 1. Convert    |
                              |    tools to   |  <---JSON---  { choices: [...],
                              |    API format  |                 tool_calls: [...] }
                              | 2. HTTP POST  |
                              | 3. Parse resp |
                              +--+--------+--+
                                 |
                                 v
                              ChatResponse {
                                text: Some("..."),
                                tool_calls: [ToolCall { id, name, args }]
                              }
```

**Provider types by API format:**

```
  +-- OpenAI-format providers --------+
  |  openai.rs                        |
  |  openrouter.rs                    |
  |  openai_codex.rs                  |
  |  compatible.rs (any OpenAI-like)  |
  +-----------------------------------+

  +-- Anthropic-format providers -----+
  |  anthropic.rs                     |
  +-----------------------------------+

  +-- Google-format providers --------+
  |  gemini.rs                        |
  +-----------------------------------+

  +-- Custom-format providers --------+
  |  glm.rs (Zhipu JWT auth)         |
  |  copilot.rs (GitHub Copilot)     |
  +-----------------------------------+

  +-- Local providers ----------------+
  |  ollama.rs (localhost:11434)      |
  +-----------------------------------+

  +-- Meta providers (wrappers) ------+
  |  reliable.rs (retry + circuit)    |
  |  router.rs  (selection + fallback)|
  +-----------------------------------+
```

### 5.3 Channels (Messaging Platforms)

**Trait definition:** `src/channels/traits.rs:48`

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;
    async fn health_check(&self) -> bool;
}
```

**Channel communication pattern:**

```
  CHANNEL LISTENER PATTERN
  ========================

  External Platform          Channel Impl          Agent Loop
  -----------------          ------------          ----------

  Telegram Server            telegram.rs
    |                          |
    | HTTP long-poll           |
    |<---getUpdates------------|
    |---new messages---------->|
    |                          |
    |                          | Parse update
    |                          | Check allowlist
    |                          | tx.send(ChannelMessage)
    |                          |                      |
    |                          |                 Process message
    |                          |                 Call LLM
    |                          |                 Execute tools
    |                          |                      |
    |                          |<---SendMessage--------|
    |<---sendMessage-----------|
    |                          |
```

**All 13+ channel implementations:**

```
  CHANNEL MAP
  ===========

  Real-time (polling/WebSocket):
    +-- cli.rs ............. Terminal stdin/stdout
    +-- telegram.rs ........ Telegram Bot API (long-poll)
    +-- discord.rs ......... Discord WebSocket gateway
    +-- slack.rs ........... Slack Web/Bot API
    +-- irc.rs ............. IRC protocol
    +-- matrix.rs .......... Matrix/Element

  Webhook-based (push):
    +-- whatsapp.rs ........ WhatsApp Business Cloud API
    +-- mattermost.rs ...... Mattermost API v4

  Email:
    +-- email_channel.rs ... SMTP send + polling receive

  Platform-specific:
    +-- signal.rs .......... Signal protocol bridge
    +-- imessage.rs ........ Apple iMessage (macOS)
    +-- dingtalk.rs ........ Alibaba DingTalk
    +-- lark.rs ............ Lark/Feishu (ByteDance)
    +-- qq.rs .............. Tencent QQ
```

### 5.4 Tools (Agent Capabilities)

**Trait definition:** `src/tools/traits.rs:22`

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
}
```

**Tool execution with security gate:**

```
  TOOL EXECUTION FLOW
  ====================

  LLM returns ToolCall { name: "shell", arguments: "{\"command\":\"ls\"}" }
       |
       v
  +----+-------+
  | Dispatcher |  <-- src/agent/dispatcher.rs
  +----+-------+
       |
       v
  +----+--------+
  | Find tool   |  Look up "shell" in tools registry
  | by name     |
  +----+--------+
       |
       v
  +----+--------+
  | Security    |  <-- SecurityPolicy.is_command_allowed("ls")
  | Policy      |      SecurityPolicy.is_path_allowed(...)
  | Check       |      SecurityPolicy.enforce_tool_operation(...)
  +----+--------+
       | Allowed
       v
  +----+--------+
  | tool.       |  <-- ShellTool.execute({"command":"ls"})
  | execute()   |
  +----+--------+
       |
       v
  ToolResult { success: true, output: "file1.txt\nfile2.rs", error: None }
       |
       v
  Scrub credentials from output (loop_.rs:43)
       |
       v
  Append to conversation as ToolResultMessage
```

**Tool categories:**

```
  TOOL REGISTRY MAP
  =================

  File I/O:
    file_read .......... Read file contents (path-scoped to workspace)
    file_write ......... Write/create files (path-scoped to workspace)

  Shell:
    shell .............. Execute commands (allowlist-gated)

  Memory:
    memory_store ....... Store entry (key + content + category)
    memory_recall ...... Recall entries (keyword + vector hybrid search)
    memory_forget ...... Delete an entry by key

  Web/Browser:
    browser ............ Full browser automation
    browser_open ....... Open URL (domain-allowlisted)
    http_request ....... HTTP GET/POST/PUT/DELETE
    screenshot ......... Take screenshots

  Git:
    git_operations ..... git status, log, diff, commit, push

  Scheduling:
    cron_add ........... Add cron-scheduled task
    cron_list .......... List scheduled tasks
    cron_remove ........ Remove scheduled task

  Hardware:
    hardware_board_info  Get board info
    hardware_memory_read Read board memory
    hardware_memory_map  Memory map dump

  Integration:
    composio ........... 1000+ OAuth apps
    delegate ........... Delegate task to sub-agent
    pushover ........... Send push notifications
```

### 5.5 Memory System

**Trait definition:** `src/memory/traits.rs:43`

ZeroClaw's memory system is a **complete search engine** with no external
dependencies. Everything runs inside a single SQLite file.

```
  MEMORY ARCHITECTURE (SQLite backend)
  =====================================

  +---------------------------------------------------+
  |                  SQLite Database                   |
  |                (~/.zeroclaw/brain.db)              |
  +---------------------------------------------------+
  |                                                   |
  |  +-------------------+   +---------------------+ |
  |  | memories table    |   | memories_fts (FTS5)  | |
  |  |                   |   | (virtual table)      | |
  |  | id (TEXT PK)      |   |                      | |
  |  | key (TEXT)        |   | Full-text index of   | |
  |  | content (TEXT)    |   | key + content fields | |
  |  | category (TEXT)   |   | BM25 scoring         | |
  |  | session_id (TEXT) |   +---------------------+ |
  |  | timestamp (TEXT)  |                            |
  |  | embedding (BLOB)  |   +---------------------+ |
  |  +-------------------+   | embedding_cache      | |
  |                          | (LRU eviction)       | |
  |                          +---------------------+ |
  +---------------------------------------------------+

  HYBRID SEARCH FLOW (memory_recall):
  ====================================

  Query: "What is the user's favourite language?"
       |
       +----------+----------+
       |                     |
       v                     v
  FTS5 keyword search   Vector cosine search
  (BM25 scoring)        (embedding similarity)
       |                     |
       v                     v
  Results + scores      Results + scores
       |                     |
       +----------+----------+
                  |
                  v
         Weighted merge (vector.rs)
         vector_weight * vec_score + keyword_weight * kw_score
                  |
                  v
         Top-N results by combined score
```

**Memory backends comparison:**

```
  +----------+----------+----------+----------+
  |          | SQLite   | Lucid    | Markdown |
  +----------+----------+----------+----------+
  | Search   | FTS5 +   | External | Filename |
  |          | Vector   | service  | only     |
  +----------+----------+----------+----------+
  | Embeddings| Yes     | No       | No       |
  +----------+----------+----------+----------+
  | Sessions | Yes      | Yes      | No       |
  +----------+----------+----------+----------+
  | Speed    | Fast     | Network  | Slow     |
  +----------+----------+----------+----------+
  | Use case | Default  | Cloud    | Simple   |
  +----------+----------+----------+----------+
```

### 5.6 Security Module

ZeroClaw enforces **defence in depth** across every layer:

```
  SECURITY LAYERS
  ===============

  Layer 1: AUTONOMY LEVEL (src/security/policy.rs:9)
  +--------------------------------------------------+
  | ReadOnly   -- can observe, cannot act             |
  | Supervised -- acts within allowlists (DEFAULT)    |
  | Full       -- autonomous within workspace sandbox |
  +--------------------------------------------------+

  Layer 2: COMMAND ALLOWLIST (policy.rs:354)
  +--------------------------------------------------+
  | Only whitelisted commands can execute             |
  | Default: git, npm, cargo, ls, cat, grep, ...     |
  | Blocks: rm, sudo, curl, wget, ssh, ...           |
  | Also blocks: backtick, $(), redirects (>), tee   |
  +--------------------------------------------------+

  Layer 3: PATH SECURITY (policy.rs:465)
  +--------------------------------------------------+
  | workspace_only = true (default)                   |
  | Blocks: ../ traversal, null bytes, symlink escape |
  | Forbidden: /etc, /root, ~/.ssh, ~/.aws, ...       |
  | Resolves real path after join (anti-symlink)      |
  +--------------------------------------------------+

  Layer 4: RATE LIMITING (policy.rs:34)
  +--------------------------------------------------+
  | Sliding window: max_actions_per_hour              |
  | Cost cap: max_cost_per_day_cents                  |
  | ActionTracker records timestamps, prunes old      |
  +--------------------------------------------------+

  Layer 5: GATEWAY SECURITY (gateway/mod.rs)
  +--------------------------------------------------+
  | Binds 127.0.0.1 only (refuses public bind)       |
  | 6-digit pairing code on startup                   |
  | Bearer token for all /webhook requests            |
  | 64KB body limit, 30s timeout                      |
  | Sliding-window rate limiter per IP                |
  +--------------------------------------------------+

  Layer 6: SECRET ENCRYPTION (security/secrets.rs)
  +--------------------------------------------------+
  | ChaCha20-Poly1305 AEAD encryption                 |
  | Key stored in ~/.zeroclaw/.secret_key             |
  | API keys encrypted at rest                        |
  | Credential scrubbing in tool output               |
  +--------------------------------------------------+

  Layer 7: SANDBOXING (security/detect.rs)
  +--------------------------------------------------+
  | Landlock (Linux kernel, preferred)                |
  | Firejail (process isolation)                      |
  | Docker (full containerisation)                    |
  | Bubblewrap (lightweight)                          |
  +--------------------------------------------------+
```

### 5.7 Gateway (HTTP Server)

**File:** `src/gateway/mod.rs`

The gateway is an Axum-based HTTP server that accepts webhooks.

```
  GATEWAY ENDPOINTS
  =================

  GET  /health ............ Public health check (no auth)
  POST /pair .............. Exchange pairing code for bearer token
  POST /webhook ........... Send message (requires bearer token)
  GET  /whatsapp .......... Meta webhook verification
  POST /whatsapp .......... WhatsApp incoming messages
```

```
  GATEWAY INTERNAL ARCHITECTURE
  =============================

  HTTP Request
       |
       v
  +----+-------+
  | Axum       |  <-- tower-http: 64KB body limit
  | Router     |      tower-http: 30s timeout
  +----+-------+
       |
       v
  +----+-------+
  | Rate       |  <-- SlidingWindowRateLimiter
  | Limiter    |      Per-IP tracking
  +----+-------+
       |
       v
  +----+-------+
  | Auth       |  <-- Bearer token validation
  | Check      |      constant_time_eq (no timing leak)
  +----+-------+
       |
       v
  +----+---------+
  | Idempotency  |  <-- Reject duplicate requests
  | Check        |
  +----+---------+
       |
       v
  +----+-------+
  | Process    |  <-- Create provider, memory, tools
  | Message    |      Run agent loop
  +----+-------+      Return response
       |
       v
  JSON response: { "response": "..." }
```

### 5.8 Observability

```
  OBSERVER IMPLEMENTATIONS
  ========================

  +-- NoopObserver ......... Does nothing (default if not configured)
  +-- LogObserver .......... Structured tracing (tracing crate)
  +-- VerboseObserver ...... Rich console output for debugging
  +-- MultiObserver ........ Combines multiple observers
  +-- OtelObserver ......... OpenTelemetry OTLP export (traces + metrics)
  +-- PrometheusObserver ... Prometheus /metrics endpoint
```

### 5.9 Runtime Adapters

```
  RUNTIME ADAPTER MAP
  ====================

  "native" -----> NativeRuntime
                  Executes shell commands directly on host OS
                  Uses tokio::process::Command

  "docker" -----> DockerRuntime
                  Executes commands inside a Docker container
                  Mounts workspace as /workspace
                  Optional: read-only rootfs, network=none, memory limit

  "wasm"   -----> WasmRuntime (PLANNED)
                  Currently errors on use with clear message
```

### 5.10 Peripherals (Hardware)

```
  PERIPHERAL ARCHITECTURE
  =======================

  +-- Raspberry Pi GPIO (rppal crate, Linux only)
  |     Read/write GPIO pins
  |     PWM, I2C, SPI interfaces
  |
  +-- Serial Port (tokio-serial)
  |     Communicate with microcontrollers
  |     STM32, Arduino, ESP32
  |
  +-- Arduino
  |     Flash firmware (.ino)
  |     GPIO bridge for agent control
  |
  +-- STM32 Nucleo
  |     Flash firmware via probe-rs
  |     Memory read/write via ST-Link
  |
  +-- ESP32
       Firmware in firmware/zeroclaw-esp32/
       WiFi-enabled edge deployment
```

---

## 6. Communication Flow Diagrams

### 6.1 CLI Interactive Chat Flow

```
  USER                  CLI Channel            Agent Loop             LLM Provider
  ----                  -----------            ----------             ------------
   |                       |                      |                       |
   |--- types "Hello" --->|                      |                       |
   |                       |--- ChannelMessage -->|                       |
   |                       |                      |--- Load memories ---> Memory
   |                       |                      |<-- Context ---------- Memory
   |                       |                      |                       |
   |                       |                      |--- ChatRequest ------>|
   |                       |                      |                       |--- HTTPS -->
   |                       |                      |                       |<-- JSON ----
   |                       |                      |<-- ChatResponse ------|
   |                       |                      |                       |
   |                       |                      | (no tool calls)       |
   |                       |                      |                       |
   |                       |<-- SendMessage ------|                       |
   |<-- prints response ---|                      |                       |
   |                       |                      |                       |
```

### 6.2 Gateway Webhook Flow

```
  HTTP Client            Gateway               Agent Loop             Provider
  -----------            -------               ----------             --------
   |                       |                      |                      |
   |-- POST /pair -------->|                      |                      |
   |   X-Pairing-Code: 123|                      |                      |
   |<-- { token: "abc" } --|                      |                      |
   |                       |                      |                      |
   |-- POST /webhook ----->|                      |                      |
   |   Authorization:      |                      |                      |
   |   Bearer abc          |                      |                      |
   |   {"message":"hi"}    |                      |                      |
   |                       |-- validate token --->|                      |
   |                       |-- rate limit check ->|                      |
   |                       |-- idempotency chk -->|                      |
   |                       |                      |                      |
   |                       |                      |-- process message -->|
   |                       |                      |   (full agent loop)  |
   |                       |                      |<-- response ---------|
   |                       |                      |                      |
   |<-- {"response":"..."}-|<-- result -----------|                      |
   |                       |                      |                      |
```

### 6.3 Channel Listener Flow

```
  Telegram Server        TelegramChannel         mpsc::channel          Agent Loop
  ---------------        ---------------         -------------          ----------
   |                       |                        |                      |
   |                       |--- GET /getUpdates --->|                      |
   |--- updates JSON ----->|                        |                      |
   |                       |                        |                      |
   |                       | Check allowlist        |                      |
   |                       | Parse message          |                      |
   |                       |--- tx.send(msg) ------>|                      |
   |                       |                        |--- rx.recv() ------->|
   |                       |                        |                      |
   |                       |                        |                 Process msg
   |                       |                        |                 (agent loop)
   |                       |                        |                      |
   |                       |<----------- SendMessage { content, recipient }|
   |<-- sendMessage -------|                        |                      |
   |                       |                        |                      |
```

### 6.4 Daemon Mode Flow

```
  DAEMON MODE (src/daemon/)
  =========================

  zeroclaw daemon
       |
       v
  +----+-------+
  | Start      |
  | Gateway    |----> Axum HTTP server (webhooks, pairing)
  +----+-------+
       |
       v
  +----+-------+
  | Start      |----> Telegram listener (if configured)
  | Channels   |----> Discord listener (if configured)
  +----+-------+----> Slack listener (if configured)
       |              ... more channels
       v
  +----+-------+
  | Start      |----> Cron scheduler (periodic tasks)
  | Scheduler  |
  +----+-------+
       |
       v
  +----+-------+
  | Start      |----> Heartbeat ticker (HEARTBEAT.md tasks)
  | Heartbeat  |
  +----+-------+
       |
       v
  tokio::select! {
      // Wait for any task to complete or Ctrl+C
      // All tasks run concurrently on the tokio runtime
  }
```

### 6.5 Tool Execution Flow

```
  TOOL CALL CYCLE (detail)
  ========================

  LLM Response: "I'll check the files for you."
  + ToolCall { id: "tc_1", name: "shell", args: {"command": "ls src/"} }
  + ToolCall { id: "tc_2", name: "file_read", args: {"path": "Cargo.toml"} }
       |
       v
  For each tool call:
       |
  +----+----------+
  | 1. Policy     |
  |    check      |  SecurityPolicy.validate_command_execution("ls src/")
  +----+----------+  SecurityPolicy.is_path_allowed("Cargo.toml")
       |
       v
  +----+----------+
  | 2. Rate       |
  |    limit      |  SecurityPolicy.enforce_tool_operation(Act, "shell")
  +----+----------+
       |
       v
  +----+----------+
  | 3. Execute    |
  |    tool       |  ShellTool.execute({"command": "ls src/"})
  +----+----------+  FileReadTool.execute({"path": "Cargo.toml"})
       |
       v
  +----+----------+
  | 4. Scrub      |
  |    output     |  Remove credentials from tool output
  +----+----------+
       |
       v
  +----+----------+
  | 5. Append     |
  |    results    |  ToolResultMessage { tool_call_id: "tc_1", content: "..." }
  +----+----------+
       |
       v
  Send updated history back to LLM for next iteration
```

### 6.6 Memory Recall Flow (Hybrid Search)

```
  HYBRID SEARCH (src/memory/sqlite.rs + vector.rs)
  =================================================

  memory_recall("user's preferred language", limit=5)
       |
       +---------------------+----------------------+
       |                     |                      |
       v                     v                      v
  FTS5 Query            Embed Query             Session filter
  "user preferred       embed("user's           session_id = Some("abc")
   language"            preferred language")
       |                     |                      |
       v                     v                      |
  BM25 ranked           cosine_similarity           |
  results               against all embeddings      |
       |                     |                      |
       +---------------------+                      |
                  |                                  |
                  v                                  |
         weighted_merge(                             |
           keyword_results,  keyword_weight=0.3,     |
           vector_results,   vector_weight=0.7       |
         )                                           |
                  |                                  |
                  v                                  |
         Filter by session if provided <-------------+
                  |
                  v
         Return top 5 MemoryEntry results
```

### 6.7 Security Validation Flow

```
  SECURITY DECISION TREE
  ======================

  Tool execution request
       |
       v
  Is autonomy ReadOnly? ---Yes---> DENY (read-only mode)
       | No
       v
  Is this a Read operation? ---Yes---> ALLOW (reads always ok)
       | No (it's an Act)
       v
  Is rate limit exceeded? ---Yes---> DENY (budget exhausted)
       | No
       v
  Is command in allowlist? ---No---> DENY (not in allowed_commands)
       | Yes
       v
  Any injection patterns? ---Yes---> DENY (backtick, $(), redirect)
  ($(), >, tee, &, etc.)
       | No
       v
  Command risk level?
       |
       +-- High -----> block_high_risk? ---Yes---> DENY
       |                                   No
       |               Supervised + !approved? ---> DENY
       |
       +-- Medium ---> Supervised + require_approval + !approved? ---> DENY
       |
       +-- Low ------> ALLOW
```

### 6.8 Pairing and Authentication Flow

```
  PAIRING FLOW (first-time gateway access)
  ==========================================

  1. Gateway starts, generates 6-digit code, prints to console:
     "Pairing code: 847293"

  2. Client exchanges code for token:

     Client                    Gateway
     ------                    -------
     POST /pair
     X-Pairing-Code: 847293
                               Validate code (constant-time compare)
                               Generate bearer token (UUID)
                               Save token to config
     <-- 200 { "token": "550e8400-e29b-..." }

  3. Subsequent requests use the bearer token:

     POST /webhook
     Authorization: Bearer 550e8400-e29b-...
     {"message": "Hello"}
                               Validate token (constant-time compare)
                               Process message
     <-- 200 {"response": "Hi there!"}
```

---

## 7. Data Flow Between Subsystems

This diagram shows how data flows between all major subsystems during
a complete agent interaction:

```
  COMPLETE DATA FLOW (one user message lifecycle)
  ================================================

  [User]
    |
    | "Summarise my project files"
    v
  [Channel] (e.g. Telegram)
    |
    | ChannelMessage { sender, content, channel, timestamp }
    v
  [Agent Loop]
    |
    |--- (1) Load context --->  [Memory] ------> brain.db
    |                                              |
    |<-- MemoryEntry[] ---------[Memory] <--------+
    |
    |--- (2) Build prompt --->  [Prompt Builder]
    |                           IDENTITY.md + SOUL.md + memories + skills
    |<-- system_prompt ----------|
    |
    |--- (3) Call LLM ------->  [Provider] (e.g. OpenRouter)
    |                              |
    |                              |----> HTTPS to api.openrouter.ai
    |                              |<---- JSON response
    |<-- ChatResponse { text, tool_calls: [shell("ls"), file_read("README.md")] }
    |
    |--- (4) Security check -->  [SecurityPolicy]
    |                              Is "ls" allowed? Yes
    |                              Is "README.md" in workspace? Yes
    |<-- OK --------------------  |
    |
    |--- (5) Execute tools ---->  [Tool: Shell] ---> OS: ls
    |                              [Tool: FileRead] -> fs::read("README.md")
    |<-- ToolResult[] ----------  |
    |
    |--- (6) Scrub output ----->  [Credential Scrubber]
    |<-- clean output ----------  |
    |
    |--- (7) Feed back to LLM -> [Provider] (second call with tool results)
    |<-- ChatResponse { text: "Here is a summary..." }
    |
    |--- (8) Auto-save -------->  [Memory] store conversation
    |
    |--- (9) Observe ---------->  [Observer] log event
    |
    | SendMessage { content: "Here is a summary..." }
    v
  [Channel] (send response back to Telegram)
    |
    v
  [User sees response]
```

---

## 8. Configuration System

```
  CONFIG LOADING (src/config/)
  ============================

  1. Check for ~/.zeroclaw/config.toml
  2. If missing, create with defaults
  3. Parse TOML into Config struct
  4. Apply environment variable overrides:
     - ZEROCLAW_API_KEY
     - ZEROCLAW_PROVIDER
     - ZEROCLAW_MODEL
     - RUST_LOG (logging level)
     - OLLAMA_API_KEY
     - ANTHROPIC_API_KEY / ANTHROPIC_AUTH_TOKEN
     - OPENAI_API_KEY
     - etc.

  CONFIG STRUCT HIERARCHY
  =======================

  Config (top-level)
  +-- api_key: Option<String>
  +-- default_provider: Option<String>
  +-- default_model: Option<String>
  +-- default_temperature: f64
  +-- autonomy: AutonomyConfig
  |     +-- level: AutonomyLevel (readonly/supervised/full)
  |     +-- workspace_only: bool
  |     +-- allowed_commands: Vec<String>
  |     +-- forbidden_paths: Vec<String>
  |     +-- max_actions_per_hour: u32
  |     +-- max_cost_per_day_cents: u32
  +-- memory: MemoryConfig
  |     +-- backend: String (sqlite/lucid/markdown/none)
  |     +-- embedding_provider: String
  |     +-- vector_weight / keyword_weight: f64
  +-- gateway: GatewayConfig
  |     +-- host / port
  |     +-- require_pairing: bool
  |     +-- allow_public_bind: bool
  +-- runtime: RuntimeConfig
  |     +-- kind: String (native/docker)
  +-- channels_config: ChannelsConfig
  |     +-- telegram: Option<TelegramConfig>
  |     +-- discord: Option<DiscordConfig>
  |     +-- slack: Option<SlackConfig>
  |     +-- whatsapp: Option<WhatsAppConfig>
  +-- browser: BrowserConfig
  +-- heartbeat: HeartbeatConfig
  +-- tunnel: TunnelConfig
  +-- secrets: SecretsConfig
  +-- peripherals: PeripheralsConfig
  +-- cost: CostConfig
  +-- ... (more sections)
```

---

## 9. Build and Release Architecture

```
  BUILD PROFILES
  ==============

  cargo build                     Dev build (fast compile, no optimisation)
  cargo build --release           Release (opt-level=z, codegen-units=1, strip)
  cargo build --profile release-fast  Fast release (codegen-units=8, needs 16GB+ RAM)
  cargo build --profile dist      Distribution (fat LTO, smallest binary)

  FEATURE FLAGS
  =============

  default         = ["hardware"]         USB enumeration + serial
  hardware        = ["nusb", "tokio-serial"]
  browser-native  = ["dep:fantoccini"]   Rust-native browser (WebDriver)
  sandbox-landlock= ["dep:landlock"]     Linux Landlock sandboxing
  peripheral-rpi  = ["rppal"]            Raspberry Pi GPIO (Linux only)
  probe           = ["dep:probe-rs"]     ST-Link probe support (~50 deps)
  rag-pdf         = ["dep:pdf-extract"]  PDF extraction

  CI PIPELINE (.github/workflows/)
  ================================

  ci-run.yml:
    1. cargo fmt --check
    2. cargo clippy -- -D warnings
    3. cargo test
    4. cargo build --release
    5. markdownlint (changed files)
    6. Link checking (changed markdown)

  sec-audit.yml:
    1. cargo deny check
    2. rustsec advisory database

  sec-codeql.yml:
    1. CodeQL static analysis
```

---

## 10. Testing Architecture

```
  TEST ORGANISATION
  =================

  Unit tests:         In each module, marked with #[cfg(test)]
  Integration tests:  tests/ directory
  Benchmarks:         benches/ directory (criterion)
  E2E tests:          .github/workflows/test-*.yml

  TEST COMMANDS
  =============

  cargo test                       All tests (1,017 total)
  cargo test agent::               Agent module tests
  cargo test --lib                 Unit tests only
  cargo test -- --nocapture        Show print output
  cargo test -- --test-threads=1   Serial execution
  cargo bench                      Performance benchmarks

  WHAT EACH MODULE TESTS
  ======================

  providers/traits.rs .... Chat message construction, serialisation,
                           capability detection, prompt-guided fallback
  channels/traits.rs ..... Message cloning, default trait methods,
                           listener sends to mpsc channel
  tools/traits.rs ........ Tool spec generation, execution, serialisation
  memory/traits.rs ....... Category display, serde roundtrip, entry fields
  security/policy.rs ..... Command allowlist (50+ injection patterns),
                           path traversal, rate limiting, autonomy levels
  gateway/mod.rs ......... Rate limiter, pairing, idempotency
```

---

This document should give you a complete mental model of ZeroClaw's
architecture. For implementation guidance, see `docs/DEVELOPER_GUIDE.md`.
For user-facing instructions, see `docs/USER_GUIDE.md`.
