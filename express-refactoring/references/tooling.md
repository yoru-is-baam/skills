# Tooling: Lint, Duplication and Tests

Use tools to catch what review misses. Prefer what the project already has; only suggest new tools to the user, don't install them silently.

## 1. Find what the project uses

```bash
cat package.json            # scripts: test, lint, format; Express version
ls eslint.config.* .eslintrc* .prettierrc* 2>/dev/null
```

Run the project's own scripts first (`npm test`, `npm run lint`). Their configuration reflects the team's decisions.

## 2. Complexity rules (ESLint)

These built-in ESLint rules flag the smells in the SKILL.md index. If the project's config lacks them, suggest adding them as warnings:

```js
// eslint.config.js (flat config)
export default [
  {
    files: ['**/*.js'],
    rules: {
      complexity: ['warn', 10],                   // too many branches → Decompose Conditional
      'max-depth': ['warn', 3],                   // deep nesting → Guard Clauses
      'max-lines-per-function': ['warn', 60],     // long function → Extract Function
      'max-params': ['warn', 4],                  // → Introduce Parameter Object
      'no-param-reassign': ['warn', { props: true }], // → Remove Assignments to Parameters
      'no-magic-numbers': ['warn', { ignore: [0, 1, -1], ignoreArrayIndexes: true }],
      'prefer-const': 'warn',                     // → Split Variable
    },
  },
];
```

`no-magic-numbers` is noisy; treat its warnings as hints, not a checklist. `no-param-reassign` with `props: true` also flags `req.user = ...` in middleware, which is idiomatic Express — allow it for `req`/`res` with `ignorePropertyModificationsFor: ['req', 'res']`.

## 3. Duplication

```bash
npx jscpd src --min-lines 5 --reporters console
```

Treat results with the rule of three: two matches may be coincidental; three or more is a strong signal to extract. Ignore generated files, migrations and test fixtures.

## 3b. Check only what you changed ("no new smells")

In a large codebase, a full lint run shows hundreds of old warnings that are not part of your task. Measure your change instead:

```bash
# JS files changed on this branch compared with the main branch
CHANGED=$(git diff --name-only --diff-filter=ACMR origin/main...HEAD -- '*.js')

# Lint only those files
[ -n "$CHANGED" ] && npx eslint $CHANGED

# Duplication: compare the main branch with your branch
git worktree add ../base-check origin/main
npx jscpd ../base-check/src --silent --reporters console | tail -3   # before
npx jscpd src --silent --reporters console | tail -3                 # after
git worktree remove ../base-check
```

If the project uses `lint-staged` with a pre-commit hook, it already lints only staged files — rely on it.

A warning on a line you didn't touch is pre-existing; mention it as a follow-up rather than fixing it in this change. A warning on a line you wrote is yours to fix.

## 4. Tests

- Use the project's test runner. If there is none, Node's built-in runner works without dependencies: `node --test`.
- For route handlers, `supertest` lets characterization tests call the app without opening a port:

```js
const request = require('supertest');
const app = require('../app');

test('GET /users/:id returns 404 for a missing user', async () => {
  const res = await request(app).get('/users/does-not-exist');
  assert.equal(res.status, 404);
  assert.deepEqual(res.body, { error: 'not found' });
});
```

Characterization tests pin the *current* behavior, including odd behavior. If you find a bug while writing them, note it and leave it for a separate, non-refactoring change.

## 5. Before reporting done

- Tests pass before and after.
- Lint shows no new warnings in the files you touched.
- Duplication did not increase.
- The report lists each refactoring by name, plus smells you noticed but left alone and why.
