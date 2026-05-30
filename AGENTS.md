# Terminus — Agent Guidance

Read [SPECIFICATION.md](SPECIFICATION.md) before making structural decisions. It is the source of truth for product behavior.

Read [ARCHITECTURE.md](ARCHITECTURE.md) for Terminus-specific scan orchestration, progress flow, and model ownership decisions.

---

## Rust Project — Type-Driven Functional Architecture

## Philosophy

Rust's ownership system already enforces discipline that other languages simulate with
patterns. This codebase leans into that: the type system carries invariants, the borrow
checker enforces boundaries, and algebraic data types model the domain. Most GoF patterns
collapse into enums, traits, and generics. Complexity lives at compile-time, not runtime.

Three axioms drive every decision:

1. **If it compiles, it's valid.** Encode business rules in types so invalid states cannot exist.
2. **Pure core, thin shell.** Domain logic computes. Infrastructure executes. They never mix.
3. **Ownership is architecture.** Who owns a value, who borrows it, and who consumes it — these
   are not implementation details, they *are* the design.

### Scope and applicability

This doctrine is optimized for service-style Rust systems: backend APIs, orchestration-heavy
applications, and business workflows with strong invariants and integration boundaries.

For game engines, databases, kernels, realtime simulation, embedded systems, or other
throughput/latency-dominated domains, treat this as a starting point and bias toward
data-oriented layout, locality, and explicit mutation where that wins.

---

## Architecture: Launcher / Core / Adapters / GUI

```
          ┌─────────────────────────────┐
          │          GUI Crate          │  egui/eframe rendering only
          │   reads model + progress    │
          └────────────┬────────────────┘
                       │ user intents / render state
          ┌────────────▼────────────────┐
          │      Launcher Binary        │  detects platform, owns refresh lifecycle
          │ starts worker, swaps model  │
          └────────────┬────────────────┘
                       │ shared contracts / scan results
          ┌────────────▼────────────────┐
          │        terminus-core        │  model, matching, classification, persistence
          │ builds SystemModel snapshots│
          └────────────┬────────────────┘
                       │ PlatformAdapter trait
          ┌────────────▼────────────────┐
          │      Platform Adapters      │  pacman/dpkg parsing, package-manager access
          │  emit snapshot scan results │
          └─────────────────────────────┘
```

- The launcher owns scan lifecycle, freshness checks, progress polling, and atomic model replacement.
- `terminus-core` owns shared data types, `SystemModel` construction, matching, classification, config discovery helpers, and SQLite persistence.
- Platform adapters own package-manager interaction and platform-specific parsing only.
- `terminus-gui` renders current model and progress state only; it never imports platform crates, touches SQLite, or shells out.
- Keep matching, classification, alias normalization, and model-building logic pure where practical, even when the surrounding crate also contains imperative helpers.

---

## Crate Layout

```
Cargo.toml                      # Workspace manifest + launcher binary package
src/
└── main.rs                     # Launcher: platform detection, cache load, scan orchestration, eframe start
crates/
├── terminus-core/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── model.rs            # Core entities and value types
│       ├── system_model.rs     # In-memory query layer used by the UI
│       ├── db.rs               # SQLite persistence
│       ├── platform.rs         # PlatformAdapter trait and shared platform contracts
│       ├── matcher.rs          # Config-to-package matching
│       ├── classify.rs         # File/config classification
│       └── aliases.rs          # Known name aliases and normalization helpers
├── terminus-platform-arch/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs              # ArchAdapter implementation
│       ├── pacman.rs           # pacman output parser
│       └── detect.rs           # Arch platform detection
├── terminus-platform-ubuntu/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs              # UbuntuAdapter implementation
│       ├── dpkg.rs             # dpkg parser + .list reader
│       └── detect.rs           # Ubuntu platform detection
└── terminus-gui/
    ├── Cargo.toml
    └── src/
        ├── lib.rs              # Render-only app shell
        ├── app.rs              # App state and tab routing
        ├── views/
        │   ├── mod.rs
        │   ├── packages.rs
        │   ├── configs.rs
        │   ├── disk.rs
        │   └── loading.rs
        ├── widgets/
        │   ├── mod.rs
        │   ├── detail_panel.rs
        │   ├── search.rs
        │   ├── confidence.rs
        │   └── treemap.rs
        └── theme.rs
```

**Rules:**
- The root package is the launcher. It owns platform detection, scan lifecycle, cache load/save, and wiring between adapters, core, and GUI.
- `terminus-core` is platform-agnostic. It owns the persistent model, `SystemModel`, matching, classification, config discovery inputs, and shared `PlatformAdapter` contracts.
- `terminus-gui` depends only on `terminus-core`. It renders data and emits user intents; it does not import or detect platforms.
- Platform crates implement `PlatformAdapter` and parse platform-specific package manager state. They do not depend on `terminus-gui`.
- The launcher binary is the only place where platform crates are selected and wired into the running app.
- Adding a new platform means adding one new `terminus-platform-*` crate and one launcher registration, with no changes required in `terminus-core` or `terminus-gui` unless the shared contract changes.

When a Terminus concern grows beyond 3–4 files, give it a focused submodule inside the owning crate instead of forcing a generic service-style split. For example, in `terminus-core`:

```
terminus-core/src/
├── matching/
│   ├── mod.rs
│   ├── package_names.rs        # Name normalization and alias lookup
│   ├── config_paths.rs         # Config discovery input shaping
│   ├── ownership.rs            # Config ownership and multiplicity rules
│   └── confidence.rs           # Match confidence scoring
└── system_model.rs
```

---

## Boundary Parsing

All external data is untrusted. Parse it into domain types at the boundary — once. After
parsing succeeds, the domain type is trusted everywhere. Never validate again.

**Inbound:** `TryFrom` / `TryInto` (fallible — external data can be invalid).
**Outbound:** `From` / `Into` (infallible — the domain type is already valid).

### Adapter record → core package

```rust
// crates/terminus-platform-arch/src/pacman.rs
pub struct PacmanPackageRecord {
    pub name: String,
    pub version: String,
    pub install_reason: String,
}

// Parsing happens HERE — boundary between raw adapter data and core types
impl TryFrom<PacmanPackageRecord> for Package {
    type Error = AdapterParseError;

    fn try_from(record: PacmanPackageRecord) -> Result<Self, Self::Error> {
        Ok(Package {
            id: 0,
            name: record.name.trim().to_owned(),
            version: record.version,
            source: PackageSource {
                manager_id: "pacman".to_owned(),
                channel: None,
            },
            size_bytes: None,
            install_date: None,
            install_reason: InstallReason::parse(&record.install_reason)?,
            description: None,
        })
    }
}
```

### SQLite row → core mapping

```rust
// crates/terminus-core/src/db.rs
struct ConfigMappingRow {
    config_path: String,
    package_id: i64,
    confidence: f64,
    method: String,
    is_primary: i64,
}

impl TryFrom<ConfigMappingRow> for ConfigMapping {
    type Error = DataCorruptionError;

    fn try_from(row: ConfigMappingRow) -> Result<Self, Self::Error> {
        Ok(ConfigMapping {
            id: 0,
            config_path: CanonicalConfigPath::parse(row.config_path)?.into_string(),
            package_id: row.package_id,
            confidence: row.confidence,
            method: MatchMethod::parse(&row.method)?,
            is_primary: row.is_primary != 0,
        })
    }
}
```

### User search text → query type

```rust
pub struct SearchQuery(String);

impl SearchQuery {
    pub fn parse(raw: impl Into<String>) -> Result<Self, ValidationError> {
        let query = raw.into().trim().to_lowercase();
        if query.is_empty() {
            Err(ValidationError::EmptySearchQuery)
        } else {
            Ok(Self(query))
        }
    }
}
```

### Completed refresh → live model swap

```rust
pub struct CompletedRefresh {
    pub scan: ScanResult,
    pub model: SystemModel,
}

impl ScanCoordinator {
    pub fn commit_refresh(&mut self, completed: CompletedRefresh) -> Result<(), LauncherError> {
        persist_snapshot(&completed.scan)?;
        self.live_model = Some(completed.model);
        Ok(())
    }
}
```

---

## Type-Driven Design

### Core rules

- **Parse, don't validate.** Parse once at boundaries, trust types forever after.
- **Make illegal states unrepresentable.** Use enums with variant-specific data, not structs with optional fields.
- **Newtypes over primitives.** No raw `String`, `i64`, or path strings in cross-crate core signatures when the value has real semantics.
- **Typestate for workflows.** Model state machines as generic type parameters. Invalid transitions must not compile.
- **ADTs are the modeling language.** Enums + structs replace class hierarchies.
- **Value objects over mutable entities.** State transitions return new values.

### Newtype pattern

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct CanonicalConfigPath(String);

impl CanonicalConfigPath {
    pub fn parse(raw: impl AsRef<str>) -> Result<Self, ValidationError> {
        let expanded = expand_home(raw.as_ref())?;
        let normalized = normalize_path(&expanded)?;
        if normalized.is_empty() {
            Err(ValidationError::EmptyPath)
        } else {
            Ok(Self(normalized))
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }

    pub fn into_string(self) -> String { self.0 }
}

// No pub constructor. The only way to get a CanonicalConfigPath is through parse().
// After parse() succeeds, the path is trusted everywhere — no re-normalization.
```

### Making illegal states unrepresentable

```rust
// BAD — optional fields invite invalid combinations
struct ConfigMatch {
    path: String,
    owner: Option<i64>,
    shared_with: Vec<i64>,
    is_orphaned: bool,
}

// GOOD — each state carries exactly its valid data
enum ConfigOwnership {
    Orphaned { path: CanonicalConfigPath },
    Owned { path: CanonicalConfigPath, primary_package_id: i64, candidates: Vec<ConfigMapping> },
    Shared { path: CanonicalConfigPath, primary_package_ids: Vec<i64>, candidates: Vec<ConfigMapping> },
}
```

### When typestate is worth it and when it isn't

**Use typestate** when: the state machine has 3+ states, invalid transitions cause real bugs,
and the type is constructed and consumed across module boundaries.

**Use a plain enum** when: the states are data-only (no behavior differs by state), or the
type is local to a single function.

**Use a smart constructor** (newtype + `parse`) when: there's no state machine, just an
invariant to enforce (e.g., canonical config path, non-empty package name, valid manager id).

---

## Error Modeling

Errors are typed, layered, and never stringly typed.

### Layer separation

```rust
// crates/terminus-core/src/error.rs — model and persistence failures
#[derive(Debug, thiserror::Error)]
pub enum CoreError {
    #[error("database query failed")]
    Database(#[from] rusqlite::Error),
    #[error("config path is not canonical: {path}")]
    InvalidConfigPath { path: String },
    #[error("unknown match method: {value}")]
    InvalidMatchMethod { value: String },
}

// crates/terminus-platform-arch/src/error.rs — adapter failures
#[derive(Debug, thiserror::Error)]
pub enum AdapterError {
    #[error("command `{command}` failed: {stderr}")]
    CommandFailed { command: &'static str, stderr: String },
    #[error("could not parse package metadata from {manager_id}: {detail}")]
    ParsePackage { manager_id: &'static str, detail: String },
    #[error("package state probe failed for {manager_id}: {detail}")]
    FreshnessCheck { manager_id: &'static str, detail: String },
}

// src/error.rs — launcher orchestration failures
#[derive(Debug, thiserror::Error)]
pub enum LauncherError {
    #[error(transparent)]
    Core(#[from] CoreError),
    #[error(transparent)]
    Adapter(#[from] AdapterError),
    #[error("GUI startup failed: {detail}")]
    GuiStartup { detail: String },
}
```

### Rules

| Layer          | Error type             | Crates allowed                        |
|----------------|------------------------|---------------------------------------|
| terminus-core  | Core-specific enum     | `thiserror`, DB/serde wrappers as needed |
| Platform adapter | Adapter-specific enum | `thiserror`, process/fs wrappers      |
| Launcher       | Composite enum         | `thiserror`, `From` impls for layers  |
| GUI            | UI mapping only        | `thiserror` or `anyhow` sparingly     |
| `main.rs`      | Top-level              | `anyhow` / `eyre` allowed             |

- `anyhow` / `eyre` never appear in shared core model code or adapter parsing code.
- Error variants carry **structured context** (paths, manager ids, package ids, scan stages) — not format strings.
- `?` composes errors through `From` impls. Write the `From` chain explicitly.
- Core errors are **model/persistence** failures. Adapter errors are **scan input** failures. Launcher errors are **orchestration/startup** failures.

---

## Behavior Rules

- **Tell, don't ask.** Don't inspect state then decide — give types the info they need and let them act.
- **Railway-oriented programming.** Chain operations through `Result`/`Option` pipelines with `and_then`, `map`, `map_err`.
- **Total functions.** Handle all input cases. No panics on valid input.
- **Exhaustive matching.** No `_` catch-alls unless justified (e.g., `#[non_exhaustive]` from external crates).
- **Cancellation-safe orchestration.** Treat every blocking scan step, DB write, and model swap as an interruption point. Keep the last good cache and last good `SystemModel` intact until the replacement snapshot is complete.

When this is required by default: any workflow that refreshes package state, persists a new snapshot, or replaces the live `SystemModel`.
Preferred patterns: temp-db write then swap, complete `ScanResult` snapshots, idempotent refresh triggers, and one-way model replacement only after success.

### Side-effect delivery semantics

- Name replacement semantics explicitly per workflow: cached-only, full-refresh, or refresh-required.
- Treat partial live updates as a bug for MVP; incomplete scans must fail without replacing the last good state.
- For critical scan effects, define:
    - freshness source,
    - temp persistence target,
    - swap point,
    - observable failure behavior.
- Prefer durable snapshot persistence before exposing a newly scanned model to the GUI.

### Function design

- Small: one level of abstraction per function.
- Single-purpose: one reason to change.
- Return values over performing side effects.
- Closures for behavior injection by default; traits only when closures are insufficient
  (object safety needed, or the behavior has multiple methods).

### Mutation discipline

- Default to immutable. State transitions return new values.
- `mut` only when performance requires it **and** mutation is local to the function.
- Never expose `&mut self` across module boundaries without justification.
- Interior mutability (`Cell`, `RefCell`, `Mutex`) is a code smell in shared model ownership and cross-thread scan state — justify every use.
- Prefer ownership-first APIs over incidental cloning, but freely clone in cold paths or for small values when it improves clarity.

### Semantic ownership vs memory ownership

- Memory ownership answers "who drops this value?".
- Semantic ownership answers "which bounded context is the source of truth?".
- Keep these separate in design reviews: a type may be memory-owned in one module while semantically owned by another context.
- APIs should preserve semantic authority boundaries even when Rust memory ownership is moved, cloned, or shared via `Arc`.

---

## Dependency Injection

- Traits define **capabilities** at crate boundaries: `trait PlatformAdapter` is the primary example in Terminus.
- Concrete implementations live in `terminus-platform-*` crates.
- The launcher selects an adapter implementation and passes its results into core-owned model building.
- Test doubles implement the same traits.

### When to introduce a trait

| Situation                                              | Use a trait? |
|--------------------------------------------------------|--------------|
| 2+ real implementations (Arch + Ubuntu adapters)       | Yes          |
| 1 real impl but need test doubles for scan lifecycle   | Yes          |
| Pure matcher/classifier helper                         | No — just call it |
| Single launcher/core helper, easily tested directly    | No — premature abstraction |

Closures are often enough for simple injection:

```rust
fn rank_config_candidates(
    path: &CanonicalConfigPath,
    candidates: &[Package],
    score: impl Fn(&CanonicalConfigPath, &Package) -> f64,
) -> Vec<ConfigMapping> {
    candidates
        .iter()
        .map(|package| build_mapping(path, package, score(path, package)))
        .collect()
}
```

---

## Naming

- Types represent **meaning**, not structure: `ManagerId` not `ValidatedString`.
- Functions describe **what**, not **how**: `refresh_model` not `run_pacman_and_write_db`.
- Constructors: `new`, `parse`, `with_capacity`, `from_*`.
- Conversions: `to_*` (allocates), `as_*` (borrowed view), `into_*` (consuming).
- Booleans: `is_` or `has_` prefix.
- Ban vague names like `data`, `info`, `item`, `utils`, `helpers`, `misc`.
- Avoid suffixes like `Manager` or `Service` for types that are actually a parser, matcher, coordinator, or cache.
- Use domain vocabulary from the business — not programmer jargon.

---

## Pattern Palette

**Use freely:** newtypes, ADTs (enum modeling), `TryFrom`/`From` at boundaries, `Result`/`Option` pipelines, tell-don't-ask, function composition, CQRS command/query split, module facades, traits-as-capabilities, smart constructors, exhaustive matching.

**Use when justified:** typestate (3+ states, cross-module), builder (many optional params), trait-based strategy (when closures won't do — multiple methods needed), `Arc<dyn Trait>` (genuinely heterogeneous collections).

**Avoid:** Visitor (use `match`), Singleton (use DI), classic Factory (use `TryFrom`/`parse`), service objects holding domain logic, `Box<dyn Any>` downcasting, marker traits without compiler enforcement.

---

## Anti-Patterns to Reject

- `ScanManager` / `PackageService` god objects accumulating launcher, adapter, and model responsibilities.
- Raw `String`, `i64`, or path strings leaking through cross-crate core APIs where the value has stronger semantics.
- Shelling out to `pacman`, `dpkg`, or any platform tool from `terminus-core` or `terminus-gui`.
- `Err("something went wrong".into())` — string errors.
- `_ => {}` silencing the compiler on core enums like `FileType`, `MatchMethod`, or `InstallReason`.
- `terminus-core` importing from `terminus-platform-*` or `terminus-gui`.
- Validating the same invariant in multiple places after the boundary.
- Traits with a single implementor and no test double.
- `&mut self` as a default in cross-module APIs — justify every mutation.
- `clone()` in hot paths or large-object loops without a reason — redesign ownership or document why clone is acceptable.
- Mutating the live `SystemModel` in place while a scan is still running.
- God-mode `App` or `Context` struct passed everywhere.

---

## Testing Strategy

### By layer

**Core (unit tests — pure or mostly pure, fast, no real package manager):**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_alias_maps_nvim_to_neovim() {
        assert_eq!(resolve_alias("nvim"), Some("neovim"));
    }

    #[test]
    fn shared_primary_owners_are_preserved_per_path() {
        let mappings = vec![
            ConfigMapping {
                id: 1,
                config_path: "/home/user/.config/foo/config.toml".into(),
                package_id: 10,
                confidence: 0.92,
                method: MatchMethod::KnownAlias,
                is_primary: true,
            },
            ConfigMapping {
                id: 2,
                config_path: "/home/user/.config/foo/config.toml".into(),
                package_id: 11,
                confidence: 0.90,
                method: MatchMethod::NameMatch,
                is_primary: true,
            },
        ];

        let grouped = group_configs_by_path(&mappings);
        assert_eq!(grouped["/home/user/.config/foo/config.toml"].len(), 2);
    }
}
```

**Boundaries (TryFrom tests — valid and invalid inputs):**

```rust
#[test]
fn pacman_install_reason_parses() {
    assert!(InstallReason::parse("Explicitly installed").is_ok());
}

#[test]
fn canonical_config_path_rejects_empty_input() {
    assert!(matches!(
        CanonicalConfigPath::parse("   "),
        Err(ValidationError::EmptyPath)
    ));
}

#[test]
fn config_mapping_row_with_unknown_method_rejects() {
    let row = ConfigMappingRow {
        config_path: "/home/user/.config/foo/config.toml".into(),
        package_id: 10,
        confidence: 0.8,
        method: "wat".into(),
        is_primary: 1,
    };

    assert!(matches!(
        ConfigMapping::try_from(row),
        Err(DataCorruptionError::InvalidField(..))
    ));
}
```

**Launcher + adapter integration tests (real orchestration, fake adapters):**

```rust
#[test]
fn coordinator_swaps_model_only_after_complete_scan() {
    let adapter = FakeAdapter::success(scan_result_with_counts(42, 900, 17));
    let mut coordinator = ScanCoordinator::new(adapter, last_good_model());

    let completed = coordinator.refresh().unwrap();

    assert_eq!(completed.model.meta.package_count, 42);
    assert_eq!(coordinator.live_model().meta.package_count, 42);
}
```

**Property tests (for value objects and parsers):**

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn canonical_config_path_parse_is_idempotent(path in "/[a-z0-9_./-]{1,64}") {
        let parsed = CanonicalConfigPath::parse(&path).unwrap();
        let reparsed = CanonicalConfigPath::parse(parsed.as_str()).unwrap();
        prop_assert_eq!(parsed, reparsed);
    }

    #[test]
    fn manager_id_is_always_lowercase(s in "[A-Za-z0-9_-]{1,16}") {
        if let Ok(manager_id) = ManagerId::parse(s) {
            prop_assert_eq!(manager_id.as_str(), manager_id.as_str().to_lowercase());
        }
    }
}
```

### Test naming

`<unit>_<scenario>_<expected>`: `known_alias_maps_nvim_to_neovim`, `canonical_config_path_rejects_empty_input`.

### What not to test

- Private helper functions — test through the public API.
- Framework glue (eframe startup wiring, serde derives) — unless custom logic is involved.
- Things the compiler already guarantees (typestate transitions that don't compile).

### Cancellation safety tests

- For each refresh workflow, include at least one test that simulates cancellation or interruption during scan work.
- Assert externally observable consistency after cancellation (last good DB remains valid, live `SystemModel` is unchanged, no partial swap occurs).

```rust
#[test]
fn cancelled_scan_keeps_last_good_model() {
    let adapter = BlockingAdapter::new(valid_scan_result());
    let previous = last_good_model();
    let mut coordinator = ScanCoordinator::new(adapter.clone(), previous.clone());

    coordinator.start_refresh();
    adapter.wait_until_stage(ScanProgress::FilesIndexed(1));
    coordinator.cancel_refresh();

    assert_eq!(coordinator.live_model().meta.scan_date, previous.meta.scan_date);
}
```

### Core invariant checklist

For each core type and query surface, document and test:

- Constructor invariants (what `new`/`parse` guarantees forever after).
- Ownership invariants (how primary/shared/orphaned config states are represented).
- Persistence invariants (DB roundtrip cannot create invalid core states).
- Search/grouping invariants (`packages_by_name`, `configs_by_path`, and `reverse_deps` stay consistent).
- Freshness/swap invariants (failed scans never replace the last good cache or model).

---

## Reliability and Operations

### Timeout and retry policy

Set defaults centrally (override only with explicit justification):

| Operation type                       | Timeout | Retries | Backoff                | Notes |
|--------------------------------------|---------|---------|------------------------|-------|
| Package-manager snapshot commands    | 1-10s   | 0-1     | small fixed/jitter     | Fall back to the last good model on failure. |
| Filesystem config discovery          | 1-5s    | 0       | none                   | Unreadable individual paths should degrade partially, not fail the whole scan. |
| SQLite persist + model load          | 1-3s    | 0-1     | small fixed/jitter     | Swap only after complete success. |
| Freshness checks                     | 0.1-1s  | 0-1     | small fixed            | Cheap enough to run on startup or refresh. |

Rules:

- Retry only transient OS/process failures (timeouts, temporary locks, interrupted commands), never parse or model-correlation failures.
- Every retry policy must define max attempts and total retry budget.
- Cancellation must stop retries immediately unless the new snapshot has already been durably persisted and is ready to swap.
- Surface retry exhaustion as a typed error variant with operation context.

### Observability requirements

Every scan lifecycle step and persistence side effect must emit:

- Structured logs (JSON or key-value) with scan/request correlation data and package/config identifiers.
- Tracing spans around each scan stage, adapter command, filesystem walk, and persistence step.
- Result markers for refresh side effects: `attempted`, `succeeded`, `failed`, `retried`, `skipped`, `swapped`.

Minimum fields for logs/spans:

- `scan_id` / `correlation_id`
- stage name
- primary identifiers (`platform_id`, `package_id`, `config_path`, etc.)
- retry attempt index (if any)
- timeout/cancellation status
- typed error class/variant on failure

---

## Rust Runtime and Boundary Rules

### Async boundary constraints (`Send`/`Sync`/`'static`)

- Values crossing task boundaries (`tokio::spawn`, scan workers, background workers) must satisfy runtime constraints explicitly (`Send + 'static` when required).
- Avoid borrowing non-`'static` references into spawned tasks; move owned data instead.
- Prefer `Arc<T>` + immutable state over shared mutable state; if mutation is necessary, justify synchronization primitives at module boundary.
- Treat `spawn_blocking` as a boundary for CPU/blocking IO work only; keep domain logic pure regardless.

### Panic policy

- `panic!`/`unwrap()`/`expect()` are forbidden in production paths in `terminus-core`, `terminus-platform-*`, launcher orchestration, and GUI update logic.
- Allowed in tests and irrecoverable bootstrap failures in `main.rs` (with clear crash message).
- Library code should return typed errors, not panic, for any invalid runtime input or dependency failure.
- If a panic boundary exists (FFI/task supervisor), convert panic to a controlled failure signal and emit telemetry.

### Versioned persisted schemas and progress messages

- SQLite schema, cached scan records, and serialized progress payloads are versioned contracts: prefer additive changes, avoid breaking removals/renames.
- Unknown fields or columns must be tolerated where the format allows; missing required values must fail with typed corruption errors.
- Introduce schema/version markers for long-lived caches or exported snapshots.
- Deprecations require a compatibility window and explicit removal criteria.
- Mapping from adapter records / DB rows / serialized messages -> core types remains at boundaries via `TryFrom`.

---

## Before / After: Common Refactorings

### God object → launcher-owned scan coordinator

```rust
// BEFORE — UI object owns commands, config walking, DB writes, and model mutation
impl TerminusApp {
    fn refresh(&mut self) -> Result<()> {
        let packages = run_pacman_qi()?;
        let files = run_pacman_ql()?;
        let configs = discover_configs()?;
        save_snapshot(&self.db, &packages, &files, &configs)?;
        self.model = SystemModel::from_db(&self.db)?;
        Ok(())
    }
}

// AFTER — launcher orchestrates, adapter scans, core builds the model
let adapter = detect_platform();
let mut coordinator = ScanCoordinator::new(adapter, db_path, progress_tx);
let completed = coordinator.run_snapshot_scan()?;
persist_snapshot(&completed.scan)?;
swap_live_model(completed.model);
Ok(())
}
```

### Primitive obsession → newtypes

```rust
// BEFORE — path and manager_id semantics are implicit and easy to misuse
fn classify_config(manager_id: String, path: String) -> Result<()> { ... }

// AFTER — boundary parsing enforces semantics up front
fn classify_config(manager_id: ManagerId, path: CanonicalConfigPath) -> Result<(), CoreError> { ... }
```

### Scattered validation → parse-once boundary

```rust
// BEFORE — validates in adapter, again in core, again while building the model
fn parse_package(record: PacmanPackageRecord) {
    if record.name.trim().is_empty() { panic!("bad package name"); }
    save_package(record.name.clone());
}

// AFTER — parsed once, trusted everywhere
fn parse_package(record: PacmanPackageRecord) -> Result<Package, AdapterError> {
    let package = Package::try_from(record)?;
    save_package(package.name.clone());
    Ok(())
}
```

---

## Tradeoff Rules

Architecture guidance that doesn't say "when not to" is dogma. Apply judgment.

| If you're tempted to...                 | Ask first                                                              |
|-----------------------------------------|------------------------------------------------------------------------|
| Add typestate to a 2-state enum         | Is a simple enum with a `transition()` method enough?                  |
| Create a trait for one implementation   | Will there ever be a second impl, or do you just need a test double?   |
| Wrap every primitive in a newtype       | Does this primitive cross a module boundary? If it's local, don't wrap.|
| Make everything immutable               | Is this a hot loop mutating a buffer? Local `mut` is fine.             |
| Split a file into 5 modules             | Is the file >300 lines? If not, one file is clearer.                   |
| Add `From` impls for every conversion   | Is this conversion used more than once? If not, inline it.             |
| Reach for `Result` chains everywhere    | Is this a straight-line function that can't fail? Just return the value.|
| Ban `clone()` absolutely                | Is this a cold path with small data? A clone is fine. Comment why.     |
| Introduce CQRS command/query split      | Does this use case have different read and write models? If not, overkill.|
| Force immutable-return modeling in hot loops | Would local mutation or data-oriented layout improve cache locality and throughput? |

The goal is **working software with clear boundaries** — not architectural purity for its own sake.

---

## Code Review Checklist

Before approving any change:

- [ ] Any raw `String`, `i64`, or path string in cross-crate core APIs where the value has stronger semantics? → Wrap in a newtype.
- [ ] Any package-manager shellout or platform-specific parser living outside `terminus-platform-*`? → Move it to an adapter crate.
- [ ] Any validation after the boundary (re-checking a parsed type)? → Remove.
- [ ] Any `_ => {}` on `FileType`, `MatchMethod`, `InstallReason`, or similar core enums? → Match exhaustively.
- [ ] Any trait with a single implementor and no test double? → Remove the trait.
- [ ] Any `clone()` without a justifying comment? → Restructure or comment.
- [ ] Any `unwrap()` / `expect()` outside tests? → Use `?` or handle.
- [ ] Any `anyhow` / `eyre` in `terminus-core` or adapter parsing code? → Use typed error.
- [ ] Any `&mut self` that could be `&self` with a returned new value? → Prefer immutable.
- [ ] Any `terminus-core` module importing from `terminus-platform-*` or `terminus-gui`? → Invert the dependency.
- [ ] Any stringly-typed error (`Err("...".into())`)? → Use a proper variant.
- [ ] Any scan workflow vulnerable to interruption before DB/model swap? → Preserve the last good cache and live model until success.
- [ ] Does the GUI read package manager output, SQLite, or filesystem state directly? → Route through launcher/core instead.
- [ ] Tests cover happy path, at least one error path, boundary parsing, and ownership multiplicity? → Add missing.
- [ ] For refresh workflows, do tests include cancellation/interruption paths? → Add at least one cancellation safety test per critical workflow.

---

## Incremental Adoption

For existing codebases, adopt this architecture incrementally:

**Phase 1 — Low effort, high payoff:**
- Introduce newtypes for manager ids, canonical config paths, and other high-value boundary types.
- Replace `String`-based errors with typed enums in new core and adapter code.
- Add `TryFrom` parsing at adapter and DB boundaries.

**Phase 2 — Structural separation:**
- Extract matching, classification, and model-building logic out of launcher/UI code into `terminus-core`.
- Move package-manager commands and parsers into `terminus-platform-*` crates.
- Introduce fake adapters where test doubles would help.

**Phase 3 — Full architecture:**
- Enforce launcher-only wiring of platform crates.
- Persist new snapshots to temporary state and swap only after success.
- Keep `terminus-gui` render-only and dependent only on `terminus-core`.

Each phase is independently valuable. Stop wherever the cost/benefit ratio peaks for your project.

---

## Commands

```bash
cargo build                         # compile
cargo test                          # run all tests
cargo clippy -- -D warnings         # lint — treat warnings as errors
cargo fmt --check                   # format check
```

Run `cargo clippy` and `cargo test` before considering any task complete.
