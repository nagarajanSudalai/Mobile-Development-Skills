---
name: flutter-in-app-purchase
description: Use when implementing, reviewing, testing, or debugging in-app purchases or subscriptions in a Flutter app using the in_app_purchase plugin (in_app_purchase_storekit, in_app_purchase_android) against the App Store and Google Play directly — shared product/entitlement model across stores, purchaseStream listener, completePurchase, Android acknowledgement/consume, plan changes per platform (ChangeSubscriptionParam), offers, restore, unified server verification (App Store Server API + Play Developer API), store policies, testing. For RevenueCat in Flutter use flutter-revenuecat-iap; for React Native / Expo use react-native-in-app-purchase; for store-specific detail in native apps see ios-in-app-purchase and android-in-app-purchase.
---

# Flutter In-App Purchases (`in_app_purchase`)

Guide for building, auditing, and testing purchases in **Flutter** apps (`in_app_purchase`) that use the **App Store** and **Google Play** directly (no RevenueCat or other purchase backend). One shared Dart layer drives two different stores, so the hard part is keeping a **single entitlement model** while respecting each store's rules.

Related skills — this skill links to them instead of repeating their content:
- **ios-in-app-purchase** — StoreKit rules, subscription groups, App Store Connect, App Review, post-purchase UI (§13)
- **android-in-app-purchase** — Play Billing rules, base plans & offers, replacement modes, acknowledgement, RTDN, Play policy
- **flutter-revenuecat-iap** — the same app built on RevenueCat instead of direct store APIs
- **react-native-in-app-purchase** — the React Native / Expo equivalent (`react-native-iap`, `expo-iap`)

Plugin APIs change across major versions (`in_app_purchase` platform packages moved to StoreKit 2 and newer Play Billing). Verify every call against the version in `pubspec.yaml` before shipping.

## 0. Architecture (mental model)

```
+-------------------------------------------------------------+
| Shared UI (Flutter widgets)                                 |
| Paywall, Plan status, Change plan, Restore, Manage          |
+-------------------------------------------------------------+
            ^ state (Stream / ChangeNotifier / Riverpod / Bloc)
+-------------------------------------------------------------+
| PurchaseService (singleton, started at app launch)          |
| - one purchase listener for the whole app lifetime          |
| - shared catalog: store product IDs → entitlement           |
| - verify (server) → grant → complete → refresh              |
| - platform branches: iOS appAccountToken / Android offers,  |
|   obfuscatedAccountId, replacement modes, consume           |
+-------------------------------------------------------------+
     | in_app_purchase (+ _storekit / _android)
     v
+--------------------------+     +-----------------------------+
| StoreKit (iOS/macOS)     |     | Play Billing (Android)      |
+--------------------------+     +-----------------------------+
     | JWS / receipt                  | purchase token
     v                                v
+-------------------------------------------------------------+
| Backend: one entitlement table per user                     |
| - App Store Server API + Notifications V2                   |
| - Play Developer API + RTDN                                 |
+-------------------------------------------------------------+
```

The two stores model the same things differently:

| Concept | App Store | Google Play | Shared layer |
|---|---|---|---|
| Tier switching | One **subscription group**; StoreKit swaps plans | **No groups**; must pass old purchase + replacement mode | `changePlan(from:to:)` with platform branches |
| Period | Separate product per period | **Base plan** inside one product | Catalog maps (product, base plan?) → tier + period |
| Trial / intro | Intro offer per group, automatic | **Offer** on a base plan, needs `offerToken` | `bestOffer()` on Android only |
| Proof of purchase | JWS transaction (SK2) / app receipt (SK1) | Purchase token | `verificationData.serverVerificationData` |
| Finish | `finish()` (redelivered until finished) | **Acknowledge** within 3 days or refunded; consume for consumables | `completePurchase` after grant |
| User link | `appAccountToken` (UUID) | `obfuscatedAccountId` (≤ 64 chars) | `accountToken(userId)` per platform |
| Restore | `AppStore.sync()` / restore | Automatic via query | `restore()` |
| Manage | `showManageSubscriptions` / App Store URL | Play subscriptions deep link | `openManage()` |

## 1. First, establish context

Find out (from the codebase or by asking):

- Plugin with **exact versions**: `in_app_purchase` (+ `in_app_purchase_storekit`, `in_app_purchase_android`)
- Target stores (App Store, Play; also macOS? Amazon?) and minimum OS versions (StoreKit 2 needs iOS 15+)
- Product catalog by type per store (consumable / non-consumable / subscriptions with groups or base plans / add-ons)
- Backend? Strongly recommended — without it you can't know subscription expiry on Android or verify purchases reliably on either store
- Account system and whether entitlements must work across platforms (bought on iOS, used on Android/web)
- State management in use (Riverpod/Bloc/Provider) — the service should plug into it, not replace it

Write a **cross-store entitlement map** before coding:

```
entitlement   App Store product (group: premium)     Google Play product : base plan     tier
plus          com.app.plus.monthly / .yearly          plus : monthly / yearly             PLUS
pro           com.app.pro.monthly  / .yearly          pro  : monthly / yearly             PRO
lifetime      com.app.lifetime                        lifetime                            PRO
cloud_backup  com.app.addon.backup (own group)        addon_backup : monthly              add-on
credits_100   com.app.credits.100                     credits_100                         +100
```

Use the same IDs on both stores where the store allows it; keep the mapping explicit either way. Gate features on **entitlements/tiers**, never on raw product IDs or substrings.

## 2. Store setup

- [ ] **App Store Connect**: products, subscription group levels, add-ons in separate groups, Paid Apps agreement (ios-in-app-purchase §3, §8)
- [ ] **Play Console**: subscription products with base plans and offers, one-time products, app uploaded to a testing track, license testers (android-in-app-purchase §3, §8)
- [ ] iOS: **In-App Purchase** capability in the Xcode project (`ios/Runner`)
- [ ] Android: billing permission comes from the Play Billing dependency; `minSdk` meets the plugin's requirement
- [ ] Bundle ID / applicationId identical to the store listings; Play: upload a signed build once so products can be queried
- [ ] Backend has App Store Server API credentials (In-App Purchase key) and a Play service account; App Store Server Notifications V2 and Play RTDN point to it

## 3. Service lifecycle

- [ ] **One** `PurchaseService` created at app start (before `runApp` work completes), not inside the paywall
- [ ] Subscribe to the purchase stream **immediately**: `InAppPurchase.instance.purchaseStream.listen`. This catches renewals, Ask-to-Buy approvals, pending Play purchases, promo codes, and transactions left over from a crash
- [ ] Keep the subscription alive for the app's lifetime; cancel it only on app teardown (`dispose`)
- [ ] Check availability (`isAvailable()`); show "store unavailable" if false (no Play Store, restricted device, emulator without Play)
- [ ] Query products once per session (`queryProductDetails`); log `notFoundIDs` and show the rest
- [ ] Re-sync on app resume: Android re-query purchases (catches out-of-app purchases and missed acknowledgements); refresh entitlements from the server on both platforms

## 4. Purchasing — shared rules

- [ ] Purchase only from a user action; button disabled while in flight
- [ ] Link purchases to your user: iOS `appAccountToken` (UUID derived from the user ID); Android `obfuscatedAccountId` (hash ≤ 64 chars). Flutter passes `applicationUserName` in `PurchaseParam` (used for both — check how your platform package maps it)
- [ ] Handle every status: **purchased**, **restored** (iOS), **pending** (Ask to Buy / slow payment — show "waiting", grant nothing), **canceled** (no alert), **error** (short retryable message)
- [ ] Grant only after **server verification** (send `serverVerificationData` + platform + product ID), idempotent by transaction ID / purchase token
- [ ] **Complete only after granting**: `completePurchase` when `pendingCompletePurchase` is true. On Android this acknowledges (or consumes) — skipping it means an automatic refund after 3 days; on iOS unfinished transactions are redelivered
- [ ] Consumables: `buyConsumable(autoConsume: false)` then consume (`InAppPurchaseAndroidPlatformAddition.consumePurchase`) after the balance is persisted (or `autoConsume: true` only if you credit before completing)
- [ ] Never complete transactions before verification
- [ ] Gate the UI on the **refreshed entitlements** (server or recomputed), not on "purchase call returned"

## 5. Platform branches

### 5.1 iOS
- [ ] Prefer **StoreKit 2** mode of the plugin (`in_app_purchase_storekit` SK2 opt-in where required) — verify for your version
- [ ] Server verifies the **JWS** (SK2) or the app receipt (SK1) with the App Store Server API (ios-in-app-purchase §7)
- [ ] Plan changes inside a subscription group are a normal purchase of the other product — don't block "already subscribed"; downgrades apply at renewal (ios-in-app-purchase §3.2)
- [ ] Offer code sheet: `InAppPurchaseStoreKitPlatformAddition` / `SKPaymentQueueWrapper().presentCodeRedemptionSheet()` — redemption arrives via the purchase stream
- [ ] Price-increase consent and promotional offers need platform-specific APIs (promotional offers require a server signature)
- [ ] Restore button required (`restorePurchases()`) — restored transactions also need completing

### 5.2 Android
- [ ] Subscriptions carry **offers**: `in_app_purchase_android` returns one `GooglePlayProductDetails` per offer (with `offerToken` / `subscriptionIndex`) — **choose the offer explicitly** (eligible trial/intro, else base plan offer) and pass its token
- [ ] Tier change **must** pass the old purchase and a replacement mode — `GooglePlayPurchaseParam(changeSubscriptionParam: ChangeSubscriptionParam(oldPurchaseDetails:, replacementMode:))` (names vary by version). Otherwise the user is billed for two subscriptions (android-in-app-purchase §3.2)
- [ ] Each plan change creates a **new purchase token** → verify + complete it
- [ ] Pending purchases (`PurchaseStatus.pending`) → no access until purchased
- [ ] Re-query purchases on resume (`queryPastPurchases`/`restorePurchases`) and complete anything unacknowledged
- [ ] The client has no expiry, base plan, or grace/hold state — get them from the server (subscriptionsv2)

## 6. Entitlements & restore

- [ ] **Server is the source of truth** for both platforms: `GET /entitlements` returns tier, add-ons, expiry, renewal state, store, pending change
- [ ] Cache the last server response for offline use; re-verify when online
- [ ] Without a server: iOS can derive state from StoreKit (plugin-specific APIs are limited — consider a small native module or `in_app_purchase_storekit` SK2 APIs); Android can only tell "active or not" from the purchase list
- [ ] Cross-platform: a user who subscribed on iOS gets access on Android via the server entitlement, but **manages** the subscription on the store they paid on — show "Manage on your iPhone / in Google Play" based on `store`
- [ ] Restore button → platform restore + server re-link; nothing found → "No previous purchases found" (informational)
- [ ] Don't sell a second subscription on platform B to a user already subscribed on platform A (check the server entitlement before showing buy buttons)

## 7. Offers & pricing

- [ ] Prices from product details (`price` / `rawPrice` + `currencyCode`) — never hardcoded; Android subscription price = the offer's final (recurring) pricing phase
- [ ] iOS intro eligibility per group (plugin or server); Android shows only offers Play returns for the user
- [ ] Trial text: duration + price after the trial, per platform
- [ ] Yearly savings = `(monthly × 12 − yearly) / (monthly × 12)` using the same currency on the same platform

## 8. Backend: one entitlement model, two stores

- [ ] Verification endpoint accepts `{ platform, productId, transactionId | purchaseToken, jws | receipt, userId }` and routes to the **App Store Server API** (Apple's App Store Server Library) or **Play Developer API** (`purchases.subscriptionsv2.get`, `purchases.products.get`)
- [ ] Checks: bundle ID / package name, product in the catalog, `appAccountToken` / `obfuscatedExternalAccountId` matches the user, not revoked
- [ ] Android: acknowledge server-side if the client may not (or rely on client `completePurchase` — but make sure someone does it within 3 days); retire `linkedPurchaseToken` on plan changes
- [ ] Notifications: App Store Server Notifications V2 + Play RTDN update the same entitlement table; idempotent; re-fetch on each event (ios-in-app-purchase §7, android-in-app-purchase §7)
- [ ] Data model: user → many purchases, each with `store`, product, original transaction ID / purchase token, expiry, state; entitlements computed as the union
- [ ] Sandbox / test purchases flagged and excluded from production analytics

## 9. Store policies

- [ ] Both stores: digital goods via IAP / Play Billing (except programs you're enrolled in for specific regions), Restore available (iOS required), Terms + Privacy links, clear subscription terms, auto-renew and cancellation info, no dark patterns (ios-in-app-purchase §9, android-in-app-purchase §9)
- [ ] Platform-specific disclosure text ("Manage in Settings" on iOS, "Google Play → Payments & subscriptions" on Android)
- [ ] Don't mention the other platform's payment or prices in-app ("cheaper on web") unless a program allows it in that region


## 10. Testing plan

1. **iOS**: Xcode StoreKit Configuration file — open `ios/Runner.xcworkspace`, set it in the scheme, run from Xcode; then sandbox account on a device; TestFlight
2. **Android**: license testers, internal testing track, test cards (approve / decline / slow), accelerated renewals, Play Billing Lab
3. **Dart unit tests** with a fake store adapter (Appendix) — no plugin in tests
4. **Server**: sandbox notifications from both stores

### Scenario matrix

| # | Scenario | iOS | Android |
|---|---|---|---|
| X1 | Store unavailable | Error state | No Play / emulator → error state |
| X2 | Products partly missing | Valid ones shown; IDs logged | Same |
| X3 | Buy subscription with trial | Intro shown only if eligible | Trial offer token passed |
| X4 | Cancel the sheet | No alert | No alert |
| X5 | Pending (Ask to Buy / slow card) | "Waiting"; unlock via stream | Same |
| X6 | App killed before complete | Redelivered on launch; granted once | Re-query on launch; acknowledged within 3 days |
| X7 | Verification fails | Not completed; retried | Not acknowledged; retried before 3 days |
| X8 | Upgrade | Same-group purchase, immediate | `changeSubscriptionParam` + CHARGE_PRORATED_PRICE |
| X9 | Downgrade | Scheduled at renewal | DEFERRED; scheduled |
| X10 | Consumable | Credited once; finished | Credited once; consumed |
| X11 | Restore / reinstall | Restore → granted | Re-query → granted |
| X12 | Subscribed on iOS, open Android app | Server entitlement → access; "Manage on iPhone" | — |
| X13 | Refund | Access removed via server | Same |
| X14 | Expiry / renewal (accelerated) | Server updates; UI refreshes on resume | Same |
| X15 | Offer code / promo code | Arrives via stream | Re-query on resume |
| X16 | Sign out / sign in as another user | Purchases not granted to the wrong account | Same |
| X17 | Offline launch | Cached entitlements | Same |
| X18 | Post-purchase UI states | ios-in-app-purchase §13.6 | android-in-app-purchase §13 |

## 11. Common bugs to look for in review

- Purchase listener created in the paywall widget → renewals, Ask-to-Buy approvals, leftover transactions missed; or several listeners → duplicate grants
- `completePurchase` never called (Android refunds after 3 days; iOS redelivers forever) — or called before verification/granting
- Restored iOS transactions not completed
- Granting on `pending`; error alert on `canceled`
- `autoConsume: true` with credit-after-complete logic
- Android tier change without old purchase + replacement mode → double billing
- Android offer picked by index (`subscriptionOfferDetails[0]`) or no `offerToken` → wrong price / developer error
- Trusting `localVerificationData` on the device instead of server verification
- Gating on product IDs / substrings; same entitlement logic duplicated per platform and drifting
- No server → Android "active forever" (no expiry) or iOS expiry ignored
- Selling a second subscription on Android to a user already subscribed on iOS
- StoreKit config not selected because the app was run from the CLI (`flutter run`) instead of Xcode
- Plugin upgraded across a major version without migrating the API
- Hardcoded prices; iOS-only disclosure text shown on Android or vice versa
- No Restore button on iOS; missing Terms/Privacy links

## 12. Output format

When asked to **review/audit**:
1. **Summary** — plugin + versions, platforms, StoreKit mode, backend yes/no, cross-store entitlement map
2. **Checklist results** — sections 2–9 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android** where relevant (console items listed for the user to confirm)
3. **Critical issues first** (refunds from missing acknowledgement, lost or double-granted purchases, double billing, cross-platform double subscriptions, policy risks) with fix code
4. **Testing gaps** — X1–X18 not covered, per platform

When asked to **implement**: start from Appendix A, adapt the catalog, remove unused parts, then list App Store Connect, Play Console, backend, and policy steps.

---

## Appendix A — Flutter (`in_app_purchase` 3.x)

Verify class names against your `in_app_purchase`, `in_app_purchase_storekit` and `in_app_purchase_android` versions.

### A.1 Catalog

```dart
enum Tier { free, plus, pro }

class Catalog {
  // iOS product IDs (one subscription group) and Android product IDs (base plans inside).
  static const iosTier = {
    'com.app.plus.monthly': Tier.plus, 'com.app.plus.yearly': Tier.plus,
    'com.app.pro.monthly': Tier.pro, 'com.app.pro.yearly': Tier.pro,
  };
  static const androidTier = {'plus': Tier.plus, 'pro': Tier.pro};
  static const lifetime = {'com.app.lifetime', 'lifetime'};
  static const credits = {'com.app.credits.100': 100, 'credits_100': 100};

  static Map<String, Tier> get tiers => Platform.isIOS ? iosTier : androidTier;
  static Set<String> get allIds => {...tiers.keys, ...lifetime, ...credits.keys};
}
```

### A.2 PurchaseService

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:crypto/crypto.dart';
import 'package:flutter/foundation.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:in_app_purchase_android/in_app_purchase_android.dart';

/// Your backend: verifies with Apple/Google, records idempotently, returns entitlements.
abstract class PurchaseBackend {
  Future<bool> verify({required String platform, required PurchaseDetails purchase, required String userId});
  Future<Entitlements> entitlements(String userId);
}

class Entitlements {
  const Entitlements({this.tier = Tier.free, this.addons = const {}, this.store, this.expiresAt, this.willRenew});
  final Tier tier;
  final Set<String> addons;
  final String? store;          // 'app_store' | 'play_store' — where to manage
  final DateTime? expiresAt;
  final bool? willRenew;
}

class PurchaseService extends ChangeNotifier {
  PurchaseService(this._backend);

  final PurchaseBackend _backend;
  final _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _sub;

  bool available = false;
  List<ProductDetails> products = [];
  Entitlements entitlements = const Entitlements();
  bool purchasing = false;
  bool pending = false;
  String? error;
  String? userId;

  /// Call once at app start (before showing UI that depends on purchases).
  Future<void> start() async {
    _sub = _iap.purchaseStream.listen(_onPurchases, onError: (Object e) {
      error = 'Purchase failed. Please try again.';
      notifyListeners();
    });
    available = await _iap.isAvailable();
    if (!available) { notifyListeners(); return; }

    final response = await _iap.queryProductDetails(Catalog.allIds);
    if (response.notFoundIDs.isNotEmpty) debugPrint('[IAP] Missing: ${response.notFoundIDs}');
    products = response.productDetails;
    await refresh();
  }

  Future<void> refresh() async {
    final id = userId;
    if (id != null) entitlements = await _backend.entitlements(id);
    notifyListeners();
  }

  /// Android: on resume, re-deliver purchases (unacknowledged ones get completed in _onPurchases).
  Future<void> onResume() async {
    if (Platform.isAndroid && available) await _iap.restorePurchases(applicationUserName: _accountToken());
    await refresh();
  }

  // MARK: Android offers

  /// in_app_purchase_android returns one GooglePlayProductDetails per offer — pick explicitly.
  GooglePlayProductDetails? bestAndroidOffer(String productId, String basePlanId) {
    final offers = products.whereType<GooglePlayProductDetails>().where((p) {
      final offer = p.subscriptionIndex == null ? null
          : p.productDetails.subscriptionOfferDetails![p.subscriptionIndex!];
      return p.id == productId && offer?.basePlanId == basePlanId;
    }).toList();
    final trial = offers.where((p) =>
        p.productDetails.subscriptionOfferDetails![p.subscriptionIndex!].offerId != null);
    return trial.isNotEmpty ? trial.first : (offers.isNotEmpty ? offers.first : null);
  }

  // MARK: Purchase

  Future<void> buy(ProductDetails product, {GooglePlayPurchaseDetails? replacing, ReplacementMode? mode}) async {
    if (purchasing || userId == null) return;
    purchasing = true; error = null; notifyListeners();

    final PurchaseParam param = Platform.isAndroid && replacing != null
        ? GooglePlayPurchaseParam(
            productDetails: product,
            applicationUserName: _accountToken(),
            changeSubscriptionParam: ChangeSubscriptionParam(
              oldPurchaseDetails: replacing,
              replacementMode: mode ?? ReplacementMode.chargeProratedPrice,
            ),
          )
        : PurchaseParam(productDetails: product, applicationUserName: _accountToken());

    try {
      if (Catalog.credits.containsKey(product.id)) {
        await _iap.buyConsumable(purchaseParam: param, autoConsume: false);   // consume after crediting
      } else {
        await _iap.buyNonConsumable(purchaseParam: param);                    // also used for subscriptions
      }
    } catch (_) {
      purchasing = false; error = 'Purchase failed. Please try again.'; notifyListeners();
    }
  }

  Future<void> restore() async {
    await _iap.restorePurchases(applicationUserName: _accountToken());
    await refresh();
    if (entitlements.tier == Tier.free) { error = 'No previous purchases found.'; notifyListeners(); }
  }

  // MARK: Stream handling — verify → grant → complete → refresh

  Future<void> _onPurchases(List<PurchaseDetails> purchases) async {
    for (final p in purchases) {
      var canComplete = false;
      switch (p.status) {
        case PurchaseStatus.pending:
          pending = true;                                   // Ask to Buy / slow payment — grant nothing
        case PurchaseStatus.canceled:                       // no alert
        case PurchaseStatus.error:
          if (p.status == PurchaseStatus.error) error = 'Purchase failed. Please try again.';
          canComplete = true;                               // nothing to grant
        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          canComplete = await _grant(p);                    // false → not completed → redelivered / re-queried
      }
      // Complete ONLY after a successful grant (Android: acknowledges; iOS: finishes).
      if (p.pendingCompletePurchase && canComplete) await _iap.completePurchase(p);
    }
    purchasing = false;
    await refresh();
  }

  Future<bool> _grant(PurchaseDetails p) async {
    final id = userId;
    if (id == null) return false;                           // retried after sign-in
    final ok = await _backend.verify(platform: Platform.isIOS ? 'ios' : 'android', purchase: p, userId: id);
    if (!ok) return false;
    if (Platform.isAndroid && Catalog.credits.containsKey(p.productID)) {
      // Credits were persisted by the backend during verify; now consume so it can be bought again.
      await _iap.getPlatformAddition<InAppPurchaseAndroidPlatformAddition>().consumePurchase(p);
    }
    pending = false;
    return true;
  }

  String? _accountToken() {
    final id = userId;
    if (id == null) return null;
    // Android: ≤ 64 chars, no PII. iOS: the platform package may need a UUID — check its docs.
    return sha256.convert(utf8.encode('app:$id')).toString();
  }

  @override
  void dispose() { _sub?.cancel(); super.dispose(); }
}
```

Keep `PurchaseService` in a top-level provider (Provider/Riverpod) so it lives for the whole app, and call `onResume()` from an `AppLifecycleListener` when the app returns to the foreground.

## Appendix B — Unified verification endpoint (Node/TypeScript)

```ts
import { AppStoreServerAPIClient, SignedDataVerifier, Environment } from "@apple/app-store-server-library";
import { google } from "googleapis";

const play = google.androidpublisher({ version: "v3",
  auth: new google.auth.GoogleAuth({ scopes: ["https://www.googleapis.com/auth/androidpublisher"] }) });

app.post("/iap/verify", requireUser, async (req, res) => {
  const { platform, productId, transactionId, purchaseToken } = req.body;
  const user = req.user;

  if (platform === "ios") {
    // SK2: fetch the signed transaction from Apple and verify it (don't trust client JWS blindly).
    const api = new AppStoreServerAPIClient(KEY, KEY_ID, ISSUER_ID, BUNDLE_ID, envFor(req));
    const tx = await verifier.verifyAndDecodeTransaction((await api.getTransactionInfo(transactionId)).signedTransactionInfo!);
    if (tx.bundleId !== BUNDLE_ID || tx.appAccountToken !== uuidFromUser(user.id)) return res.status(403).end();
    await db.purchases.upsert({ userId: user.id, store: "app_store", productId: tx.productId,
                                originalTransactionId: tx.originalTransactionId, expiresAt: tx.expiresDate,
                                revoked: !!tx.revocationDate });
  } else {
    const isSub = Catalog.androidSubs.has(productId);
    const data = isSub
      ? (await play.purchases.subscriptionsv2.get({ packageName: PACKAGE, token: purchaseToken })).data
      : (await play.purchases.products.get({ packageName: PACKAGE, productId, token: purchaseToken })).data;
    const account = isSub ? (data as any).externalAccountIdentifiers?.obfuscatedExternalAccountId
                          : (data as any).obfuscatedExternalAccountId;
    if (account !== hashUser(user.id)) return res.status(403).end();
    if (isSub && (data as any).linkedPurchaseToken) await db.purchases.retire((data as any).linkedPurchaseToken);
    await db.purchases.upsert({ userId: user.id, store: "play_store", productId, purchaseToken, raw: data });
  }

  res.json({ ok: true, entitlements: await computeEntitlements(user.id) });   // union across stores
});
```

Notifications from both stores (App Store Server Notifications V2, Play RTDN) update the same `purchases` table; `computeEntitlements` derives tier, add-ons, expiry and `store` from it.
