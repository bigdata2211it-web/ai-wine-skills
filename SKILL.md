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

See `_content/wine-exe-fix.md` for the full skill body. This file is
the opencode-facing frontmatter stub; opencode loads
`opencode/wine-exe-fix/SKILL.md`, which in turn references the shared
content.

For the multi-CLI packaging (Claude Code, Cursor, Codex, Cline) see
the sibling `mybd-cryfs` skill's layout in this repo. To install into
opencode:

```bash
cp -r opencode/wine-exe-fix ~/.config/opencode/skills/
```
