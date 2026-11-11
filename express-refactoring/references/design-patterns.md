# Design Patterns for Node.js / Express

All 22 patterns from the Refactoring.Guru catalog (the classic "Gang of Four" set), with what each looks like in idiomatic Node.js, when it helps in a backend, and when to avoid it. Pattern names follow the catalog; explanations and examples here are this skill's own.

## How to use patterns in this skill

**A pattern is a fix for a specific problem, never a goal.** Introduce one only when you can name the smell or concrete need it solves in the current task (e.g. "this `switch` on payment provider appears in four handlers" → Strategy). If you can't name it, the plain function is better. All the guardrails and minimal-impact rules in SKILL.md apply.

**JavaScript changes the shape of many patterns.** Functions are values, modules are cached singletons, objects can be created with literals, and duck typing replaces interfaces. So many classic patterns shrink to a few lines — or are already built into Node and Express. Prefer the lightest form shown under each pattern.

**Already in use whether you notice or not:**
- Express middleware → Chain of Responsibility
- `EventEmitter` → Observer
- `require` / ES module caching → Singleton
- Streams and `for await` → Iterator
- Query builders (Knex, Prisma, Mongoose queries) → Builder

## Quick selector

| Problem you see | Pattern to consider | Usefulness in Express backends |
|---|---|---|
| `switch` on a type/provider repeated in several places | Strategy | High |
| Creating one of several variants based on config or input | Factory Method | High |
| A third-party SDK's API leaking all over your code | Adapter | High |
| Handlers that must call several services in the right order | Facade | High |
| Adding logging/caching/retries around a function without editing it | Decorator / Proxy | High |
| A sequence of checks where each may stop the request | Chain of Responsibility | High (it's middleware) |
| One action should trigger several independent reactions | Observer | High |
| Work that should be queued, retried or undone later | Command | Medium |
| An object whose allowed actions depend on its status | State | Medium |
| Same steps in the same order, details differ | Template Method | Medium |
| Building a complex object or query step by step | Builder | Medium |
| One shared instance (DB pool, logger) | Singleton | Medium — use module caching, carefully |
| Tree structures (permissions, categories, menus) | Composite | Low–medium |
| Many modules talking to each other directly | Mediator | Low |
| Families of related objects that must match | Abstract Factory | Low |
| Two dimensions of variation multiplying classes | Bridge | Low |
| Custom traversal of a collection | Iterator | Low — use generators |
| Copying preconfigured objects | Prototype | Low — use spread / `structuredClone` |
| Huge numbers of similar small objects in memory | Flyweight | Low |
| Saving and restoring an object's previous state | Memento | Low |
| New operations over a fixed set of node types (ASTs) | Visitor | Low |

## Contents
- Creational: Factory Method, Abstract Factory, Builder, Prototype, Singleton
- Structural: Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
- Behavioral: Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor

---

# Creational patterns

## Factory Method

**Problem it solves:** code needs to create one of several variants, and the choice depends on config or input — so `new X()` / `if (provider === ...)` creation logic spreads around.

**Node idiom:** a factory function that returns the right implementation. All implementations share the same functions (duck typing).

```js
// storage/index.js
const { createS3Storage } = require('./s3');
const { createLocalStorage } = require('./local');

const storageFactories = { s3: createS3Storage, local: createLocalStorage };

function createStorage(config) {
  if (!Object.hasOwn(storageFactories, config.storageDriver)) {
    throw new Error(`unknown storage driver ${config.storageDriver}`);
  }
  return storageFactories[config.storageDriver](config);
}
module.exports = { createStorage };
```

**Avoid when** there's only one variant. `new` or a direct call is clearer.

Link: https://refactoring.guru/design-patterns/factory-method

---

## Abstract Factory

**Problem it solves:** you create *families* of related objects that must match (e.g. a payment provider's client, webhook verifier and refund handler must all be for the same provider).

**Node idiom:** a factory that returns an object of matching factories or instances:

```js
const providers = {
  stripe: () => ({ client: createStripeClient(), verifyWebhook: verifyStripeSignature, refunds: createStripeRefunds() }),
  paypal: () => ({ client: createPaypalClient(), verifyWebhook: verifyPaypalSignature, refunds: createPaypalRefunds() }),
};
```

**Avoid when** you only have one family. Rarely needed in typical Express apps.

Link: https://refactoring.guru/design-patterns/abstract-factory

---

## Builder

**Problem it solves:** constructing something step by step with many optional parts — most often a database query or a complex response/report.

**Node idiom:** usually you *use* a builder rather than write one — Knex, Prisma, Mongoose and SQL query builders are builders. Write your own only when the same multi-step construction is repeated:

```js
function buildOrderQuery({ status, customerId, from, to, sort = '-createdAt', limit = 20 }) {
  let query = Order.find();
  if (status) query = query.where('status').equals(status);
  if (customerId) query = query.where('customerId').equals(customerId);
  if (from) query = query.where('createdAt').gte(from);
  if (to) query = query.where('createdAt').lte(to);
  return query.sort(sort).limit(limit);
}
```

**Avoid when** an object literal or an options object (Introduce Parameter Object) is enough. A fluent `new XBuilder().setA().setB().build()` class for a plain object is overkill in JS.

**Security:** when building SQL, always use the builder's parameter binding; never concatenate request values into query strings.

Link: https://refactoring.guru/design-patterns/builder

---

## Prototype

**Problem it solves:** creating new objects by copying a preconfigured one.

**Node idiom:** spread syntax for shallow copies (`{ ...defaults, ...overrides }`) and `structuredClone()` for deep copies. JavaScript's own inheritance is prototype-based, but you rarely need to work with it directly.

**Watch for:** spread copies are shallow — nested objects are still shared. `structuredClone` doesn't copy functions or class instances' prototypes.

Link: https://refactoring.guru/design-patterns/prototype

---

## Singleton

**Problem it solves:** exactly one shared instance — a database pool, a logger, a cache client.

**Node idiom:** module caching already does this. A module is evaluated once per process, so exporting an instance gives every importer the same one:

```js
// db.js
const { Pool } = require('pg');
const config = require('./config');
module.exports = new Pool({ connectionString: config.databaseUrl });
```

**Cautions:**
- It's one instance **per process**, not per app. With cluster mode, PM2 or several pods, each process has its own — never rely on a module singleton for shared state such as rate limits or locks; use Redis or the database.
- Module-level singletons make tests harder (they can't easily be replaced). Where testing matters, export a `create...()` function and create the instance once in the app's composition root (`app.js` / `server.js`), passing it to services.
- Never keep per-request or per-user data in a singleton — every request shares it.

Link: https://refactoring.guru/design-patterns/singleton

---

# Structural patterns

## Adapter

**Problem it solves:** a third-party library or external API has an interface that doesn't match what your code expects, and its details leak into many files.

**Node idiom:** a small wrapper module that exposes *your* functions and translates to the library. The rest of the code imports the adapter.

```js
// lib/mailer.js — adapter over the provider's SDK
const provider = require('some-email-sdk');
const client = provider.createClient({ apiKey: config.emailApiKey });

async function sendEmail({ to, subject, html }) {
  await client.messages.send({ recipient: to, title: subject, body: { html } });
}
module.exports = { sendEmail };
```

Changing email provider now touches one file. This is the most useful pattern in backend work. (See also Introduce Local Extension in `moving-features.md`.)

**Avoid when** the library is already stable, simple and used in one place.

Link: https://refactoring.guru/design-patterns/adapter

---

## Bridge

**Problem it solves:** two independent dimensions of variation would otherwise multiply into many combinations (e.g. notification *type* × delivery *channel*: `WelcomeEmail`, `WelcomeSms`, `InvoiceEmail`, `InvoiceSms` ...).

**Node idiom:** separate the two dimensions and combine them at runtime:

```js
const messages = { welcome: (user) => `Welcome, ${user.name}!`, invoice: (user, inv) => `Invoice ${inv.id} is ready` };
const channels = { email: sendEmail, sms: sendSms };

async function notify(messageType, channel, user, data) {
  const text = messages[messageType](user, data);
  await channels[channel]({ to: user, text });
}
```

**Avoid when** only one dimension actually varies.

Link: https://refactoring.guru/design-patterns/bridge

---

## Composite

**Problem it solves:** tree structures where a single item and a group of items should be treated the same way — nested permission groups, category trees, nested menus, bundles of products.

**Node idiom:** recursive functions over plain objects:

```js
// A product or a bundle of products — both have a price
function priceOf(node) {
  return node.children ? node.children.reduce((sum, child) => sum + priceOf(child), 0) : node.price;
}
```

**Watch for:** deep or cyclic trees from user data — guard recursion depth and detect cycles.

Link: https://refactoring.guru/design-patterns/composite

---

## Decorator

**Problem it solves:** adding behavior (logging, timing, retries, caching, metrics) around a function without changing it or its callers.

**Node idiom:** a higher-order function that returns a wrapped version with the same signature:

```js
function withTiming(name, fn) {
  return async (...args) => {
    const start = performance.now();
    try {
      return await fn(...args);
    } finally {
      metrics.histogram(`${name}.duration_ms`, performance.now() - start);
    }
  };
}

const findUser = withTiming('db.findUser', rawFindUser);
```

This is also the "Wrap Function" technique in `legacy-changes.md`. Express middleware is decorator-like for whole requests.

**Avoid** stacking many wrappers until stack traces become unreadable.

Link: https://refactoring.guru/design-patterns/decorator

---

## Facade

**Problem it solves:** route handlers have to coordinate several modules (DB, payment, email, inventory) in the right order, and that coordination is repeated or tangled with HTTP code.

**Node idiom:** a service function that offers one simple call for the whole workflow. The service layer in `express-patterns.md` is a facade.

```js
// services/checkout.js
async function checkout({ userId, cartId, paymentMethodId }) {
  const cart = await carts.load(cartId);
  await inventory.reserve(cart.items);
  const payment = await payments.charge({ userId, amount: cart.total, paymentMethodId });
  const order = await orders.create({ userId, cart, paymentId: payment.id });
  await emails.sendOrderConfirmation(userId, order);
  return order;
}
```

**Avoid** a facade that only forwards single calls — that's the Middle Man smell.

Link: https://refactoring.guru/design-patterns/facade

---

## Flyweight

**Problem it solves:** memory pressure from huge numbers of similar objects, by sharing the common, immutable part.

**In Node backends:** rarely needed. Relevant cases are large in-memory caches or processing millions of records, where repeated identical sub-objects (e.g. the same currency/locale config on every row) can be shared via a lookup table instead of copied. Measure memory first; don't apply it speculatively.

Link: https://refactoring.guru/design-patterns/flyweight

---

## Proxy

**Problem it solves:** controlling access to an object — caching, lazy initialization, access checks, rate limiting — with the same interface as the original.

**Node idiom:** usually a wrapper object or function with the same API (same as Decorator in shape; the intent is *control*, not *added behavior*). JavaScript's built-in `Proxy` exists, but it's harder to read and debug — prefer an explicit wrapper.

```js
// Caching proxy for an external rates API
function createCachedRatesClient(client, { ttlMs = 60_000 } = {}) {
  const cache = new Map();
  return {
    async getRate(from, to) {
      const key = `${from}:${to}`;
      const hit = cache.get(key);
      if (hit && Date.now() - hit.at < ttlMs) return hit.value;
      const value = await client.getRate(from, to);
      cache.set(key, { value, at: Date.now() });
      return value;
    },
  };
}
```

**Watch for:** unbounded in-memory caches grow forever — add a size limit or use an LRU library or Redis. In-process caches are per process, not shared across instances.

Link: https://refactoring.guru/design-patterns/proxy

---

# Behavioral patterns

## Chain of Responsibility

**Problem it solves:** a request passes through a sequence of handlers, each of which may handle it, change it, or stop it.

**Node idiom:** this *is* Express middleware — `(req, res, next)` and the order of `app.use` / route arguments define the chain:

```js
router.post('/payouts', authenticate, requireRole('admin'), validate(payoutSchema), createPayout);
```

Use it by writing small, single-purpose middleware (see `express-patterns.md`) rather than one large function with many checks. **Order is behavior** — moving a middleware changes what happens.

Link: https://refactoring.guru/design-patterns/chain-of-responsibility

---

## Command

**Problem it solves:** turning an action into an object, so it can be queued, scheduled, retried, logged or undone.

**Node idiom:** a job payload plus a handler — exactly what job queues (BullMQ, Agenda, SQS consumers) do:

```js
// enqueue (in the request)
await queue.add('send-invoice', { invoiceId: invoice.id });

// handle (in the worker)
const handlers = {
  'send-invoice': async ({ invoiceId }) => { /* ... */ },
};
```

Keep job payloads small and serializable (IDs, not whole objects), and make handlers **idempotent**, because queues may deliver a job more than once.

Link: https://refactoring.guru/design-patterns/command

---

## Iterator

**Problem it solves:** walking through a collection without exposing how it's stored.

**Node idiom:** built in — `for...of`, generators, async generators and streams. Useful for paginated APIs and large DB result sets, so you don't load everything into memory:

```js
async function* allCustomers(pageSize = 500) {
  for (let page = 0; ; page++) {
    const batch = await Customer.find().skip(page * pageSize).limit(pageSize);
    if (batch.length === 0) return;
    yield* batch;
  }
}

for await (const customer of allCustomers()) { /* ... */ }
```

(For large tables, cursor/keyset pagination is faster than `skip` — but changing it is a behavior/performance change, not a refactoring.)

Link: https://refactoring.guru/design-patterns/iterator

---

## Mediator

**Problem it solves:** many modules call each other directly, creating a tangled web of dependencies; a central coordinator reduces the connections.

**In Node backends:** an in-process event bus or a message broker (Redis pub/sub, Kafka, RabbitMQ) plays this role between services. Within one app, a facade/service often suffices.

**Avoid** a "god" mediator that knows everything — it becomes a Large Class.

Link: https://refactoring.guru/design-patterns/mediator

---

## Memento

**Problem it solves:** saving and restoring an object's previous state without exposing its internals.

**In Node backends:** rare as an in-memory pattern. The backend equivalents are audit/history tables, versioned documents, event sourcing, and database transactions (rollback). Use those rather than building in-memory snapshots.

Link: https://refactoring.guru/design-patterns/memento

---

## Observer

**Problem it solves:** one event should trigger several independent reactions, without the source knowing about each of them.

**Node idiom:** `EventEmitter`, or a message broker across services.

```js
// events.js
const { EventEmitter } = require('node:events');
module.exports = new EventEmitter();

// services/orders.js — emits, doesn't know who listens
events.emit('order.paid', { orderId: order.id });

// listeners registered once at startup
events.on('order.paid', ({ orderId }) => emails.sendReceipt(orderId).catch(logger.error));
events.on('order.paid', ({ orderId }) => analytics.track('order_paid', { orderId }));
```

**Cautions:**
- `emit` is synchronous and doesn't wait for async listeners; errors in async listeners won't reach the emitter — catch them in each listener.
- In-process events are lost if the process crashes. For anything that must happen (payments, emails), use a durable queue (Command pattern).
- Hidden control flow: document which events exist and who listens.

Link: https://refactoring.guru/design-patterns/observer

---

## State

**Problem it solves:** an object behaves differently depending on its status, and `if (status === ...)` checks are scattered everywhere.

**Node idiom:** a state table (object of states with allowed transitions and per-state behavior) — see Replace Type Code with State/Strategy in `organizing-data.md` for a full order-status example.

For complex workflows, an explicit state machine library (e.g. XState) can be worth it; for a handful of states, a plain object is enough.

**Concurrency:** check-and-transition must be atomic in the database (a conditional update `WHERE status = 'PENDING'`), or two requests can both make the same transition.

Link: https://refactoring.guru/design-patterns/state

---

## Strategy

**Problem it solves:** several interchangeable algorithms (pricing rules, shipping calculators, payment providers, export formats), selected at runtime, with the selection `switch` duplicated across the code.

**Node idiom:** a lookup map of functions — see Replace Conditional with Polymorphism in `simplifying-conditionals.md`:

```js
const exporters = { csv: toCsv, json: toJson, xlsx: toXlsx };

router.get('/reports/:id/export', async (req, res) => {
  const { format } = req.query;
  if (!Object.hasOwn(exporters, format)) return res.status(400).json({ error: 'unsupported format' });
  const report = await reports.load(req.params.id);
  res.send(await exporters[format](report));
});
```

**Avoid when** there's one algorithm, or a single short `switch` in one place.

Link: https://refactoring.guru/design-patterns/strategy

---

## Template Method

**Problem it solves:** several processes follow the same steps in the same order, but some steps differ.

**Node idiom:** a higher-order function that fixes the sequence and takes the varying steps as functions — see Form Template Method in `generalization.md` for an importer example. A base class with abstract methods also works in class-based code.

**Avoid when** the "shared" sequence is only two lines — duplication is cheaper than the abstraction.

Link: https://refactoring.guru/design-patterns/template-method

---

## Visitor

**Problem it solves:** adding new operations over a fixed set of node types (an AST, a document tree) without changing the node classes.

**In Node backends:** rare. It shows up when writing ESLint rules, Babel plugins or query/AST transformers — and those tools already provide visitor APIs (`{ Identifier(node) { ... } }`). Use theirs; don't build your own in application code.

Link: https://refactoring.guru/design-patterns/visitor
