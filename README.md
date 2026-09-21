# cmdtab
Fast and lightweight Alt-Tab window switcher replacement for Windows in macOS-style. Written in the Lord's language, C.

![cmdtab-screenshot](https://github.com/stianhoiland/cmdtab/assets/2081712/ec5d0d61-005f-4123-b191-8d5b49d1f7db)

### What's the deal?
1. On Windows Alt-Tab cycles through different windows from different apps all mixed together, showing small window previews
2. On macOS Cmd-Tab cycles through different apps, showing big, clear app icons
3. On macOS there is an additional hotkey that cycles through windows of the same app

Here's a real life comparison GIF between Alt-Tab and **cmdtab** (notice the scrollbar in Alt-Tab, haha):

![comparision-gif](https://github.com/user-attachments/assets/440e2d71-6bbc-4299-acf5-cdc707371193)

So, you like the way Apple does it, but you're using Windows? **cmdtab** for Windows fixes that:

- A hotkey to cycle apps (Chrome → Spotify → File Explorer)
- A different hotkey to cycle windows (Chrome1 → Chrome2 → Chrome3)
- Big readable app icons
- Super lightweight program (~64kb)
- Lots of tiny, useful QoL features—see below
- Simple, clean, clear, commented C source code (easy to change/fix/extend by you/me/everyone!)
- So fast!
- C!
- The best macOS-style window switcher for Windows!

### Features
So why is **cmdtab** *the best* macOS-style window switcher alternative for Windows? Because it packs so many useful features into such a small package without bloat:

- Hotkey to cycle apps (Alt-Tab)
- Hotkey to cycle windows of the same app (Alt-Tilde/Backquote)
- Reverse direction by holding Shift
- Can use Arrow Keys and Enter to select
- Mouse support
- Big readable app icons
- Key presses don't unexpectedly bleed through to other apps
- Cancel and hide the switcher by pressing Escape
- Toggle app-grouping on/off (Alt-G)
- Wrap bump is hard to explain but easy to feel: Try holding Alt-Tab until the end, then press Tab again—works in reverse, too!
- Press Q to quit the selected app
- Press W to close the selected window
- Press F4 while the switcher is open to quit **cmdtab**
- Tray icon in the notification area: left click for settings, right click for a menu
- The desktop behind the switcher is blurred, and can be turned off in settings

That's a lot of useful stuff, and the code is small! Go read it, and learn some C while you're at it.

### Settings
**cmdtab** puts an icon in the notification area (behind the `^` chevron next to the clock, unless you drag it out). Left click it to open the settings window; right click it for a small menu with *Settings...* and *Quit cmdtab*.

The settings window covers switching behavior—app grouping, raising all windows of an app, whether each hotkey shows the switcher or switches straight away, and wrap bump—plus whether the desktop behind the switcher is blurred, and a checkbox for starting **cmdtab** with Windows. Settings are stored under `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab` and survive a restart. The in-switcher hotkeys `Alt-G` and `Alt-R` still work and are faster if you only want to flip one thing.

The blur needs Windows 10 version 1803 or newer with *Transparency effects* turned on in Windows Settings. Where it is unavailable the switcher simply draws its usual solid background.

Hotkeys, the blacklist and the rest of the switcher's appearance are not in the settings window yet; those still live in `InitConfig` in `cmdtab.c`.

## Installing **cmdtab**
There's no installation. Just download the [latest version](https://github.com/stianhoiland/cmdtab/releases/latest) from the Releases section, unzip, and run. 

### Run as administrator
**cmdtab** cannot see elevated applications like Task Manager unless you "Run as administrator", but also works well otherwise.

### Autorun
Tick *Start cmdtab with Windows* in the settings window to turn autorun on or off at any time, but note that the autorun that you can enable with **cmdtab** is not "Run as administrator". In the future, **cmdtab** will support autorun ***as admin***, but for now you must manually configure this by using the command below.

It makes sense to have **cmdtab** "Run as administrator", and it makes sense to have **cmdtab** autorun on login. Doing either is easy, but doing both, i.e. autorun as admin, is not so easy. The only way to autorun as admin is to use the Windows Task Scheduler. To create an appropriate scheduled task from the Command Prompt run this command:
```console
schtasks /create /sc onlogon /rl highest /tn "cmdtab elevated autorun" /tr "C:\Users\%USERNAME%\Downloads\cmdtab-v1.7-win-x86_64\cmdtab.exe --autorun"
```
You can further customize the scheduled task created by that command by running `taskschd.msc` and looking for "cmdtab elevated autorun".

### Uninstalling
**cmdtab** leaves no trace on your system, except for the settings it stores under `HKEY_CURRENT_USER\Software\stianhoiland\cmdtab`, the autorun registry key if you enabled autorun (and the scheduled task mentioned above if you manually created it). Before you delete `cmdtab.exe`, run it one last time and untick *Start cmdtab with Windows* in the settings window to remove the autorun key; the settings key can be deleted with `reg delete "HKCU\Software\stianhoiland\cmdtab" /f`.

## Buildling from source
**cmdtab** comes with a `CMakeLists.txt` for building with `CMake` and a `Makefile` for building with `make`. I use `make`.

#### MSVC/CMake
These instructions require `git`, `cmake`, and *Visual Studio* or *MSBuild*.
1. `git clone https://github.com/stianhoiland/cmdtab.git`
2. `cd cmdtab`
3. `mkdir build`
4. `cmake -G "Visual Studio 17 2022" -B build`
5. *(in developer console)* `devenv build\cmdtab.sln /build "MinSizeRel|x64"`
6. or: *(in developer console)* `msbuild build\cmdtab.sln /property:Configuration=MinSizeRel`
7. *(run cmdtab)* `build\MinSizeRel\cmdtab.exe`

#### mingw-w64 GCC/make
These instructions require `git`, `make`, `windres`, and `gcc`. I use [w64devkit](https://github.com/skeeto/w64devkit).
1. `git clone https://github.com/stianhoiland/cmdtab.git`
2. `cd cmdtab`
3. `make release`
4. *(run cmdtab)* `./cmdtab.exe`
