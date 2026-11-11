---
name: express-refactoring
description: Keeps Node.js / JavaScript / Express.js backend code clean by spotting code smells and applying named refactorings (Extract Method, Inline Temp, Decompose Conditional, Change Value to Reference, etc.) and design patterns (Strategy, Adapter, Facade, Decorator, etc.) safely, in small steps, with minimal impact on existing behavior, and in the project's own structure, naming and style. Use this skill whenever writing, modifying, reviewing or refactoring backend JavaScript — route handlers, middleware, services, controllers, models, utilities — even if the user only says "add an endpoint", "fix this bug", "clean this up" or "review my PR" and never mentions refactoring.
---

# Express Refactoring

Coding agents tend to make four mistakes in backend JavaScript:

1. **Additive bias** — writing a new helper instead of finding and reusing the one that already exists, so duplication grows.
2. **Shallow cleanup** — renaming and reformatting while leaving the real design problems (fat route handlers, tangled conditionals, shared mutable data) untouched.
3. **Over-abstraction** — extracting tiny one-use helpers, adding layers and classes "for flexibility" that nobody needs.
4. **Ignoring the house style** — writing code in the agent's own preferred structure, libraries and naming instead of the way this codebase already does things.

This skill exists to counter all four. The goal is not "apply as many refactorings as possible" — it is to leave each file you touch a little easier to change than you found it, without changing what it does, and looking like it was written by the same team.

## Learn the project first

**The project's existing structure and conventions take priority over every default in this skill.** Before writing code, spend a few minutes detecting how this codebase works (details and commands in `references/project-conventions.md`):

- **Module system and language:** CommonJS or ESM, JavaScript or TypeScript, Express 4 or 5.
- **Libraries already chosen** for validation, database, logging, testing and config — use those; never add a second library for the same job.
- **Folder structure** (layer-based, feature-based or flat) — put new code where its siblings live.
- **File naming, export style, error classes, response shape, test layout** — read the 2–3 files most similar to what you're writing and mirror them.
- **Written team rules** in `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` or architecture docs — these override everything inferred.

The examples in this skill use CommonJS and plain functions to show *ideas*; translate them into the project's form. The one exception to "the project wins": don't copy a pattern that is unsafe or causes bugs (string-built SQL, per-request data in module-level variables, unhandled rejections) into new code — write the safe version and say why in your report.

## When to refactor

- **Before a change:** if the code you need to modify is hard to change, refactor first so the feature lands cleanly ("make the change easy, then make the easy change").
- **After a change:** once tests pass, look at the diff you just wrote for the smells in the index below.
- **On request:** when the user asks for cleanup or a review.

Stay within the area the task touches. Don't start refactoring unrelated files — propose it to the user instead, because unasked-for churn makes reviews harder.

## Minimal-impact changes in existing code

In a real project you cannot clean up the whole codebase, and you shouldn't try. The goal for any change to existing code is: **the smallest possible blast radius, and no new smells.** A messy file that stays exactly as messy (plus one clean, tested addition) is a success. A tidy-looking diff that rewrites 300 lines nobody asked about is not.

**Three zones, three standards:**

| Zone | Standard |
|---|---|
| Code you write (new functions, new modules) | Fully clean: no smells, tested |
| The function you are changing | Leave it slightly better *only if* tests cover it, and only inside that function |
| Everything else | Don't touch it. List smells you noticed in your report as follow-ups |

**Prefer additive techniques over rewriting** (details and Express examples in `references/legacy-changes.md`):

- **Sprout Function / Sprout Module** — write the new logic as a new, clean, tested function and add a single call to it from the old code.
- **Wrap Function** — keep the old function untouched, and put the new behavior in a wrapper with the original name, so no caller changes.
- **Optional parameter with a default** — extend a function without breaking existing callers.
- **Parallel change** (expand → migrate → contract) — when a signature or response shape really must change, add the new version beside the old one, move callers over, and remove the old one in a later change.

**Before changing any existing function, find its callers:**

```bash
grep -rn "functionName" src/ test/ --include=*.js
```

If it is exported, used in many places, or part of an HTTP response, treat its signature and output as a contract and use one of the additive techniques above.

**Measure "no new smells" on the diff, not the codebase.** Run lint on changed files only and compare duplication before/after (`references/tooling.md`). Pre-existing warnings in lines you didn't write are not yours to fix in this change.

**Keep the diff focused.** No reformatting, renaming, reordering imports or "while I'm here" fixes outside the lines the task needs — that noise hides the real change from reviewers and raises the risk of conflicts with teammates' branches.

## Before writing new code: search first

Before adding any function, middleware, validator, constant or utility, search the repo for something that already does it:

```bash
grep -rn "functionName\|keyword" src/ --include=*.js
```

Look for similar names, similar string literals (error messages, SQL fragments, route paths) and similar shapes. If something close exists, reuse or generalize it. This single habit prevents most duplication.

## Safety protocol

Refactoring means changing structure **without changing behavior**. That is only verifiable with tests.

1. **Check the safety net.** Run the tests that cover the code. If there are none, write a few characterization tests first (tests that pin down current behavior, even odd behavior) — especially for route handlers, using `supertest` or the existing test setup.
2. **One refactoring at a time.** Apply a single named refactoring, then run the tests. Small steps make a failure easy to locate.
3. **Never mix refactoring with behavior change in one commit.** Reviewers need to see that a refactor commit changes nothing observable.
4. **Name what you did.** Commit messages and PR descriptions should name the refactoring so the change is easy to review:
   - `refactor: Extract Method buildOrderSummary() from POST /orders handler`
   - `refactor: Replace Magic Number with Symbolic Constant (MAX_UPLOAD_BYTES)`
   - `refactor: Replace Conditional with Polymorphism (Strategy) for shipping cost`
5. **Keep public contracts stable.** Response shapes, status codes, headers, route paths, exported module APIs and error messages that clients may depend on must stay the same unless the user asked to change them.

## Smell → refactoring index

Spot the smell first, then pick the refactoring. This table covers the smells that come up most in Express backends; **`references/code-smells.md` has the full catalog of 23 smells** with their refactorings. Read the linked reference file for details and JavaScript examples.

Refactoring names follow the Refactoring.Guru catalog, which uses object-oriented terms: read *Method* as "function", *Class* as "module or class", *Field* as "property or module-level variable". Use the catalog names in commit messages so reviewers can look them up.

| Smell (what you notice) | Refactorings to consider | Reference |
|---|---|---|
| Route handler doing validation, business logic, DB access and response formatting | Extract Method → move to a service (Facade); Extract Middleware | `express-patterns.md` |
| Same validation / auth / error handling repeated across routes | Extract Middleware (Chain of Responsibility) | `express-patterns.md` |
| `try/catch` + `next(err)` boilerplate in every handler | Central error handler; async wrapper (Express 4) or native promise handling (Express 5); Replace Error Code with Exception | `express-patterns.md`, `simplifying-function-calls.md` |
| `process.env.X` read all over the codebase | Extract Config Module | `express-patterns.md` |
| Callback pyramids or mixed `.then()` and `await` | Convert to `async/await` consistently | `express-patterns.md` |
| `req` / `res` passed deep into services | Separate HTTP layer from domain logic; Introduce Parameter Object | `express-patterns.md` |
| Long Method (needs scrolling, comment headings inside) | Extract Method; Decompose Conditional; Replace Method with Method Object | `composing-functions.md` |
| Complicated expression you have to read twice | Extract Variable | `composing-functions.md` |
| Variable or function that adds nothing over what it wraps | Inline Temp; Inline Method | `composing-functions.md` |
| Same temp variable reassigned for different purposes | Split Temporary Variable | `composing-functions.md` |
| Function argument reassigned or mutated inside the body | Remove Assignments to Parameters | `composing-functions.md` |
| Hand-written loop that a built-in already does | Substitute Algorithm | `composing-functions.md` |
| Deeply nested `if/else` | Replace Nested Conditional with Guard Clauses | `simplifying-conditionals.md` |
| `switch` / `if` chain on a type or status string, repeated | Replace Conditional with Polymorphism (lookup map / Strategy); Replace Type Code with State/Strategy | `simplifying-conditionals.md`, `organizing-data.md` |
| Same code in every branch of an `if` | Consolidate Duplicate Conditional Fragments | `simplifying-conditionals.md` |
| `null`/`undefined` checks scattered everywhere | Introduce Null Object; validate at the boundary | `simplifying-conditionals.md` |
| Unexplained numbers or strings (`86400`, `'ACTIVE'`) | Replace Magic Number with Symbolic Constant; Replace Type Code with Class | `organizing-data.md` |
| Several copies of the same entity drifting out of sync | Change Value to Reference | `organizing-data.md` |
| Shared object or array mutated from many places | Change Reference to Value; Encapsulate Field; Encapsulate Collection | `organizing-data.md` |
| Money, dates, emails as bare primitives with repeated logic | Replace Data Value with Object; Replace Array with Object | `organizing-data.md` |
| 4+ positional parameters, or values always passed together | Introduce Parameter Object; Preserve Whole Object | `simplifying-function-calls.md` |
| Boolean flag parameter switching behavior | Replace Parameter with Explicit Methods | `simplifying-function-calls.md` |
| Function that both returns a value and changes state | Separate Query from Modifier (mind atomicity) | `simplifying-function-calls.md` |
| Unclear name (`process`, `handle`, `getData` that writes) | Rename Method | `simplifying-function-calls.md` |
| Function using another module's data more than its own (Feature Envy) | Move Method; Move Field | `moving-features.md` |
| `a.b.c.d.value` chains through other objects | Hide Delegate | `moving-features.md` |
| Service that only forwards calls (Middle Man) | Remove Middle Man; Inline Class | `moving-features.md` |
| One module changing for many unrelated reasons | Extract Class (Extract Module) | `moving-features.md` |
| Third-party library missing a function you need repeatedly | Introduce Foreign Method; Introduce Local Extension (Adapter) | `moving-features.md` |
| Class extending another but using little of it (Refused Bequest) | Replace Inheritance with Delegation | `generalization.md` |
| Base class with one child, unused hooks (Speculative Generality) | Collapse Hierarchy; Inline Class | `generalization.md` |
| Same steps in the same order with different details | Form Template Method (higher-order function) | `generalization.md` |

## Design patterns

`references/design-patterns.md` covers all 22 patterns from the catalog, with a quick selector table and the idiomatic Node.js form of each. The ones that matter most in Express backends are **Strategy, Factory Method, Adapter, Facade, Decorator/Proxy, Chain of Responsibility (middleware), Observer and Command (job queues)**.

Rules for using them:
- **A pattern must answer a named problem** in the current task — a repeated `switch`, a leaking SDK, a tangled handler. "It might be useful later" is not a reason.
- **Use the lightest JavaScript form**: a lookup map instead of a Strategy class hierarchy, a higher-order function instead of a Decorator class, module caching instead of a Singleton class.
- **Recognize patterns that are already there** (middleware, `EventEmitter`, query builders) and extend them, instead of adding a parallel mechanism.
- Patterns change structure, so the safety protocol and minimal-impact rules apply: introduce one in a separate refactoring commit, with tests passing before and after.

## Reference files

Open only the file the current smell or task points to.

| File | Contents |
|---|---|
| `references/project-conventions.md` | **Read at the start of a task in an unfamiliar codebase.** How to detect the module system, libraries, folder structure, file naming, code shape and team rules — and how to follow them |
| `references/style-and-naming.md` | Naming conventions, consistent vocabulary, Express route/middleware naming, blank lines between logical steps, module ordering, comments |
| `references/code-smells.md` | All 23 code smells (Bloaters, OO Abusers, Change Preventers, Dispensables, Couplers) with JS signs and fixes |
| `references/composing-functions.md` | Composing Methods: Extract/Inline Method, Extract Variable, Inline Temp, Replace Temp with Query, Split Temporary Variable, Remove Assignments to Parameters, Replace Method with Method Object, Substitute Algorithm |
| `references/moving-features.md` | Moving Features between Objects: Move Method/Field, Extract/Inline Class, Hide Delegate, Remove Middle Man, Introduce Foreign Method, Introduce Local Extension |
| `references/organizing-data.md` | Organizing Data: constants, value vs reference, encapsulation, type codes, associations (15 refactorings) |
| `references/simplifying-conditionals.md` | Simplifying Conditional Expressions (8 refactorings) |
| `references/simplifying-function-calls.md` | Simplifying Method Calls: names, parameters, query/modifier, factories, errors (14 refactorings) |
| `references/generalization.md` | Dealing with Generalization: pull up / push down, extract superclass/subclass/interface, template method, inheritance vs delegation (12 refactorings) |
| `references/design-patterns.md` | All 22 design patterns, Node.js idioms, when to use and when not |
| `references/express-patterns.md` | Express-specific smells: fat handlers, middleware, error handling, config, async/await, HTTP vs domain |
| `references/legacy-changes.md` | Minimal-impact techniques: Sprout, Wrap, optional parameters, parallel change |
| `references/tooling.md` | ESLint complexity rules, duplication checks, linting only changed files, characterization tests |

Each catalog entry links to its Refactoring.Guru page for further reading. The explanations and examples in this skill are original, written for Node.js and Express.

## Guardrails against over-refactoring

These matter as much as the catalog. Each one exists because agents tend to overdo it.

- **Rule of three.** Duplicated once? Tolerate it. A third copy is the signal to extract. Two similar-looking blocks are often coincidentally similar and will diverge.
- **Don't extract trivial one-use helpers.** A function earns its existence by naming an idea or removing duplication. `const isEmpty = (s) => s.length === 0` used once makes code harder to read, not easier.
- **No speculative abstraction.** Don't add a class, factory, interface-like base class, or plugin system for a variation that doesn't exist yet. Plain functions and modules are idiomatic in Node — reach for classes only when you have real state plus behavior to bundle.
- **Don't add layers for their own sake.** A route → service split is useful once a handler holds real business logic. A 5-line CRUD handler doesn't need a controller, a service and a repository.
- **Prefer the smallest refactoring that removes the smell.** Extract Variable before Extract Function; Extract Function before a new module.
- **Leave mechanical formatting to the tools.** Indentation, quotes, semicolons and line wrapping belong to Prettier/ESLint, not to refactoring commits. Readability decisions the tools can't make — names, blank lines between logical steps, the order of functions — are part of this skill (`references/style-and-naming.md`), applied to code you write and lines you change.

If you are unsure whether a refactoring improves the code, compare before/after and ask: would a teammate new to this file understand the "after" version faster? If not, don't do it.

## Workflow summary

1. Learn the project's conventions (structure, libraries, naming, the 2–3 most similar files). Read the code the task touches; find the callers of anything you will change; search for existing helpers before writing new ones.
2. Run the tests (add characterization tests for the function you will change if missing).
3. Decide the approach with the least impact: Sprout or Wrap for messy legacy code; a small, named refactoring of the touched function only when tests cover it.
4. Make the behavior change. Tests pass.
5. Scan **your own diff** (not the whole file) against the smell index and the style rules: does it look like the surrounding code — names, blank lines, structure? Fix what's clearly worth it; respect the guardrails.
6. Run linters on changed files and compare duplication before/after (`references/tooling.md`).
7. Report: the conventions you detected and followed (one line), the refactorings and techniques you applied by name, and any smells or inconsistencies you noticed but deliberately left alone (with a one-line reason), as follow-up suggestions.
