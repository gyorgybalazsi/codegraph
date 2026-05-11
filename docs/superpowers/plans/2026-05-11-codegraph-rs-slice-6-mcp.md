# codegraph-rs Slice 6 — MCP (graph-traversal tools)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `codegraph-rs serve` work as an MCP stdio server that exposes the graph-traversal subset of spec §10. After this slice, Claude Code can connect to a codegraph-rs workspace and discover symbols by name, fetch a symbol's full detail with source snippet, and walk callers/callees/impact. **No embeddings.** Semantic search, context bundles, and explore are explicitly deferred to the embeddings slice and the slice that follows it.

**Architecture:** Thin MCP server (rmcp) wrapping a new `graph` module. Each tool is a 5-line adapter: deserialize input → call `graph::*` method → serialize output. Heavy logic stays in `graph/` and in the existing `db/` queries. State is a single struct (`McpServer`) holding the Neo4j `Connection`, the `WorkspaceConfig`, and the workspace dir (needed for source-snippet reads).

**Tech Stack additions:**
- `rmcp = { version = "1.5", features = ["server", "macros", "transport-io", "transport-async-rw", "schemars"] }` — official Rust MCP SDK from Anthropic. The `transport-async-rw` feature is required because rmcp's `IntoTransport for (R, W)` impl (used to pass a tuple of `(AsyncRead, AsyncWrite)` to `serve(...)`) is gated on that feature. Without it, neither the `(Stdin, Stdout)` tuple in production nor the `tokio::io::duplex` halves in the integration test can be turned into a transport.
- `schemars = "0.8"` — JSON Schema generation for tool input/output structs (rmcp generates `inputSchema` / `outputSchema` from these).

No other new dependencies.

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §10 (MCP server) and §11 (CLI surface, the `serve` row).

**Slice 5 baseline:** HEAD `62a885b`, tagged `slice-5-sync`. 61 tests pass.

---

## Tool surface in this slice

All tools prefixed `codegraph_rs_` per spec §10 to coexist with the TS-version tools without collision.

| Tool | Inputs | Returns | Notes |
|---|---|---|---|
| `codegraph_rs_status` | — | `{ workspace, database, roots[], file_count, symbol_count, unresolved_count, last_indexed_at? }` | Reuses status logic refactored out of `cli/status.rs` |
| `codegraph_rs_files` | `root?: string` | `{ files: [{ root_name, relative_path, symbol_count }] }` | Optional `root` filter |
| `codegraph_rs_search` | `query: string`, `kinds?: string[]`, `root?: string`, `limit?: number=10` | `{ results: [{ qid, name, kind, qualified_name, root_name, relative_path }] }` | Cypher uses Neo4j FTS index `symbol_fts` (already present in schema since slice 2) |
| `codegraph_rs_node` | `qid: string` | `{ node: { qid, name, kind, qualified_name, root_name, relative_path, start_line, end_line }, source: string }` | Reads source slice from disk using `root_name` + `relative_path` |
| `codegraph_rs_callers` | `qid: string`, `depth?: number=2` | `{ callers: [{ qid, name, kind, hops }] }` | Reverse direction of `CALLS` |
| `codegraph_rs_callees` | `qid: string`, `depth?: number=2` | `{ callees: [{ qid, name, kind, hops }] }` | Forward direction of `CALLS` |
| `codegraph_rs_impact` | `qid: string`, `depth?: number=3` | `{ impact: [{ qid, name, kind, hops, edge_kind }] }` | Reverse-transitive closure of `CALLS`/`REFERENCES`/`TYPE_OF` |

## Out of scope for Slice 6

- **Semantic search** (`codegraph_rs_semantic_search`). Needs embeddings; deferred to the embeddings slice.
- **Context bundles and explore** (`codegraph_rs_context`, `codegraph_rs_explore`). Far more valuable with embeddings; designed in the slice after embeddings.
- **Streaming responses, auth, multi-client.** Per spec §10 "Not in v1".
- **Background reindex during `serve`.** User runs `codegraph-rs sync` separately. Per spec §10 and prior slices.
- **MCP `Resources` and `Prompts` capabilities.** Slice exposes only `Tools`.
- **Error suggestion fuzzy-match** (spec §10 "Unknown qid → fuzzy-match suggestion"). v1 returns a plain "qid not found" error; fuzzy match deferred.
- **HTTP/SSE transport.** stdio only.

---

## File structure produced by this slice

```
codegraph-rs/
  Cargo.toml                  # add rmcp + schemars
  src/
    lib.rs                    # add `pub mod mcp;` and `pub mod graph;`
    cli/
      mod.rs                  # wire Command::Serve -> serve::run
      serve.rs                # NEW: codegraph-rs serve command
      status.rs               # MODIFY: delegate to graph::status::collect()
    graph/
      mod.rs                  # NEW: re-exports + module index
      status.rs               # NEW: StatusSnapshot + collect()
      files.rs                # NEW: FileEntry + list()
      search.rs               # NEW: SearchHit + search()
      node.rs                 # NEW: NodeDetail + find()
      traverse.rs             # NEW: callers(), callees(), impact()
      source.rs               # NEW: read_snippet() — disk read by root_name + relative_path + line range
    mcp/
      mod.rs                  # NEW: McpServer struct + ServerHandler impl + 7 tools
  tests/
    graph_unit.rs             # NEW: pure-Rust tests for source::read_snippet
    integration_mcp.rs        # NEW: gated; in-process MCP via tokio::io::duplex
```

**Lines of new code estimate:** ~600 production + ~250 test. Most code is short Cypher wrappers and thin MCP adapters.

---

## Why a separate `graph` module

The MCP tools must dispatch to **pure** functions: take typed args, return typed structs, no Cypher leakage. Today the only candidates for this work are `db::queries::*` (resolution-specific) and inline Cypher in `cli/status.rs`. Inlining 7 more Cypher queries into MCP tool handlers would put all the schema knowledge in one place where it's hardest to test (inside the MCP server). Extracting them into `graph::*` modules:

- Keeps each query independently testable (later, via integration tests against tiny-rust).
- Makes `mcp/tools.rs` adapter code stay thin (one liners), matching spec §10's "Heavy logic stays in non-MCP modules".
- Lets `cli/status.rs` reuse `graph::status::collect()` instead of duplicating count Cypher.
- Sets up the API surface that the embeddings slice and the context/explore slice will extend.

---

### Task 1: Module skeleton + dependencies

**Files:**
- Modify: `Cargo.toml`
- Modify: `src/lib.rs`
- Create: `src/graph/mod.rs`, `src/mcp/mod.rs`

- [ ] **Step 1: Add deps to Cargo.toml**

Append to `[dependencies]`:

```toml
rmcp = { version = "1.5", features = ["server", "macros", "transport-io", "transport-async-rw", "schemars"] }
schemars = "0.8"
```

- [ ] **Step 2: Derive `Clone` on workspace config types**

Slice 6's MCP server takes ownership of `WorkspaceConfig` and the integration test also needs to retain the config for post-test cleanup. Edit `src/config/workspace.rs` and add `Clone` to the four config struct derives:

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct WorkspaceConfig { ... }

#[derive(Debug, Clone, Deserialize)]
pub struct Workspace { ... }

#[derive(Debug, Clone, Deserialize)]
pub struct Neo4j { ... }

#[derive(Debug, Clone, Deserialize)]
pub struct Root { ... }

#[derive(Debug, Clone, Deserialize)]
pub struct Indexing { ... }

#[derive(Debug, Clone, Deserialize)]
pub struct Embeddings { ... }
```

No other change to those types.

- [ ] **Step 3: Module declarations**

In `src/lib.rs`, add (alphabetical, after `pub mod extraction;`):

```rust
pub mod graph;
pub mod mcp;
```

- [ ] **Step 4: Create empty module roots**

`src/graph/mod.rs`:
```rust
//! Read-only graph queries — typed wrappers over Cypher.
//!
//! Each submodule defines a small set of pure-data response structs and a single
//! `pub async fn` that takes a `&Connection` (and parameters) and returns those
//! structs. No Cypher leaks beyond this module.
```

`src/mcp/mod.rs`:
```rust
//! MCP stdio server. See spec §10.
```

- [ ] **Step 5: Build + clippy + fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -3
```

Expected: clean build (downloads rmcp + schemars + transitive deps the first time), 61 tests still pass.

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml Cargo.lock src/lib.rs src/config/workspace.rs src/graph/mod.rs src/mcp/mod.rs
git commit -m "chore(mcp): add rmcp + schemars deps and module skeleton"
```

---

### Task 2: `graph::status` and `graph::source`

**Files:**
- Create: `src/graph/status.rs`
- Create: `src/graph/source.rs`
- Modify: `src/graph/mod.rs` (declare submodules)
- Modify: `src/cli/status.rs` (delegate)

The `status` query is a no-arg snapshot of workspace counts. The `source` helper is needed early because the `node` query depends on it. Combining them in one task keeps Task 5 (`graph::node`) tight.

- [ ] **Step 1: `src/graph/status.rs`**

```rust
//! Workspace snapshot — file / symbol / unresolved counts + last-indexed timestamp.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct StatusSnapshot {
    pub workspace: String,
    pub database: String,
    pub roots: Vec<String>,
    pub file_count: i64,
    pub symbol_count: i64,
    pub unresolved_count: i64,
    pub last_indexed_at: Option<i64>,
}

pub async fn collect(conn: &Connection, cfg: &WorkspaceConfig) -> Result<StatusSnapshot> {
    let file_count = single_count(conn, "MATCH (n:File) RETURN count(n) AS c").await?;
    let symbol_count = single_count(conn, "MATCH (n:Symbol) RETURN count(n) AS c").await?;
    let unresolved_count =
        single_count(conn, "MATCH (n:UnresolvedRef) RETURN count(n) AS c").await?;
    let last_indexed_at = max_last_indexed(conn).await?;

    Ok(StatusSnapshot {
        workspace: cfg.workspace.name.clone(),
        database: cfg.workspace.neo4j.database.clone(),
        roots: cfg.workspace.roots.iter().map(|r| r.name.clone()).collect(),
        file_count,
        symbol_count,
        unresolved_count,
        last_indexed_at,
    })
}

async fn single_count(conn: &Connection, cypher: &str) -> Result<i64> {
    let mut stream = conn
        .graph
        .execute(query(cypher))
        .await
        .with_context(|| format!("running `{cypher}`"))?;
    let row = stream
        .next()
        .await
        .with_context(|| format!("stream error reading `{cypher}`"))?
        .with_context(|| format!("`{cypher}` returned no row"))?;
    row.get("c").context("row missing `c`")
}

async fn max_last_indexed(conn: &Connection) -> Result<Option<i64>> {
    let mut stream = conn
        .graph
        .execute(query("MATCH (n:File) RETURN max(n.last_indexed_at) AS t"))
        .await?;
    match stream.next().await? {
        None => Ok(None),
        Some(row) => row.get::<Option<i64>>("t").context("missing `t`"),
    }
}
```

This is a verbatim port of the helpers in the current `cli/status.rs` — same Cypher, same error contexts.

- [ ] **Step 2: `src/graph/source.rs`**

```rust
//! Read source-code slices from disk using stored (root_name, relative_path, line range).

use anyhow::{anyhow, Context, Result};
use std::path::{Path, PathBuf};

use crate::config::workspace::WorkspaceConfig;

/// Resolves a `root_name` to an absolute filesystem path under `workspace_dir`.
pub fn root_path(
    workspace_dir: &Path,
    cfg: &WorkspaceConfig,
    root_name: &str,
) -> Result<PathBuf> {
    let root = cfg
        .workspace
        .roots
        .iter()
        .find(|r| r.name == root_name)
        .ok_or_else(|| anyhow!("unknown root `{root_name}` (not in workspace config)"))?;
    Ok(workspace_dir.join(&root.path))
}

/// Reads `[start_line..=end_line]` (1-based, inclusive) from the given file.
/// Returns the trimmed slice; empty string if the file is shorter than `start_line`.
pub fn read_snippet(
    workspace_dir: &Path,
    cfg: &WorkspaceConfig,
    root_name: &str,
    relative_path: &str,
    start_line: u32,
    end_line: u32,
) -> Result<String> {
    let abs = root_path(workspace_dir, cfg, root_name)?.join(relative_path);
    let content = std::fs::read_to_string(&abs)
        .with_context(|| format!("reading {}", abs.display()))?;
    let s = start_line.saturating_sub(1) as usize;
    let e = end_line.saturating_sub(1) as usize;
    let lines: Vec<&str> = content.lines().collect();
    if s >= lines.len() {
        return Ok(String::new());
    }
    let end = e.min(lines.len().saturating_sub(1));
    Ok(lines[s..=end].join("\n"))
}
```

- [ ] **Step 3: Wire submodules**

In `src/graph/mod.rs`, add:
```rust
pub mod source;
pub mod status;
```

- [ ] **Step 4: Refactor `src/cli/status.rs`**

Replace the body of `run()` so it uses `graph::status::collect()` and only handles `Connection::open` + `println!` formatting:

```rust
//! `codegraph-rs status` — print workspace stats.

use anyhow::{Context, Result};
use std::path::Path;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::Connection;
use crate::graph::status;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg
        .map(Path::to_path_buf)
        .unwrap_or_else(|| std::env::current_dir().expect("cwd readable"));
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

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

        let snap = status::collect(&conn, &cfg).await?;

        println!("Workspace: {}", snap.workspace);
        println!("Database:  {}", snap.database);
        println!("Roots:     {}", snap.roots.join(", "));
        println!("Files:     {}", snap.file_count);
        println!("Symbols:   {}", snap.symbol_count);
        println!("Unresolved refs: {}", snap.unresolved_count);
        println!(
            "Last indexed:    {}",
            match snap.last_indexed_at {
                Some(t) => t.to_string(),
                None => "never".into(),
            }
        );
        Ok::<(), anyhow::Error>(())
    })
}
```

Remove the now-unused `single_count` / `max_last_indexed` helpers and the `neo4rs::query` import from this file.

- [ ] **Step 5: Add a pure-Rust unit test for `read_snippet`**

The config types in `src/config/workspace.rs` are `WorkspaceConfig` / `Workspace` / `Neo4j` / `Root` (not `WorkspaceSection` / `Neo4jConfig` / `RootConfig`) and `WorkspaceConfig` has required fields (`schema_version`, `indexing`, `embeddings`) beyond what the test needs. Rather than construct a struct literal, build the config by parsing a TOML literal via `workspace::parse(...)` (a public function at `src/config/workspace.rs:83`).

Create `tests/graph_unit.rs`:

```rust
//! Pure-Rust tests for graph helpers that don't need Neo4j.

use codegraph_rs::config::workspace::{self, WorkspaceConfig};
use codegraph_rs::graph::source::read_snippet;
use std::fs;
use tempfile::tempdir;

fn dummy_cfg(root_name: &str) -> WorkspaceConfig {
    let toml = format!(
        r#"
schema_version = 1

[workspace]
name = "test"

[workspace.neo4j]
uri = "bolt://localhost:7687"
database = "test-db"
username = "neo4j"

[[workspace.roots]]
name = "{root_name}"
path = "src"
"#
    );
    workspace::parse(&toml).expect("parse test config")
}

#[test]
fn read_snippet_returns_inclusive_line_range() {
    let dir = tempdir().unwrap();
    let src = dir.path().join("src");
    fs::create_dir_all(&src).unwrap();
    fs::write(src.join("file.rs"), "one\ntwo\nthree\nfour\nfive\n").unwrap();

    let cfg = dummy_cfg("primary");
    let snippet = read_snippet(dir.path(), &cfg, "primary", "file.rs", 2, 4).unwrap();
    assert_eq!(snippet, "two\nthree\nfour");
}

#[test]
fn read_snippet_clamps_end_beyond_eof() {
    let dir = tempdir().unwrap();
    let src = dir.path().join("src");
    fs::create_dir_all(&src).unwrap();
    fs::write(src.join("file.rs"), "a\nb\nc\n").unwrap();

    let cfg = dummy_cfg("primary");
    let snippet = read_snippet(dir.path(), &cfg, "primary", "file.rs", 2, 999).unwrap();
    assert_eq!(snippet, "b\nc");
}

#[test]
fn read_snippet_empty_when_start_past_eof() {
    let dir = tempdir().unwrap();
    let src = dir.path().join("src");
    fs::create_dir_all(&src).unwrap();
    fs::write(src.join("file.rs"), "a\nb\n").unwrap();

    let cfg = dummy_cfg("primary");
    let snippet = read_snippet(dir.path(), &cfg, "primary", "file.rs", 50, 60).unwrap();
    assert_eq!(snippet, "");
}

#[test]
fn read_snippet_errors_on_unknown_root() {
    let dir = tempdir().unwrap();
    let cfg = dummy_cfg("primary");
    let err = read_snippet(dir.path(), &cfg, "other", "file.rs", 1, 1).unwrap_err();
    assert!(err.to_string().contains("unknown root"));
}
```

- [ ] **Step 6: Build + tests**

Expected: 65 tests pass (61 + 4 new graph_unit tests). Clippy + fmt clean.

- [ ] **Step 7: Commit**

```bash
git add src/graph/mod.rs src/graph/status.rs src/graph/source.rs src/cli/status.rs tests/graph_unit.rs
git commit -m "feat(graph): status + source snippet helpers; refactor cli/status to reuse"
```

---

### Task 3: `graph::files`

**Files:**
- Create: `src/graph/files.rs`
- Modify: `src/graph/mod.rs`

- [ ] **Step 1: `src/graph/files.rs`**

```rust
//! List indexed files with per-file symbol counts. Optional `root` filter.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;

use crate::db::Connection;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct FileEntry {
    pub root_name: String,
    pub relative_path: String,
    pub symbol_count: i64,
}

pub async fn list(conn: &Connection, root: Option<&str>) -> Result<Vec<FileEntry>> {
    let cypher = match root {
        Some(_) => {
            "MATCH (f:File {root_name: $root}) \
             OPTIONAL MATCH (f)-[:CONTAINS*]->(s:Symbol) \
             RETURN f.root_name AS root_name, f.relative_path AS relative_path, \
                    count(s) AS symbol_count \
             ORDER BY relative_path"
        }
        None => {
            "MATCH (f:File) \
             OPTIONAL MATCH (f)-[:CONTAINS*]->(s:Symbol) \
             RETURN f.root_name AS root_name, f.relative_path AS relative_path, \
                    count(s) AS symbol_count \
             ORDER BY root_name, relative_path"
        }
    };

    let mut q = query(cypher);
    if let Some(r) = root {
        q = q.param("root", r);
    }
    let mut stream = conn.graph.execute(q).await.context("listing files")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await? {
        out.push(FileEntry {
            root_name: row.get("root_name").context("row missing root_name")?,
            relative_path: row.get("relative_path").context("row missing relative_path")?,
            symbol_count: row.get("symbol_count").context("row missing symbol_count")?,
        });
    }
    Ok(out)
}
```

- [ ] **Step 2: Wire submodule**

In `src/graph/mod.rs`:
```rust
pub mod files;
```

- [ ] **Step 3: Build + clippy + fmt + test**

No new tests — exercised end-to-end via `integration_mcp.rs` in Task 10.

- [ ] **Step 4: Commit**

```bash
git add src/graph/mod.rs src/graph/files.rs
git commit -m "feat(graph): files() listing with optional root filter"
```

---

### Task 4: `graph::search`

**Files:**
- Create: `src/graph/search.rs`
- Modify: `src/graph/mod.rs`

The schema already has FTS index `symbol_fts` on `[n.name, n.qualified_name, n.signature, n.doc]` (per `src/db/schema.rs`). The query uses `CALL db.index.fulltext.queryNodes("symbol_fts", $query)` and filters by optional `kinds` / `root` after.

- [ ] **Step 1: `src/graph/search.rs`**

```rust
//! Symbol search via Neo4j FTS index `symbol_fts`.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;
use serde_json::json;

use crate::db::write::to_bolt;
use crate::db::Connection;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct SearchHit {
    pub qid: String,
    pub name: String,
    pub kind: String,
    pub qualified_name: Option<String>,
    pub root_name: String,
    pub relative_path: String,
    pub score: f64,
}

pub async fn search(
    conn: &Connection,
    q: &str,
    kinds: Option<&[String]>,
    root: Option<&str>,
    limit: i64,
) -> Result<Vec<SearchHit>> {
    // FTS query string is passed through verbatim. Neo4j tokenizes it
    // with the standard analyzer (lowercases, splits on whitespace/punct).
    let cypher = "\
        CALL db.index.fulltext.queryNodes('symbol_fts', $q) YIELD node AS n, score \
        WHERE ($root IS NULL OR n.root_name = $root) \
          AND ($kinds IS NULL OR any(k IN $kinds WHERE k IN labels(n))) \
        RETURN n.qid AS qid, n.name AS name, n.qualified_name AS qualified_name, \
               n.root_name AS root_name, n.relative_path AS relative_path, \
               [l IN labels(n) WHERE l <> 'Symbol'][0] AS kind, score \
        ORDER BY score DESC \
        LIMIT $limit";

    // Nullable params go through the established JSON→BoltType path
    // (db::write::to_bolt) — same pattern used by write_extraction.
    // serde_json maps None -> Value::Null which BoltType::TryFrom handles.
    let root_param = to_bolt(json!(root)).context("encoding `root` param")?;
    let kinds_param = to_bolt(json!(kinds)).context("encoding `kinds` param")?;

    let mut stream = conn
        .graph
        .execute(
            query(cypher)
                .param("q", q)
                .param("root", root_param)
                .param("kinds", kinds_param)
                .param("limit", limit),
        )
        .await
        .context("running search")?;

    let mut out = Vec::new();
    while let Some(row) = stream.next().await? {
        out.push(SearchHit {
            qid: row.get("qid").context("row missing qid")?,
            name: row.get("name").context("row missing name")?,
            kind: row.get::<Option<String>>("kind")?.unwrap_or_default(),
            qualified_name: row.get::<Option<String>>("qualified_name")?,
            root_name: row.get("root_name").context("row missing root_name")?,
            relative_path: row.get("relative_path").context("row missing relative_path")?,
            score: row.get("score").context("row missing score")?,
        });
    }
    Ok(out)
}
```

> **Visibility note.** `db::write::to_bolt` is currently `pub(crate)`; that's already crate-visible so `graph::search` can call it as-is.

> **Implementer note on `kind` extraction.** Each symbol carries two labels: `:Symbol` and exactly one of `Function` / `Method` / `Class` / etc. (the writer at `src/db/write.rs:66` always merges with two labels: `MERGE (n:Symbol:{label} ...)`). The Cypher `[l IN labels(n) WHERE l <> 'Symbol'][0]` picks the only non-`Symbol` label. If a node somehow had more than one kind label, Neo4j's label order is unspecified — but that invariant doesn't hold today and isn't relied on elsewhere.

> **FTS injection note.** The `query` string is passed verbatim into `db.index.fulltext.queryNodes`. Lucene-syntax errors (`name:`, unbalanced `"`, trailing `~`) will produce a Neo4j error that the MCP server surfaces as `ErrorData::internal_error` with the underlying message. Acceptable for v1; sanitization deferred.

- [ ] **Step 2: Wire submodule**

```rust
pub mod search;
```

- [ ] **Step 3: Build + clippy + fmt + test**

Exercised end-to-end via `integration_mcp.rs`.

- [ ] **Step 4: Commit**

```bash
git add src/graph/mod.rs src/graph/search.rs
git commit -m "feat(graph): FTS-backed search() with kind/root filters"
```

---

### Task 5: `graph::node`

**Files:**
- Create: `src/graph/node.rs`
- Modify: `src/graph/mod.rs`

- [ ] **Step 1: `src/graph/node.rs`**

```rust
//! Detail lookup for a single symbol by qid, plus its source snippet.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;
use std::path::Path;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;
use crate::graph::source::read_snippet;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct NodeDetail {
    pub qid: String,
    pub name: String,
    pub kind: String,
    pub qualified_name: Option<String>,
    pub root_name: String,
    pub relative_path: String,
    pub start_line: u32,
    pub end_line: u32,
    pub signature: Option<String>,
    pub doc: Option<String>,
}

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct NodeResponse {
    pub node: NodeDetail,
    pub source: String,
}

pub async fn find(
    conn: &Connection,
    workspace_dir: &Path,
    cfg: &WorkspaceConfig,
    qid: &str,
) -> Result<NodeResponse> {
    let cypher = "\
        MATCH (n:Symbol {qid: $qid}) \
        RETURN n.qid AS qid, n.name AS name, n.qualified_name AS qualified_name, \
               [l IN labels(n) WHERE l <> 'Symbol'][0] AS kind, \
               n.root_name AS root_name, n.relative_path AS relative_path, \
               n.start_line AS start_line, n.end_line AS end_line, \
               n.signature AS signature, n.doc AS doc \
        LIMIT 1";
    let mut stream = conn
        .graph
        .execute(query(cypher).param("qid", qid))
        .await
        .context("running node lookup")?;
    let row = stream
        .next()
        .await
        .context("stream error")?
        .with_context(|| format!("qid `{qid}` not found"))?;

    let detail = NodeDetail {
        qid: row.get("qid")?,
        name: row.get("name")?,
        kind: row.get::<Option<String>>("kind")?.unwrap_or_default(),
        qualified_name: row.get::<Option<String>>("qualified_name")?,
        root_name: row.get("root_name")?,
        relative_path: row.get("relative_path")?,
        start_line: row.get::<i64>("start_line")? as u32,
        end_line: row.get::<i64>("end_line")? as u32,
        signature: row.get::<Option<String>>("signature")?,
        doc: row.get::<Option<String>>("doc")?,
    };

    let source = read_snippet(
        workspace_dir,
        cfg,
        &detail.root_name,
        &detail.relative_path,
        detail.start_line,
        detail.end_line,
    )
    .unwrap_or_default();

    Ok(NodeResponse { node: detail, source })
}
```

> **Implementer note.** If `start_line` / `end_line` are stored as `i64` in Neo4j, the `as u32` cast is fine; if they can be NULL, fall back to 0 / 0 — the snippet will just be empty.

> **Snippet read failures are non-fatal.** `read_snippet` may fail (file deleted since last index). We use `.unwrap_or_default()` so the response still carries the detail; the empty source signals to callers that the file is unreadable.

- [ ] **Step 2: Wire submodule**

```rust
pub mod node;
```

- [ ] **Step 3: Build + clippy + fmt + test**

- [ ] **Step 4: Commit**

```bash
git add src/graph/mod.rs src/graph/node.rs
git commit -m "feat(graph): node() detail lookup + source snippet"
```

---

### Task 6: `graph::traverse` — callers + callees + impact

**Files:**
- Create: `src/graph/traverse.rs`
- Modify: `src/graph/mod.rs`

All three are variable-length path queries with a `hops` count.

- [ ] **Step 1: `src/graph/traverse.rs`**

```rust
//! Callers, callees, and impact (reverse-transitive closure) via variable-length paths.

use anyhow::{Context, Result};
use neo4rs::query;
use serde::Serialize;

use crate::db::Connection;

#[derive(Debug, Clone, Serialize, schemars::JsonSchema)]
pub struct Neighbor {
    pub qid: String,
    pub name: String,
    pub kind: String,
    pub hops: i64,
}

pub async fn callers(conn: &Connection, qid: &str, depth: i64) -> Result<Vec<Neighbor>> {
    let cypher = "\
        MATCH path = (caller:Symbol)-[:CALLS*1..]->(target:Symbol {qid: $qid}) \
        WHERE length(path) <= $depth \
        WITH caller, min(length(path)) AS hops \
        RETURN caller.qid AS qid, caller.name AS name, \
               [l IN labels(caller) WHERE l <> 'Symbol'][0] AS kind, hops \
        ORDER BY hops ASC, name ASC";
    rows_to_neighbors(conn, cypher, qid, depth).await
}

pub async fn callees(conn: &Connection, qid: &str, depth: i64) -> Result<Vec<Neighbor>> {
    let cypher = "\
        MATCH path = (source:Symbol {qid: $qid})-[:CALLS*1..]->(callee:Symbol) \
        WHERE length(path) <= $depth \
        WITH callee, min(length(path)) AS hops \
        RETURN callee.qid AS qid, callee.name AS name, \
               [l IN labels(callee) WHERE l <> 'Symbol'][0] AS kind, hops \
        ORDER BY hops ASC, name ASC";
    rows_to_neighbors(conn, cypher, qid, depth).await
}

/// Impact = reverse-transitive closure across CALLS, REFERENCES, TYPE_OF.
/// "What's affected if this symbol changes?"
pub async fn impact(conn: &Connection, qid: &str, depth: i64) -> Result<Vec<Neighbor>> {
    let cypher = "\
        MATCH path = (affected:Symbol)-[:CALLS|REFERENCES|TYPE_OF*1..]->(target:Symbol {qid: $qid}) \
        WHERE length(path) <= $depth \
        WITH affected, min(length(path)) AS hops \
        RETURN affected.qid AS qid, affected.name AS name, \
               [l IN labels(affected) WHERE l <> 'Symbol'][0] AS kind, hops \
        ORDER BY hops ASC, name ASC";
    rows_to_neighbors(conn, cypher, qid, depth).await
}

async fn rows_to_neighbors(
    conn: &Connection,
    cypher: &str,
    qid: &str,
    depth: i64,
) -> Result<Vec<Neighbor>> {
    let mut stream = conn
        .graph
        .execute(query(cypher).param("qid", qid).param("depth", depth))
        .await
        .context("running traversal")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await? {
        out.push(Neighbor {
            qid: row.get("qid")?,
            name: row.get("name")?,
            kind: row.get::<Option<String>>("kind")?.unwrap_or_default(),
            hops: row.get("hops")?,
        });
    }
    Ok(out)
}
```

> **Implementer note on relationship type names.** The Cypher above assumes the relationship types are exactly `CALLS`, `REFERENCES`, `TYPE_OF`. Verify against `src/extraction/result.rs` (or wherever edge kinds are serialized) — if any differ, adjust.

- [ ] **Step 2: Wire submodule**

```rust
pub mod traverse;
```

- [ ] **Step 3: Build + clippy + fmt + test**

- [ ] **Step 4: Commit**

```bash
git add src/graph/mod.rs src/graph/traverse.rs
git commit -m "feat(graph): callers/callees/impact traversal queries"
```

---

### Task 7: MCP server skeleton + `serve` CLI

**Files:**
- Create: `src/mcp/mod.rs` (replace skeleton)
- Create: `src/cli/serve.rs`
- Modify: `src/cli/mod.rs`

Establishes the rmcp plumbing with **only one tool** wired (`codegraph_rs_status`). Task 8 will add the remaining six. This split keeps the rmcp wiring isolated from the per-tool wiring so any rmcp API quirks surface in a small commit.

- [ ] **Step 1: Replace `src/mcp/mod.rs`**

```rust
//! MCP stdio server exposing graph-traversal tools. See spec §10.

use anyhow::{Context, Result};
use rmcp::{
    handler::server::tool::Parameters,
    model::{Implementation, ProtocolVersion, ServerCapabilities, ServerInfo},
    schemars,
    serde_json::{self, Value},
    service::ServiceExt,
    tool, tool_handler, tool_router,
    ErrorData, Json,
};
use std::path::PathBuf;
use std::sync::Arc;

use crate::config::workspace::WorkspaceConfig;
use crate::db::Connection;
use crate::graph;

#[derive(Clone)]
pub struct McpServer {
    inner: Arc<Inner>,
}

struct Inner {
    conn: Connection,
    cfg: WorkspaceConfig,
    workspace_dir: PathBuf,
}

impl McpServer {
    pub fn new(conn: Connection, cfg: WorkspaceConfig, workspace_dir: PathBuf) -> Self {
        Self { inner: Arc::new(Inner { conn, cfg, workspace_dir }) }
    }
}

#[tool_router(server_handler)]
impl McpServer {
    #[tool(
        name = "codegraph_rs_status",
        description = "Return workspace stats: file/symbol/unresolved counts and last-indexed timestamp."
    )]
    async fn status(&self) -> Result<Json<graph::status::StatusSnapshot>, ErrorData> {
        graph::status::collect(&self.inner.conn, &self.inner.cfg)
            .await
            .map(Json)
            .map_err(|e| ErrorData::internal_error(e.to_string(), None))
    }
}

/// Run the server on stdio until stdin closes.
pub async fn serve(conn: Connection, cfg: WorkspaceConfig, workspace_dir: PathBuf) -> Result<()> {
    let server = McpServer::new(conn, cfg, workspace_dir);
    let (stdin, stdout) = rmcp::transport::io::stdio();
    let running = server
        .serve((stdin, stdout))
        .await
        .context("starting MCP server")?;
    // `RunningService::waiting()` returns `Result<QuitReason, JoinError>`.
    // A normal stdin EOF (e.g. parent process closing) yields `Ok(QuitReason::Closed)`;
    // a panic in the task yields `Err(JoinError)`. We treat both as "server stopped"
    // for the CLI exit path — JoinError is logged but doesn't propagate as anyhow.
    match running.waiting().await {
        Ok(reason) => {
            tracing::info!("MCP server exited: {reason:?}");
        }
        Err(join_err) => {
            tracing::warn!("MCP server task join error: {join_err}");
        }
    }
    Ok(())
}
```

> **rmcp 1.5 API anchors (verified before writing this plan):**
> - `#[tool_router(server_handler)]` on the `impl` block auto-derives the `ServerHandler` trait. No separate `#[tool_handler] impl ServerHandler for McpServer {}` block is required.
> - `serve(transport).await` returns `Result<RunningService, _>`; `RunningService::waiting(self).await -> Result<QuitReason, JoinError>`.
> - `ErrorData::internal_error(message: impl Into<Cow<'static, str>>, data: Option<Value>)` — pass `None` for `data`.
> - The default `ServerInfo` (name + version from `Cargo.toml`) is generated by the `server_handler` flag of `#[tool_router]`. To add a custom "instructions" string, override `fn get_info(&self) -> ServerInfo` explicitly inside the impl block.

- [ ] **Step 2: `src/cli/serve.rs`**

`serve` is a read-only consumer. Unlike `index` and `sync` (which write nodes/edges), it never mutates the graph, so it does NOT call `apply_schema` — the schema is established by `init` and maintained by `index`/`sync`. Calling `apply_schema` here would add latency on every `serve` start and could mask drift in upstream slices.

```rust
//! `codegraph-rs serve` — MCP stdio server.

use anyhow::{Context, Result};
use std::path::Path;
use tracing::info;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::Connection;
use crate::mcp;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg
        .map(Path::to_path_buf)
        .unwrap_or_else(|| std::env::current_dir().expect("cwd readable"));
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

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

        info!("MCP serve: workspace `{}`, db `{}`", cfg.workspace.name, cfg.workspace.neo4j.database);
        mcp::serve(conn, cfg, workspace_dir).await
    })
}
```

- [ ] **Step 3: Wire dispatch in `src/cli/mod.rs`**

Add `pub mod serve;` next to `pub mod sync;`. Replace the stub arm:
```rust
Command::Query { .. } | Command::Serve => {
    anyhow::bail!("not yet implemented");
}
```
with:
```rust
Command::Serve => serve::run(cli.workspace.as_deref()),
Command::Query { .. } => {
    anyhow::bail!("not yet implemented");
}
```

- [ ] **Step 4: Update cli_smoke test**

`tests/cli_smoke.rs` currently has a `stub_commands_fail_with_not_implemented` test whose body asserts that `serve` exits non-zero with "not yet implemented". After this task wires `serve`, that assertion no longer holds — `serve` will now exit non-zero with a *different* message ("no workspace found" or similar, depending on where this test runs). Rename and retarget the test to `query`, which remains a stub in this slice:

Replace:
```rust
#[test]
fn stub_commands_fail_with_not_implemented() {
    let sub = "serve";
    let out = bin().arg(sub).output().expect("run subcommand");
    assert!(!out.status.success(), "`{sub}` should exit nonzero");
    let stderr = String::from_utf8_lossy(&out.stderr);
    assert!(
        stderr.contains("not yet implemented"),
        "`{sub}` should print 'not yet implemented' in stderr; got: {stderr}"
    );
}
```

With:
```rust
#[test]
fn query_stub_fails_with_not_implemented() {
    let out = bin().arg("query").arg("foo").output().expect("run query");
    assert!(!out.status.success(), "`query` should exit nonzero");
    let stderr = String::from_utf8_lossy(&out.stderr);
    assert!(
        stderr.contains("not yet implemented"),
        "`query` should print 'not yet implemented'; got: {stderr}"
    );
}
```

The `help_exits_zero_and_lists_all_subcommands` test is unchanged — it still asserts all six subcommands appear in `--help`.

- [ ] **Step 5: Build + clippy + fmt + test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -10
```

Expected: 65 tests pass (test count unchanged from Task 2; cli_smoke loses one assert-loop iteration but the renamed test covers it). All integration tests still SKIP cleanly without `CODEGRAPH_RS_TEST_NEO4J`.

- [ ] **Step 6: Commit**

```bash
git add src/mcp/mod.rs src/cli/serve.rs src/cli/mod.rs tests/cli_smoke.rs
git commit -m "feat(mcp): rmcp stdio server skeleton with status tool; wire serve CLI"
```

---

### Task 8: Wire remaining 6 MCP tools

**Files:**
- Modify: `src/mcp/mod.rs`

Add `files`, `search`, `node`, `callers`, `callees`, `impact` tools — each a 5-10 line adapter dispatching to `graph::*`.

- [ ] **Step 1: Add parameter structs**

At the top of `src/mcp/mod.rs`, after the `use` block, add:

```rust
#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct FilesParams {
    pub root: Option<String>,
}

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct SearchParams {
    pub query: String,
    pub kinds: Option<Vec<String>>,
    pub root: Option<String>,
    pub limit: Option<i64>,
}

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct NodeParams {
    pub qid: String,
}

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct CallersParams {
    pub qid: String,
    pub depth: Option<i64>,
}

pub type CalleesParams = CallersParams;

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct ImpactParams {
    pub qid: String,
    pub depth: Option<i64>,
}

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct FilesResponse { pub files: Vec<graph::files::FileEntry> }

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct SearchResponse { pub results: Vec<graph::search::SearchHit> }

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct CallersResponse { pub callers: Vec<graph::traverse::Neighbor> }

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct CalleesResponse { pub callees: Vec<graph::traverse::Neighbor> }

#[derive(serde::Serialize, schemars::JsonSchema)]
pub struct ImpactResponse { pub impact: Vec<graph::traverse::Neighbor> }
```

- [ ] **Step 2: Add the 6 tool handlers to the `#[tool_router]` impl**

Inside the `impl McpServer` block, alongside `status`:

```rust
#[tool(name = "codegraph_rs_files", description = "List indexed files with per-file symbol counts. Optional root filter.")]
async fn files(&self, Parameters(p): Parameters<FilesParams>) -> Result<Json<FilesResponse>, ErrorData> {
    graph::files::list(&self.inner.conn, p.root.as_deref())
        .await
        .map(|files| Json(FilesResponse { files }))
        .map_err(internal_err)
}

#[tool(name = "codegraph_rs_search", description = "Search symbols by name via Neo4j FTS. Supports kind/root filters.")]
async fn search(&self, Parameters(p): Parameters<SearchParams>) -> Result<Json<SearchResponse>, ErrorData> {
    graph::search::search(
        &self.inner.conn,
        &p.query,
        p.kinds.as_deref(),
        p.root.as_deref(),
        p.limit.unwrap_or(10),
    )
    .await
    .map(|results| Json(SearchResponse { results }))
    .map_err(internal_err)
}

#[tool(name = "codegraph_rs_node", description = "Get full detail and source snippet for a symbol by qid.")]
async fn node(&self, Parameters(p): Parameters<NodeParams>) -> Result<Json<graph::node::NodeResponse>, ErrorData> {
    graph::node::find(&self.inner.conn, &self.inner.workspace_dir, &self.inner.cfg, &p.qid)
        .await
        .map(Json)
        .map_err(internal_err)
}

#[tool(name = "codegraph_rs_callers", description = "Functions that call this symbol up to `depth` hops (default 2).")]
async fn callers(&self, Parameters(p): Parameters<CallersParams>) -> Result<Json<CallersResponse>, ErrorData> {
    graph::traverse::callers(&self.inner.conn, &p.qid, p.depth.unwrap_or(2))
        .await
        .map(|callers| Json(CallersResponse { callers }))
        .map_err(internal_err)
}

#[tool(name = "codegraph_rs_callees", description = "Functions called by this symbol up to `depth` hops (default 2).")]
async fn callees(&self, Parameters(p): Parameters<CalleesParams>) -> Result<Json<CalleesResponse>, ErrorData> {
    graph::traverse::callees(&self.inner.conn, &p.qid, p.depth.unwrap_or(2))
        .await
        .map(|callees| Json(CalleesResponse { callees }))
        .map_err(internal_err)
}

#[tool(name = "codegraph_rs_impact", description = "Reverse-transitive closure across CALLS/REFERENCES/TYPE_OF (default depth 3).")]
async fn impact(&self, Parameters(p): Parameters<ImpactParams>) -> Result<Json<ImpactResponse>, ErrorData> {
    graph::traverse::impact(&self.inner.conn, &p.qid, p.depth.unwrap_or(3))
        .await
        .map(|impact| Json(ImpactResponse { impact }))
        .map_err(internal_err)
}
```

Add the small helper at module scope:
```rust
fn internal_err(e: anyhow::Error) -> ErrorData {
    ErrorData::internal_error(e.to_string(), None)
}
```

- [ ] **Step 3: Build + clippy + fmt + test**

Expected: 65 tests still pass.

- [ ] **Step 4: Commit**

```bash
git add src/mcp/mod.rs
git commit -m "feat(mcp): wire files/search/node/callers/callees/impact tools"
```

---

### Task 9: Integration test — in-process MCP

**Files:**
- Create: `tests/integration_mcp.rs`

Exercises every tool against a freshly-indexed tiny-rust workspace. Uses `tokio::io::duplex` for in-process JSON-RPC framing (per spec §811: "in-memory channel pair").

- [ ] **Step 1: Create the test**

```rust
//! Integration test for the MCP server: index tiny-rust, spin up the server
//! over duplex pipes, and exercise initialize + list_tools + each tool.
//! Gated on CODEGRAPH_RS_TEST_NEO4J=1.

use codegraph_rs::cli::{index, init::{self, InitArgs}};
use codegraph_rs::config::workspace::load_from_path;
use codegraph_rs::db::Connection;
use codegraph_rs::mcp::McpServer;
use neo4rs::query;
use rmcp::model::{CallToolRequestParam, ClientInfo, Implementation, ProtocolVersion};
use rmcp::service::ServiceExt;
use serde_json::json;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn mcp_exposes_graph_tools_against_tiny_rust() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("mcp-test-{}", std::process::id());

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

    let cfg = load_from_path(dir.path()).unwrap();
    let conn = Connection::open(
        &cfg.workspace.neo4j.uri,
        &cfg.workspace.neo4j.username,
        &pw,
        &cfg.workspace.neo4j.database,
    ).await.unwrap();

    // Capture the db name before moving cfg into the server.
    let db_name = cfg.workspace.neo4j.database.clone();

    let server = McpServer::new(conn, cfg, dir.path().to_path_buf());

    // Build paired duplex pipes: server side ↔ client side.
    // Each DuplexStream is AsyncRead + AsyncWrite; rmcp's `IntoTransport for (R, W)`
    // requires a tuple, so split each half into its read/write components.
    let (server_io, client_io) = tokio::io::duplex(64 * 1024);
    let (server_rx, server_tx) = tokio::io::split(server_io);
    let (client_rx, client_tx) = tokio::io::split(client_io);

    let server_running = server
        .serve((server_rx, server_tx))
        .await
        .expect("server serve");

    // Client side speaks MCP over the other half.
    let client_info = ClientInfo {
        protocol_version: ProtocolVersion::default(),
        capabilities: Default::default(),
        client_info: Implementation { name: "test".into(), version: "0".into() },
    };
    let client = client_info
        .serve((client_rx, client_tx))
        .await
        .expect("client init");

    // list_tools should return all 7.
    let tools = client.list_tools(Default::default()).await.expect("list_tools").tools;
    let names: Vec<&str> = tools.iter().map(|t| t.name.as_ref()).collect();
    for expected in [
        "codegraph_rs_status", "codegraph_rs_files", "codegraph_rs_search",
        "codegraph_rs_node", "codegraph_rs_callers", "codegraph_rs_callees",
        "codegraph_rs_impact",
    ] {
        assert!(names.contains(&expected), "tool `{expected}` missing; have {names:?}");
    }

    // status
    let status = client.call_tool(CallToolRequestParam {
        name: "codegraph_rs_status".into(),
        arguments: None,
    }).await.expect("status call");
    let body = status.structured_content.expect("status structured");
    assert_eq!(body["workspace"], json!(workspace_name));
    assert!(body["file_count"].as_i64().unwrap() >= 3);

    // search: find `library_entry`
    let search = client.call_tool(CallToolRequestParam {
        name: "codegraph_rs_search".into(),
        arguments: Some(json!({ "query": "library_entry" }).as_object().unwrap().clone()),
    }).await.expect("search call");
    let results = &search.structured_content.expect("search structured")["results"];
    let arr = results.as_array().expect("results array");
    assert!(arr.iter().any(|r| r["name"] == "library_entry"), "library_entry should match search");

    // Pick a qid for downstream tests.
    let library_qid = arr.iter()
        .find(|r| r["name"] == "library_entry")
        .and_then(|r| r["qid"].as_str())
        .expect("found qid")
        .to_string();

    // node
    let node = client.call_tool(CallToolRequestParam {
        name: "codegraph_rs_node".into(),
        arguments: Some(json!({ "qid": library_qid }).as_object().unwrap().clone()),
    }).await.expect("node call");
    let nbody = node.structured_content.expect("node structured");
    assert_eq!(nbody["node"]["name"], json!("library_entry"));
    assert!(nbody["source"].as_str().unwrap().contains("library_entry"));

    // callees: library_entry → helper
    let callees = client.call_tool(CallToolRequestParam {
        name: "codegraph_rs_callees".into(),
        arguments: Some(json!({ "qid": library_qid, "depth": 2 }).as_object().unwrap().clone()),
    }).await.expect("callees call");
    let cbody = callees.structured_content.expect("callees structured");
    let callees_arr = cbody["callees"].as_array().unwrap();
    assert!(callees_arr.iter().any(|c| c["name"] == "helper"), "library_entry should call helper");

    // callers: who calls `helper`? library_entry should (transitively at depth 1)
    let helper_qid = arr.iter()
        .find(|r| r["name"] == "helper")
        .and_then(|r| r["qid"].as_str())
        .map(str::to_string);
    if let Some(helper_qid) = helper_qid {
        let callers = client.call_tool(CallToolRequestParam {
            name: "codegraph_rs_callers".into(),
            arguments: Some(json!({ "qid": helper_qid, "depth": 2 }).as_object().unwrap().clone()),
        }).await.expect("callers call");
        let crs = callers.structured_content.expect("callers structured");
        let callers_arr = crs["callers"].as_array().unwrap();
        assert!(callers_arr.iter().any(|c| c["name"] == "library_entry"),
                "helper should be called by library_entry");

        // impact: with depth 3, helper's reverse-transitive closure should be non-empty
        let impact = client.call_tool(CallToolRequestParam {
            name: "codegraph_rs_impact".into(),
            arguments: Some(json!({ "qid": helper_qid, "depth": 3 }).as_object().unwrap().clone()),
        }).await.expect("impact call");
        let ibody = impact.structured_content.expect("impact structured");
        let impact_arr = ibody["impact"].as_array().unwrap();
        assert!(!impact_arr.is_empty(), "impact of helper should not be empty");
    }

    // files: 3 files in tiny-rust
    let files = client.call_tool(CallToolRequestParam {
        name: "codegraph_rs_files".into(),
        arguments: None,
    }).await.expect("files call");
    let fbody = files.structured_content.expect("files structured");
    let files_arr = fbody["files"].as_array().unwrap();
    assert_eq!(files_arr.len(), 3);

    // Clean shutdown
    drop(client);
    drop(server_running);

    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    // Cleanup database (db_name was captured before moving cfg into the server)
    let sys = Connection::system("bolt://localhost:7687", "neo4j", &pw).await.unwrap();
    sys.graph.execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
        .await.unwrap().next().await.ok();
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

> **rmcp 1.5 client + transport notes:**
> - `ClientInfo::serve(transport)` (via the `ServiceExt` trait) returns a running client handle. `ClientInfo` is a model struct with `protocol_version`, `capabilities`, and `client_info: Implementation`.
> - `client.call_tool(CallToolRequestParam { name, arguments }).await` returns `CallToolResult`. Structured JSON is on `.structured_content: Option<serde_json::Value>`. If a tool returns `Json<T>`, that payload appears in `structured_content`.
> - `client.list_tools(...).await.tools` is `Vec<Tool>` where `Tool.name` is the registered string.
> - `tokio::io::duplex` returns two `DuplexStream`s, each `AsyncRead + AsyncWrite`. The `IntoTransport for (R, W)` impl (gated on the `transport-async-rw` feature, added in Task 1) wants the read and write halves as a tuple; the test splits each half via `tokio::io::split` and passes the resulting tuple.

- [ ] **Step 2: Verify SKIP path**

```bash
cargo test --test integration_mcp -- --nocapture 2>&1 | tail -5
```

Expected: 1 test, SKIP, passes.

- [ ] **Step 3: Build + clippy + fmt + test**

Expected: 66 tests pass (65 + 1 new SKIP-passing integration test).

- [ ] **Step 4: Commit**

```bash
git add tests/integration_mcp.rs
git commit -m "test(mcp): in-process MCP integration test against tiny-rust"
```

---

### Task 10: README + slice-6-mcp tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update README Status section**

Replace:
```markdown
## Status

In development. Currently implements **Slice 5: Sync** —
workspace initialization, indexing, resolution, and incremental sync (diff-driven
add/modify/delete handling with global resolution rebuild). Vector embeddings,
MCP server, and Daml support land in subsequent slices.
```

With:
```markdown
## Status

In development. Currently implements **Slice 6: MCP (graph tools)** —
workspace initialization, indexing, resolution, incremental sync, and a
stdio MCP server exposing 7 graph-traversal tools (status, files, search,
node, callers, callees, impact). Semantic search, context bundles, and
Daml support land in subsequent slices.
```

Also update the Quick start section to mention `codegraph-rs serve`:
```markdown
codegraph-rs serve  # stdio MCP server for Claude Code (config it as an MCP server)
```

- [ ] **Step 2: Final test suite**

```bash
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected: 66 tests pass total. All clippy/fmt clean.

| Test binary | Count |
|---|---|
| `extraction_unit` | 31 |
| `resolution_unit` | 2 |
| `sync_unit` | 6 |
| `graph_unit` (new) | 4 |
| `config_parsing` | 13 |
| `cli_smoke` | 2 |
| `integration_db` (SKIP) | 2 |
| `integration_init` (SKIP) | 1 |
| `integration_status` (SKIP) | 1 |
| `integration_index` (SKIP) | 1 |
| `integration_resolve` (SKIP) | 1 |
| `integration_sync` (SKIP) | 1 |
| `integration_mcp` (new, SKIP) | 1 |
| **Total** | **66** |

- [ ] **Step 3: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 6"
git tag slice-6-mcp
```

- [ ] **Step 4: Verify**

```bash
git log --oneline | head -12
git tag --list
```

Expected tags: `slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`, `slice-6-mcp`.

---

## Slice 6 Done

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — 66 tests pass, all integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all 66 pass; the new `integration_mcp` test exercises all 7 tools end-to-end
- [ ] Manual: configure `codegraph-rs serve` as an MCP server in Claude Code; verify the 7 tools appear in the tool listing and the typical flow works (search → node → callers / callees / impact)
- [ ] `cli/status.rs` is now ~20 lines shorter (the count helpers moved to `graph::status`)
- [ ] No embeddings yet (next slice)
- [ ] No `context` / `explore` tools yet (after embeddings slice)

When all check, slice 6 is complete. **Next slice: embeddings** (`fastembed-rs` model loading, on-write embedding during extract, vector storage, and a `codegraph-rs reembed` command). Then a follow-up MCP slice adds `codegraph_rs_semantic_search`, `codegraph_rs_context`, and `codegraph_rs_explore` to complete the spec §10 surface.
