# 7. Contributing

Pine Commerce is MIT-licensed and contributions are welcome: bug reports, fixes, features and
documentation.

**Contents**

- [Where things go](#where-things-go)
- [Repository layout (ecom-core)](#repository-layout-ecom-core)
- [Setting up for development](#setting-up-for-development)
- [Running the tests](#running-the-tests)
- [Coding conventions](#coding-conventions)
- [Compatibility rules](#compatibility-rules)
- [Pull request process](#pull-request-process)
- [Reporting a security issue](#reporting-a-security-issue)
- [Documentation contributions](#documentation-contributions)

---

## Where things go

| Change | Repository |
|---|---|
| Platform code, default theme, admin, importer, commands, the core's reference docs | [ecom-core](https://github.com/SetWebUK/ecom-core) |
| The starter app | `stubs/client-skeleton` **in ecom-core** – [ecom-skeleton](https://github.com/SetWebUK/ecom-skeleton) is generated from it, so don't send changes there |
| Guides, onboarding and overview docs | this repository, [pine-commerce](https://github.com/SetWebUK/pine-commerce) |
| Something only one shop needs | that shop's own project – via config, its theme or the extension API ([Building features](04-building-features.md)) |

## Repository layout (ecom-core)

```
composer.json · VERSION · CHANGELOG.md · LICENSE · phpunit.xml.dist
bin/export-client-skeleton.sh   renders stubs/client-skeleton into the ecom-skeleton repository
config/commerce.php             package defaults – every key documented (feature flags: 'features')
config/commerce-import.php      importer defaults
database/migrations/            store schema (additive only)
docs/                           Playbook, architecture contract, themes, importer, extending, upgrading, admin UI …
resources/assets/admin/         admin CSS/JS/images → published (copied) to public/vendor/commerce/admin
resources/themes/default/       the neutral default theme (end of every theme chain)
resources/views/admin/          the back office (commerce::admin.*); components/admin → <x-admin.*>
routes/                         storefront.php, admin.php + admin/*.php
src/                            Pine\Commerce\… (Commerce.php = extension API, Console, Http, Models, Services,
                                Theme, Import, Scheduling, Updater, Support …)
stubs/client-skeleton/          the starter-app template (commerce:new-client, ecom-skeleton)
tests/                          package tests (Testbench, in-memory SQLite)
```

## Setting up for development

```bash
git clone https://github.com/SetWebUK/ecom-core.git && cd ecom-core
composer install
composer test
composer validate --strict
```

To work on the core against a real shop on your machine, point the shop at your clone with a temporary **path
repository** – never commit it:

```json
"repositories": [{ "type": "path", "url": "../ecom-core", "options": { "symlink": true } }],
"require": { "pine/commerce": "*@dev" }
```

```bash
composer update pine/commerce          # vendor/pine/commerce is now a symlink to your clone
php artisan optimize:clear && php artisan commerce:publish && php artisan commerce:theme:publish
```

Switch back before committing anything in the shop: `git checkout composer.json composer.lock && composer install`.

## Running the tests

```bash
composer test          # PHPUnit on Orchestra Testbench, in-memory SQLite – never a real database
```

- Tests extend `Pine\Commerce\Tests\TestCase` and set their own config.
- Database tests use in-memory SQLite. A few tests that need MySQL's `scratch` connection (`zz_` tables, dropped in
  `tearDown()`) skip when run standalone.
- Every extension point has a test in `tests/Feature/ExtensionApiTest.php`; feature flags in
  `tests/Feature/FeatureFlagsTest.php`. Add or extend tests with every change.
- **Never** use `RefreshDatabase` (or `migrate:fresh`) against a database with real data.

## Coding conventions

- **No admin or e-commerce frameworks.** No Filament, Livewire, Nova, Backpack, Lunar, Bagisto or similar. The back
  office is hand-built Blade + Alpine.js with the `<x-admin.*>` components – see
  [ADMIN_UI.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/ADMIN_UI.md).
- **No Node build step.** Hand-written CSS and plain JavaScript; vendored libraries are committed as files (Alpine,
  SortableJS, Chart.js, TinyMCE in the admin).
- **The package stays client-neutral.** It names no shop, brand, host, server path, person or credential – in code,
  tests, fixtures or docs. Shop-specific values become config keys with neutral defaults, theme files, or extension
  points. Use `acme` / `example.test` in examples.
- **Shared hosting first.** Assume the `sync` queue, no cron (scheduled work must also run from the web fallback and be
  safe to run late or twice), no symlinks out of `public/`, no Redis, PHP 8.3.
- **Laravel conventions**: thin controllers, FormRequests that return a normalised data array (never
  `$request->all()`), eager loading and pagination on every list, CSRF on every form, `{{ }}` escaping.
- **Raw SQL** names tables through `Pine\Commerce\Support\Sql` so prefixed connections keep working.
- **Friendly copy** for non-technical shop managers; accessible markup (labels, focus, ARIA) in the admin and the
  default theme.
- Follow the surrounding code style (PSR-12 / Laravel style); keep diffs focused – no unrelated reformatting.

## Compatibility rules

Every existing shop must keep working – and look identical – after a minor or patch release:

1. New behaviour ships **behind a config key or feature flag** whose default keeps current behaviour.
2. Migrations are **additive only**: new tables and columns, never renames or drops.
3. The public API (theme contract, config keys, `Commerce::` API, events, importer interfaces, route names, schema) only
   breaks in a major release – see [UPGRADING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/UPGRADING.md).
4. Default-theme changes must not alter the markup of shops built on it unless they opt in.
5. Add a line under `## [Unreleased]` in `CHANGELOG.md`: what changed, config keys added, contract changes, and any
   **client actions required**.

## Pull request process

1. Open an issue first for anything larger than a small fix, so the approach can be agreed.
2. Fork ecom-core, branch from `main` (`feature/short-name` or `fix/short-name`).
3. Make the change with tests; update the relevant docs in `docs/` and the `[Unreleased]` CHANGELOG section.
4. `composer test` and `composer validate --strict` pass.
5. Open the pull request against `main`, describing the problem, the change, how you tested it, and any upgrade notes.
6. A maintainer reviews it. Releases are cut by maintainers ([Updates and releases](06-updates-and-releases.md#releasing-a-new-version-maintainers)).

By contributing you agree that your contribution is licensed under the MIT licence.

## Reporting a security issue

**Do not open a public issue for a vulnerability.** Use GitHub's private vulnerability reporting on
[ecom-core](https://github.com/SetWebUK/ecom-core/security) (*Security › Report a vulnerability*). If that option is
not available, open an issue that says only that you have a security report and ask a maintainer for a private
contact – no details. Please include the affected version, steps to reproduce and the impact; give maintainers a
reasonable time to release a fix before disclosing.

## Documentation contributions

- **This hub** (overview and guides): edit the Markdown here and open a pull request. Keep it plain English, short
  paragraphs, copy-pasteable commands, and link to ecom-core's reference docs with full GitHub URLs.
- **Reference docs** (`docs/*.md` in ecom-core): change them in the same pull request as the code they describe.
- Check that every command and option you document exists (`php artisan list commerce`, `php artisan help <command>`)
  and that relative links resolve.
- No shop names, real host names, server paths, personal details or secrets – use `acme` and `example.test`.
