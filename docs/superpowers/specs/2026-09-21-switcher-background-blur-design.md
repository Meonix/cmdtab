# Blurred background for the cmdtab switcher

**Status:** implemented
**Date:** 2026-09-21

## Problem

The switcher is an opaque rectangle. `RedrawSwitcher` fills its whole client
area with a solid `RGB(32, 32, 32)` brush (cmdtab.c:1498) and `OnSwitcherPaint`
blits that straight to the screen (cmdtab.c:2021). Whatever the user was
looking at disappears behind a flat dark slab for as long as Alt-Tab is held.

Windows 11's own Alt-Tab blurs what is behind it. cmdtab should too.

## What was measured before this was designed

The obvious approach — `DwmSetWindowAttribute` with
`DWMWA_SYSTEMBACKDROP_TYPE = DWMSBT_TRANSIENTWINDOW` — does not work. It is the
documented Windows 11 backdrop API and it returns `S_OK`, but it does not blur
what is behind the window.

Four candidates were built and photographed on this machine (Windows 11 build
26200, dark mode, *Transparency effects* on), each drawn over a backdrop of
hard-edged red and blue blocks:

| Candidate | Blurs the content behind? |
|---|---|
| `DWMSBT_TRANSIENTWINDOW`, layered window | No. A flat grey sheet; no colour from the backdrop survives |
| `DWMSBT_TRANSIENTWINDOW` + `DwmExtendFrameIntoClientArea(-1)`, ordinary window | No. Flat grey steps |
| `ACCENT_ENABLE_BLURBEHIND` | Yes, but crushed to near black. Unusable |
| `ACCENT_ENABLE_ACRYLICBLURBEHIND` | **Yes.** The hard red/blue edge behind becomes a soft gradient |

Two further findings from the same probes, both of which shape the design:

- **A layered window is not needed.** The ordinary-window variant looked
  identical to the layered one, which means `BitBlt` from a 32bpp DIB carries
  the alpha channel through to the screen. `OnSwitcherPaint` can stay exactly
  as it is.
- **A heavy tint destroys the effect.** At alpha 200 the blur is invisible; the
  panel is as flat as today's opaque fill. Alpha 120 is where the blur reads
  while the panel still looks dark.

The probes were throwaway and are not part of the repository.

## Decisions

| Question | Decision |
|---|---|
| Blur mechanism | `SetWindowCompositionAttribute` with `ACCENT_ENABLE_ACRYLICBLURBEHIND` |
| When it is unavailable | Fall back to today's opaque drawing |
| Tint | cmdtab's own `#202020` at alpha 120, carried by the accent policy's gradient colour |
| User control | A checkbox in the settings dialog, default on |
| Window type | Ordinary window, unchanged `WM_PAINT` + `BitBlt` |

`SetWindowCompositionAttribute` is an undocumented `user32` export. It has been
stable since Windows 10 1803 and is widely used by Win32 applications, but
Microsoft promises nothing about it. This was put to the user explicitly after
the measurements above; the alternative that delivers real blur without it is a
hand-written screen-capture blur, which was declined as more code for a worse
result. The risk is contained by resolving the export through `GetProcAddress`
and treating failure as "no blur", never as an error.

## Design

### 1. The drawing surface gains an alpha channel

`ResizeSwitcher` (cmdtab.c:1355) creates the off-screen buffer with
`CreateCompatibleBitmap`, which has no usable alpha channel. Replace it with
`CreateDIBSection` for a 32bpp top-down DIB: `biBitCount = 32`,
`biCompression = BI_RGB`, `biHeight` negative.

`struct gui` gains a pointer to the DIB bits so the fixup pass below can reach
them. Width and height are already tracked in `DrawingDims.drawRect`.

`OnSwitcherPaint` is not touched.

### 2. An alpha fixup pass

`DrawApp` and the body of `RedrawSwitcher` are not touched either. Instead
`RedrawSwitcher` calls one new function after all drawing is finished and
before `RedrawWindow` (cmdtab.c:1550):

```c
GdiFlush();
for each pixel:
    if (b, g, r) == (32, 32, 32)  ->  write (0, 0, 0, 0)   // background: punch through
    else                          ->  write alpha = 255    // icons, text, selection
```

The rule works because of how the existing code already draws:

- `DrawIconEx` is passed `backgroundBrush` (cmdtab.c:1407), so an icon's
  transparent pixels come out as exactly the background colour and become
  holes on their own.
- Text is drawn by `DrawTextW` onto the opaque `#202020` fill before the pass
  runs, so glyph edges are blends of `#EBEBEB` and `#202020`. They differ from
  the background colour, so they are stamped fully opaque and look exactly as
  they do today.
- `RoundRect` does not antialias, so the selection rectangle produces no
  in-between pixels.

Accepted cost: an icon pixel that happens to be exactly `RGB(32, 32, 32)`
becomes transparent. At icon scale this is not noticeable. The pass is one
sweep over roughly 200k pixels, well under a millisecond.

**Font quality must change.** `RedrawSwitcher` selects
`GetStockObject(DEFAULT_GUI_FONT)` (cmdtab.c:1500), which renders with
ClearType. ClearType emits colour-fringed subpixels that do not match the
background colour, so they survive the fixup as opaque red and blue dots on
what should be a transparent area — clearly visible in the probe images. Fix:
read the stock font's `LOGFONT` with `GetObjectW`, set
`lfQuality = ANTIALIASED_QUALITY`, and recreate it with `CreateFontIndirectW`.
Same typeface, greyscale antialiasing.

### 3. Turning the blur on

Declare the undocumented structures locally, since no SDK header has them:

```c
typedef enum { ACCENT_DISABLED = 0, ACCENT_ENABLE_ACRYLICBLURBEHIND = 4 } ACCENT_STATE;
typedef struct { ACCENT_STATE AccentState; DWORD AccentFlags; DWORD GradientColor; DWORD AnimationId; } ACCENT_POLICY;
typedef struct { DWORD Attrib; PVOID pvData; SIZE_T cbData; } WINDOWCOMPOSITIONATTRIBDATA;
```

`InitSwitcherWindow` (cmdtab.c:830) resolves the export once:

```c
SetWindowCompositionAttribute = GetProcAddress(GetModuleHandleW(L"user32.dll"),
                                               "SetWindowCompositionAttribute");
```

A new `ApplySwitcherBlur(bool on)` builds the policy and calls it:

```c
ACCENT_POLICY policy = { on ? ACCENT_ENABLE_ACRYLICBLURBEHIND : ACCENT_DISABLED,
                         2, 0x78202020, 0 };          // AABBGGRR, 0x78 = alpha 120
WINDOWCOMPOSITIONATTRIBDATA data = { 19 /* WCA_ACCENT_POLICY */, &policy, sizeof policy };
```

**The tint lives in `GradientColor`, not in the pixels.** `0x78202020` is
`#202020` at alpha 120, applied by DWM underneath the window's own content.
This is why the fixup pass in section 2 writes alpha 0 for background pixels
rather than 120: two tint layers stacked would come out twice as dark.

A global `BlurActive` records whether the blur is in effect. It is false when
the export does not resolve, when the call returns `FALSE`, or when the user
has turned the checkbox off. `RedrawSwitcher` runs the fixup pass only when
`BlurActive` is true; otherwise every pixel keeps alpha 255 and the switcher is
opaque exactly as it is today. That single `if` is the entire fallback.

`DWMWA_WINDOW_CORNER_PREFERENCE` (cmdtab.c:846) stays. Rounded corners were
confirmed to survive per-pixel alpha, but were not tested together with an
accent policy. Verify this first during implementation; if the corners are
lost, round them in the fixup pass by forcing alpha 0 in the corner arcs.

### 4. The setting

| File | Change |
|---|---|
| `resource.h` | `#define IDC_BLUR_BACKGROUND 1009` |
| `cmdtab.rc` | One `AUTOCHECKBOX "Blur the switcher background"`, dialog height grown to fit |
| `struct ini`, cmdtab.c:665 | `bool blurBackground;` under *Appearance*, next to `darkmode` |
| `InitConfig`, cmdtab.c:754 | `.blurBackground = true`, then `GetRegKeyBool(L"blurBackground", Config.blurBackground)` |
| `SettingsDialogProcedure`, cmdtab.c:2133 | `CheckDlgButton` on open; `SetConfigBool` on OK, then `ApplySwitcherBlur(Config.blurBackground)` and `RedrawSwitcher()` so the change takes effect without a restart |
| `README.md` | Document the checkbox, and that the blur needs Windows 10 1803 or newer with *Transparency effects* on |

The registry value `blurBackground` joins the other seven under the existing
key `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab` as a `REG_DWORD`, through
the `GetRegKeyBool`/`SetConfigBool` pair already in place.

## Out of scope

Tint colour and opacity stay compiled-in constants. No new numeric registry
key: all eight settings remain booleans. Light-mode adaptation is not part of
this work — the switcher stays dark in both themes, which is what it does
today.

## Testing

Manual, on this machine:

1. Build, open the switcher over a window with strong colour, screenshot, and
   confirm the background is blurred rather than flat.
2. Confirm icons, titles and the selection rectangle look unchanged against
   today's build.
3. Untick the checkbox, press OK, reopen the switcher: opaque immediately, no
   restart.
4. Tick it again: blurred again.
5. Turn off *Transparency effects* in Windows Settings and confirm the switcher
   still renders legibly.
6. Force `BlurActive` false by hand and confirm the result is pixel-identical
   to the current build.
