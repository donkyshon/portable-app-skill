# Config app

A small separate exe (`<App>Config.exe`) so the launcher stays windowless and starts instantly.

## What it shows

Checkboxes are generated from the ini, not hardcoded:

| Control | ini key |
|---|---|
| Auto clean cache and logs after exit | `[Options] AutoCleanCache` |
| Check for updates on start | `[Options] AutoUpdate` |
| Allow two copies side by side | `[Options] AllowMultipleCopies` |
| One checkbox per mod | `[Mod.<id>] Enabled`, label from `Title`, tooltip from `Source` |
| Proxy (text box) | `[Network] Proxy` |

Buttons: **Clean cache and logs**, **Check for updates**, **Open Data folder**, **OK**. A link to the
project page from `AppInfo.ini`.

## Rules

- Same `PortableSettings` class as the launcher (shared source file), so both never disagree on keys or defaults.
- Refuse to change mods and run cleanup while the app is running: check the launcher mutex, show why buttons
  are disabled.
- "Check for updates" calls the launcher with a switch (`SpotifyPortable.exe --update-only`) instead of
  duplicating download code.
- Write the ini atomically (`.tmp` + `File.Replace`), preserve unknown keys and comments – users edit it by hand.
- Mod toggles take effect on the next launch; say so in the window instead of patching from the config app.
- UI strings from `App\Languages\<lang>.ini`, language picked by the system UI culture with English fallback.
- First run without `Data\`: create it from `App\DefaultData`, then open the window – the setup may have
  "Run config after install" as an option.
