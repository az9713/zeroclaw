# ZeroClaw Technology Stack

A complete guide to every technology in ZeroClaw — written for readers with zero prior exposure
to any of these tools. Each entry explains what the technology is, why it exists, what problem
it solves for ZeroClaw specifically, and where to find it in the code.

---

## Technology Map

The diagram below shows how ZeroClaw's layers relate to each other. Read it top-down: the
top layers are what users touch; the bottom layers are what the machine touches.

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  USER INTERFACES                                                        │
  │  clap (CLI parsing)  +  dialoguer/console (interactive prompts)        │
  └─────────────────────────────┬───────────────────────────────────────────┘
                                │
  ┌─────────────────────────────▼───────────────────────────────────────────┐
  │  AGENT ORCHESTRATION LAYER                                              │
  │  Rust + Tokio (async runtime) + async-trait                            │
  └──────┬──────────────┬────────────────────────────┬──────────────────────┘
         │              │                            │
  ┌──────▼──────┐  ┌────▼──────────────────────┐  ┌─▼──────────────────────┐
  │  PROVIDERS  │  │  CHANNELS                  │  │  TOOLS / PERIPHERALS   │
  │  reqwest    │  │  axum (gateway server)     │  │  tokio-serial (serial) │
  │  (HTTP out) │  │  tokio-tungstenite (WS)    │  │  nusb (USB)            │
  │  ring (JWT) │  │  lettre (email out)        │  │  rppal (RPi GPIO)      │
  │  prost (pb) │  │  mail-parser (email in)    │  │  probe-rs (STM32)      │
  └──────┬──────┘  │  rustls/TLS                │  └────────────────────────┘
         │         └────────────────────────────┘
  ┌──────▼──────────────────────────────────────────────────────────────────┐
  │  SECURITY LAYER                                                         │
  │  chacha20poly1305 (secrets)  hmac+sha2 (webhooks)  rand (tokens)       │
  └──────┬──────────────────────────────────────────────────────────────────┘
         │
  ┌──────▼──────────────────────────────────────────────────────────────────┐
  │  MEMORY / STORAGE LAYER                                                 │
  │  rusqlite + FTS5 (keyword search) + vector BLOBs (semantic search)     │
  │  serde / serde_json (serialization)  toml (config)  chrono (time)      │
  └──────┬──────────────────────────────────────────────────────────────────┘
         │
  ┌──────▼──────────────────────────────────────────────────────────────────┐
  │  OBSERVABILITY LAYER                                                    │
  │  tracing / tracing-subscriber  opentelemetry-otlp  prometheus          │
  └──────┬──────────────────────────────────────────────────────────────────┘
         │
  ┌──────▼──────────────────────────────────────────────────────────────────┐
  │  BUILD, CI/CD, QUALITY                                                  │
  │  Cargo  Docker  GitHub Actions  Blacksmith  CodeQL  Dependabot          │
  │  clippy  rustfmt  cargo-fuzz  cargo-deny  lychee  markdownlint          │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Core Language and Runtime](#1-core-language-and-runtime)
2. [CLI Framework](#2-cli-framework)
3. [HTTP and Networking](#3-http-and-networking)
4. [Serialization and Data Formats](#4-serialization-and-data-formats)
5. [Database and Storage](#5-database-and-storage)
6. [Cryptography and Security](#6-cryptography-and-security)
7. [Observability and Logging](#7-observability-and-logging)
8. [Async and Concurrency Primitives](#8-async-and-concurrency-primitives)
9. [Time and Scheduling](#9-time-and-scheduling)
10. [Email](#10-email)
11. [Hardware and Peripherals](#11-hardware-and-peripherals)
12. [Testing and Quality](#12-testing-and-quality)
13. [Containerization and Deployment](#13-containerization-and-deployment)
14. [CI/CD and GitHub Tooling](#14-cicd-and-github-tooling)
15. [Python Companion Package](#15-python-companion-package)
16. [Error Handling](#16-error-handling)
17. [Other Utilities](#17-other-utilities)

---

## 1. Core Language and Runtime

### Rust (edition 2021, toolchain 1.92.0)

**What it is.**
Rust is a systems programming language created by Mozilla Research and now maintained by the
Rust Foundation. It compiles to native machine code — the same kind of code the CPU executes
directly, with no interpreter or virtual machine in the way. Rust's defining property is
*memory safety without garbage collection*.

To understand why that matters, consider the two most common ways programming languages prevent
crashes from bad memory use:

- Garbage-collected languages (Python, Go, Java) run a background process that automatically
  frees memory when it is no longer needed. This is safe but adds CPU overhead and unpredictable
  pauses called GC pauses.
- C and C++ give you full manual control, which is fast but requires the programmer to free
  every allocation correctly. A missed `free()` leaks memory; a double `free()` crashes the
  program; accessing freed memory is a security vulnerability.

Rust takes a third path: a system called *ownership* enforced at compile time by the Rust
compiler. The compiler tracks which part of the code "owns" each piece of memory and
automatically inserts the correct cleanup code. If you write code that would cause a
use-after-free or a data race, the program simply will not compile. The safety guarantee is
free — it costs zero runtime overhead.

**Why ZeroClaw uses it.**
ZeroClaw aims for the smallest possible binary size, the lowest memory footprint, and the
ability to run on resource-constrained hardware like a Raspberry Pi 3 with 1 GB of RAM. Its
security surfaces (gateway, tools, secret store) handle real-world sensitive data. Rust
satisfies all of these requirements simultaneously: production-grade safety, small binaries
(around 3–5 MB stripped), fast startup (under 10 ms), and no runtime to attack.

The `edition = "2021"` in `Cargo.toml` selects a specific version of the Rust language
specification. Editions are backwards-compatible snapshots that let the language evolve without
breaking existing code. Edition 2021 brought cleaner closures and more ergonomic pattern
matching, both used throughout ZeroClaw's agent loop.

The `rust-toolchain.toml` file pins the compiler to version `1.92.0`. This ensures every
developer and every CI machine uses exactly the same compiler, making builds reproducible.

**Where in the code.**
Every `.rs` file is Rust. The toolchain pin is at `rust-toolchain.toml`.

**Learn more.** https://www.rust-lang.org/learn

---

### Cargo

**What it is.**
Cargo is Rust's official build system and package manager, similar in role to `npm` for
JavaScript or `pip` for Python. It handles downloading dependencies, compiling code, running
tests, and building release binaries.

Key files:

- `Cargo.toml` — the project manifest. It declares the package name and version, lists
  dependencies (called *crates*), defines *features* (optional compilable pieces), and
  configures *profiles* (how to compile for different purposes).
- `Cargo.lock` — a lockfile that records the exact version of every dependency that was used
  in the last successful build. This is committed to the repository so that any clone produces
  an identical binary.

**Features.**
A Cargo feature is a named compile-time switch. When a feature is disabled, its code is not
compiled at all, shrinking the binary. ZeroClaw uses features extensively to keep the default
binary small:

```
hardware     = ["nusb", "tokio-serial"]          -- USB and serial support (on by default)
peripheral-rpi = ["rppal"]                        -- Raspberry Pi GPIO (Linux only)
browser-native = ["dep:fantoccini"]               -- Native browser automation
sandbox-landlock = ["dep:landlock"]               -- Linux Landlock sandboxing
probe         = ["dep:probe-rs"]                  -- STM32 debug probe
rag-pdf       = ["dep:pdf-extract"]               -- PDF ingestion
```

**Build profiles.**
A profile controls how the compiler optimizes the output:

- `release` — optimizes for the smallest possible binary (`opt-level = "z"`), strips debug
  symbols, uses Link Time Optimization (LTO) to remove dead code across crate boundaries,
  and replaces Rust's default stack-unwinding on panic with an immediate abort. This is the
  profile used for production deployments.
- `release-fast` — inherits `release` but uses 8 parallel code-generation units for faster
  compilation on powerful machines. Slightly larger binary in exchange for faster build times.
- `dist` — uses "fat" LTO, which is the most aggressive cross-crate dead-code elimination.
  Used for the final distribution binary where size is the top priority.

**Where in the code.** `Cargo.toml`, `Cargo.lock`.

**Learn more.** https://doc.rust-lang.org/cargo/

---

### Tokio (1.42)

**What it is.**
Tokio is an *asynchronous runtime* for Rust. To understand what that means, consider the
problem it solves.

When an agent calls an LLM API, it sends a network request and then waits — sometimes for
several seconds — for the response. During that wait, the CPU is doing nothing useful. If
ZeroClaw handled one request at a time (synchronously), it could serve only one Telegram
message at a time, one webhook at a time, and so on. Every user would have to wait in line.

Asynchronous programming lets a single thread handle many concurrent tasks. Instead of
blocking (sitting idle) while waiting for a network response, the program pauses that task,
switches to another ready task, and returns to the original task when its response arrives.
This is called cooperative multitasking.

In Rust, asynchronous code is written with `async` and `await` keywords. A function marked
`async fn` does not execute immediately; it returns a *Future* — a description of work to be
done. Futures do nothing until a runtime drives them. Tokio is that runtime. It maintains
a pool of worker threads, schedules Futures across them, handles I/O readiness from the
operating system, and wakes tasks when their awaited operations complete.

The features enabled for Tokio are precisely chosen to avoid compiling unused code:
`rt-multi-thread` (multi-core scheduling), `macros` (the `#[tokio::main]` annotation),
`time` (async timers), `net` (TCP/UDP), `io-util`, `sync` (channels, mutexes), `process`,
`io-std`, `fs` (async file I/O), `signal`.

**Why ZeroClaw uses it.**
ZeroClaw runs a gateway server, multiple chat channel listeners, a heartbeat scheduler, a
cron scheduler, and LLM API calls simultaneously. All of these involve waiting on network I/O.
Tokio lets a small number of threads handle all of this concurrency efficiently, keeping
memory usage low and latency high.

**Where in the code.**
`src/main.rs` uses `#[tokio::main]` to start the runtime. Every `async fn` in the codebase
runs on Tokio. The gateway (`src/gateway/mod.rs`), agent loop (`src/agent/loop_.rs`), and
channel listeners (`src/channels/*.rs`) all rely on Tokio for scheduling.

**Learn more.** https://tokio.rs

---

## 2. CLI Framework

### clap (4.5, derive feature)

**What it is.**
A CLI (Command-Line Interface) is the text-based interface you use when you type commands
into a terminal. `clap` is a library that makes building CLIs in Rust easy by turning ordinary
Rust structs and enums into argument parsers.

Without clap, you would have to manually inspect `std::env::args()`, parse `--flag value`
pairs, handle missing arguments, generate `--help` text, and validate types by hand. clap does
all of this automatically.

The `derive` feature lets you annotate a struct with `#[derive(Parser)]`. clap reads those
annotations and generates the complete argument parser at compile time. Adding a new CLI flag
is as simple as adding a new field to the struct.

**Why ZeroClaw uses it.**
ZeroClaw exposes over a dozen subcommands (`onboard`, `agent`, `gateway`, `daemon`, `cron`,
`channel`, `hardware`, `peripheral`, etc.), each with their own flags. clap turns the `Cli`
struct and `Commands` enum in `src/main.rs` directly into a fully-featured CLI with
auto-generated `--help`, subcommand routing, type validation, and shell completion.

**Where in the code.**
`src/main.rs` — the `Cli` struct, `Commands` enum, and all subcommand enums are annotated
with clap's derive macros.

**Learn more.** https://docs.rs/clap

---

### dialoguer (0.12) + console (0.16)

**What they are.**
`dialoguer` is a library for interactive terminal prompts: text input boxes, password fields
(where typed characters are hidden), selection menus, and fuzzy-search pickers.

`console` is a library for terminal styling and control: colored output, cursor movement,
clearing lines, and detecting whether stdout is a real terminal or a pipe (so you can skip
colors when output is being redirected to a file).

**Why ZeroClaw uses them.**
The `zeroclaw onboard` wizard and the `zeroclaw auth` commands ask the user for API keys,
provider names, and memory backend choices interactively. `dialoguer`'s `Password` widget
hides the input so API keys are not visible on screen. The `Input` widget collects plain text.
`console` powers the styled output in the terminal (bold headers, colored status lines).

**Where in the code.**
`src/main.rs` (`read_auth_input`, `read_plain_input`), `src/onboard/wizard.rs`.

**Learn more.**
https://docs.rs/dialoguer
https://docs.rs/console

---

## 3. HTTP and Networking

### reqwest (0.12)

**What it is.**
HTTP (HyperText Transfer Protocol) is the protocol your browser uses to fetch web pages. An
HTTP *client* sends requests to a server and receives responses. `reqwest` is Rust's most
popular HTTP client library.

When ZeroClaw sends a message to the OpenAI API, it sends an HTTP POST request with a JSON
body containing the conversation history and model parameters. The response is an HTTP reply
with a JSON body containing the model's reply. `reqwest` handles all the low-level details:
TCP connection setup, TLS handshake, sending the request bytes, reading the response bytes,
and decoding the response.

The features enabled are: `json` (automatic JSON serialization/deserialization), `rustls-tls`
(encrypted connections without depending on OpenSSL), `blocking` (synchronous mode used in
the onboarding wizard), `multipart` (for multipart form uploads), `stream` (for streaming
responses token by token).

**Why ZeroClaw uses it.**
Every LLM provider (OpenAI, Anthropic, Gemini, Ollama, etc.) communicates over HTTP. reqwest
is the bridge between ZeroClaw's Rust code and those external APIs. It is also used by the
Discord and Telegram channels to call their REST APIs.

**Where in the code.**
`src/providers/openai.rs`, `src/providers/anthropic.rs`, `src/providers/gemini.rs`, and all
other provider implementations. `src/channels/discord.rs`, `src/channels/telegram.rs`.

**Learn more.** https://docs.rs/reqwest

---

### axum (0.8)

**What it is.**
While reqwest is an HTTP *client* (ZeroClaw talking to external services), axum is an HTTP
*server* framework (external services talking to ZeroClaw).

A web server framework provides the scaffolding for receiving HTTP requests, routing them to
the right handler function based on the URL path and method, parsing request bodies, and
sending responses. Without a framework you would need to parse raw HTTP bytes yourself, which
is error-prone and complex.

axum is built on top of Tokio and integrates naturally with async Rust. It uses a *router*
to map URL patterns to handler functions. Handler functions are ordinary async Rust functions
that receive typed parameters extracted from the request.

The features enabled are: `http1` (HTTP/1.1), `json` (automatic JSON request/response), `tokio`
(async integration), `query` (URL query parameter parsing), `ws` (WebSocket upgrade support).

**Why ZeroClaw uses it.**
ZeroClaw's gateway server receives webhooks from Telegram, Slack, Discord, and custom
integrations. It also serves the `/metrics` endpoint for Prometheus scraping and the
`/health` endpoint for load balancer health checks. axum provides proper HTTP/1.1 compliance
with body size limits (64 KB maximum), request timeouts (30 seconds) to prevent slow-loris
attacks, and a clean routing table.

**Where in the code.**
`src/gateway/mod.rs` — the entire gateway is built with axum's `Router`, handlers, and
extractors.

**Learn more.** https://docs.rs/axum

---

### tower (0.5) + tower-http (0.6)

**What they are.**
tower is a framework for composable *middleware* — reusable pieces of logic that wrap around
a service to add behavior. Think of middleware as layers in a stack: a request passes through
each layer before reaching the handler, and the response passes back through each layer in
reverse.

```
  Request
     |
  [timeout layer]      -- abort request if it takes too long
     |
  [body limit layer]   -- reject requests with bodies that are too large
     |
  [your handler]       -- the actual business logic
     |
  Response
```

tower-http provides HTTP-specific middleware built on tower: request body size limits,
timeouts, CORS headers, request tracing, and more.

**Why ZeroClaw uses them.**
The gateway wraps all routes with `RequestBodyLimitLayer` (64 KB limit) and `TimeoutLayer`
(30-second cutoff). These middleware layers prevent denial-of-service attacks where a
malicious client sends a huge body or deliberately holds a connection open. With tower, adding
these protections is a one-liner rather than defensive code scattered through every handler.

**Where in the code.**
`src/gateway/mod.rs` — `tower_http::limit::RequestBodyLimitLayer` and
`tower_http::timeout::TimeoutLayer` are applied to the router.

**Learn more.**
https://docs.rs/tower
https://docs.rs/tower-http

---

### tokio-tungstenite (0.24)

**What it is.**
WebSocket is a protocol that provides a persistent, full-duplex communication channel between
a client and a server. Unlike HTTP (where the client sends a request and the server sends a
response and then the connection may close), a WebSocket connection stays open. Either side
can send messages to the other at any time. This is essential for real-time applications.

tungstenite is a WebSocket library for Rust. tokio-tungstenite wraps it with Tokio's async
I/O so WebSocket connections can be driven without blocking threads.

**Why ZeroClaw uses it.**
Discord and Lark (Feishu) deliver real-time messages via WebSocket. ZeroClaw must maintain
a persistent WebSocket connection to Discord's Gateway API to receive events (new messages,
reactions, etc.) in real time. The Lark channel uses the same mechanism for Feishu's
long-connection mode. Without WebSocket, ZeroClaw would have to poll the APIs repeatedly,
which is slower, consumes more API quota, and is often prohibited by the providers.

**Where in the code.**
`src/channels/discord.rs`, `src/channels/lark.rs`.

**Learn more.** https://docs.rs/tokio-tungstenite

---

### rustls (0.23) + tokio-rustls + webpki-roots

**What they are.**
TLS (Transport Layer Security) is the protocol that makes HTTPS secure. When your browser
connects to `https://api.openai.com`, TLS encrypts the connection so that network observers
cannot read the API key or response bodies passing between your machine and OpenAI's servers.
TLS also authenticates the server — it proves you are talking to the real OpenAI, not an
impostor.

`rustls` is a TLS implementation written entirely in Rust. It is an alternative to OpenSSL,
the traditional TLS library written in C. Using rustls means ZeroClaw gets TLS without
depending on the system's OpenSSL installation, which varies across operating systems, is a
historically common source of vulnerabilities, and is a pain to cross-compile.

`tokio-rustls` integrates rustls with Tokio's async I/O. `webpki-roots` provides the
standard set of trusted certificate authorities (CAs) — the list of organizations whose
certificates prove a server is legitimate — bundled directly into the binary, so ZeroClaw
does not depend on the operating system's certificate store.

**Why ZeroClaw uses them.**
All outbound API calls (to LLM providers), all webhook servers, and all channel connections
require TLS. rustls provides this entirely in Rust, keeping the dependency chain clean and the
binary self-contained. The `rustls::crypto::ring::default_provider().install_default()` call
in `src/main.rs` initializes the cryptographic backend before any TLS is attempted.

**Where in the code.**
`src/main.rs` (crypto provider initialization), throughout `src/channels/` and
`src/providers/` via reqwest's and tungstenite's rustls TLS features.

**Learn more.** https://docs.rs/rustls

---

## 4. Serialization and Data Formats

### serde (1.0) + serde_json (1.0)

**What they are.**
Serialization is the process of converting a data structure in memory into a format that can
be stored or transmitted. Deserialization is the reverse — converting that stored format back
into a live data structure. A simple example: converting a Rust struct `User { name: "Alice",
age: 30 }` into the JSON string `{"name":"Alice","age":30}`, and back again.

`serde` is Rust's serialization framework. It is not tied to any particular format. Instead,
it defines a common interface that data types can implement once, and then any serde-compatible
format (JSON, TOML, MessagePack, etc.) can serialize or deserialize that type.

`serde_json` is the JSON backend for serde. It handles reading and writing JSON, the dominant
format for web APIs.

The `derive` feature of serde allows annotating a struct with `#[derive(Serialize, Deserialize)]`,
and serde generates the serialization code at compile time. No runtime overhead, no reflection.

**Why ZeroClaw uses them.**
Every LLM API uses JSON for request and response bodies. Every webhook payload is JSON. Every
config file value that gets stored in memory is serialized as JSON. serde + serde_json are
the glue between Rust's type system and the JSON-speaking world of external APIs.

**Where in the code.**
Virtually every source file. Key examples: `src/providers/openai.rs` (request/response
structs), `src/config/schema.rs` (config deserialization), `src/memory/sqlite.rs` (storing
and loading memory entries), `src/security/secrets.rs` (serializing encrypted values).

**Learn more.**
https://serde.rs
https://docs.rs/serde_json

---

### toml (1.0)

**What it is.**
TOML (Tom's Obvious Minimal Language) is a configuration file format designed to be easy for
humans to read and write. A comparison:

```toml
# TOML — clear key = value pairs with sections
[gateway]
port = 3000
host = "[::]"

[memory]
backend = "sqlite"
auto_save = true
```

```json
// JSON — the same data, harder to read/write by hand
{"gateway":{"port":3000,"host":"[::]"},"memory":{"backend":"sqlite","auto_save":true}}
```

```yaml
# YAML — similar to TOML but notorious for surprising edge cases
gateway:
  port: 3000
  host: "[::]"
memory:
  backend: sqlite
  auto_save: true
```

TOML is predictable. YAML has many ambiguities (the string `no` can be parsed as the boolean
`false`). JSON requires quoting all keys and does not support comments. For a user-edited
config file, TOML is the most practical choice.

**Why ZeroClaw uses it.**
ZeroClaw's main configuration file is `~/.zeroclaw/config.toml`. Users edit this file directly
to set their API key, default provider, gateway port, memory backend, and other settings. The
`toml` crate parses this file into the `Config` struct defined in `src/config/schema.rs`.

**Where in the code.**
`src/config/mod.rs` (loading and parsing), `src/config/schema.rs` (the struct definition),
`dev/config.template.toml` (the default template).

**Learn more.** https://toml.io

---

### prost (0.14)

**What it is.**
Protocol Buffers (protobuf) is a binary serialization format created by Google. Unlike JSON
(text-based, human-readable) or TOML (text-based, config-oriented), protobuf encodes data as
compact binary. A message that takes 100 bytes in JSON might take 30 bytes in protobuf.
Protobuf is also schema-driven: you define the message structure in a `.proto` file, and a
code generator produces typed structs in your language of choice.

`prost` is a protobuf library for Rust. It allows defining protobuf-compatible structs directly
in Rust code using derive macros, without a separate code generation step.

**Why ZeroClaw uses it.**
The Feishu/Lark messaging platform (a Chinese workplace communication tool similar to Slack)
uses a proprietary WebSocket protocol called pbbp2. Each frame sent over the WebSocket
connection is encoded as a protobuf message with a specific schema. ZeroClaw uses prost to
encode and decode these frames when communicating with Feishu's long-connection WebSocket API.

**Where in the code.**
`src/channels/lark.rs` — the `PbHeader` and `PbFrame` structs use `#[derive(prost::Message)]`
and prost's field annotations (`#[prost(string, tag = "1")]`).

**Learn more.** https://docs.rs/prost

---

## 5. Database and Storage

### rusqlite (0.38, bundled)

**What it is.**
SQLite is an embedded relational database. The key word is *embedded*: unlike PostgreSQL or
MySQL, which run as separate server processes that you must connect to over a network, SQLite
runs as a library linked directly into your application. The entire database — data, indexes,
and all — lives in a single file on disk. No server to install, no daemon to start, no
connection to manage.

This makes SQLite ideal for applications that need a real database (with transactions, indexes,
and query power) but cannot or should not depend on an external server. It is the most widely
deployed database engine in the world.

`rusqlite` is the Rust library for SQLite. The `bundled` feature compiles SQLite directly into
the ZeroClaw binary rather than linking against the system's SQLite. This makes ZeroClaw
self-contained — it works even on systems where SQLite is not installed or is the wrong version.

The database is stored at `~/.zeroclaw/workspace/memory/brain.db`.

**Why ZeroClaw uses it.**
ZeroClaw needs to remember things across sessions: conversations, user preferences, learned
facts, scheduled tasks, auth profiles. SQLite provides a transactional, crash-safe store for
all of this. It supports WAL (Write-Ahead Logging) mode, which allows concurrent reads while
a write is in progress — important when the agent loop is reading memory at the same time as
a background task writes to it.

The SQLite connection is opened with production-grade PRAGMA settings in `src/memory/sqlite.rs`:
WAL mode for concurrency, mmap for hot-page caching, and in-memory temp tables.

**Where in the code.**
`src/memory/sqlite.rs` — the main `SqliteMemory` implementation.
`src/cron/store.rs` — scheduled tasks are stored in SQLite.
`src/auth/profiles.rs` — auth profiles use SQLite-backed storage.

**Learn more.**
https://www.sqlite.org
https://docs.rs/rusqlite

---

### FTS5

**What it is.**
FTS5 is SQLite's built-in full-text search extension. Full-text search is different from
normal database queries. A normal SQL query like `WHERE content = 'Rust is fast'` only
finds exact matches. Full-text search finds documents that *contain* the query terms, ranks
them by relevance (how many times the terms appear, how prominently), and handles variations.

FTS5 uses the BM25 ranking algorithm, a widely used information retrieval algorithm that
weighs term frequency, document length, and corpus-wide term frequency to produce a relevance
score. A document containing the query word many times ranks higher than one that contains it
once.

ZeroClaw creates an FTS5 virtual table called `memories_fts` that mirrors the main `memories`
table. SQLite triggers keep them in sync automatically: when a memory is inserted, updated, or
deleted, the FTS index is updated in the same transaction.

**Why ZeroClaw uses it.**
When the agent asks "what do I remember about Rust?", ZeroClaw searches the memory database
for relevant entries. A plain SQL `LIKE '%Rust%'` query would be slow on large databases and
would not rank results by relevance. FTS5 makes keyword recall fast and relevance-ranked.

**Where in the code.**
`src/memory/sqlite.rs` — the `init_schema` function creates the `memories_fts` virtual table
and its triggers. The `fts5_search` method executes BM25-ranked keyword queries.

**Learn more.** https://www.sqlite.org/fts5.html

---

### Vector Storage (embeddings as BLOBs with cosine similarity)

**What it is.**
An *embedding* is a list of floating-point numbers (a vector) that represents the semantic
meaning of a piece of text. Two pieces of text that mean similar things have similar embeddings
— their vectors point in similar directions in high-dimensional space. A model trained on
large amounts of text produces these embeddings.

*Cosine similarity* measures the angle between two vectors. If the angle is 0 degrees (vectors
point in the same direction), similarity is 1.0. If they are perpendicular, similarity is 0.
This gives a number between 0 and 1 that captures semantic relatedness regardless of the
exact words used.

*Vector search* finds memories whose embeddings are most similar to the embedding of the
query. This captures semantic meaning, not just keyword overlap. A query like "programming
language that is memory safe" would find memories about Rust even if the word "Rust" never
appears.

ZeroClaw stores embeddings as binary blobs (`BLOB` column type) in the `memories` table and
performs cosine similarity computation in Rust code by scanning all stored embeddings.

**Why ZeroClaw uses it.**
ZeroClaw combines FTS5 keyword search and vector similarity search in a *hybrid merge*
weighted fusion: 70% vector score + 30% keyword score by default. This gives the best of both
approaches — precise keyword matching and fuzzy semantic matching — so the agent retrieves
the most contextually relevant memories.

**Where in the code.**
`src/memory/sqlite.rs` — `vector_search` and the embedding cache.
`src/memory/vector.rs` — `cosine_similarity`, `bytes_to_vec`, `vec_to_bytes`, `hybrid_merge`.
`src/memory/embeddings.rs` — the `EmbeddingProvider` trait and `NoopEmbedding`.

---

## 6. Cryptography and Security

### chacha20poly1305 (0.10)

**What it is.**
AEAD stands for Authenticated Encryption with Associated Data. It combines two guarantees into
a single operation:

1. *Confidentiality*: the ciphertext reveals nothing about the plaintext to anyone who does not
   have the key.
2. *Integrity*: any modification of the ciphertext is detected. You cannot alter a ciphertext
   and have it decrypt to a different plaintext without the modification being caught.

ChaCha20-Poly1305 is a specific AEAD algorithm. ChaCha20 is the encryption cipher (turns
plaintext into ciphertext using the key). Poly1305 is the authentication tag (a 16-byte
checksum computed over the ciphertext using the key). The two together ensure that a ciphertext
can only be decrypted by someone with the key, and any tampering is detected on decryption.

A *nonce* (number used once) is a random 12-byte value generated fresh for every encryption
operation. It ensures that encrypting the same plaintext twice produces different ciphertexts,
preventing attackers from learning patterns by observing repeated encryptions.

**Why ZeroClaw uses it.**
API keys stored in the config file are encrypted with ChaCha20-Poly1305 (`enc2:` prefix). The
encryption key is stored in `~/.zeroclaw/.secret_key` with file permissions set to `0600`
(owner-read-only). This means even if someone reads your config file (e.g., via accidental
git commit), they see only ciphertext — not your API keys. The Poly1305 tag prevents anyone
from tampering with the ciphertext to trick ZeroClaw into using a modified key.

**Where in the code.**
`src/security/secrets.rs` — `encrypt` and `decrypt` methods of `SecretStore`.

**Learn more.** https://docs.rs/chacha20poly1305

---

### hmac (0.12) + sha2 (0.10) + hex (0.4)

**What they are.**
HMAC (Hash-based Message Authentication Code) is a way to verify that a message was sent by
someone who knows a secret key and that the message was not modified in transit.

Here is the concept: a hash function (like SHA-256) takes any input and produces a fixed-size
output called a digest. The same input always produces the same digest. If you change even one
bit of the input, the digest changes completely and unpredictably. An HMAC mixes a secret key
into the hash computation: `HMAC(key, message) = hash(key + message)`. Without the key, you
cannot produce a valid HMAC.

Webhook providers (Telegram, Slack, GitHub) sign their webhook requests with an HMAC. They
compute `HMAC(shared_secret, request_body)` and include the result in a request header. The
receiver (ZeroClaw's gateway) recomputes the same HMAC and compares. If the values match, the
request is authentic. If they differ, the request is rejected.

`sha2` provides SHA-256 and SHA-512. `hmac` provides HMAC built on top of any hash function.
`hex` encodes binary hashes into printable hexadecimal strings for comparison with the values
in webhook headers.

**Why ZeroClaw uses them.**
The gateway receives webhooks from Telegram, Slack, and custom integrations. Without HMAC
verification, anyone who knows the gateway URL could send fake messages and trick the agent
into executing commands. HMAC signature verification ensures only legitimate senders with the
shared secret can trigger the agent.

SHA-256 is also used in `src/memory/sqlite.rs` as a deterministic content hash for the
embedding cache (replacing `DefaultHasher`, which is explicitly documented as unstable across
Rust versions).

**Where in the code.**
`src/gateway/mod.rs` — webhook signature verification.
`src/security/pairing.rs` — pairing token HMAC.
`src/memory/sqlite.rs` — content hashing for the embedding cache.

**Learn more.** https://docs.rs/hmac

---

### ring (0.17)

**What it is.**
`ring` is a cryptographic library for Rust focused on correctness, performance, and simplicity.
It implements a carefully chosen subset of cryptographic operations: HMAC, AEAD ciphers, key
derivation, and digital signatures.

**Why ZeroClaw uses it.**
The Zhipu GLM provider (a Chinese LLM provider) requires JWT (JSON Web Token) authentication.
A JWT is a signed token that proves the caller's identity without sending a password. ZeroClaw
generates JWTs by signing a payload with the API key using HMAC-SHA256 — and uses `ring`'s
`hmac` module for this specific operation. The `ring` crate is used here because it provides
a clean interface for the key+sign operation required by GLM's JWT format.

**Where in the code.**
`src/providers/glm.rs` — JWT generation with `ring::hmac`.

**Learn more.** https://docs.rs/ring

---

### rand (0.9)

**What it is.**
`rand` is Rust's random number generation library. It provides a range of random number
generators from fast-but-predictable (for simulations) to cryptographically secure (for
security-sensitive values).

A *cryptographically secure* random number generator (CSPRNG) is one whose output cannot
be predicted by an attacker even if they observe many previous outputs. This matters for
security tokens: if your session token is `random_int % 1000`, an attacker can guess it
quickly. If it is 32 bytes from a CSPRNG, it is computationally infeasible to guess.

**Why ZeroClaw uses it.**
ZeroClaw generates random pairing tokens, session identifiers, and nonces for the ChaCha20
cipher. All of these use rand's CSPRNG. A predictable token would let an attacker forge
pairing handshakes or session identifiers.

**Where in the code.**
`src/security/secrets.rs` (nonce generation for encryption via `OsRng`),
`src/security/pairing.rs` (token generation).

**Learn more.** https://docs.rs/rand

---

## 7. Observability and Logging

### tracing (0.1) + tracing-subscriber (0.3)

**What they are.**
Logging is recording what an application does while it runs. Plain logging writes text lines
to a file or terminal: `[INFO] Agent started`. Structured logging attaches machine-readable
key-value metadata to each log event: `agent_started provider=openrouter model=claude-3.5`.
Structured logs can be queried, filtered, and analyzed by log management tools.

`tracing` is Rust's structured logging and diagnostics framework. It provides macros like
`info!`, `warn!`, `error!`, and `debug!` for emitting log events with attached fields. It
also supports *spans* — named regions of execution that can be entered and exited, creating
a hierarchical view of what the application was doing when something happened.

`tracing-subscriber` is the backend that receives events from `tracing` and formats them for
output. It is configured to read the `RUST_LOG` environment variable, which controls which
events are emitted. For example, `RUST_LOG=debug` shows everything; `RUST_LOG=warn` shows
only warnings and errors.

**Why ZeroClaw uses them.**
ZeroClaw is a long-running daemon that may be running unattended. When something goes wrong
(a provider times out, a webhook fails, a tool returns an error), structured logs make it
possible to diagnose the problem. The `RUST_LOG` environment variable lets operators increase
verbosity without recompiling.

**Where in the code.**
`src/main.rs` — subscriber initialization. `tracing::{info, warn, error, debug}` macros are
used throughout the codebase wherever events worth recording occur.

**Learn more.** https://docs.rs/tracing

---

### opentelemetry (0.31) + opentelemetry-otlp

**What they are.**
OpenTelemetry (OTel) is an open-source observability framework that standardizes how
applications produce and export telemetry data: traces, metrics, and logs. It was created
because every monitoring vendor (Datadog, Jaeger, Zipkin, New Relic) had a different API,
so switching vendors required rewriting all instrumentation code. OpenTelemetry defines one
API that all backends understand.

A *trace* is a record of a specific operation's execution path through a system. A *span* is
one unit of work within a trace. A *metric* is a measured value collected over time (a counter,
a gauge, a histogram).

OTLP (OpenTelemetry Protocol) is the standard wire format for exporting telemetry data.
ZeroClaw exports over HTTP to port 4318, the default OTLP port. This data can be received by
any OTel-compatible backend: Jaeger (traces), Prometheus (metrics), Grafana, etc.

**Why ZeroClaw uses them.**
ZeroClaw needs to answer questions like: how long do LLM calls take? How many tokens are
consumed per hour? Which tools are called most frequently? How many errors are occurring in
which components? OpenTelemetry provides this visibility without coupling ZeroClaw to any
specific monitoring vendor.

The `OtelObserver` in `src/observability/otel.rs` records spans for every agent invocation,
LLM call, tool execution, and error. These spans carry timing, provider, model, and success
metadata.

**Where in the code.**
`src/observability/otel.rs` — the `OtelObserver` implementation.
`src/observability/mod.rs` — observer factory.

**Learn more.** https://opentelemetry.io

---

### prometheus (0.14)

**What it is.**
Prometheus is an open-source metrics monitoring system widely used for production services.
It works by *scraping* — periodically making an HTTP GET request to a `/metrics` endpoint on
your service and collecting the metrics it finds there. Those metrics are stored in a time-
series database and can be visualized in Grafana or alerted on with Alertmanager.

Prometheus metrics come in four types:
- Counter: a value that only ever goes up (e.g., total requests served).
- Gauge: a value that can go up or down (e.g., current number of active sessions).
- Histogram: a distribution of values (e.g., distribution of request latency).
- Summary: similar to histogram but computed client-side.

The `prometheus` crate implements the Prometheus exposition format in Rust, letting ZeroClaw
define and collect metrics that Prometheus can scrape.

**Why ZeroClaw uses it.**
The `PrometheusObserver` in `src/observability/prometheus.rs` tracks counters (agent starts,
tool calls, channel messages, errors), histograms (agent duration, tool duration, request
latency), and gauges (active sessions, queue depth). The gateway exposes a `/metrics` endpoint
that Prometheus scrapes. This enables dashboards showing agent activity over time.

**Where in the code.**
`src/observability/prometheus.rs`, `src/gateway/mod.rs` (the `/metrics` endpoint).

**Learn more.** https://prometheus.io

---

## 8. Async and Concurrency Primitives

### async-trait (0.1)

**What it is.**
In Rust, traits define shared behavior — they are similar to interfaces in other languages.
However, Rust's async functions are more complex than normal functions: they return a Future
type that depends on the function's return type and lifetime parameters. Rust's trait system
does not yet (as of toolchain 1.92.0) natively support async functions in traits in all
contexts. The `async-trait` crate is a procedural macro that rewrites `async fn` in traits
into a form that works correctly, using boxed Futures.

**Why ZeroClaw uses it.**
ZeroClaw's architecture is built on traits: `Provider`, `Channel`, `Tool`, `Memory`,
`Observer`, `Peripheral`. All of these traits have async methods (because their operations
involve waiting for I/O). Without `async-trait`, these traits could not have async methods.
The `#[async_trait]` attribute is on every trait and every implementation.

**Where in the code.**
`src/providers/traits.rs`, `src/channels/traits.rs`, `src/tools/traits.rs`,
`src/memory/traits.rs`, `src/observability/traits.rs`, `src/peripherals/traits.rs` — and all
implementations of those traits.

**Learn more.** https://docs.rs/async-trait

---

### parking_lot (0.12)

**What it is.**
A *mutex* (mutual exclusion lock) is a synchronization primitive. When multiple threads might
access the same piece of data simultaneously, a mutex ensures that only one thread accesses it
at a time. A thread that wants to access the data must first *acquire* the lock. If another
thread already holds the lock, the waiting thread blocks until the lock is released.

Rust's standard library includes `std::sync::Mutex`. parking_lot provides a faster alternative.
The standard library mutex marks itself as "poisoned" when a thread panics while holding it —
meaning all future attempts to lock it will return an error. ZeroClaw uses `parking_lot::Mutex`
throughout because parking_lot mutexes do not poison on panic (ZeroClaw's error handling model
uses explicit errors via `anyhow`, not panics), and they are measurably faster in benchmarks.

**Why ZeroClaw uses it.**
The SQLite connection, Discord's typing indicator task handle, auth caches, and the GLM JWT
token cache all need shared mutable state across async tasks. `parking_lot::Mutex` protects
all of these.

**Where in the code.**
`src/memory/sqlite.rs` (connection guard), `src/channels/discord.rs` (typing handle),
`src/channels/email_channel.rs`, `src/gateway/mod.rs`.

**Learn more.** https://docs.rs/parking_lot

---

### futures (0.3) + futures-util

**What they are.**
`futures` provides additional utilities for working with Rust's async ecosystem beyond what
Tokio includes. Key components:

- `futures::channel` — async-friendly channels for passing values between tasks.
- `futures::stream` — combinators for working with async streams (sequences of values produced
  over time): `map`, `filter`, `fold`, `select`, etc.
- `futures-util` — utility functions extracted from `futures`, specifically the `SinkExt` and
  `StreamExt` extension traits used for working with WebSocket connections.

**Why ZeroClaw uses them.**
WebSocket connections are bidirectional streams. Sending to a WebSocket uses the `Sink`
abstraction (push values in). Receiving uses `Stream` (pull values out). The Discord and Lark
channel implementations use `SinkExt::send` and `StreamExt::next` from `futures-util` to
drive the WebSocket protocol.

**Where in the code.**
`src/channels/discord.rs`, `src/channels/lark.rs`.

**Learn more.** https://docs.rs/futures

---

## 9. Time and Scheduling

### chrono (0.4) + chrono-tz (0.10)

**What they are.**
`chrono` is Rust's date and time library. It provides types for representing dates (`NaiveDate`),
times (`NaiveTime`), and timestamps with timezone offsets (`DateTime<Utc>`, `DateTime<Local>`).
It handles date arithmetic (add 30 days to today), formatting (convert a timestamp to a
human-readable string), and parsing (read a timestamp from a string like `2024-01-15T14:30:00Z`).

`chrono-tz` extends chrono with support for named IANA timezones (e.g., `America/Los_Angeles`,
`Asia/Tokyo`). Without chrono-tz, you can only work with UTC or a fixed offset. With it, you
can correctly compute "what is 9 AM Pacific time on this date?" accounting for daylight saving
time transitions.

**Why ZeroClaw uses them.**
Memory entries are timestamped (created_at, updated_at). Cron schedules need to fire at
correct local times. Auth token expiry times must be compared to the current time. The `chrono`
crate handles all of this. `chrono-tz` is needed specifically for the cron scheduler, which
allows tasks to be scheduled in specific timezones.

**Where in the code.**
`src/memory/sqlite.rs` (timestamps), `src/cron/schedule.rs` (timezone-aware next-run
computation), `src/main.rs` (token expiry formatting).

**Learn more.** https://docs.rs/chrono

---

### cron (0.15)

**What it is.**
A cron expression is a compact string that describes a repeating schedule. For example:

```
┌──────────── second (0-59)
│ ┌────────── minute (0-59)
│ │ ┌──────── hour (0-23)
│ │ │ ┌────── day of month (1-31)
│ │ │ │ ┌──── month (1-12)
│ │ │ │ │ ┌── day of week (0-7, Sunday=0 or 7)
│ │ │ │ │ │
* * * * * *   (every second)
0 0 * * * *   (every day at midnight)
0 30 9 * * 1  (every Monday at 9:30 AM)
```

The `cron` crate parses these expressions and computes the next time the schedule should fire,
given a starting timestamp.

**Why ZeroClaw uses it.**
Users can schedule recurring tasks: "check email every hour", "summarize today's activity
at 6 PM", "send a report every Monday at 9 AM". The cron scheduler in ZeroClaw parses these
expressions and uses `cron::Schedule::after(&from).next()` to compute the next firing time.

**Where in the code.**
`src/cron/schedule.rs` — `next_run_for_schedule` uses the `cron` crate.
`src/cron/scheduler.rs` — the scheduler loop.

**Learn more.** https://docs.rs/cron

---

## 10. Email

### lettre (0.11)

**What it is.**
SMTP (Simple Mail Transfer Protocol) is the standard protocol for sending email. When you
click Send in Gmail, Gmail's servers communicate with your recipient's mail server using SMTP.

`lettre` is a Rust library that implements an SMTP client, allowing ZeroClaw to programmatically
send emails. It handles SMTP authentication, TLS encryption for the connection, constructing
well-formed email messages (headers, body, attachments), and submitting the message to the
mail server.

**Why ZeroClaw uses it.**
ZeroClaw's email channel (`src/channels/email_channel.rs`) can send messages via email in
addition to chat platforms. When the agent generates a response, it can deliver it to an email
address over SMTP. lettre handles the actual delivery.

**Where in the code.**
`src/channels/email_channel.rs` — `SmtpTransport` and `Message` from lettre.

**Learn more.** https://docs.rs/lettre

---

### mail-parser (0.11)

**What it is.**
Receiving email is more complex than sending it. Raw email messages are encoded in the MIME
format — a multi-part format with headers, encoded bodies, and nested attachments. Parsing
a raw MIME email reliably is non-trivial: encodings vary, headers have special syntax, and
attachments can be deeply nested.

`mail-parser` is a Rust library that parses raw email byte streams into a structured
representation with typed access to headers, body parts, and attachments.

**Why ZeroClaw uses it.**
The email channel polls an IMAP mailbox for incoming messages and parses them as incoming
channel messages. `mail-parser` converts the raw IMAP message bytes into a structured form
from which ZeroClaw can extract the sender, subject, and body text to pass to the agent.

**Where in the code.**
`src/channels/email_channel.rs` — `MessageParser` from mail-parser.

**Learn more.** https://docs.rs/mail-parser

---

## 11. Hardware and Peripherals

ZeroClaw uniquely extends beyond software by supporting physical hardware. The agent can
communicate with microcontrollers and single-board computers to execute real-world actions.

```
  Natural language command (via Telegram, WhatsApp, etc.)
         |
  ZeroClaw agent interprets command
         |
  ┌──────┴──────────────────────────────────────────────┐
  │             Hardware abstraction layer               │
  │  (Peripheral trait + tool dispatch)                 │
  └──────┬───────────────────────────────────────────────┘
         |
  ┌──────┴──────────────────────────────────────┐
  │  tokio-serial     nusb      rppal            │
  │  (serial port)  (USB enum) (RPi GPIO)        │
  │                                              │
  │  probe-rs        embassy    esp-idf           │
  │  (STM32 flash)  (STM32 FW) (ESP32 FW)        │
  └──────────────────────────────────────────────┘
         |
  ┌──────┴────────────────────────────────────────────────────┐
  │  Physical hardware                                        │
  │  Raspberry Pi GPIO  |  STM32 Nucleo  |  ESP32  |  Arduino │
  └───────────────────────────────────────────────────────────┘
```

---

### tokio-serial (5)

**What it is.**
A serial port (also called a UART connection) is one of the oldest and most universal ways for
a computer to communicate with external hardware. It sends data one bit at a time over a pair
of wires (transmit and receive). USB-to-serial adapters make serial ports available on modern
computers via USB. Microcontrollers like the STM32 Nucleo appear as serial ports when plugged
in via USB.

`tokio-serial` is an async serial port library for Rust built on Tokio. It lets ZeroClaw open
serial ports and read/write bytes asynchronously without blocking the async runtime.

**Why ZeroClaw uses it.**
ZeroClaw communicates with connected microcontrollers (STM32 Nucleo, Arduino Uno) over their
USB-to-serial connection. Commands are sent as text or binary over the serial port; responses
arrive the same way. This is the primary communication channel between ZeroClaw and the
microcontroller in host-mediated mode.

**Where in the code.**
`src/peripherals/serial.rs` — the serial port peripheral driver.
Only compiled when the `hardware` feature is enabled (which is on by default).

**Learn more.** https://docs.rs/tokio-serial

---

### nusb (0.2)

**What it is.**
USB (Universal Serial Bus) is the standard for connecting peripherals to computers. USB
device enumeration is the process of discovering what USB devices are currently connected.
Every USB device has a Vendor ID (VID) and Product ID (PID) — two numbers that uniquely
identify the device type. For example, an STM32 Nucleo has VID `0x0483`.

`nusb` is a Rust library for USB device enumeration that works without requiring the
operating system's libusb library. It can list all connected USB devices and their VID/PID
identifiers.

**Why ZeroClaw uses it.**
The `zeroclaw hardware list` command enumerates USB devices so users (and the agent itself)
can see what hardware is connected. This is the first step in the hardware discovery flow —
before ZeroClaw can communicate with a device, it must identify what is connected.

**Where in the code.**
`src/hardware/discover.rs` — USB enumeration using nusb.
Only compiled when the `hardware` feature is enabled.

**Learn more.** https://docs.rs/nusb

---

### rppal (0.22, Linux only)

**What it is.**
GPIO (General-Purpose Input/Output) pins are physical pins on a microcontroller or
single-board computer that can be programmatically set to high voltage (1) or low voltage (0).
By toggling GPIO pins, software can control LEDs, motors, relays, and sensors.

`rppal` (Raspberry Pi Peripheral Access Library) is a Rust library for accessing GPIO pins,
I2C buses, SPI buses, and other peripherals on the Raspberry Pi. It communicates directly
with the Linux kernel's GPIO interface, requiring no external libraries.

**Why ZeroClaw uses it.**
When ZeroClaw runs directly on a Raspberry Pi and a user sends a command like "turn on pin 17",
the `RpiGpioPeripheral` uses rppal to toggle the physical GPIO pin. This is the hardware
control path for RPi-hosted deployments.

**Where in the code.**
`src/peripherals/rpi.rs` — `RpiGpioPeripheral` calls `rppal::gpio::Gpio::new()`.
Only compiled on Linux with the `peripheral-rpi` feature.

**Learn more.** https://docs.rs/rppal

---

### probe-rs (0.30, optional)

**What it is.**
Modern microcontrollers have a debug interface (often called SWD or JTAG) that allows an
external debugger to read and write the chip's memory, pause execution, set breakpoints, and
flash new firmware — all without modifying the chip's code. `probe-rs` is a Rust library that
talks to debug probes (hardware adapters like J-Link, CMSIS-DAP, and ST-Link that connect the
host computer to the microcontroller's debug interface).

**Why ZeroClaw uses it.**
The `probe` feature enables ZeroClaw to flash firmware to STM32 Nucleo boards and read their
memory maps. This is used by the `zeroclaw peripheral nucleo flash` command to deploy new
firmware to a connected STM32. This is optional because it adds approximately 50 transitive
dependencies.

**Where in the code.**
`src/peripherals/nucleo_flash.rs`.
Only compiled with `--features probe`.

**Learn more.** https://probe.rs

---

### Arduino IDE and Arduino Uno Firmware

**What Arduino is.**
Arduino is an ecosystem for easy microcontroller programming. The Arduino Uno is a small
microcontroller board based on the ATmega328P chip. The Arduino IDE is a development
environment that lets you write C/C++ code, compile it with the AVR-GCC toolchain, and
upload it to the board over USB. Arduino's popularity comes from its simple IDE, large
community, and extensive library ecosystem.

ZeroClaw itself is written in Rust. However, the Arduino Uno cannot run Rust code (its
toolchain does not support Rust targets in ZeroClaw's configuration). The Arduino firmware
is a small C/C++ sketch that implements ZeroClaw's serial command protocol: it listens for
text commands over the serial port and responds with sensor readings or pin states.

**Why ZeroClaw uses it.**
Arduino Uno is one of the most common hobbyist boards. Supporting it lets users who have an
Arduino Uno use it as a peripheral in their ZeroClaw setup — the agent can read sensors or
control actuators via the serial command bridge.

**Where in the code.**
`src/peripherals/uno_q_bridge.rs` — the host-side serial bridge.
`src/peripherals/arduino_flash.rs` — firmware upload via avrdude.
`docs/arduino-uno-q-setup.md`, `docs/datasheets/arduino-uno.md`.

**Learn more.** https://www.arduino.cc

---

### esp-idf-svc / esp-idf-hal / esp-idf-sys

**What they are.**
The ESP32 is a popular Wi-Fi and Bluetooth enabled microcontroller from Espressif Systems. It
runs significantly more powerful hardware than an Arduino Uno and can run an operating system.
Espressif provides the esp-idf (IoT Development Framework), a C-based SDK for ESP32 development.

The `esp-idf-svc`, `esp-idf-hal`, and `esp-idf-sys` crates form the Rust binding layer to
esp-idf. They allow writing ESP32 firmware in Rust while leveraging the underlying esp-idf C
library for Wi-Fi, Bluetooth, GPIO, and display drivers.

**Why ZeroClaw uses them.**
In edge-native mode, ZeroClaw runs directly on the ESP32, connecting to Wi-Fi and running
the agent loop on-device. The esp-idf crates provide the hardware abstraction layer for GPIO,
networking, and display output.

**Where in the code.**
`firmware/esp32/` directory (firmware code for the ESP32 target).
`docs/datasheets/esp32.md`.

**Learn more.** https://github.com/esp-rs/esp-idf-hal

---

### embassy (embassy-executor, embassy-stm32, embassy-time)

**What it is.**
Embassy is an async embedded framework for Rust that brings Tokio-style async/await to bare-
metal microcontrollers (no operating system, no heap in the traditional sense). It provides:

- `embassy-executor` — an async task executor for microcontrollers.
- `embassy-stm32` — hardware abstraction layer for STM32 microcontrollers.
- `embassy-time` — timer and delay primitives for embedded systems.

Embassy makes it possible to write async Rust on an STM32 that runs multiple concurrent tasks
(reading a sensor, responding to serial commands, blinking an LED) without an RTOS.

**Why ZeroClaw uses it.**
The STM32 Nucleo-F401RE firmware is written using Embassy, enabling async multitasking on the
microcontroller itself. When ZeroClaw flashes custom firmware onto the Nucleo, that firmware
uses Embassy to handle concurrent peripheral communication.

**Where in the code.**
`firmware/nucleo/` directory.
`docs/nucleo-setup.md`, `docs/datasheets/nucleo-f401re.md`.

**Learn more.** https://embassy.dev

---

### Slint (1.10)

**What it is.**
Slint is a declarative UI toolkit for embedded systems and desktop applications. It uses a
domain-specific language (`.slint` files) to describe user interfaces and generates Rust (or
C++) code that renders those interfaces. Slint is designed for resource-constrained devices
where full web-based UIs are not practical.

**Why ZeroClaw uses it.**
ESP32 boards can be equipped with small SPI display screens. Slint provides the UI framework
for rendering a graphical interface on those displays — status information, sensor readings,
or agent response summaries that appear on the physical device screen.

**Where in the code.**
`firmware/esp32/` — Slint UI definitions for ESP32 display peripherals.

**Learn more.** https://slint.dev

---

## 12. Testing and Quality

### cargo test

**What it is.**
`cargo test` is the built-in test runner for Rust. Any function annotated with `#[test]` is
collected and run as a test case. Tests can be unit tests (testing a single function in
isolation) or integration tests (testing the interaction of multiple components).

Rust tests run in parallel by default and are compiled with the same type system guarantees
as production code — no "tests are in a different language" mismatch. Async tests use
`#[tokio::test]` to run inside a Tokio runtime.

**Where in the code.**
`src/memory/sqlite.rs` has over 60 test cases for the SQLite memory backend. Every module
with non-trivial logic has corresponding tests in a `#[cfg(test)]` module.

---

### criterion (0.5)

**What it is.**
A benchmark measures how long a piece of code takes to run. Benchmarks help detect performance
regressions: if a code change makes the agent loop 50% slower, a benchmark catches it before
the change ships.

`criterion` is a statistical benchmarking library for Rust. Unlike a simple "run this function
100 times and average", criterion uses statistical analysis to detect whether a measured
difference is real signal or measurement noise. It generates HTML reports with charts showing
the distribution of measurements.

The `async_tokio` feature allows benchmarking async functions.

**Why ZeroClaw uses it.**
`benches/agent_benchmarks.rs` benchmarks the hot paths: tool dispatch (XML and native
parsing), SQLite memory store/recall cycles, and the full agent turn cycle. These benchmarks
are run in CI on a weekly schedule to detect regressions.

**Where in the code.**
`benches/agent_benchmarks.rs`.

**Learn more.** https://docs.rs/criterion

---

### tempfile (3.14)

**What it is.**
Tests that involve file I/O need temporary files and directories that are cleaned up after the
test finishes. Creating them manually in `/tmp/my_test_file` risks collisions between parallel
tests and leaves garbage if a test panics.

`tempfile` creates temporary files and directories that are automatically deleted when the
`TempDir` or `TempFile` object is dropped (Rust's RAII cleanup). It handles platform
differences in temp directory locations.

**Why ZeroClaw uses it.**
SQLite memory tests in `src/memory/sqlite.rs` create a `TempDir` for each test case,
giving each test its own isolated database file. The directory is deleted when the test ends,
even if the test panics.

**Where in the code.**
`src/memory/sqlite.rs` — `TempDir::new()` in test helper `temp_sqlite()`.

**Learn more.** https://docs.rs/tempfile

---

### clippy

**What it is.**
A linter is a static analysis tool that reads code and reports issues without running it.
clippy is Rust's official linter. It checks for hundreds of patterns: code that compiles but
is probably wrong, unnecessary operations, performance anti-patterns, style inconsistencies,
and potential panics.

clippy runs as `cargo clippy --all-targets -- -D warnings`. The `-D warnings` flag treats
every clippy warning as a compile error, making clippy violations a hard CI failure.

`clippy.toml` configures clippy, and `src/main.rs` uses `#[allow(...)]` attributes to
suppress specific lint categories that are intentionally accepted in ZeroClaw's codebase.

**Where in the code.**
`clippy.toml`, `src/main.rs` (allow list at the top of the file).

**Learn more.** https://doc.rust-lang.org/clippy/

---

### rustfmt

**What it is.**
rustfmt is Rust's official code formatter. It automatically reformats Rust source code to
follow a consistent style: indentation, line length, brace placement, import ordering.
Having a standard formatter means no time is wasted on formatting debates in code review.

`rustfmt.toml` configures rustfmt's behavior. `cargo fmt --all -- --check` runs in CI and
fails if any file is not formatted correctly.

**Where in the code.**
`rustfmt.toml` — configuration.

**Learn more.** https://github.com/rust-lang/rustfmt

---

### cargo-fuzz

**What it is.**
Fuzz testing (fuzzing) is an automated testing technique where random or semi-random inputs
are fed to a function to find crashes, hangs, or incorrect behavior. A fuzzer generates
thousands of inputs per second, exploring edge cases that human-written tests miss.

`cargo-fuzz` integrates libFuzzer (LLVM's fuzzing engine) with Rust. It requires the Rust
nightly toolchain and compiles the target code with address sanitizers that detect memory
errors.

**Why ZeroClaw uses it.**
ZeroClaw parses untrusted external input: config files from disk, webhook request bodies from
the internet, tool parameters from LLM-generated JSON. Bugs in parsers that process malformed
input are a common source of security vulnerabilities. Fuzzing targets (`fuzz_config_parse`
and `fuzz_tool_params`) run weekly in CI to find crashes or panics in input-handling code.

**Where in the code.**
`.github/workflows/test-fuzz.yml`, `fuzz/` directory.

**Learn more.** https://rust-fuzz.github.io/book/cargo-fuzz.html

---

### cargo-deny

**What it is.**
`cargo-deny` audits a Rust project's dependency tree against multiple criteria:

- **Vulnerabilities**: checks every dependency against the RustSec Advisory Database, a
  database of known security vulnerabilities in Rust crates.
- **Licenses**: verifies that every dependency uses a license compatible with the project's
  license policy.
- **Bans**: detects duplicate versions of the same crate or banned crates.
- **Sources**: ensures all crates come from trusted registries.

**Why ZeroClaw uses it.**
ZeroClaw's dependency tree includes over 200 crates. Manually auditing all of them for
security issues and license compliance is infeasible. cargo-deny automates this in CI. The
`deny.toml` file declares the allowed licenses and configures the advisory checks.

**Where in the code.**
`deny.toml`, `.github/workflows/sec-audit.yml`.

**Learn more.** https://embarkstudios.github.io/cargo-deny/

---

## 13. Containerization and Deployment

### Docker

**What it is.**
A container is a lightweight, isolated environment that packages an application with all its
dependencies. Unlike a virtual machine (which emulates an entire computer with its own OS
kernel), a container shares the host OS kernel but has its own filesystem, networking, and
process space. This makes containers far smaller and faster to start than VMs.

A Docker image is a snapshot of a container's filesystem. A Dockerfile is the recipe for
building an image. You run `docker build .` and Docker executes the Dockerfile step by step,
producing an image. You then run that image as a container with `docker run`.

**Multi-stage builds.**
ZeroClaw's Dockerfile uses a multi-stage build, which is an optimization technique:

```
Stage 1: builder (rust:1.93-slim)
  - Has the Rust compiler, all build tools, and development headers
  - Compiles the release binary
  - Copies only the compiled binary to the next stage

Stage 2: dev (debian:trixie-slim)
  - A minimal Debian image with ca-certificates and curl
  - Gets the binary from stage 1
  - Used for local development with Ollama

Stage 3: release (gcr.io/distroless/cc-debian13:nonroot)
  - A distroless image with no shell, no package manager, no tools
  - Only the C runtime library needed to run the binary
  - Gets the binary from stage 1
  - Production deployment target
```

The build stage is large (the Rust compiler is several hundred MB). The production image is
tiny (only the binary plus the C runtime, approximately 10 MB total).

**Distroless images.**
A distroless image contains nothing except the minimum needed to run a compiled binary: the
C runtime library (`libc`, `libgcc`) and nothing else. There is no shell, no `ls`, no `cat`,
no package manager. This dramatically reduces the attack surface: an attacker who compromises
the container has almost no tools available to them.

`gcr.io/distroless/cc-debian13:nonroot` additionally runs as the nobody user (UID 65534)
rather than root, following least-privilege principles.

**Where in the code.**
`Dockerfile`, `docker-compose.yml`, `dev/docker-compose.yml`, `dev/ci/Dockerfile`.

**Learn more.**
https://docs.docker.com
https://github.com/GoogleContainerTools/distroless

---

### Docker Compose

**What it is.**
Docker Compose orchestrates multiple containers together. Instead of running individual
`docker run` commands for each service and configuring their networking manually, you define
all services in a `docker-compose.yml` file and start everything with `docker compose up`.

**Why ZeroClaw uses it.**
The development environment (`dev/docker-compose.yml`) runs ZeroClaw alongside Ollama (for
local LLM inference) and optional telemetry backends. The CI environment
(`dev/docker-compose.ci.yml`) mirrors this setup for reproducible test runs.

**Where in the code.**
`docker-compose.yml`, `dev/docker-compose.yml`, `dev/docker-compose.ci.yml`.

**Learn more.** https://docs.docker.com/compose/

---

## 14. CI/CD and GitHub Tooling

CI/CD stands for Continuous Integration / Continuous Delivery. Continuous Integration means
every code change is automatically built and tested before it is merged. Continuous Delivery
means the tested build can be automatically deployed.

```
  Developer opens a PR
         |
  GitHub Actions triggers workflows
         |
  ┌──────┴──────────────────────────────────────┐
  │  Lint (rustfmt + clippy)                     │
  │  Tests (cargo test)                          │
  │  Build (cargo build --release)               │
  │  Docs quality (markdownlint + lychee)        │
  │  Security audit (cargo-deny + CodeQL)        │
  │  Fuzz testing (weekly)                       │
  │  Benchmarks (on request)                     │
  └──────────────────────────────────────────────┘
         |
  CI Required Gate (all must pass)
         |
  PR merged to main
         |
  Docker image built and published
  Release binary built and published
```

---

### GitHub Actions

**What it is.**
GitHub Actions is GitHub's built-in CI/CD system. Workflows are defined in YAML files in
`.github/workflows/`. Each workflow runs automatically in response to events: a push to main,
a pull request opened, a schedule (daily, weekly), or a manual trigger.

A workflow consists of jobs. Each job runs on a fresh virtual machine (a *runner*). Jobs can
depend on each other and can run in parallel. Steps within a job run sequentially.

**Why ZeroClaw uses it.**
ZeroClaw has 15+ workflow files covering: the main CI gate, PR intake checks, label routing,
auto-responses, stale PR detection, Docker image publishing, release binary publishing,
security audits, fuzz testing, benchmarks, end-to-end tests, and documentation quality.

**Where in the code.**
`.github/workflows/` — all workflow YAML files.

**Learn more.** https://docs.github.com/en/actions

---

### Blacksmith Runners

**What they are.**
GitHub Actions workflows run on *runners* — virtual machines that execute the steps. GitHub
provides standard runners (`ubuntu-latest`). Blacksmith provides faster runners with better
performance. ZeroClaw uses `blacksmith-2vcpu-ubuntu-2404` runners, which provide Ubuntu 24.04
with 2 vCPUs and faster disk I/O. Faster runners mean shorter CI wait times for contributors.

**Where in the code.**
Every workflow file uses `runs-on: blacksmith-2vcpu-ubuntu-2404`.

---

### CodeQL

**What it is.**
CodeQL is GitHub's static security analysis engine. It treats source code as a database of
facts and runs queries against it to find security vulnerabilities: SQL injection, command
injection, buffer overflows, path traversal, and more. CodeQL understands program data flow —
it can trace a value from user input all the way to a dangerous function call, even across
multiple functions and files.

**Why ZeroClaw uses it.**
ZeroClaw's gateway and tools handle untrusted input from the internet. CodeQL scans run twice
daily (6 AM and 6 PM UTC) to catch any newly introduced security vulnerabilities before they
can be exploited.

**Where in the code.**
`.github/workflows/sec-codeql.yml`, `.github/codeql/codeql-config.yml`.

**Learn more.** https://codeql.github.com

---

### Dependabot

**What it is.**
Dependabot is GitHub's automated dependency update service. It periodically checks your
dependency specifications (Cargo.toml for Rust, package.json for npm, etc.) against the
latest available versions and opens pull requests to update dependencies that have new
releases.

**Why ZeroClaw uses it.**
Dependencies get security patches, bug fixes, and new features over time. Manually tracking
all dependencies and filing update PRs is impractical. Dependabot automates this, opening
separate PRs for each outdated dependency so they can be reviewed and merged independently.

**Where in the code.**
`.github/dependabot.yml`.

**Learn more.** https://docs.github.com/en/code-security/dependabot

---

### lychee

**What it is.**
lychee is a link checker. It reads Markdown and HTML files, extracts all hyperlinks, and
verifies that each URL is reachable (returns a successful HTTP status code). Dead links in
documentation are a common source of confusion for users.

**Why ZeroClaw uses it.**
ZeroClaw has extensive documentation in `docs/`. The CI pipeline extracts any links added
in changed documentation files and runs lychee to verify they work before the PR is merged.

**Where in the code.**
`.github/workflows/ci-run.yml` — the `docs-quality` job.

**Learn more.** https://lychee.cli.rs

---

### markdownlint

**What it is.**
markdownlint is a linter for Markdown files. It checks that Markdown follows consistent style
rules: headings should be in the correct hierarchy, code blocks should have language tags,
lines should not have trailing whitespace, and so on. Consistent Markdown renders more
predictably across different renderers (GitHub, documentation sites, IDEs).

**Where in the code.**
`.markdownlint-cli2.yaml`, `.github/workflows/ci-run.yml` — the `docs-quality` job.

---

## 15. Python Companion Package

ZeroClaw is primarily a Rust binary. The Python package (`python/`) is a companion for use
cases where Python's AI/ML ecosystem is valuable.

### Python (3.10+)

**What it is.**
Python is an interpreted, dynamically typed general-purpose programming language known for
its readability and massive ecosystem of scientific and AI libraries. Python 3.10 introduced
structural pattern matching and improved type hints; requiring 3.10+ ensures these features
are available.

---

### LangGraph + LangChain

**What they are.**
LangChain is a Python framework for building applications with LLMs. It provides abstractions
for prompt templates, conversation memory, and chain composition. LangGraph builds on LangChain
to provide a *stateful graph execution engine*: you define nodes (agent, tools) and edges
(conditions for moving between nodes), and LangGraph executes the graph, managing state across
iterations.

**Why ZeroClaw uses them.**
Some LLM providers (particularly Chinese models like GLM-5) have inconsistent native tool-
calling behavior. LangGraph guarantees that tool calls are executed correctly regardless of
the model, by managing the agent loop explicitly. The `create_agent` function in
`python/zeroclaw_tools/agent.py` creates a LangGraph `StateGraph` with an agent node and a
tools node.

**Where in the code.**
`python/zeroclaw_tools/agent.py`, `python/pyproject.toml`.

**Learn more.**
https://langchain-ai.github.io/langgraph/
https://python.langchain.com

---

### httpx

**What it is.**
httpx is an async HTTP client for Python, similar in role to reqwest for Rust or requests for
synchronous Python. It supports HTTP/1.1 and HTTP/2, async/await, and connection pooling.

**Why ZeroClaw uses it.**
The Python package makes HTTP calls to LLM providers and external services. httpx is the
async HTTP client used by LangChain's OpenAI integration.

**Where in the code.**
`python/pyproject.toml` (`httpx>=0.25.0` dependency).

---

### hatchling

**What it is.**
hatchling is a Python build backend — the tool that takes your Python source code and packages
it into a distributable format (a wheel, the `.whl` file you install with `pip install`).
It is the backend for the Hatch project management tool. It is configured by `[build-system]`
in `pyproject.toml`.

**Where in the code.**
`python/pyproject.toml`.

---

### pytest

**What it is.**
pytest is Python's most popular testing framework. It discovers test functions (any function
named `test_*`), runs them, and reports failures. The `pytest-asyncio` extension adds support
for `async def` test functions.

**Where in the code.**
`python/tests/test_tools.py`, `python/pyproject.toml`.

---

### ruff

**What it is.**
ruff is a fast Python linter and formatter written in Rust. It replaces several older Python
tools (flake8, isort, pyupgrade) with a single, extremely fast tool. It checks for style
issues, unused imports, undefined variables, and many other patterns.

**Where in the code.**
`python/pyproject.toml` (`[tool.ruff]` configuration).

---

## 16. Error Handling

### anyhow (1.0)

**What it is.**
In Rust, errors must be handled explicitly — you cannot ignore a function that might fail.
The result of a fallible operation is a `Result<T, E>` type: either a success value `T` or
an error `E`. The challenge is that different functions return different error types, and
combining them requires converting between types manually.

`anyhow` solves this for application code. `anyhow::Error` is a single error type that can
hold any error from any library. The `?` operator automatically converts any error into
`anyhow::Error`. The `bail!` macro returns an `anyhow::Error` with a message. The
`context()` method wraps an error with additional context ("SQLite failed to open database:
permission denied" rather than just "permission denied").

**Why ZeroClaw uses it.**
Application-level code (the agent loop, the gateway, the config loading) uses anyhow
throughout. It makes error propagation ergonomic: functions return `anyhow::Result<T>`, and
`?` handles error conversion automatically.

**Where in the code.**
`src/main.rs`, `src/agent/loop_.rs`, `src/gateway/mod.rs`, `src/config/mod.rs` — any module
that runs application logic.

**Learn more.** https://docs.rs/anyhow

---

### thiserror (2.0)

**What it is.**
While anyhow is for applications, `thiserror` is for libraries. A library should define its
own specific error types so callers can programmatically inspect and handle different error
cases. `thiserror` provides a `#[derive(Error)]` macro that generates the standard error
trait implementations from enum variants with human-readable messages.

```rust
#[derive(Debug, thiserror::Error)]
pub enum SecurityError {
    #[error("Signature mismatch: expected HMAC does not match received")]
    SignatureMismatch,
    #[error("Pairing token expired: {token}")]
    TokenExpired { token: String },
}
```

**Why ZeroClaw uses it.**
Modules with specific, inspectable error conditions (security, config parsing, memory) define
their own error enums with `thiserror`. This lets callers match on specific error variants
rather than treating all errors as opaque strings.

**Where in the code.**
`src/security/`, `src/config/`, `src/memory/` — modules that define library-style APIs.

**Learn more.** https://docs.rs/thiserror

---

## 17. Other Utilities

### uuid (1.11)

**What it is.**
A UUID (Universally Unique Identifier) is a 128-bit number formatted as a string like
`550e8400-e29b-41d4-a716-446655440000`. UUIDs are designed so that independently generated
UUIDs have an astronomically low probability of collision. Version 4 UUIDs are entirely
random (using a CSPRNG). The `std` feature enables converting a UUID to/from a string.

**Why ZeroClaw uses it.**
Memory entries, webhook message IDs, session identifiers, and temporary file names are all
assigned UUID v4 identifiers. UUIDs avoid collisions without requiring a central counter or
database coordination.

**Where in the code.**
`src/memory/sqlite.rs` (`Uuid::new_v4()`), `src/gateway/mod.rs`, `src/channels/discord.rs`.

**Learn more.** https://docs.rs/uuid

---

### base64 (0.22)

**What it is.**
Base64 is an encoding scheme that converts arbitrary binary data into printable ASCII
characters. It is not encryption — it provides no confidentiality. It just makes binary data
safe to embed in contexts that only handle text (HTTP headers, JSON strings, email bodies).

For example, an image file is binary data. To include it in a JSON string, you base64-encode
it first: the binary bytes become a sequence of letters, numbers, `+`, and `/`. The receiver
decodes it back to binary.

**Why ZeroClaw uses it.**
LLM providers that support vision capabilities (sending images to the model) require image
data as base64-encoded strings in the JSON request body. ZeroClaw encodes screenshots and
image files to base64 before sending them to multimodal LLM providers. Discord bot tokens
also encode the bot's user ID as base64 at the start of the token string.

**Where in the code.**
`src/channels/discord.rs` (`base64_decode` for token parsing), `src/tools/` (screenshot
capture tools).

**Learn more.** https://docs.rs/base64

---

### regex (1.10)

**What it is.**
A regular expression (regex) is a pattern for matching text. For example, the pattern
`\d{4}-\d{2}-\d{2}` matches any string that looks like a date (four digits, dash, two
digits, dash, two digits). Regex engines can find matches, extract groups, and replace text.

**Why ZeroClaw uses it.**
ZeroClaw uses regex to extract structured data from LLM responses (tool call parsing), to
validate user input in onboarding, and to parse message formats from channel providers. The
`regex` crate compiles patterns at startup for efficient repeated use.

**Where in the code.**
`src/agent/dispatcher.rs` (tool call extraction), `src/channels/` (message parsing).

**Learn more.** https://docs.rs/regex

---

### glob (0.3)

**What it is.**
Glob patterns are file path wildcards, like `*.toml` (any file ending in .toml) or
`src/**/*.rs` (any .rs file anywhere inside src/). Glob pattern matching is simpler than
regex but sufficient for file path matching.

**Why ZeroClaw uses it.**
The hardware discovery module uses glob patterns to find serial port device paths on Linux
(`/dev/ttyACM*`, `/dev/ttyUSB*`) and macOS (`/dev/cu.usbmodem*`). Different microcontrollers
appear under different device names, and glob matching finds them all without enumerating
the `/dev` directory manually.

**Where in the code.**
`src/hardware/discover.rs`, `src/peripherals/serial.rs`.

**Learn more.** https://docs.rs/glob

---

### shellexpand (3.1)

**What it is.**
Shells (bash, zsh) automatically expand `~` to the user's home directory and `$VARIABLE` to
the value of environment variables. When users type a path like `~/zeroclaw/workspace` in
the config file, they expect `~` to expand to their home directory.

`shellexpand` performs this expansion in Rust: it takes a string like `~/zeroclaw/workspace`
and returns `/home/alice/zeroclaw/workspace` (on Linux) or `C:\Users\Alice\zeroclaw\workspace`
(on Windows).

**Why ZeroClaw uses it.**
Config file paths are specified by users who naturally use `~` for their home directory.
ZeroClaw must expand these before passing the path to file system operations.

**Where in the code.**
`src/config/mod.rs` — path expansion when loading configuration.

**Learn more.** https://docs.rs/shellexpand

---

### hostname (0.4)

**What it is.**
A hostname is the human-readable name of a computer on a network. On most systems it can be
retrieved with the `hostname` command. The `hostname` crate provides a Rust API for this.

**Why ZeroClaw uses it.**
The daemon's identity and the heartbeat system use the machine's hostname to identify which
instance of ZeroClaw is running. In multi-machine deployments, the hostname distinguishes
log entries and metrics from different nodes.

**Where in the code.**
`src/identity.rs`, `src/heartbeat/engine.rs`.

**Learn more.** https://docs.rs/hostname

---

### hex (0.4)

**What it is.**
Hex encoding converts binary data to a string of hexadecimal digits (0-9, a-f). For example,
the byte `0xFF` becomes the string `"ff"`. Hex is commonly used to display binary values that
need to be human-readable or compared as strings: cryptographic hashes, keys, and nonces.

**Why ZeroClaw uses it.**
The `SecretStore` formats encrypted secrets as hex-encoded strings prefixed with `enc2:` for
storage in the config file. Webhook HMAC values from headers arrive as hex strings and must
be decoded for comparison. `hex::encode` and `hex::decode` handle both directions.

**Where in the code.**
`src/security/secrets.rs` (ciphertext encoding), `src/gateway/mod.rs` (HMAC verification).

**Learn more.** https://docs.rs/hex

---

### directories (6.0)

**What it is.**
Different operating systems store user configuration files in different places: `~/.config/`
on Linux (XDG convention), `~/Library/Application Support/` on macOS, `C:\Users\Name\AppData\Roaming\`
on Windows. The `directories` crate provides a cross-platform API for finding these standard
directories without hard-coding platform-specific paths.

**Why ZeroClaw uses it.**
ZeroClaw stores its configuration and workspace data in the user's home directory. Using
`directories` ensures the config is stored in the platform-appropriate location on every
supported OS without special-casing each platform.

**Where in the code.**
`src/config/mod.rs` — finding the default config directory.

**Learn more.** https://docs.rs/directories

---

*Document generated for ZeroClaw 0.1.0 (toolchain 1.92.0, edition 2021).*
*For the authoritative list of dependencies, see `Cargo.toml` and `Cargo.lock`.*
