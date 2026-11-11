# Simplifying Method (Function) Calls

Refactorings that make functions easier to call and understand from the outside: names, parameters, return values and errors.

**Almost everything here changes a function's signature.** Before applying any of these to an existing function, find all callers (`grep -rn "name" src/ test/`). If the function is exported or widely used, apply it as a parallel change — new version beside the old one, migrate callers, remove the old one later (`legacy-changes.md`).

## Contents
- Rename Method
- Add Parameter
- Remove Parameter
- Introduce Parameter Object
- Preserve Whole Object
- Replace Parameter with Method Call
- Parameterize Method
- Replace Parameter with Explicit Methods
- Separate Query from Modifier
- Hide Method
- Remove Setting Method
- Replace Constructor with Factory Method
- Replace Error Code with Exception
- Replace Exception with Test

---

## Rename Method (Rename Function)

**Use when** a name doesn't say what the function does — `process()`, `handle()`, `doStuff()`, `getData()` that also writes to the DB.

Good names in a Node backend say the effect: `findUserByEmail`, `chargeCustomer`, `sendPasswordResetEmail`, `isOrderCancellable` (booleans start with `is`/`has`/`can`). Use the project's existing vocabulary and casing — see `style-and-naming.md`.

For an exported function, keep the old name as a deprecated alias for a while:

```js
function findActiveUsers(orgId) { /* ... */ }
/** @deprecated use findActiveUsers */
const getUsers = findActiveUsers;
module.exports = { findActiveUsers, getUsers };
```

Search for dynamic references too (string keys, `module[name]`, route tables) — editors' rename tools miss these.

Link: https://refactoring.guru/rename-method

---

## Add Parameter

**Use when** a function genuinely needs information from its caller that it doesn't receive.

Prefer adding to an options object with a default that keeps old behavior (see "Optional parameter with a default" in `legacy-changes.md`):

```js
// Before
async function listOrders(userId) { /* ... */ }

// After — existing calls listOrders(42) behave exactly as before
async function listOrders(userId, { status } = {}) { /* ... */ }
```

**Before adding,** check whether the function could get the value itself (Replace Parameter with Method Call), or whether it's the third value in a group that always travels together (Introduce Parameter Object).

Link: https://refactoring.guru/add-parameter

---

## Remove Parameter

**Use when** a parameter is no longer used inside the function, or callers always pass the same value.

Check every caller first; with an exported function, other services or scripts may still pass it. Removing a trailing unused parameter in JS is harmless to callers that still pass it, so the risk is mostly confusion — but removing a *middle* positional parameter shifts the others and breaks every call site.

Link: https://refactoring.guru/remove-parameter

---

## Introduce Parameter Object

**Use when** a function takes 4+ positional parameters, or the same group of values travels together through several functions (Long Parameter List, Data Clumps).

```js
// Before
function createInvite(email, orgId, role, invitedBy, expiresInDays) { ... }
createInvite('a@b.com', 12, 'member', 3, 7);

// After — an options object makes call sites self-describing
function createInvite({ email, orgId, role, invitedBy, expiresInDays = 7 }) { ... }
createInvite({ email: 'a@b.com', orgId: 12, role: 'member', invitedBy: 3 });
```

Link: https://refactoring.guru/introduce-parameter-object

---

## Preserve Whole Object

**Use when** a caller pulls several values out of one object only to pass them separately.

```js
// Before
const allowed = canAccessProject(req.user.id, req.user.orgId, req.user.roles, project);

// After
const allowed = canAccessProject(req.user, project);
```

**Counterpoint:** passing a whole object couples the function to its shape. Don't pass `req` itself into services (see `express-patterns.md`), and don't pass a large object when the function needs one field.

Link: https://refactoring.guru/preserve-whole-object

---

## Replace Parameter with Method Call

**Use when** the caller computes a value only to pass it in, and the function could compute it just as easily itself.

```js
// Before
const discountRate = getDiscountRate(customer);
const price = finalPrice(order, customer, discountRate);

// After
function finalPrice(order, customer) {
  const discountRate = getDiscountRate(customer);
  // ...
}
```

**Don't** do this if it would hide an expensive call (a DB query) inside a function called in a loop, or make the function harder to test. Passing values in is often the better design for pure, testable functions.

Link: https://refactoring.guru/replace-parameter-with-method-call

---

## Parameterize Method

**Use when** several functions do the same thing except for a value.

```js
// Before
const findPendingOrders = () => Order.find({ status: 'PENDING' });
const findPaidOrders = () => Order.find({ status: 'PAID' });
const findShippedOrders = () => Order.find({ status: 'SHIPPED' });

// After
const findOrdersByStatus = (status) => Order.find({ status });
```

This is the opposite of the next refactoring — choose by readability at the call site.

Link: https://refactoring.guru/parameterize-method

---

## Replace Parameter with Explicit Methods

**Use when** a parameter (often a boolean or a mode string) makes one function do clearly different things.

```js
// Before
function getUsers(includeDeleted) { ... }
getUsers(true); // the reader has to look up what `true` means

// After
function getActiveUsers() { ... }
function getAllUsersIncludingDeleted() { ... }
```

If the flag only tweaks one small detail, an options object (`getUsers({ includeDeleted: true })`) is a lighter fix.

Link: https://refactoring.guru/replace-parameter-with-explicit-methods

---

## Separate Query from Modifier

**Use when** a function both returns a value *and* changes state, so callers can't ask a question without causing a side effect.

```js
// Before — calling it to check also consumes a token
async function hasRemainingQuota(userId) {
  const quota = await Quota.findOne({ userId });
  quota.remaining -= 1;
  await quota.save();
  return quota.remaining >= 0;
}

// After
async function getRemainingQuota(userId) {
  const quota = await Quota.findOne({ userId });
  return quota.remaining;
}
async function consumeQuota(userId) {
  await Quota.updateOne({ userId }, { $inc: { remaining: -1 } });
}
```

**Concurrency warning:** in a backend, splitting a check-and-update into two calls can create a race condition (two requests both see quota 1, both consume). When the check and the change must be atomic, keep them in one function and make it one atomic operation (a conditional DB update or a transaction), and name it clearly as a modifier — e.g. `tryConsumeQuota()` returning whether it succeeded.

Link: https://refactoring.guru/separate-query-from-modifier

---

## Hide Method

**Use when** a function is exported but only used inside its own module.

Stop exporting it (or use a `#private` method in a class). A smaller public surface means fewer things that can't be changed later. Check tests first — if tests import the helper directly, consider testing it through the public function instead.

Link: https://refactoring.guru/hide-method

---

## Remove Setting Method

**Use when** a value should be set once at creation and never changed, but a setter (or a writable property) allows changes.

```js
class ApiClient {
  #baseUrl;
  constructor(baseUrl) { this.#baseUrl = baseUrl; }
  get baseUrl() { return this.#baseUrl; }   // read-only from outside
}
```

For plain objects, `Object.freeze` gives the same guarantee.

Link: https://refactoring.guru/remove-setting-method

---

## Replace Constructor with Factory Method

**Use when** creating an object needs more than a constructor can do cleanly: choosing a variant, async setup (constructors can't be `async`), validation that might return an error, or reusing an existing instance.

```js
// Before — async work hidden in a constructor (can't be awaited)
class DbClient {
  constructor(url) { this.ready = connect(url); }
}

// After
class DbClient {
  constructor(connection) { this.connection = connection; }
  static async create(url) {
    const connection = await connect(url);
    return new DbClient(connection);
  }
}
const db = await DbClient.create(config.databaseUrl);
```

In function-based code, a plain factory function (`createDbClient(url)`) is the same idea. See Factory Method in `design-patterns.md`.

Link: https://refactoring.guru/replace-constructor-with-factory-method

---

## Replace Error Code with Exception

**Use when** a function signals failure with a special return value (`null`, `-1`, `false`, `{ error: '...' }`) that callers must remember to check.

```js
// Before
async function withdraw(accountId, amount) {
  const account = await Account.findById(accountId);
  if (account.balance < amount) return -1;
  // ...
}
// every caller: if (result === -1) ...

// After
class InsufficientFundsError extends AppError {
  constructor() { super('insufficient funds', 422); }
}
async function withdraw(accountId, amount) {
  const account = await Account.findById(accountId);
  if (account.balance < amount) throw new InsufficientFundsError();
  // ...
}
```

In Express, typed errors carrying a status code plug straight into the central error handler (`express-patterns.md`).

Returning `null` for "not found" from a `find...` function is a normal convention and doesn't need changing; this refactoring targets *failures*.

Link: https://refactoring.guru/replace-error-code-with-exception

---

## Replace Exception with Test

**Use when** code uses `try/catch` to handle a situation that a simple check could prevent — exceptions used for normal control flow.

```js
// Before — an exception used where a simple check is enough
let item;
try {
  item = cart.items[index].product;
} catch {
  item = null;
}

// After
const item = index < cart.items.length ? cart.items[index].product : null;
```

Keep `try/catch` for genuinely exceptional or unpredictable failures: parsing external input (`JSON.parse` on a request body or file), network calls, file I/O, database errors. A pre-check can't reliably predict those.

Link: https://refactoring.guru/replace-exception-with-test
