# Session Handoff — 2026-05-11

Written so you (and the next Claude Code session) can resume on a different machine.

## How to resume on the new machine

```bash
mkdir -p ~/projects && cd ~/projects
git clone git@github-personal:getarobo/unreal-ai-dev.git
cd unreal-ai-dev
claude
```

First prompt to give that session:

> Read HANDOFF.md and CLAUDE.md. We left off mid-decision on whether to stay with chongdashu/unreal-mcp (popular but possibly abandoned) plus apply UE 5.6 patches, or switch SETUP.md to target flopperam/unreal-engine-mcp or GenOrca/unreal-mcp instead. Continue from there.

## What this repo is

A research workspace evaluating how to use Claude (Claude Code + Claude API) to develop Unreal Engine 5 projects. **Not a UE project itself** — there's no `.uproject` here. The recommended stack will be installed in a separate UE project once decisions land.

## What's in the repo

| File | Purpose |
|---|---|
| [REPORT.md](./REPORT.md) | 341-line cited research report — MCP servers, plugins, workflows, top repos, user reviews, onboarding. Source of truth for "why we picked X". |
| [SETUP.md](./SETUP.md) | Step-by-step bootstrap for a new UE 5.6 C++ project with `chongdashu/unreal-mcp` + `gdep`. Every command verified on 2026-05-11. |
| [RIDER.md](./RIDER.md) | JetBrains Rider 2026.x setup for UE 5.6 — install, RiderLink, Live Coding, debugging, pairing with terminal Claude Code. |
| [CLAUDE.md](./CLAUDE.md) | Repo-level context for future Claude sessions: what this workspace is, where the report is, recommended stack, known pitfalls. |
| `.omc/research/research-20260511-unreal-ai-claude/` | Original sciomc research session artifacts (a copy of report.md + state.json). |
| `.omc/state/` | OMC runtime state from this session. Append-only; don't hand-edit. |

## User profile (locked in)

- **Engine target:** Unreal Engine 5.6 (5.5 too constrained, 5.7 still has plugin gaps).
- **Project style:** C++ project structure, but Blueprints / Materials / Niagara / UMG are all in scope. Not pure-C++.
- **IDE:** JetBrains Rider (see RIDER.md). External terminal for `claude`.
- **Project type:** Experimental visual-art / interactive installations.
- **Inputs:** Sensors and cameras via OSC, NDI, Spout/Syphon, MediaPipe, MIDI, serial — handled by separate runtime plugins (not by the MCP).
- **Lots of experimental projects** → values low per-project setup friction.
- **Model strategy:** Opus 4.6 for sketch starts / large refactors, Sonnet 4.6 for grind.

## Decisions made

1. **MCP choice:** Initially picked `chongdashu/unreal-mcp` (1.9k★, most popular, MIT, EXPERIMENTAL flag).
2. **Companion MCP:** `gdep` (codebase-map tool — solves the "Claude reads files alphabetically" problem).
3. **Install pattern:** Clone chongdashu **once outside the project** to `~/src/chongdashu-unreal-mcp`, then symlink (or copy) just `MCPGameProject/Plugins/UnrealMCP` into each per-project `Plugins/`. Python bridge lives once at `~/src/chongdashu-unreal-mcp/Python` and every project's `claude mcp add` points there.
4. **CLAUDE.md template** for new projects: live in SETUP.md §8.
5. **`.claudeignore`** content also in SETUP.md §8.

## Open decision (where we stopped)

**Should we stay on chongdashu or switch?**

A commenter on the chongdashu repo's PR #45 wrote *"I think this project is abandoned."* The main branch hasn't merged any of the open community fixes for UE 5.6/5.7. There are two unmerged PRs:

- [PR #45](https://github.com/chongdashu/unreal-mcp/pull/45) — Jan 12 2026, fixes `ANY_PACKAGE`, `CompressImageArray`, `TArray64` for UE 5.7+
- [PR #51](https://github.com/chongdashu/unreal-mcp/pull/51) — Apr 24 2026, "UE 5.4+ compat (ANY_PACKAGE removed) + fastmcp 3.x compat"

To use chongdashu on UE 5.6 you currently need to:

1. **Easiest (Windows-only):** grab `UnrealMCP_UE5.6_WinX64.zip` from [issue #27](https://github.com/chongdashu/unreal-mcp/issues/27)
2. **Cross-platform:** pull PR #45 or #51 branch
3. **Manual patch:**
   - `ANY_PACKAGE` → `nullptr` in `UnrealMCPBlueprintCommands.cpp`, `UnrealMCPBlueprintNodeCommands.cpp`, `UnrealMCPEditorCommands.cpp`
   - `FImageUtils::CompressImageArray` → `PNGCompressImageArray`
   - `TArray<uint8>` → `TArray64<uint8>` for image/asset buffers
   - Rename global `BufferSize` in `MCPServerRunnable.cpp` (issue #27)
   - Strip the `//` comment from `UnrealMCP.uplugin` JSON (issue #27)

**Alternatives to chongdashu** (both better-maintained for 5.6 right now):
- **[flopperam/unreal-engine-mcp](https://github.com/flopperam/unreal-engine-mcp)** (903★, MIT) — explicitly lists UE 5.5/5.6/5.7 support; free local MCP + optional hosted tier ($15/mo with 17 paid tools). Most mature commercial-ish option.
- **[GenOrca/unreal-mcp](https://github.com/GenOrca/unreal-mcp)** (93★, Apache-2.0) — UE 5.6+, 68 tools incl. Blueprint Graph / UMG / Behavior Tree, v1.4.0 released May 9 2026.

### What to ask the user on resume

> "Stay with chongdashu and apply the UE 5.6 patches, or switch SETUP.md to target flopperam (broader maintained surface, optional paid tier) or GenOrca (smaller / Apache-2.0 / very active)?"

Their answer determines whether SETUP.md gets a "UE 5.6 Patches" section added or gets rewritten against a different repo.

## Concrete state (verifiable)

- Latest commit on `main`: see `git log -1`.
- All MD files are pushed to `getarobo/unreal-ai-dev`.
- `.omc/state/` will have local edits at handoff time; they're committed when this file is.
- Nothing has actually been installed on this machine — no UE 5.6 install, no Rider, no `~/src/chongdashu-unreal-mcp` clone yet. SETUP.md is theory until tried on a real UE project.

## SSH note for the new machine

This repo is pushed via `git@github-personal:getarobo/unreal-ai-dev.git` — that's an SSH config alias on the old machine pointing at `~/.ssh/id_ed25519_personal`. On the new machine you'll either:

1. Replicate the SSH config alias, or
2. Use `git@github.com:getarobo/unreal-ai-dev.git` directly (will work if the new machine's default SSH key has `getarobo` access).

After cloning, check `git remote -v` and adjust if needed.

## Memory file (other side of the handoff)

There's a project-memory file at:
`/Users/genehan/.claude/projects/-Users-genehan-projects-claudehome-projects-research-unreal-ai-dev/memory/`

That directory **does not transfer** with the git repo — it's local to this machine's `~/.claude`. The next Claude Code session on the new machine will read this HANDOFF.md and CLAUDE.md as primary context instead.
