# Organizing Data

Refactorings about how data is represented, shared and protected. In JavaScript, read *field* as a property or a module-level variable, and *class* as a class or a module.

## Contents
- Replace Magic Number with Symbolic Constant
- Change Value to Reference
- Change Reference to Value
- Encapsulate Field
- Self Encapsulate Field
- Encapsulate Collection
- Replace Data Value with Object
- Replace Array with Object
- Replace Type Code with Class
- Replace Type Code with Subclasses
- Replace Type Code with State/Strategy
- Replace Subclass with Fields
- Change Unidirectional Association to Bidirectional
- Change Bidirectional Association to Unidirectional
- Duplicate Observed Data

(Introduce Parameter Object and splitting functions by a flag parameter are in `simplifying-function-calls.md`.)

---

## Replace Magic Number with Symbolic Constant

**Use when** a literal has a meaning that isn't obvious (`86400`, `5242880`, `'ACTIVE'`), or the same literal appears in several places and must change together.

```js
// Before
if (file.size > 5242880) return res.status(413).json({ error: 'too large' });
const users = await User.find({ status: 'ACTIVE' });

// After
const MAX_UPLOAD_BYTES = 5 * 1024 * 1024;
const UserStatus = Object.freeze({ ACTIVE: 'ACTIVE', SUSPENDED: 'SUSPENDED' });

if (file.size > MAX_UPLOAD_BYTES) return res.status(413).json({ error: 'too large' });
const users = await User.find({ status: UserStatus.ACTIVE });
```

- Search first: the project may already have a `constants.js`, an enum module, or status codes from a library.
- Status codes like `200`, `201`, `404` are well known; a named constant is optional. Follow the project's convention.
- Values that differ per environment belong in the config module, not in constants.

Link: https://refactoring.guru/replace-magic-number-with-symbolic-constant

---

## Change Value to Reference

**Use when** the same real-world entity exists as several independent copies, and an update to one copy should be visible through all of them — but isn't.

```js
// Before — every order embeds its own copy of the customer
const order1 = { id: 1, customer: { id: 7, name: 'Lan', tier: 'GOLD' } };
const order2 = { id: 2, customer: { id: 7, name: 'Lan', tier: 'GOLD' } };
// Upgrading the tier on order1.customer does nothing for order2.customer.

// After — store the customer once; orders refer to it
const customers = new Map([[7, { id: 7, name: 'Lan', tier: 'GOLD' }]]);
const order1 = { id: 1, customerId: 7 };
const order2 = { id: 2, customerId: 7 };
const customerOf = (order) => customers.get(order.customerId);
```

In a backend, the "single place" is usually the database: store `customerId` and look it up, instead of denormalizing the customer into each order document. If you need a per-request cache, scope it to the request (a `Map` created per request, or a DataLoader) — a module-level `Map` is shared by every request and every user.

**Check first:** sometimes a copy is intentional. An invoice must keep the customer's name and address *as they were* when it was issued. Don't convert that to a reference.

Link: https://refactoring.guru/change-value-to-reference

---

## Change Reference to Value

**Use when** one object is shared and mutated from many places, and it's hard to know who changed it. Module-level objects and arrays in Node are shared by every request, so accidental mutation there is a real bug source.

Make the object immutable and return new copies instead of changing it:

```js
// Before — anyone can change the shared defaults
const defaultPagination = { page: 1, limit: 20 };
// in a handler: const p = defaultPagination; p.limit = req.query.limit;  ← changes it for everyone

// After
const DEFAULT_PAGINATION = Object.freeze({ page: 1, limit: 20 });
const paginationFrom = (query) => ({ ...DEFAULT_PAGINATION, ...pickPagination(query) });
```

`Object.freeze` is shallow. For nested objects, freeze each level or build them so nothing else holds a mutable reference.

Link: https://refactoring.guru/change-reference-to-value

---

## Encapsulate Field

**Use when** a module exports a mutable variable (or an object exposes a property) that other modules read and write directly, so there's no single place to validate or track changes.

```js
// Before — config.js
module.exports = { featureFlags: { newCheckout: false } };
// anywhere: require('./config').featureFlags.newCheckout = true;

// After
const featureFlags = { newCheckout: false };
module.exports = {
  isEnabled: (flag) => Boolean(featureFlags[flag]),
  setFlag: (flag, value) => {          // one place to validate and log changes
    if (!(flag in featureFlags)) throw new Error(`unknown flag ${flag}`);
    featureFlags[flag] = value;
  },
};
```

In classes, use private fields (`#balance`) with methods or getters. Don't add getters/setters for every property by reflex — plain data objects (API payloads, DB rows) don't need them.

Link: https://refactoring.guru/encapsulate-field

---

## Self Encapsulate Field

**Use when** even code *inside* the class or module should go through a getter, because the value is about to become computed, lazy-loaded, or validated.

```js
class RateLimiter {
  #limit;
  constructor(limit) { this.#limit = limit; }
  get limit() { return this.#limit; }      // all internal code reads this.limit
  isAllowed(count) { return count < this.limit; }
}
```

Now `limit` can later come from config or a per-tenant lookup by changing only the getter. Don't do this speculatively — only when such a change is actually coming.

Link: https://refactoring.guru/self-encapsulate-field

---

## Encapsulate Collection

**Use when** a module returns its internal array or `Map` directly, so callers can modify it without the owner knowing.

```js
// Before
const allowedOrigins = ['https://app.example.com'];
module.exports = { allowedOrigins };
// elsewhere: allowedOrigins.push(req.headers.origin)  ← affects every later request

// After — expose read access and explicit operations only
const allowedOrigins = new Set(['https://app.example.com']);
module.exports = {
  isAllowedOrigin: (origin) => allowedOrigins.has(origin),
  listAllowedOrigins: () => [...allowedOrigins],   // a copy
};
```

Link: https://refactoring.guru/encapsulate-collection

---

## Replace Data Value with Object

**Use when** a primitive keeps collecting related logic around it — a money amount with its currency, a date range with its validation, an email normalized in five places (Primitive Obsession).

```js
// Before — money logic duplicated wherever amounts are added
const total = a.amount + b.amount; // what if currencies differ?

// After
function money(amount, currency) {
  return Object.freeze({ amount, currency });
}
function addMoney(a, b) {
  if (a.currency !== b.currency) throw new Error('currency mismatch');
  return money(a.amount + b.amount, a.currency);
}
```

Plain functions plus frozen objects are usually enough; a class is fine if the project already uses classes for domain objects.

**Guardrail:** don't wrap every string. Do it when the logic around the value is already repeated, or the value has rules (format, units) that are easy to get wrong.

Storing money as integer minor units (cents, đồng) instead of floats is a data/behavior change, not a refactoring — raise it with the user.

Link: https://refactoring.guru/replace-data-value-with-object

---

## Replace Array with Object

**Use when** an array is used as a record, with positions carrying meaning: `[lat, lng]`, `[status, body]`, `row[3]`.

```js
// Before
function getPrice(product) { return [product.price, product.currency]; }
const result = getPrice(p);
res.json({ amount: result[0], currency: result[1] });

// After
function getPrice(product) { return { amount: product.price, currency: product.currency }; }
const { amount, currency } = getPrice(p);
res.json({ amount, currency });
```

Tuples that are a well-known convention (`Object.entries` pairs, `[err, result]` in a codebase that uses it consistently) can stay.

Link: https://refactoring.guru/replace-array-with-object

---

## Replace Type Code with Class

**Use when** a set of string or number codes (`'admin' | 'member' | 'guest'`) is passed around and typos or invalid values slip through, but the code doesn't change *behavior* — it's only stored and compared.

**JS idiom:** a frozen enum object plus a validation function used at the boundary:

```js
const Role = Object.freeze({ ADMIN: 'admin', MEMBER: 'member', GUEST: 'guest' });
const isRole = (value) => Object.values(Role).includes(value);

// validation middleware rejects anything that isn't a Role
```

If the project uses TypeScript or JSDoc, a union type (`/** @typedef {'admin'|'member'|'guest'} Role */`) adds editor checking at no runtime cost.

Link: https://refactoring.guru/replace-type-code-with-class

---

## Replace Type Code with Subclasses

**Use when** the code affects behavior *and* never changes for a given object, and several functions branch on it.

**In Node, prefer the lighter State/Strategy form below** (a lookup of behavior by code). Use real subclasses only if the project is already class-based and each variant has several methods and its own data.

```js
class Notification { send() { throw new Error('not implemented'); } }
class EmailNotification extends Notification { send() { /* ... */ } }
class SmsNotification extends Notification { send() { /* ... */ } }

const notificationTypes = { email: EmailNotification, sms: SmsNotification };
function createNotification(type, data) {
  if (!Object.hasOwn(notificationTypes, type)) throw new Error(`unknown notification type ${type}`);
  return new notificationTypes[type](data);
}
```

Link: https://refactoring.guru/replace-type-code-with-subclasses

---

## Replace Type Code with State/Strategy

**Use when** the code affects behavior and either changes over the object's life (an order moving from `PENDING` to `PAID`) or the project doesn't use subclassing.

```js
// Each state describes what is allowed from it
const orderStates = {
  PENDING:   { canCancel: () => true,  next: ['PAID', 'CANCELLED'] },
  PAID:      { canCancel: (o) => !o.shippedAt, next: ['SHIPPED', 'REFUNDED'] },
  SHIPPED:   { canCancel: () => false, next: ['DELIVERED'] },
  DELIVERED: { canCancel: () => false, next: [] },
};

function transition(order, to) {
  const state = orderStates[order.status];
  if (!state?.next.includes(to)) throw new ConflictError(`cannot go from ${order.status} to ${to}`);
  return { ...order, status: to };
}
```

This replaces `if (order.status === ...)` checks scattered across handlers with one table that describes the rules. See also State and Strategy in `design-patterns.md`.

Link: https://refactoring.guru/replace-type-code-with-state-strategy

---

## Replace Subclass with Fields

**Use when** subclasses differ only in the constant values they return — no real behavior differences.

```js
// Before
class FreePlan { get maxProjects() { return 3; } get price() { return 0; } }
class ProPlan  { get maxProjects() { return 50; } get price() { return 19; } }

// After — data, not classes
const PLANS = Object.freeze({
  free: { maxProjects: 3, price: 0 },
  pro:  { maxProjects: 50, price: 19 },
});
```

Link: https://refactoring.guru/replace-subclass-with-fields

---

## Change Unidirectional Association to Bidirectional

**Use when** two objects need to reach each other, but only one side holds the link (e.g. code often needs to go from a team to its members *and* from a member to their team).

In a backend, links usually live in the database, so this mostly means adding a query or index for the reverse direction (e.g. `findMembersByTeam(teamId)`), or an ORM association — not storing object references both ways in memory.

**Cost:** two-way links must be kept in sync, and in-memory two-way references create cycles that break `JSON.stringify`. Add the reverse direction only when code actually needs it.

Link: https://refactoring.guru/change-unidirectional-association-to-bidirectional

---

## Change Bidirectional Association to Unidirectional

**Use when** two objects reference each other but one direction is no longer used — the extra link adds coupling and sync bugs.

Typical Node signs: two modules that `require` each other (circular dependency, which can cause one of them to see a half-initialized `module.exports`), or objects with parent/child references that complicate serialization. Remove the unused direction and look it up when needed.

Link: https://refactoring.guru/change-bidirectional-association-to-unidirectional

---

## Duplicate Observed Data

**Use when** domain data is stored inside a UI or presentation component and needs to be shared or tested separately.

This is mostly a front-end refactoring. The backend equivalent is **business data and rules mixed into HTTP-layer code**: response formatting, view templates or route handlers holding domain rules. The fix is the same idea — move domain data and logic into a domain/service module, and let the HTTP layer only map it to responses (see "Fat route handler → service layer" in `express-patterns.md`).

Link: https://refactoring.guru/duplicate-observed-data
