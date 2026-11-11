# Simplifying Conditional Expressions

Refactorings for branching logic: nested `if`s, long `switch` statements, flags and scattered null checks.

## Contents
- Replace Nested Conditional with Guard Clauses
- Decompose Conditional
- Consolidate Conditional Expression
- Consolidate Duplicate Conditional Fragments
- Remove Control Flag
- Replace Conditional with Polymorphism (lookup map / strategy)
- Introduce Null Object (and validating at the boundary)
- Introduce Assertion

---

## Replace Nested Conditional with Guard Clauses

**Use when** the main path of a function is buried inside several levels of `if`. Handle the unusual cases first and return early, so the normal case reads straight down.

```js
// Before
async function getProfile(req, res) {
  const user = await User.findById(req.params.id);
  if (user) {
    if (!user.deletedAt) {
      if (user.isPublic || req.user?.id === user.id) {
        res.json(toProfile(user));
      } else {
        res.status(403).json({ error: 'private profile' });
      }
    } else {
      res.status(404).json({ error: 'not found' });
    }
  } else {
    res.status(404).json({ error: 'not found' });
  }
}

// After
async function getProfile(req, res) {
  const user = await User.findById(req.params.id);
  if (!user || user.deletedAt) return res.status(404).json({ error: 'not found' });

  const isOwner = req.user?.id === user.id;
  if (!user.isPublic && !isOwner) return res.status(403).json({ error: 'private profile' });

  res.json(toProfile(user));
}
```

In Express, a forgotten `return` after `res.status(...).json(...)` leads to "Cannot set headers after they are sent". Always `return` from a guard clause that sends a response.

Link: https://refactoring.guru/replace-nested-conditional-with-guard-clauses

---

## Decompose Conditional

**Use when** a condition or its branches are long enough that you have to think to understand them. Extract the condition and each branch into functions or variables whose names state the intent.

```js
// Before
if (order.createdAt < Date.now() - 30 * 86400000 && order.status !== 'REFUNDED' && !order.items.some(i => i.digital)) {
  ...
}

// After
const REFUND_WINDOW_MS = 30 * 24 * 60 * 60 * 1000;
const isOutsideRefundWindow = (order) => order.createdAt < Date.now() - REFUND_WINDOW_MS;
const hasDigitalItems = (order) => order.items.some((i) => i.digital);

if (isOutsideRefundWindow(order) && order.status !== OrderStatus.REFUNDED && !hasDigitalItems(order)) {
  ...
}
```

Link: https://refactoring.guru/decompose-conditional

---

## Consolidate Conditional Expression

**Use when** several separate checks lead to the same result. Combine them into one condition with a name.

```js
// Before
if (!req.body.email) return res.status(400).json({ error: 'invalid input' });
if (!req.body.password) return res.status(400).json({ error: 'invalid input' });
if (req.body.password.length < 8) return res.status(400).json({ error: 'invalid input' });

// After
const isValidSignup = ({ email, password }) => Boolean(email) && password?.length >= 8;
if (!isValidSignup(req.body)) return res.status(400).json({ error: 'invalid input' });
```

If the checks return *different* messages, they are not duplicates — leave them separate, or move them into a validation schema.

Link: https://refactoring.guru/consolidate-conditional-expression

---

## Consolidate Duplicate Conditional Fragments

**Use when** the same code appears in every branch of a conditional. Move it out, before or after the `if`.

```js
// Before
if (isNewCustomer) {
  total = applyWelcomeDiscount(total);
  await Order.create({ items, total });
  logger.info('order created');
} else {
  await Order.create({ items, total });
  logger.info('order created');
}

// After
if (isNewCustomer) total = applyWelcomeDiscount(total);
await Order.create({ items, total });
logger.info('order created');
```

Check that the fragments are truly identical and that moving them doesn't change order relative to other code in the branch (e.g. an `await` that must happen before a check).

Link: https://refactoring.guru/consolidate-duplicate-conditional-fragments

---

## Remove Control Flag

**Use when** a boolean variable controls a loop or a sequence of checks (`let found = false; ... if (!found) ...`). Use `break`, `continue`, `return` or a built-in array method instead.

```js
// Before
let hasBlockedItem = false;
for (const item of cart.items) {
  if (!hasBlockedItem) {
    if (blockedSkus.has(item.sku)) hasBlockedItem = true;
  }
}
if (hasBlockedItem) return res.status(422).json({ error: 'cart contains blocked items' });

// After
const hasBlockedItem = cart.items.some((item) => blockedSkus.has(item.sku));
if (hasBlockedItem) return res.status(422).json({ error: 'cart contains blocked items' });
```

Careful with `await` inside loops: `.some()` and `.forEach()` don't wait for async callbacks. For async checks, keep a `for...of` loop and use `return` or `break` instead of a flag.

Link: https://refactoring.guru/remove-control-flag

---

## Replace Conditional with Polymorphism (lookup map / strategy)

**Use when** a `switch` or `if/else if` chain picks behavior by a type or status string, especially when the same switch appears in more than one place.

In JavaScript, a plain object or `Map` of functions is usually the idiomatic replacement. Classes with polymorphism are only worth it when each variant carries real state and several related behaviors.

```js
// Before
function shippingCost(order) {
  switch (order.shippingMethod) {
    case 'standard': return 5;
    case 'express': return 15;
    case 'overnight': return order.total > 200 ? 20 : 35;
    default: throw new Error(`unknown method ${order.shippingMethod}`);
  }
}

// After
const shippingCalculators = {
  standard: () => 5,
  express: () => 15,
  overnight: (order) => (order.total > 200 ? 20 : 35),
};

function shippingCost(order) {
  if (!Object.hasOwn(shippingCalculators, order.shippingMethod)) {
    throw new Error(`unknown method ${order.shippingMethod}`);
  }
  return shippingCalculators[order.shippingMethod](order);
}
```

Keep the "unknown value" error. Use `Object.hasOwn` (or a `Map`) for keys that come from a request: a plain lookup like `shippingCalculators['constructor']` would otherwise find a function inherited from `Object.prototype`.

**Guardrail:** a single 3-case `switch` used in one place is fine. Replace it when it repeats or keeps growing. See also Strategy in `design-patterns.md`.

Link: https://refactoring.guru/replace-conditional-with-polymorphism

---

## Introduce Null Object (and validating at the boundary)

**Use when** the same `if (x == null)` checks, or `x?.y ?? fallback` chains, are scattered through the code for a value that is often missing.

**Option 1 — a default ("null") object** that behaves safely when there's nothing there:

```js
// A guest user that answers the same questions as a real one
const GUEST_USER = Object.freeze({
  id: null,
  roles: [],
  isGuest: true,
  can: () => false,
});

// authentication middleware
req.user = authenticatedUser ?? GUEST_USER;

// handlers no longer need: if (req.user && req.user.can('edit')) ...
if (!req.user.can('edit')) return res.status(403).json({ error: 'forbidden' });
```

**Option 2 — normalize once at the boundary.** Validate and fill defaults where data enters the system (validation middleware, a mapping function from DB rows, the config module). After that, inner code can trust the shape and drop defensive checks.

```js
const EMPTY_ADDRESS = Object.freeze({ street: '', city: 'Unknown' });
const toUser = (row) => ({ ...row, address: { ...EMPTY_ADDRESS, ...row.address } });
```

**Careful with optional chaining:** sprinkling `?.` can hide bugs by turning "this should never be missing" into a silent `undefined`. If a value is required, fail loudly at the boundary instead.

Link: https://refactoring.guru/introduce-null-object

---

## Introduce Assertion

**Use when** code only works if some condition holds, and that assumption is currently implicit (or in a comment).

```js
const assert = require('node:assert/strict');

function applyRefund(order, amount) {
  assert.ok(amount > 0, 'refund amount must be positive');
  assert.ok(amount <= order.total, 'refund cannot exceed order total');
  // ...
}
```

- Assertions are for **programmer errors** — conditions that should be impossible if the code is correct. They should crash loudly in tests and surface as 500s.
- For **bad user input**, don't assert. Validate and return a 4xx with a clear message.
- Don't assert things already guaranteed by validation middleware, unless the function is also called from places that skip it.

Link: https://refactoring.guru/introduce-assertion
