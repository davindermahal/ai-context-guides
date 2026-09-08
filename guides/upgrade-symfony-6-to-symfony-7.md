# Upgrade a Symfony 6 Project to Symfony 7 (PHPUnit + Ubuntu-based Docker image)

Purpose: upgrade a Symfony 6.4 (LTS) app — PHPUnit via `symfony/phpunit-bridge`, Docker image `FROM
ubuntu:<version>` with PHP installed via `apt` — to Symfony 7.4 (LTS), with the Docker image
rebuilt on a PHP version Symfony 7.4 supports (≥ 8.2.0).

**Rules for following this guide:**
1. Run every command exactly as written. Do not substitute a different command that seems similar.
2. Match command output against the table/list given for that command. Do not skip this step.
3. If output matches no row/case listed: STOP. Tell the user the exact command and exact output.
   Do not guess the closest match.
4. Every "if/then" below is a complete decision table. If a case isn't listed, that's rule 3, not a
   judgment call.
5. Do the Phases in order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12. Do not skip ahead
   unless a step explicitly says to.

**Why this guide has more phases than the 5→6 one:** Symfony 6.4 and 7.0 were released
simultaneously — 7.0 is 6.4 with every deprecated feature actually removed, and nothing more. That
makes `UPGRADE-7.0.md` the authoritative list of hard breaks (Phases 2–7, 10). But this guide lands
on 7.4, not bare 7.0, and 7.1–7.4 each add their own new deprecations on top (preparing for a future
8.0) — so Phase 11 repeats the deprecation-zeroing pass *after* the version bump, not just before it.

## When to use this guide

Use when ALL of these are true:
- [ ] `composer.json` requires `symfony/*` at `^6.x`
- [ ] Docker image: `FROM ubuntu:*` + PHP installed via `apt`/PPA (NOT `FROM php:*`)
- [ ] Tests run via `symfony/phpunit-bridge` (`bin/phpunit` or `vendor/bin/simple-phpunit`)

Do NOT use for:
- Symfony 5→6 upgrades — use [`upgrade-symfony-5-to-symfony-6.md`](upgrade-symfony-5-to-symfony-6.md) first
- Symfony 7→8 upgrades (different guide, doesn't exist yet as of this writing)
- `FROM php:8.x-fpm`-style images (Phase 9 doesn't apply — just edit the tag)
- Greenfield Symfony 7 projects (this is migration-only)

## Prerequisites — verify, do not assume

| # | Check | Command | Pass condition |
|---|---|---|---|
| P1 | Composer-managed | `test -f composer.json && echo yes \|\| echo no` | `yes` — if `no`, STOP, guide doesn't apply |
| P2 | On Symfony 6.4 | See Investigation #1 | Must show `v6.4.x` before Phase 1 |
| P3 | PHP ≥ 8.2.0 available | See Investigation #2 | Confirmed in Phase 9, not assumed from `composer.json` alone |
| P4 | Test suite exists | `test -f phpunit.xml.dist -o -f phpunit.dist.xml && echo yes \|\| echo no` | `yes` — if `no`, STOP and tell user before proceeding (upgrade is riskier without tests) |

If P2 fails (project on 6.0–6.3): run `composer require "symfony/*:6.4.*"`, then
`composer update "symfony/*"`, then fix any deprecations `./bin/phpunit` reports, then re-run
Investigation #1 until it shows `6.4.x`. Only then continue.

## Investigation checklist

Run every command below first. Keep the results — phases reference them by number.

### #1 — Current Symfony version

```bash
grep -A2 '"name": "symfony/framework-bundle"' composer.lock | grep '"version"'
```

| Output starts with | Meaning | Action |
|---|---|---|
| `v6.4.` | On 6.x LTS | Go to Phase 1 |
| `v6.0.`–`v6.3.` | Earlier 6.x | Do the P2 fix above first |
| `v7.` or higher | Already past Symfony 6 | STOP — guide doesn't apply |
| `v5.` or lower | Not yet on Symfony 6 | STOP — use `upgrade-symfony-5-to-symfony-6.md` first |
| (no output) | Package not locked under this name | Re-run with `"symfony/symfony"` instead of `"symfony/framework-bundle"`, apply same table |

### #2 — PHP version vs. Dockerfile

```bash
grep '"php"' composer.json
grep -A5 '"platform-overrides"' composer.lock
grep -n -E '^FROM|php[0-9]\.[0-9]|PHP_VERSION|ondrej' Dockerfile
```

| Source | What it tells you | Trust level |
|---|---|---|
| `composer.json` `require.php` | Declared minimum, e.g. `"php": "^8.1"` | Low — can be stale |
| `composer.lock` `platform-overrides` | Only non-empty if `config.platform.php` is explicitly set in `composer.json`. Empty = normal, tells you nothing. Non-empty = Composer is told to pretend this version is real | Untrustworthy — may mask real mismatch |
| Dockerfile | What actually runs in prod/CI | **Ground truth — use this** |

If the three disagree: Dockerfile wins. Report the discrepancy to the user.

Ubuntu release → default `apt` PHP version (no PPA), read off the `FROM` line match:

| `FROM` line contains | Ubuntu release | Default PHP | Meets Symfony 7.4's PHP ≥ 8.2.0 floor? |
|---|---|---|---|
| `ubuntu:18.04` / `ubuntu:bionic` | 18.04 bionic | 7.2 | No |
| `ubuntu:20.04` / `ubuntu:focal` | 20.04 focal | 7.4 | No |
| `ubuntu:22.04` / `ubuntu:jammy` | 22.04 jammy | 8.1 | **No — one minor short.** (This was sufficient for the 5→6 guide's 6.4 target; it is not sufficient here.) |
| `ubuntu:24.04` / `ubuntu:noble` | 24.04 noble | 8.3 | Yes |
| anything else / multiple `FROM` lines | — | — | STOP — tell user which `FROM` lines exist, ask which stage ships, and confirm its default PHP version before assuming it meets the floor |

From the third grep's output, also record:
- Every `php<version>-<extension>` package name (e.g. `php8.1-fpm`, `php8.1-xml`,
  `php8.1-mbstring`) → Phase 9 needs version-bumped equivalents for each.
- Whether `ondrej` appears anywhere → PPA already configured; changes what Phase 9 adds.

### #3 — Direct dependency list

```bash
composer show -D
```

Output: one line per direct dependency, format `vendor/package   1.2.3   Description`. Save it.
Phase 7 uses this to identify current versions when Composer reports a conflict — no manual
Packagist research needed up front.

### #4 — Security: `enable_authenticator_manager` config key

```bash
grep -n 'enable_authenticator_manager' config/packages/security.yaml
```

| Result | Meaning | Action |
|---|---|---|
| Match found | Key still present (it was mandatory-`true` since Symfony 6.0) | Phase 2 removes it — Symfony 7.0 deletes the option from the schema entirely, and a security config with an unrecognized key fails to compile |
| No match | Already absent | Phase 2 is a no-op — confirm and move on |

### #5 — Doctrine annotations vs. PHP attributes

```bash
grep -rl '@ORM\\\|@Route(\|@Assert\\' src/ 2>/dev/null | wc -l
grep -q '"doctrine/annotations"' composer.json && echo "ANNOTATIONS_PKG=yes" || echo "ANNOTATIONS_PKG=no"
```

| First command | `ANNOTATIONS_PKG` | Meaning | Action |
|---|---|---|---|
| `0` | either | No live annotation usage | No action needed |
| `> 0` | `no` | Stray annotation-style docblocks, reader not installed | Dead comments, not live config — confirm with user before touching |
| `> 0` | `yes` | Entities/routes/validation use live Doctrine annotations | **Mandatory in this guide** (unlike the 5→6 guide, where this was optional) — Symfony 7.0 removes the annotations integration entirely from FrameworkBundle, Routing, Serializer, and Validator. The app will not boot on 7.0 with live annotations still in place |

Do not mix annotations and attributes on the same class — migrate one whole class at a time.

### #6 — PHPUnit setup

```bash
grep -n 'SYMFONY_PHPUNIT_VERSION\|SYMFONY_MAX_PHPUNIT_VERSION' phpunit.xml.dist phpunit.dist.xml 2>/dev/null
```

(Recent Symfony skeletons renamed the file `phpunit.xml.dist` → `phpunit.dist.xml`; check both.)

- Output like `<server name="SYMFONY_PHPUNIT_VERSION" value="9.6"/>` → Phase 12 bumps it.
- No output → nothing pinned; Phase 12 adds an explicit pin.

### #7 — Console commands using the old command-naming API

```bash
grep -rln '\$defaultName\s*=\|\$defaultDescription\s*=' src/Command/
```

Any match → Phase 4 migrates that command class. `Command::$defaultName`/`$defaultDescription` are
removed in 7.0.

### #8 — Messenger handlers using the old handler API

```bash
grep -rln 'implements MessageHandlerInterface\|implements MessageSubscriberInterface' src/
```

Any match → Phase 5 migrates that handler class. Both interfaces are removed in 7.0.

### #9 — Other deprecated/removed dependencies

No command — cross-reference step. Done in Phase 10, against the #3 dependency list and
`https://github.com/symfony/symfony/blob/7.0/UPGRADE-7.0.md`.

### #10 — CI pipeline

```bash
ls -a .github/workflows .gitlab-ci.yml 2>/dev/null
```

Record which files exist — they likely build/pull the same Docker image Phase 9 changes.

## Step-by-step implementation plan

### Phase 1 — Land on 6.4, zero deprecations

1. `BRIDGE` check: `grep -q '"symfony/phpunit-bridge"' composer.json && echo yes || echo no`. If
   `no` → `composer require --dev symfony/phpunit-bridge`.
2. Run `./bin/phpunit` (or `./vendor/bin/simple-phpunit`). Read the `Remaining deprecation notices`
   block at the end. Fix each one; re-run after each fix or small batch.
3. Load app with `APP_ENV=dev` in a browser. Check web debug toolbar's deprecation counter for
   anything CLI tests missed (template/routing deprecations only surface on HTTP requests).
4. Repeat steps 2–3 until the count is `0`. Do not start Phase 2 with a nonzero count. Every 6.4
   deprecation fixed here is a 7.0 fatal error avoided later — this is the single most important
   phase in this guide.

### Phase 2 — Remove `enable_authenticator_manager`

Skip if Investigation #4 found no match.

1. Delete the `enable_authenticator_manager: true` line from `config/packages/security.yaml`
   entirely — do not set it to `false`, just remove the key.
2. ```bash
   php bin/console debug:config security
   ```
   Exit `0` with no "Unrecognized option" error → done. Error naming this key → the line wasn't
   fully removed; re-check for a duplicate under a per-environment override
   (`config/packages/{dev,prod,test}/security.yaml`).

### Phase 3 — Migrate Doctrine annotations to PHP attributes

Skip entirely if Investigation #5 found no live annotation usage.

1. For each file Investigation #5's first grep listed, convert one class at a time:
   ```php
   // Before (Doctrine annotation)
   /**
    * @ORM\Entity
    * @ORM\Table(name="product")
    */
   class Product { /* ... */ }

   // After (PHP 8 attribute)
   #[ORM\Entity]
   #[ORM\Table(name: 'product')]
   class Product { /* ... */ }
   ```
   Same pattern for `@Route(...)` → `#[Route(...)]` and `@Assert\...` → `#[Assert\...]`.
2. Once a class's annotations are fully replaced, remove its `use` import for the annotation-style
   base class if one existed, and confirm the attribute-style import is present instead.
3. After every listed file is converted:
   ```bash
   composer remove doctrine/annotations
   ```
   Only if nothing else in the project still needs it — re-run Investigation #5's first grep first
   to confirm `0`.
4. Run `./bin/phpunit`. Any `Class ... not found` or mapping errors here mean a class was missed —
   re-run Investigation #5's grep to find it.

### Phase 4 — Migrate console commands to `#[AsCommand]`

Skip entirely if Investigation #7 found no match.

1. For each file listed:
   ```php
   // Before
   class CreateUserCommand extends Command
   {
       protected static $defaultName = 'app:create-user';
       protected static $defaultDescription = 'Creates users';
   }

   // After
   #[AsCommand(name: 'app:create-user', description: 'Creates users')]
   class CreateUserCommand extends Command
   {
   }
   ```
2. ```bash
   php bin/console list | grep 'app:create-user'
   ```
   (substitute the real command name) — command still listed with the same name → migration
   succeeded for that command.

### Phase 5 — Migrate Messenger handlers to `#[AsMessageHandler]`

Skip entirely if Investigation #8 found no match.

1. For each file listed:
   ```php
   // Before
   class SmsNotificationHandler implements MessageHandlerInterface
   {
       public function __invoke(SmsNotification $message): void { /* ... */ }
   }

   // After
   #[AsMessageHandler]
   class SmsNotificationHandler
   {
       public function __invoke(SmsNotification $message): void { /* ... */ }
   }
   ```
   A class implementing `MessageSubscriberInterface` with multiple handled messages becomes one
   `#[AsMessageHandler]` attribute per method (see `UPGRADE-7.0.md`'s Messenger section for the
   multi-method form).
2. ```bash
   php bin/console debug:messenger
   ```
   Confirm every migrated handler is still listed against the same message class.

### Phase 6 — Add native PHP return/property types (second pass)

Symfony 7.0 adds another round of native return and property types on top of Symfony 6.0's — the
same class of risk as the 5→6 guide's Phase 3, and the same fix.

1. ```bash
   test -f vendor/bin/patch-type-declarations && echo yes || echo no
   ```
   `no` → `composer require symfony/error-handler`, re-check.
2. ```bash
   composer dump-autoload -o
   SYMFONY_PATCH_TYPE_DECLARATIONS="force=2" vendor/bin/patch-type-declarations
   ```
3. ```bash
   git diff --stat
   ```
   Review — the tool only adds return/property-type declarations, it doesn't change logic. Run
   `./bin/phpunit` after; new failures mean a method's actual return value doesn't match the
   inferred type — fix the method body, not the type.

### Phase 7 — Bump Composer constraints to 7.4

1. Edit `composer.json`: every `"symfony/<name>": "6.4.*"` or `"^6.4"` → `"^7.4"`. Confirm:
   ```bash
   grep '"symfony/' composer.json
   ```
   Every line shows `7.4` — none show `6.4`. Delete any `"symfony/symfony": "..."` line entirely
   (deprecated metapackage).
2. Run:
   ```bash
   composer update "symfony/*" --with-all-dependencies 2>&1 | tee /tmp/composer-update.log
   ```
3. Check `/tmp/composer-update.log`:
   | Log contains | Meaning | Action |
   |---|---|---|
   | Line matching `Package operations:` | Success | Go to step 5 |
   | Block(s) starting `Problem 1`, `Problem 2`, etc. | Conflict(s) | Go to step 4, once per named package |
4. For each conflicting package (named on the first indented line of its `Problem` block, before
   the version number):
   ```bash
   composer why-not symfony/framework-bundle 7.4.0
   ```
   (substitute the actual `symfony/*` package from the `Problem` block)
   - Lists every package blocking that version + its exact constraint.
   - `composer show <package> --all` → list available versions.
   - `composer show <package> <version>` → check that version's own `require` for a
     `symfony/*: ^7.0`-compatible line.
   - Found a compatible version → widen its constraint in `composer.json`, re-run step 2.
   - No compatible version exists anywhere → STOP. Do NOT use `--ignore-platform-reqs` or remove
     the package unasked. Tell the user the exact package name; let them decide.
5. Repeat steps 2–4 until step 2's command completes with no `Problem` blocks.
6. Error mentions `requires php` + `does not satisfy that requirement`? That's Phase 9's concern —
   note it, keep resolving dependency versions, don't fix PHP here.
7. `ProxyManagerBridge` (`symfony/proxy-manager-bridge`) conflict or removal notice? The bridge is
   removed entirely in 7.0 — remove it from `composer.json` and switch any direct usage to
   VarExporter's native lazy objects instead.

### Phase 8 — Flex recipes / config files

1. `FLEX_PLUGIN` check: `grep -q '"name": "symfony/flex"' composer.lock && echo yes || echo no`.
   `yes`:
   ```bash
   git add -A && git commit -m "Bump Symfony to 7.4"
   composer recipes
   ```
   For each package marked `outdated`: `composer recipes:update <package-name>` (one at a time),
   then `git diff` to review before moving to the next package.
2. `FLEX_PLUGIN=no`, or a package has no recipe update:
   ```bash
   composer create-project symfony/skeleton:"7.4.*" /tmp/symfony-7.4-skeleton
   ```
   Manually diff project's `config/packages/*.yaml`, `config/routes.yaml`, `config/services.yaml`,
   `.env` against the skeleton's equivalents. Apply differences by hand — preserve the project's
   actual config values, don't overwrite wholesale.
3. Review every config default this table changed — these are **silent behavior changes**, not
   errors, so nothing will flag them automatically:

   | Option | Old default (<7.0) | New default (7.0+) | Action |
   |---|---|---|---|
   | `framework.http_method_override` | `true` | `false` | If the app relies on `_method=PUT`-style form overrides, set it back to `true` explicitly |
   | `framework.handle_all_throwables` | `false` | `true` | Non-`\Exception` throwables (e.g. `\Error`) now get converted to HTTP responses too — confirm no code depends on them propagating as fatal errors |
   | `framework.php_errors.log` | `'%kernel.debug%'` | `true` | PHP errors are now always logged, even in prod with debug off — check log volume/retention |
   | `framework.session.cookie_secure` | `false` | `'auto'` | Cookie now `Secure` automatically when the request is HTTPS — fine for HTTPS-only apps, verify any HTTP-only internal environment |
   | `framework.session.cookie_samesite` | `null` | `'lax'` | Cross-site session cookie behavior changes — check any legitimate cross-origin flow (SSO, embedded iframe) |
   | `framework.uid.default_uuid_version` / `time_based_uuid_version` | `6` | `7` | New UUIDs generated after the upgrade are v7, not v6 — only matters if code parses/validates the UUID version explicitly |
   | `framework.validation.email_validation_mode` | `'loose'` | `'html5'` | Some previously-valid email addresses may now fail validation — re-test any email-format edge cases the app relies on |

### Phase 9 — Docker image → PHP ≥ 8.2.0

1. Investigation #2's table: does the current default-apt PHP meet the ≥ 8.2.0 floor? (Note:
   `ubuntu:22.04`'s default of 8.1 does **not** — this is a real bump even if Phase 6 of the 5→6
   guide already got the image to 8.1.)
2. Decision:

   | Condition | Action |
   |---|---|
   | Table shows PHP 8.3 (`ubuntu:24.04`), no PPA needed | No base-image change. Skip to step 5 |
   | Default PHP < 8.2, or Ubuntu release not one of the 4 table rows | Add `ondrej/php` PPA (below). Only if base image is an Ubuntu LTS release (18.04/20.04/22.04/24.04) — otherwise STOP, tell user |
   | Prefer bumping the base image itself | Change `FROM` line to `ubuntu:24.04` or newer, then re-check every other `apt-get install` line still exists under that name on the new release |

   `ondrej/php` PPA block (illustrative — substitute real target version + extension list from
   Investigation #2):
   ```dockerfile
   RUN apt-get update && apt-get install -y software-properties-common \
       && add-apt-repository ppa:ondrej/php \
       && apt-get update \
       && apt-get install -y \
          php8.2-fpm php8.2-cli \
          php8.2-xml php8.2-mbstring php8.2-intl php8.2-pgsql
          # one line per extension package from Investigation #2, version number updated
   ```
3. Rebuild without cache:
   ```bash
   docker build --no-cache -t symfony7-upgrade-test .
   ```
4. Verify:
   ```bash
   docker run --rm symfony7-upgrade-test php -v
   ```
   Output matches target exactly? If not (multiple PHP versions installed, `$PATH` points at old
   one): add `RUN update-alternatives --set php /usr/bin/php<target-version>`, rebuild, re-check.
5. Re-run Investigation #2's third grep against the updated Dockerfile. Every match must reflect the
   new version — none may still reference the old one.
6. Cross-check extensions:
   ```bash
   grep '"ext-' composer.json
   ```
   Every `ext-<name>` printed → matching `php<version>-<name>` apt package must exist in Dockerfile.

### Phase 10 — Symfony-7 breaking changes sweep

Full authoritative list: `https://github.com/symfony/symfony/blob/7.0/UPGRADE-7.0.md`. Below: the
changes most likely to still be present in an unmigrated 6.4 app (items already removed since 6.0
are excluded — they can't be present in a working 6.4 codebase). Run every grep; a match means that
fix is required, not optional.

| Grep | Fix |
|---|---|
| `grep -rln 'ContainerAwareInterface\|ContainerAwareTrait' src/` | Both removed — inject the needed service via constructor instead of pulling from `$this->container->get(...)` |
| `grep -rln 'ObjectNormalizer \$\|PropertyNormalizer \$' src/` (constructor parameter type-hints) | The autowiring aliases for these concrete classes are removed — type-hint `NormalizerInterface`/`DenormalizerInterface` instead, or explicitly `#[Autowire(service: 'serializer.normalizer.object')]` if the concrete implementation is required |
| `grep -rln 'CacheableSupportsMethodInterface\|ContextAwareNormalizerInterface\|ContextAwareDenormalizerInterface' src/` | All three removed — implement `getSupportedTypes(?string $format): array` on `NormalizerInterface`/`DenormalizerInterface` directly instead |
| `grep -rln 'symfony/templating' composer.json` | Templating component fully removed — if still required, migrate remaining templates to Twig first |
| `grep -n 'autoescape:' config/packages/twig.yaml` | The `twig.autoescape` config option is removed — implement a class per `FileExtensionEscapingStrategy::guess()` and reference it via `twig.autoescape_service` instead |
| `grep -q '"twig/twig": "^2' composer.json && echo yes` | Twig 2 support dropped — bump to Twig 3 |
| `grep -rln 'new RequestMatcher(' src/` | `RequestMatcher`'s multi-condition constructor is removed — use `ChainRequestMatcher` composing single-condition matchers instead |
| `grep -rln 'StopWorkerOnSigtermSignalListener\|StopWorkerOnSignalsListener' src/ config/` | Both removed — implement `SignalableCommandInterface` on the consuming command instead |
| `grep -rln 'Transport\\\\InMemoryTransport\b' src/` (direct `use` of the old namespace, not the `in-memory://` DSN) | Namespace moved to `Transport\InMemory\InMemoryTransport` |
| `grep -rln "'silent'" src/Command/` (option literally named `silent` on a custom command) | Symfony 7.2 adds a global `--silent` option — rename the custom one to avoid the collision |

General sweep: run the full test suite (after Phase 12). Every fatal `Error` on an unknown
class/method not covered by the table above → search `UPGRADE-7.0.md` for that exact name.

### Phase 11 — Deprecation sweep against 7.1–7.4

Unlike a same-target-version guide, landing on 7.4 means picking up every deprecation introduced by
7.1, 7.2, 7.3, and 7.4 individually — none of these were visible while still on 6.4, and none of
them are hard breaks yet, but leaving them unresolved makes the eventual 8.0 upgrade harder later.

1. Run `./bin/phpunit` again, now against 7.4. Read the `Remaining deprecation notices` block.
2. For each one, cross-reference `UPGRADE-7.1.md` through `UPGRADE-7.4.md` in the
   `symfony/symfony` repo (same repo, matching branch) by component name to find the recommended
   replacement.
3. Fix or explicitly defer each with a reason recorded in the final report — do not silently ignore
   a nonzero count here the way Phase 1 didn't allow one before the major bump.

### Phase 12 — PHPUnit → Symfony-7-compatible version

1. `phpunit.xml.dist`/`phpunit.dist.xml`: existing `SYMFONY_PHPUNIT_VERSION` entry → set to the
   current latest `11.x` patch (check `https://packagist.org/packages/phpunit/phpunit#11` first —
   don't hardcode blindly). No entry → add inside `<php>`:
   ```xml
   <php>
       <server name="SYMFONY_PHPUNIT_VERSION" value="11.5"/>
   </php>
   ```
   (Illustrative version — confirm the real current `11.x` patch before writing it.)
2. Existing `SYMFONY_MAX_PHPUNIT_VERSION` entry → remove it. Re-add only if a specific dependency's
   incompatibility with PHPUnit 11 shows up after step 4.
3. ```bash
   rm -rf vendor/bin/.phpunit
   ```
4. Run `./bin/phpunit` or `./vendor/bin/simple-phpunit`. Two failure categories — fix separately:
   - PHPUnit method/annotation no longer exists (removed between 9.x and 11.x, e.g.
     `@expectedException`-style doc-comment annotations) → test-code fix, migrate to attributes
     (`#[Test]`, `#[DataProvider]`, etc.) or the modern assertion equivalent.
   - Application code failure → continue Phase 10's removed-API sweep.
5. Repeat until `0 failures, 0 errors`.

## Verification steps

| # | Check | Command / method |
|---|---|---|
| 1 | Suite passes in the rebuilt image | `docker run --rm symfony7-upgrade-test ./bin/phpunit` |
| 2 | Container/config sane | `php bin/console debug:container` and `debug:config` both exit `0` |
| 3 | Zero deprecations | Boot `APP_ENV=dev` in browser, web debug toolbar deprecation counter = `0` — any nonzero count left over from Phase 11 must be an explicitly recorded, deliberate deferral, not an oversight |
| 4 | Auth works | Manually exercise every auth path (form login, API token, custom authenticator) |
| 5 | Console commands work | Run every command migrated in Phase 4 with `--help`, confirm name/description intact |
| 6 | Messenger works | Dispatch one real message per handler migrated in Phase 5, confirm consumed |
| 7 | Config-default behavior review done | Every row of Phase 8 step 3's table has an explicit yes/no answer for this app, not left unexamined |
| 8 | Clean-cache rebuild | `docker build --no-cache -t symfony7-upgrade-final .`, re-run checks 1–3 against it |
| 9 | Sources agree | Re-run Investigation #2's 3 commands; `composer.json`, `composer.lock`, Dockerfile now consistent (or discrepancy was flagged on purpose) |

## Rollback / risk notes

- Phase 1 (deprecation fixes) and Phase 6 (return-type patches) are their own commits on the
  still-6.4 codebase, merged independently before Phase 7 → safe to keep even if everything after
  is rolled back.
- Phases 2–5 (security config, annotations, commands, messenger) are each independent, mechanical
  migrations with no version-bump dependency — land and verify each as its own PR before Phase 7,
  so a regression in one is bisectable from the others and from the dependency bump itself.
- Phase 7 onward: dedicated branch. `composer.json`/`composer.lock` diff + Dockerfile diff are
  coupled — revert together. Never ship the Phase 9 PHP bump without the Phase 7 Symfony bump (old
  Symfony 6 code on untested new PHP, no passing test run backing it).
- Phase 8 step 3's config-default changes are behavioral, not code — a production incident traced to
  one of those seven options is fixed by pinning the old value explicitly, not by reverting the
  whole upgrade.
- Keep the pre-upgrade Docker image tag available until the new one has run clean through one full
  staging deployment cycle.
- Phase 11 deprecations deferred → record explicitly in PR/task description with the reason. They
  become Symfony 8's hard breaks, same relationship as this guide's own Phase 1/Phase 10 pairing.

## References

- [Symfony 7.0 Release](https://symfony.com/releases/7.0) — PHP requirement (≥8.2.0); note 7.0
  itself reached end-of-support July 2024, which is why this guide lands on 7.4
- [Symfony 7.4 Release](https://symfony.com/releases/7.4) — LTS status, PHP requirement (≥8.2.0),
  bug-fix support to November 2028 / security support to November 2029
- [Symfony | endoflife.date](https://endoflife.date/symfony) — support/security-EOL dates (re-check;
  dates shift)
- [UPGRADE-7.0.md](https://github.com/symfony/symfony/blob/7.0/UPGRADE-7.0.md) — authoritative 6→7
  breaking-change list; read directly (raw file), not summarized, to build Phase 10's table.
  Confirmed the 6.4→7.0 relationship stated in this guide's intro directly from the file's own
  opening paragraph
- [UPGRADE-7.1.md](https://github.com/symfony/symfony/blob/7.1/UPGRADE-7.1.md),
  [UPGRADE-7.2.md](https://github.com/symfony/symfony/blob/7.2/UPGRADE-7.2.md),
  [UPGRADE-7.3.md](https://github.com/symfony/symfony/blob/7.3/UPGRADE-7.3.md),
  [UPGRADE-7.4.md](https://github.com/symfony/symfony/blob/7.4/UPGRADE-7.4.md) — read directly and
  confirmed each is deprecations-only for typical app code, with narrow `[BC BREAK]` exceptions
  (the 7.2 `--silent` option and 7.4 PHP-config-file scope changes are the two folded into Phase 10;
  the rest are internal/rare enough to be left to Phase 11's general deprecation sweep instead of a
  dedicated grep)
- [Symfony 7.0 Type Declarations announcement](https://symfony.com/blog/symfony-7-0-type-declarations)
  — explains the second return/property-type patch round Phase 6 is based on
  ([UPGRADE-7.0.md](https://github.com/symfony/symfony/blob/7.0/UPGRADE-7.0.md) links this same post)
- [The PHPUnit Bridge (Symfony 7.4 docs)](https://symfony.com/doc/7.4/components/phpunit_bridge.html)
  — `SYMFONY_PHPUNIT_VERSION` / `SYMFONY_MAX_PHPUNIT_VERSION` mechanics; current docs example pins
  an `11.x` schema version, which is why Phase 12 targets `11.x` rather than repeating the 6.4
  guide's `9.6`
- [Composer `why-not`/`prohibits`](https://getcomposer.org/doc/03-cli.md) — used mechanically in
  Phase 7 to find blocking packages
- [Composer config platform reference](https://getcomposer.org/doc/06-config.md) — confirms
  `platform-overrides` only reflects an explicit override, not real ambient PHP (why Investigation
  #2 treats the Dockerfile as ground truth)
- Ubuntu default PHP-per-release mapping (18.04→7.2, 20.04→7.4, 22.04→8.1, 24.04→8.3) + `ondrej/php`
  PPA — general community knowledge; re-verify if applying to an Ubuntu release not listed, mapping
  shifts with each new LTS.
