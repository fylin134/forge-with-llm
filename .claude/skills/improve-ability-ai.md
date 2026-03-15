# Skill: improve-ability-ai

## Purpose
Improves existing AI decision-making for card abilities by analyzing suboptimal behavior and implementing better logic.

## When to Use
- AI plays cards at the wrong time
- AI makes poor targeting choices
- AI doesn't recognize optimal plays
- AI evaluation needs refinement
- Bug reports about AI behavior

## Prerequisites
- Ability/card that AI plays incorrectly
- Description of the problem behavior
- Expected correct behavior
- Understanding of the ability's effect

## Steps

### 1. Identify the Problem

**Document the Issue:**
- What card/ability is involved?
- What is AI doing wrong?
- What should AI do instead?
- When does the problem occur?

**Example Issues:**
- "AI casts removal on 1/1 creature when 10/10 is available"
- "AI never blocks even when would die"
- "AI plays combat tricks in main phase"
- "AI doesn't counter game-winning spells"

### 2. Locate the AI Implementation

**Find the AI class:**
```
forge-ai/src/main/java/forge/ai/ability/{Ability}Ai.java
```

Use Grep to find it:
```bash
grep -r "class.*Ai extends SpellAbilityAi" forge-ai/src/main/java/forge/ai/ability/
```

**Read the current implementation:**
- Focus on `checkApiLogic()` method
- Review targeting logic (`targetAI()`)
- Check phase restrictions
- Look for special case handling

### 3. Understand Current Decision Logic

**Analyze the flow:**
```java
@Override
protected AiAbilityDecision checkApiLogic(Player ai, SpellAbility sa) {
    // What checks are done?
    // What conditions cause AI to play/not play?
    // What's the rating logic?
    // What AiPlayDecision is returned?
}
```

**Common issues to look for:**
- Missing checks for important conditions
- Wrong priority in targeting
- Phase restrictions too strict or too loose
- Cost evaluation too conservative or aggressive
- No consideration of game state

### 4. Create Test Demonstrating the Problem

**Write a failing test first:**
```java
@Test
public void aiShouldTargetBiggestThreat() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);
    Player human = game.getPlayers().get(0);

    ai.setTeam(1);
    human.setTeam(0);

    // Setup scenario that demonstrates problem
    Card removal = addCardToZone("Murder", ai, ZoneType.Hand);
    ai.addMana("B B B");

    Card small = addCard("Llanowar Elves", human); // 1/1
    Card big = addCard("Colossal Dreadmaw", human); // 6/6

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.MAIN2);

    // Current behavior: AI kills 1/1
    // Expected behavior: AI should kill 6/6
    AssertJUnit.assertTrue("AI should kill the bigger threat",
        big.isInZone(ZoneType.Graveyard));
    AssertJUnit.assertTrue("AI should leave the small creature",
        small.isInZone(ZoneType.Battlefield));
}
```

Run test to confirm it fails with current implementation.

### 5. Analyze Similar Implementations

**Find well-implemented AI:**
```bash
# Find similar ability AI
grep -l "similar pattern" forge-ai/src/main/java/forge/ai/ability/*.java
```

**Read and compare:**
- How do they handle targeting?
- What conditions do they check?
- How do they prioritize?
- What helper methods do they use?

**Good examples:**
- `DestroyAi.java` - sophisticated targeting priority
- `CounterAi.java` - conditional decision-making
- `PumpAi.java` - complex combat evaluation

### 6. Identify the Fix

**Common improvements:**

**Better Targeting Priority:**
```java
// Before: No prioritization
Card target = list.get(0);

// After: Prioritize threats
list = ComputerUtilCard.prioritizeCreaturesWorthRemovingNow(ai, list, false);
Card target = ComputerUtilCard.getBestCreatureAI(list);
```

**Better Timing:**
```java
// Before: Always plays immediately
return new AiAbilityDecision(100, AiPlayDecision.WillPlay);

// After: Holds for better opportunities
if (!ComputerUtilCard.useRemovalNow(sa, target, 0, ZoneType.Graveyard)) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForMain2);
}
```

**Better Cost Evaluation:**
```java
// Before: No cost check
return new AiAbilityDecision(100, AiPlayDecision.WillPlay);

// After: Check if cost is acceptable
if (!ComputerUtilCost.checkLifeCost(ai, cost, source, 4, sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.CostNotAcceptable);
}
```

**Better Phase Restrictions:**
```java
// Before: Plays anytime
@Override
protected boolean checkPhaseRestrictions(...) {
    return true;
}

// After: Only during combat
@Override
protected boolean checkPhaseRestrictions(Player ai, SpellAbility sa, PhaseHandler ph) {
    if (!ph.inCombat()) {
        return false;
    }
    return super.checkPhaseRestrictions(ai, sa, ph);
}
```

### 7. Implement the Improvement

**Read the current implementation first:**
Use the Read tool to read the entire file to understand context.

**Make targeted changes:**
Use Edit tool to modify specific methods.

**Example improvement:**
```java
// Improve targeting to prioritize bigger creatures
private boolean targetAI(Player ai, SpellAbility sa, boolean mandatory) {
    CardCollection list = CardLists.getTargetableCards(
        ai.getOpponents().getCardsIn(ZoneType.Battlefield), sa);

    // Filter out poor targets
    list = CardLists.filter(list, new Predicate<Card>() {
        @Override
        public boolean apply(Card c) {
            // Don't waste on tiny creatures unless mandatory
            return mandatory || c.getCMC() >= 2 || c.getNetPower() >= 2;
        }
    });

    if (list.isEmpty()) {
        return mandatory;
    }

    // Prioritize by threat level
    list = ComputerUtilCard.prioritizeCreaturesWorthRemovingNow(ai, list, false);

    // Choose best target
    Card target = ComputerUtilCard.getBestCreatureAI(list);
    sa.resetTargets();
    sa.getTargets().add(target);

    return true;
}
```

### 8. Add Helper Methods if Needed

**Extract complex logic:**
```java
private boolean isWorthRemoval(Card target, Player ai) {
    // High power/toughness
    if (target.getNetPower() >= 4 || target.getNetToughness() >= 4) {
        return true;
    }

    // Has dangerous keywords
    if (target.hasKeyword("Hexproof") || target.hasKeyword("Indestructible")) {
        return true;
    }

    // Has problematic abilities
    if (target.hasStartOfKeyword("Protection")) {
        return true;
    }

    return false;
}
```

**Use in main logic:**
```java
list = CardLists.filter(list, c -> isWorthRemoval(c, ai));
```

### 9. Test the Improvement

**Run the failing test:**
```bash
cd forge-gui-desktop
mvn test -Dtest=YourTest
```

**Verify:**
- Test now passes
- AI makes correct decision
- Doesn't break other scenarios

**Add more test cases:**
```java
@Test
public void aiStillWorksWithOnlySmallCreature() {
    // Test edge case: only small creatures available
    // AI should still target something if only option
}

@Test
public void aiDoesntWasteRemovalWhenNoBigThreats() {
    // Test: AI holds removal when no good targets
}
```

### 10. Document the Change

**Add comments explaining the logic:**
```java
@Override
protected AiAbilityDecision checkApiLogic(Player ai, SpellAbility sa) {
    // Don't waste premium removal on small threats
    // unless it's the only blocker preventing lethal
    if (!isGoodTarget(target) && !isBlockingLethal(target, ai)) {
        return new AiAbilityDecision(0, AiPlayDecision.CantPlayAi);
    }

    // Hold removal if opponent might play bigger threats
    if (ai.getOpponents().get(0).getManaLeft() > 5 && game.getPhaseHandler().isPlayerTurn(ai)) {
        return new AiAbilityDecision(0, AiPlayDecision.WaitForMain2);
    }

    return new AiAbilityDecision(100, AiPlayDecision.Removal);
}
```

## Common Improvements

### Issue: Poor Targeting Priority

**Fix: Use prioritization helpers**
```java
// Sort by threat level
list = ComputerUtilCard.prioritizeCreaturesWorthRemovingNow(ai, list, false);

// Get best option
Card best = ComputerUtilCard.getBestCreatureAI(list);

// Or: Use evaluation
CardLists.sortByEvaluateCreature(list);
```

### Issue: Wrong Timing

**Fix: Add phase/situation checks**
```java
// Don't use combat tricks outside combat
if (!game.getPhaseHandler().inCombat()) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForCombat);
}

// Don't tap creatures that need to block
if (ComputerUtil.waitForBlocking(sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForCombat);
}

// Hold for opponent's turn
if (game.getPhaseHandler().isPlayerTurn(ai) && sa.isInstant()) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForEndOfTurn);
}
```

### Issue: Doesn't Recognize Win/Loss

**Fix: Check life totals and lethal**
```java
// Play aggressively if opponent is low
if (ai.getOpponent().getLife() <= 5) {
    // More likely to use damage/removal aggressively
    return new AiAbilityDecision(100, AiPlayDecision.WillPlay);
}

// Play defensively if AI is low
if (ai.getLife() <= 5) {
    // More likely to save resources/blockers
    return new AiAbilityDecision(0, AiPlayDecision.LifeInDanger);
}

// Check for lethal
if (ComputerUtilCombat.canAttackerAttackForLethal(game, attacker, ai.getOpponent())) {
    // Enable lethal attack
    return new AiAbilityDecision(100, AiPlayDecision.WillPlay);
}
```

### Issue: Ignores Cost

**Fix: Evaluate cost vs benefit**
```java
// Check life cost
if (!ComputerUtilCost.checkLifeCost(ai, cost, source, 4, sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.CostNotAcceptable);
}

// Check sacrifice cost
if (!ComputerUtilCost.checkSacrificeCost(ai, cost, source, sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.CostNotAcceptable);
}

// Check discard cost
if (!ComputerUtilCost.checkDiscardCost(ai, cost, source, sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.CostNotAcceptable);
}
```

### Issue: Doesn't Consider Board State

**Fix: Evaluate context**
```java
// Count creatures
CardCollection myCreatures = ai.getCreaturesInPlay();
CardCollection oppCreatures = ai.getOpponents().getCreaturesInPlay();

// Check if behind on board
if (myCreatures.size() < oppCreatures.size() - 2) {
    // Play more defensively
}

// Check if need to deal with specific threat
Card biggestThreat = ComputerUtilCard.getBestCreatureAI(oppCreatures);
if (biggestThreat.getNetPower() >= 5) {
    // Prioritize removing it
}
```

## Testing Strategy

### Test Suite for AI Improvements

**1. Basic Functionality:**
```java
@Test
public void aiPlaysAbilityInBasicCase()
```

**2. Targeting Priority:**
```java
@Test
public void aiTargetsBiggestThreat()

@Test
public void aiTargetsCreatureWithDangerousKeyword()
```

**3. Timing:**
```java
@Test
public void aiWaitsForCombatToPlayTrick()

@Test
public void aiPlaysAtEndOfTurn()
```

**4. Edge Cases:**
```java
@Test
public void aiHandlesNoValidTargets()

@Test
public void aiHandlesInsufficientMana()

@Test
public void aiHandlesOnlyBadOptions()
```

**5. Regression Tests:**
```java
@Test
public void aiStillWorksInPreviousScenarios()
```

## Best Practices

1. **Read before editing** - Understand full context first
2. **Test-driven** - Write failing test, then fix
3. **Small changes** - One improvement at a time
4. **Use helpers** - Leverage ComputerUtil* classes
5. **Comment reasoning** - Explain *why* AI makes choices
6. **Check edge cases** - Test with no targets, no mana, etc.
7. **Verify no regression** - Ensure fix doesn't break other scenarios
8. **Document changes** - Note what was improved and why

## Troubleshooting

**Improvement doesn't work:**
- Verify code was actually changed (read file after edit)
- Check if conditions are reachable
- Add logging: `System.out.println("AI: " + decision);`
- Test in isolation with simple scenario

**Breaks other scenarios:**
- Review conditions - too strict?
- Check if helper methods changed behavior
- Add tests for previous working scenarios
- Consider using AI profiles for different behaviors

**Test still fails:**
- Debug step-by-step what AI sees
- Log all conditions and checks
- Verify game state is as expected
- Check if other AI code interferes

## Success Criteria

- [ ] Problem behavior identified and documented
- [ ] Test created demonstrating the issue
- [ ] Root cause found in AI code
- [ ] Improvement implemented with clear logic
- [ ] Test now passes
- [ ] Edge cases tested
- [ ] No regression in other scenarios
- [ ] Code commented with reasoning
- [ ] Changes documented

## Example Output

After improving the AI, report:
```
✅ AI Improvement Complete: DestroyAi Targeting Priority

File: forge-ai/src/main/java/forge/ai/ability/DestroyAi.java

Problem Fixed:
❌ Before: AI targeted 1/1 creatures over 6/6 threats
✅ After: AI prioritizes creatures by threat level

Changes Made:
1. Added threat evaluation filter (lines 145-155)
2. Implemented prioritizeCreaturesWorthRemovingNow
3. Added special handling for indestructible/hexproof
4. Improved timing logic to hold for better targets

Tests Added:
✓ testAiTargetsBiggestCreature - Verifies threat priority
✓ testAiTargetsHexproofCreature - Special keyword handling
✓ testAiHoldsRemovalForBetterTarget - Timing logic

Test Results:
All tests passing ✓

Next Steps:
- Monitor AI behavior in real games
- Consider adding similar logic to ExileAi
- Update documentation with new patterns
```
