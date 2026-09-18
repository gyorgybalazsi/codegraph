# codegraph-rs — type-aware Rust resolution (Phase R)

**Goal:** resolve a Rust call to the item the compiler would call, so `callers`, `callees` and `impact` answer correctly for Rust.
**Status:** proposed
**Baseline:** the v1 design (`2026-04-22-codegraph-rs-design.md`) §7, whose "Not in v1" list excludes type-aware dispatch. This design supersedes that exclusion for Rust.
**Verification:** three fresh-context reviewers checked draft 1 against the code on 2026-09-18. Their findings are folded in; §8 records what changed.

---

## 1. Why

A comparison against rust-analyzer on 2026-09-17 measured codegraph-rs on a 32-line fixture and on `alt-canton`. Every Rust method-call question returned 0 correct results.

| Question | Ground truth | codegraph-rs |
|---|---|---|
| calls to `A::submit` | 5 | 0 |
| calls to `B::submit` | 1 | 0 |
| calls to `SequencerHandle::submit` | 39 | 0 |
| calls to `run::subscribe` | 8 | 0 |
| references to `Settled::tolerated` | 7 | not expressible |

Two causes, both measured:

- **The receiver stays in the name.** `src/extraction/languages/rust.rs:175` splits a callee's text on `::` only, so `a.submit(1, 2)` is recorded under the name `a.submit`. No symbol has that name, so the ref ends `no_candidate`. In `alt-canton`, 57 refs named `*.submit` and 91 named `*.subscribe` ended that way.
- **A bare name matches every type.** `A::submit(a, 9, 10)` is recorded as `submit`, which matches `A::submit`, `B::submit` and `C::submit`. Phase C records `ambiguous` and writes no edge. codegraph holds no type information, so it cannot choose.

Import-driven resolution does not rescue this. Rust qualified names come from file paths, such as `core_lib::src::lib::A`, and Rust has no `Module` node per file, so Phase A matches nothing. The fixture index reported `imports=0`.

`Settled::tolerated` is a struct field, and the Rust extractor emits no field symbols.

## 2. Constraints

- rustc: `rust-toolchain.toml` pins **1.98.1**. Every `ra_ap_*` 0.0.352 crate declares `rust-version = "1.98"` and edition 2024, so the current pin, 1.91, cannot build them. The bump lands in slice 1.
- Analyzer crates: **`ra_ap_ide`, `ra_ap_ide_db`, `ra_ap_load-cargo`, `ra_ap_project_model`, `ra_ap_vfs`, `ra_ap_paths`, all `=0.0.352`**. They carry no API stability guarantee.
- `unicode-ident` must satisfy **`>=1.0.22, <1.0.25`**. `ra-ap-rustc_lexer-0.166.0/src/lib.rs:37` is a `const` panic comparing `unicode_properties::UNICODE_VERSION` with `unicode_ident::UNICODE_VERSION`. Measured: `unicode-properties` 0.1.4 is `(17,0,0)`; `unicode-ident` 1.0.22 and 1.0.24 are `(17,0,0)`; 1.0.25 and later are `(18,0,0)`.
- Dependency ceiling: the lockfile grows by at most **230 crates**. Measured on the spike: 222.
- Sysroot: the loader passes **`sysroot: Some(RustLibSource::Discover)`**. Measured twice, on `find_all_refs` and on `StaticIndex::compute`: without it, `A::submit` yields 4 references instead of 5, silently dropping the call through `Arc<A>`.
- Positions: the analyzer works in **UTF-8 byte offsets** (`TextSize`/`TextRange`). `LineIndex::line_col` and `LineIndex::offset` convert, and `LineCol { line, col }` is **zero-based on both**, with `col` a byte offset within the line. codegraph stores one-based lines and zero-based byte columns, so every conversion adds or removes exactly one on the line.
- Neo4j schema: **two `UnresolvedRef` properties** (`name_line`, `name_col`), **two more** written by the analyzer stage (`target_qid`, `target_source`), and **one new index** on `(:Symbol)` over `root_name, relative_path`. No new node label: `Field` already exists and Daml already uses it.
- Index time: no budget is asserted yet. Slice 2 measures a release build and sets one. Measured so far: `StaticIndex::compute` took 34.7 s on a 45-line fixture in a **debug** build, and `rust-analyzer scip`, a release build, took 75 s on `alt-canton`.

## 3. Approach

Embed rust-analyzer as a library, compute one `StaticIndex` per Cargo workspace, and store the target it proves for each call site on the `UnresolvedRef` that already records that site. Resolution then turns stored targets into edges.

**Phase R is driven by codegraph's refs, not by the analyzer's references.** This is the central correction from verification. The analyzer's `ReferenceData` is only `{ range, is_definition }` (`ra_ap_ide-0.0.352/src/static_index.rs:37`); it carries no category. Measured on a fixture: of three non-definition references to a function, one was the call, one was `use inner::double;` and one was the function named as a value. Walking the analyzer's references would therefore invent calls. codegraph's own extraction already knows a site is a `Call`, a `TypeRef` or a `Reference`, so codegraph decides the edge type and the analyzer only answers "which item does this name resolve to".

This also disposes of a second measured problem. `def_to_kind` (`ra_ap_ide-0.0.352/src/moniker.rs:196`) splits associated functions into `Function`, `Method`, `StaticMethod` and `TraitMethod`, and `S::new()` is a `StaticMethod`. Because the edge type comes from codegraph's ref kind, the target's kind never has to be classified.

### Considered and rejected

- **Walk the analyzer's references and derive edge types from target kinds.** Rejected on the measurement above: it turns imports and value mentions into calls, and its kind table would have to enumerate 22 `SymbolInformationKind` variants correctly.
- **Run `rust-analyzer scip` as a subprocess and parse SCIP.** Measured exact: 3 s on the fixture, 75 s on `alt-canton`. Rejected because it needs the *indexed* workspace to ship the `rust-analyzer` component in its own toolchain pin, and the `scip` subcommand is explicitly unstable.
- **Replace tree-sitter with the analyzer as the Rust extractor.** The analyzer does expose whole-item spans, `TokenStaticData.definition_body`, so the earlier "no item spans" reason was wrong. Rejected instead because extraction must work with no analyzer at all: it feeds the per-crate fallback, Daml, `File` nodes and sync's content hashes.
- **Query a long-lived rust-analyzer over LSP during resolution.** Rejected on cost: one request per symbol, about 2,500 on `alt-canton`, against one batch pass.
- **Split the callee name on `.` and keep name matching.** Rejected: `a.submit` becomes `submit`, still ambiguous across three types, and `v.len()` would bind to any workspace method named `len`.

## 4. Design

### 4.1 Extraction records where the name is

`UnresolvedRef` gains `name_line` and `name_col`, the position of the **callee name token**. Today it stores only the call expression's start (`rust.rs:181`), which is a different place: in

```rust
client
    .submit(x)
```

the expression starts on the first line and the name is on the second. Recording the name position removes every proximity rule, and with it two measured failure modes: a call split across lines, and two calls to the same name on one line, as in `c.submit(1, 2) + C::submit(c, 3, 4)`.

The Daml extractor sets both fields to the values it already records. `UnresolvedRef.qid` is unchanged, so sync behaviour and qid stability are unaffected.

### 4.2 `src/rust_types/` — the only module that sees `ra_ap_*`

- `discover.rs` finds the Cargo workspaces under a configured root. Measured: `load_workspace_at` resolves the **nearest parent** `Cargo.toml` (`ra_ap_load-cargo-0.0.352/src/lib.rs:62`), so one call per root misses others. On `alt-canton` it loaded the root workspace and none of `solana-programs/ordering`.
  - **The rule is two passes, so it cannot depend on walk order.** Pass 1 collects every `Cargo.toml` under the root, using an `ignore::WalkBuilder` with standard filters, because the extraction walker accepts only `.rs` and `.daml` files and cannot be reused. Pass 2 loads every manifest declaring `[workspace]`, expands each one's members, including globs, then loads each remaining manifest that no loaded workspace claims. Manifest paths are canonicalised and de-duplicated, so a package is never loaded twice.
  - A manifest that fails to load is recorded with its reason and skipped.
- `analyze.rs` calls `StaticIndex::compute` once per workspace and returns plain data: per file, the tokens with their ranges, and per token, the definition's file and range. No `ra_ap` type crosses the module boundary.

### 4.3 The analyzer stage writes targets onto refs

The stage runs inside `index`, and inside `sync` when any `.rs` file changed. For each Rust `UnresolvedRef`:

1. Find the analyzer token at the ref's `(name_line, name_col)`, converted to a byte offset. No lookup API exists, but none is needed: `StaticIndexedFile.tokens` is a `Vec<(TextRange, TokenId)>` whose ranges are the name token exactly, and both tools index by UTF-8 byte offset, so the stage builds a per-file `HashMap` keyed on `range.start()` and matches exactly.
   - **Skip the whole-file module token.** Its range spans the file and starts at offset 0. rust-analyzer's own tests skip it (`static_index.rs:385`).
   - **One range can carry two token ids.** `get_definitions` returns an `ArrayVec<_, 2>` (`static_index.rs:150`); field-init shorthand, as in `Point { x, y }`, makes `x` both a field and a local. The stage stores both, writes a target only when they agree on one definition, and counts the rest as unmapped.
2. Take that token's `definition`, a `FileRange` naming a file and a range.
3. Find the codegraph symbol in that file whose span contains the definition's range, choosing the innermost when several do. The new `(:Symbol)` index over `root_name, relative_path` makes this a lookup rather than a scan. Match by containment, never by span equality: `definition_body` keeps leading `///` doc comments (`static_index.rs:351`), while a tree-sitter item node starts after them.
4. Write `target_qid` and `target_source = "type_exact"` on the ref.

A ref the stage cannot map keeps no target, and its reason is counted.

**Three coverage limits, all measured in the sources.** `StaticIndex::compute` skips library source roots (`static_index.rs:274`), so a call to workspace code *from* `std` or a dependency never appears, which is what codegraph wants. It also skips a file belonging to no crate (`static_index.rs:176`), and a `cfg`-disabled module never reaches it at all. Refs in those files keep no target and fall back to name matching, counted as unmapped rather than silently dropped.

The stage passes **`VendoredLibrariesConfig::Excluded`**. `Included { workspace_root }` would also index path dependencies inside the tree, which §4.2's discovery already loads as workspaces in their own right; indexing them twice would produce two definitions for one symbol.

**Why the target is stored, not turned straight into an edge.** `resolve()` drops every confidence-bearing edge and rebuilds (`src/resolution/drop.rs:12`), and `sync` always calls it (`src/sync/mod.rs:176`). If Phase R only ran when Rust files changed, a Daml-only sync would delete every exact Rust edge and never rebuild it. Storing the target on the ref follows the v1 principle that an `UnresolvedRef` is the durable record of source-code intent, so any later `resolve()` rebuilds the same edges with no analyzer run.

### 4.4 Phase R turns stored targets into edges

Phase R runs **first**, before Phase A. For each ref carrying `target_source = "type_exact"` it writes one edge from `source_qid` to `target_qid`, typed by the **ref's own kind**, with `confidence: "type_exact"`:

| ref kind | edge |
|---|---|
| `Call` | `CALLS` |
| `TypeRef` | `TYPE_OF` |
| `Reference` | `REFERENCES` |
| `Import` | none; imports stay with Phase A |

Phase R never guesses. Every part of the chain is an identity: the ref's own name position, the analyzer's own resolution, and a span containment check.

**Later phases must skip these refs.** Phase B takes no such set today; it returns one (`src/resolution/phase_b_import_driven.rs:19`). Phase B gains a `resolved_earlier: &HashSet<String>` parameter, which Phase R fills and Phase C already honours. Without it, Phase B would overwrite `type_exact` with `import_resolved` on the same edge. Phase D needs no change: it selects refs whose `unresolved_reason` is `no_candidate`, which a Phase R ref never receives.

### 4.5 Extraction: named struct fields

A `field_declaration` pattern in `rust.scm` is not enough. The capture loop builds `<module>::<name>` and consults `enclosing_impl_type` only for functions (`rust.rs:76`), so a field would become `src::lib::tolerated`, and two structs with a same-named field in one file would collide into `…::tolerated` and `…::tolerated#1`, distinguished by iteration order. The slice therefore adds an `enclosing_struct_type` helper mirroring `enclosing_impl_type`, giving `<module>::<Type>::<field>`.

Tuple-struct fields have no name and stay unextracted. No test can observe them, so §6 records the accept.

### 4.6 Phase C guard

Phase C must not offer `Field` nodes as candidates for a ref of kind `call`: a field named `create` would otherwise become a candidate for an unrelated call. The guard is a label filter in `name_match_candidates` (`src/db/queries.rs`), whose only caller already carries the ref kind. This changes name matching for **every language**, so slice 4 carries a Daml regression test.

### 4.7 Failure handling

- **A workspace that fails to load** produces no targets. Its refs fall through to Phases A–D with today's confidence values, and `index` completes with a warning naming the manifest and the reason.
- **A panic inside `ra_ap_*`** must not kill the run. Each workspace is analysed on its own thread and a panic is caught and treated as a load failure. A test injects a panic to prove the catch.
- **A missing sysroot** drops references with no error, so it needs a detector, not a hope: the stage fails loudly when sysroot discovery returns nothing, and the fixture asserts the `Arc` call resolves, which is the reference that disappears first.
- **A ref that maps to no token, or a definition that maps to no symbol,** is counted in `ResolutionStats` rather than dropped silently.

### 4.8 Configuration, sync, and what consumers see

```toml
[rust]
type_aware = true    # slices 3 and 4 ship it false; slice 5 flips the default
```

`false` skips the analyzer stage and Phase R entirely. The section and key both default, so existing config files stay valid and `schema_version` does not change.

`sync` runs the analyzer stage when any `.rs` file changed, which is what the pilot chose on 2026-09-18, accepting the cost. A Daml-only sync skips it and still rebuilds every `type_exact` edge from the stored targets.

Consumers see a new `confidence` value, `"type_exact"`, on `CALLS`, `TYPE_OF` and `REFERENCES` edges, and more rows from `callers`, `callees` and `impact`. Their shapes do not change. `ResolutionStats` gains three counters: refs given a target, refs whose name position matched no token, and definitions that matched no symbol. `StatusSnapshot` (`src/graph/status.rs:17`) carries no resolution stats today; slice 5 adds the three counters to it, so `codegraph_rs_status` gains three fields additively.

## 5. Slices

Each slice ships on its own and is verified before the next begins.

**Slice 1 — toolchain and pins.** Raise `rust-toolchain.toml` to 1.98.1, add the `ra_ap_*` and `unicode-ident` pins from §2. Exit: `cargo build`, `cargo clippy --all-targets` and the existing suite pass, with no behaviour change.

**Slice 2 — `src/rust_types/`, no consumer.** Workspace discovery and `StaticIndex::compute` behind a plain API. Exit: the discovery test below passes, and a release-build run over `alt-canton` is measured and recorded in this spec as the budget. If that number exceeds 120 s, the slice stops and the pilot decides before slice 3, because §4.8 makes every Rust sync pay it.

**Slice 3 — name positions, the analyzer stage, Phase R for calls.** `type_aware` defaults to `false`. Exit: the fixture's call assertions below pass with the flag on, and the existing suite is unchanged with it off.

**Slice 4 — struct fields, `REFERENCES` and `TYPE_OF`, Phase C guard.** Exit: the field assertions and the Daml regression test pass.

**Slice 5 — sync, stats, status, default on.** Exit: the sync tests pass and `type_aware` defaults to `true`.

## 6. Testing

Every test is written first and watched failing.

**Unit, no external services.**
- Position conversion, both directions, including the one-based-to-zero-based line shift and a multi-byte character in a line.
- The token lookup: a name position that matches a token start exactly; a position inside a token but not at its start, which must not match; the whole-file module token, which must be skipped; a range carrying two token ids, which must be counted as unmapped unless both agree.
- Definition-to-symbol containment: one candidate; nested candidates, where the innermost wins; no candidate, which must be counted, not panic.
- Workspace discovery over a fixture tree holding a root workspace, a member package and a second independent workspace in a subdirectory. Two manifests load, each once, in either walk order. This fixture is on disk, so the test is a fixture test rather than a pure unit test.

**Integration, gated by `CODEGRAPH_RS_TEST_NEO4J`.** A fixture workspace under `tests/fixtures/rust_ws/` repeating the measured forms: a direct call, an alias, a call through `Arc`, `A::submit(a, …)`, a call split across two lines, two calls to the same name on one line, `S::new()` as a static method, a trait-method call, a struct field read, and a `use` import of a called function.

Assertions are counts tied to forms, not existence:
- `A::submit` has exactly 5 incoming `CALLS` edges, all `type_exact`, one per form, and `B::submit` exactly 1.
- No `CALLS` edge runs from the calling function to `B::submit` beyond that one. Today's name matching would add one, so this is the test that proves the fix.
- `S::new()` and the trait-method call each produce a `CALLS` edge, which the rejected kind-routing rule would have missed.
- The `use` import produces **no** `CALLS` edge, which the rejected rule would have produced.
- The fixture's struct field has exactly 2 incoming `REFERENCES` edges, both `type_exact`: one read and one write.
- Those refs carry `target_source = "type_exact"` and no `unresolved_reason`.

**Slice-specific tests.**
- **Fallback:** a fixture whose `Cargo.toml` is broken. The stage reports it, `index` completes, name-match edges still appear.
- **Panic:** an injected panic in the analysis thread is caught and treated as a load failure.
- **Sysroot:** with discovery forced to return nothing, the stage fails loudly rather than producing 4 of 5 references.
- **Config:** a config with no `[rust]` section loads; `type_aware = false` produces no `type_exact` edge and leaves today's edges unchanged.
- **Phase C guard (slice 4):** with `type_aware = false`, a call whose name matches a `Field` gets no edge. With Phase R on, the guard is unobservable, which is why this test sets the flag off.
- **Daml regression (slice 4):** a Daml fixture's resolution counts are identical before and after field extraction and the guard.
- **Sync (slice 5):** a Daml-only sync leaves every `type_exact` edge in place. A Rust sync reruns the stage and updates a changed call's target.

**Measurement, not a pass condition.** After slice 3 and again after slice 4, rerun the comparison probes and record counts for `A::submit`, `SequencerHandle::submit`, `run::subscribe`, `create` and `Settled::tolerated`.

## 7. Open questions and accepted gaps

- **Accepted, pilot, 2026-09-18: derive-generated calls stay unresolved.** rust-analyzer reports none of the 22 sites `#[derive(Payload)]` generates for `PayloadField::to_leaf`. A fix needs macro expansion and is separate work.
- **Accepted, pilot, 2026-09-18: `ra_ap_*` upgrades are manual work**, with no API stability guarantee.
- **Accepted, pilot, 2026-09-18: every sync touching Rust pays the analysis cost**, subject to the slice 2 gate.
- **Accepted, pilot, 2026-09-18: tuple-struct fields stay unextracted.** They have no name to match, and no test can observe them.
- **Open, owner: this session, gate: slice 3.** A SCIP run reported 48 references to `SequencerHandle::submit` where an LSP `references` run reported 39. Until the difference is explained, the fixture's exact counts stand as the contract and `alt-canton` numbers are reported as measurements only.
- **Accepted, pilot, 2026-09-18: `cfg`-disabled code resolves by name match only.** rust-analyzer never analyses a module the active `cfg` flags exclude, so those refs keep no target. They are counted, not dropped.
- **Open, owner: this session, gate: slice 2.** `StaticIndex::compute` computes hover for every definition unconditionally (`static_index.rs:205`), with no configuration knob. Slice 2's measurement decides whether that is affordable; if it is not, the alternative is the rejected subprocess approach, which pays the same cost in another process.

## 8. What verification changed

Draft 1 was reviewed by three fresh-context agents against the code. Eleven findings were load-bearing. The largest:

- Phase R was reversed to be ref-driven, because the analyzer's references include imports and value mentions.
- Targets are stored on the ref, because `resolve()` drops all edges and a Daml-only sync would have deleted the exact Rust edges.
- Name positions are recorded at extraction, replacing a proximity rule that failed on the split-line call the tests require.
- `enclosing_symbol_qid` was dropped from the design: it works only on in-memory extraction output, and the ref's `source_qid` already holds the caller.
- Phase B gained an explicit skip set; it consumes none today.
- Field extraction gained an `enclosing_struct_type` helper, without which qualified names collide.
- The `unicode-ident` pin became a range, the crate count became a ceiling, and the time budget moved to a slice 2 gate.
