# codegraph-rs — choice effect edges (CREATES / EXERCISES / FETCHES)

**Status:** Implemented — slice 12 on `main` (codegraph-rs PR #23).
**Baseline:** the external-package layer (`2026-06-12-codegraph-rs-external-packages-design.md`, slices 11a–11e) — this work reuses its vendored Daml-LF decoder and DAR ingestion.
**Scope:** Graph edges from a Daml `Choice` to the contracts it creates/fetches and the choices it exercises, derived from the **compiled Daml-LF**.

---

## 1. Goal

Answer "what does this choice do to the ledger?" — for each `Choice`, emit:

| Edge | Direction | Meaning |
|---|---|---|
| `CREATES` | Choice → Template | the choice body creates a contract of that template |
| `EXERCISES` | Choice → Choice | it exercises another choice (incl. `Archive`, cross-package) |
| `FETCHES` | Choice → Template | it fetches a contract of that template |

Targets are resolved and **package-qualified**, including cross-package (e.g. a wallet choice that exercises `AmuletRules_Transfer` in another package).

## 2. Why Daml-LF, not source (the key decision)

We spiked both. **Source extraction is structurally incomplete** for this: the target is frequently not syntactic — `create delegation` (a value, not a constructor) and `exercise cid arg` hide the template/choice type, which only type inference recovers. The MintingDelegation fixture's own `create delegation` would be missed.

**Daml-LF has the exact target** (`Update.Create{template: TypeConId}`, `Update.Exercise{template, choice}`, …) because it is fully typed/elaborated — it recovers `create delegation` as `Update.Create{MintingDelegation}`. The cost is two LF realities:

1. **Hoisting.** The compiler hoists choice logic into top-level values and routes create/exercise/fetch through generated `$$fHasCreate/Exercise/Fetch…` typeclass-instance values. A choice's *direct* body is a thin wrapper (`App` of a hoisted value); the `Update` nodes live transitively behind value references.
2. **Depth.** Those `Update` nodes sit arbitrarily deep inside `Expr` trees — past prost's hard, unconfigurable recursion limit (`RECURSION_LIMIT = 100`), so the bodies can't be decoded with prost at all.

## 3. Design

### 3.1 Wire-scanner (slice 12a — `src/lf/update_scan.rs`)
Capture `TemplateChoice.update` and `DefValue.expr` as raw **`bytes`** in the vendored proto (the names pass doesn't read them, so it's unaffected). Walk the protobuf wire format ourselves with an **iterative worklist (no recursion cap)** over borrowed sub-slices, collecting per body:
- **effects** — `Update.{Create=3, Exercise=4, Fetch=5, ExerciseByKey=10}` targets. `TypeConId`/`ValueId` resolve to `Module.Name` via the package intern tables; out-of-range indices are rejected, so misreading a string/Type field as a message yields nothing.
- **value refs** — `Expr.val` (`ValueId`), for the call graph.

Relevant LF2 field map: `Expr.{val=3, update=23}`, `Update.{create=3, exercise=4, fetch=5, exercise_by_key=10}`, `Create/Exercise/Fetch.template=1`, `Exercise.choice_interned_str=6`, `TypeConId|ValueId{module=1, name_interned_dname=2}`, `ModuleId.module_name_interned_dname=2`.

### 3.2 Attribution (slice 12b — `src/lf/effects.rs`)
Scan every value and choice body across the decoded packages, then take the **transitive closure** of each choice's effects over the value call-graph (memoized; cycle-safe — a value already on the path contributes nothing further). This recovers the hoisted effects: choice → wrapper value → … → the instance value holding the `Update`.

### 3.3 Emission (`packages::ingest_dars`, in the `extern` stage)
- Ingest external **`Choice`** stubs (choices aren't top-level LF defs, so the names pass misses them) so cross-package `EXERCISES` targets exist as nodes.
- After writing nodes, MERGE `CREATES`/`EXERCISES`/`FETCHES` from each **local** source `Choice` (matched by `qualified_name`, `external IS NULL` — the DALF choice qname `Module.Template.Choice` equals the source node's) to a **single** target (`CALL` subquery, local-first then deterministic external).
- Idempotent: clears the three edge types first.

## 4. Bugs caught by corpus verification

Both surfaced only at corpus scale, not in the gated single-DAR test:

- **Scanner cloned sub-messages** on every descent (`O(size×depth)` allocation) → the extern stage took *minutes* across ~206 packages' value bodies. Fixed by borrowing `&[u8]` sub-slices (per-DAR scan dropped to seconds).
- **Edge targets duplicated per package version** — a template is ingested once per DAR version, so an unguarded target match emitted one edge per version: **~148k** edges vs the 7,810 distinct effects. Fixed by the local-first single-target `CALL` subquery (→ ~7.5k edges).

## 5. Corpus result (`codegraph-splice`)
1,506 `CREATES` + 4,376 `EXERCISES` + 1,626 `FETCHES`. Verified exact, e.g.:
- `MintingDelegationProposal_Accept` → CREATES/FETCHES `MintingDelegation`, EXERCISES `MintingDelegation.Archive`
- `MintingDelegation_Mint` → EXERCISES `AmuletRules_Transfer` (cross-package)

## 6. Limitations
- **Edges originate from local source choices only** (`external IS NULL`); the analysis would support external→external edges but they're not emitted (not the use case, and would inflate the graph).
- **Transitive over-attribution**: a shared hoisted helper that performs an effect attributes that effect to every choice transitively reaching it. This is semantically correct (the choice *can* cause the effect) but can be broader than the syntactic call.
- **Generic wire descent** recurses into every length-delimited field; intern-table validation filters false targets, but the walk does visit non-`Expr` fields. Bounded by a budget.
- Edges are emitted only where the target node exists; ~300 of the 7,810 distinct effects target a choice/template in a package not ingested → no edge (degrades gracefully).

## 7. Tests
- Unit: wire-scanner over hand-built bytes (incl. out-of-range rejection); transitive-closure over synthetic call-graphs (multi-hop, cycles, no-effect omission).
- Gated integration: scanner over a real DAR (246 effects); `emits_choice_effect_edges` stubs source choices + ingests the wallet DAR and asserts `CREATES`/`FETCHES`/`EXERCISES` (incl. a cross-package EXERCISES to an ingested external `Choice` stub).
