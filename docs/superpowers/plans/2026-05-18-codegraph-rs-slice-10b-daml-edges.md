# codegraph-rs Slice 10b — Daml authorization edges

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract `signatory`/`controller`/`observer` party expressions, `implements` declarations, and `requires` declarations from Daml templates and interfaces. Resolve identifier-shaped operands to the Field Symbols of their enclosing Template (preferred) or workspace-wide Symbols (fallback). Add 5 new EdgeKinds (`SIGNATORY`, `CONTROLLER`, `OBSERVER`, `IMPLEMENTS`, `REQUIRES`), 5 new `UnresolvedKind` variants, a new `scope_qid` field on `UnresolvedRef`, and a new resolution Phase A.5 (scoped name match).

**Architecture:** Mirror slice 10a's tree-sitter-and-tree-walk pattern: keyword-anchored `(apply function: (variable "signatory") argument: <expr>)` matches → walk the argument subtree for identifier leaves, filter wrapper names (`Set.fromList`, `fromOptional`, `$`, etc.) → emit one `UnresolvedRef` per identifier with `kind` set to the appropriate edge type, `source_qid` = enclosing Template/Choice/Interface, `scope_qid` = enclosing Template. Resolution gains a Phase A.5 between Phase A and Phase B that builds a `HashMap<qualified_name, qid>` once and resolves party refs by looking up `<scope_qname>.<unresolved_name>`. Falls through to Phase C on miss.

**Tech Stack additions:** None. All primitives (tree-sitter, neo4rs, regex) are already in tree.

**Source spec:** [`docs/superpowers/specs/2026-05-15-codegraph-rs-v2-daml-depth-design.md`](../specs/2026-05-15-codegraph-rs-v2-daml-depth-design.md) §2.2, §2.3, §3.3, §4, §5, §7.2.

**Slice 10a baseline:** tag `slice-10a-daml-structure` at HEAD `d3df454` on `main`. 118 unit tests pass; live integration test against Neo4j passes (4 daml files → 31 nodes, 51 edges).

---

## What this slice does, conceptually

1. **Add 5 new `EdgeKind` variants** (SIGNATORY, CONTROLLER, OBSERVER, IMPLEMENTS, REQUIRES). The `as_relationship_type()` method returns the Cypher names. The Neo4j schema is unchanged — relationship types are open.

2. **Add 5 new `UnresolvedKind` variants** matching the new edges. Stored as a string in `UnresolvedRef.kind`.

3. **Add `scope_qid: Option<String>` to `UnresolvedRef`.** For party-edge refs (Signatory/Controller/Observer), `scope_qid` is the enclosing Template's qid. The write path (`src/db/write.rs::write_unresolved_refs`) gains a new column. Old v1 records lack the property; reading returns `None` — additive at the Neo4j layer.

4. **New extractor passes** in `src/extraction/languages/daml.rs`:
   - `pass_e_party_edges` — handles signatory/controller/observer keyword anchors. Walks the argument subtree for identifier leaves, filters known wrapper names, emits one `UnresolvedRef` per leaf.
   - `pass_e_interface_edges` — handles `implements` and `requires` keyword anchors. Single-constructor argument; emits one `UnresolvedRef` per match.

5. **New resolution phase A.5** in `src/resolution/`:
   - `phase_a5_scoped_match.rs` (new file). Builds a `HashMap<qualified_name, qid>` from `MATCH (s:Symbol) RETURN s.qualified_name, s.qid`. For each `UnresolvedRef` with `scope_qid: Some(s)` AND `kind` in {Signatory, Controller, Observer, Implements, Requires}, looks up `<scope_qname>.<unresolved_name>`. On hit, emits the corresponding edge. On miss, leaves the ref for Phase C (name-match fallback) to handle for party kinds; Implements/Requires don't fall back.

6. **Resolution orchestrator wiring** (`src/resolution/mod.rs`): Phase A.5 runs between Phase A and Phase B. `ResolutionStats` gains counters for the new edge kinds.

7. **Integration test extensions** assert SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS edges land on the Authorization.daml fixture (REQUIRES is exercised by a small fixture addition since the current fixture doesn't have interface-requires-interface).

8. **README bump + tag**.

---

## Out of scope for slice 10b

- `interface instance ... for ... where` body extraction (the `view = ...` and method bodies inside an instance). The `interface instance` keyword DOES anchor an `IMPLEMENTS` edge; the body is not parsed.
- `agreement`, `ensure`, `message` (template/exception body expressions) — keyword anchors only; future slice.
- `key`/`maintainer` — deprecated in Daml 2.x, dropped per spec §1.3.
- Resolution Phase C tightening for party kinds — current Phase C name-match handles cross-scope party refs adequately for v2.0.
- Cross-package interface resolution (Daml multi-DAR scenarios).

---

## Critical implementation notes from slice 10a

- The Daml extractor uses a mix of tree-sitter queries (`daml.scm`) and pure Rust tree walks (`pass_e_fields` discovered the parse shapes were too irregular for a clean SCM pattern).
- A `PassCtx<'_>` struct consolidates common parameters (root_name, relative_path, file_qid, etc.) — verify the actual signature with `grep "struct PassCtx" src/extraction/languages/daml.rs` before writing new passes.
- Helpers available: `enclosing_template_qid`, `enclosing_interface_qid`, `enclosing_daml_anchor_qid`, `first_identifier_in_subtree`.
- The grammar parses Daml keywords as `(variable)` nodes inside ERROR-wrapped contexts. Templates have 3 parse shapes (A, B, C). Interfaces have 2 (simple ERROR-direct, nested apply chains). Expect signatory/controller/observer to also have multiple shapes.
- A `Function` node named `"template"` was emitted spuriously by the Function extractor when complex templates parse as haskell `function` nodes. Slice 10a's bugfix added a keyword filter in `pass_c_definitions` to skip Daml-keyword function names. The same filter likely needs extension for `signatory`/`controller`/`observer`/`implements`/`requires` if probe shows they similarly leak through.

---

## File structure produced by this slice

```
codegraph-rs/
  src/
    extraction/
      result.rs                                # MODIFY: EdgeKind variants (5), UnresolvedKind variants (5), scope_qid field
    db/
      write.rs                                 # MODIFY: write_unresolved_refs adds scope_qid column
    extraction/
      languages/
        daml.rs                                # MODIFY: pass_e_party_edges + pass_e_interface_edges + keyword filter
        daml.scm                               # MODIFY: signatory/controller/observer/implements/requires queries
    resolution/
      mod.rs                                   # MODIFY: orchestrate Phase A.5 + stats fields
      phase_a5_scoped_match.rs                 # NEW: scoped party + implements/requires resolution
  tests/
    extraction_unit.rs                         # MODIFY: ~12 new unit tests
    integration_daml_index.rs                  # MODIFY: Cypher assertions for new edge kinds
  README.md                                    # MODIFY: bump Status to slice 10b
```

---

### Task 1: New `EdgeKind` and `UnresolvedKind` variants

**Files:**
- Modify: `src/extraction/result.rs`
- Modify: `tests/extraction_unit.rs`

- [ ] **Step 1: Append failing tests to `tests/extraction_unit.rs`**

```rust
#[test]
fn edgekind_signatory_relationship_type() {
    assert_eq!(EdgeKind::Signatory.as_relationship_type(), "SIGNATORY");
}

#[test]
fn edgekind_controller_relationship_type() {
    assert_eq!(EdgeKind::Controller.as_relationship_type(), "CONTROLLER");
}

#[test]
fn edgekind_observer_relationship_type() {
    assert_eq!(EdgeKind::Observer.as_relationship_type(), "OBSERVER");
}

#[test]
fn edgekind_implements_relationship_type() {
    assert_eq!(EdgeKind::Implements.as_relationship_type(), "IMPLEMENTS");
}

#[test]
fn edgekind_requires_relationship_type() {
    assert_eq!(EdgeKind::Requires.as_relationship_type(), "REQUIRES");
}

#[test]
fn unresolvedkind_signatory_str() {
    assert_eq!(UnresolvedKind::Signatory.as_str(), "Signatory");
}

#[test]
fn unresolvedkind_controller_str() {
    assert_eq!(UnresolvedKind::Controller.as_str(), "Controller");
}

#[test]
fn unresolvedkind_observer_str() {
    assert_eq!(UnresolvedKind::Observer.as_str(), "Observer");
}

#[test]
fn unresolvedkind_implements_str() {
    assert_eq!(UnresolvedKind::Implements.as_str(), "Implements");
}

#[test]
fn unresolvedkind_requires_str() {
    assert_eq!(UnresolvedKind::Requires.as_str(), "Requires");
}
```

- [ ] **Step 2: Run failing tests**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo test --test extraction_unit 2>&1 | tail -10
```

Expected: compile errors — new variants not defined.

- [ ] **Step 3: Add `EdgeKind` variants in `src/extraction/result.rs`**

Read the existing `EdgeKind` enum. Add the five new variants after the existing ones (alphabetical or grouped — match the existing convention; slice 9's enum has `Contains, HasRef, Imports, Calls, References, TypeOf` so probably grouped by domain).

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum EdgeKind {
    Contains,
    HasRef,
    Imports,
    Calls,
    References,
    TypeOf,
    /// `signatory <party-expr>` on a Template. Source: Template, Target: Field or Symbol.
    Signatory,
    /// `controller <party-expr>` on a Choice. Source: Choice, Target: Field or Symbol.
    Controller,
    /// `observer <party-expr>` on a Template. Source: Template, Target: Field or Symbol.
    Observer,
    /// `interface instance I for T where ...` — Template implements Interface.
    Implements,
    /// `interface I requires J where ...` — Interface requires Interface.
    Requires,
}
```

Extend `as_relationship_type`:

```rust
impl EdgeKind {
    pub fn as_relationship_type(self) -> &'static str {
        match self {
            EdgeKind::Contains => "CONTAINS",
            EdgeKind::HasRef => "HAS_REF",
            EdgeKind::Imports => "IMPORTS",
            EdgeKind::Calls => "CALLS",
            EdgeKind::References => "REFERENCES",
            EdgeKind::TypeOf => "TYPE_OF",
            EdgeKind::Signatory => "SIGNATORY",
            EdgeKind::Controller => "CONTROLLER",
            EdgeKind::Observer => "OBSERVER",
            EdgeKind::Implements => "IMPLEMENTS",
            EdgeKind::Requires => "REQUIRES",
        }
    }
}
```

- [ ] **Step 4: Add `UnresolvedKind` variants in the same file**

Find the `UnresolvedKind` enum. Add five new variants after the existing `Call, TypeRef, Import, Reference` (or whatever exists):

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum UnresolvedKind {
    Call,
    TypeRef,
    Import,
    Reference,
    Signatory,
    Controller,
    Observer,
    Implements,
    Requires,
}
```

Extend `as_str`:

```rust
impl UnresolvedKind {
    pub fn as_str(self) -> &'static str {
        match self {
            UnresolvedKind::Call => "Call",
            UnresolvedKind::TypeRef => "TypeRef",
            UnresolvedKind::Import => "Import",
            UnresolvedKind::Reference => "Reference",
            UnresolvedKind::Signatory => "Signatory",
            UnresolvedKind::Controller => "Controller",
            UnresolvedKind::Observer => "Observer",
            UnresolvedKind::Implements => "Implements",
            UnresolvedKind::Requires => "Requires",
        }
    }
}
```

- [ ] **Step 5: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 128 tests pass (118 baseline + 10 new label tests).

- [ ] **Step 6: Commit**

```bash
git add src/extraction/result.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add Signatory/Controller/Observer/Implements/Requires Edge+Unresolved kinds"
```

---

### Task 2: `UnresolvedRef.scope_qid: Option<String>` field + DB write path

**Files:**
- Modify: `src/extraction/result.rs`
- Modify: `src/db/write.rs`
- Modify: `tests/extraction_unit.rs`

The `UnresolvedRef` struct gains a new field `scope_qid: Option<String>`. The Neo4j write path (`write_unresolved_refs`) writes this as a new property. Reading is automatic (`row.get::<Option<String>>("scope_qid")`).

- [ ] **Step 1: Append failing test**

```rust
#[test]
fn unresolved_ref_scope_qid_default_is_none() {
    let r = UnresolvedRef {
        qid: "x".to_string(),
        source_qid: "src".to_string(),
        unresolved_name: "n".to_string(),
        kind: UnresolvedKind::Call,
        line: 1,
        col: 0,
        scope_qid: None,
    };
    assert_eq!(r.scope_qid, None);
}

#[test]
fn unresolved_ref_scope_qid_can_be_set() {
    let r = UnresolvedRef {
        qid: "x".to_string(),
        source_qid: "src".to_string(),
        unresolved_name: "owner".to_string(),
        kind: UnresolvedKind::Signatory,
        line: 1,
        col: 0,
        scope_qid: Some("u:Foo.daml:Foo.Counter".to_string()),
    };
    assert_eq!(r.scope_qid.as_deref(), Some("u:Foo.daml:Foo.Counter"));
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test extraction_unit unresolved_ref_scope 2>&1 | tail -10
```

Expected: compile error — `scope_qid` field doesn't exist.

- [ ] **Step 3: Add `scope_qid` to `UnresolvedRef`**

In `src/extraction/result.rs`, find the `UnresolvedRef` struct:

```rust
#[derive(Debug, Clone)]
pub struct UnresolvedRef {
    pub qid: String,
    pub source_qid: String,
    pub unresolved_name: String,
    pub kind: UnresolvedKind,
    pub line: u32,
    pub col: u32,
    /// For party-edge refs (Signatory/Controller/Observer), this is the qid of
    /// the enclosing Template. Resolution Phase A.5 uses it to scope name
    /// lookup: `<scope_qid_qname>.<unresolved_name>` resolves first against
    /// the Template's Field children. For other ref kinds (Call/TypeRef/Import/
    /// Reference/Implements/Requires), typically `None`.
    pub scope_qid: Option<String>,
}
```

- [ ] **Step 4: Find all `UnresolvedRef { ... }` constructions in the codebase**

Search:

```bash
grep -rn "UnresolvedRef {" src/ tests/
```

Each construction will fail to compile with the new field. Add `scope_qid: None,` as the last field of every existing struct literal. Known sites:
- `src/extraction/languages/daml.rs` (slice 9's pass_b_imports, pass_c_call_sites, pass_c_exercise_choices)
- `src/extraction/languages/rust.rs` (slice 2's call-site extraction)

**The build is the safety net.** If `grep` misses any construction site (e.g., one inside a macro or written with `..Default::default()`), the next `cargo build` in Step 6 will surface a compile error pointing at the missed site. Add `scope_qid: None,` to whatever the compiler flags.

- [ ] **Step 5: Update `write_unresolved_refs` in `src/db/write.rs`**

The Cypher batch currently looks like:

```rust
let cypher_text = "\
    UNWIND $batch AS row \
    MERGE (n:UnresolvedRef {qid: row.qid}) \
    SET n.source_qid = row.source_qid, \
        n.unresolved_name = row.unresolved_name, \
        n.kind = row.kind, \
        n.line = row.line, \
        n.col = row.col";
```

Extend to include `scope_qid`:

```rust
let cypher_text = "\
    UNWIND $batch AS row \
    MERGE (n:UnresolvedRef {qid: row.qid}) \
    SET n.source_qid = row.source_qid, \
        n.unresolved_name = row.unresolved_name, \
        n.kind = row.kind, \
        n.line = row.line, \
        n.col = row.col, \
        n.scope_qid = row.scope_qid";
```

And update `ref_to_json`:

```rust
fn ref_to_json(r: &UnresolvedRef) -> Value {
    let mut m = Map::new();
    m.insert("qid".into(), r.qid.clone().into());
    m.insert("source_qid".into(), r.source_qid.clone().into());
    m.insert("unresolved_name".into(), r.unresolved_name.clone().into());
    m.insert("kind".into(), r.kind.as_str().into());
    m.insert("line".into(), r.line.into());
    m.insert("col".into(), r.col.into());
    m.insert(
        "scope_qid".into(),
        r.scope_qid.clone().map(Value::String).unwrap_or(Value::Null),
    );
    Value::Object(m)
}
```

> **On Cypher `SET n.x = NULL`:** Neo4j removes the property when set to null. So v1-era reads of `n.scope_qid` on records written without it return `null`; Rust deserialization gives `Option::None`. Backwards-compatible.

- [ ] **Step 6: Build + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: 130 tests pass (128 + 2 new). All existing UnresolvedRef constructions updated.

- [ ] **Step 7: Commit**

```bash
git add src/extraction/result.rs src/db/write.rs src/extraction/languages/daml.rs src/extraction/languages/rust.rs tests/extraction_unit.rs
git commit -m "feat(extraction): add UnresolvedRef.scope_qid for party-scoped name resolution"
```

---

### Task 3: `signatory`/`controller`/`observer` extraction with multi-party decomposition (TDD)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

The keyword pattern is `signatory <expr>`, `controller <expr>`, `observer <expr>`. Each emits one or more `UnresolvedRef`s by walking the argument subtree for identifier leaves. The same pass handles all three keywords (only the `UnresolvedKind` differs).

The multi-party decomposition rule (spec §3.3):
- Collect every `(variable)` and `(constructor)` leaf inside the argument subtree.
- Filter out known wrapper identifiers: `Set`, `fromList`, `fromOptional`, `fromSome`, `concat`, `concatMap`, the `$` operator (it's parsed as `operator` not variable, so won't appear in our leaf walk).
- For each remaining identifier, emit one `UnresolvedRef`.

Examples:

| Daml expression | Refs emitted |
|---|---|
| `signatory owner` | 1: `owner` |
| `signatory [owner, delegate]` | 2: `owner`, `delegate` |
| `signatory $ Set.fromList [a, b]` | 2: `a`, `b` (`Set`/`fromList` filtered) |
| `signatory $ fromOptional [] mObservers` | 1: `mObservers` (`fromOptional` filtered) |

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_extracts_signatory_single_identifier() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n  where\n    signatory owner\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let sigs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Signatory).collect();
    assert_eq!(sigs.len(), 1, "expected 1 Signatory ref; got {:?}", sigs.iter().map(|r| &r.unresolved_name).collect::<Vec<_>>());
    assert_eq!(sigs[0].unresolved_name, "owner");
    let template_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Template).unwrap().qid;
    assert_eq!(sigs[0].scope_qid.as_deref(), Some(template_qid.as_str()),
        "scope_qid should be the enclosing Template qid");
}

#[test]
fn daml_extracts_controller_inside_choice() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n  where\n    signatory owner\n    choice Inc : ()\n      controller owner\n      do return ()\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let ctrls: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Controller).collect();
    assert_eq!(ctrls.len(), 1);
    assert_eq!(ctrls[0].unresolved_name, "owner");
    // Controller scope_qid is the enclosing Choice (more specific). Phase A.5
    // falls back to the parent Template if `<choice_qname>.<name>` misses.
    let choice_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Choice).unwrap().qid;
    assert_eq!(ctrls[0].scope_qid.as_deref(), Some(choice_qid.as_str()),
        "controller's scope_qid is the enclosing Choice; Phase A.5 falls back to Template parent");
}

#[test]
fn daml_extracts_observer() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n    obs : Party\n  where\n    signatory owner\n    observer obs\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let obs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Observer).collect();
    assert_eq!(obs.len(), 1);
    assert_eq!(obs[0].unresolved_name, "obs");
}

#[test]
fn daml_multi_party_list_decomposed_into_two_refs() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n    delegate : Party\n  where\n    signatory [owner, delegate]\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let sigs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Signatory).collect();
    assert_eq!(sigs.len(), 2, "list signatory emits 2 refs");
    let names: Vec<&str> = sigs.iter().map(|r| r.unresolved_name.as_str()).collect();
    assert!(names.contains(&"owner"));
    assert!(names.contains(&"delegate"));
}

#[test]
fn daml_multi_party_set_fromlist_filters_wrappers() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ntemplate Counter\n  with\n    a : Party\n    b : Party\n  where\n    signatory Set.fromList [a, b]\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let sigs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Signatory).collect();
    let names: Vec<&str> = sigs.iter().map(|r| r.unresolved_name.as_str()).collect();
    assert!(names.contains(&"a"), "a should be extracted; got {names:?}");
    assert!(names.contains(&"b"));
    assert!(!names.contains(&"Set"), "Set should be filtered as wrapper");
    assert!(!names.contains(&"fromList"), "fromList should be filtered as wrapper");
}

#[test]
fn daml_signatory_emits_has_ref_edge() {
    use codegraph_rs::extraction::result::{EdgeKind, UnresolvedKind};
    let source = "module Foo where\n\ntemplate Counter\n  with\n    owner : Party\n  where\n    signatory owner\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let sig = result.unresolved_refs.iter()
        .find(|r| r.kind == UnresolvedKind::Signatory && r.unresolved_name == "owner")
        .expect("signatory ref must exist");
    // The HAS_REF edge goes from the source_qid (enclosing Template) to the
    // UnresolvedRef qid, same as Call refs in slice 9.
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::HasRef && e.from_qid == sig.source_qid && e.to_qid == sig.qid));
}
```

- [ ] **Step 2: Run failing tests**

Expected: FAIL — no party extraction yet.

- [ ] **Step 3: Probe `signatory owner` parse shape**

Slice 10a found Daml keywords parse in multiple ERROR-wrapped shapes. Probe `signatory`:

```rust
#[test]
#[ignore]
fn probe_signatory_parse_shape() {
    let source = "module Foo where\n\ntemplate Counter\n  with p : Party\n  where\n    signatory owner\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}

#[test]
#[ignore]
fn probe_signatory_list_parse_shape() {
    let source = "module Foo where\n\ntemplate Counter\n  with p : Party\n  where\n    signatory [a, b]\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}
```

(`print_tree` is in `tests/extraction_unit.rs` from slice 10a tasks 2-6.)

Run: `cargo test --test extraction_unit probe_signatory_parse_shape -- --ignored --nocapture 2>&1 | head -100`

Expected: `signatory owner` likely appears as an apply (similar to slice 10a's choice probe), where `signatory` is a variable and `owner` is the argument. The list form likely produces a list literal as the argument.

**REMOVE these probe tests before commit.**

- [ ] **Step 4: Add the party queries to `daml.scm`**

Based on probe shapes:

```scheme
; Signatory anchor: `signatory <expr>`. The argument subtree is walked at
; extraction time for identifier leaves (multi-party decomposition).
((apply
  function: (variable) @signatory.kw
  argument: (_) @signatory.target) @signatory.def
  (#eq? @signatory.kw "signatory"))

; Controller anchor (inside a choice body): `controller <expr>`.
((apply
  function: (variable) @controller.kw
  argument: (_) @controller.target) @controller.def
  (#eq? @controller.kw "controller"))

; Observer anchor: `observer <expr>`.
((apply
  function: (variable) @observer.kw
  argument: (_) @observer.target) @observer.def
  (#eq? @observer.kw "observer"))
```

Adapt to the actual parse shape if probe shows different (e.g., ERROR-direct shapes like Task 4 for Interface).

- [ ] **Step 5: Add `pass_e_party_edges` to `src/extraction/languages/daml.rs`**

Place after `pass_e_fields` (slice 10a's last extraction pass):

```rust
/// Wrapper identifiers that should be filtered out of party-expression walks.
/// These are Daml stdlib functions/types that appear in patterns like
/// `signatory $ Set.fromList [a, b]` but are NOT party identifiers themselves.
const PARTY_WRAPPER_FILTERS: &[&str] = &[
    "Set", "fromList", "fromOptional", "fromSome",
    "concat", "concatMap", "map", "filter",
    // Note: `$` is parsed as `operator`, not `variable`, so it won't appear in
    // identifier-leaf walks. No need to filter it explicitly.
];

/// Extract all `(variable)` and `(constructor)` identifier leaves from a
/// subtree, filtering out known wrapper names. Used for multi-party
/// expression decomposition.
fn party_identifiers_in_subtree(
    node: tree_sitter::Node,
    source: &[u8],
    out: &mut Vec<(String, tree_sitter::Point)>,
) {
    if node.kind() == "variable" || node.kind() == "constructor" {
        if let Ok(text) = node.utf8_text(source) {
            let name = text.trim().to_string();
            // Filter known stdlib wrappers.
            if !PARTY_WRAPPER_FILTERS.contains(&name.as_str()) {
                // Also filter qualified-name segments — when parser sees
                // `Set.fromList`, it may emit either a single dotted-name
                // identifier or two separate constructors/variables. Split on
                // '.' and emit each segment that isn't a known wrapper.
                for segment in name.split('.') {
                    let seg = segment.trim();
                    if !seg.is_empty() && !PARTY_WRAPPER_FILTERS.contains(&seg) {
                        out.push((seg.to_string(), node.start_position()));
                    }
                }
            }
        }
        return;
    }
    let mut cursor = node.walk();
    for child in node.children(&mut cursor) {
        party_identifiers_in_subtree(child, source, out);
    }
}

/// Tree-sitter-based extraction of signatory/controller/observer party
/// expressions. Each keyword anchor emits one or more UnresolvedRef per
/// identifier leaf in the argument subtree.
///
/// scope_qid is the enclosing Template qid (NOT the Choice for controllers
/// — controllers' party names typically refer to Template fields, not
/// Choice fields).
fn pass_e_party_edges(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    ctx: &PassCtx<'_>,
    result: &mut ExtractionResult,
) {
    // Process each keyword (signatory / controller / observer) with the same
    // shape-walking logic. The only differences are the capture names and
    // the emitted UnresolvedKind.
    let keywords: &[(&str, &str, UnresolvedKind)] = &[
        ("signatory.def", "signatory.target", UnresolvedKind::Signatory),
        ("controller.def", "controller.target", UnresolvedKind::Controller),
        ("observer.def", "observer.target", UnresolvedKind::Observer),
    ];

    for (def_name, target_name, kind) in keywords {
        let Some(def_idx) = query.capture_index_for_name(def_name) else { continue };
        let Some(target_idx) = query.capture_index_for_name(target_name) else { continue };

        let mut cursor = QueryCursor::new();
        let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

        let mut pending: Vec<(tree_sitter::Node, tree_sitter::Node)> = Vec::new();
        while let Some(m) = matches.next() {
            let mut def_node = None;
            let mut target_node = None;
            for cap in m.captures {
                if cap.index == def_idx { def_node = Some(cap.node); }
                else if cap.index == target_idx { target_node = Some(cap.node); }
            }
            if let (Some(def), Some(target)) = (def_node, target_node) {
                pending.push((def, target));
            }
        }

        for (def, target) in pending {
            // The enclosing scope is the Template (for signatory/observer)
            // or the enclosing Template-of-the-Choice (for controller).
            let pos = def.start_position();
            let line = (pos.row as u32) + 1;
            let col = pos.column as u32;
            let template_qid = enclosing_template_qid(&result.nodes, line, col);
            let Some(template_qid) = template_qid else { continue };
            let template_qid = template_qid.to_string();

            // source_qid AND scope_qid depend on kind:
            //   - signatory/observer: source = Template, scope = Template
            //   - controller: source = Choice (the containing choice), scope = Choice
            //     (Phase A.5 will fall back to the enclosing Template when
            //     `<choice_qname>.<name>` misses — choices with `with` blocks
            //     have Choice fields that controllers can reference.)
            let (source_qid, scope_qid) = if *kind == UnresolvedKind::Controller {
                match enclosing_choice_qid(&result.nodes, line, col).map(str::to_string) {
                    Some(c) => (c.clone(), c),
                    None => (template_qid.clone(), template_qid.clone()),
                }
            } else {
                (template_qid.clone(), template_qid.clone())
            };

            // Walk the target subtree for identifier leaves.
            let mut idents: Vec<(String, tree_sitter::Point)> = Vec::new();
            party_identifiers_in_subtree(target, source.as_bytes(), &mut idents);

            for (i, (name, pt)) in idents.iter().enumerate() {
                let ref_line = (pt.row as u32) + 1;
                let ref_col = pt.column as u32;
                let qid = format!("{}:ref:{}:{}:{}:{}",
                    source_qid, ref_line, ref_col, kind.as_str(), i);

                result.unresolved_refs.push(UnresolvedRef {
                    qid: qid.clone(),
                    source_qid: source_qid.clone(),
                    unresolved_name: name.clone(),
                    kind: *kind,
                    line: ref_line,
                    col: ref_col,
                    scope_qid: Some(scope_qid.clone()),
                });
                result.edges.push(RawEdge {
                    from_qid: source_qid.clone(),
                    to_qid: qid,
                    kind: EdgeKind::HasRef,
                });
            }
        }
    }
}

/// Find the smallest Choice node in `nodes` whose span contains (line, col).
fn enclosing_choice_qid(nodes: &[RawNode], line: u32, col: u32) -> Option<&str> {
    let mut best: Option<&RawNode> = None;
    for n in nodes {
        if n.kind != NodeKind::Choice { continue; }
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

> **Adapt to `PassCtx<'_>`** — slice 10a tasks introduced this struct. Check the actual signature.

> **Note on parse-shape variability**: if the probe shows `signatory owner` doesn't match the assumed `(apply function: (variable) argument: (variable))` shape, the implementer may need to add additional queries (Shape A/B/C style, like slice 10a tasks 2/4 did for templates/interfaces). The wrapper-filter and decomposition logic stays the same regardless.

- [ ] **Step 6: Wire `pass_e_party_edges` in `extract_daml`**

Add the call AFTER `pass_e_fields` (which is currently last):

```rust
pass_e_party_edges(&query, &tree, source, &ctx, &mut result);
```

(Adapt args to match the PassCtx pattern.)

- [ ] **Step 7: Run tests**

```bash
cargo test --test extraction_unit 2>&1 | tail -15
```

Expected: 6 new tests pass.

- [ ] **Step 8: Build + clippy + fmt + regression check**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
cargo test --test integration_daml_index 2>&1 | tail -3
```

- [ ] **Step 9: REMOVE probe tests, then commit**

```bash
grep -n "probe_signatory_parse_shape\|probe_signatory_list_parse_shape" tests/extraction_unit.rs
# Expected: no output

git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml signatory/controller/observer party-expression extraction"
```

---

### Task 4: `implements`/`requires` extraction (TDD)

**Files:**
- Modify: `src/extraction/languages/daml.scm`
- Modify: `src/extraction/languages/daml.rs`
- Modify: `tests/extraction_unit.rs`

`implements` appears inside an `interface instance I for T where ...` clause. The keyword sequence anchors on `implements` (or alternatively, `instance` followed by a constructor) — probe required.

`requires` appears in `interface I requires J where ...`. Single-constructor argument.

Both emit `UnresolvedRef` with a single target constructor name.

- [ ] **Step 1: Append failing tests**

```rust
#[test]
fn daml_extracts_implements_inside_interface_instance() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    // The `interface instance Bar for T where ...` syntax (Daml 2.7+).
    let source = "module Foo where\n\ninterface Bar where\n  getOwner : Party\n\ntemplate Counter\n  with owner : Party\n  where\n    signatory owner\n    interface instance Bar for Counter where\n      getOwner = owner\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let imps: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Implements).collect();
    assert_eq!(imps.len(), 1, "expected 1 Implements ref; got {:?}", imps.iter().map(|r| &r.unresolved_name).collect::<Vec<_>>());
    assert_eq!(imps[0].unresolved_name, "Bar");
    // source_qid is the enclosing Template (not Interface — the IMPLEMENTS edge
    // comes FROM the Template).
    let template_qid = &result.nodes.iter().find(|n| n.kind == NodeKind::Template).unwrap().qid;
    assert_eq!(imps[0].source_qid, *template_qid);
}

#[test]
fn daml_extracts_requires_between_interfaces() {
    use codegraph_rs::extraction::result::UnresolvedKind;
    let source = "module Foo where\n\ninterface Base where\n  getId : Int\n\ninterface Sub requires Base where\n  getName : Text\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let reqs: Vec<_> = result.unresolved_refs.iter()
        .filter(|r| r.kind == UnresolvedKind::Requires).collect();
    assert_eq!(reqs.len(), 1);
    assert_eq!(reqs[0].unresolved_name, "Base");
    // source_qid is the requiring Interface (Sub).
    let sub_qid = &result.nodes.iter()
        .find(|n| n.kind == NodeKind::Interface && n.name == "Sub")
        .unwrap().qid;
    assert_eq!(reqs[0].source_qid, *sub_qid);
}

#[test]
fn daml_implements_emits_has_ref_edge() {
    use codegraph_rs::extraction::result::{EdgeKind, UnresolvedKind};
    let source = "module Foo where\n\ninterface Bar where\n  x : Int\n\ntemplate T\n  with owner : Party\n  where\n    signatory owner\n    interface instance Bar for T where\n      x = 1\n";
    let result = extract_daml(source, "u", "Foo.daml").expect("ok");
    let imp = result.unresolved_refs.iter()
        .find(|r| r.kind == UnresolvedKind::Implements).expect("implements ref");
    assert!(result.edges.iter().any(|e|
        e.kind == EdgeKind::HasRef && e.from_qid == imp.source_qid && e.to_qid == imp.qid));
}
```

- [ ] **Step 2: Probe `implements` and `requires` shapes**

```rust
#[test]
#[ignore]
fn probe_implements_parse_shape() {
    let source = "module Foo where\n\ninterface Bar where\n  getOwner : Party\n\ntemplate Counter\n  with owner : Party\n  where\n    signatory owner\n    interface instance Bar for Counter where\n      getOwner = owner\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}

#[test]
#[ignore]
fn probe_requires_parse_shape() {
    let source = "module Foo where\n\ninterface Base where\n  x : Int\n\ninterface Sub requires Base where\n  y : Int\n";
    let tree = codegraph_rs::extraction::parser::parse_daml(source).unwrap();
    print_tree(tree.root_node(), 0);
}
```

Run and document the shapes. `implements` may show up as `variable "interface"` followed by `variable "instance"` followed by `constructor "Bar"` inside the same ERROR — the implementer must work out the right query.

**REMOVE probe tests before commit.**

- [ ] **Step 3: Add `implements` and `requires` queries to `daml.scm`**

**Write the queries to match the shapes Step 2 actually observed.** The examples below are placeholders for the most common shape; don't paste them blindly.

Likely shapes:

```scheme
; Implements: `interface instance Bar for T where ...` (Daml 2.7+).
; The `interface instance` keyword pair may parse as:
;   - Two siblings (variable "interface", variable "instance") in an ERROR — Shape A
;   - A nested apply chain (apply (variable "interface") (apply (variable "instance") (constructor)))
;     in an ERROR — Shape B (likely, given slice 10a templates exhibited similar curried shapes)
; Pick the query form matching Step 2's probe.

; Shape A (flat siblings):
((apply
  function: (variable) @implements.kw
  argument: (constructor) @implements.interface) @implements.def
  (#eq? @implements.kw "instance"))

; Shape B (nested):
((apply
  function: (apply argument: (variable) @implements.kw)
  argument: (constructor) @implements.interface) @implements.def
  (#eq? @implements.kw "instance"))

; Requires: `interface Sub requires Base where ...`.
((apply
  function: (variable) @requires.kw
  argument: (constructor) @requires.interface) @requires.def
  (#eq? @requires.kw "requires"))
```

If Step 2's probe shows neither Shape A nor Shape B matches, write a custom query OR fall back to a pure Rust tree walk (the pattern slice 10a Task 6 used for Fields when the parse shapes were too irregular for SCM).

- [ ] **Step 4: Add `pass_e_interface_edges` to `daml.rs`**

```rust
/// Tree-sitter-based extraction of `interface instance ... for ... where ...`
/// (implements) and `interface ... requires ... where ...` (requires) clauses.
///
/// implements: source_qid = enclosing Template, target = constructor name (Interface).
/// requires:   source_qid = enclosing Interface, target = constructor name (Interface).
fn pass_e_interface_edges(
    query: &Query,
    tree: &tree_sitter::Tree,
    source: &str,
    ctx: &PassCtx<'_>,
    result: &mut ExtractionResult,
) {
    let keywords: &[(&str, &str, UnresolvedKind, bool)] = &[
        // (def_capture, target_capture, kind, source_is_template)
        ("implements.def", "implements.interface", UnresolvedKind::Implements, true),
        ("requires.def", "requires.interface", UnresolvedKind::Requires, false),
    ];

    for (def_name, target_name, kind, source_is_template) in keywords {
        let Some(def_idx) = query.capture_index_for_name(def_name) else { continue };
        let Some(target_idx) = query.capture_index_for_name(target_name) else { continue };

        let mut cursor = QueryCursor::new();
        let mut matches = cursor.matches(query, tree.root_node(), source.as_bytes());

        let mut pending: Vec<(tree_sitter::Node, String, tree_sitter::Point)> = Vec::new();
        while let Some(m) = matches.next() {
            let mut def_node = None;
            let mut target_node = None;
            for cap in m.captures {
                if cap.index == def_idx { def_node = Some(cap.node); }
                else if cap.index == target_idx { target_node = Some(cap.node); }
            }
            if let (Some(def), Some(target)) = (def_node, target_node) {
                if let Ok(name) = target.utf8_text(source.as_bytes()) {
                    let target_name = name.trim().to_string();
                    if !target_name.is_empty() {
                        pending.push((def, target_name, target.start_position()));
                    }
                }
            }
        }

        for (def, target_name, pt) in pending {
            let pos = def.start_position();
            let line = (pos.row as u32) + 1;
            let col = pos.column as u32;

            // source_qid lookup depends on the edge kind.
            let source_qid_opt = if *source_is_template {
                enclosing_template_qid(&result.nodes, line, col).map(str::to_string)
            } else {
                enclosing_interface_qid(&result.nodes, line, col).map(str::to_string)
            };
            let Some(source_qid) = source_qid_opt else { continue };

            let ref_line = (pt.row as u32) + 1;
            let ref_col = pt.column as u32;
            let qid = format!("{}:ref:{}:{}:{}:0",
                source_qid, ref_line, ref_col, kind.as_str());

            result.unresolved_refs.push(UnresolvedRef {
                qid: qid.clone(),
                source_qid: source_qid.clone(),
                unresolved_name: target_name,
                kind: *kind,
                line: ref_line,
                col: ref_col,
                scope_qid: None, // these resolve workspace-wide, not scoped
            });
            result.edges.push(RawEdge {
                from_qid: source_qid,
                to_qid: qid,
                kind: EdgeKind::HasRef,
            });
        }
    }
}
```

- [ ] **Step 5: Wire `pass_e_interface_edges` in `extract_daml`**

Add the call AFTER `pass_e_party_edges`:

```rust
pass_e_interface_edges(&query, &tree, source, &ctx, &mut result);
```

- [ ] **Step 6: Run tests, build, clippy, fmt, regression check**

```bash
cargo test --test extraction_unit 2>&1 | tail -10
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: 139 tests pass (136 + 3 new).

- [ ] **Step 7: REMOVE probe tests, then commit**

```bash
grep -n "probe_implements_parse_shape\|probe_requires_parse_shape" tests/extraction_unit.rs
# Expected: no output

git add src/extraction/languages/daml.rs src/extraction/languages/daml.scm tests/extraction_unit.rs
git commit -m "feat(extraction): Daml implements/requires interface-relationship extraction"
```

---

### Task 5: Resolution Phase A.5 — scoped name match (TDD)

**Files:**
- Create: `src/resolution/phase_a5_scoped_match.rs`
- Modify: `src/resolution/mod.rs` (just `pub mod phase_a5_scoped_match;` for now; wiring is Task 6)
- Modify: `tests/resolution_unit.rs` (or wherever resolution unit tests live)

Phase A.5 resolves `UnresolvedRef` records with `scope_qid: Some(s)` by looking up `<scope_qname>.<unresolved_name>` in a pre-built `HashMap<String, String>` (qualified_name → qid). On hit, it emits the appropriate edge (SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS/REQUIRES) via Cypher.

Note: Implements and Requires resolution doesn't use `scope_qid` — they look up the target qualified_name globally. But this slice's design keeps them in the same phase for symmetry (the spec frames "scoped name match" as the path; for implements/requires, "scoped" means "module-prefixed", which is equivalent to global since interfaces are top-level decls).

For simplicity, Phase A.5 handles ALL five new edge kinds, with the lookup key varying:
- Party kinds (Signatory/Controller/Observer): key = `<scope_qname>.<unresolved_name>`
- Implements: key = `<module-of-scope-or-source-qid>.<unresolved_name>` first, then fall back to `<unresolved_name>` global
- Requires: same as Implements

Actually — keep it simpler. Two resolution strategies:

1. **Scoped (party kinds)**: try `<scope_qname>.<unresolved_name>`. If miss, skip (Phase C handles fallback).
2. **Global by name (Implements/Requires)**: look up any `Interface` symbol whose `qualified_name` ends with `.<unresolved_name>` or equals `<unresolved_name>`.

- [ ] **Step 1: Create `src/resolution/phase_a5_scoped_match.rs` with failing test inline**

Read `src/resolution/phase_b_import_driven.rs` and `src/resolution/phase_c_name_match.rs` first to understand the resolution-phase pattern (function signature, ResolutionStats, Cypher patterns, return type).

Then create the new file:

```rust
//! Phase A.5 — Scoped name match for party-edge UnresolvedRef records.
//!
//! Runs AFTER Phase A (imports) and BEFORE Phase B (import-driven calls).
//!
//! For each UnresolvedRef with `kind` in {Signatory, Controller, Observer}:
//!   - Look up `<scope_qid_qname>.<unresolved_name>` in the Symbol-by-qname map.
//!   - For Controllers, falls back to parent-Template scope on miss.
//!   - On hit: MERGE the corresponding edge (SIGNATORY/CONTROLLER/OBSERVER).
//!
//! For Implements/Requires: target is a workspace-wide Interface by qname.
//!   - Look up any Interface whose qname equals or ends with `.<unresolved_name>`.
//!   - On hit: MERGE the IMPLEMENTS/REQUIRES edge.
//!
//! **Idempotency caveat.** MERGE is idempotent for the same `(source, target)`
//! pair, but if a Template's signatory expression changes from `owner` to
//! `creator` and sync re-runs resolution, the OLD edge to the `owner` Field
//! is not deleted — only the new edge to `creator` is added. The Template
//! ends up with two SIGNATORY edges. This is a known limitation of the v1
//! resolution model (spec §7 says "Resolution as idempotent rebuild" but
//! Phase B/C have the same staleness issue with CALLS/REFERENCES edges).
//! A v2.1 follow-up could prepend a DETACH-DELETE pass that clears all
//! resolution-derived edges before rebuilding.

use anyhow::{Context, Result};
use neo4rs::query;
use std::collections::HashMap;

use crate::db::Connection;

/// Stats returned by Phase A.5.
#[derive(Debug, Clone, Default)]
pub struct PhaseA5Stats {
    pub signatory_resolved: usize,
    pub controller_resolved: usize,
    pub observer_resolved: usize,
    pub implements_resolved: usize,
    pub requires_resolved: usize,
    pub unresolved: usize,
}

/// Run Phase A.5. Returns counts of edges emitted by kind.
pub async fn run(conn: &Connection) -> Result<PhaseA5Stats> {
    // Build qualified_name → qid map (one round-trip).
    let qname_map = build_qname_map(conn).await?;

    // Pull all UnresolvedRef records whose kind is one of the 5 new kinds.
    let refs = fetch_party_and_interface_refs(conn).await?;

    let mut stats = PhaseA5Stats::default();

    for r in refs {
        let (cypher_text, edge_resolved) = match r.kind.as_str() {
            "Signatory" | "Controller" | "Observer" => {
                // Scoped lookup: <scope_qname>.<name>.
                //
                // For Controllers, scope_qid is the enclosing Choice; if the
                // scoped lookup misses, fall back one level to the parent
                // Template's qname. The parent qname is the scope_qname with
                // its last dotted segment stripped (`Foo.T.Bump` → `Foo.T`).
                // Signatory/Observer always use Template scope, so no fallback
                // hop is needed.
                let Some(scope_qid) = &r.scope_qid else {
                    stats.unresolved += 1;
                    continue;
                };
                let scope_qname = scope_qid.split(':').skip(2).collect::<Vec<_>>().join(":");
                let primary_key = format!("{}.{}", scope_qname, r.unresolved_name);
                let target_qid = qname_map.get(&primary_key).cloned().or_else(|| {
                    // Fallback for Controllers: try the parent Template's scope.
                    if r.kind == "Controller" {
                        if let Some(idx) = scope_qname.rfind('.') {
                            let parent_qname = &scope_qname[..idx];
                            let fallback_key = format!("{}.{}", parent_qname, r.unresolved_name);
                            return qname_map.get(&fallback_key).cloned();
                        }
                    }
                    None
                });
                let Some(target_qid) = target_qid else {
                    stats.unresolved += 1;
                    continue;
                };
                let rel = match r.kind.as_str() {
                    "Signatory" => "SIGNATORY",
                    "Controller" => "CONTROLLER",
                    "Observer" => "OBSERVER",
                    _ => unreachable!(),
                };
                let cypher = format!(
                    "MATCH (s:Symbol {{qid: $source_qid}}) \
                     MATCH (t:Symbol {{qid: $target_qid}}) \
                     MERGE (s)-[:{}]->(t)",
                    rel
                );
                (cypher, (r.source_qid.clone(), target_qid.clone(), r.kind.as_str().to_string()))
            }
            "Implements" | "Requires" => {
                // Global lookup: any Interface whose qname ends with the unresolved name
                // (or equals it for unqualified references).
                let target_qid = find_interface_by_name(&qname_map, &r.unresolved_name);
                let Some(target_qid) = target_qid else {
                    stats.unresolved += 1;
                    continue;
                };
                let rel = match r.kind.as_str() {
                    "Implements" => "IMPLEMENTS",
                    "Requires" => "REQUIRES",
                    _ => unreachable!(),
                };
                let cypher = format!(
                    "MATCH (s:Symbol {{qid: $source_qid}}) \
                     MATCH (t:Symbol {{qid: $target_qid}}) \
                     MERGE (s)-[:{}]->(t)",
                    rel
                );
                (cypher, (r.source_qid.clone(), target_qid, r.kind.as_str().to_string()))
            }
            _ => continue,
        };

        // Emit the edge.
        let (source_qid, target_qid, kind_str) = edge_resolved;
        conn.graph
            .execute(
                query(&cypher_text)
                    .param("source_qid", source_qid.clone())
                    .param("target_qid", target_qid.clone()),
            )
            .await
            .with_context(|| format!("MERGE edge {} from {} to {}", kind_str, source_qid, target_qid))?
            .next()
            .await
            .ok();

        match kind_str.as_str() {
            "Signatory" => stats.signatory_resolved += 1,
            "Controller" => stats.controller_resolved += 1,
            "Observer" => stats.observer_resolved += 1,
            "Implements" => stats.implements_resolved += 1,
            "Requires" => stats.requires_resolved += 1,
            _ => {}
        }
    }

    Ok(stats)
}

async fn build_qname_map(conn: &Connection) -> Result<HashMap<String, String>> {
    let mut stream = conn
        .graph
        .execute(query("MATCH (s:Symbol) RETURN s.qualified_name AS qname, s.qid AS qid"))
        .await
        .context("building qualified_name map")?;
    let mut out = HashMap::new();
    while let Some(row) = stream.next().await? {
        let qname: Option<String> = row.get("qname").ok();
        let qid: Option<String> = row.get("qid").ok();
        if let (Some(qname), Some(qid)) = (qname, qid) {
            out.insert(qname, qid);
        }
    }
    Ok(out)
}

#[derive(Debug)]
struct UnresolvedRefRow {
    qid: String,
    source_qid: String,
    unresolved_name: String,
    kind: String,
    scope_qid: Option<String>,
}

async fn fetch_party_and_interface_refs(conn: &Connection) -> Result<Vec<UnresolvedRefRow>> {
    let cypher = "\
        MATCH (n:UnresolvedRef) \
        WHERE n.kind IN ['Signatory','Controller','Observer','Implements','Requires'] \
        RETURN n.qid AS qid, n.source_qid AS source_qid, n.unresolved_name AS name, \
               n.kind AS kind, n.scope_qid AS scope_qid";
    let mut stream = conn.graph.execute(query(cypher)).await.context("fetching party refs")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await? {
        out.push(UnresolvedRefRow {
            qid: row.get("qid")?,
            source_qid: row.get("source_qid")?,
            unresolved_name: row.get("name")?,
            kind: row.get("kind")?,
            scope_qid: row.get::<Option<String>>("scope_qid").ok().flatten(),
        });
    }
    Ok(out)
}

/// Find an Interface qid by its short name (e.g., "Authorizable") OR
/// fully-qualified name (e.g., "Splice.Foo.Authorizable"). If multiple match,
/// returns the first (workspace deduplication is best-effort here).
fn find_interface_by_name(qname_map: &HashMap<String, String>, name: &str) -> Option<String> {
    // Direct full-qname hit?
    if let Some(qid) = qname_map.get(name) {
        return Some(qid.clone());
    }
    // Suffix match: qname ends with `.<name>`.
    let suffix = format!(".{}", name);
    for (qname, qid) in qname_map.iter() {
        if qname.ends_with(&suffix) {
            return Some(qid.clone());
        }
    }
    None
}
```

> **Note on `UnresolvedRefRow.as_str()`:** the helper is just for `match`-on-string convenience; the field is already a String. Adjust if you prefer matching on `r.kind.as_str()` directly.

- [ ] **Step 2: Declare the module**

In `src/resolution/mod.rs`, add `pub mod phase_a5_scoped_match;` near the other phase declarations.

- [ ] **Step 3: Build (no tests yet — this phase is hard to unit-test without a Neo4j fixture, deferred to Task 7's integration test)**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: build clean, 139 tests still pass (no new tests; Phase A.5 is integration-tested in Task 7).

- [ ] **Step 4: Commit**

```bash
git add src/resolution/phase_a5_scoped_match.rs src/resolution/mod.rs
git commit -m "feat(resolution): add Phase A.5 — scoped name match for party + implements/requires"
```

---

### Task 6: Wire Phase A.5 into the resolution orchestrator

**Files:**
- Modify: `src/resolution/mod.rs`

The resolution orchestrator is the `resolve()` function (slice 9's `src/resolution/mod.rs`). It runs Phase A → Phase B → Phase C and returns `ResolutionStats`. Slice 10b adds Phase A.5 between A and B.

- [ ] **Step 1: Read `src/resolution/mod.rs` and find the `resolve()` function**

The existing structure is approximately:

```rust
pub async fn resolve(conn: &Connection) -> Result<ResolutionStats> {
    let mut stats = ResolutionStats::default();
    // Phase A — Imports
    stats.imports_resolved = phase_a_imports::run(conn).await?;
    // Phase B — Import-driven name match
    let phase_b = phase_b_import_driven::run(conn).await?;
    stats.calls_resolved_phase_b = phase_b.calls;
    // ... etc.
    // Phase C — Fallback name match
    let phase_c = phase_c_name_match::run(conn).await?;
    stats.calls_resolved_phase_c = phase_c.calls;
    // ... etc.
    Ok(stats)
}
```

(Actual signature may differ; verify.)

- [ ] **Step 2: Extend `ResolutionStats` with party/interface counters**

In `src/resolution/mod.rs` (or wherever `ResolutionStats` is defined):

```rust
#[derive(Debug, Clone, Default)]
pub struct ResolutionStats {
    pub imports_resolved: usize,
    pub calls_resolved_phase_b: usize,
    pub calls_resolved_phase_c: usize,
    pub refs_resolved_phase_b: usize,
    pub refs_resolved_phase_c: usize,
    pub types_resolved_phase_b: usize,
    pub types_resolved_phase_c: usize,
    pub unresolved_ambiguous: usize,
    pub unresolved_no_candidate: usize,
    // New for slice 10b:
    pub signatory_resolved: usize,
    pub controller_resolved: usize,
    pub observer_resolved: usize,
    pub implements_resolved: usize,
    pub requires_resolved: usize,
    pub party_unresolved: usize,
}
```

Add helper methods if useful:

```rust
impl ResolutionStats {
    pub fn total_resolved(&self) -> usize {
        // existing implementation ...
        // include new counters
        self.imports_resolved
            + self.calls_resolved_phase_b + self.calls_resolved_phase_c
            + self.refs_resolved_phase_b + self.refs_resolved_phase_c
            + self.types_resolved_phase_b + self.types_resolved_phase_c
            + self.signatory_resolved + self.controller_resolved + self.observer_resolved
            + self.implements_resolved + self.requires_resolved
    }
}
```

Adjust based on the actual existing `total_resolved()` body.

- [ ] **Step 3: Wire Phase A.5 in `resolve()`**

```rust
pub async fn resolve(conn: &Connection) -> Result<ResolutionStats> {
    let mut stats = ResolutionStats::default();

    // Phase A — Imports
    stats.imports_resolved = phase_a_imports::run(conn).await?;

    // Phase A.5 — Scoped party + implements/requires (new in slice 10b)
    let phase_a5 = phase_a5_scoped_match::run(conn).await?;
    stats.signatory_resolved = phase_a5.signatory_resolved;
    stats.controller_resolved = phase_a5.controller_resolved;
    stats.observer_resolved = phase_a5.observer_resolved;
    stats.implements_resolved = phase_a5.implements_resolved;
    stats.requires_resolved = phase_a5.requires_resolved;
    stats.party_unresolved = phase_a5.unresolved;

    // Phase B — Import-driven name match (existing)
    let phase_b = phase_b_import_driven::run(conn).await?;
    // ... existing wiring ...

    // Phase C — Fallback name match (existing)
    let phase_c = phase_c_name_match::run(conn).await?;
    // ... existing wiring ...

    Ok(stats)
}
```

- [ ] **Step 4: Update the `info!` log line in `cli::index` or wherever resolution stats are reported**

Find the line in `src/cli/index.rs` (or `src/sync/mod.rs`) that prints resolution stats:

```rust
info!(
    "resolution stats: imports={}, calls(B/C)={}/{}, refs(B/C)={}/{}, types(B/C)={}/{}, ambiguous={}, no_candidate={}",
    stats.imports_resolved, ...
);
```

Add the new counters:

```rust
// The single info! line is too long to read in a terminal — split into
// two logical lines (still emitted as two separate log records, which the
// user sees as adjacent lines).
info!(
    "resolution stats: imports={}, calls(B/C)={}/{}, refs(B/C)={}/{}, types(B/C)={}/{}",
    stats.imports_resolved,
    stats.calls_resolved_phase_b, stats.calls_resolved_phase_c,
    stats.refs_resolved_phase_b, stats.refs_resolved_phase_c,
    stats.types_resolved_phase_b, stats.types_resolved_phase_c,
);
info!(
    "resolution stats (auth): party(sig/ctrl/obs)={}/{}/{}, iface(impl/req)={}/{}, \
     ambiguous={}, no_candidate={}, party_unresolved={}",
    stats.signatory_resolved, stats.controller_resolved, stats.observer_resolved,
    stats.implements_resolved, stats.requires_resolved,
    stats.unresolved_ambiguous, stats.unresolved_no_candidate,
    stats.party_unresolved,
);
```

Do the same in `src/sync/mod.rs` if it has its own resolution-stats log line.

- [ ] **Step 5: Build, clippy, fmt, test, regression check**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
cargo test --test integration_daml_index 2>&1 | tail -3
```

Expected: 139 tests still pass (no new tests — orchestrator wiring is integration-tested in Task 7).

- [ ] **Step 6: Commit**

```bash
git add src/resolution/mod.rs src/cli/index.rs src/sync/mod.rs
git commit -m "feat(resolution): wire Phase A.5 into resolve() + extend ResolutionStats"
```

---

### Task 7: Integration test extensions for SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS edges

**Files:**
- Modify: `tests/fixtures/tiny-daml/Splice/Authorization.daml` (add a `requires` clause)
- Modify: `tests/integration_daml_index.rs`

The slice 10a Authorization.daml fixture already has signatory/controller/observer/implements clauses. We add one `requires` clause for completeness. Then assert all 5 new edge kinds land via Cypher queries.

- [ ] **Step 1: Extend the fixture with a `requires` clause**

Edit `tests/fixtures/tiny-daml/Splice/Authorization.daml`. Find the existing `interface Authorizable where ...` block. Add a second interface that requires it:

```daml
-- Subinterface that requires Authorizable.
interface Owned requires Authorizable where
  getOwnerParty : Party
```

Place after the `data AuthorizableView = ...` block and before `template AuthorizedCounter`.

> **Inherited assertions update required.** Adding `interface Owned` bumps the slice 10a `interface_count` from 1 to 2 and adds one more Field (`getOwnerParty`). Step 2 below updates those inherited assertions.

- [ ] **Step 2: Update inherited slice 10a count assertions**

In `tests/integration_daml_index.rs`, find these existing assertions and update them:

```rust
// Was: assert_eq!(interface_count, 1, "expected 1 Interface (Authorizable)");
assert_eq!(interface_count, 2, "expected 2 Interfaces (Authorizable, Owned)");
```

The `field_count >= 6` assertion stays (the count goes from 8 to 9, still satisfies the bound). The doc-comment breakdown above the field_count assertion should add a line:
```
//   - Owned interface: getOwnerParty                                       -> 1
// New minimum guaranteed: 9.
```

- [ ] **Step 3: Extend the integration test assertions**

Read `tests/integration_daml_index.rs`. Find the existing assertions block. After the inherited-update from Step 2, add:

```rust
// New for slice 10b: authorization edges.

// Authorization edges — exact counts based on the fixture content.
let signatory_edges: i64 = scalar(&conn,
    "MATCH (t:Template)-[:SIGNATORY]->() RETURN count(*) AS c").await;
assert!(signatory_edges >= 2,
    "expected ≥2 SIGNATORY edges (Counter→owner, AuthorizedCounter→owner); got {signatory_edges}");

let controller_edges: i64 = scalar(&conn,
    "MATCH (c:Choice)-[:CONTROLLER]->() RETURN count(*) AS c").await;
assert!(controller_edges >= 3,
    "expected ≥3 CONTROLLER edges (Increment/Reset/Bump); got {controller_edges}");

let observer_edges: i64 = scalar(&conn,
    "MATCH (t:Template)-[:OBSERVER]->() RETURN count(*) AS c").await;
assert!(observer_edges >= 1,
    "expected ≥1 OBSERVER edge (AuthorizedCounter→delegate); got {observer_edges}");

let implements_edges: i64 = scalar(&conn,
    "MATCH (t:Template)-[:IMPLEMENTS]->(i:Interface) RETURN count(*) AS c").await;
assert_eq!(implements_edges, 1,
    "expected 1 IMPLEMENTS edge (AuthorizedCounter→Authorizable)");

let requires_edges: i64 = scalar(&conn,
    "MATCH (i:Interface)-[:REQUIRES]->(j:Interface) RETURN count(*) AS c").await;
assert_eq!(requires_edges, 1,
    "expected 1 REQUIRES edge (Owned→Authorizable)");

// Named edge assertions.

// AuthorizedCounter has signatory owner.
let sig_owner: i64 = scalar(&conn,
    "MATCH (t:Template {qualified_name: 'Splice.Authorization.AuthorizedCounter'})\
     -[:SIGNATORY]->(f:Field {name: 'owner'}) RETURN count(*) AS c").await;
assert_eq!(sig_owner, 1, "AuthorizedCounter should have SIGNATORY → owner field");

// Bump choice has controller delegate.
let bump_ctrl: i64 = scalar(&conn,
    "MATCH (c:Choice {qualified_name: 'Splice.Authorization.AuthorizedCounter.Bump'})\
     -[:CONTROLLER]->(f:Field {name: 'delegate'}) RETURN count(*) AS c").await;
assert_eq!(bump_ctrl, 1, "Bump should have CONTROLLER → delegate field");

// AuthorizedCounter implements Authorizable.
let auth_impl: i64 = scalar(&conn,
    "MATCH (t:Template {qualified_name: 'Splice.Authorization.AuthorizedCounter'})\
     -[:IMPLEMENTS]->(i:Interface {qualified_name: 'Splice.Authorization.Authorizable'}) \
     RETURN count(*) AS c").await;
assert_eq!(auth_impl, 1);

// Owned requires Authorizable.
let owned_req: i64 = scalar(&conn,
    "MATCH (sub:Interface {qualified_name: 'Splice.Authorization.Owned'})\
     -[:REQUIRES]->(base:Interface {qualified_name: 'Splice.Authorization.Authorizable'}) \
     RETURN count(*) AS c").await;
assert_eq!(owned_req, 1);
```

- [ ] **Step 4: Verify SKIP path**

```bash
cargo test --test integration_daml_index 2>&1 | tail -10
```

Expected: 1 test, SKIPs cleanly (no env vars), result ok.

- [ ] **Step 5: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 139 tests pass (no new unit tests; integration test still SKIPs).

- [ ] **Step 6: Commit**

```bash
git add tests/fixtures/tiny-daml/Splice/Authorization.daml tests/integration_daml_index.rs
git commit -m "test(extraction): integration assertions for SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS/REQUIRES edges"
```

---

### Task 8: README + slice-10b-daml-edges tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read current README.md** to find the `## Status` section.

- [ ] **Step 2: Bump the Status section**

Replace with:

```markdown
## Status

In development. Currently implements **Slice 10b: Daml authorization
edges** — workspace initialization, indexing, resolution, incremental
sync, on-write embeddings, a stdio MCP server exposing 10 tools, Rust
+ Daml extraction with structural fidelity (templates, choices,
interfaces, viewtypes, exceptions, fields), AND authorization edges
(`signatory`/`controller`/`observer` party edges from templates/choices
to their Field targets, `implements` from templates to interfaces,
`requires` between interfaces).

Resolution gained a new Phase A.5 (scoped name match) that builds a
`HashMap<qualified_name, qid>` once and resolves party-edge refs by
`<scope_qname>.<unresolved_name>` lookup. Implements/Requires resolve
against workspace-wide Interface symbols by qualified-name suffix
match.

Daml v2.0 depth is complete; v2.1+ slices may add file watchers,
framework-specific resolvers, cross-package indexing, or a real
`tree-sitter-daml` grammar swap when one becomes available.
```

Quick start and Development sections need no changes.

- [ ] **Step 3: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected: 139 tests pass total.

- [ ] **Step 4: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 10b (Daml authorization edges)"
git tag slice-10b-daml-edges
```

- [ ] **Step 5: Verify**

```bash
git log --oneline | head -15
git tag --list | grep slice
```

Expected tags include `slice-10b-daml-edges`. Full list:
`slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`, `slice-6-mcp`, `slice-7-embeddings`, `slice-8-semantic-search`, `slice-9-daml`, `slice-10a-daml-structure`, `slice-10b-daml-edges`.

---

## Slice 10b Done

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — 139 tests pass, all integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass, including extended `integration_daml_index` with new edge assertions
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 CODEGRAPH_RS_TEST_EMBEDDINGS=1 cargo test -- --test-threads=1` — embeddings unchanged; new edge kinds visible to MCP traversal tools
- [ ] Manual: in Neo4j Browser, on a real Daml workspace (Splice/daml-finance):
  - `MATCH (t:Template)-[:SIGNATORY]->(f) RETURN labels(f), count(*)` returns mostly `[Symbol, Field]` targets (some `[Symbol, Variable]` if party expressions reference non-field workspace symbols)
  - `MATCH (c:Choice)-[:CONTROLLER]->(f:Field) RETURN c.qualified_name, f.qualified_name LIMIT 10` shows choice-to-party authorization
  - `MATCH (a:Template)-[:IMPLEMENTS]->(b:Interface) RETURN a.qualified_name, b.qualified_name` shows template-interface relationships
- [ ] Manual: `_callers` MCP tool on a Field returns the Templates that have SIGNATORY edges to it
- [ ] Manual: `_impact` MCP tool traverses through SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS/REQUIRES (these are reverse-transitive-closure candidates per spec §10 — verify by checking `src/graph/traverse.rs::impact` includes them)
- [ ] Resolution stats log line includes party + interface counts
- [ ] No regression of slice 10a functionality

When all check, **v2.0 Daml-depth push is complete**.

### Small follow-ups noted during this slice

- **`_impact` MCP tool edge-kind list.** Spec §10 defines impact as "Reverse-transitive closure of CALLS/REFERENCES/TYPE_OF". Slice 10b adds 5 new edge kinds that arguably should be included in impact analysis (changing a signatory field affects who can create the contract). Worth a small follow-up to extend `src/graph/traverse.rs::impact`'s relationship list.
- **`pass_e_party_edges` segment-splitting.** The current implementation splits dotted identifiers on `.` to handle `Set.fromList` cases where the parser emits them as a single dotted token. If the grammar emits them as separate tokens, the splitting is a no-op. Worth probing with a real Splice/daml-finance file to confirm.
- **`find_interface_by_name` suffix match ambiguity.** If two interfaces in the workspace share a short name (`Authorizable` in module A and module B), the function returns the first hit found. For Daml-finance scale, ambiguity is rare; if it bites in practice, the fix is to require fully-qualified names in `implements`/`requires` clauses or to enumerate candidates and require disambiguation.
- **Phase A.5 fallback to Phase C.** Currently, party-kind refs that miss Phase A.5 are counted as unresolved and Phase C doesn't re-process them. If we want fallback (per spec §4.2), Phase C needs to know about the new kinds. Defer to v2.1.
