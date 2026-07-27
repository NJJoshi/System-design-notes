SOLID isn't a design *pattern* — it's a set of five *design principles* (coined by Robert C. Martin, "Uncle Bob"). Design patterns like Factory or Strategy are concrete, reusable solutions; SOLID principles are higher-level guidelines that patterns often help you *satisfy*. I'll walk through all five with examples and, importantly, where I'd push back on applying them dogmatically — because at an architect level the interesting part is the trade-offs, not the definitions.

## S — Single Responsibility Principle

**"A class should have one reason to change."** The key word is *reason to change*, tied to actors/stakeholders — not "a class should do one thing," which is a common oversimplification.

```java
// Violation: three reasons to change bundled together
class Employee {
    double calculatePay() { ... }        // changes for Finance/payroll rules
    void save() { ... }                  // changes for DBA/persistence
    String generateReport() { ... }      // changes for HR reporting format
}

// Better: separate the actors
class Employee { /* domain data + invariants */ }
class PayrollCalculator { double calculatePay(Employee e) { ... } }
class EmployeeRepository { void save(Employee e) { ... } }
class EmployeeReportGenerator { String generate(Employee e) { ... } }
```

**Pros:** Localized changes, easier testing, less merge-conflict surface across teams.
**Cons / trade-off:** Taken too far it produces class explosion and "anemic" classes where logic is scattered and you chase behavior across ten files. The honest tension is *cohesion vs. fragmentation*. My rule of thumb: split when two responsibilities are owned by different stakeholders or change at different rates. Don't split just to hit "one method per class."

## O — Open/Closed Principle

**"Open for extension, closed for modification."** You should add behavior without editing existing, tested code — usually via polymorphism.

```java
// Violation: every new shape edits this method (and its tests)
double area(Shape s) {
    if (s instanceof Circle) ...
    else if (s instanceof Square) ...
}

// Better: extend by adding a class, not editing existing ones
interface Shape { double area(); }
class Circle implements Shape { public double area() { ... } }
class Square implements Shape { public double area() { ... } }
```

**Pros:** New requirements = new code, existing code stays stable; great when variation is genuinely open-ended (payment providers, export formats).
**Cons / trade-off:** You pay for it with abstraction and indirection *up front*. This is where I'd caution against speculative generality — if you build an elaborate plugin architecture for a variation that never materializes, you've added cost for nothing. ***I lean on the "Rule of Three": tolerate the `if/else`, and refactor to OCP when the third variant actually shows up and the axis of change is clear.***

## L — Liskov Substitution Principle

**"Subtypes must be substitutable for their base type"** without breaking the caller's expectations. It's about behavioral contracts, not just compiling.

```java
// Classic violation
class Rectangle { void setWidth(w); void setHeight(h); }
class Square extends Rectangle {
    // Forces width == height, breaking code that sets them independently
    void setWidth(w)  { this.w = this.h = w; }
}
// A method that sets width=5, height=4 and expects area 20 now silently breaks.
```

The fix is usually to not force an "is-a" relationship that doesn't hold behaviorally — a Square isn't a *mutable* Rectangle. LSP also constrains overrides: don't strengthen preconditions, weaken postconditions, or throw new unexpected exceptions.

**Pros:** Polymorphism you can actually trust; prevents subtle runtime bugs.
**Cons / trade-off:** Real domains have leaky hierarchies (the Ostrich-is-a-Bird-that-can't-fly problem). Enforcing LSP strictly sometimes means favoring composition over inheritance, or splitting interfaces — which can feel unintuitive to people who model taxonomies naively. The trade-off is modeling purity vs. matching messy real-world "is-a" language.

## I — Interface Segregation Principle

**"Don't force clients to depend on methods they don't use."** Prefer many small, role-based interfaces over one fat one.

```java
// Violation: a SimplePrinter forced to implement scan/fax it doesn't support
interface Machine { void print(); void scan(); void fax(); }

// Better
interface Printer { void print(); }
interface Scanner { void scan(); }
// A MultiFunctionDevice implements both; a basic printer implements only Printer.
```

**Pros:** Clients aren't coupled to irrelevant changes; mocks/stubs in tests get trivial; clearer contracts.
**Cons / trade-off:** Interface proliferation. In a large codebase you can end up with dozens of tiny interfaces that obscure the overall shape of a component. There's judgment in granularity — segregate by *client role*, not by "one method per interface."

## D — Dependency Inversion Principle

**"Depend on abstractions, not concretions"** — and high-level policy shouldn't depend on low-level detail; both depend on abstractions.

```java
// Violation: business logic nailed to a specific DB
class OrderService {
    private MySqlOrderRepo repo = new MySqlOrderRepo(); // hard dependency
}

// Better: depend on an interface, inject the concrete impl
interface OrderRepository { void save(Order o); }
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo) { this.repo = repo; } // injected
}
```

**Pros:** Swappable implementations, testability (inject a fake repo), and it's the backbone of clean/hexagonal architecture where the domain has zero infra dependencies.
**Cons / trade-off:** Every abstraction is a cost — indirection, DI wiring, harder "jump to implementation." Over-applied, you get an interface for every class with exactly one implementation, which is pure ceremony. I abstract at *architectural seams* (DB, external services, messaging), not at every internal collaboration.

---

## The meta-answer (what I'd actually stress as an architect)

If I zoom out, a few points I'd want the interviewer to hear:

The unifying theme of SOLID is **managing change and dependencies** — isolating what varies so change stays cheap and localized. Almost every principle trades **upfront complexity and indirection** for **future flexibility and maintainability**. That trade is only worth it when the flexibility is actually exercised.

So the real skill isn't reciting SOLID — it's knowing *when not to apply it*. On a small, stable module, aggressive SOLID is over-engineering; YAGNI and simplicity win. On a core domain touched by many teams and evolving fast, the discipline pays for itself many times over. I treat SOLID as a set of *smells-detectors and refactoring targets*, not a checklist to satisfy on day one. And these principles interact — DIP and OCP usually travel together (you extend by injecting a new implementation behind an abstraction), and ISP supports LSP (smaller contracts are easier to substitute safely).

One honest caveat I'd add: SOLID is object-oriented in origin. In functional or data-oriented styles, some of these map differently — SRP and DIP still resonate, but "interface segregation" or class-based OCP look quite different when you're composing functions or using pattern matching over closed sum types.

Want me to go deeper on any single principle, do a live-coding-style refactor of a messy class through all five, or contrast SOLID with something like CUPID or the trade-offs against YAGNI/KISS? Interviewers often follow up there.