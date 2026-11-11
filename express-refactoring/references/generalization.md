# Dealing with Generalization

Refactorings that move behavior up and down inheritance hierarchies, or replace inheritance altogether.

**Node.js context first.** Most Express codebases are built from functions and modules, not class hierarchies, and that is idiomatic. Prefer **composition** (functions calling functions, objects holding other objects) over inheritance. Use the refactorings below when the project already has class hierarchies (custom error classes, ORM models, SDK clients, service classes) — and read "Replace Inheritance with Delegation" before creating a new hierarchy.

Many of these also have a module-level equivalent, noted under each entry.

## Contents
- Pull Up Method
- Pull Up Field
- Pull Up Constructor Body
- Push Down Method
- Push Down Field
- Extract Superclass
- Extract Subclass
- Extract Interface
- Collapse Hierarchy
- Form Template Method
- Replace Inheritance with Delegation
- Replace Delegation with Inheritance

---

## Pull Up Method

**Use when** two subclasses have the same method. Move it to the parent class.

**Module equivalent:** two modules have the same helper → move it to a shared module both import (Extract Method + Move Method). Apply the rule of three, and check the copies are truly identical — small differences are often deliberate.

```js
// Before
class StripeGateway extends PaymentGateway { toMinorUnits(amount) { return Math.round(amount * 100); } }
class PaypalGateway extends PaymentGateway { toMinorUnits(amount) { return Math.round(amount * 100); } }

// After
class PaymentGateway { toMinorUnits(amount) { return Math.round(amount * 100); } }
```

Link: https://refactoring.guru/pull-up-method

---

## Pull Up Field

**Use when** subclasses each declare the same property. Declare it once in the parent.

Link: https://refactoring.guru/pull-up-field

---

## Pull Up Constructor Body

**Use when** subclass constructors repeat the same setup. Move it into the parent constructor and call `super(...)`.

```js
// After
class AppError extends Error {
  constructor(message, status) {
    super(message);
    this.name = this.constructor.name;
    this.status = status;
  }
}
class NotFoundError extends AppError { constructor(msg = 'not found') { super(msg, 404); } }
class ConflictError extends AppError { constructor(msg = 'conflict') { super(msg, 409); } }
```

Custom error classes are the most common — and most useful — class hierarchy in Express apps.

Link: https://refactoring.guru/pull-up-constructor-body

---

## Push Down Method

**Use when** a method in the parent class is only relevant to one subclass. Move it down so other subclasses don't inherit something meaningless to them.

**Module equivalent:** a shared `utils.js` function used by one module only → move it into that module.

Link: https://refactoring.guru/push-down-method

---

## Push Down Field

**Use when** a property in the parent is only used by some subclasses. Move it to those subclasses.

Link: https://refactoring.guru/push-down-field

---

## Extract Superclass

**Use when** two classes share several methods and fields. Create a parent class for the shared part.

**Before reaching for it in Node, consider:** would a shared module of functions, or composing a shared helper object, do the same job without coupling the classes? Extract a superclass only when the classes are genuinely the same *kind* of thing (e.g. several payment gateways with a common lifecycle).

Link: https://refactoring.guru/extract-superclass

---

## Extract Subclass

**Use when** some behavior of a class is used only in certain cases (usually guarded by a type flag). Move that behavior into a subclass.

**In Node, usually prefer** Replace Type Code with State/Strategy (`organizing-data.md`): a lookup of behavior by type is lighter than a subclass per variant.

Link: https://refactoring.guru/extract-subclass

---

## Extract Interface

**Use when** several modules or classes are used through the same set of functions, and you want to make that contract explicit — e.g. several storage backends (`put`, `get`, `delete`) or notification channels (`send`).

JavaScript has no `interface` keyword; it relies on duck typing. Make the contract explicit with:
- **JSDoc typedef** (no build step):
  ```js
  /**
   * @typedef {Object} Storage
   * @property {(key: string, data: Buffer) => Promise<void>} put
   * @property {(key: string) => Promise<Buffer|null>} get
   * @property {(key: string) => Promise<void>} delete
   */
  ```
- **TypeScript `interface`** if the project uses TypeScript.
- **A shared contract test** run against every implementation — the most reliable option, because it checks behavior and not only names.

Don't create an interface with a single implementation "for future flexibility" — that's Speculative Generality.

Link: https://refactoring.guru/extract-interface

---

## Collapse Hierarchy

**Use when** a subclass and its parent are practically the same — the subclass adds nothing, or the base class has only one child. Merge them.

This is a common fix for over-engineered code: an abstract `BaseService` with one `UserService` child, or a `BaseRepository` wrapping one model.

Link: https://refactoring.guru/collapse-hierarchy

---

## Form Template Method

**Use when** subclasses (or sibling functions) perform the same steps in the same order, but each step differs in detail. Put the fixed sequence in one place and let the variants supply only the differing steps.

**JS idiom:** a higher-order function is usually lighter than a parent class:

```js
// The fixed sequence: validate → transform → save → notify
function createImporter({ validate, transform }) {
  return async function importRecords(records, { save, notify }) {
    const valid = records.filter(validate);
    const rows = valid.map(transform);
    await save(rows);
    await notify({ imported: rows.length, skipped: records.length - valid.length });
  };
}

const importCustomers = createImporter({ validate: isValidCustomer, transform: toCustomerRow });
const importProducts = createImporter({ validate: isValidProduct, transform: toProductRow });
```

See also Template Method in `design-patterns.md`.

Link: https://refactoring.guru/form-template-method

---

## Replace Inheritance with Delegation

**Use when** a class extends another but uses only part of it, or inherits methods that make no sense for it (Refused Bequest). Hold an instance of the other class and forward only what you need.

```js
// Before — a cache that "is a" Map, so callers can also call .clear(), .entries() ... on it
class SessionCache extends Map {
  getSession(id) { return this.get(id); }
}

// After — a cache that "has a" Map, and exposes only what it means to
class SessionCache {
  #store = new Map();
  getSession(id) { return this.#store.get(id); }
  saveSession(id, session) { this.#store.set(id, session); }
}
```

Extending built-ins (`Map`, `Array`, `EventEmitter`) and framework classes is the most common place this is worth doing — every inherited method becomes part of your public API.

Link: https://refactoring.guru/replace-inheritance-with-delegation

---

## Replace Delegation with Inheritance

**Use when** a class forwards almost every call to one delegate, with no logic of its own — and it genuinely *is* that kind of thing.

In Node, the more common fix for pure forwarding is **Remove Middle Man** (`moving-features.md`): let callers use the delegate directly. Choose inheritance only when the subclass will add real behavior and you want the whole parent API.

Link: https://refactoring.guru/replace-delegation-with-inheritance
