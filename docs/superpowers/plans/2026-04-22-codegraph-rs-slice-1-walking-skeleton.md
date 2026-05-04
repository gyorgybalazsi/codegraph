# codegraph-rs Slice 1 — Walking Skeleton

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the codegraph-rs Cargo project with a functioning CLI that can `init` a workspace (validate Neo4j, create the database, apply schema, write config files) and report `status`. No indexing, no parsing, no MCP. Just the skeleton through to the database.

**Architecture:** Single-crate Rust binary. `clap` for CLI, `serde + toml` for config, `neo4rs` 0.8 for Bolt. Async via `tokio`. Configuration discovery walks upward to find `.codegraph/config.toml` (except `init`, which creates it). Secrets resolved via env > file > error. Database name auto-derived from workspace name and validated per Neo4j 5 naming rules.

**Tech Stack:** Rust 1.91+, clap 4, serde 1, toml 1, tokio 1, neo4rs 0.8, anyhow, tracing.

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §1–5, §11–12. This slice covers config loading, the `db/` module, and the `init` + `status` CLI commands.

**Target directory:** `/Users/gyorgybalazsi/codegraph-rs` (sibling of the current TS `codegraph` repo). The directory does **not** exist yet — Task 1 creates it.

**Prerequisites:**
- Rust toolchain 1.91+ (`rustc --version` to verify)
- Neo4j Desktop running locally with a DBMS started, default port 7687, default user `neo4j` with a known password. The DBMS must be Enterprise Edition (Desktop installs this by default for local dev).

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml
  Cargo.lock
  README.md
  .gitignore
  rust-toolchain.toml
  src/
    main.rs                 # tiny: parse args, call lib::run
    lib.rs                  # re-exports + run() entry point
    cli/
      mod.rs                # Cli struct + Command enum
      init.rs               # init command implementation
      status.rs             # status command implementation
    config/
      mod.rs                # re-exports
      workspace.rs          # WorkspaceConfig types + load
      secrets.rs            # password resolution
      naming.rs             # Neo4j db-name validation + derivation
    db/
      mod.rs                # re-exports
      connection.rs         # Connection wrapper around neo4rs::Graph
      schema.rs             # apply_schema()
  tests/
    config_parsing.rs       # unit tests using fixtures
    fixtures/
      valid_workspace.toml
      missing_version.toml
      bad_db_name.toml
    integration_init.rs     # gated on CODEGRAPH_RS_TEST_NEO4J=1
    integration_status.rs   # gated on CODEGRAPH_RS_TEST_NEO4J=1
```

## Out of scope for this slice (covered in later slices)

- Extraction (no tree-sitter yet)
- Resolution
- Vector embeddings
- MCP server
- `index`, `sync`, `query`, `serve` subcommands (stubs only — they print "not yet implemented")
- Interactive `init` prompts (this slice uses flag-based `init`; interactive mode is a later slice)

---

### Task 1: Initialize Cargo project and git repository

**Files:**
- Create: `/Users/gyorgybalazsi/codegraph-rs/Cargo.toml`
- Create: `/Users/gyorgybalazsi/codegraph-rs/src/main.rs`
- Create: `/Users/gyorgybalazsi/codegraph-rs/.gitignore`
- Create: `/Users/gyorgybalazsi/codegraph-rs/rust-toolchain.toml`

- [ ] **Step 1: Verify the target directory does not exist**

```bash
test ! -e /Users/gyorgybalazsi/codegraph-rs && echo "OK: directory does not exist" || echo "STOP: directory already exists"
```

Expected: `OK: directory does not exist`. If the directory exists, stop and ask before proceeding — do not overwrite.

- [ ] **Step 2: Create the Cargo project**

```bash
cd /Users/gyorgybalazsi
cargo new --bin codegraph-rs
cd codegraph-rs
ls
```

Expected output: `Cargo.toml  src` (Cargo also runs `git init` automatically).

- [ ] **Step 3: Pin the Rust toolchain**

Create `/Users/gyorgybalazsi/codegraph-rs/rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.91"
components = ["rustfmt", "clippy"]
```

This locks all contributors and CI to a known-good Rust version.

- [ ] **Step 4: Replace Cargo.toml with the slice 1 manifest**

Overwrite `/Users/gyorgybalazsi/codegraph-rs/Cargo.toml` with:

```toml
[package]
name = "codegraph-rs"
version = "0.1.0"
edition = "2021"
rust-version = "1.91"
description = "Local-first code intelligence — Rust port of CodeGraph, backed by Neo4j"
license = "MIT"
repository = "https://github.com/bitsafe/codegraph-rs"

[[bin]]
name = "codegraph-rs"
path = "src/main.rs"

[dependencies]
anyhow = "1"
thiserror = "2"
clap = { version = "4", features = ["derive", "env"] }
serde = { version = "1", features = ["derive"] }
toml = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "io-util", "io-std", "fs", "process", "signal"] }
neo4rs = "0.8"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
tempfile = "3"
```

- [ ] **Step 5: Replace .gitignore**

Overwrite `/Users/gyorgybalazsi/codegraph-rs/.gitignore`:

```
/target
/Cargo.lock.bak
.codegraph/secrets.toml

# Editor / OS
.DS_Store
.vscode/
.idea/
*.swp
```

`Cargo.lock` is **not** gitignored — for a binary crate it should be checked in.

- [ ] **Step 6: Verify cargo builds the empty project**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo build 2>&1 | tail -5
```

Expected to end with: `Finished \`dev\` profile [optimized + debuginfo] target(s)` (compile time may be substantial because of dependencies).

- [ ] **Step 7: Commit**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
git add Cargo.toml Cargo.lock .gitignore rust-toolchain.toml src/
git commit -m "chore: initialize codegraph-rs Cargo project"
```

---

### Task 2: Module skeleton with empty modules

**Files:**
- Modify: `src/main.rs`
- Create: `src/lib.rs`
- Create: `src/cli/mod.rs`
- Create: `src/config/mod.rs`
- Create: `src/db/mod.rs`

- [ ] **Step 1: Replace src/main.rs with a thin entry point**

Overwrite `src/main.rs`:

```rust
fn main() -> anyhow::Result<()> {
    codegraph_rs::run()
}
```

- [ ] **Step 2: Create src/lib.rs**

Create `src/lib.rs`:

```rust
//! codegraph-rs library entry point.
//!
//! This crate exposes its modules so integration tests can exercise them
//! without spawning the binary. The binary in `src/main.rs` is a thin shim.

pub mod cli;
pub mod config;
pub mod db;

/// Application entry point. Initializes logging, parses CLI args, dispatches.
pub fn run() -> anyhow::Result<()> {
    init_tracing();
    cli::dispatch()
}

fn init_tracing() {
    use tracing_subscriber::EnvFilter;
    let _ = tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::try_from_env("CODEGRAPH_RS_LOG").unwrap_or_else(|_| EnvFilter::new("info")))
        .with_target(false)
        .try_init();
}
```

- [ ] **Step 3: Create empty module files**

Create `src/cli/mod.rs`:

```rust
//! CLI subcommand dispatch.

use anyhow::Result;

pub fn dispatch() -> Result<()> {
    anyhow::bail!("CLI dispatch not implemented yet");
}
```

Create `src/config/mod.rs`:

```rust
//! Workspace configuration loading and validation.
```

Create `src/db/mod.rs`:

```rust
//! Neo4j Bolt client and schema management.
```

- [ ] **Step 4: Build to verify the skeleton compiles**

```bash
cargo build 2>&1 | tail -3
```

Expected: builds clean with no errors. Warnings about unused modules are fine for now.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "chore: add module skeleton (cli, config, db)"
```

---

### Task 3: CLI scaffolding with clap

**Files:**
- Modify: `src/cli/mod.rs`
- Create: `src/cli/init.rs`
- Create: `src/cli/status.rs`
- Test: integration test will run as `cargo test --test cli_smoke`

- [ ] **Step 1: Replace src/cli/mod.rs with the full clap definition**

Overwrite `src/cli/mod.rs`:

```rust
//! CLI subcommand dispatch.

use anyhow::Result;
use clap::{Parser, Subcommand};
use std::path::PathBuf;

pub mod init;
pub mod status;

/// codegraph-rs: local-first code intelligence backed by Neo4j.
#[derive(Debug, Parser)]
#[command(name = "codegraph-rs", version, about, long_about = None)]
pub struct Cli {
    /// Path to the workspace directory (the directory containing `.codegraph/`).
    /// Defaults to the current working directory; commands other than `init`
    /// also walk upward to find the nearest `.codegraph/`.
    #[arg(long, global = true)]
    pub workspace: Option<PathBuf>,

    #[command(subcommand)]
    pub command: Command,
}

#[derive(Debug, Subcommand)]
pub enum Command {
    /// Scaffold a new workspace: validate Neo4j, create the database, write config files.
    Init(init::InitArgs),

    /// Run a full index of the workspace. Stub in slice 1.
    Index,

    /// Run an incremental sync. Stub in slice 1.
    Sync,

    /// Print workspace statistics (file count, symbol count, last sync time).
    Status,

    /// Search for a symbol by name. Stub in slice 1.
    Query {
        /// The query string.
        query: String,
    },

    /// Run the MCP stdio server for use by Claude Code. Stub in slice 1.
    Serve,
}

pub fn dispatch() -> Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Command::Init(args) => init::run(args, cli.workspace.as_deref()),
        Command::Status => status::run(cli.workspace.as_deref()),
        Command::Index | Command::Sync | Command::Query { .. } | Command::Serve => {
            anyhow::bail!("not yet implemented in slice 1");
        }
    }
}
```

- [ ] **Step 2: Create src/cli/init.rs as a stub**

Create `src/cli/init.rs`:

```rust
//! `codegraph-rs init` — scaffold a workspace.

use anyhow::Result;
use clap::Args;
use std::path::Path;

#[derive(Debug, Args)]
pub struct InitArgs {
    /// Workspace name (used to derive the Neo4j database name).
    #[arg(long)]
    pub name: String,

    /// Neo4j Bolt URI.
    #[arg(long, default_value = "bolt://localhost:7687")]
    pub neo4j_uri: String,

    /// Neo4j user.
    #[arg(long, default_value = "neo4j")]
    pub neo4j_user: String,

    /// Neo4j password. Prefer the CODEGRAPH_RS_NEO4J_PASSWORD env var for non-interactive use.
    #[arg(long, env = "CODEGRAPH_RS_NEO4J_PASSWORD")]
    pub neo4j_password: String,

    /// Workspace roots in the form `name=path`. Repeat for multiple roots.
    /// Example: `--root splice=../splice --root utility=../utility`.
    #[arg(long, value_name = "NAME=PATH", required = true, num_args = 1..)]
    pub root: Vec<String>,
}

pub fn run(_args: InitArgs, _workspace: Option<&Path>) -> Result<()> {
    anyhow::bail!("init: not yet implemented (filled in by Task 10)");
}
```

- [ ] **Step 3: Create src/cli/status.rs as a stub**

Create `src/cli/status.rs`:

```rust
//! `codegraph-rs status` — print workspace stats.

use anyhow::Result;
use std::path::Path;

pub fn run(_workspace: Option<&Path>) -> Result<()> {
    anyhow::bail!("status: not yet implemented (filled in by Task 11)");
}
```

- [ ] **Step 4: Build and verify the CLI parses**

```bash
cargo run --quiet -- --help 2>&1 | head -25
```

Expected: shows usage with subcommands `init`, `index`, `sync`, `status`, `query`, `serve`.

- [ ] **Step 5: Verify a subcommand stub fails predictably**

```bash
cargo run --quiet -- index 2>&1 | tail -3
```

Expected: prints an error containing `not yet implemented in slice 1` and exits nonzero.

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat(cli): add clap-based CLI with init/status/index/sync/query/serve subcommands"
```

---

### Task 4: Workspace config types and TOML parsing (TDD)

**Files:**
- Create: `src/config/workspace.rs`
- Modify: `src/config/mod.rs`
- Create: `tests/fixtures/valid_workspace.toml`
- Create: `tests/fixtures/missing_version.toml`
- Create: `tests/fixtures/bad_db_name.toml`
- Test: `tests/config_parsing.rs`

- [ ] **Step 1: Create the test fixtures**

Create `tests/fixtures/valid_workspace.toml`:

```toml
schema_version = 1

[workspace]
name = "daml-stack"

[workspace.neo4j]
uri = "bolt://localhost:7687"
database = "codegraph-daml-stack"
username = "neo4j"

[[workspace.roots]]
name = "splice"
path = "../splice"

[[workspace.roots]]
name = "utility"
path = "../utility"

[indexing]
languages = ["rust", "daml"]

[embeddings]
enabled = true
model = "bge-small-en-v1.5"
```

Create `tests/fixtures/missing_version.toml`:

```toml
[workspace]
name = "x"

[workspace.neo4j]
uri = "bolt://localhost:7687"
database = "codegraph-x"
username = "neo4j"

[[workspace.roots]]
name = "x"
path = "."
```

Create `tests/fixtures/bad_db_name.toml`:

```toml
schema_version = 1

[workspace]
name = "x"

[workspace.neo4j]
uri = "bolt://localhost:7687"
database = "codegraph_x"
username = "neo4j"

[[workspace.roots]]
name = "x"
path = "."
```

(`codegraph_x` has an underscore — invalid per Neo4j 5 rules.)

- [ ] **Step 2: Write the failing test**

Create `tests/config_parsing.rs`:

```rust
//! Unit tests for TOML config parsing.

use codegraph_rs::config::workspace::{WorkspaceConfig, parse};

const VALID: &str = include_str!("fixtures/valid_workspace.toml");
const MISSING_VERSION: &str = include_str!("fixtures/missing_version.toml");
const BAD_DB_NAME: &str = include_str!("fixtures/bad_db_name.toml");

#[test]
fn parses_valid_workspace() {
    let cfg: WorkspaceConfig = parse(VALID).expect("valid TOML must parse");
    assert_eq!(cfg.schema_version, 1);
    assert_eq!(cfg.workspace.name, "daml-stack");
    assert_eq!(cfg.workspace.neo4j.uri, "bolt://localhost:7687");
    assert_eq!(cfg.workspace.neo4j.database, "codegraph-daml-stack");
    assert_eq!(cfg.workspace.roots.len(), 2);
    assert_eq!(cfg.workspace.roots[0].name, "splice");
    assert_eq!(cfg.indexing.languages, vec!["rust".to_string(), "daml".to_string()]);
    assert!(cfg.embeddings.enabled);
}

#[test]
fn missing_schema_version_errors() {
    let err = parse(MISSING_VERSION).expect_err("missing schema_version must error");
    let msg = format!("{err:#}");
    assert!(msg.contains("schema_version"), "error should mention schema_version, got: {msg}");
}

#[test]
fn bad_database_name_errors() {
    let err = parse(BAD_DB_NAME).expect_err("underscore in db name must error");
    let msg = format!("{err:#}");
    assert!(msg.contains("database"), "error should mention database, got: {msg}");
}
```

- [ ] **Step 3: Run the tests — expect compile failures (modules don't exist yet)**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: compile errors about `parse` and `WorkspaceConfig` being missing from `codegraph_rs::config::workspace`.

> **Note on intermediate broken builds:** The project will not build between steps 4 and 6 of this task — `workspace.rs` imports `crate::config::naming` which is created in step 6. This is intentional. Don't run `cargo build` until step 7.

- [ ] **Step 4: Implement the config types and parser**

Create `src/config/workspace.rs`:

```rust
//! Workspace config types deserialized from `.codegraph/config.toml`.

use serde::Deserialize;
use std::path::PathBuf;

use crate::config::naming::validate_database_name;

#[derive(Debug, Deserialize)]
pub struct WorkspaceConfig {
    pub schema_version: u32,
    pub workspace: Workspace,
    #[serde(default)]
    pub indexing: Indexing,
    #[serde(default)]
    pub embeddings: Embeddings,
}

#[derive(Debug, Deserialize)]
pub struct Workspace {
    pub name: String,
    pub neo4j: Neo4j,
    pub roots: Vec<Root>,
}

#[derive(Debug, Deserialize)]
pub struct Neo4j {
    pub uri: String,
    pub database: String,
    pub username: String,
}

#[derive(Debug, Deserialize)]
pub struct Root {
    pub name: String,
    pub path: PathBuf,
}

#[derive(Debug, Deserialize)]
pub struct Indexing {
    #[serde(default = "default_languages")]
    pub languages: Vec<String>,
}

impl Default for Indexing {
    fn default() -> Self {
        Self { languages: default_languages() }
    }
}

fn default_languages() -> Vec<String> {
    vec!["rust".to_string(), "daml".to_string()]
}

#[derive(Debug, Deserialize)]
pub struct Embeddings {
    #[serde(default = "yes")]
    pub enabled: bool,
    #[serde(default = "default_model")]
    pub model: String,
}

impl Default for Embeddings {
    fn default() -> Self {
        Self { enabled: true, model: default_model() }
    }
}

fn yes() -> bool { true }
fn default_model() -> String { "bge-small-en-v1.5".to_string() }

const SUPPORTED_SCHEMA_VERSION: u32 = 1;

/// Parse a TOML string into a validated `WorkspaceConfig`.
pub fn parse(input: &str) -> anyhow::Result<WorkspaceConfig> {
    // First pass: surface a clear error if `schema_version` is missing,
    // since serde's default error for a missing top-level key is awkward.
    if !input.contains("schema_version") {
        anyhow::bail!("config is missing required key `schema_version`");
    }

    let cfg: WorkspaceConfig = toml::from_str(input)
        .map_err(|e| anyhow::anyhow!("failed to parse workspace config: {e}"))?;

    if cfg.schema_version != SUPPORTED_SCHEMA_VERSION {
        anyhow::bail!(
            "unsupported schema_version {} (this build supports {})",
            cfg.schema_version,
            SUPPORTED_SCHEMA_VERSION
        );
    }

    if cfg.workspace.roots.is_empty() {
        anyhow::bail!("workspace must declare at least one root in [[workspace.roots]]");
    }

    validate_database_name(&cfg.workspace.neo4j.database)
        .map_err(|e| anyhow::anyhow!("invalid Neo4j database name `{}`: {e}", cfg.workspace.neo4j.database))?;

    if cfg.embeddings.enabled && cfg.embeddings.model != "bge-small-en-v1.5" {
        anyhow::bail!(
            "embedding model `{}` not supported in v1; only `bge-small-en-v1.5` is accepted",
            cfg.embeddings.model
        );
    }

    Ok(cfg)
}
```

- [ ] **Step 5: Re-export from src/config/mod.rs**

Overwrite `src/config/mod.rs`:

```rust
//! Workspace configuration loading and validation.

pub mod naming;
pub mod secrets;
pub mod workspace;

pub use workspace::{WorkspaceConfig, parse};
```

(`naming` and `secrets` modules are added in Tasks 6 and 7. They are referenced here to avoid module-graph churn later.)

- [ ] **Step 6: Create stub for naming module so the build compiles**

Create `src/config/naming.rs`:

```rust
//! Neo4j database name validation. Filled in by Task 6.

pub fn validate_database_name(_name: &str) -> Result<(), String> {
    // Slice 1 / Task 6: real validation. For now accept everything except
    // the explicit underscore case used in test fixtures.
    if _name.contains('_') {
        return Err("underscores are not permitted in Neo4j 5 database names".to_string());
    }
    Ok(())
}
```

Create `src/config/secrets.rs`:

```rust
//! Neo4j password resolution. Filled in by Task 7.
```

- [ ] **Step 7: Run the tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: 3 tests pass. If the `bad_database_name_errors` test fails, double-check that the fixture has the underscore.

- [ ] **Step 8: Commit**

```bash
git add src/config tests/fixtures tests/config_parsing.rs
git commit -m "feat(config): add WorkspaceConfig types and TOML parser with validation"
```

---

### Task 5: Config loading with upward discovery

**Files:**
- Modify: `src/config/workspace.rs`
- Modify: `src/config/mod.rs` (add re-export)
- Test: extend `tests/config_parsing.rs`

- [ ] **Step 1: Write the failing tests for path discovery and loading**

Append to `tests/config_parsing.rs`:

```rust
use codegraph_rs::config::workspace::{find_workspace_root, load_from_path};
use std::fs;
use tempfile::tempdir;

#[test]
fn finds_workspace_in_current_dir() {
    let dir = tempdir().unwrap();
    fs::create_dir(dir.path().join(".codegraph")).unwrap();
    fs::write(dir.path().join(".codegraph/config.toml"), VALID).unwrap();

    let found = find_workspace_root(dir.path()).expect("should find .codegraph");
    assert_eq!(found.canonicalize().unwrap(), dir.path().canonicalize().unwrap());
}

#[test]
fn finds_workspace_in_ancestor() {
    let dir = tempdir().unwrap();
    fs::create_dir(dir.path().join(".codegraph")).unwrap();
    fs::write(dir.path().join(".codegraph/config.toml"), VALID).unwrap();

    let nested = dir.path().join("a/b/c");
    fs::create_dir_all(&nested).unwrap();

    let found = find_workspace_root(&nested).expect("should walk up to find .codegraph");
    assert_eq!(found.canonicalize().unwrap(), dir.path().canonicalize().unwrap());
}

#[test]
fn missing_workspace_errors() {
    let dir = tempdir().unwrap();
    let err = find_workspace_root(dir.path()).expect_err("must error when no .codegraph anywhere");
    assert!(format!("{err:#}").contains(".codegraph"));
}

#[test]
fn load_from_path_canonicalizes_root_paths() {
    let dir = tempdir().unwrap();
    fs::create_dir(dir.path().join(".codegraph")).unwrap();

    // Create the root directories the fixture references
    fs::create_dir(dir.path().join("splice")).unwrap();
    fs::create_dir(dir.path().join("utility")).unwrap();

    // Adjust the fixture so root paths exist relative to dir/
    let fixture = VALID
        .replace("../splice", "splice")
        .replace("../utility", "utility");
    fs::write(dir.path().join(".codegraph/config.toml"), &fixture).unwrap();

    let cfg = load_from_path(dir.path()).expect("load should succeed");
    let splice = &cfg.workspace.roots[0];
    assert!(splice.path.is_absolute(), "paths must be canonicalized to absolute");
}
```

- [ ] **Step 2: Run the new tests — expect compile failure**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: errors about `find_workspace_root` and `load_from_path` not being defined.

- [ ] **Step 3: Implement the discovery and loading helpers**

> **Note:** "Append" here means *add to the file*. The new `use std::path::Path;` line should go up at the top with the other `use` statements (replacing `use std::path::PathBuf;` with `use std::path::{Path, PathBuf};`); the new functions go below the existing code.

Append to `src/config/workspace.rs`:

```rust
use std::path::Path;

const CONFIG_DIRNAME: &str = ".codegraph";
const CONFIG_FILENAME: &str = "config.toml";

/// Walk upward from `start` looking for the nearest directory containing
/// `.codegraph/config.toml`. Returns the directory containing `.codegraph/`.
pub fn find_workspace_root(start: &Path) -> anyhow::Result<PathBuf> {
    let mut current: PathBuf = start.canonicalize().map_err(|e| {
        anyhow::anyhow!("cannot canonicalize starting path `{}`: {e}", start.display())
    })?;

    loop {
        let candidate = current.join(CONFIG_DIRNAME).join(CONFIG_FILENAME);
        if candidate.is_file() {
            return Ok(current);
        }
        let Some(parent) = current.parent() else {
            anyhow::bail!(
                "no .codegraph/{CONFIG_FILENAME} found in `{}` or any ancestor",
                start.display()
            );
        };
        current = parent.to_path_buf();
    }
}

/// Read and parse the workspace config at `<workspace_dir>/.codegraph/config.toml`.
/// Canonicalizes all root paths to absolute paths relative to `workspace_dir`.
pub fn load_from_path(workspace_dir: &Path) -> anyhow::Result<WorkspaceConfig> {
    let config_path = workspace_dir.join(CONFIG_DIRNAME).join(CONFIG_FILENAME);
    let raw = std::fs::read_to_string(&config_path).map_err(|e| {
        anyhow::anyhow!("cannot read `{}`: {e}", config_path.display())
    })?;
    let mut cfg = parse(&raw)?;

    // Canonicalize root paths relative to the config file's directory.
    let base = workspace_dir.canonicalize().map_err(|e| {
        anyhow::anyhow!("cannot canonicalize `{}`: {e}", workspace_dir.display())
    })?;
    for root in &mut cfg.workspace.roots {
        let absolute = if root.path.is_absolute() {
            root.path.clone()
        } else {
            base.join(&root.path)
        };
        root.path = absolute.canonicalize().map_err(|e| {
            anyhow::anyhow!(
                "root `{}` points at non-existent path `{}`: {e}",
                root.name,
                absolute.display()
            )
        })?;
    }

    Ok(cfg)
}
```

- [ ] **Step 4: Run the tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: 7 tests pass total.

- [ ] **Step 5: Commit**

```bash
git add src/config tests/
git commit -m "feat(config): add upward .codegraph/ discovery and absolute-path loader"
```

---

### Task 6: Database name validation per Neo4j 5 rules

**Files:**
- Modify: `src/config/naming.rs`
- Test: append to `tests/config_parsing.rs`

- [ ] **Step 1: Write the failing tests**

Append to `tests/config_parsing.rs`:

```rust
use codegraph_rs::config::naming::{validate_database_name, derive_database_name};

#[test]
fn db_name_valid_examples() {
    for name in ["codegraph-x", "codegraph-daml-stack", "abc", "neo4j", "a1.b2", "abc-def-12.3"] {
        validate_database_name(name).unwrap_or_else(|e| panic!("`{name}` should be valid: {e}"));
    }
}

#[test]
fn db_name_invalid_examples() {
    let bad = [
        "ab",                       // too short (<3)
        "1abc",                     // starts with digit
        "_abc",                     // starts with underscore
        "ABC",                      // uppercase: codegraph-rs convention is lowercase only
        "codegraph_x",              // underscore
        "codegraph x",              // space
        "codegraph!",               // special char
        &"a".repeat(64),            // > 63
    ];
    for name in bad {
        assert!(
            validate_database_name(name).is_err(),
            "`{name}` should be invalid"
        );
    }
}

#[test]
fn derives_db_name_from_workspace() {
    assert_eq!(derive_database_name("daml-stack"), "codegraph-daml-stack");
    assert_eq!(derive_database_name("My Project"), "codegraph-my-project");
    assert_eq!(derive_database_name("codegraph-rs"), "codegraph-codegraph-rs");
    assert_eq!(derive_database_name("foo_bar"), "codegraph-foo-bar");
}
```

Note: Neo4j 5 rules allow ASCII letters (case-insensitive in practice — but the docs require starting with a letter and using letters, digits, dots, dashes). Uppercase is not strictly forbidden by the language grammar but is conventionally lowercased. We accept lowercase only via `derive_database_name` and reject uppercase in user-supplied names to keep things simple. The `"ABC"` case in the invalid set tests this rule. Adjust the test if you decide differently — but be consistent.

- [ ] **Step 2: Run the failing tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: tests fail or panic (the existing stub accepts everything without underscores).

- [ ] **Step 3: Implement the real validator + deriver**

Overwrite `src/config/naming.rs`:

```rust
//! Neo4j database name rules and derivation.
//!
//! Neo4j 5 rules:
//! - 3..=63 characters
//! - First character: ASCII letter (a-z or A-Z; we restrict to lowercase)
//! - Remaining characters: ASCII letters (lowercase), digits, '.', '-'
//! - No spaces, no underscores, no other special characters.

const MIN_LEN: usize = 3;
const MAX_LEN: usize = 63;

/// Validate a database name against Neo4j 5 rules. Returns Ok on success.
pub fn validate_database_name(name: &str) -> Result<(), String> {
    let len = name.len();
    if len < MIN_LEN {
        return Err(format!("name must be at least {MIN_LEN} characters (got {len})"));
    }
    if len > MAX_LEN {
        return Err(format!("name must be at most {MAX_LEN} characters (got {len})"));
    }

    let mut chars = name.chars();
    let first = chars.next().expect("len checked above");
    if !first.is_ascii_lowercase() {
        return Err(format!(
            "first character must be an ASCII lowercase letter (got `{first}`)"
        ));
    }

    for (i, c) in chars.enumerate() {
        if !(c.is_ascii_lowercase() || c.is_ascii_digit() || c == '.' || c == '-') {
            return Err(format!(
                "character `{c}` at position {} not allowed (only lowercase letters, digits, '.', '-')",
                i + 1
            ));
        }
    }

    Ok(())
}

/// Derive a Neo4j-compatible database name from a workspace name.
/// Lowercases and replaces any disallowed character with '-', then prefixes
/// with `codegraph-`. Collapses runs of '-' and trims leading/trailing '-'.
pub fn derive_database_name(workspace_name: &str) -> String {
    let mut sanitized = String::with_capacity(workspace_name.len());
    for c in workspace_name.chars() {
        let mapped = c.to_ascii_lowercase();
        if mapped.is_ascii_lowercase() || mapped.is_ascii_digit() || mapped == '.' {
            sanitized.push(mapped);
        } else {
            sanitized.push('-');
        }
    }
    // Collapse multiple consecutive dashes
    let mut collapsed = String::with_capacity(sanitized.len());
    let mut prev_dash = false;
    for c in sanitized.chars() {
        if c == '-' {
            if !prev_dash {
                collapsed.push('-');
            }
            prev_dash = true;
        } else {
            collapsed.push(c);
            prev_dash = false;
        }
    }
    let trimmed = collapsed.trim_matches('-');
    format!("codegraph-{trimmed}")
}
```

- [ ] **Step 4: Run the tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: all tests pass (10 total: 7 from before + 3 new naming tests).

- [ ] **Step 5: Commit**

```bash
git add src/config/naming.rs tests/config_parsing.rs
git commit -m "feat(config): real Neo4j 5 db-name validation and derivation"
```

---

### Task 7: Secrets resolution (env > file > error)

**Files:**
- Modify: `src/config/secrets.rs`
- Test: append to `tests/config_parsing.rs`

- [ ] **Step 1: Write the failing tests**

Append to `tests/config_parsing.rs`:

> **Notes for the engineer:**
> - These tests mutate process-wide environment variables. cargo by default runs unit tests in parallel within a test binary; to avoid surprise interactions with later test additions, run unit tests with `--test-threads=1` once the suite is large enough that contention matters. For slice 1, each test uses a distinct env-var name so they don't collide.
> - `std::env::set_var` may emit a future-incompat lint warning on Rust 1.91 (it's safe in edition 2021 but became `unsafe` in edition 2024). Ignore the warning for slice 1.

```rust
use codegraph_rs::config::secrets::{resolve_neo4j_password, SecretsSource};

#[test]
fn resolves_password_from_env() {
    // Use a dedicated env var name to avoid polluting global state
    let key = "CODEGRAPH_RS_TEST_PASSWORD_ENV";
    std::env::set_var(key, "from-env");

    let dir = tempdir().unwrap();
    let pw = resolve_neo4j_password(dir.path(), Some(key)).expect("env should resolve");
    assert_eq!(pw.value, "from-env");
    assert_eq!(pw.source, SecretsSource::Environment);

    std::env::remove_var(key);
}

#[test]
fn resolves_password_from_secrets_file() {
    let dir = tempdir().unwrap();
    fs::create_dir(dir.path().join(".codegraph")).unwrap();
    fs::write(
        dir.path().join(".codegraph/secrets.toml"),
        "neo4j_password = \"from-file\"\n",
    ).unwrap();

    // Make sure the env var is not set
    std::env::remove_var("CODEGRAPH_RS_TEST_NO_ENV");
    let pw = resolve_neo4j_password(dir.path(), Some("CODEGRAPH_RS_TEST_NO_ENV"))
        .expect("file should resolve");
    assert_eq!(pw.value, "from-file");
    assert_eq!(pw.source, SecretsSource::File);
}

#[test]
fn missing_password_errors() {
    let dir = tempdir().unwrap();
    std::env::remove_var("CODEGRAPH_RS_TEST_DEFINITELY_UNSET");
    let err = resolve_neo4j_password(dir.path(), Some("CODEGRAPH_RS_TEST_DEFINITELY_UNSET"))
        .expect_err("missing both env and file should error");
    let msg = format!("{err:#}");
    assert!(msg.contains("password"));
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: compile errors about missing `resolve_neo4j_password` and `SecretsSource`.

- [ ] **Step 3: Implement secrets resolution**

Overwrite `src/config/secrets.rs`:

```rust
//! Resolve the Neo4j password.
//!
//! Order of precedence:
//!   1. The configured environment variable (default `CODEGRAPH_RS_NEO4J_PASSWORD`).
//!   2. `<workspace_dir>/.codegraph/secrets.toml` with key `neo4j_password`.
//!   3. Error.
//!
//! Interactive stdin prompting is deferred to a later slice.

use serde::Deserialize;
use std::path::Path;

#[derive(Debug, PartialEq, Eq)]
pub enum SecretsSource {
    Environment,
    File,
}

#[derive(Debug)]
pub struct ResolvedPassword {
    pub value: String,
    pub source: SecretsSource,
}

#[derive(Debug, Deserialize)]
struct SecretsFile {
    neo4j_password: String,
}

const DEFAULT_ENV_VAR: &str = "CODEGRAPH_RS_NEO4J_PASSWORD";

/// Resolve the Neo4j password.
///
/// `env_var` defaults to `CODEGRAPH_RS_NEO4J_PASSWORD` when `None`.
pub fn resolve_neo4j_password(
    workspace_dir: &Path,
    env_var: Option<&str>,
) -> anyhow::Result<ResolvedPassword> {
    let var = env_var.unwrap_or(DEFAULT_ENV_VAR);
    if let Ok(v) = std::env::var(var) {
        if !v.is_empty() {
            return Ok(ResolvedPassword { value: v, source: SecretsSource::Environment });
        }
    }

    let secrets_path = workspace_dir.join(".codegraph").join("secrets.toml");
    if secrets_path.is_file() {
        let raw = std::fs::read_to_string(&secrets_path).map_err(|e| {
            anyhow::anyhow!("cannot read `{}`: {e}", secrets_path.display())
        })?;
        let parsed: SecretsFile = toml::from_str(&raw).map_err(|e| {
            anyhow::anyhow!("invalid secrets.toml: {e}")
        })?;
        return Ok(ResolvedPassword {
            value: parsed.neo4j_password,
            source: SecretsSource::File,
        });
    }

    anyhow::bail!(
        "Neo4j password not found. Set the `{var}` environment variable, \
         or create `<workspace>/.codegraph/secrets.toml` with `neo4j_password = \"...\"`."
    );
}

/// Default env-var name. Public so other modules can reference it consistently.
pub fn default_env_var() -> &'static str {
    DEFAULT_ENV_VAR
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test config_parsing 2>&1 | tail -10
```

Expected: all tests pass (13 total).

- [ ] **Step 5: Commit**

```bash
git add src/config/secrets.rs tests/config_parsing.rs
git commit -m "feat(config): resolve Neo4j password from env > file"
```

---

### Task 8: Neo4j Bolt connection wrapper

**Files:**
- Create: `src/db/connection.rs`
- Modify: `src/db/mod.rs`
- Test: `tests/integration_db.rs` (gated on env var)

- [ ] **Step 1: Implement the connection wrapper**

Create `src/db/connection.rs`:

```rust
//! Neo4j Bolt connection wrapper.

use anyhow::{Context, Result};
use neo4rs::{ConfigBuilder, Graph};

/// A connected Neo4j Bolt session targeting a specific database.
///
/// The underlying `neo4rs::Graph` is itself a connection pool; cloning is cheap.
#[derive(Clone)]
pub struct Connection {
    pub graph: Graph,
    pub database: String,
}

impl Connection {
    /// Connect to a Neo4j DBMS targeting the given `database`. The database must
    /// already exist; use [`Connection::system`] to operate on `system` and create one.
    pub async fn open(uri: &str, user: &str, password: &str, database: &str) -> Result<Self> {
        let config = ConfigBuilder::default()
            .uri(uri)
            .user(user)
            .password(password)
            .db(database)
            .build()
            .context("invalid neo4rs config")?;
        let graph = Graph::connect(config)
            .await
            .with_context(|| format!("cannot connect to Neo4j at `{uri}` as `{user}`, db `{database}`"))?;
        let conn = Self { graph, database: database.to_string() };
        conn.ping().await.with_context(|| format!("connection ping to `{database}` failed"))?;
        Ok(conn)
    }

    /// Connect to the `system` database (used to create user databases).
    pub async fn system(uri: &str, user: &str, password: &str) -> Result<Self> {
        Self::open(uri, user, password, "system").await
    }

    /// Verify the connection by running a trivial query.
    pub async fn ping(&self) -> Result<()> {
        let mut stream = self.graph.execute(neo4rs::query("RETURN 1 AS x")).await
            .context("ping query failed")?;
        // First `.context` attaches to a Result<_, neo4rs::Error> (DB-level error).
        // Second `.context` attaches to Option::None (empty stream — anyhow's
        // `Context for Option` converts None into an error with this message).
        let row = stream.next().await
            .context("ping query stream errored")?
            .context("ping returned no row (empty stream)")?;
        let x: i64 = row.get("x").context("ping row missing `x`")?;
        anyhow::ensure!(x == 1, "ping returned unexpected value: {x}");
        Ok(())
    }
}
```

- [ ] **Step 2: Re-export from src/db/mod.rs**

Overwrite `src/db/mod.rs`:

```rust
//! Neo4j Bolt client and schema management.

pub mod connection;
pub mod schema;

pub use connection::Connection;
```

- [ ] **Step 3: Create the schema module stub (filled in by Task 9)**

Create `src/db/schema.rs`:

```rust
//! Neo4j schema (constraints + indexes). Filled in by Task 9.

use crate::db::Connection;

pub async fn apply_schema(_conn: &Connection) -> anyhow::Result<()> {
    anyhow::bail!("apply_schema: not yet implemented (filled in by Task 9)")
}
```

- [ ] **Step 4: Write the integration test**

Create `tests/integration_db.rs`:

```rust
//! Integration tests that require a live Neo4j Desktop DBMS.
//! Run with: CODEGRAPH_RS_TEST_NEO4J=1 cargo test --test integration_db -- --nocapture
//!
//! Requires env vars:
//!   CODEGRAPH_RS_TEST_NEO4J_URI       (default bolt://localhost:7687)
//!   CODEGRAPH_RS_TEST_NEO4J_USER      (default neo4j)
//!   CODEGRAPH_RS_TEST_NEO4J_PASSWORD  (required)

use codegraph_rs::db::Connection;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

fn env(var: &str, default: Option<&str>) -> String {
    std::env::var(var).ok().or_else(|| default.map(String::from))
        .unwrap_or_else(|| panic!("env var `{var}` is required for this test"))
}

#[tokio::test]
async fn connects_to_system_database() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }

    let uri = env("CODEGRAPH_RS_TEST_NEO4J_URI", Some("bolt://localhost:7687"));
    let user = env("CODEGRAPH_RS_TEST_NEO4J_USER", Some("neo4j"));
    let pw = env("CODEGRAPH_RS_TEST_NEO4J_PASSWORD", None);

    let conn = Connection::system(&uri, &user, &pw).await.expect("system connect must succeed");
    conn.ping().await.expect("ping must succeed");
    assert_eq!(conn.database, "system");
}
```

- [ ] **Step 5: Build and verify the gated test compiles and skips by default**

```bash
cargo test --test integration_db 2>&1 | tail -5
```

Expected: 1 test, passes via the SKIP path (no env var set), printed `SKIP:` line.

- [ ] **Step 6: Run the integration test against a real Neo4j (manual)**

This step requires Neo4j Desktop to be running.

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-actual-password' \
cargo test --test integration_db -- --nocapture
```

Expected: `1 passed`, no `SKIP:` printed. If this fails, do NOT proceed — fix the connection issue first (most common: wrong password, DBMS not started).

- [ ] **Step 7: Commit**

```bash
git add src/db tests/integration_db.rs
git commit -m "feat(db): add Neo4j Bolt connection wrapper with ping verification"
```

---

### Task 9: Schema application

**Files:**
- Modify: `src/db/schema.rs`
- Test: append to `tests/integration_db.rs`

- [ ] **Step 1: Implement schema application**

Overwrite `src/db/schema.rs`:

```rust
//! Neo4j schema: constraints, indexes, FTS index, vector index.
//! All statements are `IF NOT EXISTS` to make this routine idempotent.

use anyhow::{Context, Result};
use neo4rs::query;

use crate::db::Connection;

const VECTOR_DIMENSIONS: u32 = 384;

/// Apply the codegraph-rs schema to the connection's target database.
/// Safe to call repeatedly — every statement uses `IF NOT EXISTS`.
pub async fn apply_schema(conn: &Connection) -> Result<()> {
    let stmts: &[(&str, &str)] = &[
        (
            "symbol_qid_unique constraint",
            "CREATE CONSTRAINT symbol_qid_unique IF NOT EXISTS \
             FOR (n:Symbol) REQUIRE n.qid IS UNIQUE",
        ),
        (
            "symbol_name_idx",
            "CREATE INDEX symbol_name_idx IF NOT EXISTS \
             FOR (n:Symbol) ON (n.name)",
        ),
        (
            "symbol_qualified_idx",
            "CREATE INDEX symbol_qualified_idx IF NOT EXISTS \
             FOR (n:Symbol) ON (n.qualified_name)",
        ),
        (
            "file_path_idx",
            "CREATE INDEX file_path_idx IF NOT EXISTS \
             FOR (n:File) ON (n.root_name, n.relative_path)",
        ),
        (
            "unresolved_ref_qid_unique",
            "CREATE CONSTRAINT unresolved_ref_qid_unique IF NOT EXISTS \
             FOR (n:UnresolvedRef) REQUIRE n.qid IS UNIQUE",
        ),
        (
            "symbol_fts (full-text)",
            "CREATE FULLTEXT INDEX symbol_fts IF NOT EXISTS \
             FOR (n:Symbol) ON EACH [n.name, n.qualified_name, n.signature, n.doc]",
        ),
    ];

    for (label, cypher) in stmts {
        conn.graph
            .execute(query(cypher))
            .await
            .with_context(|| format!("schema step `{label}` failed"))?
            .next()
            .await
            .ok();
    }

    // Vector index has a more complex options map and is created separately.
    let vector_cypher = format!(
        "CREATE VECTOR INDEX symbol_embedding IF NOT EXISTS \
         FOR (n:Symbol) ON n.embedding \
         OPTIONS {{ \
           indexConfig: {{ \
             `vector.dimensions`: {VECTOR_DIMENSIONS}, \
             `vector.similarity_function`: 'cosine' \
           }} \
         }}"
    );
    conn.graph
        .execute(query(&vector_cypher))
        .await
        .context("creating vector index `symbol_embedding` failed")?
        .next()
        .await
        .ok();

    Ok(())
}
```

- [ ] **Step 2: Append integration test for schema**

Append to `tests/integration_db.rs`:

```rust
use codegraph_rs::db::schema::apply_schema;
use neo4rs::query;

#[tokio::test]
async fn applies_schema_to_fresh_test_database() {
    if !enabled() { eprintln!("SKIP"); return; }

    let uri = env("CODEGRAPH_RS_TEST_NEO4J_URI", Some("bolt://localhost:7687"));
    let user = env("CODEGRAPH_RS_TEST_NEO4J_USER", Some("neo4j"));
    let pw = env("CODEGRAPH_RS_TEST_NEO4J_PASSWORD", None);
    let db_name = format!("codegraph-test-{}", std::process::id());

    // Create the database via system, then connect to it
    let sys = Connection::system(&uri, &user, &pw).await.unwrap();
    sys.graph
        .execute(query(&format!("CREATE DATABASE `{db_name}` IF NOT EXISTS WAIT")))
        .await
        .expect("CREATE DATABASE")
        .next()
        .await
        .ok();

    let conn = Connection::open(&uri, &user, &pw, &db_name).await.expect("connect to test db");
    apply_schema(&conn).await.expect("apply_schema must succeed on fresh db");
    apply_schema(&conn).await.expect("apply_schema must be idempotent");

    // Spot check that the constraint exists
    let mut rows = conn.graph
        .execute(query("SHOW CONSTRAINTS YIELD name RETURN name"))
        .await.unwrap();
    let mut names = Vec::new();
    while let Some(row) = rows.next().await.unwrap() {
        names.push(row.get::<String>("name").unwrap());
    }
    assert!(names.iter().any(|n| n == "symbol_qid_unique"),
        "expected symbol_qid_unique among: {names:?}");

    // Cleanup: drop the test database
    sys.graph
        .execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
        .await
        .expect("DROP DATABASE")
        .next()
        .await
        .ok();
}
```

- [ ] **Step 3: Run integration tests against Neo4j**

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-actual-password' \
cargo test --test integration_db -- --nocapture --test-threads=1
```

Expected: 2 tests pass. The `--test-threads=1` is important — these tests share a Neo4j instance and shouldn't run concurrently.

- [ ] **Step 4: Commit**

```bash
git add src/db/schema.rs tests/integration_db.rs
git commit -m "feat(db): apply codegraph-rs schema (constraints, indexes, fts, vector) idempotently"
```

---

### Task 10: `init` command — full implementation

**Files:**
- Modify: `src/cli/init.rs`
- Test: `tests/integration_init.rs` (gated)

- [ ] **Step 1: Implement init**

Overwrite `src/cli/init.rs`:

```rust
//! `codegraph-rs init` — scaffold a workspace.
//!
//! Order of operations is fail-fast: validate Neo4j and apply schema BEFORE
//! writing any config files. If anything fails, the workspace directory is
//! left as the user provided it (no half-initialized state).

use anyhow::{Context, Result};
use clap::Args;
use neo4rs::query;
use std::path::{Path, PathBuf};
use tracing::info;

use crate::config::naming::{derive_database_name, validate_database_name};
use crate::db::{Connection, schema::apply_schema};

#[derive(Debug, Args)]
pub struct InitArgs {
    /// Workspace name (used to derive the Neo4j database name).
    #[arg(long)]
    pub name: String,

    /// Neo4j Bolt URI.
    #[arg(long, default_value = "bolt://localhost:7687")]
    pub neo4j_uri: String,

    /// Neo4j user.
    #[arg(long, default_value = "neo4j")]
    pub neo4j_user: String,

    /// Neo4j password. Prefer the CODEGRAPH_RS_NEO4J_PASSWORD env var.
    #[arg(long, env = "CODEGRAPH_RS_NEO4J_PASSWORD")]
    pub neo4j_password: String,

    /// Workspace roots in the form `name=path`.
    #[arg(long, value_name = "NAME=PATH", required = true, num_args = 1..)]
    pub root: Vec<String>,
}

pub fn run(args: InitArgs, workspace: Option<&Path>) -> Result<()> {
    let workspace_dir = workspace.map(Path::to_path_buf).unwrap_or_else(|| {
        std::env::current_dir().expect("cwd readable")
    });

    // Tokio runtime for the async Neo4j calls.
    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    rt.block_on(async { run_async(args, &workspace_dir).await })
}

async fn run_async(args: InitArgs, workspace_dir: &Path) -> Result<()> {
    let db_name = derive_database_name(&args.name);
    validate_database_name(&db_name).map_err(|e| anyhow::anyhow!("derived db name `{db_name}` invalid: {e}"))?;

    let roots = parse_roots(&args.root, workspace_dir)?;

    info!("validating Neo4j connection at {}", args.neo4j_uri);
    let sys = Connection::system(&args.neo4j_uri, &args.neo4j_user, &args.neo4j_password).await
        .context("Neo4j connection failed; is the DBMS running?")?;

    info!("creating database `{db_name}` if missing");
    sys.graph
        .execute(query(&format!("CREATE DATABASE `{db_name}` IF NOT EXISTS WAIT")))
        .await
        .with_context(|| format!("CREATE DATABASE `{db_name}` failed (do you have CREATE DATABASE privilege?)"))?
        .next()
        .await
        .ok();

    let conn = Connection::open(&args.neo4j_uri, &args.neo4j_user, &args.neo4j_password, &db_name)
        .await
        .with_context(|| format!("cannot open created database `{db_name}`"))?;

    info!("applying schema");
    apply_schema(&conn).await.context("schema apply failed")?;

    // All Neo4j operations succeeded — now write config files.
    write_config_files(workspace_dir, &args, &db_name, &roots)?;
    println!("Initialized workspace `{}` -> Neo4j database `{db_name}`", args.name);
    Ok(())
}

struct ParsedRoot { name: String, path: PathBuf }

fn parse_roots(items: &[String], workspace_dir: &Path) -> Result<Vec<ParsedRoot>> {
    let mut out = Vec::with_capacity(items.len());
    for item in items {
        let (name, path) = item.split_once('=').ok_or_else(|| {
            anyhow::anyhow!("--root must be NAME=PATH (got `{item}`)")
        })?;
        let resolved = if PathBuf::from(path).is_absolute() {
            PathBuf::from(path)
        } else {
            workspace_dir.join(path)
        };
        if !resolved.is_dir() {
            anyhow::bail!("root `{name}` path `{}` is not a directory", resolved.display());
        }
        out.push(ParsedRoot { name: name.to_string(), path: PathBuf::from(path) });
    }
    Ok(out)
}

fn write_config_files(
    workspace_dir: &Path,
    args: &InitArgs,
    db_name: &str,
    roots: &[ParsedRoot],
) -> Result<()> {
    let codegraph_dir = workspace_dir.join(".codegraph");
    std::fs::create_dir_all(&codegraph_dir).with_context(|| {
        format!("creating {}", codegraph_dir.display())
    })?;

    let mut config_toml = String::new();
    config_toml.push_str("schema_version = 1\n\n");
    config_toml.push_str("[workspace]\n");
    config_toml.push_str(&format!("name = \"{}\"\n\n", args.name));
    config_toml.push_str("[workspace.neo4j]\n");
    config_toml.push_str(&format!("uri = \"{}\"\n", args.neo4j_uri));
    config_toml.push_str(&format!("database = \"{}\"\n", db_name));
    config_toml.push_str(&format!("username = \"{}\"\n\n", args.neo4j_user));
    for r in roots {
        config_toml.push_str("[[workspace.roots]]\n");
        config_toml.push_str(&format!("name = \"{}\"\n", r.name));
        config_toml.push_str(&format!("path = \"{}\"\n\n", r.path.display()));
    }
    config_toml.push_str("[indexing]\nlanguages = [\"rust\", \"daml\"]\n\n");
    config_toml.push_str("[embeddings]\nenabled = true\nmodel = \"bge-small-en-v1.5\"\n");

    let config_path = codegraph_dir.join("config.toml");
    std::fs::write(&config_path, config_toml)
        .with_context(|| format!("writing {}", config_path.display()))?;

    let secrets_path = codegraph_dir.join("secrets.toml");
    let secrets_content = format!("neo4j_password = \"{}\"\n", args.neo4j_password.replace('"', "\\\""));
    std::fs::write(&secrets_path, secrets_content)
        .with_context(|| format!("writing {}", secrets_path.display()))?;

    // Restrict secrets.toml permissions on Unix.
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let mut perm = std::fs::metadata(&secrets_path)?.permissions();
        perm.set_mode(0o600);
        std::fs::set_permissions(&secrets_path, perm)?;
    }

    // Ensure secrets.toml is gitignored. Append-or-create the workspace .gitignore.
    let gitignore_path = workspace_dir.join(".gitignore");
    let line = ".codegraph/secrets.toml\n";
    let existing = std::fs::read_to_string(&gitignore_path).unwrap_or_default();
    if !existing.lines().any(|l| l.trim() == ".codegraph/secrets.toml") {
        let combined = if existing.ends_with('\n') || existing.is_empty() {
            format!("{existing}{line}")
        } else {
            format!("{existing}\n{line}")
        };
        std::fs::write(&gitignore_path, combined)
            .with_context(|| format!("updating {}", gitignore_path.display()))?;
    }

    Ok(())
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cargo build 2>&1 | tail -3
```

Expected: clean build, possibly some unused-import warnings.

- [ ] **Step 3: Write the integration test**

Create `tests/integration_init.rs`:

```rust
//! Integration test for `codegraph-rs init`. Gated on env var.

use codegraph_rs::cli::init::{InitArgs, run};
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn init_creates_workspace_and_database() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }

    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("init-test-{}", std::process::id());
    let root_subdir = dir.path().join("root1");
    fs::create_dir(&root_subdir).unwrap();

    let args = InitArgs {
        name: workspace_name.clone(),
        neo4j_uri: "bolt://localhost:7687".into(),
        neo4j_user: "neo4j".into(),
        neo4j_password: pw.clone(),
        root: vec![format!("only=root1")],
    };

    run(args, Some(dir.path())).expect("init must succeed");

    // Files written?
    assert!(dir.path().join(".codegraph/config.toml").is_file());
    assert!(dir.path().join(".codegraph/secrets.toml").is_file());
    assert!(dir.path().join(".gitignore").is_file());

    let gi = fs::read_to_string(dir.path().join(".gitignore")).unwrap();
    assert!(gi.contains(".codegraph/secrets.toml"), "gitignore must contain secrets path");

    // Drop the database to keep the local DBMS clean.
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        let db_name = format!("codegraph-init-test-{}", std::process::id());
        sys.graph.execute(neo4rs::query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await.unwrap().next().await.ok();
    });
}
```

- [ ] **Step 4: Run the integration test**

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-actual-password' \
cargo test --test integration_init -- --nocapture
```

Expected: test passes; `init` reports success; `.codegraph/config.toml` and `secrets.toml` exist after.

- [ ] **Step 5: Manual smoke test**

```bash
mkdir -p /tmp/cg-rs-smoke && cd /tmp/cg-rs-smoke
mkdir my-root
CODEGRAPH_RS_NEO4J_PASSWORD='your-actual-password' \
cargo run --manifest-path /Users/gyorgybalazsi/codegraph-rs/Cargo.toml -- init \
  --name smoke \
  --root primary=my-root
cat .codegraph/config.toml
cat .gitignore
```

Expected: prints "Initialized workspace `smoke` -> Neo4j database `codegraph-smoke`"; config.toml shows the workspace; .gitignore includes the secrets path.

Cleanup:

```bash
cd /Users/gyorgybalazsi/codegraph-rs
# Drop the smoke db
cargo run --quiet -- 2>&1 | head -1   # noop — we'll add a 'drop' command later
# For now, drop manually via Neo4j Browser or cypher-shell:
#   DROP DATABASE `codegraph-smoke` IF EXISTS;
rm -rf /tmp/cg-rs-smoke
```

- [ ] **Step 6: Commit**

```bash
git add src/cli/init.rs tests/integration_init.rs
git commit -m "feat(cli): implement init — validate Neo4j, CREATE DATABASE, apply schema, write config"
```

---

### Task 11: `status` command — basic counts

**Files:**
- Modify: `src/cli/status.rs`
- Test: `tests/integration_status.rs` (gated)

- [ ] **Step 1: Implement status**

Overwrite `src/cli/status.rs`:

```rust
//! `codegraph-rs status` — print workspace stats.

use anyhow::{Context, Result};
use neo4rs::query;
use std::path::Path;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::Connection;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg.map(Path::to_path_buf).unwrap_or_else(|| {
        std::env::current_dir().expect("cwd readable")
    });
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        ).await.context("connecting to workspace database")?;

        let file_count: i64 = single_count(&conn, "MATCH (n:File) RETURN count(n) AS c").await?;
        let symbol_count: i64 = single_count(&conn, "MATCH (n:Symbol) RETURN count(n) AS c").await?;
        let unresolved: i64 = single_count(&conn, "MATCH (n:UnresolvedRef) RETURN count(n) AS c").await?;
        let last_indexed: Option<i64> = max_last_indexed(&conn).await?;

        println!("Workspace: {}", cfg.workspace.name);
        println!("Database:  {}", cfg.workspace.neo4j.database);
        println!("Roots:     {}", cfg.workspace.roots.iter().map(|r| r.name.as_str()).collect::<Vec<_>>().join(", "));
        println!("Files:     {file_count}");
        println!("Symbols:   {symbol_count}");
        println!("Unresolved refs: {unresolved}");
        println!("Last indexed:    {}", match last_indexed { Some(t) => t.to_string(), None => "never".into() });
        Ok::<(), anyhow::Error>(())
    })
}

async fn single_count(conn: &Connection, cypher: &str) -> Result<i64> {
    let mut stream = conn.graph.execute(query(cypher)).await
        .with_context(|| format!("running `{cypher}`"))?;
    // First context: DB-level error from the row stream.
    // Second context: stream finished without producing a row (Option::None case
    // — anyhow's Context impl for Option turns None into this error).
    let row = stream.next().await
        .with_context(|| format!("stream error reading result of `{cypher}`"))?
        .with_context(|| format!("`{cypher}` returned no row"))?;
    row.get("c").context("row missing `c`")
}

async fn max_last_indexed(conn: &Connection) -> Result<Option<i64>> {
    let mut stream = conn.graph
        .execute(query("MATCH (n:File) RETURN max(n.last_indexed_at) AS t"))
        .await?;

    // Handle both possibilities: stream may produce zero rows (no Files yet)
    // or one row whose `t` is NULL (which deserializes to None).
    match stream.next().await? {
        None => Ok(None),
        Some(row) => row.get::<Option<i64>>("t").context("missing `t`"),
    }
}
```

- [ ] **Step 2: Build to verify**

```bash
cargo build 2>&1 | tail -3
```

Expected: clean build.

- [ ] **Step 3: Write the integration test**

Create `tests/integration_status.rs`:

```rust
//! Integration test for `codegraph-rs status`. Gated on env var.

use codegraph_rs::cli::{init::{InitArgs, self}, status};
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn status_after_init_reports_zero_counts() {
    if !enabled() {
        eprintln!("SKIP");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("status-test-{}", std::process::id());
    fs::create_dir(dir.path().join("root1")).unwrap();

    let args = InitArgs {
        name: workspace_name.clone(),
        neo4j_uri: "bolt://localhost:7687".into(),
        neo4j_user: "neo4j".into(),
        neo4j_password: pw.clone(),
        root: vec!["only=root1".into()],
    };
    init::run(args, Some(dir.path())).expect("init");

    // Set env so status's secrets resolution finds it
    std::env::set_var("CODEGRAPH_RS_NEO4J_PASSWORD", &pw);
    status::run(Some(dir.path())).expect("status");
    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    // Drop test db
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        let db_name = format!("codegraph-status-test-{}", std::process::id());
        sys.graph.execute(neo4rs::query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await.unwrap().next().await.ok();
    });
}
```

- [ ] **Step 4: Run the test**

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-actual-password' \
cargo test --test integration_status -- --nocapture --test-threads=1
```

Expected: status output prints "Files: 0", "Symbols: 0", "Unresolved refs: 0", "Last indexed: never".

- [ ] **Step 5: Commit**

```bash
git add src/cli/status.rs tests/integration_status.rs
git commit -m "feat(cli): implement status — print file/symbol counts and last index time"
```

---

### Task 12: README, final cleanup, slice 1 tag

**Files:**
- Create: `/Users/gyorgybalazsi/codegraph-rs/README.md`

- [ ] **Step 1: Create the README**

Create `/Users/gyorgybalazsi/codegraph-rs/README.md`:

```markdown
# codegraph-rs

Local-first code intelligence — the Rust port of [CodeGraph](https://github.com/colbymchenry/codegraph),
backed by Neo4j.

## Status

In development. Currently implements **Slice 1: Walking Skeleton** —
workspace initialization, schema setup, and status reporting. Indexing,
resolution, vector search, and the MCP server land in subsequent slices.

## Prerequisites

- Rust 1.91+ (`rustup install 1.91`)
- Neo4j Desktop (Enterprise Edition, free for local dev). The DBMS must be
  running with default user `neo4j` and a known password.

## Quick start

```bash
cargo install --path .

mkdir my-workspace && cd my-workspace
mkdir my-source-root
export CODEGRAPH_RS_NEO4J_PASSWORD='your-neo4j-password'
codegraph-rs init --name my-workspace --root primary=my-source-root

codegraph-rs status
```

## Development

```bash
cargo test                         # unit tests
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='...' \
  cargo test -- --test-threads=1   # full suite, including integration tests
```

## Design

See [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](https://github.com/bitsafe/codegraph/blob/main/docs/superpowers/specs/2026-04-22-codegraph-rs-design.md)
in the parent CodeGraph repo for the architectural spec.

## License

MIT.
```

- [ ] **Step 2: Run the full test suite one last time**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo test 2>&1 | tail -10
```

Expected: all unit tests pass. Integration tests print `SKIP:` (no env var set) — that's fine.

- [ ] **Step 3: Run clippy and rustfmt**

```bash
cargo clippy --all-targets 2>&1 | tail -20
cargo fmt --check 2>&1 | tail -5
```

Expected: no clippy warnings (or only pre-approved ones); `cargo fmt --check` exits 0. If `cargo fmt --check` fails, run `cargo fmt` and re-stage.

- [ ] **Step 4: Run integration tests against Neo4j once more, end-to-end**

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-actual-password' \
cargo test -- --test-threads=1 2>&1 | tail -15
```

Expected: all tests (unit + integration) pass.

- [ ] **Step 5: Final commit**

```bash
git add README.md
git commit -m "docs: add README for slice 1 walking skeleton"
git tag slice-1-walking-skeleton
git log --oneline | head -15
```

Expected: clean commit history with one commit per task and a final tag.

---

## Slice 1 Done

Verify the artifact:

- [ ] `cargo build` succeeds with no errors.
- [ ] `cargo test` (no env vars) — all unit tests pass; integration tests print `SKIP:`.
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass.
- [ ] `codegraph-rs --help` shows all 6 subcommands.
- [ ] `codegraph-rs init --name X --root r=./X` against a real Neo4j Desktop creates `.codegraph/config.toml` and the database.
- [ ] `codegraph-rs status` after init prints zero counts.
- [ ] `codegraph-rs index` prints `not yet implemented in slice 1` and exits nonzero.

When all check, slice 1 is complete and we can write slice 2 (Rust extraction).
