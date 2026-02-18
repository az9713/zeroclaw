# ZeroClaw User Guide

A complete, step-by-step guide to using ZeroClaw. No technical experience
required beyond basic terminal usage. This guide includes 12 hands-on use
cases so you can see ZeroClaw in action right away.

---

## Table of Contents

1.  [What is ZeroClaw?](#1-what-is-zeroclaw)
2.  [Installation](#2-installation)
3.  [First-Time Setup (Onboarding)](#3-first-time-setup)
4.  [Quick Start: Your First Conversation](#4-quick-start)
5.  [Use Case 1: Ask a Question (Single Message)](#use-case-1-ask-a-question)
6.  [Use Case 2: Interactive Chat Session](#use-case-2-interactive-chat)
7.  [Use Case 3: Let the Agent Read Your Files](#use-case-3-read-files)
8.  [Use Case 4: Let the Agent Write Code for You](#use-case-4-write-code)
9.  [Use Case 5: Run Shell Commands Through the Agent](#use-case-5-shell-commands)
10. [Use Case 6: Store and Recall Memories](#use-case-6-memory)
11. [Use Case 7: Connect to Telegram](#use-case-7-telegram)
12. [Use Case 8: Schedule Recurring Tasks](#use-case-8-scheduled-tasks)
13. [Use Case 9: Check System Status and Health](#use-case-9-system-status)
14. [Use Case 10: Use a Local AI Model (Ollama)](#use-case-10-local-model)
15. [Use Case 11: Run as a Background Service](#use-case-11-background-service)
16. [Use Case 12: Migrate from Another Agent](#use-case-12-migrate)
17. [Configuration Reference](#17-configuration-reference)
18. [Providers Reference](#18-providers-reference)
19. [Channels Reference](#19-channels-reference)
20. [Tools Reference](#20-tools-reference)
21. [Security Reference](#21-security-reference)
22. [Troubleshooting](#22-troubleshooting)
23. [Frequently Asked Questions](#23-faq)

---

## 1. What is ZeroClaw?

ZeroClaw is your personal AI assistant that runs on your computer. Think
of it like ChatGPT, but:

- **It runs locally** on your machine (even a $10 Raspberry Pi)
- **It can do things** like read/write files, run commands, and search memory
- **It connects to any AI model** (OpenAI, Claude, Ollama, and 20+ others)
- **It works on any messaging platform** (Telegram, Discord, Slack, etc.)
- **It is tiny** (~3.4 MB) and starts in under 10 milliseconds

### What Can ZeroClaw Do?

- Answer questions using any AI model
- Read and write files on your computer
- Run shell commands (safely sandboxed)
- Remember things across conversations
- Chat via Telegram, Discord, Slack, WhatsApp, and more
- Schedule recurring tasks
- Connect to hardware devices (Raspberry Pi, Arduino, STM32)
- Run as a background service (always-on assistant)

---

## 2. Installation

### Step 1: Install Rust (the programming language ZeroClaw is built with)

**Windows:**
```powershell
winget install Microsoft.VisualStudio.2022.BuildTools
# Select "Desktop development with C++" during installation

winget install Rustlang.Rustup
# Open a NEW terminal after installation
```

**macOS:**
```bash
xcode-select --install
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

**Linux:**
```bash
sudo apt install build-essential pkg-config    # Debian/Ubuntu
# OR
sudo dnf groupinstall "Development Tools"      # Fedora

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

### Step 2: Verify Rust is Installed

```bash
rustc --version    # Should show: rustc 1.XX.0
cargo --version    # Should show: cargo 1.XX.0
```

### Step 3: Download and Build ZeroClaw

```bash
git clone https://github.com/zeroclaw-labs/zeroclaw.git
cd zeroclaw
cargo build --release --locked
cargo install --path . --force --locked
```

This takes 2-5 minutes the first time. You will see "Compiling..." messages.
When it says "Finished", ZeroClaw is ready.

### Step 4: Add ZeroClaw to Your PATH

```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

Add this line to your shell profile (`~/.bashrc`, `~/.zshrc`, or
`~/.profile`) so it persists across terminal sessions.

### Step 5: Verify Installation

```bash
zeroclaw --version    # Should show: zeroclaw 0.1.0
```

---

## 3. First-Time Setup

ZeroClaw needs an API key to talk to an AI model. The easiest option is
OpenRouter (which gives you access to 23+ models with one key).

### Option A: Quick Setup (Recommended)

```bash
zeroclaw onboard --api-key YOUR_API_KEY --provider openrouter
```

Replace `YOUR_API_KEY` with your actual API key from:
- OpenRouter: https://openrouter.ai/keys (free tier available)
- OpenAI: https://platform.openai.com/api-keys
- Anthropic: https://console.anthropic.com

### Option B: Interactive Wizard

```bash
zeroclaw onboard --interactive
```

This walks you through 7 steps:
1. Choose a provider (OpenRouter, OpenAI, Anthropic, Ollama, etc.)
2. Enter your API key
3. Choose a model
4. Choose a memory backend (SQLite recommended)
5. Configure channels (Telegram, Discord, etc.)
6. Set security preferences
7. Test the connection

### Option C: Local AI (No API Key Needed)

If you have Ollama installed (https://ollama.com):

```bash
# Start Ollama first
ollama serve

# Pull a model
ollama pull llama3.2

# Set up ZeroClaw to use it
zeroclaw onboard --provider ollama
```

---

## 4. Quick Start

```bash
# Send a single message
zeroclaw agent -m "What is Rust?"

# Start an interactive chat session
zeroclaw agent

# Check your current configuration
zeroclaw status
```

---

## Use Case 1: Ask a Question

**Goal:** Send a single question and get an answer.

```bash
zeroclaw agent -m "Explain what a Rust trait is in simple terms"
```

**What happens:**
1. ZeroClaw sends your question to the AI model
2. The model generates a response
3. The response is printed to your terminal

**Try these:**
```bash
zeroclaw agent -m "Write a haiku about programming"
zeroclaw agent -m "What are the main differences between Python and Rust?"
zeroclaw agent -m "Explain TCP/IP to a 10-year-old"
```

---

## Use Case 2: Interactive Chat

**Goal:** Have a back-and-forth conversation with the AI.

```bash
zeroclaw agent
```

This opens an interactive session. Type your messages and press Enter.
The AI remembers the conversation context.

```
You: Hi! I'm learning Rust.
Assistant: Welcome! Rust is a great choice. What aspect would you like to
start with?

You: What are ownership and borrowing?
Assistant: Great question! Ownership is Rust's core memory management concept...

You: Can you give me a simple example?
Assistant: Sure! Here's a basic example...
```

**To exit:** Press Ctrl+C or type "exit".

**Tip:** Use `--temperature` to control creativity:
```bash
zeroclaw agent --temperature 0.0    # Very precise, deterministic
zeroclaw agent --temperature 1.5    # More creative, varied
```

---

## Use Case 3: Let the Agent Read Your Files

**Goal:** Ask the agent to read and analyse files in your workspace.

```bash
zeroclaw agent -m "Read the file Cargo.toml and tell me what dependencies this project uses"
```

The agent uses the `file_read` tool to read the file, then summarises it.

**Try these:**
```bash
zeroclaw agent -m "Read src/main.rs and explain what the main function does"
zeroclaw agent -m "List all the .rs files in the src directory"
zeroclaw agent -m "Read my README.md and suggest improvements"
```

**Security note:** By default, the agent can only read files inside the
workspace directory (~/.zeroclaw/workspace). This prevents accidental
access to sensitive files.

---

## Use Case 4: Let the Agent Write Code for You

**Goal:** Ask the agent to create or modify files.

```bash
zeroclaw agent -m "Create a file called hello.py with a Python hello world program"
```

The agent uses the `file_write` tool to create the file.

**Try these:**
```bash
zeroclaw agent -m "Create a file called todo.md with a sample to-do list"
zeroclaw agent -m "Write a Bash script that backs up a directory"
zeroclaw agent -m "Create an HTML file with a simple webpage"
```

---

## Use Case 5: Run Shell Commands Through the Agent

**Goal:** Let the agent run commands on your system.

```bash
zeroclaw agent -m "Run 'ls -la' and tell me what files are here"
zeroclaw agent -m "Check the current git status"
zeroclaw agent -m "Run 'cargo test' and summarise the results"
```

**What the agent can run (by default):**
- `git` (status, log, diff)
- `cargo` (build, test, fmt)
- `npm` (install, test)
- `ls`, `cat`, `grep`, `find`, `echo`, `pwd`, `wc`, `head`, `tail`

**What is blocked (for safety):**
- `rm`, `sudo`, `curl`, `wget`, `ssh`, `chmod`, `chown`
- Anything with `>` (output redirection)
- Anything with backticks or `$(...)`

You can customise the allowlist in your config file.

---

## Use Case 6: Store and Recall Memories

**Goal:** Make the agent remember things across conversations.

```bash
# Store a memory
zeroclaw agent -m "Remember that my preferred programming language is Rust"

# In a later session, recall it
zeroclaw agent -m "What is my preferred programming language?"
```

**How it works:**
- The agent uses `memory_store` to save information
- The agent uses `memory_recall` to search for information
- Memories persist in a SQLite database (~/.zeroclaw/brain.db)
- Search uses both keyword matching and semantic similarity

**Try these:**
```bash
zeroclaw agent -m "Remember: project deadline is March 15th"
zeroclaw agent -m "Remember: the database password is stored in .env"
zeroclaw agent -m "What do you remember about my project?"
```

---

## Use Case 7: Connect to Telegram

**Goal:** Chat with your AI agent through Telegram.

### Step 1: Create a Telegram Bot

1. Open Telegram and search for `@BotFather`
2. Send `/newbot`
3. Choose a name (e.g., "My ZeroClaw Bot")
4. Choose a username (e.g., "my_zeroclaw_bot")
5. BotFather gives you a token like: `7123456789:AAH...`

### Step 2: Configure ZeroClaw

```bash
zeroclaw onboard --channels-only
```

Or edit `~/.zeroclaw/config.toml`:

```toml
[channels_config.telegram]
bot_token = "7123456789:AAH..."
allowed_users = ["your_telegram_username"]
```

### Step 3: Start Channels

```bash
zeroclaw channel start
```

### Step 4: Chat

Open Telegram, find your bot, and send a message. The agent responds.

**Tip:** To find your Telegram username or ID:
1. Start channels with an empty allowlist
2. Send a message to your bot
3. Check the logs for the sender ID
4. Add it to `allowed_users`

Or use the quick bind command:
```bash
zeroclaw channel bind-telegram your_username
```

---

## Use Case 8: Schedule Recurring Tasks

**Goal:** Have the agent run tasks on a schedule.

```bash
# List current scheduled tasks
zeroclaw cron list

# Add a task that runs every hour
zeroclaw cron add "0 * * * *" "check system health"

# Add a task that runs at 9 AM every day
zeroclaw cron add "0 9 * * *" "summarise yesterday's git commits"

# Add a task with timezone
zeroclaw cron add "0 9 * * *" --tz "America/New_York" "good morning report"

# Add a one-shot task at a specific time
zeroclaw cron add-at "2026-03-15T10:00:00Z" "project deadline reminder"

# Add a task that runs every 30 minutes
zeroclaw cron add-every 1800000 "check for new messages"

# Add a one-shot delayed task
zeroclaw cron once "30m" "remind me to take a break"

# Remove a task
zeroclaw cron remove TASK_ID

# Pause/resume a task
zeroclaw cron pause TASK_ID
zeroclaw cron resume TASK_ID
```

**Cron expression cheat sheet:**
```
* * * * *
| | | | |
| | | | +-- Day of week (0-7, 0=Sunday)
| | | +---- Month (1-12)
| | +------ Day of month (1-31)
| +-------- Hour (0-23)
+---------- Minute (0-59)

Examples:
  "0 * * * *"      Every hour
  "*/15 * * * *"   Every 15 minutes
  "0 9 * * *"      Every day at 9 AM
  "0 9 * * 1-5"    Weekdays at 9 AM
  "0 0 1 * *"      First of every month at midnight
```

---

## Use Case 9: Check System Status and Health

**Goal:** Verify everything is working correctly.

```bash
# Full system status
zeroclaw status

# Run diagnostics
zeroclaw doctor

# Check channel health
zeroclaw channel doctor

# List configured channels
zeroclaw channel list

# Check auth status
zeroclaw auth status

# List all supported providers
zeroclaw providers

# Get info about a specific integration
zeroclaw integrations info Telegram
```

**Example output of `zeroclaw status`:**
```
ZeroClaw Status

Version:     0.1.0
Workspace:   /home/user/.zeroclaw/workspace
Config:      /home/user/.zeroclaw/config.toml

Provider:      openrouter
   Model:      anthropic/claude-sonnet-4-20250514
Observability: log
Autonomy:      Supervised
Runtime:       native
Heartbeat:     disabled
Memory:        sqlite (auto-save: on)

Security:
  Workspace only:    true
  Allowed commands:  git, npm, cargo, ls, cat, grep
  Max actions/hour:  1000
  Max cost/day:      $50.00

Channels:
  CLI:       always
  Telegram:  configured
  Discord:   not configured
  Slack:     not configured
```

---

## Use Case 10: Use a Local AI Model (Ollama)

**Goal:** Run everything locally with no cloud dependency.

### Step 1: Install Ollama

Visit https://ollama.com and download for your platform.

### Step 2: Pull a Model

```bash
ollama pull llama3.2          # 2 GB, good for general use
ollama pull codellama:7b      # 4 GB, better for coding
ollama pull mistral:7b        # 4 GB, good all-rounder
```

### Step 3: Configure ZeroClaw

```bash
zeroclaw onboard --provider ollama
```

Or edit `~/.zeroclaw/config.toml`:
```toml
default_provider = "ollama"
default_model = "llama3.2"
```

### Step 4: Chat

```bash
zeroclaw agent -m "Hello! Are you running locally?"
```

**Tip:** No API key needed. No internet connection needed after model
download. Your conversations never leave your machine.

---

## Use Case 11: Run as a Background Service

**Goal:** Keep ZeroClaw running all the time so it can respond to messages
on Telegram/Discord/etc. even when you are not at the terminal.

### Option A: Daemon Mode (manual)

```bash
zeroclaw daemon
```

This starts:
- Gateway (HTTP webhook server)
- All configured channels (Telegram, Discord, etc.)
- Cron scheduler
- Heartbeat (if enabled)

Press Ctrl+C to stop.

### Option B: System Service (auto-start on boot)

```bash
# Install as a system service
zeroclaw service install

# Start the service
zeroclaw service start

# Check status
zeroclaw service status

# Stop the service
zeroclaw service stop

# Remove the service
zeroclaw service uninstall
```

On Linux, this creates a systemd user service.
On macOS, this creates a launchd user agent.

---

## Use Case 12: Migrate from Another Agent

**Goal:** Import your data from OpenClaw to ZeroClaw.

```bash
# Preview what would be migrated (safe, no changes)
zeroclaw migrate openclaw --dry-run

# Perform the migration
zeroclaw migrate openclaw

# Specify a custom source directory
zeroclaw migrate openclaw --source /path/to/openclaw/workspace
```

---

## 17. Configuration Reference

Your configuration file is at: `~/.zeroclaw/config.toml`

### Essential Settings

```toml
# API key for your chosen provider
api_key = "sk-..."

# Which AI provider to use
default_provider = "openrouter"    # See providers list below

# Which model to use
default_model = "anthropic/claude-sonnet-4-20250514"

# Response creativity (0.0 = precise, 2.0 = creative)
default_temperature = 0.7
```

### Memory Settings

```toml
[memory]
backend = "sqlite"              # "sqlite", "markdown", "none"
auto_save = true                # Automatically save conversations
embedding_provider = "openai"   # For semantic search
vector_weight = 0.7             # Weight for semantic search
keyword_weight = 0.3            # Weight for keyword search
```

### Security Settings

```toml
[autonomy]
level = "supervised"            # "readonly", "supervised", "full"
workspace_only = true           # Only access workspace directory
allowed_commands = ["git", "npm", "cargo", "ls", "cat", "grep"]
max_actions_per_hour = 1000
max_cost_per_day_cents = 5000   # $50.00 daily cap
```

### Gateway Settings

```toml
[gateway]
host = "127.0.0.1"             # Only accept local connections
port = 8080
require_pairing = true          # Require pairing code
allow_public_bind = false       # Refuse 0.0.0.0
```

### Channel Settings

```toml
[channels_config.telegram]
bot_token = "7123456789:AAH..."
allowed_users = ["username1", "123456789"]

[channels_config.discord]
bot_token = "MTIz..."
allowed_users = ["user_id_1"]

[channels_config.slack]
bot_token = "xoxb-..."
app_token = "xapp-..."
allowed_users = ["U12345678"]
```

### Environment Variables

You can override config with environment variables:

```bash
ZEROCLAW_API_KEY=sk-...         # Override api_key
OPENAI_API_KEY=sk-...           # OpenAI-specific
ANTHROPIC_API_KEY=sk-ant-...    # Anthropic-specific
OLLAMA_API_KEY=...              # Ollama remote
RUST_LOG=debug                  # Logging level
```

---

## 18. Providers Reference

ZeroClaw supports 23+ AI providers. Use `zeroclaw providers` to see all.

| Provider | Config Key | Notes |
|----------|-----------|-------|
| OpenRouter | `openrouter` | Access to 23+ models with one key |
| OpenAI | `openai` | GPT-4, GPT-3.5, etc. |
| Anthropic | `anthropic` | Claude models |
| Ollama | `ollama` | Local models, no API key needed |
| Gemini | `gemini` | Google Gemini models |
| GitHub Copilot | `copilot` | GitHub subscription |
| OpenAI Codex | `openai-codex` | ChatGPT subscription auth |
| xAI / Grok | `xai` | Grok models |
| DeepSeek | `deepseek` | DeepSeek models |
| Groq | `groq` | Fast inference |
| Together | `together` | Open-source models |
| Fireworks | `fireworks` | Fast inference |
| Mistral | `mistral` | Mistral models |
| Cohere | `cohere` | Cohere models |
| Perplexity | `perplexity` | Search-augmented |
| Venice | `venice` | Privacy-focused |
| Any OpenAI-compatible | `custom:URL` | Any endpoint |

---

## 19. Channels Reference

| Channel | Config Section | Setup |
|---------|---------------|-------|
| CLI | Always on | Built-in terminal |
| Telegram | `channels_config.telegram` | Create bot via @BotFather |
| Discord | `channels_config.discord` | Create app at discord.com/developers |
| Slack | `channels_config.slack` | Create app at api.slack.com |
| WhatsApp | `channels_config.whatsapp` | Meta Business Cloud API |
| Mattermost | `channels_config.mattermost` | Self-hosted team chat |
| Matrix | `channels_config.matrix` | Decentralised messaging |
| Email | `channels_config.email` | SMTP/IMAP |
| IRC | `channels_config.irc` | IRC networks |
| Signal | `channels_config.signal` | Signal protocol bridge |
| iMessage | `channels_config.imessage` | macOS only |
| DingTalk | `channels_config.dingtalk` | Alibaba |
| Lark | `channels_config.lark` | ByteDance |

---

## 20. Tools Reference

These are the capabilities available to the AI agent:

| Tool | What It Does |
|------|-------------|
| `shell` | Run shell commands (allowlist-gated) |
| `file_read` | Read file contents |
| `file_write` | Create or modify files |
| `memory_store` | Save information to memory |
| `memory_recall` | Search memory (keyword + semantic) |
| `memory_forget` | Delete a memory entry |
| `browser_open` | Open a URL (domain-allowlisted) |
| `browser` | Full browser automation |
| `http_request` | Make HTTP requests |
| `screenshot` | Take a screenshot |
| `git_operations` | Run git commands |
| `delegate` | Delegate a task to a sub-agent |
| `composio` | Access 1000+ OAuth apps |
| `cron_add` | Schedule a recurring task |
| `cron_list` | List scheduled tasks |
| `cron_remove` | Remove a scheduled task |
| `pushover` | Send push notifications |

---

## 21. Security Reference

ZeroClaw is secure by default. Here is what that means:

### Autonomy Levels

| Level | What the Agent Can Do |
|-------|----------------------|
| `readonly` | Read files and memory only. Cannot run commands or write files. |
| `supervised` | Can act within allowlists. Risky commands need approval. |
| `full` | Full access within workspace sandbox. |

### What is Blocked by Default

- **Commands:** rm, sudo, curl, wget, ssh, and any command not in the allowlist
- **Paths:** /etc, /root, ~/.ssh, ~/.aws, ~/.gnupg, and all system directories
- **Injection:** Backticks, $(), output redirection (>), background (&)
- **Network:** Gateway only binds to localhost (127.0.0.1)
- **Rate:** Maximum 1000 actions per hour, $50/day cost cap

### Encrypted Secrets

Your API keys are encrypted at rest using ChaCha20-Poly1305 encryption.
The encryption key is stored in `~/.zeroclaw/.secret_key`.

---

## 22. Troubleshooting

### "command not found: zeroclaw"

Your PATH does not include `~/.cargo/bin`. Fix:
```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

### "No config file found"

Run the setup wizard:
```bash
zeroclaw onboard --interactive
```

### "API key invalid" or "401 Unauthorized"

Check your API key:
```bash
zeroclaw status    # Shows current provider and key status
```

Update your key:
```bash
zeroclaw onboard --api-key NEW_KEY --provider openrouter
```

### Telegram bot not responding

1. Check channel health: `zeroclaw channel doctor`
2. Check if your username is in the allowlist
3. Enable debug logs: `RUST_LOG=debug zeroclaw channel start`
4. Rebind your identity: `zeroclaw channel bind-telegram YOUR_USERNAME`

### Build fails

```bash
git pull
cargo build --release --locked
```

### Out of memory during build

Use the standard release profile (slower but less memory):
```bash
cargo build --release    # Uses codegen-units=1 (needs ~1 GB RAM)
```

### "Cannot connect to Ollama"

Make sure Ollama is running:
```bash
ollama serve    # Start Ollama server
```

---

## 23. FAQ

**Q: Is ZeroClaw free?**
A: ZeroClaw itself is free and open source (MIT license). You may need
to pay for AI model API access, but you can also use free local models
via Ollama.

**Q: Does my data leave my computer?**
A: Only the messages you send to cloud AI providers. If you use Ollama
(local), nothing leaves your machine. Memory is always stored locally.

**Q: Can the agent delete my files?**
A: By default, no. The `rm` command is blocked, and file operations are
restricted to the workspace directory. You can change this in the config.

**Q: How do I change the AI model?**
A: Edit `~/.zeroclaw/config.toml` and change `default_model`, or use the
command-line flag: `zeroclaw agent --model gpt-4o`

**Q: Can I use multiple AI providers?**
A: Yes. You can switch providers per command:
```bash
zeroclaw agent --provider openai -m "Hello"
zeroclaw agent --provider anthropic -m "Hello"
zeroclaw agent --provider ollama -m "Hello"
```

**Q: How much disk space does ZeroClaw use?**
A: The binary is ~3.4 MB. The memory database grows with usage but
typically stays under 100 MB even with heavy use.

**Q: Can I run ZeroClaw on a Raspberry Pi?**
A: Yes. ZeroClaw is designed for low-resource devices. It runs on
Raspberry Pi 3+ with <5 MB RAM.

---

For architecture details, see `docs/ARCHITECTURE.md`.
For developer documentation, see `docs/DEVELOPER_GUIDE.md`.
