# codegraph-rs Slice 3 — Resolution

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the persistent `UnresolvedRef` records from Slice 2 into real graph edges. After this slice, `codegraph-rs index` runs extraction followed by resolution; the resulting graph has `(:Function)-[:CALLS]->(:Function)`, `(:File)-[:IMPORTS]->(:Module)`, and `(:Symbol)-[:REFERENCES]->(:Symbol)` edges with a `confidence` property, plus per-ref `unresolved_reason` and `candidate_qids` for unresolvable cases.

**Architecture:** A new top-level `resolution/` module with a single public entry point `resolve(conn) -> Result<ResolutionStats>`. Three phases per spec §7:
- **Phase A** — process every `(:Import)` Symbol node, look up the named module in the workspace, create `(:File)-[:IMPORTS]->(:Module)` edges. Best-effort; many imports point at external crates (no Module node exists) and remain unresolved.
- **Phase B** — for `UnresolvedRef`s with `kind="call"|"type_ref"|"reference"`, walk the source file's `IMPORTS` edges; candidates are `(:Module)-[:CONTAINS]->(symbol)` and `(:Module)-[:EXPORTS]->(symbol)` matching by name. Single candidate → resolution edge with `confidence: "import_resolved"`.
- **Phase C** — fallback for refs Phase B didn't resolve: exact name-match against `qualified_name` across the entire workspace. Single candidate → edge with `confidence: "name_match"`. Multiple candidates → record `candidate_qids` + `unresolved_reason = "ambiguous"`. Zero → `unresolved_reason = "no_candidate"`.

Resolution is **idempotent**: each `resolve()` call drops all existing `confidence`-bearing edges first, resets ref state, then rebuilds. Structural edges (`CONTAINS`, `HAS_REF`) are untouched.

**Tech Stack:** unchanged from Slice 2 (neo4rs 0.8 with `json` feature, anyhow, tokio). No new crates.

**Source spec:** [`docs/superpowers/specs/2026-04-22-codegraph-rs-design.md`](../specs/2026-04-22-codegraph-rs-design.md) §7 (Resolution pipeline) — fully implemented here.

**Slice 2 baseline:** HEAD of `codegraph-rs` is `097b38b` (tagged `slice-2-rust-extraction`). Slice 3 builds on it.

---

## What gets resolved in Slice 3

From the existing graph state produced by `codegraph-rs index` (Slice 2):

| Input | Phase | Output |
|---|---|---|
| `(:Import)` Symbol nodes (with `name` containing the use path) | A | `(:File)-[:IMPORTS]->(:Module)` edge, `confidence: "import_resolved"` — only when target Module exists in the workspace |
| `(:UnresolvedRef {kind: "call"})` | B then C | `(:Function|:Method)-[:CALLS]->(:Function|:Method)`, `confidence: "import_resolved"` or `"name_match"` |
| `(:UnresolvedRef {kind: "type_ref"})` | B then C | `-[:TYPE_OF]->()` edges |
| `(:UnresolvedRef {kind: "reference"})` | B then C | `-[:REFERENCES]->()` edges |
| Refs unresolved after C | — | `unresolved_reason` set on the ref (`"no_candidate"` or `"ambiguous"`); `candidate_qids` populated if ambiguous |

**Out of scope for Slice 3** (intentionally):
- Multi-hop path resolution (Phase B only walks one hop through IMPORTS — no transitive crate resolution)
- Framework-aware resolution (Rails, Express, React hooks, etc.)
- Type-aware overload resolution (trait dispatch, generic instantiation)
- Resolving references to external crates (stdlib, registry crates) — the corresponding Module nodes don't exist
- Cross-language resolution (we only have Rust in slice 2; Daml lands in slice 7)

## Known limitation: leaf-name resolution

Slice 2's call-site extraction discards the path prefix on multi-segment callees: `util::helper()` becomes `unresolved_name = "helper"`, not `"util::helper"`. This means Phase B/C resolve by leaf name only. For `tiny-rust`'s simple structure this works fine (each function name is globally unique within the workspace). Real codebases will have name collisions resolved as `unresolved_reason = "ambiguous"`.

A future slice can extend the Rust extractor to preserve the path; Slice 3's resolver will trivially benefit since the lookup logic doesn't care which form the name takes.

---

## File structure produced by this slice

```
codegraph-rs/
  src/
    lib.rs                          # add `pub mod resolution;`
    cli/
      index.rs                      # call resolve() after write_extraction
    db/
      mod.rs                        # add `pub mod queries;`
      queries.rs                    # NEW: read-side helpers (find modules, candidates, etc.)
    resolution/
      mod.rs                        # NEW: re-exports + the `resolve()` entry point
      stats.rs                      # NEW: ResolutionStats type
      drop.rs                       # NEW: drop existing confidence-bearing edges
      phase_a_imports.rs            # NEW: resolve Import nodes -> IMPORTS edges
      phase_b_import_driven.rs      # NEW: import-driven candidate lookup
      phase_c_name_match.rs         # NEW: workspace-wide name match
  tests/
    resolution_unit.rs              # in-memory tests for the candidate-selection logic
    integration_resolve.rs          # gated; runs index+resolve on tiny-rust, asserts CALLS edges
```

---

### Task 1: Module skeleton

**Files:**
- Modify: `src/lib.rs` (add `pub mod resolution;`)
- Modify: `src/db/mod.rs` (add `pub mod queries;`)
- Create: `src/db/queries.rs` (stub)
- Create: `src/resolution/mod.rs`
- Create: `src/resolution/stats.rs` (stub)
- Create: `src/resolution/drop.rs` (stub)
- Create: `src/resolution/phase_a_imports.rs` (stub)
- Create: `src/resolution/phase_b_import_driven.rs` (stub)
- Create: `src/resolution/phase_c_name_match.rs` (stub)

- [ ] **Step 1: Create the module skeleton**

Add `pub mod resolution;` to `src/lib.rs` after `pub mod extraction;`:

```rust
pub mod cli;
pub mod config;
pub mod db;
pub mod extraction;
pub mod resolution;
```

Add `pub mod queries;` to `src/db/mod.rs` after `pub mod write;`:

```rust
//! Neo4j Bolt client and schema management.

pub mod connection;
pub mod queries;
pub mod schema;
pub mod write;

pub use connection::Connection;
```

Create `src/db/queries.rs`:

```rust
//! Read-side Cypher queries used by resolution. Filled in by Task 3.
```

Create `src/resolution/mod.rs`:

```rust
//! Resolution pipeline (spec §7).
//!
//! Turns `(:UnresolvedRef)` records into `(:Symbol)-[:CALLS|REFERENCES|TYPE_OF]->(:Symbol)`
//! edges and `(:File)-[:IMPORTS]->(:Module)` edges, each with a `confidence`
//! property. Idempotent: each `resolve()` call drops existing confidence-bearing
//! edges and rebuilds.

pub mod drop;
pub mod phase_a_imports;
pub mod phase_b_import_driven;
pub mod phase_c_name_match;
pub mod stats;

pub use stats::ResolutionStats;

use anyhow::Result;
use crate::db::Connection;

/// Run the full resolution pipeline. Filled in by Task 8.
pub async fn resolve(_conn: &Connection) -> Result<ResolutionStats> {
    anyhow::bail!("resolve: not yet implemented (filled in by Task 8)")
}
```

Create `src/resolution/stats.rs`:

```rust
//! ResolutionStats — counters returned by `resolve()`. Filled in by Task 2.
```

Create the three phase stubs (`src/resolution/phase_a_imports.rs`, `phase_b_import_driven.rs`, `phase_c_name_match.rs`) each containing only a one-line doc comment:

```rust
//! Phase A: resolve Import nodes -> IMPORTS edges. Filled in by Task 5.
```

(For the b and c files, change "A" / "Imports" / "Task 5" to match.)

Create `src/resolution/drop.rs`:

```rust
//! Drop all existing confidence-bearing edges. Filled in by Task 4.
```

- [ ] **Step 2: Build**

```bash
cd /Users/gyorgybalazsi/codegraph-rs
cargo build 2>&1 | tail -5
```

Expected: clean. The `pub use stats::ResolutionStats;` line in `resolution/mod.rs` will fail because `ResolutionStats` doesn't exist yet (Task 2 adds it). To make the skeleton build, COMMENT OUT that line temporarily — Task 2 will uncomment it:

```rust
// pub use stats::ResolutionStats;  // Task 2 uncomments after defining the type
```

Re-run `cargo build` — it should now succeed.

- [ ] **Step 3: Run existing tests to confirm no regressions**

```bash
cargo test 2>&1 | tail -10
```

Expected: 51 tests pass.

- [ ] **Step 4: cargo fmt + clippy**

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

Both clean.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "chore(resolution): module skeleton for slice 3"
```

---

### Task 2: `ResolutionStats` type (TDD)

**Files:**
- Modify: `src/resolution/stats.rs`
- Modify: `src/resolution/mod.rs` (uncomment the `pub use`)
- Create: `tests/resolution_unit.rs`

- [ ] **Step 1: Write the failing test**

Create `tests/resolution_unit.rs`:

```rust
//! Unit tests for the resolution module.

use codegraph_rs::resolution::ResolutionStats;

#[test]
fn resolution_stats_default_is_zero() {
    let s = ResolutionStats::default();
    assert_eq!(s.imports_resolved, 0);
    assert_eq!(s.calls_resolved_phase_b, 0);
    assert_eq!(s.calls_resolved_phase_c, 0);
    assert_eq!(s.refs_resolved_phase_b, 0);
    assert_eq!(s.refs_resolved_phase_c, 0);
    assert_eq!(s.types_resolved_phase_b, 0);
    assert_eq!(s.types_resolved_phase_c, 0);
    assert_eq!(s.unresolved_ambiguous, 0);
    assert_eq!(s.unresolved_no_candidate, 0);
}

#[test]
fn resolution_stats_total_resolved() {
    let s = ResolutionStats {
        imports_resolved: 5,
        calls_resolved_phase_b: 3,
        calls_resolved_phase_c: 7,
        refs_resolved_phase_b: 1,
        refs_resolved_phase_c: 2,
        types_resolved_phase_b: 0,
        types_resolved_phase_c: 4,
        unresolved_ambiguous: 6,
        unresolved_no_candidate: 8,
    };
    assert_eq!(s.total_resolved(), 22);
    assert_eq!(s.total_unresolved(), 14);
}
```

- [ ] **Step 2: Run tests — expect compile failures**

```bash
cargo test --test resolution_unit 2>&1 | tail -10
```

Expected: errors about missing `ResolutionStats`.

- [ ] **Step 3: Implement the type**

Overwrite `src/resolution/stats.rs`:

```rust
//! `ResolutionStats` — counters returned by `resolve()`.
//!
//! Separated by phase so an operator can see which mechanism is doing the
//! work (Phase B = high-confidence import-driven, Phase C = lower-confidence
//! name match).

#[derive(Debug, Default, Clone, Copy, PartialEq, Eq)]
pub struct ResolutionStats {
    /// `(:File)-[:IMPORTS]->(:Module)` edges created by Phase A.
    pub imports_resolved: usize,

    pub calls_resolved_phase_b: usize,
    pub calls_resolved_phase_c: usize,

    pub refs_resolved_phase_b: usize,
    pub refs_resolved_phase_c: usize,

    pub types_resolved_phase_b: usize,
    pub types_resolved_phase_c: usize,

    /// UnresolvedRef with `unresolved_reason = "ambiguous"`.
    pub unresolved_ambiguous: usize,
    /// UnresolvedRef with `unresolved_reason = "no_candidate"`.
    pub unresolved_no_candidate: usize,
}

impl ResolutionStats {
    pub fn total_resolved(&self) -> usize {
        self.imports_resolved
            + self.calls_resolved_phase_b + self.calls_resolved_phase_c
            + self.refs_resolved_phase_b + self.refs_resolved_phase_c
            + self.types_resolved_phase_b + self.types_resolved_phase_c
    }

    pub fn total_unresolved(&self) -> usize {
        self.unresolved_ambiguous + self.unresolved_no_candidate
    }
}
```

- [ ] **Step 4: Uncomment the `pub use` line in src/resolution/mod.rs**

```rust
pub use stats::ResolutionStats;
```

- [ ] **Step 5: Run tests**

```bash
cargo test --test resolution_unit 2>&1 | tail -10
```

Expected: 2 tests pass.

- [ ] **Step 6: cargo fmt + clippy + commit**

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
git add src/resolution tests/resolution_unit.rs
git commit -m "feat(resolution): add ResolutionStats type"
```

---

### Task 3: `db::queries` read helpers

**Files:**
- Modify: `src/db/queries.rs`

This task adds read-side query helpers used by all three phases. No tests in this task — they exercise live Neo4j and are verified through the integration test in Task 10. Code is small and reviewable by inspection.

- [ ] **Step 1: Implement the helpers**

Overwrite `src/db/queries.rs`:

```rust
//! Read-side Cypher queries used by the resolution pipeline.
//!
//! These all take a `Connection` and a small set of parameters and return
//! plain Rust types (no Cypher leaks beyond this module).

use anyhow::{Context, Result};
use neo4rs::query as cypher;

use crate::db::Connection;

/// One match: a candidate Symbol's qid.
#[derive(Debug, Clone)]
pub struct CandidateQid {
    pub qid: String,
}

/// List all `(:Import)` nodes in the database with their containing File and use-path name.
pub async fn list_imports(conn: &Connection) -> Result<Vec<(String, String, String)>> {
    let mut stream = conn.graph.execute(cypher(
        "MATCH (f:File)-[:CONTAINS]->(i:Import) \
         RETURN i.qid AS ref_qid, f.qid AS file_qid, i.name AS name",
    )).await.context("list_imports")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.context("list_imports row stream")? {
        let ref_qid: String = row.get("ref_qid").context("ref_qid")?;
        let file_qid: String = row.get("file_qid").context("file_qid")?;
        let name: String = row.get("name").context("name")?;
        out.push((ref_qid, file_qid, name));
    }
    Ok(out)
}

/// Look up a `(:Module)` node by exact `qualified_name`. Returns the qid if a
/// single match exists, None if zero, Err if more than one (a workspace
/// invariant violation worth surfacing).
pub async fn find_module_by_qualified_name(
    conn: &Connection,
    qualified_name: &str,
) -> Result<Option<String>> {
    let mut stream = conn.graph
        .execute(
            cypher("MATCH (m:Module {qualified_name: $qn}) RETURN m.qid AS qid LIMIT 2")
                .param("qn", qualified_name.to_string()),
        )
        .await
        .context("find_module_by_qualified_name")?;
    let mut qids = Vec::new();
    while let Some(row) = stream.next().await.context("row stream")? {
        qids.push(row.get::<String>("qid").context("qid")?);
    }
    match qids.len() {
        0 => Ok(None),
        1 => Ok(Some(qids.into_iter().next().unwrap())),
        _ => anyhow::bail!("multiple Module nodes with qualified_name `{qualified_name}`"),
    }
}

/// List all `(:UnresolvedRef)` nodes with their source_qid and kind.
/// Returned tuples: (ref_qid, source_qid, kind, unresolved_name).
pub async fn list_unresolved_refs(conn: &Connection) -> Result<Vec<UnresolvedRefRow>> {
    let mut stream = conn.graph.execute(cypher(
        "MATCH (r:UnresolvedRef) \
         RETURN r.qid AS qid, r.source_qid AS source_qid, r.kind AS kind, r.unresolved_name AS name",
    )).await.context("list_unresolved_refs")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.context("row stream")? {
        out.push(UnresolvedRefRow {
            ref_qid: row.get("qid").context("qid")?,
            source_qid: row.get("source_qid").context("source_qid")?,
            kind: row.get("kind").context("kind")?,
            name: row.get("name").context("name")?,
        });
    }
    Ok(out)
}

#[derive(Debug, Clone)]
pub struct UnresolvedRefRow {
    pub ref_qid: String,
    pub source_qid: String,
    pub kind: String,
    pub name: String,
}

/// Given a source File qid and a leaf name, find candidates reachable via that
/// file's IMPORTS edges: targets of `(:Module)-[:CONTAINS|EXPORTS]->(:Symbol {name: $name})`.
/// Returns matching candidate qids.
pub async fn import_driven_candidates(
    conn: &Connection,
    file_qid: &str,
    leaf_name: &str,
) -> Result<Vec<CandidateQid>> {
    let mut stream = conn.graph
        .execute(
            cypher(
                "MATCH (f:File {qid: $file_qid})-[:IMPORTS]->(m:Module) \
                 MATCH (m)-[:CONTAINS|EXPORTS]->(c:Symbol {name: $name}) \
                 RETURN DISTINCT c.qid AS qid LIMIT 50",
            )
            .param("file_qid", file_qid.to_string())
            .param("name", leaf_name.to_string()),
        )
        .await
        .context("import_driven_candidates")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.context("row stream")? {
        out.push(CandidateQid { qid: row.get("qid").context("qid")? });
    }
    Ok(out)
}

/// Workspace-wide name-match candidates: Symbols whose `qualified_name` ends
/// in `::<name>` or equals `<name>` exactly. Used by Phase C.
pub async fn name_match_candidates(
    conn: &Connection,
    leaf_name: &str,
) -> Result<Vec<CandidateQid>> {
    // Match either exact qualified_name or one ending in `::<name>`.
    let mut stream = conn.graph
        .execute(
            cypher(
                "MATCH (s:Symbol) \
                 WHERE s.name = $name \
                 RETURN DISTINCT s.qid AS qid LIMIT 50",
            )
            .param("name", leaf_name.to_string()),
        )
        .await
        .context("name_match_candidates")?;
    let mut out = Vec::new();
    while let Some(row) = stream.next().await.context("row stream")? {
        out.push(CandidateQid { qid: row.get("qid").context("qid")? });
    }
    Ok(out)
}

/// Source file qid for an UnresolvedRef (walks UnresolvedRef.source_qid → Symbol → CONTAINS-parent File).
pub async fn file_qid_for_source(conn: &Connection, source_qid: &str) -> Result<Option<String>> {
    let mut stream = conn.graph
        .execute(
            cypher(
                "MATCH (s:Symbol {qid: $sq}) \
                 OPTIONAL MATCH (f:File)-[:CONTAINS*1..]->(s) \
                 RETURN f.qid AS file_qid LIMIT 1",
            )
            .param("sq", source_qid.to_string()),
        )
        .await
        .context("file_qid_for_source")?;
    match stream.next().await.context("row stream")? {
        Some(row) => Ok(row.get::<Option<String>>("file_qid").context("file_qid")?),
        None => Ok(None),
    }
}
```

> **Note on `LIMIT 50`:** the candidate-set queries cap their result count. In practice a leaf name resolving to 50+ candidates means we've blown past "ambiguous" — but the cap protects against pathological workspaces.

> **Note on `name_match_candidates`:** matches by `s.name` (not `s.qualified_name`). `name` is just the leaf identifier (e.g., `"helper"`). For slice 3 v1 this is sufficient; if collisions become a problem, future slices can use qualified-name suffix matching instead.

- [ ] **Step 2: Build and verify**

```bash
cargo build 2>&1 | tail -3
```

Expected: clean.

- [ ] **Step 3: cargo fmt + clippy**

Both clean.

- [ ] **Step 4: Commit**

```bash
git add src/db/queries.rs
git commit -m "feat(db): add read-side query helpers for resolution"
```

---

### Task 4: Drop existing confidence-bearing edges

**Files:**
- Modify: `src/resolution/drop.rs`

The idempotent-rebuild model requires dropping all existing resolution-derived edges before each pass. Spec §7 specifies a type-restricted Cypher.

- [ ] **Step 1: Implement**

Overwrite `src/resolution/drop.rs`:

```rust
//! Drop existing resolution-derived edges before each resolve() call.
//!
//! Restricted to edge types known to carry the `confidence` property:
//! `CALLS`, `IMPORTS`, `REFERENCES`, `TYPE_OF`. The type filter lets Neo4j
//! use its relationship-type index instead of a full scan.

use anyhow::{Context, Result};
use neo4rs::query as cypher;

use crate::db::Connection;

pub async fn drop_resolution_edges(conn: &Connection) -> Result<()> {
    conn.graph
        .execute(cypher(
            "MATCH ()-[r:CALLS|IMPORTS|REFERENCES|TYPE_OF]->() \
             WHERE r.confidence IS NOT NULL \
             DELETE r",
        ))
        .await
        .context("drop_resolution_edges")?
        .next()
        .await
        .ok();
    Ok(())
}

/// Reset UnresolvedRef state — clears `candidate_qids` and `unresolved_reason`
/// so a fresh resolution pass starts from a clean slate.
pub async fn reset_unresolved_state(conn: &Connection) -> Result<()> {
    conn.graph
        .execute(cypher(
            "MATCH (r:UnresolvedRef) \
             SET r.candidate_qids = NULL, r.unresolved_reason = NULL",
        ))
        .await
        .context("reset_unresolved_state")?
        .next()
        .await
        .ok();
    Ok(())
}
```

- [ ] **Step 2: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/resolution/drop.rs
git commit -m "feat(resolution): add drop_resolution_edges + reset_unresolved_state"
```

---

### Task 5: Phase A — resolve imports

**Files:**
- Modify: `src/resolution/phase_a_imports.rs`

- [ ] **Step 1: Implement**

Overwrite `src/resolution/phase_a_imports.rs`:

```rust
//! Phase A: resolve `(:Import)` Symbol nodes into `(:File)-[:IMPORTS]->(:Module)` edges.
//!
//! For each Import, look up a Module whose `qualified_name` equals the
//! Import's `name`. If exactly one Module exists in the workspace, emit the
//! IMPORTS edge with `confidence: "import_resolved"`. Otherwise leave it
//! unresolved (most imports point at external crates we don't have nodes for).

use anyhow::{Context, Result};
use neo4rs::query as cypher;

use crate::db::queries::{find_module_by_qualified_name, list_imports};
use crate::db::Connection;

/// Returns the count of IMPORTS edges created.
pub async fn run(conn: &Connection) -> Result<usize> {
    let imports = list_imports(conn).await.context("listing imports")?;
    let mut created = 0;

    for (_ref_qid, file_qid, name) in imports {
        // The Import node's `name` carries the full use path text. Try to find
        // a Module with that exact qualified_name.
        if let Some(module_qid) = find_module_by_qualified_name(conn, &name)
            .await
            .with_context(|| format!("looking up Module `{name}`"))?
        {
            // Create the IMPORTS edge with confidence.
            conn.graph
                .execute(
                    cypher(
                        "MATCH (f:File {qid: $file_qid}), (m:Module {qid: $module_qid}) \
                         MERGE (f)-[r:IMPORTS]->(m) \
                         SET r.confidence = 'import_resolved'",
                    )
                    .param("file_qid", file_qid.clone())
                    .param("module_qid", module_qid),
                )
                .await
                .with_context(|| format!("creating IMPORTS edge from `{file_qid}` to module `{name}`"))?
                .next()
                .await
                .ok();
            created += 1;
        }
        // If no Module found, leave the import unresolved (external crate).
    }

    Ok(created)
}
```

- [ ] **Step 2: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/resolution/phase_a_imports.rs
git commit -m "feat(resolution): Phase A — resolve Import nodes into IMPORTS edges"
```

---

### Task 6: Phase B — import-driven resolution for non-imports

**Files:**
- Modify: `src/resolution/phase_b_import_driven.rs`

- [ ] **Step 1: Implement**

Overwrite `src/resolution/phase_b_import_driven.rs`:

```rust
//! Phase B: import-driven candidate lookup for `UnresolvedRef`s of
//! `kind="call" | "type_ref" | "reference"`.
//!
//! For each unresolved ref, find the source File via the source Symbol's
//! `CONTAINS*` ancestor, then look up candidates reachable via the file's
//! IMPORTS edges. If exactly one candidate matches by name, create the
//! appropriate edge with `confidence: "import_resolved"`.

use anyhow::{Context, Result};
use neo4rs::query as cypher;

use crate::db::queries::{file_qid_for_source, import_driven_candidates, UnresolvedRefRow};
use crate::db::Connection;

/// Returns (calls, refs, types) — number of edges created per kind.
pub async fn run(conn: &Connection, refs: &[UnresolvedRefRow]) -> Result<(usize, usize, usize)> {
    let mut calls = 0;
    let mut refs_count = 0;
    let mut types = 0;

    for r in refs {
        if r.kind == "import" {
            continue; // Phase A handled imports.
        }

        // Locate the source file for this ref.
        let file_qid = match file_qid_for_source(conn, &r.source_qid).await
            .with_context(|| format!("file_qid_for_source({})", r.source_qid))?
        {
            Some(f) => f,
            None => continue, // orphaned source; skip
        };

        // Import-driven candidates by leaf name.
        let candidates = import_driven_candidates(conn, &file_qid, &r.name).await
            .with_context(|| format!("import_driven_candidates(file={file_qid}, name={})", r.name))?;

        if candidates.len() != 1 {
            continue; // 0 or many; let Phase C handle it
        }

        let target_qid = &candidates[0].qid;
        let rel = relationship_for_kind(&r.kind);

        // Create the edge with confidence.
        conn.graph
            .execute(
                cypher(&format!(
                    "MATCH (a {{qid: $a}}), (b {{qid: $b}}) \
                     MERGE (a)-[e:{rel}]->(b) \
                     SET e.confidence = 'import_resolved'"
                ))
                .param("a", r.source_qid.clone())
                .param("b", target_qid.clone()),
            )
            .await
            .with_context(|| format!("creating {rel} edge from {} to {}", r.source_qid, target_qid))?
            .next()
            .await
            .ok();

        // Mark the ref resolved by clearing unresolved_reason explicitly.
        conn.graph
            .execute(
                cypher("MATCH (r:UnresolvedRef {qid: $q}) SET r.unresolved_reason = NULL, r.candidate_qids = NULL")
                    .param("q", r.ref_qid.clone()),
            )
            .await
            .with_context(|| format!("clearing ref state for {}", r.ref_qid))?
            .next()
            .await
            .ok();

        match r.kind.as_str() {
            "call" => calls += 1,
            "type_ref" => types += 1,
            "reference" => refs_count += 1,
            _ => {}
        }
    }

    Ok((calls, refs_count, types))
}

fn relationship_for_kind(kind: &str) -> &'static str {
    match kind {
        "call" => "CALLS",
        "type_ref" => "TYPE_OF",
        "reference" => "REFERENCES",
        _ => "REFERENCES", // shouldn't happen; pessimistic default
    }
}
```

- [ ] **Step 2: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/resolution/phase_b_import_driven.rs
git commit -m "feat(resolution): Phase B — import-driven resolution for calls/refs/types"
```

---

### Task 7: Phase C — name-match fallback

**Files:**
- Modify: `src/resolution/phase_c_name_match.rs`

- [ ] **Step 1: Implement**

Overwrite `src/resolution/phase_c_name_match.rs`:

```rust
//! Phase C: workspace-wide name-match for refs Phase B couldn't resolve.
//!
//! Single candidate → resolution edge with `confidence: "name_match"`.
//! Multiple → record `candidate_qids` + `unresolved_reason = "ambiguous"`.
//! Zero → `unresolved_reason = "no_candidate"`.

use anyhow::{Context, Result};
use neo4rs::query as cypher;
use serde_json::Value;

use crate::db::queries::{name_match_candidates, UnresolvedRefRow};
use crate::db::Connection;

pub struct PhaseCResult {
    pub calls: usize,
    pub refs: usize,
    pub types: usize,
    pub ambiguous: usize,
    pub no_candidate: usize,
}

/// Run Phase C against all UnresolvedRefs whose `unresolved_reason` is NULL
/// (i.e. either fresh or cleared by Phase B's successful resolution).
pub async fn run(conn: &Connection, refs: &[UnresolvedRefRow]) -> Result<PhaseCResult> {
    let mut result = PhaseCResult {
        calls: 0, refs: 0, types: 0, ambiguous: 0, no_candidate: 0,
    };

    for r in refs {
        if r.kind == "import" {
            continue;
        }

        // Skip if already resolved by Phase B (we detect via a probe: does the
        // expected outgoing edge already exist?). Cheaper alternative: pass a
        // HashSet of already-resolved ref_qids from Phase B. For slice 3 we
        // accept the extra round-trip in favor of simplicity.
        if is_already_resolved(conn, r).await.unwrap_or(false) {
            continue;
        }

        let candidates = name_match_candidates(conn, &r.name).await
            .with_context(|| format!("name_match_candidates({})", r.name))?;

        match candidates.len() {
            0 => {
                mark_unresolved(conn, &r.ref_qid, "no_candidate", &[]).await?;
                result.no_candidate += 1;
            }
            1 => {
                let target = &candidates[0].qid;
                let rel = relationship_for_kind(&r.kind);
                create_edge_with_confidence(conn, &r.source_qid, target, rel, "name_match").await?;
                conn.graph.execute(
                    cypher("MATCH (r:UnresolvedRef {qid: $q}) \
                            SET r.unresolved_reason = NULL, r.candidate_qids = NULL")
                        .param("q", r.ref_qid.clone())
                ).await?.next().await.ok();
                match r.kind.as_str() {
                    "call" => result.calls += 1,
                    "type_ref" => result.types += 1,
                    "reference" => result.refs += 1,
                    _ => {}
                }
            }
            _ => {
                let qids: Vec<String> = candidates.iter().map(|c| c.qid.clone()).collect();
                mark_unresolved(conn, &r.ref_qid, "ambiguous", &qids).await?;
                result.ambiguous += 1;
            }
        }
    }

    Ok(result)
}

async fn is_already_resolved(conn: &Connection, r: &UnresolvedRefRow) -> Result<bool> {
    let rel = relationship_for_kind(&r.kind);
    let cypher_text = format!(
        "MATCH (a {{qid: $sq}})-[e:{rel}]->() WHERE e.confidence IS NOT NULL \
         RETURN count(e) AS c LIMIT 1"
    );
    let mut stream = conn.graph
        .execute(cypher(&cypher_text).param("sq", r.source_qid.clone()))
        .await?;
    let row = stream.next().await?.context("no row")?;
    let c: i64 = row.get("c")?;
    Ok(c > 0)
}

async fn create_edge_with_confidence(
    conn: &Connection,
    from: &str,
    to: &str,
    rel: &str,
    confidence: &str,
) -> Result<()> {
    let cypher_text = format!(
        "MATCH (a {{qid: $a}}), (b {{qid: $b}}) \
         MERGE (a)-[e:{rel}]->(b) SET e.confidence = $c"
    );
    conn.graph
        .execute(
            cypher(&cypher_text)
                .param("a", from.to_string())
                .param("b", to.to_string())
                .param("c", confidence.to_string()),
        )
        .await?
        .next()
        .await
        .ok();
    Ok(())
}

async fn mark_unresolved(conn: &Connection, ref_qid: &str, reason: &str, candidate_qids: &[String]) -> Result<()> {
    let candidates_json = if candidate_qids.is_empty() {
        Value::Null
    } else {
        Value::Array(candidate_qids.iter().map(|s| Value::String(s.clone())).collect())
    };
    let candidates_param = crate::db::write::to_bolt(candidates_json)?;
    conn.graph
        .execute(
            cypher(
                "MATCH (r:UnresolvedRef {qid: $q}) \
                 SET r.unresolved_reason = $reason, r.candidate_qids = $cands",
            )
            .param("q", ref_qid.to_string())
            .param("reason", reason.to_string())
            .param("cands", candidates_param),
        )
        .await?
        .next()
        .await
        .ok();
    Ok(())
}

fn relationship_for_kind(kind: &str) -> &'static str {
    match kind {
        "call" => "CALLS",
        "type_ref" => "TYPE_OF",
        "reference" => "REFERENCES",
        _ => "REFERENCES",
    }
}
```

> **Note: `crate::db::write::to_bolt`** — this is the BoltType adapter added in Slice 2 Task 10. It's currently a private function in `src/db/write.rs`. Either:
> (a) Make it `pub(crate)` in `src/db/write.rs` so Phase C can use it, OR
> (b) Duplicate the small adapter here (it's 3 lines).
> Prefer (a) — promote the existing helper to `pub(crate)` to avoid duplication.

> **Note on `is_already_resolved`:** the probe assumes Phase B's resolution edges have `confidence IS NOT NULL`, which they do (set to `'import_resolved'`). Resolution always sets confidence, so any confidence-bearing outgoing edge of the right type from `source_qid` indicates Phase B already handled this ref. The probe is per-ref so it's a Cypher round-trip — fine for slice 3, room for optimization later.

- [ ] **Step 2: Promote `to_bolt` to `pub(crate)` in `src/db/write.rs`**

Change the signature in `src/db/write.rs`:

```rust
pub(crate) fn to_bolt(v: Value) -> Result<neo4rs::BoltType> {
    // ... existing body unchanged ...
}
```

- [ ] **Step 3: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/resolution/phase_c_name_match.rs src/db/write.rs
git commit -m "feat(resolution): Phase C — workspace name-match fallback with ambiguity handling"
```

---

### Task 8: `resolve()` orchestrator

**Files:**
- Modify: `src/resolution/mod.rs`

- [ ] **Step 1: Implement the orchestrator**

Replace the stub `resolve()` in `src/resolution/mod.rs`:

```rust
//! Resolution pipeline (spec §7).

pub mod drop;
pub mod phase_a_imports;
pub mod phase_b_import_driven;
pub mod phase_c_name_match;
pub mod stats;

pub use stats::ResolutionStats;

use anyhow::{Context, Result};
use tracing::info;

use crate::db::queries::list_unresolved_refs;
use crate::db::Connection;

/// Run the full resolution pipeline against the database referenced by `conn`.
///
/// Idempotent: each call drops all `confidence`-bearing edges and resets the
/// state of every `UnresolvedRef`, then runs Phases A → B → C.
pub async fn resolve(conn: &Connection) -> Result<ResolutionStats> {
    info!("dropping existing resolution edges");
    drop::drop_resolution_edges(conn).await.context("drop_resolution_edges")?;
    drop::reset_unresolved_state(conn).await.context("reset_unresolved_state")?;

    info!("Phase A: resolving imports");
    let imports_resolved = phase_a_imports::run(conn).await.context("Phase A")?;

    info!("listing UnresolvedRefs for Phases B+C");
    let refs = list_unresolved_refs(conn).await.context("list_unresolved_refs")?;

    info!("Phase B: import-driven candidates for {} unresolved refs", refs.len());
    let (b_calls, b_refs, b_types) = phase_b_import_driven::run(conn, &refs).await.context("Phase B")?;

    info!("Phase C: name-match fallback");
    let c = phase_c_name_match::run(conn, &refs).await.context("Phase C")?;

    let stats = ResolutionStats {
        imports_resolved,
        calls_resolved_phase_b: b_calls,
        calls_resolved_phase_c: c.calls,
        refs_resolved_phase_b: b_refs,
        refs_resolved_phase_c: c.refs,
        types_resolved_phase_b: b_types,
        types_resolved_phase_c: c.types,
        unresolved_ambiguous: c.ambiguous,
        unresolved_no_candidate: c.no_candidate,
    };
    info!(
        "resolution complete: {} resolved, {} unresolved",
        stats.total_resolved(),
        stats.total_unresolved()
    );
    Ok(stats)
}
```

- [ ] **Step 2: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/resolution/mod.rs
git commit -m "feat(resolution): resolve() orchestrator — Phases A → B → C"
```

---

### Task 9: Wire resolution into `codegraph-rs index`

**Files:**
- Modify: `src/cli/index.rs`

- [ ] **Step 1: Call `resolve()` after `write_extraction`**

Add the import at the top of `src/cli/index.rs` (alongside the existing `use crate::...` lines):

```rust
use crate::resolution;
```

Inside the `rt.block_on` block, after `write_extraction(&conn, results).await?;` call `resolution::resolve(&conn).await?` and log the stats. Concretely the async block becomes:

```rust
    rt.block_on(async {
        let conn = Connection::open(
            &cfg.workspace.neo4j.uri,
            &cfg.workspace.neo4j.username,
            &pw.value,
            &cfg.workspace.neo4j.database,
        ).await.context("connecting to workspace database")?;

        apply_schema(&conn).await.context("applying schema")?;
        write_extraction(&conn, results).await.context("writing extraction to Neo4j")?;

        let stats = resolution::resolve(&conn).await.context("resolving references")?;
        info!(
            "resolution stats: imports={}, calls(B/C)={}/{}, refs(B/C)={}/{}, types(B/C)={}/{}, ambiguous={}, no_candidate={}",
            stats.imports_resolved,
            stats.calls_resolved_phase_b, stats.calls_resolved_phase_c,
            stats.refs_resolved_phase_b, stats.refs_resolved_phase_c,
            stats.types_resolved_phase_b, stats.types_resolved_phase_c,
            stats.unresolved_ambiguous, stats.unresolved_no_candidate,
        );

        Ok::<(), anyhow::Error>(())
    })?;
```

> The path is `crate::resolution::resolve` (via the `use crate::resolution;` import) — `src/cli/index.rs` lives inside the library crate, not in `src/main.rs`. Don't write `codegraph_rs::resolution`.

- [ ] **Step 2: Build, clippy, fmt, commit**

```bash
cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add src/cli/index.rs
git commit -m "feat(cli): run resolution after extraction in index command"
```

---

### Task 10: Integration test against tiny-rust

**Files:**
- Create: `tests/integration_resolve.rs` (gated)
- Optional: extend `tests/integration_index.rs` to also assert on resolution outcomes — but a separate file is cleaner.

- [ ] **Step 1: Create the integration test**

Create `tests/integration_resolve.rs`:

```rust
//! Integration test for resolution. Runs `codegraph-rs index` (which now
//! includes resolution) on the tiny-rust fixture and verifies the expected
//! `CALLS` edges exist.
//!
//! Gated on CODEGRAPH_RS_TEST_NEO4J=1.

use codegraph_rs::cli::{index, init::{InitArgs, self}};
use neo4rs::query;
use std::fs;
use tempfile::tempdir;

fn enabled() -> bool {
    std::env::var("CODEGRAPH_RS_TEST_NEO4J").as_deref() == Ok("1")
}

#[test]
fn resolves_tiny_rust_calls() {
    if !enabled() {
        eprintln!("SKIP: set CODEGRAPH_RS_TEST_NEO4J=1 to enable");
        return;
    }
    let pw = std::env::var("CODEGRAPH_RS_TEST_NEO4J_PASSWORD")
        .expect("CODEGRAPH_RS_TEST_NEO4J_PASSWORD required");

    let dir = tempdir().unwrap();
    let workspace_name = format!("resolve-test-{}", std::process::id());

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

    // Index (now runs resolution as well).
    std::env::set_var("CODEGRAPH_RS_NEO4J_PASSWORD", &pw);
    index::run(Some(dir.path())).expect("index+resolve must succeed");
    std::env::remove_var("CODEGRAPH_RS_NEO4J_PASSWORD");

    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let db_name = format!("codegraph-resolve-test-{}", std::process::id());
        let conn = codegraph_rs::db::Connection::open(
            "bolt://localhost:7687", "neo4j", &pw, &db_name,
        ).await.unwrap();

        // Expect: at least 3 CALLS edges (main → library_entry, library_entry → helper, helper → constant).
        // Some may be name_match (Phase C); some may be import_resolved (Phase B). Either is fine.
        let calls_count: i64 = scalar(&conn, "MATCH ()-[r:CALLS]->() RETURN count(r) AS c").await;
        assert!(calls_count >= 3, "expected at least 3 CALLS edges, got {calls_count}");

        // Specifically verify helper → constant.
        let helper_to_constant: i64 = scalar(
            &conn,
            "MATCH (a:Symbol {name: 'helper'})-[:CALLS]->(b:Symbol {name: 'constant'}) \
             RETURN count(*) AS c",
        ).await;
        assert_eq!(helper_to_constant, 1, "expected exactly one helper→constant CALLS edge");

        // All CALLS edges must carry a confidence property.
        let unconfidence: i64 = scalar(
            &conn,
            "MATCH ()-[r:CALLS]->() WHERE r.confidence IS NULL RETURN count(r) AS c",
        ).await;
        assert_eq!(unconfidence, 0, "every CALLS edge must have a confidence property");

        // No UnresolvedRef should still claim `unresolved_reason IS NULL` AND lack an
        // outgoing resolution edge. (Either resolved with edge OR has a reason.)
        // For tiny-rust this is fine because every call resolves via name_match.

        // Cleanup.
        let sys = codegraph_rs::db::Connection::system(
            "bolt://localhost:7687", "neo4j", &pw,
        ).await.unwrap();
        sys.graph.execute(query(&format!("DROP DATABASE `{db_name}` IF EXISTS WAIT")))
            .await.unwrap().next().await.ok();
    });
}

async fn scalar(conn: &codegraph_rs::db::Connection, cypher: &str) -> i64 {
    let mut stream = conn.graph.execute(query(cypher)).await.unwrap();
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

- [ ] **Step 2: Verify SKIP path**

```bash
cargo test --test integration_resolve -- --nocapture 2>&1 | tail -10
```

Expected: 1 test, prints SKIP, passes.

- [ ] **Step 3: cargo fmt + clippy**

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

- [ ] **Step 4: Commit**

```bash
git add tests/integration_resolve.rs
git commit -m "test(resolution): integration test verifying CALLS edges on tiny-rust"
```

---

### Task 11: README + slice-3 tag

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update README**

Update `/Users/gyorgybalazsi/codegraph-rs/README.md`'s "Status" section to mention slice 3:

```markdown
## Status

In development. Currently implements **Slice 3: Resolution** —
workspace initialization, indexing (Rust files, structural extraction), and
resolution (CALLS / IMPORTS / REFERENCES / TYPE_OF edges with confidence).
Vector embeddings, sync, MCP server, and Daml support land in subsequent slices.
```

- [ ] **Step 2: Final test suite + tag**

```bash
cargo test 2>&1 | tail -15
cargo clippy --all-targets -- -D warnings
cargo fmt --check
git add README.md
git commit -m "docs: update README for slice 3"
git tag slice-3-resolution
```

Expected test count breakdown:

| Test binary | Count |
|---|---|
| `extraction_unit` | 31 |
| `resolution_unit` (new) | 2 |
| `config_parsing` | 13 |
| `cli_smoke` | 2 |
| `integration_db` (SKIP) | 2 |
| `integration_init` (SKIP) | 1 |
| `integration_status` (SKIP) | 1 |
| `integration_index` (SKIP) | 1 |
| `integration_resolve` (new, SKIP) | 1 |
| **Total** | **54** |

---

## Slice 3 Done

Verify the artifact:

- [ ] `cargo build` clean
- [ ] `cargo test` (no env vars) — all tests pass; integration tests SKIP cleanly
- [ ] `CODEGRAPH_RS_TEST_NEO4J=1 cargo test -- --test-threads=1` — all tests pass
- [ ] `codegraph-rs index` runs both extraction and resolution; status reports the same counts as before but the graph now has `CALLS` edges with `confidence` property
- [ ] Manual smoke: index a small Rust workspace, query Neo4j Browser for `MATCH ()-[r:CALLS]->() RETURN r LIMIT 25` — see real edges
- [ ] No embeddings yet (Slice 4)
- [ ] No sync yet (Slice 5) — running `index` re-runs the full extraction + resolution

When all check, slice 3 is complete and we can write slice 4 (vector embeddings).
