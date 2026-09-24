# Gang of Four (GoF) Patterns — Structural

Structural patterns deal with object composition — how to assemble objects and classes into larger structures while keeping them flexible and efficient. Unlike creational patterns (which focus on *creating* objects) and behavioral patterns (which focus on *communication* between objects), structural patterns focus on the *relationships* between objects and the shape of the resulting system.

> **AI skill note:** Every pattern entry follows the standard template: `**Category:**`, `**One-liner:**`, `#### What It Is`, `#### Why It Matters`, `#### TypeScript Example`, `#### Curated Links`. Parse by H3 heading.

---

### Adapter

**Category:** Structural
**One-liner:** Translates one interface into another so that incompatible classes can work together.

#### What It Is

The Adapter pattern acts as a wrapper between two incompatible interfaces, allowing existing classes or third-party libraries to be used without modifying their source code. The adapter implements the interface your code expects and delegates calls internally to the wrapped object that has the incompatible interface.

There are two implementation variants. **Object adapter** uses composition: the adapter holds a reference to the adaptee object. **Class adapter** uses multiple inheritance (only possible in languages that support it). Object adapters are generally preferred because they are more flexible and work with subclasses of the adaptee as well.

Adapter is particularly valuable when integrating legacy code, third-party libraries, or external services into a system that uses a different domain vocabulary. Rather than littering your business logic with conversions and `as` casts, you isolate the translation in one well-named class.

#### Why It Matters

- Allows reuse of stable, proven code even when its interface does not match your system's contracts.
- Isolates the conversion logic in a single place, making future library upgrades easier — only the adapter needs updating.
- Enables programming to interfaces throughout the domain while wrapping concrete infrastructure at the boundary.
- Facilitates testing by letting you adapter-wrap real infrastructure and stub only the adapter in unit tests.

#### TypeScript Example

```typescript
// BAD — service is directly coupled to the third-party Stripe SDK shape;
// switching payment providers means rewriting business logic
import Stripe from 'stripe';

class PaymentService {
  private stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  async charge(amount: number, currency: string, sourceToken: string): Promise<void> {
    await this.stripe.charges.create({ amount, currency, source: sourceToken });
  }
}

// GOOD — define a domain interface; wrap each provider in an adapter
interface PaymentGateway {
  charge(amountInCents: number, currency: string, paymentToken: string): Promise<void>;
}

// Adapter for Stripe
class StripePaymentAdapter implements PaymentGateway {
  private client = new Stripe(process.env.STRIPE_SECRET_KEY!);

  async charge(amountInCents: number, currency: string, paymentToken: string): Promise<void> {
    await this.client.charges.create({
      amount: amountInCents,
      currency,
      source: paymentToken,
    });
  }
}

// Adapter for a different provider — same interface, different underlying SDK
class BraintreePaymentAdapter implements PaymentGateway {
  async charge(amountInCents: number, currency: string, paymentToken: string): Promise<void> {
    // delegate to Braintree SDK with its own parameter shape
  }
}

class OrderService {
  constructor(private readonly gateway: PaymentGateway) {}

  async checkout(orderId: string, amountInCents: number): Promise<void> {
    const token = await this.resolvePaymentToken(orderId);
    await this.gateway.charge(amountInCents, 'usd', token);
  }

  private async resolvePaymentToken(orderId: string): Promise<string> { return ''; }
}
```

#### Curated Links

- [Adapter — refactoring.guru](https://refactoring.guru/design-patterns/adapter) — Intent, applicability, implementation steps, and diagram comparing object vs. class adapter.
- [Design Patterns Overview — dofactory.com](https://www.dofactory.com/net/design-patterns) — GoF pattern catalog with real-world usage notes.

---

### Decorator

**Category:** Structural
**One-liner:** Wraps an object in a series of decorator objects to add behaviors at runtime without changing the original class.

#### What It Is

The Decorator pattern extends an object's behaviour dynamically by wrapping it in one or more decorator objects that implement the same interface. Each decorator forwards method calls to the wrapped component while adding its own logic before or after. Because decorators and components share the same interface, they are fully interchangeable and stackable: clients see only the interface, not the layering beneath.

Decorator is the runtime-composable alternative to inheritance-based extension. Inheritance adds behaviour statically and leads to class explosion when multiple independent concerns need to be combined (e.g., `LoggingCachingRetryRepository`). Decorators let each concern live in its own small class and be composed at the call site.

In backend systems, the classic Decorator use cases are: logging, caching, rate-limiting, retry logic, validation, and transaction wrapping — all applied to repositories, services, or HTTP clients without modifying the underlying implementation.

#### Why It Matters

- Adheres to the Open/Closed Principle (OCP): add behaviour by composing new decorators, not by modifying the base class.
- Allows independent cross-cutting concerns (logging, caching, retry) to be applied in any combination.
- Simpler than creating subclass hierarchies for every combination of features.
- Decorators can be removed or reordered at runtime without touching business logic.

#### TypeScript Example

```typescript
// BAD — all cross-cutting concerns baked directly into the repository;
// hard to reuse or toggle them independently
class UserRepository {
  async findById(id: string): Promise<User | null> {
    console.log(`[LOG] findById called with ${id}`);
    const cacheKey = `user:${id}`;
    const cached = cache.get(cacheKey);
    if (cached) return cached as User;
    const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
    cache.set(cacheKey, user);
    return user;
  }
}

// GOOD — each concern is its own decorator; compose them at the DI root
interface UserRepository {
  findById(id: string): Promise<User | null>;
}

class PostgresUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

class LoggingUserRepository implements UserRepository {
  constructor(private readonly inner: UserRepository) {}

  async findById(id: string): Promise<User | null> {
    console.log(`[LOG] findById(${id})`);
    const result = await this.inner.findById(id);
    console.log(`[LOG] findById(${id}) → ${result ? 'found' : 'not found'}`);
    return result;
  }
}

class CachingUserRepository implements UserRepository {
  private cache = new Map<string, User>();

  constructor(private readonly inner: UserRepository) {}

  async findById(id: string): Promise<User | null> {
    if (this.cache.has(id)) return this.cache.get(id)!;
    const user = await this.inner.findById(id);
    if (user) this.cache.set(id, user);
    return user;
  }
}

// Composition root: caching wraps logging wraps the real repo
const repo: UserRepository =
  new CachingUserRepository(
    new LoggingUserRepository(
      new PostgresUserRepository()
    )
  );
```

#### Curated Links

- [Decorator — refactoring.guru](https://refactoring.guru/design-patterns/decorator) — Full description with wrapper layering diagrams and applicability guidance.
- [Structural Patterns Overview — refactoring.guru](https://refactoring.guru/design-patterns/structural-patterns) — Overview of all structural patterns with intent comparison.

---

### Facade

**Category:** Structural
**One-liner:** Provides a simple, unified interface to a complex subsystem.

#### What It Is

A Facade is a class that wraps a complex subsystem behind a higher-level interface that is easier to use for the most common operations. It does not prevent clients from accessing the subsystem directly when they need full control — it simply provides a convenient shortcut that covers the typical workflows.

The Facade reduces coupling between subsystems. When two subsystems need to communicate, they can go through each other's facades rather than directly referencing internal classes. This makes it much easier to change either subsystem's internals without rippling changes throughout the codebase.

In practice, Facades appear at the boundaries of major subsystems: an `EmailService` that hides template rendering + SMTP + bounce tracking; an `AnalyticsService` that hides segment calls + internal event logging; or an `AuthFacade` that hides token validation + role resolution + audit logging.

#### Why It Matters

- Simplifies integration: the consumer only needs to know the facade's API, not the subsystem's internals.
- Reduces coupling — subsystem changes propagate at most to the facade, not to every caller.
- Provides a clear entry point for each layer, making system boundaries explicit and discoverable.
- Makes testing easier by letting you replace the facade with a stub rather than mocking many sub-services.

#### TypeScript Example

```typescript
// BAD — application handler directly orchestrates four internal services;
// every handler that sends a notification must repeat this orchestration
async function handleOrderPlaced(orderId: string): Promise<void> {
  const template = await templateEngine.render('order-placed', { orderId });
  const addresses = await recipientResolver.resolveFor(orderId);
  const message = smtpClient.buildMessage(addresses, template);
  await smtpClient.send(message);
  await bounceTracker.registerDelivery(orderId, addresses);
}

// GOOD — Facade hides the four-step notification pipeline
class NotificationFacade {
  constructor(
    private readonly templateEngine: TemplateEngine,
    private readonly recipientResolver: RecipientResolver,
    private readonly smtpClient: SmtpClient,
    private readonly bounceTracker: BounceTracker,
  ) {}

  async sendOrderPlaced(orderId: string): Promise<void> {
    const template = await this.templateEngine.render('order-placed', { orderId });
    const addresses = await this.recipientResolver.resolveFor(orderId);
    const message = this.smtpClient.buildMessage(addresses, template);
    await this.smtpClient.send(message);
    await this.bounceTracker.registerDelivery(orderId, addresses);
  }

  async sendShipmentDispatched(orderId: string, trackingCode: string): Promise<void> {
    const template = await this.templateEngine.render('shipment-dispatched', { orderId, trackingCode });
    const addresses = await this.recipientResolver.resolveFor(orderId);
    const message = this.smtpClient.buildMessage(addresses, template);
    await this.smtpClient.send(message);
    await this.bounceTracker.registerDelivery(orderId, addresses);
  }
}

// Application handler only depends on the facade
async function handleOrderPlaced(orderId: string, notifications: NotificationFacade): Promise<void> {
  await notifications.sendOrderPlaced(orderId);
}
```

#### Curated Links

- [Facade — refactoring.guru](https://refactoring.guru/design-patterns/facade) — Full description with subsystem layering diagram and applicability guidance.
- [Design Patterns Overview — sourcemaking.com](https://sourcemaking.com/design_patterns) — Structural patterns with motivation and trade-off discussion.

---

### Proxy

**Category:** Structural
**One-liner:** Provides a surrogate that controls access to another object, adding pre/post-processing without changing the original.

#### What It Is

A Proxy implements the same interface as the real service object and intercepts calls to it, adding logic before or after delegating. Unlike Facade (which simplifies an interface) or Decorator (which adds behaviour dynamically), a Proxy's primary concern is *access control* — managing *when* and *whether* the real object is called.

There are several practical variants. A **virtual proxy** defers the creation of an expensive object until it is actually needed (lazy initialisation). An **access proxy** checks credentials before forwarding calls. A **caching proxy** memoises results to avoid redundant computation or network calls. A **logging proxy** records all calls for auditing.

In Node.js backends, proxies often appear as HTTP client wrappers that add retry logic, circuit-breaking, or response caching; or as repository wrappers that enforce read-only access in query services.

#### Why It Matters

- Enables lazy initialisation — expensive resources (database connections, large datasets) are not loaded until required.
- Adds a consistent access-control or audit layer without modifying the real object.
- Caching proxies can dramatically reduce load on expensive downstream services.
- The real object remains unchanged and can be used directly in contexts that don't need the proxy.

#### TypeScript Example

```typescript
// BAD — every call to the external pricing service hits the network;
// no caching, no circuit-breaking, latency accumulates under load
class PricingService {
  async getPrice(productId: string): Promise<number> {
    const response = await fetch(`https://pricing-api.internal/prices/${productId}`);
    const { price } = await response.json();
    return price;
  }
}

// GOOD — Caching Proxy wraps the real service; callers see the same interface
interface PricingService {
  getPrice(productId: string): Promise<number>;
}

class HttpPricingService implements PricingService {
  async getPrice(productId: string): Promise<number> {
    const response = await fetch(`https://pricing-api.internal/prices/${productId}`);
    const { price } = await response.json();
    return price;
  }
}

class CachingPricingProxy implements PricingService {
  private cache = new Map<string, { price: number; expiresAt: number }>();
  private readonly ttlMs = 60_000; // 60 seconds

  constructor(private readonly real: PricingService) {}

  async getPrice(productId: string): Promise<number> {
    const cached = this.cache.get(productId);
    if (cached && cached.expiresAt > Date.now()) return cached.price;

    const price = await this.real.getPrice(productId);
    this.cache.set(productId, { price, expiresAt: Date.now() + this.ttlMs });
    return price;
  }
}

// Composition root
const pricingService: PricingService = new CachingPricingProxy(new HttpPricingService());
```

#### Curated Links

- [Proxy — refactoring.guru](https://refactoring.guru/design-patterns/proxy) — Full description covering virtual, protection, remote, caching, and logging proxy variants.
- [Structural Patterns Overview — refactoring.guru](https://refactoring.guru/design-patterns/structural-patterns) — Overview of all structural patterns with intent comparison.

---

### Composite

**Category:** Structural
**One-liner:** Composes objects into tree structures and lets clients treat individual objects and compositions uniformly.

#### What It Is

The Composite pattern defines a common `Component` interface for both *leaves* (objects with no children) and *composites* (objects that contain a collection of children). Because they share the same interface, client code can call the same methods on a single leaf or an entire subtree without knowing which it is talking to. Composites delegate operations recursively to their children and aggregate the results.

This pattern is the natural model whenever the core domain has part-whole hierarchies: file systems (files and folders), organisational charts (employees and teams), permission trees, discount rules, or pipeline stages. The key insight is that a *folder of folders* and a *file* should be interchangeable from the client's perspective.

Without Composite, client code is forced to distinguish between leaves and containers with `if/else` type checks, and any addition of a new node type requires changes throughout the codebase.

#### Why It Matters

- Eliminates `if/else` type checks on node types — all nodes respond to the same interface.
- New leaf and composite types can be added without changing client code (Open/Closed Principle).
- Recursive delegation keeps complex tree-walking logic confined to the nodes, not scattered across callers.
- Simplifies operations that naturally aggregate (totals, permissions, validations) across arbitrary tree depths.

#### TypeScript Example

```typescript
// BAD — client must distinguish between a single discount and a discount group
interface Discount {
  type: 'fixed' | 'percentage' | 'group';
  amount?: number;
  children?: Discount[];
}

function calculateDiscount(price: number, discount: Discount): number {
  if (discount.type === 'fixed') {
    return price - (discount.amount ?? 0);
  } else if (discount.type === 'percentage') {
    return price * (1 - (discount.amount ?? 0) / 100);
  } else if (discount.type === 'group') {
    // must manually recurse and know about 'children'
    return (discount.children ?? []).reduce(
      (acc, child) => calculateDiscount(acc, child),
      price,
    );
  }
  return price;
}

// GOOD — Composite: leaves and groups share the same interface
interface DiscountRule {
  apply(price: number): number;
}

class FixedDiscount implements DiscountRule {
  constructor(private readonly amount: number) {}
  apply(price: number): number { return price - this.amount; }
}

class PercentageDiscount implements DiscountRule {
  constructor(private readonly percent: number) {}
  apply(price: number): number { return price * (1 - this.percent / 100); }
}

class CompositeDiscount implements DiscountRule {
  private rules: DiscountRule[] = [];

  add(rule: DiscountRule): this { this.rules.push(rule); return this; }

  apply(price: number): number {
    return this.rules.reduce((acc, rule) => rule.apply(acc), price);
  }
}

// Client code is identical regardless of tree depth
const blackFridayDiscount = new CompositeDiscount()
  .add(new PercentageDiscount(20))
  .add(new FixedDiscount(5));

const finalPrice = blackFridayDiscount.apply(100); // 75
```

#### Curated Links

- [Composite — refactoring.guru](https://refactoring.guru/design-patterns/composite) — Full description with tree structure diagram and component/leaf/composite relationship.
- [Design Patterns Overview — dofactory.com](https://www.dofactory.com/net/design-patterns) — GoF pattern catalog with real-world usage notes.

---
