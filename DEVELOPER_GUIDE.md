# Developer Guide

This document is the practical handover for a developer taking over Katoub. It covers what's in the codebase, where to look first, how to be productive, what to be careful of, and where the most valuable upgrade work is likely to be.

Companion documents:

- `README.md` — user-facing overview, install/build instructions.
- `CONTRIBUTING.md` — project policy (DCO sign-off, AI disclosure, no UI designers).
- `CLAUDE.md` — condensed guidance for AI assistants (mirrors much of this document at higher density).
- `HISTORY.md` — how the project got to where it is, including the Katvan → Katoub fork.

---

## 1. One-page mental model

Katoub is a Qt-based graphical editor for Typst documents, with a strong bias for right-to-left (RTL) editing. The compiler itself (Typst) is **embedded** inside the application as a Rust library; the editor talks to it through a C++ shim that runs the engine on its own thread.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  typstdriver_rs (Rust crate, staticlib)                                 │
│    – wraps upstream typst / typst-pdf / typst-render / typst-ide / kit  │
│    – rust/src/{engine,export,world,pathmap,symbols,analysis}.rs         │
│                          │                                              │
│            cxx::bridge in typstdriver/rust/src/bridge.rs                │
│                          │                                              │
│  typstdriver (C++ SHARED library, hidden visibility, framework on mac)  │
│    – typstdriver_engine.{h,cpp}        QObject wrapping the Rust engine │
│    – typstdriver_packagemanager.cpp    @preview package fetch/cache     │
│    – typstdriver_logger.cpp            compiler diagnostics             │
└─────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ (lives on a dedicated QThread)
                                  │
┌─────────────────────────────────────────────────────────────────────────┐
│  katvan_core (C++ STATIC library, namespace katvan)                     │
│    – the ONLY entry point to typstdriver is katvan_typstdriverwrapper   │
│    – katvan_parsing  (tokenizer + parser + listener pattern)            │
│    – katvan_codemodel (state spans, env classification, brackets)       │
│    – katvan_editor / katvan_editorlayout (QTextEdit + RTL layout)       │
│    – katvan_highlighter, katvan_completionmanager, katvan_spellchecker  │
│    – katvan_outlinemodel, katvan_diagnosticsmodel, katvan_wordcounter   │
│    – katvan_symbolpicker, katvan_previewerview, katvan_editortheme      │
└─────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │
                ┌─────────────────┴─────────────────┐
                │                                   │
┌──────────────────────────────┐    ┌──────────────────────────────────┐
│  shell/  — Qt Widgets        │    │  macshell/  — AppKit (Obj-C++)   │
│  Linux + Windows             │    │  macOS (default since 2026-04)   │
│  binary: `katvan`            │    │  bundle: `Katvan.app`            │
│  central widget: MainWindow  │    │  central: WindowController       │
└──────────────────────────────┘    └──────────────────────────────────┘
```

The root `CMakeLists.txt` picks **either** `shell/` **or** `macshell/`; they share `katvan_core` but never coexist in one binary.

---

## 2. Build, run, test

### Toolchain expectations

- CMake **≥ 3.22**
- C++20 compiler supported by Qt (GCC, Clang, Apple Clang, MSVC `/W4`)
- **Qt 6.8+** (6.9.1+ recommended; some features such as visible BiDi control characters need 6.9)
- Stable Rust toolchain (`typstdriver/rust/Cargo.toml` pins `rust-version = "1.89"`)
- `pkg-config`, `libarchive`, `hunspell` (Linux only)
- GoogleTest (optional but enables the `katvan_tests` target)
- [Corrosion](https://github.com/corrosion-rs/corrosion) ≥ 0.6.0 — auto-fetched if missing

### Common build commands

```bash
# Linux / macOS Release build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j

# Linux / macOS Debug build (matches CI; required to run clazy/clippy locally)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j

# Windows (vcpkg supplies gtest/libarchive/hunspell)
# NEVER build Debug on MSVC — Rust's std always links the release CRT.
# The root CMakeLists.txt removes "Debug" from CMAKE_CONFIGURATION_TYPES on MSVC.
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake
cmake --build build -j --config RelWithDebInfo
```

### Tests

```bash
# Full suite
ctest --output-on-failure --test-dir build

# Single GoogleTest (tests are discovered with gtest_discover_tests)
./build/tests/katvan_tests --gtest_filter='TokenizerTests.Escapes'

# List discoverable tests
./build/tests/katvan_tests --gtest_list_tests
```

`tests/main.cpp` forces `QT_QPA_PLATFORM=minimal` and constructs a `QApplication`, so tests may use Qt GUI types in headless environments. Mock hunspell dictionaries in `tests/hunspell/` are copied next to the test binary at build time. On Windows and macOS, `katvan_spellchecker.t.cpp` is **excluded** (hunspell isn't built on those platforms).

### Rust-side checks

```bash
cargo clippy --no-deps --manifest-path typstdriver/rust/Cargo.toml --target-dir build/cargo/build
cargo fmt --manifest-path typstdriver/rust/Cargo.toml
```

`cargo fmt` is **mandatory** — there is no auto-formatter for C++ or Objective-C++; for those you match the surrounding style.

### Other CI checks

CI (`.github/workflows/build.yml`) also runs:

- A `clazy` pass: `cmake -S . -B build -DCMAKE_CXX_COMPILER=clazy`.
- The `typos` spell-check across the tree (configured by `typos.toml`; suppress on a single line with `// spellchecker:disable-line` or `# spellchecker:disable-line`). Note: CI currently invokes `typos || true`, so findings are reported but do not gate the build.

### CMake options that matter for packaging

| Option                       | Purpose                                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------------------- |
| `KATVAN_PORTABLE_BUILD`      | Windows portable build (settings sit next to `.exe`; `--no-portable` flag overrides).    |
| `KATVAN_DISABLE_PORTABLE`    | Strip portable mode entirely (used by Flatpak and system installs).                      |
| `KATVAN_FLATPAK_BUILD`       | Enables the `flatpak` Cargo feature on `typstdriver_rs` (`xattr`-based sandbox hooks).   |
| `APPIMAGE_INSTALL`           | Adjusts install rpaths for AppImage layout.                                              |
| `KATVAN_CARGO_PROFILE`       | Forwarded to Corrosion to pick a non-default Cargo profile.                              |

---

## 3. Where things live

### `core/` — platform-neutral editor library (`katvan_core`)

Static library. Namespace `katvan::` (parser is in sub-namespace `katvan::parsing`). Top files to know:

| File                                  | Responsibility                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------------- |
| `katvan_parsing.{h,cpp}`              | Hand-written **streaming tokenizer + parser** for Typst source. Listener-based.             |
| `katvan_codemodel.{h,cpp}`            | Higher-level queries on top of parser state spans (env classification, brackets, indent).   |
| `katvan_highlighter.{h,cpp}`          | `QSyntaxHighlighter` subclass that drives the parser and applies the resulting char formats.|
| `katvan_editor.{h,cpp}`               | `QTextEdit` subclass — the main editing widget (~50 KB, the heart of the editor).           |
| `katvan_editorlayout.{h,cpp}`         | Custom line layout: dual line-number gutters, BiDi marks, line-directionality heuristic.    |
| `katvan_editorsettings.{h,cpp}`       | Settings model + modeline parser.                                                           |
| `katvan_editortheme.{h,cpp}`          | Theme model. Themes live in `core/themes/*.json` (loaded via Qt resources).                 |
| `katvan_editortooltip.{h,cpp}`        | Tooltip widget with HTML rendering + link handling.                                         |
| `katvan_completionmanager.{h,cpp}`    | Autocomplete popup driven by `typst-ide`.                                                   |
| `katvan_spellchecker.{h,cpp}`         | Abstract base + factory.                                                                    |
| `katvan_spellchecker_hunspell.{h,cpp}`| Linux Hunspell backend.                                                                     |
| `katvan_spellchecker_windows.{h,cpp}` | Windows ISpellChecker backend. (macOS uses `macshell/macshell_spellchecker.mm` instead.)    |
| `katvan_document.{h,cpp}`             | `QTextDocument` subclass with block-data plumbing.                                          |
| `katvan_outlinemodel.{h,cpp}`         | Tree model for the document outline (headings).                                             |
| `katvan_diagnosticsmodel.{h,cpp}`     | Model for compiler errors/warnings.                                                         |
| `katvan_symbolpicker.{h,cpp}`         | Symbol picker dialog, fed by `typst-ide` symbol data.                                       |
| `katvan_wordcounter.{h,cpp}`          | Status-bar word counter.                                                                    |
| `katvan_previewerview.{h,cpp}`        | The actual preview rendering widget (paint-on-demand at the current zoom).                  |
| `katvan_typstdriverwrapper.{h,cpp}`   | **The only thing in the project that talks to `typstdriver` directly.**                     |
| `katvan_text_utils.{h,cpp}`           | Helpers for fonts, BiDi marks, etc.                                                         |

### `typstdriver/` — Typst integration boundary

Shared library (framework on macOS). Hidden visibility; symbols are exported with `TYPSTDRIVER_EXPORT` (header generated by `generate_export_header`). The actual compiler work happens in Rust.

| File                                   | Responsibility                                                                   |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| `typstdriver_engine.{h,cpp}`           | `QObject` wrapping `EngineImpl` from Rust. All public slots/signals live here.   |
| `typstdriver_logger.{h,cpp,_p.h}`      | `LoggerProxy` exposed to Rust; emits structured diagnostics back to Qt.          |
| `typstdriver_packagemanager.{h,cpp,_p.h}` | `@preview` package download + cache + Flatpak sandbox hooks.                  |
| `typstdriver_compilersettings.{h,cpp}` | Settings POD passed across the bridge.                                           |
| `typstdriver_outlinenode.{h,cpp}`      | Outline tree returned to Qt.                                                     |
| `rust/Cargo.toml`                      | Pins `typst*` 0.14.1, `cxx` 1.0.186; profile uses `lto = "thin"` for release.    |
| `rust/src/bridge.rs`                   | **The cxx bridge.** Adding/changing an FFI type touches this and a C++ header.   |
| `rust/src/engine.rs`                   | `EngineImpl` — owns the Typst `World`, edits, compile/render/export.             |
| `rust/src/world.rs`                    | Implements `typst::World` against the live source + filesystem.                  |
| `rust/src/export.rs`                   | PDF/PNG export.                                                                  |
| `rust/src/pathmap.rs`                  | Translates editor paths to Typst's virtual FS.                                   |
| `rust/src/symbols.rs`                  | Symbol-picker data.                                                              |
| `rust/src/analysis/{mod,tooltip}.rs`   | Tooltip enrichment on top of `typst-ide`.                                        |

### `shell/` — Qt Widgets cross-platform UI

Used on Linux and Windows; not used on macOS. Binary name: `katvan` (or `Katvan` on Apple, by convention). `MainWindow` (`shell/katvan_mainwindow.cpp`, ~38 KB) is the central wiring point — it owns `Editor`, `TypstDriverWrapper`, `Previewer`, `BackupHandler`, `SpellChecker`, `WordCounter`, dock widgets (preview / compiler output / outline / labels), the search bar, and dialogs. **New shell-level features almost always hook in here.**

Other files of note: `katvan_searchbar.cpp`, `katvan_exportdialog.cpp`, `katvan_settingsdialog.cpp`, `katvan_backuphandler.cpp` (crash recovery), `katvan_recentfiles.cpp`.

### `macshell/` — native AppKit shell

Used on macOS (default since `76434c8`, 2026-04-08). Objective-C++ (`.mm`), AppKit-based, **compiled with `-fobjc-arc`** (ARC is on — do not use `retain`/`release`). Mirrors `shell/` structurally:

| Qt shell file               | AppKit equivalent                             |
| --------------------------- | --------------------------------------------- |
| `katvan_mainwindow.cpp`     | `macshell_windowcontroller.mm`                |
| `katvan_previewer.cpp`      | `macshell_previewer.mm`                       |
| `katvan_settingsdialog.cpp` | `macshell_settingsdialog.mm`                  |
| `katvan_exportdialog.cpp`   | `macshell_exporter.mm`                        |
| `katvan_compileroutput.cpp` | `macshell_issuelist.mm`                       |
| `katvan_labelsview.cpp`     | `macshell_labelsview.mm`                      |
| `katvan_outlineview.cpp`    | `macshell_outlineview.mm`                     |
| `katvan_recentfiles.cpp`    | (handled by AppKit document architecture)     |
| (none — uses `core`)        | `macshell_spellchecker.mm` (NSSpellChecker)   |

Bundle is `Katvan.app` with `typstdriver` embedded as a framework. Translations live in `macshell/translations/<lang>.lproj/Localizable.{strings,stringsdict}` and are kept in sync with the source by `bartycrouch update` (run automatically by CMake if `bartycrouch` is on PATH; install with `brew install bartycrouch`).

### `assets/`, `scripts/`, `tests/`

- `assets/` — app icons (light/dark sets), fonts (incl. `KatvanControl.otf` for BiDi-control glyphs, requires Qt 6.9+), `.desktop` file, AppStream metainfo template, Windows `.rc` template, NSIS installer script.
- `scripts/format_changelog.py` — parses `CHANGELOG.md` (via `mistletoe`) into either AppStream XML or plain release notes (`--format appstream|plaintext --limit N`). Run during CMake configure if Python 3 + mistletoe are available.
- `tests/` — GoogleTest unit tests, one `_*.t.cpp` per covered core module. Big ones: `katvan_parsing.t.cpp` (~40 KB), `katvan_codemodel.t.cpp` (~33 KB), `katvan_editorsettings.t.cpp`.

---

## 4. The few architectural rules you must not break

These are non-obvious load-bearing invariants. Violating any of them creates very hard-to-debug bugs.

1. **All editor↔Typst communication goes through `katvan_typstdriverwrapper`.** It moves the engine to its own `QThread`, batches pending edits, and re-emits Typst events back to the UI thread. Don't construct a `typstdriver::Engine` from anywhere else, don't post directly to the engine's slots, don't read its state synchronously.
2. **The parser is single-pass and listener-driven.** When you need to know "what kind of Typst construct is at position X," **add a `ParsingListener` subclass** rather than re-parsing or grepping the buffer. Existing listeners (`HighlightingListener`, `ContentWordsListener`, `IsolatesListener`, `StateSpansListener`) are good models. Multiple listeners are attached to the same parse; the cost of adding one more is small, the cost of a second parse is real.
3. **Any FFI change requires editing both sides of `cxx::bridge`.** `typstdriver/rust/src/bridge.rs` is the source of truth; `corrosion_add_cxxbridge` generates the matching C++ header at build time. New types/functions added on one side must appear on the other. If a Rust function returns errors, declare it `Result<…>` in the bridge.
4. **Windows must build `RelWithDebInfo`, not `Debug`.** Rust's MSVC `std` always links the release CRT; a Debug C++ link breaks. The root `CMakeLists.txt` removes `Debug` from `CMAKE_CONFIGURATION_TYPES` for MSVC. If you find yourself trying to enable Debug on Windows, you're going down the wrong rabbit hole — see [rust-lang/rust#39016](https://github.com/rust-lang/rust/issues/39016).
5. **Qt compile-time guards in `katvan_core` and `typstdriver`.** Both define `QT_RESTRICTED_CAST_FROM_ASCII` and `QT_NO_EMIT`. Prefer `QStringLiteral` / `u"..."_s`; never write `emit signalName(...)`, just call it as a function. Adding `QT_NO_KEYWORDS` is fine; removing these guards is not.
6. **`-fobjc-arc` on the macshell.** Memory management in `macshell/*.mm` is automatic. Don't introduce manual `retain`/`release`/`autorelease`.
7. **Cross-platform parity is a project goal, not a "nice to have".** Per `CONTRIBUTING.md` goal #4: shell-level features should have a plan for the other front-end, even if not implemented immediately. When you change a `shell/` file, ask whether the same change is meaningful in `macshell/` (and vice versa).
8. **RTL is goal #1.** Per `CONTRIBUTING.md`: features that only benefit LTR users are out of scope. Any change to editor layout, cursor movement, gutters, or text input must be tested in RTL.

---

## 5. Common editing tasks — where to start

| Task                                                              | Start here                                                                                        |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| New syntax highlighting kind                                      | `parsing::HighlightingMarker::Kind`, `HighlightingListener`, theme JSON files, `Highlighter::doSyntaxHighlighting` |
| New auto-indent rule                                              | `CodeModel::shouldIncreaseIndent` / `findMatchingIndentBlock`                                     |
| New "insert helper" menu item                                     | `Editor::createInsertMenu` (core) + `MainWindow` (shell) / `macshell_mainmenu.mm`                 |
| New compiler-output column / panel                                | `core/katvan_diagnosticsmodel.cpp` + `shell/katvan_compileroutput.cpp` + `macshell/macshell_issuelist.mm` |
| New compiler setting                                              | `typstdriver/typstdriver_compilersettings.{h,cpp}`, `bridge.rs` (`set_compiler_flags`/`set_allowed_paths`), `engine.rs`, both settings dialogs |
| Upgrade Typst                                                     | `typstdriver/rust/Cargo.toml` (`typst*` versions), then build, then fix breakage in `engine.rs` / `world.rs` / `analysis/` |
| New translation string                                            | Edit code with `tr(...)`; run `lupdate` via the generated CMake targets, or `bartycrouch update` for macOS strings |
| New autocomplete suggestion source                                | `core/katvan_completionmanager.{h,cpp}` and the `get_completions` path through `bridge.rs` → `engine.rs` |
| New theme                                                         | Add JSON to `core/themes/`, register in `core/themes/themes.qrc`                                  |
| Tweak BiDi isolation heuristic                                    | `parsing::IsolatesListener` (`core/katvan_parsing.cpp`)                                           |
| Add a single new test                                             | `tests/katvan_<module>.t.cpp` — discovered automatically by `gtest_discover_tests`                |

---

## 6. Known pain points / outstanding TODOs in the source

These were found by grepping for `TODO`, `FIXME`, `XXX`, `HACK` in C++ sources. They are inherited from upstream Katvan; treat them as known weak spots, not new issues:

- **Full re-highlight on each parse** (`core/katvan_highlighter.cpp`):
  > `// TODO: This way (recalculating from scratch after every change) ensures correctness, but might be too expensive on very large documents.`
  If profiling ever shows highlighter pressure, this is the place to add incremental recomputation.
- **Throttle** something in the highlighter:
  > `// TODO: Possibly throttle this`
  Worth investigating if you see redundant work on rapid edits.
- **Duplication of static export definitions** (`macshell/macshell_exporter.mm`):
  > `// FIXME: This duplicates a lot of static definitions with the Qt shell export`
  A refactor candidate: move the PDF/PNG export option definitions into `core/` (since the data is platform-neutral) and have both shells consume them.
- **An "in principle error" suppressed silently** (`core/katvan_editor.cpp`):
  > `// XXX this is in principal an error condition, and we might [...]`
  Read in context before touching; it likely guards a race the original author chose not to crash on.

The roadmap (`ROADMAP.md`) also lists two long-standing unchecked items: **true live preview** and a **user manual**.

---

## 7. Recommendations for future upgrades and improvements

What follows is opinion grounded in the code as it stands. These are not pre-approved plans — discuss in an issue before starting anything non-trivial (per `CONTRIBUTING.md`).

### 7.1 If you are forking to make a different product (Katoub specifically)

The fork as of this commit is a rename of `README.md` only. To make Katoub a meaningfully different project, do these in order:

1. **Define the new product's scope.** Write down which of Katvan's four project goals (RTL-first, Typst-only, simple/familiar SDI, cross-platform) you are keeping, modifying, or dropping. Many downstream decisions follow from this.
2. **Rebrand the binary, bundle id, and namespaces** — not just the README. This is currently the highest-impact, lowest-risk piece of divergence work and the most awkward to do later because it touches every file. Concretely:
   - Rename `shell/` binary target from `katvan` to e.g. `katoub` (root `CMakeLists.txt`, install rules, AppImage rpath, NSIS script).
   - Change `QCoreApplication::setOrganizationDomain` / `setOrganizationName` / `setApplicationName` in `shell/main.cpp` and `macshell/main.mm`. These determine where settings get stored — changing them later **silently invalidates every user's existing settings**, so doing it before you have users is much cheaper.
   - Rename the macOS bundle (`OUTPUT_NAME Katvan` in `macshell/CMakeLists.txt`) and bundle id `app.katvan.Katvan` (in `Info.plist.in` and `assets/app.katvan.Katvan.metainfo.xml.in`).
   - Decide whether to rename the C++ `katvan::` namespace, the `katvan_` file prefix, and the `core/katvan_*.h` header names. This is mechanical (`git grep -l katvan` + sed) but invasive; if you don't do it now, do it never.
   - Update copyright headers (every file currently says `Copyright (c) 2024 - 2026 Igor Khanin`). Either add a second line for the new copyright holder, or replace the header on changed files only. **Do not strip Igor Khanin's copyright** — the project is GPLv3.
   - Replace the icon set (`assets/katvan*.png`, `katvan.svg`, `katvan.icns`, `katvan.ico`).
3. **Set up your own release pipeline.** The existing `.github/workflows/release.yml` builds AppImage / Flatpak / Windows installer / macOS bundle for the upstream project. Either adapt it (replace signing keys, update artifact names, change the `app.katvan.Katvan` Flathub id) or replace it.
4. **Decide on upstream-tracking policy.** Katvan is still actively developed; the most recent upstream commit was 2026-04-23. Options:
   - **Tracking fork** — periodically `git merge upstream/master`. Cheap if you make minimal changes to the same files.
   - **Hard fork** — abandon upstream sync. Cheaper for big divergence, but you give up upstream bug fixes (notably Typst version bumps, which are non-trivial). Recommended only after you've replaced or rewritten significant parts.

### 7.2 Technical upgrades worth considering

These are project-level improvements that would help regardless of fork direction.

- **Adopt clang-format on a per-directory basis.** The project currently has no auto-formatter for C++/Objective-C++ ("match surrounding style" is the rule). Introducing `clang-format` is contentious and would produce a huge diff, but it removes a permanent class of code-review friction. If you do this, do it on `katvan_core` first (it's the most touched area), in a single commit with no other changes, and pin the `.clang-format` to match the current style as closely as possible.
- **Pre-commit hook for `cargo fmt` and `typos`.** Both already run in CI; running them locally avoids the round trip.
- **Adopt CMake presets (`CMakePresets.json`).** The project has several common configurations (Linux Debug, macOS Debug + deployment target, Windows + vcpkg, Flatpak, AppImage, portable). They're documented in CI YAML and `CMakeLists.txt` but not formalized — a presets file would make `cmake --preset linux-debug` Just Work for new contributors.
- **Coverage report.** Test coverage of the parser and code model is reasonable; coverage of the editor (~50 KB file) is not measured. `gcov`/`llvm-cov` + `lcov` in CI would give an honest picture.
- **Profile-guided optimization (PGO) of the Rust side.** Embedded compilers tend to benefit; `lto = "thin"` is already on for release, but PGO could be added through `KATVAN_CARGO_PROFILE`.
- **Bench harness for the parser.** `katvan_parsing.t.cpp` is correctness-only. Adding a Google Benchmark target on the same data would help quantify the "rehighlight from scratch" cost noted in §6.
- **Consider Tree-sitter for syntax highlighting.** The hand-written parser is fast, well-tested, and well-suited to the listener pattern, so this is **not** an obvious win. But Tree-sitter has a mature Typst grammar (`tree-sitter-typst`), supports incremental parsing natively, and would give error-recovery for free. Worth evaluating if the TODO-flagged perf issue ever becomes real, or if you want to expose the parse tree to plugins.
- **Move the Qt-side compositional shell pieces into a `MainWindow` controller, not a `MainWindow` god-class.** `shell/katvan_mainwindow.cpp` is ~38 KB. The `macshell` rewrite already split things into separate controllers (window/previewer/sidebar/exporter/etc.); the Qt shell would benefit from the same shape, and it would make it easier to swap individual panels.

### 7.3 Feature directions left open by upstream's roadmap

These are checked or partially checked in `ROADMAP.md` but have natural follow-ups:

- **True live preview** (`ROADMAP.md`, "Top priority, a 1.0 release won't happen without it"). Incremental compilation has been in place since v0.8.0; the missing piece is UX work to surface intermediate results without flicker. Likely starts in `katvan_typstdriverwrapper`'s edit-batching code and `core/katvan_previewerview.cpp`.
- **User manual** — listed as top priority on the roadmap and never started.
- **Code folding** and **indentation guides** — both listed under "Maybe?" on the roadmap. Both are natural extensions of `CodeModel`: state spans already know where blocks start and end.
- **TextBundle-like file format** — listed under "Sandbox Readiness". Would let users keep assets and the `.typ` source together in a sandboxable bundle.

### 7.4 Things that look tempting but are probably bad ideas

Listed so a new owner doesn't burn a week relearning what upstream already considered:

- **"Just use the Qt shell on macOS too" / "Just use AppKit-everywhere".** ~3 months of 2026 work went into the AppKit shell precisely because the Qt one didn't feel native enough on macOS. Reversing that is reversing a deliberate, recent, expensive decision.
- **Subprocess-based compilation again.** Pre-v0.7.0 Katvan shelled out to `typst`. The current embedded approach is what makes incremental compilation, IDE-style tooltips, and inverse search tractable. Subprocesses would regress all three.
- **Adding LaTeX / Markdown / HTML support.** Project goal #2 (Typst-first) is non-negotiable upstream — and most of the code (parser, tooltips, completions, symbol picker) assumes a single language. Multi-format would require a near-rewrite. If you want a multi-format editor, fork a multi-format editor.
- **Building a plugin system speculatively.** No existing customer for it; would freeze internal APIs that are currently free to refactor. Add it when there is a concrete second consumer.

---

## 8. Quick reference

| Need to…                          | File to open first                                             |
| --------------------------------- | -------------------------------------------------------------- |
| Change how Typst syntax is parsed | `core/katvan_parsing.cpp`                                      |
| Add a parser-driven feature       | New `ParsingListener` subclass; register it on the `Parser`    |
| Add a menu/shortcut (Linux/Win)   | `shell/katvan_mainwindow.cpp`                                  |
| Add a menu/shortcut (macOS)       | `macshell/macshell_mainmenu.mm` + window controller            |
| Talk to Typst                     | `core/katvan_typstdriverwrapper.{h,cpp}` (and nothing else)    |
| Add a Rust-side capability        | `typstdriver/rust/src/bridge.rs` → `engine.rs` → Qt wrapper    |
| Add a unit test                   | `tests/katvan_<module>.t.cpp` + maybe `tests/CMakeLists.txt`   |
| Bump Typst                        | `typstdriver/rust/Cargo.toml` (`typst*` versions)              |
| Bump project version              | `PROJECT_VERSION` in root `CMakeLists.txt`; propagated by `core/katvan_version.h.in` |
| Add a translation string          | `tr(...)` in source; `core/translations/*.ts` + `shell/translations/*.ts`; macOS uses `bartycrouch update` |
| Suppress a `typos` false positive | Append `// spellchecker:disable-line` (or `# …`)               |
