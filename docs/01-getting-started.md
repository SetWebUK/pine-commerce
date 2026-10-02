# 1. Getting started

From nothing to a running store, then onto a real server. Commands are copy-pasteable; replace `acme`,
`example.test` and passwords with your own values.

**Contents**

- [Requirements](#requirements)
- [Install from the starter app](#install-from-the-starter-app)
  - [Option A: SQLite (local development)](#option-a-sqlite-local-development)
  - [Option B: MySQL / MariaDB (staging and live)](#option-b-mysql--mariadb-staging-and-live)
  - [Option C: a named project with `commerce:new-client`](#option-c-a-named-project-with-commercenew-client)
- [`commerce:install` in detail](#commerceinstall-in-detail)
- [First login and store settings](#first-login-and-store-settings)
- [The health check: `commerce:doctor`](#the-health-check-commercedoctor)
- [Serving locally](#serving-locally)
- [Hosting notes](#hosting-notes)
- [Scheduled tasks (cron, or no cron)](#scheduled-tasks-cron-or-no-cron)
- [Deploying changes](#deploying-changes)
- [Going live checklist](#going-live-checklist)

---

## Requirements

| | |
|---|---|
| PHP | 8.3+ on **both** the CLI and the web server. Extensions: `pdo_mysql` (or `pdo_sqlite`), `mbstring`, `intl`, `gd` or `imagick`, `curl`, `zip`, `bcmath`, `fileinfo`, `openssl`, `tokenizer`, `xml` |
| Database | MySQL 8 / MariaDB 10.6+ (SQLite is fine for local work and tests) |
| Tools | Composer 2, Git, SSH access to the server |
| Not needed | Node.js, a queue worker, Redis, Supervisor |

Quick check:

```bash
php -v
php -m | grep -iE 'pdo_mysql|pdo_sqlite|intl|mbstring|gd|imagick|zip|bcmath'
composer --version
```

## Install from the starter app

The starter app ([ecom-skeleton](https://github.com/SetWebUK/ecom-skeleton)) is a plain Laravel 13 project whose
`composer.json` pulls in `pine/commerce` from GitHub. The core repository is public, so Composer needs no credentials.

```bash
git clone https://github.com/SetWebUK/ecom-skeleton.git acme-shop && cd acme-shop
composer install
cp .env.example .env
php artisan key:generate
```

`.env.example` documents every key. **Never change `APP_KEY` after go-live** – it encrypts the payment keys stored
in the database.

### Option A: SQLite (local development)

```bash
touch database/database.sqlite
```

In `.env`:

```dotenv
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/acme-shop/database/database.sqlite
APP_URL=http://127.0.0.1:8000
SESSION_SECURE_COOKIE=false     # true (the default) would drop the session over plain http: admin login bounces back
LOG_LEVEL=debug                 # so emails written by MAIL_MAILER=log actually reach storage/logs
```

Remove (or ignore) the `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD` lines.

### Option B: MySQL / MariaDB (staging and live)

Create an empty database and a user (or use your hosting panel's *MySQL Databases* screen):

```sql
CREATE DATABASE acme_shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'acme_shop'@'localhost' IDENTIFIED BY 'a-long-random-password';
GRANT ALL PRIVILEGES ON acme_shop.* TO 'acme_shop'@'localhost';
```

In `.env`:

```dotenv
APP_NAME="Acme Tools"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://shop.example.test          # https, no trailing slash
DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=acme_shop
DB_USERNAME=acme_shop
DB_PASSWORD=a-long-random-password
SESSION_SECURE_COOKIE=true
MAIL_MAILER=log                            # switch to smtp when you are ready to send real mail
```

The shop tables are never prefixed. The skeleton also defines a `scratch` connection (same database, `zz_` table
prefix) used for importer rehearsals, and a read-only `wordpress` connection used by the importer.

### Option C: a named project with `commerce:new-client`

When you start a real shop, let the platform name everything for you (Composer package name, `.env.example`,
README, theme slug) and record which skeleton release it came from:

```bash
# from any checkout that already has pine/commerce installed (e.g. the skeleton clone above)
php artisan commerce:new-client ~/projects/acme --name="Acme Tools" --slug=acme \
    --repo=https://github.com/SetWebUK/ecom-core.git --constraint=^1.3
cd ~/projects/acme && composer install && php artisan key:generate --force
```

| Option | Meaning |
|---|---|
| `{path}` | where to create the project |
| `--name=`, `--slug=` | shop name and slug (`acme`) |
| `--repo=<git url>` | require `pine/commerce` from a VCS repository – the normal choice for every committed project |
| `--path=<dir>` | require it from a local checkout of ecom-core instead (symlinked; for platform development only, never commit it) |
| `--constraint=` | version constraint, e.g. `^1.3` |
| `--force` | write into a non-empty directory |

Then put the project in its own (usually private) Git repository.

## `commerce:install` in detail

```bash
php artisan commerce:install --admin-email=you@example.test --store-name="Acme Tools"
```

It is **idempotent** – it only creates what is missing, so it is safe to run again. It:

1. runs the migrations;
2. creates the first administrator (asks for a password, or generates and prints one when non-interactive);
3. seeds default settings (search engines are discouraged until you go live), core pages (Home, About us, Contact
   us, Terms and conditions, Privacy policy) and menus;
4. creates a UK shipping zone with free delivery and UK VAT rates (20 % / 5 % / 0 %);
5. creates `public/storage` as a **real directory** with a hardening `.htaccess` (never a symlink);
6. publishes the admin and theme assets (copies) and runs `optimize`.

| Option | Use |
|---|---|
| `--admin-email=`, `--admin-name=`, `--admin-password=` | the first administrator |
| `--force-admin` | create/promote the administrator even when one exists (resets that user's password) |
| `--store-name=`, `--store-email=` | store details (defaults: `APP_NAME`, the admin email) |
| `--theme=` | storefront theme slug – written to `COMMERCE_THEME` in `.env` |
| `--order-start=` | first order number (default 1000) |
| `--connection=` | install into another connection (e.g. `scratch` for an importer rehearsal) |
| `--skip-publish`, `--no-optimize` | skip asset publishing / the optimize step |

## First login and store settings

Sign in at `/admin` (you are sent to `/admin/login`) with the administrator you just created. Then work through
**Admin › Settings**:

| Screen | Set |
|---|---|
| Store details | name, logo, favicon, contact details, address, company/VAT numbers, social links, announcement bar |
| Checkout & orders | guest checkout, account creation, terms page, lowest order number |
| Tax | prices entered including or excluding VAT, what the shop and basket show, rate table per tax class |
| Payments (administrators) | Stripe, PayPal, bank transfer – keys are stored encrypted, never in `.env` |
| Shipping | countries you sell to, zones and their delivery options |
| Emails | sender name, order-notification recipients, which order emails are sent |
| Invoices | numbering, PDF attachments, customer downloads |
| SEO & tracking | default titles/descriptions, robots.txt, "discourage search engines", Google Tag Manager / GA4 |
| Theme | logo, colours, fonts, corner style of the active theme |
| Staff accounts (administrators) | administrators and managers |
| System (administrators) | version, active theme, feature flags, health checks, last import report |

Stripe and PayPal webhooks point at `{APP_URL}/webhooks/stripe` and `{APP_URL}/webhooks/paypal`; the payments screen
shows the exact URL and the events to subscribe to.

## The health check: `commerce:doctor`

```bash
php artisan commerce:doctor                  # every check: PASS / WARN / FAIL / INFO, each with its fix
php artisan commerce:doctor --only-problems  # just what needs attention
php artisan commerce:doctor --json           # for scripts; exits 1 when anything FAILs
```

It checks the environment (PHP, extensions, `APP_URL`, debug mode), database, `public/storage`, published assets,
the theme, mail, payment gateways, the scheduler and more. The same checks appear in **Admin › Settings › System**.

## Serving locally

```bash
php artisan serve                      # from the project root
```

or PHP's built-in server **started from `public/`** with Laravel's router script:

```bash
cd public && php -S 127.0.0.1:8000 ../vendor/laravel/framework/src/Illuminate/Foundation/resources/server.php
```

Run it from `public/`: the router loads `index.php` from the current directory, so started from the project root
every page is an empty 500. Without the router script `php -S` treats `/sitemap.xml`, `/robots.txt` and the feeds
as static files and answers 404.

## Hosting notes

Pine Commerce is built for ordinary shared hosting (LiteSpeed / Apache with cPanel or Plesk) as well as VPS setups.

- **Document root = the project's `public/` directory.** Never expose the project root. If the host forces
  `public_html`, put the project in `public_html` and point the document root at `public_html/public`.
- **No symlinks out of `public/`.** LiteSpeed does not follow them. `public/storage`, the admin assets
  (`public/vendor/commerce/admin`) and theme assets (`public/themes/{slug}`) are **real copies**, created by
  `commerce:install`, `commerce:publish` and `commerce:theme:publish`. **Never run `php artisan storage:link`.** If
  `public/storage` is already a symlink: remove it (`rm public/storage`, only when it *is* a symlink) and run
  `php artisan commerce:install` again.
- **Queue = `sync`.** Order emails and stock updates run inside the request; there is no worker to supervise.
- **Use a PHP 8.3+ CLI.** On cPanel the plain `php` in a cron job or SSH session may be older – use the full path of
  the 8.3 binary your host provides.
- **Old WordPress in a parent directory?** If its `.htaccess` sets `auto_prepend_file` (security plugins do),
  uncomment `php_value auto_prepend_file none` in the project's `public/.htaccess`.
- **Behind a CDN or load balancer:** set `TRUSTED_PROXIES` to its IP ranges, otherwise leave it empty.
- After changing `.env`, config or routes on a server: `php artisan optimize:clear && php artisan optimize`.

## Scheduled tasks (cron, or no cron)

The platform has background jobs: cancelling unpaid orders, abandoned-cart reminders, back-in-stock emails,
scheduled sale prices, the nightly tidy-up, the low-stock email and the daily update check. **One** cron line runs them
all, every minute:

```cron
* * * * * cd /path/to/acme-shop && php artisan schedule:run >> /dev/null 2>&1
```

On cPanel: *Advanced › Cron Jobs*, "Once Per Minute", and use the full path to a PHP 8.3+ binary in the command.
Leave "Cron Email" empty.

**No cron? Nothing breaks.** With `commerce.scheduler.web_fallback` on (the default) every due task runs right after a
storefront page has been sent to a visitor, at most every 5 minutes. That is fine for small shops, but reminders then
depend on traffic, so add the cron line when you can.

```bash
php artisan commerce:schedule:status                        # "Cron is running", each task with its last and next run
php artisan commerce:schedule:task carts.abandoned-emails    # run one task now
```

The same table is in **Admin › Settings › Scheduled tasks**.

## Deploying changes

The skeleton ships a Composer script that does everything a deploy needs:

```bash
git pull
composer install --no-dev --optimize-autoloader
composer deploy        # = migrate --force, commerce:publish, commerce:theme:publish, optimize
```

Take a database backup before anything that migrates:

```bash
mkdir -p ~/backups
mysqldump --single-transaction --routines acme_shop | gzip > ~/backups/acme_shop-$(date +%Y%m%d-%H%M).sql.gz
```

Never run `migrate:fresh`, `migrate:refresh`, `migrate:reset` or `db:wipe` on a database with real data – the
skeleton blocks them on MySQL (`DB::prohibitDestructiveCommands()` in `ClientServiceProvider`). Keep that guard.

## Going live checklist

The full version (with DNS, rollback plan and post-launch checks) is part 2, steps 13–15, of the
[Playbook](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md). In short:

- [ ] Database backup taken (and the old site's files and database, if you migrated).
- [ ] `.env`: `APP_URL=https://www.example.test`, `APP_ENV=production`, `APP_DEBUG=false`, `SESSION_SECURE_COOKIE=true`,
      live `MAIL_*` (SMTP or a transactional provider; SPF/DKIM/DMARC published), then
      `php artisan optimize:clear && php artisan optimize`.
- [ ] Live payment keys and live webhooks in Admin › Settings › Payments; one real low-value order and a refund.
- [ ] Tax rates and shipping zones checked with a few real addresses.
- [ ] TLS on every host name; the non-canonical host redirects to the canonical one.
- [ ] "Ask search engines not to index this site" **unticked** (Settings › SEO & tracking); `/robots.txt` allows crawling.
- [ ] Cron line added; `php artisan commerce:schedule:status` says "Cron is running".
- [ ] Daily backups of the database, `public/storage` and `.env`, kept off the server; one restore tested.
- [ ] `php artisan commerce:doctor` shows no FAIL.
- [ ] Migrated shop: `php artisan commerce:verify-urls` against the old sitemap is clean (see
      [Migrating from WooCommerce](02-migrating-from-woocommerce.md)).
- [ ] Sitemap submitted in Google Search Console; Merchant Center feed URL updated if you use
      `/feeds/google-shopping.xml`.

---

Next: [Migrating from WooCommerce](02-migrating-from-woocommerce.md) or
[Themes and templating](03-themes-and-templating.md).
