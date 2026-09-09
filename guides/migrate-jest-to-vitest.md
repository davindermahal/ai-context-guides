# Migrate from Jest (~v27) to Vitest (latest) in Node.js Projects

Produces: a Node.js project's test suite fully migrated off Jest (~v27.x) onto the current latest
Vitest major version, running on Node.js 24 LTS or later, with equivalent (or better) coverage
reporting, mocking, and snapshot behavior, and no remaining Jest dependency.

## When to use this guide

- The target project currently uses Jest around version 27.x (config format, defaults, and
  ecosystem packages consistent with that era — e.g. `testEnvironment` defaults to `"node"`, jsdom
  support may or may not require a separate `jest-environment-jsdom` package depending on the exact
  27.x patch) and wants to move to Vitest.
- Assumes the target project runs on **Node.js 24 LTS or later** — this guide does not hedge for
  older Node versions. If the target project is on an older Node, upgrade Node first (see
  `upgrade-node-22-to-node-24.md` / `upgrade-node-20-to-node-22.md` in this repo's `guides/` folder)
  before starting this migration, since the latest Vitest major has its own Node minimum (see
  Prerequisites below) that may not be satisfied otherwise.
- Covers plain Node.js/TypeScript projects and projects using a DOM testing environment (React,
  Vue, etc. component tests via jsdom/happy-dom). Does **not** specifically cover Vitest's browser
  mode (real-browser execution via Playwright/WebdriverIO) — that's a separate adoption decision
  from the core unit-test migration this guide handles, and pulls in its own additional breaking
  changes not covered here.
- Not the right guide if the source project is already on Jest 28/29/30 — later Jest versions
  changed some of the specifics this guide assumes about v27 (most notably, `jest-environment-jsdom`
  became a separate required package starting at Jest 28, not 27). If the source is Jest 28+, the
  overall migration approach still applies, but re-verify the "assumed Jest v27 default" callouts
  in the investigation checklist against the actual installed Jest version first.

## Prerequisites / assumptions

- The target project has a `package.json` with `jest` (or `@jest/core`/`ts-jest`/`babel-jest`) as a
  dependency, and Node.js 24.x or later is what's actually installed and used to run it — verify
  with `node --version`, don't assume from a version-pin file alone (it could be stale).
- The latest Vitest major at the time of this guide's research is **Vitest 5.0.0**, which requires
  **Vite ≥ 6.4.0** and **Node.js ≥ 22.12.0** — comfortably satisfied by Node 24 LTS, so this is not
  expected to block the migration, but re-verify the current latest Vitest version and its Node/Vite
  minimums at implementation time (they change with every major) rather than trusting this
  snapshot.
- The future agent must verify every fact below against the *actual* target project rather than
  assume:
  - Exact Jest version and config format in use (`jest.config.js`/`.ts`/`.json`/`.mjs`, or a
    `"jest"` key inside `package.json`).
  - Whether the project is CommonJS (`"type"` absent or `"commonjs"` in `package.json`) or ESM
    (`"type": "module"`) — this affects which Vitest config file extension and export syntax to use.
  - Whether TypeScript is in use, and via which transform (`ts-jest`, `babel-jest` +
    `@babel/preset-typescript`, or something else).
  - Whether the project is a monorepo with multiple Jest "projects" configured (`projects` array in
    Jest config) — Vitest's equivalent is `test.projects` (renamed from `workspace` in Vitest 4).

## Investigation checklist

Run every command below against the target project and record the result before planning changes.

| # | What to check | Command | What the result means |
|---|---|---|---|
| 1 | Exact installed Jest version | `npm ls jest --depth=0 2>/dev/null \|\| cat package.json \| grep '"jest"'` | Confirms the actual starting point. If it's not in the 27.x range, re-read the "When to use this guide" caveat about Jest 28+ before proceeding. |
| 2 | Jest config location and format | `ls jest.config.* 2>/dev/null; grep -A50 '"jest"' package.json 2>/dev/null` | Whichever exists is the full source of truth to translate. If config lives in `package.json`, the "jest" key's object is what needs translating; if config lives in `jest.config.js` and calls `require()`/uses dynamic logic (not a static object), read the actual resolved values by running `node -e "console.log(JSON.stringify(require('./jest.config.js')))"` rather than trying to hand-parse dynamic JS. |
| 3 | Project module system | `grep '"type"' package.json` | Absent or `"commonjs"` → use `vitest.config.ts` with `export default` (works regardless, since Vitest config files are always processed by Vite/esbuild — but match the project's existing config-file convention, e.g. if other config files in the repo are `.mjs`, follow that). `"module"` → same file works unchanged, ESM-native. |
| 4 | TypeScript transform in use | `grep -E '"ts-jest"|"babel-jest"|"@babel/preset-typescript"' package.json` | `ts-jest` present → it must be removed; Vitest transforms TypeScript natively via esbuild, no transform config needed. `babel-jest`+`@babel/preset-typescript` present → same, remove; only keep Babel if the project uses Babel for something Vitest/esbuild can't do (rare — e.g. certain experimental/stage-X JS proposals, or framework-specific Babel plugins like `@vue/babel-plugin-jsx` that have no esbuild equivalent, in which case keep `@vitejs/plugin-... ` + Babel as a Vite plugin instead, not as a Jest-style global transform). |
| 5 | `testEnvironment` setting | `grep -n "testEnvironment" jest.config.* package.json 2>/dev/null` | Not set → Jest 27 default is `"node"`; Vitest's default is also `"node"`, so no explicit setting needed unless DOM APIs are actually used (see item 6). Set to `"jsdom"` or `"happy-dom"` → must be replicated in Vitest as `test.environment`. |
| 6 | Actual DOM API usage despite no explicit `testEnvironment` | `grep -rln "document\.\|window\.\|navigator\." --include="*.test.js" --include="*.test.ts" --include="*.spec.js" --include="*.spec.ts" .` | Any match with no `testEnvironment: "jsdom"` set anywhere (including per-file `@jest-environment` docblocks — check those too, see item 7) is a latent bug already in the Jest suite (would throw `ReferenceError: document is not defined` under Jest 27's node default) — not something this migration introduces. Flag to the user rather than silently "fixing" by adding jsdom, since the existing test may simply be dead/skipped. |
| 7 | Per-file environment overrides | `grep -rln "@jest-environment" --include="*.test.js" --include="*.test.ts" .` | Jest supports a `/** @jest-environment jsdom */` docblock per file. Vitest's equivalent is a `// @vitest-environment jsdom` docblock (same mechanism, different comment tag) — translate each match. |
| 8 | `jest-environment-jsdom` package presence | `grep '"jest-environment-jsdom"' package.json` | Present → the project already installs jsdom as a separate package (true for Jest 28+, and for any Jest 27.x project that opted into jsdom explicitly); the equivalent Vitest package is `jsdom` (or `happy-dom`) installed directly, referenced via `test.environment: "jsdom"` — no `vitest-environment-jsdom` wrapper package is needed the way Jest needs one, Vitest imports `jsdom` directly. |
| 9 | `moduleNameMapper` entries | `grep -A20 "moduleNameMapper" jest.config.* package.json 2>/dev/null` | For each entry: a simple path-alias pattern (e.g. `"^@/(.*)$": "<rootDir>/src/$1"`) → move to Vite's `resolve.alias` in the same config file (Vitest reads it automatically, no duplication needed in `test.alias`). A pattern mapping file extensions to a mock module (e.g. `"\\.(css\|less\|scss)$": "identity-obj-proxy"`, or `"\\.svg$": "<rootDir>/__mocks__/svgMock.js"`) → CSS imports are handled natively by Vite in most configs (test first before assuming `identity-obj-proxy` is still needed); non-CSS asset mocks still need an equivalent `resolve.alias` entry pointing at the same mock file. |
| 10 | `transform` / custom transformers (non-TS, non-Babel) | `grep -A10 '"transform"' jest.config.* package.json 2>/dev/null` | Any entry besides the TS/Babel ones already covered in item 4 (e.g. a custom SVG-to-component transformer, a YAML loader) needs a Vite-plugin equivalent — search for `vite-plugin-<format>` for the specific asset type rather than assuming one doesn't exist. |
| 11 | `setupFiles` and `setupFilesAfterEnv` | `grep -E "setupFiles|setupFilesAfterEnv" jest.config.* package.json 2>/dev/null` | Both map to Vitest's single `test.setupFiles` array — Vitest doesn't distinguish Jest's two-phase (before-framework vs. after-framework) setup timing the same way. If both are set, concatenate into one array preserving relative order: `setupFiles` entries first, then `setupFilesAfterEnv` entries. Re-run the setup files' own code afterward (item in Implementation Plan) since anything relying on the specific Jest lifecycle timing between the two phases may behave subtly differently. |
| 12 | `collectCoverageFrom` / `coverageThreshold` / `collectCoverage` | `grep -A10 "collectCoverageFrom\|coverageThreshold\|collectCoverage" jest.config.* package.json 2>/dev/null` | `collectCoverageFrom` → `coverage.include`. `coverageThreshold` → `coverage.thresholds` (same shape: global + per-path glob keys). `collectCoverage: true` → not a Vitest config key; coverage is instead enabled by passing `--coverage` on the CLI (or setting `coverage.enabled: true` if it must always run). |
| 13 | Coverage provider currently used | `grep '"coverageProvider"' jest.config.* package.json 2>/dev/null; npm ls jest-environment-node babel-plugin-istanbul 2>/dev/null` | Jest defaults to (and usually uses) Istanbul-based coverage (`babel-plugin-istanbul` under the hood). Vitest defaults to the **v8** provider (faster, no instrumentation step, and as of Vitest ≥3.2 uses AST-aware remapping to match Istanbul-level accuracy) — this is a sensible default to keep; only install `@vitest/coverage-istanbul` and set `coverage.provider: "istanbul"` explicitly if the project specifically needs Istanbul's branch-counting behavior or a Istanbul-only reporter. Either way, `@vitest/coverage-v8` (or `-istanbul`) must be installed as a separate devDependency — Vitest does not bundle a coverage provider by default. |
| 14 | Global test API usage without imports | `grep -rLn "from 'vitest'\|from \"vitest\"" --include="*.test.js" --include="*.test.ts" . \| xargs grep -l "describe(\|it(\|test(\|expect(" 2>/dev/null \| head -5` | Jest injects `describe`/`it`/`test`/`expect`/etc. as globals; every existing test file relies on this with no import. Vitest does **not** enable globals by default. This is the single highest-edit-volume difference — see the Implementation Plan for the two ways to handle it (config-level `globals: true` vs. adding explicit imports to every file). |
| 15 | `jest.mock()` factory call sites | `grep -rn "jest\.mock(" --include="*.js" --include="*.ts" .` | For each call, open the file and check the factory function's return value (second argument). See the breaking-changes list ("Module Mocks Factory Return") below — Jest allows a factory to return the default export directly; Vitest requires an explicit `{ default: ... }` (or named-export) object. Also check the calls' **location**: Vitest 5 throws (not just warns) if `vi.mock()`/`vi.hoisted()` is called anywhere other than true module top level (not inside a `describe`, `beforeEach`, conditional, or helper function) — flag any `jest.mock()` call found inside such a block for special handling. |
| 16 | `jest.requireActual()` / `jest.requireMock()` usage | `grep -rn "jest\.requireActual(\|jest\.requireMock(" --include="*.js" --include="*.ts" .` | Replace `jest.requireActual(x)` with `await vi.importActual(x)` (note: becomes `async`, may require wrapping the enclosing function/test in `async`). Replace `jest.requireMock(x)` with `await vi.importMock(x)`. |
| 17 | `__mocks__` directory usage (manual mocks) | `find . -type d -name "__mocks__" -not -path "*/node_modules/*"` | Jest auto-loads modules from `<root>/__mocks__/<module>` for every test automatically when mocking node_modules packages (no explicit `jest.mock()` call needed for those). Vitest does **not** auto-load these — each one needs an explicit `vi.mock('<module-name>')` call (Vitest will then pick up the `__mocks__` file as the implementation), or add the mock to `test.setupFiles` if it truly needs to apply to every test file. |
| 18 | `done`-callback-style async tests | `grep -rn "it(.*,\s*(\?done)\s*=>\|test(.*,\s*(\?done)\s*=>\|it(.*,\s*function\s*(done)\|test(.*,\s*function\s*(done)" --include="*.js" --include="*.ts" .` | Vitest does not support the callback-style `(done) => {...}` test signature at all. Every match must be rewritten to return a `Promise` (see breaking-changes list, "Done Callback"). |
| 19 | Hooks (`beforeEach`/`afterEach`/etc.) returning a non-`undefined`/non-`null`, non-teardown-function value | `grep -rn "beforeEach(\|afterEach(\|beforeAll(\|afterAll(" -A3 --include="*.js" --include="*.ts" .` and manually inspect returns | Vitest treats a function returned from a hook as a teardown callback (a real, intentional feature) — if any existing hook happens to return some other value (e.g. accidentally returning the result of `setActivePinia(...)` or similar), Vitest will now try to call it as a teardown function, likely erroring. Wrap the hook body in `{ ... }` (block body, implicit-return-avoiding) if the return value isn't meant to be a teardown function. |
| 20 | Hook execution order dependencies across multiple `beforeEach`/`afterEach` calls in the same scope | Manual review of any test file with more than one `beforeEach`/`afterEach` at the same nesting level | Jest always runs same-level hooks in declaration order. Vitest's default (`sequence.hooks: 'stack'`) runs them in a different order (LIFO-like) unless configured otherwise. If test correctness depends on declared order (rare, but happens with setup/teardown pairs), set `test.sequence.hooks: 'list'` in the Vitest config to restore Jest's ordering globally, rather than restructuring every affected file. |
| 21 | Custom matchers via `expect.extend` with TypeScript ambient type declarations | `grep -rln "declare global" --include="*.d.ts" . \| xargs grep -l "jest.Matchers\|namespace jest" 2>/dev/null` | If the project augments `jest.Matchers<R>` for custom matcher types, this must be duplicated as an augmentation of Vitest's `Matchers<R, T>` interface (note: two type parameters in current Vitest, not one) — `expect.extend()` itself (the runtime registration) is unaffected and works identically. |
| 22 | `jest.setTimeout()` global call sites | `grep -rn "jest\.setTimeout(" --include="*.js" --include="*.ts" .` | Replace with `vi.setConfig({ testTimeout: <ms> })`, or move the value into `test.testTimeout` in the Vitest config file if it's a suite-wide constant rather than a dynamic per-run value. |
| 23 | `jest.replaceProperty()` usage | `grep -rn "jest\.replaceProperty(" --include="*.js" --include="*.ts" .` | Replace with `vi.stubEnv()` (for environment variables specifically) or `vi.spyOn(object, 'property', 'get')`/`vi.spyOn(object, 'property', 'set')` (for arbitrary object properties), depending on what's actually being replaced. |
| 24 | Snapshot files present | `find . -type d -name "__snapshots__" -not -path "*/node_modules/*" \| head; find . -name "*.snap" -not -path "*/node_modules/*" \| wc -l` | If any exist: Vitest reads the same `.snap` file format Jest uses (both built on `pretty-format`), so existing snapshots are expected to keep passing in most cases — but budget for a one-time diff/re-approval pass, since Vitest's `pretty-format` defaults occasionally differ from Jest's by whitespace/quoting, and the file header comment itself changes from `// Jest Snapshot v1` to `// Vitest Snapshot v1`. Treat any snapshot failure surfaced purely by this formatting difference as expected churn to re-approve (`vitest -u` / `vitest --update`), not a real regression — but confirm each diff is genuinely just formatting before re-approving, don't blanket-update without looking. |
| 25 | Custom snapshot matchers built on `jest-snapshot` internals | `grep -rn "require(['\"]jest-snapshot['\"])\|from ['\"]jest-snapshot['\"]" --include="*.js" --include="*.ts" .` | Rare, but if present: replace the `jest-snapshot` import with Vitest's `Snapshots` export (see breaking-changes list, "Custom Snapshot Matchers") — the shape of the custom matcher function changes slightly (`toMatchSnapshot.call(this, ...)` pattern is preserved, but sourced from a different module). |
| 26 | Framework-specific snapshot serializers (e.g. `jest-serializer-vue`) | `grep -E "jest-serializer-|snapshotSerializers" jest.config.* package.json 2>/dev/null` | The serializer package itself is typically framework-tool-agnostic and still installable; move the `snapshotSerializers` array into the Vitest config's `test.snapshotSerializers` key unchanged. |
| 27 | Monorepo / multi-project Jest config (`projects` array) | `grep -A20 '"projects"' jest.config.* package.json 2>/dev/null` | If present, Vitest's equivalent config key is `test.projects` (note: this was named `workspace`/`vitest.workspace.js` in older Vitest — if any tutorial or existing partial-migration artifact in the repo uses `workspace`, that's stale advice predating Vitest 4; use `test.projects` in the root `vitest.config.ts` instead, and inline project definitions there rather than a separate workspace file, since separate workspace files were removed as a first-class concept). |
| 28 | CI/build scripts invoking Jest | `grep -rn "jest" package.json .github/workflows/*.yml .gitlab-ci.yml Jenkinsfile 2>/dev/null` | Every match (npm scripts like `"test": "jest"`, CI steps running `npx jest`, coverage-upload steps referencing Jest's coverage output path) needs updating to the Vitest equivalent command and output paths (see Implementation Plan). |
| 29 | `--ci`, `--runInBand`, `--maxWorkers`, or other Jest-specific CLI flags in scripts/CI | `grep -rn "jest --\|jest -" package.json .github/workflows/*.yml .gitlab-ci.yml 2>/dev/null` | Translate flag-by-flag: `--runInBand` → `--pool=forks --poolOptions.forks.singleFork` (or simply omit if not strictly needed — Vitest's default parallelism is usually fine in CI); `--maxWorkers=N` → `--maxWorkers=N` (same flag name, still supported); `--ci` → no direct Vitest equivalent needed, Vitest already behaves non-interactively when not in a TTY. |
| 30 | Current Node.js version actually running tests | `node --version` | Must be ≥ 24.x per this guide's assumption, and specifically ≥ the exact minimum the installed Vitest major requires (checklist item re: Prerequisites — re-verify the current Vitest minimum at implementation time, since it rises with each major). If below the requirement, stop and upgrade Node first. |

## Jest → Vitest differences (complete reference)

This is a migration between two different tools, not sequential versions of one package — so unlike
a same-package upgrade, there's no meaningful "intermediate version" the target project passes
through. The equivalent of "read every changelog" here is: (A) the complete Jest-API-compatibility
differences from Vitest's own official Jest-migration documentation, and (B) the most recent
Vitest-major-version-specific defaults, since a lot of third-party "Jest to Vitest" tutorials found
online were written against Vitest 1–3 and give stale advice that no longer matches Vitest 5's
actual defaults. Both are covered in full below — nothing here is a curated "top changes" shortlist.

### A. Jest-compatibility differences (from Vitest's official Jest migration guide)

| Difference | Jest behavior | Vitest behavior | How to check target project | What to do |
|---|---|---|---|---|
| Globals | `describe`/`it`/`test`/`expect`/`jest` available with no import | Not global by default | Checklist item 14 | Either set `test.globals: true` in Vitest config (fastest path, minimal edits) or add `import { describe, it, test, expect, vi } from 'vitest'` to every test file (Vitest team's recommended long-term practice, more edits now) — see Implementation Plan step 5 for the tradeoff and a recommended default. |
| `mock.mockReset()` | Replaces implementation with an empty function returning `undefined` | Resets implementation back to whatever was passed to `vi.fn(impl)` originally | `grep -rn "\.mockReset(" --include="*.js" --include="*.ts" .` | If a test relies on `mockReset()` producing an empty/no-op function (not just clearing call history), that behavior is gone — use `.mockImplementation(() => undefined)` explicitly afterward instead. |
| `mock.mock` property persistence | A fresh state object is effectively produced around `.mockClear()` | Vitest holds a persistent reference across `.mockClear()` | Only matters if code holds a reference to `fn.mock` before calling `.mockClear()` and compares object identity afterward (rare) | No action needed for typical usage (reading `.mock.calls`, `.mock.results` after the fact); only revisit if an identity-comparison test exists |
| `jest.mock()` factory return shape | Factory can return the default export value directly: `jest.mock('./x', () => 'hello')` | Factory must return an object with explicit export keys: `vi.mock('./x', () => ({ default: 'hello' }))` | Checklist item 15 | Wrap every factory's return value in `{ default: ... }` (or the correct named-export shape if the mocked module has named exports) |
| Auto-mocking from `__mocks__` | Automatically applied to every test when mocking a node_modules package, no explicit call needed | Never auto-applied; requires an explicit `vi.mock('module-name')` call per file, or placement in `setupFiles` for suite-wide effect | Checklist item 17 | Add explicit `vi.mock()` calls, or move to `setupFiles` if truly global |
| `jest.requireActual()` | Synchronous | `vi.importActual()` — **asynchronous** | Checklist item 16 | Add `await`, and make the enclosing function `async` if it wasn't already |
| Mocking modules imported by another mocked module (transitive) | Handled implicitly in most cases | May require explicitly listing the module in `server.deps.inline: ["module-name"]` for the mock to take effect through the dependency chain | Only relevant if a mock silently doesn't take effect for a module imported *by* another module (not directly by the test file) | Add the module name to `test.server.deps.inline` |
| Test name display / `-t` filter format | `${describe title} ${test title}` (space-joined) | `${describe title} > ${test title}` (`>`-joined) | `grep -rn "\-t '" package.json .github/workflows/*.yml 2>/dev/null` for any hardcoded `-t` filter strings in scripts/CI | Update any hardcoded `-t 'some string'` filters to use ` > ` as the separator matching the new format |
| Worker-ID environment variables | `JEST_WORKER_ID` | `VITEST_POOL_ID` (bounded by `maxWorkers`, closest equivalent) and `VITEST_WORKER_ID` (unique per worker, not bounded) | `grep -rn "JEST_WORKER_ID" --include="*.js" --include="*.ts" .` | Replace with whichever of the two matches the original intent (usually `VITEST_POOL_ID` for "which of N slots am I") |
| `jest.replaceProperty()` | Built-in property-replacement helper | No direct equivalent; use `vi.stubEnv()` or `vi.spyOn(obj, 'prop', 'get'/'set')` | Checklist item 23 | Rewrite per the specific property being replaced |
| Callback-style async tests (`done`) | Supported: `it('x', (done) => { ...; done(); })` | **Not supported at all** | Checklist item 18 | Rewrite as `it('x', () => new Promise((done) => { ...; done(); }))`, or better, convert to a real `async`/`await` test if the underlying operation supports it |
| Hook return values | Hooks always run as plain functions; a returned function has no special meaning | A function returned from a hook is treated as a teardown callback | Checklist item 19 | Wrap hook bodies in `{ }` unless a teardown function is intentionally being returned |
| Hook execution order (same-level, multiple hooks) | Sequential, declaration order | Default is a "stack" order (not declaration order) unless configured | Checklist item 20 | Set `test.sequence.hooks: 'list'` in Vitest config to restore Jest's ordering, if order-dependence exists |
| `jest` TypeScript namespace | `jest.Mock<T>` etc. available as ambient types | No `jest` namespace; import types directly from `vitest` | `grep -rn "jest\.Mock\|jest\.Mocked\|jest\.SpyInstance" --include="*.ts" .` | Replace with `import type { Mock, Mocked, MockInstance } from 'vitest'` and the corresponding type names |
| Legacy (pre-modern) fake timers | Supported as an opt-in legacy implementation | **Not supported** — only the modern (`@sinonjs/fake-timers`-based) implementation exists | `grep -rn "legacyFakeTimers\|timers:\s*['\"]legacy['\"]" jest.config.* --include="*.js" --include="*.ts" .` | If the project explicitly opted into legacy timers, the fake-timer-dependent tests need re-verification against modern-timer semantics — there is no toggle to fall back to |
| `jest.setTimeout()` | Global timeout setter | `vi.setConfig({ testTimeout: ms })`, or a static `test.testTimeout` config value | Checklist item 22 | Replace call sites; prefer the static config value if the timeout is a suite-wide constant |
| Framework-specific snapshot serializers (e.g. Vue) | Installed and referenced the same way | Same mechanism, same config key (`snapshotSerializers`) | Checklist item 26 | No structural change — just confirm the serializer package itself still works under Vitest (most do, since they operate on `pretty-format`) |
| Custom snapshot matchers built on `jest-snapshot` | `require('jest-snapshot')` | `import { Snapshots } from 'vitest'` | Checklist item 25 | Swap the import source; the `.call(this, ...)` invocation pattern is preserved |

### B. Vitest-version-specific defaults to know about (v3→v4, v4→v5)

These matter specifically because a lot of "how to migrate from Jest" content already on the web
was written against Vitest 1–3 and will give advice that no longer works or no longer matches
current defaults. Everything below is current as of Vitest 5.0.0 (this guide's research date);
re-verify against the actual installed Vitest version at implementation time.

| Change | Introduced in | Old vs. new | How to check if relevant | What to do |
|---|---|---|---|---|
| `clearMocks` defaults to `true` | Vitest 5.0 | Previously (Vitest ≤4, and Jest, both default this to `false`) mock call history persisted across tests unless cleared explicitly; now Vitest clears it automatically before every test | Any test asserting on a mock's call count/arguments that depends on history accumulated across multiple `it()` blocks without an explicit `vi.clearAllMocks()` already in a `beforeEach` | If such cross-test accumulation is intentional (unusual, but possible), set `clearMocks: false` in the Vitest config to restore prior/Jest-matching behavior; otherwise this default change is likely desirable and needs no action |
| `vi.mock()`/`vi.unmock()`/`vi.hoisted()` must be at true module top level | Vitest 5.0 | Previously calling these inside a function/block/conditional only warned; now it throws | Checklist item 15 (already covers this) | Move any such call found inside a function/block to the actual top level of the file |
| `workspace` config renamed to `projects` | Vitest 4.0 | A separate `vitest.workspace.js` file is no longer a first-class concept | Checklist item 27 | Use `test.projects` inline in `vitest.config.ts`; ignore any tutorial referencing `vitest.workspace.js` |
| `coverage.all` and `coverage.extensions` removed | Vitest 4.0 | Coverage previously could include all matched files regardless of whether they were loaded during tests (`coverage.all: true`); now only files actually loaded during the run are included unless `coverage.include` is explicitly set | `grep -A5 "coverage" jest.config.* 2>/dev/null` won't show this (it's Jest→Vitest net-new), but check any *existing partial Vitest config* in the repo for `coverage.all` | If 100%-of-source coverage reporting (including untested files) is required, explicitly set `coverage.include` to the full source glob |
| Default `exclude` simplified | Vitest 4.0 | Previously excluded common build/tool directories (`dist`, `cypress`, etc.) automatically; now only `node_modules` and `.git` are excluded by default | Only matters if the project has build output or unrelated tooling directories inside the repo that could be picked up as test files | Add an explicit `test.exclude` (or narrow `test.include`) covering those directories, don't rely on the old implicit exclusions |
| `maxThreads`/`maxForks`/`singleThread`/`singleFork`/`poolOptions` consolidated | Vitest 4.0 | Replaced by a single top-level `maxWorkers` option and `isolate` flag | Checklist item 29 (CI flags) | Translate any of these old option names found in an existing partial Vitest config to `maxWorkers`/`isolate` |
| `spyOn`/`fn` now support real constructor semantics with `new` | Vitest 4.0 | A mock implementation called with `new` previously just called the function; now it actually constructs an instance — but only if the implementation is declared with `function`/`class` syntax, not an arrow function | `grep -rn "vi\.fn(.*=>.*)" --include="*.js" --include="*.ts" .` combined with checking whether any such mock is ever invoked with `new` elsewhere | If a mock is invoked with `new` anywhere, ensure its implementation uses `function`/`class`, not an arrow function, or it will throw "is not a constructor" |
| `resolveConfig()` return shape changed | Vitest 5.0 | Previously `{ vitestConfig, viteConfig }`; now returns the resolved Vite config directly, with Vitest options under its `test` property | Only relevant if the project has custom tooling/scripts calling Vitest's Node API (`resolveConfig`) directly — rare | Update the destructuring at each call site |
| Coverage ignore hints and AST-aware remapping changes | Vitest 4.0 | `coverage.ignoreEmptyLines` removed (lines without runtime code no longer appear); `coverage.experimentalAstAwareRemapping` removed because it's now always-on | Only relevant if an existing partial Vitest config sets either of these two keys | Remove both keys; AST-aware remapping (V8-provider accuracy matching Istanbul) is now unconditional |

## Implementation plan

1. **Confirm exact latest Vitest version and its Node/Vite minimums** (Prerequisites section) —
   `npm view vitest version` — and confirm the installed Node (checklist item 30) satisfies it
   before installing anything.
2. **Install Vitest and remove Jest**, in that order (install first so `package.json` always has a
   working test runner between the two operations, in case something needs to be checked
   mid-migration):
   ```bash
   npm install --save-dev vitest @vitest/coverage-v8
   npm uninstall jest ts-jest babel-jest @types/jest jest-environment-jsdom
   ```
   Only uninstall the packages actually present (checklist items 1, 4, 8) — don't blindly run all
   of these if, say, `ts-jest` was never installed.
3. **If the project uses jsdom** (checklist items 5–8): `npm install --save-dev jsdom` (or
   `happy-dom`, if the user prefers it — `happy-dom` is generally faster but has small DOM-API
   coverage gaps versus `jsdom`; default to `jsdom` unless the user has a reason to prefer
   `happy-dom`, since it's the more drop-in-compatible choice coming from Jest).
4. **Create `vitest.config.ts`** (or `.js`/`.mts` matching the project's existing config-file
   convention — checklist item 3) at the repo root, translating every Jest config key found in
   checklist items 2, 5, 9–13, 26–27 using the mapping table in the breaking-changes section above.
   Illustrative shape (adapt keys/values to what was actually found, don't copy verbatim):
   ```typescript
   import { defineConfig } from 'vitest/config'

   export default defineConfig({
     test: {
       environment: 'node', // or 'jsdom' — from checklist items 5-6
       globals: true, // see step 5 below for the tradeoff this implies
       setupFiles: ['./test/setup.ts'], // merged from setupFiles + setupFilesAfterEnv, checklist item 11
       coverage: {
         provider: 'v8', // or 'istanbul' — from checklist item 13
         include: ['src/**/*.{js,ts}'], // from collectCoverageFrom, checklist item 12
         thresholds: { /* from coverageThreshold, checklist item 12 */ },
       },
     },
     resolve: {
       alias: {
         // translated from moduleNameMapper path-alias entries, checklist item 9
       },
     },
   })
   ```
5. **Decide on the globals strategy** (checklist item 14) — this is the single biggest lever on
   total edit volume, so decide before doing any per-file editing:

   | Condition | Choice |
   |---|---|
   | Migration needs to happen fast, or the codebase is large (hundreds of test files) | Set `test.globals: true` in the config above; add `"types": ["vitest/globals"]` to `tsconfig.json`'s `compilerOptions` if using TypeScript. Zero per-file edits needed for `describe`/`it`/`test`/`expect`. |
   | The project explicitly wants to move toward Vitest's recommended explicit-import style, or already has an ESLint rule banning undeclared globals | Add `import { describe, it, test, expect, vi, beforeEach, afterEach } from 'vitest'` (only the ones actually used per file) to every test file, and leave `globals` unset (`false`, the default). Higher edit volume, no config-level toggle to maintain. |

   If the project uses `@testing-library/*` packages with automatic DOM cleanup (their `afterEach`
   auto-cleanup hook depends on the globals config matching), keep globals consistent — don't set
   `test.globals: true` for the test runner while a testing-library setup file assumes explicit
   imports, or vice versa; check the testing-library setup file's own import style and match it.
6. **Run the official Jest→Vitest codemod for the mechanical renames**, then hand-fix everything
   the codemod can't infer:
   ```bash
   npx codemod jest/vitest
   ```
   This handles straightforward renames (`jest.fn()`→`vi.fn()`, `jest.spyOn()`→`vi.spyOn()`,
   `jest.mock()`→`vi.mock()`, import insertion if globals are disabled). It does **not** reliably
   handle: the `jest.mock()` factory-return-shape change (checklist item 15), `done`-callback
   rewrites (checklist item 18), or hook-return-value issues (checklist item 19) — these need manual
   review using the checklist items as the search list.
7. **Fix every checklist item flagged as "affected" during investigation**, in this order (earlier
   items are more likely to cause immediate hard failures, so fix them first to get the suite
   running at all before chasing subtler behavioral differences):
   1. `jest.mock()` factory return shapes (checklist item 15) and top-level-only placement (same
      item) — these are compile/collection-time failures, nothing runs until these are fixed.
   2. `done`-callback tests (checklist item 18) — these hang/timeout rather than erroring clearly,
      so they're worth fixing early to avoid confusing timeout noise while debugging other issues.
   3. `__mocks__` auto-mocking (checklist item 17) and `jest.requireActual`/`requireMock` (item 16).
   4. Hook return-value and ordering issues (checklist items 19–20).
   5. Everything else (timeouts, custom matchers, snapshot serializers, coverage config) — these
      tend to surface as individual test failures rather than whole-suite breakage, so they're safe
      to work through incrementally once the suite is at least running.
8. **Update `package.json` scripts and CI** (checklist items 28–29): replace `jest`/`jest --coverage`
   invocations with `vitest run`/`vitest run --coverage` (use `vitest run`, not bare `vitest`, in
   CI and one-shot scripts — bare `vitest` starts in interactive watch mode by default, which hangs
   a CI job). Update any coverage-report-upload step to point at Vitest's coverage output location
   (`coverage/` by default, same as Jest's default — verify this wasn't overridden in either tool's
   config).
9. **Run the full suite and re-approve snapshot formatting churn** (checklist item 24): run once,
   review every snapshot diff, and run `vitest run -u` only after confirming each diff is pure
   formatting noise (header comment, quote style) and not a real behavioral regression introduced by
   this migration.

## Verification steps

- `npx vitest run` completes with the same number of passing tests as `npx jest` did on the
  pre-migration codebase (compare total counts, not just "no failures" — a silently-skipped or
  never-collected file is easy to miss otherwise).
- Coverage percentage is comparable to the pre-migration Jest report (checklist item 13) — a large
  unexplained drop usually means `coverage.include` doesn't match as many files as
  `collectCoverageFrom` did, not an actual coverage regression.
- Every snapshot diff reviewed in Implementation Plan step 9 was confirmed as formatting-only before
  being re-approved.
- CI pipeline runs `vitest run` (not bare `vitest`) and completes non-interactively.
- If the project has DOM/component tests: manually spot-check a handful under the chosen environment
  (`jsdom`/`happy-dom`) rather than trusting a green run alone — environment-specific API gaps
  (particularly with `happy-dom`) can pass trivial assertions while missing real behavior.
- Grep the repo once more for `jest\.` and bare `require\(['"]jest['"]\)` and confirm zero remaining
  references outside of this migration's own history/changelog entries.

## Rollback / risk notes

- Do the migration on a dedicated branch and keep the pre-migration `jest.config.*` and
  `package.json` (via git history) until the new suite has been green in CI at least once — reverting
  is a straightforward branch revert as long as Jest wasn't uninstalled from a shared lockfile state
  other branches depend on.
- The riskiest, easy-to-miss failure mode is a test that Vitest silently doesn't collect at all
  (e.g. a file-naming pattern mismatch between Jest's `testMatch`/`testRegex` and Vitest's default
  `include` glob) — a suite that "passes" with fewer total tests than before is a false green, not a
  successful migration. Always compare total test counts (Verification steps), not just pass/fail
  status.
- If the globals decision (Implementation Plan step 5) is revisited later (e.g. switching from
  `globals: true` to explicit imports), treat that as a separate, lower-urgency follow-up cleanup —
  don't block the initial migration on getting that stylistic choice "right" the first time.
- Keep `@vitest/coverage-v8` (or `-istanbul`) version in lockstep with the `vitest` core version —
  Vitest's coverage packages are versioned to match the core package and mismatches can produce
  confusing errors unrelated to actual coverage configuration.

## References

- [Vitest — Migrating from Jest](https://vitest.dev/guide/migration/jest) — official Jest-compatibility differences guide; fetched in full, forms the basis of section A above.
- [Vitest — Migration Guide (general, current major)](https://vitest.dev/guide/migration/) — official v4→v5 breaking-changes list; fetched in full.
- [Vitest 4.0 migration guide](https://v4.vitest.dev/guide/migration.html) — official v3→v4 breaking-changes list; fetched in full, since v4-era changes (`workspace`→`projects`, pool/coverage option consolidation) are still commonly referenced in slightly older tutorials.
- [Vitest 5.1 blog post](https://vitest.dev/blog/vitest-4-1.html) — cross-checked for currency of "latest version" claims.
- [Vitest coverage guide](https://vitest.dev/guide/coverage.html) and [vitest-dev/vitest coverage discussion #7587](https://github.com/vitest-dev/vitest/discussions/7587) — consulted for the v8-vs-istanbul default-provider rationale (checklist item 13).
- [Jest 27 release blog — "New Defaults for Jest, 2021 edition"](https://jestjs.io/blog/2021/05/25/jest-27) — consulted to confirm the exact Jest v27 default-environment change and its rationale (investigation checklist items 5–8).
- [Jest — Test Environment docs](https://jestjs.io/docs/test-environment) and the "From v27 to v28" upgrade notes — consulted to confirm `jest-environment-jsdom` became a separate required package starting at Jest 28, not 27 (the "When to use this guide" caveat).
- Community codemod: [`codemod jest/vitest`](https://app.codemod.com/registry/jest-to-vitest) (Codemod Registry) — verified as an actively maintained automated first-pass tool for the mechanical renames in Implementation Plan step 6.

**Assumptions flagged for user review:**
- "Latest Vitest" is pinned to **5.0.0** as of this guide's research (~September 2026). Re-verify
  via `npm view vitest version` at implementation time — if a newer major has since shipped, fetch
  its own migration-guide breaking changes and layer them onto section B above rather than assuming
  nothing changed since 5.0.0.
- The exact wording/existence of Jest's "New Defaults" 27.0.0 behavior was confirmed via Jest's own
  blog post; the more specific claim that `jest-environment-jsdom` remained *bundled* (not yet a
  separate package) specifically at v27 (vs. becoming separate at v28) is based on secondary
  sourcing (a search-engine synthesis of Jest's docs/release notes) rather than a directly fetched
  primary changelog entry for that exact detail — investigation checklist item 8 is written to have
  the future agent verify this directly against the actual target project rather than trust this
  guide's claim blindly.
