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

## Goal

When a Windows `.exe` (especially a 32-bit .NET / WinForms application)
refuses to run under Wine, identify the actual cause and repair it
without reinstalling Windows or switching to a VM. Covers:

- 32-bit vs 64-bit Wine prefix selection
- Missing .NET runtime (Wine Mono / MS .NET via winetricks)
- Missing NuGet-managed dependencies (`CommandLine`,
  `Newtonsoft.Json`, `System.Runtime.CompilerServices.Unsafe`, etc.)
- Assembly version vs NuGet package version mismatch
- `FileLoadException: manifest does not match` — binding redirects in
  `.exe.config`
- Creating reusable `.desktop` launchers with a fixed `WINEPREFIX`

Use when the user says: «exe не запускается под wine», «wine ошибка
32/64», «Could not load assembly», «Wine Mono is not installed»,
«FileLoadException manifest does not match», «сделай ярлык для wine
приложения», or when an agent needs to triage a Wine launch failure.

## Threat model / scope

| Situation | Covered |
|-----------|---------|
| 32-bit .exe in 64-bit prefix | Yes — separate `WINEARCH=win32` prefix |
| Missing .NET runtime | Yes — `winetricks dotnet40` (or 45/472/48 if needed) |
| Missing NuGet DLL next to `.exe` | Yes — fetch from nuget.org, drop in app folder |
| Wrong assembly version of a DLL | Yes — pick the right `lib/netXX/` variant |
| binding redirect needed | Yes — edit `.exe.config` `<assemblyBinding>` |
| Anti-cheat / kernel-level DRM | **No** — needs Windows or a VM with GPU passthrough |
| Native MSVC runtime crash | Partially — try `winetricks vcrun2019`; out of scope if it persists |
| GPU/driver issues | Out of scope — fix the host driver (NVIDIA/AMD/Mesa) first |

## Prerequisites (host)

- Linux with Wine installed (`wine`, `wine32:i386`, `wine64`,
  `winetricks`, `wine-binfmt`).
- i386 architecture enabled: `dpkg --add-architecture i386` (Debian/Ubuntu).
- `/dev/fuse` not required; `fuseiso` may be pulled by `wine-binfmt`.
- Optional: `mono-devel` for `monodis` (reads .NET assembly manifests).
- Optional: `unzip` to extract `.nupkg` (it's a ZIP).

Verify:
```bash
wine --version              # wine-10.0 or similar
which wine wine64 winetricks
dpkg --print-foreign-architectures   # should list i386
file /path/to/app.exe       # PE32 (i386) = 32-bit; PE32+ = 64-bit
```

## Triage workflow

Run the steps in order. Stop at the first one that resolves the launch.

### Step 1 — Identify the binary's architecture

```bash
file "/path/to/App.exe"
```

Output variants:
- `PE32 executable ... Intel i386` → **32-bit**. Needs a `win32` prefix.
- `PE32+ executable ... x86-64` → **64-bit**. Default `~/.wine` is fine.
- `PE32 ... Mono/.Net assembly` → .NET app; also needs a .NET runtime
  (Step 3).

### Step 2 — Use the correct Wine prefix

If the binary is 32-bit (especially 32-bit .NET), do **not** reuse the
default `~/.wine` (which is 64-bit with wow64). Create a dedicated
32-bit prefix:

```bash
WINEPREFIX=/home/USER/.wine32 WINEARCH=win32 WINEDEBUG=-all wineboot --init
```

Verify the prefix is truly 32-bit:
```bash
ls /home/USER/.wine32/drive_c/windows/syswow64 2>/dev/null \
  && echo "64-bit/wow64 — WRONG" \
  || echo "чистый win32 — OK"
```
A pure `win32` prefix has `system32/` (yes, 32-bit DLLs live in a dir
called system32) and **no** `syswow64/`.

Run the app with the explicit prefix:
```bash
WINEPREFIX=/home/USER/.wine32 WINEDEBUG=-all wine "/path/to/App.exe"
```

### Step 3 — Provide a .NET runtime

If the error is:
```
err:mscoree:CLRRuntimeInfo_GetRuntimeHost Wine Mono is not installed
```
or
```
Unhandled Exception: System.IO.FileNotFoundException: Could not load
file or assembly '...' ... (this is a .NET app)
```

Install MS .NET via winetricks into the chosen prefix:
```bash
WINEPREFIX=/home/USER/.wine32 winetricks --unattended dotnet40
```

Choosing the .NET version:
- `dotnet40` — works for most 4.x apps (4.0, 4.5, 4.7.2) because the
  4.0 runtime is backward-compatible with later 4.x reference versions,
  **provided** the app doesn't use 4.7.2-only APIs. Try this first.
- `dotnet45`, `dotnet452`, `dotnet46`, `dotnet461`, `dotnet462`,
  `dotnet471`, `dotnet472`, `dotnet48` — if `dotnet40` doesn't cut it.
  Install in increasing order; do not skip.

Known gotcha: `winetricks dotnet472` (and some others) can **hang in a
`remove_mono internal` loop** in certain prefix states. If it loops
more than 3-4 times, kill it (by PID, not `pkill -f` — `pkill -f
winetricks` can hang matching its own bash process):
```bash
pgrep -af winetricks           # note the PIDs
kill -9 <pid1> <pid2> ...
WINEPREFIX=/home/USER/.wine32 wineserver -k
```
Then either try a different dotnet version or proceed with `dotnet40`
and see if the app actually needs the newer one.

### Step 4 — Resolve missing assembly errors

After .NET is in, the app may still throw:
```
System.IO.FileNotFoundException: Could not load file or assembly
  'CommandLine, Version=1.9.71.2, Culture=neutral,
  PublicKeyToken=de6f01bd326f8c32' or one of its dependencies.
```

This is **not** a Wine bug. The app's build was shipped without its
NuGet dependencies next to the `.exe`. Repair by fetching the DLL
from nuget.org.

#### 4a. Find the package

The assembly name in the error is usually the NuGet package name. If
not, search: `https://www.nuget.org/packages/<Name>`.

List available versions:
```bash
curl -sL "https://api.nuget.org/v3-flatcontainer/<package-lowercase>/index.json"
```
Package names are lowercase in the flat container API:
`CommandLineParser` → `commandlineparser`.

#### 4b. NuGet package version ≠ assembly version

The error cites the **assembly version** (compiled into the DLL's
manifest). The NuGet package has its own **package version**, which is
usually different. Example:
- Error asks for `CommandLine, Version=1.9.71.2`
- NuGet has package versions `1.9.71`, `1.9.2.41`, etc. — **no
  `1.9.71.2` package exists** (HTTP 404 if you try).
- Package `1.9.71` contains a DLL whose assembly version is
  `1.9.71.2` (matches the error).

Rule: pick the package version whose **DLL's assembly version** matches
what the error asked for. Verify after download:
```bash
strings lib/net40/<Name>.dll | grep -E "^[0-9]+\.[0-9]+\.[0-9]+" | head
# or, more reliable:
monodis --assembly lib/net40/<Name>.dll | grep -E "Name:|Version:"
```

#### 4c. One package, multiple DLLs (target framework variants)

A single `.nupkg` ships separate DLLs for different target frameworks:
```
lib/net35/<Name>.dll      (assembly version X)
lib/net40/<Name>.dll      (assembly version Y)  <-- often what you want
lib/net45/<Name>.dll      (assembly version Z)
lib/netstandard2.0/<Name>.dll
```
Each variant can have a **different assembly version**. Real example
— `Newtonsoft.Json 13.0.3`:
- `lib/net40/Newtonsoft.Json.dll` → assembly version `13.0.0.0`
- `lib/net45/Newtonsoft.Json.dll` → assembly version `13.0.3.27908`

If the error asks for `13.0.0.0`, you must copy the **net40** variant.
Copying net45 will give:
```
System.IO.FileLoadException: ... manifest definition does not match
  (HRESULT: 0x80131040)
```

Match the target framework to the app's `.exe.config` `<supportedRuntime>`
(`v4.0` → net40-family; `v2.0` → net35/net20).

#### 4d. Download and extract

```bash
cd /tmp
curl -sL -o pkg.nupkg \
  "https://api.nuget.org/v3-flatcontainer/<package-lowercase>/<version>/<package-lowercase>.<version>.nupkg"
mkdir -p pkg_extract && cd pkg_extract
unzip -o ../pkg.nupkg
cp lib/net40/<Name>.dll /path/to/app-folder/
```

#### 4e. Verify the assembly identity

```bash
monodis --assembly /path/to/app-folder/<Name>.dll | head -10
# expected:
# Name:          Newtonsoft.Json
# Version:       13.0.0.0
```
If `monodis` is not installed, `strings <dll> | grep -E "^[0-9]+\."` is
a weaker but usually sufficient fallback.

Repeat Step 4 for each missing assembly the error names (one at a
time — the next one surfaces only after the previous is fixed).

### Step 5 — Fix `FileLoadException: manifest does not match`

If the DLL is found but its assembly version doesn't exactly match the
app's reference, you have two options:

**Option A (preferred):** replace the DLL with one whose assembly
version matches the reference exactly. Use a different
`lib/netXX/` variant or a different package version.

**Option B:** add a binding redirect in `<app>.exe.config` so the CLR
accepts a newer assembly version in place of the requested one:
```xml
<runtime>
  <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
    <dependentAssembly>
      <assemblyIdentity name="Newtonsoft.Json"
        publicKeyToken="30ad4fe6b2a6aeed" culture="neutral" />
      <bindingRedirect oldVersion="0.0.0.0-13.0.0.0"
        newVersion="13.0.0.0" />
    </dependentAssembly>
  </assemblyBinding>
</runtime>
```
The `publicKeyToken` must match the one in the error message. The
`newVersion` must match the **actual** assembly version of the DLL you
put next to the `.exe` (check with `monodis`).

Caveat: Option B only works if the DLL is **binary-compatible** with
the reference (same public API). For patch-version differences it
usually is; for major-version jumps it often isn't.

### Step 6 — Create a `.desktop` launcher (so the user can double-click)

Wine's `wine-binfmt` registers `.exe` to run with `wine` — but it uses
the **default** `~/.wine` prefix, which is wrong for any app that needs
a custom prefix or extra setup. Make a dedicated launcher:

```ini
[Desktop Entry]
Type=Application
Name=App Name
Comment=Short description (Wine, win32 prefix)
Exec=env WINEPREFIX=/home/USER/.wine32 WINEDEBUG=-all wine "/path/to/App.exe"
Icon=wine
Terminal=false
Categories=Game;
```

Install:
```bash
chmod +x /home/USER/Desktop/app-name.desktop
# also copy to applications menu if desired:
cp /home/USER/Desktop/app-name.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/
```

The `env WINEPREFIX=...` in `Exec=` is the key part — it ensures the
launcher always uses the right prefix regardless of how it's invoked.

## Reference: real-world session (Steam Desktop Authenticator)

Worked example, captured during a live session. The host was Debian 13
+ KDE Plasma 6.3.6 (Wayland) + Wine 10.0. The app lives inside a
cryfs-encrypted vault mounted at `/media/den/D/data/MYBD/SDA1/` —
files are accessible to Wine transparently; writes are encrypted
on-the-fly by cryfs.

Binary:
```
Steam Desktop Authenticator.exe: PE32 (GUI), Intel i386,
  Mono/.Net assembly
```

Errors encountered, in order, and fixes:

| # | Error | Cause | Fix |
|---|-------|-------|-----|
| 1 | «32 vs 64 бит prefix» | default `~/.wine` is 64-bit/wow64 | `WINEPREFIX=/home/den/.wine32 WINEARCH=win32 wineboot --init` |
| 2 | `Wine Mono is not installed` | no .NET runtime in prefix | `winetricks --unattended dotnet40` |
| 3 | `Could not load 'CommandLine, Version=1.9.71.2'` | DLL missing from app folder | NuGet `commandlineparser/1.9.71` → `lib/net40/CommandLine.dll` (assembly version `1.9.71.2`) |
| 4 | `Could not load 'Newtonsoft.Json, Version=13.0.0.0'` | DLL missing | NuGet `newtonsoft.json/13.0.3` → **`lib/net40/`** (assembly version `13.0.0.0`); net45 variant has `13.0.3.27908` and fails |
| 5 | `FileLoadException: manifest does not match (0x80131040)` | copied net45 DLL by mistake | replaced with net40 DLL (Option A); no binding redirect needed |
| 6 | `Could not load 'System.Runtime.CompilerServices.Unsafe, Version=6.0.0.0'` | DLL missing | NuGet `system.runtime.compilerservices.unsafe/6.0.0` → `lib/net461/` |
| 7 | `winetricks dotnet472` hung on `remove_mono` loop | known winetricks bug | killed by PID, stayed on `dotnet40` (sufficient — app doesn't use 4.7.2-only APIs) |

Final `Steam Desktop Authenticator.exe.config` `<runtime>` block (added
two binding redirects for clarity, even though DLLs matched exactly):
```xml
<runtime>
  <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
    <dependentAssembly>
      <assemblyIdentity name="System.Runtime.CompilerServices.Unsafe"
        publicKeyToken="b03f5f7f11d50a3a" culture="neutral" />
      <bindingRedirect oldVersion="0.0.0.0-6.0.0.0" newVersion="6.0.0.0" />
    </dependentAssembly>
    <dependentAssembly>
      <assemblyIdentity name="Newtonsoft.Json"
        publicKeyToken="30ad4fe6b2a6aeed" culture="neutral" />
      <bindingRedirect oldVersion="0.0.0.0-13.0.0.0" newVersion="13.0.0.0" />
    </dependentAssembly>
    <dependentAssembly>
      <assemblyIdentity name="CommandLine"
        publicKeyToken="de6f01bd326f8c32" culture="neutral" />
      <bindingRedirect oldVersion="0.0.0.0-1.9.71.2" newVersion="1.9.71.2" />
    </dependentAssembly>
  </assemblyBinding>
</runtime>
```

Files added to the app folder:

| File | Size | Assembly version | Source |
|------|------|------------------|--------|
| `CommandLine.dll` | 58368 B | 1.9.71.2 | NuGet `commandlineparser/1.9.71` (lib/net40) |
| `Newtonsoft.Json.dll` | 584976 B | 13.0.0.0 | NuGet `newtonsoft.json/13.0.3` (lib/net40) |
| `System.Runtime.CompilerServices.Unsafe.dll` | 18024 B | 6.0.0 | NuGet 6.0.0 (lib/net461) |

Launcher at `/home/den/Рабочий стол/steam-desktop-authenticator.desktop`:
```ini
[Desktop Entry]
Type=Application
Name=Steam Desktop Authenticator
Comment=Steam 2FA (Wine, win32 prefix)
Exec=env WINEPREFIX=/home/den/.wine32 WINEDEBUG=-all wine "/media/den/D/data/MYBD/SDA1/Steam Desktop Authenticator.exe"
Icon=wine
Terminal=false
Categories=Game;
```

Final launch (works):
```bash
WINEPREFIX=/home/den/.wine32 WINEDEBUG=-all \
  wine "/media/den/D/data/MYBD/SDA1/Steam Desktop Authenticator.exe"
```

## Key lessons

1. **Match the prefix to the binary.** 32-bit .exe → dedicated
   `WINEARCH=win32` prefix. Verify purity: no `syswow64/`.
2. **«Could not load assembly» is usually a missing NuGet DLL, not a
   Wine bug.** Fetch the package from nuget.org, extract the right
   `lib/netXX/` DLL, drop next to the `.exe`.
3. **NuGet package version ≠ assembly version.** The error cites the
   assembly version; verify the DLL's manifest with `monodis`.
4. **One package ships multiple DLLs.** Different `lib/netXX/`
   variants have different assembly versions — pick the one matching
   the error and the app's `<supportedRuntime>`.
5. **`FileLoadException: manifest does not match`** = wrong DLL
   variant. Replace the DLL (Option A) rather than fighting binding
   redirects (Option B) when possible.
6. **winetricks can hang on `remove_mono`**. Kill by PID; don't use
   `pkill -f winetricks` (it can match its own shell and hang).
7. **dotnet40 is often enough** for apps that declare 4.7.2 — try it
   first, escalate only if a 4.7.2-only API is actually used.
8. **`.desktop` launchers must set `WINEPREFIX` explicitly** —
   `wine-binfmt` and double-click always use the default `~/.wine`.
9. **Files inside a cryfs/FUSE vault are transparent to Wine.** No
   need to copy out; just open the vault before launching and close
   after.

## Diagnostic commands cheat sheet

```bash
file <exe>                              # PE32 (i386) = 32-bit; PE32+ = 64-bit
strings <dll> | grep -E "^[0-9]+\."     # quick assembly version probe
monodis --assembly <dll> | grep Version # authoritative assembly version
WINEPREFIX=... winetricks list-installed
WINEPREFIX=... wineserver -k            # stop all wine procs in that prefix
WINEDEBUG=-all wine ...                 # silent launch
WINEDEBUG=warn wine ...                 # show err:/warn:, hide fixme:
pgrep -af <pattern>                     # list PIDs before kill
mountpoint <path>                       # is a FUSE/cryfs mount active?
curl -sL "https://api.nuget.org/v3-flatcontainer/<pkg>/index.json"
```

## Persistence of this skill

Active whenever the user is trying to run a Windows `.exe` under Wine
and hits an error, or asks for a Wine app shortcut, or needs to fetch a
missing .NET dependency. Stays available across turns; no on/off toggle
needed.
