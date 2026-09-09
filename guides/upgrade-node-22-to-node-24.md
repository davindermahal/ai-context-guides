# Upgrade Node.js 22.x to Node.js 24.x (LTS)

Produces: a Node.js project (any framework/runtime target — server, CLI, build tooling) migrated
from the Node 22.x LTS line to the Node 24.x LTS line, with all breaking changes between the two
lines addressed, native dependencies rebuilt for the new ABI and toolchain minimums, and CI/deploy
configs updated to match.

## When to use this guide

- The target project currently runs on any Node 22.x version and needs to move to Node 24.x (LTS
  codename "Krypton").
- Covers the full jump, including everything that changed in the intermediate Node 23.x line
  (23.0.0–23.11.1, Current-only, never LTS, EOL ~2025-06) — Node 24 ships with all of Node 23's
  changes plus its own, even though a project moving straight from 22→24 never runs 23 directly.
- **Not** the right guide for a from-scratch Node.js project, nor for upgrading from Node 20 to
  Node 22 (see `upgrade-node-20-to-node-22.md` in this repo's `guides/` folder for that jump
  instead) or from Node 24 to a later line.
- Node 22 entered Maintenance LTS around October 2026 and reaches End-of-Life 2027-04-30 — if the
  target project is still on 22.x well past that Maintenance start, this upgrade stops being
  optional from a security-support standpoint.

## Prerequisites / assumptions

- The target project has a `package.json` (Node.js project of some kind — server, CLI, library,
  build tooling). Verify this before proceeding; if there's no `package.json`, this guide doesn't
  apply.
- The future agent must verify every fact below against the *actual* target project — nothing here
  should be assumed true just because it's common:
  - Current exact Node version(s) used in development, CI, and production may differ from each
    other — check all three.
  - The project may or may not use TypeScript, ESM (`"type": "module"`), native addons, HTTP/2,
    the legacy `url.parse()` API, or a process manager/container runtime — all of which change
    which steps below apply.
  - Whether the project builds native addons from source (vs. only consuming prebuilt binaries)
    determines whether the toolchain-minimum changes below are relevant at all.

## Investigation checklist

Run every command below against the target project and record the result before planning changes.

| # | What to check | Command | What the result means |
|---|---|---|---|
| 1 | Current Node version pinned for local dev | `cat .nvmrc 2>/dev/null; cat .node-version 2>/dev/null; grep -A2 '"volta"' package.json 2>/dev/null` | Whichever file(s) exist define the dev-time pinned version — update all that are found, don't assume only one mechanism is in use. |
| 2 | Current Node version constraint in `package.json` | `grep -A3 '"engines"' package.json` | If `engines.node` exists (e.g. `"22.x"`, `">=20"`), update it to require `24.x` (or a range including 24, if the user confirms broader compatibility is intended — don't assume). |
| 3 | Node version used in CI | `grep -rn "node-version\|NODE_VERSION\|node:22\|node:2[013]" .github/workflows/*.yml .gitlab-ci.yml azure-pipelines.yml Jenkinsfile 2>/dev/null` | Every match naming `22` (or `23`) must be bumped to `24`. A CI matrix intentionally testing multiple Node versions may keep 22 as a compatibility target — confirm with the user before removing it. |
| 4 | Docker base image | `grep -rn "^FROM node" Dockerfile* 2>/dev/null` | A tag like `node:22`, `node:22-alpine`, `node:22-slim`, `node:22-bookworm` must become the equivalent `node:24...` tag. Prefer pinning to a specific `24.x.y` tag/digest if the project already pins precise versions elsewhere. |
| 5 | Serverless/PaaS runtime declarations | `grep -rn "nodejs22\|nodejs20\|runtime:" serverless.yml template.yaml 2>/dev/null; cat vercel.json netlify.toml 2>/dev/null` | Update any `nodejsXX.x`-style runtime string or Node version setting to its `24` equivalent. Verify the target platform actually offers a Node 24 managed runtime before committing — managed-runtime support can lag Node's own release by weeks to months. |
| 6 | asdf / mise version managers | `cat .tool-versions 2>/dev/null` | A `nodejs 22.x.y` line must be bumped to a `24.x.y` line. |
| 7 | TypeScript type definitions | `grep '"@types/node"' package.json` | If present, bump to `^24` to match the new runtime's types. |
| 8 | Native addons requiring a rebuild | `grep -l "node-gyp\|prebuild" package-lock.json 2>/dev/null; find . -maxdepth 3 -name "*.node" 2>/dev/null; grep -E '"(bcrypt|sharp|sqlite3|better-sqlite3|canvas|node-sass|grpc|@grpc/grpc-js|puppeteer-core|serialport)"' package.json` | Any match is a native addon. Node's ABI version (`NODE_MODULE_VERSION`) changed again between 22 and 24 — prebuilt binaries built for Node 22's ABI will not load on Node 24 without either a matching Node-24 prebuild or a local rebuild. Run `node -p process.versions.modules` on both the old and new Node install to get the exact ABI numbers for this specific project's environment rather than trusting a hardcoded number — Node bumps this value across minor releases within a major line too, not just at `.0.0`. |
| 9 | Native addon C++ standard / compiler minimums | `node-gyp --version 2>/dev/null; gcc --version 2>/dev/null; g++ --version 2>/dev/null; xcodebuild -version 2>/dev/null` | If the project (or a dependency) compiles native code from source: Linux/AIX now requires **gcc ≥ 12.2**; macOS requires **Xcode ≥ 16.1**; addons must compile with **C++20** (previously C++17 was often sufficient). Upgrade the build image/toolchain if any check falls short — a too-old toolchain fails the native build, it does not silently degrade. |
| 10 | CPU architecture of build/deploy targets | `uname -m` on every build and deploy target (CI runner, container base image, production host) | Node no longer ships prebuilt binaries for **32-bit Windows (x86)** (since 23.0.0) or **32-bit Linux on armv7** (since 24.0.0, downgraded to experimental-only); 32-bit `ppc` and 32-bit `s390` support was also removed at 24.0.0. If any target is one of these, it must move to a 64-bit target — there is no Node 24 prebuilt binary to fall back to. |
| 11 | macOS build/deploy target OS version | `sw_vers -productVersion` on any macOS build/deploy machine | Node 24 prebuilt binaries require **macOS ≥ 13.5**. An older macOS version cannot run the official Node 24 binary. |
| 12 | Package manager lockfile version + bundled npm major | `node -pe "require('./package-lock.json').lockfileVersion" 2>/dev/null; npm --version` | Node 22 bundled npm ~10.5.1+; Node 24 bundles **npm 11.x**. A bundled npm major bump can itself change `npm install`/`npm ci` behavior — consult npm 11's own release notes if `npm ci` behaves differently after the Node upgrade, don't assume it's a Node-side regression. |
| 13 | Use of legacy `url.parse()` | `grep -rn "url\.parse(\|require(['\"]url['\"])\.parse\|from ['\"]url['\"];.*parse" --include="*.js" --include="*.ts" .` | `url.parse()` is runtime-deprecated as of Node 24.0.0 (prints `DeprecationWarning` on every call, not yet removed). Replace with the WHATWG `new URL(input, base)` API. |
| 14 | Use of `dirent.path` | `grep -rn "\.path\b" --include="*.js" --include="*.ts" . \| grep -i dirent` | `dirent.path` was runtime-deprecated in Node 23.0.0 and **fully removed** in Node 24.0.0 — this is a hard error on Node 24, not a warning. Replace with `dirent.parentPath`. An automated codemod exists: `npx codemod run @nodejs/dirent-path-to-parent-path`. |
| 15 | Use of `fs.truncate(fd, ...)` (file descriptor form) | `grep -rn "fs\.truncate(\|truncate(fd" --include="*.js" --include="*.ts" .` | Calling `truncate()` with a file descriptor (rather than a path) is removed at Node 24.0.0. Replace with `fs.ftruncate(fd, ...)`. Codemod: `npx codemod run @nodejs/fs-truncate-fd-deprecation`. |
| 16 | Use of `fs.F_OK`/`fs.R_OK`/`fs.W_OK`/`fs.X_OK` (top-level, not via `fs.constants`) | `grep -rnE "fs\.(F_OK|R_OK|W_OK|X_OK)\b" --include="*.js" --include="*.ts" .` | Runtime-deprecated at Node 24.0.0 (warns, not yet removed). Replace with `fs.constants.F_OK` etc. (or `fs.promises.constants`). Codemod: `npx codemod run @nodejs/fs-access-mode-constants`. |
| 17 | Use of `crypto.createSecurePair()` / `tls.createSecurePair()` | `grep -rn "createSecurePair(" --include="*.js" --include="*.ts" .` | Fully removed at Node 24.0.0. Replace with `new tls.TLSSocket(underlyingSocket, { secureContext, isServer, requestCert, rejectUnauthorized })`. Codemod: `npx codemod run @nodejs/tls-create-secure-pair-to-tls-socket`. |
| 18 | Use of `process.assert()` | `grep -rn "process\.assert(" --include="*.js" --include="*.ts" .` | Fully removed at Node 23.0.0 (DEP0100 reached End-of-Life). Replace with `assert()` from `node:assert`. Codemod: `npx codemod run @nodejs/process-assert-to-node-assert`. |
| 19 | `crypto.generateKeyPair`/`generateKeyPairSync` with `'rsa-pss'` using `hash`/`mgf1Hash` options | `grep -rn "generateKeyPair.*rsa-pss\|'rsa-pss'" --include="*.js" --include="*.ts" -A5 .` and inspect for `hash:`/`mgf1Hash:` keys | These option names are deprecated (DEP0154) in favor of `hashAlgorithm`/`mgf1HashAlgorithm`. Codemod: `npx codemod run @nodejs/crypto-rsa-pss-update`. |
| 20 | HTTP/2 priority signaling usage | `grep -rn "\.priority(\|priority:\s*{.*weight" --include="*.js" --include="*.ts" .` (check matches are actually HTTP/2 `session.request()`/`http2.connect()` options or `stream.priority()`, not an unrelated `priority` key) | Fully removed at Node 24.2.0 (and separately at Node 22.23.0 on the 22 line) — this errors, not warns, once the installed Node 24 patch is ≥ 24.2.0 (effectively all supported 24.x patches, since 24.0.0/24.0.1/24.0.2/24.1.0 predate LTS and shouldn't be targeted anyway per checklist item 21). Codemod: `npx codemod run @nodejs/http2-priority-signaling`. |
| 21 | Exact Node 24.x patch version to install | `npm view node-lts version 2>/dev/null || echo "check https://nodejs.org/en/download for current 24.x LTS patch"` | Pin to the current latest 24.x LTS patch at implementation time — this guide's research (mid-2026) found 24.20.0 as latest, but Node ships patches frequently; always re-verify. Only install a version ≥ 24.11.0, the first release where the 24.x line actually entered LTS — earlier 24.0.x–24.10.x releases were Current-only and should not be targeted for a production upgrade. |
| 22 | Reliance on `assert.deepStrictEqual`/`assert.partialDeepStrictEqual` comparing `WeakMap`/`WeakSet` instances | `grep -rn "deepStrictEqual.*WeakMap\|deepStrictEqual.*WeakSet\|new WeakMap\|new WeakSet" --include="*.test.js" --include="*.spec.js" --include="*.test.ts" --include="*.spec.ts" .` | As of Node 23.0.0, two distinct `WeakMap`/`WeakSet` instances are now always considered unequal by `assert.deepStrictEqual`, even with identical contents — only reference-identical instances are equal (previously, in some versions, content was compared where feasible). Any test relying on content-based equality for these types will now fail and must be rewritten to compare extracted entries instead, or to assert on the same instance. |
| 23 | Direct reads of `outgoingMessage._headers` / `._headersList` (Node internals, not the public API) | `grep -rn "_headers\b\|_headersList\b" --include="*.js" --include="*.ts" .` | These private `http.OutgoingMessage` properties were removed at Node 24.0.0. Only affects code (often older middleware or logging libraries) that reaches into HTTP internals directly — use `response.getHeaders()` / `response.getHeaderNames()` instead. |
| 24 | `child_process.spawn()`/`execFile()` called with both `shell: true` and an `args` array | `grep -rn "spawn(\|execFile(" -A5 --include="*.js" --include="*.ts" . \| grep -B3 "shell:\s*true"` | Passing a separate `args` array together with `shell: true` is deprecated at Node 24.0.0 — when using a shell, arguments must be part of the command string instead. Fold the arguments into the command string (with correct shell-quoting) or drop `shell: true` if it isn't actually needed. |
| 25 | GCM cipher usage without explicit `authTagLength` | `grep -rn "createDecipheriv(.*gcm\|createCipheriv(.*gcm" -i --include="*.js" --include="*.ts" .` and check whether `authTagLength` is passed | Runtime-deprecated (DEP0182) as of Node 23.0.0 — using a short/implicit auth tag length now warns. Pass `authTagLength` explicitly (16 is the standard/recommended value for GCM) at both encrypt and decrypt call sites. |
| 26 | Use of `--experimental-permission` CLI flag | `grep -rn "experimental-permission" -r . --include="*.json" --include="*.sh" --include="*.yml"` | Renamed to `--permission` at Node 24.0.0 — the old flag name no longer works at all (hard error, not a warning). Update any launch script/config using the old name. |
| 27 | Use of `zlib` classes (`Gzip`, `Deflate`, `Inflate`, etc.) without `new` | `grep -rnE "= (Gzip|Deflate|Inflate|Gunzip|BrotliCompress|BrotliDecompress)\(" --include="*.js" --include="*.ts" .` | Calling these as plain functions (no `new`) is deprecated at Node 24.0.0. Add `new`. |
| 28 | Use of `zlib.bytesRead` | `grep -rn "\.bytesRead\b" --include="*.js" --include="*.ts" . \| grep -i zlib` | Removed at Node 23.0.0. Use `.bytesWritten` instead (the property `bytesRead` was a confusingly-named alias for the same underlying counter). |
| 29 | Use of `crypto.fips` | `grep -rn "crypto\.fips\b" --include="*.js" --include="*.ts" .` | Runtime-deprecated at Node 23.0.0 (warns, not removed). If the project needs FIPS mode, use the `--enable-fips`/`--force-fips` CLI flags or the `node:crypto` FIPS-related config documented at implementation time rather than the `crypto.fips` property. |
| 30 | Reliance on `SlowBuffer` (`require('buffer').SlowBuffer` / `new SlowBuffer(...)`) | `grep -rn "SlowBuffer" --include="*.js" --include="*.ts" .` | Runtime-deprecated (and, per one code path, moved further toward end-of-life) at Node 24.0.0. Replace with `Buffer.allocUnsafeSlow(size)`. |
| 31 | Bundled Corepack reliance (`corepack enable`, Yarn/pnpm invoked via Corepack shims) | `grep -rn "corepack" package.json .github/workflows/*.yml Dockerfile* 2>/dev/null` | Not a Node 22→24 breaking change, but Node's own docs now state Corepack will be removed from the default Node distribution starting with Node 25 — a target project on 24.x still gets it bundled, but should plan to install Corepack as a separate dependency (`npm install -g corepack`) before that removal lands, rather than assuming it stays bundled indefinitely. Flag this as a follow-up note, not a blocking step for this 22→24 upgrade. |

## Breaking changes, version by version

Node's release line runs 22 → 23 (Current-only, never LTS) → 24 (LTS). A project jumping straight
from 22.x to 24.x is affected by everything introduced in *both* 23.x and 24.x, cumulatively — Node
24 contains all of 23's changes.

### Node 23.0.0 (2024-10-16)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| 32-bit Windows binaries discontinued | No more official prebuilt `x86` Windows binaries | Checklist item 10 | Move build/deploy to 64-bit Windows |
| `assert`/`util`: `WeakMap`/`WeakSet` comparison by identity only | Distinct instances with identical entries are now unequal under `assert.deepStrictEqual` | Checklist item 22 | Rewrite affected assertions to compare extracted entries or the same instance |
| `buffer`: throws when writing beyond buffer bounds | Previously silent truncation/undefined behavior in some write paths; now throws | `grep -rn "\.write(" --include="*.js" .` and review any writes with attacker- or user-controlled offsets/lengths | Add bounds checking before the write, or catch and handle the new thrown error |
| `--trace-atomics-wait` flag removed (EOL) | Was runtime-deprecated in Node 22.0.0; now using the flag errors | `grep -rn "trace-atomics-wait" -r . --include="*.json" --include="*.sh" --include="*.yml"` | Remove the flag from launch scripts |
| `--no-experimental-global-customevent`, `--no-experimental-fetch`, `--no-experimental-global-webcrypto` flags removed | These features (CustomEvent, fetch, WebCrypto globals) are now permanently stable — no way to opt out | Same search pattern as above, for each flag name | Remove the flags; the features can no longer be disabled |
| `crypto.fips` runtime-deprecated | Checklist item 29 | Checklist item 29 | Use `--enable-fips`/`--force-fips` flags instead |
| `ERR_CRYPTO_SCRYPT_INVALID_PARAMETER` removed | This specific error code/class no longer exists | `grep -rn "ERR_CRYPTO_SCRYPT_INVALID_PARAMETER" --include="*.js" .` | If code catches this specific error code, update the catch logic to the current scrypt error handling documented at implementation time |
| GCM short auth tag runtime-deprecated (DEP0182) | Checklist item 25 | Checklist item 25 | Pass `authTagLength` explicitly |
| `fs`: `dirent.path` runtime-deprecated | Warns now, removed in 24.0.0 (see below) | Checklist item 14 | Migrate to `dirent.parentPath` now, before the removal lands in 24.0.0 |
| `fs`,`win`: trailing-slash path bugs fixed | Windows-specific path handling corrected | Only relevant on Windows targets with paths ending in slashes | Re-run filesystem-path tests on Windows after upgrading |
| `net`: server hostname validated on `listen()` | Previously-tolerated invalid hostnames now throw | `grep -rn "\.listen(" --include="*.js" .` and check hostname arguments, especially any built from dynamic/config-driven strings | Ensure hostnames passed to `.listen()` are valid; validate/sanitize config-driven values |
| `path`: assorted bug/inconsistency fixes | Several edge-case path-resolution behaviors corrected, primarily on Windows | Re-run path-manipulation tests after upgrading, especially any relying on a specific (buggy) prior behavior | Adjust any test/code that depended on the old buggy behavior |
| `process.assert` removed (DEP0100 EOL) | Checklist item 18 | Checklist item 18 | Use `assert()` from `node:assert` |
| `stream`: piping to a closed/destroyed stream now disallowed in `pipeline()` | Previously silent/inconsistent; now throws | `grep -rn "pipeline(" --include="*.js" .` and review error handling around each call | Add/verify error handling on `pipeline()` calls |
| `string_decoder`: stricter encoding validation | Invalid encodings now rejected more consistently | `grep -rn "StringDecoder(" --include="*.js" .` and check the encoding argument | Ensure a valid encoding string is always passed |
| `test_runner`: `--test` no longer required to detect "only" tests; `spec` reporter always default; `lcov` reporter now a newable class | Changes test-runner CLI defaults and the `lcov` reporter's API shape | Only relevant if the project's own test suite uses `node --test` and customizes reporters | Re-run the test suite; update any custom reporter code using the old `lcov` reporter shape |
| `timers`: warns on negative/`NaN` delay | `setTimeout`/`setInterval` with an invalid delay now emits a process warning | `grep -rn "setTimeout(\|setInterval(" --include="*.js" .` and check for computed delay values that could be negative/NaN | Validate delay values before passing them |
| `tls`: `ERR_TLS_PSK_SET_IDENTIY_HINT_FAILED` error code corrected | The misspelled error code string was fixed (still misspelled in a similar but different way is possible — check the exact string at implementation time) | `grep -rn "ERR_TLS_PSK_SET_IDENTIY_HINT_FAILED\|ERR_TLS_PSK_SET_IDENTITY_HINT_FAILED" --include="*.js" .` | If code matches on the exact old string, verify against the current exact error code name in the installed Node 24's docs and update the match |
| `util.is*` family, `util._extend`, `util.log` fully removed (EOL) | These were runtime-deprecated (warning) since Node 22.0.0; now calling them throws/is undefined | Same checklist as the Node 20→22 guide's item 12 — re-run: `grep -rnE "util\.(isArray|isBoolean|isBuffer|isDate|isError|isFunction|isNull|isNullOrUndefined|isNumber|isObject|isPrimitive|isRegExp|isString|isSymbol|isUndefined|_extend|log)\(" --include="*.js" --include="*.ts" .` | Any match still present must be fixed now — this is a hard failure on Node 23+, not a warning. Replace with modern equivalents (`Array.isArray`, `typeof` checks, `Object.assign`, a real logger, etc.) |
| `zlib.bytesRead` removed | Checklist item 28 | Checklist item 28 | Use `.bytesWritten` |
| Native addon ABI bump | `NODE_MODULE_VERSION` raised (to 129, then further to 131 within the 23.x line) | Checklist item 8 | Rebuild/reinstall native addons; verify exact ABI via `node -p process.versions.modules` |
| Build toolchain minimums raised | gcc ≥ 12.2 on Linux/AIX; C++20 compilation required; Windows <10 experimental support dropped | Checklist item 9 | Update build images/toolchains if native compilation happens in CI |

### Node 24.0.0 (2025-05-06)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| `fs`: `dirent.path` fully removed | Was a warning since 23.0.0; now accessing it throws/is `undefined` | Checklist item 14 | Migrate to `dirent.parentPath` (codemod available) |
| `fs`: `truncate(fd, ...)` form removed | Checklist item 15 | Checklist item 15 | Use `ftruncate(fd, ...)` (codemod available) |
| `fs`: `F_OK`/`R_OK`/`W_OK`/`X_OK` on `fs` directly runtime-deprecated | Checklist item 16 | Checklist item 16 | Use `fs.constants.*` (codemod available) |
| `fs.existsSync`: invalid argument types deprecated | Passing a non-path-like value now warns | `grep -rn "existsSync(" --include="*.js" .` and check argument types | Ensure a string/URL/Buffer path is always passed |
| `tls.createSecurePair` removed | Checklist item 17 | Checklist item 17 | Use `new tls.TLSSocket(...)` (codemod available) |
| `tls`: `Server.prototype.setOptions` removed (EOL) | Fully removed, was previously deprecated | `grep -rn "setOptions(" --include="*.js" . ` (verify matches are actually `tls.Server` instances, not an unrelated `setOptions`) | Reconstruct the server with the desired options instead of mutating after creation |
| `net`: `_setSimultaneousAccepts()` made EOL | Internal Windows-only legacy tuning API removed | `grep -rn "_setSimultaneousAccepts" --include="*.js" .` | Extremely unlikely to be in app code; remove if found, it has had no effect for years |
| `lib`: obsolete `Cipher` export removed from `node:crypto` | `require('crypto').Cipher` (the class itself, not `createCipher`) no longer exported | `grep -rn "crypto\.Cipher\b" --include="*.js" .` | Use `crypto.createCipheriv()` as the entry point instead of referencing the class directly |
| `http`: `OutgoingMessage._headers`/`._headersList` removed | Checklist item 23 | Checklist item 23 | Use `response.getHeaders()`/`getHeaderNames()` |
| `http2`: session tracking / graceful server close behavior changed | Server-side HTTP/2 session lifecycle handling reworked | Only relevant if the project runs an HTTP/2 server with custom session-close logic | Re-test HTTP/2 server shutdown paths (e.g. `server.close()` during active sessions) after upgrading |
| `http2`: priority signaling fully removed | Checklist item 20 | Checklist item 20 | Remove all priority-related options/calls (codemod available); also see the Node 24.2.0 entry below — it was removed slightly later depending on exact 24.x starting patch, but must be gone by any 24.x LTS patch |
| `readline`: stricter validation for functions called after `close()` | Calling readline interface methods after `close()` now throws instead of silently no-op-ing | `grep -rn "readline\." --include="*.js" .` and review any use of the interface after `.close()` is called | Guard calls with a check for whether the interface is still open, or restructure to avoid post-close calls |
| `readline`: Unicode line-separator handling fixed | Certain Unicode line-separator characters that were previously ignored are now honored when splitting lines | Only relevant if input text contains Unicode line separators (`U+2028`/`U+2029`) and the project relies on the old (buggy) splitting behavior | Re-test any line-splitting logic against such input |
| `child_process`: `args` + `shell: true` combination deprecated | Checklist item 24 | Checklist item 24 | Fold arguments into the command string when using `shell: true` |
| `timers`: `clearImmediate()` validates its argument | Passing a non-`Immediate` value now behaves differently (validated) rather than silently accepted | `grep -rn "clearImmediate(" --include="*.js" .` | Ensure only values returned by `setImmediate()` are passed to `clearImmediate()` |
| `url.parse()` runtime-deprecated | Checklist item 13 | Checklist item 13 | Use `new URL(input, base)` |
| `zlib`: classes require `new` | Checklist item 27 | Checklist item 27 | Add `new` to `Gzip`/`Deflate`/etc. construction |
| `buffer.SlowBuffer` runtime-deprecated | Checklist item 30 | Checklist item 30 | Use `Buffer.allocUnsafeSlow(size)` |
| `repl`: instantiating without `new` runtime-deprecated | `repl.REPLServer(...)` without `new` now warns | `grep -rn "REPLServer(" --include="*.js" . \| grep -v "new "` | Add `new` |
| `--experimental-permission` flag renamed | Checklist item 26 | Checklist item 26 | Rename to `--permission` in launch configs |
| OpenSSL bumped to 3.5, default security level raised to 2 | RSA/DSA/DH keys < 2048 bits, ECC keys < 224 bits, and RC4-based cipher suites are now **prohibited by default** | Only relevant if the project (or a service it talks to via TLS) uses older/weaker keys or ciphers — test actual TLS handshakes against a Node 24 build before assuming compatibility | If weak keys/ciphers are required for a legacy integration, they must be regenerated/upgraded — lowering the OpenSSL security level to work around this is a security regression and should only be a deliberate, documented, temporary measure agreed with the user, never a default fix |
| `deps`: bundled npm upgraded to 11.0.0 | Checklist item 12 | Checklist item 12 | Review npm 11's own breaking-changes notes if `npm ci`/`npm install` behavior changes post-upgrade |
| `deps`: undici (bundled `fetch` implementation) updated to 7.0.0 | Stricter WHATWG `fetch()` spec compliance | `grep -rn "fetch(" --include="*.js" .` and re-run any code exercising `fetch()`, especially around request/response header handling and redirects | Re-run integration tests touching `fetch()`; adjust for any spec-compliance differences surfaced |
| `stream`: `dest.write()` errors now caught and forwarded | A destination stream's write errors, previously possibly swallowed in some paths, now propagate | Re-run stream-pipeline error-handling tests | Ensure error listeners are attached where previously they may not have needed to be |
| C++20 compilation, Xcode ≥ 16.1, macOS prebuilt binaries require ≥ 13.5 | Checklist items 9, 11 | Checklist items 9, 11 | Update build images/toolchains and deploy target OS versions |
| Native addon ABI bump | `NODE_MODULE_VERSION` raised again within 24.0.0's development (values as high as 134 then 137 appeared in different pre-release commits) | Checklist item 8 | Rebuild/reinstall native addons; verify exact ABI via `node -p process.versions.modules` rather than trusting a specific number from this table |
| 32-bit Linux armv7, `ppc` 32-bit, `s390` 32-bit dropped/downgraded | Checklist item 10 | Checklist item 10 | Move to a supported 64-bit architecture |

### Node 24.2.0 (2025-06-09)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| HTTP/2 priority signaling removed at the protocol/dependency level (nghttp2) | Confirms/completes the removal noted at 24.0.0's checklist item 20, at the underlying HTTP/2 library level | Checklist item 20 | Same as above — remove all priority-related code (codemod available) |

### Node 24.3.0 (2025-06-24) — additive, not required

TypeScript type-stripping (introduced experimentally around the Node 22 line) graduated to stable —
the experimental-feature warning printed on `node file.ts` is removed. Not a breaking change; only
relevant if the project wants to simplify its TypeScript dev loop and was previously suppressing or
tolerating that warning.

### Node 24.6.0 (2025-08-14) — documentation-only, not required

The legacy `_http_*` internal modules (`_http_client`, `_http_server`, etc.) gained a documentation
deprecation notice. Not yet a runtime warning or removal; flag as a future-facing note only if the
target project imports these internal modules directly (rare — `grep -rn "_http_client\|_http_server\|_http_common\|_http_outgoing\|_http_incoming" --include="*.js" --include="*.ts" .`).

### Node 24.17.0 (2026-06-18)

| Change | Old vs. new | How to check if affected | What to do |
|---|---|---|---|
| Bundled `nghttp2` updated to 1.69.0 | Dependency version bump for the HTTP/2 implementation | Only relevant if the project has HTTP/2-specific integration tests | Re-run HTTP/2 tests; no code change expected for typical usage |

## Implementation plan

1. **Confirm exact target patch version.** Run checklist item 21. Pin to that specific `24.x.y`
   version everywhere version strings appear.
2. **Update version-pin files together, in one commit.** Based on checklist items 1–6: `.nvmrc`,
   `.node-version`, `package.json` `engines.node`/`volta.node`, `.tool-versions`, every CI workflow
   file, every `Dockerfile`, and any PaaS/serverless runtime declaration.
3. **Bump `@types/node`** (checklist item 7) to `^24` if TypeScript is in use, in the same commit as
   step 2.
4. **Audit build/deploy infrastructure against the raised platform minimums before touching code**:
   architecture (checklist item 10), macOS version (checklist item 11), and compiler toolchain
   (checklist item 9) if native addons are compiled from source anywhere in the pipeline. These are
   infrastructure changes, not code changes, and can be planned/scheduled independently of the
   application-code fixes below.
5. **Fix every fully-removed-API usage before switching the running Node version locally**, since
   these are hard errors on Node 24, not warnings:
   - `dirent.path` → `dirent.parentPath` (checklist item 14)
   - `fs.truncate(fd, ...)` → `fs.ftruncate(fd, ...)` (checklist item 15)
   - `tls.createSecurePair`/`crypto.createSecurePair` → `new tls.TLSSocket(...)` (checklist item 17)
   - `process.assert` → `assert()` (checklist item 18)
   - HTTP/2 priority signaling calls (checklist item 20)
   - `http.OutgoingMessage._headers`/`._headersList` direct access (checklist item 23)
   - `--experimental-permission` flag → `--permission` (checklist item 26)
   - Legacy `util.is*`/`util._extend`/`util.log` calls, if any survived from a prior Node 20→22
     upgrade without being fixed (checklist item, Node 23.0.0 row) — these are removed now, not just
     warning.
   Where an official codemod exists (listed against each checklist item above), prefer running it
   over a manual rewrite — it's faster and less error-prone for mechanical renames.
6. **Switch local Node version and run the app once, expecting deprecation warnings**, not
   necessarily failures. Address each by category:
   - `url.parse()` → `new URL(...)` (checklist item 13)
   - `fs.F_OK`/etc. → `fs.constants.*` (checklist item 16)
   - `child_process` `args` + `shell: true` (checklist item 24)
   - GCM `authTagLength` (checklist item 25)
   - `zlib` classes without `new` (checklist item 27)
   - `SlowBuffer` (checklist item 30)
   - `crypto.fips` (checklist item 29)
7. **Rebuild native addons** (checklist item 8): `rm -rf node_modules && npm ci` with the new Node
   version active. If a dependency has no Node 24-compatible prebuild and fails to build from source,
   check that the toolchain from step 4 actually meets the new minimums before assuming the
   dependency itself is broken.
8. **Re-run the full test suite**, paying particular attention to:
   - Tests using `assert.deepStrictEqual`/`assert.partialDeepStrictEqual` on `WeakMap`/`WeakSet`
     values (checklist item 22) — these will now fail identity-based comparisons that previously
     passed.
   - `fetch()`-exercising integration tests, given the undici 7.0.0 bump (Node 24.0.0 row).
   - TLS handshake tests against any service using older keys/ciphers, given the OpenSSL 3.5 default
     security-level change (Node 24.0.0 row) — this is the single highest-risk item in this upgrade
     for anything talking TLS to legacy systems.
   - HTTP/2 server shutdown/session-lifecycle tests, given the 24.0.0 session-tracking rework.
9. **Update CI to actually run on the new pinned version and toolchain**, and confirm a full green
   run — including the raised gcc/Xcode/architecture minimums from step 4 — before merging.

## Verification steps

- `node --version` in every environment (local, CI runner, container, deployed instance) reports
  the exact pinned `24.x.y` from step 1.
- Full test suite passes with zero new failures and no unaddressed `DeprecationWarning` output in
  test logs.
- `npm ls` (or equivalent) shows no native-addon install errors; `node -p process.versions.modules`
  matches across all environments.
- Manually verify TLS connectivity to every external service/database the project talks to over
  TLS — the OpenSSL 3.5 default security-level change (Node 24.0.0) is the most likely source of a
  silent-until-runtime failure in this upgrade, since a weak-key/weak-cipher handshake failure may
  not be exercised by unit tests that mock the network layer.
- If HTTP/2 is used, manually exercise server shutdown under active load to confirm the graceful-close
  behavior change (Node 24.0.0) doesn't drop in-flight requests unexpectedly.
- If the project has a staging environment, deploy there first and monitor logs for unexpected
  `DeprecationWarning` output and any TLS handshake failures before promoting to production.

## Rollback / risk notes

- Keep the Node 22.x version-pin values easily revertible — a straight revert of the version-pin
  commit (step 2) plus reinstalling `node_modules` under Node 22 is the fastest rollback path.
- The OpenSSL 3.5 default security-level change (Node 24.0.0) is the riskiest *runtime* behavior
  change in this upgrade: it can cause TLS handshakes to a legacy external service to fail in
  production even when all tests passed, if those tests didn't exercise the real TLS connection.
  Treat any TLS-handshake failure discovered post-deploy as a signal to check key/cipher strength
  first, not as an unrelated network issue.
- The platform-minimum changes (architecture, macOS version, compiler toolchain) are infrastructure
  changes with their own rollback path (revert the CI/Docker image change) independent of the
  application code — keep that commit separate from the application-code fixes so either can be
  rolled back without the other.
- Native addon rebuild failures (step 7) are, as with the 20→22 upgrade, the most likely source of a
  blocked upgrade — resolve these in a branch before touching CI/production version pins.

## References

- [Node.js v22 to v24 migration guide](https://nodejs.org/en/blog/migrations/v22-to-v24) — official summary of platform-support drops, OpenSSL 3.5 default security level, toolchain minimums, and 8 available codemods (fetched in full).
- `nodejs/node` repo, `doc/changelogs/CHANGELOG_V23.md` (all 14 releases, 23.0.0–23.11.1) — fetched in full; every SEMVER-MAJOR-tagged commit and Notable-Changes section reviewed.
- `nodejs/node` repo, `doc/changelogs/CHANGELOG_V24.md` (all releases 24.0.0–24.20.0, the latest found at research time) — fetched in full; every SEMVER-MAJOR-tagged commit and Notable-Changes section reviewed.
- [Node.js release schedule / previous releases](https://nodejs.org/en/about/previous-releases) — LTS/EOL status and dates.
- [Node.js DEP0182 — Short GCM authentication tags without explicit authTagLength](https://nodejs.org/api/deprecations.html#dep0182-shortgcmauthtaglength) and [DEP0154, DEP0176, DEP0178, DEP0194, DEP0064, DEP0081, DEP0100](https://nodejs.org/api/deprecations.html) — consulted for the exact deprecation numbers referenced by the official codemods.
- `nodejs/userland-migrations` GitHub repo — source and examples for the 8 codemods referenced throughout this guide's checklist and breaking-changes sections.

**Assumption flagged for user review:** the "latest Node 24.x patch" and current LTS/EOL status
above reflect this guide's research date (~September 2026, per the fetched changelog's most recent
entry, 24.20.0 dated 2026-08-26). Re-verify the current latest 24.x patch at implementation time
(checklist item 21) rather than trusting this snapshot indefinitely. The exact `NODE_MODULE_VERSION`
ABI numbers cited in the breaking-changes tables reflect intermediate pre-release commits and may
not exactly match the final numbers for the specific patch actually installed — always confirm via
`node -p process.versions.modules` on the real installed binaries (checklist item 8) rather than
trusting the numbers in this guide.
