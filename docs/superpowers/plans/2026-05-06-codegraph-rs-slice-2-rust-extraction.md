# codegraph-rs Slice 2 — Rust Extraction

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Index a Rust codebase. After this slice, `codegraph-rs index` walks a workspace's source roots, parses every `.rs` file with tree-sitter, extracts `Symbol`/`Function`/`Struct`/`Enum`/`Trait`/`TypeAlias`/`Module`/`Import` nodes plus structural `CONTAINS` edges, persists `UnresolvedRef` records for call sites, and writes everything to Neo4j. Resolution-derived edges (`CALLS`, etc.) come in Slice 3.

**Architecture:** Walk filesystems with `ignore` (ripgrep's walker, .gitignore-aware). Parse with `tree-sitter` + `tree-sitter-rust`, one `Parser` per worker thread (parsers are not `Sync`). Each language has a `LanguageExtractor` trait impl that runs declarative tree-sitter queries (`.scm` files) over the AST and produces an `ExtractionResult { nodes, edges, unresolved_refs }`. `rayon` parallelizes parsing across files; results stream through a `tokio::mpsc` channel into an async DB writer that batches `UNWIND $batch AS row MERGE ...` statements. Content hash via `blake3` for sync diffing.

**Tech Stack:** Rust 1.91, tree-sitter 0.25, tree-sitter-rust 0.24, ignore 0.4, blake3 1.8, rayon 1.12; existing slice-1 stack (clap, serde, toml, tokio, neo4rs, anyhow, tracing).

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) — primarily §4 (schema), §6 (extraction pipeline). This slice does NOT implement §7 (resolution) or §8 (vectors).

**Slice 1 baseline:** HEAD of `codegraph-rs` is `82dde3a` (tagged `slice-1-walking-skeleton`). Slice 2 builds on it.

---

## What gets indexed in Slice 2

For each `.rs` file in a workspace root:

| AST node | Becomes | Notes |
|---|---|---|
| The file itself | `(:File:Symbol)` | Stores `content_hash` (BLAKE3) and `last_indexed_at` |
| `function_item` | `(:Function:Symbol)` | qualified_name = `<crate-relative-path>::<name>` |
| `struct_item` | `(:Struct:Symbol)` | |
| `enum_item` | `(:Enum:Symbol)` | |
| `trait_item` | `(:Trait:Symbol)` | |
| `type_item` | `(:TypeAlias:Symbol)` | |
| `mod_item` | `(:Module:Symbol)` | nested modules within a file |
| `use_declaration` | `(:Import:Symbol)` | the source-side; resolution to a real `Module` is Slice 3 |
| `call_expression` | `(:UnresolvedRef)` with `kind="call"` | persistent record; resolution to `:CALLS` edges is Slice 3 |
| `(File)-CONTAINS->(every nested Symbol)` | structural edge | parent for non-impl items is the file |
| `(Symbol)-HAS_REF->(UnresolvedRef)` | structural edge | source-side of the unresolved reference |

**`impl` blocks:** per the spec, `impl` is NOT a node. Methods in `impl Foo { fn bar() }` are emitted as `Method` nodes whose `qualified_name` carries the type prefix (`<path>::Foo::bar`). Their `CONTAINS` parent is the source file.

**Out of scope for Slice 2** (intentionally):
- `CALLS`, `IMPORTS` (target side), `REFERENCES` edges — resolution-derived, comes in Slice 3
- Field/property/parameter/variable extraction (Slice 3 might add these as warranted)
- Daml support (Slice 7)
- Embeddings (Slice 4)
- Sync (Slice 5) — full reindex only
- Any framework-specific enrichment

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml                                  # adds extraction deps
  src/
    lib.rs                                    # add `pub mod extraction;`
    cli/
      index.rs                                # NEW: real `index` command (replaces dispatch stub)
      mod.rs                                  # wire Index variant -> index::run
    extraction/
      mod.rs                                  # re-exports
      result.rs                               # ExtractionResult + RawNode + RawEdge + UnresolvedRef + enums
      walker.rs                               # ignore-crate file discovery
      parser.rs                               # per-thread tree-sitter parser
      hash.rs                                 # blake3 content hash
      orchestrator.rs                         # ties walker + parser + extractor + writer
      languages/
        mod.rs                                # LanguageExtractor trait + registry
        rust.rs                               # Rust extractor
        rust.scm                              # tree-sitter capture queries
    db/
      write.rs                                # NEW: write_extraction (UNWIND/MERGE batches)
      mod.rs                                  # add `pub mod write;`
  tests/
    extraction_unit.rs                        # in-memory unit tests for walker/hash/result types
    fixtures/
      tiny-rust/                              # 3-file Rust mini-project, used by integration_index
        Cargo.toml
        src/
          main.rs
          util.rs
          lib.rs
    integration_index.rs                      # gated; indexes tiny-rust against live Neo4j
```

---

### Task 1: Add slice-2 dependencies and module skeleton

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/lib.rs`
- Create: `src/extraction/mod.rs`
- Create: `src/extraction/result.rs` (stub)
- Create: `src/extraction/walker.rs` (stub)
- Create: `src/extraction/parser.rs` (stub)
- Create: `src/extraction/hash.rs` (stub)
- Create: `src/extraction/orchestrator.rs` (stub)
- Create: `src/extraction/languages/mod.rs` (stub)
- Create: `src/extraction/languages/rust.rs` (stub)
- Create: `src/extraction/languages/rust.scm` (empty)
- Create: `src/db/write.rs` (stub)
- Modify: `src/db/mod.rs` (add `pub mod write;`)

- [ ] **Step 1: Add extraction deps to Cargo.toml**

In `/Users/gyorgybalazsi/codegraph-rs/Cargo.toml`, add to `[dependencies]`:

```toml
tree-sitter = "0.25"
tree-sitter-rust = "0.24"
ignore = "0.4"
blake3 = "1"
rayon = "1"
```

Place these alphabetically among the existing deps. Result section should look like:

```toml
[dependencies]
anyhow = "1"
blake3 = "1"
clap = { version = "4", features = ["derive", "env"] }
ignore = "0.4"
neo4rs = "0.8"
rayon = "1"
serde = { version = "1", features = ["derive"] }
thiserror = "2"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "io-util", "io-std", "fs", "process", "signal"] }
toml = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
tree-sitter = "0.25"
tree-sitter-rust = "0.24"
```

(Order may differ from the existing file — match the existing alphabetical convention.)

- [ ] **Step 2: Update src/lib.rs**

Add `pub mod extraction;` to `src/lib.rs` after the existing `pub mod db;`:

```rust
pub mod cli;
pub mod config;
pub mod db;
pub mod extraction;
```

- [ ] **Step 3: Create the extraction module skeleton**

Create `src/extraction/mod.rs`:

```rust
//! Tree-sitter-based extraction pipeline.

pub mod hash;
pub mod languages;
pub mod orchestrator;
pub mod parser;
pub mod result;
pub mod walker;

pub use orchestrator::run_extraction;
pub use result::{ExtractionResult, RawEdge, RawNode, UnresolvedRef};
```

Create `src/extraction/result.rs`:

```rust
//! ExtractionResult and the raw node/edge/ref types. Filled in by Task 2.
```

Create `src/extraction/walker.rs`:

```rust
//! File discovery via the `ignore` crate. Filled in by Task 3.
```

Create `src/extraction/hash.rs`:

```rust
//! BLAKE3 content hashing. Filled in by Task 4.
```

Create `src/extraction/parser.rs`:

```rust
//! Per-thread tree-sitter parser. Filled in by Task 5.
```

Create `src/extraction/orchestrator.rs`:

```rust
//! Ties walker + parser + extractor + writer together. Filled in by Task 11.

use anyhow::Result;
use std::path::Path;

pub fn run_extraction(_workspace_dir: &Path) -> Result<()> {
    anyhow::bail!("run_extraction: not yet implemented (filled in by Task 11)")
}
```

Create `src/extraction/languages/mod.rs`:

```rust
//! Language-specific extractors. Filled in by Tasks 6–9.

pub mod rust;
```

Create `src/extraction/languages/rust.rs`:

```rust
//! Rust extractor. Filled in by Tasks 6–9.
```

Create an empty `src/extraction/languages/rust.scm` file (just a newline; Tasks 6–9 fill it).

- [ ] **Step 4: Create db/write.rs stub**

Create `src/db/write.rs`:

```rust
//! Batched extraction-output writer. Filled in by Task 10.

use anyhow::Result;

use crate::db::Connection;
use crate::extraction::ExtractionResult;

pub async fn write_extraction(_conn: &Connection, _results: Vec<ExtractionResult>) -> Result<()> {
    anyhow::bail!("write_extraction: not yet implemented (filled in by Task 10)")
}
```

Modify `src/db/mod.rs` to add the new module. After the existing `pub mod schema;`:

```rust
pub mod connection;
pub mod schema;
pub mod write;

pub use connection::Connection;
```

- [ ] **Step 5: Build to verify the skeleton compiles**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo build 2>&1 | tail -10
```

Expected: clean build (downloading + compiling tree-sitter-rust takes a minute on first run; subsequent builds are fast). Possibly some unused-import warnings.

- [ ] **Step 6: Run the existing test suite**

```bash
cargo test 2>&1 | tail -10
```

Expected: all 15 unit tests + 4 integration SKIP-mode tests still pass. Slice 2 hasn't broken slice 1.

- [ ] **Step 7: Commit**

```bash
git add Cargo.toml Cargo.lock src/
git commit -m "chore(extraction): add deps and module skeleton for slice 2"
```

---

### Task 2: ExtractionResult types (TDD)

**Files:**
- Modify: `src/extraction/result.rs`
- Create: `tests/extraction_unit.rs`

- [ ] **Step 1: Write the failing tests**

Create `tests/extraction_unit.rs`:

```rust
//! Unit tests for extraction types and helpers.

use codegraph_rs::extraction::result::{
    EdgeKind, ExtractionResult, NodeKind, RawEdge, RawNode, UnresolvedKind, UnresolvedRef,
};

#[test]
fn extraction_result_default_is_empty() {
    let r = ExtractionResult::default();
    assert!(r.nodes.is_empty());
    assert!(r.edges.is_empty());
    assert!(r.unresolved_refs.is_empty());
}

#[test]
fn raw_node_is_constructible() {
    let n = RawNode {
        qid: "rs:src/lib.rs:crate::foo".to_string(),
        kind: NodeKind::Function,
        name: "foo".to_string(),
        qualified_name: "crate::foo".to_string(),
        root_name: "myroot".to_string(),
        relative_path: "src/lib.rs".to_string(),
        language: "rust".to_string(),
        start_line: 10,
        end_line: 20,
        start_col: 0,
        end_col: 1,
        signature: Some("fn foo()".to_string()),
        doc: None,
        parse_reliable: true,
    };
    assert_eq!(n.name, "foo");
    assert_eq!(n.kind, NodeKind::Function);
}

#[test]
fn raw_edge_is_constructible() {
    let e = RawEdge {
        from_qid: "a".to_string(),
        to_qid: "b".to_string(),
        kind: EdgeKind::Contains,
    };
    assert_eq!(e.kind, EdgeKind::Contains);
}

#[test]
fn unresolved_ref_is_constructible() {
    let r = UnresolvedRef {
        qid: "rs:src/lib.rs:crate::foo:ref:42:7:1".to_string(),
        source_qid: "rs:src/lib.rs:crate::foo".to_string(),
        unresolved_name: "bar".to_string(),
        kind: UnresolvedKind::Call,
        line: 42,
        col: 7,
    };
    assert_eq!(r.kind, UnresolvedKind::Call);
}

#[test]
fn node_kind_round_trips_to_str() {
    use NodeKind::*;
    for k in [File, Function, Method, Struct, Enum, Trait, TypeAlias, Module, Import] {
        let s = k.as_label();
        assert!(!s.is_empty());
        assert!(s.chars().next().unwrap().is_ascii_uppercase());
    }
}
```

- [ ] **Step 2: Run tests — expect compile failures**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: errors about missing types in `codegraph_rs::extraction::result`.

- [ ] **Step 3: Implement the types**

Overwrite `src/extraction/result.rs`:

```rust
//! ExtractionResult and the raw node/edge/ref types produced by per-language extractors.
//!
//! These are the language-agnostic shape carried from extraction to the DB writer.
//! No serde derives — these are internal-only.

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NodeKind {
    File,
    Module,
    Function,
    Method,
    Struct,
    Enum,
    Trait,
    TypeAlias,
    Import,
}

impl NodeKind {
    /// The Neo4j label name (PascalCase, matches §4 of the spec).
    pub fn as_label(self) -> &'static str {
        match self {
            NodeKind::File => "File",
            NodeKind::Module => "Module",
            NodeKind::Function => "Function",
            NodeKind::Method => "Method",
            NodeKind::Struct => "Struct",
            NodeKind::Enum => "Enum",
            NodeKind::Trait => "Trait",
            NodeKind::TypeAlias => "TypeAlias",
            NodeKind::Import => "Import",
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EdgeKind {
    /// Lexical containment (File -> Function, Module -> Class, etc.)
    Contains,
    /// Symbol -> its UnresolvedRef records.
    HasRef,
}

impl EdgeKind {
    pub fn as_relationship_type(self) -> &'static str {
        match self {
            EdgeKind::Contains => "CONTAINS",
            EdgeKind::HasRef => "HAS_REF",
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum UnresolvedKind {
    Call,
    Import,
    TypeRef,
    Reference,
}

impl UnresolvedKind {
    pub fn as_str(self) -> &'static str {
        match self {
            UnresolvedKind::Call => "call",
            UnresolvedKind::Import => "import",
            UnresolvedKind::TypeRef => "type_ref",
            UnresolvedKind::Reference => "reference",
        }
    }
}

#[derive(Debug, Clone)]
pub struct RawNode {
    /// Stable identifier per spec §4: `<root>:<rel_path>:<qualified_name>[#<n>]`.
    pub qid: String,
    pub kind: NodeKind,
    pub name: String,
    pub qualified_name: String,
    pub root_name: String,
    pub relative_path: String,
    pub language: String,
    pub start_line: u32,
    pub end_line: u32,
    pub start_col: u32,
    pub end_col: u32,
    pub signature: Option<String>,
    pub doc: Option<String>,
    /// True for tree-sitter parses we trust; false for Daml template bodies
    /// extracted via tree-sitter-haskell (Slice 7+). For Rust always true.
    pub parse_reliable: bool,
}

#[derive(Debug, Clone)]
pub struct RawEdge {
    pub from_qid: String,
    pub to_qid: String,
    pub kind: EdgeKind,
}

#[derive(Debug, Clone)]
pub struct UnresolvedRef {
    /// Stable identifier per spec §7: `<source_qid>:ref:<line>:<col>:<ord>`.
    pub qid: String,
    pub source_qid: String,
    pub unresolved_name: String,
    pub kind: UnresolvedKind,
    pub line: u32,
    pub col: u32,
}

/// One file's extraction output.
#[derive(Debug, Clone, Default)]
pub struct ExtractionResult {
    pub nodes: Vec<RawNode>,
    pub edges: Vec<RawEdge>,
    pub unresolved_refs: Vec<UnresolvedRef>,
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/result.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add ExtractionResult types (RawNode, RawEdge, UnresolvedRef)"
```

---

### Task 3: File walker via `ignore` (TDD)

**Files:**
- Modify: `src/extraction/walker.rs`
- Test: append to `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
use codegraph_rs::extraction::walker::{walk_root, DiscoveredFile};
use std::fs;
use tempfile::tempdir;

#[test]
fn walker_discovers_rs_files() {
    let dir = tempdir().unwrap();
    fs::write(dir.path().join("a.rs"), "fn a() {}").unwrap();
    fs::write(dir.path().join("b.rs"), "fn b() {}").unwrap();
    fs::write(dir.path().join("c.txt"), "not a rust file").unwrap();

    let files: Vec<DiscoveredFile> = walk_root(dir.path(), "myroot", &["rust".to_string()]).collect();
    let names: Vec<&str> = files.iter().map(|f| f.relative_path.as_str()).collect();
    assert!(names.contains(&"a.rs"));
    assert!(names.contains(&"b.rs"));
    assert!(!names.iter().any(|n| n.ends_with(".txt")));
}

#[test]
fn walker_honors_gitignore() {
    let dir = tempdir().unwrap();
    fs::write(dir.path().join(".gitignore"), "ignored.rs\n").unwrap();
    fs::write(dir.path().join("kept.rs"), "fn k() {}").unwrap();
    fs::write(dir.path().join("ignored.rs"), "fn i() {}").unwrap();

    // ignore crate requires the dir to be a git repo for .gitignore to apply by default
    fs::create_dir(dir.path().join(".git")).unwrap();

    let files: Vec<DiscoveredFile> = walk_root(dir.path(), "myroot", &["rust".to_string()]).collect();
    let names: Vec<&str> = files.iter().map(|f| f.relative_path.as_str()).collect();
    assert!(names.contains(&"kept.rs"), "kept.rs should be discovered: {names:?}");
    assert!(!names.contains(&"ignored.rs"), "ignored.rs should be filtered: {names:?}");
}

#[test]
fn walker_recurses_subdirs_with_relative_paths() {
    let dir = tempdir().unwrap();
    fs::create_dir(dir.path().join("subdir")).unwrap();
    fs::write(dir.path().join("subdir/nested.rs"), "fn n() {}").unwrap();

    let files: Vec<DiscoveredFile> = walk_root(dir.path(), "myroot", &["rust".to_string()]).collect();
    let names: Vec<&str> = files.iter().map(|f| f.relative_path.as_str()).collect();
    assert!(names.iter().any(|n| n == &"subdir/nested.rs"),
        "should find nested.rs with subdir/ prefix: {names:?}");
}

#[test]
fn walker_filters_by_language() {
    let dir = tempdir().unwrap();
    fs::write(dir.path().join("a.rs"), "fn a() {}").unwrap();
    fs::write(dir.path().join("b.daml"), "module B").unwrap();

    // Only Rust enabled
    let rust_only: Vec<DiscoveredFile> = walk_root(dir.path(), "r", &["rust".to_string()]).collect();
    assert_eq!(rust_only.len(), 1);
    assert_eq!(rust_only[0].language, "rust");

    // Only Daml enabled (Slice 2 doesn't extract daml but the walker should still recognize it)
    let daml_only: Vec<DiscoveredFile> = walk_root(dir.path(), "r", &["daml".to_string()]).collect();
    assert_eq!(daml_only.len(), 1);
    assert_eq!(daml_only[0].language, "daml");
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: errors about missing `walk_root`, `DiscoveredFile`.

- [ ] **Step 3: Implement the walker**

Overwrite `src/extraction/walker.rs`:

```rust
//! File discovery via the `ignore` crate (the walker behind ripgrep).
//!
//! Honors `.gitignore`, `.ignore`, and a workspace-level `.codegraphignore`.

use ignore::WalkBuilder;
use std::path::{Path, PathBuf};

#[derive(Debug, Clone)]
pub struct DiscoveredFile {
    pub absolute_path: PathBuf,
    pub relative_path: String,
    pub root_name: String,
    pub language: String,
}

/// Walk `root_dir` recursively, yielding source files whose extension maps
/// to a language listed in `enabled_languages`.
///
/// The returned iterator is single-threaded; parallelism happens at the
/// per-file extraction stage in the orchestrator.
pub fn walk_root(
    root_dir: &Path,
    root_name: &str,
    enabled_languages: &[String],
) -> impl Iterator<Item = DiscoveredFile> {
    let root_dir = root_dir.to_path_buf();
    let root_name = root_name.to_string();
    let enabled: Vec<String> = enabled_languages.iter().map(|s| s.to_lowercase()).collect();

    let mut builder = WalkBuilder::new(&root_dir);
    builder.standard_filters(true).hidden(true).follow_links(false);
    // Add per-workspace ignore file if it exists (workspace root is the parent of `.codegraph/`,
    // not `root_dir` — but that path is owned by the orchestrator. Walker honors any
    // `.codegraphignore` inside `root_dir`).
    builder.add_custom_ignore_filename(".codegraphignore");

    builder.build().filter_map(move |result| {
        let entry = result.ok()?;
        if !entry.file_type().map(|t| t.is_file()).unwrap_or(false) {
            return None;
        }
        let absolute_path = entry.path().to_path_buf();
        let language = language_for_extension(&absolute_path)?;
        if !enabled.iter().any(|l| l == language) {
            return None;
        }
        let relative_path = absolute_path
            .strip_prefix(&root_dir)
            .ok()?
            .to_string_lossy()
            .to_string();
        Some(DiscoveredFile {
            absolute_path,
            relative_path,
            root_name: root_name.clone(),
            language: language.to_string(),
        })
    })
}

fn language_for_extension(path: &Path) -> Option<&'static str> {
    match path.extension()?.to_str()? {
        "rs" => Some("rust"),
        "daml" => Some("daml"),
        _ => None,
    }
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 9 tests pass total (5 from Task 2 + 4 walker tests).

- [ ] **Step 5: Commit**

```bash
git add src/extraction/walker.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add ignore-crate walker with extension/language filtering"
```

---

### Task 4: BLAKE3 content hashing (TDD)

**Files:**
- Modify: `src/extraction/hash.rs`
- Test: append to `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
use codegraph_rs::extraction::hash::{file_hash, source_hash};

#[test]
fn source_hash_is_deterministic() {
    let h1 = source_hash("hello world");
    let h2 = source_hash("hello world");
    assert_eq!(h1, h2);
    assert_eq!(h1.len(), 64); // BLAKE3 hex = 64 chars
}

#[test]
fn source_hash_differs_for_different_inputs() {
    assert_ne!(source_hash("a"), source_hash("b"));
}

#[test]
fn file_hash_matches_source_hash() {
    let dir = tempdir().unwrap();
    let path = dir.path().join("foo.rs");
    let content = "fn foo() {}";
    fs::write(&path, content).unwrap();

    assert_eq!(file_hash(&path).unwrap(), source_hash(content));
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: errors about missing `file_hash`, `source_hash`.

- [ ] **Step 3: Implement hashing**

Overwrite `src/extraction/hash.rs`:

```rust
//! BLAKE3 content hashing for sync diffing.

use anyhow::{Context, Result};
use std::path::Path;

/// Hex-encoded BLAKE3 hash of the given byte slice.
pub fn source_hash(content: &str) -> String {
    blake3::hash(content.as_bytes()).to_hex().to_string()
}

/// Hex-encoded BLAKE3 hash of a file's contents.
pub fn file_hash(path: &Path) -> Result<String> {
    let bytes = std::fs::read(path)
        .with_context(|| format!("reading `{}` for hashing", path.display()))?;
    Ok(blake3::hash(&bytes).to_hex().to_string())
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 12 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/hash.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add BLAKE3 file/source hashing"
```

---

### Task 5: Tree-sitter parser plumbing (per-thread)

**Files:**
- Modify: `src/extraction/parser.rs`
- Test: append to `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
use codegraph_rs::extraction::parser::parse_rust;

#[test]
fn parser_returns_tree_for_valid_rust() {
    let source = "fn foo() {}";
    let tree = parse_rust(source).expect("valid Rust must parse");
    let root = tree.root_node();
    assert_eq!(root.kind(), "source_file");
    // Either no errors, or at most ERROR nodes within the tree (we don't fail on those).
    assert!(!root.has_error() || root.named_child_count() > 0);
}

#[test]
fn parser_returns_tree_for_partial_rust_with_error_nodes() {
    // Even broken source returns a tree (with ERROR nodes inside).
    let source = "fn foo( {";
    let tree = parse_rust(source);
    assert!(tree.is_some());
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: errors about missing `parse_rust`.

- [ ] **Step 3: Implement parser plumbing**

Overwrite `src/extraction/parser.rs`:

```rust
//! Per-thread tree-sitter parsers.
//!
//! `tree_sitter::Parser` is `Send` but not `Sync`, so we use `thread_local!`
//! to give each rayon worker its own reusable parser. Avoids the cost of
//! reconstructing the parser per file.

use std::cell::RefCell;
use tree_sitter::{Parser, Tree};

thread_local! {
    static RUST_PARSER: RefCell<Parser> = RefCell::new(make_parser_for(tree_sitter_rust::LANGUAGE.into()));
}

fn make_parser_for(language: tree_sitter::Language) -> Parser {
    let mut p = Parser::new();
    p.set_language(&language).expect("valid grammar");
    p
}

/// Parse a Rust source string using this thread's pooled parser.
/// Returns None only if tree-sitter itself can't allocate a tree.
pub fn parse_rust(source: &str) -> Option<Tree> {
    RUST_PARSER.with(|cell| cell.borrow_mut().parse(source, None))
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 14 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/parser.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add per-thread tree-sitter parser pool"
```

---

### Task 6: Rust extractor — `function_item` → `Function` (TDD)

**Files:**
- Modify: `src/extraction/languages/rust.rs`
- Modify: `src/extraction/languages/rust.scm`
- Test: append to `tests/extraction_unit.rs`

This task introduces the `LanguageExtractor` trait, the Rust impl, and tree-sitter capture queries — but limited to `function_item` only. Tasks 7–9 extend the surface.

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
use codegraph_rs::extraction::languages::rust::extract_rust;
use codegraph_rs::extraction::result::NodeKind;

#[test]
fn extracts_top_level_function() {
    let source = r#"fn foo() { let x = 1; }"#;
    let result = extract_rust(source, "myroot", "src/lib.rs").expect("extraction must succeed");

    let funcs: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Function).collect();
    assert_eq!(funcs.len(), 1, "expected 1 Function, got {}", funcs.len());

    let f = &funcs[0];
    assert_eq!(f.name, "foo");
    assert!(f.qualified_name.contains("foo"));
    assert_eq!(f.relative_path, "src/lib.rs");
    assert_eq!(f.root_name, "myroot");
    assert_eq!(f.language, "rust");
    assert_eq!(f.start_line, 1);
    assert!(f.parse_reliable);
}

#[test]
fn extracts_multiple_functions_in_order() {
    let source = "fn a() {}\nfn b() {}\nfn c() {}\n";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let funcs: Vec<&str> = result.nodes.iter()
        .filter(|n| n.kind == NodeKind::Function)
        .map(|n| n.name.as_str())
        .collect();
    assert_eq!(funcs, vec!["a", "b", "c"]);
}

#[test]
fn extraction_emits_file_node() {
    let source = "fn x() {}";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let files: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::File).collect();
    assert_eq!(files.len(), 1);
    assert_eq!(files[0].name, "f.rs");
    assert_eq!(files[0].relative_path, "f.rs");
}

#[test]
fn extraction_emits_contains_edge_from_file_to_function() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "fn y() {}";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let contains: Vec<_> = result.edges.iter().filter(|e| e.kind == EdgeKind::Contains).collect();
    assert!(!contains.is_empty(), "expected at least one CONTAINS edge");
    // Every Function in result.nodes should have a CONTAINS edge from the File.
    let file_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::File).unwrap().qid;
    let func_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Function).unwrap().qid;
    assert!(contains.iter().any(|e| e.from_qid == *file_qid && e.to_qid == *func_qid));
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: errors about missing `extract_rust`.

- [ ] **Step 3: Write the tree-sitter query for functions**

Overwrite `src/extraction/languages/rust.scm`:

```scheme
; Top-level function items.
(function_item
  name: (identifier) @function.name) @function.def
```

- [ ] **Step 4: Implement the Rust extractor**

Overwrite `src/extraction/languages/rust.rs`:

```rust
//! Rust language extractor.
//!
//! Uses tree-sitter-rust + a `.scm` capture query to locate interesting AST nodes
//! declaratively. Tasks 6–9 incrementally grow the supported node kinds.

use anyhow::{Context, Result};
use tree_sitter::{Query, QueryCursor, StreamingIterator};

use crate::extraction::parser::parse_rust;
use crate::extraction::result::{
    EdgeKind, ExtractionResult, NodeKind, RawEdge, RawNode,
};

const QUERY_SOURCE: &str = include_str!("rust.scm");

/// Extract a single Rust file. Always emits one File node + zero or more
/// per-symbol nodes + the structural CONTAINS edges connecting them.
pub fn extract_rust(source: &str, root_name: &str, relative_path: &str) -> Result<ExtractionResult> {
    let tree = parse_rust(source).context("tree-sitter-rust failed to produce a tree")?;

    let language = tree_sitter_rust::LANGUAGE;
    let query = Query::new(&language.into(), QUERY_SOURCE)
        .context("invalid tree-sitter-rust query")?;
    let function_def_idx = query.capture_index_for_name("function.def")
        .context("query missing capture `function.def`")? as u32;
    let function_name_idx = query.capture_index_for_name("function.name")
        .context("query missing capture `function.name`")? as u32;

    let mut result = ExtractionResult::default();
    let file_node = build_file_node(source, root_name, relative_path);
    let file_qid = file_node.qid.clone();
    result.nodes.push(file_node);

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(&query, tree.root_node(), source.as_bytes());

    let mut function_ordinals: std::collections::HashMap<String, u32> = Default::default();

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == function_def_idx { def_node = Some(cap.node); }
            else if cap.index == function_name_idx { name_node = Some(cap.node); }
        }
        if let (Some(def), Some(name)) = (def_node, name_node) {
            let func_name = name.utf8_text(source.as_bytes()).unwrap_or("?").to_string();
            let qualified_name = format!("{}::{}", path_to_module(relative_path), func_name);
            let key = format!("{root_name}:{relative_path}:{qualified_name}");
            let ord = function_ordinals.entry(key.clone()).and_modify(|n| *n += 1).or_insert(0);
            let qid = if *ord == 0 { key.clone() } else { format!("{key}#{ord}") };

            let start = def.start_position();
            let end = def.end_position();
            let signature = signature_text(source, def);

            result.nodes.push(RawNode {
                qid: qid.clone(),
                kind: NodeKind::Function,
                name: func_name,
                qualified_name,
                root_name: root_name.to_string(),
                relative_path: relative_path.to_string(),
                language: "rust".to_string(),
                start_line: (start.row as u32) + 1,
                end_line: (end.row as u32) + 1,
                start_col: start.column as u32,
                end_col: end.column as u32,
                signature: Some(signature),
                doc: None,
                parse_reliable: true,
            });
            result.edges.push(RawEdge {
                from_qid: file_qid.clone(),
                to_qid: qid,
                kind: EdgeKind::Contains,
            });
        }
    }

    Ok(result)
}

fn build_file_node(source: &str, root_name: &str, relative_path: &str) -> RawNode {
    let qid = format!("{root_name}:{relative_path}");
    let name = std::path::Path::new(relative_path)
        .file_name()
        .map(|s| s.to_string_lossy().to_string())
        .unwrap_or_else(|| relative_path.to_string());
    let line_count = source.lines().count() as u32;
    RawNode {
        qid,
        kind: NodeKind::File,
        name,
        qualified_name: relative_path.to_string(),
        root_name: root_name.to_string(),
        relative_path: relative_path.to_string(),
        language: "rust".to_string(),
        start_line: 1,
        end_line: line_count.max(1),
        start_col: 0,
        end_col: 0,
        signature: None,
        doc: None,
        parse_reliable: true,
    }
}

/// Convert `src/util/foo.rs` into a Rust-ish module path `src::util::foo`.
/// Slice 2 doesn't read `Cargo.toml` to derive the crate root, so this is a
/// best-effort qualified-name prefix; resolution (Slice 3) treats names by
/// uniqueness within the workspace.
fn path_to_module(relative_path: &str) -> String {
    let p = std::path::Path::new(relative_path);
    let stem: Vec<&str> = p
        .with_extension("")
        .components()
        .map(|c| c.as_os_str().to_str().unwrap_or(""))
        .collect::<Vec<_>>()
        .into_iter()
        .filter(|s| !s.is_empty())
        .collect();
    stem.join("::")
}

fn signature_text(source: &str, node: tree_sitter::Node) -> String {
    // The "signature" for a function is everything before the body (`{ ... }`).
    // tree-sitter-rust exposes the body as a child by field name "body".
    let mut tail = node.start_byte();
    if let Some(body) = node.child_by_field_name("body") {
        tail = body.start_byte();
    } else {
        tail = node.end_byte();
    }
    let bytes = &source.as_bytes()[node.start_byte()..tail];
    std::str::from_utf8(bytes).unwrap_or("").trim().to_string()
}
```

> **Note on the `Path::components` chain:** `with_extension("")` returns an owned `PathBuf`. The chain takes its components, maps to `&str`, collects into a `Vec<&str>`, then converts back to an iterator and filters. The double-collect is needed because `with_extension` returns a temporary that's destroyed at the end of the expression — but the outer `let stem: Vec<&str>` keeps the strings alive via `to_str()` borrowing from `as_os_str()` of components owned by the final Vec. If the borrow checker complains, refactor to: collect once into `Vec<String>` to avoid lifetime issues.

- [ ] **Step 5: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 18 tests pass (14 + 4 new).

- [ ] **Step 6: Commit**

```bash
git add src/extraction/languages/rust.rs src/extraction/languages/rust.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Rust extractor — function_item -> Function with CONTAINS"
```

---

### Task 7: Rust extractor — extend to struct/enum/trait/type/mod (TDD)

**Files:**
- Modify: `src/extraction/languages/rust.rs`
- Modify: `src/extraction/languages/rust.scm`
- Test: append to `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
#[test]
fn extracts_struct() {
    let source = "pub struct Foo { x: u32 }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let structs: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Struct).collect();
    assert_eq!(structs.len(), 1);
    assert_eq!(structs[0].name, "Foo");
}

#[test]
fn extracts_enum() {
    let source = "enum Color { Red, Green, Blue }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let enums: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Enum).collect();
    assert_eq!(enums.len(), 1);
    assert_eq!(enums[0].name, "Color");
}

#[test]
fn extracts_trait() {
    let source = "trait Greeter { fn hi(&self); }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let traits: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Trait).collect();
    assert_eq!(traits.len(), 1);
    assert_eq!(traits[0].name, "Greeter");
}

#[test]
fn extracts_type_alias() {
    let source = "type Bar = u64;";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let types: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::TypeAlias).collect();
    assert_eq!(types.len(), 1);
    assert_eq!(types[0].name, "Bar");
}

#[test]
fn extracts_inner_module_declaration() {
    let source = "mod nested { fn inside() {} }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let modules: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Module).collect();
    assert_eq!(modules.len(), 1);
    assert_eq!(modules[0].name, "nested");
}
```

- [ ] **Step 2: Run failing tests**

Expected: 5 new tests fail.

- [ ] **Step 3: Extend the query and extractor**

Append to `src/extraction/languages/rust.scm`:

```scheme
; Type-like definitions.
(struct_item
  name: (type_identifier) @struct.name) @struct.def

(enum_item
  name: (type_identifier) @enum.name) @enum.def

(trait_item
  name: (type_identifier) @trait.name) @trait.def

(type_item
  name: (type_identifier) @type.name) @type.def

; Inner module declarations (mod foo { ... } or mod foo;)
(mod_item
  name: (identifier) @module.name) @module.def
```

In `src/extraction/languages/rust.rs`, refactor `extract_rust` to handle multiple capture pairs. Replace the function body with a generalized loop. Specifically, after the file node insertion, replace the existing `while let Some(m) = matches.next() { ... }` block with:

```rust
    // Capture-pair table: (def_capture, name_capture, NodeKind)
    let captures: &[(&str, &str, NodeKind)] = &[
        ("function.def", "function.name", NodeKind::Function),
        ("struct.def", "struct.name", NodeKind::Struct),
        ("enum.def", "enum.name", NodeKind::Enum),
        ("trait.def", "trait.name", NodeKind::Trait),
        ("type.def", "type.name", NodeKind::TypeAlias),
        ("module.def", "module.name", NodeKind::Module),
    ];
    let resolved: Vec<(u32, u32, NodeKind)> = captures.iter()
        .map(|(def, name, kind)| {
            let d = query.capture_index_for_name(def)
                .with_context(|| format!("missing capture `{def}`"))? as u32;
            let n = query.capture_index_for_name(name)
                .with_context(|| format!("missing capture `{name}`"))? as u32;
            Ok::<_, anyhow::Error>((d, n, *kind))
        })
        .collect::<Result<_>>()?;

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(&query, tree.root_node(), source.as_bytes());

    let mut ordinals: std::collections::HashMap<String, u32> = Default::default();

    while let Some(m) = matches.next() {
        for (def_idx, name_idx, kind) in &resolved {
            let mut def_node = None;
            let mut name_node = None;
            for cap in m.captures {
                if cap.index == *def_idx { def_node = Some(cap.node); }
                else if cap.index == *name_idx { name_node = Some(cap.node); }
            }
            if let (Some(def), Some(name)) = (def_node, name_node) {
                let item_name = name.utf8_text(source.as_bytes()).unwrap_or("?").to_string();
                let qualified_name = format!("{}::{}", path_to_module(relative_path), item_name);
                let key = format!("{root_name}:{relative_path}:{qualified_name}");
                let ord = ordinals.entry(key.clone())
                    .and_modify(|n| *n += 1).or_insert(0);
                let qid = if *ord == 0 { key.clone() } else { format!("{key}#{ord}") };

                let start = def.start_position();
                let end = def.end_position();
                let signature = match kind {
                    NodeKind::Function | NodeKind::Method => Some(signature_text(source, def)),
                    _ => None,
                };

                result.nodes.push(RawNode {
                    qid: qid.clone(),
                    kind: *kind,
                    name: item_name,
                    qualified_name,
                    root_name: root_name.to_string(),
                    relative_path: relative_path.to_string(),
                    language: "rust".to_string(),
                    start_line: (start.row as u32) + 1,
                    end_line: (end.row as u32) + 1,
                    start_col: start.column as u32,
                    end_col: end.column as u32,
                    signature,
                    doc: None,
                    parse_reliable: true,
                });
                result.edges.push(RawEdge {
                    from_qid: file_qid.clone(),
                    to_qid: qid,
                    kind: EdgeKind::Contains,
                });
                break; // a tree-sitter match has at most one (def, name) pair we care about
            }
        }
    }
```

Remove the now-stale `function_ordinals` variable, `function_def_idx`, and `function_name_idx`.

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 23 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/languages/rust.rs src/extraction/languages/rust.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Rust extractor — struct/enum/trait/type/mod"
```

---

### Task 8: Rust extractor — `use_declaration` → `Import` (TDD)

**Files:**
- Modify: `src/extraction/languages/rust.rs`
- Modify: `src/extraction/languages/rust.scm`
- Test: append to `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
#[test]
fn extracts_simple_use_as_import() {
    let source = "use std::collections::HashMap;\nfn main() {}";
    let result = extract_rust(source, "r", "main.rs").expect("ok");
    let imports: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Import).collect();
    assert_eq!(imports.len(), 1);
    // The Import node's name carries the full use path.
    assert!(imports[0].name.contains("HashMap") || imports[0].name.contains("collections"),
        "unexpected import name: {}", imports[0].name);
}

#[test]
fn extracts_multiple_use_declarations() {
    let source = "use a::b;\nuse c::d;\n";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    let imports: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Import).collect();
    assert_eq!(imports.len(), 2);
}
```

- [ ] **Step 2: Append to the .scm query**

Append to `src/extraction/languages/rust.scm`:

```scheme
; `use` declarations -> Import nodes.
(use_declaration
  argument: (_) @import.path) @import.def
```

(`(_)` matches any kind of argument node — `scoped_identifier`, `use_list`, `use_as_clause`, etc.)

- [ ] **Step 3: Update the captures table**

In `src/extraction/languages/rust.rs`, add to the `captures` slice:

```rust
        ("import.def", "import.path", NodeKind::Import),
```

Then handle the case that Import has a path-text-as-name rather than an identifier-token. The current code does `name.utf8_text(...)` which returns the full subtree text — that's fine for Import too. So no further code changes are required.

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 25 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/languages/rust.rs src/extraction/languages/rust.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Rust extractor — use_declaration -> Import"
```

---

### Task 9: Rust extractor — `call_expression` → `UnresolvedRef` (TDD)

**Files:**
- Modify: `src/extraction/languages/rust.rs`
- Modify: `src/extraction/languages/rust.scm`
- Test: append to `tests/extraction_unit.rs`

This task adds `UnresolvedRef` records for call sites and the structural `HAS_REF` edge from the calling Symbol to the UnresolvedRef.

- [ ] **Step 1: Append failing tests**

Append to `tests/extraction_unit.rs`:

```rust
use codegraph_rs::extraction::result::UnresolvedKind;

#[test]
fn extracts_call_as_unresolved_ref() {
    let source = "fn caller() { foo(); }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");

    assert_eq!(result.unresolved_refs.len(), 1, "expected 1 UnresolvedRef");
    let r = &result.unresolved_refs[0];
    assert_eq!(r.kind, UnresolvedKind::Call);
    assert_eq!(r.unresolved_name, "foo");
    // source_qid should be the caller function
    assert!(r.source_qid.contains("caller"), "source_qid: {}", r.source_qid);
}

#[test]
fn calls_inside_unknown_scope_attach_to_file() {
    // A call at module top level (rare, but possible in macros / tests) should
    // still produce an UnresolvedRef. Source-qid falls back to the file qid.
    let source = "const X: i32 = bar();";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    assert_eq!(result.unresolved_refs.len(), 1);
    assert_eq!(result.unresolved_refs[0].unresolved_name, "bar");
}

#[test]
fn unresolved_ref_has_ref_edge_from_source() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "fn caller() { foo(); }";
    let result = extract_rust(source, "r", "f.rs").expect("ok");
    assert_eq!(result.unresolved_refs.len(), 1);

    let ref_qid = &result.unresolved_refs[0].qid;
    let source_qid = &result.unresolved_refs[0].source_qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::HasRef && e.from_qid == *source_qid && e.to_qid == *ref_qid),
        "expected HAS_REF edge from {source_qid} to {ref_qid}");
}
```

- [ ] **Step 2: Append to the .scm query**

Append to `src/extraction/languages/rust.scm`:

```scheme
; Function calls -> UnresolvedRef of kind=Call.
(call_expression
  function: (_) @call.callee) @call.site
```

- [ ] **Step 3: Implement call extraction**

The logic is different from definition extraction (calls don't define a Symbol; they reference one). Add a separate query+iteration pass after the existing definition extraction.

In `src/extraction/languages/rust.rs`, after the existing `while let Some(m) = matches.next() { ... }` loop and before `Ok(result)`, add:

```rust
    // Pass 2: call expressions -> UnresolvedRef with HAS_REF edge.
    let call_callee_idx = query.capture_index_for_name("call.callee")
        .context("missing capture `call.callee`")? as u32;
    let call_site_idx = query.capture_index_for_name("call.site")
        .context("missing capture `call.site`")? as u32;

    let mut cursor2 = QueryCursor::new();
    let mut call_matches = cursor2.matches(&query, tree.root_node(), source.as_bytes());
    let mut ref_ord: std::collections::HashMap<String, u32> = Default::default();

    while let Some(m) = call_matches.next() {
        let mut callee = None;
        let mut site = None;
        for cap in m.captures {
            if cap.index == call_callee_idx { callee = Some(cap.node); }
            else if cap.index == call_site_idx { site = Some(cap.node); }
        }
        if let (Some(callee), Some(site)) = (callee, site) {
            let name_text = callee.utf8_text(source.as_bytes()).unwrap_or("?").to_string();
            // The "leaf" name is the last segment of a possibly-qualified call.
            let leaf = name_text.split("::").last().unwrap_or(&name_text)
                .trim_end_matches(|c: char| !c.is_alphanumeric() && c != '_').to_string();
            let pos = site.start_position();
            let line = (pos.row as u32) + 1;
            let col = pos.column as u32;
            let source_qid = enclosing_symbol_qid(&result.nodes, line, col).unwrap_or(&file_qid).clone();
            let key = format!("{source_qid}:ref:{line}:{col}");
            let ord = ref_ord.entry(key.clone()).and_modify(|n| *n += 1).or_insert(0);
            let qid = format!("{key}:{ord}");

            result.unresolved_refs.push(crate::extraction::result::UnresolvedRef {
                qid: qid.clone(),
                source_qid: source_qid.clone(),
                unresolved_name: leaf,
                kind: crate::extraction::result::UnresolvedKind::Call,
                line,
                col,
            });
            result.edges.push(RawEdge {
                from_qid: source_qid,
                to_qid: qid,
                kind: EdgeKind::HasRef,
            });
        }
    }
```

Then add this helper at the bottom of `rust.rs`:

```rust
/// Find the smallest Symbol node (Function/Method/etc.) in `nodes` whose
/// span contains the given (line, col). Used to attach a call site to its
/// enclosing function as the source of the UnresolvedRef.
fn enclosing_symbol_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&String> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        if matches!(n.kind, NodeKind::File) { continue; }
        let in_span = (n.start_line < line || (n.start_line == line && n.start_col <= col))
            && (n.end_line > line || (n.end_line == line && n.end_col >= col));
        if !in_span { continue; }
        match best {
            None => best = Some(n),
            Some(b) => {
                let n_lines = n.end_line.saturating_sub(n.start_line);
                let b_lines = b.end_line.saturating_sub(b.start_line);
                if n_lines < b_lines { best = Some(n); }
            }
        }
    }
    best.map(|n| &n.qid)
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 28 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/languages/rust.rs src/extraction/languages/rust.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Rust extractor — call_expression -> UnresolvedRef + HAS_REF"
```

---

### Task 10: `db::write_extraction` — UNWIND/MERGE batches

**Files:**
- Modify: `src/db/write.rs`
- Test: append `tests/integration_index.rs` (gated, but still creates the test fixture). The actual run-against-Neo4j happens in Task 12.

This task implements the batch writer. The integration test is added in Task 12 (which also creates the `tiny-rust` fixture). For now, write only unit-friendly code; verify by reading.

- [ ] **Step 1: Implement `write_extraction`**

Overwrite `src/db/write.rs`:

```rust
//! Persist extraction results to Neo4j via batched UNWIND/MERGE statements.
//!
//! Strategy: collect all nodes/edges/refs across all files, group by kind
//! (label / relationship type), and issue one UNWIND batch per group of ~500.

use anyhow::{Context, Result};
use neo4rs::query as cypher;
use serde_json::{Map, Value};
use std::collections::HashMap;

use crate::db::Connection;
use crate::extraction::result::{EdgeKind, NodeKind, RawEdge, RawNode, UnresolvedKind, UnresolvedRef};
use crate::extraction::ExtractionResult;

const BATCH_SIZE: usize = 500;

pub async fn write_extraction(conn: &Connection, results: Vec<ExtractionResult>) -> Result<()> {
    // Aggregate.
    let mut nodes_by_kind: HashMap<NodeKind, Vec<RawNode>> = HashMap::new();
    let mut edges_by_kind: HashMap<EdgeKind, Vec<RawEdge>> = HashMap::new();
    let mut refs: Vec<UnresolvedRef> = Vec::new();

    for r in results {
        for n in r.nodes { nodes_by_kind.entry(n.kind).or_default().push(n); }
        for e in r.edges { edges_by_kind.entry(e.kind).or_default().push(e); }
        refs.extend(r.unresolved_refs);
    }

    // Write nodes (Symbol + specific label for each).
    for (kind, nodes) in nodes_by_kind {
        write_nodes(conn, kind, &nodes).await
            .with_context(|| format!("writing {} {:?} nodes", nodes.len(), kind))?;
    }

    // Write UnresolvedRef nodes.
    write_unresolved_refs(conn, &refs).await.context("writing UnresolvedRef nodes")?;

    // Write edges (CONTAINS and HAS_REF).
    for (kind, edges) in edges_by_kind {
        write_edges(conn, kind, &edges).await
            .with_context(|| format!("writing {} {:?} edges", edges.len(), kind))?;
    }

    Ok(())
}

async fn write_nodes(conn: &Connection, kind: NodeKind, nodes: &[RawNode]) -> Result<()> {
    let label = kind.as_label();
    let cypher_text = format!(
        "UNWIND $batch AS row \
         MERGE (n:Symbol:{label} {{qid: row.qid}}) \
         SET n.name = row.name, \
             n.qualified_name = row.qualified_name, \
             n.root_name = row.root_name, \
             n.relative_path = row.relative_path, \
             n.language = row.language, \
             n.start_line = row.start_line, \
             n.end_line = row.end_line, \
             n.start_col = row.start_col, \
             n.end_col = row.end_col, \
             n.signature = row.signature, \
             n.doc = row.doc, \
             n.parse_reliable = row.parse_reliable"
    );

    for chunk in nodes.chunks(BATCH_SIZE) {
        let batch: Vec<Value> = chunk.iter().map(node_to_json).collect();
        conn.graph
            .execute(cypher(&cypher_text).param("batch", Value::Array(batch)))
            .await?
            .next()
            .await
            .ok();
    }
    Ok(())
}

async fn write_unresolved_refs(conn: &Connection, refs: &[UnresolvedRef]) -> Result<()> {
    let cypher_text = "\
        UNWIND $batch AS row \
        MERGE (n:UnresolvedRef {qid: row.qid}) \
        SET n.source_qid = row.source_qid, \
            n.unresolved_name = row.unresolved_name, \
            n.kind = row.kind, \
            n.line = row.line, \
            n.col = row.col";

    for chunk in refs.chunks(BATCH_SIZE) {
        let batch: Vec<Value> = chunk.iter().map(ref_to_json).collect();
        conn.graph
            .execute(cypher(cypher_text).param("batch", Value::Array(batch)))
            .await?
            .next()
            .await
            .ok();
    }
    Ok(())
}

async fn write_edges(conn: &Connection, kind: EdgeKind, edges: &[RawEdge]) -> Result<()> {
    let rel = kind.as_relationship_type();
    // CONTAINS connects File/Module/Symbol to a Symbol; HAS_REF connects Symbol to UnresolvedRef.
    // For the v1 schema we MATCH any node by qid (using the qid uniqueness constraint).
    let cypher_text = format!(
        "UNWIND $batch AS row \
         MATCH (a {{qid: row.from_qid}}) \
         MATCH (b {{qid: row.to_qid}}) \
         MERGE (a)-[:{rel}]->(b)"
    );

    for chunk in edges.chunks(BATCH_SIZE) {
        let batch: Vec<Value> = chunk.iter().map(edge_to_json).collect();
        conn.graph
            .execute(cypher(&cypher_text).param("batch", Value::Array(batch)))
            .await?
            .next()
            .await
            .ok();
    }
    Ok(())
}

fn node_to_json(n: &RawNode) -> Value {
    let mut m = Map::new();
    m.insert("qid".into(), n.qid.clone().into());
    m.insert("name".into(), n.name.clone().into());
    m.insert("qualified_name".into(), n.qualified_name.clone().into());
    m.insert("root_name".into(), n.root_name.clone().into());
    m.insert("relative_path".into(), n.relative_path.clone().into());
    m.insert("language".into(), n.language.clone().into());
    m.insert("start_line".into(), n.start_line.into());
    m.insert("end_line".into(), n.end_line.into());
    m.insert("start_col".into(), n.start_col.into());
    m.insert("end_col".into(), n.end_col.into());
    m.insert("signature".into(), n.signature.clone().map(Value::String).unwrap_or(Value::Null));
    m.insert("doc".into(), n.doc.clone().map(Value::String).unwrap_or(Value::Null));
    m.insert("parse_reliable".into(), n.parse_reliable.into());
    Value::Object(m)
}

fn ref_to_json(r: &UnresolvedRef) -> Value {
    let mut m = Map::new();
    m.insert("qid".into(), r.qid.clone().into());
    m.insert("source_qid".into(), r.source_qid.clone().into());
    m.insert("unresolved_name".into(), r.unresolved_name.clone().into());
    m.insert("kind".into(), r.kind.as_str().into());
    m.insert("line".into(), r.line.into());
    m.insert("col".into(), r.col.into());
    Value::Object(m)
}

fn edge_to_json(e: &RawEdge) -> Value {
    let mut m = Map::new();
    m.insert("from_qid".into(), e.from_qid.clone().into());
    m.insert("to_qid".into(), e.to_qid.clone().into());
    Value::Object(m)
}
```

> **Note on `UnresolvedKind`:** `UnresolvedKind` was used here in the import line (`use crate::extraction::result::{... UnresolvedKind ...}`). The `as_str()` method called on `r.kind` is what the writer needs. If your `result.rs` from Task 2 doesn't have `UnresolvedKind::as_str()`, add it. The Task 2 spec includes it; verify before continuing.

- [ ] **Step 2: Add serde_json dependency**

`serde_json` isn't in slice 1's Cargo.toml. Add it now since `neo4rs` 0.8 takes parameters as `serde_json::Value` (or via `BoltType`; use serde_json for ergonomics).

In `/Users/gyorgybalazsi/codegraph-rs/Cargo.toml`, add to `[dependencies]`:

```toml
serde_json = "1"
```

> Implementer note: if `neo4rs::query(...).param(...)` does NOT accept `serde_json::Value` directly, switch to `neo4rs::BoltType` construction. The neo4rs 0.8 README shows `query(...).param("k", value)` where `value: impl Into<BoltType>`. `BoltType: From<serde_json::Value>` exists; if not, adapt with `BoltType::Map`/`BoltType::List`/etc. by hand. Validate via the Task 12 integration test.

- [ ] **Step 3: Build and verify**

```bash
cargo build 2>&1 | tail -5
```

Expected: clean. If neo4rs's `param` doesn't accept `serde_json::Value` directly, you'll see compile errors — adjust per the note above.

- [ ] **Step 4: Run all tests (no integration_index yet — added in Task 12)**

```bash
cargo test 2>&1 | tail -10
```

Expected: existing tests still pass; no new tests added in this task.

- [ ] **Step 5: Commit**

```bash
git add src/db/write.rs Cargo.toml Cargo.lock
git commit -m "feat(db): write_extraction — batched UNWIND/MERGE for nodes, edges, refs"
```

---

### Task 11: Wire up `codegraph-rs index` command

**Files:**
- Create: `src/cli/index.rs`
- Modify: `src/cli/mod.rs`
- Modify: `src/extraction/orchestrator.rs`

- [ ] **Step 1: Implement the orchestrator**

Overwrite `src/extraction/orchestrator.rs`:

```rust
//! Ties walker + parser + extractor together for a workspace, returning
//! a list of ExtractionResult — one per file. Parallel via rayon.

use anyhow::{Context, Result};
use rayon::prelude::*;
use std::path::Path;
use tracing::warn;

use crate::config::workspace::WorkspaceConfig;
use crate::extraction::languages::rust::extract_rust;
use crate::extraction::result::ExtractionResult;
use crate::extraction::walker::walk_root;

pub fn run_extraction(_workspace_dir: &Path, cfg: &WorkspaceConfig) -> Result<Vec<ExtractionResult>> {
    let languages = &cfg.indexing.languages;

    let files: Vec<_> = cfg.workspace.roots.iter()
        .flat_map(|root| walk_root(&root.path, &root.name, languages))
        .collect();

    let results: Vec<Result<ExtractionResult>> = files.par_iter().map(|f| {
        let source = std::fs::read_to_string(&f.absolute_path)
            .with_context(|| format!("reading {}", f.absolute_path.display()))?;
        match f.language.as_str() {
            "rust" => extract_rust(&source, &f.root_name, &f.relative_path),
            "daml" => Ok(ExtractionResult::default()), // Slice 7 fills in
            other => {
                warn!("no extractor for language `{other}`, skipping {}", f.relative_path);
                Ok(ExtractionResult::default())
            }
        }
    }).collect();

    let mut ok = Vec::with_capacity(results.len());
    for (idx, r) in results.into_iter().enumerate() {
        match r {
            Ok(er) => ok.push(er),
            Err(e) => warn!("extraction failed for file {idx}: {e:#}"),
        }
    }
    Ok(ok)
}
```

- [ ] **Step 2: Implement the index CLI command**

Create `src/cli/index.rs`:

```rust
//! `codegraph-rs index` — full-workspace extraction + write.

use anyhow::{Context, Result};
use std::path::Path;
use tracing::info;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::{schema::apply_schema, write::write_extraction, Connection};
use crate::extraction::orchestrator::run_extraction;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg.map(Path::to_path_buf).unwrap_or_else(|| {
        std::env::current_dir().expect("cwd readable")
    });
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

    info!("extracting workspace `{}`", cfg.workspace.name);
    let results = run_extraction(&workspace_dir, &cfg).context("extraction failed")?;

    // Capture stats BEFORE moving `results` into write_extraction.
    let file_count = results.len();
    let total_nodes: usize = results.iter().map(|r| r.nodes.len()).sum();
    let total_edges: usize = results.iter().map(|r| r.edges.len()).sum();
    let total_refs: usize = results.iter().map(|r| r.unresolved_refs.len()).sum();
    info!("extracted {total_nodes} nodes, {total_edges} edges, {total_refs} unresolved refs from {file_count} files");

    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        ).await.context("connecting to workspace database")?;

        // Schema is idempotent; ensure it's there before writing.
        apply_schema(&conn).await.context("applying schema")?;

        write_extraction(&conn, results).await.context("writing extraction to Neo4j")?;
        Ok::<(), anyhow::Error>(())
    })?;

    println!("Indexed {file_count} files: {total_nodes} nodes, {total_edges} edges, {total_refs} unresolved refs");
    Ok(())
}
```

- [ ] **Step 3: Wire dispatch**

In `src/cli/mod.rs`, add `pub mod index;` near the existing `pub mod init;` and `pub mod status;`. Update the dispatch match arm:

```rust
        Command::Index => index::run(cli.workspace.as_deref()),
```

Replace the existing combined arm:
```rust
        Command::Index | Command::Sync | Command::Query { .. } | Command::Serve => {
            anyhow::bail!("not yet implemented in slice 1");
        }
```
With:
```rust
        Command::Index => index::run(cli.workspace.as_deref()),
        Command::Sync | Command::Query { .. } | Command::Serve => {
            anyhow::bail!("not yet implemented");
        }
```

- [ ] **Step 4: Build and run tests**

```bash
cargo build 2>&1 | tail -3
cargo test 2>&1 | tail -10
```

Expected: build clean. The `cli_smoke` test for `index` previously expected "not yet implemented" — but `index` now runs the real command (which will fail without a workspace, with a different error). Update the test if needed.

In `tests/cli_smoke.rs`, the test `slice1_stub_commands_fail_with_not_implemented` iterates over `["index", "sync", "serve"]`. Remove `"index"` from that list (since it's now implemented and will produce a different error message):

```rust
    for sub in &["sync", "serve"] {
```

- [ ] **Step 5: Commit**

```bash
git add src/cli src/extraction/orchestrator.rs tests/cli_smoke.rs
git commit -m "feat(cli): wire up index command — orchestrate walk + parse + extract + write"
```

---

### Task 12: Integration test against tiny-rust fixture

**Files:**
- Create: `tests/fixtures/tiny-rust/Cargo.toml`
- Create: `tests/fixtures/tiny-rust/src/main.rs`
- Create: `tests/fixtures/tiny-rust/src/util.rs`
- Create: `tests/fixtures/tiny-rust/src/lib.rs`
- Create: `tests/integration_index.rs` (gated)

- [ ] **Step 1: Create the fixture**

Create `tests/fixtures/tiny-rust/Cargo.toml`:

```toml
[package]
name = "tiny-rust"
version = "0.1.0"
edition = "2021"

[lib]
path = "src/lib.rs"
```

Create `tests/fixtures/tiny-rust/src/lib.rs`:

```rust
pub mod util;

pub fn library_entry() {
    util::helper();
}
```

Create `tests/fixtures/tiny-rust/src/util.rs`:

```rust
pub fn helper() -> u32 {
    constant() + 1
}

pub fn constant() -> u32 {
    41
}

pub struct Config {
    pub debug: bool,
}

pub enum Mode {
    Fast,
    Slow,
}
```

Create `tests/fixtures/tiny-rust/src/main.rs`:

```rust
use tiny_rust::library_entry;

fn main() {
    library_entry();
}
```

- [ ] **Step 2: Write the gated integration test**

Create `tests/integration_index.rs`:

```rust
//! Integration test for `codegraph-rs index`. Indexes the tiny-rust fixture
//! against a live Neo4j Desktop. Gated on CODEGRAPH_RS_TEST_NEO4J=1.

use codegraph_rs::cli::{init::{InitArgs, self}, index};
use neo4rs::query;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn indexes_tiny_rust_fixture() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    // Set up a fresh workspace pointing at tests/fixtures/tiny-rust.
    let dir = tempdir().unwrap();
    let workspace_name = format!("index-test-{}", std::process::id());

    // Copy fixture into the temp directory so the workspace + roots resolve cleanly.
    let fixture_src = std::path::Path::new(env!("CARGO_MANIFEST_DIR")).join("tests/fixtures/tiny-rust");
    let fixture_dst = dir.path().join("tiny-rust");
    copy_dir(&fixture_src, &fixture_dst).unwrap();

    let args = InitArgs {
        name: workspace_name.clone(),
        neo4j_uri: "bolt://localhost:7687".into(),
        neo4j_user: "neo4j".into(),
        neo4j_password: pw.clone(),
        root: vec!["tiny=tiny-rust".into()],
    };
    init::run(args, Some(dir.path())).expect("init must succeed");

    // Index.
    std::env::set_var("CODEGRAPH_RS_NEO4J_PASSWORD", &pw);
    index::run(Some(dir.path())).expect("index must succeed");
    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    // Verify counts via Cypher.
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let db_name = format!("codegraph-index-test-{}", std::process::id());
        let conn = codegraph_rs::db::Connection::open(
            "bolt://localhost:7687", "neo4j", &pw, &db_name,
        ).await.unwrap();

        let file_count: i64 = scalar(&conn, "MATCH (n:File) RETURN count(n) AS c").await;
        let func_count: i64 = scalar(&conn, "MATCH (n:Function) RETURN count(n) AS c").await;
        let struct_count: i64 = scalar(&conn, "MATCH (n:Struct) RETURN count(n) AS c").await;
        let enum_count: i64 = scalar(&conn, "MATCH (n:Enum) RETURN count(n) AS c").await;
        let unresolved: i64 = scalar(&conn, "MATCH (n:UnresolvedRef) RETURN count(n) AS c").await;

        // tiny-rust has 3 .rs files (main.rs, util.rs, lib.rs).
        // util.rs has 2 functions, 1 struct, 1 enum.
        // lib.rs has 1 function. main.rs has 1 function (main).
        assert_eq!(file_count, 3, "expected 3 File nodes");
        assert!(func_count >= 4, "expected at least 4 Function nodes (got {func_count})");
        assert_eq!(struct_count, 1, "expected 1 Struct (Config)");
        assert_eq!(enum_count, 1, "expected 1 Enum (Mode)");
        assert!(unresolved >= 2, "expected at least 2 UnresolvedRef (helper(), constant(), library_entry())");

        // Drop the test database.
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        sys.graph.execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await.unwrap().next().await.ok();
    });
}

async fn scalar(conn: &codegraph_rs::db::Connection, cypher: &str) -> i64 {
    let mut stream = conn.graph.execute(query(cypher)).await.unwrap();
    let row = stream.next().await.unwrap().unwrap();
    row.get::<i64>("c").unwrap()
}

fn copy_dir(src: &std::path::Path, dst: &std::path::Path) -> std::io::Result<()> {
    fs::create_dir_all(dst)?;
    for entry in fs::read_dir(src)? {
        let entry = entry?;
        let path = entry.path();
        let dest = dst.join(entry.file_name());
        if path.is_dir() {
            copy_dir(&path, &dest)?;
        } else {
            fs::copy(&path, &dest)?;
        }
    }
    Ok(())
}
```

- [ ] **Step 3: Verify the SKIP path works**

```bash
cargo test --test integration_index -- --nocapture 2>&1 | tail -10
```

Expected: 1 test, prints SKIP, passes.

- [ ] **Step 4: Run the test against live Neo4j (optional — for the user)**

```bash
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='your-password' \
cargo test --test integration_index -- --nocapture --test-threads=1
```

Expected: test passes, indexer produces the expected counts, database is dropped on cleanup.

- [ ] **Step 5: Final test suite + commit + tag slice 2**

```bash
cargo test 2>&1 | tail -10
cargo clippy --all-targets -- -D warnings 2>&1 | tail -5
cargo fmt --check 2>&1 | tail -3

git add tests/fixtures/tiny-rust tests/integration_index.rs
git commit -m "test(extraction): integration test indexing tiny-rust fixture against Neo4j"

git tag slice-2-rust-extraction
```

Expected: all tests pass (28 unit + 4 integration_db SKIPs + 1 integration_init SKIP + 1 integration_status SKIP + 1 integration_index SKIP + 2 cli_smoke = ~37 tests). All clippy/fmt clean. Tag at HEAD.

---

## Slice 2 Done

Verify the artifact:

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — all unit tests pass; integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass
- [ ] `codegraph-rs index` runs against a live workspace and produces non-zero counts
- [ ] `codegraph-rs status` after indexing reports the same counts
- [ ] No CALLS edges yet (Slice 3)
- [ ] No embeddings yet (Slice 4)

When all check, slice 2 is complete and we can write slice 3 (Resolution).
