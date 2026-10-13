# codegraph-rs Slice 7 — Vector Embeddings

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `codegraph-rs` produce 384-dim `BAAI/bge-small-en-v1.5` embeddings for every embed-eligible symbol, stored as `n.embedding` on the same `:Symbol` node Slice 2 wrote. After this slice, the Neo4j `symbol_embedding` vector index (already created by `apply_schema` at `src/db/schema.rs:58-77` since Slice 2) has data behind it; `codegraph-rs index` produces embeddings end-to-end; `codegraph-rs sync` re-embeds only the touched symbols; a new `codegraph-rs reembed` command re-embeds the whole workspace on demand; and `codegraph_rs_status` reports embedding coverage. **No semantic-search MCP tool yet** — that lands in the next slice, which is the first downstream consumer of this data.

**Architecture:** A new `src/vectors/` module with three small leaves and one orchestrator. Single code path for `index`, `sync`, and `reembed`: read composed text from the DB, embed in 128-row batches via `fastembed-rs::TextEmbedding::embed`, write back via one `UNWIND … SET n.embedding = row.embedding` per batch. The `fastembed-rs` ONNX model is lazy-initialized inside `vectors::run`; first invocation downloads ~130 MB to `~/.cache/codegraph-rs/fastembed/`, subsequent invocations hit the on-disk cache.

```
extract → write_extraction → resolution::resolve → vectors::run (NEW)
```

**Tech Stack additions:**
- `fastembed = "5"` — pure-Rust ONNX-runtime embedding library. Pinned to v5.x; the API used here (`TextEmbedding::try_new(InitOptions::new(EmbeddingModel::BGESmallENV15).with_cache_dir(...).with_show_download_progress(true))` and `model.embed(Vec<S: AsRef<str>>, None) -> Result<Vec<Vec<f32>>>`) is the public surface as of fastembed-rs 5.x.
- `dirs = "5"` — locate `~/.cache` portably; same crate the broader Rust ecosystem uses (`dirs::cache_dir()`).

No feature flags needed on either crate. fastembed's default features pull ONNX Runtime via `ort` (which downloads its own native binary). On macOS, no extra system deps; on Linux, glibc and `libgomp` are runtime requirements (documented in fastembed's README; Neo4j Desktop already pulls them in on typical dev machines).

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §8 (Vector embeddings, authoritative), §3 (where `vectors::embed` sits in the pipeline), §4 (`embedding: LIST<FLOAT>` property and the `symbol_embedding` vector index — already created), §9 ("Embedding after changes" — sync behavior), §10 (`codegraph_rs_status` reports embedding coverage).

**Slice 6 baseline:** HEAD `31ebd05`, tagged `slice-6-mcp`. 66 tests pass.

---

## What this slice does, conceptually

1. **One ingest path, three call sites.** Define `EmbedScope { All, Qids(Vec<String>) }` and a single `vectors::run(conn, cfg, scope) -> Result<EmbedStats>`. Wire it into:
   - `cli/index.rs` — after `resolution::resolve`, call `vectors::run(&conn, &cfg, EmbedScope::All)`.
   - `sync/mod.rs` — after `resolution::resolve`, call `vectors::run(&conn, &cfg, EmbedScope::Qids(qids_to_reembed))`, where the qid set is the symbols inside the just-written extraction batch.
   - `cli/reembed.rs` (NEW) — top-level wrapper: open connection, call `vectors::run(&conn, &cfg, EmbedScope::All)`.

2. **Compose text from the DB, not from `RawNode`.** See "Critical design questions" below for the tradeoff. The orchestrator queries Neo4j for `(qid, kind, qualified_name, signature, doc)` of every in-scope, embed-eligible symbol, composes the per-spec-§8 text in pure Rust, embeds it, writes it back. No coupling between `vectors::run` and the in-flight `ExtractionResult` structs.

3. **Embed-eligible kinds.** Per spec §8 "What gets embedded": `Function`, `Method`, `Class`, `Struct`, `Interface`, `Trait`, `Enum`, `TypeAlias`, `Module`, `Template`, `Choice`. Today's Rust extractor (per `src/extraction/result.rs`) only emits `Function`, `Method`, `Struct`, `Enum`, `Trait`, `TypeAlias`, `Module`, plus the non-embedded `File` and `Import`. The Daml slice will start emitting `Template`/`Choice`. The orchestrator's selector Cypher names the full spec list — labels missing from the current graph just yield zero matches; no harm.

4. **Batch size 128.** Per spec §8 "Batched at 128 texts per `fastembed-rs::embed()` call". This is the *embedding* batch, not the Cypher batch — both happen to be 128 in this slice. Each `model.embed(batch, None)` call produces 128 `Vec<f32>` of length 384; we immediately ship those to Neo4j via one `UNWIND` Cypher round-trip.

5. **`BoltType` for `LIST<FLOAT>`.** Neo4j's Bolt protocol carries floats as `BoltType::Float(f64)`. fastembed returns `Vec<f32>`. The conversion is: serialize each embedding as a `serde_json::Value::Array` of `serde_json::Number::from_f64(f64::from(x))` values, wrap each row `{"qid": String, "embedding": Array}` in the batch, run it through the existing `db::write::to_bolt` helper, pass via `.param("batch", bolt)`. This is the same shape we already use for node batches in `write_extraction`. The `json` feature of `neo4rs` (already in `Cargo.toml`) covers `serde_json::Value -> BoltType` including arrays of numbers.

6. **Lazy model load.** `fastembed::TextEmbedding::try_new(...)` is the hot path on first use — it downloads the ONNX file (~130 MB) the first time and loads it from disk every subsequent time. We construct the model inside `vectors::run` *exactly once per invocation* and pass `&model` down through helpers. Cost is paid once per `index`/`sync`/`reembed` run, not per batch.

7. **Status coverage.** `graph::status::StatusSnapshot` gains `embedding_coverage: EmbeddingCoverage { embedded: i64, total_eligible: i64 }`. Both come from Cypher (`MATCH (s:Symbol) WHERE labels(s) CONTAINS '<kind>' …`). `cli/status.rs` prints the ratio.

---

## Out of scope for Slice 7

- `codegraph_rs_semantic_search` MCP tool — needs both the embedding-store side (this slice) and a query-embedding side (next slice). Deferred to the post-embeddings MCP slice.
- `codegraph_rs_context` / `codegraph_rs_explore` — designed around `semantic_search`. Same next slice.
- Query-side embedding (embedding user query text). The model is locally loaded by `serve` in the next slice; this slice does not touch `mcp/` at all.
- Cross-encoder reranking — spec §8 "Not in v1".
- Embedding full function bodies — spec §8 "Body content is intentionally not included".
- Custom embedding models — spec §8 "v1 rejects other values"; already enforced in `config::workspace::parse` (`src/config/workspace.rs:114-119`).
- `--skip-embeddings` CLI flag — the spec lists it as an open question. We defer; users who want it can flip `[embeddings] enabled = false` in their config, see "Config off path" below.
- Re-embedding scope optimization (a.k.a. "only re-embed if `signature` or `doc` actually changed across runs") — current design always re-embeds touched symbols. Cheap enough for v1; spec doesn't ask for it.
- Tag for slice 7 in this slice's section count — there's no `slice-4-` tag (we skipped to 5 for sync); this slice continues that numbering.

---

## Critical design decisions

### D1. Composed text comes from the DB, not from the extraction structs.

**Options considered:**

- **(a) Re-query the DB.** `vectors::run` issues one Cypher per batch: `MATCH (s:Symbol) WHERE s.qid IN $qids AND any(k IN labels(s) WHERE k IN $kinds) RETURN s.qid, [l IN labels(s) WHERE l <> 'Symbol'][0] AS kind, s.qualified_name, s.signature, s.doc`. Compose text in Rust, embed, write.
- **(b) Plumb `RawNode`s through.** Pass the freshly extracted `Vec<RawNode>` directly from `index`/`sync` orchestrators into `vectors::run` to avoid the extra round-trip.

**Decision: (a).** Justifications:
1. **Single code path.** `reembed` already has to read from the DB (nothing fresh in memory). If `index`/`sync` use a different path, we have two code paths to maintain, two test surfaces, two failure modes. Spec §3 also frames vectors as a stage that reads from the graph (`SET n.embedding via Cypher`), which matches (a).
2. **Decoupling.** `vectors::run` only knows about Neo4j and `fastembed`. No coupling to `RawNode`, `ExtractionResult`, language extractors, or the orchestrator's batching strategy. When the Daml slice or a hypothetical Python extractor lands, `vectors::run` doesn't change.
3. **Cost is negligible.** One extra Cypher per 128 symbols. The vector index plus the `symbol_qid_unique` constraint give us $O(\log n)$ lookups; the round-trip latency to local Neo4j Desktop is ~1ms, dwarfed by the ~50ms/short-doc CPU cost of `fastembed::embed`.
4. **Robustness.** If extraction wrote a symbol but resolution somehow lost it (a bug), (a) still emits embeddings for whatever is actually in the graph. (b) would embed phantoms.

Cost of (a) over (b): one extra Cypher per batch. Negligible vs. 128 ONNX inferences.

### D2. Where the model loads, and how the latency is surfaced.

`fastembed::TextEmbedding::try_new(InitOptions::new(EmbeddingModel::BGESmallENV15).with_cache_dir(cache_dir).with_show_download_progress(true))` is the single load point. Decisions:

- **Cache dir:** `dirs::cache_dir().unwrap_or(/tmp).join("codegraph-rs/fastembed")`. Spec §8 explicitly names `~/.cache/codegraph-rs/fastembed/`.
- **Where it runs:** inside `vectors::run`, once. Not lazy via `OnceCell` because each `run()` is a complete unit of work that benefits from logging entry/exit cleanly. If a future slice needs the model in `serve` (e.g., for semantic-search query embedding), that slice introduces `McpServer::embedding_model: Arc<Mutex<TextEmbedding>>` and loads it eagerly during `serve` startup — separate concern, separate slice.
- **Latency surfacing:**
  - Log `tracing::info!("loading embedding model `bge-small-en-v1.5` (first-use download is ~130 MB, may take a minute)")` BEFORE calling `try_new`.
  - `with_show_download_progress(true)` makes fastembed print a `hf-hub` progress bar to stderr during the download phase.
  - On second + subsequent runs the message still prints but `try_new` completes in <500 ms (model is in cache).

We accept the cosmetic mismatch (always saying "may take a minute" when it usually doesn't); users see the message only when the run also includes embed work, and it's strictly better than a silent multi-minute hang on first index.

### D3. Sync re-embedding scope.

Per spec §9 "Embedding after changes": newly extracted symbols are embedded; unchanged symbols keep existing embeddings. Concretely: after sync's per-file delete + re-extraction + resolution rebuild, the symbols *in* the changed/added files have either no embedding (newly created qids) or stale ones (qids that survived the DELETE because they had no parent file — shouldn't happen, but defensively re-embed). Symbols in *unchanged* files keep their embeddings untouched.

**Implementation:** `sync::run` builds the qid set from `all_new_results` *before* calling `write_extraction`:

```rust
let touched_qids: Vec<String> = all_new_results
    .iter()
    .flat_map(|r| r.nodes.iter().map(|n| n.qid.clone()))
    .collect();
```

Then after the resolution rebuild, calls `vectors::run(conn, cfg, EmbedScope::Qids(touched_qids))`. The filter inside `vectors::run` further restricts to embed-eligible kinds via the selector Cypher, so passing every `RawNode`'s qid is fine — non-embed-eligible kinds (File, Import) get zero rows back.

**Deletes only:** if `sync` had only deletes and no adds/changes, `touched_qids` is empty and `vectors::run(..., Qids(vec![]))` short-circuits to an empty-batch no-op. Embeddings on surviving symbols are unaffected. Correct per spec §9.

### D4. `reembed` is a thin CLI over `vectors::run(..., EmbedScope::All)`.

```rust
pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    // ... open connection ...
    if !cfg.embeddings.enabled {
        anyhow::bail!(
            "embeddings disabled in config; set `[embeddings] enabled = true` in \
             `.codegraph/config.toml` to use `codegraph-rs reembed`"
        );
    }
    rt.block_on(async {
        let conn = Connection::open(...).await?;
        vectors::run(&conn, &cfg, vectors::EmbedScope::All).await
    })?;
    println!("Re-embedded workspace `{}`: ...", cfg.workspace.name);
    Ok(())
}
```

No schema apply (`apply_schema` was idempotent and is a `init`/`index`/`sync` concern). No file walk. No extraction. Just: connect, embed all eligible symbols, print stats.

### D5. Failure mode.

If `fastembed::embed` fails mid-batch (e.g., ONNX runtime panic, disk full), `vectors::run` returns the error. The caller (`cli::index`, `cli::sync`, `cli::reembed`) prints accumulated stats from the partial run *before* propagating the error, and `dispatch` returns non-zero. Partial embeddings stay in the graph; the next `index`/`sync`/`reembed` overwrites them via `SET n.embedding = …` (idempotent).

If the model download fails (network error during `try_new`), the error has fastembed's network context already; we just wrap with `anyhow::Context` (`"loading embedding model bge-small-en-v1.5"`) and propagate.

If `embeddings.enabled = false`, `vectors::run` is never called from `index`/`sync` (skipped at the call site with a `tracing::info!("embeddings disabled by config; skipping embed stage")`). `reembed` errors per D4.

### D6. Test gates.

Two env vars, both required for the integration test to run:
- `CODEGRAPH_RS_TEST_NEO4J=1` — same gate as every other Neo4j integration test.
- `CODEGRAPH_RS_TEST_EMBEDDINGS=1` — independent toggle. Loads the model. ~130 MB download on first CI invocation, ~500 ms subsequent. Off by default in CI's fast lane.

Both must be set; either-or-zero → SKIP with a clear message. Default `cargo test` continues to pass with no env vars set.

Pure-Rust unit tests for `compose_text` (no model load, no Neo4j) run unconditionally.

### D7. `embeddings.enabled = false` path and `codegraph_rs_status` coverage.

`status::collect` always queries the DB for actual embedding counts — *even when config disables embeddings*. Rationale: a user who toggles it off mid-project should see the leftover coverage from when it was on, not a falsely-zeroed number. The output is "ground truth from the graph", not "what the config claims".

If a workspace has `enabled = false` and zero embeddings, status reports `embedded: 0 / N`. If a workspace flipped from true to false, status reports `embedded: K / N` where K is the historical count.

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml                        # add fastembed = "5", dirs = "5"
  src/
    lib.rs                          # add `pub mod vectors;`
    cli/
      mod.rs                        # add Reembed variant + dispatch
      reembed.rs                    # NEW: codegraph-rs reembed command
      index.rs                      # MODIFY: call vectors::run after resolve
      status.rs                     # MODIFY: print embedding_coverage
    sync/
      mod.rs                        # MODIFY: build touched_qids, call vectors::run
    graph/
      status.rs                     # MODIFY: add EmbeddingCoverage; query coverage
    vectors/
      mod.rs                        # NEW: re-exports + EmbedScope + run() entry
      compose.rs                    # NEW: pure compose_text() helper (TDD)
      model.rs                      # NEW: model loader + cache_dir helper
      query.rs                      # NEW: fetch in-scope (qid, kind, qname, sig, doc)
      write.rs                      # NEW: UNWIND batched SET n.embedding
  tests/
    vectors_unit.rs                 # NEW: compose_text + cache_dir tests
    integration_vectors.rs          # NEW: gated; index tiny-rust + assert embeddings
```

---

### Task 1: Module skeleton + dependencies

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/lib.rs`
- Create: `src/vectors/mod.rs`

- [ ] **Step 1: Add deps to Cargo.toml**

Append to `[dependencies]`:

```toml
fastembed = "5"
dirs = "5"
```

> **Note on `fastembed` and platform compat.** `fastembed = "5"` pulls `ort` 2.x (ONNX Runtime). On macOS arm64 and Linux x86_64 (the platforms covered by Slice 1 prerequisites) `ort` downloads its native runtime at build time and caches it under `target/debug/`. No system package needed. Windows is not a Slice 7 concern.

- [ ] **Step 2: Add `pub mod vectors;` to src/lib.rs**

Alphabetical, after `pub mod sync;`:

```rust
pub mod cli;
pub mod config;
pub mod db;
pub mod extraction;
pub mod graph;
pub mod mcp;
pub mod resolution;
pub mod sync;
pub mod vectors;
```

- [ ] **Step 3: Create src/vectors/mod.rs**

```rust
//! Vector embeddings (spec §8).
//!
//! Single orchestrator (`run`) that loads a `fastembed-rs` model, fetches
//! composed text for in-scope symbols from Neo4j, embeds in 128-row batches,
//! and writes `n.embedding = $vec` back to the same `:Symbol` nodes.
//!
//! Used by `cli::index`, `sync`, and `cli::reembed`. No coupling to extraction
//! structs — see plan D1.

pub mod compose;
pub mod model;
pub mod query;
pub mod write;

use anyhow::Result;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;

/// What subset of the workspace to embed.
#[derive(Debug, Clone)]
pub enum EmbedScope {
    /// All embed-eligible symbols in the workspace.
    All,
    /// Only symbols whose qid is in the given list. Used by `sync` to
    /// re-embed just the symbols in added/changed files.
    Qids(Vec<String>),
}

/// What `run` reports back to its caller. Printed by CLI commands and rolled
/// into log lines.
#[derive(Debug, Clone, Default)]
pub struct EmbedStats {
    /// Number of symbols whose embedding was (re)written.
    pub embedded: usize,
    /// Number of fastembed batches issued.
    pub batches: usize,
    /// Number of symbols that were in scope but ineligible (skipped early).
    pub skipped_ineligible: usize,
}

/// Embed all in-scope symbols. Filled in by Task 6.
pub async fn run(
    _conn: &Connection,
    _cfg: &WorkspaceConfig,
    _scope: EmbedScope,
) -> Result<EmbedStats> {
    anyhow::bail!("vectors::run: not yet implemented (filled in by Task 6)")
}
```

- [ ] **Step 4: Create stub submodules**

`src/vectors/compose.rs`:
```rust
//! Composed-text builder per spec §8. Filled in by Task 2.
```

`src/vectors/model.rs`:
```rust
//! fastembed-rs model loader + cache dir resolution. Filled in by Task 3.
```

`src/vectors/query.rs`:
```rust
//! Cypher: fetch composed-text fields for in-scope symbols. Filled in by Task 4.
```

`src/vectors/write.rs`:
```rust
//! Cypher: batched UNWIND SET n.embedding = row.embedding. Filled in by Task 5.
```

- [ ] **Step 5: Build, clippy, fmt, run tests**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo build 2>&1 | tail -5
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: clean build (downloads fastembed + dirs + ort + transitive deps; first build is slow, ~3-5 min on first invocation). 66 tests still pass.

> **Note on first build cost.** Pulling `fastembed = "5"` brings in `ort`, `tokenizers`, `ndarray`, plus their transitive ML deps. Expect ~120 MB of crates compiled. Subsequent rebuilds are fast (incremental). The first `cargo build` in this task is the heaviest step in the entire slice.

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml Cargo.lock src/lib.rs src/vectors/
git commit -m "chore(vectors): add fastembed + dirs deps and module skeleton"
```

---

### Task 2: Composed-text builder (TDD)

The composed text formula from spec §8 is pure data — kind label, qualified name, optional signature, optional doc. Tests run without `fastembed` or Neo4j. The function signature is `compose_text(kind: &str, qualified_name: &str, signature: Option<&str>, doc: Option<&str>) -> String` so it can be called from both `vectors::run` (DB rows) and any future code path (e.g., a unit test composing from `RawNode`).

**Files:**
- Modify: `src/vectors/compose.rs`
- Create: `tests/vectors_unit.rs`

- [ ] **Step 1: Write failing tests**

Create `tests/vectors_unit.rs`:

```rust
//! Unit tests for vectors module — pure-Rust, no fastembed model load, no Neo4j.

use codegraph_rs::vectors::compose::compose_text;

#[test]
fn compose_text_full() {
    let s = compose_text(
        "Function",
        "crate::util::parse_config",
        Some("fn parse_config(path: &Path) -> Result<Config, ConfigError>"),
        Some("Parse a config file from disk, returning the parsed settings or a descriptive error."),
    );
    assert_eq!(
        s,
        "function: crate::util::parse_config\n\
         fn parse_config(path: &Path) -> Result<Config, ConfigError>\n\
         Parse a config file from disk, returning the parsed settings or a descriptive error."
    );
}

#[test]
fn compose_text_kind_is_lowercased() {
    let s = compose_text("Struct", "crate::Config", None, None);
    // The "kind:" prefix in spec §8 example is lowercase ("function:", not "Function:").
    assert!(s.starts_with("struct: crate::Config"), "got: {s:?}");
}

#[test]
fn compose_text_omits_missing_signature() {
    let s = compose_text("Trait", "crate::Greet", None, Some("Greets the user."));
    assert_eq!(s, "trait: crate::Greet\nGreets the user.");
}

#[test]
fn compose_text_omits_missing_doc() {
    let s = compose_text(
        "Function",
        "crate::main",
        Some("fn main()"),
        None,
    );
    assert_eq!(s, "function: crate::main\nfn main()");
}

#[test]
fn compose_text_kind_only() {
    let s = compose_text("Module", "crate::util", None, None);
    assert_eq!(s, "module: crate::util");
}

#[test]
fn compose_text_trims_trailing_whitespace_in_fields() {
    // Real signatures sometimes carry trailing newlines from tree-sitter capture.
    // The composed text should not produce double-blank lines.
    let s = compose_text(
        "Function",
        "crate::helper",
        Some("fn helper() -> u32\n"),
        Some("Returns 42.\n\n"),
    );
    assert_eq!(s, "function: crate::helper\nfn helper() -> u32\nReturns 42.");
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test vectors_unit 2>&1 | tail -10
```

Expected: compile errors about missing `compose_text`.

- [ ] **Step 3: Implement compose_text**

Overwrite `src/vectors/compose.rs`:

```rust
//! Composed-text builder per spec §8.
//!
//! Builds the per-symbol text that the embedding model sees:
//!
//! ```text
//! <kind>: <qualified_name>
//! <signature>
//! <doc>
//! ```
//!
//! `signature` and `doc` lines are omitted when absent. Trailing whitespace
//! per field is trimmed so the composed text never carries spurious blank
//! lines into the tokenizer.

/// Compose the per-symbol embedding text. Pure function — no I/O.
///
/// `kind` is the Neo4j label name (PascalCase, e.g. "Function"); it is
/// lowercased in the output prefix to match the spec example ("function:").
pub fn compose_text(
    kind: &str,
    qualified_name: &str,
    signature: Option<&str>,
    doc: Option<&str>,
) -> String {
    let mut out = String::with_capacity(128);
    out.push_str(&kind.to_lowercase());
    out.push_str(": ");
    out.push_str(qualified_name.trim_end());

    if let Some(sig) = signature.map(str::trim_end).filter(|s| !s.is_empty()) {
        out.push('\n');
        out.push_str(sig);
    }
    if let Some(d) = doc.map(str::trim_end).filter(|s| !s.is_empty()) {
        out.push('\n');
        out.push_str(d);
    }
    out
}
```

- [ ] **Step 4: Run tests, fmt, clippy**

```bash
cargo test --test vectors_unit 2>&1 | tail -10
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

Expected: 6 tests pass. fmt/clippy clean.

- [ ] **Step 5: Commit**

```bash
git add src/vectors/compose.rs tests/vectors_unit.rs
git commit -m "feat(vectors): compose_text builder per spec §8"
```

---

### Task 3: Model loader + cache dir

**Files:**
- Modify: `src/vectors/model.rs`
- Modify: `tests/vectors_unit.rs` (add cache_dir tests)

This task wraps `fastembed::TextEmbedding::try_new(...)` behind a thin `load_model(cache_dir: &Path) -> Result<TextEmbedding>` helper, and adds `default_cache_dir() -> PathBuf` returning `~/.cache/codegraph-rs/fastembed/` (or a `tempdir`-style fallback on systems where `dirs::cache_dir` fails).

- [ ] **Step 1: Implement `src/vectors/model.rs`**

```rust
//! fastembed-rs model loader + on-disk cache directory.
//!
//! `default_cache_dir()` returns `~/.cache/codegraph-rs/fastembed/` per
//! spec §8. The directory is created on demand (mkdir -p) before the model
//! download; failures bubble up with a clear error.
//!
//! `load_model()` returns a ready-to-use `TextEmbedding` initialized with
//! `BAAI/bge-small-en-v1.5` (the only model accepted in v1, per spec §8 and
//! workspace config validation in `src/config/workspace.rs:114-119`).

use anyhow::{Context, Result};
use fastembed::{EmbeddingModel, InitOptions, TextEmbedding};
use std::path::{Path, PathBuf};

const MODEL_DIRNAME: &str = "codegraph-rs/fastembed";

/// Resolve the on-disk model cache directory.
///
/// Order of preference:
/// 1. `$XDG_CACHE_HOME/codegraph-rs/fastembed/` (or platform equivalent via
///    `dirs::cache_dir()` — `~/.cache/...` on Linux, `~/Library/Caches/...`
///    on macOS).
/// 2. `<tempdir>/codegraph-rs/fastembed/` if `dirs::cache_dir()` returns
///    None (sandboxed or unusual environments).
///
/// Returns the directory path; does NOT create it on disk. Callers that
/// actually instantiate the model should `std::fs::create_dir_all` first.
pub fn default_cache_dir() -> PathBuf {
    match dirs::cache_dir() {
        Some(d) => d.join(MODEL_DIRNAME),
        None => std::env::temp_dir().join(MODEL_DIRNAME),
    }
}

/// Load (download if first time, mmap from disk on subsequent runs) the
/// `BAAI/bge-small-en-v1.5` model into a `TextEmbedding`.
///
/// First invocation downloads ~130 MB to `cache_dir` and may take several
/// minutes on a slow connection. Progress is printed to stderr by fastembed.
pub fn load_model(cache_dir: &Path) -> Result<TextEmbedding> {
    std::fs::create_dir_all(cache_dir)
        .with_context(|| format!("creating model cache dir `{}`", cache_dir.display()))?;

    tracing::info!(
        "loading embedding model `bge-small-en-v1.5` (first-use download is ~130 MB; \
         subsequent runs load from cache at {})",
        cache_dir.display()
    );

    let opts = InitOptions::new(EmbeddingModel::BGESmallENV15)
        .with_cache_dir(cache_dir.to_path_buf())
        .with_show_download_progress(true);

    TextEmbedding::try_new(opts).with_context(|| {
        format!(
            "loading embedding model `bge-small-en-v1.5` (cache dir: {})",
            cache_dir.display()
        )
    })
}
```

> **Implementer notes on the fastembed-rs 5.x API:**
> - `InitOptions::new(EmbeddingModel)` is the documented constructor; builder methods are `.with_cache_dir(PathBuf)` and `.with_show_download_progress(bool)`.
> - `TextEmbedding::try_new(InitOptions) -> Result<TextEmbedding, anyhow::Error>` — the error type is already `anyhow`, so `with_context` chains naturally.
> - `EmbeddingModel::BGESmallENV15` is the enum variant for `BAAI/bge-small-en-v1.5`. Verify by `cargo doc --open` on the `fastembed` crate after Task 1 — if the variant has been renamed in 5.x (e.g., `BgeSmallEnV15`), update this and Task 6's batch.

- [ ] **Step 2: Add cache_dir unit tests to tests/vectors_unit.rs**

Append to the existing test file:

```rust
use codegraph_rs::vectors::model::default_cache_dir;

#[test]
fn default_cache_dir_contains_module_dirname() {
    let p = default_cache_dir();
    // Path components include both "codegraph-rs" and "fastembed".
    let path_str = p.to_string_lossy();
    assert!(
        path_str.contains("codegraph-rs"),
        "cache dir should contain `codegraph-rs`; got {path_str}"
    );
    assert!(
        path_str.contains("fastembed"),
        "cache dir should contain `fastembed`; got {path_str}"
    );
}

#[test]
fn default_cache_dir_is_absolute() {
    let p = default_cache_dir();
    assert!(p.is_absolute(), "cache dir should be absolute; got {p:?}");
}
```

> **Note on test isolation.** These tests do NOT call `load_model` — that would trigger the ~130 MB download. The actual model load is exercised by Task 9's integration test, which is gated on `CODEGRAPH_RS_TEST_EMBEDDINGS=1`.

- [ ] **Step 3: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test --test vectors_unit 2>&1 | tail -10
cargo test 2>&1 | tail -5
```

Expected: 68 tests pass (66 + 2 new cache_dir tests; the 6 compose tests carry forward).

- [ ] **Step 4: Commit**

```bash
git add src/vectors/model.rs tests/vectors_unit.rs
git commit -m "feat(vectors): model loader with on-disk cache and progress logging"
```

---

### Task 4: Selector query — fetch composed-text fields

The selector returns one row per in-scope embed-eligible symbol with the fields needed to compose text. Two scopes:
- `All` — every symbol whose label is in the embed-eligible set.
- `Qids(vec)` — every symbol whose qid is in `vec` AND whose label is embed-eligible.

The label list per spec §8: `Function, Method, Class, Struct, Interface, Trait, Enum, TypeAlias, Module, Template, Choice`. Today's Rust extractor produces a subset; that's fine — the selector matches against the full list and unknown labels return zero rows.

**Files:**
- Modify: `src/vectors/query.rs`

- [ ] **Step 1: Implement src/vectors/query.rs**

```rust
//! Selector Cypher: fetch (qid, kind, qualified_name, signature, doc) tuples
//! for symbols that should be embedded, filtered by `EmbedScope`.

use anyhow::{Context, Result};
use neo4rs::query as cypher;
use serde_json::json;

use crate::db::write::to_bolt;
use crate::db::Connection;
use crate::vectors::EmbedScope;

/// One row from the selector — the input to `compose_text` for a single symbol.
#[derive(Debug, Clone)]
pub struct SymbolForEmbed {
    pub qid: String,
    pub kind: String,
    pub qualified_name: String,
    pub signature: Option<String>,
    pub doc: Option<String>,
}

/// The embed-eligible Neo4j labels per spec §8. Kept as a Cypher list literal
/// so the query plan is stable across calls.
const EMBED_KINDS_CYPHER_LIST: &str =
    "['Function','Method','Class','Struct','Interface','Trait','Enum',\
      'TypeAlias','Module','Template','Choice']";

/// Fetch all (qid, kind, qualified_name, signature, doc) rows for symbols
/// matching `scope` and the embed-eligible kind set.
pub async fn fetch_in_scope(conn: &Connection, scope: &EmbedScope) -> Result<Vec<SymbolForEmbed>> {
    match scope {
        EmbedScope::All => fetch_all(conn).await,
        EmbedScope::Qids(qids) if qids.is_empty() => Ok(Vec::new()),
        EmbedScope::Qids(qids) => fetch_by_qids(conn, qids).await,
    }
}

async fn fetch_all(conn: &Connection) -> Result<Vec<SymbolForEmbed>> {
    let text = format!(
        "MATCH (s:Symbol) \
         WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS_CYPHER_LIST}) \
         RETURN s.qid AS qid, \
                [l IN labels(s) WHERE l <> 'Symbol' AND l IN {EMBED_KINDS_CYPHER_LIST}][0] AS kind, \
                s.qualified_name AS qualified_name, \
                s.signature AS signature, \
                s.doc AS doc"
    );
    run_selector(conn, &text, None).await
}

async fn fetch_by_qids(conn: &Connection, qids: &[String]) -> Result<Vec<SymbolForEmbed>> {
    let text = format!(
        "UNWIND $qids AS q \
         MATCH (s:Symbol {{qid: q}}) \
         WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS_CYPHER_LIST}) \
         RETURN s.qid AS qid, \
                [l IN labels(s) WHERE l <> 'Symbol' AND l IN {EMBED_KINDS_CYPHER_LIST}][0] AS kind, \
                s.qualified_name AS qualified_name, \
                s.signature AS signature, \
                s.doc AS doc"
    );
    let qids_param = to_bolt(json!(qids)).context("encoding qids param")?;
    run_selector(conn, &text, Some(qids_param)).await
}

async fn run_selector(
    conn: &Connection,
    text: &str,
    qids_param: Option<neo4rs::BoltType>,
) -> Result<Vec<SymbolForEmbed>> {
    let mut q = cypher(text);
    if let Some(p) = qids_param {
        q = q.param("qids", p);
    }
    let mut stream = conn.graph.execute(q).await.context("selector execute")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.context("selector row stream")? {
        out.push(SymbolForEmbed {
            qid: row.get("qid").context("missing qid")?,
            kind: row
                .get::<Option<String>>("kind")
                .context("missing kind")?
                .unwrap_or_default(),
            qualified_name: row
                .get::<Option<String>>("qualified_name")
                .context("missing qualified_name")?
                .unwrap_or_default(),
            signature: row.get::<Option<String>>("signature")?,
            doc: row.get::<Option<String>>("doc")?,
        });
    }
    Ok(out)
}
```

> **Why two query shapes.** `All` avoids an UNWIND-empty trap (some Neo4j versions plan an empty UNWIND poorly). `Qids` uses UNWIND so the qid set is a parameter, not an inlined literal — same pattern as `db::write::write_extraction`.

> **Why `qualified_name.unwrap_or_default()`.** All embed-eligible symbol labels in slice 2's writer set `qualified_name` (`src/db/write.rs:67`), but defensive: a corrupted row should produce an empty composed prefix, not crash the whole batch.

- [ ] **Step 2: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 68 tests pass (no new tests in this task — exercised end-to-end by Task 9's integration test).

- [ ] **Step 3: Commit**

```bash
git add src/vectors/query.rs
git commit -m "feat(vectors): selector query for in-scope embed-eligible symbols"
```

---

### Task 5: Write-back — UNWIND SET n.embedding

The writer takes a `Vec<(String /*qid*/, Vec<f32>)>` and ships it as a single Cypher `UNWIND $batch AS row MATCH (n:Symbol {qid: row.qid}) SET n.embedding = row.embedding`. Float conversion: `f32 -> f64` via `f64::from(x)`, then `serde_json::Number::from_f64(...)` (which returns `Option<Number>`; `unwrap` is safe because fastembed never produces NaN/Inf for normal text inputs, but we error defensively).

**Files:**
- Modify: `src/vectors/write.rs`

- [ ] **Step 1: Implement src/vectors/write.rs**

```rust
//! Batched UNWIND SET n.embedding = row.embedding writer.

use anyhow::{Context, Result};
use neo4rs::query as cypher;
use serde_json::{Map, Number, Value};

use crate::db::write::to_bolt;
use crate::db::Connection;

/// Spec §8: "Batched at 128 texts per `fastembed-rs::embed()` call. Each
/// batch is written with one Cypher round-trip." The embed batch and the
/// Cypher batch are the same number.
pub const EMBED_BATCH_SIZE: usize = 128;

/// Write a single batch of embeddings. `pairs` is `(qid, embedding)`.
///
/// Idempotent: the SET clause overwrites any existing embedding on each
/// matched node. Symbols whose qid is in `pairs` but no longer exists in
/// the graph are silently skipped by Neo4j's MATCH semantics.
pub async fn write_embedding_batch(
    conn: &Connection,
    pairs: &[(String, Vec<f32>)],
) -> Result<()> {
    if pairs.is_empty() {
        return Ok(());
    }

    let text = "UNWIND $batch AS row \
                MATCH (n:Symbol {qid: row.qid}) \
                SET n.embedding = row.embedding";

    let batch_rows: Vec<Value> = pairs
        .iter()
        .map(|(qid, vec)| pair_to_json(qid, vec))
        .collect::<Result<Vec<_>>>()?;

    let bolt = to_bolt(Value::Array(batch_rows)).context("serialising embedding batch")?;

    conn.graph
        .execute(cypher(text).param("batch", bolt))
        .await
        .context("writing embedding batch")?
        .next()
        .await
        .ok();
    Ok(())
}

fn pair_to_json(qid: &str, vec: &[f32]) -> Result<Value> {
    let arr: Vec<Value> = vec
        .iter()
        .map(|x| {
            Number::from_f64(f64::from(*x))
                .map(Value::Number)
                .ok_or_else(|| anyhow::anyhow!("non-finite float in embedding for qid `{qid}`"))
        })
        .collect::<Result<Vec<_>>>()?;
    let mut m = Map::new();
    m.insert("qid".into(), Value::String(qid.to_string()));
    m.insert("embedding".into(), Value::Array(arr));
    Ok(Value::Object(m))
}
```

> **On `BoltType` for `LIST<FLOAT>`.** The `json` feature of `neo4rs` (already on in `Cargo.toml`, line 19) provides `BoltType::TryFrom<serde_json::Value>`. A `Value::Array` of `Value::Number` (f64-backed) becomes a `BoltType::List` of `BoltType::Float`. Neo4j's Bolt protocol carries those as 64-bit floats; the vector index converts to its internal storage (typically float32) when indexing. The round-trip is lossy only in that f32→f64→f32 may shift the LSB of a few mantissa bits, which the cosine-similarity index does not care about.

> **On `next().await.ok()`.** Mirrors `write_extraction` (`src/db/write.rs:88-91`) — the result stream for a write-only Cypher returns no rows; we just need to drive the future to completion.

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 68 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/vectors/write.rs
git commit -m "feat(vectors): UNWIND batched SET n.embedding writer"
```

---

### Task 6: Orchestrator — `vectors::run`

Wires together: selector → compose → model.embed → batch write. Logs and returns `EmbedStats`. Single entry point; no extraction or RawNode coupling.

**Files:**
- Modify: `src/vectors/mod.rs`

- [ ] **Step 1: Replace src/vectors/mod.rs**

```rust
//! Vector embeddings (spec §8).
//!
//! Single orchestrator (`run`) that loads a `fastembed-rs` model, fetches
//! composed text for in-scope symbols from Neo4j, embeds in 128-row batches,
//! and writes `n.embedding = $vec` back to the same `:Symbol` nodes.

pub mod compose;
pub mod model;
pub mod query;
pub mod write;

use anyhow::{Context, Result};
use tracing::info;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;
use crate::vectors::compose::compose_text;
use crate::vectors::model::{default_cache_dir, load_model};
use crate::vectors::query::fetch_in_scope;
use crate::vectors::write::{write_embedding_batch, EMBED_BATCH_SIZE};

/// What subset of the workspace to embed.
#[derive(Debug, Clone)]
pub enum EmbedScope {
    /// All embed-eligible symbols in the workspace.
    All,
    /// Only symbols whose qid is in the given list. Used by `sync` to
    /// re-embed just the symbols in added/changed files.
    Qids(Vec<String>),
}

#[derive(Debug, Clone, Default)]
pub struct EmbedStats {
    pub embedded: usize,
    pub batches: usize,
    pub skipped_ineligible: usize,
}

/// Embed all in-scope symbols, writing `n.embedding = $vec` back to Neo4j.
///
/// **Embedding model:** spec-mandated `BAAI/bge-small-en-v1.5`. The model is
/// loaded once per invocation (~500ms warm, ~minutes for first-time download
/// to `~/.cache/codegraph-rs/fastembed/`).
///
/// **Batching:** spec §8 — 128 texts per `model.embed` call; one Cypher
/// round-trip per batch.
///
/// **Idempotent:** each call overwrites the embedding of every in-scope
/// symbol. Safe to retry after a partial failure.
pub async fn run(
    conn: &Connection,
    cfg: &WorkspaceConfig,
    scope: EmbedScope,
) -> Result<EmbedStats> {
    if !cfg.embeddings.enabled {
        info!("embeddings disabled by workspace config; skipping vectors::run");
        return Ok(EmbedStats::default());
    }

    let scope_label = match &scope {
        EmbedScope::All => "all".to_string(),
        EmbedScope::Qids(q) => format!("{} qids", q.len()),
    };

    let rows = fetch_in_scope(conn, &scope)
        .await
        .with_context(|| format!("fetching in-scope symbols ({scope_label})"))?;

    if rows.is_empty() {
        info!("vectors::run({scope_label}): nothing to embed");
        return Ok(EmbedStats::default());
    }
    info!(
        "vectors::run({scope_label}): {} symbols to embed",
        rows.len()
    );

    // Load model exactly once for this invocation. `load_model` is a
    // synchronous CPU-bound call (mmaps the ONNX file); we run it directly
    // on the current async runtime thread. The download path (first use)
    // does its own progress reporting to stderr.
    let cache_dir = default_cache_dir();
    let mut model = load_model(&cache_dir).context("loading embedding model")?;

    let mut stats = EmbedStats::default();
    for batch in rows.chunks(EMBED_BATCH_SIZE) {
        let texts: Vec<String> = batch
            .iter()
            .map(|r| compose_text(&r.kind, &r.qualified_name, r.signature.as_deref(), r.doc.as_deref()))
            .collect();
        let vecs = embed_batch(&mut model, &texts).with_context(|| {
            format!("embedding batch of {} symbols", texts.len())
        })?;

        // Defensive: fastembed must return one vector per input.
        if vecs.len() != batch.len() {
            anyhow::bail!(
                "fastembed returned {} vectors for {} inputs",
                vecs.len(),
                batch.len()
            );
        }

        let pairs: Vec<(String, Vec<f32>)> = batch
            .iter()
            .zip(vecs.into_iter())
            .map(|(row, vec)| (row.qid.clone(), vec))
            .collect();

        write_embedding_batch(conn, &pairs).await.with_context(|| {
            format!("writing batch of {} embeddings", pairs.len())
        })?;

        stats.embedded += pairs.len();
        stats.batches += 1;
    }

    info!(
        "vectors::run({scope_label}) done: embedded={}, batches={}",
        stats.embedded, stats.batches
    );
    Ok(stats)
}

/// Run one fastembed batch. Synchronous; the caller is already inside a
/// tokio task, but `model.embed` is pure CPU work.
fn embed_batch(model: &mut fastembed::TextEmbedding, texts: &[String]) -> Result<Vec<Vec<f32>>> {
    let refs: Vec<&str> = texts.iter().map(String::as_str).collect();
    model
        .embed(refs, None)
        .map_err(|e| anyhow::anyhow!("fastembed embed failed: {e}"))
}

/// Helper for callers that built a qid list and want to short-circuit on empty.
#[allow(dead_code)]
pub fn would_be_noop(scope: &EmbedScope) -> bool {
    matches!(scope, EmbedScope::Qids(q) if q.is_empty())
}
```

> **Why `embed_batch` is a free function, not async.** `fastembed::TextEmbedding::embed` is synchronous (CPU-only, no I/O). Calling a sync function from an async context blocks the executor; for batches of 128 short texts on CPU, this is ~5-10s tops, which would hold up other tasks on the same tokio worker. Our CLI is a single-purpose tool that does nothing else during `vectors::run`, so this is acceptable — no other work is starving. If a future slice runs embeddings concurrently with other I/O (e.g., a `serve`-time query embedding), it should `tokio::task::spawn_blocking(move || embed_batch(...))` to avoid blocking the runtime. Not needed in slice 7.

> **On `&mut model`.** `fastembed::TextEmbedding::embed` takes `&mut self` (the model holds a tokenizer + session that mutates an internal buffer). We carry `&mut model` through `embed_batch`. The model itself is single-threaded; we never share it across tasks.

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 68 tests pass. No new tests (Task 9 adds the integration test).

> **Note: `dead_code` on `would_be_noop`.** It's used by Task 8 (`sync::run` short-circuit). Keep the `#[allow(dead_code)]` in this task to avoid a clippy failure between Task 6 and Task 8.

- [ ] **Step 3: Commit**

```bash
git add src/vectors/mod.rs
git commit -m "feat(vectors): orchestrator — load model, batch embed, batch write"
```

---

### Task 7: Hook into `cli::index`

**Files:**
- Modify: `src/cli/index.rs`

- [ ] **Step 1: Add the vectors hook in `cli::index`**

Edit `src/cli/index.rs`. The current end-of-flow is `apply_schema → write_extraction → resolve → println!`. Add `vectors::run(&conn, &cfg, vectors::EmbedScope::All).await` immediately after `resolve`, and capture stats into the printed summary.

Replace the existing `rt.block_on` body:

```rust
    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    let embed_stats = rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        )
        .await
        .context("connecting to workspace database")?;

        apply_schema(&conn).await.context("applying schema")?;

        write_extraction(&conn, results)
            .await
            .context("writing extraction to Neo4j")?;

        let stats = resolution::resolve(&conn).await.context("resolving references")?;
        info!(
            "resolution stats: imports={}, calls(B/C)={}/{}, refs(B/C)={}/{}, types(B/C)={}/{}, ambiguous={}, no_candidate={}",
            stats.imports_resolved,
            stats.calls_resolved_phase_b, stats.calls_resolved_phase_c,
            stats.refs_resolved_phase_b, stats.refs_resolved_phase_c,
            stats.types_resolved_phase_b, stats.types_resolved_phase_c,
            stats.unresolved_ambiguous, stats.unresolved_no_candidate,
        );

        let embed_stats = crate::vectors::run(
            &conn,
            &cfg,
            crate::vectors::EmbedScope::All,
        )
        .await
        .context("embedding workspace symbols")?;

        Ok::<crate::vectors::EmbedStats, anyhow::Error>(embed_stats)
    })?;

    println!(
        "Indexed {file_count} files: {total_nodes} nodes, {total_edges} edges, \
         {total_refs} unresolved refs, {} embeddings ({} batches)",
        embed_stats.embedded, embed_stats.batches
    );
    Ok(())
}
```

> **On config-disabled embeddings.** When `cfg.embeddings.enabled = false`, `vectors::run` returns an `EmbedStats::default()` (zeros) without touching the model. The summary line still prints "0 embeddings (0 batches)", which is accurate and signals to the user that the step ran but produced nothing.

> **On error propagation.** `vectors::run` returning `Err` will short-circuit the `rt.block_on` and bubble up before `println!`. The user sees `Indexed ... [error]: embedding workspace symbols → ...` and exits non-zero. Partial embeddings (whatever batches did complete) remain in the graph and will be overwritten on the next `index`/`reembed`.

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 68 tests still pass. The `cli::index` change is exercised end-to-end by the existing `tests/integration_index.rs` once Task 9's test lands; for now, the indexed graph in `integration_index.rs` simply gains embeddings on its symbols, which the existing assertions don't check, so the test remains green.

- [ ] **Step 3: Commit**

```bash
git add src/cli/index.rs
git commit -m "feat(cli): run vectors::run after resolution during index"
```

---

### Task 8: Hook into `sync::run`

**Files:**
- Modify: `src/sync/mod.rs`

`sync` builds the qid set from `all_new_results` (the symbols freshly written for added+changed files) and passes it as `EmbedScope::Qids`. Deletes don't contribute qids; the surviving symbols' embeddings are untouched.

- [ ] **Step 1: Edit src/sync/mod.rs**

In the orchestrator body, between the `write_extraction` block and the `resolution::resolve` block, capture the qid set:

```rust
    // Capture the qids of newly-written symbols BEFORE moving `all_new_results`
    // into write_extraction. Used below to scope re-embedding to just the
    // symbols inside added/changed files (spec §9 "Embedding after changes").
    let touched_qids: Vec<String> = all_new_results
        .iter()
        .flat_map(|r| r.nodes.iter().map(|n| n.qid.clone()))
        .collect();

    // Write all new extraction results in one batch (across roots).
    if !all_new_results.is_empty() {
        write_extraction(conn, all_new_results)
            .await
            .context("write_extraction")?;
    }

    // Always run resolution rebuild: even pure deletions invalidate confidence edges.
    let resolution_stats = resolution::resolve(conn).await.context("resolve")?;
    stats.resolution = resolution_stats;

    // Re-embed only the symbols in added/changed files (spec §9).
    let scope = crate::vectors::EmbedScope::Qids(touched_qids);
    if crate::vectors::would_be_noop(&scope) {
        info!("sync done with no symbol changes; skipping vectors::run");
    } else {
        let embed_stats = crate::vectors::run(conn, cfg, scope)
            .await
            .context("re-embedding touched symbols")?;
        info!(
            "sync re-embedded {} symbols across {} batches",
            embed_stats.embedded, embed_stats.batches
        );
    }

    info!(
        "sync done: added={}, changed={}, deleted={}, resolved={}, unresolved={}",
        stats.total_added(),
        stats.total_changed(),
        stats.total_deleted(),
        stats.resolution.total_resolved(),
        stats.resolution.total_unresolved()
    );

    Ok(stats)
}
```

> **On the `would_be_noop` short-circuit.** When `touched_qids` is empty, `vectors::run` would itself short-circuit (the selector returns 0 rows, no model load). The wrapper here avoids printing the "embeddings disabled" / "loading model" info-log lines for pure-delete syncs, keeping the sync output focused.

> **On `cfg.embeddings.enabled`.** `vectors::run` itself respects the flag and logs `"embeddings disabled by workspace config; skipping vectors::run"` when off. No need to gate again at the sync call site.

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 68 tests pass. `tests/sync_unit.rs` is purely in-memory and unaffected; `tests/integration_sync.rs` exercises sync end-to-end but its assertions check file/function/edge counts only, not embeddings, so it remains green.

- [ ] **Step 3: Commit**

```bash
git add src/sync/mod.rs
git commit -m "feat(sync): re-embed touched symbols after resolution rebuild"
```

---

### Task 9: `codegraph-rs reembed` CLI command + integration test

**Files:**
- Create: `src/cli/reembed.rs`
- Modify: `src/cli/mod.rs`
- Modify: `tests/cli_smoke.rs`
- Create: `tests/integration_vectors.rs`

- [ ] **Step 1: Create src/cli/reembed.rs**

```rust
//! `codegraph-rs reembed` — re-embed all embed-eligible symbols.
//!
//! Use cases:
//! - Recovering from a `vectors::run` failure mid-batch (partial embeddings
//!   stay in the graph; `reembed` overwrites them).
//! - Model upgrade (future): when we eventually accept additional models in
//!   workspace config, switching the config value and running `reembed` will
//!   re-embed everything with the new model.
//!
//! Errors out (does NOT silently no-op) when `[embeddings] enabled = false`,
//! so users who explicitly invoke `reembed` get a clear signal that their
//! config disables it.

use anyhow::{Context, Result};
use std::path::Path;
use tracing::info;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::Connection;
use crate::vectors;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg
        .map(Path::to_path_buf)
        .unwrap_or_else(|| std::env::current_dir().expect("cwd readable"));
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

    if !cfg.embeddings.enabled {
        anyhow::bail!(
            "embeddings disabled in config; set `[embeddings] enabled = true` in \
             `.codegraph/config.toml` to use `codegraph-rs reembed`"
        );
    }

    info!("re-embedding workspace `{}`", cfg.workspace.name);

    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    let stats = rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        )
        .await
        .context("connecting to workspace database")?;

        vectors::run(&conn, &cfg, vectors::EmbedScope::All)
            .await
            .context("re-embedding workspace")
    })?;

    println!(
        "Re-embedded workspace `{}`: {} symbols, {} batches",
        cfg.workspace.name, stats.embedded, stats.batches,
    );
    Ok(())
}
```

> **On not applying schema here.** `reembed` is purely a writeback against existing nodes. `apply_schema` is `init`/`index`/`sync`'s responsibility. If a user manually dropped the vector index, `reembed` will still SET the property correctly; the index just won't be present until the next `index` or `sync`. That's a defensible recovery surface — no need to add latency to every `reembed`.

- [ ] **Step 2: Wire `Reembed` variant + dispatch in src/cli/mod.rs**

Add `pub mod reembed;` alongside the others (alphabetical).

Add the `Reembed` variant to the `Command` enum, just before `Serve`:

```rust
    /// Re-embed all embed-eligible symbols. Useful after a partial embedding
    /// failure or to refresh embeddings after a config change.
    Reembed,
```

Add the dispatch arm in `dispatch()`:

```rust
        Command::Reembed => reembed::run(cli.workspace.as_deref()),
```

- [ ] **Step 3: Update tests/cli_smoke.rs**

Add `reembed` to the subcommand list the help test iterates. Find the existing assertion that checks each subcommand appears in `--help` and add `"reembed"` to the list.

- [ ] **Step 4: Create tests/integration_vectors.rs**

```rust
//! Integration test for vectors: index tiny-rust against live Neo4j, then
//! assert that every embed-eligible symbol has an `embedding` of length 384,
//! non-embed-eligible nodes do NOT have embeddings, and `reembed` is idempotent.
//!
//! Gated on BOTH `CODEGRAPH_RS_TEST_NEO4J=1` and `CODEGRAPH_RS_TEST_EMBEDDINGS=1`.

use codegraph_rs::cli::{
    index,
    init::{self, InitArgs},
    reembed,
};
use neo4rs::query;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
        && std::env::var("CODEGRAPH_RS_TEST_EMBEDDINGS").as_deref() == Ok("1")
}

const EMBED_KINDS: &str =
    "['Function','Method','Class','Struct','Interface','Trait','Enum','TypeAlias','Module','Template','Choice']";

#[test]
fn embeddings_written_and_reembed_is_idempotent() {
    if !enabled() {
        eprintln!(
            "SKIP: set both CODEGRAPH_RS_TEST_NEO4J=1 and CODEGRAPH_RS_TEST_EMBEDDINGS=1 to enable"
        );
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("vectors-test-{}", std::process::id());

    let fixture_src = std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
        .join("tests/fixtures/tiny-rust");
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
    std::env::set_var("CODEGRAPH_RS_NEO4J_PASSWORD", &pw);

    index::run(Some(dir.path())).expect("index must succeed");

    let rt = tokio::runtime::Runtime::new().unwrap();
    let cfg = codegraph_rs::config::workspace::load_from_path(dir.path()).unwrap();
    let db_name = cfg.workspace.neo4j.database.clone();

    let eligible_count = rt.block_on(scalar_async(
        &pw, &db_name,
        &format!(
            "MATCH (s:Symbol) \
             WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS}) \
             RETURN count(s) AS c"
        ),
    ));
    assert!(
        eligible_count >= 6,
        "tiny-rust should have at least 6 embed-eligible symbols (got {eligible_count})"
    );

    let embedded = rt.block_on(scalar_async(
        &pw, &db_name,
        &format!(
            "MATCH (s:Symbol) \
             WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS}) \
               AND s.embedding IS NOT NULL \
             RETURN count(s) AS c"
        ),
    ));
    assert_eq!(
        embedded, eligible_count,
        "every embed-eligible symbol should have an embedding"
    );

    let dim = rt.block_on(scalar_async(
        &pw, &db_name,
        "MATCH (s:Symbol) WHERE s.embedding IS NOT NULL \
         RETURN size(s.embedding) AS c LIMIT 1",
    ));
    assert_eq!(dim, 384, "BAAI/bge-small-en-v1.5 should produce 384-dim vectors");

    let file_embeds = rt.block_on(scalar_async(
        &pw, &db_name,
        "MATCH (f:File) WHERE f.embedding IS NOT NULL RETURN count(f) AS c",
    ));
    assert_eq!(file_embeds, 0, "File nodes should not be embedded");

    reembed::run(Some(dir.path())).expect("reembed must succeed");

    let embedded_after = rt.block_on(scalar_async(
        &pw, &db_name,
        "MATCH (s:Symbol) WHERE s.embedding IS NOT NULL RETURN count(s) AS c",
    ));
    assert_eq!(
        embedded_after, embedded,
        "reembed should not change the set of embedded symbols"
    );

    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    rt.block_on(async {
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        sys.graph
            .execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await.unwrap().next().await.ok();
    });
}

async fn scalar_async(pw: &str, db_name: &str, cypher_text: &str) -> i64 {
    let conn = codegraph_rs::db::Connection::open(
        "bolt://localhost:7687", "neo4j", pw, db_name,
    ).await.unwrap();
    let mut stream = conn.graph.execute(query(cypher_text)).await.unwrap();
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

> **On the double gate.** Most slices have one gate (`CODEGRAPH_RS_TEST_NEO4J=1`). Slice 7 adds a second (`CODEGRAPH_RS_TEST_EMBEDDINGS=1`) because the fastembed model is expensive to download. Two patterns coexist in the repo after this slice: Neo4j-only integration tests and Neo4j+embeddings integration tests. Document both in README (Task 11).

- [ ] **Step 5: Verify SKIP path**

```bash
cargo test --test integration_vectors -- --nocapture 2>&1 | tail -10
```

Expected: 1 test, SKIPs (no env vars), test result is `ok`.

- [ ] **Step 6: Build + clippy + fmt + full test suite**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -10
```

Expected total: 75 tests (66 baseline + 6 compose + 2 cache_dir + 1 integration_vectors SKIP).

- [ ] **Step 7: Commit**

```bash
git add src/cli/reembed.rs src/cli/mod.rs tests/cli_smoke.rs tests/integration_vectors.rs
git commit -m "feat(cli): add reembed command and embeddings integration test"
```

---

### Task 10: Status coverage — `embedding_coverage` in `StatusSnapshot`

**Files:**
- Modify: `src/graph/status.rs`
- Modify: `src/cli/status.rs`

Add `embedding_coverage: EmbeddingCoverage { embedded: i64, total_eligible: i64 }` to `StatusSnapshot`, queried from the same DB the rest of `collect()` uses. Update `cli/status.rs` to print it.

- [ ] **Step 1: Edit src/graph/status.rs**

Add the new struct:

```rust
#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct EmbeddingCoverage {
    pub embedded: i64,
    pub total_eligible: i64,
}
```

Add the field to `StatusSnapshot`:

```rust
#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct StatusSnapshot {
    pub workspace: String,
    pub database: String,
    pub roots: Vec<String>,
    pub file_count: i64,
    pub symbol_count: i64,
    pub unresolved_count: i64,
    pub last_indexed_at: Option<i64>,
    pub embedding_coverage: EmbeddingCoverage,
}
```

Update `collect`:

```rust
pub async fn collect(conn: &Connection, cfg: &WorkspaceConfig) -> Result<StatusSnapshot> {
    let file_count = single_count(conn, "MATCH (n:File) RETURN count(n) AS c").await?;
    let symbol_count = single_count(conn, "MATCH (n:Symbol) RETURN count(n) AS c").await?;
    let unresolved_count =
        single_count(conn, "MATCH (n:UnresolvedRef) RETURN count(n) AS c").await?;
    let last_indexed_at = max_last_indexed(conn).await?;
    let embedding_coverage = embedding_coverage(conn).await?;

    Ok(StatusSnapshot {
        workspace: cfg.workspace.name.clone(),
        database: cfg.workspace.neo4j.database.clone(),
        roots: cfg.workspace.roots.iter().map(|r| r.name.clone()).collect(),
        file_count,
        symbol_count,
        unresolved_count,
        last_indexed_at,
        embedding_coverage,
    })
}
```

Add the helper:

```rust
const EMBED_KINDS_CYPHER_LIST: &str =
    "['Function','Method','Class','Struct','Interface','Trait','Enum',\
      'TypeAlias','Module','Template','Choice']";

async fn embedding_coverage(conn: &Connection) -> Result<EmbeddingCoverage> {
    let total_eligible = single_count(
        conn,
        &format!(
            "MATCH (s:Symbol) \
             WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS_CYPHER_LIST}) \
             RETURN count(s) AS c"
        ),
    )
    .await?;
    let embedded = single_count(
        conn,
        &format!(
            "MATCH (s:Symbol) \
             WHERE any(k IN labels(s) WHERE k IN {EMBED_KINDS_CYPHER_LIST}) \
               AND s.embedding IS NOT NULL \
             RETURN count(s) AS c"
        ),
    )
    .await?;
    Ok(EmbeddingCoverage {
        embedded,
        total_eligible,
    })
}
```

> **DRY: pulling `EMBED_KINDS_CYPHER_LIST` into shared scope.** Both `src/vectors/query.rs` and `src/graph/status.rs` now hardcode the same kind list. Acceptable duplication for v1; a future cleanup pulls it into `src/extraction/result.rs` as a shared constant. Tracked in the slice-done follow-ups list.

- [ ] **Step 2: Edit src/cli/status.rs**

Add an `Embeddings:` line to the println block, between `Unresolved refs:` and `Last indexed:`:

```rust
        println!(
            "Embeddings: {} / {} eligible symbols",
            snap.embedding_coverage.embedded,
            snap.embedding_coverage.total_eligible
        );
```

- [ ] **Step 3: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -10
```

Expected: 75 tests pass. The new field on `StatusSnapshot` is additive; existing serializers/consumers (MCP `codegraph_rs_status`, integration_mcp test) gain a new key and don't break.

- [ ] **Step 4: Commit**

```bash
git add src/graph/status.rs src/cli/status.rs
git commit -m "feat(status): report embedding coverage in status snapshot"
```

---

### Task 11: README + slice-7-embeddings tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update README Status section**

Replace the current Status block:

```markdown
## Status

In development. Currently implements **Slice 7: Vector Embeddings** —
workspace initialization, indexing, resolution, incremental sync, a
stdio MCP server exposing 7 graph-traversal tools, and on-write
generation of 384-dim `BAAI/bge-small-en-v1.5` embeddings (via
`fastembed-rs`) for every embed-eligible symbol. Semantic-search MCP
tooling and Daml language support land in subsequent slices.

The embedding model is downloaded on first use to
`~/.cache/codegraph-rs/fastembed/` (~130 MB; one-time cost). Set
`[embeddings] enabled = false` in `.codegraph/config.toml` to opt out
of automatic embedding during `index` / `sync`.
```

Update Quick start with `reembed`:

```markdown
codegraph-rs reembed  # re-embed all symbols (e.g. after a failed embed step)
```

Add the second test gate to Development:

```markdown
CODEGRAPH_RS_TEST_NEO4J=1 \
CODEGRAPH_RS_TEST_EMBEDDINGS=1 \
CODEGRAPH_RS_TEST_NEO4J_PASSWORD='...' \
  cargo test -- --test-threads=1   # full suite, including the embeddings
                                   # integration test (downloads ~130 MB
                                   # model on first run)
```

- [ ] **Step 2: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected breakdown:

| Test binary | Count |
|---|---|
| `extraction_unit` | 31 |
| `resolution_unit` | 2 |
| `sync_unit` | 6 |
| `graph_unit` | 4 |
| `vectors_unit` (new) | 8 |
| `config_parsing` | 13 |
| `cli_smoke` | 2 |
| `integration_db` (SKIP) | 2 |
| `integration_init` (SKIP) | 1 |
| `integration_status` (SKIP) | 1 |
| `integration_index` (SKIP) | 1 |
| `integration_resolve` (SKIP) | 1 |
| `integration_sync` (SKIP) | 1 |
| `integration_mcp` (SKIP) | 1 |
| `integration_vectors` (new, SKIP) | 1 |
| **Total** | **75** |

- [ ] **Step 3: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 7"
git tag slice-7-embeddings
```

- [ ] **Step 4: Verify**

```bash
git log --oneline | head -15
git tag --list
```

Expected tags: `slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`, `slice-6-mcp`, `slice-7-embeddings`.

---

## Slice 7 Done

- [ ] `cargo build` clean (first build downloads `fastembed` + `ort` + transitive deps; ~3-5 min on first invocation)
- [ ] `cargo test` (no env vars) — 75 tests pass, all integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all 75 pass; `integration_vectors` still SKIPs (needs the second gate)
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 CODEGRAPH_RS_TEST_EMBEDDINGS=1 cargo test -- --test-threads=1` — all 75 pass, including `integration_vectors` (downloads model on first run, subsequent runs hit cache)
- [ ] Manual: `codegraph-rs index` on a fresh workspace prints `"Indexed N files: ... K embeddings (M batches)"` and the model download progress bar (first time only)
- [ ] Manual: `codegraph-rs status` includes an `"Embeddings: K / N eligible symbols"` line
- [ ] Manual: edit a Rust file (change a function's signature), run `codegraph-rs sync` — output mentions re-embedding only the touched symbols (`info` log line: `"sync re-embedded N symbols across M batches"`)
- [ ] Manual: `codegraph-rs reembed` re-embeds all eligible symbols; repeated invocation is idempotent
- [ ] Manual: flip `[embeddings] enabled = false` in `.codegraph/config.toml`; `index` and `sync` log `"embeddings disabled by workspace config; skipping vectors::run"` and produce 0 embeddings; `reembed` exits with the clear error message
- [ ] Neo4j Browser: `MATCH (s:Symbol) WHERE s.embedding IS NOT NULL RETURN size(s.embedding), labels(s) LIMIT 5` returns 384-dim vectors and labels like `[Symbol, Function]`, `[Symbol, Struct]`, etc.
- [ ] Neo4j Browser: `SHOW INDEXES YIELD name, state WHERE name = 'symbol_embedding' RETURN state` reports `ONLINE`
- [ ] No semantic-search MCP tool yet — `codegraph_rs_semantic_search` is the headline feature of the next slice

When all check, slice 7 is complete. **Next slice: semantic search + context** — add `codegraph_rs_semantic_search` (uses `vectors::model::load_model` to embed the query, then `CALL db.index.vector.queryNodes('symbol_embedding', $k, $queryVec)` to retrieve), plus `codegraph_rs_context` and `codegraph_rs_explore` which compose semantic hits with the existing graph-traversal tools.

### Small follow-ups noted during this slice

- **Dedupe `EMBED_KINDS` constant.** `src/vectors/query.rs` and `src/graph/status.rs` each carry a copy of the same kind list. A future cleanup pulls it into `src/extraction/result.rs` (alongside `NodeKind::as_label`) as `pub const EMBED_ELIGIBLE_LABELS: &[&str] = &[...]`, exposed via a helper `is_embed_eligible(label: &str) -> bool`. Not blocking for slice 7.
- **`embed_batch` blocking on the async runtime.** Fine for slice 7's single-call use cases, but the next slice's `serve`-time query embedding will need `tokio::task::spawn_blocking` to avoid stalling MCP handlers during model inference.
