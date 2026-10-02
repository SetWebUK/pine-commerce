# Pine Commerce documentation

Start with the [project README](../README.md) for the overview and the 5-minute quick start, then follow the guides in
order.

## Guides (this repository)

| # | Guide | Covers |
|---|---|---|
| 1 | [Getting started](01-getting-started.md) | requirements, install from the starter app (SQLite or MySQL), `commerce:install`, first login, store settings, `commerce:doctor`, serving locally, hosting, cron, deploys, go-live checklist |
| 2 | [Migrating from WooCommerce](02-migrating-from-woocommerce.md) | the importer end to end: connecting to the old site, detection, dry runs, scratch rehearsals, media, URL parity, redirects, SEO, adapters, final import |
| 3 | [Themes and templating](03-themes-and-templating.md) | theme structure, `theme.json`, view lookup order, the theme contract, child themes, assets, images, helpers, emails and PDFs, like-for-like rebuilds |
| 4 | [Building features](04-building-features.md) | the `Commerce::` extension API with worked examples (payment gateway, admin page + menu + settings + migration, scheduled job), events, feature flags, testing |
| 5 | [Admin guide](05-admin-guide.md) | a tour of the back office for shop managers |
| 6 | [Updates and releases](06-updates-and-releases.md) | versioning, Admin › Updates, CLI updates, skeleton file updates, rollback, cutting a release |
| 7 | [Contributing](07-contributing.md) | repository layout, conventions, tests, pull requests, security reports, docs |
| – | [FAQ](faq.md) | quick answers |

## Reference (ecom-core repository)

The detailed reference lives next to the code, so it always matches the release you run:

| Document | For |
|---|---|
| [PLAYBOOK.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md) | the operational runbook: repositories, onboarding a WooCommerce shop step by step, releases, data safety, troubleshooting, command and config reference |
| [ARCHITECTURE.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/ARCHITECTURE.md) | the binding design contract: layers, theme contract (every view and variable), config keys, extension points, importer |
| [THEMES.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/THEMES.md) | writing and forking themes, images |
| [EXTENDING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/EXTENDING.md) | feature flags and every extension point |
| [ADMIN_UI.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/ADMIN_UI.md) | back-office components and conventions |
| [IMPORTER.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/IMPORTER.md) | the WordPress/WooCommerce importer in depth |
| [TAX-AND-SHIPPING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/TAX-AND-SHIPPING.md) | tax settings, classes and rates; shipping zones and method types |
| [INVOICES.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/INVOICES.md) | invoice numbers, PDF invoices and packing slips, template overrides |
| [PRODUCT-CSV.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/PRODUCT-CSV.md) | product CSV import and export |
| [UPGRADING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/UPGRADING.md) | SemVer rules, what counts as breaking, upgrading shops |
| [CHANGELOG.md](https://github.com/SetWebUK/ecom-core/blob/main/CHANGELOG.md) | release history with client actions |

## Command cheat sheet

| Command | What it does |
|---|---|
| `commerce:install` | migrate, first admin, default settings/pages/menus/zone/VAT rates, `public/storage`, publish assets (idempotent) |
| `commerce:doctor` | health check with fixes (`--only-problems`, `--json`) |
| `commerce:new-client {path}` | create a named shop project from the skeleton (`--repo=`, `--constraint=`) |
| `commerce:publish` | copy the admin assets into `public/` |
| `commerce:theme:make {slug}` | scaffold a theme (`--parent=default`, `--copy`) |
| `commerce:theme:publish` / `:check` / `:cache` / `:clear` | theme assets, contract validation, manifest cache |
| `commerce:import-wordpress` | import a WordPress/WooCommerce shop (`--detect`, `--dry-run`, `--copy-uploads`, `--only=` …) |
| `commerce:verify-urls {source}` | every old URL must answer 200 or 301 → 200 |
| `commerce:images:generate` | build image sizes and WebP twins for existing media (`--missing`) |
| `commerce:products:export` / `commerce:products:import {file}` | product CSV out / in (`--dry-run`) |
| `commerce:schedule:status` / `commerce:schedule:task {task}` | scheduled tasks: status / run one now |
| `commerce:update:check` / `commerce:update:run` | find / install platform updates |
| `commerce:skeleton:baseline` / `commerce:skeleton:check` | record / compare / apply starter-app file updates |
| `commerce:scratch:drop` | drop the `zz_` rehearsal tables |

`php artisan list commerce` shows them all; `php artisan help <command>` shows every option.
