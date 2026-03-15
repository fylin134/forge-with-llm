# Agentic LLM Commander AI - Prior Art Research Session

**Date**: 2026-01-02
**Session Goal**: Comprehensive prior art survey for implementing agentic AI player in Forge Commander
**Approach**: Hybrid agent architecture (structured workflow with optional tool calling)

---

## Research Questions

1. Find examples of LLM agents playing complex strategic games
2. Research game state representation techniques for LLM prompts
3. Understand agentic frameworks and tool-calling patterns
4. Identify transferable patterns for MTG Commander implementation

---

## Section 1: Prior Art Overview - Game-Playing LLM Agents

### Current State of the Field

**Reality Check**: Despite recent advances, LLM-based game agents often perform subpar without specialized adjustments. Modern LLMs still fail to play even Tic-Tac-Toe perfectly. However, they exhibit **superior long-term planning** without specialized training—exactly what Commander needs.

### Key Research Streams

#### 1. Collectible Card Games (Most Relevant)

**UrzaGPT** (arXiv:2508.08382):
- LoRA-tuned Llama-3-8B for MTG drafting
- Achieved 66% accuracy approaching domain-specific models (68%)
- Used minimal training data (1M picks, 10k training steps)
- Surprising finding: Card names alone performed better than full card text

**MTG Dynamic Difficulty Adjustment** (ScienceDirect 2025):
- GPT-4o playing simplified MTG
- Achieved 50% win rate balancing through difficulty adjustment
- Framework: LLM-MTG-DDA

**Slay the Spire LLM Agents** (ACM FDG 2024):
- Multiple architecture patterns tested
- Modular agents showed best performance
- Chain-of-thought prompting with DAG representation
- Anonymizing card names (random 6-char strings) improved performance

#### 2. Other Strategic Games

**Chess**:
- ChessGPT (fine-tuned GPT2) demonstrates plausible strategies
- Understands classic openings but not perfect play
- LLMs applied as agents for Chess and Go

**Poker**:
- Multiple 2024 papers examined Texas Hold'em
- Even advanced LLMs like ChatGPT largely ineffective
- Incomplete information games remain challenging

**Board Games**:
- Game Reasoning Arena shows larger models adapt better
- Smaller models lock into fixed strategies early
- Adaptive reasoning patterns emerge with scale

### Critical Insight: Challenge-Centered Taxonomy

Research identifies six major game genres with dominant agent requirements:

For **MTG Commander** specifically:
- Multi-agent social reasoning (who's winning, who to attack)
- Long-term strategic planning (not micro-optimized moves)
- Partial information handling (hidden hands, unknown deck contents)
- Political decision-making (temporary alliances, threat assessment)

---

## Section 2: Architecture Patterns from Research

### Unified Reference Architecture

The comprehensive survey (arXiv:2404.02039v4) synthesizes research into a standard architecture with three interconnected components:

```
┌─────────────────────────────────────┐
│         Central LLM (Brain)         │
└──────────┬──────────┬───────────────┘
           │          │
    ┌──────▼────┐  ┌──▼──────┐  ┌──────────┐
    │  Memory   │  │Reasoning│  │Perception│
    │  System   │  │ Engine  │  │& Action  │
    └───────────┘  └─────────┘  └──────────┘
```

**Memory**: Tracks game history, learned patterns, previous decisions
**Reasoning**: Strategic planning and decision-making logic
**Perception-Action**: Interfaces with game environment (reads state, executes moves)

### The ReAct Pattern (Most Influential)

**ReAct** (Reasoning and Acting) - arXiv:2210.03629:
- Interleaves reasoning traces with task-specific actions
- Creates feedback loop: Think → Act → Observe → Think...
- Overcomes hallucination by grounding in real observations
- Outperformed imitation learning by 34% absolute in interactive environments

**Key Advantages for Games:**
- Reasoning traces help induce, track, and update action plans
- Actions gather information from environment (game state)
- Handles exceptions through reasoning
- Enables tool use for external knowledge

**Comparison to Chain-of-Thought:**
- CoT lacks external world access
- CoT prone to fact hallucination and error propagation
- ReAct addresses both through environment interaction

### Modular vs Monolithic Architectures

Slay the Spire study (OpenReview 2024) compared 5 architectures:

| Architecture | Description | Performance |
|-------------|-------------|-------------|
| **Monolithic** | Single LLM handles everything | Baseline (poor) |
| **+ Memory** | Monolithic + action history | Slight improvement |
| **Heuristic Baseline** | Rule-based (no LLM) | Fast, predictable |
| **Hybrid** ⭐ | Heuristics for navigation, LLM for combat | Significantly better |
| **Fully Modular** ⭐⭐ | Context-specific prompts per situation | Best HP preservation |

**Key Finding**: Task decomposition architectures (hybrid and modular) demonstrated significantly superior performance through:
- Specialization via tailored, situational prompts
- Separating strategic decisions from tactical execution
- Strategic stability and better resource management

### Three-Stage Pipeline (Standard Pattern)

Most successful agents follow this flow:

1. **Perception**: Parse game state into structured format
2. **Reasoning**: LLM processes state + memory → generates plan
3. **Action**: Execute decision and update memory

---

## Section 3: Memory Systems for Game Agents

### Four Types of Memory

Research identifies four distinct memory systems used in LLM game agents:

#### 1. Working Memory (Short-term)
- **What**: LLM context window with prompt template and variables
- **Implementation**: Structured prompt with current game state
- **Constraints**: Limited by context window (even with 200k tokens)
- **Usage**: Current turn state, immediate decision context

#### 2. Episodic Memory
- **What**: Stores specific past experiences and events
- **Implementation**: RAG-like systems on conversation/game histories
- **Storage**: Ground truth of all actions, outputs, and reasoning
- **Usage**: Retrieved during planning to support reasoning
- **Example**: "Last time I attacked Player 2, Player 3 retaliated"

#### 3. Semantic Memory
- **What**: Generalized knowledge and learned patterns
- **Implementation**: Vector databases for quick retrieval
- **Storage**: Abstracted skills, rules, successful strategies
- **Usage**: Query before attempting generation
- **Example**: "When ahead on board, focus on protecting lead"

#### 4. Procedural Memory
- **What**: Knowledge about how to do things
- **Implementation**: Functions, algorithms, executable code
- **Usage**: In-game actions, skill library
- **Example**: Voyager's Minecraft skill library of JavaScript code

### Game-Specific Memory Applications

**RAG for Game Knowledge**:
- Leverage game manuals and rules as semantic memory
- Affects policy decisions with authoritative knowledge

**Skill Libraries**:
- Procedural memory containing executable actions
- Example: Voyager agent with JavaScript skill library for Minecraft

**Position Memory**:
- Recent paper (arXiv:2502.06975): "Episodic Memory is the Missing Piece for Long-Term LLM Agents"
- Enables both fast and slow learning
- Critical for agents operating over extended timescales

### Memory Architecture Considerations

**Multi-agent Systems**:
- Working memory constraints necessitate external memory architectures
- Support both episodic recall AND semantic abstraction
- Shared semantic memory for multi-agent coordination

---

## Section 4: Game State Representation Techniques

### Three Classes of Game State Representation

Research identifies three general approaches:

**Class A**: States and actions compactly represented as abstract tokens
- Example: Chess (FEN notation), Go (board coordinates)
- Low token count, efficient for LLMs

**Class B**: Natural language is primary input/output
- Example: Text adventure games, social deduction games
- Native LLM strength

**Class C**: External API control
- Example: Complex games with rich state spaces
- State-to-observation transformation required

### Prompt Structure (OpenSpiel Framework Pattern)

Successful implementations use hierarchical prompt strategy:

**Base Prompt** (all games):
- Generic game framework information
- Action space description
- State representation format

**Game-Specific Extensions**:
- Private information (player's hidden state)
- Public information (visible game state)
- Move history or summary
- Legal actions with contextualized labels

**Example from Kuhn Poker**:
```
Private Information: Your card: [CARD]
Public Information:
  - Move number: [N]
  - Betting history: [SEQUENCE]
  - Pot size: [AMOUNT]
Legal Actions: [BET/FOLD/CHECK with contextual descriptions]
```

### State Representation Framework (3 Axes)

Unifying framework characterizes prompts along:

1. **Action Informativeness**: How much context about actions
2. **Reward Informativeness**: Feedback about action outcomes
3. **Prompting Style**: Natural language compression vs raw data

**Key Finding**: Agents perform better with appropriate natural language summarizations rather than full raw history. Simplification and structure help reasoning.

### Serialization Findings

**From UrzaGPT (MTG Drafting)**:
- Tested card names vs full card text
- Card names alone (concise) outperformed full text
- Full text: 100-300 tokens per card
- Likely explanation: Reduces noise, forces reasoning about game state

**From Slay the Spire**:
- Game state includes: mana, turn number, HP, block, status effects
- Both player and enemy states represented
- Intention data (enemy planned actions) improves performance
- Similar to information given to human players

**Card Anonymization**:
- Slay the Spire: Replacing card names with random 6-char strings improved performance
- Forces agent to rely on card effects rather than memorized card names
- Prevents over-reliance on training data biases

---

## Section 5: Tool Use & Hybrid Agent Patterns

### Agentic AI Core Concepts

**Definition**: Autonomous, goal-driven AI systems that can:
- Act independently
- Adapt in real time
- Make complex decisions based on context
- Pursue goals with limited supervision

**Key Components**:
1. **LLM Backbone**: Core reasoning engine
2. **Memory**: Short-term (context) and long-term (external)
3. **Planning**: Goal-setting and strategy formulation
4. **Tool Interface**: Ability to call functions and APIs

### Decision-Making Cycle

Standard agentic workflow:

1. **Perception**: Gather information from environment
2. **Reasoning**: LLM analyzes data, understands context
3. **Planning**: Formulate strategy, set goals, break into steps
4. **Action**: Execute plan, interact with systems
5. **Reflection**: Learn from results, adjust future plans

### Tool Calling Frameworks

**Popular Frameworks**:
- **LangChain**: Modular workflow design, state maintenance, API calls
- **LangGraph**: Visual graph-based workflow design (extends LangChain)
- **CrewAI**: Open-source multi-agent systems
- **Letta**: Extended agent capabilities
- **Native APIs**: OpenAI, Anthropic (Claude), Cohere function calling

### Claude API Tool Use Best Practices

**Recommended Approach - Tool Runner** (Beta):
- Out-of-the-box solution for tool use
- Automatically handles tool calls, results, conversation management
- Available in Python, TypeScript, Ruby SDKs

**Model Selection**:
- Claude Sonnet 4.5 or Opus 4.1 recommended for complex tools
- Better handling of multiple tools and ambiguous queries
- Seeks clarification when needed

**Parallel Tool Calling**:
- Execute independent tool calls simultaneously
- Prioritize parallel over sequential when no dependencies

**Explicit Instructions for Claude 4.5**:
- Benefits from explicit direction to use specific tools
- Example: "By default, implement changes rather than only suggesting them"
- Makes Claude more proactive

**Advanced Features (2025 Beta)**:
- **Tool Search Tool**: Dynamically discover tools
- **Programmatic Tool Calling**: Invoke tools in code execution environments
- **Tool Use Examples**: Demonstrate effective tool usage patterns

**Error Handling**:
- Send clear, informative error messages in tool_result block
- Helps Claude understand failures and retry intelligently

### Hybrid Architecture Pattern

**For Game Playing**:
- **Structured workflow** for core decisions (predictable, debuggable)
- **Optional tool calling** for deeper analysis when needed
- **Balance**: Control (deterministic flow) + Flexibility (adaptive reasoning)

**Example for Commander**:
```
Structured: Priority phase → Analyze threats → Choose target
Tools available:
  - evaluate_combat_outcome(attacker, defender)
  - calculate_lethal_paths(player)
  - assess_board_state_complexity()

Agent decides: Use tools when uncertainty is high
```

---

## Key Findings for Forge Implementation

### What Works Well

1. **Modular architectures** outperform monolithic approaches
2. **Hybrid systems** (LLM strategy + rule-based execution) show best results
3. **ReAct pattern** with interleaved reasoning and acting
4. **Concise state representation** (card names > full text)
5. **Natural language summarization** > raw data dumps
6. **Episodic memory** for learning from past games
7. **Task-specific prompts** for different game situations

### What Doesn't Work

1. **Zero-shot performance** on small models (7-8B) fails completely
2. **Over-information** (full card text) degrades performance
3. **Monolithic LLMs** handling everything at once
4. **Complete autonomy** without structure (unpredictable, unstable)
5. **Incomplete information games** remain challenging (poker results were poor)

### Transferable Patterns for Commander

**From Slay the Spire**:
- Modular prompts per situation (threat assessment, combat, removal decisions)
- Include "intention data" (what opponents plan to do)
- Context-specific specialization

**From UrzaGPT**:
- LoRA fine-tuning can achieve good results with minimal data
- Card names sufficient for reasoning
- Parameter efficiency enables quick iteration

**From ReAct**:
- Tool use for specific calculations (combat simulation, threat evaluation)
- Reasoning traces for explainability and debugging
- Observation feedback loop grounds decisions

### Recommended Architecture for Forge

**Hybrid Modular Agent**:

```
Phase: Priority (Main Phase)
├─> Situation: Multiple opponents, complex board
├─> Structured Flow:
│   1. Perception: Parse game state (concise representation)
│   2. Threat Assessment (LLM with context-specific prompt)
│   3. Strategic Planning (LLM: who to target, why)
│   4. Tactical Execution (Existing Forge AI)
│
└─> Optional Tools (LLM decides when to use):
    - simulate_combat(attacker, defender)
    - calculate_lethal_range(player)
    - evaluate_removal_targets()
    - assess_political_standing()
```

**Memory System**:
- **Working**: Current game state in prompt (concise format)
- **Episodic**: Key decisions and outcomes this game
- **Semantic**: Learned strategies across games (future enhancement)
- **Procedural**: Forge's existing AI for execution

---

## Research Sources

### Key Papers

1. [UrzaGPT: LoRA-Tuned LLMs for MTG Card Selection](https://arxiv.org/html/2508.08382v1)
2. [Survey on Large Language Model-Based Game Agents](https://arxiv.org/html/2404.02039v4)
3. [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
4. [Language-Driven Play: LLM Agents in Slay the Spire](https://dl.acm.org/doi/fullHtml/10.1145/3649921.3650013)
5. [Modular and Hybrid Frameworks for LLM Agents in Strategy Games](https://openreview.net/forum?id=gC3D2ESSyK)
6. [Position: Episodic Memory is the Missing Piece for Long-Term LLM Agents](https://arxiv.org/pdf/2502.06975)
7. [Dynamic Difficulty Adjustment using LLM in Magic: The Gathering](https://www.sciencedirect.com/science/article/pii/S1875952125000771)

### GitHub Resources

- [Awesome LLM Game Agent Papers](https://github.com/git-disl/awesome-LLM-game-agent-papers)
- [LLM Agents Papers Collection](https://github.com/AGI-Edgerunners/LLM-Agents-Papers)
- [Agent Memory Paper List](https://github.com/Shichun-Liu/Agent-Memory-Paper-List)

### Documentation & Guides

- [ReAct Prompting Guide](https://www.promptingguide.ai/techniques/react)
- [Claude API: Tool Use Implementation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)
- [Claude 4.5 Prompt Engineering Best Practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices)
- [IBM: What is Agentic AI](https://www.ibm.com/think/topics/agentic-ai)

### Articles & Blogs

- [Jake Boggs: Large Language Models for Magic: the Gathering](https://boggs.tech/posts/large-language-models-for-magic-the-gathering/)
- [Game Reasoning Arena: How LLMs Think, Strategize, and Compete](https://laion.ai/blog/reasoning_game_arena_blog_post/)
- [AI Agents: Tool Calling and Reasoning in Generative AI](https://towardsdatascience.com/ai-agents-the-intersection-of-tool-calling-and-reasoning-in-generative-ai-ff268eece443/)
- [Memory: The Secret Sauce of AI Agents](https://www.decodingai.com/p/memory-the-secret-sauce-of-ai-agents)

---

## Section 6: MTG-Specific Implementations

Three significant MTG + LLM projects provide direct lessons for Forge implementation:

### 1. UrzaGPT (2025) - Draft Card Selection

**What they built**: LoRA fine-tuned Llama-3-8B for MTG draft picks

**Game State Representation**:
```
My pool so far: [LIST OF CARDS IN POOL]
Current pack: [LIST OF CARDS IN PACK]
```

**Key Findings**:

| Approach | Result |
|----------|--------|
| Card names only | **Better** (concise, ~5 tokens/card) |
| Full card text | Worse (100-300 tokens/card, noisy) |
| Zero-shot GPT-4o | 43% accuracy |
| Fine-tuned Llama-3-8B | **66% accuracy** |
| Domain-specific baseline | 68% accuracy |

**Why card names beat full text**: The LLM already has MTG knowledge from pretraining. Full text adds noise and bloats context. Card names trigger latent knowledge without overwhelming the prompt.

**Implication for Commander**: Don't serialize entire card text. Use card names + key stats (power/toughness, keywords, controller).

### 2. Jake Boggs' MTG LLM Work (2024)

**What they built**: Fine-tuned Llama 3 8B for rules questions, card interactions

**Dataset approach**:
- 80,000+ QA pairs generated synthetically
- Categories: card descriptions, rules questions, card interactions
- Sources: MTGJSON, Commander Spellbook (authoritative)
- Used GPT-3.5 to reformat data into QA format

**Results**: Only **10.5% improvement** (1.62 → 1.79 on 5-point scale)

**Why it struggled**:

1. **Domain complexity**: 27,000+ unique cards, 300-page rulebook
2. **Tokenization gaps**: Mana symbols ({W}, {U}, {T}) underrepresented in pretraining
3. **Rule interactions**: Emergent complexity from card combinations

**Key Lesson**: Fine-tuning alone isn't enough for MTG reasoning. The game's complexity requires **structured approaches** (tools, decomposition) rather than hoping the LLM "gets it."

### 3. LLM-MTG-DDA (2025) - Dynamic Difficulty Adjustment

**What they built**: GPT-4o playing simplified MTG to balance game difficulty

**Architecture**:
- LLM acts as player in simplified MTG
- Adjusts play strength to achieve ~50% win rate
- Framework focuses on difficulty balancing, not optimal play

**Result**: Achieved reasonable difficulty balancing with win rates approaching 50%

**Implication**: LLMs can make "reasonable" MTG decisions even without perfect play—good enough for entertaining AI opponents.

### Synthesized Lessons for Commander

**What transfers directly**:

1. **Card names are sufficient** for most reasoning (UrzaGPT finding)
2. **Structured tools beat end-to-end** reasoning (Jake Boggs' lesson)
3. **"Good enough" is achievable** for fun gameplay (LLM-MTG-DDA)
4. **Leverage pretraining knowledge** - models already know MTG

**Commander-specific challenges** (not addressed in prior work):

| Challenge | Why It's Hard | Potential Approach |
|-----------|---------------|-------------------|
| 4-player politics | No prior work on multiplayer MTG | Explicit "who's winning" reasoning in prompt |
| Commander damage tracking | 21 damage from each commander separately | Include in state: "Cmdr dmg: P1→you: 14, P2→you: 3" |
| Long games (50+ turns) | Context window limits | Episodic memory with key events only |
| Hidden information | 3 opponents with unknown hands | Probabilistic reasoning, conservative assumptions |

---

## Section 7: Recommendations for Forge Implementation

### What's Novel: The Problem We're Solving

**What Already Exists**:
- Forge rules engine (complete MTG implementation)
- Forge AI (rule-based, plays MTG reasonably well)
- Multiplayer support (up to 8 players)
- Commander format (fully implemented)

**The Problem**: Existing Forge AI has **critical gaps in multiplayer Commander**:

1. **Random opponent selection** - Above 8 life, AI picks attack targets randomly (`AiAttackController.java:195`)
2. **No "who's winning" awareness** - Can't assess threat levels across 4 players
3. **No politics** - Doesn't reason about alliances, retaliation, or table dynamics
4. **Broken multiplayer simulation** - Hardcoded for 2 players (`GameSimulator.java:226`)

**Result**: Forge plays 4-player Commander as "1v1 but with random opponent selection"

**What We're Building**: An LLM strategic layer that plugs into Forge's existing AI:

```
┌─────────────────────────────────────┐
│  LLM Strategic Advisor (NEW)        │  ← We build this
│  - Threat assessment                │
│  - "Who should I attack?"           │
│  - Political reasoning              │
│  - Commander-specific strategy      │
└──────────────┬──────────────────────┘
               │ Strategic intent
               ▼
┌─────────────────────────────────────┐
│  Existing Forge AI (unchanged)      │  ← Already exists
│  - Card evaluation                  │
│  - Combat math                      │
│  - Spell casting logic              │
└─────────────────────────────────────┘
```

**Why It's Novel**:
- **First LLM integration in Forge** (research confirmed zero existing integrations)
- **Multiplayer strategic reasoning** - Prior MTG+LLM work focused on draft/rules, not gameplay
- **Hybrid architecture** - LLM for strategy, rule-based for tactics (research shows this works best)

### Recommended Architecture: Hybrid Modular Agent

```
┌────────────────────────────────────────────────────────┐
│                   FORGE GAME ENGINE                    │
│            (forge-game, existing rules)                │
└───────────────────────┬────────────────────────────────┘
                        │ Game State
                        ▼
┌────────────────────────────────────────────────────────┐
│              LLM STRATEGIC LAYER (New)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │   Threat    │  │   Combat    │  │   Political   │  │
│  │ Assessment  │  │  Decisions  │  │   Reasoning   │  │
│  │   Module    │  │   Module    │  │    Module     │  │
│  └─────────────┘  └─────────────┘  └───────────────┘  │
│                        │                               │
│              ┌─────────▼─────────┐                    │
│              │   Tool Interface  │ (optional calls)   │
│              │  - simulate_combat│                    │
│              │  - calc_lethal    │                    │
│              │  - eval_removal   │                    │
│              └───────────────────┘                    │
└───────────────────────┬────────────────────────────────┘
                        │ Strategic Intent
                        ▼
┌────────────────────────────────────────────────────────┐
│              FORGE AI TACTICAL LAYER                   │
│         (forge-ai, existing implementation)            │
│   AiController → AiAttackController → ComputerUtil    │
└────────────────────────────────────────────────────────┘
```

**Key principle**: LLM decides **WHAT** to do, Forge AI decides **HOW** to execute.

### Game State Serialization Format

Based on research: **concise > verbose**. Proposed format:

```
GAME STATE - Turn 7, Main Phase 1
You: Player 3 (40 life, 7 cards in hand)
Commander: Prossh, Skyraider of Kher (in command zone, cast 1x)

OPPONENTS:
- P1 (32 life, 3 cards): Atraxa [battlefield], cmdr_dmg_to_you: 6
  Board: 4 creatures (8 total power), Sol Ring, Rhystic Study
  Threat: HIGH (card advantage engine online)

- P2 (18 life, 1 card): Krenko [battlefield], cmdr_dmg_to_you: 0
  Board: 12 goblin tokens, Goblin Bombardment
  Threat: CRITICAL (lethal next turn if unchecked)

- P4 (45 life, 6 cards): Oloro [command zone]
  Board: 2 creatures, Propaganda, Ghostly Prison
  Threat: LOW (pillow fort, not aggressive)

YOUR BOARD:
- Prossh (6/6 commander, summoning sick)
- 6 Kobold tokens (0/1)
- Phyrexian Altar, Skullclamp

YOUR HAND (7 cards):
- Terminate, Cultivate, Beast Within, Diabolic Intent
- Blood Artist, Forest, Swamp

RECENT EVENTS:
- T6: P2 cast Krenko, created 8 goblins
- T5: P1 drew 4 cards off Consecrated Sphinx
- T4: You resolved Prossh

DECISION NEEDED: Priority in Main Phase 1. What do you do?
```

**Key elements**:
- Life totals + card counts (resource awareness)
- Commander damage tracking (21-damage win condition)
- Threat assessment hints (but let LLM reason further)
- Board summary (total power, key permanents only)
- Recent events (episodic memory, last 3-4 turns)
- Hand contents for decision-making

### Integration Points with Forge

Based on existing architecture documentation:

| Decision Point | File | Method | LLM Role |
|---------------|------|--------|----------|
| **Threat Assessment** | `ComputerUtil.java` | `evaluateBoardPosition()` (line 2852) | Replace random selection with LLM ranking |
| **Attack Target** | `AiAttackController.java` | `choosePreferredDefenderPlayer()` (line 195) | LLM picks target player |
| **Combat Declaration** | `AiAttackController.java` | `declareAttackers()` (line 804) | LLM suggests attack strategy |
| **Spell Priority** | `AiController.java` | `chooseSpellAbilityToPlay()` (line 1359) | LLM ranks options (optional, Phase 2) |

**Phase 1 Focus**: Just threat assessment + attack targeting (fixes the "random opponent" problem)

### Tool Definitions (For Hybrid Architecture)

```java
// Tools the LLM can optionally invoke

@Tool("simulate_combat")
CombatResult simulateCombat(Player attacker, Player defender, List<Card> attackers);
// Returns: damage dealt, creatures that would die, blocks expected

@Tool("calculate_lethal")
LethalAnalysis calculateLethal(Player target);
// Returns: turns until lethal, required resources, probability

@Tool("evaluate_removal_targets")
List<RankedTarget> evaluateRemovalTargets(Card removalSpell);
// Returns: ranked list of valid targets with threat scores

@Tool("assess_commander_damage")
CommanderDamageState assessCommanderDamage();
// Returns: damage dealt by/to each commander, proximity to 21
```

**When tools get called**: LLM decides. Prompt includes: "You may use tools for complex calculations. For straightforward decisions, reason directly."

### Prompt Template Structure

**System prompt** (constant):
```
You are an expert Magic: The Gathering Commander player. You make strategic
decisions for a 4-player Commander game.

Your goals:
1. Win the game (reduce all opponents to 0 life or 21 commander damage)
2. Assess threats accurately (who's winning? who's dangerous?)
3. Make politically smart decisions (don't draw aggro unnecessarily)
4. Play to your deck's strengths

You will receive game state and must decide on actions. Think step by step.
When uncertain about complex combat math, use available tools.
```

**Situation-specific prompts** (modular):

```
# THREAT_ASSESSMENT prompt
Given the current board state, rank opponents by threat level (1=highest).
Consider: board presence, cards in hand, commander damage dealt,
inevitability (who wins if game goes long?), and current momentum.

Output format:
THREAT_RANKING:
1. [Player] - [one sentence reason]
2. [Player] - [one sentence reason]
3. [Player] - [one sentence reason]
```

```
# COMBAT_DECISION prompt
You're declaring attackers. Consider:
- Who is the biggest threat right now?
- Can you deal lethal or near-lethal to anyone?
- What blocks are likely? Will you lose valuable creatures?
- Political implications: will attacking P1 cause P2/P3 to retaliate?

Output format:
TARGET: [Player]
ATTACKERS: [list of creatures]
REASONING: [1-2 sentences]
```

### Phased Implementation Plan

**Phase 1: Threat Assessment POC (2-3 weeks)**
- New class: `LlmStrategicAdvisor.java`
- Hook: Replace `ComputerUtil.evaluateBoardPosition()` logic
- Single LLM call per priority pass: "Rank opponents by threat"
- Measure: Does AI attack the right player?

**Phase 2: Combat Decisions (2-3 weeks)**
- Expand to attack declaration
- Add `simulate_combat` tool
- LLM picks attack target + suggests attackers
- Forge AI handles actual creature selection

**Phase 3: Removal Targeting (2 weeks)**
- When casting removal, LLM picks target
- Add `evaluate_removal_targets` tool
- Context: "You're casting Terminate. Valid targets: [list]"

**Phase 4: Full Strategic Layer (4+ weeks)**
- Spell prioritization hints
- Board wipe timing
- Political memory (who attacked whom)
- Cross-game learning (optional)

### Cost Estimation

Using Claude Haiku (~$0.25/1M input, $1.25/1M output):

| Phase | Calls/Turn | Tokens/Call | Cost/Game (50 turns) |
|-------|------------|-------------|---------------------|
| Phase 1 | 1 | ~800 | ~$0.02 |
| Phase 2 | 2 | ~1000 | ~$0.05 |
| Phase 3 | 3 | ~1200 | ~$0.08 |
| Full | 5-8 | ~1500 | ~$0.15-0.25 |

**Cost optimization strategies**:
- Cache threat rankings (valid until board changes significantly)
- Skip LLM call if board state unchanged
- Use Haiku for routine decisions, Sonnet for complex ones

### What We're Building (Concrete Deliverables)

1. **Java ↔ Claude API bridge** - HTTP client in forge-ai to call Claude
2. **Game state serializer** - Convert Forge's `Game` object → concise prompt
3. **Strategic hooks** - Intercept key decision points, ask LLM for guidance
4. **Response parser** - Convert LLM output → Forge AI instructions

---

## Next Steps

### Ready for Implementation:
1. Set up git worktree for development
2. Create detailed implementation plan for Phase 1
3. Add OkHttp dependency to forge-ai/pom.xml
4. Implement `LlmStrategicAdvisor.java` with threat assessment
5. Create game state serializer
6. Hook into `AiAttackController.choosePreferredDefenderPlayer()`
7. Test with command-line simulation

### Future Enhancements (Post-MVP):
- Episodic memory across turns
- Cross-game learning
- UI to display LLM reasoning
- Fine-tuned model for cost reduction
- Multiplayer simulation fix

---

**Session Status**: Complete
**Progress**: Completed all 7 sections of prior art research and design recommendations
**Next Action**: Implementation planning for Phase 1 POC
