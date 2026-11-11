# Express Patterns

Smells and refactorings specific to Express.js backends. These usually matter more than the generic catalog, because they decide how easy the whole service is to change and test.

## Contents
- Fat route handler → service layer
- Repeated logic across routes → middleware
- try/catch boilerplate → central error handling
- Scattered `process.env` → config module
- Callback pyramids / mixed promise styles → async/await
- `req`/`res` leaking into services
- Response shape consistency

**First, check the Express version** in `package.json`. Express 5 forwards rejected promises from async handlers to the error handler on its own; Express 4 does not. Several patterns below depend on this.

---

## Fat route handler → service layer

**Smell:** one handler parses input, validates, runs business rules, talks to the database and formats the response. It can only be tested through HTTP.

**Refactoring:** Extract Function for the business logic, then Move Function into a service module. The handler keeps only HTTP concerns: read input, call the service, map the result to a status code and body.

```js
// Before — routes/orders.js
router.post('/orders', async (req, res, next) => {
  try {
    if (!Array.isArray(req.body.items) || req.body.items.length === 0) {
      return res.status(400).json({ error: 'items required' });
    }
    const user = await User.findById(req.user.id);
    if (user.status !== 'ACTIVE') {
      return res.status(403).json({ error: 'account inactive' });
    }
    const total = req.body.items.reduce((s, i) => s + i.price * i.quantity, 0);
    const order = await Order.create({ userId: user.id, items: req.body.items, total });
    await mailer.send(user.email, 'Order received', `Total: ${total}`);
    res.status(201).json(order);
  } catch (err) {
    next(err);
  }
});

// After — services/orderService.js
async function placeOrder({ userId, items }) {
  const user = await User.findById(userId);
  if (user.status !== UserStatus.ACTIVE) throw new ForbiddenError('account inactive');
  const total = calculateTotal(items);
  const order = await Order.create({ userId, items, total });
  await mailer.send(user.email, 'Order received', `Total: ${total}`);
  return order;
}

// After — routes/orders.js
router.post('/orders', validate(orderSchema), async (req, res) => {
  const order = await orderService.placeOrder({ userId: req.user.id, items: req.body.items });
  res.status(201).json(order);
});
```

**Guardrail:** only do this when the handler holds real logic. A handler that does `res.json(await Item.find())` is already fine.

---

## Repeated logic across routes → middleware

**Smell:** the same auth check, ownership check, body validation or rate-limit logic is copied into several handlers.

**Refactoring:** Extract Middleware. Write a function `(req, res, next)` and mount it on the routes (or router) that need it.

```js
// middleware/requireRole.js
const requireRole = (role) => (req, res, next) => {
  if (!req.user?.roles.includes(role)) {
    return next(new ForbiddenError(`requires role ${role}`));
  }
  next();
};

router.delete('/users/:id', requireRole('admin'), deleteUser);
```

For validation, prefer the schema library the project already uses (Joi, Zod, express-validator, ajv…) wrapped in one `validate(schema)` middleware, rather than hand-written checks in each handler. Search the repo before introducing a new library.

**Check the order:** middleware order is behavior. Moving a check before or after `express.json()`, auth or logging can change what happens.

---

## try/catch boilerplate → central error handling

**Smell:** every async handler is wrapped in `try { ... } catch (err) { next(err) }`, or worse, each catch formats its own error response differently.

**Refactoring:**
1. Make sure there is one error-handling middleware (four arguments: `err, req, res, next`) registered after all routes. It maps known error types to status codes and hides internal details from clients.
2. Remove the per-handler boilerplate:
   - **Express 5:** async handlers that throw or reject are forwarded to the error handler automatically — just delete the try/catch.
   - **Express 4:** use the project's existing async wrapper if there is one (search for `asyncHandler`, `catchAsync`, `express-async-errors`); otherwise add one small wrapper.

```js
// Express 4 wrapper — only if the project has none
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

// errors.js — typed errors carry their own status
class AppError extends Error {
  constructor(message, status) { super(message); this.status = status; }
}
class NotFoundError extends AppError { constructor(m = 'not found') { super(m, 404); } }

// app.js — registered last
app.use((err, req, res, next) => {
  const status = err.status ?? 500;
  if (status >= 500) logger.error(err);
  res.status(status).json({ error: status >= 500 ? 'internal error' : err.message });
});
```

**Keep contracts stable:** if clients rely on the exact error body each handler used to send, the central handler must produce the same shape. Compare before and after with characterization tests.

---

## Scattered `process.env` → config module

**Smell:** `process.env.SOMETHING` is read in many files, often with different defaults or parsing (`parseInt` in one place, raw string in another).

**Refactoring:** Extract Config Module. Read and validate every variable once at startup; export a plain, frozen object. The app fails fast on a missing variable instead of failing at the first request that needs it.

```js
// config.js
function required(name) {
  const value = process.env[name];
  if (value === undefined) throw new Error(`Missing env var ${name}`);
  return value;
}

module.exports = Object.freeze({
  port: Number(process.env.PORT ?? 3000),
  databaseUrl: required('DATABASE_URL'),
  jwtSecret: required('JWT_SECRET'),
  maxUploadBytes: Number(process.env.MAX_UPLOAD_BYTES ?? 5 * 1024 * 1024),
});
```

Watch for a hidden behavior change: if a variable used to be read lazily (so it could change between tests), reading it once at import time changes that.

---

## Callback pyramids / mixed promise styles → async/await

**Smell:** nested callbacks, or one function that mixes `.then()` chains with `await`.

**Refactoring:** convert to `async/await` consistently. Use `util.promisify` for callback APIs, or the promise version of core modules (`fs/promises`).

```js
// Before
function loadUser(id, cb) {
  db.query('SELECT ...', [id], (err, rows) => {
    if (err) return cb(err);
    cache.set(id, rows[0], (err2) => {
      if (err2) return cb(err2);
      cb(null, rows[0]);
    });
  });
}

// After
async function loadUser(id) {
  const rows = await db.query('SELECT ...', [id]);
  await cache.set(id, rows[0]);
  return rows[0];
}
```

**Careful:** this changes the function's signature. Update every caller in the same change, or keep a thin callback adapter until they are migrated. Also check whether independent awaits were previously running in parallel — if so, keep that with `Promise.all`.

---

## `req`/`res` leaking into services

**Smell:** a service or model function takes `req` or `res` as a parameter, or reads `req.user` / `req.query` deep inside.

**Why it matters:** the logic can't be reused from a queue worker, a script or a test without faking HTTP objects.

**Refactoring:** Introduce Parameter Object — the handler extracts exactly what the service needs and passes plain values.

```js
// Before
async function listInvoices(req) {
  return Invoice.find({ orgId: req.user.orgId }).limit(req.query.limit);
}

// After
async function listInvoices({ orgId, limit }) {
  return Invoice.find({ orgId }).limit(limit);
}
```

---

## Response shape consistency

**Smell:** some routes return `{ data }`, others the bare object, others `{ success: true, result }`.

Don't "fix" this silently — response shapes are a public contract. Note it in your report and let the user decide. If the user wants it unified, add a small helper and migrate routes one by one, with a version bump or a coordinated client change.
