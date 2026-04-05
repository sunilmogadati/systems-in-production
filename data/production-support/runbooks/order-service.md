# Order Service Runbook

## Overview
Handles order creation, validation, and lifecycle management.
Database: PostgreSQL (primary), Redis (cache).

## Common Issues

### High Error Rate
1. Check database connection pool: `SELECT count(*) FROM pg_stat_activity WHERE datname='orders';`
2. Normal range: 20-50 connections. If above 180, connection pool is near exhaustion.
3. Check for long-running queries: `SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE state != 'idle' ORDER BY duration DESC;`
4. If deadlocks detected, check recent deployments for schema changes.

### After Deployment Failures
1. Verify current version: `kubectl get deployment order-service -o jsonpath='{.spec.template.spec.containers[0].image}'`
2. Rollback: `kubectl rollout undo deployment/order-service`
3. IMPORTANT: Clear Redis cache after rollback. Old cache entries can cause stale validation errors.
4. Monitor error rate for 30 minutes after rollback.

### Database Connection Pool Exhaustion
1. This has happened before (March 5). Root cause was a missing connection timeout in the ORM config.
2. Temporary fix: restart pods to release connections.
3. Permanent fix: set `connection_max_lifetime=300s` in database config.
4. Watch for this recurring. If it happens again, the ORM config may have been overwritten by a deployment.

## Escalation
- P1/P2: Page Commerce oncall immediately
- P3/P4: Create ticket, assign to Commerce team
