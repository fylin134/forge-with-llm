# Skill: test-card-interaction

## Purpose
Creates integration tests for Magic: The Gathering card interactions using Forge's game simulation framework.

## When to Use
- User requests a test for specific card behavior
- Testing complex card interactions (triggers, combat, etc.)
- Verifying bug fixes for card implementations
- Creating regression tests

## Prerequisites
- Card names provided by user
- Understanding of the interaction to test
- Access to test infrastructure

## Steps

### 1. Determine Test Type and Location

**For card interactions and AI behavior:**
- Location: `forge-gui-desktop/src/test/java/forge/gamesimulationtests/`
- Framework: TestNG with game simulation
- Pattern: Use `GameWrapper` with builder pattern

**For specific game logic:**
- Location: `forge-game/src/test/java/forge/game/`
- Framework: TestNG unit tests
- Pattern: Direct testing of classes

### 2. Read Test Infrastructure

Read these files to understand patterns:
```
forge-gui-desktop/src/test/java/forge/ai/AITest.java
forge-gui-desktop/src/test/java/forge/gamesimulationtests/BaseGameSimulationTest.java
```

Look for:
- `addCard(name, player)` helper
- `playUntilPhase(game, phase)` helper
- `createGame()` pattern
- Assertion patterns

### 3. Understand the Card Scripts

Read the card script files for cards involved:
```
forge-gui/res/cardsfolder/{first_letter}/{card_name}.txt
```

Understand:
- What abilities the cards have
- How they should interact
- What triggers should fire
- Expected outcomes

### 4. Create Test Class

**Naming Convention:**
- For specific interaction: `{Card1}{Card2}InteractionTest.java`
- For card feature: `{CardName}Test.java`
- For comprehensive rules: `ComprehensiveRulesSection{Number}Test.java`

**Template Structure:**
```java
package forge.gamesimulationtests;

import forge.ai.AITest;
import forge.game.Game;
import forge.game.card.Card;
import forge.game.phase.PhaseType;
import forge.game.player.Player;
import org.testng.AssertJUnit;
import org.testng.annotations.Test;

public class MyInteractionTest extends AITest {

    @Test
    public void testInteractionDescription() {
        // Setup game
        Game game = initAndCreateGame();
        Player player1 = game.getPlayers().get(0);
        Player player2 = game.getPlayers().get(1);

        // Set teams (0 vs 1 for opponents)
        player1.setTeam(0);
        player2.setTeam(1);

        // Add cards and setup board state
        Card card1 = addCard("Card Name", player1);
        card1.setSickness(false); // Remove summoning sickness if needed

        // Set life totals if needed
        player1.setLife(20, null);
        player2.setLife(20, null);

        // Advance to appropriate phase
        game.getPhaseHandler().devModeSet(PhaseType.MAIN1, player1);
        game.getAction().checkStateEffects(true);

        // Perform actions and assert results
        // ... test logic ...

        AssertJUnit.assertEquals("Expected outcome", expectedValue, actualValue);
    }
}
```

### 5. Setup Board State

**Add cards to battlefield:**
```java
Card creature = addCard("Grizzly Bears", player1);
creature.setSickness(false); // Ready to attack/activate
```

**Add cards to other zones:**
```java
Card card = addCardToZone("Lightning Bolt", player1, ZoneType.Hand);
```

**Set player state:**
```java
player.setLife(10, null);
player.addMana("R R R"); // Add mana to pool
```

**Setup combat:**
```java
game.getPhaseHandler().devModeSet(PhaseType.COMBAT_DECLARE_ATTACKERS, attacker);
game.getCombat().addAttacker(creature, defender);
```

### 6. Execute Actions

**Advance game phases:**
```java
// Move to specific phase
playUntilPhase(game, PhaseType.END_OF_TURN);

// Move to next turn
playUntilNextTurn(game);
```

**Cast spells manually:**
```java
SpellAbility sa = findSAWithPrefix(card, "SP");
if (sa != null) {
    sa.getTargets().add(target);
    game.getAction().invoke(sa);
}
```

**Activate abilities:**
```java
SpellAbility ability = findSAWithPrefix(card, "AB");
if (ability != null) {
    game.getAction().invoke(ability);
}
```

### 7. Assert Expected Outcomes

**Common assertions:**
```java
// Life totals
AssertJUnit.assertEquals("Player life incorrect", 17, player.getLife());

// Card zones
AssertJUnit.assertTrue("Card should be in graveyard",
    card.isInZone(ZoneType.Graveyard));

// Card properties
AssertJUnit.assertEquals("Creature power wrong", 3, creature.getNetPower());
AssertJUnit.assertEquals("Creature toughness wrong", 3, creature.getNetToughness());

// Game state
AssertJUnit.assertTrue("Game should be over", game.isGameOver());
AssertJUnit.assertEquals("Wrong winner", player1, game.getOutcome().getWinningPlayer());

// Keywords
AssertJUnit.assertTrue("Should have Flying",
    creature.hasKeyword(Keyword.FLYING));

// Counters
AssertJUnit.assertEquals("Should have 2 +1/+1 counters",
    2, creature.getCounters(CounterEnumType.P1P1));
```

### 8. Test AI Behavior (if applicable)

For AI tests:
```java
// Get AI's predicted combat
Combat combat = ((PlayerControllerAi)aiPlayer.getController())
    .getAi().getPredictedCombat();

int attackingCount = combat.getAttackers().size();
AssertJUnit.assertEquals("AI should attack with 3 creatures", 3, attackingCount);

// Check if AI would play a spell
SpellAbility sa = someAbility;
boolean shouldPlay = aiLogic.canPlayAi(aiPlayer, sa);
AssertJUnit.assertTrue("AI should want to play this", shouldPlay);
```

### 9. Add Test Annotations

```java
@Test
public void testBasicInteraction() {
    // Test basic case
}

@Test
public void testEdgeCase() {
    // Test edge case
}

@Test(enabled = false, description = "Known bug - see issue #1234")
public void testBrokenInteraction() {
    // Disabled test for known issue
}
```

### 10. Validate Test

Run the test:
```bash
cd forge-gui-desktop
mvn test -Dtest=MyInteractionTest
```

Verify:
- Test compiles without errors
- Test runs and passes/fails as expected
- Test is isolated (doesn't depend on other tests)
- Test is deterministic (same result every time)

## Common Patterns

### Pattern 1: Test Damage Spell
```java
@Test
public void lightningBoltDealsThreeDamage() {
    Game game = initAndCreateGame();
    Player caster = game.getPlayers().get(0);
    Player target = game.getPlayers().get(1);

    caster.setTeam(0);
    target.setTeam(1);
    target.setLife(20, null);

    Card bolt = addCardToZone("Lightning Bolt", caster, ZoneType.Hand);
    caster.addMana("R");

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, caster);

    SpellAbility sa = findSAWithPrefix(bolt, "SP");
    sa.getTargets().add(target);
    game.getAction().invoke(sa);

    AssertJUnit.assertEquals("Should deal 3 damage", 17, target.getLife());
}
```

### Pattern 2: Test Triggered Ability
```java
@Test
public void soulWardenTriggersOnCreatureETB() {
    Game game = initAndCreateGame();
    Player player = game.getPlayers().get(0);
    player.setTeam(0);
    player.setLife(20, null);

    Card warden = addCard("Soul Warden", player);
    game.getAction().checkStateEffects(true); // Process ETB

    int lifeBefore = player.getLife();

    // Add another creature (should trigger Soul Warden)
    Card bear = addCard("Grizzly Bears", player);
    game.getAction().checkStateEffects(true);

    // Resolve triggers
    playUntilPhase(game, PhaseType.MAIN1);

    AssertJUnit.assertEquals("Should gain 1 life from trigger",
        lifeBefore + 1, player.getLife());
}
```

### Pattern 3: Test Combat Interaction
```java
@Test
public void firstStrikeKillsBeforeRegularDamage() {
    Game game = initAndCreateGame();
    Player attacker = game.getPlayers().get(0);
    Player defender = game.getPlayers().get(1);

    attacker.setTeam(0);
    defender.setTeam(1);

    Card firstStriker = addCard("Elite Vanguard", attacker);
    firstStriker.setSickness(false);
    // Manually add first strike if card doesn't have it
    firstStriker.addExtrinsicKeyword("First Strike");

    Card blocker = addCard("Grizzly Bears", defender);

    // Setup combat
    game.getPhaseHandler().devModeSet(PhaseType.COMBAT_DECLARE_ATTACKERS, attacker);
    game.getCombat().addAttacker(firstStriker, defender);

    game.getPhaseHandler().devModeSet(PhaseType.COMBAT_DECLARE_BLOCKERS, defender);
    game.getCombat().addBlocker(firstStriker, blocker);

    // Resolve combat
    playUntilPhase(game, PhaseType.COMBAT_DAMAGE);

    // First striker should kill blocker before taking damage
    AssertJUnit.assertTrue("Blocker should be dead",
        blocker.isInZone(ZoneType.Graveyard));
    AssertJUnit.assertTrue("First striker should survive",
        firstStriker.isInZone(ZoneType.Battlefield));
}
```

### Pattern 4: Test Card Draw/Mill
```java
@Test
public void ancestralRecallDrawsThreeCards() {
    Game game = initAndCreateGame();
    Player player = game.getPlayers().get(0);
    player.setTeam(0);

    // Ensure library has enough cards
    for (int i = 0; i < 10; i++) {
        addCardToZone("Forest", player, ZoneType.Library);
    }

    int handSizeBefore = player.getCardsIn(ZoneType.Hand).size();
    int librarySizeBefore = player.getCardsIn(ZoneType.Library).size();

    Card recall = addCardToZone("Ancestral Recall", player, ZoneType.Hand);
    player.addMana("U");

    SpellAbility sa = findSAWithPrefix(recall, "SP");
    sa.getTargets().add(player);
    game.getAction().invoke(sa);

    AssertJUnit.assertEquals("Should draw 3 cards",
        handSizeBefore + 3, player.getCardsIn(ZoneType.Hand).size());
    AssertJUnit.assertEquals("Library should have 3 fewer cards",
        librarySizeBefore - 3, player.getCardsIn(ZoneType.Library).size());
}
```

## Troubleshooting

**Test fails to compile:**
- Check imports (TestNG, not JUnit)
- Verify card names are spelled correctly
- Ensure using correct helper methods from AITest

**Test fails unexpectedly:**
- Add logging: `System.out.println("Debug: " + variable);`
- Check state-based actions: Call `game.getAction().checkStateEffects(true);`
- Verify teams are set correctly (different teams are opponents)
- Ensure mana is added before casting spells
- Check that triggers are resolving (advance phases)

**Test is flaky:**
- Avoid depending on random events
- Set explicit game state
- Don't rely on AI decisions (use manual actions)
- Reset game state between tests

**Card not found:**
- Verify card script exists
- Check spelling and capitalization
- Ensure StaticData is initialized

## Best Practices

1. **One assertion per logical concept** - Test one thing at a time
2. **Clear test names** - Name describes what's being tested
3. **Minimal setup** - Only add cards needed for the test
4. **Explicit state** - Don't rely on defaults, set explicit values
5. **Document expected behavior** - Comments explain what should happen
6. **Test both positive and negative** - Test success and failure cases
7. **Isolate tests** - Each test should be independent
8. **Fast tests** - Avoid unnecessary game advancement

## Success Criteria

- [ ] Test compiles without errors
- [ ] Test runs successfully
- [ ] Test is deterministic (same result each run)
- [ ] Test name clearly describes what's tested
- [ ] Assertions verify expected behavior
- [ ] Test is isolated from other tests
- [ ] Edge cases are covered

## Example Output

After creating the test, report:
```
✅ Test Created: LightningBoltTest.testDealsDamageToCreature

Location: forge-gui-desktop/src/test/java/forge/gamesimulationtests/LightningBoltTest.java

Test Coverage:
- Verifies Lightning Bolt deals 3 damage
- Tests targeting a creature
- Confirms creature dies when damage >= toughness

To run:
cd forge-gui-desktop && mvn test -Dtest=LightningBoltTest

Next steps:
- Run the test to verify it passes
- Add additional test cases for other targets (player, planeswalker)
- Consider testing with damage prevention effects
```
