# Architecture Decision Record

## ADR-001: Process bank payments asynchronously

**Status:** Accepted  
**Date:** 2026-07-22

### Context

A synchronous call to a bank API can delay the client response and leaves the payment state uncertain on timeout. Fast payment acceptance conflicts with simple synchronous recovery from bank failures.

### Decision

After a payment request is durably stored, the API returns the payment identifier with status `pending`. A separate process sends the payment to the bank, retries temporary failures, and updates the final status after confirmation from the bank.

### Consequences

- The client receives a fast, predictable response.
- A bank timeout does not cause the payment to be marked as failed prematurely.
- The system needs idempotency, retry logic, status reconciliation, and monitoring of pending payments.
