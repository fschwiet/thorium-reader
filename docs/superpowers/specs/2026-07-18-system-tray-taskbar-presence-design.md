# System tray: keep the library reachable in the taskbar

Adapted to PR #3667 (commit `e76cb1176`, "feat(library): Adds an optional Windows-only
setting to minimize the LibraryWin to the system tray"). This supersedes the earlier
design `2026-06-20-system-tray-taskbar-presence-design.md`, which was written against the
rejected commit `633dd0e8` (unconditional minimize-to-tray).

## Background

`633dd0e8` made the library window minimize to the system tray **unconditionally** on
Windows. The 2026-06-20 design proposed a reconciler to prevent the app from vanishing
into a tray-only state — that reconciler was the *only* thing holding the "app is always
reachable" property, because minimize-to-tray always fired.

`e76cb1176` replaced that with an **opt-in setting**, `minimizeLibraryToTray` (default
off, Windows only), implemented in `src/main/tools/libraryTray.ts`. Its minimize handler
hides the library to the tray whenever the setting is on:

```js
libraryWindow.on("minimize", () => {
    const tray = ensureWindowsLibraryTray();
    if (tray && !libraryWindow.isDestroyed()) {
        libraryWindow.hide();          // hides regardless of reader count
        showWindowsLibraryTrayHint();
    }
});
```

The merged implementation has **no awareness of reader-window count** and **no latch**
for whether the library was ever shown. So with the setting on and the library as the
only window, minimizing it hides it entirely — the "app is easy to lose" state the
original design set out to prevent, softened only by a tray icon and a balloon hint.

This design brings the original invariant into the opt-in world, and — per product
direction — reframes the feature's purpose: the tray integration exists so that
**opening a reader tucks the library into the tray**, leaving the taskbar showing only
the reader (one entry, not two) while the library keeps running in the background.

## Invariant

> With `minimizeLibraryToTray` ON (Windows only), the app is **never** reduced to a tray
> icon alone — at least one window is always represented on the taskbar. The library may
> sit in the tray **only while a reader window holds the taskbar**. When the last reader
> closes, the library returns to the taskbar, **unless** it was never opened
> independently (a pure direct-open session), in which case the app quits, exactly as
> today.

A minimized window still occupies the taskbar; a `hide()`-d window does not.

The invariant also has to survive its own exit: turning `minimizeLibraryToTray` OFF removes
the tray icon, so it must surface the library first (see Tray lifecycle).

## Scope

- **Windows only, gated on `minimizeLibraryToTray`.** With the setting OFF, or on
  macOS/Linux, there is **no behavior change** — no tray, and the library/reader windows
  behave as normal taskbar windows, so the invariant already holds trivially.
- When the setting is ON, the reconciler is the **single authority** over library
  visibility. The existing `keepLibraryWindowInBackgroundOnReaderOpen` and
  `keepLibraryWindowInBackgroundOnReaderClose` settings are **bypassed** at their call
  sites (they remain fully in effect off-Windows / setting-off).
- "Single authority" means every existing call site that shows, hides, minimizes or
  restores the library window is routed through the adapter when the setting is ON. Those
  sites are: the tray minimize/click/menu handlers (`libraryTray.ts`), `showLibrary()`
  (`src/main/tools/showLibrary.ts` — shared by the tray menu **and** the app Window menu
  at `menu.ts`), the direct-open hide in `createLibraryWindow.ts`, the reader-open minimize
  in `sagas/reader.ts`, and the reader-close block in `sagas/win/reader.ts`. Two sites stay
  outside the reconciler and are explicitly out of scope: `appActivate`'s restore/show/
  reader-fallback dance (`sagas/win/library.ts`), which surfaces a window without changing
  the library's role in the session, and library `winClose` (same file), which is app
  teardown.
- `oneReaderWindowPerPublication` is orthogonal (reader deduplication) but **not**
  untouched: its early-return path currently calls the reader-open minimize helper, so it
  needs an explicit rule (see Integration points).

## Tray lifecycle

When `minimizeLibraryToTray` is ON, the tray icon is **persistent**: created when the
library window is created (or when the setting is toggled on), always usable to bring up
the library regardless of whether the library is hidden, minimized, or already visible,
and regardless of reader state. It is destroyed only when the setting is toggled off or
the app quits.

This replaces the merged code's lazy `ensureWindowsLibraryTray`-on-minimize approach and
decouples the tray icon (a permanent affordance) from visibility reconciliation (which
decides where the library window sits).

### Settings observer: both edges

The merged observer (`ensureWindowsLibraryTraySettingsObserver`) only handles the ON→OFF
edge, and only to destroy the tray. Both edges now matter:

- **OFF→ON** — create the tray (it is persistent, not lazy).
- **ON→OFF** — **surface the library first, then destroy the tray.** Under this design the
  library is hidden for the entire time a reader is open, so destroying the tray while the
  library is hidden leaves the app with no window *and* no tray icon: the invariant broken
  by the one transition that is supposed to opt out of it. The OFF edge runs
  `decide("userRequestedShow")` (or equivalently forces `show`) before teardown.

### Balloon hint

`showWindowsLibraryTrayHint()` / `trayHintShown` fire once, on the merged code's
user-initiated minimize. Under this design most hides are triggered by *opening a reader*
— an action the user did not frame as "minimize to tray" — so the balloon would read as
noise. The hint fires only on a `hide` produced by `libraryMinimized` or `trayToggled`
(user-initiated tucking), never on a `hide` produced by `readerOpened`. It remains
once-per-session.

## State

Two pieces of state drive every decision.

**Both are maintained unconditionally — they are *not* gated on `minimizeLibraryToTray`.**
Only the `decide()` call and the action it produces are gated. Recording provenance and
maintaining the latch cost nothing when the setting is off, and gating them would break the
OFF→ON toggle: readers already open at the moment the user ticks the box would carry no
`source`, and the latch would have no initialized value, so the first `readerClosed` after
the toggle could decide `close` and quit the app on a session the user never opted into.
This is the whole reason mid-session toggling stays cheap — the state is always there, and
the setting only decides whether anything acts on it.

### Per-reader: `openedFromLibrary`

Recorded when a reader opens; stored in the reader session state so it is available at
close. Implemented via a `source: "library" | "direct"` flag on
`readerActions.openRequest`:

- Direct entry points — `event.ts` CLI/file channel, OS "open-file", CLI `read` — set
  `"direct"` **explicitly**.
- Absent flag defaults to `"library"`. That keeps the five renderer dispatch sites
  (`PublicationCard.tsx`, `AllPublicationPage.tsx`, `OpdsControls.tsx`,
  `catalogLcpControls.tsx`, `catalogControls.tsx`) and the post-license-renewal re-open in
  `src/main/redux/sagas/lcp.ts` correct with no edit — only the direct sites change.

The flag threads `openRequest` → `readerOpenRequest` → `createReaderWindow` →
`winActions.session.registerReader`, which is what lands it in the session state.

**Read it before the unregister.** `winClose` (`sagas/win/reader.ts`) snapshots
`readersBeforeUnregistered`, then dispatches `unregisterReader`, then re-selects the
readers to get the post-transition count. Provenance must come from the *first* snapshot;
by the time the count is correct the closing reader's entry — and its `source` — is gone.
Reading it from the second selection silently degrades every close to `"direct"`.

### Library: `libraryOpenedIndependently` (latch)

`true` once the user engages the library **as itself** — a normal app launch to the
library, tray "Show Library" / tray toggle-to-show, app Window menu → Show Library, or
opening a book *from* the library. It stays `false` while the library exists only as
scaffolding to host a direct open (`appActivate`'s incidental surfacing of the library
during a direct-open flow does **not** flip it).

Initialization needs a signal that does not exist at library-window creation time. On a
cold CLI / file-association launch, `sagas/index.ts` dispatches
`winActions.library.openRequest.build()` unconditionally *before* it waits for
`openSucess` and starts `events.saga()`; the file path is already sitting on
`getOpenFileFromCliChannel()`, queued back in `cli/index.ts`. So `createLibraryWindow`
runs with zero readers and nothing distinguishing a direct-open launch from a normal one.
`appActivate` dispatching the same payload-free action, and `showLibrary()` putting a bare
`true` on the appActivate channel, close off the "thread a flag through the action" route
as well.

The signal exists earlier, at argv-parse time:

- **`launchedForDirectOpen`** — set in `src/main/cli/index.ts` where `pathArgv` is
  inspected, covering both the `setOpenUrl` deep-link branch and the
  `openFileFromCliChannel.put(...)` branch. A main-process flag, not persisted, not synced
  to renderers. Windows file associations and deep links both arrive through argv, so argv
  is the whole signal here; electron's `open-file` event is macOS-only and the feature is
  Windows-gated, so it needs no equivalent.
- The latch initializes to `!launchedForDirectOpen` when the library window is created.

Then:

- Flipped to `true` whenever the adapter executes a `show` — every path that puts the
  library in front of the user on its own (tray toggle/menu, app Window menu, the
  invariant-driven restore on last-reader-close).
- Flipped to `true` on a `readerOpened` with `source: "library"`. This is belt-and-braces
  against any surfacing path that escapes the adapter: if the user got to the library well
  enough to click a book in it, the library is engaged.
- Not reset within a session: on Windows the library window only goes away when the app
  quits, so the latch is effectively per-app-session. Minimizing the library does **not**
  reset it.

An earlier draft argued the from-library trigger was unnecessary because "a reader cannot
be opened from the library without the library already being visible/engaged". That premise
is false: visibility and the latch are deliberately decoupled — `appActivate` surfaces the
library without flipping it — so any surfacing path outside the adapter leaves a visible,
usable, unlatched library. The concrete hole in that draft:

> Cold CLI open of book A → latch `false`, library hidden → user picks Window → Show
> Library from the reader's menu bar (`showLibrary()`, direct `restore()`+`show()`, latch
> untouched) → browses, opens book B → closes both readers → last close decides `close`
> → the app quits out from under a library the user is actively using.

Routing `showLibrary()` through the adapter closes that specific path; the
`readerOpened`/`"library"` trigger closes the whole class, which is why both are in this
design. The two are cheap and independent, and the latch is one-way, so the redundancy
costs nothing.

## The pure reconciler

A single electron-free function, exhaustively unit-tested. All window manipulation happens
in the caller (adapter) based on the returned action.

```ts
type LibraryVisualState = "visible" | "minimized" | "hidden";

type TrayEvent =
  | "readerOpened" | "readerClosed"
  | "libraryMinimized" | "trayToggled" | "userRequestedShow";

type LibraryAction = "hide" | "show" | "minimize" | "close" | "none";

interface ReconcileState {
    readerCount: number;                 // from redux session state, post-transition
    libraryState: LibraryVisualState;
    openedFromLibrary: boolean;          // provenance of the reader in the current event
    libraryOpenedIndependently: boolean; // the latch
}

function decide(event: TrayEvent, state: ReconcileState): LibraryAction;
```

### `libraryState` is tracked, not derived

Do **not** derive `libraryState` from electron predicates. The hide fires on the `minimize`
event, so a tray-hidden library carries both `WS_MINIMIZE` and a cleared `WS_VISIBLE`, and
`isMinimized()` (`IsIconic()`) keeps returning `true` after `ShowWindow(SW_HIDE)`. A
predicate mapping that tests `isMinimized()` first therefore classifies a tray-hidden
library as `minimized` — "on the taskbar" — and the `readerClosed | 0, visible/minimized |
none` row then leaves the app tray-only on the last close. That is precisely the state the
invariant exists to forbid.

The adapter is the only code that calls `hide()` / `show()` / `minimize()` / `restore()` on
the library window, and it already owns mutable state for the latch, so it owns
`libraryState` too: set it from the action it just executed, plus the window's own
`minimize` / `restore` / `show` events for user-driven transitions. Electron predicates are
used only as a fallback on first observation.

### One `show`, not two

An earlier draft split `show` (restore taskbar presence without focus) from `showFocused`.
The split doesn't earn its keep: both `readerClosed` rows that produce a show fire as the
**last** reader closes, so there is no other window left to steal focus from, and every
other `show` is an explicit user request that *wants* focus. It is also the hardest to
implement
honestly — `show()` focuses, the non-focusing variant is `showInactive()`, and `restore()`
focuses on Windows regardless, so "restore without stealing focus" from a hidden-and-
minimized window is not reliably achievable anyway.

`show` therefore means: `restore()` if minimized, then `show()`, then focus.

### Decision table

| Event | Condition | Action | Rationale |
|-------|-----------|--------|-----------|
| `readerOpened` | `openedFromLibrary \|\| !libraryOpenedIndependently` | `hide` | Reader was opened from the library, or the reader caused the library to open (scaffolding) → tuck it away; taskbar shows only the reader. No-op if already hidden. |
| `readerOpened` | otherwise | `none` | Direct open over an independently-open library → leave it as a background window. |
| `readerClosed` | `readerCount ≥ 1` | `none` | Other readers still hold the taskbar. |
| `readerClosed` | `0`, library on taskbar (`visible`/`minimized`) | `none` | Invariant already satisfied. |
| `readerClosed` | `0`, `hidden`, closing reader **from library** | `show` | Return to the library — the user's flow started there. |
| `readerClosed` | `0`, `hidden`, **direct**, `libraryOpenedIndependently` | `show` | The library has its own presence → restore it to the taskbar. |
| `readerClosed` | `0`, `hidden`, **direct**, `!libraryOpenedIndependently` | `close` | Pure direct-open session → quit, exactly as today. |
| `libraryMinimized` | `readerCount ≥ 1` | `hide` | A reader holds the taskbar → tuck the library to tray. |
| `libraryMinimized` | `readerCount == 0` | `none` | Stay minimized on the taskbar — never tray-only. |
| `trayToggled` | `visible`, `readerCount ≥ 1` | `hide` | Tuck away. |
| `trayToggled` | `visible`, `readerCount == 0` | `minimize` | Keep a taskbar presence — never tray-only. |
| `trayToggled` | `minimized`/`hidden` | `show` (+ latch) | Bring the library up. |
| `userRequestedShow` | tray menu, or app Window menu, "Show Library" | `show` (+ latch) | Explicit user request. |

Whenever the adapter executes `show`, it sets `libraryOpenedIndependently = true`; it also
sets it on a `readerOpened` whose `source` is `"library"`.

### Idempotency of the close paths

On an in-app close, **both** close paths run: `readerCLoseRequestFromIdentifier`
(`sagas/reader.ts`) and `winClose` (`sagas/win/reader.ts`, reached via the reader window's
`close` event through `sagas/win/session/reader.ts`). The table is idempotent for
`show`/`none` — the first call shows the library, the second reads `visible` and returns
`none` — but **not** for `close`: the first call destroys the library window, so the second
must tolerate a missing or destroyed window and return `none` rather than throwing or
re-deciding. The adapter guards on
`libWin && !libWin.isDestroyed() && !libWin.webContents.isDestroyed()` before every action,
and treats an absent window as "nothing to reconcile".

## Integration points

All gated on `minimizeLibraryToTray`. A thin adapter reads `readerCount` from redux, supplies
its own tracked `libraryState` and the two flags, calls `decide()`, executes the returned
action, then updates `libraryState` and the latch from what it executed.

- **`src/main/tools/libraryTray.ts`** — owns the persistent tray and the adapter.
  `minimize` handler → `decide("libraryMinimized")`. Tray `click` → `decide("trayToggled")`.
  Menu "Show Library" → `decide("userRequestedShow")`. Menu "Quit" → `app.quit()`
  (unchanged). Settings observer gains the OFF→ON edge (create tray) and shows the library
  on the ON→OFF edge before destroying it.
- **`src/main/tools/showLibrary.ts`** — routed through the adapter when the setting is ON.
  This is the second door into library visibility and it is *not* tray-only:
  `src/main/menu.ts` binds it to the app Window menu, which is present on the **reader**
  window's menu bar. With the library alive but hidden it takes the direct
  `restore()`+`show()` branch, bypassing both the adapter and the latch — the hole
  described under the latch above. Its `appActivate`-channel fallback (library destroyed)
  is unchanged.
- **`src/main/cli/index.ts`** (+ the macOS pre-ready `open-file` handler) — set
  `launchedForDirectOpen` where `pathArgv` is inspected, covering both the `setOpenUrl`
  deep-link branch and the `openFileFromCliChannel.put(...)` branch.
- **`src/main/redux/sagas/win/browserWindow/createLibraryWindow.ts`** — initialize the
  latch from `!launchedForDirectOpen` at creation, and route the direct-open hide (today's
  `if (readersArray.length === 1) libWindow.hide()`) through the adapter.
- **`src/main/redux/sagas/reader.ts`** (`minimizeLibraryWindowOnReaderOpenIfEnabled`,
  reader-open path) — record the reader's provenance **unconditionally** (see State); then,
  when the setting is ON, call `decide("readerOpened")` and skip the
  `keepLibraryWindowInBackgroundOnReaderOpen` minimize.
- **`src/main/redux/sagas/reader.ts`** (`readerOpenRequest`, the
  `oneReaderWindowPerPublication` early return) — this path focuses an existing reader and
  returns without creating one, and today it still calls the reader-open minimize helper.
  When the setting is ON it fires `decide("readerOpened")` like any other open (the count
  is unchanged, but the user did just ask for a book, so the library should tuck away), and
  it **overwrites the existing reader's stored provenance with the current request's
  `source`**. A reader first opened directly and later re-requested from the library is now
  a from-library reader, and closing it should return to the library rather than quit.
- **`src/main/redux/sagas/win/reader.ts`** (`winClose`) — when the setting is ON, call
  `decide("readerClosed")` and skip the existing
  `keepLibraryWindowInBackgroundOnReaderClose` / `isMinimized → restore` /
  **`!isVisible → close`** / `show` block. The `!isVisible → close` branch is the one that
  would otherwise destroy the library on a non-last reader close now that the library is
  hidden while readers are open. (The reader-bounds-copy logic this section previously
  mentioned as "retained" no longer exists: PR #3752, merged into this branch on
  2026-08-05, deleted the reader attach/detach feature — `ReaderMode`,
  `detachModeRequest`/`detachModeSuccess`, `readerNewWindowBound`, and the `winClose` block
  that copied the closing reader's bounds onto the library window on
  `ReaderMode.Detached`. There is nothing bounds-related left to preserve here.)
- **`src/main/redux/sagas/reader.ts`** (`readerCLoseRequestFromIdentifier`) — when the
  setting is ON, replace the unconditional `libWin.show()` with `decide("readerClosed")`.
  The `restoreBrowserWindowState(libWin, libraryWindowState)` call that precedes it becomes
  conditional on the decided action being `show`: restoring bounds on a library that stays
  hidden (or is about to be closed) is at best wasted work and at worst resizes a window
  the user never sees come back.
- **`src/main/redux/sagas/event.ts`** — mark direct opens (CLI/file channel, title channel,
  `thorium://` deep link) as `source: "direct"` on the open action. The "this library is
  scaffolding" signal is *not* set here: by the time these sagas run the library window
  already exists (see the latch section) — that signal comes from `cli/index.ts` above.

## Testing

Jest has `<rootDir>/src/` in `modulePathIgnorePatterns`, so tests live under `test/`:
the reconciler at `src/main/tools/libraryTrayReconciler.ts` gets
`test/main/tools/libraryTrayReconciler.test.ts`, alongside the existing
`test/main/tools/*.test.ts`. It exercises the pure `decide()` against the full decision
table plus scenario walks:

1. From-library open → the library hides; `libraryMinimized` while the reader is open →
   `hide`; close the last reader → `show` (return to the library).
2. No readers → `libraryMinimized` → `none` (stays minimized on the taskbar).
3. Two from-library readers, library hidden → close one (`readerCount ≥ 1`) → `none`;
   close the second → `show`.
4. Cold direct-open (library scaffolding, latch `false`) → the library hides on
   `readerOpened`; close the last reader → `close` (quit).
5. As (4), but tray "Show Library" before closing → latch `true` → close the last reader →
   `show` (library stays, no quit).
6. Mixed: from-library reader1 hides the library; direct reader2 opens (already hidden →
   `none`); reader1 closes (`readerCount ≥ 1` → `none`); reader2 (direct) closes last →
   `show` (invariant wins over "leave a direct reader alone").
7. Direct open over an independently-open library (latch `true`) → `readerOpened` → `none`
   (library remains a background window).
8. `trayToggled` with no readers and a visible library → `minimize` (never tray-only).

Regression cases for the failure modes this revision closes:

9. Latch flip on a from-library open: cold direct-open session (latch `false`), then a
    `readerOpened` with `source: "library"` → latch `true` → last close → `show`, not
    `close`. The app must not quit under a library the user was using.
10. Tracked-vs-derived visual state: library `hidden` while its underlying window would
    still report `isMinimized()` → `readerClosed` at count 0 must yield `show`, never
    `none`. (Reconciler-level this is just "`hidden` means hidden"; the value is that it
    pins the contract the adapter has to uphold.)
11. `userRequestedShow` while `hidden` **and** the window was minimized before being
    hidden → `show` (the adapter must then `restore()` before `show()`).

Adapters and the electron `Tray` stay thin and are covered by manual Windows verification
(there is no harness for live window/tray behavior). Manual checks mirror scenarios 1–7
with real windows, plus three adapter-specific ones the pure tests cannot reach:

- Toggling `minimizeLibraryToTray` OFF while the library is hidden → the library reappears
  before the tray icon is destroyed.
- Window menu → Show Library from a reader while the library is hidden → the library
  appears *and* the latch flips.
- Double `readerClosed` at count 0 where the first decided `close` → the second call finds
  a destroyed library window and no-ops instead of throwing.

## Non-goals

- No change to macOS/Linux, or to Windows with the setting off.
- No change to reader-window bounds handling, or to `oneReaderWindowPerPublication`'s
  deduplication behavior (its early-return path gains a reconciler call and a provenance
  overwrite, but which window it reuses is untouched).
- Live-toggling `minimizeLibraryToTray` mid-session is not retroactive: it takes effect on
  the next relevant transition (reader open/close, minimize, tray action). Turning the
  setting off destroys the tray — but, unlike today, only after surfacing the library, so
  the toggle can never strand the app with neither a window nor a tray icon.
- **The setting does not require an app restart**, and shouldn't: no setting in Thorium
  does today (there is no `restart`/`relaunch` string in the locales and no `app.relaunch()`
  in `src/`), so this would be the first. Restart would delete only the two settings-observer
  edges — every expensive part of this design (the latch, provenance threading,
  `showLibrary()` routing, tracked visual state, close-path idempotency) is unaffected by
  *when* the setting is read. It would also cost a relaunch prompt, a new i18n string across
  ~25 locale files, and a "requires restart" affordance the settings UI has no pattern for,
  in exchange for tearing down the user's session to tick a window-management checkbox.
  Live toggling stays cheap specifically because the two state values above are maintained
  unconditionally.
- `appActivate` keeps its own show/restore logic and is not routed through the reconciler.
  It surfaces an existing reader in preference to the library, so under this design it does
  not fight the invariant; revisit only if that preference changes.
