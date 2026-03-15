# Forge AI & Testing Skills for Claude Code

This directory contains specialized skills for working with the Forge Magic: The Gathering codebase, focusing on **testing** and **AI development**.

## Available Skills

### 🧪 Testing Skills

#### 1. [test-card-interaction](test-card-interaction.md)
Creates integration tests for Magic: The Gathering card interactions using game simulation.

**Use when:**
- Testing card behavior and interactions
- Verifying triggers, combat, zone changes
- Creating regression tests for bug fixes
- Testing complex multi-card scenarios

**Example:**
```
User: Create a test for Lightning Bolt dealing damage to a creature
Claude: [Uses test-card-interaction skill to create comprehensive test]
```

#### 2. [test-ai-behavior](test-ai-behavior.md)
Creates tests that verify AI decision-making for cards, abilities, and game situations.

**Use when:**
- Testing if AI plays cards correctly
- Verifying AI targeting decisions
- Testing AI combat decisions (attacking/blocking)
- Creating regression tests for AI improvements

**Example:**
```
User: Test if AI uses removal on the biggest threat
Claude: [Uses test-ai-behavior skill to create AI decision test]
```

---

### 🤖 AI Development Skills

#### 3. [add-ability-ai](add-ability-ai.md)
Creates a new AI implementation for a card ability type (ApiType).

**Use when:**
- Adding AI for a new ApiType that doesn't have AI yet
- Card abilities aren't being played by AI
- Implementing AI for custom/new mechanics
- Replacing placeholder AI (AlwaysPlayAi/CannotPlayAi)

**Example:**
```
User: Add AI for the Scry ability
Claude: [Uses add-ability-ai skill to create ScryAi.java with proper logic]
```

#### 4. [improve-ability-ai](improve-ability-ai.md)
Improves existing AI decision-making by fixing suboptimal behavior.

**Use when:**
- AI plays cards at the wrong time
- AI makes poor targeting choices
- AI doesn't recognize optimal plays
- Bug reports about AI behavior

**Example:**
```
User: AI is using Murder on 1/1 creatures instead of big threats
Claude: [Uses improve-ability-ai skill to fix targeting priority]
```

---

## How to Use These Skills

### Method 1: Direct Invocation
Claude Code can use these skills automatically when appropriate. Simply describe what you need:

```
❌ Don't: "Use the test-card-interaction skill"
✅ Do: "Create a test for Lightning Bolt dealing 3 damage to a player"
```

Claude will recognize this matches a skill and use it automatically.

### Method 2: Explicit Request
You can explicitly request a skill if needed:

```
User: I need to test if the AI attacks correctly with multiple creatures
Claude: I'll use the test-ai-behavior skill to create a comprehensive test...
```

### Method 3: Guided Workflow
Claude can guide you through skill usage:

```
User: The AI isn't playing counterspells correctly
Claude: I'll help fix that using the improve-ability-ai skill. First, let me understand the problem...
[Claude asks clarifying questions]
[Claude uses skill to analyze and fix]
```

---

## Skill Workflow Examples

### Example 1: Testing a New Card

```
User: I added a new card "Custom Bolt" that deals 5 damage. Can you test it?

Claude: I'll create a comprehensive test for your card.

[Uses test-card-interaction skill]

1. Creates test class CustomBoltTest.java
2. Tests damage to creatures
3. Tests damage to players
4. Tests with damage prevention
5. Verifies card goes to graveyard

Result: Test file created at forge-gui-desktop/src/test/java/forge/gamesimulationtests/CustomBoltTest.java
```

### Example 2: Fixing AI Behavior

```
User: The AI never uses Lightning Bolt even when it would win the game

Claude: Let me investigate and fix this.

[Uses improve-ability-ai skill]

1. Locates DamageDealAi.java
2. Reads current implementation
3. Identifies issue: AI doesn't check for lethal damage
4. Adds logic to prioritize lethal damage
5. Creates test to verify fix
6. Tests pass ✓

Result: AI now correctly uses bolt for lethal damage
```

### Example 3: Adding New AI

```
User: I need AI for the new "Discover" mechanic

Claude: I'll implement AI for the Discover ability.

[Uses add-ability-ai skill]

1. Checks if DiscoverAi exists (it doesn't)
2. Reads similar AI (DigAi, CascadeAi)
3. Creates DiscoverAi.java with appropriate logic
4. Registers in SpellApiToAi
5. Creates test to verify AI uses it correctly

Result: AI now plays Discover cards appropriately
```

### Example 4: Testing Complex Interactions

```
User: Test if First Strike kills blocker before taking damage

Claude: I'll create a combat interaction test.

[Uses test-card-interaction skill]

1. Creates test with first strike attacker
2. Sets up normal blocker
3. Simulates combat
4. Verifies first striker survives
5. Verifies blocker dies

Result: Test confirms first strike interaction works correctly
```

---

## Quick Reference

### When Working on Tests
- **Card interactions** → `test-card-interaction`
- **AI decisions** → `test-ai-behavior`

### When Working on AI
- **New AI** → `add-ability-ai`
- **Fix existing AI** → `improve-ability-ai`

---

## Skill Features

Each skill provides:

✅ **Step-by-step guidance** - Clear instructions for each stage
✅ **Code templates** - Ready-to-use code patterns
✅ **Common patterns** - Examples for typical scenarios
✅ **Troubleshooting** - Solutions for common problems
✅ **Best practices** - Guidelines for quality code
✅ **Success criteria** - Checklist to verify completion

---

## Project Context

### Forge Architecture

**forge-core** (Data Layer)
- Card database and metadata
- Deck structures
- Shared utilities

**forge-game** (Rules Engine)
- MTG comprehensive rules
- Game state management
- Card effects and triggers
- 757 files, 100K+ lines

**forge-ai** (AI Layer)
- AI decision-making
- 150+ ability AI implementations
- Combat AI (attacking/blocking)
- 174 files, 29K+ lines

**forge-gui-*** (UI Layer)
- Desktop, Mobile, Android interfaces
- Game visualization
- User interaction

### Testing Framework

**TestNG** (not JUnit!)
- Use `@Test` annotations
- Use `AssertJUnit` for assertions
- Extend `AITest` base class for helpers

**Test Locations:**
- `forge-gui-desktop/src/test/java/forge/gamesimulationtests/` - Integration tests
- `forge-gui-desktop/src/test/java/forge/ai/` - AI tests
- `forge-game/src/test/java/forge/game/` - Unit tests

**Test Infrastructure:**
- `AITest` - Base class with helper methods
- `addCard()` - Add cards to game
- `playUntilPhase()` - Advance game
- `createGame()` - Initialize game

### AI Architecture

**Decision Flow:**
```
Card script → ApiType → SpellApiToAi → AbilityAi
                                          ↓
                                    checkApiLogic()
                                          ↓
                                   AiAbilityDecision
                                   (rating + reason)
```

**Key AI Classes:**
- `SpellAbilityAi` - Base class for all ability AI
- `SpellApiToAi` - Registry mapping ApiType to AI
- `AiController` - Main AI brain
- `AiAttackController` - Attack decisions
- `AiBlockController` - Block decisions
- `ComputerUtil*` - Helper utilities

---

## Common Tasks

### Create a Card Interaction Test
```
1. Identify cards involved
2. Determine expected behavior
3. Use test-card-interaction skill
4. Run test to verify
```

### Create an AI Behavior Test
```
1. Identify AI decision to test
2. Setup scenario
3. Use test-ai-behavior skill
4. Verify AI makes correct choice
```

### Add New Ability AI
```
1. Find ApiType in ApiType.java
2. Check if AI exists in SpellApiToAi
3. Use add-ability-ai skill
4. Test with real cards
```

### Fix AI Bug
```
1. Document incorrect behavior
2. Create failing test
3. Use improve-ability-ai skill
4. Verify test passes
5. Test edge cases
```

---

## Tips for Success

### For Testing

1. **Start simple** - Test one interaction at a time
2. **Explicit state** - Set teams, life, mana explicitly
3. **Clear assertions** - Test one thing per assertion
4. **Isolated tests** - Each test should be independent
5. **Document expected behavior** - Comment what should happen

### For AI Development

1. **Study similar AI** - Read existing implementations
2. **Use helpers** - Leverage ComputerUtil* classes extensively
3. **Test edge cases** - No targets, no mana, low life
4. **Phase awareness** - Consider when to play abilities
5. **Return descriptive reasons** - Use appropriate AiPlayDecision enum

### For Code Quality

1. **Read before editing** - Understand full context
2. **Small changes** - One improvement at a time
3. **Add comments** - Explain *why*, not just *what*
4. **Run tests** - Verify changes don't break anything
5. **Document decisions** - Note what was changed and why

---

## Troubleshooting

### Test Won't Compile
- Check imports (TestNG, not JUnit!)
- Verify card names are spelled correctly
- Ensure using `AITest` base class
- Check `AssertJUnit` (not `Assert`)

### Test Fails Unexpectedly
- Add logging: `System.out.println("Debug: " + value);`
- Call `game.getAction().checkStateEffects(true);` after changes
- Verify teams are set (different teams = opponents)
- Check mana was added before casting spells

### AI Not Playing Ability
- Check registration in `SpellApiToAi`
- Verify `checkApiLogic` returns positive decision
- Review phase restrictions
- Add logging to see AI's decision

### AI Makes Wrong Decision
- Review targeting priority logic
- Check timing/phase restrictions
- Verify cost evaluation
- Test with simpler scenarios first

---

## Further Documentation

- [ARCHITECTURE.md](../forge-ai/ARCHITECTURE.md) - AI architecture overview
- [Card-scripting-API](../docs/Card-scripting-API/) - Card script format
- [AITest.java](../forge-gui-desktop/src/test/java/forge/ai/AITest.java) - Test helper methods

---

## Contributing

When creating or improving skills:

1. **Clear purpose** - What problem does it solve?
2. **Step-by-step** - Break down into actionable steps
3. **Examples** - Provide concrete code examples
4. **Patterns** - Show common use cases
5. **Troubleshooting** - Address common issues
6. **Success criteria** - Define what "done" means

---

## Feedback

These skills are designed to improve agentic coding quality in the Forge codebase. If you find issues or have suggestions:

1. Document the problem clearly
2. Provide example of expected behavior
3. Suggest improvements to skill content
4. Share successful usage patterns

---

## Version

Skills Version: 1.0
Last Updated: 2025-12-31
Forge Target: master branch

---

## Quick Start

**I want to test a card:**
→ Use `test-card-interaction`

**I want to test AI behavior:**
→ Use `test-ai-behavior`

**I need to add AI for a new ability:**
→ Use `add-ability-ai`

**I need to fix AI behavior:**
→ Use `improve-ability-ai`

Simply describe what you need to Claude, and the appropriate skill will be used automatically!
