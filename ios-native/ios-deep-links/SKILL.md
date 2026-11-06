---
name: ios-deep-links
description: Use when implementing, reviewing, or debugging deep links on iOS/iPadOS in SwiftUI (onOpenURL, onContinueUserActivity, NavigationStack path) or UIKit (UIScene delegate, NSUserActivity, application(_:open:)) — universal links vs custom URL schemes, Associated Domains and apple-app-site-association (AASA) hosting/CDN caching/validation, URL design, URL → route parsing and back-stack synthesis, cold/warm launch handling, links that arrive before login or onboarding (pending link), account/portal-aware links, deferred deep links after install (vendor-neutral), push/widget/shortcut/Spotlight/QR/OAuth entry points, App Clips, fallbacks, security, analytics, and testing. Siblings: android-deep-links, flutter-deep-links, react-native-deep-links.
---

# iOS Deep Links Skill

Guide for building and auditing **deep linking** in consumer iOS apps: marketing links, shared content links, email links, invites, password-reset / magic-link sign-in, QR codes, push taps, widgets, and Home Screen quick actions. Covers **SwiftUI App lifecycle** (`.onOpenURL`, `.onContinueUserActivity`, `NavigationStack(path:)`) and **UIKit** (`UIWindowSceneDelegate`, `NSUserActivity`, legacy `UIApplicationDelegate` callbacks). Siblings: **android-deep-links**, **flutter-deep-links**, **react-native-deep-links** — keep the URL contract (paths, params, fallbacks) identical across platforms so one link works everywhere. Related iOS skills: **ios-multi-account** (link targets another account/portal), **ios-push-notifications** (notification tap payloads), **ios-in-app-purchase** / **ios-revenuecat-iap** (paywall and offer links), **ios-list-ux** (landing on a list item).

Associated Domains behaviour, AASA syntax (`components`, substitution variables), Apple CDN caching, `ASWebAuthenticationSession` callbacks, App Clip size limits and Developer-settings diagnostics change between iOS releases. **Verify fast-moving APIs and limits against current Apple docs before shipping**, and gate with `if #available` when the deployment target is lower.

Key OS gates (verify): universal links iOS 9; `UIScene` lifecycle iOS 13 (app-delegate URL callbacks are **not called** when scenes are adopted); SwiftUI `App`, `.onOpenURL`, `.onContinueUserActivity`, App Clips, AASA `components` syntax iOS 14; Associated Domains served via Apple's CDN iOS 14+; `NavigationStack(path:)` / `NavigationPath` iOS 16; `@Observable` iOS 17; `ASWebAuthenticationSession.Callback.https` iOS 17.4.

## 0. Mental model

```
  Entry points                       Normalise                 Decide                     Apply
 ─────────────────────────────    ───────────────────    ────────────────────────    ──────────────────────
  Universal link (https)  ─┐
  Custom scheme (myapp://) ─┤
  Push tap (userInfo url) ─┤                              ┌─ gate: onboarding? ─┐
  Widget / Live Activity  ─┼─▶  URL ─▶ DeepLinkParser ─▶  │  gate: signed in?   │─▶ ready ─▶ AppRouter
  Quick action / Spotlight─┤        (pure, tested)        │  gate: right acct/  │           selects tab,
  QR / NFC / App Clip     ─┤         returns DeepLink     │        portal?      │           dismisses modals,
  Deferred (post-install) ─┘         or .unsupported      └─────────┬───────────┘           sets path
                                                                    │ not ready
                                                                    ▼
                                                         PendingLinkStore (one slot,
                                                         TTL) → replay when gate clears
```

Rules that follow from the diagram:

1. **Every entry point funnels into one function**: `DeepLinkCoordinator.handle(url:source:)`. No screen parses URLs itself.
2. **Parsing is pure** (URL in, enum out, no side effects) so it can be unit tested exhaustively.
3. **Gating is separate from routing**: the router never knows about login; the coordinator decides whether to route now or park the link.
4. **Routing is declarative**: a `DeepLink` maps to a full navigation state (tab + stack + optional sheet), not to a sequence of `push` calls.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Link types in use**: universal links (`https://example.com/...`), custom scheme (`myapp://`), both? Which are already printed on QR codes, in emails, in old app versions' share sheets (these can never be retired)?
- **Domains**: primary web domain, marketing/short-link domain, email click-tracking domain, regional or staging domains (`staging.example.com`). Who controls hosting and the CDN for each?
- **Web parity**: does every app URL also render a useful web page? (It must — it is the fallback when the app is not installed or the link is opened in Safari.)
- **Destinations**: list every route a link can open: content detail, profile, invite, checkout/paywall, settings, magic-link sign-in, password reset, email verification.
- **Logged-in vs logged-out destinations**: which routes are public (product page), which require auth (order detail), which *are* auth (magic link, reset)? What happens if onboarding has not been completed?
- **Accounts/portals**: does the app support multiple accounts or workspaces? Do URLs carry an account/portal identifier? (See section 8 and **ios-multi-account**.)
- **Router**: SwiftUI `NavigationStack(path:)`, `TabView(selection:)`, a coordinator over `UINavigationController`, a third-party router, or ad-hoc `NavigationLink(isActive:)`? Is navigation state centralised?
- **Lifecycle**: SwiftUI `App`, UIKit with `UIScene` (`UIApplicationSceneManifest` in Info.plist), or legacy app-delegate-only? Multiple windows on iPad?
- **Install attribution**: do campaigns need **deferred** deep links (open the right screen on first launch after install)? What does the privacy policy / App Privacy label allow?
- **App Clip**: is there one, or planned? (Section 4.6.)
- **Min OS** and whether Mac Catalyst / visionOS ship the same target.

Write a **link spec** before coding:

```
| Pattern                         | DeepLink               | Auth     | Account-scoped | Stack (tab ▸ path)           | Web fallback          |
|---------------------------------|------------------------|----------|----------------|------------------------------|-----------------------|
| /p/{productId}                  | .product(id)           | public   | no             | shop ▸ [product]             | product web page      |
| /orders/{orderId}               | .order(id)             | required | yes (account)  | account ▸ [orders, order]    | web order page        |
| /w/{portalId}/projects/{id}     | .project(portal, id)   | required | yes (portal)   | projects ▸ [project]         | web app               |
| /invite/{token}                 | .invite(token)         | either   | resolves       | modal sheet                  | invite landing        |
| /auth/magic?token=…             | .magicLink(token)      | is auth  | n/a            | handled by AuthFlow          | web sign-in           |
| /settings/notifications         | .notificationSettings  | required | yes            | profile ▸ [settings, notif]  | web settings          |
```

## 2. Link types: universal links vs custom schemes

| | Universal link (`https://`) | Custom scheme (`myapp://`) |
|---|---|---|
| Ownership | Proven via AASA on a domain you control | **Anyone** can register the same scheme; iOS picks one unpredictably |
| Not installed | Opens the web page (graceful) | Error / nothing happens |
| Works from | Messages, Mail, Notes, Safari *cross-domain* taps, other apps via `open(_:)`, QR scanner | Other apps, Safari (with confirmation prompt), your own code |
| Doesn't open app | Typed/pasted into Safari address bar; same-domain navigation in Safari; user chose "Open in Safari"; some in-app browsers; JS redirects without user gesture | — |
| Use for | **All public/shared/marketing/email links** | Internal plumbing: widget URLs, push payload routes, callbacks between your own apps, legacy compatibility |

- [ ] **Default to universal links** for anything a human shares or a campaign sends. Never print a custom scheme in email, SMS, or a QR code.
- [ ] Keep a custom scheme for **internal** sources (widgets, notification payloads, `UIApplicationShortcutItem`) where you control both ends — but parse it with the **same parser** by mapping `myapp://p/123` and `https://example.com/p/123` to the same `DeepLink`.
- [ ] Do not use a custom scheme as an OAuth redirect if you can use `ASWebAuthenticationSession` with an `https` callback (iOS 17.4+) or at least a scheme unique to the app (reverse-DNS, e.g. `com.example.app`). See section 12.
- [ ] Remember the user can **disable** universal links per domain (long-press → "Open in Safari"; re-enable via the Smart App Banner or long-press → "Open in <App>"). Your web page must stay useful and ideally show a Smart App Banner (`<meta name="apple-itunes-app" content="app-id=…, app-argument=…">`).

## 3. Domain verification: apple-app-site-association

### 3.1 Hosting rules

- [ ] Serve at `https://<domain>/.well-known/apple-app-site-association` (root `/apple-app-site-association` is legacy fallback). **No `.json` extension.**
- [ ] **HTTPS with a valid certificate** chain; no self-signed certs. TLS failure = silent non-association.
- [ ] **No redirects** of any kind (not `http→https`, not `apex→www`, not to a CDN bucket). Each host is verified separately: `example.com` and `www.example.com` both need the file and both need an entitlement entry (or use `applinks:*.example.com` wildcard, which does **not** cover the apex).
- [ ] `Content-Type: application/json` (recommended; verify). Not signed (signing was required only before iOS 9).
- [ ] Uncompressed size within Apple's limit (historically 128 KB — verify). Keep it small: use wildcard components, not one entry per URL.
- [ ] Reachable **without auth, geo-blocking, bot protection, or WAF challenge**. Apple's CDN fetches from its own IP ranges — allowlist them or exempt the path. A bot-protection JS challenge on `/.well-known/` breaks every install.
- [ ] Return **200**, not 404/403, for each associated host, including staging hosts used by internal builds.

### 3.2 Apple CDN caching (iOS 14+)

- Devices do **not** fetch your file directly; they ask Apple's CDN, which fetches and caches it (`https://app-site-association.cdn-apple.com/a/v1/<domain>` — inspect this to see what devices actually get).
- The device refreshes at **install and app update**, and periodically afterwards (not on every launch). A changed AASA may take **hours to days** to reach installed devices.
- [ ] Treat AASA changes like a release: add new paths **before** shipping app code that relies on them; remove paths **after** the app versions that need them have aged out.
- [ ] For development, use the **alternate mode**: entitlement `applinks:staging.example.com?mode=developer` makes the device fetch directly from your server (bypasses CDN), only on devices with **Associated Domains Development** enabled in Settings ▸ Developer. Use `?mode=managed` only for MDM-managed enterprise devices. Never ship `mode=developer` entries expecting them to work in production builds.
- [ ] Also set `Cache-Control` sensibly on your origin (e.g. a few hours) so your own CDN doesn't serve a stale file to Apple for weeks.

### 3.3 File contents

- [ ] `appIDs` use `<TeamID>.<bundleID>` (Team ID from the developer account — **not** the App ID prefix of an old app, verify for apps transferred between teams).
- [ ] Use `components` (iOS 13+/14+) instead of legacy `paths`. Order matters: **first match wins**, so put `exclude: true` rules before broad wildcards.
- [ ] Explicitly **exclude** paths that must stay on the web: `/account/delete`, `/legal/*`, `/blog/*`, `/help/*`, OAuth `/oauth/authorize`, web-only checkout, unsubscribe pages (users tapping "unsubscribe" in an email expect a web page, not your app).
- [ ] Match query items with the `"?"` key when routing depends on them (e.g. exclude `?web=1`). Fragments (`"#"`) are matched too but are rarely a good idea.
- [ ] Use `"caseSensitive": false` and `"percentEncoded"` deliberately (verify defaults for your min OS).
- [ ] Add `webcredentials` (Password AutoFill / passkeys) and `appclips` sections to the **same file** if needed; one file per host.
- [ ] Multiple apps on one domain (main app + another brand): separate `details` entries with disjoint components.
- See **Appendix A.8** for a full example, plus `assetlinks.json` for the Android sibling sharing the same domain.

### 3.4 Validation

- [ ] `curl -sI https://example.com/.well-known/apple-app-site-association` → `200`, no `Location`, correct `content-type`.
- [ ] `curl -s https://app-site-association.cdn-apple.com/a/v1/example.com` → shows what Apple cached (may lag).
- [ ] JSON lint in CI; a CI check that fetches production AASA after every web deploy and diffs it against the version in the repo (web deploys silently deleting `/.well-known/` is a classic outage).
- [ ] On device: Settings ▸ Developer ▸ Universal Links ▸ Diagnostics (iOS 16+, verify) — enter a URL and see whether it matches. On macOS, `swcutil` can inspect association state (verify flags for your macOS version).
- [ ] Monitoring: alert on non-200 for the AASA URL from an external uptime checker.

## 4. App setup (iOS)

### 4.1 Entitlement and Info.plist

- [ ] Signing & Capabilities ▸ **Associated Domains**: `applinks:example.com`, `applinks:www.example.com`, `applinks:go.example.com` (short links), `applinks:click.example.com` (email tracking, section 10), `webcredentials:example.com` if using AutoFill.
- [ ] The App ID on the developer portal must have Associated Domains enabled; the provisioning profile must be regenerated (automatic signing does this).
- [ ] Separate entitlements per configuration (`Debug` → staging hosts with `?mode=developer`; `Release` → production) via per-config `.entitlements` files.
- [ ] Custom scheme: `CFBundleURLTypes` → `CFBundleURLSchemes` = `["myapp"]` with a unique `CFBundleURLName` (reverse-DNS).
- [ ] To open other apps' schemes with `canOpenURL`, list them in `LSApplicationQueriesSchemes` (not needed for `open`).

### 4.2 SwiftUI App lifecycle

- `.onOpenURL { url in }` receives **custom scheme URLs and universal links** (and widget `widgetURL`/`Link` URLs) for the scene — both on cold launch and while running. Attach it **once**, at the root view of the `WindowGroup`.
- `.onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in activity.webpageURL }` is the explicit universal-link path; also used for Handoff, Spotlight (`CSSearchableItemActionType`), and `NSUserActivity`-based Siri/App Intents. Universal links normally arrive via `onOpenURL` in SwiftUI; handle both and **dedupe** (verify on your min OS).
- [ ] Do not attach `.onOpenURL` on child views that may not exist yet on cold launch — the event is delivered to views in the hierarchy at that moment.
- [ ] Forward to the coordinator immediately; don't do async work in the closure before recording the link (the coordinator decides if it is ready).
- [ ] App delegate (`@UIApplicationDelegateAdaptor`) is still needed for push registration and `UNUserNotificationCenterDelegate` (section 10).

### 4.3 UIKit with scenes (`UIWindowSceneDelegate`)

| State | Universal link | Custom scheme | Quick action |
|---|---|---|---|
| Cold launch | `scene(_:willConnectTo:options:)` → `connectionOptions.userActivities` | `connectionOptions.urlContexts` | `connectionOptions.shortcutItem` |
| Running / background | `scene(_:continue:)` | `scene(_:openURLContexts:)` | `windowScene(_:performActionFor:completionHandler:)` |

- [ ] **Handle cold launch in `willConnectTo`.** The most common UIKit bug: only `scene(_:continue:)` is implemented, so links work when the app is running but are dropped on cold start.
- [ ] Root UI must exist before routing: build the window/root coordinator first, then pass the URL to the coordinator (which may park it).
- [ ] When scenes are adopted, `application(_:open:options:)` and `application(_:continue:restorationHandler:)` are **not called** — remove dead code or keep it only for a non-scene fallback.
- [ ] iPad multi-window: decide per link whether to route in the **active** scene or open a new one (`UIApplication.shared.requestSceneSessionActivation`). Most consumer apps: route in the foreground-active scene.
- [ ] `UIOpenURLContext.options.sourceApplication` is only populated for apps from the same team — don't rely on it for security.

### 4.4 Legacy app-delegate-only apps

- `application(_:open:options:)` for schemes; `application(_:continue:restorationHandler:)` for universal links; `launchOptions[.url]` / `.userActivityDictionary` on cold launch. Return `true` only when handled.

### 4.5 Opening links from inside the app

- [ ] Links to your own domain inside the app (tapped in a `Text` with `AttributedString` links, chat messages, `WKWebView`): intercept with `OpenURLAction` (`.environment(\.openURL, OpenURLAction { url in … .handled })`) or `WKNavigationDelegate`, and route **internally** — iOS will not open your own universal link into your own app via Safari.
- [ ] `UIApplication.shared.open(url, options: [.universalLinksOnly: true])` opens another app's universal link only if installed, else returns `false` (then show `SFSafariViewController`).

### 4.6 App Clips (note)

- App Clip invocations (App Clip Codes, NFC, QR, Safari Smart App Banner, Messages, Maps) deliver the invocation URL as an `NSUserActivity` of type `NSUserActivityTypeBrowsingWeb` — handle with `.onContinueUserActivity` / `scene(_:willConnectTo:)` in the Clip target.
- [ ] AASA needs an `appclips` section with the Clip's `<TeamID>.<bundleID>.Clip`; Clip target gets `appclips:example.com` entitlement; App Clip experiences are configured in App Store Connect (default + advanced experiences by URL prefix).
- [ ] **The full app must handle every URL the Clip handles** — once the full app is installed, the same invocation opens the full app instead of the Clip.
- [ ] Share the parser/`DeepLink` enum between app and Clip (Swift package or shared target); Clip routes only a subset and offers "Get the full app" (`SKOverlay.AppClipConfiguration`) for the rest.
- [ ] Pass Clip state to the full app via a shared App Group container (`group.com.example`) — this is also the most reliable "deferred link" on iOS when the user upgrades from Clip to app.
- [ ] Clip size limits and invocation-type rules change per iOS version — verify current limits. Test with Local Experiences (Settings ▸ Developer ▸ App Clips Testing) and `_XCAppClipURL` environment variable in the Clip scheme.

## 5. URL design

- [ ] **Human-readable, stable, resource-oriented**: `/p/{id}`, `/u/{handle}`, `/orders/{id}`. Path = *what*, query = *how* (`?tab=reviews`, `utm_*`).
- [ ] **Same URL as the web page.** One canonical URL per resource; the app and website both understand it. No `/app/` or `/deeplink/` prefixes that only the app understands.
- [ ] Use **opaque, stable IDs** (not database row numbers that differ per region, not names that can change). Slugs are fine as decoration: `/p/123-red-shoes` → parse the leading ID, ignore the slug.
- [ ] **Versioning**: don't put `v1` in public URLs. Evolve by adding routes; keep old routes parsing forever (map old → new in the parser). If you must change semantics, use a new path, not a new meaning for an old one.
- [ ] **Account/portal scoping in the path** when content lives in a workspace: `/w/{portalId}/projects/{id}`. Account is usually *not* in the URL (it's whoever is signed in), but invites and shared links may carry an account hint (`?acct=` hash or email hint) — never an access token.
- [ ] **Encoding**: build URLs with `URLComponents` + `URLQueryItem`, never string concatenation. Percent-encode path segments (`addingPercentEncoding(withAllowedCharacters: .urlPathAllowed)` for segments containing `/` is *wrong* — encode `/` too). `+` in queries is **not** a space for `URLComponents`; agree with the backend.
- [ ] Parse with `URLComponents(url:resolvingAgainstBaseURL:)`; compare host case-insensitively; tolerate trailing slashes, `www.`, and unknown query items (marketing tools append them).
- [ ] **Stable marketing links**: short links (`go.example.com/spring`) resolved server-side to canonical URLs, so campaigns can be retargeted without reprinting. If the short domain is associated, the app receives the short URL and must resolve it (HEAD/GET to your resolver API) before parsing — design for that (Appendix A.7).
- [ ] Never put secrets in URLs that persist (tokens in links are logged by proxies, analytics, and email scanners). One-time, short-TTL, single-use tokens only (magic link, reset), redeemed server-side.
- [ ] Email security scanners **pre-fetch** links: GET on a magic-link URL must not consume the token on the web side; require a POST/user action on the web page, while the app redeems via API.

## 6. Routing

### 6.1 URL → DeepLink parsing

- [ ] A single `DeepLinkParser.parse(_ url: URL) -> DeepLink` (Appendix A.2), accepting both `https://<associated hosts>` and `myapp://` forms. Return `.unsupported(url)` rather than `nil` so fallbacks can act on it.
- [ ] Strict validation per param: IDs match a pattern (`^[A-Za-z0-9_-]{1,64}$`), enums from a closed set, numbers in range. Unknown host → `.unsupported`.
- [ ] Parser is `nonisolated`, synchronous, no network, no global state → **table-driven unit tests** (Appendix A.3).

### 6.2 Stack rules

- [ ] **Synthesize the back stack**: a link to `/orders/42` lands on `Account ▸ Orders ▸ Order 42`, so Back goes somewhere sensible instead of exiting to a blank root. Define this per route in the link spec.
- [ ] **Replace vs push**: an external link (from another app, push, widget) **replaces** the target tab's stack with the synthesized stack. An *internal* link (tapped in your own chat/feed) **pushes** onto the current stack.
- [ ] **Select the tab** first (`TabView(selection:)`), then set that tab's `NavigationPath`/`[Route]` in one assignment. Setting path element by element animates multiple pushes and can drop some.
- [ ] **Dismiss modals** (sheets, full-screen covers, alerts, `UIViewController.presentedViewController` chain) before routing — or present the target *over* them only when the route is itself a modal (invite sheet). Never leave a half-filled form silently underneath: if a modal has unsaved changes (checkout, compose), ask "Discard draft?" or park the link until it closes (section 7.3).
- [ ] **Same destination already showing** (link to product 123 while on product 123): refresh in place, don't push a duplicate.
- [ ] Use typed routes (`enum Route: Hashable`) with `.navigationDestination(for: Route.self)` registered **once per stack** at the root of the stack, not inside lazy containers (destinations inside `List`/`LazyVStack` are ignored).
- [ ] Keep route payloads small (IDs, not models); the destination screen loads its own data and shows loading/error/not-found states (section 11).
- [ ] UIKit: a `Coordinator` per tab with `setViewControllers(_:animated:)` for replacement; `tabBarController.selectedIndex`; `dismiss(animated:)` on the presented chain with completion before navigating.
- [ ] Animate only the final transition (or none on cold launch — users should see the destination, not a cascade of pushes).

## 7. App states and gating

### 7.1 Cold, warm, foreground

| State | Delivery | Watch out for |
|---|---|---|
| Cold launch | SwiftUI `onOpenURL` after first scene render; UIKit `willConnectTo` options | Root UI/DI not ready; splash/launch animation; remote config not loaded; state restoration fighting the link |
| Warm (backgrounded) | `onOpenURL` / `scene(_:continue:)` / `openURLContexts` | User was mid-flow (modal, form, video call); stale data on screen |
| Foreground (in-app tap, push tap) | Same, or internal `OpenURLAction` | Don't animate tab switch + pushes + sheet at once |

- [ ] **State restoration vs link**: on cold launch with a link, the link wins; skip restoring the previous navigation state (`SceneStorage`/`stateRestorationActivity`).
- [ ] Splash/launch screen: queue the link until the root is mounted (coordinator `isReady` flag), never route into a view hierarchy that is about to be replaced.

### 7.2 Link before login or onboarding (pending link)

- [ ] Coordinator evaluates **gates** in order: app-update-required → onboarding complete → authenticated → correct account/portal → feature flag/entitlement. First failing gate **parks** the link in `PendingLinkStore` (Appendix A.4) and shows the gate's UI.
- [ ] **Public routes skip the auth gate** (product pages, articles). Don't force sign-in to view something the website shows logged-out.
- [ ] **Auth routes are handled by the auth flow itself** (magic link, email verification, password reset) — they are not parked behind login; they *are* the login.
- [ ] When the gate clears (login success, onboarding done), the coordinator **replays** the pending link exactly once, then clears it.
- [ ] One slot, last link wins. TTL (e.g. 10–30 min) so a link tapped yesterday doesn't hijack today's launch. Store in memory + `UserDefaults` only if it must survive the app being killed during sign-up (common for email-verification and invite flows); **never persist links containing tokens** beyond the TTL; prefer Keychain if they do.
- [ ] Show context during the gate: "Sign in to view your order" / "Sign up to join Alex's group" (invite landing), not a generic login screen.
- [ ] If the user **cancels** login or signs in to an account that can't access the target, drop the pending link and show a not-available state — don't loop. Sign-out clears the pending link.

### 7.3 Link during a modal or critical flow

- [ ] Define "uninterruptible" states: active purchase sheet (StoreKit), payment, call, camera capture, unsaved compose. Park the link and replay when the flow ends, or show a non-blocking banner "Open link?".
- [ ] Never route while a system sheet (StoreKit, `ASWebAuthenticationSession`, share sheet) is up — you cannot dismiss those reliably.

## 8. Account and portal awareness

- [ ] For multi-account / multi-portal apps, the coordinator resolves **which (account, portal)** owns the target: from the URL (`/w/{portalId}/…`), from the server (resolve endpoint returns owner), or from push payload (`accountId`, `portalId`).
- [ ] Target belongs to the **active** account/portal → route.
- [ ] Target belongs to **another signed-in account or another portal** → confirm ("Switch to Work account to open this?"), switch, then route. Account switch tears down the session graph, so the link must be parked across the switch and replayed after the new session is ready.
- [ ] Target belongs to **no signed-in account** → "Sign in to an account with access" (add-account flow) with the link parked.
- [ ] User lacks access in that portal → not-available state; never silently open the wrong portal's copy of a resource with the same ID.
- [ ] Hand off the switching mechanics (registry, teardown, rebuild, restore) to **ios-multi-account**; this skill only defines the resolution and confirmation contract.

## 9. Deferred deep links (vendor-neutral)

Goal: user taps a link, does **not** have the app, installs from the App Store, opens the app → lands on the linked content. iOS **passes nothing** through the App Store install, so every approach is a trade-off.

| Mechanism | How it works | Accuracy | Privacy / review risk |
|---|---|---|---|
| **Explicit user identity** | Link carries an invite/claim token tied to an email or phone; after sign-up with that identity the server returns the pending destination | Exact | Low — best option for invites, referrals, shared content to known people |
| **Server-side match ("fingerprinting")** | Web landing page records IP, user agent, screen, time; first app launch sends the same signals; server matches within a short window | Probabilistic; breaks with iCloud Private Relay, carrier NAT, VPNs, shared Wi-Fi | **High**: Apple's guidelines and ATT treat device fingerprinting for tracking as prohibited; must be disclosed in App Privacy; only defensible for first-party, non-tracking, short-window use — get legal review |
| **Clipboard handoff** | Landing page copies a token; app reads `UIPasteboard` on first launch | Exact if it works | iOS shows a paste banner/permission prompt (iOS 14/16+) — looks shady to users; avoid, or use `UIPasteControl` / `detectPatterns` with explicit user action ("Continue where you left off") |
| **App Clip → full app** | Clip writes state to shared App Group; full app reads it on first launch | Exact | Low; only when an App Clip exists |
| **Web sign-in first** | Landing page lets user sign in/up on the web; app picks up server-side "continue" state after login | Exact after login | Low |
| **Re-tap** | First-launch screen says "Tap the link again"; universal link now opens the app | Exact, needs user | None |

- [ ] Prefer **deterministic, identity-based** matching. Use probabilistic matching only for non-sensitive destinations (open a product page), never for anything that grants access or reveals personal data (invites to private groups, order details).
- [ ] Deferred payload is **a destination suggestion**, validated by the same parser and gates; it can expire; the server should only return it once.
- [ ] Ask for the deferred link **once**, on first launch, with a short timeout (≤1–2 s) so it never blocks onboarding; if it arrives late, offer it as a banner rather than yanking the user.
- [ ] **Firebase Dynamic Links shut down on August 25, 2025.** Migration checklist: inventory every `*.page.link` / custom FDL domain in emails, QR codes, printed material and old app versions; move the domain (if custom) to your own hosting with an AASA and a server-side redirect/landing page; host your own short-link resolver; replace FDL SDK calls in the app with your own parser + resolver; for `page.link` subdomains you don't own, links are dead — reissue them. Evaluate any replacement service on: custom-domain support, AASA/assetlinks hosting control, data retention and privacy terms, deferred-matching method (identity vs fingerprint), export of link data, and exit plan.
- [ ] Measure: fraction of installs from links that land on the intended screen; never assume.

## 10. Other entry points

- [ ] **Push notification taps** (`UNUserNotificationCenterDelegate.userNotificationCenter(_:didReceive:)`): payload carries a `url` (or `deeplink`) field using the same URL contract plus `accountId`/`portalId`; call the coordinator with `source: .push`. Set the delegate in `application(_:didFinishLaunchingWithOptions:)` so cold-launch taps aren't lost. See **ios-push-notifications**.
- [ ] **Widgets / Live Activities**: `widgetURL(_:)` (whole small widget) and `Link(destination:)` (medium/large) → delivered via `onOpenURL`. Use custom scheme or https; include the widget family in the source for analytics. Widgets for a non-active account must carry `accountId`.
- [ ] **Home Screen quick actions** (`UIApplicationShortcutItem`): map `type`/`userInfo` to a `DeepLink` (not a URL) or embed a URL in `userInfo`. Cold launch: `connectionOptions.shortcutItem`; running: `windowScene(_:performActionFor:)`. SwiftUI: needs a scene delegate via `UIApplicationDelegateAdaptor` → `application(_:configurationForConnecting:options:)`.
- [ ] **Spotlight / Handoff / Siri**: `NSUserActivity` with `webpageURL` set to the canonical URL (Handoff to/from Safari works for free) or `CSSearchableItemActionType` with `CSSearchableItemActivityIdentifier` → map to `DeepLink`. App Intents (`OpenIntent`, `AppEntity`) → call the same coordinator.
- [ ] **Email click tracking breaks universal links**: ESPs rewrite links to `https://click.esp-domain.com/…` which redirects to your URL. iOS evaluates the *first* URL (tracker's domain, not associated) → opens Safari → redirect to your site → stays in Safari. Fixes: (a) a **custom click-tracking domain you control** (`click.example.com`) with its own AASA and entitlement, then the app receives the tracking URL and resolves it (request it, follow the redirect, record the click, parse the final URL — Appendix A.7); or (b) disable click tracking for deep-linkable links. Test with the real ESP, real Mail app and Gmail app.
- [ ] **In-app browsers** (social apps' webviews) often don't trigger universal links. Landing pages should show an explicit "Open in app" button (user gesture navigation to the universal link or to the App Store) as the fallback.
- [ ] **QR codes**: encode universal links (camera app opens universal links directly); keep them short (lower QR density) via your short domain; never encode custom schemes. Printed QR codes live for years → route must never be removed (section 11).
- [ ] **OAuth / SSO redirects**: use `ASWebAuthenticationSession` and handle the callback in its completion handler; **do not** route OAuth callbacks through the general deep-link router (`/oauth/callback` must be excluded from AASA if you use a web callback page, or matched only by the auth session via `.https(host:path:)` on iOS 17.4+). Validate `state`, use PKCE.
- [ ] **Payment/3-D Secure return URLs** from payment SDKs: these are callbacks to the SDK, not navigation; forward them to the SDK handler before the router.
- [ ] **Paywall / offer links** (`/offer/{code}`, `/upgrade`): route to the paywall; redemption must go through StoreKit (`presentOfferCodeRedeemSheet`) — see **ios-in-app-purchase** / **ios-revenuecat-iap**.

## 11. Fallbacks

- [ ] **App not installed**: universal link opens the web page → the page renders the content (or a landing page) with App Store button / Smart App Banner. Every associated path must have a web response that is not a 404.
- [ ] **Unknown route** (`.unsupported`): open the URL in `SFSafariViewController` (if it's on your domain and web can handle it) or land on home with a toast "This link isn't supported in this version — update the app" when the server says it's a newer route. Never crash, never show a blank screen.
- [ ] **Newer link, older app**: include a minimum-version hint server-side (resolve endpoint returns `minAppVersion`) → prompt to update.
- [ ] **Expired/used link** (invite expired, reset token used): destination screen shows a specific state with a next action ("Ask Alex for a new invite", "Send a new reset link").
- [ ] **Content not found / no access**: dedicated not-found state inside the synthesized stack, with Back working; distinguish 404 vs 403 vs offline.
- [ ] **Removed routes**: keep them in the parser mapped to the closest equivalent (`/collections/{id}` → search results) forever; remove from AASA only if the web handles them better.
- [ ] **Offline on cold launch**: route immediately to the destination screen, which shows its own offline/cached state; don't block in the coordinator waiting for network (except for resolve-required short links, with timeout → web fallback).

## 12. Security

- [ ] **Treat every link as untrusted input** from an attacker — anyone can send your user `myapp://` or `https://example.com/…` with crafted params.
- [ ] **No destructive or state-changing action directly from a link**: no "delete", "transfer", "follow", "subscribe", "accept invite", "change email" without an in-app confirmation screen with the user's explicit tap. Links *navigate*; users *act*.
- [ ] Validate every param (pattern, length, enum); reject rather than sanitize. Don't pass raw link params into SQL/predicates, file paths, `WKWebView.loadHTMLString`, or JavaScript.
- [ ] **Open redirect**: never honour `?redirect=`/`?next=`/`?url=` params that load arbitrary URLs in your in-app webview or `open` them. Allowlist hosts; prefer route names over URLs as params.
- [ ] **WebView with privileges** (JS bridge, auth cookies): never load a link-supplied URL in it unless host is allowlisted.
- [ ] **Custom scheme hijacking**: another app can register `myapp://` and receive links meant for you (including OAuth codes and magic-link tokens). Never deliver tokens via custom schemes; use universal links or `ASWebAuthenticationSession`.
- [ ] Magic-link / reset tokens: single-use, short TTL, bound to the device or session that requested them where feasible ("Open on the same device you requested from" with a fallback code), redeemed via POST, not logged.
- [ ] Don't log full URLs with tokens in analytics/crash reports — redact query params by allowlist (Appendix A.2 `redactedForLogging`).
- [ ] Account confusion: links must never perform actions in whichever account happens to be active without checking ownership (section 8).
- [ ] Rate-limit/abuse: a link that triggers a network call on open (resolve endpoint) must not be usable to spam the backend from many devices — standard server rate limits.

## 13. Analytics

- [ ] Emit `deep_link_opened` with: `source` (universal_link, custom_scheme, push, widget, quick_action, spotlight, qr, deferred, internal), `route` (route name, not raw URL), `launch_type` (cold/warm), `result` (routed, parked_for_auth, parked_for_onboarding, account_switch, unsupported, not_found, error), `utm_source/medium/campaign/content/term`, and a link/campaign ID if present.
- [ ] Emit `deep_link_completed` when the destination actually rendered content (or failed), to measure drop-off between open and land, and `pending_link_replayed` after gates.
- [ ] Attribute sign-ups/purchases to the link that started the session (store `firstTouch` / `lastTouch` campaign params with a sensible window).
- [ ] UTM params are for analytics only — strip them before routing; never route on them.
- [ ] Install attribution for paid campaigns on iOS goes through Apple's privacy-preserving frameworks (SKAdNetwork / AdAttributionKit) and ATT consent rules; deep-link analytics must not become cross-app tracking without consent.
- [ ] No PII in event params (no emails, tokens, full URLs with IDs of private content if your policy forbids it).

## 14. Testing

### 14.1 Tools

```bash
# Simulator: custom scheme and universal links (associated domains work on simulator once AASA is fetched; use ?mode=developer for staging)
xcrun simctl openurl booted "myapp://p/123"
xcrun simctl openurl booted "https://example.com/p/123?utm_source=test"

# Push with a URL payload (drag an .apns file onto the simulator, or):
xcrun simctl push booted com.example.app push.apns

# Verify hosting
curl -sI https://example.com/.well-known/apple-app-site-association
curl -s  https://app-site-association.cdn-apple.com/a/v1/example.com | python3 -m json.tool
```

- On device: paste links into **Notes** or **Messages** and tap them (typing in Safari's address bar does **not** trigger universal links). Settings ▸ Developer ▸ Universal Links ▸ Diagnostics to check matching.
- Xcode scheme ▸ Run ▸ Options ▸ "Wait for the executable to be launched", then tap a link → debug the **cold launch** path.
- Console.app, filter `swcd` (shared web credentials daemon) on device to see AASA fetch errors.
- App Clips: `_XCAppClipURL` env var; Settings ▸ Developer ▸ Local Experiences.

### 14.2 Test plan

1. **Parser unit tests**: every link-spec row, every legacy route, malformed IDs, unknown hosts, case/trailing slash/`www`, percent-encoding, extra query items, both scheme and https forms (Appendix A.3).
2. **Coordinator unit tests** with fake gates: parked when logged out → replayed after login once; TTL expiry; sign-out clears; account mismatch → confirm; modal-in-progress → parked.
3. **Router tests**: `DeepLink` → expected `(tab, path, sheet)` state; same-destination dedupe.
4. **UI tests** (XCUITest): `XCUIApplication().open(url)` (iOS 16.4+, verify) or `XCUIDevice.shared.system.open(_:)` for cold/warm link handling; `launchArguments` to simulate states.
5. **Hosting checks in CI** (AASA 200/no redirect/valid JSON/expected appIDs).
6. **Manual device matrix** with production AASA via CDN: Mail, Gmail, Messages, Notes, messaging/social in-app browsers, Camera QR, Safari cross-domain.

### 14.3 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| D1 | Universal link, app not installed | Web page renders content; Smart App Banner shown |
| D2 | Universal link, cold launch, signed in | Lands on destination with synthesized back stack; no flash of home → push cascade |
| D3 | Universal link warm on another tab; custom scheme from widget | Tab switched, stack replaced; same destination as https form |
| D4 | Signed out: auth-required route / public route | Login with context copy then destination once / opens directly |
| D5 | Link during onboarding | Onboarding continues; destination after completion |
| D6 | Pending link older than TTL; user cancels login | Dropped; normal home; no loop |
| D7 | Link for another signed-in account or portal | Confirm switch → switch → destination |
| D8 | Link for no signed-in account | Add-account flow; destination after sign-in if access |
| D9 | Link while unsaved draft or StoreKit sheet open | Draft not lost: prompt or park until sheet closes |
| D10 | Same link twice quickly; link to screen already showing | One navigation; refresh in place; no duplicate |
| D11 | Unknown path on associated domain | Safari VC or home + "update app" toast; no crash |
| D12 | Malformed ID / injection in param | `.unsupported`; nothing loaded |
| D13 | Content deleted / no access; expired invite / used token | Not-found / no-access / expired state with next action; Back works |
| D14 | Email link via ESP click tracking; short marketing link | App opens and lands correctly; click recorded; resolver timeout → web fallback |
| D15 | QR code via Camera; link in social in-app browser | App opens destination; landing page "Open in app" works |
| D16 | Push tap, cold launch; push for inactive account | Destination shown (delegate set early); account switch first if needed |
| D17 | Quick action cold/warm; Spotlight / Handoff | Correct destination every way |
| D18 | Fresh install from link (deferred) | Lands on destination or graceful home; never blocks onboarding |
| D19 | Magic link on same device | Signed in; token redeemed once; second tap → "link used" |
| D20 | OAuth callback | Handled by auth session only, never by router |
| D21 | Destructive path (`/account/delete`) | Excluded from AASA / opens confirmation, never acts |
| D22 | Offline cold launch with link | Destination screen offline state |
| D23 | iPad, two windows | Routes in active window only |
| D24 | User disabled universal links ("Open in Safari") | Web page usable; banner re-enables |
| D25 | App Clip URL with full app installed | Full app handles same URL |

## 15. Common bugs to look for in review

- Only `scene(_:continue:)` / `openURLContexts` implemented → links dropped on **cold launch** (`connectionOptions` ignored); app-delegate URL methods in a scene-based app → never called
- `.onOpenURL` attached to a child view that isn't mounted at launch, or attached in several places → missed or double-handled links
- AASA served with a redirect (`http→https`, apex→www), behind auth/WAF/bot challenge, with `.json` extension, 404 on `www`, or wrong Team ID
- AASA updated after app release and expected to work immediately (CDN caching), or `mode=developer` entries expected to work in production
- Wildcard `applinks:*.example.com` assumed to cover `example.com`; broad `"/": "*"` capturing `/unsubscribe`, `/oauth/*`, `/help` → users trapped in the app
- Custom scheme used in emails/QR codes; tokens delivered via custom scheme (hijackable)
- Each screen parsing URLs ad hoc; parser with side effects or network; no tests
- Pushing route by route (`path.append` ×3 with animation) instead of assigning the full path; `navigationDestination` inside lazy containers
- Modals not dismissed → navigation happens behind a sheet; or unsaved draft discarded silently
- No back stack synthesis → Back exits to a blank root or the app's previous unrelated screen
- Pending link never cleared (reopens every launch), lost when login replaces the root, or persisted with a token in `UserDefaults`
- Link opens in the wrong account/portal because IDs collide across workspaces
- Links that perform actions (accept invite, follow, delete) without confirmation
- `?redirect=` param loaded in an authenticated WebView → open redirect / token theft
- Email ESP click tracking breaking universal links unnoticed (tested only with raw URLs)
- Magic-link GET consumes token (email scanner pre-fetch burns it); full URLs with tokens logged to analytics/crash reports
- Deferred link lookup blocking first launch on a slow network
- State restoration overriding the link on cold launch
- String-concatenated URLs with unencoded IDs; `+` vs space mismatch with backend
- Firebase Dynamic Links (`page.link`) URLs still in templates/QR codes after the shutdown

## 16. Output format

When asked to **review/audit**:
1. **Summary** — lifecycle (SwiftUI/UIKit scenes/legacy), link types and domains, router type, min OS, multi-account?, deferred strategy; the link spec table as found (or reconstructed)
2. **Hosting report** — per domain: AASA URL status, redirects, content type, Apple CDN copy vs origin, appIDs, components, excludes
3. **Checklist results** — sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
4. **Critical issues first** (cold-launch drops, hijackable tokens, destructive actions from links, open redirects, wrong-account routing, broken hosting) with fix code
5. **Testing gaps** — D1–D25 not covered

When asked to **implement**: write the link spec, start from the Appendix (parser + tests first, then coordinator/pending store, then router wiring for the app's lifecycle), produce the AASA file and entitlements, delete unused parts (multi-account, App Clip, resolver), and list server/web dependencies (web pages for every route, resolve endpoint, AASA deploy) and OS-gated APIs used.

When asked to **debug** "link doesn't open the app": check in order — entitlement in the signed build (`codesign -d --entitlements :- App.app`), AASA reachable with no redirect, Apple CDN copy, components match (Diagnostics), how the link was opened (typed in Safari? same domain? in-app browser? click-tracking domain?), user-disabled per domain. "Opens app but wrong screen": cold vs warm path, parser test for that URL, gates/pending store, modal state, router assignment.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI with UIKit scene notes)

Adjust for your deployment target. Replace `example.com`, `myapp`, Team ID and routes with your link spec.

### A.1 DeepLink model

```swift
import Foundation

/// Everything a link can mean. Payloads are validated IDs only — never models, never raw URLs to load.
enum DeepLink: Hashable, Sendable {
    case home
    case product(id: String, tab: ProductTab?)
    case order(id: String)
    case project(portalID: String, projectID: String)
    case invite(token: String)
    case magicLink(token: String)
    case notificationSettings
    case offer(code: String)
    case unsupported(URL)

    enum ProductTab: String, Sendable { case overview, reviews }
    enum Access: Sendable { case `public`, requiresAuth, isAuth }

    var access: Access {
        switch self {
        case .home, .product, .unsupported: .public
        case .magicLink: .isAuth
        case .invite: .public            // landing is public; accepting requires auth + confirmation
        case .order, .project, .notificationSettings, .offer: .requiresAuth
        }
    }

    /// Portal scope encoded in the URL, if any (account is resolved separately).
    var portalID: String? {
        if case let .project(portalID, _) = self { return portalID }
        return nil
    }

    /// Stable name for analytics — never the raw URL.
    var analyticsName: String {
        switch self {
        case .home: "home"
        case .product: "product"
        case .order: "order"
        case .project: "project"
        case .invite: "invite"
        case .magicLink: "magic_link"
        case .notificationSettings: "notification_settings"
        case .offer: "offer"
        case .unsupported: "unsupported"
        }
    }
}

enum LinkSource: String, Sendable {
    case universalLink = "universal_link", customScheme = "custom_scheme", push, widget
    case quickAction = "quick_action", spotlight, deferred, internalLink = "internal"
}
```

### A.2 Parser (pure, nonisolated)

```swift
import Foundation

struct DeepLinkParser: Sendable {
    let associatedHosts: Set<String>     // lowercased, without "www."
    let customScheme: String

    static let live = DeepLinkParser(
        associatedHosts: ["example.com", "go.example.com"],
        customScheme: "myapp"
    )

    private static let idPattern = /^[A-Za-z0-9_-]{1,64}$/
    private static let tokenPattern = /^[A-Za-z0-9_-]{16,256}$/
    private static let offerPattern = /^[A-Z0-9]{4,20}$/

    func parse(_ url: URL) -> DeepLink {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let scheme = components.scheme?.lowercased() else { return .unsupported(url) }

        // Normalise both forms to a list of path segments.
        // https://example.com/p/123  → ["p","123"]
        // myapp://p/123              → host "p" is the first segment → ["p","123"]
        var segments: [String]
        switch scheme {
        case "https":
            guard var host = components.host?.lowercased() else { return .unsupported(url) }
            if host.hasPrefix("www.") { host.removeFirst(4) }
            guard associatedHosts.contains(host) else { return .unsupported(url) }
            segments = Self.segments(components.percentEncodedPath)
        case customScheme:
            segments = (components.host.map { [$0] } ?? []) + Self.segments(components.percentEncodedPath)
        default:
            return .unsupported(url)
        }
        segments = segments.compactMap { $0.removingPercentEncoding }.filter { !$0.isEmpty }

        let query = Dictionary(
            (components.queryItems ?? []).compactMap { item in item.value.map { (item.name.lowercased(), $0) } },
            uniquingKeysWith: { first, _ in first }
        )

        switch segments.map({ $0.lowercased() }).first {
        case nil:
            return .home
        case "p", "product", "products":                          // "products" = legacy route, kept forever
            guard segments.count == 2, let id = Self.leadingID(segments[1]) else { return .unsupported(url) }
            return .product(id: id, tab: query["tab"].flatMap(DeepLink.ProductTab.init(rawValue:)))
        case "orders":
            guard segments.count == 2, Self.isID(segments[1]) else { return .unsupported(url) }
            return .order(id: segments[1])
        case "w":
            guard segments.count == 4, segments[2].lowercased() == "projects",
                  Self.isID(segments[1]), Self.isID(segments[3]) else { return .unsupported(url) }
            return .project(portalID: segments[1], projectID: segments[3])
        case "invite":
            guard segments.count == 2, Self.isToken(segments[1]) else { return .unsupported(url) }
            return .invite(token: segments[1])
        case "auth":
            guard segments.count == 2, segments[1].lowercased() == "magic",
                  let token = query["token"], Self.isToken(token) else { return .unsupported(url) }
            return .magicLink(token: token)
        case "settings":
            guard segments.count == 2, segments[1].lowercased() == "notifications" else { return .unsupported(url) }
            return .notificationSettings
        case "offer":
            guard segments.count == 2 else { return .unsupported(url) }
            let code = segments[1].uppercased()
            guard code.wholeMatch(of: Self.offerPattern) != nil else { return .unsupported(url) }
            return .offer(code: code)
        default:
            return .unsupported(url)
        }
    }

    // MARK: helpers

    private static func segments(_ path: String) -> [String] {
        path.split(separator: "/", omittingEmptySubsequences: true).map(String.init)
    }
    private static func isID(_ s: String) -> Bool { s.wholeMatch(of: idPattern) != nil }
    private static func isToken(_ s: String) -> Bool { s.wholeMatch(of: tokenPattern) != nil }

    /// "123-red-shoes" → "123" (slug is decoration).
    private static func leadingID(_ s: String) -> String? {
        let id = String(s.prefix { $0 != "-" })
        return isID(id) ? id : nil
    }
}

extension URL {
    /// For logs/analytics: keep scheme, host, path shape; drop query values except an allowlist.
    func redactedForLogging(allowedQuery: Set<String> = ["utm_source", "utm_medium", "utm_campaign", "tab"]) -> String {
        guard var c = URLComponents(url: self, resolvingAgainstBaseURL: false) else { return "<invalid>" }
        c.queryItems = c.queryItems?.map { allowedQuery.contains($0.name.lowercased()) ? $0 : URLQueryItem(name: $0.name, value: "REDACTED") }
        c.fragment = nil
        return c.string ?? "<invalid>"
    }

    var utmParameters: [String: String] {
        let items = URLComponents(url: self, resolvingAgainstBaseURL: false)?.queryItems ?? []
        return Dictionary(items.compactMap { i in
            i.name.lowercased().hasPrefix("utm_") ? i.value.map { (i.name.lowercased(), $0) } : nil
        }, uniquingKeysWith: { a, _ in a })
    }
}
```

### A.3 Parser unit tests (Swift Testing)

```swift
// @Test(arguments:) table: one (rawURL, expected DeepLink) row per link-spec row, legacy route,
// www/case/trailing-slash/slug/utm variant, and both https + myapp:// forms.
// Second table: URLs that must parse to .unsupported — unknown host, "example.com.evil.com" suffix trick,
// http, other scheme, missing/extra segments, encoded traversal/injection, over-long IDs, short tokens.
// Plus: url.redactedForLogging() drops token values but keeps utm_*.
```

### A.4 Pending link store

```swift
import Foundation

struct PendingLink: Codable, Sendable, Equatable {
    let url: URL
    let source: String
    let createdAt: Date
}

/// One slot, last link wins, TTL. Persists only links without secrets (so a sign-up that kills
/// the app — e.g. email verification in Mail — can still resume). Token links stay in memory.
@MainActor
final class PendingLinkStore {
    private let defaults: UserDefaults, ttl: TimeInterval, now: () -> Date
    private let key = "pendingDeepLink.v1"
    private var memory: PendingLink?

    init(defaults: UserDefaults = .standard, ttl: TimeInterval = 30 * 60, now: @escaping () -> Date = Date.init) {
        self.defaults = defaults; self.ttl = ttl; self.now = now
    }

    func save(_ url: URL, source: LinkSource, containsSecret: Bool) {
        let link = PendingLink(url: url, source: source.rawValue, createdAt: now())
        memory = link
        if containsSecret {
            defaults.removeObject(forKey: key)
        } else if let data = try? JSONEncoder().encode(link) {
            defaults.set(data, forKey: key)
        }
    }

    /// Returns the pending link once (and clears it) if still fresh.
    func take() -> PendingLink? {
        let link = memory ?? defaults.data(forKey: key).flatMap { try? JSONDecoder().decode(PendingLink.self, from: $0) }
        clear()
        guard let link, now().timeIntervalSince(link.createdAt) <= ttl else { return nil }
        return link
    }

    var hasPending: Bool { memory != nil || defaults.data(forKey: key) != nil }

    func clear() { memory = nil; defaults.removeObject(forKey: key) }
}
```

### A.5 Router (navigation state) and coordinator (gates)

```swift
import SwiftUI
import Observation

enum AppTab: Hashable { case home, shop, projects, account }

enum Route: Hashable {
    case product(id: String, tab: DeepLink.ProductTab?), orders, order(id: String), project(id: String)
    case settings, notificationSettings
}

enum SheetRoute: Identifiable, Hashable {
    case invite(token: String)
    case paywall(offerCode: String?)
    var id: Self { self }
}

/// Owns all navigation state. Knows nothing about auth or accounts.
@MainActor @Observable
final class AppRouter {
    var selectedTab: AppTab = .home
    var paths: [AppTab: [Route]] = [:]
    var sheet: SheetRoute?
    /// Set by screens with unsaved work (compose, checkout) — the coordinator parks links while true.
    var isInUninterruptibleFlow = false

    func binding(for tab: AppTab) -> Binding<[Route]> {
        Binding(get: { self.paths[tab, default: []] }, set: { self.paths[tab] = $0 })
    }

    /// External links replace the target tab's stack with a synthesized back stack.
    func apply(_ link: DeepLink, presentation: Presentation = .replace) {
        let (tab, stack, sheet) = Self.destination(for: link)
        self.sheet = nil                                            // dismiss any presented sheet first
        selectedTab = tab
        switch presentation {
        case .replace:
            if paths[tab] != stack { paths[tab] = stack }          // same destination → no duplicate push
        case .push:
            if let last = stack.last, paths[tab, default: []].last != last { paths[tab, default: []].append(last) }
        }
        if let sheet { self.sheet = sheet }
    }

    enum Presentation { case replace, push }

    /// The link spec, in code. Keep in sync with section 1 table.
    static func destination(for link: DeepLink) -> (AppTab, [Route], SheetRoute?) {
        switch link {
        case .home, .unsupported, .magicLink:
            (.home, [], nil)
        case let .product(id, tab):
            (.shop, [.product(id: id, tab: tab)], nil)
        case let .order(id):
            (.account, [.orders, .order(id: id)], nil)
        case let .project(_, projectID):
            (.projects, [.project(id: projectID)], nil)
        case .notificationSettings:
            (.account, [.settings, .notificationSettings], nil)
        case let .invite(token):
            (.home, [], .invite(token: token))
        case let .offer(code):
            (.home, [], .paywall(offerCode: code))
        }
    }
}

/// Abstractions over app state the gates need (implemented by your session/onboarding layers).
@MainActor protocol LinkGateContext: AnyObject {
    var isUIReady: Bool { get }
    var hasCompletedOnboarding: Bool { get }
    var isSignedIn: Bool { get }
    var activePortalID: String? { get }
    /// Returns which signed-in account/portal can open the link, if known (see ios-multi-account).
    func owner(of link: DeepLink) async -> LinkOwner
}

enum LinkOwner: Sendable { case active, otherSession(accountID: String, portalID: String?), noSignedInAccount, unknown }

enum LinkOutcome: String { case routed, parkedNotReady, parkedOnboarding, parkedAuth, parkedFlow, needsSwitch, needsAddAccount, authFlow, unsupported }

@MainActor @Observable
final class DeepLinkCoordinator {
    private let parser: DeepLinkParser
    private let router: AppRouter
    private let pending: PendingLinkStore
    private unowned let context: LinkGateContext
    private let analytics: (String, [String: String]) -> Void
    private let openInSafari: (URL) -> Void
    private let handleAuthLink: (DeepLink) -> Void
    private var lastHandled: (url: URL, at: Date)?

    /// UI binds to this to show "Switch to Work account to open this?"
    var switchPrompt: (link: DeepLink, url: URL, accountID: String, portalID: String?)?

    init(parser: DeepLinkParser = .live, router: AppRouter, pending: PendingLinkStore, context: LinkGateContext,
         analytics: @escaping (String, [String: String]) -> Void,
         openInSafari: @escaping (URL) -> Void,
         handleAuthLink: @escaping (DeepLink) -> Void) {
        self.parser = parser; self.router = router; self.pending = pending; self.context = context
        self.analytics = analytics; self.openInSafari = openInSafari; self.handleAuthLink = handleAuthLink
    }

    /// Single entry point for every source.
    @discardableResult
    func handle(url: URL, source: LinkSource, isColdLaunch: Bool = false) async -> LinkOutcome {
        // Dedupe: SwiftUI may deliver a universal link via onOpenURL and onContinueUserActivity.
        if let last = lastHandled, last.url == url, Date().timeIntervalSince(last.at) < 1 { return .routed }
        lastHandled = (url, Date())

        let link = parser.parse(url)
        let outcome = await route(link, url: url, source: source)
        let params = ["source": source.rawValue, "route": link.analyticsName, "result": outcome.rawValue,
                      "launch_type": isColdLaunch ? "cold" : "warm"]
        analytics("deep_link_opened", url.utmParameters.merging(params) { _, new in new })
        return outcome
    }

    private func route(_ link: DeepLink, url: URL, source: LinkSource) async -> LinkOutcome {
        if case .unsupported = link {
            if url.scheme == "https" { openInSafari(url) }
            return .unsupported
        }
        let secret: Bool = { switch link { case .magicLink, .invite: true; default: false } }()

        guard context.isUIReady else { pending.save(url, source: source, containsSecret: secret); return .parkedNotReady }
        if link.access == .isAuth { handleAuthLink(link); return .authFlow }
        guard context.hasCompletedOnboarding else { pending.save(url, source: source, containsSecret: secret); return .parkedOnboarding }
        if link.access == .requiresAuth, !context.isSignedIn {
            pending.save(url, source: source, containsSecret: secret); return .parkedAuth
        }
        guard !router.isInUninterruptibleFlow else { pending.save(url, source: source, containsSecret: secret); return .parkedFlow }

        if link.access == .requiresAuth {
            switch await context.owner(of: link) {
            case .active, .unknown:
                break                                   // destination screen handles 403/404
            case let .otherSession(accountID, portalID):
                switchPrompt = (link, url, accountID, portalID)
                return .needsSwitch
            case .noSignedInAccount:
                pending.save(url, source: source, containsSecret: secret)
                return .needsAddAccount
            }
        }

        let presentation: AppRouter.Presentation = source == .internalLink ? .push : .replace
        router.apply(link, presentation: presentation)
        return .routed
    }

    /// Call when a gate clears: UI mounted, onboarding finished, login succeeded, account switched, flow ended.
    func replayPendingIfAny() async {
        guard let link = pending.take() else { return }
        analytics("pending_link_replayed", ["source": link.source])
        lastHandled = nil
        await handle(url: link.url, source: LinkSource(rawValue: link.source) ?? .universalLink)
    }

    /// After the user confirms the switch prompt; the multi-account layer performs the switch.
    func confirmSwitch(perform switchSession: (String, String?) async -> Void) async {
        guard let prompt = switchPrompt else { return }
        switchPrompt = nil
        pending.save(prompt.url, source: .internalLink, containsSecret: false)
        await switchSession(prompt.accountID, prompt.portalID)   // tears down + rebuilds session graph
        await replayPendingIfAny()                              // now owner == .active
    }

    func cancelSwitch() { switchPrompt = nil }
    func onSignOut() { pending.clear(); switchPrompt = nil }
}
```

### A.6 SwiftUI wiring

```swift
import SwiftUI
import UserNotifications

@main
struct ExampleApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    @State private var env = AppEnvironment()          // owns router, coordinator, session

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(env.router)
                .environment(env.coordinator)
                // Attach once, at the root. Receives custom schemes, universal links and widget URLs.
                .onOpenURL { url in
                    Task { await env.coordinator.handle(url: url, source: url.scheme == "https" ? .universalLink : .customScheme) }
                }
                .onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
                    guard let url = activity.webpageURL else { return }
                    Task { await env.coordinator.handle(url: url, source: .universalLink) }
                }
                .task {
                    env.markUIReady()                       // sets isUIReady, then:
                    await env.coordinator.replayPendingIfAny()
                }
                .onAppear { appDelegate.env = env }
        }
    }
}

// RootView (sketch): TabView(selection: $router.selectedTab) with one NavigationStack(path: router.binding(for: tab))
// per tab, .navigationDestination(for: Route.self), .sheet(item: $router.sheet), a confirmationDialog bound to
// coordinator.switchPrompt (Switch → coordinator.confirmSwitch(perform:) via ios-multi-account; Cancel → cancelSwitch()),
// and .environment(\.openURL, OpenURLAction { … }) so internal links that parse route in-app with source .internalLink
// (push) and unsupported ones return .systemAction.

/// Push taps: set the delegate early so cold-launch taps are delivered.
final class AppDelegate: NSObject, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    var env: AppEnvironment?

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        return true
    }

    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                didReceive response: UNNotificationResponse) async {
        guard let raw = response.notification.request.content.userInfo["url"] as? String,
              let url = URL(string: raw) else { return }
        // accountId / portalId in userInfo are resolved by LinkGateContext.owner(of:) — see ios-push-notifications.
        // If env is not attached yet on cold launch, the coordinator parks the link via isUIReady anyway.
        await MainActor.run { if let env { Task { await env.coordinator.handle(url: url, source: .push) } } }
    }
}
```

UIKit scene equivalent (replace the SwiftUI entry points, keep parser/coordinator/router):

```swift
import UIKit

final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private var env: AppEnvironment { (UIApplication.shared.delegate as! AppDelegate).env! }

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        // … create window, set root VC (coordinators observe AppRouter), makeKeyAndVisible, env.markUIReady()
        // COLD LAUNCH — the step most implementations forget.
        if let url = connectionOptions.userActivities.first(where: { $0.activityType == NSUserActivityTypeBrowsingWeb })?.webpageURL {
            Task { await env.coordinator.handle(url: url, source: .universalLink, isColdLaunch: true) }
        } else if let url = connectionOptions.urlContexts.first?.url {
            Task { await env.coordinator.handle(url: url, source: .customScheme, isColdLaunch: true) }
        } else {
            Task { await env.coordinator.replayPendingIfAny() }
        }
    }

    // WARM: scene(_:continue:) → userActivity.webpageURL (.universalLink);
    //       scene(_:openURLContexts:) → URLContexts.first?.url (.customScheme, widget URLs). Same coordinator.handle.
    // Quick actions: map connectionOptions.shortcutItem (cold) and windowScene(_:performActionFor:completionHandler:)
    // (warm) to item.userInfo["url"] → coordinator.handle(url:source: .quickAction).
}
```

`AppEnvironment` (not shown) constructs `AppRouter`, `PendingLinkStore`, the session/onboarding objects implementing `LinkGateContext`, and `DeepLinkCoordinator`; it calls `coordinator.replayPendingIfAny()` after login success, onboarding completion, account switch, and when `isInUninterruptibleFlow` flips to `false`, and `coordinator.onSignOut()` on sign-out.

### A.7 Resolving short links and email click-tracking URLs

```swift
// LinkResolver (sketch): for wrapper hosts (go./click.example.com) issue a GET (trackers often ignore HEAD) on an
// ephemeral URLSession with a ~3 s timeout and a delegate that refuses redirects; take the 3xx Location, and accept it
// only if DeepLinkParser parses it to a supported link (never an open redirect). Otherwise return the original URL.
// In DeepLinkCoordinator.handle: if resolver.needsResolution(url) { url = await resolver.resolve(url, parser: parser) }
// On failure/timeout the original URL is unsupported → opened in Safari, so the user still gets the web page.
```

### A.8 apple-app-site-association (and Android sibling's assetlinks.json)

Served at `https://example.com/.well-known/apple-app-site-association` (and on every associated host), `Content-Type: application/json`, no redirects:

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["ABCDE12345.com.example.app"],
        "components": [
          { "/": "/oauth/*", "exclude": true }, { "/": "/unsubscribe*", "exclude": true },
          { "/": "/account/delete", "exclude": true }, { "/": "/help/*", "exclude": true }, { "/": "/legal/*", "exclude": true },
          { "/": "*", "?": { "web": "1" }, "exclude": true, "comment": "Escape hatch: force web" },
          { "/": "/p/*" }, { "/": "/products/*", "comment": "Legacy route, keep forever" }, { "/": "/orders/*" },
          { "/": "/w/*/projects/*" }, { "/": "/invite/*" }, { "/": "/auth/magic", "?": { "token": "?*" } },
          { "/": "/settings/notifications" }, { "/": "/offer/*" }
        ]
      }
    ]
  },
  "webcredentials": { "apps": ["ABCDE12345.com.example.app"] },
  "appclips": { "apps": ["ABCDE12345.com.example.app.Clip"] }
}
```

Entitlements: `applinks:` for every host above (`example.com`, `www.`, `go.`, `click.`) plus `webcredentials:example.com`; Debug adds `applinks:staging.example.com?mode=developer`. The **android-deep-links** sibling's `assetlinks.json` lives on the same domain (Play App Signing SHA-256 plus upload/debug keys) — see that skill's Appendix A.2.

Hosting checklist for both files: HTTPS, 200, no redirects, `application/json`, excluded from auth/WAF challenges, deployed by CI with a post-deploy fetch check, and changed *before* the app release that depends on new paths.
