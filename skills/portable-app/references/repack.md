# Third-party programs

No source code means working around however the binary resolves its paths. Climb the rungs top to bottom and stop at the first one that passes `verify.md`.

## 0. Reconnaissance

1. Take a system snapshot before installing (`verify.md`).
2. Install on a clean machine or VM, launch, change settings, close.
3. Snapshot again and compare: profile folders, `HKCU`/`HKLM` keys, services, scheduled tasks, fonts, COM.
4. Program Files content goes to `App\<Name>`. Default settings go to `App\DefaultData`.
5. Process Monitor shows how the program resolves paths: a read of `User Shell Folders` right before file access means the Known-Folder API, environment variables will not help.

## 1. Native flag or marker

| Platform | Technique |
|---|---|
| Chromium, Electron | `--user-data-dir="%~dp0Data\Profile"` (the app may override it in code, verify) |
| Java | `-Duser.home=…`, `-Djava.io.tmpdir=…`, vendor-specific `-D` flags |
| Firefox and forks | `-profile "…" -no-remote` |
| VS Code | `data` folder next to `Code.exe` |
| Many programs | empty `portable.dat`, `portable.txt`, `portable.ini` next to the exe – check docs and forums |

Supported by the vendor, survives updates, needs no cleanup. Wrap it in `assets/launch.cmd.template`.

## 2. Environment variables for the process

```cmd
set "APPDATA=%~dp0Data\AppDataRoaming"
set "LOCALAPPDATA=%~dp0Data\AppDataLocal"
set "TEMP=%~dp0Data\Temp"
set "TMP=%~dp0Data\Temp"
"%~dp0App\TheApp.exe"
```

The variables live only in the process tree, nothing changes system-wide.

| Reads variables | Asks the API, rung 2 is useless |
|---|---|
| Python (`os.getenv`, `expanduser` via `USERPROFILE`), Node (`process.env`), Go (`os.UserConfigDir`), scripts | .NET `Environment.GetFolderPath`, Qt `QStandardPaths`, Java `user.home`, Electron `app.getPath`, Win32 `SHGetKnownFolderPath`, Rust `dirs` |

Override `USERPROFILE` and `HOME` only when necessary: SSH, Git and child shells break. A mixed result is common: settings land in `Data`, crash logs still go to C:. Check with Process Monitor.

Some host programs set their own variables for child processes after your launcher did, silently winning. If the app keeps reaching C: despite the launcher, it overwrites the variable instead of reading it – move to rung 3.

## 3. Directory junction

A link from the expected profile location into `Data`. Bytes physically land on the drive, nothing is copied – good for large caches. `mklink /J` needs no admin rights.

```cmd
set "TARGET=%APPDATA%\TheApp"
set "BACKUP=%TARGET%-BackupByTheAppPortable"
if exist "%TARGET%" ren "%TARGET%" "TheApp-BackupByTheAppPortable"
mklink /J "%TARGET%" "%~dp0Data\AppDataRoaming\TheApp" >nul
"%~dp0App\TheApp.exe"
rmdir "%TARGET%"
if exist "%BACKUP%" ren "%BACKUP%" "TheApp"
```

- Remove the junction only with `rmdir` without `/s`: with `/s` the content on the drive is deleted. In C# – `Directory.Delete(path, false)` after checking `FileAttributes.ReparsePoint`.
- Never silently skip an existing folder (`if not exist … mklink`): the program then writes into the real folder, and `rmdir` after exit removes it if empty.
- Some programs delete and recreate their folder – the junction is lost and data goes to C:. Verify.
- Junctions to network drives do not work; those need a symbolic link (`mklink /D`, admin rights or developer mode).
- After a crash the link stays in the profile. Protect with `Session.lock` and restore on next launch; find all dangling junctions with `dir /AL /S %APPDATA%`.

## 4. Move with backup, plus registry

The PortableApps.com Launcher scheme: on launch, folders and keys from `Data` are put into the profile, foreign data is renamed to a backup, after exit everything is carried back. Moving, not junctions. Needed where the program does not tolerate junctions or keeps settings in the registry.

- No code: `pal-launcher.md`.
- Custom launcher: `csharp-launcher.md`.
- Conflicts and parallel runs explained: `how-it-works.md`.

Do not move large caches – wipe them (`DirectoriesCleanupForce`) or link them with rung 3: copying gigabytes to a flash drive on every exit is unacceptable.

## 5. Virtualization

Enigma Virtual Box (free), Cameyo, VMware ThinApp intercept file system and registry calls and write into a virtual layer inside the package. Zero C: touches, but heavier to build, higher risk of antivirus false positives; drivers, services and copy protection that detects virtualization do not work. Only when rungs 1–4 structurally cannot cope: COM registration, `HKLM`, installers with many components.

## What cannot be done at all

Drivers, services, shell extensions, system fonts without `FR_PRIVATE`, licenses bound to `HKLM` or hardware. Say so honestly and offer a regular install with a clean uninstall.
