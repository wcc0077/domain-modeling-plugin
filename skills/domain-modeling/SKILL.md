---
name: domain-modeling
description: "Use when defining or clarifying the data model and core business processes. Minimalist: 4 concepts, no redundant fields."
origin: custom
triggers:
  - /domain-modeling
tags:
  - modeling
  - architecture
version: "2.1"
---

# domain-modeling

Define a data model and core processes in 4 steps. Only 4 concepts exist — everything else is metadata on them.

---

## The 4 Concepts

```
Entity     → what you store
Relation   → how entities connect
Transition → how state changes
Process    → ordered steps with rollback
```

Anything extra (sub-entity, derived field, business rule, cross-entity trigger) is just a **label** on one of these 4 — not a new concept.

---

## Step 1 — Entities

Ask: *"What survives a server restart?"*

For each entity, write one line:
```
Entity: <Name> — <what it represents>
```

Then for each, note only the non-obvious fields:
- Fields that are not `id`, `created_at`, `updated_at`
- Fields that have validation rules (max length, enum values, required)

```
Entity: User       — account that owns projects
  + username: unique, max 50
  + email: unique
  + password_hash: never expose in API
  + project_count: int (derived: COUNT Project)  ← stored or computed?

Entity: Project   — workspace for tasks
Entity: Task      — unit of work
```

---

## Step 2 — Relations

For each pair of entities, write one line:
```
<Entity> → <Entity>  [1:N / N:1 / N:N / Self]
```

Then annotate with the only 2 questions that matter:
- **Required?** — can the child exist without the parent? (required FK = sub-entity)
- **Cascade?** — delete parent, what happens to children?

```
User → Project    1:N  (required, cascade delete)
Project → Task    1:N  (required, cascade delete)
Task → Task      Self  (parent_id, optional — subtask)
Task ↔ Task     N:N  (depends_on, via JSON array)
Task ↔ Tag      N:N  (via task_tags junction)
```

Annotations (all optional):
- `(required)` — sub-entity, FK non-nullable
- `(cascade)` — delete parent removes children
- `(business rule: ...)` — e.g. "max 10 per user"

---

## Step 3 — Transitions

For each entity with state, list valid transitions:
```
<Entity>.<state> → <Entity>.<state>
  Trigger:    [user action / event / timer]
  Precondition: [what must be true]
  Side effect:  [what else changes]
```

```
Task.pending → Task.running
  Trigger: agent picks up task
  Side effect: emit SSE event

Task.running → Task.done
  Trigger: review quality >= 0.7
  Side effect: save output, emit SSE

Task.running → Task.failed
  Trigger: retry_count >= 3
  Side effect: emit SSE

Task.any → Task.cancelled
  Trigger: user cancels task
```

Cross-entity trigger — write it inline:
```
Order.pending → Order.cancelled
  Trigger: Payment.failed (cross-entity)
```

---

## Step 4 — Processes

For each main workflow, write:
```
Process: <Name>
Trigger:  [what starts it]
Steps:    [numbered]
Rollback: [what on step N failure]
```

```
Process: Place Order
Trigger:  user clicks "Place Order"
Steps:
  1. Validate cart
  2. Charge payment
  3. Create order (status=paid)
  4. Create fulfillment records
  5. Send confirmation email (async)
Rollback: unlock inventory if step 3 fails

Process: Cancel Order
Trigger:  user or payment timeout
Steps:
  1. Validate order.status allows cancellation
  2. Refund payment (if paid)
  3. Release inventory
  4. order.status = cancelled
Rollback: none (refund is idempotent)
```

---

## Output Template

Copy and fill in — nothing else:

```markdown
## Entities
- <Entity>: <description>
  + <field>: <validation/notes>
  + <derived_field>: <type> (derived: COUNT|SUM|..., stored|computed)

## Relations
<Parent> → <Child> [1:N/N:1/N:N/Self]  [(required)] [(cascade)] [(business rule: ...)]

## Transitions
<Entity>.<from> → <Entity>.<to>
  Trigger: ...
  Side effect: ...

## Processes
Process: <Name>
Trigger: ...
Steps: 1. ... 2. ...
Rollback: ...
```

## Verification

Do each of these in order — stop at the first failure:

1. **FK required?** If a child can't exist without its parent, the FK must be non-nullable (no orphan records).
2. **Cascade explicit?** For every required FK, you have decided: cascade delete, restrict, or nullify.
3. **State coverage?** For every entity with status, every state is reachable from some trigger — including error and cancel states.
4. **Rollback defined?** Every process with multiple steps has a rollback plan for step N failure.
5. **No implicit state.** If one entity's transition depends on another entity's state, that's a cross-entity trigger — write it inline.
6. **Derived fields?** Every aggregate field has `(derived: ..., stored|computed)` — never leave sync strategy implicit.
