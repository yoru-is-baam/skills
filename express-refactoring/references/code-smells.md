# Code Smell Catalog

All 23 code smells from the Refactoring.Guru catalog, grouped the same way, with what each looks like in a Node.js / Express backend and which refactorings fix it. Smell names follow the catalog; the explanations here are this skill's own.

A smell is a *hint*, not a verdict. Before acting, confirm that it actually makes the code harder to change, and respect the guardrails and minimal-impact rules in SKILL.md.

Terminology: the catalog speaks in object-oriented terms. In JavaScript, read **method** as *function* (or method), **class** as *module or class*, and **field** as *property or module-level variable*.

## Contents
- Bloaters: Long Method, Large Class, Primitive Obsession, Long Parameter List, Data Clumps
- Object-Orientation Abusers: Switch Statements, Temporary Field, Refused Bequest, Alternative Classes with Different Interfaces
- Change Preventers: Divergent Change, Shotgun Surgery, Parallel Inheritance Hierarchies
- Dispensables: Comments, Duplicate Code, Lazy Class, Data Class, Dead Code, Speculative Generality
- Couplers: Feature Envy, Inappropriate Intimacy, Message Chains, Middle Man, Incomplete Library Class

---

## Bloaters

Code that has grown too large to work with comfortably. It usually builds up slowly, one small addition per task.

### Long Method
- **Signs in JS:** a route handler or service function you have to scroll; comment headings inside (`// validate`, `// save`, `// notify`); several levels of nesting.
- **Refactorings:** Extract Method, Decompose Conditional, Replace Temp with Query, Introduce Parameter Object, Replace Method with Method Object. In Express, move business logic out of the handler into a service (`express-patterns.md`).
- **In legacy code:** prefer Sprout Function for new behavior rather than lengthening it further (`legacy-changes.md`).
- Link: https://refactoring.guru/smells/long-method

### Large Class
- **Signs in JS:** a `utils.js` or `helpers.js` that everything imports; a service module with 40 exported functions covering unrelated topics; a 1,500-line `app.js` or `routes.js`.
- **Refactorings:** Extract Class (→ Extract Module), Extract Subclass, Extract Interface. Split route files by resource with `express.Router()`.
- Link: https://refactoring.guru/smells/large-class

### Primitive Obsession
- **Signs in JS:** money as a bare number, currency as a separate string; statuses as raw strings compared in many files; `[lat, lng]` arrays; emails normalized (`.trim().toLowerCase()`) in several places.
- **Refactorings:** Replace Data Value with Object, Replace Type Code with Class, Replace Type Code with State/Strategy, Replace Array with Object, Introduce Parameter Object, Replace Magic Number with Symbolic Constant.
- Link: https://refactoring.guru/smells/primitive-obsession

### Long Parameter List
- **Signs in JS:** more than 3–4 positional parameters; call sites full of `null, undefined, true` placeholders.
- **Refactorings:** Introduce Parameter Object (an options object), Preserve Whole Object, Replace Parameter with Method Call.
- Link: https://refactoring.guru/smells/long-parameter-list

### Data Clumps
- **Signs in JS:** the same group of values travels together — `userId, orgId, role` or `page, limit, sort` — through several functions.
- **Refactorings:** Introduce Parameter Object, Preserve Whole Object, Extract Class.
- Link: https://refactoring.guru/smells/data-clumps

---

## Object-Orientation Abusers

Features of the language used in a way that works against good design.

### Switch Statements
- **Signs in JS:** a `switch` or `if/else if` chain on a type or status string, especially the same chain repeated in several functions.
- **Refactorings:** Replace Conditional with Polymorphism (in JS, usually a lookup map of functions), Replace Type Code with State/Strategy, Replace Parameter with Explicit Methods, Introduce Null Object.
- **Leave it** when it's a single, short switch in one place.
- Link: https://refactoring.guru/smells/switch-statements

### Temporary Field
- **Signs in JS:** an object property or module-level variable that is only set during one operation and is `undefined` the rest of the time — e.g. `this.currentRequest` on a long-lived service instance (dangerous with concurrent requests).
- **Refactorings:** Extract Class (or Replace Method with Method Object) for the operation that needs the field; pass the value as a parameter instead; Introduce Null Object.
- Link: https://refactoring.guru/smells/temporary-field

### Refused Bequest
- **Signs in JS:** a class extends a base class but overrides methods to throw `not supported`, or ignores most of what it inherits.
- **Refactorings:** Replace Inheritance with Delegation, Extract Superclass, Push Down Method / Push Down Field.
- Link: https://refactoring.guru/smells/refused-bequest

### Alternative Classes with Different Interfaces
- **Signs in JS:** two modules do the same job with different function names — `mailer.send(to, subject)` vs `emailClient.deliver({ recipient, title })`; two storage adapters with different method names.
- **Refactorings:** Rename Method, Move Method, Add Parameter / Parameterize Method, Extract Superclass, then remove the duplicate. An Adapter can bridge them during migration.
- Link: https://refactoring.guru/smells/alternative-classes-with-different-interfaces

---

## Change Preventers

Smells that make one change require edits in many places.

### Divergent Change
- **Signs in JS:** one module changes for many unrelated reasons — `orderService.js` is edited for tax rules, email templates, and database schema changes.
- **Refactorings:** Extract Class (split by reason to change), Extract Superclass / Extract Subclass.
- Link: https://refactoring.guru/smells/divergent-change

### Shotgun Surgery
- **Signs in JS:** one small change (a new user status, a renamed config variable, a new error format) requires edits in ten files.
- **Refactorings:** Move Method, Move Field (gather the scattered pieces into one module), Inline Class. A config module, a constants module, or a central error handler often fixes this in Express.
- Link: https://refactoring.guru/smells/shotgun-surgery

### Parallel Inheritance Hierarchies
- **Signs in JS:** every time you add a subclass in one hierarchy, you must add a matching one in another (e.g. `StripePayment` + `StripePaymentValidator`). Rare in idiomatic Node.
- **Refactorings:** Move Method, Move Field — make one hierarchy refer to the other, then remove one of them.
- Link: https://refactoring.guru/smells/parallel-inheritance-hierarchies

---

## Dispensables

Things the code would be better without.

### Comments
- **Signs in JS:** comments that explain *what* a block does (a sign the block should be a named function), or comments that have drifted out of date.
- **Refactorings:** Extract Variable, Extract Method, Rename Method, Introduce Assertion.
- **Keep** comments that explain *why* (a business rule, a workaround for a library bug, a security decision), and JSDoc for exported APIs.
- Link: https://refactoring.guru/smells/comments

### Duplicate Code
- **Signs in JS:** copy-pasted validation, error formatting, query building or response mapping across handlers. Check with `jscpd` (`tooling.md`).
- **Refactorings:** Extract Method, Pull Up Method, Form Template Method, Substitute Algorithm, Consolidate Conditional Expression, Consolidate Duplicate Conditional Fragments, Extract Class. In Express: Extract Middleware.
- **Rule of three:** two copies may diverge; three is the signal.
- Link: https://refactoring.guru/smells/duplicate-code

### Lazy Class
- **Signs in JS:** a module or class that only forwards to something else, or holds one tiny function and is imported in one place; a `services/x.js` that adds nothing between the route and the model.
- **Refactorings:** Inline Class, Collapse Hierarchy.
- Link: https://refactoring.guru/smells/lazy-class

### Data Class
- **Signs in JS:** an object with only properties, while all the rules about it live elsewhere and are repeated (every caller checks `order.status === 'PAID' && !order.refundedAt`).
- **Refactorings:** Encapsulate Field, Encapsulate Collection, Move Method (put the rules next to the data — a function in the same module), Remove Setting Method, Hide Method.
- **Note:** plain data objects (DTOs, API payloads, DB rows) are normal in Node. It's only a smell when logic about them is duplicated around the codebase.
- Link: https://refactoring.guru/smells/data-class

### Dead Code
- **Signs in JS:** unused exports, unused route handlers, commented-out code, feature flags that are always on, `if (false)`.
- **Refactorings:** delete it (git keeps history); Inline Class, Collapse Hierarchy, Remove Parameter.
- **Careful:** check dynamic usage first — string-based `require`, routes registered by loops, functions referenced from config, and code used by other services or scripts. Removing an export is an API change.
- Link: https://refactoring.guru/smells/dead-code

### Speculative Generality
- **Signs in JS:** an abstract base class with one subclass; a plugin system with one plugin; parameters and options nobody passes; a "generic" repository layer wrapping a single query.
- **Refactorings:** Collapse Hierarchy, Inline Class, Inline Method, Remove Parameter.
- This is the smell agents create most often. See the guardrails in SKILL.md.
- Link: https://refactoring.guru/smells/speculative-generality

---

## Couplers

Too much dependency between modules.

### Feature Envy
- **Signs in JS:** a function that reads many properties of another module's object and few of its own — e.g. a route handler computing an order's total from `order.items`, `order.discounts` and `order.customer.tier`.
- **Refactorings:** Move Method (to where the data lives), Extract Method then move part of it.
- Link: https://refactoring.guru/smells/feature-envy

### Inappropriate Intimacy
- **Signs in JS:** a module reaches into another's internals — imports its private helpers, mutates its internal state, or depends on undocumented properties; two modules `require` each other (circular dependency).
- **Refactorings:** Move Method, Move Field, Extract Class, Hide Delegate, Change Bidirectional Association to Unidirectional, Replace Inheritance with Delegation.
- Link: https://refactoring.guru/smells/inappropriate-intimacy

### Message Chains
- **Signs in JS:** `req.user.account.subscription.plan.limits.maxSeats` — the caller depends on the whole object graph. (Optional chaining `?.` along a long chain often signals this too.)
- **Refactorings:** Hide Delegate, Extract Method, Move Method.
- Link: https://refactoring.guru/smells/message-chains

### Middle Man
- **Signs in JS:** a service whose every function just calls one model or client function with the same arguments.
- **Refactorings:** Remove Middle Man, Inline Method, Replace Delegation with Inheritance.
- **Balance with Hide Delegate:** a thin layer is fine when it isolates a third-party library or a database you may swap; it's a smell when it isolates nothing.
- Link: https://refactoring.guru/smells/middle-man

### Incomplete Library Class
- **Signs in JS:** a dependency lacks a function you need, and the workaround is repeated in several places.
- **Refactorings:** Introduce Foreign Method (a helper in your own module), Introduce Local Extension (your own wrapper module around the library). **Never** patch a library's prototype or the global object.
- Link: https://refactoring.guru/smells/incomplete-library-class
