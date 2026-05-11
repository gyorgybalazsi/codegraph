# codegraph-rs Slice 5 — Sync (Incremental Updates)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `codegraph-rs sync` work. After this slice, re-running on an unchanged workspace is cheap; modifying/adding/deleting files updates only what changed; resolution edges remain consistent across edits. Skipping Slice 4 (vector embeddings) for now — sync is more practically useful and can be implemented without embedding plumbing.

**Architecture:** Diff-driven, per spec §9. For each workspace root: walk the filesystem with the `ignore` crate, BLAKE3-hash every source file, query Neo4j for the currently-indexed `(relative_path, content_hash)` pairs, then compute a three-bucket diff:
- **Added** — in filesystem, not in DB → extract + write
- **Changed** — in both, hashes differ → DETACH DELETE old subtree + UnresolvedRefs, then re-extract
- **Deleted** — in DB, not in filesystem → DETACH DELETE old subtree

After all per-file changes, run the global resolution rebuild from §7 (already implemented in Slice 3). The rebuild drops every `confidence`-bearing edge and recomputes from the surviving `UnresolvedRef` records, so cross-file references self-heal against new target qids.

**Tech Stack:** unchanged from Slice 3. `ignore` + `blake3` are already in Cargo.toml (from Slice 2). No new dependencies.

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §9 (Sync). Reuses §6 (extraction) and §7 (resolution) infrastructure end-to-end.

**Slice 3 baseline:** HEAD `97eb68b`, tagged `slice-3-resolution`. 54 tests pass.

---

## What sync does

For a workspace with N roots:

1. **Walk** each root with the `ignore` crate (same walker Slice 2 uses).
2. **Hash** each discovered file with BLAKE3 (`source_hash` helper from Slice 2 Task 4).
3. **Query** Neo4j for the currently-indexed File nodes' `(root_name, relative_path, content_hash)` triples.
4. **Diff** filesystem state vs DB state into `Added`, `Changed`, `Deleted` buckets.
5. **Delete** subtrees for `Changed` + `Deleted` files. Cypher (per spec §9):
   ```cypher
   MATCH (f:File {root_name: $root, relative_path: $rel})
   OPTIONAL MATCH (f)-[:CONTAINS*]->(child:Symbol)
   OPTIONAL MATCH (child)-[:HAS_REF]->(ref:UnresolvedRef)
   DETACH DELETE f, child, ref
   ```
6. **Extract** `Added` + `Changed` files (same orchestrator from Slice 2 Task 11).
7. **Write** the new nodes/edges/UnresolvedRefs (same `write_extraction` from Slice 2 Task 10).
8. **Resolve** globally (same `resolution::resolve` from Slice 3).

The resolve step at the end is what makes cross-file edges self-heal: even if a function's qid changed in a re-extracted file, the surviving `UnresolvedRef`s in other files still describe the original intent, and Phase B/C re-create the edges against the new target qids.

## Out of scope for Slice 5

- Watch-mode (`codegraph-rs sync --watch`) — not v1
- Git hook auto-sync — not v1
- Tree-sitter incremental reparse (`edit()` API) — v1 reparses whole-file
- Parallel-root sync — v1 iterates roots sequentially
- Locking against concurrent sync sessions — documented as "don't run two at once" (same caveat as `index`)
- Embeddings (Slice 4) — sync re-embedding will be added when Slice 4 lands

---

## File structure produced by this slice

```
codegraph-rs/
  src/
    lib.rs                      # add `pub mod sync;`
    cli/
      sync.rs                   # NEW: codegraph-rs sync command
      mod.rs                    # wire Sync variant -> sync::run
    sync/
      mod.rs                    # NEW: re-exports + `run()` entry point
      diff.rs                   # NEW: FileDiff types + diff algorithm (TDD)
    db/
      queries.rs                # add `list_files_in_root(root_name)`
      write.rs                  # add `delete_file_subtree(root_name, relative_path)`
  tests/
    sync_unit.rs                # NEW: diff algorithm tests
    integration_sync.rs         # NEW: gated; full index → modify → sync → verify
```

---

### Task 1: Module skeleton

**Files:**
- Modify: `src/lib.rs` (add `pub mod sync;`)
- Create: `src/sync/mod.rs`
- Create: `src/sync/diff.rs` (stub)

- [ ] **Step 1: Add `pub mod sync;` to src/lib.rs**

After the existing `pub mod resolution;`:

```rust
pub mod cli;
pub mod config;
pub mod db;
pub mod extraction;
pub mod resolution;
pub mod sync;
```

- [ ] **Step 2: Create src/sync/mod.rs**

```rust
//! Sync (incremental update) pipeline (spec §9).
//!
//! Compares the filesystem state of each workspace root against the indexed
//! state in Neo4j, applies per-file delete/extract operations, and runs the
//! global resolution rebuild.

pub mod diff;

use anyhow::Result;

use crate::db::Connection;

/// Run the full sync pipeline. Filled in by Task 5.
pub async fn run(_conn: &Connection) -> Result<()> {
    anyhow::bail!("sync::run: not yet implemented (filled in by Task 5)")
}
```

- [ ] **Step 3: Create src/sync/diff.rs**

```rust
//! FileDiff types and the diff algorithm. Filled in by Task 2.
```

- [ ] **Step 4: Build, clippy, fmt, run tests**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

All clean, 54 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/lib.rs src/sync/
git commit -m "chore(sync): module skeleton for slice 5"
```

---

### Task 2: `FileDiff` type + diff algorithm (TDD)

The diff algorithm is purely in-memory: take a map of `(relative_path, content_hash)` for the filesystem and another for the database; produce three vectors. This is the only logic-heavy piece of sync that's not just plumbing existing components together.

**Files:**
- Modify: `src/sync/diff.rs`
- Create: `tests/sync_unit.rs`

- [ ] **Step 1: Write failing tests**

Create `tests/sync_unit.rs`:

```rust
//! Unit tests for sync diff algorithm.

use codegraph_rs::sync::diff::{diff_files, FileDiff};
use std::collections::HashMap;

#[test]
fn diff_empty_state_is_empty() {
    let fs = HashMap::new();
    let db = HashMap::new();
    let d = diff_files(&fs, &db);
    assert!(d.added.is_empty());
    assert!(d.changed.is_empty());
    assert!(d.deleted.is_empty());
}

#[test]
fn diff_added_files() {
    let fs: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_a".to_string()),
        ("b.rs".to_string(), "hash_b".to_string()),
    ].into_iter().collect();
    let db = HashMap::new();
    let d = diff_files(&fs, &db);
    let mut added = d.added.clone();
    added.sort();
    assert_eq!(added, vec!["a.rs", "b.rs"]);
    assert!(d.changed.is_empty());
    assert!(d.deleted.is_empty());
}

#[test]
fn diff_deleted_files() {
    let fs = HashMap::new();
    let db: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_a".to_string()),
    ].into_iter().collect();
    let d = diff_files(&fs, &db);
    assert!(d.added.is_empty());
    assert!(d.changed.is_empty());
    assert_eq!(d.deleted, vec!["a.rs"]);
}

#[test]
fn diff_changed_files() {
    let fs: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_v2".to_string()),
    ].into_iter().collect();
    let db: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_v1".to_string()),
    ].into_iter().collect();
    let d = diff_files(&fs, &db);
    assert!(d.added.is_empty());
    assert_eq!(d.changed, vec!["a.rs"]);
    assert!(d.deleted.is_empty());
}

#[test]
fn diff_unchanged_files_appear_in_no_bucket() {
    let fs: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_a".to_string()),
    ].into_iter().collect();
    let db: HashMap<String, String> = [
        ("a.rs".to_string(), "hash_a".to_string()),
    ].into_iter().collect();
    let d = diff_files(&fs, &db);
    assert!(d.added.is_empty(), "unchanged file should not appear in any bucket: {d:?}");
    assert!(d.changed.is_empty());
    assert!(d.deleted.is_empty());
}

#[test]
fn diff_mixed_buckets() {
    let fs: HashMap<String, String> = [
        ("kept.rs".to_string(), "h1".to_string()),
        ("changed.rs".to_string(), "h2_new".to_string()),
        ("added.rs".to_string(), "h3".to_string()),
    ].into_iter().collect();
    let db: HashMap<String, String> = [
        ("kept.rs".to_string(), "h1".to_string()),
        ("changed.rs".to_string(), "h2_old".to_string()),
        ("deleted.rs".to_string(), "h4".to_string()),
    ].into_iter().collect();
    let d = diff_files(&fs, &db);
    assert_eq!(d.added, vec!["added.rs"]);
    assert_eq!(d.changed, vec!["changed.rs"]);
    assert_eq!(d.deleted, vec!["deleted.rs"]);
}
```

- [ ] **Step 2: Run failing tests**

```bash
cargo test --test sync_unit 2>&1 | tail -10
```

Expected: compile errors about missing `diff_files`, `FileDiff`.

- [ ] **Step 3: Implement the diff**

Overwrite `src/sync/diff.rs`:

```rust
//! FileDiff types and the diff algorithm.
//!
//! Pure functions over (path, content_hash) maps — no I/O. Slice 5 Task 5
//! wraps this with filesystem walking + Neo4j querying.

use std::collections::HashMap;

#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct FileDiff {
    /// Files present in the filesystem but not in the DB.
    pub added: Vec<String>,
    /// Files in both but with different content hashes.
    pub changed: Vec<String>,
    /// Files present in the DB but not in the filesystem.
    pub deleted: Vec<String>,
}

/// Compute the three-bucket diff. Inputs are maps of `relative_path -> content_hash`.
pub fn diff_files(
    filesystem: &HashMap<String, String>,
    database: &HashMap<String, String>,
) -> FileDiff {
    let mut diff = FileDiff::default();

    for (path, fs_hash) in filesystem {
        match database.get(path) {
            None => diff.added.push(path.clone()),
            Some(db_hash) if db_hash != fs_hash => diff.changed.push(path.clone()),
            Some(_) => { /* unchanged — no bucket */ }
        }
    }

    for path in database.keys() {
        if !filesystem.contains_key(path) {
            diff.deleted.push(path.clone());
        }
    }

    // Deterministic order makes test assertions stable and helps debugging.
    diff.added.sort();
    diff.changed.sort();
    diff.deleted.sort();
    diff
}
```

- [ ] **Step 4: Run tests, fmt, clippy**

```bash
cargo test --test sync_unit 2>&1 | tail -10
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

Expected: 6 tests pass. fmt/clippy clean.

- [ ] **Step 5: Commit**

```bash
git add src/sync/diff.rs tests/sync_unit.rs
git commit -m "feat(sync): add FileDiff types and diff_files algorithm"
```

---

### Task 3: `db::queries::list_files_in_root`

**Files:**
- Modify: `src/db/queries.rs`

Adds one query helper that fetches all `File` nodes for a given root, returning `(relative_path, content_hash)` pairs. Used by the sync orchestrator (Task 5) to populate the DB-side of the diff.

- [ ] **Step 1: Append to src/db/queries.rs**

After the existing `file_qid_for_source` function, append:

```rust
/// List all `(:File)` nodes in the given workspace root with their content hashes.
/// Returns a HashMap of `relative_path -> content_hash`. Files without a
/// `content_hash` property are skipped (shouldn't happen for slice-2-indexed
/// files but defensive against partial schema migrations).
pub async fn list_files_in_root(
    conn: &Connection,
    root_name: &str,
) -> Result<std::collections::HashMap<String, String>> {
    let mut stream = conn
        .graph
        .execute(
            cypher(
                "MATCH (f:File {root_name: $root}) \
                 WHERE f.content_hash IS NOT NULL \
                 RETURN f.relative_path AS path, f.content_hash AS hash",
            )
            .param("root", root_name.to_string()),
        )
        .await
        .context("list_files_in_root")?;
    let mut out = std::collections::HashMap::new();
    while let Some(row) = stream.next().await.context("row stream")? {
        let path: String = row.get("path").context("path")?;
        let hash: String = row.get("hash").context("hash")?;
        out.insert(path, hash);
    }
    Ok(out)
}
```

- [ ] **Step 2: Build, clippy, fmt, test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 60 tests pass (54 prior + 6 from Task 2 sync_unit).

- [ ] **Step 3: Commit**

```bash
git add src/db/queries.rs
git commit -m "feat(db): add list_files_in_root query helper"
```

---

### Task 4: `db::write::delete_file_subtree`

**Files:**
- Modify: `src/db/write.rs`

Adds the per-file deletion helper used by sync for Changed and Deleted files. Reaches the File, all CONTAINS-children Symbols, and the HAS_REF UnresolvedRef nodes hanging off those Symbols — exactly the subtree to delete on a file change.

- [ ] **Step 1: Append to src/db/write.rs**

After the existing `to_bolt` function, append:

```rust
/// Delete a File and everything it transitively contains, including the
/// UnresolvedRef records attached to its Symbol children via HAS_REF.
///
/// Per spec §9: cross-file `UnresolvedRef` nodes (whose `source_qid` lives in
/// a DIFFERENT file but happen to name a symbol in this file) are NOT touched.
/// The global resolution rebuild that follows sync recreates cross-file edges
/// against the new target qids.
pub async fn delete_file_subtree(
    conn: &Connection,
    root_name: &str,
    relative_path: &str,
) -> Result<()> {
    conn.graph
        .execute(
            neo4rs::query(
                "MATCH (f:File {root_name: $root, relative_path: $rel}) \
                 OPTIONAL MATCH (f)-[:CONTAINS*]->(child:Symbol) \
                 OPTIONAL MATCH (child)-[:HAS_REF]->(ref:UnresolvedRef) \
                 DETACH DELETE f, child, ref",
            )
            .param("root", root_name.to_string())
            .param("rel", relative_path.to_string()),
        )
        .await
        .with_context(|| {
            format!("delete_file_subtree(root={root_name}, rel={relative_path})")
        })?
        .next()
        .await
        .ok();
    Ok(())
}
```

> Note: imports at the top of `write.rs` already include `Context`, `Result`, `neo4rs::query as cypher`, and `Connection`. If `delete_file_subtree`'s body uses `neo4rs::query` directly (not the `cypher` alias) for clarity, that's fine — both are valid.

- [ ] **Step 2: Build, clippy, fmt, test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 60 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/db/write.rs
git commit -m "feat(db): add delete_file_subtree for sync's per-file cleanup"
```

---

### Task 5: Sync orchestrator

**Files:**
- Modify: `src/sync/mod.rs`

Replaces the stub `run()` with the real sync orchestrator. Walks roots, hashes, diffs, deletes, extracts, writes, resolves.

- [ ] **Step 1: Implement**

Overwrite `src/sync/mod.rs`:

```rust
//! Sync (incremental update) pipeline (spec §9).
//!
//! Compares the filesystem state of each workspace root against the indexed
//! state in Neo4j, applies per-file delete/extract operations, and runs the
//! global resolution rebuild.

pub mod diff;

use anyhow::{Context, Result};
use std::collections::HashMap;
use tracing::{info, warn};

use crate::config::workspace::WorkspaceConfig;
use crate::db::queries::list_files_in_root;
use crate::db::write::{delete_file_subtree, write_extraction};
use crate::db::Connection;
use crate::extraction::hash::source_hash;
use crate::extraction::languages::rust::extract_rust;
use crate::extraction::result::ExtractionResult;
use crate::extraction::walker::walk_root;
use crate::resolution;
use crate::resolution::ResolutionStats;
use diff::{diff_files, FileDiff};

/// One per-root sync outcome.
#[derive(Debug, Clone, Default)]
pub struct RootSyncStats {
    pub root_name: String,
    pub added: usize,
    pub changed: usize,
    pub deleted: usize,
}

/// Aggregate sync stats for the whole workspace.
#[derive(Debug, Default)]
pub struct SyncStats {
    pub per_root: Vec<RootSyncStats>,
    pub resolution: ResolutionStats,
}

impl SyncStats {
    pub fn total_added(&self) -> usize { self.per_root.iter().map(|r| r.added).sum() }
    pub fn total_changed(&self) -> usize { self.per_root.iter().map(|r| r.changed).sum() }
    pub fn total_deleted(&self) -> usize { self.per_root.iter().map(|r| r.deleted).sum() }
}

/// Run the full sync pipeline against the workspace's Neo4j database.
pub async fn run(conn: &Connection, cfg: &WorkspaceConfig) -> Result<SyncStats> {
    let languages = &cfg.indexing.languages;
    let mut stats = SyncStats::default();
    let mut all_new_results: Vec<ExtractionResult> = Vec::new();

    for root in &cfg.workspace.roots {
        info!("syncing root `{}`", root.name);

        // Filesystem state: walk + hash.
        let fs_state: HashMap<String, String> = walk_root(&root.path, &root.name, languages)
            .map(|f| {
                let source = std::fs::read_to_string(&f.absolute_path)
                    .with_context(|| format!("reading {}", f.absolute_path.display()))?;
                Ok((f.relative_path.clone(), source_hash(&source)))
            })
            .collect::<Result<HashMap<_, _>>>()?;

        // DB state for this root.
        let db_state = list_files_in_root(conn, &root.name).await
            .with_context(|| format!("list_files_in_root({})", root.name))?;

        let diff: FileDiff = diff_files(&fs_state, &db_state);
        info!(
            "root `{}`: added={}, changed={}, deleted={}",
            root.name,
            diff.added.len(),
            diff.changed.len(),
            diff.deleted.len()
        );

        // Delete changed + deleted file subtrees.
        for rel in diff.changed.iter().chain(diff.deleted.iter()) {
            delete_file_subtree(conn, &root.name, rel).await
                .with_context(|| format!("delete_file_subtree({}, {})", root.name, rel))?;
        }

        // Re-extract added + changed files.
        for rel in diff.added.iter().chain(diff.changed.iter()) {
            let abs = root.path.join(rel);
            let source = match std::fs::read_to_string(&abs) {
                Ok(s) => s,
                Err(e) => {
                    warn!("read failed for {}: {e:#}", abs.display());
                    continue;
                }
            };
            // Slice 5 only supports Rust here. Slice 7 adds Daml dispatch.
            // Source-extension routing matches the walker's `language` field.
            let result = match abs.extension().and_then(|s| s.to_str()) {
                Some("rs") => extract_rust(&source, &root.name, rel),
                Some(other) => {
                    warn!("no extractor for `.{other}`, skipping {}", rel);
                    continue;
                }
                None => continue,
            };
            match result {
                Ok(er) => all_new_results.push(er),
                Err(e) => warn!("extraction failed for {}: {e:#}", abs.display()),
            }
        }

        stats.per_root.push(RootSyncStats {
            root_name: root.name.clone(),
            added: diff.added.len(),
            changed: diff.changed.len(),
            deleted: diff.deleted.len(),
        });
    }

    // Write all new extraction results in one batch (across roots).
    if !all_new_results.is_empty() {
        write_extraction(conn, all_new_results).await.context("write_extraction")?;
    }

    // Always run resolution rebuild: even pure deletions invalidate confidence edges.
    let resolution_stats = resolution::resolve(conn).await.context("resolve")?;
    stats.resolution = resolution_stats;

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

> Note on the language dispatch — sync's per-file extraction has a small switch that mirrors the orchestrator's. Slice 7 (Daml) will need to update both call sites. Cleaner refactor: extract a `dispatch_by_language(lang, &source, root_name, relative_path) -> Result<ExtractionResult>` helper and call it from both places. Defer to slice 7 unless trivial to do now.

- [ ] **Step 2: Build, clippy, fmt, test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 60 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/sync/mod.rs
git commit -m "feat(sync): orchestrator — walk/hash/diff/delete/extract/resolve"
```

---

### Task 6: Wire `codegraph-rs sync` CLI command

**Files:**
- Create: `src/cli/sync.rs`
- Modify: `src/cli/mod.rs`

- [ ] **Step 1: Create src/cli/sync.rs**

```rust
//! `codegraph-rs sync` — incremental update.

use anyhow::{Context, Result};
use std::path::Path;
use tracing::info;

use crate::config::secrets::resolve_neo4j_password;
use crate::config::workspace;
use crate::db::{schema::apply_schema, Connection};
use crate::sync;

pub fn run(workspace_arg: Option<&Path>) -> Result<()> {
    let starting = workspace_arg.map(Path::to_path_buf).unwrap_or_else(|| {
        std::env::current_dir().expect("cwd readable")
    });
    let workspace_dir = workspace::find_workspace_root(&starting)
        .context("no workspace found; run `codegraph-rs init` first")?;
    let cfg = workspace::load_from_path(&workspace_dir)?;
    let pw = resolve_neo4j_password(&workspace_dir, None)?;

    info!("syncing workspace `{}`", cfg.workspace.name);

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

        // Schema is idempotent; ensure it's present (defensive — should already be).
        apply_schema(&conn).await.context("applying schema")?;

        sync::run(&conn, &cfg).await.context("sync")
    })?;

    println!(
        "Synced workspace `{}`: added={}, changed={}, deleted={}",
        cfg.workspace.name,
        stats.total_added(),
        stats.total_changed(),
        stats.total_deleted(),
    );
    Ok(())
}
```

- [ ] **Step 2: Wire dispatch in src/cli/mod.rs**

Add `pub mod sync;` alongside `pub mod index;`:

```rust
pub mod index;
pub mod init;
pub mod status;
pub mod sync;
```

Replace the existing combined stub arm:
```rust
        Command::Sync | Command::Query { .. } | Command::Serve => {
            anyhow::bail!("not yet implemented");
        }
```

With:
```rust
        Command::Sync => sync::run(cli.workspace.as_deref()),
        Command::Query { .. } | Command::Serve => {
            anyhow::bail!("not yet implemented");
        }
```

- [ ] **Step 3: Update the cli_smoke test**

In `tests/cli_smoke.rs`, the `stub_commands_fail_with_not_implemented` test iterates over `["sync", "serve"]`. Remove `"sync"` from the list — it's now a real command and will produce a different error message (likely "no workspace found").

```rust
    for sub in &["serve"] {
```

- [ ] **Step 4: Build, clippy, fmt, test**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
cargo test 2>&1 | tail -5
```

Expected: 60 tests pass. The cli_smoke test still passes with its reduced loop.

- [ ] **Step 5: Commit**

```bash
git add src/cli/sync.rs src/cli/mod.rs tests/cli_smoke.rs
git commit -m "feat(cli): wire up sync command"
```

---

### Task 7: Integration test — full sync lifecycle

**Files:**
- Create: `tests/integration_sync.rs` (gated)

Exercises the full sync lifecycle against live Neo4j:
1. Index tiny-rust → verify initial state
2. Add a new file → sync → verify new file's symbols appear
3. Modify an existing file (remove a call) → sync → verify CALLS edge count drops
4. Delete a file → sync → verify the file's symbols are gone

- [ ] **Step 1: Create the integration test**

Create `tests/integration_sync.rs`:

```rust
//! Integration test for sync. Walks through a realistic sequence:
//!  1) full index
//!  2) add a new file -> sync
//!  3) modify an existing file -> sync
//!  4) delete a file -> sync
//! Gated on CODEGRAPH_RS_TEST_NEO4J=1.

use codegraph_rs::cli::{index, init::{InitArgs, self}, sync as cli_sync};
use neo4rs::query;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn sync_handles_add_modify_delete() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("sync-test-{}", std::process::id());

    let fixture_src = std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
        .join("tests/fixtures/tiny-rust");
    let fixture_dst = dir.path().join("tiny-rust");
    copy_dir(&fixture_src, &fixture_dst).unwrap();

    // Init + initial index.
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
    let db_name = format!("codegraph-sync-test-{}", std::process::id());

    // Capture initial counts.
    let initial_file_count = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:File) RETURN count(n) AS c"));
    let initial_func_count = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:Function) RETURN count(n) AS c"));
    let initial_calls = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH ()-[r:CALLS]->() RETURN count(r) AS c"));
    assert_eq!(initial_file_count, 3, "tiny-rust has 3 .rs files");
    assert!(initial_func_count >= 4, "expected ≥4 Functions, got {initial_func_count}");
    assert!(initial_calls >= 3, "expected ≥3 CALLS, got {initial_calls}");

    // ── Step 1: Add a new file ──────────────────────────────────────────
    fs::write(
        fixture_dst.join("src/extra.rs"),
        "pub fn brand_new_function() -> u32 { 99 }\n",
    ).unwrap();
    cli_sync::run(Some(dir.path())).expect("sync after add must succeed");

    let file_count = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:File) RETURN count(n) AS c"));
    assert_eq!(file_count, 4, "expected File count to grow by 1 after adding extra.rs");

    let new_fn_exists = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:Function {name: 'brand_new_function'}) RETURN count(n) AS c"));
    assert_eq!(new_fn_exists, 1, "brand_new_function should be indexed");

    // ── Step 2: Modify an existing file (remove a CALLS source) ─────────
    fs::write(
        fixture_dst.join("src/lib.rs"),
        // Remove the `util::helper();` call from library_entry
        "pub mod util;\n\npub fn library_entry() {\n    // no calls anymore\n}\n",
    ).unwrap();
    cli_sync::run(Some(dir.path())).expect("sync after modify must succeed");

    let calls_after_modify = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH ()-[r:CALLS]->() RETURN count(r) AS c"));
    assert!(
        calls_after_modify < initial_calls,
        "CALLS count should have dropped (was {initial_calls}, now {calls_after_modify})"
    );

    // library_entry → helper should be gone specifically.
    let library_to_helper = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (:Function {name: 'library_entry'})-[:CALLS]->(:Function {name: 'helper'}) RETURN count(*) AS c"));
    assert_eq!(library_to_helper, 0, "library_entry → helper CALLS edge should be gone after modify");

    // ── Step 3: Delete a file ───────────────────────────────────────────
    fs::remove_file(fixture_dst.join("src/util.rs")).unwrap();
    cli_sync::run(Some(dir.path())).expect("sync after delete must succeed");

    let file_count_after_delete = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:File) RETURN count(n) AS c"));
    assert_eq!(
        file_count_after_delete, 3,
        "expected File count 3 (extra.rs + lib.rs + main.rs) after deleting util.rs"
    );

    let util_funcs = rt.block_on(scalar_async(&pw, &db_name,
        "MATCH (n:Function) WHERE n.name IN ['helper', 'constant'] RETURN count(n) AS c"));
    assert_eq!(util_funcs, 0, "helper/constant should be gone after util.rs deletion");

    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    // ── Cleanup ─────────────────────────────────────────────────────────
    rt.block_on(async {
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        sys.graph.execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
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

> Note: each `scalar_async` opens its own connection. Inefficient but simple — the test runs sequentially with `--test-threads=1`. Cleanup at the end uses a system connection.

- [ ] **Step 2: Verify SKIP path**

```bash
cargo test --test integration_sync -- --nocapture 2>&1 | tail -10
```

Expected: 1 test, SKIP, passes.

- [ ] **Step 3: Build, clippy, fmt**

```bash
cargo build 2>&1 | tail -3
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

All clean.

- [ ] **Step 4: Run all tests**

```bash
cargo test 2>&1 | tail -15
```

Expected: 61 total tests (60 from prior tasks + 1 new SKIP-passing integration test).

- [ ] **Step 5: Commit**

```bash
git add tests/integration_sync.rs
git commit -m "test(sync): integration test for add/modify/delete on tiny-rust"
```

---

### Task 8: README + slice-5 tag

**Files:**
- Modify: `/Users/gyorgybalazsi/codegraph-rs/README.md`

- [ ] **Step 1: Update README Status section**

Replace the current Status section with:

```markdown
## Status

In development. Currently implements **Slice 5: Sync** —
workspace initialization, indexing, resolution, and incremental sync (diff-driven
add/modify/delete handling with global resolution rebuild). Vector embeddings,
MCP server, and Daml support land in subsequent slices.
```

(Adjust the wording if the current section uses a different format.)

- [ ] **Step 2: Final test suite**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo test 2>&1 | tail -20
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Expected:

| Test binary | Count |
|---|---|
| `extraction_unit` | 31 |
| `resolution_unit` | 2 |
| `sync_unit` (new) | 6 |
| `config_parsing` | 13 |
| `cli_smoke` | 2 |
| `integration_db` (SKIP) | 2 |
| `integration_init` (SKIP) | 1 |
| `integration_status` (SKIP) | 1 |
| `integration_index` (SKIP) | 1 |
| `integration_resolve` (SKIP) | 1 |
| `integration_sync` (new, SKIP) | 1 |
| **Total** | **61** |

All clippy/fmt clean.

- [ ] **Step 3: Commit and tag**

```bash
git add README.md
git commit -m "docs: update README for slice 5"
git tag slice-5-sync
```

- [ ] **Step 4: Verify**

```bash
git log --oneline | head -5
git tag --list
```

Expected: 4 slice tags now (`slice-1-walking-skeleton`, `slice-2-rust-extraction`, `slice-3-resolution`, `slice-5-sync`). No `slice-4` — we skipped embeddings; will come later.

---

## Slice 5 Done

Verify the artifact:

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — 61 tests pass, integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all 61 pass
- [ ] Manual: `codegraph-rs sync` runs against an already-initialized workspace; idempotent (running twice on unchanged workspace produces 0 added/changed/deleted)
- [ ] Modifying a file and re-running `sync` updates only that file's subgraph; resolution edges re-emerge
- [ ] No embeddings yet (Slice 4 deferred)
- [ ] No MCP server yet (Slice 6)

When all check, slice 5 is complete and we can write the next slice. Likely Slice 4 (vector embeddings) next, then Slice 6 (MCP) so the embeddings get a consumer.
