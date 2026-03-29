---
name: domain-modeling
description: "Use when clarifying what data a system stores, how entities relate, and what workflows drive state changes. Works for any project regardless of language or framework."
origin: custom
triggers:
  - /domain-modeling
tags:
  - modeling
  - data-model
  - architecture
  - process
version: "1.0"
---

# domain-modeling

Clarify the data model and core business processes for any project. Use this when starting a new project, rearchitecting an existing system, or adding a new entity type.

---

## Step 1 — What Data Exists

List every noun that represents data the system must persist across restarts.

Common examples across projects:
- User, Account, Organization
- Project, Workspace, Folder
- Document, File, Attachment
- Order, LineItem, Payment
- Task, Job, Build, Deployment

For each entity, identify:
- **Identity** — what field uniquely identifies it (usually `id: UUID`)
- **Core state** — the fields that change over time
- **Timestamps** — `created_at` always; `updated_at` if mutable

Ask: *"If the server restarts, what must survive?"* — only persistent data belongs here.

---

## Step 2 — How Entities Connect

For each pair of entities, ask: **does one own, reference, or belong to the other?**

```
Relationship patterns:
  1:1    One-to-One     │ User ↔ UserProfile
  1:N    One-to-Many    │ User → Projects  (one user owns many projects)
  N:1    Many-to-One    │ Task → Project  (many tasks belong to one project)
  N:N    Many-to-Many   │ Task ↔ Tag     (needs junction table)
  Self   Self-Reference │ Task → Task    (parent_id: task has subtasks)
```

**For each relationship ask:**
- Can the child exist without the parent? (cascade delete or nullify?)
- Is the FK nullable? (can a Task exist without a Project?)
- Do siblings need ordering? (e.g., `depends_on` for task ordering)

---

## Step 3 — State Machines

For each entity with a lifecycle, map the valid transitions.

```
Example: Order lifecycle
┌────────┐ new ┌──────────┐ paid ┌────────┐ shipped ┌─────────┐ delivered ┌────────┐
│  new   │ ──► │ pending  │ ───►│  paid  │ ──────► │ shipped │ ────────► │ done   │
└────────┘     └──────────┘      └────────┘         └─────────┘           └────────┘
                     │                   │
                     │ cancel            │ cancel
                     │                   │
                     └─────────┬─────────┘
                               ▼
                         ┌──────────┐
                         │ cancelled│  ← single node, two paths in
                         └──────────┘
```

For each transition, define:
- **Trigger** — user action, system event, or timer
- **Precondition** — what must be true before the transition is allowed
- **Side effect** — what else changes (audit log, notification, other fields)

**Cross-entity state propagation** — ask for each transition:
> Does this trigger state changes in other entities?

Example: payment fails → order is cancelled. Payment and Order are two different entities, but the payment failure causes the order to transition. Document this alongside the transition:
- `Order.pending → Order.cancelled` triggered by `Payment.failed` (not by a direct user action on Order)

---

## Step 4 — Sub-entities and Derived Data

### Sub-entities

Some entities cannot exist without a parent. These are sub-entities.

| Pattern | Example | FK location |
|---------|---------|------------|
| Line item belongs to Order | LineItem without Order has no meaning | LineItem.order_id (required, non-nullable) |
| Comment belongs to Post | Comment without Post has no meaning | Comment.post_id (required) |
| Task belongs to Project | Task without Project has no meaning | Task.project_id (required) |

For each sub-entity ask:
- Can it exist without its parent? (if no: required FK, cascade delete)
- Does the parent need a count of children? (e.g., `order.line_items.count`) — consider storing as a field or computing on read
- Are children ordered? (if yes: add `position` / `sort_order` field)

### Denormalized / Derived Fields

Sometimes you store data that could be computed from other tables, for query performance:

```
Example:
  Order.total_amount = SUM(LineItem.quantity * LineItem.unit_price)  ← could be stored
  Task.execution_count = COUNT(Execution)                            ← could be stored
  Project.task_count  = COUNT(Task)                                  ← could be stored
```

Ask:
- What aggregate queries does this system run frequently? (dashboard counts, totals)
- Can you afford to keep the derived field in sync, or should you compute it on read?
- Is eventual consistency acceptable, or must the derived field always be exact?

### Business Rules / Constraints

Not everything fits in a schema. Some rules are purely logical:

```
Examples:
  - A user can have at most 10 projects
  - A task can depend on at most 5 other tasks
  - An order cannot be cancelled after it has been shipped
  - A username must be unique within an organization (not globally)
```

Document these as invariant rules alongside the model — they are part of the data model's contract.

---

## Step 5 — Core Business Processes

For each main workflow, define:

```
Process: [Name]
Trigger:    [what starts this process]
Steps:      [numbered, ordered actions]
Outcome:    [what the system looks like after it succeeds]
Rollback:   [what happens if step N fails mid-way]
```

```
Example: Process a User Signup
─────────────────────────────────
Trigger:  POST /signup with email + password
Steps:
  1. Validate email is not already registered
  2. Hash password with bcrypt/scrypt
  3. Create user record (status=active)
  4. Generate JWT token
  5. Send welcome email (async, non-blocking)
Outcome:  user created, token returned
Rollback: delete user record if token generation fails
```

```
Example: Process an Order Checkout
─────────────────────────────────
Trigger:  User clicks "Place Order"
Steps:
  1. Validate cart is not empty
  2. Lock inventory for each item (prevent oversell)
  3. Charge payment (Stripe/PayPal)
  4. If payment fails: unlock inventory, abort
  5. Create order record (status=paid)
  6. Create fulfillment records for each line item
  7. Send order confirmation email (async)
Outcome:  order placed, inventory decremented, payment captured
Rollback: unlock inventory if order creation fails
```

For each process also note:
- **Who triggers it**: user / system / scheduled job / another process
- **What events fire**: webhooks, emails, SSE, internal events

---

## Step 6 — Model Summary Template

Fill this in for each project:

```markdown
## Data Model

### Entity: <Name>
| Field      | Type      | Notes                        |
|------------|-----------|------------------------------|
| id         | UUID      | PK                           |
| name/title | String    | max N chars                  |
| status     | Enum      | A/B/C states                 |
| ...        | ...       | ...                          |
| created_at | DateTime  | auto                         |
| updated_at | DateTime  | auto (if mutable)            |

### Relationships
- A → B: 1:N (cascade delete / nullify / restrict)
- B → C: N:1 (required FK / optional FK)
- C ↔ D: N:N via junction table (or JSON array if SQLite)

### Sub-entities
- <SubName> belongs to <Parent>: FK is required, cascade delete

### Denormalized / Derived Fields
- <field>: computed from <source> (stored for performance / computed on read)

### State Machine: <Entity>
- A → B: [trigger] ([precondition])
- B → C: [trigger] ([precondition])
- any → X: [trigger] ([precondition])
- Cross-entity: [EntityA.state] triggered by [EntityB.transition]

### Business Rules
- [Rule 1]: e.g. "max 10 projects per user"
- [Rule 2]: e.g. "order cannot cancel after shipped"

### Core Processes
- **Process X**: trigger → N steps → outcome; rollback on step K failure
- **Process Y**: ...
```

---

## Verification Checklist

- [ ] Every entity has exactly one primary key field
- [ ] Every FK field is indexed
- [ ] Every entity has `created_at`; mutable entities also have `updated_at`
- [ ] All state transitions are defined (including error/cancel states)
- [ ] Every transition has a trigger, precondition, and side effect
- [ ] Cross-entity state propagation is documented (e.g. payment fails → order cancelled)
- [ ] Cascade behavior is explicit for every relationship
- [ ] Sub-entities are identified: can they exist without their parent?
- [ ] Nullable FKs are intentional — can the child exist without the parent?
- [ ] Denormalized/derived fields are intentional and their sync strategy is defined
- [ ] Business rules are documented as explicit invariants
- [ ] Sensitive fields (passwords, tokens, PII) are never returned in API responses
- [ ] Concurrent access is considered — do two processes modifying the same entity race?
- [ ] Soft delete vs hard delete decision is made for each entity
