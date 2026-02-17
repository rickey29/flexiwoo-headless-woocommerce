# Headless WooCommerce — Architecture, Challenges, and Reference Implementation

## What is Headless WooCommerce

Headless WooCommerce is an architectural pattern that separates the frontend rendering layer from the WooCommerce backend. Instead of relying on a PHP theme to generate HTML, a headless setup delegates page rendering to an external service — typically built with a modern framework like Next.js, Nuxt, or Astro — while WooCommerce continues to manage products, customers, orders, carts, checkout, and payments.

In a traditional WooCommerce store, WordPress loads a theme, executes PHP template files, and outputs HTML in a single request cycle. Everything — routing, data retrieval, rendering, and session management — happens inside the same PHP process. A headless architecture breaks this cycle by moving the rendering step to an independent service that receives structured data and returns HTML.

The appeal is straightforward: modern JavaScript frameworks offer better developer ergonomics, component-based design, and performance optimizations like static generation and edge rendering. But WooCommerce was not designed with this separation in mind, and decoupling it introduces real engineering challenges.

---

## Why Traditional WooCommerce Coupling Causes Issues

WooCommerce is deeply coupled to WordPress at the theme layer. Template files, action hooks, and filters assume they execute inside a WordPress request lifecycle with access to global state, PHP sessions, and the full plugin ecosystem. This coupling is not a bug — it is how WordPress themes are designed to work.

The problem emerges when you remove the theme. WooCommerce hooks that fire inside template files stop executing. Plugins that inject markup via `woocommerce_before_cart` or `woocommerce_checkout_order_review` have no execution context. Payment gateways that enqueue JavaScript on the checkout page cannot load their scripts into an external frontend.

This means headless WooCommerce is not simply a matter of calling an API and rendering the response. The transactional behavior of WooCommerce — sessions, nonces, gateway interactions, and plugin side effects — remains tied to the WordPress request, and any headless implementation must account for this.

---

## Common Headless WooCommerce Problems

### Sessions

WooCommerce uses server-side PHP sessions to track cart contents, applied coupons, chosen shipping methods, and customer identity. These sessions are tied to cookies issued by WordPress. When the frontend runs on a different domain or origin, session cookies may not be sent with API requests, causing WooCommerce to treat every request as a new visitor. Bridging sessions across domains requires careful cookie forwarding, nonce synchronization, and CORS configuration.

### Cart Drift

Cart state can diverge between the frontend and WooCommerce. If a customer adds an item on the frontend but a concurrent process — a coupon expiration, an inventory change, or a shipping rate recalculation — modifies the cart on the server, the frontend may display stale data. Without a mechanism to reconcile state, customers can see incorrect totals, missing items, or failed checkout attempts.

### Payment Gateway Redirects

Many WooCommerce payment gateways redirect customers to external payment processors and expect the return URL to be a WordPress-routed page. In a headless setup, the return URL points to the external frontend, which has no access to the WordPress session that initiated the payment. The gateway callback cannot verify the session, and the order may not be marked as paid. Gateways that rely on server-side webhook confirmation are less affected, but client-side redirect flows frequently break.

### Return-to-Checkout Issues

Related to gateway redirects, several checkout flows depend on return URLs that WordPress controls. 3-D Secure verification callbacks, order-pay links for failed payments, and "return to cart" links from payment processors all expect WordPress-routed endpoints with active sessions. A headless frontend must intercept or reconstruct these URLs and restore the correct session context, which requires coordination between the frontend and the WordPress backend.

### Plugin Compatibility

WooCommerce extensions commonly inject functionality through PHP hooks that execute inside theme templates. A headless architecture that bypasses the theme also bypasses these hooks. Plugins that add custom fields to checkout, modify cart display, or inject upsell widgets may appear to be installed and active but produce no visible output. There is no universal solution to this — each plugin's integration must be evaluated individually.

---

## FlexiWoo Architecture Overview

FlexiWoo is a reference implementation that addresses these challenges using a two-component architecture.

**flexi** is a Next.js rendering engine. It receives structured JSON payloads describing a WooCommerce page — product data, cart contents, checkout fields — and renders them into HTML. It is stateless: it does not call WordPress APIs, manage sessions, or process payments. It is a pure rendering service.

**flexi-woo** is a WordPress plugin that intercepts WooCommerce page requests at the `template_redirect` hook. When a customer visits a product page, cart, or checkout, flexi-woo collects the relevant WooCommerce data, sends it to flexi as a POST request, and receives rendered HTML in response. WordPress then serves that HTML to the visitor.

This architecture keeps WooCommerce as the transactional authority. Sessions, nonces, payment processing, and plugin hooks all execute inside WordPress as they normally would. The only thing that changes is where the HTML comes from. WooCommerce does not know or care that the HTML was rendered by an external service — from its perspective, the page was served normally.

---

## How Fallback Works

Fallback is a first-class behavior in FlexiWoo, not an error state. If the flexi renderer is unavailable — due to a deployment, a network issue, or an unhandled template — flexi returns an HTTP 503 with a reason header. flexi-woo interprets this signal and allows WordPress to continue with native theme rendering.

This means a FlexiWoo deployment is never all-or-nothing. If the renderer goes down, customers still see a working store rendered by the existing WordPress theme. Pages can be migrated to headless rendering incrementally, and any page that is not yet supported by flexi simply falls back to the theme.

---

## When to Use Headless WooCommerce

Headless WooCommerce is appropriate when you need full control over the frontend presentation layer — custom design systems, framework-specific tooling, or performance characteristics that a PHP theme cannot provide — while preserving WooCommerce's transactional capabilities.

It is a good fit for development teams and agencies that already work with modern JavaScript frameworks and want to apply those skills to WooCommerce projects. It is less appropriate for store owners who want a plug-and-play solution without frontend development involvement.

FlexiWoo is specifically designed for this audience: developers and agencies who want a self-hosted, fork-friendly starting point rather than a SaaS dependency.

---

## Reference Implementation

FlexiWoo is open source and available on GitHub:

- **flexi (Next.js Renderer):** [github.com/rickey29/flexiwoo-headless-woocommerce](https://github.com/rickey29/flexiwoo-headless-woocommerce)
- **flexi-woo (WordPress Plugin):** [github.com/rickey29/flexi-woo](https://github.com/rickey29/flexi-woo)

For architecture details, see the [README](../README.md). For the boundary between flexi and flexi-woo, see [BOUNDARY.md](BOUNDARY.md).
