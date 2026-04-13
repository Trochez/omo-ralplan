# Interactive Mode Examples

Interactive mode gives you control over the planning process with prompts at key decision points.

## When to Use

- Complex decisions requiring human judgment
- First time using the skill
- High-stakes projects
- When you want to review before execution

## Example 1: Authentication System

```
$omo-ralplan --interactive "Implement OAuth2 authentication"
```

**Workflow:**

```
Step 1: Planner creates plan
  - Principles: Security, User Experience, Maintainability
  - Options: Auth0, Firebase Auth, Custom OAuth2

Step 2: User feedback requested
  > Proceed to review / Request changes / Skip review?
  [You select: Proceed to review]

Step 3: Architect review
  - Verdict: ITERATE
  - Feedback: Consider token refresh strategy

Step 4: Critic evaluation
  - Verdict: APPROVE
  - Feedback: Add security audit step

Step 6: Final approval requested
  > Approve (ralph) / Approve (team) / Request changes / Reject?
  [You select: Approve (ralph)]

Step 7: Handoff to $ralph for execution
```

## Example 2: Database Migration

```
$omo-ralplan --interactive "Migrate from MongoDB to PostgreSQL"
```

**Workflow:**

```
Step 1: Planner creates plan
  - Principles: Data Integrity, Minimal Downtime, Reversibility
  - Options: Live migration, Dual-write, Big bang

Step 2: User feedback requested
  > Proceed to review / Request changes / Skip review?
  [You select: Request changes]
  [You provide: Add rollback plan for each phase]

Step 1 (revised): Planner updates plan
  - Added rollback procedures for each migration phase

Step 2: User feedback requested
  > Proceed to review / Request changes / Skip review?
  [You select: Proceed to review]

Step 3-4: Reviews complete
  - Both APPROVE

Step 6: Final approval
  > Approve (ralph) / Approve (team) / Request changes / Reject?
  [You select: Approve (team)]

Step 7: Handoff to $team for parallel execution
```

## Example 3: API Redesign

```
$omo-ralplan --interactive "Redesign REST API to follow JSON:API spec"
```

**Key Decision Points:**

1. **Draft Review** - Review initial plan before expert review
2. **Execution Mode** - Choose between ralph (sequential) or team (parallel)
3. **Request Changes** - Iterate on plan if needed

## Tips for Interactive Mode

1. **Read the RALPLAN-DR summary** - It shows principles, drivers, and options
2. **Use "Request changes"** - Don't hesitate to iterate
3. **Choose execution mode wisely**:
   - `ralph` for sequential, verified execution
   - `team` for parallel, coordinated execution
4. **Review the ADR** - Final plan includes Architecture Decision Record
