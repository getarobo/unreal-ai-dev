# JetBrains Rider Setup for Unreal Engine 5.6

Companion to [SETUP.md](./SETUP.md). Covers everything from install through your daily Live-Coding + Claude-Code-in-terminal workflow.

> **Note on the product name.** "Rider for Unreal Engine" (the separate preview product) was sunset and folded into the main **JetBrains Rider** in 2024. Use plain **Rider** today — it has full UE support built in.

## 1. Install Rider

- **macOS / Windows / Linux:** [https://www.jetbrains.com/rider/download/](https://www.jetbrains.com/rider/download/) — latest 2026.x.
- **License:** Personal/educational/non-commercial license is free; commercial needs JetBrains All Products Pack or Rider standalone.
- **macOS specifics:** Apple Silicon native build is the default. Install Xcode Command Line Tools first: `xcode-select --install`.
- **Windows specifics:** Install Visual Studio Build Tools 2022 with the "Game development with C++" workload (Rider uses MSVC for compiling but doesn't need full VS). If you already have Visual Studio installed, even better.

After install, launch Rider once → through the welcome screen choose **Customize → Plugins** and confirm these bundled plugins are enabled:

- **Unreal Engine** — ships bundled with Rider 2024+; gives you `.uproject` understanding, UCLASS/UPROPERTY awareness, Live Coding integration.
- **Terminal** — so you can run `claude` in a side panel.

**Note on C++ language support:** Rider's C++ engine (ReSharper C++ analyzer) is built into the product itself — there is no separate "clangd" or "C++" plugin to find or enable. Debuggers (LLDB on macOS/Linux, MSVC/LLDB on Windows) are also built-in. An experimental clangd-based engine exists under **Settings → Languages & Frameworks → C++ → Language Engine** in recent versions — leave it on the default ReSharper engine; it's what every Rider+UE write-up assumes.

## 2. Open the project

From Rider's welcome screen → **Open** → select your `.uproject` (**not** the `.sln` on Windows, and not the `Source/` folder — pick the `.uproject` itself).

Rider will:
1. Read `.uproject` and detect the UE install
2. Run `GenerateProjectFiles` automatically
3. Prompt to install **RiderLink** into `Plugins/Developer/RiderLink/` — say **Yes** (this is the helper plugin that lets Rider talk to the running editor)
4. Index the project (5–15 min on first open, faster on warm cache)

If Rider can't find the UE 5.6 install automatically, point it explicitly: **File → Settings → Build, Execution, Deployment → Unreal Engine → Engine root**.

## 3. Pick your build configuration

Top toolbar → configuration dropdown. The ones you actually use:

| Config | Use for |
|---|---|
| `<ProjectName>Editor \| Development Editor` | **Daily work.** Launches the editor with your code attached, debuggable. |
| `<ProjectName>Editor \| DebugGame Editor` | Slower, more debuggable. Use when you're hunting a tricky bug. |
| `<ProjectName> \| Development` | Standalone game (no editor). For quick play-test of packaged behavior. |
| `<ProjectName> \| Shipping` | Final build for distribution. Rarely needed during iteration. |

Run target = "Development Editor" by default. Press **Shift+F10** (Run) or **Shift+F9** (Debug).

## 4. Live Coding workflow

This is the killer feature — recompile C++ without restarting the editor.

**Enable Live Coding (once per project):**
- In Unreal Editor: **Edit → Editor Preferences → Live Coding** → "Enable Live Coding" ✓
- Default hotkey: **Ctrl+Alt+F11** (Windows/Linux), **Cmd+Alt+F11** (macOS)

**Daily loop:**
1. Editor running in Debug from Rider
2. Claude Code in a terminal (Rider has a built-in terminal panel — `Alt+F12`) writes/edits C++
3. Save the file in Rider (or Claude saves it directly)
4. Hit Live Coding hotkey → patch compiles and hot-applies, usually 3–10s
5. Test in Play-In-Editor immediately

Live Coding **does not** handle:
- Header changes that alter UCLASS/USTRUCT layout (UPROPERTY add/remove) → restart editor
- New classes (sometimes works, often requires restart)
- Module reorganization

When Live Coding refuses, do a full **Build** (Ctrl+F9) and relaunch the editor.

**Hot Reload is deprecated** — don't use it.

## 5. Debugging

- **Attach to a running editor:** Run → "Attach to Process…" → pick your `UnrealEditor` instance
- **Launch + debug from Rider:** select "DebugGame Editor", click the green bug icon
- Breakpoints work in `.cpp` and most engine headers if you have Engine source attached
- **Engine source for stepping into engine code:** Epic Games Launcher → UE 5.6 → "Options" → check "Engine Source". Re-open project; Rider will index Engine source (slow but worth it).

Debug toolbar: **F8** step over, **F7** step in, **Shift+F8** step out, **F9** resume.

## 6. Code style for UE

Rider auto-detects UE coding conventions but a few setting flips help:

- **Settings → Editor → Code Style → C/C++ → Naming Convention** — Rider applies Unreal-style prefixes (`A`, `U`, `F`, `T`, `I`) when generating classes/structs via "New → Unreal Class…". Leave default.
- **Settings → Editor → Inspections → C/C++** — keep "Unreal Engine" inspection group enabled (catches missing `UCLASS()`, wrong `UPROPERTY` macros, GC-unsafe raw pointers).
- **Settings → Languages & Frameworks → C++ → Unreal Engine** — confirm "Generate Visual Studio project files when opening Unreal project" is on.

For consistent formatting with Claude's output, add `.editorconfig` at the project root:

```ini
root = true

[*.{h,cpp,cs}]
indent_style = tab
indent_size = 4
tab_width = 4
end_of_line = crlf      # use lf on macOS/Linux
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

## 7. Pairing with Claude Code in terminal

Recommended layout:

```
┌──────────────────────────────┬──────────────────────────┐
│                              │                          │
│   Rider editor (C++ code)    │   Terminal: claude       │
│                              │   --model opus           │
│                              │                          │
├──────────────────────────────┤   (or Sonnet for grind)  │
│                              │                          │
│   Rider terminal: build,     │                          │
│   git, logs                  │                          │
│                              │                          │
└──────────────────────────────┴──────────────────────────┘
```

- **Rider terminal panel** (`Alt+F12`): use for `git`, `clang-format`, quick scripts
- **External terminal** (iTerm2/WezTerm/Windows Terminal): use for `claude` — gives more buffer and survives Rider restarts

When Claude edits a file, Rider notices and offers "External changes detected → Reload" — accept. Don't have both Rider and Claude edit the same file in the same minute.

**Useful Rider actions for the Claude workflow:**
- `Cmd/Ctrl+Shift+A` → action search → "Reload from Disk" if Rider lags noticing Claude's edit
- `Cmd/Ctrl+E` → recent files (often shows the file Claude just touched)
- `Cmd/Ctrl+Shift+F` → grep across project (faster than Claude's own grep for large symbol searches)

## 8. Optional: turn off the JetBrains AI features you don't need

Rider 2026 ships with JetBrains AI Assistant. It will not break Claude Code, but **disable inline completion** to avoid double-suggestions while typing:

- **Settings → Tools → AI Assistant → Enable AI completion** ✗

Keep "AI Actions" enabled if you like Rider-side explain/refactor — they call JetBrains models, not Claude, so no conflict.

## 9. Daily checklist

Before starting a session:

1. Pull latest → `git pull` (in Rider terminal)
2. **Open editor first** (Run "Development Editor" in Rider, or launch from the .uproject)
3. Wait for shader compile to settle
4. Open external terminal → `cd <project> && claude --model opus`
5. First prompt: "Read CLAUDE.md, gdep me current state, then …"

## 10. Common Rider+UE issues

| Symptom | Fix |
|---|---|
| Rider stuck "Indexing…" forever | Invalidate caches: **File → Invalidate Caches… → Invalidate and Restart**. Re-open .uproject. |
| `cannot find UnrealHeaderTool` | UE install path wrong. **Settings → Unreal Engine → Engine root** → fix path. |
| Live Coding silently does nothing | Editor not running with Live Coding enabled. Toggle off + on in Editor Preferences. |
| Breakpoints "won't be hit, no symbols" | Wrong configuration (Shipping instead of DebugGame Editor). Switch. |
| New `.h` added, intellisense missing | **Tools → Unreal Engine → Refresh Unreal Project**. |
| RiderLink build fails after engine update | Delete `Plugins/Developer/RiderLink/`, reopen project, accept "install RiderLink" prompt to regenerate. |
| MSVC errors only on Windows after Claude edit | Line endings (CRLF/LF). Set `.editorconfig` end_of_line = crlf on Windows. |
| Live Coding rejects header change | Restart editor — UCLASS layout change. |

## 11. References

- [JetBrains Rider Unreal Engine docs](https://www.jetbrains.com/help/rider/Unreal_Engine.html)
- [Live Coding (Epic docs)](https://docs.unrealengine.com/5.6/en-US/using-live-coding-to-recompile-unreal-engine-applications-at-runtime/)
- [RiderLink plugin (JetBrains)](https://github.com/JetBrains/UnrealLink)
- [SharkPillow: Rider + Claude Code setup](https://sharkpillow.com/post/rider-claude/) — the write-up this guide draws on
