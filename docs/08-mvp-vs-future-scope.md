# 08. MVP vs Future Scope

## 8.1 What "MVP" Means Here

The MVP is the smallest product that completes the full value chain once,
safely, for one real organization:

    Supplier -> Request -> Approval -> Purchase Order -> Delivery -> Invoice -> Payment

It is not a half-built version of everything. It is a thin version of the
whole chain, with the controls that make it trustworthy (permissions, approval
rules, audit log, tenant isolation).

## 8.2 MVP Scope (Release 1)

| Area | Included |
|---|---|
| Platform | Organizations, users, invitations, departments, custom roles and permissions, audit log |
| Supplier | Registration, profile, categories, verification, status management |
| Request | Create, submit, track status |
| Approval | Configurable rules by amount and department, multi-step approval, approver limits, snapshot of workflow |
| Purchase Order | Create from approved request, items, status, delivery recording |
| Finance | Invoice recording, matching, discrepancy review, payment recording |
| Data collection | Supplier performance records are stored from day one |

## 8.3 Release Plan

| Release | Focus | Features |
|---|---|---|
| **Release 1 (MVP)** | Complete the chain safely | Everything in 8.2 |
| **Release 2** | Visibility and speed | Supplier documents with expiry alerts, email and in-app notifications, request comments, basic spending dashboards |
| **Release 3** | Intelligence | Supplier scoring, risk indicators, approval time analytics, delegation, CSV export, escalation rules |
| **Later** | Growth | Supplier portal, multi-currency, ERP and accounting integration, bank payment integration, OCR invoice reading, RFQ and bidding, mobile app, predictive risk (AI) |

## 8.4 Why Each Decision Was Made

**Why the full chain is in the MVP.** A procurement tool that stops at approval
does not solve the business problem. The value is in the control from request
to payment, so every link must exist, even if each link is simple.

**Why audit log and permissions are in the MVP.** They cannot be added later.
Records written without an audit trail cannot be reconstructed, and retrofitting
permissions into a finished system means rewriting every endpoint.

**Why notifications are in Release 2, not the MVP.** The chain works without
them because users can open their dashboard. They also add infrastructure (email
provider, queue, templates). The database and code are still designed so
notification events can be added without changing business logic.

**Why supplier scoring is in Release 3.** Scoring needs history. We store
performance records from day one, so when scoring arrives it has real data.

**Why supplier documents are in Release 2.** The MVP verification works with
a status and a notes field. Document storage needs file handling, virus scanning,
and expiry tracking, which is a meaningful amount of extra work.

**Why multi-currency waits.** Wrong exchange rates cause financial errors.
We keep a currency column on money tables so we can add conversion later without
a painful migration.

## 8.5 Designing Now for the Future

These choices cost almost nothing today and save large rework later.

| Future need | What we do now |
|---|---|
| Multi-currency | Store currency code with every money amount |
| Notifications | Emit domain events (for example `RequestApproved`) even when nothing listens yet |
| Supplier portal | Keep supplier data separate from user accounts |
| New approval rule types | Store rules as data with a condition and step model, not as code |
| Scoring | Store raw performance events, not only totals |
| Microservices (if ever needed) | Strict module boundaries inside one codebase |

## 8.6 MVP Success Criteria

The MVP is done when one organization can:

1. Invite its staff and assign roles.
2. Register and verify a supplier.
3. Submit a request that follows the organization's own approval rules.
4. Turn the approved request into a purchase order.
5. Record delivery and the supplier invoice, and see it matched.
6. Record the payment.
7. Show an auditor the full history of that purchase.
