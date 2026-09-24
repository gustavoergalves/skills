# Anti-Patterns & Code Smells

Anti-patterns are recurring solutions that seem reasonable but cause more problems than they solve. Code smells are surface-level symptoms that suggest a deeper structural problem. Both categories are worth studying not just to recognise them — but to understand which SOLID (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) principle or Gang of Four (GoF) pattern provides the most direct resolution.

Each entry follows the standard template with an added **Resolves/Violated By** field.

> **AI skill note:** Parse anti-pattern entries by H3 heading. Each entry includes `**Category:**`, `**One-liner:**`, `**What It Is:**`, `**Why It Matters:**`, `**TypeScript Example:**`, `**Resolves/Violated By:**`, and `**Curated Links:**`.

---

### God Object (a.k.a. The Blob)

**Category:** Anti-Pattern
**One-liner:** A class that knows too much or does too much, accumulating unrelated responsibilities across the entire system.

**What It Is:**

A God Object is a class that serves as an over-centralised control point. It accumulates methods, attributes, and dependencies across unrelated domains, growing from rapid prototyping without refactoring cycles or from gradual feature accumulation in legacy systems. A classic sign is a class named after the entire application or a broad subsystem — `ApplicationManager`, `SystemController`, or `DataHandler`.

The class typically runs to hundreds or thousands of lines, exhibits low cohesion (methods share no clear common purpose), and so many other classes depend on it that changing anything triggers ripple effects everywhere. Unit testing becomes practically impossible — isolating a single concern requires pulling the entire object into the test harness.

God Objects tend to accumulate incrementally. Each individual addition feels small and justified. It is only when stepping back that the aggregated violation becomes obvious — the class is simultaneously responsible for data access, business logic, validation, logging, and notification.

**Why It Matters:**

- Changes to one feature break seemingly unrelated features
- Onboarding is slow because understanding one thing requires understanding everything
- The class cannot be reused in isolation
- High cyclomatic complexity makes testing each responsibility in isolation infeasible

**TypeScript Example:**

```typescript
// BAD — God Object: UserService handles auth, profile, email, payments, and logging
class UserService {
  async login(email: string, password: string): Promise<string> {
    const user = await db.query('SELECT * FROM users WHERE email = ?', [email]);
    if (!user || user.password !== hash(password)) throw new Error('Invalid credentials');
    const token = jwt.sign({ id: user.id }, process.env.SECRET!);
    await db.query('INSERT INTO audit_log VALUES (?, ?, NOW())', [user.id, 'login']);
    await mailer.send(user.email, 'New login detected', '...');
    await stripe.customers.retrieve(user.stripeCustomerId);
    return token;
  }

  async updateProfile(userId: string, data: any) { /* 80 lines */ }
  async processPayment(userId: string, amount: number) { /* 60 lines */ }
  async sendWelcomeEmail(email: string) { /* 40 lines */ }
  async generateReport(userId: string) { /* 100 lines */ }
}

// GOOD — Decomposed into single-responsibility services
class AuthService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly tokenService: TokenService,
    private readonly auditLog: AuditLogService,
    private readonly notifier: LoginNotifier,
  ) {}

  async login(email: string, password: string): Promise<string> {
    const user = await this.userRepo.findByEmail(email);
    if (!user?.passwordMatches(password)) throw new UnauthorisedError();
    const token = this.tokenService.issue(user.id);
    await this.auditLog.record(user.id, 'login');
    await this.notifier.sendLoginAlert(user.email);
    return token;
  }
}

// ProfileService, PaymentService, ReportService each live in their own file
```

**Resolves/Violated By:** Violated by ignoring Single Responsibility Principle (SRP) — the class has more than one reason to change. Resolved by applying SRP: decompose into focused, single-purpose classes. GoF patterns that assist: **Strategy** (separate algorithms), **Observer** (decouple notifications), **Factory Method** (delegate creation), **Facade** (provide a simplified interface over the decomposed classes), **Mediator** (manage cross-service communication).

**Curated Links:**
- [The God Object Anti-Pattern — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/the-god-object/) — Comprehensive coverage of symptoms, causes, and SOLID/GoF resolution
- [Anti-Patterns Overview — quokkalabs.com](https://quokkalabs.com/blog/anti-patterns-you-should-know-while-coding/) — God Class, Spaghetti Code, Lava Flow, Golden Hammer with SOLID references

---

### Spaghetti Code

**Category:** Anti-Pattern
**One-liner:** Code with tangled, convoluted control flow that lacks structure, modularity, and separation of concerns.

**What It Is:**

Originally associated with excessive `goto` statements in procedural code, Spaghetti Code in object-oriented programming (OOP) manifests as methods that invoke large chains of other methods scattered across the codebase with no clear layering, business logic mixed with persistence or transport concerns, and no architectural intent.

Developers typically begin work without an architectural plan and apply iterative patches without refactoring. Each patch adds coupling. Eventually, removing any single method risks breaking large portions of the application.

New developers cannot trace program flow without extensive time investment. Because the logic is entangled, the same fix must be applied in multiple places — and it is easy to miss one.

**Why It Matters:**

- Impossible to isolate units for testing
- A single bug fix introduces regressions elsewhere
- Code cannot be reused because methods are tightly coupled to specific contexts
- Onboarding time increases exponentially with codebase age

**TypeScript Example:**

```typescript
// BAD — business logic, persistence, email, and payment entangled in one handler
async function handleOrderSubmit(req: Request, res: Response) {
  const user = await db.query('SELECT * FROM users WHERE id = ?', [req.body.userId]);
  if (!user) { res.status(404).send('User not found'); return; }
  const items = await db.query('SELECT * FROM cart WHERE user_id = ?', [req.body.userId]);
  let total = 0;
  for (const item of items) {
    const product = await db.query('SELECT * FROM products WHERE id = ?', [item.productId]);
    total += product.price * item.quantity;
    if (product.stock < item.quantity) { res.status(400).send('Out of stock'); return; }
    await db.query('UPDATE products SET stock = stock - ? WHERE id = ?', [item.quantity, product.id]);
  }
  const charge = await stripe.charges.create({ amount: total, currency: 'usd', source: req.body.token });
  await db.query('INSERT INTO orders VALUES (?, ?, ?, NOW())', [req.body.userId, total, charge.id]);
  await mailer.send(user.email, 'Order Confirmed', `Your order of $${total / 100} was placed.`);
  await db.query('DELETE FROM cart WHERE user_id = ?', [req.body.userId]);
  res.json({ success: true });
}

// GOOD — each layer has a focused role; the handler orchestrates
async function handleOrderSubmit(req: Request, res: Response) {
  const result = await orderService.placeOrder({
    userId: req.body.userId,
    paymentToken: req.body.token,
  });
  res.json({ orderId: result.orderId });
}

class OrderService {
  constructor(
    private readonly inventory: InventoryService,
    private readonly payments: PaymentService,
    private readonly orders: OrderRepository,
    private readonly notifications: OrderNotifier,
    private readonly cart: CartRepository,
  ) {}

  async placeOrder(cmd: PlaceOrderCommand): Promise<OrderResult> {
    const items = await this.cart.findByUser(cmd.userId);
    await this.inventory.reserveAll(items);
    const charge = await this.payments.charge(cmd.paymentToken, items.total());
    const order = await this.orders.save(Order.create(cmd.userId, items, charge.id));
    await this.notifications.confirmOrder(order);
    await this.cart.clearByUser(cmd.userId);
    return { orderId: order.id };
  }
}
```

**Resolves/Violated By:** Violated by ignoring **SRP** (each piece of code has one job) and **OCP** (no stable extension seams exist). Resolved by introducing layers (controller → service → repository) and applying **Command** (encapsulate actions as objects), **Template Method** (define a structured algorithm skeleton), or **Strategy** (cleanly separate varying behaviour from stable structure).

**Curated Links:**
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — Spaghetti Code, God Object, Lava Flow, Golden Hammer with SOLID mappings
- [Anti-Patterns Overview — quokkalabs.com](https://quokkalabs.com/blog/anti-patterns-you-should-know-while-coding/) — Practical resolution strategies

---

### Lava Flow

**Category:** Anti-Pattern
**One-liner:** Dead or vestigial code left over from previous versions, experiments, or prototypes that accumulates and cannot safely be removed.

**What It Is:**

Named after hardened lava — immovable and opaque — Lava Flow is code that appears integral but serves no clear purpose. It originates from heritage systems, rushed initial development, or spike phases that were never formally retired. Nobody dares remove it because there is no documentation explaining whether it is safe to do so.

Commented-out blocks, unused variables, deprecated classes still imported and referenced, and undocumented code paths are all classic symptoms. The team learns to work around the dead code rather than removing it, which trains them to treat all unfamiliar code as potentially active.

Over time the codebase develops a culture of fear: every file touched requires archaeology before modification. The cognitive overhead of understanding what is active versus vestigial multiplies as the system grows.

**Why It Matters:**

- Developers waste time debugging code paths that are never executed
- Dead code inflates cognitive load and onboarding time
- Vestigial tests give false confidence
- Code coverage metrics and static analysis are distorted by unreachable paths

**TypeScript Example:**

```typescript
// BAD — accumulated dead code from previous payment provider experiments
class PaymentProcessor {
  // TODO: remove after Stripe migration (from 2022 — it's 2024 now)
  // async processWithBraintree(amount: number) {
  //   const gateway = new braintree.BraintreeGateway({ ... });
  //   return gateway.transaction.sale({ amount: String(amount / 100) });
  // }

  // Kept for "backward compat" — nothing calls this anymore
  async legacyChargeCard(cardNumber: string, expiry: string, cvv: string, amount: number) {
    // Direct card charge — was used before PCI compliance changes
    return directCardCharge({ cardNumber, expiry, cvv, amount });
  }

  async processWithStripe(token: string, amount: number) {
    return stripe.charges.create({ amount, currency: 'usd', source: token });
  }

  // Experimental multi-currency support — prototype never merged
  private readonly _currencyRates: Record<string, number> = {};
  async convertAndCharge(token: string, amount: number, currency: string) {
    const rate = this._currencyRates[currency] ?? 1;
    return this.processWithStripe(token, Math.round(amount * rate));
  }
}

// GOOD — dead code removed; only live paths remain
class StripePaymentProcessor implements PaymentProcessor {
  async charge(token: string, amountInCents: number): Promise<ChargeResult> {
    const charge = await stripe.charges.create({
      amount: amountInCents,
      currency: 'usd',
      source: token,
    });
    return { chargeId: charge.id, status: charge.status };
  }
}
```

**Resolves/Violated By:** Violated by ignoring **OCP** (code paths were added speculatively rather than through stable extension points) and **Interface Segregation Principle (ISP)** (broadly scoped classes make it hard to see what is actively used). Resolved by: adding test coverage first to establish safety, then using static analysis tools (`ts-prune`, TypeScript's `noUnusedLocals`) to surface unreferenced code, and removing it with version control documentation. **Facade** can provide a clean interface over legacy internals while old code is incrementally retired.

**Curated Links:**
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — Lava Flow covered alongside other classic anti-patterns
- [Anti-Patterns Overview — quokkalabs.com](https://quokkalabs.com/blog/anti-patterns-you-should-know-while-coding/) — Practical resolution strategies with SOLID references

---

### Golden Hammer

**Category:** Anti-Pattern
**One-liner:** Over-reliance on a familiar tool, technology, or pattern applied to every problem regardless of suitability.

**What It Is:**

From the saying "if all you have is a hammer, everything looks like a nail." A team becomes attached to a solution that worked once and defaults to it universally — using a relational database for graph data, a full enterprise framework for a simple script, or a message queue where a direct call would do.

Developer comfort and familiarity, organisational momentum, and a lack of culture around technical evaluation all contribute. The hammer is rarely the wrong tool for every job — it genuinely solves some problems well. The anti-pattern is the failure to evaluate whether it is the right tool for the current problem.

**Why It Matters:**

- Suboptimal solutions: enterprise frameworks for trivial problems, queues for synchronous calls
- Increased complexity and reduced performance where a simpler solution would perform better
- Teams resist evaluating alternatives, closing off better options
- Technical debt accumulates as the chosen tool is stretched beyond its design envelope

**TypeScript Example:**

```typescript
// BAD — Golden Hammer: heavy event-bus used for all communication,
// even synchronous same-process calls that have no need for async decoupling
class UserService {
  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(User.create(dto));
    // Everything goes through the event bus — even synchronous validations
    await this.eventBus.publish('user.created', user);
    const valid = await this.eventBus.request('validate.email', user.email);
    if (!valid) throw new Error('Invalid email');
    const formatted = await this.eventBus.request('format.name', user.name);
    user.name = formatted;
    return user;
  }
}

// GOOD — choose the right tool per communication pattern:
// direct call for same-process synchronous logic;
// event bus only for genuine cross-boundary, async notifications
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailValidator: EmailValidator,   // synchronous, in-process
    private readonly nameFormatter: NameFormatter,     // synchronous, in-process
    private readonly eventBus: EventBus,              // async, cross-boundary
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    this.emailValidator.validate(dto.email);           // direct call — no bus overhead
    const name = this.nameFormatter.format(dto.name); // direct call
    const user = await this.userRepo.save(User.create({ ...dto, name }));
    await this.eventBus.publish('user.created', user); // bus only for external subscribers
    return user;
  }
}
```

**Resolves/Violated By:** Violated by ignoring **Dependency Inversion Principle (DIP)** (coding to the concrete tool rather than an abstraction) and **ISP** (over-broad abstractions force a single tool to serve every consumer). Resolved by introducing abstractions behind interfaces — **Strategy** (swap implementations), **Abstract Factory** (create families of objects without depending on concrete classes), **Bridge** (decouple abstraction from implementation so either can vary independently).

**Curated Links:**
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — Golden Hammer with SOLID mappings
- [Anti-Patterns Overview — quokkalabs.com](https://quokkalabs.com/blog/anti-patterns-you-should-know-while-coding/) — Practical examples and resolution strategies

---

### Shotgun Surgery (a.k.a. Solution Sprawl)

**Category:** Code Smell
**One-liner:** A single logical change requires making many small modifications scattered across multiple unrelated classes.

**What It Is:**

Shotgun Surgery is the structural inverse of a God Object. Where a God Object centralises too much, Shotgun Surgery fragments a single responsibility across too many places. Implementing one feature or fixing one bug requires touching five, ten, or twenty different files, increasing the risk that one location is missed and introducing a subtle bug.

The smell commonly arises from overzealous refactoring that over-distributed a single concern, or from copy-paste patterns that scatter the same logic at different call sites. Duplicated conditional checks replicated across multiple classes are a classic symptom.

**Why It Matters:**

- Easy to miss a change location, causing silent bugs that surface in production
- High learning curve for new developers who cannot predict where changes propagate
- Every cross-cutting concern (logging, validation, security) is duplicated across the codebase
- Code review is tedious when a single story touches dozens of files

**TypeScript Example:**

```typescript
// BAD — tax calculation logic scattered across Order, Invoice, and Cart
class Order {
  getTotal(): number {
    return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0) * 1.2; // 20% VAT
  }
}
class Invoice {
  getAmount(): number {
    return this.lineItems.reduce((sum, l) => sum + l.unitPrice * l.qty, 0) * 1.2; // same
  }
}
class Cart {
  getEstimatedTotal(): number {
    return this.cartItems.reduce((sum, c) => sum + c.cost * c.count, 0) * 1.2; // same
  }
}

// GOOD — tax logic centralised in a single TaxCalculator; consumers depend on the abstraction
interface TaxCalculator {
  applyTax(subtotal: number): number;
}

class VatCalculator implements TaxCalculator {
  constructor(private readonly rate: number = 0.2) {}
  applyTax(subtotal: number): number {
    return subtotal * (1 + this.rate);
  }
}

class Order {
  constructor(private readonly tax: TaxCalculator) {}
  getTotal(): number {
    const subtotal = this.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
    return this.tax.applyTax(subtotal);
  }
}
// Invoice and Cart now also accept TaxCalculator — one change point for VAT rate
```

**Resolves/Violated By:** Violated by ignoring **SRP** (a responsibility should be cohesively owned by one class) and **OCP** (when a change is needed, it should affect one place, not many). Resolved by centralising the fragmented responsibility. **Template Method** centralises an algorithm varying only in sub-steps; **Decorator** adds behaviour in one place; **Visitor** centralises operations over an object structure.

**Curated Links:**
- [Shotgun Surgery — refactoring.guru](https://refactoring.guru/smells/shotgun-surgery) — Root causes, Move Method/Field and Inline Class resolution
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — SOLID mappings for common code smells

---

### Feature Envy

**Category:** Code Smell
**One-liner:** A method that is more interested in the data and methods of another class than its own.

**What It Is:**

The classic sign is a method that repeatedly accesses fields or calls methods on a foreign object rather than its own state. Data and the functions that use that data should live together — Feature Envy violates encapsulation by allowing the processing logic to reside in a different class from the data it processes.

The smell commonly results from refactoring that moved data into a new class but left the corresponding operations behind. An `Order` class that repeatedly accesses `ShoppingItem.price` and `ShoppingItem.taxRate` to compute a total is demonstrating Feature Envy — the `taxedPrice()` method belongs on `ShoppingItem`, not on `Order`.

**Why It Matters:**

- The method cannot be tested in isolation without instantiating or mocking the foreign class
- Business logic is separated from the data it acts on, making it easy to violate invariants
- Code reuse is inhibited because the method drags along a dependency on the foreign class everywhere it travels
- Getters proliferate on the foreign class to satisfy the envious caller

**TypeScript Example:**

```typescript
// BAD — InvoiceService.calculateDiscount is envious of Customer's internal state
class Customer {
  readonly tier: 'standard' | 'premium' | 'enterprise';
  readonly yearsActive: number;
  readonly totalSpend: number;
}

class InvoiceService {
  calculateDiscount(customer: Customer, subtotal: number): number {
    // envious: accesses customer's data to make a decision that belongs on Customer
    if (customer.tier === 'enterprise' && customer.yearsActive > 5) {
      return subtotal * 0.20;
    } else if (customer.tier === 'premium' && customer.totalSpend > 10_000) {
      return subtotal * 0.10;
    } else if (customer.tier === 'standard') {
      return subtotal * 0.05;
    }
    return 0;
  }
}

// GOOD — discount logic belongs on Customer, which knows its own state
class Customer {
  constructor(
    readonly tier: 'standard' | 'premium' | 'enterprise',
    readonly yearsActive: number,
    readonly totalSpend: number,
  ) {}

  discountRate(): number {
    if (this.tier === 'enterprise' && this.yearsActive > 5) return 0.20;
    if (this.tier === 'premium' && this.totalSpend > 10_000) return 0.10;
    return 0.05;
  }
}

class InvoiceService {
  calculateDiscount(customer: Customer, subtotal: number): number {
    return subtotal * customer.discountRate(); // no envy — asks the right object
  }
}
```

**Resolves/Violated By:** Violated by ignoring **SRP** (methods that primarily operate on another class's data belong in that class) and the general encapsulation principle (data and behaviour should be co-located). Note: **Strategy** and **Visitor** are intentional exceptions — they deliberately separate behaviour from data for extensibility purposes. In non-pattern contexts, the Move Method refactoring directly resolves the smell.

**Curated Links:**
- [Feature Envy — refactoring.guru](https://refactoring.guru/smells/feature-envy) — Description, refactoring techniques (Move Method, Extract Method), connection to SRP and encapsulation
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — SOLID mappings for common code smells

---

### Primitive Obsession

**Category:** Code Smell
**One-liner:** Overuse of language primitive types (strings, integers, booleans) to represent domain concepts that deserve their own type abstractions.

**What It Is:**

Instead of a `PhoneNumber` class, code passes raw strings everywhere. Instead of a `Money` class, it passes `number` values with implicit currency assumptions. Named constants like `USER_ADMIN_ROLE = 1` simulate types instead of using proper type abstractions. Validation and formatting logic for a concept is then duplicated across every usage site — every caller must independently validate the phone format or ensure the money value is non-negative.

Primitive Obsession is the code smell most directly addressed by the **Value Object** pattern from Domain-Driven Design (DDD). The extracted class is immutable, self-validating, and carries its own behaviour (formatting, comparison, arithmetic). Once a `Money` class exists, currency-unit errors become impossible — the type system enforces correctness.

**Why It Matters:**

- Validation logic duplicated at every call site — one missed site creates a security or data-integrity bug
- Hidden intent: a raw `string` does not communicate that it must be a valid email address
- Related fields passed together constantly signal they want to be a class (see Data Clumps)
- Refactoring is costly once primitives are embedded deeply in the codebase

**TypeScript Example:**

```typescript
// BAD — phone number passed as raw string; validation duplicated everywhere
class UserRegistrationService {
  register(name: string, email: string, phone: string, roleCode: number): void {
    if (!/^\+?[1-9]\d{1,14}$/.test(phone)) throw new Error('Invalid phone'); // duplicated
    if (roleCode !== 1 && roleCode !== 2 && roleCode !== 3) throw new Error('Invalid role');
    // ...
  }
}
class NotificationService {
  sendSms(phone: string, message: string): void {
    if (!/^\+?[1-9]\d{1,14}$/.test(phone)) throw new Error('Invalid phone'); // duplicated again
    smsGateway.send(phone, message);
  }
}

// GOOD — domain concepts wrapped in dedicated value objects
class PhoneNumber {
  private readonly value: string;
  private constructor(raw: string) { this.value = raw; }
  static parse(raw: string): PhoneNumber {
    if (!/^\+?[1-9]\d{1,14}$/.test(raw)) throw new InvalidPhoneNumberError(raw);
    return new PhoneNumber(raw);
  }
  toString(): string { return this.value; }
}

type UserRole = 'admin' | 'editor' | 'viewer'; // type alias replaces magic integer

class UserRegistrationService {
  register(name: string, email: string, phone: PhoneNumber, role: UserRole): void {
    // PhoneNumber is guaranteed valid; UserRole is exhaustively typed — no defensive checks needed
    userRepo.save(User.create({ name, email, phone, role }));
  }
}
```

**Resolves/Violated By:** Violated by ignoring **SRP** (the class that holds a primitive is forced to own its validation) and the encapsulation principle (values have no boundaries without a type). Resolved by extracting Value Objects. **State** replaces integer type codes with a State object hierarchy; **Strategy** replaces behavioural type codes with Strategy classes; **Decorator** adds capabilities to simple types without modifying existing code.

**Curated Links:**
- [Primitive Obsession — refactoring.guru](https://refactoring.guru/smells/primitive-obsession) — Causes, symptoms, refactoring via Replace Data Value with Object, State/Strategy patterns
- [Understanding Anti-Patterns — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/understanding-anti-patterns/) — SOLID mappings for common code smells

---

### Data Clumps

**Category:** Code Smell
**One-liner:** Groups of variables that always appear together, suggesting they belong in a dedicated class.

**What It Is:**

A canonical example: three or four database connection parameters (`host`, `port`, `username`, `password`) that are passed as individual arguments across multiple methods. If removing one variable from the group makes the remaining ones meaningless, it is a data clump. The same field cluster typically appears as instance variables in several classes and as repeated parameter lists in multiple method signatures.

Data Clumps are closely related to Primitive Obsession — the clumping fields are often primitives that together represent a cohesive concept. The resolution is the same: extract a dedicated class. The extracted class is usually a **Value Object** from Domain-Driven Design (DDD) — immutable, self-validating, and carrying its own behaviour (connection string formatting, date range overlap checking).

**Why It Matters:**

- Adding a new field to the concept requires updating every call site independently
- The group's collective invariants (a date range's start must be before end) cannot be enforced without a dedicated class
- Method signatures grow unwieldy and become easy to misorder
- No single class owns the group — knowledge about the concept is scattered

**TypeScript Example:**

```typescript
// BAD — database connection params scattered across multiple signatures
class DatabaseMigrator {
  run(host: string, port: number, user: string, password: string, dbName: string): void { /* ... */ }
}
class ConnectionPool {
  connect(host: string, port: number, user: string, password: string, dbName: string): Pool { /* ... */ }
}
class HealthChecker {
  ping(host: string, port: number, user: string, password: string): boolean { /* ... */ }
}

// GOOD — extracted DatabaseCredentials value object
class DatabaseCredentials {
  readonly host: string;
  readonly port: number;
  readonly user: string;
  readonly password: string;
  readonly dbName: string;

  constructor(params: {
    host: string; port: number; user: string; password: string; dbName: string;
  }) {
    if (params.port < 1 || params.port > 65535) throw new RangeError('Invalid port');
    Object.assign(this, params);
  }

  toConnectionString(): string {
    return `postgresql://${this.user}:${this.password}@${this.host}:${this.port}/${this.dbName}`;
  }
}

class DatabaseMigrator {
  run(creds: DatabaseCredentials): void { /* ... */ }
}
class ConnectionPool {
  connect(creds: DatabaseCredentials): Pool { /* ... */ }
}
class HealthChecker {
  ping(creds: DatabaseCredentials): boolean { /* ... */ }
}
```

**Resolves/Violated By:** Violated by ignoring **SRP** (the clumping data forms a cohesive concept that deserves its own class) and **Don't Repeat Yourself (DRY)** (the group should be described once). Resolved by extracting a **Value Object** (immutable, self-validating domain class) and — where needed — a **Factory Method** to create canonical instances. Caution: passing whole objects rather than primitives can introduce unwanted coupling if the extracted class is too broad — keep extracted classes tightly focused.

**Curated Links:**
- [Data Clumps — refactoring.guru](https://refactoring.guru/smells/data-clumps) — Causes, Extract Class and Introduce Parameter Object refactoring
- [The God Object Anti-Pattern — softwarepatternslexicon.com](https://softwarepatternslexicon.com/mastering-design-patterns/anti-patterns-and-code-smells/the-god-object/) — Broader anti-pattern context with SOLID resolution mapping

---
