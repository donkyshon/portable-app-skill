# Verifying that a program leaves no traces

Framework and dependency docs change between versions. Only testing this build on this machine tells the truth.

## System snapshot (fast, catches deletions and registry)

`Snapshot.ps1` – run before launch, while running and after exit:

```powershell
[CmdletBinding()]
param([Parameter(Mandatory)][string]$Name)
Set-StrictMode -Version Latest
$ErrorActionPreference = 'SilentlyContinue'

$folders = @($env:APPDATA, $env:LOCALAPPDATA, "$env:USERPROFILE\AppData\LocalLow", $env:TEMP, $env:ProgramData)
$folders | ForEach-Object { Get-ChildItem $_ -Recurse -Force } |
    ForEach-Object { "$($_.FullName)`t$($_.LastWriteTimeUtc.Ticks)" } |
    Sort-Object | Set-Content "$Name.files.txt"

reg export HKCU "$Name.hkcu.reg" /y | Out-Null
```

```powershell
Compare-Object (Get-Content before.files.txt) (Get-Content after.files.txt)
fc.exe /u before.hkcu.reg after.hkcu.reg
```

Modification times in the list catch edits to existing files, comparing the lists catches new and deleted ones.

## Process Monitor (precise, shows who and how)

1. `procmon64.exe` from Sysinternals, no install. Press `Ctrl+E` right away to pause.
2. `Ctrl+L`, filters: `Process Name is TheApp.exe` Include; with a launcher add it and its children too (or `Parent PID`).
3. Second filter: `Path begins with C:\Users` Include, a separate run for `C:\ProgramData` and `HKCU`. Rules on different columns combine with AND.
4. `Ctrl+E`, launch, work for a minute (menus, saving, settings), close, `Ctrl+E`.

How to read it:

- `NAME NOT FOUND`, `PATH NOT FOUND` – the program probed an old path and found nothing. Not a leak.
- `CreateFile` with `SUCCESS` and write access, `WriteFile`, `SetRenameInformationFile` under `C:\Users` – a leak; the path shows what to redirect.
- `RegSetValue`, `RegCreateKey` with `SUCCESS` – registry write.
- A read of `HKCU\…\Explorer\User Shell Folders` before file access – Known-Folder API: variables (rung 2) will not help, use a junction or moving.
- `Tools → Process Tree` shows every child process the launcher must wait for.

## Scenarios

- [ ] Clean system: the "before" vs "after exit" diff is empty except Prefetch, Jump Lists and Windows logs.
- [ ] Settings persist: change, close, reopen – still there.
- [ ] Another computer or drive letter: settings and recent files work.
- [ ] Conflict: a folder with the same path and a file inside created beforehand. After exit the folder and file are intact.
- [ ] A foreign file in the vendor folder stays, the vendor folder is not deleted.
- [ ] Two programs from the same vendor at once: settings do not mix, closing in any order loses nothing or shows a warning.
- [ ] Crash: program and launcher killed via Task Manager. Next launch restores the system and `Data`, no dangling junctions.
- [ ] Second launcher start while the program is open: nothing moved twice.
- [ ] Read-only flash drive or Program Files: clear message, no silent writes to the system.
- [ ] Standard user without admin rights: everything works.
- [ ] Self-extraction: no `_MEI*`, `.net` or unpacked Electron in `%TEMP%` while running.
- [ ] Icon refresh after exit fires, if enabled.
