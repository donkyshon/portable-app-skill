# Setup exe

Three ways from lightest to most controllable. All produce one exe that unpacks the portable folder.

## 1. 7-Zip SFX (free)

The stock `7zSD.sfx` from the LZMA SDK only extracts to a temp folder and runs a program. For a folder picker
use the extended module `7zsd_All.sfx` (Oleg Scherbakov's 7z SFX Constructor), which adds `InstallPath`,
`GUIMode` and dialog texts. Config: `assets/7z-sfx-config.txt`.

```
7z a -t7z -mx9 -m0=lzma2 payload.7z ".\Spotify WebPortable\*"
copy /b 7zsd_All.sfx + config.txt + payload.7z "Spotify WebPortable Setup.exe"
```

Config must be UTF-8 and sit between `;!@Install@!UTF-8!` and `;!@InstallEnd@!`. Icon and version info are
set on the `.sfx` module before concatenation (Resource Hacker), never on the finished exe – that breaks the
archive tail. Check key names against the documentation of the module version you ship.

## 2. WinRAR SFX (what DVLZ uses)

Script lives in the archive comment, template: `assets/winrar-sfx-comment.txt`.

```
rar a -sfx -z"sfx-comment.txt" -iicon"app.ico" -ep1 -r -m5 "Setup.exe" ".\Spotify WebPortable"
```

- A relative `Path=` resolves into `Program Files` and triggers a UAC prompt. Use an absolute path with
  environment variables, e.g. `Path=%USERPROFILE%\Desktop`.
- Do not add `-iadm` (request administrator).
- `Overwrite=1` plus excluding `Data` from the payload gives a safe update over an old copy.
- WinRAR is shareware: building SFX with an unregistered copy is a licensing grey zone, 7-Zip has no such issue.

## 3. Own C# setup stub (full control, no third-party tools)

A generic stub exe reads a zip appended to its own end. One build of the stub serves every app.

File layout: `[stub.exe][payload.zip][int64 payloadOffset][8 bytes "PORTSFX1"]`.

```csharp
internal static class Payload
{
    private const string Magic = "PORTSFX1";
    private const int TrailerSize = 16;

    public static ZipArchive Open()
    {
        var exePath = Process.GetCurrentProcess().MainModule.FileName;
        var bytes = File.ReadAllBytes(exePath);
        var magic = Encoding.ASCII.GetString(bytes, bytes.Length - 8, 8);
        if (magic != Magic)
            throw new InvalidDataException("Payload not found");

        var offset = BitConverter.ToInt64(bytes, bytes.Length - TrailerSize);
        var length = bytes.Length - TrailerSize - (int)offset;
        return new ZipArchive(new MemoryStream(bytes, (int)offset, length), ZipArchiveMode.Read);
    }
}

internal static class Installer
{
    private static readonly string[] PreservedRoots = { "Data/" };

    public static void Extract(ZipArchive archive, string targetFolder, IProgress<double> progress)
    {
        var root = Path.GetFullPath(targetFolder).TrimEnd('\\') + "\\";
        var isUpdate = Directory.Exists(Path.Combine(targetFolder, "Data"));
        var entries = archive.Entries
            .Where(entry => !isUpdate || !PreservedRoots.Any(root => entry.FullName.StartsWith(root)))
            .ToList();

        for (var index = 0; index < entries.Count; index++)
        {
            var entry = entries[index];
            var destination = Path.GetFullPath(Path.Combine(targetFolder, entry.FullName));
            if (!destination.StartsWith(root, StringComparison.OrdinalIgnoreCase))
                continue;

            if (entry.FullName.EndsWith("/"))
                Directory.CreateDirectory(destination);
            else
            {
                Directory.CreateDirectory(Path.GetDirectoryName(destination));
                entry.ExtractToFile(destination, true);
            }

            progress.Report((index + 1) / (double)entries.Count);
        }
    }
}
```

Build the setup with PowerShell, no compiler needed per app:

```powershell
$stub = [IO.File]::ReadAllBytes("SetupStub.exe")
$zip  = [IO.File]::ReadAllBytes("payload.zip")
$out  = [IO.File]::Create("Spotify WebPortable Setup.exe")
$out.Write($stub, 0, $stub.Length); $out.Write($zip, 0, $zip.Length)
$out.Write([BitConverter]::GetBytes([int64]$stub.Length), 0, 8)
$out.Write([Text.Encoding]::ASCII.GetBytes("PORTSFX1"), 0, 8); $out.Close()
```

Stub UI (one window): app name and version from a `setup.ini` entry inside the zip, folder box with Browse,
"Desktop shortcut" and "Run after install" checkboxes, progress bar, Install button. Before extracting check
that no process runs from the target folder (`Process.GetProcesses()` + `MainModule.FileName` in try/catch
for `Win32Exception`), otherwise ask the user to close it.

Notes:
- Reading the whole exe into memory is fine for WebPortable payloads (a few MB). For hundreds of MB use a
  bounded read-only stream over `FileStream` instead of `MemoryStream`.
- The path check against `targetFolder` blocks "zip slip" entries like `..\..\Windows`. Compare against the
  root **with a trailing separator**, otherwise `C:\Apps\Foo` also accepts `C:\Apps\Foo-evil\x`.
- Build `payload.zip` with `7z a -tzip` (forward slashes in entry names). `Compress-Archive` in Windows
  PowerShell 5.1 stores backslashes, so the `Data/` filter and the directory check in `Extract` would miss.
- Desktop shortcut without COM references: `WScript.Shell` via `Type.GetTypeFromProgID` and `dynamic`.
- Manifest `asInvoker`. Name the stub without "setup"/"install" in the file name *only* if UAC heuristics
  still prompt – Windows auto-elevates such names for exes without a manifest.
