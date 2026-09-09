# Upgrade Node.js 20.x to Node.js 22.x (LTS)

Produces: a Node.js project (any framework/runtime target — server, CLI, build tooling) migrated
from the Node 20.x line to the Node 22.x LTS line, with all breaking changes between the two lines
addressed, native dependencies rebuilt for the new ABI, and CI/deploy configs updated to match.

## When to use this guide

- The target project currently runs on any Node 20.x version (20.0.0–20.20.2, the final 20.x
  release) and needs to move to Node 22.x (LTS codename "Jod").
- Covers the full jump, including everything that changed in the intermediate Node 21.x line
  (21.0.0–21.7.3) — Node 22 ships with all of Node 21's changes plus its own, even though a project
  moving straight from 20→22 never runs 21 directly.
- **Not** the right guide for a from-scratch Node.js project (no upgrade path needed) or for
  upgrading from Node 22 to Node 24 (a separate jump with its own breaking-changes set — see
  `nodejs.org/en/blog/migrations/v22-to-v24` if that guide is ever needed).
- Node 20 reached End-of-Life on 2026-04-30 — if the target project is still on 20.x, this upgrade
  is not optional from a security-support standpoint.

## Prerequisites / assumptions

- The target project has a `package.json` (Node.js project of some kind — server, CLI, library,
  build tooling). Verify this is true before proceeding; if there's no `package.json`, this guide
  doesn't apply.
- The future agent must verify every fact below against the *actual* target project — nothing here
  should be assumed true just because it's common:
  - Current exact Node version(s) used in development, CI, and production may differ from each
    other — check all three.
  - The project may or may not use TypeScript, ESM (`"type": "module"`), native addons, or a
    process manager/container runtime — all of which change which steps below apply.

## Investigation checklist

Run every command below against the target project and record the result before planning changes.

| # | What to check | Command | What the result means |
|---|---|---|---|
| 1 | Current Node version pinned for local dev | `cat .nvmrc 2>/dev/null; cat .node-version 2>/dev/null; grep -A2 '"volta"' package.json 2>/dev/null` | Whichever file(s) exist define the dev-time pinned version — update all that are found, don't assume only one mechanism is in use. |
| 2 | Current Node version constraint in `package.json` | `grep -A3 '"engines"' package.json` | If an `engines.node` range exists (e.g. `"20.x"`, `">=18"`), it must be updated to require `22.x` (or `>=22 <23` if the project intends to also stay compatible with 24+, but confirm that intent with the user rather than assuming it). |
| 3 | Node version used in CI | `grep -rn "node-version\|NODE_VERSION\|node:20\|node:2[01]" .github/workflows/*.yml .gitlab-ci.yml azure-pipelines.yml Jenkinsfile 2>/dev/null` | Every match naming `20` (or `21`) must be bumped to `22`. A CI matrix testing multiple Node versions may intentionally keep 20 as a compatibility target — confirm with the user before removing it rather than assuming it should go. |
| 4 | Docker base image | `grep -rn "^FROM node" Dockerfile* 2>/dev/null` | A tag like `node:20`, `node:20-alpine`, `node:20-slim`, `node:20-bookworm` etc. must become the equivalent `node:22...` tag. Prefer pinning to a specific `22.x.y` digest/tag matching what's decided in the investigation, not a bare `node:22` (mutable) tag, if the project already pins precise versions elsewhere. |
| 5 | Serverless/PaaS runtime declarations | `grep -rn "nodejs20\|nodejs18\|runtime:" serverless.yml template.yaml 2>/dev/null; cat vercel.json netlify.toml 2>/dev/null` | AWS Lambda's `nodejs20.x` runtime string, or Vercel/Netlify Node version settings, must be updated to their `22` equivalents. Verify the target platform actually offers a Node 22 runtime before committing to this (it does for AWS Lambda, Vercel, and Netlify as of this guide's research — but confirm current platform support, since managed-runtime support lags Node's own release by weeks to months). |
| 6 | asdf / mise version managers | `cat .tool-versions 2>/dev/null` | A `nodejs 20.x.y` line must be bumped to a `22.x.y` line. |
| 7 | TypeScript type definitions | `grep '"@types/node"' package.json` | If present, bump to `^22` to match the new runtime's types — mismatched `@types/node` majors can mask or fabricate type errors unrelated to real behavior. |
| 8 | Native addons (anything requiring a C++ rebuild) | `grep -l "node-gyp\|prebuild" package-lock.json 2>/dev/null; find . -maxdepth 3 -name "*.node" 2>/dev/null; grep -E '"(bcrypt|sharp|sqlite3|better-sqlite3|canvas|node-sass|grpc|@grpc/grpc-js|puppeteer-core|serialport)"' package.json` | Any match is a native addon. Node's ABI version (`NODE_MODULE_VERSION`) changed between 20 and 22 (120 in early 21.x, up to 127 by 22.0.0) — prebuilt binaries built for Node 20's ABI will not load on Node 22 without either a matching prebuilt release for Node 22, or a local rebuild (`npm rebuild` after switching the running Node version). Plan a rebuild/reinstall step for each match. |
| 9 | Package manager lockfile version + npm major | `node -pe "require('./package-lock.json').lockfileVersion" 2>/dev/null; npm --version` | Node 20 bundled npm ~9.6.4; Node 22 bundles npm ~10.5.1+ (some 22.x patch releases bundle npm 11). A bundled npm major bump can itself change `npm install`/`npm ci` behavior — check npm's own release notes for the specific old→new npm major jump if `npm ci` behavior changes unexpectedly after the Node upgrade. |
| 10 | Use of `import ... assert { type: 'json' }` | `grep -rn "assert { type:" --include="*.js" --include="*.mjs" --include="*.ts" .` | Any match must be changed to `with { type: 'json' }` (see breaking-changes list, Node 22.0.0) — `import assert` was removed, not just deprecated. |
| 11 | Use of `crypto.createCipher(` / `crypto.createDecipher(` | `grep -rn "createCipher(\|createDecipher(" --include="*.js" --include="*.ts" .` | Any match is calling APIs fully removed in Node 22.0.0 (moved from deprecated to EOL). Must be rewritten to `createCipheriv`/`createDecipheriv` with explicit key derivation and IV (see breaking-changes entry below for the exact replacement pattern). |
| 12 | Use of legacy `util.is*` helpers and `util._extend`/`util.log` | `grep -rnE "util\.(isArray|isBoolean|isBuffer|isDate|isError|isFunction|isNull|isNullOrUndefined|isNumber|isObject|isPrimitive|isRegExp|isString|isSymbol|isUndefined|_extend|log)\(" --include="*.js" --include="*.ts" .` | Any match now prints a runtime `DeprecationWarning` on every call under Node 22 (not removed yet, but noisy and slated for future removal). Replace with the modern equivalent (e.g. `Array.isArray()` for `util.isArray`, `Object.assign({}, ...)` for `util._extend`, `console.log`/a logger for `util.log`). |
| 13 | Use of `crypto.Hash`/`crypto.Hmac` as constructors (`new crypto.Hash(...)`, `new crypto.Hmac(...)`) | `grep -rn "new crypto.Hash(\|new crypto.Hmac(\|new Hash(\|new Hmac(" --include="*.js" --include="*.ts" .` | Runtime-deprecated in Node 22.0.0. Replace with the factory functions `crypto.createHash(...)` / `crypto.createHmac(...)`, which were always the documented public API. |
| 14 | Use of `node:punycode` | `grep -rn "require(.punycode.)\|from ['\"]punycode['\"]\|from ['\"]node:punycode['\"]" --include="*.js" --include="*.ts" .` | Runtime-deprecated since Node 21.0.0 (prints a warning on every `require`/`import`). Replace with the userland `punycode` npm package (`npm install punycode`) — same API, actively maintained outside Node core. |
| 15 | Reliance on the default stream `highWaterMark` | `grep -rn "highWaterMark" --include="*.js" --include="*.ts" .` | Not a required change, but the default rose from 16KiB to 64KiB in Node 22.0.0 — any code or test that assumed the old 16KiB default (buffering thresholds, backpressure-timing-sensitive tests, memory budgets in memory-constrained environments) should be reviewed. If memory-sensitive, explicitly pass `highWaterMark` (or use `require('stream').setDefaultHighWaterMark(size)`, added alongside this change) rather than relying on the default. |
| 16 | Reliance on the global `Iterator` not existing | `grep -rn "class Iterator\|var Iterator\b\|let Iterator\b\|const Iterator\b" --include="*.js" --include="*.ts" .` | Node 22 (via the updated V8) adds a global `Iterator` object. Any code defining its own top-level `Iterator` identifier will now collide/shadow the built-in in that scope — check for naming collisions, not usually a real-world issue but worth a grep pass. |
| 17 | Dual-package (CJS+ESM) `exports` maps in any package the project *authors* (not just consumes) | `grep -A20 '"exports"' package.json` | If the project publishes its own package with dual CJS/ESM exports, consider adding the new `"module-sync"` condition (Node 22.10.0+) to avoid the dual-package hazard once `require(esm)` is broadly available — optional, not required for the upgrade to succeed, so treat as a follow-up recommendation rather than a blocking step. |
| 18 | `net.createServer`/`http.Server` code relying on `maxConnections = 0` meaning unlimited | `grep -rn "maxConnections" --include="*.js" --include="*.ts" .` | Since Node 21.0.0, `server.maxConnections = 0` now means "reject all connections," not "unlimited" (previous, arguably buggy, behavior). If any code sets this to `0` expecting unlimited connections, it must be changed (remove the assignment, or set it to a very large number / `Infinity` explicitly is not supported — omit the property instead). |
| 19 | Use of `stream.Writable`/`Readable`/`Transform` internals via non-public symbols | `grep -rn "_writableState\.\|_readableState\.\|Symbol\.for('nodejs" --include="*.js" --include="*.ts" .` | Node 21.0.0 moved several internal stream properties (encoding, compression, strategies) to private class fields. Code touching stream internals directly (rare, but happens in low-level libraries) may break; public stream APIs are unaffected. |
| 20 | HTTP/2 priority signaling usage | `grep -rn "\.priority(\|setPriority\|http2.*priority" --include="*.js" --include="*.ts" .` | Deprecated in Node 22.17.0, and support fully removed in Node 22.23.0. If the target Node 22.x patch is ≥ 22.23.0, this API is gone, not just deprecated — check the exact patch version being installed (checklist item 21) before assuming a deprecation warning is all that happens. |
| 21 | Exact Node 22.x patch version to install | `npm view node-lts version 2>/dev/null || echo "check https://nodejs.org/en/download for current 22.x LTS patch"` | Pin to the current latest 22.x patch at implementation time — this guide's research (mid-2026) found 22.23.2 as latest, but Node ships patches frequently; always re-verify rather than hardcoding this guide's snapshot. |

## Breaking changes, version by version

Node's release line runs 20 → 21 (Current-only, never LTS) → 22 (LTS). A project jumping straight
from 20.x to 22.x is affected by everything introduced in *both* 21.x and 22.x, cumulatively — Node
22 contains all of 21's changes.

### Node 21.0.0 (2023-10-17)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| `fs` position validation tightened | Passing an invalid `position` argument to `fs.read`/`fs.write` family methods now throws where it previously silently misbehaved | Checklist item covers general fs usage; review any direct `fs.read(fd, buffer, offset, length, position, ...)` calls with computed/dynamic `position` values | Ensure `position` is a valid integer or `null`/`-1` per docs; add validation before the call if the value is computed |
| `events`: `on`/`once` option validation | `EventEmitter`/`EventTarget`'s `on`/`once` now validate their `options` argument and throw on invalid input instead of ignoring it | `grep -rn "\.on(.*,.*{.*}\|\.once(.*,.*{.*}" --include="*.js" .` and inspect any options objects passed | Fix any malformed options objects (e.g., wrong key names) surfaced by the new validation |
| `net`: `maxConnections = 0` | Previously treated as `Infinity` (unlimited); now means "reject all connections" | Checklist item 18 | Remove the `= 0` assignment if unlimited connections was the intent |
| `lib`: URL/URLSearchParams uncloneable | `structuredClone(new URL(...))` or postMessage-transferring a `URL`/`URLSearchParams` now throws `DataCloneError` instead of silently producing a broken clone | `grep -rn "structuredClone(.*URL\|postMessage(.*URL" --include="*.js" .` | Serialize to string (`url.toString()`) before cloning/transferring, reconstruct with `new URL(...)` on the other side |
| `punycode` runtime-deprecated | `require('punycode')` / `require('node:punycode')` now prints a `DeprecationWarning` on every use (not removed) | Checklist item 14 | Switch to the userland `punycode` npm package |
| `util.promisify` runtime-deprecated for double-Promise functions | Calling `util.promisify()` on a function that already returns a Promise now warns | `grep -rn "promisify(" --include="*.js" .` and check whether the wrapped function is `async`/already Promise-returning | Remove the unnecessary `promisify()` wrapper — the function is already awaitable |
| `tls`: stricter option validation | `options.minDHSize` and `options.checkServerIdentity` now use strict type validators and throw `TypeError` on bad input instead of behaving unpredictably | `grep -rn "minDHSize\|checkServerIdentity" --include="*.js" .` | Ensure `minDHSize` is a number and `checkServerIdentity` is a function; fix call sites that pass the wrong type |
| Native addon ABI bump | `NODE_MODULE_VERSION` raised to 120 | Checklist item 8 | Rebuild/reinstall native addons after switching Node versions |
| Build toolchain minimums raised | Dropped Visual Studio 2019 support (Windows builds); bumped minimum supported macOS/Xcode versions; ICU minimum bumped to 73 | Only relevant if the project builds Node itself or compiles native addons from source on these platforms | Update build images/toolchains accordingly if native compilation happens in CI |

### Node 22.0.0 (2024-04-24)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| `import assert` removed | `import x from './f.json' assert { type: 'json' }` no longer parses; must use `with` | Checklist item 10 | Replace `assert` with `with` in every JSON (and other typed) import assertion |
| `crypto.createCipher()`/`createDecipher()` removed (EOL) | Fully removed, not just deprecated (was DEP0106) | Checklist item 11 | Rewrite using `crypto.createCipheriv()`/`createDecipheriv()` with an explicitly derived key (e.g. `crypto.scryptSync(password, randomSalt, keyLen)`) and a randomly generated IV (`crypto.randomBytes(ivLen)`); persist the salt and IV alongside the ciphertext — decryption needs both. This is **not** backward-compatible with data encrypted via the old API. |
| Default stream `highWaterMark` raised | 16 KiB → 64 KiB for both Readable/Writable/Duplex streams | Checklist item 15 | Set `highWaterMark` explicitly wherever the app is memory-sensitive; adjust tests asserting the old default |
| `WebSocket` global enabled by default | Previously behind `--experimental-websocket`; now a global, no flag needed | `grep -rn "experimental-websocket\|global.WebSocket\|new WebSocket(" --include="*.js" .` | If the project polyfilled/imported a userland WebSocket client under the same global name, check for collisions; otherwise no action needed, this is additive |
| `util.is*` family + `util._extend`/`util.log` runtime-deprecated | `util.isArray`, `isBoolean`, `isBuffer`, `isDate`, `isError`, `isFunction`, `isNull`, `isNullOrUndefined`, `isNumber`, `isObject`, `isPrimitive`, `isRegExp`, `isString`, `isSymbol`, `isUndefined`, `util._extend`, `util.log` all now warn on every call | Checklist item 12 | Replace each with its modern equivalent (native `Array.isArray`, `typeof` checks, `Buffer.isBuffer`, `Object.assign`, a real logger, etc.) |
| `crypto.Hash`/`crypto.Hmac` constructors runtime-deprecated | `new crypto.Hash(...)` / `new crypto.Hmac(...)` now warn | Checklist item 13 | Use `crypto.createHash(...)` / `crypto.createHmac(...)` factory functions instead |
| `fs.Stats` constructor runtime-deprecated | Constructing `fs.Stats` directly now warns | `grep -rn "new fs.Stats(\|new Stats(" --include="*.js" .` | Obtain `Stats` objects only from `fs.stat()`/`fs.lstat()`/etc., never construct manually |
| `fs.Stats` date fields made lazy | Date-typed properties (`.atime`, `.mtime`, etc.) are now computed lazily via getters rather than eagerly at stat time | Only matters if code does `Object.keys(stats)`, `JSON.stringify(stats)`, or spreads a `Stats` object expecting all date fields as own enumerable properties upfront | If such introspection exists, verify it still captures the expected fields — access the property directly (`stats.mtime`) rather than relying on enumeration |
| `console.assert()` argument handling changed | Non-string first arguments are now treated as a separate format argument rather than concatenated oddly | `grep -rn "console.assert(" --include="*.js" .` | Review any `console.assert()` calls with non-string first arguments and confirm output still reads as intended |
| `--trace-atomics-wait` runtime-deprecated | Flag now warns if used | `grep -rn "trace-atomics-wait" -r . --include="*.json" --include="*.sh" --include="*.yml"` | Remove the flag from any launch scripts/configs |
| Native addon ABI bump | `NODE_MODULE_VERSION` raised to 127 (cumulative with 21's bump to 120) | Checklist item 8 | Rebuild/reinstall native addons |
| V8 updated to 12.4 | New JS engine features (`Array.fromAsync`, `Set` methods, iterator helpers, global `Iterator`) | Checklist item 16 for the `Iterator` global collision case specifically | Usually additive; only actionable if a naming collision is found |
| `process`: exit ordering changed | Node now waits for the `'exit'` event before printing certain results/output in some internal paths | Only relevant if code has tight assumptions about output ordering relative to `process.on('exit', ...)` handlers | Re-run tests that assert on stdout ordering around process exit; adjust if a race surfaces |

### Node 22.10.0 (2024-10-16) — additive, not required

New `"module-sync"` package.json `exports` condition for dual CJS/ESM package authors (checklist
item 17). Not a breaking change for consumers; a follow-up recommendation for the target project
only if it *publishes* a dual-format package.

### Node 22.12.0 (2024-12-03)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| `require(esm)` enabled by default | Previously behind `--experimental-require-module`; now Node 22.x can `require()` a synchronous ES module without the flag, no longer throwing `ERR_REQUIRE_ESM` in that case | `grep -rn "experimental-require-module" -r . --include="*.json" --include="*.sh" --include="*.yml"` | Remove the now-unnecessary flag from launch scripts. Note the new failure mode: `require()`-ing an ES module that (or whose dependencies) use top-level `await` now throws `ERR_REQUIRE_ASYNC_MODULE` instead of `ERR_REQUIRE_ESM` — if the project relies on catching `ERR_REQUIRE_ESM` specifically, update that error-handling code. |

### Node 22.17.0 (2025-06-24)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| `node:http` classes without `new` discouraged | Constructing `IncomingMessage`/`ServerResponse` etc. without `new` now warns (not yet an error) | `grep -rn "IncomingMessage(\|ServerResponse(" --include="*.js" . \| grep -v "new "` | Add `new` to any such construction found without it |
| `child_process` `shell: ""` deprecated | Previously had undefined behavior; now warns | `grep -rn "shell:\s*['\"]['\"]" --include="*.js" .` | Change to `shell: true` or an explicit shell path |
| HTTP/2 priority signaling deprecated | `stream.priority()` and related API now warn on use | Checklist item 20 | Stop using priority hints; see the Node 22.23.0 entry below — this is later fully removed |

### Node 22.18.0 (2025-07-31) — additive, not required

TypeScript type-stripping enabled by default (experimental): `node file.ts` now runs plain
TypeScript syntax directly without a separate transpile step, for a supported syntax subset
(no enums, namespaces, or other constructs requiring actual code transformation — see
`nodejs.org/api/typescript.html#type-stripping` for the exact subset at implementation time). Not a
breaking change; only relevant if the project wants to simplify its TypeScript dev loop. Can be
disabled with `--no-experimental-strip-types` if it interferes with an existing custom TS pipeline
(e.g. ts-node, tsx) that the project prefers to keep using unchanged.

### Node 22.20.0 (2025-09-24)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| Bundled OpenSSL updated | OpenSSL 3.0.x → 3.5.2 | Only relevant if the project depends on very specific OpenSSL version behavior (rare) or ships its own OpenSSL-linked native addon | Generally safe; re-run the full test suite's TLS/crypto-related tests after upgrading, since minor OpenSSL behavior differences (cipher suite defaults, certain error message text) are possible even across compatible major versions |

### Node 22.23.0 (2026-06-18)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| HTTP/2 priority signaling removed | The API deprecated in 22.17.0 is now fully removed (not just warning) | Checklist item 20 | If any code still calls `stream.priority()` or configures HTTP/2 priority, it must be removed before landing on Node 22.23.0 or later — this errors, it doesn't just warn |

## Implementation plan

1. **Confirm exact target patch version.** Run checklist item 21. Pin to that specific `22.x.y`
   version everywhere version strings appear (don't mix a floating `22` in one place and a pinned
   patch elsewhere) — consistency avoids "works in CI, breaks in prod" drift.
2. **Update version-pin files together, in one commit.** Based on checklist items 1–6: `.nvmrc`,
   `.node-version`, `package.json` `engines.node` and/or `volta.node`, `.tool-versions`, every CI
   workflow file, every `Dockerfile`, and any PaaS/serverless runtime declaration. Doing these
   together avoids a state where CI tests one Node version while the Dockerfile ships another.
3. **Bump `@types/node`** (checklist item 7) to `^22` if TypeScript is in use, in the same commit as
   step 2 — a mismatched types package can hide real type errors introduced by the runtime bump.
4. **Fix every removed-API usage before switching the running Node version locally**, since these
   are hard errors, not warnings, once actually running on Node 22:
   - `import assert` → `import with` (checklist item 10)
   - `crypto.createCipher`/`createDecipher` → `createCipheriv`/`createDecipheriv` (checklist item 11)
   - Any HTTP/2 priority API calls, if targeting Node ≥ 22.23.0 (checklist item 20)
5. **Switch local Node version and run the app once, expecting deprecation warnings**, not
   necessarily failures. Address each warning by category (checklist items 12–14):
   - `util.is*`/`util._extend`/`util.log` → modern equivalents
   - `crypto.Hash`/`Hmac` constructors → `createHash`/`createHmac`
   - `punycode` → userland package
6. **Rebuild native addons** (checklist item 8): delete `node_modules`, reinstall with the new Node
   version active (`rm -rf node_modules && npm ci`), so any native modules compile against/fetch
   prebuilds for the new ABI. If a dependency has no Node 22-compatible prebuild and fails to build
   from source, that's a hard blocker — check the dependency's own issue tracker for Node 22 support
   status before assuming it's fixable locally.
7. **Re-run the full test suite.** Pay particular attention to:
   - Stream-related tests that might assume the old 16 KiB `highWaterMark` (checklist item 15).
   - Any test asserting on `fs.Stats` object shape/enumeration (breaking-changes list, Node 22.0.0).
   - TLS/crypto tests, given the OpenSSL bump in 22.20.0.
8. **Review and simplify now-unnecessary flags/workarounds**: remove
   `--experimental-require-module` and `--experimental-websocket` if present anywhere in launch
   scripts (both features are now default-on).
9. **Update CI to actually run on the new pinned version** and confirm a full green run before
   merging — don't rely on local testing alone given the native-addon and OS-level differences step
   6 can surface.

## Verification steps

- `node --version` in every environment (local, CI runner, container, deployed instance) reports
  the exact pinned `22.x.y` from step 1 — no environment silently still on 20.x or a different 22.x
  patch.
- Full test suite passes with zero new failures and, ideally, zero new `DeprecationWarning` output
  in test logs (each one addressed per the implementation plan, not suppressed).
- `npm ls` (or the project's package manager equivalent) shows no native-addon install errors.
- Manually exercise any code path that touches: JSON imports with import attributes, crypto
  cipher/decipher usage, and any HTTP/2 code if present — these are the hard-failure-risk areas.
- If the project has a staging environment, deploy there first and monitor logs for unexpected
  `DeprecationWarning`/`ExperimentalWarning` output before promoting to production.

## Rollback / risk notes

- Keep the Node 20.x version pin files (or their values, in the commit history) easily revertible —
  a straight revert of the version-pin commit (step 2) plus reinstalling `node_modules` under Node
  20 is the fastest rollback path if a production issue surfaces post-upgrade.
- The riskiest, hardest-to-reverse step is the `crypto.createCipher`/`createDecipher` rewrite
  (checklist item 11): data encrypted under the new `createCipheriv` scheme with a freshly generated
  salt/IV is not decryptable by the old code path, and vice versa. If the project encrypts
  long-lived data (not just in-transit/ephemeral use), coordinate a data-migration plan separately
  from the Node version bump — don't ship both changes in the same deploy without a way to decrypt
  data written by either scheme during the transition.
- Native addon rebuild failures (step 6) are the most likely source of a blocked upgrade — identify
  and resolve these in a branch before touching CI/production version pins, so the rollback surface
  stays limited to version-pin files if everything else is already proven working.

## References

- [Node.js v20 to v22 migration guide](https://nodejs.org/en/blog/migrations/v20-to-v22) — official codemods for `import assert`→`with` and `createCipher`/`createDecipher`.
- [Node.js 22.0.0 release announcement](https://nodejs.org/en/blog/announcements/v22-release-announce)
- `nodejs/node` repo, `doc/changelogs/CHANGELOG_V21.md` (all 13 releases, 21.0.0–21.7.3) — fetched in full.
- `nodejs/node` repo, `doc/changelogs/CHANGELOG_V22.md` (all releases 22.0.0–22.23.2, the latest found at research time) — fetched in full; Notable Changes and Semver-Major-tagged commits reviewed for every release in that range.
- `nodejs/node` repo, `doc/changelogs/CHANGELOG_V20.md` — consulted to confirm the Node 20 line's final release (20.20.2, 2026-03-24) and bundled npm version at 20.0.0.
- [Node.js release schedule / previous releases](https://nodejs.org/en/about/previous-releases) — LTS/EOL status and dates for Node 20, 21, and 22.
- [Node.js TypeScript type-stripping docs](https://nodejs.org/api/typescript.html#type-stripping) — referenced for the 22.18.0 entry; consult at implementation time for the current supported syntax subset, since this feature is explicitly experimental and subject to change.

**Assumption flagged for user review:** the "latest Node 22.x patch" and "current LTS/EOL status"
facts above reflect this guide's research date (~September 2026, per the fetched changelog's most
recent entry, 22.23.2 dated 2026-07-29). Re-verify the current latest 22.x patch at implementation
time (checklist item 21) rather than trusting this snapshot indefinitely.
