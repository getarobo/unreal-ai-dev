# Setup: New Unreal 5.6 C++ Project with chongdashu/unreal-mcp + Claude Code

Opinionated, fastest-path setup for the stack recommended in [REPORT.md](./REPORT.md) §6, tuned for the profile in this repo's [CLAUDE.md](./CLAUDE.md): C++ project structure (Blueprint/Material/Niagara/UMG usage is fine), Rider, experimental visual-art work with sensor/camera input.

Target time: **~90 minutes** end-to-end, including the first editor build.

## 0. Prerequisites

- Unreal Engine **5.6** installed via Epic Games Launcher
- [JetBrains Rider](https://www.jetbrains.com/rider/) (latest 2026.x) — see [RIDER.md](./RIDER.md) for full IDE setup. Or VS Code + clangd if you prefer.
- `claude` CLI authenticated (`claude --version` works)
- Python 3.10+ on PATH
- `uv` package manager (`brew install uv` on macOS, `pipx install uv`, or `pip install uv`)
- Node 18+ (for the `gdep` codebase-map MCP)
- Git

## 1. Clone chongdashu/unreal-mcp once (outside any UE project)

The repo bundles three things — only two of them are useful to your project:

```
chongdashu-unreal-mcp/
├── MCPGameProject/                   ← author's demo project, IGNORE
│   └── Plugins/UnrealMCP/            ← THE PLUGIN (what you want)
│       └── UnrealMCP.uplugin
├── Python/                           ← THE MCP BRIDGE (what you want)
│   └── unreal_mcp_server.py
├── Docs/                             ← reference
└── README.md
```

Clone the whole repo somewhere you'll keep it long-term — you'll reuse it across every project:

```bash
mkdir -p ~/src
git clone https://github.com/chongdashu/unreal-mcp.git ~/src/chongdashu-unreal-mcp
```

You'll never touch this clone again except `git pull` to update.

## 2. Set up the Python bridge (one-time, shared across all projects)

```bash
cd ~/src/chongdashu-unreal-mcp/Python
uv venv
uv pip install -e .
```

Quick smoke test (will print help and exit — UE doesn't have to be running for this):

```bash
uv run unreal_mcp_server.py --help 2>&1 | head
```

The bridge connects to a TCP socket on **127.0.0.1:55557** opened by the in-editor plugin you'll install in step 4. Useful to know for troubleshooting.

## 3. Install `gdep` (codebase-map MCP, shared across all projects)

```bash
npm install -g gdep-mcp
gdep --version
```

`gdep` scans your C++/BP project in <0.5s and lets Claude ask "what touches `AMyActor`?" instead of reading files alphabetically. Especially valuable for C++ projects.

## 4. Create your UE 5.6 C++ project

From the Epic Games Launcher → **Unreal Engine 5.6** → Launch.

- **Games → Blank** template
- **C++** (not Blueprint)
- Starter Content: off
- Raytracing: off (toggle later)
- Target: Desktop, Maximum Quality
- Project name: `<your-experiment>`
- Location: somewhere outside `~/src/chongdashu-unreal-mcp`

Once it opens and finishes initial compile, **close the editor and Rider**. We'll add the plugin before reopening.

## 5. Drop the plugin into your project

Pick one — either copy (self-contained, version-pinned per project) or symlink (every project picks up `git pull` from the shared clone):

```bash
cd <your-experiment>
mkdir -p Plugins

# Option A — symlink (recommended for experimental work):
ln -s ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP

# Option B — copy (use if you want each project locked to a specific commit):
cp -R ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP
```

Verify the plugin landed correctly:

```bash
ls Plugins/UnrealMCP/UnrealMCP.uplugin
# should print the path, not error
```

Regenerate IDE project files:

- **macOS / Linux:** in Rider, **Tools → Unreal Engine → Refresh Project**. Or from CLI:
  `<UE_INSTALL>/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh -project="$(pwd)/<your-experiment>.uproject" -game -rocket -progress`
- **Windows:** right-click the `.uproject` → "Generate Visual Studio project files".

## 6. Build the project (first time = 10–30 minutes)

Open the `.uproject` in Rider. Rider prompts to build the editor target — confirm. First build is slow; subsequent builds use Live Coding (see [RIDER.md](./RIDER.md) §4).

When the editor opens:

1. **Edit → Plugins**, search "UnrealMCP" → enable it
2. Also enable **Python Editor Script Plugin** and **Editor Scripting Utilities** if not already on
3. Restart editor when prompted
4. Check **Window → Developer Tools → Output Log**, filter for "UnrealMCP" — you should see it listening on `127.0.0.1:55557`

## 7. Register MCP servers with Claude Code

From your project root:

```bash
cd <your-experiment>

# chongdashu unreal-mcp (uses your shared Python bridge from step 2)
claude mcp add unreal-engine -- \
  uv --directory ~/src/chongdashu-unreal-mcp/Python run unreal_mcp_server.py

# gdep (codebase map for this project)
claude mcp add gdep -- gdep-mcp --project "$(pwd)"
```

Verify both registered:

```bash
claude mcp list
```

If you prefer editing config directly (e.g., for Claude Desktop), the equivalent JSON:

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
      "args": ["--project", "/abs/path/to/<your-experiment>"]
    }
  }
}
```

## 8. Drop in CLAUDE.md and .claudeignore

Create `CLAUDE.md` at the project root:

```markdown
# Project: <your-experiment>

## Stack
- Unreal Engine 5.6, C++ project (BPs, Materials, Niagara, UMG all fair game)
- Rider; Live Coding active (see RIDER.md in unreal-ai-dev research repo)
- Experimental visual-art / interactive installation

## Input pipeline
- OSC on UDP 7000 from external sensors
- (add as needed: NDI labels, Spout/Syphon names, MIDI device, serial port)

## Conventions
- Gameplay framework in C++; BP for designer-facing tweaks (params, UI, sequencer)
- Generate C++ classes/components first; I'll create BP children as needed
- For BP edits, use MCP node tools — don't read .uasset bytes
- Renderable params on a `UArtParamSet` UDataAsset, hot-reloadable
- Don't suggest WebSockets/REST for sensor input — OSC is wired up

## Don't touch
- Engine/ source
- Plugins/UnrealMCP/ (it's a symlink to a shared clone; modify the source clone instead)
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

1. **Open the editor first.** MCP can't connect if it isn't running. Wait for shader compile to settle.
2. In a separate terminal:

```bash
cd <your-experiment>
claude --model opus
```

3. Paste this prompt:

> Read CLAUDE.md. Then have `gdep` give me a project overview. Then scaffold a `UOscInputBusSubsystem` (UGameInstanceSubsystem) that listens on UDP 7000, parses OSC bundles addressed `/sensor/<id>/<channel>` with a single float arg, and broadcasts a multicast delegate `FOnSensorValue(int32 SensorId, FName Channel, float Value)`. C++ only for the subsystem. Then use the `unreal-engine` MCP to spawn an `AActor` in the level with a `UTextRenderComponent` that prints the most recent value. Compile and confirm the test actor exists in the level.

If Claude:
- Calls `gdep` tools first ✓
- Generates C++ that compiles first try (or near it) ✓
- Spawns the actor via `unreal-engine` MCP tools (not by editing .umap text) ✓

…your stack is healthy. Otherwise see Troubleshooting below.

## 10. Model strategy

- **Opus 4.6** — new sketch start, new subsystem scaffolding, >3-file refactor
- **Sonnet 4.6** — material/shader tweaks, Niagara wiring, single-function debug, BP node edits via MCP

Switch with `/model` mid-session, or start with `claude --model opus` / `claude --model sonnet`.

## 11. Per-experiment bootstrap (optional)

Once steps 1–3 are done once, your **per-new-project** flow shrinks to steps 4–9. To automate further, drop this in `~/.bin/ue-init`:

```bash
#!/usr/bin/env bash
# usage: ue-init <existing-empty-c++-project-dir>
set -euo pipefail
proj="$1"
[ -d "$proj" ] || { echo "not a dir: $proj"; exit 1; }
[ -f "$proj"/*.uproject ] || { echo "no .uproject in $proj"; exit 1; }
mkdir -p "$proj/Plugins"
ln -s ~/src/chongdashu-unreal-mcp/MCPGameProject/Plugins/UnrealMCP "$proj/Plugins/UnrealMCP"
cp ~/templates/CLAUDE.md "$proj/CLAUDE.md"
cp ~/templates/.claudeignore "$proj/.claudeignore"
cd "$proj"
claude mcp add unreal-engine -- uv --directory ~/src/chongdashu-unreal-mcp/Python run unreal_mcp_server.py
claude mcp add gdep -- gdep-mcp --project "$(pwd)"
echo "OK — open the .uproject in Rider, enable UnrealMCP plugin, build, then 'claude --model opus'"
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `claude mcp list` doesn't show servers | Run `claude mcp add` from the project dir, not parent. Or check `~/.claude/claude_desktop_config.json` for typos. |
| MCP "spawn unreal-engine error" or empty tool list | UE editor not running. Start editor first, then `claude`. |
| `uv run` fails with import error | Re-run step 2: `cd ~/src/chongdashu-unreal-mcp/Python && uv venv && uv pip install -e .`. Confirm Python 3.10+. |
| Output Log doesn't show "UnrealMCP listening" | Plugin not enabled or build failed. Edit → Plugins → enable UnrealMCP, restart editor. If build failed, see Rider's build log. |
| TCP port 55557 in use | Another UE editor instance is still running; kill it. `lsof -i:55557` on macOS/Linux to find the PID. |
| Claude reads files alphabetically | `gdep` not registered. `claude mcp list` → re-add if missing. |
| Symlink broken on Windows | Windows needs admin or developer mode for symlinks. Use `cp -R` (Option B) instead. |
| Build error: "module 'UnrealMCP' not found" | Plugin's `.uplugin` is in wrong place. Should be at `Plugins/UnrealMCP/UnrealMCP.uplugin`. Verify with `ls Plugins/UnrealMCP/`. |
| Editor crashes on enable plugin | UE version mismatch. The README claims 5.5+; if 5.6 misbehaves, check repo issues (esp. issue #27 about UE 5.6). |
| MCP connection drops mid-batch | Too many ops in one call. Re-prompt for fewer; restart MCP server if persistent. |
| Want to update the plugin | `cd ~/src/chongdashu-unreal-mcp && git pull`. With Option A symlink, all projects update; with Option B copy, you re-copy per project. |

## Where to look when something's off

- **chongdashu issues:** https://github.com/chongdashu/unreal-mcp/issues (esp. #27 for UE 5.6)
- **gdep issues:** https://github.com/pirua-game/gdep/issues
- **Claude Code MCP debug:** `claude --debug` shows MCP traffic
- **UE Output Log:** filter for "UnrealMCP" or "MCP" to see plugin-side activity
- **The Python bridge log:** `~/src/chongdashu-unreal-mcp/Python/unreal_mcp.log`
