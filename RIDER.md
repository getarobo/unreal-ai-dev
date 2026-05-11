# JetBrains Rider for Unreal Engine 5.6

Companion to [SETUP.md](./SETUP.md). Covers Rider-specific setup, Live Coding, debugging, and the daily Rider + terminal-Claude-Code loop.

Every claim below is sourced to the JetBrains help pages, the [JetBrains/UnrealLink](https://github.com/JetBrains/UnrealLink) repo, or the Epic UE community wiki. Sources linked inline.

> **Product naming.** Use plain **JetBrains Rider** today — the standalone "Rider for Unreal Engine" preview was folded into the main product. Rider 2024+ ships with full UE support built in.

## 1. Install Rider

Download from [jetbrains.com/rider/download](https://www.jetbrains.com/rider/download/).

- **macOS:** Apple Silicon native build is the default. Install Xcode Command Line Tools: `xcode-select --install`.
- **Windows:** install Visual Studio Build Tools 2022 with the **Game development with C++** workload. (Rider uses MSVC for compiling on Windows but does not require the full VS IDE.)
- **License:** free for personal/educational use, requires a subscription for commercial.

## 2. Open the project — RiderLink installs itself

Per the [JetBrains UnrealLink/RiderLink docs](https://www.jetbrains.com/help/rider/Unreal_Engine__UnrealLink_RiderLink.html):

> You can work with the `.uproject` directly in JetBrains Rider, without generating a Visual Studio solution, Xcode project files, or any extra project models like Makefiles. You can either drag-drop the `.uproject` file into the Rider App or browse directly to the file from inside Rider by clicking "Open" in the main window of Rider.

On first open, Rider shows a notification: **"RiderLink plugin is missing"** with two install options:

| Option | Installs into | When to choose |
|---|---|---|
| **Engine** | `<UE_INSTALL>/Engine/Plugins/Developer/RiderLink/` | All projects on this engine version use the same RiderLink. Recommended if you're the only user of this engine install. |
| **Game** | `<Project>/Plugins/Developer/RiderLink/` | Per-project install. Use if you share the engine with other tools or other users. |

If you dismiss the notification, install it later: **Settings → Languages & Frameworks → Unreal Engine**, or **Ctrl+Shift+A** (Find Action) → search **"Force Install RiderLink in Engine"** / **"Force Install RiderLink in Game"**.

RiderLink is what gives Rider Blueprint integration, the colored/filterable UE log browser, and "Play in Editor" control from the IDE.

## 3. Bundled plugins to verify

After install, **File → Settings → Plugins → Installed**. These ship bundled, just confirm enabled:

- **Unreal Engine** — bundled UE support
- **Terminal** — built-in terminal panel (`Alt+F12`)

Rider's C++ language engine (ReSharper C++ analyzer) and debuggers (LLDB on macOS/Linux, MSVC/LLDB on Windows) are part of the product itself — no separate plugin to find. An experimental clangd engine exists under **Settings → Languages & Frameworks → C++ → Language Engine** in recent versions; leave the default ReSharper engine, which every Rider+UE guide assumes.

## 4. Build configurations

Top-toolbar configuration dropdown. The ones you actually use:

| Config | Use for |
|---|---|
| `<Project>Editor \| Development Editor` | **Daily work.** Launches the editor with your code attached, debuggable. |
| `<Project>Editor \| DebugGame Editor` | Slower but more debuggable — pick when hunting a tricky bug. |
| `<Project> \| Development` | Standalone game (no editor). For quick play-test of packaged behavior. |
| `<Project> \| Shipping` | Final build. Rarely needed during iteration. |

**Shift+F10** runs the selected config; **Shift+F9** debugs it.

## 5. Live Coding

The killer feature: recompile C++ without restarting the editor.

**Enable Live Coding (per project):**
- In the editor: click the dropdown arrow next to the **Compile** button → toggle **"Enable Live Coding"** ([UE community wiki](https://unrealcommunity.wiki/live-compiling-in-unreal-projects-tp14jcgs)).
- Or **Edit → Editor Preferences → Live Coding → "Enable Live Coding"** ✓.

**Default hotkey: `Ctrl+Alt+F11`** ([Epic Dev Community](https://dev.epicgames.com/community/learning/knowledge-base/GDdl/unreal-engine-live-coding-primer); also called out in [JetBrains issue RIDER-68333](https://youtrack.jetbrains.com/issue/RIDER-68333)). The hotkey works whether the IDE or the editor has focus.

> **macOS note:** Live Coding originally shipped Windows-only in UE 4.22; UE 5 added macOS/Linux support, but the hotkey on macOS may differ. Open **Editor Preferences → Live Coding** to confirm/rebind on first run. If the menu shows no Live Coding section, the feature isn't enabled on your build of UE 5.6 for macOS — fall back to a full Rider rebuild (Ctrl+F9) and editor relaunch.

**Daily loop:**
1. Run "Development Editor" from Rider — editor opens with code attached
2. Open a terminal (Rider's own `Alt+F12` panel works, or external) → `claude --model opus`
3. Claude edits C++ files
4. Hit Live Coding hotkey → patch compiles + hot-applies in 3–10s
5. Test in Play-In-Editor immediately

**Live Coding will NOT apply:**
- Changes that alter UCLASS/USTRUCT layout (add/remove UPROPERTY) → restart editor
- New classes (sometimes works; often needs restart)
- Module reorganization

For those cases, hit **Build** (Ctrl+F9) and relaunch the editor.

**Hot Reload is deprecated** — don't use it.

## 6. Debugging

- **Launch + debug from Rider:** select "DebugGame Editor", click the bug icon.
- **Attach to a running editor:** **Run → Attach to Process…** → pick the `UnrealEditor` process.
- **Step into engine code:** in Epic Games Launcher → UE 5.6 → ⋯ → "Options" → check **Engine Source**, then re-open the project. Rider will index Engine source (slow but worth it).
- Standard JetBrains debug keys apply: **F8** step over, **F7** step in, **Shift+F8** step out, **F9** resume.

## 7. Code style for UE

`File → Settings → Editor`:

- **Code Style → C/C++ → Naming Convention** — Rider applies UE-style prefixes (`A`, `U`, `F`, `T`, `I`) when generating classes via "New → Unreal Class…". Leave default.
- **Inspections → C/C++** — keep the **Unreal Engine** inspection group enabled (catches missing UCLASS, wrong UPROPERTY macros, GC-unsafe raw pointers).

For consistent formatting between you and Claude's output, drop an `.editorconfig` at the project root:

```ini
root = true

[*.{h,cpp,cs}]
indent_style = tab
indent_size = 4
tab_width = 4
end_of_line = lf            # crlf on Windows
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

## 8. Pairing with Claude Code

Recommended layout:

```
┌─────────────────────────────┬──────────────────────────┐
│                             │                          │
│   Rider editor (C++)        │   External terminal:     │
│                             │   claude --model opus    │
│                             │                          │
├─────────────────────────────┤   (or sonnet for grind)  │
│                             │                          │
│   Rider terminal (Alt+F12): │                          │
│   git, builds, logs         │                          │
│                             │                          │
└─────────────────────────────┴──────────────────────────┘
```

External terminal (iTerm2 / WezTerm / Windows Terminal) is better for `claude` than Rider's built-in panel — more scrollback and it survives Rider restarts.

When Claude edits a file, Rider notices and prompts "External changes detected → Reload" — accept. Don't have both Rider and Claude editing the same file in the same minute.

Useful Rider actions during a Claude session:
- **Ctrl/Cmd+Shift+A** → Find Action — e.g. "Reload from Disk", "Force Install RiderLink"
- **Ctrl/Cmd+E** → recent files (often shows what Claude just touched)
- **Ctrl/Cmd+Shift+F** → grep across project, often faster than Claude's own grep for big symbol searches

## 9. Optional: disable JetBrains AI inline completion

Rider 2026 ships with JetBrains AI Assistant. It coexists with Claude Code but the inline-completion suggestions can step on each other while you type:

- **Settings → Tools → AI Assistant → Enable AI completion** → off

Keep AI Actions enabled if you like Rider-side explain/refactor — they call JetBrains' models, not Claude, so no conflict.

## 10. Daily checklist

1. `git pull` (Rider terminal)
2. Open the editor — Run "Development Editor" from Rider
3. Wait for shaders to settle
4. External terminal → `cd <project> && claude --model opus`
5. First prompt: "Read CLAUDE.md, gdep me current state, then …"

## 11. Common issues

| Symptom | Fix |
|---|---|
| Rider stuck "Indexing…" forever | **File → Invalidate Caches… → Invalidate and Restart**. Re-open `.uproject`. |
| `cannot find UnrealHeaderTool` | UE install path wrong. **Settings → Languages & Frameworks → Unreal Engine → Engine root** → fix. |
| Live Coding silently does nothing | Editor doesn't have it enabled. Toggle off + on in Editor Preferences → Live Coding, or via the dropdown next to Compile. |
| Breakpoints not hit ("no symbols") | Wrong configuration (Shipping instead of DebugGame Editor). Switch. |
| New `.h` not picked up | **Ctrl+Shift+A** → "Refresh Unreal Project". |
| RiderLink build fails after engine update | **Ctrl+Shift+A** → "Force Install RiderLink in Engine" (or Game). |
| MSVC errors only on Windows after Claude edit | Line endings (CRLF vs LF). Set `.editorconfig` `end_of_line = crlf` on Windows. |
| Live Coding rejects header change | UCLASS layout changed — restart the editor. |

## 12. Verified sources

- [JetBrains: UnrealLink and RiderLink](https://www.jetbrains.com/help/rider/Unreal_Engine__UnrealLink_RiderLink.html) — RiderLink install behavior, Engine vs Game options, Force Install actions
- [JetBrains/UnrealLink](https://github.com/JetBrains/UnrealLink) — what RiderLink does, plugin install path
- [UE Community Wiki: Live Coding](https://unrealcommunity.wiki/live-compiling-in-unreal-projects-tp14jcgs) — Ctrl+Alt+F11 default, enable via Compile dropdown
- [Epic Dev Community: Live Coding Primer](https://dev.epicgames.com/community/learning/knowledge-base/GDdl/unreal-engine-live-coding-primer)
- [JetBrains issue RIDER-68333](https://youtrack.jetbrains.com/issue/RIDER-68333) — confirms Ctrl+Alt+F11 default
- [Tom Looman: Setup Rider for C++ and Unreal Engine](https://tomlooman.com/setup-unreal-engine-cpp-rider/) — community walk-through
- [SharkPillow: Rider + Claude Code](https://sharkpillow.com/post/rider-claude/) — the pairing pattern this doc draws on
