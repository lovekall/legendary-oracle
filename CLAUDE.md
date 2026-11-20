# CLAUDE.md — WrithDeck Android

Instructions for Claude Code in this repository.

---

## Architecture

Pure Kotlin + Jetpack Compose. No Tcl/JNI engine. All logic is in Kotlin:
- INI parsing/writing: `IniParser` in `AppConfig.kt`
- State persistence: `StateStore` in `StateStore.kt`
- Color schemes: `ColorSchemes.kt` — 9 built-in schemes
- Business logic + UI state: `WrithdeckViewModel.kt`
- UI: Jetpack Compose screens in `ui/`

---

## Build

```sh
# From writhdeck-android/
./gradlew assembleDebug
# -> app/build/outputs/apk/debug/writhdeck-debug.apk
```

### Auto-incrementing version (`app/build.gradle.kts`)

`versionCode`/`versionName` are computed at configuration time as `"YYYY-MM-DDx"` (e.g. `2026-06-07a`, then `2026-06-07b`, …) — `x` is a spreadsheet-column-style letter suffix (`a`…`z`, `aa`, `ab`, …) that increments per build on the same day. State is persisted in `app/version.properties` (`date=`/`letter=`, auto-generated, gitignore-able, do not edit by hand). `versionCode` is the numeric encoding `YYYYMMDD * 100 + (letterIndex + 1)` (e.g. `2026060701`).

The letter only advances when the requested Gradle tasks contain `assemble`/`bundle`/`install` (checked via `gradle.startParameter.taskNames`) — IDE syncs, `lint`, `test`, etc. don't burn through the alphabet.

**Why:** a static `versionCode = 1` made Android's package manager sometimes treat a reinstall as a no-op (binary not actually replaced — "it says updated but nothing changed", requiring an uninstall first to see changes). A unique, monotonically increasing `versionCode` per build forces a real replace. **How to apply:** if you ever see a stale APK behavior again, also try `./gradlew clean assembleDebug` — Gradle's incremental up-to-date check can occasionally report `compileDebugKotlin UP-TO-DATE` right after a source edit (observed 2026-06-07; `--rerun-tasks`/`clean` forces a true recompile).

---

## Key files

| File | Role |
|---|---|
| `app/src/main/java/com/writhdeck/app/AppConfig.kt` | `AppConfig` + `IniParser` (parse/write/patchKeys/patchProfileKey) + `ThemeColors` |
| `app/src/main/java/com/writhdeck/app/StateStore.kt` | `AppState` + hand-rolled JSON load/save + path migration |
| `app/src/main/java/com/writhdeck/app/ColorSchemes.kt` | `SchemeColors` + `BUILTIN_SCHEMES` map (9 schemes) |
| `app/src/main/java/com/writhdeck/app/Fonts.kt` | `FontManager` — user `.ttf`/`.otf` fonts in `fontsDir`, typeface resolution |
| `app/src/main/java/com/writhdeck/app/WrithdeckViewModel.kt` | ViewModel: StateFlows, timer, docs, favorites, config, autosave, workspaces |
| `app/src/main/java/com/writhdeck/app/MainActivity.kt` | Entry point, system dark mode, storage permission |
| `app/src/main/java/com/writhdeck/app/ui/BrowserScreen.kt` | File browser: scratchpad item, keyboard shortcuts |
| `app/src/main/java/com/writhdeck/app/ui/EditorScreen.kt` | Editor: AndroidView(FlingEditText), syntax spans, TOC, command mode, workspace toggle |
| `app/src/main/java/com/writhdeck/app/ui/SchemeConfigScreen.kt` | Color scheme selector + custom scheme editor |
| `app/src/main/java/com/writhdeck/app/ui/SettingsScreen.kt` | Settings: tabbed (Profile/Display/Fonts/Schemes/Timer/Misc) |
| `app/build.gradle.kts` | Gradle config (no NDK, no Tcl dependencies) |

---

## Critical patterns

### INI parsing

`IniParser.parse()` is section-aware, format identical to Tcl/JS: `[section]` headers (regex `^\[(\w+)\]$`). `[profiles]`/`[schemes]` are containers followed by named `[<name>]` sub-sections; `TOPLEVEL_SECTIONS` (union of Tcl/web top-level sections + Android's `status_bar`) disambiguates a `[<name>]` seen inside `[profiles]`/`[schemes]` (top-level name → closes the container; otherwise → names a profile/scheme). `profile = <name>` inside `[editor]` selects which profile's keys override the globals.

`IniParser.migrateLegacyFormat(text)` / `migrateProfileScopedKeys(text)` are one-shot, lossless, idempotent migrations run in `initApp()`: the former converts the legacy `= profile: <name> =`/`active_profile` dialect to the `[section]` format above; the latter moves `PROFILE_SCOPED_KEYS` values from global scope into each `[profiles] -> [<name>]` sub-section (without overwriting a profile that already defines the key).

`IniParser.write()` generates a complete template with:
- Comment listing all 9 schemes
- `[profiles]` containing `[default]` and `[novel]` sub-sections (12 per-profile keys each — see below)
- `profile = default` inside `[editor]`

`IniParser.patchKeys()` patches key=value pairs anywhere in the INI (global keys like `font_bold`).

`IniParser.patchProfileKey(text, profile, key, value)` patches a key only inside a `[profiles] -> [<profile>]` sub-section — used for the 15 per-profile keys (`PROFILE_SCOPED_KEYS` ∪ the original 6: `scheme`, `heading_marker`, `markdown_support`, `margin_width`, `margin_height`, `word_goal`, plus `font_size`, `font_family`, `line_spacing`, `line_numbers`, `android_dark_mode`, `block_cursor`, `status_left`, `status_center`, `status_right`) to avoid clobbering values in other profiles/sections. The status bar zones joined `PROFILE_SCOPED_KEYS` on 2026-09-16, aligning with the Tcl desktop (where the same move exposed a real bug — see the status-bar section below and `writhdeck/.claude/CLAUDE.browser.md`).

`IniParser.writeCustomScheme(text, name, colors)` / `removeSchemeSection(text, name)` manage `= scheme: name =` sections for custom schemes.

`initApp()` creates `writhdeck.ini` with default content on first launch so it always appears in the documents list.

### State persistence (`StateStore`)

Hand-rolled JSON, compatible with the desktop `.writhdeck.json` format. Path migration on load converts old Tcl-normalized paths:

```kotlin
// Tcl `file normalize` used to convert /storage/emulated/0/ → /data/media/0/ (unreadable)
private fun migratePath(path: String): String =
    if (path.startsWith("/data/media/0/"))
        "/storage/emulated/0/" + path.removePrefix("/data/media/0/")
    else path
```

Applied in `load()` to cursors keys, favorites list, recent list, and daily map keys.

### Cursor restore

`openFile()` sets `_initialCursorOffset` **before** `_currentFile` so that `remember(currentFile?.path, wsActive)` in EditorScreen reads the correct value during composition. `saveCursor(offset)` converts char offset → (cy, cx) via `textOffsetToLinecol` and persists to `StateStore`.

**Cursor/daily-stats key mode (`AppConfig.cursorKeyMode`, Settings > Misc, 2026-09-14)** — user-reported bug: reopening a document didn't restore its position. Investigation found `openFile()`'s own path-keyed lookup logically sound (verified by code reading — no device was connected this session to pull `.writhdeck.json` and confirm empirically), but found a real, confirmed gap: **`openExternalContent()`** (opening a document via `ACTION_VIEW`/"Open with"/a file manager rather than the in-app browser) always reset `_initialCursorOffset` to 0, never consulting `appState.cursors` even when the `content://` URI resolved to a real local path already tracked in `recent` — fixed to look up/restore exactly like `openFile()`. Separately, added the requested switchable key mode: `"path"` (default, unchanged — keyed by the note's absolute file path, so a moved/renamed note loses its position/history) vs `"name"` (keyed by the bare filename only — survives a move/rename into another folder, at the cost of two different notes sharing a filename in different folders then sharing one saved position/history). `WrithdeckViewModel.statKey(path)` is the single lookup point consulted by `saveCursor`/`openFile`/`openExternalContent`/`getDailyStats`/every `StateStore.updateDaily` call — **applies to both `cursors` and `daily`** (the per-file word-count history, follow-up request the same day extending the feature beyond cursor position). `renameFile`/`deleteFile` remap/drop both maps together via the new `StateStore.renameStatKey`/`removeStatKey` (kept separate from `renamePath`/`removePath`, which stay path-keyed unconditionally for favorites/recent only — those never had a "by name" option requested). **Note**: the ini already had a decorative, never-wired-up `cursor_restore = yes` key (default template only, no `AppConfig` field, no reader) predating this round — left as-is (unrelated to this fix, out of scope to repurpose without a separate decision; the same class of dead setting was also found and fixed in writhdeck-web's `cursorRestore`, see its own CLAUDE.md).

### AndroidView EditText (editor)

The editor uses `AndroidView { FlingEditText }` (a private `EditText` subclass — not `BasicTextField`). This uses Android's `DynamicLayout` which only renders visible lines, making it handle large files (500K+ chars) efficiently.

**`FlingEditText`** — adds momentum/fling scrolling. A plain multi-line `EditText` scrolls 1:1 with the drag and stops dead on release (no inertia), unlike `ScrollView`/most text editors and browsers. Wrapping in `ScrollView` would add fling but forces full-content-height measurement, defeating `DynamicLayout`'s visible-lines-only rendering on large files — so fling is implemented by driving the `EditText`'s own `scrollY` with an `OverScroller` + `GestureDetector.onFling`, exactly like `ScrollView` does internally (`onTouchEvent` aborts any running scroll on `ACTION_DOWN`, detects fling, `computeScroll()` advances the animation via `postInvalidateOnAnimation`). Guards `hasSelection()` so flicking after a text-selection drag doesn't trigger a spurious scroll.

**Key state refs** (captured by factory closure, updated by `update {}` block):
- `ignoreTextChange: booleanArrayOf(false)` — prevents TextWatcher feedback loop when `setText()` is called programmatically
- `keyHandlerRef: Array<(Int, AKeyEvent) -> Boolean>` — updated each recomposition to close over current compose state; factory calls `keyHandlerRef[0](keyCode, event)`
- `originalKeyListener: Array<KeyListener?>` — stored after `inputType` is set; restored/nulled in `update {}` for read-only toggle
- `editorRef: MutableState<EditText?>` — set in `factory { }.also { editorRef.value = it }`

**File/workspace change detection** — `editText.tag = "${currentFile?.path}:${wsActive}"`. In `update {}`, if tag differs → `ignoreTextChange[0] = true; setText(content); setSelection(liveCursor); tag = key; ignoreTextChange[0] = false`. This is the ONLY place `setText()` is called; all other recompositions skip it.

**TextWatcher** — `afterTextChanged` calls `vm.updateLiveCursor(selectionStart)` + `vm.updateContent(text)`. Guards with `if (ignoreTextChange[0]) return`.

**Syntax highlighting** — `LaunchedEffect(content, headingMarker, markdownSupport, commentMarker, boldMarker, italicMarker, underlineMarker, strikethroughMarker, hdColorInt, cmtColorInt, markupColorInt)` with `delay(300)` debounce + `withContext(Dispatchers.Default)`. Applies via marker span subclasses (`SyntaxHeadingSpan`, `SyntaxCommentSpan`, `SyntaxMarkupSpan`) on the Editable. `setSpan()` does NOT trigger TextWatcher — no guard needed.

`computeSyntaxSpans` takes all 5 inline/comment markers as configurable `String` params (mirrors Tcl/web, see "Configurable markup markers" below) — comment detection is `commentMarker.isNotEmpty() && text.startsWith(commentMarker, ts)` (anchored after leading spaces via `ts`), and inline markup is one combined `Regex` built by alternating `Regex.escape(marker) + ".+?" + Regex.escape(marker)` for each non-empty marker among bold/italic/underline/strikethrough — all four map to the single `markupColorInt`/`SyntaxMarkupSpan` (Android only has one `markupColor` per scheme, like the web version's single `hl-markup` CSS class; Tcl's 4-separate-colors model was NOT replicated).

**Search spans** — `LaunchedEffect(searchMatches, findMatchIndex)` runs on main dispatcher, directly calls `editable.setSpan(SearchBgSpan/SearchCurrentBgSpan/SearchCurrentFgSpan, ...)`. Then `editText.setSelection(match)` to scroll.

**Undo/redo** — native Android EditText handles Ctrl+Z automatically (API 23+, minSdk=26). No `ArrayDeque` needed.

**Format menu (≡ → Format)** — three pure functions, ports of `editor.js`'s `applyHeading`/`applyLineMarker`/`applyInlineMarker`, all `(text, selStart, selEnd, marker[, level]): Triple<String, Int, Int>?` (`null` if marker empty):
- `applyHeadingLevel(text, selStart, selEnd, marker, level)` — H1/H2/H3, repeats `marker` ×`level`, detects/replaces existing heading level.
- `applyLineMarkerResult(text, selStart, selEnd, marker)` — Comment, toggles `marker` at the start of each selected line.
- `applyInlineMarkerResult(text, selStart, selEnd, marker)` — Bold/Italic/Underline/Strike, wraps/unwraps the selection with `marker...marker`, or inserts `marker+marker` with cursor between if no selection.

Apply via shared helper `applyEditResult(r: Triple<String, Int, Int>?)`: `ignoreTextChange[0] = true; editText.setText(newText); editText.setSelection(...); ignoreTextChange[0] = false; vm.updateContent(newText)`.

**TOC navigation** — `editorRef.value?.setSelection(entry.charOffset)`. TOC is a `ModalBottomSheet` (`showToc`), not a persistent panel like Tcl/web's pinned side panel — `tocPinned` only controls whether selecting a chapter closes the sheet (`if (!tocPinned) showToc = false`), it doesn't keep the sheet open while editing continues (editing is blocked behind a modal sheet anyway). **Opens preselected on the current chapter (2026-08-23)**: `tocFocusedIndex` (previously always `-1` on open, only ever set by keyboard Up/Down nav) is now initialized to `toc.indexOfLast { it.charOffset <= vm.liveCursor }` in the `remember(showToc) { ... }` block, and the initial `LaunchedEffect(showToc)` scrolls to it (`tocListState.scrollToItem`) alongside the existing focus-requester call. One-time snapshot at open time, not live-updating — unlike Tcl/web's persistent-panel highlight, there's no "while you keep editing" case to track since the sheet is modal. Reuses the same visual treatment as keyboard-nav focus (`colorScheme.primary.copy(alpha = 0.12f)` background) rather than a separate style — arrow-key navigation away from the preselected chapter simply moves `tocFocusedIndex` like any other nav step, no extra state needed (unlike Tcl/web, which do need a separate "current" vs "keyboard-focused" concept since their panels stay open during editing).

**Repetitions (`RepetitionEntry`/`getRepetitions()` in `WrithdeckViewModel.kt`, dialog in `EditorScreen.kt`, 2026-08-23)** — new feature (didn't exist on Android before; ported from `writhdeck-web`'s `findRepetitions`, itself close to the Tcl `find-repetitions` level-1 tier). Same word repeated within `REPETITION_SCOPE` (100) words of each other, 3+ chars, EN+FR stopword list (`REPETITION_STOPWORDS`) — no hidden-substring tier (level 2 in Tcl), matching the web port which doesn't expose it via settings either. Reached from the `⋮` menu ("Repetitions", next to "Word frequency" — both `BuildConfig.FULL_VERSION`-gated, unlike Tcl/web where it's nested as a button inside the Structure/Analyse dialog; kept as Android's own established convention of one independent dialog per analysis tool, see "Word frequency" menu item right above it). Tapping a row jumps (`editorRef.value?.setSelection(rep.offset1)`) and closes the dialog (AlertDialog is modal, unlike web's non-modal draggable `rep-dlg`).

**Open-time centering** — same idea as the Tcl/web ports: `bestRepIndex` picks the row whose `line1` is closest to `vm.liveCursor`'s line. Centering itself needs two passes since Compose doesn't have a `scrollIntoView({block:'center'})` equivalent: `repListState.scrollToItem(bestRepIndex)` first (brings it into view, item lands at the top), then a corrective `scrollToItem(bestRepIndex, -target)` using `layoutInfo.viewportSize.height`/the now-known item `size` from `visibleItemsInfo` to actually center it — the item's height isn't known until after the first scroll triggers layout.

**File info word count** — `editor_dialog_file_info_words_no_comments` row (`countWordsNoComments(content, commentMarker)`, top-level fun in `EditorScreen.kt` near `computeSectionAnalysis`) added next to the existing Words row, 2026-08-23. Skips lines starting with `commentMarker` (same `line.startsWith(...)` convention as the syntax highlighter's own comment detection at line ~357, not the Tcl/web "line-start after config, no whitespace-skip" nuance — they already match here). `content`/`commentMarker` are both live Compose state (`vm.content`/`vm.commentMarker`), so — unlike the web port, which had a pre-existing staleness bug in the equivalent dialog (`doc.content` not synced until save) that had to be fixed alongside — no such fix was needed here.

**Cursor save** — `doBack()` and `DisposableEffect.onDispose` read `editorRef.value?.selectionStart` directly.

### IME / keyboard in BrowserScreen

The invisible `BasicTextField` (1dp, alpha 0) captures hardware keyboard shortcuts. It must never auto-show the IME on navigation return from the editor.

Pattern (`imeAllowed` flag):
```kotlin
var imeAllowed by remember { mutableStateOf(false) }

// Keyboard icon button:
if (imeVisible) {
    imeAllowed = false
    keyboardController?.hide()
} else {
    imeAllowed = true
    focusRequester.requestFocus()
    keyboardController?.show()
}

// BasicTextField modifier:
.onFocusChanged { fs ->
    if (fs.isFocused && !imeAllowed) keyboardController?.hide()
}

// Each onOpenFile call:
imeAllowed = false; onOpenFile(entry)
```

### Color themes

`AppConfig.themeColors(useDark: Boolean): ThemeColors` returns `ThemeColors(bg, fg, headingColor)` from the active scheme. `ThemeColors` is defined in `AppConfig.kt`.

`applyConfig()` calls `config.themeColors(resolveUseDark())` and pushes to `_themeColors`. `updateThemeColors(systemDark)` is called by `MainActivity` when the system dark mode changes.

### Config reload

After saving `writhdeck.ini` from the editor: `reloadConfig()` re-parses the file, updates `config`, calls `applyConfig()` to push all affected StateFlows.

### Read-only files (`fileWritable`)

`_fileWritable: MutableStateFlow<Boolean>` — set in `openFile` (`File.canWrite()`), `openExternalContent` (from intent `canWrite` flag), and `saveFile` (set to `false` on write failure).

EditorScreen behavior when `!fileWritable`:
- Title shows `[read-only]` suffix
- Save button disabled
- AlertDialog on file open
- "Discard changes?" confirmation on back navigation when dirty

### Closing a modified document — `requestClose()`

`requestClose()` (in `EditorScreen`, next to `doBack()`) is the single entry point for "leave this document": `dirty && !fileWritable` → `showDiscardConfirm` ("Discard changes?" — changes can't be saved); `dirty && fileWritable` → `showSaveConfirm` ("Save changes?" with **Save** / **Don't save** / **Cancel** — a 3-way `AlertDialog` using a `Row` of two `TextButton`s in the `dismissButton` slot, since `AlertDialog` only exposes two button slots); otherwise → `doBack()` directly. Wired from the `TopAppBar` back `IconButton`, the system back gesture (`BackHandler { showSaveConfirm = true }` when `dirty && fileWritable && !distractionFree`), and `Ctrl+Q`.

**Why:** previously a dirty+writable document auto-saved silently on exit with no way to discard changes — the user asked for a confirmation with the option not to save ("il faudrait une confirmation pour être sûr (et la possibilité de ne pas le faire)").

**Never use `storagePermissionGranted` to bypass `File.canWrite()`** — that permission doesn't grant access to other apps' private directories (e.g. Termux `/data/data/com.termux/files/`). Files from those paths must be written via `contentResolver.openOutputStream(uri)` when the intent grants `FLAG_GRANT_WRITE_URI_PERMISSION`.

### Margins

No upper limit enforced by the parser — `IniParser.parse()` uses `coerceAtLeast(0)` only. Settings UI caps at 200 dp.

### Autosave

`restartAutosave()` cancels/restarts a coroutine loop (`delay(interval * 60_000L)`) that writes `autosave_ws0N.txt` in `docsDir`. Format: `"$name\n$ts\n\n-------------------------\n$content"`. Triggered by `applyConfig()` when autosave settings change.

### Second workspace

`WsSnapshot` captures full workspace state (file, content, dirty, wordCount, writable, extUri, cursorOffset). `toggleWorkspace(cursorOffset)` saves current state to `ws1Snap`/`ws2Snap`, restores the other, sets `_wsDualMode=true`. `_wsActive` (1 or 2) and `_wsDualMode` are exposed as StateFlows. Status bar shows `[1]`/`[2]` prefix when `wsDualMode` is true.

`WrithdeckViewModel.buildStatusSegment(tokens, cursorCc, cursorWc)` renders `statusLeft/Center/Right` token-by-token (unknown tokens pass through as literal text - lets users put separators like `|` directly in a zone string): `ws filename dirty words chars goal timer clock readtime pages`, plus `cursor_percent`/`cursor_chars`/`cursor_words` (added 2026-09-16, mirroring the Tcl desktop and TUI's ctrl+h "Cursor position" stat) — words-before-cursor/total-words as a rounded percentage (0% on an empty document or at the very start, 100% at the very end), and the raw char/word counts from the start of the document up to the cursor. `liveCursor` (a plain `var`, not a StateFlow - see below) is the character offset used for this; `refreshStatus()` only scans up to it (`countWords` on a substring) when at least one zone actually contains one of the three cursor tokens (`needsCursorStats`), to avoid that scan on every keystroke/cursor-move on a large document otherwise (see the keyboard-perf notes below). `liveCursor` is kept current on pure cursor movement (arrow keys, tap, drag - no text change) via `FlingEditText.selectionChangeListener` in `EditorScreen.kt` calling `vm.updateLiveCursor(offset)`, which now also calls `refreshStatus()` - previously that listener only drove typewriter-mode re-centering, so these three tokens would go stale until the next keystroke.

### Scratchpad

`openScratchpad()` is an **in-memory ephemeral buffer** — no file on disk. Sets `_isScratchpad = true`, `_fileWritable = false`, `_content = ""`, `_currentFile = DocEntry("scratchpad", "")` (empty path sentinel), cursor at 0. Matches Tcl's `open-scratchpad` concept: `::filename = ""`, `::scratchpad = 1`, always starts empty. `saveFile()` is a no-op for scratchpad; user must use "Sauvegarder sous…" to persist content to a real file (after which `_isScratchpad` becomes `false`). BrowserScreen shows it as a permanent first item with `t` shortcut. `_isScratchpad` is saved/restored in `WsSnapshot` for workspace toggle. EditorScreen skips the "[read-only]" label and the read-only alert when `isScratchpad` is true (even though `fileWritable = false`).

### Subfolder navigation (`browser_subdirs`)

Explorer-style browsing of subfolders inside the documents folder. Gated by `config.browserSubdirs` (INI `browser_subdirs`, default `yes`; Misc "Browse subfolders" toggle in `SettingsScreen`). Mirrors the Tcl desktop's optional subfolder navigation.

- **VM state**: `_browserDir: MutableStateFlow<String?>` (null = root dual `docsDir`/`configDir` view; a path = single-folder view), `_subdirs: MutableStateFlow<List<DocEntry>>` (immediate subfolders, `backups` and dotfolders skipped). `enterDir(path)` / `browserUp()` navigate (up returns to root when the parent is `docsDir`/`configDir`). `refreshDocs()` branches on `_browserDir`; `createFile()` creates in the current folder; `applySettings()` resets `_browserDir` to null.
- **UI** (`BrowserScreen`): at root, a "Folders" section lists `subname/` rows (`FolderNavItem`); inside a subfolder, a breadcrumb header + ".." + folders + files (no Favorites/Recent). Tapping a folder calls `enterDir`; `..` and the system back button (via `BackHandler(enabled = inSubdir)`) call `browserUp`. Folders are tap-only; arrow-key nav (`navEntries`/`navItemIndex`) covers files and accounts for the folder rows.

### Color schemes

`SchemeColors` has 16 fields (8 dark + 8 light). `BUILTIN_SCHEMES` has 9 built-in schemes (the 9th, `eink`, see below). `AppConfig.customSchemes: Map<String, SchemeColors>` holds INI-parsed `[schemes] -> [<name>]` sub-sections; `schemeColors()` checks custom first, then builtins. `SchemeConfigScreen` lists all schemes with color swatches, lets user select the active scheme, edit any scheme (builtin or custom), or create new custom schemes. Custom schemes are persisted via `IniParser.writeCustomScheme`.

### Settings screen

`SettingsData` data class mirrors the user-configurable fields of `AppConfig`. `getSettingsData()` reads current config; `applySettings(s)` calls `config.copy(...)`, `applyConfig()`, then persists fields via `IniParser.patchKeys` (global, e.g. `font_bold`) and `IniParser.patchProfileKey` for the per-profile keys (`scheme`, `heading_marker`, `markdown_support`, `margin_width`, `margin_height`, `word_goal`, `font_size`, `font_family`, `line_spacing`, `android_dark_mode`, `block_cursor`, `status_left`, `status_center`, `status_right`), scoped to `config.activeProfile`. `line_numbers` is per-profile too but toggled through its own dedicated quick-action, not through this `SettingsData` round-trip. `newProfile(name, s)` (Profile tab, "New...") writes the same per-profile keys into a fresh `[profiles] -> [<name>]` section from whatever `s` is showing at the time - i.e. the new profile starts as a snapshot of the current settings, status bar zones included since 2026-09-16 (previously `status_left/center/right` were global-only, so a new profile silently inherited whatever zones happened to be configured rather than getting its own copy - now editing one profile's status bar can no longer leak into another's, same fix as the Tcl desktop's).

`SettingsScreen` is tabbed via `ScrollableTabRow`/`Tab`, mirroring the desktop Tcl/Tk config dialog's tab set and order (`SETTINGS_TABS`): **Profile** (margins, word goal, line spacing), **Display** (heading marker, status bar zones), **Fonts** (font size, font family, bold, live preview), **Schemes** (scheme dropdown + "Edit scheme colors" → `SchemeConfigScreen`), **Timer** (type, duration, sound, alert, status bar display), **Misc** (Hemingway mode, autosave, "Edit INI directly"). Each tab is a private `@Composable` taking `(s: SettingsData, onChange: (SettingsData) -> Unit)`. `selectedTab` is local `mutableIntStateOf` — not persisted.

### Editor font family

Android has no portable font-enumeration API (unlike Tk's `font families`), so the Fonts tab offers a fixed list of **8 built-in generic font-family aliases** (`EDITOR_FONTS` in `AppConfig.kt`) — a mix of monospace, serif and sans-serif faces (`monospace`, `serif-monospace`, `sans-serif`, `sans-serif-condensed`, `sans-serif-light`, `sans-serif-medium`, `serif`, `casual`). These resolve via `Typeface.create(familyName, Typeface.NORMAL)` against the system `fonts.xml` — **no font files are embedded** (avoids APK bloat and font-licensing concerns; guaranteed present on AOSP).

**User fonts** — in addition to the 8 aliases, the user can drop `.ttf`/`.otf` files into a `fonts/` subfolder and they appear in the Fonts tab. Two locations are scanned (`WrithdeckViewModel.fontDirs`): the primary `configDir/fonts` (`fontsDir`, next to `writhdeck.ini`, auto-created on first run) and — when a custom documents folder is set (`docs_dir`) — `docsDir/fonts` (scanned if present, not auto-created, so the user's documents folder isn't cluttered). `configDir/fonts` wins on a filename collision (listed first). `FontManager` (`Fonts.kt`) handles them: `listUserFonts(fontDirs)` scans+merges+dedups (by filename) into `EditorFont(label = nameWithoutExtension, familyName = filename)`; `WrithdeckViewModel.refreshUserFonts()` publishes them to the `userFonts: StateFlow<List<EditorFont>>` (refreshed at startup and via a `LaunchedEffect` each time `SettingsScreen` opens). A user font is identified in the INI by its bare filename (e.g. `font_family = JetBrainsMono.ttf`) — `FontManager.isUserFont()` distinguishes it from a generic alias by the `.ttf`/`.otf` extension, and `IniParser.parse()` accepts it on that basis. The Fonts tab merges `EDITOR_FONTS + userFonts` in the dropdown and lists the scanned `fontDirs` paths as a hint.

**Typeface resolution** — all `Typeface.create(family, style)` call sites (the `EditText` factory + style `update {}` and the line-numbers gutter in `EditorScreen.kt`, plus `FontPreview`/dropdown previews in `SettingsScreen.kt`) go through `FontManager.resolveTypeface(fontDirs, familyName, style)`: for a user font it `Typeface.createFromFile()`s the first of `fontDirs` holding that filename (cached by filename, style synthesised on top via `Typeface.create(base, style)`) and otherwise falls back to the system `Typeface.create(familyName, style)`. A missing/invalid file degrades gracefully to the default.

Persisted as `font_family` (+ `font_bold` boolean) in the INI (patched via `IniParser.patchProfileKey`/`patchKeys`). Exposed as `WrithdeckViewModel.fontFamily: StateFlow<String>` / `fontBold: StateFlow<Boolean>`. `EditorScreen` applies the resolved typeface to the `EditText` — set once in the `factory` block and re-applied in `update {}` only when family or weight changes (guarded by `EditorStyle.fontFamily`/`fontBold` inside the existing `lastStyle[0]` comparison, avoiding layout jank on unrelated recompositions).

**Live preview** — `FontPreview` in `SettingsScreen.kt` renders a sample sentence with the currently selected family/size/weight (`FontFamily(FontManager.resolveTypeface(...))`), so the user sees the effect of each setting immediately, without leaving the Fonts tab.

### Block cursor

The Display tab's "Editor" section has a "Block cursor" `SwitchSettingRow` (`blockCursor` in `AppConfig`/`SettingsData`/`WrithdeckViewModel`, persisted as `block_cursor` — mirrors Tcl/Tk's `block_cursor_gui` and the web version's `blockCursor`, both using the editor foreground colour as cursor colour, e.g. Tk's `-insertbackground $fg`).

Android has no native block-cursor toggle (unlike Tk's `-blockcursor`). `FlingEditText` draws **the entire caret** (both modes) itself by overriding `onDraw()` (`super.onDraw()` first, then the caret on top) — `isCursorVisible` is forced to `false` so the native caret never draws. Configured via `setBlockCursor(enabled, fillColor, textColor)`; `onSelectionChanged`/`onFocusChanged` call `invalidate()` so the caret follows movement and only shows while focused. Drawing is guarded to a collapsed caret (`selectionStart == selectionEnd`); a range selection keeps the normal highlight.

The caret is **confined to the glyph box** — vertically `baseline + paint.fontMetrics.ascent` … `baseline + descent`, **not** `getLineTop/Bottom(line)` — so it never spills into the inter-line gap added by `line_spacing`.

- **Block cursor** (`enabled = true`): width spans from `getPrimaryHorizontal(pos)` to `getPrimaryHorizontal(pos+1)` — i.e. **the exact width of the glyph under the caret** (correct for proportional fonts, narrow chars and spaces), falling back to `paint.measureText("e")` (a representative average-width char, not the widest `"M"` which looked square) at end-of-line / end-of-text / before a `\n` where there's no following glyph on the same line. Filled with `editorFgInt`, then the covered glyph is **re-drawn in `editorBgInt`** on top (inverted look — matches Tk's `-blockcursor` + `-insertbackground $fg`).
- **Normal caret** (`enabled = false`): a **2dp-wide bar** (`caretWidthPx`, `coerceAtLeast(2f)`) in `editorFgInt`, replacing Android's ~1px hairline which was almost invisible at rest.

**Blinking** (`cursor_blink`, global key in `[misc]`, **off by default**; Display > Editor "Blinking cursor" toggle, `cursorBlink` in `AppConfig`/`SettingsData`/`WrithdeckViewModel`/`EditorStyle`): `setCursorBlink(enabled)` drives a `BLINK_MS` (500ms) `postDelayed` loop toggling `caretShown`; `onDraw` skips the caret while `blinkEnabled && !caretShown`. `restartBlink()` (called on selection/focus change) resets `caretShown = true` and re-phases the timer so the caret stays solid right after it moves; `onDetachedFromWindow` removes the callback. When off, the caret is steady and always on (more visible at rest, not less — applies to both block and thin).

**Why this replaced the old approach:** the block cursor was previously a fixed-width `GradientDrawable` set via `EditText.setTextCursorDrawable()` (API 29+ only), sized to `measureText("M")` — too wide on narrow chars/spaces in proportional fonts, and an opaque rectangle that just *hid* the letter under the caret instead of inverting it; the normal caret was the near-invisible native hairline. The `onDraw` approach fixes all of it and works on all API levels (no `Build.VERSION` gate).

Driven from the `lastStyle[0]`-guarded style block in `EditorScreen`'s `update {}` (`EditorStyle` carries `blockCursor: Boolean` plus `fgColor`/`bgColor`) — recomputed only when style actually changes.

### Page up/down zones for e-ink readers (`page_nav_buttons`, 2026-09-20)

Opt-in reading aid: on e-ink readers swipe-scrolling is slow and unreliable, so the editor can show two large, faint tap zones (chevron up/down, 88x72dp each, 40% `editorFg`) stacked at the bottom-right. Modelled on the "next screen" zone of the Kai Chronicles reader (`kaichronicles/www/views/game.html`, `#game-pageturn-zone`: fixed bottom-right, 0.9 viewport step), plus a "previous" zone since proofreading needs to go back.

- Config: global key `page_nav_buttons` in `[misc]` (default `no`), Settings > Display > Editor switch; `pageNavButtons` in `AppConfig`/`SettingsData`/`WrithdeckViewModel` (`_pageNavButtons` StateFlow, patched via `IniParser.patchKeys`). Strings in the 5 languages (`settings_display_page_nav_buttons[_hint]`, `editor_cd_page_up/down`).
- `FlingEditText.pageScroll(direction)`: instant `scrollTo` (no smooth animation, it would only smear on e-ink), step = 90% of the visible height snapped to a line boundary (down: line at-or-above the target, up: line at-or-below, so the overlap never drops under 10%). The caret follows to the first visible line only when the turn left it off-screen, so the saved cursor position (restored on reopen) tracks where the user has read to. `setSelection` fires `selectionChangeListener` (status bar percent refresh).
- **"Where the previous page ended" rule** (2026-09-20): page turns are not line-exact, so after each turn `FlingEditText` draws a thin (1.5dp) horizontal rule in a mid grey (`currentTextColor` at 50% alpha, deliberately subtle) at the boundary of what was on screen before: under the last visible line (baseline on screen, so a line whose line_spacing gap is cut still counts; a first version tested the line box and was one line too high, seen on device) when going down, under the line just above the old view when going up (`pageMarkOffset` = start offset of that line, drawn in `onDraw` between the glyphs and the next line box; both positions stay inside the new view thanks to the 10% overlap). Not moved when the turn did nothing (already at top/bottom). Cleared by any touch on the editor (`onTouchEvent` ACTION_DOWN, so a fling/tap ends it), any text change (`onTextChanged`), and when the option is switched off; the Compose overlay buttons are separate views so pressing them keeps it.
- **Hardware keys** (same option): PageUp/PageDown and the volume rocker (VolumeDown = next, VolumeUp = previous; many e-ink readers wire their side buttons to these) call the same `pageScroll()` from the `keyHandlerRef` at the top of the handler. ACTION_DOWN *and* ACTION_UP are consumed (else the system volume UI still shows), key repeat is ignored (holding must not race through the document), skipped in command mode / typewriter mode. Side effect while the option is on: a hardware keyboard's PageUp/PageDown scroll instead of moving the caret by a page. The EditText's key listener only sees keys while it has focus, which on a touch-only reader is not the case until something is tapped (seen on device: `adb shell input keyevent KEYCODE_PAGE_DOWN` did nothing), so `EditorScreen` also installs `vm.pageKeyHandler` (a `DisposableEffect` keyed on option/typewriter/find/cmd mode) and `MainActivity.onKeyDown`/`onKeyUp` forward to it; the two paths never both fire (a consumed key stops at the first). Verified on the M6: PageDown, VolumeDown and VolumeUp all turn the page with no focus.
- Overlay lives in `EditorScreen`'s content `Box`, no ripple (`indication = null`). Hidden while the find bar is open (it covers the bottom edge) and in typewriter mode (which re-centres the caret line and would fight the scroll).
- **Untested on a device** (none connected when written): compiled only (`compileFullDebugKotlin`/`compileFreeDebugKotlin`). Things to check on real hardware: touch target vs. right margin text, `bringPointIntoView` not nudging the scroll after the caret moves, e-ink refresh feel.

### `cursor_percent`/`cursor_chars`/`cursor_words` throttling (`cursor_stats_interval`, 2026-09-20)

`refreshStatus()` used to do `_content.value.substring(0, cursor)` + a word count on **every** keystroke and caret move (main thread, O(text before the cursor)). Now it keeps `cursorStatsCc/Wc` cached and rescans at most once per `AppConfig.cursorStatsInterval` seconds (`[misc]` key `cursor_stats_interval`, default 2, 0 = every refresh; INI-only, no Settings row), stretched to 10x the measured scan cost (`cursorStatsCostMs`) so a slow device self-limits to ~10% CPU. The scan counts in place (`countWords(text, end)`, no substring copy). A different document/workspace (`cursorStatsDocKey`) rescans immediately; between scans one delayed `refreshStatus()` (`cursorStatsJob`) guarantees the value never stays stale once the caret stops. Same design as the Tcl `cstat-*` helpers (`writhdeck/.claude/CLAUDE.editor.md`).

### E-ink scheme (`eink`, 2026-09-20)

Built-in scheme `eink` in `ColorSchemes.kt`: black on white (light) / white on black (dark), heading = full foreground, comment `#555555`/`#bbbbbb`, markup `#333333`/`#dddddd`, selection `#c0c0c0`/`#666666` (mid greys kept dark enough for e-ink dithering). Any scheme whose **name starts with `eink`** (built-in or user-defined, e.g. a custom `eink-big`) sets `ThemeColors.highContrast` (`AppConfig.themeColors`), which makes `MainActivity.writhdeckColorScheme` return a flat palette instead of the usual `lerp(bg, dim, ...)` tinted surfaces: every surface/container = `bg`, primary/secondary/tertiary/outline = `fg`, and only `primaryContainer`/`secondaryContainer`/`tertiaryContainer` (selected/active rows) get one light grey (`lerp(bg, fg, 0.18)`). Also raises the page-turn chevrons' alpha from 0.4 to 0.8. Not applied automatically: pick it in Settings > Schemes (existing INI files get the `[eink]` section added on the next settings save, but the built-in lookup works without it).

### In-document Settings access

`EditorScreen`'s "≡" `DropdownMenu` (the in-document overflow menu) has a "Settings" `DropdownMenuItem` right below "Command mode" (above the divider preceding the Save section), calling `onNavigateSettings()` — a new optional param threaded from `AppNavigation.kt` (`nav.navigate("settings")`). Mirrors writhdeck-web, which places "Settings" right after "Command mode" at the top of its "≡" menu. Lets the user reach Settings without leaving an open document.

### Configurable markup markers

Mirrors the Tcl/Tk and web versions: the Display tab's "Markup" section exposes `commentMarker`/`boldMarker`/`italicMarker`/`underlineMarker`/`strikethroughMarker` (all `String`, default `%`/`**`/`//`/`__`/`--`) plus a "Markdown support" `SwitchSettingRow` toggle for `markdownSupport`. Persisted via `IniParser.parse()`'s `marker(k, def)` helper and `IniParser.patchKeys`, exposed as `WrithdeckViewModel` StateFlows, and consumed by `computeSyntaxSpans` in `EditorScreen.kt` (see "Syntax highlighting" above). `markdownSupport` gates `#`..`######` headings and `* ` list bullets; `- ` list bullets are always highlighted. INI key `markdown_support` (canonical) with `markdown_headings` kept as a deprecated alias on read, and both written.

`marker(k, def)` replicates the desktop INI convention where a literal `"0"` value means "marker disabled" (round-trips an empty string through hand-edited INI files): `keys[k]?.let { if (it == "0") "" else it } ?: def`.

### External file save

Via `contentResolver.openOutputStream()` if `canWrite`. URI stored in `externalUri` for subsequent saves without re-picking.

---

## Kotlin/Compose patterns

- `collectAsStateWithLifecycle()` for all ViewModel StateFlows
- `remember(key) { ... }` for expensive derived values (`parseHexColor`, `buildToc`)
- `BackHandler` to intercept the system back gesture
- `AndroidView { factory / update }` for native views inside Compose — `factory` runs once, `update` runs on every recomposition
- Mutable ref arrays (`arrayOf<() -> Unit>({ })`) to pass current compose lambdas into `factory` closures

---

---

## What not to do

- Do not add Tcl/JNI — the engine is gone, all logic is pure Kotlin.
- Do not replace `AndroidView { EditText }` with `BasicTextField` — BasicTextField is not virtualized and becomes unusable on large files.
- Do not call `editText.setText()` from the TextWatcher (feedback loop) — use `ignoreTextChange[0]` guard.
- Do not update `keyHandlerRef[0]` inside the `factory` block — update it in `update {}` to capture current compose state.
- Do not call `focusRequester.requestFocus()` automatically on BrowserScreen open — use the `imeAllowed` pattern.
- Never commit on behalf of the user — let the user decide when and how to commit.
