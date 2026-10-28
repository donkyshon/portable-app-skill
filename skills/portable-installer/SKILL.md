---
name: portable-installer
description: Packages a portable Windows program into a single setup exe that unpacks it into a folder of the user's choice – no admin rights, no Program Files, no registry entries – and optionally builds the "WebPortable" trio around it – a launcher exe that downloads the latest vendor build on first run, auto-updates it and applies optional post-install mods, plus a config exe with checkboxes for those options. Use when the user wants a setup.exe, an installer or a self-extracting archive for a portable app, a repack like "Spotify/Discord WebPortable", a first-run downloader, a config tool for a portable build, or asks how such repacks work. Companion of the portable-app skill – that one makes the program leave no traces, this one ships and maintains it.
license: MIT-0
metadata:
  author: Donkyshon
  version: "1.1.0"
---

# Portable Installer

Ships a portable program as one exe and keeps it alive afterwards. The traces-free runtime itself (Data folder,
moving AppData, registry backup, crash recovery) belongs to the **portable-app** skill – read it first if the
package is not portable yet.

## Anatomy of a WebPortable-style repack

```
Spotify WebPortable v2.6.exe        setup: self-extracting archive, asks for a folder
└─ Spotify WebPortable\
   ├─ SpotifyPortable.exe           launcher: download/update, mods, portable session, run
   ├─ SpotifyConfig.exe             config: checkboxes and buttons over one ini file
   ├─ ReadMe.txt
   ├─ App\
   │  ├─ AppInfo\AppInfo.ini        name, publisher, version
   │  ├─ AppInfo\Launcher\*.ini     portable rules: env vars, moves, registry
   │  ├─ DefaultData\               seed copied into Data on first run
   │  ├─ Languages\En.ini, Ru.ini   UI strings
   │  └─ Spotify\                   vendor binaries, empty in the setup, filled by the launcher
   └─ Data\                         user state, never shipped, never overwritten by an update
```

The setup is tiny because vendor binaries are **not** inside: the launcher downloads the official build on
first run. This keeps the package small, always fresh and avoids redistributing someone else's binaries.

## Three deliverables, pick what the task needs

| Need | Deliverable | Reference |
|---|---|---|
| One exe that unpacks a ready folder | setup (SFX or own C# stub) | `references/setup-exe.md` |
| Fetch vendor build, update, apply mods | launcher | `references/launcher.md` |
| Let the user toggle options without editing ini | config app | `references/config-app.md` |

A plain portable tool needs only the setup. A repack of a frequently updated free client (music, chat, browser)
needs all three.

## Setup rules

1. **No admin.** Never request elevation, never default to `Program Files`. Default path:
   `%USERPROFILE%\Desktop\<App> Portable` or the folder the setup was started from.
2. **Folder picker** with the default path prefilled, plus "run after install" and "desktop shortcut" options.
3. **Update over an existing copy** replaces `App\` and the exes, **never touches `Data\`** and keeps the user's ini.
4. Close a running copy first or refuse to install over it.
5. No registry, no Uninstall entry, no Start menu – deleting the folder is the uninstall.
6. Icon, version info and product name on every exe. Sign them if possible: unsigned SFX stubs trigger
   SmartScreen and some antiviruses.

## Launcher pipeline

```
read settings ini → lock session → recover after crash (portable-app)
→ vendor build missing or update due? download to Data\Temp, unpack into App\<Vendor>
→ for each enabled mod with a stale marker: download, apply, write marker
→ portable session: seed Data from DefaultData, env vars, move folders, registry (portable-app)
→ run vendor exe, wait for the whole process tree
→ restore system, optional cache and log cleanup, unlock
```

Every network step is optional at runtime: no internet → run the installed build, never block the start.

## Mods and patches

Third-party patchers (themes, plugin loaders, proxies, client tweaks) are declared in the ini, off by default,
applied by the launcher **after** the vendor build is unpacked and **re-applied after each update**, because an
update replaces patched files. Pin versions or verify checksums of what you download, show the user the source
URL in the config app and respect the licenses and terms of the tools and services involved. Details in
`references/launcher.md`.

## Files in this skill

- `references/setup-exe.md` – 7-Zip SFX, WinRAR SFX, own C# setup stub, update-over logic.
- `references/launcher.md` – settings ini, first-run download, updates, mods, cleanup, C# skeleton.
- `references/config-app.md` – config exe: options, buttons, sharing code with the launcher.
- `assets/WebPortable.ini.template` – settings file shared by launcher and config.
- `assets/7z-sfx-config.txt`, `assets/winrar-sfx-comment.txt`, `assets/build-setup.cmd.template`.

## Checklist

- Setup runs as a normal user, default path is writable, picker works, shortcut optional.
- Updating over an old copy keeps `Data\` and user options.
- First run offline shows a clear message, later runs offline start the installed build.
- Disabling a mod and restarting restores clean vendor files (re-unpack from the cached package).
- After exit the system snapshot matches the one before launch (`verify.md` of portable-app).
- Two copies side by side work if `SinglePortableAppInstance` allows it.
