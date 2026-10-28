# How portable programs work

Walkthrough on well-made Adobe Photoshop and Illustrator portable builds.

## 1. Clean system

Before launch there is no `Adobe` folder in `%APPDATA%`, `%LOCALAPPDATA%` or `%USERPROFILE%\AppData\LocalLow`. The program has never been on this system.

## 2. Launch

The launcher creates all three `Adobe` folders and puts the settings from the build's `Data` there. The program runs as if installed.

## 3. Exit

All three folders disappear. In fact they are moved back into `Data` next to the program (on the flash drive or anywhere). The next launch on any computer has the same settings.

## 4. Conflict with existing files

The system already has `%APPDATA%\Adobe\file.dat` (from another program or the user).

- Launch: Photoshop puts its folders next to it, `file.dat` is left alone.
- Exit: Photoshop folders go to `Data`, `file.dat` stays. The `Adobe` folder is not deleted because it is not empty.

If the name matches fully (a folder with the exact path the program needs already exists), the launcher renames it to a backup and restores it after exit.

## 5. Two programs from the same vendor

Photoshop and Illustrator both use `%APPDATA%\Adobe\Adobe PDF`.

1. Illustrator starts: its `Adobe PDF` is put in place.
2. Photoshop starts: sees the existing `Adobe PDF`, renames it to `Adobe PDF-BackupByPhotoshopPortable` and puts its own. Settings do not mix.
3. Photoshop exits: its `Adobe PDF` goes to `Data`, Illustrator's backup returns.
4. Illustrator exits: takes its folder, `Adobe` is empty (except foreign files) and gets deleted.

Weak spot: the backup is restored correctly only if programs close in reverse order (a stack). If Illustrator closes first, it takes Photoshop's settings. Therefore:

- design builds so their folders never overlap;
- if overlap is unavoidable, run no more than 2–4 such programs at once and close them in reverse order;
- the launcher can check that `SystemPath` currently belongs to it (a marker file inside the folder) and otherwise refuse to move with a clear message.

## 6. Full-exit signal

`RefreshShellIcons=both` in the `[Launch]` section of the launcher ini refreshes the icon cache on launch and on exit. Icons blinking after exit mean the launcher finished moving and the next program can be started.

## 7. Where portable programs still litter

| Place | Cause | Fix |
|---|---|---|
| `%TEMP%` | Temporary files | Redirect `TEMP`/`TMP` to `Data\Temp` via environment variables, wipe after exit |
| `HKCU\Software\<Vendor>` | Settings in the registry | Export to `Data\settings.reg`, delete the key, restore the backup |
| `HKCU\Software\Classes` | File associations | Do not register, or register with undo |
| `HKLM`, services, drivers | Need admin rights | Full portability impossible, warn |
| `ProgramData` | Licenses, shared data | Move like a normal folder if not locked |
| Prefetch, Jump Lists, Windows MRU | Written by the system | Not the program's litter |
| Fonts, COM objects | Global registration | `AddFontResourceEx` with `FR_PRIVATE`, registration-free COM via manifest |

## 8. Portability levels (full ladder in `repack.md`)

1. By design: the program keeps everything next to the exe. Best option.
2. Launcher: moving back and forth. Works for most programs, requires a clean exit.
3. Virtualization: the program sees a virtual registry and file system. Nothing is written to the system at all, but higher risk of incompatibility and antivirus alerts.
