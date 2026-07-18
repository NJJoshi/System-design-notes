## Strangler Pattern (a.k.a. Strangler Fig Pattern)

**Origin & name:** Coined by Martin Fowler, inspired by the *strangler fig* tree that grows around a host tree, gradually taking over until the original tree dies and only the fig remains. In software, you grow a new system *around* a legacy system, incrementally routing functionality to the new one until the legacy system can be safely retired.

**One-line definition you can say in an interview:**
> "The Strangler pattern is an incremental migration strategy where you gradually replace a legacy system by building new functionality around it and rerouting traffic piece by piece, instead of doing a risky big-bang rewrite."

---

### The core problem it solves

Big-bang rewrites (throw away the old system, build a new one, switch over on a Friday night) are notoriously dangerous:
- The rewrite takes years while the old system keeps changing underneath you.
- You can't ship value until the whole thing is done.
- The cutover is high-risk — one bug and you roll back everything.

The Strangler pattern de-risks this by making migration **incremental, reversible, and continuously delivering value**.

---

### How it actually works (the mechanics)

Three moving parts, and interviewers love when you name them explicitly:

1. **Interception / Facade layer** — Put a routing layer in front of the legacy system (an API gateway, reverse proxy, or a facade). All traffic flows through it.
2. **Incremental replacement** — Build one slice of functionality in the new system. Flip the router so that slice now goes to the new code; everything else still hits the legacy system.
3. **Decommission** — Once *all* slices are migrated and stable, remove the legacy system and often the facade too.

The router is the key. It's the thing that lets you migrate one endpoint, one module, or one user segment at a time.

---

### Practical Example 1 — Monolith to microservices (the classic)

Imagine a 15-year-old e-commerce **monolith** handling auth, catalog, cart, orders, payments, and shipping.

- Put an **API gateway** in front of it. Day one, it routes 100% of requests to the monolith. Nothing changes for users.
- You extract the **"Product Catalog"** into a new microservice. Configure the gateway: `/api/products/**` → new catalog service; everything else → monolith.
- Validate it in production (canary/percentage rollout — send 5% of catalog traffic to the new service first, then 100%).
- Next quarter, extract **Payments**. Route `/api/payments/**` to the new service.
- Repeat until the monolith is hollowed out, then delete it.

**Why this is safe:** at every step you can roll back a single route without touching the rest. You're never betting the whole business on one deploy.

---

### Practical Example 2 — Replacing a legacy database / data layer

A mainframe stores customer records; you want to move to a modern Postgres-backed service.

- New writes go to **both** old and new (dual-write) via the facade.
- Reads are gradually switched to the new store, feature-by-feature.
- A reconciliation job verifies parity between old and new.
- Once confidence is high, cut reads fully over, stop dual-writing, retire the mainframe.

This is where you'd mention the **hard part**: data synchronization and consistency during the transition (more on that below — interviewers push here).

---

### Practical Example 3 — Frontend / UI migration

Migrating a legacy JSP/server-rendered app to React.

- A reverse proxy sits in front. New pages (`/dashboard`, `/settings`) are served by the React app; old pages still served by the legacy app.
- Users don't perceive two apps — the URL structure and shell stay consistent.
- Page by page gets migrated until the old frontend is gone.

(This is essentially how micro-frontends and tools like the "strangler" routing in many modern replatforming efforts work.)

---

### How to *justify* it in an interview (trade-offs — this is what separates senior answers)

**Advantages you should cite:**
- **Reduced risk** — small, reversible steps instead of one catastrophic cutover.
- **Continuous value delivery** — the business gets improvements throughout, not after 2 years.
- **Learn as you go** — early migrations teach you patterns for later ones.
- **No feature freeze** — the legacy system keeps running and evolving during migration.

**Costs / when it's NOT the right choice (mention these unprompted — it shows maturity):**
- **You run two systems in parallel** for a long time — operational overhead, double monitoring, double the on-call surface.
- **The facade/routing layer adds complexity** and can become a bottleneck or single point of failure.
- **Data consistency during dual-running is genuinely hard** — you need dual-writes, reconciliation, and a clear source of truth.
- **Discipline required** — teams sometimes never finish. The legacy system lingers for years ("we'll delete it next quarter") and you're stuck maintaining both forever. This is the #1 real-world failure mode.
- **Poor fit for small systems** — if the legacy app is small, a clean rewrite may genuinely be cheaper than the overhead of strangling it.

---

### A crisp "when would you use it" answer

> "I'd use it for a large, business-critical legacy system that can't afford downtime and that the business still depends on daily — like migrating a core monolith to microservices, or replatforming without a feature freeze. I would *not* use it for a small app where a rewrite is quick, or where I can't reasonably put a routing layer in front of the legacy system."

---

### Related patterns to name-drop (shows breadth)

- **Anti-Corruption Layer (ACL)** — often used *with* the strangler to stop the legacy model's design from leaking into the new services.
- **Branch by Abstraction** — a code-level cousin for in-process migrations (vs. the strangler's system/network-level routing).
- **Feature toggles / canary releases** — the mechanism that lets you shift traffic gradually and safely.

---

If it's useful, tell me your domain (payments, healthcare, telecom, etc.) and I'll tailor Example 1 into a story from *your* world so it sounds like lived experience rather than a textbook answer — that tends to land much better in interviews. I can also give you 3–4 likely follow-up questions an interviewer would fire at you and strong answers for each.