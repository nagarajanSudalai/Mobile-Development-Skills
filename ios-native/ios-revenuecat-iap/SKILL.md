---
name: ios-revenuecat-iap
description: Use when implementing, reviewing, testing, or debugging iOS in-app purchases or subscriptions built on RevenueCat (purchases-ios / RevenueCatUI) — SDK setup, offerings & packages, entitlements, paywalls, Customer Center, user identity (logIn/logOut, transfers), offers, webhooks / REST API, dashboard and App Store Connect configuration. For native StoreKit without RevenueCat, use the ios-in-app-purchase skill.
---

# iOS In-App Purchases with RevenueCat

Guide for building, auditing, and testing iOS purchases on top of the **RevenueCat iOS SDK** (`purchases-ios`, v5+, StoreKit 2 by default) and **RevenueCatUI** (Paywalls, Customer Center). RevenueCat handles receipt/JWS validation, the App Store Server Notifications pipeline, cross-device entitlement sync, promotional-offer signing, and analytics. **You still own**: App Store Connect setup, the entitlement design, user identity, the UI, consumable balances, and App Review compliance.

Companion skill: **ios-in-app-purchase** — use its sections for App Store Connect product/group setup (§3, §8), App Review rules (§9), and post-purchase UI patterns (§13). This skill says how to drive them from RevenueCat data. Android sibling: **android-revenuecat-iap**.

SDK APIs move between major versions. Before shipping, verify method names against the current RevenueCat iOS docs and the SDK version in `Package.resolved`.

## 0. Architecture (mental model)

```
+------------------+  Server Notifications V2   +-----------------------------+
| App Store Server | -------------------------> | RevenueCat backend          |
+------------------+                            | - validates purchases       |
        ^                                       | - CustomerInfo per user     |
        | StoreKit 2                            | - Offerings / Experiments   |
+------------------------------+  HTTPS         | - Webhooks, REST API, ETL   |
| RevenueCat SDK (on device)   | <------------> +-----------------------------+
| - Purchases.configure        |                      | webhooks (+ REST re-fetch)
| - offerings() / purchase()   |                      v
| - customerInfoStream         |               +---------------------------+
| - logIn / logOut / restore   |               | Your backend (optional)   |
+------------------------------+               | - mirror entitlements     |
        ^                                      | - consumable ledger       |
+-----------------------------------------+    +---------------------------+
| SubscriptionManager (@MainActor,        |
|  @Observable) — created once at launch  |
| - maps entitlement IDs → app features   |
+-----------------------------------------+
        ^
+-----------------------------------------+
| SwiftUI: RevenueCatUI PaywallView or     |
| custom paywall, status card, Customer    |
| Center / Manage, Restore                 |
+-----------------------------------------+
```

Three RevenueCat concepts — get these right first:

| Concept | What it is | Your code uses it for |
|---|---|---|
| **Product** | The App Store product (same ID as App Store Connect) | Rarely referenced directly |
| **Entitlement** | A level of access (`pro`, `cloud_backup`) that one or more products unlock | **All feature gating** |
| **Offering** → **Package** | A remotely configured set of products to show on a paywall (`$rc_monthly`, `$rc_annual`, `$rc_lifetime`, custom) | **What the paywall displays**, changeable without an app release |

## 1. First, establish context

Find out (from the codebase or by asking):

- Minimum iOS version and SDK version (`purchases-ios` 5.x = StoreKit 2 default; 4.x = StoreKit 1 default). Swift Package: `https://github.com/RevenueCat/purchases-ios-spm`, products `RevenueCat` and (optional) `RevenueCatUI`
- **Who completes purchases**: RevenueCat (default) or your own StoreKit code (`purchasesAreCompletedBy: .myApp`, formerly "observer mode") — section 11
- **Account system?** Anonymous-only, or your own user IDs (`logIn`). This decides restore/transfer behaviour
- Product catalog by type (consumable / non-consumable / auto-renewable + group / add-on / non-renewing) — same classification as ios-in-app-purchase §1
- Paywall: **RevenueCat Paywalls** (remote, template-based) or **custom SwiftUI**
- Backend that needs purchase state? (webhooks + REST API)
- Migrating from an existing native StoreKit implementation? (section 10)

Write the **entitlement map** before coding:

```
product ID (App Store) → RevenueCat entitlement(s) → app features
com.app.sub.plus.monthly → plus            → ad-free, sync
com.app.sub.pro.monthly  → plus, pro       → + offline, 4K export   (attach higher tiers to lower entitlements too)
com.app.lifetime         → plus, pro       → same as Pro
com.app.addon.backup     → cloud_backup    → backup (requires plus)
com.app.credits.100      → (none)          → +100 credits (ledger)
```

Code checks **entitlement IDs**, never product IDs. Attaching higher tiers to every lower entitlement keeps checks simple (`isActive("plus")` is true for Pro users too).

## 2. RevenueCat dashboard & App Store Connect configuration

### 2.1 App Store Connect (unchanged from native)
- [ ] Paid Apps agreement, tax, banking; products created with localizations, review screenshot
- [ ] Subscription groups and levels per ios-in-app-purchase §3; add-ons in **their own groups** (§4)
- [ ] Billing Grace Period enabled (RevenueCat keeps entitlements active during grace)

### 2.2 RevenueCat project
- [ ] App Store app added with the correct **bundle ID**
- [ ] **In-App Purchase Key** (.p8, key ID, issuer ID) uploaded — required for StoreKit 2 validation and for RevenueCat to sign promotional offers
- [ ] App Store Connect API key uploaded (optional; imports products and enables price/metadata features)
- [ ] **App Store Server Notifications V2** URL in App Store Connect set to the RevenueCat URL for **both** Production and Sandbox. If your server also needs Apple's raw notifications, use RevenueCat's notification forwarding — App Store Connect allows only one URL per environment
- [ ] Products imported; **entitlements** created and products attached per the entitlement map
- [ ] **Offerings** with packages; one marked **Current**. Use Placements / Targeting / Experiments only if you'll actually use them
- [ ] **Restore behavior** chosen deliberately (Project settings): *Transfer to new App User ID* (default), *Transfer if no active subscriptions*, or *Keep with original App User ID* — section 5.3
- [ ] Webhook URL + **Authorization header value** configured if you have a backend
- [ ] Public **Apple API key** (`appl_…`) goes in the app; **secret keys** (`sk_…`) only on the server

## 3. SDK setup checklist

- [ ] `Purchases.configure(...)` called **exactly once**, as early as possible (App `init` / `didFinishLaunching`), **before** anything touches `Purchases.shared`
- [ ] `Purchases.logLevel = .debug` only in DEBUG builds
- [ ] `appUserID` passed at configure time if the user is already signed in (avoids creating an anonymous user then aliasing)
- [ ] `entitlementVerificationMode(.informational)` (Trusted Entitlements) on; log/alert on `customerInfo.entitlements.verification == .failed`. Use `.enforced` only if you've tested the failure path
- [ ] The **In-App Purchase capability** is enabled in Xcode
- [ ] One long-lived observer of `Purchases.shared.customerInfoStream` (or a `PurchasesDelegate`) created at launch — this is how renewals, expirations, refunds, purchases from other devices, offer codes, and Ask-to-Buy approvals reach the UI
- [ ] No other code in the app calls `Transaction.finish()` or runs its own `Transaction.updates` granting logic while RevenueCat completes purchases (double handling). Read-only StoreKit 2 queries are fine (section 7.3)

## 4. Offerings, packages & the paywall

### 4.1 Loading
- [ ] `try await Purchases.shared.offerings()` → `offerings.current` (or `offerings.currentOffering(forPlacement:)` / `offering(identifier:)`)
- [ ] Handle `current == nil` and empty `availablePackages`: no current offering set, products not yet approved / not in the StoreKit config, agreements unsigned. Show a retry state, never a blank screen or a crash
- [ ] Package lookup by type (`offering.monthly`, `.annual`, `.lifetime`) or by identifier — **never by array index**
- [ ] Prices from `package.storeProduct.localizedPriceString` (and `localizedPricePerMonth` etc. for comparisons) — never hardcoded
- [ ] Paywall copy that varies by offering lives in **offering metadata**, read with defaults (`offering.getMetadataValue(for:default:)`), so remote changes can't break the UI

### 4.2 RevenueCat Paywalls (RevenueCatUI) vs custom

| | RevenueCatUI `PaywallView` | Custom SwiftUI paywall |
|---|---|---|
| Change design without release | ✅ | Only via offering metadata |
| A/B tests (Experiments) | ✅ | ✅ (offering-level) |
| Intro-eligibility, prices, localization | Handled | You implement (section 6) |
| Upgrade / Downgrade labels, add-on rules | Limited | ✅ full control |

- [ ] RevenueCatUI: `.presentPaywallIfNeeded(requiredEntitlementIdentifier:)` for gating; `PaywallView(offering:displayCloseButton:)` for explicit presentation; handle `.onPurchaseCompleted`, `.onRestoreCompleted`, `.onPurchaseCancelled` / errors
- [ ] Template includes **Terms of Use** and **Privacy Policy** links and the auto-renew disclosure (configure them in the paywall editor) — ios-in-app-purchase §9
- [ ] Close button visible (no dark patterns)
- [ ] Custom paywall: follow ios-in-app-purchase §13 for Current plan / Upgrade / Downgrade / Switch labels and alerts, driven by the data in section 7

## 5. User identity

### 5.1 Rules
- [ ] Anonymous users get `$RCAnonymousID:…` automatically — fine for apps without accounts
- [ ] With accounts: `logIn(yourStableUserID)` right after sign-in, **before** showing the paywall; `logOut()` on sign-out
- [ ] App User ID is a **stable, non-guessable, non-PII** ID (your DB UUID). Never email, device ID, IDFA, `""`, `"nil"`, `"0"`, or a hardcoded string — shared IDs merge strangers' purchases
- [ ] Don't call `logOut()` when `Purchases.shared.isAnonymous` (it errors)
- [ ] Same ID used on all platforms you support later (web, Android) so entitlements follow the user
- [ ] Set subscriber attributes you need for support/analytics (`Purchases.shared.attribution.setEmail(_:)`, `setDisplayName`, `setAttributes`) — only with consent and per your privacy policy

### 5.2 `appAccountToken`
RevenueCat sends its own identifiers to Apple. If your backend also needs Apple's `appAccountToken`, check the current SDK docs for how RevenueCat sets it rather than injecting it yourself.

### 5.3 Restore & transfers
- [ ] Restore button → `Purchases.shared.restorePurchases()`; if no entitlement becomes active, say "No previous purchases found" (not an error)
- [ ] Restore under a **different** App User ID triggers the project's restore behavior: default **transfers** the purchase to the current user (webhook `TRANSFER`). Decide whether that's acceptable for your account model (e.g. family members sharing an Apple ID)
- [ ] `syncPurchases()` is for **silent** migration of existing purchases (section 10), not for the Restore button

## 6. Purchasing

- [ ] `try await Purchases.shared.purchase(package:)` → `(transaction, customerInfo, userCancelled)`; apply the returned `customerInfo` immediately
- [ ] `userCancelled == true` / `ErrorCode.purchaseCancelledError` → **no alert**
- [ ] `ErrorCode.paymentPendingError` → Ask to Buy / SCA: show "Waiting for approval"; the result arrives later through `customerInfoStream`
- [ ] `networkError` / `offlineConnectionError` → retryable message. `productAlreadyPurchasedError`, `receiptAlreadyInUseError` (purchase belongs to another App User ID), `ineligibleError`, `storeProblemError` → short, specific messages
- [ ] Purchase button disabled while in flight
- [ ] Gate on **entitlements** after purchase (`customerInfo.entitlements[id]?.isActive`), not on "purchase returned without error"
- [ ] Add-ons purchasable only while the base entitlement is active (Apple doesn't enforce it; neither does RevenueCat)
- [ ] In-group upgrades/downgrades are normal `purchase(package:)` calls — StoreKit performs the change; don't block "already subscribed"

### 6.1 Offers
| Offer | How with RevenueCat |
|---|---|
| Introductory / free trial | Applied automatically by StoreKit. Show trial text only if `checkTrialOrIntroDiscountEligibility(packages:)` says `.eligible` (RevenueCat Paywalls do this for you) |
| Promotional | `promotionalOffer(forProductDiscount:product:)` → `purchase(package:promotionalOffer:)`. **RevenueCat signs it** using your uploaded IAP key — no signing server, no key in the app |
| Offer codes | Present Apple's sheet (`.offerCodeRedemption(isPresented:)` in SwiftUI); the redemption reaches `customerInfoStream` |
| Win-back (iOS 18+) | `eligibleWinBackOffers(forPackage:)` → `purchase(package:winBackOffer:)` (SDK 5.x; verify names) |
| Price increase consent | Apple's sheet; RevenueCat may defer/show it — check the SDK's `showStoreMessages` behaviour |

## 7. Reading state for the UI

### 7.1 Field mapping (`EntitlementInfo`)

| UI need (ios-in-app-purchase §13.1) | RevenueCat field |
|---|---|
| Has access | `isActive` (true during grace period) |
| Current plan | `productIdentifier` |
| Renews / ends on | `expirationDate` (+ `willRenew`) — `nil` expiration = lifetime |
| Cancelled | `willRenew == false` / `unsubscribeDetectedAt != nil` |
| Billing problem | `billingIssueDetectedAt != nil` (grace if still `isActive`, retry if not) |
| Trial / intro running | `periodType == .trial` / `.intro` |
| Sandbox | `isSandbox` |
| Bought on another store / web | `store` |
| Family Sharing | `ownershipType == .familyShared` |

`customerInfo.subscriptionsByProductIdentifier[productID]` (SDK 5.x) adds per-product detail (e.g. `gracePeriodExpiresDate`, `refundedAt`). `customerInfo.managementURL` / `Purchases.shared.showManageSubscriptions()` open subscription management; RevenueCatUI's **Customer Center** gives a ready-made manage/cancel/refund/support screen.

### 7.2 Freshness
- [ ] `customerInfoStream` drives the UI; it emits the cached value first, then updates
- [ ] Cached `CustomerInfo` is fine for gating; force a fetch (`customerInfo(fetchPolicy: .fetchCurrent)`) only where it matters (e.g. right before an expensive server action)
- [ ] Offline: the SDK serves cached / offline-computed entitlements; don't lock paying users out because a request failed

### 7.3 Hybrid: pending downgrades
`CustomerInfo` doesn't tell you which product renews next. For "You'll move to Plus on 12 Dec", read StoreKit 2 directly (read-only, safe alongside RevenueCat): `Product.SubscriptionInfo.status(for: groupID)` → `renewalInfo.autoRenewPreference` (Appendix A.5). Or send users to Customer Center.

## 8. Consumables & non-renewing purchases

- [ ] RevenueCat **finishes consumable transactions itself** — you can't rely on StoreKit redelivery. Credit balances **server-side from the `NON_RENEWING_PURCHASE` webhook**, idempotent by `transaction_id`; client-side crediting after `purchase()` is only for offline-only apps and must also be idempotent
- [ ] Alternatively evaluate RevenueCat **Virtual Currencies** (balances managed by RevenueCat) if available on your plan/SDK — check the docs
- [ ] Purchase history: `customerInfo.nonSubscriptions` (`transactionIdentifier`, `productIdentifier`, `purchaseDate`)
- [ ] Non-renewing passes: RevenueCat does **not** compute expiry — compute and **stack** it yourself (server preferred; client fallback in Appendix A.3), and sync across devices (App Review requirement)

## 9. Backend: webhooks & REST API

- [ ] Verify the `Authorization` header equals the value you configured; reject otherwise
- [ ] Idempotent by `event.id`; respond `200` quickly (RevenueCat retries on failure) and do slow work async
- [ ] Simplest robust pattern: on any event, **re-fetch** the subscriber (`GET /v1/subscribers/{app_user_id}` with a secret key, or the v2 API) and overwrite your mirror — avoids ordering bugs
- [ ] Handle `environment` (`SANDBOX` vs `PRODUCTION`) — App Review and TestFlight purchases are sandbox
- [ ] Identity: events carry `app_user_id`, `original_app_user_id`, `aliases`; `TRANSFER` uses `transferred_from` / `transferred_to`
- [ ] Secret API keys never in the app or in client-visible config

| Event `type` | Action |
|---|---|
| `INITIAL_PURCHASE` | Grant entitlement(s) in `entitlement_ids`; record `period_type` (TRIAL/INTRO/NORMAL) |
| `RENEWAL` | Extend expiry (also trial → paid conversion) |
| `PRODUCT_CHANGE` | Plan change; `new_product_id`. Downgrades take effect at renewal — re-fetch rather than switching tier immediately |
| `CANCELLATION` | Auto-renew off (`cancel_reason`); keep access until expiry; retention messaging. With `cancel_reason = CUSTOMER_SUPPORT` it can mean a refund — re-fetch |
| `UNCANCELLATION` | Auto-renew back on |
| `BILLING_ISSUE` | Prompt to fix payment; access per grace period |
| `EXPIRATION` | Remove access (`expiration_reason`) |
| `NON_RENEWING_PURCHASE` | Credit consumable / extend pass (idempotent by `transaction_id`) |
| `TRANSFER` | Move entitlements between `transferred_from` → `transferred_to` |
| `SUBSCRIPTION_EXTENDED` | Expiry moved later (e.g. Apple outage extension) |
| `TEMPORARY_ENTITLEMENT_GRANT` | Short grant while Apple is unreachable; will be confirmed or revoked |
| `REFUND_REVERSED` | Restore access / balance |
| `TEST` | Dashboard test event — return 200 |

Check the RevenueCat webhook docs for event types added after this skill was written.

## 10. Migrating an existing native StoreKit app

- [ ] Remove your own `Transaction.updates` granting/finishing logic once RevenueCat completes purchases (or use `purchasesAreCompletedBy: .myApp` during a transition, section 11)
- [ ] Existing subscribers: call `Purchases.shared.syncPurchases()` **once** per install after upgrading (e.g. guarded by a `UserDefaults` flag), and/or import historical purchases server-side via RevenueCat's import tools
- [ ] Keep product IDs unchanged; recreate the entitlement map in the dashboard
- [ ] Paid-to-freemium grandfathering still uses `AppTransaction.originalAppVersion` (ios-in-app-purchase §14.4); grant via your backend or RevenueCat promotional entitlements (REST API)
- [ ] Server: switch from your own App Store Server Notifications handler to RevenueCat webhooks (or forwarding) — don't run both as sources of truth

## 11. `purchasesAreCompletedBy: .myApp` (your StoreKit code completes purchases)

Use only if you must keep your own StoreKit purchase flow.
- [ ] Configure with `.with(purchasesAreCompletedBy: .myApp, storeKitVersion: .storeKit2)`
- [ ] After each StoreKit 2 purchase, report it to RevenueCat (SDK 5.x: `Purchases.shared.recordPurchase(_:)` with the `Product.PurchaseResult`) — verify the current API
- [ ] **You** still call `finish()`, handle `Transaction.updates`, and restore (all of ios-in-app-purchase §6 applies)
- [ ] Gating can still come from `CustomerInfo` — but don't mix two sources of truth for the same feature

## 12. Testing plan

1. **Xcode StoreKit Configuration file** — synced from App Store Connect; select it in the scheme. Follow RevenueCat's "Xcode StoreKit testing" doc (some setups need the config's public certificate uploaded to the dashboard). Purchases appear under sandbox data in the dashboard
2. **RevenueCat Test Store** (if available for your project/SDK) — test paywalls and entitlements without App Store products
3. **Sandbox** Apple Account on device — accelerated renewals; toggle **View sandbox data** in the RevenueCat dashboard
4. **TestFlight** — end to end including webhooks to your staging backend

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| R1 | Fresh install, no account | Anonymous ID; offerings load; paywall shows prices |
| R2 | `offerings.current == nil` (unset current / missing products) | Retry state, no crash; error logged |
| R3 | Subscribe with trial (eligible) | Trial text shown; entitlement active; `periodType == .trial` |
| R4 | Same user, second group trial | Eligibility per group respected |
| R5 | Cancel purchase sheet | No alert; nothing changes |
| R6 | Ask to Buy | "Waiting for approval"; approval unlocks via `customerInfoStream` without relaunch |
| R7 | Upgrade Plus → Pro | Pro active immediately; status card updates |
| R8 | Downgrade Pro → Plus | Still Pro; pending banner (hybrid A.5); switches at renewal |
| R9 | Renewal / expiry (accelerated) | Expiry extends / access removed; UI updates while app open |
| R10 | Billing issue with grace | Access kept; fix-payment banner from `billingIssueDetectedAt` |
| R11 | Refund (Transaction Manager / sandbox) | Access removed; `EXPIRATION`/`CANCELLATION` webhook handled |
| R12 | Anonymous purchase, then `logIn` | Purchase aliased to the account; entitlement kept |
| R13 | Restore on a **second** account (same Apple ID) | Behaves per restore-behavior setting; `TRANSFER` handled |
| R14 | `logOut` | Back to anonymous; premium UI hidden |
| R15 | Promotional offer | Discounted price; entitlement active |
| R16 | Offer code redemption | Entitlement active via stream |
| R17 | Consumable purchase | Webhook credits once; app shows server balance; replaying the webhook doesn't double-credit |
| R18 | Airplane mode launch as subscriber | Cached/offline entitlements keep access |
| R19 | Change Current offering in dashboard | New packages shown after relaunch, no release |
| R20 | Trusted Entitlements failure (tampered / MITM) | Logged; behaviour per chosen mode |
| R21 | Post-purchase UI states | Run ios-in-app-purchase §13.6 (U1–U16) against RevenueCat data |

## 13. Common bugs to look for in review

- `Purchases.configure` missing, called twice, or after `Purchases.shared` is accessed (crash / wrong user)
- Gating on `activeSubscriptions.contains("com.app.pro.monthly")` or product IDs instead of entitlements → yearly/lifetime/promo users locked out
- Entitlement checks on a `customerInfo` fetched once at launch; no `customerInfoStream` observer → stale UI after renewals, refunds, Ask to Buy
- Paywall picks packages by index (`availablePackages[0]`); crashes or shows the wrong plan when the offering changes
- No handling for `offerings.current == nil`
- `logIn` with email / device ID / empty or constant strings; `logIn` called after the purchase instead of before; `logOut` on anonymous users
- Restore button calling `syncPurchases()`, or sync called on every launch
- Error alert on user cancellation; pending (Ask to Buy) treated as failure
- Secret API key (`sk_…`) shipped in the app; public key from the wrong platform (`goog_…`)
- In-App Purchase Key not uploaded → StoreKit 2 validation / promotional offers fail
- App Store Server Notifications still pointing at an old server → cancellations and refunds arrive late
- App code still finishing transactions / running its own `Transaction.updates` granting alongside RevenueCat
- Consumables credited client-side without idempotency, or expected to be redelivered
- Trial text shown without an eligibility check
- Webhook handler without auth check, not idempotent, or switching tier immediately on a downgrade `PRODUCT_CHANGE`
- Sandbox events mixed into production analytics / entitlements
- Debug log level in release builds
- Paywall template without Terms/Privacy links or renewal disclosure

## 14. Output format

When asked to **review/audit**:
1. **Summary** — SDK version, who completes purchases, identity model (anonymous / logIn), paywall type, backend yes/no, entitlement map
2. **Checklist results** — sections 2–11 with ✅ Pass / ❌ Fail / ⚠️ Needs check, each with file:line (or "dashboard" for configuration items you can't see — list them for the user to confirm)
3. **Critical issues first** (lost access for paying users, wrong user merges, double credits, secret key exposure, rejection risks) with fix code
4. **Testing gaps** — scenarios R1–R21 not covered

When asked to **implement**: start from the Appendix, adapt `Entitlement` / `ProductCatalog` to the user's map, remove unused parts (add-ons, passes, consumables), then list dashboard, App Store Connect, backend, and compliance steps the user must do by hand.

---

## Appendix — Reference implementation (RevenueCat iOS SDK 5.x, iOS 17+, Swift 6)

Adapt IDs and policies. Verify API names against the SDK version you install.

### A.1 Catalog — entitlement IDs, not product IDs

```swift
import Foundation

enum Entitlement: String, CaseIterable, Sendable {
    case plus            // attached to Plus, Pro and Lifetime products
    case pro             // attached to Pro and Lifetime products
    case cloudBackup = "cloud_backup"   // add-on, own subscription group
}

enum Tier: Int, Comparable, Sendable {
    case free = 0, plus, pro
    static func < (l: Tier, r: Tier) -> Bool { l.rawValue < r.rawValue }
}

enum ProductCatalog {
    // Only for things RevenueCat doesn't model as entitlements
    static let creditsByProductID: [String: Int] = [
        "com.app.credits.100": 100,
        "com.app.credits.500": 500,
    ]
    static let passDurationByProductID: [String: DateComponents] = [
        "com.app.pass.3months": DateComponents(month: 3),
    ]
    static let addonEntitlements: Set<Entitlement> = [.cloudBackup]
}

enum RevenueCatConfig {
    // Public SDK key (appl_…) — safe in the app. Load per build configuration if you
    // use separate RevenueCat projects for staging and production.
    static let appleAPIKey = "appl_XXXXXXXXXXXXXXXX"
}
```

### A.2 App entry — configure once, before anything else

```swift
import RevenueCat
import SwiftUI

@main
struct MyApp: App {
    @State private var subscriptions: SubscriptionManager
    @Environment(\.scenePhase) private var scenePhase

    init() {
        #if DEBUG
        Purchases.logLevel = .debug
        #endif
        Purchases.configure(
            with: Configuration.Builder(withAPIKey: RevenueCatConfig.appleAPIKey)
                .with(appUserID: AuthStore.currentUserID)          // nil → anonymous
                .with(entitlementVerificationMode: .informational) // Trusted Entitlements
                .build()
        )
        _subscriptions = State(initialValue: SubscriptionManager())  // after configure
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(subscriptions)
        }
        .onChange(of: scenePhase) { _, phase in
            if phase == .active { Task { await subscriptions.refresh() } }
        }
    }
}
```

### A.3 SubscriptionManager

```swift
import RevenueCat
import SwiftUI

/// What the UI needs to describe the user's base plan (ios-in-app-purchase §13.1).
struct PlanStatus: Equatable, Sendable {
    var entitlementID: String
    var productID: String
    var isActive: Bool
    var expirationDate: Date?        // nil → lifetime
    var willRenew: Bool
    var isTrialOrIntro: Bool
    var billingIssueDetectedAt: Date?
    var isSandbox: Bool

    var isLifetime: Bool { isActive && expirationDate == nil }
    var inGracePeriod: Bool { isActive && billingIssueDetectedAt != nil }
    var inBillingRetry: Bool { !isActive && billingIssueDetectedAt != nil }

    init(_ info: EntitlementInfo) {
        entitlementID = info.identifier
        productID = info.productIdentifier
        isActive = info.isActive
        expirationDate = info.expirationDate
        willRenew = info.willRenew
        isTrialOrIntro = info.periodType == .trial || info.periodType == .intro
        billingIssueDetectedAt = info.billingIssueDetectedAt
        isSandbox = info.isSandbox
    }
}

struct AppEntitlements: Equatable, Sendable {
    var tier: Tier = .free
    var activeAddons: Set<Entitlement> = []
    var plan: PlanStatus?            // highest-tier entitlement, active or most recent
    var passExpiry: Date?

    var hasBasePlan: Bool { tier > .free }
    /// Policy: add-ons only usable while a base plan is active.
    var usableAddons: Set<Entitlement> { hasBasePlan ? activeAddons : [] }
    var hasActivePass: Bool { (passExpiry ?? .distantPast) > .now }
}

enum PurchaseOutcome: Equatable, Sendable {
    case activated(productID: String)
    case scheduled(productID: String)     // downgrade / crossgrade at renewal
    case pendingApproval                  // Ask to Buy / SCA
}

@MainActor
@Observable
final class SubscriptionManager {
    private(set) var offering: Offering?
    private(set) var entitlements = AppEntitlements()
    private(set) var introEligibleProductIDs: Set<String> = []
    private(set) var purchaseInFlight = false
    private(set) var awaitingApproval = false
    private(set) var offeringsFailed = false
    var lastOutcome: PurchaseOutcome?
    var errorMessage: String?

    private var streamTask: Task<Void, Never>?

    /// Create ONCE, after Purchases.configure (A.2).
    init() {
        streamTask = Task { [weak self] in
            // Emits the cached value first, then every change: renewals, refunds,
            // other-device purchases, offer codes, Ask-to-Buy approvals.
            for await info in Purchases.shared.customerInfoStream {
                self?.apply(info)
            }
        }
        Task { await loadOfferings() }
    }

    // MARK: Offerings

    func loadOfferings() async {
        do {
            let offerings = try await Purchases.shared.offerings()
            offering = offerings.current
            offeringsFailed = offerings.current?.availablePackages.isEmpty ?? true
            if let packages = offering?.availablePackages {
                let eligibility = await Purchases.shared.checkTrialOrIntroDiscountEligibility(packages: packages)
                introEligibleProductIDs = Set(eligibility.compactMap {
                    $0.value.status == .eligible ? $0.key.storeProduct.productIdentifier : nil
                })
            }
        } catch {
            offeringsFailed = true
            errorMessage = "Couldn't load the store. Please try again."
        }
    }

    /// Subscriptions (base plans) vs add-ons, split by package metadata or identifier convention.
    var basePackages: [Package] {
        (offering?.availablePackages ?? []).filter { !$0.identifier.hasPrefix("addon_") }
    }
    var addonPackages: [Package] {
        (offering?.availablePackages ?? []).filter { $0.identifier.hasPrefix("addon_") }
    }

    // MARK: Purchase

    func canPurchase(_ package: Package) -> Bool {
        if addonPackages.contains(package) { return entitlements.hasBasePlan }
        return true
    }

    func purchase(_ package: Package, promotionalOffer: PromotionalOffer? = nil) async {
        guard !purchaseInFlight, canPurchase(package) else { return }
        purchaseInFlight = true
        defer { purchaseInFlight = false }

        do {
            let result: PurchaseResultData
            if let promotionalOffer {
                result = try await Purchases.shared.purchase(package: package, promotionalOffer: promotionalOffer)
            } else {
                result = try await Purchases.shared.purchase(package: package)
            }
            guard !result.userCancelled else { return }   // no alert
            apply(result.customerInfo)
            awaitingApproval = false

            let productID = package.storeProduct.productIdentifier
            let isSubscription = package.storeProduct.productType == .autoRenewableSubscription
            // A downgrade/crossgrade "succeeds" but the old product keeps the entitlement until renewal.
            if isSubscription && !result.customerInfo.activeSubscriptions.contains(productID) {
                lastOutcome = .scheduled(productID: productID)
            } else {
                lastOutcome = .activated(productID: productID)
            }
        } catch {
            handle(error)
        }
    }

    /// Promotional offer — RevenueCat signs it with your uploaded In-App Purchase key.
    func purchase(_ package: Package, offerIdentifier: String) async {
        guard let discount = package.storeProduct.discounts.first(where: { $0.offerIdentifier == offerIdentifier }) else { return }
        do {
            let offer = try await Purchases.shared.promotionalOffer(forProductDiscount: discount,
                                                                    product: package.storeProduct)
            await purchase(package, promotionalOffer: offer)
        } catch {
            handle(error)   // ineligible users land here
        }
    }

    private func handle(_ error: Error) {
        guard let rcError = error as? RevenueCat.ErrorCode else {
            errorMessage = "Purchase failed. Please try again."
            return
        }
        switch rcError {
        case .purchaseCancelledError:
            break
        case .paymentPendingError:
            awaitingApproval = true
            lastOutcome = .pendingApproval
        case .networkError, .offlineConnectionError:
            errorMessage = "You're offline. Check your connection and try again."
        case .receiptAlreadyInUseError:
            errorMessage = "This purchase belongs to another account. Sign in with that account or restore purchases."
        case .ineligibleError:
            errorMessage = "This offer isn't available for your account."
        default:
            errorMessage = "Purchase failed. Please try again."
        }
    }

    // MARK: Restore & refresh

    func restorePurchases() async {
        do {
            let info = try await Purchases.shared.restorePurchases()
            apply(info)
            if info.entitlements.active.isEmpty { errorMessage = "No previous purchases found." }
        } catch {
            errorMessage = "Restore failed. Please try again."
        }
    }

    func refresh() async {
        if let info = try? await Purchases.shared.customerInfo() { apply(info) }
    }

    // MARK: Identity

    func signIn(userID: String) async {
        do {
            let (info, _) = try await Purchases.shared.logIn(userID)
            apply(info)
        } catch {
            errorMessage = "Couldn't load your purchases. Please try again."
        }
    }

    func signOut() async {
        guard !Purchases.shared.isAnonymous else { return }
        if let info = try? await Purchases.shared.logOut() { apply(info) }
    }

    // MARK: Mapping CustomerInfo → app entitlements

    private func apply(_ info: CustomerInfo) {
        if info.entitlements.verification == .failed {
            print("[IAP] Entitlement verification failed")   // report to your monitoring
        }

        var e = AppEntitlements()
        let active = info.entitlements.active
        if active[Entitlement.pro.rawValue] != nil { e.tier = .pro }
        else if active[Entitlement.plus.rawValue] != nil { e.tier = .plus }

        e.activeAddons = Set(ProductCatalog.addonEntitlements.filter { active[$0.rawValue] != nil })

        // Plan card: highest tier, falling back to an inactive one so "Ended on …" / billing retry can show.
        let planInfo = info.entitlements.all[Entitlement.pro.rawValue].flatMap { $0.isActive ? $0 : nil }
            ?? info.entitlements.all[Entitlement.plus.rawValue]
            ?? info.entitlements.all[Entitlement.pro.rawValue]
        e.plan = planInfo.map(PlanStatus.init)

        // Non-renewing passes: RevenueCat doesn't compute expiry — stack it yourself
        // (prefer your server for cross-device consistency).
        var expiry: Date?
        for tx in info.nonSubscriptions.sorted(by: { $0.purchaseDate < $1.purchaseDate }) {
            guard let duration = ProductCatalog.passDurationByProductID[tx.productIdentifier] else { continue }
            let start = max(tx.purchaseDate, expiry ?? tx.purchaseDate)
            expiry = Calendar.current.date(byAdding: duration, to: start)
        }
        e.passExpiry = expiry

        entitlements = e
    }
}
```

Consumable balances come from your server (credited by the `NON_RENEWING_PURCHASE` webhook, A.6) — fetch them after a successful consumable purchase rather than adding locally.

### A.4 Paywall options

```swift
import RevenueCat
import RevenueCatUI
import SwiftUI

// Option 1 — RevenueCat Paywalls: gate a screen behind an entitlement.
struct ProFeatureScreen: View {
    var body: some View {
        ProFeatureContent()
            .presentPaywallIfNeeded(requiredEntitlementIdentifier: Entitlement.pro.rawValue)
    }
}

// Option 2 — explicit presentation + Customer Center.
struct SettingsSubscriptionRow: View {
    @Environment(SubscriptionManager.self) private var subscriptions
    @State private var showPaywall = false
    @State private var showCustomerCenter = false

    var body: some View {
        Group {
            if subscriptions.entitlements.hasBasePlan {
                Button("Manage subscription") { showCustomerCenter = true }
            } else {
                Button("Upgrade") { showPaywall = true }
            }
        }
        .sheet(isPresented: $showPaywall) {
            PaywallView(displayCloseButton: true)
                .onPurchaseCompleted { _ in showPaywall = false }
                .onRestoreCompleted { info in
                    if !info.entitlements.active.isEmpty { showPaywall = false }
                }
        }
        .presentCustomerCenter(isPresented: $showCustomerCenter)
    }
}

// Option 3 — custom paywall driven by the current offering.
struct CustomPaywallView: View {
    @Environment(SubscriptionManager.self) private var subscriptions

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if subscriptions.offeringsFailed {
                    ContentUnavailableView("Store unavailable", systemImage: "wifi.exclamationmark",
                                           description: Text("Please try again."))
                    Button("Retry") { Task { await subscriptions.loadOfferings() } }
                }

                ForEach(subscriptions.basePackages, id: \.identifier) { PackageRow(package: $0) }

                if subscriptions.entitlements.hasBasePlan, !subscriptions.addonPackages.isEmpty {
                    Text("Add-ons").font(.headline)
                    ForEach(subscriptions.addonPackages, id: \.identifier) { PackageRow(package: $0) }
                }

                if subscriptions.awaitingApproval {
                    Text("Waiting for approval…").font(.footnote)
                }

                Text("Subscriptions renew automatically at the price shown unless cancelled at least 24 hours before the end of the current period. Manage or cancel in Settings.")
                    .font(.caption2).foregroundStyle(.secondary)
                HStack {
                    Link("Terms of Use", destination: URL(string: "https://example.com/terms")!)
                    Link("Privacy Policy", destination: URL(string: "https://example.com/privacy")!)
                }.font(.footnote)
                Button("Restore Purchases") { Task { await subscriptions.restorePurchases() } }
                    .font(.footnote)
            }
            .padding()
        }
        .disabled(subscriptions.purchaseInFlight)
    }
}

struct PackageRow: View {
    @Environment(SubscriptionManager.self) private var subscriptions
    let package: Package

    var body: some View {
        let product = package.storeProduct
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(product.localizedTitle).font(.headline)
                if let intro = product.introductoryDiscount,
                   subscriptions.introEligibleProductIDs.contains(product.productIdentifier) {
                    Text("\(intro.localizedPriceString) for \(intro.subscriptionPeriod.value) \(String(describing: intro.subscriptionPeriod.unit)), then \(product.localizedPriceString)")
                        .font(.caption2)
                }
            }
            Spacer()
            Button(product.localizedPriceString) { Task { await subscriptions.purchase(package) } }
                .buttonStyle(.borderedProminent)
                .disabled(!subscriptions.canPurchase(package))
        }
    }
}
```

For Current plan / Upgrade / Downgrade labels and confirmation alerts, port ios-in-app-purchase Appendix A.6, replacing `GroupStatus` with `PlanStatus` and `groupLevel` lookups with `StoreProduct.sk2Product?.subscription?.groupLevel`.

### A.5 Hybrid: pending downgrade from StoreKit 2 (read-only)

```swift
import StoreKit

extension SubscriptionManager {
    /// Product ID that will renew next if it differs from the current one ("You'll move to Plus on …").
    func pendingRenewalProductID(for package: Package) async -> String? {
        guard let groupID = package.storeProduct.subscriptionGroupIdentifier,
              let statuses = try? await Product.SubscriptionInfo.status(for: groupID) else { return nil }
        for status in statuses {
            guard case .verified(let renewal) = status.renewalInfo,
                  case .verified(let tx) = status.transaction,
                  let next = renewal.autoRenewPreference, next != tx.productID else { continue }
            return next
        }
        return nil
    }
}
```

Reading status is safe alongside RevenueCat; never call `finish()` on these transactions.

### A.6 Webhook handler (Node/TypeScript, Express)

```ts
import express from "express";

const app = express();

app.post("/webhooks/revenuecat", express.json(), async (req, res) => {
  // RevenueCat sends exactly the Authorization value configured in the dashboard.
  if (req.get("authorization") !== process.env.RC_WEBHOOK_AUTH) return res.sendStatus(401);

  const event = req.body.event;
  if (event.type === "TEST") return res.sendStatus(200);
  if (await db.processedEvents.has(event.id)) return res.sendStatus(200);   // idempotent

  const isSandbox = event.environment === "SANDBOX";

  if (event.type === "NON_RENEWING_PURCHASE") {
    const credits = CREDITS_BY_PRODUCT[event.product_id];
    if (credits) {
      // Unique constraint on transaction_id makes this safe to replay.
      await db.ledger.creditOnce(event.app_user_id, event.transaction_id, credits, { isSandbox });
    }
  }

  // Robust default: re-fetch and overwrite instead of applying deltas (ordering-safe).
  const userIDs: string[] =
    event.type === "TRANSFER" ? [...event.transferred_from, ...event.transferred_to] : [event.app_user_id];
  for (const userID of userIDs) {
    const r = await fetch(`https://api.revenuecat.com/v1/subscribers/${encodeURIComponent(userID)}`, {
      headers: { Authorization: `Bearer ${process.env.RC_SECRET_KEY}` },
    });
    const { subscriber } = await r.json();
    await db.entitlements.replace(userID, subscriber.entitlements, { isSandbox });
  }

  await db.processedEvents.add(event.id);
  res.sendStatus(200);
});
```

Do the work in a queue if it can take more than a few seconds — RevenueCat retries slow or failed deliveries.

### A.7 Testing the mapping without RevenueCat

`CustomerInfo` is hard to construct in unit tests. Keep the mapping logic testable by extracting it behind a small protocol:

```swift
protocol EntitlementSnapshot {
    var activeEntitlementIDs: Set<String> { get }
}

extension CustomerInfo: EntitlementSnapshot {
    var activeEntitlementIDs: Set<String> { Set(entitlements.active.keys) }
}

func tier(from snapshot: EntitlementSnapshot) -> Tier {
    if snapshot.activeEntitlementIDs.contains(Entitlement.pro.rawValue) { return .pro }
    if snapshot.activeEntitlementIDs.contains(Entitlement.plus.rawValue) { return .plus }
    return .free
}

// Tests
struct FakeSnapshot: EntitlementSnapshot { var activeEntitlementIDs: Set<String> }

import Testing
@Test func proImpliesPro() { #expect(tier(from: FakeSnapshot(activeEntitlementIDs: ["plus", "pro"])) == .pro) }
@Test func addonAloneIsFree() { #expect(tier(from: FakeSnapshot(activeEntitlementIDs: ["cloud_backup"])) == .free) }
```

Cover purchase flows end to end with a StoreKit Configuration file (R-scenarios) and UI tests; RevenueCat's own SDK has no public test double for `Purchases.shared`.
