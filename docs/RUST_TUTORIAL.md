# Rust for ZeroClaw Developers

A self-contained Rust textbook for developers joining the ZeroClaw project. Written for people who already know C, C++, or Java and want to be productive in Rust quickly. Every concept is grounded in real code from `src/`.

---

## Table of Contents

1. [Part 1: Getting Started](#part-1-getting-started)
2. [Part 2: Basic Syntax](#part-2-basic-syntax)
3. [Part 3: Ownership and Borrowing](#part-3-ownership-and-borrowing)
4. [Part 4: Structs and Enums](#part-4-structs-and-enums)
5. [Part 5: Error Handling](#part-5-error-handling)
6. [Part 6: Traits](#part-6-traits)
7. [Part 7: Collections and Iterators](#part-7-collections-and-iterators)
8. [Part 8: Modules and Crates](#part-8-modules-and-crates)
9. [Part 9: Async Programming](#part-9-async-programming)
10. [Part 10: Smart Pointers and Interior Mutability](#part-10-smart-pointers-and-interior-mutability)
11. [Part 11: Testing](#part-11-testing)
12. [Part 12: Common Patterns in ZeroClaw](#part-12-common-patterns-in-zeroclaw)
13. [Part 13: Cheat Sheet](#part-13-cheat-sheet)

---

## Part 1: Getting Started

### What Is Rust and Why Does It Exist?

Rust is a systems programming language created at Mozilla Research, with its first stable release in 2015. It targets the same domain as C and C++: low-level, high-performance code where you need direct control over memory and hardware. But Rust adds something neither C nor C++ has: a compile-time guarantee that your program is memory-safe.

**Problems Rust solves vs C/C++:**

- **No dangling pointers.** C lets you free memory and then accidentally use the freed address. Rust's compiler refuses to compile programs that would do this.
- **No data races.** Sharing mutable data between threads in C++ requires careful manual locking. Rust makes data races a compile error.
- **No null pointer dereferences.** C has `NULL`. Rust has no null at all. You use `Option<T>` instead, and the compiler forces you to handle the "no value" case.
- **No buffer overflows.** Array accesses are bounds-checked at runtime (or provably safe at compile time).
- **No undefined behavior from memory bugs.** Entire classes of CVEs simply cannot exist in safe Rust.

**Problems Rust solves vs Java:**

- **No garbage collector.** Java's GC pauses threads to collect memory. Rust frees memory deterministically, the moment a value goes out of scope. This gives Rust performance similar to C with latency you cannot achieve in Java.
- **No JVM startup cost.** Rust compiles to a native binary. ZeroClaw starts in milliseconds.
- **Smaller binary size.** The ZeroClaw release binary is optimized for minimal footprint — not something Java can offer.
- **True zero-cost abstractions.** Rust's iterators, generics, and traits compile down to the same machine code as hand-written loops. Java's abstraction layers have runtime overhead.

ZeroClaw is Rust-first because it runs as an autonomous agent that may execute shell commands, handle webhooks, and interact with hardware. Memory safety and deterministic performance are non-negotiable there.

---

### Installing Rust

Rust is installed through `rustup`, a toolchain manager similar to `nvm` for Node.js or `pyenv` for Python.

**On macOS and Linux:**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the on-screen prompts. Choose "Proceed with installation (default)". Then reload your shell:

```bash
source ~/.cargo/env
```

**On Windows:**

Download and run `rustup-init.exe` from https://rustup.rs. You will also need the Microsoft C++ build tools. The installer will tell you if they are missing and link to the Visual Studio Build Tools installer.

**Verify your installation:**

```bash
rustc --version    # e.g. rustc 1.82.0 (f6e511eec 2024-10-15)
cargo --version    # e.g. cargo 1.82.0 (8f40fc59f 2024-08-21)
```

**The three tools you will use constantly:**

- `rustc` — the compiler. You rarely call this directly.
- `cargo` — the build tool and package manager. You call this constantly.
- `rustup` — the toolchain manager. Use it to update Rust or install additional targets.

---

### The Toolchain Concept

Rust releases come in three channels:
- **stable** — released every six weeks, production ready
- **beta** — release candidate for the next stable
- **nightly** — daily build, has experimental features, sometimes used for benchmarking tools

ZeroClaw uses stable Rust. When a project pins a specific toolchain version, it uses a `rust-toolchain.toml` file in the project root:

```toml
[toolchain]
channel = "stable"
```

If you open ZeroClaw and see this file, `cargo` will automatically download and use the specified toolchain. You do not need to do anything special.

Rust also has **editions**. Editions let the language evolve without breaking old code. The current edition is 2021. ZeroClaw's `Cargo.toml` declares:

```toml
[package]
edition = "2021"
```

This tells the compiler which syntax rules and standard library features to use. Editions are a package-level setting — they do not affect other crates you depend on.

---

### Your First Program: Hello World

Create a new project:

```bash
cargo new hello_world
cd hello_world
```

This creates:

```
hello_world/
  Cargo.toml      # project manifest
  src/
    main.rs       # entry point
```

Open `src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

Run it:

```bash
cargo run
```

You will see:

```
   Compiling hello_world v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
     Running `target/debug/hello_world`
Hello, world!
```

Other essential cargo commands:

```bash
cargo build           # compile without running (debug mode)
cargo build --release # compile with optimizations (production)
cargo test            # compile and run all tests
cargo check           # type-check without producing a binary (fast)
cargo fmt             # auto-format all source files
cargo clippy          # run the linter
```

The ZeroClaw validation matrix (from `CLAUDE.md`) requires all three:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

---

### Project Structure

```
zeroclaw/
  Cargo.toml        # package manifest and dependency list
  Cargo.lock        # exact locked versions of all dependencies
  src/
    main.rs         # binary entry point (fn main)
    lib.rs          # library root (public API for other crates or tests)
    agent/          # module directory
      mod.rs        # declares the agent module's public API
    tools/
      mod.rs
      traits.rs     # Tool trait definition
    providers/
      traits.rs     # Provider trait definition
    ...
```

`Cargo.toml` is the project manifest. A simplified version:

```toml
[package]
name = "zeroclaw"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1.0"
tokio = { version = "1.42", features = ["rt-multi-thread", "macros"] }
serde = { version = "1.0", features = ["derive"] }
```

`Cargo.lock` records the exact version of every transitive dependency. It is committed to version control for binary projects (like ZeroClaw) so that every developer builds the exact same binary. Libraries do not commit `Cargo.lock`.

---

## Part 2: Basic Syntax

### Variables and Mutability

In Rust, **all variables are immutable by default**. This is the opposite of C and Java, where variables are mutable unless you say `const` or `final`.

```rust
fn main() {
    let x = 5;
    // x = 6;  // ERROR: cannot assign twice to immutable variable

    let mut y = 10;
    y = 20;  // OK: y is declared mutable
}
```

Compare to other languages:

| Language | Mutable by default | Immutable keyword |
|----------|--------------------|-------------------|
| C        | Yes                | `const`           |
| Java     | Yes                | `final`           |
| Rust     | No                 | (default)         |

Why does this matter? Immutability by default prevents an entire class of bugs where a function accidentally modifies data you expected to stay constant. The compiler enforces it, so there is no runtime cost.

**Constants** in Rust use `const` (not `let`) and must have a type annotation and a value known at compile time:

```rust
const MAX_RETRIES: u32 = 3;
const KEY_LEN: usize = 32;  // from src/security/secrets.rs
```

**Type inference**: Rust infers types in most cases, but you can always annotate explicitly:

```rust
let count: usize = 0;
let name: String = String::from("zeroclaw");
```

---

### Data Types

**Integers:**

| Type   | Size    | Signed | Notes                          |
|--------|---------|--------|--------------------------------|
| `i8`   | 8-bit   | Yes    |                                |
| `i32`  | 32-bit  | Yes    | Default integer type           |
| `i64`  | 64-bit  | Yes    |                                |
| `u8`   | 8-bit   | No     | Byte value                     |
| `u16`  | 16-bit  | No     | Port numbers (0–65535)         |
| `u32`  | 32-bit  | No     |                                |
| `u64`  | 64-bit  | No     |                                |
| `usize`| pointer | No     | Array indices, collection sizes|
| `isize`| pointer | Yes    | Pointer arithmetic             |

Use `usize` for array indices and collection sizes — it matches the platform's pointer width. From ZeroClaw's `src/config/schema.rs`:

```rust
pub cache_max: usize,  // collection size
pub max_actions_per_hour: u32,  // a bounded count
pub port: u16,  // TCP port number
```

**Floats:**

```rust
let temperature: f64 = 0.7;  // 64-bit double, used throughout zeroclaw
let weight: f32 = 0.3;       // 32-bit float
```

**Booleans:**

```rust
let is_valid: bool = true;
let auto_save: bool = false;
```

**Characters:** A Rust `char` is a Unicode scalar value (4 bytes), not a byte like in C:

```rust
let crab: char = '🦀';  // perfectly valid
```

---

### String vs &str — The Critical Distinction

This is the most common confusion point for newcomers. Rust has two string types:

**`String`** — an owned, heap-allocated, growable string. Think of it like `std::string` in C++ or `StringBuilder` in Java.

**`&str`** — a borrowed string slice. A view into some existing string data. It does not own the data. Think of it like `const char*` in C (but with a length and safety guarantees).

```rust
// String: owned, on the heap, can grow
let owned: String = String::from("hello");
let also_owned: String = "hello".to_string();

// &str: borrowed view, usually a literal in the binary or a slice of a String
let borrowed: &str = "hello";  // string literal, lives in the binary
let slice: &str = &owned[0..3]; // borrow the first 3 bytes of owned
```

**When to use which:**

- Use `&str` for function parameters when you just need to read the string:
  ```rust
  fn greet(name: &str) {
      println!("Hello, {name}");
  }
  ```
- Use `String` when you need to own the string, store it in a struct, or modify it:
  ```rust
  struct Config {
      name: String,  // Config owns this data
  }
  ```
- Use `impl Into<String>` in constructors to accept both:
  ```rust
  // From src/providers/traits.rs
  pub fn system(content: impl Into<String>) -> Self {
      Self {
          role: "system".into(),
          content: content.into(),  // works with both &str and String
      }
  }
  ```

**Converting between them:**

```rust
let s: String = String::from("hello");
let slice: &str = &s;           // String -> &str: just borrow it
let owned: String = slice.to_string();  // &str -> String: allocates
let owned2: String = slice.into();      // same thing, shorter
```

**Common beginner mistake:** Returning a `&str` that refers to a local `String`:

```rust
// ERROR: this does not compile
fn make_greeting() -> &str {
    let s = String::from("hello");
    &s  // ERROR: s is dropped at end of function, &s would dangle
}

// CORRECT: return the owned String
fn make_greeting() -> String {
    String::from("hello")
}
```

The compiler catches this and gives a clear error message. This is exactly the kind of dangling pointer that causes crashes in C.

---

### Functions

```rust
fn add(x: i32, y: i32) -> i32 {
    x + y  // no semicolon: this is the return expression
}

fn greet(name: &str) {
    // no -> means the return type is () (unit, like void)
    println!("Hello, {name}!");
}
```

Notice: the last expression in a function body without a semicolon is the return value. You can also use `return` explicitly, but idiomatic Rust uses the expression form:

```rust
fn max_of(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }  // if is an expression that returns a value
}
```

---

### Comments

```rust
// Single-line comment

/*
   Multi-line comment
*/

/// This is a doc comment — it generates HTML documentation.
/// It goes above the item it documents.
///
/// # Examples
/// ```
/// let x = my_function(5);
/// ```
pub fn my_function(x: i32) -> i32 {
    x * 2
}
```

Doc comments (starting with `///`) become searchable documentation when you run `cargo doc --open`.

---

### Control Flow

**if/else — expressions, not statements:**

```rust
let score = 85;
let grade = if score >= 90 {
    "A"
} else if score >= 80 {
    "B"
} else {
    "C"
};
```

Note that `if` returns a value here. Both branches must return the same type.

**loop — infinite loop with break-value:**

```rust
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;  // loop returns 20
    }
};
```

**while:**

```rust
while !done {
    done = try_operation();
}
```

**for — always iterates over an iterator:**

```rust
// Range
for i in 0..5 {
    println!("{i}");  // 0, 1, 2, 3, 4
}

// Inclusive range
for i in 0..=5 {
    println!("{i}");  // 0, 1, 2, 3, 4, 5
}

// Slice
let names = ["alice", "bob", "charlie"];
for name in &names {
    println!("{name}");
}
```

---

### match — Pattern Matching

`match` is like `switch` in C, but dramatically more powerful. It is exhaustive: the compiler forces you to handle every possible case.

```rust
let number = 3;
match number {
    1 => println!("one"),
    2 | 3 => println!("two or three"),   // OR pattern
    4..=9 => println!("four to nine"),   // range pattern
    _ => println!("something else"),      // wildcard, like default:
}
```

Unlike C's `switch`, there is no fall-through. Each arm is independent.

`match` works on any type, including structs and enums. From `src/memory/traits.rs`:

```rust
match self {
    Self::Core => write!(f, "core"),
    Self::Daily => write!(f, "daily"),
    Self::Conversation => write!(f, "conversation"),
    Self::Custom(name) => write!(f, "{name}"),  // destructure the enum variant's data
}
```

**match is an expression:**

```rust
let description = match level {
    AutonomyLevel::ReadOnly => "read-only mode",
    AutonomyLevel::Supervised => "supervised mode",
    AutonomyLevel::Full => "full autonomy",
};
```

**Common beginner mistake:** Forgetting the wildcard arm and getting a compile error because not all cases are covered. This is a feature, not a bug — the compiler forces you to think about every case.

---

## Part 3: Ownership and Borrowing

This is the section most newcomers struggle with most. Read it slowly. Every concept here has a direct impact on how you write ZeroClaw code.

### What Is Ownership?

In C, you manually `malloc` and `free` memory. It is your job to track whether memory is still in use. Get it wrong and you have a use-after-free bug or a memory leak.

In Java, the garbage collector tracks all live references and frees memory when nothing refers to it. This is safe but introduces unpredictable GC pauses and higher memory usage.

Rust takes a third approach: **ownership**. Every value has exactly one owner. When the owner goes out of scope, the value is automatically freed. No GC, no manual free, no leaks, no dangling pointers.

### The Three Rules of Ownership

1. Every value in Rust has exactly one owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value is dropped (freed).

```rust
fn main() {
    let s = String::from("hello");  // s owns the String
    // s is the owner; when main() ends, s is dropped and the String is freed
}
```

### Move Semantics

When you assign a heap-allocated value to another variable, **ownership moves**. The original variable is no longer valid.

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 is MOVED into s2. s1 is now invalid.

// println!("{s1}");  // ERROR: s1 was moved
println!("{s2}");     // OK
```

This is different from C++ where assignment copies by default, and Java where assignment copies the reference (so both variables point to the same object). In Rust, assigning moves the value and the original is gone.

For simple types that fit in a register (`i32`, `f64`, `bool`, `char`), Rust copies the value instead of moving it, because copying is cheap:

```rust
let x = 5;
let y = x;  // x is COPIED, not moved
println!("{x} {y}");  // both are valid
```

Types that implement the `Copy` trait behave this way. `String` does not implement `Copy` because copying it would require a heap allocation.

### References and Borrowing

If you want to use a value without taking ownership, you borrow it using a reference `&`:

```rust
fn print_length(s: &String) {  // borrows s, does not take ownership
    println!("Length: {}", s.len());
}

fn main() {
    let s = String::from("hello");
    print_length(&s);  // pass a reference
    println!("{s}");   // s is still valid here
}
```

This is like passing a pointer in C, but the compiler guarantees the reference is always valid.

**Mutable references** use `&mut`:

```rust
fn append_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{s}");  // "hello, world"
}
```

**The critical constraint on mutable references:**

At any given time, you can have either:
- Any number of immutable references (`&T`), OR
- Exactly one mutable reference (`&mut T`)

But never both at the same time. This is what prevents data races at compile time.

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;
// let r3 = &mut s;  // ERROR: cannot borrow s as mutable while immutably borrowed

println!("{r1} {r2}");
// r1 and r2 are no longer used after here

let r3 = &mut s;  // OK now — r1 and r2 are out of scope
r3.push_str("!");
```

### The Borrow Checker

The borrow checker is the part of the Rust compiler that enforces ownership and borrowing rules. It analyzes your program at compile time and rejects code that would violate the rules.

When the borrow checker rejects your code, it gives you an error message explaining what went wrong and often suggests a fix. These errors feel painful at first, but each one is preventing a real bug.

From `src/providers/traits.rs`, here is the borrow checker in action with a legitimate use pattern:

```rust
// chat_with_history borrows messages (no ownership transfer)
async fn chat_with_history(
    &self,
    messages: &[ChatMessage],  // borrow a slice — no copy needed
    model: &str,               // borrow a string
    temperature: f64,          // f64 is Copy, so this is fine
) -> anyhow::Result<String> {
    // Find the system message (searching without modifying)
    let system = messages
        .iter()
        .find(|m| m.role == "system")
        .map(|m| m.content.as_str());  // borrow the content, do not clone
    // ...
}
```

### Lifetimes

Sometimes the compiler cannot figure out how long a reference stays valid. That is when you need **lifetime annotations**.

A lifetime annotation is written as `'a` (a tick followed by a name). It does not change how long data lives — it just tells the compiler how long references are expected to be valid.

**When you need lifetimes:**

When a function returns a reference, and the compiler cannot tell which input the reference comes from:

```rust
// Without lifetime annotation, this does not compile:
fn longest(x: &str, y: &str) -> &str {  // ERROR: missing lifetime specifier

// With lifetime annotation:
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The `'a` says: "the returned reference lives at least as long as both input references." The caller is guaranteed the return value does not outlive either input.

**When you do NOT need lifetimes:**

Most of the time. The compiler has lifetime elision rules that infer them in common patterns. You only write explicit lifetimes when the compiler complains.

From `src/providers/traits.rs`:

```rust
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],  // borrows messages from somewhere
    pub tools: Option<&'a [ToolSpec]>,  // borrows tools from the same lifetime
}
```

The `'a` here says: "the `ChatRequest` cannot outlive the slices it borrows." This is correct — a request object should not persist after the messages it references are freed.

**Common lifetime errors and fixes:**

1. **Returning a reference to a local variable:**
   ```rust
   // ERROR
   fn broken() -> &str {
       let s = String::from("oops");
       &s  // s is dropped when function returns
   }
   // FIX: return the owned value
   fn fixed() -> String {
       String::from("ok")
   }
   ```

2. **Storing a reference in a struct without annotating the lifetime:**
   ```rust
   // ERROR
   struct Holder {
       data: &str,
   }
   // FIX: annotate the lifetime
   struct Holder<'a> {
       data: &'a str,
   }
   ```

**Practical advice:** In ZeroClaw, you will rarely write explicit lifetimes. They appear in `ChatRequest<'a>` and a few other places, but most code uses owned types (`String`, `Vec`, `Box`) or simple borrows that the compiler handles automatically.

---

## Part 4: Structs and Enums

### Defining Structs

A Rust struct is like a C struct or a Java class without inheritance.

```rust
// Simple struct
struct Point {
    x: f64,
    y: f64,
}

// Instantiate it
let p = Point { x: 3.0, y: 4.0 };
println!("{}", p.x);  // field access
```

Compare to C:
```c
// C
typedef struct { double x; double y; } Point;
```

Rust structs are value types by default (stored on the stack or moved). To put them on the heap, you use `Box<T>` or `Arc<T>` (covered in Part 10).

**Derive macros** automatically generate common trait implementations:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}
```

- `Debug` — lets you print the struct with `{:?}`: `println!("{:?}", result);`
- `Clone` — adds a `.clone()` method to make a deep copy
- `Serialize, Deserialize` — enables JSON serialization (from the `serde` crate)

From `src/tools/traits.rs` — the actual ZeroClaw code:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}
```

### impl Blocks — Adding Methods

Methods go in `impl` blocks, separate from the struct definition:

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Associated function (no self) — like a static method in Java
    pub fn new(width: f64, height: f64) -> Self {
        Self { width, height }
    }

    // Method with immutable self — read-only access
    pub fn area(&self) -> f64 {
        self.width * self.height
    }

    // Method with mutable self — can modify the struct
    pub fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }

    // Method that consumes self — takes ownership
    pub fn into_square(self) -> Rectangle {
        let side = (self.area()).sqrt();
        Rectangle { width: side, height: side }
    }
}

let rect = Rectangle::new(4.0, 3.0);  // associated function: Rectangle::new
println!("{}", rect.area());           // method: rect.area()
```

From `src/providers/traits.rs` — `ChatMessage` constructors:

```rust
impl ChatMessage {
    // Associated function: ChatMessage::system("prompt")
    pub fn system(content: impl Into<String>) -> Self {
        Self {
            role: "system".into(),
            content: content.into(),
        }
    }

    pub fn user(content: impl Into<String>) -> Self {
        Self {
            role: "user".into(),
            content: content.into(),
        }
    }

    pub fn assistant(content: impl Into<String>) -> Self {
        Self {
            role: "assistant".into(),
            content: content.into(),
        }
    }
}
```

### The self Parameter

| Form         | Meaning                                    | Equivalent in Java    |
|--------------|--------------------------------------------|-----------------------|
| `&self`      | Immutable borrow of self                   | `this` (read methods) |
| `&mut self`  | Mutable borrow of self                     | `this` (write methods)|
| `self`       | Takes ownership of self (consumes it)      | No direct equivalent  |
| `Self`       | The type of the implementing struct/trait  | (return type only)    |

---

### Enums — Rust's Most Powerful Feature

In C, enums are just integers with names. In Rust, each enum variant can carry different data:

```rust
// C-style: just a discriminant
enum Direction { North, South, East, West }

// Rust-style: variants can hold data
enum Message {
    Quit,
    Move { x: i32, y: i32 },       // struct-like variant
    Write(String),                   // tuple-like variant
    ChangeColor(u8, u8, u8),         // multiple values
}
```

You must use `match` (or `if let`) to get data out of an enum variant.

From `src/memory/traits.rs` — `MemoryCategory`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum MemoryCategory {
    Core,
    Daily,
    Conversation,
    Custom(String),   // carries a String value
}
```

And from `src/security/policy.rs` — `AutonomyLevel`:

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Serialize, Deserialize)]
pub enum AutonomyLevel {
    ReadOnly,
    #[default]
    Supervised,  // this is the default variant
    Full,
}
```

---

### Option<T> — Rust's Null Replacement

Rust has no `null`, `nil`, or `None` as a surprise value. Instead, `Option<T>` is an enum that explicitly represents "maybe a value":

```rust
enum Option<T> {
    Some(T),   // there is a value
    None,      // there is no value
}
```

This forces you to handle the missing case. You cannot call methods on an `Option<T>` as if it were `T` — you must unwrap it safely.

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 42 {
        Some(String::from("zeroclaw_user"))
    } else {
        None
    }
}

// Pattern 1: match
match find_user(42) {
    Some(name) => println!("Found: {name}"),
    None => println!("Not found"),
}

// Pattern 2: if let (when you only care about Some)
if let Some(name) = find_user(42) {
    println!("Found: {name}");
}

// Pattern 3: unwrap_or (provide a default)
let name = find_user(99).unwrap_or_else(|| String::from("guest"));

// Pattern 4: ? operator (propagate None as an error — in Result context)
fn do_something() -> Option<()> {
    let name = find_user(42)?;  // returns None from this function if None
    println!("{name}");
    Some(())
}
```

From `src/tools/traits.rs`:

```rust
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,  // error message, or None if successful
}
```

From `src/providers/traits.rs`:

```rust
pub struct ChatResponse {
    pub text: Option<String>,        // might have text
    pub tool_calls: Vec<ToolCall>,   // might have tool calls
}

impl ChatResponse {
    pub fn text_or_empty(&self) -> &str {
        self.text.as_deref().unwrap_or("")  // safely get &str or ""
    }
}
```

**Common beginner mistake:** Calling `.unwrap()` everywhere. This will panic if the value is `None`. In production code, use `?`, `unwrap_or`, `unwrap_or_else`, or `match` instead.

---

### Result<T, E> — Rust's Error Handling

`Result` is another enum, used for operations that can fail:

```rust
enum Result<T, E> {
    Ok(T),   // success with value T
    Err(E),  // failure with error E
}
```

From `src/tools/traits.rs`:

```rust
async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
    Ok(ToolResult {
        success: true,
        output: "done".to_string(),
        error: None,
    })
}
```

`anyhow::Result<T>` is a shorthand for `Result<T, anyhow::Error>`, where the error type is a flexible error that can hold any error. This is the standard pattern in ZeroClaw.

---

## Part 5: Error Handling

### Why Rust Has No Exceptions

Exceptions in Java/C++ are invisible. A function can throw at any call site, and the caller is not warned by the type system. This makes it easy to forget error handling.

In Rust, if a function can fail, it returns `Result<T, E>`. The caller must explicitly handle it. This is enforced at compile time.

### Result<T, E> in Depth

```rust
use std::fs;

fn read_config() -> Result<String, std::io::Error> {
    let contents = fs::read_to_string("config.toml")?;  // ? propagates error
    Ok(contents)
}

fn main() {
    match read_config() {
        Ok(contents) => println!("Config: {contents}"),
        Err(e) => eprintln!("Failed to read config: {e}"),
    }
}
```

### The ? Operator

The `?` operator is shorthand for "if this is an `Err`, return it from this function; otherwise unwrap the `Ok` value." It dramatically reduces boilerplate:

```rust
// Without ?
fn read_and_process() -> Result<String, MyError> {
    let data = match read_file() {
        Ok(d) => d,
        Err(e) => return Err(e.into()),
    };
    let result = match process(data) {
        Ok(r) => r,
        Err(e) => return Err(e.into()),
    };
    Ok(result)
}

// With ?
fn read_and_process() -> Result<String, MyError> {
    let data = read_file()?;
    let result = process(data)?;
    Ok(result)
}
```

From `src/security/secrets.rs`:

```rust
pub fn encrypt(&self, plaintext: &str) -> Result<String> {
    // ...
    let key_bytes = self.load_or_create_key()?;  // propagate IO error
    let cipher = ChaCha20Poly1305::new(key);

    let nonce = ChaCha20Poly1305::generate_nonce(&mut OsRng);
    let ciphertext = cipher
        .encrypt(&nonce, plaintext.as_bytes())
        .map_err(|e| anyhow::anyhow!("Encryption failed: {e}"))?;  // convert and propagate

    Ok(format!("enc2:{}", hex_encode(&blob)))
}
```

### unwrap() and expect()

Both of these extract the `Ok` or `Some` value and **panic** if it is `Err` or `None`:

```rust
let value = some_option.unwrap();         // panics with generic message
let value = some_option.expect("msg");    // panics with your message
```

**When to use them:**
- In tests: panicking makes the test fail immediately with a clear message.
- When you know the value cannot be `None`/`Err` in context and want to document that fact.
- In early prototype code (but replace with proper error handling before shipping).

**When to avoid them:**
- In library or production code paths where the failure could happen at runtime.
- When the calling context can recover from the failure.

From tests in `src/tools/traits.rs`:

```rust
#[test]
fn spec_uses_tool_metadata_and_schema() {
    let tool = DummyTool;
    let spec = tool.spec();
    assert_eq!(spec.name, "dummy_tool");
    // .unwrap() is fine in tests — panic = test failure
    let json = serde_json::to_string(&spec).unwrap();
    assert!(json.contains("dummy_tool"));
}
```

### anyhow::Result — Simplified Error Handling

The `anyhow` crate provides a flexible error type that ZeroClaw uses throughout:

```rust
use anyhow::{Result, Context, bail};

// anyhow::Result<T> = Result<T, anyhow::Error>
// anyhow::Error can hold any error type

fn load_config(path: &str) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {path}"))?;
    //           ^^^^^^^^^ adds context to the error message

    let config: Config = toml::from_str(&contents)
        .context("Failed to parse config TOML")?;

    Ok(config)
}
```

### bail! — Early Return with Error

`bail!` is a macro from `anyhow` that returns an `Err` immediately:

```rust
use anyhow::bail;

fn validate_port(port: u16) -> Result<()> {
    if port == 0 {
        bail!("Port 0 is not allowed");
    }
    Ok(())
}
```

From `src/main.rs`:

```rust
if interactive && channels_only {
    bail!("Use either --interactive or --channels-only, not both");
}
```

### thiserror — Custom Error Types

For library code, you want typed errors. The `thiserror` crate generates `std::error::Error` implementations:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StreamError {
    #[error("HTTP error: {0}")]
    Http(reqwest::Error),

    #[error("JSON parse error: {0}")]
    Json(serde_json::Error),

    #[error("Invalid SSE format: {0}")]
    InvalidSse(String),

    #[error("Provider error: {0}")]
    Provider(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),  // auto-convert from std::io::Error
}
```

This is from `src/providers/traits.rs`. The `#[error("...")]` attribute provides the `Display` implementation. The `#[from]` attribute generates an automatic conversion from the inner error type.

---

## Part 6: Traits

### What Are Traits?

A trait is a collection of method signatures that a type can implement. They are similar to Java interfaces and C++ abstract base classes, but more powerful.

```rust
// Define a trait
trait Greet {
    fn greeting(&self) -> String;

    // Default implementation (overridable)
    fn greet(&self) {
        println!("{}", self.greeting());
    }
}

// Implement the trait for a type
struct ZeroClawAgent;

impl Greet for ZeroClawAgent {
    fn greeting(&self) -> String {
        String::from("ZeroClaw agent online.")
    }
    // greet() uses the default implementation
}
```

Compare to Java:
```java
// Java interface — similar concept
interface Greet {
    String greeting();
    default void greet() { System.out.println(greeting()); }
}
```

### Defining and Implementing Traits

From `src/tools/traits.rs` — the actual `Tool` trait:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait Tool: Send + Sync {
    /// Tool name (used in LLM function calling)
    fn name(&self) -> &str;

    /// Human-readable description
    fn description(&self) -> &str;

    /// JSON schema for parameters
    fn parameters_schema(&self) -> serde_json::Value;

    /// Execute the tool with given arguments
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;

    /// Get the full spec for LLM registration — has a default implementation
    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}
```

And an implementation of that trait (from the same file's tests):

```rust
struct DummyTool;

#[async_trait]
impl Tool for DummyTool {
    fn name(&self) -> &str { "dummy_tool" }

    fn description(&self) -> &str { "A deterministic test tool" }

    fn parameters_schema(&self) -> serde_json::Value {
        serde_json::json!({
            "type": "object",
            "properties": {
                "value": { "type": "string" }
            }
        })
    }

    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        Ok(ToolResult {
            success: true,
            output: args
                .get("value")
                .and_then(serde_json::Value::as_str)
                .unwrap_or_default()
                .to_string(),
            error: None,
        })
    }
}
```

`spec()` is not overridden — `DummyTool` gets the default implementation.

### Trait Bounds and Generics

Generics in Rust look like generics in Java but work differently (they use monomorphization — the compiler generates separate code for each concrete type, so there is no runtime overhead):

```rust
// Generic function: T must implement the Display trait
fn print_item<T: std::fmt::Display>(item: T) {
    println!("{item}");
}

// Multiple bounds with +
fn print_and_debug<T: std::fmt::Display + std::fmt::Debug>(item: T) {
    println!("{item}");
    println!("{item:?}");
}

// Where clause (more readable for complex bounds)
fn complex_function<T, U>(t: T, u: U) -> String
where
    T: Clone + std::fmt::Debug,
    U: std::fmt::Display,
{
    format!("{u}")
}
```

In ZeroClaw, this appears in function signatures that accept any implementation of a trait:

```rust
pub fn system(content: impl Into<String>) -> Self {
    Self {
        role: "system".into(),
        content: content.into(),
    }
}
```

`impl Into<String>` is shorthand for `<T: Into<String>>(content: T)`. It means "accept any type that can be converted into a `String`" — so you can pass either `&str` or `String`.

### dyn — Dynamic Dispatch

When you use generics, the compiler generates specialized code for each type at compile time (static dispatch, zero runtime cost). When you use `dyn Trait`, the compiler uses a vtable lookup at runtime (dynamic dispatch, small runtime cost, but more flexible).

```rust
// Static dispatch — compiler generates one version per concrete type
fn static_tool<T: Tool>(tool: &T) {
    println!("{}", tool.name());
}

// Dynamic dispatch — one function, works with any Tool at runtime
fn dynamic_tool(tool: &dyn Tool) {
    println!("{}", tool.name());
}

// Storing heterogeneous tools in a Vec requires dyn
let tools: Vec<Box<dyn Tool>> = vec![
    Box::new(ShellTool::new()),
    Box::new(FileReadTool::new()),
    Box::new(MemoryTool::new()),
];
```

From `src/memory/sqlite.rs`:

```rust
pub struct SqliteMemory {
    embedder: Arc<dyn EmbeddingProvider>,  // dyn: any EmbeddingProvider at runtime
    // ...
}
```

ZeroClaw uses `dyn` for the factory pattern — you can swap out providers, channels, and tools at runtime without recompiling.

### The Provider Trait — A Complete ZeroClaw Example

From `src/providers/traits.rs`:

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    /// Query provider capabilities (default: no native tool calling)
    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities::default()
    }

    /// Simple one-shot chat
    async fn simple_chat(
        &self,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String> {
        self.chat_with_system(None, message, model, temperature).await
    }

    /// One-shot chat with optional system prompt — REQUIRED (no default)
    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String>;

    // ... more methods with defaults
}
```

A new provider only needs to implement `chat_with_system`. Everything else has a default. This follows the Interface Segregation Principle: implementors are not burdened with methods they do not need to customize.

### How the Factory Pattern Uses Traits

ZeroClaw registers implementations in factory functions. This lets the runtime pick a provider by name from config:

```rust
// Pseudocode of the factory pattern
pub fn create_provider(name: &str, config: &Config) -> anyhow::Result<Arc<dyn Provider>> {
    match name {
        "openai" => Ok(Arc::new(OpenAIProvider::new(config)?)),
        "anthropic" => Ok(Arc::new(AnthropicProvider::new(config)?)),
        "ollama" => Ok(Arc::new(OllamaProvider::new(config)?)),
        other => bail!("Unknown provider: {other}"),
    }
}
```

The caller gets back `Arc<dyn Provider>` — it does not care which concrete type it is. All it knows is that the value implements `Provider`.

---

## Part 7: Collections and Iterators

### Vec<T> — Dynamic Arrays

`Vec<T>` is Rust's equivalent of `std::vector<T>` in C++ or `ArrayList<T>` in Java:

```rust
let mut tools: Vec<String> = Vec::new();
tools.push("shell".to_string());
tools.push("file_read".to_string());
tools.push("memory".to_string());

println!("{}", tools.len());     // 3
println!("{}", tools[0]);        // "shell"
println!("{:?}", tools);         // ["shell", "file_read", "memory"]

// Create with initial values
let nums = vec![1, 2, 3, 4, 5];  // vec! macro

// Iterate
for tool in &tools {  // borrow: does not consume the Vec
    println!("{tool}");
}
```

### HashMap<K, V> — Key-Value Maps

```rust
use std::collections::HashMap;

let mut config: HashMap<String, String> = HashMap::new();
config.insert("provider".to_string(), "openai".to_string());
config.insert("model".to_string(), "gpt-4o".to_string());

// Get a value
if let Some(provider) = config.get("provider") {
    println!("Using provider: {provider}");
}

// Entry API — insert only if absent
config.entry("temperature".to_string())
      .or_insert("0.7".to_string());

// Iterate
for (key, value) in &config {
    println!("{key} = {value}");
}
```

From `src/config/schema.rs`:

```rust
pub agents: HashMap<String, DelegateAgentConfig>,
```

### Iterators — Rust's Functional Pipeline

Rust iterators are lazy — they compute values on demand. Chaining them together creates a processing pipeline:

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

let sum_of_even_squares: i32 = numbers
    .iter()           // create an iterator
    .filter(|&&n| n % 2 == 0)   // keep even numbers
    .map(|&n| n * n)             // square them
    .sum();                       // add them up
// sum_of_even_squares = 4 + 16 + 36 + 64 + 100 = 220
```

**Common iterator methods:**

| Method              | Description                              |
|---------------------|------------------------------------------|
| `.iter()`           | Borrow each element                      |
| `.into_iter()`      | Consume the collection, yield owned items|
| `.iter_mut()`       | Mutably borrow each element              |
| `.map(f)`           | Transform each element                   |
| `.filter(pred)`     | Keep elements where pred returns true    |
| `.find(pred)`       | Return first matching element            |
| `.any(pred)`        | True if any element matches              |
| `.all(pred)`        | True if all elements match               |
| `.count()`          | Number of elements                       |
| `.sum()`            | Sum of elements                          |
| `.collect()`        | Collect into a Vec or other collection   |
| `.enumerate()`      | Yield (index, element) pairs             |
| `.zip(other)`       | Pair elements from two iterators         |
| `.flat_map(f)`      | Map then flatten (like flatMap in Java)  |
| `.take(n)`          | First n elements                         |
| `.skip(n)`          | Skip first n elements                    |
| `.chain(other)`     | Concatenate two iterators                |

From `src/providers/traits.rs` — searching through messages:

```rust
let system = messages
    .iter()
    .find(|m| m.role == "system")      // find first system message
    .map(|m| m.content.as_str());       // extract its content as &str

let last_user = messages
    .iter()
    .rfind(|m| m.role == "user")       // rfind: search from the right
    .map(|m| m.content.as_str())
    .unwrap_or("");
```

From `src/main.rs` — checking aliases:

```rust
let is_active = p.name.eq_ignore_ascii_case(&current)
    || p.aliases
        .iter()
        .any(|alias| alias.eq_ignore_ascii_case(&current));
```

### Closures

Closures are anonymous functions that can capture variables from their surrounding scope:

```rust
let multiplier = 3;
let triple = |x| x * multiplier;  // captures `multiplier` from outer scope
println!("{}", triple(5));  // 15

// Closure with type annotations
let add: fn(i32, i32) -> i32 = |a, b| a + b;

// Closure used with iterator
let names = vec!["alice", "bob", "charlie"];
let upper: Vec<String> = names
    .iter()
    .map(|name| name.to_uppercase())  // closure: |param| body
    .collect();
```

---

## Part 8: Modules and Crates

### mod — Organizing Code

A **module** is a namespace. You declare modules with `mod`:

```rust
// In src/main.rs
mod agent;      // tells Rust to look for src/agent/mod.rs or src/agent.rs
mod tools;
mod providers;
mod security;
```

Inside a module, items are **private by default**. Use `pub` to make them visible outside:

```rust
// src/tools/mod.rs
pub mod traits;     // public: accessible outside the tools module
mod internal;       // private: only accessible within tools module

pub use traits::Tool;      // re-export Tool at the tools:: level
pub use traits::ToolResult;
```

### pub — Visibility

```rust
pub struct Public;            // visible everywhere
pub(crate) struct CrateOnly;  // visible within this crate only
pub(super) struct ParentOnly; // visible in the parent module
struct Private;               // visible only in this module
```

In ZeroClaw, most types in `src/tools/traits.rs` are `pub` because they form the public API of the tools subsystem.

### use — Bringing Items Into Scope

```rust
// Bring a specific item into scope
use std::collections::HashMap;

// Bring multiple items from the same module
use std::fs::{self, File, OpenOptions};

// Use an alias to avoid naming conflicts
use parking_lot::Mutex as PLMutex;
use std::sync::Mutex as StdMutex;

// Glob import (usually discouraged outside of prelude patterns)
use std::io::prelude::*;
```

From `src/memory/sqlite.rs`:

```rust
use super::embeddings::EmbeddingProvider;
use super::traits::{Memory, MemoryCategory, MemoryEntry};
use anyhow::Context;
use async_trait::async_trait;
use parking_lot::Mutex;
use rusqlite::{params, Connection};
use std::sync::Arc;
```

`super::` refers to the parent module (the `memory` module). This is how sub-modules refer to sibling modules.

### File-Based Modules

Rust maps module paths to file paths:

```
src/
  main.rs          -> root module (binary)
  lib.rs           -> root module (library)
  tools/
    mod.rs         -> mod tools { ... }
    traits.rs      -> mod tools::traits { ... }
  security/
    mod.rs
    secrets.rs     -> mod security::secrets { ... }
    policy.rs
```

When you write `mod tools;` in `main.rs`, Rust looks for either `src/tools.rs` or `src/tools/mod.rs`. ZeroClaw uses the `mod.rs` convention for modules with multiple files.

### Crates — External Dependencies

A **crate** is a compiled Rust library or binary. You add dependencies in `Cargo.toml`:

```toml
[dependencies]
anyhow = "1.0"
tokio = { version = "1.42", features = ["rt-multi-thread", "macros"] }
serde = { version = "1.0", features = ["derive"] }
parking_lot = "0.12"
async-trait = "0.1"
```

Then use them in your code:

```rust
use anyhow::Result;
use tokio::sync::Mutex;
use serde::{Serialize, Deserialize};
```

From `Cargo.toml` — the dependency choices reflect ZeroClaw's values:

- `anyhow` — ergonomic error handling without boilerplate
- `tokio` — async runtime, with only the specific features enabled (size optimization)
- `parking_lot` — faster mutexes than std, do not poison on panic
- `async-trait` — enables `async fn` in trait definitions

### Feature Flags — Conditional Compilation

Features let you opt in to optional parts of a crate. From ZeroClaw's `Cargo.toml`:

```toml
[features]
default = []
sandbox-bubblewrap = ["dep:bubblewrap"]
sandbox-landlock = []
```

In code, you conditionally compile with `#[cfg(feature = "...")]`:

```rust
// src/security/mod.rs
#[cfg(feature = "sandbox-bubblewrap")]
pub mod bubblewrap;

#[cfg(target_os = "linux")]
pub mod firejail;
```

This means the bubblewrap sandbox code is only compiled when the feature is enabled. This keeps the default binary small.

---

## Part 9: Async Programming

### What Is Async/Await?

Async programming lets you write code that waits for slow operations (network requests, disk I/O) without blocking a thread. Instead of spawning a thread per request (expensive), you run many async tasks on a small thread pool.

**Compare to other approaches:**

| Approach         | Language  | How it works                           | Problem                       |
|------------------|-----------|----------------------------------------|-------------------------------|
| Threads          | C, Java   | OS thread per request                  | Expensive, limits concurrency |
| Callbacks        | JavaScript| Callback functions for completion      | Callback hell                 |
| async/await      | Rust, JS  | Suspend and resume tasks on a shared pool | Requires async runtime       |

In Rust, an `async fn` returns a `Future` — a description of computation that has not happened yet. The runtime (tokio) drives futures to completion.

### async fn and .await

```rust
// A regular function that blocks
fn fetch_sync(url: &str) -> String {
    // blocks the thread until the request completes
    reqwest::blocking::get(url).unwrap().text().unwrap()
}

// An async function — does not block the thread
async fn fetch_async(url: &str) -> anyhow::Result<String> {
    let response = reqwest::get(url).await?;   // .await suspends here
    let text = response.text().await?;          // .await suspends here too
    Ok(text)
}
```

The `.await` points are where the task can be suspended and other tasks can run. This is what makes async efficient.

### tokio — The Async Runtime

`async fn` in Rust does nothing on its own — it returns a `Future` that needs to be driven by a runtime. ZeroClaw uses tokio:

```rust
// src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // This is an async main function
    // tokio::main starts the tokio runtime, then runs this function
    let result = fetch_async("https://api.example.com/data").await?;
    println!("{result}");
    Ok(())
}
```

`#[tokio::main]` is a macro that wraps your `async fn main` in tokio runtime setup. Without it, you cannot use `.await` in `main`.

### Spawning Tasks

To run multiple async operations concurrently:

```rust
use tokio::task;

async fn run_parallel() -> anyhow::Result<()> {
    // Spawn two independent tasks
    let task1 = task::spawn(async {
        fetch_from_provider_a().await
    });
    let task2 = task::spawn(async {
        fetch_from_provider_b().await
    });

    // Wait for both
    let (result1, result2) = tokio::join!(task1, task2);
    println!("{:?} {:?}", result1?, result2?);
    Ok(())
}
```

From `src/main.rs` — spawning a blocking task on the thread pool:

```rust
let config = tokio::task::spawn_blocking(move || {
    // This runs on a dedicated blocking thread, not the async thread pool
    // Use spawn_blocking for CPU-intensive or blocking I/O operations
    onboard::run_wizard()
})
.await??;  // the double ?? unwraps both the JoinError and the inner Result
```

### async-trait — Async Functions in Traits

Rust does not natively support `async fn` in traits yet. The `async-trait` crate works around this:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait Tool: Send + Sync {
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
}

#[async_trait]
impl Tool for MyTool {
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        // ...
    }
}
```

The `#[async_trait]` macro transforms the `async fn` into one that returns `Pin<Box<dyn Future>>` — the underlying mechanism for dynamic dispatch with async. You do not need to understand this transformation; just know that both the trait definition and every implementation need `#[async_trait]`.

ZeroClaw uses `#[async_trait]` on `Tool`, `Provider`, `Channel`, `Memory`, and `Observer` traits — all extension points that may involve I/O.

### Real Async Code from ZeroClaw

From `src/providers/traits.rs` — an async trait with a default implementation that calls another async method:

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    async fn simple_chat(
        &self,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String> {
        // Default: delegate to chat_with_system
        self.chat_with_system(None, message, model, temperature).await
    }

    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String>;  // no default — implementors must provide this
}
```

---

## Part 10: Smart Pointers and Interior Mutability

### Box<T> — Heap Allocation

`Box<T>` allocates a value on the heap and gives you a unique (owned) pointer to it:

```rust
let on_stack: i32 = 5;
let on_heap: Box<i32> = Box::new(5);
println!("{}", *on_heap);  // dereference with *

// Common use: recursive types that cannot have a size known at compile time
enum List {
    Cons(i32, Box<List>),  // Box breaks the infinite size recursion
    Nil,
}
```

In ZeroClaw, `Box<dyn Tool>` is used to store trait objects:

```rust
let tools: Vec<Box<dyn Tool>> = vec![
    Box::new(ShellTool::new()),
    Box::new(FileReadTool::new()),
];
```

### Arc<T> — Shared Ownership Across Threads

`Arc` stands for Atomically Reference Counted. It is like `std::shared_ptr<T>` in C++ but thread-safe:

```rust
use std::sync::Arc;

let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);  // cheap: increments a counter

// Both `data` and `data_clone` point to the same Vec
// The Vec is freed when the last Arc is dropped
```

From `src/memory/sqlite.rs`:

```rust
pub struct SqliteMemory {
    conn: Mutex<Connection>,
    db_path: PathBuf,
    embedder: Arc<dyn EmbeddingProvider>,  // shared reference to the embedder
}

impl SqliteMemory {
    pub fn new(workspace_dir: &Path) -> anyhow::Result<Self> {
        Self::with_embedder(
            workspace_dir,
            Arc::new(super::embeddings::NoopEmbedding),  // wrap in Arc to share
            // ...
        )
    }
}
```

`Arc<dyn EmbeddingProvider>` means: a reference-counted pointer to anything that implements `EmbeddingProvider`. Multiple parts of the system can share the same embedder without copying it.

### Mutex<T> and parking_lot::Mutex — Thread-Safe Mutation

`Mutex<T>` provides interior mutability — it lets you modify data even through shared references, as long as only one thread accesses it at a time:

```rust
use parking_lot::Mutex;  // ZeroClaw uses parking_lot, not std

let data = Mutex::new(vec![1, 2, 3]);

// Lock the mutex to get access
{
    let mut guard = data.lock();
    guard.push(4);
}  // lock is released when guard goes out of scope
```

ZeroClaw uses `parking_lot::Mutex` instead of `std::sync::Mutex` because:
1. `parking_lot` mutexes are faster (use OS parking efficiently)
2. They do not **poison** on panic — `std::Mutex` becomes permanently unusable if a thread panics while holding the lock; `parking_lot::Mutex` does not

From `src/security/policy.rs`:

```rust
pub struct ActionTracker {
    actions: Mutex<Vec<Instant>>,  // parking_lot::Mutex
}

impl ActionTracker {
    pub fn record(&self) -> usize {
        let mut actions = self.actions.lock();
        // trim old actions
        let cutoff = Instant::now()
            .checked_sub(std::time::Duration::from_secs(3600))
            .unwrap_or_else(Instant::now);
        actions.retain(|t| *t > cutoff);
        actions.push(Instant::now());
        actions.len()
    }
}
```

Note that `record(&self)` takes `&self` (immutable reference), yet it modifies `actions`. This is **interior mutability** — the `Mutex` wraps the data and provides controlled mutation even through a shared reference.

### Arc<Mutex<T>> — The Standard Shared Mutable State Pattern

When you need to share mutable data across threads:

```rust
use std::sync::Arc;
use parking_lot::Mutex;

// Create shared mutable state
let state: Arc<Mutex<Vec<String>>> = Arc::new(Mutex::new(Vec::new()));

// Clone the Arc to share it with other threads
let state_clone = Arc::clone(&state);

tokio::spawn(async move {
    let mut v = state_clone.lock();
    v.push("hello from task".to_string());
});

// The original can still be used
{
    let mut v = state.lock();
    v.push("hello from main".to_string());
}
```

From `src/memory/sqlite.rs`:

```rust
pub struct SqliteMemory {
    conn: Mutex<Connection>,  // SQLite connection, protected by mutex
}
```

The SQLite connection is not thread-safe, so it is wrapped in a `Mutex`. Every read and write locks the mutex, does the operation, and releases it. This is safe because only one thread accesses the connection at a time.

---

## Part 11: Testing

### #[test] — Unit Tests

Tests in Rust live in the same file as the code they test, in a `mod tests` block:

```rust
// src/tools/traits.rs
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}

#[cfg(test)]
mod tests {
    use super::*;  // import everything from the parent module

    #[test]
    fn tool_result_serialization_roundtrip() {
        let result = ToolResult {
            success: false,
            output: String::new(),
            error: Some("boom".into()),
        };

        let json = serde_json::to_string(&result).unwrap();
        let parsed: ToolResult = serde_json::from_str(&json).unwrap();

        assert!(!parsed.success);
        assert_eq!(parsed.error.as_deref(), Some("boom"));
    }
}
```

`#[cfg(test)]` means the module is only compiled during `cargo test`, keeping it out of release binaries.

### #[tokio::test] — Async Tests

For testing async code, you need the tokio runtime:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn execute_returns_expected_output() {
        let tool = DummyTool;
        let result = tool
            .execute(serde_json::json!({ "value": "hello-tool" }))
            .await
            .unwrap();

        assert!(result.success);
        assert_eq!(result.output, "hello-tool");
        assert!(result.error.is_none());
    }
}
```

`#[tokio::test]` starts a tokio runtime for the duration of the test function.

### Assertions

```rust
assert!(condition);                    // panics if false
assert_eq!(left, right);              // panics if left != right
assert_ne!(left, right);              // panics if left == right
assert!(condition, "message {}", x);  // with format string

// From src/tools/traits.rs tests:
assert_eq!(spec.name, "dummy_tool");
assert_eq!(spec.description, "A deterministic test tool");
assert_eq!(spec.parameters["type"], "object");
assert!(result.success);
assert!(result.error.is_none());
```

### Running Tests

```bash
cargo test               # run all tests
cargo test --lib         # run only library tests (not integration tests)
cargo test my_test_name  # run tests whose name contains "my_test_name"
cargo test -- --nocapture  # show println! output during tests
cargo test -- --test-threads=1  # run tests single-threaded
```

### Test Organization

ZeroClaw uses **inline tests** (in the same file as the code) for unit tests. This makes it easy to test private helpers. From `src/security/secrets.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::TempDir;  // creates a temporary directory that auto-cleans

    #[test]
    fn encrypt_decrypt_roundtrip() {
        let tmp = TempDir::new().unwrap();
        let store = SecretStore::new(tmp.path(), true);
        let secret = "sk-my-secret-api-key-12345";

        let encrypted = store.encrypt(secret).unwrap();
        assert!(encrypted.starts_with("enc2:"), "Should have enc2: prefix");
        assert_ne!(encrypted, secret, "Should not be plaintext");

        let decrypted = store.decrypt(&encrypted).unwrap();
        assert_eq!(decrypted, secret, "Roundtrip must preserve original");
    }

    #[test]
    fn encrypting_same_value_produces_different_ciphertext() {
        let tmp = TempDir::new().unwrap();
        let store = SecretStore::new(tmp.path(), true);

        let e1 = store.encrypt("secret").unwrap();
        let e2 = store.encrypt("secret").unwrap();
        // Each encryption uses a random nonce, so ciphertexts differ
        assert_ne!(e1, e2);
        // But both decrypt to the same value
        assert_eq!(store.decrypt(&e1).unwrap(), "secret");
        assert_eq!(store.decrypt(&e2).unwrap(), "secret");
    }
}
```

Test naming in ZeroClaw follows the convention `<subject>_<expected_behavior>`:
- `encrypt_decrypt_roundtrip` — what it does and what we expect
- `tool_result_serialization_roundtrip` — subject is tool result serialization

---

## Part 12: Common Patterns in ZeroClaw

### The Trait + Factory Pattern

This is the central extensibility mechanism in ZeroClaw. You define a trait, implement it for each concrete backend, and a factory function returns the right implementation based on configuration.

```rust
// 1. Define the trait (src/providers/traits.rs)
#[async_trait]
pub trait Provider: Send + Sync {
    async fn chat_with_system(...) -> anyhow::Result<String>;
    // ...
}

// 2. Implement the trait for each backend (src/providers/openai.rs)
pub struct OpenAIProvider { /* ... */ }

#[async_trait]
impl Provider for OpenAIProvider {
    async fn chat_with_system(...) -> anyhow::Result<String> {
        // OpenAI-specific HTTP request
    }
}

// 3. Register in the factory (src/providers/mod.rs)
pub fn create_provider(name: &str, config: &Config)
    -> anyhow::Result<Arc<dyn Provider>>
{
    match name.trim().to_lowercase().as_str() {
        "openai" => Ok(Arc::new(OpenAIProvider::new(config)?)),
        "anthropic" => Ok(Arc::new(AnthropicProvider::new(config)?)),
        other => bail!("Unknown provider: {other}"),
    }
}

// 4. Use via the trait — caller does not know the concrete type
let provider: Arc<dyn Provider> = create_provider(&config.default_provider, &config)?;
let response = provider.chat_with_system(None, "Hello", "gpt-4o", 0.7).await?;
```

Adding a new provider means: implement the trait, add one line to the factory. No other code changes required.

### The serde Pattern — JSON Serialization

ZeroClaw uses `serde` for all JSON and TOML serialization. The derive macro generates all the boilerplate:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}

// Serialize to JSON
let result = ToolResult { success: true, output: "done".into(), error: None };
let json = serde_json::to_string(&result)?;
// {"success":true,"output":"done","error":null}

// Deserialize from JSON
let parsed: ToolResult = serde_json::from_str(&json)?;
```

**serde attributes** control the serialization behavior:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]  // serialize variant names as snake_case
pub enum MemoryCategory {
    Core,           // serializes as "core"
    Daily,          // serializes as "daily"
    Custom(String), // serializes as "custom"
}

#[derive(Serialize, Deserialize)]
pub struct Config {
    #[serde(skip)]  // do not serialize this field
    pub workspace_dir: PathBuf,

    #[serde(default)]  // use Default::default() if field is missing during deserialization
    pub memory: MemoryConfig,

    #[serde(skip_serializing_if = "Option::is_none")]  // omit if None
    pub api_key: Option<String>,
}
```

### JSON Schema for Tool Parameters

Tools declare their parameter schema as `serde_json::Value`. This is sent to the LLM so it knows how to call the tool:

```rust
fn parameters_schema(&self) -> serde_json::Value {
    serde_json::json!({
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "The shell command to execute"
            },
            "timeout_seconds": {
                "type": "integer",
                "description": "Maximum execution time",
                "default": 30
            }
        },
        "required": ["command"]
    })
}
```

The `serde_json::json!` macro creates JSON values inline using a JavaScript-like syntax. It is evaluated at runtime.

### Logging with tracing

ZeroClaw uses the `tracing` crate for structured logging:

```rust
use tracing::{info, warn, error, debug, trace};

// Structured log with fields
info!(provider = "openai", model = "gpt-4o", "Starting LLM request");

// Warning
warn!("Rate limit approaching: {} actions this hour", count);

// Error with context
error!(component = "gateway", "Failed to bind port {port}: {e}");
```

The log level is controlled by the `RUST_LOG` environment variable:

```bash
RUST_LOG=debug cargo run    # all debug and above
RUST_LOG=zeroclaw=trace     # trace-level logs for zeroclaw only
```

From `src/main.rs`:

```rust
let subscriber = fmt::Subscriber::builder()
    .with_env_filter(
        EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")),
    )
    .finish();
tracing::subscriber::set_global_default(subscriber)
    .expect("setting default subscriber failed");
```

### derive Macros — clap for CLI

ZeroClaw's CLI is built with the `clap` crate using derive macros:

```rust
use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(name = "zeroclaw", about = "The fastest, smallest AI assistant.")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Start the AI agent loop
    Agent {
        #[arg(short, long)]
        message: Option<String>,

        #[arg(short, long)]
        provider: Option<String>,

        #[arg(long, default_value = "0.7")]
        temperature: f64,
    },
    // ...
}

// Parse in main:
let cli = Cli::parse();
```

The doc comments (`///`) become help text in the CLI. This pattern eliminates hundreds of lines of argument parsing boilerplate.

---

## Part 13: Cheat Sheet

### Rust vs C/C++/Java Concept Map

| Concept                 | C               | C++                   | Java              | Rust                        |
|-------------------------|-----------------|------------------------|-------------------|-----------------------------|
| Memory management       | malloc/free     | new/delete, RAII       | Garbage collector | Ownership (compile-time)    |
| Null                    | NULL            | nullptr                | null              | `Option<T>` (no null)       |
| Error handling          | errno, -1       | exceptions             | exceptions        | `Result<T, E>`              |
| String (owned)          | `char*` + malloc| `std::string`          | `String`          | `String`                    |
| String (view)           | `const char*`   | `std::string_view`     | N/A               | `&str`                      |
| Dynamic array           | `arr[]` + realloc| `std::vector<T>`      | `ArrayList<T>`    | `Vec<T>`                    |
| Hash map                | (manual)        | `std::unordered_map`   | `HashMap<K,V>`    | `HashMap<K, V>`             |
| Shared pointer          | (manual)        | `std::shared_ptr<T>`   | (default refs)    | `Arc<T>`                    |
| Unique pointer          | (manual)        | `std::unique_ptr<T>`   | N/A               | `Box<T>` or owned value     |
| Thread-safe mutation    | pthread_mutex   | `std::mutex`           | `synchronized`    | `Mutex<T>`                  |
| Interface/abstract class| (function ptr)  | pure virtual class     | `interface`       | `trait`                     |
| Generic types           | (macros)        | `template<typename T>` | `<T>`             | `<T>` or `impl Trait`       |
| Dynamic dispatch        | (function ptr)  | virtual/vtable         | interface refs    | `dyn Trait`                 |
| Enum with data          | (union hack)    | `std::variant`         | sealed class      | enum with variants          |
| Package manager         | (none)          | (none/vcpkg)           | Maven/Gradle      | Cargo                       |
| Build system            | Make/CMake      | Make/CMake             | Maven/Gradle      | Cargo                       |
| Constants               | `#define`, `const`| `constexpr`           | `static final`    | `const`, `static`           |
| Immutable variable      | `const`         | `const`                | `final`           | `let` (default)             |
| Mutable variable        | (default)       | (default)              | (default)         | `let mut`                   |
| Async I/O               | callbacks/select| std::future (C++20)    | CompletableFuture | `async fn` + tokio          |

---

### Common Compiler Errors and What They Mean

**E0382: use of moved value**
```
error[E0382]: use of moved value: `s`
  --> src/main.rs:5:20
   |
3  |     let s2 = s;
   |              - value moved here
4  |     println!("{s}");
   |              ^^^ value used after move
```
Fix: either use `s2` instead of `s`, or clone: `let s2 = s.clone()`.

**E0502: cannot borrow as mutable because it is also borrowed as immutable**
```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```
Fix: drop the immutable reference before taking a mutable one. Restructure so the immutable borrow ends before `&mut` begins.

**E0515: cannot return reference to local data**
```
error[E0515]: cannot return reference to local variable `s`
```
Fix: return an owned `String` instead of a `&str`.

**E0308: mismatched types**
```
error[E0308]: mismatched types
  expected `String`, found `&str`
```
Fix: convert with `.to_string()` or `.into()`.

**E0277: the trait `X` is not implemented for `Y`**
```
error[E0277]: the trait `Send` is not implemented for `*mut u8`
```
Fix: the type cannot be sent across threads. Wrap it in `Arc<Mutex<>>` or use a type that is `Send`.

**E0499: cannot borrow `x` as mutable more than once**
```
error[E0499]: cannot borrow `v` as mutable more than once at a time
```
Fix: you have two `&mut` references. Restructure so only one exists at a time.

---

### Useful cargo Commands

```bash
# Build and run
cargo run                          # build + run in debug mode
cargo run --release                # build + run optimized
cargo run -- --help                # pass arguments to the program
cargo run -- agent --message "hi"

# Build only
cargo build                        # debug build
cargo build --release              # release build
cargo check                        # type-check only (fast)

# Testing
cargo test                         # run all tests
cargo test my_test                 # run tests matching "my_test"
cargo test -- --nocapture          # show stdout from tests
cargo test -- --test-threads=1     # sequential (helpful for debugging)

# Code quality
cargo fmt                          # format code
cargo fmt -- --check               # check formatting without modifying
cargo clippy                       # run linter
cargo clippy -- -D warnings        # fail on warnings (used in CI)

# Documentation
cargo doc --open                   # build and open docs in browser
cargo doc --no-deps                # skip dependency docs (faster)

# Dependency management
cargo add anyhow                   # add a dependency
cargo add tokio --features full    # add with features
cargo remove anyhow                # remove a dependency
cargo update                       # update dependencies within semver bounds
cargo tree                         # show dependency tree

# Benchmarking (requires nightly for built-in, or criterion crate)
cargo bench

# ZeroClaw validation sequence
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

---

### Quick Syntax Reference

```rust
// Variables
let x = 5;               // immutable
let mut y = 10;           // mutable
const MAX: u32 = 100;    // compile-time constant
static NAME: &str = "zeroclaw";  // static lifetime

// Types
let i: i32 = -5;
let u: u64 = 1_000_000;   // underscores for readability
let f: f64 = 3.14;
let b: bool = true;
let c: char = '🦀';
let s: &str = "literal";
let owned: String = "owned".to_string();

// Functions
fn add(x: i32, y: i32) -> i32 { x + y }
async fn fetch(url: &str) -> anyhow::Result<String> { todo!() }

// Structs
struct Point { x: f64, y: f64 }
impl Point {
    fn new(x: f64, y: f64) -> Self { Self { x, y } }
    fn distance(&self, other: &Point) -> f64 {
        ((self.x - other.x).powi(2) + (self.y - other.y).powi(2)).sqrt()
    }
}

// Enums
enum Status { Active, Inactive, Pending(String) }
match status {
    Status::Active => println!("active"),
    Status::Inactive => println!("inactive"),
    Status::Pending(msg) => println!("pending: {msg}"),
}

// Option
let maybe: Option<i32> = Some(42);
let nothing: Option<i32> = None;
let value = maybe.unwrap_or(0);
if let Some(v) = maybe { println!("{v}"); }

// Result
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 { Err("division by zero".into()) }
    else { Ok(a / b) }
}
let result = divide(10.0, 2.0)?;  // propagate error with ?

// Collections
let mut v: Vec<i32> = vec![1, 2, 3];
v.push(4);
let sum: i32 = v.iter().sum();
let doubled: Vec<i32> = v.iter().map(|&x| x * 2).collect();

// HashMap
use std::collections::HashMap;
let mut map: HashMap<String, i32> = HashMap::new();
map.insert("key".into(), 42);
let val = map.get("key");  // Option<&i32>

// Closures
let double = |x: i32| x * 2;
let add = |a, b| a + b;

// References
let r: &i32 = &x;           // immutable reference
let rm: &mut i32 = &mut y;  // mutable reference
*rm = 20;                    // dereference to write

// Traits
trait Greet { fn greet(&self) -> String; }
impl Greet for Point {
    fn greet(&self) -> String { format!("({}, {})", self.x, self.y) }
}
fn print_greeting(g: &impl Greet) { println!("{}", g.greet()); }
fn print_dynamic(g: &dyn Greet)   { println!("{}", g.greet()); }

// Generics
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

// Async
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let data = fetch("https://example.com").await?;
    println!("{data}");
    Ok(())
}

// Smart pointers
use std::sync::Arc;
use parking_lot::Mutex;
let shared = Arc::new(Mutex::new(vec![1, 2, 3]));
let clone = Arc::clone(&shared);
shared.lock().push(4);

// Tests
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }

    #[tokio::test]
    async fn async_test() {
        let result = fetch("http://localhost").await;
        assert!(result.is_err());  // expected: no server running
    }
}
```

---

### Where to Go Next

After working through this tutorial:

1. **Read the code**: Browse `src/tools/traits.rs`, `src/providers/traits.rs`, and `src/memory/traits.rs`. These define the core extension points.

2. **The Rust Book**: https://doc.rust-lang.org/book/ — the official, free, excellent textbook. Chapter 4 (ownership) is worth reading a second time.

3. **Rustlings**: https://github.com/rust-lang/rustlings — small exercises that drill the borrow checker until it becomes intuitive.

4. **rust-analyzer**: Install the rust-analyzer extension for your editor. It shows types inline, suggests fixes for borrow checker errors, and makes the learning curve much gentler.

5. **Try adding a tool**: The cleanest way to learn the codebase is to implement a toy `Tool` in `src/tools/` following the `DummyTool` pattern in `src/tools/traits.rs`. The trait tells you exactly what you need to implement.

6. **Read the error messages carefully**: Rust's compiler error messages are some of the best in any language. When you are stuck, read the full message — it usually explains what went wrong and suggests a fix.

---

*This tutorial covers Rust as used in ZeroClaw. For questions about project-specific conventions, see `CLAUDE.md`. For the full Rust specification, see https://doc.rust-lang.org/reference/.*
