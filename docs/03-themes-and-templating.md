# 3. Themes and templating

A storefront theme is a folder of **Blade views, plain CSS/JS, images and fonts** – no build step. The core's
controllers never change per theme: they hand each view a documented set of variables (the *theme contract*) and
the theme decides the markup.

Reference: [THEMES.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/THEMES.md) and
[ARCHITECTURE.md §7–8](https://github.com/SetWebUK/ecom-core/blob/main/docs/ARCHITECTURE.md#7-theme-system) (the
binding contract).

**Contents**

- [Choosing an approach](#choosing-an-approach)
- [How themes work](#how-themes-work)
- [`theme.json`](#themejson)
- [The theme contract](#the-theme-contract)
- [Commands](#commands)
- [Walkthrough: a child theme](#walkthrough-a-child-theme)
- [Assets](#assets)
- [Theme settings, config and helpers](#theme-settings-config-and-helpers)
- [`Theme.php` hooks](#themephp-hooks)
- [Images](#images)
- [Feature flags in themes](#feature-flags-in-themes)
- [JSON-LD and the `@context` pitfall](#json-ld-and-the-context-pitfall)
- [Emails and PDF templates](#emails-and-pdf-templates)
- [Building a bespoke like-for-like theme](#building-a-bespoke-like-for-like-theme)
- [Rules every theme must follow](#rules-every-theme-must-follow)

---

## Choosing an approach

| Approach | When | How |
|---|---|---|
| **Default theme + settings** | a clean modern look is fine | keep `COMMERCE_THEME=default`; logo and brand in Admin › Settings, home page blocks in Admin › Content › Pages › Home, menus in Admin › Content › Menus |
| **Child theme** | the default layout with your branding and a few different templates | `php artisan commerce:theme:make acme --parent=default`; override only the files that differ |
| **Bespoke like-for-like** | the shop must look exactly like the old site | full fork: `php artisan commerce:theme:make acme --parent=default --copy`, then rebuild the old markup ([below](#building-a-bespoke-like-for-like-theme)) |

## How themes work

```
project/
├── themes/
│   └── acme/                      ← your theme (slug = folder name)
│       ├── theme.json             ← name, parent, supports, settings schema
│       ├── Theme.php              ← optional hooks (composers, body classes, routes)
│       ├── views/                 ← Blade views; only the ones you override
│       ├── assets/                ← css/ js/ images/ fonts/  (published as copies to public/themes/acme)
│       └── config/                ← optional: menus.php, home.php, blocks.php, product.php, seo.php
└── vendor/pine/commerce/resources/themes/default/   ← the complete neutral theme, end of every chain
```

**View lookup order** – the first file that exists wins:

```
resources/views (rare one-off overrides)  →  themes/{active}  →  its parent(s)  →  default (in the package)
```

**Active theme**: a staff preview (`?preview_theme=acme`) › the *Theme* setting in Admin › Settings › Theme ›
`COMMERCE_THEME` in `.env` › `default`.

So a child theme overrides **one file** by creating it at the same relative path, and inherits everything else.
Bare view names (`@include('partials.header')`, `@extends('layouts.app')`, `emails.orders.*`) resolve through the
chain; the `theme::` namespace (used by core controllers) is the same chain without `resources/views`.

## `theme.json`

```json
{
    "name": "Acme",
    "slug": "acme",
    "parent": "default",
    "version": "1.0.0",
    "requires": "pine/commerce ^1.0",
    "class": "Themes\\Acme\\Theme",
    "supports": ["side-cart", "ajax-catalog", "quick-view", "mobile-menu", "wishlist", "reviews", "stock-alerts",
                 "newsletter", "cookie-banner", "blog", "order-tracking", "contact-form", "emails"],
    "editor_css": ["css/app.css"],
    "settings": {
        "show_usp_bar": { "type": "bool", "default": true, "label": "Show the delivery message bar", "group": "Layout" }
    }
}
```

| Key | Required | Meaning |
|---|---|---|
| `name`, `slug`, `version` | yes | `slug` = folder name, `[a-z0-9-]+` |
| `parent` | no | parent theme slug; `default` is always last in the chain |
| `requires` | no | semver constraint on `pine/commerce`; `commerce:doctor` warns on a mismatch |
| `public_path` | no | where assets are published, relative to `public/` (default `themes/{slug}`). A like-for-like theme can keep the old site's asset URLs, e.g. `assets` |
| `class` | no | your `ThemeDefinition` subclass in `Theme.php` |
| `autoload` | no | PSR-4 map for `themes/{slug}/src` (no Composer change needed) |
| `supports` | yes | feature keys the theme implements; `commerce:theme:check` checks the matching views exist |
| `editor_css` | no | theme stylesheets loaded inside the admin rich-text editors |
| `settings` | no | owner-editable settings (`text`, `textarea`, `bool`, `int`, `image`, `link`, `color`, `select`), shown in Admin › Settings › Theme; `"settings_inherit": false` hides the parent's |

## The theme contract

Core renders these views (the first name that exists in the chain) and passes documented variables. The full table
with every variable is in [ARCHITECTURE.md §8](https://github.com/SetWebUK/ecom-core/blob/main/docs/ARCHITECTURE.md#8-theme-contract-views-rendered-by-core);
the most used:

| View | Rendered for | Key variables |
|---|---|---|
| `layouts.app` | every page (themes `@extends` it) | – |
| `home` / `pages.home` | the home page | `$page`, `$b` (home blocks), `$bestSellers`, `$seo` |
| `pages.{template}`, `pages.default` | CMS pages | `$page`, `$contentHtml`, `$showTitle`, `$seo`, `$bodyClass` (+ `$blocks` for block templates) |
| `catalog.archive` (or `catalog.shop`, `catalog.category`, `search.results`) | shop, categories, search | `$products` (paginator), `$facets`, `$activeFilters`, `$sorts`, `$heading`, `$breadcrumbs`, `$category`, `$seo` … |
| `product.show` | product page | `$product`, `$category`, `$crumbs`, `$variationData`, `$variationAttributes`, `$related`, `$reviews`, `$schema` (JSON-LD), `$seo` |
| `cart.side-cart` | the basket drawer | `$cart`, `$totals`, `$notices` |
| `checkout.show`, `checkout.thankyou`, `checkout.pay`, `checkout.verify-email` | checkout | `$cart`, `$totals`, `$gateways`, `$values`, `$order` … |
| `account.*`, `auth.*` | My account, login, password reset | `$user`, `$orders`, `$order`, `$addresses` … |
| `blog.index`, `blog.category`, `blog.show` | blog | `$posts`, `$post`, `$contentHtml`, `$seo` |
| `errors.404`, `errors.410`, `errors.500` | error pages | – |
| `emails.*` | every email | `$order`, `$heading`, `$storeName` … |

Every storefront view also receives `$store` (name, phone, email, address), `$cartCount` and `$theme`.
`partials.product-card` is a theme convention (core never renders it) and receives `$product`.

`commerce:theme:make` writes `views/README.md` into a new theme listing every contract view and every parent view you
can override.

## Commands

```bash
php artisan commerce:theme:make acme --parent=default [--name="Acme"] [--copy]   # scaffold (--copy = full fork)
php artisan commerce:theme:publish acme        # copy assets to public/themes/acme (or --all; --prune removes deleted files)
php artisan commerce:theme:check acme          # theme.json, parent chain, public_path, Theme.php, every contract view
php artisan commerce:theme:cache               # cache the theme manifest (also run by `php artisan optimize`)
php artisan commerce:theme:clear               # clear it (also run by `php artisan optimize:clear`)
```

Publishing **copies** files (never symlinks), keeps the modification time of unchanged files so `?v=` cache busting
only changes for edited ones, and refuses to overwrite a directory owned by another theme. Run it after every deploy
(`composer deploy` does).

## Walkthrough: a child theme

**1. Scaffold and publish**

```bash
php artisan commerce:theme:make acme --parent=default --name="Acme"
php artisan commerce:theme:publish acme
```

This creates `themes/acme/theme.json`, `Theme.php`, `config/`, `views/README.md` and `assets/css/theme.css`.

**2. Brand colours and fonts**

The default theme is driven by CSS custom properties. The owner-facing way is **Admin › Settings › Theme**
(`?theme=acme` edits the settings of a theme that is not active yet): logo, logo height, brand / accent / text /
background / panel colours, body and heading fonts, corner style, header style.

For values that belong to the brand and should live in Git, put them in `themes/acme/assets/css/theme.css`. It is
loaded automatically **after** the default stylesheet:

```css
/* themes/acme/assets/css/theme.css */
:root {
    --c-primary: #0f766e;          /* buttons, links, prices */
    --c-primary-hover: #115e59;
    --c-accent: #f59e0b;           /* sale badges */
    --font-heading: Georgia, "Times New Roman", serif;
    --radius: 4px;
}

.acme-usp { background: var(--c-primary); color: #fff; text-align: center; padding: .4rem; font-size: .9rem; }
```

Other tokens you can set: `--c-text`, `--c-muted`, `--c-bg`, `--c-surface`, `--c-border`, `--font-body`,
`--radius-lg`, `--container`, `--header-h`, `--logo-h` (see the top of the default theme's `assets/css/app.css`).

**3. Override the header**

```bash
mkdir -p themes/acme/views/partials
cp vendor/pine/commerce/resources/themes/default/views/partials/header.blade.php themes/acme/views/partials/
```

Edit the copy – for example add a message bar at the top, controlled by a theme setting:

```blade
{{-- themes/acme/views/partials/header.blade.php (first lines) --}}
@if (theme_setting('show_usp_bar', true))
    <div class="acme-usp">Free UK delivery on orders over £50</div>
@endif
{{-- … the rest of the copied header … --}}
```

Declare the setting in `themes/acme/theme.json` so the owner can switch it in Admin › Settings › Theme:

```json
"settings": {
    "show_usp_bar": { "type": "bool", "default": true, "label": "Show the delivery message bar", "group": "Layout" }
}
```

**4. Override the product card**

```bash
cp vendor/pine/commerce/resources/themes/default/views/partials/product-card.blade.php themes/acme/views/partials/
```

The card receives `$product`. Use the product presenter for prices, stock and sale badges, and `<x-media-image>` for
the photo:

```blade
{{-- themes/acme/views/partials/product-card.blade.php (simplified) --}}
@php
    $P = commerce_presenter();
    $image = $product->images->first();
    $sale = $P::sale($product);
@endphp
<article class="product-card {{ $P::inStock($product) ? '' : 'is-out' }}">
    <a class="product-card__media" href="{{ $product->url }}">
        <x-media-image :path="$image?->path" size="card" sizes="(min-width: 1100px) 280px, 46vw" :alt="$product->name" />
    </a>
    @if ($sale)<span class="badge badge--accent">-{{ $sale['percent'] }}%</span>@endif
    <h3 class="product-card__title"><a href="{{ $product->url }}">{{ $product->name }}</a></h3>
    {{-- keep the price, add-to-basket and wishlist markup from the copied file --}}
</article>
```

Prefer small overrides (a partial) over copying whole pages, so improvements to the parent keep reaching you.

**5. Check, preview, switch on**

```bash
php artisan commerce:theme:publish acme
php artisan commerce:theme:check acme          # "Theme [acme] is valid (chain: acme → default)."
```

Sign in to `/admin`, then open `/?preview_theme=acme` (or Admin › Settings › Theme › Preview). The preview is per
staff session, never cached, `noindex`, and shows an "Exit preview" bar – customers are unaffected. When it is
approved, an administrator switches it on in Admin › Settings › Theme, or set `COMMERCE_THEME=acme` in `.env` and run
`php artisan optimize:clear && php artisan optimize`.

## Assets

- Source: `themes/{slug}/assets/**`. Published copies: `public/{public_path}` (default `public/themes/{slug}`) –
  build output, git-ignored.
- Link them with `theme_asset('css/site.css')` or `@themeAsset('css/site.css')`: the URL of the first theme in the
  chain whose **published** copy exists, with `?v={filemtime}` for cache busting.
- A child theme's `assets/css/theme.css` is loaded automatically after the default stylesheet.
- Vendor libraries (Alpine, Swiper …) are plain files in `assets/js/` – there is no npm.
- After any asset change: `php artisan commerce:theme:publish acme`.

## Theme settings, config and helpers

| Helper | Returns |
|---|---|
| `theme()` | the active `Theme` |
| `theme_setting('key', $default)` / `@themeSetting('key')` | owner value (`theme.{slug}.key`) ?? `theme.json` default (child over parent) ?? `$default` |
| `theme_config('menus.fallbacks')` | a value from the chain's `config/*.php` files (child over parent per top-level key) |
| `theme_asset('css/app.css')` / `@themeAsset(…)` | asset URL with cache-busting version |
| `setting('store.phone')` | any store setting |
| `menu_tree('main')` | a normalised menu tree `[{label, url, children, …}]` for a location |
| `commerce_presenter()` | the product presenter class (`$P::sale()`, `$P::inStock()`, `$P::priceHtml()` …) |
| `commerce_feature('wishlist')` | is a feature switched on (and supported by the theme)? |
| `money($amount)` | formatted price in the store currency |
| `media_url()`, `image_srcset()`, `<x-media-image>` | images ([below](#images)) |

Theme config files (all optional): `config/menus.php` (menu locations and fallback menus), `config/home.php` (default
home-page blocks), `config/blocks.php` (extra page templates and their block schemas), `config/product.php` (product
copy), `config/seo.php` (default descriptions).

Menu labels and links may contain store-setting tokens such as `{store.phone}` or `tel:{store.phone}`.

## `Theme.php` hooks

```php
<?php

namespace Themes\Acme;

use Pine\Commerce\Theme\ThemeDefinition;

class Theme extends ThemeDefinition
{
    /** View composers, Blade directives, shortcodes … – only while this theme is in the active chain. */
    public function boot(): void
    {
        \Pine\Commerce\Commerce::shortcodeAlias('acme_contact', 'contact_form');   // an old WordPress shortcode
    }

    /** Extra data for views (also works for the `theme::` names core renders). */
    public function composers(): array
    {
        return [
            'product.show' => fn ($view) => $view->with('deliveryNote', setting('acme.delivery_note')),
        ];
    }

    /** Extra storefront routes (the catch-all is a fallback, so no ordering issues). */
    public function routes(): void {}

    /** Add body classes to a core-rendered view ($key e.g. 'product.show', 'catalog.category'). */
    public function bodyClass(string $key, string $classes, array $data): string
    {
        return $classes.' theme-acme';
    }
}
```

## Images

Uploads get **size variants next to the original**, named like WordPress's, plus WebP twins:

```
uploads/2026/05/photo.jpg              original
uploads/2026/05/photo-150x150.jpg      "thumbnail"  150×150 crop
uploads/2026/05/photo-400x300.jpg      "card"       fit 400×400 (real size in the name)
uploads/2026/05/photo-400x300.webp     WebP twin
```

Default sizes (`config('commerce.images.sizes')`): `thumbnail` 150×150 crop, `card` 400, `medium` 800, `large` 1600.
Asking for a size always returns the best file that **exists** – nothing breaks before sizes are generated.

| API | Returns |
|---|---|
| `media_url($path)` | the original |
| `media_url($path, 'card')`, `media_url($path, 240)` | the best variant for a named size / a 240×240 box |
| `image_srcset($path, 'card')` | `url 300w, url 768w, …` |
| `image_srcset($path, 'card', 'webp')` | the same list of WebP twins |

Use the component – it writes `<picture>`, `srcset`, `sizes`, real `width`/`height`, lazy loading and the WebP source:

```blade
<x-media-image :path="$image->path" size="card" sizes="(min-width: 1100px) 280px, 46vw" :alt="$image->alt" class="card__img" />
<x-media-image :path="$post->featured_image" size="large" sizes="100vw" :priority="true" />   {{-- LCP image: no lazy, fetchpriority --}}
```

Props: `path`, `size`, `sizes`, `alt`, `lazy` (true), `priority`, `picture` (true), `width`/`height`, `fallback`.
Add `picture { display: contents; }` to your CSS so the `<img>` stays the layout box. Never print a bare
`media_url($path)` for a photo. Existing media: `php artisan commerce:images:generate --missing`.

## Feature flags in themes

Optional features (wishlist, reviews, quick view, blog, newsletter, stock alerts, contact form, order tracking,
coupons …) can be switched off per shop. Their routes then answer 404, so wrap every entry point:

```blade
@if (commerce_feature('wishlist'))
    <button type="button" data-wishlist="{{ $product->id }}">Add to wishlist</button>
@endif
```

View-bearing features are only on when the theme lists them in `theme.json` `supports`.

## JSON-LD and the `@context` pitfall

Laravel 13 has a `@context` Blade directive, so `'@context'` inside `{!! json_encode([...]) !!}` in a Blade file is
compiled as PHP and breaks the page. Either escape it as `@@context`:

```blade
<script type="application/ld+json">{!! json_encode(['@@context' => 'https://schema.org', '@type' => 'Organization', 'name' => $store['name']], JSON_UNESCAPED_SLASHES | JSON_HEX_TAG | JSON_HEX_AMP) !!}</script>
```

or build the array inside `@php … @endphp`, where Blade directives are not compiled. Always pass `JSON_HEX_TAG` so a
`</script>` inside a product name cannot break out of the tag. The product page already gets a ready `$schema` array
from core – the default theme prints it with
`json_encode($schema, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_HEX_TAG | JSON_HEX_AMP)`.

## Emails and PDF templates

- **Emails** are views too and resolve through the theme chain: ship `views/emails/layouts/base.blade.php` to restyle
  every email, or a single file such as `views/emails/orders/customer-processing-order.blade.php`. Variables: `$order`,
  `$heading`, `$storeName` (+ extras per email). List: ARCHITECTURE.md §8.3. To *replace* an order email with your
  own Mailable, see [Building features](04-building-features.md#order-events-and-emails).
- **PDF invoices and packing slips**: `views/pdf/invoice.blade.php` and `views/pdf/packing-slip.blade.php` in the theme
  (or `resources/views/pdf/…`) replace the core templates. They are rendered by dompdf: use tables rather than
  flex/grid and the embedded `DejaVu Sans` font. See
  [INVOICES.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/INVOICES.md).
- **Admin views** can be overridden in `resources/views/vendor/commerce/admin/…`, but prefer adding pages – overrides
  must be re-checked after every platform update.

## Building a bespoke like-for-like theme

When a migrated shop must look exactly as before:

1. **Capture reference HTML** of every page type from the running old site *before anything changes* – one file per
   URL (path with `/` → `__`, `home.html` for `/`), including states such as a full basket, checkout, My account,
   search, page 2 of a category and the 404 page:

   ```bash
   mkdir -p storage/app/wp-reference/html
   while read -r path; do
     f=$(echo "${path#/}" | sed 's#/$##; s#/#__#g'); [ -z "$f" ] && f=home
     curl -s "https://www.example.test${path}" -o "storage/app/wp-reference/html/${f}.html"; sleep 1
   done < storage/app/url-paths.txt
   ```

   `storage/app` is git-ignored, so reference material never reaches the repository. The importer can read the same
   folder (`--snapshots=storage/app/wp-reference/html`).

2. **Fork the default theme**: `php artisan commerce:theme:make acme --parent=default --copy`.
3. **Extract the CSS actually used** (theme, page builder, plugins, inline styles) into a few stylesheets under
   `themes/acme/assets/css/`. Fonts and icons the same way. If the old asset URLs must survive (logos in sent emails,
   imported content), set `"public_path": "assets"` in `theme.json`.
4. **Rebuild the views** with the same DOM structure and class names: layout, header/footer/menus, product card,
   category, product, basket/checkout, account, blog, pages. Use the contract variables; add old body classes in
   `Theme::bodyClass()`; register old shortcode names with `Commerce::shortcodeAlias()`.
5. **Compare** page by page against the reference HTML (title, meta description, canonical, H1, visible text, layout
   in the browser) and fix differences.
6. **Keep a regression check**: a URL list, a snapshot script and a diff script (normalising CSRF tokens, nonces,
   `?v=` versions and dates) in the shop's repository. Run it before every platform update – zero unexplained
   differences, or it does not ship. See [Playbook 3.4](https://github.com/SetWebUK/ecom-core/blob/main/docs/PLAYBOOK.md#34-regression-check-clients-with-a-like-for-like-theme).

## Rules every theme must follow

- Checkout posts the core field names (`billing_email`, `billing_*`, `shipping_*`, `shipping_method`,
  `payment_method`, `terms` …) and uses the core JSON endpoints (`/checkout/update`, `/cart/coupon`, `/checkout`).
- The basket uses `POST /cart/add|update|remove` with `Accept: application/json`; the response's `html` is your
  `cart.side-cart` view.
- Accessibility: skip link, labelled controls, visible focus, focus trap in drawers, `aria-live` updates.
- Images through `<x-media-image>`; optional features wrapped in `commerce_feature()`.
- `php artisan commerce:theme:check` passes after every platform upgrade.

The full list: [THEMES.md "Rules the default theme follows"](https://github.com/SetWebUK/ecom-core/blob/main/docs/THEMES.md#rules-the-default-theme-follows-keep-them-in-forks).

---

Next: [Building features](04-building-features.md).
