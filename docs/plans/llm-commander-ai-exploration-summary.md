# LLM Commander AI — Exploration Summary

**Date:** 2026-03-14
**Status:** Brainstorming phase, ready for individual proposals

---

## What We're Building

An LLM-powered strategic layer for Forge's Commander AI that replaces random opponent selection and primitive threat assessment with intelligent multiplayer reasoning. The LLM decides **what** to do (strategy); existing Forge AI decides **how** (tactics).

---

## Full System Map

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    FORGE LLM COMMANDER AI — FULL SYSTEM MAP                ║
╚══════════════════════════════════════════════════════════════════════════════╝

┌──────────────────────────────────────────────────────────────────────────────┐
│                         FORGE GAME ENGINE (existing)                        │
│  Game ─── Player ─── Card ─── PhaseHandler ─── Combat ─── MagicStack      │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ reads game state
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  A. LLM INFRASTRUCTURE LAYER  (forge.ai.llm.*)                            │
│                                                                            │
│  ┌─────────────────────┐  ┌──────────────────────┐  ┌───────────────────┐  │
│  │ A1. LLM Client      │  │ A2. Game State        │  │ A3. Response     │  │
│  │                     │  │     Serializer        │  │     Parser       │  │
│  │ • java.net.http     │  │                      │  │                  │  │
│  │ • Blocking calls    │  │ • Composable methods │  │ • Per-decision   │  │
│  │ • Gson for JSON     │  │ • ~490 tokens/state  │  │   type parsing   │  │
│  │ • ThreadUtil.limit()│  │ • Key permanent      │  │ • Structured     │  │
│  │   for timeouts      │  │   heuristics         │  │   extraction     │  │
│  └─────────┬───────────┘  └──────────┬───────────┘  └────────┬──────────┘  │
│            │                         │                       │             │
│  ┌─────────▼─────────────────────────▼───────────────────────▼──────────┐  │
│  │ A4. Caching & Cost Management                                       │  │
│  │ • Board fingerprint for cache invalidation                          │  │
│  │ • Per-decision-type caches                                          │  │
│  │ • Est. ~$0.005-0.02/game                                            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ A5. Graceful Fallback                                               │  │
│  │ • StrategicAdvisor interface with LLM + RuleBased implementations   │  │
│  │ • Auto-switch on: no API key, timeout, error, parse failure         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ A6. Event Recorder                                                  │  │
│  │ • Observer pattern on game engine, ring buffer of ~20 events        │  │
│  │ • Feeds "recent events" section of serialized state                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ provides StrategicAdvice
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  B. STRATEGIC DECISION LAYER                                               │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ B0. Prompt Assembler                                                │  │
│  │ • system prompt + serialized state + decision question + format     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │B1. Threat  │ │B2. Attack  │ │B3. Removal │ │B4. Board   │              │
│  │ Assessment │ │ Targeting  │ │ Targeting  │ │ Wipe Timing│              │
│  │            │ │            │ │            │ │            │              │
│  │ Replaces:  │ │ Replaces:  │ │ New logic  │ │ New logic  │              │
│  │ evaluate   │ │ choosePrefd│ │            │ │            │              │
│  │ BoardPos() │ │ Defender() │ │            │ │            │              │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘              │
│                                                                            │
│  ┌────────────┐ ┌────────────────────────────────┐                        │
│  │B5. Cmdr    │ │B6. Political Reasoning /       │                        │
│  │ Damage     │ │    Episodic Memory             │                        │
│  │ Strategy   │ │                                │                        │
│  └────────────┘ └────────────────────────────────┘                        │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ returns StrategicAdvice
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  FORGE AI TACTICAL LAYER (existing, modified to consume advice)             │
│  AiController ─── AiAttackController ─── AiBlockController                 │
│  ComputerUtil ─── PlayerControllerAi ─── SpellAbilityAi (150+)             │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  C. INDEPENDENT TRACKS                                                     │
│                                                                            │
│  ┌──────────────────────────┐  ┌──────────────────────────────────────┐    │
│  │ C1. Multiplayer Sim Fix  │  │ C2. Reasoning Display               │    │
│  │ (no LLM dependency)     │  │ (developer + player feedback loop)  │    │
│  └──────────────────────────┘  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| LLM layer location | `forge.ai.llm.*` package in forge-ai | Needs intimate access to Player/Game/Card; not a general-purpose library |
| HTTP client | `java.net.http.HttpClient` | Zero new deps, built into Java 17, no Kotlin baggage (OkHttp pulls Kotlin stdlib) |
| JSON library | Gson (~300KB) | Simple API, small footprint, sufficient for request/response |
| Streaming | No (blocking calls) | Response is ~50-100 tokens; TTFT dominates latency; streaming saves ~100ms, not worth complexity |
| Batching | No (one decision per call) | Keep it simple; cost is low enough (~$0.02/game) |
| Serializer | Purpose-built (not reusing GameState.java) | GameState designed for save/restore, not LLM reasoning |
| Serializer style | Composable methods, concise natural language | Card names > full text (UrzaGPT research); ~490 tokens per state |
| Fallback | Built into architecture from day one | StrategicAdvisor interface with LLM + RuleBased implementations |
| Caching | Board fingerprint per decision type | Skip LLM call when board hasn't meaningfully changed |

---

## Exploration Depth Map

```
████ = deeply explored    ▓▓▓ = discussed     ░░░ = mentioned only

A. LLM INFRASTRUCTURE
─────────────────────
A1. LLM Client               ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░
    Done: tech choices (java.net.http, Gson, blocking, ThreadUtil.limit())
    TODO: API key storage, retry policy, request/response models

A2. Game State Serializer     ████████████████████░
    Done: composable architecture, key permanent heuristics,
          token budget (~490), tiered opponent serialization,
          sample output, all Player/Card/Game methods mapped
    TODO: exact Java implementation

A3. Response Parser           ▓▓▓░░░░░░░░░░░░░░░░░
    Done: per-decision-type parsing concept
    TODO: format spec, regex vs structured, error recovery

A4. Caching & Cost Mgmt      ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░
    Done: fingerprint approach, invalidation triggers, cost estimates
    TODO: cache data structure, TTL, eviction, cost tracking impl

A5. Graceful Fallback         ▓▓▓▓▓▓▓▓░░░░░░░░░░░░
    Done: interface pattern, two implementations, switch triggers
    TODO: partial degradation, sticky vs per-call fallback

A6. Event Recorder            ▓▓▓▓░░░░░░░░░░░░░░░░
    Done: event types, ring buffer concept
    TODO: engine hook mechanism, natural language formatting

B. STRATEGIC DECISIONS
──────────────────────
B0. Prompt Assembler          ▓▓▓▓▓▓░░░░░░░░░░░░░░
    Done: structure (system + state + question), section mapping table
    TODO: actual prompt text, output formats, prompt versioning

B1. Threat Assessment         ▓▓▓▓▓▓▓░░░░░░░░░░░░░
    Done: replaces evaluateBoardPosition(), read current impl
    TODO: ThreatRanking model, integration wiring

B2. Attack Targeting          ▓▓▓▓▓▓░░░░░░░░░░░░░░
    Done: read choosePreferredDefenderPlayer() (random!), replaces it
    Note: upstream added removeUnpayableAttackers() post-declaration step
    TODO: AttackRecommendation model, mapping to declareAttackers()

B3. Removal Targeting         ░░░░░░░░░░░░░░░░░░░░
    TODO: everything — where targeting happens, prompt design

B4. Board Wipe Timing         ░░░░░░░░░░░░░░░░░░░░
    TODO: everything — where decisions happen, factors, prompt

B5. Commander Damage Strategy ░░░░░░░░░░░░░░░░░░░░
    TODO: everything — voltron detection, strategy implications

B6. Political Reasoning       ░░░░░░░░░░░░░░░░░░░░
    TODO: everything — memory model, aggro management

C. INDEPENDENT TRACKS
─────────────────────
C1. Multiplayer Sim Fix       ░░░░░░░░░░░░░░░░░░░░
    TODO: scope of fix, what's broken in GameSimulator

C2. Reasoning Display         ░░░░░░░░░░░░░░░░░░░░
    TODO: dev tool vs player-facing, UI integration
```

---

## OpenSpec Organization

Architecture specs (the "what" and contracts):
- `openspec/specs/llm-infrastructure/spec.md` — **CREATED** (detailed)
- `openspec/specs/strategic-decisions/spec.md` — not yet created
- `openspec/specs/observability/spec.md` — not yet created

Individual feature proposals (the "how" and tasks):
- Each of the 13 workstreams becomes its own change in `openspec/changes/`
- Specs define contracts; proposals implement against those contracts

---

## Dependency Graph

```
  MUST BUILD FIRST (infrastructure gates everything):
  A1 + A2 + A3 + A5 → one proposal: "llm-infrastructure-core"
  A6 → can be separate: "event-recorder"
  A4 → can be woven into core or separate: "caching-cost-mgmt"

  THEN (strategic features, roughly in order):
  B1 (Threat Assessment) → foundation for B2-B4, B6
  B2 (Attack Targeting) → highest impact, fixes #1 complaint
  B5 (Commander Damage) → can parallel with B2
  B3 (Removal Targeting) → after B1
  B4 (Board Wipe Timing) → after B1
  B6 (Political Reasoning) → last, most complex, depends on A6

  INDEPENDENT (can be done anytime):
  C1 (Multiplayer Sim Fix) → pure Java, no LLM, benefits all AI
  C2 (Reasoning Display) → after at least B1 exists to display

  CROSS-CUTTING (baked into design, not separate proposals):
  A5 (Fallback) → part of StrategicAdvisor interface in core
```

---

## Upstream Sync (2026-03-14, +704 commits)

Merged 704 upstream commits from Card-Forge/forge. Key findings:

**Our integration points are stable:**
- `choosePreferredDefenderPlayer()` — unchanged (still random above 8 life)
- `evaluateBoardPosition()` — unchanged (still `life*3 + lands*2`)
- `GameState.java`, `GameStateEvaluator`, `ThreadUtil`, `AiBlockController` — unchanged

**API changes to account for:**
- `Card.keywordsToText()` now **private** — use `hasKeyword()` or `getKeywords()` instead
- `ComputerUtil.handlePlayingSpellAbility` — `Game` param removed (gets it from spell ability)
- `GlobalAttackRestrictions` — `Integer` (nullable) replaces `int` (-1 sentinel)
- `CardType.CoreType` enum replaces `String` in type analysis utilities
- Keyword system refactored to `IKeywordsChange`/`ICardTraitChanges` interfaces
- Logging migrated from minlog → **tinylog** (`org.tinylog.Logger`)

**New pattern in attack flow:**
- `AiController` now calls `removeUnpayableAttackers(Combat)` after `declareAttackers` — validates AI can pay attack costs (Propaganda, etc.). Our B2 attack targeting integration must account for this post-declaration validation step.

**Impact on GameSimulator (C1):**
- Minor API updates (`handlePlayingSpellAbility` signature, `GameCopier` type methods) but no structural changes. Multiplayer fix scope unchanged.

---

## Next Steps

1. **Brainstorm strategic-decisions spec** — define the decision flow, threat model, how B1-B6 interact, prompt templates
2. **Brainstorm observability spec** — reasoning display scope and architecture
3. **Create individual proposals** starting with `llm-infrastructure-core` (A1+A2+A3+A5)
4. **Explore C1** — read GameSimulator.java to scope the multiplayer fix
