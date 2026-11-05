---
name: react-native-revenuecat-iap
description: Use when implementing, reviewing, testing, or debugging in-app purchases or subscriptions built on RevenueCat in a React Native or Expo app (react-native-purchases, react-native-purchases-ui) — bare RN and Expo dev builds, one RevenueCat project for App Store + Google Play (+ web), platform API keys, shared entitlements and offerings, paywalls, Customer Center, identity across platforms, Android plan changes, iOS offers, webhooks, migration from react-native-iap / expo-iap, testing. For native iOS/Android apps use ios-revenuecat-iap / android-revenuecat-iap; for React Native without RevenueCat use react-native-in-app-purchase; for Flutter use flutter-revenuecat-iap.
---

# React Native & Expo In-App Purchases with RevenueCat

Guide for building, auditing, and testing purchases in **React Native / Expo** apps (`react-native-purchases`, `react-native-purchases-ui`) on RevenueCat. The SDK wraps the native RevenueCat iOS and Android SDKs, so RevenueCat handles store verification, finishing/acknowledging, notifications, and cross-device entitlement sync on both stores. **You still own**: both store consoles, the shared entitlement design, user identity, platform differences in the UI (plan changes, offers, manage links), consumable balances, and store-policy compliance.

Platform skills hold the store- and SDK-specific detail — this skill links to them instead of repeating it:
- **ios-revenuecat-iap** — dashboard setup for the App Store app, iOS offers, field mapping, webhooks (§9 event table)
- **android-revenuecat-iap** — Play service credentials, RTDN, subscription options/offer tags, Google replacement modes
- **ios-in-app-purchase** / **android-in-app-purchase** — store rules, App Review / Play policy, post-purchase UI
- **react-native-in-app-purchase** — React Native / Expo purchases without RevenueCat (`react-native-iap`, `expo-iap`)
- **flutter-revenuecat-iap** — the same RevenueCat setup for Flutter

Hybrid SDK APIs track the native SDKs but rename things between majors (e.g. purchase methods returning `CustomerInfo` vs a result object, proration vs replacement mode naming). Verify every call against the version in `package.json` and its migration guide.

## 0. Architecture (mental model)

```
+-------------------------------------------------------------+
| Shared UI (React components)                                |
| RevenueCatUI Paywall or custom paywall, status card,        |
| Customer Center, Restore                                    |
+-------------------------------------------------------------+
            ^ state (Zustand/Redux/context)
+-------------------------------------------------------------+
| SubscriptionService (singleton, configured at app start)    |
| - Purchases.configure(platform API key, appUserID)          |
| - customer-info listener → entitlements → app tiers         |
| - offerings, purchase, restore, logIn/logOut                |
| - platform branches: Android plan change, iOS promo offers, |
|   manage location by store                                  |
+-------------------------------------------------------------+
     | react-native-purchases (native module bridge)
     v
+--------------------------+     +-----------------------------+
| RevenueCat iOS SDK       |     | RevenueCat Android SDK      |
| (StoreKit 2)             |     | (Play Billing)              |
+--------------------------+     +-----------------------------+
              \                    /
               v                  v
+-------------------------------------------------------------+
| RevenueCat project: App Store app + Play Store app (+ Web)  |
| shared entitlements & offerings, one customer per App User  |
| ID, webhooks → your backend                                 |
+-------------------------------------------------------------+
```

Concepts (same as the native RevenueCat skills): **Product** (per store) → attached to **Entitlements** (shared) → shown through **Offerings/Packages** (shared identifiers, store-specific products). Code gates on **entitlement IDs** only, so the same TS code works on both platforms.

## 1. First, establish context

Find out (from the codebase or by asking):

- SDK **exact versions** (`react-native-purchases`, `react-native-purchases-ui`); React Native version; **Expo managed vs bare** (and Expo SDK version)
- Platforms: iOS, Android, and maybe **web** (RevenueCat Web Billing / Stripe via `purchases-js`, e.g. for an Expo web target) or macOS
- **Who completes purchases**: RevenueCat (default) or an existing plugin like `react-native-iap` / `expo-iap` (section 11)
- Account system: anonymous only, or your own user IDs (`logIn`) — required for sharing entitlements across platforms
- Paywall: RevenueCatUI (remote, both platforms) or custom
- Backend that needs purchase state? (webhooks + REST API)
- Migrating from a native-plugin implementation? (section 10)

Write the **cross-store entitlement map**:

```
entitlement    App Store product               Play product : base plan        Web (optional)
plus           com.app.plus.monthly / .yearly   plus : monthly / yearly         plus_monthly (Stripe/Web Billing)
pro            com.app.pro.monthly  / .yearly   pro  : monthly / yearly         pro_monthly
               com.app.lifetime                 lifetime                        —        (attach to plus + pro)
cloud_backup   com.app.addon.backup (own group) addon_backup : monthly          —
(none)         com.app.credits.100              credits_100                     —        (ledger)
```

Attach higher tiers to lower entitlements too (`pro` products → `plus` + `pro`) so checks stay simple on every platform.

## 2. RevenueCat project & store configuration

- [ ] **One RevenueCat project** with an **App Store app** and a **Play Store app** (and Web Billing app if used) — not one project per platform, or entitlements won't be shared
- [ ] App Store app: bundle ID, In-App Purchase key, App Store Server Notifications V2 → RevenueCat (ios-revenuecat-iap §2)
- [ ] Play Store app: package name, service account credentials (allow ~36 h to activate), RTDN topic (android-revenuecat-iap §2)
- [ ] Products from **each store** imported and attached to the **same entitlements**
- [ ] **Offerings** contain packages with the same identifiers (`$rc_monthly`, `$rc_annual`, `$rc_lifetime`, custom) and a product for **each** platform in every package; one offering marked Current
- [ ] Restore behavior chosen deliberately (Transfer / Transfer if no active / Keep with original)
- [ ] Android-only: non-consumable one-time products marked non-consumable; retention offers tagged `rc-ignore-offer` in Play Console
- [ ] Webhook + Authorization header if you have a backend
- [ ] Public SDK keys per platform: **`appl_…` for iOS, `goog_…` for Android** (and the web key for `purchases-js`); secret keys only on the server
- [ ] Store setup itself (products, groups/base plans, agreements, testers): ios-in-app-purchase §3/§8, android-in-app-purchase §3/§8

## 3. SDK setup checklist

- [ ] `Purchases.configure(...)` **once**, at app start (root layout / `App.tsx`), with the **platform's** API key (`Platform.select({ ios, android })`) — before any other Purchases call
- [ ] Debug log level only in debug builds (`__DEV__`)
- [ ] `appUserID` passed at configure time if already signed in
- [ ] One **customer-info listener** for the app's lifetime (`Purchases.addCustomerInfoUpdateListener`) feeding your state store; remove it only on teardown
- [ ] iOS: In-App Purchase capability in `ios/<App>`; `pod install` after upgrades (bare). Android: `minSdk` per SDK requirement; React Native activity setup per docs
- [ ] **Expo**: install via `npx expo install`, use a **development build** (EAS / prebuild). Expo Go can't make real purchases (recent SDK versions run a preview/mock mode there — never test purchase logic in Expo Go)
- [ ] No other purchase plugin listening to and finishing transactions at the same time (unless section 11)

## 4. Offerings, packages & paywall

- [ ] `getOfferings()` → `current`; handle `null` / empty packages with a retry state (wrong key for platform, products not attached for this platform, Play credentials not active yet)
- [ ] Packages by **type/identifier**, never by index — the same code then works on iOS and Android
- [ ] Prices from `package.product.priceString` (and `pricePerMonthString` etc. where available)
- [ ] Trial/intro text: iOS via `checkTrialOrIntroductoryPriceEligibility` + `introPrice`; Android via the product's default subscription option / `subscriptionOptions` (Play only returns eligible offers) — or let RevenueCatUI handle it
- [ ] Offering metadata for copy, with defaults
- [ ] RevenueCatUI: `RevenueCatUI.presentPaywallIfNeeded({ requiredEntitlementIdentifier: "pro" })` / `<RevenueCatUI.Paywall />`. Handle result (purchased / restored / cancelled / error)
- [ ] Paywall template shows Terms, Privacy, auto-renew disclosure, and works on both platforms' disclosure rules
- [ ] Custom paywall: Current plan / Upgrade / Downgrade labels per ios-in-app-purchase §13.2 — remember the **Android plan-change branch** (section 6.1)

## 5. User identity (cross-platform)

- [ ] Same **stable, non-PII App User ID** on every platform (your DB UUID) → the user's entitlement follows them from iPhone to Android to web
- [ ] `logIn(userId)` right after sign-in and **before** showing the paywall; `logOut()` on sign-out (not when anonymous — check `isAnonymous()`)
- [ ] Never email, device IDs, `""`, `"null"`, or constants as IDs
- [ ] Anonymous-only apps can't share entitlements across platforms — tell the user if that's a requirement
- [ ] Restore: `restorePurchases()`; empty result → "No previous purchases found"; restore under another App User ID follows the project's restore behavior (`TRANSFER` webhook)
- [ ] `syncPurchases()` only for silent migration (section 10)

## 6. Purchasing

- [ ] `purchasePackage(pkg)` (or `purchase(…)` params on newer SDKs) → apply the returned `customerInfo` immediately
- [ ] Cancellation → no alert: `e.userCancelled`
- [ ] Pending (`PURCHASES_ERROR_CODE.PAYMENT_PENDING_ERROR`) → "Waiting for approval/payment"; result arrives through the listener
- [ ] Network / store problems → retryable; `RECEIPT_ALREADY_IN_USE_ERROR` (belongs to another App User ID), `PRODUCT_ALREADY_PURCHASED_ERROR`, `INELIGIBLE_ERROR`, `PURCHASE_NOT_ALLOWED_ERROR` → specific messages
- [ ] Purchase button disabled while in flight
- [ ] Gate on **entitlements** in the returned/updated `CustomerInfo`
- [ ] Add-ons only while the base entitlement is active
- [ ] Don't offer a store purchase to a user whose active entitlement comes from **another store** (check `entitlement.store`) — show "Manage on your iPhone / in Google Play / on the web" instead, or they'll pay twice

### 6.1 Plan changes — the platform branch
| | iOS | Android |
|---|---|---|
| How | Normal `purchasePackage` of the other product in the same subscription group — StoreKit swaps | **Must** pass the current product + mode as the third argument: `{ oldProductIdentifier, prorationMode }` (newer SDKs: replacement-mode naming) |
| Upgrade | Immediate | `IMMEDIATE_AND_CHARGE_PRORATED_PRICE` (or `IMMEDIATE_WITH_TIME_PRORATION`) |
| Downgrade | At renewal | `DEFERRED` |
| Forgetting it | — | User billed for **two** subscriptions |

- [ ] After a downgrade (either store) the entitlement shows the old product until renewal → "Change scheduled", not "Purchase successful"
- [ ] Only apply the Android branch when the current entitlement's `store` is Play Store

### 6.2 Offers
| Offer | iOS | Android |
|---|---|---|
| Trial / intro | Automatic if eligible | Default subscription option (longest trial / cheapest intro) |
| Promotional / retention | `getPromotionalOffer(product, discount)` → `purchaseDiscountedPackage` (RevenueCat signs it) | Select a tagged `SubscriptionOption` (`rc-ignore-offer` + your tag) → `purchaseSubscriptionOption` |
| Codes | `presentCodeRedemptionSheet()` | Promo codes in Play Store, picked up on sync |
| Win-back | SDK win-back APIs where available | Developer-determined offers |

## 7. Reading state for the UI

- [ ] Map `CustomerInfo.entitlements` → app tiers in **one shared function** (Appendix) — not duplicated per platform
- [ ] Field mapping per ios-revenuecat-iap §7.1: `isActive`, `productIdentifier` (+ `productPlanIdentifier` on Play), `expirationDate` (ISO string in the hybrid SDK — parse it), `willRenew`, `periodType`, `billingIssueDetectedAt`, `store`, `isSandbox`
- [ ] **Manage**: `customerInfo.managementURL` points to the store the user paid on; or RevenueCatUI **Customer Center** (`RevenueCatUI.presentCustomerCenter()`); if `store` isn't the current platform's store, explain where to manage
- [ ] Freshness: listener drives the UI; `getCustomerInfo()` on start/resume (`AppState` → `active`); cached values for offline
- [ ] Pending downgrades: remember the target locally at purchase time or use Customer Center / your backend (`PRODUCT_CHANGE` webhook)

## 8. Consumables & non-renewing

- [ ] RevenueCat finishes/consumes consumables on both stores — credit balances **server-side from the `NON_RENEWING_PURCHASE` webhook**, idempotent by `transaction_id`; or evaluate **Virtual Currencies** if available
- [ ] Non-renewing passes: compute and stack expiry yourself (server), as in ios-revenuecat-iap §8
- [ ] Purchase history: `customerInfo.nonSubscriptionTransactions`

## 9. Backend

Same as ios-revenuecat-iap §9 — webhooks are store-agnostic; each event carries `store` (`APP_STORE`, `PLAY_STORE`, `STRIPE`, `RC_BILLING`…) and `environment`. Re-fetch the subscriber and overwrite; idempotent by `event.id`; secret key only on the server.

## 10. Migrating from native plugins (`react-native-iap`, `expo-iap`)

- [ ] Remove the old plugin's purchase listener and its `finishTransaction` calls once RevenueCat completes purchases — two systems finishing transactions race each other
- [ ] Keep product IDs (and Play base plan IDs); attach them to entitlements in RevenueCat
- [ ] Existing subscribers: `syncPurchases()` **once** per install after the update (guarded by a flag), and/or import purchases server-side
- [ ] Move App Store Server Notifications and Play RTDN to RevenueCat (or forward) — one source of truth
- [ ] Replace your own receipt/token verification endpoint with webhook-driven entitlement mirroring

## 11. Keeping an existing purchase plugin (`purchasesAreCompletedBy: myApp`)

Use only during a transition.
- [ ] Configure with `purchasesAreCompletedBy` = my app (set the StoreKit version for iOS) — option names differ per SDK version
- [ ] Your plugin still verifies, grants and finishes/acknowledges (react-native-in-app-purchase §4–5 apply)
- [ ] Call `syncPurchases()` (or the per-purchase record API your SDK offers) after each purchase so RevenueCat sees it
- [ ] Gate features from one source only

## 12. Testing plan

1. **iOS**: StoreKit Configuration file selected in the Xcode scheme (run from Xcode, not `npx react-native run-ios` / `npx expo run:ios`, if the config isn't picked up), then sandbox, then TestFlight
2. **Android**: license testers on an internal testing track, test cards (approve / decline / slow), accelerated renewals, Play Billing Lab
3. **RevenueCat dashboard**: sandbox data toggle; customer view shows purchases from both stores under one App User ID
4. **Cross-platform**: subscribe on one platform, sign in on the other
5. **Unit tests**: test the shared entitlement mapping with plain arrays (Appendix B.3); there's no public test double for the hybrid SDK
6. **Expo**: always a development build, never Expo Go

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| C1 | Fresh install (each platform) | Correct platform key; offerings load; prices in local currency |
| C2 | Current offering missing a product for one platform | That package hidden / retry state on that platform only |
| C3 | Subscribe with trial (iOS eligible / Android default option) | Trial shown; entitlement active |
| C4 | Cancel the store sheet | No alert |
| C5 | Pending (Ask to Buy / slow card) | "Waiting"; listener unlocks |
| C6 | Upgrade (iOS same group / Android with product change info) | One subscription; higher entitlement |
| C7 | Downgrade | "Change scheduled"; old tier until renewal |
| C8 | Android tier change **without** product change info (bug check) | Not reachable from the UI |
| C9 | Subscribed on iOS, sign in on Android | Entitlement active; no buy buttons; "Manage on your iPhone" |
| C10 | Subscribed on Android, sign in on iOS | Mirror of C9 |
| C11 | Anonymous purchase, then `logIn` | Purchase aliased; entitlement kept |
| C12 | Restore on second account | Per restore behavior; `TRANSFER` handled |
| C13 | `logOut` | Anonymous; premium hidden |
| C14 | Renewal / expiry / refund (each store) | Listener updates UI while open |
| C15 | Billing issue (grace) | Access kept; fix-payment message |
| C16 | iOS promotional offer / Android tagged offer | Correct price; entitlement active |
| C17 | Consumable | Webhook credits once on each store |
| C18 | RevenueCatUI paywall `presentPaywallIfNeeded` | Shown only without the entitlement; result handled |
| C19 | Customer Center | Opens; actions reflect the right store |
| C20 | Offline start as subscriber | Cached entitlements keep access |
| C21 | Expo Go (bug check) | Purchase logic not tested here; dev build used |

## 13. Common bugs to look for in review

- One RevenueCat **project per platform** → entitlements not shared across iOS/Android
- iOS key used on Android (or vice versa); key chosen at build time for the wrong platform
- `configure` called in a screen, twice, or after other Purchases calls; listener added in a component and duplicated on re-render
- Packages picked by index; offering missing a product for one platform and the code assumes it exists
- Gating on product IDs (`com.app.pro.monthly` vs `pro:monthly`) instead of entitlements — breaks on the other platform
- Android plan change without product change info → double billing; iOS code path passing Android-only params
- Selling on platform B to a user subscribed on platform A
- `expirationDate` treated as a `Date` without parsing (the hybrid SDK returns ISO strings)
- Different App User IDs on iOS and Android (e.g. one anonymous) → no cross-platform access
- Error alert on cancellation; pending shown as failure
- Old `react-native-iap` / `expo-iap` listener still finishing transactions alongside RevenueCat
- `syncPurchases()` on every launch or as the Restore button
- Testing in Expo Go; StoreKit config not applied because the app was launched from the CLI
- Secret key in the app; debug logging in release
- Paywall lacking Terms/Privacy or platform-correct cancellation text

## 14. Output format

When asked to **review/audit**:
1. **Summary** — bare RN vs Expo, SDK versions, platforms (+ web), project/apps setup, who completes purchases, identity model, paywall type, entitlement map across stores
2. **Checklist results** — sections 2–11 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android** where relevant (dashboard and console items listed for the user to confirm)
3. **Critical issues first** (no cross-platform sharing, double billing, cross-store double subscriptions, lost access, wrong-user merges, key exposure, policy risks) with fix code
4. **Testing gaps** — C1–C21 not covered

When asked to **implement**: start from the Appendix, adapt entitlement IDs, remove unused parts, then list dashboard, App Store Connect, Play Console, backend, and policy steps.

---

## Appendix B — React Native / Expo (`react-native-purchases` 8.x / `react-native-purchases-ui`)

### B.1 SubscriptionService (Zustand store example)

```ts
import { Platform } from "react-native";
import Purchases, { CustomerInfo, LOG_LEVEL, PACKAGE_TYPE, PurchasesOffering, PurchasesPackage,
                    PURCHASES_ERROR_CODE, PRORATION_MODE } from "react-native-purchases";
import { create } from "zustand";

type Tier = "free" | "plus" | "pro";

export const tierFor = (activeIds: string[]): Tier =>
  activeIds.includes("pro") ? "pro" : activeIds.includes("plus") ? "plus" : "free";   // pure, testable

type State = {
  offering: PurchasesOffering | null;
  tier: Tier;
  store?: string;
  expiresAt?: string;
  willRenew: boolean;
  billingIssue: boolean;
  purchasing: boolean;
  awaitingPayment: boolean;
  scheduledProductId?: string;
  message?: string;
};

export const useSubscriptions = create<State>(() => ({
  offering: null, tier: "free", willRenew: false, billingIssue: false, purchasing: false, awaitingPayment: false,
}));

export function apply(info: CustomerInfo) {
  const active = info.entitlements.active;
  const plan = active.pro ?? active.plus ?? info.entitlements.all.pro ?? info.entitlements.all.plus;
  useSubscriptions.setState((s) => ({
    tier: tierFor(Object.keys(active)),
    store: plan?.store,                       // "APP_STORE" | "PLAY_STORE" | ...
    expiresAt: plan?.expirationDate ?? undefined,
    willRenew: plan?.willRenew ?? false,
    billingIssue: !!plan?.billingIssueDetectedAt,
    awaitingPayment: false,
    scheduledProductId: s.scheduledProductId === plan?.productIdentifier ? undefined : s.scheduledProductId,
  }));
}

/** Call once at app start (root layout / App.tsx). */
export async function configureSubscriptions(userId?: string) {
  if (__DEV__) Purchases.setLogLevel(LOG_LEVEL.DEBUG);
  Purchases.configure({
    apiKey: Platform.select({ ios: "appl_XXXXXXXX", android: "goog_XXXXXXXX" })!,
    appUserID: userId,
  });
  Purchases.addCustomerInfoUpdateListener(apply);   // register once — not inside a component that re-renders
  apply(await Purchases.getCustomerInfo());
  await loadOfferings();
}

export async function loadOfferings() {
  try {
    const offerings = await Purchases.getOfferings();
    useSubscriptions.setState({ offering: offerings.current });
  } catch {
    useSubscriptions.setState({ message: "Couldn't load the store. Please try again." });
  }
}

export const managedElsewhere = () => {
  const { tier, store } = useSubscriptions.getState();
  if (tier === "free" || !store) return false;
  return Platform.OS === "ios" ? store !== "APP_STORE" : store !== "PLAY_STORE";
};

export async function purchase(pkg: PurchasesPackage, opts: { replacingProductId?: string; downgrade?: boolean } = {}) {
  const s = useSubscriptions.getState();
  if (s.purchasing || managedElsewhere()) return;
  useSubscriptions.setState({ purchasing: true, message: undefined });
  try {
    // Android tier change: REQUIRED (third argument), or the user is billed twice. Ignored on iOS.
    const changeInfo = Platform.OS === "android" && opts.replacingProductId
      ? { oldProductIdentifier: opts.replacingProductId,
          prorationMode: opts.downgrade ? PRORATION_MODE.DEFERRED : PRORATION_MODE.IMMEDIATE_AND_CHARGE_PRORATED_PRICE }
      : null;
    const { customerInfo } = await Purchases.purchasePackage(pkg, null, changeInfo);
    apply(customerInfo);
    if (opts.downgrade) useSubscriptions.setState({ scheduledProductId: pkg.product.identifier });
  } catch (e: any) {
    if (e.userCancelled) return;                                       // no alert
    if (e.code === PURCHASES_ERROR_CODE.PAYMENT_PENDING_ERROR) useSubscriptions.setState({ awaitingPayment: true });
    else if (e.code === PURCHASES_ERROR_CODE.NETWORK_ERROR) useSubscriptions.setState({ message: "You're offline. Try again." });
    else if (e.code === PURCHASES_ERROR_CODE.RECEIPT_ALREADY_IN_USE_ERROR)
      useSubscriptions.setState({ message: "This purchase belongs to another account." });
    else useSubscriptions.setState({ message: "Purchase failed. Please try again." });
  } finally {
    useSubscriptions.setState({ purchasing: false });
  }
}

export async function restore() {
  try {
    const info = await Purchases.restorePurchases();
    apply(info);
    if (Object.keys(info.entitlements.active).length === 0) useSubscriptions.setState({ message: "No previous purchases found." });
  } catch {
    useSubscriptions.setState({ message: "Restore failed. Please try again." });
  }
}

export async function signIn(userId: string) { apply((await Purchases.logIn(userId)).customerInfo); }
export async function signOut() { if (!(await Purchases.isAnonymous())) apply(await Purchases.logOut()); }

export const packageOf = (o: PurchasesOffering | null, type: PACKAGE_TYPE) =>
  o?.availablePackages.find((p) => p.packageType === type);          // never by index
```

### B.2 Paywall & Customer Center

```tsx
import RevenueCatUI, { PAYWALL_RESULT } from "react-native-purchases-ui";

export async function openProFeature(navigate: (route: string) => void) {
  const result = await RevenueCatUI.presentPaywallIfNeeded({ requiredEntitlementIdentifier: "pro" });
  if ([PAYWALL_RESULT.PURCHASED, PAYWALL_RESULT.RESTORED, PAYWALL_RESULT.NOT_PRESENTED].includes(result)) {
    navigate("ProFeature");
  }
}

export const openManage = () => RevenueCatUI.presentCustomerCenter();

// Embedded paywall component
export function PaywallScreen({ onClose }: { onClose: () => void }) {
  return (
    <RevenueCatUI.Paywall
      onPurchaseCompleted={({ customerInfo }) => { apply(customerInfo); onClose(); }}
      onRestoreCompleted={({ customerInfo }) => { apply(customerInfo); onClose(); }}
      onDismiss={onClose}
    />
  );
}
```

**Expo:** `npx expo install react-native-purchases react-native-purchases-ui`, then build a development client (`eas build --profile development` or `npx expo run:ios|android`). Don't test purchases in Expo Go.

**Bare React Native:** `npm install react-native-purchases react-native-purchases-ui`, then `cd ios && pod install`; rebuild the native app.

### B.3 Tests for the shared mapping

```ts
import { tierFor } from "./subscriptions";

test("pro implies pro", () => expect(tierFor(["plus", "pro"])).toBe("pro"));
test("add-on alone is free", () => expect(tierFor(["cloud_backup"])).toBe("free"));
```
