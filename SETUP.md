# Setup: New Unreal 5.6 C++ Project with chongdashu/unreal-mcp + Claude Code

Opinionated, fastest-path setup for the stack recommended in [REPORT.md](./REPORT.md) §6, tuned for the profile in this repo's [CLAUDE.md](./CLAUDE.md): C++ project structure with Blueprint/Material/Niagara/UMG usage, Rider, experimental visual-art work, sensor/camera input.

Target time: **~90 minutes** end-to-end, including the first build.

## 0. Prerequisites

- Unreal Engine **5.6** installed via Epic Games Launcher
- Rider for Unreal Engine (latest), or VS Code + clangd
- `claude` CLI authenticated (`claude --version` works)
- Node 18+ (for `gdep`)
- Python 3.10+ on PATH
- `uv` package manager (`brew install uv` on macOS, or `pip install uv`)
- Git

## 1. Create the UE 5.6 C++ project

From the Epic Games Launcher → **Unreal Engine 5.6** → Launch.

- **Games → Blank** template
- **C++** (not Blueprint)
- Starter Content: off (lighter project, you'll add what you need)
- Raytracing: off (toggle later per-project)
- Target: Desktop, Maximum Quality
- Project name: `<your-experiment>`
- Location: somewhere outside this `unreal-ai-dev` repo

Once it opens and finishes initial compile, **close the editor and Rider**. We'll add the plugin before reopening.

## 2. Install chongdashu/unreal-mcp

```bash
cd <your-experiment>
mkdir -p Plugins
git clone https://github.com/chongdashu/unreal-mcp.git Plugins/UnrealMCP
```

The plugin ships its `.uplugin` at `Plugins/UnrealMCP/MCPGameProject/Plugins/UnrealMCP/UnrealMCP.uplugin`. Either:

**Option A — symlink into the standard Plugins location** (recommended):

```bash
rm -rf Plugins/UnrealMCP-tmp && mv Plugins/UnrealMCP Plugins/UnrealMCP-tmp
ln -s Plugins/UnrealMCP-tmp/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP
```

**Option B — copy the inner plugin folder**:

```bash
cp -R Plugins/UnrealMCP/MCPGameProject/Plugins/UnrealMCP Plugins/UnrealMCP-plugin
rm -rf Plugins/UnrealMCP && mv Plugins/UnrealMCP-plugin Plugins/UnrealMCP
```

Regenerate IDE project files:

- **macOS / Linux**: `<UE_INSTALL>/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh -project="$(pwd)/<your-experiment>.uproject" -game -rocket -progress`
- **Windows**: right-click the `.uproject` → "Generate Visual Studio project files" (or use Rider's "Refresh Unreal Project" action)

If the chongdashu repo has a 5.6 community build branch (see issue #27), check it out first:

```bash
cd Plugins/UnrealMCP-tmp && git checkout <ue5.6-branch-if-exists> && cd -
```

## 3. Build the project

Open the `.uproject` in Rider → it will prompt to build the editor target. Confirm. First build takes 10–30 minutes depending on machine.

When the editor opens:

1. **Edit → Plugins**, search "UnrealMCP" → enable it
2. Also enable `PythonScriptPlugin` and `Editor Scripting Utilities` if not already on
3. Restart editor when prompted

## 4. Wire up the Python bridge

In your project root (next to `.uproject`):

```bash
cd Plugins/UnrealMCP-tmp/Python
uv sync   # or: pip install -e .
```

Verify the bridge starts:

```bash
uv run unreal_mcp_server.py --help
```

You should see the FastMCP server help text. Stop it with Ctrl-C.

## 5. Install gdep (codebase map MCP)

```bash
npm install -g gdep-mcp
gdep --version
```

`gdep` scans your C++/BP project in <0.5s and lets Claude ask "what touches `AMyActor`?" instead of reading files one by one. Critical for C++ projects.

## 6. Register both MCPs with Claude Code

```bash
cd <your-experiment>

# chongdashu unreal-mcp
claude mcp add unreal-engine -- \
  uv --directory "$(pwd)/Plugins/UnrealMCP-tmp/Python" run unreal_mcp_server.py

# gdep
claude mcp add gdep -- gdep-mcp --project "$(pwd)"
```

Verify both are registered:

```bash
claude mcp list
```

## 7. Drop in CLAUDE.md and .claudeignore

Create `CLAUDE.md` at the project root:

```markdown
# Project: <your-experiment>

## Stack
- Unreal Engine 5.6, C++ project (BPs, Materials, Niagara, UMG all fair game)
- Rider on $(uname); Live Coding active
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
- Plugins/ third-party plugin sources
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

## 8. First validation prompt

1. **Open the editor first** (MCP needs it running).
2. In a separate terminal:

```bash
cd <your-experiment>
claude --model opus
```

3. Paste this prompt:

> Read CLAUDE.md. Then have `gdep` give me a project structure overview. Then scaffold a `UOscInputBusSubsystem` (UGameInstanceSubsystem) that listens on UDP 7000, parses OSC bundles addressed `/sensor/<id>/<channel>` with a single float arg, and broadcasts a multicast delegate `FOnSensorValue(int32 SensorId, FName Channel, float Value)`. C++ only for the subsystem. Then use the unreal-engine MCP to spawn a test `AActor` in the level with a `UTextRenderComponent` that prints the most recent value. Compile and confirm the BP test actor exists.

If Claude:
- Calls `gdep` tools first ✓
- Generates C++ that compiles first try ✓
- Spawns the actor via `unreal-engine` MCP tools (not by editing .umap text) ✓

…your stack is healthy. If any step fails see Troubleshooting.

## 9. Model strategy

- **Opus 4.6** — new sketch start, new subsystem scaffolding, >3-file refactor
- **Sonnet 4.6** — material/shader tweaks, Niagara wiring, single-function debug, BP node edits via MCP

Switch with `/model` mid-session, or start with `claude --model opus` / `claude --model sonnet`.

## 10. Per-experiment template (optional)

For frictionless new-project bootstrap, after step 8 works copy these into a `~/.ue-ai-template/` directory:

```
~/.ue-ai-template/
├── CLAUDE.md
├── .claudeignore
└── post-init.sh   # runs steps 2,4,5,6 against $1 = new project path
```

Then `~/.ue-ai-template/post-init.sh <new-project-path>` after each `Games → C++` template creation.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `claude mcp list` doesn't show servers | Run `claude mcp add` from the project dir, not parent. Check `claude_desktop_config.json` for typos. |
| MCP "spawn unreal-engine error" | UE editor not running. Start editor first, then `claude`. |
| `uv run` fails with import error | `cd Plugins/UnrealMCP-tmp/Python && uv sync` again. Make sure Python 3.10+. |
| Claude reads files alphabetically anyway | gdep not registered or not picked up. `claude mcp list` → if missing, re-add. |
| Build error: "no module named …" in UnrealMCP | Plugin's `.uplugin` is in wrong folder. Should be at `Plugins/UnrealMCP/UnrealMCP.uplugin` (not nested). |
| Editor crashes on enable plugin | Wrong engine version. UE 5.6 needs the 5.6 branch of chongdashu if main targets 5.5. Check repo issues. |
| MCP connection drops mid-batch | Modifying too much at once. Re-prompt for fewer assets per call; restart MCP server if persistent. |

## Where to look when something's off

- **chongdashu issues**: https://github.com/chongdashu/unreal-mcp/issues
- **gdep issues**: https://github.com/pirua-game/gdep/issues
- **Claude Code MCP debug**: `claude --debug` shows MCP traffic
- **UE output log**: filter for "MCP" to see plugin-side activity
