# Setup: Unreal 5.6 C++ Project with chongdashu/unreal-mcp + Claude Code

Opinionated path for the stack recommended in [REPORT.md](./REPORT.md) §6, tuned for the profile in [CLAUDE.md](./CLAUDE.md): C++ project structure (BPs/Materials/Niagara/UMG fine), Rider, experimental visual-art with sensor/camera input.

Every command and path below was verified against the live `chongdashu/unreal-mcp` and `pirua-game/gdep` repos on 2026-05-11. Sources are linked inline.

## 0. Prerequisites

- Unreal Engine **5.6** installed via Epic Games Launcher
- [JetBrains Rider](https://www.jetbrains.com/rider/) 2026.x — see [RIDER.md](./RIDER.md)
- `claude` CLI authenticated (`claude --version` works)
- **Python 3.12+** ([chongdashu requirement](https://github.com/chongdashu/unreal-mcp/blob/main/Python/README.md))
- **`uv`** package manager (`curl -LsSf https://astral.sh/uv/install.sh | sh`, or `brew install uv`)
- **Node 18+** + **.NET Runtime 8.0+** (both required by [gdep](https://github.com/pirua-game/gdep))
- Git

## 1. Clone chongdashu/unreal-mcp once (outside any UE project)

The repo bundles three things:

```
chongdashu-unreal-mcp/
├── MCPGameProject/                  ← author's demo project; ignore
│   └── Plugins/UnrealMCP/           ← THE PLUGIN you want
│       └── UnrealMCP.uplugin
├── Python/                          ← MCP bridge (unreal_mcp_server.py)
├── Docs/
└── README.md
```

Clone to a long-term home:

```bash
mkdir -p ~/src
git clone https://github.com/chongdashu/unreal-mcp.git ~/src/chongdashu-unreal-mcp
```

> **UE 5.6 status:** the official README still says "5.5+". A working UE 5.6 Win x64 binary plus required fixes (renaming a global `BufferSize`, removing a comment from the `.uplugin` JSON) are documented in [open issue #27](https://github.com/chongdashu/unreal-mcp/issues/27) with PR #45 in flight. If main fails to compile on 5.6, that issue is the first stop.

## 2. Install the Python MCP bridge (one-time, shared)

Per [Python/README.md](https://github.com/chongdashu/unreal-mcp/blob/main/Python/README.md):

```bash
cd ~/src/chongdashu-unreal-mcp/Python
uv venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
uv pip install -e .
```

The bridge entrypoint is `unreal_mcp_server.py`. It logs to `unreal_mcp.log` in the same directory. The Python README specifies "Unreal Engine editor is loaded and running" must be active before the server connects — UE editor must be up first when you run Claude Code later.

The README does not document a fixed TCP port; if you need to diagnose connection issues, watch the editor's Output Log filtered for `UnrealMCP` to see what port the C++ plugin opens.

## 3. Install gdep (codebase-map MCP, one-time, shared)

Per [gdep README](https://github.com/pirua-game/gdep):

```bash
npm install -g gdep-mcp
gdep --version
```

`gdep-mcp` picks up the project from the **current working directory** when launched — you don't pass a `--project` argument. It exposes 30 tools including `analyze_ue5_gas`, `analyze_ue5_behavior_tree`, `analyze_ue5_state_tree`, `analyze_ue5_animation`, `analyze_ue5_blueprint_mapping`, `find_method_callers`, `find_call_path`, `find_class_hierarchy`, and a wiki interface for project notes.

## 4. Create your UE 5.6 C++ project

Epic Games Launcher → **Unreal Engine 5.6** → Launch.

- **Games → Blank** template
- **C++** (not Blueprint)
- Starter Content: off
- Project name: `<your-experiment>`
- Location: somewhere outside `~/src/chongdashu-unreal-mcp`

Once the editor finishes its initial compile and opens, **close it** so we can drop the plugin in.

## 5. Drop the plugin into your project

```bash
cd <your-experiment>
mkdir -p Plugins

# Option A — symlink (recommended; one source clone updates every project):
ln -s ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP

# Option B — copy (use on Windows without dev-mode/admin, or to pin a commit):
cp -R ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP
```

Verify the `.uplugin` resolves at the expected path:

```bash
ls Plugins/UnrealMCP/UnrealMCP.uplugin
```

## 6. Open in Rider and build

Rider opens `.uproject` files directly — **no need to generate `.sln`/Xcode files manually** ([JetBrains docs](https://www.jetbrains.com/help/rider/Unreal_Engine__UnrealLink_RiderLink.html)). Drag-drop the `.uproject` into Rider, or use **Open** from the welcome screen.

First open will prompt to install **RiderLink** — accept it (see [RIDER.md](./RIDER.md) §2 for details on Engine vs Game install).

Rider auto-builds the editor target. First build is 10–30 min cold; subsequent iterations use Live Coding.

When the editor launches:

1. **Edit → Plugins**, search "UnrealMCP" → enable it
2. Also enable **Python Editor Script Plugin** (required for the bridge's reflection calls)
3. Restart the editor when prompted

## 7. Register MCP servers with Claude Code

From your project root:

```bash
cd <your-experiment>

# chongdashu unreal-mcp — points at the shared Python bridge from step 2
claude mcp add unreal-engine -- \
  uv --directory ~/src/chongdashu-unreal-mcp/Python run unreal_mcp_server.py

# gdep — picks up cwd as the project
claude mcp add gdep -- gdep-mcp
```

Verify both registered:

```bash
claude mcp list
```

If you prefer editing the config file directly, the equivalent JSON ([chongdashu's documented snippet](https://github.com/chongdashu/unreal-mcp#mcp-configuration) and [gdep's documented snippet](https://github.com/pirua-game/gdep)):

```json
{
  "mcpServers": {
    "unreal-engine": {
      "command": "uv",
      "args": [
        "--directory",
        "/Users/<you>/src/chongdashu-unreal-mcp/Python",
        "run",
        "unreal_mcp_server.py"
      ]
    },
    "gdep": {
      "command": "gdep-mcp",
      "env": { "PYTHONUTF8": "1" }
    }
  }
}
```

Config file locations per chongdashu's README:
- **Claude Desktop:** `~/.config/claude-desktop/mcp.json` (macOS/Linux) or `%USERPROFILE%\.config\claude-desktop\mcp.json` (Windows)
- **Cursor:** `.cursor/mcp.json` (per-project)
- **Claude Code:** `claude mcp add` writes to `~/.claude/` config

## 8. CLAUDE.md and .claudeignore

Create `CLAUDE.md` at the project root:

```markdown
# Project: <your-experiment>

## Stack
- Unreal Engine 5.6, C++ project (BPs, Materials, Niagara, UMG all fair game)
- Rider; Live Coding active
- Experimental visual-art / interactive installation

## Input pipeline
- OSC on UDP 7000 from external sensors
- (add as needed: NDI labels, Spout/Syphon names, MIDI device, serial port)

## Conventions
- Gameplay framework in C++; BP for designer-facing tweaks (params, UI, sequencer)
- Generate C++ classes/components first; I'll create BP children as needed
- For BP edits, use MCP node tools — don't try to read .uasset bytes
- Renderable params on a `UArtParamSet` UDataAsset, hot-reloadable
- Don't suggest WebSockets/REST for sensor input — OSC is wired up

## Don't touch
- Engine/ source
- Plugins/UnrealMCP/ (symlink to a shared clone)
```

Create `.claudeignore`:

```
Binaries/
Intermediate/
Saved/
DerivedDataCache/
*.uasset
*.umap
.vs/
.idea/
```

## 9. First validation prompt

**Open the editor first** so MCP can connect. Wait for shader compile to settle. Then in a separate terminal:

```bash
cd <your-experiment>
claude --model opus
```

Prompt:

> Read CLAUDE.md. Then have `gdep` give me a project overview. Then scaffold a `UOscInputBusSubsystem` (UGameInstanceSubsystem) that listens on UDP 7000, parses OSC bundles addressed `/sensor/<id>/<channel>` with a single float arg, and broadcasts a multicast delegate `FOnSensorValue(int32 SensorId, FName Channel, float Value)`. C++ only for the subsystem. Then use the `unreal-engine` MCP to spawn an `AActor` in the level with a `UTextRenderComponent` that prints the most recent value. Compile and confirm the test actor exists in the level.

Healthy stack:
- Claude calls `gdep` tools first ✓
- Generated C++ compiles (Live Coding green) ✓
- Test actor is spawned via `unreal-engine` MCP tools, not by editing `.umap` text ✓

## 10. Model strategy

- **Opus 4.6** — new sketches, scaffolding, refactors touching >3 files
- **Sonnet 4.6** — shader/material expression tweaks, Niagara wiring, single-function debug, BP node edits via MCP

`/model` switches mid-session.

## 11. Optional automation

Once you've done steps 1–3 once, save this at `~/.bin/ue-init` and `chmod +x` it:

```bash
#!/usr/bin/env bash
# usage: ue-init <existing-c++-project-dir>
set -euo pipefail
proj="$1"
[ -d "$proj" ] || { echo "not a dir: $proj"; exit 1; }
[ -n "$(ls "$proj"/*.uproject 2>/dev/null)" ] || { echo "no .uproject in $proj"; exit 1; }
mkdir -p "$proj/Plugins"
ln -s ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP "$proj/Plugins/UnrealMCP"
[ -f ~/templates/CLAUDE.md ]    && cp ~/templates/CLAUDE.md    "$proj/CLAUDE.md"
[ -f ~/templates/.claudeignore ] && cp ~/templates/.claudeignore "$proj/.claudeignore"
cd "$proj"
claude mcp add unreal-engine -- uv --directory ~/src/chongdashu-unreal-mcp/Python run unreal_mcp_server.py
claude mcp add gdep -- gdep-mcp
echo "OK — open the .uproject in Rider, enable UnrealMCP plugin, build, then 'claude --model opus'."
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `claude mcp list` doesn't show servers | Run `claude mcp add` from the project dir. Check the config file paths in step 7. |
| MCP tool calls fail / empty tool list | UE editor not running. Start editor first, then `claude`. |
| `uv run` fails with import error | Re-run step 2 (`uv venv`, activate, `uv pip install -e .`). Check Python is 3.12+. |
| Output Log doesn't show UnrealMCP activity | Plugin not enabled (Edit → Plugins → UnrealMCP) or Python Editor Script Plugin disabled. |
| Symlink not allowed (Windows) | Enable Developer Mode in Windows settings, or run shell as admin. Or use copy (Option B). |
| Build error: "module 'UnrealMCP' not found" | Plugin folder layout wrong. Confirm `Plugins/UnrealMCP/UnrealMCP.uplugin` exists. |
| Editor won't compile plugin on UE 5.6 | Apply fixes from [issue #27](https://github.com/chongdashu/unreal-mcp/issues/27): rename the `BufferSize` global, remove the comment from `.uplugin` JSON. Or pull a fork that includes PR #45. |
| MCP connection drops during a long task | Reduce ops per call; restart the bridge with the editor running. |
| Update the plugin/bridge across all projects | `cd ~/src/chongdashu-unreal-mcp && git pull`. With Option A symlink this is enough; with Option B copy you re-copy per project. |

## Where to look when something's off

- **chongdashu issues:** https://github.com/chongdashu/unreal-mcp/issues (especially #27 for UE 5.6)
- **gdep issues:** https://github.com/pirua-game/gdep/issues
- **Claude Code MCP debug:** `claude --debug`
- **UE Output Log:** filter `UnrealMCP`
- **Bridge log:** `~/src/chongdashu-unreal-mcp/Python/unreal_mcp.log`

## Verified sources

- chongdashu/unreal-mcp [README](https://github.com/chongdashu/unreal-mcp) + [Python/README.md](https://github.com/chongdashu/unreal-mcp/blob/main/Python/README.md) + [issue #27](https://github.com/chongdashu/unreal-mcp/issues/27)
- gdep [README](https://github.com/pirua-game/gdep) — install + JSON config + tool list
- JetBrains Rider [UnrealLink/RiderLink docs](https://www.jetbrains.com/help/rider/Unreal_Engine__UnrealLink_RiderLink.html)
