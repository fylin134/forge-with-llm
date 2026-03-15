# Forge AI Module - Architecture Documentation

## Overview

**Forge AI** is an artificial intelligence engine for playing Magic: The Gathering. It provides computer opponents capable of making intelligent gameplay decisions across thousands of unique card interactions.

---

## What It Does

The forge-ai module implements an AI player that can:

- **Make strategic decisions** about playing spells and activating abilities
- **Evaluate combat scenarios** for optimal attacking and blocking
- **Handle complex targeting** and cost payment decisions
- **Manage triggered abilities** (optional and mandatory)
- **Simulate future game states** to evaluate outcomes
- **Build and sideboard decks** using evaluation heuristics

---

## Tech Stack

### Core Technologies
- **Language**: Java
- **Build System**: Maven
- **Module Type**: AI Decision Layer (no UI, no rules enforcement)

### Dependencies
```xml
<dependency>
    <groupId>forge</groupId>
    <artifactId>forge-core</artifactId>      <!-- Card data, deck structures -->
</dependency>
<dependency>
    <groupId>forge</groupId>
    <artifactId>forge-game</artifactId>      <!-- Game rules engine -->
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-math3</artifactId>   <!-- Mathematical utilities -->
    <version>3.6.1</version>
</dependency>
```

### Architecture Style
- **Rule-based AI** (not machine learning)
- **Modular, registry-based** ability system
- **Strategy and Visitor patterns** for extensibility
- **Simulation-based** for complex decisions

---

## Architecture

### Module Structure

```
forge-ai/
└── src/main/java/forge/ai/
    ├── Core Controllers (24 files)
    │   ├── PlayerControllerAi.java (1,650 lines) - Main AI interface
    │   ├── AiController.java - AI "brain"
    │   ├── AiAttackController.java - Attack logic
    │   ├── AiBlockController.java - Block logic
    │   └── ...
    ├── ability/ (150+ AI implementations, ~29,000 lines)
    │   ├── CounterAi.java - Counterspell logic
    │   ├── DamageDealAi.java - Burn spell logic
    │   ├── DestroyAi.java - Removal logic
    │   ├── DrawAi.java - Card draw logic
    │   └── ...
    └── simulation/ (Game state simulation)
        ├── GameSimulator.java
        ├── SpellAbilityPicker.java
        └── GameStateEvaluator.java
```

### Architectural Layers

#### 1. Interface Layer
**PlayerControllerAi** (`PlayerControllerAi.java:1`)
- Extends `PlayerController` from forge-game
- Acts as the AI's interface to the game engine
- Implements all player decision methods (1,650+ lines)
- Handles: mulligans, targeting, blocking, attacking, card choices, sideboarding

#### 2. Strategic Layer
**AiController** (`AiController.java:1`)
- Core AI "brain" containing game-playing logic
- **Memory system** (`AiCardMemory`) tracks revealed cards and game state
- **Combat prediction** for evaluating attacks/blocks
- **Simulation support** for complex decision-making
- Deck evaluation and card filtering

#### 3. Tactical Layer
**SpellAbilityAi** (abstract base class)
- Template pattern for ability evaluation
- Standard decision flow:
  1. Check phase restrictions
  2. Check AI logic flags
  3. Evaluate API-specific logic
  4. Check cost acceptability
  5. Validate targeting
- Returns `AiAbilityDecision` (rating + reason)

#### 4. Ability-Specific Layer
**150+ AI implementations** in `ability/` package
- One specialized AI class per effect type
- Examples:
  - `CounterAi` - Counterspell decisions
  - `DamageDealAi` - Burn spell targeting (54KB, sophisticated logic)
  - `DestroyAi` - Removal evaluation
  - `PumpAi` - Combat trick timing
  - `TokenAi` - Token generation evaluation

---

## Key Components

### Decision System

#### AiPlayDecision (enum with 26+ states)
**Positive Decisions:**
- `WillPlay` - Default play decision
- `MandatoryPlay` - Must activate/play
- `ImpactCombat` - Affects combat
- `Removal` - Kills opponent's permanents
- `CardAdvantage` - Generates card advantage

**Wait Decisions:**
- `WaitForCombat` - Hold until combat
- `WaitForMain2` - Play in second main phase
- `WaitForEndOfTurn` - Play during opponent's end step

**Negative Decisions:**
- `CantPlayAi` - AI chooses not to play
- `CantAfford` - Insufficient resources
- `TargetingFailed` - No valid targets
- `CostNotAcceptable` - Cost too high
- `LifeInDanger` - Playing would risk losing

#### AiAbilityDecision (record)
```java
record AiAbilityDecision(int rating, AiPlayDecision decision)
```
- **rating**: Numerical score (> 30 = willing to play)
- **decision**: Enum reason for the decision
- Allows nuanced evaluation beyond yes/no

#### AiCostDecision
- Handles payment decisions for all cost types (865+ lines)
- Visitor pattern for 30+ cost types:
  - Discard, Sacrifice, Tap, Exile, Life payment, etc.
- Smart prioritization:
  - Remove negative counters first
  - Sacrifice least useful permanents
  - Discard worst cards

### Combat Controllers

#### AiAttackController (`AiAttackController.java:1`)
- Multi-threaded attack evaluation (supports timeouts)
- Aggression calculation based on life totals
- Defender prioritization (players vs planeswalkers vs battles)
- Evaluates all attack combinations
- Simulates combat outcomes

#### AiBlockController (`AiBlockController.java:1`)
- Threat-based attacker sorting
- Distinguishes "safe blockers" (won't die) from "killing blockers" (kill attacker)
- Life-in-danger detection
- Minimizes damage and maximizes favorable trades

### Ability Registry

**SpellApiToAi** (`SpellApiToAi.java:1`)
- Central registry mapping `ApiType` → AI implementation
- EnumMap for fast lookups (200+ mappings)
- Examples:
  ```
  ApiType.Counter → CounterAi
  ApiType.DealDamage → DamageDealAi
  ApiType.Draw → DrawAi
  ApiType.Destroy → DestroyAi
  ApiType.Pump → PumpAi
  ```
- Lazy instantiation via reflection

### Simulation System

**GameSimulator**
- Copies game state for look-ahead analysis
- Deep clones all game objects
- Used for complex decisions

**SpellAbilityPicker**
- Selects optimal spell sequences
- Evaluates multiple play lines

**GameStateEvaluator**
- Scores board positions
- Considers life totals, board presence, card advantage

### Utility Classes

**ComputerUtil*** family:
- `ComputerUtil` - 200+ static utility methods
- `ComputerUtilCard` - Card evaluation and selection
- `ComputerUtilCombat` - Combat calculations
- `ComputerUtilMana` - Mana availability and payment
- `ComputerUtilCost` - Cost payment logic
- `ComputerUtilAbility` - Ability evaluation utilities

**AiCardMemory** (`AiCardMemory.java:1`)
- Tracks revealed cards (from scry, mill, etc.)
- Maintains known information state
- Helps AI make informed decisions

**CreatureEvaluator** (`CreatureEvaluator.java:1`)
- Evaluates creature strength
- Considers power/toughness, keywords, abilities

---

## Decision Flow

### Main Priority Phase Loop

```
1. PRIORITY PHASE (chooseSpellAbilityToPlay)
   ├─> Get all playable abilities from hand/battlefield
   ├─> Sort by evaluation (best spells first)
   ├─> For each ability:
   │   ├─> Check if can play (canPlaySa)
   │   │   ├─> Check restrictions
   │   │   ├─> Check phase restrictions
   │   │   ├─> Check AI logic flags
   │   │   └─> Check API-specific logic
   │   ├─> Check costs (willPayCosts)
   │   ├─> Check conditions
   │   └─> Return decision with rating
   └─> Play highest-rated ability (if rating > MIN_RATING)

2. COMBAT PHASE
   ├─> DECLARE ATTACKERS (AiAttackController)
   │   ├─> Evaluate all potential attackers
   │   ├─> Calculate aggression level
   │   ├─> Choose defenders (players/planeswalkers/battles)
   │   ├─> Simulate combat outcomes
   │   └─> Declare optimized attack
   └─> DECLARE BLOCKERS (AiBlockController)
       ├─> Sort attackers by threat level
       ├─> For each attacker:
       │   ├─> Find safe blockers (won't die)
       │   ├─> Find killing blockers (will kill attacker)
       │   └─> Assign optimal blocks
       └─> Minimize damage/maximize trades

3. TRIGGERED ABILITIES (confirmTrigger)
   ├─> Mandatory: Always trigger
   └─> Optional: Evaluate with doTrigger()

4. STACK INTERACTION
   ├─> Counter spells (chooseCounterSpell)
   ├─> Change targets (if beneficial)
   └─> Activate response abilities
```

### Spell Evaluation Process

```
SpellAbilityAi.canPlayWithSubs()
├─> canPlay(ai, sa)
│   ├─> Check restrictions (timing, zone, etc.)
│   ├─> canPlayWithoutRestrict()
│   │   ├─> checkAiLogic() - Special logic flags
│   │   ├─> checkPhaseRestrictions() - Timing
│   │   ├─> checkApiLogic() - Main evaluation
│   │   │   └─> API-specific AI (DrawAi, CounterAi, etc.)
│   │   ├─> willPayCosts() - Cost evaluation
│   │   └─> checkConditions() - Spell conditions
│   └─> Return AiAbilityDecision
└─> chkDrawbackWithSubs() - Evaluate sub-abilities recursively
```

---

## Design Patterns

1. **Strategy Pattern** - Different AI for different card types
2. **Visitor Pattern** - Cost decision handling (30+ cost types)
3. **Template Method** - SpellAbilityAi provides evaluation structure
4. **Registry Pattern** - SpellApiToAi maps effects to implementations
5. **Memento Pattern** - Game state copying for simulation
6. **Factory Pattern** - Creating ability-specific AI instances

---

## Example AI Logic

### Counterspell AI (CounterAi)
```java
// Key logic considerations:
- Check if spell on stack is counterable
- Verify it's an opponent's spell
- Evaluate CMC thresholds (MinCMC logic)
- Use AI profiles for counter probability
- Always counter certain spell types (if configured)
- Handle "unless" costs (Mana Leak effects)
- Avoid wasting counterspells on low-value targets
```

### Burn Spell AI (DamageDealAi, 54KB)
```java
// Sophisticated targeting:
- Prioritize killing opponents below 5 life
- Consider combat phases and blocking scenarios
- Evaluate probabilistically based on hand size
- Handle "Sudden Impact" style cards (damage = hand size)
- Factor in lifelink and damage prevention
- Don't burn opponents with life gain unless beneficial
```

### Combat Trick AI (PumpAi)
```java
// Phase-aware evaluation:
- Extensive keyword analysis (Flying, Trample, First Strike)
- Activate during combat, not outside
- Evaluate combat relevance of keywords
  (e.g., Flying only useful if opponent has non-flyers)
- Consider offensive (attacking) and defensive (blocking)
- Filter targets based on combat state
- Special logic for curse effects (negative pumps)
```

---

## Special Features

### AI Logic Parameters
Cards can specify `AILogic` parameters for custom handling:
- `MinCMC.3` - Only counter spells with CMC ≥ 3
- `AtOppEOT` - Play during opponent's end step
- `Polymorph` - Destroy own weak creatures
- `Fight` - Special fight spell logic
- `MoveCounter` - Counter manipulation logic

Allows fine-tuning without changing Java code.

### AI Profiles
Configurable AI difficulty/personality:
- `CHANCE_TO_COUNTER_CMC_1` - Probability of countering 1-CMC spells
- Aggression levels
- Risk tolerance
- Mulligan strategies

### Special Card Handling
**SpecialCardAi** class handles edge cases for specific cards that need unique logic beyond their standard API type.

---

## Relationship to Other Modules

### Dependencies

```
forge-core (Foundation)
    ↓
forge-game (Rules Engine)
    ↓
forge-ai (Decision Layer)  ← YOU ARE HERE
    ↓ consumed by
forge-gui-* (User Interfaces)
```

### What forge-ai Provides

#### To **forge-game**:
- Implementation of `PlayerController` interface
- AI player that responds to game engine queries
- Decision-making for all player choices

#### To **forge-gui-***:
- `LobbyPlayerAi` - AI player representation in lobby
- AI configuration options
- Deck building algorithms

### What forge-ai Consumes

#### From **forge-core**:
- `CardDb` - Query card pool for deck generation
- `Deck` - Deck structures
- `PaperCard` - Card instances
- `CardAiHints` - Deck building hints

#### From **forge-game**:
- `PlayerController` interface - What AI must implement
- `Game`, `Card`, `Player` - Game state to evaluate
- `SpellAbility` - Abilities to evaluate
- `Combat` - Combat state for attack/block decisions

---

## Key Statistics

- **Total Java Files**: 174
- **Core Controllers**: 24 files
- **Ability AI Implementations**: 150+ files
- **Total Lines of Code**: ~29,000+ in ability package alone
- **Largest Modules**:
  - `ChangeZoneAi.java` - 96KB
  - `AttachAi.java` - 70KB
  - `CountersPutAi.java` - 57KB
  - `DamageDealAi.java` - 54KB
- **Effect Types Handled**: 200+

---

## Strengths & Design Philosophy

### Strengths
1. **Modularity** - Easy to add new card-specific AI
2. **Extensibility** - Registry pattern allows clean extensions
3. **Separation of Concerns** - Clear layers (interface, strategic, tactical)
4. **Type Safety** - Visitor pattern for cost decisions
5. **Performance** - Lazy loading, efficient lookups

### Design Philosophy
- **No rule enforcement** - AI only makes decisions
- **Pure decision layer** - Consumes game state, doesn't modify it directly
- **Simulation-based** - Can look ahead for complex decisions
- **Profile-driven** - Customizable AI personality
- **Heuristic-based** - Fast, deterministic (with randomness to avoid predictability)

---

## Future Considerations

The AI is rule-based and highly specialized for Magic: The Gathering. Potential improvements could include:
- Machine learning for card evaluation
- Monte Carlo tree search for complex decisions
- Better opponent modeling
- More sophisticated bluffing/information hiding
- Improved deck building algorithms

However, the current architecture handles thousands of unique cards effectively and provides challenging gameplay for most scenarios.
