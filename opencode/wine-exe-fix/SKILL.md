---
name: wine-exe-fix
description: >
  Diagnose and fix Windows .exe launch failures under Wine on Linux.
  Covers 32-bit vs 64-bit Wine prefix selection, missing .NET runtime
  (Wine Mono / MS .NET via winetricks), missing NuGet-managed
  dependencies (CommandLine, Newtonsoft.Json,
  System.Runtime.CompilerServices.Unsafe, etc.), the critical
  distinction between NuGet package version and .NET assembly version,
  target-framework variants (lib/net40 vs lib/net45) inside one .nupkg,
  FileLoadException "manifest does not match" and binding redirects in
  .exe.config, the winetricks remove_mono hang workaround, and creating
  reusable .desktop launchers that pin a custom WINEPREFIX. Use when
  the user says "exe не запускается под wine", "wine ошибка 32/64",
  "Could not load assembly", "Wine Mono is not installed",
  "FileLoadException manifest does not match", "сделай ярлык для wine
  приложения", or when an agent needs to triage a Wine launch failure
  for a .NET/WinForms application.
---

# wine-exe-fix — diagnose and fix Windows .exe launch failures under Wine

The body of this skill is shared across CLI tools and lives in
`_content/wine-exe-fix.md` at the repo root. See that file for the full
content. The sections below are the opencode-facing summary; opencode
loads this file directly.

## When this skill activates

- User tries to launch a Windows `.exe` under Wine and gets an error.
- User asks for a desktop shortcut for a Wine application.
- Agent hits `Could not load file or assembly ...` while running an
  .exe via `wine`.
- User mentions 32-bit vs 64-bit Wine prefix confusion.
- User sees `Wine Mono is not installed` or
  `FileLoadException: manifest does not match`.

## Quick triage (in order)

1. `file <exe>` → PE32 i386 = 32-bit (needs `WINEARCH=win32` prefix);
   PE32+ = 64-bit (default `~/.wine` is fine).
2. Create / use the right prefix:
   `WINEPREFIX=/home/USER/.wine32 WINEARCH=win32 wineboot --init`.
   Verify purity: `~/.wine32/drive_c/windows/syswow64` must NOT exist.
3. If `Wine Mono is not installed`:
   `WINEPREFIX=/home/USER/.wine32 winetricks --unattended dotnet40`.
4. For each `Could not load assembly 'Name, Version=X.Y.Z.W'` error:
   - fetch the matching NuGet package from
     `https://api.nuget.org/v3-flatcontainer/<pkg-lowercase>/index.json`
   - download the `.nupkg`, unzip, pick the right `lib/netXX/` variant
     (its assembly version must match `X.Y.Z.W`; verify with
     `monodis --assembly <dll> | grep Version`)
   - copy the DLL next to the `.exe`
5. If `FileLoadException: manifest does not match`: wrong DLL variant.
   Replace with the variant whose assembly version matches the app's
   reference. Only fall back to a `<bindingRedirect>` in `.exe.config`
   if no exact-match DLL is available.
6. If `winetricks dotnetNNN` hangs on `remove_mono internal` in a loop:
   kill by PID (`pgrep -af winetricks` then `kill -9 <pids>`,
   `WINEPREFIX=... wineserver -k`). Do NOT use `pkill -f winetricks`
   — it can match its own shell and hang. `dotnet40` is often
   sufficient for 4.7.2-declared apps.
7. Create a `.desktop` launcher with
   `Exec=env WINEPREFIX=/home/USER/.wine32 WINEDEBUG=-all wine "/path/to/App.exe"`
   so double-click uses the correct prefix (wine-binfmt's default
   handler always uses `~/.wine`).

## Key facts to remember

- NuGet **package version** ≠ .NET **assembly version**. The error
  cites the assembly version; verify the DLL's manifest with
  `monodis`.
- One `.nupkg` ships multiple DLLs (`lib/net35`, `lib/net40`,
  `lib/net45`, `lib/netstandard2.0`, ...) with **different assembly
  versions**. Pick the one matching the app's `<supportedRuntime>`.
- `FileLoadException: manifest does not match (0x80131040)` = you
  copied the wrong variant. Replace the DLL (preferred) over adding a
  binding redirect.
- Files inside a cryfs/FUSE vault are transparent to Wine — just open
  the vault before launching, close after.

## Full reference

See `_content/wine-exe-fix.md` for the complete workflow, a worked
example (Steam Desktop Authenticator), the final `.exe.config` with
binding redirects, and the diagnostic command cheat sheet.
