# Auth Service Runbook

## Overview
Handles authentication, authorization, session management.
Session store: Redis (cluster mode).
Identity provider: Internal LDAP + OAuth (Google, GitHub).

## Common Issues

### Login Failures
1. Check Redis connectivity: `redis-cli -h auth-redis-cluster.internal -p 6379 ping`
2. Check LDAP sync status: `curl http://auth-service:8080/health/ldap`
3. If OAuth failures, verify callback URLs in provider dashboard.

### Session Store Issues
1. Redis connection string: `redis://auth-redis-cluster.internal:6379/0`
2. Check memory usage: `redis-cli info memory`
3. If memory > 80%, check for session leak (sessions not expiring).

## IMPORTANT NOTE
Redis was migrated to cluster mode on Feb 15.
Old config: `redis://auth-redis.internal:6379`
New config: `redis://auth-redis-cluster.internal:6379`
If you see connection errors, verify the service is pointing to the cluster endpoint.

## Escalation
- P1/P2: Page Identity oncall
- P3/P4: Create ticket, assign to Identity team
