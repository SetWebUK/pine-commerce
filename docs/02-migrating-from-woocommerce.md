# 2. Migrating from WooCommerce

How to move a WordPress/WooCommerce shop onto Pine Commerce end to end: products, customers (with their existing
passwords), orders, content, menus, redirects, SEO and media – while every old URL keeps working.

This guide is the overview. The step-by-step runbook is part 2 of the
[Playbook](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md), and every importer option, adapter and
config key is in [IMPORTER.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/IMPORTER.md).

**Contents**

- [How the importer works](#how-the-importer-works)
- [The onboarding flow at a glance](#the-onboarding-flow-at-a-glance)
- [1. Collect before you start](#1-collect-before-you-start)
- [2. Create and install the new shop](#2-create-and-install-the-new-shop)
- [3. Point the importer at the old site](#3-point-the-importer-at-the-old-site)
- [4. Detect](#4-detect)
- [5. Rehearse, dry-run, import](#5-rehearse-dry-run-import)
- [6. Media and image sizes](#6-media-and-image-sizes)
- [7. URL parity and redirects](#7-url-parity-and-redirects)
- [8. SEO](#8-seo)
- [9. Review in the admin](#9-review-in-the-admin)
- [10. Client-specific data: adapters and config](#10-client-specific-data-adapters-and-config)
- [11. Final import and go-live](#11-final-import-and-go-live)
- [Troubleshooting](#troubleshooting)

---

## How the importer works

`php artisan commerce:import-wordpress` (alias `import:wordpress`) is a **one-way, repeatable** import:

- **Read-only source.** The old database is opened on the `wordpress` connection, which forces read-only sessions.
  Nothing is ever written to the old site.
- **Idempotent.** Every row is matched on its WordPress id (or a natural key) and updated in place. Import early,
  build the theme, re-import as often as you like; the final run before go-live just brings the delta.
- **Generic and detecting.** It reads the site profile (WordPress/WooCommerce versions, table prefix, HPOS or posts
  order storage, permalinks, active plugins) and switches on matching **adapters** – Yoast, Rank Math, Redirection,
  Elementor, WPBakery, ACF, sequential order numbers, product brands, wishlists, back-in-stock plugins, Contact Form
  DB and more.
- **Extensible.** Anything bespoke on the old site is handled by small client adapters in your project
  (`app/Import/…`), never by editing the core.

What it brings across, in order: settings → media → users (customers keep their WordPress password hashes and can sign
in as before) → categories → attributes → products → variations → brands/tags → orders (with refunds and notes) →
reviews → coupons → shipping zones → tax classes and rates → extras (stock alerts, wishlists, form submissions) →
pages → posts → menus → redirects.

## The onboarding flow at a glance

```
collect access ─► create project ─► commerce:install ─► backup ─► point at old site ─► --detect
      ─► scratch rehearsal / --dry-run ─► import --copy-uploads ─► images:generate ─► verify-urls
      ─► fix (redirects, config, adapters) ─► theme ─► payments, mail, SEO ─► QA
      ─► content freeze ─► final import ─► go live ─► verify-urls on the live host
```

## 1. Collect before you start

- [ ] Hosting for the new shop (see [Getting started](01-getting-started.md#requirements)).
- [ ] Access to the old site: a **copy** of its database (or read-only credentials), `wp-config.php`, and
      `wp-content/uploads`.
- [ ] The old sitemap, **saved to a file before anything changes**:
      ```bash
      curl -s https://www.example.test/sitemap_index.xml -o storage/app/old-sitemap.xml   # Yoast / Rank Math
      # core WordPress: /wp-sitemap.xml
      ```
- [ ] Every host name the old site used (live, `www` / non-`www`, old staging hosts).
- [ ] Payment accounts (Stripe, PayPal, bank details) and mail provider credentials.
- [ ] Decisions: which theme approach ([Themes](03-themes-and-templating.md#choosing-an-approach)), go-live date,
      content freeze for the final import.

## 2. Create and install the new shop

```bash
php artisan commerce:new-client ~/projects/acme --name="Acme Tools" --slug=acme \
    --repo=https://github.com/SetWebUK/ecom-core.git --constraint=^1.3
cd ~/projects/acme && composer install && php artisan key:generate --force
# fill in .env (APP_URL, DB_*, MAIL_MAILER=log for now)
php artisan commerce:install --admin-email=you@example.test --store-name="Acme Tools"
```

Then **back up** the new, empty shop database. Back up again before every import:

```bash
mkdir -p ~/backups
mysqldump --single-transaction --routines acme_shop | gzip > ~/backups/acme_shop-$(date +%Y%m%d-%H%M)-pre-import.sql.gz
```

## 3. Point the importer at the old site

**WordPress files on the same server** – the importer *parses* `wp-config.php` (it never executes it) for the
database credentials and table prefix:

```dotenv
# .env
WP_PATH=/path/to/old/wordpress
```

or per run: `--wp-path=/path/to/old/wordpress`. Values from `wp-config.php` win over `WP_DB_*`; `--db-*` options win
over both.

**Database elsewhere** – import a dump of the old database into a separate database on this server (safest), or use
a read-only user on the old host:

```dotenv
WP_DB_HOST=127.0.0.1
WP_DB_PORT=3306
WP_DB_DATABASE=acme_wp
WP_DB_USERNAME=readonly
WP_DB_PASSWORD=secret
WP_DB_PREFIX=wp_
```

or per run: `--db-host= --db-port= --db-name= --db-user= --db-pass= [--db-socket=] [--prefix=wp_]`.

Copy the media across too (read-only on the old server), then pass the directory with `--uploads-path=`:

```bash
rsync -az olduser@old-host.example.test:/path/to/wordpress/wp-content/uploads/ ~/wp-uploads/
```

**Page-builder content** (Elementor, WPBakery): the importer reads the *rendered* page when it can. Give it a
running copy of the old site (`WP_SITE_URL`, plus `WP_SITE_HOST` when reached by IP) or a directory of saved HTML
snapshots (`--snapshots=storage/app/wp-reference/html`, one file per URL path with `/` replaced by `__`,
`home.html` for the front page).

List every old host name in **both** `config/commerce-import.php` → `legacy_hosts` and `config/commerce.php` →
`legacy_hosts`, so links and redirect targets on those hosts become relative URLs.

## 4. Detect

```bash
php artisan commerce:import-wordpress --detect
```

Shows the WordPress and WooCommerce versions, table prefix, order storage (HPOS or posts), permalink settings, active
plugins, every adapter (active or not, priority, capabilities), the step order, and where each credential came from.
Fix credentials until it is clean.

## 5. Rehearse, dry-run, import

For a first or unusual site, rehearse on the **scratch** connection – the same database with `zz_`-prefixed tables:

```bash
php artisan commerce:install --connection=scratch --admin-email=you@example.test --no-interaction
php artisan commerce:import-wordpress --target=scratch --copy-uploads
# compare counts and spot-check, adjust config/adapters, repeat …
php artisan commerce:scratch:drop --force
```

Then the real thing:

```bash
php artisan commerce:import-wordpress --dry-run           # everything read and transformed, all writes rolled back
php artisan commerce:import-wordpress --copy-uploads      # the import, plus media copied into public/storage/uploads
php artisan commerce:import-wordpress --only=catalog,content   # re-run just some sections later
```

Useful options (full table in [IMPORTER.md §2](https://github.com/SetWebUK/ecom-core/blob/main/docs/IMPORTER.md#2-options)):

| Option | Use |
|---|---|
| `--only=` / `--skip=` | sections (`settings, media, users, catalog, orders, extras, content, menus, redirects`), step keys (`catalog.products`) or aliases (`products, customers, coupons, pages, posts, …`) |
| `--dry-run` | run everything in one transaction and roll it back |
| `--target=` | import into another connection (`scratch`) |
| `--orders-source=auto\|hpos\|posts` | force the order storage |
| `--site-url=`, `--site-host=`, `--snapshots=` | rendered page-builder content |
| `--uploads-path=` | a copied `uploads/` directory |
| `--core-only` | ignore your client config and adapters (shows what they add) |
| `--no-wp-cli` | do not ask wp-cli for permalinks |
| `--fresh` | purge the selected steps' imported rows first (**needs `--force` on a database with data – avoid on live**) |

The summary (WordPress count vs imported count, plus warnings) is printed, written to
`storage/logs/import-wordpress.log`, and shown in **Admin › Settings › System**. Each step runs in its own transaction:
a failing step rolls back only itself – fix it and re-run with `--only=<step>`.

## 6. Media and image sizes

`--copy-uploads` copies `wp-content/uploads` into `public/storage/uploads` as **real files** (never symlinks).
Executable files are quarantined, WooCommerce private downloads and logs go to private storage, cache folders are
skipped, same-size files are skipped as already verified.

Imported media keeps the sizes WordPress generated. Add the platform's own sizes and WebP twins once the media is in
place – it only ever *adds* files:

```bash
php artisan commerce:images:generate --missing --dry-run
php artisan commerce:images:generate --missing
php artisan commerce:images:generate --missing --scan     # also files with no media-library row
```

## 7. URL parity and redirects

Pine Commerce serves WordPress-style URLs with trailing slashes: categories at `/{category path}/`, products at
`/{primary category path}/{slug}/`, posts at `/blog/{slug}/`, pages at their path. The importer works out the old
site's **real** URLs (wp-cli, Permalink Manager, Premmerce, or the permalink settings) and generates redirects where
the new URL differs. Redirects from the Redirection plugin, Rank Math, Yoast Premium and `_wp_old_slug` are imported.
Old `/product/{slug}/`, `/product-category/{path}/` and `?add-to-cart=` links are handled by the storefront itself.

Prove it – every old URL must answer **200**, or **301 to a 200**:

```bash
php artisan commerce:verify-urls storage/app/old-sitemap.xml \
    --base=https://staging.example.test --report=storage/app/url-parity.csv
```

| Option | Use |
|---|---|
| `{source}` | sitemap URL, sitemap file, or a text file with one URL/path per line |
| `--base=` | the new site (default `APP_URL`) |
| `--resolve=<ip>` | test the new server under the live host name before DNS moves |
| `--report=<csv>` | write every result to a CSV |
| `--allow-302`, `--no-follow`, `--concurrency=8`, `--timeout=20`, `--limit=`, `--insecure` | tuning |

Fix failures with redirects (**Admin › Content › Redirects**, CSV import available) or importer config, and re-run
until it exits 0.

## 8. SEO

- Titles, meta descriptions, canonicals and robots come from Yoast or Rank Math (variables converted). Where the
  plugin generated a value, the importer uses the rendered `<title>` / meta description from the old page.
- Primary categories follow the SEO plugin, so product URLs and breadcrumbs match.
- Spot-check titles, descriptions, canonicals and Open Graph against the old pages; check `/sitemap.xml`,
  `/robots.txt`, structured data (Google Rich Results Test) and the Google Shopping feed if used.
- Google Tag Manager / GA4 ids: **Admin › Settings › SEO & tracking**.

## 9. Review in the admin

- [ ] Products: prices, sale prices, stock, variations, images, categories – compare ten against the old site.
- [ ] Customers can sign in with their old passwords.
- [ ] Order numbers continue the old sequence (Settings › Checkout & orders › lowest order number).
- [ ] Pages and posts: content and SEO; delete or redirect starter pages that the imported ones replace
      (`/about` next to `/about-us`).
- [ ] Menus: WordPress menus without a theme location arrive as `wp-{slug}`; map them with `menus.by_term_id` in
      `config/commerce-import.php` and re-run `--only=menus`, or assign them in Admin › Content › Menus.
- [ ] Tax rates and shipping zones (imported from WooCommerce) give the same totals as the old shop.

## 10. Client-specific data: adapters and config

Simple mappings are configuration in `config/commerce-import.php` (copy the whole top-level block you change – the
merge is shallow):

| Key | For |
|---|---|
| `legacy_hosts` | old host names whose links become relative |
| `content.replace` | ordered search → replace on imported content |
| `content.page_templates`, `content.selectors` | page templates; where the content sits in rendered pages |
| `attributes.map`, `attributes.filterable` | attributes → product columns; which attributes become shop filters |
| `menus.by_term_id`, `menus.by_location` | WordPress menus → theme menu locations |
| `acf.product_fields`, `acf.term_fields` | ACF fields → product / category columns |
| `settings.defaults`, `settings.seed` | settings precedence |
| `adapters.disable`, `adapters.enable` | switch built-in adapters (or one capability, e.g. `rank-math:redirects`) off or on |

Anything the generic importer does not know (custom meta, a bespoke page builder, menus that only exist in the
rendered HTML, a delivery method hard-coded in the old theme) gets a small **adapter** in your project:

```php
// app/Import/Acme/AcmeCatalogAdapter.php
namespace App\Import\Acme;

use Pine\Commerce\Import\Adapters\AbstractAdapter;
use Pine\Commerce\Import\Contracts\ProductMapper;
use Pine\Commerce\Import\Data\WpProduct;
use Pine\Commerce\Import\ImportContext;
use Pine\Commerce\Import\Source\SiteProfile;
use Pine\Commerce\Import\Source\WordPressSource;

class AcmeCatalogAdapter extends AbstractAdapter implements ProductMapper
{
    public function key(): string { return 'acme-catalog'; }
    public function priority(): int { return -10; }            // negative: runs after the core mappers (last word)
    public function detect(SiteProfile $site, WordPressSource $wp): bool { return true; }

    public function mapProduct(array $row, WpProduct $product, ImportContext $ctx): array
    {
        $row['subtitle'] = $product->meta['_acme_strapline'] ?? null;   // any products column
        return $row;
    }

    public function specRows(WpProduct $product, ImportContext $ctx): ?array
    {
        return null;
    }
}
```

Register it in `app/Providers/ClientServiceProvider.php`:

```php
\Pine\Commerce\Commerce::importAdapter(\App\Import\Acme\AcmeCatalogAdapter::class);
\Pine\Commerce\Commerce::importStep(\App\Import\Acme\LoyaltyPointsStep::class);   // a step that always runs
```

Other capabilities an adapter can implement: `SettingsProvider`, `MenuProvider`, `RedirectProvider`, `SeoProvider`,
`PermalinkProvider`, `RenderedContentProvider`, `ContentTransformer`, `TermMapper`, `OrderNumberProvider`,
`BreadcrumbTermProvider` – see [IMPORTER.md §9](https://github.com/SetWebUK/ecom-core/blob/main/docs/IMPORTER.md#9-writing-a-client-adapter).
`--detect` lists your adapters next to the built-in ones. Rehearse on `scratch`, then re-import.

## 11. Final import and go-live

1. Content freeze on the old site (no new orders or edits).
2. Back up the new shop database (and keep the old site's files and database dump for at least three months).
3. Final import (the delta): `php artisan commerce:import-wordpress --copy-uploads`.
4. Switch `.env` to the live URL, live mail and live payment keys; `php artisan optimize:clear && php artisan optimize`.
5. Before DNS moves, test the new server under the live name:
   `php artisan commerce:verify-urls storage/app/old-sitemap.xml --base=https://www.example.test --resolve=<new server IP>`.
6. Move DNS (lower the TTL a day before), untick "discourage search engines", submit the sitemap in Search Console,
   and watch 404s for two weeks.
7. Disable the old site's mail and cron so it stops emailing customers.

Decide a **rollback plan** before go-live (DNS back to the old server; export orders taken in the meantime from
Admin › Orders › Export). Details: Playbook part 2, steps 14–15.

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Cannot open the WordPress database" | check `WP_PATH` / `WP_DB_*`; `--detect` shows where each credential came from; non-literal values in `wp-config.php` need `--db-*` |
| "Several WordPress installs … pass --prefix" | the database holds more than one install: pass `--prefix=wp_` |
| Page bodies empty or full of shortcodes | page-builder content needs a rendered copy: `--site-url` (+ `--site-host`) or `--snapshots` |
| Media missing | run with `--copy-uploads` (or `--uploads-path=`) |
| Product/category URLs differ from the old site | `--detect` shows the permalink provider; add a `PermalinkProvider` adapter or rely on the generated redirects, then `commerce:verify-urls` |
| Links or redirects still point at the old host | add the host to `legacy_hosts` in both config files, re-run `--only=content,menus,redirects` |
| Wrong order numbers | check `--detect` for `sequential-order-numbers`; add an `OrderNumberProvider` for other plugins |
| Images load the full-size original | `php artisan commerce:images:generate --missing` |

More: [IMPORTER.md §11](https://github.com/SetWebUK/ecom-core/blob/main/docs/IMPORTER.md#11-troubleshooting) and
[Playbook part 5](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md#part-5--troubleshooting).

---

Next: [Themes and templating](03-themes-and-templating.md).
