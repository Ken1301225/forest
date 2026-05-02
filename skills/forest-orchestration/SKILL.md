---
name: forest-orchestration
description: Use when serving as the Forest primary agent and receiving an analysis, planning, or solution design question from the user
---

# Forest Orchestration

Coordinate a configurable set of subagents through structured multi-round discussion to produce optimal plans.

## Key Rules

- Subagents have **isolated context** — each dispatch is a fresh session with no memory of prior rounds
- ALL information flows through Forest — you collect, organize, and redistribute results between rounds
- **Maximize parallel dispatch** — independent dispatches go out simultaneously
- Respect the user's chosen **round count** — do not exceed it

---

## Step 1: Configure the Deliberation (via question tool)

<HARD-GATE>
Do NOT dispatch ANY subagents, explore ANY code, or begin ANY analysis until you have received the user's answers to BOTH configuration questions below. The `question` tool MUST be called and answered before Phase 0, Phase A, Phase B, or any other action beyond loading this skill.
</HARD-GATE>

Before any analysis, ask the user two questions in a single `question` call:

### Question A: Select subagents (multiple choice)

```
Question: "Which AI models should participate in the discussion?"
Header: "Select agents"
Options:
  - "DeepSeek V4 Pro" (forest-dsv4)
  - "Kimi K2.6" (forest-kimi)
  - "MiMo V2.5 Pro" (forest-mimo)
  - "Qwen 3.6 Plus" (forest-qwen)
  - "MiniMax M2.7" (forest-minimax)
  - "GLM 5.1" (forest-glm)
multiple: true
```

Recommend 2 agents for a focused discussion, 3 for broader coverage.

### Question B: Select rounds (single choice)

```
Question: "How many discussion rounds?"
Header: "Discussion rounds"
Options:
  - "1 round (quick analysis)"
  - "2 rounds (balanced, recommended)"
  - "3 rounds (deep deliberation)"
multiple: false
```

Store the user's answers:
- `AGENTS`: list of selected subagent names (e.g., ["forest-dsv4", "forest-kimi"])
- `ROUNDS`: the number of rounds (1, 2, or 3)
- `M`: the count of selected agents
- `M_active`: initially equal to `M`. Decrement it whenever a subagent fails permanently (after the single retry in Error Handling). All subsequent cross-validation dispatch counts and consensus checks use `M_active`, not `M`.

---

## Step 2: Phase 0 — Project Reconnaissance

Before dispatching subagents, explore the project yourself:

1. Read `AGENTS.md` or equivalent project guidance if present
2. List the top-level directory (use `read` on project root)
3. Read 1-2 key config files (package.json, pyproject.toml, Cargo.toml, etc.)
4. Build a ~15-line `PROJECT CONTEXT` summary covering:
   - Project type, language, framework
   - Key directories and their purpose
   - Relevant files related to the user's question

**Constrain**: 1-2 minutes or 8 tool calls, whichever comes first.

---

## Step 3: Execute N Rounds of Deliberation

For each round `i` (1 to `ROUNDS`):

### Phase A: Produce/Revise Plans (M parallel dispatches)

Dispatch all `AGENTS` in parallel.

**If round 1:**
```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT FROM PHASE 0]
---

Task: Analyze this problem. Explore the project freely using read, glob, and grep. Produce your initial plan.

Include:
1. Problem analysis — what needs to be solved and why
2. Solution approach — how to solve it, with reasoning
3. Files and changes — specific files, what to change in each
4. Risks and trade-offs — alternatives considered
5. Open questions — what needs clarification

Principles: Minimal code changes, optimal efficiency, evidence-based.
```

**If round > 1:**
```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT FROM PHASE 0]
---

Peer questions about your previous plan:
---
[ALL "Questions for [you]" FROM OTHER AGENTS' PHASE B OUTPUTS]
---

Your previous self-reflections:
---
[YOUR OWN "Improvements to my plan" FROM PHASE B OF ROUND i-1]
---

Your previous plan (for reference):
---
[THIS SUBAGENT'S PLAN FROM ROUND i-1]
---

Task: Revise your plan based on the feedback. Address the questions raised, incorporate valid suggestions, and strengthen your approach.

Include:
1. Revised problem analysis
2. Revised solution approach — explain what changed and why
3. Updated files and changes
4. Updated risks and trade-offs
5. Responses to specific questions raised by others
```

**After dispatch**: Store all M plans verbatim.

---

### Phase B: Cross-Validation (M_active parallel dispatches)

> **Streaming optimization**: You may begin Phase B dispatches as soon as ≥2 Phase A results are available. Do not wait for all Phase A dispatches to complete. Agents who finish Phase A early can begin cross-validating immediately against whatever plans are available at that moment. Agents finishing later will receive all available plans.
>
> This may result in incomplete cross-validation (earlier agents may not have seen later agents' plans when dispatched), which is acceptable — the gap is compensated in Round > 1 when those later agents' feedback reaches the earlier agents during revision. If M_active drops below 2, skip Phase B per Degradation rules.

For each subagent, dispatch with ALL OTHER subagents' plans from Phase A:

**Template for agent X:**
```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT FROM PHASE 0]
---

Below are the latest plans from the other analysts. Your own plan is included for reference.

[FOR EACH OTHER AGENT Y]:
===== [Y'S NAME] PLAN =====
[Y'S FULL PLAN FROM PHASE A]

===== YOUR PLAN (for reference) =====
[YOUR FULL PLAN FROM PHASE A]

Task:
1. Cross-validate the other analysts' plans against the actual project code — verify assumptions, check feasibility
2. Identify strengths from their plans you can incorporate into yours
3. Identify problems, gaps, or errors in their plans
4. Raise questions you want them to address
5. Propose how your plan should improve by absorbing their strengths

Output:
[FOR EACH OTHER AGENT Y]:
### [Y's name] plan validation
(Verify assumptions, identify strengths, gaps, errors in Y's plan)

### Questions for [Y's name]
(Specific questions you want Y to address in the next round)

<!-- The following section appears ONCE, after all per-agent validations above -->
## Improvements to my plan
(How your own plan should improve, drawing from ALL cross-validation insights above. Be concrete about what to add, change, or reconsider.)
```

**After dispatch**: For each subagent, collect:
- All "Questions for [X]" entries (from OTHER subagents — these form the peer feedback for the next round)
- "Improvements to my plan" section (the subagent's own self-reflection — include this in the next round's feedback)
- The subagent's own plan and cross-validation output are stored for Forest Synthesis reference

---

### Consensus Check (after Phase B)

Before proceeding to the next round, evaluate:

- If `M_active` = 1, skip the Consensus Check entirely. Proceed directly to Final Synthesis (a single agent's plan has no cross-validation peers to converge with).

- If all `M_active` subagents' Phase B results show "Improvements to my plan" that are minor (only wording/formatting refinements; no new files, no new approaches, no structural changes recommended) or none, AND no subagent raised blocking concerns about another's plan:
  → **Early consensus reached.** Skip remaining rounds. Proceed to Final Synthesis.

- If substantial disagreements remain:
  → Continue to the next round.

If this was the last round (`i == ROUNDS`), proceed to Final Synthesis regardless.

---

## Final: Forest Synthesis

You now have:
- M subagent plans from each round
- Cross-validation results from each round
- All feedback and responses

Do your own work:

1. **Investigate the codebase yourself** — read relevant files, verify claims and assumptions from the discussion
2. **Weigh consensus against disagreements** — where all agree, that's solid. Where they differ, judge which reasoning is stronger based on your own code investigation
3. **Synthesize the final unified plan** — combine the best insights from all subagents, resolve disagreements with evidence, and produce one coherent plan

**Principles**:
- Minimal code changes — reuse what exists
- Optimal efficiency — simplest approach, shortest path
- Actionable — concrete steps, specific files, clear rationale

The final plan's format is at your discretion, but it must be structured, complete, and ready to hand off for implementation.

---

## Error Handling

### Dispatch Failures

If a subagent dispatch fails (timeout, empty response, unparseable output):
1. Retry the dispatch ONCE with the same prompt
2. If retry fails → mark that subagent as failed for this round
3. Proceed with remaining subagents (n-1 degradation)

### Quality Failures

If a subagent returns coherent but incorrect/malformed output:
- Flag in your notes; do NOT retry (retrying a wrong plan wastes tokens)
- Continue with remaining subagents

### Degradation

- **M_active-1 subagents succeed**: Proceed. Adjust cross-validation: remaining subagents review each other's plans. Use `M_active` (not `M`) for all subsequent dispatch counts and consensus checks.
- **1 subagent in Phase A (M_active == 1)**: Skip cross-validation (Phase B) entirely. Forest must conduct independent investigation and produce the plan.
- **0 subagents succeed in Phase A**: Abort. Tell user: "All subagent dispatches failed. Here's what I found from Phase 0 reconnaissance. Would you like me to retry or produce a simplified analysis?"
- **All Phase A succeeded but all Phase B failed**: Proceed to the next round using Phase A plans only (no cross-validation feedback). In Final Synthesis, note that cross-validation was unavailable and compensate with your own independent code investigation. Do NOT abort.

### Response Validation

Validate responses based on which phase produced them:

**Phase A validation**: Response must contain all expected plan sections: "Problem analysis" (or "Revised problem analysis" for round > 1), "Solution approach" (or "Revised solution approach"), "Files and changes" (or "Updated files and changes"), "Risks and trade-offs", and "Open questions" (or "Responses to specific questions"). Missing sections → treat as dispatch failure (retry once).

**Phase B validation**: Response must contain at least one "plan validation" section per other analyst, a single "Improvements to my plan" section, and "Questions for [Y's name]" entries for each other analyst. Missing required sections → treat as dispatch failure (retry once).

Do NOT apply Phase A validation criteria to Phase B outputs, or vice versa.

### Important

Failed subagents are NOT re-dispatched in later phases. They remain absent for the rest of the deliberation. This preserves the "fresh session, no memory of prior rounds" constraint in the Key Rules.
