# Session Learnings & Design Decisions

This document captures key learnings and design decisions made during the development of OMO-Ralplan.

---

## Session Overview

**Date**: 2025-04-13
**Task**: Create a model-agnostic clone of `/ralplan` skill for OpenCode compatibility
**Result**: Successfully created and published OMO-Ralplan skill

---

## Key Learnings

### 1. Model-Agnostic Design Principles

**Learning**: Skills should never reference specific AI models or MCP tools that may not exist.

**Implementation**:
- Use native `task()` tool for all agent delegation
- Reference configuration, not model names
- Document reasoning effort expectations, not model names

**Code Pattern**:
```typescript
// ✅ CORRECT: Native task delegation
task(
  subagent_type="oracle",
  load_skills=[],
  run_in_background=false,
  prompt="..."
)

// ❌ WRONG: MCP tool that may not exist
ask_codex(agent_role: "architect", ...)
```

### 2. Sequential Consensus Workflow

**Learning**: Architect and Critic reviews MUST run sequentially, never in parallel.

**Why**: The Critic evaluation depends on Architect feedback. Parallel execution breaks this dependency.

**Implementation**:
```typescript
// Step 3: Architect review (BLOCKING)
task(subagent_type="oracle", run_in_background=false, ...)

// WAIT for result before Step 4

// Step 4: Critic evaluation (BLOCKING)
task(subagent_type="oracle", run_in_background=false, ...)
```

### 3. Environment Variable Configuration

**Learning**: Never hardcode timeouts. Use environment variables for configurability.

**Implementation**:
| Variable | Default | Description |
|----------|---------|-------------|
| `OMX_CONSENSUS_AGENT_TIMEOUT_MS` | 120000 | Per-agent call timeout |
| `OMX_CONSENSUS_TOTAL_TIMEOUT_MS` | 600000 | Total workflow timeout |
| `OMX_CONSENSUS_MAX_REVIEW_ITERATIONS` | 5 | Max re-review loops |

### 4. Graceful Degradation Strategy

**Learning**: Skills must handle cases where subagents are unavailable.

**Degradation Path**:
1. Try: `task(subagent_type="oracle", ...)`
2. Fallback: Local analysis with read/grep/glob
3. Fallback: Use `category="deep"` for autonomous analysis
4. Last resort: Present plan with "expert review unavailable" warning

### 5. Pre-Execution Gate Pattern

**Learning**: Underspecified execution requests waste agent cycles.

**Solution**: Gate intercepts vague requests and redirects to planning.

**Pass Signals**:
- File path reference
- Function/class name
- Issue/PR number
- Numbered steps
- Acceptance criteria

**Gate Bypass**: `force:` or `!` prefix

### 6. Skill File Structure

**Learning**: Skills need proper frontmatter and directory structure.

**Required Structure**:
```
.agents/skills/omo-ralplan/
├── SKILL.md          # Main skill definition (required)
└── README.md         # Documentation (recommended)
```

**Frontmatter**:
```yaml
---
name: omo-ralplan
description: Consensus planning with Planner/Architect/Critic loop
---
```

### 7. Mandatory Task Parameters

**Learning**: All `task()` calls must include specific parameters.

**Required Parameters**:
- `load_skills=[]` - Always include, even if empty
- `run_in_background` - Explicit true/false
- `subagent_type` or `category` - Never omit

### 8. Documentation Completeness

**Learning**: Comprehensive documentation reduces support burden.

**Required Sections**:
1. Purpose - What the skill does
2. Usage - How to invoke it
3. Flags - Available options
4. Workflow - Step-by-step process
5. Tool Usage - Mandatory patterns
6. Timeout Configuration - Environment variables
7. Examples - Concrete usage examples
8. Skill Datasheet - All arguments and outputs

---

## Design Decisions

### Decision 1: Native Task Delegation

**Context**: Original ralplan used MCP tools (`ask_codex`)
**Decision**: Use native `task()` tool instead
**Rationale**: MCP tools may not exist in all OpenCode configurations
**Trade-off**: Less specialized, but more portable

### Decision 2: Sequential Consensus

**Context**: Could parallelize Architect and Critic for speed
**Decision**: Keep sequential execution
**Rationale**: Critic depends on Architect feedback
**Trade-off**: Slower, but correct

### Decision 3: Environment Variable Timeouts

**Context**: Could hardcode sensible defaults
**Decision**: Use environment variables
**Rationale**: Different tasks need different timeout profiles
**Trade-off**: Requires configuration, but flexible

### Decision 4: Graceful Degradation

**Context**: Could fail fast when subagents unavailable
**Decision**: Implement fallback strategy
**Rationale**: Skills should work in degraded mode
**Trade-off**: More complex, but resilient

### Decision 5: Pre-Execution Gate

**Context**: Could allow all requests through
**Decision**: Gate underspecified requests
**Rationale**: Prevents wasted agent cycles
**Trade-off**: May frustrate users, but saves resources

---

## Anti-Patterns Identified

### Anti-Pattern 1: MCP Tool References

```markdown
<!-- ❌ NEVER USE -->
- Use `ask_codex` with `agent_role: "architect"`
- Use `mcp__x__ask_codex` with `agent_role: "critic"`
```

### Anti-Pattern 2: Model-Specific Instructions

```markdown
<!-- ❌ NEVER USE -->
- Use GPT-5.4 for architecture review
- Use Claude for analysis
```

### Anti-Pattern 3: Hardcoded Timeouts

```markdown
<!-- ❌ NEVER USE -->
- Wait 120 seconds for agent response
- Timeout after 2 minutes
```

### Anti-Pattern 4: Parallel Consensus Calls

```typescript
// ❌ NEVER DO THIS
task(subagent_type="oracle", run_in_background=true, ...) // Architect
task(subagent_type="oracle", run_in_background=true, ...) // Critic
```

---

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

---

## Future Improvements

1. **Add Unit Tests**: Test consensus workflow logic
2. **Add Integration Tests**: Test with different model configurations
3. **Add Performance Metrics**: Track iteration counts and timeouts
4. **Add More Examples**: Real-world use cases
5. **Add Video Tutorials**: Visual walkthroughs

---

## References

- Original specification: `ralplan-opencode-compatible.md`
- Oh-My-OpenCode documentation: https://github.com/oh-my-opencode
- Skill template: `/ralplan` skill
