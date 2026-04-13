# Deliberate Mode Examples

Deliberate mode adds extra rigor for high-risk tasks: pre-mortem analysis and expanded test planning.

## When to Use

Deliberate mode auto-triggers for:
- Authentication/Security keywords
- Migration keywords
- Destructive change indicators
- Production incident references
- Compliance/PII mentions
- Public API breakage signals

Or force it with `--deliberate` flag.

## Example 1: Database Migration

```
$omo-ralplan --deliberate "Migrate from MySQL to PostgreSQL"
```

**Deliberate Mode Additions:**

### Pre-Mortem Scenarios

```
1. Data loss during migration
   - Mitigation: Full backup before migration
   - Detection: Row count validation
   - Recovery: Restore from backup

2. Performance regression post-migration
   - Mitigation: Load testing before cutover
   - Detection: APM monitoring
   - Recovery: Feature flag to rollback queries

3. Breaking existing integrations
   - Mitigation: API compatibility layer
   - Detection: Integration test suite
   - Recovery: Compatibility mode toggle
```

### Expanded Test Plan

```
Unit Tests:
- Query translation validation
- Data type conversion
- Constraint verification

Integration Tests:
- API endpoint compatibility
- Transaction handling
- Connection pooling

E2E Tests:
- Full user journey with new database
- Performance benchmarks
- Failover scenarios

Observability:
- Query performance metrics
- Connection pool health
- Migration progress tracking
- Error rate monitoring
```

## Example 2: Authentication Overhaul

```
$omo-ralplan --deliberate "Replace session auth with JWT"
```

**Pre-Mortem Scenarios:**

```
1. Token theft vulnerability
   - Mitigation: Short-lived tokens + refresh tokens
   - Detection: Anomaly detection on token usage
   - Recovery: Token revocation endpoint

2. Breaking mobile app authentication
   - Mitigation: Backward compatibility layer
   - Detection: Mobile app test suite
   - Recovery: Fallback to session auth

3. Token refresh race conditions
   - Mitigation: Atomic refresh operations
   - Detection: Concurrent request testing
   - Recovery: Request deduplication
```

**Expanded Test Plan:**

```
Unit Tests:
- Token generation/validation
- Refresh token rotation
- Signature verification

Integration Tests:
- Auth middleware chain
- Token blacklisting
- Rate limiting

E2E Tests:
- Login/logout flows
- Token refresh flows
- Multi-device sessions

Observability:
- Token generation rate
- Refresh failure rate
- Authentication latency
- Token validation errors
```

## Example 3: API Breaking Changes

```
$omo-ralplan --deliberate "Remove deprecated API endpoints"
```

**Pre-Mortem Scenarios:**

```
1. External integrations break
   - Mitigation: Deprecation notices + migration guide
   - Detection: Usage analytics
   - Recovery: Temporary re-enable with warnings

2. Internal services fail
   - Mitigation: Service inventory audit
   - Detection: Service mesh telemetry
   - Recovery: Feature flag rollback

3. Documentation outdated
   - Mitigation: Doc review before deprecation
   - Detection: Doc link checker
   - Recovery: Redirect pages
```

## Deliberate Mode Checklist

Before approving a deliberate mode plan, verify:

- [ ] At least 3 pre-mortem scenarios
- [ ] Each scenario has mitigation, detection, recovery
- [ ] Expanded test plan covers unit/integration/e2e/observability
- [ ] Rollback strategy is documented
- [ ] Monitoring/alerting is defined
