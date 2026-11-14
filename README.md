# SupplyMesh: Global Procurement and Supplier Intelligence Platform

Design-first engineering documentation for **SupplyMesh**, a multi-tenant SaaS platform that manages suppliers, purchase requests, approvals, purchase orders, invoices, payments, and supplier performance for medium and large organizations. No application code is included, as the assessment requires.

**In one paragraph.** Buying something in a company is a chain: an employee needs it, a manager approves, procurement picks a supplier, an order is placed, goods arrive, an invoice comes, finance pays. On email and spreadsheets this chain loses control, money, and knowledge. SupplyMesh turns it into one controlled, traceable system. Every decision here comes from keeping that chain **consistent, isolated per organization, configurable, and explainable to an auditor**.

## Repository Structure

```
.
├── README.md
└── docs/
    ├── 01-business-understanding.md
    ├── 02-product-features.md
    ├── 03-database-design.md
    ├── 04-application-architecture.md
    ├── 05-system-design.md
    ├── 06-api-structure.md
    ├── 07-architecture-decision-records.md
    ├── 08-mvp-vs-future-scope.md
    └── 09-risk-and-scalability-analysis.md
```

Suggested reading order: 01 to 05, then 07 for the reasoning behind the decisions, then 06, 08, and 09.

## Required Submission Items

| #   | Submission item                      | Where                                                                              |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------- |
| 1   | Product Feature Specification        | [02](docs/02-product-features.md), sections 2.1 to 2.6                             |
| 2   | User Journey Diagrams                | [02](docs/02-product-features.md), section 2.7 (three journeys)                    |
| 3   | Complete ER Diagram                  | [03](docs/03-database-design.md), section 3.3                                      |
| 4   | Database Schema Documentation        | [03](docs/03-database-design.md), sections 3.4 to 3.13                             |
| 5   | High-Level Architecture Diagram      | [04](docs/04-application-architecture.md), section 4.3                             |
| 6   | System Design Diagrams               | [05](docs/05-system-design.md), state and sequence diagrams in sections 5.2 to 5.5 |
| 7   | API Structure Proposal               | [06](docs/06-api-structure.md)                                                     |
| 8   | Technical Decision Records (11 ADRs) | [07](docs/07-architecture-decision-records.md)                                     |
| 9   | MVP vs Future Scope Document         | [08](docs/08-mvp-vs-future-scope.md)                                               |
| 10  | Risk and Scalability Analysis        | [09](docs/09-risk-and-scalability-analysis.md)                                     |

## Evaluation Areas

| Evaluation area (marks)       | Covered in                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------ |
| Business Understanding (6)    | [01](docs/01-business-understanding.md)                                        |
| Product Feature Design (9)    | [02](docs/02-product-features.md), [08](docs/08-mvp-vs-future-scope.md)        |
| Database Design (12)          | [03](docs/03-database-design.md)                                               |
| Application Architecture (9)  | [04](docs/04-application-architecture.md), [06](docs/06-api-structure.md)      |
| System Design (12)            | [05](docs/05-system-design.md), [09](docs/09-risk-and-scalability-analysis.md) |
| Technical Decision Making (9) | [07](docs/07-architecture-decision-records.md)                                 |
| Documentation Quality (3)     | This README and the numbered `docs/` folder                                    |

## Assessment Deliverables

| Deliverable                                                                                                                             | Covered in                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 01 Product Feature Design (features, roles, rules, workflows, dependencies, limitations, MVP, MoSCoW, journeys)                         | [02](docs/02-product-features.md), [08](docs/08-mvp-vs-future-scope.md)   |
| 02 Database Design (ER, tables, keys, constraints, indexes, normalization, multi-tenancy, growth, alternatives)                         | [03](docs/03-database-design.md)                                          |
| 03 Application Architecture (style choice, frontend, backend, API, auth, authorization, storage, notifications, jobs, caching, diagram) | [04](docs/04-application-architecture.md), [06](docs/06-api-structure.md) |
| 04 System Design (four workflows, bottlenecks, scaling)                                                                                 | [05](docs/05-system-design.md)                                            |
| 05 Technical Decision Document (at least 8 ADRs)                                                                                        | [07](docs/07-architecture-decision-records.md)                            |

## Engineering Requirements and Considerations

| Topic                                            | Where                                                                                                                                                     |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Object-oriented programming, SOLID               | [04](docs/04-application-architecture.md) section 4.4.4                                                                                                   |
| Clean Architecture, clean code, maintainability  | [04](docs/04-application-architecture.md) sections 4.4.2, 4.4.3, 4.4.6                                                                                    |
| Database design techniques                       | [03](docs/03-database-design.md) sections 3.8 to 3.10                                                                                                     |
| Multi-tenant architecture                        | [03](docs/03-database-design.md) section 3.11, ADR-004                                                                                                    |
| Role-based access control, permission boundaries | [02](docs/02-product-features.md) section 2.2, [04](docs/04-application-architecture.md) section 4.8, [06](docs/06-api-structure.md) section 6.4, ADR-010 |
| API design                                       | [06](docs/06-api-structure.md)                                                                                                                            |
| System design, scalability                       | [05](docs/05-system-design.md) section 5.6, [09](docs/09-risk-and-scalability-analysis.md) section 9.3                                                    |
| Security                                         | [04](docs/04-application-architecture.md) sections 4.7, 4.8, 4.17, ADR-009                                                                                |
| Testing strategy                                 | [04](docs/04-application-architecture.md) section 4.16                                                                                                    |
| Auditability                                     | [03](docs/03-database-design.md) (`audit_logs`), [02](docs/02-product-features.md) rule BR-X2, ADR-007                                                    |
| Data consistency                                 | ADR-007, [05](docs/05-system-design.md) transaction handling in each workflow                                                                             |
| Fault tolerance                                  | [05](docs/05-system-design.md) section 5.6.5                                                                                                              |
| Future extensibility                             | [08](docs/08-mvp-vs-future-scope.md) section 8.5, ADR-006                                                                                                 |
| Technologies used and not needed yet             | [04](docs/04-application-architecture.md) section 4.14, ADR-011                                                                                           |

## Technology Summary

| Area                               | Choice                                                           | Decision record                           |
| ---------------------------------- | ---------------------------------------------------------------- | ----------------------------------------- |
| Architecture                       | Modular monolith                                                 | ADR-002                                   |
| Backend                            | NestJS (TypeScript) on Node.js                                   | [04](docs/04-application-architecture.md) |
| Frontend                           | Next.js (TypeScript)                                             | [04](docs/04-application-architecture.md) |
| Database                           | PostgreSQL with Row-Level Security                               | ADR-001, ADR-004                          |
| Data access                        | Drizzle ORM                                                      | ADR-003                                   |
| Cache, queue delivery, rate limits | Redis (narrow use)                                               | ADR-005                                   |
| Background jobs                    | Transactional outbox and BullMQ worker                           | ADR-008                                   |
| Authentication                     | Short-lived JWT and rotating refresh tokens stored in PostgreSQL | ADR-009                                   |
| Authorization                      | RBAC with permissions as data, plus scope and rule checks        | ADR-010                                   |
| Files                              | S3 with signed URLs                                              | [04](docs/04-application-architecture.md) |
| Delivery                           | Docker, NGINX, GitHub Actions, AWS (production), VPS (staging)   | ADR-011                                   |

## Key Design Ideas

1. **Tenant isolation in four layers:** token, repository filter, composite foreign keys, Row-Level Security.
2. **Approval rules are data.** Templates are copied into an instance at submission, so a rule change never alters a request already in approval.
3. **Money is immutable and double-checked.** Payments are never edited, and reversals are new rows. Each payment is protected by a row lock, a balance check, and a database constraint.
4. **One command, one transaction.** The business change, the audit log, and the outbox event commit together.
5. **PostgreSQL is the source of truth.** Redis only accelerates. If it fails, the system slows down but loses nothing.

## Conventions

- Diagrams are written in [Mermaid](https://mermaid.js.org/) and render on GitHub. If one does not display, paste its code block into [mermaid.live](https://mermaid.live).
- **MVP, R2, R3** mean Release 1, 2, and 3 (document 08).
- **BR-xx** identifiers (for example BR-R7) are business rules defined in document 02, section 2.4.
- Permission codes (for example `payment.record`) are listed in document 06, section 6.4.

## AI Assistance Statement

This documentation was drafted with the help of an AI assistant (Claude). The author (**Abhijit Mondal**) reviewed the content, made the final decisions, and explains and justifies the design in the recorded presentation, as the assessment requires.
