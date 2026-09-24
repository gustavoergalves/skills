# Pattern Decision Guide, Relationships & Quick Reference

> **AI skill note:** Use this file for three things:
> 1. **Pattern selection** — match a problem description to a Gang of Four (GoF) pattern in the Pattern Decision Guide tables.
> 2. **Pattern composition** — reason about combining or replacing patterns using the Pattern Relationships section.
> 3. **Single-glance lookup** — use the Quick Reference Cheat Sheet for a one-row summary of all 24 entries.

---

## Pattern Decision Guide

This section maps common backend engineering problems to the GoF pattern most likely to resolve them. Match the problem description to a row; the pattern column gives the recommended starting point. Refer to the full pattern entry in the relevant section file for implementation detail.

### Creational: Which pattern should I use to create objects?

| Problem | Recommended Pattern | Why |
|---|---|---|
| I need a single shared resource (DB pool, config, logger) with no duplicate instances | **Singleton** | Guarantees one instance; provides a global access point |
| I need to decouple object creation from the calling code, or let subclasses decide the type | **Factory Method** | Moves `new` into an overridable method; caller sees only the interface |
| I need to swap out an entire family of related objects together (e.g., real vs. in-memory infra) | **Abstract Factory** | One factory per variant; all products guaranteed compatible |
| My constructor has too many optional parameters and callers can't tell the arguments apart | **Builder** | Fluent step-by-step API; terminal `build()` validates the result |
| Creating an object is expensive; I need many similar instances cheaply | **Prototype** | Clone a pre-initialised prototype; skip re-construction cost |

### Structural: Which pattern should I use to compose objects?

| Problem | Recommended Pattern | Why |
|---|---|---|
| I need to use a third-party or legacy library but its interface doesn't match mine | **Adapter** | Wraps the incompatible object behind your domain interface |
| I need to add cross-cutting concerns (logging, caching, retry) without modifying the original class | **Decorator** | Wrap the object; each concern is a composable layer |
| I need to hide a complex multi-step subsystem behind a simple entry point | **Facade** | Exposes only the common workflow; subsystem internals stay hidden |
| I need to control access to an object — lazy init, caching, auth checks, or circuit-breaking | **Proxy** | Same interface as the real object; intercepts calls before/after |
| I have a tree structure and I want leaf nodes and containers to be treated identically | **Composite** | Shared interface for leaves and composites; recursive delegation |

### Behavioral: Which pattern should I use to coordinate object communication?

| Problem | Recommended Pattern | Why |
|---|---|---|
| When X happens, multiple unrelated things need to react, and I don't want X to know about them | **Observer** | Publisher emits; subscribers register independently |
| I have a `switch/if` that selects among algorithms, and it grows with every new case | **Strategy** | Each algorithm is a class; inject the one you need |
| I need to queue, log, schedule, or undo an operation | **Command** | Encapsulate the operation as an object with `execute()`/`undo()` |
| I need to traverse or stream a collection without exposing its internal structure | **Iterator** | Implement `Symbol.iterator`; callers use `for...of` |
| Multiple classes share the same multi-step algorithm but differ in one or two steps | **Template Method** | Skeleton in base class; variable steps are abstract/overridden |
| I have a linear pipeline (validation, auth, middleware) where each step may reject early | **Chain of Responsibility** | Each handler checks and either handles or passes to the next |

### Anti-Pattern Diagnosis

| Symptom | Anti-Pattern / Smell | Primary Fix |
|---|---|---|
| One class does everything; changing one thing breaks something unrelated | **God Object** | Decompose with SRP; extract Strategy/Observer/Facade |
| Business logic, DB queries, HTTP calls all tangled in one function | **Spaghetti Code** | Introduce layers; extract Service/Repository/Command |
| Files full of commented-out code and unused methods nobody dares delete | **Lava Flow** | Add coverage, run `ts-prune`, delete dead paths |
| Same heavy tool (event bus, queue, framework) used for everything | **Golden Hammer** | Program to interfaces; inject Strategy/Abstract Factory |
| Changing one rule requires editing 10+ files | **Shotgun Surgery** | Centralise with Template Method, Decorator, or Strategy |
| A method accesses another class's fields more than its own | **Feature Envy** | Move the method to the class it's envious of |
| Raw strings/numbers used for domain concepts; validation duplicated everywhere | **Primitive Obsession** | Extract Value Objects |
| The same 4–5 parameters appear together in every method signature | **Data Clumps** | Extract a Value Object; pass the object instead |

---

## Pattern Relationships

Understanding how patterns relate helps you compose them correctly and avoid applying conflicting solutions to the same problem.

### Decorator vs. Proxy

Both wrap an object behind the same interface and delegate calls. The distinction is *intent*:

- **Decorator** adds behaviour (logging, caching, retry) — it *enriches* the target.
- **Proxy** controls access — it *guards* the target (lazy init, auth, circuit-breaking).

In practice the implementation is nearly identical; the name signals your intent to future readers. If you're adding a concern, use Decorator; if you're managing when the real object is called, use Proxy.

### Decorator vs. Chain of Responsibility

Both wrap or chain objects sharing a common interface. The distinction is *propagation*:

- **Decorator** always delegates to the next layer — every decorator in the stack runs.
- **Chain of Responsibility** handlers may short-circuit — a handler either processes the request or passes it on, but not necessarily both.

Use Decorator for stacked transformations; use Chain of Responsibility when any handler might terminate the pipeline early (e.g., auth rejection, validation failure).

### Strategy vs. Template Method

Both encapsulate varying algorithmic steps. The distinction is *mechanism*:

- **Strategy** uses composition — the algorithm is a separate injected object; it can change at runtime.
- **Template Method** uses inheritance — the skeleton is in a base class; subclasses override steps at compile time.

Prefer Strategy when the algorithm variant may change at runtime or when you want to avoid subclassing. Use Template Method when the algorithm structure is stable, the steps are tightly related, and subclassing is acceptable.

### Factory Method vs. Abstract Factory

Both decouple object creation from the caller. The distinction is *scope*:

- **Factory Method** creates one type of product; subclasses decide the concrete type.
- **Abstract Factory** creates a *family* of related products; the entire family comes from one factory implementation.

Use Factory Method when you need to defer creation of a single product. Use Abstract Factory when you need to ensure compatibility among multiple products (e.g., all infrastructure objects come from the same Postgres or InMemory factory).

### Observer vs. Command

Both are event-driven patterns. The distinction is *directionality and storage*:

- **Observer** is a publish/subscribe mechanism — the publisher notifies many subscribers synchronously.
- **Command** encapsulates a single request as an object that can be queued, logged, deferred, and undone.

They are often used together: an Observer subscriber might enqueue a Command for deferred processing. Use Observer for in-process fan-out; use Command when you need deferral, undo/redo, or audit trails.

### Composite + Iterator

**Composite** defines tree structures with uniform node interfaces. **Iterator** traverses them without exposing internal structure. The two are natural partners: a Composite tree's `[Symbol.iterator]()` can yield nodes in any traversal order (depth-first, breadth-first) while callers use a simple `for...of`.

### Facade + Abstract Factory

A **Facade** simplifies access to a subsystem. An **Abstract Factory** provides the subsystem's objects. Combining them: the Abstract Factory creates the subsystem's concrete components (real vs. test doubles), and the Facade exposes a stable high-level API over whatever family the factory provided. This is the standard pattern for testable infrastructure layers.

### Adapter + Facade

Both wrap other objects, but:

- **Adapter** translates an incompatible interface into a compatible one — the *shape* changes.
- **Facade** simplifies a complex subsystem — the *depth* changes.

An Adapter might sit behind a Facade: the Facade provides the simple API, and internally uses Adapters to integrate with third-party libraries that have incompatible interfaces.

---

## Quick Reference Cheat Sheet

| Name | Category | One-liner | Resolves / Caused By |
|---|---|---|---|
| **Singleton** | GoF — Creational | Ensure a class has exactly one instance with a global access point | Resolves: duplicate resource initialisation (DB pools, config managers) |
| **Factory Method** | GoF — Creational | Delegate object creation to subclasses, decoupling the caller from concrete types | Resolves: concrete-class coupling at construction sites |
| **Abstract Factory** | GoF — Creational | Produce families of related objects without specifying their concrete classes | Resolves: mismatched product variants, Golden Hammer lock-in |
| **Builder** | GoF — Creational | Construct complex objects step by step, avoiding telescoping constructors | Resolves: Primitive Obsession in constructors, unreadable positional arguments |
| **Prototype** | GoF — Creational | Copy existing objects without depending on their concrete class | Resolves: costly repeated initialisation of complex or async-initialised objects |
| **Adapter** | GoF — Structural | Translate one interface into another so incompatible classes can collaborate | Resolves: third-party API mismatches, legacy interface incompatibilities |
| **Decorator** | GoF — Structural | Attach new behaviour to objects at runtime by wrapping them in same-interface decorators | Resolves: class explosion from subclassing, rigid compile-time feature composition |
| **Facade** | GoF — Structural | Provide a simplified, unified interface over a complex subsystem | Resolves: God Object coupling, Spaghetti Code entry points |
| **Proxy** | GoF — Structural | Control access to an object, adding lazy init, caching, or auth checks transparently | Resolves: eager resource loading, cross-cutting access control concerns |
| **Composite** | GoF — Structural | Compose objects into trees and treat individual elements and groups uniformly | Resolves: type-check branching on tree structures (if leaf / if branch) |
| **Observer** | GoF — Behavioral | Notify many subscribers automatically when a publisher's state changes | Resolves: tight coupling between publishers and consumers; Shotgun Surgery |
| **Strategy** | GoF — Behavioral | Define a family of algorithms and make them interchangeable at runtime | Resolves: Spaghetti Code switch/if chains, God Object algorithm accumulation |
| **Command** | GoF — Behavioral | Encapsulate a request as an object, enabling queuing, logging, and undo/redo | Resolves: scattered imperative logic, lack of operation history |
| **Iterator** | GoF — Behavioral | Traverse a collection's elements without exposing its internal structure | Resolves: client coupling to collection internals, duplicate traversal logic |
| **Template Method** | GoF — Behavioral | Define an algorithm skeleton in a base class; let subclasses override specific steps | Resolves: Shotgun Surgery across similar algorithm variants |
| **Chain of Responsibility** | GoF — Behavioral | Pass a request along a handler chain until one processes it | Resolves: nested conditionals for multi-step request dispatch |
| **God Object** | Anti-Pattern | A class that knows and does too much, accumulating unrelated responsibilities | Caused by: SRP violations; Resolved by: SRP, Strategy, Observer, Facade |
| **Spaghetti Code** | Anti-Pattern | Tangled, unstructured code with no clear layers or separation of concerns | Caused by: SRP + OCP violations; Resolved by: Command, Template Method, Strategy |
| **Lava Flow** | Anti-Pattern | Dead or vestigial code left from experiments that cannot safely be removed | Caused by: OCP + ISP violations; Resolved by: OCP, ISP, Facade, static analysis |
| **Golden Hammer** | Anti-Pattern | Over-reliance on a familiar tool applied to every problem regardless of suitability | Caused by: DIP + ISP violations; Resolved by: Strategy, Abstract Factory, Bridge |
| **Shotgun Surgery** | Code Smell | A single logical change requires many small modifications scattered across many files | Caused by: SRP violations (fragmented responsibility); Resolved by: Template Method, Decorator |
| **Feature Envy** | Code Smell | A method is more interested in another class's data than its own | Caused by: poor encapsulation, SRP violation; Resolved by: Move Method, SRP |
| **Primitive Obsession** | Code Smell | Overuse of language primitives to represent domain concepts that deserve their own types | Caused by: missing value objects; Resolved by: State, Strategy, Decorator |
| **Data Clumps** | Code Smell | Groups of variables that always appear together, suggesting a missing class | Caused by: SRP + DRY violations; Resolved by: Value Object, Factory Method |

---

## Further Reading

### GoF Design Patterns — Overviews

| Title | URL | Notes |
|---|---|---|
| Creational Patterns — refactoring.guru | https://refactoring.guru/design-patterns/creational-patterns | Overview of all creational patterns |
| Structural Patterns — refactoring.guru | https://refactoring.guru/design-patterns/structural-patterns | Overview of all structural patterns |
| Behavioral Patterns — refactoring.guru | https://refactoring.guru/design-patterns/behavioral-patterns | Overview of all behavioral patterns |
| All GoF Patterns — dofactory.com | https://www.dofactory.com/net/design-patterns | Full reference with UML and C# examples |
| Design Patterns — sourcemaking.com | https://sourcemaking.com/design_patterns | Pattern catalogue with intent, motivation, and applicability |

### GoF Design Patterns — Individual Pages (refactoring.guru)

| Pattern | URL |
|---|---|
| Singleton | https://refactoring.guru/design-patterns/singleton |
| Factory Method | https://refactoring.guru/design-patterns/factory-method |
| Abstract Factory | https://refactoring.guru/design-patterns/abstract-factory |
| Builder | https://refactoring.guru/design-patterns/builder |
| Prototype | https://refactoring.guru/design-patterns/prototype |
| Adapter | https://refactoring.guru/design-patterns/adapter |
| Decorator | https://refactoring.guru/design-patterns/decorator |
| Facade | https://refactoring.guru/design-patterns/facade |
| Proxy | https://refactoring.guru/design-patterns/proxy |
| Composite | https://refactoring.guru/design-patterns/composite |
| Observer | https://refactoring.guru/design-patterns/observer |
| Strategy | https://refactoring.guru/design-patterns/strategy |
| Command | https://refactoring.guru/design-patterns/command |
| Iterator | https://refactoring.guru/design-patterns/iterator |
| Template Method | https://refactoring.guru/design-patterns/template-method |
| Chain of Responsibility | https://refactoring.guru/design-patterns/chain-of-responsibility |

### Anti-Patterns and Code Smells

| Title | URL | Notes |
|---|---|---|
| Feature Envy — refactoring.guru | https://refactoring.guru/smells/feature-envy | Description, refactoring techniques, SOLID connection |
| Primitive Obsession — refactoring.guru | https://refactoring.guru/smells/primitive-obsession | Causes, symptoms, value object refactoring |
| Data Clumps — refactoring.guru | https://refactoring.guru/smells/data-clumps | Extract Class and Introduce Parameter Object |
| Shotgun Surgery — refactoring.guru | https://refactoring.guru/smells/shotgun-surgery | Root causes, Move Method/Field and Inline Class |
| The God Object Anti-Pattern — softwarepatternslexicon.com | https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/the-god-object/ | Symptoms, causes, SOLID/GoF resolution |
| Understanding Anti-Patterns — softwarepatternslexicon.com | https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/ | Spaghetti Code, Lava Flow, Golden Hammer with SOLID mappings |
