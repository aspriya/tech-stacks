# Payment Processing for Global SaaS — Sri Lanka Founder Edition

> **Purpose of this document.** This is a decision-aid for picking a payment processor for a new SaaS product. It accounts for the specific situation of a **Sri Lanka–based founder selling globally**, where the founder ultimately wants funds to land in a personal Sri Lankan bank account. It pairs with the tech stack and Cloudflare automation docs.
>
> **Last researched: April 2026.** Pricing and country-support lists change frequently — verify on the provider's site before committing.

---

## 0. The Core Constraint (Read This First)

**Stripe does not support Sri Lanka.** Stripe currently onboards businesses in only ~46 countries, and Sri Lanka is not on the list. This rules out Stripe as a direct merchant — and rules out anything built directly on top of Stripe that requires you to have your own Stripe account.

The right architecture for this situation is a **Merchant of Record (MoR)** model. Read the next section before looking at any provider.

---

## 1. Merchant of Record (MoR) vs. Payment Gateway — The Most Important Distinction

There are two fundamentally different models. Picking the right model matters more than picking the right brand within a model.

### Payment Gateway (e.g., Stripe direct)
You are the legal seller. You collect the customer's money. You are responsible for:
- Registering your business in your country (and possibly in customers' countries)
- Calculating and remitting **VAT, GST, and sales tax** in every jurisdiction where you sell
- Handling **chargebacks and disputes**
- Submitting **KYC / business documents** to the processor
- Managing **fraud risk** and chargeback ratios

This is what people mean when they say "Stripe is a lot of paperwork." For a Sri Lankan founder, this path is effectively closed because Stripe won't onboard you anyway.

### Merchant of Record (e.g., Paddle, Polar, Lemon Squeezy, Dodo)
The MoR is the legal seller. You sell *to the MoR*; the MoR sells to your end customer. The MoR handles:
- Global sales tax / VAT / GST collection and remittance
- Chargebacks and fraud
- Customer-facing invoices and refunds
- Currency conversion
- Compliance with foreign payment regulations

You just receive a single net payout in your chosen currency. **For a Sri Lankan founder selling globally, MoR is the correct choice.** It bypasses the need for a foreign legal entity, foreign bank account, or complex tax filings in 30+ countries.

The tradeoff: MoR fees are higher than raw gateway fees (typically 4–5% + ~$0.40 vs. Stripe's ~2.9% + $0.30), but you save dozens of hours of compliance work and eliminate tax-filing risk.

---

## 2. Sri Lanka-Specific Considerations

A few things that matter in your context that most generic guides skip:

1. **Sri Lanka is on Polar's official supported payouts list.** Confirmed on `polar.sh/docs/merchant-of-record/supported-countries` as of April 2026. Polar uses **Stripe Connect Express** (different product from Stripe Payments) for payouts, which has wider country coverage than Stripe Payments. This is the cleanest path.
2. **Paddle pays out globally** with the only exclusions being sanctioned countries — Sri Lanka is fine.
3. **Lemon Squeezy** offers ~79 countries via direct bank payout *plus* PayPal payouts to 200+ countries. Verify Sri Lanka direct-bank status on signup; PayPal is the safe fallback.
4. **Dodo Payments** is a strong fit for solo founders/indie hackers globally; specifically courts founders in markets where Stripe is closed. Confirm Sri Lanka payout on signup.
5. **Sri Lankan Central Bank** allows USD inward remittances up to USD 200,000/year without significant friction for individual exporters of digital services. Most founders will operate well under this ceiling.
6. **Wise and Payoneer are the standard rails** for Sri Lankan freelancers/SaaS founders to receive USD and convert to LKR. Many MoRs payout to Wise/Payoneer, which then settle to your local bank — this is often cheaper and faster than direct bank wire.
7. **Sri Lankan Inland Revenue (IRD)** treats SaaS export income as self-employment / business income, taxable above the personal threshold. Whatever payout currency you receive, you'll need to report it. Track all payouts; get a basic tax-compliant accounting setup early.
8. **Local payment gateways** (PayHere, Onepay, etc.) are useful only if you're selling primarily to Sri Lankan customers. They're not a substitute for an international MoR.

---

## 3. The Shortlist: Three MoRs Worth Choosing Between

For a Sri Lankan founder building a global SaaS in 2026, the realistic options narrow down to three. Everything else is either unavailable, materially worse, or designed for a different use case.

---

### Option 1: Polar — Recommended Default

**One-liner:** Developer-first MoR with Sri Lanka explicitly supported, the cleanest API, and the cheapest base rate among MoRs.

**Pricing**
- 4% + $0.40 per transaction (base)
- +1.5% surcharge on international card transactions
- +0.5% surcharge on subscription payments
- $15 per chargeback (industry standard)
- Payout fees passed through from Stripe Connect (no Polar markup)
- No monthly fees, no setup fees

**What it includes**
- Full MoR — handles VAT, GST, US sales tax globally
- Subscriptions, one-time purchases, **usage-based billing**, **license keys**, **GitHub repo / Discord access entitlements**
- Open-source platform (`polarsource/polar` on GitHub)
- Clean REST API + SDKs for Next.js, React, Python, Go
- Customer portal, refunds, dunning, analytics
- Sandbox environment

**Setup signup → first payment**
1. Sign up at `polar.sh` with GitHub or email
2. Complete KYC (typically <1 week review)
3. Connect a Stripe Connect Express account in Sri Lanka for payouts (Polar walks you through this; you don't need a regular Stripe account)
4. Create products in dashboard or via API
5. Embed checkout via SDK / hosted checkout / overlay
6. Configure webhook for entitlement provisioning

**Pros for your situation**
- **Sri Lanka is on the official supported list** — this is the single biggest factor
- Lowest fees among full MoRs
- Best developer experience among MoRs (clean SDKs, good docs, MCP integration)
- Fits the Next.js + Cloudflare stack naturally
- Strong fit if you're building developer tools or AI products (built-in usage billing)

**Cons**
- ~120 payout countries (vs Dodo's ~220) — irrelevant for you, matters if you ever hire global affiliates
- Newer platform; some users report slower support response than incumbents
- Open-ended clause reserving the right to pass on future Stripe fee changes

**Use Polar when:** You're building a SaaS for global customers, you want clean Next.js integration, you're comfortable with the MoR model, and you want the lowest cost path that explicitly supports Sri Lanka.

---

### Option 2: Paddle — The Established Enterprise Choice

**One-liner:** The most mature MoR, broadest country coverage for payouts, slightly more enterprise-feeling.

**Pricing**
- 5% + $0.50 per transaction
- No additional fee for international sales (this is genuinely useful)
- No monthly fees

**What it includes**
- Full MoR — taxes, compliance, fraud, chargebacks all handled
- Subscriptions with proration, dunning, retries
- 200+ countries supported (sellers and buyers)
- Hosted checkout + Paddle.js for inline integration
- ProfitWell Metrics included
- Invoice billing for higher-value B2B

**Setup signup → first payment**
1. Sign up at `paddle.com` — note this requires more business documentation than Polar/Dodo, including a website and business description
2. Approval review (typically 1–3 business days for established products, longer for very early-stage)
3. Configure products, prices, and tax categories
4. Integrate via Paddle.js (inline) or hosted Paddle Checkout
5. Set up payout method — Paddle pays out in many currencies; for Sri Lanka, USD via wire to a local USD account or to Wise/Payoneer is common

**Pros**
- Most mature platform; battle-tested by thousands of SaaS companies
- Pays out to virtually every non-sanctioned country including Sri Lanka
- No international transaction surcharge
- Strong subscription tooling

**Cons**
- Higher base fee than Polar
- More documentation required at signup (less "ship today" than Polar/Dodo)
- Approval can be a hurdle for very early products with no MRR or domain
- Customer support quality has been mixed in recent reviews

**Use Paddle when:** You have an established product (or polished landing page + clear pitch), you sell to B2B at higher price points where the 5% feels less material, or you specifically want the most established player.

---

### Option 3: Dodo Payments — The Indie-Founder-Friendly Newcomer

**One-liner:** Built explicitly for solo developers, indie hackers, and founders in markets where Stripe is closed. Exceptional onboarding speed.

**Pricing**
- 4% + $0.40 per transaction (matches Polar)
- +1.5% on international transactions (similar to Polar)
- 226 countries for accepting payments
- $5 fee on payouts under $1,000 (otherwise free)
- No monthly fees, no setup fees
- No per-chargeback surcharge (uses RDR — Rapid Dispute Resolution — to handle disputes pre-chargeback)

**What it includes**
- Full MoR — taxes, GST, VAT, US sales tax across 190+ jurisdictions
- 25–30+ local payment methods (UPI, PIX, SEPA, Alipay, etc.) — better local checkout in many emerging markets than Polar
- Subscriptions, one-time, **usage-based billing**, license keys
- REST API + SDKs, hosted checkout, overlay checkout
- Fast onboarding — many founders report verified-and-live within an hour

**Setup signup → first payment**
1. Sign up at `dodopayments.com`
2. Submit basic business info (KYC) — typically minutes-to-hours, not days
3. Configure products and prices
4. Integrate via SDK or hosted checkout
5. Set payout destination — confirm Sri Lanka at signup (their supported list is broader than Polar's; they pay out to most major markets)

**Pros**
- Fastest signup of the three (designed for it)
- Rate matches Polar's, with better local payment method coverage in some emerging markets
- Aggressive about supporting founders in markets traditional players exclude (heavy India/SEA presence)
- No per-chargeback fee (small but adds up)
- Active, responsive team (founders frequently respond on X / Discord)

**Cons**
- Newer than Polar and Paddle; less battle-tested at high scale
- Confirm Sri Lanka payout explicitly before committing — public docs list 226 countries for **payment acceptance** but the **payout** country list isn't as cleanly published as Polar's
- Smaller ecosystem (fewer third-party tools and integrations than Paddle)
- $5 small-payout fee can sting at very low volumes

**Use Dodo when:** You want to ship today, you value fast onboarding above all, you sell into Asia-Pacific markets where local payment methods matter, or your product is extremely early and you don't want to argue with Paddle's approval team.

---

## 4. Quick Comparison Table

| Criterion | Polar | Paddle | Dodo Payments |
|---|---|---|---|
| Base fee | 4% + $0.40 | 5% + $0.50 | 4% + $0.40 |
| International surcharge | +1.5% | $0 (included) | +1.5% |
| Subscription surcharge | +0.5% | $0 | $0 |
| Chargeback fee | $15 | included | $0 (uses RDR) |
| Sri Lanka payouts | ✅ Listed | ✅ Global ex-sanctions | ✅ (verify on signup) |
| Onboarding speed | Days | Days–weeks | Hours |
| Best for | Default for your stack | Established SaaS, B2B | Solo / indie / very early |
| Maturity | Medium | High | Medium-Low |
| Local payment methods | Card-heavy | Broad | Broadest (25–30+) |
| Developer experience | Excellent | Good | Good |
| Open source | Yes | No | No |
| Recommended if | You want lowest fees + clean DX + explicit SL support | You're past early stage and value maturity | You want to ship today |

---

## 5. Lemon Squeezy — Why It's Off the Shortlist

Lemon Squeezy was a strong option historically and may still appear in many guides, but **Stripe acquired Lemon Squeezy in July 2024**, and as of 2026 the platform is now in a transitional phase tied to Stripe Managed Payments. The team has acknowledged slower feature development and reduced support during this transition.

For a new SaaS in 2026, this introduces unnecessary risk:
- Roadmap is uncertain
- Pricing alignment with Stripe Managed Payments (3.5% surcharge on top of Stripe fees) may shift Lemon Squeezy's economics
- Polar's open-source positioning has filled the indie/developer niche Lemon Squeezy used to own

If you have an existing Lemon Squeezy account, it still works. For new products, pick Polar or Dodo instead.

---

## 6. The Money Flow: What Your LKR Bank Receives

This is the part most guides skip. Here's how money actually flows from a customer in Germany to your account in Colombo:

```
[Customer in DE] 
    pays €20 via card 
    → MoR (Polar/Paddle/Dodo) 
        receives, deducts VAT (≈€3.20), 
        retains processing fee (~€0.85), 
        balance ≈ $17.50 USD held for you
    → on payout schedule (e.g., monthly), MoR sends USD via:
        • Stripe Connect (Polar)
        • Wire / Wise / Payoneer (Paddle, Dodo)
    → arrives at your USD-receiving rail:
        • Wise USD account (most cost-effective)
        • Payoneer USD account
        • Direct USD account at Sri Lankan bank (e.g., Commercial Bank, NTB, HNB)
    → you convert USD → LKR
        • Wise: ~mid-market rate, ~0.5% fee — best
        • Payoneer: ~2% fee
        • Local bank: ~3–4% spread + wire fees
    → final LKR balance lands in your personal/business savings account
```

**Recommended setup for Sri Lanka:**
1. Open a **Wise account** (free, fully online). Get USD, EUR, and GBP "receiving accounts" — these are essentially virtual local bank accounts in those currencies that you can give to your MoR.
2. Configure your MoR to pay out to your Wise USD account.
3. From Wise, convert USD → LKR at near-mid-market rates and transfer to your Sri Lankan bank.

Alternatively, **Payoneer** works similarly and integrates more smoothly with some platforms (it's a default option in Lemon Squeezy / Paddle in some regions). Wise is generally cheaper for FX conversion; Payoneer is sometimes more convenient for marketplace integrations.

**Sri Lankan bank choice:** Commercial Bank, HNB, Sampath Bank, and DFCC all handle USD inward remittances reasonably well. For higher volumes, look into a USD savings account (Foreign Currency Banking Unit) so you can hold USD without forced conversion.

---

## 7. Decision Tree

For each new SaaS product, walk this:

1. **Is your customer base mostly Sri Lankan?**
   → Use a Sri Lankan local gateway (PayHere, Onepay) for LKR collection. Skip the rest of this doc. (Note: this is rare for a SaaS with global ambitions.)

2. **Is the product live with paying customers and stable?** Do you want the most mature, lowest-risk option?
   → **Paddle.** Eat the higher fee for the maturity premium.

3. **Is this a brand-new product you want to ship today, with minimum signup friction?**
   → **Dodo Payments.** Verify Sri Lanka payout on signup; if confirmed, ship.

4. **Default case (most products):**
   → **Polar.** Lowest fees, best DX, Sri Lanka explicitly supported, clean integration with Next.js + Cloudflare.

---

## 8. Implementation Notes for the Coding Agent

When the agent is wiring up payments, follow these patterns regardless of provider:

### Where each piece lives in the Next.js + Cloudflare stack
- **Webhook handler** → Route Handler at `app/api/webhooks/<provider>/route.ts`. Verify signature using the provider's secret. **Always.**
- **Checkout initiation** → Server Action that creates a checkout session via the provider's API and redirects to the hosted URL.
- **Customer portal link** → Server Action that generates a portal URL and redirects.
- **Subscription state** → Stored in **D1** in a `subscriptions` table, keyed by provider's customer/subscription ID. Updated only via webhook handlers (never trust the client).
- **Entitlements / feature flags** → Cached in **KV** with the user's ID as the key. Webhook handlers invalidate the cache on change.
- **Provider secrets** → `wrangler secret put` (production) and `.dev.vars` (local). Never in `wrangler.jsonc` or env files.

### Webhook reliability pattern
1. Receive webhook, verify signature
2. Return `200` immediately if signature is valid
3. Push event to a **Cloudflare Queue** for processing
4. Queue consumer applies the change to D1 and invalidates KV cache
5. This decouples webhook latency from your DB and prevents the provider from retrying due to slow processing

### Idempotency
Every webhook handler must be idempotent. Use the provider's event ID as a deduplication key (store in D1 with a unique index). Replay-safe by design.

### What NOT to do
- Do not store raw card data anywhere — let the MoR's hosted checkout handle PCI scope entirely.
- Do not trust `successUrl` redirect parameters as source of truth — only webhooks update DB state.
- Do not call the payments API from the browser; route everything through Server Actions or Route Handlers.
- Do not hardcode prices in the app — fetch from the provider's product/price IDs so changes don't require redeploys.
- Do not skip the customer portal — every provider gives you a hosted one for free; build the link, don't reimplement subscription management UI.

---

## 9. What to Verify Before Final Commit

Before the agent generates the payment integration code for a specific product:

1. Confirm **Sri Lanka payout** is currently supported on the chosen provider's signup flow (not just the marketing page).
2. Note the provider's **payout schedule** (Polar: manual or monthly; Paddle: weekly/monthly; Dodo: monthly typical) and minimum payout threshold.
3. Confirm **Wise / Payoneer / direct bank** is set up to receive USD payouts before going live.
4. Decide on **payout currency**: USD is universal and cheapest; converting later via Wise typically beats letting the MoR convert to LKR.
5. For subscription products, confirm the provider supports your **billing model** (per-seat, usage-based, tiered, free trials, proration on plan change).
6. Pull the latest **fee schedule** from the provider's pricing page — providers do change pricing.

---

## 10. Summary Recommendation for New SaaS Products

> **Default: start with Polar.** It's the right call for ~80% of cases for your situation: Sri Lanka explicitly supported, lowest fees, cleanest API, ideal for the Next.js + Cloudflare stack, and good fit for the indie/SaaS founder.
>
> **Switch to Paddle** if and only if you have a real reason: a B2B product with high ACVs where the 5% is irrelevant, or an enterprise customer that demands a more recognized billing partner.
>
> **Switch to Dodo Payments** if and only if onboarding speed is the binding constraint — e.g., you're prototyping and want to ship paywall-tomorrow, or Polar's KYC review is taking longer than you can afford.
>
> **Avoid Lemon Squeezy** for new builds while the Stripe acquisition transition is still ongoing.
>
> **Forget Stripe direct** until Sri Lanka is added to their supported countries list.

---

*Last updated: April 2026. Pricing, country availability, and platform direction in the MoR space change frequently — verify on the provider's site before final integration.*
