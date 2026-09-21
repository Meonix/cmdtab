# Switcher Background Blur Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the area behind the cmdtab switcher blur instead of being hidden behind a flat opaque slab, with a checkbox in the settings dialog to turn it off.

**Architecture:** The switcher's off-screen buffer becomes a 32bpp DIB section so it carries an alpha channel, and a single sweep over its pixels after drawing marks the background transparent and everything drawn on top of it opaque. The blur itself comes from `SetWindowCompositionAttribute` with `ACCENT_ENABLE_ACRYLICBLURBEHIND`, an undocumented `user32` export resolved at runtime; when it is missing or fails, the same sweep marks every pixel opaque and the switcher renders exactly as it does today. The window stays an ordinary window — no layering — so `OnSwitcherPaint` is untouched.

**Tech Stack:** C99, Win32 (user32, gdi32, dwmapi), windres resource compiler, GNU make, mingw-w64 GCC under MSYS2 UCRT64.

**Spec:** `docs/superpowers/specs/2026-09-21-switcher-background-blur-design.md`

## Global Constraints

- Language is C99. The Makefile passes `-std=c99 -Wall -Wextra -pedantic -Wno-unused-parameter`. **Every task must build with zero new warnings.**
- Build command on this machine, from PowerShell:
  `$env:MSYSTEM="UCRT64"; & C:\msys64\usr\bin\bash.exe -lc "cd /d/Software/cmdtab && make CC=gcc"`
  `CC=gcc` is mandatory — the Makefile's default `CC = c99` has no binary under mingw. `CMakeLists.txt` is stale and is not an option.
- **Warning baseline for the debug build is 8 warnings**, all pre-existing: unused `oldPen`, `oldBrush`, `oldFont`, `SEL_OUTLINE`, `_`, `StringEndsWith`, `UWPAppsPath`, `UWPAppHostExe`. Anything beyond those eight is yours and must be fixed before committing.
- Unicode only. Call the `...W` form of every Win32 function and use `L"..."` literals.
- Indentation in `cmdtab.c` and `cmdtab.rc` is **tabs**, not spaces. Match it.
- The project is deliberately one translation unit. Do not create new `.c` files. `resource.h` holds nothing but `#define`s.
- Do not add libraries to `LDLIBS`. `user32` and `gdi32` are already linked (`gdi32` explicitly, `user32` by default under mingw-w64).
- Do not touch the main message loop at `cmdtab.c:907`. It filters on `Switcher`, and the program's only exit path depends on `GetMessageW` returning `-1` after `DestroyWindow(Switcher)`.
- The switcher background colour is `RGB(32, 32, 32)`, currently a local in `RedrawSwitcher` (cmdtab.c:1469). Task 1 promotes it to a file-scope macro; after that, **never hardcode 32 again** — the alpha sweep and the accent tint must both derive from that one macro.
- Tint value is `0x50202020` in AABBGGRR, i.e. `#202020` at alpha 80. Alpha 200 is a measured ceiling, not guessed: at alpha 200 the blur is invisible. The shipped value of 80 (lowered from an earlier 120) is a user preference chosen after looking at the result on screen, not a re-measurement — do not raise it without re-measuring the 200 ceiling first.

## Testing approach — read this before Task 1

**This repository has no test framework, no test directory and no automated tests.** It is a 2200-line Win32 GUI program whose visible behaviour is a composited window drawn by the desktop window manager. There is no way to assert on "is the background blurred" from code, and adding a test harness is out of scope and was not part of the approved design.

So the usual red/green TDD cycle does not apply here, and this plan does not pretend otherwise. Each task has a two-part gate and **both parts must pass before the commit step**:

1. **Compile gate (automated).** Build with the command above. Zero errors; zero warnings beyond the baseline of 8.
2. **Behaviour gate (manual, scripted).** Each task lists the exact commands to run and exactly what you must see on screen. Every one of them must be run.

**Before every manual gate, kill any running instance**, or the singleton mutex makes the new build show "cmdtab is already running" and you will be testing the old binary:

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
```

(Double slashes are for MSYS2 bash, which would otherwise rewrite `/F` into a path. From PowerShell use single slashes: `taskkill /F /IM cmdtab.exe`.)

**Set up a backdrop you can judge a blur against.** A blur over a dark, uniform desktop is indistinguishable from an opaque dark rectangle, and you will convince yourself the feature works when it does not. Before the Task 2 gate, open a window filling most of the screen showing something with hard, high-contrast colour edges — a photo, a colourful web page, or Paint filled with wide red and blue stripes. The switcher must be opened over *that* window.

**Reference images from the design spike** are in the session scratchpad at
`C:\Users\Mionix\AppData\Local\Temp\claude\D--Software-cmdtab\ac2e22f1-0e5c-420c-a5d6-f7574a2eb636\scratchpad\accent3.png`.
The leftmost third of the `acrylic` and `plainacry` panels is what a correct result looks like: a hard red/blue edge behind becomes a soft gradient. The `dwmsbt` panel, flat grey with no colour at all, is what the wrong API produces. If your result looks like `dwmsbt`, the accent policy is not being applied.

---

### Task 1: Give the off-screen buffer an alpha channel, changing nothing on screen

`ResizeSwitcher` builds the double-buffer with `CreateCompatibleBitmap`, which produces a device-format bitmap with no usable alpha channel. Swap it for a 32bpp top-down DIB section and keep a pointer to its pixels.

GDI does not maintain the alpha byte when it draws into a 32bpp DIB — `FillRect` and `DrawTextW` leave it at whatever was there, usually zero. So this task also adds the sweep that stamps the alpha byte, and for now that sweep writes 255 everywhere. The result must be pixel-identical to the current build. Task 2 makes the sweep conditional.

**Files:**
- Modify: `cmdtab.c:698-710` — `struct gui`, add the pixel pointer
- Modify: `cmdtab.c:1355-1362` — `ResizeSwitcher`, the bitmap creation block
- Modify: `cmdtab.c:1465-1470` — `RedrawSwitcher`, promote the background colour to a macro
- Modify: `cmdtab.c:1548-1550` — `RedrawSwitcher`, call the new sweep before `RedrawWindow`
- Create: nothing

**Interfaces:**
- Consumes: `struct gui DrawingDims` and `handle DrawingBitmap`, `handle DrawingContext`, all already file-scope globals.
- Produces:
  - `#define SWITCHER_BG RGB(32, 32, 32)` — file-scope macro, the single source of truth for the switcher background colour. Task 2 derives both the alpha key and the accent tint from it.
  - `u8 *DrawingDims.drawBits` — start of the DIB pixel buffer, BGRA order, top-down, no padding (a 32bpp DIB's stride is always `width * 4`). NULL until the first `ResizeSwitcher`.
  - `static void SetSwitcherAlpha(void)` — stamps the alpha byte of every pixel in the buffer. Task 2 rewrites its body.

- [ ] **Step 1: Add the pixel pointer to `struct gui`**

In `cmdtab.c:698`, `struct gui` ends with `u32 selVertOff;`. Add one field:

```c
struct gui {
	f32 drawScale; // DPI scale of the monitor where the cursor is, which is where the switcher will be displayed
	RECT drawRect; // Size of the scaled off-screen bitmap
	u8 *drawBits;  // Pixels of the off-screen bitmap, BGRA, top-down. Owned by DrawingBitmap
	u32 switcherHorzMargin;
	u32 switcherVertMargin;
	u32 iconSize;
	u32 iconHorzMargin;
	u32 selOutline;
	u32 selRadius;
	u32 selHorzOff;
	u32 selVertOff;
};
```

- [ ] **Step 2: Create a DIB section instead of a compatible bitmap**

In `ResizeSwitcher`, replace this block (currently `cmdtab.c:1355-1362`):

```c
		handle context = GetDC(Switcher);
		DeleteDC(DrawingContext);
		DrawingContext = CreateCompatibleDC(context);
		DeleteObject(DrawingBitmap);
		DrawingBitmap = CreateCompatibleBitmap(context, DrawingDims.drawRect.right - DrawingDims.drawRect.left, DrawingDims.drawRect.bottom - DrawingDims.drawRect.top);
		handle oldBitmap = (handle)SelectObject(DrawingContext, DrawingBitmap);
		DeleteObject(oldBitmap);
		ReleaseDC(Switcher, context);
```

with:

```c
		handle context = GetDC(Switcher);
		DeleteDC(DrawingContext);
		DrawingContext = CreateCompatibleDC(context);
		DeleteObject(DrawingBitmap);
		// A DIB section rather than a compatible bitmap, because the switcher
		// needs an alpha channel it can write to (see SetSwitcherAlpha)
		BITMAPINFO info = {0};
		info.bmiHeader.biSize        = sizeof info.bmiHeader;
		info.bmiHeader.biWidth       = DrawingDims.drawRect.right - DrawingDims.drawRect.left;
		info.bmiHeader.biHeight      = -(DrawingDims.drawRect.bottom - DrawingDims.drawRect.top); // Negative means top-down, so row 0 is the top row
		info.bmiHeader.biPlanes      = 1;
		info.bmiHeader.biBitCount    = 32;
		info.bmiHeader.biCompression = BI_RGB;
		DrawingDims.drawBits = NULL;
		DrawingBitmap = CreateDIBSection(context, &info, DIB_RGB_COLORS, (void **)&DrawingDims.drawBits, NULL, 0);
		handle oldBitmap = (handle)SelectObject(DrawingContext, DrawingBitmap);
		DeleteObject(oldBitmap);
		ReleaseDC(Switcher, context);
```

Note `DrawingDims.drawBits` is assigned before the call as well: `CreateDIBSection` leaves it untouched on failure, and a stale pointer into a deleted bitmap would be far worse than a NULL one.

- [ ] **Step 3: Promote the background colour to a macro**

Put this directly above `static handle Mutex;`, the first line of the globals block (currently `cmdtab.c:711`). It goes there rather than next to `RedrawSwitcher` because Task 2 needs it from a function defined much earlier in the file:

```c
// The switcher background. SetSwitcherAlpha uses it as the key for which pixels
// are background, and ApplySwitcherBlur tints the acrylic with it, so it has to
// be one value in one place
#define SWITCHER_BG RGB(32, 32, 32)
```

Then in `RedrawSwitcher`, change `cmdtab.c:1469` from:

```c
	COLORREF WIN_COLOR_BG = RGB(32, 32, 32); // dark mode?
```

to:

```c
	COLORREF WIN_COLOR_BG = SWITCHER_BG; // dark mode?
```

- [ ] **Step 4: Write the alpha sweep**

Add this function directly above `static void RedrawSwitcher(void)` (currently `cmdtab.c:1465`):

```c
// GDI does not maintain the alpha byte when drawing into a 32bpp DIB, so after
// everything is drawn the alpha channel is written here in one sweep
static void SetSwitcherAlpha(void)
{
	GdiFlush(); // GDI batches drawing calls; the pixels are not final until this returns
	u8 *pixel = DrawingDims.drawBits;
	if (!pixel) {
		return;
	}
	iz count = (iz)(DrawingDims.drawRect.right - DrawingDims.drawRect.left)
	         * (iz)(DrawingDims.drawRect.bottom - DrawingDims.drawRect.top);
	for (iz i = 0; i < count; i++, pixel += 4) {
		pixel[3] = 255;
	}
}
```

- [ ] **Step 5: Call the sweep at the end of `RedrawSwitcher`**

At the end of `RedrawSwitcher` (currently `cmdtab.c:1548-1550`), change:

```c
	// Invalidate window rectangle
	RedrawWindow(Switcher, NULL, NULL, RDW_INVALIDATE | RDW_ERASE | RDW_UPDATENOW);
```

to:

```c
	SetSwitcherAlpha();

	// Invalidate window rectangle
	RedrawWindow(Switcher, NULL, NULL, RDW_INVALIDATE | RDW_ERASE | RDW_UPDATENOW);
```

- [ ] **Step 6: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
```

Then from PowerShell:

```
$env:MSYSTEM="UCRT64"; & C:\msys64\usr\bin\bash.exe -lc "cd /d/Software/cmdtab && make CC=gcc"
```

Expected: builds, and the warning count is still 8.

- [ ] **Step 7: Behaviour gate — nothing may have changed**

Run `./cmdtab.exe`, hold Alt and press Tab.

You must see:
- The switcher appears with the same flat dark background as before.
- Icons, titles and the selection rectangle look exactly as they did.
- No transparent, black or garbled areas anywhere in the window.

If any part of the window is transparent or black, the sweep is not running or `drawBits` is NULL — fix before committing.

- [ ] **Step 8: Commit**

```bash
git add cmdtab.c
git commit -m "Draw the switcher into a DIB section with an alpha channel"
```

---

### Task 2: Turn on the acrylic blur

Resolve the undocumented `SetWindowCompositionAttribute`, apply an acrylic accent policy to the switcher window, and make the alpha sweep punch the background through so the blur is visible. Also fix the text antialiasing, which only becomes a problem once the background is transparent.

**Files:**
- Modify: `cmdtab.c:705-711` — above the globals block, the undocumented API declarations
- Modify: `cmdtab.c:730-740` — the GUI globals block, add the font handle and the blur state
- Modify: `cmdtab.c:829-846` — above and inside `InitSwitcherWindow`
- Modify: `cmdtab.c` — `SetSwitcherAlpha`, rewrite the loop
- Modify: `cmdtab.c:1500` — `RedrawSwitcher`, the font selection

**Interfaces:**
- Consumes: `SWITCHER_BG`, `DrawingDims.drawBits` and `SetSwitcherAlpha` from Task 1; `handle Switcher`.
- Produces:
  - `static bool BlurActive` — true only when the acrylic is actually in effect. Read by `SetSwitcherAlpha`.
  - `static void ApplySwitcherBlur(bool on)` — applies or clears the accent policy and sets `BlurActive`. Task 3 calls it from the settings dialog.
  - `static HFONT DrawingFont` — the title font, cached, created once.

- [ ] **Step 1: Declare the undocumented API**

This goes directly above the `#define SWITCHER_BG` line Task 1 added, which sits just above the globals block (currently around `cmdtab.c:705`). It has to be above the globals because Step 2 declares a global of the function-pointer type defined here.

Add:

```c
// SetWindowCompositionAttribute is an undocumented user32 export, and the only
// way to blur what is behind a window. The documented DWMWA_SYSTEMBACKDROP_TYPE
// returns S_OK but only paints a flat sheet - measured on Windows 11 build
// 26200 while designing this. No SDK header declares any of the following, so
// it is declared here. It is resolved at runtime and its absence is not an
// error: see ApplySwitcherBlur
#define ACCENT_DISABLED                 0
#define ACCENT_ENABLE_ACRYLICBLURBEHIND 4
#define ACCENT_FLAG_FILL_WINDOW         2 // Apply the gradient colour over the whole window, not just its border
#define WCA_ACCENT_POLICY               19

struct accent_policy {
	u32 state;
	u32 flags;
	u32 gradient; // AABBGGRR, note the byte order is not COLORREF's
	u32 animation;
};

struct composition_attribute {
	u32 attribute;
	void *data;
	iz size;
};

typedef i32(__stdcall *SetWindowCompositionAttributeFn)(handle, struct composition_attribute *);
```

- [ ] **Step 2: Add the globals**

In the GUI globals block, after `static HPEN NoneOutline;` (currently `cmdtab.c:733`), add:

```c
static HFONT       DrawingFont;     // Title font. Same face as DEFAULT_GUI_FONT but greyscale-antialiased, see RedrawSwitcher
static bool        BlurActive;      // Is the acrylic blur actually in effect? False means draw opaque, exactly as cmdtab always did
static SetWindowCompositionAttributeFn SetWindowCompositionAttribute; // NULL on Windows without the undocumented export
```

- [ ] **Step 3: Write `ApplySwitcherBlur`**

Directly above `static void InitSwitcherWindow(handle instance)` (currently `cmdtab.c:829`), add:

```c
static void ApplySwitcherBlur(bool on)
{
	BlurActive = false;
	if (!SetWindowCompositionAttribute) {
		return; // Windows too old, or Microsoft finally dropped the export. Draw opaque
	}
	struct accent_policy policy = {
		on ? ACCENT_ENABLE_ACRYLICBLURBEHIND : ACCENT_DISABLED,
		ACCENT_FLAG_FILL_WINDOW,
		// The switcher's own background colour at alpha 80. This is the only
		// tint: SetSwitcherAlpha leaves background pixels fully transparent so
		// the two do not stack. At alpha 200 the blur stops being visible
		(80u << 24) | (GetBValue(SWITCHER_BG) << 16) | (GetGValue(SWITCHER_BG) << 8) | GetRValue(SWITCHER_BG),
		0,
	};
	struct composition_attribute attribute = {WCA_ACCENT_POLICY, &policy, sizeof policy};
	i32 applied = SetWindowCompositionAttribute(Switcher, &attribute);
	BlurActive = on && applied;
}
```

- [ ] **Step 4: Resolve the export and apply the policy**

At the end of `InitSwitcherWindow`, after the `DwmSetWindowAttribute` call (currently `cmdtab.c:846`), add:

```c
	// Acrylic blur behind the switcher
	SetWindowCompositionAttribute = (SetWindowCompositionAttributeFn)(void *)GetProcAddress(GetModuleHandleW(L"user32.dll"), "SetWindowCompositionAttribute");
	ApplySwitcherBlur(true);
```

`ApplySwitcherBlur(true)` is temporary — Task 3 replaces `true` with `Config.blurBackground`.

- [ ] **Step 5: Make the alpha sweep punch the background through**

Replace the body of `SetSwitcherAlpha` from Task 1 with:

```c
static void SetSwitcherAlpha(void)
{
	GdiFlush(); // GDI batches drawing calls; the pixels are not final until this returns
	u8 *pixel = DrawingDims.drawBits;
	if (!pixel) {
		return;
	}
	iz count = (iz)(DrawingDims.drawRect.right - DrawingDims.drawRect.left)
	         * (iz)(DrawingDims.drawRect.bottom - DrawingDims.drawRect.top);
	if (!BlurActive) {
		for (iz i = 0; i < count; i++, pixel += 4) {
			pixel[3] = 255; // Opaque everywhere, exactly as before the blur existed
		}
		return;
	}
	// Pixels still showing the background colour are background: punch them
	// through so the acrylic below shows. Everything drawn on top of them -
	// icons, title text and its antialiased edges, the selection rectangle -
	// differs from it and stays opaque
	u8 b = GetBValue(SWITCHER_BG), g = GetGValue(SWITCHER_BG), r = GetRValue(SWITCHER_BG);
	for (iz i = 0; i < count; i++, pixel += 4) {
		if (pixel[0] == b && pixel[1] == g && pixel[2] == r) {
			pixel[0] = pixel[1] = pixel[2] = pixel[3] = 0;
		} else {
			pixel[3] = 255;
		}
	}
}
```

- [ ] **Step 6: Switch the title font to greyscale antialiasing**

`DEFAULT_GUI_FONT` renders with ClearType, which emits colour-fringed subpixels. Those differ from the background colour, so the sweep in Step 5 keeps them fully opaque and they appear as loose red and blue dots around every title. Replace `cmdtab.c:1500`:

```c
	// Select text font for DrawTextW (used to draw title text)
	HFONT oldFont = (HFONT)SelectObject(DrawingContext, GetStockObject(DEFAULT_GUI_FONT));
```

with:

```c
	// Select text font for DrawTextW (used to draw title text).
	// Same face as DEFAULT_GUI_FONT, but greyscale-antialiased: ClearType's
	// coloured subpixels would survive SetSwitcherAlpha as opaque dots
	if (!DrawingFont) {
		LOGFONTW logfont = {0};
		GetObjectW(GetStockObject(DEFAULT_GUI_FONT), sizeof logfont, &logfont);
		logfont.lfQuality = ANTIALIASED_QUALITY;
		DrawingFont = CreateFontIndirectW(&logfont);
	}
	HFONT oldFont = (HFONT)SelectObject(DrawingContext, DrawingFont);
```

- [ ] **Step 7: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
```

From PowerShell:

```
$env:MSYSTEM="UCRT64"; & C:\msys64\usr\bin\bash.exe -lc "cd /d/Software/cmdtab && make CC=gcc"
```

Expected: builds, warning count still 8.

- [ ] **Step 8: Behaviour gate — the blur must be visible and the corners must survive**

Put a high-contrast, hard-edged colourful window on screen first (see "Testing approach"). Run `./cmdtab.exe`, hold Alt and press Tab over that window.

You must see, all four:
1. **The colours behind the switcher bleed through as soft, smeared shapes.** A hard edge behind becomes a gradient. A flat grey panel with no colour at all means the accent policy did not take — compare against the reference image named in "Testing approach".
2. **The switcher's corners are still rounded.** `DWMWA_WINDOW_CORNER_PREFERENCE` was confirmed to survive per-pixel alpha but was never tested together with an accent policy. If the corners are now square, stop and report it — the fix is to force alpha 0 in the corner arcs inside `SetSwitcherAlpha`, which is a design change and needs a decision, not an improvised patch.
3. **App titles have no coloured speckle around them.** If they do, Step 6 did not take effect.
4. **Icons look unchanged** — no square dark box around them, no missing transparency.

Then check the fallback: temporarily change `ApplySwitcherBlur(true)` to `ApplySwitcherBlur(false)` in `InitSwitcherWindow`, rebuild, and confirm the switcher is opaque and identical to the Task 1 build. Change it back to `true`, rebuild, and confirm the blur returns.

- [ ] **Step 9: Commit**

```bash
git add cmdtab.c
git commit -m "Blur the desktop behind the switcher"
```

---

### Task 3: Put the blur behind a settings checkbox

Add `blurBackground` as the eighth stored setting and a ninth checkbox in the settings dialog, in its own *Appearance* group. Document it.

The spec says the dialog should call `ApplySwitcherBlur` and then force a redraw. The redraw is unnecessary: `ShowSwitcher` (`cmdtab.c:1597`) already calls `RedrawSwitcher()` every time the switcher opens, so the next Alt-Tab redraws with the new `BlurActive` anyway. Only `ApplySwitcherBlur` is called here.

**Files:**
- Modify: `resource.h` — one `#define`
- Modify: `cmdtab.rc:52-66` — the `IDD_SETTINGS` dialog template
- Modify: `cmdtab.c:665-688` — `struct ini`
- Modify: `cmdtab.c:753-800` — `InitConfig`, the defaults literal and the registry reads
- Modify: `cmdtab.c:846` — `InitSwitcherWindow`, replace the temporary `true`
- Modify: `cmdtab.c:2128-2155` — `SettingsDialogProcedure`
- Modify: `README.md` — the Settings section

**Interfaces:**
- Consumes: `ApplySwitcherBlur(bool)` from Task 2; `GetRegKeyBool(u16 *keyname, bool fallback)` (`cmdtab.c:540`) and `SetConfigBool(u16 *keyname, bool *setting, bool value)` (`cmdtab.c:547`), both already in the file.
- Produces: `Config.blurBackground`, and the `REG_DWORD` value `blurBackground` under `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab`.

- [ ] **Step 1: Add the control id**

In `resource.h`, after `#define IDC_AUTORUN 1008`:

```c
#define IDC_BLUR_BACKGROUND          1009
```

- [ ] **Step 2: Add the checkbox to the dialog template**

In `cmdtab.rc`, replace the whole `IDD_SETTINGS` block with this. The *Appearance* group is new, *Startup* and the buttons move down by 34, and the dialog grows from 196 to 230:

```
IDD_SETTINGS DIALOGEX 0, 0, 252, 230
STYLE DS_SETFONT | DS_MODALFRAME | WS_POPUP | WS_CAPTION | WS_SYSMENU
CAPTION "cmdtab settings"
FONT 9, "Segoe UI", 400, 0, 1
{
	GROUPBOX     "Behavior", -1, 7, 7, 238, 128
	AUTOCHECKBOX "&Group windows by app",                            IDC_GROUP_BY_APP,           16,  22, 220, 12
	AUTOCHECKBOX "&Raise all windows of the selected app",           IDC_RAISE_ALL_WINDOWS,      16,  38, 220, 12
	AUTOCHECKBOX "Switch &apps without showing a switcher",          IDC_FAST_SWITCHING_APPS,    16,  54, 220, 12
	AUTOCHECKBOX "Switch &windows without showing a switcher",       IDC_FAST_SWITCHING_WINDOWS, 16,  70, 220, 12
	AUTOCHECKBOX "Show the switcher when switching a&pps",           IDC_SHOW_SWITCHER_APPS,     16,  86, 220, 12
	AUTOCHECKBOX "Show the switcher when switching win&dows",        IDC_SHOW_SWITCHER_WINDOWS,  16, 102, 220, 12
	AUTOCHECKBOX "&Bump at the ends of the list instead of wrapping", IDC_WRAPBUMP,              16, 118, 220, 12
	GROUPBOX     "Appearance", -1, 7, 141, 238, 30
	AUTOCHECKBOX "Bl&ur the desktop behind the switcher",            IDC_BLUR_BACKGROUND,        16, 155, 220, 12
	GROUPBOX     "Startup", -1, 7, 175, 238, 30
	AUTOCHECKBOX "&Start cmdtab with Windows",                       IDC_AUTORUN,                16, 189, 220, 12
	DEFPUSHBUTTON "OK",     IDOK,     141, 211, 50, 14
	PUSHBUTTON    "Cancel", IDCANCEL, 195, 211, 50, 14
}
```

- [ ] **Step 3: Add the setting to `struct ini`**

In `struct ini` the *Appearance* section starts with `bool darkmode;` (currently `cmdtab.c:678`). Add one line after it:

```c
	// Appearance
	bool darkmode;
	bool blurBackground;
	u32 switcherHorzMargin;
```

- [ ] **Step 4: Default it on and read it from the registry**

In `InitConfig`, after `.darkmode = false,` (currently `cmdtab.c:767`):

```c
		.darkmode                = false,
		.blurBackground          = true,
```

Then, with the other `GetRegKeyBool` calls at the end of `InitConfig`, after the `wrapbump` line (currently `cmdtab.c:795`):

```c
	Config.blurBackground          = GetRegKeyBool(L"blurBackground",          Config.blurBackground);
```

- [ ] **Step 5: Honour the setting at startup**

In `InitSwitcherWindow`, replace the temporary line from Task 2:

```c
	ApplySwitcherBlur(true);
```

with:

```c
	ApplySwitcherBlur(Config.blurBackground);
```

`InitConfig()` runs before `InitSwitcherWindow()` in `RunCmdTab`, so `Config.blurBackground` is already loaded at this point.

- [ ] **Step 6: Wire up the dialog**

In `SettingsDialogProcedure`, under `WM_INITDIALOG`, after the `IDC_WRAPBUMP` line:

```c
			CheckDlgButton(hwnd, IDC_BLUR_BACKGROUND,        Config.blurBackground          ? BST_CHECKED : BST_UNCHECKED);
```

Under `case IDOK:`, after the `wrapbump` line and before `SetAutorun`:

```c
					SetConfigBool(L"blurBackground",          &Config.blurBackground,          IsDlgButtonChecked(hwnd, IDC_BLUR_BACKGROUND)        == BST_CHECKED);
					ApplySwitcherBlur(Config.blurBackground);
```

- [ ] **Step 7: Update the README**

In `README.md`, the Settings section currently ends its second paragraph with "...plus a checkbox for starting **cmdtab** with Windows." Replace that paragraph's tail and the sentence that follows the next paragraph so the section reads:

```markdown
The settings window covers switching behavior—app grouping, raising all windows of an app, whether each hotkey shows the switcher or switches straight away, and wrap bump—plus whether the desktop behind the switcher is blurred, and a checkbox for starting **cmdtab** with Windows. Settings are stored under `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab` and survive a restart. The in-switcher hotkeys `Alt-G` and `Alt-R` still work and are faster if you only want to flip one thing.

The blur needs Windows 10 version 1803 or newer with *Transparency effects* turned on in Windows Settings. Where it is unavailable the switcher simply draws its usual solid background.

Hotkeys, the blacklist and the rest of the switcher's appearance are not in the settings window yet; those still live in `InitConfig` in `cmdtab.c`.
```

Also add one bullet to the Features list, after the tray icon bullet:

```markdown
- The desktop behind the switcher is blurred, and can be turned off in settings
```

- [ ] **Step 8: Compile gate**

```bash
taskkill //F //IM cmdtab.exe 2>/dev/null || true
```

From PowerShell:

```
$env:MSYSTEM="UCRT64"; & C:\msys64\usr\bin\bash.exe -lc "cd /d/Software/cmdtab && make CC=gcc"
```

Expected: builds, warning count still 8. `cmdtab.o` is rebuilt because `cmdtab.rc` and `resource.h` changed — if `windres` errors, the dialog template is malformed.

- [ ] **Step 9: Behaviour gate**

Run `./cmdtab.exe`. With a colourful window on screen:

1. Alt-Tab: the background is blurred (the setting defaults to on).
2. Left-click the tray icon. The dialog opens with an *Appearance* group holding one ticked checkbox, "Blur the desktop behind the switcher". Nothing is clipped, no control overlaps another, both buttons are fully inside the dialog.
3. Untick it, press OK, Alt-Tab again: **opaque, immediately, without restarting cmdtab.**
4. Reopen the dialog: the checkbox is still unticked.
5. Quit cmdtab from the tray menu, run it again, Alt-Tab: still opaque. The setting survived the restart.
6. Check the registry:
   ```
   reg query "HKCU\Software\stianhoiland\cmdtab" /v blurBackground
   ```
   Expected: `blurBackground    REG_DWORD    0x0`
7. Tick it again, press OK, Alt-Tab: blurred again.
8. Turn off *Transparency effects* in Windows Settings (Settings → Personalization → Colors) and Alt-Tab: the switcher must still be readable. Turn it back on afterwards.

- [ ] **Step 10: Mark the spec implemented and commit**

In `docs/superpowers/specs/2026-09-21-switcher-background-blur-design.md`, change the status line from `**Status:** approved design, not yet implemented` to `**Status:** implemented`.

```bash
git add resource.h cmdtab.rc cmdtab.c README.md docs/superpowers/specs/2026-09-21-switcher-background-blur-design.md
git commit -m "Add a settings checkbox for the switcher background blur"
```
