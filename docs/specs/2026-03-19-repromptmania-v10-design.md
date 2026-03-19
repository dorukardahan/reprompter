# Repromptmania v10.0.0 - Design Spec

> Codename: **Repromptmania**
> Version: 10.0.0
> Date: 2026-03-19
> Status: Draft (revision 2 - post spec review)

## Problem Statement

Reprompter (current version) has two UX gaps in Repromptverse (multi-agent) mode:

1. **No interview in Repromptverse.** Single mode has a Smart Interview that gathers requirements via AskUserQuestion. Repromptverse mode skips it entirely - Phase 1 goes directly from raw prompt scoring to team planning. Result: raw prompts scoring 3.9/10 trigger 5 opus agents ($2-4) with no clarification.

2. **No agent transparency.** Users see "5 agents spawned" but don't know: what each agent does, what scope it covers, what it found. The value proposition of Repromptverse is invisible.

## Features

### Feature 1: Dimension Interview (Repromptverse Phase 1)

**Name convention:** "Dimension Interview" for Repromptverse (score-driven, 0-4 questions). Distinct from "Smart Interview" in Single mode (standard question list, 2-5 questions).

**Core idea:** Reuse the existing raw prompt score (5 dimensions) to derive interview questions. Low-scoring dimensions become questions. No separate gap analysis engine needed.

#### Trigger Logic

```
scores = score_raw_prompt(rawInput)  # Clarity, Specificity, Structure, Constraints, Decomposition

# Structure is EXCLUDED from interview — reprompter fixes structure itself.
# Only 4 dimensions are interview-eligible:
askable = [d for d in scores if d.name != "Structure" and d.value < 5]

# Threshold: strict less-than. Scores of 5+ do NOT trigger questions.
if len(askable) == 0:
    SKIP interview → proceed to team plan
elif len(askable) <= 2:
    ASK 1-2 questions (one per low dimension)
else:
    ASK 3-4 questions (max 4, prioritized by lowest score first)
```

#### Dimension-to-Question Mapping

| Dimension | Score < 5 triggers | Question type |
|-----------|-------------------|---------------|
| **Clarity** | Task is ambiguous or has multiple interpretations | Open-ended: "What exactly do you want to accomplish?" with dynamic options extracted from prompt keywords |
| **Specificity** | Scope is vague, no concrete targets named | Dynamic options: extract systems/modules from prompt + top-level directory names, ask which to include |
| **Constraints** | No boundaries defined | "Any areas to exclude from scope?" with context-aware options |
| **Decomposition** | Unclear how work splits across agents | "How many independent work streams does this split into?" with suggested split based on detected systems |

**Language:** Question language follows the user's input language. Examples above are in English; for Turkish input, equivalent Turkish questions are generated.

#### Question Generation Rules

- Questions use `AskUserQuestion` with clickable options (consistent with Single mode)
- Options are **dynamic**, not hardcoded: extracted from prompt keywords + codebase context (same approach as Single mode, SKILL.md line 84)
- Codebase context is **lightweight only**: config file scanning (package.json, tsconfig, etc.) + top-level directory names. No deep module analysis.
- Every question includes a free-text escape hatch option
- Questions are asked in priority order (lowest scoring dimension first)

#### Interview Skip/Dismiss Handling

- If user skips or dismisses all questions: proceed with empty interviewContext, as if interview was skipped. Note in Plan Cards: "Interview: skipped by user"
- If user answers some but skips others: populate interviewContext with answered fields only, leave skipped fields empty

#### Interview Output

Interview responses are merged into an `interviewContext` written as a section in the team brief file (`/tmp/rpt-brief-{taskname}.md`):

```
interviewContext = {
  scope: [extracted from Specificity answer],
  excludes: [extracted from Constraints answer],
  successCriteria: [extracted from answers, or omitted if not gathered],
  taskClarification: [extracted from Clarity answer, if asked]
}
```

When `successCriteria` is not gathered (no question asked or user skipped), it is omitted and existing template defaults apply.

This context directly influences:
- **Agent count and roles** - scope determines which agents are created
- **Per-agent `<constraints>`** - excludes are injected into each agent's prompt
- **Per-agent `<success_criteria>`** - user expectations are propagated
- **Template selection** - clarified task type may route to a different swarm profile

**Precedence rule:** Interview responses override auto-detected codebase context. If conflict exists (e.g., auto-detected Next.js but user says "Python backend only"), interview wins. Conflict is noted in Plan Cards.

#### Flywheel Interaction

interviewContext is **excluded** from the recipe fingerprint hash. Rationale: the fingerprint captures the strategy (template + patterns + tier), not the user's specific scope answers. Including it would fragment fingerprints and reduce flywheel learning signal. The interview influences the prompt content (via Phase 2), which the flywheel already captures through outcome scoring.

### Feature 2: Agent Cards (Transparency Layer)

Three card types rendered at different phases, all using fixed templates (not AI-generated formats) to minimize token cost.

The card format definitions live in a single "Agent Cards" section in SKILL.md (after Phase 1, before Phase 2). Each card type specifies which phase renders it.

#### Plan Cards - rendered at end of Phase 1 (~100-300 tokens)

Shown after team plan is complete, before Phase 2 prompt writing. Table format:

```markdown
## Team: {N} Opus Agents ({Parallel|Sequential})

| # | Agent | Scope | Excludes | Output |
|---|-------|-------|----------|--------|
| 1 | {role} | {scope} | {excludes or "-"} | {output path} |
| 2 | {role} | {scope} | {excludes or "-"} | {output path} |
...

Interview context applied: {summary of interview influence, or "No interview (high-quality prompt)", or "Interview: skipped by user"}
```

Rules:
- Table MUST appear before any agent is launched
- If interview ran, show which constraints came from interview vs auto-detected
- **User confirmation gate:** After showing Plan Cards, ask: "Team plan ready. Proceed to execution?" User can approve, adjust agents, or cancel. If no response within context (e.g., automated/batch run), auto-proceed.
- For single-agent Repromptverse runs (rare): table renders with one row. This is valid and expected.

#### Status Line - rendered during Phase 3 (~20 tokens per poll)

During execution polling, compact one-line status:

```
Agents: {emoji}{count}/{total} ...
```

Emoji key: completed, in-progress, retry

Rules:
- Shown with each poll cycle
- No additional token cost beyond what polling already uses
- If an agent is retrying, show retry count
- **Platform-dependent:** For TeamCreate, derived from TaskList status (structured). For tmux, best-effort parsing from `tmux capture-pane` output. For sequential, trivially derived from execution order.

#### Result Cards - rendered at start of Phase 4 (~150-250 tokens)

After reading all agent outputs, before synthesis:

```markdown
## Results

| Agent | Score | Findings | Key Insight |
|-------|-------|----------|-------------|
| {role} | {score}/10 {emoji} | {count} findings | {one-sentence top finding} |
...

Total: {N} findings | {accepted}/{total} accepted | {retry_count} retries
```

Rules:
- Table MUST appear before synthesis is written
- "Key Insight" is the single most important finding from each agent (forces prioritization)
- Retry agents show their retry reason in the findings column

#### Token Budget

| Phase | Extra tokens | Source |
|-------|-------------|--------|
| Phase 1 (interview) | 100-400 | AskUserQuestion calls (0-4 questions) + reasoning for dynamic options |
| Phase 1 (plan cards) | 100-300 | Table render (varies by team size) |
| Phase 3 (status) | ~20/poll | Compact status line |
| Phase 4 (result cards) | 150-250 | Summary table |
| **Total** | **~400-1000** | **0.5-2% of typical 50K-200K run** |

Note: Token estimates include output tokens and minimal reasoning overhead. The total remains negligible relative to multi-agent execution cost.

## SKILL.md Change Map

### 1. Phase 1 block (current lines 256-263)

Add Dimension Interview gate between step 1 (score) and step 2 (pick mode). Add Plan Cards + user confirmation at end. Update time estimate from ~30s to ~45s.

New Phase 1 steps:
1. Score raw prompt (5 dimensions)
2. **Dimension Interview gate** - askable dimensions (excluding Structure) scoring < 5 become AskUserQuestion calls
3. Pick mode (parallel/sequential)
4. Define team (informed by interviewContext)
5. **Show Plan Cards table**
6. **User confirmation gate** - "Team plan ready. Proceed?"
7. Write team brief to `/tmp/rpt-brief-{taskname}.md` (includes interviewContext section)

### 2. New section: Agent Cards (between Phase 1 and Phase 2)

Define all three card templates in one section. Specify which phase renders each:
- Plan Cards: end of Phase 1 (step 5)
- Status Line: during Phase 3 polling
- Result Cards: start of Phase 4

### 3. Phase 3 execution blocks (current lines 281-424)

Add Status Line format requirement to all polling sections across all platform options (tmux, TeamCreate, sessions_spawn, Codex, sequential). Note platform-dependent parsing.

### 4. Phase 4 block (current lines 336-365)

Add Result Cards table as mandatory step before "Deliver final report to user" (current step 4).

Note: In current SKILL.md, Phase 4 (lines 336-365) appears before Phase 3 Options B-E (lines 370-424). This is a pre-existing structural oddity. This spec does not reorganize Phase 3/4 ordering - changes are inserted at the correct logical locations regardless of line ordering.

### 5. Version and metadata

- `version: 10.0.0` in frontmatter and heading
- Update frontmatter description to mention Dimension Interview + Agent Cards
- CHANGELOG entry for v10.0.0

### 6. Dimension Interview section

Add new subsection under Phase 1 documenting:
- Trigger logic (with Structure exclusion clearly stated)
- Dimension-to-question mapping table
- Dynamic option generation rules (lightweight codebase context only)
- interviewContext schema and materialization (in team brief file)
- Skip/dismiss handling
- How interview feeds into Phase 2
- Precedence over auto-detected context

## New Test Scenarios (TESTING.md)

### Scenario 34: Dimension Interview - Low Specificity Triggers Question

**Input:** "repromptverse - audit the system"
**Expected:** Specificity scores < 5. Interview asks scope clarification with dynamic options derived from codebase top-level directories.
**Verify:** AskUserQuestion called, options reference actual project modules, interviewContext.scope is populated.

### Scenario 35: Dimension Interview - High Score Skips Interview

**Input:** "repromptverse - audit auth module and payment gateway for SQL injection, CSRF, and token expiry. Min 10 findings per agent. Frontend out of scope."
**Expected:** All askable dimensions (Clarity, Specificity, Constraints, Decomposition) score >= 5. Interview skipped entirely.
**Verify:** No AskUserQuestion call, Plan Cards shown immediately after raw score.

### Scenario 36: Plan Cards + User Confirmation Before Execution

**Input:** Any Repromptverse task.
**Expected:** Plan Cards table shown after team plan, user confirmation requested before execution.
**Verify:** Table includes all agents with role, scope, excludes, output path. Execution does not start until user confirms.

### Scenario 37: Result Cards Rendered After Execution

**Input:** Any completed Repromptverse run.
**Expected:** Result Cards table shown after all agents complete (or retry), before synthesis.
**Verify:** Table includes score, finding count, key insight per agent. Total row shows aggregate stats.

### Scenario 38: Interview Context Flows Into Agent Constraints

**Input:** "repromptverse - audit katman" then answer interview with "frontend out of scope" and "min 10 findings"
**Expected:** Every agent's `<constraints>` includes "Frontend out of scope". Every agent's `<success_criteria>` includes "minimum 10 findings".
**Verify:**
- Read agent prompts from `/tmp/rpt-agent-prompts-*.md`, confirm interview context is embedded
- Plan Cards table distinguishes interview-sourced vs auto-detected constraints

### Scenario 39: Dimension Interview - Maximum Questions (All Dimensions Low)

**Input:** "repromptverse - do stuff to the thing"
**Expected:** All 4 askable dimensions (Clarity, Specificity, Constraints, Decomposition) score < 5. Exactly 4 questions asked in priority order (lowest score first). Structure is NOT asked about.
**Verify:** 4 AskUserQuestion calls, no Structure question, priority ordering matches score ranking.

### Scenario 40: Status Line Rendered During Polling

**Input:** Any Repromptverse task during Phase 3 execution.
**Expected:** Each poll cycle shows compact status line with emoji indicators.
**Verify:** Status line format matches `Agents: {emoji}{count}/{total}` pattern. Retry agents show retry count.

### Scenario 41: Interview Dismissed - Graceful Fallback

**Input:** "repromptverse - audit the system" then skip/dismiss all interview questions.
**Expected:** Proceed with empty interviewContext. Plan Cards show "Interview: skipped by user".
**Verify:** No interviewContext applied to agent prompts. Team plan uses auto-detected context only.

## Non-Goals

- **No changes to Single mode.** Smart Interview already exists there.
- **No runtime code changes.** This is a SKILL.md behavioral spec, not application code.
- **No new templates.** Existing templates are sufficient; Agent Cards are inline format specs.
- **No flywheel fingerprint changes.** interviewContext is excluded from recipe hash (see Flywheel Interaction section).
- **No Phase 3/4 reordering.** Pre-existing structural oddity in SKILL.md is acknowledged but not addressed in this version.

## Migration

- Breaking change: Phase 1 flow adds new steps (interview + user confirmation). Existing Repromptverse prompts will now potentially get interview questions and a confirmation gate they didn't have before.
- Non-breaking: Agent Cards are additive output. No existing behavior is removed.
- CHANGELOG must clearly document the Dimension Interview as a new capability, not a fix.
