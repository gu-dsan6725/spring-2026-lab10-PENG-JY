# Lab 2 Analysis: Multi-Turn Agent Evaluation

## Overall Assessment

The evaluation ran 5 out of 10 scenarios, all of which passed with goal_completed=True. According to metrics.txt, GoalCompletion, ToolUsage, ConversationQuality, and PolicyAdherence all averaged 100%, while TurnEfficiency was the only scorer with variance: average 80%, ranging from 0.60 to 1.00. The pattern across personas was clear — the polite persona resolved issues in an average of 1.5 turns, demanding and neutral both took 2 turns, and the confused persona required 3 turns, which directly brought its TurnEfficiency down to 0.60. Only one scenario (Customer changes shipping address, polite persona) achieved a perfect score across all five metrics. The agent's overall performance is strong, and the main takeaway is that TurnEfficiency is highly sensitive to persona type rather than agent quality.

---

## Single Scenario Deep Dive: Customer wants to return a product

I chose this scenario because it involves two tools in sequence and directly tests the return policy check in PolicyAdherence, making it the most representative of how the agent handles a multi-step resolution.

**Scenario metadata**: neutral persona, return category, 2 turns, 12.7s

### Conversation Flow

The conversation starts at log line 51:

```
Turn 1: user says: I received my order ORD-1002 but the laptop stand is wobbly. I want to return...
```

The user's message is specific and reasonable — they provide both the order ID and the defect reason. The agent immediately calls two tools in sequence. At line 54–55, it first calls `lookup_order('ORD-1002')` and confirms the order status is `delivered`. It then calls `process_return` at lines 56–57:

```
[Tool] process_return: order_id='ORD-1002', reason='defective product - wobbly laptop stand'
[Tool] process_return: return RET-1002 initiated for ORD-1002
```

The return succeeds because ORD-1002 was delivered on 2026-03-14, which is 23 days before the evaluation date (April 6), falling within the 30-day return window. At line 58, the agent responds:

```
Turn 1: agent responds: I'm sorry to hear the laptop stand isn't working properly. I've looked up y...
```

The response initiates return RET-1002, explains the return process, and provides the refund amount ($79.97). The simulated user then asks a follow-up at line 59:

```
Turn 2: user says: Great, thank you. Will I get a refund for the full amount?...
```

This is consistent with a neutral persona — the user is not upset, but wants to confirm one specific detail before closing. At line 61, the agent confirms the full refund. At line 62, the actor sends the stop token:

```
Turn 2: goal completed (actor sent stop token)
```

### Influence of Persona Traits

The neutral persona traits (reasonable, clear, expects-resolution) shaped the conversation in two ways. First, the initial message was well-formed and included the order ID and a clear description of the defect, which allowed the agent to call both tools in the first turn without any back-and-forth clarification. Second, the follow-up question in Turn 2 was practical and brief — the neutral persona did not express frustration or skepticism, just one clarifying question about the refund amount. A demanding persona would likely have pushed back more (e.g., questioning why they need to repack the items), while a confused persona might have asked what a return even is. The neutral setting resulted in a clean, efficient 2-turn resolution.

### Score Analysis

Line 64 of debug.log shows:

```
Scores: GoalCompletion=1.00, ToolUsage=1.00, TurnEfficiency=0.80, ConversationQuality=1.00, PolicyAdherence=1.00
```

- **GoalCompletion: 1.00** — The return was successfully initiated and the user confirmed satisfaction before sending the stop token. This score is fair.

- **ToolUsage: 1.00** — The expected tools were `[lookup_order, process_return]`, and both were called exactly as expected. The scorer uses set-based recall with a small penalty for extra tools, but here there were no extra calls. This score is fair.

- **TurnEfficiency: 0.80** — The formula is `1.0 - (turns - 1) / (max_turns - 1)` = `1.0 - 1/5 = 0.80`. The deduction comes purely from using 2 turns instead of 1. However, the second turn existed only because the user (simulated by the ActorSimulator) asked a follow-up question — the agent answered it correctly and concisely. This is a mild scorer limitation: TurnEfficiency penalizes any turn beyond the first regardless of whether the extra turn was necessary or added value.

- **ConversationQuality: 1.00** — All four heuristic checks pass: the agent's responses were non-empty, contained no error patterns, had above-average length, and the conversation had at least 3 entries. This score is fair.

- **PolicyAdherence: 1.00** — The scorer checks for polite language and, for return-category scenarios, for return policy keywords (e.g., "30-day", "return window", "original packaging"). The agent explicitly mentioned the 30-day return window and original packaging requirement in its Turn 1 response. This score is fair.

Overall, I think the scores accurately reflect the agent's behavior in this scenario. The only questionable point is TurnEfficiency — a score of 0.80 implies the conversation was less than ideal, but the second turn was entirely driven by a reasonable follow-up question from the simulated user. If the goal is to measure agent efficiency, the scorer could be improved by distinguishing agent-initiated turns from user-initiated follow-ups.
