# Portable by design

If the code never asks Windows "where is my AppData", there is nothing to redirect or clean up. The app root is computed once at startup and every path is built from it – before loggers, databases and SDKs are initialized, because they pick a default path the moment they are created.

```
YourApp\
  YourApp.exe
  Data\
    settings.json
    Cache\      safe to delete, settings offer a clear button with the size
    Logs\
    Temp\       wiped on start and exit
```

## General rules

- Forbidden: `Environment.SpecialFolder`, `Application.UserAppDataPath`, `IsolatedStorage`, `Properties.Settings`, `os.path.expanduser`, `Path.home()`, `platformdirs`, `app.getPath()` without an override.
- Atomic writes: `.tmp` + `File.Replace`, so pulling the flash drive does not corrupt the file.
- Do not touch the registry. Autostart, file associations, context menu – only on explicit user action and with an undo button.
- Paths on the same drive are stored relative.
- No write access next to the exe (Program Files, CD, read-only flash drive) – clear error at startup.
- Package as a folder or as a single exe without self-extraction.

## C# / WPF, .NET Framework 4.8

```csharp
internal static class AppPaths
{
    public static readonly string ExeFolder = AppDomain.CurrentDomain.BaseDirectory;
    public static readonly string DataFolder = Path.Combine(ExeFolder, "Data");
    public static readonly string SettingsFile = Path.Combine(DataFolder, "settings.json");
    public static readonly string CacheFolder = Path.Combine(DataFolder, "Cache");
    public static readonly string LogsFolder = Path.Combine(DataFolder, "Logs");
    public static readonly string TempFolder = Path.Combine(DataFolder, "Temp");

    public static void EnsureCreated()
    {
        Directory.CreateDirectory(CacheFolder);
        Directory.CreateDirectory(LogsFolder);
        Directory.CreateDirectory(TempFolder);
        Environment.SetEnvironmentVariable("TEMP", TempFolder);
        Environment.SetEnvironmentVariable("TMP", TempFolder);
    }

    public static string ToPortable(string filePath)
    {
        var fullPath = Path.GetFullPath(filePath);
        return fullPath.StartsWith(ExeFolder, StringComparison.OrdinalIgnoreCase)
            ? fullPath.Substring(ExeFolder.Length)
            : fullPath;
    }

    public static string FromPortable(string storedPath) =>
        Path.IsPathRooted(storedPath) ? storedPath : Path.Combine(ExeFolder, storedPath);
}
```

`Path.GetTempPath()` reads `TMP`, so after `EnsureCreated` it points at `Data\Temp`.

| What | Writes by itself to | How to avoid |
|---|---|---|
| `Properties.Settings` | `%LOCALAPPDATA%\<Company>\<exe>_Url_<hash>` | Your own `settings.json` |
| WebView2 | `%LOCALAPPDATA%\<exe>.WebView2` | `CoreWebView2Environment.CreateAsync(null, Data\WebView2)` or `WEBVIEW2_USER_DATA_FOLDER` |
| `JumpList`, recent files | The system | Do not use |
| Third-party library logs | `%APPDATA%` | Explicit path in `Data\Logs` |

## .NET 6+ (single-file)

Root is `AppContext.BaseDirectory` (the real exe folder even in single-file). Publish with `dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true` without `IncludeNativeLibrariesForSelfExtract` and `IncludeAllContentForSelfExtract`: with them the host extracts DLLs into `%TEMP%\.net` before your code runs. If you need the flag, set `DOTNET_BUNDLE_EXTRACT_BASE_DIR` in a launcher.

## Python

```python
import os
import sys
import tempfile
from pathlib import Path

ROOT = Path(sys.executable if getattr(sys, "frozen", False) else __file__).resolve().parent
DATA = ROOT / "Data"
TEMP = DATA / "Temp"
TEMP.mkdir(parents=True, exist_ok=True)
tempfile.tempdir = str(TEMP)
os.environ["TEMP"] = os.environ["TMP"] = str(TEMP)
```

Package with `pyinstaller --onedir`. `--onefile` extracts into `%TEMP%\_MEIxxxx` on every launch; if you need one exe, use `--runtime-tmpdir .`. For small tools a portable CPython via `uv` is simpler (`UV_PYTHON_INSTALL_DIR`, `UV_CACHE_DIR` on the drive before the first call) plus a `.cmd` running `pythonw.exe`.

Library variables: `HF_HOME`, `TORCH_HOME`, `MPLCONFIGDIR`, `PIP_CACHE_DIR`, `CUDA_CACHE_PATH` (otherwise the CUDA cache lands in `%APPDATA%\NVIDIA\ComputeCache`).

## Node.js

`path.dirname(process.execPath)` when built with `pkg`/`nexe`, `__dirname` for `node app.js`: pick one and test it. Set `process.env.TEMP` and `TMP` to `Data\Temp` on the first line. Tool caches: `NPM_CONFIG_CACHE`, `PNPM_HOME`.

## Electron

`app.setPath('userData', path.join(path.dirname(process.execPath), 'Data'))` before the `ready` event – moves localStorage, cache, cookies, GPU cache. `app.setPath('temp', …)` and `'logs'` – same place. Package with electron-builder `dir` or `zip`, not `nsis` with `portable: true`: that one extracts into `%TEMP%` on every launch.

## C / C++

`GetModuleFileNameW(NULL, …)`, strip the file name, immediately `SetEnvironmentVariableW(L"TEMP", …)` and `TMP` before any `GetTempPathW`.

## AutoIt, PowerShell, cmd

| Language | Root | Temp |
|---|---|---|
| AutoIt | `@ScriptDir` | `EnvSet("TEMP", …)` |
| PowerShell | `$PSScriptRoot` | `$env:TEMP = …` |
| cmd | `%~dp0` | `set "TEMP=…"` |

Never `.`, `Get-Location` or `cd`: the working directory depends on where the program was launched from.
