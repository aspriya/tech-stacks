# Tip Jars & Donations — Sri Lanka Founder Edition

> **Purpose of this document.** This is a decision aid for adding **optional, voluntary support** to a product — the "buy me a coffee" / tip jar / donation flow where users *can* contribute but aren't paying for anything they get in return. It accounts for the constraint that the founder is in **Sri Lanka** and ultimately wants funds to land in a personal Sri Lankan bank account.
>
> This document pairs with the existing tech-stack and payment-processing docs. Read those first if you haven't.
>
> **Last researched: April 2026.** Country-support and platform availability change frequently — verify on the provider's site before committing.

---

## 0. Why This Is a Separate Decision from "Payment Processing"

Tipping is a different product than a SaaS subscription. The bar for the user is "would I drop $5 into a tip jar without much thought" — friction kills it. So the priorities are different:

| SaaS payments | Tip jars |
|---|---|
| Reliability + compliance > convenience | Convenience > everything |
| Customers expect formal checkout | Supporters expect one-click |
| MoR pays for itself in tax savings | MoR fees may be too high % at small amounts |
| Recurring billing matters | One-time dominant; recurring is a bonus |
| Branded checkout | Trust signal of recognized platform sometimes helps |

For your situation, you actually have **two viable paths** for tips:
1. **Use the same MoR (Polar) you're using for paid SaaS** — adds a "Pay what you want" product. Zero new accounts, zero new infrastructure.
2. **Use a dedicated tip platform** — gives you a recognized public profile (`buymeacoffee.com/yourname`) that lives outside your product.

Most likely you want both for different surfaces. Read on.

---

## 1. The Sri Lanka Reality Check (Read Before Anything Else)

Most popular tip-jar platforms quietly stopped supporting Sri Lanka because they all sit on top of Stripe, and Stripe doesn't onboard Sri Lankan merchants. Before recommending anything, here's the actual current state as of April 2026:

| Platform | Sri Lanka status | Why |
|---|---|---|
| **Buy Me a Coffee** | ❌ Not viable | Stripe-only since 2024. Dropped Payoneer. No Sri Lanka support. |
| **Ko-fi** | ❌ Not viable | Stripe or PayPal-receive only. Stripe is closed to LK; PayPal-receive is still send-only in LK as of Feb 2026. |
| **Patreon** | ⚠️ Limited | Payouts via Stripe or PayPal, both blocked for LK receive. Some report Payoneer still works in select cases — verify on signup. |
| **PayPal direct** | ❌ Receive blocked | "Send only" in Sri Lanka. Government announced "final stage" of inward remittances in Feb 2026 but no working timeline. Don't depend on it. |
| **GitHub Sponsors** | ✅ **Officially supported** | Sri Lanka added to supported regions. Uses Stripe Connect Express (different product, wider coverage). |
| **Polar (Pay what you want)** | ✅ **Officially supported** | Sri Lanka on Polar's payout list. Same MoR you'd use for SaaS. |
| **Gumroad (tips/PWYW)** | ⚠️ Verify | Pays out via PayPal in 200+ countries and direct bank in some. PayPal-receive limitation still applies. |
| **Stripe (direct tip page)** | ❌ Not viable | Stripe doesn't onboard LK merchants. |

**Bottom line:** the platforms most foreigners would recommend (Buy Me a Coffee, Ko-fi) **don't work for you**. The two that genuinely do are **Polar** and **GitHub Sponsors**. Everything else is workaround territory.

---

## 2. Recommended Setup — Two-Tier Strategy

Use both, for different audiences:

### Tier 1 — Inside your product: Polar "Pay what you want"

For supporters who are already using your product and want to chip in, build the tip flow into your product itself using Polar. You're already integrated for SaaS billing; tipping is one more product type.

**Why this works well:**
- No new accounts to set up — same Polar org, same payouts, same dashboard
- Polar natively supports **"pay what you want" pricing** — supporters pick the amount
- Same Sri Lanka payout rails (Stripe Connect Express → Wise/local bank)
- Money lands with the rest of your revenue; one P&L, one tax filing
- Same code patterns the agent already uses for SaaS — just a different `productId`

### Tier 2 — Public, outside your product: GitHub Sponsors

For people who follow you on GitHub, see your open source work, or want to support you separately from any specific product. This is where a recognized, hosted profile helps — supporters trust GitHub, the URL is shareable (`github.com/sponsors/yourname`), and you don't have to host anything.

**Why both, not just one:**
- Polar lives inside *each product* — its tips are scoped to that product's audience
- GitHub Sponsors is a *public profile* — works for blog readers, Twitter followers, OSS users who don't even know which products you've shipped
- Different audiences donate through different surfaces. Using both maximizes coverage with minimal overhead.

---

## 3. Option Deep-Dive: Polar (Tier 1)

### What it is
The same Merchant of Record you'd use for SaaS, configured with a **"Pay what you want"** product. Renders as a normal Polar checkout where the supporter types in any amount.

### Sri Lanka status
✅ Confirmed. Polar's payout supported-countries page explicitly lists Sri Lanka. Polar uses Stripe Connect Express for payouts, which has wider country support than Stripe Payments.

### Pricing
Same as Polar's standard SaaS pricing:
- 4% + $0.40 per transaction (base)
- +1.5% on international cards
- No additional fee for one-time / PWYW vs subscriptions

For a $5 tip from an EU customer, that's roughly $0.83 in fees → you keep ~$4.17 before VAT. For a $20 tip, fees are ~$1.51 → you keep ~$18.49. (Polar handles the VAT remittance as MoR.)

### Setup steps
1. In your existing Polar dashboard, **Products → New Product**
2. Choose **"Pay what you want"** as the price type
3. Set a minimum amount (e.g., $1) and an optional preset (e.g., $5)
4. Configure benefits to be empty — supporters get nothing in return; this is genuinely a tip
5. Save → you get a `productId`
6. Build a checkout link or button in your Next.js app that points to a Polar checkout for that product

### Implementation pattern (Next.js + Cloudflare)

The agent should use **the exact same patterns from the payment-processing doc**, with one product:

- Create a Server Action or Route Handler that calls Polar's checkout API for the tip product ID
- The success URL returns the user to a "Thank you" page in your product
- Webhooks update D1 (optional — for tracking who tipped, you may not need this for an anonymous tip)
- All secrets (`POLAR_ACCESS_TOKEN`, `POLAR_WEBHOOK_SECRET`) via `wrangler secret put`

A minimal tip button:

```tsx
// app/tip/route.ts
import { Checkout } from "@polar-sh/nextjs";

export const GET = Checkout({
  accessToken: process.env.POLAR_ACCESS_TOKEN!,
  successUrl: "/thanks?checkout_id={CHECKOUT_ID}",
  // server: "sandbox" while testing
});
```

```tsx
// in any component:
<a href="/tip?products=YOUR_PWYW_PRODUCT_ID">☕ Tip the dev</a>
```

That's it. The Polar Next.js adapter handles the redirect to checkout.

### Pros
- Zero net-new infrastructure if you already use Polar for SaaS
- Same payout rails — one Wise account, one tax filing
- PWYW feels native to supporters and lets them pick amount
- Lowest fees of any tip platform that works for Sri Lanka
- Supports recurring (set the product as monthly subscription with PWYW for "support every month")

### Cons
- No public hosted page like `buymeacoffee.com/yourname` — it lives inside your app
- You build the UI yourself (though it's just a button + checkout redirect)
- No leaderboard / supporter wall feature out of the box

### When to use this tier
- Supporter is already in your product
- You want maximum control of the UX and minimum fees
- You want one consolidated payout pipeline

---

## 4. Option Deep-Dive: GitHub Sponsors (Tier 2)

### What it is
GitHub's native sponsorship platform. You get a profile page at `github.com/sponsors/<your-handle>` where anyone can sponsor you one-time or monthly. GitHub handles checkout, tax (in supported jurisdictions), and payout.

### Sri Lanka status
✅ **Officially supported.** GitHub Sponsors expanded to Sri Lanka in late 2023 and Sri Lanka remains on the current supported-regions list as of April 2026.

### Pricing
- **GitHub charges 0% platform fee** — every dollar a sponsor gives, you receive (minus payment processing)
- Payment processing is handled via Stripe Connect Express (same Sri Lanka-compatible product Polar uses)
- No monthly fees, no setup costs

This is genuinely the best tip-jar economics available to you.

### Setup steps
1. Go to `github.com/sponsors`
2. Click "Join GitHub Sponsors"
3. Provide identity verification (KYC)
4. Set up a Stripe Connect Express account for payouts (GitHub walks you through it)
5. Configure your sponsor tiers (e.g., $1/mo, $5/mo, $25/mo, plus one-time amounts)
6. Optionally describe what sponsors get (early access, name in README, just "thanks!" — anything is fine)
7. GitHub reviews and approves (typically a few days)
8. Your profile goes live at `github.com/sponsors/yourname`

### Integration with your stack
You don't really integrate GitHub Sponsors into a product — it's a separate public profile. Common patterns:

- Add a "Sponsor" badge/link to your **GitHub README**
- Add a sponsor link to your **product's footer or about page** (just a regular `<a href>`)
- Add to your **Twitter bio**, blog, or personal site

You can use the [GitHub Sponsors webhook](https://docs.github.com/en/sponsors/integrating-with-github-sponsors/configuring-webhooks-for-events-in-your-sponsored-account) to be notified when someone sponsors, but for most use cases the dashboard is enough.

### Pros
- **Zero platform fee** — significantly cheaper than Polar at small amounts
- Hosted profile page — no UI to build
- Trust signal — supporters know GitHub
- Sri Lanka officially supported
- Works for OSS, side projects, blog readers — a public surface that doesn't tie to one product
- Recurring monthly support is built-in
- Optional Patreon link if you also have a Patreon (GitHub partnered with Patreon for sponsorship recognition)

### Cons
- Requires you to have a GitHub presence — best for technical creators
- Less direct branding than Polar (it's GitHub's UI, not yours)
- Approval can take several days
- Audience skews developer-heavy

### When to use this tier
- For your *public* / OSS / personal-brand surface, not embedded in any one product
- When you specifically want zero platform fees on small tips
- When the audience is technical and already on GitHub

---

## 5. Alternatives Considered (and Why I'm Recommending Against Them)

These come up in every "tip jar" guide, so worth saying explicitly why they don't fit your situation.

### Buy Me a Coffee — ❌ Don't use
Switched to Stripe-only payouts in 2024 and dropped Payoneer support. Stripe doesn't onboard Sri Lankan merchants. Even though they don't publish a country-blocklist clearly, the practical effect is no payout option for you. There are documented cases of Sri Lankan / Ukrainian / Indian creators losing access to accumulated funds when this transition happened. **Avoid.**

### Ko-fi — ❌ Don't use (currently)
Ko-fi requires a working Stripe or PayPal-receive account. Stripe is closed for LK; PayPal in Sri Lanka is currently send-only. If/when PayPal inward remittances actually go live (Sri Lanka government has been saying "final stage" for months), Ko-fi might become viable. **Re-evaluate later, not now.**

### Patreon — ⚠️ Avoid for new setups
Payouts via Stripe / PayPal / Payoneer. Stripe and PayPal-receive don't work for LK. Payoneer support has been inconsistent. Some Sri Lankan creators report it works; others don't. **Not reliable enough for a primary recommendation in 2026.** If you already have a Patreon working, fine — don't break it. Don't start there.

### Gumroad (tips / pay-what-you-want) — ⚠️ Verify before committing
Gumroad pays out via PayPal in most regions and direct bank in some. With PayPal-receive blocked in Sri Lanka, verify your specific payout option **before** publishing a Gumroad page. Even if it works, fees are 10% + $0.50 per transaction (vs Polar's 4% + $0.40), which is poor for tips.

### Stripe direct (custom tip page) — ❌ Not viable
Sri Lankan merchants can't open Stripe accounts. This option only exists for founders in Stripe-supported countries.

### "Just put a Wise/Payoneer link" — ❌ Don't do this
- Sharing your Wise/Payoneer account details publicly is unsafe and unprofessional
- There's no checkout flow — supporter has to manually initiate a transfer
- Conversion is terrible for one-time small amounts
- No tax handling, no records

### Crypto (tips via wallet address) — ❌ Avoid
- Hostile UX for non-crypto supporters
- Off-ramping to LKR involves the same banking issues plus extra steps
- Sri Lanka's central bank discourages crypto transactions
- You'd be solving a payment problem with a worse payment problem

---

## 6. Money Flow (How LKR Lands in Your Bank)

The good news: both recommended options use **the same payout pipeline** as your SaaS revenue from Polar, so you don't need a new flow.

```
Supporter → Polar (or GitHub Sponsors) → Stripe Connect Express
  → Wise USD account (or Payoneer, or LK USD account)
  → Wise FX (~mid-market, ~0.5%) → LKR
  → Sri Lankan bank (Commercial / HNB / Sampath / DFCC)
```

If you've already set up the SaaS payout pipeline per the payment-processing doc, you don't need to do anything new — tips flow through the same plumbing.

**Tax note:** these are still business / self-employment income for IRD purposes. Track them in your accounting just like SaaS revenue. Polar gives you payout reports; GitHub Sponsors does too. Don't co-mingle "tips" with personal gifts in your reporting just because the spirit feels different — IRD treats it as income.

---

## 7. Decision Tree

For each new product or surface where you want to add a "support me" option, walk this:

1. **Is this surface a public personal/OSS profile, or your blog/Twitter?**
   → **GitHub Sponsors.** Zero fees, hosted, Sri Lanka officially supported.

2. **Is this inside a product where users are already engaged?**
   → **Polar PWYW product.** Reuse your existing Polar setup. Zero new infrastructure.

3. **Both (most common case for indie founders)?**
   → Set up **both.** Polar for in-product, GitHub Sponsors for public profile. Link to each from the appropriate surface. They don't compete — different audiences hit different surfaces.

4. **Edge case: you're not on GitHub and don't have a SaaS yet, just want a public tip page?**
   → Wait for PayPal Sri Lanka inward remittances to actually launch (re-check status), then revisit Ko-fi. Until then, set up Polar even just for tips — you'll want it for paid SaaS eventually anyway.

---

## 8. Implementation Notes for the Coding Agent

When the agent is wiring up tip flows in Next.js + Cloudflare:

### For Polar PWYW
- Create one product per "tip context" if you want to track them separately (e.g., one for "general tip", one for "thanks for X feature"). Or just use a single tip product and free-form attribution.
- **Reuse the Polar webhook handler** from the payment-processing setup. Add a check: if `productId === TIP_PRODUCT_ID`, route to a different handler (e.g., log to analytics, but don't grant entitlements).
- The minimum for PWYW is recommended at **$1–$3** to avoid spam/test-card abuse. Polar's checkout will enforce whatever minimum you set.
- For one-time tips, don't bother creating a customer record in D1 unless you specifically want a "thanks, [name]" wall. Keep it stateless.

### For GitHub Sponsors
- No code integration needed for the basic case. It's an external link.
- If you want notifications, set up a GitHub Sponsors webhook → forward to a Cloudflare Worker that pushes to a Cloudflare Queue → consumer logs/notifies you.
- A small "Sponsors" badge on the marketing page using the GitHub Sponsors logo + link is enough.

### Where to expose tip links

Conventional locations for indie SaaS:
- **Footer** — a small "💖 Sponsor" or "☕ Tip" link
- **Empty states** — "Built by one person — if this saved you time, [tip]"
- **Settings page → About section** — for engaged users
- **Email signature on transactional emails** — gentle, not pushy
- **404 page** — quirky, classic spot
- **README of your GitHub repos** — links to GitHub Sponsors

Don't put it in the main nav or behind primary CTAs. The whole point is it should feel optional and gentle.

### What NOT to do
- Don't gate features behind "tipping" — that's a subscription, not a tip. Use Polar's normal SaaS pricing for that.
- Don't auto-prompt or modal-spam tip requests — kills goodwill instantly.
- Don't use the same product ID for tips that you use for paid features. Separate `productId`s let you keep MRR analytics clean.
- Don't store supporter names/emails for "tips wall" without an explicit opt-in checkbox — privacy law (GDPR) still applies even when supporters seem to be giving freely.

---

## 9. Quick Comparison Table

| Criterion | Polar PWYW | GitHub Sponsors |
|---|---|---|
| Sri Lanka payouts | ✅ | ✅ |
| Platform fee | 4% + $0.40 (+1.5% intl) | **0%** |
| Where it lives | Inside your product | Public GitHub profile |
| Setup time | Hours (if Polar already used) | Days (KYC review) |
| Recurring tips | ✅ | ✅ |
| Hosted page | No (you build the button) | Yes (`github.com/sponsors/you`) |
| Audience fit | In-product users | Devs, OSS users, public followers |
| Tax handling | MoR (Polar handles VAT) | You're responsible (no MoR) |
| Best for | Embedding tip flow in a SaaS | Public personal-brand support |

The 0% fee on GitHub Sponsors looks dramatic, but note: GitHub Sponsors is **not** an MoR, so you may need to handle VAT/sales-tax obligations yourself if your sponsor revenue grows large. For typical indie volumes this is rarely material; for a Sri Lankan founder reporting business income to IRD, you treat it as ordinary export income.

---

## 10. Summary

> **Default recommendation:** Set up **both**.
> - **Polar PWYW** for an in-product tip flow. Reuse your existing Polar Next.js + Cloudflare integration; just a new product with "pay what you want" pricing. ~5 minutes of dashboard work and a button in your UI.
> - **GitHub Sponsors** for a public, hosted, zero-fee profile. Sri Lanka is officially supported. Link from your README, blog, Twitter, and product footer.
>
> **Avoid:** Buy Me a Coffee, Ko-fi (until PayPal-receive opens up in Sri Lanka), Patreon for new setups, anything Stripe-direct, anything that requires PayPal-receive in Sri Lanka.
>
> **Stay informed:** Sri Lanka's PayPal inward-remittance situation is "in final stage" as of February 2026. If/when it actually launches, the tip-jar landscape changes — Ko-fi becomes viable, Buy Me a Coffee remains dead due to Stripe dependency. Re-check this section every 6 months.

---

*Last updated: April 2026. Tip platform support and Sri Lanka payment infrastructure are both moving — verify on each provider before final commitment.*
