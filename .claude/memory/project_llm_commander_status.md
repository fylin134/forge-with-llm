---
name: llm-commander-ai-status
description: Current status of LLM Commander AI project — what's been explored, what's next, key files to read for context
type: project
---

LLM Commander AI project is in brainstorming/exploration phase (using /opsx:explore). No code written yet.

**Key context files (in-repo, read these first):**
- `docs/plans/llm-commander-ai-exploration-summary.md` — full system map, decisions made, exploration depth, next steps
- `openspec/specs/llm-infrastructure/spec.md` — detailed spec for infrastructure layer (A1-A6)
- `openspec/config.yaml` — project context, tech stack, conventions
- `docs/forge-ai-arch.md` — existing AI architecture reference

**What's done:**
- Infrastructure layer (A1-A6) deeply explored and spec'd
- 13 workstreams identified and mapped with dependency graph
- Technology decisions finalized (java.net.http, Gson, tinylog, blocking calls)
- Upstream sync analyzed (+704 commits, integration points stable)

**What's next:**
1. Brainstorm strategic-decisions spec (B0-B6) — decision flow, prompt templates, how components interact
2. Brainstorm observability spec (C2)
3. Create individual OpenSpec change proposals starting with `llm-infrastructure-core`
4. Explore C1 (multiplayer sim fix — read GameSimulator.java)

**Why:** Building an LLM-powered strategic layer for Commander AI (1 human vs 3 AI). LLM decides WHAT (strategy), existing Forge AI decides HOW (tactics).

**How to apply:** Read the exploration summary first, then the infra spec. Resume with /opsx:explore to continue brainstorming the strategic-decisions layer.
