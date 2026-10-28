# Launcher: download, update, mods, run

The launcher is the exe the user clicks every time. It owns the vendor build lifecycle and delegates the
traces-free session to the portable-app logic (`csharp-launcher.md` or PortableApps.com Launcher there).

## Settings

One ini next to the launcher, `<App>Portable.ini` in `Data\` (survives updates), seeded from
`assets/WebPortable.ini.template`. Launcher and config app read and write the same file.

State the user never edits goes to `Data\State.ini`:

```
[Vendor]
Version=1.2.53.440
PackageFile=Data\Cache\vendor-1.2.53.440.exe
LastCheck=2026-10-28

[Mods]
ThemeLoader=2.40.1
```

## First run and updates

```
installed = State.Vendor.Version
if installed is empty            -> must download, no internet = show message and exit
elif AutoUpdate and LastCheck older than UpdateIntervalHours -> query latest, download if newer
else                             -> skip network
```

- **Latest version source**, in order of reliability: vendor JSON/API endpoint → GitHub Releases API
  (`/releases/latest`, `tag_name` + `assets[].browser_download_url`) → redirect of a "latest" URL.
  Keep the URL in the ini (`UrlPrefix`, `UrlPostfix`, `UpdateUrl`) so the user can fix it without a rebuild.
- **Download** into `Data\Temp\*.part`, rename on success, compare size or SHA-256 when the source publishes it.
  Set a `User-Agent` (configurable), `SecurityProtocolType.Tls12` on .NET Framework, timeout and cancellation.
- **Unpack into `App\<Vendor>`**, never run the vendor installer normally – it writes to AppData and the registry.
  Options: the installer's own extract switch if it has one, Squirrel packages (`*-full.nupkg` is a zip,
  files under `lib\net45`), or 7-Zip for NSIS/Inno/MSI (`7z x` / `innounp` / `msiexec /a` administrative
  install into a folder). Find what works once, hardcode it per app.
- Unpack into `App\<Vendor>.new`, then swap folders – an interrupted update must not break the working copy.
- Keep the last package in `Data\Cache` so mods can be removed by re-unpacking without a download.
- Disable the vendor's own updater (delete or rename `Update.exe`, block its URL in its config if supported),
  otherwise it updates itself into AppData behind your back.

## Mods

Declared in the ini, each with a stable id:

```
[Mod.ThemeLoader]
Enabled=0
Source=https://api.github.com/repos/<owner>/<repo>/releases/latest
Asset=*.zip
Apply=cmd /c "{ModDir}\install.cmd" "{AppDir}"
Arguments=
```

Rules:
- Apply order is the order in the ini. Each mod runs with the same per-process environment as the app
  (`APPDATA`, `LOCALAPPDATA`, `USERPROFILE` into Data) so it patches the portable copy, not the system.
- After applying write `Mods.<id>=<version>` to State.ini. Re-apply when the vendor version changes,
  the mod version changes or the user toggles it on.
- Turning a mod off: re-unpack the clean vendor build from `Data\Cache`, then apply the remaining mods.
- Mods that are PowerShell scripts: run with `-NoProfile -ExecutionPolicy Bypass -File`, never pipe a remote
  script into `iex` – download, pin the version, then run the file.
- A failing mod must not block the start: log to `Data\Logs\launcher.log`, show a non-modal notice, launch clean.

## Run and cleanup

1. Single instance per copy: named mutex from a hash of the launcher path, so two copies in different folders
   can run side by side when the ini allows it.
2. Seed `Data` from `App\DefaultData` only for files that do not exist yet.
3. Portable session from portable-app: env vars, folder moves with backup, registry import, `Session.lock`.
4. Start the vendor exe with `WorkingDirectory` from the ini, wait for the whole process tree
   (Job Object or polling by image path), not just the first process – Electron clients relaunch themselves.
5. Restore the system, then if `AutoCleanCache=1` delete cache and log folders listed in the ini.

## C# skeleton

```csharp
internal static class Program
{
    [STAThread]
    private static int Main(string[] arguments)
    {
        var settings = PortableSettings.Load();
        using (var instance = SingleInstance.TryAcquire(AppPaths.LauncherPath))
        {
            if (instance == null)
                return ActivateRunningCopy();

            Session.RecoverAfterCrash();
            if (!VendorBuild.EnsureInstalled(settings))
                return 1;

            ModRunner.ApplyPending(settings);
            Session.Begin(settings);
            try
            {
                ProcessTree.RunAndWait(settings.ProgramExecutable, arguments);
            }
            finally
            {
                Session.End(settings);
                CacheCleaner.CleanIfEnabled(settings);
            }
        }

        return 0;
    }
}
```

Keep the launcher windowless (WinExe without a main window); show a small progress window only while downloading.
