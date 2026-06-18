# wine-exe-fix — diagnose and fix Windows .exe launch failures under Wine

> Skill/instruction set for Codex CLI. Trigger keywords: "exe не запускается
> под wine", "wine ошибка 32/64", "Could not load assembly", "Wine Mono is not
> installed", "FileLoadException manifest does not match", "Failed to create
> Direct3D device", "missing MSVCR140.dll", "tofu boxes wine", "сделай ярлык
> для wine приложения". Apply when the user wants to run a Windows .exe under
> Wine and hits an error, or asks for a Wine app shortcut, or an agent needs
> to triage a Wine launch failure for a .NET/WinForms or native Win32
> application.


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
| Missing DirectX (9/10/11) runtime or DLLs | Yes — `winetricks d3dx9 d3dcompiler_47 dxdiag` etc. |
| Missing fonts (tofu / wrong glyphs) | Yes — `winetricks corefonts cjkfonts` |
| Missing Visual C++ runtime | Yes — `winetricks vcrun6 vcrun2005 vcrun2008 vcrun2010 vcrun2012 vcrun2013 vcrun2019` |
| Missing COM/ActiveX component | Partially — `regsvr32 <dll>`; DCOM/config beyond registry is out of scope |
| MSI installer with custom actions | Partially — `wine msiexec /i`; native InstallShield bootstrappers often fail |
| Registry stub missing / wrong value | Yes — `wine reg add` / `wine regedit` to plant the expected keys |
| Anti-cheat / kernel-level DRM (EAC, BattlEye, Denuvo) | **No** — needs Windows or a VM with GPU passthrough |
| GPU/driver issues (shader crashes, no hw accel) | Out of scope — fix the host driver (NVIDIA/AMD/Mesa) first |
| copy-protected installers (SecuROM, StarForce, SafeDisc) | **No** — DRM-locked; needs Windows or no-CD patch |
| 16-bit installers | **No** — Wine dropped 16-bit support on 64-bit hosts; use a VM (DOSBox for pure DOS) |

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

### Step 6 — Provide DirectX runtime and DLLs

Common symptoms:
- `err:d3d:...` or `fixme:d3d:...` spam in the log, then crash.
- App complains about missing `d3dx9_XX.dll`, `d3dcompiler_XX.dll`,
  `D3DX9_43.dll`, `xinput1_3.dll`, `xactengine*.dll`.
- Black window, no rendering, or "Failed to create Direct3D device".

Fix via winetricks (install into the same prefix as the app):
```bash
WINEPREFIX=/home/USER/.wine32 winetricks --unattended \
  d3dx9 d3dx10 d3dx11_42 d3dx11_43 \
  d3dcompiler_47 d3dcompiler_43 \
  dxdiag dxvk \
  xinput xact
```
Verb glossary:
- `d3dx9` — installs the whole DirectX 9 D3DX family (d3dx9_24..43).
- `d3dcompiler_47` — modern HLSL compiler, used by many .NET/WinForms
  apps that ship CefSharp or WPF.
- `dxdiag` — the DirectX Diagnostic Tool + the `dxgi.dll`/`d3d11.dll`
  stubs some apps probe for at startup.
- `xinput` / `xact` — Xbox controller + XAudio (game-ish apps).
- `dxvk` — **optional but high-impact**: replaces Wine's d3d9/10/11
  with Vulkan-based implementations via DXVK. Big win for games and
  for .NET apps that use hardware-accelerated rendering. Requires
  Vulkan-capable GPU + `vulkan-tools` installed on the host. Verify:
  `vulkaninfo | head -5`. If no Vulkan, skip `dxvk`.

For apps that bundle their own DirectX installer (`DXSETUP.exe`,
`vcredist_x86.exe`), prefer running those bundled installers **inside
the prefix** over winetricks when possible:
```bash
WINEPREFIX=/home/USER/.wine32 wine "/path/to/app/Redist/DXSETUP.exe"
```

### Step 7 — Provide fonts (fixes "tofu" boxes and broken layout)

Symptoms:
- Text shows as rectangles/boxes ("tofu").
- WinForms controls render with wrong metrics, UI overlaps.
- App crashes drawing a specific font that isn't present.

Fix:
```bash
WINEPREFIX=/home/USER/.wine32 winetricks --unattended corefonts
# for CJK / Cyrillic / other scripts:
WINEPREFIX=/home/USER/.wine32 winetricks --unattended cjkfonts
# also: install tahoma (many MS apps hardcode Tahoma):
WINEPREFIX=/home/USER/.wine32 winetricks --uninstalled tahoma
```
If the app hardcodes a font not in `corefonts` (e.g. `Segoe UI`), the
cleanest fix is to install the font system-wide on the host and let
fontconfig expose it to Wine:
```bash
sudo apt install fonts-dejavu fonts-liberation
# copy any custom .ttf to ~/.local/share/fonts/ and run:
fc-cache -fv
```
Wine reads fontconfig; host fonts become available without restarting
the prefix.

### Step 8 — Provide Visual C++ runtime (MSVCRT / VC++ Redistributable)

Many apps ship without the VC++ runtime they were built against. Error
variants:
- `err:module:import_dll Library MSVCR120.dll not found`
- `err:module:import_dll Library MSVCP140.dll not found`
- App launches then immediately exits with no message (missing
  `_Crt*` symbols).

Map the missing DLL to the right verb:
| Missing DLL | winetricks verb |
|-------------|-----------------|
| `MSVCRT.dll`, `MSVCR60.dll` | `vcrun6` |
| `MSVCR80.dll`, `MSVCP80.dll` | `vcrun2005` |
| `MSVCR90.dll`, `MSVCP90.dll` | `vcrun2008` |
| `MSVCR100.dll`, `MSVCP100.dll` | `vcrun2010` |
| `MSVCR110.dll`, `MSVCP110.dll` | `vcrun2012` |
| `MSVCR120.dll`, `MSVCP120.dll` | `vcrun2013` |
| `MSVCR140.dll`, `MSVCP140.dll`, `vcruntime140.dll`, `concrt140.dll` | `vcrun2015` (also installs 2017/2019/2022) |
| `MSVCP140_1.dll`, `MSVCP140_2.dll` | `vcrun2019` (newer variant) |

Install:
```bash
WINEPREFIX=/home/USER/.wine32 winetricks --unattended vcrun2019
```
Install only what the error names — installing all of them at once can
cause DLL conflicts (different MSVCRT versions overwrite each other).

### Step 9 — MSI installers and bundled redistributables

If the app ships as `setup.msi` or `Setup.exe` (InstallShield/NSIS/Inno):
```bash
WINEPREFIX=/home/USER/.wine32 wine msiexec /i "/path/to/setup.msi"
# or for .exe bootstrappers:
WINEPREFIX=/home/USER/.wine32 wine "/path/to/Setup.exe"
```
Tips:
- Add `/qn` for silent MSI: `msiexec /i setup.msi /qn` (still uses
  the prefix's C: drive).
- InstallShield `Setup.exe` that fails with "ISScript.msi" error →
  install `winetricks isscript` first.
- For very old NSIS installers that hang at "Extracting...", running
  with `WINEDEBUG=-all` can help (NSIS spams fixme channels).
- If the installer needs Windows version 10 set: open
  `WINEPREFIX=/home/USER/.wine32 winecfg` → Windows version → Windows 10.

### Step 10 — Register COM / ActiveX components

If the error is `err:ole:CoGetClassObject no class object` for a CLSID,
or the app tries `regsvr32 <dll>` and fails, register the DLL manually:
```bash
WINEPREFIX=/home/USER/.wine32 wine regsvr32 "/path/to/component.dll"
```
For 32-bit DLLs in a win32 prefix this just works. For 64-bit DLLs in
a wow64 prefix use `wine64 regsvr32`.

If the CLSID is not in any DLL you have (the component is genuinely
missing from the app's bundle), this is out of scope — Wine cannot
invent a COM object. Look for a redistributable installer from the
vendor (e.g. "MS XML Core Services", "WebView2 Runtime").

### Step 11 — Plant registry keys the app probes for

Some apps check for registry keys before doing work (e.g. "is
Internet Explorer >= 7?", "is .NET 4 installed?", "is Office
installed?") and bail out if the key is missing even when the actual
feature isn't used.

Inspect what's missing by running with `WINEDEBUG=reg`:
```bash
WINEPREFIX=/home/USER/.wine32 WINEDEBUG=reg wine "/path/to/App.exe" 2>&1 | \
  grep -iE "RegOpenKey|RegQueryValue" | head -30
```
Look for keys returning `STATUS_OBJECT_NAME_NOT_FOUND`. Plant them:
```bash
WINEPREFIX=/home/USER/.wine32 wine reg add \
  "HKLM\\Software\\Microsoft\\Internet Explorer" /v Version /t REG_SZ \
  /d "9.11.10240.16384" /f
```
Or edit interactively:
```bash
WINEPREFIX=/home/USER/.wine32 wine regedit
```
Back up the registry first:
```bash
cp -r /home/USER/.wine32/system.reg /home/USER/.wine32/system.reg.bak
cp -r /home/USER/.wine32/user.reg  /home/USER/.wine32/user.reg.bak
```
Common keys apps probe:
- `HKLM\Software\Microsoft\Internet Explorer\Version` — fake IE version.
- `HKLM\Software\Microsoft\NET Framework Setup\NDP\v4\Full\Version` —
  fake .NET 4.x presence (only if `dotnet40` already installed).
- `HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall\...` —
  install records some launchers check.

### Step 12 — Create a `.desktop` launcher (so the user can double-click)

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
10. **DirectX/D3DX missing is the #1 cause of "Failed to create
    Direct3D device"**. `winetricks d3dx9 d3dcompiler_47` fixes most
    .NET WinForms + CefSharp apps. `dxvk` is a major upgrade for
    GPU-accelerated rendering if Vulkan is available.
11. **Fonts come from fontconfig**. If an app hardcodes a font Wine
    doesn't have, install the `.ttf` on the host
    (`~/.local/share/fonts/`) and run `fc-cache -fv` — Wine picks it
    up without a prefix restart.
12. **VC++ runtime: install only the version the error names.**
    Installing all `vcrun*` verbs at once causes DLL conflicts —
    different MSVCRT versions overwrite each other. Map the missing
    DLL to the right verb (see Step 8 table).
13. **`WINEDEBUG=reg` reveals registry probes.** Apps that bail with
    "needs IE 7" or "needs Office" usually just check a registry key —
    plant it via `wine reg add` or `wine regedit` instead of actually
    installing IE/Office.
14. **Prefer the app's bundled redistributables over winetricks** when
    the app ships `Redist/DXSETUP.exe` or `vcredist_x86.exe` — the
    app expects its exact version.
15. **Anti-cheat / kernel DRM / copy-protected installers cannot be
    fixed in Wine.** Stop early and recommend a Windows VM with GPU
    passthrough (or a no-CD patch for old games, where legal).

## Diagnostic commands cheat sheet

```bash
file <exe>                              # PE32 (i386) = 32-bit; PE32+ = 64-bit
strings <dll> | grep -E "^[0-9]+\."     # quick assembly version probe
monodis --assembly <dll> | grep Version # authoritative assembly version
WINEPREFIX=... winetricks list-installed
WINEPREFIX=... winetricks dlls list     # show available winetricks verbs
WINEPREFIX=... wineserver -k            # stop all wine procs in that prefix
WINEPREFIX=... wineboot -r              # restart the prefix (re-scan fonts/reg)
WINEPREFIX=... winecfg                  # GUI: windows version, drives, audio
WINEPREFIX=... wine regedit             # GUI registry editor
WINEPREFIX=... wine reg add "HKLM\\..." /v X /t REG_SZ /d Y /f  # plant a key
WINEPREFIX=... wine regsvr32 <dll>      # register COM/ActiveX component
WINEPREFIX=... wine msiexec /i setup.msi /qn   # silent MSI install
WINEDEBUG=-all wine ...                 # silent launch
WINEDEBUG=warn wine ...                 # show err:/warn:, hide fixme:
WINEDEBUG=reg wine ... 2>&1 | grep RegOpenKey   # see what registry keys the app probes
WINEDEBUG=module wine ... 2>&1 | grep import_dll  # see what DLLs the app fails to load
pgrep -af <pattern>                     # list PIDs before kill
mountpoint <path>                       # is a FUSE/cryfs mount active?
vulkaninfo | head -5                    # is Vulkan available for DXVK?
fc-cache -fv                            # rebuild font cache (Wine reads fontconfig)
curl -sL "https://api.nuget.org/v3-flatcontainer/<pkg>/index.json"
```

## Persistence of this skill

Active whenever the user is trying to run a Windows `.exe` under Wine
and hits an error, or asks for a Wine app shortcut, or needs to fetch a
missing .NET dependency. Stays available across turns; no on/off toggle
needed.
