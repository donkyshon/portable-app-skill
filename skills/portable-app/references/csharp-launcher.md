# Custom C# launcher

.NET Framework 4.8, no packages, a single exe next to the `App` and `Data` folders. Implements the same scheme as PortableApps.com Launcher.

## Workflow

1. Single instance (Mutex). A second start only passes arguments to the program.
2. If `Data\Session.lock` exists, the previous session crashed: `Restore()`.
3. Create `Session.lock`, `Prepare()`.
4. Start the program, wait for every process from `App`.
5. `Restore()`, delete `Session.lock`, refresh icons.

## Redirect model

```csharp
internal sealed class DirectoryRedirect
{
    public string DataName;
    public string SystemPath;

    public string PortablePath => Path.Combine(LauncherPaths.DataFolder, DataName);
    public string BackupPath => $"{SystemPath}-BackupBy{LauncherPaths.AppId}";
}

internal sealed class RegistryRedirect
{
    public string DataName;
    public string KeyPath;

    public string PortableFile => Path.Combine(LauncherPaths.DataFolder, "Registry", DataName + ".reg");
    public string BackupFile => Path.Combine(LauncherPaths.DataFolder, "Registry", DataName + ".backup.reg");
}
```

## Moving folders

```csharp
internal static class DirectoryMover
{
    public static void Prepare(DirectoryRedirect redirect)
    {
        if (Directory.Exists(redirect.SystemPath))
            Move(redirect.SystemPath, redirect.BackupPath);

        if (Directory.Exists(redirect.PortablePath))
            Move(redirect.PortablePath, redirect.SystemPath);
    }

    public static void Restore(DirectoryRedirect redirect)
    {
        if (Directory.Exists(redirect.SystemPath))
            Move(redirect.SystemPath, redirect.PortablePath);

        if (Directory.Exists(redirect.BackupPath))
            Move(redirect.BackupPath, redirect.SystemPath);

        DeleteEmptyParents(redirect.SystemPath);
    }

    private static void Move(string sourcePath, string targetPath)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(targetPath));

        if (Directory.Exists(targetPath))
            Directory.Delete(targetPath, true);

        if (IsSameVolume(sourcePath, targetPath))
        {
            Directory.Move(sourcePath, targetPath);
            return;
        }

        CopyDirectory(sourcePath, targetPath);
        Directory.Delete(sourcePath, true);
    }

    private static void CopyDirectory(string sourcePath, string targetPath)
    {
        Directory.CreateDirectory(targetPath);

        foreach (var filePath in Directory.GetFiles(sourcePath))
            File.Copy(filePath, Path.Combine(targetPath, Path.GetFileName(filePath)), true);

        foreach (var folderPath in Directory.GetDirectories(sourcePath))
            CopyDirectory(folderPath, Path.Combine(targetPath, Path.GetFileName(folderPath)));
    }

    private static bool IsSameVolume(string firstPath, string secondPath) =>
        string.Equals(Path.GetPathRoot(Path.GetFullPath(firstPath)),
            Path.GetPathRoot(Path.GetFullPath(secondPath)), StringComparison.OrdinalIgnoreCase);

    private static void DeleteEmptyParents(string folderPath)
    {
        var parentPath = Path.GetDirectoryName(folderPath);
        var localAppData = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
        var userRoots = new[] { Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), localAppData,
            Path.Combine(Path.GetDirectoryName(localAppData), "LocalLow"),
            Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData) };

        while (parentPath != null
            && !userRoots.Contains(parentPath, StringComparer.OrdinalIgnoreCase)
            && Directory.Exists(parentPath)
            && !Directory.EnumerateFileSystemEntries(parentPath).Any())
        {
            Directory.Delete(parentPath);
            parentPath = Path.GetDirectoryName(parentPath);
        }
    }
}
```

Deleting `targetPath` in `Move` is safe: in `Prepare` the target is either the backup (already renamed) or an empty spot; in `Restore` it is the stale copy in `Data`. An existing backup from a previous crash is never overwritten: `Restore` always runs before `Prepare`.

## Moving the registry

Through `reg.exe`: reliable, keeps all value types.

```csharp
internal static class RegistryMover
{
    public static void Prepare(RegistryRedirect redirect)
    {
        if (Export(redirect.KeyPath, redirect.BackupFile))
            Delete(redirect.KeyPath);

        if (File.Exists(redirect.PortableFile))
            Import(redirect.PortableFile);
    }

    public static void Restore(RegistryRedirect redirect)
    {
        if (Export(redirect.KeyPath, redirect.PortableFile))
            Delete(redirect.KeyPath);

        if (!File.Exists(redirect.BackupFile))
            return;

        Import(redirect.BackupFile);
        File.Delete(redirect.BackupFile);
    }

    private static bool Export(string keyPath, string filePath)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(filePath));
        return RunReg($"export \"{keyPath}\" \"{filePath}\" /y");
    }

    private static void Import(string filePath) => RunReg($"import \"{filePath}\"");

    private static void Delete(string keyPath) => RunReg($"delete \"{keyPath}\" /f");

    private static bool RunReg(string arguments)
    {
        var startInfo = new ProcessStartInfo("reg.exe", arguments) { CreateNoWindow = true, UseShellExecute = false };
        using (var process = Process.Start(startInfo))
        {
            process.WaitForExit();
            return process.ExitCode == 0;
        }
    }
}
```

## Waiting for all processes

```csharp
private const int PollDelayMs = 1000;

private static void WaitForAppProcesses(string appFolder)
{
    while (Process.GetProcesses().Any(process => IsInsideFolder(process, appFolder)))
        Thread.Sleep(PollDelayMs);
}

private static bool IsInsideFolder(Process process, string folderPath)
{
    var filePath = NativeMethods.GetProcessImagePath(process.Id);
    return filePath != null && filePath.StartsWith(folderPath, StringComparison.OrdinalIgnoreCase);
}
```

`process.MainModule` throws for 64-bit processes from a 32-bit launcher and for elevated processes, so the path is taken via `QueryFullProcessImageName`:

```csharp
private const int PROCESS_QUERY_LIMITED_INFORMATION = 0x1000;
private const int MaxPathLength = 32768;

[DllImport("kernel32.dll", SetLastError = true)]
private static extern IntPtr OpenProcess(int access, bool inherit, int processId);

[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
private static extern bool QueryFullProcessImageName(IntPtr process, int flags, StringBuilder name, ref int size);

[DllImport("kernel32.dll")]
private static extern bool CloseHandle(IntPtr handle);

public static string GetProcessImagePath(int processId)
{
    var handle = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, false, processId);
    if (handle == IntPtr.Zero)
        return null;

    try
    {
        var size = MaxPathLength;
        var name = new StringBuilder(size);
        return QueryFullProcessImageName(handle, 0, name, ref size) ? name.ToString() : null;
    }
    finally
    {
        CloseHandle(handle);
    }
}
```

## Junction mode for large folders

Do not copy multi-gigabyte caches on every exit: `DirectoryRedirect` gets a `UseJunction` flag, `Prepare` runs `mklink /J` via `cmd.exe /c` instead of `Move(PortablePath, SystemPath)`, `Restore` removes the link:

```csharp
private static void RemoveJunction(string folderPath)
{
    if (!Directory.Exists(folderPath))
        return;

    if ((File.GetAttributes(folderPath) & FileAttributes.ReparsePoint) == 0)
        return;

    Directory.Delete(folderPath, false);
}
```

`Directory.Delete(path, true)` on a junction is forbidden: it deletes the content on the drive.

## Before picking this rung

The launcher is rung 4 from `repack.md`. If the program accepts a profile flag or reads variables, `ProcessStartInfo.Arguments` and `EnvironmentVariables` are enough, no moving needed.

## Temporary files and environment

```csharp
var startInfo = new ProcessStartInfo(programPath) { UseShellExecute = false, WorkingDirectory = programFolder };
startInfo.EnvironmentVariables["TEMP"] = LauncherPaths.TempFolder;
startInfo.EnvironmentVariables["TMP"] = LauncherPaths.TempFolder;
```

After exit `Data\Temp` is wiped.

## Refreshing icons

```csharp
private const int SHCNE_ASSOCCHANGED = 0x08000000;
private const int SHCNF_IDLIST = 0x0000;

[DllImport("shell32.dll")]
private static extern void SHChangeNotify(int eventId, int flags, IntPtr item1, IntPtr item2);

public static void RefreshShellIcons() => SHChangeNotify(SHCNE_ASSOCCHANGED, SHCNF_IDLIST, IntPtr.Zero, IntPtr.Zero);
```

## Errors

- Catch `IOException` and `UnauthorizedAccessException` around each move separately, so one failed folder does not stop restoring the rest.
- File is locked (the program still holds it) – retry 3–5 times with a pause, then tell the user and keep `Session.lock`: the next launch finishes the restore.
- The path config (`Launcher.ini` next to the exe) is open and readable: the user sees what gets moved.
