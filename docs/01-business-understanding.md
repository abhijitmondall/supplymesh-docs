# 01. Business Understanding

## 1.1 The Core Problem

In a mid-to-large organization, buying things is not one task. It is a chain
of many people and decisions: an employee needs something, a manager approves
it, procurement finds a supplier, an order is placed, goods arrive, an invoice
comes, finance pays.

When this chain runs on email, spreadsheets, and chat messages, three things
break:

1. **Control is lost.** Nobody can prove who approved what, or why.
2. **Money leaks.** Duplicate orders, overpriced suppliers, invoices that do
   not match what was ordered, and purchases made outside policy.
3. **Knowledge is lost.** The organization cannot tell which suppliers are
   reliable, because performance history lives in people's heads.

SupplyMesh exists to turn this chain into one controlled, traceable,
measurable system.

## 1.2 Problem-to-Solution Mapping

| Business Pain | Real Consequence | SupplyMesh Response |
|---|---|---|
| Hundreds of suppliers, no single record | Duplicate suppliers, expired documents, no verification | Central supplier registry with verification and document tracking |
| Requests tracked by email | Lost requests, slow decisions, no visibility | Structured purchase requests with live status |
| Different approval rules per company | Hard-coded rules cannot fit every client | Configurable approval workflow engine |
| Unauthorized purchasing | Policy violations, fraud risk | Mandatory approval before any purchase order exists |
| Invoice does not match the order | Overpayment | Three-way matching: purchase order, delivery record, invoice |
| No supplier performance data | Repeated use of unreliable suppliers | Supplier intelligence from real delivery and invoice data |
| No audit trail | Cannot answer auditors or investigate fraud | Immutable audit log of every important action |
| Many companies on one platform | One data leak ends the business | Strict multi-tenant isolation |

## 1.3 Who Feels the Pain

| Role | Their Main Frustration | What They Need From the System |
|---|---|---|
| Employee | "I do not know where my request is." | Clear status and history |
| Department Manager | "I approve blindly, and I lose track of my budget." | Limited approval authority and spending view |
| Procurement Manager | "I chase approvals and compare suppliers manually." | One queue of approved requests, supplier data |
| Finance Officer | "Invoices arrive that I cannot verify." | Automatic matching and discrepancy flags |
| Organization Owner | "I cannot see or control spending across the company." | Settings, roles, and organization-wide analytics |

## 1.4 Why This Is a SaaS Product and Not a Single Company's Tool

SupplyMesh serves many organizations on one platform. This changes
the design in three ways:

- **Data isolation is the first rule**, not a feature. A leak between two
  customers is a business-ending event.
- **Rules must be configurable.** Company A approves everything under $500
  by a department manager. Company B may require two people. The code
  cannot contain either rule. The data must.
- **Roles must be customizable per organization.** The five roles in the
  brief are defaults, not a fixed list.

## 1.5 The Value Chain (the spine of the whole design)

```
Supplier Onboarding -> Purchase Request -> Approval -> Purchase Order
    -> Delivery -> Invoice -> Matching -> Payment -> Supplier Performance
```

Every module in this project is one link in this chain. Every database
table, API, and workflow in later documents attaches to one of these links.
The chain also shows the dependency order: nothing later can exist
without the earlier link.

## 1.6 Business Rules That Shape the Design

1. A purchase order cannot be created without a fully approved request.
2. A person cannot approve their own request.
3. An invoice cannot be paid before it is matched to a purchase order.
4. Only a verified and active supplier can receive a purchase order.
5. Every organization's data is invisible to every other organization.
6. Financial records are never deleted, only reversed or cancelled.
7. Every state change on a financial record is written to the audit log.

These seven rules are the reason the database uses strict constraints and
transactions, and the reason the architecture centers on consistency over
raw speed.

## 1.7 What Success Looks Like

- A request has a visible status at every moment.
- Approval follows the organization's own policy, with no manual chasing.
- Finance catches invoice mismatches before payment.
- Management can answer "where did our money go?" in minutes.
- An auditor can reconstruct any purchase from request to payment.
