# Payment Service Runbook

## Overview
Handles payment processing, refunds, and reconciliation.
Payment gateways: Stripe (primary), PayPal (secondary).
Database: PostgreSQL with strict ACID compliance.

## Common Issues

### Transaction Failures
1. Check gateway status: https://status.stripe.com
2. Check for idempotency key collisions in logs.
3. Verify TLS certificates are current.

### Batch Job Conflicts (2-3 AM)
1. Daily reconciliation batch runs at 2:00 AM.
2. This job locks the transactions table for bulk updates.
3. Any real-time payment attempts during this window may timeout.
4. Known issue. Mitigation: batch job should use row-level locking, not table-level.
5. TODO: Fix has been scheduled for Q2 2026 but not yet implemented.

### Duplicate Charges
1. Check idempotency key implementation.
2. Common cause: client-side retry without idempotency key.
3. Run reconciliation query to identify affected transactions.

## PCI Compliance
- NEVER log full card numbers.
- If PAN detected in logs, escalate to Security immediately.
- Mask all card data before logging: first 6, last 4 only.

## Escalation
- P1/P2: Page Commerce oncall + FinOps
- P3/P4: Create ticket, assign to Commerce team
