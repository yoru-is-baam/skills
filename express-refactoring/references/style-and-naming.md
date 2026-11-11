# Style and Naming

Readability decisions that formatting tools can't make: what to call things, and how to lay out code so its structure is visible at a glance.

**The project's conventions come first** (`project-conventions.md`). Everything below is a default for when the project has no established convention, or for choices the project leaves open. And as with every rule in this skill, apply it to **code you write and lines you change** — never reformat or rename untouched code as a side effect.

## Contents
- What tools handle vs what this file handles
- Naming conventions
- Consistent vocabulary
- Express-specific naming
- Vertical spacing: code paragraphs
- Ordering inside a module
- Comments
- Tooling support

---

## What tools handle vs what this file handles

| Concern | Who decides |
|---|---|
| Indentation, quotes, semicolons, trailing commas, line wrapping | Prettier (or the project's formatter) — just run it |
| Collapsing multiple blank lines into one | Prettier |
| **Where** a blank line belongs between logical steps | **You** — Prettier keeps blank lines but never adds them |
| Identifier names, file names, route names | **You**, following the project's convention |
| Order of functions in a module | **You** |

---

## Naming conventions

Defaults for JavaScript / Node.js:

| What | Convention | Examples |
|---|---|---|
| Variables, functions, parameters, object properties | `camelCase` | `orderTotal`, `findUserByEmail` |
| Classes, constructor functions | `PascalCase` | `OrderService`, `NotFoundError` |
| True constants (fixed values, module level) | `UPPER_SNAKE_CASE` | `MAX_UPLOAD_BYTES`, `DEFAULT_PAGE_SIZE` |
| Frozen enum objects | `PascalCase` object, `UPPER_SNAKE` keys | `OrderStatus.PAID` |
| Environment variables | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `JWT_SECRET` |
| Private class members | `#name` | `#cache` |

**Names that describe meaning:**

- **Functions start with a verb** that says what they do: `createInvoice`, `sendPasswordResetEmail`, `calculateShippingCost`, `toProfileResponse`. Avoid vague verbs (`handle`, `process`, `manage`, `doStuff`) unless the project uses them as a convention (e.g. Express handler names).
- **Booleans read as yes/no questions:** `isActive`, `hasPaid`, `canRefund`, `shouldRetry`. Avoid negatives (`isNotValid`) — `!isValid` reads better.
- **Collections are plural:** `users`, `orderItems`. A lookup map says its key: `usersById`, `pricesBySku`.
- **Include units when a number has one:** `timeoutMs`, `maxSizeBytes`, `ttlSeconds`, `priceCents`.
- **Name length matches scope:** short names are fine in a 3-line callback (`users.map((u) => u.email)`); module-level and exported names should be fully descriptive.
- **Avoid unclear abbreviations.** Widely understood ones are fine: `id`, `url`, `db`, `req`, `res`, `next`, `err`, `ctx`, `config`.
- **Error classes end in `Error`:** `ValidationError`, `PaymentDeclinedError`.
- **Async functions don't need an `Async` suffix** in modern Node (everything I/O is async) — unless the project uses one.

---

## Consistent vocabulary

Use one word per concept across the codebase, so a name tells the reader what to expect. Detect what the project already uses and keep to it. A common scheme:

| Word | Meaning |
|---|---|
| `find...` | Look up; may return `null` / empty array |
| `get...` | Look up; the thing must exist (throws `NotFoundError` otherwise) — or a cheap in-memory read |
| `list...` | Return many, usually paginated |
| `create...` / `update...` / `delete...` | Persisted changes |
| `build...` / `to...` | Pure construction or conversion, no I/O (`buildQuery`, `toOrderResponse`) |
| `is...` / `has...` / `can...` | Boolean checks |
| `validate...` | Throws or returns errors for bad input |

Mixing `fetchUser`, `getUser`, `loadUser` and `retrieveUser` for the same kind of operation is a small Shotgun Surgery waiting to happen — readers can't tell whether they differ.

---

## Express-specific naming

- **Route paths:** lowercase, plural nouns, kebab-case for multi-word resources: `/users`, `/order-items`, `/users/:userId/orders`. Use actions only for operations that aren't CRUD: `POST /orders/:orderId/cancel`.
- **Route parameters:** `camelCase` and specific — `:orderId` rather than `:id` when routes are nested, so handlers aren't ambiguous.
- **Query parameters and JSON fields:** follow the API's existing style (`camelCase` is most common in Node APIs; some use `snake_case`). Never mix styles inside one API.
- **Router files:** one per resource, named like their siblings (`orders.routes.js`, `orderRoutes.js` …).
- **Middleware:** named after what it checks or does — `requireAuth`, `requireRole('admin')`, `validate(orderSchema)`, `rateLimitByIp`.
- **Handlers / controller functions:** match the project (`createOrder`, `getOrder`, or `OrdersController.create`).
- **Database columns:** if the DB uses `snake_case` (`created_at`), map to `camelCase` at the data-access boundary (ORM config or a mapper) rather than spreading `snake_case` through the code.
- **Event and job names:** pick one format and stick to it (`order.paid`, `send-invoice`).

---

## Vertical spacing: code paragraphs

A function is like a paragraph of prose: blank lines separate *steps*, so a reader can see the structure before reading any line. Prettier won't add these — you have to.

```js
// Before — one dense block
async function createOrder(req, res) {
  const { items, couponCode } = req.body;
  if (!items?.length) return res.status(400).json({ error: 'items required' });
  const customer = await customers.get(req.user.id);
  if (customer.status !== CustomerStatus.ACTIVE) return res.status(403).json({ error: 'inactive account' });
  const total = calculateTotal(items, couponCode);
  const order = await orders.create({ customerId: customer.id, items, total });
  events.emit('order.created', { orderId: order.id });
  res.status(201).json(order);
}

// After — guard clauses, main work, side effects, response
async function createOrder(req, res) {
  const { items, couponCode } = req.body;
  if (!items?.length) return res.status(400).json({ error: 'items required' });

  const customer = await customers.get(req.user.id);
  if (customer.status !== CustomerStatus.ACTIVE) return res.status(403).json({ error: 'inactive account' });

  const total = calculateTotal(items, couponCode);
  const order = await orders.create({ customerId: customer.id, items, total });

  events.emit('order.created', { orderId: order.id });

  res.status(201).json(order);
}
```

Guidelines:
- **One blank line between logical steps** inside a function: input/validation, loading data, the main work, side effects, the response.
- **Keep tightly related lines together** — a value and the check that immediately uses it, or a query and its result handling, belong in the same paragraph.
- **No blank line right after `{` or right before `}`.**
- **Short functions (2–4 lines) need no internal blank lines.**
- **One blank line between functions,** and between class methods.
- **Declare variables close to first use,** not all at the top of the function.
- **If a function has many paragraphs, consider Extract Method** — the paragraphs are often the functions waiting to be named (`composing-functions.md`). Blank lines show structure; they don't fix a function that's too long.

---

## Ordering inside a module

Follow the project's habit first. A common, readable default:

1. Imports, grouped with a blank line between groups: Node built-ins (`node:fs`), packages, then local modules.
2. Module-level constants.
3. Exported / high-level functions first, helpers below them — so the file reads top-down like an outline.
4. `module.exports = { ... }` at the bottom (CommonJS), or `export` on each declaration (ESM), as the project does.

**Caveat:** helpers written as `const x = () => {}` are not hoisted. Placing them *below* the functions that use them is fine as long as they're only *called* later at runtime (not at module load). If the project defines arrow-function helpers, keep the existing order rather than risk a `ReferenceError`.

In existing files, **don't reorder imports or functions** you aren't otherwise changing — it creates noisy diffs and merge conflicts.

---

## Comments

- Comment the **why**, not the what: business rules, workarounds, security decisions, surprising choices.
  ```js
  // Payment provider retries webhooks for 72h, so this must be idempotent.
  ```
- If a comment explains **what** a block does, prefer a function or variable whose name says it (Extract Method / Extract Variable).
- **JSDoc on exported functions** when the project uses it — especially parameter shapes and thrown errors.
- `TODO` comments carry context: what, why, and ideally a ticket — `// TODO(INV-231): remove after all clients use /v2/orders`.
- Delete commented-out code; git keeps history.

---

## Tooling support

Only suggest adding these — don't change lint config without the user's agreement.

- **Prettier** for mechanical formatting; run it on the files you changed.
- **ESLint `camelcase`** rule flags non-camelCase identifiers (configure exceptions for DB fields if needed).
- **`padding-line-between-statements`** can enforce some blank-line rules (e.g. a blank line before `return` in multi-step functions). In current ESLint versions the formatting rules live in `@stylistic/eslint-plugin` rather than ESLint core.
- **TypeScript projects:** `@typescript-eslint/naming-convention` can enforce casing per identifier kind.

Treat lint output on lines you didn't write as pre-existing (`tooling.md`).
