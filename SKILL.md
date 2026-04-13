---
name: omo-ralplan
description: Consensus planning with Planner/Architect/Critic loop - OpenCode model-agnostic version
---

# OMO-Ralplan (Consensus Planning)

OMO-Ralplan is a model-agnostic consensus planning skill that triggers iterative planning with Planner, Architect, and Critic agents until consensus is reached. This version is compatible with oh-my-opencode and works across different model configurations.

## Purpose

This skill provides structured deliberation for planning complex tasks. It uses a Planner → Architect → Critic loop to ensure plans are architecturally sound, thoroughly reviewed, and testable before execution.

## Usage

```
$omo-ralplan "task description"
```

## Flags

- `--interactive`: Enables user prompts at key decision points (draft review in step 2 and final approval in step 6). Without this flag the workflow runs fully automated — Planner → Architect → Critic loop — and outputs the final plan without asking for confirmation.
- `--deliberate`: Forces deliberate mode for high-risk work. Adds pre-mortem (3 scenarios) and expanded test planning (unit/integration/e2e/observability). Without this flag, deliberate mode can still auto-enable when the request explicitly signals high risk (auth/security, migrations, destructive changes, production incidents, compliance/PII, public API breakage).

## Usage with interactive mode

```
$omo-ralplan --interactive "task description"
```

## Workflow

The consensus workflow:

1. **Planner** creates initial plan and a compact **RALPLAN-DR summary** before review:
   - Principles (3-5)
   - Decision Drivers (top 3)
   - Viable Options (>=2) with bounded pros/cons
   - If only one viable option remains, explicit invalidation rationale for alternatives
   - Deliberate mode only: pre-mortem (3 scenarios) + expanded test plan (unit/integration/e2e/observability)

2. **User feedback** *(--interactive only)*: If `--interactive` is set, present the draft plan **plus the Principles / Drivers / Options summary** before review (Proceed to review / Request changes / Skip review). Otherwise, automatically proceed to review.

3. **Architect** reviews for architectural soundness — **await completion before step 4**:
   - Must provide the strongest steelman antithesis
   - At least one real tradeoff tension
   - When possible, synthesis path
   - In deliberate mode: explicitly flag principle violations
   - Returns verdict: `APPROVE`, `ITERATE`, or `REJECT` with specific feedback

4. **Critic** evaluates against quality criteria — run only after step 3 completes:
   - Principle-option consistency
   - Fair alternatives exploration
   - Risk mitigation clarity
   - Testable acceptance criteria
   - Concrete verification steps
   - In deliberate mode: reject missing/weak pre-mortem or expanded test plan
   - Returns verdict: `APPROVE`, `ITERATE`, or `REJECT` with specific feedback

5. **Re-review loop** (max 5 iterations): Any non-`APPROVE` Critic verdict (`ITERATE` or `REJECT`) MUST run the same full closed loop:
   - a. Collect Architect + Critic feedback
   - b. Revise the plan with Planner
   - c. Return to Architect review
   - d. Return to Critic evaluation
   - e. Repeat this loop until Critic returns `APPROVE` or 5 iterations are reached
   - f. If 5 iterations are reached without `APPROVE`, present the best version to the user

6. On Critic approval *(--interactive only)*: Present the plan with approval options (Approve and execute via ralph / Approve and implement via team / Request changes / Reject). Final plan must include:
   - ADR (Decision, Drivers, Alternatives considered, Why chosen, Consequences, Follow-ups)
   - Explicit available-agent-types roster
   - Concrete follow-up staffing guidance for both `ralph` and `team`
   - Suggested reasoning levels by lane
   - Explicit `omx team` / `$team` launch hints
   - Concrete **team -> ralph** verification path

7. *(--interactive only)* User chooses: Approve (ralph or team), Request changes, or Reject

8. *(--interactive only)* On approval: invoke `$ralph` for sequential execution or `$team` for parallel team execution with the explicit available-agent-types roster, reasoning-by-lane guidance, role/staffing allocation guidance, launch hints, and verification-path guidance from the approved plan -- never implement directly

> **CRITICAL:** Steps 3 and 4 MUST run sequentially. Do NOT issue both agent calls in the same parallel batch. Always await the Architect result before invoking Critic.

## Tool Usage

This skill uses native `task()` tool for all agent delegation:

```typescript
// Architect review (BLOCKING - must wait)
task(
  subagent_type="oracle",
  load_skills=[],
  run_in_background=false,
  prompt=`Review this plan for architectural soundness:

**Plan**: {plan_content}

**Requirements**:
1. Provide strongest steelman counterargument (antithesis)
2. Identify at least one meaningful tradeoff tension
3. If possible, suggest synthesis path
4. In deliberate mode: flag principle violations

Return verdict: APPROVE, ITERATE, or REJECT with specific feedback.`
)

// Critic evaluation (BLOCKING, after Architect completes)
task(
  subagent_type="oracle",
  load_skills=[],
  run_in_background=false,
  prompt=`Evaluate this plan against quality criteria:

**Plan**: {plan_content}
**Architect Feedback**: {architect_feedback}

**Verify**:
1. Principle-option consistency
2. Fair alternative exploration
3. Risk mitigation clarity
4. Testable acceptance criteria
5. Concrete verification steps

Return verdict: APPROVE, ITERATE, or REJECT with specific feedback.`
)
```

### Available Subagent Types

| Subagent Type | Purpose | Category |
|---------------|---------|----------|
| `oracle` | Architecture, debugging, expert consultation | consultant |
| `explore` | Fast codebase search, pattern discovery | search |
| `librarian` | External documentation, web research | research |
| `metis` | Pre-planning analysis, ambiguity detection | analyst |
| `momus` | Plan review, quality verification | reviewer |

### Category-Based Delegation (Alternative)

For domain-specific work, use category-based delegation:

```typescript
task(
  category="visual-engineering", // or: ultrabrain, deep, quick, artistry, writing
  load_skills=[],
  run_in_background=true,
  prompt="..."
)
```

### Mandatory Parameters

- **ALWAYS** include `load_skills=[]` parameter
- **ALWAYS** include `run_in_background` parameter (true for parallel, false for blocking)
- **CRITICAL**: Consensus mode agent calls MUST be sequential

## Timeout Configuration

Timeouts are configurable via environment variables (do not hardcode):

| Variable | Default | Description |
|----------|---------|-------------|
| `OMX_CONSENSUS_AGENT_TIMEOUT_MS` | 120000 | Per-agent call timeout |
| `OMX_CONSENSUS_TOTAL_TIMEOUT_MS` | 600000 | Total workflow timeout |
| `OMX_ASK_USER_TIMEOUT_MS` | 300000 | User response timeout |
| `OMX_CONSENSUS_MAX_REVIEW_ITERATIONS` | 5 | Max re-review loops |
| `OMX_CONSENSUS_CIRCUIT_BREAKER_THRESHOLD` | 3 | Same error recurrence limit |

### Timeout Handling

If agent call exceeds timeout:
1. Log timeout event with agent role and duration
2. Fall back to local analysis
3. Continue workflow with fallback result

## Fallback Strategy

### When Subagent Unavailable

If `task(subagent_type="oracle", ...)` fails or is unavailable:
1. Log the failure
2. Perform local analysis using available tools (read, grep, glob)
3. Continue workflow with local analysis result
4. Note in output that expert consultation was unavailable

### Graceful Degradation

Consensus workflow degradation path:
1. Try: `task(subagent_type="oracle", ...)`
2. Fallback: Local analysis with read/grep/glob
3. Fallback: Use `category="deep"` for autonomous analysis
4. Last resort: Present plan with "expert review unavailable" warning

## Pre-context Intake

Before consensus planning or execution handoff, ensure a grounded context snapshot exists:

1. Derive a task slug from the request.
2. Reuse the latest relevant snapshot in `.omx/context/{slug}-*.md` when available.
3. If none exists, create `.omx/context/{slug}-{timestamp}.md` (UTC `YYYYMMDDTHHMMSSZ`) with:
   - task statement
   - desired outcome
   - known facts/evidence
   - constraints
   - unknowns/open questions
   - likely codebase touchpoints
4. If ambiguity remains high, run `explore` first for brownfield facts, then run `$deep-interview --quick <task>` before continuing.

Do not hand off to execution modes until this intake is complete; if urgency forces progress, explicitly document the risk tradeoffs.

## Pre-Execution Gate

### Why the Gate Exists

Execution modes (ralph, autopilot, team, ultrawork) spin up heavy multi-agent orchestration. When launched on a vague request like "ralph improve the app", agents have no clear target — they waste cycles on scope discovery that should happen during planning, often delivering partial or misaligned work that requires rework.

The ralplan-first gate intercepts underspecified execution requests and redirects them through the ralplan consensus planning workflow. This ensures:
- **Explicit scope**: A PRD defines exactly what will be built
- **Test specification**: Acceptance criteria are testable before code is written
- **Consensus**: Planner, Architect, and Critic agree on the approach
- **No wasted execution**: Agents start with a clear, bounded task

### Good vs Bad Prompts

**Passes the gate** (specific enough for direct execution):
- `ralph fix the null check in src/hooks/bridge.ts:326`
- `autopilot implement issue #42`
- `team add validation to function processKeywordDetector`
- `ralph do:\n1. Add input validation\n2. Write tests\n3. Update README`
- `ultrawork add the user model in src/models/user.ts`

**Gated — redirected to ralplan** (needs scoping first):
- `ralph fix this`
- `autopilot build the app`
- `team improve performance`
- `ralph add authentication`
- `ultrawork make it better`

**Bypass the gate** (when you know what you want):
- `force: ralph refactor the auth module`
- `! autopilot optimize everything`

### When the Gate Does NOT Trigger

The gate auto-passes when it detects **any** concrete signal. You do not need all of them — one is enough:

| Signal Type | Example prompt | Why it passes |
|---|---|---|
| File path | `ralph fix src/hooks/bridge.ts` | References a specific file |
| Issue/PR number | `ralph implement #42` | Has a concrete work item |
| camelCase symbol | `ralph fix processKeywordDetector` | Names a specific function |
| PascalCase symbol | `ralph update UserModel` | Names a specific class |
| snake_case symbol | `team fix user_model` | Names a specific identifier |
| Test runner | `ralph npm test && fix failures` | Has an explicit test target |
| Numbered steps | `ralph do:\n1. Add X\n2. Test Y` | Structured deliverables |
| Acceptance criteria | `ralph add login - acceptance criteria: ...` | Explicit success definition |
| Error reference | `ralph fix TypeError in auth` | Specific error to address |
| Code block | `ralph add: \`\`\`ts ... \`\`\`` | Concrete code provided |
| Escape prefix | `force: ralph do it` or `! ralph do it` | Explicit user override |

## Configuration Requirements

This skill expects these agents to be configured in `oh-my-opencode.json`:

```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant",
      "tools": { "read": true, "grep": true, "glob": true }
    },
    "explore": {
      "mode": "subagent",
      "category": "search",
      "tools": { "read": true, "grep": true, "glob": true }
    },
    "librarian": {
      "mode": "subagent",
      "category": "research",
      "tools": { "webfetch": true, "websearch": true, "read": true }
    }
  }
}
```

### Category Requirements

```json
{
  "categories": {
    "visual-engineering": { ... },
    "ultrabrain": { ... },
    "deep": { ... },
    "quick": { ... },
    "artistry": { ... },
    "writing": { ... }
  }
}
```

## Anti-Patterns to Avoid

### MCP Tool References (NEVER USE)

```markdown
<!-- ❌ NEVER USE -->
- Use `ask_codex` with `agent_role: "architect"`
- Use `mcp__x__ask_codex` with `agent_role: "critic"`
- Call `ToolSearch("mcp")` to discover MCP tools
```

### Model-Specific Instructions (NEVER USE)

```markdown
<!-- ❌ NEVER USE -->
- Use GPT-5.4 for architecture review
- Use Claude for analysis
- Model X is better for Y task
```

### Hardcoded Timeouts (NEVER USE)

```markdown
<!-- ❌ NEVER USE -->
- Wait 120 seconds for agent response
- Timeout after 2 minutes
```

### Parallel Consensus Calls (NEVER DO)

```typescript
// ❌ NEVER DO THIS
// Running Architect and Critic in parallel
task(subagent_type="oracle", run_in_background=true, ...) // Architect
task(subagent_type="oracle", run_in_background=true, ...) // Critic

// ✅ CORRECT: Sequential calls
task(subagent_type="oracle", run_in_background=false, ...) // Architect
// WAIT for result
task(subagent_type="oracle", run_in_background=false, ...) // Critic
```

## Examples

### Example: Complete Consensus Workflow

```typescript
// Step 1: Planner creates initial plan (local)
const plan = generateInitialPlan(task, context);

// Step 2: (optional) User feedback
if (interactive) {
  const feedback = await askUserQuestion(plan);
  if (feedback === "request_changes") {
    return step1(); // Loop back
  }
}

// Step 3: Architect review (BLOCKING)
const architectResult = await task({
  subagent_type: "oracle",
  load_skills: [],
  run_in_background: false,
  prompt: `Review plan for architectural soundness: ${plan}`
});

// Step 4: Critic evaluation (BLOCKING, after step 3)
const criticResult = await task({
  subagent_type: "oracle",
  load_skills: [],
  run_in_background: false,
  prompt: `Evaluate plan quality: ${plan}\nArchitect feedback: ${architectResult}`
});

// Step 5: Re-review loop (if needed)
let iterations = 0;
while (criticResult.verdict !== "APPROVE" && iterations < 5) {
  iterations++;
  // Revise plan based on feedback
  plan = revisePlan(plan, architectResult, criticResult);

  // Sequential re-review
  const archResult = await task({ subagent_type: "oracle", load_skills: [], run_in_background: false, ... });
  const critResult = await task({ subagent_type: "oracle", load_skills: [], run_in_background: false, ... });
}

// Step 6: Apply improvements
const finalPlan = applyImprovements(plan, architectResult, criticResult);

// Step 7: Output or handoff
if (interactive) {
  const choice = await askUserQuestion(finalPlan);
  if (choice === "approve_ralph") {
    invokeSkill("ralph", finalPlan);
  } else if (choice === "approve_team") {
    invokeSkill("team", finalPlan);
  }
} else {
  output(finalPlan);
}
```

## Scenario Examples

**Good:** The user says `continue` after the workflow already has a clear next step. Continue the current branch of work instead of restarting or re-asking the same question.

**Good:** The user changes only the output shape or downstream delivery step (for example `make a PR`). Preserve earlier non-conflicting workflow constraints and apply the update locally.

**Bad:** The user says `continue`, and the workflow restarts discovery or stops before the missing verification/evidence is gathered.

## Testing Checklist

Before using this skill, verify:

- [ ] No `ask_codex` or `mcp__x__ask_codex` references
- [ ] No hardcoded model names
- [ ] All agent calls use `task()` tool
- [ ] All `task()` calls include `load_skills=[]`
- [ ] All `task()` calls include `run_in_background` parameter
- [ ] Architect and Critic calls are documented as sequential
- [ ] Timeout configuration references environment variables
- [ ] Fallback strategy documented for unavailable subagents
- [ ] No parallel execution of consensus steps 3 and 4
- [ ] Configuration requirements documented

## Summary

This omo-ralplan skill:

1. **Uses native `task()` delegation** - Never references MCP tools
2. **Is model-agnostic** - References configuration, not models
3. **Follows sequential consensus** - Architect before Critic, never parallel
4. **Includes mandatory parameters** - `load_skills=[]`, `run_in_background`
5. **Documents timeouts** - Uses environment variables, not hardcoded values
6. **Provides fallback** - Local analysis when subagents unavailable
7. **Specifies configuration requirements** - Documents expected agents/categories

This ensures the skill works across different oh-my-opencode configurations regardless of the underlying models.
