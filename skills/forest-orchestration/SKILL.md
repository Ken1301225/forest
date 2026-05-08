---
name: forest-orchestration
description: Use when serving as the Forest primary agent and receiving an analysis, planning, or solution design question from the user
---

# Forest Orchestration

Coordinate a configurable set of subagents through structured multi-round discussion to produce optimal plans.

## Overview of Deliberation Modes

| Mode | Flow | Rounds | Best for |
|------|------|--------|----------|
| **Fast** | 1 round, role-based lenses, Forest merges directly | 1 | Well-understood problems needing multi-perspective validation |
| **Balanced** | 1 round of full plans → Forest extracts disputes → 1 focused dispute discussion | 1 + dispute resolution | Problems with likely consensus on most points, disagreements on a few |
| **Deep** | Full N-round cross-validation of complete plans | 1-3 | Architecturally complex, high-risk decisions |

## Key Rules

- Subagents have **isolated context** — each dispatch is a fresh session with no memory of prior rounds
- ALL information flows through Forest — you collect, organize, and redistribute results between rounds
- **Maximize parallel dispatch** — independent dispatches go out simultaneously
- Respect the user's chosen **mode and round count** — do not exceed them
- **Websearch is a Forest-only capability** — subagents do not search the web; Forest gathers external intelligence and injects it into context

---

## Step 1: Configure the Deliberation (via question tool)

<HARD-GATE>
Do NOT dispatch ANY subagents, explore ANY code, or begin ANY analysis until you have received the user's answers to ALL configuration questions below. The `question` tool MUST be called and answered before Phase 0, Phase A, Phase B, or any other action beyond loading this skill.
</HARD-GATE>

Ask the user three questions in a single `question` call:

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

### Question B: Select deliberation mode (single choice)

```
Question: "Which deliberation mode?"
Header: "Deliberation mode"
Options:
  - "Balanced (Recommended) — full plans, then focused discussion only on disagreements"
  - "Fast — role-based lenses, one round, Forest merges directly"
  - "Deep — full multi-round cross-validation of complete plans"
multiple: false
```

### Question C: Select rounds (single choice, only affects Deep mode)

```
Question: "How many discussion rounds? (Only applies to Deep mode. Fast and Balanced use 1 round.)"
Header: "Discussion rounds"
Options:
  - "1 round"
  - "2 rounds (recommended for Deep)"
  - "3 rounds"
multiple: false
```

Store the user's answers:
- `AGENTS`: list of selected subagent names (e.g., ["forest-dsv4", "forest-kimi"])
- `MODE`: one of "fast", "balanced", "deep"
- `ROUNDS`: the number of rounds for Deep mode (1, 2, or 3; ignored for fast/balanced)
- `M`: the count of selected agents
- `M_active`: initially equal to `M`. Decrement it whenever a subagent fails permanently (after the single retry in Error Handling). All subsequent dispatch counts and consensus checks use `M_active`, not `M`.

---

## Step 2: Phase 0 — Project Reconnaissance

### Phase 0.1: Local Reconnaissance

Before dispatching subagents, explore the project yourself:

1. Read `AGENTS.md` or equivalent project guidance if present
2. List the top-level directory (use `read` on project root)
3. Read 1-2 key config files (package.json, pyproject.toml, Cargo.toml, etc.)
4. Build a ~15-line `PROJECT CONTEXT` summary covering:
   - Project type, language, framework
   - Key directories and their purpose
   - Relevant files related to the user's question

**Constrain**: 1-2 minutes or 8 tool calls, whichever comes first.

### Phase 0.2: Web Augmentation

After local reconnaissance, gather external intelligence to enrich the context available to subagents:

1. Identify key technologies from the project config (language version, framework, 3-5 core dependencies)
2. For each key technology, run 1-2 websearches to gather:
   - Recent best practices, deprecations, or breaking changes
   - Community-recommended patterns relevant to the user's question
3. Append findings to `PROJECT CONTEXT` as an `EXTERNAL INTELLIGENCE` block:

```
PROJECT CONTEXT:
---
[Local reconnaissance summary...]

EXTERNAL INTELLIGENCE:
- React 19: Server Components stable since 19.0, useEffectEvent still experimental
- FastAPI 0.115+: lifespan context managers replace on_event decorator
- SQLAlchemy 2.0: Query API removed, use select() style exclusively
---
```

**Constrain**: max 4 websearches, 30 seconds total. Only search technical facts; never search project names, business logic, or file contents.

### Phase 0.3: Raise Questions Gate

After Phase 0.1 and Phase 0.2, evaluate whether any ambiguities exist that could mislead subagents. Check the following:

| ID | Ambiguity Type | Examples |
|----|---------------|----------|
| A1 | User intent is unclear or ambiguous | "Optimize performance" — latency, throughput, or memory? |
| A2 | Project structure is confusing | Multiple similar directories, no clear entry point |
| A3 | Technology version conflicts | package.json says v18 but code uses v19 APIs |
| A4 | Dependency issues | Missing packages, conflicting versions |
| A5 | Web search results contradict each other | Two authoritative sources recommend different approaches |
| A6 | User question exceeds current codebase scope | "Rebuild auth system" but no auth code exists |
| A7 | Multiple viable paths with no clear user preference | ORM vs raw SQL, monorepo vs separate services |

**Decision rule**:

- **IF any ambiguity (A1-A7) is detected**: HALT. Do NOT proceed to Step 3. Raise questions to the user via the `question` tool. Present all ambiguities together in a single call — do not ask repeatedly. Only proceed after the user has resolved all questions.

  Format for raising questions:
  ```
  Forest detected ambiguities during Phase 0 reconnaissance:

  [For each ambiguity, include:]
  - What I found: [observation]
  - What I'm uncertain about: [specific question]
  - Impact if wrong: [consequence of acting on the wrong assumption]
  ```

- **IF zero ambiguities detected**: Proceed to Step 3. Briefly note: "Phase 0 complete. No ambiguities detected."

- **IF user says "proceed with your best judgment"**: Record the assumptions made. Mark affected decisions as `ASSUMED` in the final plan so the user can revisit them later.

**Re-entry**: After receiving user answers, if the answers introduce new technical uncertainties (e.g., "use library X" but X's compatibility is unknown), run one supplemental websearch. Then proceed directly to Step 3 — do not raise questions a second time unless entirely new scope is revealed.

---

## Step 3: Execute by Mode

All subagent dispatches use the **Common Prompt Template** below, regardless of mode.

### Common Prompt Template

Every subagent prompt MUST include these sections in this exact order:

#### Required Tools Block

```
<REQUIRED-TOOLS>
You MUST use `read`, `glob`, or `grep` to explore the project before writing your plan.
Do NOT write a plan based solely on the project context provided — verify claims against real code.
</REQUIRED-TOOLS>
```

#### Structured Output Format

All Phase A dispatches MUST require the following section markers. Every marker is mandatory. Missing markers → Gatekeeping failure → retry once.

```
[REQ-1] Problem analysis:
(What needs to be solved and why)

[REQ-2] Solution approach:
(How to solve it, with reasoning)

[REQ-3] Files and changes:
(Format: `file_path:line_range — change description`. Must reference real files verified by tool use.)

[REQ-4] Risks and trade-offs:
(Alternatives considered, why they were rejected)

[REQ-5] Open questions:
(What needs clarification from the user or further investigation)

[SELF-VERIFY]
SV-1: I explored the project codebase using read/glob/grep — Tools used: [LIST]
SV-2: I addressed ALL sections [REQ-1] through [REQ-5]
SV-3: [REQ-3] contains specific file:line references backed by code I read
SV-4: My claims are supported by evidence from the codebase, not assumptions
(If any SV is NO, revise your plan NOW before submitting.)
```

For Deep mode Phase A round > 1, replace `[REQ-5]` heading with `[REQ-5] Responses to peer questions` and add peer feedback context as before.

---

### MODE = "fast"

#### Fast Phase A: Role-Based Lens Analysis (M parallel dispatches)

Assign each subagent a specialized lens based on their position in the `AGENTS` list:

| Position | Lens | Focus |
|----------|------|-------|
| 1st agent | **Architect** | System design, data flow, component boundaries, scalability |
| 2nd agent | **Implementer** | Code-level details, specific files/changes, API contracts, edge cases |
| 3rd+ agent | **Auditor** | Risks, failure modes, security, performance, testing gaps |

If M > 3, rotate through the three lenses (4th = Architect, 5th = Implementer, etc.).

Each subagent receives the Common Prompt Template with an added lens instruction:

```
[COMMON PROMPT TEMPLATE]

YOUR LENS: [Architect / Implementer / Auditor]
Focus your analysis through this lens. You are NOT writing a general plan — 
you are providing a specialized perspective. Other analysts will cover the other lenses.
```

#### Fast Gatekeeping

After all M dispatches complete, apply Gatekeeping (see Gatekeeping section below).

#### Fast Synthesis

Forest directly synthesizes the M lens-specific plans into one unified plan. No Phase B or cross-validation. Jump to Step 4.

---

### MODE = "balanced"

#### 3b.1 Phase A — Produce Plans (M parallel dispatches)

Dispatch all `AGENTS` in parallel using the Common Prompt Template.

Prompt (round 1, the only round in Balanced mode):

```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT INCLUDING EXTERNAL INTELLIGENCE]
---

<REQUIRED-TOOLS>
[REQUIRED-TOOLS BLOCK]
</REQUIRED-TOOLS>

Task: Analyze this problem. Explore the project freely using read, glob, and grep. Produce your initial plan.

[REQ-1] Problem analysis:
(What needs to be solved and why)

[REQ-2] Solution approach:
(How to solve it, with reasoning)

[REQ-3] Files and changes:
(Format: `file_path:line_range — change description`. Must reference real files verified by tool use.)

[REQ-4] Risks and trade-offs:
(Alternatives considered, why they were rejected)

[REQ-5] Open questions:
(What needs clarification from the user or further investigation)

[SELF-VERIFY]
SV-1: I explored the project codebase using read/glob/grep — Tools used: [LIST]
SV-2: I addressed ALL sections [REQ-1] through [REQ-5]
SV-3: [REQ-3] contains specific file:line references backed by code I read
SV-4: My claims are supported by evidence from the codebase, not assumptions
(If any SV is NO, revise your plan NOW before submitting.)
```

**After dispatch**: Store all M plans verbatim.

#### 3b.2 Gatekeeping

Apply Gatekeeping checks on all Phase A outputs (see Gatekeeping section below).

#### 3b.3 Extract Decision Matrix

Forest parses all Phase A plans and extracts key decision points into a structured matrix:

```
DECISION                    | [AGENT 1]   | [AGENT 2]   | [AGENT 3]   | STATUS
─────────────────────────────────────────────────────────────────────────────────
[Decision topic 1]          | [Choice]     | [Choice]    | [Choice]    | CONSENSUS / DISPUTED
[Decision topic 2]          | [Choice]     | [Choice]    | [Choice]    | CONSENSUS / DISPUTED
...
```

Rules:
- **CONSENSUS**: All agents agree on the same approach
- **DISPUTED**: 2+ different approaches proposed

Include ALL decisions mentioned by any agent, even trivial ones. The goal is a complete map.

#### 3b.4 Phase B — Dispute-Only Discussion (M parallel dispatches)

If ALL decisions are CONSENSUS, skip Phase B entirely and proceed to Step 4 (Forest Synthesis).

Otherwise, for each subagent, dispatch a focused prompt containing ONLY disputed decisions:

```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

## Disputed Decisions

The group has reached consensus on some decisions but disagrees on others. 
Only the DISPUTED decisions need your analysis below.

[FOR EACH DISPUTED DECISION]:
### [DECISION NAME]
- Your position (from your Phase A plan): [YOUR ANSWER]
- Your original reasoning from Phase A: [YOUR [REQ-2] REASONING FOR THIS DECISION, max 3 sentences]
- Other positions: [OTHER AGENTS' ANSWERS AND THEIR REASONING SNIPPETS, max 2 sentences each]

---

## Task

For each disputed decision above, provide ALL of the following. Every field is mandatory.

[DC-CERTAIN] What is CERTAIN — code evidence that supports your position (1 sentence):
[DC-UNCERTAIN] What is UNCERTAIN — missing information that could change this (1 sentence):

[DP-1] Your refined position (1-2 sentences):
[DP-2] Your strongest argument — the single best reason your approach is correct (1-2 sentences):
[DP-3] One risk or limitation of your approach (1 sentence):
[DP-4] Concession condition — what MEASURABLE evidence would change your mind? (1 sentence):
(Only concede on specific, verifiable criteria — e.g., "if benchmarks show 2x faster" or "if migration guide confirms backward compatibility". NOT: "if the other approach seems better".)

[FOR EACH OTHER AGENT WHO TOOK A DIFFERENT POSITION ON THIS DECISION]:
[DP-REBUT-{AgentName}] One specific flaw or gap in {AgentName}'s position (1 sentence):
(Reference code evidence or logical reasoning. Do NOT give generic criticism.)

## Token Budget
Each field is limited to the sentence count specified above. Be precise and evidence-based.

[SELF-VERIFY]
SV-1: I addressed ALL disputed decisions listed above — YES/NO
SV-2: I provided a rebuttal for every other agent with a different position on each decision — YES/NO
```

**After dispatch**: For each disputed decision, Forest collects all agents' [DC-CERTAIN], [DC-UNCERTAIN], [DP-1] through [DP-4], and all [DP-REBUT-{AgentName}] responses.

#### 3b.5 Phase C — Self-Reflection (M parallel dispatches)

After Phase B, each agent receives the rebuttals directed against their own positions and gets a chance to concede, refine, or defend.

For each subagent X, Forest collects all `[DP-REBUT-{AgentName}]` entries from OTHER agents that targeted X's positions, grouped by disputed decision. Then dispatch:

```
You are an analyst in the Forest deliberation system.

## Rebuttals Against Your Positions

Other analysts have identified flaws in your positions from the dispute discussion. 
Review each rebuttal and respond honestly.

[FOR EACH DISPUTED DECISION WHERE OTHERS REBUTTED YOU]:
### [DECISION NAME]
- Your position: [YOUR DP-1 FROM PHASE B]
- Your strongest argument: [YOUR DP-2 FROM PHASE B]
- Your certainty claim: [YOUR DC-CERTAIN FROM PHASE B]

Rebuttals against you:
  - [Agent Y] says: [DP-REBUT-Y'S SPECIFIC FLAW, 1 sentence]
  - [Agent Z] says: [DP-REBUT-Z'S SPECIFIC FLAW, 1 sentence]

---

For each rebuttal above, respond with EXACTLY ONE of:

[SR-CONCEDE] I concede this point. The rebuttal is correct because: (1 sentence)
[SR-REFINE] I refine my position. The rebuttal is partially valid, so I adjust: (1-2 sentences)
[SR-REJECT] I reject this rebuttal. Counter-evidence: (1 sentence with code reference or logical reasoning)

You may mix responses: concede on one decision, refine on another, reject on a third.
Be honest — conceding a valid point strengthens your credibility, not weakens it.

[SELF-VERIFY]
SV-1: I responded to every rebuttal against my positions — YES/NO
```

**After dispatch**: Forest collects each agent's self-reflection. The combination of Phase B outputs + Phase C self-reflections gives Forest a complete picture:

- If an agent **concedes** ([SR-CONCEDE]): the dispute on that decision is effectively resolved. Forest notes the concession and adopts the opposing position.
- If an agent **refines** ([SR-REFINE]): Forest considers the refined position as the agent's updated stance, potentially narrowing the disagreement.
- If an agent **rejects** ([SR-REJECT]): Forest evaluates the counter-evidence. Strong rejections (with code citations) keep the dispute alive; weak rejections (vague claims) weaken that agent's position.

#### 3b.6 Resolve Disputes

Forest resolves each DISPUTED decision by evaluating:
1. **Concessions from Phase C**: If any agent conceded ([SR-CONCEDE]) on a disputed decision, that dispute is resolved in favor of the opposing position. No further evaluation needed.
2. **Certainty grounding**: How strong is [DC-CERTAIN] — does it cite real code evidence or handwave? Agents with concrete evidence-backed certainty should be weighted higher.
3. **Rebuttal quality from Phase B**: Do [DP-REBUT] entries identify genuine flaws backed by evidence, or are they generic criticism? Strong rebuttals weaken the target position, especially if the target agent could not effectively reject them in Phase C.
4. **Self-reflection quality from Phase C**: Did agents who refined ([SR-REFINE]) produce meaningfully adjusted positions? Did agents who rejected ([SR-REJECT]) provide strong counter-evidence? Honest self-reflection (even conceding) should be weighted positively — an agent that acknowledges valid criticism and adjusts is more trustworthy than one that stubbornly rejects everything.
5. **Strongest argument**: How compelling is [DP-2]? Does it reference specific technical advantages or architectural principles?
6. **Self-awareness**: Whether the agent honestly acknowledged risks in [DP-3] — but this should not unduly reward being uncertain. A strong [DC-CERTAIN] + honest [DP-3] is better than a weak position with no risk acknowledgment.
7. **Measurable concession**: Does [DP-4] from Phase B provide concrete, verifiable conditions? Vague concessions ("if the other approach seems better") indicate weak conviction. Specific concessions ("if benchmarks show 2x faster") indicate strong conviction with realistic boundaries. Cross-check against Phase C: did the agent's [SR] response align with their [DP-4] concession condition?
8. **Evidence chain**: Code evidence from Phase A (which agent's [REQ-3] citations better support their position?) + whether [DC-CERTAIN] claims are verifiable.
9. **Websearch (optional)**: If the dispute hinges on a factual technical claim (e.g., "X library supports Y feature"), Forest MAY run 1 websearch per disputed decision to verify. Constrain: max 3 searches total.

Record the resolution for each dispute. Proceed to Step 4.

---

### MODE = "deep"

For each round `i` (1 to `ROUNDS`):

#### Deep Phase A: Produce/Revise Plans (M parallel dispatches)

**If round 1:**

```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT INCLUDING EXTERNAL INTELLIGENCE]
---

<REQUIRED-TOOLS>
[REQUIRED-TOOLS BLOCK FROM COMMON PROMPT TEMPLATE]
</REQUIRED-TOOLS>

Task: Analyze this problem. Explore the project freely using read, glob, and grep. Produce your initial plan.

[REQ-1] Problem analysis:
(What needs to be solved and why)

[REQ-2] Solution approach:
(How to solve it, with reasoning)

[REQ-3] Files and changes:
(Format: `file_path:line_range — change description`. Must reference real files verified by tool use.)

[REQ-4] Risks and trade-offs:
(Alternatives considered, why they were rejected)

[REQ-5] Open questions:
(What needs clarification from the user or further investigation)

[SELF-VERIFY]
SV-1: I explored the project codebase using read/glob/grep — Tools used: [LIST]
SV-2: I addressed ALL sections [REQ-1] through [REQ-5]
SV-3: [REQ-3] contains specific file:line references backed by code I read
SV-4: My claims are supported by evidence from the codebase, not assumptions
(If any SV is NO, revise your plan NOW before submitting.)
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
[PROJECT CONTEXT INCLUDING EXTERNAL INTELLIGENCE]
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

<REQUIRED-TOOLS>
[REQUIRED-TOOLS BLOCK FROM COMMON PROMPT TEMPLATE]
</REQUIRED-TOOLS>

Task: Revise your plan based on the feedback. Address the questions raised, incorporate valid suggestions, and strengthen your approach.

[REQ-1] Revised problem analysis:
[REQ-2] Revised solution approach — explain what changed and why:
[REQ-3] Updated files and changes (format: `file_path:line_range — change description`):
[REQ-4] Updated risks and trade-offs:
[REQ-5] Responses to specific questions raised by others:

[SELF-VERIFY]
SV-1: I explored the project codebase using read/glob/grep — Tools used: [LIST]
SV-2: I addressed ALL sections [REQ-1] through [REQ-5]
SV-3: [REQ-3] contains specific file:line references backed by code I read
SV-4: My claims are supported by evidence from the codebase, not assumptions
SV-5: I addressed every peer question raised about my previous plan
(If any SV is NO, revise your plan NOW before submitting.)
```

**After dispatch**: Store all M plans verbatim.

#### Deep Gatekeeping

After each round's Phase A, apply Gatekeeping checks (see Gatekeeping section below).

#### Deep Phase B: Cross-Validation (M_active parallel dispatches)

> **Streaming optimization**: You may begin Phase B dispatches as soon as ≥2 Phase A results are available. Do not wait for all Phase A dispatches to complete.

For each subagent, dispatch with ALL OTHER subagents' plans from Phase A:

```
You are an analyst in the Forest deliberation system.

User question:
---
[EXACT USER QUESTION]
---

Project context:
---
[PROJECT CONTEXT INCLUDING EXTERNAL INTELLIGENCE]
---

Below are the latest plans from the other analysts. Your own plan is included for reference.

[FOR EACH OTHER AGENT Y]:
===== [Y'S NAME] PLAN =====
[Y'S FULL PLAN FROM PHASE A]

===== YOUR PLAN (for reference) =====
[YOUR FULL PLAN FROM PHASE A]

Task: Cross-validate the other analysts' plans against the actual project code.

[FOR EACH OTHER AGENT Y]:
### [Y's name] plan validation
(Check their assumptions against real code. Identify strengths, gaps, and errors.)

### Questions for [Y's name]
(Specific questions you want Y to address in the next round. Max 3 questions.)

[SELF-VERIFY]
SV-1: I cross-validated every other analyst's plan — YES/NO
SV-2: I verified at least one claim from each plan against actual code — YES/NO

## Improvements to my plan
(How your own plan should improve, drawing from ALL cross-validation insights above. Be concrete: what to add, change, or reconsider. If your plan needs no changes, state "No changes needed" with justification.)
```

**After dispatch**: For each subagent, collect:
- All "Questions for [X]" entries (from OTHER subagents — these form the peer feedback for the next round)
- "Improvements to my plan" section (the subagent's own self-reflection — include in next round's feedback)
- The subagent's own plan and cross-validation output are stored for Forest Synthesis reference

#### Consensus Check (after Phase B)

Before proceeding to the next round, evaluate:

- If `M_active` = 1, skip Consensus Check. Proceed directly to Step 4 (Forest Synthesis).

- If all `M_active` subagents' Phase B results show "Improvements to my plan" that are minor (only wording/formatting refinishments; no new files, no new approaches, no structural changes recommended) or none, AND no subagent raised blocking concerns about another's plan:
  → **Early consensus reached.** Skip remaining rounds. Proceed to Step 4.

- If substantial disagreements remain:
  → Continue to the next round.

If this was the last round (`i == ROUNDS`), proceed to Step 4 regardless.

---

## Gatekeeping (applies to ALL modes)

After every Phase A (and after Deep Phase A in each round), verify each subagent's output against these checks:

| Check | Criteria | Action if fail |
|-------|----------|---------------|
| G1: Self-verify complete | All SV-1 through SV-4 (SV-5 for Deep round >1) are present and answered YES | Retry dispatch once |
| G2: Section markers present | [REQ-1] through [REQ-5] all present in output | Retry dispatch once |
| G3: Code evidence | [REQ-3] contains at least 1 specific file path or file:line reference | Retry dispatch once |
| G4: No hallucinated files | If any file path in [REQ-3] seems suspicious, verify with `glob` or `grep` | Flag claim; if confirmed hallucination, retry dispatch once |

**Retry behavior**: If a subagent fails Gatekeeping, retry ONCE with the same prompt plus an appended note:

```
Your previous submission was missing required sections or evidence. 
Please ensure ALL [REQ-N] markers are present, [SELF-VERIFY] is complete with YES answers, 
and [REQ-3] contains specific file:line references.
```

If retry also fails → mark subagent as failed, proceed with degradation rules.

**Optional fact-check**: If any claim in a subagent's output seems factually questionable (e.g., API existence, version compatibility), Forest MAY run a websearch to verify and flag the claim.

---

## Step 4: Forest Synthesis + Web Verification

You now have:
- M subagent plans (from Phase A, and from Phase B if applicable)
- Cross-validation results (Deep mode) or dispute discussion results (Balanced mode) if applicable
- For Balanced mode: certainty/uncertainty analysis ([DC-CERTAIN]/[DC-UNCERTAIN]), strongest arguments ([DP-2]), rebuttals ([DP-REBUT]), measurable concession conditions ([DP-4]), and self-reflection responses ([SR-CONCEDE]/[SR-REFINE]/[SR-REJECT]) per disputed decision
- All feedback and responses
- Decision matrix (Balanced mode) or consensus/dissensus map

Do your own work:

1. **Investigate the codebase yourself** — read relevant files, verify claims and assumptions from the discussion
2. **Weigh consensus against disagreements** — where all agree, that's solid. Where they differ, judge which reasoning is stronger based on your own code investigation
3. **Resolve disputes using evidence** — for each DISPUTED decision (Balanced) or unresolved disagreement (Deep), weigh code evidence, logical reasoning, and external intelligence
4. **Web-verify key decisions** — Before writing the final plan, identify the 2-3 most impactful architectural decisions. For each, run one websearch to confirm alignment with current best practices. If search reveals a better alternative or deprecation warning, adjust the plan. **Constrain**: max 3 searches, 20 seconds total.
5. **Synthesize the final unified plan** — combine the best insights from all subagents, resolve disagreements with evidence, and produce one coherent plan

**Principles**:
- Minimal code changes — reuse what exists
- Optimal efficiency — simplest approach, shortest path
- Actionable — concrete steps, specific files, clear rationale
- Transparent assumptions — mark any `ASSUMED` decisions from Phase 0.3 for user review

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

### Gatekeeping Failures

If a subagent's output fails Gatekeeping checks (G1-G4):
- Retry ONCE with appended correction note (see Gatekeeping section)
- If retry also fails → mark subagent as failed, proceed with degradation

### Degradation

- **M_active-1 subagents succeed**: Proceed normally. Adjust cross-validation: remaining subagents review each other's plans. Use `M_active` (not `M`) for all subsequent counts and checks.
- **1 subagent in Phase A (M_active == 1)**:
  - Fast mode: Proceed to Fast Synthesis with the single lens plan.
  - Balanced mode: Skip Gatekeeping for the single agent. Skip Decision Matrix (no disputes possible with 1 agent). Proceed directly to Step 4.
  - Deep mode: Skip Phase B entirely. Forest conducts independent investigation and produces the plan.
- **0 subagents succeed in Phase A**: Abort. Tell user: "All subagent dispatches failed. Here's what I found from Phase 0 reconnaissance. Would you like me to retry or produce a simplified analysis?"
- **All Phase A succeeded but all Phase B failed** (Balanced mode):
  - Skip Phase C (self-reflection) — no rebuttals to reflect on.
  - Proceed to Resolve Disputes using Phase A positions only (no dispute discussion input).
  - In Forest Synthesis, note that dispute discussion was unavailable and compensate with your own independent investigation.
  - Do NOT abort.
- **All Phase A succeeded but all Phase B failed** (Deep mode):
  - Proceed to the next round using Phase A plans only (no cross-validation feedback).
  - In Forest Synthesis, note that cross-validation was unavailable and compensate with your own independent code investigation.
  - Do NOT abort.

### Response Validation

Validate responses based on which phase and mode produced them:

**Phase A validation (all modes)**: Response must contain all markers [REQ-1] through [REQ-5] and a [SELF-VERIFY] section with SV-1 through SV-4 (SV-5 for Deep round >1). Missing markers → Gatekeeping failure.

**Phase B validation (Balanced mode)**: Response must contain [DC-CERTAIN], [DC-UNCERTAIN], [DP-1] through [DP-4], and at least one [DP-REBUT-{AgentName}] for each disputed decision, and a [SELF-VERIFY] section with SV-1 and SV-2. Missing required sections → treat as dispatch failure (retry once).

**Phase C validation (Balanced mode)**: Response must contain an [SR-CONCEDE], [SR-REFINE], or [SR-REJECT] response for every rebuttal directed at the agent, and a [SELF-VERIFY] section with SV-1. Missing required sections → treat as dispatch failure (retry once).

**Phase B validation (Deep mode)**: Response must contain at least one "plan validation" section per other analyst, "Questions for [Y's name]" entries for each other analyst, and an "Improvements to my plan" section. Missing required sections → treat as dispatch failure (retry once).

Do NOT apply one mode's validation criteria to another mode's outputs.

### Important

Failed subagents are NOT re-dispatched in later phases. They remain absent for the rest of the deliberation. This preserves the "fresh session, no memory of prior rounds" constraint in the Key Rules.

---

## Step Flow Summary

```
Step 1: Configure
  ├─ Question A: Select agents
  ├─ Question B: Select mode (fast / balanced / deep)
  └─ Question C: Select rounds (deep only)

Step 2: Phase 0
  ├─ 0.1: Local Reconnaissance
  ├─ 0.2: Web Augmentation
  └─ 0.3: Raise Questions Gate
       ├─ Ambiguities found → question tool → wait for user → update context
       └─ No ambiguities → proceed

Step 3: Execute by Mode
  ├─ Fast:
  │    ├─ Phase A (role-based lenses)
  │    ├─ Gatekeeping
  │    └─ → Step 4
  │
  ├─ Balanced:
  │    ├─ Phase A (full plans)
  │    ├─ Gatekeeping
  │    ├─ Decision Matrix extraction
  │    ├─ Phase B (dispute-only, if disputes exist)
  │    ├─ Phase C (self-reflection, agents respond to rebuttals)
  │    ├─ Resolve Disputes
  │    └─ → Step 4
  │
  └─ Deep:
       ├─ Round i:
       │    ├─ Phase A (full plans / revised plans)
       │    ├─ Gatekeeping
       │    ├─ Phase B (cross-validation)
       │    └─ Consensus Check
       └─ → Step 4

Step 4: Forest Synthesis + Web Verification
  ├─ Investigate codebase
  ├─ Weigh consensus/disagreements
  ├─ Resolve disputes (Balanced) or weigh cross-validation (Deep)
  ├─ Web-verify 2-3 key decisions
  └─ Produce final unified plan
```