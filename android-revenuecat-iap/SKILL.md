---
name: android-revenuecat-iap
description: Use when implementing, reviewing, testing, or debugging Android in-app purchases or subscriptions built on RevenueCat (purchases-android / purchases-ui) — SDK setup, Google Play service credentials & RTDN, offerings & packages, subscription options (base plans/offers), entitlements, paywalls, Customer Center, user identity (logIn/logOut, transfers), upgrades/downgrades with replacement modes, webhooks / REST API. For native Play Billing without RevenueCat, use android-in-app-purchase.
---

# Android In-App Purchases with RevenueCat

Guide for building, auditing, and testing Android purchases on top of the **RevenueCat Android SDK** (`com.revenuecat.purchases:purchases`, v8+) and **RevenueCatUI** (`purchases-ui`: Paywalls, Customer Center). RevenueCat wraps Google Play Billing: it verifies purchases with Google, **acknowledges and consumes** them, ingests Real-time Developer Notifications, syncs entitlements across devices and platforms, and provides analytics. **You still own**: Play Console setup, the entitlement design, user identity, the UI, consumable balances, plan-change choices, and Play policy compliance.

Companion skill: **android-in-app-purchase** — use its sections for Play Console product/base plan/offer setup (§3, §8), Play policy (§9), and post-purchase UI on Android (§13). This skill says how to drive them from RevenueCat data. Sibling: **ios-revenuecat-iap** (same structure for iOS).

SDK APIs move between major versions. Before shipping, verify method names against the current RevenueCat Android docs and the SDK version in `build.gradle`.

## 0. Architecture (mental model)

```
+------------------+  RTDN (Cloud Pub/Sub)     +-----------------------------+
|   Google Play    | ------------------------> | RevenueCat backend          |
+------------------+  Play Developer API <---- | - verifies & acknowledges   |
        ^                                      | - CustomerInfo per user     |
        | Play Billing                         | - Offerings / Experiments   |
+------------------------------+  HTTPS        | - Webhooks, REST API, ETL   |
| RevenueCat SDK (on device)   | <-----------> +-----------------------------+
| - Purchases.configure        |                      | webhooks (+ REST re-fetch)
| - awaitOfferings/awaitPurchase|                     v
| - UpdatedCustomerInfoListener|               +---------------------------+
| - logIn / logOut / restore   |               | Your backend (optional)   |
+------------------------------+               | - mirror entitlements     |
        ^                                      | - consumable ledger       |
+-----------------------------------------+    +---------------------------+
| SubscriptionRepository (app singleton,  |
|  StateFlow) — created in Application    |
| - maps entitlement IDs → app features   |
+-----------------------------------------+
        ^
+-----------------------------------------+
| Compose: RevenueCatUI Paywall or custom |
| paywall, status card, Customer Center,  |
| Manage (Play), Restore                  |
+-----------------------------------------+
```

Concepts — get these right first:

| Concept | What it is | Your code uses it for |
|---|---|---|
| **Product** | A Play subscription **base plan** (`pro:monthly`) or one-time product (`lifetime`) | Rarely referenced directly |
| **Subscription option** | A base plan or one of its **offers** (trial/intro) — `SubscriptionOption` | Choosing which offer to buy (default is picked for you) |
| **Entitlement** | A level of access (`pro`, `cloud_backup`) that one or more products unlock | **All feature gating** |
| **Offering** → **Package** | Remotely configured set of products for a paywall (`$rc_monthly`, `$rc_annual`, `$rc_lifetime`, custom) | **What the paywall displays**, changeable without an app release |

## 1. First, establish context

Find out (from the codebase or by asking):

- SDK version (`purchases` 8.x+ uses Play Billing Library 7/8 under the hood — check it meets Play's minimum PBL requirement), `minSdk`, and whether `purchases-ui` is used
- **Who completes purchases**: RevenueCat (default) or your own Play Billing code (`purchasesAreCompletedBy(MY_APP)`) — section 11
- **Account system?** Anonymous-only, or your own user IDs (`logIn`). Decides restore/transfer behaviour
- Product catalog: subscription products + base plans + offers (and offer tags), one-time products (consumable vs non-consumable), add-ons — same classification as android-in-app-purchase §1
- Paywall: **RevenueCat Paywalls** or **custom Compose**
- Backend that needs purchase state? (webhooks + REST API)
- Other Android stores (Amazon Appstore is supported by RevenueCat with a separate API key) or cross-platform (iOS/web) — same App User IDs everywhere
- Migrating from a native Play Billing implementation? (section 10)

Write the **entitlement map** before coding:

```
Play product : base plan → RevenueCat entitlement(s) → app features
plus:monthly, plus:yearly → plus           → ad-free, sync
pro:monthly,  pro:yearly  → plus, pro      → + offline, 4K export   (attach higher tiers to lower entitlements too)
lifetime                  → plus, pro      → same as Pro
addon_backup:monthly      → cloud_backup   → backup (requires plus)
credits_100               → (none)         → +100 credits (ledger)
```

Code checks **entitlement IDs**, never product IDs. Attaching higher tiers to every lower entitlement keeps checks simple (`isActive("plus")` is true for Pro users too).

## 2. RevenueCat dashboard & Play Console configuration

### 2.1 Play Console (unchanged from native)
- [ ] Merchant/payments profile; app uploaded to a testing track; products, **base plans and offers active** with intended eligibility and tags (android-in-app-purchase §3, §8)
- [ ] Grace period and account hold set deliberately (RevenueCat keeps entitlements active during grace)
- [ ] License testers added and opted in

### 2.2 RevenueCat project
- [ ] Play Store app added with the correct **package name**
- [ ] **Google Play service credentials** uploaded: a Google Cloud service account JSON with the Play Console permissions RevenueCat requires (view app info, view financial data, manage orders and subscriptions). New credentials can take up to ~36 hours to become valid — plan for it
- [ ] **Real-time Developer Notifications** connected: create/attach the Pub/Sub topic from the RevenueCat dashboard and set it in Play Console → Monetization setup; send a test notification
- [ ] Products imported (subscriptions appear as `productId:basePlanId`); **entitlements** created and products attached per the map
- [ ] **Offerings** with packages; one marked **Current**. Placements / Targeting / Experiments only if you'll use them
- [ ] **Restore behavior** chosen deliberately: *Transfer to new App User ID* (default), *Transfer if no active subscriptions*, or *Keep with original App User ID* — section 5.3
- [ ] Consumables: make sure one-time products that must **not** be consumed are marked non-consumable in RevenueCat (otherwise the SDK consumes one-time purchases)
- [ ] Webhook URL + **Authorization header value** configured if you have a backend
- [ ] Public **Google API key** (`goog_…`) in the app; **secret keys** (`sk_…`) only on the server

## 3. SDK setup checklist

- [ ] `Purchases.configure(PurchasesConfiguration.Builder(context, apiKey)...)` called **exactly once** in `Application.onCreate()`, **before** anything touches `Purchases.sharedInstance`
- [ ] `Purchases.logLevel = LogLevel.DEBUG` only in debug builds
- [ ] `appUserID(...)` passed at configure time if the user is already signed in (avoids creating an anonymous user then aliasing)
- [ ] Trusted Entitlements (`entitlementVerificationMode(INFORMATIONAL)`) on if available for your SDK version; log/alert on failed verification
- [ ] One long-lived **`UpdatedCustomerInfoListener`** (or a flow collected app-wide) — this is how renewals, expirations, refunds, purchases from other devices, promo codes, and completed pending purchases reach the UI
- [ ] No other code in the app creates its own `BillingClient` that acknowledges/consumes while RevenueCat completes purchases (double handling)
- [ ] ProGuard/R8: rely on the SDK's consumer rules; don't strip `com.revenuecat.purchases` classes

## 4. Offerings, packages & the paywall

### 4.1 Loading
- [ ] `Purchases.sharedInstance.awaitOfferings()` → `offerings.current` (or `offerings.getCurrentOfferingForPlacement(...)` / `offerings[identifier]`)
- [ ] Handle `current == null` and empty `availablePackages`: no current offering, products/base plans inactive, app not on a track, credentials not valid yet, `BILLING_UNAVAILABLE`. Show a retry state — never a blank screen or a crash
- [ ] Package lookup by type (`offering.monthly`, `.annual`, `.lifetime`) or identifier — **never by list index**
- [ ] Prices from `package.product.price.formatted` (and `pricePerMonth` etc. for comparisons) — never hardcoded
- [ ] Offering-specific copy in **offering metadata**, read with defaults

### 4.2 Subscription options (base plans & offers)
- [ ] By default the SDK buys `product.defaultOption`: the eligible offer with the longest free trial / cheapest intro, else the base plan. Offers tagged **`rc-ignore-offer`** in Play Console are never auto-selected — use that tag for developer-determined/retention offers
- [ ] To buy a specific offer, pick from `product.subscriptionOptions` (e.g. by tag via `withTag(...)`, `freeTrial`, `introOffer`, `basePlan`) and pass it in `PurchaseParams.Builder(activity, subscriptionOption)`
- [ ] Trial/intro text from the chosen option's `pricingPhases` (`freePhase`, `introPhase`, `fullPricePhase`); Play only returns offers the user is eligible for
- [ ] Prepaid base plans: `purchasingData` / pricing phases show no renewal — say "one-time payment for <period>"

### 4.3 RevenueCat Paywalls (RevenueCatUI) vs custom

| | RevenueCatUI | Custom Compose paywall |
|---|---|---|
| Change design without release | ✅ | Only via offering metadata |
| A/B tests (Experiments) | ✅ | ✅ (offering-level) |
| Eligibility, prices, localization | Handled | You implement |
| Upgrade / Downgrade labels, add-on rules, replacement modes | Limited | ✅ full control |

- [ ] RevenueCatUI (Compose): `PaywallDialog(PaywallDialogOptions.Builder().setRequiredEntitlementIdentifier("pro")...)` for gating, or `Paywall(PaywallOptions.Builder(dismissRequest).build())`; Views: `PaywallActivityLauncher`. Handle purchase-completed / restore-completed / error callbacks via `PaywallListener`
- [ ] Template has **Terms** and **Privacy** links, auto-renew and cancellation disclosure (android-in-app-purchase §9)
- [ ] Close button visible
- [ ] Custom paywall: Current plan / Upgrade / Downgrade / Switch labels and alerts per ios-in-app-purchase §13.2–13.3 (Android notes in android-in-app-purchase §13)

## 5. User identity

### 5.1 Rules
- [ ] Anonymous users get `$RCAnonymousID:…` automatically — fine for apps without accounts
- [ ] With accounts: `awaitLogIn(yourStableUserId)` right after sign-in, **before** showing the paywall; `awaitLogOut()` on sign-out
- [ ] App User ID is **stable, non-guessable, non-PII** (your DB UUID). Never email, Android ID, advertising ID, `""`, `"null"`, or a constant
- [ ] Don't log out anonymous users (`Purchases.sharedInstance.isAnonymous`)
- [ ] Same ID on every platform (iOS, web, Amazon) so entitlements follow the user
- [ ] Subscriber attributes (`setEmail`, `setDisplayName`, `setAttributes`) only with consent and per the privacy policy / Data safety form

### 5.2 Obfuscated account ID
RevenueCat sets Play's `obfuscatedAccountId` from the App User ID (hashed) for you. Don't set it yourself through a separate BillingClient.

### 5.3 Restore & transfers
- [ ] Restore button → `awaitRestore()`; if no entitlement becomes active, say "No previous purchases found" (not an error)
- [ ] Restore under a **different** App User ID applies the project's restore behavior (default **transfers**, webhook `TRANSFER`). Decide whether that fits your account model (shared Google accounts, family devices)
- [ ] Purchases follow the Google account; the SDK also syncs on app start, so Restore is mainly for re-linking
- [ ] `syncPurchases()` is for **silent** migration (section 10), not for the Restore button

## 6. Purchasing

- [ ] `awaitPurchase(PurchaseParams.Builder(activity, package).build())` → `PurchaseResult(storeTransaction, customerInfo)`; apply `customerInfo` immediately
- [ ] User cancelled → `PurchasesTransactionException` with `userCancelled == true` → **no alert**
- [ ] `PurchasesErrorCode.PaymentPendingError` (slow card, pending) → "Waiting for payment"; the result arrives later through the customer-info listener
- [ ] `NetworkError` / `StoreProblemError` → retryable message. `ProductAlreadyPurchasedError`, `ReceiptAlreadyInUseError` (purchase belongs to another App User ID), `IneligibleError`, `PurchaseNotAllowedError` → short, specific messages
- [ ] Purchase button disabled while in flight
- [ ] Gate on **entitlements** after purchase (`customerInfo.entitlements[id]?.isActive == true`), not on "no exception"
- [ ] Add-ons purchasable only while the base entitlement is active
- [ ] RevenueCat **acknowledges/consumes** — never call `acknowledgePurchase`/`consumePurchase` yourself (unless section 11)

### 6.1 Upgrades, downgrades & switches
Play doesn't replace subscriptions automatically (android-in-app-purchase §3.2). With RevenueCat:
- [ ] Pass the current subscription with `PurchaseParams.Builder(...).oldProductId("<current productId>")` and a `googleReplacementMode(GoogleReplacementMode.X)`
- [ ] Upgrade → `CHARGE_PRORATED_PRICE` (or `WITH_TIME_PRORATION`); downgrade → `DEFERRED`; same tier, longer period → `CHARGE_FULL_PRICE` or `DEFERRED`
- [ ] Never buy a second tier without `oldProductId` → the user is billed for two subscriptions
- [ ] After a `DEFERRED` change, the entitlement still shows the old product until renewal → say "Change scheduled", not "Purchase successful"
- [ ] RevenueCat handles the new purchase token and the linked old token on its backend

### 6.2 Offers
| Offer | How with RevenueCat |
|---|---|
| Free trial / intro (Play-eligible) | Default option, or pick `subscriptionOptions.freeTrial` / `.introOffer` |
| Developer-determined (lapsed, retention) | Tag in Play Console (+ `rc-ignore-offer` so it's not auto-selected); select by tag and buy that option |
| Promo codes | Redeemed in Play; the SDK picks it up on next sync → customer-info listener |
| Payment issues (grace / hold) | RevenueCat can show Play's in-app messages automatically (`showInAppMessagesAutomatically`, default on in recent SDKs) — verify for your version |

## 7. Reading state for the UI

### 7.1 Field mapping (`EntitlementInfo`)

| UI need (android-in-app-purchase §13) | RevenueCat field |
|---|---|
| Has access | `isActive` (true during grace period) |
| Current plan | `productIdentifier` (+ `productPlanIdentifier` = base plan on Play) |
| Renews / ends on | `expirationDate` + `willRenew` — `null` expiration = lifetime |
| Cancelled | `willRenew == false` / `unsubscribeDetectedAt != null` |
| Billing problem | `billingIssueDetectedAt != null` (grace if still active, hold if not) |
| Trial / intro running | `periodType == PeriodType.TRIAL` / `INTRO` |
| Sandbox / test | `isSandbox` |
| Store | `store` (PLAY_STORE, APP_STORE, AMAZON, STRIPE…) — manage on the right store |

`customerInfo.managementURL` gives the correct manage link for the store the user paid on (Play subscriptions page on Android). RevenueCatUI's **Customer Center** gives a ready-made manage/cancel/feedback screen.

### 7.2 Freshness
- [ ] The customer-info listener drives the UI; read `awaitCustomerInfo()` on start
- [ ] Cached `CustomerInfo` is fine for gating; force a fetch (`CacheFetchPolicy.FETCH_CURRENT`) only where it matters
- [ ] Offline: cached / offline entitlements keep paying users in; don't lock them out because a request failed
- [ ] Paused / on hold: entitlement inactive — show "paused / payment problem" using `billingIssueDetectedAt` and your messaging, not "free"

### 7.3 Pending downgrades
`CustomerInfo` shows the active product, not the deferred one. Remember the deferred target locally at purchase time to show "You'll move to Plus on <date>" (and clear it when the entitlement's product changes), or rely on Customer Center / your backend (webhook `PRODUCT_CHANGE`).

## 8. Consumables & non-renewing purchases

- [ ] The SDK **consumes** consumable one-time purchases automatically — Play won't redeliver them. Credit balances **server-side from the `NON_RENEWING_PURCHASE` webhook**, idempotent by `transaction_id`; client-side crediting only for offline-only apps (and idempotent)
- [ ] Alternatively evaluate RevenueCat **Virtual Currencies** if available on your plan/SDK
- [ ] Non-consumable one-time products (lifetime) attached to an entitlement — mark them non-consumable in RevenueCat
- [ ] Prepaid base plans: entitlement active until expiry; show "top up" instead of "renews"
- [ ] Purchase history: `customerInfo.nonSubscriptionTransactions`

## 9. Backend: webhooks & REST API

Same as ios-revenuecat-iap §9 — the events are store-agnostic:
- [ ] Verify the `Authorization` header; idempotent by `event.id`; respond `200` fast; re-fetch the subscriber (`GET /v1/subscribers/{app_user_id}` or v2 API) and overwrite
- [ ] Handle `environment` (`SANDBOX` = license testers / test purchases)
- [ ] `store == "PLAY_STORE"` events: `PRODUCT_CHANGE` for upgrades/downgrades (deferred downgrades apply at renewal — re-fetch), `BILLING_ISSUE` for grace/hold, `SUBSCRIPTION_PAUSED` for Play pauses, `EXPIRATION` with `expiration_reason`, `CANCELLATION`, `UNCANCELLATION`, `RENEWAL`, `INITIAL_PURCHASE`, `NON_RENEWING_PURCHASE`, `TRANSFER`, `SUBSCRIPTION_EXTENDED`, `TEMPORARY_ENTITLEMENT_GRANT`, `REFUND_REVERSED`, `TEST`
- [ ] Secret keys never in the app

## 10. Migrating an existing native Play Billing app

- [ ] Remove your own `BillingClient` acknowledge/consume logic once RevenueCat completes purchases (or use section 11 during a transition)
- [ ] Existing subscribers: `syncPurchases()` **once** per install after upgrading (guarded by a flag), and/or import historical purchase tokens via RevenueCat's import tools
- [ ] Keep product IDs and base plan IDs unchanged; recreate the entitlement map in the dashboard
- [ ] Switch RTDN to RevenueCat's Pub/Sub topic (or keep yours and forward) — one source of truth
- [ ] Your backend: move from Play Developer API polling/RTDN handling to RevenueCat webhooks

## 11. `purchasesAreCompletedBy(MY_APP)` (your Play Billing code completes purchases)

Use only if you must keep your own purchase flow.
- [ ] Configure with `.purchasesAreCompletedBy(PurchasesAreCompletedBy.MY_APP)`
- [ ] **You** acknowledge/consume within 3 days and handle `PurchasesUpdatedListener` and re-queries (all of android-in-app-purchase §6 applies)
- [ ] Call `Purchases.sharedInstance.syncPurchases()` after each purchase (or the per-purchase sync API your SDK version offers) so RevenueCat sees it
- [ ] Gate from `CustomerInfo` *or* your own state — not both for the same feature

## 12. Testing plan

1. **License testers + internal testing track** — test cards (approve / decline / slow approve for pending); accelerated renewals (~5 min monthly); dashboard **sandbox data** toggle shows these purchases
2. **Play Billing Lab** — change country, reset trial eligibility
3. **RevenueCat Test Store** (if available for your project/SDK) — paywall/entitlement flows without Play products
4. **Staging backend** receiving webhooks

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| R1 | Fresh install, no account | Anonymous ID; offerings load; prices shown |
| R2 | `offerings.current == null` (no current offering / inactive base plans / new credentials) | Retry state, no crash; error logged |
| R3 | Subscribe with eligible free trial | Trial shown; entitlement active; `periodType == TRIAL` |
| R4 | Trial already used | Default option is the base plan; no trial text |
| R5 | Cancel Play sheet | No alert |
| R6 | Slow test card (pending) | "Waiting for payment"; listener unlocks when complete |
| R7 | Upgrade Plus → Pro with `oldProductId` | One subscription; Pro immediately |
| R8 | Downgrade Pro → Plus (DEFERRED) | Still Pro; "Change scheduled"; Plus at renewal |
| R9 | Tier change **without** `oldProductId` (bug check) | Not reachable from the UI |
| R10 | Renewal / expiry (accelerated) | UI updates while app open |
| R11 | Grace period / account hold | Access kept / removed; fix-payment messaging |
| R12 | Cancel in Play | `willRenew == false`; "Ends on …" |
| R13 | Refund from Play Console | Entitlement removed after RTDN |
| R14 | Anonymous purchase, then `logIn` | Purchase aliased; entitlement kept |
| R15 | Restore on a **second** account (same Google account) | Per restore-behavior setting; `TRANSFER` handled |
| R16 | `logOut` | Back to anonymous; premium hidden |
| R17 | Retention offer (tagged, `rc-ignore-offer`) | Only shown where your logic offers it; correct price |
| R18 | Promo code redeemed in Play Store | Entitlement active on next app start |
| R19 | Consumable | Webhook credits once; replay doesn't double-credit |
| R20 | Airplane-mode start as subscriber | Cached entitlements keep access |
| R21 | Change Current offering in dashboard | New packages after relaunch, no release |
| R22 | Post-purchase UI states | Run ios-in-app-purchase §13.6 (U1–U16) with Android test tools |

## 13. Common bugs to look for in review

- `Purchases.configure` missing, called twice, or called from an Activity instead of `Application.onCreate`
- Gating on `activeSubscriptions.contains("pro:monthly")` or product IDs instead of entitlements
- No customer-info listener; entitlements read once at launch → stale after renewals, refunds, pending completions
- Packages picked by list index; no handling for `offerings.current == null`
- Tier change without `oldProductId` / replacement mode → double billing
- "Purchase successful" after a `DEFERRED` downgrade
- `subscriptionOptions` ignored when a specific offer is intended, or a retention offer auto-selected because it wasn't tagged `rc-ignore-offer`
- `logIn` with email / Android ID / constants; `logIn` after the purchase instead of before; `logOut` on anonymous users
- Restore button calling `syncPurchases()`, or sync on every launch
- Error dialog on user cancellation; pending purchases shown as failures
- Secret API key in the app; iOS key (`appl_…`) used on Android
- Service credentials missing/invalid or RTDN not connected → purchases not validated / late cancellations and refunds
- Lifetime product consumed because it wasn't marked non-consumable
- App code acknowledging/consuming alongside RevenueCat
- Consumables credited client-side without idempotency
- Webhook handler without auth, not idempotent, or switching tier immediately on a deferred `PRODUCT_CHANGE`
- Debug log level in release builds
- Paywall template without Terms/Privacy links or renewal/cancel disclosure

## 14. Output format

When asked to **review/audit**:
1. **Summary** — SDK version (and underlying PBL), who completes purchases, identity model, paywall type, backend yes/no, entitlement map
2. **Checklist results** — sections 2–11 with ✅ / ❌ / ⚠️ and file:line (or "dashboard" / "Play Console" for configuration items you can't see — list them for the user to confirm)
3. **Critical issues first** (lost access for paying users, double billing, wrong user merges, double credits, secret key exposure, policy risks) with fix code
4. **Testing gaps** — scenarios R1–R22 not covered

When asked to **implement**: start from the Appendix, adapt `Entitlement` / catalog to the user's map, remove unused parts, then list dashboard, Play Console, backend, and policy steps the user must do by hand.

---

## Appendix — Reference implementation (RevenueCat Android SDK 8.x, Kotlin coroutines, Compose)

Verify API names against the SDK version you install.

### A.1 Catalog — entitlement IDs, not product IDs

```kotlin
enum class Entitlement(val id: String) {
    PLUS("plus"),                 // attached to Plus, Pro and Lifetime products
    PRO("pro"),                   // attached to Pro and Lifetime products
    CLOUD_BACKUP("cloud_backup"), // add-on, its own subscription product
}

enum class Tier { FREE, PLUS, PRO }

object Catalog {
    val addonEntitlements = setOf(Entitlement.CLOUD_BACKUP)
    /** Packages whose identifier starts with this are add-ons (set in the offering). */
    const val ADDON_PACKAGE_PREFIX = "addon_"
    /** Play offer tag for retention offers you select explicitly. */
    const val TAG_RETENTION = "retention"
}

object RevenueCatConfig {
    const val GOOGLE_API_KEY = "goog_XXXXXXXXXXXXXXXX"   // public SDK key — safe in the app
}
```

### A.2 Application — configure once

```kotlin
class App : Application() {
    val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
    lateinit var subscriptions: SubscriptionRepository

    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) Purchases.logLevel = LogLevel.DEBUG
        Purchases.configure(
            PurchasesConfiguration.Builder(this, RevenueCatConfig.GOOGLE_API_KEY)
                .appUserID(AuthStore.currentUserId(this))     // null → anonymous
                .build()
        )
        subscriptions = SubscriptionRepository(appScope)     // after configure
    }
}
```

### A.3 SubscriptionRepository

```kotlin
import android.app.Activity
import com.revenuecat.purchases.*
import com.revenuecat.purchases.models.GoogleReplacementMode
import com.revenuecat.purchases.models.StoreProduct
import com.revenuecat.purchases.models.SubscriptionOption
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import java.util.Date

data class PlanStatus(
    val entitlementId: String,
    val productId: String,
    val basePlanId: String?,
    val isActive: Boolean,
    val expirationDate: Date?,     // null → lifetime
    val willRenew: Boolean,
    val isTrialOrIntro: Boolean,
    val billingIssueDetectedAt: Date?,
    val isSandbox: Boolean,
) {
    val isLifetime get() = isActive && expirationDate == null
    val inGracePeriod get() = isActive && billingIssueDetectedAt != null
    val onHoldOrRetry get() = !isActive && billingIssueDetectedAt != null

    companion object {
        fun from(info: EntitlementInfo) = PlanStatus(
            entitlementId = info.identifier,
            productId = info.productIdentifier,
            basePlanId = info.productPlanIdentifier,
            isActive = info.isActive,
            expirationDate = info.expirationDate,
            willRenew = info.willRenew,
            isTrialOrIntro = info.periodType == PeriodType.TRIAL || info.periodType == PeriodType.INTRO,
            billingIssueDetectedAt = info.billingIssueDetectedAt,
            isSandbox = info.isSandbox,
        )
    }
}

data class AppEntitlements(
    val tier: Tier = Tier.FREE,
    val activeAddons: Set<Entitlement> = emptySet(),
    val plan: PlanStatus? = null,
) {
    val hasBasePlan get() = tier > Tier.FREE
    val usableAddons get() = if (hasBasePlan) activeAddons else emptySet()
}

sealed interface PurchaseOutcome {
    data class Activated(val productId: String) : PurchaseOutcome
    data class Scheduled(val productId: String) : PurchaseOutcome   // DEFERRED change
    data object Pending : PurchaseOutcome
}

data class SubscriptionState(
    val offering: Offering? = null,
    val offeringsFailed: Boolean = false,
    val entitlements: AppEntitlements = AppEntitlements(),
    val purchaseInFlight: Boolean = false,
    val awaitingPayment: Boolean = false,
    val scheduledProductId: String? = null,      // deferred downgrade target (local hint)
    val lastOutcome: PurchaseOutcome? = null,
    val errorMessage: String? = null,
)

class SubscriptionRepository(private val scope: CoroutineScope) {
    private val _state = MutableStateFlow(SubscriptionState())
    val state: StateFlow<SubscriptionState> = _state.asStateFlow()

    init {
        // Renewals, refunds, other-device purchases, promo codes, completed pending purchases.
        Purchases.sharedInstance.updatedCustomerInfoListener = UpdatedCustomerInfoListener { apply(it) }
        scope.launch {
            runCatching { apply(Purchases.sharedInstance.awaitCustomerInfo()) }
            loadOfferings()
        }
    }

    // MARK: Offerings

    suspend fun loadOfferings() {
        try {
            val offering = Purchases.sharedInstance.awaitOfferings().current
            _state.update { it.copy(offering = offering, offeringsFailed = offering?.availablePackages.isNullOrEmpty()) }
        } catch (e: PurchasesException) {
            _state.update { it.copy(offeringsFailed = true, errorMessage = "Couldn't load the store. Please try again.") }
        }
    }

    val basePackages get() = _state.value.offering?.availablePackages.orEmpty()
        .filterNot { it.identifier.startsWith(Catalog.ADDON_PACKAGE_PREFIX) }
    val addonPackages get() = _state.value.offering?.availablePackages.orEmpty()
        .filter { it.identifier.startsWith(Catalog.ADDON_PACKAGE_PREFIX) }

    fun canPurchase(pkg: Package) = pkg !in addonPackages || _state.value.entitlements.hasBasePlan

    /** Retention offer selected by tag (tag it rc-ignore-offer too, so it's never the default). */
    fun retentionOption(product: StoreProduct): SubscriptionOption? =
        product.subscriptionOptions?.withTag(Catalog.TAG_RETENTION)?.firstOrNull()

    // MARK: Purchase

    /**
     * @param replace current subscription product ID for an upgrade/downgrade (never omit it for a tier change)
     * @param mode replacement mode (android-in-app-purchase §3.2)
     */
    fun purchase(
        activity: Activity,
        pkg: Package,
        option: SubscriptionOption? = null,
        replace: String? = null,
        mode: GoogleReplacementMode = GoogleReplacementMode.CHARGE_PRORATED_PRICE,
    ) {
        if (_state.value.purchaseInFlight || !canPurchase(pkg)) return
        _state.update { it.copy(purchaseInFlight = true, errorMessage = null) }

        scope.launch {
            try {
                val builder = if (option != null) PurchaseParams.Builder(activity, option)
                              else PurchaseParams.Builder(activity, pkg)
                if (replace != null) builder.oldProductId(replace).googleReplacementMode(mode)

                val result = Purchases.sharedInstance.awaitPurchase(builder.build())
                apply(result.customerInfo)

                val productId = pkg.product.id.substringBefore(":")
                val scheduled = replace != null && mode == GoogleReplacementMode.DEFERRED
                _state.update {
                    it.copy(
                        awaitingPayment = false,
                        scheduledProductId = if (scheduled) productId else it.scheduledProductId,
                        lastOutcome = if (scheduled) PurchaseOutcome.Scheduled(productId)
                                      else PurchaseOutcome.Activated(productId),
                    )
                }
            } catch (e: PurchasesTransactionException) {
                if (!e.userCancelled) handle(e.error)              // no alert on cancel
            } catch (e: PurchasesException) {
                handle(e.error)
            } finally {
                _state.update { it.copy(purchaseInFlight = false) }
            }
        }
    }

    private fun handle(error: PurchasesError) {
        val message = when (error.code) {
            PurchasesErrorCode.PaymentPendingError -> {
                _state.update { it.copy(awaitingPayment = true, lastOutcome = PurchaseOutcome.Pending) }
                return
            }
            PurchasesErrorCode.NetworkError -> "You're offline. Check your connection and try again."
            PurchasesErrorCode.ReceiptAlreadyInUseError ->
                "This purchase belongs to another account. Sign in with that account or restore purchases."
            PurchasesErrorCode.ProductAlreadyPurchasedError -> "You already own this."
            PurchasesErrorCode.PurchaseNotAllowedError -> "Purchases aren't allowed on this device or account."
            else -> "Purchase failed. Please try again."
        }
        _state.update { it.copy(errorMessage = message) }
    }

    // MARK: Restore & identity

    fun restore() = scope.launch {
        try {
            val info = Purchases.sharedInstance.awaitRestore()
            apply(info)
            if (info.entitlements.active.isEmpty()) _state.update { it.copy(errorMessage = "No previous purchases found.") }
        } catch (e: PurchasesException) {
            _state.update { it.copy(errorMessage = "Restore failed. Please try again.") }
        }
    }

    fun signIn(userId: String) = scope.launch {
        runCatching { apply(Purchases.sharedInstance.awaitLogIn(userId).customerInfo) }
            .onFailure { _state.update { s -> s.copy(errorMessage = "Couldn't load your purchases.") } }
    }

    fun signOut() = scope.launch {
        if (Purchases.sharedInstance.isAnonymous) return@launch
        runCatching { apply(Purchases.sharedInstance.awaitLogOut()) }
    }

    fun clearOutcome() = _state.update { it.copy(lastOutcome = null, errorMessage = null) }

    // MARK: Mapping CustomerInfo → app entitlements

    private fun apply(info: CustomerInfo) {
        val active = info.entitlements.active
        val tier = when {
            Entitlement.PRO.id in active -> Tier.PRO
            Entitlement.PLUS.id in active -> Tier.PLUS
            else -> Tier.FREE
        }
        val addons = Catalog.addonEntitlements.filter { it.id in active }.toSet()

        // Plan card: highest tier, falling back to an inactive one so "Ended" / payment-problem states can show.
        val all = info.entitlements.all
        val planInfo = all[Entitlement.PRO.id]?.takeIf { it.isActive }
            ?: all[Entitlement.PLUS.id]
            ?: all[Entitlement.PRO.id]
        val plan = planInfo?.let(PlanStatus::from)

        _state.update {
            // Clear the deferred-downgrade hint once the plan has actually switched.
            val scheduled = it.scheduledProductId?.takeIf { target -> plan?.productId != target }
            it.copy(entitlements = AppEntitlements(tier, addons, plan), scheduledProductId = scheduled)
        }
    }
}
```

Consumable balances come from your server (credited by the `NON_RENEWING_PURCHASE` webhook) — fetch them after a successful consumable purchase rather than adding locally.

### A.4 Paywall options (Compose)

```kotlin
// Option 1 — RevenueCat Paywall gating a screen behind an entitlement.
@Composable
fun ProFeatureScreen() {
    PaywallDialog(
        PaywallDialogOptions.Builder()
            .setRequiredEntitlementIdentifier(Entitlement.PRO.id)
            .setListener(object : PaywallListener {
                override fun onPurchaseCompleted(customerInfo: CustomerInfo, storeTransaction: StoreTransaction) {}
                override fun onRestoreCompleted(customerInfo: CustomerInfo) {}
            })
            .build()
    )
    ProFeatureContent()
}

// Option 2 — Customer Center from settings (manage / cancel / feedback). Verify the composable's
// parameters for your purchases-ui version.
@Composable
fun ManageSubscriptionScreen(onDismiss: () -> Unit) {
    CustomerCenter(modifier = Modifier.fillMaxSize(), onDismiss = onDismiss)
}

// Option 3 — custom paywall driven by the current offering.
@Composable
fun CustomPaywall(repo: SubscriptionRepository, activity: Activity) {
    val state by repo.state.collectAsStateWithLifecycle()
    val uriHandler = LocalUriHandler.current
    val scope = rememberCoroutineScope()
    val current = state.entitlements.plan?.takeIf { it.isActive }?.productId

    Column(Modifier.verticalScroll(rememberScrollState()).padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        if (state.offeringsFailed) {
            Text("Store unavailable. Please try again.")
            Button(onClick = { scope.launch { repo.loadOfferings() } }) { Text("Retry") }
        }
        state.scheduledProductId?.let { Text("Change scheduled — $it starts at your next renewal.") }

        repo.basePackages.forEach { pkg ->
            val product = pkg.product
            val option = product.defaultOption
            val fullPrice = option?.fullPricePhase?.price?.formatted ?: product.price.formatted
            Row(Modifier.fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
                Column(Modifier.weight(1f)) {
                    Text(product.title, style = MaterialTheme.typography.titleMedium)
                    Text("$fullPrice / ${product.period?.iso8601 ?: "one-time"}")
                    option?.freePhase?.let { Text("Free for ${it.billingPeriod.iso8601}, then $fullPrice",
                                                  style = MaterialTheme.typography.bodySmall) }
                }
                val productId = product.id.substringBefore(":")
                val isCurrent = productId == current
                Button(
                    enabled = !isCurrent && !state.purchaseInFlight,
                    onClick = {
                        // For a downgrade: show the confirmation (ios-in-app-purchase §13.3) and use DEFERRED.
                        repo.purchase(activity, pkg, replace = current?.takeIf { it != productId })
                    },
                ) { Text(if (isCurrent) "Current plan" else if (current != null) "Change" else "Subscribe") }
            }
        }

        if (state.entitlements.hasBasePlan) repo.addonPackages.forEach { pkg ->
            Button(onClick = { repo.purchase(activity, pkg) }, enabled = !state.purchaseInFlight) {
                Text("Add ${pkg.product.title} — ${pkg.product.price.formatted}")
            }
        }

        if (state.awaitingPayment) Text("Waiting for payment…", style = MaterialTheme.typography.bodySmall)
        Text("Subscriptions renew automatically until cancelled. Cancel anytime in Google Play → Payments & subscriptions.",
             style = MaterialTheme.typography.bodySmall)
        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            TextButton(onClick = { uriHandler.openUri("https://example.com/terms") }) { Text("Terms") }
            TextButton(onClick = { uriHandler.openUri("https://example.com/privacy") }) { Text("Privacy") }
            TextButton(onClick = { repo.restore() }) { Text("Restore") }
        }
    }
}
```

Use the tier ranking from android-in-app-purchase A.4 (`PlanChanges.classify` / `replacementMode`) to label Upgrade / Downgrade / Switch and pick `GoogleReplacementMode` instead of the single "Change" button above.

### A.5 Testing the mapping without RevenueCat

`CustomerInfo` is hard to construct in unit tests — extract the mapping behind a small interface:

```kotlin
interface EntitlementSnapshot { val activeEntitlementIds: Set<String> }

fun CustomerInfo.snapshot() = object : EntitlementSnapshot {
    override val activeEntitlementIds = entitlements.active.keys
}

fun tierOf(s: EntitlementSnapshot) = when {
    Entitlement.PRO.id in s.activeEntitlementIds -> Tier.PRO
    Entitlement.PLUS.id in s.activeEntitlementIds -> Tier.PLUS
    else -> Tier.FREE
}

class TierTest {
    private fun snap(vararg ids: String) = object : EntitlementSnapshot { override val activeEntitlementIds = ids.toSet() }
    @Test fun proImpliesPro() = assertEquals(Tier.PRO, tierOf(snap("plus", "pro")))
    @Test fun addonAloneIsFree() = assertEquals(Tier.FREE, tierOf(snap("cloud_backup")))
}
```

Cover purchase flows end to end with license testers on an internal testing track (R-scenarios); there is no public test double for `Purchases.sharedInstance`.
