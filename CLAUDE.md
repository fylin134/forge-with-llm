# CLAUDE.md — Forge with LLM

This file provides guidance for AI assistants (Claude and others) working in this repository.

## What Is This Project?

**Forge** is an open-source, production-grade Java implementation of the Magic: The Gathering rules engine (GPL-3.0). It has been in active development since 2007 and supports ~27,000 unique MTG cards, multiple UIs (Desktop, Mobile, Android, iOS), and a sophisticated rule-based AI.

This fork (`forge-with-llm`) adds research and implementation for integrating a Claude LLM as a strategic advisor for multiplayer Commander games, addressing a key gap in the existing AI.

**Key docs:**
- Architecture overview: [`docs/forge-game-arch.md`](docs/forge-game-arch.md), [`docs/forge-ai-arch.md`](docs/forge-ai-arch.md), [`docs/forge-core-arch.md`](docs/forge-core-arch.md)
- AI system: [`docs/AI.md`](docs/AI.md)
- LLM integration research: [`docs/plans/2026-01-02-agentic-llm-commander-ai-research.md`](docs/plans/2026-01-02-agentic-llm-commander-ai-research.md), [`docs/plans/llm-commander-ai-exploration-summary.md`](docs/plans/llm-commander-ai-exploration-summary.md)
- Development setup: [`docs/Development/DevMode.md`](docs/Development/DevMode.md)

---

## Build & Run

**Requirements:** Java 17+, Maven

```bash
# Full build
mvn clean install

# Run desktop GUI
java -jar forge-gui-desktop/target/forge-gui-desktop-*.jar

# Run headless AI vs AI simulation
java -jar forge-gui-desktop/target/forge-gui-desktop-*.jar sim -d deck1.dck deck2.dck -n 10

# Commander format simulation
java -jar forge-gui-desktop/target/forge-gui-desktop-*.jar sim -d d1.dck d2.dck d3.dck d4.dck -f commander -n 5
```

Simulation flags: `-n` (games), `-m` (best-of), `-f` (format), `-t` (tournament type: Bracket/RoundRobin/Swiss), `-p` (players), `-q` (quiet).

---

## Module Structure

Forge is a Maven multi-module project with strict layering. **Never add dependencies that violate layer boundaries.**

| Module | Purpose | Key constraint |
|--------|---------|----------------|
| `forge-core` | Data layer: CardDb, CardRules, Deck | No game logic, no AI, no UI |
| `forge-game` | Rules engine: Game, Card, Player, Stack | No UI, no AI |
| `forge-ai` | Decision layer: AiController, SpellAbilityAi | No UI, no rule enforcement |
| `forge-gui-desktop` / `forge-gui-mobile` | Presentation | Defers all logic downward |

---

## Architecture Quick Reference

### forge-core
- `StaticData` — singleton providing global access to CardDb, editions, formats
- `CardDb` — multi-index card database (~27k cards)
- `CardRules` — immutable card rules representation
- `PaperCard` — flyweight physical card instance
- `Deck` — complete deck with sections (Main, Sideboard, Commander, etc.)

### forge-game
- `Game` — central orchestrator (stack, phases, triggers, replacements, combat)
- `Card` — individual card instance (~8,300 lines; MTG cards are complex)
- `Player` — player state (~4,100 lines)
- `SpellAbility` — base for all spells/abilities; holds Cost, Targets, ApiType, Effects
- `ApiType` — enum of 200+ effect types (DealDamage, Draw, Destroy, ChangeZone, etc.)
- `AbilityFactory` — parses card script files into SpellAbility objects
- `TriggerHandler` / `ReplacementHandler` / `StaticEffects` — continuous/triggered/replacement effects
- Card scripting: text files in `res/cardsfolder/` using `API$`, `Cost$`, `ValidTgts$` syntax

### forge-ai
- `PlayerControllerAi` — AI's interface to the game engine (100+ decision methods)
- `AiController` — core AI "brain" with memory, combat prediction, simulation
- `SpellAbilityAi` — template base class for per-effect AI; subclasses in `ability/`
- `SpellApiToAi` — registry mapping `ApiType` → AI implementation
- `AiAttackController` / `AiBlockController` — combat decisions
- `ComputerUtil*` — 200+ static utility methods (card eval, combat calc, mana, costs)
- `AiPlayDecision` — 26-state enum for AI decisions (WillPlay, WaitForCombat, CantPlayAi, etc.)

---

## LLM Integration (This Fork)

### Motivation
The existing Forge AI has a critical gap in multiplayer Commander:
- Randomly selects attack targets when all opponents are above 8 life
- No awareness of who is "winning" or threat levels
- No political reasoning (deals, threat assessment across 4 players)
- Multiplayer simulation in `GameSimulator.java` is hardcoded for 2 players

### Proposed Architecture
LLM decides **WHAT** to do; Forge AI handles **HOW** to execute it.

```
Forge Game Engine
      │ game state
      ▼
LlmStrategicAdvisor  ← NEW (forge-ai module)
  • Threat assessment
  • Combat target selection
  • Removal target prioritization
  • Political reasoning
      │ strategic intent
      ▼
Existing AI Tactical Layer (AiController, AiAttackController, etc.)
```

### Implementation Phases
1. **Phase 1 (POC):** Threat assessment — single LLM call per priority pass ranking opponents
2. **Phase 2:** Combat decisions — LLM picks attack target with `simulate_combat` tool
3. **Phase 3:** Removal targeting — LLM selects removal targets with `evaluate_removal_targets` tool
4. **Phase 4:** Full strategic layer — spell prioritization, board wipe timing, political memory

### Game State Serialization
Concise format (~490 tokens/state): life totals, card counts, commander damage, board summary (total power, key permanents), recent events (last 3-4 turns), hand contents. Use card names, not full card text.

### Cost Reference (Claude Haiku)
~$0.02/game (Phase 1) → ~$0.15–0.25/game (full integration)

---

## Development Guidelines

### Conventions
- AI ability classes: `{Effect}Ai.java` (e.g., `CounterAi`, `DamageDealAi`)
- Controllers: `{Area}Controller.java`
- Use **tinylog** (`org.tinylog.Logger`) for all logging — the project migrated from minlog
- Run CheckStyle (`checkstyle.xml`) before submitting changes

### Key Gotchas (Upstream Divergence)
This fork diverges from upstream Card-Forge/forge. Known breaking changes:
- `Card.keywordsToText()` is now **private** — use `hasKeyword()` / `getKeywords()`
- `ComputerUtil.handlePlayingSpellAbility` no longer takes a `Game` parameter
- `GlobalAttackRestrictions` uses `Integer` (nullable) instead of `int` (-1 sentinel)
- `CardType.CoreType` enum replaces `String` in type analysis
- `AiController` calls `removeUnpayableAttackers(Combat)` after attack declaration

### Testing
- Framework: TestNG
- Headless simulation (`sim` command) is the primary tool for AI regression testing
- Developer mode (in-game): Settings → Gameplay Options → Developer Mode
  - "View Zone", "Generate Mana", "Setup Game State" from file

### Game State File Format (for DevMode testing)
```
HumanLife=20
AILife=20
HumanCardsInPlay=Forest; Forest; Llanowar Elves
AICardsInPlay=Mountain; Mountain; Goblin Guide
HumanCardsInHand=Counterspell; Island
AICardsInHand=Lightning Bolt
ActivePlayer=Human
ActivePhase=Main1
# Comments use # prefix; sets via CardName|SETCODE
```

---

## Key Files for LLM Integration Work

| File | Purpose |
|------|---------|
| `forge-ai/src/main/java/forge/ai/AiController.java` | Core AI brain — entry point for strategic changes |
| `forge-ai/src/main/java/forge/ai/AiAttackController.java` | Attack decisions — primary target for Commander fixes |
| `forge-ai/src/main/java/forge/ai/PlayerControllerAi.java` | AI interface to game engine |
| `forge-ai/src/main/java/forge/ai/ComputerUtil.java` | Utility methods including board evaluation |
| `forge-game/src/main/java/forge/game/Game.java` | Game state — source for LLM context |
| `forge-game/src/main/java/forge/game/player/Player.java` | Player state (life, board, hand) |
| `docs/plans/` | Research notes and implementation plans |

---

## Resources

- Upstream repo: https://github.com/Card-Forge/forge
- Card scripting API: [`docs/Card-scripting-API/Card-scripting-API.md`](docs/Card-scripting-API/Card-scripting-API.md)
- User guide: [`docs/User-Guide.md`](docs/User-Guide.md)
- FAQ: [`docs/Frequently-Asked-Questions.md`](docs/Frequently-Asked-Questions.md)
