⏺ Forge Commander AI + LLM Integration - Session Summary

## Project Goal

Build or improve a 3 AI + 1 human player Commander game simulator in Forge, leveraging LLMs to address multiplayer AI limitations.

---
## Key Findings: Current State of Forge

### ✅ What Works Well

- Excellent Commander format support: Deck validation, color identity, singleton rules, partner commanders all fully implemented
- Multiplayer infrastructure: Supports up to 8 players, game setup works
- Command-line simulation: Can run headless tests java -jar forge.jar sim -d deck1 deck2 deck3 deck4 -f commander
- Rules engine: Commander tracking, commander damage tracking, command zone tax
- Card coverage: 99.8% of cards implemented (30,331+ cards)

### ❌ Critical Gaps in Multiplayer Commander AI

1. Random opponent selection - Above 8 life, AI picks targets randomly (AiAttackController.java:195-230)
2. Primitive threat assessment - Only considers life total, ignores board state/card advantage (ComputerUtil.java:2852)
3. Broken simulation engine - Hardcoded for 2 players, can't look ahead in multiplayer (GameSimulator.java:226, 262)
4. No politics/diplomacy - Zero "who's winning" awareness, no temporary alliances
5. No Commander-specific strategy - Doesn't track commander damage toward 21, no voltron awareness
6. Multiple TODO comments - Code explicitly acknowledges these limitations

Bottom line: Forge plays 4-player games as "1v1 but with random opponent selection"

---
## AI Architecture Deep Dive

### Decision Flow

```
Priority → PlayerControllerAi.chooseSpellAbilityToPlay()
         → AiController.chooseSpellAbilityToPlay() (line 1359)
         → Evaluates lands, spells, abilities
         → Returns List<SpellAbility> to play
```

### Key Decision Points

| Decision   | File                    | Method                        | Line |
|------------|-------------------------|-------------------------------|------|
| Main loop  | AiController.java       | chooseSpellAbilityToPlay()    | 1359 |
| Spell eval | AiController.java       | canPlaySa()                   | 899  |
| Mulligan   | PlayerControllerAi.java | mulliganKeepHand()            | 779  |
| Attacks    | AiAttackController.java | declareAttackers()            | 804  |
| Blocks     | AiBlockController.java  | assignBlockersForCombat()     | 997  |
| Targeting  | PlayerControllerAi.java | chooseSingleEntityForEffect() | 352  |

### Architecture Pattern

- Centralized controller with distributed evaluators
- 150+ SpellAbilityAi classes for card-specific logic
- AiPlayDecision enum (26 states) represents decisions
- Perfect information about board, hidden information about hands/libraries

---
## LLM Integration Opportunities

### No Existing LLM Integrations

- Searched entire codebase - zero LLM/GPT/OpenAI/Anthropic integrations
- Only ML is forge-lda module: Latent Dirichlet Allocation for deck generation (traditional 2003-era ML)
- No HTTP clients in AI module
- You would be the first to integrate LLMs into Forge

### Why LLMs for Commander

- Natural language reasoning about complex board states
- Political/social decision-making (who's winning, who to attack)
- Multi-factor decisions without explicit programming
- Can explain reasoning (debuggable)
- Adaptable to new cards without hardcoding

---
## Scope Options (Ranked by Complexity)

### Option 1: Strategic Advisor (1-2 weeks, ~$0.02-0.05/game)

- LLM provides high-level hints, existing AI executes
- Hook: AiController.chooseSpellAbilityToPlay()
- Impact: Better strategic coherence, "who's winning" awareness

### Option 2: Threat Assessment + Combat ⭐ (3-4 weeks, ~$0.05-0.10/game)

- LLM handles threat assessment and combat decisions
- Hooks: evaluateBoardPosition(), choosePreferredDefenderPlayer(), declareAttackers()
- Impact: Solves #1 pain point (random targeting), politics-aware combat
- Recommended starting point

### Option 3: Decision Overrider (4-6 weeks, ~$0.15-0.30/game)

- LLM evaluates every major decision, overrides existing AI when confident
- Hook: canPlaySa() - intercepts all spell evaluations
- Impact: Comprehensive improvement, but more expensive

### Option 4: Full LLM AI (2-3 months, ~$0.50-1.00/game)

- LLM makes ALL decisions with custom action parser
- Replace entire PlayerControllerAi implementation
- Impact: Maximum flexibility but complex to build

### Option 5: Hybrid - "LLM Brain, Rule-Based Hands" ⭐⭐ (6-8 weeks, ~$0.10-0.20/game)

- LLM decides WHAT to do (strategy), existing AI decides HOW (execution)
- Clean separation of concerns
- Impact: Best of both worlds - smart + fast
- Recommended end goal

---
## Recommended Path Forward

### Phase 1: Proof of Concept (2 weeks)

- Implement Option 2 - LLM Threat Assessment + Combat
- Focus on fixing random opponent selection
- Key files to modify:
  - forge-ai/src/main/java/forge/ai/ComputerUtil.java (line 2852)
  - forge-ai/src/main/java/forge/ai/AiAttackController.java (line 195)
- Test with command-line simulation
- Measure: Does AI attack the right player?

### Phase 2: Expand Scope (4 weeks)

- Add strategic advisor concepts
- Integrate removal spell targeting
- Add board wipe timing decisions
- Measure: Quality of plays in complex scenarios

### Phase 3: Hybrid System (4 weeks)

- Refactor into Option 5 architecture
- LLM provides strategic intent, existing AI executes
- Build clean API boundary between systems

### Phase 4: Polish (2-3 weeks)

- Prompt engineering optimization
- Game state caching for similar positions
- Error handling and fallbacks
- (Optional) UI to show LLM reasoning

---
## Technical Requirements

### Your Setup

- ✅ Working directory: /Users/franklin/Projects/forge/docs
- ✅ Repository already cloned
- ✅ LLM experience: Mid-level
- ✅ API preference: Claude API
- ✅ Latency tolerance: 10s per decision (high - perfect for Commander)

### Dependencies to Add

```xml
<!-- Add to forge-ai/pom.xml -->
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
</dependency>
```

### Environment Setup

1. IntelliJ IDEA Community Edition
2. Java 17+
3. API key: ANTHROPIC_API_KEY environment variable

---
## Key File Paths Reference

### Documentation

- /Users/franklin/Projects/forge/docs/AI.md - AI system overview, simulation syntax
- /Users/franklin/Projects/forge/docs/Development/IntelliJ-setup/IntelliJ-setup.md - Dev environment setup
- /Users/franklin/Projects/forge/docs/Development/ownership.md - AI subsystem owners (Hanmac, Agetian)

### AI Source Code

- /Users/franklin/Projects/forge/forge-ai/src/main/java/forge/ai/
  - AiController.java - Main AI brain (1,650 lines)
  - AiAttackController.java - Combat decisions (1,770 lines)
  - AiBlockController.java - Blocking decisions (1,378 lines)
  - ComputerUtil.java - Utility functions including threat assessment
  - PlayerControllerAi.java - Interface to game engine
  - ability/*.java - 150+ card-specific AI evaluators

### Game Engine

- /Users/franklin/Projects/forge/forge-game/src/main/java/forge/game/
  - player/Player.java - Player state (line 188-191: commander tracking)
  - GameType.java - Format definitions
  - GameRules.java - Rules engine

### Deck Validation

- /Users/franklin/Projects/forge/forge-core/src/main/java/forge/deck/DeckFormat.java - Commander validation (lines 68-361)

---
## Testing Strategy

### DevMode for Manual Testing

Create test scenarios (Game Settings → Preferences → Developer Mode):
```
# test-threat-assessment.txt
HumanLife=30
AI1Life=30
AI1CardsInPlay=3x Mountain
AI2Life=20
AI2CardsInPlay=Ancient Tomb; Sol Ring; Rhystic Study; Consecrated Sphinx
AI3Life=40
AI3CardsInPlay=2x Plains
ActivePhase=Main1

# Expected: AI should recognize AI2 as biggest threat despite middle life total
```

### Automated Testing

```bash
# Run 100 games to measure win rates
java -jar forge.jar sim -d deck1 deck2 deck3 deck4 -f commander -n 100 -q

# Compare LLM AI vs traditional AI
# Track: win rate, game length, decisions per turn
```

---
## Open Questions for Next Session

1. Prompt design: What format works best for game state serialization?
2. Caching strategy: How to recognize similar game states?
3. Error handling: What happens when LLM returns invalid/ambiguous responses?
4. Blending weights: When to trust LLM vs existing AI?
5. Performance: Can we batch multiple decisions in one LLM call?

---
## Community Context

- Discord: Active community, AI owners (Hanmac, Agetian) available
- GitHub: https://github.com/Card-Forge/forge
- Ownership: AI subsystem has "understaffed" areas (Limited, Simulation, DeckHints)
- Reception: Docs explicitly say "If you want to train a model for the AI, please do. We would love to see something like that implemented in Forge."
- Your contribution: Would be novel - first LLM integration in Forge

---
## Cost Estimates

| Scope Option       | Calls/Game | Cost/Game  | Cost/100 Games |
|--------------------|------------|------------|----------------|
| Strategic Advisor  | 10-20      | $0.02-0.05 | $2-5           |
| Threat + Combat    | 20-40      | $0.05-0.10 | $5-10          |
| Decision Overrider | 50-100     | $0.15-0.30 | $15-30         |
| Full LLM AI        | 200-400    | $0.50-1.00 | $50-100        |
| Hybrid             | 20-50      | $0.10-0.20 | $10-20         |

Based on Claude Haiku pricing (~$0.25 per 1M input tokens, ~2000 tokens per decision)

---
## Next Concrete Steps

1. Set up dev environment - IntelliJ + Java 17 + fork repo
2. Run baseline test - sim -f commander to see current AI behavior
3. Create LLM service wrapper - HTTP client + Claude API integration
4. Implement threat assessment hook - Replace evaluateBoardPosition() logic
5. A/B test - Compare LLM vs traditional AI in 4-player games

---
## Success Metrics

- AI attacks the most threatening player (not random)
- AI recognizes commander damage threats
- AI makes sensible political decisions
- Games feel more like playing against humans
- Win rate distribution is balanced (not always same player winning)

---
Ready to pick up: You now have a complete map of Forge's AI architecture, identified integration points, and a clear path from proof-of-concept to production hybrid system. The codebase is well-organized, the community is welcoming, and the problem space is fascinating. Good luck building the future of Forge Commander AI! 🎯
