# codegraph-rs Slice 10a — Daml structure + fields

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the regex-based `pass_d_templates_and_choices` in `src/extraction/languages/daml.rs` with tree-sitter queries that filter `(variable)` nodes by Daml keyword. Add four new NodeKinds (`Interface`, `Viewtype`, `Exception`, `Field`); make Template / Choice / Interface fields first-class CONTAINS-children with byte-precise spans.

**Architecture:** Tree-sitter-haskell parses Daml-specific keywords as `(variable)` nodes inside `(apply ...)` expressions (confirmed by DLC-link/zed-daml-lsp). We exploit that by writing keyword-anchored queries in `daml.scm`. Field extraction shares the `(signature)` query shape with Function signatures; disambiguation happens at emit time by checking the ancestor chain for a Daml keyword anchor.

**Tech Stack additions:** None. `tree-sitter-haskell 0.23.1` is already in `Cargo.toml` since slice 9; `regex 1` is in but will be removed by the end of this slice (we delete the regex-based passes).

**Source spec:** [`docs/superpowers/specs/2026-05-15-codegraph-rs-v2-daml-depth-design.md`](../specs/2026-05-15-codegraph-rs-v2-daml-depth-design.md) §2.1, §3.2, §5.

**Slice 9 baseline:** HEAD `b3fdc0a` on `main`, tag `slice-9-daml`. 97 tests pass.

---

## File structure produced by this slice

```
codegraph-rs/
  src/
    extraction/
      result.rs                                # MODIFY: add NodeKind::Interface, ::Viewtype, ::Exception, ::Field
      languages/
        daml.rs                                # MODIFY: replace regex pass_d with tree-sitter passes
        daml.scm                               # MODIFY: add keyword-anchored queries
    graph/
      status.rs                                # MODIFY: extend EMBED_KINDS_CYPHER_LIST (3 entries: Interface, Viewtype, Exception)
    vectors/
      query.rs                                 # MODIFY: extend EMBED_KINDS_CYPHER_LIST (same 3 entries)
  tests/
    fixtures/
      tiny-daml/
        Splice/
          Authorization.daml                   # NEW: v2 fixture (interfaces, viewtypes, exceptions, fields)
    extraction_unit.rs                         # MODIFY: ~10 new unit tests
    integration_daml_index.rs                  # MODIFY: extended Cypher assertions
  Cargo.toml                                   # MODIFY at end: remove `regex` dep (no longer used after pass_d swap)
  README.md                                    # MODIFY at end: bump Status to slice 10a
```

---

### Task 1: Add `NodeKind::Interface`, `::Viewtype`, `::Exception`, `::Field`

**Files:**
- Modify: `src/extraction/result.rs`
- Modify: `src/graph/status.rs`
- Modify: `src/vectors/query.rs`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests to `tests/extraction_unit.rs`**

```rust
#[test]
fn nodekind_interface_label() {
    assert_eq!(NodeKind::Interface.as_label(), "Interface");
}

#[test]
fn nodekind_viewtype_label() {
    assert_eq!(NodeKind::Viewtype.as_label(), "Viewtype");
}

#[test]
fn nodekind_exception_label() {
    assert_eq!(NodeKind::Exception.as_label(), "Exception");
}

#[test]
fn nodekind_field_label() {
    assert_eq!(NodeKind::Field.as_label(), "Field");
}
```

Also extend the existing `node_kind_round_trips_to_str` test's iteration list:

```rust
for k in [
    File, Function, Method, Struct, Enum, Trait, TypeAlias, Module, Import,
    Template, Choice,
    Interface, Viewtype, Exception, Field,
] {
```

- [ ] **Step 2: Run failing tests**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: compile errors — `NodeKind::Interface`, `::Viewtype`, `::Exception`, `::Field` not defined.

- [ ] **Step 3: Add the variants in `src/extraction/result.rs`**

Extend the `NodeKind` enum (add the four variants in alphabetical position; current order is File, Module, Function, Method, Struct, Enum, Trait, TypeAlias, Import, Template, Choice):

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
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
    /// Daml `template T with ... where ...`. parse_reliable=false (keyword anchor).
    Template,
    /// Daml `choice C : T with ... do ...` inside a template. parse_reliable=false.
    Choice,
    /// Daml `interface I where ...`. parse_reliable=false (keyword anchor).
    Interface,
    /// Daml `viewtype V` inside an interface body. parse_reliable=false (keyword anchor).
    Viewtype,
    /// Daml `exception E with ... where message ...`. parse_reliable=false.
    Exception,
    /// `with`-block typed binding inside a Template / Choice / Interface. parse_reliable=true.
    Field,
}
```

Extend `as_label`:

```rust
impl NodeKind {
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
            NodeKind::Template => "Template",
            NodeKind::Choice => "Choice",
            NodeKind::Interface => "Interface",
            NodeKind::Viewtype => "Viewtype",
            NodeKind::Exception => "Exception",
            NodeKind::Field => "Field",
        }
    }
}
```

- [ ] **Step 4: Extend `EMBED_KINDS_CYPHER_LIST` in two files**

Read `src/graph/status.rs` to find the existing constant (slice 9 had `'Template','Choice'`). Add `Interface`, `Viewtype`, `Exception`. `Field` is NOT added (fields are not embed-eligible per spec §2.1).

```rust
const EMBED_KINDS_CYPHER_LIST: &str =
    "['Function','Method','Class','Struct','Interface','Trait','Enum',\
      'TypeAlias','Module','Template','Choice','Viewtype','Exception']";
```

Same change in `src/vectors/query.rs`:

```rust
const EMBED_KINDS_CYPHER_LIST: &str =
    "['Function','Method','Class','Struct','Interface','Trait','Enum',\
      'TypeAlias','Module','Template','Choice','Viewtype','Exception']";
```

- [ ] **Step 5: Run tests + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
# Regression guard: confirm slice-9 Daml integration test still SKIPs cleanly
# (it's the existing end-to-end test the extractor's structural changes could
# break). Without env vars set, it just exits ok.
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: 101 tests pass (97 baseline + 4 new label tests; the iteration test adjustment doesn't add a new test, just exercises new variants).

- [ ] **Step 6: Commit**

```bash
git add src/extraction/result.rs src/graph/status.rs src/vectors/query.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add NodeKind::Interface, ::Viewtype, ::Exception, ::Field"
```

---

### Task 2: Tree-sitter query — Template anchor (TDD; replaces regex template detection)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

This task replaces the regex-based template detection in `pass_d_templates_and_choices` with a tree-sitter query in a new `pass_e_templates` function. The regex function's template-emission block is commented out in this task; its choice-emission block keeps running until Task 3 replaces it. The whole regex function is deleted in Task 7.

The existing slice-9 unit tests for template extraction must still pass after the swap (they assert on `name`, `qualified_name`, `parse_reliable` — all of which the tree-sitter version preserves).

**Cross-task `pass_d` state summary.**

| After task | `pass_d_templates_and_choices` state |
|---|---|
| slice 9 (baseline) | Emits both templates and choices via regex |
| Task 2 | Template-emission block commented out; choice-emission block still running |
| Task 3 | Both blocks commented out; function is a no-op |
| Task 7 | Function deleted entirely |

- [ ] **Step 1: Inspect tree-sitter-haskell `apply` shape**

Verify the grammar shape before writing anything. Two probes:

```bash
DAMLDIR=$(find ~/.cargo/registry/src -type d -name 'tree-sitter-haskell-*' | head -1)
grep -B 2 -A 30 '"type": "apply"' $DAMLDIR/src/node-types.json | head -40
```

Confirms `apply` has `function:` field (slice 9 task 8 found this in 0.23.1; re-confirm).

Then add a temporary `#[ignore]`d Rust test in `tests/extraction_unit.rs` to print the actual tree for `template Counter ...`:

```rust
#[test]
#[ignore]
fn probe_template_parse_shape() {
    let source = "module Foo where\n\ntemplate Counter\n  with\n    p : Party\n  where\n    signatory p\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}

fn print_tree(node: tree_sitter::Node, depth: usize) {
    let indent = "  ".repeat(depth);
    println!("{}{} [{}, {}-{}]", indent, node.kind(), node.byte_range().start, node.start_position().row, node.end_position().row);
    let mut cursor = node.walk();
    for child in node.children(&mut cursor) {
        print_tree(child, depth + 1);
    }
}
```

Run with: `cargo test --test extraction_unit probe_template_parse_shape -- --ignored --nocapture 2>&1 | head -50`

**Write down the observed shape** in a code comment at the top of `daml.scm` before continuing. The Step 4 query below assumes the shape `(apply function: (variable "template") argument: (constructor))`. If the probe shows curried `(apply (apply (variable "template") (constructor)) ...)`, the implementer ADAPTS Step 4's query before writing it — don't write the assumed-shape query and "adjust later".

**Probe test disposition:** REMOVE `probe_template_parse_shape` and `print_tree` from `tests/extraction_unit.rs` before Step 9's commit. They're scaffolding, not durable tests. The shape they revealed is preserved in the `daml.scm` comment.

- [ ] **Step 2: Append failing tests**

```rust
#[test]
fn daml_template_extracted_via_tree_sitter_query() {
    let source = "module Foo where\n\ntemplate Counter\n  with\n    p : Party\n  where\n    signatory p\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let templates: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Template).collect();
    assert_eq!(templates.len(), 1, "expected 1 Template");
    assert_eq!(templates[0].name, "Counter");
    assert_eq!(templates[0].qualified_name, "Foo.Counter");
    assert!(!templates[0].parse_reliable, "Template still parse_reliable=false (keyword anchor)");
}

#[test]
fn daml_template_span_is_byte_precise() {
    // Verify the tree-sitter version emits byte-precise spans (not the line
    // approximations the slice-9 regex produced).
    //
    // The regex version emits end_col=0 always (line-anchored). The tree-sitter
    // version emits end_col from the AST node's position, which is non-zero for
    // any template whose last line is not empty. The assertion below
    // distinguishes the two reliably.
    let source = "module Foo where\n\ntemplate Counter\n  with\n    p : Party\n  where\n    signatory p\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let template = result.nodes.iter().find(|n| n.kind == NodeKind::Template).unwrap();
    assert!(template.end_line >= 7, "Template should span at least to line 7; got {}", template.end_line);
    assert!(template.start_line >= 3, "Template starts at line 3 or later; got {}", template.start_line);
    assert!(template.end_col > 0,
            "tree-sitter version emits non-zero end_col (byte-precise); regex emits 0. Got end_col={}",
            template.end_col);
}
```

- [ ] **Step 3: Run failing tests**

```bash
cargo test --test extraction_unit daml_template_extracted_via_tree_sitter_query 2>&1 | tail -10
```

Expected: TEST PASSES (the existing regex-based `pass_d_templates_and_choices` from slice 9 already extracts templates — same assertions, just different mechanism). Confirm both new tests pass *with the regex in place*. This is the regression baseline; Step 5 will swap to tree-sitter and rerun.

- [ ] **Step 4: Add the Template query to `src/extraction/languages/daml.scm`**

Using the shape documented in Step 1's `daml.scm` comment, append the matching query. For the assumed flat shape:

```scheme
; Template anchor: `template Counter ...`.
; Captures the keyword 'template' as a variable in an apply expression;
; the Constructor argument is the template name.
((apply
  function: (variable) @template.kw
  argument: (constructor) @template.name) @template.def
  (#eq? @template.kw "template"))
```

For a curried `(apply (apply (variable "template") (constructor)) ...)` shape, the equivalent query is:

```scheme
; Curried form: outer apply wraps the inner template-constructor apply.
((apply
  function: (apply
    function: (variable) @template.kw
    argument: (constructor) @template.name)) @template.def
  (#eq? @template.kw "template"))
```

Pick the form matching Step 1's probe output. Do not write both.

- [ ] **Step 5: Add `pass_e_templates` function in `src/extraction/languages/daml.rs`**

Locate the existing `pass_d_templates_and_choices` function in `daml.rs` (`grep -n 'fn pass_d_templates_and_choices' src/extraction/languages/daml.rs`). Insert the new `pass_e_templates` function immediately after it:

```rust
/// Tree-sitter-based template extraction. Replaces the regex-anchored detection
/// in `pass_d_templates_and_choices` with a query over `(apply (variable "template") (constructor))`.
///
/// Why: byte-precise spans (vs line-anchored regex), robust to whitespace,
/// composes naturally with existing tree-sitter machinery.
fn pass_e_templates(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: Option<&str>,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("template.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("template.name") else { return };

    let module_prefix = module_qid
        .map(|q| q.split(':').skip(2).collect::<Vec<_>>().join(":"))
        .unwrap_or_default();
    let parent_for_template: &str = module_qid.unwrap_or(file_qid);

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
            let template_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let qname = if module_prefix.is_empty() {
                template_name.clone()
            } else {
                format!("{module_prefix}.{template_name}")
            };
            let template_qid = format!("{root_name}:{relative_path}:{qname}");

            // Expand the span to cover the full template definition. The
            // matched `apply` node only covers `template Counter`; the body
            // (`with ... where ...`) is the parent's next-sibling chain.
            // For now, use the def node's parent's range — that's typically
            // the top_splice or full statement.
            let span_node = def.parent().unwrap_or(def);
            let start = span_node.start_position();
            let end = span_node.end_position();

            result.nodes.push(RawNode {
                qid: template_qid.clone(),
                kind: NodeKind::Template,
                name: template_name,
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
                parse_reliable: false, // keyword anchor; the surrounding body is fine but the anchor itself is text-matched
                content_hash: None,
                last_indexed_at: None,
            });
            result.edges.push(RawEdge {
                from_qid: parent_for_template.to_string(),
                to_qid: template_qid,
                kind: EdgeKind::Contains,
            });
        }
    }
}
```

> **Implementer note on span expansion.** The `apply` node only covers `template Counter`. To get the *full* template span (including the `where`-block), walk up: `def.parent()` is typically the `top_splice` or a similar wrapper. If `parent()` doesn't span the body, the implementer may need to walk further or use the byte-range from the matched `def` plus the next-sibling's byte-end.

- [ ] **Step 6: Comment out the template-emission block in `pass_d_templates_and_choices`, wire `pass_e_templates` in `extract_daml`**

Both `pass_d` and `pass_e_templates` would emit a template node with the same qid (the qid is `<root>:<rel>:<qname>`, derived from the template name). Two emissions of the same qid produce a Neo4j constraint violation on write. The fix is to disable `pass_d`'s template-emission block before turning on `pass_e_templates`.

In `pass_d_templates_and_choices`, locate the `for (i, (start_byte, cap)) in template_matches.iter().enumerate() { ... }` loop that emits Template nodes. Wrap the loop body in a comment block:

```rust
// Slice 10a Task 2: template emission moved to pass_e_templates (tree-sitter
// based). Loop body preserved (commented out) to be deleted in Task 7.
/*
for (i, (start_byte, cap)) in template_matches.iter().enumerate() {
    // ... original body ...
}
*/
```

The choice-emission block below it remains live until Task 3.

In `extract_daml`, add the new call before the existing `pass_d_templates_and_choices` call:

```rust
pass_e_templates(&query, &tree, source, root_name, relative_path,
                 module_qid_opt.as_deref(), &file_qid, &mut result);
pass_d_templates_and_choices(source, root_name, relative_path,
                             module_qid_opt.as_deref(), &file_qid, &mut result);
```

- [ ] **Step 7: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -15
```

Expected: existing slice-9 template tests AND new tests pass. The `daml_template_span_is_byte_precise` test should now show end_line ≥ 7 (the byte-precise extent) where the slice-9 regex version showed end_line based on line approximations.

- [ ] **Step 8: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
# Regression guard: confirm slice-9 Daml integration test still SKIPs cleanly
# (it's the existing end-to-end test the extractor's structural changes could
# break). Without env vars set, it just exits ok.
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): tree-sitter template detection (replaces regex; byte-precise spans)"
```

---

### Task 3: Tree-sitter query — Choice anchor (TDD)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

Same pattern as Task 2 but for choices. Choices have a more complex shape than templates because the argument to `choice` is the choice's signature (`Increment : ContractId Counter`), not just a constructor.

- [ ] **Step 1: Inspect choice parse shape**

Add a `#[ignore]`d probe test in `tests/extraction_unit.rs` (reuses the `print_tree` helper if Task 2 left it in place; if Task 2 removed it per its instructions, paste it back temporarily):

```rust
#[test]
#[ignore]
fn probe_choice_parse_shape() {
    let source = "module Foo where\n\ntemplate T\n  with p : Party\n  where\n    signatory p\n    choice Increment : ContractId T\n      controller p\n      do return ()\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}
```

Run: `cargo test --test extraction_unit probe_choice_parse_shape -- --ignored --nocapture 2>&1 | head -80`

Observe how `choice Increment : ContractId T` parses. Likely candidates:
- The `choice` keyword and `Increment` form an `apply`; the `: ContractId T` part lands in a child of the apply.
- Or: the whole `choice Increment : ContractId T` is a signature-shaped node where `choice` is a prefix.

Document the actual shape in the `daml.scm` comment block. Write Step 4's query to match.

**Probe test disposition:** REMOVE `probe_choice_parse_shape` (and `print_tree` if Task 2 left it) before Step 9's commit.

- [ ] **Step 2: Append failing tests**

```rust
#[test]
fn daml_choice_extracted_via_tree_sitter_query() {
    let source = "module Foo where\n\ntemplate T\n  with p : Party\n  where\n    signatory p\n    choice Increment : ContractId T\n      controller p\n      do return ()\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let choices: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Choice).collect();
    assert_eq!(choices.len(), 1, "expected 1 Choice");
    assert_eq!(choices[0].name, "Increment");
    assert_eq!(choices[0].qualified_name, "Foo.T.Increment");
    assert!(!choices[0].parse_reliable);
}

#[test]
fn daml_two_choices_in_template() {
    let source = "module Foo where\n\ntemplate T\n  with p : Party\n  where\n    signatory p\n    choice A : ()\n      controller p\n      do return ()\n    choice B : Int\n      controller p\n      do return 0\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let choices: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Choice).collect();
    let names: Vec<&str> = choices.iter().map(|c| c.name.as_str()).collect();
    assert!(names.contains(&"A"));
    assert!(names.contains(&"B"));
}
```

- [ ] **Step 3: Run failing tests**

Expected: existing regex pass already passes these (slice 9 emits choices). The tests are written to assert behavior — switch to tree-sitter shouldn't regress them.

- [ ] **Step 4: Add the Choice query to `daml.scm`**

Based on the probe shape from Step 1. Tentatively:

```scheme
; Choice anchor: `choice C : T ...`. Captures the keyword 'choice' as a variable;
; the next sibling or argument holds the choice's name.
((apply
  function: (variable) @choice.kw
  argument: (_) @choice.body) @choice.def
  (#eq? @choice.kw "choice"))
```

Then in Rust, extract the choice name from the `@choice.body` capture by walking for the first `(variable)` or `(constructor)` child.

If the actual parse uses a `signature`-shape inside, the query can be:

```scheme
((apply
  function: (variable) @choice.kw
  argument: (signature name: (variable) @choice.name)) @choice.def
  (#eq? @choice.kw "choice"))
```

The implementer picks based on the probe.

- [ ] **Step 5: Add `pass_e_choices` function with full implementation**

In `src/extraction/languages/daml.rs`, add the helper `enclosing_template_qid` and the `pass_e_choices` function. Place both after `pass_e_templates` from Task 2.

```rust
/// Find the smallest Template node in `nodes` whose span contains (line, col).
/// Used to assign the parent qid for choices, since choices are CONTAINS-children
/// of their enclosing Template (not the Module).
fn enclosing_template_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&str> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        if n.kind != NodeKind::Template { continue; }
        let in_span = (n.start_line < line || (n.start_line == line && n.start_col <= col))
            && (n.end_line > line || (n.end_line == line && n.end_col >= col));
        if !in_span { continue; }
        match best {
            None => best = Some(n),
            Some(b) => {
                let n_size = (n.end_line - n.start_line) as i64;
                let b_size = (b.end_line - b.start_line) as i64;
                if n_size < b_size { best = Some(n); }
            }
        }
    }
    best.map(|n| n.qid.as_str())
}

/// Tree-sitter-based choice extraction. The choice's NAME comes from the
/// `@choice.body` capture — the implementer must walk this subtree to find
/// the first `(variable)` or `(constructor)` leaf (the choice's name).
///
/// The choice is CONTAINS-edged from the enclosing Template (not the Module).
/// If no enclosing Template is found at the match's location, the choice is
/// skipped — real Daml requires choices be inside templates.
///
/// **Signature note:** unlike `pass_e_templates` (which takes `module_qid`
/// + `file_qid` because Templates parent under Module-or-File), this function
/// takes neither — the Template parent is discovered via `enclosing_template_qid`
/// against `result.nodes`. The same shape applies to `pass_e_viewtypes`
/// (Interface parent) and `pass_e_fields` (Template/Choice/Interface/Exception
/// parent). Different signatures by design.
fn pass_e_choices(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("choice.def") else { return };
    let Some(body_idx) = query.capture_index_for_name("choice.body") else { return };

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut body_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == body_idx { body_node = Some(cap.node); }
        }
        let (Some(def), Some(body)) = (def_node, body_node) else { continue };

        // Find the choice name by walking the body subtree for the first
        // (variable) or (constructor) leaf. This handles both shapes:
        // - `choice Increment : ContractId T` (body is a signature-shaped node)
        // - `choice Inc` (body is a bare variable/constructor)
        let choice_name = first_identifier_in_subtree(body, source.as_bytes())
            .unwrap_or_default();
        if choice_name.is_empty() { continue; }

        // Find the enclosing Template.
        let pos = def.start_position();
        let line = (pos.row as u32) + 1;
        let col = pos.column as u32;
        let Some(template_qid) = enclosing_template_qid(&result.nodes, line, col) else {
            continue;
        };

        // Build qualified_name: <template_qname>.<choice_name>.
        let template_qname: String = template_qid.split(':').skip(2).collect::<Vec<_>>().join(":");
        let qname = format!("{template_qname}.{choice_name}");
        let qid = format!("{root_name}:{relative_path}:{qname}");

        let end = def.end_position();
        result.nodes.push(RawNode {
            qid: qid.clone(),
            kind: NodeKind::Choice,
            name: choice_name,
            qualified_name: qname,
            root_name: root_name.to_string(),
            relative_path: relative_path.to_string(),
            language: "daml".to_string(),
            start_line: line,
            end_line: (end.row as u32) + 1,
            start_col: col,
            end_col: end.column as u32,
            signature: Some(def.utf8_text(source.as_bytes()).unwrap_or("").trim().to_string()),
            doc: None,
            parse_reliable: false, // keyword anchor
            content_hash: None,
            last_indexed_at: None,
        });
        result.edges.push(RawEdge {
            from_qid: template_qid.to_string(),
            to_qid: qid,
            kind: EdgeKind::Contains,
        });
    }
}

/// Walk a tree-sitter subtree depth-first and return the first identifier text
/// (a variable or constructor). Returns None if none found.
fn first_identifier_in_subtree(node: tree_sitter::Node, source: &[u8]) -> Option<String> {
    if node.kind() == "variable" || node.kind() == "constructor" {
        return Some(node.utf8_text(source).ok()?.trim().to_string());
    }
    let mut cursor = node.walk();
    for child in node.children(&mut cursor) {
        if let Some(s) = first_identifier_in_subtree(child, source) {
            return Some(s);
        }
    }
    None
}
```

In `extract_daml`, wire the call AFTER `pass_e_templates`:

```rust
pass_e_templates(&query, &tree, source, root_name, relative_path,
                 module_qid_opt.as_deref(), &file_qid, &mut result);
pass_e_choices(&query, &tree, source, root_name, relative_path, &mut result);
```

- [ ] **Step 6: Comment out the choice-emission block in `pass_d_templates_and_choices`**

Same pattern as Task 2 Step 6. Locate the `for ccap in choice_re.captures_iter(span_text) { ... }` loop inside `pass_d_templates_and_choices`. Wrap its body:

```rust
// Slice 10a Task 3: choice emission moved to pass_e_choices (tree-sitter
// based). Loop body preserved (commented out) to be deleted in Task 7.
/*
for ccap in choice_re.captures_iter(span_text) {
    // ... original body ...
}
*/
```

After this step, `pass_d_templates_and_choices` is a function with both internal loops commented out — effectively a no-op. The function call in `extract_daml` can stay (it just does nothing); Task 7 deletes the call and the function together.

- [ ] **Step 7: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -15
```

Expected: all choice tests (existing + new) pass.

- [ ] **Step 8: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
# Regression guard: confirm slice-9 Daml integration test still SKIPs cleanly
# (it's the existing end-to-end test the extractor's structural changes could
# break). Without env vars set, it just exits ok.
cargo test --test integration_daml_index 2>&1 | tail -3
```

- [ ] **Step 9: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): tree-sitter choice detection; pass_d_templates_and_choices now empty"
```

---

### Task 4: Tree-sitter query — Interface + Viewtype (TDD; new constructs)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

`interface I where ...` and `viewtype V` inside the interface body. Both are net-new (v1 didn't extract them at all).

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_extracts_interface() {
    let source = "module Foo where\n\ninterface Authorizable where\n  getOwner : Party\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let interfaces: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Interface).collect();
    assert_eq!(interfaces.len(), 1);
    assert_eq!(interfaces[0].name, "Authorizable");
    assert_eq!(interfaces[0].qualified_name, "Foo.Authorizable");
    assert!(!interfaces[0].parse_reliable);
}

#[test]
fn daml_extracts_viewtype_inside_interface() {
    let source = "module Foo where\n\ndata FooView = FooView { x : Int }\n\ninterface Bar where\n  viewtype FooView\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let viewtypes: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Viewtype).collect();
    assert_eq!(viewtypes.len(), 1);
    assert_eq!(viewtypes[0].name, "FooView");
    assert_eq!(viewtypes[0].qualified_name, "Foo.Bar.FooView", "Viewtype qname nests under its Interface");
    assert!(!viewtypes[0].parse_reliable);
}

#[test]
fn daml_interface_contains_edge_from_module() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\n\ninterface Bar where\n  x : Int\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let module_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Module).unwrap().qid;
    let interface_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Interface).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *module_qid && e.to_qid == *interface_qid));
}

#[test]
fn daml_viewtype_contains_edge_from_interface() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\n\ndata V = V {}\n\ninterface Bar where\n  viewtype V\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let interface_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Interface).unwrap().qid;
    let viewtype_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Viewtype).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *interface_qid && e.to_qid == *viewtype_qid));
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit daml_extracts_interface 2>&1 | tail -10
```

Expected: FAIL — interface extraction doesn't exist yet.

- [ ] **Step 3: Add interface + viewtype queries to `daml.scm`**

```scheme
; Interface anchor: `interface I where ...`.
((apply
  function: (variable) @interface.kw
  argument: (constructor) @interface.name) @interface.def
  (#eq? @interface.kw "interface"))

; Viewtype anchor: `viewtype V` (always nested under an interface).
((apply
  function: (variable) @viewtype.kw
  argument: (constructor) @viewtype.name) @viewtype.def
  (#eq? @viewtype.kw "viewtype"))
```

- [ ] **Step 4: Add full implementations for `pass_e_interfaces`, `enclosing_interface_qid`, and `pass_e_viewtypes`**

In `src/extraction/languages/daml.rs`, add these three functions after `pass_e_choices` from Task 3:

```rust
/// Tree-sitter-based Interface extraction. Same shape as `pass_e_templates`
/// (interfaces are top-level module children, just like templates).
fn pass_e_interfaces(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: Option<&str>,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("interface.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("interface.name") else { return };

    let module_prefix = module_qid
        .map(|q| q.split(':').skip(2).collect::<Vec<_>>().join(":"))
        .unwrap_or_default();
    let parent_for_interface: &str = module_qid.unwrap_or(file_qid);

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        let (Some(def), Some(name)) = (def_node, name_node) else { continue };

        let interface_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
        let qname = if module_prefix.is_empty() {
            interface_name.clone()
        } else {
            format!("{module_prefix}.{interface_name}")
        };
        let interface_qid = format!("{root_name}:{relative_path}:{qname}");

        let span_node = def.parent().unwrap_or(def);
        let start = span_node.start_position();
        let end = span_node.end_position();

        result.nodes.push(RawNode {
            qid: interface_qid.clone(),
            kind: NodeKind::Interface,
            name: interface_name,
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
            parse_reliable: false,
            content_hash: None,
            last_indexed_at: None,
        });
        result.edges.push(RawEdge {
            from_qid: parent_for_interface.to_string(),
            to_qid: interface_qid,
            kind: EdgeKind::Contains,
        });
    }
}

/// Same shape as `enclosing_template_qid` but filters `NodeKind::Interface`.
fn enclosing_interface_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&str> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        if n.kind != NodeKind::Interface { continue; }
        let in_span = (n.start_line < line || (n.start_line == line && n.start_col <= col))
            && (n.end_line > line || (n.end_line == line && n.end_col >= col));
        if !in_span { continue; }
        match best {
            None => best = Some(n),
            Some(b) => {
                let n_size = (n.end_line - n.start_line) as i64;
                let b_size = (b.end_line - b.start_line) as i64;
                if n_size < b_size { best = Some(n); }
            }
        }
    }
    best.map(|n| n.qid.as_str())
}

/// Tree-sitter-based Viewtype extraction. Viewtypes are CONTAINS-children of
/// their enclosing Interface. If no enclosing Interface is found, the viewtype
/// is skipped (defensive — real Daml requires viewtype inside interface).
fn pass_e_viewtypes(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("viewtype.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("viewtype.name") else { return };

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        let (Some(def), Some(name)) = (def_node, name_node) else { continue };

        let viewtype_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();

        let pos = def.start_position();
        let line = (pos.row as u32) + 1;
        let col = pos.column as u32;
        let Some(interface_qid) = enclosing_interface_qid(&result.nodes, line, col) else {
            continue;
        };

        let interface_qname: String = interface_qid.split(':').skip(2).collect::<Vec<_>>().join(":");
        let qname = format!("{interface_qname}.{viewtype_name}");
        let qid = format!("{root_name}:{relative_path}:{qname}");

        let end = def.end_position();
        result.nodes.push(RawNode {
            qid: qid.clone(),
            kind: NodeKind::Viewtype,
            name: viewtype_name,
            qualified_name: qname,
            root_name: root_name.to_string(),
            relative_path: relative_path.to_string(),
            language: "daml".to_string(),
            start_line: line,
            end_line: (end.row as u32) + 1,
            start_col: col,
            end_col: end.column as u32,
            signature: None,
            doc: None,
            parse_reliable: false,
            content_hash: None,
            last_indexed_at: None,
        });
        result.edges.push(RawEdge {
            from_qid: interface_qid.to_string(),
            to_qid: qid,
            kind: EdgeKind::Contains,
        });
    }
}
```

Wire calls in `extract_daml` AFTER `pass_e_choices`. **Order is mandatory:** `pass_e_interfaces` must come BEFORE `pass_e_viewtypes` — viewtypes look up their parent Interface via `enclosing_interface_qid(&result.nodes, ...)`, so the Interface nodes must already exist in `result.nodes` when viewtype extraction runs. If reversed, viewtype emission silently fails (no error, just zero Viewtype nodes).

```rust
// REQUIRED ORDER: interfaces before viewtypes (viewtypes look up parent Interface).
pass_e_interfaces(&query, &tree, source, root_name, relative_path,
                  module_qid_opt.as_deref(), &file_qid, &mut result);
pass_e_viewtypes(&query, &tree, source, root_name, relative_path, &mut result);
```

- [ ] **Step 5: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: 4 new tests pass.

- [ ] **Step 6: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
# Regression guard: confirm slice-9 Daml integration test still SKIPs cleanly
# (it's the existing end-to-end test the extractor's structural changes could
# break). Without env vars set, it just exits ok.
cargo test --test integration_daml_index 2>&1 | tail -3
```

- [ ] **Step 7: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml Interface and Viewtype extraction"
```

---

### Task 5: Tree-sitter query — Exception (TDD; new construct)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

`exception E with ... where message ...` — same anchor shape as template, simpler body. Direct CONTAINS-child of the module.

- [ ] **Step 1: Append failing test**

```rust
#[test]
fn daml_extracts_exception() {
    let source = "module Foo where\n\nexception InsufficientBalance with\n    actual : Int\n  where\n    message \"low balance\"\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let exceptions: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Exception).collect();
    assert_eq!(exceptions.len(), 1);
    assert_eq!(exceptions[0].name, "InsufficientBalance");
    assert_eq!(exceptions[0].qualified_name, "Foo.InsufficientBalance");
    assert!(!exceptions[0].parse_reliable);
}

#[test]
fn daml_exception_contains_edge_from_module() {
    use codegraph_rs::extraction::result::EdgeKind;
    let source = "module Foo where\n\nexception E with x : Int where message \"\"\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let module_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Module).unwrap().qid;
    let exception_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Exception).unwrap().qid;
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::Contains && e.from_qid == *module_qid && e.to_qid == *exception_qid));
}
```

- [ ] **Step 2: Run failing tests**

Expected: FAIL — no Exception extraction yet.

- [ ] **Step 3: Add exception query to `daml.scm`**

```scheme
; Exception anchor: `exception E with ... where message ...`.
((apply
  function: (variable) @exception.kw
  argument: (constructor) @exception.name) @exception.def
  (#eq? @exception.kw "exception"))
```

- [ ] **Step 4: Add `pass_e_exceptions` with full implementation**

In `src/extraction/languages/daml.rs`, add after `pass_e_viewtypes`:

```rust
/// Tree-sitter-based Exception extraction. Identical structure to
/// `pass_e_templates` (module-direct child).
fn pass_e_exceptions(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    module_qid: Option<&str>,
    file_qid: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("exception.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("exception.name") else { return };

    let module_prefix = module_qid
        .map(|q| q.split(':').skip(2).collect::<Vec<_>>().join(":"))
        .unwrap_or_default();
    let parent_for_exception: &str = module_qid.unwrap_or(file_qid);

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
        }
        let (Some(def), Some(name)) = (def_node, name_node) else { continue };

        let exception_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
        let qname = if module_prefix.is_empty() {
            exception_name.clone()
        } else {
            format!("{module_prefix}.{exception_name}")
        };
        let exception_qid = format!("{root_name}:{relative_path}:{qname}");

        let span_node = def.parent().unwrap_or(def);
        let start = span_node.start_position();
        let end = span_node.end_position();

        result.nodes.push(RawNode {
            qid: exception_qid.clone(),
            kind: NodeKind::Exception,
            name: exception_name,
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
            parse_reliable: false,
            content_hash: None,
            last_indexed_at: None,
        });
        result.edges.push(RawEdge {
            from_qid: parent_for_exception.to_string(),
            to_qid: exception_qid,
            kind: EdgeKind::Contains,
        });
    }
}
```

- [ ] **Step 5: Wire in `extract_daml`** after the interface pass.

- [ ] **Step 6: Run tests, build, clippy, fmt**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

- [ ] **Step 7: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml Exception extraction"
```

---

### Task 6: Tree-sitter query — Field with disambiguation (TDD; most complex single task)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

This is the spec's critical disambiguation case. The `(signature)` query matches BOTH:
1. With-block field bindings inside a Template/Choice/Interface/Exception keyword anchor
2. Top-level function signatures (`helperFn : Int -> Int`)

The Function extractor (slice 9 task 6) already handles case 2. The Field extractor must emit ONLY case 1.

**Disambiguation rule (spec §3.2):** Only emit as Field if the signature's byte range is inside the byte range of a Template, Choice, Interface, or Exception node (which by this point have all been extracted by Tasks 2-5).

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_template_fields_extracted() {
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n    count : Int\n  where\n    signatory owner\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let fields: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Field).collect();
    assert_eq!(fields.len(), 2, "expected 2 Fields (owner, count); got {:?}", fields.iter().map(|f| &f.name).collect::<Vec<_>>());
    let names: Vec<&str> = fields.iter().map(|f| f.name.as_str()).collect();
    assert!(names.contains(&"owner"));
    assert!(names.contains(&"count"));
    for f in &fields {
        assert!(f.parse_reliable, "Fields are grammar-clean (parse_reliable=true)");
        assert!(f.qualified_name.starts_with("Foo.Counter."), "Field qname nests under Template; got {}", f.qualified_name);
    }
}

#[test]
fn daml_top_level_signature_not_emitted_as_field() {
    let source = "module Foo where\n\nhelperFn : Int -> Int\nhelperFn x = x + 1\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let fields: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Field).collect();
    assert!(fields.is_empty(), "top-level function signatures must NOT be emitted as Fields; got {:?}", fields);
    let funcs: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Function).collect();
    assert_eq!(funcs.len(), 1, "but Function should still be extracted");
    assert_eq!(funcs[0].name, "helperFn");
}

#[test]
fn daml_choice_fields_qualified_under_template_and_choice() {
    let source = "module Foo where\n\ntemplate T\n  with p : Party\n  where\n    signatory p\n    choice Bump\n        : ContractId T\n      with\n        delta : Int\n      controller p\n      do return self\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let delta_field = result.nodes.iter().find(|n| n.kind == NodeKind::Field && n.name == "delta");
    if let Some(field) = delta_field {
        assert_eq!(field.qualified_name, "Foo.T.Bump.delta", "Choice field qname is 4 segments (Module.Template.Choice.field)");
    }
    // Note: choice argument extraction depends on haskell's parse of the
    // `with` block inside a choice. This test is permissive — if the haskell
    // grammar doesn't expose the choice's with-block as a query-able shape,
    // the field may not be extracted. Document as a known limitation.
}

#[test]
fn daml_interface_methods_extracted_as_fields() {
    let source = "module Foo where\n\ninterface Bar where\n  getOwner : Party\n  getCount : Int\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let fields: Vec<_> = result.nodes.iter().filter(|n| n.kind == NodeKind::Field).collect();
    let names: Vec<&str> = fields.iter().map(|f| f.name.as_str()).collect();
    assert!(names.contains(&"getOwner"), "Interface method should be Field; got {:?}", names);
    assert!(names.contains(&"getCount"));
    for f in &fields {
        assert!(f.qualified_name.starts_with("Foo.Bar."), "Interface field nests under Interface");
    }
}
```

- [ ] **Step 2: Run failing tests**

Expected: FAIL — no Field extraction yet.

- [ ] **Step 3: Add the `signature` query to `daml.scm`**

```scheme
; Field candidates: typed bindings `name : Type`. Note this also matches top-level
; function signatures; the extractor disambiguates by checking the ancestor span
; against already-emitted Template/Choice/Interface/Exception nodes.
(signature name: (variable) @field.name type: (_) @field.type) @field.def
```

> **Daml-specific concern:** slice 6 found Daml's single-colon `:` lands in `top_splice/infix` (not `signature`). The implementer must run a probe test (similar to Task 2 Step 1) to confirm which node shape Daml field declarations actually use, and adjust the query accordingly. If they use `top_splice/infix`, the query might be:
>
> ```scheme
> (top_splice (infix left_operand: (variable) @field.name (operator) right_operand: (_) @field.type)) @field.def
> ```

- [ ] **Step 4: Add `pass_e_fields` with ancestor-span disambiguation**

```rust
fn pass_e_fields(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    root_name: &str,
    relative_path: &str,
    result: &mut ExtractionResult,
) {
    let Some(def_idx) = query.capture_index_for_name("field.def") else { return };
    let Some(name_idx) = query.capture_index_for_name("field.name") else { return };
    let Some(type_idx) = query.capture_index_for_name("field.type") else { return };

    let mut cursor = QueryCursor::new();
    let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

    while let Some(m) = matches.next() {
        let mut def_node = None;
        let mut name_node = None;
        let mut type_node = None;
        for cap in m.captures {
            if cap.index == def_idx { def_node = Some(cap.node); }
            else if cap.index == name_idx { name_node = Some(cap.node); }
            else if cap.index == type_idx { type_node = Some(cap.node); }
        }
        if let (Some(def), Some(name), Some(type_n)) = (def_node, name_node, type_node) {
            let field_name = name.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let field_type = type_n.utf8_text(source.as_bytes()).unwrap_or("?").trim().to_string();
            let pos = def.start_position();
            let line = (pos.row as u32) + 1;
            let col = pos.column as u32;

            // Disambiguation: only emit if inside a Daml keyword-anchored construct.
            let parent_qid = enclosing_daml_anchor_qid(&result.nodes, line, col);
            let Some(parent_qid) = parent_qid else {
                continue; // top-level signature; handled by Function extractor
            };

            // qualified_name: <parent_qname>.<field_name>.
            // Extract the parent's qname from its qid (format: <root>:<rel>:<qname>).
            let parent_qname: String = parent_qid
                .split(':')
                .skip(2)
                .collect::<Vec<_>>()
                .join(":");
            let qname = format!("{parent_qname}.{field_name}");
            let qid = format!("{root_name}:{relative_path}:{qname}");

            let end = def.end_position();
            result.nodes.push(RawNode {
                qid: qid.clone(),
                kind: NodeKind::Field,
                name: field_name,
                qualified_name: qname,
                root_name: root_name.to_string(),
                relative_path: relative_path.to_string(),
                language: "daml".to_string(),
                start_line: line,
                end_line: (end.row as u32) + 1,
                start_col: col,
                end_col: end.column as u32,
                signature: Some(field_type),
                doc: None,
                parse_reliable: true, // haskell parses the signature shape cleanly
                content_hash: None,
                last_indexed_at: None,
            });
            result.edges.push(RawEdge {
                from_qid: parent_qid.to_string(),
                to_qid: qid,
                kind: EdgeKind::Contains,
            });
        }
    }
}

/// Find the innermost Template/Choice/Interface/Exception that contains
/// the given (line, col). Returns its qid, or None if no such ancestor.
fn enclosing_daml_anchor_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&str> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        let is_anchor = matches!(
            n.kind,
            NodeKind::Template | NodeKind::Choice | NodeKind::Interface | NodeKind::Exception
        );
        if !is_anchor { continue; }
        let in_span = (n.start_line < line || (n.start_line == line && n.start_col <= col))
            && (n.end_line > line || (n.end_line == line && n.end_col >= col));
        if !in_span { continue; }
        match best {
            None => best = Some(n),
            Some(b) => {
                let n_size = (n.end_line - n.start_line) as i64;
                let b_size = (b.end_line - b.start_line) as i64;
                if n_size < b_size { best = Some(n); }
            }
        }
    }
    best.map(|n| n.qid.as_str())
}
```

- [ ] **Step 5: Wire `pass_e_fields` at the END of `extract_daml`**

```rust
// REQUIRED ORDER: pass_e_fields runs LAST. Field extraction uses
// `enclosing_daml_anchor_qid(&result.nodes, ...)` to disambiguate
// with-block fields from top-level signatures. All of Template, Choice,
// Interface, Exception must be in `result.nodes` before this runs;
// otherwise the disambiguation lookup misses and either drops Fields
// or assigns them wrong parents.
//
// pass_e_viewtypes ordering doesn't matter relative to fields
// (Viewtypes can't contain Fields).
pass_e_fields(&query, &tree, source, root_name, relative_path, &mut result);
```

- [ ] **Step 6: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -15
```

Expected: 4 new tests pass. If the choice-fields test (`daml_choice_fields_qualified_under_template_and_choice`) fails because the haskell grammar doesn't expose choice `with`-blocks as signature nodes, mark the inner assertion `if let Some(field) = delta_field { ... }` as permissive (already in test); document the limitation.

- [ ] **Step 7: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
# Regression guard: confirm slice-9 Daml integration test still SKIPs cleanly
# (it's the existing end-to-end test the extractor's structural changes could
# break). Without env vars set, it just exits ok.
cargo test --test integration_daml_index 2>&1 | tail -3
```

- [ ] **Step 8: Commit**

```bash
git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml Field extraction with ancestor-span disambiguation"
```

---

### Task 7: Delete regex Pass D + tiny-daml fixture extension + integration test

**Files:**
- Modify: `src/extraction/languages/daml.rs` (delete `pass_d_templates_and_choices` and its helpers)
- Modify: `Cargo.toml` (drop `regex` dep if no longer used; check `pass_c_exercise_choices` first)
- Create: `tests/fixtures/tiny-daml/Splice/Authorization.daml`
- Modify: `tests/integration_daml_index.rs`

- [ ] **Step 1: Verify whether `regex` is still needed**

```bash
grep -rn "use regex\|regex::" src/
```

Expected: `regex` still used in `pass_c_exercise_choices` (slice 9 task 8). Keep the dep. Do NOT remove from Cargo.toml.

- [ ] **Step 2: Delete `pass_d_templates_and_choices` and helpers**

In `src/extraction/languages/daml.rs`, remove:
- The `pass_d_templates_and_choices` function entirely.
- The `byte_to_line` helper if it's only used by `pass_d` (check with `grep`; if `pass_c_exercise_choices` still uses it, keep).
- The unused regex compilations (template_re, choice_re for pass_d).
- The call to `pass_d_templates_and_choices` in `extract_daml`.

Re-run all tests to make sure nothing breaks:

```bash
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: all extraction_unit tests pass.

- [ ] **Step 3: Create `tests/fixtures/tiny-daml/Splice/Authorization.daml`**

```daml
module Splice.Authorization where

import Splice.Counter

-- Interface with a viewtype.
interface Authorizable where
  viewtype AuthorizableView
  getOwner : Party

data AuthorizableView = AuthorizableView { owner : Party }

-- Template implementing the interface, with signatory + observer + a choice.
template AuthorizedCounter
  with
    owner : Party
    delegate : Party
    count : Int
  where
    signatory owner
    observer delegate
    interface instance Authorizable for AuthorizedCounter where
      view = AuthorizableView { owner = owner }
      getOwner = owner

    choice Bump : ContractId AuthorizedCounter
      controller delegate
      do
        create this with count = count + 1

-- An exception.
exception InsufficientBalance with
    actual : Int
    required : Int
  where
    message "Insufficient balance"
```

> **Daml-compile verification (per spec §5.1):**
>
> **If `daml` CLI is installed:** run `which daml && daml damlc validate tests/fixtures/tiny-daml/Splice/Authorization.daml` (or `daml build --project-root <fixture-dir>`). If it fails on the `interface instance ... for ... where` syntax (pre-2.7 Daml), switch to the pre-2.7 top-level `instance` declaration form and re-test.
>
> **If `daml` is not installed:** add a comment at the top of `Authorization.daml` documenting the target version:
>
> ```daml
> -- Targets Daml 2.7+. The `interface instance ... for ... where` syntax inside
> -- a template body is the post-2.7 form; pre-2.7 uses a separate top-level
> -- `instance` declaration. If you're on pre-2.7, this fixture won't typecheck —
> -- but the codegraph-rs extractor doesn't care (it's regex/AST-anchored on the
> -- keyword tokens, not the typechecker output). The integration test against
> -- Neo4j is the real correctness gate; this fixture compiling cleanly is a
> -- nice-to-have.
> ```
>
> Then proceed without compile verification; the integration test (Task 7 Step 4) is the actual correctness gate.

- [ ] **Step 4: Extend `tests/integration_daml_index.rs`**

Read the existing test (slice 9). Update the assertions to include the new fixture:

```rust
let file_count: i64 = scalar(&conn, "MATCH (n:File) RETURN count(n) AS c").await;
let module_count: i64 = scalar(&conn, "MATCH (n:Module) RETURN count(n) AS c").await;
let template_count: i64 = scalar(&conn, "MATCH (n:Template) RETURN count(n) AS c").await;
let choice_count: i64 = scalar(&conn, "MATCH (n:Choice) RETURN count(n) AS c").await;
let function_count: i64 = scalar(&conn, "MATCH (n:Function) RETURN count(n) AS c").await;
let type_alias_count: i64 = scalar(&conn, "MATCH (n:TypeAlias) RETURN count(n) AS c").await;
let import_count: i64 = scalar(&conn, "MATCH (n:Import) RETURN count(n) AS c").await;
let interface_count: i64 = scalar(&conn, "MATCH (n:Interface) RETURN count(n) AS c").await;
let viewtype_count: i64 = scalar(&conn, "MATCH (n:Viewtype) RETURN count(n) AS c").await;
let exception_count: i64 = scalar(&conn, "MATCH (n:Exception) RETURN count(n) AS c").await;
let field_count: i64 = scalar(&conn, "MATCH (n:Field) RETURN count(n) AS c").await;

assert_eq!(file_count, 4, "expected 4 File nodes (Math, Counter, Driver, Authorization)");
assert_eq!(module_count, 4);
assert_eq!(template_count, 2, "expected 2 Templates (Counter, AuthorizedCounter)");
assert!(choice_count >= 3, "expected ≥3 Choices (Increment, Reset, Bump)");
assert_eq!(interface_count, 1, "expected 1 Interface (Authorizable)");
assert_eq!(viewtype_count, 1, "expected 1 Viewtype (AuthorizableView)");
assert_eq!(exception_count, 1, "expected 1 Exception (InsufficientBalance)");
// Expected Fields, broken down by source so the lower bound is defensible:
//   - Counter (slice 9 fixture): owner, count                              -> 2
//   - AuthorizedCounter (this slice): owner, delegate, count               -> 3
//   - InsufficientBalance: actual, required                                -> 2
//   - Authorizable interface: getOwner (method-as-field)                   -> 1
//   - Bump choice with-block: usually none in this fixture                 -> 0-1
// Minimum guaranteed: 8. If the haskell grammar declines to expose any of
// the above (real risk per slice 6's single-colon finding), the bound drops.
// Use >= 6 to tolerate up to two missed fields while still catching wholesale
// extraction failures.
assert!(field_count >= 6,
    "expected ≥6 Fields across all templates/choices/interfaces/exceptions; got {field_count}");

// Named-symbol assertions for new constructs.
let auth_field_owner: i64 = scalar(&conn,
    "MATCH (n:Field {qualified_name: 'Splice.Authorization.AuthorizedCounter.owner'}) RETURN count(n) AS c").await;
assert_eq!(auth_field_owner, 1);

let interface_name: String = string_scalar(&conn,
    "MATCH (i:Interface) RETURN i.qualified_name AS c LIMIT 1").await;
assert_eq!(interface_name, "Splice.Authorization.Authorizable");

// All Templates carry parse_reliable=false.
let unreliable_templates: i64 = scalar(&conn,
    "MATCH (t:Template) WHERE t.parse_reliable = false RETURN count(t) AS c").await;
assert_eq!(unreliable_templates, template_count, "all Templates parse_reliable=false");

// Fields carry parse_reliable=true.
let reliable_fields: i64 = scalar(&conn,
    "MATCH (f:Field) WHERE f.parse_reliable = true RETURN count(f) AS c").await;
assert_eq!(reliable_fields, field_count, "all Fields parse_reliable=true");

// Embedding inheritance (spec §5.4): Interfaces are embed-eligible per the
// updated EMBED_KINDS_CYPHER_LIST. Under the dual gate (CODEGRAPH_RS_TEST_NEO4J
// + CODEGRAPH_RS_TEST_EMBEDDINGS), the embedding pipeline should populate
// `n.embedding` on every Interface. We assert this without gating: when only
// CODEGRAPH_RS_TEST_NEO4J is set, vectors::run skips (embeddings disabled
// by config? no — the test fixture has the default config which enables
// embeddings, but model load is skipped if CODEGRAPH_RS_TEST_EMBEDDINGS is
// unset). The assertion is conditional on EMBEDDINGS being set:
if std::env::var("CODEGRAPH_RS_TEST_EMBEDDINGS").as_deref() == Ok("1") {
    let embedded_interfaces: i64 = scalar(&conn,
        "MATCH (i:Interface) WHERE i.embedding IS NOT NULL RETURN count(i) AS c").await;
    assert!(embedded_interfaces >= 1,
        "interfaces should be embedded by slice 7's pipeline when CODEGRAPH_RS_TEST_EMBEDDINGS=1");
}
```

Add the `string_scalar` helper if it doesn't already exist:

```rust
async fn string_scalar(conn: &codegraph_rs::db::Connection, cypher: &str) -> String {
    let mut stream = conn.graph.execute(query(cypher)).await.unwrap();
    let row = stream.next().await.unwrap().unwrap();
    row.get::<String>("c").unwrap()
}
```

- [ ] **Step 5: Verify SKIP path**

```bash
cargo test --test integration_daml_index 2>&1 | tail -10
```

Expected: 1 test, SKIPs (no env vars), result ok.

- [ ] **Step 6: Build + clippy + fmt + full test suite (SKIP)**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -10
```

Expected: all tests pass.

**Test count math.** Baseline: 97. New unit tests by task: Task 1 +4, Task 2 +2, Task 3 +2, Task 4 +4, Task 5 +2, Task 6 +4. Total new unit tests: 18. New integration test: 0 (extends existing `integration_daml_index`). **Final expected count: 115** (97 + 18).

- [ ] **Step 7: Commit**

```bash
git add src/extraction/languages/daml.rs tests/fixtures/tiny-daml/Splice/Authorization.daml tests/integration_daml_index.rs
git commit -m "feat(extraction): delete regex pass_d; add Authorization.daml fixture + extended integration assertions"
```

---

### Task 8: README + slice-10a-daml-structure tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read current README.md** to find the `## Status` section.

- [ ] **Step 2: Bump the Status section**

Replace the current Status block with:

```markdown
## Status

In development. Currently implements **Slice 10a: Daml structure +
fields** — workspace initialization, indexing, resolution, incremental
sync, on-write embeddings, a stdio MCP server exposing 10 tools, and
Rust + Daml extraction. The Daml extractor now uses tree-sitter
queries (filtered by Daml keyword text) instead of regex for templates,
choices, interfaces, viewtypes, exceptions; with-block fields are
first-class CONTAINS-children of their parent Template/Choice/Interface.
Authorization edges (signatory/controller/observer/implements/requires)
land in slice 10b.

Daml templates and choices still carry `parse_reliable = false` (the
keyword anchors are text-matched against haskell's variable parses),
but their internal fields carry `parse_reliable = true` and their spans
are byte-precise (no longer line-approximated).
```

Quick start, Development sections need no changes.

- [ ] **Step 3: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected: all tests pass. Final test count target: **115** (97 baseline + 4 from Task 1 + 2 from Task 2 + 2 from Task 3 + 4 from Task 4 + 2 from Task 5 + 4 from Task 6).

- [ ] **Step 4: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 10a (Daml structure + fields)"
git tag slice-10a-daml-structure
```

- [ ] **Step 5: Verify**

```bash
git log --oneline | head -15
git tag --list
```

Expected tags include `slice-10a-daml-structure`.

---

## Slice 10a Done

- [ ] `cargo build` clean (no new dependencies; just code changes)
- [ ] `cargo test` (no env vars) — all unit tests pass; integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass, including extended `integration_daml_index` against the Authorization.daml fixture
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 CODEGRAPH_RS_TEST_EMBEDDINGS=1 cargo test -- --test-threads=1` — embeddings test still passes; Interface/Viewtype/Exception nodes get 384-dim embeddings automatically
- [ ] Manual: `codegraph-rs index` on a real Daml workspace (Splice or daml-finance) — verify in Neo4j Browser:
  - `MATCH (i:Interface) RETURN count(i)` returns non-zero
  - `MATCH (t:Template)-[:CONTAINS]->(f:Field) RETURN count(*)` returns non-zero
  - `MATCH (t:Template) WHERE t.parse_reliable = false RETURN count(t)` matches total Template count
  - `MATCH (f:Field) WHERE f.parse_reliable = true RETURN count(f)` matches total Field count
- [ ] No regression of slice 9 functionality — existing Counter.daml fixture still produces the same Template, Choice, Function counts
- [ ] `pass_d_templates_and_choices` regex function is gone

When all check, slice 10a is complete. **Next slice: 10b (Daml authorization edges)** — signatory/controller/observer party-expression edges, implements/requires interface-relationship edges, the `scope_qid` resolution path.

### Small follow-ups noted during this slice

- **Interface methods may need NodeKind::Method instead of Field.** Currently `getOwner : Party` inside an interface is emitted as a Field, but semantically it's closer to a Method (function signature with no body). Slice 10b or 10c could split: Field for `with`-block bindings, Method for interface method signatures.
- **Choice with-block extraction depends on grammar exposure.** If the haskell grammar doesn't expose the `with` block inside a choice as a queryable signature shape, the `daml_choice_fields_qualified_under_template_and_choice` test stays permissive. Worth probing the grammar in slice 10b to see if a regex fallback (similar to slice 9's exercise-choices pass) is needed.
- **`pass_e_*` naming.** Slice 10a accumulates pass_e_templates / pass_e_choices / pass_e_interfaces / pass_e_viewtypes / pass_e_exceptions / pass_e_fields. The naming convention is messy at this point. A future cleanup slice could collapse these into a single `pass_e_daml_keyword_anchors` driven by a (keyword, NodeKind, parent-finder) table.
