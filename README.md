# portable-app-skill

Two companion [Agent Skills](https://agentskills.io): **portable-app** makes a program leave no traces,
**portable-installer** ships it as one setup exe and keeps it updated.

## portable-app

An [Agent Skill](https://agentskills.io) that teaches AI coding agents to make **Windows programs truly portable** –
both your own apps written from scratch and third-party executables you cannot modify.

A portable program runs from a USB stick or any drive, survives a drive letter change, keeps all state in `Data`
next to the exe and after exit leaves **no traces** in `AppData`, `LocalLow`, `ProgramData`, `%TEMP%` or the registry.

## What it covers

- **Your own code** – portable by design: app-root path resolution and temp redirection for C#/.NET, Python,
  Node.js, Electron, C/C++, AutoIt, PowerShell and cmd, plus packaging traps (`--onefile`, NSIS portable,
  `IncludeNativeLibrariesForSelfExtract`) that silently unpack into `%TEMP%`.
- **Someone else's exe** – a five-rung ladder from lightest to heaviest:
  native flag → per-process environment variables → directory junctions → move-with-backup plus registry →
  virtualization.
- **PortableApps.com Launcher** – ready `Launcher.ini` and `appinfo.ini` templates.
- **Custom C# launcher** – folder and registry moving, waiting for all child processes, crash recovery via
  `Session.lock`, junction mode for large caches.
- **Conflicts** – backups of pre-existing folders, two portable apps from the same vendor running at once.
- **Verification** – system snapshot script, Process Monitor filters and a scenario checklist.

## portable-installer

- **Setup exe** – 7-Zip SFX, WinRAR SFX or an own C# stub with an appended zip: folder picker, no admin,
  no Program Files, update over an old copy without touching `Data`.
- **WebPortable launcher** – downloads the official vendor build on first run, auto-updates it, unpacks
  without running the vendor installer, applies optional mods and re-applies them after updates.
- **Config app** – checkboxes generated from a shared ini, cache cleanup, update check.
- Templates: shared settings ini, 7-Zip SFX config, WinRAR SFX comment, one-click build script.

## Install

**skills.sh (Claude Code, Cursor, Codex, Copilot and others)**

```
npx skills add Donkyshon/portable-app-skill
```

**Claude Code plugin**

```
/plugin marketplace add Donkyshon/portable-app-skill
/plugin install portable-app@donkyshon-skills
```

**ClawHub (OpenClaw)**

```
clawhub install portable-app
clawhub install portable-installer
```

**Manual** – copy `skills/portable-app` and `skills/portable-installer` into `~/.claude/skills/` (personal) or `.claude/skills/` (project).
On claude.ai upload `portable-app.skill` and `portable-installer.skill` from Releases in *Settings → Capabilities → Skills*.

## Usage

Just ask, the skill triggers by itself:

- "Make this Python tool portable so it runs from a flash drive"
- "Repack this app so it doesn't litter C:"
- "Write a PortableApps.com launcher for Photoshop"
- "Pack this portable folder into a setup exe with a folder picker"
- "Make a WebPortable launcher that downloads the latest build and a config tool for it"

## Structure

```
skills/portable-app/
  SKILL.md                     entry point: decision tree, ladder, rules
  references/
    born-portable.md           your own apps, per-stack recipes
    repack.md                  third-party apps, rungs 0-5
    how-it-works.md            walkthrough on Adobe builds
    pal-launcher.md            PortableApps.com Launcher config
    csharp-launcher.md         custom launcher in C#
    verify.md                  snapshot, Process Monitor, checklist
  assets/
    launch.cmd.template        cmd launcher with crash recovery
skills/portable-installer/
  SKILL.md                     entry point: anatomy, pipeline, rules
  references/
    setup-exe.md               7-Zip SFX, WinRAR SFX, C# setup stub
    launcher.md                download, update, mods, run, cleanup
    config-app.md              config exe over the shared ini
  assets/
    WebPortable.ini.template   shared settings
    7z-sfx-config.txt          7-Zip SFX config
    winrar-sfx-comment.txt     WinRAR SFX script
    build-setup.cmd.template   one-click setup build
```

> The templates and C# snippets are reference code. Test them on a VM before using on real data.

## License

[MIT-0](LICENSE) – do anything you want, no attribution required.
