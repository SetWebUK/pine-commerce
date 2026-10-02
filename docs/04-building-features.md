# 4. Building features

How to bolt your own features onto Pine Commerce **without editing the core**, so platform updates keep installing
cleanly. Everything here lives in your shop project.

Reference: [EXTENDING.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/EXTENDING.md) (every extension point),
[ADMIN_UI.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/ADMIN_UI.md) (back-office components) and the
source of the API itself,
[`src/Commerce.php`](https://github.com/SetWebUK/ecom-core/blob/main/src/Commerce.php).

**Contents**

- [The rule and the order of preference](#the-rule-and-the-order-of-preference)
- [Where client code lives](#where-client-code-lives)
- [The extension API at a glance](#the-extension-api-at-a-glance)
- [Worked example 1: a custom payment gateway](#worked-example-1-a-custom-payment-gateway)
- [Worked example 2: an admin page, menu item, settings screen and migration](#worked-example-2-an-admin-page-menu-item-settings-screen-and-migration)
- [Worked example 3: a scheduled job](#worked-example-3-a-scheduled-job)
- [Shipping calculators](#shipping-calculators)
- [Settings fields and dashboard widgets](#settings-fields-and-dashboard-widgets)
- [Content: page templates, shortcodes, menu locations](#content-page-templates-shortcodes-menu-locations)
- [Product presentation](#product-presentation)
- [Order events and emails](#order-events-and-emails)
- [Importer adapters](#importer-adapters)
- [Feature flags](#feature-flags)
- [Config overrides](#config-overrides)
- [Your own data: migrations and models](#your-own-data-migrations-and-models)
- [Storefront routes](#storefront-routes)
- [Testing](#testing)
- [Checklist](#checklist)

---

## The rule and the order of preference

**Never edit `vendor/pine/commerce`.** Change behaviour through, in this order:

1. **Config** – `config/commerce.php`, `config/commerce-import.php` (feature flags, currency, catalogue filters …).
2. **Settings** – what the shop owner edits in Admin › Settings.
3. **The theme** – views, CSS/JS, theme config, `Theme.php` ([Themes](03-themes-and-templating.md)).
4. **Client code** – `App\Providers\ClientServiceProvider` using the `Pine\Commerce\Commerce` extension API, your own
   routes, models, tables, migrations and importer adapters.

If none of these can do it, the change belongs in the platform – made generic, behind a config key or feature flag
whose default keeps current behaviour ([Contributing](07-contributing.md)).

## Where client code lives

```
app/
├── Providers/ClientServiceProvider.php   ← register everything here (boot())
├── Providers/ExtensionExamples.php       ← worked examples shipped with the skeleton (not called; delete when done)
├── Payments/                             ← gateways
├── Shipping/                             ← shipping calculators
├── Http/Controllers/Admin/               ← back-office pages
├── Scheduling/                           ← scheduled task classes
├── Import/                               ← WordPress importer adapters
└── Models/                               ← your models (App\Models\User extends Pine\Commerce\Models\User)
database/migrations/                      ← your tables
resources/views/admin/                    ← your back-office views
tests/                                    ← your tests (in-memory SQLite)
```

`ClientServiceProvider` is already registered in `bootstrap/providers.php`. Keep its
`DB::prohibitDestructiveCommands()` guard.

## The extension API at a glance

All static methods on `Pine\Commerce\Commerce`, called from `ClientServiceProvider::boot()` (or a theme's
`Theme::boot()`):

| Method | Purpose |
|---|---|
| `gateway(string $code, string\|PaymentGateway\|null $gateway)` | add / replace / remove (`null`) a payment gateway |
| `shippingCalculator(string $pattern, string\|ShippingCalculator\|Closure $calculator)` | price or hide delivery options whose code matches (`fnmatch`) |
| `adminRoutes(Closure $routes)` | back-office routes: `/admin` prefix, `admin.` names, staff middleware |
| `adminMenu()->add(…)` / `->child(…)` / `->remove($route)` | sidebar entries |
| `settings(string $key, array $group, array $sections)` (alias `settingsGroup`) | a new Admin › Settings screen |
| `settingsFields(string $group, array $section)` | an extra card of fields on an existing settings screen |
| `dashboardWidget(string $key, array\|string\|DashboardWidget $widget)` / `removeDashboardWidget($key)` | dashboard cards |
| `pageTemplate(string $key, array $meta, array $schema = [])` | page templates with a block schema |
| `menuLocation(string $key, string $label)` | menu locations in Admin › Menus |
| `shortcode(string $name, callable $render, array $patterns = [], bool $unwrap = true)` / `shortcodeAlias($alias, $name)` | content shortcodes |
| `presenter(string $class)` / `presenterMethod(string $name, callable $method)` | product presenter subclass / extra presenter methods |
| `facetSorter(string $class)` | order of shop-filter options |
| `onOrderPlaced(callable $cb)` / `onOrderStatus(string\|array $statuses, callable $cb)` | order hooks |
| `orderEmail(string $key, string $mailable)` | replace an order email |
| `scheduledTask(string $key, array $options)` | a scheduled job (cron + web fallback, status, last result) |
| `importAdapter(string\|Adapter $adapter)` / `importStep(string\|Step $step)` | WordPress importer extensions |
| `feature(string $key, bool $theme = true)` | is a feature flag on? |

Also available: `Commerce::payments()` (the payment manager), `Commerce::shortcodes()`, `Commerce::userModel()`,
`Commerce::registry()`. Contracts you implement live in `Pine\Commerce\Contracts` (`PaymentGateway`,
`ShippingCalculator`, `DashboardWidget`) and `Pine\Commerce\Import\Contracts`.

---

## Worked example 1: a custom payment gateway

A "Pay on collection" method: the customer pays in the shop when they pick the order up. The easiest base class is
`Pine\Commerce\Services\Payments\Gateway` (the built-in Stripe, PayPal and bank-transfer gateways extend it).

```php
<?php
// app/Payments/CollectionGateway.php

namespace App\Payments;

use Illuminate\Http\Request;
use Pine\Commerce\Models\Order;
use Pine\Commerce\Services\Payments\Gateway;
use Pine\Commerce\Services\Payments\PaymentResult;

class CollectionGateway extends Gateway
{
    public function code(): string
    {
        return 'collection';                 // stored in orders.payment_method
    }

    protected function defaultTitle(): string
    {
        return 'Pay when you collect';
    }

    public function isConfigured(): bool
    {
        return filled($this->setting('instructions'));
    }

    /** The card shown in Admin › Settings › Payments (rendered, validated and saved by the core). */
    public function adminSettings(): array
    {
        return [
            'label' => 'Pay on collection',
            'icon' => 'credit-card',
            'description' => 'Customers pay by card or cash when they collect their order.',
            'fields' => static::baseSettingsFields($this->defaultTitle()) + [   // enabled, title, description
                'instructions' => ['type' => 'textarea', 'label' => 'Collection instructions',
                    'help' => 'Shown on the order-received page.'],
            ],
        ];
    }

    /** Trusted HTML shown inside this method's panel at checkout. */
    public function checkoutHtml(): string
    {
        return '<p>'.e((string) $this->setting('instructions')).'</p>';
    }

    /** Called by the checkout after the order was created (status "pending"). */
    public function process(Order $order, Request $request): PaymentResult
    {
        $order->updateStatus('on-hold', 'Awaiting payment on collection.');

        return PaymentResult::success($order->view_url);
    }
}
```

Register it:

```php
// app/Providers/ClientServiceProvider.php, in boot()
\Pine\Commerce\Commerce::gateway('collection', \App\Payments\CollectionGateway::class);
```

An administrator now sees a **Pay on collection** card in Admin › Settings › Payments, enables it and fills in the
instructions; `commerce:doctor` lists it with the other gateways.

**An online gateway** (redirect to a provider) adds three things:

```php
public function process(Order $order, Request $request): PaymentResult
{
    $session = Http::timeout(10)->withToken($this->secret('api_key'))
        ->post('https://api.provider.example/sessions', [
            'amount' => $this->pence((float) $order->total),
            'reference' => $order->number,
            'return_url' => route('checkout.payment.return', ['gateway' => $this->code(), 'order' => $order->number, 'key' => $order->order_key]),
        ])->throw()->json();

    return PaymentResult::redirect($session['redirect_url']);   // or success($url) / action([...]) / failure($message)
}

/** GET /checkout/payment/{code}/return – the core looks the order up and checks its key first. */
public function handleReturn(Order $order, Request $request): PaymentResult
{
    // confirm with the provider, then book the money (idempotent – the webhook may arrive first)
    \Pine\Commerce\Services\Payments\PaymentManager::complete($order, $this->code(), $request->query('transaction'), (float) $order->total);

    return PaymentResult::success($order->view_url);
}

/** POST /webhooks/{code} (CSRF-exempt) – verify the signature, then PaymentManager::complete() or ::fail(). */
public function handleWebhook(Request $request): \Symfony\Component\HttpFoundation\Response
{
    return response('ok');
}
```

- Field types for `adminSettings()`: `bool`, `text`, `textarea`, `secret` (encrypted with `APP_KEY`, write-only in the
  form). Options: `help`, `default`, `placeholder`, `pattern` + `message`, `mono`, `optional`, `wide`.
  Add `'webhook' => ['provider' => 'Provider', 'where' => 'Dashboard › Webhooks']` to show the webhook URL.
- Read values with `$this->setting('key')` and `$this->secret('key')`; validate cross-field rules in
  `validateSettings(array $values, Validator $validator)`.
- `PaymentManager::complete($order, $code, $transactionId, $amount)` marks the order paid (pending → processing,
  notes, emails); `PaymentManager::fail($order, $code, $reference, $message)` records a failed attempt.
- Refunds from the order screen: return `true` from `supportsRefunds()` and implement
  `refund(Order $order, float $amount, ?string $reason = null): ?string`.
- Remove a built-in gateway with `Commerce::gateway('bacs', null)`.

## Worked example 2: an admin page, menu item, settings screen and migration

A "Trade applications" page: businesses apply for a trade account, staff approve them in the back office.

**Migration** (your own table – never add columns to core tables):

```php
<?php
// database/migrations/2026_10_01_000000_create_trade_applications_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('trade_applications', function (Blueprint $table) {
            $table->id();
            $table->string('company');
            $table->string('email');
            $table->string('status')->default('pending');   // pending | approved | expired
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('trade_applications');
    }
};
```

**Model**:

```php
<?php
// app/Models/TradeApplication.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class TradeApplication extends Model
{
    protected $fillable = ['company', 'email', 'status'];
}
```

**Controller** – an ordinary Laravel controller:

```php
<?php
// app/Http/Controllers/Admin/TradeApplicationController.php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\TradeApplication;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class TradeApplicationController extends Controller
{
    public function index(): View
    {
        $applications = TradeApplication::latest()->paginate(25);

        return view('admin.trade.index', compact('applications'));
    }

    public function approve(TradeApplication $application): RedirectResponse
    {
        $application->update(['status' => 'approved']);

        return back()->with('success', "{$application->company} approved.");   // toast
    }
}
```

**View** – extends the admin layout and uses the `<x-admin.*>` components (no admin framework):

```blade
{{-- resources/views/admin/trade/index.blade.php --}}
@extends('commerce::admin.layouts.app', ['width' => 'wide'])

@section('title', 'Trade applications')

@section('content')
    <x-admin.page-header title="Trade applications" subtitle="Businesses asking for a trade account." />

    <x-admin.card flush>
        @if ($applications->isEmpty())
            <x-admin.empty title="No applications yet" />
        @else
            <x-admin.table>
                <x-slot:head>
                    <x-admin.th>Company</x-admin.th>
                    <x-admin.th>Email</x-admin.th>
                    <x-admin.th>Status</x-admin.th>
                    <x-admin.th align="right">Action</x-admin.th>
                </x-slot:head>
                @foreach ($applications as $application)
                    <tr>
                        <td>{{ $application->company }}</td>
                        <td>{{ $application->email }}</td>
                        <td><x-admin.badge>{{ ucfirst($application->status) }}</x-admin.badge></td>
                        <td class="num">
                            @if ($application->status === 'pending')
                                <form method="post" action="{{ route('admin.trade.approve', $application) }}">
                                    @csrf
                                    <x-admin.button type="submit" size="sm">Approve</x-admin.button>
                                </form>
                            @endif
                        </td>
                    </tr>
                @endforeach
            </x-admin.table>
            <x-admin.pagination :paginator="$applications" />
        @endif
    </x-admin.card>
@endsection
```

**Registration** – routes, sidebar entry with a badge, and a settings screen:

```php
// app/Providers/ClientServiceProvider.php
use App\Http\Controllers\Admin\TradeApplicationController;
use App\Models\TradeApplication;
use Illuminate\Support\Facades\Route;
use Pine\Commerce\Commerce;

public function boot(): void
{
    // URL prefix /admin, route names prefixed "admin.", staff-only middleware – like the core's own pages
    Commerce::adminRoutes(function () {
        Route::get('trade-applications', [TradeApplicationController::class, 'index'])->name('trade.index');            // admin.trade.index
        Route::post('trade-applications/{application}/approve', [TradeApplicationController::class, 'approve'])->name('trade.approve');
    });

    Commerce::adminMenu()->add('Trade applications', 'briefcase', 'admin.trade.index',
        active: ['admin.trade.*'],
        after: 'admin.customers.index',
        count: fn () => TradeApplication::where('status', 'pending')->count());   // sidebar badge

    // Admin › Settings › Trade accounts – rendered, validated and saved by the core; read with setting('trade.…')
    Commerce::settings('trade', ['label' => 'Trade accounts', 'icon' => 'briefcase', 'description' => 'How trade applications are handled.'], [
        ['title' => 'Applications', 'fields' => [
            ['key' => 'trade.expire_after_days', 'label' => 'Expire unanswered applications after (days)', 'type' => 'int', 'default' => 30, 'min' => 1, 'max' => 365],
            ['key' => 'trade.auto_expire', 'label' => 'Expire old applications automatically', 'type' => 'bool', 'default' => true],
        ]],
    ]);
}
```

Then `php artisan migrate` and open `/admin/trade-applications`.

Notes:

- `adminMenu()->add(string $label, ?string $icon, string $route, array $active = [], ?string $after = null,
  array $children = [], ?string $feature = null, bool $admin = false, int|Closure|null $count = null)`.
  `->child('admin.orders.index', 'Trade orders', 'admin.trade.orders')` adds a sub-item under a core section;
  `->remove('admin.reports.index')` hides a core entry.
- Administrators only: `->middleware('admin:admin')` on the route and `admin: true` on the menu entry.
- Don't put client pages under `admin/settings/…` – that path belongs to the settings screens.
- Components: `page-header`, `card`, `table`, `th`, `row-check`, `pagination`, `filters`, `status-tabs`, `button`,
  `badge`, `empty`, `form`, `field`, `input`, `select`, `textarea`, `money`, `image-picker`, `product-picker`, `modal`,
  `confirm` and more – each has a props comment at the top of its file; patterns in
  [ADMIN_UI.md](https://github.com/SetWebUK/ecom-core/blob/main/docs/ADMIN_UI.md).
- Alpine on a component tag needs `x-bind:` / `x-on:` (`:attr` on `<x-…>` is Blade's PHP binding).

## Worked example 3: a scheduled job

Expire trade applications nobody answered. A task class gives you a "skip" reason and a one-line summary that shows
up in the status table:

```php
<?php
// app/Scheduling/ExpireTradeApplications.php

namespace App\Scheduling;

use App\Models\TradeApplication;
use Pine\Commerce\Scheduling\Task;

class ExpireTradeApplications extends Task
{
    /** Why there is nothing to do right now, or null to run. */
    public function skipReason(): ?string
    {
        return TradeApplication::where('status', 'pending')->exists() ? null : 'No pending applications';
    }

    /** Do the work; the returned line is the summary shown in the status table. */
    public function handle(): string
    {
        $days = (int) setting('trade.expire_after_days', 30);

        $expired = TradeApplication::where('status', 'pending')
            ->where('created_at', '<', now()->subDays($days))
            ->update(['status' => 'expired']);

        return "{$expired} application(s) expired";
    }
}
```

```php
// ClientServiceProvider::boot()
Commerce::scheduledTask('acme.trade-expire', [
    'label' => 'Expire trade applications',
    'schedule' => 'dailyAt:03:00',
    'task' => \App\Scheduling\ExpireTradeApplications::class,
    'setting' => 'trade.auto_expire',      // skipped while the owner has switched this off …
    'setting_default' => true,             // … (value when the setting was never saved)
]);
```

```bash
php artisan commerce:schedule:status                    # acme.trade-expire (client) | 0 3 * * * | idle: No pending applications
php artisan commerce:schedule:task acme.trade-expire    # run it now
```

It runs from the one cron line, from the **web fallback** on sites without cron, or by hand; its last result appears
in Admin › Settings › Scheduled tasks (badge "Custom").

| Option | |
|---|---|
| `schedule` (required) | a cron expression (`'*/15 * * * *'`), a Laravel frequency (`'hourly'`, `'everyFifteenMinutes'`), a method with arguments (`'dailyAt:02:30'`, `'weeklyOn:1,08:00'`), or several chained with `\|` (`'weekdays\|dailyAt:09:00'`). Store timezone. No sub-minute schedules |
| exactly one of `task` / `call` / `command` | a `Task` subclass; a closure or invokable (container-injected; the returned string is the summary); or an artisan command line (non-zero exit = failed) |
| `label`, `description` | for the status table and admin screen |
| `feature` | skip while this feature flag is off |
| `setting`, `setting_default` | skip while this owner setting is off |
| `skip` | a callable returning a skip reason or `null` |

Rules: prefix keys with your shop slug (`acme.…`; core keys are refused). A task must be **safe to run late, twice, or
from the web fallback**: keep it short, give HTTP calls a timeout, never make checkout depend on it. Switch one off
without code: `'scheduler' => ['tasks' => ['acme.trade-expire' => false]]` in `config/commerce.php`.

---

## Shipping calculators

Delivery methods are data: Admin › Settings › Shipping › zone › method, with five built-in types (flat rate, free
shipping, weight-based rates, price-based rates, local pickup) and a **Code** per method. There is no API to add a new
method type; instead, a **calculator** re-prices (or hides) the methods whose code matches a pattern, *after* the
method's own type has priced it.

```php
<?php
// app/Shipping/WeightSurcharge.php

namespace App\Shipping;

use Illuminate\Support\Collection;
use Pine\Commerce\Contracts\ShippingCalculator;
use Pine\Commerce\Models\ShippingMethod;

class WeightSurcharge implements ShippingCalculator
{
    public function cost(ShippingMethod $method, float $cost, Collection $lines, array $context): ?float
    {
        $kg = $lines->sum(fn ($line) => (float) ($line->product->weight ?? 0) * $line->quantity);

        return $kg > 30 ? null : $cost + max(0, ceil($kg - 2)) * 1.50;   // null hides the option; + £1.50/kg over 2 kg
    }
}
```

```php
Commerce::shippingCalculator('courier_*', \App\Shipping\WeightSurcharge::class);
Commerce::shippingCalculator('pallet', fn ($method, $cost, $lines, $context) => $context['subtotal'] >= 500 ? 0.0 : null);
```

`$context` holds `subtotal`, `discount`, `country`, `postcode`, `zone`, `free_shipping`, `lines`. The first matching
registration wins; tax on the result follows the method's tax status.

## Settings fields and dashboard widgets

```php
// an extra card on an existing settings screen (general, checkout, emails, seo, or one of yours)
Commerce::settingsFields('checkout', ['title' => 'Gift messages', 'fields' => [
    ['key' => 'gifts.enabled', 'label' => 'Offer a gift message at checkout', 'type' => 'bool', 'default' => false],
]]);

// a dashboard card; 'data' only runs when the card is shown, a widget that throws is logged and left out
Commerce::dashboardWidget('trade', [
    'title' => 'Trade applications', 'view' => 'admin.widgets.trade', 'sort' => 50, 'wide' => false,
    'data' => fn (\Illuminate\Http\Request $request) => ['pending' => \App\Models\TradeApplication::where('status', 'pending')->count()],
]);
```

Settings field types: `text`, `textarea`, `email`, `emails`, `url`, `link`, `image`, `bool`, `int`, `decimal`,
`money`, `select` (`options` array or `'pages'`), `list`; options `default`, `help`, `drives`, `required`, `min`,
`max`, `pattern`, `placeholder`. Group options: `label`, `icon`, `description`, `admin` (administrators only),
`feature`.

## Content: page templates, shortcodes, menu locations

```php
// Admin › Pages › Template "Landing page"; the page builder shows these block fields;
// the storefront renders the view with $page, $blocks, $contentHtml, $seo, $bodyClass
Commerce::pageTemplate('landing', ['label' => 'Landing page', 'help' => 'Campaign page', 'view' => 'pages.landing'], [
    'headline' => 'text',
    'hero' => ['object', ['image' => 'image', 'button_text' => 'text', 'button_link' => 'link']],
    'products' => 'ids',
    'faq' => ['list', ['question' => 'text', 'answer' => 'html'], ['label' => 'Questions', 'item' => 'question', 'max' => 20]],
]);

// [store_hours days="Mon–Sat"] in pages and posts
Commerce::shortcode('store_hours', fn (array $atts, array $context) => view('partials.store-hours', ['days' => $atts['days'] ?? 'Mon–Fri'])->render());
Commerce::shortcodeAlias('opening_times', 'store_hours');    // keep an old WordPress shortcode name working

Commerce::menuLocation('top_bar', 'Top bar links');          // then render it in the theme
```

Block types: `text`, `multiline`, `textarea`, `html`, `link`, `image`, `int`, `bool`, `ids` (product ids),
`['list', fields, meta]`, `['object', fields, meta]`. Core shortcodes: `[contact_form]`, `[blog_index]`, `[sitemap]`,
`[order_tracking]`.

## Product presentation

Themes call the product presenter through `commerce_presenter()`. Add a method without subclassing:

```php
Commerce::presenterMethod('deliveryPromise', fn (\Pine\Commerce\Models\Product $product) =>
    $product->stock_status === 'instock' ? 'Order by 3pm for next-day delivery' : null);
```

```blade
@php $P = commerce_presenter(); @endphp
@if ($promise = $P::deliveryPromise($product))<p class="delivery-promise">{{ $promise }}</p>@endif
```

For bigger changes subclass `Pine\Commerce\Services\Catalog\ProductPresenter` and register it with
`Commerce::presenter(\App\Catalog\AcmePresenter::class)` – every method is called through `static::`, so one override
changes every caller. Order shop-filter options with `Commerce::facetSorter()` (a class implementing
`Pine\Commerce\Services\Catalog\FacetSorter`).

## Order events and emails

| Event | When | Payload |
|---|---|---|
| `Pine\Commerce\Events\OrderPlaced` | checkout created the order (before payment) | `$event->order` |
| `Pine\Commerce\Events\OrderStatusChanged` | any status change | `$event->order`, `$event->from` (?string), `$event->to` |

Statuses: `pending` → `processing` (paid) → `completed`; also `on-hold`, `failed`, `cancelled`, `refunded`.

```php
use Pine\Commerce\Models\Order;

Commerce::onOrderPlaced(fn (Order $order) => \App\Attribution::record($order));
Commerce::onOrderStatus('completed', fn (Order $order, ?string $from, string $to) =>
    Http::timeout(5)->post(config('services.erp.url'), ['order' => $order->number]));
Commerce::onOrderStatus(['cancelled', 'refunded'], fn (Order $order) => \App\Loyalty::revoke($order));

// or plain Laravel
Event::listen(\Pine\Commerce\Events\OrderStatusChanged::class, fn ($event) => /* … */ null);
```

The queue is `sync`, so listeners run inside the customer's request: keep them fast and always give HTTP calls a
timeout.

Replace an order email with your own Mailable (its constructor receives the `Order`). Keys: `new_order`,
`cancelled_order`, `failed_order`, `customer_processing`, `customer_on_hold`, `customer_completed`,
`customer_refunded`:

```php
Commerce::orderEmail('customer_completed', \App\Mail\CompletedWithReviewInvite::class);
```

To restyle rather than replace, override the email views in the theme ([Themes](03-themes-and-templating.md#emails-and-pdf-templates)).

## Importer adapters

Bespoke WordPress data gets an adapter in `app/Import/{Shop}/` registered with `Commerce::importAdapter()`, or a step
that always runs with `Commerce::importStep()`. Example and capabilities:
[Migrating from WooCommerce](02-migrating-from-woocommerce.md#10-client-specific-data-adapters-and-config).

## Feature flags

Feature flags are per shop, in `config/commerce.php` → `features` (merged key by key, so you only list what you
change). Read them with `commerce_feature('key')` in Blade or `Commerce::feature('key')` in PHP.

```php
// config/commerce.php
return [
    'features' => [
        'blog' => false,
        'reviews' => env('COMMERCE_REVIEWS', true),
        'pay_in_3' => true,
    ],
];
```

On by default: `blog`, `wishlist`, `reviews`, `stock_alerts`, `newsletter`, `contact_form`, `order_tracking`,
`quick_view`, `google_feed`, `abandoned_carts`, `coupons`, `guest_checkout`, `registration`, `reports`, `redirects`,
`multi_shipping`, `product_brand`, `legacy_content`, `wp_404_guess`, `add_to_cart_query`, `product_csv`, `updater`.
Off by default: `product_condition`, `spec_highlights`, `pay_in_3`.

A switched-off feature is off everywhere: its routes answer 404 (names kept), its admin pages and menu entries
disappear, the shipped themes hide its entry points, and the sitemap, emails and importer follow. Gate your own pieces
the same way: `->middleware(\Pine\Commerce\Http\Middleware\RequireFeature::for('reviews'))` on routes, `'feature' =>
'reviews'` on menu entries, settings groups, widgets and scheduled tasks. Shop owners see every flag (read-only) in
Admin › Settings › System. After changing one on a server: `php artisan optimize:clear && php artisan optimize`.

## Config overrides

`config/commerce.php` in your project **replaces whole top-level keys** (the merge is shallow) – to change one
`catalog` key, copy the complete `catalog` block from `vendor/pine/commerce/config/commerce.php`. `features` is the
exception (merged key by key). Never change cookie or header names (`cart.cookie`, `catalog.ajax_header` …) after
go-live – live baskets would be lost.

## Your own data: migrations and models

- Put migrations in `database/migrations` – they run with the core's on `php artisan migrate` / `composer deploy`.
- **Your own tables only.** Never add columns to core tables: a later platform migration could collide.
- Attach your data to core models without editing them:

  ```php
  use Pine\Commerce\Models\Product;

  Product::resolveRelationUsing('tradePrices', fn (Product $p) => $p->hasMany(\App\Models\TradePrice::class));
  ```

- The user model is the one swappable model: `App\Models\User extends Pine\Commerce\Models\User`.
- Raw SQL (`DB::raw`, `whereRaw` …) names tables through `Pine\Commerce\Support\Sql::table()` / `col()` so it keeps
  working on a table-prefixed connection.

## Storefront routes

Add them to `routes/web.php`. The storefront catch-all (pages, products, categories, redirects) is a **fallback**
route, so your routes always win:

```php
Route::get('trade-account', [\App\Http\Controllers\TradeAccountController::class, 'show'])->name('trade.show');
```

Render storefront pages with theme views (`view('trade.show')` resolves through the theme chain, so the view can
`@extends('layouts.app')`).

## Testing

The skeleton is set up so tests can never touch real data:

- `phpunit.xml` forces **in-memory SQLite** (`DB_CONNECTION=sqlite`, `DB_DATABASE=:memory:`) and separate cache files,
  so a production config cache is never used;
- `tests/TestCase.php` **refuses** `RefreshDatabase`, `DatabaseMigrations` and `DatabaseTruncation` unless the
  database is in-memory SQLite;
- run `php artisan config:clear` before `php artisan test` on any server, and never point tests at a live database.

A feature test for worked examples 2 and 3:

```php
<?php
// tests/Feature/TradeApplicationsTest.php

namespace Tests\Feature;

use App\Models\TradeApplication;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TradeApplicationsTest extends TestCase
{
    use RefreshDatabase;   // allowed: in-memory SQLite only (tests/TestCase.php enforces it)

    public function test_staff_can_approve_an_application(): void
    {
        $staff = User::factory()->create(['role' => 'manager', 'is_active' => true]);
        $application = TradeApplication::create(['company' => 'Acme Ltd', 'email' => 'buyer@example.test']);

        $this->actingAs($staff)->get('/admin/trade-applications')->assertOk()->assertSee('Acme Ltd');
        $this->actingAs($staff)->post("/admin/trade-applications/{$application->id}/approve")->assertRedirect();

        $this->assertSame('approved', $application->fresh()->status);
    }

    public function test_guests_are_sent_to_the_login_page(): void
    {
        $this->get('/admin/trade-applications')->assertRedirect();
    }

    public function test_old_applications_expire(): void
    {
        TradeApplication::create(['company' => 'Old Ltd', 'email' => 'old@example.test'])
            ->forceFill(['created_at' => now()->subDays(40)])->save();

        $this->artisan('commerce:schedule:task', ['task' => 'acme.trade-expire'])->assertSuccessful();

        $this->assertSame('expired', TradeApplication::first()->status);
    }
}
```

```bash
php artisan config:clear && composer test
```

Staff roles are `admin` (administrator) and `manager`; customers cannot open the back office. The core's own tests
show every extension point in use: `tests/Feature/ExtensionApiTest.php` in ecom-core.

## Checklist

- [ ] Lives in the shop project – nothing changed in `vendor/`.
- [ ] Uses config, the theme or the extension API – no copied core classes.
- [ ] Works with the `sync` queue and without cron.
- [ ] Plain Blade + Alpine for UI; no admin or e-commerce framework packages; no Node build.
- [ ] Own tables only; raw SQL goes through `Pine\Commerce\Support\Sql`.
- [ ] Optional features wrapped in `commerce_feature()` / `RequireFeature`.
- [ ] Covered by a test; `php artisan commerce:doctor` stays clean.

---

Next: [Admin guide](05-admin-guide.md).
