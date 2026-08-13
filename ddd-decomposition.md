# Service Decomposition: Corporate Cashflow & Payments Platform

This note defines logical service boundaries for the MVP. The current implementation remains a **modular monolith**: the services below are business modules inside the Core Backend and are deployed together. The boundaries protect ownership of business rules and data now, while keeping an option to extract a service later if independent deployment becomes necessary.

## 1. Business Capabilities and Subdomains

| Business capability | DDD classification | Rationale |
| --- | --- | --- |
| Cash-flow and balance visibility | **Core** | This is a primary product value: an owner can understand the company’s actual financial position. |
| Employee expense reimbursement | **Core** | It is the main end-to-end MVP workflow: expense claim, review, and reimbursement. |
| Payment execution and status tracking | **Supporting** | It is essential to reimburse employees, but bank payment execution is not the product’s differentiator. |
| Organisation users and access | **Generic** | Invitations, blocking, fixed roles, and authentication are common capabilities. |
| Push notifications | **Generic** | Delivery of status notifications is a standard capability provided through an external service. |

## 2. Service Boundaries

Decomposition by business capability and decomposition by DDD subdomain produce the same boundaries. This is a useful validation: each service represents both a meaningful business capability and a bounded model with its own rules and data.

| Business capability / subdomain | Service | Responsibility | Owned data |
| --- | --- | --- | --- |
| Cash-flow and balance visibility / Cash Flow & Accounts | **Cash Flow Service** | Shows read-only accounts, balances, banking operations, and actual cash flow for a selected period. | Account and balance snapshots, imported banking operations, cash-flow aggregates. |
| Employee expense reimbursement / Expense Reimbursements | **Compensations Service** | Manages an employee claim from draft through submission to accountant approval or rejection. | Compensation claims, amounts, categories, comments, accountant decisions, receipt metadata. |
| Payment execution / Payment Execution | **Payments Service** | Creates a reimbursement payment, sends it to the bank reliably, and tracks its final status. | Payments, idempotency keys, delivery attempts, payment statuses, bank payment identifiers. |
| Organisation users and access / Identity & Access | **Organisation Access Service** | Manages organisation membership, invitations, blocking, and fixed business roles. | Organisation memberships, roles, invitations, blocked status. |
| Push notifications / Notifications | **Notifications Service** | Creates business notifications and requests their push delivery. | Notification requests and, if required, delivery history and statuses. |

The external **Auth Service** owns credentials, MFA, and tokens. Organisation Access owns only the product-specific question: which user belongs to which organisation and which role they have.

## 3. Cohesion and Coupling Check

| Service | Why cohesion is high | Coupling risk | Mitigation |
| --- | --- | --- | --- |
| **Cash Flow** | Accounts, balances, banking operations, and cash-flow calculation all serve one purpose: showing actual money movement. | It could treat a `pending` payment as an actual outgoing transaction. | Use confirmed bank operations or final payment status through a contract/event; do not read Payments data directly. |
| **Compensations** | Drafting, receipt attachment, submission, approval, and rejection are one claim lifecycle. | It could create a payment in the Payments data store or call the bank directly. | Publish `CompensationApproved` to Payments through a command or domain event. |
| **Payments** | Idempotency, `pending` status, retries, bank submission, and final status are one payment-execution concern. | It could update a compensation claim in the Compensations data store. | Publish `PaymentCompleted` or `PaymentFailed`; Compensations updates its own claim data. |
| **Organisation Access** | Invitation, membership, blocking, and fixed role assignment are all organisation-access rules. | It could duplicate credential, MFA, or token logic from Auth Service. | Delegate authentication to the external Auth Service; retain only organisation membership and roles. |
| **Notifications** | Selecting a recipient, preparing a push notification, and tracking delivery are one notification concern. | Compensations and Payments could each depend directly on the push provider. | Consume their business events and be the only service that calls the external Notification Service. |

## 4. DIP Example

**Payments depends on an abstraction, not on the bank provider.**

`Payments Service` uses the `BankingGateway` contract to submit a payment and query its status. The concrete `BankingSystemAdapter` translates that contract into the external bank API. Therefore, payment business rules do not depend on a particular bank SDK, HTTP client, or provider format. The adapter can be replaced or tested with a fake implementation without changing payment logic.
