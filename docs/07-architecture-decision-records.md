# 07. Architecture Decision Records (ADRs)

An ADR records **one important decision**: the problem, the options, what was chosen, and why. Each decision below is tied to a specific requirement of SupplyMesh, not to general preference.

**Format used for every ADR:** Problem, Available alternatives, Selected solution, Reason for selection, Advantages, Disadvantages, Trade-offs. A short "Revisit when" line is added to say what would change the decision.

## ADR Index

| ADR | Decision | Selected |
|---|---|---|
| 001 | Primary database | PostgreSQL |
| 002 | System architecture style | Modular monolith |
| 003 | ORM and data access | Drizzle ORM |
| 004 | Multi-tenancy model | Shared schema with `organization_id`, composite keys, and Row-Level Security |
| 005 | Redis | Used narrowly (queue delivery, rate limits, cache), never as source of truth |
| 006 | Storing approval workflows | Relational templates plus copied instances |
| 007 | Keeping orders, invoices, and payments consistent | One ACID transaction, targeted locks, constraints, reconciliation |
| 008 | Background jobs and events | Transactional outbox plus BullMQ worker |
| 009 | Authentication and sessions | Short-lived JWT plus rotating refresh tokens stored in PostgreSQL |
| 010 | Authorization model | RBAC with roles and permissions stored as data, plus scope and rule checks |
| 011 | Deployment and infrastructure | Docker, NGINX, GitHub Actions, AWS (production), VPS (staging) |

Status of all records: **Accepted** (13 November 2026).

---

## ADR-001: PostgreSQL as the Primary Database

### Problem

SupplyMesh stores facts that refer to each other: organizations, suppliers, requests, approvals, orders, deliveries, invoices, payments. Money values must always add up. Approval, order, and payment changes must succeed or fail together. Customers' data must be separated. We need to choose the database that makes these properties cheapest and safest to guarantee.

### Available alternatives

| Option | Description |
|---|---|
| **PostgreSQL** | Relational database with transactions, constraints, `jsonb`, Row-Level Security, partitioning |
| MongoDB | Document database |
| MySQL / MariaDB | Relational database |
| PostgreSQL + MongoDB | Relational for core data, documents for flexible data |

### Selected solution

**PostgreSQL**, with `jsonb` for the few flexible fields.

### Reason for selection

- **Relationships are the core of the domain.** An invoice points to an order, which points to a request and a supplier. We need real foreign keys, including composite ones that include `organization_id`, so the database itself refuses a link across tenants.
- **Money rules map directly to constraints.** `UNIQUE (organization, supplier, invoice_number)` prevents duplicate invoices. `CHECK (paid_amount <= total_amount)` prevents overpayment. `CHECK (total = subtotal + tax)` keeps totals honest. Partial unique indexes enforce "one active approval per request."
- **Tenant isolation inside the database.** Row-Level Security is built in (ADR-004).
- **Reporting needs joins and aggregation** across orders, invoices, and payments (spend by department, by supplier, approval time). SQL does this natively.
- **Growth tools exist already:** declarative partitioning for the audit log, partial indexes for work queues, read replicas, and `jsonb` for settings and audit snapshots.
- **Honest comparison with MongoDB.** MongoDB does have multi-document transactions and unique indexes, so the decision is not "MongoDB cannot do it." The difference is **where integrity lives**. PostgreSQL enforces foreign keys, constraints, and row-level security inside the database, for every code path. MongoDB has no foreign keys and no built-in row-level security, and its document validation works inside a single document. So links between collections, rules across records, and tenant row filters would live in application code, where one missed path can corrupt money data.
- MySQL could also serve, but PostgreSQL has stronger support for the features above (Row-Level Security, partial indexes, `jsonb`, richer constraints).

### Advantages

- Strong integrity guarantees enforced for every writer, including future scripts and tools.
- One system for relational data, flexible fields, search (trigram), and analytics.
- Mature ecosystem and managed offering on AWS (RDS).

### Disadvantages

- Writes scale vertically first. Horizontal sharding is harder than with some NoSQL systems.
- Schema changes need discipline (migrations).
- Rigid schema is less convenient for highly variable data.

### Trade-offs

We give up the convenience of schema-free documents and easy horizontal write scaling. We gain integrity and simpler reasoning about money. For a procurement system with steady traffic, correctness matters more than extreme write throughput.

### Revisit when

A single write node can no longer keep up even after partitioning, replicas, and moving heavy tenants out. Then evaluate sharding by `organization_id`.

---

## ADR-002: Modular Monolith over Microservices or SOA

### Problem

We must choose how to structure the backend. The main business operations cross several modules: issuing a purchase order changes the order, changes the request, writes the audit log, and records an event. The team is small and the first release must be delivered quickly.

### Available alternatives

| Option | Description |
|---|---|
| **Modular monolith** | One deployable application, strict internal modules |
| Microservices | Many small services, each with its own database |
| Service-Oriented Architecture | Few large services connected by shared middleware |

### Selected solution

**Modular monolith**, with module boundaries enforced by lint rules, one shared PostgreSQL database, and a separate worker process started from the same code.

### Reason for selection

- **Consistency across modules is the main requirement.** In a monolith, "order plus request status plus audit plus event" is one database transaction. With separate services it becomes a chain of calls plus compensation logic. A failure in the middle leaves a request marked "converted" with no order, which in a financial system is the failure we most want to avoid.
- **Traffic pattern does not call for independent scaling.** Procurement is a business workflow with steady daytime traffic. We do not need to scale approvals separately from suppliers on day one.
- **Team size and time.** Microservices would spend the first months on infrastructure (service discovery, tracing, many pipelines) instead of the product.
- **Tenant isolation and audit** are applied in one place.
- **Boundaries are still taken seriously.** Modules own their tables, talk through public interfaces and domain events, and a build check rejects forbidden imports. This keeps the option to extract a module later.

### Advantages

- Simple operations: one deployable, one database, one log stream.
- Fast development and easy debugging.
- Strong consistency without distributed transaction machinery.
- Low cost.

### Disadvantages

- The whole backend is deployed together.
- One module cannot be scaled alone.
- Boundaries depend on tooling and discipline.
- A very heavy module can affect others on the same instance.

### Trade-offs

We trade independent scaling and deployment for consistency, speed, and low operational cost. We reduce the downside with stateless API containers that can be duplicated, a separate worker for heavy tasks, automated tests, and boundary checks.

### Revisit when

A module has a clearly different scaling profile or ownership, such as notifications or document processing. Extract that module first, because it is already asynchronous.

---

## ADR-003: Drizzle ORM instead of Prisma

### Problem

We need a data access layer in TypeScript for a schema that uses features beyond basic tables: composite foreign keys, partial unique indexes, check constraints, partitioned tables, Row-Level Security, triggers, and a per-transaction setting for the tenant (`SET LOCAL`). We also need explicit control over transactions and row locks.

### Available alternatives

| Option | Description |
|---|---|
| **Drizzle ORM** | TypeScript schema, SQL-like query builder |
| Prisma | Own schema language, generated client |
| TypeORM | Decorator-based ORM |
| Query builder only (Kysely) | Typed SQL builder, no ORM features |
| Raw SQL with a driver | Full control, no help |

### Selected solution

**Drizzle ORM** with `node-postgres`, plus hand-written SQL migrations for features the schema builder does not generate.

### Reason for selection

- **SQL stays visible.** Our correctness depends on exact SQL: `SELECT ... FOR UPDATE`, `FOR SHARE`, guarded updates (`UPDATE ... WHERE received_quantity + :q <= quantity`), `ON CONFLICT`. Drizzle's query style maps almost one-to-one to these statements. Nothing is hidden.
- **The schema lives in TypeScript**, next to the code, and types flow from it. Composite foreign keys, check constraints, and partial indexes can be declared in it.
- **Transactions are explicit.** A single transaction function can run `SET LOCAL app.current_org_id` first, then every repository call in it shares the same connection. This is the heart of the tenant isolation design.
- **Custom SQL is a first-class step.** Row-Level Security policies, triggers, and partition setup are written as plain SQL migrations beside the generated ones.
- **Light runtime.** It adds little overhead and no separate query engine process.
- **Honest comparison with Prisma.** Prisma has the more mature tooling, a larger community, and a very friendly developer experience. Prisma also supports interactive transactions and raw queries. But many of the features we rely on (partial indexes, partitions, RLS, triggers, row locks) end up as hand-written SQL either way, and then Prisma's abstraction gives less. Prisma's own schema language is a second place to describe the schema. Since our design is database-centric, we prefer the tool that stays closer to SQL.
- TypeORM was rejected for weaker type safety and a history of subtle behavior in complex queries. A query-builder-only approach (Kysely) is a strong option, but Drizzle also provides schema definition and migration generation in the same package.

### Advantages

- Full control of locks, transactions, and SQL features.
- Type-safe queries without a code-generation step in the build.
- Easy to mix generated and hand-written migrations.
- Small and fast.

### Disadvantages

- Younger and smaller ecosystem than Prisma. Fewer examples and guides.
- Migration tooling has fewer safeguards. Reviews and tests must catch mistakes.
- Developers must know SQL well. The ORM does not protect against bad queries.
- Familiarity matters: a team that already knows Prisma needs a short learning period.

### Trade-offs

We give up some convenience and community size in return for transparency and control exactly where our risk is: money, locking, and tenant isolation. We reduce the downside with a repository layer (so the ORM is replaceable), code review of every migration, and integration tests against a real PostgreSQL.

### Revisit when

The team grows and productivity matters more than SQL control, or Drizzle's migration tooling proves unreliable in production. Because repositories hide the ORM from the domain, switching would affect only the infrastructure layer.

---

## ADR-004: Multi-Tenancy with Shared Schema, Composite Keys, and Row-Level Security

### Problem

Many organizations use the same platform. One organization must never see, change, or even reference another's data. The model must stay affordable for many small customers and manageable for operations (one migration for everyone).

### Available alternatives

| Option | Description |
|---|---|
| **Shared database and schema, `organization_id` column** | All tenants in the same tables |
| Schema per tenant | One PostgreSQL schema per organization |
| Database per tenant | One database per organization |
| Shared schema without extra protection | Rely on application filters only |

### Selected solution

**Shared schema with `organization_id` on every tenant table**, protected by four layers: tenant taken from the token, tenant filter in the repository base class, composite foreign keys, and PostgreSQL Row-Level Security.

### Reason for selection

- **Cost and operations.** Schema-per-tenant and database-per-tenant multiply migrations, backups, and monitoring by the number of customers. With many small organizations this is not realistic for a small team.
- **The weakness of the shared model is a missing filter.** The design removes that weakness with layers, so that no single bug can leak data: even a query that forgets the filter returns nothing, because Row-Level Security fails closed when no tenant is set.
- **Composite foreign keys** make cross-tenant links impossible at write time, which a plain filter cannot do.
- **Escape hatch.** Because every row has `organization_id`, one very large customer can be copied to its own database later.
- Database-per-tenant would be preferred if customers demanded physical separation by contract. That can be offered later as a premium option using the same code.

### Advantages

- One migration, one backup strategy, one monitoring setup.
- Cheapest per tenant.
- Strong protection thanks to layers.
- Cross-tenant platform reports (for the operator) are straightforward and audited.

### Disadvantages

- Isolation is logical, not physical. A serious database misconfiguration (for example granting the application the right to bypass RLS) would break it.
- Noisy neighbors: one tenant's heavy query affects others.
- Extra care for every new table (needs a policy).
- Row-Level Security adds a small cost to each query.

### Trade-offs

We accept logical instead of physical isolation to keep cost and complexity low, and we compensate with defense in depth and tests: cross-tenant API tests, RLS tests, and a test that fails if any table with `organization_id` has no policy. We also set per-tenant rate limits and query time limits against noisy neighbors.

### Revisit when

A customer requires physical isolation or data residency in a specific region, or one tenant's load dominates. Move that tenant to a dedicated database.

---

## ADR-005: Redis Is Used, but Only as an Accelerator

### Problem

The brief lists Redis and asks whether it is required. Redis adds a component to run, monitor, and secure. We must decide whether the product needs it, and for what, without letting it become a source of truth for financial data.

### Available alternatives

| Option | Description |
|---|---|
| **Redis for queue delivery, rate limits, and cache** | Narrow, replaceable use |
| No Redis at all | PostgreSQL does everything: job table with polling, counters in tables, in-memory cache |
| Redis as the main store for sessions and state | Wider use |
| A message broker (RabbitMQ or Kafka) | Heavier infrastructure for events |

### Selected solution

**Redis for three jobs only**: BullMQ job delivery, login and API rate limiting, and short-lived cache (permissions, settings, dashboard numbers). **PostgreSQL remains the source of truth** for sessions, outbox events, and all business data.

### Reason for selection

- **It is not required for correctness.** Every function would still work without Redis, only slower, because sessions, outbox, and data are in PostgreSQL. That is why the decision is "use it narrowly," not "depend on it."
- **Permission checks happen on almost every request.** Caching the permission set removes a repeated query from nearly all traffic, with instant invalidation through a version number.
- **Rate limiting** needs fast atomic counters, which are natural in Redis and wasteful in PostgreSQL.
- **Job queues** need retries, delays, and scheduling. BullMQ provides these in a proven way (ADR-008).
- **Financial state is never cached** (requests, approvals, orders, invoices, payments), because correctness matters more than speed there.
- A message broker such as Kafka or RabbitMQ is more than our event volume needs.

### Advantages

- Faster reads and lower database load.
- Reliable rate limiting and job scheduling.
- Failure is safe: if Redis is lost, the system slows down and queue jobs are rebuilt from the outbox.

### Disadvantages

- One more component to run, secure, and monitor.
- Cache invalidation must be correct (a stale permission set is a security issue, which is why we version it).
- Cost of a managed instance.

### Trade-offs

We accept one extra component to avoid hot paths hitting the database and to get dependable rate limiting and jobs. We keep it harmless by making it optional for correctness. If the team wants the absolute minimum infrastructure for the first release, the cache and rate limit can start in application memory and the queue can run from the outbox table alone, then Redis can be added when load requires it.

### Revisit when

Operating Redis becomes a burden compared to its benefit, or event volume grows enough to justify a dedicated broker.

---

## ADR-006: Storing Approval Workflows as Relational Templates with Copied Instances

### Problem

Each organization has its own purchasing policy. Examples: under $500 the department manager approves; from $500 to $5,000 the procurement manager; above $5,000 several people in order. Policies change over time. Requests already in approval must not be affected by a policy change, and every approval must be explainable later to an auditor.

### Available alternatives

| Option | Description |
|---|---|
| **Relational templates plus instances copied at submission** | Workflow and step tables for rules. Instance and instance-step tables for each request. |
| Rules written in code | `if (amount < 500) ...` |
| One JSON document per workflow | The whole policy in a `jsonb` field, interpreted by code |
| External workflow engine (Camunda, Temporal) | A separate system that runs the process |
| Generic rules engine with expressions | Rules as expressions evaluated at runtime |

### Selected solution

**Relational tables.** Templates (`approval_workflows`, `approval_workflow_steps`) hold the policy as data. When a request is submitted, the system matches a template and **copies its steps** into `approval_instances` and `approval_instance_steps`. The eligible approvers for the active step are stored in `approval_step_assignees`. Each vote is a row in `approval_decisions`.

### Reason for selection

- **Different rules per organization means rules must be data.** Rules in code would require a release for every customer's policy change.
- **The most common queries are relational.** "Show me my pending approvals" is a single indexed query on assignee rows. A JSON document cannot answer it efficiently.
- **Integrity can be enforced.** Constraints ensure an approver column set matches the approver type, ranges are valid (`max > min`), a user votes once per step, and only one approval runs per request.
- **Copying gives stability and auditability.** The instance records exactly which steps applied to this request, even if the template changes tomorrow. The `attempt_no` keeps the history of rejected and resubmitted requests.
- **The first-match-by-order rule is simple to explain and to test.** `evaluation_order` decides between overlapping rules.
- **Extensibility without redesign.** New approver types are new strategy classes (Open/Closed principle). If conditions beyond amount and department are ever needed (for example category), a `conditions` field can be added and evaluated by the same matcher.
- An external workflow engine would bring a second system, a second deployment, and a second place for permissions and tenancy, which is too much for a flow that has four to five steps. A generic expression engine is flexible but hard to validate, explain, and index.

### Advantages

- Policy changes need no deployment.
- Fast inbox and status queries.
- Clear audit trail and per-attempt history.
- Database-level protection against invalid configuration.

### Disadvantages

- More tables (six in the approval area).
- Complex conditions (for example "if category is IT and amount above X unless requester is Y") are not supported in the first release.
- Copying steps duplicates data (deliberately).
- Steps are sequential. Parallel branches are not supported.

### Trade-offs

We give up maximum flexibility (arbitrary rules, parallel branches) for integrity, speed, and a model that people can understand. The brief's rules fit it well, and the "required approvals per step" field covers "several approvers" at one level.

### Revisit when

Customers need parallel approvals, conditional routing across many attributes, or visual workflow design. Then add conditions and branching to the template model, or evaluate a workflow engine.

---

## ADR-007: Keeping Orders, Invoices, and Payments Consistent

### Problem

Several rules connect these records: orders must stay within the approved amount; an invoice must match its order and deliveries; the same invoice cannot be recorded twice; payments cannot exceed the invoice; totals must always agree with their lines. Many users act at the same time (two payers, two receivers, two approvers). A mistake in money data is expensive and hard to explain.

### Available alternatives

| Option | Description |
|---|---|
| **One ACID transaction per command, targeted row locks, constraints, guarded updates, reconciliation** | Consistency enforced at the database |
| `SERIALIZABLE` isolation everywhere | Let the database detect all conflicts and retry |
| Optimistic locking only | Version numbers, retry on conflict |
| Application checks only | Validate in code before saving |
| Eventual consistency with sagas | Each step committed separately, with compensation |
| Event-sourced ledger | Store events and derive balances |

### Selected solution

A layered approach, all inside PostgreSQL:

1. **One transaction per command.** Business change, audit row, and outbox event commit together.
2. **Targeted row locks** (`SELECT ... FOR UPDATE`) on the single record being decided: the approval step, the invoice being paid, the order receiving goods. Locks are taken in a fixed order and held briefly.
3. **Constraints as the last line of defense:** unique invoice number per supplier, `paid_amount <= total_amount`, total equals subtotal plus tax, one reversal per payment, one active approval per request.
4. **Guarded updates** such as `UPDATE ... WHERE received_quantity + :q <= quantity` for counters.
5. **Idempotency keys** for payment creation.
6. **Immutable money records:** payments are never edited. Reversals are new rows.
7. **Optimistic locking** (`version`) for ordinary edits.
8. **Nightly reconciliation** that recomputes stored totals and balances and alerts on differences.
9. Default isolation `READ COMMITTED`.

### Reason for selection

- **A single database makes real transactions possible** (ADR-001, ADR-002). We should use them rather than rebuild them with sagas.
- **Locks are narrow.** The lock is on one invoice or one step. Unrelated work does not wait. This gives safety without a large performance cost.
- **Two independent protections for the most dangerous operation.** Payment is guarded by a lock plus a balance check in code **and** a `CHECK` in the database. Both must fail for money to be lost.
- **`SERIALIZABLE` everywhere** would be safe in theory but produces frequent retries under concurrency and spreads retry logic across all code. Targeted locks are easier to understand and test.
- **Optimistic locking alone** is right for edits but poor for money counters, where retrying after a conflict could repeat a side effect without an idempotency key.
- **Sagas** solve a problem we avoided by not splitting the system (ADR-002).
- **Event sourcing** gives perfect history but adds large complexity. The audit log plus immutable payments gives the history needed.
- **Reconciliation** is a safety net for the unexpected: a bug, a manual database change, or a mistake in a future migration.

### Advantages

- Strong guarantees with simple mental model: "one command, one transaction."
- No half-finished states visible to users.
- Layers: a bug in code is caught by a constraint.
- Easy to test with parallel integration tests.

### Disadvantages

- Locks can create waiting when many people act on the same record.
- Developers must follow the lock-order rule.
- Denormalized totals (`paid_amount`, `received_quantity`) must be updated in the same transaction. A missed update would be caught by reconciliation, not instantly.
- Transactions must stay short, so long calls (email, file upload) must never sit inside them.

### Trade-offs

We trade a little concurrency on the same record for correctness. Because contention happens only on one invoice or one step at a time, the cost is small in practice. We choose explicit, readable locking over the more automatic but more surprising `SERIALIZABLE` mode.

### Revisit when

Reconciliation finds differences repeatedly, or lock waits show up in monitoring. Then consider moving the payments ledger to an append-only model with balances derived from entries.

---

## ADR-008: Background Jobs and Events with a Transactional Outbox and BullMQ

### Problem

Some work must not happen inside the user's request: sending email, nightly checks, report refresh, document scanning. But some of it must **never be lost**: a notification that a request needs approval, or a supplier event that feeds performance data. A naive approach ("save, then send email") loses events if the server stops between the two steps, or sends an email for a change that was rolled back.

### Available alternatives

| Option | Description |
|---|---|
| **Transactional outbox in PostgreSQL, relayed to BullMQ workers** | Event saved with the change, then processed asynchronously |
| Do the work inside the request | Send email and run checks before replying |
| Publish to a queue directly after commit | Save, then enqueue |
| Job table in PostgreSQL only (for example a polling worker or a library such as pg-boss) | No Redis needed |
| Message broker (RabbitMQ, Kafka) | Dedicated event infrastructure |
| Cron scripts only | Scheduled shell jobs |

### Selected solution

**Transactional outbox.** The use case writes an `outbox_events` row in the same transaction as the business change. A relay in the worker reads pending rows with `FOR UPDATE SKIP LOCKED`, enqueues BullMQ jobs, and marks them processed. Handlers are idempotent. Scheduled work (nightly checks, partition maintenance) uses BullMQ repeatable jobs. The worker is a separate container from the same code.

### Reason for selection

- **No lost events, no phantom events.** The event exists if and only if the change committed. "Save then enqueue" can lose the event in a crash, and "enqueue then save" can announce something that failed.
- **Slow work leaves the request path.** Responses stay fast. An email provider outage does not break approvals.
- **Retries, delays, and schedules** are mature in BullMQ, so we do not write our own.
- **Safe if Redis is lost.** The outbox rows are in PostgreSQL, so the relay recreates the queue jobs.
- **Multiple workers can share work** without double processing, thanks to `SKIP LOCKED` and idempotent handlers.
- **A Postgres-only job queue** is a credible option and would remove Redis from the design. We still chose BullMQ because Redis is already used for rate limits and cache (ADR-005), and its scheduling and retry features are more mature. The outbox keeps the choice reversible.
- A message broker is more infrastructure than our event volume justifies.

### Advantages

- Reliable delivery with at-least-once semantics.
- Decouples modules: the approval module raises an event and does not know who listens.
- Easy to add new reactions later (Release 2 notifications) without touching business code.
- Failures are visible: failed jobs and outbox backlog age can be monitored.

### Disadvantages

- Delivery is **at least once**, so handlers must be idempotent.
- Events are slightly delayed (seconds).
- Two moving parts (relay and queue) instead of one.
- The outbox table needs cleanup and a partial index to stay small.

### Trade-offs

We accept a small delay and the discipline of idempotent handlers in exchange for never losing or inventing an event. We also accept running the worker and Redis.

### Revisit when

Event volume or number of consumers grows to the point where a broker is justified, or if the team decides to remove Redis entirely. The outbox pattern would still be used with a different relay.

---

## ADR-009: Authentication with Short-Lived JWT and Rotating Refresh Tokens in PostgreSQL

### Problem

Users sign in from browsers. A stolen credential or token could expose financial data and allow fraudulent approvals. Access must be revocable (disabled user, password reset, lost device). Most requests should not need an extra database lookup just to identify the user. Login must not depend on optional infrastructure.

### Available alternatives

| Option | Description |
|---|---|
| **Short-lived JWT plus rotating refresh token stored (hashed) in PostgreSQL** | Hybrid |
| Server-side sessions stored in Redis | Session ID in a cookie |
| Long-lived JWT only | Stateless, no storage |
| External identity provider (Cognito, Auth0, Keycloak) | Managed login |
| Basic or API-key style authentication | Not suitable for browser users |

### Selected solution

- Passwords hashed with **Argon2id**.
- **Access token:** JWT valid about 15 minutes, with user ID, organization ID, session ID, and permission version.
- **Refresh token:** random opaque value, stored only as a hash in `user_sessions`, **rotated on every use**, with a `family_id` for reuse detection.
- Tokens are delivered in `httpOnly`, `Secure`, `SameSite` cookies. CSRF token for state-changing requests.
- Rate limiting and lockout on login. Invitations and password resets use single-use hashed tokens.

### Reason for selection

- **Most requests are authenticated without a database call** because the access token is a signed, short-lived JWT.
- **Revocation works** because the refresh token is stored. Logout, password reset, disabled user, or detected token reuse revokes the session, and a stolen access token dies within minutes.
- **Reuse detection** turns a stolen refresh token into a detectable event: if both the thief and the user try to refresh, the family is revoked.
- **PostgreSQL, not Redis, for sessions**, so sign-in never depends on Redis (ADR-005).
- **`httpOnly` cookies** keep tokens away from JavaScript, reducing the impact of XSS. `localStorage` was rejected for this reason.
- A **long-lived JWT** cannot be revoked and is rejected. **Server sessions in Redis** are a good model but would make Redis critical to sign-in.
- An **external identity provider** is a serious option and the right answer when enterprise single sign-on (SAML or OIDC) is required. For the first release, our flows (invitations to a specific organization and role, tenant-aware tokens) are simple enough to build with proven libraries, and avoiding a vendor keeps cost and coupling low. The design keeps authentication behind an interface so an identity provider can be added for SSO later.

### Advantages

- Fast per-request authentication.
- Revocable sessions and theft detection.
- No dependency on Redis for sign-in.
- Clear path to MFA and SSO.

### Disadvantages

- We own the security of the login system, including password reset, lockout, and rotation logic.
- Cookie-based auth needs CSRF protection.
- Access tokens remain valid until they expire (up to 15 minutes) even after a permission change, which is why permissions are checked from the live permission version and not from token content.
- Slightly more complex than plain sessions.

### Trade-offs

We trade the simplicity of buying a login service for control and lower coupling, accepting responsibility for getting it right. We reduce the risk by using standard algorithms and libraries, avoiding custom cryptography, and testing the token flows.

### Revisit when

A customer requires SSO, or the team does not have the capacity to maintain the login system securely. Move authentication to an identity provider and keep tenant and permission logic in SupplyMesh.

---

## ADR-010: RBAC with Roles and Permissions Stored as Data

### Problem

The brief defines five roles, but also says the Organization Owner must "configure roles and permissions." Different companies will want different splits of duty. Rules must also depend on context: an employee sees only their own requests, a department manager only their department, and an approver only up to a limit.

### Available alternatives

| Option | Description |
|---|---|
| **RBAC with permissions in tables, plus scope and business-rule checks** | Roles are data. Permission codes include scope. |
| Fixed roles in code | `if (user.role === 'FINANCE')` |
| Full ABAC or policy engine (OPA, Cedar) | Rules as policies over attributes |
| Permissions attached directly to users | No roles |

### Selected solution

Roles and permissions are tables (`roles`, `permissions`, `role_permissions`, `user_roles`). The five roles from the brief are seeded per organization as system roles. Permission codes carry scope (`purchase_request.read_own`, `read_department`, `read_all`). Checks run in four steps, denying by default: permission, scope, business rule, then database row-level security. Permission sets are cached with a version for instant invalidation.

### Reason for selection

- **The requirement is configurability.** Roles stored as data let the Owner create and change roles with no deployment.
- **Code checks permissions, not role names.** The code asks "may this user do `payment.record`?" and never "is this user Finance?", so changing how roles are assembled never changes code.
- **Scope in the permission code keeps it simple.** There is no need for a policy language to say "own versus department versus all."
- **Business rules stay in the domain.** Rules such as "not your own request" and "within your approval limit" depend on record state, so they belong with the use case, not in a permission table.
- **Separation of duties is built into defaults:** no role has both order creation and payment recording.
- A **full policy engine** would offer more power but adds a language, tooling, and a service to learn and operate for needs we do not have yet. It could replace the scope step later.
- **Fixed roles in code** do not meet the "configure roles" requirement.

### Advantages

- Configurable per organization.
- Easy to audit: the permission list of a role is a query.
- Simple, fast checks with caching.
- Role names can change without code changes.

### Disadvantages

- More tables and a seeding process for new organizations.
- A misconfigured custom role can grant too much, so Owner changes are audited and system roles are protected from edits.
- Fine-grained, attribute-based rules (for example by supplier category or cost center) are not supported without extending the model.

### Trade-offs

We accept limited expressiveness (no arbitrary attribute rules) for clarity, speed, and a model the Owner can understand. Cached permissions use a version so a role change takes effect immediately.

### Revisit when

Customers need attribute-based rules that cannot be expressed as scoped permissions. Introduce a policy layer for the scope step.

---

## ADR-011: Deployment with Docker, NGINX, GitHub Actions, AWS for Production, and a VPS for Staging

### Problem

We need a deployment approach that is repeatable, cheap at first, secure, and able to grow, without spending the first months on infrastructure. The brief lists Docker, multi-container architecture, NGINX, AWS, GitHub Actions, CI/CD, and VPS, and asks us to say which are useful now.

### Available alternatives

| Option | Description |
|---|---|
| **Docker containers on a VPS (staging) and AWS (production: managed PostgreSQL, S3, SES, containers on EC2, ECS later)** | Pragmatic and upgradeable |
| Kubernetes | Full orchestration |
| Serverless (Lambda, managed functions) | Pay per call |
| Single VPS for everything including production database | Cheapest |
| Platform-as-a-service (Heroku-style) | Simplest hosting |

### Selected solution

- **Docker** images for web, API, and worker. **Docker Compose** defines the multi-container setup: NGINX, web, API, worker, Redis, and PostgreSQL locally.
- **NGINX** as reverse proxy: TLS, routing, compression, rate limiting, static assets.
- **GitHub Actions** pipeline: lint, type check, unit tests, integration tests, security scan, build, push, deploy to staging, end-to-end tests, manual approval, migrations, deploy to production, health check, rollback to previous image.
- **Staging** on a VPS with Docker Compose. **Production** on AWS: **RDS for PostgreSQL** (backups, point-in-time recovery, Multi-AZ standby), **S3** for files, **SES** for email, containers on **EC2**, with a move to **ECS** when needed.
- **Not used:** Kubernetes, serverless, service mesh.

### Reason for selection

- **Managed database in production.** Backups, failover, and patching are the most valuable things to not run ourselves. Our data is financial, so recovery is a business need.
- **Docker gives identical environments** from laptop to production, which removes "works on my machine."
- **The system has few moving parts** (web, API, worker, database, Redis). Kubernetes would cost more in learning and operations than it saves.
- **Serverless** would fit the stateless API, but a connection-heavy, transaction-based PostgreSQL workload with row-level security context per transaction and a long-lived worker is a more natural fit for containers.
- **A VPS for staging** gives a cheap, realistic environment for demos and release checks.
- **AWS services match the needs**: S3 and SES are the standard choices for files and email, and RDS removes database operations.
- **Image tags per commit** make rollback a one-step action.

### Advantages

- Reproducible builds and deployments.
- Low cost at the beginning, with a clear upgrade path (EC2 to ECS, add replicas).
- Production data protected by managed backups and failover.
- Full pipeline with tests, scans, and manual approval before production.

### Disadvantages

- Some vendor lock-in to AWS for S3, SES, and RDS. Interfaces (`FileStorage`) and standard PostgreSQL limit it.
- Docker Compose on EC2 has limited self-healing and rolling updates compared with an orchestrator.
- Two different environments (VPS and AWS) can drift if infrastructure is not described as code.

### Trade-offs

We trade the power of an orchestrator for simplicity and cost, and we choose managed services where operating them ourselves would be risky (database, email, storage). We keep drift under control by using the same images and the same Compose definitions, and by describing AWS resources as code.

### Revisit when

We need rolling deployments with zero downtime, many service instances, or multiple regions. Move to ECS (or Kubernetes if the team and scale justify it).

---

## Decisions Postponed on Purpose

| Topic | Why not decided now |
|---|---|
| Message broker (Kafka, RabbitMQ) | The outbox plus BullMQ covers the volume. |
| Search engine (Elasticsearch) | PostgreSQL trigram and full-text search are enough for now. |
| Read replica and analytics warehouse | Not needed until reports slow down the main database. |
| Identity provider and SSO | Deferred until a customer needs it. Authentication is behind an interface. |
| Multi-currency model | Currency is stored with all amounts, but conversion is out of scope. |
| Multi-region deployment | A business decision that comes after product-market fit. |

## Summary Table: Decisions and the Requirement Behind Each

| ADR | Requirement that drove it |
|---|---|
| 001 | Financial integrity and relational data |
| 002 | Cross-module consistency and small team |
| 003 | Need for explicit SQL, locks, RLS, and custom migrations |
| 004 | Strict data isolation between organizations |
| 005 | Speed on hot paths without making cache a source of truth |
| 006 | Configurable approval policy per organization, auditability |
| 007 | Correct money under concurrency |
| 008 | Reliable notifications and nightly work without blocking users |
| 009 | Secure and revocable sign-in |
| 010 | Configurable roles and context-aware permissions |
| 011 | Repeatable delivery with low cost and safe data |
