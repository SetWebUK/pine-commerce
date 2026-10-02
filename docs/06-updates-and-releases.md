# 6. Updates and releases

How shops receive platform updates, and how maintainers release them.

Reference: [UPGRADING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/UPGRADING.md) and Playbook
[part 3](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md#part-3--day-to-day-releasing-and-rolling-out-updates).

**Contents**

- [Versioning](#versioning)
- [How a shop is wired to the platform](#how-a-shop-is-wired-to-the-platform)
- [Updating a shop from the admin](#updating-a-shop-from-the-admin)
- [Updating from the command line](#updating-from-the-command-line)
- [Updating by hand with Composer](#updating-by-hand-with-composer)
- [Skeleton file updates](#skeleton-file-updates)
- [What to read in the release notes](#what-to-read-in-the-release-notes)
- [Backups and rollback](#backups-and-rollback)
- [Releasing a new version (maintainers)](#releasing-a-new-version-maintainers)
- [Hotfixes](#hotfixes)

---

## Versioning

`pine/commerce` uses **Semantic Versioning**, tagged `vMAJOR.MINOR.PATCH` in
[ecom-core](https://github.com/SetWebUK/ecom-core/tags). There is no `version` key in `composer.json`: Composer reads
versions from the tags.

| Release | Contains |
|---|---|
| **Patch** (1.3.x) | bug fixes only |
| **Minor** (1.x.0) | additions that keep current behaviour: new config keys with safe defaults, optional theme views, new events, commands and options, **additive** migrations |
| **Major** (x.0.0) | anything that removes or renames part of the public API |

The **public API** is: the theme contract (view names, their variables, `supports` keys, `theme.json` keys), config
keys and environment variables, the `Commerce::` extension API, `ThemeDefinition` hooks, events and their payloads,
importer interfaces and command options, route names and URLs, cookie and header names, and the database schema.
Tables and columns are never renamed or dropped – shops hold live data in them.

The starter app ([ecom-skeleton](https://github.com/SetWebUK/ecom-skeleton/tags)) is tagged with the core version it
was exported from.

## How a shop is wired to the platform

A shop's `composer.json` requires the package from the public GitHub repository:

```json
"repositories": [{ "type": "vcs", "url": "https://github.com/SetWebUK/ecom-core.git" }],
"require": { "pine/commerce": "^1.3" }
```

`composer.lock` pins the exact release. The caret constraint decides what an update may install: `^1.3` takes 1.4.0
but never 2.0.0. (During platform development you can point a shop at a local checkout with a `path` repository and
`"pine/commerce": "*@dev"` – never commit that.)

## Updating a shop from the admin

*Since 1.3. Administrators only; feature flag `updater` (on by default).*

**Finding updates.** Once a day (scheduled task `updates.check`) and whenever someone presses **Check now**, the shop
reads the release tags of the repository in its `composer.json`. It offers the newest stable release **that the
Composer constraint allows**, shows every changelog section in between with **Client actions required** highlighted,
and puts a notice on the dashboard and a badge on the sidebar. A newer major version (or anything outside the
constraint) is shown as "requires a developer".

**Installing.** The administrator presses **Approve & install**, re-enters their password and confirms. The update
runs in the background (no queue worker needed) while the page follows its log:

1. Pre-flight checks: PHP CLI and Composer found, free disk space, writable `vendor/`, `storage/`, `bootstrap/cache`,
   `public/`, a warning for uncommitted Git changes, `commerce:doctor` before.
2. Database backup (`mysqldump` + gzip, or a copy of the SQLite file) in `storage/app/private/updater/backups/`
   (newest 5 kept).
3. `composer.json` and `composer.lock` recorded.
4. Maintenance mode (`php artisan down` with a secret; the approving administrator gets the bypass cookie).
5. `composer update pine/commerce --with-dependencies --with=pine/commerce:X.Y.Z` – pinned to the approved version.
6. `migrate --force` → `commerce:publish` → `commerce:theme:publish` → `optimize:clear` + `optimize`.
7. Health check: a `commerce:doctor` check that fails now but did not fail before fails the update.
8. `php artisan up`.

**If a step fails**, the updater restores `composer.json` / `composer.lock`, runs `composer install` (the previous code
comes back), republishes assets, rebuilds caches and brings the site back up. The run is marked *failed* with its
full log. The database is **not** restored automatically – migrations are additive, so the previous version keeps
working; the log prints the restore command in case you need it.

Server settings for the updater (PHP and Composer binaries, HOME, backup path, timeouts, a GitHub token for rate
limits) live in the `updater` block of `config/commerce.php` and `COMMERCE_UPDATER_*` / `COMMERCE_PHP_BINARY` /
`COMMERCE_COMPOSER_BINARY` environment variables – see
[EXTENDING.md "Updates"](https://github.com/SetWebUK/ecom-core/blob/main/docs/EXTENDING.md#updates).

## Updating from the command line

Same steps, same audit log:

```bash
php artisan commerce:update:check                  # what is available, changelog, client actions (never installs)
php artisan commerce:update:check --json
php artisan commerce:update:run --approve --yes    # approve the newest installable release as "CLI" and install it now
php artisan commerce:update:run 12                 # run update #12 that was approved in the admin
```

`commerce:update:run` without an approved id refuses to run. The second form is the fallback when the web server
cannot start background processes (for example `proc_open` disabled).

## Updating by hand with Composer

Staging first, then live. On the shop project (never a blanket `composer update` on a live site):

```bash
mysqldump --single-transaction acme_shop | gzip > ~/backups/acme_shop-$(date +%Y%m%d-%H%M)-pre-update.sql.gz
composer update pine/commerce
git diff composer.lock                   # shows the version change
php artisan migrate --force
php artisan commerce:publish
php artisan commerce:theme:publish
php artisan optimize:clear && php artisan optimize
php artisan commerce:doctor
composer test
git commit -am "pine/commerce 1.3.1" && git push
```

On the shop's other servers: `git pull && composer install --no-dev --optimize-autoloader && composer deploy`.

## Skeleton file updates

Files a project received from the starter app (`bootstrap/app.php`, `config/*.php` stubs, `tests/TestCase.php` …) are
updated separately from the core. `commerce:new-client` records the skeleton release and a hash of every file it wrote
in `.commerce-skeleton.json`; projects made before 1.3 record it once:

```bash
php artisan commerce:skeleton:baseline v1.3.0          # record which skeleton release the project matches
php artisan commerce:skeleton:baseline --detect        # or compare with recent releases to find it
php artisan commerce:skeleton:check                    # compare with the newest skeleton release
php artisan commerce:skeleton:check --apply-safe --yes # apply the files this project never changed
```

Each changed file is classified: **safe** (new, or unchanged here – can be applied, with a backup), **needs a developer**
(changed both here and upstream; the diff is shown), or **never updated automatically** (`.env*`, `composer.lock`,
`themes/`, `README.md`, `LICENSE`, storage data, published assets). The same is available in Admin › Updates ›
*Project files from the skeleton*. Update the core first, then the skeleton files.

## What to read in the release notes

Every [CHANGELOG](https://github.com/SetWebUK/ecom-core/blob/main/CHANGELOG.md) entry lists config keys added,
theme-contract changes and **client actions required**. In particular:

- **Config keys** – your `config/commerce.php` replaces whole top-level blocks (except `features`). If you override a
  block that gained a key, copy the new key into your file.
- **Theme contract** – run `php artisan commerce:theme:check` and add any newly required views.
- **Admin view overrides** in `resources/views/vendor/commerce/…` – diff them against the new core views.

## Backups and rollback

- **Before every update, import or migration**, take a dated database dump outside the web root (the admin updater
  does this for you). Restore only into a database you mean to overwrite:

  ```bash
  gunzip -c ~/backups/acme_shop-YYYYMMDD-HHMM-pre-update.sql.gz | mysql acme_shop
  ```

- **Code rollback**: pin the previous version and deploy:

  ```bash
  composer require pine/commerce:1.3.0
  composer deploy
  git commit -am "Roll back pine/commerce to 1.3.0" && git push
  ```

- **Data rollback** only when nothing new has been written since the backup; otherwise repair forward.
- Shops with a like-for-like theme keep a **regression check** (snapshot every page type, diff after the update) –
  zero unexplained differences before an update goes live.

## Releasing a new version (maintainers)

All platform work happens in **ecom-core**. A release is an annotated tag on `main`:

```bash
cd ecom-core && git switch main && git pull
# 1. CHANGELOG.md: rename "## [Unreleased]" to "## [1.4.0] - YYYY-MM-DD", add a fresh empty [Unreleased],
#    and list config keys added, contract changes and "Client actions required"
# 2. the same number in VERSION and in src/Commerce.php (const VERSION) – a test checks all three agree
composer test
composer validate --strict
git commit -am "Release v1.4.0"
git tag -a v1.4.0 -m "pine/commerce v1.4.0"
git push origin main v1.4.0
```

Then refresh the starter app from the same clone and **tag it with the same version** (skeleton baselines and Admin ›
Updates refer to skeleton tags):

```bash
bin/export-client-skeleton.sh ../ecom-skeleton --tag --push=git@github.com:SetWebUK/ecom-skeleton.git
```

`bin/export-client-skeleton.sh <out-dir> [--repo=] [--constraint=] [--push=<url>] [--tag] [--name=] [--slug=]`
renders `stubs/client-skeleton` into the skeleton repository, commits the difference and optionally tags and pushes.
Optionally create a GitHub release from the tag with the CHANGELOG section as notes. Shops then see the release in
Admin › Updates within a day.

## Hotfixes

```bash
git switch -c hotfix/1.3.2 v1.3.1          # branch from the tag shops run
# … the smallest possible fix + a test …
composer test
# CHANGELOG [1.3.2] + VERSION + Commerce::VERSION, commit "Release v1.3.2"
git tag -a v1.3.2 -m "pine/commerce v1.3.2" && git push origin hotfix/1.3.2 v1.3.2
git switch main && git merge --no-ff hotfix/1.3.2 && git push   # the fix must reach main too
```
