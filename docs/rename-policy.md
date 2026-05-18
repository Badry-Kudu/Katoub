# Katvan to Katoub Rename Policy

This document defines how the fork should be renamed from Katvan to Katoub without breaking packaging, user data, attribution, or GPLv3 compliance.

Katoub is a fork of Katvan. The rename must separate product identity from upstream provenance. User-facing identity should move toward Katoub. Historical attribution, license notices, and upstream references must remain accurate.

## License and copyright requirements

Katoub must continue to comply with the GNU General Public License version 3, as distributed in `COPYING`.

Do not remove, rewrite, or weaken existing Katvan copyright notices, upstream author names, or license headers.

When a file is materially updated for Katoub, add an additional Katoub copyright notice while preserving the existing upstream notice. Use the form below unless a file already follows a different comment style:

```text
Copyright (C) 2026 Badry Darkoush
```

For source files, add the notice to the existing license header. Do not add a Katoub copyright notice to a file that was not materially changed. Mechanical rename-only changes may include the notice only if the file is being intentionally adopted as part of Katoub-maintained code.

Every newly created source, script, or documentation file should include GPLv3-compatible licensing information when practical. Documentation files may include a short statement such as:

```text
This file is part of Katoub and is distributed under the GNU General Public License v3.0 or later.
```

## Rename categories

### 1. Safe to rename now

These are user-facing identity changes. They are low risk and should be handled first.

| Area | Current file or identifier | Target | Notes |
|---|---|---|---|
| README product name | `README.md` | Katoub | Keep upstream Katvan attribution where historically relevant. |
| README screenshots | `README.md` | Katoub screenshots | Replace `katvan.app` screenshots when Katoub screenshots exist. |
| README release links | `README.md` | Katoub release links | Existing `IgKh/katvan` links should remain only when explicitly pointing to upstream. |
| About dialog text | source file containing about text | Katoub | Preserve upstream credits. |
| Window title | source file defining application display name | Katoub | User-visible only. |
| Desktop launcher visible name | `assets/katvan.desktop` | `Name=Katoub` | File rename belongs to phase 2. |
| AppStream visible name and summary | `assets/app.katvan.Katvan.metainfo.xml.in` | Katoub visible text | App ID rename belongs to phase 2 or 3. |
| Icons shown to users | `assets/katvan.svg`, `assets/katvan-48.png`, `assets/katvan-256.png` | Katoub artwork | Preserve old icon files temporarily if packaging still references them. |
| Translation-visible strings | `*.ts` translation files | Katoub | Do not alter translated historical references unless user-visible branding requires it. |
| Future changelog headings | `CHANGELOG.md` | Katoub | Do not rewrite old upstream release history. |

### 2. Safe with coordinated build and packaging edits

These names are safe to change, but only as a single packaging-focused change set. Changing one without the others can break builds or installation.

| Area | Current file or identifier | Target | Required coordination |
|---|---|---|---|
| CMake project name | `CMakeLists.txt`: `project(katvan ...)` | `project(katoub ...)` | Check generated package names and CMake variables. |
| Main executable target | `CMakeLists.txt`: `katvan` target references | `katoub` | Update all `install(TARGETS katvan ...)` and dependent target references. |
| Linux binary name | installed target `katvan` | `katoub` | Consider compatibility symlink `katvan -> katoub` for at least one release. |
| Runtime library path | `CMakeLists.txt`: `${CMAKE_INSTALL_LIBDIR}/katvan` | `${CMAKE_INSTALL_LIBDIR}/katoub` | Update RPATH and install destination together. |
| Desktop file path | `assets/katvan.desktop` | `assets/katoub.desktop` | Update `install(FILES assets/katvan.desktop ...)`. |
| SVG icon file | `assets/katvan.svg` | `assets/katoub.svg` | Update desktop file icon key and install rule. |
| PNG icon files | `assets/katvan-48.png`, `assets/katvan-256.png` | `assets/katoub-48.png`, `assets/katoub-256.png` | Update install rules and icon rename targets. |
| AppStream template filename | `assets/app.katvan.Katvan.metainfo.xml.in` | `assets/app.katoub.Katoub.metainfo.xml.in` | Only if the application ID is intentionally changed. |
| Generated metainfo file | `app.katvan.Katvan.metainfo.xml` | `app.katoub.Katoub.metainfo.xml` | Coordinate with AppStream/Flatpak identity. |
| Release artifacts | files containing `katvan` in artifact names | files containing `katoub` | Update CI and release scripts together. |
| Windows executable | `katvan.exe` | `katoub.exe` | Update installer, shortcuts, portable-mode docs, and upgrade behavior. |
| macOS bundle path | `Katvan.app` | `Katoub.app` | Update bundle metadata, signing, install paths, and docs together. |

### 3. Complicated to rename

These affect saved settings, code structure, platform identity, or upstream mergeability. Rename only after deciding that Katoub will diverge independently from Katvan.

| Area | Current pattern | Why complicated |
|---|---|---|
| C++ namespaces and classes | `Katvan*`, `katvan::*` if present | Large code churn, high upstream merge conflict risk. |
| Source filenames | `katvan_*.cpp`, `katvan_*.h` if present | Requires build-system, include, and test updates. |
| Qt settings identity | application or organization names set in code | May move user settings to a new location. Requires migration. |
| Settings keys | keys containing `katvan` | Renaming may lose existing user preferences. |
| Recent files/cache paths | directories containing `katvan` | Requires migration or compatibility fallback. |
| Flatpak/AppStream ID | `app.katvan.Katvan` | Changing identity can make package managers treat Katoub as a separate app. |
| Windows installer identity | product name, upgrade code, registry paths | Incorrect rename can break upgrade/uninstall behavior. |
| macOS bundle identifier | identifier containing `katvan` | Affects signing, preferences, updates, and user data paths. |
| CI workflow internals | scripts assuming `katvan` | Rename with release validation only. |
| Test snapshots | expected strings containing Katvan | Update only when testing user-visible Katoub behavior. |

### 4. Should not be renamed

These preserve legal, historical, and upstream accuracy.

| Area | Keep Katvan? | Reason |
|---|---:|---|
| Existing copyright notices | Yes | GPLv3 compliance and attribution. |
| License text in `COPYING` | Yes | The GPL text must remain intact. |
| Historical changelog entries | Yes | They describe upstream history. |
| Upstream author credits | Yes | Fork attribution must remain accurate. |
| URLs intentionally pointing to upstream | Yes | They identify the original project. |
| Comments explaining upstream behavior | Usually yes | Avoid meaningless churn. |
| Git history, old tags, old release names | Yes | Historical integrity. |
| Third-party package references | Yes | Do not imply Katoub ownership. |
| Compatibility aliases | Yes | Useful during transition. |
| Internal upstream-aligned names | Usually yes | Reduces rebase and merge conflicts. |

## Recommended phases

### Phase 1: user-visible identity

Update documentation, UI display strings, about dialog, visible desktop name, screenshots, and visible AppStream text. Keep explicit upstream references intact.

Expected files include:

- `README.md`
- `CHANGELOG.md` for future Katoub-only entries
- `assets/katvan.desktop`
- `assets/app.katvan.Katvan.metainfo.xml.in`
- translation files such as `*.ts`
- source files containing the application title, about dialog, and settings UI strings

### Phase 2: packaging identity

Rename coordinated packaging artifacts and build targets. Validate Linux, Windows, and macOS builds after this phase.

Expected files include:

- `CMakeLists.txt`
- `assets/katvan.desktop`
- `assets/katvan.svg`
- `assets/katvan-48.png`
- `assets/katvan-256.png`
- `assets/app.katvan.Katvan.metainfo.xml.in`
- installer, CI, or release scripts if present

Add a compatibility launcher or symlink from `katvan` to `katoub` when practical.

### Phase 3: application identity and migration

Decide whether Katoub is a hard fork with a new app identity or a compatible continuation.

If Katoub receives a new identity, add first-run migration from Katvan settings, cache, recent files, and package data paths where applicable.

If Katoub remains compatible with Katvan identity paths, document that old internal names are intentionally retained.

### Phase 4: internal code rename

Rename classes, namespaces, filenames, and tests only after upstream rebasing is no longer a priority. This phase is optional and should not block product branding.

## Commit discipline

Keep rename commits narrow:

1. Documentation rename.
2. UI-visible rename.
3. Packaging rename.
4. Settings/app identity migration.
5. Internal code rename, if ever needed.

Avoid mixing functional changes with rename-only commits. Rename-only commits should be easy to review mechanically.

## Review checklist

Before merging a Katvan to Katoub rename change, verify:

- Existing Katvan copyright notices remain intact.
- Badry Darkoush copyright notice is added to materially updated Katoub-maintained files.
- GPLv3 terms remain preserved and referenced.
- Upstream attribution remains clear.
- User-visible strings say Katoub where appropriate.
- Historical records still say Katvan where appropriate.
- Linux desktop, AppStream, icon, and binary names are consistent.
- Windows and macOS package names are consistent if touched.
- Existing settings and cache are either migrated or intentionally preserved.
- Build and packaging scripts pass after coordinated renames.
