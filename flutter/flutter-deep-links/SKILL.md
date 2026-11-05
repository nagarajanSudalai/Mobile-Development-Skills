---
name: flutter-deep-links
description: Use when implementing, reviewing, testing, or debugging deep links in a Flutter app — iOS Universal Links and Android App Links (https), custom URL schemes, apple-app-site-association and assetlinks.json hosting and verification, Flutter's built-in deep linking flag (FlutterDeepLinkingEnabled / flutter_deeplinking_enabled), go_router routes, redirects and back-stack synthesis, the app_links package, cold/warm/foreground link handling, links that arrive before login or onboarding (pending link store), links for another account or portal, deferred deep links after install (vendor-neutral, incl. Firebase Dynamic Links migration), push/widget/shortcut/QR/OAuth entry points, fallbacks, link security and analytics. Siblings: ios-deep-links, android-deep-links, react-native-deep-links; related flutter-multi-account, flutter-push-notifications.
---

# Flutter Deep Links Skill

Guide for building, auditing, and testing **deep links** in consumer Flutter apps: marketing links, shared content links, email links (password reset, magic login, invitations), QR codes, OAuth redirects, push taps, home-screen widgets and shortcuts. Covers **link types**, **domain verification**, **iOS and Android platform config**, **URL design**, **routing with go_router**, **app states and pending links**, **account awareness**, **deferred deep links**, **fallbacks**, **security**, **analytics** and **testing**. Siblings on other platforms: **ios-deep-links**, **android-deep-links**, **react-native-deep-links**. Related Flutter skills: **flutter-multi-account**, **flutter-push-notifications**, **flutter-in-app-purchase**, **flutter-revenuecat-iap**, **flutter-list-ux**.

Deep linking APIs change between Flutter and package releases. Before shipping, **verify against current docs**:

- **Flutter's built-in deep linking flag**: `FlutterDeepLinkingEnabled` (iOS `Info.plist`) and `flutter_deeplinking_enabled` (Android `<meta-data>`). Since **Flutter 3.27** the default is **on**. Older versions default to off, so you had to opt in. If a plugin such as `app_links` handles links itself, set the flag to **`false`**. Otherwise both the framework and the plugin act on the same link.
- **go_router**: the major version has changed often (`redirect` signature, `onException`, `GoRouterState` fields, `ShellRoute` / `StatefulShellRoute`, case-sensitive matching, `pop` result handling). Read the go_router CHANGELOG for the version in `pubspec.lock`.
- **app_links**: v6 renamed `getInitialAppLink` → `getInitialLink`. `uriLinkStream` now also emits the initial link, and its desktop/web setup changed. Check the version you have.
- **iOS scene lifecycle**: recent Flutter templates are moving to `UIScene`. In a scene-based app, links reach `scene(_:continue:)` / `scene(_:openURLContexts:)`, not the `AppDelegate` methods. Plugins have to support it.
- **Android 12+ (API 31)**: an unverified `https` link opens in the browser, not in a chooser. **Android 15+** lets you match on query params and fragments in `<uri-relative-filter-group>`.
- **Firebase Dynamic Links shut down on 25 August 2025.** Existing `page.link` URLs no longer resolve. Any code that still uses `firebase_dynamic_links` is dead and must be migrated (§9).

Check `flutter --version`, `pubspec.lock`, the iOS deployment target and Android `minSdk`/`targetSdk` before applying any of these rules.

## 0. Mental model

```
 Entry points                     Platform layer                       Dart layer
+------------------+     +-------------------------------+     +----------------------------------+
| https link (UL / |     | iOS: Associated Domains +     |     | Source A: Flutter built-in       |
|  App Link)       | --> |  AASA verified by Apple CDN   | --> |  (FlutterDeepLinkingEnabled)     |
| custom scheme    |     | Android: intent-filter        |     |  -> Router's RouteInformation    |
| push tap         |     |  autoVerify + assetlinks.json |     |  -> GoRouter matches path        |
| widget/shortcut  |     | AppDelegate / SceneDelegate / |     |            OR                    |
| QR / email       |     |  MainActivity (onNewIntent)   |     | Source B: app_links stream       |
| OAuth redirect   |     +-------------------------------+     |  -> DeepLinkCoordinator          |
| deferred(install)|                                           +----------------+-----------------+
+------------------+                                                            |
                                                                                v
                     +----------------------------------------------------------------------+
                     | DeepLinkCoordinator                                                  |
                     |  1 normalize + validate (host allowlist, schema, param types)        |
                     |  2 parse -> DeepLinkTarget (sealed)  | unknown -> fallback           |
                     |  3 gate: ready? signed in? onboarded? right account/portal?          |
                     |       not ready -> PendingLinkStore (resume after gate)              |
                     |       other account -> flutter-multi-account switch flow             |
                     |  4 route: go (replace stack) / push (on top) + synthesized back stack|
                     |  5 analytics: link_open {source, target, outcome}                    |
                     +----------------------------------------------------------------------+
```

Pick **one** source of truth per link. Either the framework flag feeds go_router directly, which is simple and fine when every link maps straight to a route, **or** `app_links` feeds a coordinator and the flag is off. The coordinator is the right choice once you need auth gating, pending links, account switching, analytics or confirmation dialogs. Never run both on the same link.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Link types in use**: https Universal Links / App Links, a custom scheme (`myapp://`), or both. Which **domains and subdomains** (`example.com`, `www.`, `links.`, the click-tracking domain of the email provider, staging domains)?
- **Who creates links**: the marketing team (campaign links), users (share sheet), the backend (emails, push payloads), partners, printed QR codes. Printed links live for years, so their paths must never break.
- **Router**: go_router (version), auto_route, Beamer, plain `Navigator` 1.0 with `onGenerateRoute`, or `MaterialApp.router` with a custom `RouterDelegate`. Is `StatefulShellRoute` used for bottom tabs?
- **Link handling today**: the built-in flag, `app_links`, `uni_links` (unmaintained; migrate), `firebase_dynamic_links` (dead; migrate), or a vendor SDK?
- **Destinations by auth state**: which routes are public (product, article, invite landing page) and which need sign-in (orders, settings, inbox)? Are there links that *are* auth: magic-login, email verification, password reset, OAuth callbacks?
- **Onboarding**: are there mandatory steps (consent, permissions, profile setup) that run before any content is shown?
- **Accounts and portals**: can several accounts be signed in at once, or can a user belong to several workspaces? If yes, links must carry the account or portal they target (§8, **flutter-multi-account**).
- **Deferred links**: must a link survive an install (referral codes, invite codes, the campaign destination after install)?
- **Platforms**: iOS, Android, web (go_router serves the URL bar on web too), desktop (`app_links` has Windows/macOS/Linux support with extra setup).
- **Flavors / environments**: separate bundle IDs or applicationIds and separate domains per flavor (dev/staging/prod)? Every flavor needs its own AASA/assetlinks entries.

## 2. Link types: when to use each

| Type | Example | Verified ownership | Opens app when installed | When not installed | Use for |
|---|---|---|---|---|---|
| iOS Universal Link / Android App Link | `https://example.com/p/123` | Yes (AASA / assetlinks) | Yes, no prompt | Opens the website | Everything public-facing: share, email, marketing, QR |
| Custom scheme | `myapp://p/123` | **No** (any app can claim it) | Yes (Android may show a chooser; iOS shows a prompt from Safari) | Error / nothing | OAuth redirects (with PKCE), internal app-to-app hand-off, widget/shortcut URIs, dev testing |
| Android intent URL | `intent://p/123#Intent;scheme=https;package=com.example;end` | Package-pinned | Yes | Play Store via `S.browser_fallback_url` | Android-only web pages that need a forced hand-off |

Rules:

- [ ] **Default to https links.** They are verified, they fall back to a real web page, and they are what users see in messages.
- [ ] Do not put custom schemes in emails, SMS or marketing. Many clients don't make them clickable, and they fail with no fallback when the app isn't installed.
- [ ] Every https deep link path must also have a **web page** (content preview, "Open in app" / store badges). The web page *is* the fallback.
- [ ] Use custom schemes only where verification doesn't matter or isn't possible, and never treat what they carry as trusted (§12).
- [ ] Use a reverse-DNS-like scheme (`com.example.myapp://`) rather than a generic one (`app://`, `shop://`) to reduce collisions.

## 3. Domain verification

### 3.1 iOS: apple-app-site-association (AASA)

- [ ] Serve the file at `https://<domain>/.well-known/apple-app-site-association`. No file extension.
- [ ] Serve it over **HTTPS with a valid certificate**, return **HTTP 200**, and use **no redirects** (no http→https, no apex→www, no trailing-slash rewrites). Apple will not follow them.
- [ ] Set `Content-Type: application/json`. Keep the file under 128 KB.
- [ ] Apple fetches AASA through **its own CDN** (`app-site-association.cdn-apple.com`), not from the device. Changes can take **hours to days** to reach users. Devices refresh it on install and app update, and periodically after that. Plan path changes ahead of the release.
- [ ] To test with your own server directly, use developer mode: add `applinks:example.com?mode=developer` to the entitlement and enable *Settings → Developer → Associated Domains Development* on the device. Never ship `?mode=developer` in a release build.
- [ ] `appIDs` use the form `<TeamID>.<bundleId>`. Add one entry per flavor or bundle ID that should open links.
- [ ] Use the `components` format (iOS 13+). Put exclusions **first**, because the first match wins.
- [ ] Bot protection, WAFs and geo-blocking must not challenge Apple's CDN. A 403 to the CDN breaks links for every user.
- [ ] Each subdomain needs its own file and its own entitlement entry. `*.example.com` in the entitlement covers subdomains, but each subdomain still serves its own file.

### 3.2 Android: assetlinks.json

- [ ] Serve the file at `https://<domain>/.well-known/assetlinks.json`, with `Content-Type: application/json`, HTTP 200 and **no redirects**.
- [ ] List **every SHA-256 signing cert fingerprint** that signs a build users install:
  - The **Play App Signing key** (Play Console → *Test and release → App integrity → App signing*). This is the key on the build users actually get from Play. Teams forget it most often.
  - The **upload key**, if you side-load or distribute builds signed with it (internal testers, Firebase App Distribution).
  - The **debug key**, on staging domains only (`keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android`).
- [ ] Play Console shows a ready-made Digital Asset Links JSON on its *Deep links* page. Use it to cross-check.
- [ ] Verification runs **per host** listed in the intent filter. On Android 12+, a host whose file is unreachable is unverified, but the other hosts still verify. Before Android 12, one failing host fails them all. Do not list hosts you don't control.
- [ ] Verification runs at install time and again on update (and, on newer versions, retries in the background). A fix on the server is not seen right away. Force it with `adb shell pm verify-app-links --re-verify <package>`.
- [ ] For multiple apps on the same domain, list each `package_name` and its fingerprints in the same file.

### 3.3 Hosting and CDN checklist (both files)

- [ ] Deploy both files with the website, not by hand. Add them to the CI smoke test (`curl -sI` checks status 200, the content-type, and no `Location` header).
- [ ] Set a sensible cache TTL on your CDN (minutes to an hour). Purge the CDN after a change.
- [ ] Do not put the files behind auth, cookies, a consent wall, or a "you are in country X" redirect.
- [ ] Do not serve the SPA `index.html` with a 200 for `/.well-known/*`. A single-page-app catch-all route is a classic cause: the file "exists" but holds HTML.
- [ ] Validate with Apple's CDN (`curl https://app-site-association.cdn-apple.com/a/v1/example.com`), Google's Statement List API (`https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://example.com&relation=delegate_permission/common.handle_all_urls`), and the Play Console *Deep links* report.

## 4. App setup for Flutter

### 4.1 Choose the handling mode

| Mode | Flag | How links arrive | Good for |
|---|---|---|---|
| **A. Built-in + go_router** | `FlutterDeepLinkingEnabled` = true (default on 3.27+) | The engine pushes the link path to `Router`; GoRouter matches it like a typed URL. `redirect` does the gating. | Simple apps: every link is a route, gating is "signed in or not" |
| **B. app_links + coordinator** | **false** on both platforms | `AppLinks().uriLinkStream` (plus the initial link) → your coordinator → `router.go/push` | Pending links across login/onboarding, account switching, confirmations, analytics, custom schemes with hosts, deferred links |

Mode B is recommended for most consumer apps. Mode A is simpler, but you can't easily tell a deep link apart from a normal navigation, and a cold-start link can race the auth state restore.

Notes on Mode A:
- The engine passes only **path + query + fragment** to the router. It drops the host, so `https://a.com/x` and `https://b.com/x` look the same. For a custom scheme `myapp://p/123`, the `p` becomes the *host* and is dropped, so the router sees `/123`. Use `myapp://open/p/123`-style paths, or use Mode B.
- A link that arrives while the app is running replaces the current location (go semantics), unless your redirect rewrites it.

### 4.2 iOS

- [ ] **Associated Domains**: in Xcode, *Runner target → Signing & Capabilities → + Associated Domains* → `applinks:example.com`, `applinks:www.example.com`. This writes `ios/Runner/Runner.entitlements`. The App ID needs the capability in the developer portal (automatic signing handles this).
- [ ] Add **per-flavor entitlements** files when bundle IDs differ (`Runner-dev.entitlements` → `applinks:staging.example.com`), selected per build configuration (`CODE_SIGN_ENTITLEMENTS`).
- [ ] **Flag**: `ios/Runner/Info.plist` → `<key>FlutterDeepLinkingEnabled</key><false/>` for Mode B (or `<true/>` explicitly for Mode A on Flutter < 3.27).
- [ ] **Custom scheme**: `CFBundleURLTypes` → `CFBundleURLSchemes` = `com.example.myapp`. Also add the scheme to `LSApplicationQueriesSchemes` only if *your* app calls `canLaunchUrl` on *other* apps' schemes.
- [ ] If the AppDelegate is customized (FlutterAppDelegate subclass or a `UIScene` migration), make sure `application(_:continue:restorationHandler:)` / `scene(_:continue:)` still call `super`, or forward to the plugins. Otherwise `app_links` never sees the link.
- [ ] Vendor SDKs that swizzle the AppDelegate (analytics, attribution) can swallow links. Test with each SDK enabled.

### 4.3 Android

- [ ] In `android/app/src/main/AndroidManifest.xml`, on `MainActivity`:
  - Keep `android:launchMode="singleTop"` (Flutter template default). With it, a link to the running app arrives through `onNewIntent` and does not start a second activity. `standard` creates duplicate Flutter activities. `singleTask` works, but it clears activities above the root.
  - Add one `<intent-filter android:autoVerify="true">` for https links with `VIEW`, `DEFAULT`, `BROWSABLE`, `scheme="https"`, and each `host`. Add optional `pathPrefix` / `pathPattern` (or, on API 31+, `pathAdvancedPattern`).
  - Put the custom scheme in a **separate** intent filter without `autoVerify`. Mixing the scheme into the https filter muddles verification.
  - Add `<meta-data android:name="flutter_deeplinking_enabled" android:value="false" />` for Mode B.
- [ ] Keep `android:exported="true"` on the activity (required on API 31+ when it has intent filters).
- [ ] Use `manifestPlaceholders` per flavor for the host (`android:host="${deepLinkHost}"`), so staging builds don't claim prod links.
- [ ] Narrow path filters (`pathPrefix="/p/"`) keep the app from capturing web-only pages (`/blog`, `/legal`). Do this filtering in the manifest, not in Dart, because a captured URL can't go back to the browser.
- [ ] Android 12+: users can turn link handling off per app (*Settings → Apps → Open by default*). If verification fails, consider a gentle in-app prompt that deep-links to that settings screen.

### 4.4 Dart: go_router baseline

- [ ] Use `MaterialApp.router(routerConfig: router)`. Create the router **once**, outside `build` (a top-level final, a provider, or a field in a State). Recreating it resets navigation and drops links.
- [ ] Use one top-level `redirect` for auth and onboarding gates, with `refreshListenable` bound to the auth/session state, so the redirect runs again when the user signs in.
- [ ] Use `errorBuilder` / `onException` for unknown paths: show a friendly "link not found" screen with a way home. Never show a blank page.
- [ ] Turn on `debugLogDiagnostics: true` in debug builds.

## 5. URL design

- [ ] **Readable and stable paths**: `/p/{productId}`, `/u/{handle}`, `/orders/{orderId}`, `/invite/{code}`. A path is a public contract. Printed QR codes and old emails will keep hitting it for years.
- [ ] Use **IDs, not titles**, as the key (`/p/123-blue-shoes` is fine if only `123` is parsed; the slug is cosmetic and SEO).
- [ ] Use the **same paths on web and app**. The website renders the fallback for the same URL.
- [ ] **Versioning**: don't put `/v1/` in public paths. Evolve with optional query params, and keep old paths as **aliases** in the parser (`/product/123` → `/p/123`) forever. Remove an alias only when analytics show zero traffic for months.
- [ ] Give **campaign links** a stable short path you control (`https://example.com/go/summer`), resolved server-side to a destination, so marketing can change the destination without changing printed links.
- [ ] **Encoding**: build URLs with `Uri(...)` / `Uri.https(...)`, never string concatenation. Read values with `uri.queryParameters` (decoded). Watch for `+` vs `%20` and double-encoding by email click trackers.
- [ ] **Query params for state, path for identity**: `/search?q=shoes&sort=price`, `/p/123?variant=red`. Ignore unknown params, never fail on them (UTM params, tracker params like `fbclid`, `gclid`).
- [ ] Keep **case-insensitive hosts** and case-sensitive paths. Decide on trailing slashes and normalize them.
- [ ] Carry **account / portal hints** only as opaque IDs (`?ws=w_123`), never emails or names (§8).
- [ ] Keep URLs under ~2,000 characters. Some clients truncate.

## 6. Routing

### 6.1 Parsing

- [ ] Parse the URL into a **typed target** (`sealed class DeepLinkTarget`) in a **pure function** that is unit-tested (Appendix A.3–A.4). Never scatter `uri.pathSegments[1]` across widgets.
- [ ] Check the **host against an allowlist** (prod, www, staging per flavor). For a custom scheme, check the scheme and the first segment.
- [ ] Validate each param with a strict pattern (`^[A-Za-z0-9_-]{1,64}$` for IDs, `int.tryParse` for numbers) and reject the link if one fails. Never pass a raw param to a query, a file path or a WebView.
- [ ] Unknown path → `UnknownLink(uri)` → fallback (§11). Never crash or silently do nothing.

### 6.2 Stack rules

| Situation | Action | go_router |
|---|---|---|
| Link to top-level destination (tab root, home) | Replace stack | `router.go('/home')` |
| Link to detail screen, cold start | Synthesize parent(s) so Back works | Nest route `/shop/p/:id` under `/shop` and call `go('/shop/p/123')`; the parent page sits below |
| Link to detail screen while app open | Usually replace with synthesized stack (go). Push only if the current screen is a natural parent | `go(...)` or `push(...)` |
| Link to a flow (checkout, invite accept) | Push over current context; closing returns user to where they were | `push('/invite/abc')` |
| Link while a modal/sheet/dialog is open | Dismiss transient UI first (or ask if there is unsaved input), then navigate | `Navigator.of(context).popUntil(...)` on the root navigator, or design modals as routes |
| Same screen already showing with same ID | No-op, maybe refresh | compare `router.state.uri` before navigating |
| Same screen, different ID | Replace the top, don't stack ten product pages | `pushReplacement` or `go` |

- [ ] Shape the **route tree like the back stack you want**: nested `GoRoute`s give back-stack synthesis for free with `go`. A flat tree with `go('/p/123')` gives a single-page stack, so Back exits the app.
- [ ] **StatefulShellRoute (tabs)**: a link to `/shop/p/123` should select the Shop tab and push inside that branch. The other tabs keep their state. Test that Back pops within the branch first.
- [ ] Android system Back on a synthesized root goes Home, not out of the app. On iOS, a single-page deep link stack still needs a visible way home (close/home button).
- [ ] Never navigate from inside `build`. Navigate from the coordinator (it holds the `GoRouter`, not a `BuildContext`), or in a post-frame callback.

## 7. App states

| State | How the link arrives (Mode B, app_links) | Pitfalls |
|---|---|---|
| **Cold start** (process dead) | `getInitialLink()`, and in v6+ also the first event of `uriLinkStream` | Router not built yet; auth state still restoring; splash/onboarding redirect overwrites the link; handling it twice (initial + stream) |
| **Warm** (backgrounded) | `uriLinkStream` event (Android `onNewIntent`, iOS `continueUserActivity`) | Modal open; half-filled form; stale screen under the link destination |
| **Foreground** (link tapped inside your own app, e.g. in a WebView or chat message) | Usually you handle it yourself (`url_launcher` would bounce out and back) | Don't round-trip through the OS for your own domain; route internally |

- [ ] **Readiness gate**: the coordinator queues links until (a) the router exists and (b) the session restore finished (token read from secure storage, remote config loaded if routes depend on it). Then it processes the queue in order, keeping only the **latest** link.
- [ ] **Dedupe** the cold-start link. In app_links v6, the initial link may show up in both `getInitialLink()` and the stream. Ignore an identical URI seen within ~2 s, or use only the stream.
- [ ] **Splash/launch screen**: don't hard-code `initialLocation` navigation (e.g. `go('/home')` after splash) that overwrites a pending link. After splash, ask the coordinator "is there a pending link?" first.
- [ ] **Link before login**: store the parsed target in a `PendingLinkStore` → send the user to sign-in. On success, resume the pending link (and clear it). With go_router, the classic pattern is `redirect: '/login?from=${Uri.encodeComponent(state.uri.toString())}'`. Validate `from` on the way back (only relative paths), or it becomes an open redirect (§12).
- [ ] **Link before onboarding**: same store. Resume after the last mandatory step. Skip optional steps for high-intent links (invite accept) if the product allows it.
- [ ] **Pending link lifetime**: in memory for the normal case. Persist it (shared_preferences is fine: no secrets) only when the gate can outlive the process, for example email verification in another app. Add an **expiry** (e.g. 1 hour for a normal link, 24 hours for an invite) and drop it on sign-out.
- [ ] **Auth links themselves** (magic login, email verify, password reset) are **not** queued behind the login gate. They *are* the login. Route them to a handler that exchanges the token and then continues.
- [ ] **Link during a modal or dirty form**: dismiss sheets and dialogs. If there is unsaved input, confirm ("Discard draft?") before leaving. Never lose a checkout in progress silently.
- [ ] **Link during a running flow** (payment in progress, in-app purchase sheet): defer until the flow ends (see **flutter-in-app-purchase** for purchase sheets that must not be interrupted).

## 8. Account awareness

- [ ] If the app supports several accounts or portals (workspaces/organizations/teams), the link must say **which** one it is for. Use an opaque `?acct=` / `?ws=` param or a path prefix (`/w/{portalId}/...`), or have the server resolve it from the resource ID.
- [ ] Resolution order: the explicit hint in the link → the server lookup ("which of my accounts can see order 123?") → the active account.
- [ ] Target is in the **active** account and portal → navigate.
- [ ] Target is in another **signed-in account** or another **portal** of the active account → confirm ("Open in Acme workspace?"), then run the switch flow from **flutter-multi-account** (teardown, rebuild scoped state, reset navigation), *then* navigate. Never show account B's data under account A's session.
- [ ] Target is in an account that **isn't signed in** → "Sign in to view": store the pending link, run add-account, resume.
- [ ] **No access** (removed from portal, wrong user) → a clear "You don't have access" screen with a switch-account option. Don't show a generic 404 or a crash.
- [ ] Logged-out user + invite link → signup/login, then accept the invite, then land in the new portal.

## 9. Deferred deep links (vendor-neutral)

A deferred deep link carries a destination (or referral/invite code) **through an app install** and applies it on first launch. Neither OS offers this as a single built-in feature. It is built from these mechanisms:

| Mechanism | Platform | Accuracy | Privacy | Notes |
|---|---|---|---|---|
| **Play Install Referrer API** | Android (Play Store installs) | Deterministic | Good: Google passes only the `referrer` param you put on the Play URL | Put `referrer=<url-encoded token>` on the `play.google.com/store/apps/details?id=...&referrer=...` link; read it once on first launch via a plugin or a small platform channel to the `InstallReferrerClient`. Valid for a limited period after install |
| **Server-side match with a token the user carries** | Both | Deterministic | Good | The web landing page shows a code or saves a token; after install the user signs in / enters the code, or the app sends a first-launch token the server can tie to a click |
| **Clipboard** | Both (mostly iOS) | Deterministic if present | Poor UX: iOS 16+ shows a paste permission prompt; users find it creepy | Landing page copies a token; app reads clipboard on first launch. Use only with clear user action ("Tap to continue in app"), never silently |
| **Probabilistic fingerprinting** (IP, user agent, screen, time window) | Both | Low/medium; breaks on shared IPs, VPNs, iCloud Private Relay | **Poor**: likely counts as fingerprinting under Apple's rules and the ATT policy; may require consent under GDPR/ePrivacy | Avoid for anything account-related. If used at all, only for non-sensitive content routing, with a short match window and disclosure |
| **Account-based** | Both | Deterministic | Good | Invite emails → user signs up with that email → server attaches the invite. Often the best answer for invites |

Evaluation criteria when picking a mechanism or a third-party link service:
- [ ] **Match method** (deterministic vs probabilistic) and the measured match rate for *your* traffic.
- [ ] **Privacy**: no fingerprinting without consent, a data processing agreement, data region, App Store privacy label impact, whether ATT is required.
- [ ] **Hosting of your domain**: can you keep links on *your* domain (so AASA/assetlinks stay yours and links survive a vendor switch)?
- [ ] **Exit cost**: if the service shuts down, do existing links still resolve? The Firebase Dynamic Links shutdown broke every `page.link` URL that wasn't on a custom domain the owner controlled.
- [ ] **SDK weight and swizzling**: what does it hook into AppDelegate/Activity, and what does it collect?
- [ ] Web fallback quality, social preview (Open Graph) support, analytics export.

**Firebase Dynamic Links migration** (service shut down 25 Aug 2025):
- [ ] Remove `firebase_dynamic_links` and its `FirebaseDynamicLinks.instance.onLink` / `getInitialLink` code.
- [ ] Move link hosting to your own domain with AASA + assetlinks (§3) and handle it with Mode A or B.
- [ ] Replace "deferred after install" with the Play Install Referrer (Android) plus a server-side token or account-based match (iOS).
- [ ] Find every place that **generated** FDL links (backend emails, share code, marketing sheets, printed QR codes) and switch it to the new URLs. Old `page.link` URLs are dead. For printed material, budget for a reprint, or for a redirect if you used a custom domain with FDL that you can still point somewhere.

Rules for first launch:
- [ ] Read the deferred source **once**, on first launch only. Mark it consumed. Never apply a stale referral weeks later.
- [ ] Treat it like any other link: validate, parse, gate, route. Onboarding usually comes first, and the deferred target is resumed after it.

## 10. Other entry points

All of these should produce a `Uri` and enter the **same coordinator**, so they share parsing, gating and analytics.

- [ ] **Push taps**: put a `link` (https URL or relative path) in the payload, not a screen name. Handle cold start (`FirebaseMessaging.instance.getInitialMessage()` / the initial notification response in `flutter_local_notifications`) and warm start (`onMessageOpenedApp`, or the notification tap callback). Route the push to the right account/portal. See **flutter-push-notifications**.
- [ ] **Home-screen widgets** (`home_widget` package): set the tap URI to your custom scheme or https path. Read it through the widget plugin's launch callback, or through `app_links` if the widget opens a VIEW intent / URL.
- [ ] **Shortcuts / app actions** (`quick_actions`): map the shortcut type → URI → coordinator. iOS cold-start shortcuts arrive early. Queue them behind the readiness gate.
- [ ] **Email click tracking** breaks Universal Links. Providers rewrite `https://example.com/p/1` to `https://click.esp.com/abc`. That domain isn't associated with your app, so the link opens Safari. Fixes: turn tracking off for deep-link URLs; use a **branded click-tracking domain you control** that serves your AASA/assetlinks and is listed in your entitlements and intent filters; or let the redirector's landing page re-open the app with a smart banner. Test real sends from your provider.
- [ ] **In-app browsers** (social apps' WebViews) often don't hand Universal Links to the OS. The web fallback should have an explicit "Open in app" button (it navigates to the same URL; a user tap often works where the auto-open didn't).
- [ ] **QR codes**: encode the https link (never a custom scheme). Keep it short (so the QR is less dense). Use a `/go/...` redirect path you control (§5). The iOS camera and Android lens open Universal Links / App Links directly.
- [ ] **OAuth / SSO redirects**: prefer `ASWebAuthenticationSession` / Custom Tabs via `flutter_web_auth_2` or the auth SDK, with a custom scheme or https callback **and PKCE**. Exclude the callback path from the general router (the auth library consumes it). Check `state`.
- [ ] **Universal Links pasted in Safari's address bar** don't open the app (by design). Tapping a link does. Don't treat this as a bug.

## 11. Fallbacks

- [ ] **App not installed**: the website for the same URL shows the content (or a preview) plus App Store / Play badges. On iOS, add a Smart App Banner (`<meta name="apple-itunes-app" content="app-id=..., app-argument=https://example.com/p/123">`).
- [ ] **Unknown path** (old version doesn't know the new route): "This link needs a newer version" with an update button when the link is a known future feature (keep a list, or have the server say so). Otherwise "Link not found" with a way home. Keep the URL in analytics.
- [ ] **Expired / used links** (password reset, one-time invites): an explicit message plus the next step ("Request a new link").
- [ ] **Removed content**: a 404 screen in the app with a way home. Never show an empty detail screen with a spinner forever.
- [ ] **Removed routes**: keep an alias that redirects to the nearest live screen (§5). Never delete a path the app once handled.
- [ ] **Feature-flagged / region-locked routes**: handle "not available" explicitly. Do not just render home.
- [ ] **App too old to verify** (the AASA has changed but the user hasn't updated): the website still works. That is why every path needs a web page.

## 12. Security

- [ ] **Treat every link as untrusted input.** Anyone can craft and send it, and on custom schemes any app can fire it.
- [ ] **No destructive or state-changing actions straight from a link**: no "delete", "pay", "transfer", "accept terms", "change email" on open. Show the screen with an explicit confirm button. A link can *pre-fill*, never *execute*.
- [ ] Only tokens that are **one-time, short-lived and server-validated** can authenticate (magic link, email verify). Bind them to the device or session where possible. Never log full tokens. Strip them from analytics and crash breadcrumbs.
- [ ] **Open redirect**: never navigate to (or open in a WebView / `url_launcher`) a URL taken from a param (`?next=`, `?redirect=`, `?from=`) unless it is a **relative path** that your router knows, or a host on your allowlist. Reject `//evil.com`, `javascript:`, `file:`, `intent:`, and other schemes.
- [ ] **WebView targets**: don't load arbitrary link-provided URLs in a WebView that has JavaScript bridges or auth cookies.
- [ ] **Custom scheme hijacking**: another app can register your scheme. Never send secrets (auth codes without PKCE, tokens) to a custom scheme. Use PKCE, and prefer https callbacks with verification.
- [ ] **Android intent spoofing**: don't export extra activities that take links. Keep link handling on `MainActivity` and validate inside Dart.
- [ ] **Route guards still apply**: a link to `/admin/...` or to another user's `/orders/123` must go through the same auth and permission checks as in-app navigation. The server enforces access. The client only shows "no access" nicely.
- [ ] **Rate-limit / protect** server endpoints that resolve links (short-link lookups, invite codes) against enumeration.
- [ ] Watch for **PII in URLs** (emails, phone numbers, names). URLs end up in logs, analytics, and referrer headers.

## 13. Analytics

- [ ] Log one `deep_link_opened` event per link with: `source` (`universal_link`, `app_link`, `custom_scheme`, `push`, `widget`, `shortcut`, `qr`, `deferred`), `route` (the parsed target type, not the raw URL), `app_state` (`cold`/`warm`), `outcome` (`routed`, `pending_auth`, `pending_onboarding`, `account_switch`, `unknown`, `invalid`, `no_access`), and any UTM fields.
- [ ] **UTM params**: capture `utm_source/medium/campaign/content/term` before stripping them from the route. Attach them to the session for conversion attribution. Don't put them in the router's state (they break "same screen" comparisons).
- [ ] Log the outcome of **pending links** (resumed / expired / abandoned at login) to see how much the auth gate costs.
- [ ] Track the **unknown link rate** per app version. Spikes mean a marketing link went out ahead of the release, or an alias was removed.
- [ ] Measure the **verification failure rate** on the server side: traffic to deep-link pages from mobile user agents with the app installed (the web page can detect it via a smart banner click or an "open in app" click).
- [ ] Never log full URLs with tokens (§12). Redact query params with an allowlist.
- [ ] Attribution through installs is §9. Keep the privacy rules there (no fingerprinting without consent).

## 14. Testing

### 14.1 Commands

```bash
# iOS Simulator — Universal Link (simulator honours Associated Domains for signed builds; developer mode recommended)
xcrun simctl openurl booted "https://example.com/p/123?utm_source=test"
xcrun simctl openurl booted "com.example.myapp://open/p/123"
# iOS device: send the link via Notes/Messages and tap it (pasting in Safari's address bar does not trigger UL)
# iOS: check Apple's CDN copy of AASA
curl -s https://app-site-association.cdn-apple.com/a/v1/example.com | jq .
# iOS device logs: Console.app, filter "swcd" for association fetch errors

# Android — App Link (cold if app is killed first)
adb shell am force-stop com.example.myapp
adb shell am start -W -a android.intent.action.VIEW -c android.intent.category.BROWSABLE \
  -d "https://example.com/p/123" com.example.myapp
# Android — warm (app in background; omit package to test the chooser / verification)
adb shell am start -W -a android.intent.action.VIEW -d "https://example.com/p/123"
adb shell am start -W -a android.intent.action.VIEW -d "com.example.myapp://open/p/123"   # custom scheme
# Android — verification state per host (verified / none / 1024 = failed etc.)
adb shell pm get-app-links com.example.myapp
adb shell pm verify-app-links --re-verify com.example.myapp
# Android — reset user choice / turn on approval for testing
adb shell pm set-app-links --package com.example.myapp 0 all
adb shell pm set-app-links-user-selection --user cur --package com.example.myapp true example.com

# Hosting checks
curl -sI https://example.com/.well-known/apple-app-site-association   # 200, application/json, no Location
curl -sI https://example.com/.well-known/assetlinks.json
curl -s "https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://example.com&relation=delegate_permission/common.handle_all_urls"
# Install referrer (Android): test with an internal testing track link carrying &referrer=...
```

Flutter-level tests:
- [ ] **Unit tests** for the parser (Appendix A.8): every path, every alias, invalid params, foreign hosts, encoding edge cases, UTM stripping, open-redirect attempts.
- [ ] **Coordinator tests** with a fake router, fake auth and a fake clock: cold-start dedupe, the pending link resumed after login, expiry, the account switch prompt.
- [ ] **Widget tests** for go_router: `router.go('/p/123')` then `pumpAndSettle()`, then check the screen and `router.routerDelegate.currentConfiguration` / back behaviour (`router.canPop()`).
- [ ] **Integration tests** (`integration_test`) on a device: launch with a link (`adb` or `simctl` from the CI script before or while the test runs), then check the screen.
- [ ] A release build **signed by Play** (internal testing track) for App Link verification. Debug/upload-signed builds verify against different fingerprints.

### 14.2 Scenario matrix

| Scenario | Expect |
|---|---|
| Cold start, signed in, `https://example.com/p/123` | Product 123 shown; Back → Shop root (synthesized), not exit |
| Cold start, signed out, link to `/orders/9` | Login shown; after login → order 9; pending link cleared (also dropped on sign-out) |
| Cold start, first launch, onboarding required | Onboarding, then link destination |
| Warm start, link to product while another product open | Replaces top; one product screen, not two |
| Warm start, bottom sheet open / unsaved form | Sheet dismissed, destination shown / confirm discard; cancel keeps form |
| Same link tapped twice quickly, or delivered by both `getInitialLink` and stream | Handled once |
| Link to tab branch (`/shop/p/1`) with tabs | Shop tab selected, other tabs' state intact |
| Unknown path `/new-feature/1` | "Link not found / update app" screen with Home |
| Invalid param `/p/<script>` | Rejected → not-found; nothing rendered from param |
| `?next=//evil.com` on login link | Ignored; lands on home |
| Link for portal B while A active / for an account not signed in | Confirm + switch / add-account login, then destination |
| Link to resource user can't access | "No access" screen with switch-account option |
| Expired reset link | "Link expired" + request new link |
| App not installed | Website shows content + store badges |
| Deferred: install via Play link with referrer | First launch after onboarding → destination; second launch: nothing |
| Email with click tracking (real send) | Opens app (branded tracking domain) or web with Open-in-app |
| Push / widget / shortcut tap, cold and warm | Same coordinator path after readiness gate; correct account |
| OAuth callback | Consumed by auth flow; not routed as a page |
| Flavor: staging app installed beside prod | Prod links open prod app only |
| Android 12+, verification failed | Opens browser; `pm get-app-links` shows failure; fix server, re-verify |
| AASA changed (new path excluded) | After CDN refresh, excluded path opens web |
| Web build | URL bar route matches same parser; refresh keeps page |

### 14.3 Common bugs to look for in review

- `FlutterDeepLinkingEnabled` / `flutter_deeplinking_enabled` left **on** (default on 3.27+) while `app_links` also handles links → double navigation, or the framework route overwrites the coordinator's.
- Flag off with no `app_links`, or flag on with Flutter < 3.27 never set → links open the app at the home screen.
- `GoRouter` created inside `build` → navigation reset, links lost on rebuild.
- Splash screen calls `go('/home')` after a delay and overwrites the cold-start link; or the cold-start link is handled twice (initial + stream).
- `redirect` to `/login` without remembering the original location, or `from` param used without validation (open redirect).
- `redirect` loops (`/login` redirects to `/login`), or `redirect` that `await`s slow network calls on every navigation.
- Flat route tree → deep-linked detail screen has no Back; `context.push` for every link → the stack grows without limit.
- Navigating with a `BuildContext` that is gone (captured before `await`) → "Looking up a deactivated widget's ancestor".
- AASA served with a redirect, wrong content type, behind a WAF, or as the SPA `index.html`; assetlinks.json missing the **Play App Signing** SHA-256 (works with local builds, fails for every Play user).
- Staging host in the prod intent filter → verification fails for the unreachable host (all hosts before Android 12).
- `launchMode="standard"` → a second Flutter activity per link.
- Custom AppDelegate / SceneDelegate doesn't forward `continueUserActivity` → iOS links ignored when warm.
- Custom scheme `myapp://p/123` with Mode A → host `p` dropped, route `/123` not found.
- Email links break because of the provider's click tracking domain.
- Still calling `firebase_dynamic_links` (dead since Aug 2025).
- Tokens from magic links logged in analytics or crash breadcrumbs; link performs an action (accept, delete, subscribe) without a confirm screen.
- Link for another account shown in the active account's session.
- Pending link never expires → user lands on a weeks-old destination after their next login.
- UTM/tracker params break route matching or "already on this screen" checks.

## 15. Output format

**When reviewing** existing deep link handling, respond with:

1. **Context summary**: link types and domains, handling mode (flag / app_links / other), router and version, auth/onboarding gates, accounts/portals, Flutter and package versions.
2. **Verification report**: AASA and assetlinks status per domain (URL, status code, content-type, redirect, fingerprints incl. Play signing key, CDN copy), entitlements and intent filters per flavor.
3. **Findings** grouped by area (Platform config, Verification, Routing/stack, App states, Account awareness, Deferred, Entry points, Fallbacks, Security, Analytics), each with **severity** (Critical / High / Medium / Low), file:line, what's wrong, the user-visible symptom, and the fix.
4. **Scenario gaps**: §14.2 rows not handled or not tested.
5. **Version notes**: APIs to verify against the project's versions (§ intro).

**When implementing**, deliver:

1. Platform config: entitlements, `Info.plist` flag/schemes, manifest intent filters and meta-data per flavor.
2. Hosted files: AASA and assetlinks.json for each domain, plus a CI smoke check.
3. Dart: `DeepLinkTarget` + parser with unit tests, `PendingLinkStore`, `DeepLinkCoordinator` (sources, readiness gate, dedupe, account check, routing, analytics), and the go_router config with redirect and back-stack-shaped routes.
4. Fallback screens (not found, expired, no access, update needed).
5. A test plan with the commands in §14.1 and the scenario matrix rows that apply, plus assumptions and versions to verify.

## Appendix — Reference implementation (Dart 3, Flutter 3.27+, go_router 14+, app_links 6+)

Mode B: built-in flag **off**, `app_links` feeds a coordinator, go_router renders. Adapt names to the project's state management (this uses plain `ChangeNotifier`/`ValueNotifier`). Verify the package APIs against the versions you use.

### A.1 Platform config

`ios/Runner/Runner.entitlements`: `com.apple.developer.associated-domains` array with `applinks:example.com`, `applinks:www.example.com`, and `applinks:click.example.com` (branded email click-tracking domain).

`ios/Runner/Info.plist` (excerpt):

```xml
<key>FlutterDeepLinkingEnabled</key>
<false/>  <!-- app_links handles links; framework must not also route them -->
<key>CFBundleURLTypes</key>
<array><dict>
  <key>CFBundleURLName</key><string>com.example.myapp</string>
  <key>CFBundleURLSchemes</key><array><string>com.example.myapp</string></array>
</dict></array>
```

`android/app/src/main/AndroidManifest.xml` (excerpt):

```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTop" ...>

    <meta-data android:name="flutter_deeplinking_enabled" android:value="false" />

    <!-- Verified https App Links: only paths the app really handles -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" />
        <data android:host="${deepLinkHost}" />
        <data android:host="www.${deepLinkHost}" />
        <data android:pathPrefix="/p/" />
        <data android:pathPrefix="/orders/" />
        <data android:pathPrefix="/invite/" />
        <!-- ...one <data android:pathPrefix> per claimed prefix (/w/, /go/) -->
    </intent-filter>

    <!-- Custom scheme: separate filter, no autoVerify -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.example.myapp" android:host="open" />
    </intent-filter>
    <!-- MAIN/LAUNCHER filter unchanged -->
</activity>
```

Note that `<data>` elements inside one intent filter **combine** (every scheme × host × path). That is why the custom scheme lives in its own filter.

`android/app/build.gradle.kts`: per flavor set `manifestPlaceholders["deepLinkHost"]` (`example.com` for prod, `staging.example.com` with `applicationIdSuffix = ".staging"` for staging).

### A.2 Hosted files

`https://example.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": { "details": [{
    "appIDs": ["ABCDE12345.com.example.myapp"],
    "components": [
      { "/": "/p/*/reviews/internal*", "exclude": true, "comment": "web only" },
      { "/": "/p/*" }, { "/": "/orders/*" }, { "/": "/invite/*" }, { "/": "/w/*" }, { "/": "/go/*" },
      { "/": "/auth/magic", "?": { "token": "?*" } }
    ]
  }] },
  "webcredentials": { "apps": ["ABCDE12345.com.example.myapp"] }
}
```

`https://example.com/.well-known/assetlinks.json`:

```json
[{ "relation": ["delegate_permission/common.handle_all_urls"],
   "target": { "namespace": "android_app", "package_name": "com.example.myapp",
               "sha256_cert_fingerprints": ["AA:BB:...:PLAY_APP_SIGNING_KEY_SHA256", "CC:DD:...:UPLOAD_KEY_SHA256"] } }]
```

### A.3 Link target model

```dart
// lib/deep_links/deep_link_target.dart
sealed class DeepLinkTarget {
  const DeepLinkTarget();
  bool get requiresAuth => true; // destination needs a signed-in session
  String? get portalId => null;  // opaque account/portal hint carried by the link
}

final class ProductLink extends DeepLinkTarget {
  const ProductLink(this.productId, {this.variant});
  final String productId;
  final String? variant;
  @override bool get requiresAuth => false;
}

final class OrderLink extends DeepLinkTarget {
  const OrderLink(this.orderId, {this.portalId});
  final String orderId;
  @override final String? portalId;
}

final class InviteLink extends DeepLinkTarget {
  const InviteLink(this.code);
  final String code;
  @override bool get requiresAuth => false; // landing page explains, then asks to sign in
}

final class MagicLoginLink extends DeepLinkTarget {
  const MagicLoginLink(this.token);
  final String token;
  @override bool get requiresAuth => false; // this IS the login
  @override String toString() => 'MagicLoginLink(<redacted>)';
}

final class CampaignLink extends DeepLinkTarget {
  const CampaignLink(this.slug);
  final String slug; // resolved server-side to a destination
  @override bool get requiresAuth => false;
}

final class HomeLink extends DeepLinkTarget {
  const HomeLink();
  @override bool get requiresAuth => false;
}

final class UnknownLink extends DeepLinkTarget {
  const UnknownLink(this.uri);
  final Uri uri;
  @override bool get requiresAuth => false;
}

/// Result of parsing: target plus attribution metadata stripped from the URL.
final class ParsedLink {
  const ParsedLink(this.target, {this.utm = const {}, required this.source});
  final DeepLinkTarget target;
  final Map<String, String> utm;
  final LinkSource source;
}

enum LinkSource { universalLink, customScheme, push, widget, shortcut, deferred, internal }
```

### A.4 Parser (pure, unit-tested)

```dart
// lib/deep_links/deep_link_parser.dart
import 'deep_link_target.dart';

class DeepLinkParser {
  DeepLinkParser({required Set<String> allowedHosts, required this.customScheme})
      : _hosts = allowedHosts.map((h) => h.toLowerCase()).toSet();

  final Set<String> _hosts;
  final String customScheme;

  static final _id = RegExp(r'^[A-Za-z0-9_-]{1,64}$');
  static final _token = RegExp(r'^[A-Za-z0-9._~-]{16,512}$');
  static const _utmKeys = {'utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term'};

  ParsedLink parse(Uri raw, {LinkSource? source}) {
    final uri = _normalize(raw);
    final src = source ??
        (uri.scheme == customScheme ? LinkSource.customScheme : LinkSource.universalLink);
    final utm = {
      for (final e in uri.queryParameters.entries)
        if (_utmKeys.contains(e.key)) e.key: e.value,
    };

    final segments = _segmentsFor(uri);
    if (segments == null) return ParsedLink(UnknownLink(raw), source: src);

    return ParsedLink(_match(segments, uri.queryParameters, raw), utm: utm, source: src);
  }

  /// Relative in-app paths only (for `?from=` / `?next=`). Returns null if unsafe.
  static String? safeRelativeLocation(String? value) {
    if (value == null || value.isEmpty) return null;
    final uri = Uri.tryParse(value);
    if (uri == null || uri.hasScheme || uri.hasAuthority) return null;
    if (!value.startsWith('/') || value.startsWith('//') || value.contains(r'\')) return null;
    return value;
  }

  Uri _normalize(Uri uri) {
    var path = uri.path;
    if (path.length > 1 && path.endsWith('/')) path = path.substring(0, path.length - 1);
    return uri.replace(scheme: uri.scheme.toLowerCase(), host: uri.host.toLowerCase(), path: path);
  }

  /// Returns path segments, or null when the scheme/host is not ours.
  List<String>? _segmentsFor(Uri uri) {
    if (uri.scheme == 'https') {
      return _hosts.contains(uri.host) ? uri.pathSegments : null;
    }
    if (uri.scheme == customScheme) {
      // com.example.myapp://open/p/123  -> host "open", segments [p, 123]
      return uri.host == 'open' ? uri.pathSegments : null;
    }
    return null; // http, javascript:, file:, intent:, anything else
  }

  DeepLinkTarget _match(List<String> s, Map<String, String> q, Uri raw) {
    bool ok(String v) => _id.hasMatch(v);

    // Optional portal prefix: /w/{portalId}/...
    String? portal;
    var seg = s;
    if (seg.length >= 2 && seg[0] == 'w') {
      if (!ok(seg[1])) return UnknownLink(raw);
      portal = seg[1];
      seg = seg.sublist(2);
    }
    portal ??= q['ws'] != null && ok(q['ws']!) ? q['ws'] : null;

    switch (seg) {
      case []:
        return const HomeLink();
      case ['p', final id] when ok(id):
      case ['product', final id] when ok(id): // legacy alias, keep forever
        final variant = q['variant'];
        return ProductLink(id, variant: variant != null && ok(variant) ? variant : null);
      case ['orders', final id] when ok(id):
        return OrderLink(id, portalId: portal);
      case ['invite', final code] when ok(code):
        return InviteLink(code);
      case ['go', final slug] when ok(slug):
        return CampaignLink(slug);
      case ['auth', 'magic']:
        final t = q['token'];
        return t != null && _token.hasMatch(t) ? MagicLoginLink(t) : UnknownLink(raw);
      default:
        return UnknownLink(raw);
    }
  }
}
```

Slugs like `/p/123-blue-shoes` pass the ID pattern as a whole. If you need the numeric part only, split on the first `-` before validating.

### A.5 Pending link store

```dart
// lib/deep_links/pending_link_store.dart
import 'deep_link_target.dart';

/// In-memory by default. Persist only non-secret targets if the gate can outlive
/// the process (e.g. email verification in another app). Never persist tokens.
class PendingLinkStore {
  PendingLinkStore({DateTime Function()? clock, this.ttl = const Duration(hours: 1)})
      : _clock = clock ?? DateTime.now;

  final DateTime Function() _clock;
  final Duration ttl;
  ({ParsedLink link, DateTime createdAt})? _pending;

  void save(ParsedLink link) => _pending = (link: link, createdAt: _clock()); // latest wins

  /// Returns and clears the pending link if it has not expired.
  ParsedLink? take() {
    final p = _pending;
    _pending = null;
    if (p == null) return null;
    final ttlFor = p.link.target is InviteLink ? const Duration(hours: 24) : ttl;
    return _clock().difference(p.createdAt) <= ttlFor ? p.link : null;
  }

  bool get hasPending => _pending != null;
  void clear() => _pending = null; // call on sign-out
}
```

### A.6 Coordinator

```dart
// lib/deep_links/deep_link_coordinator.dart
import 'dart:async';

import 'package:app_links/app_links.dart';
import 'package:flutter/widgets.dart';
import 'package:go_router/go_router.dart';

import 'deep_link_parser.dart';
import 'deep_link_target.dart';
import 'pending_link_store.dart';

/// What the coordinator needs from the session layer (see flutter-multi-account).
abstract interface class SessionGate {
  bool get isRestored;          // secure storage read, session known
  bool get isSignedIn;
  bool get isOnboarded;
  String? get activePortalId;
  bool canAccessPortal(String portalId);
  /// Shows a confirm dialog and runs the switch flow. Returns false if cancelled.
  Future<bool> confirmAndSwitchPortal(String portalId);
  Future<bool> signInWithMagicToken(String token);
  Listenable get changes;       // fires on restore/sign-in/onboarding/switch
}

typedef LinkAnalytics = void Function(String event, Map<String, Object?> params);

class DeepLinkCoordinator {
  DeepLinkCoordinator({
    required this.router,
    required this.session,
    required this.parser,
    required this.pending,
    required this.analytics,
    AppLinks? appLinks,
    DateTime Function()? clock,
  })  : _appLinks = appLinks ?? AppLinks(),
        _clock = clock ?? DateTime.now;

  final GoRouter router;
  final SessionGate session;
  final DeepLinkParser parser;
  final PendingLinkStore pending;
  final LinkAnalytics analytics;
  final AppLinks _appLinks;
  final DateTime Function() _clock;
  StreamSubscription<Uri>? _sub;
  Uri? _lastUri;
  DateTime? _lastAt;
  bool _coldStart = true, _busy = false;

  Future<void> start() async {
    session.changes.addListener(_onSessionChanged);
    // app_links 6+: uriLinkStream also emits the initial link; the dedupe in
    // [_onUri] guards against platforms/versions that deliver it twice.
    _sub = _appLinks.uriLinkStream.listen(_onUri, onError: (Object e) {
      analytics('deep_link_error', {'error': e.runtimeType.toString()});
    });
    final initial = await _appLinks.getInitialLink();
    if (initial != null) _onUri(initial);
    // Anything that arrives after this frame is a warm link.
    WidgetsBinding.instance.addPostFrameCallback((_) => _coldStart = false);
  }

  void dispose() { _sub?.cancel(); session.changes.removeListener(_onSessionChanged); }

  /// Entry for push taps, widgets, shortcuts, deferred links, in-app links.
  void handleUri(Uri uri, LinkSource source) => _handle(parser.parse(uri, source: source));

  void _onUri(Uri uri) {
    final now = _clock();
    if (uri == _lastUri && _lastAt != null && now.difference(_lastAt!) < const Duration(seconds: 2)) {
      return; // same link delivered twice (initial + stream, or double tap)
    }
    (_lastUri, _lastAt) = (uri, now);
    _handle(parser.parse(uri));
  }

  void _onSessionChanged() {
    if (!_ready) return;
    final link = pending.take();
    if (link != null) {
      analytics('deep_link_resumed', {'route': link.target.runtimeType.toString()});
      _handle(link);
    }
  }

  bool get _ready => session.isRestored;

  Future<void> _handle(ParsedLink link) async {
    final t = link.target;
    void log(String outcome) => analytics('deep_link_opened', {'source': link.source.name,
          'route': t.runtimeType.toString(), 'app_state': _coldStart ? 'cold' : 'warm', 'outcome': outcome, ...link.utm});

    if (!_ready) {
      pending.save(link);
      return log('pending_restore');
    }
    if (_busy) {
      pending.save(link); // latest wins; picked up when the current one finishes
      return log('queued');
    }
    _busy = true;
    try {
      if (t is UnknownLink) { router.go('/not-found'); return log('unknown'); }
      if (t is MagicLoginLink) {
        final ok = await session.signInWithMagicToken(t.token);
        router.go(ok ? '/home' : '/link-expired');
        return log(ok ? 'routed' : 'expired');
      }
      if (t.requiresAuth && !session.isSignedIn) {
        pending.save(link);
        router.go('/login');
        return log('pending_auth');
      }
      if (session.isSignedIn && !session.isOnboarded) {
        pending.save(link);
        router.go('/onboarding');
        return log('pending_onboarding');
      }
      final portal = t.portalId;
      if (portal != null && portal != session.activePortalId) {
        if (!session.canAccessPortal(portal)) { router.go('/no-access'); return log('no_access'); }
        final switched = await session.confirmAndSwitchPortal(portal);
        if (!switched) return log('switch_cancelled');
        log('account_switch');
      }
      _route(t);
      log('routed');
    } finally {
      _busy = false;
      if (pending.hasPending && _ready && session.isSignedIn && session.isOnboarded) {
        final next = pending.take();
        if (next != null) unawaited(_handle(next));
      }
    }
  }

  void _route(DeepLinkTarget t) {
    final location = switch (t) {
      ProductLink(:final productId, :final variant) =>
        Uri(path: '/shop/p/$productId', queryParameters: variant == null ? null : {'variant': variant}).toString(),
      OrderLink(:final orderId) => '/account/orders/$orderId',
      InviteLink(:final code) => '/invite/$code',
      CampaignLink(:final slug) => '/go/$slug', // screen resolves slug via API
      HomeLink() => '/home',
      MagicLoginLink() || UnknownLink() => '/home', // handled above
    };
    // Same screen already showing? Do nothing.
    if (router.routerDelegate.currentConfiguration.uri.toString() == location) return;
    // go() with nested routes synthesizes the back stack (e.g. /shop below /shop/p/1).
    router.go(location);
  }
}
```

### A.7 Router config

```dart
// lib/router.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

import 'deep_links/deep_link_parser.dart';

GoRouter buildRouter(AuthState auth) => GoRouter(
      initialLocation: '/home',
      refreshListenable: auth,
      debugLogDiagnostics: kDebugMode,
      redirect: (context, state) {
        final loc = state.matchedLocation;
        final publicRoutes = {'/login', '/not-found', '/link-expired', '/no-access'};
        final isPublic = publicRoutes.contains(loc) ||
            ['/shop', '/invite', '/go'].any(loc.startsWith);
        if (!auth.isRestored) return null; // splash shown by the screens themselves
        if (!auth.isSignedIn && !isPublic) {
          return Uri(path: '/login', queryParameters: {'from': state.uri.toString()}).toString();
        }
        if (auth.isSignedIn && loc == '/login') {
          // Only relative in-app paths: blocks ?from=//evil.com
          return DeepLinkParser.safeRelativeLocation(state.uri.queryParameters['from']) ?? '/home';
        }
        return null;
      },
      errorBuilder: (context, state) => const NotFoundScreen(),
      routes: [
        // plus plain GoRoutes for /login, /onboarding, /not-found, /link-expired, /no-access
        GoRoute(path: '/invite/:code', builder: (_, s) => InviteScreen(code: s.pathParameters['code']!)),
        GoRoute(path: '/go/:slug', builder: (_, s) => CampaignResolverScreen(slug: s.pathParameters['slug']!)),
        StatefulShellRoute.indexedStack(
          builder: (_, __, shell) => HomeShell(shell: shell),
          branches: [
            StatefulShellBranch(routes: [GoRoute(path: '/home', builder: (_, __) => const HomeScreen())]),
            StatefulShellBranch(routes: [
              // /shop/p/:id -> Back pops to /shop (synthesized stack)
              GoRoute(path: '/shop', builder: (_, __) => const ShopScreen(), routes: [
                GoRoute(path: 'p/:id', builder: (_, s) => ProductScreen(
                    id: s.pathParameters['id']!, variant: s.uri.queryParameters['variant'])),
              ]),
            ]),
            StatefulShellBranch(routes: [
              GoRoute(path: '/account', builder: (_, __) => const AccountScreen(), routes: [
                GoRoute(path: 'orders/:id', builder: (_, s) => OrderScreen(id: s.pathParameters['id']!)),
              ]),
            ]),
          ],
        ),
      ],
    );
```

With the coordinator in charge, the redirect is a **second line of defence** for in-app navigation and web. It is not the main link gate. On web (no `app_links`), the redirect plus the `from` param is the pending-link mechanism.

Wiring in `main.dart`:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final auth = AuthState();             // ChangeNotifier implementing SessionGate
  final router = buildRouter(auth);     // created once
  final coordinator = DeepLinkCoordinator(
    router: router,
    session: auth,
    parser: DeepLinkParser(allowedHosts: {'example.com', 'www.example.com'}, customScheme: 'com.example.myapp'),
    pending: PendingLinkStore(),
    analytics: (e, p) => debugPrint('$e $p'), // replace with your analytics
  );
  runApp(MaterialApp.router(routerConfig: router));
  unawaited(auth.restore());            // reads secure storage, then notifies
  await coordinator.start();
}
```

### A.8 Parser and store unit tests

```dart
// test/deep_link_parser_test.dart (sketch): parse() maps each allowed https host + custom scheme to the
// right target; rejects foreign hosts, http, bad ids, oversized input -> UnknownLink; strips utm_* into
// .utm; safeRelativeLocation rejects absolute, '//' and '\\' paths. PendingLinkStore: take() returns
// once then null, honours ttl (24 h for InviteLink) with an injected clock, clear() on sign-out.
```

### A.9 Android install referrer (deferred, first launch only)

A small platform channel. Use a maintained plugin instead if the project already has one.

```kotlin
// MainActivity.kt (sketch; dependency com.android.installreferrer:installreferrer): in configureFlutterEngine
// register MethodChannel "app/install_referrer"; on "get" build InstallReferrerClient, startConnection, and in
// onInstallReferrerSetupFinished return installReferrer.installReferrer when the code is OK (else null),
// then endConnection(). Reply exactly once.
```

```dart
// lib/deep_links/deferred_link.dart (sketch) — Future<Uri?> consumeDeferredLink():
// 1. SharedPreferences flag 'deferred_link_consumed': return null if set; set it BEFORE reading (never re-apply).
// 2. iOS: return null (server-side token / account-based match instead).
// 3. Android: invoke MethodChannel('app/install_referrer').get; Uri.splitQueryString(raw)['deep_link']
//    (Play URL: &referrer=deep_link%3Dhttps%253A%252F%252Fexample.com%252Fp%252F123) -> Uri.tryParse;
//    PlatformException -> null.
// After onboarding: if (uri != null) coordinator.handleUri(uri, LinkSource.deferred);
```

The deferred URI goes through the same parser. A forged referrer (anyone can craft a Play URL) gets the same validation as any other link.
