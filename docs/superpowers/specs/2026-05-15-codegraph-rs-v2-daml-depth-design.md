# codegraph-rs v2.0 — Daml-depth design

**Status:** Approved, awaiting implementation plan.
**Baseline:** v1 shipped at `slice-9-daml` (HEAD `b3fdc0a` on `main`, 97 tests pass).
**Scope:** Two sequential slices (10a, 10b) deepening Daml extraction. v2.1+ items deferred.

---

## 1. Goal and scope

### 1.1 Goal

Make Daml extraction in `codegraph-rs` deep enough to answer the questions a real Daml engineer actually asks:

- *"Who can exercise this choice?"*
- *"Which templates implement this interface?"*
- *"What's the authorization model around this contract?"*
- *"Which fields back the signatory expression of this template?"*

v1 emits Templates and Choices but they are regex-anchored opaque blobs with line-approximate spans, `parse_reliable: false`, and no internal structure. v2 makes them first-class graph citizens: byte-precise spans, internal field children, and party-shaped edges to the Symbols that back signatory/controller/observer expressions.

### 1.2 Scope — two sequential slices

| Slice | Scope | Tag |
|---|---|---|
| **10a — Daml structure + fields** | Replace regex `pass_d_templates_and_choices` with tree-sitter queries that filter `(variable)` nodes by Daml keyword. Add NodeKinds: `Interface`, `Viewtype`, `Exception`, `Field`. Fields are CONTAINS-children of their parent Template / Choice / Interface. | `slice-10a-daml-structure` |
| **10b — Daml authorization edges** | Extract `signatory` / `controller` / `observer` party expressions; resolve identifier-shaped operands to Field Symbols (preferred) or workspace-scoped Symbols (fallback). Decompose multi-party expressions (lists, set-builders) into one ref per identifier — see §3.3. Extract `implements` / `requires` relationships between Templates and Interfaces. Add 5 new EdgeKinds + 5 new UnresolvedKinds. | `slice-10b-daml-edges` |

Each slice is independently shippable, ~7-9 tasks, passing its own integration test gate. v1's pain point ("templates are regex-only opaque blobs") is gone after 10a alone.

### 1.3 Out of scope for v2.0

- **Contract keys / maintainers.** Deprecated in Daml 2.x in favor of interface-based contract lookups. The original slice 10c plan is dropped.
- **File watchers, framework-specific resolvers, cross-crate dep indexing, web visualizer, LSP server.** Potential v2.1+ slices; the user's chosen v2 direction is *depth, not breadth*.
- **A real `tree-sitter-daml` grammar.** Confirmed unavailable: crates.io has no `tree-sitter-daml` crate; the prominent Daml editor tooling (DLC-link/zed-daml-lsp) uses tree-sitter-haskell as a proxy, identical to our approach. Spec §2 already defers grammar work to "a separate project tracked outside this spec".

---

## 2. New schema additions

### 2.1 New NodeKinds (slice 10a)

| NodeKind | Daml construct | qualified_name shape | parse_reliable | Embed-eligible? |
|---|---|---|---|---|
| `Interface` | `interface I where ...` | `Module.I` | false (keyword anchor) | yes (add to `EMBED_KINDS_CYPHER_LIST`) |
| `Viewtype` | `viewtype V` inside an interface body | `Module.I.V` | false (keyword anchor) | yes |
| `Exception` | `exception E with ... where message ...` | `Module.E` | false (keyword anchor) | yes |
| `Field` | `with`-block typed bindings inside a Template / Choice / Interface (e.g., `owner : Party`) | Template field: `Module.Template.field_name`. Choice field: `Module.Template.Choice.field_name`. Interface field: `Module.Interface.field_name`. The qualified-name carries the full container chain so a Field is unambiguously addressable across the workspace. | true (haskell parses `name : type` as a signature) | no (fields are not embed-eligible per spec §8 — they're leaf state) |

Templates and Choices remain `parse_reliable: false`. The *keyword anchors* are still detected by text-matching against haskell's `(variable)` nodes; only the surrounding structure (constructor names, with-block field signatures) is grammar-clean. Everything *inside* a template body — signatory expressions, choice signatures, field types — gets `parse_reliable: true`.

**Why `Viewtype` is a NodeKind rather than an edge.** `viewtype V` inside an interface body names an existing `data V = ...` Struct. A cleaner model would be an edge `Interface -[VIEWTYPE]→ Struct(V)`, with the Struct already existing from `data` extraction. We choose the NodeKind model anyway because: (a) it makes Viewtype embed-eligible without bridging through another node, which matters for semantic search queries like *"what's the view of this contract?"*; (b) it keeps slice 10a's surface symmetric — every Daml keyword-anchored construct becomes a NodeKind. The alternative (Viewtype as `EdgeKind::Viewtype` introduced in slice 10b) is a viable v3 refactor if Viewtype-as-Symbol turns out to be redundant in practice.

Slice 7's `EMBED_KINDS_CYPHER_LIST` constants in `src/graph/status.rs` and `src/vectors/query.rs` each gain three entries: `'Interface', 'Viewtype', 'Exception'`. The embedding pipeline picks them up automatically — no orchestrator changes needed.

### 2.2 New EdgeKinds (slice 10b)

| EdgeKind | Direction | Source-side construct | Target |
|---|---|---|---|
| `SIGNATORY` | `Template → Symbol` | `signatory <expr>` in a template's where-block | Typically a Field of the same Template; can be a free Symbol if the expression names something workspace-scoped |
| `CONTROLLER` | `Choice → Symbol` | `controller <expr>` in a choice block | Same target shape as Signatory |
| `OBSERVER` | `Template → Symbol` | `observer <expr>` (optional in templates) | Same target shape |
| `IMPLEMENTS` | `Template → Interface` | `interface instance I for T where ...` | An `Interface` node anywhere in the workspace |
| `REQUIRES` | `Interface → Interface` | `interface I requires J where ...` | An `Interface` node anywhere |

### 2.3 New UnresolvedKinds (slice 10b)

Extraction emits `UnresolvedRef { kind: Signatory | Controller | Observer | Implements | Requires, ... }`. Resolution converts them to the corresponding edge types per §4.1.

### 2.4 No data migration

All additions are pure label / property additions at the Neo4j layer; no constraint changes, no FTS index changes, no vector index changes. The `apply_schema` step in `src/db/schema.rs` is unchanged. `schema_version` stays at **1**.

For details of the Rust ↔ Neo4j compatibility story (adding `UnresolvedRef.scope_qid` as a new field, v1 records lacking it), see §4.3.

---

## 3. Extraction strategy (the technical core)

### 3.1 The central observation

tree-sitter-haskell parses Daml-specific keywords as `(variable)` nodes inside `(apply ...)` expressions. The Zed Daml LSP extension (the most prominent non-Daml-Studio Daml editor tooling) confirms this approach.

`template Counter with owner : Party where signatory owner` parses (approximately) as:

```
(top_splice
  (apply
    function: (variable "template")
    argument: (constructor "Counter")
    ...))
```

`signatory owner` inside the where-block parses as:

```
(apply
  function: (variable "signatory")
  argument: (variable "owner"))
```

Same shape applies to every Daml keyword: `choice`, `controller`, `observer`, `interface`, `viewtype`, `implements`, `requires`, `exception`, `signatory`.

We exploit this by writing tree-sitter queries that filter `(apply (variable))` nodes by keyword text, then walk the surrounding tree for the data we want.

### 3.2 Slice 10a — replace regex Pass D with tree-sitter queries

New queries in `src/extraction/languages/daml.scm` (illustrative; the implementer must verify exact node shapes against `tree-sitter-haskell-0.23.1/src/node-types.json` per the same workflow used in slices 5-9):

```scheme
; Template anchor.
((apply
  function: (variable) @kw
  argument: (constructor) @template.name) @template.def
  (#eq? @kw "template"))

; Interface anchor.
((apply
  function: (variable) @kw
  argument: (constructor) @interface.name) @interface.def
  (#eq? @kw "interface"))

; Viewtype.
((apply
  function: (variable) @kw
  argument: (constructor) @viewtype.name) @viewtype.def
  (#eq? @kw "viewtype"))

; Exception.
((apply
  function: (variable) @kw
  argument: (constructor) @exception.name) @exception.def
  (#eq? @kw "exception"))

; Choice anchor (shape depends on haskell's parse of `choice C : T ...`).
((apply
  function: (variable) @kw
  argument: (_) @choice.body) @choice.def
  (#eq? @kw "choice"))

; Field: typed bindings of the form `name : Type` inside with-blocks.
; Haskell parses these as `signature` nodes (or `top_splice/infix` in some Daml
; variants — slice 5/6 found Daml's single-colon `:` lands in `top_splice/infix`).
; The implementer must verify the actual subtree shape.
;
; CRITICAL: this query shape ALSO matches top-level function signatures
; (`helperFn : Int -> Int`). The Function extractor in pass 2 already captures
; those. To distinguish a Field from a top-level signature, the extractor MUST
; check the ancestor chain at emit time: only emit as Field if the signature
; node has a Template / Choice / Interface / Exception keyword-anchored
; `(apply)` somewhere in its ancestor chain (i.e., it sits inside a with-block
; under a Daml-anchored construct). Top-level signatures keep their existing
; Function-tied behavior. Pass 3 must NOT emit Fields for signatures that
; pass 2 already paired with a Function.
(signature name: (variable) @field.name type: (_) @field.type) @field.def
```

**Curried `apply` shape.** Haskell parses multi-argument applications left-associatively: `template Counter with owner: Party where ...` produces a nested chain like `(apply (apply (apply (variable "template") (constructor "Counter")) (where_block ...)) ...)`. The keyword-anchor queries above implicitly rely on tree-sitter's pattern matcher walking the tree until it finds a match — so an `(apply function: (variable "template") argument: (constructor) @template.name)` pattern matches at the **innermost** apply in the chain (the one where `template` is directly applied to its first argument). The implementer must verify by capturing real fixture parses and confirm the match position; if patterns fail to match because they expected the outermost apply, adjust by anchoring at the leftmost-variable apply with a `descendant` traversal in the query.

Pass ordering inside `extract_daml` (the implementer assigns concrete names; the slice 9 codebase uses an idiosyncratic mix of `pass_a`/`pass_b`/`pass_c`/`pass_d`):

1. **Module + Imports** (existing, unchanged).
2. **Top-level haskell-parseable declarations** (existing): Functions, TypeAliases, Structs from `data_type`.
3. **Daml keyword-anchored constructs** (NEW, replaces the regex pass): Templates, Choices, Interfaces, Viewtypes, Exceptions, Fields. All via tree-sitter queries.
4. **Call-site UnresolvedRefs** (existing): function applications + `exercise` choices.

Pass 3 must run after pass 2 so Field qualified-name prefixing works (Fields nest under their parent Template/Choice/Interface, whose definitions are emitted in pass 3 itself — within pass 3 the implementer ensures Template comes before its Fields).

The Field extraction in Pass D produces CONTAINS edges from the parent Template / Choice / Interface to each field, matching the existing pattern (File → Module → Function/Type/etc.).

### 3.3 Slice 10b — extract authorization edges

Additional queries:

```scheme
; Signatory party.
((apply
  function: (variable) @kw
  argument: (_) @signatory.target) @signatory.def
  (#eq? @kw "signatory"))

; Controller party (inside choice scope).
((apply
  function: (variable) @kw
  argument: (_) @controller.target) @controller.def
  (#eq? @kw "controller"))

; Observer party.
((apply
  function: (variable) @kw
  argument: (_) @observer.target) @observer.def
  (#eq? @kw "observer"))

; Template implements Interface.
((apply
  function: (variable) @kw
  argument: (constructor) @implements.interface) @implements.def
  (#eq? @kw "implements"))

; Interface requires Interface.
((apply
  function: (variable) @kw
  argument: (constructor) @requires.interface) @requires.def
  (#eq? @kw "requires"))
```

Each match emits an `UnresolvedRef { kind: <Signatory|Controller|Observer|Implements|Requires>, unresolved_name: <target_text>, source_qid: <Template|Choice|Interface qid>, scope_qid: Some(<enclosing-template-qid>) }`. The `scope_qid` is new — see §4.1.

**Multi-party expression decomposition (Signatory / Controller / Observer).** Real Daml uses several expression shapes for parties:

| Daml expression | Decomposition |
|---|---|
| `signatory owner` | One ref: `unresolved_name = "owner"` |
| `signatory [owner, delegate]` | Two refs: `"owner"` and `"delegate"` |
| `signatory $ Set.fromList [a, b]` | Two refs: `"a"` and `"b"` (the `Set.fromList` and `$` are dropped; identifiers inside the list are extracted) |
| `signatory $ fromOptional [] mObservers` | One ref: `"mObservers"` (single identifier inside the optional) |
| `signatory (parties config)` | One ref: `"config"` (or `"parties"`, depending on which the extractor prefers — see below) |
| `signatory (owner :: signatories)` | Two refs: `"owner"` and `"signatories"` |

Concretely, after matching the keyword-anchored `(apply)`, the extractor runs a sub-walk over the `@target` subtree:

1. Collect every `(variable)` and `(constructor)` leaf inside the subtree.
2. Filter out known wrapper identifiers: `Set.fromList`, `Set.singleton`, `fromOptional`, `fromSome`, `concat`, `concatMap`, the `$` operator (it's actually parsed as an operator, not a variable), and list/tuple syntactic nodes.
3. For each remaining identifier, emit one `UnresolvedRef` with `unresolved_name` = that identifier's text. All refs share the same `source_qid` and `scope_qid`; they only differ in line/col.

For `parties config` (a function call), this rule emits **both** "parties" and "config" — the resolver's Phase A.5 + Phase C will resolve each independently. False positives (e.g., resolving `parties` to a workspace-wide helper function) get the edge, but the relevant `config` ref also gets resolved, so the graph still answers "who can exercise this?" correctly. Document this as acceptable v2.0 precision; tighter decomposition is a v2.1 refinement.

`Implements` and `Requires` follow the simpler single-identifier model since their argument is always a single `(constructor)`.

### 3.4 Why this is strictly better than regex Pass D

| Aspect | Regex (v1) | Tree-sitter (v2) |
|---|---|---|
| Span precision | Line-anchored, approximate | Byte-precise from AST |
| Nested constructs | Hard (would need recursive regex) | Free (queries walk the tree) |
| Whitespace robustness | Fragile (depends on indentation matching) | Robust |
| Adding a new keyword | New regex + state machine per kind | Add one `#eq?` filter |
| Cross-reference resolution | Manually compute byte-offset → enclosing-span | Use existing `enclosing_symbol_qid` helper (lifted in slice 9 task 4) |

---

## 4. Resolution and migration

### 4.1 `scope_qid` on UnresolvedRef

`signatory owner` inside `template Counter with owner: Party where ...` must resolve to the `Counter.owner` Field, **not** to some unrelated `owner` Symbol elsewhere in the workspace. The current resolver (`src/resolution/`) only has `source_qid` (the symbol referencing the name); it has no notion of "look up names in this enclosing scope first."

Slice 10b adds an `Option<String> scope_qid` field to `UnresolvedRef`. For party-edge kinds (Signatory, Controller, Observer), `scope_qid` is the enclosing Template's qid. Resolution adds a new phase that runs *before* Phase B's import-driven match:

**Phase A.5 — Scoped name match (new).**
For each `UnresolvedRef` with `scope_qid: Some(s)`, look up `<s>.<unresolved_name>` in the Symbol set. If found, emit the appropriate edge directly. If not, fall through to **Phase C only** (party kinds skip Phase B's import-driven match — party names are always local scope, never workspace-imported).

**Algorithm.** Phase A.5 builds a `HashMap<String, String>` once (mapping `qualified_name` → `qid`) by issuing one Cypher `MATCH (s:Symbol) RETURN s.qualified_name AS qname, s.qid AS qid` at phase start. Each subsequent ref lookup is `O(1)` against the map. Total cost: one Cypher per resolve invocation, regardless of ref count. This is cheaper than Phase C's existing per-ref name-match and amortizes well even for small workspaces.

For `Implements` / `Requires` kinds, `scope_qid` is `None` — they resolve against workspace-wide Interface symbols by qualified_name, the same shape as `IMPORTS` resolution (also a Phase A-style lookup).

### 4.2 Phase assignments

| UnresolvedKind | Resolution phases tried (in order) |
|---|---|
| `Signatory`, `Controller`, `Observer` | Phase A.5 (scoped HashMap lookup) → Phase C (fallback workspace name-match). Phase B is intentionally skipped — party names are scope-local, not workspace-imported. |
| `Implements`, `Requires` | Phase A-shape (qualified_name lookup of an Interface node, same shape as `IMPORTS`). No fallback — if an unknown interface name is referenced, the edge stays unresolved (existing behavior for unknown qualified names). |

### 4.3 Backwards compatibility

The slice changes are additive:

- **New Neo4j labels** (`Interface`, `Viewtype`, `Exception`, `Field`) — Neo4j labels are open, no constraint changes.
- **New relationship types** (`SIGNATORY`, `CONTROLLER`, `OBSERVER`, `IMPLEMENTS`, `REQUIRES`) — relationship types are open, no schema changes.
- **New `UnresolvedRef.kind` enum values** — `kind` is serialized as a string property, free-form.
- **New `UnresolvedRef.scope_qid` property** — additive, `Option<String>`. Existing v1 records have `scope_qid` absent; resolver treats absent as `None`.

Old v1-indexed workspaces upgraded to v2 binaries keep their existing graph queryable. Running `codegraph-rs sync` re-indexes touched files and adds new nodes/edges. Running `codegraph-rs index` from scratch produces the full v2 surface.

### 4.4 `schema_version` stays at 1

Per §2.4, all changes are forward-compatible. The first non-additive change in some future slice (file-watcher introducing a new mandatory config field, say) is the right place to bump.

---

## 5. Testing strategy

### 5.1 Fixture extension

`tests/fixtures/tiny-daml/` gains `Splice/Authorization.daml`:

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

> **Daml version compatibility.** The `interface instance I for T where ...` syntax inside a template body is the Daml 2.7+ form (pre-2.7 used a separate top-level `instance` declaration). The fixture targets 2.7+ syntax. **Slice 10a Task 1 includes a verification step**: attempt to compile the fixture with the local `daml` compiler (if installed) or paste it into Daml Studio. If the compiler rejects it, switch to the pre-2.7 top-level form and update the fixture inline before proceeding. The extractor changes (tree-sitter queries) are insensitive to which form is used as long as the `interface instance` keyword sequence is in the source — only the IMPLEMENTS-edge source-side computation changes (sourced from Template's nested where-block in 2.7+, sourced from a top-level declaration node in 2.6 and earlier).

Per-slice-10a expectations (asserted by the integration test):

- 1 new Module (`Splice.Authorization`).
- 1 new Interface (`Authorizable`).
- 1 new Viewtype (`AuthorizableView`) — note this also generates a `data AuthorizableView` parsed as a Struct.
- 1 new Template (`AuthorizedCounter`) with 3 Field children (`owner`, `delegate`, `count`).
- 1 new Choice (`Bump`) — choice arguments depend on grammar exposure; may have 0 or 1 Field children.
- 1 new Exception (`InsufficientBalance`) with 2 Field children (`actual`, `required`).
- At least 1 Import (`Splice.Counter` from the existing slice 9 fixture).

### 5.2 Unit tests

Following slices 5-9's pattern, append to `tests/extraction_unit.rs`.

**Slice 10a (10 tests):**

- `daml_extracts_interface_name_and_qualified_name`
- `daml_extracts_viewtype_inside_interface_scope`
- `daml_extracts_exception_with_parse_reliable_false`
- `daml_template_with_fields_emits_field_children`
- `daml_choice_with_fields_emits_field_children`
- `daml_byte_precise_template_span` — Template start/end lines match actual AST extent, not regex line block
- `daml_pass_d_replaces_regex_no_regression` — comprehensive: indexes Counter.daml fixture from slice 9 and verifies counts match prior slice 9 baseline (regression guard)
- `daml_field_qualified_name_nests_under_template` — `Foo.Counter.owner`, not `Foo.owner`
- `daml_field_qualified_name_nests_under_choice` — `Foo.Counter.Bump.delta` (4-segment qname when Choice has its own `with`-block)
- `daml_top_level_signature_not_emitted_as_field` — disambiguation guard: `helperFn : Int -> Int` at top level remains a Function signature, NOT a stray Field

**Slice 10b (10 tests):**

- `daml_signatory_emits_unresolved_ref_with_scope_qid`
- `daml_controller_inside_choice_uses_choice_scope`
- `daml_observer_emits_unresolved_ref`
- `daml_implements_targets_workspace_interface`
- `daml_requires_emits_interface_to_interface_ref`
- `daml_signatory_resolves_to_template_field_in_phase_a5`
- `daml_party_unresolved_ref_falls_through_to_phase_c_when_scoped_miss`
- `daml_multi_party_list_decomposed` — `signatory [a, b]` emits two refs, not one
- `daml_multi_party_set_fromlist_decomposed` — `signatory $ Set.fromList [a, b]` emits two refs; `Set.fromList` is filtered out
- `daml_party_unwrapped_from_optional` — `signatory $ fromOptional [] mObservers` emits one ref for `mObservers`

### 5.3 Integration tests

Both slices extend (or add files alongside) `tests/integration_daml_index.rs` — same env gate `CODEGRAPH_RS_TEST_NEO4J=1`.

**Slice 10a integration assertions:**

```cypher
MATCH (n:Interface) RETURN count(n)              -- expect ≥ 1
MATCH (n:Viewtype) RETURN count(n)               -- expect ≥ 1
MATCH (n:Exception) RETURN count(n)              -- expect ≥ 1
MATCH (n:Field) RETURN count(n)                  -- expect ≥ 5
MATCH (n:Field {qualified_name: "Splice.Authorization.AuthorizedCounter.owner"}) RETURN count(n) = 1
```

**Slice 10b integration assertions:**

```cypher
MATCH (t:Template {name: 'AuthorizedCounter'})-[:SIGNATORY]->(f:Field {name: 'owner'}) RETURN count(*) = 1
MATCH (c:Choice {name: 'Bump'})-[:CONTROLLER]->(f:Field {name: 'delegate'}) RETURN count(*) = 1
MATCH (t:Template {name: 'AuthorizedCounter'})-[:IMPLEMENTS]->(i:Interface {qualified_name: 'Splice.Authorization.Authorizable'}) RETURN count(*) = 1
```

### 5.4 Embedding test inheritance

Slice 7's `tests/integration_vectors.rs` needs no structural changes — `EMBED_KINDS_CYPHER_LIST` already names the new Daml kinds. Slice 10a's integration test adds one assertion under the dual gate (`CODEGRAPH_RS_TEST_NEO4J=1` AND `CODEGRAPH_RS_TEST_EMBEDDINGS=1`):

```rust
let embedded_interfaces: i64 = scalar(&conn,
    "MATCH (i:Interface) WHERE i.embedding IS NOT NULL RETURN count(i) AS c").await;
assert!(embedded_interfaces >= 1, "interfaces should be embedded by slice 7's pipeline");
```

---

## 6. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| **tree-sitter-haskell node shapes differ from expected.** Slices 5-9 already had to verify `signature` (Daml uses `:` not `::`, ends up as `top_splice/infix`) and `apply.function:` field. v2's queries assume `(apply function: (variable) argument: (constructor))` for keyword anchors — that may need adjustment per-keyword. | High | Each task in slice 10a/10b starts with a node-types.json inspection step. The implementer subagent verifies and adjusts before writing queries. |
| **Daml keywords as user identifiers.** A user could legitimately name a function `signatory` or `observer` (uncommon but possible). Our `#eq?` filter would misfire. | Low | The query also requires the variable be in `function:` position of an `apply` and its argument shape match the expected Daml construct. False positives on `let x = signatory + 1` would need `+ 1` to parse as a constructor — extremely unlikely. Document as a known limitation. |
| **`interface instance ... for ... where` Daml 2.x syntax.** Spec was written from spec text; the actual Daml 2.x syntax may differ. | Medium | Slice 10a's fixture step verifies against the real Daml compiler (or Splice/daml-finance examples). If the fixture doesn't typecheck under real Daml, fix syntax before extending tests. |
| **`scope_qid` resolution change touches `src/resolution/`.** That module hasn't been modified since slice 3; adding a new resolution path could regress existing resolution. | Medium | Slice 10b's Phase A.5 runs in addition to existing Phase B/C. Existing tests (`resolution_unit.rs`, `integration_resolve.rs`) must pass unchanged. |
| **Backwards compat with v1-indexed workspaces.** A user with an existing `.codegraph/` directory should be able to upgrade to v2 binary and (a) re-run `sync` to pick up new node/edge kinds, (b) keep existing graph queryable. | Low | Schema is fully additive (per §4.3). Slice 10a's first task includes a manual smoke test: index a workspace with v1 binary, switch to v2, run `sync`, assert no errors and new nodes appear. |
| **Resolution performance.** Phase C name-match is currently the slowest step. Adding Phase A.5 (`scope_qid` HashMap lookups) could push latency further. | Low | Workspace sizes are small (<10k symbols typical). If a benchmark shows regression, the optimization is straightforward: cache `<scope_qid>.<name>` lookups in a HashMap built once per resolve. Defer until measured. |

---

## 7. Slice contracts summary

### 7.1 Slice 10a — Daml structure + fields

**Tag:** `slice-10a-daml-structure`
**Estimated tasks:** ~8

1. NodeKind additions (Interface, Viewtype, Exception, Field) + EMBED_KINDS extensions.
2. Tree-sitter query for templates (replaces regex template_re).
3. Tree-sitter query for choices.
4. Tree-sitter query for interfaces + viewtypes.
5. Tree-sitter query for exceptions.
6. Tree-sitter query for fields (with-block bindings).
7. Fixture extension + integration test additions.
8. README bump + tag.

**Done criteria:** All 97 v1 tests still pass; new unit tests pass; integration test against extended fixture asserts new node counts; manual verification on a real Daml workspace (Splice or daml-finance) shows Template fields appearing.

### 7.2 Slice 10b — Daml authorization edges

**Tag:** `slice-10b-daml-edges`
**Estimated tasks:** ~9

1. EdgeKind additions (SIGNATORY, CONTROLLER, OBSERVER, IMPLEMENTS, REQUIRES) + UnresolvedKind additions.
2. `UnresolvedRef.scope_qid: Option<String>` schema addition + DB write/read path.
3. Tree-sitter queries for signatory / controller / observer.
4. Tree-sitter queries for implements / requires.
5. Resolution Phase A.5 (scoped name match) implementation.
6. Wire phases into the resolution orchestrator.
7. Fixture extension verification (Authorization.daml).
8. Integration test with full Cypher assertions for all 5 new edge types.
9. README bump + tag.

**Done criteria:** All slice-10a tests still pass; new resolution_unit tests pass; integration test asserts SIGNATORY/CONTROLLER/OBSERVER/IMPLEMENTS/REQUIRES edges land with correct targets; manual smoke check that `_callers`/`_callees`/`_impact` MCP tools still work and return sensible results when called on a Choice (existing graph traversal should be edge-type-agnostic).

---

## 8. Out of scope / future

| Item | Why deferred | Future slice |
|---|---|---|
| File watcher / git post-commit hook | Spec §9 lists as v2 territory; user picked depth over breadth | v2.1+ |
| Framework-specific resolvers (axum, tokio, Daml stdlib) | Resolver pipeline is generic; framework knowledge is a separate concern | v2.1+ |
| Cross-crate / external dep indexing | Spec §7 "Not in v1" | v2+ |
| Real `tree-sitter-daml` grammar | Multi-month external project; current haskell-proxy approach is community-validated | If/when grammar appears |
| LSP server | Different consumer surface | v3 |
| Web visualizer | Spec §2 says "Neo4j Browser replaces it" for v1 | v3 |

---

## 9. References

- Source spec: [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](2026-04-22-codegraph-rs-design.md) — §4 (node labels including reserved Template/Choice/Field), §7 (resolution pipeline), §8 (embedding eligibility).
- Slice 9 plan: [`docs/superpowers/plans/2026-05-14-codegraph-rs-slice-9-daml.md`](../plans/2026-05-14-codegraph-rs-slice-9-daml.md) — current Daml extractor implementation (the regex Pass D this spec replaces).
- DLC-link/zed-daml-lsp — community Daml editor extension confirming the haskell-proxy approach: `https://github.com/DLC-link/zed-daml-lsp` (specifically `languages/daml/highlights.scm` for the Daml keyword set).
- tree-sitter-haskell 0.23.1 — `~/.cargo/registry/src/index.crates.io-*/tree-sitter-haskell-0.23.1/src/node-types.json` for grammar shapes.
