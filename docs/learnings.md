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

### 2. Model Limit Handling in Delegations

**Learning**: When delegated agents reach model limits (context window, token limits, rate limits), the system must handle gracefully.

**Types of Model Limits**:

| Limit Type | Cause | Behavior | Handling |
|------------|-------|----------|----------|
| **Context Window** | Input exceeds model's context limit | Task fails with context overflow error | Truncate input or use chunking |
| **Token Limit** | Output exceeds max tokens | Response truncated mid-generation | Use continuation or reduce scope |
| **Rate Limit** | Too many requests per minute | HTTP 429 error, task rejected | Retry with exponential backoff |
| **Quota Limit** | Daily/monthly quota exceeded | Task rejected with quota error | Queue task or use fallback model |

**What Happens When Limits Are Hit**:

```typescript
// When model reaches context window limit:
// 1. Task fails with error
// 2. Error propagates to calling agent
// 3. Workflow must decide: retry, fallback, or abort

// Example error handling:
try {
  const result = await task({
    subagent_type: "oracle",
    load_skills: [],
    run_in_background: false,
    prompt: largePrompt
  });
} catch (error) {
  if (error.code === 'CONTEXT_LIMIT_EXCEEDED') {
    // Option 1: Truncate and retry
    const truncatedPrompt = truncatePrompt(largePrompt, maxTokens);
    return task({ ...params, prompt: truncatedPrompt });
    
    // Option 2: Use chunking
    const chunks = chunkPrompt(largePrompt);
    return processChunks(chunks);
    
    // Option 3: Fallback to local analysis
    return localAnalysis(largePrompt);
  }
  
  if (error.code === 'RATE_LIMIT') {
    // Retry with exponential backoff
    await sleep(calculateBackoff(retryCount));
    return task(params);
  }
}
```

**Graceful Degradation for Model Limits**:

```
Model Limit Hit
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    ERROR DETECTED                            │
│  • Context window exceeded                                   │
│  • Token limit reached                                       │
│  • Rate limit hit                                            │
│  • Quota exceeded                                            │
└─────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    RECOVERY STRATEGY                         │
│                                                              │
│  1. RETRY (if rate limit)                                    │
│     └─ Wait + exponential backoff                            │
│                                                              │
│  2. TRUNCATE (if context limit)                              │
│     └─ Reduce input size, preserve key context               │
│                                                              │
│  3. CHUNK (if large input)                                   │
│     └─ Split into smaller tasks, aggregate results           │
│                                                              │
│  4. FALLBACK (if quota/quota)                                │
│     └─ Use local analysis with read/grep/glob                │
│                                                              │
│  5. ABORT (if unrecoverable)                                 │
│     └─ Report error to user with context                     │
└─────────────────────────────────────────────────────────────┘
```

**Best Practices for Avoiding Model Limits**:

1. **Pre-flight Check**: Estimate token count before delegation
2. **Incremental Processing**: Process large inputs in chunks
3. **Context Pruning**: Remove irrelevant context before delegation
4. **Summary First**: Summarize large documents before analysis
5. **Fallback Ready**: Always have local analysis fallback

**Implementation in OMO-Ralplan**:

```typescript
// Pre-flight token estimation
function estimateTokens(prompt: string): number {
  // Rough estimate: ~4 chars per token
  return Math.ceil(prompt.length / 4);
}

// Check before delegation
const estimatedTokens = estimateTokens(planContent);
const modelContextLimit = getModelContextLimit('oracle');

if (estimatedTokens > modelContextLimit * 0.8) {
  // Use 80% threshold for safety
  console.warn('Approaching context limit, truncating...');
  planContent = truncateToLimit(planContent, modelContextLimit * 0.8);
}

// Delegate with confidence
task({
  subagent_type: "oracle",
  load_skills: [],
  run_in_background: false,
  prompt: planContent
});
```

**Key Insight**: Model limits are not failures—they're constraints to design around. Always have a fallback strategy.

### 2.1. Agent Freezing on Time-Based Limits (Critical Issue)

**Problem**: When time-based model limits are reached (e.g., GPT-5h limit), agents can become **frozen** and block the entire workflow.

**What Happens**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIME-BASED LIMIT REACHED                     │
│                                                                 │
│  Example: "You've reached your GPT-5h limit"                    │
│                                                                 │
│  Agent State: FROZEN                                            │
│  - Cannot complete current task                                 │
│  - Cannot accept new tasks                                      │
│  - Workflow is BLOCKED                                          │
│  - No automatic recovery                                        │
└─────────────────────────────────────────────────────────────────┘
```

**Why This Is Critical**:

| Impact | Description | Severity |
|--------|-------------|----------|
| **Workflow Blocked** | Entire consensus workflow stops | 🔴 Critical |
| **No Auto-Recovery** | System doesn't automatically switch models | 🔴 Critical |
| **User Unaware** | No notification that agent is frozen | 🟡 High |
| **Time Waste** | Waiting for limit to reset (hours) | 🟡 High |
| **Cascading Failure** | Other agents waiting on frozen agent | 🟡 High |

**Detection**:

```typescript
// Signs that an agent is frozen due to time-based limit:
// 1. Task hangs indefinitely (no timeout)
// 2. Error message contains "limit reached" or "quota exceeded"
// 3. Agent status shows "frozen" or "blocked"
// 4. No response after timeout period

// Example frozen agent detection:
const result = await task({
  subagent_type: "oracle",
  load_skills: [],
  run_in_background: false,
  prompt: "Review this plan..."
});

// If task hangs beyond timeout, agent may be frozen
// Timeout should be set via OMX_CONSENSUS_AGENT_TIMEOUT_MS
```

**Recovery Strategies**:

```
┌─────────────────────────────────────────────────────────────────┐
│                 FROZEN AGENT RECOVERY OPTIONS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION 1: TIMEOUT + FALLBACK (Recommended)                     │
│  ─────────────────────────────────────────                      │
│  • Set aggressive timeout (OMX_CONSENSUS_AGENT_TIMEOUT_MS)      │
│  • On timeout, immediately switch to fallback                   │
│  • Use local analysis or alternative subagent                   │
│                                                                 │
│  OPTION 2: MODEL SWITCHING                                      │
│  ─────────────────────────────────────────                      │
│  • Configure backup model in oh-my-opencode.json                │
│  • Automatically retry with backup model                        │
│  • Requires pre-configuration of alternative models             │
│                                                                 │
│  OPTION 3: USER NOTIFICATION                                    │
│  ─────────────────────────────────────────                      │
│  • Detect frozen state                                          │
│  • Notify user: "Agent frozen, limit reached"                   │
│  • Offer options: wait, switch, abort                           │
│                                                                 │
│  OPTION 4: PARALLEL DELEGATION (Prevention)                     │
│  ─────────────────────────────────────────                      │
│  • Delegate to multiple agents in parallel                      │
│  • Use first successful response                                │
│  • Reduces single-point-of-failure risk                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation: Timeout + Fallback**:

```typescript
// Recommended approach: Timeout + Fallback
async function delegateWithFallback(prompt: string) {
  const TIMEOUT_MS = 120000; // 2 minutes
  
  try {
    // Attempt primary delegation with timeout
    const result = await Promise.race([
      task({
        subagent_type: "oracle",
        load_skills: [],
        run_in_background: false,
        prompt: prompt
      }),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('TIMEOUT')), TIMEOUT_MS)
      )
    ]);
    
    return result;
    
  } catch (error) {
    // Agent may be frozen due to time-based limit
    if (error.message === 'TIMEOUT' || error.code === 'LIMIT_REACHED') {
      console.warn('⚠️ Agent frozen or limit reached, using fallback');
      
      // Fallback 1: Try alternative subagent
      try {
        return await task({
          subagent_type: "metis", // Alternative agent
          load_skills: [],
          run_in_background: false,
          prompt: prompt
        });
      } catch (fallbackError) {
        // Fallback 2: Local analysis
        console.warn('⚠️ Fallback agent also failed, using local analysis');
        return localAnalysis(prompt);
      }
    }
    
    throw error; // Re-throw unexpected errors
  }
}
```

**Configuration for Model Switching**:

```json
{
  "agents": {
    "oracle": {
      "mode": "subagent",
      "category": "consultant",
      "model": "gpt-4",
      "fallback_model": "claude-3",  // Backup model
      "fallback_on_limit": true,      // Auto-switch on limit
      "timeout_ms": 120000            // Timeout before fallback
    }
  }
}
```

**Prevention: Parallel Delegation**:

```typescript
// Prevent frozen agent blocking workflow
async function delegateWithRedundancy(prompt: string) {
  // Delegate to multiple agents in parallel
  const results = await Promise.allSettled([
    task({ subagent_type: "oracle", load_skills: [], run_in_background: true, prompt }),
    task({ subagent_type: "metis", load_skills: [], run_in_background: true, prompt }),
  ]);
  
  // Use first successful result
  const successful = results.find(r => r.status === 'fulfilled');
  
  if (successful) {
    return successful.value;
  }
  
  // All failed - use local analysis
  return localAnalysis(prompt);
}
```

**Best Practices for Frozen Agent Prevention**:

| # | Practice | Implementation |
|---|----------|----------------|
| 1 | **Set aggressive timeouts** | `OMX_CONSENSUS_AGENT_TIMEOUT_MS=120000` |
| 2 | **Configure fallback models** | Add `fallback_model` to agent config |
| 3 | **Use parallel delegation** | Delegate to multiple agents simultaneously |
| 4 | **Implement circuit breaker** | Track failures, skip frozen agents |
| 5 | **Notify user immediately** | Don't let workflow hang silently |
| 6 | **Have local analysis ready** | Always have non-model fallback |

**Circuit Breaker Pattern**:

```typescript
// Track agent failures to detect frozen agents
const agentHealth = {
  oracle: { failures: 0, lastFailure: null },
  metis: { failures: 0, lastFailure: null },
};

const CIRCUIT_BREAKER_THRESHOLD = 3;

function isAgentFrozen(agentType: string): boolean {
  const health = agentHealth[agentType];
  
  // Check if agent has too many recent failures
  if (health.failures >= CIRCUIT_BREAKER_THRESHOLD) {
    const timeSinceLastFailure = Date.now() - health.lastFailure;
    
    // If failures happened within 5 minutes, agent is likely frozen
    if (timeSinceLastFailure < 300000) {
      return true;
    }
  }
  
  return false;
}

// Use circuit breaker before delegation
async function delegateWithCircuitBreaker(agentType: string, prompt: string) {
  if (isAgentFrozen(agentType)) {
    console.warn(`⚠️ Agent ${agentType} appears frozen, using fallback`);
    return localAnalysis(prompt);
  }
  
  try {
    const result = await task({
      subagent_type: agentType,
      load_skills: [],
      run_in_background: false,
      prompt
    });
    
    // Reset failure count on success
    agentHealth[agentType].failures = 0;
    return result;
    
  } catch (error) {
    // Track failure
    agentHealth[agentType].failures++;
    agentHealth[agentType].lastFailure = Date.now();
    
    throw error;
  }
}
```

**Key Insight**: Time-based limits cause **agent freezing**, which is more severe than rate limits. Always implement timeout + fallback + circuit breaker patterns to prevent workflow blocking.

### 3. Sequential Consensus Workflow

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

### 4. Environment Variable Configuration

**Learning**: Never hardcode timeouts. Use environment variables for configurability.

**Implementation**:
| Variable | Default | Description |
|----------|---------|-------------|
| `OMX_CONSENSUS_AGENT_TIMEOUT_MS` | 120000 | Per-agent call timeout |
| `OMX_CONSENSUS_TOTAL_TIMEOUT_MS` | 600000 | Total workflow timeout |
| `OMX_CONSENSUS_MAX_REVIEW_ITERATIONS` | 5 | Max re-review loops |

### 5. Graceful Degradation Strategy

**Learning**: Skills must handle cases where subagents are unavailable.

**Degradation Path**:
1. Try: `task(subagent_type="oracle", ...)`
2. Fallback: Local analysis with read/grep/glob
3. Fallback: Use `category="deep"` for autonomous analysis
4. Last resort: Present plan with "expert review unavailable" warning

### 6. Pre-Execution Gate Pattern

**Learning**: Underspecified execution requests waste agent cycles.

**Solution**: Gate intercepts vague requests and redirects to planning.

**Pass Signals**:
- File path reference
- Function/class name
- Issue/PR number
- Numbered steps
- Acceptance criteria

**Gate Bypass**: `force:` or `!` prefix

### 7. Skill File Structure

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

### 8. Mandatory Task Parameters

**Learning**: All `task()` calls must include specific parameters.

**Required Parameters**:
- `load_skills=[]` - Always include, even if empty
- `run_in_background` - Explicit true/false
- `subagent_type` or `category` - Never omit

### 9. Documentation Completeness

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
