# ai-wine-skills

AI agent skill: **wine-exe-fix** — diagnose and fix Windows `.exe` launch
failures under Wine on Linux. Packaged for **multiple CLI/IDE tools**: opencode,
Claude Code, Cursor, Codex CLI, and Cline. Same content, different wrappers.

## What the skill does

When a Windows `.exe` (especially a 32-bit .NET / WinForms application)
refuses to run under Wine, this skill walks the agent through a triage
workflow that resolves the most common causes without reinstalling
Windows or spinning up a VM:

- 32-bit vs 64-bit Wine prefix selection (`WINEARCH=win32` vs default `~/.wine`)
- Missing .NET runtime (Wine Mono / MS .NET via `winetricks dotnet40`)
- Missing NuGet-managed dependencies (`CommandLine`,
  `Newtonsoft.Json`, `System.Runtime.CompilerServices.Unsafe`, ...)
- The critical distinction between **NuGet package version** and **.NET
  assembly version** — and how one `.nupkg` ships multiple DLLs with
  different assembly versions under `lib/net35/`, `lib/net40/`,
  `lib/net45/`, ...
- `FileLoadException: manifest does not match (0x80131040)` — wrong DLL
  variant; replace the DLL (preferred) over fighting binding redirects
- binding redirects in `.exe.config` `<assemblyBinding>` as a fallback
- **Missing DirectX runtime** — `d3dx9`, `d3dcompiler_47`, `dxdiag`,
  `xinput`/`xact`, optional `dxvk` (Vulkan-based d3d9/10/11) when a
  Vulkan-capable GPU is available
- **Missing fonts** — `corefonts`, `cjkfonts`, `tahoma`, plus host
  fontconfig integration (`~/.local/share/fonts/` + `fc-cache -fv`)
  for app-hardcoded fonts like Segoe UI
- **Missing Visual C++ runtimes** — `vcrun6`, `vcrun2005..vcrun2019`
  with the missing-DLL → verb mapping table (MSVCR120, MSVCP140,
  vcruntime140, concrt140, etc.)
- **MSI / InstallShield / NSIS installers** — `wine msiexec /i`,
  running bundled `Setup.exe`, `winetricks isscript` for old
  InstallShield, `winecfg` Windows version override
- **COM / ActiveX registration** — `wine regsvr32 <dll>` when an app
  fails with `CoGetClassObject no class object`
- **Registry probes** — using `WINEDEBUG=reg` to find missing keys and
  `wine reg add` / `wine regedit` to plant them (fake IE/.NET/Office
  presence without installing those)
- The `winetricks dotnetNNN` `remove_mono internal` loop hang and how to
  kill it safely (by PID, **not** `pkill -f winetricks` — that can hang
  matching its own shell)
- Creating reusable `.desktop` launchers that pin a custom `WINEPREFIX`
  (because `wine-binfmt` and double-click always use the default `~/.wine`)

A worked example — Steam Desktop Authenticator under Debian 13 + KDE
Plasma + Wine 10.0 — is included, walking through seven distinct errors
in the order they surfaced and the fix for each.

Trigger keywords: "exe не запускается под wine", "wine ошибка 32/64",
"Could not load assembly", "Wine Mono is not installed",
"FileLoadException manifest does not match", "Failed to create
Direct3D device", "missing MSVCR140.dll", "tofu boxes wine", "сделай
ярлык для wine приложения".

### Out of scope (the skill will say so and stop)

- **Anti-cheat / kernel-level DRM** (EAC, BattlEye, Denuvo) — needs
  Windows or a VM with GPU passthrough.
- **Copy-protected installers** (SecuROM, StarForce, SafeDisc) — DRM
  locked; needs Windows or a no-CD patch where legal.
- **16-bit installers** — Wine dropped 16-bit support on 64-bit hosts;
  use a VM (or DOSBox for pure DOS).
- **GPU/driver problems** — fix the host driver (NVIDIA/AMD/Mesa)
  first; the skill won't help with shader crashes from a broken
  driver.

## Repo layout

```
wine-exe-fix/
  SKILL.md                              # skill-level entry (frontmatter + full body)
  _content/wine-exe-fix.md              # agnostic body (no frontmatter) — source of truth
  opencode/wine-exe-fix/SKILL.md        # opencode wrapper
  claude-code/wine-exe-fix/SKILL.md     # Claude Code wrapper
  cursor/wine-exe-fix/wine-exe-fix.mdc  # Cursor wrapper
  codex/wine-exe-fix/AGENTS.md          # Codex CLI wrapper
  cline/wine-exe-fix/wine-exe-fix.md    # Cline wrapper
```

## Pick your tool

| Tool | Path in this repo | Install target | Format |
|------|-------------------|----------------|--------|
| [opencode](https://opencode.ai) | `opencode/wine-exe-fix/SKILL.md` | `~/.config/opencode/skills/wine-exe-fix/SKILL.md` | frontmatter `name`/`description` + body |
| [Claude Code](https://claude.com/claude-code) | `claude-code/wine-exe-fix/SKILL.md` | `~/.claude/skills/wine-exe-fix/SKILL.md` | same frontmatter, skills dir |
| [Cursor](https://cursor.com) | `cursor/wine-exe-fix/wine-exe-fix.mdc` | `.cursor/rules/wine-exe-fix.mdc` | `.mdc` with `description`/`globs`/`alwaysApply` |
| [Codex CLI](https://github.com/openai/codex) | `codex/wine-exe-fix/AGENTS.md` | `AGENTS.md` at project root (or `~/.codex/AGENTS.md`) | plain markdown |
| [Cline](https://cline.bot) | `cline/wine-exe-fix/wine-exe-fix.md` | `.clinerules/wine-exe-fix.md` | plain markdown in `.clinerules/` |

Agnostic body (no wrapper, for porting to other tools) lives at
`_content/wine-exe-fix.md`.

## Install

### opencode

```bash
cp -r opencode/wine-exe-fix ~/.config/opencode/skills/
```
Restart opencode. Or register the repo as a skill source:
```json
{ "skills": { "urls": ["https://github.com/bigdata2211it-web/ai-wine-skills"] } }
```

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -r claude-code/wine-exe-fix ~/.claude/skills/
```
Restart Claude Code. The skill is auto-discovered from `~/.claude/skills/`.

### Cursor

Copy the `.mdc` file into your project's rules folder:
```bash
mkdir -p .cursor/rules
cp cursor/wine-exe-fix/wine-exe-fix.mdc .cursor/rules/
```
`alwaysApply: false` means it triggers on keyword matches / manual `@`-mention.
Set `alwaysApply: true` if you want it in every request.

### Codex CLI

Drop `AGENTS.md` at the project root (or merge into an existing one):
```bash
cat codex/wine-exe-fix/AGENTS.md >> ./AGENTS.md
```
For global use, place at `~/.codex/AGENTS.md`.

### Cline

Place the rule file in the project's `.clinerules/` folder:
```bash
mkdir -p .clinerules
cp cline/wine-exe-fix/wine-exe-fix.md .clinerules/
```
Cline auto-loads everything under `.clinerules/`.

## Skill format (per tool)

### opencode / Claude Code
```
wine-exe-fix/
  SKILL.md    # frontmatter (name, description) + markdown body
```

### Cursor
```
.cursor/rules/
  wine-exe-fix.mdc   # frontmatter (description, globs, alwaysApply) + markdown body
```

### Codex CLI
```
AGENTS.md           # plain markdown, instructions for the Codex agent
```

### Cline
```
.clinerules/
  wine-exe-fix.md   # plain markdown, auto-loaded
```

## Porting to another tool

Take `_content/wine-exe-fix.md` (no frontmatter) and wrap it in your tool's
format. Open an issue/PR if you add a new wrapper.

## Verification checklist (after fixing an .exe)

- [ ] `file <exe>` reports the expected architecture (PE32 i386 or PE32+ x86-64).
- [ ] The correct `WINEPREFIX` is used (dedicated `win32` prefix for 32-bit .NET).
- [ ] In a pure `win32` prefix, `~/.wine32/drive_c/windows/syswow64` does NOT exist.
- [ ] The .NET runtime is installed (`winetricks list-installed` shows `dotnet40`+).
- [ ] Every missing assembly named in an error has its DLL next to the `.exe`.
- [ ] Each DLL's assembly version (via `monodis --assembly`) matches the error's request.
- [ ] If a binding redirect was added, `newVersion` matches the DLL's actual assembly version.
- [ ] The app launches without `Unhandled Exception` and stays alive past startup.
- [ ] A `.desktop` launcher exists with `Exec=env WINEPREFIX=... wine ... "/path/to/App.exe"`.
- [ ] Double-clicking the launcher opens the app in the correct prefix.

## License

MIT
