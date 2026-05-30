# Terminus Implementation Plan

Current state: the repository is still a single binary crate with a placeholder `src/main.rs`. This plan assumes the goal is to move from that skeleton to the architecture described in `SPECIFICATION.md`.

## Phase 0: Restructure Into the Target Workspace

- [ ] Convert the root `Cargo.toml` into a workspace plus launcher binary.
- [ ] Keep the root `src/main.rs` as the launcher entrypoint.
- [ ] Create `crates/terminus-core` as a library crate.
- [ ] Create `crates/terminus-gui` as a render-only library crate.
- [ ] Create `crates/terminus-platform-arch` and `crates/terminus-platform-ubuntu` as adapter crates.
- [ ] Wire feature flags in the launcher for `arch` and `ubuntu`.
- [ ] Ensure `terminus-gui` depends only on `terminus-core`.
- [ ] Add minimal crate scaffolding files so the workspace builds before any real logic lands.

Validation:

- [ ] `cargo check`
- [ ] `cargo test`

## Phase 1: Build `terminus-core`

### 1.1 Data Model and Public API

- [ ] Implement `model.rs` with `Package`, `PackageSource`, `FileNode`, `ConfigMapping`, `Dependency`, `ScanMetadata`, and related enums.
- [ ] Implement `platform.rs` with `ScanProgress`, `ScanResult`, `ConfigSearchSpec`, and `PlatformAdapter`.
- [ ] Implement `system_model.rs` with indexes and query helpers.
- [ ] Define `PackageDetails` and `SearchResults` return types used by the query API.

### 1.2 SQLite Persistence

- [ ] Implement `db.rs` with schema creation and full-scan replacement semantics.
- [ ] Persist packages, files, dependencies, config mappings, orphaned configs, and scan metadata.
- [ ] Load a complete `SystemModel` from the database.
- [ ] Handle unreadable or corrupt state by rebuilding from a fresh scan.

### 1.3 Matching and Classification

- [ ] Implement `aliases.rs` with the hardcoded alias table.
- [ ] Implement `classify.rs` with longest-prefix classification and Linux heuristics.
- [ ] Implement config discovery that expands `ConfigSearchSpec`, walks allowed directories, includes known files, and collects metadata for matching.
- [ ] Implement `matcher.rs` with package-files and name-match scoring.
- [ ] Support multi-owner config mappings with `is_primary` semantics.
- [ ] Emit orphaned configs when no candidate reaches the threshold.

Validation:

- [ ] `cargo test -p terminus-core`
- [ ] Add unit tests for config matching tie behavior.
- [ ] Add unit tests for `ConfigSearchSpec` expansion and config candidate collection.
- [ ] Add unit tests for path classification edge cases.
- [ ] Add unit tests for DB round-tripping `SystemModel`.

## Phase 2: Implement the Arch Adapter

- [ ] Add `ArchAdapter` construction and detection.
- [ ] Parse `pacman -Qi` package metadata.
- [ ] Parse `pacman -Ql` file ownership.
- [ ] Parse dependency clauses from pacman metadata and resolve installed package ids when possible.
- [ ] Use `pacman -Qm` to tag foreign packages via `source.channel`.
- [ ] Provide the default Linux `ConfigSearchSpec`.
- [ ] Emit `PackagesFound`, `FilesIndexed`, and `ConfigsMatched` progress updates during the scan.
- [ ] Implement package freshness with `/var/log/pacman.log`.
- [ ] Return a complete `ScanResult` or a hard failure without partial state replacement.

Validation:

- [ ] `cargo test -p terminus-platform-arch`
- [ ] Add parser fixture tests for `pacman -Qi`, `pacman -Ql`, and `pacman -Qm`.
- [ ] Run one end-to-end scan on an Arch machine.

## Phase 3: Implement the Ubuntu/Debian Adapter

- [ ] Add `UbuntuAdapter` construction and detection.
- [ ] Parse `dpkg-query -W -f='...'` package output.
- [ ] Read `/var/lib/dpkg/info/*.list` for file ownership.
- [ ] Resolve explicit vs dependency installs via `apt-mark showmanual`.
- [ ] Parse dependency clauses from dpkg metadata and resolve installed package ids when possible.
- [ ] Approximate install dates from `.list` file mtimes.
- [ ] Provide the default Linux `ConfigSearchSpec`.
- [ ] Emit `PackagesFound`, `FilesIndexed`, and `ConfigsMatched` progress updates during the scan.
- [ ] Implement package freshness with `/var/log/dpkg.log`.
- [ ] Continue scanning when individual `.list` files are unreadable.

Validation:

- [ ] `cargo test -p terminus-platform-ubuntu`
- [ ] Add parser fixture tests for dpkg output and `.list` readers.
- [ ] Run one end-to-end scan on an Ubuntu or Debian machine.

## Phase 4: Launcher, Scan Orchestration, and Caching

- [ ] Implement launcher-side platform selection.
- [ ] Load cached state on startup before refreshing in the background.
- [ ] Run scans on a background thread.
- [ ] Wire `ScanProgress` delivery from the scanner thread into launcher and loading-state updates.
- [ ] Swap completed `SystemModel` instances into shared app state.
- [ ] Implement package freshness checks through adapters.
- [ ] Implement shallow config freshness checks using `ConfigSearchSpec` roots and known files.
- [ ] Show separate package-refresh and config-refresh prompts.
- [ ] Preserve the last good DB state until a new scan completes successfully.

Validation:

- [ ] `cargo check`
- [ ] Add a launcher test for platform selection fallback paths where practical.
- [ ] Add integration tests for `ScanProgress` flow, completion-only model swaps, and package/config freshness state transitions.
- [ ] Manual startup check with cached state present.
- [ ] Manual startup check with no cached state present.
- [ ] Manual loading-screen check with live progress updates and no UI blocking.

## Phase 5: Build the GUI Shell

- [ ] Add the `terminus-gui` library entrypoint and app state.
- [ ] Implement theme and base layout.
- [ ] Add the loading screen with progress updates.
- [ ] Add left navigation, main content area, detail panel, and status bar scaffolding.
- [ ] Keep all data access flowing through `SystemModel` only.

Validation:

- [ ] `cargo check -p terminus-gui`
- [ ] Manual launch of the empty-state and loading-state UI.

## Phase 6: Packages View and Detail Panel

- [ ] Implement the searchable, sortable packages table.
- [ ] Add package detail rendering for binaries, configs, dependencies, reverse dependencies, and files.
- [ ] Display resolved dependency package names when available, otherwise raw clauses.
- [ ] Format source/channel values for display without leaking adapter-specific implementation details into layout code.

Validation:

- [ ] Manual UI check with real scan data.
- [ ] Add focused tests for package sorting and search behavior where feasible.

## Phase 7: Configs View

- [ ] Keep config paths as the canonical records and derive app-grouped sections from primary owners.
- [ ] Group single-owner paths under their related app while preserving per-path ownership details.
- [ ] Place multi-owner paths in a dedicated shared section instead of duplicating the same path under multiple apps.
- [ ] Add filters for matched, shared, and orphaned configs.
- [ ] Display shared or ambiguous ownership with a warning badge.
- [ ] Surface all primary owners in row details or tooltip content.
- [ ] Show confidence markers and hover explanations.

Validation:

- [ ] Manual UI check for single-owner, shared-owner, and orphan cases.
- [ ] Add focused tests for app grouping, shared-owner presentation, and filter behavior where feasible.

## Phase 8: Disk View, Global Search, and Polish

- [ ] Implement the treemap, or fall back to a horizontal bar chart if treemap complexity is not paying off.
- [ ] Add disk-view hover details, click-to-inspect behavior, and the summary bar totals.
- [ ] Add the explicit-only vs all-packages toggle.
- [ ] Bind global search to `Ctrl+K` and `/`.
- [ ] Implement case-insensitive substring global search across package names, descriptions, file paths, and config paths.
- [ ] Group global search results by type and make result selection navigate to the correct view and open the relevant detail panel.
- [ ] Add keyboard navigation and confidence tooltips.
- [ ] Finalize the status bar wording for freshness states.

Validation:

- [ ] Manual UI walkthrough across all three views.
- [ ] Check responsiveness with a realistically sized scan.

## Phase 9: Packaging and Documentation

- [ ] Add `assets/terminus.desktop`.
- [ ] Add `assets/terminus.svg`.
- [ ] Update `README.md` with build, run, platform support, and screenshots.
- [ ] Document refresh behavior and readonly guarantees.
- [ ] Produce an Arch AUR package definition.
- [ ] Produce an Ubuntu distribution artifact via `cargo-deb` or AppImage.
- [ ] Verify release build output.

Validation:

- [ ] `cargo build --release`
- [ ] Smoke-test the release binary on at least one Arch system and one Ubuntu or Debian system.

## Cross-Cutting Work

- [ ] Keep fixture-based parser tests for every external command format.
- [ ] Prefer deterministic unit tests in `terminus-core` over UI-coupled behavior tests.
- [ ] Add small manual test datasets or fixtures for shared config ownership cases.
- [ ] Revisit performance once end-to-end scans work correctly.
- [ ] Revisit config freshness after MVP if shallow checks prove too noisy.

## Suggested Execution Order

1. Finish Phase 0 completely.
2. Get `terminus-core` compiling with tests before writing adapter code.
3. Finish Arch end-to-end first as the reference implementation.
4. Port the abstraction to Ubuntu/Debian and correct model assumptions that break.
5. Only then build the GUI against a stable `SystemModel` API.
