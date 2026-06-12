# codegraph-rs — external-package references design

**Status:** Approved (scope), awaiting implementation plan.
**Baseline:** `main` @ `fa88834` (after PR #15 — Phase A/B import resolution fixed; `IMPORTS`/Phase-B cross-module calls now work within the indexed workspace).
**Scope:** A general, package-aware layer so references into *external* dependency packages resolve. External symbols exist purely as resolution targets — excluded from embeddings and search.

This spec extends the v1 design (`2026-04-22-codegraph-rs-design.md`, §7 Resolution) and the v2 Daml-depth design (`2026-05-15-…`). Section refs to "v1 spec §N" point at the former.

---

## 1. Goal and scope

### 1.1 Goal

After #15, intra-workspace resolution works: `import Splice.X` binds to in-workspace Modules, and Phase B resolves cross-module calls precisely (`confidence: import_resolved`). What remains unresolved is **references into packages we don't index**. On the Splice corpus this is a single, well-defined population:

- **6,669** `call` refs with `unresolved_reason = "no_candidate"` — names like `pure`, `map`, `show`, `submit`, `Some`, `Party`, `ContractId`.
- **20** distinct unresolved `import` module paths, *all* Daml SDK: `DA.*` (stdlib), `Prelude`, `Daml.Script`.
- **0** unresolved `Splice.*` imports — sibling Splice packages already resolve (they live under the indexed `daml` root).

The goal is a **general** mechanism: today it closes the SDK-stdlib gap; tomorrow it handles any third-party DAR dependency, with no per-dependency special-casing.

### 1.2 Design decisions (locked)

| Axis | Decision |
|---|---|
| **Model** | A first-class **`Package`** layer with package-qualified resolution (not workspace-wide name match into externals). |
| **Generality** | A **general** external-package mechanism driven by compiled artifacts, not a stdlib-only shortcut. |
| **Population source** | **DAR/DALF metadata** — decode each dependency's compiled Daml-LF to enumerate its exported symbol *names* per module. Source-independent; works for SDK stdlib and third-party DARs uniformly. |
| **Dependency mapping** | Derived from each local package's `daml.yaml` (`dependencies` / `data-dependencies`) plus DAR `MANIFEST.MF` dependency lists. |
| **Visibility** | External symbols are **resolution targets only** — marked `external`, excluded from embeddings, `semantic_search`, and keyword `search`. `CALLS`/`TYPE_OF`/`REFERENCES` edges to them still resolve. |

### 1.3 Out of scope

- **External source / bodies.** We ingest exported *names + kinds + module/package*, not definitions or `source` snippets. `graph/node` on an external symbol returns metadata, no snippet.
- **Semantic search / navigation into external code.** Externals never get embeddings and are filtered out of all search read-paths (§5).
- **Rust external dependencies (crates).** This layer is Daml-LF-specific. A Rust analogue (resolving into `cargo` deps) is a future slice.
- **Cross-workspace / multi-repo indexing.** Unchanged from v1.
- **Re-deriving external symbols from source.** Even where source is available locally (the SDK ships stdlib `.daml`), we standardize on DAR/DALF metadata so the path is uniform and version-faithful to what the code actually compiled against.

---

## 2. Data model additions

### 2.1 `Package` node

| Property | Meaning |
|---|---|
| `package_id` | Daml-LF package hash (content-addressed; stable across machines/runs). Primary key. |
| `name`, `version` | From the DAR manifest / `daml.yaml`. |
| `external` | `true` for dependency packages; `false` for the workspace's own packages. |
| `qid` | `pkg:<package_id>` — a new qid namespace, disjoint from local `<root>:<rel_path>:<qname>`. |

Both local and external packages get `Package` nodes so the dependency graph is uniform, **but their identity differs**:

- **External** packages are keyed by the content-addressed LF `package_id` (`qid = pkg:<package_id>`).
- **Local** packages are *not* decoded from a DAR (we extract them from source and have no compiled `package_id`), so they use a synthetic key derived from their `daml.yaml` `name` (+ `version`): `qid = pkg:local:<name>`, `external = false`. Each `File` maps to its local package by **nearest-ancestor `daml.yaml`** (walk up from the file's directory). A local package's `DEPENDS_ON` set is built from its `daml.yaml` `dependencies` / `data-dependencies` (resolved to the `package_id`s learned during ingestion) plus the implicit SDK packages (§3.1).

### 2.2 External `Module` / symbol nodes

External modules and their exported symbols are ingested as ordinary `:Module` / `:Symbol`-labelled nodes **plus** an `:External` marker label and `external: true` property:

- qid: `pkg:<package_id>:<module_qname>[.<symbol_qname>]`.
- Symbol kinds map from LF: data types → `Struct`, templates → `Template`, choices → `Choice`, interfaces → `Interface`, values/functions → `Function`. (Names + kinds only; no fields, no bodies, no spans.)
- They keep the `:Symbol` label **on purpose** — resolution candidate lookups match `(:Symbol {name})` (v1 spec §7), so externals must be findable there. Search/embed exclusion is done at the *read* layer (§5), not by withholding the label.

### 2.3 New edges

| Edge | Direction | Meaning |
|---|---|---|
| `IN_PACKAGE` | `Module → Package` | Which package a module belongs to (local and external). |
| `DEPENDS_ON` | `Package → Package` | Local package → each external package it may import from. Scopes resolution (§4). |

### 2.4 Schema change — additive constraint, **no `schema_version` bump**

`apply_schema` (`src/db/schema.rs`) gains a uniqueness constraint on `Package(qid)` (alongside the existing `symbol_qid_unique` and `unresolved_ref_qid_unique`). Like the others it is idempotent (`CREATE CONSTRAINT … IF NOT EXISTS`), so existing databases simply gain it on the next `index`.

**Do NOT bump `schema_version`.** `SUPPORTED_SCHEMA_VERSION` (`src/config/workspace.rs:80`) is checked with strict equality at config load — `cfg.schema_version != SUPPORTED_SCHEMA_VERSION` ⇒ `bail!` (`workspace.rs:95`). Bumping 1→2 would make *every existing* `.codegraph/config.toml` (all written with `schema_version = 1`, e.g. `~/splice`) fail to load until regenerated. The `schema_version` field guards the **config-file format**, which this change does not alter — adding a Neo4j constraint is orthogonal. If a future change genuinely alters the config format, that bump must ship with a migration (auto-rewrite the field, or relax the gate to `<=`).

The FTS (`symbol_fts`) and vector (`symbol_embedding`) indexes are unchanged — see §5 for why external `:Symbol`s sitting in those indexes is acceptable.

---

## 3. Ingestion — new `extern` stage

A new pipeline stage runs **before resolution** (extract → write → **extern** → resolve → embed) and is idempotent (MERGE by content-addressed `package_id`; identical inputs ⇒ no-ops). It is gated/skippable like embeddings.

### 3.1 Enumerate dependency DARs

For each local `daml.yaml`:
- `sdk-version` → the SDK package-db at `~/.daml/sdk/<ver>/damlc/resources/pkg-db_dir/<lf>/…` supplies `daml-prim`, `daml-stdlib`, and (for scripts) `daml-script` DALFs. **These SDK packages are *implicit* dependencies of every local package** — they are not listed in `daml.yaml dependencies` but are imported (`Prelude`, `DA.*`, `Daml.Script`). The `extern` stage adds them to every local package's `DEPENDS_ON`.
- `dependencies` / `data-dependencies` → built DARs (e.g. Splice's `daml/dars/*.dar`).

**Pin one LF version per package.** The SDK `pkg-db_dir` ships `daml-stdlib`/`daml-prim` compiled for *multiple* LF versions (`2.1`, `2.dev`), which are *distinct `package_id`s exposing the same module names* (e.g. `DA.List`). Ingesting more than one would create duplicate module qualified-names and break module lookup (§4.1). Resolve the **single** `<lf>` matching the workspace's target (from `daml.yaml` / `build-options`) and ingest only that. The DAR set to ingest is the transitive closure of the above, deduped by `package_id`.

### 3.2 Decode DAR/DALF — reuse the canton-toolbox decoder

A DAR is a zip: `META-INF/MANIFEST.MF` lists the main DALF + dependency DALFs; each `.dalf` is a Daml-LF protobuf `Archive` (envelope → `ArchivePayload` → LF2 `Package`). Per package we extract: `package_id`, dependency `package_id`s, and per module its exported definitions (name + kind).

**This is already solved in the sibling `canton-toolbox` repo — we reuse it rather than build from scratch.** Its `codegen` crate has a working prost-based LF reader:

- Vendored schema: `codegen/resources/protobuf/com/digitalasset/daml/lf/archive/daml_lf.proto` + `daml_lf2.proto`, compiled by `codegen/build.rs` via `prost-build` into `lf_protobuf.rs` (`com::daml::daml_lf_2`, `…daml_lf_dev`).
- `archive.rs`: `archive_from_dar`, `extract_dalfs_from_dar` (zip + `Archive::decode`).
- `package.rs`: `parse_dar(path) -> ParsedDar { main_package_id, packages: HashMap<package_id, Package> }` and `parse_dars(&[..])` (dedupes by hash). **`package_id` is `archive.hash`** — the content-addressed key §2.1 specifies, for free. Targets **DamlLf2** (LF 2.x), which matches Canton 3.x / the local SDK (`pkg-db_dir/2.1`, `2.dev`).

So the ingestion work is: (1) bring this decoder into codegraph-rs, and (2) walk the decoded LF `Package` for exported definition names + kinds (data types → `Struct`, templates → `Template`, choices → `Choice`, interfaces → `Interface`, values → `Function`) and inter-package dependency hashes.

**LF names are interned — not direct strings.** In LF2, module names and definition names are stored as *indices* into the package's interned-string / interned-dotted-name tables, not as inline strings. Extracting a readable `qualified_name` means dereferencing those indices against the package tables — exactly what canton-toolbox's `codegen/src/resolve_type.rs` already does (`*_interned_str` / `*_interned_dname` → `&[String]`). The 11b walk must reuse that dereferencing, not read `.name` fields directly (which yield integers). This is the main correctness subtlety of the AST walk.

**Reuse strategy: vendor (decided).** Copy the minimal decoder into codegraph-rs — the two `.proto`s, the `build.rs` prost-build step, and `archive.rs` / `package.rs` / `lf_protobuf.rs` (plus the interned-name dereferencing helpers from `resolve_type.rs`) — rather than a cross-repo path dependency on canton-toolbox. Keeps codegraph-rs self-contained and local-first; avoids coupling its build to a sibling repo's layout. The 11b spike confirms the exact minimal file set and that LF2 covers every DALF we ingest. (Trade-off accepted: the vendored protos/decoder must be re-synced if upstream LF evolves — low frequency, and pinned to the LF version the workspace's SDK emits.)

### 3.3 Write external nodes + edges

The `extern` stage runs **after `write`** (so local `File`/`Module` nodes already exist) and:
1. Creates **local** `Package` nodes from each `daml.yaml`, links local `Module`s via `IN_PACKAGE`, and maps each `File` to its local package (nearest-ancestor `daml.yaml`, §2.1).
2. Ingests external DARs: MERGE external `Package`, external `Module` nodes, and external symbol stubs (`external: true`, `:External`). **Each external `Module` `CONTAINS` its exported symbols** — Phase B's `import_driven_candidates` matches `(m:Module)-[:CONTAINS|EXPORTS]->(:Symbol {name})`, so the containment edge is mandatory for resolution to find them.
3. Adds `IN_PACKAGE` (module→package) and `DEPENDS_ON` (local→external, incl. implicit SDK packages).

MERGE-by-`package_id` makes it idempotent; repeated `index` runs skip already-ingested external packages. The `extern` stage is **not** part of per-file `sync` — external deps change rarely; re-ingest is triggered by an explicit re-`index` (or a future `daml.yaml`/DAR-change check).

---

## 4. Package-qualified resolution

Resolution gains package scoping. Precedence is **local-import > external-import > local-name-match**; externals bind *only* through an import path, never via fallback.

### 4.1 Phase A (imports)

`import M` in a file resolves to a `Module {qualified_name: M}` reachable from the file's local package — first the workspace (local-first), then the package's `DEPENDS_ON` external packages. This both lights up the 20 stdlib imports and disambiguates same-named modules across packages.

`find_module_by_qualified_name` (`src/db/queries.rs`) currently **hard-`bail!`s on >1 match** — fine when module qnames were workspace-unique, but externals reintroduce duplicates (e.g. a workspace module and a dep both named, or the multi-LF stdlib problem §3.1). The package-scope filter must therefore (a) restrict candidates to the importing file's package + its `DEPENDS_ON` set, and (b) prefer the local match, returning the single in-scope module **instead of bailing**. (§3.1's single-LF pinning removes the most common duplicate at the source.)

### 4.2 Phase B (import-driven)

Already restricts candidates to imported modules (`import_driven_candidates`); externals participate automatically once their `IMPORTS` edges exist. Calls into `DA.*`/`Prelude` resolve here with `confidence: import_resolved`.

### 4.3 Phase C (name-match fallback) — exclude externals

`name_match_candidates` adds `WHERE c.external IS NULL`. Rationale: a bare name like `map` must *not* silently bind to `DA.List.map` workspace-wide — that would manufacture thousands of false edges. Externals are import-scoped only. Local-to-local name match is unchanged.

### 4.4 Phase A.5 (party / implements / requires)

Phase A.5 builds a workspace-wide `qualified_name → qid` map (`build_qname_map` = `MATCH (s:Symbol) …`), which will now also contain external symbols. Implications:
- **Signatory/Controller/Observer** use *scoped* keys (`<local-scope-qname>.<name>`) — these never collide with `pkg:…`-namespaced externals, so party edges are unaffected.
- **Implements/Requires** use `find_interface_by_name` (exact, then `.<suffix>` match) — an external `Interface` can now match, and that is **allowed (decided)**: a template implementing an interface defined in a dependency is a real relationship, so `IMPLEMENTS`/`REQUIRES` may point at external interfaces. `build_qname_map` is left unfiltered. (If suffix-matching ever proves too loose against the larger external namespace, tighten `find_interface_by_name` to prefer in-`DEPENDS_ON` packages — a later refinement, not v1.)

### 4.5 Idempotency

`drop_resolution_edges` already clears `CALLS`/`IMPORTS`/`REFERENCES`/`TYPE_OF` by `confidence`. External *nodes/packages* are not resolution-derived (they're ingested facts) and are left intact across re-resolves; only the edges into them are rebuilt.

---

## 5. Visibility — resolution-only

External symbols are excluded from every value-bearing read surface; they exist solely so refs can bind.

| Surface | Change |
|---|---|
| **Embeddings** | `EmbedScope` / `fetch_in_scope` (`src/vectors/query.rs`) and the `EMBED_KINDS_CYPHER_LIST` selection add `AND n.external IS NULL`. No external symbol is ever embedded. |
| **`semantic_search`** | `graph/semantic.rs` query filters `WHERE node.external IS NULL` (externals have no embedding anyway, but the filter is explicit). |
| **keyword `search`** | `graph/search.rs` FTS query post-filters `WHERE NOT n:External`. |
| **`status`** | Reports external packages/symbols as a separate line, not folded into workspace symbol counts. |

**FTS-index tradeoff (accepted).** External `:Symbol`s physically sit in the `symbol_fts` index (it's defined on `:Symbol`), but the `search` query excludes them. Rebuilding the FTS index to exclude externals by label is not worth it; the bloat is bounded and search results are correct.

---

## 6. Slices (each independently shippable)

| Slice | Scope | Tag |
|---|---|---|
| **11a — model + visibility** | `Package` node, `external` flag/`:External` label, qid namespace, schema v2 constraint, exclude-external filters in embed/search read-paths. No ingestion yet → no behavior change, but the surfaces are ready. | `slice-11a-package-model` |
| **11b — DAR/DALF ingestion** | Vendor the canton-toolbox LF decoder (§3.2); `extern` pipeline stage; enumerate deps from `daml.yaml` + SDK package-db; walk LF `Package` for exported names/kinds; write external `Package`/`Module`/symbol stubs + `IN_PACKAGE`/`DEPENDS_ON`. | `slice-11b-dar-ingestion` |
| **11c — package-qualified resolution** | Scope Phase A/B by `DEPENDS_ON`; exclude externals from Phase C; precedence rules. | `slice-11c-package-resolution` |
| **11d — corpus validation** | Re-index splice; confirm the ~6,669 `no_candidate` stdlib calls bind to external `DA.*`; confirm externals absent from embeddings/search. | (validation, no tag) |

Slice 11a is safe to ship alone (additive). 11b is the heaviest (the LF reader). 11c delivers the user-visible win.

---

## 7. Risks and open items

- **LF decoder (was the primary risk — now de-risked).** Reuses the proven prost-based reader from the sibling `canton-toolbox` `codegen` crate (vendored `daml_lf2.proto` + `prost-build` + `parse_dar`/`Package` walk; `package_id = archive.hash`; LF2). Residual risk is narrowed to the reuse strategy (vendor vs path-dep, §3.2) and walking the LF `Package` AST for exported names — not building a decoder. The 11b spike confirms the minimal vendored file set.
- **DAR availability.** Resolution quality depends on the dependency DARs being present (Splice's `daml/dars/` + the SDK package-db). If a dep isn't built/available, its refs stay `no_candidate` — degrades gracefully, no errors.
- **SDK/version coupling + multi-LF duplicates.** External `package_id`s are content hashes, so they're stable and self-identifying. But the SDK ships `daml-stdlib`/`daml-prim` for multiple LF versions under one `pkg-db_dir`, all exposing the same module names — ingesting >1 creates duplicate module qnames that break lookup. Mitigated by pinning to the workspace's single target LF (§3.1) and by the dup-tolerant scoped lookup (§4.1). This is the subtlest correctness trap in the design.
- **Graph size.** External symbols add nodes but are never embedded (the expensive axis), so the embedding pipeline cost is flat. FTS index gains bounded entries (§5).
- **Daml-only.** This layer is LF-specific; Rust crate resolution is a separate future slice and shares only the `Package`/`external` model, not the ingestion.

---

## 8. Test plan

- **Unit (11b):** LF/DAR parser against a tiny committed fixture DAR (or DALF) — assert package-id, module list, and a few exported definition names/kinds. **Explicitly assert names resolve to readable strings** (guards the interned-name dereferencing, §3.2) and that re-parsing the same DAR is byte-stable (content-hash dedup).
- **Unit/integration (3.1):** given a `daml.yaml` + multi-LF SDK package-db, only the pinned-LF `daml-stdlib`/`daml-prim` is ingested (no duplicate `DA.List` modules); `find_module_by_qualified_name` returns the single in-scope module rather than bailing.
- **Integration (11a/11c, gated on `CODEGRAPH_RS_TEST_NEO4J`):** index `tiny-daml` plus a small external-package fixture; assert (a) local + external `Package` nodes and `IN_PACKAGE`/`DEPENDS_ON` edges (incl. implicit SDK dep); (b) external `Module` `CONTAINS` its symbols and a call into it resolves with `confidence: import_resolved`; (c) external symbols absent from `semantic_search`, keyword `search`, and the embedded set; (d) Phase C does not bind a bare local name to an external symbol; (e) an existing `schema_version = 1` config still loads (no version-bump regression).
- **Corpus validation (11d):** on `codegraph-splice`, calls into **explicitly-imported** external modules bind (e.g. `DA.Foldable.forA_`, `DA.Optional.fromOptional`); `no_candidate` drops and `import_resolved` rises; embedded count unchanged from the local-only baseline. (The drop is partial — see §10 for the names that do *not* bind.)

---

## 9. Relationship to the deferred cleanup

This work intentionally precedes the graph-granularity cleanup discussed separately (pruning inert `UnresolvedRef`/`Import` scaffolding). Resolving external refs converts a large fraction of today's `no_candidate`/`import` refs into edges, so the genuinely-inert remainder — the real cleanup target — is only well-defined *after* this lands. Re-measure then.

---

## 10. Known limitations (as-built after 11a–11c)

Shipped slices resolve calls into **explicitly-imported external modules whose symbols are real top-level LF definitions**. Measured on `codegraph-splice` (172 DARs → 205 external packages, ~19.6k external symbols): Phase A `IMPORTS` 656 → 1,006, `import_resolved` calls 716 → 992, `no_candidate` 6,669 → 6,299.

The drop is modest because the remaining `no_candidate` calls fall into three buckets that the as-built design **cannot** bind, each needing disproportionate work for a capped payoff:

| Bucket | ~count | Why unbound | What it would take |
|---|---:|---|---|
| `Daml.Script` API (`submit`, `query`, `exerciseCmd`, `actAs`, `readAs`, `submitMustFail`) | ~2,056 | The **daml-script package is not ingested** — it's a dev/test dependency, not bundled in the app DARs under `daml/dars/`. | SDK-package-db enumeration (the deferred §3.1 SDK path) **and** re-export handling (below) — `Daml.Script` is itself a re-export aggregator. |
| Prelude / GHC re-exports (`pure`, `map`, `show`) | ~700 | `Prelude` is a **re-export aggregator with 0 own symbols**; the real definitions live in `DA.Internal.Prelude` / `GHC.Base` / `GHC.Show`. No `import` statement exists (Daml auto-imports Prelude), and even with one the module exposes nothing. Some names are ambiguous (`map` ∈ `GHC.Base` *and* `DA.NonEmpty`). | Capture the LF **export / re-export graph** (which module re-exports which names) — significant decoder work — plus implicit-Prelude import. |
| Data constructors & builtins (`Some`, `None`, `Party`, `ContractId`, `Int`) | ~1,400 | **Not top-level LF symbols at all** — constructors are part of a `data` decl, builtins are LF primitives. The ingestion walk (§3.2) emits only top-level definitions. | Out of reach without modelling constructors/builtins as distinct symbol kinds; arguably never worth it for resolution. |

**Decision: stop at explicit-import resolution.** The clean, correct win (real `DA.*` bindings via explicit imports) is delivered; pushing further requires both SDK-package-db ingestion *and* LF export-graph capture, and a large share of the remainder (constructors/builtins) is structurally unbindable. The remaining `no_candidate` count is therefore expected, not a defect — it is dominated by Prelude/Script re-exports and LF builtins/constructors. Revisit only if a concrete need (e.g. "jump to the definition of `submit`") justifies the export-graph + SDK-ingestion effort.
