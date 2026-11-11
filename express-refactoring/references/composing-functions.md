# Composing Methods (Functions)

Refactorings that shape individual functions: how big they are, what they name, and how values flow through them. This covers the catalog's "Composing Methods" group; in JavaScript, read *method* as *function*.

## Contents
- Extract Method
- Inline Method
- Extract Variable
- Inline Temp
- Replace Temp with Query
- Split Temporary Variable
- Remove Assignments to Parameters
- Replace Method with Method Object
- Substitute Algorithm

---

## Extract Method (Extract Function)

**Use when** a block of code can be described by a short name, especially if there's a comment above it explaining what it does — the comment is usually the function name waiting to happen.

**Steps**
1. Create a new function named after *what* the block does, not *how*.
2. Move the block in; pass in variables it reads, return the value it produces.
3. If the block modifies more than one outer variable, it is probably two functions — or not ready to extract.
4. Replace the original block with the call. Run tests.

```js
// Before
async function createOrder(req, res) {
  const { items, couponCode } = req.body;

  // calculate total
  let total = 0;
  for (const item of items) {
    total += item.price * item.quantity;
  }
  if (couponCode === 'WELCOME10') {
    total = total * 0.9;
  }

  const order = await Order.create({ items, total });
  res.status(201).json(order);
}

// After
const COUPON_DISCOUNTS = { WELCOME10: 0.1 };

function applyCoupon(amount, couponCode) {
  const discount = Object.hasOwn(COUPON_DISCOUNTS, couponCode) ? COUPON_DISCOUNTS[couponCode] : 0;
  return amount * (1 - discount);
}

function calculateTotal(items, couponCode) {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  return applyCoupon(subtotal, couponCode);
}

async function createOrder(req, res) {
  const { items, couponCode } = req.body;
  const order = await Order.create({ items, total: calculateTotal(items, couponCode) });
  res.status(201).json(order);
}
```

**Don't** extract a block that is already clearer inline, or one that is used once and needs a long name to explain it.

Link: https://refactoring.guru/extract-method

---

## Inline Method (Inline Function)

**Use when** a function's body says the same thing as its name, or when a chain of tiny delegating functions makes the reader jump around for no gain. Also useful as a first step before re-extracting along better boundaries.

```js
// Before
function isAdult(user) { return user.age >= 18; }
function canRegister(user) { return isAdult(user); }

// After — canRegister was pure indirection
function isAdult(user) { return user.age >= 18; }
// callers use isAdult(user) directly
```

**Check first:** is the function exported, overridden, or used as a callback reference in many places? Inlining an exported function is an API change.

Link: https://refactoring.guru/inline-method

---

## Extract Variable

**Use when** an expression is hard to read at a glance. Giving parts of it names documents intent without a function.

```js
// Before
if (req.user && req.user.roles.includes('admin') && Date.now() - req.user.lastLogin < 15 * 60 * 1000) {
  ...
}

// After
const FIFTEEN_MINUTES_MS = 15 * 60 * 1000;
const isAdmin = req.user?.roles.includes('admin');
const hasRecentLogin = Date.now() - req.user?.lastLogin < FIFTEEN_MINUTES_MS;
if (isAdmin && hasRecentLogin) {
  ...
}
```

If the same expression appears in several functions, prefer Extract Method instead.

Link: https://refactoring.guru/extract-variable

---

## Inline Temp (Inline Variable)

**Use when** a variable's name adds nothing over the expression, or the variable gets in the way of another refactoring.

```js
// Before
const userId = req.params.id;
return User.findById(userId);

// After (when the function is short and the meaning is obvious)
return User.findById(req.params.id);
```

Keep the variable if it is used several times, if the expression is expensive, or if the name genuinely explains something.

Link: https://refactoring.guru/inline-temp

---

## Replace Temp with Query

**Use when** a temp variable stores a derived value that several functions need. Turning it into a small function lets it be reused and makes Extract Method easier.

```js
// Before
const basePrice = product.price * quantity;
const discount = basePrice > 1000 ? basePrice * 0.05 : 0;

// After
const basePrice = (product, quantity) => product.price * quantity;
const discountFor = (price) => (price > 1000 ? price * 0.05 : 0);
```

Skip this if the computation is expensive (e.g. a DB call) — then caching in a variable is the right choice.

Link: https://refactoring.guru/replace-temp-with-query

---

## Split Temporary Variable

**Use when** one `let` variable is reassigned to hold different meanings over the function's life. Each meaning deserves its own `const`.

```js
// Before
let temp = req.body.width * req.body.height;
console.log(`area: ${temp}`);
temp = 2 * (req.body.width + req.body.height);
console.log(`perimeter: ${temp}`);

// After
const area = req.body.width * req.body.height;
const perimeter = 2 * (req.body.width + req.body.height);
```

Accumulators (`sum += x` in a loop) are not this smell.

Link: https://refactoring.guru/split-temporary-variable

---

## Remove Assignments to Parameters

**Use when** a function reassigns its parameter. In JS this doesn't affect the caller for primitives, but mutating an object parameter *does* — which is a frequent source of bugs (e.g. mutating `req.body` or a shared config object).

```js
// Before
function normalizeQuery(query) {
  query.limit = Math.min(query.limit || 20, 100);
  query.page = query.page || 1;
  return query;
}

// After — return a new object; caller's data is untouched
function normalizeQuery(query) {
  return {
    ...query,
    limit: Math.min(query.limit || 20, 100),
    page: query.page || 1,
  };
}
```

Keep the same operators (`||` here) when refactoring. Switching `||` to `??` changes how `0` and `''` are treated — that is a behavior change, so do it separately and only if it's intended.

Link: https://refactoring.guru/remove-assignments-to-parameters

---

## Replace Method with Method Object

**Use when** a long function has so many local variables that Extract Method is impractical — every extracted piece would need five parameters and return three values.

**JS idiom:** turn the function into a small class (or a closure) whose local variables become properties, then split the work into methods that share them. Create a **new instance per call** — never keep per-request state on a long-lived, shared object.

```js
// Before — one 120-line function with a dozen shared locals
async function generateInvoice(order, customer, taxRules) {
  let subtotal = 0, taxTotal = 0, discountTotal = 0, lines = [];
  // ...long logic reading and writing all of them...
}

// After
class InvoiceCalculation {
  constructor(order, customer, taxRules) {
    Object.assign(this, { order, customer, taxRules });
    this.lines = [];
    this.subtotal = 0;
    this.taxTotal = 0;
  }
  run() {
    this.buildLines();
    this.applyDiscounts();
    this.applyTaxes();
    return { lines: this.lines, subtotal: this.subtotal, tax: this.taxTotal };
  }
  buildLines() { /* ... */ }
  applyDiscounts() { /* ... */ }
  applyTaxes() { /* ... */ }
}

const generateInvoice = (order, customer, taxRules) =>
  new InvoiceCalculation(order, customer, taxRules).run();
```

The exported `generateInvoice` keeps its signature, so callers don't change. Once split, you may find the pieces can become plain functions after all.

Link: https://refactoring.guru/replace-method-with-method-object

---

## Substitute Algorithm

**Use when** there's a clearer or built-in way to do what a block does — a hand-written loop that `Array.prototype` methods, `Set`, `Map`, or a well-established library already provide.

```js
// Before
function uniqueEmails(users) {
  const result = [];
  for (let i = 0; i < users.length; i++) {
    let found = false;
    for (let j = 0; j < result.length; j++) {
      if (result[j] === users[i].email) found = true;
    }
    if (!found) result.push(users[i].email);
  }
  return result;
}

// After
const uniqueEmails = (users) => [...new Set(users.map((u) => u.email))];
```

**Check the edge cases carefully** — this is where substitutions silently change behavior: ordering, duplicates, `undefined`/`null` entries, case sensitivity, and performance on large inputs. Write tests that pin these down before swapping.

Link: https://refactoring.guru/substitute-algorithm
