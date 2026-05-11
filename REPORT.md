# Research Report: Using Claude to Develop Unreal Engine Projects

**Session ID:** research-20260511-unreal-ai-claude
**Date:** 2026-05-11
**Status:** Complete
**Scope:** MCP servers, UE plugins, practitioner workflows, top OSS repos, user reviews, onboarding recommendations
**Recency weight:** 2024–2026 sources prioritized; most are Q1–Q2 2026

---

## Executive Summary

The Claude × Unreal Engine ecosystem went from "experimental" in early 2025 to a real production pattern by mid‑2026. The stack that works **right now** is: **Claude Code (Opus 4.6 or Sonnet 4.6) in a terminal next to Rider/VS Code**, with an **MCP server** plugin running inside the live UE editor so the model can read scene state, edit Blueprints, spawn actors, compile, and screenshot — instead of trying to read `.uasset` binaries blind. Pick **one** MCP server (chongdashu, flopperam, kvick, Natfii's UnrealClaude, NAJEMWEHBE's UnrealClaudeMCP, or a paid Fab plugin like CLAUDIUS / Rekall / NWIRO) and a **CLAUDE.md** at the project root, and you're 80% of the way to the workflow people are actually shipping with.

The hard problems are real and well‑documented: Blueprints are binary so naive file‑reading hallucinates dependencies; UE source is huge so context windows blow up; UE 5.6/5.7 has API drift the models lag on; and editor MCP connections drop under heavy batch ops. The community has converged on patterns that mitigate each — covered in §3 and §6.

For this user (greenfield project, on macOS Darwin, no existing UE repo in cwd yet), the recommended starting stack is in **§6 Onboarding Recommendations**.

---

## 1. MCP Servers for Unreal Engine

MCP is the dominant integration pattern. Claude Code has the deepest MCP support of any client because Anthropic authored the spec ([StraySpark, Apr 2026](https://www.strayspark.studio/blog/claude-code-vs-cursor-vs-windsurf-unreal-engine-2026)). Below are the credible servers as of May 2026 — note stars and last‑activity figures shift weekly.

### Tier 1 — Mature open source

| Project | Stars | UE Support | License | What you get |
|---|---|---|---|---|
| [chongdashu/unreal-mcp](https://github.com/chongdashu/unreal-mcp) | **1.9k** | 5.5+ (5.6 build exists) | MIT | Actor mgmt, Blueprint dev, BP node graph, editor control. Marked **EXPERIMENTAL**, "breaking changes may occur." Comes with a UE 5.5 starter project. Python bridge. |
| [flopperam/unreal-engine-mcp](https://github.com/flopperam/unreal-engine-mcp) | **903** | 5.5 / 5.6 / 5.7 | MIT | Two impls: free local MCP + hosted "Flop MCP" (50+ tools, 9 domains: Blueprint authoring, materials, VFX, animation, landscape, AI/BT, cinematics, PCG, data assets). Hosted needs API key + subscription. |
| [Natfii/UnrealClaude](https://github.com/Natfii/UnrealClaude) | **587** | 5.7 (Windows/Linux/macOS-AS) | MIT | In‑editor chat panel that shells out to `claude` CLI; 20+ MCP tools; "Dynamic UE 5.7 Context System" surfaces API docs on demand; session persistence; CLAUDE.md aware. |
| [prajwalshettydev/UnrealGenAISupport](https://github.com/prajwalshettydev/UnrealGenAISupport) | **584** | 5.4–5.7+ | MIT | Both runtime LLM access (Claude Opus/Sonnet, Gemini, GPT‑5, Grok, Deepseek, Ollama) AND MCP server for editor. Also 3D gen (Meshy/Tripo/Hunyuan3D/Rodin), TTS (ElevenLabs/Inworld). |
| [kvick-games/UnrealMCP](https://github.com/kvick-games/UnrealMCP) | **574** | 5.5 only | MIT | Scene manipulation, object create/delete, transforms, Python exec. Roadmap (mostly unbuilt): Niagara, Metasound, PCG, Landscape, Modeling. Used Blender MCP as reference. |
| [softdaddy-o/soft-ue-cli](https://github.com/softdaddy-o/soft-ue-cli) | **150** | 5.7 | MIT (assumed) | Python CLI + UE plugin; 60+ ops (actors, Blueprints, materials, PIE, screenshots, UE Insights, StateTree, widgets). Notable: **works with packaged Dev/DebugGame builds**, not just editor. v1.28.0 May 2026. |
| [runreal/unreal-mcp](https://github.com/runreal/unreal-mcp) | (smaller) | UE 5+ | MIT | Uses UE's built‑in **Python Remote Execution** — minimal plugin footprint. Good choice if you don't want a custom C++ plugin. |
| [GenOrca/unreal-mcp](https://github.com/GenOrca/unreal-mcp) | **93** | 5.6+ | Apache‑2.0 | 68 tools across 9 categories (incl. Behavior Tree construction, UMG widget edits). v1.4.0 May 9 2026 — actively maintained. |
| [ColtonWilley/ue-llm-toolkit](https://github.com/ColtonWilley/ue-llm-toolkit) | **74** | 5.7 | MIT | HTTP REST API (localhost:3000) → 37 tools, 200+ ops. Specifically built to **let LLMs read binary Blueprints / Animation Graphs / Control Rig / BlendSpace**. Author also runs a frame‑grabber for image analysis. Positioned as a debugging tool, not autogen. ([dev.to writeup](https://dev.to/colton_willey_ef3d8727ae9/how-i-taught-claude-to-see-inside-unreal-engine-23a9), Mar 2 2026) |
| [NAJEMWEHBE/UnrealClaudeMCP](https://github.com/NAJEMWEHBE/UnrealClaudeMCP) | 2 (new) | 5.7.4 tested | MIT | 64 native C++ handlers + 4 bridge tools = 68 total. Includes arbitrary `unreal.*` Python exec, asset registry, Blueprint/widget edits, materials, sequences, viewport screenshots, console, audio, camera. ~50ms round‑trip. v0.9.1 May 8 2026. |
| [pirua-game/gdep](https://github.com/pirua-game/gdep) | **43** | UE5 (any) | Apache‑2.0 | **Not** an editor MCP. It's a static dependency analyzer that ships an MCP server with 30 tools so AI can ask "what does my CombatManager touch?" in <0.5s instead of reading files alphabetically. Killer companion to the editor MCPs above. v0.2.10 Apr 12 2026. |
| [runeape-sats/unreal-mcp](https://github.com/runeape-sats/unreal-mcp) | (early) | 5.3 + Remote Control API | (check repo) | Pure Python, no C++ plugin needed — uses UE's Remote Control API. Easiest setup, but UE 5.3 era and limited tool surface. |

### Install pattern (chongdashu / kvick / NAJEMWEHBE style)

```bash
# 1. Drop plugin into your project
cd YourProject/Plugins && git clone https://github.com/chongdashu/unreal-mcp.git
# 2. Regenerate VS/Rider project files (right-click .uproject → Generate)
# 3. Build Development Editor target
# 4. Open project, Edit → Plugins → enable "UnrealMCP" and "Python Editor Script Plugin"
# 5. Register the server with Claude Code:
claude mcp add unreal-engine -- /path/to/unreal-mcp-server   # or:
# claude_desktop_config.json:
# {"mcpServers":{"unrealMCP":{"command":"uv","args":["--directory","<path>","run","unreal_mcp_server.py"]}}}
```

Source: [StraySpark's 15‑minute setup guide](https://www.strayspark.studio/blog/setting-up-first-mcp-server-claude-unreal-15-minutes) (Feb 14 2026), [chongdashu README](https://github.com/chongdashu/unreal-mcp).

### Install pattern (flopperam hosted)

```bash
claude mcp add -H "Authorization: Bearer YOUR_API_KEY" --transport http flopperam-unreal https://agent.flopperam.com/mcp
```

44 tools free, 17 paid (Blueprint editing, data assets, code exec, API docs) — from $15/mo after a 1‑week trial.

---

## 2. UE Plugins (Marketplace / Fab / Indie) Bridging LLMs

These are **commercial** or **larger turnkey** options, mostly shipped via Fab.com or independent stores. Most use MCP under the hood now.

### Editor‑side AI assistants

- **[Ludus AI](https://ludusengine.com/)** — Full toolkit: C++ gen, Blueprint copilot, scene gen, "Unreal expert" Q&A. UE 5.4–5.7. Project‑aware (parses your BPs/code). Connects from Rider/VS/VS Code. Mixed reviews — strong as a time‑saver, weak as an autonomous builder ([SmythOS review](https://smythos.com/ai-trends/ludus-ai-in-unreal-engine/)).
- **[Ultimate Engine CoPilot](https://www.fab.com/listings/8d776721-5da3-44ce-b7ef-be17a023be59)** (formerly Ultimate Blueprint Generator) — Voice‑controllable, runs concurrent agents inside the editor, generates BPs / animations / materials / Niagara / sequences / UMG / PCG / behavior trees.
- **[Aura: AI Agent for Unreal Editor](https://forums.unrealengine.com/t/aura-ai-agent-for-unreal-editor/2689209)** — Launched Jan 2026; public beta with **Claude Code integration** added later. UE 5.3–5.7. Plan Mode + subagents. Credit‑based pricing, ~$40 free trial credit. Mixed reviews — promising, still buggy.
- **[NWIRO AI Integration Kit](https://www.fab.com/listings/278f28b3-69f5-44e8-bd4b-fff5e833e24f)** — Generates Blueprints, Materials, C++ classes, PCG, Niagara, UMG, lighting "from natural language prompts, powered by your own Claude or GPT AI through the **Claude CLI & Codex CLI**." Most direct Claude Code integration on Fab.
- **[CLAUDIUS Code](https://forums.unrealengine.com/t/claudius-code-claudius-ai-powered-editor-automation-framework/2689084)** — 230+ commands across 19 categories. v3.1 Apr 2026. Has 8 workflow presets (fps, platformer, animation_studio, etc.). Reduces context bloat with per‑project command curation. UE 5.4–5.7, Win64. Sold on Fab.
- **[Rekall](https://forums.unrealengine.com/t/marius-store-rekall-ai-coding-assistant-plugin-claude-cursor-windsurf/2719545)** — Native C++ MCP server, **no Python bridge** (runs on 127.0.0.1:30080). **480+ tools / 47 areas** — claims the largest surface. UE 5.7 Win64. Integrates Tripo AI for 3D mesh gen.
- **[LocoAI](https://forums.unrealengine.com/t/locodev-loco-ai-ai-assistant-that-builds-blueprints-inside-ue5/2719549)** — Free up to 500 prompts/mo; six pre‑built gameplay templates. Prompts tuned for **Claude Sonnet/Opus** (BYO API key). $10/mo premium tier Q2 2026.
- **[NoGlyph Blueprint AI](https://forums.unrealengine.com/t/noglyph-blueprint-ai-blueprint-ai-assistant-and-generator/2701067)** — Pure Blueprint‑focused assistant.
- **[Widget Semantic Bridge](https://forums.unrealengine.com/t/ai-agent-umg-widget-blueprint-generator-widget-semantic-bridge/2715585)** — Specialized UMG widget generation.

### LLM connectivity plugins (use Claude API at runtime or dev‑time)

- **[Muddy Terrain Games – Gen AI](https://forums.unrealengine.com/t/muddy-terrain-games-gen-ai-chatgpt-gemini-claude-grok-4-llm-api-chat-vision-npcs-openai/2605790)** ([Fab](https://www.fab.com/listings/68e7f092-1fea-4e6d-8d31-c6b96b06a02e)) — Production-ready unified interface for ChatGPT (incl. o3/o4), Claude Sonnet/Opus 4, Gemini 2.5, Grok 4, Deepseek R1.
- **[AI Chat Plus](https://www.fab.com/listings/0e49d138-10e1-452e-ba07-9a4bea578ace)** — OpenAI, Azure OpenAI, Claude, Gemini, Ollama, local llama.cpp, DeepSeek.
- **[LLM-Connect](https://github.com/yigit-altun-rootrootq/LLM-Connect)** — Blueprint‑ready: dropdown to switch models (GPT/Gemini/Claude/Ollama). Free OSS.
- **[Universal Offline LLM Plugin](https://www.fab.com/listings/c5981158-7add-4977-9e08-440831058e5d)** — Local-only models.
- **[ClaudeAI Plugin (claudeaiplugin.com)](https://claudeaiplugin.com/)** — Standalone Claude‑specific UE5 assistant.
- **[Epic community Claude chat tutorial](https://dev.epicgames.com/community/learning/tutorials/RmwD/claude-anthropic-ai-integration-in-unreal-engine-chat-tutorial)** — Official Epic Dev Community tutorial: integrating Anthropic API for in‑game chat.

---

## 3. Practitioner Tips and Proven Workflows

Recurring themes from real practitioner write‑ups (2025–2026):

### IDE pairing

**JetBrains Rider + Claude Code in terminal** is the most‑recommended combo for C++ work ([SharkPillow, Aug 2025](https://sharkpillow.com/post/rider-claude/) — Kyle Smyth). Rider handles `.uproject`, UE build system, reflection macros (UPROPERTY/UFUNCTION), and Live Coding. Claude Code handles scaffolding, refactors, pattern matching. Quote: *"recompiling C++ while the editor is running in debug mode has been a game‑changer."*

If you don't want Rider, **VS Code + Cursor + clangd** is the other dominant setup ([NVIDIA blog, Mar 2026](https://developer.nvidia.com/blog/reliable-ai-coding-for-unreal-engine-improving-accuracy-and-reducing-token-costs/) — Paul Logan).

### CLAUDE.md content that actually works

From [StraySpark game‑dev setup guide](https://www.strayspark.studio/blog/claude-code-game-developers-setup-guide) (Mar 23 2026):

- Project structure (maps/characters/environments/UI)
- Naming conventions (`BP_PascalCase`, `M_PascalCase`, `S_` prefix for interfaces)
- Current sprint focus + perf targets
- Tech choices (Nanite, Lumen, Mass, Iris)
- Confirmation rules: *"Always confirm before: deleting actors, removing components, changing level streaming settings"*
- Keep under **150 lines**. Use progressive disclosure — tell Claude **how to find** info, don't dump it.
- Pair with `.claudeignore`: `Binaries/, Intermediate/, Saved/, *.uasset, *.umap`
- Create `.claude/commands/` directory for repeated workflows (`setup-level.md`, `audit-performance.md`)

### Model choice

- **Claude Opus 4.6** — for deep multi‑file refactors and complex architectural work. ~$15–21/M tokens. Best on Terminal‑Bench (59.3%) → strong at navigating filesystem + running builds. ([Kevuru benchmarks, Mar 30 2026](https://kevuru games.com/blog/claude-vs-chatgpt-for-game-development-capabilities-benchmarks-and-data/))
- **Claude Sonnet 4.6** — daily driver, faster + cheaper. Used for typing, smaller refactors, code review.
- **Heuristic from the field**: Opus for architecture and the first 1–2 prompts of any new feature; Sonnet for the grind.
- For Unreal specifically, Claude **"handles pointer logic and memory safety better"** and produces **"40% fewer hallucinations"** around `TObjectPtr` / GC than GPT‑5.4 in Kevuru's tests.

### Reduce token costs

- **AST‑based chunking** (NVIDIA) — feed coherent functions, not arbitrary file slices.
- **Hybrid retrieval** — semantic + lexical (identifiers, error strings).
- Use a **codebase map tool** like [gdep](https://github.com/pirua-game/gdep) so Claude asks the map for dependencies instead of reading 40 files alphabetically — one practitioner reports **40+ msg → 1 query** ([Epic forum post, Mar 29 2026](https://forums.unrealengine.com/t/after-wasting-hours-watching-claude-read-my-ue5-project-file-by-file-i-built-an-open-source-mcp-tool-that-gives-ai-a-map-of-your-game-codebase/2709804)).
- Use [ColtonWilley's ue-llm-toolkit](https://github.com/ColtonWilley/ue-llm-toolkit) **interface layer** — author reports **>50% token reduction** for Blueprint inspection.
- `/compact` mid‑session in Claude Code; `/clear` before pivoting.

### Workflow patterns that work

1. **Spec‑driven**: write the spec in a markdown file first, hand it to Claude. ([dev.to: How I stopped Claude from hallucinating on Day 4](https://dev.to/samhath03/how-i-stopped-claude-code-from-hallucinating-on-day-4-the-spec-driven-workflow-3lim))
2. **Editor must be running** before Claude Code starts — MCP can't connect otherwise.
3. **Zone batches** — modify 100 assets at a time, not 5000. MCP connections drop under heavy load.
4. **Be explicit about which level** ("modify Maps/L_Forest, not the persistent level").
5. **Vague requests fail** — say "volumetric fog density 0.02," not "make it look good."
6. **C++ first, Blueprint second** — AI is markedly stronger at UE C++ than at BPs. Use BP‑edit MCP tools for surgical changes, not whole‑graph generation.

### YouTube + community

- [Claude Wrote My Gameplay Code… Here's the Catch](https://www.youtube.com/watch?v=Q5VEIELTXog) — honest demo
- [I Connected Claude to Unreal Engine (Full MCP Setup)](https://www.youtube.com/watch?v=1evxQpJkXzM)
- [How to Easily Code With AI in Unreal Engine 5](https://www.youtube.com/watch?v=up7FPD6yBlw)
- [Ludus AI YouTube channel](https://www.youtube.com/@LudusAI)
- [Aura: AI Agent for Unreal Editor — Epic Dev Community](https://forums.unrealengine.com/t/aura-ai-agent-for-unreal-editor/2689209)

---

## 4. Top OSS Repos for AI‑Assisted UE Dev

Already enumerated under §1. Beyond MCP servers, notable repos:

### Runtime LLM / NPC plugins (in‑game AI, not dev‑time)

- [tobenot/TobenotLLMGameplay](https://github.com/tobenot/TobenotLLMGameplay) — "Langchain for UE C++"
- [joe-gibbs/local-llms-ue5](https://github.com/joe-gibbs/local-llms-ue5) — Mistral7B + llama.cpp + StyleTTS NPCs in UE 5.3
- [Haqquee/Persona-UnrealEngine](https://github.com/Haqquee/Persona-UnrealEngine) — LLM dialog characters
- [Sovahero/UnrealAiConnector](https://github.com/Sovahero/UnrealAiConnector) — LLM connector
- [Akiya-Research-Institute/LocalLLM-Demo-UE5](https://github.com/Akiya-Research-Institute/LocalLLM-Demo-UE5) — UE 5.4 demo for the Local LLM Plugin

### Dev‑time agentic + helpers

- [ChiR24/Unreal_mcp](https://github.com/ChiR24/Unreal_mcp) — TS + C++, native automation bridge
- [mirno-ehf/ue5-mcp](https://github.com/mirno-ehf/ue5-mcp) — Claude Code → Blueprint edit specifically
- [jl-codes/unreal-5-mcp](https://github.com/jl-codes/unreal-5-mcp) — Cursor/Windsurf/Claude Desktop fork
- [AlexKissiJr/UnrealMCP](https://github.com/AlexKissiJr/UnrealMCP) — kvick fork

### Skills & guides

- [Claude Skills for Unreal Engine C++ Development](https://claudecodeguides.com/claude-skills-for-unreal-engine-c-development/)
- [UE5 Development Skill for Claude Code (mcpmarket)](https://mcpmarket.com/tools/skills/ue5-development-debugging)
- [UE5 project setup skill (lobehub)](https://lobehub.com/skills/antonioinnovateops-unrealengineagent-ue5-project-setup)

### Marketplaces / discovery

- [PulseMCP / mcpservers.org](https://mcpservers.org/) — directories tagging UE MCP servers
- [LobeHub MCP catalog](https://lobehub.com/mcp) — community ratings
- [DeepWiki autogen docs for flopperam](https://deepwiki.com/flopperam/unreal-engine-mcp)
- [mcp.so/server/unreal-mcp/runeape-sats](https://mcp.so/server/unreal-mcp/runeape-sats)

---

## 5. Honest User Reviews and Pitfalls

### What works (consistent praise)

- **C++ macro handling** — Claude reliably uses `UPROPERTY`, `UFUNCTION`, `TObjectPtr`, `UCLASS` correctly. 40% fewer pointer/GC hallucinations than GPT‑5.4 ([Kevuru](https://kevurugames.com/blog/claude-vs-chatgpt-for-game-development-capabilities-benchmarks-and-data/)).
- **Multi‑file refactors** — Claude Code compiled first‑try in 7/10 in StraySpark's tests, beating Cursor and Windsurf ([StraySpark comparison](https://www.strayspark.studio/blog/claude-code-vs-cursor-vs-windsurf-unreal-engine-2026)).
- **Bulk asset ops via MCP** — Claude Code "wins decisively for bulk asset operations, procedural content, and automated content pipeline work."
- **`build.cs` / project files** — better than GPT at fixing CMake / build pipeline weirdness (Terminal‑Bench 59.3% vs 47.6%).
- **Live editor manipulation** — once MCP is wired, "spawn a cube at origin" → cube appears. Real‑time tactile feedback.

### What's broken (consistent complaints)

- **Blueprints are binary** — without an MCP/toolkit that parses `.uasset`, Claude either skips BPs or invents node graphs that don't exist. Solved by [ue‑llm‑toolkit](https://github.com/ColtonWilley/ue-llm-toolkit), [gdep](https://github.com/pirua-game/gdep), or any of the editor MCPs in §1.
- **File‑by‑file reading** — "40+ messages watching Claude attempt to trace class dependencies alphabetically" (Epic forum, Mar 2026). Fix: codebase‑map MCP.
- **Context window blowup** on UE Source trees. Fix: `.claudeignore` Engine/Source, use selective imports.
- **MCP connection drops under load** — modify thousands of assets in zone batches, not one mega‑prompt.
- **UE 5.6 / 5.7 API drift** — models still occasionally suggest deprecated APIs (e.g., MovieScene). Mitigate with Dynamic Context System (UnrealClaude) or a `unreal-version.md` note in your CLAUDE.md.
- **Aura installer bugs** — some users report broken source files across engine versions (forum thread).
- **CLAUDIUS Code v3.x install issues** — `CLAUDE.md` collision and unresponsive support reported April 2026.
- **Hot‑reload occasionally breaks** when Claude regenerates headers; Live Coding handles most of it but full restart sometimes needed.
- **Per‑task token cost** — Claude Code $8–15/day typical, $25–40/day heavy. Higher than Cursor's flat $20–120/mo for similar work, but with deeper MCP capability.

### Honest comparison (StraySpark, Apr 2026)

| | Claude Code | Cursor | Windsurf |
|---|---|---|---|
| MCP depth | Best — native | Late add, less fluent | Port, not native |
| C++ intellisense | Editor‑dependent | Strong (VS Code + clangd) | OK |
| Multi‑file refactor reliability | 7/10 first‑try compile | Misses sites occasionally | Most aggressive, riskiest |
| Cost (5 devs/mo) | $150–300 | $60–120 | Lowest baseline |
| Best for | MCP/editor automation, rotating projects | Daily typing on one project | Solo / budget |

### Pitfall checklist

1. Don't try to feed Claude the whole `Source/` tree — `.claudeignore` it.
2. Don't generate Blueprints "from scratch" in BP — generate C++ and let Claude wire up the BP via MCP node tools.
3. Always start the **editor first**, then `claude` in the project dir.
4. Use **gdep** or **ue‑llm‑toolkit** before any large refactor to give Claude a map.
5. Watch the **MCP connection** — restart server if a multi‑hundred‑asset op stalls.
6. Pin your UE version in CLAUDE.md ("This project is UE 5.6.1 — do not use APIs deprecated in 5.4+").
7. Don't enable MCP write tools on Production branches without confirmation guardrails.

---

## 6. Onboarding Recommendations for This Project

Working directory `/Users/genehan/projects/claudehome-projects/research/unreal-ai-dev` is empty (no UE project yet). On macOS (Darwin 25.4). Recommended **stack to stand up first**:

### Phase 0 — Decide your engine target this week

- **UE 5.6** (stable, broad plugin support) is the safe choice. UE 5.7 has the newest features but several MCP plugins are still catching up. Avoid 5.4 or older unless you have a reason.

### Phase 1 — Minimum viable Claude × UE stack (1 evening)

1. Install Unreal Engine 5.6 via Epic Games Launcher → enable for macOS.
2. Generate a new C++ project (Third Person or Blank).
3. `npm install -g @anthropic-ai/claude-code` (already installed if you're reading this).
4. Pick **one** MCP server. Recommendations in order:
   - **First‑time / lowest friction**: [runreal/unreal-mcp](https://github.com/runreal/unreal-mcp) — uses UE's Python Remote Execution, no C++ plugin to build.
   - **Most surface area free**: [chongdashu/unreal-mcp](https://github.com/chongdashu/unreal-mcp) (1.9k★, EXPERIMENTAL but most popular).
   - **Highest tool count**: [Rekall](https://forums.unrealengine.com/t/marius-store-rekall-ai-coding-assistant-plugin-claude-cursor-windsurf/2719545) (paid Fab, 480+ tools, **no Python sidecar**).
   - **Best in‑editor UX**: [Natfii/UnrealClaude](https://github.com/Natfii/UnrealClaude) — chat panel in the editor, shells out to your existing `claude` auth. macOS‑AS supported.
5. Add a `CLAUDE.md` at the project root (see §3 for content guide).
6. Add a `.claudeignore` excluding `Binaries/ Intermediate/ Saved/ DerivedDataCache/ *.uasset *.umap`.
7. Install [gdep](https://github.com/pirua-game/gdep) as a second MCP — gives Claude a map of your code so it doesn't read files alphabetically.

### Phase 2 — IDE + model setup (same evening)

- On macOS: **VS Code + clangd** or **JetBrains Rider for Unreal (macOS)**. Rider on macOS is now stable enough for daily UE work (per SharkPillow).
- Model: **Claude Opus 4.6** for the first prompt of any new feature, **Sonnet 4.6** otherwise. Set `model=opus` explicitly via `claude --model opus` or in your project's `settings.json`.

### Phase 3 — First real session (next day)

- Start the editor first.
- `cd YourProject && claude --model opus`
- First prompt: "Read CLAUDE.md and gdep me a project overview."
- Second prompt: a small, well‑scoped task — e.g., "Create a `UInteractableComponent` that broadcasts an `OnInteracted` event, with a Blueprint subclass `BP_Door` that opens when interacted. Use MCP to spawn one in the level."
- Watch tool calls. Confirm BP edits land. Iterate.

### Phase 4 — When to add what

- Hitting token walls → add **ue‑llm‑toolkit**'s interface layer (50% reduction on BP inspection).
- Heavy procedural/level work → switch to **flopperam hosted** or **CLAUDIUS Code**.
- In‑editor chat is friction (you keep switching to terminal) → install **Natfii/UnrealClaude**.
- Need runtime LLM in the game itself (NPCs, dialog) → install **Muddy Terrain Gen AI** or **prajwalshettydev/UnrealGenAISupport** *in addition to* the editor MCP — they serve different purposes.

### Concrete first‑week budget

- API: ~$15/day heavy day on Opus → assume $100–200/week trying things.
- Plugins: $0 (start free) or $15/mo (flopperam) or one‑time Fab purchase ($30–80 typical) for CLAUDIUS / Rekall.

### Skills to invest in (vs delegate to AI)

- **Project structure decisions** — AI is bad at deciding what should be a `GameInstanceSubsystem` vs `WorldSubsystem` vs a service. Decide yourself.
- **Replication / GAS architecture** — verify every attribute/ability AI writes; this is the single most hallucination‑prone area.
- **Performance budgets** — AI won't catch a 200µs tick. Profile yourself.
- **Asset pipeline policy** — Nanite/Lumen on/off, LOD strategy, texture budgets. AI follows it, doesn't set it.

---

## Limitations of This Research

- No live access to actual Reddit thread bodies (search returns surfaced summaries, not raw thread content).
- Fab marketplace listings 403 on direct fetch; details come from forum threads and Fab search snippets.
- Several star counts and "last commit" dates were inferred from README/release pages, not direct GitHub API calls — likely accurate to within a week.
- Plugin pricing on Fab fluctuates; verify before purchase.
- No first‑hand testing — all claims are sourced citations, not reproduced experiments.

## Appendix — Primary Sources

**Setup guides & comparisons**
- [StraySpark — MCP setup in 15 minutes (Feb 14 2026)](https://www.strayspark.studio/blog/setting-up-first-mcp-server-claude-unreal-15-minutes)
- [StraySpark — Claude Code for Game Devs setup guide (Mar 23 2026)](https://www.strayspark.studio/blog/claude-code-game-developers-setup-guide)
- [StraySpark — Claude Code vs Cursor vs Windsurf for UE (Apr 17 2026)](https://www.strayspark.studio/blog/claude-code-vs-cursor-vs-windsurf-unreal-engine-2026)
- [SharkPillow — Rider + Claude Code (Aug 20 2025)](https://sharkpillow.com/post/rider-claude/)
- [NVIDIA — Reliable AI Coding for UE (Mar 10 2026)](https://developer.nvidia.com/blog/reliable-ai-coding-for-unreal-engine-improving-accuracy-and-reducing-token-costs/)
- [Kevuru Games — Claude vs ChatGPT for Game Dev benchmarks (Mar 30 2026)](https://kevurugames.com/blog/claude-vs-chatgpt-for-game-development-capabilities-benchmarks-and-data/)

**MCP servers & toolkits**
- [chongdashu/unreal-mcp](https://github.com/chongdashu/unreal-mcp)
- [flopperam/unreal-engine-mcp](https://github.com/flopperam/unreal-engine-mcp) + [flopperam.com/mcp](https://www.flopperam.com/mcp)
- [Natfii/UnrealClaude](https://github.com/Natfii/UnrealClaude)
- [NAJEMWEHBE/UnrealClaudeMCP](https://github.com/NAJEMWEHBE/UnrealClaudeMCP)
- [kvick-games/UnrealMCP](https://github.com/kvick-games/UnrealMCP)
- [runreal/unreal-mcp](https://github.com/runreal/unreal-mcp)
- [GenOrca/unreal-mcp](https://github.com/GenOrca/unreal-mcp)
- [runeape-sats/unreal-mcp](https://github.com/runeape-sats/unreal-mcp)
- [softdaddy-o/soft-ue-cli](https://github.com/softdaddy-o/soft-ue-cli)
- [ColtonWilley/ue-llm-toolkit](https://github.com/ColtonWilley/ue-llm-toolkit) + [dev.to writeup](https://dev.to/colton_willey_ef3d8727ae9/how-i-taught-claude-to-see-inside-unreal-engine-23a9)
- [prajwalshettydev/UnrealGenAISupport](https://github.com/prajwalshettydev/UnrealGenAISupport)
- [pirua-game/gdep](https://github.com/pirua-game/gdep) + [Epic forum announcement](https://forums.unrealengine.com/t/after-wasting-hours-watching-claude-read-my-ue5-project-file-by-file-i-built-an-open-source-mcp-tool-that-gives-ai-a-map-of-your-game-codebase/2709804)
- [ChiR24/Unreal_mcp](https://github.com/ChiR24/Unreal_mcp)
- [mirno-ehf/ue5-mcp](https://github.com/mirno-ehf/ue5-mcp)
- [jl-codes/unreal-5-mcp](https://github.com/jl-codes/unreal-5-mcp)

**Plugins & marketplaces**
- [Ludus AI](https://ludusengine.com/) + [SmythOS review](https://smythos.com/ai-trends/ludus-ai-in-unreal-engine/)
- [CLAUDIUS Code thread](https://forums.unrealengine.com/t/claudius-code-claudius-ai-powered-editor-automation-framework/2689084) + [claudiuscode.com](https://claudiuscode.com/)
- [Rekall thread](https://forums.unrealengine.com/t/marius-store-rekall-ai-coding-assistant-plugin-claude-cursor-windsurf/2719545)
- [LocoAI thread](https://forums.unrealengine.com/t/locodev-loco-ai-ai-assistant-that-builds-blueprints-inside-ue5/2719549)
- [Aura thread](https://forums.unrealengine.com/t/aura-ai-agent-for-unreal-editor/2689209)
- [Ultimate Engine CoPilot thread](https://forums.unrealengine.com/t/ultimate-blueprint-generator-the-ai-co-pilot-for-unreal-engine/2618922)
- [Muddy Terrain Gen AI Fab listing](https://www.fab.com/listings/68e7f092-1fea-4e6d-8d31-c6b96b06a02e)
- [NWIRO Fab listing](https://www.fab.com/listings/278f28b3-69f5-44e8-bd4b-fff5e833e24f)
- [AI Chat Plus Fab listing](https://www.fab.com/listings/0e49d138-10e1-452e-ba07-9a4bea578ace)
- [Epic Dev Community: Claude Anthropic tutorial](https://dev.epicgames.com/community/learning/tutorials/RmwD/claude-anthropic-ai-integration-in-unreal-engine-chat-tutorial)
- [Carlos Li — Claude Unreal forum thread](https://forums.unrealengine.com/t/carlos-li-claude-unreal/2720506)
- [CodeGPT UE5 agent](https://www.codegpt.co/agents/unreal-engine-v5)

**Discovery / catalogs**
- [mcpservers.org](https://mcpservers.org)
- [mcp.so](https://mcp.so)
- [LobeHub MCP](https://lobehub.com/mcp)
- [DeepWiki flopperam autodocs](https://deepwiki.com/flopperam/unreal-engine-mcp)

[PROMISE:RESEARCH_COMPLETE]
