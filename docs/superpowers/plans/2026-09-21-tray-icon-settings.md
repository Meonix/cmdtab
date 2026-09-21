# Tray Icon and Settings Dialog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give cmdtab a notification-area icon whose left click opens a modal settings dialog for the seven behavior booleans plus autorun, and whose right click offers Settings and Quit.

**Architecture:** The tray icon hangs off the existing switcher window (`Switcher`), so tray callbacks pass the main loop's hwnd filter untouched. The settings dialog is a modal `DialogBoxParamW` driven by a template in `cmdtab.rc`, so it runs its own nested message loop and the main loop needs no change at all. Settings persist as `REG_DWORD` values under the registry key the program already uses.

**Tech Stack:** C99, Win32 (user32, shell32, comctl32 v6 via manifest), windres resource compiler, GNU make, mingw-w64 GCC under MSYS2 UCRT64.

**Spec:** `docs/superpowers/specs/2026-09-21-tray-icon-settings-design.md`

## Global Constraints

- Language is C99. The Makefile passes `-std=c99 -Wall -Wextra -pedantic -Wno-unused-parameter`. **Every task must build with zero new warnings.**
- Build command on this machine, from PowerShell:
  `$env:MSYSTEM="UCRT64"; & C:\msys64\usr\bin\bash.exe -lc "cd /d/Software/cmdtab && make CC=gcc"`
  `CC=gcc` is mandatory — the Makefile's default `CC = c99` has no binary under mingw. Add `RELEASE=1` for an optimized, windowed build; omit it for the debug build used while developing.
- Unicode only. Call the `...W` form of every Win32 function and use `L"..."` literals. `UNICODE` and `_UNICODE` are defined at the top of `cmdtab.c`.
- Indentation in `cmdtab.c` and `cmdtab.rc` is **tabs**, not spaces. Match it.
- The project is deliberately one translation unit. Do not create new `.c` files. `resource.h` (Task 3) is the only new header and contains nothing but `#define`s.
- Do not add libraries to `LDLIBS`. `shell32` and `user32` are linked by mingw-w64 by default; for the MSVC path add a `#pragma comment(lib, ...)` inside the existing `#ifdef _MSC_VER` block instead.
- Registry key for settings: `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab`, values are `REG_DWORD`. Autorun key: `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run`, value name `cmdtab`, `REG_SZ`.
- Do not touch the main message loop at `cmdtab.c:846`. It filters on `Switcher`, and the program's only exit path depends on `GetMessageW` returning `-1` after `DestroyWindow(Switcher)`. Changing it breaks quitting.

## Testing approach — read this before Task 1

**This repository has no test framework, no test directory and no automated tests.** It is a 2100-line Win32 GUI program whose behavior is global keyboard hooks, shell windows and the notification area. Adding a unit-test harness is out of scope and was not part of the approved design.

So the usual red/green TDD cycle does not apply here, and this plan does not pretend otherwise. Each task instead has a two-part gate, and **both parts must pass before the commit step**:

1. **Compile gate (automated).** Build with the command above. Zero errors, and zero warnings that were not already present before the task. Capture the warning count before you start if you are unsure.
2. **Behavior gate (manual, scripted).** Each task lists exact commands to run and exactly what you must observe on screen. These are written so a failure is unambiguous. Run every one of them.

If a behavior gate fails, fix it before committing. Do not commit a task with a failing gate and a note to fix it later.

**Before every manual gate, kill any running instance**, or the singleton mutex makes the new build show "cmdtab is already running" and you will be testing the old binary:

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
```

(Double slashes are for MSYS2 bash, which would otherwise rewrite `/F` into a path. From PowerShell use single slashes: `taskkill /F /IM cmdtab.exe`.)

---

### Task 1: Persist the five unsaved behavior settings

`struct ini` has seven behavior booleans but only two of them, `groupByApp` and `raiseAllWindows`, survive a restart. This task makes all seven readable from the registry, and fixes a trap in how defaults are computed.

The trap: `GetRegKey` returns `-1` when the value does not exist (`cmdtab.c:513`), and `-1` is truthy. Today the code leans on that on purpose so `raiseAllWindows` defaults to on. Two of the five new settings default to **false**, so reusing `GetRegKey` directly would silently switch them on for every user with a clean registry.

**Files:**
- Modify: `cmdtab.c` — add `GetRegKeyBool` after `SetRegKey` (ends at `cmdtab.c:530`)
- Modify: `cmdtab.c:758-759` — the two `GetRegKey` lines in `InitConfig`

**Interfaces:**
- Consumes: `GetRegKey(u16 *keyname)` and `SetRegKey(u16 *keyname, int value)`, both already in the file.
- Produces: `static bool GetRegKeyBool(u16 *keyname, bool fallback)` — returns the stored value as a bool, or `fallback` when the value is absent. Task 3 does not call it, but the settings it reads here are what the dialog displays.

- [ ] **Step 1: Record the current behavior so you can prove the change works**

Build and run the current binary, then confirm that `showSwitcherForWindows` is not honored from the registry today:

```bash
make CC=gcc
reg add "HKCU\\Software\\stianhoiland\\cmdtab" //v showSwitcherForWindows //t REG_DWORD //d 1 //f
taskkill //F //IM cmdtab.exe 2>/dev/null || true
./cmdtab.exe --autorun &
```

Hold `Alt` and tap the key above `Tab` (backquote/tilde). Expected **now**: the switcher window does *not* appear — window switching happens silently, because `Config.showSwitcherForWindows` is hardcoded `false` and the registry value is ignored. Note that you saw this. This is the behavior Task 1 changes.

- [ ] **Step 2: Add the default-aware registry read**

In `cmdtab.c`, immediately after the closing brace of `SetRegKey` (currently `cmdtab.c:530`), add:

```c
static bool GetRegKeyBool(u16 *keyname, bool fallback)
{
	int value = GetRegKey(keyname);
	return value < 0 ? fallback : !!value; // GetRegKey returns -1 when the value does not exist, and -1 is truthy
}
```

- [ ] **Step 3: Read all seven settings in InitConfig**

In `InitConfig`, replace these two lines (`cmdtab.c:758-759`):

```c
	Config.groupByApp = GetRegKey(L"groupByApp");
	Config.raiseAllWindows = GetRegKey(L"raiseAllWindows"); // GetRegKey returns -1 when unset, which is truthy, so this defaults to on
```

with:

```c
	// Stored settings override the defaults set in the struct literal above, which double as the fallbacks
	Config.groupByApp              = GetRegKeyBool(L"groupByApp",              Config.groupByApp);
	Config.raiseAllWindows         = GetRegKeyBool(L"raiseAllWindows",         Config.raiseAllWindows);
	Config.fastSwitchingForApps    = GetRegKeyBool(L"fastSwitchingForApps",    Config.fastSwitchingForApps);
	Config.fastSwitchingForWindows = GetRegKeyBool(L"fastSwitchingForWindows", Config.fastSwitchingForWindows);
	Config.showSwitcherForApps     = GetRegKeyBool(L"showSwitcherForApps",     Config.showSwitcherForApps);
	Config.showSwitcherForWindows  = GetRegKeyBool(L"showSwitcherForWindows",  Config.showSwitcherForWindows);
	Config.wrapbump                = GetRegKeyBool(L"wrapbump",                Config.wrapbump);
```

Passing the field itself as the fallback is deliberate: the compiled-in default is written once, in the `Config = (struct ini){...}` literal at the top of `InitConfig`, and never duplicated. (The spec wrote the fallbacks as literal `true`/`false`; this is the same behavior without the duplication.)

- [ ] **Step 4: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make CC=gcc
```

Expected: builds, no errors, no new warnings.

- [ ] **Step 5: Behavior gate — the registry value is now honored**

The value from Step 1 is still in the registry. Run the new build:

```bash
./cmdtab.exe --autorun &
```

Hold `Alt` and tap the key above `Tab`. Expected **now**: the switcher window appears, unlike in Step 1.

- [ ] **Step 6: Behavior gate — the default still wins when the value is absent**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
reg delete "HKCU\\Software\\stianhoiland\\cmdtab" //v showSwitcherForWindows //f
./cmdtab.exe --autorun &
```

Hold `Alt` and tap the key above `Tab`. Expected: the switcher does **not** appear — `showSwitcherForWindows` fell back to its `false` default. If it appears, `GetRegKeyBool` is returning the `-1` as truthy and the fix is wrong.

- [ ] **Step 7: Behavior gate — nothing regressed for the two old settings**

With cmdtab running, hold `Alt`, tap `Tab` to open the switcher, and press `R` while still holding `Alt`. Release. Then:

```bash
reg query "HKCU\\Software\\stianhoiland\\cmdtab"
```

Expected: `raiseAllWindows` is present and has flipped from its previous value. Restart cmdtab and repeat `reg query`: the value is unchanged, and pressing `Alt-R` again flips it back.

- [ ] **Step 8: Commit**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
git add cmdtab.c
git commit -m "Read all seven behavior settings from the registry

GetRegKey returns -1 for a missing value and -1 is truthy, which the old
code used on purpose to default raiseAllWindows to on. Two of the newly
persisted settings default to false, so route every read through a
default-aware GetRegKeyBool instead.

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Tray icon with a right-click menu

After this task cmdtab is visible in the notification area and can be quit from there. The Settings item exists but does nothing yet — Task 3 wires it up.

**Files:**
- Modify: `cmdtab.c:12` — remove `#define NOMENUS`
- Modify: `cmdtab.c:46-56` — add a `shell32.lib` pragma inside the `#ifdef _MSC_VER` block
- Modify: `cmdtab.c` — tray constants and globals, `InitTrayIcon`, `RemoveTrayIcon`, `ShowTrayMenu`, `OnTrayMessage`, three new cases in `SwitcherWindowProcedure`, one call in `RunCmdTab`

**Interfaces:**
- Consumes: `Switcher` (the switcher `HWND`), `DestroyWindow`, and icon resource id `2` from `cmdtab.rc`.
- Produces:
  - `#define WM_CMDTAB_TRAY (WM_APP + 1)` — the tray callback message
  - `static void InitTrayIcon(handle instance)` — adds the icon; call once, after `InitSwitcherWindow`
  - `static void RemoveTrayIcon(void)` — deletes the icon and destroys the loaded `HICON`
  - `static void ShowTrayMenu(i32 x, i32 y)` — the right-click popup; Task 3 adds the Settings action to its `TRAY_MENU_SETTINGS` case
  - `static i64 OnTrayMessage(u32 event, i32 x, i32 y)` — Task 3 adds the Settings action to its `WM_LBUTTONUP` case

- [ ] **Step 1: Unblock the menu constants**

`cmdtab.c:12` has `#define NOMENUS`, which makes `windows.h` skip the `MF_*` menu-flag constants that `AppendMenuW` needs. Delete that one line:

```c
#define NOMENUS
```

Leave `NOICONS` and the rest alone — this code uses `MAKEINTRESOURCEW(2)`, not the `IDI_*` constants that `NOICONS` removes.

- [ ] **Step 2: Declare the shell32 dependency for the MSVC path**

Inside the existing `#ifdef _MSC_VER` block in `cmdtab.c` (around `cmdtab.c:46-56`), next to the other pragmas, add:

```c
#pragma comment(lib, "shell32.lib") // Shell_NotifyIconW
```

mingw-w64 links shell32 by default, so the Makefile needs no change; this line only keeps the MSVC build honest.

- [ ] **Step 3: Add the tray constants and state**

Add these `#define`s just above the `// Settings` / `struct ini` block (near `cmdtab.c:638`):

```c
#define WM_CMDTAB_TRAY     (WM_APP + 1) // Notification area callback message, sent to the switcher window
#define TRAY_MENU_SETTINGS 1
#define TRAY_MENU_QUIT     2
```

Then, in the block of file-scope statics under the `// GUI` comment (near `cmdtab.c:700`), add:

```c
static NOTIFYICONDATAW TrayIcon;    // Notification area icon, owned by the switcher window
static u32         TaskbarCreated;  // Message explorer.exe broadcasts when it restarts, so we can re-add the tray icon
```

- [ ] **Step 4: Write the tray icon lifecycle functions**

Add these three functions immediately before `static int RunCmdTab(...)` (near `cmdtab.c:824`), after `InitSwitcherWindow`:

```c
static void AddTrayIcon(void)
{
	Shell_NotifyIconW(NIM_ADD, &TrayIcon);
	TrayIcon.uVersion = NOTIFYICON_VERSION_4;
	Shell_NotifyIconW(NIM_SETVERSION, &TrayIcon); // Version 4 puts the event in lparam and the screen coords in wparam
}

static void InitTrayIcon(handle instance)
{
	// explorer.exe broadcasts this when it restarts, at which point every tray icon has to be added again
	TaskbarCreated = RegisterWindowMessageW(L"TaskbarCreated");

	TrayIcon = (NOTIFYICONDATAW){0};
	TrayIcon.cbSize = sizeof TrayIcon;
	TrayIcon.hWnd = Switcher;
	TrayIcon.uID = 1;
	TrayIcon.uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP;
	TrayIcon.uCallbackMessage = WM_CMDTAB_TRAY;
	// Resource id 2 is the ICON entry in cmdtab.rc
	TrayIcon.hIcon = LoadImageW(instance, MAKEINTRESOURCEW(2), IMAGE_ICON, GetSystemMetrics(SM_CXSMICON), GetSystemMetrics(SM_CYSMICON), 0);
	StringCchCopyW(TrayIcon.szTip, countof(TrayIcon.szTip), L"cmdtab");

	AddTrayIcon();
}

static void RemoveTrayIcon(void)
{
	Shell_NotifyIconW(NIM_DELETE, &TrayIcon);
	if (TrayIcon.hIcon) {
		DestroyIcon(TrayIcon.hIcon);
		TrayIcon.hIcon = NULL;
	}
}
```

- [ ] **Step 5: Write the popup menu and the tray message handler**

Add these two functions immediately before `static LRESULT CALLBACK SwitcherWindowProcedure(...)` at the bottom of the file (near `cmdtab.c:2072`), next to the other `On...` handlers:

```c
static void ShowTrayMenu(i32 x, i32 y)
{
	HMENU menu = CreatePopupMenu();
	if (!menu) return;
	AppendMenuW(menu, MF_STRING, TRAY_MENU_SETTINGS, L"Settings...");
	AppendMenuW(menu, MF_SEPARATOR, 0, NULL);
	AppendMenuW(menu, MF_STRING, TRAY_MENU_QUIT, L"Quit cmdtab");
	// Without this the menu stays on screen when the user clicks elsewhere
	SetForegroundWindow(Switcher);
	i32 choice = TrackPopupMenu(menu, TPM_RIGHTBUTTON | TPM_RETURNCMD | TPM_NONOTIFY, x, y, 0, Switcher, NULL);
	DestroyMenu(menu);
	switch (choice) {
		case TRAY_MENU_SETTINGS:
			break; // Wired up in the next task
		case TRAY_MENU_QUIT:
			DestroyWindow(Switcher); // The program's normal exit: the next GetMessageW fails and the message loop ends
			break;
	}
}

static i64 OnTrayMessage(u32 event, i32 x, i32 y)
{
	switch (event) {
		case WM_LBUTTONUP:
			break; // Wired up in the next task
		case WM_CONTEXTMENU:
			ShowTrayMenu(x, y);
			break;
	}
	return 1;
}
```

- [ ] **Step 6: Route the messages in the switcher window procedure**

In `SwitcherWindowProcedure` (`cmdtab.c:2072`), add a case above `case WM_CLOSE:`:

```c
		case WM_CMDTAB_TRAY:
			// NOTIFYICON_VERSION_4: event in the low word of lparam, screen coords in wparam
			return OnTrayMessage(LOWORD(lparam), (i16)LOWORD(wparam), (i16)HIWORD(wparam));
		case WM_DESTROY:
			RemoveTrayIcon();
			return 0;
```

and replace the `default:` case so the `TaskbarCreated` broadcast is caught:

```c
		default:
			if (TaskbarCreated && message == TaskbarCreated) {
				AddTrayIcon(); // explorer.exe restarted and forgot about us
				return 0;
			}
			return DefWindowProcW(hwnd, message, wparam, lparam);
```

The `TaskbarCreated &&` guard matters: the static is `0` until `InitTrayIcon` runs, and message `0` is `WM_NULL`, which the window would otherwise swallow.

- [ ] **Step 7: Create the icon at startup**

In `RunCmdTab`, immediately after `InitSwitcherWindow(instance);` (`cmdtab.c:846`), add:

```c
	InitTrayIcon(instance);
```

- [ ] **Step 8: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make CC=gcc
```

Expected: builds, no errors, no new warnings. A `MF_STRING undeclared` error means `#define NOMENUS` was not removed in Step 1.

- [ ] **Step 9: Behavior gate — the icon appears**

```bash
./cmdtab.exe --autorun &
```

Open the notification area overflow (the `^` chevron next to the clock). Expected: the cmdtab icon is there, and hovering it shows the tooltip `cmdtab`.

- [ ] **Step 10: Behavior gate — the menu works and dismisses**

Right-click the icon. Expected: a menu with `Settings...`, a separator, and `Quit cmdtab`. Click somewhere else on the desktop. Expected: the menu closes. (If it stays open, the `SetForegroundWindow` call is missing or misplaced.)

Right-click again and choose `Settings...`. Expected: nothing happens — correct for this task.

- [ ] **Step 11: Behavior gate — Alt-Tab still works, then Quit removes the icon**

Hold `Alt` and tap `Tab`. Expected: the switcher appears and switches windows as before.

Right-click the icon and choose `Quit cmdtab`. Expected: the icon disappears immediately and the process is gone:

```bash
tasklist //FI "IMAGENAME eq cmdtab.exe"
```

Expected: `INFO: No tasks are running which match the specified criteria.`

- [ ] **Step 12: Behavior gate — the icon survives an explorer restart**

```bash
./cmdtab.exe --autorun &
taskkill //F //IM explorer.exe
```

Windows restarts explorer on its own within a few seconds (if it does not, run `explorer.exe &`). Expected: once the taskbar comes back, the cmdtab icon is in the notification area again. If it is missing, the `TaskbarCreated` handling in Step 6 is wrong.

- [ ] **Step 13: Commit**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
git add cmdtab.c
git commit -m "Add a notification area icon with a right-click menu

The icon hangs off the switcher window so its callbacks pass the main
loop's hwnd filter, and it is re-added when explorer.exe restarts. The
menu can quit cmdtab; its Settings item is wired up next.

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Settings dialog

**Files:**
- Create: `resource.h`
- Modify: `cmdtab.rc` — two includes at the top, the dialog template at the bottom
- Modify: `Makefile:26-27` — the `cmdtab.o` rule
- Modify: `cmdtab.c` — `#include "resource.h"`, `GetAutorun`, `SetConfigBool`, `SettingsDialog` static, `SettingsDialogProcedure`, `ShowSettingsDialog`, and the two one-line hookups left behind in Task 2

**Interfaces:**
- Consumes: `Config` (`struct ini`), `SetRegKey`, `SetAutorun(bool enabled, u16 *keyname, u16 *args)`, `Switcher`, and the `TRAY_MENU_SETTINGS` / `WM_LBUTTONUP` cases Task 2 left empty.
- Produces:
  - `static bool GetAutorun(u16 *keyname)` — true when the `Run` value exists
  - `static void SetConfigBool(u16 *keyname, bool *setting, bool value)` — assigns and persists in one call
  - `static void ShowSettingsDialog(void)` — opens the modal dialog, or raises it if already open

- [ ] **Step 1: Create resource.h**

```c
#ifndef CMDTAB_RESOURCE_H
#define CMDTAB_RESOURCE_H

#define IDD_SETTINGS                 100

#define IDC_GROUP_BY_APP             1001
#define IDC_RAISE_ALL_WINDOWS        1002
#define IDC_FAST_SWITCHING_APPS      1003
#define IDC_FAST_SWITCHING_WINDOWS   1004
#define IDC_SHOW_SWITCHER_APPS       1005
#define IDC_SHOW_SWITCHER_WINDOWS    1006
#define IDC_WRAPBUMP                 1007
#define IDC_AUTORUN                  1008

#endif
```

- [ ] **Step 2: Add the dialog template to cmdtab.rc**

At the very top of `cmdtab.rc`, above the existing `1 24 "cmdtab.manifest"` line:

```c
#include <windows.h>
#include "resource.h"
```

`windows.h` is what supplies `DS_MODALFRAME`, `WS_POPUP` and friends to the resource compiler; it defines `RC_INVOKED`, so only the resource-relevant parts are pulled in.

At the very bottom of `cmdtab.rc`, after the existing `2 ICON "cmdtab.ico"` line:

```c
IDD_SETTINGS DIALOGEX 0, 0, 252, 196
STYLE DS_SETFONT | DS_MODALFRAME | WS_POPUP | WS_CAPTION | WS_SYSMENU
CAPTION "cmdtab settings"
FONT 9, "Segoe UI", 400, 0, 1
{
	GROUPBOX     "Behavior", -1, 7, 7, 238, 128
	AUTOCHECKBOX "&Group windows by app",                    IDC_GROUP_BY_APP,           16,  22, 220, 12
	AUTOCHECKBOX "&Raise all windows of the selected app",   IDC_RAISE_ALL_WINDOWS,      16,  38, 220, 12
	AUTOCHECKBOX "Switch &apps without showing a switcher",  IDC_FAST_SWITCHING_APPS,    16,  54, 220, 12
	AUTOCHECKBOX "Switch &windows without showing a switcher", IDC_FAST_SWITCHING_WINDOWS, 16, 70, 220, 12
	AUTOCHECKBOX "Show the switcher when switching a&pps",   IDC_SHOW_SWITCHER_APPS,     16,  86, 220, 12
	AUTOCHECKBOX "Show the switcher when switching win&dows", IDC_SHOW_SWITCHER_WINDOWS, 16, 102, 220, 12
	AUTOCHECKBOX "&Bump at the ends of the list instead of wrapping", IDC_WRAPBUMP,      16, 118, 220, 12
	GROUPBOX     "Startup", -1, 7, 141, 238, 30
	AUTOCHECKBOX "&Start cmdtab with Windows",               IDC_AUTORUN,                16, 155, 220, 12
	DEFPUSHBUTTON "OK",     IDOK,     141, 177, 50, 14
	PUSHBUTTON    "Cancel", IDCANCEL, 195, 177, 50, 14
}
```

- [ ] **Step 3: Fix the resource rule in the Makefile**

`Makefile:26-27` currently reads:

```make
cmdtab.o: cmdtab.rc
	windres $^ $@
```

Change it to:

```make
cmdtab.o: cmdtab.rc resource.h
	windres $< $@
```

Both edits are required. `$^` expands to *all* prerequisites, so adding `resource.h` while leaving `$^` would invoke `windres cmdtab.rc resource.h cmdtab.o` and fail. `$<` is the first prerequisite only.

- [ ] **Step 4: Compile gate for the resource script**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make CC=gcc
```

Expected: builds. The dialog is in the binary but nothing opens it yet. If windres reports an unknown keyword such as `DS_SETFONT`, the `#include <windows.h>` from Step 2 is missing or below the first resource statement.

- [ ] **Step 5: Include the ids in cmdtab.c**

In `cmdtab.c`, after the last system include (`#include <shellscalingapi.h>`, `cmdtab.c:45`), add:

```c
#include "resource.h"
```

- [ ] **Step 6: Add the autorun reader**

`SetAutorun` writes the `Run` value but nothing reads it, so the checkbox has no way to show its current state. Add this immediately after the closing brace of `SetAutorun` (`cmdtab.c:508`):

```c
static bool GetAutorun(u16 *keyname)
{
	u16 buffer[1024];
	ULONG size = sizeof buffer;
	// The value simply being present means autorun is on
	return !RegGetValueW(HKEY_CURRENT_USER, L"Software\\Microsoft\\Windows\\CurrentVersion\\Run", keyname, RRF_RT_REG_SZ, NULL, buffer, &size);
}
```

- [ ] **Step 7: Add the assign-and-persist helper**

Immediately after `GetRegKeyBool` (added in Task 1), add:

```c
static void SetConfigBool(u16 *keyname, bool *setting, bool value)
{
	*setting = value;
	SetRegKey(keyname, value);
}
```

- [ ] **Step 8: Add the dialog state and procedure**

Add `SettingsDialog` to the file-scope statics under the `// GUI` comment, next to `TrayIcon` from Task 2:

```c
static handle SettingsDialog; // Non-NULL while the settings dialog is open
```

Then add the dialog procedure and its opener immediately before `ShowTrayMenu` (added in Task 2, near the bottom of the file):

```c
static INT_PTR CALLBACK SettingsDialogProcedure(HWND hwnd, UINT message, WPARAM wparam, LPARAM lparam)
{
	switch (message) {
		case WM_INITDIALOG:
			SettingsDialog = hwnd;
			CheckDlgButton(hwnd, IDC_GROUP_BY_APP,           Config.groupByApp              ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_RAISE_ALL_WINDOWS,      Config.raiseAllWindows         ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_FAST_SWITCHING_APPS,    Config.fastSwitchingForApps    ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_FAST_SWITCHING_WINDOWS, Config.fastSwitchingForWindows ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_SHOW_SWITCHER_APPS,     Config.showSwitcherForApps     ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_SHOW_SWITCHER_WINDOWS,  Config.showSwitcherForWindows  ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_WRAPBUMP,               Config.wrapbump                ? BST_CHECKED : BST_UNCHECKED);
			CheckDlgButton(hwnd, IDC_AUTORUN,                GetAutorun(L"cmdtab")          ? BST_CHECKED : BST_UNCHECKED);
			return TRUE;
		case WM_COMMAND:
			switch (LOWORD(wparam)) {
				case IDOK:
					SetConfigBool(L"groupByApp",              &Config.groupByApp,              IsDlgButtonChecked(hwnd, IDC_GROUP_BY_APP)           == BST_CHECKED);
					SetConfigBool(L"raiseAllWindows",         &Config.raiseAllWindows,         IsDlgButtonChecked(hwnd, IDC_RAISE_ALL_WINDOWS)      == BST_CHECKED);
					SetConfigBool(L"fastSwitchingForApps",    &Config.fastSwitchingForApps,    IsDlgButtonChecked(hwnd, IDC_FAST_SWITCHING_APPS)    == BST_CHECKED);
					SetConfigBool(L"fastSwitchingForWindows", &Config.fastSwitchingForWindows, IsDlgButtonChecked(hwnd, IDC_FAST_SWITCHING_WINDOWS) == BST_CHECKED);
					SetConfigBool(L"showSwitcherForApps",     &Config.showSwitcherForApps,     IsDlgButtonChecked(hwnd, IDC_SHOW_SWITCHER_APPS)     == BST_CHECKED);
					SetConfigBool(L"showSwitcherForWindows",  &Config.showSwitcherForWindows,  IsDlgButtonChecked(hwnd, IDC_SHOW_SWITCHER_WINDOWS)  == BST_CHECKED);
					SetConfigBool(L"wrapbump",                &Config.wrapbump,                IsDlgButtonChecked(hwnd, IDC_WRAPBUMP)               == BST_CHECKED);
					SetAutorun(IsDlgButtonChecked(hwnd, IDC_AUTORUN) == BST_CHECKED, L"cmdtab", L"--autorun");
					SettingsDialog = NULL;
					EndDialog(hwnd, IDOK);
					return TRUE;
				case IDCANCEL:
					SettingsDialog = NULL;
					EndDialog(hwnd, IDCANCEL);
					return TRUE;
			}
			return FALSE;
		case WM_CLOSE:
			SettingsDialog = NULL;
			EndDialog(hwnd, IDCANCEL);
			return TRUE;
	}
	return FALSE;
}

static void ShowSettingsDialog(void)
{
	if (SettingsDialog) {
		SetForegroundWindow(SettingsDialog); // Already open — raise it instead of opening a second one
		return;
	}
	// Modal: DialogBoxParamW runs its own message loop, so the main loop's hwnd filter is left alone
	// and the low-level keyboard hook keeps being pumped, which is why Alt-Tab still works while this is open
	DialogBoxParamW(GetModuleHandleW(NULL), MAKEINTRESOURCEW(IDD_SETTINGS), Switcher, SettingsDialogProcedure, 0);
}
```

Nothing needs refreshing after OK: the keyboard hook reads `Config` at the moment it uses each field, and the switcher rebuilds its list through `UpdateApps()` every time it is shown.

- [ ] **Step 9: Wire the two hookups Task 2 left empty**

In `OnTrayMessage`, replace

```c
		case WM_LBUTTONUP:
			break; // Wired up in the next task
```

with

```c
		case WM_LBUTTONUP:
			ShowSettingsDialog();
			break;
```

In `ShowTrayMenu`, replace

```c
		case TRAY_MENU_SETTINGS:
			break; // Wired up in the next task
```

with

```c
		case TRAY_MENU_SETTINGS:
			ShowSettingsDialog();
			break;
```

- [ ] **Step 10: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make CC=gcc
```

Expected: builds, no errors, no new warnings.

- [ ] **Step 11: Behavior gate — the dialog opens and reflects the current state**

```bash
reg query "HKCU\\Software\\stianhoiland\\cmdtab"
./cmdtab.exe --autorun &
```

Left-click the tray icon. Expected: a themed dialog titled `cmdtab settings` with two group boxes, eight checkboxes, OK and Cancel. Every checkbox matches either the `reg query` output above or, for values not listed there, the defaults from the spec's table: group by app on, raise all windows on, switch apps without showing a switcher **off**, switch windows without showing a switcher **on**, show switcher for apps **on**, show switcher for windows **off**, bump at list ends **on**.

- [ ] **Step 12: Behavior gate — Alt-Tab still works while the dialog is open**

With the dialog still open, hold `Alt` and tap `Tab`. Expected: the switcher appears and switches windows. This is the thing the modal design exists to preserve; if it fails, the dialog is not pumping the hook and the design has to be revisited.

- [ ] **Step 13: Behavior gate — left-clicking again raises rather than duplicates**

With the dialog open, left-click the tray icon once more. Expected: the existing dialog comes to the front. There is exactly one `cmdtab settings` window.

- [ ] **Step 14: Behavior gate — Cancel changes nothing**

Tick `Bump at the ends of the list instead of wrapping` to the opposite of what it was, then press Cancel.

```bash
reg query "HKCU\\Software\\stianhoiland\\cmdtab"
```

Expected: no `wrapbump` value appears (or it is unchanged if one was already there). Reopen the dialog: the checkbox is back to its original state.

- [ ] **Step 15: Behavior gate — OK persists everything**

Open the dialog, tick `Switch apps without showing a switcher` and `Show the switcher when switching windows` on, then press OK.

```bash
reg query "HKCU\\Software\\stianhoiland\\cmdtab"
```

Expected: all seven value names are present, with `fastSwitchingForApps` = `0x1` and `showSwitcherForWindows` = `0x1`.

Now restart cmdtab and reopen the dialog:

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
./cmdtab.exe --autorun &
```

Expected: both boxes are still ticked. Untick them again and press OK, so the rest of the plan runs against defaults.

- [ ] **Step 16: Behavior gate — the startup checkbox drives the Run key**

Open the dialog, tick `Start cmdtab with Windows`, press OK.

```bash
reg query "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Run" //v cmdtab
```

Expected: a `REG_SZ` value whose data is the quoted path to `cmdtab.exe` followed by `--autorun`.

Reopen the dialog. Expected: the box is ticked (this is `GetAutorun` working). Untick it, press OK, and run the same `reg query`. Expected: `ERROR: The system was unable to find the specified registry key or value.`

- [ ] **Step 17: Behavior gate — `Alt-G` and `Alt-R` still agree with the dialog**

With cmdtab running, hold `Alt`, tap `Tab`, press `G`, release. Open the dialog. Expected: `Group windows by app` shows the flipped state. Press Cancel, then repeat with `R` and `Raise all windows of the selected app`.

- [ ] **Step 18: Commit**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
git add resource.h cmdtab.rc cmdtab.c Makefile
git commit -m "Add a settings dialog behind the tray icon

A modal dialog from a template in cmdtab.rc, so it pumps its own message
loop and the main loop stays untouched. Covers the seven behavior
booleans plus autorun; OK commits to the registry, Cancel discards.

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Drop the startup autorun prompt

Autorun is now a checkbox that can be changed at any time, so the modal `MessageBox` on every launch — the one whose text tells the user to relaunch the executable to change their mind — has no reason to exist. Removing it also removes a modal that blocks the message loop before it ever starts, which makes the program much easier to test.

**Files:**
- Modify: `cmdtab.c:770-780` — delete `HasAutorunLaunchArgument` and `AskAutorun`
- Modify: `cmdtab.c:833-835` — delete the call in `RunCmdTab`

**Interfaces:**
- Consumes: nothing new.
- Produces: nothing. This task only deletes.

- [ ] **Step 1: Delete the call**

In `RunCmdTab`, delete these three lines (`cmdtab.c:833-835`):

```c
	if (!HasAutorunLaunchArgument(args)) {
		AskAutorun();
	}
```

- [ ] **Step 2: Delete the two now-unused functions**

Delete `HasAutorunLaunchArgument` (`cmdtab.c:770-773`) and `AskAutorun` (`cmdtab.c:775-780`) in full:

```c
static bool HasAutorunLaunchArgument(u16 *args)
{
	return !!wcsstr(args, L"--autorun");
}

static void AskAutorun(void)
{
	#ifndef _DEBUG // Can't be bothered to be asked about this every single debug run
	SetAutorun(Ask(Switcher, L"Start cmdtab automatically?\nRelaunch cmdtab.exe to change your mind."), L"cmdtab", L"--autorun");
	#endif
}
```

Leaving them in place would trip `-Wunused-function` under the build's `-Wall`.

**Do not touch `Ask`** — `OnSwitcherClose` still uses it for the "Quit cmdtab?" confirmation. **Do not touch `HasDebugLaunchArgument`** — `RunCmdTab` still calls it. The `--autorun` argument stays accepted and ignored, so existing `Run` entries and the `make install` scheduled task keep working.

- [ ] **Step 3: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make CC=gcc
```

Expected: builds, no errors, no new warnings. A `-Wunused-function` warning means one of the two functions in Step 2 is still there.

- [ ] **Step 4: Behavior gate — no prompt, with or without the flag**

```bash
./cmdtab.exe &
```

Expected: **no** message box. The tray icon appears straight away. Previously this exact command showed "Start cmdtab automatically?" and blocked until answered.

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
./cmdtab.exe --autorun &
```

Expected: identical behavior — the flag is now inert.

- [ ] **Step 5: Behavior gate — the settings dialog is the only autorun control left**

Left-click the tray icon, tick `Start cmdtab with Windows`, press OK, then:

```bash
reg query "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Run" //v cmdtab
```

Expected: the value is there. Untick and OK, and it is gone again.

- [ ] **Step 6: Commit**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
git add cmdtab.c
git commit -m "Drop the startup autorun prompt

Autorun is a checkbox in the settings dialog now, so the launch-time
MessageBox that told the user to relaunch cmdtab.exe to change their mind
is redundant. The --autorun argument stays accepted and ignored.

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Release build and README

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: everything above.
- Produces: nothing code-facing.

- [ ] **Step 1: Read the README and find where behavior is documented**

```bash
grep -n "Alt-R\|Alt-G\|autorun\|settings\|Settings" README.md
```

Note which section lists the hotkeys and which, if any, describes first-run behavior.

- [ ] **Step 2: Document the tray icon and remove the stale autorun sentence**

Add a short section describing the notification-area icon: left click opens settings, right click offers Settings and Quit, and the settings dialog covers switching behavior and starting with Windows. Keep the existing `Alt-R` / `Alt-G` hotkey documentation — those still work and are faster than the dialog.

If the README says cmdtab asks about autorun on first launch, or that you must relaunch it to change that answer, delete that claim. It is no longer true.

- [ ] **Step 3: Full clean release build**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
make clean
make RELEASE=1 CC=gcc
```

Expected: builds from scratch with no errors. `RELEASE=1` adds `-mwindows`, so this is the first build that runs with no console attached — confirm the tray icon still appears and the dialog still opens.

- [ ] **Step 4: Final pass over the whole feature**

With the release build running: icon present, tooltip reads `cmdtab`, left click opens the dialog, right click offers both items, `Quit cmdtab` removes the icon and ends the process, and `Alt-Tab` switches windows throughout.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "Document the tray icon and settings dialog

Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>"
```
