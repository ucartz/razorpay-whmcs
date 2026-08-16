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
