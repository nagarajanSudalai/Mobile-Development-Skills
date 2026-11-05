---
name: react-native-in-app-purchase
description: Use when implementing, reviewing, testing, or debugging in-app purchases or subscriptions in a React Native or Expo app using react-native-iap or expo-iap against the App Store and Google Play directly — shared product/entitlement model across stores, purchaseUpdatedListener/purchaseErrorListener or useIAP, finishTransaction, Android acknowledgement/consume, plan changes per platform (replacement modes), offers, restore, unified server verification (App Store Server API + Play Developer API), store policies, Expo development builds, testing. For RevenueCat in React Native use react-native-revenuecat-iap; for Flutter use flutter-in-app-purchase; for store-specific detail in native apps see ios-in-app-purchase and android-in-app-purchase.
---

# React Native In-App Purchases (`react-native-iap` / `expo-iap`)

Guide for building, auditing, and testing purchases in **React Native / Expo** apps (`react-native-iap`, `expo-iap`) that use the **App Store** and **Google Play** directly (no RevenueCat or other purchase backend). One shared TypeScript layer drives two different stores, so the hard part is keeping a **single entitlement model** while respecting each store's rules.

Related skills — this skill links to them instead of repeating their content:
- **ios-in-app-purchase** — StoreKit rules, subscription groups, App Store Connect, App Review, post-purchase UI (§13)
- **android-in-app-purchase** — Play Billing rules, base plans & offers, replacement modes, acknowledgement, RTDN, Play policy
- **react-native-revenuecat-iap** — the same app built on RevenueCat instead of direct store APIs
- **flutter-in-app-purchase** — the Flutter equivalent (`in_app_purchase`)

Plugin APIs change across major versions (`react-native-iap` v14 reworked its API around the OpenIAP spec; `expo-iap` follows the same spec). Verify every call against the version in `package.json` before shipping.

## 0. Architecture (mental model)

```
+-------------------------------------------------------------+
| Shared UI (React components)                                |
| Paywall, Plan status, Change plan, Restore, Manage          |
+-------------------------------------------------------------+
            ^ state (Redux / Zustand / React Query / hooks)
+-------------------------------------------------------------+
| PurchaseService (singleton, started at app launch)          |
| - one purchase listener for the whole app lifetime          |
| - shared catalog: store product IDs → entitlement           |
| - verify (server) → grant → finish → refresh                |
| - platform branches: iOS appAccountToken / Android offers,  |
|   obfuscatedAccountId, replacement modes, consume           |
+-------------------------------------------------------------+
     | react-native-iap / expo-iap
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
| Proof of purchase | JWS transaction (SK2) / app receipt (SK1) | Purchase token | `transactionReceipt` / JWS / `purchaseToken` |
| Finish | `finish()` (redelivered until finished) | **Acknowledge** within 3 days or refunded; consume for consumables | `finishTransaction` after grant |
| User link | `appAccountToken` (UUID) | `obfuscatedAccountId` (≤ 64 chars) | `accountToken(userId)` per platform |
| Restore | `AppStore.sync()` / restore | Automatic via query | `restore()` |
| Manage | `showManageSubscriptions` / App Store URL | Play subscriptions deep link | `openManage()` |

## 1. First, establish context

Find out (from the codebase or by asking):

- Library with **exact versions**: `react-native-iap` or `expo-iap`; Expo managed vs bare (Expo needs a **development build** with the config plugin — IAP doesn't work in Expo Go)
- Target stores (App Store, Play; also macOS? Amazon?) and minimum OS versions (StoreKit 2 needs iOS 15+)
- Product catalog by type per store (consumable / non-consumable / subscriptions with groups or base plans / add-ons)
- Backend? Strongly recommended — without it you can't know subscription expiry on Android or verify purchases reliably on either store
- Account system and whether entitlements must work across platforms (bought on iOS, used on Android/web)
- State management in use (Redux/Zustand/React Query) — the service should plug into it, not replace it

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
- [ ] iOS: **In-App Purchase** capability in the Xcode project (`ios/<App>`); Expo: config plugin + `expo prebuild` / EAS build
- [ ] Android: billing permission comes from the Play Billing dependency; `minSdk` meets the library's requirement
- [ ] Bundle ID / applicationId identical to the store listings; Play: upload a signed build once so products can be queried
- [ ] Backend has App Store Server API credentials (In-App Purchase key) and a Play service account; App Store Server Notifications V2 and Play RTDN point to it

## 3. Service lifecycle

- [ ] **One** `PurchaseService` created at app start (in the root component / store init), not inside the paywall
- [ ] Subscribe to purchase events **immediately**: `purchaseUpdatedListener` + `purchaseErrorListener` (or `useIAP` callbacks at the app root). This catches renewals, Ask-to-Buy approvals, pending Play purchases, promo codes, and transactions left over from a crash
- [ ] Keep the listeners alive for the app's lifetime; remove them only on app teardown (`remove()` + `endConnection()`)
- [ ] Check availability (`initConnection()`); show "store unavailable" if false (no Play Store, restricted device, emulator without Play)
- [ ] Query products once per session (`getProducts` + `getSubscriptions`, or `fetchProducts` in v14 / expo-iap); log missing IDs and show the rest
- [ ] Re-sync on app resume: Android re-query purchases (catches out-of-app purchases and missed acknowledgements); refresh entitlements from the server on both platforms

## 4. Purchasing — shared rules

- [ ] Purchase only from a user action; button disabled while in flight
- [ ] Link purchases to your user: iOS `appAccountToken` (UUID derived from the user ID); Android `obfuscatedAccountId` (hash ≤ 64 chars). Pass `appAccountToken` / `obfuscatedAccountIdAndroid`
- [ ] Handle every status: **purchased**, **restored** (iOS), **pending** (Ask to Buy / slow payment — show "waiting", grant nothing), **canceled** (no alert), **error** (short retryable message)
- [ ] Grant only after **server verification** (send `transactionReceipt` / JWS / `purchaseToken` + platform + product ID), idempotent by transaction ID / purchase token
- [ ] **Finish only after granting**: `finishTransaction({ purchase, isConsumable })`. On Android this acknowledges (or consumes) — skipping it means an automatic refund after 3 days; on iOS unfinished transactions are redelivered
- [ ] Consumables: `finishTransaction({ isConsumable: true })` after crediting
- [ ] Never auto-finish transactions before verification (`andDangerouslyFinishTransactionAutomaticallyIOS` must stay false)
- [ ] Gate the UI on the **refreshed entitlements** (server or recomputed), not on "purchase call returned"

## 5. Platform branches

### 5.1 iOS
- [ ] Prefer **StoreKit 2** (react-native-iap/expo-iap use SK2 on recent versions) — verify for your version
- [ ] Server verifies the **JWS** (SK2) or the app receipt (SK1) with the App Store Server API (ios-in-app-purchase §7)
- [ ] Plan changes inside a subscription group are a normal purchase of the other product — don't block "already subscribed"; downgrades apply at renewal (ios-in-app-purchase §3.2)
- [ ] Offer code sheet: `presentCodeRedemptionSheetIOS()` — redemption arrives via the purchase listener
- [ ] Price-increase consent and promotional offers need platform-specific APIs (promotional offers require a server signature)
- [ ] Restore button required (`getAvailablePurchases()`) — restored transactions also need finishing

### 5.2 Android
- [ ] Subscriptions carry **offers**: products expose `subscriptionOfferDetails` — **choose the offer explicitly** (eligible trial/intro, else base plan offer) and pass its token
- [ ] Tier change **must** pass the old purchase and a replacement mode — `requestSubscription({ purchaseTokenAndroid, replacementModeAndroid, subscriptionOffers })` (names vary by version). Otherwise the user is billed for two subscriptions (android-in-app-purchase §3.2)
- [ ] Each plan change creates a **new purchase token** → verify + finish it
- [ ] Pending purchases (`purchaseStateAndroid === PENDING`) → no access until purchased
- [ ] Re-query purchases on resume (`getAvailablePurchases`) and finish anything unacknowledged
- [ ] The client has no expiry, base plan, or grace/hold state — get them from the server (subscriptionsv2)

## 6. Entitlements & restore

- [ ] **Server is the source of truth** for both platforms: `GET /entitlements` returns tier, add-ons, expiry, renewal state, store, pending change
- [ ] Cache the last server response for offline use; re-verify when online
- [ ] Without a server: iOS can derive state from StoreKit (library APIs such as active subscriptions are limited — verify for your version or add a small native module); Android can only tell "active or not" from the purchase list
- [ ] Cross-platform: a user who subscribed on iOS gets access on Android via the server entitlement, but **manages** the subscription on the store they paid on — show "Manage on your iPhone / in Google Play" based on `store`
- [ ] Restore button → platform restore + server re-link; nothing found → "No previous purchases found" (informational)
- [ ] Don't sell a second subscription on platform B to a user already subscribed on platform A (check the server entitlement before showing buy buttons)

## 7. Offers & pricing

- [ ] Prices from product details (`localizedPrice` / `displayPrice`) — never hardcoded; Android subscription price = the offer's final (recurring) pricing phase
- [ ] iOS intro eligibility per group (library or server); Android shows only offers Play returns for the user
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

1. **iOS**: Xcode StoreKit Configuration file — open `ios/<App>.xcworkspace`, set it in the scheme, run from Xcode; then sandbox account on a device; TestFlight. Expo: use a development build (not Expo Go)
2. **Android**: license testers, internal testing track, test cards (approve / decline / slow), accelerated renewals, Play Billing Lab
3. **TS unit tests** with a fake store adapter (Appendix) — no native module in tests
4. **Server**: sandbox notifications from both stores

### Scenario matrix

| # | Scenario | iOS | Android |
|---|---|---|---|
| X1 | Store unavailable | Error state | No Play / emulator → error state |
| X2 | Products partly missing | Valid ones shown; IDs logged | Same |
| X3 | Buy subscription with trial | Intro shown only if eligible | Trial offer token passed |
| X4 | Cancel the sheet | No alert | No alert |
| X5 | Pending (Ask to Buy / slow card) | "Waiting"; unlock via listener | Same |
| X6 | App killed before finish | Redelivered on launch; granted once | Re-query on launch; acknowledged within 3 days |
| X7 | Verification fails | Not finished; retried | Not acknowledged; retried before 3 days |
| X8 | Upgrade | Same-group purchase, immediate | `purchaseTokenAndroid` + CHARGE_PRORATED_PRICE |
| X9 | Downgrade | Scheduled at renewal | DEFERRED; scheduled |
| X10 | Consumable | Credited once; finished | Credited once; consumed |
| X11 | Restore / reinstall | Restore → granted | Re-query → granted |
| X12 | Subscribed on iOS, open Android app | Server entitlement → access; "Manage on iPhone" | — |
| X13 | Refund | Access removed via server | Same |
| X14 | Expiry / renewal (accelerated) | Server updates; UI refreshes on resume | Same |
| X15 | Offer code / promo code | Arrives via listener | Re-query on resume |
| X16 | Sign out / sign in as another user | Purchases not granted to the wrong account | Same |
| X17 | Offline launch | Cached entitlements | Same |
| X18 | Post-purchase UI states | ios-in-app-purchase §13.6 | android-in-app-purchase §13 |

## 11. Common bugs to look for in review

- Purchase listener created in the paywall screen → renewals, Ask-to-Buy approvals, leftover transactions missed; or several listeners → duplicate grants
- `finishTransaction` never called (Android refunds after 3 days; iOS redelivers forever) — or called before verification/granting
- Restored iOS transactions not finished
- Granting on `pending`; error alert on user cancel
- `andDangerouslyFinishTransactionAutomaticallyIOS: true`
- Android tier change without old purchase + replacement mode → double billing
- Android offer picked by index (`subscriptionOfferDetails[0]`) or no `offerToken` → wrong price / developer error
- Trusting the receipt on the device instead of server verification
- Gating on product IDs / substrings; same entitlement logic duplicated per platform and drifting
- No server → Android "active forever" (no expiry) or iOS expiry ignored
- Selling a second subscription on Android to a user already subscribed on iOS
- Expo Go used for testing IAP (doesn't work) / missing config plugin; StoreKit config not selected because the app was run from the CLI
- Library upgraded across a major version without migrating the API (e.g. react-native-iap v14)
- Hardcoded prices; iOS-only disclosure text shown on Android or vice versa
- No Restore button on iOS; missing Terms/Privacy links

## 12. Output format

When asked to **review/audit**:
1. **Summary** — library + versions, Expo managed/bare, platforms, StoreKit mode, backend yes/no, cross-store entitlement map
2. **Checklist results** — sections 2–9 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android** where relevant (console items listed for the user to confirm)
3. **Critical issues first** (refunds from missing acknowledgement, lost or double-granted purchases, double billing, cross-platform double subscriptions, policy risks) with fix code
4. **Testing gaps** — X1–X18 not covered, per platform

When asked to **implement**: start from Appendix A, adapt the catalog and API names to your library version, remove unused parts, then list App Store Connect, Play Console, backend, and policy steps.

---

## Appendix A — React Native (`react-native-iap` v12/13 API)

`react-native-iap` v14 and `expo-iap` renamed most of these (`fetchProducts`, unified `requestPurchase({ request: { ios, android } })`, `useIAP`). Keep the same flow and update names to your version.

### A.1 Catalog

```ts
import { Platform } from "react-native";

export type Tier = "free" | "plus" | "pro";

const iosTier: Record<string, Tier> = {
  "com.app.plus.monthly": "plus", "com.app.plus.yearly": "plus",
  "com.app.pro.monthly": "pro", "com.app.pro.yearly": "pro",
};
const androidTier: Record<string, Tier> = { plus: "plus", pro: "pro" };

export const Catalog = {
  tiers: Platform.OS === "ios" ? iosTier : androidTier,
  inApp: Platform.select({ ios: ["com.app.lifetime", "com.app.credits.100"], default: ["lifetime", "credits_100"] })!,
  consumables: new Set(["com.app.credits.100", "credits_100"]),
  get subs() { return Object.keys(this.tiers); },
};
```

### A.2 PurchaseService

```ts
import * as IAP from "react-native-iap";
import { Platform, EmitterSubscription } from "react-native";
import { sha256 } from "js-sha256";

type Backend = {
  verify(args: { platform: "ios" | "android"; purchase: IAP.Purchase; userId: string }): Promise<boolean>;
  entitlements(userId: string): Promise<{ tier: Tier; store?: string; expiresAt?: string }>;
};

export class PurchaseService {
  private updates?: EmitterSubscription;
  private errors?: EmitterSubscription;
  private inFlight = new Set<string>();
  userId?: string;
  onChange?: () => void;
  state = { available: false, products: [] as IAP.Product[], subscriptions: [] as IAP.Subscription[],
            tier: "free" as Tier, pending: false, error: undefined as string | undefined };

  constructor(private backend: Backend) {}

  /** Call once at app start (root component / store init). */
  async start() {
    this.state.available = await IAP.initConnection().catch(() => false);
    if (!this.state.available) return this.emit();
    if (Platform.OS === "android") await IAP.flushFailedPurchasesCachedAsPendingAndroid().catch(() => {});

    this.updates = IAP.purchaseUpdatedListener((p) => void this.process(p));
    this.errors = IAP.purchaseErrorListener((e) => {
      if (e.code !== IAP.ErrorCode.E_USER_CANCELLED) this.state.error = "Purchase failed. Please try again.";
      this.emit();
    });

    this.state.subscriptions = await IAP.getSubscriptions({ skus: Catalog.subs });
    this.state.products = await IAP.getProducts({ skus: Catalog.inApp });
    await this.refresh();
  }

  stop() { this.updates?.remove(); this.errors?.remove(); IAP.endConnection(); }

  /** Android: on resume, complete anything unacknowledged and refresh. */
  async onResume() {
    if (Platform.OS === "android") (await IAP.getAvailablePurchases()).forEach((p) => void this.process(p));
    await this.refresh();
  }

  private accountId() { return this.userId ? sha256(`app:${this.userId}`) : undefined; }   // 64 hex chars

  /** Android: choose an offer explicitly (eligible trial, else base plan). */
  androidOffer(sub: IAP.SubscriptionAndroid, basePlanId: string) {
    const offers = sub.subscriptionOfferDetails.filter((o) => o.basePlanId === basePlanId);
    return offers.find((o) => o.offerId) ?? offers.find((o) => !o.offerId);
  }

  async buySubscription(sku: string, opts: { offerToken?: string; replacingToken?: string; replacementMode?: number } = {}) {
    if (!this.userId) return;
    await IAP.requestSubscription({
      sku,
      appAccountToken: Platform.OS === "ios" ? uuidFromUser(this.userId) : undefined,
      obfuscatedAccountIdAndroid: this.accountId(),
      ...(opts.offerToken && { subscriptionOffers: [{ sku, offerToken: opts.offerToken }] }),
      ...(opts.replacingToken && {                     // Android tier change — never omit
        purchaseTokenAndroid: opts.replacingToken,
        replacementModeAndroid: opts.replacementMode ?? IAP.ReplacementModesAndroid.CHARGE_PRORATED_PRICE,
      }),
    });
  }

  async buyProduct(sku: string) {
    if (!this.userId) return;
    await IAP.requestPurchase({
      sku, skus: [sku],
      andDangerouslyFinishTransactionAutomaticallyIOS: false,   // never auto-finish
      obfuscatedAccountIdAndroid: this.accountId(),
    });
  }

  async restore() {
    const purchases = await IAP.getAvailablePurchases();
    for (const p of purchases) await this.process(p);
    await this.refresh();
    if (this.state.tier === "free") { this.state.error = "No previous purchases found."; this.emit(); }
  }

  // verify → grant → finish → refresh
  private async process(p: IAP.Purchase) {
    const key = p.purchaseToken ?? p.transactionId ?? "";
    if (!this.userId || this.inFlight.has(key)) return;
    if (Platform.OS === "android" && p.purchaseStateAndroid === IAP.PurchaseStateAndroid.PENDING) {
      this.state.pending = true; return this.emit();
    }
    this.inFlight.add(key);
    try {
      const ok = await this.backend.verify({ platform: Platform.OS as "ios" | "android", purchase: p, userId: this.userId });
      if (!ok) return;                                    // not finished → redelivered / re-queried
      await IAP.finishTransaction({ purchase: p, isConsumable: Catalog.consumables.has(p.productId) });
      this.state.pending = false;
    } finally {
      this.inFlight.delete(key);
      await this.refresh();
    }
  }

  async refresh() {
    if (this.userId) this.state.tier = (await this.backend.entitlements(this.userId)).tier;
    this.emit();
  }

  private emit() { this.onChange?.(); }
}
```

`uuidFromUser` is a deterministic UUID derived from the user ID (SHA-256 → 16 bytes with version/variant bits set; the same function runs on the server to check `appAccountToken`). Expose `PurchaseService` through your state library; keep one instance for the app lifetime.

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
