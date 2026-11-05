---
name: flutter-revenuecat-iap
description: Use when implementing, reviewing, testing, or debugging in-app purchases or subscriptions built on RevenueCat in a Flutter app (purchases_flutter, purchases_ui_flutter) — one RevenueCat project for App Store + Google Play (+ web), platform API keys, shared entitlements and offerings, paywalls, Customer Center, identity across platforms, Android plan changes, iOS offers, webhooks, migration from in_app_purchase, testing. For native iOS/Android apps use ios-revenuecat-iap / android-revenuecat-iap; for Flutter without RevenueCat use flutter-in-app-purchase; for React Native / Expo use react-native-revenuecat-iap.
---

# Flutter In-App Purchases with RevenueCat

Guide for building, auditing, and testing purchases in **Flutter** apps (`purchases_flutter`, `purchases_ui_flutter`) on RevenueCat. The SDK wraps the native RevenueCat iOS and Android SDKs, so RevenueCat handles store verification, finishing/acknowledging, notifications, and cross-device entitlement sync on both stores. **You still own**: both store consoles, the shared entitlement design, user identity, platform differences in the UI (plan changes, offers, manage links), consumable balances, and store-policy compliance.

Platform skills hold the store- and SDK-specific detail — this skill links to them instead of repeating it:
- **ios-revenuecat-iap** — dashboard setup for the App Store app, iOS offers, field mapping, webhooks (§9 event table)
- **android-revenuecat-iap** — Play service credentials, RTDN, subscription options/offer tags, Google replacement modes
- **ios-in-app-purchase** / **android-in-app-purchase** — store rules, App Review / Play policy, post-purchase UI
- **flutter-in-app-purchase** — Flutter purchases without RevenueCat (`in_app_purchase`)
- **react-native-revenuecat-iap** — the same RevenueCat setup for React Native / Expo

Hybrid SDK APIs track the native SDKs but rename things between majors (e.g. purchase methods returning `CustomerInfo` vs a result object, proration vs replacement mode naming). Verify every call against the version in `pubspec.yaml` and its migration guide.

## 0. Architecture (mental model)

```
+-------------------------------------------------------------+
| Shared UI (Flutter widgets)                                 |
| RevenueCatUI Paywall or custom paywall, status card,        |
| Customer Center, Restore                                    |
+-------------------------------------------------------------+
            ^ state (Riverpod/Bloc/Provider)
+-------------------------------------------------------------+
| SubscriptionService (singleton, configured at app start)    |
| - Purchases.configure(platform API key, appUserID)          |
| - customer-info listener → entitlements → app tiers         |
| - offerings, purchase, restore, logIn/logOut                |
| - platform branches: Android plan change, iOS promo offers, |
|   manage location by store                                  |
+-------------------------------------------------------------+
     | purchases_flutter (bridge)
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

Concepts (same as the native RevenueCat skills): **Product** (per store) → attached to **Entitlements** (shared) → shown through **Offerings/Packages** (shared identifiers, store-specific products). Code gates on **entitlement IDs** only, so the same Dart code works on both platforms.

## 1. First, establish context

Find out (from the codebase or by asking):

- SDK **exact versions** (`purchases_flutter`, `purchases_ui_flutter`) and Flutter version
- Platforms: iOS, Android, and maybe **web** (RevenueCat Web Billing / Stripe via `purchases-js`) or macOS
- **Who completes purchases**: RevenueCat (default) or an existing plugin like `in_app_purchase` (section 11)
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

- [ ] `Purchases.configure(...)` **once**, at app start, with the **platform's** API key (`Platform.isIOS ? appleKey : googleKey`) — before any other Purchases call
- [ ] Debug log level only in debug builds (`kDebugMode`)
- [ ] `appUserID` passed at configure time if already signed in
- [ ] One **customer-info listener** for the app's lifetime (`Purchases.addCustomerInfoUpdateListener`) feeding your state store; remove it only on teardown
- [ ] iOS: In-App Purchase capability in `ios/Runner`; `pod install` after upgrades. Android: `minSdk` per SDK requirement; `MainActivity` extends `FlutterFragmentActivity` (needed by RevenueCatUI)
- [ ] No other purchase plugin listening to and finishing transactions at the same time (unless section 11)

## 4. Offerings, packages & paywall

- [ ] `getOfferings()` → `current`; handle `null` / empty packages with a retry state (wrong key for platform, products not attached for this platform, Play credentials not active yet)
- [ ] Packages by **type/identifier**, never by index — the same code then works on iOS and Android
- [ ] Prices from `package.storeProduct.priceString` (and `pricePerMonthString` etc. where available)
- [ ] Trial/intro text: iOS via `checkTrialOrIntroductoryPriceEligibility` + `introductoryPrice`; Android via the product's default subscription option / `subscriptionOptions` (Play only returns eligible offers) — or let RevenueCatUI handle it
- [ ] Offering metadata for copy, with defaults
- [ ] RevenueCatUI: `RevenueCatUI.presentPaywallIfNeeded("pro")` / `PaywallView`. Handle result (purchased / restored / cancelled / error)
- [ ] Paywall template shows Terms, Privacy, auto-renew disclosure, and works on both platforms' disclosure rules
- [ ] Custom paywall: Current plan / Upgrade / Downgrade labels per ios-in-app-purchase §13.2 — remember the **Android plan-change branch** (section 6.1)

## 5. User identity (cross-platform)

- [ ] Same **stable, non-PII App User ID** on every platform (your DB UUID) → the user's entitlement follows them from iPhone to Android to web
- [ ] `logIn(userId)` right after sign-in and **before** showing the paywall; `logOut()` on sign-out (not when anonymous — check `isAnonymous`)
- [ ] Never email, device IDs, `""`, `"null"`, or constants as IDs
- [ ] Anonymous-only apps can't share entitlements across platforms — tell the user if that's a requirement
- [ ] Restore: `restorePurchases()`; empty result → "No previous purchases found"; restore under another App User ID follows the project's restore behavior (`TRANSFER` webhook)
- [ ] `syncPurchases()` only for silent migration (section 10)

## 6. Purchasing

- [ ] `purchasePackage(pkg)` (or `purchase(PurchaseParams…)` on newer SDKs) → apply the returned `CustomerInfo` immediately
- [ ] Cancellation → no alert: `PurchasesErrorHelper.getErrorCode(e) == PurchasesErrorCode.purchaseCancelledError` (from `PlatformException`)
- [ ] Pending (`paymentPendingError`) → "Waiting for approval/payment"; result arrives through the listener
- [ ] Network / store problems → retryable; `receiptAlreadyInUseError` (belongs to another App User ID), `productAlreadyPurchasedError`, `ineligibleError`, `purchaseNotAllowedError` → specific messages
- [ ] Purchase button disabled while in flight
- [ ] Gate on **entitlements** in the returned/updated `CustomerInfo`
- [ ] Add-ons only while the base entitlement is active
- [ ] Don't offer a store purchase to a user whose active entitlement comes from **another store** (check `entitlement.store`) — show "Manage on your iPhone / in Google Play / on the web" instead, or they'll pay twice

### 6.1 Plan changes — the platform branch
| | iOS | Android |
|---|---|---|
| How | Normal `purchasePackage` of the other product in the same subscription group — StoreKit swaps | **Must** pass the current product + mode: `googleProductChangeInfo: GoogleProductChangeInfo(oldProductId, prorationMode: …)` (newer SDKs: replacement-mode naming) |
| Upgrade | Immediate | `immediateAndChargeProratedPrice` (or `immediateWithTimeProration`) |
| Downgrade | At renewal | `deferred` |
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

- [ ] Map `CustomerInfo.entitlements` → app tiers in **one shared function** (Appendix A.1) — not duplicated per platform
- [ ] Field mapping per ios-revenuecat-iap §7.1: `isActive`, `productIdentifier` (+ `productPlanIdentifier` on Play), `expirationDate` (ISO string in the hybrid SDK — parse it with `DateTime.tryParse`), `willRenew`, `periodType`, `billingIssueDetectedAt`, `store`, `isSandbox`
- [ ] **Manage**: `customerInfo.managementURL` points to the store the user paid on; or RevenueCatUI **Customer Center** (`RevenueCatUI.presentCustomerCenter()`); if `store` isn't the current platform's store, explain where to manage
- [ ] Freshness: listener drives the UI; `getCustomerInfo()` on start/resume (`AppLifecycleState.resumed`); cached values for offline
- [ ] Pending downgrades: remember the target locally at purchase time or use Customer Center / your backend (`PRODUCT_CHANGE` webhook)

## 8. Consumables & non-renewing

- [ ] RevenueCat finishes/consumes consumables on both stores — credit balances **server-side from the `NON_RENEWING_PURCHASE` webhook**, idempotent by `transaction_id`; or evaluate **Virtual Currencies** if available
- [ ] Non-renewing passes: compute and stack expiry yourself (server), as in ios-revenuecat-iap §8
- [ ] Purchase history: `customerInfo.nonSubscriptionTransactions`

## 9. Backend

Same as ios-revenuecat-iap §9 — webhooks are store-agnostic; each event carries `store` (`APP_STORE`, `PLAY_STORE`, `STRIPE`, `RC_BILLING`…) and `environment`. Re-fetch the subscriber and overwrite; idempotent by `event.id`; secret key only on the server.

## 10. Migrating from native plugins (`in_app_purchase`)

- [ ] Remove the old plugin's purchase stream listener and its `completePurchase` calls once RevenueCat completes purchases — two systems finishing transactions race each other
- [ ] Keep product IDs (and Play base plan IDs); attach them to entitlements in RevenueCat
- [ ] Existing subscribers: `syncPurchases()` **once** per install after the update (guarded by a flag), and/or import purchases server-side
- [ ] Move App Store Server Notifications and Play RTDN to RevenueCat (or forward) — one source of truth
- [ ] Replace your own receipt/token verification endpoint with webhook-driven entitlement mirroring

## 11. Keeping an existing purchase plugin (`purchasesAreCompletedBy: myApp`)

Use only during a transition.
- [ ] Configure with `purchasesAreCompletedBy` = my app (set the StoreKit version for iOS) — option names differ per SDK version
- [ ] Your plugin still verifies, grants and finishes/acknowledges (flutter-in-app-purchase §4–5 apply)
- [ ] Call `syncPurchases()` (or the per-purchase record API your SDK offers) after each purchase so RevenueCat sees it
- [ ] Gate features from one source only

## 12. Testing plan

1. **iOS**: StoreKit Configuration file selected in the Xcode scheme (run from Xcode, not `flutter run`, if the config isn't picked up), then sandbox, then TestFlight
2. **Android**: license testers on an internal testing track, test cards (approve / decline / slow), accelerated renewals, Play Billing Lab
3. **RevenueCat dashboard**: sandbox data toggle; customer view shows purchases from both stores under one App User ID
4. **Cross-platform**: subscribe on one platform, sign in on the other
5. **Unit tests**: test the shared entitlement mapping (`tierFor`) with plain sets (Appendix A.1, A.4); there's no public test double for the hybrid SDK

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
| C21 | Android RevenueCatUI with `FlutterFragmentActivity` (bug check) | Paywall opens on Android |

## 13. Common bugs to look for in review

- One RevenueCat **project per platform** → entitlements not shared across iOS/Android
- iOS key used on Android (or vice versa); key chosen at build time for the wrong platform
- `configure` called in a screen, twice, or after other Purchases calls; listener added in a widget and duplicated on rebuild
- Packages picked by index; offering missing a product for one platform and the code assumes it exists
- Gating on product IDs (`com.app.pro.monthly` vs `pro:monthly`) instead of entitlements — breaks on the other platform
- Android plan change without `GoogleProductChangeInfo` → double billing; iOS code path passing Android-only params
- Selling on platform B to a user subscribed on platform A
- `expirationDate` treated as a `DateTime` without parsing (the hybrid SDK returns ISO strings)
- Different App User IDs on iOS and Android (e.g. one anonymous) → no cross-platform access
- Error alert on cancellation; pending shown as failure
- Old `in_app_purchase` listener still completing purchases alongside RevenueCat
- `syncPurchases()` on every launch or as the Restore button
- StoreKit config not applied because the app was launched from the CLI (`flutter run`)
- `MainActivity` not a `FlutterFragmentActivity` → RevenueCatUI paywall fails on Android
- Secret key in the app; debug logging in release
- Paywall lacking Terms/Privacy or platform-correct cancellation text

## 14. Output format

When asked to **review/audit**:
1. **Summary** — SDK versions, platforms (+ web), project/apps setup, who completes purchases, identity model, paywall type, entitlement map across stores
2. **Checklist results** — sections 2–11 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android** where relevant (dashboard and console items listed for the user to confirm)
3. **Critical issues first** (no cross-platform sharing, double billing, cross-store double subscriptions, lost access, wrong-user merges, key exposure, policy risks) with fix code
4. **Testing gaps** — C1–C21 not covered

When asked to **implement**: start from the Appendix, adapt entitlement IDs, remove unused parts, then list dashboard, App Store Connect, Play Console, backend, and policy steps.

---

## Appendix A — Flutter (`purchases_flutter` 8.x / `purchases_ui_flutter`)

### A.1 Shared mapping (pure Dart, unit-testable)

```dart
enum Tier { free, plus, pro }

class Entitlement {
  static const plus = 'plus';
  static const pro = 'pro';
  static const cloudBackup = 'cloud_backup';
}

class AppEntitlements {
  const AppEntitlements({this.tier = Tier.free, this.addons = const {}, this.store,
                         this.expiresAt, this.willRenew = false, this.billingIssue = false});
  final Tier tier;
  final Set<String> addons;
  final String? store;          // 'APP_STORE' | 'PLAY_STORE' | 'STRIPE' | ...
  final DateTime? expiresAt;
  final bool willRenew;
  final bool billingIssue;

  bool get hasBasePlan => tier != Tier.free;
}

/// Pure function over active entitlement IDs — test it with plain sets.
Tier tierFor(Set<String> activeIds) =>
    activeIds.contains(Entitlement.pro) ? Tier.pro
    : activeIds.contains(Entitlement.plus) ? Tier.plus
    : Tier.free;
```

### A.2 SubscriptionService

```dart
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:purchases_flutter/purchases_flutter.dart';

class SubscriptionService extends ChangeNotifier {
  static const _appleKey = 'appl_XXXXXXXX';
  static const _googleKey = 'goog_XXXXXXXX';

  Offering? offering;
  AppEntitlements entitlements = const AppEntitlements();
  bool purchasing = false;
  bool awaitingPayment = false;
  String? scheduledProductId;   // deferred downgrade hint
  String? message;

  /// Call once at app start, before runApp's UI needs purchase state.
  Future<void> configure({String? userId}) async {
    if (kDebugMode) await Purchases.setLogLevel(LogLevel.debug);
    final config = PurchasesConfiguration(Platform.isIOS ? _appleKey : _googleKey)..appUserID = userId;
    await Purchases.configure(config);
    Purchases.addCustomerInfoUpdateListener(_apply);   // renewals, refunds, other devices, approvals
    _apply(await Purchases.getCustomerInfo());
    await loadOfferings();
  }

  Future<void> loadOfferings() async {
    try {
      offering = (await Purchases.getOfferings()).current;
    } on PlatformException {
      message = "Couldn't load the store. Please try again.";
    }
    notifyListeners();
  }

  /// Another store owns the active subscription → don't sell here.
  bool get managedElsewhere {
    final store = entitlements.store;
    if (!entitlements.hasBasePlan || store == null) return false;
    return Platform.isIOS ? store != 'APP_STORE' : store != 'PLAY_STORE';
  }

  Future<void> purchase(Package package, {String? replacingProductId, bool downgrade = false}) async {
    if (purchasing || managedElsewhere) return;
    purchasing = true; message = null; notifyListeners();
    try {
      final info = await Purchases.purchasePackage(
        package,
        // Android tier change: REQUIRED, or the user is billed twice. Ignored on iOS.
        googleProductChangeInfo: Platform.isAndroid && replacingProductId != null
            ? GoogleProductChangeInfo(replacingProductId,
                prorationMode: downgrade ? GoogleProrationMode.deferred
                                         : GoogleProrationMode.immediateAndChargeProratedPrice)
            : null,
      );
      _apply(info);
      if (downgrade) scheduledProductId = package.storeProduct.identifier;
    } on PlatformException catch (e) {
      switch (PurchasesErrorHelper.getErrorCode(e)) {
        case PurchasesErrorCode.purchaseCancelledError:
          break;                                         // no alert
        case PurchasesErrorCode.paymentPendingError:
          awaitingPayment = true;
        case PurchasesErrorCode.networkError:
          message = "You're offline. Check your connection and try again.";
        case PurchasesErrorCode.receiptAlreadyInUseError:
          message = 'This purchase belongs to another account.';
        default:
          message = 'Purchase failed. Please try again.';
      }
    } finally {
      purchasing = false; notifyListeners();
    }
  }

  Future<void> restore() async {
    try {
      final info = await Purchases.restorePurchases();
      _apply(info);
      if (info.entitlements.active.isEmpty) message = 'No previous purchases found.';
    } on PlatformException {
      message = 'Restore failed. Please try again.';
    }
    notifyListeners();
  }

  Future<void> signIn(String userId) async => _apply((await Purchases.logIn(userId)).customerInfo);

  Future<void> signOut() async {
    if (await Purchases.isAnonymous) return;
    _apply(await Purchases.logOut());
  }

  void _apply(CustomerInfo info) {
    final active = info.entitlements.active;
    final plan = active[Entitlement.pro] ?? active[Entitlement.plus]
        ?? info.entitlements.all[Entitlement.pro] ?? info.entitlements.all[Entitlement.plus];
    entitlements = AppEntitlements(
      tier: tierFor(active.keys.toSet()),
      addons: {if (active.containsKey(Entitlement.cloudBackup)) Entitlement.cloudBackup},
      store: plan?.store.name.toUpperCase(),                         // verify enum naming for your SDK
      expiresAt: plan?.expirationDate != null ? DateTime.tryParse(plan!.expirationDate!) : null,
      willRenew: plan?.willRenew ?? false,
      billingIssue: plan?.billingIssueDetectedAt != null,
    );
    if (scheduledProductId != null && plan?.productIdentifier == scheduledProductId) scheduledProductId = null;
    awaitingPayment = false;
    notifyListeners();
  }
}
```

### A.3 Paywall & Customer Center

```dart
import 'package:purchases_ui_flutter/purchases_ui_flutter.dart';

Future<void> openProFeature(BuildContext context) async {
  // Shows the remote paywall only if the user lacks the entitlement.
  final result = await RevenueCatUI.presentPaywallIfNeeded(Entitlement.pro);
  if (result == PaywallResult.purchased || result == PaywallResult.restored || result == PaywallResult.notPresented) {
    if (context.mounted) Navigator.of(context).pushNamed('/pro-feature');
  }
}

Future<void> openManage() => RevenueCatUI.presentCustomerCenter();
```

Android: make `MainActivity` extend `FlutterFragmentActivity` for RevenueCatUI.

### A.4 Tests for the shared mapping

```dart
import 'package:flutter_test/flutter_test.dart';
import 'subscriptions.dart';

void main() {
  test('pro implies pro', () => expect(tierFor({'plus', 'pro'}), Tier.pro));
  test('add-on alone is free', () => expect(tierFor({'cloud_backup'}), Tier.free));
}
```
