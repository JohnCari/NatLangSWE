# The Power of 10 Rules for Safety-Critical Cargo + Rustup (Rust Toolchain) Configuration

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** Cargo is Rust's build system and package manager. Rustup is Rust's toolchain multiplexer, managing compiler versions, targets, and components. Together they form the complete Rust toolchain. These rules treat `Cargo.toml` as the single source of truth, `Cargo.lock` as the integrity contract, and `rust-toolchain.toml` as the compiler version pin. For Rust language-level rules, see the [Rust Power of 10](../../languages/rust/POWER_OF_10.md). For Axum framework patterns, see the [Axum Power of 10](../../frameworks/axum/POWER_OF_10.md).

---

## The Original 10 Rules (C Language)

| # | Rule |
|---|------|
| 1 | Restrict all code to very simple control flow constructs—no `goto`, `setjmp`, `longjmp`, or recursion |
| 2 | Give all loops a fixed upper bound provable by static analysis |
| 3 | Do not use dynamic memory allocation after initialization |
| 4 | No function longer than one printed page (~60 lines) |
| 5 | Assertion density: minimum 2 assertions per function |
| 6 | Declare all data objects at the smallest possible scope |
| 7 | Check all return values and validate all function parameters |
| 8 | Limit preprocessor to header includes and simple macros |
| 9 | Restrict pointers: single dereference only, no function pointers |
| 10 | Compile with all warnings enabled; use static analyzers daily |

---

## The Power of 10 Rules — Cargo + Rustup Edition

### Rule 1: Simple Control Flow — Single Toolchain, No Wrapper Scripts

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**Cargo + Rustup Adaptation:**

```bash
# BAD: Makefile wrapping cargo with fragile shell logic
build:
	@if [ "$(TARGET)" = "wasm" ]; then \
		rustup target add wasm32-unknown-unknown && \
		cargo build --target wasm32-unknown-unknown --release; \
	elif [ "$(TARGET)" = "musl" ]; then \
		rustup target add x86_64-unknown-linux-musl && \
		RUSTFLAGS="-C target-feature=+crt-static" cargo build --target x86_64-unknown-linux-musl --release; \
	else \
		cargo build --release; \
	fi

# BAD: Shell scripts orchestrating multiple cargo invocations
#!/bin/bash
cd crate-a && cargo build && cd ..
cd crate-b && cargo build && cd ..
cd crate-c && cargo build --features "crate-a/special" && cd ..
# Fragile, order-dependent, no parallelism

# BAD: Mixing cargo with manual rustc invocations
rustc src/main.rs -o output       # Bypasses Cargo entirely
cargo build                        # Different flags, different result

# GOOD: Cargo handles everything
cargo build --release
cargo test
cargo clippy
cargo doc

# GOOD: Workspace for multi-crate projects (Cargo handles ordering)
cargo build --workspace
cargo test --workspace
cargo clippy --workspace

# GOOD: Aliases for common compound commands
```

```toml
# .cargo/config.toml — aliases keep commands simple
[alias]
ci = "clippy --workspace --all-targets -- -D warnings"
ta = "test --workspace --all-targets"
ba = "build --workspace --all-targets"
```

```bash
# GOOD: Use aliases instead of Makefiles
cargo ci    # Runs workspace-wide clippy with deny warnings
cargo ta    # Runs all tests across workspace
```

**Guidelines:**
- Cargo is the build system — no Makefiles, shell scripts, or manual `rustc` calls for building
- Use Cargo workspaces for multi-crate projects — Cargo resolves build ordering automatically
- Use `.cargo/config.toml` aliases for frequently used compound commands
- No chaining `cd` between crates — use `--workspace` or `-p <package>` flags
- One command to build, one to test, one to lint: `cargo build`, `cargo test`, `cargo clippy`

---

### Rule 2: Fixed Bounds — Pin Toolchain, Pin Editions, Lock Dependencies

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**Cargo + Rustup Adaptation:**

```toml
# BAD: No toolchain pinning — different developers get different compilers
# (no rust-toolchain.toml exists)

# BAD: No edition specified — defaults to 2015
[package]
name = "myapp"
version = "0.1.0"
# edition is missing — inherits ancient default

# BAD: Cargo.lock not committed (for binaries/applications)
# .gitignore
Cargo.lock   # Produces non-reproducible builds for applications
```

```toml
# GOOD: Pin the toolchain version
# rust-toolchain.toml
[toolchain]
channel = "1.83.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
```

```toml
# GOOD: Explicit edition and Rust version
# Cargo.toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"
rust-version = "1.83.0"   # MSRV — minimum supported Rust version
```

```bash
# GOOD: Commit Cargo.lock for binaries and applications
git add Cargo.lock

# GOOD: Verify lockfile is in sync in CI
cargo generate-lockfile --check  # Fails if Cargo.lock is stale (nightly)
cargo update --locked            # Fails if Cargo.lock doesn't match Cargo.toml
```

**Guidelines:**
- Always create `rust-toolchain.toml` at the project root to pin the compiler version
- Include required components (`clippy`, `rustfmt`, `rust-analyzer`) in the toolchain file
- Always set `edition` explicitly in `Cargo.toml` — use the latest stable edition
- Set `rust-version` (MSRV) for libraries to communicate the minimum compiler requirement
- Commit `Cargo.lock` for binaries and applications (omit for libraries)
- Use `--locked` in CI to verify the lockfile matches `Cargo.toml`

---

### Rule 3: No Dynamic Allocation After Init — Deterministic Builds, No Network at Build Time

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**Cargo + Rustup Adaptation:**

```rust
// BAD: build.rs that downloads files at build time
// build.rs
fn main() {
    let response = reqwest::blocking::get("https://example.com/schema.json")
        .unwrap()
        .text()
        .unwrap();
    std::fs::write("src/generated_schema.rs", generate_code(&response)).unwrap();
    // Build fails when the network is down, produces different results each time
}

// BAD: build.rs that shells out to arbitrary commands
// build.rs
fn main() {
    let output = std::process::Command::new("python3")
        .arg("generate_bindings.py")
        .output()
        .expect("python3 must be installed");
    // Requires Python installed, version-dependent output
}
```

```dockerfile
# BAD: Dockerfile without locked dependencies
FROM rust:1.83
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/myapp"]

# GOOD: Deterministic Docker build with cargo-chef
FROM rust:1.83 AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release --locked

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/
CMD ["myapp"]
```

```bash
# BAD: Building without the lockfile
cargo build --release           # May resolve dependencies differently each time

# GOOD: Locked build — fails if Cargo.lock is out of sync
cargo build --release --locked

# GOOD: Offline build (no network access)
cargo build --release --locked --offline  # Uses only cached/vendored crates
```

```bash
# GOOD: Vendor dependencies for fully offline, reproducible builds
cargo vendor                    # Downloads all deps to ./vendor/
mkdir -p .cargo
echo '[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"' > .cargo/config.toml

cargo build --release --locked  # Uses vendored sources — no network needed
```

**Guidelines:**
- Use `--locked` for all CI and production builds — fails if `Cargo.lock` is stale
- No network fetches in `build.rs` — all build inputs must be local and deterministic
- No shelling out to external interpreters (Python, Node, etc.) from `build.rs`
- Use `cargo vendor` for fully offline, air-gapped builds
- Docker: use `cargo-chef` for dependency layer caching
- Use `--offline` in environments where network access should be prohibited

---

### Rule 4: Keep It Small — Lean Cargo.toml, Organized Workspaces, Minimal Features

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**Cargo + Rustup Adaptation:**

```toml
# BAD: Monolithic crate with 50+ dependencies and everything enabled
[package]
name = "myapp"
version = "0.1.0"

[dependencies]
tokio = { version = "1", features = ["full"] }       # "full" enables everything
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.12", features = ["json", "cookies", "gzip", "brotli", "deflate", "multipart", "stream"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "mysql", "sqlite", "json", "uuid", "chrono", "migrate"] }
# Every possible feature enabled — massive compile times, huge binary

# BAD: One giant crate with 50,000 lines
# src/
# ├── lib.rs    (2,000 lines)
# ├── models.rs (3,000 lines)
# └── api.rs    (5,000 lines)

# GOOD: Only enable features you actually use
[dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres"] }
```

```toml
# GOOD: Workspace for organized multi-crate projects
# Cargo.toml (workspace root)
[workspace]
members = [
    "crates/api",
    "crates/db",
    "crates/domain",
    "crates/cli",
]
resolver = "2"

# Shared dependency versions across workspace
[workspace.dependencies]
tokio = { version = "1.40", features = ["rt-multi-thread", "macros"] }
serde = { version = "1", features = ["derive"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres"] }
tracing = "0.1"
thiserror = "2"
anyhow = "1"
```

```toml
# crates/api/Cargo.toml — inherits from workspace
[package]
name = "myapp-api"
version.workspace = true
edition.workspace = true

[dependencies]
tokio.workspace = true
serde.workspace = true
tracing.workspace = true
myapp-domain = { path = "../domain" }
myapp-db = { path = "../db" }
```

```toml
# GOOD: Feature flags for optional functionality
[features]
default = []
postgres = ["sqlx/postgres"]
mysql = ["sqlx/mysql"]
telemetry = ["opentelemetry", "tracing-opentelemetry"]

[dependencies]
sqlx = { version = "0.8", optional = true }
opentelemetry = { version = "0.24", optional = true }
tracing-opentelemetry = { version = "0.25", optional = true }
```

**Guidelines:**
- Never use `features = ["full"]` — enable only the features you need
- Use Cargo workspaces for projects with more than 2-3 logical modules
- Use `[workspace.dependencies]` to centralize version management across workspace members
- Keep each crate focused on a single responsibility
- Use feature flags for optional integrations — don't compile what you don't use
- Prefer path dependencies (`{ path = "../core" }`) within workspaces over duplication

---

### Rule 5: Validate the Environment — Assert Toolchain, Lint Relentlessly, Test Everything

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**Cargo + Rustup Adaptation:**

```bash
# BAD: Running clippy/tests without ensuring the right toolchain
cargo test            # Which compiler version? Which components installed?
cargo clippy          # Might not be installed

# BAD: Ignoring clippy warnings
cargo clippy          # 47 warnings — "they're just warnings"

# GOOD: rust-toolchain.toml ensures correct toolchain automatically
# (rustup reads it and installs/switches to the pinned version)
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace --all-targets
```

```toml
# rust-toolchain.toml — the environment assertion
[toolchain]
channel = "1.83.0"
components = ["rustfmt", "clippy", "rust-analyzer"]
targets = ["x86_64-unknown-linux-gnu"]
```

```yaml
# GOOD: Full environment validation in CI (GitHub Actions)
name: CI
on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-D warnings"

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: "1.83.0"
          components: clippy, rustfmt

      - name: Cache cargo registry and build
        uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all --check

      - name: Clippy (deny all warnings)
        run: cargo clippy --workspace --all-targets -- -D warnings

      - name: Build
        run: cargo build --workspace --locked

      - name: Test
        run: cargo test --workspace --all-targets --locked

      - name: Doc tests
        run: cargo test --workspace --doc --locked
```

```rust
// GOOD: Assert invariants in debug builds
fn process_batch(items: &[Item]) -> Vec<Result> {
    debug_assert!(!items.is_empty(), "batch must not be empty");
    debug_assert!(items.len() <= MAX_BATCH_SIZE, "batch exceeds maximum size");

    items.iter().map(process_item).collect()
}
```

**Guidelines:**
- `rust-toolchain.toml` is mandatory — it ensures every developer and CI uses the same compiler
- Include `clippy`, `rustfmt`, and `rust-analyzer` as required components
- Set `RUSTFLAGS="-D warnings"` in CI to turn all warnings into errors
- Run `cargo fmt --check` before `cargo clippy` — formatting first, linting second
- Use `--workspace --all-targets` to cover all crates, tests, benches, and examples
- Run both `cargo test` (unit + integration) and `cargo test --doc` (doc tests) separately

---

### Rule 6: Minimal Scope — Workspace Isolation, Targeted Dependencies, No Global State

**Original Intent:** Reduce state complexity and potential for misuse.

**Cargo + Rustup Adaptation:**

```bash
# BAD: Installing tools globally with cargo install
cargo install cargo-watch
cargo install cargo-expand
cargo install sqlx-cli
# Different versions per developer, conflicts with other projects

# BAD: Global rustup overrides
rustup override set nightly       # Affects all projects in this directory tree
rustup default nightly            # Affects all projects on this machine

# GOOD: Project-scoped toolchain via rust-toolchain.toml
# (No global overrides needed — rustup reads the file automatically)

# GOOD: Use cargo-binstall or CI-specific installs for tools
cargo binstall cargo-watch        # Downloads pre-built binary (faster)
cargo binstall sqlx-cli
```

```toml
# BAD: Every crate depends on everything
# crates/cli/Cargo.toml
[dependencies]
myapp-api = { path = "../api" }
myapp-db = { path = "../db" }
myapp-domain = { path = "../domain" }
myapp-worker = { path = "../worker" }
myapp-migrations = { path = "../migrations" }
# CLI depends on the entire project — changes anywhere trigger recompilation

# GOOD: Minimal dependency graph — each crate imports only what it needs
# crates/cli/Cargo.toml
[dependencies]
myapp-domain = { path = "../domain" }  # Only domain types — no DB, no API

# crates/api/Cargo.toml
[dependencies]
myapp-domain = { path = "../domain" }
myapp-db = { path = "../db" }

# crates/db/Cargo.toml
[dependencies]
myapp-domain = { path = "../domain" }  # Domain types only
```

```toml
# GOOD: Dev dependencies scoped to the crate that needs them
[dev-dependencies]
tokio = { version = "1", features = ["test-util", "macros"] }
assert_matches = "1.5"
tempfile = "3.12"
# These are only compiled for tests — never in production
```

```toml
# GOOD: Target-specific dependencies
[target.'cfg(unix)'.dependencies]
nix = { version = "0.29", features = ["signal"] }

[target.'cfg(windows)'.dependencies]
windows = { version = "0.58", features = ["Win32_Foundation"] }
```

**Guidelines:**
- Use `rust-toolchain.toml` instead of `rustup override` or `rustup default`
- Minimize inter-crate dependencies in workspaces — not every crate needs every other crate
- Use `[dev-dependencies]` for test-only crates — they never enter the production binary
- Use `[target.'cfg(...)'.dependencies]` for platform-specific deps instead of conditional compilation in code
- Prefer `cargo binstall` over `cargo install` for pre-built tool binaries
- No global `cargo install` of project-specific tools — document them in CI and let developers install locally

---

### Rule 7: Check Return Values — Deny Warnings, Audit Dependencies, Fail on Any Issue

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**Cargo + Rustup Adaptation:**

```toml
# BAD: Allowing warnings to accumulate
# (no deny attributes, no RUSTFLAGS, warnings pile up)

# BAD: Allowing known-bad dependencies
# (no cargo-deny, no cargo-audit, vulnerable crates in production)
```

```rust
// BAD: Suppressing warnings instead of fixing them
#[allow(dead_code)]       // Why is this dead? Remove it or use it
#[allow(unused_imports)]  // Remove the import
#[allow(clippy::all)]     // Disabling all of clippy defeats the purpose
```

```toml
# GOOD: Crate-level lint configuration in Cargo.toml
[lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
todo = "deny"
dbg_macro = "deny"
```

```bash
# GOOD: Deny warnings everywhere
RUSTFLAGS="-D warnings" cargo build --workspace --locked
RUSTFLAGS="-D warnings" cargo clippy --workspace --all-targets

# GOOD: Audit dependencies for security vulnerabilities
cargo install cargo-audit
cargo audit

# GOOD: Policy enforcement with cargo-deny
cargo install cargo-deny
cargo deny check
```

```toml
# GOOD: cargo-deny configuration (deny.toml)
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-3.0",
]

[bans]
multiple-versions = "warn"
wildcards = "deny"          # No * version specifiers
highlight = "all"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
allow-git = []              # No git dependencies by default
```

```yaml
# GOOD: CI pipeline with strict verification
- name: Audit dependencies
  run: cargo audit

- name: Check dependency policy
  run: cargo deny check

- name: Build (deny warnings)
  run: cargo build --workspace --locked
  env:
    RUSTFLAGS: "-D warnings"
```

**Guidelines:**
- Set `RUSTFLAGS="-D warnings"` in CI — no warnings pass review
- Use `[lints]` in `Cargo.toml` for crate-level lint configuration (stable since Rust 1.74)
- `forbid(unsafe_code)` unless the crate explicitly requires unsafe
- `deny` clippy lints for `unwrap_used`, `expect_used`, `panic`, `todo`, and `dbg_macro`
- Run `cargo audit` for security vulnerability scanning
- Run `cargo deny check` for license compliance, duplicate detection, and source restriction
- Never use `#[allow(clippy::all)]` — fix the warnings or allow specific lints with justification

---

### Rule 8: Minimize Build Complexity — Simple build.rs, No Proc Macro Abuse, Standard Tooling

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**Cargo + Rustup Adaptation:**

```rust
// BAD: build.rs with hundreds of lines of logic
// build.rs
fn main() {
    // 200 lines: downloads protobuf compiler, generates bindings,
    // detects system libraries with pkg-config, runs code generation,
    // writes multiple files, sets 15 environment variables...
    let proto_path = download_protoc();
    let includes = find_system_headers();
    generate_ffi_bindings(&includes);
    compile_c_sources();
    link_system_libraries();
    set_feature_flags_from_environment();
}

// BAD: Custom proc macro for simple derive functionality
// my-derive/src/lib.rs (entire crate just to add one method)
#[proc_macro_derive(MyDisplay)]
pub fn my_display(input: TokenStream) -> TokenStream {
    // 80 lines that could be a manual impl or a 2-line derive(Display)
}
```

```rust
// GOOD: Minimal build.rs — only for things that genuinely require it
// build.rs
fn main() {
    // Compile protobufs (using a well-maintained crate)
    prost_build::compile_protos(&["proto/service.proto"], &["proto/"]).unwrap();
}

// GOOD: No build.rs at all — prefer compile-time solutions
// Use `include_str!` and `include_bytes!` for embedding static files
const SCHEMA: &str = include_str!("../schema.sql");
const LOGO: &[u8] = include_bytes!("../assets/logo.png");
```

```toml
# GOOD: Use well-maintained derive macros instead of custom proc macros
[dependencies]
serde = { version = "1", features = ["derive"] }
thiserror = "2"
strum = { version = "0.26", features = ["derive"] }

# GOOD: Use standard code generation tools
[build-dependencies]
prost-build = "0.13"       # Protobuf — well-maintained, standard
```

```rust
// GOOD: Use derive macros from established crates
use serde::{Deserialize, Serialize};
use thiserror::Error;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Config {
    host: String,
    port: u16,
}

#[derive(Debug, Error)]
enum AppError {
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("not found: {resource} with id {id}")]
    NotFound { resource: &'static str, id: String },
}
```

**Guidelines:**
- Avoid `build.rs` entirely when possible — prefer `include_str!`, `include_bytes!`, and const evaluation
- If `build.rs` is needed, keep it under 30 lines and delegate to well-maintained crates (`prost-build`, `bindgen`, `cc`)
- No network access, no shelling out to external tools from `build.rs`
- Use established derive macros (`serde`, `thiserror`, `strum`, `clap`) instead of writing custom proc macros
- Custom proc macros are justified only when they save significant boilerplate across many call sites
- Keep `[build-dependencies]` minimal — each one adds to compile time

---

### Rule 9: Typed Dependencies — No Wildcards, No Unpinned Git Sources, Explicit Registries

**Original Intent:** Restrict pointers: single dereference only, no function pointers.

**Cargo + Rustup Adaptation:**

```toml
# BAD: Wildcard and unpinned dependencies
[dependencies]
serde = "*"                                          # Any version
tokio = ">=1"                                        # No upper bound
my-lib = { git = "https://github.com/org/my-lib" }  # Unpinned — tracks HEAD
sketchy = { git = "https://random-git.example.com/sketchy" }  # Untrusted source

# BAD: Overly broad version ranges
[dependencies]
reqwest = ">=0.1"   # Matches 0.1 through any future version

# BAD: Path dependencies leaking outside the workspace
[dependencies]
utils = { path = "/home/user/other-project/utils" }  # Absolute path — breaks on other machines
```

```toml
# GOOD: Precise semver constraints
[dependencies]
tokio = "1.40"          # Equivalent to >=1.40.0, <2.0.0 (Cargo semver)
serde = "1.0"           # Equivalent to >=1.0.0, <2.0.0
axum = "0.8"            # Equivalent to >=0.8.0, <0.9.0 (pre-1.0 semver)
sqlx = "0.8"

# GOOD: Git dependencies pinned to specific revisions or tags
[dependencies]
my-lib = { git = "https://github.com/org/my-lib", tag = "v0.5.2" }
internal = { git = "https://github.com/org/internal", rev = "a1b2c3d" }

# GOOD: Path dependencies only within the workspace
[dependencies]
myapp-domain = { path = "../domain" }
myapp-db = { path = "../db" }
```

```toml
# GOOD: Workspace-level dependency pinning
[workspace.dependencies]
tokio = { version = "1.40", features = ["rt-multi-thread", "macros"] }
serde = { version = "1.0", features = ["derive"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres"] }

# GOOD: Explicit registry configuration (for private registries)
# .cargo/config.toml
[registries.my-company]
index = "sparse+https://cargo.my-company.com/index/"

# Cargo.toml
[dependencies]
internal-lib = { version = "0.3", registry = "my-company" }
```

```toml
# GOOD: cargo-deny blocks wildcards and unknown sources
# deny.toml
[bans]
wildcards = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

**Guidelines:**
- Every dependency must have a version constraint — no `*` or bare `>=`
- Cargo's semver interpretation: `"1.40"` means `>=1.40.0, <2.0.0` — this is already bounded
- Pin git dependencies to tags or commit SHAs — never track a branch
- Path dependencies must stay within the workspace — no absolute paths or cross-project references
- Configure private registries explicitly in `.cargo/config.toml`
- Use `cargo-deny` to enforce no wildcards and restrict allowed registries/git sources
- Prefer crates.io over git dependencies whenever possible

---

### Rule 10: Static Analysis — Clippy Pedantic, cargo-deny, Miri, Zero Warnings CI

**Original Intent:** Catch issues at development time; use every available tool.

**Cargo + Rustup Adaptation:**

```yaml
# GOOD: Complete CI pipeline (GitHub Actions)
name: Rust Safety Checks
on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-D warnings"

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: "1.83.0"
          components: clippy, rustfmt

      - name: Cache
        uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --all --check

      - name: Clippy (deny warnings)
        run: cargo clippy --workspace --all-targets -- -D warnings

      - name: Build (locked)
        run: cargo build --workspace --all-targets --locked

      - name: Unit and integration tests
        run: cargo test --workspace --all-targets --locked

      - name: Doc tests
        run: cargo test --workspace --doc --locked

      - name: Audit vulnerabilities
        run: |
          cargo install cargo-audit
          cargo audit

      - name: Dependency policy check
        run: |
          cargo install cargo-deny
          cargo deny check
```

```yaml
# GOOD: Additional safety jobs for critical projects
  miri:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
        with:
          components: miri
      - name: Run Miri (detects undefined behavior)
        run: cargo +nightly miri test

  semver:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check semver compliance
        uses: obi1kenobi/cargo-semver-checks-action@v2
```

```toml
# GOOD: Comprehensive lint configuration in workspace Cargo.toml
[workspace.lints.rust]
unsafe_code = "forbid"
missing_debug_implementations = "warn"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
# Deny dangerous patterns
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
todo = "deny"
dbg_macro = "deny"
unimplemented = "deny"
# Deny common mistakes
large_futures = "warn"
large_stack_arrays = "warn"
cast_possible_truncation = "warn"
```

```toml
# Per-crate opt-in to workspace lints
# crates/api/Cargo.toml
[lints]
workspace = true
```

```bash
# BAD: No analysis at all
cargo build   # "It compiles, ship it"

# BAD: Running clippy without deny
cargo clippy  # 200 warnings — "we'll fix them later"

# GOOD: The full local development check
cargo fmt --all --check
RUSTFLAGS="-D warnings" cargo clippy --workspace --all-targets
cargo test --workspace --all-targets --locked
cargo test --workspace --doc --locked
cargo audit
cargo deny check
```

**Guidelines:**
- `RUSTFLAGS="-D warnings"` is mandatory in CI — zero warnings policy
- Enable `clippy::pedantic` at workspace level — selectively `allow` specific lints with justification
- `forbid(unsafe_code)` by default — only `allow` it in specific modules that genuinely require it
- Use `[workspace.lints]` so all crates inherit the same lint configuration
- Run `cargo fmt --check` as the first CI step — fast and catches trivial issues
- Run `cargo audit` for CVE scanning on every PR
- Run `cargo deny check` for license compliance, ban detection, and source verification
- Use Miri (`cargo +nightly miri test`) for detecting undefined behavior in unsafe code
- Use `cargo-semver-checks` for libraries to prevent accidental breaking changes
- Automate dependency updates with Renovate or Dependabot

---

## Summary: Cargo + Rustup Adaptation

| # | Original Rule | Cargo + Rustup Guideline |
|---|---------------|--------------------------|
| 1 | No goto/recursion | Single toolchain, no wrapper scripts, Cargo aliases for compound commands |
| 2 | Fixed loop bounds | `rust-toolchain.toml`, explicit edition, commit `Cargo.lock`, `--locked` in CI |
| 3 | No dynamic allocation | Deterministic builds with `--locked`, no network in `build.rs`, `cargo vendor` |
| 4 | 60-line functions | Lean `Cargo.toml`, workspaces, `[workspace.dependencies]`, minimal features |
| 5 | 2+ assertions/function | `rust-toolchain.toml` pins env, `RUSTFLAGS="-D warnings"`, clippy + tests in CI |
| 6 | Minimize scope | Per-crate isolation, `[dev-dependencies]`, `cargo binstall`, no global overrides |
| 7 | Check returns | Deny warnings, `cargo audit`, `cargo deny`, forbid unsafe, no `#[allow(clippy::all)]` |
| 8 | Limit preprocessor | Minimal `build.rs`, established derive macros, no custom proc macros without justification |
| 9 | Restrict pointers | No wildcards, pinned git deps, explicit registries, `cargo-deny` source enforcement |
| 10 | All warnings enabled | Clippy pedantic, `cargo audit`, `cargo deny`, Miri, `cargo-semver-checks`, zero warnings |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Cargo Documentation](https://doc.rust-lang.org/cargo/) — The Rust Project
- [Rustup Documentation](https://rust-lang.github.io/rustup/) — The Rust Project
- [Cargo.toml Manifest Format](https://doc.rust-lang.org/cargo/reference/manifest.html) — The Rust Project
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/master/) — The Rust Project
- [cargo-deny](https://github.com/EmbarkStudios/cargo-deny) — Embark Studios
- [cargo-audit](https://github.com/RustSec/rustsec/tree/main/cargo-audit) — RustSec
- [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) — Predrag Gruevski
- [Miri: Undefined Behavior Detector](https://github.com/rust-lang/miri) — The Rust Project
- [cargo-chef (Docker Caching)](https://github.com/LukeMathWalker/cargo-chef) — Luca Palmieri
- [Workspace Lints (RFC 3389)](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-lints-table) — The Rust Project
