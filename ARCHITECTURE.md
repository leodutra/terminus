# Terminus Architecture

## Scanning Model

The recommended scan architecture for Terminus is:

- synchronous snapshot scans on a dedicated worker thread
- streamed progress events during the scan
- atomic swap of the completed `SystemModel` only after the scan succeeds

This is intentionally not a design that streams partial packages, files, or configs directly into the live UI model.

## Why This Is The Right Fit

Terminus already has three strong constraints in the specification:

- the UI must never block
- only a completed scan replaces the previous saved state
- the GUI reads from `SystemModel`, not from the filesystem, database, or package manager directly

That makes partial-model streaming a poor fit. The package graph, config ownership, reverse dependencies, and orphan detection all become more complicated if the UI sees incomplete state. The app would need merge logic, partial-validity flags, and rollback behavior for failures. That is complexity without much payoff for MVP.

By contrast, progress-event streaming gives the user responsiveness and visibility while preserving a simple correctness model:

- old model stays live while scanning
- progress updates stream continuously
- new model swaps in once, at the end

## Recommended Pattern

### 1. Worker-thread scan

The launcher owns scan lifecycle.

- It selects the adapter.
- It spawns a scanner thread.
- It keeps the current `SystemModel` readable by the GUI.
- It polls progress and result channels from the main thread.

From the app's point of view, scanning is asynchronous.

From the scanner's point of view, scanning is a normal synchronous pipeline.

## 2. Progress events, not partial state streaming

Use `ScanProgress` as the public progress stream.

```rust
pub enum ScanProgress {
    Stage(String),
    PackagesFound(usize),
    FilesIndexed(usize),
    ConfigsMatched(usize),
    Complete,
    Error(String),
}
```

This is effectively an observer pattern implemented with messages.

The simplest MVP implementation is to keep `Sender<ScanProgress>` at the trait boundary and wrap it with a small helper such as `ProgressReporter` so lower-level code does not repeat channel-send boilerplate.

```rust
pub struct ProgressReporter {
    sender: Sender<ScanProgress>,
}

impl ProgressReporter {
    pub fn stage(&self, message: impl Into<String>) { /* send */ }
    pub fn packages_found(&self, count: usize) { /* send */ }
    pub fn files_indexed(&self, count: usize) { /* send */ }
    pub fn configs_matched(&self, count: usize) { /* send */ }
}
```

This gives us observer-style reporting without introducing a separate async runtime or complex stream abstraction.

## 3. Snapshot result handoff

The scan thread should build a full `ScanResult`, persist it, build or load the new `SystemModel`, and only then hand the completed result back to the launcher.

Recommended handoff shape:

- one progress channel for `ScanProgress`
- one result channel for `Result<SystemModel>` or `Result<ScanArtifacts>`

The GUI should never render from partially scanned data.

## 4. Atomic persistence and swap

Persistence and in-memory swap should follow the same rule:

- do not overwrite the last good database until the new scan succeeds
- do not replace the live `SystemModel` until the new scan succeeds

That keeps failure behavior simple and matches the product promise.

## Ownership Of Responsibilities

### Launcher binary

Owns:

- platform detection
- scan thread creation
- progress polling
- result polling
- freshness checks
- atomic swap of the shared model reference

The launcher should contain a `ScanCoordinator` or similarly named orchestration type.

### `terminus-core`

Owns:

- data model
- config discovery helpers
- config matching
- path classification
- SQLite persistence
- `SystemModel` construction and query surface

### Platform adapters

Own:

- platform detection
- package-manager interaction
- package metadata collection
- file ownership collection
- dependency clause collection
- package freshness checks
- progress emission during adapter work

Adapters should return a complete snapshot result. They should not mutate UI state.

### GUI

Owns:

- rendering current model state
- rendering progress state
- rendering refresh prompts
- reacting to completed model swaps

The GUI only reads shared state and polled progress. It never drives the scan directly.

## Recommended MVP Execution Pipeline

1. Launcher loads cached `SystemModel` if present.
2. Launcher spawns `ScanCoordinator` worker thread.
3. Worker emits `Stage(...)` progress.
4. Adapter gathers packages, files, dependencies, and emits incremental counts.
5. Core expands `ConfigSearchSpec`, discovers config paths, matches ownership, classifies files, and emits config progress.
6. Worker persists the snapshot to a new database state.
7. Worker builds the final `SystemModel`.
8. Worker sends the completed result back.
9. Launcher swaps the live model once.
10. GUI starts rendering the new model on the next frame.

## What We Should Not Do For MVP

- Do not stream partial `Package`, `FileNode`, or `ConfigMapping` records into the live UI state.
- Do not mutate the currently displayed `SystemModel` in place while a scan is still running.
- Do not introduce an async runtime just for scanning.
- Do not make the GUI depend on package-manager-specific code paths.

## Why Not Full Streaming?

Full data streaming sounds attractive, but it creates the wrong kind of complexity here:

- config ownership can change as more candidates arrive
- reverse dependencies are only trustworthy once dependency resolution finishes
- orphan detection is only final after matching completes
- partial DB writes complicate failure handling
- UI semantics become harder because rows may appear, move groups, or disappear mid-scan

For Terminus, the better split is:

- stream progress
- snapshot data

## Best Implementation Path

1. Keep the current `PlatformAdapter::scan(progress: Sender<ScanProgress>) -> Result<ScanResult>` shape for MVP.
2. Add a small `ProgressReporter` helper around `Sender<ScanProgress>`.
3. Add a launcher-owned `ScanCoordinator` that owns the worker thread and channels.
4. Keep the UI polling a progress receiver each frame.
5. Swap in the new `SystemModel` only after successful completion.

If we later need richer cancellation, multiple concurrent scans, or external integrations, we can evolve the progress transport. We should not pay that complexity cost before the snapshot-based MVP works.
