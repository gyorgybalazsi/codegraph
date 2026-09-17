# codegraph-rs — type-aware Rust resolution (Phase R)

**Goal:** resolve a Rust method call to the method the compiler would call, so `callers`, `callees` and `impact` answer correctly for Rust.
**Status:** proposed
**Baseline:** the v1 design (`2026-04-22-codegraph-rs-design.md`) §7, whose "Not in v1" list excludes type-aware dispatch. This design supersedes that exclusion for Rust.

---

## 1. Why

A comparison against rust-analyzer on 2026-09-17 measured codegraph-rs on two workspaces: a 32-line fixture and `alt-canton`. Every Rust method-call test returned 0 correct results.

| Question | Ground truth | codegraph-rs | Cause |
|---|---|---|---|
| calls to `A::submit` | 5 | 0 | see below |
| calls to `B::submit` | 1 | 0 | see below |
| calls to `SequencerHandle::submit` | 39 | 0 | see below |
| calls to `run::subscribe` | 8 | 0 | see below |
| references to `Settled::tolerated` | 7 | not expressible | codegraph extracts no Rust struct fields |

Two causes, both measured:

- **The receiver stays in the name.** `src/extraction/languages/rust.rs` splits a callee's text on `::` only, so `a.submit(1, 2)` is recorded under the name `a.submit`. No symbol has that name, so the ref ends as `no_candidate`. In `alt-canton`, 57 refs named `*.submit` and 91 named `*.subscribe` ended that way.
- **A bare name matches every type.** `A::submit(a, 9, 10)` is recorded as `submit`, which matches `A::submit`, `B::submit` and `C::submit`. Phase C records `ambiguous` and writes no edge. codegraph holds no type information, so it cannot choose.

Import-driven resolution does not rescue this. Rust qualified names come from file paths, such as `core_lib::src::lib::A`, and Rust has no `Module` node per file, so Phase A matches nothing. The fixture index reported `imports=0`.

## 2. Constraints

- rustc: the repo's `rust-toolchain.toml` must pin **1.98.1 or later**. `ra_ap_*` 0.0.352 does not build on the current pin, 1.91. The bump lands in its own PR before this work.
- Analyzer crates: **`ra_ap_ide`, `ra_ap_ide_db`, `ra_ap_load-cargo`, `ra_ap_project_model`, `ra_ap_vfs`, `ra_ap_paths`, all at `=0.0.352`**, pinned exactly. They carry no API stability guarantee.
- `unicode-ident` must be pinned to **`=1.0.22`**. `unicode-properties` 0.1.4 ships Unicode 17, `unicode-ident` 1.0.25 and later ship Unicode 18, and `ra-ap-rustc_lexer` 0.166 panics at compile time when the two disagree.
- Dependency count: this adds **223 crates** to the lockfile.
- Sysroot: the loader must set `sysroot: Some(RustLibSource::Discover)`. Measured: without it, `find_all_refs` returned 4 of 5 references and dropped the call through `Arc<A>` with no error.
- Index time budget: **90 s** for `alt-canton` (180 files, 3,895 nodes). Measured 75 s for a full `rust-analyzer scip` run on the same tree.
- Neo4j schema: unchanged. No new node label and no new property.

## 3. Approach

Embed rust-analyzer as a library and compute one whole-workspace index per Cargo workspace, then convert its definitions and references into codegraph edges in a new resolution phase.

`ra_ap_ide::StaticIndex::compute` returns every token in the workspace, each with its definition, its references and its documentation. It is the same entry point rust-analyzer's own `scip` command uses.

### Considered and rejected

- **Run `rust-analyzer scip` as a subprocess and parse the SCIP file.** Measured exact on both workspaces: 3 s on the fixture, 75 s on `alt-canton`. Rejected because it depends on the *indexed* workspace shipping the `rust-analyzer` component in its own toolchain pin. The fixture pins no toolchain and therefore has no binary. The `scip` subcommand is also explicitly unstable.
- **Replace tree-sitter with SCIP or `ra_ap_*` as the Rust extractor.** Rejected because the analyzer reports name ranges, not item bodies. codegraph needs item spans for source snippets and for "which symbol encloses this position". The per-crate fallback also needs tree-sitter, so this means maintaining two extractors.
- **Query a long-lived rust-analyzer over LSP during resolution.** Rejected on cost: one request per symbol, about 2,500 on `alt-canton`, against one batch pass.
- **Split the callee name on `.` and keep name matching.** Rejected: `a.submit` then becomes `submit`, which stays ambiguous across three types, and `v.len()` would bind to any workspace method named `len`.

## 4. Design

### 4.1 `src/rust_types/` — the only module that sees `ra_ap_*`

- `load.rs` discovers every Cargo workspace under a configured root, and loads each one with `load_workspace_at`, setting `sysroot: Some(RustLibSource::Discover)`.
  - **Discovery rule:** walk the root for `Cargo.toml` files, honouring `.gitignore` as the extraction walker does. Load a manifest when it declares `[workspace]`, or when it is a package that no already-loaded workspace lists as a member. Skip every other manifest, because its packages arrive with their workspace.
  - **One load per root is not enough.** Measured on 2026-09-17: `alt-canton` holds a root workspace plus a second, independent one at `solana-programs/ordering`. A single whole-tree analysis covered `crates`, `example-contracts` and `adversarial-guests`, and none of `solana-programs`.
- `index.rs` calls `StaticIndex::compute` once per Cargo workspace and converts the result into plain data:
  - a **definition**: relative path, name position, name, and a token id;
  - a **reference**: relative path, position, and the token id it resolves to.
- No `ra_ap` type crosses this module's boundary, so the rest of codegraph does not depend on the analyzer's API.

### 4.2 Phase R — a new resolution stage

Phase R runs **before Phase B**. Phases B, C and D skip every ref it resolved, exactly as they already skip refs resolved by an earlier phase.

1. Map each definition to a codegraph symbol: the symbol whose span contains the definition's name position, and whose `name` equals the token's name.
2. Map each reference to its calling symbol the same way, reusing `enclosing_symbol_qid`.
3. Write the edge that fits the target kind, always with `confidence: "type_exact"`:
   - `Function`, `Method` → `CALLS`
   - `Field`, `Struct`, `Enum`, `Trait`, `TypeAlias` → `REFERENCES`
4. Clear the matching `UnresolvedRef`, so Phase C cannot add a weaker second edge to a different target. A line can hold two calls to the same name, as `c.submit(1, 2) + C::submit(c, 3, 4)` does, and an `UnresolvedRef` stores the position where the call *expression* starts while the analyzer reports the position of the *name* token. The match is therefore the ref on that line whose start position is the closest one at or before the name position, among refs whose `unresolved_name` ends with that name.

`resolve()` already threads a set of resolved ref qids from Phase B into Phase C. Phase R fills that same set before Phase B runs, and Phase D sees the cleared `unresolved_reason` in the database.

Phase R never guesses. A reference it cannot map is left to the existing phases.

### 4.3 Extraction: named struct fields

`rust.scm` gains a `field_declaration` pattern inside `struct_item`. Each named field becomes a `Field` node with `qualified_name` `<module>::<Type>::<field>` and a `CONTAINS` edge from the file, mirroring how `Method` nodes already work. Tuple-struct fields have no name and stay unextracted.

This is what makes `Settled::tolerated` answerable: the field becomes a symbol, and Phase R attaches its 7 references.

### 4.4 Phase C guard

Phase C must stop offering `Field` nodes as candidates for a ref of kind `call`. Without this guard, a new field named `create` becomes a candidate for an unrelated call, in Rust or in Daml, and name matching writes a wrong edge. The guard is a label filter in `name_match_candidates`.

### 4.5 Failure handling

- **A Cargo workspace that fails to load** produces no index. Phase R logs the path and the reason; every ref inside it falls through to Phases B and C with today's confidence values. `index` completes.
- **A panic inside `ra_ap_*`** must not kill the run. Each workspace's analysis runs on its own thread, and Phase R catches a panic and treats it as a load failure.
- **A reference that maps to no codegraph symbol** is counted, not dropped silently.

### 4.6 Configuration

```toml
[rust]
type_aware = true    # default
```

`false` skips Phase R and restores today's index time. The section and the key both default, so existing config files stay valid and `schema_version` does not change.

### 4.7 Sync

`sync` reruns Phase R whenever any `.rs` file changed. A sync touching only Daml files skips it. A one-line Rust edit therefore pays the full analysis cost, up to the 90 s budget in §2. The pilot accepted this cost on 2026-09-18, preferring correctness after every sync.

### 4.8 What consumers see

- A new `confidence` value, `"type_exact"`, on `CALLS` and `REFERENCES` edges.
- `ResolutionStats` gains two counters: refs resolved by Phase R, and reference positions that matched no symbol. `codegraph_rs_status` reports stats, so its response gains two fields. The change is additive.
- `callers`, `callees` and `impact` return more rows. Their shapes do not change.

## 5. Testing

Every test below is written first and watched failing before the code lands.

**Unit, no external services.** The definition-to-symbol mapping, the reference-to-caller mapping and the ref-matching rule of §4.2 are pure functions over positions and spans. Cases: a position inside one symbol; a position inside a nested symbol, where the innermost wins; a position inside no symbol, which must be counted, not panic; two calls to the same name on one line, where each name position must pick its own ref; a workspace discovery walk over a tree holding a root workspace and a second workspace in a subdirectory, which must return both.

**Integration, gated by `CODEGRAPH_RS_TEST_NEO4J`.** A new fixture workspace under `tests/fixtures/rust_ws/`: two crates holding a direct call, an alias, a call through `Arc`, the `A::submit(a, …)` form, a call split across two lines, a same-crate type with the same method name, and a struct field read.

- `A::submit` has a `CALLS` edge from `calls` with `confidence: "type_exact"`; `B::submit` and `C::submit` each have theirs.
- No `CALLS` edge runs from `calls` to `B::submit` other than the real one. Today's name matching would get this wrong, so this is the test that proves the fix.
- The struct field has a `REFERENCES` edge with `confidence: "type_exact"`.
- The `UnresolvedRef`s at those call sites carry no `unresolved_reason`.

The integration test needs no rust-analyzer binary, because the analyzer is compiled into codegraph.

**Two-workspace test.** The fixture tree gains a second, independent Cargo workspace in a subdirectory. Symbols in it get `type_exact` edges too, which fails before the discovery rule exists.

**Fallback test.** A fixture whose `Cargo.toml` is broken. Phase R reports the load failure, `index` completes, and name-match edges still appear.

**Measurement, not a pass condition.** After the change, rerun the comparison probes and report counts for `A::submit`, `SequencerHandle::submit`, `run::subscribe`, `create` and `Settled::tolerated`.

## 6. Open questions and accepted gaps

- **Accepted, pilot, 2026-09-18: derive-generated calls stay unresolved.** rust-analyzer reports none of the 22 call sites that `#[derive(Payload)]` generates for `PayloadField::to_leaf`, so embedding it does not fix them. Any fix needs macro expansion and is separate work.
- **Accepted, pilot, 2026-09-18: `ra_ap_*` upgrades are manual work.** The API has no stability guarantee, and today's build already needs the `unicode-ident` pin in §2.
- **Open: a reference-count discrepancy.** A SCIP run reported 48 references to `SequencerHandle::submit`; an LSP `references` run by another session reported 39. The cause is unknown. Resolve it before trusting Phase R's counts as exact, because the same analyzer produced both numbers.
- **Open: release-build query cost is unmeasured.** The spike ran a debug build, where one `find_all_refs` took 28 s after a cold sysroot load. The 90 s budget in §2 comes from the `scip` CLI, which is a release build. Measure a release `StaticIndex::compute` on `alt-canton` in the first implementation slice, and revisit the budget if it misses.

## 7. Out of scope

- Daml. Its extraction and resolution are untouched.
- Rust doc comments, which is a separate bounded change: the extractor sets `doc: None` today, and 0 of 1,981 `alt-canton` functions carry a doc. `StaticIndex` exposes documentation per token, so that change may later read docs from Phase R instead of tree-sitter.
- The MCP server's log channel, a separate bounded change: `serve` writes tracing output to stdout, which is the MCP protocol channel.
