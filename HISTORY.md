# Project History

This document traces the evolution of the project from its origin as **Katvan** through the fork into **Katoub**, drawing on `CHANGELOG.md`, the Git history available in this repository, and the structure of the codebase itself. Dates are listed in `YYYY-MM-DD` form.

> **Note on Git history depth.** The Git log in this repository only goes back to commit `2fa87f3` (2025-11-21), which sits inside the v0.12.0 release cycle. Material before that point is reconstructed from `CHANGELOG.md`; release notes are quoted closely but condensed to what matters for understanding how the project grew.

## 1. Origin: Katvan (2024)

### v0.1.0 — 2024-01-22, "Initial release!"

The project began as **Katvan**, written by Igor Khanin as a personal solution to a very specific problem described in `README.md`:

> For University, I have a need to write documents that are mainly in Hebrew but also heavily incorporate math and inline English terms. […] Therefore Katvan is a new editor application, with a very specific focus on this particular use case; starting with a plain text editor that gets the basics right (at least for me), and goes from there.

At this point Katvan was a plain Qt Widgets editor that shelled out to the external `typst` CLI for compilation and preview. The whole RTL-first promise was already the headline feature.

### v0.2.0 — 2024-02-10

- **Syntax highlighting** of Typst sources. The hand-written tokenizer + parser that still lives in `core/katvan_parsing.{h,cpp}` was introduced here.
- **Official Windows 10/11 support.**
- Enhanced text find (regex, case matching, etc.).

### v0.3.0 — 2024-02-24

- **As-you-type spell checking** powered by Hunspell, syntax-aware so only content/markup is checked. This is the origin of `core/katvan_spellchecker_hunspell.{h,cpp}` and of the `ContentWordsListener` in the parser.

### v0.4.0 — 2024-06-08

- **Editor configuration system** with a settings dialog and per-file modelines (`core/katvan_editorsettings.cpp`).
- Find-in-selection and find/replace.
- Spell-check dictionaries loaded in the background to keep the UI responsive.

### v0.5.0 — 2024-06-22

- Typst now run in **continuous watch mode** as a background process, making follow-up compilations faster.
- Click an error in the compiler output to jump to the source location (the start of forward/inverse-search infrastructure).
- Crash-recovery from the compilation buffer.
- Dark-theme detection respecting the active Qt style.

### v0.6.0 — 2024-08-05

- **Bracket matching** and **auto-insertion** of closing brackets in a syntax-aware way. This is where `core/katvan_codemodel.{h,cpp}` started to take serious shape on top of the parser's state spans.
- "Normal" vs "Smart" auto-indentation modes.
- Preview pane can be undocked into a floating window.

### v0.7.0 — 2024-08-24 — **Embedded Typst compiler**

The most architecturally significant release in the project's history. From the changelog:

> Starting from this release, Katvan directly embeds the Typst compiler. This means that separately installing the Typst CLI is not required anymore […]

This is when the **`typstdriver/`** subproject was born: a Rust crate (`typstdriver_rs`) wrapping the upstream `typst`, `typst-pdf`, `typst-render`, `typst-ide`, and `typst-kit` crates, exposed to the C++ side through a `cxx::bridge` in `rust/src/bridge.rs`. Corrosion is used to integrate Cargo with CMake. All of the architectural complexity around the `core/katvan_typstdriverwrapper.{h,cpp}` shim (which runs the engine on its own `QThread`) traces to this moment.

Other v0.7.0 highlights:

- Preview pane rebuilt to render pages directly from the Typst intermediate result; no more PDF round-trip.
- **Forward search** (`Ctrl+J` → preview) and **inverse search** (`Ctrl+Click` on preview → editor).

### v0.7.1 — 2024-10-09

- Persistent line-directionality overrides (Katvan automatically maintains the appropriate BiDi control characters).
- New line-directionality heuristic: a line without strongly-directional characters inherits the previous line's direction.
- **Experimental macOS support.** This is the precursor of `macshell/` — though at this point the macOS build still used the Qt shell.

### v0.8.0 — 2024-11-15

- Embedded Typst upgraded to **0.12.0**.
- **Auto-completion** (manually triggered) and **compiler-assisted tooltips**.
- **Incremental compilation** — editor edits are synced directly to the Typst compiler via `applyContentEdit`, paving the way for instant previews.
- Compiler-settings tab: Typst Universe packages can be disabled, download-cache size shown.
- Windows installer.

### v0.8.1 — 2025-02-07

- Configurable additional include paths for documents.
- Fixed blurry previews under display scaling.

### v0.9.0 — 2025-03-30

- Embedded Typst upgraded to **0.13.1**.
- **Automatic directionality isolation** based on Typst syntax (inline math inside RTL, label references, inline code, content blocks inside code). This is where `IsolatesListener` in the parser became load-bearing.
- Errors and warnings highlighted directly on the editor text.
- **Breaking:** Windows spell-check now uses the system spell checker instead of Hunspell (origin of `core/katvan_spellchecker_windows.{h,cpp}`).

### v0.10.0 — 2025-06-03

A large release that filled in many of the "code-editor niceties" gaps:

- **Visible BiDi control characters** (with custom `KatvanControl.otf` font) — requires Qt 6.9.
- **Outline pane** (`core/katvan_outlinemodel.{h,cpp}`, `shell/katvan_outlineview.{h,cpp}`).
- Cursor-location history with a `Go` menu (back / forward).
- **Go-to-definition** (`F12` / `Ctrl`-click).
- `Ctrl+Space` autocomplete shortcut; **automatic suggestion popups as you type**.
- Universe `@preview` packages appear in `#import` completions.
- Editor font zoom (mouse wheel + `Ctrl`).
- File-on-disk watcher detects external edits/moves.
- `TYPST_FONT_PATHS` environment variable honored.

### v0.11.0 — 2025-08-14

- **Symbol Picker** dialog (`core/katvan_symbolpicker.{h,cpp}`).
- **Color Picker** with syntax-aware Typst color expression generation.
- **Labels panel** (`shell/katvan_labelsview.{h,cpp}`) with drag-and-drop reference insertion.
- Status-bar word count (`core/katvan_wordcounter.{h,cpp}`).
- New app icon by Noam Sahar.

### v0.11.1 — 2025-10-11

A bugfix release with `Ctrl/Cmd`-wheel preview zoom; the changelog already warned that v0.12 would raise the minimum Qt to 6.8.

### v0.12.0 — 2025-12-10 — "Pretty much all features that I envisioned are now implemented"

Quoting the changelog header:

> Katvan is just shy of two years old! At this point, pretty much all features that I envisioned are now implemented. From now on, we start the final road to 1.0, where the focus is going to be on strengthening existing features, catching up on gaps in the macOS version, and building up project infrastructure with the goal of making Katvan open to contribution and sustainable long term.

Key changes:

- Embedded Typst upgraded to **0.14.1**.
- Minimum Qt raised to **6.8**, minimum CMake to **3.22**.
- New **Export As** dialog with PDF/A standards and PNG export.
- Toggle for experimental PDF accessibility features.
- Symbol Picker shows deprecation status.
- Many tooltip/theming/CJK-input bugs fixed.

This is the most recent tagged release in the repository as of the time of writing.

## 2. The post-v0.12 "road to 1.0" — native macOS shell

The unreleased work in `master` after v0.12.0 is almost entirely about closing the Qt-vs-AppKit gap on macOS. From the Git log:

| Date         | Theme                                                           |
| ------------ | --------------------------------------------------------------- |
| 2026-01-03   | Copyright year bump                                             |
| 2026-02-01   | `CONTRIBUTING.md` + GitHub issue forms added                    |
| 2026-02-07   | Translation files split into `core` + `shell`                   |
| 2026-03-14   | Auto-complete polish                                            |
| 2026-03-20   | `macshell`: issue list in sidebar (first macshell-prefixed commit) |
| 2026-03-26   | `macshell`: sidebar tab persistence, window-size fixes          |
| 2026-03-30   | `macshell`: initial settings dialog                             |
| 2026-03-31   | `macshell`: compiler-settings parity                            |
| 2026-04-03   | `macshell`: status bar, compilation status, cursor-style save   |
| 2026-04-04   | `macshell`: spell checking, autocomplete menu item              |
| 2026-04-05   | `macshell`: labels panel; refactor PDF export                   |
| 2026-04-06   | `macshell`: PDF export options, find wraparound fix             |
| 2026-04-07   | `macshell`: refactor main window, change previewer location     |
| 2026-04-08   | `macshell`: follow cursor in preview; **macshell becomes default on macOS** (`76434c8`) |
| 2026-04-16   | Pin external GitHub Actions; small CONTRIBUTING tweaks; crate bumps |
| 2026-04-18   | `macshell`: Hebrew translation; BartyCrouch integration; pluralization |
| 2026-04-23   | `macshell`: RTL layout fixes; drag-and-drop labels              |

The big architectural change in this window is captured by `76434c8` ("Make macshell the default on macOS"): the project moved from "build the Qt shell everywhere" to a **mutually exclusive front-end model** — non-Apple platforms build `shell/`, Apple platforms build `macshell/`. Both consume the same `katvan_core` static library and `typstdriver` shared library. This is enforced in the root `CMakeLists.txt`:

```cmake
if(APPLE)
    add_subdirectory(macshell)
else()
    add_subdirectory(shell)
endif()
```

## 3. The fork: Katoub (2026-05-07)

On **2026-05-07**, commit `85962cb` by **Badry Darkoush** (`hi@badry.email`) created the Katoub fork:

```
commit 85962cb
Author: Badry Darkoush <hi@badry.email>
Date:   Thu May 7 22:57:46 2026 +0300

    Rename project to Katoub and update description

    Updated project name from 'Katvan' to 'Katoub' and clarified description.
```

This commit touches only `README.md`. The rename so far is **purely a project framing change**:

- The product is now described as "Katoub … based on Katvan".
- Binary names (`katvan` / `Katvan.app`), the `Q*CoreApplication::setApplicationName("Katvan")` call in `shell/main.cpp`, the bundle id `app.katvan.Katvan`, the assets prefix `katvan-`, namespaces (`katvan::`, `katvan::typstdriver`), file-name prefix `katvan_`, settings paths, and copyright headers (`Copyright (c) 2024 - 2026 Igor Khanin`) are **all still Katvan**.
- No functional, behavioral, or architectural divergence yet.

Subsequent commits on this branch (`claude/add-claude-documentation-wRHTI`):

- `409e5d8` (2026-05-16) — `Add CLAUDE.md guidance file` (AI-assistant project guide).
- This change — adds `HISTORY.md` (this document) and `DEVELOPER_GUIDE.md`.

In other words: as of writing, **Katoub is Katvan v0.12.0-plus-macshell-work with a renamed `README.md`**. Anything that diverges from upstream Katvan will start from here.

## 4. Inherited architectural decisions worth knowing

The fork inherits a few decisions that strongly shape what the codebase looks like today and what is easy vs. hard to change. They are listed here because they only make sense in light of the history above:

1. **Two front-ends, one core.** Choosing AppKit on macOS rather than reusing the Qt shell was a deliberate ~3-month investment in 2026. Any plan to consolidate back to a single Qt front-end (or, conversely, to add a third native front-end) is undoing or extending that decision — not a fresh design choice.
2. **Embedded Typst via `cxx::bridge`.** Since v0.7.0 (2024-08-24) the project does not shell out to an external `typst` CLI; it links the Rust crates directly. Replacing this with subprocesses, or with a different rendering backend, is a v0.7-scale change.
3. **Hand-written parser, listener pattern.** Syntax highlighting, spell checking, BiDi isolates, code-model state spans, outline, and labels are **all separate `ParsingListener` subclasses** running on the same single-pass parse. This was put in place starting v0.2 and steadily grew through v0.9 — it is the load-bearing abstraction for "what does the editor know about Typst syntax".
4. **`QT_RESTRICTED_CAST_FROM_ASCII` + `QT_NO_EMIT`.** These compile-time guards have been on since `katvan_core` and `typstdriver` existed. Code throughout the project uses `QStringLiteral`/`u"..."_s` and writes signal emissions without `emit`. Changing this would mean touching every file.
5. **DCO + AI-disclosure policy.** Introduced in February 2026 by `f60e28a` ("Add contribution guidelines and issue forms"). Inherited by Katoub.

## 5. Timeline summary

```
2024-01-22  v0.1.0   Katvan initial release (Qt shell + external Typst CLI)
2024-02-10  v0.2.0   Syntax highlighting (parser+tokenizer introduced)
2024-02-24  v0.3.0   Spell checking (Hunspell)
2024-06-08  v0.4.0   Settings + modelines
2024-06-22  v0.5.0   Background Typst watch mode
2024-08-05  v0.6.0   Bracket matching / auto-insert
2024-08-24  v0.7.0   *** Embedded Typst via Rust crate + cxx::bridge ***
2024-10-09  v0.7.1   Experimental macOS support (still Qt shell)
2024-11-15  v0.8.0   Typst 0.12, autocomplete, incremental compilation
2025-02-07  v0.8.1   Allowed paths, scaling fix
2025-03-30  v0.9.0   Typst 0.13, auto BiDi isolation, Windows native spellcheck
2025-06-03  v0.10.0  Outline, go-to-definition, visible BiDi controls
2025-08-14  v0.11.0  Symbol/color picker, labels panel
2025-10-11  v0.11.1  Bugfix
2025-12-10  v0.12.0  Typst 0.14, advanced export dialog, Qt 6.8 baseline
2026-03-20  --       macshell development begins
2026-04-08  --       macshell becomes default on macOS
2026-04-23  --       Last upstream Katvan commit before the fork
2026-05-07  --       *** Fork: Badry Darkoush renames the project to Katoub ***
2026-05-16  --       AI-assistant + developer documentation added
```
