---
name: android-in-app-purchase
description: Use when implementing, reviewing, testing, or debugging Android in-app purchases or subscriptions with native Google Play Billing Library (BillingClient, ProductDetails, base plans & offers, upgrades/downgrades with replacement modes, acknowledgement/consumption, pending purchases, Play Developer API, Real-time Developer Notifications, Play Console setup, Play payments policy). For apps using the RevenueCat SDK, use android-revenuecat-iap.
---

# Android In-App Purchase (Google Play Billing) Skill

Guide for building, auditing, and testing in-app purchases on Android with the **Google Play Billing Library (PBL)**. Covers one-time products (consumable / non-consumable), subscriptions with **base plans and offers**, multiple concurrent subscriptions (add-ons), plan changes with **replacement modes**, acknowledgement and consumption, pending purchases, the **Google Play Developer API** and **Real-time Developer Notifications (RTDN)**, and the post-purchase UI. Default to the **latest PBL major version** Google Play currently requires (PBL 7/8 at the time of writing) with Kotlin coroutines (`billing-ktx`).

Sibling skill: **ios-in-app-purchase** has the same structure for StoreKit. Its §13 (post-purchase UI copy, alerts, accessibility) applies to Android too; this skill notes where Play behaves differently.

PBL removes and renames APIs between major versions (e.g. PBL 8 removed `queryPurchaseHistoryAsync` and changed one-time product offers). Verify method names against the release notes for the version in `build.gradle` before shipping.

## 0. Architecture (mental model)

```
                 +-----------------------------+
                 |        Google Play          |
                 +-----------------------------+
   purchase token ^          | RTDN (Cloud Pub/Sub)
                  |          v
+---------------------------+   +--------------------------------+
| Play Billing Library      |   | Backend                        |
| - queryProductDetails     |   | - Play Developer API:          |
| - launchBillingFlow       |   |   subscriptionsv2.get,         |
| - PurchasesUpdatedListener|   |   products.get, acknowledge    |
| - queryPurchasesAsync     |   | - Ledger / entitlements        |
| - acknowledge / consume   |   | - Voided Purchases API         |
+---------------------------+   +--------------------------------+
            ^
            | coroutines / StateFlow
+-------------------------------------------------------------+
| BillingRepository (app-scoped singleton) — created at start |
| - one BillingClient, auto-reconnect                         |
| - processes purchases: verify → grant → ack/consume         |
| - re-queries purchases on start / resume                    |
| - exposes StateFlow<Entitlements>                           |
+-------------------------------------------------------------+
            ^
+-------------------------------------------------------------+
| Compose/Views: Paywall, Plan status, Change plan, Manage    |
+-------------------------------------------------------------+
```

Play's product model differs from Apple's:

| Concept | Play | Notes |
|---|---|---|
| Subscription | One **product** (e.g. `pro`) | Usually one product per tier |
| Billing period / renewal type | **Base plan** (`monthly`, `yearly`, prepaid, installment) | Several per product |
| Trial / intro / discounts | **Offer** on a base plan (`free-trial`, `intro-50`) | Eligibility rules + **offer tags** |
| Subscription group | **Doesn't exist** | Play lets a user hold several subscriptions at once — switching tiers must use the upgrade/downgrade flow |
| Receipt | **Purchase token** per purchase | Verified server-side with the Play Developer API |
| Finish transaction | **Acknowledge** (non-consumables, subs) or **consume** (consumables) | Within **3 days** or Play refunds automatically |

## 1. First, establish context

Before writing or reviewing code, find out (from the codebase or by asking):

- PBL version (`com.android.billingclient:billing-ktx`) and whether it meets Play's current minimum required version; `minSdk`
- Product catalog, classified by type:
  - **Consumable** (coins, credits, boosts)
  - **Non-consumable** one-time (lifetime unlock, remove ads)
  - **Subscription** — products, **base plans** (auto-renewing / prepaid / installment), **offers** and their eligibility (new customer, upgrade, developer-determined via tags)
  - **Add-on subscriptions** held alongside a base subscription
- Is there a backend? (purchase verification, RTDN, consumable balances, cross-platform entitlements). Without one you can't reliably know subscription state (section 6.6)
- Third-party SDK (RevenueCat, Adapty, Qonversion)? Switch to the matching skill
- Account system? (needed for `obfuscatedAccountId` and cross-device/cross-platform access)
- Distribution: Play only, or also other stores (Amazon, Huawei, Galaxy) or sideloading? Play Billing only works for Play-installed apps

Write an **entitlement map** before coding: product ID → base plan → type → feature(s) unlocked. Map IDs explicitly (`Map`/`enum`) — never infer a tier from substrings like `productId.contains("pro")`.

```
pro            / monthly, yearly        → subscription → Pro features
plus           / monthly, yearly        → subscription → Plus features
addon_backup   / monthly                → subscription add-on → backup (requires plus+)
lifetime                                → non-consumable → Pro features
credits_100                             → consumable → +100 credits
```

## 2. Product types — rules and checks

### 2.1 Consumable
- [ ] Credited **exactly once** per `purchaseToken` (idempotent ledger, ideally server-side)
- [ ] Balance **persisted** (server or local DB) — never only in memory, never reset when entitlements are recomputed
- [ ] `consumePurchase()` only **after** the balance is persisted. Unconsumed consumables can't be bought again (`ITEM_ALREADY_OWNED`) and are refunded after 3 days if not acknowledged
- [ ] Consumption acknowledges the purchase — don't also call `acknowledgePurchase`
- [ ] Multi-quantity purchases (`purchase.quantity`) credited `amount × quantity` if quantity is enabled in Play Console
- [ ] Voided purchases (refund/chargeback) → decide policy: deduct balance, flag account (Voided Purchases API / `ONE_TIME_PRODUCT_CANCELED`)

### 2.2 Non-consumable one-time product
- [ ] Access from `queryPurchasesAsync(INAPP)` with `PURCHASED` state (survives reinstall/new device on the same Google account)
- [ ] `acknowledgePurchase()` after granting (client or server)
- [ ] Revoked/refunded → access removed (server: Voided Purchases API; client: purchase disappears from `queryPurchasesAsync`)
- [ ] If lifetime and a subscription unlock the same feature: entitlement = lifetime OR active subscription

### 2.3 Subscriptions
- [ ] Entitlement only for purchases in `PURCHASED` state; `PENDING` → no access yet
- [ ] **Acknowledge every new subscription purchase**, including upgrades/downgrades (each creates a new purchase token)
- [ ] Know what the client can and can't see (section 6.6): `queryPurchasesAsync(SUBS)` returns active subscriptions (including canceled-but-not-expired and grace period); it does **not** return subscriptions on **account hold** or **paused**, and gives **no expiry date**
- [ ] Full state (`SUBSCRIPTION_STATE_ACTIVE / CANCELED / IN_GRACE_PERIOD / ON_HOLD / PAUSED / EXPIRED / PENDING`), expiry, base plan and offer come from `purchases.subscriptionsv2.get` on the backend
- [ ] Grace period → keep access + "fix payment" prompt. Account hold → no access + fix-payment prompt. Paused → no access + "resumes on <date>"
- [ ] Prepaid base plans: no auto-renew; user tops up; access until expiry. Enable `enablePrepaidPlans()` for pending prepaid purchases
- [ ] Installment base plans (where available): commitment period — show it in the paywall and status UI

## 3. Base plans, offers & plan changes

### 3.1 Configuration (Play Console)
- [ ] One subscription product per tier; base plans per period; **same base plan IDs across tiers** (`monthly`, `yearly`) keeps code simple
- [ ] Offers with eligibility: *New customer acquisition* (never had this subscription / any subscription), *Upgrade*, or *Developer determined* (your server/app decides; use **offer tags**)
- [ ] Backwards-compatible base plan/offer set if older app versions still use deprecated APIs
- [ ] Grace period and account hold durations set intentionally; resubscribe enabled if wanted; pause enabled only if your UI supports it

### 3.2 Changing plans (upgrade / downgrade / switch)

Play does **not** replace one subscription with another automatically. Buying a second tier with a plain `launchBillingFlow` gives the user **two subscriptions, billed twice**. Always pass `SubscriptionUpdateParams` with the **old purchase token** and a replacement mode.

| Change | Recommended `ReplacementMode` | Effect |
|---|---|---|
| **Upgrade** (higher tier) | `CHARGE_PRORATED_PRICE` (keeps billing date) or `WITH_TIME_PRORATION` | Immediate |
| **Downgrade** (lower tier) | `DEFERRED` | Old plan until renewal, then new plan |
| **Same tier, longer period** (monthly → yearly) | `CHARGE_FULL_PRICE` (immediate, new cycle) or `DEFERRED` | Per mode |
| **Same tier, shorter period** | `DEFERRED` | At renewal |
| Switch, no extra charge now | `WITHOUT_PRORATION` | Immediate, new price at next renewal |

Checks:
- [ ] Current plan = active subscription purchase for a tier product; pass its `purchaseToken` as `oldPurchaseToken`
- [ ] Each change creates a **new purchase token**; acknowledge it. Backend uses `linkedPurchaseToken` (subscriptionsv2) to retire the old token's entitlement — don't double-grant
- [ ] Deferred change shown in UI: "You'll move to Plus on <date>" (backend: subscriptionsv2 line item / deferred replacement info)
- [ ] Never offer a "Subscribe" button for a second tier while one is active — label it Upgrade / Downgrade / Switch (ios-in-app-purchase §13.2)
- [ ] Changing base plans within the **same product** also uses `SubscriptionUpdateParams`

## 4. Add-on subscriptions (multiple active subscriptions)

- [ ] Each add-on is its **own subscription product**; buy it with a normal billing flow (no update params)
- [ ] Entitlements = **union** of all active subscription purchases
- [ ] Dependency rule in code: add-on features only while the base tier is active; decide/document what happens when the base lapses (the add-on keeps billing!) — warn and link to Play subscription management
- [ ] Only offer add-ons to users with an active base plan
- [ ] Server: a user maps to **multiple purchase tokens**; store entitlements per product, not one "subscription" row per user
- [ ] If your PBL version supports buying several subscription products in one flow (multi-line/add-on purchases), treat each line item separately — verify availability in the current docs

## 5. Offers & pricing

| Offer type | Configured in | Eligibility | How it's applied |
|---|---|---|---|
| **Free trial / intro price** | Play Console offer on a base plan | Play decides (new customer / upgrade rules) | Play only returns offers the user is eligible for in `subscriptionOfferDetails`; pass its `offerToken` |
| **Developer-determined** | Play Console offer + **offer tags** | Your logic (e.g. lapsed users, retention) | Filter `subscriptionOfferDetails` by tag, pass that `offerToken` |
| **Promo codes** | Play Console | Anyone with a code | Redeemed in Play Store or purchase flow; arrives via `PurchasesUpdatedListener` / `queryPurchasesAsync` |
| **Win-back / retention** | Play Console offers (developer-determined) or Play-surfaced | Lapsed / canceling users | Same as developer-determined |
| **One-time product offers** (PBL 8+) | Play Console, multiple purchase options & offers for one-time products | Per offer | Pass the one-time offer token — verify API |

Checks:
- [ ] Choose the offer explicitly: e.g. the eligible offer with a free trial, else the **base plan offer** (`offerId == null`). Never take `subscriptionOfferDetails[0]` blindly
- [ ] Paywall shows trial length and price after the trial from `pricingPhases` (`formattedPrice`, `billingPeriod` ISO 8601, `billingCycleCount`, `recurrenceMode`)
- [ ] All prices from `ProductDetails` — never hardcoded; currency from the user's Play country
- [ ] Price changes: Play notifies users (opt-in for increases above thresholds, opt-out otherwise); handle `SUBSCRIPTION_PRICE_CHANGE_UPDATED` server-side and show nothing custom unless needed
- [ ] Payment issues: call `showInAppMessages(activity, InAppMessageParams(TRANSACTIONAL))` on app start so Play can show "fix your payment method" for grace/hold

## 6. Core Play Billing implementation checklist

### 6.1 Connection
- [ ] **One** `BillingClient` per app (app-scoped singleton / DI), created at app start, not per screen
- [ ] `enablePendingPurchases(PendingPurchasesParams.newBuilder().enableOneTimeProducts()[.enablePrepaidPlans()].build())`
- [ ] Reconnect on disconnect (PBL 8: `enableAutoServiceReconnection()`; older: retry with backoff in `onBillingServiceDisconnected`)
- [ ] Handle `BILLING_UNAVAILABLE` (Play Store missing/old, unsupported country, not a Play install) with a clear message

### 6.2 Product details
- [ ] `queryProductDetails` with `ProductType.SUBS` and `ProductType.INAPP` lists (separate params per type), IDs from the entitlement map
- [ ] Graceful handling of missing/unfetched products (wrong ID, not active, app not uploaded to a track, tester not opted in) — log them, show the rest
- [ ] Cache `ProductDetails` in memory for the session; refetch on paywall open if stale

### 6.3 Launching the purchase
- [ ] `launchBillingFlow(activity, params)` on the main thread with a live `Activity`
- [ ] `ProductDetailsParams` with the chosen `offerToken` for subscriptions
- [ ] `setObfuscatedAccountId(hash(userId))` (≤ 64 chars, no PII) when you have accounts — your server uses it to match purchases to users and Play uses it for fraud detection
- [ ] Plan changes use `SubscriptionUpdateParams` (section 3.2)
- [ ] Purchase button disabled while a flow is in progress

### 6.4 Processing purchases (`PurchasesUpdatedListener` + re-query)
- [ ] Handle response codes: `OK`, `USER_CANCELED` (no alert), `ITEM_ALREADY_OWNED` (re-query + process — usually an unacknowledged/unconsumed purchase), `SERVICE_*` / `NETWORK_ERROR` (retryable), `DEVELOPER_ERROR` (bug: wrong offer token/params — log)
- [ ] `PENDING` → show "Waiting for payment / approval", grant nothing; complete when it becomes `PURCHASED` (listener or re-query)
- [ ] `PURCHASED` → **verify** (server, or at minimum local signature check) → grant / persist → **acknowledge or consume** → refresh entitlements
- [ ] Idempotent processing by `purchaseToken` (the same purchase arrives via listener and re-query)
- [ ] **Re-query purchases** (`queryPurchasesAsync` SUBS + INAPP) on app start **and** `onResume` — catches purchases made outside the app (promo codes, Play Store, another device), pending purchases that completed, refunds, and anything the listener missed (process death)
- [ ] Unacknowledged purchases found on re-query are processed (granted + acknowledged) — this is the safety net for the 3-day rule

### 6.5 Restore & management
- [ ] Purchases follow the **Google account** automatically; `queryPurchasesAsync` on start replaces a "restore" call
- [ ] A **Restore** button is still useful when you have accounts: re-query and re-send purchase tokens to the backend to re-link
- [ ] Manage subscription → deep link `https://play.google.com/store/account/subscriptions?sku=<productId>&package=<packageName>` (or without `sku` for the list)
- [ ] Cancellation happens in Play; the app reflects it after re-query / RTDN

### 6.6 What the client can't know
`queryPurchasesAsync` returns `Purchase` objects with `products`, `purchaseToken`, `orderId`, `purchaseState`, `purchaseTime`, `isAcknowledged`, `isAutoRenewing`, `quantity`, `accountIdentifiers` — but **no expiry date, no base plan ID, no offer ID, no grace/hold state**. For "Renews on …", "Ends on …", current base plan, deferred downgrades, and hold/pause UI, call your backend (subscriptionsv2). Client-only apps can show: active vs not, `isAutoRenewing` (cancelled or not), and the base plan the user picked at purchase time on this device.

## 7. Server-side checklist (if backend exists)

- [ ] Verify every purchase token with the **Google Play Developer API** (service account with *View financial data* / *Manage orders* permissions in Play Console): `purchases.subscriptionsv2.get`, `purchases.products.get` (or the newer one-time products v2 endpoint if your version uses it)
- [ ] Check `packageName`, product ID, `obfuscatedExternalAccountId` matches the user, state, `acknowledgementState`
- [ ] Acknowledge server-side (`purchases.subscriptions.acknowledge` / `purchases.products.acknowledge`) or let the client do it — but make sure **someone** does within 3 days
- [ ] **RTDN** configured: Play Console → Monetization setup → Cloud Pub/Sub topic; push subscription to your endpoint (authenticate the push, e.g. OIDC token); respond 2xx quickly
- [ ] RTDN payloads only carry the token and type — always **re-fetch** with subscriptionsv2 and overwrite your state (ordering-safe)
- [ ] Idempotent by `messageId` / event; handle duplicates and out-of-order delivery
- [ ] Upgrades/downgrades: on a new token with `linkedPurchaseToken`, retire the old token's entitlement
- [ ] Poll the **Voided Purchases API** (or handle `voidedPurchaseNotification`) for refunds/chargebacks
- [ ] Data model: user → many purchase tokens; subscriptions per product; consumable ledger keyed by token
- [ ] Test purchases (`testPurchase` / license testers) kept separate from production analytics

### Notification handling (`subscriptionNotification.notificationType`)

| Type | Action (after re-fetching subscriptionsv2) |
|---|---|
| `SUBSCRIPTION_PURCHASED` (4) | Grant; acknowledge if not yet; link to user via `obfuscatedExternalAccountId` |
| `SUBSCRIPTION_RENEWED` (2) | Extend expiry |
| `SUBSCRIPTION_RECOVERED` (1) | Restore access (left hold) |
| `SUBSCRIPTION_IN_GRACE_PERIOD` (6) | Keep access; prompt to fix payment |
| `SUBSCRIPTION_ON_HOLD` (5) | Remove access; prompt to fix payment |
| `SUBSCRIPTION_CANCELED` (3) | Auto-renew off; keep access until expiry; retention messaging |
| `SUBSCRIPTION_RESTARTED` (7) | Auto-renew back on |
| `SUBSCRIPTION_DEFERRED` (9) | Expiry moved (deferral) |
| `SUBSCRIPTION_PAUSED` (10) / `PAUSE_SCHEDULE_CHANGED` (11) | Remove access while paused / show resume date |
| `SUBSCRIPTION_REVOKED` (12) | Remove access immediately (refund) |
| `SUBSCRIPTION_EXPIRED` (13) | Remove access |
| `SUBSCRIPTION_PRICE_CHANGE_UPDATED` (19) | Track price change state |
| `SUBSCRIPTION_PENDING_PURCHASE_CANCELED` (20) | Drop pending record |
| `oneTimeProductNotification` (`PURCHASED` 1 / `CANCELED` 2) | Credit / void one-time purchase |
| `voidedPurchaseNotification` | Revoke access / deduct balance |
| `testNotification` | Return 2xx |

Check the RTDN reference for types added after this skill was written.

## 8. Play Console configuration checks

- [ ] Payments profile / merchant account set up; app has the `com.android.vending.BILLING` permission (added by PBL automatically)
- [ ] App uploaded to at least one track (internal testing is fine) — products can't be queried otherwise
- [ ] Products **active**; base plans **active**; offers active with correct eligibility and tags
- [ ] Prices set for all countries you distribute in (or auto-converted intentionally)
- [ ] Grace period, account hold, pause, resubscribe settings chosen deliberately
- [ ] RTDN topic configured; service account linked with the needed permissions
- [ ] License testers added; testers opted in to the testing track
- [ ] Data safety form declares purchase history / financial info collection; account deletion flow if the app has accounts

## 9. Google Play policy compliance

- [ ] Digital goods/features sold **via Google Play Billing** (Payments policy), unless an alternative billing / user choice billing / external offers program applies in that country and you're enrolled. Programs differ by region (EEA, India, Korea, Indonesia, Japan, US changes after the 2025 rulings) — **check the current policy and enrollment requirements** before adding any non-Play payment or link-out
- [ ] Subscription paywall clearly shows: price, billing period, what's included, free trial terms and the price after it, that it auto-renews, how to cancel (Subscriptions policy)
- [ ] Link to manage/cancel in Play; Terms of Service and Privacy Policy links
- [ ] No deceptive UI (hidden close, fake urgency, misleading trial wording, pre-selected expensive plan)
- [ ] Physical goods/services use other payment methods, not Play Billing
- [ ] Meets Play's minimum PBL version requirement for new apps and updates

## 10. Testing plan

1. **License testers** (Play Console → Settings → License testing) on a device signed into a tester Google account, app installed from (or matching) a testing track. Test payment methods: *Test card, always approves*, *always declines*, *slow test card, approves after a few minutes* (pending), *slow, declines*
2. **Accelerated renewals** for testers (e.g. monthly and weekly renew every ~5 minutes, yearly ~30 minutes; a test subscription renews up to 6 times). Check the current table in the Play docs
3. **Play Billing Lab** app — change the test country (prices, availability), reset free-trial/intro eligibility, test price changes and in-app messages
4. **Internal testing track** end to end with RTDN on your staging backend
5. Unit tests with a fake `BillingClient` wrapper (Appendix A.6)

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| 1 | Buy consumable | Balance +N once; persisted; consumed |
| 2 | Consumable — app killed before consume | Re-query on next start credits once and consumes |
| 3 | Buy non-consumable | Unlocked; acknowledged; survives reinstall |
| 4 | Subscribe with free trial (eligible) | Trial offer shown; access; acknowledged |
| 5 | Same account, trial already used | Trial offer not returned; base plan price shown |
| 6 | Slow test card (pending) | "Waiting for payment"; unlock when it becomes PURCHASED |
| 7 | Pending purchase declined | Pending UI cleared; no access |
| 8 | Not acknowledged for 3 days (test: ~5 min for testers) | Play refunds — must never happen; verify re-query acknowledges |
| 9 | Renewal / expiry (accelerated) | Access extends / removed |
| 10 | Grace period / account hold | Access per policy; fix-payment prompt / in-app message |
| 11 | Cancel in Play | Access until expiry; UI shows "Ends on …" (backend) |
| 12 | Resubscribe / restart | Auto-renew back on |
| 13 | **Upgrade** Plus → Pro | One subscription; immediate; new token acknowledged; old token retired |
| 14 | **Downgrade** Pro → Plus (DEFERRED) | Still Pro; pending change shown; Plus at renewal |
| 15 | Monthly → yearly | Per chosen replacement mode |
| 16 | Buy add-on with base active | Both active |
| 17 | Base expires, add-on active | Add-on features locked per policy; user warned |
| 18 | Buy second tier **without** update params (bug check) | Must not be possible from the UI |
| 19 | Promo code redeemed in Play Store | Granted on next start/resume via re-query |
| 20 | Refund / revoke from Play Console | Access/balance removed (RTDN / voided) |
| 21 | Pause / resume (if enabled) | Access removed while paused; restored on resume |
| 22 | Different Play country (Billing Lab) | Localized prices / availability |
| 23 | Offline start | Cached entitlements used; re-verified later |
| 24 | Second device, same Google account | Purchases present; acknowledged once |
| 25 | Different app account, same Google account | Backend refuses/handles per `obfuscatedAccountId` policy |
| 26 | Buy consumable, then entitlement refresh | Balance unchanged |

Post-purchase UI scenarios: run ios-in-app-purchase §13.6 (U1–U16) with Android equivalents (Play Billing Lab / Play Console for expire, refund, hold).

## 11. Common bugs to look for in review

- Purchase never **acknowledged** (or only acknowledged in the listener) → Play auto-refunds after 3 days
- Acknowledging/consuming **before** the grant is persisted
- Granting access for `PENDING` purchases
- `BillingClient` created per Activity/screen; not reconnected after disconnect; `endConnection` never called / called while in use
- Purchases not re-queried on start/resume → missed out-of-app purchases, completed pending purchases, process-death losses
- Tier change via a plain `launchBillingFlow` (no `SubscriptionUpdateParams`) → user billed for two subscriptions
- Old purchase token still granting access after an upgrade (`linkedPurchaseToken` ignored)
- `subscriptionOfferDetails[0]` used as "the price" (may be a trial or developer offer); `offerToken` missing → `DEVELOPER_ERROR`
- Assuming the client knows expiry, base plan, or hold/pause state
- Tier inferred from product ID substrings
- Consumable balance kept in the entitlement object and reset on refresh, or credited without `purchaseToken` idempotency
- `ITEM_ALREADY_OWNED` shown as an error instead of re-querying and processing
- Error dialog on `USER_CANCELED`
- `launchBillingFlow` from a background thread or with a destroyed Activity
- Purchase verification only on the client with the public key hardcoded, or not at all, when a backend exists
- RTDN handler trusting the payload without re-fetching; not idempotent; unauthenticated push endpoint
- Hardcoded prices; trial text shown without an eligible trial offer
- No manage-subscription link; missing auto-renew/cancel disclosure, Terms/Privacy links
- Using removed APIs (`querySkuDetailsAsync`, `SkuDetails`, `queryPurchaseHistoryAsync` on PBL 8) or a PBL version below Play's minimum

## 12. Output format

When asked to **review/audit** IAP code:

1. **Summary** — PBL version, entitlement map (product → base plan → type → feature), backend yes/no, RTDN yes/no
2. **Checklist results** — grouped by sections 2–9 and 13 with ✅ Pass / ❌ Fail / ⚠️ Needs check, each with file:line reference (Play Console items listed for the user to confirm)
3. **Critical issues first** (unacknowledged purchases/refunds, double billing on tier change, lost purchases, double crediting, wrong tier, policy risks) with fix code
4. **Testing gaps** — scenario numbers from section 10 not covered

When asked to **implement**, start from the Appendix, adapt `ProductCatalog` to the user's entitlement map, then list the remaining Play Console, server, and policy steps. Remove parts the app doesn't use (add-ons, consumables, prepaid plans) rather than leaving dead code.

## 13. Post-purchase UI on Android

Follow ios-in-app-purchase §13 for copy, alerts and accessibility. Android-specific mapping:

| State | Source | UI |
|---|---|---|
| Subscribed, renewing | Client: `isAutoRenewing == true`; backend: `SUBSCRIPTION_STATE_ACTIVE` + expiry | "Pro · Monthly — Renews on 12 Dec" (date needs backend) |
| Cancelled, still active | Client: `isAutoRenewing == false`; backend: `CANCELED` | "Ends on 12 Dec" + "Resubscribe" (Play deep link) |
| Grace period | Backend `IN_GRACE_PERIOD`; Play in-app message | ⚠️ fix payment, access kept |
| Account hold | Backend `ON_HOLD` (client: purchase missing) | 🔴 fix payment, access removed |
| Paused | Backend `PAUSED` | "Paused — resumes on …" |
| Downgrade scheduled | Backend deferred replacement | ℹ️ "You'll move to Plus on …" |
| Pending payment | Client `PENDING` | "Waiting for payment" |
| Add-on active, base lapsed | Client/backend | ⚠️ "Backup needs an active plan. You're still being charged." |

Rules:
- [ ] Current plan marked **Current plan**, buy button disabled; others labelled Upgrade / Downgrade / Switch
- [ ] Downgrade confirmation alert listing lost features (Play's sheet doesn't say it)
- [ ] After a `DEFERRED` downgrade, say "Change scheduled", not "Purchase successful"
- [ ] Features unlock immediately after purchase (collect `StateFlow` in UI)
- [ ] Works with large font scale, dark theme, RTL, small screens, TalkBack (state conveyed in text)

## 14. Newer Play Billing features & platform rules

Verify availability against the current PBL release notes.
- **PBL 7**: installment subscriptions, pending prepaid plans (`enablePrepaidPlans`), `PendingPurchasesParams` required
- **PBL 8**: `queryPurchaseHistoryAsync` removed (use backend history), automatic service reconnection, one-time products with multiple purchase options and offers, `QueryProductDetailsResult` with unfetched products
- **In-app messaging**: `showInAppMessages` for payment-declined recovery
- **Alternative billing / user choice billing / external offers**: region-specific programs with their own APIs (`enableUserChoiceBilling`, `enableAlternativeBillingOnly`, external offer APIs) and reporting requirements — only after enrolling; follow the current policy
- **Play Billing Lab**: country switching, trial eligibility reset, accelerated testing
- **Play Integrity API**: optional extra signal against tampered clients for high-value consumables

---

## Appendix — Reference implementation (PBL 7/8 + billing-ktx, Kotlin coroutines, Jetpack Compose)

Adapt IDs and policies. Verify method names against your PBL version.

### A.1 Catalog — explicit entitlement map

```kotlin
enum class Tier { FREE, PLUS, PRO }

enum class Period { MONTHLY, YEARLY }

object ProductCatalog {
    // One subscription product per tier; base plans share IDs across tiers.
    val tierBySubscriptionId = mapOf(
        "plus" to Tier.PLUS,
        "pro" to Tier.PRO,
    )
    val periodByBasePlanId = mapOf(
        "monthly" to Period.MONTHLY,
        "yearly" to Period.YEARLY,
    )
    // Add-ons — each its own subscription product
    val addonIds = setOf("addon_backup")
    // One-time
    const val LIFETIME_ID = "lifetime"
    val creditsByProductId = mapOf(
        "credits_100" to 100,
        "credits_500" to 500,
    )

    val subscriptionIds: Set<String> get() = tierBySubscriptionId.keys + addonIds
    val inAppIds: Set<String> get() = creditsByProductId.keys + LIFETIME_ID

    /** Offer tag used for developer-determined offers you never auto-select. */
    const val TAG_RETENTION = "retention"
}
```

### A.2 Ledger & backend interfaces

```kotlin
interface ConsumableLedger {
    suspend fun balance(): Int
    /** Idempotent by purchaseToken. Returns new balance. */
    suspend fun credit(purchaseToken: String, amount: Int): Int
}

/** Your backend. Verifies with the Play Developer API and links the token to the user. */
interface PurchaseVerifier {
    /** Returns true when the purchase is valid and recorded. Must be idempotent by token. */
    suspend fun verify(purchase: Purchase): Boolean
}
```

### A.3 BillingRepository

```kotlin
import android.app.Activity
import android.content.Context
import com.android.billingclient.api.*
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import java.security.MessageDigest
import kotlin.coroutines.resume

data class Entitlements(
    val tier: Tier = Tier.FREE,
    val hasLifetime: Boolean = false,
    val activeAddonIds: Set<String> = emptySet(),
    /** Active subscription purchases by product ID (needed as oldPurchaseToken for plan changes). */
    val subscriptionPurchases: Map<String, Purchase> = emptyMap(),
) {
    val hasBasePlan get() = tier > Tier.FREE
    /** Policy: add-ons only usable while a base plan is active. */
    val usableAddonIds get() = if (hasBasePlan) activeAddonIds else emptySet()
}

sealed interface PurchaseOutcome {
    data class Activated(val productId: String) : PurchaseOutcome
    data class Scheduled(val productId: String) : PurchaseOutcome   // DEFERRED downgrade
    data object Pending : PurchaseOutcome
}

data class BillingState(
    val subscriptions: List<ProductDetails> = emptyList(),
    val inApps: List<ProductDetails> = emptyList(),
    val entitlements: Entitlements = Entitlements(),
    val creditBalance: Int = 0,          // separate from entitlements — never reset by refresh
    val purchaseInFlight: Boolean = false,
    val pendingProductIds: Set<String> = emptySet(),
    val lastOutcome: PurchaseOutcome? = null,
    val errorMessage: String? = null,
)

/** App-scoped singleton (create in Application or via DI). */
class BillingRepository(
    context: Context,
    private val scope: CoroutineScope,          // application scope
    private val verifier: PurchaseVerifier,
    private val ledger: ConsumableLedger,
) : PurchasesUpdatedListener {

    private val client: BillingClient = BillingClient.newBuilder(context.applicationContext)
        .setListener(this)
        .enablePendingPurchases(
            PendingPurchasesParams.newBuilder()
                .enableOneTimeProducts()
                .enablePrepaidPlans()
                .build()
        )
        .enableAutoServiceReconnection()        // PBL 8+; remove on older versions and reconnect manually
        .build()

    private val _state = MutableStateFlow(BillingState())
    val state: StateFlow<BillingState> = _state.asStateFlow()

    /** Set when the user signs in. Hashed — never send raw IDs/PII. */
    var obfuscatedAccountId: String? = null
        private set

    private val processing = mutableSetOf<String>()      // purchase tokens in flight
    private var pendingScheduledProductId: String? = null

    fun setUser(userId: String?) {
        obfuscatedAccountId = userId?.let { sha256(it) }  // 64 hex chars (Play limit: 64)
    }

    // MARK: Connection

    private suspend fun ensureConnected(): Boolean {
        if (client.isReady) return true
        return suspendCancellableCoroutine { cont ->
            client.startConnection(object : BillingClientStateListener {
                override fun onBillingSetupFinished(result: BillingResult) {
                    if (cont.isActive) cont.resume(result.responseCode == BillingClient.BillingResponseCode.OK)
                }
                override fun onBillingServiceDisconnected() {
                    if (cont.isActive) cont.resume(false)
                }
            })
        }
    }

    /** Call from Application start and from the main Activity's onResume. */
    fun refresh() = scope.launch {
        if (!ensureConnected()) {
            _state.update { it.copy(errorMessage = "Google Play billing isn't available right now.") }
            return@launch
        }
        if (_state.value.subscriptions.isEmpty()) loadProducts()
        _state.update { it.copy(creditBalance = ledger.balance()) }
        reconcilePurchases()
    }

    // MARK: Products

    private suspend fun loadProducts() {
        suspend fun query(type: String, ids: Set<String>): List<ProductDetails> {
            val params = QueryProductDetailsParams.newBuilder()
                .setProductList(ids.map {
                    QueryProductDetailsParams.Product.newBuilder().setProductId(it).setProductType(type).build()
                })
                .build()
            val result = client.queryProductDetails(params)
            val list = result.productDetailsList.orEmpty()
            val missing = ids - list.map { it.productId }.toSet()
            if (missing.isNotEmpty()) android.util.Log.w("IAP", "Missing products: $missing")
            return list
        }
        val subs = query(BillingClient.ProductType.SUBS, ProductCatalog.subscriptionIds)
        val inApps = query(BillingClient.ProductType.INAPP, ProductCatalog.inAppIds)
        _state.update { it.copy(subscriptions = subs, inApps = inApps) }
    }

    /** Best offer for a base plan: eligible trial/intro without a reserved tag, else the base offer. */
    fun offerFor(details: ProductDetails, basePlanId: String): ProductDetails.SubscriptionOfferDetails? {
        val offers = details.subscriptionOfferDetails.orEmpty().filter { it.basePlanId == basePlanId }
        return offers.firstOrNull { it.offerId != null && ProductCatalog.TAG_RETENTION !in it.offerTags }
            ?: offers.firstOrNull { it.offerId == null }
    }

    // MARK: Purchase

    fun canPurchase(details: ProductDetails): Boolean =
        if (details.productId in ProductCatalog.addonIds) _state.value.entitlements.hasBasePlan else true

    /**
     * Launches the Play purchase sheet. For a tier change, pass the current tier's purchase
     * and a replacement mode (section 3.2) — never buy a second tier without it.
     */
    fun launchPurchase(
        activity: Activity,
        details: ProductDetails,
        offer: ProductDetails.SubscriptionOfferDetails? = null,
        replace: Purchase? = null,
        replacementMode: Int = BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.CHARGE_PRORATED_PRICE,
    ) {
        if (_state.value.purchaseInFlight || !canPurchase(details)) return

        val productParams = BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(details)
            .apply { offer?.let { setOfferToken(it.offerToken) } }
            .build()
        val builder = BillingFlowParams.newBuilder().setProductDetailsParamsList(listOf(productParams))
        obfuscatedAccountId?.let { builder.setObfuscatedAccountId(it) }
        if (replace != null) {
            builder.setSubscriptionUpdateParams(
                BillingFlowParams.SubscriptionUpdateParams.newBuilder()
                    .setOldPurchaseToken(replace.purchaseToken)
                    .setSubscriptionReplacementMode(replacementMode)
                    .build()
            )
        }
        pendingScheduledProductId =
            if (replacementMode == BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.DEFERRED && replace != null)
                details.productId else null

        _state.update { it.copy(purchaseInFlight = true, errorMessage = null) }
        val result = client.launchBillingFlow(activity, builder.build())
        if (result.responseCode != BillingClient.BillingResponseCode.OK) {
            _state.update { it.copy(purchaseInFlight = false, errorMessage = message(result)) }
        }
    }

    override fun onPurchasesUpdated(result: BillingResult, purchases: MutableList<Purchase>?) {
        scope.launch {
            when (result.responseCode) {
                BillingClient.BillingResponseCode.OK -> purchases.orEmpty().forEach { process(it, fromFlow = true) }
                BillingClient.BillingResponseCode.USER_CANCELED -> Unit          // no alert
                BillingClient.BillingResponseCode.ITEM_ALREADY_OWNED -> reconcilePurchases()
                else -> _state.update { it.copy(errorMessage = message(result)) }
            }
            _state.update { it.copy(purchaseInFlight = false) }
        }
    }

    // MARK: Processing — verify → grant → acknowledge/consume → refresh

    private suspend fun process(purchase: Purchase, fromFlow: Boolean = false) {
        val token = purchase.purchaseToken
        if (!processing.add(token)) return
        try {
            when (purchase.purchaseState) {
                Purchase.PurchaseState.PENDING -> {
                    _state.update {
                        it.copy(pendingProductIds = it.pendingProductIds + purchase.products,
                                lastOutcome = if (fromFlow) PurchaseOutcome.Pending else it.lastOutcome)
                    }
                    return
                }
                Purchase.PurchaseState.PURCHASED -> Unit
                else -> return
            }

            // Someone else's purchase on this Google account? Don't grant it to the current user.
            val owner = purchase.accountIdentifiers?.obfuscatedAccountId
            if (owner != null && obfuscatedAccountId != null && owner != obfuscatedAccountId) return

            if (!verifier.verify(purchase)) {
                android.util.Log.w("IAP", "Verification failed for ${purchase.orderId}")
                return                                   // not acknowledged → retried on next reconcile
            }

            val productId = purchase.products.first()
            val credits = ProductCatalog.creditsByProductId[productId]
            if (credits != null) {
                val balance = ledger.credit(token, credits * purchase.quantity)   // persist FIRST
                val consumed = client.consumePurchase(ConsumeParams.newBuilder().setPurchaseToken(token).build())
                if (consumed.billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    _state.update { it.copy(creditBalance = balance) }
                }
            } else if (!purchase.isAcknowledged) {
                client.acknowledgePurchase(AcknowledgePurchaseParams.newBuilder().setPurchaseToken(token).build())
            }

            _state.update { it.copy(pendingProductIds = it.pendingProductIds - purchase.products.toSet()) }
            if (fromFlow) {
                val scheduled = pendingScheduledProductId
                _state.update {
                    it.copy(lastOutcome = if (scheduled == productId) PurchaseOutcome.Scheduled(productId)
                                          else PurchaseOutcome.Activated(productId))
                }
                pendingScheduledProductId = null
            }
        } finally {
            processing.remove(token)
        }
        refreshEntitlements()
    }

    /** Safety net: processes anything unacknowledged/unconsumed and rebuilds entitlements. */
    private suspend fun reconcilePurchases() {
        for (type in listOf(BillingClient.ProductType.SUBS, BillingClient.ProductType.INAPP)) {
            val result = client.queryPurchasesAsync(QueryPurchasesParams.newBuilder().setProductType(type).build())
            result.purchasesList.forEach { purchase ->
                val needsWork = purchase.purchaseState == Purchase.PurchaseState.PURCHASED &&
                    (!purchase.isAcknowledged || purchase.products.any { it in ProductCatalog.creditsByProductId })
                if (needsWork || purchase.purchaseState == Purchase.PurchaseState.PENDING) process(purchase)
            }
        }
        refreshEntitlements()
    }

    // MARK: Entitlements (client view — see section 6.6 for its limits)

    private suspend fun refreshEntitlements() {
        val subs = client.queryPurchasesAsync(
            QueryPurchasesParams.newBuilder().setProductType(BillingClient.ProductType.SUBS).build()
        ).purchasesList.filter { it.purchaseState == Purchase.PurchaseState.PURCHASED }
        val inApps = client.queryPurchasesAsync(
            QueryPurchasesParams.newBuilder().setProductType(BillingClient.ProductType.INAPP).build()
        ).purchasesList.filter { it.purchaseState == Purchase.PurchaseState.PURCHASED }

        var tier = Tier.FREE
        val addons = mutableSetOf<String>()
        val byProduct = mutableMapOf<String, Purchase>()
        for (p in subs) for (id in p.products) {
            byProduct[id] = p
            ProductCatalog.tierBySubscriptionId[id]?.let { if (it > tier) tier = it }
            if (id in ProductCatalog.addonIds) addons += id
        }
        val hasLifetime = inApps.any { ProductCatalog.LIFETIME_ID in it.products }
        if (hasLifetime && tier < Tier.PRO) tier = Tier.PRO                    // policy: lifetime == Pro

        _state.update {
            it.copy(entitlements = Entitlements(tier, hasLifetime, addons, byProduct))
        }
    }

    fun clearOutcome() = _state.update { it.copy(lastOutcome = null, errorMessage = null) }

    fun showPaymentIssueMessages(activity: Activity) {
        val params = InAppMessageParams.newBuilder()
            .addInAppMessageCategoryToShow(InAppMessageParams.InAppMessageCategoryId.TRANSACTIONAL)
            .build()
        client.showInAppMessages(activity, params) { result ->
            if (result.responseCode == InAppMessageResult.InAppMessageResponseCode.SUBSCRIPTION_STATUS_UPDATED) refresh()
        }
    }

    private fun message(result: BillingResult): String = when (result.responseCode) {
        BillingClient.BillingResponseCode.NETWORK_ERROR,
        BillingClient.BillingResponseCode.SERVICE_UNAVAILABLE,
        BillingClient.BillingResponseCode.SERVICE_DISCONNECTED -> "Couldn't reach Google Play. Please try again."
        BillingClient.BillingResponseCode.BILLING_UNAVAILABLE -> "Purchases aren't available on this device or account."
        BillingClient.BillingResponseCode.ITEM_UNAVAILABLE -> "This item isn't available right now."
        else -> "Purchase failed. Please try again."
    }

    private fun sha256(value: String): String =
        MessageDigest.getInstance("SHA-256").digest(value.toByteArray()).joinToString("") { "%02x".format(it) }
}
```

### A.4 Plan-change helper

```kotlin
enum class PlanChange { NEW, CURRENT, UPGRADE, DOWNGRADE, SWITCH_LONGER, SWITCH_SHORTER }

object PlanChanges {
    /** Compares the target (product + base plan) with the user's current tier and base plan. */
    fun classify(currentProductId: String?, currentBasePlanId: String?,
                 targetProductId: String, targetBasePlanId: String): PlanChange {
        val current = currentProductId?.let { ProductCatalog.tierBySubscriptionId[it] } ?: return PlanChange.NEW
        val target = ProductCatalog.tierBySubscriptionId[targetProductId] ?: return PlanChange.NEW
        if (currentProductId == targetProductId && currentBasePlanId == targetBasePlanId) return PlanChange.CURRENT
        if (target > current) return PlanChange.UPGRADE
        if (target < current) return PlanChange.DOWNGRADE
        val cur = ProductCatalog.periodByBasePlanId[currentBasePlanId]
        val tgt = ProductCatalog.periodByBasePlanId[targetBasePlanId]
        return if (tgt == Period.YEARLY && cur == Period.MONTHLY) PlanChange.SWITCH_LONGER else PlanChange.SWITCH_SHORTER
    }

    fun replacementMode(change: PlanChange): Int = when (change) {
        PlanChange.UPGRADE -> BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.CHARGE_PRORATED_PRICE
        PlanChange.SWITCH_LONGER -> BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.CHARGE_FULL_PRICE
        PlanChange.DOWNGRADE, PlanChange.SWITCH_SHORTER -> BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.DEFERRED
        PlanChange.NEW, PlanChange.CURRENT -> BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.WITH_TIME_PRORATION
    }
}
```

`currentBasePlanId` comes from your backend (subscriptionsv2 `lineItems[].offerDetails.basePlanId`); without a backend, store the base plan the user picked on this device and treat it as a hint.

### A.5 Compose paywall (policy-ready)

```kotlin
@Composable
fun PaywallScreen(repo: BillingRepository, activity: Activity) {
    val state by repo.state.collectAsStateWithLifecycle()
    val uriHandler = LocalUriHandler.current
    val current = state.entitlements.subscriptionPurchases.keys
        .firstOrNull { it in ProductCatalog.tierBySubscriptionId }

    Column(Modifier.verticalScroll(rememberScrollState()).padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        state.lastOutcome?.let { OutcomeBanner(it, onDismiss = repo::clearOutcome) }
        if (state.pendingProductIds.isNotEmpty()) Text("Waiting for payment…", style = MaterialTheme.typography.bodySmall)

        state.subscriptions.filter { it.productId in ProductCatalog.tierBySubscriptionId }.forEach { product ->
            ProductCatalog.periodByBasePlanId.keys.forEach { basePlanId ->
                val offer = repo.offerFor(product, basePlanId) ?: return@forEach
                val change = PlanChanges.classify(current, /* from backend */ null, product.productId, basePlanId)
                PlanRow(
                    product = product,
                    offer = offer,
                    label = when (change) {
                        PlanChange.CURRENT -> "Current plan"
                        PlanChange.UPGRADE -> "Upgrade"
                        PlanChange.DOWNGRADE -> "Downgrade"
                        PlanChange.SWITCH_LONGER, PlanChange.SWITCH_SHORTER -> "Switch"
                        PlanChange.NEW -> "Subscribe"
                    },
                    enabled = change != PlanChange.CURRENT && !state.purchaseInFlight,
                    onClick = {
                        // Show the downgrade confirmation (ios-in-app-purchase §13.3) before DOWNGRADE.
                        repo.launchPurchase(
                            activity, product, offer,
                            replace = current?.let { state.entitlements.subscriptionPurchases[it] },
                            replacementMode = PlanChanges.replacementMode(change),
                        )
                    },
                )
            }
        }

        if (state.entitlements.hasBasePlan) {
            state.subscriptions.filter { it.productId in ProductCatalog.addonIds }.forEach { addon ->
                repo.offerFor(addon, "monthly")?.let { offer ->
                    PlanRow(addon, offer, "Add", enabled = !state.purchaseInFlight) {
                        repo.launchPurchase(activity, addon, offer)
                    }
                }
            }
        }

        // Required disclosures (Subscriptions policy)
        Text(
            "Subscriptions renew automatically until cancelled. Cancel anytime in Google Play → Payments & subscriptions.",
            style = MaterialTheme.typography.bodySmall,
        )
        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            TextButton(onClick = { uriHandler.openUri("https://example.com/terms") }) { Text("Terms") }
            TextButton(onClick = { uriHandler.openUri("https://example.com/privacy") }) { Text("Privacy") }
            TextButton(onClick = {
                uriHandler.openUri("https://play.google.com/store/account/subscriptions?package=${activity.packageName}")
            }) { Text("Manage") }
            TextButton(onClick = { repo.refresh() }) { Text("Restore") }
        }
    }
}

@Composable
fun PlanRow(product: ProductDetails, offer: ProductDetails.SubscriptionOfferDetails,
            label: String, enabled: Boolean, onClick: () -> Unit) {
    val phases = offer.pricingPhases.pricingPhaseList
    val recurring = phases.last()                        // the price after any trial/intro phases
    val intro = phases.dropLast(1).firstOrNull()
    Row(Modifier.fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
        Column(Modifier.weight(1f)) {
            Text(product.name, style = MaterialTheme.typography.titleMedium)
            Text("${recurring.formattedPrice} / ${recurring.billingPeriod}")   // format ISO 8601 (P1M, P1Y) for display
            intro?.let {
                val what = if (it.priceAmountMicros == 0L) "Free" else it.formattedPrice
                Text("$what for ${it.billingPeriod} × ${it.billingCycleCount}, then ${recurring.formattedPrice}",
                     style = MaterialTheme.typography.bodySmall)
            }
        }
        Button(onClick = onClick, enabled = enabled) { Text(label) }
    }
}
```

`OutcomeBanner` shows "You're on Pro 🎉" for `Activated`, "Change scheduled — Plus starts at your next renewal" for `Scheduled`, and "Waiting for payment" for `Pending`.

### A.6 Wiring & tests

```kotlin
class App : Application() {
    val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
    lateinit var billing: BillingRepository

    override fun onCreate() {
        super.onCreate()
        billing = BillingRepository(this, appScope, BackendVerifier(), RoomConsumableLedger(this))
        billing.refresh()
    }
}

class MainActivity : ComponentActivity() {
    private val billing get() = (application as App).billing

    override fun onResume() {
        super.onResume()
        billing.refresh()                       // out-of-app purchases, completed pending, refunds
        billing.showPaymentIssueMessages(this)  // grace / hold recovery
    }
}
```

For unit tests, put `BillingClient` behind a small interface (`queryProductDetails`, `queryPurchases`, `acknowledge`, `consume`, `launch`) and fake it. Test: purchase processed once when it arrives via listener **and** reconcile; credits persisted before consume; `PENDING` grants nothing; verification failure leaves the purchase unacknowledged; tier change always passes `SubscriptionUpdateParams`.

### A.7 Backend verification & RTDN (Node/TypeScript)

```ts
import express from "express";
import { google } from "googleapis";

const auth = new google.auth.GoogleAuth({ scopes: ["https://www.googleapis.com/auth/androidpublisher"] });
const publisher = google.androidpublisher({ version: "v3", auth });
const PACKAGE = process.env.ANDROID_PACKAGE!;

// Called by the app (PurchaseVerifier) after a purchase.
export async function verifySubscription(userHash: string, token: string) {
  const { data } = await publisher.purchases.subscriptionsv2.get({ packageName: PACKAGE, token });
  if (data.externalAccountIdentifiers?.obfuscatedExternalAccountId !== userHash) throw new Error("account mismatch");
  if (data.linkedPurchaseToken) await db.subscriptions.retire(data.linkedPurchaseToken);   // upgrade/downgrade
  await db.subscriptions.upsert(token, userHash, data);   // state, lineItems (productId, expiryTime, basePlan, offer)
  if (data.acknowledgementState === "ACKNOWLEDGEMENT_STATE_PENDING") {
    const productId = data.lineItems![0].productId!;
    await publisher.purchases.subscriptions.acknowledge({ packageName: PACKAGE, subscriptionId: productId, token });
  }
  return data.subscriptionState;
}

// Cloud Pub/Sub push endpoint for RTDN. Authenticate the push (OIDC token) in production.
const app = express();
app.post("/rtdn", express.json(), async (req, res) => {
  const message = req.body.message;
  if (await db.processedMessages.has(message.messageId)) return res.sendStatus(204);
  const payload = JSON.parse(Buffer.from(message.data, "base64").toString());

  if (payload.subscriptionNotification) {
    const token = payload.subscriptionNotification.purchaseToken;
    const { data } = await publisher.purchases.subscriptionsv2.get({ packageName: PACKAGE, token });
    if (data.linkedPurchaseToken) await db.subscriptions.retire(data.linkedPurchaseToken);
    await db.subscriptions.upsertFromNotification(token, data);   // re-fetch and overwrite — ordering-safe
  } else if (payload.voidedPurchaseNotification) {
    await db.purchases.void(payload.voidedPurchaseNotification.purchaseToken);
  } else if (payload.oneTimeProductNotification) {
    await db.oneTime.refresh(payload.oneTimeProductNotification.purchaseToken);
  }

  await db.processedMessages.add(message.messageId);
  res.sendStatus(204);
});
```
