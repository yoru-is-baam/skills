# Following the Project's Structure and Conventions

**Consistency with the existing codebase beats any "best practice" in this skill.** Code that is textbook-clean but looks different from everything around it is harder for the team to read, review and maintain. A codebase with two styles side by side is a smell of its own.

So before writing or changing code in a project, learn how *this* project does things — then do it the same way.

## Contents
- What to detect, and how
- Rules for writing code that fits
- When the project is inconsistent
- When a project convention conflicts with this skill
- Report what you detected

---

## What to detect, and how

Spend a few minutes on this at the start of a task. It's cheap, and it prevents most "this doesn't look like our code" review comments.

### 1. Language and module system

```bash
cat package.json          # "type": "module" → ES modules; dependencies; scripts
ls tsconfig.json jsconfig.json 2>/dev/null
```

- **CommonJS** (`require` / `module.exports`) or **ES modules** (`import` / `export`)? Examples in this skill use CommonJS; translate them if the project uses ESM.
- **JavaScript or TypeScript?** In TypeScript, add types the way neighboring files do (interfaces vs type aliases, explicit return types or not).
- **Express version** (4 or 5) — it changes how async errors are handled (`express-patterns.md`).

### 2. The libraries already in use

Read `dependencies` in `package.json`. For each job below, the project's existing choice is the one to use:

| Job | Common options |
|---|---|
| Validation | Joi, Zod, express-validator, Yup, celebrate, ajv |
| Database / ORM | Mongoose, Prisma, Sequelize, TypeORM, Knex, raw `pg`/`mysql2` |
| Logging | pino, winston, a custom `logger` module |
| Testing | Jest, Vitest, Mocha, `node:test`, supertest |
| Config | dotenv, a `config/` module, convict |
| Async error handling | `express-async-errors`, an `asyncHandler`/`catchAsync` wrapper, Express 5 native |

**Never add a second library for a job the project already covers** (e.g. Zod in a Joi project). If no library covers a job you need, propose one to the user instead of installing it.

### 3. Folder structure

```bash
find src -maxdepth 3 -type d -not -path '*/node_modules*'   # adjust "src" to the project's root
```

Recognize the layout and keep to it:

- **Layer-based:** `routes/`, `controllers/`, `services/`, `models/`, `middlewares/`, `utils/`
- **Feature-based:** `modules/users/{users.routes.js, users.controller.js, users.service.js}`
- **Flat / small:** a few files at the root — don't impose layers on it for a small change.

New code goes **where similar code already lives**. A new route for invoices goes beside the other route files, in the same shape — not into a new folder structure.

### 4. File naming

```bash
find src -name '*.js' -not -path '*/node_modules*' | head -40
```

Match the pattern you see: `user-service.js` (kebab-case), `userService.js` (camelCase), `user.service.js` (dot suffix), `User.js` (PascalCase models), `index.js` barrels.

### 5. Code shape — read 2–3 neighbors

Open the two or three files most similar to what you're about to write (the nearest route, controller, service, test) and use them as templates. Note:

- **Exports:** named vs default; `module.exports = { a, b }` at the bottom vs `exports.a =` inline; classes vs plain functions.
- **Handlers:** inline in the router, or in controller files; functions or class methods; wrapped in an async helper or not.
- **Errors:** existing error classes (`grep -rn "extends Error" src/`), how errors reach the error middleware, the JSON error shape.
- **Responses:** envelope (`{ data, meta }`, `{ success, result }`) or bare objects; status code habits (201 on create? 204 on delete?).
- **Validation:** where it happens (middleware vs inside the handler) and the error format it produces.
- **Async style:** `async/await` everywhere, or promise chains.
- **Logging:** `logger.info(...)` with structured fields, or `console.log`.
- **Imports:** order and grouping, path aliases (`@/services`, `~/`), relative depth.
- **Tests:** location (`__tests__/`, `test/`, co-located `*.test.js`), naming (`*.test.js` vs `*.spec.js`), `describe`/`it` wording, how the app and DB are set up.

### 6. Style configuration

```bash
ls .prettierrc* .editorconfig eslint.config.* .eslintrc* 2>/dev/null
```

These define quotes, semicolons, indentation, line length and lint rules. Run them rather than guessing (`tooling.md`).

### 7. Project instructions

If the repo has a `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, an architecture doc or an ADR folder, read the relevant parts. Written team rules take priority over everything inferred above.

---

## Rules for writing code that fits

1. **Mirror the nearest similar file.** Same structure, same export style, same error handling, same naming pattern.
2. **Place new files where their siblings live**, with the same naming pattern and file suffix.
3. **Reuse the project's building blocks** — its error classes, validation middleware, logger, config module, response helpers, test utilities. Search before writing (`grep -rn`), as SKILL.md says.
4. **Don't introduce a new library, folder, layer or architectural style** when the project already has one for that job. If something new is genuinely needed, propose it in your report and let the user decide.
5. **Translate this skill's examples** into the project's form: ESM instead of CommonJS, TypeScript types, class-based controllers, the project's ORM and validation library. The examples show the *idea*, not a required syntax.

---

## When the project is inconsistent

Real codebases often mix conventions (old files in one style, newer ones in another).

- **Inside an existing file:** follow that file's style, even if it's the older one.
- **For a new file:** follow the nearest siblings (same folder or feature). If they disagree, follow the most recent files (`git log --format='%ad %h' -- <file>` shows dates), since they usually reflect where the team is heading.
- **Don't "fix" the inconsistency** in code you aren't changing. Mention it in your report as a follow-up; unifying a convention across a codebase is a team decision.

---

## When a project convention conflicts with this skill

- **Style and structure** (naming, folder layout, export style, response envelope, classes vs functions): **the project wins.** Follow it even where this skill's defaults differ.
- **Correctness, security and reliability** — e.g. string-concatenated SQL, unhandled promise rejections, per-request data stored in module-level variables, secrets in code: **don't copy the harmful pattern into new code.** Write the safe version, keep it as close to the project's style as possible, and point out the deviation and the reason in your report. Don't rewrite existing occurrences unless asked — list them as follow-ups.

---

## Report what you detected

At the end of a task, state briefly which conventions you followed, so the user can correct a wrong guess. For example:

> Followed existing conventions: CommonJS, feature folders (`modules/<name>/`), `*.service.js` naming, Joi validation via `validate()` middleware, `AppError` subclasses, `{ data }` response envelope, Jest tests in `__tests__/`.
