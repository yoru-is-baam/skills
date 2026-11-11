# Moving Features between Objects (Modules)

Refactorings that decide *where* code lives. In a Node backend, the "objects" are usually modules (files), and sometimes classes. Getting placement right is what keeps a change in one place instead of ten.

## Contents
- Move Method
- Move Field
- Extract Class
- Inline Class
- Hide Delegate
- Remove Middle Man
- Introduce Foreign Method
- Introduce Local Extension

---

## Move Method (Move Function)

**Use when** a function uses another module's data more than its own (Feature Envy), or when related functions are scattered so that one change needs edits in many files (Shotgun Surgery).

```js
// Before — routes/orders.js knows the details of an order's state
const canBeCancelled = (order) =>
  order.status === 'PENDING' && Date.now() - order.createdAt < CANCEL_WINDOW_MS;

// After — the rule lives in models/order.js beside the data it describes
// models/order.js
function canBeCancelled(order) {
  return order.status === OrderStatus.PENDING && Date.now() - order.createdAt < CANCEL_WINDOW_MS;
}
module.exports = { canBeCancelled };
```

**Steps:** copy the function to its new home, make the old one delegate to it, move callers over, then delete the old one. In the minimal-impact setting, the delegating step may be as far as this task goes.

**Watch for:** circular `require`s. If the destination module already imports the source, moving can create a cycle — check the import graph first.

Link: https://refactoring.guru/move-method

---

## Move Field

**Use when** a piece of data (a property, a constant, a module-level variable) is used more by another module than by the one that defines it.

Typical Express cases:
- A constant like `MAX_PAGE_SIZE` defined in one route file but used by several → move it to a constants or config module.
- A cached value kept in a route file but managed by a service → move it into that service.

Update every reader in the same change; a field left in two places quickly drifts.

Link: https://refactoring.guru/move-field

---

## Extract Class (Extract Module)

**Use when** one module or class does two jobs that change for different reasons (Divergent Change, Large Class).

```js
// Before — services/userService.js mixes accounts and notification preferences
// createUser, updateUser, deleteUser, getNotificationSettings, muteChannel, unmuteChannel ...

// After
// services/userService.js           → createUser, updateUser, deleteUser
// services/notificationSettings.js  → getNotificationSettings, muteChannel, unmuteChannel
```

To keep impact low, the old module can re-export the moved functions for a while (`module.exports = { ...accountFns, ...require('./notificationSettings') }`) so existing imports keep working — then migrate imports as a parallel change (`legacy-changes.md`).

Link: https://refactoring.guru/extract-class

---

## Inline Class (Inline Module)

**Use when** a module or class does too little to justify its existence (Lazy Class) — often left behind after other refactorings, or created speculatively.

Move its few functions into the module that uses them, update imports, delete the file. Check first that nothing else imports it, including tests, scripts and other services.

Link: https://refactoring.guru/inline-class

---

## Hide Delegate

**Use when** callers navigate through one object to reach another (`a.b.c.doSomething()`), so they break whenever the object structure changes (Message Chains).

```js
// Before — every route reaches into the nested structure
const maxSeats = req.user.account.subscription.plan.limits.maxSeats;

// After — one function owns the navigation
// services/subscription.js
function seatLimitFor(user) {
  return user.account.subscription.plan.limits.maxSeats;
}
const maxSeats = seatLimitFor(req.user);
```

If the shape of `account` changes, only `seatLimitFor` needs updating.

Link: https://refactoring.guru/hide-delegate

---

## Remove Middle Man

**Use when** a module only forwards calls without adding anything (Middle Man) — the opposite of Hide Delegate.

```js
// Before — services/productService.js
const findById = (id) => Product.findById(id);
const findAll = () => Product.find();
const create = (data) => Product.create(data);

// After — routes call the model directly, or the service is kept only for
// functions that contain real business rules
```

**Balance:** keep a thin layer when it isolates something likely to change (a third-party API client, an ORM you might replace) or when the team's architecture requires a consistent service layer. Remove it when it isolates nothing.

Link: https://refactoring.guru/remove-middle-man

---

## Introduce Foreign Method

**Use when** a library or built-in object lacks a function you need, and the workaround is repeated in several places.

```js
// Before — repeated in several handlers
const nextBusinessDay = new Date(date);
nextBusinessDay.setDate(date.getDate() + (date.getDay() === 5 ? 3 : 1));

// After — one helper in your own code, taking the library object as a parameter
// lib/dates.js
function nextBusinessDay(date) {
  const result = new Date(date);
  const daysToAdd = date.getDay() === 5 ? 3 : date.getDay() === 6 ? 2 : 1;
  result.setDate(date.getDate() + daysToAdd);
  return result;
}
```

Note: the "after" version also handles Saturday, which the copied snippet didn't. That is a behavior change — make it deliberately, in its own commit, with a test.

**Never** add the function to the built-in prototype (`Date.prototype.nextBusinessDay = ...`). Prototype patching affects every module in the process, including your dependencies.

Link: https://refactoring.guru/introduce-foreign-method

---

## Introduce Local Extension

**Use when** you need several extra functions around a library — enough that scattered foreign methods become messy.

**JS idiom:** create your own wrapper module that imports the library and exposes the extra functions (and, if useful, the original ones). The rest of your code imports the wrapper, not the library.

```js
// lib/http.js — local extension of the HTTP client
const axios = require('axios');
const client = axios.create({ timeout: 5000 });

async function getJson(url, options) {
  const res = await client.get(url, options);
  return res.data;
}

async function getJsonWithRetry(url, { attempts = 3, ...options } = {}) {
  for (let i = 1; ; i++) {
    try {
      return await getJson(url, options);
    } catch (err) {
      if (i >= attempts) throw err;
    }
  }
}

module.exports = { getJson, getJsonWithRetry };
```

A side benefit: it is also an Adapter (`design-patterns.md`), so swapping the library later only touches this file.

Link: https://refactoring.guru/introduce-local-extension
