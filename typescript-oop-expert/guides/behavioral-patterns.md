# Gang of Four (GoF) Patterns — Behavioral

Behavioral patterns deal with algorithms and the assignment of responsibilities between objects. Where creational and structural patterns focus on *what* gets built and *how* it is arranged, behavioral patterns focus on *how* objects communicate and cooperate at runtime. The six patterns in this section cover the most commonly encountered collaboration problems in backend systems: event propagation, algorithm selection, request encapsulation, collection traversal, algorithmic skeleton reuse, and middleware chains.

> **AI skill note:** Every pattern entry follows the standard template: `**Category:**`, `**One-liner:**`, `#### What It Is`, `#### Why It Matters`, `#### TypeScript Example`, `#### Curated Links`. Parse by H3 heading.

---

### Observer

**Category:** Behavioral
**One-liner:** Define a subscription mechanism so dependents are notified and updated automatically when a subject changes state.

#### What It Is

The Observer pattern decouples the object that produces events (the *publisher* or *subject*) from the objects that react to those events (the *subscribers* or *observers*). Publishers expose `subscribe` and `unsubscribe` methods that accept any object implementing the subscriber interface, then iterate through their registry calling `update` (or a more domain-specific method) whenever state changes.

The key insight is that the publisher never knows the concrete type of its subscribers, only their interface. This means subscribers can be added, removed, or replaced at runtime without touching the publisher's code. New subscriber types — a logger, a webhook notifier, an audit trail writer — slot in without modification.

In backend systems, the Observer pattern appears wherever an event-driven or reactive model is appropriate: domain events published after an aggregate is mutated, a metrics collector listening to service calls, or an in-process event bus coordinating loosely coupled modules. It is the conceptual foundation behind Node.js `EventEmitter`, RxJS Observables, and message-broker consumers.

A common mistake is conflating Observer with a message queue. Observer is in-process and synchronous by default — subscribers are called immediately on the same call stack. For true async decoupling (across services or processes), use a message broker; Observer is the *within-process* analogue of that pattern.

#### Why It Matters

- Eliminates hard-coded dependencies between a publisher and its consumers — adding a new reaction requires no change to the publishing object.
- Supports the Open/Closed Principle (OCP): the publisher is open to extension (new subscribers) but closed to modification.
- Makes it straightforward to trigger multiple reactions to a single event without conditional fan-out logic.
- Enables runtime composition of reactive behaviours (e.g., toggling email notifications without redeploying the core service).

#### TypeScript Example

```typescript
// BAD — OrderService directly calls every downstream action after an order is placed,
// coupling it tightly to email, inventory, and audit systems.
class OrderService {
  placeOrder(order: Order): void {
    this.saveOrder(order);
    emailService.sendConfirmation(order.customerId, order.id);
    inventoryService.reserve(order.items);
    auditLogger.log('order_placed', order.id);
    // Adding another reaction requires editing this method every time.
  }
}

// GOOD — OrderService publishes a domain event; each subscriber handles its own concern.
interface OrderEventSubscriber {
  onOrderPlaced(order: Order): void;
}

class OrderService {
  private subscribers: OrderEventSubscriber[] = [];

  subscribe(subscriber: OrderEventSubscriber): void {
    this.subscribers.push(subscriber);
  }

  unsubscribe(subscriber: OrderEventSubscriber): void {
    this.subscribers = this.subscribers.filter(s => s !== subscriber);
  }

  placeOrder(order: Order): void {
    this.saveOrder(order);
    this.notify(order);
  }

  private saveOrder(order: Order): void { /* persist */ }

  private notify(order: Order): void {
    for (const subscriber of this.subscribers) {
      subscriber.onOrderPlaced(order);
    }
  }
}

class EmailNotificationSubscriber implements OrderEventSubscriber {
  onOrderPlaced(order: Order): void {
    emailService.sendConfirmation(order.customerId, order.id);
  }
}

class InventoryReservationSubscriber implements OrderEventSubscriber {
  onOrderPlaced(order: Order): void {
    inventoryService.reserve(order.items);
  }
}

// Wired at composition root — adding reactions requires zero changes to OrderService.
const orderService = new OrderService();
orderService.subscribe(new EmailNotificationSubscriber());
orderService.subscribe(new InventoryReservationSubscriber());
```

#### Curated Links

- [Observer — refactoring.guru](https://refactoring.guru/design-patterns/observer) — Full pattern description with UML, pros/cons, and multi-language examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all six behavioral patterns
- [GoF Patterns — dofactory.com](https://www.dofactory.com/net/design-patterns) — Classic GoF catalogue with structural diagrams

---

### Strategy

**Category:** Behavioral
**One-liner:** Define a family of algorithms, encapsulate each one, and make them interchangeable so the algorithm can vary independently of the clients that use it.

#### What It Is

The Strategy pattern extracts varying algorithmic behaviour from a class into separate *strategy* objects, each behind a shared interface. A *context* object stores a reference to the active strategy and delegates the computation to it. Clients can inject different strategies at construction time, or swap them at runtime, without modifying the context.

The pattern's primary motivation is eliminating large conditional trees (`if/switch` blocks) that select among algorithmic variants. Instead of branching inside the context, you branch once at composition time by choosing which strategy to inject. The result is that each algorithm lives in isolation, can be tested independently, and can be extended or replaced without touching the context or other strategies.

In backend development, Strategy appears frequently in payment processing (selecting a payment gateway), shipping cost calculation (flat rate vs. weight-based vs. zone-based), discount application (percentage, fixed, buy-one-get-one), report generation (CSV vs. PDF vs. JSON), and authentication (JWT vs. API-key vs. OAuth).

Strategy is closely related to the Open/Closed Principle (OCP): the context is closed for modification but open for extension via new strategy implementations. It differs from Template Method in that Strategy uses *composition* (the algorithm is a separate object) whereas Template Method uses *inheritance* (the algorithm skeleton is in a superclass).

#### Why It Matters

- Replaces conditionals with polymorphism — each branch becomes a standalone, testable class.
- New algorithmic variants can be added without modifying the context or existing strategies.
- Strategies can be unit-tested in complete isolation from the context and from each other.
- Runtime algorithm switching is trivial — swap the strategy reference on the context object.

#### TypeScript Example

```typescript
// BAD — ShippingCalculator uses a switch to select the algorithm, requiring
// modification every time a new shipping method is added.
class ShippingCalculator {
  calculate(order: Order, method: string): number {
    switch (method) {
      case 'flat':
        return 5.99;
      case 'weight':
        return order.weightKg * 2.5;
      case 'express':
        return order.weightKg * 4.0 + 10;
      default:
        throw new Error(`Unknown shipping method: ${method}`);
    }
  }
}

// GOOD — Each shipping algorithm is a separate class behind a common interface.
interface ShippingStrategy {
  calculate(order: Order): number;
}

class FlatRateShipping implements ShippingStrategy {
  calculate(_order: Order): number {
    return 5.99;
  }
}

class WeightBasedShipping implements ShippingStrategy {
  calculate(order: Order): number {
    return order.weightKg * 2.5;
  }
}

class ExpressShipping implements ShippingStrategy {
  calculate(order: Order): number {
    return order.weightKg * 4.0 + 10;
  }
}

class OrderCheckoutService {
  constructor(private readonly shippingStrategy: ShippingStrategy) {}

  calculateTotal(order: Order): number {
    const shippingCost = this.shippingStrategy.calculate(order);
    return order.subtotal + shippingCost;
  }
}

// Adding a new strategy (e.g., FreeTierShipping) requires zero changes to
// OrderCheckoutService or any existing strategy class.
const service = new OrderCheckoutService(new WeightBasedShipping());
```

#### Curated Links

- [Strategy — refactoring.guru](https://refactoring.guru/design-patterns/strategy) — Full pattern description with UML and examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all behavioral patterns
- [GoF Patterns — dofactory.com](https://www.dofactory.com/net/design-patterns) — Classic GoF catalogue with structural diagrams

---

### Command

**Category:** Behavioral
**One-liner:** Encapsulate a request as an object so you can parameterise clients, queue or log requests, and support undoable operations.

#### What It Is

The Command pattern turns an action or request into a self-contained object. Each command object holds everything needed to perform the action: the receiver that will do the work, the method to call, and any required arguments. An *invoker* triggers commands without knowing anything about what they do — it simply calls `execute()`.

This indirection enables capabilities that are impossible with direct method calls. Commands can be stored in a queue and dispatched asynchronously or at a scheduled time. They can be serialised and sent over the wire for remote execution. They can be logged to a history stack and reversed via an `undo()` method. They can be composed into macro-commands that execute multiple actions as a single unit.

In backend systems, Command is the conceptual model behind job/task queues (BullMQ jobs are commands), Command Query Responsibility Segregation (CQRS) write-side command handlers, database migration runners, and undo/redo in collaborative editing. The pattern cleanly separates *what* is requested from *when* and *how* it executes.

A useful distinction: in CQRS, a "command" in the domain sense is not exactly the GoF Command pattern, but shares the same encapsulation philosophy. The GoF pattern is more general — it covers any invocable object with optional undo semantics, not only domain write operations.

#### Why It Matters

- Decouples the invoker from the receiver — neither knows the other's concrete type.
- Enables queueing, scheduling, and asynchronous dispatch of operations without modifying the original logic.
- Provides a natural extension point for logging, metrics, and retry logic on any operation.
- Makes undo/redo implementable without modifying individual business operations — just push/pop the command history stack.

#### TypeScript Example

```typescript
// BAD — InvoiceController directly applies a discount inline, making it
// impossible to queue, log, retry, or undo the operation independently.
class InvoiceController {
  applyDiscount(invoiceId: string, percent: number): void {
    const invoice = this.invoiceRepo.findById(invoiceId);
    invoice.applyDiscount(percent);
    this.invoiceRepo.save(invoice);
    auditLog.record(`discount:${percent}% applied to ${invoiceId}`);
  }
}

// GOOD — Encapsulate the operation as a Command object with execute/undo.
interface Command {
  execute(): void;
  undo(): void;
}

class ApplyDiscountCommand implements Command {
  private previousDiscount: number = 0;

  constructor(
    private readonly invoice: Invoice,
    private readonly percent: number,
    private readonly invoiceRepo: InvoiceRepository,
  ) {}

  execute(): void {
    this.previousDiscount = this.invoice.discountPercent;
    this.invoice.applyDiscount(this.percent);
    this.invoiceRepo.save(this.invoice);
  }

  undo(): void {
    this.invoice.applyDiscount(this.previousDiscount);
    this.invoiceRepo.save(this.invoice);
  }
}

class InvoiceCommandInvoker {
  private history: Command[] = [];

  run(command: Command): void {
    command.execute();
    this.history.push(command);
  }

  undoLast(): void {
    const last = this.history.pop();
    last?.undo();
  }
}

// The invoker works with any Command — new operations plug in without changes.
const invoker = new InvoiceCommandInvoker();
invoker.run(new ApplyDiscountCommand(invoice, 10, invoiceRepo));
invoker.undoLast(); // restores previous discount
```

#### Curated Links

- [Command — refactoring.guru](https://refactoring.guru/design-patterns/command) — Full pattern description with UML and examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all behavioral patterns
- [GoF Patterns — sourcemaking.com](https://sourcemaking.com/design_patterns) — Classic GoF descriptions with intent and applicability notes

---

### Iterator

**Category:** Behavioral
**One-liner:** Provide a way to access elements of a collection sequentially without exposing its underlying data structure.

#### What It Is

The Iterator pattern extracts traversal behaviour from a collection into a separate *iterator* object. The iterator tracks the current position and knows how to advance to the next element. The collection exposes a factory method that creates an appropriate iterator. Client code interacts only with the iterator interface — it neither knows nor cares whether the collection is backed by an array, a linked list, a database result set, or a graph.

The pattern becomes powerful when traversal is non-trivial. A balanced binary search tree (BST) can expose both an in-order iterator and a breadth-first iterator without the collection needing to know which client wants which. Multiple independent iterators can traverse the same collection simultaneously without interfering, because each iterator maintains its own position state.

TypeScript and modern JavaScript have this pattern built into the language via the iterable protocol (`Symbol.iterator`) and generator functions (`function*`). Any object that implements `[Symbol.iterator](): Iterator<T>` works with `for...of`, the spread operator, and destructuring. Implementing this interface makes custom domain collections (a paginated database cursor, an event log window) first-class citizens of the language's iteration ecosystem.

In backend systems, custom iterators are most valuable for collections that cannot be fully materialised in memory: streaming database result sets, paginated API (Application Programming Interface) responses, file-line readers, or sliding-window event processors.

#### Why It Matters

- Hides internal data structures from clients — the underlying representation can change without affecting traversal code.
- Supports multiple simultaneous independent traversals of the same collection.
- Integrates with language-level iteration constructs (`for...of`, spread, destructuring) via the iterable protocol.
- Enables lazy, memory-efficient traversal of large or infinite sequences (database cursors, event streams).

#### TypeScript Example

```typescript
// BAD — OrderBatch exposes its internal array directly, forcing all clients
// to know the underlying structure and preventing encapsulated traversal logic.
class OrderBatch {
  readonly orders: Order[] = []; // public internal state — hard to change later

  add(order: Order): void {
    this.orders.push(order);
  }
}

// Client must know about the internal array:
const batch = new OrderBatch();
for (let i = 0; i < batch.orders.length; i++) {
  processOrder(batch.orders[i]);
}

// GOOD — OrderBatch implements the iterable protocol; clients use for...of.
class OrderBatch implements Iterable<Order> {
  private orders: Order[] = [];

  add(order: Order): void {
    this.orders.push(order);
  }

  [Symbol.iterator](): Iterator<Order> {
    let index = 0;
    const orders = this.orders;
    return {
      next(): IteratorResult<Order> {
        if (index < orders.length) {
          return { value: orders[index++], done: false };
        }
        return { value: undefined as unknown as Order, done: true };
      },
    };
  }
}

// Or more concisely with a generator:
class OrderBatchGen implements Iterable<Order> {
  private orders: Order[] = [];

  add(order: Order): void { this.orders.push(order); }

  *[Symbol.iterator](): Generator<Order> {
    for (const order of this.orders) {
      yield order;
    }
  }
}

// Client code is identical regardless of the internal structure:
const batch = new OrderBatchGen();
for (const order of batch) {
  processOrder(order);
}
```

#### Curated Links

- [Iterator — refactoring.guru](https://refactoring.guru/design-patterns/iterator) — Full pattern description with UML and examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all behavioral patterns
- [GoF Patterns — dofactory.com](https://www.dofactory.com/net/design-patterns) — Classic GoF catalogue with structural diagrams

---

### Template Method

**Category:** Behavioral
**One-liner:** Define the skeleton of an algorithm in a base class, deferring some steps to subclasses without changing the algorithm's overall structure.

#### What It Is

The Template Method pattern codifies an algorithm's invariant structure in an abstract base class as a *template method* — typically a `final` or non-overridable method that calls a sequence of steps. Some steps have default implementations in the base class; others are declared abstract and *must* be overridden by subclasses; optional *hook* methods are empty by default and may be overridden to inject behaviour at specific points.

The pattern formalises the Hollywood Principle: "don't call us, we'll call you." The base class calls the subclass-provided steps at predetermined points in the algorithm — subclasses never call the template method themselves. This inversion ensures the overall algorithm flow remains under the base class's control regardless of which subclass is used.

Template Method is the inheritance-based complement to the Strategy pattern. When you want a fixed algorithm skeleton with configurable steps, and you are comfortable expressing that via subclassing, Template Method is appropriate. If the steps may need to change at runtime or you want to avoid subclassing, prefer Strategy with composition.

In backend development, Template Method is common in report generators (a fixed pipeline of gather → transform → render, with each step overridden per report type), data importers (validate → parse → persist, with different parsing per source format), and test fixtures (a fixed arrange/act/assert structure with domain-specific setup steps overridden per test case in integration test base classes).

#### Why It Matters

- Eliminates code duplication across multiple classes that share the same algorithmic skeleton but differ in individual steps.
- The invariant part of the algorithm is written and maintained exactly once in the base class.
- Subclasses are constrained to safe extension points — they cannot inadvertently break the algorithm's overall flow.
- Hook methods let subclasses inject optional behaviour without requiring every subclass to implement the same step.

#### TypeScript Example

```typescript
// BAD — Two report generators duplicate the same pipeline (fetch, transform, render header,
// render body) with only the format step differing, leading to divergence over time.
class CsvReportGenerator {
  generate(reportId: string): string {
    const data = this.fetchData(reportId);
    const rows = this.transformData(data);
    const header = 'id,name,amount\n';
    return header + rows.map(r => `${r.id},${r.name},${r.amount}`).join('\n');
  }
  private fetchData(id: string): RawData { /* ... */ return {} as RawData; }
  private transformData(data: RawData): Row[] { /* ... */ return []; }
}

class JsonReportGenerator {
  generate(reportId: string): string {
    const data = this.fetchData(reportId);       // duplicated
    const rows = this.transformData(data);        // duplicated
    return JSON.stringify({ rows }, null, 2);
  }
  private fetchData(id: string): RawData { /* ... */ return {} as RawData; }
  private transformData(data: RawData): Row[] { /* ... */ return []; }
}

// GOOD — The shared pipeline lives in the abstract base class; only the rendering step varies.
abstract class ReportGenerator {
  // Template method — the invariant algorithm skeleton.
  generate(reportId: string): string {
    const data = this.fetchData(reportId);
    const rows = this.transformData(data);
    this.onBeforeRender(rows); // optional hook
    return this.renderRows(rows);
  }

  // Invariant steps — defined once, shared by all subclasses.
  private fetchData(reportId: string): RawData {
    // fetch from DB or API
    return {} as RawData;
  }

  private transformData(data: RawData): Row[] {
    // shared normalisation logic
    return [];
  }

  // Hook — subclasses may override; default is no-op.
  protected onBeforeRender(_rows: Row[]): void {}

  // Abstract step — every subclass must provide its own implementation.
  protected abstract renderRows(rows: Row[]): string;
}

class CsvReportGenerator extends ReportGenerator {
  protected renderRows(rows: Row[]): string {
    const header = 'id,name,amount\n';
    return header + rows.map(r => `${r.id},${r.name},${r.amount}`).join('\n');
  }
}

class JsonReportGenerator extends ReportGenerator {
  protected renderRows(rows: Row[]): string {
    return JSON.stringify({ rows }, null, 2);
  }
}
```

#### Curated Links

- [Template Method — refactoring.guru](https://refactoring.guru/design-patterns/template-method) — Full pattern description with UML and examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all behavioral patterns
- [GoF Patterns — sourcemaking.com](https://sourcemaking.com/design_patterns) — Classic GoF descriptions with intent and applicability notes

---

### Chain of Responsibility

**Category:** Behavioral
**One-liner:** Pass a request along a chain of handlers, giving each a chance to process it or forward it to the next handler.

#### What It Is

The Chain of Responsibility pattern assembles a set of *handlers* into a linked chain. Each handler holds a reference to the next handler in the chain. When a request arrives, the handler either processes it (and optionally stops propagation) or forwards it to the next handler. Handlers share a common interface, so the chain can be assembled and reordered at runtime without modifying individual handlers.

The pattern decouples the *sender* of a request from its *receiver*. The sender passes the request to the first handler and has no knowledge of which handler will ultimately process it — or whether any handler will. This is different from the Decorator pattern (which always passes to the next decorator and transforms the result) because Chain of Responsibility handlers may short-circuit the chain entirely.

In backend development, the pattern is the conceptual backbone of middleware pipelines (Express/Koa/NestJS middleware, where each middleware either handles the request or calls `next()`), validation chains (a series of validation rules applied in order until one fails or all pass), and authorization pipelines (role check → permission check → rate-limit check, each capable of rejecting early).

A practical concern: chains assembled from arbitrary handlers can be hard to debug because responsibility is implicit. Mitigate this by keeping handlers small and single-purpose, and logging at each step in development. The pattern trades explicit branching for implicit delegation — choose it when the tradeoff is worth it.

#### Why It Matters

- Decouples senders from receivers — the sender needs no knowledge of the concrete handler that will process its request.
- Handlers can be assembled, reordered, and removed at runtime without touching either the sender or any individual handler.
- Each handler has a single responsibility; adding new validation, authorization, or transformation steps requires no modification to existing handlers.
- Mirrors idiomatic middleware patterns already familiar to Node.js/Express developers.

#### TypeScript Example

```typescript
// BAD — A monolithic validation method grows with every new rule, coupling all
// validation logic into a single place and making individual rules hard to test.
class OrderValidator {
  validate(order: Order): ValidationResult {
    if (!order.customerId) {
      return { valid: false, reason: 'Missing customer ID' };
    }
    if (order.items.length === 0) {
      return { valid: false, reason: 'Order has no items' };
    }
    if (order.total <= 0) {
      return { valid: false, reason: 'Order total must be positive' };
    }
    if (!this.customerIsActive(order.customerId)) {
      return { valid: false, reason: 'Customer account is inactive' };
    }
    return { valid: true };
  }
  private customerIsActive(id: string): boolean { return true; }
}

// GOOD — Each validation rule is a standalone handler in a chain.
interface ValidationResult {
  valid: boolean;
  reason?: string;
}

abstract class OrderValidationHandler {
  private next: OrderValidationHandler | null = null;

  setNext(handler: OrderValidationHandler): OrderValidationHandler {
    this.next = handler;
    return handler; // allows fluent chaining
  }

  handle(order: Order): ValidationResult {
    if (this.next) {
      return this.next.handle(order);
    }
    return { valid: true };
  }
}

class CustomerIdHandler extends OrderValidationHandler {
  handle(order: Order): ValidationResult {
    if (!order.customerId) {
      return { valid: false, reason: 'Missing customer ID' };
    }
    return super.handle(order);
  }
}

class NonEmptyItemsHandler extends OrderValidationHandler {
  handle(order: Order): ValidationResult {
    if (order.items.length === 0) {
      return { valid: false, reason: 'Order has no items' };
    }
    return super.handle(order);
  }
}

class PositiveTotalHandler extends OrderValidationHandler {
  handle(order: Order): ValidationResult {
    if (order.total <= 0) {
      return { valid: false, reason: 'Order total must be positive' };
    }
    return super.handle(order);
  }
}

// Assemble the chain at composition time — order and membership are runtime decisions.
const customerIdHandler = new CustomerIdHandler();
customerIdHandler
  .setNext(new NonEmptyItemsHandler())
  .setNext(new PositiveTotalHandler());

const result = customerIdHandler.handle(order);
```

#### Curated Links

- [Chain of Responsibility — refactoring.guru](https://refactoring.guru/design-patterns/chain-of-responsibility) — Full pattern description with UML and examples
- [Behavioral Patterns overview — refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns) — Context for all behavioral patterns
- [GoF Patterns — dofactory.com](https://www.dofactory.com/net/design-patterns) — Classic GoF catalogue with structural diagrams

---
