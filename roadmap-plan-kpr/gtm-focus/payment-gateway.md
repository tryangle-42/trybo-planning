# Payment Gateway Integration — Baseline Definition

> Owner: KPR
> Scope: Payment Gateway — 10 subcomponents (PAY.1–PAY.10)
> Module mapping: Payment Gateway Integration
> Current completion: 0% — 0 of 106 capabilities exist, 0 partial, 106 missing
> Source: GTM_SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

The revenue engine. No payment fails silently, no card data touches the platform unprotected, no subscription lapses without intelligent recovery. This is the bridge between "product that works" and "product that makes money."

10 subcomponents covering the full money lifecycle: transaction processing → provider abstraction → subscription billing → usage metering → invoicing → fraud prevention → dispute management → analytics → customer self-service → payouts. Entirely greenfield.

**Key architectural decision:** Leverage Stripe (primary) and Razorpay (India) as processors. Do NOT build card processing — use hosted fields / Stripe Elements so card data never touches our servers. This keeps us at PCI SAQ A (simplest level) instead of SAQ D (full audit).

## Current State

| What Exists | What's Missing |
|---|---|
| (nothing — entirely greenfield) | Payment processing (auth, capture, refund) |
| | Subscription management + dunning |
| | Usage metering + rating |
| | Invoicing + tax calculation |
| | Fraud detection (Stripe Radar) |
| | Chargeback management |
| | Revenue analytics (MRR/ARR/churn) |
| | Customer self-service portal |

## Baseline Definition

**The line is:** The platform can accept payments via Stripe, manage subscriptions with dunning, meter usage, generate compliant invoices with taxes, detect fraud via Radar, track chargebacks, compute revenue metrics, and let customers self-serve their billing. Anything beyond this is refinement.

---

## Baseline Components

### PAY.1 Payment Processing Engine — ~9 dev days

**What this is:** The core transaction layer — the actual movement of money. Authorization, capture, void, refund, idempotency, 3DS/SCA, and webhook-driven status tracking.

#### Baseline deliverables

**1. Authorization + Capture + Sale**
Stripe `PaymentIntent.create()` for immediate sale. Separate auth/capture for delayed settlement. Void for pre-settlement cancellation. Why baseline: this is literally accepting money.

**2. Refund (full and partial)**
`Refund.create(payment_intent, amount)`. Track via webhooks. Link to original for reconciliation. Why baseline: manual Stripe dashboard refunds are unscalable.

**3. Idempotency**
`Idempotency-Key` header on every payment request. `payment_transactions` table with unique constraint. Why baseline: timeout + retry without idempotency = double charge. #1 cause of disputes.

**4. 3DS / SCA orchestration**
Stripe's built-in 3DS with `request_three_d_secure: "automatic"`. Frontend handles redirect/challenge via Stripe.js. Why baseline: EU law (PSD2/SCA) requires it. Without 3DS, EU cards get declined.

**5. Webhook-driven status updates**
Handler for `payment_intent.succeeded`, `payment_failed`, `charge.refunded`, `charge.dispute.created`. Update local state. Emit to Event Bus. Why baseline: payment is async. Without webhooks, you don't know what happened.

| Refinement (deferred) | Why |
|---|---|
| Pre-auth / incremental auth | Standard auth/capture covers baseline |
| Multi-currency | Start with USD/INR |
| Smart routing / cascading | Single provider at baseline |
| Retry on soft declines | Dunning (PAY.3) handles this |

**Dependencies:** Stripe account (infra), Config Registry (1.6), Event Bus (1.2), Data Store (1.8)

**Certifications:** PCI DSS v4.0 (Req 3, 4, 6), PSD2/SCA, EMV 3DS2

---

### PAY.2 Multi-Provider Abstraction — 0 dev days (DEFERRED)

**What this is:** Unified API routing transactions to the optimal provider. Failover, cost-based routing, geographic routing.

**Does not have its own baseline at launch.** Reasons:
1. **Stripe alone covers launch.** Including India via Stripe India.
2. **Premature abstraction is expensive.** Need two concrete examples to abstract from.
3. **Risk is low.** Stripe uptime >99.99%.

**Recommendation:** Design PAY.1 with a clean `PaymentProvider` interface. When Razorpay or Adyen is added, introduce the abstraction layer then. ~8 dev days when triggered.

---

### PAY.3 Subscription & Recurring Billing — ~10 dev days

**What this is:** The recurring revenue mechanism. Plan management, subscription lifecycle, proration, trials, and dunning. Dunning alone recovers 15-30% of failed payments — the highest-ROI feature in the payment stack.

#### Baseline deliverables

**1. Plan management**
Stripe Products + Prices API. Plans: `{name, price, interval, trial_days, features}`. Local metadata for feature gating. Why baseline: without plans, nothing to subscribe to.

**2. Subscription lifecycle**
Stripe Subscriptions: create, upgrade, downgrade, pause, cancel. Map events to local state. Emit via Event Bus. Why baseline: this is recurring revenue.

**3. Proration**
Stripe calculates prorated credit/charge on mid-cycle changes. Surface preview before confirming. Why baseline: without it, customers are overcharged (disputes) or undercharged (revenue leak).

**4. Trial management**
14-day free → Day 14: charge → success: active → fail: dunning. Track conversion rate. Why baseline: trials are the primary acquisition mechanism.

**5. Dunning management**
Day 0: email → Day 1: retry → Day 3: retry + email → Day 7: retry + banner → Day 14: degraded → Day 30: cancelled. Stripe Smart Retries + custom escalation. Why baseline: at $285K MRR, recovers $42K-$85K/month.

| Refinement (deferred) | Why |
|---|---|
| Account Updater | Stripe handles for most issuers |
| Backup payment methods | Single method at baseline |
| Coupon/discount engine | Manual Stripe coupons |
| Grandfathering | No plan changes at launch |
| Revenue recognition hooks | Manual rev rec at early stage |
| Cancellation retention flows | "Are you sure?" sufficient |

**Dependencies:** PAY.1 (payment processing), Notification Service (Platform 1.10), Config Registry (Platform 1.6)

**Certifications:** PCI DSS v4.0, PSD2/SCA (recurring exemption), RBI e-Mandate, ASC 606 / IFRS 15

---

### PAY.4 Usage Metering & Rating Engine — ~9.5 dev days

**What this is:** Bills for consumption — voice minutes, WhatsApp messages, task executions, AI compute. Ingest, deduplicate, aggregate, and rate usage against plan allowances. Accuracy to the cent — billing disputes erode trust faster than any feature gap.

#### Baseline deliverables

**1. Event ingestion with deduplication**
Usage events: `{customer_id, meter, quantity, idempotency_key, timestamp}`. Queued and deduped before storage. Why baseline: double-counting = overcharging.

**2. Platform-specific meters**
Voice minutes (in/out, domestic/intl), WhatsApp messages (template/session), task executions, AI compute (tokens), phone provisioning (monthly), storage (GB). Each has its own aggregation. Why baseline: generic metering doesn't know 12.5 minutes rounds to 13.

**3. Rating engine**
Cycle end: compare usage vs allowance per meter. Overage = (usage - allowance) × rate. Output: rated line items for invoicing. Why baseline: this is where usage becomes money.

**4. Spend alerts**
At 80%, 90%, 100% of allowance: email + in-app. At 100%: "Overage active at $X/unit." Why baseline: bill shock destroys trust. $500 bill when expecting $99 = immediate churn.

| Refinement (deferred) | Why |
|---|---|
| Real-time usage dashboard | Portal (PAY.9) shows current usage |
| Usage caps / hard limits | Overages are more revenue; caps for enterprise |
| Billing period auto-aggregation | Manual at cycle end initially |

**Dependencies:** Event Bus (Platform 1.2), Job Queue (Platform 1.9), PAY.3 (plan allowances), Data Store (Platform 1.8)

**Certifications:** SOC 2 (CC5.2), ASC 606 / IFRS 15

---

### PAY.5 Invoicing & Tax Engine — ~6 dev days

**What this is:** Every billing cycle must produce a compliant invoice — auto-generated, tax-calculated, and delivered. Stripe Tax handles the complexity of US sales tax, EU VAT, and India GST.

#### Baseline deliverables

**1. Automated invoice generation**
Cycle end: subscription + usage + credits → subtotal → tax → invoice. PDF + table record. Sequential numbering with no gaps. Why baseline: customers need invoices for accounting.

**2. Tax calculation via Stripe Tax**
Auto-calculates correct rate by location, product type, and regulation. Handles US nexus, EU VAT reverse-charge, India GST. Why baseline: tax errors = government liability or customer disputes.

**3. Tax ID validation**
EU VAT (VIES), India GST. Valid B2B = reverse-charge (VAT=0). Why baseline: charging VAT to a valid B2B customer is incorrect.

**4. Invoice delivery**
Email with PDF + portal download + webhook. Why baseline: undelivered invoice isn't an invoice.

| Refinement (deferred) | Why |
|---|---|
| Credit notes | Manual via Stripe dashboard |
| Multi-currency invoicing | Start with USD/INR |
| Invoice customization | Stripe default template sufficient |
| E-invoicing compliance | Region-specific; build when entering |

**Dependencies:** PAY.3 (subscription charges), PAY.4 (usage items), Stripe Tax (infra), Notification Service (Platform 1.10)

**Certifications:** EU VAT Directive, India GST Act, ASC 606 / IFRS 15, SOC 2 (CC5.2)

---

### PAY.6 Fraud Detection & Risk Management — ~1 dev day

**What this is:** Pre-transaction (velocity, blocklists), at-transaction (AVS, CVV, 3DS), post-transaction (monitoring). Stripe Radar provides ML-based scoring out of the box.

#### Baseline deliverables

**1. Enable Stripe Radar**
ML-based risk scoring on every transaction. Blocks high-risk automatically. Why baseline: essentially free fraud prevention.

**2. Basic custom rules**
Block: >3 fails from same card in 1hr (velocity), billing ≠ IP country (geo-mismatch), disposable email domains. Why baseline: catches obvious patterns ML might miss on new accounts.

**3. 3DS for high-risk**
Risk score > 65 → 3DS challenge. Below → frictionless. Configuration on top of PAY.1, not new infra.

| Refinement (deferred) | Why |
|---|---|
| Custom ML scoring | Stripe Radar is better than anything we'd build |
| Device fingerprinting | Radar includes basic signals |
| Manual review queue | Review in Stripe dashboard at low volume |
| Platform behavioral signals | Needs data accumulation first |

**Dependencies:** PAY.1 (payment flow), Stripe Radar (infra)

**Certifications:** PCI DSS v4.0 (Req 6, 10), PSD2/SCA, EMV 3DS2

---

### PAY.7 Chargeback & Dispute Management — ~4 dev days

**What this is:** Chargebacks cost 2-3x the transaction amount. Visa monitors at 0.9% dispute ratio — exceeding it means fines or account termination. Ingest notifications, compile evidence, track deadlines, monitor ratio.

#### Baseline deliverables

**1. Dispute ingestion via webhooks**
`charge.dispute.created` → create record with deadline, reason, amount. `charge.dispute.closed` → update outcome. Why baseline: without automation, you discover chargebacks from buried Stripe emails.

**2. Evidence compilation framework**
Auto-gather: transaction record, subscription status, usage records. The platform's unique data (call recordings, chat transcripts, task logs) makes evidence unusually compelling. Operator submits via Stripe dashboard at baseline. Why baseline: the evidence exists — the framework assembles it.

**3. Dispute ratio monitoring**
Disputes / transactions over rolling 30 days. Alert at 0.5% (warning), 0.7% (urgent), 0.9% (critical — Visa VDMP). Why baseline: crossing 0.9% can be business-ending.

| Refinement (deferred) | Why |
|---|---|
| Automated representment | Manual via Stripe dashboard at low volume |
| Pre-dispute alerts (Verifi/Ethoca) | Cost vs value doesn't justify at low volume |
| Refund-to-prevent | Manual decision acceptable |
| Reason code analytics | Review individually at low volume |

**Dependencies:** PAY.1 (webhooks), Data Store (Platform 1.8), Event Bus (Platform 1.2)

**Certifications:** PCI DSS v4.0 (Req 10), Visa VDMP, Mastercard ECP

---

### PAY.8 Payment Analytics & Reporting — ~4 dev days

**What this is:** Revenue metrics (MRR, ARR, churn, net retention) are the language of SaaS health. Transaction reporting shows volume, approval rates, and decline reasons.

#### Baseline deliverables

**1. Revenue metrics computation**
MRR, ARR, churn rate, expansion, contraction, net revenue retention. Time-series for trending. Why baseline: these are the numbers investors and executives look at daily.

**2. Transaction reporting**
Daily/weekly/monthly volume, approval rate, top decline reasons, refund volume. Filterable by date, plan. Why baseline: "why did revenue drop yesterday?" requires this.

| Refinement (deferred) | Why |
|---|---|
| Provider performance comparison | Single provider at launch |
| Cohort analysis | Product strategy, not operations |
| Fraud metrics | Stripe Radar dashboard |
| Settlement reconciliation | Stripe payouts are reliable |
| Financial reporting (ASC 606) | Spreadsheets initially |
| Custom reports / data export | Fixed reports cover baseline |

**Dependencies:** PAY.1 (transaction data), PAY.3 (subscription data), Data Store (Platform 1.8)

**Certifications:** SOC 2 (CC5.2, CC7.1), ASC 606 / IFRS 15

---

### PAY.9 Customer Payment Portal — ~4.5 dev days

**What this is:** Self-service billing. Customers manage payment methods, view invoices, change subscriptions, see usage — without contacting support. Reduces billing support tickets 40-60%.

#### Baseline deliverables

**1. Stripe Customer Portal integration**
`billing_portal.Session.create()` provides: payment methods, invoices, subscription changes, cancellation. Pre-built, PCI-compliant. Why baseline: building custom payment UI requires PCI scope. Stripe's portal has zero burden.

**2. Usage display**
Custom UI: "Voice: 820 / 1,000 minutes (82%)" with progress bar. Data from PAY.4. Why baseline: Stripe portal doesn't show platform-specific usage.

**3. Billing notifications**
Email for: successful payment, failed payment, upcoming renewal (3 days before), plan change. Why baseline: surprise charge with no notification = dispute.

| Refinement (deferred) | Why |
|---|---|
| Spend controls (caps, budgets) | Enterprise feature; most users don't set caps |
| Multi-user billing access | Single-user until teams/org exists |
| Custom embeddable components | Portal redirect is acceptable UX |

**Dependencies:** PAY.3 (subscriptions), PAY.4 (usage), PAY.5 (invoices), Stripe Customer Portal (infra)

**Certifications:** PCI DSS v4.0 (SAQ A), SOC 2 (CC6.1), GDPR (billing data access)

---

### PAY.10 Payout & Marketplace Payments — 0 dev days (DEFERRED)

**What this is:** Split payments, connected account onboarding (KYC/KYB), payout scheduling, tax reporting (1099) for a marketplace model.

**Does not have its own baseline.** Reasons:
1. **No marketplace at launch.** Direct subscription model only.
2. **Payout infrastructure is expensive.** 10+ dev weeks for a model that doesn't exist.
3. **Reversible.** Stripe Connect adds cleanly later.

**Recommendation:** Defer entirely. Revisit when marketplace validated with 5+ pilot providers. ~15 dev days when triggered.

---

## Build Order

```
Phase 1 — Accept Money (cannot launch without these)
├── PAY.1  Payment Processing (auth, capture, refund, 3DS, webhooks)
├── PAY.3  Subscription Billing (plans, trials, proration, dunning)
└── PAY.6  Fraud Detection (Stripe Radar + basic rules)

Phase 2 — Bill Correctly (Month 1, usage-based billing)
├── PAY.4  Usage Metering (event ingestion, meters, rating, alerts)
└── PAY.5  Invoicing & Tax (auto-generate, Stripe Tax, delivery)

Phase 3 — Manage & Optimize (Month 1-2, self-service + ops)
├── PAY.9  Customer Portal (Stripe portal + usage display + notifications)
├── PAY.7  Chargeback Management (ingestion, evidence, ratio monitoring)
└── PAY.8  Analytics (MRR/ARR/churn, transaction reporting)

Phase 4 — Scale (when triggered by business need)
├── PAY.2  Multi-Provider Abstraction (when Razorpay/Adyen added)
└── PAY.10 Marketplace Payments (when marketplace model validated)
```

## What Counts as Refinement (explicitly deferred)

| Category | Examples | Why deferred |
|---|---|---|
| Multi-provider | Abstraction layer, failover, cost routing | Single provider at launch |
| Advanced billing | Account Updater, backup methods, coupons, grandfathering | Stripe handles basics |
| Advanced fraud | Custom ML, device fingerprint, manual review | Stripe Radar sufficient |
| Dispute automation | Programmatic representment, pre-dispute alerts | Manual at low volume |
| Analytics depth | Cohorts, settlement recon, custom reports | Fixed reports cover baseline |
| Marketplace | Splits, payouts, KYC, 1099 | No marketplace model yet |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Stripe account + Radar | Infrastructure | Payment provider + fraud |
| Stripe Tax | Infrastructure | Tax calculation |
| Config Registry | Platform 1.6 | API keys, plan config |
| Event Bus | Platform 1.2 | Payment status events, usage events |
| Job Queue | Platform 1.9 | Usage ingestion pipeline |
| Data Store | Platform 1.8 | All payment tables |
| Notification Service | Platform 1.10 | Dunning, billing, invoice delivery |

## Effort Estimate

| Module | Effort |
|---|---|
| PAY.1 Payment Processing Engine | 9 days |
| PAY.2 Multi-Provider Abstraction | 0 days (deferred) |
| PAY.3 Subscription & Recurring Billing | 10 days |
| PAY.4 Usage Metering & Rating | 9.5 days |
| PAY.5 Invoicing & Tax Engine | 6 days |
| PAY.6 Fraud Detection & Risk | 1 day |
| PAY.7 Chargeback & Dispute Mgmt | 4 days |
| PAY.8 Payment Analytics & Reporting | 4 days |
| PAY.9 Customer Payment Portal | 4.5 days |
| PAY.10 Payout & Marketplace | 0 days (deferred) |
| **Total** | **~48 dev days** |

## Key Concepts Explained

### What is PCI SAQ A?
PCI DSS has different self-assessment questionnaires based on how you handle card data. SAQ A is the simplest: "card data never touches your servers" — you use hosted payment fields (Stripe Elements) that send details directly to Stripe. Reduces PCI from 300+ controls to ~20 questions.

### What is dunning?
When a subscription payment fails, dunning is the automated recovery: retry at different times/days, escalate with emails, show banners, eventually suspend. Good dunning recovers 15-30% of failed payments.

### What is 3DS / SCA?
3D Secure is the protocol behind "Verified by Visa." Strong Customer Authentication (PSD2) requires two-factor for EU electronic payments. Merchants can request exemptions for low-risk to reduce friction.
