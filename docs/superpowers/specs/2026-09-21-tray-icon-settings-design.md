# Tray icon and settings dialog for cmdtab

**Status:** approved design, not yet implemented
**Date:** 2026-09-21

## Problem

cmdtab runs entirely in the background. It has no visible presence, and the
only way to change a setting is a hidden hotkey pressed while the switcher is
open (`Alt-R` toggles `raiseAllWindows`, `Alt-G` toggles `groupByApp`,
cmdtab.c:1866-1878). Autorun is asked about exactly once, through a modal
`MessageBox` at startup (`AskAutorun`, cmdtab.c:775), whose own text tells the
user to relaunch the executable to change their mind.

Give the user a tray icon in the notification area and a settings dialog behind
it.

## Scope

In scope: the seven behavior booleans in `struct ini`, plus autorun.

| Setting | Default | Registry value name |
|---|---|---|
| `groupByApp` | true | `groupByApp` (exists) |
| `raiseAllWindows` | true | `raiseAllWindows` (exists) |
| `fastSwitchingForApps` | false | `fastSwitchingForApps` (new) |
| `fastSwitchingForWindows` | true | `fastSwitchingForWindows` (new) |
| `showSwitcherForApps` | true | `showSwitcherForApps` (new) |
| `showSwitcherForWindows` | false | `showSwitcherForWindows` (new) |
| `wrapbump` | true | `wrapbump` (new) |
| autorun | off | `cmdtab` under `...\CurrentVersion\Run` |

All seven behavior values live under `HKCU\Software\stianhoiland\cmdtab` as
`REG_DWORD`, the key the existing `GetRegKey`/`SetRegKey` pair already uses.

Out of scope, deliberately: hotkey rebinding, the blacklist editor, and the
appearance numbers (`iconSize`, `fontSize`, the three margins). They stay at
their compiled-in defaults and can be added later without redesign.

## Constraint that shapes the whole design

The main message loop filters by window handle:

```c
for (MSG msg; GetMessageW(&msg, Switcher, 0, 0) > 0;) DispatchMessageW(&msg);
```
(cmdtab.c:846)

Nothing in the program ever calls `PostQuitMessage`. cmdtab exits because
`DestroyWindow(Switcher)` makes the next `GetMessageW` return `-1` for a window
that no longer exists, which fails the `> 0` test and falls out of the loop —
the source comment reads "Not handling -1 errors. Whatever".

Two consequences, both binding:

1. Removing the `Switcher` filter, or passing `NULL`, breaks the program's only
   exit path. This design therefore does not touch the loop at all.
2. Every window this feature creates must either post its messages to
   `Switcher` or pump its own loop.

The tray icon satisfies (2) by using `Switcher` as its callback window. The
settings dialog satisfies it by being **modal**: `DialogBoxParamW` runs a nested
message loop of its own, which is not hwnd-filtered.

That nested loop also keeps the low-level keyboard hook alive — `WH_KEYBOARD_LL`
callbacks are delivered through the installing thread's message queue, and any
pump will do — so Alt-Tab keeps working while the settings dialog is open.

## Components

### 1. Tray icon

`InitTrayIcon(handle instance)`, called from `RunCmdTab` immediately after
`InitSwitcherWindow`, before the event hooks.

- `NOTIFYICONDATAW` with `hWnd = Switcher`, `uID = 1`,
  `uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP`,
  `uCallbackMessage = WM_CMDTAB_TRAY` (`WM_APP + 1`), tip `"cmdtab"`.
- Icon: `LoadImageW(instance, MAKEINTRESOURCEW(2), IMAGE_ICON,
  GetSystemMetrics(SM_CXSMICON), GetSystemMetrics(SM_CYSMICON), 0)`. Resource
  id 2 is the existing `cmdtab.ico` entry in cmdtab.rc.
- `Shell_NotifyIconW(NIM_ADD, &nid)`, then `nid.uVersion = NOTIFYICON_VERSION_4`
  and `Shell_NotifyIconW(NIM_SETVERSION, &nid)`.

With version 4 the callback arrives as `LOWORD(lparam)` = event,
`GET_X_LPARAM(wparam)` / `GET_Y_LPARAM(wparam)` = screen coordinates.

Explorer restart: register `TaskbarCreated` once at init with
`RegisterWindowMessageW(L"TaskbarCreated")`, store it in a static, and re-add
the icon when the switcher window procedure sees that message.

Teardown: `Shell_NotifyIconW(NIM_DELETE, &nid)` on `WM_DESTROY` of the switcher,
so the icon disappears whether the user quits from the tray menu or through the
existing `OnSwitcherClose` path.

### 2. Tray interaction

Handled in `SwitcherWindowProcedure` under `case WM_CMDTAB_TRAY`:

- `WM_LBUTTONUP` opens the settings dialog.
- `WM_CONTEXTMENU` shows a popup menu with two items, *Settings…* and
  *Quit cmdtab*, built with `CreatePopupMenu` / `AppendMenuW` and shown with
  `TrackPopupMenu(menu, TPM_RIGHTBUTTON | TPM_RETURNCMD, x, y, 0, Switcher, NULL)`.
  `SetForegroundWindow(Switcher)` must precede it or the menu will not dismiss
  when the user clicks elsewhere. This is safe here: `OnSwitcherFocusChange`
  (cmdtab.c:1982) only logs.
- *Quit* calls `DestroyWindow(Switcher)`, the same exit the existing code uses,
  without the "Quit cmdtab?" confirmation — the user chose Quit from a menu
  explicitly.

No *About* item. Nothing needs it.

### 3. Settings dialog

`cmdtab.rc` gains an `IDD_SETTINGS DIALOGEX`, roughly 250x190 dialog units,
style `DS_MODALFRAME | DS_SETFONT | WS_POPUP | WS_CAPTION | WS_SYSMENU`, font
`"Segoe UI"` 9, caption `"cmdtab settings"`:

- Group box *Behavior* with seven `AUTOCHECKBOX` controls.
- Group box *Startup* with one `AUTOCHECKBOX`, "Start cmdtab with Windows".
- `DEFPUSHBUTTON "OK", IDOK` and `PUSHBUTTON "Cancel", IDCANCEL`.

The manifest already declares Common Controls 6.0 and `PerMonitorV2`, so the
dialog gets themed controls and per-monitor DPI scaling with no extra code.

`SettingsDialogProcedure`:

- `WM_INITDIALOG` — `CheckDlgButton` for each of the seven from the live
  `Config`, and for the startup box from `GetAutorun()`.
- `WM_COMMAND` / `IDOK` — for each box, read `IsDlgButtonChecked`, assign into
  `Config`, and `SetRegKey` it; call
  `SetAutorun(checked, L"cmdtab", L"--autorun")` for the startup box;
  `EndDialog(hwnd, IDOK)`.
- `WM_COMMAND` / `IDCANCEL`, and `WM_CLOSE` — `EndDialog(hwnd, IDCANCEL)`,
  changing nothing.

No live apply and no Apply button: OK commits, Cancel discards.

Nothing needs to be refreshed after OK. The keyboard hook reads `Config` fields
at the moment it uses them, and the switcher rebuilds its app list through
`UpdateApps()` every time it is shown, so a changed `groupByApp` is picked up on
the next Alt-Tab.

Single instance: a file-scope `static handle SettingsDialog`. The open function
returns early with `SetForegroundWindow(SettingsDialog)` when it is non-NULL;
`WM_INITDIALOG` sets it, and both `EndDialog` paths clear it.

Control ids live in a new `resource.h` included by both `cmdtab.rc` and
`cmdtab.c`, so the two cannot drift apart:

```c
#define IDD_SETTINGS                 100
#define IDC_GROUP_BY_APP             1001
#define IDC_RAISE_ALL_WINDOWS        1002
#define IDC_FAST_SWITCHING_APPS      1003
#define IDC_FAST_SWITCHING_WINDOWS   1004
#define IDC_SHOW_SWITCHER_APPS       1005
#define IDC_SHOW_SWITCHER_WINDOWS    1006
#define IDC_WRAPBUMP                 1007
#define IDC_AUTORUN                  1008
```

### 4. Registry helpers

`GetRegKey` returns `-1` when a value is absent (cmdtab.c:513), and `-1` is
truthy. The current code relies on that on purpose so `raiseAllWindows` defaults
to on (cmdtab.c:759). Reusing it as-is for `fastSwitchingForApps` and
`showSwitcherForWindows`, whose defaults are **false**, would silently turn them
on for every user with a clean registry.

Add a default-aware wrapper and route all seven through it:

```c
static bool GetRegKeyBool(u16 *keyname, bool fallback)
{
	int value = GetRegKey(keyname);
	return value < 0 ? fallback : !!value;
}
```

Rewrite the two existing `InitConfig` lines to use it with explicit `true`
defaults rather than the `-1` trick. Observable behavior is unchanged; the
intent becomes readable.

Autorun currently has a writer but no reader. Add:

```c
static bool GetAutorun(u16 *keyname);
```

`RegGetValueW` with `RRF_RT_REG_SZ` against
`Software\Microsoft\Windows\CurrentVersion\Run`; the value being present means
autorun is on. Used only to initialize the checkbox.

### 5. Removing the startup prompt

Delete the `if (!HasAutorunLaunchArgument(args)) AskAutorun();` call in
`RunCmdTab` (cmdtab.c:833) together with `AskAutorun` and
`HasAutorunLaunchArgument`. Leaving them would draw `-Wunused-function` under
the build's `-Wall -Wextra`.

The `--autorun` argument itself stays accepted and ignored, so existing Run-key
entries and the `make install` scheduled task keep working unchanged.

## Files

| File | Change |
|---|---|
| `resource.h` | new, control and dialog ids |
| `cmdtab.rc` | `#include "resource.h"`, `IDD_SETTINGS` template |
| `cmdtab.c` | tray init/teardown, `WM_CMDTAB_TRAY` case, context menu, dialog proc, `GetRegKeyBool`, `GetAutorun`, `InitConfig` rewrite, `AskAutorun` removal |
| `Makefile` | `cmdtab.o: cmdtab.rc resource.h` |

## Verification

The repository has no test framework and no automated tests; verification is
manual.

Build under MSYS2 UCRT64 with `make RELEASE=1 CC=gcc` (the Makefile's `CC = c99`
has no binary under mingw).

1. Icon appears in the notification area; tip reads "cmdtab".
2. Left click opens the dialog; left click again while it is open raises the
   existing one instead of opening a second.
3. Right click shows Settings and Quit; clicking outside dismisses the menu.
4. Each checkbox, after OK, is reflected in
   `reg query "HKCU\Software\stianhoiland\cmdtab"`.
5. Cancel after toggling leaves both `Config` and the registry untouched.
6. Restarting cmdtab restores every toggled setting.
7. Alt-Tab still switches windows while the dialog is open.
8. The startup box adds and removes the `cmdtab` value under the `Run` key, and
   reopening the dialog shows it in the state it was left.
9. Quit from the tray menu removes the icon and ends the process.
10. `Alt-R` and `Alt-G` inside the switcher still work and their effect is
    visible the next time the dialog is opened.
11. First run against a clean `HKCU\Software\stianhoiland\cmdtab` shows the
    defaults from the table above — in particular *fast switching for apps* and
    *show switcher for windows* unchecked.

## Known limitations

The startup checkbox governs only the `Run` registry value. `make install`
installs autorun as an elevated scheduled task instead, which this dialog
neither shows nor changes; a user who installed that way will see the box
unchecked while cmdtab still starts with Windows.

Hotkeys, the blacklist and appearance values remain editable only by editing
`InitConfig` and rebuilding.
