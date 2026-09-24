# OOP Design Patterns — Table of Contents

> **AI skill note:** This file is a navigation index. Do not read it for pattern content — read the section files listed below instead. Load only the section(s) relevant to the current task.

This guide covers Gang of Four (GoF) design patterns, anti-patterns, and code smells for TypeScript. It is split into focused section files so you can load only what you need.

---

## Section Files

| File | Contents | When to load |
|---|---|---|
| `guides/creational-patterns.md` | Singleton, Factory Method, Abstract Factory, Builder, Prototype | Object construction problems; telescoping constructors; shared resources; expensive initialisation |
| `guides/structural-patterns.md` | Adapter, Decorator, Facade, Proxy, Composite | Third-party integration; cross-cutting concerns; subsystem simplification; tree structures |
| `guides/behavioral-patterns.md` | Observer, Strategy, Command, Iterator, Template Method, Chain of Responsibility | Event propagation; algorithm selection; undo/redo; middleware pipelines; collection traversal |
| `guides/anti-patterns.md` | God Object, Spaghetti Code, Lava Flow, Golden Hammer, Shotgun Surgery, Feature Envy, Primitive Obsession, Data Clumps | Diagnosing code smells; identifying structural violations; recommending remediations |
| `guides/pattern-decision-guide.md` | Pattern Decision Guide tables, Pattern Relationships, Quick Reference Cheat Sheet, Further Reading | Selecting a pattern from a problem description; comparing two patterns; single-glance lookup |

---

## Pattern Index

### Creational (→ `guides/creational-patterns.md`)

| Pattern | One-liner |
|---|---|
| **Singleton** | One instance, global access point. |
| **Factory Method** | Subclasses decide which class to instantiate. |
| **Abstract Factory** | Creates families of related objects without coupling to concrete classes. |
| **Builder** | Constructs complex objects step by step via a fluent API. |
| **Prototype** | Creates new objects by cloning an existing instance. |

### Structural (→ `guides/structural-patterns.md`)

| Pattern | One-liner |
|---|---|
| **Adapter** | Translates one interface into another so incompatible classes can work together. |
| **Decorator** | Wraps an object to add behaviours at runtime without changing the original class. |
| **Facade** | Provides a simple, unified interface to a complex subsystem. |
| **Proxy** | Surrogate that controls access to another object (lazy init, caching, auth). |
| **Composite** | Composes objects into tree structures; treats leaves and composites uniformly. |

### Behavioral (→ `guides/behavioral-patterns.md`)

| Pattern | One-liner |
|---|---|
| **Observer** | Subjects notify subscribers automatically when state changes. |
| **Strategy** | Encapsulates interchangeable algorithms; select at runtime. |
| **Command** | Encapsulates a request as an object (undo, queuing, logging). |
| **Iterator** | Provides sequential access to a collection without exposing its internals. |
| **Template Method** | Defines the skeleton of an algorithm; subclasses fill in the steps. |
| **Chain of Responsibility** | Passes a request along a chain of handlers until one handles it. |

### Anti-Patterns & Code Smells (→ `guides/anti-patterns.md`)

| Name | Signal |
|---|---|
| **God Object** | One class that knows or does too much. |
| **Spaghetti Code** | Tangled control flow with no clear structure. |
| **Lava Flow** | Dead code kept out of fear — "don't touch it, it might break". |
| **Golden Hammer** | Applying the same pattern everywhere regardless of fit. |
| **Shotgun Surgery** | One change requires edits scattered across many classes. |
| **Feature Envy** | A method uses another class's data more than its own. |
| **Primitive Obsession** | Using raw primitives instead of small domain value objects. |
| **Data Clumps** | Groups of data that always travel together but aren't encapsulated. |

---

## Entry Template

Every pattern entry in the section files follows this structure:

```
### <Name>

**Category:** <Creational | Structural | Behavioral | Anti-Pattern | Code Smell>
**One-liner:** <One sentence that captures the essence.>

#### What It Is
2–4 paragraphs of theory.

#### Why It Matters
- Bullet list of practical benefits or risks.

#### TypeScript Example
// BAD — description of what is wrong
<code>

// GOOD — description of what is right
<code>

#### Curated Links
- [Title](url) — annotation
```

Anti-patterns and code smells follow the same structure with an added **Resolves/Violated By** field.
