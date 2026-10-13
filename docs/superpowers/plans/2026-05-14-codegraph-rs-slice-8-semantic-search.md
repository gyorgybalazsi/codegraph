# codegraph-rs Slice 8 — Semantic Search + Context + Explore

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the embeddings written by Slice 7 actually queryable. Add three MCP tools — `codegraph_rs_semantic_search`, `codegraph_rs_context`, and `codegraph_rs_explore` — backed by Neo4j's `db.index.vector.queryNodes` against the `symbol_embedding` vector index (already populated since Slice 7). The query text is embedded with the same `BAAI/bge-small-en-v1.5` model used at index time; the model is loaded once at `serve` startup and reused across tool invocations behind a `parking_lot::Mutex`. `tokio::task::spawn_blocking` keeps query-time inference off the async runtime. After this slice, a Claude Code session can discover seed symbols by topic, drill into them with `_node`, and expand the neighborhood with `_callers` / `_callees` / `_impact` / `_context` / `_explore` — closing the full tool surface from spec §10 except the Daml-only tools.

**Architecture:** Three thin MCP adapters in `src/mcp/mod.rs` calling three thin `graph::*` modules: `graph/semantic.rs` (one Cypher round-trip), `graph/context.rs` (semantic + 1-hop neighborhoods, composed in Rust), `graph/explore.rs` (semantic seeds + bounded BFS, one Cypher round-trip per scope). The MCP server gains an `embedding_model: Option<Arc<parking_lot::Mutex<TextEmbedding>>>` field, loaded by `cli::serve::run` before constructing `McpServer`. Query embedding runs inside `tokio::task::spawn_blocking` so the rmcp event loop keeps serving other requests during the ~50ms inference.

```
McpServer { conn, cfg, workspace_dir, embedding_model } (NEW field)
        │
        ├── semantic_search(query, root?, limit?) ─► embed query ─► CALL db.index.vector.queryNodes
        ├── context(task, limit?)               ─► semantic_search ─► for each hit: node + 1-hop callers/callees
        └── explore(topic, depth?)              ─► semantic_search (k=3) ─► BFS CALLS|REFERENCES|TYPE_OF, depth-bounded
```

**Tech Stack additions:**
- `parking_lot = "0.12"` — fast non-poisoning sync `Mutex`. Used to guard `TextEmbedding` (which is `Send` but `embed()` requires `&mut self`). Adds ~30 KB; no transitive ML deps. We never need `MutexGuard` to cross an `.await`, so the sync flavor is correct.

No new fastembed/rmcp/neo4rs dependencies — every query-side primitive is already in tree (Slice 7 added fastembed; Slice 6 added rmcp).

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §8 "Search" (the `CALL db.index.vector.queryNodes` Cypher pattern, line ~600), §10 "Tool surface" rows for `semantic_search`, `context`, `explore` (lines ~724/730/731), §10 "Lifecycle" ("preload the `fastembed-rs` model" on startup).

**Slice 7 baseline:** HEAD `5edbb3f` on `main`, tag `slice-7-embeddings` at `9e2ccca`. 75 tests pass.

---

## What this slice does, conceptually

1. **One model, loaded once.** `cli::serve::run` calls `vectors::model::load_model(&default_cache_dir())` before opening the rmcp transport. The returned `TextEmbedding` is wrapped in `Arc<parking_lot::Mutex<TextEmbedding>>` and passed into `McpServer::new`. First-time `serve` startup downloads ~130 MB (with progress to stderr); subsequent startups load from disk in <500 ms.

2. **Query embedding off the runtime.** Each `semantic_search` (directly or via `context`/`explore`) calls a small helper:
   ```rust
   pub async fn embed_query(model: &Arc<parking_lot::Mutex<TextEmbedding>>, text: String) -> Result<Vec<f32>>
   ```
   The body uses `tokio::task::spawn_blocking(move || { let mut m = model.lock(); m.embed(vec![text], None)? })`. The closure owns an `Arc` clone; the lock is held only for the duration of the synchronous `embed` call.

3. **Cypher query.** `graph::semantic::search(conn, &qvec, root, limit)` runs the spec §8 query, asking the index for `k = limit * 3` (or 30 if no root) so a root filter can shrink the post-filter set without starving:
   ```cypher
   CALL db.index.vector.queryNodes('symbol_embedding', $k, $vec)
     YIELD node, score
   WHERE $root IS NULL OR node.root_name = $root
   RETURN node.qid AS qid, node.name AS name, node.qualified_name AS qualified_name,
          [l IN labels(node) WHERE l <> 'Symbol'][0] AS kind,
          node.root_name AS root_name, node.relative_path AS relative_path, score
   ORDER BY score DESC
   LIMIT $limit
   ```

4. **`context` = semantic hits + 1-hop neighborhoods.** Reuses `graph::semantic::search`, then for each hit fetches a `NodeResponse` (same as `_node` tool) plus 1-hop callers and 1-hop callees via the existing `graph::traverse` functions. Composed in Rust; one `Vec<ContextBundle>` returned.

5. **`explore` = semantic seeds + multi-hop BFS.** Reuses `graph::semantic::search` with hard-coded `limit = 3` for seeds, then issues ONE Cypher per seed scope that traverses `CALLS|REFERENCES|TYPE_OF` outward up to `depth` hops, returning `{qid, name, kind, distance_from_nearest_seed}` rows. Seeds and reachable nodes are deduped in Rust.

6. **Failure-mode hygiene.** If `cfg.embeddings.enabled = false`, `cli::serve::run` skips model load entirely. The three new tools then fail with a stable error: `"embeddings disabled in this workspace; enable in .codegraph/config.toml and run codegraph-rs reembed"`. Slice 6's seven graph tools continue to work — semantic tools are an additive layer.

7. **Integration test parity.** A new `tests/integration_semantic.rs` exercises all three tools end-to-end against tiny-rust, gated on BOTH `CODEGRAPH_RS_TEST_NEO4J=1` and `CODEGRAPH_RS_TEST_EMBEDDINGS=1` (same double gate Slice 7 introduced for `integration_vectors.rs`). Assertions are structural (correct shapes, dim 384, finite scores, top-hit name in a plausible set) rather than rank-exact, since tiny-rust has only 7 embed-eligible symbols.

---

## Out of scope for Slice 8

- Daml language support — separate slice; spec §6 / §8 lists `Template` and `Choice` as embed-eligible kinds the orchestrator already names, but the extractor for them lands later.
- Cross-encoder reranking — spec §8 "Not in v1".
- Streaming responses, auth, multi-client — `serve` remains a single-client stdio server.
- Schema changes — the `symbol_embedding` vector index already exists (Slice 2 created it; Slice 7 populated it).
- Pagination — semantic results return `limit` rows; no cursor.
- Per-query model swap — `embeddings.model` is locked to `bge-small-en-v1.5` (validated in `src/config/workspace.rs:114-119`).
- Embedding the user query body separately from a single embedding call. `TextEmbedding::embed(vec![query], None)` is one synchronous call returning one 384-dim vector; same surface as the index-time path.
- Background warm-up of the model after `serve` startup — load is synchronous and blocking inside `cli::serve::run`. If a future slice wants non-blocking startup, it can wrap the load in `tokio::task::spawn_blocking` *and* serve a "model warming, please retry" error for tool calls that race the warmup. Out of v1.

---

## Critical design decisions

### D1. Where the model loads — `cli::serve::run`, not `mcp::serve` or lazily.

Decision: **Load in `cli::serve::run`, pass `Option<Arc<parking_lot::Mutex<TextEmbedding>>>` into `mcp::serve` and onward to `McpServer::new`.** Justifications:

1. Spec §10 "Lifecycle" mandates preload on startup.
2. CLI layer owns user-facing startup messaging (tracing logs, stderr download progress).
3. `mcp::serve` keeps its responsibility narrow: transport + handler loop.
4. `cfg.embeddings.enabled = false` short-circuits cleanly at the CLI layer: pass `None`.

### D2. Disabled embeddings — all three tools error consistently.

Decision: **All three semantic tools return `ErrorData::invalid_params(...)` with a stable, actionable message.** Justifications:

1. FTS fallback for `context`/`explore` would silently change semantics.
2. Slice 6's graph tools (search, node, callers, callees, impact) already cover the no-embedding path.
3. Hiding tools dynamically from `list_tools` requires runtime introspection of cfg, which rmcp's static `#[tool_router]` doesn't accommodate.

Error string (one helper produces it for all three tools):
```
"embeddings disabled in this workspace; enable `[embeddings] enabled = true` in `.codegraph/config.toml` and run `codegraph-rs reembed` to populate embeddings"
```

### D3. `context` tool shape — flat array, with source snippet.

Decision: **`ContextResponse { entries: Vec<ContextEntry> }` where each entry has `node: NodeDetail`, `source: String`, `score: f64`, `callers: Vec<Neighbor>`, `callees: Vec<Neighbor>` (1-hop only).** Justifications:

1. Flat array is immediately iterable for the consumer (Claude Code).
2. Source snippet bundled inline so the consumer doesn't have to chase N follow-up `_node` calls.
3. Reuses existing serializable types (no new schema work).

### D4. `explore` tool shape — merged reachable set with `distance_from_nearest_seed`.

Decision: **`ExploreResponse { seeds: Vec<SeedNode>, reachable: Vec<ReachableNode> }` where `ReachableNode` carries `distance_from_nearest_seed` and each reachable qid appears once.** Justifications:

1. Topic exploration wants "what's around this topic?", not "what's around each seed independently?". Min-distance over all seeds is the right signal.
2. Smaller wire payload than per-seed denormalization.
3. Per-edge listing (option c) duplicates what `_callers`/`_callees`/`_impact` already do for single qids.

Seeds carry full `NodeDetail` + source snippet (same as `context` entries minus 1-hop neighborhood). Reachable nodes carry only `{qid, name, kind, distance}`.

### D5. Mutex flavor — `parking_lot::Mutex`, never crossing `.await`.

Decision: **`parking_lot::Mutex<TextEmbedding>` inside `tokio::task::spawn_blocking`.** Justifications:

1. The guard is dropped before `spawn_blocking` returns; never crosses an `.await`.
2. `parking_lot` doesn't poison on panic — a malformed input doesn't break subsequent tool calls.
3. Faster than `std::sync::Mutex` (negligible at our scale but free).

Helper signature:
```rust
async fn embed_query(
    model: &Arc<parking_lot::Mutex<fastembed::TextEmbedding>>,
    text: String,
) -> Result<Vec<f32>>
```

Caller passes `text: String` (owned) to satisfy `spawn_blocking`'s `'static + Send` bound.

### D6. Vector-index `$k` — `max(limit * 3, 30)`, post-filter, then `LIMIT`.

`db.index.vector.queryNodes` doesn't take property predicates; `WHERE node.root_name = $root` runs *after* the top-K. Asking for `$k = limit` and then root-filtering can starve.

Decision: **`k = max(limit * 3, 30) as i64`.** Justifications:

1. 3x multiplier covers single-root filters where root has ≥30% of the workspace.
2. Floor of 30 handles small-`limit` calls cheaply.
3. Adaptive retry (option c) is over-engineering for v1; switch to it later if real users hit the starvation case.

Documented in `graph::semantic::search` rustdoc.

### D7. Test fixture — structural assertions only.

tiny-rust has 7 embed-eligible symbols; cosine ranking is near-uniform. Assert structural properties (correct shapes, finite scores, valid kind labels, distance in `[1, depth]`), not exact rankings. The "embeddings exist and are 384-dim" invariant lives in `tests/integration_vectors.rs` (Slice 7).

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml                          # add parking_lot = "0.12"
  src/
    cli/
      serve.rs                        # MODIFY: load model, pass into mcp::serve
    graph/
      mod.rs                          # MODIFY: pub mod semantic; pub mod context; pub mod explore;
      semantic.rs                     # NEW: db.index.vector.queryNodes wrapper
      context.rs                      # NEW: semantic + 1-hop bundle composer
      explore.rs                      # NEW: semantic seeds + BFS reachable composer
    mcp/
      mod.rs                          # MODIFY: McpServer field + 3 new tools + embed_query helper
  tests/
    integration_mcp.rs                # MODIFY: pass None for new embedding_model arg
    integration_semantic.rs           # NEW: gated; exercises all 3 new tools end-to-end
  README.md                           # MODIFY: bump Status to Slice 8
```

---

### Task 1: Dependency + module skeleton

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/graph/mod.rs`
- Create: `src/graph/semantic.rs` (stub)
- Create: `src/graph/context.rs` (stub)
- Create: `src/graph/explore.rs` (stub)

- [ ] **Step 1: Add `parking_lot` to Cargo.toml**

Append to `[dependencies]`:

```toml
parking_lot = "0.12"
```

- [ ] **Step 2: Update `src/graph/mod.rs`**

Replace body with:

```rust
//! Read-only graph queries — typed wrappers over Cypher.
//!
//! Each submodule defines a small set of pure-data response structs and a single
//! `pub async fn` that takes a `&Connection` (and parameters) and returns those
//! structs. No Cypher leaks beyond this module.

pub mod context;
pub mod explore;
pub mod files;
pub mod node;
pub mod search;
pub mod semantic;
pub mod source;
pub mod status;
pub mod traverse;
```

- [ ] **Step 3: Create stub `src/graph/semantic.rs`**

```rust
//! Semantic search over the `symbol_embedding` Neo4j vector index. Filled in by Task 2.
```

- [ ] **Step 4: Create stub `src/graph/context.rs`**

```rust
//! Task-oriented context bundler: semantic hits + 1-hop neighborhoods. Filled in by Task 3.
```

- [ ] **Step 5: Create stub `src/graph/explore.rs`**

```rust
//! Topic exploration: semantic seeds + multi-hop reachable set. Filled in by Task 4.
```

- [ ] **Step 6: Build, clippy, fmt, test**

```bash
cargo build 2>&1 | tail -5
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: clean build (`parking_lot` is small, ~5s incremental). 75 tests still pass.

- [ ] **Step 7: Commit**

```bash
git add Cargo.toml Cargo.lock src/graph/mod.rs src/graph/semantic.rs src/graph/context.rs src/graph/explore.rs
git commit -m "chore(graph): add parking_lot dep and semantic/context/explore stubs"
```

---

### Task 2: `graph::semantic::search`

**Files:**
- Modify: `src/graph/semantic.rs`

Implements one Cypher round-trip against `db.index.vector.queryNodes` with the optional `root` filter. Returns `Vec<SemanticHit>` shaped like `SearchHit` from `graph::search`.

- [ ] **Step 1: Implement `src/graph/semantic.rs`**

```rust
//! Semantic search over the `symbol_embedding` Neo4j vector index.
//!
//! One Cypher round-trip. Per spec §8 ("Search"), uses
//! `CALL db.index.vector.queryNodes('symbol_embedding', $k, $vec)` and
//! optionally filters by `root_name`. The caller is responsible for embedding
//! the query text with `BAAI/bge-small-en-v1.5` (the same model used at
//! index time) — `search` only deals with the resulting vector.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;
use serde_json::json;

use crate::db::write::to_bolt;
use crate::db::Connection;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct SemanticHit {
    pub qid: String,
    pub name: String,
    pub kind: String,
    pub qualified_name: Option<String>,
    pub root_name: String,
    pub relative_path: String,
    pub score: f64,
}

/// Run a vector-index lookup. `qvec` is a 384-dim embedding of the user query
/// text; `root` optionally filters to a single workspace root; `limit` caps
/// the post-filter result count.
///
/// **Index `$k`:** asks Neo4j for `max(limit * 3, 30)` candidates, then applies
/// the `root` filter and trims to `limit`. See plan D6.
pub async fn search(
    conn: &Connection,
    qvec: &[f32],
    root: Option<&str>,
    limit: i64,
) -> Result<Vec<SemanticHit>> {
    let k: i64 = std::cmp::max(limit * 3, 30);

    let cypher = "\
        CALL db.index.vector.queryNodes('symbol_embedding', $k, $vec) \
          YIELD node, score \
        WHERE $root IS NULL OR node.root_name = $root \
        RETURN node.qid AS qid, node.name AS name, \
               node.qualified_name AS qualified_name, \
               [l IN labels(node) WHERE l <> 'Symbol'][0] AS kind, \
               node.root_name AS root_name, \
               node.relative_path AS relative_path, \
               score \
        ORDER BY score DESC \
        LIMIT $limit";

    let vec_json = serde_json::Value::Array(
        qvec.iter()
            .map(|x| {
                serde_json::Number::from_f64(f64::from(*x))
                    .map(serde_json::Value::Number)
                    .ok_or_else(|| anyhow::anyhow!("non-finite float in query embedding"))
            })
            .collect::<Result<Vec<_>>>()?,
    );
    let vec_param = to_bolt(vec_json).context("encoding query vector")?;
    let root_param = to_bolt(json!(root)).context("encoding `root` param")?;

    let mut stream = conn
        .graph
        .execute(
            query(cypher)
                .param("k", k)
                .param("vec", vec_param)
                .param("root", root_param)
                .param("limit", limit),
        )
        .await
        .context("running semantic_search")?;

    let mut out = Vec::new();
    while let Some(row) = stream.next().await? {
        out.push(SemanticHit {
            qid: row.get("qid").context("row missing qid")?,
            name: row.get("name").context("row missing name")?,
            kind: row.get::<Option<String>>("kind")?.unwrap_or_default(),
            qualified_name: row.get::<Option<String>>("qualified_name")?,
            root_name: row.get("root_name").context("row missing root_name")?,
            relative_path: row
                .get("relative_path")
                .context("row missing relative_path")?,
            score: row.get("score").context("row missing score")?,
        });
    }
    Ok(out)
}
```

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 75 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/graph/semantic.rs
git commit -m "feat(graph): semantic search via db.index.vector.queryNodes"
```

---

### Task 3: `graph::context::bundle`

**Files:**
- Modify: `src/graph/context.rs`

Composes semantic hits + per-hit `NodeDetail` + source snippet + 1-hop callers and 1-hop callees.

- [ ] **Step 1: Implement `src/graph/context.rs`**

```rust
//! Task-oriented context bundle: semantic search hits paired with their
//! 1-hop neighborhoods (callers and callees) and source snippets.

use anyhow::{Context, Result};
use serde::Serialize;
use std::path::Path;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;
use crate::graph::{node, semantic, traverse};

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct ContextEntry {
    pub node: node::NodeDetail,
    pub source: String,
    pub score: f64,
    pub callers: Vec<traverse::Neighbor>,
    pub callees: Vec<traverse::Neighbor>,
}

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct ContextResponse {
    pub entries: Vec<ContextEntry>,
}

/// Build a context bundle from a pre-computed query embedding.
pub async fn bundle(
    conn: &Connection,
    workspace_dir: &Path,
    cfg: &WorkspaceConfig,
    qvec: &[f32],
    limit: i64,
) -> Result<ContextResponse> {
    let hits = semantic::search(conn, qvec, None, limit)
        .await
        .context("semantic search for context bundle")?;

    let mut entries = Vec::with_capacity(hits.len());
    for hit in hits {
        let detail = node::find(conn, workspace_dir, cfg, &hit.qid)
            .await
            .with_context(|| format!("looking up node `{}` for context", hit.qid))?;
        let callers = traverse::callers(conn, &hit.qid, 1)
            .await
            .with_context(|| format!("1-hop callers for `{}`", hit.qid))?;
        let callees = traverse::callees(conn, &hit.qid, 1)
            .await
            .with_context(|| format!("1-hop callees for `{}`", hit.qid))?;

        entries.push(ContextEntry {
            node: detail.node,
            source: detail.source,
            score: hit.score,
            callers,
            callees,
        });
    }
    Ok(ContextResponse { entries })
}
```

> **Sequential N+1 round-trips.** N=5 default × 3 round-trips = ~15 Cypher calls, ~15 ms total on local Neo4j. Negligible vs. ~50 ms query embedding. Parallelize with `try_join_all` later if `limit` trends > 20.

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 75 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/graph/context.rs
git commit -m "feat(graph): context bundle composer — semantic hits + 1-hop neighborhood"
```

---

### Task 4: `graph::explore::expand`

**Files:**
- Modify: `src/graph/explore.rs`

Composes top-3 semantic seeds (full detail + snippet) + ONE Cypher that BFS-expands across `CALLS|REFERENCES|TYPE_OF` up to `depth` hops, returning `distance_from_nearest_seed` per reachable qid.

- [ ] **Step 1: Implement `src/graph/explore.rs`**

```rust
//! Topic exploration: semantic seeds + multi-hop reachable set.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;
use serde_json::json;
use std::path::Path;

use crate::config::workspace::WorkspaceConfig;
use crate::db::write::to_bolt;
use crate::db::Connection;
use crate::graph::{node, semantic};

const NUM_SEEDS: i64 = 3;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct SeedNode {
    pub node: node::NodeDetail,
    pub source: String,
    pub score: f64,
}

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct ReachableNode {
    pub qid: String,
    pub name: String,
    pub kind: String,
    pub distance_from_nearest_seed: i64,
}

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct ExploreResponse {
    pub seeds: Vec<SeedNode>,
    pub reachable: Vec<ReachableNode>,
}

/// Build an exploration response from a pre-computed query embedding.
pub async fn expand(
    conn: &Connection,
    workspace_dir: &Path,
    cfg: &WorkspaceConfig,
    qvec: &[f32],
    depth: i64,
) -> Result<ExploreResponse> {
    let hits = semantic::search(conn, qvec, None, NUM_SEEDS)
        .await
        .context("semantic search for explore seeds")?;

    let mut seeds = Vec::with_capacity(hits.len());
    let mut seed_qids: Vec<String> = Vec::with_capacity(hits.len());
    for hit in &hits {
        let detail = node::find(conn, workspace_dir, cfg, &hit.qid)
            .await
            .with_context(|| format!("looking up seed `{}`", hit.qid))?;
        seeds.push(SeedNode {
            node: detail.node,
            source: detail.source,
            score: hit.score,
        });
        seed_qids.push(hit.qid.clone());
    }

    if seed_qids.is_empty() {
        return Ok(ExploreResponse {
            seeds,
            reachable: Vec::new(),
        });
    }

    let cypher = "\
        UNWIND $seed_qids AS sqid \
        MATCH path = (seed:Symbol {qid: sqid})-[:CALLS|REFERENCES|TYPE_OF*1..]->(r:Symbol) \
        WHERE length(path) <= $depth AND NOT r.qid IN $seed_qids \
        WITH r, min(length(path)) AS distance \
        RETURN r.qid AS qid, r.name AS name, \
               [l IN labels(r) WHERE l <> 'Symbol'][0] AS kind, distance \
        ORDER BY distance ASC, name ASC";

    let seed_qids_param = to_bolt(json!(seed_qids)).context("encoding seed qids")?;

    let mut stream = conn
        .graph
        .execute(
            query(cypher)
                .param("seed_qids", seed_qids_param)
                .param("depth", depth),
        )
        .await
        .context("running explore BFS")?;

    let mut reachable = Vec::new();
    while let Some(row) = stream.next().await? {
        reachable.push(ReachableNode {
            qid: row.get("qid").context("row missing qid")?,
            name: row.get("name").context("row missing name")?,
            kind: row.get::<Option<String>>("kind")?.unwrap_or_default(),
            distance_from_nearest_seed: row.get("distance").context("row missing distance")?,
        });
    }

    Ok(ExploreResponse { seeds, reachable })
}
```

- [ ] **Step 2: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 75 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/graph/explore.rs
git commit -m "feat(graph): topic exploration — semantic seeds + BFS reachable"
```

---

### Task 5: `McpServer` gains `embedding_model`; load it in `cli::serve::run`

**Files:**
- Modify: `src/mcp/mod.rs`
- Modify: `src/cli/serve.rs`
- Modify: `tests/integration_mcp.rs`

- [ ] **Step 1: Update `src/mcp/mod.rs`**

Add imports near the top:

```rust
use parking_lot::Mutex;
use fastembed::TextEmbedding;
```

Update `Inner` and `McpServer::new`:

```rust
struct Inner {
    conn: Connection,
    cfg: WorkspaceConfig,
    workspace_dir: PathBuf,
    /// `None` when `cfg.embeddings.enabled = false`. Semantic tools return
    /// the "embeddings disabled" error in that case.
    embedding_model: Option<Arc<Mutex<TextEmbedding>>>,
}

impl McpServer {
    pub fn new(
        conn: Connection,
        cfg: WorkspaceConfig,
        workspace_dir: PathBuf,
        embedding_model: Option<Arc<Mutex<TextEmbedding>>>,
    ) -> Self {
        Self {
            inner: Arc::new(Inner {
                conn,
                cfg,
                workspace_dir,
                embedding_model,
            }),
        }
    }
}
```

Update `pub async fn serve` signature to accept `embedding_model: Option<Arc<Mutex<TextEmbedding>>>` and pass it through to `McpServer::new`.

- [ ] **Step 2: Update `src/cli/serve.rs`**

Add the model-load step before `rt.block_on`:

```rust
pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg
        .map(Path::to_path_buf)
        .unwrap_or_else(|| std::env::current_dir().expect("cwd readable"));
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

    // Spec §10 "Lifecycle": preload the fastembed-rs model on startup.
    let embedding_model = if cfg.embeddings.enabled {
        let cache = crate::vectors::model::default_cache_dir();
        let model = crate::vectors::model::load_model(&cache)
            .context("loading embedding model for serve")?;
        Some(std::sync::Arc::new(parking_lot::Mutex::new(model)))
    } else {
        info!("embeddings disabled; semantic tools will return a disabled-config error");
        None
    };

    let rt = tokio::runtime::Runtime::new().context("starting tokio runtime")?;
    rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        )
        .await
        .context("connecting to workspace database")?;

        info!(
            "MCP serve: workspace `{}`, db `{}`, embeddings={}",
            cfg.workspace.name,
            cfg.workspace.neo4j.database,
            embedding_model.is_some()
        );
        mcp::serve(conn, cfg, workspace_dir, embedding_model).await
    })
}
```

- [ ] **Step 3: Update `tests/integration_mcp.rs`**

Find `McpServer::new(conn, cfg, dir.path().to_path_buf())` and add a fourth `None` argument:

```rust
let server = McpServer::new(conn, cfg, dir.path().to_path_buf(), None);
```

The test exercises only the seven graph tools; `None` is correct.

- [ ] **Step 4: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -5
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 75 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/mcp/mod.rs src/cli/serve.rs tests/integration_mcp.rs
git commit -m "feat(mcp): load embedding model at serve startup; thread through McpServer"
```

---

### Task 6: Wire the three new MCP tools

**Files:**
- Modify: `src/mcp/mod.rs`

- [ ] **Step 1: Add `embeddings_disabled_err` and `embed_query` helpers**

Append at module scope (near `internal_err`):

```rust
fn embeddings_disabled_err() -> ErrorData {
    ErrorData::invalid_params(
        "embeddings disabled in this workspace; enable `[embeddings] enabled = true` \
         in `.codegraph/config.toml` and run `codegraph-rs reembed` to populate embeddings",
        None,
    )
}

/// Embed a query string off the async runtime. Holds the parking_lot mutex
/// only inside the spawn_blocking closure; the guard never crosses an `.await`.
async fn embed_query(
    model: &Arc<parking_lot::Mutex<fastembed::TextEmbedding>>,
    text: String,
) -> Result<Vec<f32>> {
    let model = Arc::clone(model);
    tokio::task::spawn_blocking(move || {
        let mut m = model.lock();
        let mut vecs = m
            .embed(vec![text], None)
            .map_err(|e| anyhow::anyhow!("fastembed embed failed: {e}"))?;
        vecs.pop()
            .ok_or_else(|| anyhow::anyhow!("fastembed returned no vectors for query"))
    })
    .await
    .context("spawn_blocking join")?
}
```

> `Result` here means `anyhow::Result`. Add `use anyhow::{Context, Result};` if not already imported (it likely is).

- [ ] **Step 2: Add `Params` and `Response` structs**

```rust
#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct SemanticSearchParams {
    pub query: String,
    pub root: Option<String>,
    pub limit: Option<i64>,
}

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct SemanticSearchResponse {
    pub results: Vec<graph::semantic::SemanticHit>,
}

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct ContextParams {
    pub task: String,
    pub limit: Option<i64>,
}

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct ExploreParams {
    pub topic: String,
    pub depth: Option<i64>,
}
```

(`ContextResponse` and `ExploreResponse` live in `graph::context` / `graph::explore` — serialize them directly via `Json<...>`.)

- [ ] **Step 3: Add the three tool handlers**

Inside the `#[tool_router(server_handler)] impl McpServer { ... }` block, after `impact`:

```rust
    #[tool(
        name = "codegraph_rs_semantic_search",
        description = "Search symbols by topic/intent via the `BAAI/bge-small-en-v1.5` embedding model and Neo4j's `symbol_embedding` vector index. Returns top-k hits ordered by cosine similarity."
    )]
    async fn semantic_search(
        &self,
        Parameters(p): Parameters<SemanticSearchParams>,
    ) -> Result<Json<SemanticSearchResponse>, ErrorData> {
        let model = self
            .inner
            .embedding_model
            .as_ref()
            .ok_or_else(embeddings_disabled_err)?;
        let qvec = embed_query(model, p.query).await.map_err(internal_err)?;
        graph::semantic::search(
            &self.inner.conn,
            &qvec,
            p.root.as_deref(),
            p.limit.unwrap_or(10),
        )
        .await
        .map(|results| Json(SemanticSearchResponse { results }))
        .map_err(internal_err)
    }

    #[tool(
        name = "codegraph_rs_context",
        description = "Build a task-oriented context bundle: top-k semantic hits paired with their 1-hop callers/callees and source snippets. `limit` defaults to 5."
    )]
    async fn context(
        &self,
        Parameters(p): Parameters<ContextParams>,
    ) -> Result<Json<graph::context::ContextResponse>, ErrorData> {
        let model = self
            .inner
            .embedding_model
            .as_ref()
            .ok_or_else(embeddings_disabled_err)?;
        let qvec = embed_query(model, p.task).await.map_err(internal_err)?;
        graph::context::bundle(
            &self.inner.conn,
            &self.inner.workspace_dir,
            &self.inner.cfg,
            &qvec,
            p.limit.unwrap_or(5),
        )
        .await
        .map(Json)
        .map_err(internal_err)
    }

    #[tool(
        name = "codegraph_rs_explore",
        description = "Topic-driven graph exploration: 3 semantic seeds plus every Symbol reachable from any seed via `CALLS | REFERENCES | TYPE_OF` within `depth` hops. `depth` defaults to 2."
    )]
    async fn explore(
        &self,
        Parameters(p): Parameters<ExploreParams>,
    ) -> Result<Json<graph::explore::ExploreResponse>, ErrorData> {
        let model = self
            .inner
            .embedding_model
            .as_ref()
            .ok_or_else(embeddings_disabled_err)?;
        let qvec = embed_query(model, p.topic).await.map_err(internal_err)?;
        graph::explore::expand(
            &self.inner.conn,
            &self.inner.workspace_dir,
            &self.inner.cfg,
            &qvec,
            p.depth.unwrap_or(2),
        )
        .await
        .map(Json)
        .map_err(internal_err)
    }
```

- [ ] **Step 4: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -5
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 75 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/mcp/mod.rs
git commit -m "feat(mcp): wire semantic_search, context, explore tools"
```

---

### Task 7: Integration test — all three tools end-to-end

**Files:**
- Create: `tests/integration_semantic.rs`

Mirrors `tests/integration_mcp.rs` (Slice 6) plus loads the model and passes it into `McpServer::new`. Gated on both `CODEGRAPH_RS_TEST_NEO4J=1` and `CODEGRAPH_RS_TEST_EMBEDDINGS=1`.

- [ ] **Step 1: Create `tests/integration_semantic.rs`**

```rust
//! Integration test: index tiny-rust, spin up the MCP server with the
//! embedding model loaded, and exercise codegraph_rs_semantic_search /
//! _context / _explore end-to-end.
//!
//! Gated on BOTH `CODEGRAPH_RS_TEST_NEO4J=1` and `CODEGRAPH_RS_TEST_EMBEDDINGS=1`.

use codegraph_rs::cli::{
    index,
    init::{self, InitArgs},
};
use codegraph_rs::config::workspace::load_from_path;
use codegraph_rs::db::Connection;
use codegraph_rs::mcp::McpServer;
use codegraph_rs::vectors::model::{default_cache_dir, load_model};
use neo4rs::query;
use rmcp::model::{CallToolRequestParams, ClientInfo, Implementation};
use rmcp::service::ServiceExt;
use serde_json::json;
use std::fs;
use std::sync::Arc;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
        && std::env::var("CODEGRAPH_RS_TEST_EMBEDDINGS").as_deref() == Ok("1")
}

#[test]
fn semantic_tools_against_tiny_rust() {
    if !enabled() {
        eprintln!(
            "SKIP: set both CODEGRAPH_RS_TEST_NEO4J=1 and CODEGRAPH_RS_TEST_EMBEDDINGS=1 to enable"
        );
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("semantic-test-{}", std::process::id());

    let fixture_src =
        std::path::Path::new(env!("CARGO_MANIFEST_DIR")).join("tests/fixtures/tiny-rust");
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

    let cfg = load_from_path(dir.path()).unwrap();
    let db_name = cfg.workspace.neo4j.database.clone();

    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw,
            &cfg.workspace.neo4j.database,
        )
        .await
        .unwrap();

        let model = load_model(&default_cache_dir()).expect("load embedding model");
        let embedding_model = Some(Arc::new(parking_lot::Mutex::new(model)));

        let server = McpServer::new(conn, cfg, dir.path().to_path_buf(), embedding_model);

        let (server_io, client_io) = tokio::io::duplex(64 * 1024);
        let client_info = ClientInfo::new(Default::default(), Implementation::new("test", "0"));
        let (server_result, client_result) =
            tokio::join!(server.serve(server_io), client_info.serve(client_io));
        let server_running = server_result.expect("server serve");
        let client = client_result.expect("client init");

        let tools = client
            .list_tools(Default::default())
            .await
            .expect("list_tools")
            .tools;
        let names: Vec<&str> = tools.iter().map(|t| t.name.as_ref()).collect();
        for expected in [
            "codegraph_rs_semantic_search",
            "codegraph_rs_context",
            "codegraph_rs_explore",
        ] {
            assert!(
                names.contains(&expected),
                "tool `{expected}` missing; have {names:?}"
            );
        }

        // semantic_search
        let sem = client
            .call_tool(
                CallToolRequestParams::new("codegraph_rs_semantic_search").with_arguments(
                    json!({ "query": "configuration parsing", "limit": 3 })
                        .as_object()
                        .unwrap()
                        .clone(),
                ),
            )
            .await
            .expect("semantic_search call");
        let sbody = sem.structured_content.expect("semantic_search structured");
        let results = sbody["results"].as_array().expect("results array");
        assert!(!results.is_empty(), "semantic_search should return at least one hit");
        assert!(results.len() <= 3, "limit=3 respected; got {}", results.len());
        for hit in results {
            assert!(
                hit["qid"].as_str().map(|s| !s.is_empty()).unwrap_or(false),
                "hit has non-empty qid: {hit}"
            );
            assert!(
                hit["score"].as_f64().map(|f| f.is_finite()).unwrap_or(false),
                "hit score is finite: {hit}"
            );
            let kind = hit["kind"].as_str().unwrap_or("");
            assert!(
                matches!(
                    kind,
                    "Function" | "Method" | "Struct" | "Trait" | "Enum"
                        | "TypeAlias" | "Module" | "Class" | "Interface"
                        | "Template" | "Choice"
                ),
                "hit kind is embed-eligible label; got `{kind}`"
            );
        }

        // semantic_search with root filter
        let sem_tiny = client
            .call_tool(
                CallToolRequestParams::new("codegraph_rs_semantic_search").with_arguments(
                    json!({ "query": "library", "root": "tiny", "limit": 5 })
                        .as_object()
                        .unwrap()
                        .clone(),
                ),
            )
            .await
            .expect("semantic_search w/ root call");
        let stbody = sem_tiny.structured_content.expect("root structured");
        let stresults = stbody["results"].as_array().expect("root results array");
        assert!(!stresults.is_empty(), "root-filtered should hit");
        for hit in stresults {
            assert_eq!(hit["root_name"], json!("tiny"), "root filter holds: {hit}");
        }

        // context
        let ctx = client
            .call_tool(
                CallToolRequestParams::new("codegraph_rs_context").with_arguments(
                    json!({ "task": "entry point and its helpers", "limit": 2 })
                        .as_object()
                        .unwrap()
                        .clone(),
                ),
            )
            .await
            .expect("context call");
        let cbody = ctx.structured_content.expect("context structured");
        let entries = cbody["entries"].as_array().expect("entries array");
        assert!(!entries.is_empty() && entries.len() <= 2);
        for e in entries {
            assert!(e["node"]["qid"].as_str().map(|s| !s.is_empty()).unwrap_or(false));
            assert!(e["callers"].is_array());
            assert!(e["callees"].is_array());
            assert!(e["source"].is_string());
            assert!(e["score"].as_f64().map(|f| f.is_finite()).unwrap_or(false));
        }

        // explore
        let exp = client
            .call_tool(
                CallToolRequestParams::new("codegraph_rs_explore").with_arguments(
                    json!({ "topic": "library entry", "depth": 2 })
                        .as_object()
                        .unwrap()
                        .clone(),
                ),
            )
            .await
            .expect("explore call");
        let ebody = exp.structured_content.expect("explore structured");
        let seeds = ebody["seeds"].as_array().expect("seeds array");
        assert!(!seeds.is_empty() && seeds.len() <= 3);
        for s in seeds {
            assert!(s["node"]["qid"].as_str().map(|str| !str.is_empty()).unwrap_or(false));
            assert!(s["score"].as_f64().map(|f| f.is_finite()).unwrap_or(false));
        }
        let reachable = ebody["reachable"].as_array().expect("reachable array");
        for r in reachable {
            let dist = r["distance_from_nearest_seed"].as_i64().expect("distance i64");
            assert!(dist >= 1 && dist <= 2, "distance in [1, depth]; got {dist}");
            assert!(r["qid"].as_str().map(|s| !s.is_empty()).unwrap_or(false));
            assert!(r["name"].as_str().map(|s| !s.is_empty()).unwrap_or(false));
        }

        drop(client);
        drop(server_running);

        let sys = Connection::system("bolt://localhost:7687", "neo4j", &pw)
            .await
            .unwrap();
        sys.graph
            .execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await
            .unwrap()
            .next()
            .await
            .ok();
    });

    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");
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

- [ ] **Step 2: Verify SKIP path runs cleanly**

```bash
cargo test --test integration_semantic -- --nocapture 2>&1 | tail -10
```

Expected: 1 test, SKIPs with "set both …" message, result `ok`.

- [ ] **Step 3: Build + clippy + fmt + full test suite (SKIP path)**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -10
```

Expected total: 76 tests (75 baseline + 1 new SKIP).

- [ ] **Step 4: Commit**

```bash
git add tests/integration_semantic.rs
git commit -m "test(mcp): integration test for semantic_search/context/explore"
```

---

### Task 8: README + slice-8 tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update README Status section**

Replace the current Status paragraph:

```markdown
## Status

In development. Currently implements **Slice 8: Semantic Search +
Context + Explore** — workspace initialization, indexing, resolution,
incremental sync, on-write `BAAI/bge-small-en-v1.5` embeddings, and
a stdio MCP server exposing 10 tools: 7 graph traversal/introspection
tools (`status`, `files`, `search`, `node`, `callers`, `callees`,
`impact`) plus 3 embedding-backed tools (`semantic_search`, `context`,
`explore`). Daml language support lands in the next slice.

The embedding model is downloaded on first use to
`~/.cache/codegraph-rs/fastembed/` (~130 MB; one-time cost) and reloaded
at `serve` startup so query-time embedding is cache-hot. Set
`[embeddings] enabled = false` in `.codegraph/config.toml` to opt out
of automatic embedding during `index` / `sync`; the three embedding-
backed MCP tools will then return a clear "embeddings disabled" error.
```

- [ ] **Step 2: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected: 76 tests pass.

- [ ] **Step 3: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 8"
git tag slice-8-semantic-search
```

- [ ] **Step 4: Verify**

```bash
git log --oneline | head -15
git tag --list
```

Expected tags: `slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`, `slice-6-mcp`, `slice-7-embeddings`, `slice-8-semantic-search`.

---

## Slice 8 Done

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — 76 tests pass, all integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all 76 pass; `integration_semantic` and `integration_vectors` SKIP (need second gate)
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 CODEGRAPH_RS_TEST_EMBEDDINGS=1 cargo test -- --test-threads=1` — all 76 pass, including both `integration_vectors` and `integration_semantic`
- [ ] Manual: `codegraph-rs serve` logs `"MCP serve: workspace `…`, db `…`, embeddings=true"` and (cold cache) the fastembed model download progress before opening stdio
- [ ] Manual: From a Claude Code session, `codegraph_rs_semantic_search { query: "configuration parsing", limit: 5 }` returns hits with `qid`, `name`, `kind`, `score`
- [ ] Manual: `codegraph_rs_context { task: "entry point and its helpers" }` returns entries each with `node`, `source`, `callers`, `callees`, `score`
- [ ] Manual: `codegraph_rs_explore { topic: "library", depth: 2 }` returns 3 seeds plus deduped `reachable` with every `distance_from_nearest_seed` ∈ [1, 2]
- [ ] Manual: flip `[embeddings] enabled = false`; `codegraph-rs serve` logs the disabled message; the three semantic tools return the disabled-embeddings error; the seven graph tools work unchanged

When all check, slice 8 is complete. **Next slice: Daml language support.**

### Small follow-ups noted during this slice

- **Parallelize `context` neighborhood fetches** with `try_join_all` if `limit > 5` becomes common.
- **Make `NUM_SEEDS` a tool parameter** if callers want wider semantic seed sets.
- **Pull the embed-eligible label list into one constant** (Slice 7 already noted this).
- **Add a unit test for the "embeddings disabled" error path** without spinning up Neo4j (construct `McpServer::new(..., None)` and exercise one tool).
