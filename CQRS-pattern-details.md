## CQRS (Command Query Responsibility Segregation)

**Origin & name:** Coined by Greg Young, evolved from Bertrand Meyer's **CQS (Command Query Separation)** principle. CQS says a method should either *change state* (a command) or *return data* (a query), but never both. CQRS takes that principle up to the **architectural level**: split the entire model — and often the data stores — into a write side and a read side.

**One-line definition you can say in an interview:**
> "CQRS separates the read model from the write model. Commands that change state go through one model, queries that read state go through a different model. Instead of one model trying to serve both, you optimize each side independently."

---

### The core problem it solves

In the traditional **CRUD / single-model** approach, the *same* data model (and often the same tables/ORM entities) handles both writes and reads. This works fine until:

- **Reads and writes have wildly different needs.** Writes need normalization, validation, and consistency. Reads need denormalized, pre-joined, fast-to-query shapes. One model forced to do both ends up compromised for both.
- **Read and write loads are asymmetric** — most systems read *far* more than they write (often 100:1). You can't scale them independently with a shared model.
- **Complex domains** — the write side has rich business rules; the read side just wants flat DTOs for screens. Mixing them creates a bloated, hard-to-maintain model.
- **Multiple read shapes** — the same data needs to be presented many different ways (dashboards, search, reports), each wanting a different projection.

CQRS lets each side be modeled, optimized, and scaled **on its own terms**.

---

### How it actually works (the spectrum — this is important)

CQRS is not one thing; it's a **spectrum**. Interviewers love when you show you know it's not all-or-nothing:

**Level 1 — Same database, separate models (lightweight):**
Commands and queries use different code paths / objects, but hit the *same* database. Simplest form. Gets you clean separation with minimal infrastructure.

**Level 2 — Same DB, separate read schema (materialized views):**
Write to normalized tables; read from denormalized views or read-optimized tables in the same database.

**Level 3 — Separate read and write databases (full CQRS):**
The write side owns its store (e.g., normalized Postgres). The read side has its *own* store (e.g., Elasticsearch for search, Redis for fast lookups, a denormalized read replica). The write side publishes events; a **projection/handler** updates the read store.

The key insight to state: **as you move up the spectrum, you gain optimization and scalability but pay with complexity and eventual consistency.** Most systems should stay at Level 1 or 2.

---

### The flow in full CQRS (Level 3)

1. **Command** comes in (`PlaceOrder`) → **Command Handler** validates business rules → writes to the **write store**.
2. The write side **publishes an event** (`OrderPlaced`).
3. A **projector / event handler** consumes the event and updates one or more **read models** (a denormalized "order summary" table, a search index, etc.).
4. **Queries** (`GetOrderHistory`) hit the **read store** directly — no business logic, just fast reads of pre-shaped data.

The gap between step 1 and step 3 is where **eventual consistency** lives — the read side lags the write side by milliseconds to seconds.

---

### Practical Example — E-commerce order system

**Write side:**
- `PlaceOrder`, `CancelOrder`, `AddItem` are commands.
- They run through handlers enforcing rules (stock checks, credit limits) and write to a **normalized relational store** (orders, order_items, customers — clean, consistent, ACID).

**Read side:**
- The "My Orders" page needs order + items + customer + shipping status, all joined, sorted, paginated. Instead of an expensive multi-table join on every page load, you maintain a **denormalized `order_summary` read model** — one row with everything the screen needs.
- The product search page reads from **Elasticsearch**, kept in sync via events.
- An analytics dashboard reads from a **columnar store**.

**Result:** the write side stays clean and correct; each read side is optimized for its specific query pattern. You can scale the read stores (which take 99% of the traffic) independently — add read replicas, cache aggressively — without touching the write path.

---

### The relationship with Event Sourcing (name this — it's the #1 confusion to clear up)

**CQRS and Event Sourcing are separate patterns that are frequently used together but do NOT require each other.** Saying this clearly is a strong senior signal because many people conflate them.

- You can do **CQRS without Event Sourcing** (just separate read/write models over normal stores).
- You can do **Event Sourcing without CQRS** (though in practice ES almost always uses CQRS, because an event log is terrible for querying — you need projections to read from).
- When combined: the write side stores **events** as the source of truth; projections build read models from the event stream. This is the "full" architecture people often picture, but it's the *heavy* end and shouldn't be your default.

---

### How to justify it in an interview (trade-offs)

**Advantages:**
- **Independent optimization** — write model for consistency/rules, read models for query performance.
- **Independent scalability** — scale reads and writes separately (huge for read-heavy systems).
- **Multiple tailored read models** — one write side feeding search, dashboards, reports, each in its ideal store.
- **Simpler models on each side** — no single model bloated by conflicting concerns; often maps cleanly onto DDD bounded contexts.
- **Security/separation** — easier to lock down who can write vs. read.

**Costs / when it's NOT the right choice (say these unprompted — this is what separates seniors):**
- **Significant added complexity** — two models, sync mechanism, more moving parts. This is the biggest cost.
- **Eventual consistency** — the read side lags the write side. The UI must handle "I just saved it but don't see it yet" (e.g., optimistic UI updates, or read-your-own-writes strategies). If your domain needs immediate read-after-write everywhere, CQRS fights you.
- **More infrastructure** — message bus, projections, possibly multiple databases to operate and monitor.
- **Data duplication** — the same data lives in multiple shapes; you must keep them in sync and handle projection rebuilds.
- **Over-engineering risk** — this is the killer. **CQRS is often applied where it shouldn't be.** For a simple CRUD app it's pure overhead. Greg Young himself has warned that most systems don't need it.

---

### A crisp "when would you use it" answer

> "I'd apply CQRS to a specific bounded context that's read-heavy, has complex or divergent read vs. write needs, or needs the same data in several optimized shapes — like an order or catalog subsystem. And I'd apply it *per bounded context*, not across the whole system. I would *not* use it for a simple CRUD domain, or where the business can't tolerate any read lag — the complexity wouldn't pay for itself."

The bounded-context point is important: **CQRS is a tactical pattern you apply selectively, not a system-wide architecture you commit to everywhere.**

---

### Related patterns to name-drop (shows breadth)

- **Event Sourcing** — the frequent partner (clarified above).
- **Materialized View** — the read-model projection is essentially this.
- **Saga** — sagas often coordinate across CQRS/event-sourced services (ties back to our last discussion).
- **Eventual Consistency / Read Replicas** — the underlying mechanics.
- **Database-per-service / Polyglot Persistence** — CQRS naturally enables using the right store for each read model.

---

### Likely interviewer follow-ups (prep these)

1. *"How do you handle the eventual consistency on the read side in the UI?"* → Optimistic updates, return the new state from the command, version numbers, or route the immediate read back to the write model.
2. *"CQRS vs Event Sourcing — what's the difference?"* → Separate patterns; often combined; neither requires the other (answer above).
3. *"How do you keep the read model in sync?"* → Publish domain events; projectors update read stores; handle idempotency and out-of-order events; support projection rebuilds by replaying.
4. *"What if a projection fails or falls behind?"* → Monitoring lag, retryable idempotent handlers, ability to rebuild projections from the event stream/source of truth.
5. *"Isn't this overkill?"* → Yes, usually — and knowing *when not* to use it is the real skill. Apply per bounded context where the read/write asymmetry justifies it.
