---
title: "Getting Started with .NET: From Console Apps to Full Stack"
summary: "An overview of the .NET platform for developers—covering the core components, supported app types, and a minimal console-based To-Do app in C#."
image: "https://github.com/x7dl8p/blogs/blob/main/images/Screenshot%202025-10-02%20131326.png"
publishedAt: "2025-07-05"
---

# How I Fixed the `wt` Command for Windows Terminal: A Personal Walkthrough

I installed Windows Terminal and expected the `wt` command to work system-wide. It did not. This is the step-by-step account of what I received, how I reasoned about the problem, and how I solved it.

## What I received
These are the key outputs and messages I saw as I debugged the problem.

1. Confirmed installation location:
```text
PS C:\Users\Admin> Get-AppxPackage *WindowsTerminal* | Select InstallLocation

InstallLocation
---------------
C:\Program Files\WindowsApps\Microsoft.WindowsTerminal_1.23.12681.0_x64__8wekyb3d8bbwe
````

2. `wt` not recognized when called directly:

```text
PS C:\Users\Admin> wt -d $PWD
wt : The term 'wt' is not recognized as the name of a cmdlet, function, script file, or operable program.
...
```

3. Direct launch of the executable worked:

```powershell
& "C:\Program Files\WindowsApps\Microsoft.WindowsTerminal_1.23.12681.0_x64__8wekyb3d8bbwe\wt.exe"
# Windows Terminal opened
```

4. Attempt to add WindowsApps to PATH with `setx` produced truncation warning:

```text
WARNING: The data being saved is truncated to 1024 characters.

SUCCESS: Specified value was saved.
```

5. After the safe fix, the command worked:

```powershell
PS C:\Users\Admin> wt -d $PWD
# Windows Terminal opened in the current directory
```

## Thought process (how I moved)

* Verify the install. If the app is not installed, no point troubleshooting PATH.
* Try running `wt` normally. The error indicates PATH resolution, not a broken binary.
* Confirm the binary runs when launched directly from the install folder. If so, the binary is fine; the problem is how the system resolves the command.
* Inspect PATH for the Store execution-alias folder `C:\Users\<User>\AppData\Local\Microsoft\WindowsApps`. If missing, add it.
* Avoid `setx` because it truncates long PATH values to 1024 characters. Use the .NET API to update the user PATH without truncation and avoid duplicates.
* Restart shells and test.

## The commands I ran (exact)

#### Check install location

```powershell
Get-AppxPackage *WindowsTerminal* | Select InstallLocation
```

#### Verify direct launch

```powershell
& "C:\Program Files\WindowsApps\Microsoft.WindowsTerminal_1.23.12681.0_x64__8wekyb3d8bbwe\wt.exe"
```

#### Check current PATH entries

```powershell
$env:PATH -split ';'
```

#### (Bad attempt) setx — shows truncation risk

```powershell
setx PATH "$($env:PATH);$env:USERPROFILE\AppData\Local\Microsoft\WindowsApps"
# WARNING: truncated to 1024 characters
```

#### Correct, safe method — append WindowsApps to user PATH without truncation

```powershell
$oldPath = [Environment]::GetEnvironmentVariable("PATH", "User")
$newPath = $oldPath.Split(';') | ForEach-Object { $_ } | Where-Object { $_ -ne "$env:USERPROFILE\AppData\Local\Microsoft\WindowsApps" }
$newPath = ($newPath + "$env:USERPROFILE\AppData\Local\Microsoft\WindowsApps") -join ";"
[Environment]::SetEnvironmentVariable("PATH", $newPath, "User")
```

Then close all shells and open a new PowerShell session.

### Final test

```powershell
wt -d $PWD
```

## Result

After updating the PATH using the .NET API, `wt` resolved correctly and opened Windows Terminal in the current working directory. The installation was never the problem; the execution alias folder was simply not present in the PATH until I appended it correctly.

## Notes and tips

* Avoid `setx` for long PATH edits; it truncates to 1024 characters.
* Prefer using [Environment]::SetEnvironmentVariable for user PATH modifications when the current PATH is long.
* If `wt` still fails after PATH is correct, check Settings → Apps → App execution aliases to ensure `wt.exe` is enabled.
* If you installed from an MSIX bundle, the binary will live under `C:\Program Files\WindowsApps\…`.

--Mohammad