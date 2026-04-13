# OMO-Ralplan Skill

**Model-Agnostic Consensus Planning for Oh-My-OpenCode**

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Quick Start](#quick-start)
4. [How It Works](#how-it-works)
5. [Usage Guide](#usage-guide)
6. [Examples](#examples)
7. [Skill Datasheet](#skill-datasheet)
8. [Configuration](#configuration)
9. [Troubleshooting](#troubleshooting)
10. [Architecture](#architecture)

---

## Overview

OMO-Ralplan is a **model-agnostic consensus planning skill** that orchestrates iterative planning through a Planner → Architect → Critic loop until consensus is reached. This skill is designed to work with any underlying AI model configuration, making it portable across different oh-my-opencode deployments.

### Key Features

- **Model-Agnostic**: Works with any configured AI model (GPT, Claude, Gemini, etc.)
- **Consensus Workflow**: Three-agent review loop ensures plan quality
- **Sequential Execution**: Architect and Critic reviews run sequentially (never parallel)
- **Graceful Degradation**: Falls back to local analysis when subagents unavailable
- **Configurable Timeouts**: Environment variable-based timeout configuration
- **Pre-Execution Gate**: Prevents underspecified execution requests

### What Problem Does It Solve?

When you have a complex task that needs careful planning before execution, omo-ralplan:

1. Creates an initial plan with multiple viable options
2. Has an Architect review for structural soundness
3. Has a Critic evaluate against quality criteria
4. Iterates until all reviewers approve
5. Hands off to execution mode (ralph or team)

---

## Installation

### Prerequisites

- Oh-My-OpenCode installed and configured
- Access to subagent types: `oracle`, `explore`, `librarian`, `metis`, `momus`

### Step 1: Copy Skill Files

```bash
# Create skill directory if it doesn't exist
mkdir -p ~/.agents/skills/omo-ralplan

# Copy the skill file
cp SKILL.md ~/.agents/skills/omo-ralplan/
```

### Step 2: Verify Configuration

Ensure your `oh-my-opencode.json` includes the required agents:

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

### Step 3: Verify Installation

```bash
# Check that the skill is recognized
ls -la ~/.agents/skills/omo-ralplan/

# Should show:
# SKILL.md
# README.md
```

### Step 4: Test the Skill

```
$omo-ralplan "Add user authentication to the REST API"
```

If the skill is properly installed, you should see the consensus planning workflow begin.

---

## Quick Start

### Basic Usage

```
$omo-ralplan "task description"
```

### Interactive Mode (Recommended for First Use)

```
$omo-ralplan --interactive "Add rate limiting to the API"
```

### High-Risk Tasks

```
$omo-ralplan --deliberate "Migrate database from PostgreSQL to MySQL"
```

---

## How It Works

### The Consensus Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                     OMO-RALPLAN WORKFLOW                        │
└─────────────────────────────────────────────────────────────────┘

Step 1: PLANNER
    │
    │  Creates initial plan with:
    │  • Principles (3-5)
    │  • Decision Drivers (top 3)
    │  • Viable Options (≥2)
    │  • Pros/Cons for each option
    │
    ▼
Step 2: USER FEEDBACK (--interactive only)
    │
    │  Options: Proceed / Request Changes / Skip Review
    │
    ▼
Step 3: ARCHITECT REVIEW (BLOCKING)
    │
    │  Reviews for:
    │  • Architectural soundness
    │  • Steelman counterargument
    │  • Tradeoff tensions
    │  • Synthesis paths
    │  • Principle violations (deliberate mode)
    │
    │  Returns: APPROVE / ITERATE / REJECT
    │
    ▼
Step 4: CRITIC EVALUATION (BLOCKING - after Step 3)
    │
    │  Evaluates:
    │  • Principle-option consistency
    │  • Fair alternative exploration
    │  • Risk mitigation clarity
    │  • Testable acceptance criteria
    │  • Concrete verification steps
    │
    │  Returns: APPROVE / ITERATE / REJECT
    │
    ▼
Step 5: RE-REVIEW LOOP (if not APPROVE)
    │
    │  Max 5 iterations:
    │  a. Collect feedback
    │  b. Revise plan
    │  c. Return to Architect
    │  d. Return to Critic
    │
    ▼
Step 6: FINAL APPROVAL (--interactive only)
    │
    │  User chooses:
    │  • Approve (ralph) - sequential execution
    │  • Approve (team) - parallel execution
    │  • Request changes
    │  • Reject
    │
    ▼
Step 7: EXECUTION HANDOFF
    │
    │  Invokes $ralph or $team with:
    │  • Agent roster
    │  • Reasoning guidance
    │  • Staffing allocation
    │  • Verification path
    │
    ▼
    DONE
```

### Critical Rule: Sequential Execution

**Steps 3 and 4 MUST run sequentially.** The Critic evaluation cannot begin until the Architect review completes.

```
❌ WRONG: Parallel execution
   task(subagent_type="oracle", run_in_background=true, ...) // Architect
   task(subagent_type="oracle", run_in_background=true, ...) // Critic

✅ CORRECT: Sequential execution
   task(subagent_type="oracle", run_in_background=false, ...) // Architect
   // WAIT for result
   task(subagent_type="oracle", run_in_background=false, ...) // Critic
```

---

## Usage Guide

### Command Syntax

```
$omo-ralplan [flags] "task description"
```

### Flags

| Flag | Description | Use Case |
|------|-------------|----------|
| `--interactive` | Enables user prompts at key decision points | When you want control over the planning process |
| `--deliberate` | Forces deliberate mode for high-risk work | Security, migrations, destructive changes, compliance |

### When to Use Each Mode

| Mode | When to Use | Example |
|------|-------------|---------|
| **Default** | Standard tasks, automated workflow | `$omo-ralplan "Add pagination to user list"` |
| **Interactive** | Complex decisions, need oversight | `$omo-ralplan --interactive "Design authentication system"` |
| **Deliberate** | High-risk, security, migrations | `$omo-ralplan --deliberate "Migrate to new database schema"` |

### Deliberate Mode Auto-Triggers

Deliberate mode automatically enables when the request contains:

- Authentication/Security keywords
- Migration keywords
- Destructive change indicators
- Production incident references
- Compliance/PII mentions
- Public API breakage signals

---

## Examples

### Example 1: Basic Planning

```
$omo-ralplan "Add caching layer to the API"
```

**Output:**
```
Step 1: Planner creates initial plan
  - Principles: Performance, Consistency, Simplicity
  - Drivers: Response time, Cache invalidation, Memory usage
  - Options: Redis, Memcached, In-memory cache

Step 3: Architect review (BLOCKING)
  - Verdict: ITERATE
  - Feedback: Consider cache stampede protection

Step 4: Critic evaluation (BLOCKING)
  - Verdict: APPROVE
  - Feedback: Add cache hit rate monitoring

Final Plan: Redis with stampede protection and monitoring
```

### Example 2: Interactive Mode

```
$omo-ralplan --interactive "Implement user roles and permissions"
```

**Workflow:**
```
Step 1: Planner creates plan
  - Principles: Security, Flexibility, Auditability
  - Options: RBAC, ABAC, Hybrid

Step 2: User feedback requested
  > Proceed to review / Request changes / Skip review?
  [User selects: Proceed to review]

Step 3: Architect review
  - Verdict: APPROVE with suggestions

Step 4: Critic evaluation
  - Verdict: APPROVE

Step 6: Final approval requested
  > Approve (ralph) / Approve (team) / Request changes / Reject?
  [User selects: Approve (ralph)]

Step 7: Handoff to $ralph for execution
```

### Example 3: High-Risk Task

```
$omo-ralplan --deliberate "Migrate from REST to GraphQL"
```

**Deliberate Mode Additions:**
```
Pre-mortem scenarios:
1. Performance regression on complex queries
2. Breaking changes for mobile clients
3. Authentication header handling differences

Expanded test plan:
- Unit tests: Query parsing, field resolution
- Integration tests: API gateway, auth layer
- E2E tests: Client compatibility
- Observability: Query complexity metrics, error rates
```

### Example 4: Pre-Execution Gate

**Underspecified request:**
```
$omo-ralplan "improve the app"
```

**Gate response:**
```
⚠️  Request is underspecified. Redirecting to ralplan for planning.

Please provide more detail:
- Which aspect of the app? (performance, UX, security)
- What is the desired outcome?
- Any constraints or preferences?
```

**Well-specified request:**
```
$omo-ralplan "improve the app" --file src/App.tsx
```

**Gate passes:** File reference provides concrete anchor.

---

## Skill Datasheet

### Metadata

| Field | Value |
|-------|-------|
| **Name** | `omo-ralplan` |
| **Version** | 1.0.0 |
| **Category** | Planning |
| **Type** | Skill |
| **Scope** | User-level |
| **Dependencies** | oracle, explore, librarian subagents |

### Arguments

| Argument | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `task_description` | string | Yes | - | The task to plan |
| `--interactive` | flag | No | false | Enable user prompts |
| `--deliberate` | flag | No | false | Force deliberate mode |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OMX_CONSENSUS_AGENT_TIMEOUT_MS` | 120000 | Per-agent call timeout (ms) |
| `OMX_CONSENSUS_TOTAL_TIMEOUT_MS` | 600000 | Total workflow timeout (ms) |
| `OMX_ASK_USER_TIMEOUT_MS` | 300000 | User response timeout (ms) |
| `OMX_CONSENSUS_MAX_REVIEW_ITERATIONS` | 5 | Max re-review loops |
| `OMX_CONSENSUS_CIRCUIT_BREAKER_THRESHOLD` | 3 | Same error recurrence limit |

### Subagent Types Used

| Subagent | Purpose | When Used |
|----------|---------|-----------|
| `oracle` | Architecture review, quality evaluation | Steps 3, 4 (Architect, Critic) |
| `explore` | Codebase search, pattern discovery | Pre-context intake |
| `librarian` | External documentation lookup | When external refs needed |
| `metis` | Pre-planning analysis | Ambiguity detection |
| `momus` | Plan review | Quality verification |

### Categories Available

| Category | Use Case |
|----------|----------|
| `visual-engineering` | UI/UX work |
| `ultrabrain` | Hard logic problems |
| `deep` | Autonomous analysis |
| `quick` | Trivial changes |
| `artistry` | Creative solutions |
| `writing` | Documentation |

### Output Format

The skill produces a structured plan containing:

```markdown
# Plan: [Task Name]

## ADR (Architecture Decision Record)
- **Decision**: [What was decided]
- **Drivers**: [Why this decision]
- **Alternatives Considered**: [Other options]
- **Why Chosen**: [Rationale]
- **Consequences**: [Impact]
- **Follow-ups**: [Next steps]

## Agent Roster
- Available agent types and their roles

## Staffing Guidance
- Recommended agent allocation for execution

## Reasoning Levels
- Suggested reasoning effort by task lane

## Launch Hints
- Explicit $ralph or $team invocation details

## Verification Path
- How to verify the plan was executed correctly
```

---

## Configuration

### Required Agents

```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    },
    "explore": {
      "mode": "subagent",
      "category": "search",
      "tools": {
        "read": true,
        "grep": true,
        "glob": true
      }
    },
    "librarian": {
      "mode": "subagent",
      "category": "research",
      "tools": {
        "webfetch": true,
        "websearch": true,
        "read": true
      }
    }
  }
}
```

### Required Categories

```json
{
  "categories": {
    "visual-engineering": {},
    "ultrabrain": {},
    "deep": {},
    "quick": {},
    "artistry": {},
    "writing": {}
  }
}
```

### Timeout Configuration

Override defaults via environment variables:

```bash
# Increase agent timeout for complex reviews
export OMX_CONSENSUS_AGENT_TIMEOUT_MS=300000

# Increase total workflow timeout
export OMX_CONSENSUS_TOTAL_TIMEOUT_MS=900000

# Decrease max iterations for faster feedback
export OMX_CONSENSUS_MAX_REVIEW_ITERATIONS=3
```

---

## Troubleshooting

### Common Issues

#### Issue: "oracle subagent unavailable"

**Cause:** Oracle agent not configured in oh-my-opencode.json

**Solution:**
```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant"
    }
  }
}
```

#### Issue: Workflow hangs at Architect step

**Cause:** Timeout too short for complex review

**Solution:**
```bash
export OMX_CONSENSUS_AGENT_TIMEOUT_MS=300000
```

#### Issue: "Task is underspecified" gate triggers

**Cause:** Request lacks concrete anchors

**Solution:** Add one of:
- File path: `src/api/auth.ts`
- Function name: `processLogin`
- Issue number: `#42`
- Acceptance criteria

#### Issue: Critic keeps rejecting plan

**Cause:** Plan missing required elements

**Solution:** Ensure plan includes:
- Testable acceptance criteria
- Risk mitigation steps
- Concrete verification steps
- Fair alternative exploration

### Fallback Behavior

When subagents are unavailable:

```
1. Try: task(subagent_type="oracle", ...)
2. Fallback: Local analysis with read/grep/glob
3. Fallback: Use category="deep" for autonomous analysis
4. Last resort: Present plan with "expert review unavailable" warning
```

---

## Architecture

### Component Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    OMO-Ralplan Skill                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │   Planner   │───▶│  Architect  │───▶│   Critic    │      │
│  │   (Local)   │    │  (Oracle)   │    │  (Oracle)   │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
│         │                  │                  │              │
│         │                  │                  │              │
│         ▼                  ▼                  ▼              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Pre-Execution Gate                      │    │
│  │  • Validates task specificity                       │    │
│  │  • Redirects underspecified requests                │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Execution Handoff                       │    │
│  │  • $ralph (sequential)                              │    │
│  │  • $team (parallel)                                 │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request
     │
     ▼
┌─────────────────┐
│ Pre-Context     │ ──▶ .omx/context/{slug}-*.md
│ Intake          │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ Planner         │ ──▶ Initial Plan + RALPLAN-DR
│ (Local)         │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ Architect       │ ──▶ Review + Verdict
│ (Oracle)        │     (APPROVE/ITERATE/REJECT)
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ Critic          │ ──▶ Evaluation + Verdict
│ (Oracle)        │     (APPROVE/ITERATE/REJECT)
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ Final Plan      │ ──▶ .omx/plans/ralplan-*.md
│ Output          │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ Execution       │ ──▶ $ralph or $team
│ Handoff         │
└─────────────────┘
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|--------------|----------------|------------------|
| Using `ask_codex` | MCP tool may not exist | Use `task(subagent_type="oracle", ...)` |
| Hardcoding model names | Not model-agnostic | Reference configuration |
| Parallel consensus calls | Breaks sequential dependency | Run Architect → wait → Critic |
| Hardcoded timeouts | Not configurable | Use environment variables |
| Skipping pre-context | Ungrounded planning | Always run pre-context intake |

---

## Summary

OMO-Ralplan provides:

1. **Native `task()` delegation** - No MCP tool dependencies
2. **Model-agnostic design** - Works with any configured model
3. **Sequential consensus** - Proper Architect → Critic ordering
4. **Mandatory parameters** - `load_skills=[]`, `run_in_background`
5. **Configurable timeouts** - Environment variable based
6. **Graceful fallbacks** - Local analysis when subagents unavailable
7. **Clear configuration** - Documented agent/category requirements

This ensures the skill works across different oh-my-opencode configurations regardless of the underlying AI models.

---

## License

Part of the Oh-My-OpenCode project.

---

## Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Verify your [Configuration](#configuration)
3. Review the [Examples](#examples) for usage patterns
