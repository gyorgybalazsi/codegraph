# codegraph-rs — Design Spec

**Status:** Draft (pre-implementation)
**Date:** 2026-04-22
**Target repository:** `../codegraph-rs` (sibling of the current `codegraph` repo)

## 1. Goal and motivation

Reimplement the existing TypeScript CodeGraph project in Rust, backed by Neo4j as the graph database. The new tool fully replaces the TypeScript version; both will run side-by-side during transition.

Drivers:

- **Author preference for Rust** over TypeScript for this kind of work (AST walking, graph traversal, parallelism).
- **Neo4j visualization** (Neo4j Browser/Bloom) for understanding what the graph contains, which the SQLite-backed TS version cannot offer.
- **Daml language support** as a new feature (Daml is the BitSafe team's primary language). Daml support could be added to the TS version, but is bundled into the rewrite because the rewrite is happening anyway.

Non-goals:

- Maintaining TypeScript compatibility or interop.
- Keeping the SQLite storage path.
- Day-one parity with every TS-version feature.

## 2. Scope

### In scope for v1

| Module | Status |
|---|---|
| Tree-sitter extraction | Yes |
| Neo4j-backed storage | Yes |
| Reference resolution (imports + name match) | Yes |
| Graph queries (callers, callees, impact, traversal) | Yes |
| Vector embeddings via `fastembed-rs` + Neo4j vector index | Yes |
| Context builder | Yes |
| Incremental sync | Yes |
| MCP server (stdio) | Yes |
| Workspace config | Yes |

### Out of scope for v1

- Visualizer web UI (Neo4j Browser replaces it).
- Framework-specific resolvers (React, Express, Laravel, etc.) — the resolver pipeline is generic; framework knowledge is added later as needed.
- Auto-installed git hooks for sync — sync is user-invoked.
- Interactive installer wizard — `init` accepts CLI flags.
- Languages other than Rust and Daml (Python, Go, Java, etc. arrive in later versions).

### Languages at v1 launch

- **Rust** — via `tree-sitter-rust` crate.
- **Daml** — via `tree-sitter-haskell` crate (Daml syntax is close to Haskell). This produces correct nodes for modules, types, functions, imports, but cannot see template- or choice-specific structure. Bringing up a real `tree-sitter-daml` grammar is a separate project tracked outside this spec.

## 3. High-level architecture

### Crate layout

A single crate (`codegraph-rs`) with internal modules. No premature workspace splitting. The compile-time and API-design overhead of multiple crates is not justified for v1; can be extracted later when a real consumer appears.

```
codegraph-rs/
  Cargo.toml
  src/
    main.rs             # CLI entry (clap)
    lib.rs              # public re-exports

    config/             # workspace config types and loading
      mod.rs
      workspace.rs

    db/                 # Neo4j Bolt client wrapper
      mod.rs
      schema.rs         # constraints, indexes, vector index setup
      queries.rs        # parameterized Cypher queries

    extraction/         # tree-sitter -> raw nodes/edges
      mod.rs
      walker.rs
      parser.rs
      languages/
        rust.rs
        rust.scm        # tree-sitter capture queries
        daml.rs
        daml.scm

    resolution/         # unresolved refs -> resolved edges
      mod.rs
      imports.rs
      calls.rs

    vectors/            # fastembed-rs orchestration
      mod.rs
      embedder.rs

    graph/              # read-side: traversals, callers/callees, impact
      mod.rs
      traversal.rs

    context/            # context bundles for MCP tools
      mod.rs

    sync/               # incremental updates
      mod.rs

    mcp/                # MCP server
      mod.rs
      tools.rs
      transport.rs

  tests/                # integration tests against a real Neo4j
  benches/              # indexing throughput benchmarks
```

### Runtime

- **Async runtime:** `tokio`. Required by `neo4rs` (Neo4j Bolt client) and `rmcp` (MCP SDK).
- **CPU-bound work:** `rayon`. Parallel parsing across files, with results flowing through a `tokio::mpsc` channel into the async DB writer.
- **Single process.** No sidecars, no embedded JVM. The tool is one binary that talks to an external Neo4j over Bolt.

### Indexing dataflow

```
CLI -> config::load
    -> extraction::walker      (iterate workspace roots, find source files)
    -> extraction::parser      (rayon: parse each file with tree-sitter)
    -> raw {nodes, structural_edges, unresolved_refs}
    -> db::write_extraction    (Cypher UNWIND/MERGE for nodes + UnresolvedRef
                                + structural edges: CONTAINS, EXTENDS,
                                IMPLEMENTS produced directly from AST)
    -> resolution              (UnresolvedRefs -> CALLS, IMPORTS-targets,
                                REFERENCES, TYPE_OF — derived from
                                UnresolvedRef records, not extraction)
    -> db::write_resolution    (resolution-produced edges in batches)
    -> vectors::embed          (fastembed-rs batches; SET embedding via Cypher)
```

Edges fall into two cleanly separated buckets:
- **Structural edges** (produced by extraction, derivable from a single file's AST): `CONTAINS`, `EXTENDS`, `IMPLEMENTS`, `EXPORTS`, `INSTANTIATES`, `DECORATES`, `OVERRIDES`, `RETURNS`, `TYPE_OF` (when the type is in the same file).
- **Resolution-derived edges** (produced by resolution from `UnresolvedRef` records, may cross files): `CALLS`, `IMPORTS` (target side), `REFERENCES`, cross-file `TYPE_OF`.

### Query dataflow

```
Claude Code stdin -> mcp::transport
                  -> mcp::tools::<tool>
                  -> graph::traversal | vectors::search
                  -> db::queries (parameterized Cypher)
                  -> JSON response on stdout
```

## 4. Neo4j schema

### Conventions

- **Labels:** PascalCase (`Function`, `Class`, `File`)
- **Relationship types:** UPPER_SNAKE_CASE (`CALLS`, `IMPORTS`)
- **Properties:** snake_case (`qualified_name`, `start_line`)

### Node labels

Every code node gets two labels: `Symbol` (supertype) plus its specific kind. `(:Symbol:Function)` makes "any symbol" queries trivial while preserving kind-specific filtering. **Extractors must always emit both labels** — single-label nodes are a bug.

The word "module" in this spec means the Neo4j label `Module` (a graph node) unless preceded by a language qualifier like "Rust module" or "Daml module."

| Kind label | Use |
|---|---|
| `File` | A source file |
| `Module`, `Namespace` | Container |
| `Function`, `Method` | Callable |
| `Class`, `Struct`, `Interface`, `Trait`, `Enum`, `TypeAlias` | Types (v1: only `Struct`, `Trait`, `Enum`, `TypeAlias` produced; others reserved for future languages) |
| `Property`, `Field`, `Variable`, `Constant`, `Parameter`, `EnumMember` | Values |
| `Import`, `Export` | Module boundaries |
| `UnresolvedRef` | Permanent record of an unresolved name reference (see §7) |
| `Template`, `Choice` | Daml-specific (reserved; populated only when `tree-sitter-daml` ships) |

**Note on `impl` blocks (Rust) and instance declarations (Daml/Haskell):** v1 does NOT emit a separate `Impl` node. Methods inside `impl Foo { fn bar() }` are emitted as `Method` nodes whose `qualified_name` carries the type prefix (`crate::module::Foo::bar`), and whose `CONTAINS` parent is the source file. The type-to-method relationship is captured implicitly by the `qualified_name` and can be derived by querying for methods whose `qualified_name` starts with `<TypeQualName>::`.

### Relationship types

| Type | Meaning |
|---|---|
| `CONTAINS` | Lexical parent/child (`File`-`Function`, `Module`-`Class`, etc.) |
| `CALLS` | Call from one callable to another |
| `IMPORTS` | File or module imports another module |
| `EXPORTS` | Symbol exported from a module |
| `EXTENDS`, `IMPLEMENTS`, `OVERRIDES` | Inheritance / impl |
| `REFERENCES` | Generic symbol reference |
| `TYPE_OF`, `RETURNS` | Type relationships |
| `INSTANTIATES` | Constructor or factory invocation |
| `DECORATES` | Decorator or attribute |
| `HAS_REF` | From a `Symbol` to its `UnresolvedRef` records (structural; produced during extraction) |
| `HAS_CHOICE`, `SIGNATORY`, `CONTROLLER`, `OBSERVER` | Daml-specific (reserved) |

### Common properties

```
qid                TEXT     UNIQUE     "<root>:<rel_path>:<qualified_name>[#<n>]"
name               TEXT                short identifier
qualified_name     TEXT                fully-qualified identifier
root_name          TEXT                workspace root this symbol belongs to
relative_path      TEXT                path of containing file, relative to root
language           TEXT
start_line         INTEGER
end_line           INTEGER
start_col          INTEGER
end_col            INTEGER
signature          TEXT NULL           callables: param + return type as text
doc                TEXT NULL           leading doc comment if any
embedding          LIST<FLOAT> NULL    384-dim vector (set on embed-eligible nodes)
parse_reliable     BOOLEAN             false on Daml `template`/`choice`/`signatory`
                                       blocks where the Haskell grammar misparses
```

### File-specific properties

```
content_hash       TEXT                BLAKE3 hash of file contents (for sync diffing)
last_indexed_at    INTEGER             unix timestamp
```

### Indexes and constraints

```cypher
CREATE CONSTRAINT symbol_qid_unique
  FOR (n:Symbol) REQUIRE n.qid IS UNIQUE;

CREATE INDEX symbol_name_idx
  FOR (n:Symbol) ON (n.name);

CREATE INDEX symbol_qualified_idx
  FOR (n:Symbol) ON (n.qualified_name);

CREATE FULLTEXT INDEX symbol_fts
  FOR (n:Symbol) ON EACH [n.name, n.qualified_name, n.signature, n.doc];

CREATE VECTOR INDEX symbol_embedding
  FOR (n:Symbol) ON n.embedding
  OPTIONS {
    indexConfig: {
      `vector.dimensions`: 384,
      `vector.similarity_function`: 'cosine'
    }
  };
```

- **384 dims** matches `fastembed-rs`'s default `BAAI/bge-small-en-v1.5` model.
- **Full-text index** replaces the SQLite FTS5 table.
- **Vector index** lives on the same node — embedding is a property, not a separate row.

### Identity scheme

```
qid = "<root_name>:<relative_path>:<qualified_name>[#<ordinal>]"
```

- `qualified_name` is the language-specific fully qualified identifier (e.g., `crate::util::parse_config`, `Splice.Foo.bar`, `Splice.Foo.Bar::baz` for a method on a type).
- The `#<ordinal>` suffix is omitted unless the same `(root, relative_path, qualified_name)` triple has multiple symbols — extremely rare in Rust and Daml since both languages disallow same-name overloads at one scope. When a collision is detected during extraction, the extractor assigns `#1`, `#2`, … by source-order.
- **`start_line` is deliberately NOT in the qid.** Including it would invalidate every downstream symbol's qid whenever a function is added above it in the same file, cascading edge invalidation across the graph. Position lives in the `start_line`/`end_line` properties only.
- On reindex, MERGE by `qid` is naturally idempotent.

### Edge properties

Edges may carry small property sets. v1 uses:

| Edge type | Property | Values | Meaning |
|---|---|---|---|
| `CALLS`, `REFERENCES`, `TYPE_OF` (etc., resolution-produced) | `confidence` | `"import_resolved"` \| `"name_match"` | How the resolver derived this edge (see §7) |

Other edge types carry no properties in v1.

## 5. Workspace model and database isolation

### Workspace = one logical graph

A **workspace** is one logical project that may span multiple source root folders. Examples:

- A multi-root Daml workspace combining `splice/` and `utility/` into one connected graph, so a `Utility.Foo` import inside `splice/` becomes a real `IMPORTS` edge in the graph.
- A single-root Rust workspace (e.g., codegraph-rs dogfooding itself) is the degenerate case.

One workspace maps to one Neo4j database. Database-per-workspace, not database-per-root, gives:

- Hard isolation between workspaces (e.g., the codegraph-rs source graph is fully separate from the daml-stack graph).
- No per-query filtering boilerplate inside Cypher.
- Cheap drop/reindex of one workspace via `DROP DATABASE`.

This relies on Neo4j Enterprise Edition for multi-database support, which is what Neo4j Desktop installs by default — free for local development use.

### Database creation and privileges

`codegraph-rs init` (and `codegraph-rs index` if the database does not yet exist) issues `CREATE DATABASE <name> IF NOT EXISTS` against Neo4j's `system` database. This requires the configured Neo4j user to have the `CREATE DATABASE` privilege. The Neo4j Desktop default `neo4j` admin user has this privilege; users with restricted accounts must either grant the privilege or pre-create the database manually.

### Database naming

Neo4j 5 database names must:
- be 3–63 characters,
- start with an ASCII letter,
- contain only ASCII letters, digits, dots, and dashes (**no underscores**, no other special characters).

Spec convention for generated database names: `codegraph-<workspace-name>` with the workspace name lowercased and any non-conforming characters replaced by `-`.

### Workspace config file

Canonical path: `<workspace_dir>/.codegraph/config.toml`.

CLI discovery: walk upward from cwd until a `.codegraph/config.toml` is found, or use `--workspace <path>` to specify explicitly. Same pattern as `git` finding `.git/`.

Example (multi-root Daml workspace):

```toml
schema_version = 1

[workspace]
name = "daml-stack"

[workspace.neo4j]
uri = "bolt://localhost:7687"
database = "codegraph-daml-stack"   # see Database naming above
username = "neo4j"
# password lives in .codegraph/secrets.toml or env

[[workspace.roots]]
name = "splice"
path = "../splice"               # relative to this config file

[[workspace.roots]]
name = "utility"
path = "../utility"

[indexing]
languages = ["rust", "daml"]     # optional whitelist; default: all detected

[embeddings]
enabled = true
model = "bge-small-en-v1.5"      # only valid value in v1
```

Custom file-pattern excludes go in `<workspace_dir>/.codegraph/.codegraphignore`, which uses gitignore syntax. There is no inline TOML escape hatch — keeping all ignore rules in one place avoids confusion about precedence.

Single-root self-host config is the obvious shrink of the above.

### Secrets

- `.codegraph/secrets.toml` holds `neo4j_password = "..."`. Auto-gitignored by `codegraph-rs init`.
- Environment variable `CODEGRAPH_RS_NEO4J_PASSWORD` overrides the file.
- If neither is present, `codegraph-rs` prompts on stdin in interactive CLI mode. Never prompts in `serve` mode.

### Validation

- Config parsed with `serde` + `toml`. Schema errors produce human-friendly messages pointing at the offending field.
- `schema_version` required; unknown versions fail fast.
- Root paths canonicalized at load time; non-existent roots are a hard error.
- Neo4j connectivity verified before indexing — fail fast if the DBMS isn't running.

## 6. Extraction pipeline

### File discovery

- `ignore` crate (the walker behind ripgrep). Honors `.gitignore`, `.ignore`, and a workspace-level `.codegraphignore`.
- Walks all roots in parallel; each root contributes files with its `root_name` baked in.
- Language detected by extension: `.rs` → Rust, `.daml` → Daml.

### Parsing

- One `tree_sitter::Parser` per worker thread (`thread_local!`) since parsers are not `Sync`.
- Grammars loaded at startup from linked crates: `tree_sitter_rust::LANGUAGE`, `tree_sitter_haskell::LANGUAGE`.
- `.daml` files are mapped to the Haskell language in `languages/daml.rs` — a one-line mapping that flips to `tree_sitter_daml::LANGUAGE` when that crate exists.

### Per-language extractors

A trait abstracts the extraction step:

```rust
pub trait LanguageExtractor: Send + Sync {
    fn extract(
        &self,
        source: &str,
        tree: &tree_sitter::Tree,
        file_id: &str,
        root_name: &str,
    ) -> ExtractionResult;
}

pub struct ExtractionResult {
    pub nodes: Vec<RawNode>,
    pub edges: Vec<RawEdge>,
    pub unresolved_refs: Vec<UnresolvedRef>,
}
```

- `languages/rust.rs` maps Rust AST nodes — `function_item` → `Function`, `struct_item` → `Struct`, `enum_item` → `Enum`, `trait_item` → `Trait`, `type_item` → `TypeAlias`, `mod_item` → `Module`, `use_declaration` → `Import`, `call_expression` → captures an `UnresolvedRef`. `impl_item` is not a node itself; its methods are emitted as `Method` nodes whose `qualified_name` includes the impl'd type (`<crate>::<mod>::<Type>::<method>`) and whose `CONTAINS` parent is the source file (see §4 note on `impl` blocks).
- `languages/daml.rs` maps Haskell-grammar nodes — `function` → `Function`, `decl/type_synonym` → `TypeAlias`, `data_type` → `Struct`, `module` → `Module`, `import` → `Import`. Templates, choices, and signatories are not extracted in v1; the Haskell grammar does not see them.
- Each extractor uses tree-sitter `.scm` query files (capture queries) to locate interesting AST nodes declaratively rather than hand-walking.

### Parallelism and writes

- `rayon` par-iter drives the file list; each file emits one `ExtractionResult`.
- Results stream into a `tokio::mpsc` channel feeding the async DB writer.
- DB writer batches nodes/edges in groups of ~500 and writes via `UNWIND $batch AS row MERGE ...` — one round-trip per batch, not per node.
- File `content_hash` (BLAKE3) and `last_indexed_at` are written when extraction completes for that file.

### UnresolvedRef

Captures "this call/reference needs resolution." Stored as a permanent `(:UnresolvedRef)` node connected to its source by `(:Symbol)-[:HAS_REF]->(:UnresolvedRef)`. The full schema is in §4 / §7 — extraction emits these records and never touches the resolution-derived edges that depend on them.

```rust
pub struct UnresolvedRef {
    pub source_qid: String,         // the caller / referencer
    pub unresolved_name: String,    // "foo", "Module.foo", "crate::util::foo"
    pub kind: UnresolvedKind,       // Call, Import, TypeRef, Reference
    pub line: u32,
    pub col: u32,
}
```

### Error handling

- Parse failures on individual files: log + skip, do not abort the run.
- Tree-sitter `ERROR` nodes inside a file: extraction continues over valid portions; logged at debug level.

### Not in v1

- Framework-specific enrichment (no React, Express, Laravel resolvers).
- Tree-sitter incremental parsing — v1 reparses changed files whole-file.

## 7. Resolution pipeline

### Core idea: `UnresolvedRef` is the durable record of source-code intent

Every cross-symbol reference produced by extraction (a function call, an import, a type reference, etc.) is materialized as a permanent `(:UnresolvedRef)` node. These nodes are **the source of truth for what the source code says**, independent of whether the resolver currently has a target for them.

Resolution-derived edges (`CALLS`, `IMPORTS`-target, `REFERENCES`, cross-file `TYPE_OF`) are **derived/cached** from `UnresolvedRef` records. They are recomputed whenever the source records or the candidate targets change.

This shape makes incremental sync correct: when file B is re-extracted with potentially different `qid`s, B's `UnresolvedRef` records are deleted and re-created, but file A's `UnresolvedRef` records (which referenced symbols *in* B) are untouched. Re-running resolution recreates the cross-file edges against B's new symbols.

### `UnresolvedRef` schema

```
qid                TEXT  UNIQUE  "<source_qid>:ref:<line>:<col>:<ord>"
source_qid         TEXT          the caller / referencer
unresolved_name    TEXT          "foo", "Module.foo", "crate::util::foo"
kind               TEXT          "call" | "import" | "type_ref" | "reference"
line               INTEGER
col                INTEGER
candidate_qids     LIST<TEXT>    set when stage-2 finds >1 candidate (else NULL)
unresolved_reason  TEXT NULL     "no_candidate" | "ambiguous" | NULL when resolved
```

An `UnresolvedRef` is connected to its source by `(source)-[:HAS_REF]->(:UnresolvedRef)`. The `HAS_REF` edges are structural and produced during extraction. They do not require resolution.

### Order

Resolution runs **after all files are extracted/written**, so every candidate target exists. During incremental sync, resolution runs after the changed/added/deleted files have been processed.

### Two-stage resolver

**Stage 1 — Import-driven (high confidence)**

For each `UnresolvedRef`:
1. Look at the source file's `IMPORTS` edges.
2. Walk from each imported module/file to its `EXPORTS` or `CONTAINS` children.
3. Match by name. If exactly one candidate → create a resolution edge of the appropriate type (`CALLS` for `kind="call"`, etc.) from `source_qid` to the candidate, with `confidence: "import_resolved"`.

Rust specifics:
- `use crate::util::foo;` → resolves `foo` calls to `crate::util::foo`.
- `use crate::util;` then `util::foo()` → walks module prefix.
- `mod` declarations build the module tree; path segments traverse it.

Daml specifics (via Haskell grammar):
- `import Utility.Foo` → file imports module.
- Calls resolve through imports in scope.
- Cross-root works naturally: both roots populate the same Neo4j database; module qualified names are globally unique.

**Stage 2 — Name-match fallback (lower confidence)**

For `UnresolvedRef`s not resolved by stage 1:
- Exact name match against `qualified_name` across the whole workspace.
- If exactly one candidate → resolution edge created with `confidence: "name_match"`.
- If multiple candidates → no edge created; record candidates on the `UnresolvedRef` node:

  ```cypher
  MATCH (r:UnresolvedRef {qid: $ref_qid})
  SET r.candidate_qids = $candidate_qids,
      r.unresolved_reason = 'ambiguous'
  ```

- If zero candidates → set `unresolved_reason = 'no_candidate'`.

### Cross-root module resolution

A Daml file under `splice/` with `import Utility.Foo` must resolve to a module inside `utility/`.

- The resolver looks up modules by their **qualified-name string** (e.g., `"Utility.Foo"`) across the full workspace, ignoring `root_name` boundaries. If exactly one `Module` node has that `qualified_name`, the import resolves there.
- The resolver does not parse `Cargo.toml` or `daml.yaml` to derive package boundaries in v1; resolution is purely on extracted qualified names.
- Ambiguity (same `qualified_name` in two roots) → recorded as ambiguous.

### Resolution as idempotent rebuild

`codegraph-rs index` and `codegraph-rs sync` invoke the same resolution routine. The routine:

1. Drops all existing edges with a `confidence` property:
   ```cypher
   MATCH ()-[r WHERE r.confidence IS NOT NULL]-()
   DELETE r
   ```
2. Walks every `UnresolvedRef` and applies stages 1 then 2.
3. Updates `unresolved_reason` and `candidate_qids` on each ref accordingly.

Because `UnresolvedRef`s are permanent, resolution is idempotent — running it twice produces the same edges.

### Performance

- v1: full global resolution pass on every `index` and `sync`. For an `UnresolvedRef` count `R`, complexity is roughly `O(R × log N)` where `N` is the candidate count per name (small in practice). Expect <2s for a 50k-symbol workspace.
- v2 candidate optimization: scope re-resolution to refs whose source file or candidate set changed. Out of v1 scope.

### Not in v1

- Framework resolvers.
- Generic / type-aware dispatch reasoning.
- Cross-crate / external dependency indexing.

## 8. Vector embeddings

### Model

- `fastembed-rs` with `BAAI/bge-small-en-v1.5` (384 dims, ~130 MB ONNX file, ~50 ms per short doc on CPU).
- Auto-downloaded on first use to `~/.cache/codegraph-rs/fastembed/`.
- Single model across all languages.

### What gets embedded

Not every node — embedding `Variable` nodes named `i` is noise. v1 embeds:

- `Function`, `Method`
- `Class`, `Struct`, `Interface`, `Trait`, `Enum`
- `TypeAlias`
- `Module`
- `Template`, `Choice` (when `tree-sitter-daml` ships)

Leaf values (`Variable`, `Constant`, `Parameter`, `Field`, `EnumMember`) are not embedded.

### Composed text

```
<kind>: <qualified_name>
<signature>
<doc>
```

Example:

```
function: crate::util::parse_config
fn parse_config(path: &Path) -> Result<Config, ConfigError>
Parse a config file from disk, returning the parsed settings or a descriptive error.
```

Body content is intentionally not included — keeps the embedding focused on intent rather than implementation detail.

### When

During indexing, after extraction and resolution. Batched at 128 texts per `fastembed-rs::embed()` call. Each batch is written with one Cypher round-trip:

```cypher
UNWIND $batch AS row
MATCH (n:Symbol {qid: row.qid})
SET n.embedding = row.embedding
```

where `row.embedding` is a `LIST<FLOAT>` of length 384.

### Search

```cypher
CALL db.index.vector.queryNodes('symbol_embedding', $k, $queryVec)
  YIELD node, score
WHERE node.root_name IN $roots         // optional
RETURN node, score
ORDER BY score DESC
```

Query vector produced by embedding the user query text with the same model. `$k` defaults to 10. Optional root filter scopes to a subset of roots.

### Sync re-embedding

- Changed file → contained embedded-kind nodes are re-extracted, re-resolved, re-embedded. Old embeddings overwritten.
- Unchanged file → embeddings untouched.

### Storage

Neo4j stores 384-dim float32 vectors inline. ~1.5 KB per embedded symbol. A 50k-symbol workspace ≈ 75 MB. Acceptable.

### Not in v1

- Embedding of full function bodies.
- Custom or alternate models (config supports `model = "..."` but only one valid value).
- Cross-encoder reranking.

## 9. Sync (incremental updates)

### Principle

`codegraph-rs sync` is idempotent and diff-driven. Same end state as a full reindex, but only touches what changed.

### Detection

For each root in the workspace:
1. Walk the filesystem with the `ignore` crate. Collect `{relative_path, content_hash}` using BLAKE3.
2. Query Neo4j: `MATCH (f:File {root_name: $root}) RETURN f.relative_path, f.content_hash`.
3. Diff:
   - **Added** — in filesystem, not in DB.
   - **Changed** — in both, hashes differ.
   - **Deleted** — in DB, not in filesystem.

### Per-bucket actions

**Added** — extract → write nodes, structural edges, and `UnresolvedRef`s. (Resolution runs centrally afterwards.)

**Changed** — delete the file's nodes and everything it contains, including its `UnresolvedRef` records. Other files' `UnresolvedRef` records (which may reference symbols in this file) are NOT touched:

```cypher
MATCH (f:File {root_name: $root, relative_path: $rel})
OPTIONAL MATCH (f)-[:CONTAINS*]->(child:Symbol)
OPTIONAL MATCH (child)-[:HAS_REF]->(ref:UnresolvedRef)
DETACH DELETE f, child, ref
```

The second `OPTIONAL MATCH` reaches each Symbol's own `UnresolvedRef` records (those representing references *from* this file's symbols). Cross-file references — `UnresolvedRef` nodes whose `source_qid` lives in a different file but happen to name a symbol in this file — are unaffected; the global resolution rebuild that follows will recreate cross-file edges against the new targets.

Then re-extract this file, producing new node `qid`s as needed.

**Deleted** — same `DETACH DELETE`, no re-extraction.

### Resolution after partial changes

After the changed/added/deleted set is processed, run the global resolution rebuild from §7 ("Resolution as idempotent rebuild"). All resolution-derived edges are dropped and recomputed from the surviving `UnresolvedRef` records against the now-current candidate set.

This is the mechanism that fixes cross-file edges when a callee's `qid` changes (e.g., file B is edited): file A's `UnresolvedRef` records still describe the original source-code intent, and resolution recreates the `CALLS` edges from A to B's new symbols.

### Embedding after changes

- Newly extracted symbols → embedded in the same batch as initial indexing.
- Unchanged symbols → keep existing embeddings.

### Workspace-level sync

One `codegraph-rs sync` iterates roots in sequence. Parallel-root sync is not v1.

### Consistency

- No transactional sync. If the process crashes mid-sync, the next run recovers via the same diff.
- No locking against concurrent invocations of `codegraph-rs index` or `codegraph-rs sync` on the same workspace — running two at once will cause MERGE conflicts and corrupted intermediate state. Documented as "don't run two at once." A v2 lockfile (`<workspace>/.codegraph/.lock` via `flock`) is planned.

### Not in v1

- Git hooks / file watchers.
- Tree-sitter incremental reparse.

## 10. MCP server

### SDK

- `rmcp` — official Rust MCP SDK from Anthropic. Active, first-party, supports stdio transport out of the box.
- Avoids hand-rolling JSON-RPC framing, the initialize handshake, and tool listing.

### Lifecycle

- `codegraph-rs serve` launches the MCP server on stdio.
- On startup: locate workspace config, open Neo4j connection (Bolt, auth from config), preload the `fastembed-rs` model.
- Shutdown: close the Bolt connection cleanly on stdin EOF or SIGTERM.

### Typical tool flow

A Claude Code session does not know any `qid` values up front. The expected pattern is:

1. `codegraph_rs_search` or `codegraph_rs_semantic_search` to find seed symbols by name or topic — these return `qid`s.
2. `codegraph_rs_node` for full details of a specific `qid`.
3. `codegraph_rs_callers` / `_callees` / `_impact` / `_context` / `_explore` for relationship traversal.

`codegraph_rs_status` and `codegraph_rs_files` are introspection tools and do not require `qid` inputs.

### Tool surface

All prefixed `codegraph_rs_` so they coexist with the TS-version tools without naming collision in Claude Code.

| Tool | Inputs | Returns |
|---|---|---|
| `codegraph_rs_search` | `query: string`, `kinds?: string[]`, `root?: string`, `limit?: number` | Symbol matches via FTS + name prefix |
| `codegraph_rs_semantic_search` | `query: string`, `root?: string`, `limit?: number` | Top-k semantic matches with score |
| `codegraph_rs_callers` | `qid: string`, `depth?: number` | Functions calling this symbol up to `depth` hops |
| `codegraph_rs_callees` | `qid: string`, `depth?: number` | Functions called by this symbol |
| `codegraph_rs_impact` | `qid: string`, `depth?: number` | Reverse-transitive closure |
| `codegraph_rs_node` | `qid: string` | Full details + source snippet |
| `codegraph_rs_files` | `root?: string` | Indexed files with counts |
| `codegraph_rs_context` | `task: string`, `limit?: number` | Curated bundle: semantic hits + neighborhoods |
| `codegraph_rs_explore` | `topic: string`, `depth?: number` | Multi-hop expansion around seed symbols |
| `codegraph_rs_status` | — | Node/edge/file counts, embedding coverage, last sync time |

### Implementation pattern

Each tool is a thin adapter: validate input → call a method on `graph::*`, `vectors::*`, or `context::*` → serialize JSON.

```rust
#[tool(name = "codegraph_rs_callers")]
async fn callers(&self, args: CallersArgs) -> Result<CallersResponse> {
    let results = self.graph.callers(&args.qid, args.depth.unwrap_or(2)).await?;
    Ok(CallersResponse::from(results))
}
```

Heavy logic stays in non-MCP modules; `mcp/tools.rs` stays thin.

### Responses

- Stable JSON structs (`serde::Serialize`); schemas published via MCP `inputSchema`/`outputSchema`.
- Source snippets read on demand from disk using stored `relative_path` + `root_name`. Files are not stored in Neo4j.

### Errors

- Neo4j unreachable → structured MCP error with actionable message ("Neo4j Desktop not running; start the DBMS and retry"). Server stays up.
- Unknown `qid` → structured error with fuzzy-match suggestion.
- Input validation errors → tool-level errors, not server-level.

### Not in v1

- Streaming responses.
- Auth / multi-client.
- Background reindex during `serve` — user runs `codegraph-rs sync` separately.

## 11. CLI surface

```
codegraph-rs init                  scaffold .codegraph/, prompt for Neo4j details,
                                   create the Neo4j database (CREATE DATABASE)
codegraph-rs index                 full build
codegraph-rs sync                  incremental
codegraph-rs status                stats
codegraph-rs query <text>          codegraph_rs_search from terminal (FTS / name search only)
codegraph-rs serve                 MCP stdio server
```

All commands accept `--workspace <path>`. The CLI exposes only `query` for terminal use; richer queries (semantic, callers, callees, impact) are reachable only via the MCP interface, intentionally — that surface is designed for AI assistants, not humans. No `codegraph-rs hooks install` in v1.

## 12. Distribution

- `cargo install codegraph-rs` is the v1 distribution channel. Users need a Rust toolchain first.
- No pre-built binaries, no Homebrew formula in v1. Both can be added later with minimal change.
- Neo4j is **not** distributed with the tool. Documentation directs users to install Neo4j Desktop (free for local development) and provides a "first 5 minutes" walkthrough.

## 13. Testing

### Unit tests

- `config/` — TOML parsing edge cases, schema version mismatch, missing roots, path canonicalization.
- `extraction/walker` — gitignore honoring, root-scoped iteration, extension → language mapping.
- `extraction/languages/*` — small inline Rust/Daml snippets, run extractor, assert on `RawNode`/`RawEdge` shape and counts. One fixture per language.
- `resolution/` — synthetic in-memory graph + unresolved-ref lists. No Neo4j needed.
- `graph/traversal` — in-memory graph, assert traversal results.
- `vectors/embedder` — gated on `CODEGRAPH_RS_TEST_EMBEDDINGS=1` since loading the model is slow and downloads weights. Default-off in CI.

### Integration tests (real Neo4j)

- Test fixture spins up a fresh dedicated database (`codegraph-test-<pid>` — dashes per Neo4j 5 naming rules in §5).
- Tests annotated `#[tokio::test]` with `#[ignore]` unless `CODEGRAPH_RS_TEST_NEO4J=1` is set.
- Each test:
  1. Writes a small codebase into a `tempfile::tempdir()`.
  2. Runs the full indexing pipeline against the test DB.
  3. Asserts via Cypher.
- Checked-in fixtures under `tests/fixtures/`:
  - `tiny-rust/` — 3-file Rust project with cross-module calls.
  - `tiny-daml/` — Daml project mirroring real Daml shape.
  - `multi-root/` — two roots with cross-root imports; used for the workspace integration test.

### End-to-end MCP test

A single integration test launches the MCP server in-process, sends JSON-RPC requests through an in-memory channel pair, and asserts response shapes. Covers initialize → list_tools → call_tool for each tool → shutdown.

### CI

GitHub Actions, two jobs:

- **Fast** (always runs): unit tests + compile + clippy. No Neo4j.
- **Slow** (on PR label or nightly): integration tests + MCP test. Uses a `neo4j:5.x-enterprise` service container with CI-only license acceptance.

Plus `cargo deny` for license/advisory checks.

## 14. Risks and open questions

### Risks

| Risk | Mitigation |
|---|---|
| `tree-sitter-haskell` parse quality on real Daml templates is poor enough to produce misleading `Function` nodes from template bodies | Accept as v1 limitation. Set `parse_reliable = false` on nodes extracted from inside template/choice/signatory blocks (see §4 schema). Document the gap in README until `tree-sitter-daml` ships |
| `fastembed-rs` model download (~130 MB) can fail on slow connections | Fail with clear retry message; document cache location; plan a `codegraph-rs prepare` command in v2 |
| `neo4rs` driver maturity / Bolt edge cases | Pin a known-good version. Wrap behind `db/` so the driver can be swapped locally |
| Neo4j Enterprise licensing for users distributing the tool or running in CI | v1 documents "use Neo4j Desktop (free for local dev)"; CI accepts license explicitly. Tool itself never bundles Neo4j |
| Parallelism bug surface (rayon + tokio + tree-sitter thread-locals) | Narrow join points: rayon collects, channel hands off, tokio writes. No shared mutable state across tasks |
| `codegraph_rs_explore` response sizes growing unmanageable | Default response budget (~50 KB) with `limit` parameter and a "truncated — use more specific query" marker |
| Pre-1.0 dependencies (`fastembed-rs`, `rmcp` may evolve) | Pin specific versions in `Cargo.toml`. Wrap behind module boundaries |

### Open questions

1. **Eager vs deferred initial embedding generation.** *Proposed:* run eagerly during the first index; add `--skip-embeddings` for impatient bootstrapping.
2. **`codegraph-rs init` ergonomics.** *Proposed:* interactive by default; `--non-interactive` plus flags for scripting.
3. **Tree-sitter grammar pinning.** *Proposed:* pin in `Cargo.toml` only; do not surface in workspace config.

## 15. Out of scope (intentionally deferred)

- Visualizer web UI (Neo4j Browser replaces it).
- Framework-specific resolvers.
- Auto-installed git hooks.
- Languages other than Rust and Daml.
- Pre-built binaries / Homebrew distribution.
- A real `tree-sitter-daml` grammar — separate project; can swap into `extraction/languages/daml.rs` with a one-line change when ready.
- Per-root language overrides.
- Body-content embeddings, custom embedding models, cross-encoder reranking.
- Cross-crate / cross-package external dependency indexing.

## 16. Decisions log

Decisions made during the brainstorming phase, in chronological order:

| # | Decision | Rationale |
|---|---|---|
| 1 | Full replacement of TS version | User intent |
| 2 | Storage: Neo4j Desktop (Enterprise), Bolt protocol | Visualization driver + multi-database support, accepted install friction |
| 3 | Embeddings: `fastembed-rs` + Neo4j vector index | Single store, native vector index in Neo4j, pure-Rust embedding lib |
| 4 | v1 modules: extraction, db, graph, mcp, sync, resolution, vectors, context | Drop visualizer, framework resolvers, installer wizard for v1 |
| 5 | v1 languages: Rust + Daml | Rust = author preference + dogfooding; Daml = primary user language |
| 6 | Tree-sitter binding: native `tree-sitter` crate | Rust ecosystem standard; faster than WASM |
| 7 | Daml grammar in v1: `tree-sitter-haskell` (proxy) | Real `tree-sitter-daml` is a separate project; Haskell covers ~80% |
| 8 | Multi-project = workspace with multiple source roots | User wants cross-project Daml graph (splice + utility) |
| 9 | Database isolation: one Neo4j database per workspace | Hard isolation, clean queries; supported by Neo4j Enterprise |
| 10 | MCP tool naming: `codegraph_rs_*` | Coexist with TS version's `codegraph_*` tools without collision |
| 11 | Distribution: `cargo install` | Simplest for v1; users have Rust if they're using Rust tooling |
| 12 | Crate layout: single crate, internal modules | Premature workspace splitting adds friction without payoff |
| 13 | `qid` excludes `start_line`; permanent `(:UnresolvedRef)` records as source-of-truth | Found during spec review: line-shift edits and cross-file edges would silently corrupt the graph if `qid` depended on position or if unresolved refs were transient |
| 14 | No separate `Impl` node for Rust `impl` blocks | Methods carry the type prefix in `qualified_name`; structural simplicity over a parallel hierarchy |
