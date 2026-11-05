---
name: react-native-deep-links
description: Use when implementing, reviewing, or debugging deep links in React Native and Expo — iOS universal links and Android App Links vs custom URL schemes, apple-app-site-association and assetlinks.json hosting and verification (incl. Play App Signing SHA-256), Expo app.json `scheme` / `ios.associatedDomains` / `android.intentFilters`, bare RN native setup (AppDelegate RCTLinkingManager, AndroidManifest intent filters), the `Linking` API and expo-linking, React Navigation `linking` config, Expo Router file-based links and `+native-intent`, URL design, back-stack synthesis, cold/warm starts, links that arrive before login or onboarding, account-aware links, vendor-neutral deferred deep links (install referrer, server-side matching, Firebase Dynamic Links migration), push/widget/shortcut/QR/OAuth entry points, fallbacks, security, analytics, and testing. Siblings: ios-deep-links (Swift), android-deep-links (Kotlin), flutter-deep-links; related: react-native-multi-account, react-native-push-notifications.
---

# React Native Deep Links Skill

Guide for building, auditing, and testing **deep links** in React Native and Expo consumer apps: marketing links, shared content links, email links, password-reset and magic-login links, QR codes, push-notification taps, OAuth redirects. Covers **link types**, **domain verification**, **app configuration (Expo managed and bare RN)**, **URL design**, **routing into React Navigation / Expo Router**, **app states and pending links**, **account awareness**, **deferred deep links**, **fallbacks**, **security**, **analytics**, and **testing**. Siblings: **ios-deep-links**, **android-deep-links**, **flutter-deep-links**. Related: **react-native-multi-account** (links for another account/portal), **react-native-push-notifications** (notification taps), **react-native-in-app-purchase** / **react-native-revenuecat-iap** (paywall/offer links), **react-native-list-ux** (list targets and scroll restoration).

Deep-link plumbing changes across releases. **Verify fast-moving APIs against current docs** and check `package.json`, `app.json`/`app.config.ts`, and the native projects before writing code. Version gates to keep in mind:

- **Expo Router** v3 (SDK 50) introduced `+native-intent`; v4 (SDK 52) and v5 (SDK 53) changed `Stack.Protected`, redirect, and `router.dismissTo` behaviour. Check which one is installed.
- **React Navigation 7** added the **static config API** where `linking: { enabled: 'auto' }` generates paths from screen names; the dynamic (`linking.config.screens`) API still works. v6 → v7 also changed `navigate` semantics (use `popTo` for "go back to existing", `navigate` no longer goes back by default).
- **expo-linking**: `Linking.useURL()` exists for years; newer SDKs add `Linking.useLinkingURL()` (returns the initial URL and later ones without the "null first render" race). Confirm in the installed SDK.
- **RN 0.77+** bare templates use a **Swift `AppDelegate`**; older ones are Objective-C (`AppDelegate.mm`). Copy the matching snippet.
- **Android 12+ (API 31)**: unverified `https` links open in the browser, not a chooser. Verification is per-host and needs `android:autoVerify="true"`.
- **Firebase Dynamic Links** stopped serving on **25 Aug 2025**. Any `page.link` / custom FDL domain is dead; see §9.4.
- **Expo Go** only understands `exp://` URLs and cannot receive universal links / App Links for your domain. Test real links in a **development build** (`expo-dev-client`) or a release build.

## 0. Mental model

```
  user taps https://example.com/p/123?utm_source=mail
        │
        ├─ OS: is this domain verified for an installed app?
        │     iOS:   AASA from Apple CDN  ── associatedDomains entitlement
        │     And:   assetlinks.json      ── intent-filter autoVerify
        │        no ──► browser ──► web page (fallback, smart banner, store)
        │        yes
        ▼
  native entry  (iOS continueUserActivity / openURL, Android Intent VIEW)
        │   RCTLinkingManager / ReactActivity forward the URL
        ▼
  JS: Linking.getInitialURL() (cold)  |  'url' event (warm)
        │   (Expo Router: +native-intent redirectSystemPath() rewrites first)
        ▼
  parseLink(url) ──► Route | Unknown        pure, unit-tested, allow-listed
        │
        ├─ gate: onboarding done? signed in? right account/portal?
        │      no ──► PendingLinkStore.save(route) ──► login ──► resume
        ▼
  navigate: synthesize back stack (Home ► List ► Detail), push vs replace
        ▼
  screen loads by id; 404/403 ──► friendly fallback, never a crash
        ▼
  analytics: link_open {source, campaign, route, state: cold|warm, matched}
```

One parser, one router entry point, one pending-link store. Every source (URL, push, widget, shortcut, QR, deferred) becomes the same `Route` value before navigation.

## 1. First, establish context

Find out from the codebase, or ask:

- **Link types in use**: universal links / App Links on `https://`, a custom scheme (`myapp://`), both? Which domains and subdomains (`example.com`, `www.example.com`, `go.example.com` short links, regional domains)? Who owns the web server/CDN for each?
- **Expo or bare**: managed (CNG, `npx expo prebuild`) / bare with `ios/` + `android/` committed / brownfield? Expo SDK and RN version? Dev build available (not only Expo Go)?
- **Router**: Expo Router (file-based) or React Navigation (v6 / v7, static or dynamic)? Any custom navigator? Tabs with their own stacks?
- **Destinations**: list every linkable screen. For each: requires login? requires onboarding? requires a specific account/portal? Works logged-out (public content preview)?
- **Link producers**: marketing (email, ads, social), in-app share, transactional email (reset password, verify email, magic login), push payloads, QR codes printed on physical media (cannot be changed later), OAuth providers, partner apps.
- **Deferred**: must a new user land on the shared content after installing? Is there an existing attribution vendor or legacy Firebase Dynamic Links?
- **Accounts**: multi-account or multi-portal app? (→ **react-native-multi-account**)
- **Platforms**: iOS, Android, also web (Expo Router web / react-native-web share the same URLs)?
- **Signing**: Android signed by Play App Signing? EAS-managed credentials? Multiple flavors/bundle ids (dev, staging, prod)?

Write a **link spec** before coding:

```
path                     route             auth     account   back stack            web fallback
/p/:id                   Product           public   any       Home ► Product        product page
/orders/:id              OrderDetail       login    owner     Home ► Orders ► Order  login → order
/invite/:token           AcceptInvite      login*   target    Home ► Invite          invite landing
/reset-password?t=…      ResetPassword     logged-out only     Login ► Reset          reset form
/settings/notifications  NotifSettings     login    active    Settings ► Notif       app-only page
(*) logged-out users can preview, then sign up; token is single-use, server-validated
```

## 2. Link types

| | Universal links (iOS) / App Links (Android) | Custom scheme (`myapp://`) |
|---|---|---|
| Ownership | Proven by files on your HTTPS domain | Anyone can register the same scheme |
| Not installed | Opens the web page (graceful) | Error / nothing happens |
| Shareable on web/social/email | Yes | Often stripped or unclickable |
| Prompt | None (opens directly when verified) | iOS may show "Open in app?" from Safari |
| Setup | Domain files + entitlement/intent filter | `scheme` only |
| Use for | Everything user-facing | OAuth redirects (when not using https), dev tools, app-to-app callbacks, Expo dev |

- [ ] **Default to https links** for anything a human shares, receives by email, or scans
- [ ] Keep a **custom scheme** too (Expo needs `scheme` for dev builds, `Linking.createURL`, and many auth SDKs), but never print it in marketing material
- [ ] Every https deep link has a **real web page** at the same URL (content, a landing page, or a store redirect page) — the link must work without the app
- [ ] Do not use **Android intent:// URLs** or iOS-only tricks in shared links; they break on the other platform and on desktop
- [ ] Decide on **short links** (`go.example.com/abc`) vs canonical links. Short links that redirect in the browser before reaching the app domain can break universal links (§10.3); prefer the short domain itself to be verified and resolved in-app

## 3. Domain verification

### 3.1 apple-app-site-association (AASA)

- [ ] Served at `https://<domain>/.well-known/apple-app-site-association` (no `.json` extension), also fine at the root path; **one per host** — `www.` and apex are different hosts
- [ ] HTTPS with a valid certificate, **HTTP 200 directly — no redirects**, `Content-Type: application/json`, < 128 KB
- [ ] Not behind auth, geo-blocking, bot protection, or a WAF challenge (Apple's CDN fetches it, not the device)
- [ ] `appIDs` uses **`<TeamID>.<bundleId>`**; add every bundle id that should open (prod, staging) — or host different files on staging domains
- [ ] Use the modern `components` syntax (iOS 13+) with `exclude: true` for paths the web must keep (`/blog/*`, `/help/*`, `/api/*`, `/auth/callback` if web-handled)
- [ ] Apple fetches AASA via **its CDN** (`https://app-site-association.cdn-apple.com/a/v1/<domain>`) at install/update and periodically; changes can take **hours to days**. Check what the CDN currently serves, not just your origin
- [ ] For development, `applinks:example.com?mode=developer` in the entitlement bypasses the CDN on devices with Developer Mode / associated-domains development enabled. Remove it for release
- [ ] Also include `webcredentials` if the app uses password AutoFill / passkeys on the same domain

### 3.2 assetlinks.json (Android)

- [ ] Served at `https://<domain>/.well-known/assetlinks.json`, HTTP 200, no redirects, `Content-Type: application/json`, publicly reachable
- [ ] `relation: ["delegate_permission/common.handle_all_urls"]`, `target.namespace: "android_app"`, `package_name`, `sha256_cert_fingerprints`
- [ ] Fingerprints include **the Play App Signing key** (Play Console → Test and release → App integrity → App signing) — *not only* your upload key. Builds from the Play Store are re-signed with Google's key; missing it is the #1 "works in debug/internal APK, fails from Play" bug
- [ ] Also add the **upload key** / EAS keystore (for sideloaded/internal builds) and **debug key** on a non-production domain or file if you test App Links on debug builds
- [ ] EAS: `eas credentials -p android` shows the keystore SHA-256; Play Console shows the app signing SHA-256. Bare: `keytool -list -v -keystore <file>` or `./gradlew signingReport`
- [ ] Verification is **per host in every intent filter that has `autoVerify`**: if *any* listed host fails, older Android versions (≤11) fail them all. Keep the host list tight
- [ ] Multiple apps/flavors can be listed in one file as separate statements

### 3.3 Hosting rules and CDN

- [ ] Files served by the edge/CDN with a modest cache (e.g. `max-age=3600`); purge on change. Do not let an SPA catch-all return `index.html` for `/.well-known/*`
- [ ] No cookie banners, interstitials, or locale redirects (`/en/.well-known/...`) on these paths
- [ ] Monitor them: a synthetic check that fetches both files from every domain, asserts status 200, content type, and that the expected app ids/fingerprints are present. Breakage is silent otherwise
- [ ] Validate: Apple's AASA via the Apple CDN URL above; Google's Statement List API (`https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://example.com&relation=delegate_permission/common.handle_all_urls`); on device `adb shell pm get-app-links <package>`

## 4. App setup

### 4.1 Expo (managed / CNG) — `app.json` / `app.config.ts`

- [ ] `scheme: "myapp"` (lowercase, unique-ish, no underscores). Multiple schemes allowed as an array
- [ ] `ios.associatedDomains: ["applinks:example.com", "applinks:www.example.com"]` (add `webcredentials:` if needed). Requires the Associated Domains capability on the App ID — EAS Build syncs it when it manages credentials
- [ ] `android.intentFilters` with `autoVerify: true`, `action: "VIEW"`, `category: ["BROWSABLE", "DEFAULT"]`, `data` entries per host and path prefix
- [ ] Per-environment config in `app.config.ts`: staging bundle id + staging domain, prod bundle id + prod domain; never ship staging domains in prod
- [ ] Run `npx expo prebuild --clean` (or a fresh EAS build) after changing these; config plugins write native files only at prebuild time. A JS-only/OTA update **cannot** add a domain or scheme
- [ ] Expo Router: links work automatically from the file structure; `expo-router` uses the first `scheme` and reads `origin` (`plugins: [["expo-router", { origin: "https://example.com" }]]`) for web/server URLs

### 4.2 Bare React Native — iOS

- [ ] Xcode → Signing & Capabilities → **Associated Domains**: `applinks:example.com`. This writes `com.apple.developer.associated-domains` into the `.entitlements` file; check it is in the **Release** configuration too
- [ ] Custom scheme: `Info.plist` → `CFBundleURLTypes` → `CFBundleURLSchemes = [myapp]`
- [ ] `AppDelegate` forwards **both** callbacks to `RCTLinkingManager`: `application(_:open:options:)` (custom schemes) and `application(_:continue:restorationHandler:)` (universal links). See Appendix A.8
- [ ] Scene-based apps / brownfield with `UISceneDelegate`: forward `scene(_:openURLContexts:)` and `scene(_:continue:)` too, plus the cold-start `connectionOptions` (URLs and user activities), or cold starts lose the link
- [ ] Third-party SDKs that swizzle or override these callbacks (auth, attribution, push) must still forward to `RCTLinkingManager`; return `true` only when handled

### 4.3 Bare React Native — Android

- [ ] `AndroidManifest.xml`: `MainActivity` keeps `android:launchMode="singleTask"` (RN template default) so warm links hit `onNewIntent` instead of creating a second activity and a second JS runtime
- [ ] Separate intent filters for the custom scheme and for https (the https one carries `android:autoVerify="true"`); see Appendix A.8
- [ ] `ReactActivity` already forwards `onNewIntent` to `Linking`; if you override `onNewIntent`, call `super` and `setIntent(intent)`
- [ ] Use `pathPrefix` / `pathPattern` / `pathAdvancedPattern` (API 31+) to claim only the paths the app handles. Claiming `/` hands every page of the site to the app
- [ ] Product flavors: `applicationIdSuffix` changes the package name — the fingerprints statement must list each package that should verify

### 4.4 JS entry — `Linking` / expo-linking

- [ ] **Cold start**: `Linking.getInitialURL()` once at startup (can be `null`; on Android it returns the launch intent's URL until the activity is recreated)
- [ ] **Warm**: `Linking.addEventListener('url', ({ url }) => …)`; keep the subscription and call `.remove()` on cleanup (the old `removeEventListener` is gone)
- [ ] Never handle the same URL twice: initial URL and the first `url` event can overlap on some platforms/SDK combos; dedupe by URL + timestamp (Appendix A.4)
- [ ] expo-linking: `Linking.createURL('/p/123')` produces `exp://…` in Expo Go, `myapp://…` in builds — fine for dev and OAuth redirects, not for shared links. `Linking.parse(url)` gives `{ scheme, hostname, path, queryParams }`
- [ ] With React Navigation's `linking` prop or Expo Router, **let the router own** `getInitialURL`/`subscribe` and plug your logic in there rather than adding a second listener that also navigates

## 5. URL design

- [ ] Paths are **resources, not screens**: `/p/123`, `/orders/987`, `/u/jane`; not `/ProductDetailScreen?id=123`. Screen refactors must not break printed links
- [ ] Stable ids (server ids or slugs + id: `/p/123-red-sneakers`; parse the numeric id, ignore the slug)
- [ ] Optional **version** only if you expect breaking changes: `/v2/...` or a `v=` param; prefer additive changes and keep old paths mapped forever (QR codes and emails live for years)
- [ ] Encode every param with `encodeURIComponent`; decode once. Watch `+` vs `%20` in query strings, emoji/Unicode slugs, and double-encoding by email click trackers
- [ ] Keep links **short and meaningful**; no PII (emails, phone numbers) or long-lived secrets in URLs — they land in logs, analytics, browser history, and referrers
- [ ] One-time tokens (reset, magic login, invites): short TTL, single-use, server-validated; the app exchanges them over HTTPS immediately and strips them from state/logs
- [ ] Marketing links: stable path + UTM params (`utm_source`, `utm_medium`, `utm_campaign`, `utm_content`); the router ignores unknown params instead of failing
- [ ] Same URL works on **web, iOS, Android** (and Expo Router web). Write the path table once and share it (Appendix A.1)

## 6. Routing

### 6.1 Parse first, navigate second

- [ ] A pure `parseLink(url): ParsedLink` function maps URL → typed `Route` or `{ kind: 'unknown' }`; allow-list hosts and schemes; validate params (id formats, enums, lengths). Unit-test it heavily (Appendix A.2–A.3)
- [ ] The same `Route` type is produced by push payloads, widgets, shortcuts and deferred links
- [ ] Navigation is a separate function `navigateTo(route, { source })` that knows the stack rules

### 6.2 React Navigation `linking`

- [ ] `linking = { prefixes: ['myapp://', 'https://example.com', 'https://www.example.com'], config: { screens: { … } } }` passed to `NavigationContainer`
- [ ] Nested navigators mirror the nesting: `{ Tabs: { screens: { HomeTab: { initialRouteName: 'Home', screens: { Home: '', Product: 'p/:id' } } } } }`. `initialRouteName` in the config is what **synthesizes the back stack** (Product has Home beneath it, so Back doesn't exit the app)
- [ ] `parse: { id: (v) => v }` / `stringify` for typed params; reject garbage in a custom `getStateFromPath` that falls back to a `NotFound` screen (`'*'` path)
- [ ] `getInitialURL` / `subscribe` overrides are where to add: pending-link gating, push-notification URLs, dedupe, analytics, deferred links (Appendix A.6)
- [ ] `filter: (url) => !url.includes('/auth/callback')` to keep OAuth/redirect URLs that an auth library handles out of the navigator
- [ ] v7 static API: `linking: { enabled: 'auto', prefixes }` auto-generates paths from screen names — convenient but couples URLs to screen names; set explicit `linking: { path: 'p/:id' }` per screen for public links
- [ ] Use `NavigationContainer`'s `onReady` before calling `navigationRef.navigate`; queue links that arrive earlier

### 6.3 Expo Router

- [ ] Files define the URLs: `app/p/[id].tsx` handles `/p/123`; `useLocalSearchParams<{ id: string }>()` reads params (always strings — validate)
- [ ] `app/+not-found.tsx` for unknown paths; make it helpful (search, home), not a dead end
- [ ] `app/+native-intent.tsx` exports `redirectSystemPath({ path, initial })` to **rewrite incoming native URLs** before routing: map legacy paths, short links, marketing aliases, strip tracking params, route deferred/unknown to `/`. It must be fast and must never throw (return a safe path on error)
- [ ] Back-stack synthesis: `export const unstable_settings = { initialRouteName: 'index' }` in a stack's `_layout.tsx` makes deep links into `/p/123` load `index` beneath it. Check the current API name in the installed version (`withAnchor` on `router.push` exists in newer versions for imperative navigation)
- [ ] Gating: route groups + `Stack.Protected guard={isSignedIn}` (SDK 53+) or `<Redirect href={`/login?next=${encodeURIComponent(path)}`} />` in a layout; in both cases preserve the original target (§7.2)
- [ ] Imperative navigation from push/widgets: `router.push(href)` (adds), `router.replace(href)` (swaps current), `router.navigate(href)` (reuses existing route when possible), `router.dismissTo` for modals
- [ ] Typed routes (`experiments.typedRoutes`) catch broken internal hrefs at compile time; external URLs still need runtime validation

### 6.4 Stack rules

| Situation | Behaviour |
|---|---|
| Cold start into detail | Synthesize parent stack (Home ► List ► Detail); Back goes to parent, not out of the app |
| Warm, same screen type already on top with same id | No-op or refresh; don't push a duplicate |
| Warm, same screen type, different id | Push (user can go back to what they were reading) |
| Warm, different tab | Switch tab, then push inside that tab's stack; keep other tabs' stacks |
| Link to a top-level destination (Home, Inbox) | Navigate/reset to it, don't push copies |
| Auth-flow links (reset password, verify email) | Replace; the old screen should not sit below |
| Modal open (§7.4) | Dismiss or stack per spec, never leave a half-filled form silently lost |

## 7. App states

### 7.1 Cold, warm, foreground

- [ ] **Cold**: link available at launch via `getInitialURL`; app must finish splash, restore session, hydrate stores, *then* navigate. Keep the splash (`expo-splash-screen` `preventAutoHideAsync`) until the first route is decided to avoid a Home flash
- [ ] **Warm (background)**: `url` event while the JS runtime lives; navigation container is ready
- [ ] **Foreground**: user taps a link inside the app (in-app browser, WebView, chat message). Route internally instead of `Linking.openURL` to your own domain
- [ ] Android process death: activity recreated, `getInitialURL` may return the *original* launch URL again — dedupe by intent/URL + handled flag so you don't re-open an old link
- [ ] OTA updates on launch (`expo-updates` reload) restart JS; persist pending links before a reload

### 7.2 Link before login or onboarding

- [ ] Gate checks in one place: `needsOnboarding`, `needsAuth(route)`, `needsAccount(route)`
- [ ] If gated, **save the route** in a `PendingLinkStore` (memory + persisted with a TTL, e.g. 30–60 min, because sign-up may involve email verification and app restarts), then show onboarding/login
- [ ] After success, `consume()` the pending route **once** and navigate; clear it on sign-out and on expiry
- [ ] Logged-out-only links (reset password) while signed in: ask ("You're signed in as X. Sign out to reset?") rather than silently ignoring
- [ ] Public content (product pages) should not force login; show it, and ask for login only on actions
- [ ] Do not put pending one-time tokens in persisted storage longer than necessary; store the route, re-fetch the token state from the server

### 7.3 Splash, permissions, forced updates

- [ ] Forced-update or maintenance screens keep the pending link and resume after
- [ ] Permission prompts or ATT prompts at launch must not swallow the link; route first, prompt later

### 7.4 Link while a modal or flow is open

- [ ] Checkout, compose, or a form with unsaved input: ask before leaving, or open the target on top and return; never discard silently
- [ ] Transient modals (menus, sheets, alerts): dismiss, then navigate
- [ ] Video/call in progress: keep it (PiP) or queue the link

## 8. Account awareness

- [ ] Links can carry an **account/portal hint** (`/w/:workspaceId/…`, or the server resolves which account owns `/orders/987`)
- [ ] Target belongs to the active account/portal → navigate
- [ ] Target belongs to another signed-in account or another portal of this account → **confirm and switch** ("Open in Acme workspace?"), then navigate after the switch completes; never show data from account B inside account A's session
- [ ] Target belongs to no signed-in account → offer "Sign in to another account", keep the link pending
- [ ] Hand off the switch itself to **react-native-multi-account** (session teardown, scoped stores, navigation reset); the deep-link layer only resolves the target and passes `(accountId, portalId, route)`
- [ ] Server returns 403 for the active account → don't leak the reason; offer switching if another account might have access

## 9. Deferred deep links (vendor-neutral)

Goal: a user without the app taps a link, installs, opens — and lands on the linked content.

### 9.1 Mechanisms

| Mechanism | Platform | Accuracy | Privacy / review notes |
|---|---|---|---|
| **Play Install Referrer** (`referrer` param on the Play Store URL, read via the Install Referrer API) | Android (Play) | Deterministic | Clean; first-party; no extra permission |
| **Server-side match**: link click creates a short-lived record (token), the app later claims it via a deterministic id it got from the web (e.g. login, email) | Both | Deterministic when an identity exists | Disclose; expire quickly |
| **Clipboard**: web page copies a token, app reads clipboard on first launch | iOS mostly | Deterministic if the user didn't copy something else | iOS shows a paste notice/permission; feels creepy; ask or use `UIPasteControl`-style explicit paste |
| **Probabilistic fingerprinting** (IP + device model + OS + time window) | Both | Probabilistic, wrong on shared networks | Apple prohibits fingerprinting for tracking; disclose in privacy labels; avoid for sensitive targets |
| **App Store custom product pages / campaign links** | iOS | Campaign-level only | No per-user content |

- [ ] Prefer **deterministic** mechanisms; use probabilistic only for low-stakes content (a public product page), never for account-specific or authenticated targets
- [ ] Claim deferred data **once**, on first launch, with a short window (e.g. 1 h after click / 24 h max), then mark as consumed
- [ ] Run deferred resolution in parallel with onboarding; treat the result like any other link (same parser, same gates)
- [ ] React Native: Play Install Referrer through a small native module or a maintained community package (verify New Architecture support); iOS has no equivalent — clipboard or server match

### 9.2 Evaluating a deferred-link provider or building in-house

- [ ] Data ownership and retention, where data is processed, SDK size, New Architecture/Expo config-plugin support
- [ ] Uses OS mechanisms (install referrer) first, documents any fingerprinting, offers a no-fingerprinting mode
- [ ] Supports your own verified domain (not only a provider subdomain) so links survive a provider change
- [ ] Fallback behaviour configurable (web page, store), link analytics exportable
- [ ] Exit plan: your links resolve on *your* domain; the provider is an implementation detail

### 9.3 Building in-house (minimal)

- [ ] Link service on your domain: `GET /p/123` → if mobile and not installed (app didn't intercept), render page with "Get the app" → store link carrying `referrer=link_token` (Android) and set a first-party server record keyed by token
- [ ] App first launch: Android reads install referrer → `link_token` → `GET /deferred/{token}` → route. iOS: offer "Continue where you left off?" with an explicit paste or post-login server match

### 9.4 Firebase Dynamic Links migration

- [ ] FDL was **shut down on 25 Aug 2025**; `*.page.link` and FDL custom domains no longer resolve. Search the codebase for `page.link`, `dynamicLinks()`, `@react-native-firebase/dynamic-links`, and remove the SDK
- [ ] Replace with **plain universal links / App Links on your own domain** (§3–4) plus the deferred mechanism of §9.1
- [ ] Map old FDL short links that still circulate (printed QR codes, old emails) if you controlled a custom domain: host a redirect service on that domain that resolves old codes to new canonical URLs; `page.link` links cannot be recovered
- [ ] Keep handling legacy long-link formats in `parseLink` / `+native-intent` (e.g. `?link=` param unwrapping) for a while

## 10. Other entry points

### 10.1 Push notification taps

- [ ] Payload carries a URL (`data.url`) or a typed route; the tap handler runs it through the **same parser** (→ **react-native-push-notifications**)
- [ ] Cold start from a notification: expo-notifications `useLastNotificationResponse()` / `getLastNotificationResponseAsync()`; Notifee `getInitialNotification()`; RN Firebase `getInitialNotification()`. With React Navigation, merge it into `linking.getInitialURL` (Appendix A.6)
- [ ] Notification for another account/portal → §8

### 10.2 Widgets, shortcuts, app actions

- [ ] iOS widgets use `widgetURL` / `Link` with your scheme or https URL; Android widgets use a `PendingIntent` with a VIEW intent — both arrive through `Linking`
- [ ] Home-screen quick actions (`expo-quick-actions` or similar): map the action id/params to a `Route`; handle cold start (initial action) as well as warm
- [ ] Siri / App Intents / Android App Actions: same parser

### 10.3 Email click-tracking and in-app browsers

- [ ] Email providers wrap links in tracking redirects (`click.mail.example.net/…`). The tap opens the **tracking domain**, so universal links don't fire; the browser then redirects to your domain and iOS may stay in Safari
- [ ] Fixes: use a **branded click-tracking domain you own** and verify it in AASA/assetlinks (exclude nothing the tracker needs), or disable tracking for deep-link CTAs; then unwrap the tracking URL in `+native-intent` / `parseLink`
- [ ] Social/in-app browsers (some messaging and social apps) may not hand off universal links; the web page needs an "Open in app" button pointing to the same https URL (a user tap in Safari to a *different* domain can trigger it) and a store fallback
- [ ] iOS: links typed in the address bar or tapped on the **same domain** in Safari open the web page; that's by design. Long-press → "Open in <App>" re-enables it if a user disabled it

### 10.4 QR codes

- [ ] Encode https URLs, never custom schemes; include a source param (`?src=qr-poster-2026`)
- [ ] Printed codes are permanent: the path must be supported forever; route retired campaigns to a sensible screen

### 10.5 OAuth / auth redirects

- [ ] Use the auth library's flow (`expo-auth-session` / `expo-web-browser` `openAuthSessionAsync`, `react-native-app-auth`); PKCE always
- [ ] Redirect URI: a dedicated path (`myapp://oauth/callback` or `https://example.com/oauth/callback` verified) **excluded** from the navigator (`linking.filter`, or a handled route in `+native-intent` that returns the current path)
- [ ] Validate `state`; never accept tokens in the fragment from a link the app didn't initiate

## 11. Fallbacks

- [ ] **App not installed**: the web page renders content or a landing page with store badges; iOS Smart App Banner meta tag (`apple-itunes-app` with `app-argument=` the same URL); Android store link with `referrer` (§9)
- [ ] **Unknown path**: Expo Router `+not-found` / React Navigation `NotFound` (`'*'`) screen with search/home; log `link_unmatched` with the path (no PII)
- [ ] **Expired or used token**: friendly screen ("This link has expired") with a resend action
- [ ] **Removed route / content**: keep the path mapped to a replacement or a "no longer available" screen; never crash on a 404/410
- [ ] **Old app version** without the route: web fallback or "Update the app" screen — when the server knows the minimum version for a path, show the update prompt
- [ ] Web-only pages (help, legal) excluded from AASA/assetlinks or opened via `expo-web-browser` from the app

## 12. Security

- [ ] **Treat every link as untrusted input**: validate types, lengths, enums, id formats in `parseLink`; reject unknown hosts and schemes
- [ ] **No destructive or state-changing actions from a link alone**: links open screens; the user confirms (delete, pay, transfer, change email, accept invite, follow)
- [ ] **Open redirects**: never navigate to or `Linking.openURL` a `?redirect=`/`?next=` value unless it's a relative in-app path that passes the allow-list. Never load arbitrary URLs from a link in a WebView with JS bridges
- [ ] **Custom scheme hijacking**: another app can register `myapp://`; don't send secrets (auth codes without PKCE, tokens) to a custom scheme; prefer verified https redirects for OAuth when the provider supports them
- [ ] Magic-login / reset tokens: single-use, short TTL, bound server-side to the requesting device or email session where possible; confirm the account before logging in ("Sign in as jane@…?") to stop login-CSRF
- [ ] Don't log full URLs with tokens to analytics or crash reporters; redact query params by allow-list
- [ ] Android: exported activities only for intended filters; don't forward arbitrary `Intent` extras into JS
- [ ] Rate-limit and monitor link-claim endpoints (deferred tokens, invites)

## 13. Analytics

- [ ] One `link_open` event: `{ source: 'universal'|'app_link'|'scheme'|'push'|'widget'|'shortcut'|'qr'|'deferred', route, matched: boolean, app_state: 'cold'|'warm'|'foreground', gated: 'none'|'auth'|'onboarding'|'account', utm_* }`
- [ ] `link_resumed` after login/onboarding; `link_fallback` for not-found/expired; deferred `deferred_claimed` with mechanism
- [ ] Capture UTM params at open, attribute the session/install, then strip them from navigation state
- [ ] Compare server-side web hits on deep-link paths vs `link_open` in app: a sudden gap often means AASA/assetlinks broke
- [ ] Respect consent (ATT on iOS for cross-app tracking, regional consent); first-party link analytics don't need IDFA

## 14. Testing

### 14.1 Tools

1. **Unit**: Jest tests for `parseLink`, `redirectSystemPath`, stack synthesis and the pending-link store (Appendix A.3). These catch most bugs cheaply
2. **iOS Simulator**: `xcrun simctl openurl booted "https://example.com/p/123"` and `"myapp://p/123"`. Universal links in the simulator need the entitlement and AASA reachable (or `?mode=developer`)
3. **Android**: `adb shell am start -W -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "https://example.com/p/123" <package>`; check verification with `adb shell pm get-app-links <package>`, re-verify with `adb shell pm verify-app-links --re-verify <package>`, and for local testing `adb shell pm set-app-links --package <package> 0 all` / `pm set-app-links-user-selection`
4. **Expo**: `npx uri-scheme open "myapp://p/123" --ios|--android`; in Expo Go use the `exp://<host>:8081/--/p/123` form; test https links only in dev/release builds
5. **Validators**: Apple CDN URL for AASA, Google Digital Asset Links Statement List API, `curl -sI` for status/content-type/redirects
6. **Real devices**: Notes / Messages / Mail taps (not the Safari address bar), email via the real ESP with click tracking, QR from the camera app, Play internal-testing build (Play-signed)
7. **E2E**: Maestro `openLink` / Detox `device.openURL` + `launchApp({ url })` for cold start

### 14.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| D1 | Cold start via universal link (iOS) | Splash → Detail with parent stack; Back → parent |
| D2 | Cold start via App Link (Android, Play-signed build) | Same as D1; no browser, no chooser |
| D3 | Warm link while on another tab | Switches tab, pushes Detail; other tab's stack intact |
| D4 | Warm link to the screen already open with same id; same URL delivered twice (initial + event, process death) | No duplicate push; handled once |
| D5 | Custom scheme link | Opens same route as https equivalent |
| D6 | Link logged out, auth required | Login → after success lands on Detail; pending cleared |
| D7 | Link during onboarding | Onboarding completes → Detail |
| D8 | Link pending, app killed during sign-up email verification, reopened within TTL | Resumes target |
| D9 | Pending link past TTL, or user signs out with a pending link | Ignored / cleared, normal home |
| D11 | Link for another signed-in account, or another portal of the active account | Confirm switch → switched → Detail |
| D13 | Link for an account not signed in | Offer sign-in; target after login |
| D14 | Unknown path | Not-found screen; `link_unmatched` logged |
| D15 | Malformed params (`/p/abc`, very long, unicode, `%00`) | Not-found or safe fallback; no crash |
| D16 | Expired one-time token / content deleted (404) | Expired screen with resend / "No longer available" |
| D18 | Link while checkout modal has input | Confirm dialog; no silent loss |
| D19 | Push tap cold start with URL; widget / quick action cold and warm | Same route + stack as D1 |
| D21 | Email link via click tracker | App opens (branded tracking domain) or web page with working "Open in app" |
| D22 | Not installed | Web page renders; store link carries referrer |
| D23 | Deferred: install via link, first open | Lands on content once; second launch normal |
| D24 | `?next=https://evil.example` | Ignored; stays in app |
| D25 | Link that would delete/pay | Opens confirmation screen only |
| D26 | OAuth redirect | Handled by auth flow, not routed as a deep link |
| D28 | AASA/assetlinks served with redirect or wrong content type (staging check) | Monitoring alert fires |
| D29 | Link opened in Safari address bar / same-domain web tap | Web page (expected), banner offers app |
| D30 | OTA update reload at launch with link | Link survives the reload |

### 14.3 Common bugs to look for in review

- Missing **Play App Signing** SHA-256 in assetlinks → works on local/internal APK, opens browser from the Play Store
- AASA/assetlinks behind a redirect (`apex → www`, `http → https`, locale), served as `text/html` by an SPA fallback, or blocked by bot protection; AASA changes tested only against the origin while Apple's CDN still serves the old file
- `associatedDomains` / `intentFilters` added to `app.json` but shipped via OTA update or without `prebuild` → native config unchanged; universal links tested in **Expo Go** and declared broken
- Bare iOS `AppDelegate` forwards `openURL` but not `continueUserActivity` → custom scheme works, universal links don't (or scene delegate ignores `connectionOptions` on cold start)
- Android `launchMode` changed from `singleTask` → second activity/JS instance on warm links
- Claiming the whole domain (`pathPrefix="/"`, AASA `"/": "*"`) → help/legal/blog pages hijacked by the app
- Two listeners (custom `Linking.addEventListener` + router's `linking`) both navigating → double push
- No `initialRouteName` / `unstable_settings` → Back from a deep-linked detail exits the app
- Link before login dropped because navigation happened before the auth state hydrated; or pending link never cleared and re-opens on every launch
- `getInitialURL` handled again after Android activity recreation → old link reopens
- Params trusted (`useLocalSearchParams` as typed numbers) → crashes on `/p/abc`; `?next=` used as an open redirect
- Destructive actions (accept invite, unsubscribe, delete) executed on link open; tokens and emails logged with full URLs to analytics/crash tools
- `@react-native-firebase/dynamic-links` or `page.link` URLs still in code/email templates after the FDL shutdown
- Email click-tracking domain not owned/verified → email CTAs never open the app
- Staging domains or `?mode=developer` left in production entitlements

## 15. Output format

When asked to **review/audit**:
1. **Summary**: Expo SDK / RN version, Expo Router vs React Navigation (version, static/dynamic), link types and domains, the link spec table (§1)
2. **Verification status per domain**: AASA and assetlinks fetch results (status, content type, redirects, app ids, fingerprints incl. Play App Signing), plus `pm get-app-links` output if available
3. **Checklist results**: sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check, with file:line
4. **Critical issues first** (links open the browser, lost links on login, open redirects, actions from links, wrong account data) with fix code
5. **Testing gaps**: §14.2 scenarios not covered, plus the unit tests to add

When asked to **implement**: confirm versions and Expo vs bare, write the link spec, start from the Appendix (parser + tests first), configure domains and native setup, wire the router, then gating/pending links, then other entry points. List what must be verified on device: Play-signed build, real email via the ESP, cold start on both platforms, AASA via the Apple CDN.

When asked to **debug**: identify the failing layer from the mental model — OS didn't open the app (verification, entitlement/intent filter, tracking domain) vs app opened but wrong screen (parser, router config, gating, timing). Log the raw URL at the native boundary and at `getInitialURL`/`url` event, and compare.

---

## Appendix — Reference implementation (TypeScript)

Assumes Expo SDK 53+ (or bare RN 0.77+), `expo-linking`, either **React Navigation 7** or **Expo Router 5**, `react-native-mmkv` or `@react-native-async-storage/async-storage` for the pending-link store, Jest. Verify APIs against installed versions. Replace `example.com`, `myapp`, `com.example.app`, and `TEAMID`.

### A.1 Route types and config

```ts
// src/links/routes.ts
export type Route =
  | { name: 'home' } | { name: 'notificationSettings' }
  | { name: 'product'; id: string } | { name: 'order'; id: string }
  | { name: 'invite'; token: string } | { name: 'resetPassword'; token: string }
  | { name: 'workspaceItem'; workspaceId: string; itemId: string };

export type Requirement = 'none' | 'auth' | 'loggedOut';

// invite: preview logged-out; accepting requires auth + confirmation
export const ROUTE_REQUIREMENTS: Record<Route['name'], Requirement> = {
  home: 'none', product: 'none', order: 'auth', invite: 'none',
  resetPassword: 'loggedOut', notificationSettings: 'auth', workspaceItem: 'auth',
};

export const LINK_CONFIG = {
  schemes: ['myapp'],
  hosts: ['example.com', 'www.example.com'],
  // Hosts we unwrap (branded click-tracking, short links) — must also be verified domains.
  trackingHosts: ['click.example.com'],
} as const;

export type LinkSource =
  | 'universal' | 'scheme' | 'push' | 'widget' | 'shortcut' | 'qr' | 'deferred' | 'internal';

export type Attribution = Partial<Record<
  'utm_source' | 'utm_medium' | 'utm_campaign' | 'utm_content' | 'utm_term' | 'src', string>>;

export type ParsedLink =
  | { kind: 'route'; route: Route; attribution: Attribution }
  | { kind: 'external'; reason: 'oauth' } // handled by an auth library
  | { kind: 'unknown'; path: string; attribution: Attribution };
```

### A.2 Parser

```ts
// src/links/parseLink.ts
import { LINK_CONFIG, type Attribution, type ParsedLink, type Route } from './routes';

const ID = /^[0-9]{1,18}$/;
const SLUG_ID = /^([0-9]{1,18})(?:-[a-z0-9-]{0,80})?$/i;
const TOKEN = /^[A-Za-z0-9_-]{16,256}$/;
const WORKSPACE = /^[a-z0-9][a-z0-9-]{1,62}$/i;
const ATTRIBUTION_KEYS = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term', 'src'] as const;
const MAX_URL_LENGTH = 2048;

function safeUrl(raw: string): URL | null {
  if (!raw || raw.length > MAX_URL_LENGTH) return null;
  try { return new URL(raw); } catch { return null; }
}

function attributionFrom(params: URLSearchParams): Attribution {
  const out: Attribution = {};
  for (const key of ATTRIBUTION_KEYS) { const v = params.get(key); if (v && v.length <= 100) out[key] = v; }
  return out;
}

/** Returns path segments for a supported URL, or null if the origin is not ours. */
function segmentsFor(url: URL): string[] | null {
  const scheme = url.protocol.replace(/:$/, '').toLowerCase();
  let path: string;
  if (scheme === 'https') {
    if (!(LINK_CONFIG.hosts as readonly string[]).includes(url.hostname.toLowerCase())) return null;
    path = url.pathname;
  } else if ((LINK_CONFIG.schemes as readonly string[]).includes(scheme)) {
    // myapp://p/123 → host "p", path "/123"; myapp:///p/123 → host "", path "/p/123"
    path = `/${url.hostname}${url.pathname}`;
  } else {
    return null;
  }
  // invalid %-encoding becomes '\u0000' and fails validation below
  return path.split('/').filter(Boolean).map((s) => {
    try { return decodeURIComponent(s); } catch { return '\u0000'; }
  });
}

/** Unwrap a branded click-tracking or legacy wrapper: https://click.example.com/?u=<encoded> or ?link=<encoded>. */
function unwrap(url: URL, depth = 0): URL {
  if (depth > 2) return url;
  const isTracker = (LINK_CONFIG.trackingHosts as readonly string[]).includes(url.hostname.toLowerCase());
  const inner = isTracker ? url.searchParams.get('u') : url.searchParams.get('link');
  if (!inner) return url;
  const next = safeUrl(inner);
  return next ? unwrap(next, depth + 1) : url;
}

export function parseLink(raw: string): ParsedLink {
  const first = safeUrl(raw);
  if (!first) return { kind: 'unknown', path: '', attribution: {} };
  const url = unwrap(first);
  const attribution = attributionFrom(url.searchParams);
  const segs = segmentsFor(url);
  if (!segs) return { kind: 'unknown', path: '', attribution };

  const route = matchRoute(segs, url.searchParams);
  if (route === 'oauth') return { kind: 'external', reason: 'oauth' };
  if (route) return { kind: 'route', route, attribution };
  return { kind: 'unknown', path: `/${segs.join('/')}`.slice(0, 200), attribution };
}

function matchRoute(segs: string[], q: URLSearchParams): Route | 'oauth' | null {
  const [a, b, c, d] = segs;
  switch (segs.length) {
    case 0: return { name: 'home' };
    case 1: {
      const t = q.get('t');
      return a === 'reset-password' && t && TOKEN.test(t) ? { name: 'resetPassword', token: t } : null;
    }
    case 2: {
      const m = a === 'p' ? SLUG_ID.exec(b) : null;
      if (m) return { name: 'product', id: m[1] };
      if (a === 'orders' && ID.test(b)) return { name: 'order', id: b };
      if (a === 'invite' && TOKEN.test(b)) return { name: 'invite', token: b };
      if (a === 'settings' && b === 'notifications') return { name: 'notificationSettings' };
      if (a === 'oauth' && b === 'callback') return 'oauth';
      return null;
    }
    case 4:
      if (a === 'w' && WORKSPACE.test(b) && c === 'items' && ID.test(d)) {
        return { name: 'workspaceItem', workspaceId: b.toLowerCase(), itemId: d };
      }
      return null;
    default: return null;
  }
}

/** Relative in-app path for `?next=` — rejects absolute and protocol-relative URLs. */
export function safeNextPath(next: string | null | undefined): string | null {
  if (!next || next.length > 512) return null;
  if (!next.startsWith('/') || next.startsWith('//') || next.includes('\\')) return null;
  const parsed = parseLink(`https://${LINK_CONFIG.hosts[0]}${next}`);
  return parsed.kind === 'route' ? next : null;
}
```

### A.3 Parser unit tests

```ts
// src/links/__tests__/parseLink.test.ts (sketch) — it.each over accepted URLs (www, uppercase host,
// trailing slash, slug, both scheme forms, workspace, reset token) and rejected ones (foreign host, http,
// bad id, broken %-encoding, %00, short token, javascript:, oversized), oauth -> 'external',
// click-tracking unwrap keeps attribution, safeNextPath rejects absolute / '//' / '\\' / unknown paths.
```

### A.4 Pending-link store and dedupe

```ts
// src/links/pendingLinkStore.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import type { LinkSource, Route } from './routes';

const KEY = 'pendingLink.v1';
const TTL_MS = 60 * 60 * 1000; // 1 h: covers sign-up + email verification + app restart

export type PendingLink = { route: Route; source: LinkSource; savedAt: number; accountHint?: string };

let memory: PendingLink | null = null;

export const PendingLinkStore = {
  async save(link: Omit<PendingLink, 'savedAt'>, now = Date.now()): Promise<void> {
    memory = { ...link, savedAt: now };
    // Never persist one-time secrets; keep the route shape only.
    const persisted = sanitizeForStorage(memory);
    try { await AsyncStorage.setItem(KEY, JSON.stringify(persisted)); } catch { /* memory copy still works */ }
  },

  /** Returns the pending link once and clears it. Expired links are dropped. */
  async consume(now = Date.now()): Promise<PendingLink | null> {
    let link = memory;
    if (!link) {
      try { const raw = await AsyncStorage.getItem(KEY); link = raw ? (JSON.parse(raw) as PendingLink) : null; }
      catch { link = null; }
    }
    await PendingLinkStore.clear();
    if (!link || now - link.savedAt > TTL_MS) return null;
    return link;
  },

  async clear(): Promise<void> {
    memory = null;
    try { await AsyncStorage.removeItem(KEY); } catch { /* ignore */ }
  },
};

function sanitizeForStorage(link: PendingLink): PendingLink {
  if (link.route.name === 'resetPassword') {
    // Reset tokens stay in memory only; after a restart the user requests a new email.
    return { ...link, route: { name: 'home' } };
  }
  return link;
}

/** Drops the same URL delivered twice within a window (initial URL + first event, activity recreation). */
export function createDedupe(windowMs = 2000) {
  const seen = new Map<string, number>();
  return (url: string, now = Date.now()): boolean => {
    for (const [k, t] of seen) if (now - t > windowMs) seen.delete(k);
    if (seen.has(url)) return false;
    seen.set(url, now);
    return true;
  };
}
```

```ts
// pendingLinkStore.test.ts (sketch): consumes once; expires after TTL; reset tokens never persisted;
// createDedupe drops the same URL inside the window and accepts it again after.
```

### A.5 Link coordinator (gating, account awareness, analytics)

```ts
// src/links/linkCoordinator.ts
import { parseLink } from './parseLink';
import { PendingLinkStore, createDedupe } from './pendingLinkStore';
import { ROUTE_REQUIREMENTS, type LinkSource, type Route } from './routes';

export type AppGate = {
  isHydrated: () => boolean; needsOnboarding: () => boolean; isSignedIn: () => boolean;
  /** Resolves which signed-in account/portal owns a route; null = active one or public. */
  resolveOwner: (route: Route) => Promise<{ accountId: string; portalId?: string } | null>;
  activeOwner: () => { accountId: string; portalId?: string } | null;
};

export type LinkEffects = {
  navigate: (route: Route, opts: { source: LinkSource; coldStart: boolean }) => void;
  showNotFound: (path: string) => void; showOnboardingOrLogin: () => void;
  /** Delegates to react-native-multi-account: confirm, switch, then call back. */
  confirmSwitch: (owner: { accountId: string; portalId?: string }, then: () => void) => void;
  confirmSignOutFor: (route: Route) => void;
  track: (event: string, props: Record<string, unknown>) => void;
};

export function createLinkCoordinator(gate: AppGate, fx: LinkEffects) {
  const accept = createDedupe();
  const queue: Array<() => void> = [];

  async function handleUrl(url: string, source: LinkSource, coldStart: boolean): Promise<void> {
    if (!accept(url)) return;
    const parsed = parseLink(url);
    if (parsed.kind === 'external') return; // auth library owns it
    if (parsed.kind === 'unknown') {
      fx.track('link_unmatched', { source, path: parsed.path, cold: coldStart });
      fx.showNotFound(parsed.path);
      return;
    }
    fx.track('link_open', { source, route: parsed.route.name, cold: coldStart, ...parsed.attribution });
    await handleRoute(parsed.route, source, coldStart);
  }

  async function handleRoute(route: Route, source: LinkSource, coldStart: boolean): Promise<void> {
    if (!gate.isHydrated()) { queue.push(() => void handleRoute(route, source, coldStart)); return; }
    const req = ROUTE_REQUIREMENTS[route.name];

    if (req === 'loggedOut' && gate.isSignedIn()) return fx.confirmSignOutFor(route);
    if (gate.needsOnboarding() || (req === 'auth' && !gate.isSignedIn())) { // save, then gate
      await PendingLinkStore.save({ route, source });
      fx.track('link_gated', { route: route.name, gate: gate.needsOnboarding() ? 'onboarding' : 'auth' });
      fx.showOnboardingOrLogin();
      return;
    }
    if (req === 'auth') {
      const owner = await gate.resolveOwner(route);
      const active = gate.activeOwner();
      if (owner && (owner.accountId !== active?.accountId || owner.portalId !== active?.portalId)) {
        fx.confirmSwitch(owner, () => fx.navigate(route, { source, coldStart }));
        return;
      }
    }
    fx.navigate(route, { source, coldStart });
  }

  return {
    handleUrl,
    handleRoute,
    onHydrated: () => queue.splice(0).forEach((run) => run()), // auth/store hydration finished
    /** Call after login or onboarding completes. */
    async resumePending(): Promise<boolean> {
      const pending = await PendingLinkStore.consume();
      if (!pending) return false;
      fx.track('link_resumed', { route: pending.route.name, source: pending.source });
      await handleRoute(pending.route, pending.source, false);
      return true;
    },
    clearPending: () => PendingLinkStore.clear(), // call on sign-out
  };
}
```

### A.6 React Navigation 7 wiring (dynamic config)

```tsx
// src/navigation/linking.ts
import * as Linking from 'expo-linking';
import * as Notifications from 'expo-notifications';
import { getStateFromPath, type LinkingOptions } from '@react-navigation/native';
import { parseLink } from '../links/parseLink';
import type { RootParamList } from './types';

export function buildLinking(onUrl: (url: string, cold: boolean) => boolean): LinkingOptions<RootParamList> {
  return {
    prefixes: [Linking.createURL('/'), 'myapp://', 'https://example.com', 'https://www.example.com'],
    filter: (url) => parseLink(url).kind !== 'external',
    config: {
      screens: {
        Tabs: {
          screens: {
            // initialRouteName synthesizes the tab root beneath the deep-linked screen
            HomeTab: { initialRouteName: 'Home', screens: { Home: '', Product: 'p/:id' } },
            OrdersTab: { initialRouteName: 'Orders', screens: { Orders: 'orders', Order: 'orders/:id' } },
            SettingsTab: {
              initialRouteName: 'Settings',
              screens: { Settings: 'settings', NotificationSettings: 'settings/notifications' },
            },
          },
        },
        Invite: 'invite/:token',
        ResetPassword: 'reset-password',
        NotFound: '*',
      },
    },
    // Validate before building state: invalid ids land on NotFound instead of a crashing screen.
    getStateFromPath(path, options) {
      const parsed = parseLink(`https://example.com/${path.replace(/^\//, '')}`);
      if (parsed.kind === 'unknown') return getStateFromPath('not-found', options);
      if (parsed.kind === 'route' && parsed.route.name === 'product') {
        return getStateFromPath(`p/${parsed.route.id}`, options); // drop slug and tracking params
      }
      return getStateFromPath(path, options);
    },
    async getInitialURL() {
      const url = await Linking.getInitialURL();
      if (url) return onUrl(url, true) ? url : null; // coordinator may gate it (returns false)
      const response = await Notifications.getLastNotificationResponseAsync();
      const pushUrl = response?.notification.request.content.data?.url;
      return typeof pushUrl === 'string' && onUrl(pushUrl, true) ? pushUrl : null;
    },
    subscribe(listener) {
      const linkSub = Linking.addEventListener('url', ({ url }) => {
        if (onUrl(url, false)) listener(url);
      });
      const pushSub = Notifications.addNotificationResponseReceivedListener((response) => {
        const url = response.notification.request.content.data?.url;
        if (typeof url === 'string' && onUrl(url, false)) listener(url);
      });
      return () => { linkSub.remove(); pushSub.remove(); };
    },
  };
}
```

`onUrl` returns `true` when the link can be routed right now (public, or signed in as the right account) and `false` after it was saved as pending or handed to an account-switch confirmation; implement it with the coordinator's synchronous checks plus `PendingLinkStore.save`. After login, `resumePending()` navigates with `navigationRef.navigate(...)` once `navigationRef.isReady()`.

### A.7 Expo Router wiring

```tsx
// app/+native-intent.tsx — rewrites OS-delivered URLs before Expo Router matches them.
import { parseLink } from '../src/links/parseLink';
import { routeToHref } from '../src/links/routeToHref';

export function redirectSystemPath({ path, initial }: { path: string; initial: boolean }): string {
  try {
    const absolute = /^[a-z][a-z0-9+.-]*:/i.test(path) ? path : `https://example.com${path.startsWith('/') ? '' : '/'}${path}`;
    const parsed = parseLink(absolute);
    if (parsed.kind === 'external') return path; // let the auth session receive it
    if (parsed.kind === 'unknown') return '/not-found';
    return routeToHref(parsed.route); // canonical path, no tracking params
  } catch {
    return initial ? '/' : path; // never throw here
  }
}
```

```ts
// src/links/routeToHref.ts
import type { Route } from './routes';

export function routeToHref(route: Route): string {
  switch (route.name) {
    case 'home': return '/';
    case 'notificationSettings': return '/settings/notifications';
    case 'product': return `/p/${encodeURIComponent(route.id)}`;
    case 'order': return `/orders/${encodeURIComponent(route.id)}`;
    case 'invite': return `/invite/${encodeURIComponent(route.token)}`;
    case 'resetPassword': return `/reset-password?t=${encodeURIComponent(route.token)}`;
    case 'workspaceItem':
      return `/w/${encodeURIComponent(route.workspaceId)}/items/${encodeURIComponent(route.itemId)}`;
  }
}
```

```tsx
// Sketch, remaining Expo Router pieces:
// app/(app)/_layout.tsx — export unstable_settings = { initialRouteName: 'index' } (Home beneath deep-linked
//   screens); while loading return null (splash up); if signed out <Redirect href={`/login?next=${encoded path+query}`} />.
// app/login.tsx — after sign-in router.replace(safeNextPath(next) ?? '/') so Login never stays below the target.
// app/p/[id].tsx — params are untrusted: validate id with the parser's regex, else <Redirect href="/not-found" />.
```

Push taps in Expo Router: in the root layout, subscribe with `Notifications.addNotificationResponseReceivedListener` and read `useLastNotificationResponse()` for cold start; run `data.url` through `parseLink` and `router.push(routeToHref(route))` (dedupe with `createDedupe`).

### A.8 Native and app config

```ts
// app.config.ts (sketch): per APP_ENV pick domain + scheme ('myapp' / 'myapp-staging') + bundle id/package;
// ios.associatedDomains = applinks:<domain>, applinks:www.<domain>, applinks:click.<domain>;
// android.intentFilters = [{ action: 'VIEW', autoVerify: true, category: ['BROWSABLE','DEFAULT'],
//   data: [{ scheme: 'https', host: domain, pathPrefix: '/p/' }, ...one entry per claimed prefix/path] }];
// plugins: [['expo-router', { origin: `https://${domain}` }], 'expo-notifications'].
```

```swift
// ios/<App>/AppDelegate.swift (bare RN 0.77+) — add inside the AppDelegate class
import React

func application(_ app: UIApplication, open url: URL,
                 options: [UIApplication.OpenURLOptionsKey: Any] = [:]) -> Bool {
  return RCTLinkingManager.application(app, open: url, options: options)
}

func application(_ application: UIApplication, continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
  return RCTLinkingManager.application(application, continue: userActivity,
                                       restorationHandler: restorationHandler)
}
```

```xml
<!-- AndroidManifest.xml (bare, sketch): MainActivity exported, launchMode="singleTask"; one VIEW/BROWSABLE
     filter for <data android:scheme="myapp" />; one android:autoVerify="true" filter with https hosts
     example.com + www.example.com and only the claimed pathPrefix/path entries (/p/, /orders, /invite/, /w/, /reset-password). -->
```

Note: `<data>` elements inside one intent filter are merged (every scheme × host × path combination), so keep only hosts you verify in the same filter.

### A.9 apple-app-site-association

```json
{
  "applinks": { "details": [{
    "appIDs": ["TEAMID.com.example.app"],
    "components": [
      { "/": "/oauth/*", "exclude": true, "comment": "web-handled auth" },
      { "/": "/help/*", "exclude": true },
      { "/": "/p/*" }, { "/": "/orders", "comment": "list" }, { "/": "/orders/*" },
      { "/": "/invite/*" }, { "/": "/w/*/items/*" }, { "/": "/settings/notifications" },
      { "/": "/reset-password", "?": { "t": "?*" } }
    ]
  }] },
  "webcredentials": { "apps": ["TEAMID.com.example.app"] }
}
```

### A.10 assetlinks.json

```json
[{ "relation": ["delegate_permission/common.handle_all_urls"],
   "target": { "namespace": "android_app", "package_name": "com.example.app",
               "sha256_cert_fingerprints": ["AA:BB:...:PLAY_APP_SIGNING_KEY_SHA256", "CC:DD:...:UPLOAD_OR_EAS_KEY_SHA256"] } }]
```

### A.11 Hosting check (CI / synthetic monitor)

```ts
// scripts/check-link-files.ts (sketch): for each domain fetch /.well-known/apple-app-site-association and
// /.well-known/assetlinks.json with redirect: 'manual'; fail on non-200, non-application/json, non-JSON body,
// missing TEAMID.bundle / package / Play App Signing SHA-256; also fetch
// https://app-site-association.cdn-apple.com/a/v1/<domain> (what devices actually get); exit 1 on any error.
```

Notes for adapting:
- Keep `parseLink` as the single source of truth for paths; derive `routeToHref`, AASA components, and intent-filter prefixes from the same table where possible, and add a unit test that every `Route` round-trips (`parseLink(routeToHref(r))` equals `r`)
- For React Navigation v7 static config, move the `linking.path` onto each screen and keep `getStateFromPath`/`getInitialURL`/`subscribe` as above
- Multi-account apps: `resolveOwner` asks the server (or a local id → account index) which account/portal owns the target; the switch itself lives in **react-native-multi-account**
- Deferred links: call `coordinator.handleRoute(route, 'deferred', true)` with the route your install-referrer or server-match step returns, exactly once per install
