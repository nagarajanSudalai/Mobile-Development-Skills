---
name: ios-in-app-purchase
description: Use when implementing, reviewing, testing, or debugging iOS in-app purchases or subscriptions with native StoreKit (StoreKit 2/StoreKit 1, paywalls, subscription groups, upgrades/downgrades, add-ons, consumables, offers, promoted IAPs, paid-to-freemium migration, App Store Server API / Notifications V2, App Review IAP rules). For apps using the RevenueCat SDK, use ios-revenuecat-iap.
---

# iOS In-App Purchase (IAP) Skill

Guide for building, auditing, and testing in-app purchases on Apple platforms. Covers one-time purchases (consumable / non-consumable), auto-renewable subscriptions, subscription groups (upgrade / downgrade / crossgrade), add-on subscriptions, non-renewing subscriptions, offers, App Store Server Notifications V2, and the post-purchase UI (current plan, plan changes, alerts). Default to **StoreKit 2** (iOS 15+; the reference implementation in the Appendix targets iOS 17+) unless the project must support older OS versions or already uses StoreKit 1. Android sibling: **android-in-app-purchase** (Google Play Billing).

## 0. Architecture (mental model)

```
                 +---------------------------+
                 |     App Store Server      |
                 +---------------------------+
          JWS purchases ^        | Server Notifications V2
                        |        v
+--------------------------+   +-------------------------------+
| StoreKit 2 (on device)   |   | Backend webhook               |
| - Product.products(for:) |   | - Verify JWS (Apple library)  |
| - product.purchase()     |   | - Ledger / entitlements per   |
| - Transaction.updates    |   |   subscription group          |
| - currentEntitlements    |   | - App Store Server API        |
+--------------------------+   +-------------------------------+
            ^
            | async/await
+-------------------------------------------------------------+
| StoreManager (@MainActor, @Observable) — created at launch  |
| - Products grouped: base subs, add-ons, one-time, passes    |
| - Listener on Transaction.updates + Transaction.unfinished  |
| - Grants content → persists → finish() → refresh            |
| - Publishes entitlements (status per group, add-ons, credits)|
+-------------------------------------------------------------+
            ^
+-------------------------------------------------------------+
| SwiftUI: Paywall, Plan status, Change plan, Redeem, Manage  |
+-------------------------------------------------------------+
```

## 1. First, establish context

Before writing or reviewing code, find out (from the codebase or by asking):

- Minimum iOS version → StoreKit 2 (iOS 15+) or StoreKit 1 (`SKPaymentQueue`). Note features gated by OS: `SubscriptionInfo.status(for: groupID)` and `@Observable` (iOS 17+), `groupLevel` (iOS 16.4+), win-back offers (iOS 18+).
- Product catalog, classified by type:
  - **Consumable** (coins, credits, boosts, tip jar)
  - **Non-consumable** (lifetime unlock, remove ads, content pack)
  - **Auto-renewable subscription** — which **subscription group(s)** and **levels**
  - **Add-on subscriptions** (extra subscriptions active alongside a base plan)
  - **Non-renewing subscription** (fixed-duration pass, e.g. "3-month season pass")
- Is there a backend? (server validation, consumable balances, entitlement sync across devices/platforms)
- Third-party SDK (RevenueCat, Adapty, Qonversion)? If so, follow its patterns and only check what it does not handle. For RevenueCat, switch to the **ios-revenuecat-iap** skill (this skill still applies to App Store Connect setup, App Review rules, and post-purchase UI).
- Was the app ever **paid up front**? If moving to freemium, existing buyers must be grandfathered (section 14.4).
- Family Sharing enabled on any product?

Write down an **entitlement map** before coding: product ID → product type → group (if any) → feature(s) unlocked. Every check below is verified against this map. Map product IDs to features **explicitly** (dictionary/enum) — never infer a tier from substrings like `productID.contains("pro")`.

## 2. Product types — rules and checks

### 2.1 Consumable (one-time, repeatable)
- [ ] Credited **exactly once** per `transaction.id` (idempotent ledger, ideally server-side)
- [ ] Balance is **persisted** (server or local store) — never only an in-memory property, and never reset when entitlements are recomputed
- [ ] `finish()` only after the balance is persisted — unfinished consumables are redelivered via `Transaction.updates` / `Transaction.unfinished`
- [ ] Consumables **do NOT appear in `Transaction.currentEntitlements`** and are not restorable — the balance must live in your own storage/server, tied to the user account
- [ ] (iOS 18+) If you need consumable history in `Transaction.all`, set Info.plist `SKIncludeConsumableInAppPurchaseHistory = YES`
- [ ] Handle `CONSUMPTION_REQUEST` server notification (respond with consumption info within 12 hours) if you have a backend
- [ ] Refund (`revocationDate`) of a consumable → decide policy: deduct balance, flag account

### 2.2 Non-consumable (one-time, permanent)
- [ ] Access derived from `Transaction.currentEntitlements` (survives reinstall, new device)
- [ ] Restore Purchases button calls `AppStore.sync()`
- [ ] Revoked transaction (refund / Family Sharing removed) → access removed
- [ ] Family Sharing: `transaction.ownershipType == .familyShared` handled if enabled
- [ ] If a "lifetime" non-consumable and a subscription unlock the same feature, entitlement = lifetime OR active subscription

### 2.3 Auto-renewable subscription
- [ ] Status from `Product.SubscriptionInfo.status(for: groupID)` (or `product.subscription?.status`) — handle every `RenewalState`: `.subscribed`, `.inGracePeriod`, `.inBillingRetryPeriod`, `.expired`, `.revoked`
- [ ] Grant access in `.inGracePeriod`; decide policy for `.inBillingRetryPeriod` (usually no access, show "fix payment" prompt)
- [ ] Billing Grace Period enabled in App Store Connect (recommended)
- [ ] Skip superseded transactions: ignore `transaction.isUpgraded == true`
- [ ] Use `expirationDate` + `revocationDate` from verified transactions — never trust a locally cached flag without re-checking
- [ ] Show renewal state in UI (section 13): next renewal date, "Ends on <date>" when `renewalInfo.willAutoRenew == false`

### 2.4 Non-renewing subscription
- [ ] Apple does **not** track expiry — your app/server computes `expiry = purchaseDate + duration`
- [ ] Repeat purchases **stack** (extend from current expiry, not from purchase date)
- [ ] Never rely on `currentEntitlements` to drop expired passes — compute expiry yourself (e.g. from `Transaction.all` or your server)
- [ ] Must be available across all the user's devices → requires your own account system / server sync (App Review expects this)
- [ ] Remind users before expiry; no auto-renew messaging on the paywall

## 3. Subscription groups — rules and checks

Core rule: **a user can have only ONE active subscription per group.** Put mutually exclusive tiers (Basic / Pro / Premium, monthly / yearly of the same tier) in the **same group**; put things that should be held together in **different groups**.

### 3.1 Configuration (App Store Connect)
- [ ] Levels ordered from highest value (level 1) to lowest — this ordering drives upgrade/downgrade behaviour
- [ ] Same-tier monthly/yearly at the **same level**
- [ ] Group display name and localizations set (shown in Settings → Subscriptions)
- [ ] Intro offer eligibility is **per group**: one intro offer per group per user

### 3.2 Changing plans within a group
| Change | When it takes effect | Billing |
|---|---|---|
| **Upgrade** (to higher level) | Immediately | Prorated refund of the remaining old period |
| **Downgrade** (to lower level) | At next renewal date | Old plan continues until then |
| **Crossgrade**, same duration | Immediately | Prorated refund |
| **Crossgrade**, different duration | At next renewal date | — |

Checks:
- [ ] Purchasing another product in the same group is a normal `purchase()` — StoreKit handles the switch; do NOT block it as "already subscribed"
- [ ] Pending downgrade/crossgrade shown in UI from `status.renewalInfo.autoRenewPreference` (product ID that will renew next) — "You'll move to Basic on <date>"
- [ ] Current tier = the verified, non-upgraded, non-expired transaction's `productID` in that group (not the most recent purchase blindly)
- [ ] Paywall highlights the current plan and labels others as Upgrade / Downgrade / Switch based on `groupLevel` (section 13.2)
- [ ] Server handles `DID_CHANGE_RENEWAL_PREF` (pending change) and `DID_CHANGE_RENEWAL_STATUS` (auto-renew on/off), plus `SUBSCRIBED` with subtype `RESUBSCRIBE`

## 4. Add-on subscriptions (multiple active subscriptions)

Use when a user can hold a **base plan plus extras** (e.g. Pro plan + Extra Storage + Team seats).

- [ ] Each add-on is in its **own subscription group** (if it were in the base plan's group it would replace the base plan)
- [ ] Add-ons are listed **separately** from base plans in the paywall (both are `.autoRenewable`, so filter by your entitlement map, not by `product.type`)
- [ ] Entitlements computed as the **union** across all groups: iterate `Transaction.currentEntitlements` and/or check `status(for:)` for every group ID
- [ ] Dependency rule enforced in code: add-on features only active while the required base plan is active (Apple does not enforce this). Decide and document what happens if the base lapses (add-on keeps billing!) — warn the user and point to Manage Subscriptions
- [ ] Paywall: only offer add-ons to users with an active base plan (or bundle clearly)
- [ ] Manage Subscriptions lets users cancel each subscription independently — UI must reflect partial cancellations
- [ ] Server: a user may map to **multiple `originalTransactionId`s** (one per group); store entitlements per group, not one "subscription" row per user
- [ ] Restore restores all groups; test restoring base + add-on on a new device
- [ ] Each group has its own intro-offer eligibility and its own renewal date — don't assume they're aligned

## 5. Offers & pricing

| Offer type | Configured in | Eligibility | How it's applied |
|---|---|---|---|
| **Introductory** (free trial / pay-as-you-go / pay-up-front) | App Store Connect, 1 per group per user | `await product.subscription?.isEligibleForIntroOffer` | Automatically on purchase if eligible |
| **Promotional** | App Store Connect + **server-generated signature** | Your own logic (existing / lapsed subscribers) | `Product.PurchaseOption.promotionalOffer(...)` in `purchase(options:)` |
| **Offer codes** | App Store Connect (one-time or custom codes) | Anyone with a code | `.offerCodeRedemption(isPresented:)` sheet or App Store; arrives via `Transaction.updates` |
| **Win-back** (iOS 18+) | App Store Connect, lapsed subscribers | StoreKit / `subscription.winBackOffers` | App Store surfaces it, or in-app win-back purchase option; arrives via `Transaction.updates` |

Checks:
- [ ] Intro offer shown only when eligible; paywall shows trial length and the price after the trial
- [ ] Promotional-offer signing key (In-App Purchase key) lives **only on the server**, never in the app
- [ ] Offer-code / win-back redemptions handled by the launch-time `Transaction.updates` listener
- [ ] Offer applied is visible in `transaction.offer` (newer OS) / `offerType` — use for analytics, not entitlements
- [ ] **Price increases**: handle `StoreKit.Message` with reason `.priceIncreaseConsent` (listen to `Message.messages`, defer display while a critical flow is on screen); server handles `PRICE_INCREASE`
- [ ] All prices from `displayPrice` / offer `displayPrice` — never hardcoded

## 6. Core StoreKit 2 implementation checklist

### Product loading
- [ ] `Product.products(for:)` with IDs matching App Store Connect exactly (same IDs in Debug and Release)
- [ ] Graceful handling of missing products (wrong ID, agreements unsigned, product not ready)
- [ ] Group products by your entitlement map: base subscriptions, add-ons, non-consumables, consumables, non-renewing

### Purchasing
- [ ] Handle all results: `.success(verification)`, `.userCancelled`, `.pending` (Ask to Buy / SCA — show "waiting for approval", not an error)
- [ ] Grant only for `.verified`; log `.unverified`
- [ ] `transaction.finish()` only after content/entitlement is saved
- [ ] `appAccountToken` (UUID tied to your user) set when backend links purchases to accounts
- [ ] Purchase button disabled while in flight

### Transaction listener
- [ ] `Transaction.updates` listener started **at app launch** (in the `App` initializer or `@main` app state), retained for app lifetime — catches renewals, Ask-to-Buy approvals, refunds, other-device purchases, offer codes, win-back, plan changes
- [ ] Listener **refreshes entitlements** after each update (not just finishes the transaction)
- [ ] Also process `Transaction.unfinished` on launch
- [ ] Swift 6 concurrency: no synchronous calls into `@MainActor` methods from `Task.detached`

### Entitlement refresh
- [ ] Recompute entitlements on launch, on foreground, after every transaction update, and after the Manage Subscriptions sheet closes
- [ ] SwiftUI: `.subscriptionStatusTask(for: groupID)` / `.currentEntitlementTask(for:)` where suitable
- [ ] Entitlement state is never only a `UserDefaults` bool — caching is fine for offline, but always re-verify

### Restore & management
- [ ] Restore Purchases button → `AppStore.sync()` then refresh
- [ ] Manage subscriptions → `.manageSubscriptionsSheet(isPresented:)`
- [ ] Optional refund request → `beginRefundRequest(for:in:)`

### StoreKit 1 (legacy) extra checks
- [ ] `SKPaymentTransactionObserver` added in `didFinishLaunching`
- [ ] All states handled: purchasing, purchased, failed, restored, deferred
- [ ] `finishTransaction` for purchased/failed/restored
- [ ] `verifyReceipt` is deprecated — plan migration to App Store Server API

## 7. Server-side checklist (if backend exists)

- [ ] Validate with **App Store Server API** (JWS) — not `verifyReceipt`
- [ ] Verify JWS with **Apple's App Store Server Library** (Swift/Java/Python/Node) rather than hand-rolled code. What it does: decode `signedPayload` → take the `x5c` chain from the header → verify the chain up to Apple Root CA (with revocation checks) → verify the signature → decode inner `signedTransactionInfo` / `signedRenewalInfo`
- [ ] **App Store Server Notifications V2** configured for production and sandbox URLs
- [ ] Idempotent by `notificationUUID`; ordering-safe (use `signedDate`, re-fetch with Get All Subscription Statuses when in doubt)
- [ ] Data model: user → many purchases; subscriptions stored **per subscription group** (`originalTransactionId` per group); consumable ledger; non-renewing expiry computed and stacked
- [ ] Correct environment handling (Sandbox vs Production; App Review uses sandbox against your production server)
- [ ] Map `appAccountToken` / `originalTransactionId` to the user account

### Notification handling

| `notificationType` (subtype) | Action |
|---|---|
| `SUBSCRIBED` (`INITIAL_BUY` / `RESUBSCRIBE`) | Grant entitlement for that group; store `originalTransactionId` |
| `DID_RENEW` | Extend expiry |
| `DID_CHANGE_RENEWAL_PREF` (`UPGRADE` / `DOWNGRADE`) | Upgrade: switch tier now. Downgrade/crossgrade: store pending product, switch at renewal |
| `DID_CHANGE_RENEWAL_STATUS` (`AUTO_RENEW_ENABLED` / `_DISABLED`) | Update "will renew" flag; trigger retention messaging if disabled |
| `OFFER_REDEEMED` | Record offer; entitlement via the transaction |
| `DID_FAIL_TO_RENEW` (`GRACE_PERIOD` or none) | Grace period: keep access, prompt to fix payment. No subtype: billing retry — usually remove access |
| `GRACE_PERIOD_EXPIRED` | Remove access |
| `EXPIRED` | Remove access for that group (grace is NOT handled here) |
| `REFUND` / `REVOKE` | Remove access immediately (consumable: deduct per policy) |
| `REFUND_REVERSED` | Restore access / balance |
| `PRICE_INCREASE` | Track consent status |
| `CONSUMPTION_REQUEST` | Send consumption info within 12 hours |
| `ONE_TIME_CHARGE` | Record consumable / non-consumable / non-renewing purchase |

## 8. App Store Connect / configuration checks

- [ ] Paid Applications Agreement, tax and banking complete
- [ ] Each product: reference name, localized name/description, price, review screenshot, review notes
- [ ] Subscription groups and levels set per section 3; add-ons in separate groups per section 4
- [ ] Family Sharing toggled intentionally (cannot be turned off once enabled)
- [ ] In-App Purchase capability enabled in Xcode
- [ ] First release: IAPs attached to the app version being submitted

## 9. App Review compliance (Guideline 3.1)

- [ ] Digital goods/features only via IAP (3.1.1), except where an allowed exception/region rule applies
- [ ] Restore Purchases available
- [ ] Subscription paywall shows: title, length, price per period, what's included, auto-renew statement, trial terms and post-trial price
- [ ] Links to **Terms of Use (EULA)** and **Privacy Policy** in the paywall and App Store metadata
- [ ] Non-renewing subscriptions available on all the user's devices
- [ ] No misleading UI (hidden close button, fake urgency, deceptive plan pre-selection)
- [ ] Physical goods/services use other payment methods, not IAP

## 10. Testing plan

1. **Xcode StoreKit Configuration file** — mirror all products, groups, levels and offers. Use Transaction Manager to refund, approve/decline Ask to Buy, expire, trigger billing retry / grace period, speed up renewals, and send price-increase messages. Automate with `StoreKitTest` (`SKTestSession`).
2. **Sandbox** — sandbox Apple Account on device; accelerated renewals; clear purchase history between runs.
3. **TestFlight** — end-to-end with server notifications.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| 1 | Buy consumable | Balance +N once; persisted; finished |
| 2 | Consumable — app killed before finish() | Redelivered, still credited once |
| 3 | Buy non-consumable | Unlocked; survives reinstall via restore |
| 4 | Subscribe (with intro offer) | Access; trial shown only if eligible |
| 5 | Subscription renews / expires | Access extends / removed |
| 6 | Grace period / billing retry | Access per policy; fix-payment prompt |
| 7 | Cancel auto-renew | Access until expiry; UI shows end date |
| 8 | **Upgrade** within group | Immediate new tier; old transaction ignored (`isUpgraded`) |
| 9 | **Downgrade** within group | Current tier kept; pending change shown; switches at renewal |
| 10 | **Crossgrade** monthly ↔ yearly | Timing per section 3.2 |
| 11 | Buy **add-on** with base active | Both entitlements active |
| 12 | Base plan expires, add-on active | Add-on features locked per policy; user warned |
| 13 | Cancel add-on only | Base unaffected |
| 14 | Restore base + add-on on new device | Both restored |
| 15 | Non-renewing pass bought twice | Expiry stacked; synced to another device |
| 16 | Ask to Buy (pending) | Pending UI; unlock later via updates |
| 17 | Refund / revoke (any type) | Access/balance adjusted |
| 18 | Promotional / offer code / win-back | Correct price; entitlement granted |
| 19 | Price increase consent | Message shown at a safe moment |
| 20 | Family Sharing member | Access; removed on revoke |
| 21 | Different storefront | Localized prices |
| 22 | Offline launch | Cached entitlements used, re-verified later |
| 23 | Buy consumable, then any entitlement refresh | Balance unchanged (not reset) |

Post-purchase UI and plan-change scenarios are in section 13.6.

## 11. Common bugs to look for in review

- Transaction listener only on the paywall → missed renewals, plan changes, Ask-to-Buy
- Listener finishes transactions but never refreshes entitlements → UI stale until relaunch
- `finish()` before granting content, or never calling it
- Granting access on `.unverified`
- Consumable balance kept in the entitlement struct and **reset to 0** when entitlements are rebuilt; or kept only in memory
- Consumables credited without `transaction.id` idempotency → double credit on redelivery
- Tier inferred from product ID substrings (`contains("pro")`)
- Treating "has any subscription transaction" as "is subscribed" (ignores expiry, revocation, `isUpgraded`)
- Base plan and add-on in the **same group** → buying the add-on replaces the base plan
- Add-ons shown in the base-plan list / purchasable without a base plan
- Assuming one subscription per user in the data model (breaks add-ons)
- Blocking in-group upgrades with "you're already subscribed"
- Showing the downgraded tier immediately instead of at renewal
- Downgrade without a confirmation alert, or a "Purchase successful" message after a downgrade that is only scheduled
- After purchase, the paywall still shows "Subscribe" for the plan the user is already on
- Payment problems (grace / billing retry) not surfaced, so users lose access without warning
- Expecting consumables in `currentEntitlements`, or non-renewing subscriptions to expire on their own
- Synchronous call to a `@MainActor` method from `Task.detached` (Swift 6 compile error)
- `@StateObject` wrapping a shared singleton (use `@ObservedObject`, or `@Environment` with `@Observable`)
- Hardcoded prices; intro offer shown to ineligible users
- Promotional-offer signing key shipped inside the app
- `verifyReceipt` on the server; mixing sandbox/production; treating `EXPIRED` as grace period
- No Restore button; missing Terms/Privacy links or auto-renew disclosure
- Promoted IAPs configured in App Store Connect but no `PurchaseIntent.intents` listener → tapping the product on the App Store opens the app and nothing happens
- Paid-to-freemium migration without checking `AppTransaction.originalAppVersion` → original buyers hit the paywall
- Showing a "success" outcome for an `.unverified` purchase result
- Calling `product.purchase()` from a multi-window / visionOS app without a confirmation scene (use `@Environment(\.purchase)` or `purchase(confirmIn:)`)
- External purchase links added without the matching entitlement / storefront check

## 12. Output format

When asked to **review/audit** IAP code:

1. **Summary** — StoreKit version, entitlement map (product → type → group → feature), backend yes/no
2. **Checklist results** — grouped by sections 2–9, 13 and 14 with ✅ Pass / ❌ Fail / ⚠️ Needs check, each with file:line reference
3. **Critical issues first** (lost purchases, double crediting, wrong tier, revenue leaks, rejection risks) with fix code
4. **Testing gaps** — scenario numbers from sections 10 and 13.6 not covered

When asked to **implement**, start from the Appendix reference implementation, adapt the `ProductCatalog` to the user's entitlement map, then list the remaining App Store Connect, server, and compliance steps. Remove parts the app doesn't use (e.g. non-renewing, add-ons) rather than leaving dead code.

## 13. Post-purchase UI, plan changes & alerts

After a purchase, the screen must stop selling and start **explaining the user's current state**. Drive every element from the refreshed `GroupStatus` (Appendix A.3), never from "the last button tapped".

### 13.1 What to show for each state

| State | Header / card | Banner | Actions |
|---|---|---|---|
| Free | Paywall | — | Plans, Restore, Redeem code |
| Subscribed | "**Pro · Monthly** — Renews on 12 Dec for ₹299" | — | Change plan, Manage |
| Free trial / intro | "Free trial ends 12 Dec, then ₹299/month" | — | Change plan, Manage |
| Downgrade / crossgrade scheduled | Current plan card (still Pro) | ℹ️ "You'll move to **Plus** on 12 Dec. You keep Pro until then." | "Keep Pro" → Manage Subscriptions |
| Cancelled (auto-renew off) | "Pro — **Ends on** 12 Dec" (not "Renews") | ℹ️ "Your plan won't renew." | "Resubscribe" → Manage Subscriptions |
| Grace period | Current plan card | ⚠️ "Payment problem. Update your payment method by 15 Dec to keep Pro." | "Update payment" → Manage Subscriptions |
| Billing retry | Plan locked (per policy) | 🔴 "We couldn't renew Pro. Update your payment method to restore access." | "Update payment" |
| Expired | "Your Pro plan ended on 12 Dec" | — | Resubscribe (win-back / promo offer if eligible) |
| Refunded / revoked | Back to free | Neutral message, no blame | Plans |
| Ask to Buy pending | "Waiting for approval" | — | Disable buying the same plan |
| Add-on active, base active | Add-on listed under plan: "Cloud Backup — Renews 20 Dec" | — | Manage |
| Add-on active, **base lapsed** | Add-on shown as paused | ⚠️ "Cloud Backup needs an active plan. You're still being charged for it." | Resubscribe base / Manage (cancel add-on) |
| Lifetime owner | "Lifetime — no renewals" | — | Hide base-plan upsell; warn if they try to subscribe |
| Non-renewing pass | "Pass active until 5 Feb" | Reminder near expiry | "Extend" (stacks) |
| Consumables | Balance always visible: "🪙 600 credits" | — | Buy more |

Rules:
- [ ] The plan the user is on is marked **Current plan**, its buy button is disabled — never "Subscribe" again
- [ ] Dates and prices come from `GroupStatus.expirationDate` and `displayPrice`, formatted for the user's locale
- [ ] "Renews on" vs "Ends on" is decided by `willAutoRenew`
- [ ] Features unlock **immediately** after an upgrade/purchase (no relaunch); views observe the store
- [ ] Settings screen has a "Subscription" row that opens this status view, not the paywall

### 13.2 Plan-change options (same group)

Label every other plan in the group by comparing `product.subscription.groupLevel` with the current plan's level. **Lower `groupLevel` number = higher tier.**

| Compared with current | Button label | Note under the button | When it applies |
|---|---|---|---|
| Same product | "Current plan" (disabled) | Renewal / end date | — |
| Scheduled next plan | "Starts 12 Dec" (disabled) | "Scheduled — you'll keep Pro until then" | Next renewal |
| Higher tier | "**Upgrade**" | "Starts now. You'll get a prorated refund for the unused time on your current plan." | Immediately |
| Lower tier | "**Downgrade**" | "Starts on 12 Dec. You keep Pro until then." | Next renewal |
| Same tier, same period | "Switch" | "Starts now." | Immediately |
| Same tier, different period (monthly ↔ yearly) | "Switch to yearly" (show savings %) | "Starts on 12 Dec." | Next renewal |

- [ ] Never block in-group purchases with "already subscribed" — StoreKit performs the change
- [ ] Cancelling a scheduled downgrade: send the user to Manage Subscriptions to pick their current plan again
- [ ] Show the savings for yearly using `price` values, computed per month, with the locale's currency formatting

### 13.3 Alerts & confirmations

Apple's purchase sheet confirms the price, but **not** what the user loses. Add your own alert before:

| Trigger | Alert |
|---|---|
| **Downgrade** | Title: "Downgrade to Plus?" · Body: "You'll keep Pro until 12 Dec. After that you'll lose: Offline mode, 4K export." · Buttons: "Downgrade" (destructive), "Keep Pro" (cancel) |
| Crossgrade to a different period | "Switch to yearly? Your yearly plan starts on 12 Dec, when your monthly plan ends." |
| Lifetime owner taps a subscription | "You already have Lifetime access to these features. Subscribe anyway?" |
| Add-on when base is inactive | Don't allow — button disabled with "Requires an active plan" |
| Base plan will lapse while add-on renews (cancelled base + active add-on) | "Your Cloud Backup add-on will keep renewing after Pro ends. Cancel it in Manage Subscriptions if you don't need it." |

No extra alert for upgrades or first purchases (the system sheet is enough). Never use an alert for "user cancelled" — just return to the screen.

### 13.4 Feedback after the purchase

Use `PurchaseOutcome` (set by `StoreManager.purchase`, computed from the **refreshed** status):

| Outcome | Feedback |
|---|---|
| `.activated` (new / upgrade / immediate switch) | Success sheet or toast: "You're on **Pro** 🎉" + list of what's unlocked; dismiss the paywall |
| `.scheduled` (downgrade / crossgrade at renewal) | Toast: "Change scheduled. **Plus** starts on 12 Dec." Stay on the status screen. **Not** "Purchase successful" |
| `.pendingApproval` | "Waiting for approval. We'll unlock Pro as soon as it's approved." |
| Error | Short, retryable message; no technical details |

After Manage Subscriptions closes, refresh entitlements and show what changed (e.g. cancel → "Ends on 12 Dec").

### 13.5 Accessibility & layout checks
- [ ] Current plan / Upgrade / Downgrade are conveyed by text, not colour alone
- [ ] Works with Dynamic Type (largest sizes), Dark Mode, small screens (iPhone SE), and RTL languages
- [ ] Long localized plan names and prices don't truncate the button
- [ ] VoiceOver reads plan, price, period and state ("Pro, monthly, ₹299, current plan, renews 12 December")

### 13.6 Testing the post-purchase UI

**Setup:** StoreKit Configuration file with the base group (Pro level 1, Plus level 2; monthly + yearly each) and an add-on group. Set renewal rate to a fast rate (e.g. "1 month = 1 minute"), enable Billing Grace Period, and use Xcode → Debug → StoreKit → **Manage Transactions** to expire, refund, toggle auto-renew, and fail renewals.

| # | Steps | Expected UI |
|---|---|---|
| U1 | Buy Plus monthly | Success "You're on Plus"; card "Plus · Monthly — Renews on …"; Plus row = Current plan; Pro row = Upgrade |
| U2 | From Plus, buy Pro | No custom alert; success "You're on Pro"; Pro features unlocked instantly; Plus row = Downgrade |
| U3 | From Pro, tap Plus | Downgrade alert lists lost features; Cancel → nothing changes |
| U4 | Confirm downgrade | Toast "Plus starts on …"; still Pro; banner "You'll move to Plus on …"; Plus row = "Starts …" disabled |
| U5 | Let the renewal pass (fast renewal rate) | Card switches to Plus; banner gone; Pro features locked |
| U6 | Pro monthly → Pro yearly | Crossgrade alert; scheduled at renewal; savings shown |
| U7 | Turn off auto-renew in Manage Transactions | Card says "Ends on …"; "Resubscribe" visible |
| U8 | Fail renewal with grace enabled | ⚠️ grace banner with fix-by date; features still unlocked |
| U9 | Grace period ends without payment | Billing retry banner; features locked per policy |
| U10 | Expire subscription | "Your Pro plan ended on …"; paywall offers resubscribe |
| U11 | Refund the transaction | Back to free; neutral message |
| U12 | Ask to Buy enabled, buy Pro | "Waiting for approval"; approve in Transaction Manager → unlocks without relaunch |
| U13 | Buy add-on, then expire base | Add-on shown paused + charged-warning banner |
| U14 | Own Lifetime, tap a subscription | Lifetime alert |
| U15 | Relaunch app in each state above | Same UI restored (state comes from StoreKit, not local flags) |
| U16 | Dynamic Type XXXL, Dark Mode, RTL, iPhone SE | No truncation; labels readable |

Automate U1, U2, U4, U5, U10, U11 with `StoreKitTest` (see Appendix A.7). Test U3, U6, U14 alerts with UI tests or snapshot tests.

## 14. Newer StoreKit APIs & platform rules

Verify OS availability against the current SDK before relying on any of these.

### 14.1 SwiftUI merchandising views (iOS 17+)
- `SubscriptionStoreView(groupID:)` — Apple-built, compliance-friendly subscription paywall (shows prices, periods, intro offers, renewal terms). Customize with `.subscriptionStoreControlStyle`, `.subscriptionStoreButtonLabel`, `.storeButton(.visible, for: .restorePurchases, .redeemCode)`, `.subscriptionStorePolicyDestination(url:for:)` for Terms/Privacy
- `ProductView(id:)` / `StoreView(ids:)` for one-time products
- React to results with `.onInAppPurchaseCompletion { product, result in … }` and `.onInAppPurchaseStart` — still refresh entitlements via the `Transaction.updates` listener; these views don't replace it
- [ ] Good default for simple apps; use a custom paywall (Appendix A.4) only when you need plan-change labels, add-on rules, or custom design
- [ ] Still needs Terms of Use + Privacy Policy links configured

### 14.2 Purchase action & confirmation scene
- [ ] In SwiftUI (iOS 17+), prefer `@Environment(\.purchase) private var purchase` → `try await purchase(product, options: …)`; it attaches the right scene for the payment sheet
- [ ] Outside SwiftUI on iOS 17+/visionOS/macOS multi-window, use `product.purchase(confirmIn: scene, options:)`
- [ ] (iOS 18.4+) `Transaction.currentEntitlements(for: productID)` (a sequence, covering Family Sharing + own purchase) replaces the deprecated single `currentEntitlement(for:)`

### 14.3 Promoted in-app purchases (iOS 16.4+)
Products promoted on the App Store product page open the app with a purchase intent.
- [ ] Listen for `PurchaseIntent.intents` at launch (next to `Transaction.updates`) and call `purchase()` for `intent.product` — or defer if the user must sign in / finish onboarding first, then resume
- [ ] Don't offer promoted products the user already owns (control order/visibility with `Product.PromotionInfo` on iOS 18+)

```swift
Task {
    for await intent in PurchaseIntent.intents {
        await store.purchase(intent.product)   // or queue until onboarding completes
    }
}
```

### 14.4 Paid → freemium migration (`AppTransaction`)
- [ ] `let result = try await AppTransaction.shared` → if `.verified(let app)`, compare `app.originalAppVersion` (the **CFBundleVersion** build number on iOS, not the marketing version) with the last paid build
- [ ] Grandfathered users get the "lifetime" entitlement in `refreshEntitlements()`; record it server-side too
- [ ] `AppTransaction` also gives `originalPurchaseDate` and `environment` — useful for fraud checks and for routing sandbox vs production to your backend
- [ ] Test: in Xcode StoreKit config, set the *Original Purchase Version*

### 14.5 App Store Server API — beyond validation
| Endpoint | Use |
|---|---|
| Get All Subscription Statuses | Source of truth when notifications arrive out of order |
| Get Transaction History (v2) | Rebuild a user's ledger, support tickets |
| Look Up Order ID | User sends the order ID from Apple's email receipt → find the transaction |
| Send Consumption Information | Reply to `CONSUMPTION_REQUEST` (refund decisions) |
| Extend Subscription Renewal Date / mass extend | Compensate users after an outage |
| Request a Test Notification | Verify your webhook end to end |

### 14.6 External purchases & link-outs (region-specific)
Rules change often — **always check the current App Review Guidelines and Apple's storefront-specific entitlement docs** before implementing.
- [ ] Any link to an external website for buying digital goods must match a rule that applies to that storefront (e.g. EU alternative-terms entitlements, US storefront link-out rules after the 2025 court ruling, reader-app entitlement, Korea/Netherlands programs)
- [ ] Gate the link by storefront (`Storefront.current?.countryCode`) and, where required, use `ExternalPurchase` / `ExternalPurchaseLink` APIs and required disclosure sheets
- [ ] Keep IAP available alongside the external option unless the rule explicitly allows otherwise

### 14.7 Xcode tooling
- [ ] **Sync** the StoreKit Configuration file with App Store Connect (Xcode: *Editor → Sync with App Store Connect*) instead of hand-copying products
- [ ] Xcode 16+ StoreKit testing supports win-back offers, offer codes, and Family Sharing toggles in the Transaction Manager

---

## Appendix — Reference implementation (StoreKit 2, iOS 17+, Swift 6)

Adapt IDs and policies. For iOS 15/16, replace `@Observable` with `ObservableObject` + `@Published`, and replace `status(for: groupID)` with `product.subscription?.status`. Verify APIs against the current Xcode SDK before shipping.

### A.1 Catalog — explicit entitlement map

```swift
import Foundation
import StoreKit

enum Tier: Int, Comparable, Sendable {
    case free = 0, plus, pro
    static func < (l: Tier, r: Tier) -> Bool { l.rawValue < r.rawValue }
}

enum ProductCatalog {
    // Base plans — ONE subscription group (levels set in App Store Connect)
    static let tierByProductID: [String: Tier] = [
        "com.app.sub.plus.monthly": .plus,
        "com.app.sub.plus.annual":  .plus,
        "com.app.sub.pro.monthly":  .pro,
        "com.app.sub.pro.annual":   .pro,
    ]
    // Add-ons — EACH in its OWN subscription group
    static let addonIDs: Set<String> = ["com.app.addon.cloudbackup.monthly"]
    // Non-consumable
    static let lifetimeID = "com.app.lifetime"
    // Consumables → credits
    static let creditsByProductID: [String: Int] = [
        "com.app.credits.100": 100,
        "com.app.credits.500": 500,
    ]
    // Non-renewing passes → duration
    static let passDurationByProductID: [String: DateComponents] = [
        "com.app.pass.3months": DateComponents(month: 3),
    ]

    static var allIDs: Set<String> {
        Set(tierByProductID.keys)
            .union(addonIDs)
            .union([lifetimeID])
            .union(creditsByProductID.keys)
            .union(passDurationByProductID.keys)
    }
}
```

### A.2 Consumable ledger — idempotent and persisted

Prefer a server ledger keyed by `transaction.id`. The local version below is for prototypes/offline-only apps.

```swift
protocol ConsumableLedger: Sendable {
    func balance() async -> Int
    /// Idempotent: crediting the same transaction twice has no effect. Returns new balance.
    func credit(transactionID: UInt64, amount: Int) async throws -> Int
    /// Idempotent refund handling. Returns new balance.
    func revoke(transactionID: UInt64, amount: Int) async throws -> Int
}

actor LocalConsumableLedger: ConsumableLedger {
    private let balanceKey = "iap.credits.balance"
    private let processedKey = "iap.credits.processed"
    private var defaults: UserDefaults { .standard }

    func balance() -> Int { defaults.integer(forKey: balanceKey) }

    func credit(transactionID: UInt64, amount: Int) throws -> Int {
        apply(key: "credit-\(transactionID)", delta: amount)
    }

    func revoke(transactionID: UInt64, amount: Int) throws -> Int {
        apply(key: "revoke-\(transactionID)", delta: -amount)
    }

    private func apply(key: String, delta: Int) -> Int {
        var processed = Set(defaults.stringArray(forKey: processedKey) ?? [])
        guard !processed.contains(key) else { return balance() }
        let newBalance = max(0, balance() + delta)
        defaults.set(newBalance, forKey: balanceKey)
        processed.insert(key)
        defaults.set(Array(processed), forKey: processedKey)
        return newBalance
    }
}
```

### A.3 StoreManager

```swift
import StoreKit
import SwiftUI

/// Everything the UI needs to describe ONE subscription group (see section 13).
struct GroupStatus: Equatable, Sendable {
    var productID: String                       // plan the user is on now
    var state: Product.SubscriptionInfo.RenewalState
    var expirationDate: Date?                   // renews / ends on
    var willAutoRenew: Bool                     // false → cancelled, ends on expirationDate
    var pendingProductID: String?               // scheduled downgrade / crossgrade
    var gracePeriodExpirationDate: Date?        // payment must be fixed by
    var isIntroOffer: Bool                      // free trial / intro price running

    var hasAccess: Bool { state == .subscribed || state == .inGracePeriod }
    var hasBillingIssue: Bool { state == .inGracePeriod || state == .inBillingRetryPeriod }
}

struct Entitlements: Equatable, Sendable {
    var tier: Tier = .free
    var hasLifetime = false
    var activeAddonIDs: Set<String> = []
    var passExpiry: Date?
    /// groupID → status of that group (base plan group + one per add-on)
    var groups: [String: GroupStatus] = [:]

    var hasBasePlan: Bool { tier > .free }
    /// Policy: add-ons only usable while a base plan is active.
    var usableAddonIDs: Set<String> { hasBasePlan ? activeAddonIDs : [] }
    var hasActivePass: Bool { (passExpiry ?? .distantPast) > .now }
}

@MainActor
@Observable
final class StoreManager {
    // Products, grouped by the entitlement map (NOT by product.type alone)
    private(set) var basePlans: [Product] = []
    private(set) var addons: [Product] = []
    private(set) var nonConsumables: [Product] = []
    private(set) var consumables: [Product] = []
    private(set) var passes: [Product] = []
    private(set) var introEligibleGroups: Set<String> = []
    private(set) var productsByID: [String: Product] = [:]

    private(set) var entitlements = Entitlements()
    var lastOutcome: PurchaseOutcome?                // drives post-purchase UI (section 13.4); UI clears it
    private(set) var creditBalance = 0          // separate from entitlements — never reset by refresh
    private(set) var purchaseInFlight = false
    private(set) var awaitingApproval = false   // Ask to Buy / SCA
    var errorMessage: String?

    /// Set when the user signs in; links purchases to your account system.
    var appAccountToken: UUID?

    private let ledger: ConsumableLedger
    private var updatesTask: Task<Void, Never>?

    /// Create ONCE at app launch (see A.5) so the listener runs for the app's lifetime.
    init(ledger: ConsumableLedger = LocalConsumableLedger()) {
        self.ledger = ledger
        updatesTask = Task { [weak self] in
            for await result in Transaction.updates {
                await self?.handle(result)
            }
        }
        Task { await start() }
    }

    func start() async {
        await loadProducts()
        for await result in Transaction.unfinished {
            await handle(result, refresh: false)
        }
        creditBalance = await ledger.balance()
        await refreshEntitlements()
    }

    // MARK: Products

    func loadProducts() async {
        do {
            let products = try await Product.products(for: ProductCatalog.allIDs)
            productsByID = Dictionary(uniqueKeysWithValues: products.map { ($0.id, $0) })
            basePlans = products.filter { ProductCatalog.tierByProductID[$0.id] != nil }
                                .sorted { $0.price < $1.price }
            addons = products.filter { ProductCatalog.addonIDs.contains($0.id) }
            nonConsumables = products.filter { $0.id == ProductCatalog.lifetimeID }
            consumables = products.filter { ProductCatalog.creditsByProductID[$0.id] != nil }
                                  .sorted { $0.price < $1.price }
            passes = products.filter { ProductCatalog.passDurationByProductID[$0.id] != nil }

            var eligible: Set<String> = []
            for product in basePlans + addons {
                guard let sub = product.subscription else { continue }
                if await sub.isEligibleForIntroOffer { eligible.insert(sub.subscriptionGroupID) }
            }
            introEligibleGroups = eligible

            let missing = ProductCatalog.allIDs.subtracting(products.map(\.id))
            if !missing.isEmpty { print("[IAP] Missing products: \(missing)") }
        } catch {
            errorMessage = "Couldn't load the store. Please try again."
        }
    }

    // MARK: Purchase

    func canPurchase(_ product: Product) -> Bool {
        // Add-ons require an active base plan
        if ProductCatalog.addonIDs.contains(product.id) { return entitlements.hasBasePlan }
        return true
    }

    func purchase(_ product: Product, options: Set<Product.PurchaseOption> = []) async {
        guard !purchaseInFlight, canPurchase(product) else { return }
        purchaseInFlight = true
        defer { purchaseInFlight = false }

        var opts = options
        if let token = appAccountToken { opts.insert(.appAccountToken(token)) }

        do {
            switch try await product.purchase(options: opts) {
            case .success(let verification):
                guard case .verified = verification else {
                    errorMessage = "We couldn't verify this purchase. Please try again or contact support."
                    return
                }
                await handle(verification)
                // Decide what to tell the user from the REFRESHED state, not the transaction:
                // an upgrade is active now; a downgrade/crossgrade is only scheduled.
                if let groupID = product.subscription?.subscriptionGroupID,
                   let group = entitlements.groups[groupID], group.productID != product.id {
                    lastOutcome = .scheduled(productID: product.id, startDate: group.expirationDate)
                } else {
                    lastOutcome = .activated(productID: product.id)
                }
            case .pending:
                awaitingApproval = true          // completes later via Transaction.updates
                lastOutcome = .pendingApproval
            case .userCancelled:
                break                            // not an error — no alert
            @unknown default:
                break
            }
        } catch {
            errorMessage = "Purchase failed. Please try again."
        }
    }

    /// Promotional offer: signature MUST come from your server (key never in app).
    func purchase(_ product: Product, promotionalOfferID offerID: String,
                  signature: PromoOfferSignature) async {
        let option = Product.PurchaseOption.promotionalOffer(
            offerID: offerID,
            keyID: signature.keyID,
            nonce: signature.nonce,
            signature: signature.signature,
            timestamp: signature.timestamp
        )
        await purchase(product, options: [option])
    }

    // MARK: Transaction handling — grant → persist → finish → refresh

    private func handle(_ result: VerificationResult<Transaction>, refresh: Bool = true) async {
        guard case .verified(let tx) = result else {
            // Do not grant. Log for investigation (jailbreak / tampering / clock issues).
            print("[IAP] Unverified transaction ignored")
            return
        }

        switch tx.productType {
        case .consumable:
            if let amount = ProductCatalog.creditsByProductID[tx.productID] {
                do {
                    if tx.revocationDate == nil {
                        creditBalance = try await ledger.credit(transactionID: tx.id, amount: amount)
                    } else {
                        creditBalance = try await ledger.revoke(transactionID: tx.id, amount: amount)
                    }
                } catch {
                    // Not persisted → do NOT finish; StoreKit will redeliver.
                    errorMessage = "Couldn't save your purchase. It will be retried."
                    return
                }
            }
        case .nonRenewable:
            // If you have a backend, send the transaction to it here (cross-device requirement).
            break
        default:
            break // subscriptions & non-consumables are derived in refreshEntitlements()
        }

        await tx.finish()
        awaitingApproval = false
        if refresh { await refreshEntitlements() }
    }

    // MARK: Entitlements

    func refreshEntitlements() async {
        var e = Entitlements()

        for await result in Transaction.currentEntitlements {
            guard case .verified(let tx) = result,
                  tx.revocationDate == nil,
                  !tx.isUpgraded else { continue }
            if let exp = tx.expirationDate, exp < .now { continue }

            switch tx.productType {
            case .autoRenewable:
                if let tier = ProductCatalog.tierByProductID[tx.productID] {
                    e.tier = max(e.tier, tier)
                } else if ProductCatalog.addonIDs.contains(tx.productID) {
                    e.activeAddonIDs.insert(tx.productID)
                }
            case .nonConsumable:
                if tx.productID == ProductCatalog.lifetimeID { e.hasLifetime = true }
            default:
                break
            }
        }

        if e.hasLifetime { e.tier = max(e.tier, .pro) }   // policy: lifetime == Pro

        // Non-renewing passes: compute and STACK expiry yourself
        var passPurchases: [(date: Date, duration: DateComponents)] = []
        for await result in Transaction.all {
            guard case .verified(let tx) = result,
                  tx.productType == .nonRenewable,
                  tx.revocationDate == nil,
                  let duration = ProductCatalog.passDurationByProductID[tx.productID] else { continue }
            passPurchases.append((tx.purchaseDate, duration))
        }
        var expiry: Date?
        for p in passPurchases.sorted(by: { $0.date < $1.date }) {
            let start = max(p.date, expiry ?? p.date)
            expiry = Calendar.current.date(byAdding: p.duration, to: start)
        }
        e.passExpiry = expiry

        // Per-group status for the UI: current plan, renewal date, pending change, billing issues
        let groupIDs = Set((basePlans + addons).compactMap { $0.subscription?.subscriptionGroupID })
        for groupID in groupIDs {
            guard let statuses = try? await Product.SubscriptionInfo.status(for: groupID) else { continue }
            // Several statuses can exist (e.g. own + Family Sharing) — prefer one that grants access.
            let ordered = statuses.filter { $0.state == .subscribed || $0.state == .inGracePeriod }
                        + statuses.filter { $0.state != .subscribed && $0.state != .inGracePeriod }
            for status in ordered {
                guard case .verified(let renewal) = status.renewalInfo,
                      case .verified(let tx) = status.transaction else { continue }
                let pending = renewal.autoRenewPreference.flatMap { $0 != tx.productID ? $0 : nil }
                e.groups[groupID] = GroupStatus(
                    productID: tx.productID,
                    state: status.state,
                    expirationDate: tx.expirationDate,
                    willAutoRenew: renewal.willAutoRenew,
                    pendingProductID: pending,
                    gracePeriodExpirationDate: renewal.gracePeriodExpirationDate,
                    isIntroOffer: tx.offerType == .introductory   // iOS 17.2+: tx.offer?.type
                )
                break
            }
        }

        entitlements = e
    }

    // MARK: Restore

    func restorePurchases() async {
        do {
            try await AppStore.sync()
        } catch {
            errorMessage = "Restore failed. Please try again."
        }
        await refreshEntitlements()
    }
}

struct PromoOfferSignature: Sendable {
    let keyID: String
    let nonce: UUID
    let signature: Data
    let timestamp: Int
}
```

### A.4 Paywall (compliance-ready)

```swift
struct PaywallView: View {
    @Environment(StoreManager.self) private var store
    @State private var showRedeem = false
    @State private var showManage = false

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let outcome = store.lastOutcome {
                    PurchaseOutcomeBanner(outcome: outcome)   // A.6 — "You're on Pro" / "Plus starts on …"
                }

                if store.baseGroupStatus() != nil {
                    SubscriptionStatusCard()             // A.6 — current / cancelled / expired plan + banners
                    Text("Change plan").font(.headline)
                } else {
                    Text("Unlock Premium").font(.largeTitle.bold())
                }

                // Base plans — labelled Current / Upgrade / Downgrade / Switch (A.6)
                ForEach(store.basePlans) { product in
                    PlanOptionRow(product: product)
                }

                // Add-ons — only when a base plan is active
                if store.entitlements.hasBasePlan, !store.addons.isEmpty {
                    Text("Add-ons").font(.headline)
                    ForEach(store.addons) { PlanOptionRow(product: $0) }
                }

                // One-time
                ForEach(store.nonConsumables + store.consumables + store.passes) { product in
                    PlanRow(product: product)
                }

                if store.awaitingApproval {
                    Text("Waiting for approval…").font(.footnote)
                }

                // Required disclosures (Guideline 3.1.2)
                Text("Subscriptions renew automatically at the price shown unless cancelled at least 24 hours before the end of the current period. Manage or cancel in Settings.")
                    .font(.caption2).foregroundStyle(.secondary)

                HStack {
                    Link("Terms of Use", destination: URL(string: "https://example.com/terms")!)
                    Link("Privacy Policy", destination: URL(string: "https://example.com/privacy")!)
                }.font(.footnote)

                HStack {
                    Button("Restore Purchases") { Task { await store.restorePurchases() } }
                    Button("Redeem Code") { showRedeem = true }
                    Button("Manage") { showManage = true }
                }.font(.footnote)
            }
            .padding()
        }
        .disabled(store.purchaseInFlight)
        .offerCodeRedemption(isPresented: $showRedeem) { _ in
            // Redemption arrives via Transaction.updates; nothing else needed here.
        }
        .manageSubscriptionsSheet(isPresented: $showManage)
    }
}

struct PlanRow: View {
    @Environment(StoreManager.self) private var store
    let product: Product

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(product.displayName).font(.headline)
                Text(product.description).font(.caption).foregroundStyle(.secondary)
                if let sub = product.subscription {
                    if let intro = sub.introductoryOffer,
                       store.introEligibleGroups.contains(sub.subscriptionGroupID) {
                        Text("Intro: \(intro.displayPrice) for \(intro.period.value) \(String(describing: intro.period.unit)), then \(product.displayPrice)")
                            .font(.caption2)
                    }
                }
            }
            Spacer()
            Button(product.displayPrice) { Task { await store.purchase(product) } }
                .buttonStyle(.borderedProminent)
                .disabled(!store.canPurchase(product))
        }
    }
}
```

### A.5 App entry — listener starts at launch

```swift
@main
struct MyApp: App {
    @State private var store = StoreManager()   // created once, at launch
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(store)
                .task { await store.refreshEntitlements() }
        }
        // Expiries and refunds that happened while backgrounded show up on return.
        .onChange(of: scenePhase) { _, phase in
            if phase == .active { Task { await store.refreshEntitlements() } }
        }
    }
}
```

If you use promoted IAPs, start the `PurchaseIntent.intents` listener here too (section 14.3).

### A.6 Post-purchase UI — status card, plan-change rows, alerts, feedback

```swift
import StoreKit
import SwiftUI

enum PurchaseOutcome: Equatable, Sendable {
    case activated(productID: String)                     // new purchase / upgrade / immediate switch
    case scheduled(productID: String, startDate: Date?)   // downgrade / crossgrade at renewal
    case pendingApproval                                  // Ask to Buy / SCA
}

enum PlanChange: Equatable {
    case new, current, scheduled, upgrade, downgrade, switchNow, switchAtRenewal
}

extension ProductCatalog {
    /// Used in the downgrade alert ("you'll lose: …"). Keep in sync with your feature gates.
    static let featuresByTier: [Tier: [String]] = [
        .free: [],
        .plus: ["Ad-free", "Cloud sync"],
        .pro:  ["Ad-free", "Cloud sync", "Offline mode", "4K export"],
    ]
}

extension StoreManager {
    var baseGroupID: String? { basePlans.first?.subscription?.subscriptionGroupID }

    func baseGroupStatus() -> GroupStatus? {
        baseGroupID.flatMap { entitlements.groups[$0] }
    }

    /// Compares a product with the user's current plan in the same group.
    func planChange(to product: Product) -> PlanChange {
        guard let sub = product.subscription,
              let current = entitlements.groups[sub.subscriptionGroupID], current.hasAccess,
              let currentSub = productsByID[current.productID]?.subscription else { return .new }

        if product.id == current.productID { return .current }
        if product.id == current.pendingProductID { return .scheduled }
        // Lower groupLevel number = higher tier
        if sub.groupLevel < currentSub.groupLevel { return .upgrade }
        if sub.groupLevel > currentSub.groupLevel { return .downgrade }
        return sub.subscriptionPeriod == currentSub.subscriptionPeriod ? .switchNow : .switchAtRenewal
    }

    func featuresLost(from currentID: String, to targetID: String) -> [String] {
        let from = ProductCatalog.tierByProductID[currentID] ?? .free
        let to = ProductCatalog.tierByProductID[targetID] ?? .free
        let keep = Set(ProductCatalog.featuresByTier[to] ?? [])
        return (ProductCatalog.featuresByTier[from] ?? []).filter { !keep.contains($0) }
    }
}

// MARK: Formatting helpers

extension Product.SubscriptionPeriod {
    var label: String {
        switch unit {
        case .day:   return value == 1 ? "Daily" : "\(value) days"
        case .week:  return value == 1 ? "Weekly" : "\(value) weeks"
        case .month: return value == 1 ? "Monthly" : "\(value) months"
        case .year:  return value == 1 ? "Yearly" : "\(value) years"
        @unknown default: return ""
        }
    }
}

private func formatDate(_ date: Date?) -> String {
    date?.formatted(date: .abbreviated, time: .omitted) ?? "—"
}

struct Banner: View {
    enum Kind { case info, warning, error }
    let kind: Kind
    let text: String

    var body: some View {
        Label(text, systemImage: kind == .info ? "info.circle"
                                : kind == .warning ? "exclamationmark.triangle" : "xmark.octagon")
            .font(.footnote)
            .padding(10)
            .frame(maxWidth: .infinity, alignment: .leading)
            .background(kind == .info ? Color.blue.opacity(0.12)
                        : kind == .warning ? Color.orange.opacity(0.15) : Color.red.opacity(0.15),
                        in: RoundedRectangle(cornerRadius: 10))
    }
}

// MARK: Current plan card (section 13.1)

struct SubscriptionStatusCard: View {
    @Environment(StoreManager.self) private var store
    @State private var showManage = false

    var body: some View {
        if let group = store.baseGroupStatus(), let product = store.productsByID[group.productID] {
            VStack(alignment: .leading, spacing: 8) {
                Text("\(product.displayName) · \(product.subscription?.subscriptionPeriod.label ?? "")")
                    .font(.title3.bold())
                Text(statusLine(group, product))
                    .font(.subheadline).foregroundStyle(.secondary)

                if let pendingID = group.pendingProductID, let next = store.productsByID[pendingID] {
                    Banner(kind: .info, text: "You'll move to \(next.displayName) on \(formatDate(group.expirationDate)). You keep \(product.displayName) until then.")
                }
                if group.state == .inGracePeriod {
                    Banner(kind: .warning, text: "Payment problem. Update your payment method by \(formatDate(group.gracePeriodExpirationDate ?? group.expirationDate)) to keep \(product.displayName).")
                } else if group.state == .inBillingRetryPeriod {
                    Banner(kind: .error, text: "We couldn't renew \(product.displayName). Update your payment method to restore access.")
                }

                // Add-ons whose base plan is gone (still billing!)
                ForEach(Array(store.entitlements.activeAddonIDs), id: \.self) { addonID in
                    if !store.entitlements.hasBasePlan, let addon = store.productsByID[addonID] {
                        Banner(kind: .warning, text: "\(addon.displayName) needs an active plan. You're still being charged for it.")
                    }
                }

                Button(buttonTitle(group)) { showManage = true }
                    .buttonStyle(.bordered)
            }
            .padding()
            .background(.thinMaterial, in: RoundedRectangle(cornerRadius: 16))
            .manageSubscriptionsSheet(isPresented: $showManage)
            .onChange(of: showManage) { _, isShown in
                if !isShown { Task { await store.refreshEntitlements() } }   // reflect cancel / resubscribe
            }
        }
    }

    private func statusLine(_ g: GroupStatus, _ p: Product) -> String {
        if g.state == .expired { return "Ended on \(formatDate(g.expirationDate))" }
        if g.state == .revoked { return "This plan was refunded" }
        if !g.willAutoRenew { return "Ends on \(formatDate(g.expirationDate))" }
        if g.isIntroOffer { return "Intro offer ends \(formatDate(g.expirationDate)), then \(p.displayPrice)" }
        return "Renews on \(formatDate(g.expirationDate)) for \(p.displayPrice)"
    }

    private func buttonTitle(_ g: GroupStatus) -> String {
        if g.hasBillingIssue { return "Update payment" }
        if g.state == .expired || !g.willAutoRenew || g.pendingProductID != nil { return "Manage / Resubscribe" }
        return "Manage subscription"
    }
}

// MARK: Plan rows with Upgrade / Downgrade / Switch + confirmation alerts (13.2, 13.3)

struct PlanOptionRow: View {
    @Environment(StoreManager.self) private var store
    let product: Product

    private enum Confirm: Identifiable { case downgrade, switchLater, lifetime; var id: Self { self } }
    @State private var confirm: Confirm?

    var body: some View {
        let change = store.planChange(to: product)
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(product.displayName).font(.headline)
                Text("\(product.displayPrice) · \(product.subscription?.subscriptionPeriod.label ?? "One-time")")
                    .font(.subheadline)
                if let note = note(change) {
                    Text(note).font(.caption2).foregroundStyle(.secondary)
                }
                if ProductCatalog.addonIDs.contains(product.id), !store.entitlements.hasBasePlan {
                    Text("Requires an active plan").font(.caption2).foregroundStyle(.secondary)
                }
            }
            Spacer()
            Button(label(change)) { tapped(change) }
                .buttonStyle(.borderedProminent)
                .disabled(change == .current || change == .scheduled
                          || !store.canPurchase(product) || store.purchaseInFlight)
        }
        .alert(alertTitle, isPresented: Binding(get: { confirm != nil },
                                                set: { if !$0 { confirm = nil } })) {
            Button(confirmTitle, role: confirm == .downgrade ? .destructive : nil) {
                Task { await store.purchase(product) }
            }
            Button("Cancel", role: .cancel) {}
        } message: {
            Text(alertMessage)
        }
    }

    private func tapped(_ change: PlanChange) {
        switch change {
        case .downgrade:       confirm = .downgrade
        case .switchAtRenewal: confirm = .switchLater
        case .new where store.entitlements.hasLifetime && product.subscription != nil
                     && !ProductCatalog.addonIDs.contains(product.id):
            confirm = .lifetime
        default:
            Task { await store.purchase(product) }   // first purchase / upgrade: system sheet is enough
        }
    }

    private func label(_ c: PlanChange) -> String {
        switch c {
        case .current:         return "Current plan"
        case .scheduled:       return "Starts \(formatDate(store.baseGroupStatus()?.expirationDate))"
        case .upgrade:         return "Upgrade"
        case .downgrade:       return "Downgrade"
        case .switchNow:       return "Switch"
        case .switchAtRenewal: return "Switch to \(product.subscription?.subscriptionPeriod.label.lowercased() ?? "")"
        case .new:             return ProductCatalog.addonIDs.contains(product.id) ? "Add" : "Subscribe"
        }
    }

    private func note(_ c: PlanChange) -> String? {
        let date = formatDate(store.baseGroupStatus()?.expirationDate)
        switch c {
        case .upgrade:         return "Starts now. You'll get a prorated refund for unused time."
        case .downgrade:       return "Starts on \(date). You keep your current plan until then."
        case .switchNow:       return "Starts now."
        case .switchAtRenewal: return "Starts on \(date)."
        case .scheduled:       return "Scheduled — you keep your current plan until then."
        default:               return nil
        }
    }

    private var currentName: String {
        store.baseGroupStatus().flatMap { store.productsByID[$0.productID]?.displayName } ?? "your plan"
    }

    private var alertTitle: String {
        switch confirm {
        case .downgrade:   return "Downgrade to \(product.displayName)?"
        case .switchLater: return "Switch to \(product.displayName)?"
        case .lifetime:    return "You already have Lifetime access"
        case nil:          return ""
        }
    }

    private var alertMessage: String {
        let date = formatDate(store.baseGroupStatus()?.expirationDate)
        switch confirm {
        case .downgrade:
            let lost = store.baseGroupStatus().map { store.featuresLost(from: $0.productID, to: product.id) } ?? []
            let lostText = lost.isEmpty ? "" : " After that you'll lose: \(lost.joined(separator: ", "))."
            return "You'll keep \(currentName) until \(date).\(lostText)"
        case .switchLater:
            return "Your new plan starts on \(date), when your current plan ends."
        case .lifetime:
            return "Your Lifetime purchase already includes these features. Subscribe anyway?"
        case nil:
            return ""
        }
    }

    private var confirmTitle: String {
        switch confirm {
        case .downgrade: return "Downgrade"
        case .lifetime:  return "Subscribe anyway"
        default:         return "Switch"
        }
    }
}

// MARK: Post-purchase feedback (13.4)

struct PurchaseOutcomeBanner: View {
    @Environment(StoreManager.self) private var store
    let outcome: PurchaseOutcome

    var body: some View {
        Banner(kind: .info, text: message)
            .task(id: outcome) {
                try? await Task.sleep(for: .seconds(5))
                store.lastOutcome = nil
            }
    }

    private var message: String {
        switch outcome {
        case .activated(let id):
            return "You're on \(store.productsByID[id]?.displayName ?? "your new plan") 🎉"
        case .scheduled(let id, let start):
            return "Change scheduled. \(store.productsByID[id]?.displayName ?? "Your new plan") starts on \(formatDate(start))."
        case .pendingApproval:
            return "Waiting for approval. We'll unlock it as soon as it's approved."
        }
    }
}
```

### A.7 Automated plan-change tests (StoreKitTest)

Run against a `Products.storekit` configuration with the IDs from A.1. `SKTestSession` method names can change between Xcode versions — check the current StoreKitTest docs if one doesn't compile.

```swift
import StoreKit
import StoreKitTest
import XCTest
@testable import MyApp

@MainActor
final class PlanChangeTests: XCTestCase {
    private var session: SKTestSession!

    override func setUp() async throws {
        session = try SKTestSession(configurationFileNamed: "Products")
        session.resetToDefaultState()
        session.clearTransactions()
        session.disableDialogs = true
    }

    private func product(_ id: String) async throws -> Product {
        try XCTUnwrap(try await Product.products(for: [id]).first)
    }

    private func makeStore() async -> StoreManager {
        let store = StoreManager()
        await store.start()
        return store
    }

    func testUpgradeIsImmediate() async throws {                       // U1, U2
        let store = await makeStore()
        await store.purchase(try await product("com.app.sub.plus.monthly"))
        XCTAssertEqual(store.entitlements.tier, .plus)

        let pro = try await product("com.app.sub.pro.monthly")
        XCTAssertEqual(store.planChange(to: pro), .upgrade)
        await store.purchase(pro)
        XCTAssertEqual(store.entitlements.tier, .pro)
        XCTAssertEqual(store.lastOutcome, .activated(productID: pro.id))
    }

    func testDowngradeIsScheduledThenAppliedAtRenewal() async throws {  // U4, U5
        let store = await makeStore()
        await store.purchase(try await product("com.app.sub.pro.monthly"))

        let plus = try await product("com.app.sub.plus.monthly")
        XCTAssertEqual(store.planChange(to: plus), .downgrade)
        await store.purchase(plus)

        XCTAssertEqual(store.entitlements.tier, .pro)                   // still Pro until renewal
        XCTAssertEqual(store.baseGroupStatus()?.pendingProductID, plus.id)
        guard case .scheduled = store.lastOutcome else { return XCTFail("Expected .scheduled") }

        try session.forceRenewalOfSubscription(productIdentifier: "com.app.sub.pro.monthly")
        try await Task.sleep(for: .seconds(2))                          // let Transaction.updates deliver
        await store.refreshEntitlements()
        XCTAssertEqual(store.entitlements.tier, .plus)
    }

    func testExpiryRemovesAccess() async throws {                       // U10
        let store = await makeStore()
        await store.purchase(try await product("com.app.sub.pro.monthly"))
        try session.expireSubscription(productIdentifier: "com.app.sub.pro.monthly")
        await store.refreshEntitlements()
        XCTAssertEqual(store.entitlements.tier, .free)
        XCTAssertEqual(store.baseGroupStatus()?.state, .expired)
    }

    func testRefundRemovesAccess() async throws {                       // U11
        let store = await makeStore()
        await store.purchase(try await product("com.app.sub.pro.monthly"))
        let tx = try XCTUnwrap(session.allTransactions().first {
            $0.productIdentifier == "com.app.sub.pro.monthly"
        })
        try session.refundTransaction(identifier: tx.identifier)
        try await Task.sleep(for: .seconds(2))
        await store.refreshEntitlements()
        XCTAssertEqual(store.entitlements.tier, .free)
    }
}
```
