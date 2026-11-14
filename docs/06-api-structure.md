# 06. API Structure Proposal

This document proposes the REST API of SupplyMesh. The API is served by the NestJS backend and documented automatically as an OpenAPI specification. The frontend uses a typed client generated from that specification.

Release markers: **MVP** (Release 1), **R2** (Release 2), **R3** (Release 3), as defined in document 08.

## 6.1 Conventions

| Topic | Rule |
|---|---|
| Base path and version | `/api/v1`. Additive changes (new fields, new endpoints) stay in `v1`. A breaking change creates `v2` and `v1` stays available for a notice period. |
| Style | REST over HTTPS, JSON request and response bodies, `snake_case` field names |
| Resource names | Plural nouns: `/suppliers`, `/purchase-requests`, `/invoices` |
| State changes | **Explicit action endpoints**, for example `POST /purchase-requests/{id}/submit`, not a free edit of a `status` field. This keeps each business rule in one use case. |
| Tenant | **Never in the URL or body.** It comes from the access token. |
| Authentication | `httpOnly` cookies (browser) or `Authorization: Bearer` (tools). CSRF token on state-changing browser requests. |
| IDs | UUID strings |
| Dates and time | ISO 8601. Timestamps in UTC. Plain dates (`required_by_date`) as `YYYY-MM-DD`. |
| Money | Decimal **strings** with a currency code: `{"amount": "1250.50", "currency": "USD"}`. Never JSON numbers, to avoid floating point errors. |
| Pagination | Keyset (cursor) based: `?limit=50&cursor=...`. Response contains `data` and `next_cursor`. No offsets. |
| Filtering and sorting | Query string: `?status=approved&department_id=...&sort=-created_at` |
| Concurrency | Update endpoints take the record `version`. A stale version returns `409 VERSION_CONFLICT`. |
| Idempotency | Endpoints that create money records require an `Idempotency-Key` header. A repeated key returns the first result. |
| Rate limits | Login: 5 attempts per minute per IP and per account. General API: 600 requests per minute per user. Headers `RateLimit-Limit`, `RateLimit-Remaining`, `Retry-After`. |
| Cross-tenant access | Returns `404`, never `403`. |
| Documentation | OpenAPI document served at `/api/v1/openapi.json`, with an interactive page in non-production environments. |

## 6.2 Response and Error Format

**Success (single item):**

```json
{
  "data": {
    "id": "0190c1f2-7a3e-7c11-9b6f-3a2d4e5f6a7b",
    "request_number": "PR-2026-000123",
    "status": "in_approval",
    "total_estimated_cost": { "amount": "1250.00", "currency": "USD" },
    "version": 4
  }
}
```

**Success (list):**

```json
{
  "data": [ { "id": "...", "status": "approved" } ],
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0xMS0xMlQxMDowMDowMFoifQ"
}
```

**Error (all failures use one shape, based on RFC 7807):**

```json
{
  "type": "https://docs.supplymesh.example/errors/approval-limit-exceeded",
  "title": "Approval limit exceeded",
  "status": 422,
  "code": "APPROVAL_LIMIT_EXCEEDED",
  "detail": "This request total is above your approval limit.",
  "request_id": "6f1d6c58-2a64-4a8b-8d88-7a7d3f6f1c01",
  "errors": []
}
```

For validation failures, `errors` lists the problem fields: `[{"field": "quantity", "message": "must be greater than 0"}]`.

| HTTP status | Meaning | Example codes |
|---|---|---|
| 400 | Malformed or invalid input | `VALIDATION_FAILED` |
| 401 | Not signed in or token expired | `UNAUTHENTICATED` |
| 403 | Signed in but not allowed | `PERMISSION_DENIED`, `SEPARATION_OF_DUTIES` |
| 404 | Not found, or belongs to another tenant | `NOT_FOUND` |
| 409 | Wrong state or conflict | `VERSION_CONFLICT`, `DUPLICATE_INVOICE`, `STEP_NOT_PENDING`, `SUPPLIER_NAME_EXISTS` |
| 422 | Business rule violated | `NO_APPROVAL_RULE`, `EXCEEDS_APPROVED_AMOUNT`, `INVOICE_HAS_OPEN_DISCREPANCY`, `EXCEEDS_BALANCE` |
| 429 | Rate limit | `RATE_LIMITED` |
| 500 | Unexpected error (details only in logs, with `request_id`) | `INTERNAL_ERROR` |

## 6.3 Endpoints

The "Permission" column uses the permission codes from section 6.4.

### 6.3.1 Authentication and profile

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `POST /auth/login` | Sign in with email and password | Public (rate limited) | MVP |
| `POST /auth/refresh` | Rotate refresh token, new access token | Refresh cookie | MVP |
| `POST /auth/logout` | Revoke current session | Signed in | MVP |
| `POST /auth/forgot-password` | Send a reset link | Public (rate limited) | MVP |
| `POST /auth/reset-password` | Set a new password with a reset token | Reset token | MVP |
| `POST /auth/accept-invitation` | Set password and activate account from an invitation | Invitation token | MVP |
| `POST /auth/mfa/enroll`, `POST /auth/mfa/verify` | Two-step sign-in for sensitive roles | Signed in | R2 |
| `GET /me` | Current user, roles, permission list | Signed in | MVP |
| `GET /me/sessions`, `DELETE /me/sessions/{id}` | See and revoke own sessions | Signed in | R2 |

### 6.3.2 Organization, departments, users, roles

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /organization` | Organization profile | `organization.read` | MVP |
| `PATCH /organization` | Update name and base settings | `organization.manage` | MVP |
| `GET /organization/settings`, `PATCH /organization/settings` | Tolerances, four-eyes switches, required documents | `organization.manage` | MVP |
| `GET /departments`, `GET /departments/{id}` | List and read | `department.read` | MVP |
| `POST /departments`, `PATCH /departments/{id}` | Create and edit (name, code, manager, budget) | `department.manage` | MVP |
| `POST /departments/{id}/archive` | Archive a department | `department.manage` | MVP |
| `GET /users`, `GET /users/{id}` | List and read users | `user.read` | MVP |
| `PATCH /users/{id}` | Edit department, approval limit | `user.manage` | MVP |
| `POST /users/{id}/disable`, `POST /users/{id}/enable` | Control access | `user.manage` | MVP |
| `PUT /users/{id}/roles` | Set the user's roles | `role.manage` | MVP |
| `POST /invitations`, `GET /invitations`, `POST /invitations/{id}/revoke` | Invite employees | `user.invite` | MVP |
| `GET /roles`, `GET /roles/{id}` | List and read roles | `role.read` | MVP |
| `POST /roles`, `PATCH /roles/{id}` | Create and edit custom roles (system roles are protected) | `role.manage` | MVP |
| `PUT /roles/{id}/permissions` | Replace a role's permission set | `role.manage` | MVP |
| `GET /permissions` | Catalog of available permissions | `role.read` | MVP |

### 6.3.3 Suppliers

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /supplier-categories`, `POST /supplier-categories` | Categories | `supplier.read` / `supplier.manage` | MVP |
| `GET /suppliers` | List with filters: `status`, `category_id`, `q` (name search) | `supplier.read` | MVP |
| `GET /suppliers/{id}` | Supplier profile with latest verification | `supplier.read` | MVP |
| `POST /suppliers` | Create draft | `supplier.manage` | MVP |
| `PATCH /suppliers/{id}` | Edit (with `version`) | `supplier.manage` | MVP |
| `POST /suppliers/{id}/submit-for-verification` | Send for review | `supplier.manage` | MVP |
| `POST /suppliers/{id}/verification/decision` | `approved`, `rejected`, `changes_requested` with `comment` | `supplier.verify` | MVP |
| `POST /suppliers/{id}/suspend`, `/reactivate`, `/blacklist` | Status management with `reason` | `supplier.manage` | MVP |
| `GET /suppliers/{id}/verifications` | Review history | `supplier.read` | MVP |
| `POST /suppliers/{id}/documents/upload-url` | Get signed upload link | `supplier.manage` | R2 |
| `POST /suppliers/{id}/documents` | Register an uploaded file | `supplier.manage` | R2 |
| `GET /suppliers/{id}/documents`, `GET /supplier-documents/{id}/download-url` | List and download | `supplier.read` | R2 |
| `GET /suppliers/{id}/performance-events` | Raw performance events | `supplier_performance.read` | MVP |
| `GET /suppliers/{id}/performance` | Calculated scores and trends | `supplier_performance.read` | R3 |

### 6.3.4 Purchase requests and approvals

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /purchase-requests` | List. Scope depends on permission (own, department, all). Filters: `status`, `department_id`, `priority`. | `purchase_request.read_*` | MVP |
| `GET /purchase-requests/{id}` | Request with items | `purchase_request.read_*` | MVP |
| `POST /purchase-requests` | Create draft with items | `purchase_request.create` | MVP |
| `PATCH /purchase-requests/{id}` | Edit draft or rejected request (with `version`) | `purchase_request.create` (own) | MVP |
| `POST /purchase-requests/{id}/submit` | Submit for approval | `purchase_request.create` (own) | MVP |
| `POST /purchase-requests/{id}/cancel` | Cancel | `purchase_request.create` (own) | MVP |
| `GET /purchase-requests/{id}/approval` | Approval progress: steps, assignees, decisions, attempts | `purchase_request.read_*` | MVP |
| `GET /approvals/inbox` | My pending steps | `approval.decide` | MVP |
| `POST /approval-steps/{id}/approve` | Approve (optional `comment`) | `approval.decide` | MVP |
| `POST /approval-steps/{id}/reject` | Reject (`comment` required) | `approval.decide` | MVP |
| `GET /approval-workflows`, `GET /approval-workflows/{id}` | Read rules | `workflow.read` | MVP |
| `POST /approval-workflows`, `PATCH /approval-workflows/{id}` | Create and edit rules | `workflow.manage` | MVP |
| `PUT /approval-workflows/{id}/steps` | Replace the ordered steps | `workflow.manage` | MVP |
| `POST /approval-workflows/{id}/activate`, `/deactivate` | Switch a rule on or off | `workflow.manage` | MVP |
| `POST /approval-workflows/simulate` | "Which rule would apply to amount X in department Y?" | `workflow.read` | R2 |
| `POST /approval-steps/{id}/reassign` | Owner moves a stalled step to someone else | `workflow.manage` | R3 |
| `POST /purchase-requests/{id}/comments`, `GET ...` | Discussion with procurement | `purchase_request.read_*` | R2 |

### 6.3.5 Purchase orders and delivery

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /purchase-orders` | List. Scope by permission. Filters: `status`, `supplier_id`, `department_id`. | `purchase_order.read_*` | MVP |
| `GET /purchase-orders/{id}` | Order with items and delivery status | `purchase_order.read_*` | MVP |
| `POST /purchase-requests/{id}/purchase-orders` | Create draft order from an approved request | `purchase_order.create` | MVP |
| `PATCH /purchase-orders/{id}` | Edit while draft (with `version`) | `purchase_order.manage` | MVP |
| `POST /purchase-orders/{id}/issue` | Issue the order | `purchase_order.manage` | MVP |
| `POST /purchase-orders/{id}/cancel` | Cancel (only if no receipts or invoices) | `purchase_order.manage` | MVP |
| `POST /purchase-orders/{id}/close` | Close | `purchase_order.manage` | MVP |
| `POST /purchase-orders/{id}/receipts` | Record a delivery with lines | `goods_receipt.record` | MVP |
| `GET /purchase-orders/{id}/receipts` | Delivery history | `purchase_order.read_*` | MVP |
| `GET /purchase-orders/{id}/history` | Status and change history for this order | `purchase_order.read_*` | MVP |

### 6.3.6 Invoices and payments

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /invoices` | List. Filters: `status`, `supplier_id`, `due_before`. | `invoice.read` | MVP |
| `GET /invoices/{id}` | Invoice with lines, match result, discrepancies | `invoice.read` | MVP |
| `POST /invoices` | Record invoice for an order and run matching | `invoice.record` | MVP |
| `POST /invoices/{id}/rematch` | Run matching again | `invoice.record` | MVP |
| `GET /invoices/{id}/discrepancies` | Findings | `invoice.read` | MVP |
| `POST /invoice-discrepancies/{id}/resolve` | `resolved`, `waived`, or `disputed`, with `note` | `invoice.resolve_discrepancy` | MVP |
| `POST /invoices/{id}/approve-for-payment` | Approve a matched invoice | `invoice.approve_payment` | MVP |
| `POST /invoices/{id}/void` | Void an unpaid invoice (with `reason`) | `invoice.void` | MVP |
| `POST /invoices/{id}/payments` | Record payment. **Requires `Idempotency-Key`.** | `payment.record` | MVP |
| `GET /invoices/{id}/payments`, `GET /payments`, `GET /payments/{id}` | Payment history | `payment.read` | MVP |
| `POST /payments/{id}/reverse` | Reverse a payment (with `reason`) | `payment.reverse` | MVP |
| `POST /invoices/{id}/file/upload-url` | Upload scanned invoice | `invoice.record` | R2 |

### 6.3.7 Analytics, audit, notifications, system

| Method and path | Purpose | Permission | Release |
|---|---|---|---|
| `GET /analytics/spend?group_by=department\|supplier\|month&from=&to=` | Total and grouped spending | `analytics.read_*` | R2 |
| `GET /analytics/pending-requests` | Counts of requests waiting, by step | `analytics.read_*` | R2 |
| `GET /analytics/approval-times` | Average and percentile processing time | `analytics.read_*` | R3 |
| `GET /analytics/supplier-performance` | Supplier performance trends | `analytics.read_*` | R3 |
| `GET /audit-logs?entity_type=&entity_id=&from=&to=` | Change history | `audit.read` | MVP |
| `GET /exports/{type}.csv` | CSV export of lists | matching read permission | R3 |
| `GET /notifications`, `POST /notifications/{id}/read`, `GET/PUT /notification-preferences` | In-app notifications | Signed in | R2 |
| `GET /health` (outside `/api/v1`) | Process is alive | Public | MVP |
| `GET /ready` (outside `/api/v1`) | Database reachable | Internal | MVP |

## 6.4 Permission Catalog and Default Roles

Permissions are rows in the `permissions` table. The five roles from the brief are seeded as system roles. The Owner can create custom roles from any combination.

Legend: ✓ granted by default.

| Permission | Owner | Procurement Mgr | Dept Mgr | Finance | Employee |
|---|:-:|:-:|:-:|:-:|:-:|
| `organization.read`, `organization.manage` | ✓ | | | | |
| `department.read` | ✓ | ✓ | ✓ | ✓ | |
| `department.manage` | ✓ | | | | |
| `user.read` | ✓ | ✓ | ✓ | ✓ | |
| `user.invite`, `user.manage` | ✓ | | | | |
| `role.read`, `role.manage` | ✓ | | | | |
| `workflow.read` | ✓ | ✓ | | | |
| `workflow.manage` | ✓ | | | | |
| `supplier.read` | ✓ | ✓ | ✓ | ✓ | |
| `supplier.manage`, `supplier.verify` | | ✓ | | | |
| `purchase_request.create` | | | ✓ | | ✓ |
| `purchase_request.read_own` | | | | | ✓ |
| `purchase_request.read_department` | | | ✓ | | |
| `purchase_request.read_all` | ✓ | ✓ | | ✓ | |
| `approval.decide` | ✓ | ✓ | ✓ | | |
| `purchase_order.create`, `purchase_order.manage` | | ✓ | | | |
| `goods_receipt.record` | | ✓ | | | |
| `purchase_order.read_all` | ✓ | ✓ | | ✓ | |
| `purchase_order.read_department` | | | ✓ | | |
| `purchase_order.read_assigned` | | | | | ✓ |
| `invoice.read` | ✓ | ✓ | | ✓ | |
| `invoice.record`, `invoice.resolve_discrepancy`, `invoice.approve_payment`, `invoice.void` | | | | ✓ | |
| `payment.read` | ✓ | ✓ | | ✓ | |
| `payment.record`, `payment.reverse` | | | | ✓ | |
| `supplier_performance.read` | ✓ | ✓ | | ✓ | |
| `analytics.read_all` | ✓ | | | | |
| `analytics.read_procurement` | | ✓ | | | |
| `analytics.read_department` | | | ✓ | | |
| `analytics.read_finance` | | | | ✓ | |
| `audit.read` | ✓ | | | | |
| `audit.read_finance` | | | | ✓ | |

This matches the role capability matrix in document 02. Note the separation of duties: no role holds both `purchase_order.create` and `payment.record`.

## 6.5 Examples

### Submit a request

```
POST /api/v1/purchase-requests/0190c1f2-7a3e-7c11-9b6f-3a2d4e5f6a7b/submit
Content-Type: application/json

{ "version": 3 }
```

Response `200`:

```json
{
  "data": {
    "id": "0190c1f2-7a3e-7c11-9b6f-3a2d4e5f6a7b",
    "request_number": "PR-2026-000123",
    "status": "in_approval",
    "total_estimated_cost": { "amount": "1250.00", "currency": "USD" },
    "approval": {
      "workflow": "Mid-value purchases",
      "attempt_no": 1,
      "current_step": { "order": 1, "name": "Procurement Manager approval", "required_approvals": 1 }
    },
    "version": 4
  }
}
```

### Record a payment

```
POST /api/v1/invoices/0190c200-1b2c-7d33-8e44-5f6a7b8c9d01/payments
Idempotency-Key: 5b2e7c1a-8d44-4c0e-9d1f-3a6b7c8d9e0f
Content-Type: application/json

{
  "amount": "1250.00",
  "currency": "USD",
  "payment_date": "2026-11-13",
  "method": "bank_transfer",
  "reference": "TRX-88231"
}
```

Response `201`:

```json
{
  "data": {
    "id": "0190c201-aa11-7bb2-9cc3-0d1e2f3a4b5c",
    "invoice_id": "0190c200-1b2c-7d33-8e44-5f6a7b8c9d01",
    "payment_type": "payment",
    "amount": { "amount": "1250.00", "currency": "USD" },
    "invoice_status": "paid",
    "invoice_balance": { "amount": "0.00", "currency": "USD" }
  }
}
```

Sending the same request again with the same `Idempotency-Key` returns the same response and creates nothing new.

### A business rule error

```
POST /api/v1/approval-steps/0190c1f3-0000-7000-8000-000000000001/approve
```

Response `422`:

```json
{
  "type": "https://docs.supplymesh.example/errors/approval-limit-exceeded",
  "title": "Approval limit exceeded",
  "status": 422,
  "code": "APPROVAL_LIMIT_EXCEEDED",
  "detail": "The request total is above your approval limit.",
  "request_id": "6f1d6c58-2a64-4a8b-8d88-7a7d3f6f1c01",
  "errors": []
}
```

## 6.6 API Design Decisions

| Decision | Reason |
|---|---|
| Action endpoints (`/submit`, `/approve`) instead of editing `status` | Each action runs one use case with its own checks. A status field cannot express "who may do what, and when." It also gives a clear audit event per action. |
| Tenant from the token, never the URL | A URL tenant can be tampered with. The token is signed. |
| `404` for other tenants' records | `403` would confirm the record exists. |
| Money as decimal strings | JSON numbers are floating point and lose precision. |
| Keyset pagination | Stays fast on large tables, and is stable when rows are added during browsing. |
| `Idempotency-Key` on money creation | Network retries and double clicks must not create two payments. |
| `version` in update bodies | Detects two people editing the same record, without long locks. |
| REST instead of GraphQL | Screens and workflows are known in advance. REST is simpler to secure, rate limit, and cache, and our action-based design maps cleanly to it. GraphQL would add schema, query cost, and authorization complexity without a matching benefit. |
| One error format with stable codes | The frontend shows the right message from the `code` without parsing text. |
| OpenAPI generated from code | Documentation and the typed client cannot drift away from the real API. |

## 6.7 Webhooks and Integration (Future)

Not part of the first release. When needed, the same outbox events that drive notifications (`PurchaseRequestApproved`, `PurchaseOrderIssued`, `PaymentRecorded`) can be delivered to customer systems as signed webhooks, with retries and a delivery log. Integration with ERP or accounting software would be built on this mechanism.
