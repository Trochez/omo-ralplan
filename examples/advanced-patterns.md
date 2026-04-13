# Advanced Usage Patterns

This document covers advanced patterns for using OMO-Ralplan effectively.

## Pattern 1: Chained Planning

Use ralplan output as input for another ralplan session.

```
# First, plan the overall architecture
$omo-ralplan "Design microservices architecture for e-commerce platform"

# Then, plan specific services
$omo-ralplan "Implement user service based on architecture plan in .omx/plans/microservices-*.md"
$omo-ralplan "Implement order service based on architecture plan in .omx/plans/microservices-*.md"
```

## Pattern 2: Pre-Context Integration

Leverage existing context snapshots for grounded planning.

```
# Create context snapshot first
$deep-interview "What are the requirements for the payment system?"

# Then use ralplan with the context
$omo-ralplan "Implement payment processing"
```

## Pattern 3: Iterative Refinement

Use the re-review loop intentionally for complex decisions.

```
# Start with a rough plan
$omo-ralplan --interactive "Design caching strategy"

# When Architect returns ITERATE:
# - Review the feedback
# - Provide additional context if needed
# - Let the loop refine the plan

# Continue until APPROVE or max iterations
```

## Pattern 4: Multi-Phase Projects

Break large projects into phases with separate ralplan sessions.

```
# Phase 1: Foundation
$omo-ralplan --deliberate "Set up authentication and authorization"

# Phase 2: Core Features
$omo-ralplan "Implement user management"
$omo-ralplan "Implement product catalog"

# Phase 3: Integration
$omo-ralplan "Integrate payment gateway"
$omo-ralplan "Set up notification system"

# Phase 4: Optimization
$omo-ralplan "Add caching layer"
$omo-ralplan "Implement rate limiting"
```

## Pattern 5: Risk-Based Mode Selection

Choose mode based on risk assessment.

| Risk Level | Mode | Example |
|------------|------|---------|
| Low | Default | Add UI component |
| Medium | Interactive | Add new API endpoint |
| High | Deliberate | Database migration |
| Critical | Deliberate + Interactive | Auth system overhaul |

## Pattern 6: Execution Handoff Strategy

Choose between ralph and team based on task characteristics.

### Use Ralph (Sequential) When:

- Dependencies between steps
- Need verification between steps
- Single-threaded work
- Debugging or fixing

### Use Team (Parallel) When:

- Independent tasks
- Multiple files/modules
- Time-sensitive work
- Large refactoring

## Pattern 7: Combining with Other Skills

```
# Research first
$deep-research "Best practices for GraphQL pagination"

# Then plan
$omo-ralplan "Implement GraphQL pagination"

# Then execute
$ralph "Implement the pagination plan"
```

## Pattern 8: Pre-Execution Gate Bypass

When you know exactly what you want:

```
# Bypass gate with force prefix
force: $ralph "Fix the null check in src/auth.ts:42"

# Or with escape character
! $ralph "Update the config"
```

## Pattern 9: Context Preservation

Preserve context across sessions.

```
# Session 1: Initial planning
$omo-ralplan "Design API structure"
# Creates: .omx/context/api-design-*.md

# Session 2: Continue with context
$omo-ralplan "Implement API endpoints"
# Reuses: .omx/context/api-design-*.md
```

## Pattern 10: Quality Gates

Use the consensus workflow as quality gates.

```
# Gate 1: Architect Review
# - Checks architectural soundness
# - Identifies tradeoffs
# - Suggests synthesis

# Gate 2: Critic Evaluation
# - Verifies principle consistency
# - Checks testability
# - Validates risk mitigation

# Gate 3: User Approval (interactive mode)
# - Final decision on execution
# - Choice of execution mode
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Skipping Planning

```
# ❌ Wrong: Jumping straight to execution
$ralph "Add authentication"

# ✅ Correct: Plan first
$omo-ralplan "Add authentication"
# Then execute
$ralph "Implement authentication plan"
```

### Anti-Pattern 2: Vague Requests

```
# ❌ Wrong: Too vague
$omo-ralplan "improve the app"

# ✅ Correct: Specific
$omo-ralplan "Improve API response time by adding caching"
```

### Anti-Pattern 3: Ignoring Feedback

```
# ❌ Wrong: Ignoring ITERATE verdict
# Just proceeding without addressing feedback

# ✅ Correct: Address feedback
# Review Architect/Critic feedback
# Provide additional context
# Let the loop refine
```

### Anti-Pattern 4: Wrong Execution Mode

```
# ❌ Wrong: Using team for dependent tasks
$team "1. Create DB schema 2. Add API 3. Write tests"

# ✅ Correct: Use ralph for dependencies
$ralph "1. Create DB schema 2. Add API 3. Write tests"
```

## Best Practices Summary

1. **Always plan before executing** - Use ralplan for non-trivial tasks
2. **Be specific** - Provide concrete anchors (files, functions, issues)
3. **Use appropriate mode** - Match risk level to mode
4. **Leverage context** - Use pre-context intake
5. **Trust the loop** - Let consensus workflow refine plans
6. **Choose execution wisely** - ralph vs team based on task
7. **Preserve context** - Use .omx/context for continuity
