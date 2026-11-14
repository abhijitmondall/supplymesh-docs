# 09. Risk and Scalability Analysis

This document lists what could go wrong with the product and the design, how likely and how costly each risk is, and what the design does about it. It ends with the known limitations of the current design and the assumptions that still need to be validated.

**How ratings work.** Likelihood (L) and Impact (I) are rated Low (1), Medium (2), High (3). Score = L x I. Scores of 6 or 9 are **top risks**.

## 9.1 Risk Register

### 9.1.1 Security and data protection

| ID | Risk | L | I | Score | Mitigation in the design | Residual risk |
|---|---|:-:|:-:|:-:|---|---|
| S1 | **Cross-tenant data leak** (one organization sees another's data) | 2 | 3 | **6** | Four layers: tenant from token, repository filter, composite foreign keys, Row-Level Security. Automated isolation tests. A test that fails if a tenant table has no policy. App role cannot bypass RLS. | Low. A misconfigured database role is the remaining path, so role setup is reviewed and tested. |
| S2 | **Stolen or guessed credentials** | 3 | 3 | **9** | Argon2id, login rate limit and lockout, generic login errors, MFA for Owner and Finance (R2), session list and revocation. | Medium until MFA ships. MFA is the first R2 security item. |
| S3 | **Stolen token** | 2 | 3 | **6** | Short access token, rotating refresh token with reuse detection, `httpOnly` cookies, CSRF protection. | Low |
| S4 | **Insider fraud** (fake supplier, self-approval, paying yourself) | 2 | 3 | **6** | Separation of duties in default roles, no self-approval (code and trigger), four-eyes on supplier verification and invoice approval, immutable payments, full audit log. | Medium. Two colluding people can still defeat controls. Reports on unusual patterns come in R3. |
| S5 | **Malicious file upload** | 2 | 2 | 4 | Type and size checks, content check, private bucket, signed links, malware scan (R2). | Low. Until the scan ships (R2), documents are downloaded only as files, not rendered in the browser. |
| S6 | **Injection and web attacks** (SQL injection, XSS, CSRF) | 2 | 3 | **6** | Parameterized queries, React escaping, Content-Security-Policy, SameSite cookies, validation at the edge. | Low |
| S7 | **Vulnerable dependencies** | 3 | 2 | **6** | Dependency and image scanning in CI, regular updates, lock files. | Medium. This is an ongoing task. |
| S8 | **Stale permissions** (a removed role still works) | 2 | 2 | 4 | Permission version in the token, versioned cache. Access removal is effective on the next request. | Low |
| S9 | **Sensitive data in logs** | 2 | 2 | 4 | Structured logging with field allow-lists, no tokens or passwords logged, code review rule. | Low |

### 9.1.2 Data integrity and money

| ID | Risk | L | I | Score | Mitigation | Residual risk |
|---|---|:-:|:-:|:-:|---|---|
| D1 | **Duplicate or excess payment** | 2 | 3 | **6** | Invoice row lock, balance check, database `CHECK`, idempotency key, unique duplicate-invoice rule. | Low |
| D2 | **Totals drift** (stored total differs from lines or payments) | 2 | 3 | **6** | Updated in the same transaction. Nightly reconciliation with alerts. | Low. A drift is detected within a day. |
| D3 | **Lost or phantom events** (notification for a failed change, or a missing one) | 2 | 2 | 4 | Transactional outbox, idempotent handlers. | Low |
| D4 | **Wrong approval rule applied** (misconfigured workflow) | 2 | 3 | **6** | Range and column constraints, first-match order is visible, `simulate` endpoint (R2), all changes audited, Owner-only. | Medium. A wrongly configured rule is still a valid rule. R2 simulation reduces it. |
| D5 | **Rounding and currency errors** | 2 | 2 | 4 | Decimal arithmetic, `numeric` columns, currency stored with every amount, single currency per organization. | Low |
| D6 | **Backup that cannot be restored** | 1 | 3 | 3 | Automated backups with point-in-time recovery, scheduled restore drills. | Low if drills are actually done |
| D7 | **Bad migration damages data** | 2 | 3 | **6** | Backward-compatible steps, review, test on staging with production-like data, backup before release. | Medium. Needs discipline. |
| D8 | **Invoice matching produces too many false alarms** | 2 | 2 | 4 | Configurable tolerances, waive with note, rematch. | Medium. Tolerances need tuning per customer. |

### 9.1.3 Availability and performance

| ID | Risk | L | I | Score | Mitigation | Residual risk |
|---|---|:-:|:-:|:-:|---|---|
| A1 | **Database outage** | 1 | 3 | 3 | Managed PostgreSQL with Multi-AZ standby and automatic failover. | Low, with a short outage during failover |
| A2 | **Redis outage** | 2 | 1 | 2 | Redis is not a source of truth. Fallback to the database. Sessions are in PostgreSQL. | Low |
| A3 | **Worker outage or backlog** | 2 | 2 | 4 | Outbox keeps events. Alerts on backlog age. Workers can be duplicated. | Low |
| A4 | **Slow dashboards and reports** | 3 | 2 | **6** | Targeted indexes, caching, then read replica and summary tables. | Medium until summary tables exist (R2 and R3) |
| A5 | **Noisy neighbor** (one tenant slows others) | 2 | 2 | 4 | Per-tenant rate limits, query time limits, option to move a tenant to its own database. | Medium |
| A6 | **Document number lock** (many requests in one organization) | 2 | 1 | 2 | Number assigned last, short lock, other organizations unaffected, fallback to native sequence. | Low |
| A7 | **Audit log grows very large** | 3 | 2 | **6** | Monthly partitions, archive old partitions, narrow rows. | Low |
| A8 | **Bad deployment** | 2 | 2 | 4 | Automated tests, staging, manual production approval, health checks, one-step rollback. | Low |
| A9 | **Email delivery problems** (spam folder, provider outage) | 2 | 1 | 2 | Retries, bounce handling, in-app notifications as alternative. Business actions never depend on email. | Low |

### 9.1.4 Product, process, and team

| ID | Risk | L | I | Score | Mitigation | Residual risk |
|---|---|:-:|:-:|:-:|---|---|
| P1 | **Stalled approval** (the only approver left, was disabled, or is away) | 3 | 2 | **6** | Fall back to Owner at submit. Nightly job finds steps with no active assignee and notifies the Owner. Reassign (R3), delegation (R3), reminders and escalation (R3). | **Medium in the MVP.** This is the most visible gap. Until R3, the Owner must act manually. |
| P2 | **Users bypass the system** (buy outside it, "shadow procurement") | 3 | 3 | **9** | Make the happy path fast: simple request form, clear status, mobile-friendly web. Finance rule: invoices without a matched order are not paid. Dashboards that show off-system spending (R2). | Medium. This is a business risk the software can reduce, not remove. |
| P3 | **Scope grows beyond the MVP** | 3 | 2 | **6** | Written MVP boundary, Won't-Have list, release plan (document 08). | Medium |
| P4 | **Team unfamiliar with chosen tools** (Drizzle, NestJS patterns, Row-Level Security) | 2 | 2 | 4 | Short spike on RLS and transactions first. Code review. Repository layer isolates the ORM. | Medium |
| P5 | **Dependency on a young tool** (Drizzle) | 2 | 2 | 4 | Repository interfaces. SQL migrations in plain SQL. The domain does not import the ORM. | Low |
| P6 | **Compliance and legal** (tax invoice rules, data retention, personal data laws) | 2 | 3 | **6** | Immutable financial records, audit log, retention by partition, encryption at rest and in transit. Country-specific tax and invoice formats are out of scope and must be reviewed per market. | Medium. Needs legal input before selling in regulated markets. |
| P7 | **Key-person dependency** | 2 | 2 | 4 | Documentation repository, ADRs, standard tools, automated pipeline. | Medium |
| P8 | **Load assumptions are wrong** | 2 | 2 | 4 | Treated as assumptions. Load test before launch (k6) on inbox, list, approve, payment. Metrics and alerts after launch. | Medium until tested |
| P9 | **Vendor lock-in to AWS** | 1 | 2 | 2 | Standard PostgreSQL, S3-compatible `FileStorage` interface, containers. | Low |

## 9.2 Top Risks and What We Do About Them First

| Rank | Risk | First action |
|---|---|---|
| 1 | S2 Credential theft | Ship MFA for Owner and Finance early in Release 2. Rate limit and lockout from day one. |
| 2 | P2 Users bypass the system | Design the request form and approval flow to be quick. Include "no matched order, no payment" in the onboarding guide. |
| 3 | S1 Cross-tenant leak | Write the isolation tests in the first sprint, before most features exist. |
| 4 | P1 Stalled approval | Nightly detection and Owner alert in the MVP. Reassign and delegation in R3. |
| 5 | D1, D2 Money errors | Concurrency tests for payment and approval. Reconciliation job in the MVP. |
| 6 | A4 Slow dashboards | Build indexes first. Measure. Add summary tables in R2 and R3 as data grows. |

## 9.3 Scalability Analysis

The detailed plan is in document 05 (section 5.6). This section summarizes it as a capacity ladder.

| Stage | Setup | Handles (rough, to be validated) | Move to the next stage when |
|---|---|---|---|
| **1. Launch** | 2 API containers, 1 worker, 1 PostgreSQL (managed), Redis, PgBouncer | Dozens of organizations, hundreds of concurrent users | p95 latency or database CPU stays high at peak |
| **2. Reduce reads** | Add read replica for reports, more cache | Hundreds of organizations | Dashboards or exports slow the main database |
| **3. Partition and summarize** | Partition large append-only tables, summary tables for analytics | Many years of data per organization | Table size or vacuum time grows |
| **4. Isolate heavy tenants** | Move large organizations to dedicated databases | Very large customers | One tenant dominates load or requires physical isolation |
| **5. Scale writes** | Extract async modules, then consider sharding by `organization_id` | Beyond one write node | Single-node write limit reached despite all above |

Why the design scales this way:
- **API is stateless**, so adding containers is easy.
- **Every key and index starts with `organization_id`**, so splitting by tenant later is natural.
- **Heavy and slow work is already asynchronous** (worker, outbox), so it can move out without touching the core.
- **Money flows stay in one database** for as long as possible, because that is where simplicity and safety come from.

Metrics that decide the move between stages: p95 and p99 latency, error rate, database CPU, connections and slow queries, queue depth and job delay, outbox backlog age, cache hit rate, table and partition sizes.

## 9.4 Known Limitations of the Current Design

These are honest limits, not hidden ones. Each has a reason and a path forward.

| # | Limitation | Why accepted | Path forward |
|---|---|---|---|
| 1 | **Single currency per organization.** No exchange rates. | Wrong conversion causes real financial errors. Prioritize correctness. | Currency is stored on every amount. Add conversion with rate history later. |
| 2 | **Sequential approvals only.** No parallel branches. Conditions limited to amount and department. | Covers the brief's rules and keeps the model easy to explain and index. | Add conditions and branching to the template model. |
| 3 | **Stalled approvals need manual action** in the MVP. | Reassign and delegation need more screens and rules. | R3 |
| 4 | **Payments are recorded, not executed.** No bank connection. | Payment gateways add compliance and security scope. | Integration phase |
| 5 | **Manual invoice entry.** No OCR. | OCR is costly and error-prone. Manual entry works. | Later, with human review of OCR results |
| 6 | **No supplier portal.** Suppliers cannot self-register or confirm orders. | Needs a second authentication context and UI. | Later release |
| 7 | **One organization per user account.** | Simpler login and permissions. | Add a membership table when needed |
| 8 | **Single region.** | Cost and complexity. | Backups and infrastructure as code for recovery. Multi-region is a business decision. |
| 9 | **Logical, not physical, tenant isolation.** | Cost. Layers compensate. | Dedicated database option for customers who require it |
| 10 | **Whole backend deploys together.** | Modular monolith trade-off (ADR-002). | Extract async modules if needed |
| 11 | **Supplier scoring is not in the first release.** | Needs accumulated history. | Raw events are stored from day one, scoring in R3 |
| 12 | **No ERP or accounting integration.** | Each customer differs. | Webhooks and integration after demand is proven |
| 13 | **Country-specific tax and legal invoice formats are not handled.** | Needs per-market legal review. | Per-market work before selling in regulated regions |
| 14 | **Load assumptions are unverified.** | No production data exists yet. | Load test before launch |

## 9.5 Assumptions to Validate

| Assumption | How to validate |
|---|---|
| About 400 requests per second at peak for 200 organizations | Load test with realistic data. Measure after launch. |
| Customers accept sequential approval with limits | Ask pilot customers about their real policies. |
| A tolerance-based matching rule (2% price, 0% quantity) reduces noise enough | Review discrepancy rates during pilot and tune. |
| Customers need one organization per user account | Ask pilot customers whether staff work across organizations. |
| Email plus dashboard is enough for approvals in the MVP | Measure approval times. Add reminders if they are slow. |
| Customers do not need physical tenant separation at first | Ask during sales. Offer a dedicated database option if required. |

## 9.6 What We Would Test Before Launch

1. **Tenant isolation suite** across all endpoints and raw queries.
2. **Concurrency tests:** two approvers at once, two payments at once, two orders for the same request.
3. **Load test** (k6) on the inbox, request list, approve, invoice record, and payment endpoints.
4. **Failure drills:** stop Redis, stop the worker, restart the database, and confirm the system degrades without losing data.
5. **Restore drill:** restore a backup to a new database and verify records.
6. **Security review** against the OWASP checklist and a dependency scan.
7. **Migration rehearsal** on staging with a copy of realistic data.
