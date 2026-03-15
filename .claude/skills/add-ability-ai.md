# Skill: add-ability-ai

## Purpose
Creates a new AI implementation for a card ability type (ApiType) in the Forge AI system.

## When to Use
- Adding AI for a new ApiType that doesn't have AI yet
- Card abilities aren't being played by AI
- Implementing AI for custom/new mechanics
- AI uses `AlwaysPlayAi` or `CannotPlayAi` placeholder

## Prerequisites
- ApiType to implement (e.g., `Scry`, `Explore`, `Connive`)
- Understanding of what the ability does
- Knowledge of when AI should/shouldn't use it

## Steps

### 1. Identify the ApiType

Find the ApiType in:
```
forge-game/src/main/java/forge/game/ability/ApiType.java
```

Example ApiTypes:
- `DealDamage` - Deal damage to targets
- `Draw` - Draw cards
- `Destroy` - Destroy permanents
- `Counter` - Counter spells
- `Pump` - Modify P/T
- `Scry` - Scry cards
- `Mill` - Mill cards

### 2. Check Current AI Implementation

Look up current mapping in:
```
forge-ai/src/main/java/forge/ai/SpellApiToAi.java
```

Search for your ApiType:
```java
.put(ApiType.YourType, SomeAi.class)
```

If it maps to `AlwaysPlayAi` or `CannotPlayAi`, it needs a proper implementation.

### 3. Find Similar Ability AI

Search for similar abilities to use as templates:
```bash
# In forge-ai/src/main/java/forge/ai/ability/
grep -l "extends SpellAbilityAi" *.java | head -5
```

**Good templates by complexity:**
- **Simple**: `DrawAi.java` - straightforward decision
- **Medium**: `CounterAi.java` - conditional with targeting
- **Complex**: `DestroyAi.java` - sophisticated targeting priority

Read 2-3 similar AI files to understand patterns.

### 4. Create New AI Class File

**Location**: `forge-ai/src/main/java/forge/ai/ability/{ApiType}Ai.java`

**Naming**: `{ApiType}Ai.java` - e.g., `ScryAi.java`, `ExploreAi.java`

**Template:**
```java
package forge.ai.ability;

import forge.ai.*;
import forge.game.ability.AbilityUtils;
import forge.game.card.Card;
import forge.game.player.Player;
import forge.game.spellability.SpellAbility;

public class YourAbilityAi extends SpellAbilityAi {

    @Override
    protected AiAbilityDecision checkApiLogic(Player ai, SpellAbility sa) {
        // Main decision logic here

        // Check targeting if needed
        if (sa.usesTargeting() && !targetAI(ai, sa, false)) {
            return new AiAbilityDecision(0, AiPlayDecision.TargetingFailed);
        }

        // Check if beneficial to play
        if (!shouldPlay(ai, sa)) {
            return new AiAbilityDecision(0, AiPlayDecision.CantPlayAi);
        }

        return new AiAbilityDecision(100, AiPlayDecision.WillPlay);
    }

    // Override other methods as needed
    @Override
    protected AiAbilityDecision doTriggerNoCost(Player ai, SpellAbility sa, boolean mandatory) {
        if (mandatory) {
            return new AiAbilityDecision(100, AiPlayDecision.MandatoryPlay);
        }
        return checkApiLogic(ai, sa);
    }
}
```

### 5. Implement Main Decision Logic (checkApiLogic)

This is the core method that decides if AI should play the ability.

**Structure:**
```java
@Override
protected AiAbilityDecision checkApiLogic(Player ai, SpellAbility sa) {
    final Card source = sa.getHostCard();
    final Game game = ai.getGame();

    // 1. Check if ability can target (if targeting required)
    if (sa.usesTargeting() && !targetAI(ai, sa, false)) {
        return new AiAbilityDecision(0, AiPlayDecision.TargetingFailed);
    }

    // 2. Check if this is the right time to play
    if (!isGoodTiming(ai, sa)) {
        return new AiAbilityDecision(0, AiPlayDecision.WaitForMain2);
    }

    // 3. Check if playing is beneficial
    if (!isBeneficial(ai, sa)) {
        return new AiAbilityDecision(0, AiPlayDecision.CantPlayAi);
    }

    // 4. Return positive decision
    return new AiAbilityDecision(100, AiPlayDecision.WillPlay);
}
```

**Common checks:**
```java
// Don't play in combat if you need blockers
if (ComputerUtil.waitForBlocking(sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForCombat);
}

// Check if it's worth the cost
if (!ComputerUtilCost.checkLifeCost(ai, cost, source, 4, sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.CostNotAcceptable);
}

// Check if tapping creatures is safe
if (ComputerUtil.waitForBlocking(sa)) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForCombat);
}
```

### 6. Implement Targeting Logic (if needed)

If ability targets, implement `targetAI()` method:

```java
private boolean targetAI(Player ai, SpellAbility sa, boolean mandatory) {
    final Card source = sa.getHostCard();

    // Get all potential targets
    CardCollection list = CardLists.getTargetableCards(
        ai.getOpponents().getCardsIn(ZoneType.Battlefield), sa);

    // Filter to good targets
    list = CardLists.filter(list, new Predicate<Card>() {
        @Override
        public boolean apply(Card c) {
            // Return true if c is a good target
            return isGoodTarget(c, ai);
        }
    });

    // Prioritize targets
    list = ComputerUtilCard.prioritizeCreaturesWorthRemovingNow(ai, list, false);

    if (list.isEmpty()) {
        return mandatory; // Only target if mandatory
    }

    // Choose best target
    Card target = list.get(0);
    sa.resetTargets();
    sa.getTargets().add(target);

    return true;
}
```

**Targeting helpers:**
```java
// Get creatures owned by opponents
CardLists.filterControlledBy(list, ai.getOpponents())

// Filter by characteristic
CardLists.filter(list, CardPredicates.hasGreatestPower(list))

// Prioritize threats
ComputerUtilCard.prioritizeCreaturesWorthRemovingNow(ai, list, false)

// Sort by evaluation
ComputerUtilCard.getBestCreatureAI(list)
```

### 7. Implement Phase Restrictions (if needed)

Control when AI can play this ability:

```java
@Override
protected boolean checkPhaseRestrictions(Player ai, SpellAbility sa, PhaseHandler ph) {
    // Don't play before Main 2 unless instant speed
    if (ph.getPhase().isBefore(PhaseType.MAIN2) && !sa.hasParam("ActivationPhases")) {
        return false;
    }

    // Or: Only play during combat
    if (!ph.inCombat()) {
        return false;
    }

    // Or: Only on opponent's turn
    if (ph.isPlayerTurn(ai)) {
        return false;
    }

    return super.checkPhaseRestrictions(ai, sa, ph);
}
```

### 8. Implement Triggered Ability Logic

For abilities that can be triggered:

```java
@Override
protected AiAbilityDecision doTriggerNoCost(Player ai, SpellAbility sa, boolean mandatory) {
    if (mandatory) {
        // Must do it, just make it work
        targetAI(ai, sa, true);
        return new AiAbilityDecision(100, AiPlayDecision.MandatoryPlay);
    }

    // Optional trigger, decide if beneficial
    return checkApiLogic(ai, sa);
}
```

### 9. Handle Special Cases

Use `checkAiLogic()` for special AILogic parameters:

```java
@Override
protected boolean checkAiLogic(Player ai, SpellAbility sa, String aiLogic) {
    if ("Never".equals(aiLogic)) {
        return false;
    }
    if ("Always".equals(aiLogic)) {
        return true;
    }
    // Add custom logic handling
    if ("OnlyWhenLowLife".equals(aiLogic)) {
        return ai.getLife() < 10;
    }

    return super.checkAiLogic(ai, sa, aiLogic);
}
```

### 10. Register in SpellApiToAi

Add mapping in `forge-ai/src/main/java/forge/ai/SpellApiToAi.java`:

Find the appropriate place (alphabetical order) and add:
```java
.put(ApiType.YourAbility, YourAbilityAi.class)
```

Example:
```java
.put(ApiType.Scry, ScryAi.class)
.put(ApiType.Seek, SeekAi.class)
.put(ApiType.SetState, SetStateAi.class)
```

### 11. Test the AI

Create a test to verify AI behavior:
```java
@Test
public void aiUsesAbilityCorrectly() {
    Game game = initAndCreateGame();
    Player ai = game.getPlayers().get(1);

    Card card = addCard("Card With Ability", ai);
    ai.addMana("mana needed");

    game.getPhaseHandler().devModeSet(PhaseType.MAIN1, ai);
    playUntilPhase(game, PhaseType.MAIN2);

    // Verify AI used the ability
    AssertJUnit.assertTrue("AI should have activated ability",
        /* check result */);
}
```

## Decision Making Guidelines

### When AI Should Play Abilities

**Card Advantage:**
- Drawing cards (almost always good)
- Recurring cards from graveyard
- Tutoring for specific cards

**Board Control:**
- Removing opponent's threats
- Protecting own threats
- Clearing blockers before combat

**Resource Generation:**
- Mana acceleration
- Treasure/energy generation
- Land fetching

**Win Conditions:**
- Lethal damage possible
- Inevitable win (lock pieces)
- Combo pieces

### When AI Should NOT Play Abilities

**Bad Timing:**
- Using removal on small threats when bigger ones exist
- Playing combat tricks outside of combat
- Tapping blockers before opponent's combat

**Insufficient Value:**
- High cost for minimal benefit
- Better options available
- Saving for more impactful play

**Resource Constraints:**
- Can't afford the cost safely
- Need resources for other plays
- Would die from life payment

## Common Patterns by Ability Type

### Card Draw / Selection (Draw, Scry, Surveil)
```java
// Almost always play
// Check: Don't mill yourself out
// Timing: Main phase preferred
return new AiAbilityDecision(100, AiPlayDecision.CardAdvantage);
```

### Removal (Destroy, Exile, Bounce)
```java
// Target prioritization crucial
// Check: Is target worth the removal?
// Timing: Before combat if clearing attackers/blockers
if (!ComputerUtilCard.useRemovalNow(sa, choice, 0, ZoneType.Graveyard)) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForMain2);
}
return new AiAbilityDecision(100, AiPlayDecision.Removal);
```

### Pump Effects (Pump, Counter effects)
```java
// Only during combat or for survival
// Check: Does keyword/size matter now?
// Timing: Combat phase only
if (!game.getPhaseHandler().inCombat()) {
    return new AiAbilityDecision(0, AiPlayDecision.WaitForCombat);
}
return new AiAbilityDecision(100, AiPlayDecision.ImpactCombat);
```

### Mana Abilities (Mana, Treasure)
```java
// Play to enable other spells
// Check: Do we need mana now?
// Timing: Automatically handled (mana abilities don't use stack)
```

## Troubleshooting

**AI never plays the ability:**
- Check registration in SpellApiToAi
- Verify checkApiLogic returns positive decision
- Check phase restrictions aren't too restrictive
- Add logging: `System.out.println("AI decision: " + decision);`

**AI plays ability at wrong time:**
- Review checkPhaseRestrictions implementation
- Check timing logic in checkApiLogic
- Add conditions for combat/main phase

**AI makes bad targeting choices:**
- Review targetAI prioritization
- Use better filtering predicates
- Check ComputerUtilCard helper methods

**AI doesn't handle mandatory correctly:**
- Implement doTriggerNoCost properly
- Check for mandatory flag
- Ensure targetAI works with mandatory=true

## Best Practices

1. **Start simple** - Basic "should I play this" logic first
2. **Use helpers** - Leverage ComputerUtil* classes extensively
3. **Test edge cases** - Low life, no mana, no targets
4. **Document decisions** - Comment why AI makes choices
5. **Return descriptive reasons** - Use appropriate AiPlayDecision enum
6. **Handle mandatory** - Always implement doTriggerNoCost
7. **Phase awareness** - Consider when ability should be used
8. **Cost evaluation** - Check if cost is acceptable

## Success Criteria

- [ ] AI class created in correct location
- [ ] Extends SpellAbilityAi or appropriate base
- [ ] checkApiLogic implemented with clear logic
- [ ] Targeting logic implemented (if needed)
- [ ] Phase restrictions appropriate
- [ ] Registered in SpellApiToAi
- [ ] Tested with actual cards
- [ ] AI makes reasonable decisions
- [ ] Edge cases handled (no targets, no mana, etc.)

## Example Output

After creating the AI, report:
```
✅ Ability AI Created: ScryAi

Location: forge-ai/src/main/java/forge/ai/ability/ScryAi.java

Implementation Details:
- Main Logic: AI scries to improve card quality
- Targeting: Not applicable (targets own library)
- Timing: Main phase preferred, avoids tapping during combat
- Special Handling: Checks for evasion keywords in scry logic

Registered in SpellApiToAi:
- ApiType.Scry → ScryAi.class

Decision Logic:
✓ AI plays scry abilities to filter draws
✓ AI doesn't scry when would leave shields down
✓ AI values scry higher when looking for specific cards

Next Steps:
- Create test to verify AI uses scry correctly
- Test with various scry values (1, 2, X)
- Consider adding card evaluation for scry choices
```
