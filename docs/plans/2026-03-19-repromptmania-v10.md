# Repromptmania v10.0.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Dimension Interview and Agent Cards to Repromptverse mode, bumping reprompter to v10.0.0.

**Architecture:** Two behavioral features added entirely via SKILL.md prose changes. Dimension Interview inserts a score-driven question gate into Phase 1. Agent Cards add three fixed-format templates (Plan Cards, Status Line, Result Cards) rendered at Phase 1/3/4 respectively. No runtime code changes.

**Tech Stack:** Markdown (SKILL.md, TESTING.md, CHANGELOG.md), JSON (package.json)

**Spec:** `docs/specs/2026-03-19-repromptmania-v10-design.md`

---

### Task 1: Update version and metadata to v10.0.0

**Files:**
- Modify: `package.json:3` (version field)
- Modify: `SKILL.md:15` (frontmatter version)
- Modify: `SKILL.md:18` (heading)
- Modify: `SKILL.md:20` (tagline)

- [ ] **Step 1: Update package.json version**

Change line 3 (the `"version"` field):
```json
"version": "10.0.0",
```

- [ ] **Step 2: Update SKILL.md frontmatter version**

Change line 15:
```
  version: 10.0.0
```

- [ ] **Step 3: Update SKILL.md heading**

Change line 18:
```markdown
# RePrompter v10.0.0
```

- [ ] **Step 4: Update SKILL.md tagline**

Change line 20:
```markdown
> **Your prompt sucks. Let's fix that.** Single prompts or full agent teams — one skill, two modes. **v10.0 adds Dimension Interview + Agent Cards to Repromptverse.**
```

- [ ] **Step 5: Update SKILL.md frontmatter description**

Add Dimension Interview and Agent Cards mention to the description field (line 4-7). Append to the existing `Outputs:` line:
```
  Outputs: Structured XML/Markdown prompt, quality score (before/after), optional team brief + per-agent sub-prompts, agent team output files, Agent Cards (plan/status/result).
```

- [ ] **Step 6: Verify changes**

Run: `grep -n "10.0.0" SKILL.md package.json`
Expected: 3 matches (package.json:3, SKILL.md:15, SKILL.md:18)

- [ ] **Step 7: Commit**

```bash
git add package.json SKILL.md
git commit -m "chore: bump version to 10.0.0 (Repromptmania)"
```

---

### Task 2: Add Dimension Interview to Phase 1

**Files:**
- Modify: `SKILL.md:220-262` (TL;DR block + Phase 1 block)

This is the largest change. We insert the Dimension Interview gate into Phase 1 and update the TL;DR.

- [ ] **Step 1: Update TL;DR block (lines 220-229)**

Replace the TL;DR content (lines 222-228) with:
```
Raw task in → quality output out. Every agent gets a reprompted prompt.

Phase 1: Score raw prompt, dimension interview if needed, plan team, show Agent Cards (YOU do this, ~45s)
Phase 2: Write XML-structured prompt per agent (YOU do this, ~2min)
Phase 3: Launch agents (tmux, TeamCreate, sessions_spawn, Codex, or sequential) (AUTOMATED)
Phase 4: Show Result Cards, score, retry if needed (YOU do this)
```

- [ ] **Step 2: Rewrite Phase 1 block (lines 256-262)**

Replace the entire Phase 1 section. Note: heading changes from `~30 seconds` to `~45 seconds`. Replace with:

```markdown
### Phase 1: Team plan (~45 seconds)

1. **Score raw prompt** (1-10): Clarity, Specificity, Structure, Constraints, Decomposition
   - Phase 1 uses 5 quick-assessment dimensions. The full 6-dimension scoring (adding Verifiability) is used in Phase 4 evaluation.
2. **Dimension Interview gate** — check which askable dimensions scored < 5 (see Dimension Interview section below)
3. **Pick mode:** parallel (independent agents) or sequential (pipeline with dependencies)
4. **Define team:** 2-5 agents max, each owns ONE domain, no overlap (informed by interviewContext if interview ran)
5. **Show Plan Cards** (see Agent Cards section below)
6. **User confirmation gate** — "Team plan ready. Proceed to execution?" User can approve, adjust, or cancel. In automated/batch runs, auto-proceed.
7. **Write team brief** to `/tmp/rpt-brief-{taskname}.md` (use unique tasknames to avoid collisions; includes interviewContext section if interview ran)
```

- [ ] **Step 3: Add Dimension Interview subsection after Phase 1**

Insert a new subsection immediately after the Phase 1 block (after the new step 7, before Phase 2).

**IMPORTANT: The content below is the literal text to insert into SKILL.md. It contains its own code fences - copy as-is, do not wrap in additional fences.**

**--- BEGIN CONTENT TO INSERT ---**

### Dimension Interview (Repromptverse only)

Score-driven interview for Repromptverse mode. Distinct from Single mode's "Smart Interview" (which uses a standard question list). The Dimension Interview derives questions from low-scoring raw prompt dimensions.

#### Trigger logic

    scores = score_raw_prompt(rawInput)  # 5 dimensions from step 1

    # Structure is EXCLUDED — reprompter fixes structure via templates.
    # Only 4 dimensions are interview-eligible:
    askable = [d for d in scores if d.name != "Structure" and d.value < 5]

    # Threshold: strict less-than. Scores of 5+ do NOT trigger questions.
    if len(askable) == 0:
        SKIP interview → proceed to step 3 (pick mode)
    elif len(askable) <= 2:
        ASK 1-2 questions (one per low dimension)
    else:
        ASK 3-4 questions (max 4, prioritized by lowest score first)

Note: In SKILL.md, wrap this pseudocode in a triple-backtick code fence.

#### Dimension-to-question mapping

| Dimension | Score < 5 triggers | Question approach |
|-----------|-------------------|-------------------|
| **Clarity** | Task is ambiguous or multi-interpretable | Open-ended with dynamic options extracted from prompt keywords |
| **Specificity** | Scope is vague, no concrete targets | Dynamic options from prompt keywords + top-level directory names |
| **Constraints** | No boundaries defined | "Any areas to exclude?" with context-aware options |
| **Decomposition** | Unclear work split | "How many independent streams?" with suggested splits |

**Question rules:**
- Use `AskUserQuestion` with clickable options (consistent with Single mode)
- Options are **dynamic**: extracted from prompt keywords + codebase context (config files + top-level dirs only - no deep analysis)
- Every question includes a free-text escape hatch option
- Priority order: lowest scoring dimension first
- Language follows user's input language

#### Skip/dismiss handling

- User skips all questions -> proceed with empty interviewContext. Plan Cards note: "Interview: skipped by user"
- User answers some, skips others -> populate only answered fields

#### Interview output (interviewContext)

Responses merge into an interviewContext written to the team brief file:

    interviewContext = {
      scope: [from Specificity answer],
      excludes: [from Constraints answer],
      successCriteria: [from answers, or omitted - Phase 2 derives from requirements],
      taskClarification: [from Clarity answer, if asked]
    }

Note: In SKILL.md, wrap this schema in a triple-backtick code fence.

When `successCriteria` is not gathered (question not asked or user skipped), omit the field entirely. Phase 2 will derive success criteria from requirements as it does today.

**How interviewContext feeds into later phases:**
- **Agent count and roles** - scope determines which agents are created
- **Per-agent `<constraints>`** - excludes injected into each agent's prompt
- **Per-agent `<success_criteria>`** - user expectations propagated
- **Template selection** - clarified task type may route to a different swarm profile

**Precedence:** Interview responses override auto-detected codebase context. Conflicts noted in Plan Cards.

**Flywheel:** interviewContext is excluded from recipe fingerprint hash. The fingerprint captures strategy (template + patterns + tier), not user scope answers.

**--- END CONTENT TO INSERT ---**

- [ ] **Step 4: Verify the new sections read correctly**

Read SKILL.md lines 256-340 (approximate) and confirm:
- Phase 1 has 7 steps
- Dimension Interview section follows Phase 1
- No broken markdown formatting

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Dimension Interview to Repromptverse Phase 1"
```

---

### Task 3: Add Agent Cards section

**Files:**
- Modify: `SKILL.md` (insert new section between Dimension Interview and Phase 2)

- [ ] **Step 1: Insert Agent Cards section after Dimension Interview, before Phase 2**

Add immediately before the `### Phase 2: Repromptverse prompt pack` heading.

**IMPORTANT: The content below is the literal text to insert into SKILL.md. It contains its own code fences - copy as-is, do not wrap in additional fences.**

**--- BEGIN CONTENT TO INSERT ---**

### Agent Cards (transparency layer)

Three fixed-format card types rendered at different phases. Templates are exact - do not invent new formats.

#### Plan Cards - rendered at end of Phase 1 (step 5)

After team plan is complete, before Phase 2 prompt writing. Use this exact table format:

    ## Team: {N} Opus Agents ({Parallel|Sequential})

    | # | Agent | Scope | Excludes | Output |
    |---|-------|-------|----------|--------|
    | 1 | {role} | {scope} | {excludes or "-"} | {output path} |
    | 2 | {role} | {scope} | {excludes or "-"} | {output path} |

    Interview context applied: {summary of influence, including override conflicts,
    or "No interview (high-quality prompt)", or "Interview: skipped by user"}

Note: In SKILL.md, wrap this template in a triple-backtick markdown code fence.

**Rules:**
- MUST appear before any agent is launched
- If interview ran, show which constraints came from interview vs auto-detected
- If user requests agent adjustments at confirmation gate, re-render Plan Cards with updated team
- Single-agent runs: table renders with one row (valid)

#### Status Line - rendered during Phase 3 polling

Compact one-line status with each poll cycle:

    Agents: ✅ 2/4  ⏳ 1/4  🔄 1/4 (retry 1)

Note: In SKILL.md, wrap this example in a triple-backtick code fence.

**Rules:**
- Replace verbose poll output with this compact format
- Platform-dependent: TeamCreate uses TaskList status; tmux uses best-effort pane parsing; sequential is trivial
- Show retry count for retrying agents

#### Result Cards - rendered at start of Phase 4

After reading all agent outputs, before synthesis. Use this exact table format:

    ## Results

    | Agent | Score | Findings | Key Insight |
    |-------|-------|----------|-------------|
    | {role} | {score}/10 {pass/retry emoji} | {count} findings | {one-sentence top finding} |

    Total: {N} findings | {accepted}/{total} accepted | {retry_count} retries

Note: In SKILL.md, wrap this template in a triple-backtick markdown code fence.

**Rules:**
- MUST appear before synthesis is written
- "Key Insight" = single most important finding per agent (forces prioritization)
- Retry agents show retry reason in findings column

#### Token budget (Agent Cards + Dimension Interview)

| Phase | Extra tokens | Source |
|-------|-------------|--------|
| Phase 1 (interview) | 100-400 | AskUserQuestion calls (0-4 questions) + option generation from config/directory scan |
| Phase 1 (plan cards) | 100-300 | Table render (varies by team size) |
| Phase 3 (status) | ~20/poll | Compact status line |
| Phase 4 (result cards) | 150-250 | Summary table |
| **Total** | **~400-1000** | **0.5-2% of typical 50K-200K run** |

**--- END CONTENT TO INSERT ---**

- [ ] **Step 2: Verify markdown nesting**

Read the newly inserted section and confirm:
- No broken code fences (nested markdown blocks use 4-backtick fences or indentation)
- Section hierarchy: `### Agent Cards` is same level as `### Phase 2`

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Agent Cards transparency layer (Plan/Status/Result)"
```

---

### Task 4: Update Phase 3 with Status Line requirement

**Files:**
- Modify: `SKILL.md` (Phase 3 polling sections — tmux step 4, TeamCreate step 4, and all other options)

- [ ] **Step 1: Add Status Line note to tmux Option A polling section**

After the `# 4. Monitor (poll every 15-30s)` comment (currently around line 312-313), add a comment line:

```bash
# Show Status Line with each poll: Agents: ✅ N/T ⏳ N/T 🔄 N/T
tmux capture-pane -t {session} -p -S -100
```

- [ ] **Step 2: Add Status Line note to TeamCreate Option B**

After the `# 4. Wait for teammates to complete (messages arrive automatically)` comment (around line 390), add:

```text
# 4. Wait for teammates to complete — show Status Line per poll cycle
# Status Line: Agents: ✅ N/T ⏳ N/T 🔄 N/T (derived from TaskList status)
```

- [ ] **Step 3: Add general Status Line rule to Phase 3 intro**

After the Phase 3 opening paragraph ("Phase 3 has platform-specific execution methods..."), add:

```markdown
**Status Line (all platforms):** During polling, show compact agent status with each cycle. See Agent Cards section for format.
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Status Line requirement to Phase 3 polling"
```

---

### Task 5: Update Phase 4 with Result Cards requirement

**Files:**
- Modify: `SKILL.md` (Phase 4 block, currently around lines 336-358)

**Note:** In the current SKILL.md, Phase 4 (line 336) appears BEFORE Phase 3 Options B-E (lines 370-424). This is a pre-existing structural oddity. Edit Phase 4 at its actual location (line 336), not after all Phase 3 options.

- [ ] **Step 1: Add Result Cards step before "Deliver final report"**

The current Phase 4 step 4 is "Deliver final report to user." Insert a new step before it. The new Phase 4 becomes:

```markdown
### Phase 4: Evaluate + retry

1. Read each agent's report
2. Score against success criteria from Phase 2:
   - 8+/10 → ACCEPT
   - 4-6/10 → RETRY with delta prompt (tell them what's missing)
   - < 4/10 → RETRY with full rewrite

   **Accept checklist** (use alongside score — all must pass):
   - [ ] All required output sections present
   - [ ] Requirements from Phase 2 independently verifiable
   - [ ] No hallucinated file paths or line numbers
   - [ ] Scope boundaries respected (no overlap with other agents)
3. Max 2 retries (3 total attempts)
4. **Show Result Cards** — render summary table before synthesis (see Agent Cards section for format)
5. Deliver final report to user
```

- [ ] **Step 2: Verify Phase 4 reads correctly**

Read the Phase 4 section and confirm step numbering is 1-5 with Result Cards at step 4.

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Result Cards to Phase 4 before synthesis"
```

---

### Task 6: Add new test scenarios to TESTING.md

**Files:**
- Modify: `TESTING.md` (append after Scenario 33, before Anti-Patterns)

- [ ] **Step 1: Insert 8 new scenarios before Anti-Patterns section**

Find the `---` separator before `## Anti-Patterns` (currently line 334) and insert before it:

```markdown
## Scenario 34: Dimension Interview - Low Specificity Triggers Question

**Input:** "repromptverse - audit the system"
**Expected:** Specificity scores < 5. Interview asks scope clarification with dynamic options derived from codebase top-level directories.
**Verify:** AskUserQuestion called, options reference actual project modules, interviewContext.scope is populated.

## Scenario 35: Dimension Interview - High Score Skips Interview

**Input:** "repromptverse - audit auth module and payment gateway for SQL injection, CSRF, and token expiry. Min 10 findings per agent. Frontend out of scope."
**Expected:** All askable dimensions (Clarity, Specificity, Constraints, Decomposition) score >= 5. Interview skipped entirely.
**Verify:** No AskUserQuestion call, Plan Cards shown immediately after raw score.

## Scenario 36: Plan Cards + User Confirmation Before Execution

**Input:** Any Repromptverse task.
**Expected:** Plan Cards table shown after team plan, user confirmation requested before execution.
**Verify:** Table includes all agents with role, scope, excludes, output path. Execution does not start until user confirms.

## Scenario 37: Result Cards Rendered After Execution

**Input:** Any completed Repromptverse run.
**Expected:** Result Cards table shown after all agents complete (or retry), before synthesis.
**Verify:** Table includes score, finding count, key insight per agent. Total row shows aggregate stats.

## Scenario 38: Interview Context Flows Into Agent Constraints

**Input:** "repromptverse - audit katman" then answer interview with "frontend out of scope" and "min 10 findings"
**Expected:** Every agent's `<constraints>` includes "Frontend out of scope". Every agent's `<success_criteria>` includes "minimum 10 findings".
**Verify:**
- Read agent prompts from `/tmp/rpt-agent-prompts-*.md`, confirm interview context is embedded
- Plan Cards table distinguishes interview-sourced vs auto-detected constraints

## Scenario 39: Dimension Interview - Maximum Questions (All Dimensions Low)

**Input:** "repromptverse - do stuff to the thing"
**Expected:** All 4 askable dimensions (Clarity, Specificity, Constraints, Decomposition) score < 5. Exactly 4 questions asked in priority order (lowest score first). Structure is NOT asked about.
**Verify:** 4 AskUserQuestion calls, no Structure question, priority ordering matches score ranking.

## Scenario 40: Status Line Rendered During Polling

**Input:** Any Repromptverse task during Phase 3 execution.
**Expected:** Each poll cycle shows compact status line with emoji indicators.
**Verify:** Status line format matches `Agents: ✅ N/T ⏳ N/T 🔄 N/T` pattern. Retry agents show retry count.

## Scenario 41: Interview Dismissed - Graceful Fallback

**Input:** "repromptverse - audit the system" then skip/dismiss all interview questions.
**Expected:** Proceed with empty interviewContext. Plan Cards show "Interview: skipped by user".
**Verify:** No interviewContext applied to agent prompts. Team plan uses auto-detected context only.
```

- [ ] **Step 2: Add new anti-patterns**

Append to the Anti-Patterns table:

```markdown
| Launch agents without showing Plan Cards | User must see agent plan before $2-4 execution |
| Write synthesis without Result Cards | Result summary is mandatory before synthesis |
| Ask Structure questions in Dimension Interview | Structure is auto-fixed, never asked |
| Show Dimension Interview for high-score prompts | All askable dimensions >= 5 means skip |
```

- [ ] **Step 3: Update SKILL.md scenario count reference**

SKILL.md line 661 says "33 verification scenarios". Update to "41 verification scenarios".

- [ ] **Step 4: Verify scenario numbering**

Run: `grep -c "^## Scenario" TESTING.md`
Expected: 41

- [ ] **Step 5: Commit**

```bash
git add TESTING.md SKILL.md
git commit -m "test: add 8 scenarios for Dimension Interview + Agent Cards"
```

---

### Task 7: Write CHANGELOG entry for v10.0.0

**Files:**
- Modify: `CHANGELOG.md:1` (prepend new entry)

- [ ] **Step 1: Prepend v10.0.0 entry at top of changelog**

Insert after the `# RePrompter Changelog` heading (line 1), before the `## v9.2.1` entry (line 3):

```markdown

## v10.0.0 (2026-03-19) — Repromptmania

### Added
- **Dimension Interview** — score-driven interview for Repromptverse Phase 1. Askable dimensions (Clarity, Specificity, Constraints, Decomposition) scoring < 5 trigger targeted AskUserQuestion calls (0-4 questions). Structure excluded (auto-fixed by templates). Interview responses feed into agent planning via interviewContext.
- **Agent Cards** — three transparency templates for Repromptverse:
  - **Plan Cards** (Phase 1): team roster table with roles, scopes, excludes, output paths
  - **Status Line** (Phase 3): compact emoji-based polling status per agent
  - **Result Cards** (Phase 4): per-agent score, finding count, and key insight summary
- **User confirmation gate** — Plan Cards shown before execution; user must approve team plan before agents launch
- 8 new test scenarios (34-41) covering Dimension Interview triggers, Agent Cards rendering, interview-to-constraint flow, and edge cases

### Changed
- Phase 1 expanded from 4 steps to 7 (score → interview → pick mode → define team → Plan Cards → confirm → write brief)
- Phase 1 time estimate updated from ~30s to ~45s
- Phase 4 adds Result Cards as mandatory step before synthesis (step 4 of 5)
- Phase 3 polling now shows Status Line format across all platform options
- 4 new anti-patterns added to TESTING.md

### Migration
- Breaking: Repromptverse Phase 1 now includes optional interview and mandatory confirmation gate. Existing workflows may see new AskUserQuestion calls.
```

- [ ] **Step 2: Verify changelog ordering**

Run: `head -5 CHANGELOG.md`
Expected: Line 3 starts with `## v10.0.0`

- [ ] **Step 3: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: add v10.0.0 Repromptmania changelog entry"
```

---

### Task 8: Final verification

**Files:** All modified files

- [ ] **Step 1: Verify version consistency**

```bash
grep -n "10.0.0" SKILL.md CHANGELOG.md package.json
```
Expected: At least 4 matches (package.json:3, SKILL.md frontmatter, SKILL.md heading, CHANGELOG.md entry)

- [ ] **Step 2: Verify SKILL.md Phase 1 has 7 steps**

Read Phase 1 section, confirm steps 1-7 are present and numbered correctly.

- [ ] **Step 3: Verify Dimension Interview section exists and has all subsections**

Confirm these subsections exist in SKILL.md:
- Trigger logic (with pseudocode)
- Dimension-to-question mapping (table)
- Question rules
- Skip/dismiss handling
- Interview output (interviewContext schema)
- Precedence rule
- Flywheel interaction

- [ ] **Step 4: Verify Agent Cards section has all three card types**

Confirm Plan Cards, Status Line, Result Cards templates are present with exact formats.

- [ ] **Step 5: Verify Phase 4 has 5 steps with Result Cards at step 4**

Read Phase 4 section and confirm.

- [ ] **Step 6: Verify test scenario count**

```bash
grep -c "^## Scenario" TESTING.md
```
Expected: 41

- [ ] **Step 7: Run existing test suite (no runtime changes, should still pass)**

```bash
npm run check
```
Expected: All existing tests pass (no runtime code was changed).

- [ ] **Step 8: Commit verification (if any formatting fixes needed)**

```bash
git status
# If clean: done
# If changes: git add SKILL.md TESTING.md CHANGELOG.md package.json && git commit -m "fix: formatting fixes from final verification"
```
