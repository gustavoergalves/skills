# Gang of Four (GoF) Patterns — Creational

Creational patterns deal with object creation mechanisms. They decouple client code from the specifics of how objects are constructed, represented, and configured — increasing flexibility and reuse of existing code.

> **AI skill note:** Every pattern entry follows the standard template: `**Category:**`, `**One-liner:**`, `#### What It Is`, `#### Why It Matters`, `#### TypeScript Example`, `#### Curated Links`. Parse by H3 heading.

---

### Singleton

**Category:** Creational
**One-liner:** Ensures a class has exactly one instance and provides a global access point to it.

#### What It Is

The Singleton pattern restricts instantiation of a class to a single object. The constructor is made private so that no external code can call `new`, and a static `getInstance()` method handles lazy creation: it creates the instance on the first call and returns the cached instance on every subsequent call.

This pattern shows up wherever a single shared resource must be co-ordinated across the entire application — a database connection pool, a configuration store, or a logging service. Without Singleton, callers independently creating their own instances can produce inconsistent state or exhaust scarce resources.

In languages and runtimes where modules are cached (like Node.js), a module-level exported constant is often sufficient to achieve Singleton semantics without the classical pattern's boilerplate. In multithreaded environments, the creation step must be guarded with a lock to prevent multiple threads racing to create the instance simultaneously.

#### Why It Matters

- Guarantees a single point of control over a shared resource (e.g., connection pool, cache).
- Avoids inconsistency caused by multiple instances with diverging state.
- Reduces memory footprint for expensive objects that should not be duplicated.
- Provides a well-known access point without resorting to raw global variables.

#### TypeScript Example

```typescript
// BAD — every caller creates its own database connection pool, wasting resources
class OrderService {
  private db = new DatabasePool({ max: 10 }); // new pool on every instantiation

  async findOrder(id: string): Promise<Order> {
    return this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
  }
}

class InvoiceService {
  private db = new DatabasePool({ max: 10 }); // another pool — now 20 connections open
  // ...
}

// GOOD — a single DatabasePool instance shared across all services
class DatabasePool {
  private static instance: DatabasePool | null = null;
  private pool: Pool;

  private constructor(config: PoolConfig) {
    this.pool = new Pool(config);
  }

  static getInstance(): DatabasePool {
    if (!DatabasePool.instance) {
      DatabasePool.instance = new DatabasePool({ max: 10 });
    }
    return DatabasePool.instance;
  }

  query<T>(sql: string, params: unknown[]): Promise<T> {
    return this.pool.query<T>(sql, params);
  }
}

class OrderService {
  private db = DatabasePool.getInstance(); // shared instance
  async findOrder(id: string): Promise<Order> {
    return this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
  }
}
```

#### Curated Links

- [Singleton — refactoring.guru](https://refactoring.guru/design-patterns/singleton) — Full pattern description with structure diagram, pros/cons, and pseudocode.
- [Creational Patterns Overview — refactoring.guru](https://refactoring.guru/design-patterns/creational-patterns) — Context for all five creational patterns.

---

### Factory Method

**Category:** Creational
**One-liner:** Defines an interface for creating an object but lets subclasses (or overrides) decide which class to instantiate.

#### What It Is

The Factory Method pattern moves the `new` call out of client code and into a dedicated method — the factory method. Client code calls this method without knowing the concrete class it will receive. Subclasses override the factory method to return different concrete implementations, allowing behaviour to vary by subclass without modifying the calling code.

The pattern is sometimes called a "virtual constructor." The factory method can also implement pooling or caching logic: rather than always constructing a fresh object it can return a pre-existing one, which is invisible to callers.

In TypeScript, the factory method is typically a `protected` or `abstract` method in a base class, overridden in concrete subclasses. It also appears as a standalone factory function or static factory on an interface when full subclassing is unnecessary.

#### Why It Matters

- Removes hard dependencies on concrete classes from client code, enabling the Open/Closed Principle (OCP).
- Lets library/framework users inject their own object types without modifying core code.
- Centralises construction logic, making it easier to enforce invariants or add logging.
- Supports object pooling and caching transparently.

#### TypeScript Example

```typescript
// BAD — client code is coupled to a concrete notifier class via direct instantiation
class OrderService {
  notifyUser(order: Order): void {
    const notifier = new EmailNotifier(); // hard-coded; can't swap to SMS without editing here
    notifier.send(order.userId, `Order ${order.id} confirmed.`);
  }
}

// GOOD — factory method decouples the caller from the concrete notifier
interface Notifier {
  send(userId: string, message: string): Promise<void>;
}

abstract class NotificationService {
  // Factory method — subclasses decide which notifier to create
  protected abstract createNotifier(): Notifier;

  async notifyUser(order: Order): Promise<void> {
    const notifier = this.createNotifier();
    await notifier.send(order.userId, `Order ${order.id} confirmed.`);
  }
}

class EmailNotificationService extends NotificationService {
  protected createNotifier(): Notifier {
    return new EmailNotifier(process.env.SMTP_HOST!);
  }
}

class SmsNotificationService extends NotificationService {
  protected createNotifier(): Notifier {
    return new SmsNotifier(process.env.TWILIO_SID!);
  }
}
```

#### Curated Links

- [Factory Method — refactoring.guru](https://refactoring.guru/design-patterns/factory-method) — Full pattern description with UML, pros/cons, and real-world examples.
- [GoF Patterns Catalogue — dofactory.com](https://www.dofactory.com/net/design-patterns) — Pattern index with frequency ratings and applicability notes.

---

### Abstract Factory

**Category:** Creational
**One-liner:** Produces families of related objects without coupling client code to any concrete class.

#### What It Is

Where Factory Method handles the creation of one product, Abstract Factory handles an entire *family* of related products. A factory interface declares one creation method per product type (e.g., `createRepository()`, `createEventPublisher()`). Concrete factory implementations produce a consistent variant of the whole family — for example, a `PostgresInfrastructureFactory` creates a `PostgresOrderRepository` and a `PostgresEventPublisher`, while a `InMemoryInfrastructureFactory` creates in-memory stubs for both.

This consistency guarantee is the key distinction. Because both products come from the same factory, they are guaranteed to be compatible with each other. Client code receives the factory via dependency injection and never learns which concrete products it is working with.

Abstract Factory is the pattern behind most infrastructure abstraction layers and test double factories. Switching from a real database to an in-memory test double is as simple as swapping the factory.

#### Why It Matters

- Guarantees that products from the same factory are always compatible.
- Eliminates concrete class references from business logic, enabling the Dependency Inversion Principle (DIP).
- Swapping an entire infrastructure stack (e.g., Postgres ↔ in-memory) requires changing only the factory registration.
- Simplifies testing: inject a `TestInfrastructureFactory` to get consistent, fast fakes for all dependencies.

#### TypeScript Example

```typescript
// BAD — service directly imports concrete Postgres classes; testing requires a real database
import { PostgresOrderRepository } from './postgres-order-repository';
import { SqsEventPublisher } from './sqs-event-publisher';

class OrderApplicationService {
  private repo = new PostgresOrderRepository();
  private events = new SqsEventPublisher();
  // ...
}

// GOOD — Abstract Factory abstracts the entire infrastructure family
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

interface EventPublisher {
  publish(event: DomainEvent): Promise<void>;
}

interface InfrastructureFactory {
  createOrderRepository(): OrderRepository;
  createEventPublisher(): EventPublisher;
}

class PostgresInfrastructureFactory implements InfrastructureFactory {
  createOrderRepository(): OrderRepository {
    return new PostgresOrderRepository(DatabasePool.getInstance());
  }
  createEventPublisher(): EventPublisher {
    return new SqsEventPublisher(process.env.SQS_URL!);
  }
}

class InMemoryInfrastructureFactory implements InfrastructureFactory {
  createOrderRepository(): OrderRepository { return new InMemoryOrderRepository(); }
  createEventPublisher(): EventPublisher { return new InMemoryEventPublisher(); }
}

class OrderApplicationService {
  private repo: OrderRepository;
  private events: EventPublisher;

  constructor(factory: InfrastructureFactory) {
    this.repo = factory.createOrderRepository();
    this.events = factory.createEventPublisher();
  }
}
```

#### Curated Links

- [Abstract Factory — refactoring.guru](https://refactoring.guru/design-patterns/abstract-factory) — Full description with product-family diagram and applicability guidance.
- [Design Patterns Overview — sourcemaking.com](https://sourcemaking.com/design_patterns) — Creational patterns with intent and motivation discussion.

---

### Builder

**Category:** Creational
**One-liner:** Constructs complex objects step by step, separating construction from representation.

#### What It Is

The Builder pattern splits the construction of a complex object from the code that uses it. A `Builder` interface (or class) exposes a fluent API of step methods — `setName()`, `addItem()`, `setShippingAddress()` — and a terminal `build()` method that returns the fully configured product. The builder collects configuration incrementally, validating and assembling the final object only at the end.

This pattern directly solves the "telescoping constructor" problem: when a class has many optional parameters, callers must either pass `undefined`/`null` for parameters they don't need or memorise a large number of overloaded signatures. A builder makes each option explicit and self-documenting.

An optional `Director` class can encapsulate a specific sequence of builder calls for a predefined configuration (e.g., a "standard order" vs. a "priority order"). The director is optional — callers can assemble steps directly when they need a custom configuration.

#### Why It Matters

- Eliminates unreadable telescoping constructors with long optional-parameter lists.
- Makes it possible to construct the same type of object through different step combinations.
- Allows deferred validation: each step can be checked individually before `build()` is called.
- Enables construction of immutable objects from mutable intermediate state.

#### TypeScript Example

```typescript
// BAD — telescoping constructor; caller must pass undefined for every unused option
class Order {
  constructor(
    public userId: string,
    public items: OrderItem[],
    public discountCode?: string,
    public giftMessage?: string,
    public shippingPriority?: 'standard' | 'express',
    public invoiceRequired?: boolean,
  ) {}
}

// Hard to read — what does the 4th argument mean?
const order = new Order('usr_1', items, undefined, undefined, 'express', true);

// GOOD — Builder makes each option explicit and optional
class OrderBuilder {
  private userId!: string;
  private items: OrderItem[] = [];
  private discountCode?: string;
  private giftMessage?: string;
  private shippingPriority: 'standard' | 'express' = 'standard';
  private invoiceRequired = false;

  forUser(userId: string): this { this.userId = userId; return this; }
  withItems(items: OrderItem[]): this { this.items = items; return this; }
  withDiscount(code: string): this { this.discountCode = code; return this; }
  withGiftMessage(msg: string): this { this.giftMessage = msg; return this; }
  asExpressShipping(): this { this.shippingPriority = 'express'; return this; }
  requireInvoice(): this { this.invoiceRequired = true; return this; }

  build(): Order {
    if (!this.userId) throw new Error('userId is required');
    if (this.items.length === 0) throw new Error('Order must have at least one item');
    return new Order(this.userId, this.items, this.discountCode, this.giftMessage,
      this.shippingPriority, this.invoiceRequired);
  }
}

const order = new OrderBuilder()
  .forUser('usr_1')
  .withItems(items)
  .asExpressShipping()
  .requireInvoice()
  .build();
```

#### Curated Links

- [Builder — refactoring.guru](https://refactoring.guru/design-patterns/builder) — Full description including Director, pros/cons, and multi-language examples.
- [Creational Patterns Overview — refactoring.guru](https://refactoring.guru/design-patterns/creational-patterns) — Overview and comparison of all five creational patterns.

---

### Prototype

**Category:** Creational
**One-liner:** Creates new objects by cloning an existing instance rather than constructing from scratch.

#### What It Is

The Prototype pattern lets you duplicate objects without coupling to their concrete classes. Objects that support cloning implement a `Cloneable` interface with a `clone()` method. The clone performs a copy — shallow or deep depending on the object's structure — and returns a new instance with identical state. Client code calls `clone()` on any `Cloneable` without knowing what type it is dealing with.

This is valuable when object construction is expensive or complex — for example, an object loaded from a database with many nested associations. Rather than re-querying for every new instance, a single loaded prototype is cloned and then modified for each variant.

A *prototype registry* (sometimes called a prototype cache) stores pre-configured objects keyed by a name or identifier. Clients retrieve and clone prototypes from the registry, avoiding repeated setup. This is particularly common in game engines (enemy types) and document editors (pre-configured templates).

#### Why It Matters

- Avoids expensive re-initialization for complex objects that can be cloned instead.
- Reduces subclass proliferation: instead of a subclass per configuration, use a pre-configured prototype.
- Enables runtime object creation without knowing concrete classes.
- Prototype registries provide a convenient pool of reusable pre-configured instances.

#### TypeScript Example

```typescript
// BAD — each report template is re-constructed from scratch, repeating expensive setup
async function generateMonthlyReport(month: string): Promise<Report> {
  const template = new Report();
  await template.loadSchema('monthly');       // expensive DB call
  await template.loadStyling('corporate');    // expensive DB call
  template.setTitle(`Monthly Report — ${month}`);
  template.setPeriod(month);
  return template;
}

// Called hundreds of times — same schema/styling loaded every time

// GOOD — Prototype caches a loaded template; each call clones and customises it
interface Cloneable<T> {
  clone(): T;
}

class ReportTemplate implements Cloneable<ReportTemplate> {
  private schema!: ReportSchema;
  private styling!: ReportStyling;
  private title = '';
  private period = '';

  async init(schemaName: string, stylingName: string): Promise<void> {
    this.schema = await ReportSchemaLoader.load(schemaName);   // expensive — done once
    this.styling = await StylingLoader.load(stylingName);      // expensive — done once
  }

  setTitle(title: string): this { this.title = title; return this; }
  setPeriod(period: string): this { this.period = period; return this; }

  clone(): ReportTemplate {
    const copy = Object.create(Object.getPrototypeOf(this)) as ReportTemplate;
    // schema and styling are immutable, so shallow copy is safe
    Object.assign(copy, this);
    return copy;
  }
}

// Bootstrap: load the prototype once
const monthlyTemplate = new ReportTemplate();
await monthlyTemplate.init('monthly', 'corporate');

// Runtime: clone and customise — no DB calls
function generateMonthlyReport(month: string): ReportTemplate {
  return monthlyTemplate.clone().setTitle(`Monthly Report — ${month}`).setPeriod(month);
}
```

#### Curated Links

- [Prototype — refactoring.guru](https://refactoring.guru/design-patterns/prototype) — Full description with prototype registry example and applicability guidance.
- [GoF Patterns Catalogue — dofactory.com](https://www.dofactory.com/net/design-patterns) — Pattern applicability ratings and structural diagrams.

---
