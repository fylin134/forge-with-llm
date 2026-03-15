# LLM Infrastructure Layer — Specification

## Overview

The LLM infrastructure layer provides the foundational services that all strategic AI features depend on. It lives inside `forge-ai` as the `forge.ai.llm` package, providing a clean boundary without introducing a new Maven module.

This layer handles: calling the Claude API, serializing game state into concise prompts, parsing LLM responses into structured advice, caching results, managing cost, and falling back gracefully when the LLM is unavailable.

---

## System Architecture

```
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
│  └─────────┬───────────┘  └──────────┬───────────┘  └────────┬──────────┘  │
│            │                         │                       │             │
│  ┌─────────▼─────────────────────────▼───────────────────────▼──────────┐  │
│  │ A4. Caching & Cost Management                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ A5. Graceful Fallback (StrategicAdvisor interface)                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ A6. Event Recorder                                                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ provides StrategicAdvice
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  B. STRATEGIC DECISION LAYER  (consumes infrastructure)                    │
│  B1. Threat Assessment  B2. Attack Targeting  B3. Removal Targeting        │
│  B4. Board Wipe Timing  B5. Cmdr Damage       B6. Political Reasoning     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Specifications

### A1. LLM Client

**Purpose:** Java-to-Claude API bridge. Sends prompts, receives completions.

**Technology Decisions:**
| Decision | Choice | Rationale |
|----------|--------|-----------|
| HTTP client | `java.net.http.HttpClient` | Zero new deps, built into Java 17, async-capable, no Kotlin baggage (OkHttp pulls Kotlin stdlib) |
| JSON library | Gson (~300KB) | Simple API, small footprint, sufficient for our request/response needs |
| Call pattern | Blocking | Response is ~50-100 tokens; TTFT dominates latency; streaming saves ~100ms, not worth complexity |
| Timeout | `ThreadUtil.limit()` | Existing Forge utility, handles timeout + cancellation |

**Responsibilities:**
- Construct API request (system prompt + user message)
- Send HTTP POST to Claude API
- Handle auth (API key from configuration)
- Timeout after configurable duration (default ~10s)
- Retry on transient errors (429, 500, 503)
- Return raw response text to caller

**Open Questions:**
- API key storage pattern: environment variable vs properties file vs ForgePreferences?
- Retry policy: exponential backoff? Max retries?
- Request/response model classes: hand-rolled POJOs or generated?

---

### A2. Game State Serializer

**Purpose:** Convert Forge's `Game`, `Player`, and `Card` objects into concise natural language prompts optimized for LLM reasoning.

**Key Design Principle:** Concise > verbose. Research shows card names outperform full card text (UrzaGPT finding). Target ~490 tokens per serialized state.

**Architecture:** Composable methods, not one monolithic serialize().

```java
public class GameStateSerializer {
    String serializeHeader(Game game, Player self);
    String serializeSelfState(Player self, boolean includeHand);
    String serializeOpponent(Player opponent, Player self);
    String serializeAllOpponents(Game game, Player self);
    String serializeRecentEvents(GameEventLog log, int turnDepth);

    // Convenience: picks sections based on decision type
    String serialize(Game game, Player self, DecisionType type);
}
```

**Decision-type → sections mapping:**

| Decision Type | Header | Self | Self Hand | Opponents | Events |
|---------------|--------|------|-----------|-----------|--------|
| Threat Assessment | yes | board only | no | yes | yes |
| Attack Targeting | yes | yes | no | yes | yes |
| Removal Targeting | yes | yes | no | yes (boards) | no |
| Board Wipe Timing | yes | yes | **yes** | yes | no |
| Commander Strategy | yes | yes | no | yes (cmdr focus) | yes |
| Political Reasoning | yes | yes | no | yes | **yes (deep)** |

**Opponent serialization — tiered approach:**

Always include:
- Life total, cards in hand (count), commander (name, zone, cast count)
- Commander damage dealt to self
- Creature summary: count, total power, total toughness

Include by name (key permanents):
- Non-creature enchantments and artifacts (most matter strategically)
- Planeswalkers (name + loyalty)
- Equipment attached to commander
- Creatures with power >= 4 or evasion keywords
- Commander (always)

Skip:
- Basic lands, generic mana rocks, vanilla tokens, tapped creatures without keywords

**Key permanent heuristic:**
```
isKeyPermanent(Card c):
  c.isPlaneswalker()                              → true
  c.isEnchantment() && !c.isCreature()            → true
  c.isArtifact() && !c.isCreature()               → true
  c.isCreature() && c.getNetPower() >= 4           → true
  c.isCreature() && hasEvasion(c)                  → true
  c.isCommander()                                  → true
  c.hasKeyword("Hexproof") || ("Indestructible")  → true
  otherwise → summary stats only
```

**Token budget estimate:**

| Section | Est. Tokens |
|---------|-------------|
| Context header | ~30 |
| Self state (with hand) | ~120 |
| Opponent × 3 | ~240 |
| Decision context | ~40 |
| Recent events | ~60 |
| **Total** | **~490** |

**Sample output:**
```
GAME STATE — Turn 7, Your Main Phase 1
Format: Commander (4 players)
You: Player 3 (Prossh, Skyraider of Kher)

YOUR STATE:
  Life: 38 | Hand: 7 cards | Commander: Prossh (command zone, cast 0x)
  Mana available: 2R 1B 1G 3 generic
  Board: Prossh (6/6 commander), 6 Kobold tokens (0/1),
         Phyrexian Altar, Skullclamp
  Hand: Terminate, Cultivate, Beast Within, Diabolic Intent,
        Blood Artist, Forest, Swamp

OPPONENTS:
  P1 — Atraxa (32 life, 3 cards in hand)
    Commander: Atraxa, Praetors' Voice (battlefield, cast 1x)
    Cmdr damage to you: 6
    Board: 4 creatures (8 total power), Sol Ring, Rhystic Study,
           Consecrated Sphinx

  P2 — Krenko (18 life, 1 card in hand)
    Commander: Krenko, Mob Boss (battlefield, cast 1x)
    Cmdr damage to you: 0
    Board: Krenko (3/3), 12 Goblin tokens (1/1, 12 power total),
           Goblin Bombardment

  P4 — Oloro (45 life, 6 cards in hand)
    Commander: Oloro, Ageless Ascetic (command zone, cast 0x)
    Cmdr damage to you: 0
    Board: 2 creatures (3 total power), Propaganda, Ghostly Prison

RECENT EVENTS:
  Turn 6: P2 activated Krenko, created 8 Goblin tokens
  Turn 6: P1 drew 4 cards from Consecrated Sphinx trigger
  Turn 5: You cast Phyrexian Altar
  Turn 5: P4 cast Ghostly Prison
```

**Forge API methods used (verified in codebase):**
- `Player.getLife()`, `Player.getCardsIn(ZoneType)`, `Player.getCommanderDamage(Card)`
- `Player.getOpponents()`, `Player.getCreaturesInPlay()`, `Player.getLandsInPlay()`
- `Card.getName()`, `Card.getNetPower()`, `Card.getNetToughness()`, `Card.isCommander()`
- `Card.isCreature()`, `Card.isEnchantment()`, `Card.isArtifact()`, `Card.isPlaneswalker()`
- `Card.hasKeyword(String)`, `Card.getController()`
- `Game.getPhaseHandler().getTurn()`, `.getPhase()`, `.getPlayerTurn()`

**Upstream API changes (synced 2026-03-14, +704 commits):**
- `Card.keywordsToText()` is now **private** — use `Card.hasKeyword(String)` for individual checks or `Card.getKeywords()` to iterate
- `ComputerUtil.handlePlayingSpellAbility` no longer takes a `Game` parameter — game is obtained from `sa.getHostCard().getGame()`
- `GlobalAttackRestrictions` uses `Integer` (nullable) instead of `int` (-1 sentinel) for attack limits
- `CardType.CoreType` enum replaces `String` in `getMostProminentCardType` and related methods
- Keyword system now uses `IKeywordsChange` and `ICardTraitChanges` interfaces
- `Player.addKeyword(String)` convenience method removed — use the full keyword API
- Logging migrated from minlog to **tinylog** (`org.tinylog.Logger`)

---

### A3. Response Parser

**Purpose:** Extract structured advice from LLM text responses.

**Design:** Per-decision-type parsers, each knowing the expected output format.

Expected response formats (specified in prompt templates):
```
# Threat Assessment response
THREAT_RANKING:
1. P2 — Krenko has lethal next turn with goblin tokens
2. P1 — Atraxa card advantage engine will overwhelm long-term
3. P4 — Pillow fort, not aggressive, low priority

# Attack Targeting response
TARGET: P2
ATTACKERS: Prossh
REASONING: Krenko at 18 life, removing goblin threat is priority

# Removal Targeting response
TARGET: Krenko, Mob Boss
REASONING: Token generator is the engine; removing it neutralizes P2

# Board Wipe response
DECISION: WAIT
REASONING: We have Prossh and Blood Artist; wipe hurts us more than opponents
```

**Open Questions:**
- Regex-based parsing vs asking for JSON output?
- Error recovery: what if LLM doesn't follow format?
- Should we validate extracted targets against valid game targets?
- Confidence signal: can we detect when LLM is uncertain?

---

### A4. Caching & Cost Management

**Purpose:** Avoid redundant LLM calls; track and limit API costs.

**Board fingerprint approach:**
```
fingerprint = hash(
    per player:
        life_total,
        creature_count,
        total_power,
        nonland_permanent_count,
        cards_in_hand_count,
        commander_zone
)
```

**Invalidation:** Cache is valid as long as fingerprint matches. Significant board changes that trigger invalidation:
- Creature enters/leaves battlefield
- Player loses > 5 life
- Commander zone change
- Key permanent played or removed

**What does NOT invalidate:**
- Land drops, drawing cards, minor life changes, phase changes within a turn

**Cache structure:** Per-decision-type. Threat assessment cache is separate from attack targeting cache, because they depend on different state subsets.

**Cost estimates (Claude Haiku, ~$0.25/1M input, $1.25/1M output):**

| Scenario | Calls/Game | Tokens/Call | Est. Cost/Game |
|----------|------------|-------------|----------------|
| Phase 1 (threat only) | ~50 uncached, ~15 after cache | ~800 | ~$0.005 |
| Full system | ~200 uncached, ~60 after cache | ~1,000 | ~$0.02 |

**Open Questions:**
- Cache data structure: simple HashMap or Guava Cache with TTL?
- Max cache size / eviction policy?
- Cost tracking: per-game counter or persistent across games?
- User-configurable cost limit per game?

---

### A5. Graceful Fallback

**Purpose:** The game must always work without an API key. LLM is an enhancement, not a requirement.

**Interface pattern:**
```java
public interface StrategicAdvisor {
    ThreatRanking assessThreats(Game game, Player self);
    AttackRecommendation recommendAttack(Game game, Player self,
                                          List<Card> availableAttackers);
    TargetRecommendation recommendTarget(Game game, Player self,
                                          SpellAbility removal,
                                          List<GameEntity> validTargets);
    boolean shouldBoardWipe(Game game, Player self, SpellAbility wipe);
    boolean isAvailable();
}
```

**Two implementations:**
- `LlmStrategicAdvisor` — calls Claude API, parses responses, caches results
- `RuleBasedStrategicAdvisor` — wraps current Forge logic (evaluateBoardPosition, random selection)

**Auto-switch triggers:**
- No API key configured → RuleBased from start
- API timeout → RuleBased for this call, retry LLM next call
- API error (401, 500) → RuleBased, log warning
- Parse failure → RuleBased for this call

**Open Questions:**
- Partial degradation: what if LLM works for threats but not attacks? Per-method fallback?
- Should fallback be sticky (once failed, stay on RuleBased for N turns) or per-call?
- Logging/alerting when fallback activates?

---

### A6. Event Recorder

**Purpose:** Capture recent game events for the "recent events" section of serialized state. Provides episodic memory within a single game.

**Design:** Observer pattern, hooks into game engine without modifying it.

**Events to record:**
- Player cast a spell (card name, target if relevant)
- Creature died (name, controller, cause if available)
- Player attacked player (attacker → defender, creatures involved)
- Significant life loss (> 3 life in one event)
- Commander entered/left battlefield or command zone
- Board wipe occurred (spell name)
- Player drew multiple cards (3+, source if known)

**Storage:** Ring buffer of last ~20 events, tagged with turn number.

**Serialization:** Natural language, filtered to last 2-3 turns for prompt inclusion.

**Open Questions:**
- How to hook into Forge's game engine: does it have an event/observer system we can subscribe to, or do we need to instrument specific methods?
- Natural language formatting: templates per event type?
- Should political memory (who attacked whom, cumulative) be part of this or separate (B6)?

---

## Dependencies

### New dependencies to add to forge-ai/pom.xml:
```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.11.0</version>
</dependency>
```

### Existing dependencies leveraged:
- `java.net.http.HttpClient` (Java 17 stdlib)
- `forge.util.ThreadUtil` (timeout/cancellation)
- tinylog (logging — project migrated from minlog as of upstream sync 2026-03-14)
- Guava (potential cache via `CacheBuilder`, already in forge-core)

---

## Package Structure

```
forge-ai/src/main/java/forge/ai/llm/
├── LlmClient.java                  # A1: HTTP client for Claude API
├── GameStateSerializer.java         # A2: Game state → prompt text
├── ResponseParser.java              # A3: LLM text → structured advice
├── BoardFingerprint.java            # A4: Cache key computation
├── AdvisorCache.java                # A4: Per-decision-type caching
├── CostTracker.java                 # A4: Token/cost accounting
├── StrategicAdvisor.java            # A5: Interface
├── LlmStrategicAdvisor.java         # A5: LLM implementation
├── RuleBasedStrategicAdvisor.java   # A5: Fallback (wraps existing logic)
├── GameEventRecorder.java           # A6: Event observer + ring buffer
├── GameEvent.java                   # A6: Event data model
└── model/
    ├── DecisionType.java            # Enum: THREAT, ATTACK, REMOVAL, etc.
    ├── ThreatRanking.java           # Advice model
    ├── AttackRecommendation.java    # Advice model
    ├── TargetRecommendation.java    # Advice model
    └── LlmConfig.java              # API key, model, timeout settings
```

---

## Relationship to Other Specs

- **strategic-decisions** spec defines the decision modules (B1-B6) that consume this infrastructure
- **observability** spec defines the reasoning display (C2) that reads from this layer's outputs
- **multiplayer-sim-fix** (C1) is fully independent — no dependency on this layer
