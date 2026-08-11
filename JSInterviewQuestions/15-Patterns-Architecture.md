# 15 — Patterns & Architecture
### 15 Questions | Advanced

---

**Q1. What is the Module Pattern and how does it compare to ES6 modules?** — Medium

**Answer:**
The Module Pattern uses IIFEs (Immediately Invoked Function Expressions) and closures to simulate private state and public APIs, predating ES6 modules.

```js
// Classic Module Pattern (IIFE):
const UserService = (function () {
  // Private state — not accessible outside:
  let users = [];
  let nextId = 1;

  // Private helper:
  function validate(user) {
    return user.name && user.email;
  }

  // Public API — exposed via return:
  return {
    add(user) {
      if (!validate(user)) throw new Error("Invalid user");
      const newUser = { id: nextId++, ...user };
      users.push(newUser);
      return newUser;
    },
    findById(id) {
      return users.find(u => u.id === id);
    },
    count() {
      return users.length;
    }
  };
})();

UserService.add({ name: "Alice", email: "a@example.com" });
UserService.users; // undefined — private!

// Revealing Module Pattern — cleaner syntax:
const Calculator = (function () {
  let history = [];

  function add(a, b) { const r = a + b; history.push(r); return r; }
  function subtract(a, b) { const r = a - b; history.push(r); return r; }
  function getHistory() { return [...history]; }

  return { add, subtract, getHistory }; // reveal what's public
})();

// ES6 Module (file: userService.mjs):
let users = [];
let nextId = 1;

function validate(user) { return user.name && user.email; }

export function add(user) {
  if (!validate(user)) throw new Error("Invalid user");
  const newUser = { id: nextId++, ...user };
  users.push(newUser);
  return newUser;
}

export function findById(id) { return users.find(u => u.id === id); }
export function count() { return users.length; }
// validate is NOT exported — module-scoped private
```

**Comparison:**
- Both achieve encapsulation and public/private APIs
- ES6 modules are statically analyzable (tree-shaking, IDE tooling)
- ES6 modules have file-level scope — no IIFE needed
- ES6 modules have live bindings; module pattern uses copies
- Module pattern can be used anywhere (including `<script>` without type=module)

**GOTCHA:** ES6 module state is per-module, not per-import. All importers share the same module-level state. This is both a feature (shared singleton) and a potential issue for testing (you need to reset module state between tests, or use dependency injection).

---

**Q2. What is the Observer / Pub-Sub pattern and how is it implemented in JavaScript?** — Medium

**Answer:**
The Observer pattern defines a one-to-many dependency — when one object (subject) changes state, all dependents (observers) are notified automatically.

```js
// EventEmitter — Node.js built-in (Observer pattern):
const { EventEmitter } = require("events");
const emitter = new EventEmitter();

emitter.on("data", payload => console.log("Observer 1:", payload));
emitter.on("data", payload => console.log("Observer 2:", payload));

emitter.emit("data", { value: 42 });
// Observer 1: { value: 42 }
// Observer 2: { value: 42 }

// Custom Pub-Sub (decoupled — publisher doesn't know subscribers):
class EventBus {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(handler);
    // Return unsubscribe function:
    return () => this.off(event, handler);
  }

  off(event, handler) {
    this.#listeners.get(event)?.delete(handler);
  }

  once(event, handler) {
    const wrapper = (data) => {
      handler(data);
      this.off(event, wrapper);
    };
    return this.on(event, wrapper);
  }

  emit(event, data) {
    this.#listeners.get(event)?.forEach(handler => {
      try { handler(data); }
      catch (e) { console.error(`Error in handler for ${event}:`, e); }
    });
  }
}

const bus = new EventBus();

// Subscriber:
const unsubscribe = bus.on("user:created", user => {
  console.log("Send welcome email to:", user.email);
});

// Publisher:
bus.emit("user:created", { id: 1, email: "alice@example.com" });

// Later — cleanup:
unsubscribe();

// Observer vs Pub-Sub distinction:
// Observer: subjects know their observers directly (tight coupling)
// Pub-Sub: a broker/message bus sits between publishers and subscribers (loose coupling)
// JavaScript's EventEmitter is technically Pub-Sub (the emitter is the broker)
```

**GOTCHA:** Memory leaks are the most common issue with the Observer pattern. If you attach listeners but never remove them (when components unmount, or objects are destroyed), the listener keeps the subject alive and the callback may run on dead state. Always return and call unsubscribe functions, or use `once()` for one-time events.

---

**Q3. What is the Singleton pattern and what are its trade-offs in JavaScript?** — Medium

**Answer:**
The Singleton ensures a class has only one instance and provides a global access point.

```js
// Classic Singleton:
class Database {
  static #instance = null;
  #connection;

  constructor() {
    if (Database.#instance) {
      return Database.#instance;
    }
    this.#connection = this.#connect();
    Database.#instance = this;
  }

  #connect() {
    console.log("Connecting to database...");
    return { /* connection object */ };
  }

  query(sql) { /* ... */ }

  static getInstance() {
    if (!Database.#instance) {
      new Database();
    }
    return Database.#instance;
  }
}

const db1 = Database.getInstance();
const db2 = Database.getInstance();
db1 === db2; // true — same instance

// Module-level singleton — simplest in ESM:
// database.mjs:
let pool = null;
export function getPool() {
  if (!pool) pool = createConnectionPool();
  return pool;
}
// Every importer calls getPool() and gets the same pool

// Dependency injection alternative (testable):
class UserService {
  constructor(db) {
    this.db = db; // inject dependency
  }
  getUser(id) { return this.db.query(`SELECT * FROM users WHERE id = ${id}`); }
}

// Production:
const userService = new UserService(Database.getInstance());
// Testing — inject a mock:
const mockDb = { query: jest.fn().mockResolvedValue({ id: 1, name: "Alice" }) };
const testUserService = new UserService(mockDb);
```

**Trade-offs:**

Pros:
- Ensures one instance (DB pool, logger, config)
- Lazy initialization — created on first use
- Global access point

Cons:
- Global state — hard to reason about in large codebases
- Tight coupling — classes access the singleton directly
- Testing is hard — singletons carry state between tests
- Breaks in distributed systems — each process has its own singleton

**GOTCHA:** In Node.js, modules are singletons — each module is loaded once and cached. An ES module's top-level state IS the singleton. You don't need a class-based singleton pattern when you can just use module-level state. In testing, reset module state by clearing `require.cache` or using jest's `jest.resetModules()`.

---

**Q4. What is the Factory pattern and when would you prefer it over a constructor?** — Medium

**Answer:**
A Factory function or method encapsulates object creation, hiding instantiation logic and returning different types based on input.

```js
// Simple factory function:
function createShape(type, options) {
  switch (type) {
    case "circle":
      return {
        type: "circle",
        radius: options.radius,
        area() { return Math.PI * this.radius ** 2; },
        perimeter() { return 2 * Math.PI * this.radius; }
      };
    case "rectangle":
      return {
        type: "rectangle",
        width: options.width,
        height: options.height,
        area() { return this.width * this.height; },
        perimeter() { return 2 * (this.width + this.height); }
      };
    default:
      throw new RangeError(`Unknown shape type: ${type}`);
  }
}

const circle = createShape("circle", { radius: 5 });
circle.area(); // 78.54...

// Factory that returns different classes:
class Circle { /* ... */ }
class Rectangle { /* ... */ }

function ShapeFactory(type, options) {
  if (type === "circle") return new Circle(options);
  if (type === "rectangle") return new Rectangle(options);
  throw new RangeError(`Unknown type: ${type}`);
}

// Abstract Factory — create families of related objects:
function createTheme(name) {
  const themes = {
    dark: {
      createButton: (label) => DarkButton(label),
      createModal: (content) => DarkModal(content),
      createInput: (placeholder) => DarkInput(placeholder)
    },
    light: {
      createButton: (label) => LightButton(label),
      createModal: (content) => LightModal(content),
      createInput: (placeholder) => LightInput(placeholder)
    }
  };
  return themes[name] ?? themes.light;
}

// When to prefer factory over constructor:
// 1. Async initialization:
class Session {
  static async create(userId) {  // factory — constructors can't be async
    const session = new Session();
    session.user = await db.findUser(userId);
    session.permissions = await db.getPermissions(userId);
    return session;
  }
}
const session = await Session.create(42);

// 2. Return type varies based on input
// 3. Complex initialization logic you want to hide
// 4. Subtype selection at runtime (plugin systems)
```

**GOTCHA:** Factories create NEW objects on every call unless you implement caching. If you need a singleton-like factory, add a cache: `const cache = new Map(); if (cache.has(key)) return cache.get(key);`. Also, factory functions don't work with `instanceof` checks unless you specifically set the prototype — `createShape("circle") instanceof Circle` is `false` for plain object factories.

---

**Q5. What is the Strategy pattern and how does it enable swappable algorithms?** — Medium

**Answer:**
The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. The algorithm varies independently from the client that uses it.

```js
// Sorting strategy:
class DataSorter {
  constructor(strategy) {
    this.strategy = strategy;
  }

  sort(data) {
    return this.strategy(data);
  }
}

// Concrete strategies:
const quickSort = data => [...data].sort((a, b) => a - b);   // simplified
const bubbleSort = data => {                                   // O(n²) for teaching
  const arr = [...data];
  for (let i = 0; i < arr.length; i++)
    for (let j = 0; j < arr.length - i - 1; j++)
      if (arr[j] > arr[j + 1]) [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
  return arr;
};

const sorter = new DataSorter(quickSort);
sorter.sort([3, 1, 4, 1, 5]); // [1, 1, 3, 4, 5]
sorter.strategy = bubbleSort; // swap at runtime
sorter.sort([3, 1, 4, 1, 5]); // [1, 1, 3, 4, 5] — same result, different algorithm

// Real-world: payment processing strategies:
const PaymentStrategies = {
  stripe: {
    process(amount, card) { /* Stripe API call */ },
    refund(chargeId) { /* Stripe refund */ }
  },
  paypal: {
    process(amount, email) { /* PayPal API call */ },
    refund(transactionId) { /* PayPal refund */ }
  },
  crypto: {
    process(amount, wallet) { /* Crypto transfer */ },
    refund() { throw new Error("Crypto payments are non-refundable") }
  }
};

class Checkout {
  #paymentStrategy;

  setPaymentMethod(method) {
    this.#paymentStrategy = PaymentStrategies[method];
    if (!this.#paymentStrategy) throw new Error(`Unknown payment method: ${method}`);
  }

  pay(amount, details) {
    return this.#paymentStrategy.process(amount, details);
  }
}

// Functional approach — strategies as plain functions:
function compress(data, algorithm) {
  return algorithm(data);
}

const gzip = data => zlib.gzipSync(data);
const brotli = data => zlib.brotliCompressSync(data);

compress(fileData, gzip);
compress(fileData, brotli);
```

**GOTCHA:** The Strategy pattern adds indirection — don't use it for simple if/else logic. It's worth the complexity when: algorithms are likely to change or grow, multiple algorithms are used at runtime, or you want to test algorithms independently. In JavaScript, passing functions directly (first-class functions) is often simpler than creating a Strategy class hierarchy.

---

**Q6. What is the Decorator pattern and how does TypeScript decorators relate to it?** — Hard

**Answer:**
The Decorator pattern attaches additional responsibilities to an object dynamically, providing a flexible alternative to subclassing.

```js
// Functional decorator:
function withLogging(fn) {
  return function (...args) {
    console.log(`Calling ${fn.name} with:`, args);
    const result = fn.apply(this, args);
    console.log(`${fn.name} returned:`, result);
    return result;
  };
}

function add(a, b) { return a + b; }
const loggedAdd = withLogging(add);
loggedAdd(2, 3); // logs: "Calling add with: [2, 3]", "add returned: 5"

// Composing multiple decorators:
const withTiming = fn => function (...args) {
  const start = performance.now();
  const result = fn.apply(this, args);
  console.log(`${fn.name} took ${performance.now() - start}ms`);
  return result;
};

const withMemoize = fn => {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
};

// Compose decorators:
const superAdd = withLogging(withTiming(withMemoize(add)));

// Class method decorator (TC39 Stage 3 / TypeScript):
class UserService {
  @log        // decorator
  @validate   // stacked decorators — applied bottom-up
  async createUser(data) {
    return db.insert("users", data);
  }
}

// Implementing a method decorator:
function log(target, context) {
  if (context.kind !== "method") return;
  return function (...args) {
    console.log(`[${context.name}] called with`, args);
    const result = target.apply(this, args);
    console.log(`[${context.name}] returned`, result);
    return result;
  };
}

// Component decorator pattern (React Higher-Order Components):
function withAuthentication(Component) {
  return function AuthenticatedComponent(props) {
    const { user } = useAuth();
    if (!user) return <Redirect to="/login" />;
    return <Component {...props} user={user} />;
  };
}

const ProtectedProfile = withAuthentication(UserProfile);
```

**Follow-up:** What is the difference between decorators and mixins?

Decorators add behavior around an existing object/method (wrapping pattern). Mixins blend behaviors into a class at definition time. Decorators are applied at use-time and can be composed; mixins are applied at class creation time and affect all instances.

**GOTCHA:** JavaScript decorators (TC39 proposal) have a complex history — they went through multiple spec revisions. TypeScript's `experimentalDecorators` mode uses an older spec that is incompatible with the current Stage 3 proposal. Migrating from TypeScript experimental decorators to the new spec is a breaking change in TypeScript 5.0+.

---

**Q7. What is the Command pattern and how does it enable undo/redo?** — Hard

**Answer:**
The Command pattern encapsulates a request as an object, allowing you to parameterize operations, queue them, log them, and support undoable operations.

```js
// Command interface:
class Command {
  execute() { throw new Error("Not implemented"); }
  undo() { throw new Error("Not implemented"); }
}

// Concrete commands:
class AddTextCommand extends Command {
  constructor(editor, text, position) {
    super();
    this.editor = editor;
    this.text = text;
    this.position = position;
  }

  execute() {
    this.editor.insertAt(this.text, this.position);
  }

  undo() {
    this.editor.deleteAt(this.position, this.text.length);
  }
}

// Command History (enables undo/redo):
class CommandHistory {
  #done = [];
  #undone = [];

  execute(command) {
    command.execute();
    this.#done.push(command);
    this.#undone = []; // clear redo stack on new action
  }

  undo() {
    const command = this.#done.pop();
    if (!command) return;
    command.undo();
    this.#undone.push(command);
  }

  redo() {
    const command = this.#undone.pop();
    if (!command) return;
    command.execute();
    this.#done.push(command);
  }

  canUndo() { return this.#done.length > 0; }
  canRedo() { return this.#undone.length > 0; }
}

// Usage:
const history = new CommandHistory();
const editor = new TextEditor();

history.execute(new AddTextCommand(editor, "Hello", 0));
history.execute(new AddTextCommand(editor, " World", 5));
history.undo(); // removes " World"
history.redo(); // re-adds " World"

// Functional approach — simpler for pure state:
function createHistoryManager(initialState) {
  const past = [];
  const future = [];
  let present = initialState;

  return {
    get state() { return present; },
    do(newState) {
      past.push(present);
      present = newState;
      future.length = 0;
    },
    undo() {
      if (!past.length) return;
      future.push(present);
      present = past.pop();
    },
    redo() {
      if (!future.length) return;
      past.push(present);
      present = future.pop();
    }
  };
}
```

**GOTCHA:** The undo stack can grow unboundedly — always cap it (e.g., last 100 commands). Also, the Command pattern requires that all mutations go through commands — if some code mutates state directly, undo/redo breaks. Consider using immutable state (Redux-style) where each command produces a new state snapshot, making undo trivially "go back to previous snapshot."

---

**Q8. What is the Proxy pattern and how do JavaScript `Proxy` objects implement it?** — Hard

**Answer:**
The Proxy pattern provides a surrogate or placeholder for another object, controlling access to it. JavaScript's built-in `Proxy` is a direct language-level implementation.

```js
// Proxy — validation:
function createValidatedUser(data) {
  const validators = {
    age: v => typeof v === "number" && v >= 0 && v <= 150,
    email: v => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
    name: v => typeof v === "string" && v.length >= 1
  };

  return new Proxy(data, {
    set(target, prop, value) {
      if (validators[prop] && !validators[prop](value)) {
        throw new RangeError(`Invalid value for ${prop}: ${value}`);
      }
      target[prop] = value;
      return true; // must return true to indicate success
    }
  });
}

const user = createValidatedUser({ name: "Alice", age: 30 });
user.age = 25;       // OK
user.age = -5;       // RangeError: Invalid value for age: -5
user.email = "bad";  // RangeError: Invalid value for email: bad

// Proxy — lazy loading:
function createLazyLoader(loadFn) {
  let loaded = null;
  return new Proxy({}, {
    get(target, prop) {
      if (!loaded) {
        loaded = loadFn();
        console.log("Loaded data on first access");
      }
      return loaded[prop];
    }
  });
}

// Proxy — logging / debugging:
function createLoggingProxy(obj, name = "obj") {
  return new Proxy(obj, {
    get(target, prop) {
      console.log(`GET ${name}.${String(prop)}`);
      return typeof target[prop] === "object" && target[prop] !== null
        ? createLoggingProxy(target[prop], `${name}.${String(prop)}`)
        : target[prop];
    },
    set(target, prop, value) {
      console.log(`SET ${name}.${String(prop)} = ${JSON.stringify(value)}`);
      target[prop] = value;
      return true;
    }
  });
}

// Proxy — reactive data binding (basis for Vue 3's reactivity):
function reactive(obj) {
  const listeners = new Map();

  return new Proxy(obj, {
    get(target, prop) {
      // Track which computed values depend on this property
      return target[prop];
    },
    set(target, prop, value) {
      target[prop] = value;
      // Notify all watchers of this property
      listeners.get(prop)?.forEach(fn => fn(value));
      return true;
    }
  });
}
```

All `Proxy` handler traps:
- `get`, `set`, `has` (`in` operator), `deleteProperty`
- `apply` (function calls), `construct` (`new`)
- `defineProperty`, `getOwnPropertyDescriptor`, `ownKeys`
- `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `preventExtensions`

**GOTCHA:** Proxies can break `===` identity checks — the proxy is not `===` the original target. Libraries that use identity-based caching (like some React internals) can behave unexpectedly with proxied objects. Also, `Proxy` cannot intercept private class fields (`#field`) — the `get` trap is not invoked for private field access.

---

**Q9. What is dependency injection (DI) and how is it applied in JavaScript?** — Hard

**Answer:**
Dependency Injection is a pattern where a component receives its dependencies from the outside rather than creating them itself, promoting loose coupling and testability.

```js
// WITHOUT DI — tightly coupled:
class OrderService {
  constructor() {
    this.db = new PostgresDatabase();       // hard-coded dependency
    this.emailer = new SendGridEmailer();   // hard-coded
    this.logger = new FileLogger();         // hard-coded
  }

  async createOrder(data) {
    const order = await this.db.insert("orders", data);
    await this.emailer.send(data.email, "Order confirmed");
    this.logger.log(`Order created: ${order.id}`);
    return order;
  }
}
// Problems: can't test without real DB, can't swap implementations

// WITH DI — loosely coupled:
class OrderService {
  constructor({ db, emailer, logger }) {  // dependencies injected
    this.db = db;
    this.emailer = emailer;
    this.logger = logger;
  }

  async createOrder(data) {
    const order = await this.db.insert("orders", data);
    await this.emailer.send(data.email, "Order confirmed");
    this.logger.log(`Order created: ${order.id}`);
    return order;
  }
}

// Production:
const orderService = new OrderService({
  db: new PostgresDatabase(config.dbUrl),
  emailer: new SendGridEmailer(config.sendgridKey),
  logger: new FileLogger("orders.log")
});

// Testing — inject mocks:
const mockDb = { insert: jest.fn().mockResolvedValue({ id: 99 }) };
const mockEmailer = { send: jest.fn().mockResolvedValue() };
const mockLogger = { log: jest.fn() };

const testOrderService = new OrderService({
  db: mockDb,
  emailer: mockEmailer,
  logger: mockLogger
});

// DI container (lightweight):
class Container {
  #registry = new Map();

  register(name, factory) {
    this.#registry.set(name, factory);
  }

  resolve(name) {
    const factory = this.#registry.get(name);
    if (!factory) throw new Error(`Unknown dependency: ${name}`);
    return factory(this);
  }
}

const container = new Container();
container.register("db", () => new PostgresDatabase(config.dbUrl));
container.register("emailer", () => new SendGridEmailer(config.key));
container.register("orderService", (c) => new OrderService({
  db: c.resolve("db"),
  emailer: c.resolve("emailer"),
  logger: console
}));

const orderService = container.resolve("orderService");
```

**GOTCHA:** DI increases constructor complexity — if a class has 8+ dependencies, it may be a sign it's doing too much (violating the Single Responsibility Principle). Also, DI containers that use string keys for dependencies lose type safety — TypeScript-based DI frameworks (InversifyJS, tsyringe) use decorators and reflection metadata to provide typed injection.

---

**Q10. What is the Repository pattern and how does it separate data access concerns?** — Medium

**Answer:**
The Repository pattern abstracts the data layer, providing a collection-like interface for accessing domain objects while hiding storage implementation details.

```js
// Repository interface (conceptual):
// findById(id) → User
// findAll(filter?) → User[]
// save(user) → User
// delete(id) → void

// Concrete implementation — PostgreSQL:
class UserRepository {
  constructor(db) {
    this.db = db;
  }

  async findById(id) {
    const rows = await this.db.query(
      "SELECT * FROM users WHERE id = $1",
      [id]
    );
    return rows[0] ? this.#toModel(rows[0]) : null;
  }

  async findAll({ page = 1, limit = 20, role } = {}) {
    const offset = (page - 1) * limit;
    const rows = await this.db.query(
      `SELECT * FROM users ${role ? "WHERE role = $3" : ""}
       ORDER BY created_at DESC LIMIT $1 OFFSET $2`,
      role ? [limit, offset, role] : [limit, offset]
    );
    return rows.map(this.#toModel.bind(this));
  }

  async save(user) {
    if (user.id) {
      // Update:
      await this.db.query(
        "UPDATE users SET name = $2, email = $3 WHERE id = $1",
        [user.id, user.name, user.email]
      );
      return user;
    }
    // Insert:
    const [row] = await this.db.query(
      "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
      [user.name, user.email]
    );
    return this.#toModel(row);
  }

  async delete(id) {
    await this.db.query("DELETE FROM users WHERE id = $1", [id]);
  }

  #toModel(row) {
    return { id: row.id, name: row.name, email: row.email, role: row.role };
  }
}

// In-memory repository for testing — SAME interface:
class InMemoryUserRepository {
  #users = new Map();
  #nextId = 1;

  async findById(id) { return this.#users.get(id) ?? null; }
  async findAll() { return [...this.#users.values()]; }
  async save(user) {
    if (!user.id) user = { ...user, id: this.#nextId++ };
    this.#users.set(user.id, user);
    return user;
  }
  async delete(id) { this.#users.delete(id); }
}

// Service layer — works with ANY repository:
class UserService {
  constructor(userRepo) {
    this.userRepo = userRepo; // injected
  }

  async getUserById(id) {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return user;
  }
}

// Production: new UserService(new UserRepository(pgPool))
// Testing:    new UserService(new InMemoryUserRepository())
```

**GOTCHA:** The Repository pattern can lead to anemic domain models — all business logic lives in services, none in the model objects. In complex domains, consider the Domain Model pattern (rich models with behavior) instead of simple data-holder objects. Also, Repositories should return domain objects, not raw database rows — don't let database schema leak into your business logic.

---

**Q11. What is the Iterator pattern and how do custom iterators work in JavaScript?** — Medium

**Answer:**
The Iterator pattern provides a way to sequentially access elements of a collection without exposing its underlying representation.

```js
// Custom iterator — implements iterator protocol:
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  // Make it iterable:
  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
      // Also make the iterator itself iterable:
      [Symbol.iterator]() { return this; }
    };
  }
}

const range = new Range(1, 10, 2);
for (const n of range) console.log(n); // 1, 3, 5, 7, 9

// Spread, destructuring, and Array.from work automatically:
[...new Range(1, 5)];      // [1, 2, 3, 4, 5]
const [a, b, c] = new Range(10, 30, 10); // 10, 20, 30

// Generator as iterator — much simpler:
function* range2(start, end, step = 1) {
  for (let i = start; i <= end; i += step) yield i;
}
[...range2(1, 5)]; // [1, 2, 3, 4, 5]

// Infinite iterator with lazy evaluation:
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

// Take first N from an infinite iterator:
function* take(n, iterable) {
  let i = 0;
  for (const item of iterable) {
    if (i++ >= n) break;
    yield item;
  }
}

[...take(5, naturals())]; // [1, 2, 3, 4, 5]

// Iterator helpers (Iterator.prototype proposal — TC39):
// naturals().take(5).map(n => n * 2).toArray(); // [2, 4, 6, 8, 10]
```

**GOTCHA:** Iterators are stateful and single-use — once exhausted, they don't restart. `for...of` calls `[Symbol.iterator]()` on the iterable, so iterables can be iterated multiple times (each call creates a fresh iterator). But if the iterator IS the iterable (generator objects), iterating twice doesn't reset — you get an already-exhausted iterator the second time.

---

**Q12. What is the Builder pattern and when is it useful?** — Medium

**Answer:**
The Builder pattern constructs complex objects step-by-step, separating construction from representation. Useful when an object requires many configuration options.

```js
// Without builder — constructor with too many parameters:
new Query(table, fields, whereClause, orderBy, limit, offset, joins); // unreadable!

// With builder — fluent API:
class QueryBuilder {
  #query = {
    table: null,
    fields: ["*"],
    where: [],
    joins: [],
    orderBy: null,
    limit: null,
    offset: 0
  };

  from(table) {
    this.#query.table = table;
    return this; // fluent interface — return this for chaining
  }

  select(...fields) {
    this.#query.fields = fields;
    return this;
  }

  where(condition, params) {
    this.#query.where.push({ condition, params });
    return this;
  }

  join(table, on) {
    this.#query.joins.push({ table, on });
    return this;
  }

  orderBy(field, direction = "ASC") {
    this.#query.orderBy = `${field} ${direction}`;
    return this;
  }

  limit(n) {
    this.#query.limit = n;
    return this;
  }

  offset(n) {
    this.#query.offset = n;
    return this;
  }

  build() {
    if (!this.#query.table) throw new Error("table is required");
    let sql = `SELECT ${this.#query.fields.join(", ")} FROM ${this.#query.table}`;
    if (this.#query.joins.length) {
      sql += this.#query.joins.map(j => ` JOIN ${j.table} ON ${j.on}`).join("");
    }
    if (this.#query.where.length) {
      sql += " WHERE " + this.#query.where.map(w => w.condition).join(" AND ");
    }
    if (this.#query.orderBy) sql += ` ORDER BY ${this.#query.orderBy}`;
    if (this.#query.limit) sql += ` LIMIT ${this.#query.limit} OFFSET ${this.#query.offset}`;
    return sql;
  }
}

// Fluent, readable construction:
const sql = new QueryBuilder()
  .from("users")
  .select("id", "name", "email")
  .join("orders", "orders.user_id = users.id")
  .where("users.active = true")
  .where("users.role = $1", ["admin"])
  .orderBy("users.created_at", "DESC")
  .limit(20)
  .offset(0)
  .build();
```

**GOTCHA:** The Builder pattern's fluent interface relies on returning `this`. If you forget `return this` on any method, the chain breaks with a `TypeError: Cannot read properties of undefined`. TypeScript helps catch this with return type annotations. Also, builders with `build()` validation are great, but builders that SILENTLY produce invalid objects (no `build()` validation) are dangerous — always validate at `build()` time.

---

**Q13. What is the SOLID principle and how does it apply to JavaScript?** — Hard

**Answer:**
SOLID is an acronym for five design principles that make software more understandable, flexible, and maintainable.

```js
// S — Single Responsibility Principle (SRP):
// Each class/module has ONE reason to change

// WRONG — UserManager does too much:
class UserManager {
  createUser(data) { /* DB insert */ }
  sendWelcomeEmail(user) { /* Email logic */ }
  generateReport() { /* Reporting logic */ }
  hashPassword(pwd) { /* Crypto */ }
}

// RIGHT — separate concerns:
class UserRepository { createUser(data) { /* DB only */ } }
class EmailService { sendWelcome(user) { /* Email only */ } }
class UserReporter { generateReport() { /* Reporting only */ } }
class PasswordHasher { hash(pwd) { /* Crypto only */ } }

// O — Open/Closed Principle:
// Open for extension, closed for modification

// WRONG — must modify class to add new discount type:
class PriceCalculator {
  calculate(price, discountType) {
    if (discountType === "10%") return price * 0.9;
    if (discountType === "seasonal") return price * 0.8;
    // Must edit this class every time a new discount is added
  }
}

// RIGHT — add new strategies without modifying existing code:
class PriceCalculator {
  constructor(discountStrategy) { this.strategy = discountStrategy; }
  calculate(price) { return this.strategy(price); }
}
const tenPercent = price => price * 0.9;
const seasonal = price => price * 0.8;
const blackFriday = price => price * 0.5; // NEW — no modification to calculator

// L — Liskov Substitution Principle:
// Subtypes must be substitutable for their base types

// I — Interface Segregation Principle:
// Clients should not depend on interfaces they don't use

// D — Dependency Inversion Principle:
// Depend on abstractions, not concretions
class UserService {
  constructor(userRepo) { // depends on abstraction (interface)
    this.userRepo = userRepo;
  }
}
// Any object with {findById, save, delete} methods works
```

**GOTCHA:** SOLID principles are guidelines, not dogma. Over-applying them leads to over-engineering — dozens of tiny classes, excessive abstraction, and code that's harder to follow than a straightforward procedural solution. Apply them where complexity and change frequency justify it. In small projects or scripts, SOLID compliance can be unnecessary overhead.

---

**Q14. What is event sourcing and CQRS? When would you apply them?** — Hard

**Answer:**
**Event Sourcing** stores the history of state changes (events) rather than current state. **CQRS** (Command Query Responsibility Segregation) separates read and write models.

```js
// Event Sourcing — store events, derive state from replay:
class EventStore {
  #events = [];

  append(aggregateId, event) {
    this.#events.push({
      aggregateId,
      type: event.type,
      payload: event.payload,
      timestamp: Date.now(),
      version: this.#events.filter(e => e.aggregateId === aggregateId).length + 1
    });
  }

  getEvents(aggregateId) {
    return this.#events.filter(e => e.aggregateId === aggregateId);
  }
}

// Rebuild state from events:
function rehydrate(events) {
  return events.reduce((state, event) => {
    switch (event.type) {
      case "AccountOpened":
        return { ...state, id: event.payload.id, balance: 0, open: true };
      case "MoneyDeposited":
        return { ...state, balance: state.balance + event.payload.amount };
      case "MoneyWithdrawn":
        return { ...state, balance: state.balance - event.payload.amount };
      case "AccountClosed":
        return { ...state, open: false };
      default:
        return state;
    }
  }, {});
}

// Benefits:
// - Complete audit trail (compliance, debugging)
// - Temporal queries ("what was the state on 2024-01-15?")
// - Event replay for projections
// - Easy undo (rewind events)

// CQRS — separate read and write models:
// Write model (commands → events):
class AccountCommandHandler {
  constructor(eventStore) { this.store = eventStore; }

  deposit(accountId, amount) {
    if (amount <= 0) throw new RangeError("Amount must be positive");
    this.store.append(accountId, {
      type: "MoneyDeposited",
      payload: { amount }
    });
  }
}

// Read model (optimized projection for queries):
class AccountReadModel {
  #balances = new Map();

  // Updated by event projection (eventually consistent):
  apply(event) {
    if (event.type === "MoneyDeposited") {
      const cur = this.#balances.get(event.aggregateId) ?? 0;
      this.#balances.set(event.aggregateId, cur + event.payload.amount);
    }
  }

  getBalance(accountId) {
    return this.#balances.get(accountId) ?? 0;
  }
}
```

**When to apply:**
- Event Sourcing: audit trails required (financial, medical), complex domain with many state transitions, need for temporal queries
- CQRS: heavily read-optimized systems, complex queries that don't match write model shape, teams working independently on reads vs writes

**GOTCHA:** Event Sourcing + CQRS adds SIGNIFICANT complexity — eventual consistency between write and read models, event schema versioning (what happens when your event structure changes?), and snapshot management for aggregates with thousands of events. Only use it when the domain truly needs it — most CRUD apps don't.

---

**Q15. What is the Middleware pattern and how do Express and Redux implement it?** — Medium

**Answer:**
The Middleware pattern creates a pipeline of functions, each receiving a request (or action), optionally modifying it, and passing control to the next function.

```js
// Express middleware pattern:
// Each middleware: (req, res, next) => void
// next() — passes control to next middleware
// next(err) — passes to error handling middleware

const express = require("express");
const app = express();

// Global middleware:
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // MUST call next or the request hangs
});

// Error handling middleware — 4 parameters:
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});

// Building a custom middleware pipeline:
function compose(...middlewares) {
  return function (context) {
    function dispatch(i) {
      if (i === middlewares.length) return Promise.resolve();
      const middleware = middlewares[i];
      return Promise.resolve(middleware(context, () => dispatch(i + 1)));
    }
    return dispatch(0);
  };
}

// Koa-style middleware:
const pipeline = compose(
  async (ctx, next) => { console.log("before"); await next(); console.log("after"); },
  async (ctx, next) => { ctx.data = "processed"; await next(); },
  async (ctx, next) => { console.log("end of pipeline"); }
);

await pipeline({});
// "before" → "processed" → "end of pipeline" → "after" (onion model)

// Redux middleware:
// Middleware signature: store => next => action => result
const logger = store => next => action => {
  console.log("dispatch:", action);
  const result = next(action);
  console.log("next state:", store.getState());
  return result;
};

const thunk = store => next => action => {
  if (typeof action === "function") {
    return action(store.dispatch, store.getState);
  }
  return next(action);
};

// applyMiddleware — composes middlewares:
// [logger, thunk] → logger(thunk(store.dispatch))
```

**Follow-up:** What is the difference between Express and Koa middleware?

Express middleware uses `(req, res, next)` — it's a linear pipeline where errors must be passed via `next(err)`. Koa uses async generators and the "onion model" — `await next()` pauses the current middleware, runs all subsequent ones, then resumes after `next()` returns. This makes "after" logic (like response time logging) much cleaner in Koa.

**GOTCHA:** In Express, forgetting `next()` in middleware freezes the request — it never completes. Forgetting `return` after `next()` causes both the current middleware and the next to run in parallel, leading to "headers already sent" errors when both try to send a response.

---

*Next: [16-Output-Tricky-Questions.md](./16-Output-Tricky-Questions.md)*
