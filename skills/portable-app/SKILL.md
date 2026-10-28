---
name: portable-app
description: Makes Windows programs truly portable – both your own apps built from scratch and third-party executables you cannot modify. The program runs from a USB stick or any drive, keeps all state in a Data folder next to itself and leaves no traces in AppData, LocalLow, ProgramData, Temp or the registry after it exits. Use when the user asks for a portable version, a portable build, a launcher for running from a flash drive, the PortableApps.com format, picks packaging (PyInstaller, dotnet publish, electron-builder) for a portable build, complains that a program "litters C:", or demands "don't touch the system drive" – even if the word portable is never said.
license: MIT-0
metadata:
  author: Donkyshon
  version: "1.1.0"
---

# Portable Apps

A portable program starts from any folder, survives a drive letter or computer change, and after exit leaves the system exactly as it was before launch. All state lives in `Data` next to the exe.

## First question: do you have the source?

- **Yes, it is your own program** → portable by design: zero calls to user profile APIs. No launcher, no moving, no cleanup. See `references/born-portable.md`.
- **No, it is someone else's exe** → climb the ladder below from lightest to heaviest. Stop at the first rung that passes verification. See `references/repack.md`.

## The ladder for third-party programs

| Rung | Technique | Touches C: | After a crash |
|---|---|---|---|
| 1 | Native flag or marker: `--user-data-dir`, `-Duser.home`, `portable.txt` | no | nothing |
| 2 | Environment variables for the process only: `APPDATA`, `LOCALAPPDATA`, `TEMP` | no | nothing |
| 3 | Directory junction from the profile into `Data` while running | link in profile | dangling link |
| 4 | Move "unpack – run – pack back" with backup, plus registry | data in profile | data in profile |
| 5 | Virtualization: Enigma Virtual Box, Cameyo, ThinApp | no | nothing |

Rungs can be mixed: a flag for the main profile, junctions for folders the app resolves through the API, moving for the registry.

Rung 2 works only if the program reads the variables (Python, Node, Go, scripts). .NET, Qt, Java, Electron and native Win32 call `SHGetKnownFolderPath` and ignore the variables.

Rungs 3–4 are explained with Adobe builds as the example in `references/how-it-works.md`. A ready no-code mechanism is in `references/pal-launcher.md`, a custom C# launcher in `references/csharp-launcher.md`, a `.cmd` template in `assets/launch.cmd.template`.

## Rules

- Paths only relative to the exe folder. `Data`: `settings.json`, `Cache`, `Logs`, `Temp`. A tiny utility is fine with a single ini next to the exe.
- Always point `TEMP`/`TMP` at `Data\Temp`: third-party libraries and packers write there first.
- Self-extracting packaging (`--onefile`, NSIS portable, `IncludeNativeLibrariesForSelfExtract`) unpacks into `%TEMP%` before your first line of code runs. Package as a folder or set `TEMP` in the launcher.
- Move and link the narrowest paths: `%APPDATA%\Adobe\Adobe Photoshop 2020`, not all of `%APPDATA%\Adobe`.
- Never delete what existed before launch: foreign data is renamed to `<path>-BackupBy<AppID>` and restored after exit. Delete the vendor parent folder only when empty.
- Registry: export `HKCU\Software\<Vendor>\<App>` to `Data\Registry`, delete it, restore the backup after exit. For `HKLM`, services and drivers say honestly that it cannot be done without virtualization.
- The launcher waits for every process from the program folder, not just the main one: child processes, updaters, crash reporters. `start ""` without `/wait` is forbidden for rungs 3–4: cleanup will never run.
- Session marker `Data\Session.lock`: if it exists on start, the previous session crashed – restore first.
- A second launcher start while the program is running moves nothing a second time.
- A flash drive is a different volume: moving folders means copy plus delete; junctions are unaffected.
- Store paths to files on the same drive as relative: the drive letter changes.
- No write access next to the exe – show a clear error, never silently fall back to `%APPDATA%`.
- If folders of two portable programs overlap, the backup is restored correctly only when they close in reverse order. Warn the user or separate the paths.
- `RefreshShellIcons` after exit is a visible signal that the launcher finished cleanup.
- Keep launcher configs open and readable: the user sees what gets moved.

## Verification is mandatory

Do not trust framework documentation: take a system snapshot before and after, run Process Monitor, test crash and conflict scenarios – see `references/verify.md`. Call it portable only after the diff is empty.

## Shipping it

To pack the finished portable folder into one setup exe, add a first-run downloader, auto-updates, optional mods and a config app, use the companion skill **portable-installer**.
