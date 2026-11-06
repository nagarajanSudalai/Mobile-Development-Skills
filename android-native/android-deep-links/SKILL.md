---
name: android-deep-links
description: Use when implementing, reviewing, testing, or debugging deep links on Android — verified Android App Links (intent filters with android:autoVerify, assetlinks.json, SHA-256 fingerprints incl. the Play App Signing key, Android 12+ verification behaviour, DomainVerificationManager), custom URI schemes, Jetpack Navigation (Compose) deepLinks / type-safe navDeepLink, onNewIntent handling for warm starts, synthetic back stacks with TaskStackBuilder / NavDeepLinkBuilder, pending links across login and onboarding, multi-account link resolution, deferred deep links via the Play Install Referrer API (vendor-neutral), migration off Firebase Dynamic Links, push/widget/shortcut entry points, security (open redirect, scheme hijacking, intent validation) and adb-based testing. Siblings: ios-deep-links, flutter-deep-links, react-native-deep-links; related: android-multi-account, android-push-notifications, android-in-app-purchase, android-revenuecat-iap, android-list-ux.
---

# Android Deep Links Skill

Guide for building, auditing, and testing **deep links** in consumer Android apps: verified **Android App Links** (`https://` links that open the app without a chooser), **custom schemes** (`myapp://`), and every other "open the app at a specific place" entry point — push taps, widgets, app shortcuts, QR codes, OAuth redirects and **deferred deep links** (link clicked before the app was installed).

Sibling skills: **ios-deep-links**, **flutter-deep-links**, **react-native-deep-links** — same structure per platform; keep URL design and routing behaviour identical across platforms. Related: **android-multi-account** (link targets another account/portal), **android-push-notifications** (notification tap → route), **android-in-app-purchase** / **android-revenuecat-iap** (paywall/offer links), **android-list-ux** (link into a list item that isn't loaded yet).

> Link verification, Navigation and the Play Console change often. Several things below are version-gated or recently changed: per-host verification and "no chooser for unverified web links" (**Android 12 / API 31**), `pathAdvancedPattern` (API 31), `DomainVerificationManager` (API 31), stricter intent-filter matching for external intents (**Android 13 / targetSdk 33**), dynamic App Links rules in `assetlinks.json` (**Android 15+**, rolled out via Google Play services), type-safe `navDeepLink<T>()` (**Navigation 2.8+**), and **Firebase Dynamic Links shut down on 25 Aug 2025**. **Verify against the project's `targetSdk`, Navigation version in `libs.versions.toml`, and current developer.android.com docs before writing code.**

## 0. Mental model

```
   Link sources                        Android system                          App
 ┌───────────────┐   tap   ┌─────────────────────────────────┐    ┌──────────────────────────────────┐
 │ browser, mail │ ──────▶ │ Intent(ACTION_VIEW, data=https:)│    │ MainActivity (singleTop/          │
 │ chat, social  │         │  ├ verified App Link for host?  │──▶ │   exported, single-activity)      │
 │ QR, ads, SMS  │         │  │   yes → app directly         │    │  cold:  onCreate(intent)          │
 └───────────────┘         │  │   no  → browser (API 31+)    │    │  warm:  onNewIntent(intent)       │
 ┌───────────────┐         │  └ custom scheme → app / chooser│    │        │                          │
 │ push, widget, │ ──────▶ │ PendingIntent (explicit)        │──▶ │  DeepLinkParser: Uri → DeepLink   │
 │ shortcut      │         └─────────────────────────────────┘    │        │ (validated, typed)       │
 └───────────────┘                    ▲                           │  DeepLinkRouter                   │
                                      │ verifies at install/      │   ├ needs auth/onboarding?        │
            https://example.com/.well-known/assetlinks.json       │   │   → PendingLinkStore, resume  │
            (package_name + SHA-256 of signing cert)              │   ├ other account/portal?         │
                                                                  │   │   → android-multi-account     │
 ┌───────────────┐  Play Store install   ┌────────────────────┐   │   └ NavController: push/replace,  │
 │ link, app not │ ────────────────────▶ │ Install Referrer   │──▶│      synthesize back stack        │
 │ installed     │  &referrer=<token>    │ API (first launch) │   └──────────────────────────────────┘
 └───────────────┘                       └────────────────────┘
```

One rule carries most of the design: **every entry point produces a `Uri` (or typed `DeepLink`), and one parser + one router handle all of them.** Push payloads, widgets, shortcuts, QR scans, deferred links and real `https` links must not each have their own navigation code.

| Launch situation | Callback | Typical bug |
|---|---|---|
| **Cold** (process not running) | `onCreate` with `intent.data` | Link handled before DI/session ready; handled again after rotation |
| **Warm** (process alive, activity in back stack, `singleTop`/`singleTask`) | `onNewIntent(intent)` | Ignored entirely — app comes to front on old screen |
| **Foreground** (user inside app, taps link in a WebView/notification) | `onNewIntent` | Destination pushed on top of a modal / half-finished checkout |
| **Restored from Recents / history** | `onCreate` with the *old* intent + `FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY` | Stale link replayed days later |
| **Process death + restore** | `onCreate(savedInstanceState != null)` | Link re-processed on restore |

## 1. First, establish context

Find out (from the codebase or by asking):

- **Link types in use**: verified App Links (`https`), custom scheme (`myapp://`), both? Any links from a former vendor/Firebase Dynamic Links (`*.page.link`, short links on a vendor domain) still in the wild?
- **Domains**: which hosts (`example.com`, `www.example.com`, `m.example.com`, `links.example.com`, regional ccTLDs)? Who controls the web server / CDN for each? Is there a website at those URLs for people without the app?; **Build variants**: package names per flavor (`com.example.app`, `.debug`, `.staging`) and which signing keys exist (debug, upload, **Play App Signing** key, any third-party store key).
- **Router**: Navigation Compose (string routes or type-safe `@Serializable` routes, Navigation 2.8+), Navigation with Fragments, a custom back stack, or multiple activities? Single-activity?; **Destinations**: list every linkable screen and whether it requires **login**, **onboarding/consent**, **a specific account or portal**, an **entitlement** (subscription), or a **feature flag**.
- **Logged-out behaviour**: show a public preview, require login then continue, or open the Play Store listing / web page?; **Multi-account**: can several accounts or portals be signed in (see **android-multi-account**)? Do links carry an account/portal identifier?
- **Deferred deep links**: needed? (Install campaigns, referral invites, "open this item after install".) Is there an existing attribution SDK?; **Other entry points**: push (FCM data/notification payloads), widgets (Glance/AppWidget), app shortcuts (static/dynamic/pinned), QR codes, NFC, OAuth/SSO redirects, email campaigns with click tracking, App Actions/Assistant.
- **minSdk / targetSdk** — verification behaviour differs before and after API 31.; **Analytics**: what attribution is expected (UTM, campaign id, referral code)?

Write a **link contract** before coding:

```
pattern                              dest            auth      account-aware  back stack        fallback (web)
https://example.com/p/{productId}    ProductDetail   public    no             Home > Product    /p/{id} page
https://example.com/orders/{id}      OrderDetail     login     yes (acct)     Home > Orders > X /orders → login
https://example.com/w/{portal}/t/{t} TaskDetail      login     yes (portal)   Home > Tasks > X  web app
https://example.com/invite/{code}    InviteAccept    either    yes            Home > Invite     /invite page
https://example.com/offer/{offerId}  Paywall         login     yes            Home > Paywall    /offer page
myapp://oauth/callback               (auth result)   n/a       n/a            none              n/a
```

## 2. Link types: App Links vs custom schemes

| | Verified Android App Links (`https`) | Custom scheme (`myapp://`) | Unverified `https` intent filter |
|---|---|---|---|
| Opens app without chooser | Yes (once verified) | Yes if only one app claims it; otherwise chooser | **API 31+: no — opens browser.** Pre-31: chooser |
| Works when app not installed | Yes — the web page loads | No — dead link / error | Web page loads |
| Ownership proof | `assetlinks.json` on your domain | None — any app can claim the scheme | None |
| Shareable / clickable everywhere | Yes (chat, email, social) | Often not linkified; blocked by many apps | Yes |
| Use for | All user-facing links, marketing, sharing, email, QR | OAuth redirect (if not using https), internal widget/shortcut URIs, dev tooling | Nothing — make it verified or remove it |

Rules:

- [ ] **Default to verified App Links** for anything a human might share, click, or scan.
- [ ] Custom schemes only for app-internal URIs (widgets, shortcuts, push) or OAuth redirects where the IdP doesn't support https redirects. Treat their params as **untrusted** — any app can fire them.
- [ ] Use a **reverse-DNS-ish, unique scheme** (`com.example.app://` or `examplecom://`), never generic (`app://`, `myapp://` in production).
- [ ] Don't rely on `intent://` URLs from web pages; Chrome supports them, many browsers don't, and they're a spoofing surface.
- [ ] Every App Link path must also **work on the web** (real page, or a smart landing page with a Play Store button). The app being uninstalled is the most common case for a link click.

## 3. Domain verification (`assetlinks.json`)

### 3.1 The file

Served at `https://<host>/.well-known/assetlinks.json` **for every host** in your `autoVerify` intent filters: a JSON array of `delegate_permission/common.handle_all_urls` statements with `namespace: android_app`, `package_name` and `sha256_cert_fingerprints` (full example in Appendix A.2).

### 3.2 Hosting rules

- [ ] Served over **HTTPS** with a valid, publicly trusted certificate (no self-signed, full chain). `Content-Type: application/json`.
- [ ] **No redirects** (no 301/302, not even http → https or apex → www). The verifier does not follow them for this file.
- [ ] Publicly reachable: **no auth, no cookie walls, no bot protection / WAF challenge, no geo-blocking** of Google's verifier. Check bot-management rules on the CDN — they are the #1 silent failure.
- [ ] Return **200** with the file for **each host** separately: `example.com` and `www.example.com` are different hosts. Wildcard filter `*.example.com` → the file must be on the apex `example.com` (and each subdomain you actually use is safest). Valid JSON (no trailing commas, no comments, no BOM). Keep it small.
- [ ] **CDN caching**: set a short `Cache-Control` (e.g. ≤ 1 hour) while iterating; purge after changing fingerprints. Devices re-verify on install/update, and Google's side caches statements too — expect delays. Don't serve it from a SPA catch-all route that returns `index.html` with 200. Version it with your infra (Terraform/CDN config), not hand-edited.

### 3.3 Fingerprints

- [ ] Include the **SHA-256 of the Play App Signing key** (Play Console → Test and release → App integrity → App signing; the console also shows a ready-made Digital Asset Links JSON). With Play App Signing, **the key on users' devices is Google's, not your upload key** — the most common "works in debug/internal, broken in production" cause.
- [ ] Also include the **upload key** (internal app sharing / locally installed release builds) and optionally the **debug key** for dev (or host a separate statement for `com.example.app.debug`). One statement per **package name** (flavors with `applicationIdSuffix` need their own entry). Get local fingerprints: `./gradlew signingReport` or `keytool -list -v -keystore release.jks -alias upload`. Key rotation (Play signing key upgrade): add the new fingerprint **before** rollout; keep old until no installs use it. Other stores (Samsung, Huawei, sideload) re-sign with their own keys → add those fingerprints or accept that links won't verify there.

### 3.4 Validation

- [ ] Google Digital Asset Links API: `https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://example.com&relation=delegate_permission/common.handle_all_urls`.
- [ ] Play Console **Deep links** page (Grow users → Deep links) shows per-domain verification status and path issues for the published app. Android Studio **App Links Assistant** (Tools → App Links Assistant) to generate filters and test.
- [ ] On device: `adb shell pm get-app-links com.example.app` → each host should show `verified`. See §14.
- [ ] Add a **CI/monitoring check** that fetches each host's `assetlinks.json`, asserts 200, content-type, no redirect, valid JSON, and the expected fingerprint. A marketing-site redeploy that drops the file silently breaks links for new installs.

### 3.5 Android 12+ verification behaviour

- **API 31+**: an `https` intent filter that isn't verified **does not open the app** from a link — the default browser handles it. No chooser. Users can manually enable hosts in *Settings → Apps → App → Open by default → Add link*.
- **Per-host verification** (API 31+): each host in your filters is verified independently — one broken host no longer breaks the others. **Pre-31: all hosts across all `autoVerify` filters had to verify or none did.** With `minSdk < 31`, one misconfigured host still breaks every host on older devices.
- Verification runs **at install and update** (asynchronously, needs network). A device offline at install may stay unverified until re-verification (periodic/on update). `pm verify-app-links --re-verify` forces it for testing.
- `DomainVerificationManager.getDomainVerificationUserState(packageName)` (API 31+) lets the app check whether its hosts are verified/selected and, if not, offer to open `Settings.ACTION_APP_OPEN_BY_DEFAULT_SETTINGS`. Use sparingly — only in a relevant moment (e.g. a settings row "Open links in app"), never a launch-time nag.
- **Dynamic App Links (Android 15+)**: `assetlinks.json` can include `relation_extensions` with `dynamic_app_link_components` to include/exclude paths/query/fragment server-side without an app update. Treat as an enhancement — the manifest filter must still match, and older devices ignore it. Verify current schema before using.

## 4. App setup (Android)

### 4.1 Manifest

- [ ] Single entry activity (`MainActivity`) with `android:exported="true"` and `android:launchMode="singleTop"` (or `singleTask` if you must guarantee one instance; understand it changes task behaviour). `standard` launch mode + links = duplicate activities stacked on top of each other.
- [ ] Separate `<intent-filter android:autoVerify="true">` for https links: `ACTION_VIEW`, categories `DEFAULT` **and** `BROWSABLE`, `scheme="https"` (add `http` only if you really have http links — it must also verify), `host`, and path elements.
- [ ] **`<data>` elements in one filter are combined as a cross-product** (every scheme × every host × every path). Split unrelated hosts/paths into separate filters so you don't accidentally claim `www.example.com/anything` or a custom scheme with an https host.
- [ ] Prefer **`pathPrefix`** / `pathPattern` / `pathAdvancedPattern` (API 31+) over claiming the whole host. Claim only paths the app can render; everything else (blog, help, legal, checkout on web) must stay in the browser. Custom scheme filter in a **separate** filter, without `autoVerify`. Don't put link filters on non-launcher activities you'd then need to coordinate; route everything through the single activity. targetSdk 33+: intents from other apps must match a declared filter exactly (action/category/data), so explicitly declare everything you expect.

### 4.2 Gradle / Navigation

- [ ] Navigation Compose 2.8+ for type-safe routes and `navDeepLink<Route>(basePath = "https://example.com/p")`; string-route `navDeepLink { uriPattern = "https://example.com/p/{productId}" }` on older versions.
- [ ] Manifest filters are **not generated** from Compose `deepLinks` (unlike XML nav graphs with `<nav-graph>` in the manifest). Keep manifest filters and nav-graph patterns in sync — ideally test them against the same fixture list (Appendix A.4).
- [ ] Decide whether Navigation handles links directly (`NavController.handleDeepLink`) or your own `DeepLinkParser` maps to routes. **Recommended for consumer apps: own parser + router**, because auth gating, pending links, account switching and analytics don't fit Navigation's implicit handling. Navigation's built-in deep link matching is fine for simple public-only apps.

### 4.3 Activity handling

- [ ] In `onCreate`: handle `intent` **only when `savedInstanceState == null`** and not `FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY`.
- [ ] Override `onNewIntent(intent)` (or `addOnNewIntentListener` in `ComponentActivity`), call `setIntent(intent)` and route it. Mark an intent consumed (e.g. remove data / set an extra) after handling so configuration changes and `recreate()` don't replay it. Route **after** the app is ready (session restored, DI graph built, nav graph set) — buffer the link in the ViewModel until then. Don't block the splash screen on network for routing. If you use `installSplashScreen().setKeepOnScreenCondition`, cap the wait; a link must never freeze the splash.

## 5. URL design

- [ ] **Readable, stable, resource-oriented** paths: `/p/{productId}`, `/orders/{orderId}`, `/w/{portalSlug}/t/{taskId}`. Mirror the website's URLs so one link works on web and in app. IDs are **opaque strings**, not positional or row indexes. Never encode screen names (`/screen/ProductDetailFragment`).
- [ ] Put **identity in the path**, options in the query (`?tab=reviews&variant=red`). Unknown query params are ignored, never errors. Analytics params (`utm_*`, `cid`, `ref`) are **stripped before routing** and sent to analytics separately.
- [ ] **Versioning**: avoid `/v1/` in public links; instead keep old paths working forever with server and app-side redirects. If you must, an optional `v=` query lets the parser branch.
- [ ] **Encoding**: use `Uri.getPathSegments()` / `getQueryParameter()` (decoded) — never split raw strings. Percent-encode when building links (`Uri.Builder().appendPath()`). Test unicode, `+`, spaces, `%2F` in IDs. Case: hosts are case-insensitive, paths are case-sensitive; normalize host and trailing slash before matching.
- [ ] **Stable marketing links**: campaigns get a short path on your own domain (`example.com/go/spring`) that the server resolves to a canonical link; the app can resolve the same alias via an API. Never print a link you can't redirect later (QR codes on packaging live for years).
- [ ] Account/portal-aware links carry an explicit identifier (`/w/{portalSlug}/...` or `?account=<hint>`) — never a token or email. Keep a **link registry** (the contract table) shared by app, web and marketing; links are an API.

## 6. Routing

### 6.1 Parse → typed model → route

- [ ] `DeepLinkParser.parse(uri): DeepLink?` — pure Kotlin, no Android framework beyond `Uri` (or use `java.net.URI` for JVM tests), returns a **sealed type** (`DeepLink.Product(id)`, `DeepLink.Order(id)`, `DeepLink.Unknown(uri)`). Validate each param (regex/length/charset) in the parser; invalid → `Unknown`, never a crash. Accept every **host alias** (`example.com`, `www.`, `m.`) and the custom scheme equivalents in one place. Router maps `DeepLink` → navigation action, given current state (auth, account, current back stack).

### 6.2 Stack rules

| Link | Rule |
|---|---|
| Top-level destination (Home tab, Orders tab) | Switch tab / `popUpTo(start)`, `launchSingleTop = true`, `restoreState` per your bottom-nav setup |
| Detail (product, order) | **Synthesize parent**: Home → Orders → Order X so Up/back is sensible; use `popUpTo(graph.start)` then navigate parent then child |
| Same destination already on top with same args | No-op or refresh (`launchSingleTop`); with different args → replace, not stack 10 identical screens |
| Flows (checkout, onboarding, editor with unsaved changes) | Don't destroy silently: confirm or queue the link until the flow ends |
| Modal / bottom sheet / dialog showing | Dismiss it first (unless it's a blocking flow) then navigate |
| Auth/result links (OAuth callback, email verification) | Deliver result to the waiting flow; no navigation push |

- [ ] **External (cold or warm from another app) links**: build a **predictable stack** (start + parents + target); back from the target goes to the parent, not out of the app — unless product decides "back returns to the referrer" (then don't synthesize, and set nothing special; system back finishes the task back to the caller only if the activity was launched into the caller's task).
- [ ] Navigation's own `handleDeepLink`: if the intent has `FLAG_ACTIVITY_NEW_TASK`, it **clears the task and builds the synthetic back stack** from the graph's start destinations; without it, it navigates within the current stack. Know which you're getting. Use `TaskStackBuilder` / `NavDeepLinkBuilder` for **PendingIntents** (push, widgets) so the system creates a correct task when the app isn't running (§10, Appendix A.8). The "Up" button on a deep-linked screen must go to its logical parent, never close the app unexpectedly. Avoid `singleTask` + `taskAffinity` tricks unless you understand the Recents consequences; `singleTop` + in-app back stack control is usually enough.

## 7. App states

### 7.1 Cold / warm / foreground

- [ ] Cold: parse in `onCreate`, hand to ViewModel, route once `AppState` is `Ready`. Warm: `onNewIntent` → same path. Test with the app in Recents and with the activity destroyed by "Don't keep activities". Foreground: link while the user is mid-task — apply §6.2 (modal dismissal, flow protection). Links must be **idempotent**: processing twice (rotation, restore) must not create two orders/accept an invite twice. Stale intents: ignore `FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY`; optionally ignore links older than N minutes when restored.

### 7.2 Before login / onboarding

- [ ] Link requires auth and user is logged out → **store a pending link** (in memory + `SavedStateHandle`, and DataStore if login can span process death, e.g. email magic link or browser-based SSO), show login, **resume** after success.
- [ ] Same for onboarding, consent (GDPR/ATT-equivalent prompts), forced update, and "select portal" screens — the pending link survives all of them. Pending link has a **TTL** (e.g. 30 minutes – 24 hours) and is **single-use**; cleared on consume, on sign-out, and on explicit back-out of login. Only one pending link — newest wins. Public destinations (product page) should **not** require login; render a public preview and gate actions. If after login the link resolves to a different account than expected → §8. Never put the user on a blank screen while waiting: show the login screen with context ("Sign in to view your order").

### 7.3 Links during modals and critical flows

- [ ] Payment in progress, unsaved editor, active call → queue the link and show a non-blocking banner/snackbar ("Open link after you finish?"), or confirm discarding. System dialogs (Play Billing sheet, permission prompts) can't be dismissed by you — queue until they return.

## 8. Account awareness

- [ ] Links targeting account- or portal-scoped content (`/w/{portal}/...`, orders, messages) are resolved against the **account registry** (see **android-multi-account**):
  - Target belongs to the active account/portal → navigate.
  - Belongs to another **signed-in** account/portal → **confirm**: "Switch to Acme workspace to view this task?" → run the account/portal switch flow → then navigate.
  - Belongs to no signed-in account → offer "Sign in to another account" with the pending link preserved.
  - Unknown (server says 403/404) → generic "You don't have access / link not found" — **don't reveal** whether the item exists in another org. Resolution may need an API call (`GET /links/resolve?url=`) when the path doesn't carry the account; show a lightweight loading screen, time out to a fallback. Never switch accounts **silently** from a link — a malicious link could otherwise move the user into an attacker-controlled account (account-confusion phishing). Invite links (`/invite/{code}`): accept into the **chosen** account, explicitly shown to the user.

## 9. Deferred deep links (vendor-neutral)

Goal: user taps a link, doesn't have the app, installs from Play, first launch opens the linked content.

### 9.1 Mechanisms (evaluate in this order on Android)

1. **Play Install Referrer API** (deterministic, first-party, recommended):
   - Store link: `https://play.google.com/store/apps/details?id=com.example.app&referrer=<url-encoded>`. Put a **short opaque token** (`ref%3Dabc123`) or the encoded deep link path — not PII.
   - On first launch, `InstallReferrerClient` returns `installReferrer`, `referrerClickTimestampSeconds`, `installBeginTimestampSeconds`, and `googlePlayInstantParam`. Available for a limited time after install (~90 days); call it **once**, persist the result and a "consumed" flag.
   - Only works for installs **from Google Play** that went through a URL with `referrer`. Your web landing page must forward the original link into the Play URL's `referrer`.
   - Handle `FEATURE_NOT_SUPPORTED`, `SERVICE_UNAVAILABLE`, `SERVICE_DISCONNECTED` (retry once with backoff), and empty referrer (organic install).
2. **Server-side match on an explicit token**: landing page creates a short-lived token server-side tied to the link; token travels in the referrer; app exchanges token → link via your API. Lets you revoke/expire links and keeps the Play URL short.
3. **Login-based match**: if the user signs in on the web before installing (or on first app login), the server knows the pending destination for that account. Deterministic, privacy-friendly, works across stores.
4. **Clipboard**: landing page copies a token to the clipboard; app reads it on first launch. Android 12+ shows a toast when an app reads the clipboard, Android 13+ adds preview/visibility indicators — users perceive it as snooping. Only with clear user action ("Copy & open app"), read once, never on every launch. Generally **avoid**.
5. **Probabilistic fingerprinting** (IP + user-agent + timestamp matching): unreliable on carrier NAT/VPN/Wi-Fi, a **privacy and Play policy risk** (device fingerprinting for tracking), and must be disclosed in the Data safety section. **Don't build it**; if a vendor uses it, require it to be off by default.

### 9.2 Evaluation criteria (any vendor or in-house)

- [ ] Uses Install Referrer as the primary Android signal; fingerprinting off or opt-in. Links on **your own domain** (custom domain support), so you keep them if you switch vendors — the Firebase Dynamic Links shutdown stranded every `*.page.link` link.
- [ ] Export of all links/aliases; ability to redirect old links. Data safety / privacy: what is collected (advertising ID, IP), retention, consent integration, region data residency. Works with verified App Links (doesn't wrap links in a redirect domain that kills verification). SDK size, startup cost (no blocking network on main thread at launch), and failure behaviour (app works if the service is down). Attribution windows, fraud protection, raw event export.

### 9.3 Firebase Dynamic Links migration (shut down 25 Aug 2025)

- [ ] Inventory every FDL link: printed QR codes, emails, partner integrations, `*.page.link` and custom-domain FDL links. Old `page.link` links **no longer resolve** — reprint/replace what you can; for custom domains you still control, serve redirects to the new links. Replace with: verified App Links on your domain + web landing page + Play Install Referrer for deferred links (+ any vendor meeting §9.2). Remove the `firebase-dynamic-links` dependency and `FirebaseDynamicLinks.getInstance().getDynamicLink(intent)` calls; route `intent.data` through your parser. Map FDL params (`link=`, `apn`, `afl`, `utm_*`) to your new URL design; keep the parser accepting old long-link query shapes if any are still reachable via your redirects.
- [ ] Email verification / passwordless sign-in links that used FDL (Firebase Auth) need the current Firebase Auth replacement (Hosting-based links) — check current Firebase docs.

### 9.4 Deferred link app behaviour

- [ ] First launch: run onboarding as normal, but carry the deferred link as a **pending link** (§7.2) and route after onboarding/login. Offer, don't force: "Continue to *Spring Sale*?" is fine; dropping a brand-new user deep into a screen without context is not. Never treat a deferred link as more trusted than a normal link (same parser, same validation).

## 10. Other entry points

- [ ] **Push notifications** (see **android-push-notifications**): payload carries a `link` (your canonical URI). Build the tap `PendingIntent` with `NavDeepLinkBuilder` or `TaskStackBuilder` + explicit `Intent(context, MainActivity::class.java).setData(uri)`; `FLAG_IMMUTABLE`; **unique request code per notification** (or `PendingIntent` extras collide and every notification opens the last link). Route through the same parser. No trampolines (Android 12+ blocks notification trampolines via broadcast receivers/services starting activities).
- [ ] **Widgets** (Glance `actionStartActivity` / RemoteViews `setOnClickPendingIntent`): same — explicit intent with `data = uri`; distinct `data` or request code per item.
- [ ] **App shortcuts** (static `shortcuts.xml`, `ShortcutManagerCompat` dynamic/pinned): `Intent(ACTION_VIEW, uri)` targeting `MainActivity`; pinned shortcuts live forever — the route must keep working or fall back. **App Actions / Assistant**: map capabilities to the same URIs.
- [ ] **QR codes**: encode **https App Links** on your domain (work with or without the app); keep them short; never encode custom schemes.
- [ ] **Email click tracking**: ESPs rewrite links to a tracking domain (`click.esp.com/...`) that 302-redirects. The first URL isn't yours → **browser opens, App Link doesn't trigger** (Chrome usually doesn't hand off after a redirect chain started in an email app's webview). Fixes: use a **branded tracking subdomain you control** (`email.example.com`) with its own `assetlinks.json` and a manifest filter; the app then receives the tracking URL, logs/forwards the click, and resolves the destination (follow the redirect via a HEAD/GET or parse a known param). Or disable click tracking for deep-linked CTAs. **In-app browsers** (social apps' webviews) often won't open App Links; the web page should show an "Open in app" button that uses the same https link (user gesture) and a Play Store fallback.
- [ ] **OAuth / SSO redirects**: use AppAuth (Custom Tabs) with either a verified https redirect or a unique custom scheme; validate `state` and PKCE; handle the redirect in a dedicated `RedirectUriReceiverActivity` (AppAuth) or the router's "result" path, never as a navigation destination. Payment/3DS returns: same. **NFC / Nearby / Android Beam replacements**: NDEF URI records → same https links.

## 11. Fallbacks

- [ ] **App not installed**: the https URL must serve a real web page (content or landing page with Play Store link carrying `referrer`, §9).
- [ ] **Unknown path** (`DeepLink.Unknown`): open the app at Home with a gentle message, or open the URL in a **Custom Tab** if it's a web-only page on your domain. Never a crash, never a blank screen. **Expired / revoked** (invite used, offer ended): dedicated screen with explanation and a next action. **Removed routes** after an app update: keep parser support for old paths and map them to new destinations; maintain a test fixture of every link ever shipped. **Not found / no access** (API 404/403 after navigating): detail screen shows a not-found state with a way back — not an infinite spinner. **Older app version** receiving a link for a feature it doesn't have: `Unknown` → Home + "Update the app to open this" with Play Store link (optionally use in-app updates). **Feature flag off / region unavailable**: graceful screen, not a crash. Links your app claims but shouldn't (over-broad filter) → narrow the filter; as a stopgap open them in a Custom Tab.

## 12. Security

- [ ] **Treat every link as untrusted input** — even verified App Links (anyone can craft a URL on your domain) and especially custom schemes and explicit intents from other apps. Validate types and ranges for every param; enforce allowlists for enums (`tab=reviews|specs`).
- [ ] **No destructive or state-changing action from a link alone**: links open a screen; the user confirms (pay, delete, transfer, accept invite, change email, unsubscribe-with-side-effects). No `GET`-style "/delete?id=".
- [ ] **Open redirect**: never load a URL from a param (`?next=https://evil.com`, `?url=`) into a WebView or Custom Tab without an allowlist of your hosts. Never forward link params into a WebView with JavaScript bridges (`addJavascriptInterface`).
- [ ] Don't accept file/content URIs, `intent:` URIs or nested `Intent` extras from links (intent redirection — Play's security scanner flags `getParcelableExtra<Intent>()` → `startActivity`).
- [ ] **Custom scheme hijacking**: another app can register your scheme → never send secrets (auth codes without PKCE, tokens, magic-link codes) via custom schemes; use verified https or PKCE. No **tokens, emails, or PII** in URLs (they end up in logs, analytics, referrers, browser history). Magic-link/email-verify tokens: single-use, short TTL, bound to the requesting device/session where possible, exchanged server-side. Account-scoped links never switch accounts silently (§8). Keep other activities **`exported="false"`**; only the entry activity is exported. Rate-limit and log suspicious resolution failures server-side. `PendingIntent` for push/widgets: `FLAG_IMMUTABLE` + explicit component.

## 13. Analytics

- [ ] Log a **`deep_link_opened`** event: source (`app_link`, `custom_scheme`, `push`, `widget`, `shortcut`, `qr`, `deferred`), normalized path pattern (`/orders/{id}`, not the raw id), launch state (cold/warm/foreground), result (`routed`, `pending_auth`, `account_switch`, `unknown`, `not_found`). Capture **UTM** and campaign params from the URL, then strip them before routing. Deferred installs: log `install_referrer_received` with the token/campaign and `referrerClickTimestampSeconds` → install latency. Track **funnel**: opened → routed → pending → resumed after login → content viewed. Use `Activity.getReferrer()` (best effort, spoofable) only as a hint, never for security. Respect consent: attribution params are personal data in some regions; gate analytics accordingly; no raw URLs with ids/PII in analytics. Monitor `deep_link_opened` with `result=unknown` per app version — catches marketing links pointing at paths the app doesn't parse.

## 14. Testing

### 14.1 Commands

```bash
# Fire a link (cold if app not running, warm otherwise)
adb shell am start -W -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://example.com/orders/123?utm_source=test" com.example.app
# Same without package → tests what the SYSTEM picks (verification, chooser, browser)
adb shell am start -W -a android.intent.action.VIEW -c android.intent.category.BROWSABLE \
  -d "https://example.com/p/abc"
# Custom scheme (quote the URL: & breaks shells)
adb shell am start -W -a android.intent.action.VIEW -d "'com.example.app://p/abc?tab=reviews'"

# Verification state (API 31+)
adb shell pm get-app-links com.example.app
adb shell pm verify-app-links --re-verify com.example.app
adb shell pm set-app-links --package com.example.app 0 all     # reset state
adb shell pm set-app-links-user-selection --user cur --package com.example.app true example.com
adb shell dumpsys package d                                     # domain preferred apps (verbose)

# Cold start
adb shell am force-stop com.example.app
# Process death while backgrounded
adb shell am kill com.example.app

# Install referrer (testing): install from a Play internal testing track via
# a Play URL that carries &referrer=..., or inject a fake referrer client in debug builds.
```

- `pm get-app-links` states: `verified` (good), `none`/`legacy_failure` (fetch failed — check §3.2), `1024`+ codes = specific errors; `approved/denied` = user/adb override. After fixing the server, `--re-verify` and recheck.
- Locally installed release builds are signed with the **upload** key — they verify only if that fingerprint is in the file.
- Play Console → Deep links page and **internal app sharing / internal testing** track verify with the real Play signing key.

### 14.2 Automated tests

1. **Unit (JVM)**: `DeepLinkParser` against a fixture table of every supported URL (valid, invalid, legacy, encoded, aliases, utm stripping) — Appendix A.4. Router against fake `AuthState`/`AccountRegistry`: logged-out → pending, other portal → confirm, unknown → fallback.
2. **Manifest ↔ parser consistency**: a test that loads the fixture list and asserts the manifest filter would match (Robolectric `PackageManager.queryIntentActivities`) **and** the parser returns non-Unknown.
3. **Instrumented**: `ActivityScenario.launch<MainActivity>(Intent(ACTION_VIEW, uri))` → Compose assertions on destination; `scenario.onActivity { it.onNewIntent(...) }` for warm; `scenario.recreate()` to assert no replay.
4. **Pending link**: login flow test that the destination appears after login; process death test with `SavedStateHandle`/DataStore.
5. **Server monitoring**: scheduled check of `assetlinks.json` per host (§3.4).
6. **Back stack**: after link, `navController.currentBackStack` equals expected (Home → Orders → Order).

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| D1 | Tap https link, app installed, verified, cold | App opens directly at target; back → parent → Home |
| D2 | Same, warm (Recents on another screen) or foreground with sheet open | `onNewIntent` routes; no duplicate activity; sheet dismissed |
| D3 | Link during checkout / unsaved editor | Queued or confirm; no data loss |
| D4 | Link, app **not** installed | Web page / landing page with Play link |
| D5 | Host **unverified** (API 31+) or device offline at install | Opens browser; Settings shows host; fix server / adb re-verify succeeds |
| D6 | Production build from Play (Play signing key) | Verified (`pm get-app-links` = verified) |
| D7 | `www.` vs apex vs `m.` host | All route the same |
| D8 | Unknown, legacy or removed path on claimed host | Mapped to new destination, or Home + message / Custom Tab; no crash |
| D9 | Invalid param (`/orders/../../x`, 5 KB id, unicode) | `Unknown`/not-found; no crash |
| D10 | Logged out, link needs auth (incl. process killed during browser SSO) | Login with context → pending link restored → target |
| D11 | Pending link older than TTL | Discarded; Home |
| D12 | Link during onboarding (fresh install) | Onboarding completes → target |
| D13 | Link for another signed-in account/portal | Confirm switch → switch → target |
| D14 | Link for account not signed in | "Sign in to another account" with pending link |
| D15 | Link to item user can't access | Generic no-access/not-found; no info leak |
| D16 | Rotate / dark mode toggle; reopen from Recents days later | No re-navigation; old link not replayed |
| D17 | Same link tapped twice quickly | One destination on stack |
| D18 | Push tap, app killed; two pushes, tap the first | Correct task Home → parent → target; first's target (unique PendingIntent) |
| D19 | Widget / shortcut / pinned shortcut | Same router; removed route falls back |
| D20 | QR code scan; email link with click tracking | App Link opens app (branded tracking domain) or documented web fallback |
| D21 | Deferred: click → install from Play → first open; organic install | Referrer read once → target after onboarding; organic → normal onboarding |
| D22 | Former FDL `page.link` link | Documented: dead or redirected from own domain |
| D23 | OAuth redirect returns | Auth flow completes; no extra screen on stack |
| D24 | `?next=https://evil.com`; custom scheme fired by another app | Ignored / allowlist; validated; no destructive action |
| D25 | Link with `utm_*` | Analytics logged; params stripped from route |
| D26 | Old app version, link to new feature | "Update to open" fallback |

## 15. Common bugs to look for in review

- `assetlinks.json` lists only the **upload/debug** fingerprint, not the Play App Signing key → works in dev, browser in production
- File behind a redirect (apex → www, http → https), a WAF/bot challenge, auth, or served as `text/html` by a SPA catch-all; `www.example.com` in the manifest but the file only on `example.com` (or vice versa)
- `minSdk < 31` with one broken host → **all** hosts unverified on Android 11 and below; Missing `BROWSABLE` category or `android:autoVerify` placed on a filter that also contains a custom scheme
- Several hosts/schemes/paths in one filter → unintended cross-product claims; claiming the entire host, so web-only pages (help, legal, password reset on web) open an app that can't render them
- `onNewIntent` not overridden (or not calling `setIntent`) → warm links ignored or old link re-read later
- Link handled in `onCreate` on every recreation → re-navigation on rotation; not checking `FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY` → stale replay from Recents; `launchMode="standard"` → new activity per link stacked in the browser's task
- Routing before session/DI ready → crash or login screen flashes then disappears
- Pending link lost across login (kept only in a local `var`), never expired, or replayed after sign-out; Account-scoped link opened in the wrong account; silent account switch
- Manifest filters and Navigation `deepLinks` out of sync; parser accepts paths the manifest doesn't claim (or vice versa); String splitting instead of `Uri` APIs → broken on encoded chars; `utm_*` params treated as required args
- Push `PendingIntent` with the same request code and no distinct `data` → every notification opens the last link; missing `FLAG_IMMUTABLE`; notification trampoline; No synthetic back stack → back from deep-linked detail exits the app
- WebView loading `?url=` param → open redirect / JS bridge exposure; destructive action executed directly from a link (`/unsubscribe`, `/accept`, `/pay`)
- Install referrer read on every launch, read on the main thread blocking startup, or never ended (`endConnection`); `firebase-dynamic-links` still in dependencies after the shutdown; QR codes still printing `page.link` URLs; No CI check on `assetlinks.json`; marketing site redeploy deletes it

## 16. Output format

When asked to **review/audit**:
1. **Summary** — link contract table (pattern → destination → auth → account-aware → back stack → fallback), hosts and their `assetlinks.json` status, package names/fingerprints, minSdk/targetSdk, Navigation version
2. **Checklist results** — sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line (or URL for server files)
3. **Critical issues first** (production links opening the browser, wrong fingerprints, ignored warm links, destructive or account-switching links, open redirects, secrets over custom schemes, lost pending links) with fix code/config
4. **Testing gaps** — D1–D26 not covered, and which require a Play-signed build or a real server

When asked to **implement**: fill in the link contract first, then start from the Appendix: manifest filters, `assetlinks.json`, `DeepLinkParser` + fixture tests, `PendingLinkStore`, `DeepLinkRouter`, `MainActivity` intent handling, PendingIntent builder and Install Referrer reader; delete what the contract doesn't need; match Navigation/Compose versions to the project; finish with the commands from §14.1 you ran and those left for the user (server deployment, Play Console checks).

---

## Appendix — Reference implementation (Kotlin, Navigation Compose 2.8+, DataStore)

> Written against Navigation Compose 2.8/2.9 (type-safe routes with kotlinx.serialization), Activity 1.9+, DataStore 1.1, Install Referrer 2.2, targetSdk 35. Adjust package names, routes and the auth/account interfaces to the project.

### A.1 `AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <application ...>
    <activity
        android:name=".MainActivity"
        android:exported="true"
        android:launchMode="singleTop"
        android:theme="@style/Theme.App.Starting">

      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>

      <!-- Verified App Links: one filter per host group; only paths the app renders -->
      <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" />
        <data android:host="example.com" />
        <data android:host="www.example.com" />
        <data android:pathPrefix="/p/" />
        <data android:pathPrefix="/orders" />
        <data android:pathPrefix="/w/" />
        <data android:pathPrefix="/invite/" />
        <data android:pathPrefix="/offer/" />
        <data android:pathPrefix="/go/" />
      </intent-filter>

      <!-- Branded click-tracking subdomain (needs its own assetlinks.json) -->
      <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="email.example.com" android:pathPrefix="/c/" />
      </intent-filter>

      <!-- Custom scheme: separate filter, never autoVerify -->
      <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.example.app" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

> Within the first filter, both hosts × all prefixes are claimed (cross-product) — intended here. Never mix the custom scheme into an https filter.

### A.2 `assetlinks.json` (served at `https://example.com/.well-known/assetlinks.json` and on every other host)

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.app",
      "sha256_cert_fingerprints": [
        "14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5",
        "5F:3A:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:01"
      ]
    }
  }
]
```

Add a second statement with `"package_name": "com.example.app.debug"` and its debug fingerprint if debug builds need verified links. First fingerprint: Play App Signing key (from Play Console). Second: upload key. Fingerprints above are placeholders. Serve with `Content-Type: application/json`, 200, no redirect, no auth.

### A.3 `DeepLink` model and parser

```kotlin
package com.example.app.links

import android.net.Uri

sealed interface DeepLink {
    /** Whether the destination needs an authenticated session. */
    val requiresAuth: Boolean

    data object Home : DeepLink { override val requiresAuth = false }
    data class Product(val productId: String, val tab: ProductTab?) : DeepLink { override val requiresAuth = false }
    data object Orders : DeepLink { override val requiresAuth = true }
    data class Order(val orderId: String) : DeepLink { override val requiresAuth = true }
    data class PortalTask(val portalSlug: String, val taskId: String) : DeepLink { override val requiresAuth = true }
    data class Invite(val code: String) : DeepLink { override val requiresAuth = false }
    data class Offer(val offerId: String) : DeepLink { override val requiresAuth = true }
    /** Marketing alias, resolved server-side to a canonical link. */
    data class Alias(val slug: String) : DeepLink { override val requiresAuth = false }
    /** Click-tracking URL; resolve to destination via API. */
    data class Tracked(val uri: String) : DeepLink { override val requiresAuth = false }
    data class Unknown(val uri: String) : DeepLink { override val requiresAuth = false }
}

enum class ProductTab { REVIEWS, SPECS }

data class ParsedLink(val link: DeepLink, val attribution: Map<String, String>)

/**
 * Pure parser: no navigation, no I/O. Accepts verified hosts and the custom scheme.
 * Uses a minimal [LinkUri] abstraction so it runs in plain JVM unit tests.
 */
class DeepLinkParser(
    private val hosts: Set<String> = setOf("example.com", "www.example.com", "m.example.com"),
    private val trackingHosts: Set<String> = setOf("email.example.com"),
    private val customScheme: String = "com.example.app",
) {
    fun parse(uri: Uri): ParsedLink = parse(LinkUri.from(uri))

    fun parse(uri: LinkUri): ParsedLink {
        val attribution = uri.query.filterKeys { it in ATTRIBUTION_KEYS || it.startsWith("utm_") }
        val segments: List<String> = when {
            uri.scheme == "https" && uri.host in hosts -> uri.pathSegments
            uri.scheme == "https" && uri.host in trackingHosts ->
                return ParsedLink(DeepLink.Tracked(uri.raw), attribution)
            // com.example.app://p/abc → host "p" is the first segment
            uri.scheme == customScheme -> listOfNotNull(uri.host) + uri.pathSegments
            else -> return ParsedLink(DeepLink.Unknown(uri.raw), attribution)
        }.filter { it.isNotEmpty() }

        val link = route(segments, uri.query) ?: DeepLink.Unknown(uri.raw)
        return ParsedLink(link, attribution)
    }

    private fun route(s: List<String>, q: Map<String, String>): DeepLink? = when {
        s.isEmpty() -> DeepLink.Home
        s.size == 2 && s[0] == "p" -> id(s[1])?.let { DeepLink.Product(it, tab(q["tab"])) }
        s.size == 1 && s[0] == "orders" -> DeepLink.Orders
        s.size == 2 && s[0] == "orders" -> id(s[1])?.let(DeepLink::Order)
        // Legacy path kept forever: /order?id=123
        s.size == 1 && s[0] == "order" -> q["id"]?.let(::id)?.let(DeepLink::Order)
        s.size == 4 && s[0] == "w" && s[2] == "t" -> {
            val portal = slug(s[1]); val task = id(s[3])
            if (portal != null && task != null) DeepLink.PortalTask(portal, task) else null
        }
        s.size == 2 && s[0] == "invite" -> code(s[1])?.let(DeepLink::Invite)
        s.size == 2 && s[0] == "offer" -> id(s[1])?.let(DeepLink::Offer)
        s.size == 2 && s[0] == "go" -> slug(s[1])?.let(DeepLink::Alias)
        else -> null
    }

    private fun tab(value: String?): ProductTab? = when (value) {
        "reviews" -> ProductTab.REVIEWS
        "specs" -> ProductTab.SPECS
        else -> null // unknown values ignored, never an error
    }

    private fun id(v: String) = v.takeIf { ID.matches(it) }
    private fun slug(v: String) = v.takeIf { SLUG.matches(it) }
    private fun code(v: String) = v.takeIf { CODE.matches(it) }

    companion object {
        private val ID = Regex("^[A-Za-z0-9_-]{1,64}$")
        private val SLUG = Regex("^[a-z0-9](?:[a-z0-9-]{0,62}[a-z0-9])?$")
        private val CODE = Regex("^[A-Z0-9]{6,32}$")
        private val ATTRIBUTION_KEYS = setOf("cid", "ref", "gclid")
    }
}

/** Framework-free view of a URI (decoded segments and params). */
data class LinkUri(
    val raw: String, val scheme: String?, val host: String?,
    val pathSegments: List<String>, val query: Map<String, String>,
) {
    companion object {
        fun from(uri: Uri) = LinkUri(
            raw = uri.toString(),
            scheme = uri.scheme?.lowercase(),
            host = uri.host?.lowercase(),
            pathSegments = uri.pathSegments,
            query = uri.queryParameterNames.associateWith { uri.getQueryParameter(it).orEmpty() },
        )

        /** JVM-friendly parser for unit tests (java.net.URI). */
        fun parse(raw: String): LinkUri {
            val u = java.net.URI(raw)
            fun dec(s: String) = java.net.URLDecoder.decode(s, Charsets.UTF_8)
            val segments = (u.rawPath ?: "").split('/').filter { it.isNotEmpty() }.map {
                dec(it.replace("+", "%2B"))
            }
            val query = (u.rawQuery ?: "").split('&').filter { it.isNotEmpty() }.associate {
                val (k, v) = it.split('=', limit = 2).let { p -> p[0] to p.getOrElse(1) { "" } }
                dec(k) to dec(v)
            }
            return LinkUri(raw, u.scheme?.lowercase(), u.host?.lowercase(), segments, query)
        }
    }
}
```

### A.4 Parser unit tests (fixture table)

```kotlin
// DeepLinkParserTest (sketch): `fixtures: List<Pair<String, DeepLink>>` holds every link ever shipped (also used by the
// manifest consistency test): home, /p/{id} with www/case/trailing slash/?tab (valid + bogus), /orders, /orders/{id},
// legacy /order?id=, /w/{portal}/t/{task}, /invite, /offer, /go alias, and com.example.app:// forms. Also assert:
// utm_* / cid land in `attribution`, not the route; invalid params (encoded traversal, over-long or bad-charset IDs,
// non-ASCII, foreign host, http, intent://, paths not ours to render) → DeepLink.Unknown; email.example.com/c/… → Tracked.
```

### A.5 Pending link store (survives login and process death)

```kotlin
package com.example.app.links

// Imports: Context; androidx.datastore.preferences.core.{edit, longPreferencesKey, stringPreferencesKey};
// androidx.datastore.preferences.preferencesDataStore; kotlinx.coroutines.flow.first.

private val Context.linkStore by preferencesDataStore(name = "pending_link")

class PendingLinkStore(
    private val context: Context,
    private val clock: () -> Long = System::currentTimeMillis,
    private val ttlMillis: Long = 30 * 60 * 1000L,
) {
    private val uriKey = stringPreferencesKey("uri")
    private val savedAtKey = longPreferencesKey("saved_at")
    private val sourceKey = stringPreferencesKey("source")

    /** Newest wins; stores the raw URI (re-parsed on resume so validation always runs). */
    suspend fun save(uri: String, source: LinkSource) {
        context.linkStore.edit { it[uriKey] = uri; it[savedAtKey] = clock(); it[sourceKey] = source.name }
    }

    /** Single-use: returns and clears. Expired links are discarded. */
    suspend fun consume(): Pair<String, LinkSource>? {
        val prefs = context.linkStore.data.first()
        val uri = prefs[uriKey] ?: return null
        val savedAt = prefs[savedAtKey] ?: 0L
        val source = prefs[sourceKey]?.let { runCatching { LinkSource.valueOf(it) }.getOrNull() }
            ?: LinkSource.APP_LINK
        clear()
        return if (clock() - savedAt <= ttlMillis) uri to source else null
    }

    suspend fun clear() { context.linkStore.edit { it.clear() } }
}

enum class LinkSource { APP_LINK, CUSTOM_SCHEME, PUSH, WIDGET, SHORTCUT, DEFERRED, TRACKED }
```

Call `clear()` on sign-out (all accounts) — see **android-multi-account**.

### A.6 Router (auth gating, account awareness, stack rules)

```kotlin
package com.example.app.links

import androidx.navigation.NavController
import androidx.navigation.NavOptionsBuilder
import kotlinx.serialization.Serializable

// Type-safe routes (Navigation 2.8+)
@Serializable data object HomeRoute; @Serializable data object OrdersRoute; @Serializable data object LoginRoute
@Serializable data class OrderRoute(val orderId: String)
@Serializable data class ProductRoute(val productId: String, val tab: String? = null)
@Serializable data class TasksRoute(val portalSlug: String)
@Serializable data class TaskRoute(val portalSlug: String, val taskId: String)
@Serializable data class InviteRoute(val code: String); @Serializable data class PaywallRoute(val offerId: String)

/** Minimal views of app state the router needs — implemented by the session layer. */
interface AuthState { val isSignedIn: Boolean; val isOnboarded: Boolean }

interface AccountResolver {
    sealed interface Result {
        data object ActivePortal : Result
        data class OtherSignedIn(val accountId: String, val portalId: String, val label: String) : Result
        data object NotSignedIn : Result
        data object NoAccess : Result
    }
    suspend fun resolvePortal(portalSlug: String): Result
}

interface LinkResolverApi {
    /** Resolves /go/{slug} aliases and tracking URLs to canonical https links (or null). */
    suspend fun resolve(uriOrSlug: String): String?
}

sealed interface RouteOutcome {
    data object Routed : RouteOutcome
    data object PendingAuth : RouteOutcome
    data class ConfirmAccountSwitch(val accountId: String, val portalId: String, val label: String, val link: DeepLink) : RouteOutcome
    data object NoAccess : RouteOutcome
    data object Unknown : RouteOutcome
    data object Deferred : RouteOutcome // queued behind a critical flow
}

class DeepLinkRouter(
    private val parser: DeepLinkParser,
    private val pending: PendingLinkStore,
    private val auth: AuthState,
    private val accounts: AccountResolver,
    private val api: LinkResolverApi,
    private val analytics: (String, Map<String, String>) -> Unit,
    private val isInCriticalFlow: () -> Boolean,
) {
    suspend fun handle(raw: String, source: LinkSource, nav: NavController, depth: Int = 0): RouteOutcome {
        val parsed = parser.parse(LinkUri.parse(raw))
        val outcome = route(raw, parsed.link, source, nav, depth)
        analytics("deep_link_opened", parsed.attribution + mapOf(
            "source" to source.name.lowercase(), "pattern" to parsed.link.pattern(),
            "result" to outcome::class.simpleName.orEmpty()))
        return outcome
    }

    private suspend fun route(raw: String, link: DeepLink, source: LinkSource, nav: NavController, depth: Int): RouteOutcome {
        // Aliases / tracked links: resolve once, then re-enter (bounded to avoid loops).
        if (link is DeepLink.Alias || link is DeepLink.Tracked) {
            if (depth > 1) return showUnknown(nav)
            val key = if (link is DeepLink.Alias) link.slug else raw
            val target = runCatching { api.resolve(key) }.getOrNull() ?: return showUnknown(nav)
            return handle(target, source, nav, depth + 1)
        }
        if (link is DeepLink.Unknown) return showUnknown(nav)

        if (link.requiresAuth && (!auth.isSignedIn || !auth.isOnboarded)) {
            pending.save(raw, source)
            if (!auth.isSignedIn) nav.navigate(LoginRoute) { launchSingleTop = true }
            return RouteOutcome.PendingAuth
        }
        if (isInCriticalFlow()) { pending.save(raw, source); return RouteOutcome.Deferred }
        if (link is DeepLink.PortalTask) {
            when (val r = accounts.resolvePortal(link.portalSlug)) {
                AccountResolver.Result.ActivePortal -> Unit
                is AccountResolver.Result.OtherSignedIn ->
                    // UI shows a confirmation; on confirm: switch (android-multi-account) then navigateTo(link)
                    return RouteOutcome.ConfirmAccountSwitch(r.accountId, r.portalId, r.label, link)
                AccountResolver.Result.NotSignedIn -> {
                    pending.save(raw, source)
                    nav.navigate(LoginRoute) { launchSingleTop = true } // "add account" mode
                    return RouteOutcome.PendingAuth
                }
                AccountResolver.Result.NoAccess -> return RouteOutcome.NoAccess
            }
        }
        navigateTo(link, nav)
        return RouteOutcome.Routed
    }

    /** Builds a predictable back stack: start → parent(s) → target. */
    fun navigateTo(link: DeepLink, nav: NavController) {
        val resetToHome: NavOptionsBuilder.() -> Unit = {
            popUpTo(nav.graph.startDestinationId) { inclusive = false }
            launchSingleTop = true
        }
        when (link) {
            DeepLink.Home -> nav.navigate(HomeRoute, resetToHome)
            is DeepLink.Product -> {
                nav.navigate(HomeRoute, resetToHome)
                nav.navigate(ProductRoute(link.productId, link.tab?.name?.lowercase())) { launchSingleTop = true }
            }
            DeepLink.Orders -> nav.navigate(OrdersRoute, resetToHome)
            is DeepLink.Order -> { nav.navigate(OrdersRoute, resetToHome); nav.navigate(OrderRoute(link.orderId)) { launchSingleTop = true } }
            is DeepLink.PortalTask -> { nav.navigate(TasksRoute(link.portalSlug), resetToHome); nav.navigate(TaskRoute(link.portalSlug, link.taskId)) { launchSingleTop = true } }
            is DeepLink.Invite -> {
                nav.navigate(HomeRoute, resetToHome)
                nav.navigate(InviteRoute(link.code)) { launchSingleTop = true } // user confirms there
            }
            is DeepLink.Offer -> { nav.navigate(HomeRoute, resetToHome); nav.navigate(PaywallRoute(link.offerId)) { launchSingleTop = true } }
            is DeepLink.Alias, is DeepLink.Tracked, is DeepLink.Unknown -> showUnknown(nav)
        }
    }

    /** Call after login, onboarding, portal selection or the end of a critical flow. */
    suspend fun resumePending(nav: NavController): RouteOutcome? {
        val (raw, source) = pending.consume() ?: return null
        return handle(raw, source, nav)
    }

    private fun showUnknown(nav: NavController): RouteOutcome {
        nav.navigate(HomeRoute) { popUpTo(nav.graph.startDestinationId); launchSingleTop = true }
        return RouteOutcome.Unknown // UI shows "This link isn't available in the app"
    }
}

private fun DeepLink.pattern(): String = when (this) {
    DeepLink.Home -> "/"
    is DeepLink.Product -> "/p/{id}"
    DeepLink.Orders -> "/orders"
    is DeepLink.Order -> "/orders/{id}"
    is DeepLink.PortalTask -> "/w/{portal}/t/{id}"
    is DeepLink.Invite -> "/invite/{code}"
    is DeepLink.Offer -> "/offer/{id}"
    is DeepLink.Alias -> "/go/{slug}"
    is DeepLink.Tracked -> "tracked"
    is DeepLink.Unknown -> "unknown"
}
```

### A.7 `MainActivity`: cold, warm, rotation, Recents

```kotlin
package com.example.app

// Imports: Intent, Bundle, ComponentActivity, setContent, viewModels, LaunchedEffect, collectAsState, getValue,
// installSplashScreen (core-splashscreen), SavedStateHandle, ViewModel, rememberNavController, LinkSource,
// kotlinx.coroutines.channels.Channel, kotlinx.coroutines.flow.receiveAsFlow.

class MainActivity : ComponentActivity() {

    private val vm: AppViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        val splash = installSplashScreen()
        super.onCreate(savedInstanceState)
        splash.setKeepOnScreenCondition { !vm.isReady.value } // vm caps this wait

        // Only on a genuine launch: not on recreation, not when reopened from Recents.
        if (savedInstanceState == null) offer(intent)

        setContent {
            val nav = rememberNavController()
            val ready by vm.isReady.collectAsState()
            AppNavHost(nav) // NavHost with HomeRoute, OrderRoute, ... (not shown)
            LaunchedEffect(ready) {
                if (!ready) return@LaunchedEffect
                vm.links.collect { (uri, source) -> vm.router.handle(uri, source, nav) }
            }
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        offer(intent)
    }

    private fun offer(intent: Intent) {
        if (intent.flags and Intent.FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY != 0) return
        val uri = intent.data ?: return
        if (intent.getBooleanExtra(EXTRA_CONSUMED, false)) return
        intent.putExtra(EXTRA_CONSUMED, true)
        val source = when {
            intent.hasExtra(EXTRA_SOURCE) ->
                runCatching { LinkSource.valueOf(intent.getStringExtra(EXTRA_SOURCE)!!) }
                    .getOrDefault(LinkSource.APP_LINK)
            uri.scheme == "https" -> LinkSource.APP_LINK
            else -> LinkSource.CUSTOM_SCHEME
        }
        vm.offerLink(uri.toString(), source)
    }

    companion object {
        const val EXTRA_SOURCE = "link_source"
        private const val EXTRA_CONSUMED = "link_consumed"
    }
}

class AppViewModel(private val state: SavedStateHandle /*, injected deps */) : ViewModel() {
    lateinit var router: com.example.app.links.DeepLinkRouter // inject via Hilt in real code
    val isReady = kotlinx.coroutines.flow.MutableStateFlow(false) // set true after session restore

    // Buffered until the UI (and session) is ready; survives config changes.
    private val _links = Channel<Pair<String, LinkSource>>(capacity = Channel.CONFLATED)
    val links = _links.receiveAsFlow()

    fun offerLink(uri: String, source: LinkSource) { _links.trySend(uri to source) }
}
```

> If the session needs network to restore, set `isReady` from cached state and refresh in the background; cap the splash at ~1–2 s.

### A.8 PendingIntents for push, widgets, shortcuts (synthetic back stack)

```kotlin
package com.example.app.links

import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.net.Uri
import androidx.core.app.TaskStackBuilder
import com.example.app.MainActivity

object LinkIntents {

    /** Explicit intent into the single activity; routed by the same parser/router. */
    fun viewIntent(context: Context, uri: Uri, source: LinkSource): Intent =
        Intent(Intent.ACTION_VIEW, uri, context, MainActivity::class.java)
            .putExtra(MainActivity.EXTRA_SOURCE, source.name)

    /**
     * For notifications: TaskStackBuilder creates a fresh task when the app isn't running,
     * so back from the target behaves. The router then builds the in-app stack.
     * Unique requestCode + distinct data per notification prevents PendingIntent collisions.
     */
    fun notificationPendingIntent(context: Context, uri: Uri, notificationId: Int): PendingIntent =
        TaskStackBuilder.create(context)
            .addNextIntentWithParentStack(viewIntent(context, uri, LinkSource.PUSH))
            .getPendingIntent(
                notificationId,
                PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
            )!!

    // Widgets: PendingIntent.getActivity(context, requestCode, viewIntent(…, LinkSource.WIDGET), FLAG_IMMUTABLE or
    // FLAG_UPDATE_CURRENT), one per item distinguished by data URI. Shortcuts: ShortcutInfoCompat.Builder(context, "order_$id")
    // .setIntent(viewIntent(context, Uri.parse("https://example.com/orders/$id"), LinkSource.SHORTCUT)).
}
```

> Alternative when Navigation handles links itself: `NavDeepLinkBuilder(context).setGraph(R.navigation.nav_graph).setDestination(R.id.order).setArguments(args).createPendingIntent()` — it synthesizes the graph's start destinations as the back stack. With Compose type-safe graphs, routing through `MainActivity` + router (above) is simpler and keeps auth gating in one place.

### A.9 Navigation Compose `deepLinks` (when letting Navigation match directly)

```kotlin
// Type-safe (Navigation 2.8+): composable<ProductRoute>(deepLinks = listOf(
//     navDeepLink<ProductRoute>(basePath = "https://example.com/p"),   // /p/{productId}?tab={tab}
//     navDeepLink<ProductRoute>(basePath = "com.example.app://p"))) { entry -> entry.toRoute<ProductRoute>() … }
// Older string routes: navDeepLink { uriPattern = "https://example.com/p/{productId}?tab={tab}" }.
// Warm: navController.handleDeepLink(intent) in onNewIntent; cold: NavHost reads activity.intent on first composition.
```

> Navigation's implicit handling bypasses your auth gate and pending-link logic. Use it only for **public** destinations, or don't register `deepLinks` and use the router (A.6). Don't do both for the same path — the link will be handled twice.

### A.10 Install Referrer (deferred deep link, read once)

```kotlin
package com.example.app.links

import android.net.Uri

// implementation("com.android.installreferrer:installreferrer:2.2")

data class InstallReferral(val raw: String, val clickTimestampSeconds: Long, val installBeginTimestampSeconds: Long)

// InstallReferrerReader.read() (sketch): on Dispatchers.IO with withTimeoutOrNull(5_000), wrap
// InstallReferrerClient.newBuilder(context).build().startConnection(listener) in suspendCancellableCoroutine;
// on InstallReferrerResponse.OK read client.installReferrer (installReferrer, referrerClickTimestampSeconds,
// installBeginTimestampSeconds), otherwise null (FEATURE_NOT_SUPPORTED, SERVICE_UNAVAILABLE, …); always
// endConnection() (also on cancellation); onInstallReferrerServiceDisconnected → null. Never throws.
class InstallReferrerReader(private val context: android.content.Context) {
    suspend fun read(): InstallReferral? = TODO("see sketch above")
}

/**
 * Referrer format we put in the Play URL (url-encoded):
 *   referrer=link%3D%252Fp%252Fabc123%26utm_source%3Dweb   (path-only link + utm)
 *   referrer=t%3Dk3J9xQ                                       (opaque server token)
 * Organic Play installs typically report "utm_source=google-play&utm_medium=organic".
 */
class DeferredLinkResolver(
    private val reader: InstallReferrerReader,
    private val api: LinkResolverApi,
    private val flags: FirstLaunchFlags,
    private val pending: PendingLinkStore,
) {
    suspend fun runOnce() {
        if (flags.referrerConsumed()) return
        val referral = reader.read()
        flags.markReferrerConsumed() // once, even if empty — don't hammer the service
        val params = referral?.raw?.let { parseQuery(it) } ?: return

        val link: String? = when {
            params["t"] != null -> runCatching { api.resolve("token:${params["t"]}") }.getOrNull()
            params["link"] != null -> {
                val path = params["link"]!!
                // Only accept a path on our own host — never a full foreign URL.
                if (path.startsWith("/") && !path.startsWith("//")) "https://example.com$path" else null
            }
            else -> null
        }
        // Stored as pending: routed after onboarding/login through the normal parser + router.
        link?.let { pending.save(it, LinkSource.DEFERRED) }
    }

    private fun parseQuery(q: String): Map<String, String> =
        Uri.parse("x://x?$q").let { u -> u.queryParameterNames.associateWith { u.getQueryParameter(it).orEmpty() } }
}

interface FirstLaunchFlags { suspend fun referrerConsumed(): Boolean; suspend fun markReferrerConsumed() }
```

> Pending deferred links use a longer TTL than normal pending links (install + onboarding can take an hour+); configure a second `PendingLinkStore` instance or a TTL per source.

### A.11 Optional: check verification state (API 31+) for a settings row

```kotlin
// API 31+ (Build.VERSION_CODES.S): context.getSystemService(DomainVerificationManager::class.java)
//   .getDomainVerificationUserState(packageName) → if !isLinkHandlingAllowed all hosts are off; otherwise hosts whose
//   hostToStateMap value is DomainVerificationUserState.DOMAIN_STATE_NONE are unverified.
// Settings row: Intent(Settings.ACTION_APP_OPEN_BY_DEFAULT_SETTINGS, Uri.parse("package:$packageName")) on 31+,
//   ACTION_APPLICATION_DETAILS_SETTINGS below.
```

> `DOMAIN_STATE_VERIFIED` and `DOMAIN_STATE_SELECTED` both mean links open the app. Report unverified hosts to your logs (not to the user on launch) — a sudden rise after a release or web deploy means the server file broke.
