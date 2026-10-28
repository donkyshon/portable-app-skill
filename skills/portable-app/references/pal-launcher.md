# PortableApps.com Launcher

A ready launcher configured with an ini file, no code. Build: PortableApps.com Launcher Generator creates `<AppID>.exe` in the root. Docs: https://portableapps.com/development

## Layout

```
PhotoshopPortable\
  PhotoshopPortable.exe            launcher
  App\
    AppInfo\
      appinfo.ini                  package description
      appicon.ico, appicon_16.png, appicon_32.png
      Launcher\
        PhotoshopPortable.ini      what to move and clean
    Photoshop\                     program files, read-only
    DefaultData\                   default settings for first launch
  Data\                            all state, created by the launcher
  Other\Source\                    launcher sources, licenses
```

`App` never changes at runtime, everything mutable lives in `Data`. On first launch `DefaultData` is copied to `Data`.

## Launcher\PhotoshopPortable.ini

```ini
[Launch]
ProgramExecutable=Photoshop\Photoshop.exe
WorkingDirectory=%PAL:AppDir%\Photoshop
DirectoryMoveOK=yes
SinglePortableAppInstance=true
WaitForEXE1=Adobe Desktop Service.exe
RefreshShellIcons=both

[Activate]
Registry=true

[Environment]
TEMP=%PAL:DataDir%\Temp
TMP=%PAL:DataDir%\Temp

[DirectoriesMove]
PhotoshopRoaming=%APPDATA%\Adobe\Adobe Photoshop 2020
PhotoshopLocal=%LOCALAPPDATA%\Adobe\Adobe Photoshop 2020
AdobePDF=%APPDATA%\Adobe\Adobe PDF
LocalLow=%USERPROFILE%\AppData\LocalLow\Adobe\Photoshop

[DirectoriesCleanupIfEmpty]
1=%APPDATA%\Adobe
2=%LOCALAPPDATA%\Adobe
3=%USERPROFILE%\AppData\LocalLow\Adobe

[DirectoriesCleanupForce]
1=%LOCALAPPDATA%\Adobe\Adobe Photoshop 2020\CrashReports

[RegistryKeys]
photoshop=HKCU\Software\Adobe\Photoshop

[RegistryCleanupIfEmpty]
1=HKCU\Software\Adobe

[RegistryCleanupForce]
1=HKCU\Software\Adobe\Photoshop\Telemetry
```

## What each section does

| Section | Action |
|---|---|
| `[Launch]` | What to run, what to wait for, whether to refresh icons (`before`, `after`, `both`) |
| `[Activate] Registry=true` | Enables registry handling |
| `[Environment]` | Environment variables for the program process only |
| `[DirectoriesMove]` | Key – folder in `Data`, value – where to put it. An existing folder goes to `<path>-BackupBy<AppID>` and comes back after exit |
| `[DirectoriesCleanupIfEmpty]` | Delete if empty (vendor parents) |
| `[DirectoriesCleanupForce]` | Always delete (cache, crash reports) |
| `[RegistryKeys]` | Key – file `Data\settings\<name>.reg`, value – registry branch. An existing branch is backed up |
| `[RegistryCleanupIfEmpty]`, `[RegistryCleanupForce]` | Same for the registry |
| `[FilesMove]` | Moving individual files |
| `[FileWrite1]` | Rewrites paths in configs to the current drive letter (`%PAL:Drive%`) |

## appinfo.ini

```ini
[Format]
Type=PortableAppsFormat
Version=3.5

[Details]
Name=Photoshop Portable
AppID=PhotoshopPortable
Publisher=Adobe & Package Author
Category=Graphics & Pictures
Description=Image editor
Language=Multilingual

[License]
Shareable=false
OpenSource=false
Freeware=false

[Version]
PackageVersion=21.0.0.0
DisplayVersion=2020

[Control]
Icons=1
Start=PhotoshopPortable.exe
```

## Building the path list

1. System snapshot on a clean machine (see `verify.md`).
2. Install the program, launch, change settings, close.
3. Snapshot again and compare: new folders go to `[DirectoriesMove]`, keys to `[RegistryKeys]`, cache and logs to `CleanupForce`.
4. Program Files content goes to `App\<Name>`.
