# Terminus — MVP Specification

**Version:** 1.0.0-draft
**Name:** Terminus — the Roman god of boundaries, markers, and limits
**What it is:** A readonly desktop application that shows you everything on your system
**What it is not:** A package manager. It never installs, removes, or modifies anything.
**MVP platforms:** Arch Linux (pacman) + Ubuntu/Debian (apt/dpkg)
**Ship target:** 4–5 weeks solo development

---

## 1. What Terminus Is

Terminus is a desktop application that answers one question:

> "What's on my system, where does it live, and how is it all connected?"

You open it. You see your system. You understand it.

Terminus never modifies your system. No installs. No uninstalls. No config edits. No file deletions. It is a pure observer — a read-only lens into your machine's current state.

This is a permanent design constraint, not a temporary MVP limitation. Package managers already exist and they're good at what they do. Terminus exists because understanding your system is a different problem than managing it, and no tool solves the understanding problem well.

---

## 2. The Core Insight

Linux has excellent tools for doing things: `pacman`, `apt`, `systemctl`, `rm`.

Linux has no tool for *seeing* things — not as isolated facts (`pacman -Qi neovim`), but as a connected whole:

- This binary belongs to this package
- This package owns these configs in three different locations
- This config folder belongs to nothing — it's an orphan from something you removed
- These 200MB of dependencies exist only because of this one package you forgot you installed

Terminus builds that connected picture and shows it to you. What you do with that knowledge is your business.

---

## 3. Architectural Principle

Hard separation between what you see and where the data comes from.

```
┌──────────────────────────────────────────────────────┐
│                    terminus-gui                      │
│            UI rendering, interaction                 │
│         Knows nothing about pacman, apt,             │
│         or any specific data source                  │
│                                                      │
│         Depends only on: terminus-core               │
├──────────────────────────────────────────────────────┤
│                   terminus-core                      │
│          Platform-agnostic data model                │
│      SystemModel, Package, FileNode, Config          │
│      Query operations, storage, traits               │
│                                                      │
│         Depends on: nothing platform-specific        │
├──────────────┬──────────────┬────────────────────────┤
│ terminus-    │ terminus-    │ (future adapters)      │
│ platform-    │ platform-    │                        │
│ arch         │ ubuntu       │ fedora, windows, ...   │
│              │              │                        │
│ pacman       │ apt/dpkg     │                        │
└──────────────┴──────────────┴────────────────────────┘
```

The GUI crate never imports a platform crate. The core never shells out to a command. The launcher binary owns platform detection and wires a chosen adapter into the UI. Adding a new platform means adding one adapter crate and one launcher registration — zero changes to `terminus-gui` or `terminus-core`.

Two adapters ship on day one. This forces the abstraction to be real, not theoretical.

---

## 4. Core Data Model

### 4.1 Entities

```rust
// ── terminus-core/src/model.rs ──

pub struct Package {
    pub id: i64,
    pub name: String,
    pub version: String,
    pub source: PackageSource,
    pub size_bytes: Option<u64>,
    pub install_date: Option<String>,
    pub install_reason: InstallReason,
    pub description: Option<String>,
}

pub struct PackageSource {
    pub manager_id: String,      // pacman, apt, dnf, flatpak, winget, ...
    pub channel: Option<String>, // official, foreign, aur, ppa, store, ...
}

pub enum InstallReason {
    Explicit,
    Dependency,
    Unknown,
}

pub struct FileNode {
    pub id: i64,
    pub path: String,
    pub package_id: Option<i64>,
    pub file_type: FileType,
}

pub enum FileType {
    Binary,
    Config,
    Library,
    Data,
    Cache,
    Other,
}

pub struct ConfigMapping {
    pub id: i64,
    pub config_path: String,
    pub package_id: i64,
    pub confidence: f64,
    pub method: MatchMethod,
    pub is_primary: bool,
}

pub enum MatchMethod {
    PackageFiles,
    NameMatch,
    KnownAlias,
    DesktopFile,    // post-MVP
}

pub struct OrphanedConfig {
    pub path: String,
    pub size_bytes: Option<u64>,
    pub last_modified: Option<String>,
}

pub struct Dependency {
    pub from_package_id: i64,
    pub raw_clause: String,
    pub resolved_package_id: Option<i64>,
}

pub struct ScanMetadata {
    pub scan_date: String,
    pub platform: String,
    pub package_count: usize,
    pub file_count: usize,
    pub config_count: usize,
    pub orphaned_count: usize,
    pub total_size_bytes: u64,
    pub duration_ms: u64,
}
```

### 4.2 SystemModel (in-memory query layer)

```rust
// ── terminus-core/src/system_model.rs ──

pub struct SystemModel {
    pub packages: Vec<Package>,
    pub packages_by_name: HashMap<String, Vec<usize>>,
    pub packages_by_id: HashMap<i64, usize>,

    pub files_by_package: HashMap<i64, Vec<FileNode>>,
    pub package_by_file: HashMap<String, i64>,

    pub configs: Vec<ConfigMapping>,
    pub configs_by_path: HashMap<String, Vec<ConfigMapping>>,
    pub configs_by_package: HashMap<i64, Vec<ConfigMapping>>,
    pub orphaned_configs: Vec<OrphanedConfig>,

    pub dependencies: HashMap<i64, Vec<Dependency>>,
    pub reverse_deps: HashMap<i64, Vec<i64>>,

    pub meta: ScanMetadata,
}

impl SystemModel {
    pub fn from_db(db: &Database) -> Result<Self>;
    pub fn owner_of(&self, path: &str) -> Option<&Package>;
    pub fn package_details(&self, package_id: i64) -> PackageDetails;
    pub fn packages_by_size(&self) -> Vec<&Package>;
    pub fn search(&self, query: &str) -> SearchResults;
    pub fn orphans(&self) -> &[OrphanedConfig];
}
```

The GUI only talks to SystemModel. Never to SQLite. Never to the filesystem. Never to a package manager.

Package identity inside core is always `id`. Names are for display and search, not relationship keys. Package source is descriptive adapter metadata, not a core enum that must grow for every new platform. Dependency edges store the raw package-manager clause plus an optional resolved package id when the clause maps to an installed package.

Config ownership is intentionally one-to-many. `ConfigMapping` stores every candidate above the orphan threshold. `is_primary` marks the current best owner set. If more than one mapping for a path is primary, the path is shared or ambiguous and the UI should surface that explicitly rather than collapsing it to one silent winner.

Configs are collected and de-duplicated canonically by path. App-level groupings in the UI are derived from those path-keyed mappings, not stored as a separate source of truth.

### 4.3 SQLite Schema

```sql
CREATE TABLE packages (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    version         TEXT NOT NULL,
    manager_id      TEXT NOT NULL,
    source_channel  TEXT NOT NULL DEFAULT '',
    size_bytes      INTEGER,
    install_date    TEXT,
    install_reason  TEXT,
    description     TEXT,
    UNIQUE(name, manager_id, source_channel)
);

CREATE TABLE files (
    id          INTEGER PRIMARY KEY,
    path        TEXT NOT NULL UNIQUE,
    package_id  INTEGER REFERENCES packages(id),
    file_type   TEXT NOT NULL
);

CREATE TABLE dependencies (
    from_pkg_id      INTEGER NOT NULL REFERENCES packages(id),
    raw_clause       TEXT NOT NULL,
    resolved_pkg_id  INTEGER REFERENCES packages(id),
    PRIMARY KEY (from_pkg_id, raw_clause)
);

CREATE TABLE config_mappings (
    id          INTEGER PRIMARY KEY,
    config_path TEXT NOT NULL,
    package_id  INTEGER NOT NULL REFERENCES packages(id),
    confidence  REAL NOT NULL,
    method      TEXT NOT NULL,
    is_primary  INTEGER NOT NULL DEFAULT 0,
    UNIQUE(config_path, package_id)
);

CREATE TABLE orphaned_configs (
    id              INTEGER PRIMARY KEY,
    path            TEXT NOT NULL UNIQUE,
    size_bytes      INTEGER,
    last_modified   TEXT
);

CREATE TABLE scan_metadata (
    id              INTEGER PRIMARY KEY,
    scan_date       TEXT NOT NULL,
    platform        TEXT NOT NULL,
    package_count   INTEGER,
    file_count      INTEGER,
    config_count    INTEGER,
    orphaned_count  INTEGER,
    total_size_bytes INTEGER,
    duration_ms     INTEGER
);

CREATE INDEX idx_files_package ON files(package_id);
CREATE INDEX idx_files_path ON files(path);
CREATE INDEX idx_config_path ON config_mappings(config_path);
CREATE INDEX idx_config_package ON config_mappings(package_id);
CREATE INDEX idx_deps_from ON dependencies(from_pkg_id);
CREATE INDEX idx_deps_resolved ON dependencies(resolved_pkg_id);
CREATE INDEX idx_packages_manager ON packages(manager_id);
```

MVP persistence rule: only a completed scan replaces the previous saved state. If the database is unreadable or corrupt, rebuild it from a fresh scan instead of trying to use partial data.

---

## 5. Platform Adapter Trait

```rust
// ── terminus-core/src/platform.rs ──

pub enum ScanProgress {
    Stage(String),
    PackagesFound(usize),
    FilesIndexed(usize),
    ConfigsMatched(usize),
    Complete,
    Error(String),
}

pub struct ScanResult {
    pub packages: Vec<Package>,
    pub files: Vec<FileNode>,
    pub config_mappings: Vec<ConfigMapping>,
    pub orphaned_configs: Vec<OrphanedConfig>,
    pub dependencies: Vec<Dependency>,
    pub meta: ScanMetadata,
}

pub struct ConfigSearchSpec {
    pub directories: Vec<String>,
    pub known_directories: Vec<String>,
    pub known_files: Vec<String>,
}

pub trait PlatformAdapter: Send + Sync {
    fn platform_name(&self) -> &str;
    fn platform_id(&self) -> &str;
    fn scan(&self, progress: Sender<ScanProgress>) -> Result<ScanResult>;
    fn detect() -> bool where Self: Sized;
    fn config_search_spec(&self) -> ConfigSearchSpec;
    fn has_package_state_changes_since(&self, scan_date: &str) -> Result<bool>;
}
```

Implementation note: `scan()` returns a complete result or an error. It should not partially overwrite the last good scan. Package freshness and config freshness are separate concerns. `has_package_state_changes_since()` is based on package-manager state only. Config freshness is checked outside adapters because config scanning lives in `terminus-core` and the launcher.

### Shared Linux Config Search Spec

For MVP, the Linux adapters start from the same default config search scope:

```rust
fn default_linux_config_search_spec() -> ConfigSearchSpec {
    ConfigSearchSpec {
        directories: vec!["/etc", "/etc/xdg", "~/.config"],
        known_directories: vec!["~/.ssh", "~/.gnupg", "~/.vim", "~/.emacs.d"],
        known_files: vec![
            "~/.gitconfig",
            "~/.zshrc",
            "~/.bashrc",
            "~/.profile",
            "~/.bash_profile",
            "~/.vimrc",
            "~/.tmux.conf",
        ],
    }
}
```

Adapters may extend this later when a platform needs additional known config roots.

---

## 6. Platform Adapter: Arch Linux

### Detection

```rust
fn detect() -> bool {
    Path::new("/usr/bin/pacman").exists()
}
```

### Data Sources

| Data             | Command               | Notes                                                           |
|------------------|-----------------------|-----------------------------------------------------------------|
| All packages     | `pacman -Qi`          | Single call, key-value parser                                   |
| File ownership   | `pacman -Ql`          | Single call, line splitting                                     |
| Dependencies     | `pacman -Qi`          | Parse `Depends On`, resolve to installed packages when possible |
| Source detail    | `pacman -Qm`          | Marks channel `foreign`; future enrichers may tag `aur` when detectable |
| Staleness        | `/var/log/pacman.log` | Compare last entry timestamp                                    |

### Config Search Spec

Uses `default_linux_config_search_spec()`.

---

## 7. Platform Adapter: Ubuntu/Debian

### Detection

```rust
fn detect() -> bool {
    Path::new("/usr/bin/dpkg").exists() && Path::new("/usr/bin/apt").exists()
}
```

### Data Sources

| Data             | Source                                   | Notes                                                                        |
|------------------|------------------------------------------|------------------------------------------------------------------------------|
| All packages     | `dpkg-query -W -f='...'`                 | Custom format string                                                         |
| File ownership   | `/var/lib/dpkg/info/*.list`              | Read files directly, not per-pkg                                             |
| Dependencies     | `dpkg-query -W -f='...'`                 | Parse `Depends` / `Pre-Depends`, resolve to installed packages when possible |
| Install dates    | `.list` file mtimes                      | Approximate                                                                  |
| Install reason   | `apt-mark showmanual`                    | Explicit vs dependency                                                       |
| Staleness        | `/var/log/dpkg.log`                      | Last entry timestamp                                                         |

### Config Search Spec

Uses `default_linux_config_search_spec()`.

### Key Difference

dpkg doesn't store install dates. Approximate from `.list` file modification times. Display as "~Mar 2025" to signal uncertainty. This is honest — showing nothing or guessing silently are both worse.

If a `.list` file is unreadable, keep scanning and treat the affected package data as incomplete rather than failing the whole scan.

---

## 8. Config Matching Engine

Lives in `terminus-core`. Platform-agnostic. Takes package names, file lists, and config search spec as input.

### Method 1: package-files (confidence 0.80–0.95)

Config file is in a package's file list.
- Exact file match: 0.95
- File in a subdirectory owned by package: 0.85

### Method 2: name-match (confidence 0.55–0.92)

Config path component matches a package name.
- Exact match: 0.92
- Known alias (nvim → neovim): 0.85
- Substring: 0.70
- Fuzzy: 0.55

### Alias Table

```rust
pub static ALIASES: &[(&str, &str)] = &[
    ("nvim", "neovim"),
    ("vim", "vim"),
    ("py", "python3"),
    ("python", "python3"),
    ("node", "nodejs"),
    ("gpg", "gnupg"),
    ("ssh", "openssh"),
];
```

Starts hardcoded. Migrates to a config file post-MVP.

MVP scope rule: the search spec is a conservative allowlist, not a recursive `~/.*` sweep. Avoid `~/.local/share` and arbitrary home dot-directories by default; they contain too much state and cache noise to present as trustworthy config ownership.

### Orphan Rule

If no candidate for a config path reaches 0.45 confidence, treat it as orphaned.

Persist every candidate at or above 0.45 confidence.

If one candidate clearly wins, mark it primary.

If multiple candidates tie for the top score, or land within a narrow confidence window of the best score, keep the multiplicity and mark all of those candidates primary. Prefer the higher-confidence method first (`package-files` over `known-alias` over `name-match`), then package name for stable ordering, but do not discard effectively-equal owners just to force a single result.

The UI should highlight shared or ambiguous ownership with an explicit warning marker, not hide it behind a single package label.

---

## 9. File Type Classification

```rust
pub struct PathClassifier {
    prefix_rules: Vec<(String, FileType)>,
}

impl PathClassifier {
    pub fn linux() -> Self {
        Self { prefix_rules: vec![
            ("/usr/bin/",      FileType::Binary),
            ("/usr/sbin/",     FileType::Binary),
            ("/bin/",          FileType::Binary),
            ("/sbin/",         FileType::Binary),
            ("/usr/libexec/",  FileType::Binary),
            ("/etc/",          FileType::Config),
            ("/etc/xdg/",      FileType::Config),
            ("/usr/lib/",      FileType::Library),
            ("/usr/lib64/",    FileType::Library),
            ("/lib/",          FileType::Library),
            ("/lib64/",        FileType::Library),
            ("/usr/share/",    FileType::Data),
            ("/var/lib/",      FileType::Data),
            ("/opt/",          FileType::Data),
            ("/var/cache/",    FileType::Cache),
        ]}
    }
}
```

Classification rule: use the longest matching prefix first. Home config paths from `ConfigSearchSpec` always classify as `Config`. Add small runtime heuristics for `/opt/*/bin/`, `/opt/*/lib/`, and `.so` / `.so.*` shared objects before falling back to `Other`. Favor predictable broad categories over speculative fine-grained guesses.

---

## 10. Crate Structure

```
terminus/
├── Cargo.toml                          (workspace + launcher binary)
├── src/
│   └── main.rs                        # Launcher: platform detection, cache load, eframe start
├── crates/
│   ├── terminus-core/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── model.rs               # Entities
│   │   │   ├── system_model.rs         # In-memory query layer
│   │   │   ├── db.rs                   # SQLite persistence
│   │   │   ├── platform.rs            # PlatformAdapter trait
│   │   │   ├── matcher.rs             # Config-to-package matching
│   │   │   ├── classify.rs            # File type classification
│   │   │   └── aliases.rs             # Known name aliases
│   │   └── Cargo.toml
│   │
│   ├── terminus-platform-arch/
│   │   ├── src/
│   │   │   ├── lib.rs                 # ArchAdapter impl
│   │   │   ├── pacman.rs              # pacman output parser
│   │   │   └── detect.rs
│   │   └── Cargo.toml
│   │
│   ├── terminus-platform-ubuntu/
│   │   ├── src/
│   │   │   ├── lib.rs                 # UbuntuAdapter impl
│   │   │   ├── dpkg.rs                # dpkg parser + .list reader
│   │   │   └── detect.rs
│   │   └── Cargo.toml
│   │
│   └── terminus-gui/
│       ├── src/
│       │   ├── lib.rs                 # Render-only app shell
│       │   ├── app.rs                 # App state, tab routing
│       │   ├── views/
│       │   │   ├── mod.rs
│       │   │   ├── packages.rs
│       │   │   ├── configs.rs
│       │   │   ├── disk.rs
│       │   │   └── loading.rs
│       │   ├── widgets/
│       │   │   ├── mod.rs
│       │   │   ├── detail_panel.rs
│       │   │   ├── search.rs
│       │   │   ├── confidence.rs
│       │   │   └── treemap.rs
│       │   └── theme.rs
│       └── Cargo.toml
│
├── assets/
│   ├── terminus.desktop
│   └── terminus.svg
└── README.md
```

### Dependency Enforcement

```toml
# Cargo.toml (workspace root / launcher binary)
[features]
default = ["arch", "ubuntu"]
arch = ["dep:terminus-platform-arch"]
ubuntu = ["dep:terminus-platform-ubuntu"]

[dependencies]
terminus-core = { path = "crates/terminus-core" }
terminus-gui = { path = "crates/terminus-gui" }
terminus-platform-arch = { path = "crates/terminus-platform-arch", optional = true }
terminus-platform-ubuntu = { path = "crates/terminus-platform-ubuntu", optional = true }
```

```toml
# crates/terminus-gui/Cargo.toml
[dependencies]
terminus-core = { path = "../terminus-core" }
eframe = "0.29"
egui = "0.29"
egui_extras = "0.29"
```

Platform crates are linked only by the launcher binary. The GUI crate remains render-only and depends only on `terminus-core`.

### Runtime Detection

```rust
// ── terminus/src/main.rs ──

fn detect_platform() -> Box<dyn PlatformAdapter> {
    #[cfg(feature = "arch")]
    if terminus_platform_arch::ArchAdapter::detect() {
        return Box::new(terminus_platform_arch::ArchAdapter::new());
    }

    #[cfg(feature = "ubuntu")]
    if terminus_platform_ubuntu::UbuntuAdapter::detect() {
        return Box::new(terminus_platform_ubuntu::UbuntuAdapter::new());
    }

    panic!("No supported platform detected");
}
```

The launcher detects the platform, loads or refreshes the `SystemModel`, and then starts the GUI. `terminus-gui` receives model data and user events only.

---

## 11. Key Dependencies

```toml
# terminus-core
rusqlite = { version = "0.31", features = ["bundled"] }
serde = { version = "1", features = ["derive"] }
dirs = "5"
crossbeam-channel = "0.5"
walkdir = "2"
chrono = "0.4"

# terminus-gui
eframe = "0.29"
egui = "0.29"
egui_extras = "0.29"
```

---

## 12. Interface

### 12.1 Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  Terminus                                [search] [⟳] [⚙]       │
├────────────┬─────────────────────────────────────────────────────┤
│            │                                                     │
│ ◉ Packages │  [Main content area]                                │
│            │                                                     │
│ ○ Configs  │                                                     │
│            │                                                     │
│ ○ Disk     │                                                     │
│            │                                                     │
│            ├─────────────────────────────────────────────────────┤
│            │  [Detail panel — contextual, collapsible]           │
│            │                                                     │
├────────────┴─────────────────────────────────────────────────────┤
│  Arch Linux · 1,847 packages · 12.4 GB · Scanned 2 hours ago     │
└──────────────────────────────────────────────────────────────────┘
```

### 12.2 Packages View

Searchable, sortable table:

| Name       | Version | Source  | Size     | Installed     | Reason     |
|------------|---------|---------|----------|---------------|------------|
| neovim     | 0.10.4  | pacman  | 48.2 MB  | Jan 15, 2025  | explicit   |
| firefox    | 128.0   | apt     | 234.1 MB | ~Mar 2025     | explicit   |
| libssl3    | 3.0.13  | apt     | 5.8 MB   | ~Nov 2024     | dependency |

- Type to filter (no separate search box — the table is the search)
- Click column to sort
- Click row → detail panel

### Detail Panel

```
neovim 0.10.4
Installed Jan 15, 2025 · pacman
48.2 MB across 127 files

── Binaries ────────────────────
/usr/bin/nvim

── Configs ─────────────────────
~/.config/nvim/          ●●●●○  name
/etc/xdg/nvim/           ●●●○○  files

── Dependencies (12) ───────────
luajit · libuv · tree-sitter · ...

── Required By (2) ─────────────
neovim-qt · python-pynvim

── Files (127) ─────────────────
[collapsible tree]
```

Dependencies display the resolved installed package name when available; otherwise they fall back to the raw package-manager clause.

### 12.3 Configs View

Collected by path, presented grouped under the related app. Orphans remain prominent at the bottom.

Single-owner paths appear under their primary app section. Shared or ambiguous paths stay path-keyed internally, but are surfaced in a dedicated Shared / Ambiguous section so the same config path is not duplicated under multiple apps.

```
── neovim ──────────────────────────────────
   ~/.config/nvim/init.lua          ●●●●○
   /etc/xdg/nvim/sysinit.vim        ●●●○○

── git ─────────────────────────────────────
   ~/.gitconfig                      ●●●●●

── Shared / Ambiguous ─────────────────────
    ~/.config/foo/                    !  foo-cli · foo-gui

── Unmatched ───────────────────────────────
   ~/.config/polybar/                  ?
   ~/.config/rofi/                     ?
   ~/.config/dunst/                    ?
```

Filters: All / Matched / Shared / Orphaned

If a config path has multiple primary owners, show a warning badge and surface all primary candidates in the row details or tooltip.

### 12.4 Disk View

Treemap — packages as rectangles, sized by installed footprint. Hover for details, click to inspect.

Summary bar: "1,847 packages · 12.4 GB total · 423 explicitly installed"

Toggle: explicit only / all (including deps)

Fallback: if treemap proves too complex in egui, ship a horizontal bar chart — same data, less visual impact, but shippable.

### 12.5 Loading Screen (first launch)

```
        Terminus

Detected: Arch Linux (pacman)

████████████░░░░░░░░  58%
Reading package files...
1,247 / 1,847 packages
```

### 12.6 Global Search

Ctrl+K or `/`. Searches across package names, descriptions, file paths, config paths. Results grouped by type. Select a result → navigate to appropriate view + open detail panel.

For MVP, substring search is enough. It should be case-insensitive.

### 12.7 Status Bar

```
Arch Linux · 1,847 packages · 12.4 GB · Scanned 2 hours ago
```

If package state has changed since last scan: "System packages changed. Refresh?"

If config files may have changed since last scan: "Config files may have changed. Refresh?"

---

## 13. Confidence Display

| Confidence | Dots  | Label            | Color  |
|------------|-------|------------------|--------|
| 0.90–1.00  | ●●●●● | Confirmed        | Green  |
| 0.75–0.89  | ●●●●○ | Very likely      | Green  |
| 0.60–0.74  | ●●●○○ | Likely           | Amber  |
| 0.45–0.59  | ●●○○○ | Uncertain        | Amber  |
| 0.00–0.44  | ●○○○○ | Guess            | Grey   |
| No match   |   ?   | Unmatched        | Red    |

Hover → "Matched by name similarity (confidence: 0.82)"

Beginners see dots. Power users hover for numbers. Both get honest information.

---

## 14. Threading

```
Main Thread (egui)          Scanner Thread
──────────────────          ──────────────
reads SystemModel  ←─swap── builds SystemModel
polls progress     ←─chan── sends ScanProgress
renders every frame         writes to SQLite
never blocks                never touches UI
```

Communication: `Arc<RwLock<SystemModel>>` for the data swap, `crossbeam::channel` for progress updates.

Only a completed scan swaps into `SystemModel` and replaces the saved database.

---

## 15. First Launch Flow

1. Open Terminus
2. Check for `~/.local/share/terminus/state.db`
3. If it exists, load `SystemModel` and show the main UI immediately
4. In the background, check package freshness via `has_package_state_changes_since()`
5. In the background, run a shallow config freshness pass over `ConfigSearchSpec` directories and known files
6. If package state changed → show "System packages changed. Refresh?"
7. If package state is unchanged but config roots changed or freshness is uncertain → show "Config files may have changed. Refresh?"
8. If no state exists → show loading screen, scan on background thread, transition to main UI on completion

MVP freshness is intentionally split:

- Package freshness is authoritative and adapter-owned.
- Config freshness is best-effort and launcher-owned.

Future improvement: replace the shallow config freshness pass with persisted per-root fingerprints or filesystem notifications so config-only changes can be detected cheaply and accurately.

Subsequent launches: under 1 second to interactive.

---

## 16. Visual Design

- Dark theme default, light theme available
- Custom `egui::Visuals` — not stock egui appearance
- Monospace for paths, proportional for labels
- Treemap: custom-painted via `egui::Painter`
- Minimum window: 900×600
- Detail panel: ~35% width, collapsible
- No decorative chrome. Data-dense. Every pixel earns its place.

---

## 17. Distribution

Single binary with both adapters compiled in. Runtime detection picks the right one.

```bash
cargo build --release
# → target/release/terminus
```

- Arch: AUR package
- Ubuntu: `.deb` via cargo-deb or AppImage
- Future platforms: add a crate, recompile

---

## 18. What Success Looks Like

1. Opens on Arch and Ubuntu, auto-detects, scans, shows identical UI with platform-specific data
2. User finds orphaned configs from apps uninstalled months ago
3. User discovers a 200MB package they forgot they installed
4. User clicks a package and sees configs they didn't know existed in locations they didn't know to look
5. User understands their system better after 5 minutes with Terminus than after years of manual exploration

---

## 19. Development Sequence

### Week 1: Core + Arch Adapter
- Day 1: Core model, SQLite schema, PlatformAdapter trait
- Day 2–3: Arch adapter (pacman parser, file listing, detection)
- Day 4: Config matcher + file classifier
- Day 5: Full scan integration test on Arch

### Week 2: Ubuntu Adapter + Validation
- Day 6–7: Ubuntu adapter (dpkg parser, .list file reader)
- Day 8: Test on Ubuntu VM — validate trait boundary
- Day 9: Fix model assumptions that broke across distros
- Day 10: SystemModel query layer (search, sort, filter)

### Week 3: GUI
- Day 11–12: eframe skeleton, theme, tabs, loading screen
- Day 13: Packages view (table + detail panel)
- Day 14: Configs view (grouped + orphans)
- Day 15: Disk view (treemap or bar chart — decide day 15)

### Week 4: Integration + Polish
- Day 16: Global search, status bar, staleness detection
- Day 17: Threading (background scan, progress, RwLock swap)
- Day 18: Confidence widget, tooltips, keyboard nav
- Day 19: Cross-platform testing, edge cases
- Day 20: .desktop file, README, screenshots, packaging

---

## 20. Post-MVP Roadmap

| Version | Feature                              |
|---------|--------------------------------------|
| v0.3    | `.desktop` file parsing              |
| v0.3    | Dependency graph visualization       |
| v0.4    | Flatpak adapter                      |
| v0.4    | Snap adapter                         |
| v0.5    | Fedora (dnf) adapter                 |
| v0.5    | Snapshot + diff ("what changed?")    |
| v0.6    | Export system report (JSON/HTML)     |
| v0.7    | Windows adapter                      |
| v1.0    | System blueprint (full state export) |

Every version is readonly. Terminus observes. It never acts.

---

## 21. Architectural Invariants

1. **terminus-gui never imports any terminus-platform-* crate.**
2. **terminus-core never shells out to any command.**
3. **The GUI reads only from SystemModel. Never SQLite. Never the filesystem.**
4. **Platform detection is automatic. The user never configures their distro.**
5. **Adding a platform means adding one adapter crate and wiring it into the launcher. Zero changes to terminus-gui or terminus-core.**
6. **Terminus never writes to the filesystem beyond its own database.**
