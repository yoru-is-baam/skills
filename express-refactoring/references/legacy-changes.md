# Changing Legacy Code with Minimal Impact

Techniques for adding or changing behavior in existing code without rewriting it. Use them when the code is messy, poorly tested, widely used, or simply outside the scope of the task.

The idea behind all of them: **put the new behavior in new, clean, tested code, and connect it to the old code at one small, safe point.** The old code is not made worse, and the new code is not made messy by living inside it.

## Contents
- Choosing a technique
- Sprout Function
- Sprout Module
- Wrap Function
- Optional parameter with a default
- Parallel change (expand → migrate → contract)
- Adding behavior to routes: route-level middleware
- When sprouting stops being enough

---

## Choosing a technique

| Situation | Technique |
|---|---|
| Adding a step or rule inside an existing messy function | Sprout Function |
| The new logic is large or has its own dependencies | Sprout Module |
| New behavior must happen before/after the old function, for every caller | Wrap Function |
| A function needs one extra input, and old callers should keep working | Optional parameter with a default |
| A signature, return shape or API response must change | Parallel change |
| A route or group of routes needs a new check or side effect | Route-level middleware |

---

## Sprout Function

**When:** you need to add logic in the middle of a long, untested function, and writing tests for the whole function is out of scope.

**Steps**
1. Identify the exact point in the old function where the new behavior belongs, and the values it needs there.
2. Write the new logic as a separate, pure-as-possible function with its own unit tests. It takes plain values and returns a value; it doesn't read `req` or globals.
3. Add one line in the old function that calls it.
4. Run the existing tests (or characterization tests around the route) to confirm nothing else changed.

```js
// New, clean, unit-tested — services/loyalty.js
const LOYALTY_DISCOUNT = Object.freeze({ GOLD: 0.05, PLATINUM: 0.1 });

function applyLoyaltyDiscount(total, customerTier) {
  const rate = LOYALTY_DISCOUNT[customerTier] ?? 0;
  return total * (1 - rate);
}
module.exports = { applyLoyaltyDiscount };

// Legacy handler — the only change is one line
router.post('/checkout', async (req, res) => {
  // ...150 untouched lines that compute `total` and load `customer`...
  total = applyLoyaltyDiscount(total, customer.tier);   // ← sprouted
  // ...untouched lines that save the order and respond...
});
```

**Why it's safe:** the new rule is fully tested on its own, and the old function's structure didn't change, so its existing behavior is preserved except at the one call site you added.

---

## Sprout Module

**When:** the new behavior is big enough to need several functions, or its own dependencies (an external API client, a queue).

Create a new module (e.g. `services/invoiceExport.js`) with its own tests, designed cleanly from the start. The old code imports it and makes one call. This also stops the new feature from adding even more length and dependencies to an already large file.

---

## Wrap Function

**When:** every call to an existing function should now also do something new (logging, metrics, caching, an audit record), and you don't want to edit the function body or every caller.

```js
// Before — services/userService.js
async function deleteUser(id) {
  // ...legacy body, untouched...
}
module.exports = { deleteUser };

// After
async function deleteUserRecord(id) {      // renamed, body untouched
  // ...legacy body, untouched...
}

async function deleteUser(id, { actorId } = {}) {   // same name, same default behavior
  await deleteUserRecord(id);
  await auditLog.record({ action: 'user.deleted', targetId: id, actorId });
}

module.exports = { deleteUser };
```

Callers don't change. The new behavior lives in a small function you can test by stubbing `deleteUserRecord` and `auditLog`.

**Check:** does the order matter (before vs after the original call)? What should happen if the new step fails — should the original action still count? Decide this explicitly and test it; it is a behavior decision, so mention it to the user.

---

## Optional parameter with a default

**When:** a function needs one more input, and existing callers must keep their current behavior.

```js
// Before
function listOrders(userId) { ... }

// After — old calls listOrders(42) behave exactly as before
function listOrders(userId, { includeArchived = false } = {}) { ... }
```

- Choose the default so that it reproduces the **old** behavior exactly.
- Prefer an options object over a new positional parameter, so the next addition doesn't create a long parameter list.
- If you find yourself adding a third or fourth option that changes the function's whole behavior, consider Replace Parameter with Explicit Methods instead (`simplifying-function-calls.md`).

---

## Parallel change (expand → migrate → contract)

**When:** something really must change shape — a function's signature, a module's export, or an API response — and it has many callers or external clients.

1. **Expand:** add the new version beside the old one. Both work.
2. **Migrate:** move callers to the new version, one at a time or in small batches, each with tests passing.
3. **Contract:** when nothing uses the old version, remove it — usually in a separate, later change.

For internal functions:

```js
// Expand: new signature beside the old one
async function findUsers({ orgId, status, limit }) { ... }

/** @deprecated use findUsers({ orgId, status, limit }) */
async function getUsers(orgId, status, limit) {
  return findUsers({ orgId, status, limit });
}
```

For HTTP responses, don't change an existing endpoint's response shape in place — clients outside this repo may depend on it. Options, in order of preference:
- Add new fields without removing or renaming old ones (additive change).
- Add a new endpoint or version (`/v2/orders`) and keep the old one until clients migrate.

Within one task, usually only the **expand** step (and maybe migrating the callers you touch) is in scope. Leave the remaining migration and the contract step as follow-ups in your report.

---

## Adding behavior to routes: route-level middleware

**When:** a few routes need a new check or side effect.

Don't edit a shared, global middleware (one registered with `app.use`) for a need that only some routes have — every route in the app would be affected. Write a new middleware and mount it only where it's needed:

```js
// New, clean, tested
const requireVerifiedEmail = (req, res, next) =>
  req.user?.emailVerified ? next() : next(new ForbiddenError('email not verified'));

// Only the routes that need it
router.post('/payouts', requireVerifiedEmail, createPayout);
```

Middleware order is behavior: place the new middleware after authentication (so `req.user` exists) and before the handler.

---

## When sprouting stops being enough

Sprouting keeps each change safe, but if every task adds another sprout to the same 500-line handler, the handler keeps growing. Signals that it is time for a real refactoring:

- The same function or file is touched in most recent tasks.
- New sprouts keep needing the same values pulled out of the old code.
- Bugs keep appearing in that area.

When you notice this, don't start a large refactoring on your own. Say so in your report, with a short, concrete proposal (for example: "add characterization tests for POST /checkout, then extract pricing, payment and notification into a service, one step at a time"), and let the user schedule it.
