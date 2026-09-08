# Upgrade a Symfony 5 Project to Symfony 6 (PHPUnit + Ubuntu-based Docker image)

Purpose: upgrade a Symfony 5.x app — PHPUnit via `symfony/phpunit-bridge`, Docker image `FROM
ubuntu:<version>` with PHP installed via `apt` — to Symfony 6.4 (LTS), with the Docker image
rebuilt on a PHP version Symfony 6.4 supports (≥ 8.1.0). This guide is deliberately long: it
enumerates every breaking change and every deprecation from every official Symfony changelog
between the starting point and the target, not a curated highlight list. Skimming for "the
important parts" defeats the purpose — a change that looks irrelevant to most projects may be the
one thing this specific project actually hits.

**Rules for following this guide:**
1. Run every command exactly as written. Do not substitute a different command that seems similar.
2. Match command output against the table/list given for that command. Do not skip this step.
3. If output matches no row/case listed: STOP. Tell the user the exact command and exact output.
   Do not guess the closest match.
4. Every "if/then" below is a complete decision table. If a case isn't listed, that's rule 3, not a
   judgment call.
5. Do the Phases in order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8. Do not skip ahead unless a step explicitly
   says to.
6. For every entry in the "Complete breaking-changes list" section: run its Check command. No
   match/no output means this project isn't affected by that entry — move to the next one. A match
   means the Fix is required (unless the entry itself says otherwise, e.g. a narrow, unlikely-to-
   apply case it tells you to confirm manually).

Every Symfony 5.x project is already Flex-structured (`config/bundles.php`,
`config/packages/*.yaml`, `public/index.php`) — unlike the 4→5 guide, there is no pre-Flex
directory-layout phase here.

## When to use this guide

Use when ALL of these are true:
- [ ] `composer.json` requires `symfony/*` at `^5.x`
- [ ] Docker image: `FROM ubuntu:*` + PHP installed via `apt`/PPA (NOT `FROM php:*`)
- [ ] Tests run via `symfony/phpunit-bridge` (`bin/phpunit` or `vendor/bin/simple-phpunit`)

Do NOT use for:
- Symfony 4→5 upgrades — use [`upgrade-symfony-4-to-symfony-5.md`](upgrade-symfony-4-to-symfony-5.md) first
- Symfony 6→7 upgrades (different guide)
- `FROM php:8.x-fpm`-style images (Phase 6 doesn't apply — just edit the tag)
- Greenfield Symfony 6 projects (this is migration-only)

## Prerequisites — verify, do not assume

| # | Check | Command | Pass condition |
|---|---|---|---|
| P1 | Composer-managed | `test -f composer.json && echo yes \|\| echo no` | `yes` — if `no`, STOP, guide doesn't apply |
| P2 | On Symfony 5.4 | See Investigation #1 | Must show `v5.4.x` before Phase 1 |
| P3 | PHP ≥ 8.1.0 available | See Investigation #2 | Confirmed in Phase 6, not assumed from `composer.json` alone |
| P4 | Test suite exists | `test -f phpunit.xml.dist -o -f phpunit.xml && echo yes \|\| echo no` | `yes` — if `no`, STOP and tell user before proceeding (upgrade is riskier without tests) |

If P2 fails (project on 5.0–5.3): run `composer require "symfony/*:5.4.*"`, then
`composer update "symfony/*"`, then do Phase 1's deprecation fixes, then re-run Investigation #1
until it shows `5.4.x`. Only then continue.

No separate breaking-change table is needed to get from an earlier 5.x to 5.4: `UPGRADE-5.4.md`
(fetched and read in full to confirm this) contains *only* deprecation notices, zero removals —
Symfony guarantees no breaking changes within a major version. Every 5.4 deprecation either
reappears as a hard 6.0 removal (already covered in this guide's breaking-changes list below) or is
harmless to leave until then. Phase 1's "fix every deprecation notice, then re-run" loop is
sufficient and is the officially recommended process for this leg.

## Investigation checklist

Run every command below first. Keep the results — phases reference them by number.

### #1 — Current Symfony version

```bash
grep -A2 '"name": "symfony/framework-bundle"' composer.lock | grep '"version"'
```

| Output starts with | Meaning | Action |
|---|---|---|
| `v5.4.` | On 5.x LTS | Go to Phase 1 |
| `v5.0.`–`v5.3.` | Earlier 5.x | Do the P2 fix above first |
| `v6.` or higher | Already past Symfony 5 | STOP — guide doesn't apply |
| `v4.` or lower | Not yet on Symfony 5 | STOP — use `upgrade-symfony-4-to-symfony-5.md` first |
| (no output) | Package not locked under this name | Re-run with `"symfony/symfony"` instead of `"symfony/framework-bundle"`, apply same table |

### #2 — PHP version vs. Dockerfile

```bash
grep '"php"' composer.json
grep -A5 '"platform-overrides"' composer.lock
grep -n -E '^FROM|php[0-9]\.[0-9]|PHP_VERSION|ondrej' Dockerfile
```

| Source | What it tells you | Trust level |
|---|---|---|
| `composer.json` `require.php` | Declared minimum, e.g. `"php": "^7.2.5"` | Low — can be stale |
| `composer.lock` `platform-overrides` | Only non-empty if `config.platform.php` is explicitly set in `composer.json`. Empty = normal, tells you nothing. Non-empty = Composer is told to pretend this version is real | Untrustworthy — may mask real mismatch |
| Dockerfile | What actually runs in prod/CI | **Ground truth — use this** |

If the three disagree: Dockerfile wins. Report the discrepancy to the user.

Ubuntu release → default `apt` PHP version (no PPA), read off the `FROM` line match:

| `FROM` line contains | Ubuntu release | Default PHP | Meets Symfony 6.4's PHP ≥ 8.1.0 floor? |
|---|---|---|---|
| `ubuntu:18.04` / `ubuntu:bionic` | 18.04 bionic | 7.2 | No |
| `ubuntu:20.04` / `ubuntu:focal` | 20.04 focal | 7.4 | No |
| `ubuntu:22.04` / `ubuntu:jammy` | 22.04 jammy | 8.1 | Yes — exactly at the floor |
| `ubuntu:24.04` / `ubuntu:noble` | 24.04 noble | 8.3 | Yes |
| anything else / multiple `FROM` lines | — | — | STOP — tell user which `FROM` lines exist, ask which stage ships |

From the third grep's output, also record:
- Every `php<version>-<extension>` package name (e.g. `php7.4-fpm`, `php7.4-xml`,
  `php7.4-mbstring`) → Phase 6 needs version-bumped equivalents for each.
- Whether `ondrej` appears anywhere → PPA already configured; changes what Phase 6 adds.

### #3 — Direct dependency list

```bash
composer show -D
```

Output: one line per direct dependency, format `vendor/package   1.2.3   Description`. Save it.
Phase 5 uses this to identify current versions when Composer reports a conflict — no manual
Packagist research needed up front.

### #4 — Security: authenticator-based system

```bash
FILE=config/packages/security.yaml
grep -n 'enable_authenticator_manager' "$FILE"
grep -rln 'implements.*AuthenticatorInterface\|extends AbstractGuardAuthenticator' src/
echo "SECURITY_FILE=$FILE"
```

| First grep matched | Second grep found files | Meaning | Action |
|---|---|---|---|
| Yes | No | Already on new authenticator system | Phase 2 is a no-op — just remove the now-unneeded config key in Phase 2 step 1 |
| Yes | Yes | Config opted in, but Guard authenticators still exist | Phase 2 migrates each listed file |
| No | (either) | Still on legacy security system | Phase 2 is mandatory — Symfony 6.0 removes the Guard component and the legacy system entirely; the app will not boot on 6.0 until this is done |

### #5 — Doctrine annotations vs. PHP attributes

```bash
grep -rl '@ORM\\\|@Route(\|@Assert\\' src/ 2>/dev/null | wc -l
grep -q '"doctrine/annotations"' composer.json && echo "ANNOTATIONS_PKG=yes" || echo "ANNOTATIONS_PKG=no"
```

| First command | `ANNOTATIONS_PKG` | Meaning | Action |
|---|---|---|---|
| `0` | either | No annotation usage found | No action needed regardless of package presence |
| `> 0` | `no` | Code has stray `@ORM`/`@Route`/`@Assert`-style comments but the reader package isn't installed | These are dead docblocks, not live config — confirm with the user before touching, don't assume |
| `> 0` | `yes` | Entities/routes/validation use live Doctrine annotations | **This will produce an active deprecation notice on the 6.4 target** (`framework.annotations`, `routing`, `serializer`, and `validator` all deprecate their annotation integrations by 6.4 — see the 6.4 table below) — Verification #3's deprecation check will show it. Either migrate every annotated class to PHP 8 attributes now (`#[ORM\...]`, `#[Route(...)]`, `#[Assert\...]`), or explicitly record these as known, accepted deprecations in the final report rather than treating them as a mystery failure |

Do not mix annotations and attributes on the same class — if migrating, migrate one whole class at
a time.

### #6 — PHPUnit setup

```bash
grep -n 'SYMFONY_PHPUNIT_VERSION\|SYMFONY_MAX_PHPUNIT_VERSION' phpunit.xml.dist
```

- Output like `<server name="SYMFONY_PHPUNIT_VERSION" value="8.5"/>` or `9.x` → Phase 8 bumps it.
- No output → nothing pinned; Phase 8 adds an explicit pin.

### #7 — SwiftMailer (leftover from a deferred 4→5 migration)

```bash
grep -q '"symfony/swiftmailer-bundle"' composer.json && echo "SWIFTMAILER=yes" || echo "SWIFTMAILER=no"
```

`SWIFTMAILER=yes` → STOP before Phase 1. This package is incompatible with Symfony 6 entirely (not
just deprecated). Go run `upgrade-symfony-4-to-symfony-5.md` Phase 6 Pitfall A first, then restart
this guide from Investigation #1.

### #8 — SensioFrameworkExtraBundle

```bash
grep -q '"sensio/framework-extra-bundle"' composer.json && echo "SENSIO=yes" || echo "SENSIO=no"
```

`SENSIO=yes` → its `@Route`/`@ParamConverter`/`@Cache`/`@Security` annotations still work through
6.1 but the bundle itself is superseded by FrameworkBundle's native equivalents in 6.2+. Note in
final report as recommended follow-up (native `#[Route]` attribute, `#[MapEntity]`, etc.) — not a
blocker for reaching 6.4, since the bundle still functions there.

### #9 — Custom test classes extending Symfony's `*TestCase` transport/provider test bases

```bash
grep -rln 'extends TransportTestCase\|extends TransportFactoryTestCase\|extends ProviderTestCase\|extends ProviderFactoryTestCase' tests/
```

Any match → the breaking-changes list's Notifier/Mailer/Translation entries (5.4 and 6.2/6.3
tables below) apply: several of these base classes' data-provider and factory methods became
`static`, and for `TransportTestCase` specifically this is an enforced `[BC BREAK]` by 6.2, not a
deferrable deprecation.

### #10 — Other deprecated/removed dependencies

No command — cross-reference step. Done in Phase 5 step 3, against the #3 dependency list and
this guide's breaking-changes list below (which was itself built directly from
`UPGRADE-6.0.md` through `UPGRADE-6.4.md`).

### #11 — CI pipeline

```bash
ls -a .github/workflows .gitlab-ci.yml 2>/dev/null
```

Record which files exist — they likely build/pull the same Docker image Phase 6 changes.

## Complete breaking-changes list, version by version

This section is the direct output of reading `UPGRADE-6.0.md`, `UPGRADE-6.1.md`,
`UPGRADE-6.2.md`, `UPGRADE-6.3.md`, and `UPGRADE-6.4.md` in full (see References) — every entry in
each file, not a curated subset. Phase 7 of the implementation plan is simply "work through every
row below." For every entry: run the Check; no output means this project isn't affected, skip to
the next entry; any output means apply the Fix.

**Important distinction between the 6.0 table and the 6.1–6.4 tables:** 6.0 is a major version —
everything in its table is an unconditional, already-happened removal. Landing this guide's target
of 6.4 means the app must satisfy every 6.0 entry. 6.1 through 6.4 are minor versions within the
same major, where Symfony's BC promise mostly holds — so most of their entries are *deprecations*
aimed at a future 7.0 upgrade, not required to reach 6.4. **The exception, and the reason this
guide reads every one of these files individually instead of assuming "later minors are just
deprecations":** each of 6.2, 6.3, and 6.4 contains a handful of entries explicitly marked
`[BC BREAK]` in the source file. Those are real, immediate breaks that apply the moment the app
runs on that version — they are marked **mandatory now** below. Every other 6.1–6.4 entry is marked
**optional (7.0 prep)** — safe to defer past this guide's 6.4 target, and listed here so the
project doesn't have to re-derive them from scratch on a future 6→7 upgrade.

### Symfony 6.0 (mandatory — hard removals, no deprecation period)

**Asset**
- **`RemoteJsonManifestVersionStrategy` removed** → use `JsonManifestVersionStrategy`.
  Check: `grep -rn 'RemoteJsonManifestVersionStrategy' src/ config/`. Fix: replace the class
  reference.

**DoctrineBridge**
- **`UserLoaderInterface::loadUserByUsername()` removed** → `loadUserByIdentifier()`.
  Check: `grep -rln 'loadUserByUsername' src/`. Fix: rename the method in every implementing class.
- **`AbstractDoctrineExtension::getMappingDriverBundleConfigDefaults()` gains a required
  `$bundleDir` argument.** Check: `grep -rln 'getMappingDriverBundleConfigDefaults' src/`. Fix:
  only relevant if a bundle extends `AbstractDoctrineExtension` directly; add the argument.
- **`AbstractDoctrineExtension::getMappingResourceConfigDirectory()` gains a required `$bundleDir`
  argument.** Check: `grep -rln 'getMappingResourceConfigDirectory' src/`. Fix: same as above.

**Cache**
- **`DoctrineProvider`/`DoctrineAdapter` removed** (moved to the `doctrine/cache` package).
  Check: `grep -rln 'Cache\\\\DoctrineProvider\|Cache\\\\DoctrineAdapter' src/`. Fix: use the
  classes now shipped in `doctrine/cache`, or migrate to a native Symfony Cache adapter.
- **`PdoAdapter` no longer accepts a `Doctrine\DBAL\Connection` or DBAL URL.**
  Check: `grep -rln 'new PdoAdapter' src/`. Fix: if constructed with a DBAL connection/URL, switch
  to `DoctrineDbalAdapter`.

**Config**
- **`NodeDefinition::setDeprecated()` / `BaseNode::setDeprecated()` signature changed** to
  `setDeprecation(string $package, string $version, string $message)`; passing `null` to
  un-deprecate no longer supported; `BaseNode::getDeprecationMessage()` removed → `getDeprecation()`.
  Check: `grep -rln 'setDeprecated(\|getDeprecationMessage(' src/`. Fix: only relevant if the
  project defines its own Config tree builders (bundle/library authors); update call signatures.

**Console**
- **`Command::setHidden()`'s `$hidden` parameter now defaults to `true`.**
  Check: `grep -rn '\->setHidden(' src/`. Fix: review each zero-argument call — it now hides the
  command; pass `false` explicitly if that's not intended.
- **`Helper::strlen()` removed** → `Helper::width()`; **`Helper::strlenWithoutDecoration()`
  removed** → `Helper::removeDecoration()`.
  Check: `grep -rln 'Helper::strlen\|->strlenWithoutDecoration' src/`. Fix: rename calls.
- **`HelperSet::setCommand()`/`getCommand()` removed, no replacement.**
  Check: `grep -rln 'HelperSet.*->setCommand(\|HelperSet.*->getCommand(' src/`. Fix: remove usage;
  store the command reference elsewhere if still needed.

**DependencyInjection**
- **`Definition::setDeprecated()` / `Alias::setDeprecated()` / `DeprecateTrait::deprecate()`
  signatures changed** to the 3-argument `(package, version, message)` form.
  Check: `grep -rln '\->setDeprecated(\|\->deprecate(' src/`. Fix: only relevant to custom compiler
  passes/bundle extensions defining services in PHP; update call signatures.
- **`Psr\Container\ContainerInterface` / `Symfony\Component\DependencyInjection\ContainerInterface`
  aliases of `service_container` removed.**
  Check: `grep -rn 'ContainerInterface \$container' src/`. Fix: configure the alias explicitly in
  `services.yaml` if still relied upon, or inject specific services instead.
- **`Definition::getDeprecationMessage()` / `Alias::getDeprecationMessage()` removed** →
  `getDeprecation()`. Check: `grep -rln 'getDeprecationMessage(' src/`. Fix: rename calls.
- **PHP-DSL `inline()` removed** → `inline_service()`; **`ref()` removed** → `service()`.
  Check: `grep -rln '\binline(\|\bref(' config/*.php`. Fix: rename DSL function calls in PHP-format
  service config.
- **`Definition::setPrivate()` / `Alias::setPrivate()` removed** → use `setPublic()`.
  Check: `grep -rln '\->setPrivate(' src/`. Fix: replace with `->setPublic(false)` /
  `->setPublic(true)` as appropriate.

**DomCrawler**
- **`parents()` removed** → `ancestors()`.
  Check: `grep -rln '\->parents(' src/ tests/`. Fix: rename calls.

**Dotenv**
- **`Dotenv` constructor's `$usePutenv` argument removed** → use `Dotenv::usePutenv()`.
  Check: `grep -rln 'new Dotenv(' src/ config/bootstrap.php`. Fix: if a boolean was passed to the
  constructor, call `->usePutenv()` fluently instead.

**EventDispatcher**
- **`LegacyEventDispatcherProxy` removed.**
  Check: `grep -rln 'LegacyEventDispatcherProxy' src/`. Fix: use the event dispatcher directly.

**Finder**
- **`Comparator::setTarget()`/`setOperator()` removed; `$target` constructor argument now
  mandatory.** Check: `grep -rln 'new Comparator(\|->setTarget(\|->setOperator(' src/`. Fix: pass
  `$target` to the constructor; remove setter calls.

**Form**
- **`FormErrorIterator::children()` now throws if the current element isn't iterable.**
  Check: `grep -rln '\->children()' src/`. Fix: ensure the element is iterable before calling.
- **`PercentType`/`PercentToLocalizedStringTransformer` default `rounding_mode` changed to
  `\NumberFormatter::ROUND_HALFUP`.** Check: `grep -rln 'PercentType' src/`. Fix: if the app relies
  on the old default rounding, set `rounding_mode` explicitly.
- **`NumberToLocalizedStringTransformer::ROUND_*` constants removed** → use
  `\NumberFormatter::ROUND_*`. Check: `grep -rln 'NumberToLocalizedStringTransformer::ROUND_' src/`.
  Fix: replace constant references.
- **`Form\Extension\Validator\Util\ServerParams` removed** → use `Form\Util\ServerParams`.
  Check: `grep -rln 'Extension\\\\Validator\\\\Util\\\\ServerParams' src/`. Fix: update the import.
- **`PropertyPathMapper` removed** → `DataMapper` + `PropertyPathAccessor`.
  Check: `grep -rln 'PropertyPathMapper' src/`. Fix: replace with `DataMapper`/`PropertyPathAccessor`.
- **`DataMapper`/`CheckboxListMapper`/`RadioListMapper`'s `mapDataToForms()`/`mapFormsToData()`
  parameter type changed `iterable` → `\Traversable`.**
  Check: `grep -rln 'implements DataMapperInterface' src/`. Fix: only relevant to a custom data
  mapper; ensure the argument passed is a `\Traversable`, not a plain array (wrap with
  `ArrayIterator` if needed).
- Additive, no action required: `ChoiceListFactoryInterface::createListFromChoices()`/
  `createListFromLoader()` gain a `callable|null $filter` argument;
  `FormConfigInterface::getIsEmptyCallback()` and `FormConfigBuilderInterface::setIsEmptyCallback()`
  added (implement only if the project has a custom `ChoiceListFactoryInterface` or
  `FormConfigInterface`).

**FrameworkBundle**
- **`framework.translator.enabled_locales` removed** → `framework.enabled_locales` (top-level, not
  under `translator:`). Check: `grep -rn 'enabled_locales' config/packages/*.yaml`. Fix: move the
  key to the top-level `framework.enabled_locales`.
- **`session.storage` alias / `session.storage.*` services removed** →
  `session.storage.factory` / `session.storage.factory.*`.
  Check: `grep -rln "session.storage\b" src/ config/`. Fix: update service references.
- **`framework.session.storage_id` removed** → `framework.session.storage_factory_id`.
  Check: `grep -n 'storage_id' config/packages/*.yaml`. Fix: rename the config key.
- **`session` service / `SessionInterface` alias removed** → `Request::getSession()` /
  `RequestStack::getSession()`. Check: `grep -rln "container->get('session')\|SessionInterface \$session" src/`.
  Fix: inject `RequestStack` and call `getSession()`, or use the current `Request`'s `getSession()`.
- **`MicroKernelTrait::configureRoutes()` now always called with a `RoutingConfigurator`.**
  Check: `grep -rln 'configureRoutes' src/Kernel.php`. Fix: update the method to the
  `RoutingConfigurator` API.
- **`framework.router.utf8` now defaults to `true`.**
  Check: `grep -n 'utf8' config/packages/*.yaml`. Fix: if the app relied on non-UTF-8 routing, set
  `framework.router.utf8: false` explicitly.
- **`session.attribute_bag` / `session.flash_bag` services removed.**
  Check: `grep -rln 'session.attribute_bag\|session.flash_bag' src/ config/`. Fix: use
  `Session::getBag()` / `Request::getSession()->getFlashBag()` instead of the standalone services.
- **`form.factory`, `form.type.file`, `profiler`, `translator`, `security.csrf.token_manager`,
  `serializer`, `cache_clearer`, `filesystem`, `validator` services are now private.**
  Check: `grep -rln "container->get('\(form.factory\|form.type.file\|profiler\|translator\|security.csrf.token_manager\|serializer\|cache_clearer\|filesystem\|validator\)')" src/`.
  Fix: autowire the matching interface instead of fetching by service id.
- **`lock.RESOURCE_NAME[.store]` services / `lock`, `LockInterface`, `lock.store`,
  `PersistingStoreInterface` aliases removed** → `lock.RESOURCE_NAME.factory`, `lock.factory`,
  `LockFactory`. Check: `grep -rln "'lock'\|LockInterface \$lock\|'lock.store'\|PersistingStoreInterface" src/`.
  Fix: inject `LockFactory` (or the named `.factory` service) and call `createLock()`.
- **`KernelTestCase::$container` removed** → `getContainer()`.
  Check: `grep -rln '\$this->container\b' tests/`. Fix: replace with `self::getContainer()`.
- **Registered workflow services are now private.**
  Check: `grep -rln "container->get('workflow\." src/`. Fix: autowire
  `WorkflowInterface $<name>Workflow` or the `Registry` instead.
- **`translation:update --xliff-version` / `--output-format` options removed.**
  Check: `grep -rn 'xliff-version' .github/ .gitlab-ci.yml bin/ 2>/dev/null`. Fix: use
  `--output-format=xlf20` (or the matching target format).
- **`AdapterInterface` autowiring alias removed** → `CacheItemPoolInterface`.
  Check: `grep -rln 'AdapterInterface \$' src/`. Fix: type-hint `CacheItemPoolInterface` instead.
- **`AbstractController::get()`/`has()`/`getDoctrine()`/`dispatchMessage()` removed.**
  Check: `grep -rln '\->get(.*)\|\->has(.*)\|\->getDoctrine()\|\->dispatchMessage(' src/Controller/`.
  Fix: inject the needed service (`ManagerRegistry`, `MessageBusInterface`, etc.) via constructor or
  controller-action argument autowiring.
- **`cache.adapter.doctrine` service deprecated** (the Doctrine Cache library itself is deprecated).
  Check: `grep -rn 'cache.adapter.doctrine\b' config/packages/*.yaml`. Fix: switch to a native
  Symfony Cache adapter or Doctrine Cache's own PSR-6 adapters.
- **`framework.messenger.reset_on_message` now defaults to `true`.**
  Check: `grep -n 'reset_on_message' config/packages/messenger.yaml`. Fix: if the app relied on
  services *not* resetting after each message, set it to `false` explicitly.
- **`framework.cache` with `cache.adapter.pdo` + a Doctrine DBAL connection no longer supported** →
  `cache.adapter.doctrine_dbal`. Check: `grep -n 'cache.adapter.pdo' config/packages/cache.yaml`.
  Fix: switch the adapter name if a DBAL connection/URL is configured.

**HttpFoundation**
- **`NamespacedAttributeBag` removed.** Check: `grep -rln 'NamespacedAttributeBag' src/`. Fix: use
  `AttributeBag` (flat) instead.
- **`Response::create()`, `JsonResponse::create()`, `RedirectResponse::create()`,
  `StreamedResponse::create()`, `BinaryFileResponse::create()` removed.**
  Check: `grep -rln '\(Response\|JsonResponse\|RedirectResponse\|StreamedResponse\|BinaryFileResponse\)::create(' src/`.
  Fix: replace with `new <Class>(...)`.
- **`ParameterBag::filter()` / `InputBag::filter()` with `FILTER_CALLBACK` now require a `Closure`,
  else throw `\InvalidArgumentException`.** Check: `grep -rln 'FILTER_CALLBACK' src/`. Fix: wrap the
  filter callback in a `Closure`.
- **`Request::HEADER_X_FORWARDED_ALL` removed.**
  Check: `grep -rln 'HEADER_X_FORWARDED_ALL' src/ config/`. Fix: use the explicit bitmask
  combination, or `HEADER_X_FORWARDED_AWS_ELB` / `HEADER_X_FORWARDED_TRAEFIK`.
- **`RequestStack::getMasterRequest()` renamed `getMainRequest()`.**
  Check: `grep -rln 'getMasterRequest(' src/`. Fix: rename calls.
- **`InputBag::filter()` on an array without `FILTER_REQUIRE_ARRAY`/`FILTER_FORCE_ARRAY` now throws
  `BadRequestException`.** Check: `grep -rln '\$request->query->filter(\|\$request->request->filter(' src/`.
  Fix: pass the appropriate flag when filtering an array value.
- **`InputBag::get()` on a non-scalar value now throws `BadRequestException`; non-scalar default
  argument throws `\InvalidArgumentException`.**
  Check: `grep -rln '\$request->query->get(\|\$request->request->get(' src/`. Fix: use
  `InputBag::all()` for array values; pass only scalar defaults.
- **`InputBag::set()` with a non-scalar, non-array value now throws `\InvalidArgumentException`.**
  Check: `grep -rln '\$request->query->set(\|\$request->request->set(' src/`. Fix: ensure the value
  is scalar or array.
- **`IpUtils::checkIp()/checkIp4()/checkIp6()` no longer accept `null` `$requestIp`.**
  Check: `grep -rln 'IpUtils::checkIp' src/`. Fix: pass a real IP string.
- **`upload_progress.*` / `url_rewriter.tags` session options removed.**
  Check: `grep -n 'upload_progress\|url_rewriter' config/packages/framework.yaml`. Fix: remove these
  keys — no replacement (the underlying PHP features are gone).

**HttpKernel**
- **`ArgumentInterface` removed.** Check: `grep -rln 'ArgumentInterface' src/`. Fix: remove usage.
- **`ArgumentMetadata::getAttribute()` removed** → `getAttributes()`.
  Check: `grep -rln '\->getAttribute(' src/`. Fix: rename to `getAttributes()` (returns an array).
- **`WarmableInterface::warmUp()` now returns a list of classes/files to preload (PHP 7.4+).**
  Check: `grep -rln 'implements WarmableInterface' src/`. Fix: update custom cache warmers to
  `return []` (or the list to preload) instead of `void`.
- **`service:action` controller syntax removed** → `serviceOrFqcn::method`.
  Check: `grep -rn "controller: *['\"].*:.*['\"]" config/routes.yaml config/routes/*.yaml`. Fix:
  rewrite controller references to `Service::method` or `Fqcn::method`.
- **Returning a `ContainerBuilder` from `KernelInterface::registerContainerConfiguration()` no
  longer supported.** Check: `grep -rln 'registerContainerConfiguration' src/Kernel.php`. Fix:
  ensure the method matches the current signature/behavior.
- **`HttpKernelInterface::MASTER_REQUEST` renamed `MAIN_REQUEST`; `KernelEvent::isMasterRequest()`
  renamed `isMainRequest()`.** Check: `grep -rln 'MASTER_REQUEST\|isMasterRequest(' src/`. Fix:
  rename references.

**Inflector**
- **Component removed entirely** → use `EnglishInflector` from the String component.
  Check: `grep -q '"symfony/inflector"' composer.json && echo yes || echo no`. Fix: if `yes`, remove
  the dependency and switch to `Symfony\Component\String\Inflector\EnglishInflector`.

**Ldap**
- **`LdapAuthenticator::createAuthenticatedToken()` removed** → `createToken()`.
  Check: `grep -rln 'createAuthenticatedToken' src/`. Fix: rename method/calls.

**Lock**
- **`NotSupportedException` removed** (no longer thrown).
  Check: `grep -rln 'Lock\\\\Exception\\\\NotSupportedException' src/`. Fix: remove any catch block
  for it — dead code.
- **`RetryTillSaveStore` removed** (logic folded into `Lock`).
  Check: `grep -rln 'RetryTillSaveStore' src/`. Fix: remove usage — retry is automatic now.
- **`PdoStore` / `PostgreSqlStore` with a `Doctrine\DBAL\Connection` or DBAL URL no longer
  supported.** Check: `grep -rln 'new PdoStore(\|new PostgreSqlStore(' src/`. Fix: switch to
  `DoctrineDbalStore` / `DoctrineDbalPostgreSqlStore` if constructed with a DBAL connection/URL.

**Mailer**
- **`SesApiTransport` removed** → `SesApiAsyncAwsTransport`; **`SesHttpTransport` removed** →
  `SesHttpAsyncAwsTransport`. Check: `grep -rln 'SesApiTransport\|SesHttpTransport' src/ config/`.
  Fix: rename to the AsyncAws-based classes.

**Messenger**
- **AmqpExt / Doctrine / RedisExt transports removed from core**, moved to standalone packages.
  Check: `grep -n 'amqp://\|redis://\|doctrine://' config/packages/messenger.yaml`. Fix:
  `composer require symfony/amqp-messenger` and/or `symfony/redis-messenger` and/or
  `symfony/doctrine-messenger` for each transport DSN scheme in use (also Phase 4 step 7).
- **Invalid Redis/AMQP connection options now throw.**
  Check: `grep -n 'amqp://\|redis://' config/packages/messenger.yaml`. Fix: review the DSN's
  query-string options against the current transport docs.
- **`RetryStrategyInterface::isRetryable()`/`getWaitingTime()` gain a
  `\Throwable $throwable = null` parameter.** Check: `grep -rln 'implements RetryStrategyInterface' src/`.
  Fix: only relevant to a custom retry strategy; add the parameter.
- **AMQP `prefetch_count` parameter removed.** Check: `grep -n 'prefetch_count' config/packages/messenger.yaml`.
  Fix: remove the option — no longer configurable.
- **Redis transport TLS option changed:** `?tls=1` removed → use the `rediss://` scheme.
  Check: `grep -n 'redis://.*tls=1' config/packages/messenger.yaml`. Fix: switch the DSN scheme to
  `rediss://`.
- **Redis transport `delete_after_ack` now defaults to `true`.**
  Check: `grep -n 'delete_after_ack' config/packages/messenger.yaml`. Fix: set explicitly to
  `false` if messages must remain in the stream after ack.

**Mime**
- **`Address::fromString()` removed** → `Address::create()`.
  Check: `grep -rln 'Address::fromString(' src/`. Fix: rename calls.

**Monolog**
- **`NotFoundActivationStrategy`/`HttpCodeActivationStrategy` constructor's `$actionLevel` replaced
  by `$inner` (an `ActivationStrategyInterface` to decorate); both classes now `final`.**
  Check: `grep -rln 'NotFoundActivationStrategy\|HttpCodeActivationStrategy' src/ config/`. Fix:
  update constructor args to decorate an inner strategy; remove any subclassing.
- **`ResetLoggersWorkerSubscriber` removed** → use the `reset_on_message` messenger option.
  Check: `grep -rln 'ResetLoggersWorkerSubscriber' src/ config/`. Fix: remove; rely on
  `reset_on_message` instead.

**Notifier**
- **`SlackOptions::channel()` removed** → `SlackOptions::recipient()`.
  Check: `grep -rln 'SlackOptions.*->channel(' src/`. Fix: rename calls.

**OptionsResolver**
- **`OptionsResolver::setDeprecated()` signature changed** to
  `(string $option, string $package, string $version, $message)`.
  Check: `grep -rln '\->setDeprecated(' src/`. Fix: only relevant to custom options-resolver usage
  (bundle/library config); update the call signature.
- **`OptionsResolverIntrospector::getDeprecationMessage()` removed** → `getDeprecation()`.
  Check: `grep -rln 'getDeprecationMessage(' src/`. Fix: rename calls.

**PhpUnitBridge**
- **`@expectedDeprecation` annotation removed** → `ExpectDeprecationTrait::expectDeprecation()`.
  Check: `grep -rln '@expectedDeprecation' tests/`. Fix: use the trait's method instead.
- **`SetUpTearDownTrait` removed** → use `setUp()`/`tearDown()` with `void` return type directly.
  Check: `grep -rln 'SetUpTearDownTrait' tests/`. Fix: remove the trait; declare `setUp(): void` /
  `tearDown(): void` directly.

**PropertyAccess**
- **`PropertyAccessor::__construct()` no longer accepts booleans as its 1st/2nd argument** → bitwise
  flags. Check: `grep -rln 'new PropertyAccessor(' src/`. Fix: pass a combination of the
  `PropertyAccessor::*` bitwise constants instead of booleans.

**PropertyInfo**
- **`Type::getCollectionKeyType()`/`getCollectionValueType()` removed** →
  `getCollectionKeyTypes()`/`getCollectionValueTypes()` (plural, now return arrays).
  Check: `grep -rln 'getCollectionKeyType(\|getCollectionValueType(' src/`. Fix: rename to plural.
- **`ReflectionExtractor`'s `enable_magic_call_extraction` context option removed** →
  `enable_magic_methods_extraction`. Check: `grep -rln 'enable_magic_call_extraction' src/`. Fix:
  rename the context key.

**Routing**
- **`RouteCollectionBuilder` removed.** Check: `grep -rln 'RouteCollectionBuilder' src/`. Fix:
  build routes via `RouteCollection` directly, or plain YAML/attribute routes.
- **`RouteCompiler::REGEX_DELIMITER` constant removed.**
  Check: `grep -rln 'RouteCompiler::REGEX_DELIMITER' src/`. Fix: remove usage — was internal.
- **`Route` annotation class constructor's `$data` parameter removed.**
  Check: `grep -rln 'new Route(' src/`. Fix: rare — only affects code instantiating the annotation
  class programmatically, not the `@Route`/`#[Route]` usage itself.
- Additive, no action required: `RouteCollection::add()` gains a `$priority` argument.

**Security** (the largest single component — cross-reference with Phase 2's authenticator
migration, which implements the fixes for several of these)
- **`AuthenticationEvents::AUTHENTICATION_FAILURE` removed** → listen for `LoginFailureEvent`.
  Check: `grep -rln 'AUTHENTICATION_FAILURE' src/`. Fix: switch the listened event.
- **`ChannelListener` constructor's `$authenticationEntryPoint` argument removed.**
  Check: `grep -rln 'new ChannelListener(' src/`. Fix: rare — only if manually constructing it.
- **`RetryAuthenticationEntryPoint` removed** (inlined into `ChannelListener`).
  Check: `grep -rln 'RetryAuthenticationEntryPoint' src/`. Fix: remove usage.
- **`FormAuthenticationEntryPoint`/`BasicAuthenticationEntryPoint` removed** →
  `FormLoginAuthenticator`/`HttpBasicAuthenticator`.
  Check: `grep -rln 'FormAuthenticationEntryPoint\|BasicAuthenticationEntryPoint' src/`. Fix: use
  the authenticator classes instead.
- **`AbstractRememberMeServices`, `PersistentTokenBasedRememberMeServices`,
  `RememberMeServicesInterface`, `TokenBasedRememberMeServices` removed** → remember-me handler
  alternatives. Check: `grep -rln 'RememberMeServices' src/`. Fix: migrate to a
  `RememberMeHandlerInterface` implementation.
- **`AnonymousToken` removed.** Check: `grep -rln 'AnonymousToken' src/`. Fix: anonymous requests
  are simply unauthenticated tokens now (`getUser()` returns `null`); remove usage.
- **`Token::getCredentials()` removed.** Check: `grep -rln '\->getCredentials()' src/`. Fix: tokens
  no longer carry credentials post-authentication; remove usage.
- **`Token::getUser()` return type restricted to `UserInterface`** (no more `string|\Stringable`).
  Check: `grep -rln '\->getUser()' src/`. Fix: review any code assuming a string return.
- **`AuthenticatedVoter::IS_AUTHENTICATED_ANONYMOUSLY`/`IS_ANONYMOUS` removed** → `PUBLIC_ACCESS`.
  Check: `grep -rn 'IS_AUTHENTICATED_ANONYMOUSLY\|IS_ANONYMOUS' config/packages/security.yaml src/ templates/`.
  Fix: replace with `PUBLIC_ACCESS`, e.g.
  `- { path: ^/login, roles: PUBLIC_ACCESS }`.
- **`AuthenticationTrustResolverInterface::isAnonymous()` / `is_anonymous()` expression removed.**
  Check: `grep -rln 'isAnonymous(\|is_anonymous(' src/ templates/`. Fix: use `isFullFledged()` or
  `isAuthenticated()`.
- **`AuthorizationChecker`'s 4th/5th constructor argument removed; `AccessListener`'s 5th argument
  removed.** Check: `grep -rln 'new AuthorizationChecker(\|new AccessListener(' src/`. Fix: rare —
  only if manually constructing these classes.
- **`Security\Core\User\User` class removed** → `InMemoryUser` or a custom implementation.
  Check: `grep -rln 'Security\\\\Core\\\\User\\\\User\b' src/`. Fix: switch to `InMemoryUser`;
  re-implement `isAccountNonLocked()`/`isAccountNonExpired()`/`isCredentialsNonExpired()` in a
  custom class if relied upon (not part of `InMemoryUser`'s API).
- **`UserChecker` class removed** → `InMemoryUserChecker` or a custom implementation.
  Check: `grep -rln 'Security\\\\Core\\\\User\\\\UserChecker\b' src/`. Fix: switch accordingly.
- **`UserInterface::getPassword()` removed from the interface.**
  Check: `grep -rln 'implements UserInterface' src/`. Fix: if `getPassword()` returns non-null, also
  implement `PasswordAuthenticatedUserInterface`.
- **`UserInterface::getSalt()` removed from the interface.**
  Check: same file(s) as above. Fix: if `getSalt()` returns non-null, also implement
  `LegacyPasswordAuthenticatedUserInterface`.
- **`UserInterface::getUsername()` removed** → `getUserIdentifier()`; same rename for
  `TokenInterface::getUsername()`, `UserProviderInterface::loadUserByUsername()` →
  `loadUserByIdentifier()`, `UsernameNotFoundException` → `UserNotFoundException`,
  `PersistentTokenInterface::getUsername()` → `getUserIdentifier()`.
  Check: `grep -rln 'getUsername()\|loadUserByUsername(\|UsernameNotFoundException' src/`. Fix:
  rename throughout (this is Phase 2 step 3–4).
- **`PasswordUpgraderInterface::upgradePassword()` / `UserPasswordHasherInterface` methods now
  throw `\TypeError`** if the user doesn't implement `PasswordAuthenticatedUserInterface`.
  Check: `grep -rln 'implements PasswordUpgraderInterface\|UserPasswordHasherInterface' src/`. Fix:
  ensure `User` implements `PasswordAuthenticatedUserInterface` (should follow from the fix above).
- **All classes in `Core\Encoder\*` removed** → `PasswordHasher` component.
  Check: `grep -rln 'Security\\\\Core\\\\Encoder' src/`. Fix: switch to the equivalent class in
  `symfony/password-hasher`.
- **`SessionTokenStorage` no longer accepts `SessionInterface $session`** → inject `RequestStack`;
  **`UsageTrackingTokenStorage` no longer accepts `session` from its ServiceLocator** → needs
  `request_stack`; **`SessionTokenStorage` now throws `SessionNotFoundException` outside a request
  context.** Check: `grep -rln 'new SessionTokenStorage(' src/`. Fix: rare — internal services,
  only relevant with manual construction or CLI-context calls.
- **`ROLE_PREVIOUS_ADMIN` removed** → `IS_IMPERSONATOR` attribute.
  Check: `grep -rn 'ROLE_PREVIOUS_ADMIN' src/ templates/ config/`. Fix: replace with
  `is_granted('IS_IMPERSONATOR')`.
- **`LogoutSuccessHandlerInterface`/`LogoutHandlerInterface` removed** → listen to `LogoutEvent`;
  **`DefaultLogoutSuccessHandler` removed** → `DefaultLogoutListener`.
  Check: `grep -rln 'implements LogoutSuccessHandlerInterface\|implements LogoutHandlerInterface\|DefaultLogoutSuccessHandler' src/`.
  Fix: convert each implementation to a `LogoutEvent` listener.
- **`RememberMeServicesInterface` gains a `logout()` method.**
  Check: `grep -rln 'implements RememberMeServicesInterface' src/`. Fix: only relevant to a custom
  remember-me services implementation (should already be migrated per the row above).
- **`setProviderKey()`/`getProviderKey()` removed** from `PreAuthenticatedToken`, `RememberMeToken`,
  `SwitchUserToken`, `UsernamePasswordToken`, `DefaultAuthenticationSuccessHandler` →
  `setFirewallName()`/`getFirewallName()`; `AbstractRememberMeServices::$providerKey` →
  `$firewallName`. Check: `grep -rln 'ProviderKey\|getProviderKey(\|setProviderKey(' src/`. Fix:
  rename throughout.
- **`AccessDecisionManager` now throws if a voter returns an invalid decision.**
  Check: — (behavioral). Fix: only surfaces with a custom `VoterInterface` returning something
  other than the defined constants; review any custom voter.
- **`AuthenticationManagerInterface`, `AuthenticationProviderManager`,
  `AnonymousAuthenticationProvider`, `AuthenticationProviderInterface`, `DaoAuthenticationProvider`,
  `LdapBindAuthenticationProvider`, `PreAuthenticatedAuthenticationProvider`,
  `RememberMeAuthenticationProvider`, `UserAuthenticationProvider`, `AuthenticationFailureEvent`
  removed from security-core.** Check: `grep -rln 'AuthenticationProvider\|AuthenticationManagerInterface\|AuthenticationFailureEvent' src/`.
  Fix: migrate to the new authenticator system (Phase 2).
- **`AbstractAuthenticationListener`, `AbstractPreAuthenticatedListener`,
  `AnonymousAuthenticationListener`, `BasicAuthenticationListener`, `RememberMeListener`,
  `RemoteUserAuthenticationListener`, `UsernamePasswordFormAuthenticationListener`,
  `UsernamePasswordJsonAuthenticationListener`, `X509AuthenticationListener` removed from
  security-http.** Check: `grep -rln 'AuthenticationListener\b' src/`. Fix: migrate to the new
  authenticator system (Phase 2).
- **Guard component removed entirely.**
  Check: `grep -rln 'extends AbstractGuardAuthenticator\|GuardAuthenticatorInterface' src/`. Fix:
  this is Investigation #4 / Phase 2's mandatory migration.
- **`TokenInterface::isAuthenticated()`/`setAuthenticated()` removed.**
  Check: `grep -rln '\->isAuthenticated()\|\->setAuthenticated(' src/`. Fix: check
  `getUser() !== null` instead.
- **`DeauthenticatedEvent` removed** → `TokenDeauthenticatedEvent`.
  Check: `grep -rln 'DeauthenticatedEvent\b' src/`. Fix: rename references.
- **`CookieClearingLogoutHandler`, `SessionLogoutHandler`, `CsrfTokenClearingLogoutHandler`
  removed** → `CookieClearingLogoutListener`, `SessionLogoutListener`,
  `CsrfTokenClearingLogoutListener`.
  Check: `grep -rln 'CookieClearingLogoutHandler\|SessionLogoutHandler\|CsrfTokenClearingLogoutHandler' src/`.
  Fix: switch to the listener classes.
- **`AuthenticatorInterface::createAuthenticatedToken()` removed** → `createToken()`; return type
  of `authenticate()` changed to `Passport`.
  Check: `grep -rln 'createAuthenticatedToken\|function authenticate(Request \$request): PassportInterface' src/`.
  Fix: rename method; update return type (see Phase 2's before/after).
- **`PassportInterface`, `UserPassportInterface`, `PassportTrait` removed** → use `Passport`
  directly. Check: `grep -rln 'PassportInterface\|UserPassportInterface\|PassportTrait' src/`. Fix:
  type-hint `Passport` instead.
- **`AccessDecisionManager` no longer accepts a string strategy** → `AccessDecisionStrategyInterface`
  instance. Check: `grep -rln 'new AccessDecisionManager(' src/`. Fix: rare — pass a strategy
  instance if manually constructing it.
- **`$credentials` argument removed** from `PreAuthenticatedToken`, `SwitchUserToken`,
  `UsernamePasswordToken` constructors.
  Check: `grep -rln 'new PreAuthenticatedToken(\|new SwitchUserToken(\|new UsernamePasswordToken(' src/`.
  Fix: drop the credentials argument from each call (see Phase 2's before/after).

**SecurityBundle**
- **`FirewallConfig::getListeners()` removed** → `getAuthenticators()`.
  Check: `grep -rln 'getListeners()' src/`. Fix: rename calls.
- **`security.authentication.basic_entry_point`/`retry_entry_point` services removed** (logic moved
  into `HttpBasicAuthenticator`/`ChannelListener`).
  Check: `grep -rln 'security.authentication.basic_entry_point\|security.authentication.retry_entry_point' src/ config/`.
  Fix: remove references.
- **`SecurityFactoryInterface`/`addSecurityListenerFactory()` removed** →
  `AuthenticatorFactoryInterface`/`addAuthenticatorFactory()`; `getPriority()` replaces
  `getPosition()` (position→priority mapping: `pre_auth`→-10, `form`→-30, `http`→-50,
  `remember_me`→-60, `anonymous`→-70).
  Check: `grep -rln 'SecurityFactoryInterface\|addSecurityListenerFactory' src/`. Fix: only
  relevant to a custom security authentication-method bundle; migrate the factory interface and
  priority per the mapping.
- **`MainConfiguration` no longer accepts an array-of-arrays as its 1st constructor argument.**
  Check: `grep -rln 'new MainConfiguration(' src/`. Fix: rare — internal; pass a sorted flat array
  of factories.
- **`always_authenticate_before_granting` option removed.**
  Check: `grep -n 'always_authenticate_before_granting' config/packages/security.yaml`. Fix: remove
  the key — no replacement.
- **`UserPasswordEncoderCommand`/`user:encode-password` removed** →
  `UserPasswordHashCommand`/`user:hash-password`.
  Check: `grep -rln 'user:encode-password\|UserPasswordEncoderCommand' src/ bin/`. Fix: use the new
  command name.
- **`security.encoder_factory.generic`/`security.encoder_factory`/`EncoderFactoryInterface`
  removed** → `security.password_hasher_factory`/`PasswordHasherFactoryInterface`.
  Check: `grep -rln 'EncoderFactoryInterface\|security.encoder_factory' src/`. Fix: switch to the
  password-hasher equivalents.
- **`security.user_password_encoder.generic`/`security.password_encoder`/
  `UserPasswordEncoderInterface` removed** →
  `security.user_password_hasher`/`security.password_hasher`/`UserPasswordHasherInterface`.
  Check: `grep -rln 'UserPasswordEncoderInterface\|security.password_encoder' src/`. Fix: switch to
  the password-hasher equivalents.
- **`security.authorization_checker`/`security.token_storage` services now private.**
  Check: `grep -rln "container->get('security.authorization_checker'\|container->get('security.token_storage'" src/`.
  Fix: autowire `AuthorizationCheckerInterface`/`TokenStorageInterface` instead.
- **Not setting `enable_authenticator_manager: true` now throws.**
  Check: `grep -n 'enable_authenticator_manager' config/packages/security.yaml`. Fix: must be
  `true` (Investigation #4 / Phase 2 step 1).
- **`security.authentication.provider.*` / `security.authentication.listener.*` services removed;
  Guard component integration removed from SecurityBundle.**
  Check: `grep -rln 'security.authentication.provider\.\|security.authentication.listener\.' src/ config/`
  (also see Investigation #4). Fix: remove references — replaced by the authenticator system.
- **Default provider for `custom_authenticators` removed when more than one provider is
  registered.** Check: `grep -n 'custom_authenticators' config/packages/security.yaml`. Fix: if
  multiple user providers exist, explicitly set `provider:` on the firewall.

**Serializer**
- **`ArrayDenormalizer::setSerializer()` removed** → `setDenormalizer()`; no longer implements
  `SerializerAwareInterface`. Check: `grep -rln 'ArrayDenormalizer' src/`. Fix: rename call if used
  directly.
- **Annotation classes no longer accept an array of parameters as the first constructor
  argument.** Check: `grep -rln 'new .*Constraint(\[' src/`. Fix: only relevant to code manually
  instantiating Serializer annotation classes; switch to named arguments.

**TwigBundle**
- **`twig` service now private.** Check: `grep -rln "container->get('twig')" src/`. Fix: autowire
  `Environment $twig` instead.

**Validator**
- **`Length` constraint's `allowEmptyString` option removed.**
  Check: `grep -rn 'allowEmptyString' src/`. Fix: replace with
  `#[Assert\AtLeastOneOf([new Assert\Blank(), new Assert\Length(min: ...)])]`.
- **`NumberConstraintTrait` removed.** Check: `grep -rln 'NumberConstraintTrait' src/`. Fix: remove
  usage — was internal.
- **`ValidatorBuilder::enableAnnotationMapping()` no longer accepts a Doctrine annotation reader
  argument; no longer auto-configures one.** Check: `grep -rln 'enableAnnotationMapping(' src/`.
  Fix: call `->enableAnnotationMapping()->setDoctrineAnnotationReader($reader)`, or
  `->addDefaultDoctrineAnnotationReader()` if no custom reader is needed.

**Workflow**
- **`InvalidTokenConfigurationException` removed.**
  Check: `grep -rln 'InvalidTokenConfigurationException' src/`. Fix: remove usage — dead code.

**Yaml**
- **Leading-zero numbers now parsed as strings, not octal** — prefix intentional octal with `0o`.
  Check: `grep -rnE "^\s*[A-Za-z0-9_]+: 0[0-7]" config/*.yaml config/packages/*.yaml` (manual review
  recommended — this grep is not fully reliable for nested YAML). Fix: prefix any intentional octal
  literal with `0o`, e.g. `Yaml::parse('072')` → `Yaml::parse('0o72')`.
- **`!php/object` / `!php/const` tags without a value no longer supported.**
  Check: `grep -rn '!php/object\s*$\|!php/const\s*$' config/`. Fix: provide a value after the tag.

### Symfony 6.1 (optional — 7.0 prep; nothing here blocks reaching 6.4)

**All components**
- `symfony/symfony` metapackage deprecated — already handled by Phase 4 step 1 (delete the line).
- Public/protected properties are now considered final. Check: no single reliable grep; manually
  review any class in `src/` that `extends` a Symfony class and overrides (not just reads) a
  property rather than a method. Fix: move the override into the constructor instead.

**Console**
- `Command::$defaultName`/`$defaultDescription` deprecated → `#[AsCommand]` attribute.
  Check: `grep -rln 'protected static \$defaultName\|protected static \$defaultDescription' src/`.
  Fix: replace with `#[AsCommand(name: '...', description: '...')]` on the command class.

**DependencyInjection**
- `ReferenceSetArgumentTrait` deprecated. Check: `grep -rln 'ReferenceSetArgumentTrait' src/`. Fix:
  inline the equivalent logic; check the current DI component API before a future 7.0 upgrade.

**FrameworkBundle**
- `reset_on_message` config option deprecated (only `true` has effect now).
  Check: `grep -n 'reset_on_message' config/packages/messenger.yaml`. Fix: remove an explicit
  `false`; use `--no-reset` on `messenger:consume` if that behavior is still needed.
- Not setting `http_method_override` deprecated (default flips to `false` in 7.0).
  Check: `grep -n 'http_method_override' config/packages/framework.yaml`. Fix: set explicitly
  (`true` to keep 6.x behavior) to silence the notice.

**HttpKernel**
- `StreamedResponseListener` deprecated (not needed anymore).
  Check: `grep -rln 'StreamedResponseListener' src/ config/`. Fix: remove any explicit reference.

**Serializer**
- `ContextAwareNormalizerInterface`/`ContextAwareDenormalizerInterface` deprecated →
  `NormalizerInterface`/`DenormalizerInterface`.
  Check: `grep -rln 'ContextAwareNormalizerInterface\|ContextAwareDenormalizerInterface' src/`. Fix:
  implement the base interfaces directly (context is now passed to their methods).
- `UidNormalizer` denormalizing to `AbstractUid` or an abstract class deprecated.
  Check: `grep -rln 'UidNormalizer' src/`. Fix: denormalize to a concrete `Uid` subclass.

**Validator**
- `Constraint::$errorNames` deprecated → `ERROR_NAMES` constant.
  Check: `grep -rln '::\$errorNames\|->errorNames' src/`. Fix: rename.
- `ExpressionLanguageSyntax` constraint deprecated → `ExpressionSyntax`.
  Check: `grep -rln 'ExpressionLanguageSyntax' src/`. Fix: rename.
- `ConstraintViolationInterface`/`ConstraintViolationListInterface` implementations without
  `__toString()` deprecated. Check: `grep -rln 'implements ConstraintViolationInterface\|implements ConstraintViolationListInterface' src/`.
  Fix: add a `__toString()` method.

### Symfony 6.2

**[BC BREAK] — mandatory now**

- **Notifier: `TransportTestCase::toStringProvider()`, `supportedMessagesProvider()`,
  `unsupportedMessagesProvider()`, and `createTransport()` are now `static`.**
  Check: `grep -rln 'extends TransportTestCase' tests/`. Fix: mark every overridden one of these
  four methods `static` in each test class extending `TransportTestCase` — this is required at
  6.2, not deferrable. (See also Investigation #9 — the same base class was already flagged as
  deprecated-non-static back in 5.4.)

**Optional — 7.0 prep**

- Config: `NodeBuilder::setParent()` called with zero arguments deprecated.
  Check: `grep -rln '\->setParent()' src/`. Fix: pass the parent explicitly.
- Console: `*Command::setApplication()`, `*FormatterStyle::setForeground/setBackground()`,
  `Helper::setHelpSet()`, `Input*::setDefault()`,
  `Question::setAutocompleterCallback/setValidator()` called with zero arguments deprecated;
  `OutputFormatterStyleInterface::setForeground/setBackground()` and
  `HelperInterface::setHelperSet()` signatures now nullable-typed.
  Check: `grep -rln '\->setApplication()\|\->setForeground()\|\->setBackground()\|\->setHelpSet()\|\->setDefault()\|\->setAutocompleterCallback()\|\->setValidator()' src/`.
  Fix: pass `null` explicitly instead of omitting the argument; update custom implementations'
  signatures to accept `?string`/`?HelperSet`.
- DependencyInjection: `ContainerAwareInterface::setContainer()` signature now
  `setContainer(?ContainerInterface)`; `ContainerAwareTrait::setContainer()` called with zero
  arguments deprecated; numeric parameter names deprecated.
  Check: `grep -rln 'ContainerAwareTrait\|ContainerAwareInterface' src/`; `grep -rnE '^\s*[0-9]+:' config/services.yaml`.
  Fix: pass `null` explicitly; rename numeric parameter keys to named strings.
- Form: `Button/Form::setParent()`, `ButtonBuilder/FormConfigBuilder::setDataMapper()`,
  `TransformationFailedException::setInvalidMessage()` called with zero arguments deprecated;
  `FormConfigBuilderInterface::setDataMapper()` now `setDataMapper(?DataMapperInterface)`;
  `FormInterface::setParent()` now `setParent(?self)`.
  Check: `grep -rln '\->setParent()\|\->setDataMapper()\|\->setInvalidMessage()' src/`. Fix: pass
  `null` explicitly; update custom form-type implementations' signatures.
- FrameworkBundle: `ObjectNormalizer`/`PropertyNormalizer` autowiring aliases deprecated →
  type-hint `NormalizerInterface` or implement `NormalizerAwareInterface`.
  Check: `grep -rln 'ObjectNormalizer \$\|PropertyNormalizer \$' src/`. Fix: type-hint
  `NormalizerInterface`, or implement `NormalizerAwareInterface`.
- FrameworkBundle: `AbstractController::renderForm()` deprecated → `render()` (which now handles
  form status codes itself). Check: `grep -rln '\->renderForm(' src/Controller/`. Fix: rename to
  `render()`.
- FrameworkBundle: `FrameworkExtension::registerRateLimiter()` deprecated.
  Check: `grep -rln 'registerRateLimiter' src/`. Fix: rare — only bundle/extension authors; use the
  current rate-limiter config API.
- HttpFoundation: `Request::getContentType()` deprecated → `getContentTypeFormat()`.
  Check: `grep -rln '\->getContentType()' src/`. Fix: rename calls.
- HttpFoundation: `JsonResponse::setCallback()`, `Response::setExpires/setLastModified/setEtag()`,
  `MockArraySessionStorage/NativeSessionStorage::setMetadataBag()`,
  `NativeSessionStorage::setSaveHandler()` called with zero arguments deprecated.
  Check: `grep -rln '\->setCallback()\|\->setExpires()\|\->setLastModified()\|\->setEtag()\|\->setMetadataBag()\|\->setSaveHandler()' src/`.
  Fix: pass `null` explicitly.
- HttpClient: implementing `Http\Message\RequestFactory`/`StreamFactory`/`UriFactory` on
  `HttplugClient` deprecated. Check: `grep -rln 'HttplugClient' src/`. Fix: rare — only custom
  HTTPlug factory implementations.
- HttpKernel: `ArgumentValueResolverInterface` deprecated → `ValueResolverInterface`.
  Check: `grep -rln 'implements ArgumentValueResolverInterface' src/`. Fix: implement
  `ValueResolverInterface` instead.
- HttpKernel: `ConfigDataCollector::setKernel()`, `RouterListener::setCurrentRequest()` called with
  zero arguments deprecated. Check: `grep -rln '\->setKernel()\|\->setCurrentRequest()' src/`. Fix:
  pass `null` explicitly.
- Ldap: `{username}` parameter deprecated → `{user_identifier}`.
  Check: `grep -rn '{username}' config/packages/security.yaml src/`. Fix: rename the placeholder.
- Mailer: `OhMySMTP` transport deprecated → `MailPace`.
  Check: `grep -rn 'ohmysmtp://' config/packages/mailer.yaml .env*`. Fix: switch the DSN scheme to
  `mailpace://`.
- Messenger: `MessageHandlerInterface`/`MessageSubscriberInterface` deprecated →
  `#[AsMessageHandler]` attribute. Check: `grep -rln 'implements MessageHandlerInterface\|implements MessageSubscriberInterface' src/`.
  Fix: replace with `#[AsMessageHandler]` on `__invoke()`.
- Mime: `Email::attachPart()` deprecated → `addPart()`; `Message::setBody()` called with zero
  arguments deprecated. Check: `grep -rln '\->attachPart(\|\->setBody()' src/`. Fix: rename to
  `addPart()`; pass `null` explicitly to `setBody()`.
- PropertyAccess: `PropertyAccessorBuilder::setCacheItemPool()` called with zero arguments
  deprecated; `PropertyPathInterface` implementations without `isNullSafe()` deprecated.
  Check: `grep -rln '\->setCacheItemPool()\|implements PropertyPathInterface' src/`. Fix: pass
  `null` explicitly; add `isNullSafe()` to any custom `PropertyPathInterface` implementation.
- Security: `UserBadge` enforces a 4096-character max username length (CVE-2016-4423 mitigation) —
  behavioral, not a code change. No action required.
- Security: `Security` class/service deprecated → `Bundle\SecurityBundle\Security`; several
  `Security::*_ERROR`/`LAST_USERNAME`/`MAX_USERNAME_LENGTH` constants deprecated →
  `SecurityRequestAttributes`/`UserBadge` equivalents.
  Check: `grep -rln "Security\\\\Core\\\\Security\b\|Security::ACCESS_DENIED_ERROR\|Security::AUTHENTICATION_ERROR\|Security::LAST_USERNAME\|Security::MAX_USERNAME_LENGTH" src/`.
  Fix: switch the `use` import to `Symfony\Bundle\SecurityBundle\Security`; use
  `SecurityRequestAttributes::*` / `UserBadge::MAX_USERNAME_LENGTH` for the constants.
- Security: `JsonLoginAuthenticator` with an empty username/password parameter no longer
  supported. Check: — (behavioral; review the login payload validation). Fix: ensure the client
  always sends non-empty username/password fields.
- Security: `TokenStorageInterface::setToken()` signature now `setToken(?TokenInterface $token)`;
  `TokenStorage`/`UsageTrackingTokenStorage::setToken()` called with zero arguments deprecated.
  Check: `grep -rln '\->setToken()' src/`. Fix: pass `null` explicitly.
- SecurityBundle: `security.enable_authenticator_manager` config option itself deprecated (the key
  is still required as `true` for 6.4 — only the *option* is slated for removal in 7.0).
  Check: `grep -n 'enable_authenticator_manager' config/packages/security.yaml`. Fix: no action
  needed for 6.4; plan to delete the line for a future 7.0 upgrade.
- Serializer: `AttributeMetadata::setSerializedName()`, `ClassMetadata::setClassDiscriminatorMapping()`
  called with zero arguments deprecated; interface signatures now nullable-typed.
  Check: `grep -rln '\->setSerializedName()\|\->setClassDiscriminatorMapping()' src/`. Fix: pass
  `null` explicitly.
- Translation: `PhpExtractor` deprecated → `PhpAstExtractor` (requires `nikic/php-parser`).
  Check: `grep -rln 'PhpExtractor\b' src/ config/`. Fix: switch to `PhpAstExtractor`; `composer
  require nikic/php-parser` if not already present.
- Validator: `loose` e-mail validation mode deprecated → `html5`.
  Check: `grep -rn "mode.*loose\|'loose'" config/packages/validator.yaml src/`. Fix: set
  `mode: 'html5'` explicitly (this also matches the 7.0-default change tracked in the 6.4 table).
- VarDumper: `VarDumper::setHandler()` called with zero arguments deprecated.
  Check: `grep -rln 'VarDumper::setHandler()' src/`. Fix: pass `null` explicitly.
- Workflow: `Registry` marked internal (use a tagged locator instead:
  `tagged_locator('workflow', 'name')`); `WorkflowDumpCommand`'s first argument should be a
  `ServiceLocator` of workflows by name; `Definition::setInitialPlaces()` called with zero
  arguments deprecated. Check: `grep -rln 'Workflow\\\\Registry\|\->setInitialPlaces()' src/`. Fix:
  use the tagged locator instead of `Registry`; pass `null` explicitly to `setInitialPlaces()`.

### Symfony 6.3

**[BC BREAK] — mandatory now**

- Same Notifier `TransportTestCase` static-method requirement as 6.2 above — the upstream 6.3 file
  repeats this entry verbatim. If already fixed for 6.2, nothing further to do; if not, fix now
  (this is the second and final place this guide will mention it).

**Optional — 7.0 prep**

- Console: `SignalableCommandInterface::handleSignal()` should return `int|false` instead of
  `void`, plus a new `$previousExitCode` 2nd argument.
  Check: `grep -rln 'implements SignalableCommandInterface' src/`. Fix: update the signature/return
  type in any custom command implementing this interface.
- DependencyInjection: `PhpDumper` options `inline_factories_parameter`/`inline_class_loader_parameter`
  deprecated → `inline_factories`/`inline_class_loader`; `#[MapDecorated]` deprecated →
  `#[AutowireDecorated]`; `@required` annotation deprecated → `Required` attribute; undefined/
  numeric keys in `service_locator` config deprecated.
  Check: `grep -rln '#\[MapDecorated\]\|@required' src/`; `grep -n 'inline_factories_parameter\|inline_class_loader_parameter' config/`.
  Fix: rename each per the mapping above.
- DoctrineBridge: passing Doctrine subscribers to `ContainerAwareEventManager` deprecated → use
  listeners via `#[AsDoctrineListener(event: ...)]` instead;
  `DoctrineDbalCacheAdapterSchemaSubscriber`/`MessengerTransportDoctrineSchemaSubscriber`/
  `RememberMeTokenProviderDoctrineSchemaSubscriber` deprecated → `*Listener` equivalents.
  Check: `grep -rln 'implements EventSubscriberInterface' src/` (then manually confirm Doctrine
  events); `grep -rln 'DoctrineDbalCacheAdapterSchemaSubscriber\|MessengerTransportDoctrineSchemaSubscriber\|RememberMeTokenProviderDoctrineSchemaSubscriber' src/`.
  Fix: convert each Doctrine event subscriber to a listener using `#[AsDoctrineListener]` (or tag
  it `doctrine.event_listener`); rename the schema-subscriber classes to their `*Listener`
  equivalents.
- Form: not configuring the `widget` option of date/time form types deprecated (defaults to
  `single_text` in 7.0). Check: `grep -rln 'DateType::class\|DateTimeType::class\|TimeType::class' src/`.
  Fix: set `'widget' => '...'` explicitly to avoid a silent behavior change on a future 7.0 upgrade.
- FrameworkBundle: `framework:exceptions` XML tag deprecated (unwrap it; `framework:exception`'s
  `name` attribute → `class`); `notifier.logger_notification_listener` deprecated →
  `notifier.notification_logger_listener`; `Http\Client\HttpClient` service deprecated →
  `Psr\Http\Client\ClientInterface`.
  Check: `grep -n '<framework:exceptions>' config/packages/framework.xml 2>/dev/null`; `grep -rln 'notifier.logger_notification_listener\|Http\\\\Client\\\\HttpClient' src/ config/`.
  Fix: restructure the XML per the source's before/after; rename service references.
- HttpClient: minimum TLS now defaults to v1.2; default user-agent strings renamed
  (`Symfony HttpClient/Amp` → `Symfony HttpClient (Amp)`, similarly for Curl/Native — RFC 9110
  compliance). Check: `grep -rn 'crypto_method\|Symfony HttpClient/' src/ config/ tests/`. Fix: set
  `crypto_method` explicitly if a server needs TLS < 1.2; update any test/log assertion matching
  the old user-agent string format.
- HttpFoundation: `ParameterBag::getInt()`/`getBoolean()` invalid-value conversion deprecated;
  `ParameterBag::filter()` ignoring invalid values deprecated unless `FILTER_NULL_ON_FAILURE` is
  set. Check: `grep -rln '\->getInt(\|\->getBoolean(\|\->filter(' src/`. Fix: review for invalid
  input handling; pass `FILTER_NULL_ON_FAILURE` explicitly where silent-ignore is intended.
- HttpKernel: `container.dumper.inline_factories`/`container.dumper.inline_class_loader`
  parameters deprecated → dot-prefixed names.
  Check: `grep -rn 'container.dumper.inline_factories\|container.dumper.inline_class_loader' config/ src/`.
  Fix: rename to `.container.dumper.inline_factories` / `.container.dumper.inline_class_loader`.
- Lock: `gcProbablity` (typo) deprecated → `gcProbability`.
  Check: `grep -rn 'gcProbablity' config/packages/lock.yaml src/`. Fix: rename the key.
- Messenger: `Messenger\Transport\InMemoryTransport`/`InMemoryTransportFactory` deprecated (moved
  to the `Transport\InMemory\` namespace); `StopWorkerOnSigtermSignalListener` deprecated →
  `StopWorkerOnSignalsListener`.
  Check: `grep -rln 'Messenger\\\\Transport\\\\InMemoryTransport\b\|StopWorkerOnSigtermSignalListener' src/`.
  Fix: update the `use` import to `Transport\InMemory\`; rename the listener.
- Security: `PersistentRememberMeHandler` constructor's 2nd argument (a secret) deprecated.
  Check: `grep -rln 'new PersistentRememberMeHandler(' src/`. Fix: remove the 2nd argument if
  passed.
- SecurityBundle: enabling the bundle without configuring it deprecated;
  `firewalls.logout.csrf_token_generator` deprecated → `csrf_token_manager`.
  Check: `grep -n 'csrf_token_generator' config/packages/security.yaml`. Fix: ensure at least one
  firewall is configured; rename the config key.
- Serializer: `CacheableSupportsMethodInterface` deprecated →
  `getSupportedTypes(?string $format)`; `ConstraintViolationListNormalizer`, `CustomNormalizer`,
  `DataUriNormalizer`, `DateIntervalNormalizer`, `DateTimeNormalizer`, `DateTimeZoneNormalizer`,
  `GetSetMethodNormalizer`, `JsonSerializableNormalizer`, `ObjectNormalizer`, `PropertyNormalizer`
  will become `final` in 7.0 (extending them is deprecated now).
  Check: `grep -rln 'implements CacheableSupportsMethodInterface' src/`; `grep -rlnE 'extends (ConstraintViolationListNormalizer|CustomNormalizer|DataUriNormalizer|DateIntervalNormalizer|DateTimeNormalizer|DateTimeZoneNormalizer|GetSetMethodNormalizer|JsonSerializableNormalizer|ObjectNormalizer|PropertyNormalizer)' src/`.
  Fix: implement `getSupportedTypes()` instead of `hasCacheableSupportsMethod()`; switch any
  subclass of the listed normalizers to composition/decoration instead of inheritance (inject the
  original via `#[Autowire(service: 'serializer.normalizer.object')]` — see source for the full
  before/after).
- Validator: `ConstraintViolationInterface` implementations without `getConstraint()` deprecated.
  Check: `grep -rln 'implements ConstraintViolationInterface' src/`. Fix: add a `getConstraint()`
  method.

### Symfony 6.4 (target version)

**[BC BREAK] — mandatory now**

- **Cache: `EarlyExpirationHandler` no longer implements `MessageHandlerInterface`.**
  Check: `grep -rln 'EarlyExpirationHandler' src/`. Fix: if any code type-checks against
  `MessageHandlerInterface` for this class, remove that assumption — it relies on
  `#[AsMessageHandler]` now.
- **DoctrineBridge: `ProxyCacheWarmer::warmUp()` gains a required `$buildDir` argument;
  `EntityFactory` gains return type-hints; `DoctrineTokenProvider::updateToken()`'s `$lastUsed`
  argument is now typed `DateTimeInterface`.**
  Check: `grep -rln 'extends ProxyCacheWarmer\|extends EntityFactory' src/`; `grep -rln 'DoctrineTokenProvider' src/`.
  Fix: update any overridden method signature to match.
- **ErrorHandler: `FlattenExceptionNormalizer` no longer implements
  `ContextAwareNormalizerInterface`.** Check: `grep -rln 'FlattenExceptionNormalizer' src/`. Fix:
  remove any type-check against `ContextAwareNormalizerInterface` for this class.
- **FrameworkBundle: native return type added to `Translator` and to `Application::reset()`.**
  Check: `grep -rln 'extends Translator\b' src/`; `grep -rln 'extends Application\b' src/bin 2>/dev/null`.
  Fix: update the overridden method signature(s) to match the new native return type.
- **FrameworkBundle: `AbstractController::render()` no longer calls `renderView()`.**
  Check: `grep -rln '\->renderView(' src/Controller/`. Fix: if a controller overrides `renderView()`
  expecting `render()` to invoke it, call `renderView()` explicitly instead.
- **HttpFoundation: `HeaderBag::getDate()`, `Response::getDate()`/`getExpires()`/
  `getLastModified()` now return `DateTimeImmutable` instead of `DateTime`.**
  Check: `grep -rln '\->getDate()\|\->getExpires()\|\->getLastModified()' src/`. Fix: remove any
  code that mutates the returned object in place (e.g. `->modify()`); reassign the result instead,
  since `DateTimeImmutable` methods return a new instance.
- **HttpKernel: `BundleInterface` no longer extends `ContainerAwareInterface`.**
  Check: `grep -rln 'implements BundleInterface' src/`. Fix: remove any assumption that a bundle is
  container-aware by default.
- **HttpKernel: native return types added to `TraceableEventDispatcher` and
  `MergeExtensionConfigurationPass`.**
  Check: `grep -rln 'extends TraceableEventDispatcher\|extends MergeExtensionConfigurationPass' src/`.
  Fix: update the overridden method signature(s).
- **MonologBridge: native return type added to `Logger::clear()` and `DebugProcessor::clear()`.**
  Check: `grep -rln 'extends.*Bridge\\\\Monolog\\\\Logger\|extends DebugProcessor' src/`. Fix:
  update the overridden method signature(s).
- **PsrHttpMessageBridge: `PsrServerRequestResolver` no longer implements
  `ArgumentValueResolverInterface`.** Check: `grep -rln 'PsrServerRequestResolver' src/`. Fix:
  remove any type-check against `ArgumentValueResolverInterface` for this class (it should already
  implement `ValueResolverInterface` per the 6.2 deprecation).
- **Routing: native return type added to `AnnotationClassLoader::setResolver()`.**
  Check: `grep -rln 'extends AnnotationClassLoader' src/`. Fix: update the overridden signature.
- **Security: `UserValueResolver` no longer implements `ArgumentValueResolverInterface`.**
  Check: `grep -rln 'UserValueResolver' src/`. Fix: remove any type-check against
  `ArgumentValueResolverInterface` for this class.
- **Security: `PersistentToken` made immutable.**
  Check: `grep -rln 'new PersistentToken(\|PersistentToken.*->set' src/`. Fix: stop mutating a
  `PersistentToken` after construction; construct a new one instead.
- **Security: `DefaultLoginRateLimiter`'s constructor gains a required `string $secret`
  parameter.** Check: `grep -rln 'new DefaultLoginRateLimiter(' src/`. Fix: if the app manually
  instantiates or decorates this class (rather than relying on the `login_throttling` config to
  wire it automatically), pass the app secret explicitly.
- **Translator: `DataCollectorTranslator::warmUp()` gains a required `$buildDir` argument.**
  Check: `grep -rln 'extends DataCollectorTranslator' src/`. Fix: update the overridden signature.

**Optional — 7.0 prep**

- DependencyInjection: `ContainerAwareInterface`/`ContainerAwareTrait` deprecated → constructor
  injection or a service subscriber. Check: `grep -rln 'implements ContainerAwareInterface\|use ContainerAwareTrait' src/`.
  Fix: inject the needed services via the constructor instead of fetching from `$this->container`;
  use a service subscriber for lazy fetching.
- DoctrineBridge: `DbalLogger` deprecated → use a DBAL middleware; `DoctrineDataCollector`
  construction without a `DebugDataHolder` deprecated; `DoctrineDataCollector::addLogger()`
  deprecated; `ContainerAwareLoader` deprecated → dependency injection in fixtures.
  Check: `grep -rln 'DbalLogger\|ContainerAwareLoader' src/`. Fix: switch `DbalLogger` usage to a
  DBAL middleware; construct `DoctrineDataCollector` with a `DebugDataHolder`; use DI in fixtures.
- Form: `DateType`/`DateTimeType`/`TimeType` with model data in a different timezone than
  `model_timezone` deprecated; `PostSetDataEvent::setData()` deprecated →
  `PreSetDataEvent::setData()`; `PostSubmitEvent::setData()` deprecated →
  `PreSubmitDataEvent`/`SubmitDataEvent::setData()`.
  Check: `grep -rln 'PostSetDataEvent\|PostSubmitEvent' src/`. Fix: move `setData()` calls to the
  earlier event; ensure model data's timezone matches `model_timezone`.
- FrameworkBundle: doctrine/annotations integration deprecated (already tracked as Investigation
  #5); 8 config-option defaults changing in 7.0:

  | option | default in 6.x | default in 7.0+ |
  |---|---|---|
  | `framework.http_method_override` | `true` | `false` |
  | `framework.handle_all_throwables` | `false` | `true` |
  | `framework.php_errors.log` | `'%kernel.debug%'` | `true` |
  | `framework.session.cookie_secure` | `false` | `'auto'` |
  | `framework.session.cookie_samesite` | `null` | `'lax'` |
  | `framework.session.handler_id` | `'session.handler.native'` | `null` if `save_path` unset, else `'session.handler.native_file'` |
  | `framework.uid.default_uuid_version` | `6` | `7` |
  | `framework.uid.time_based_uuid_version` | `6` | `7` |
  | `framework.validation.email_validation_mode` | `'loose'` | `'html5'` |

  Check: `grep -n 'http_method_override\|handle_all_throwables\|php_errors\|cookie_secure\|cookie_samesite\|handler_id\|default_uuid_version\|time_based_uuid_version\|email_validation_mode' config/packages/framework.yaml`.
  Fix: set each option explicitly (either the 6.x value to keep current behavior, or the 7.0 value
  to opt in early) — not required for 6.4, but each unset option produces a deprecation notice.
- FrameworkBundle: `framework.validation.enable_annotations` deprecated →
  `enable_attributes`; `framework.serializer.enable_annotations` deprecated → `enable_attributes`;
  `routing.loader.annotation[.directory/.file]` services deprecated →
  `routing.loader.attribute[.directory/.file]`; `AnnotatedRouteControllerLoader` deprecated →
  `AttributeRouteControllerLoader`.
  Check: `grep -n 'enable_annotations' config/packages/*.yaml`; `grep -rln 'routing.loader.annotation\|AnnotatedRouteControllerLoader' src/ config/`.
  Fix: rename per the mapping.
- HttpKernel: `Kernel::stripComments()` deprecated; `UriSigner` deprecated (use HttpFoundation's);
  `FileLinkFormatter` deprecated (use ErrorHandler's).
  Check: `grep -rln 'HttpKernel\\\\UriSigner\|HttpKernel\\\\.*FileLinkFormatter' src/`. Fix: switch
  the `use` imports to the new component namespaces.
- Messenger: `StopWorkerOnSignalsListener` deprecated in favor of implementing
  `SignalableCommandInterface` directly; `HandlerFailedException::getNestedExceptions()`/
  `getNestedExceptionsOfClass()` and `DelayedMessageHandlingException::getExceptions()` deprecated
  → `getWrappedExceptions()`.
  Check: `grep -rln 'getNestedExceptions(\|getNestedExceptionsOfClass(\|DelayedMessageHandlingException' src/`.
  Fix: rename to `getWrappedExceptions()`; implement `SignalableCommandInterface` on the consume
  command instead of registering the listener.
- RateLimiter: `SlidingWindow::getRetryAfter` deprecated → `calculateTimeForTokens`.
  Check: `grep -rln '\->getRetryAfter(' src/`. Fix: rename calls.
- Routing: Doctrine annotations support deprecated in favor of native attributes;
  `AnnotationClassLoader`/`AnnotationDirectoryLoader`/`AnnotationFileLoader` deprecated →
  `Attribute*` equivalents. Check: `grep -rln 'AnnotationClassLoader\|AnnotationDirectoryLoader\|AnnotationFileLoader' src/`.
  Fix: rename the loader classes.
- Security: `TokenProviderInterface::updateToken()` accepting only `DateTime` deprecated →
  `DateTimeInterface`. Check: `grep -rln 'implements TokenProviderInterface' src/`. Fix: widen the
  type hint if overridden.
- SecurityBundle: `require_previous_session` config option deprecated (already has no effect).
  Check: `grep -n 'require_previous_session' config/packages/security.yaml`. Fix: safe to delete —
  it already does nothing.
- Serializer: Doctrine annotations support deprecated → native attributes; `AnnotationLoader`
  deprecated → `AttributeLoader`.
  Check: `grep -rln 'Serializer\\\\Mapping\\\\Loader\\\\AnnotationLoader' src/ config/`. Fix: switch
  to `AttributeLoader`.
- Templating: the component is deprecated entirely, removed in 7.0 → use Twig.
  Check: `grep -q '"symfony/templating"' composer.json && echo yes || echo no`. Fix: if `yes`, plan
  a migration to Twig before any future 7.0 upgrade — not required for 6.4 itself.
- Validator: Doctrine annotations support deprecated → native attributes;
  `ValidatorBuilder::setDoctrineAnnotationReader()`/`addDefaultDoctrineAnnotationReader()`/
  `enableAnnotationMapping()`/`disableAnnotationMapping()` deprecated →
  `enableAttributeMapping()`/`disableAttributeMapping()`; `AnnotationLoader` deprecated →
  `AttributeLoader`. Check: `grep -rln 'enableAnnotationMapping\|disableAnnotationMapping\|setDoctrineAnnotationReader\|addDefaultDoctrineAnnotationReader' src/`.
  Fix: rename to the `*AttributeMapping()` equivalents (related to Investigation #5).
- VarExporter: per-property lazy-initializers deprecated.
  Check: `grep -rln 'VarExporter' src/` (manual review — narrow, internal use). Fix: rare; review
  against current docs if flagged.
- Workflow: `GuardEvent::getContext()` deprecated, will be removed in 7.0.
  Check: `grep -rln '\->getContext()' src/` (on a `GuardEvent` specifically — this grep will
  over-match other `getContext()` methods; confirm the receiver type manually). Fix: rare — review
  workflow guard listeners; no direct replacement documented yet, track for a future 7.0 upgrade.

Additive-only, no action required at any version above (listed for completeness, since they were
in the source files): `Command::addArgument`/`addOption`/`InputArgument`/`InputOption` gain
`$suggestedValues` (6.1); `RouteCollection::add()` gains `$priority` (6.0);
`LoginLinkHandlerInterface::createLoginLink()` gains `$lifetime` (6.2);
`UrlMatcher::handleRouteRequirements()` gains `$routeParameters` (6.1);
`AbstractBrowser::click()`/`clickLink()` gain `$serverParameters` (6.4); `Crawler::attr()` gains
`$default` (6.4); `Response::sendHeaders()` gains optional `$statusCode` (6.3);
`FormConfigInterface::getIsEmptyCallback()`/`FormConfigBuilderInterface::setIsEmptyCallback()`
(6.0); `ChoiceListFactoryInterface` filter argument (6.0); `DoctrineDbalAdapter`/
`DoctrineTransport`/`DoctrineTokenProvider`/`DoctrineDbalStore` gain optional `$isSameDatabase`
(6.3); `Translation\PhpAstExtractor` added (6.1).

## Step-by-step implementation plan

### Phase 1 — Land on 5.4, zero deprecations

1. `BRIDGE` check: `grep -q '"symfony/phpunit-bridge"' composer.json && echo yes || echo no`. If
   `no` → `composer require --dev symfony/phpunit-bridge`.
2. Run `./bin/phpunit` (or `./vendor/bin/simple-phpunit`). Read the `Remaining deprecation notices`
   block at the end. Fix each one; re-run after each fix or small batch.
3. Load app with `APP_ENV=dev` in a browser. Check web debug toolbar's deprecation counter for
   anything CLI tests missed (template/routing deprecations only surface on HTTP requests).
4. Repeat steps 2–3 until the count is `0`. Do not start Phase 2 with a nonzero count.

### Phase 2 — Migrate to the authenticator-based security system

Skip entirely if Investigation #4 showed "Already on new authenticator system."

1. In `config/packages/security.yaml`, set (or confirm) `security.enable_authenticator_manager: true`.
2. For each file listed by Investigation #4's second grep:
   ```php
   // Before: extends AbstractGuardAuthenticator
   public function getCredentials(Request $request) { /* ... */ }
   public function getUser($credentials, UserProviderInterface $userProvider) { /* ... */ }
   public function checkCredentials($credentials, UserInterface $user): bool { /* ... */ }

   // After: implements AuthenticatorInterface (or extend AbstractAuthenticator)
   public function authenticate(Request $request): Passport
   {
       return new Passport(
           new UserBadge($identifier, fn () => $this->userRepository->find($identifier)),
           new PasswordCredentials($password)
       );
   }
   ```
   Illustrative only — the exact `Passport` badges depend on what `getCredentials`/`checkCredentials`
   did (password check → `PasswordCredentials`, API token → `SelfValidatingPassport`, etc.).
3. On the `User` entity (or wherever `UserInterface` is implemented):
   - `getUsername()` → rename to `getUserIdentifier()` (interface no longer declares `getUsername()`).
   - `getPassword()` → keep only if implementing `PasswordAuthenticatedUserInterface`; otherwise remove.
   - `getSalt()` → remove unless implementing `LegacyPasswordAuthenticatedUserInterface` (modern
     hashers don't use a separate salt).
4. `grep -rln 'getUsername()\|->getSalt()' src/` → fix every remaining caller (controllers, voters,
   event subscribers) to call `getUserIdentifier()` instead.
5. Run `./bin/phpunit`. Any `AuthenticationException` or `Undefined method` failures here are this
   phase's responsibility — fix before moving to Phase 3.

### Phase 3 — Add native PHP return types

Symfony 6 declares return types on (almost all) methods it defines. Any class in this project that
extends/implements a Symfony class/interface without a matching return type will fatal with
`Declaration ... must be compatible with ...` the moment Phase 4 lands 6.4 — fix this now, while
still on 5.4, so the failure surfaces as a deprecation instead of a fatal error.

1. Confirm the tool exists (ships with `symfony/error-handler`, already a transitive dependency of
   `symfony/framework-bundle`):
   ```bash
   test -f vendor/bin/patch-type-declarations && echo yes || echo no
   ```
   `no` → `composer require symfony/error-handler`, re-check.
2. ```bash
   composer dump-autoload -o
   SYMFONY_PATCH_TYPE_DECLARATIONS="force=2" vendor/bin/patch-type-declarations
   ```
   `force=2` patches every method it can, including non-`@internal`/non-`final` ones — needed here
   because the goal is full 6.4 compatibility, not just Symfony's own conservative default.
3. ```bash
   git diff --stat
   ```
   Review the diff — the tool only adds return-type declarations, it doesn't change logic. Run
   `./bin/phpunit` after; any new failures mean a method's actual return value doesn't match the
   type the tool inferred — fix the method body, not the type.

### Phase 4 — Bump Composer constraints to 6.4

1. Edit `composer.json`: every `"symfony/<name>": "5.4.*"` or `"^5.4"` → `"^6.4"`. Confirm:
   ```bash
   grep '"symfony/' composer.json
   ```
   Every line shows `6.4` — none show `5.4`. Delete any `"symfony/symfony": "..."` line entirely
   (deprecated metapackage — also the 6.1 deprecation entry above).
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
   composer why-not symfony/framework-bundle 6.4.0
   ```
   (substitute the actual `symfony/*` package from the `Problem` block)
   - Lists every package blocking that version + its exact constraint.
   - `composer show <package> --all` → list available versions.
   - `composer show <package> <version>` → check that version's own `require` for a
     `symfony/*: ^6.0`-compatible line.
   - Found a compatible version → widen its constraint in `composer.json`, re-run step 2.
   - No compatible version exists anywhere → STOP. Do NOT use `--ignore-platform-reqs` or remove
     the package unasked. Tell the user the exact package name; let them decide.
5. Repeat steps 2–4 until step 2's command completes with no `Problem` blocks.
6. Error mentions `requires php` + `does not satisfy that requirement`? That's Phase 6's concern —
   note it, keep resolving dependency versions, don't fix PHP here.
7. Messenger transport error mentioning `amqp`, `redis`, or `doctrine` transport DSN not found?
   These transports moved to separate packages in 6.0 (also the 6.0 breaking-changes entry above):
   ```bash
   composer require symfony/amqp-messenger  # only if messenger.yaml uses an amqp:// DSN
   composer require symfony/redis-messenger # only if messenger.yaml uses a redis:// DSN
   composer require symfony/doctrine-messenger # only if messenger.yaml uses a doctrine:// DSN
   ```

### Phase 5 — Flex recipes / config files

1. `FLEX_PLUGIN` check: `grep -q '"name": "symfony/flex"' composer.lock && echo yes || echo no`.
   `yes`:
   ```bash
   git add -A && git commit -m "Bump Symfony to 6.4"
   composer recipes
   ```
   For each package marked `outdated`: `composer recipes:update <package-name>` (one at a time),
   then `git diff` to review before moving to the next package.
2. `FLEX_PLUGIN=no`, or a package has no recipe update:
   ```bash
   composer create-project symfony/skeleton:"6.4.*" /tmp/symfony-6.4-skeleton
   ```
   Manually diff project's `config/packages/*.yaml`, `config/routes.yaml`, `config/services.yaml`,
   `.env` against the skeleton's equivalents. Apply differences by hand — preserve the project's
   actual config values, don't overwrite wholesale.
3. Cross-reference remaining direct dependencies from Investigation #3 against the "Complete
   breaking-changes list" section above for any package-specific note not already surfaced by a
   grep (some third-party bundles depend on Symfony internals this guide's checks don't cover).

### Phase 6 — Docker image → PHP ≥ 8.1.0

1. Investigation #2's table: does the current default-apt PHP meet the ≥ 8.1.0 floor?
2. Decision:

   | Condition | Action |
   |---|---|
   | Table shows PHP 8.1 or 8.3 (ubuntu:22.04 / ubuntu:24.04), no PPA needed | No base-image change. Skip to step 5 |
   | Default PHP < 8.1, or Ubuntu release not one of the 4 table rows | Add `ondrej/php` PPA (below). Only if base image is an Ubuntu LTS release (18.04/20.04/22.04/24.04) — otherwise STOP, tell user |
   | Prefer bumping the base image itself | Change `FROM` line to `ubuntu:22.04` or newer, then re-check every other `apt-get install` line still exists under that name on the new release |

   `ondrej/php` PPA block (illustrative — substitute real target version + extension list from
   Investigation #2):
   ```dockerfile
   RUN apt-get update && apt-get install -y software-properties-common \
       && add-apt-repository ppa:ondrej/php \
       && apt-get update \
       && apt-get install -y \
          php8.1-fpm php8.1-cli \
          php8.1-xml php8.1-mbstring php8.1-intl php8.1-pgsql
          # one line per extension package from Investigation #2, version number updated
   ```
3. Rebuild without cache:
   ```bash
   docker build --no-cache -t symfony6-upgrade-test .
   ```
4. Verify:
   ```bash
   docker run --rm symfony6-upgrade-test php -v
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

### Phase 7 — Work through the complete breaking-changes list

1. Go through every entry in the "Complete breaking-changes list, version by version" section's
   **Symfony 6.0** table — every one is mandatory. For each: run the Check; a match means apply the
   Fix now, before moving to the next entry.
2. Go through the **6.2**, **6.3**, and **6.4** tables' `[BC BREAK] — mandatory now` subsections —
   these apply immediately even though the app is jumping straight to 6.4. (6.1 has no `[BC BREAK]`
   entries; skip straight past it for mandatory fixes.)
3. Optionally (recommended, not required to reach 6.4): go through the `Optional — 7.0 prep`
   subsections of 6.1–6.4. Fixing these now means a future 6→7 upgrade starts with a much smaller
   deprecation list. Record which ones were deferred, and why, in the final report.
4. Run the full test suite (`./bin/phpunit`, then Phase 8 covers pinning its version). Every fatal
   `Error` on an unknown class/method not already covered by a row above → re-search this section
   for that exact symbol name; if genuinely absent, that's a gap in this guide's research — tell
   the user the exact class/method name and where it was found, don't guess a fix.

### Phase 8 — PHPUnit → Symfony-6-compatible version

1. `phpunit.xml.dist`: existing `SYMFONY_PHPUNIT_VERSION` entry → set `value="9.6"` (check
   `https://packagist.org/packages/phpunit/phpunit#9.6` for the current latest `9.x` patch first —
   don't hardcode blindly). No entry → add inside `<php>`:
   ```xml
   <php>
       <server name="SYMFONY_PHPUNIT_VERSION" value="9.6"/>
   </php>
   ```
2. Existing `SYMFONY_MAX_PHPUNIT_VERSION` entry → remove it. Re-add only if a specific dependency's
   incompatibility with PHPUnit 9 shows up after step 4.
3. ```bash
   rm -rf vendor/bin/.phpunit
   ```
4. Run `./bin/phpunit` or `./vendor/bin/simple-phpunit`. Two failure categories — fix separately:
   - PHPUnit method no longer exists (removed in 9.x) → test-code fix.
   - Application code failure → continue Phase 7's removed-API sweep.
5. Repeat until `0 failures, 0 errors`.

## Verification steps

| # | Check | Command / method |
|---|---|---|
| 1 | Suite passes in the rebuilt image | `docker run --rm symfony6-upgrade-test ./bin/phpunit` |
| 2 | Container/config sane | `php bin/console debug:container` and `debug:config` both exit `0` |
| 3 | Deprecation count is understood, not just "zero" | Boot `APP_ENV=dev` in browser, read the web debug toolbar's deprecation list. Every notice must be one of: (a) an accepted, explicitly deferred item from Investigation #5 (Doctrine annotations) or the "Optional — 7.0 prep" subsections above, or (b) a genuinely new regression. Landing on 6.4 legitimately surfaces deprecations that Symfony itself introduced in 6.1–6.4 preparing for 7.0 (`Command::$defaultName`, `ContainerAwareInterface`, annotation-loader deprecations, the 8 config-default changes, etc.) — these are expected at this guide's 6.4 target and are not blockers; fixing them now is optional hardening for a future 7.0 upgrade, not a requirement here. A deprecation notice naming a class/method this guide's breaking-changes list does not mention anywhere is the one case that's a real, unaccounted-for regression — treat it as such |
| 4 | Auth works | Manually exercise every auth path (form login, API token, custom authenticator) |
| 5 | Messenger works (if Phase 4 step 7 applied) | Dispatch one real message per changed transport, confirm consumed |
| 6 | Clean-cache rebuild | `docker build --no-cache -t symfony6-upgrade-final .`, re-run checks 1–3 against it |
| 7 | Sources agree | Re-run Investigation #2's 3 commands; `composer.json`, `composer.lock`, Dockerfile now consistent (or discrepancy was flagged on purpose) |

## Rollback / risk notes

- Phase 1 (deprecation fixes) and Phase 3 (return-type patches) are their own commits on the
  still-5.4 codebase, merged independently before Phase 4 → safe to keep even if everything after
  is rolled back.
- Phase 2 (security migration) is a behavioral change independent of the Symfony version bump —
  land and verify it as its own PR before Phase 4, so an auth regression is bisectable from a
  dependency-bump regression.
- Phase 4 onward: dedicated branch. `composer.json`/`composer.lock` diff + Dockerfile diff are
  coupled — revert together. Never ship the Phase 6 PHP bump without the Phase 4 Symfony bump (old
  Symfony 5 code on untested new PHP, no passing test run backing it).
- Phase 7's sweep touches many small, unrelated call sites across the codebase — commit it in
  logical chunks per component (e.g. one commit for Security fixes, one for FrameworkBundle service
  renames, one for Messenger transport packages) rather than one giant commit, so a regression in
  one area doesn't force reverting all of them.
- Keep the pre-upgrade Docker image tag available until the new one has run clean through one full
  staging deployment cycle.
- Investigation #6/#8 (SensioFrameworkExtraBundle removal, annotations→attributes) and any
  "Optional — 7.0 prep" items deferred → record explicitly in PR/task description with the reason.
  All of these get harder to defer once a future Symfony 7 upgrade or Doctrine ORM 3.0 bump is in
  scope.

## References

Fetched and read in full (not summarized from a secondary source) to build the "Complete
breaking-changes list" section above:

- [UPGRADE-5.4.md](https://github.com/symfony/symfony/blob/5.4/UPGRADE-5.4.md) — confirmed this
  file contains only deprecations, zero removals, which is why Prerequisites' P2 fix-up path relies
  on Phase 1's deprecation-driven process rather than a separate breaking-change table
- [UPGRADE-6.0.md](https://github.com/symfony/symfony/blob/6.0/UPGRADE-6.0.md) — every entry across
  all ~35 listed components transcribed into the 6.0 table above
- [UPGRADE-6.1.md](https://github.com/symfony/symfony/blob/6.1/UPGRADE-6.1.md) — every entry
  transcribed into the 6.1 table above; confirmed zero `[BC BREAK]`-tagged entries
- [UPGRADE-6.2.md](https://github.com/symfony/symfony/blob/6.2/UPGRADE-6.2.md) — every entry
  transcribed; the two Notifier `[BC BREAK]` entries are called out separately from the
  deprecations
- [UPGRADE-6.3.md](https://github.com/symfony/symfony/blob/6.3/UPGRADE-6.3.md) — every entry
  transcribed; confirmed the same two Notifier `[BC BREAK]` entries recur verbatim from 6.2 (not a
  second, different break)
- [UPGRADE-6.4.md](https://github.com/symfony/symfony/blob/6.4/UPGRADE-6.4.md) — every entry across
  all 21 listed components/bridges transcribed; this file has by far the most `[BC BREAK]`-tagged
  entries of the four 6.x minors, which directly contradicts a "later minors are BC-safe" assumption
- [Symfony 6.0 Release](https://symfony.com/releases/6.0) — PHP requirement (≥8.0.2); note 6.0 itself
  reached end-of-support January 2023, which is why this guide lands on 6.4
- [Symfony 6.4 Release](https://symfony.com/releases/6.4) — LTS status, PHP requirement (≥8.1.0),
  bug-fix support to November 2026 / security support to November 2027
- [Symfony | endoflife.date](https://endoflife.date/symfony) — support/security-EOL dates (re-check;
  dates shift)
- [Upgrading a Major Version (Symfony 6.4 docs)](https://symfony.com/doc/6.4/setup/upgrade_major.html)
  — official process Phases 1–5 are based on, including the `patch-type-declarations` tool
- [Upgrading a Minor Version (Symfony 6.4 docs)](https://symfony.com/doc/6.4/setup/upgrade_minor.html)
  — explains the `[BC BREAK]` marker convention used throughout the 6.1–6.4 tables above
- [Symfony 6 Native Typing (Wouter J)](https://wouterj.nl/2021/09/symfony-6-native-typing) —
  explains why Phase 3's return-type patch is needed before the version bump, not after
- [From annotations to attributes (Doctrine blog)](https://www.doctrine-project.org/2022/11/04/annotations-to-attributes.html)
  — confirms annotations remain supported through Symfony 6.x but not future Doctrine ORM 3.0
- [SensioFrameworkExtraBundle README](https://github.com/sensiolabs/SensioFrameworkExtraBundle) —
  confirms its annotations were merged into FrameworkBundle as native attributes in Symfony 6.2
- [The PHPUnit Bridge (Symfony 6.x docs)](https://symfony.com/doc/6.4/components/phpunit_bridge.html)
  — `SYMFONY_PHPUNIT_VERSION` / `SYMFONY_MAX_PHPUNIT_VERSION` mechanics
- [Composer `why-not`/`prohibits`](https://getcomposer.org/doc/03-cli.md) — used mechanically in
  Phase 4 to find blocking packages
- [Composer config platform reference](https://getcomposer.org/doc/06-config.md) — confirms
  `platform-overrides` only reflects an explicit override, not real ambient PHP (why Investigation
  #2 treats the Dockerfile as ground truth)
- Ubuntu default PHP-per-release mapping (18.04→7.2, 20.04→7.4, 22.04→8.1, 24.04→8.3) + `ondrej/php`
  PPA — general community knowledge; re-verify if applying to an Ubuntu release not listed, mapping
  shifts with each new LTS.
