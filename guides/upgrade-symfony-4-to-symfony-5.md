# Upgrade a Symfony 4 Project to Symfony 5 (PHPUnit + Ubuntu-based Docker image)

Purpose: upgrade a Symfony 4.x app — PHPUnit via `symfony/phpunit-bridge`, Docker image `FROM
ubuntu:<version>` with PHP installed via `apt` — to Symfony 5.4 (LTS), with the Docker image
rebuilt on a PHP version Symfony 5 supports.

**Rules for following this guide:**
1. Run every command exactly as written. Do not substitute a different command that seems similar.
2. Match command output against the table/list given for that command. Do not skip this step.
3. If output matches no row/case listed: STOP. Tell the user the exact command and exact output.
   Do not guess the closest match.
4. Every "if/then" below is a complete decision table. If a case isn't listed, that's rule 3, not a
   judgment call.
5. Do the Phases in order: 1 → 2 → 3 → 4 → 5 → 6 → 7. Do not skip ahead unless a step explicitly
   says to.

**Before anything else: determine the directory structure (Investigation #2).** Two possible
layouts:
- Pre-Flex: `app/AppKernel.php`, `app/config/*.yml`, `web/app.php`
- Flex: `config/bundles.php`, `config/packages/*.yaml`, `public/index.php`, flat `src/`

Every file path in Phases 2–7 assumes the Flex layout. Wrong layout detected = wrong paths used
everywhere else in this guide.

## When to use this guide

Use when ALL of these are true:
- [ ] `composer.json` requires `symfony/*` at `^4.x`
- [ ] Docker image: `FROM ubuntu:*` + PHP installed via `apt`/PPA (NOT `FROM php:*`)
- [ ] Tests run via `symfony/phpunit-bridge` (`bin/phpunit` or `vendor/bin/simple-phpunit`)

Do NOT use for:
- Symfony 5→6/7 upgrades (different guide)
- `FROM php:7.4-fpm`-style images (Investigation #4 / Phase 5 don't apply — just edit the tag)
- Greenfield Symfony 5 projects (this is migration-only)

Works regardless of pre-Flex vs. Flex layout — Phase 1 detects and branches.

## Prerequisites — verify, do not assume

| # | Check | Command | Pass condition |
|---|---|---|---|
| P1 | Composer-managed | `test -f composer.json && echo yes \|\| echo no` | `yes` — if `no`, STOP, guide doesn't apply |
| P2 | On Symfony 4.4 | See Investigation #1 | Must show `v4.4.x` before Phase 2 |
| P3 | PHP ≥ 7.2.5 available | See Investigation #4 | Confirmed in Phase 5, not assumed from `composer.json` alone |
| P4 | Test suite exists | `test -f phpunit.xml.dist -o -f phpunit.xml && echo yes \|\| echo no` | `yes` — if `no`, STOP and tell user before proceeding (upgrade is riskier without tests) |

If P2 fails (project on 4.0–4.3): run `composer require "symfony/*:4.4.*"`, then
`composer update "symfony/*"`, then do Phase 2's deprecation fixes, then re-run Investigation #1
until it shows `4.4.x`. Only then continue.

## Investigation checklist

Run every command below first. Keep the results — phases reference them by number.

### #1 — Current Symfony version

```bash
grep -A2 '"name": "symfony/framework-bundle"' composer.lock | grep '"version"'
```

| Output starts with | Meaning | Action |
|---|---|---|
| `v4.4.` | On 4.x LTS | Go to Phase 1 |
| `v4.0.`–`v4.3.` | Earlier 4.x | Do the P2 fix above first |
| `v5.` or higher | Already past Symfony 4 | STOP — guide doesn't apply |
| (no output) | Package not locked under this name | Re-run with `"symfony/symfony"` instead of `"symfony/framework-bundle"`, apply same table |

### #2 — Directory structure + Flex

```bash
test -f app/AppKernel.php && echo "PRE_FLEX_KERNEL=yes" || echo "PRE_FLEX_KERNEL=no"
test -f config/bundles.php && echo "FLEX_BUNDLES=yes" || echo "FLEX_BUNDLES=no"
test -f public/index.php && echo "FLEX_PUBLIC=yes" || echo "FLEX_PUBLIC=no"
grep -q '"name": "symfony/flex"' composer.lock && echo "FLEX_PLUGIN=yes" || echo "FLEX_PLUGIN=no"
```

| PRE_FLEX_KERNEL | FLEX_BUNDLES / FLEX_PUBLIC | Meaning | Action |
|---|---|---|---|
| no | yes / yes | Clean Flex | Skip Phase 1 → go to Phase 2 |
| yes | no / no | Clean pre-Flex | Do full Phase 1 |
| yes | yes / yes (or mixed) | Hybrid | Do full Phase 1, tell user it's a hybrid state |
| no | no / no | Neither found | STOP — tell user all 4 command outputs |

`FLEX_PLUGIN` is separate: tracks whether the Composer plugin is installed (needed for
`composer recipes:update`, Phase 4), independent of directory layout.

### #3 — Direct dependency list

```bash
composer show -D
```

Output: one line per direct dependency, format `vendor/package   1.2.3   Description`. Save it.
Phase 3 uses this to identify current versions when Composer reports a conflict — no manual
Packagist research needed up front.

### #4 — PHP version vs. Dockerfile

```bash
grep '"php"' composer.json
grep -A5 '"platform-overrides"' composer.lock
grep -n -E '^FROM|php[0-9]\.[0-9]|PHP_VERSION|ondrej' Dockerfile
```

| Source | What it tells you | Trust level |
|---|---|---|
| `composer.json` `require.php` | Declared minimum, e.g. `"php": "^7.2"` | Low — can be stale |
| `composer.lock` `platform-overrides` | Only non-empty if `config.platform.php` is explicitly set in `composer.json`. Empty = normal, tells you nothing. Non-empty = Composer is told to pretend this version is real | Untrustworthy — may mask real mismatch |
| Dockerfile | What actually runs in prod/CI | **Ground truth — use this** |

If the three disagree: Dockerfile wins. Report the discrepancy to the user.

Ubuntu release → default `apt` PHP version (no PPA), read off the `FROM` line match:

| `FROM` line contains | Ubuntu release | Default PHP |
|---|---|---|
| `ubuntu:18.04` / `ubuntu:bionic` | 18.04 bionic | 7.2 |
| `ubuntu:20.04` / `ubuntu:focal` | 20.04 focal | 7.4 |
| `ubuntu:22.04` / `ubuntu:jammy` | 22.04 jammy | 8.1 |
| `ubuntu:24.04` / `ubuntu:noble` | 24.04 noble | 8.3 |
| anything else / multiple `FROM` lines | — | STOP — tell user which `FROM` lines exist, ask which stage ships |

From the third grep's output, also record:
- Every `php<version>-<extension>` package name (e.g. `php7.4-fpm`, `php7.4-xml`,
  `php7.4-mbstring`) → Phase 5 needs version-bumped equivalents for each.
- Whether `ondrej` appears anywhere → PPA already configured; changes what Phase 5 adds.

### #5 — PHPUnit setup

```bash
grep -q '"symfony/phpunit-bridge"' composer.json && echo "BRIDGE=yes" || echo "BRIDGE=no"
grep -n 'SYMFONY_PHPUNIT_VERSION\|SYMFONY_MAX_PHPUNIT_VERSION' phpunit.xml.dist
```

- `BRIDGE=no` → Phase 2 adds it.
- Second command output like `<server name="SYMFONY_PHPUNIT_VERSION" value="7.5"/>` → that's the
  currently pinned version; Phase 7 bumps it.
- No output → nothing pinned; Phase 7 adds an explicit pin.

### #6 — Mailer

```bash
grep -q '"symfony/swiftmailer-bundle"' composer.json && echo "SWIFTMAILER=yes" || echo "SWIFTMAILER=no"
```

`SWIFTMAILER=yes` → Phase 6 Pitfall A is required, not optional.

### #7 — Security config

```bash
FILE=config/packages/security.yaml
test -f "$FILE" || FILE=app/config/security.yml
grep -n 'enable_authenticator_manager' "$FILE"
echo "SECURITY_FILE=$FILE"
```

Record `$FILE` found and whether the grep matched. No match → legacy security system → Phase 6
Pitfall B applies as a recommendation.

### #8 — Other deprecated/removed dependencies

No command — cross-reference step. Done in Phase 3 step 3, against the #3 dependency list and
`https://github.com/symfony/symfony/blob/5.4/UPGRADE-5.0.md`.

### #9 — CI pipeline

```bash
ls -a .github/workflows .gitlab-ci.yml 2>/dev/null
```

Record which files exist — they likely build/pull the same Docker image Phase 5 changes.

## Step-by-step implementation plan

### Phase 1 — Directory structure

Skip entirely if Investigation #2 said "Clean Flex."

1. `FLEX_PLUGIN=no` → run `composer require symfony/flex`.
2. Get a reference skeleton (don't invent file contents from memory):
   ```bash
   composer create-project symfony/skeleton:"4.4.*" /tmp/symfony-4.4-skeleton
   ```
3. Create `public/index.php`, copying from `/tmp/symfony-4.4-skeleton/public/index.php`.
4. Create `config/bundles.php`. One line per bundle in `app/AppKernel.php`'s `registerBundles()`:
   ```php
   <?php
   return [
       Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
       Symfony\Bundle\TwigBundle\TwigBundle::class => ['all' => true],
       // one line per bundle in registerBundles(); dev/test-only bundles get
       // ['dev' => true] / ['test' => true] instead of an environment if-check
   ];
   ```
5. For each bundle in step 4, one at a time:
   - Create/update `config/packages/<bundle_alias>.yaml` (+ `config/packages/{dev,prod,test}/<bundle_alias>.yaml` as needed) from the matching block in `app/config/config.yml` / `config_dev.yml` / `config_prod.yml` / `config_test.yml`.
   - Run `php bin/console cache:clear` then `php bin/console debug:config <bundle_alias>`.
   - Error? Fix it now. Do not move to the next bundle with a known error still present.
6. Migrate `app/config/parameters.yml`:
   - Each `parameter_name: value` → add `PARAMETER_NAME=value` to `.env.local`.
   - `grep -rn '%parameter_name%' config/ src/` for every usage.
   - Confirm each resolves via `debug:container --parameters` or the matching `debug:config` from step 5.
7. Verify (do not proceed until all pass):
   - `php bin/console cache:clear --env=prod` → exit 0
   - `php bin/console cache:clear --env=dev` → exit 0
   - `./bin/phpunit` (or `./vendor/bin/simple-phpunit`) → same pass rate as before this phase
8. Only after step 7 passes: delete `app/` and `web/`, as their own commit, separate from any
   Symfony-5 change.
9. If the user decides to defer this migration: STOP, confirm with user explicitly, and record in
   final report that Phase 4/6/7's `config/packages/*`-style paths must be substituted by hand with
   `app/config/*.yml` equivalents.

### Phase 2 — Land on 4.4, zero deprecations

1. `BRIDGE=no` (Investigation #5) → `composer require --dev symfony/phpunit-bridge`.
2. Run `./bin/phpunit` (or `./vendor/bin/simple-phpunit`). Read the `Remaining deprecation notices`
   block at the end. Fix each one; re-run after each fix or small batch.
3. Load app with `APP_ENV=dev` in a browser. Check web debug toolbar's deprecation counter for
   anything CLI tests missed (template/routing deprecations only surface on HTTP requests).
4. Repeat steps 2–3 until the count is `0`. Do not start Phase 3 with a nonzero count.

### Phase 3 — Bump Composer constraints to 5.4

1. Edit `composer.json`: every `"symfony/<name>": "4.4.*"` or `"^4.4"` → `"^5.4"`. Confirm:
   ```bash
   grep '"symfony/' composer.json
   ```
   Every line shows `5.4` — none show `4.4`. Delete any `"symfony/symfony": "..."` line entirely
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
   composer why-not symfony/framework-bundle 5.4.0
   ```
   (substitute the actual `symfony/*` package from the `Problem` block)
   - Lists every package blocking that version + its exact constraint.
   - `composer show <package> --all` → list available versions.
   - `composer show <package> <version>` → check that version's own `require` for a
     `symfony/*: ^5.0`-compatible line.
   - Found a compatible version → widen its constraint in `composer.json`, re-run step 2.
   - No compatible version exists anywhere → STOP. Do NOT use `--ignore-platform-reqs` or remove
     the package unasked. Tell the user the exact package name; let them decide.
5. Repeat steps 2–4 until step 2's command completes with no `Problem` blocks.
6. Error mentions `requires php` + `does not satisfy that requirement`? That's Phase 5's concern —
   note it, keep resolving dependency versions, don't fix PHP here.

### Phase 4 — Flex recipes / config files

If Phase 1 step 9 recorded a deferred migration: skip step 1, use step 2 for every file.

1. `FLEX_PLUGIN=yes`:
   ```bash
   git add -A && git commit -m "Bump Symfony to 5.4"
   composer recipes
   ```
   For each package marked `outdated`: `composer recipes:update <package-name>` (one at a time),
   then `git diff` to review before moving to the next package.
2. `FLEX_PLUGIN=no`, or a package has no recipe update:
   ```bash
   composer create-project symfony/skeleton:"5.4.*" /tmp/symfony-5.4-skeleton
   ```
   Manually diff project's `config/packages/*.yaml`, `config/routes.yaml`, `config/services.yaml`,
   `.env` against the skeleton's equivalents. Apply differences by hand — preserve the project's
   actual config values, don't overwrite wholesale.

### Phase 5 — Docker image → compatible PHP

1. Investigation #4's table: current default-apt PHP < 7.4? Treat as needing upgrade even if it
   technically meets Symfony's 7.2.5 floor (Doctrine/PHPUnit 9/most modern packages assume ≥7.4).
2. Decision:

   | Condition | Action |
   |---|---|
   | Table shows PHP 7.4–8.1, no PPA needed | No base-image change. Skip to step 5 |
   | Default PHP < 7.4, or Ubuntu release not one of the 4 table rows | Add `ondrej/php` PPA (below). Only if base image is an Ubuntu LTS release (18.04/20.04/22.04/24.04) — otherwise STOP, tell user |
   | Prefer bumping the base image itself | Change `FROM` line, then re-check every other `apt-get install` line still exists under that name on the new release |

   `ondrej/php` PPA block (illustrative — substitute real target version + extension list from
   Investigation #4):
   ```dockerfile
   RUN apt-get update && apt-get install -y software-properties-common \
       && add-apt-repository ppa:ondrej/php \
       && apt-get update \
       && apt-get install -y \
          php7.4-fpm php7.4-cli \
          php7.4-xml php7.4-mbstring php7.4-intl php7.4-pgsql
          # one line per extension package from Investigation #4, version number updated
   ```
3. Rebuild without cache:
   ```bash
   docker build --no-cache -t symfony5-upgrade-test .
   ```
4. Verify:
   ```bash
   docker run --rm symfony5-upgrade-test php -v
   ```
   Output matches target exactly? If not (multiple PHP versions installed, `$PATH` points at old
   one): add `RUN update-alternatives --set php /usr/bin/php<target-version>`, rebuild, re-check.
5. Re-run Investigation #4's third grep against the updated Dockerfile. Every match must reflect the
   new version — none may still reference the old one.
6. Cross-check extensions:
   ```bash
   grep '"ext-' composer.json
   ```
   Every `ext-<name>` printed → matching `php<version>-<name>` apt package must exist in Dockerfile.

### Phase 6 — Symfony-5 breaking changes

Full authoritative list: `https://github.com/symfony/symfony/blob/5.4/UPGRADE-5.0.md`. Below: the
two pitfalls flagged by Investigations #6/#7.

**Pitfall A — SwiftMailer** (only if `SWIFTMAILER=yes`):
1. `composer require symfony/mailer && composer remove symfony/swiftmailer-bundle`
2. `grep -rln 'Swift_Message\|swiftmailer' src/ config/` → for each file, replace:
   ```php
   // Before:
   $message = (new \Swift_Message('Subject'))
       ->setFrom('from@example.com')->setTo('to@example.com')->setBody('Body text');
   $this->mailer->send($message);

   // After:
   $email = (new \Symfony\Component\Mime\Email())
       ->from('from@example.com')->to('to@example.com')
       ->subject('Subject')->text('Body text');
   $this->mailer->send($email); // $this->mailer: Symfony\Component\Mailer\MailerInterface
   ```
3. Replace `swiftmailer.yaml` transport config with `MAILER_DSN` in `.env`/`.env.local` (e.g.
   `MAILER_DSN=smtp://user:pass@smtp.example.com:587`), matching the old transport type.
4. Send one real test email in a non-prod environment (see Verification).

**Pitfall B — Security authenticator system** (if #7 found no `enable_authenticator_manager`):
- Not mandatory here. Removed entirely in Symfony 6, so recommended now.
- Ask user: do now, or defer? Either way, record the decision in the final report.

**General removed-API sweep**: run the full suite (after Phase 7). Every fatal `Error` on an unknown
class/method → search `UPGRADE-5.0.md` for that exact name.

### Phase 7 — PHPUnit → Symfony-5-compatible version

1. `phpunit.xml.dist`: existing `SYMFONY_PHPUNIT_VERSION` entry → set `value="9.5"` (check
   `https://packagist.org/packages/phpunit/phpunit#9.5` for the current latest `9.x` patch first —
   don't hardcode blindly). No entry → add inside `<php>`:
   ```xml
   <php>
       <server name="SYMFONY_PHPUNIT_VERSION" value="9.5"/>
   </php>
   ```
2. Existing `SYMFONY_MAX_PHPUNIT_VERSION` entry → remove it. Re-add only if a specific dependency's
   incompatibility with PHPUnit 9 shows up after step 4.
3. ```bash
   rm -rf vendor/bin/.phpunit
   ```
4. Run `./bin/phpunit` or `./vendor/bin/simple-phpunit`. Two failure categories — fix separately:
   - PHPUnit method no longer exists (removed in 9.x) → test-code fix.
   - Application code failure → continue Phase 6's removed-API sweep.
5. Repeat until `0 failures, 0 errors`.

## Verification steps

| # | Check | Command / method |
|---|---|---|
| 1 | Suite passes in the rebuilt image | `docker run --rm symfony5-upgrade-test ./bin/phpunit` |
| 2 | Container/config sane | `php bin/console debug:container` and `debug:config` both exit `0` |
| 3 | Zero deprecations | Boot `APP_ENV=dev` in browser, web debug toolbar deprecation counter = `0` |
| 4 | Mailer works (if Pitfall A done) | Send one real test email in non-prod, confirm received |
| 5 | Auth works (if Pitfall B done) | Manually exercise every auth path (form login, API token, custom Guard) |
| 6 | Clean-cache rebuild | `docker build --no-cache -t symfony5-upgrade-final .`, re-run checks 1–3 against it |
| 7 | Sources agree | Re-run Investigation #4's 3 commands; `composer.json`, `composer.lock`, Dockerfile now consistent (or discrepancy was flagged on purpose) |

## Rollback / risk notes

- Phase 1 commit is separate from and before any Symfony-5 change → revert with plain `git revert`
  if needed, no version-constraint impact.
- Phase 2 (deprecation fixes) is its own commit(s) on the still-4.4 codebase, merged independently
  before Phase 3 → safe to keep even if everything after is rolled back.
- Phase 3 onward: dedicated branch. `composer.json`/`composer.lock` diff + Dockerfile diff are
  coupled — revert together. Never ship the Phase 5 PHP bump without the Phase 3 Symfony bump (old
  Symfony 4 code on untested new PHP, no passing test run backing it).
- Keep the pre-upgrade Docker image tag available until the new one has run clean through one full
  staging deployment cycle.
- Pitfall A/B deferred → record explicitly in PR/task description with the reason. Both get harder
  to defer once a future Symfony 6 upgrade is in scope.

## References

- [Symfony 5.4 Release](https://symfony.com/releases/5.4) — LTS status, PHP requirement (≥7.2.5)
- [Symfony | endoflife.date](https://endoflife.date/symfony) — support/security-EOL dates (re-check;
  Symfony has extended 5.4's security support via sponsorship before)
- [UPGRADE-5.0.md (5.4 branch)](https://github.com/symfony/symfony/blob/5.4/UPGRADE-5.0.md) —
  authoritative 4→5 breaking-change list
- [Upgrading a Major Version (Symfony 4.4 docs)](https://symfony.com/doc/4.4/setup/upgrade_major.html) — official process Phases 2–4 are based on
- [The PHPUnit Bridge (Symfony 5.x docs)](https://symfony.com/doc/5.x/components/phpunit_bridge.html) — `SYMFONY_PHPUNIT_VERSION` / `SYMFONY_MAX_PHPUNIT_VERSION` mechanics
- [The end of Swiftmailer (Symfony blog)](https://symfony.com/blog/the-end-of-swiftmailer) —
  SwiftMailer → Mailer migration rationale/timeline
- [Composer `why-not`/`prohibits`](https://getcomposer.org/doc/03-cli.md) — used mechanically in
  Phase 3 to find blocking packages
- [Composer config platform reference](https://getcomposer.org/doc/06-config.md) — confirms
  `platform-overrides` only reflects an explicit override, not real ambient PHP (why Investigation
  #4 treats the Dockerfile as ground truth)
- Ubuntu default PHP-per-release mapping (18.04→7.2, 20.04→7.4, 22.04→8.1, 24.04→8.3) + `ondrej/php`
  PPA — general community knowledge; re-verify if applying to an Ubuntu release not listed, mapping
  shifts with each new LTS.
