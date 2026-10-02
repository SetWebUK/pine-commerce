# 5. Admin guide

A short tour of the back office for shop managers. Sign in at `https://your-shop/admin`.

**Contents**

- [Roles](#roles)
- [Getting around](#getting-around)
- [Home (dashboard)](#home-dashboard)
- [Orders](#orders)
- [Products](#products)
- [Customers](#customers)
- [Discounts](#discounts)
- [Content](#content)
- [Inbox](#inbox)
- [Analytics](#analytics)
- [Settings](#settings)
- [Updates](#updates)
- [Everyday tips](#everyday-tips)

---

## Roles

| Role | Can |
|---|---|
| **Administrator** | everything, including payment keys, staff accounts, the System screen and installing updates |
| **Manager** | run the shop day to day: orders, products, customers, discounts, content, reports, most settings |

Staff are managed in **Settings › Staff accounts** (administrators only). Customers cannot open the back office.

## Getting around

- The **sidebar** lists the sections below. Sections for features your shop has switched off are hidden.
- The **search** box at the top finds orders, customers and products.
- Lists have status tabs, filters, sortable columns and bulk actions (tick rows, then choose an action).
- Forms warn you about unsaved changes; messages appear as small notifications ("toasts") in the corner.

## Home (dashboard)

Sales and orders at a glance, orders waiting for you, low-stock and out-of-stock products, recent messages,
abandoned-checkout results, and a notice when a platform update is available. Click any number to open the matching
filtered list.

## Orders

- **All orders** with status tabs: *Pending payment*, *Processing* (paid – ready to fulfil), *On hold* (e.g. awaiting
  a bank transfer), *Completed*, *Cancelled*, *Refunded*, *Failed*.
- **Order screen**: items, totals and VAT lines, customer and addresses, payment details, timeline and notes (private,
  or a note to the customer, which emails them).
- **Change the status** to send the matching email (e.g. *Completed* sends "your order is complete", optionally with
  a tracking link).
- **Refunds**: full or partial; card and PayPal refunds go back through the gateway when it supports it, and stock can
  be returned.
- **Print**: invoice PDF and packing slip.
- **Export** orders to CSV.
- **Abandoned checkouts**: baskets where the shopper typed an email but did not order, the reminders sent and the
  revenue recovered. *Stop reminders* stops emails for one basket.

## Products

- **All products**: simple products and **variable products** (e.g. size × colour) with a price, SKU, stock and image
  per variation. Each product has images and a gallery, categories, attributes, description, sale price with optional
  start/end dates, stock management, weight and shipping class, tax class, related / up-sell / cross-sell products,
  SEO title and description, and a draft / published status.
- **Categories**: a tree (drag to reorder), each with an image, description and SEO fields. The category path is the
  URL.
- **Attributes**: global attributes (Size, Colour, Brand …) and their values; choose which ones appear as **shop
  filters**.
- **Reviews**: approve, reply ("Response from {store}") or remove.
- **Stock alerts**: customers waiting for a back-in-stock email.
- **Inventory**: change stock and prices for every product and variation in one list (each change saves as you
  leave the box), or import stock and prices from a file.
- **Import / Export**: the whole catalogue as a CSV (WooCommerce product exports are accepted). Export first as a
  backup; every import starts with a **dry run** that shows what would change.

## Customers

Customer accounts with their orders, addresses, total spent and notes. Customers imported from WooCommerce sign in
with their old passwords.

## Discounts

Coupon codes: a percentage, a fixed basket discount or a fixed discount per product, optionally with free shipping; start and end dates, minimum and
maximum spend, usage limits (in total and per customer), included or excluded products and categories, "exclude sale
items", "cannot be combined with other codes", and codes restricted to particular email addresses.

## Content

- **Pages**: a rich-text editor plus page templates with blocks (the home page, landing pages …). Each page has its own
  SEO fields. *Home* holds the home page sections.
- **Blog posts** and **Blog categories**.
- **Menus**: the header, mobile and footer menus. Labels can contain `{store.phone}`, `{store.email}`, `{store.name}`
  and update automatically when the store details change.
- **Media**: the image library. Uploaded images get every size the theme needs (and WebP versions) automatically.
- **Redirects**: send an old URL to a new one (301), or mark it as gone (410). CSV import available. Add one whenever
  you delete or rename a page, product or category that people may have linked to.

## Inbox

**Form submissions** from the contact form, and **Newsletter** sign-ups (export them for your mailing tool).

## Analytics

Sales for any date range (optionally compared with an earlier one): gross and net sales, discounts, shipping, tax,
refunds, orders and average order value over time, top products, sales by category and by payment method, new vs
returning customers, and tax by rate – with CSV export.

## Settings

| Screen | What is in it |
|---|---|
| Store details | name, logo, favicon, contact details, address, legal details, social links, announcement bar, footer text |
| Checkout & orders | guest checkout, account creation, terms page, order number sequence |
| Tax | prices entered with or without VAT, how prices are shown, tax classes and rates (CSV import from WooCommerce) |
| Payments *(administrators)* | Stripe, PayPal, bank transfer and any custom gateways – keys are stored encrypted |
| Shipping | countries you sell to, shipping zones (first matching zone wins; postcode patterns such as `BT*`), delivery options and shipping classes |
| Emails | sender, who receives order notifications, which emails are sent |
| Invoices | invoice numbering, PDF on order emails, customer downloads, paper size, notes and footers |
| Abandoned carts | reminder emails: who may receive them, up to three emails, optional discount codes (off until you switch it on) |
| Scheduled tasks | background jobs, their last result, low-stock email, how long abandoned guest baskets are kept |
| SEO & tracking | default titles and descriptions, robots.txt, "Ask search engines not to index this site", Google Tag Manager / GA4, Google Shopping feed |
| Theme | logo, colours, fonts and corners of the storefront theme; preview another theme privately |
| Staff accounts *(administrators)* | administrators and managers |
| System *(administrators)* | platform version, active theme, feature switches (read-only), health checks, last import report |

Your developer may have added more screens (they appear in the same list).

## Updates

*Administrators only.* **Updates** in the sidebar shows a badge when a new platform release is available.

1. Open **Updates** and read the release notes. Anything marked **Client actions required** is for your developer.
2. Press **Approve & install**, re-enter your password and tick the confirmation.
3. The update runs in the background while the page shows its progress. The shop is in maintenance mode for a minute
   or two; you can keep using it through the bypass link shown on the page.
4. A database backup is taken first. If anything fails, the previous version is restored automatically and the run
   is marked *failed* with its log – send that to your developer.

Releases that need a developer (a new major version) are shown but cannot be installed from here. **Project files
from the skeleton** lists starter-app files that can be updated safely; leave anything marked "needs a developer" to
your developer.

## Everyday tips

- Test a new payment method, delivery price or coupon with a real checkout before you announce it.
- Changing a product or page URL? Add a redirect from the old one.
- Before going live: untick "Ask search engines not to index this site" (Settings › SEO & tracking).
- Settings › System shows the health checks; anything marked FAIL needs attention.
