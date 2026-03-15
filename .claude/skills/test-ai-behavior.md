# Skill: test-ai-behavior

## Purpose
Creates tests that verify AI decision-making for specific cards, abilities, or game situations in Forge.

## When to Use
- Testing if AI plays a card correctly
- Verifying AI targeting decisions
- Testing AI combat decisions
- Creating regression tests for AI behavior
- Validating AI improvements or fixes

## Prerequisites
- Understanding of the AI decision to test
- Card(s) involved in the decision
- Expected AI behavior description

## Steps

### 1. Choose Test Location

**For AI ability decisions:**
- Location: `forge-gui-desktop/src/test/java/forge/ai/`
- Base class: Extend `AITest`

**For AI combat decisions:**
- Location: `forge-gui-desktop/src/test/java/forge/ai/attacking/` or `/blocking/`
- Base class: Extend `AITest`

**For comprehensive AI tests:**
- Location: `forge-gui-desktop/src/test/java/forge/ai/AIIntegrationTests.java`
- Add to existing test class

### 2. Read Relevant AI Implementation

Understand what the AI should do:
```
forge-ai/src/main/java/forge/ai/ability/{AbilityType}Ai.java
forge-ai/src/main/java/forge/ai/AiAttackController.java
forge-ai/src/main/java/forge/ai/AiBlockController.java
```

Look for:
- Decision logic in `checkApiLogic()`
- Targeting priorities
- Phase restrictions
- Cost evaluation

### 3. Read Existing AI Tests

Study patterns from similar tests:
```
forge-gui-desktop/src/test/java/forge/ai/AIIntegrationTests.java
forge-gui-desktop/src/test/java/forge/ai/attacking/BasicAttackTests.java
forge-gui-desktop/src/test/java/forge/ai/blocking/BlockTests.java
```

### 4. Create Test Class (if needed)

**Naming Convention:**
- For ability AI: `{CardName}AiTest.java` or `{AbilityType}AiTest.java`
- For combat AI: `{Scenario}AttackTest.java` or `{Scenario}BlockTest.java`

**Template:**
```java
package forge.ai;

import forge.game.Game;
import forge.game.card.Card;
import forge.game.combat.Combat;
import forge.game.phase.PhaseType;
import forge.game.player.Player;
import forge.game.player.PlayerControllerAi;
import org.testng.AssertJUnit;
import org.testng.annotations.Test;

public class MyAiTest extends AITest {

    @Test
    public void testAiBehaviorDescription() {
        // Setup game with AI player
        Game game = initAndCreateGame();
        Player ai = game.getPlayers().get(1); // AI is usually player 2
        Player human = game.getPlayers().get(0);

        ai.setTeam(1);
        human.setTeam(0);

        // Setup board state
        // ...

        // Get AI's decision
        // ...

        // Assert expected behavior
        AssertJUnit.assertTrue("AI should...", condition);
    }
}
```

### 5. Setup Board State for AI Test

**Add cards to AI's battlefield:**
```java
Card aiCard = addCard("Card Name", ai);
aiCard.setSickness(false);
```

**Add cards to opponent's battlefield:**
```java
Card oppCard = addCard("Opposing Card", human);
```

**Set life totals to influence AI decisions:**
```java
ai.setLife(5, null); // AI low on life (defensive)
human.setLife(3, null); // Opponent low (AI should be aggressive)
```

**Give AI mana:**
```java
// Add lands
Card land = addCard("Mountain", ai);
land.setTapped(false);

// Or add mana directly
ai.addMana("R R R");
```

**Add cards to AI's hand:**
```java
Card cardInHand = addCardToZone("Lightning Bolt", ai, ZoneType.Hand);
```

### 6. Test AI Spell/Ability Decisions

**Method 1: Let AI play through priority:**
```java
// Advance to AI's turn
game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
game.getAction().checkStateEffects(true);

// Let AI have priority and make decisions
playUntilPhase(game, PhaseType.END_OF_TURN);

// Check what AI did
AssertJUnit.assertTrue("AI should have cast the spell",
    spell.isInZone(ZoneType.Graveyard));
AssertJUnit.assertTrue("Target should be dead",
    target.isInZone(ZoneType.Graveyard));
```

**Method 2: Check AI's ability evaluation:**
```java
SpellAbility sa = findSAWithPrefix(card, "SP");

// Get AI controller
PlayerControllerAi aiController = (PlayerControllerAi) ai.getController();

// Check if AI wants to play this
boolean wouldPlay = aiController.getAi().canPlaySa(sa) != null;
AssertJUnit.assertTrue("AI should want to play this ability", wouldPlay);
```

**Method 3: Check AI targeting:**
```java
SpellAbility sa = findSAWithPrefix(card, "SP");
if (sa != null && sa.usesTargeting()) {
    // Let AI choose targets
    PlayerControllerAi aiController = (PlayerControllerAi) ai.getController();

    // AI should target the best option
    // Advance game and let AI play
    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.MAIN2);

    // Verify target choice
    // (check spell on stack or in graveyard)
}
```

### 7. Test AI Combat Decisions

**Test AI attacking decisions:**
```java
@Test
public void aiAttacksWhenCanWin() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);
    human.setLife(2, null); // Can kill with a 2/2

    // Give AI a creature
    Card attacker = addCard("Grizzly Bears", ai);
    attacker.setSickness(false);

    // Move to combat
    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    game.getAction().checkStateEffects(true);

    // Get AI's predicted combat
    PlayerControllerAi aiController = (PlayerControllerAi) ai.getController();
    Combat combat = aiController.getAi().getPredictedCombat();

    // AI should attack for lethal
    AssertJUnit.assertEquals("AI should attack with 1 creature",
        1, combat.getAttackers().size());
    AssertJUnit.assertTrue("AI should attack with the bear",
        combat.getAttackers().contains(attacker));
}
```

**Test AI blocking decisions:**
```java
@Test
public void aiBlocksLethalAttacker() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);
    ai.setLife(2, null); // Would die to 2/2 attacker

    // Human attacks
    Card attacker = addCard("Grizzly Bears", human);
    attacker.setSickness(false);

    // AI has blocker
    Card blocker = addCard("Runeclaw Bear", ai);

    // Setup combat phase
    game.getPhaseHandler().devModeSet(PhaseType.COMBAT_DECLARE_ATTACKERS, human);
    game.getCombat().addAttacker(attacker, ai);

    // Move to declare blockers
    game.getPhaseHandler().devModeSet(PhaseType.COMBAT_DECLARE_BLOCKERS, ai);

    // AI should block
    playUntilPhase(game, PhaseType.COMBAT_DAMAGE);

    // Check if AI blocked
    Combat combat = game.getCombat();
    AssertJUnit.assertTrue("AI should have blocked the attacker",
        combat.isBlocked(attacker));
    AssertJUnit.assertTrue("AI should block with its creature",
        combat.getBlockers(attacker).contains(blocker));
}
```

### 8. Test AI Priority and Timing

**Test AI waits for combat to play instant:**
```java
@Test
public void aiWaitsForCombatToPlayCombatTrick() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);

    // Give AI a combat trick
    Card trick = addCardToZone("Giant Growth", ai, ZoneType.Hand);
    Card creature = addCard("Grizzly Bears", ai);
    creature.setSickness(false);

    // Give AI mana
    Card forest = addCard("Forest", ai);
    forest.setTapped(false);

    // In main phase 1, AI should NOT play combat trick yet
    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.COMBAT_BEGIN);

    AssertJUnit.assertTrue("AI should hold combat trick until combat",
        trick.isInZone(ZoneType.Hand));
}
```

### 9. Test AI Doesn't Make Obvious Mistakes

**Test AI doesn't waste removal:**
```java
@Test
public void aiDoesntWasteRemovalOnSmallCreature() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);

    // AI has premium removal
    Card removal = addCardToZone("Murder", ai, ZoneType.Hand);
    ai.addMana("B B B");

    // Human only has a small creature
    Card smallCreature = addCard("Llanowar Elves", human);

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.END_OF_TURN);

    // AI should NOT waste Murder on 1/1
    AssertJUnit.assertTrue("AI should not have cast Murder",
        removal.isInZone(ZoneType.Hand));
}
```

### 10. Test AI Card Evaluation

**Test AI values card advantage:**
```java
@Test
public void aiPlaysCardDrawSpell() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);

    ai.setTeam(1);

    // Give AI divination and mana
    Card divination = addCardToZone("Divination", ai, ZoneType.Hand);
    ai.addMana("U U U");

    // Ensure library has cards
    for (int i = 0; i < 10; i++) {
        addCardToZone("Island", ai, ZoneType.Library);
    }

    int handSizeBefore = ai.getCardsIn(ZoneType.Hand).size();

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.MAIN2);

    // AI should have played divination
    AssertJUnit.assertTrue("AI should play card draw",
        ai.getCardsIn(ZoneType.Hand).size() > handSizeBefore);
}
```

### 11. Test AI with Multiple Options

**Test AI chooses best target:**
```java
@Test
public void aiTargetsBestCreatureWithRemoval() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);

    // AI has removal
    Card removal = addCardToZone("Shock", ai, ZoneType.Hand);
    ai.addMana("R");

    // Human has small and medium threat
    Card small = addCard("Llanowar Elves", human); // 1/1
    Card medium = addCard("Grizzly Bears", human); // 2/2

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.MAIN2);

    // AI should target the bigger threat
    AssertJUnit.assertTrue("AI should kill the 2/2",
        medium.isInZone(ZoneType.Graveyard));
    AssertJUnit.assertTrue("AI should leave the 1/1",
        small.isInZone(ZoneType.Battlefield));
}
```

## Common Patterns

### Pattern 1: Test AI Plays Card
```java
// Setup: AI has card and mana
// Execute: Advance to AI's turn
// Assert: Card was played
```

### Pattern 2: Test AI Doesn't Play Card
```java
// Setup: Unfavorable condition
// Execute: Advance through AI's turn
// Assert: Card still in hand
```

### Pattern 3: Test AI Combat Aggression
```java
// Setup: AI can win by attacking
// Execute: Get predicted combat
// Assert: AI attacks with sufficient creatures
```

### Pattern 4: Test AI Combat Defense
```java
// Setup: AI would die if doesn't block
// Execute: Human attacks, AI blocks
// Assert: AI blocked lethal attacker
```

### Pattern 5: Test AI Targeting Priority
```java
// Setup: Multiple valid targets
// Execute: AI plays targeted ability
// Assert: Best target was chosen
```

## Troubleshooting

**AI doesn't play when expected:**
- Check mana is available: `ai.addMana()` or untapped lands
- Verify card is in hand/battlefield
- Check phase restrictions in AI logic
- Ensure StaticData loaded (card scripts readable)
- Add logging to see AI's decision: `AiPlayDecision`

**AI plays unexpectedly:**
- Check life totals (affects aggression)
- Verify opponent board state
- Check if AI thinks it can win
- Review AI logic in corresponding `*Ai.java` file

**Test is flaky:**
- Don't rely on random AI behavior
- Set explicit game state
- Test predicted decisions, not execution
- Ensure deterministic setup

**Can't access AI's decision:**
- Cast to `PlayerControllerAi`: `(PlayerControllerAi)player.getController()`
- Get AI controller: `.getAi()`
- Get predicted combat: `.getPredictedCombat()`

## Best Practices

1. **Test AI logic, not randomness** - Use predicted actions
2. **Clear setup** - Make it obvious why AI should/shouldn't act
3. **Test extreme cases** - Very high/low life, lethal situations
4. **One decision per test** - Focus on specific AI behavior
5. **Document expected reasoning** - Comment why AI should act
6. **Test both positive and negative** - Should play AND shouldn't play
7. **Isolate AI decisions** - Don't test multiple decisions in one test

## Success Criteria

- [ ] Test compiles and runs
- [ ] Test clearly shows AI decision
- [ ] Expected AI behavior is documented
- [ ] Test is deterministic
- [ ] Edge cases covered (lethal, no mana, etc.)
- [ ] Test name describes AI behavior tested

## Example Output

After creating the test, report:
```
✅ AI Test Created: TestAiUsesRemovalOnThreats

Location: forge-gui-desktop/src/test/java/forge/ai/RemovalAiTest.java

Test Coverage:
- AI casts Lightning Bolt on opponent's creature
- AI targets the biggest threat (3/3 over 1/1)
- AI doesn't waste bolt when no good targets

AI Logic Verified:
✓ Targeting priority (bigger threats first)
✓ Cost evaluation (worth spending bolt)
✓ Timing (main phase before combat)

To run:
cd forge-gui-desktop && mvn test -Dtest=RemovalAiTest

Next steps:
- Run test to verify it passes
- Consider testing with multiple removal spells
- Test AI holds removal for better targets
```
