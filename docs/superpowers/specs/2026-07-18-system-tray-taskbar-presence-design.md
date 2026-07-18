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

## Scope

- **Windows only, gated on `minimizeLibraryToTray`.** With the setting OFF, or on
  macOS/Linux, there is **no behavior change** — no tray, and the library/reader windows
  behave as normal taskbar windows, so the invariant already holds trivially.
- When the setting is ON, the reconciler is the **single authority** over library
  visibility. The existing `keepLibraryWindowInBackgroundOnReaderOpen` and
  `keepLibraryWindowInBackgroundOnReaderClose` settings are **bypassed** at their call
  sites (they remain fully in effect off-Windows / setting-off).
- `oneReaderWindowPerPublication` is orthogonal (reader deduplication) and untouched.

## Tray lifecycle

When `minimizeLibraryToTray` is ON, the tray icon is **persistent**: created when the
library window is created (or when the setting is toggled on), always usable to bring up
the library regardless of whether the library is hidden, minimized, or already visible,
and regardless of reader state. It is destroyed only when the setting is toggled off or
the app quits.

This replaces the merged code's lazy `ensureWindowsLibraryTray`-on-minimize approach and
decouples the tray icon (a permanent affordance) from visibility reconciliation (which
decides where the library window sits).

## State

Two pieces of state drive every decision.

### Per-reader: `openedFromLibrary`

Recorded when a reader opens; stored in the reader session state so it is available at
close. Implemented via a `source: "library" | "direct"` flag on
`readerActions.openRequest`:

- Direct entry points — `event.ts` CLI/file channel, OS "open-file", CLI `read` — set
  `"direct"`.
- Every other dispatch (clicking a book in the library UI) is `"library"`.

### Library: `libraryOpenedIndependently` (latch)

`true` once the user engages the library **as itself** — a normal app launch to the
library, or tray "Show Library" / tray toggle-to-show. It stays `false` while the library
exists only as scaffolding to host a direct open (`appActivate`'s incidental surfacing of
the library during a direct-open flow does **not** flip it).

- Initialized when the library window is created: `false` if creation is driven by a
  direct-open flow (scaffolding), `true` for a normal launch.
- Flipped to `true` whenever the user surfaces the library on its own (tray show/toggle).
- Not reset within a session: on Windows the library window only goes away when the app
  quits, so the latch is effectively per-app-session. Minimizing the library does **not**
  reset it.

Note: tracking "a from-library reader open" as a trigger is unnecessary — a reader cannot
be opened from the library without the library already being visible/engaged, so the latch
is already `true` in that case.

## The pure reconciler

A single electron-free function, exhaustively unit-tested. All window manipulation happens
in the caller (adapter) based on the returned action.

```ts
type LibraryVisualState = "visible" | "minimized" | "hidden";

type TrayEvent =
  | "readerOpened" | "readerClosed"
  | "libraryMinimized" | "trayToggled" | "userRequestedShow";

type LibraryAction = "hide" | "show" | "showFocused" | "minimize" | "close" | "none";

interface ReconcileState {
    readerCount: number;                 // from redux session state, post-transition
    libraryState: LibraryVisualState;
    openedFromLibrary: boolean;          // provenance of the reader in the current event
    libraryOpenedIndependently: boolean; // the latch
}

function decide(event: TrayEvent, state: ReconcileState): LibraryAction;
```

Distinctions in `libraryState` (electron reports these ambiguously, so the adapter maps
them explicitly): `minimized` = `isMinimized()`; `hidden` = `!isVisible() &&
!isMinimized()` (i.e. `hide()` was called); `visible` = otherwise.

### Decision table

| Event | Condition | Action | Rationale |
|-------|-----------|--------|-----------|
| `readerOpened` | `openedFromLibrary \|\| !libraryOpenedIndependently` | `hide` | Reader was opened from the library, or the reader caused the library to open (scaffolding) → tuck it away; taskbar shows only the reader. No-op if already hidden. |
| `readerOpened` | otherwise | `none` | Direct open over an independently-open library → leave it as a background window. |
| `readerClosed` | `readerCount ≥ 1` | `none` | Other readers still hold the taskbar. |
| `readerClosed` | `0`, library on taskbar (`visible`/`minimized`) | `none` | Invariant already satisfied. |
| `readerClosed` | `0`, `hidden`, closing reader **from library** | `showFocused` | Return to the library — the user's flow started there. |
| `readerClosed` | `0`, `hidden`, **direct**, `libraryOpenedIndependently` | `show` | Restore to the taskbar without stealing focus — the library has its own presence. |
| `readerClosed` | `0`, `hidden`, **direct**, `!libraryOpenedIndependently` | `close` | Pure direct-open session → quit, exactly as today. |
| `libraryMinimized` | `readerCount ≥ 1` | `hide` | A reader holds the taskbar → tuck the library to tray. |
| `libraryMinimized` | `readerCount == 0` | `none` | Stay minimized on the taskbar — never tray-only. |
| `trayToggled` | `visible`, `readerCount ≥ 1` | `hide` | Tuck away. |
| `trayToggled` | `visible`, `readerCount == 0` | `minimize` | Keep a taskbar presence — never tray-only. |
| `trayToggled` | `minimized`/`hidden` | `showFocused` (+ latch) | Bring the library up. |
| `userRequestedShow` | tray menu "Show Library" | `showFocused` (+ latch) | Explicit user request. |

The reader-close decision is idempotent, so the two close sagas both landing on it reach
the same action safely.

`showFocused` shows and focuses (the library is becoming the primary window / the user
asked for it); `show` restores taskbar presence without stealing focus (the invariant
forces the library back, but the user was in a directly-opened reader). Whenever the
adapter executes `show`, `showFocused`, or a `userRequestedShow`/`trayToggled` surface, it
sets `libraryOpenedIndependently = true`.

## Integration points

All gated on `minimizeLibraryToTray`. A thin adapter reads `readerCount` from redux and
`libraryState` from the live window, supplies the two flags, calls `decide()`, executes
the returned action, and updates the latch.

- **`src/main/tools/libraryTray.ts`** — owns the persistent tray and the adapter.
  `minimize` handler → `decide("libraryMinimized")`. Tray `click` → `decide("trayToggled")`.
  Menu "Show Library" → `decide("userRequestedShow")`. Menu "Quit" → `app.quit()`
  (unchanged).
- **`src/main/redux/sagas/win/browserWindow/createLibraryWindow.ts`** — initialize the
  latch (scaffolding vs. independent) at creation, and route the direct-open hide (today's
  `if (readersArray.length === 1) libWindow.hide()`) through the adapter.
- **`src/main/redux/sagas/reader.ts`** (`minimizeLibraryWindowOnReaderOpenIfEnabled`,
  reader-open path) — when the setting is ON, record the reader's provenance and call
  `decide("readerOpened")`; skip the `keepLibraryWindowInBackgroundOnReaderOpen` minimize.
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
- **`src/main/redux/sagas/event.ts` and `src/main/cli/`** — mark direct opens as
  `source: "direct"` on the open action, and signal that the resulting library is
  scaffolding (so the latch initializes `false`).

## Testing

New unit test mirroring the reconciler's source path (jest excludes `<rootDir>/src/`),
exercising the pure `decide()` against the full decision table plus scenario walks:

1. From-library open → the library hides; `libraryMinimized` while the reader is open →
   `hide`; close the last reader → `showFocused` (return to the library).
2. No readers → `libraryMinimized` → `none` (stays minimized on the taskbar).
3. Two from-library readers, library hidden → close one (`readerCount ≥ 1`) → `none`;
   close the second → `showFocused`.
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

Adapters and the electron `Tray` stay thin and are covered by manual Windows verification
(there is no harness for live window/tray behavior). Manual checks mirror scenarios 1–7
with real windows.

## Non-goals

- No change to macOS/Linux, or to Windows with the setting off.
- No change to reader-window bounds handling, or to `oneReaderWindowPerPublication`.
- Live-toggling `minimizeLibraryToTray` mid-session is not retroactive: it takes effect on
  the next relevant transition (reader open/close, minimize, tray action). Turning the
  setting off destroys the tray, as today.
