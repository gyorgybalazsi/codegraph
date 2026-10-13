# codegraph-rs Slice 9 — Daml Language Support

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `codegraph-rs index` and `codegraph-rs sync` understand `.daml` files. After this slice, a multi-root workspace combining Rust and Daml roots produces one connected Neo4j graph: Daml `Module`, `Template`, `Choice`, `Function`, `TypeAlias`, `Import`, and `Struct` (from `data` ADT-shaped declarations) nodes appear under `:Symbol`, with `CONTAINS` structural edges and `UnresolvedRef`/`HAS_REF` records for function applications, choice exercises, and module imports. Slice 7's embedding orchestrator already names `Template` and `Choice` in `EMBED_KINDS_CYPHER_LIST`, so the embedding stage automatically picks the new Daml symbols up — no orchestrator changes needed. Slice 3's resolution pipeline already handles workspace-wide `qualified_name` matching, so Daml `import Utility.Foo` resolves to a `Module` whose `qualified_name = "Utility.Foo"` regardless of which root it lives in.

**Architecture:** Mirror Slice 2's Rust extractor surface. Add `tree-sitter-haskell` to `Cargo.toml`, plumb a per-thread `DAML_PARSER: thread_local!(RefCell<Parser>)` through `src/extraction/parser.rs`, create `src/extraction/languages/daml.rs` with `pub fn extract_daml(source: &str, root_name: &str, relative_path: &str) -> Result<ExtractionResult>`, plus its companion `daml.scm` capture queries. Add `Template` and `Choice` to `NodeKind` in `src/extraction/result.rs`. Add `Some("daml") => extract_daml(...)` arms to two call sites: `src/extraction/orchestrator.rs` and `src/sync/mod.rs`. Add a `tests/fixtures/tiny-daml/` fixture, unit tests, and a gated integration test.

```
walker .daml → orchestrator dispatch → extract_daml → ExtractionResult
                                                          ↓
                                               write_extraction (unchanged)
                                                          ↓
                                                   resolution (unchanged)
                                                          ↓
                                                vectors::run (unchanged — picks up
                                                              Template/Choice via the
                                                              existing EMBED_KINDS list)
```

**Tech Stack additions:**
- `tree-sitter-haskell = "0.23"` — Haskell grammar for tree-sitter, used as v1 proxy for Daml per spec §2 / §6. v0.23.1 (released 2024-11-10) ships its `tree_sitter::Language` via the `tree-sitter-language ^0.1` ABI crate, the same way `tree-sitter-rust 0.24` does. Despite tree-sitter-haskell's dev-deps pinning `tree-sitter ^0.23`, the runtime ABI is stable and integrates cleanly with codegraph-rs's `tree-sitter = "0.25"`.
- `regex = "1"` — line-anchored regex scan for Daml-only constructs (`template`, `choice`) the Haskell grammar can't see.

No other new deps. When a real `tree-sitter-daml` grammar ever appears, the one-line swap in `parser.rs` enables deleting the Pass B regex code.

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §2 (Daml at v1 launch), §3 (crate layout), §4 (Template, Choice labels), §6 (per-language extractors), §8 (Template/Choice are embed-eligible).

**Slice 8 baseline:** HEAD `6cc3249` on `main`, tag `slice-8-semantic-search`. 76 tests pass.

---

## What this slice does, conceptually

1. **Add `Template` and `Choice` to `NodeKind`.** `src/extraction/result.rs` gains two variants. `NodeKind::as_label` returns `"Template"` and `"Choice"` — strings Slice 7's `EMBED_KINDS_CYPHER_LIST` already names, so the embedding selector immediately matches.

2. **`tree-sitter-haskell` as the parser.** Spec §6 commits to using Haskell as Daml's proxy grammar until a real grammar exists. Verified at spec time and confirmed for this slice: no `tree-sitter-daml` crate exists on crates.io.

3. **Two-pass extraction strategy:**
   - **Pass A — declarative tree-sitter capture queries** for grammar-clean constructs: Module, Import, Function, Struct (from `data_type`), TypeAlias.
   - **Pass B — line-anchored regex over raw source** for Daml-only constructs: Template, Choice. Pass B nodes carry `parse_reliable: false` per spec §4.
   - **Pass C — call-site extraction** runs LAST so `enclosing_symbol_qid` lookup sees Pass B's Template/Choice spans.

4. **`UnresolvedRef` semantics:**
   - `Call` — function applications + `exercise[ByKey] cid Choice`.
   - `TypeRef` — type identifiers in function signatures.
   - `Import` — `import M qualified as A`.

5. **Qualified-name shape:** `module Splice.Utility.Foo where` → Module qualified_name is `"Splice.Utility.Foo"`. Functions get `Splice.Utility.Foo.helperFn`. Templates get `Splice.Utility.Foo.Counter`. Choices get `Splice.Utility.Foo.Counter.Increment`. Dot-separators (Daml's native style), not Rust's `::`.

6. **Structural edges (CONTAINS):**
   - File → Module
   - Module → Template / Function / Import / TypeAlias / Struct
   - Template → Choice
   - When no module header is found, fall back to File as parent.

7. **No new edge types.** Spec §4 reserves `HAS_CHOICE`, `SIGNATORY`, etc. but lists them as "reserved". v1 uses `CONTAINS` for Template-Choice.

8. **Walker is already done.** `src/extraction/walker.rs` already maps `.daml` → `"daml"`.

---

## Out of scope for Slice 9

- Daml `interface` and `viewtype` declarations.
- Cross-package (DAR archive) resolution.
- Daml LSP integration.
- Daml-specific MCP tools (existing 10 tools handle Daml symbols transparently).
- Embeddings configuration changes (EMBED_KINDS already names Template/Choice).
- Schema migration (label-agnostic Symbol constraints).
- Signatory / controller / observer extraction as edges (reserved for a later slice).
- A real `tree-sitter-daml` grammar — not in scope; the spec explicitly defers this.

---

## Critical design decisions

### D1. tree-sitter-daml crate strategy

**Verification performed:**
- `curl https://crates.io/api/v1/crates/tree-sitter-daml` → does not exist.
- Web search: no canonical GitHub repo for tree-sitter-daml.
- `tree-sitter-haskell` exists at v0.23.1, integrates via `tree-sitter-language ^0.1` ABI — same path `tree-sitter-rust 0.24` already uses in tree with our `tree-sitter = "0.25"`.

**Decision: use `tree-sitter-haskell = "0.23"` as the proxy grammar.** Spec lines 50, 384, 409, 849 commit to this exact path. When a real tree-sitter-daml appears, swap is one line in `parser.rs` + delete Pass B.

### D2. tree-sitter-haskell node names

The Haskell grammar uses these node-type strings (per `tree-sitter-haskell`'s `src/node-types.json`):

| Daml construct | Haskell grammar node | Name field |
|---|---|---|
| Module header `module Foo.Bar where` | `header` | `name: (module)` |
| Import `import Foo.Bar` | `import` | `module: (module)` |
| Type signature `f : T -> T` | `signature` | `name: (variable)` |
| Function definition `f x = ...` | `function` | `name: (variable)` |
| `data D = ...` | `data_type` | `name: (name)` |
| `type T = ...` | `type_synomym` (sic — preserve typo) | `name: (name)` |
| Function application | `apply` | `function:` field |

**Implementer must verify these against the installed `tree-sitter-haskell = "0.23.1"` `node-types.json`** before writing queries. The `type_synomym` typo is upstream; do NOT "fix" it.

### D3. `UnresolvedRef` kinds emitted from Daml

- **Function application `helper x y`** → tree-sitter `(apply function: (variable) @call.callee) @call.site` → `UnresolvedRef{kind=Call}`.
- **Choice exercise `exercise cid Accept`** → Pass C regex `\bexercise(?:ByKey)?\b\s+(?:@\w+\s+)?\S+\s+([A-Z]\w*)` → `UnresolvedRef{kind=Call, unresolved_name="Accept"}`.
- **Import `import M`** → `UnresolvedRef{kind=Import}`, resolution Phase A converts to `IMPORTS` edge.
- **Type references in signatures** → `UnresolvedRef{kind=TypeRef}`.

### D4. Test fixture content — `tests/fixtures/tiny-daml/`

Three files exercising templates, choices, cross-module imports, function calls.

### D5. Test gates

- Unit tests: unconditional.
- Integration test `tests/integration_daml_index.rs`: gated on `CODEGRAPH_RS_TEST_NEO4J=1`.
- NO additional `CODEGRAPH_RS_TEST_EMBEDDINGS=1` gate (Slice 7's existing test covers the embedding kinds list).

### D6. Symbol ordering

Pass A (Module/Function/Import/Struct/TypeAlias) → Pass B (Template/Choice) → Pass C (call sites). Pass C's `enclosing_symbol_qid` lookup sees Pass B's nodes.

### D7. Daml `data` mapping

Emit every `data_type` as `NodeKind::Struct`. ADT discrimination (multi-constructor → `Enum`) is a follow-up.

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml                                  # add tree-sitter-haskell + regex
  src/
    extraction/
      result.rs                               # MODIFY: NodeKind::Template, ::Choice
      parser.rs                               # MODIFY: DAML_PARSER + parse_daml()
      orchestrator.rs                         # MODIFY: "daml" arm
      languages/
        mod.rs                                # MODIFY: pub mod daml; shared helper
        daml.rs                               # NEW: Daml extractor
        daml.scm                              # NEW: tree-sitter queries
        rust.rs                               # MODIFY: use shared helper
    sync/
      mod.rs                                  # MODIFY: Some("daml") arm
  tests/
    extraction_unit.rs                        # MODIFY: Daml extractor unit tests
    fixtures/
      tiny-daml/Splice/{Counter,Driver,Utility/Math}.daml   # NEW
    integration_daml_index.rs                 # NEW: gated integration test
  README.md                                   # MODIFY: bump Status
```

---

### Task 1: Add `tree-sitter-haskell` dependency

**Files:** Modify `Cargo.toml`.

- [ ] **Step 1: Add the dep**

Add to `[dependencies]` (alphabetical between `tree-sitter` and `tree-sitter-rust`):

```toml
tree-sitter-haskell = "0.23"
```

- [ ] **Step 2: Verify build**

```bash
cargo build 2>&1 | tail -10
```

Expected: clean build. First compile takes ~30-60s as `cc` builds the C grammar.

- [ ] **Step 3: Verify existing tests still pass**

```bash
cargo test 2>&1 | tail -5
```

Expected: all 76 tests still pass.

- [ ] **Step 4: Commit**

```bash
git add Cargo.toml Cargo.lock
git commit -m "chore(extraction): add tree-sitter-haskell dependency for Daml support"
```

---

### Task 2: Add `NodeKind::Template` and `NodeKind::Choice`

**Files:**
- Modify: `src/extraction/result.rs`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Extend the existing `node_kind_round_trips_to_str` test**

Find the test in `tests/extraction_unit.rs`. Extend its kinds list to include `Template, Choice`. Add a new test asserting the labels:

```rust
#[test]
fn template_and_choice_labels_match_spec_section_4() {
    assert_eq!(NodeKind::Template.as_label(), "Template");
    assert_eq!(NodeKind::Choice.as_label(), "Choice");
}
```

- [ ] **Step 2: Run failing test**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: compile error.

- [ ] **Step 3: Extend `NodeKind` in `src/extraction/result.rs`**

```rust
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
    /// Daml `template T with ... where ...`. Carries `parse_reliable = false`
    /// when emitted via the tree-sitter-haskell proxy.
    Template,
    /// Daml `choice C : T with ... do ...` inside a template.
    Choice,
}
```

Extend `as_label`:

```rust
NodeKind::Template => "Template",
NodeKind::Choice => "Choice",
```

- [ ] **Step 4: Run tests + verify EMBED_KINDS lists**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
grep -n "Template\|Choice" src/graph/status.rs src/vectors/query.rs
```

Both `EMBED_KINDS_CYPHER_LIST` constants should already list these — no changes.

- [ ] **Step 5: Commit**

```bash
git add src/extraction/result.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add NodeKind::Template and NodeKind::Choice"
```

---

### Task 3: Tree-sitter Daml parser via Haskell grammar

**Files:**
- Modify: `src/extraction/parser.rs`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

```rust
use codegraph_rs::extraction::parser::parse_daml;

#[test]
fn daml_parser_returns_tree_for_valid_module_header() {
    let source = "module Foo where\n\nf : Int -> Int\nf x = x + 1\n";
    let tree = parse_daml(source).expect("must parse");
    let root = tree.root_node();
    assert_eq!(root.kind(), "haskell");
}

#[test]
fn daml_parser_handles_template_with_error_recovery() {
    let source = "module Foo where\n\ntemplate T with\n    p : Party\n  where\n    signatory p\n";
    let tree = parse_daml(source);
    assert!(tree.is_some(), "must return a tree on Daml-specific syntax");
}

#[test]
fn daml_parser_is_thread_local_reusable() {
    let a = parse_daml("module A where").unwrap();
    let b = parse_daml("module B where").unwrap();
    assert_ne!(a.root_node().byte_range(), b.root_node().byte_range());
}
```

- [ ] **Step 2: Add the parser plumbing in `src/extraction/parser.rs`**

```rust
use std::cell::RefCell;
use tree_sitter::{Parser, Tree};

thread_local! {
    static RUST_PARSER: RefCell<Parser> = RefCell::new(make_parser_for(tree_sitter_rust::LANGUAGE.into()));
    /// Daml uses tree-sitter-haskell as a proxy grammar per spec §6. When a real
    /// tree-sitter-daml grammar ships, replace `tree_sitter_haskell::LANGUAGE`
    /// here and delete the Pass B regex in `languages/daml.rs`.
    static DAML_PARSER: RefCell<Parser> = RefCell::new(make_parser_for(tree_sitter_haskell::LANGUAGE.into()));
}

fn make_parser_for(language: tree_sitter::Language) -> Parser {
    let mut p = Parser::new();
    p.set_language(&language).expect("valid grammar");
    p
}

pub fn parse_rust(source: &str) -> Option<Tree> {
    RUST_PARSER.with(|cell| cell.borrow_mut().parse(source, None))
}

pub fn parse_daml(source: &str) -> Option<Tree> {
    DAML_PARSER.with(|cell| cell.borrow_mut().parse(source, None))
}
```

> **Verify root node kind.** If `daml_parser_returns_tree_for_valid_module_header` fails with `root.kind() != "haskell"`, capture the actual value (could be `"source_file"` in older versions) and update the test assertion.

- [ ] **Step 3: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

- [ ] **Step 4: Commit**

```bash
git add src/extraction/parser.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add per-thread tree-sitter-haskell parser for Daml"
```

---

### Task 4: Lift `enclosing_symbol_qid` to a shared helper (refactor)

**Files:**
- Modify: `src/extraction/languages/mod.rs`
- Modify: `src/extraction/languages/rust.rs`

Pull `enclosing_symbol_qid` (currently private in `rust.rs`) into `languages/mod.rs` for shared use.

- [ ] **Step 1: Add to `src/extraction/languages/mod.rs`**

```rust
//! Language-specific extractors.

pub mod rust;
// pub mod daml;  // added in Task 5

use crate::extraction::result::{NodeKind, RawNode};

/// Find the smallest non-File Symbol node in `nodes` whose span contains
/// `(line, col)`. Used by per-language extractors to attach an UnresolvedRef
/// to its enclosing Symbol.
pub(crate) fn enclosing_symbol_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&String> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        if matches!(n.kind, NodeKind::File) {
            continue;
        }
        let in_span = (n.start_line < line || (n.start_line == line && n.start_col <= col))
            && (n.end_line > line || (n.end_line == line && n.end_col >= col));
        if !in_span {
            continue;
        }
        match best {
            None => best = Some(n),
            Some(b) => {
                let n_lines = n.end_line.saturating_sub(n.start_line);
                let b_lines = b.end_line.saturating_sub(b.start_line);
                if n_lines < b_lines {
                    best = Some(n);
                }
            }
        }
    }
    best.map(|n| &n.qid)
}
```

- [ ] **Step 2: Update `src/extraction/languages/rust.rs`**

Delete the local `enclosing_symbol_qid` function. Update the call site to use `crate::extraction::languages::enclosing_symbol_qid(...)`.

- [ ] **Step 3: Build + test**

```bash
cargo build 2>&1 | tail -3
cargo test 2>&1 | tail -10
cargo clippy --all-targets -- -D warnings
```

Expected: pure refactor, all tests still pass.

- [ ] **Step 4: Commit**

```bash
git add src/extraction/languages/mod.rs src/extraction/languages/rust.rs
git commit -m "refactor(extraction): lift enclosing_symbol_qid into shared helper"
```

---

### Task 5: Daml extractor — module + imports (TDD)

**Files:**
- Create: `src/extraction/languages/daml.rs`
- Create: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/mod.rs` (add `pub mod daml;`)
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Wire the module**

Add `pub mod daml;` to `src/extraction/languages/mod.rs` after `pub mod rust;`.

- [ ] **Step 2: Append failing tests**

```rust
use codegraph_rs::extraction::languages::daml::extract_daml;

#[test]
fn daml_extracts_module_header() {
    let source = "module Splice.Utility.Math where\n";
    let result = extract_daml(source, "u", "Splice/Utility/Math.daml").expect("ok");
    let modules: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Module).collect();
    assert_eq!(modules.len(), 1);
    assert_eq!(modules[0].name, "Math");
    assert_eq!(modules[0].qualified_name, "Splice.Utility.Math");
    assert_eq!(modules[0].language, "daml");
    assert!(modules[0].parse_reliable);
}

#[test]
fn daml_extracts_file_node_with_hash() {
    let source = "module Foo where\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let files: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::File).collect();
    assert_eq!(files.len(), 1);
    assert_eq!(files[0].name, "Foo.daml");
    assert!(files[0].content_hash.is_some());
    assert_eq!(files[0].content_hash.as_ref().unwrap().len(), 64);
}

#[test]
fn daml_extracts_import_with_unresolved_ref() {
    use codegraph_rs::extraction::result::{EdgeKind, UnresolvedKind};
    let source = "module Splice.Driver where\n\nimport Splice.Counter\n";
    let result = extract_daml(source, "u", "Splice/Driver.daml").expect("ok");
    let imports: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Import).collect();
    assert_eq!(imports.len(), 1);
    assert_eq!(imports[0].qualified_name, "Splice.Counter");
    let import_refs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Import).collect();
    assert_eq!(import_refs.len(), 1);
    assert_eq!(import_refs[0].unresolved_name, "Splice.Counter");
    let contains: Vec<_> = result.edges.iter().filter(|e| e.kind == EdgeKind::Contains).collect();
    assert!(contains.len() >= 2);
}

#[test]
fn daml_extracts_qualified_import_alias() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\nimport Splice.Bar qualified as B\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let import_refs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Import).collect();
    assert_eq!(import_refs.len(), 1);
    assert_eq!(import_refs[0].unresolved_name, "Splice.Bar");
}

#[test]
fn daml_emits_contains_edges_from_module_not_file() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\nimport Bar\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let module_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Module).unwrap().qid;
    let import_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Import).unwrap().qid;
    let module_contains_import = result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *module_qid && e.to_qid == *import_qid);
    assert!(module_contains_import, "Module should CONTAINS Import");
}
```

- [ ] **Step 3: Write `src/extraction/languages/daml.scm`**

```scheme
; Module header: `module Foo.Bar.Baz where`.
(header (module) @module.name) @module.def

; Imports: `import Foo.Bar` or `import Foo.Bar qualified as B`.
(import (module) @import.name) @import.def
```

> **Verify against installed grammar:**
> ```bash
> find ~/.cargo/registry/src -type d -name 'tree-sitter-haskell-*' -exec ls {}/src/node-types.json \;
> ```
> Open the JSON. Confirm `"type": "header"` (or `module_header` in some versions) and `"type": "import"`. Adjust the .scm if names differ.

- [ ] **Step 4: Implement `src/extraction/languages/daml.rs`**

```rust
//! Daml language extractor.
//!
//! Uses tree-sitter-haskell as a proxy grammar per spec §6. Pass A handles
//! Haskell-parseable constructs (Module, Import, Function, Struct, TypeAlias);
//! Pass B handles Daml-only constructs (Template, Choice) via regex; Pass C
//! handles call-site UnresolvedRefs and runs LAST so Pass B's spans are visible.

use anyhow::{Context, Result};
use std::collections::HashMap;
use tree_sitter::{Query, QueryCursor, StreamingIterator};

use crate::extraction::hash::source_hash;
use crate::extraction::parser::parse_daml;
use crate::extraction::result::{
    EdgeKind, ExtractionResult, NodeKind, RawEdge, RawNode, UnresolvedKind, UnresolvedRef,
};

const QUERY_SOURCE: &str = include_str!("daml.scm");

pub fn extract_daml(
    source: &str,
    root_name: &str,
    relative_path: &str,
) -> Result<ExtractionResult> {
    let tree = parse_daml(source).context("tree-sitter-haskell failed to produce a tree")?;
    let language = tree_sitter_haskell::LANGUAGE;
    let query = Query::new(&language.into(), QUERY_SOURCE)
        .context("invalid tree-sitter-haskell query for Daml")?;

    let mut result = ExtractionResult::default();

    let file_node = build_file_node(source, root_name, relative_path);
    let file_qid = file_node.qid.clone();
    result.nodes.push(file_node);

    let module_qid_opt = pass_a_module(
        &query, &tree, source, root_name, relative_path, &file_qid, &mut result,
    );

    pass_a_imports(
        &query, &tree, source, root_name, relative_path,
        &module_qid_opt, &file_qid, &mut result,
    );

    // Task 6: pass_a_definitions (functions, type aliases, structs)
    // Task 7: pass_b_templates_and_choices (regex-based)
    // Task 8: pass_c_call_sites + pass_c_exercise_choices

    Ok(result)
}

fn build_file_node(source: &str, root_name: &str, relative_path: &str) -> RawNode {
    let qid = format!("{root_name}:{relative_path}");
    let name = std::path::Path::new(relative_path)
        .file_name()
        .map(|s| s.to_string_lossy().to_string())
        .unwrap_or_else(|| relative_path.to_string());
    let line_count = source.lines().count() as u32;
    let now_unix = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|d| d.as_secs() as i64)
        .unwrap_or(0);
    RawNode {
        qid,
        kind: NodeKind::File,
        name,
        qualified_name: relative_path.to_string(),
        root_name: root_name.to_string(),
        relative_path: relative_path.to_string(),
        language: "daml".to_string(),
        start_line: 1,
        end_line: line_count.max(1),
        start_col: 0,
        end_col: 0,
        signature: None,
        doc: None,
        parse_reliable: true,
        content_hash: Some(source_hash(source)),
        last_indexed_at: Some(now_unix),
    }
}

fn pass_a_module(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    file_qid: &str,
    result: &mut ExtractionResult,
) -> Option<String> {
    let def_idx = query.capture_index_for_name("module.def")?;
    let name_idx = query.capture_index_for_name("module.name")?;
    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        if let (Some(def), Some(name)) = (def_node, name_node) {
            let qname = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let leaf = qname.rsplit('.').next().unwrap_or(&qname).to_string();
            let qid = format!("{root_name}:{relative_path}:{qname}");
            let start = def.start_position();
            let end = def.end_position();
            result.nodes.push(RawNode {
                qid: qid.clone(),
                kind: NodeKind::Module,
                name: leaf,
                qualified_name: qname,
                root_name: root_name.to_string(),
                relative_path: relative_path.to_string(),
                language: "daml".to_string(),
                start_line: (start.row as u32) + 1,
                end_line: (end.row as u32) + 1,
                start_col: start.column as u32,
                end_col: end.column as u32,
                signature: None,
                doc: None,
                parse_reliable: true,
                content_hash: None,
                last_indexed_at: None,
            });
            result.edges.push(RawEdge {
                from_qid: file_qid.to_string(),
                to_qid: qid.clone(),
                kind: EdgeKind::Contains,
            });
            return Some(qid);
        }
    }
    None
}

fn pass_a_imports(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: &Option<String>,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("import.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("import.name") else { return };
    let parent_qid: &str = module_qid.as_deref().unwrap_or(file_qid);
    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());
    let mut ordinal: u32 = 0;

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        if let (Some(def), Some(name)) = (def_node, name_node) {
            let module_path = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let key = format!("{root_name}:{relative_path}:import:{module_path}");
            let qid = if ordinal == 0 { key.clone() } else { format!("{key}#{ordinal}") };
            ordinal += 1;
            let start = def.start_position();
            let end = def.end_position();
            let line = (start.row as u32) + 1;
            let col = start.column as u32;
            result.nodes.push(RawNode {
                qid: qid.clone(),
                kind: NodeKind::Import,
                name: module_path.clone(),
                qualified_name: module_path.clone(),
                root_name: root_name.to_string(),
                relative_path: relative_path.to_string(),
                language: "daml".to_string(),
                start_line: line,
                end_line: (end.row as u32) + 1,
                start_col: col,
                end_col: end.column as u32,
                signature: None,
                doc: None,
                parse_reliable: true,
                content_hash: None,
                last_indexed_at: None,
            });
            result.edges.push(RawEdge {
                from_qid: parent_qid.to_string(),
                to_qid: qid.clone(),
                kind: EdgeKind::Contains,
            });
            let ref_qid = format!("{qid}:ref:{line}:{col}:0");
            result.unresolved_refs.push(UnresolvedRef {
                qid: ref_qid.clone(),
                source_qid: qid.clone(),
                unresolved_name: module_path,
                kind: UnresolvedKind::Import,
                line,
                col,
            });
            result.edges.push(RawEdge {
                from_qid: qid,
                to_qid: ref_qid,
                kind: EdgeKind::HasRef,
            });
        }
    }
}
```

- [ ] **Step 5: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -20
```

Expected: 5 new Daml tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm src/extraction/languages/mod.rs tests/extraction_unit.rs
git commit -m "feat(extraction): Daml extractor — module header + imports"
```

---

### Task 6: Daml extractor — functions, type synonyms, data types (TDD)

**Files:**
- Modify: `src/extraction/languages/daml.rs`
- Modify: `src/extraction/languages/daml.scm`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_extracts_function_with_signature() {
    let source = "module Foo where\n\nhelperFn : Int -> Int\nhelperFn x = x + 1\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let funcs: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Function).collect();
    assert_eq!(funcs.len(), 1);
    assert_eq!(funcs[0].name, "helperFn");
    assert_eq!(funcs[0].qualified_name, "Foo.helperFn");
    assert!(funcs[0].signature.as_deref().unwrap_or("").contains("Int -> Int"));
}

#[test]
fn daml_extracts_type_synonym() {
    let source = "module Foo where\n\ntype CounterId = ContractId Counter\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let types: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::TypeAlias).collect();
    assert_eq!(types.len(), 1);
    assert_eq!(types[0].name, "CounterId");
    assert_eq!(types[0].qualified_name, "Foo.CounterId");
}

#[test]
fn daml_extracts_data_as_struct() {
    let source = "module Foo where\n\ndata Config = Config { debug : Bool, level : Int }\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let structs: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Struct).collect();
    assert_eq!(structs.len(), 1);
    assert_eq!(structs[0].name, "Config");
    assert_eq!(structs[0].qualified_name, "Foo.Config");
}

#[test]
fn daml_function_contains_edge_from_module() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\n\nf : Int -> Int\nf x = x\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let module_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Module).unwrap().qid;
    let func_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Function).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *module_qid && e.to_qid == *func_qid));
}
```

- [ ] **Step 2: Extend `daml.scm`**

Append:

```scheme
; Top-level function bindings.
(function name: (variable) @function.name) @function.def

; Type signatures — paired with function decl by name.
(signature name: (variable) @signature.name) @signature.def

; Data declarations.
(data_type name: (name) @struct.name) @struct.def

; Type synonyms. NOTE upstream typo `type_synomym` (sic).
(type_synomym name: (name) @type.name) @type.def
```

> **Verify field names against `node-types.json`** — `signature` in particular varies between grammar revisions.

- [ ] **Step 3: Extend `daml.rs`**

Inside `extract_daml`, after `pass_a_imports`:

```rust
    let signatures = pass_a_collect_signatures(&query, &tree, source);
    pass_a_definitions(
        &query, &tree, source, root_name, relative_path,
        module_qid_opt.as_deref(), &signatures, &mut result,
    );
```

Add the helpers:

```rust
fn pass_a_collect_signatures(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
) -> HashMap<String, String> {
    let mut sigs = HashMap::new();
    let Some(def_idx) = query.capture_index_for_name("signature.def") else { return sigs };
    let Some(name_idx) = query.capture_index_for_name("signature.name") else { return sigs };
    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());
    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        if let (Some(def), Some(name)) = (def_node, name_node) {
            let fn_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let sig_text = def.utf8_text(source.as_bytes()).unwrap_or("").trim().to_string();
            sigs.insert(fn_name, sig_text);
        }
    }
    sigs
}

fn pass_a_definitions(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: Option<&str>,
    signatures: &HashMap<String, String>,
    result: &mut ExtractionResult,
) {
    let captures: &[(&str, &str, NodeKind)] = &[
        ("function.def", "function.name", NodeKind::Function),
        ("struct.def", "struct.name", NodeKind::Struct),
        ("type.def", "type.name", NodeKind::TypeAlias),
    ];
    let resolved: Vec<(u32, u32, NodeKind)> = captures.iter()
        .filter_map(|(d, n, k)| {
            Some((
                query.capture_index_for_name(d)?,
                query.capture_index_for_name(n)?,
                *k,
            ))
        })
        .collect();

    // module_qid format: "<root>:<rel>:<qname>". Split out qname.
    let module_prefix = module_qid
        .map(|q| q.split(':').skip(2).collect::<Vec<_>>().join(":"))
        .unwrap_or_default();
    let file_qid = result.nodes[0].qid.clone();

    let mut ordinals: HashMap<String, u32> = HashMap::new();
    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        for (def_idx, name_idx, kind) in &resolved {
            let mut def_node = None;
            let mut name_node = None;
            for cap in m.captures {
                if cap.index == *def_idx { def_node = Some(cap.node); }
                else if cap.index == *name_idx { name_node = Some(cap.node); }
            }
            if let (Some(def), Some(name)) = (def_node, name_node) {
                let item_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
                let qualified_name = if module_prefix.is_empty() {
                    item_name.clone()
                } else {
                    format!("{module_prefix}.{item_name}")
                };
                let key = format!("{root_name}:{relative_path}:{qualified_name}");
                let ord = ordinals.entry(key.clone()).and_modify(|n| *n += 1).or_insert(0);
                let qid = if *ord == 0 { key.clone() } else { format!("{key}#{ord}") };
                let start = def.start_position();
                let end = def.end_position();
                let signature = if *kind == NodeKind::Function {
                    signatures.get(&item_name).cloned()
                } else {
                    None
                };
                result.nodes.push(RawNode {
                    qid: qid.clone(),
                    kind: *kind,
                    name: item_name,
                    qualified_name,
                    root_name: root_name.to_string(),
                    relative_path: relative_path.to_string(),
                    language: "daml".to_string(),
                    start_line: (start.row as u32) + 1,
                    end_line: (end.row as u32) + 1,
                    start_col: start.column as u32,
                    end_col: end.column as u32,
                    signature,
                    doc: None,
                    parse_reliable: true,
                    content_hash: None,
                    last_indexed_at: None,
                });
                let parent_qid = module_qid.unwrap_or(&file_qid).to_string();
                result.edges.push(RawEdge {
                    from_qid: parent_qid,
                    to_qid: qid,
                    kind: EdgeKind::Contains,
                });
                break;
            }
        }
    }
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml extractor — functions, type synonyms, data types"
```

---

### Task 7: Pass B — Template + Choice regex extraction (TDD)

**Files:**
- Modify: `Cargo.toml` (add `regex = "1"`)
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Add `regex` dep**

Append to `[dependencies]` (alphabetical):

```toml
regex = "1"
```

- [ ] **Step 2: Append failing tests**

```rust
#[test]
fn daml_extracts_template_via_regex_pass() {
    let source = "module Foo where\n\ntemplate Counter\n  with\n    p : Party\n    count : Int\n  where\n    signatory p\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let templates: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Template).collect();
    assert_eq!(templates.len(), 1);
    assert_eq!(templates[0].name, "Counter");
    assert_eq!(templates[0].qualified_name, "Foo.Counter");
    assert!(!templates[0].parse_reliable, "regex-derived Template must be parse_reliable=false");
}

#[test]
fn daml_extracts_choices_inside_template() {
    let source = "module Foo where\n\ntemplate Counter\n  with\n    p : Party\n  where\n    signatory p\n    choice Increment : ContractId Counter\n      controller p\n      do create this with count = count + 1\n    nonconsuming choice Peek : Int\n      controller p\n      do return count\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let choices: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Choice).collect();
    assert_eq!(choices.len(), 2);
    let names: Vec<&str> = choices.iter().map(|c| c.name.as_str()).collect();
    assert!(names.contains(&"Increment"));
    assert!(names.contains(&"Peek"));
    for c in &choices {
        assert!(c.qualified_name.starts_with("Foo.Counter."));
        assert!(!c.parse_reliable);
    }
}

#[test]
fn daml_choice_contains_edge_from_template() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\ntemplate Counter\n  with p : Party\n  where\n    signatory p\n    choice Inc : ()\n      controller p\n      do return ()\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let template_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Template).unwrap().qid;
    let choice_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Choice).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *template_qid && e.to_qid == *choice_qid));
}

#[test]
fn daml_template_contains_edge_from_module() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\ntemplate T with p : Party where signatory p\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let module_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Module).unwrap().qid;
    let template_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Template).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *module_qid && e.to_qid == *template_qid));
}
```

- [ ] **Step 3: Implement Pass B**

In `daml.rs`, add inside `extract_daml` after Pass A definitions:

```rust
    pass_b_templates_and_choices(
        source, root_name, relative_path,
        module_qid_opt.as_deref(), &file_qid, &mut result,
    );
```

Add the helpers:

```rust
use regex::Regex;

fn pass_b_templates_and_choices(
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: Option<&str>,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let template_re = Regex::new(r"(?m)^template\s+([A-Z]\w*)").expect("template regex");
    let choice_re = Regex::new(r"(?m)^\s+(?:nonconsuming\s+|preconsuming\s+|postconsuming\s+)?choice\s+([A-Z]\w*)\s*:\s*(\S+)").expect("choice regex");

    let module_prefix = module_qid
        .map(|q| q.split(':').skip(2).collect::<Vec<_>>().join(":"))
        .unwrap_or_default();
    let parent_for_template: &str = module_qid.unwrap_or(file_qid);

    let template_matches: Vec<(usize, regex::Captures)> = template_re
        .captures_iter(source)
        .map(|c| (c.get(0).unwrap().start(), c))
        .collect();

    for (i, (start_byte, cap)) in template_matches.iter().enumerate() {
        let template_name = cap.get(1).unwrap().as_str().to_string();
        let end_byte = template_matches.get(i + 1)
            .map(|(b, _)| *b)
            .unwrap_or(source.len());
        let span_text = &source[*start_byte..end_byte];
        let start_line = byte_to_line(source, *start_byte) + 1;
        let end_line = byte_to_line(source, end_byte.saturating_sub(1)) + 1;
        let qname = if module_prefix.is_empty() {
            template_name.clone()
        } else {
            format!("{module_prefix}.{template_name}")
        };
        let template_qid = format!("{root_name}:{relative_path}:{qname}");
        result.nodes.push(RawNode {
            qid: template_qid.clone(),
            kind: NodeKind::Template,
            name: template_name,
            qualified_name: qname.clone(),
            root_name: root_name.to_string(),
            relative_path: relative_path.to_string(),
            language: "daml".to_string(),
            start_line: start_line as u32,
            end_line: end_line as u32,
            start_col: 0,
            end_col: 0,
            signature: None,
            doc: None,
            parse_reliable: false,
            content_hash: None,
            last_indexed_at: None,
        });
        result.edges.push(RawEdge {
            from_qid: parent_for_template.to_string(),
            to_qid: template_qid.clone(),
            kind: EdgeKind::Contains,
        });

        for ccap in choice_re.captures_iter(span_text) {
            let choice_name = ccap.get(1).unwrap().as_str().to_string();
            let choice_byte = *start_byte + ccap.get(0).unwrap().start();
            let cline = byte_to_line(source, choice_byte) + 1;
            let choice_qname = format!("{qname}.{choice_name}");
            let choice_qid = format!("{root_name}:{relative_path}:{choice_qname}");
            result.nodes.push(RawNode {
                qid: choice_qid.clone(),
                kind: NodeKind::Choice,
                name: choice_name,
                qualified_name: choice_qname,
                root_name: root_name.to_string(),
                relative_path: relative_path.to_string(),
                language: "daml".to_string(),
                start_line: cline as u32,
                end_line: cline as u32,
                start_col: 0,
                end_col: 0,
                signature: Some(ccap.get(0).unwrap().as_str().trim().to_string()),
                doc: None,
                parse_reliable: false,
                content_hash: None,
                last_indexed_at: None,
            });
            result.edges.push(RawEdge {
                from_qid: template_qid.clone(),
                to_qid: choice_qid,
                kind: EdgeKind::Contains,
            });
        }
    }
}

fn byte_to_line(source: &str, byte: usize) -> usize {
    source[..byte.min(source.len())].bytes().filter(|&b| b == b'\n').count()
}
```

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml Cargo.lock src/extraction/languages/daml.rs tests/extraction_unit.rs
git commit -m "feat(extraction): Daml extractor — Pass B regex-based template + choice scan"
```

---

### Task 8: Pass C — call sites + exercise choices

**Files:**
- Modify: `src/extraction/languages/daml.rs`
- Modify: `src/extraction/languages/daml.scm`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_extracts_function_application_as_unresolved_ref() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\nf : Int -> Int\nf x = helper x\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let call_refs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Call).collect();
    assert!(call_refs.iter().any(|r| r.unresolved_name == "helper"));
}

#[test]
fn daml_extracts_exercise_choice_as_call() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ndriver : Update ()\ndriver = do\n  cid <- create someCounter\n  exercise cid Increment\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let call_refs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Call).collect();
    assert!(call_refs.iter().any(|r| r.unresolved_name == "Increment"));
}

#[test]
fn daml_call_ref_attaches_to_enclosing_function() {
    use codegraph_rs::extraction::result::{EdgeKind, UnresolvedKind};
    let source = "module Foo where\n\nf : Int -> Int\nf x = helper x\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let func_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Function).unwrap().qid;
    let helper_ref = result.unresolved_refs.iter()
        .find(|r| r.kind == UnresolvedKind::Call && r.unresolved_name == "helper")
        .expect("helper ref must exist");
    assert_eq!(&helper_ref.source_qid, func_qid);
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::HasRef && e.from_qid == *func_qid && e.to_qid == helper_ref.qid));
}
```

- [ ] **Step 2: Extend `daml.scm`**

```scheme
; Function application.
(apply function: [(variable) (constructor)] @apply.callee) @apply.site
```

> If the `function:` field name doesn't exist on `apply` in the installed grammar, change to `(apply [(variable) (constructor)] @apply.callee) @apply.site` (positional).

- [ ] **Step 3: Implement Pass C**

Inside `extract_daml`, after Pass B:

```rust
    pass_c_call_sites(&query, &tree, source, &file_qid, &mut result);
    pass_c_exercise_choices(source, &file_qid, &mut result);

    Ok(result)
```

(The `Ok(result)` moves here from earlier.)

Helpers:

```rust
fn pass_c_call_sites(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let Some(callee_idx) = query.capture_index_for_name("apply.callee") else { return };
    let Some(site_idx) = query.capture_index_for_name("apply.site") else { return };
    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());
    let mut ref_ord: HashMap<String, u32> = HashMap::new();

    while let Some(m) = matches.next() {
        let mut callee = None;
        let mut site = None;
        for cap in m.captures {
            if cap.index == callee_idx { callee = Some(cap.node); }
            else if cap.index == site_idx { site = Some(cap.node); }
        }
        if let (Some(callee), Some(site)) = (callee, site) {
            let callee_text = callee.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let leaf = callee_text.rsplit('.').next().unwrap_or(&callee_text).to_string();
            if leaf.is_empty() || !leaf.chars().next().map(|c| c.is_alphabetic() || c == '_').unwrap_or(false) {
                continue;
            }
            let pos = site.start_position();
            let line = (pos.row as u32) + 1;
            let col = pos.column as u32;
            let source_qid = crate::extraction::languages::enclosing_symbol_qid(&result.nodes, line, col)
                .unwrap_or(&file_qid.to_string())
                .clone();
            let key = format!("{source_qid}:ref:{line}:{col}");
            let ord = ref_ord.entry(key.clone()).and_modify(|n| *n += 1).or_insert(0);
            let qid = format!("{key}:{ord}");
            result.unresolved_refs.push(UnresolvedRef {
                qid: qid.clone(),
                source_qid: source_qid.clone(),
                unresolved_name: leaf,
                kind: UnresolvedKind::Call,
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
}

fn pass_c_exercise_choices(
    source: &str,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let exercise_re = Regex::new(r"(?m)\bexercise(?:ByKey)?\b[^\n]*?\s([A-Z]\w*)\b")
        .expect("exercise regex");
    for cap in exercise_re.captures_iter(source) {
        let choice_name = cap.get(1).unwrap().as_str().to_string();
        let byte = cap.get(0).unwrap().start();
        let line = (byte_to_line(source, byte) + 1) as u32;
        let col = 0u32;
        let source_qid = crate::extraction::languages::enclosing_symbol_qid(&result.nodes, line, col)
            .unwrap_or(&file_qid.to_string())
            .clone();
        let key = format!("{source_qid}:ref:{line}:{col}:exercise");
        let qid = format!("{key}:0");
        if result.unresolved_refs.iter().any(|r| r.qid == qid) {
            continue;
        }
        result.unresolved_refs.push(UnresolvedRef {
            qid: qid.clone(),
            source_qid: source_qid.clone(),
            unresolved_name: choice_name,
            kind: UnresolvedKind::Call,
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

- [ ] **Step 4: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml extractor — Pass C call sites + exercise choices"
```

---

### Task 9: Wire Daml dispatch in orchestrator + sync

**Files:**
- Modify: `src/extraction/orchestrator.rs`
- Modify: `src/sync/mod.rs`

- [ ] **Step 1: Orchestrator dispatch**

Add `use crate::extraction::languages::daml::extract_daml;` import. Replace the `"daml" => Ok(ExtractionResult::default())` arm (if it exists, otherwise add) with:

```rust
"daml" => extract_daml(&source, &f.root_name, &f.relative_path),
```

- [ ] **Step 2: Sync dispatch**

Add `use crate::extraction::languages::daml::extract_daml;` import. In the match around line 136:

```rust
Some("rs") => extract_rust(source, &root.name, rel),
Some("daml") => extract_daml(source, &root.name, rel),
Some(other) => {
    warn!("no extractor for `.{other}`, skipping {}", rel);
    continue;
}
```

- [ ] **Step 3: Build, test, clippy**

```bash
cargo build 2>&1 | tail -3
cargo test 2>&1 | tail -10
cargo clippy --all-targets -- -D warnings
```

- [ ] **Step 4: Commit**

```bash
git add src/extraction/orchestrator.rs src/sync/mod.rs
git commit -m "feat(extraction): wire extract_daml into orchestrator and sync dispatch"
```

---

### Task 10: tiny-daml fixture + gated integration test

**Files:**
- Create: `tests/fixtures/tiny-daml/Splice/Utility/Math.daml`
- Create: `tests/fixtures/tiny-daml/Splice/Counter.daml`
- Create: `tests/fixtures/tiny-daml/Splice/Driver.daml`
- Create: `tests/integration_daml_index.rs`

- [ ] **Step 1: Create the fixture files**

`tests/fixtures/tiny-daml/Splice/Utility/Math.daml`:

```daml
module Splice.Utility.Math where

type Counter = Int

incrementBy : Int -> Int -> Int
incrementBy delta n = n + delta

double : Int -> Int
double n = incrementBy n n
```

`tests/fixtures/tiny-daml/Splice/Counter.daml`:

```daml
module Splice.Counter where

import Splice.Utility.Math

template Counter
  with
    owner : Party
    count : Int
  where
    signatory owner

    choice Increment : ContractId Counter
      controller owner
      do
        let next = incrementBy 1 count
        create this with count = next

    choice Reset : ContractId Counter
      controller owner
      do
        create this with count = 0
```

`tests/fixtures/tiny-daml/Splice/Driver.daml`:

```daml
module Splice.Driver where

import Splice.Counter

runScenario : Party -> ContractId Counter -> Update (ContractId Counter)
runScenario _owner cid = do
  exercise cid Increment
```

- [ ] **Step 2: Create `tests/integration_daml_index.rs`**

```rust
//! Integration test for `codegraph-rs index` against tiny-daml. Gated on
//! CODEGRAPH_RS_TEST_NEO4J=1.

use codegraph_rs::cli::{
    index,
    init::{self, InitArgs},
};
use neo4rs::query;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn indexes_tiny_daml_fixture() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("daml-test-{}", std::process::id());

    let fixture_src = std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
        .join("tests/fixtures/tiny-daml");
    let fixture_dst = dir.path().join("tiny-daml");
    copy_dir(&fixture_src, &fixture_dst).unwrap();

    let args = InitArgs {
        name: workspace_name.clone(),
        neo4j_uri: "bolt://localhost:7687".into(),
        neo4j_user: "neo4j".into(),
        neo4j_password: pw.clone(),
        root: vec!["tiny=tiny-daml".into()],
    };
    init::run(args, Some(dir.path())).expect("init must succeed");

    std::env::set_var("CODEGRAPH_RS_NEO4J_PASSWORD", &pw);
    index::run(Some(dir.path())).expect("index must succeed");
    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    let cfg = codegraph_rs::config::workspace::load_from_path(dir.path()).unwrap();
    let db_name = cfg.workspace.neo4j.database.clone();

    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let conn = codegraph_rs::db::Connection::open(
            "bolt://localhost:7687", "neo4j", &pw, &db_name,
        ).await.unwrap();

        let file_count: i64 = scalar(&conn, "MATCH (n:File) RETURN count(n) AS c").await;
        let module_count: i64 = scalar(&conn, "MATCH (n:Module) RETURN count(n) AS c").await;
        let template_count: i64 = scalar(&conn, "MATCH (n:Template) RETURN count(n) AS c").await;
        let choice_count: i64 = scalar(&conn, "MATCH (n:Choice) RETURN count(n) AS c").await;
        let function_count: i64 = scalar(&conn, "MATCH (n:Function) RETURN count(n) AS c").await;
        let type_alias_count: i64 = scalar(&conn, "MATCH (n:TypeAlias) RETURN count(n) AS c").await;
        let import_count: i64 = scalar(&conn, "MATCH (n:Import) RETURN count(n) AS c").await;

        assert_eq!(file_count, 3, "expected 3 File nodes");
        assert_eq!(module_count, 3, "expected 3 Module nodes");
        assert_eq!(template_count, 1, "expected 1 Template (Counter)");
        assert_eq!(choice_count, 2, "expected 2 Choices");
        assert!(function_count >= 3, "expected >=3 Functions");
        assert_eq!(type_alias_count, 1, "expected 1 TypeAlias");
        assert_eq!(import_count, 2, "expected 2 Imports");

        let choice_names: Vec<String> = string_list(&conn,
            "MATCH (c:Choice) RETURN c.name AS c ORDER BY c.name").await;
        assert_eq!(choice_names, vec!["Increment", "Reset"]);

        let unreliable: i64 = scalar(&conn,
            "MATCH (t:Template) WHERE t.parse_reliable = false RETURN count(t) AS c").await;
        assert_eq!(unreliable, 1, "Template should carry parse_reliable=false");

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

async fn string_list(conn: &codegraph_rs::db::Connection, cypher: &str) -> Vec<String> {
    let mut stream = conn.graph.execute(query(cypher)).await.unwrap();
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.unwrap() {
        out.push(row.get::<String>("c").unwrap());
    }
    out
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

- [ ] **Step 3: Verify SKIP path**

```bash
cargo test --test integration_daml_index 2>&1 | tail -10
```

- [ ] **Step 4: Commit**

```bash
git add tests/fixtures/tiny-daml tests/integration_daml_index.rs
git commit -m "test(extraction): tiny-daml fixture + gated integration test"
```

---

### Task 11: README + slice-9-daml tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Bump the Status section**

Replace with:

```markdown
## Status

In development. Currently implements **Slice 9: Daml language support**
— workspace initialization, indexing, resolution, incremental sync,
on-write `BAAI/bge-small-en-v1.5` embeddings, a stdio MCP server
exposing 10 tools, AND Rust + Daml extraction. Daml files (`.daml`)
are parsed via `tree-sitter-haskell` as a proxy grammar (per spec §6);
modules, imports, top-level functions, type synonyms, and data
declarations come out of the tree-sitter pass with `parse_reliable =
true`, while `template` and `choice` declarations are extracted via a
regex-based pass and carry `parse_reliable = false` until a real
`tree-sitter-daml` grammar ships.
```

- [ ] **Step 2: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected: all tests pass (count depends on exactly how many unit tests landed in tasks 5-8; target is ~90 with all the new Daml unit tests).

- [ ] **Step 3: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 9 (Daml language support)"
git tag slice-9-daml
```

- [ ] **Step 4: Verify**

```bash
git log --oneline | head -15
git tag --list
```

Expected tags: `slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`, `slice-6-mcp`, `slice-7-embeddings`, `slice-8-semantic-search`, `slice-9-daml`.

---

## Slice 9 Done

- [ ] `cargo build` clean (first build downloads + compiles tree-sitter-haskell C grammar; ~30-60 s)
- [ ] `cargo test` (no env vars) — all unit tests pass; integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass, including `integration_daml_index`
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 CODEGRAPH_RS_TEST_EMBEDDINGS=1 cargo test -- --test-threads=1` — all tests pass; embeddings test unchanged (still uses tiny-rust)
- [ ] Manual: `codegraph-rs init --name daml-stack --root splice=../splice --root utility=../utility` works on a real multi-root Daml workspace; `codegraph-rs index` populates the graph
- [ ] Manual: in Neo4j Browser, `MATCH (s:Symbol) WHERE s.language = 'daml' AND s.embedding IS NOT NULL RETURN labels(s), count(s)` shows Template and Choice nodes with embeddings
- [ ] `parse_reliable = false` on every Template/Choice; `true` on every Module/Function/Import/TypeAlias/Struct

When all check, slice 9 is complete and Daml is a v1 launch language per spec mandate.

### Small follow-ups noted during this slice

- **Daml `data` ADT discrimination.** Emit multi-constructor `data` as `Enum`. ~30 lines.
- **`interface` / `viewtype` extraction.** Future slice via Pass-B-style regex.
- **Signatory/controller/observer edges.** Spec §4 reserves `SIGNATORY`, `CONTROLLER`, `OBSERVER`. Future slice.
- **Replace `tree-sitter-haskell` with real `tree-sitter-daml`.** One-line swap in `parser.rs` + delete Pass B when it exists.
- **Daml embeddings parity test.** `tests/integration_daml_vectors.rs` gated on both env vars, asserting Template/Choice get 384-dim embeddings.
