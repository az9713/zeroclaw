# ZeroClaw Developer Guide

A step-by-step, hand-holding guide for developers who want to contribute
to ZeroClaw. Written for people with C/C++/Java experience who are new
to Rust, async programming, and trait-based architectures.

---

## Table of Contents

1.  [What You Need to Know First](#1-what-you-need-to-know-first)
2.  [Setting Up Your Development Environment](#2-setting-up-your-development-environment)
3.  [Building ZeroClaw From Source](#3-building-zeroclaw-from-source)
4.  [Rust Crash Course for C/C++/Java Developers](#4-rust-crash-course)
5.  [Understanding the Codebase](#5-understanding-the-codebase)
6.  [How to Add a New Provider](#6-how-to-add-a-new-provider)
7.  [How to Add a New Channel](#7-how-to-add-a-new-channel)
8.  [How to Add a New Tool](#8-how-to-add-a-new-tool)
9.  [How to Add a New Memory Backend](#9-how-to-add-a-new-memory-backend)
10. [Running Tests](#10-running-tests)
11. [Writing Tests](#11-writing-tests)
12. [Debugging](#12-debugging)
13. [Code Style and Conventions](#13-code-style-and-conventions)
14. [Git Workflow](#14-git-workflow)
15. [CI/CD Pipeline](#15-cicd-pipeline)
16. [Common Pitfalls and Solutions](#16-common-pitfalls-and-solutions)
17. [Glossary](#17-glossary)

---

## 1. What You Need to Know First

### What is ZeroClaw?

ZeroClaw is an AI agent runtime. Think of it as a framework that:

1. **Receives messages** from users (via terminal, Telegram, Discord, etc.)
2. **Sends them to an AI model** (OpenAI, Claude, local Ollama, etc.)
3. **Lets the AI use tools** (run shell commands, read files, search memory)
4. **Sends the response back** to the user

### What is Rust?

Rust is a systems programming language (like C/C++) that:

- Compiles to native machine code (no runtime/VM needed)
- Guarantees memory safety without a garbage collector
- Has a package manager called `cargo` (like npm, pip, or maven)
- Uses `Cargo.toml` as its project manifest (like package.json or pom.xml)

### What is async/await?

ZeroClaw makes many network calls (to LLM APIs, messaging platforms, etc.).
Instead of blocking a thread for each call, Rust uses `async`/`await`:

```rust
// This does NOT block the thread while waiting for the HTTP response
let response = client.get("https://api.openai.com/...").send().await?;
```

The `tokio` runtime manages these async operations efficiently.

### What are traits?

A Rust trait is like a Java interface or a C++ abstract class:

```rust
// Define what any "Provider" must be able to do
trait Provider {
    async fn chat(&self, message: &str) -> Result<String>;
}

// Implement it for a specific provider
struct OpenAiProvider { api_key: String }

impl Provider for OpenAiProvider {
    async fn chat(&self, message: &str) -> Result<String> {
        // Call OpenAI API here
    }
}
```

---

## 2. Setting Up Your Development Environment

### Step 1: Install Rust

**Windows:**
```powershell
# Install Visual Studio Build Tools first
winget install Microsoft.VisualStudio.2022.BuildTools
# During installation, select "Desktop development with C++" workload

# Install Rust
winget install Rustlang.Rustup

# Open a NEW terminal, then verify
rustc --version    # Should print something like: rustc 1.XX.0
cargo --version    # Should print something like: cargo 1.XX.0
```

**macOS:**
```bash
# Install Xcode command line tools
xcode-select --install

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Reload your shell
source ~/.cargo/env

# Verify
rustc --version
cargo --version
```

**Linux (Debian/Ubuntu):**
```bash
# Install build essentials
sudo apt update
sudo apt install build-essential pkg-config

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Reload your shell
source ~/.cargo/env

# Verify
rustc --version
cargo --version
```

### Step 2: Install a Code Editor

We recommend **VS Code** with the `rust-analyzer` extension:

1. Install VS Code: https://code.visualstudio.com
2. Open VS Code
3. Go to Extensions (Ctrl+Shift+X / Cmd+Shift+X)
4. Search for "rust-analyzer" and install it
5. Search for "Even Better TOML" and install it (for Cargo.toml files)

### Step 3: Clone the Repository

```bash
git clone https://github.com/zeroclaw-labs/zeroclaw.git
cd zeroclaw
```

### Step 4: Verify the Build

```bash
# This will download all dependencies and compile the project
# First build takes 2-5 minutes. Subsequent builds are much faster.
cargo build

# Run all tests to make sure everything works
cargo test
```

If you see `Finished` with no errors, you are ready to develop.

---

## 3. Building ZeroClaw From Source

### Build Commands

```bash
# Development build (fast to compile, not optimised)
cargo build

# Release build (slow to compile, optimised for size)
# Uses codegen-units=1, works even on Raspberry Pi (1 GB RAM)
cargo build --release

# Fast release build (needs 16+ GB RAM)
cargo build --profile release-fast

# Install globally (puts binary in ~/.cargo/bin/)
cargo install --path . --force --locked

# Run without installing
cargo run -- status        # The -- separates cargo args from program args
cargo run -- agent -m "Hello"
```

### Understanding Cargo.toml

Open `Cargo.toml` at the project root. Key sections:

```toml
[package]
name = "zeroclaw"        # The binary name
version = "0.1.0"        # Current version
edition = "2021"         # Rust edition (determines language features)

[dependencies]
clap = "4.5"             # CLI argument parsing (like argparse in Python)
tokio = "1.42"           # Async runtime (like Node.js event loop)
reqwest = "0.12"         # HTTP client (like axios or fetch)
serde = "1.0"            # Serialization (like Jackson in Java)
serde_json = "1.0"       # JSON support
rusqlite = "0.38"        # SQLite database
axum = "0.8"             # HTTP server framework

[features]
default = ["hardware"]   # Features enabled by default
hardware = [...]         # USB device enumeration
browser-native = [...]   # Optional browser automation
```

### Feature Flags

Some features are optional to keep the binary small:

```bash
# Build with browser support
cargo build --release --features browser-native

# Build with all features
cargo build --release --all-features

# Build without default features (smallest binary)
cargo build --release --no-default-features
```

---

## 4. Rust Crash Course

This section maps Rust concepts to C/C++/Java equivalents.

### Variables and Types

```rust
// Immutable by default (like const in C++)
let x: i32 = 42;

// Mutable variable
let mut counter: u32 = 0;
counter += 1;

// Type inference (like auto in C++)
let name = "ZeroClaw";          // &str (string slice)
let owned = String::from("hi"); // String (heap-allocated, like std::string)
```

### Structs (like C structs or Java classes)

```rust
struct Config {
    api_key: Option<String>,     // Optional field (null-safe!)
    temperature: f64,            // f64 = double
    workspace_only: bool,
}
```

### Enums (much more powerful than C/Java enums)

```rust
enum AutonomyLevel {
    ReadOnly,
    Supervised,
    Full,
}

// Enums can hold data (like tagged unions)
enum MemoryCategory {
    Core,
    Daily,
    Custom(String),  // This variant carries a String
}
```

### Error Handling (no exceptions!)

```rust
// Rust uses Result<T, E> instead of try/catch
fn read_file(path: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}

// The ? operator propagates errors (like throw in Java)
fn process() -> Result<()> {
    let content = read_file("config.toml")?;  // Returns error if fails
    println!("{}", content);
    Ok(())
}
```

### Option (no null pointers!)

```rust
// Option<T> is either Some(value) or None
let api_key: Option<String> = Some("sk-abc123".into());
let missing: Option<String> = None;

// Use match or if let to handle
if let Some(key) = api_key {
    println!("Key: {}", key);
} else {
    println!("No key configured");
}
```

### Traits (like Java interfaces)

```rust
// Define the interface
trait Tool: Send + Sync {      // Send + Sync = safe to use across threads
    fn name(&self) -> &str;
    async fn execute(&self, args: Value) -> Result<ToolResult>;
}

// Implement it
struct ShellTool { policy: SecurityPolicy }

impl Tool for ShellTool {
    fn name(&self) -> &str { "shell" }
    async fn execute(&self, args: Value) -> Result<ToolResult> {
        // Implementation here
    }
}
```

### Ownership and Borrowing (the big Rust concept)

```rust
// Ownership: each value has exactly one owner
let s1 = String::from("hello");
let s2 = s1;           // s1 is MOVED to s2, s1 is no longer valid
// println!("{}", s1);  // COMPILE ERROR: s1 was moved

// Borrowing: lend a reference without transferring ownership
let s1 = String::from("hello");
let len = calculate_length(&s1);  // &s1 borrows s1 (like a pointer)
println!("{} has length {}", s1, len);  // s1 is still valid

fn calculate_length(s: &str) -> usize {
    s.len()
}
```

**The key rule:** You can have either:
- One mutable reference (`&mut T`), OR
- Any number of immutable references (`&T`)

This prevents data races at compile time.

### Async/Await

```rust
// Mark a function as async
async fn fetch_data(url: &str) -> Result<String> {
    let response = reqwest::get(url).await?;  // .await pauses here
    let text = response.text().await?;
    Ok(text)
}

// The #[tokio::main] macro sets up the async runtime
#[tokio::main]
async fn main() -> Result<()> {
    let data = fetch_data("https://example.com").await?;
    println!("{}", data);
    Ok(())
}
```

---

## 5. Understanding the Codebase

### Where to Start Reading

Read these files in order:

1. **`src/main.rs`** - The entry point. See all CLI commands.
2. **`src/lib.rs`** - All module declarations.
3. **`src/providers/traits.rs`** - The Provider trait (most important).
4. **`src/channels/traits.rs`** - The Channel trait.
5. **`src/tools/traits.rs`** - The Tool trait.
6. **`src/memory/traits.rs`** - The Memory trait.
7. **`src/config/schema.rs`** - All configuration options.
8. **`src/security/policy.rs`** - Security enforcement.
9. **`src/agent/loop_.rs`** - The main agent loop.

### How a Message Flows Through the Code

1. User types a message in the terminal
2. `src/main.rs` dispatches to `Commands::Agent`
3. `src/agent/agent.rs` creates the Agent with provider, memory, tools
4. `src/agent/loop_.rs` runs the agentic loop:
   a. `memory_loader.rs` fetches relevant memories
   b. `prompt.rs` builds the system prompt
   c. Provider's `chat()` sends messages + tools to LLM
   d. If LLM returns tool calls, `dispatcher.rs` executes them
   e. Results feed back into the conversation
   f. Repeat until no more tool calls
5. Final response sent back through the channel

### The Factory Pattern

Every module has a factory function in its `mod.rs`:

```
src/providers/mod.rs   ->  create_provider(config) -> Box<dyn Provider>
src/channels/mod.rs    ->  create_channels(config) -> Vec<Box<dyn Channel>>
src/tools/mod.rs       ->  create_tools(config)    -> Vec<Box<dyn Tool>>
src/memory/mod.rs      ->  create_memory(config)   -> Box<dyn Memory>
```

These factories read the config and return the right implementation.

---

## 6. How to Add a New Provider

Let's walk through adding a hypothetical "Mistral" provider.

### Step 1: Create the File

Create `src/providers/mistral.rs`:

```rust
use crate::providers::traits::{
    ChatMessage, ChatRequest, ChatResponse, Provider, ProviderCapabilities,
    ToolCall, ToolsPayload,
};
use async_trait::async_trait;

pub struct MistralProvider {
    api_key: String,
    base_url: String,
    client: reqwest::Client,
}

impl MistralProvider {
    pub fn new(api_key: &str) -> Self {
        Self {
            api_key: api_key.to_string(),
            base_url: "https://api.mistral.ai/v1".to_string(),
            client: reqwest::Client::new(),
        }
    }
}

#[async_trait]
impl Provider for MistralProvider {
    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            native_tool_calling: true,
        }
    }

    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String> {
        // Build the request body
        let mut messages = vec![];
        if let Some(sys) = system_prompt {
            messages.push(serde_json::json!({"role": "system", "content": sys}));
        }
        messages.push(serde_json::json!({"role": "user", "content": message}));

        let body = serde_json::json!({
            "model": model,
            "messages": messages,
            "temperature": temperature,
        });

        // Send HTTP request
        let response = self.client
            .post(format!("{}/chat/completions", self.base_url))
            .header("Authorization", format!("Bearer {}", self.api_key))
            .json(&body)
            .send()
            .await?;

        // Parse response
        let json: serde_json::Value = response.json().await?;
        let text = json["choices"][0]["message"]["content"]
            .as_str()
            .unwrap_or("")
            .to_string();

        Ok(text)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn mistral_provider_has_native_tools() {
        let provider = MistralProvider::new("test-key");
        assert!(provider.capabilities().native_tool_calling);
    }
}
```

### Step 2: Register in the Module

Edit `src/providers/mod.rs` and add:

```rust
mod mistral;

// In the factory function:
pub fn create_provider(config: &Config) -> anyhow::Result<Box<dyn Provider>> {
    match provider_name {
        // ... existing providers ...
        "mistral" => Ok(Box::new(mistral::MistralProvider::new(api_key))),
        // ...
    }
}
```

### Step 3: Add Tests

Your tests should cover:
- Basic chat works (mock the HTTP call in tests)
- Error handling (what if API returns 429 rate limit?)
- Tool calling format (if the provider supports native tools)

### Step 4: Update Documentation

Add the provider to the providers table in `README.md`.

---

## 7. How to Add a New Channel

### Step 1: Create the File

Create `src/channels/my_platform.rs`:

```rust
use crate::channels::traits::{Channel, ChannelMessage, SendMessage};
use async_trait::async_trait;
use tokio::sync::mpsc;

pub struct MyPlatformChannel {
    token: String,
    allowed_users: Vec<String>,
    client: reqwest::Client,
}

impl MyPlatformChannel {
    pub fn new(token: &str, allowed_users: Vec<String>) -> Self {
        Self {
            token: token.to_string(),
            allowed_users,
            client: reqwest::Client::new(),
        }
    }
}

#[async_trait]
impl Channel for MyPlatformChannel {
    fn name(&self) -> &str {
        "my_platform"
    }

    async fn send(&self, message: &SendMessage) -> anyhow::Result<()> {
        // Send message via your platform's API
        self.client
            .post("https://api.myplatform.com/messages")
            .header("Authorization", format!("Bearer {}", self.token))
            .json(&serde_json::json!({
                "to": message.recipient,
                "text": message.content,
            }))
            .send()
            .await?;
        Ok(())
    }

    async fn listen(&self, tx: mpsc::Sender<ChannelMessage>) -> anyhow::Result<()> {
        // Long-poll or WebSocket loop
        loop {
            // Fetch new messages from the platform
            // For each message:
            //   1. Check if sender is in allowed_users
            //   2. Create ChannelMessage
            //   3. tx.send(msg).await
            tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        }
    }

    async fn health_check(&self) -> bool {
        // Check if the platform API is reachable
        self.client
            .get("https://api.myplatform.com/health")
            .send()
            .await
            .is_ok()
    }
}
```

### Step 2: Register in mod.rs

Edit `src/channels/mod.rs` and add the module + factory entry.

### Step 3: Add Config Support

Edit `src/config/schema.rs` to add a config section:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MyPlatformConfig {
    pub token: String,
    #[serde(default)]
    pub allowed_users: Vec<String>,
}
```

Add it to `ChannelsConfig`:

```rust
pub struct ChannelsConfig {
    // ... existing fields ...
    pub my_platform: Option<MyPlatformConfig>,
}
```

---

## 8. How to Add a New Tool

### Step 1: Create the File

Create `src/tools/my_tool.rs`:

```rust
use crate::tools::traits::{Tool, ToolResult};
use async_trait::async_trait;
use serde_json::Value;

pub struct MyTool;

#[async_trait]
impl Tool for MyTool {
    fn name(&self) -> &str {
        "my_tool"
    }

    fn description(&self) -> &str {
        "Does something useful. Provide a 'query' parameter."
    }

    fn parameters_schema(&self) -> Value {
        serde_json::json!({
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                }
            },
            "required": ["query"]
        })
    }

    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
        let query = args["query"]
            .as_str()
            .unwrap_or_default();

        // Do something with the query
        let result = format!("Results for: {}", query);

        Ok(ToolResult {
            success: true,
            output: result,
            error: None,
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn my_tool_returns_success() {
        let tool = MyTool;
        let result = tool
            .execute(serde_json::json!({"query": "test"}))
            .await
            .unwrap();
        assert!(result.success);
        assert!(result.output.contains("test"));
    }

    #[test]
    fn my_tool_has_correct_schema() {
        let tool = MyTool;
        let schema = tool.parameters_schema();
        assert_eq!(schema["type"], "object");
        assert!(schema["properties"]["query"].is_object());
    }
}
```

### Step 2: Register in mod.rs

Edit `src/tools/mod.rs`:

```rust
mod my_tool;

// In the create_tools function:
tools.push(Box::new(my_tool::MyTool));
```

---

## 9. How to Add a New Memory Backend

### Step 1: Create the File

Create `src/memory/my_backend.rs`:

```rust
use crate::memory::traits::{Memory, MemoryCategory, MemoryEntry};
use async_trait::async_trait;

pub struct MyBackend {
    // Your storage fields here
}

#[async_trait]
impl Memory for MyBackend {
    fn name(&self) -> &str { "my_backend" }

    async fn store(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
    ) -> anyhow::Result<()> {
        // Store the memory entry
        Ok(())
    }

    async fn recall(
        &self,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>> {
        // Search for matching entries
        Ok(vec![])
    }

    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>> {
        Ok(None)
    }

    async fn list(
        &self,
        category: Option<&MemoryCategory>,
        session_id: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>> {
        Ok(vec![])
    }

    async fn forget(&self, key: &str) -> anyhow::Result<bool> {
        Ok(false)
    }

    async fn count(&self) -> anyhow::Result<usize> {
        Ok(0)
    }

    async fn health_check(&self) -> bool {
        true
    }
}
```

### Step 2: Register in mod.rs and config

Add to `src/memory/mod.rs` factory and `src/config/schema.rs`.

---

## 10. Running Tests

```bash
# Run ALL tests (1,017 tests)
cargo test

# Run tests for a specific module
cargo test providers::     # All provider tests
cargo test channels::      # All channel tests
cargo test tools::         # All tool tests
cargo test security::      # All security tests

# Run a specific test by name
cargo test command_injection_semicolon_blocked

# Show output from println! in tests
cargo test -- --nocapture

# Run tests one at a time (useful for debugging)
cargo test -- --test-threads=1

# Run only unit tests (no integration tests)
cargo test --lib

# Run benchmarks
cargo bench
```

---

## 11. Writing Tests

### Where to Put Tests

Tests go in the same file as the code, inside a `#[cfg(test)]` module:

```rust
// At the bottom of your_file.rs:

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn my_sync_test() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }

    #[tokio::test]
    async fn my_async_test() {
        let tool = MyTool;
        let result = tool.execute(json!({})).await.unwrap();
        assert!(result.success);
    }
}
```

### Test Naming Convention

Name tests as `<subject>_<expected_behaviour>`:

```rust
#[test] fn command_injection_semicolon_blocked() { ... }
#[test] fn relative_paths_allowed() { ... }
#[test] fn rate_limit_exactly_at_boundary() { ... }
#[test] fn autonomy_default_is_supervised() { ... }
```

### Test Privacy Rules

From CLAUDE.md: Never use real names, emails, or tokens in tests. Use:
- `zeroclaw_user`, `test_user`, `user_a`
- `example.com`, `test.example.com`
- `"test-api-key"`, `"dummy-token"`

---

## 12. Debugging

### Enable Debug Logging

```bash
# Set log level via environment variable
RUST_LOG=debug cargo run -- agent -m "test"
RUST_LOG=trace cargo run -- agent -m "test"   # Very verbose
RUST_LOG=zeroclaw=debug cargo run -- agent     # Only ZeroClaw logs
```

### Use the Doctor Command

```bash
zeroclaw doctor    # Checks config, memory, channels, scheduler
```

### Common Debug Techniques

```rust
// Add temporary debug prints
eprintln!("DEBUG: value = {:?}", my_value);

// Use dbg! macro (prints expression and value)
let result = dbg!(some_function());

// Trace logging
tracing::debug!("Processing message: {:?}", message);
```

---

## 13. Code Style and Conventions

### Formatting

```bash
# Format all code (must pass before commit)
cargo fmt

# Check formatting without modifying
cargo fmt -- --check
```

### Linting

```bash
# Run clippy (must pass with zero warnings)
cargo clippy --all-targets -- -D warnings
```

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Files/modules | snake_case | `file_read.rs` |
| Types/traits | PascalCase | `SecurityPolicy` |
| Functions/vars | snake_case | `is_path_allowed` |
| Constants | SCREAMING_SNAKE | `MAX_BODY_SIZE` |
| Factory keys | lowercase | `"openai"`, `"sqlite"` |

### Module Structure

Every module follows this layout:

```
src/subsystem/
  traits.rs     -- Trait definition + data types
  mod.rs        -- Factory function + module declarations
  impl_a.rs     -- Implementation A
  impl_b.rs     -- Implementation B
```

---

## 14. Git Workflow

### Branch Naming

```
feat/add-mistral-provider
fix/telegram-message-split
docs/update-architecture
chore/update-dependencies
```

### Commit Messages

Use conventional commits:

```
feat(providers): add Mistral AI provider
fix(telegram): handle messages over 4096 chars
docs(architecture): add memory system diagram
test(security): add path traversal edge cases
chore(deps): update tokio to 1.43
```

### PR Process

1. Create a feature branch from `main`
2. Make your changes
3. Run all checks:
   ```bash
   cargo fmt --all -- --check
   cargo clippy --all-targets -- -D warnings
   cargo test
   ```
4. Commit with a clear message
5. Open a PR to `main`
6. Wait for CI checks and code review

---

## 15. CI/CD Pipeline

The CI runs automatically on every PR:

| Step | What It Checks | How to Run Locally |
|------|---------------|-------------------|
| Format | Code formatting | `cargo fmt -- --check` |
| Clippy | Lint warnings | `cargo clippy -- -D warnings` |
| Tests | All 1,017 tests | `cargo test` |
| Build | Release compiles | `cargo build --release` |
| Markdown | Doc quality | `markdownlint docs/` |
| Security | Known CVEs | `cargo audit` |

---

## 16. Common Pitfalls and Solutions

### "Cannot find type Provider"
You need to import the trait:
```rust
use crate::providers::traits::Provider;
```

### "the trait Send is not implemented"
Your type holds a non-Send type. Wrap it in `Arc<Mutex<T>>`:
```rust
use std::sync::Arc;
use parking_lot::Mutex;
struct MyStruct {
    data: Arc<Mutex<Vec<String>>>,
}
```

### "borrow of moved value"
You are using a value after moving it. Clone it or use references:
```rust
let s = String::from("hello");
let s2 = s.clone();  // Clone instead of move
use_both(&s, &s2);
```

### "mismatched types: expected &str, found String"
Use `.as_str()` or `&` to convert:
```rust
let owned: String = "hello".to_string();
let borrowed: &str = owned.as_str();
// or
let borrowed: &str = &owned;
```

### Build fails with "openssl-sys"
Use the locked dependency file:
```bash
cargo build --release --locked
```

### Tests fail with "Cannot start a runtime from within a runtime"
You are mixing blocking and async code. Use `tokio::task::spawn_blocking`:
```rust
tokio::task::spawn_blocking(move || {
    // blocking code here
}).await?;
```

---

## 17. Glossary

| Term | Meaning |
|------|---------|
| **Agent** | The AI assistant that processes messages and uses tools |
| **Provider** | An LLM backend (OpenAI, Claude, Ollama, etc.) |
| **Channel** | A messaging platform (Telegram, Discord, CLI, etc.) |
| **Tool** | A capability the agent can use (shell, file read, etc.) |
| **Memory** | Persistent storage for the agent's knowledge |
| **Trait** | A Rust interface that defines behaviour |
| **Factory** | A function that creates the right implementation from config |
| **Workspace** | The directory where the agent operates (~/.zeroclaw/workspace) |
| **Gateway** | The HTTP server for receiving webhooks |
| **Daemon** | Long-running mode with gateway + channels + scheduler |
| **Pairing** | One-time code exchange for gateway authentication |
| **Autonomy** | How much freedom the agent has (readonly/supervised/full) |
| **FTS5** | SQLite Full-Text Search (keyword search engine) |
| **Vector search** | Finding similar text using embedding vectors |
| **Hybrid search** | Combining keyword + vector search for best results |
| **Allowlist** | List of explicitly permitted items (commands, users) |
| **Sandbox** | Isolated execution environment for security |
| **AEAD** | Authenticated Encryption with Associated Data (ChaCha20-Poly1305) |
| **CSPRNG** | Cryptographically Secure Random Number Generator |
| **mpsc** | Multi-Producer Single-Consumer channel (async queue) |
| **tokio** | The async runtime that manages concurrent tasks |
| **axum** | The web framework used for the HTTP gateway |
| **serde** | Serialization/deserialization framework (like Jackson) |
| **anyhow** | Error handling library (Result type with context) |
| **clippy** | Rust linter (like ESLint or checkstyle) |
| **rustfmt** | Rust code formatter (like prettier or gofmt) |

---

For architecture diagrams, see `docs/ARCHITECTURE.md`.
For user-facing instructions, see `docs/USER_GUIDE.md`.
