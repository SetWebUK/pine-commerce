# FAQ

**Is Pine Commerce a Laravel package or an application?**
Both, in a way. The platform is a Composer package (`pine/commerce`, repository
[ecom-core](https://github.com/SetWebUK/ecom-core)). Each shop is an ordinary Laravel 13 application that requires
it – start from the [ecom-skeleton](https://github.com/SetWebUK/ecom-skeleton) starter app.

**Do I need Node.js, npm or Vite?**
No. Themes and the back office are plain CSS and JavaScript (Alpine.js is a committed file). There is no build step.

**Why no Filament / Livewire / Nova?**
The back office is hand-built with Blade, Alpine.js and a small component library, so it stays fast, light on shared
hosting, and free of a framework's upgrade cycle. Your own admin pages use the same components
([Building features](04-building-features.md#worked-example-2-an-admin-page-menu-item-settings-screen-and-migration)).

**Will it run on shared hosting (cPanel, LiteSpeed)?**
Yes – that is the main target. It needs PHP 8.3+, MySQL/MariaDB, SSH and a document root you can point at `public/`.
No queue worker, no Redis, cron optional. See [Hosting notes](01-getting-started.md#hosting-notes).

**Why must I never run `php artisan storage:link`?**
LiteSpeed does not follow symlinks out of `public/`. `public/storage` is a real directory created by
`commerce:install`, and assets are published as copies.

**What if I have no cron?**
Scheduled tasks run from a web fallback after storefront page views (at most every 5 minutes). Add the cron line when
you can: `* * * * * cd /path/to/shop && php artisan schedule:run >> /dev/null 2>&1`.

**Can I edit files in `vendor/pine/commerce`?**
No – they are replaced on every update. Use config, the theme or the extension API
([Building features](04-building-features.md)). If the platform really needs a change, contribute it
([Contributing](07-contributing.md)).

**Can customers keep their WooCommerce passwords?**
Yes. WordPress password hashes are imported as they are and verified on the customer's first login.

**Will my old URLs keep working after a migration?**
That is the goal and it is testable: products and categories keep their WooCommerce paths, redirects are imported or
generated, and `php artisan commerce:verify-urls` proves every URL in the old sitemap answers 200 or 301 → 200
([Migrating](02-migrating-from-woocommerce.md#7-url-parity-and-redirects)).

**Which payment methods are included?**
Stripe (cards and wallets), PayPal and bank transfer. Add others as a gateway class
([worked example](04-building-features.md#worked-example-1-a-custom-payment-gateway)).

**Where do payment keys go – `.env`?**
No. An administrator enters them in Admin › Settings › Payments, where they are stored encrypted with `APP_KEY`. Never
change `APP_KEY` after go-live.

**Does it handle VAT?**
Yes: tax classes, rates per country/postcode, prices entered with or without VAT, VAT lines on baskets, orders, emails
and invoices. `commerce:install` seeds UK rates.

**How do updates reach a shop?**
Admin › Updates finds new releases daily; an administrator approves one and it installs with a backup, maintenance mode,
health check and automatic rollback. Or use `composer update pine/commerce`
([Updates and releases](06-updates-and-releases.md)).

**Something is wrong – where do I start?**
`php artisan commerce:doctor` (or Admin › Settings › System), then the log in `storage/logs/`. Common fixes are in the
[Playbook troubleshooting table](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md#part-5--troubleshooting).
After changing `.env` or config on a server, always run `php artisan optimize:clear && php artisan optimize`.

**The admin login just returns to the login form.**
You are on plain http with `SESSION_SECURE_COOKIE=true`. Use https, or set it to `false` for local http work.
