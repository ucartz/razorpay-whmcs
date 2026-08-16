## Razorpay Payment Gateway for WHMCS

A Razorpay payment gateway module for WHMCS, developed and maintained by [Ucartz](https://www.ucartz.com).

## Background

This module is inspired by Razorpay's official open-source [razorpay-whmcs](https://github.com/razorpay/razorpay-whmcs) plugin, which we found to be full of long-standing bugs affecting real installations — payments silently failing on modern PHP/WHMCS versions, incorrect amounts being credited to invoices, stale checkout sessions, and more. We rebuilt it: fixed the underlying bugs, hardened it for the latest WHMCS and PHP releases, and added functionality (refunds, dispute/chargeback handling, gateway fee tracking) the original never had.

It has been tested on [Ucartz](https://www.ucartz.com)'s own WHMCS installation and on other WHMCS installations, across a range of payment scenarios (standard checkout, partial payments, late fees, refunds, webhooks).

## Features

- Full compatibility with WHMCS 9 and PHP 8.2–8.3.
- Razorpay Standard Checkout using Razorpay's current recommended integration pattern.
- Accurate invoice crediting — always reflects the amount actually captured by Razorpay for a transaction, never a stale or unrelated invoice value.
- Database-backed order tracking with safe fallback handling, so a payment attempt is never verified against a stale or incorrect order.
- Refunds initiated from the WHMCS admin area reach Razorpay, with configurable refund speed (normal / instant).
- Chargeback/dispute webhook handling — a lost dispute automatically reverses the payment in WHMCS.
- Actual Razorpay gateway fee recorded against each transaction for reconciliation.
- Duplicate-transaction protection and signature verification hardened against replay/tampering.

## Bugs Fixed

The official module had a number of serious, long-standing bugs. Here's everything that was wrong and how it was fixed.

### Payments failing outright on modern PHP / WHMCS

- **Fatal error on PHP 7+**: the callback and webhook handlers called `mysql_fetch_assoc()` / `select_query()` — `mysql_fetch_assoc()` is a native PHP function that was removed from PHP core in PHP 7.0. WHMCS 9 requires PHP 8.2+, so this call would fatal-error on any modern install, breaking every payment. Replaced with proper database queries (Capsule ORM).
- **TLS 1.1 forced on every API call to Razorpay**: the SDK explicitly pinned the connection to TLS 1.1. TLS 1.1 is rejected by most gateways today — PCI-DSS v4.0.1 requires TLS 1.2+ — so this silently broke the underlying connection to Razorpay's API. Fixed to use TLS 1.2.
- **Outdated vendored SDK**: the bundled Razorpay SDK was version 2.8.1 with a HTTP library from 2015. Updated to the latest official release (2.9.3), keeping every other fix intact on top of it.

### Wrong amounts being credited to invoices

- **The most serious bug**: the payment callback credited invoices using the invoice's stored *total*, not the amount actually captured by Razorpay for that specific transaction. If a customer paid only a late fee, or made a partial payment, Razorpay correctly charged the smaller amount — but WHMCS then credited the invoice's full original total regardless, silently over- or under-crediting the account. Fixed to always fetch and credit the exact amount Razorpay actually captured for that payment.
- **Stale Razorpay order reuse**: if a customer opened an invoice, left, and returned after a late fee or credit was applied, the module could reuse a cached Razorpay order created for the old (wrong) amount — including silently reusing it if the API call to re-verify that order failed. Fixed to always create a fresh order whenever the current invoice balance no longer matches the existing order, or whenever that order can't be re-verified.
- **Inconsistent amount rounding**: order creation and the checkout form each rounded the same amount differently, causing floating-point "Amount Mismatch" errors. Both paths now round identically.
- **`receipt` field type rejection**: Razorpay's API requires the `receipt` field to be a string; WHMCS passes invoice IDs as integers, which Razorpay's API rejected outright with `expected string but provided ...`. Fixed by explicitly casting `receipt` and `currency` to string.

### Checkout and signature verification

- **Checkout button not working**: the module used an old, auto-embedding `<script data-*>` checkout pattern that Razorpay's own current documentation says is unreliable — Razorpay requires `checkout.open()` to be triggered by a direct user click. Rebuilt using Razorpay's current recommended Standard Checkout pattern (explicit button + click handler), which is the likely root cause of long-standing "Pay Now button does nothing" reports.
- **Missing response fields**: the checkout form only captured the payment ID from Razorpay's response, never the order ID or signature, so server-side signature verification had nothing to actually verify and would always fail. Fixed by capturing and submitting all three fields.
- **Unreliable session-based verification**: signature verification originally depended on a PHP session set when the invoice page was rendered. Under AJAX-driven "Make Payment" tabs, expired sessions, or 3DS/OTP delays, that session could be lost by the time the callback ran, breaking verification for an otherwise legitimate payment. Verification now prioritizes the order ID Razorpay's own checkout handler returns directly, falling back to the database and then session only if that's missing.
- **Null-pointer risk**: the database-backed order lookup could crash when no matching row existed instead of failing safely.
- **No duplicate-transaction protection**: neither the callback nor the webhook guarded against the same transaction being processed twice.
- **Double slashes in redirect URLs**: a trailing slash on the configured system URL produced broken `//viewinvoice.php` redirect links.
- **Stored-XSS-shaped gap**: client name/email and other dynamic values were concatenated directly into HTML/JS without escaping. Fixed with proper JSON/HTML escaping.

### Missing functionality

- **No refund support**: refunds initiated from the WHMCS admin area only updated the local WHMCS record — they never actually reached Razorpay. Added real refund support via the SDK, with a configurable refund speed (normal / instant).
- **Gateway fee never recorded**: every transaction showed a hardcoded $0 fee, making it impossible to reconcile actual Razorpay settlement amounts. Now records the real fee (including tax) Razorpay deducts, matching the Razorpay Dashboard.
- **Chargebacks/disputes silently ignored**: Razorpay's dispute webhooks were never handled at all — a lost chargeback left the invoice marked Paid and the service running with no reversal and no admin notification. Added full dispute webhook handling: disputes are logged for admin review while open, and a lost dispute automatically reverses the payment in WHMCS (invoice returns to unpaid, standard WHMCS overdue handling takes over).
- **Wrong internal gateway name recorded**: the module passed its human-readable display name ("Razorpay") instead of its internal system name to WHMCS's payment-recording function, causing the "Payment Method" column to show blank on the Transactions list and breaking other internal lookups keyed on the module name.
- **Missing company name and other PHP 8 warnings**: the checkout modal never showed the merchant's company name, and several `$_GET`/`$_POST` accesses and SDK-level deprecation notices were left unguarded on PHP 8.1+. All fixed.

## Installation

1. Ensure your WHMCS installation is on WHMCS 9 (or WHMCS 6–8) with PHP 8.2 or later.
2. Download this repository.
3. Upload the contents of the `modules/` directory into your WHMCS installation's `modules/` directory.

## Configuration

1. Log into WHMCS as administrator (`https://yourdomain.com/admin`).
2. Navigate to **Setup → Payments → Payment Gateways**.
3. Choose **Razorpay** from the Activate dropdown and activate it.
4. Enter your Razorpay **Key Id** and **Key Secret**.
5. Optionally enable webhooks and set the **Webhook Secret** to match what you configure in the Razorpay Dashboard.
6. Click **Save Changes**.

## Support

For issues specific to this module, open an issue in this repository. For general Razorpay account or payment support, visit [https://razorpay.com](https://razorpay.com).

## License

Licensed under the [MIT License](LICENSE). Portions of this software are derived from Razorpay's official [razorpay-whmcs](https://github.com/razorpay/razorpay-whmcs) module and [razorpay-php](https://github.com/razorpay/razorpay-php) SDK, both © Razorpay, used and modified under the terms of the MIT License.
