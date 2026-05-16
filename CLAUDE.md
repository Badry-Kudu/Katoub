# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Katoub is a graphical editor for [Typst](https://github.com/typst/typst) files with a strong bias for Right-to-Left editing, based on Katvan (binary/bundle/app id are still `katvan`/`Katvan` — only the project framing was renamed). It targets Linux, Windows 10/11, and (experimentally) macOS 12+.

Project goals — in priority order, used to decide whether a feature is in scope (see `CONTRIBUTING.md`):

1. **RTL first.** Features that only benefit LTR users are out of scope.
2. **Typst only.** No LaTeX/Markdown/HTML support; tight Typst-compiler integration is preferred.
3. **Simple/familiar SDI editor**, optimized for short-to-medium standalone documents.
4. **Cross-platform** (Linux Wayland+X11, Windows, macOS) with equivalent — not identical — features.

## Build and test

The build is driven by CMake (≥ 3.22) with the Rust `typstdriver_rs` crate pulled in via [Corrosion](https://github.com/corrosion-rs/corrosion) (auto-fetched if absent). Toolchain expectations: C++20 compiler supported by Qt, Qt 6.8+ (6.9.1+ recommended), Rust stable (crate pins `rust-version = "1.89"`), `pkg-config`, libarchive, hunspell (Linux only), GoogleTest (optional, for tests).

```bash
# Linux / macOS configure+build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release      # macOS CI uses Debug + -DCMAKE_OSX_DEPLOYMENT_TARGET=12.0
cmake --build build -j

# Windows: vcpkg supplies gtest/libarchive/hunspell. NEVER build Debug on MSVC —
# Rust's std always links the release CRT, so a Debug C++ link is broken.
# Use RelWithDebInfo (Debug is removed from CMAKE_CONFIGURATION_TYPES for MSVC).
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake
cmake --build build -j --config RelWithDebInfo

# Run the full test suite
ctest --output-on-failure --test-dir build

# Run a single GoogleTest case (tests are discovered via gtest_discover_tests)
./build/tests/katvan_tests --gtest_filter='TokenizerTests.Escapes'

# Rust-side checks (run from repo root)
cargo clippy --no-deps --manifest-path typstdriver/rust/Cargo.toml --target-dir build/cargo/build
cargo fmt --manifest-path typstdriver/rust/Cargo.toml
```

CI extras worth knowing about (`.github/workflows/build.yml`): a `clazy` C++ lint pass (build with `-DCMAKE_CXX_COMPILER=clazy`), `cargo clippy`, and a `typos` spell-check pass (configured in `typos.toml`; currently invoked as `typos || true`, so it reports findings but does **not** fail the build).

Useful CMake options used by packagers/installers:

- `KATVAN_PORTABLE_BUILD` — Windows portable build (settings go next to the .exe; runtime flag `--no-portable` overrides).
- `KATVAN_DISABLE_PORTABLE` — strip portable mode entirely (Flatpak/system installs).
- `KATVAN_FLATPAK_BUILD` — enables the `flatpak` Cargo feature on `typstdriver_rs` (adds `xattr`-based sandbox hooks).
- `APPIMAGE_INSTALL` — adjusts install rpaths for AppImage layout.
- `KATVAN_CARGO_PROFILE` — forwarded to Corrosion to pick the Cargo profile.

The unit-test runner (`tests/main.cpp`) forces `QT_QPA_PLATFORM=minimal` and creates a `QApplication`, so tests can use Qt GUI types in headless environments. Mock hunspell dictionaries in `tests/hunspell/` are copied next to the test binary by the `test_dicts` custom target.

## Architecture

There are **four** CMake targets that compose the app, plus a Rust crate:

```
typstdriver_rs (Rust)  ──cxx bridge──▶  typstdriver (C++ SHARED lib)
                                          │
                                          ▼
                                    katvan_core (C++ STATIC lib, Qt Widgets)
                                          │
                          ┌───────────────┼───────────────┐
                          ▼                               ▼
                  shell/katvan (Linux/Win)        macshell/Katvan.app (macOS)
                  Qt Widgets MainWindow           Native AppKit (Objective-C++)
```

`CMakeLists.txt` (root) picks **either** `shell/` (non-Apple) **or** `macshell/` (Apple) — they are mutually exclusive UI front-ends sharing all editor/parser logic via `katvan_core`. When making changes that touch UI behavior, the corresponding change must be considered for the other shell, even if not implemented there immediately (cross-platform goal #4 in `CONTRIBUTING.md`).

### `typstdriver/` — Typst integration boundary

`typstdriver/rust/` contains crate `typstdriver_rs` (`crate-type = ["staticlib"]`), which wraps the upstream `typst`, `typst-pdf`, `typst-render`, `typst-ide`, `typst-kit` crates. **All Rust↔C++ communication crosses through `typstdriver/rust/src/bridge.rs`** via `cxx::bridge`; the `corrosion_add_cxxbridge` call generates the matching C++ header at build time. Adding or changing an FFI type/function requires editing **both** sides of the bridge.

The C++ side (`typstdriver_engine.{h,cpp}`, `typstdriver_packagemanager.cpp`, `typstdriver_logger.cpp`) wraps the Rust engine in QObject classes. The shared library exports symbols via `typstdriver_export.h` (generated by `generate_export_header`), uses hidden visibility by default, and is bundled as a macOS Framework on Apple.

`core/katvan_typstdriverwrapper.{h,cpp}` is the **only** place the rest of the app talks to the engine — it moves the engine to its own `QThread`, batches pending edits, and re-emits compiler events back on the UI thread. Don't bypass it.

### `core/` — Platform-neutral editor library

Static library `katvan_core` (namespace `katvan::`, with sub-namespace `katvan::parsing` for the parser). It links Qt6 Widgets and `typstdriver`, plus hunspell (Linux) or the native Windows spellchecker (Win). On macOS, the platform spell checker lives in `macshell/` instead. Notable building blocks:

- `katvan_parsing.{h,cpp}` — Hand-written streaming **tokenizer + parser** for Typst source, exposed through a **listener pattern** (`ParsingListener` subclasses receive `initializeState`/`finalizeState`/`handleLooseToken`). Multiple listeners (e.g. `HighlightingListener`, `ContentWordsListener`, `IsolatesListener`, `StateSpansListener` in `katvan_codemodel.h`) run on the same parse to extract syntax highlighting markers, natural-language segments for spell checking, BiDi isolate ranges, and persistent state spans. When adding a feature that needs to know "what kind of Typst construct is at position X," prefer adding a `ParsingListener` rather than reparsing or grepping the buffer.
- `katvan_codemodel.{h,cpp}` — Higher-level queries on top of parser state spans: environment classification (content/code/math), bracket matching, indent rules, symbol/color/label expression generation. Lives on the `QTextDocument`.
- `katvan_editor.{h,cpp}` + `katvan_editorlayout.{h,cpp}` — `QTextEdit` subclass with custom layout for RTL handling (line directionality heuristics, BiDi marks, dual line-number gutters, logical-vs-visual cursor movement). The editor wires up `Highlighter`, `CompletionManager`, `SpellChecker`, `CodeModel`, theming, and per-file modelines (see `katvan_editorsettings.cpp`).
- `katvan_typstdriverwrapper.{h,cpp}` — see above.

### `shell/` — Qt Widgets cross-platform UI

`MainWindow` (`katvan_mainwindow.cpp`, ~38 KB) is the central wiring point: it owns the `TypstDriverWrapper`, `Editor`, `Previewer`, `BackupHandler`, `SpellChecker`, dock widgets (preview / compiler output / outline / labels), the search bar, and dialogs. New shell-level features generally hook into here. The binary is named `katvan` on Linux/Windows and `Katvan` on macOS (the shell isn't used on macOS, but the convention is kept).

### `macshell/` — Native AppKit shell

Objective-C++ (`.mm`), AppKit-based, compiled with `-fobjc-arc` (ARC is enabled — do not use `retain`/`release`). The structure mirrors `shell/` (`macshell_windowcontroller.mm`, `macshell_previewer.mm`, `macshell_editorview.mm`, etc.) but uses Cocoa text/window/menu APIs while still calling into `katvan_core`. Bundled as `Katvan.app` with `typstdriver` embedded as a framework.

### `assets/`, `scripts/`, `tests/`

- `assets/` — App icons (light/dark), fonts (incl. `KatvanControl.otf` for rendering BiDi control characters), `.desktop` file, AppStream metainfo template (`@METAINFO_RELEASES_XML@` is substituted from `CHANGELOG.md` at configure time by `scripts/format_changelog.py`), Windows `.rc` template, NSIS installer script.
- `scripts/format_changelog.py` — Parses `CHANGELOG.md` (mistletoe) into either AppStream XML or plain release notes (`--format appstream|plaintext --limit N`). Run during CMake configure when Python 3 + mistletoe are available.
- `tests/` — GoogleTest-based unit tests; one `_*.t.cpp` per covered core module. macOS/Windows skip `katvan_spellchecker.t.cpp` (hunspell isn't built there).

## Conventions

- **File naming.** C++ files use the `katvan_` prefix matching their primary class; test files use the `_*.t.cpp` suffix. Objective-C++ uses the `macshell_` prefix instead.
- **Formatting.** `.editorconfig` sets 4-space indent, LF, UTF-8, trailing-whitespace trim for `{cpp,mm,h,rs}`. Rust **must** be `cargo fmt`'d; **there is no auto-formatter for C++ or Objective-C++ — match surrounding style**.
- **No UI designers.** Do not introduce `.ui` files / Qt Creator forms or Xcode Interface Builder XIBs (explicit project policy in `CONTRIBUTING.md`).
- **Qt compile-time guards.** `katvan_core` and `typstdriver` define `QT_RESTRICTED_CAST_FROM_ASCII` and `QT_NO_EMIT`: prefer `QStringLiteral`/`u"..."_s`, and write signal emissions without the `emit` keyword.
- **Warnings as policy.** Linux/macOS build with `-Wall -Wextra -pedantic -Wno-switch`; MSVC with `/W4`. Fix new warnings rather than silencing them.
- **Translations.** Qt strings live in `core/translations/*.ts` and `shell/translations/*.ts` (managed via `lupdate`/`qt_add_translations`). macOS strings live in `macshell/translations/<lang>.lproj/Localizable.{strings,stringsdict}` and are kept in sync by `bartycrouch update` (run automatically if `bartycrouch` is on PATH; install with `brew install bartycrouch`). Only Hebrew (`he`) is currently a translated language (`TRANSLATION_LANGUAGES` in root `CMakeLists.txt`).
- **Versioning.** `PROJECT_VERSION` in the root `CMakeLists.txt` is propagated via `core/katvan_version.h.in` (compiled with the short Git SHA). Bump it there.
- **Spell-check exemptions.** Append `// spellchecker:disable-line` (or `# …` in shell/Python) to a line you need `typos` to ignore.

## Contribution guardrails (apply when editing on behalf of a contributor)

These come from `CONTRIBUTING.md` and are enforced socially, not by tooling:

- **No "drive-by" PRs.** Non-trivial changes must reference an existing issue where the approach was discussed.
- **DCO sign-off required.** Every commit needs a `Signed-off-by:` trailer (`git commit -s`). Don't author commits that lack one.
- **AI usage must be disclosed.** If an LLM-style tool produced any non-trivial code in a PR, that fact (and scope) must be in the PR description. Issue/PR prose itself must be human-written. Be conservative and surface AI involvement to the user.
- **Tests** are expected for new features where reasonably possible — extend `tests/katvan_*.t.cpp`.
