---
name: flutter-multi-account
description: Use when implementing, reviewing, testing, or debugging multi-account and multi-portal (workspace / organization / team) support in a Flutter consumer app — an account registry with several signed-in identities, per-account tokens in flutter_secure_storage, per-account (and per-portal) Drift / Isar / Hive / sqflite databases, dio clients with per-account auth interceptors and single-flight token refresh, X-Portal-Id headers or portal-scoped tokens, a per-session Riverpod ProviderContainer / ProviderScope override (or Bloc/get_it scope) that is torn down on switch, go_router reset, account and portal switcher UI, add account, sign out one vs all, push and deep links routed to the right account+portal, background work, purchases, analytics and privacy. Siblings: ios-multi-account, android-multi-account, react-native-multi-account; related: flutter-deep-links, flutter-push-notifications, flutter-in-app-purchase, flutter-revenuecat-iap, flutter-list-ux.
---

# Flutter Multi-Account Skill

Guide for building, auditing, and testing apps where **several user identities are signed in on one device** and **each identity can belong to several portals** (workspaces, organizations, teams, households, stores). The user switches **account** (heavy: different credentials, different data, different backend region maybe) and switches **portal within an account** (lighter: same credentials, different tenant). Both layers have to be modelled everywhere — storage, networking, caches, DI, navigation, push, links, background work, purchases, analytics. Siblings: **ios-multi-account**, **android-multi-account**, **react-native-multi-account**. Related: **flutter-deep-links**, **flutter-push-notifications**, **flutter-in-app-purchase**, **flutter-revenuecat-iap**, **flutter-list-ux**.

Flutter packages move fast. Before shipping, **verify the current API** of: `flutter_secure_storage` (v10 deprecated `AndroidOptions(encryptedSharedPreferences: true)` and migrates to its own ciphers — read the migration notes before changing options on an installed base; iOS `accessibility` / `synchronizable` semantics), `flutter_riverpod` (Riverpod 3 changed `Ref` typing, added `ref.mounted`, auto-retry, and removed `StateNotifier` from the main import; `UncontrolledProviderScope` + `ProviderContainer` is what lets you own a container's lifetime), `go_router` (redirect signature, `refreshListenable`, `StatefulShellRoute` state restoration), `dio` (v5 `QueuedInterceptor`, `DioException`), `drift` (`NativeDatabase.createInBackground`, `drift_flutter`'s `driftDatabase(name:)`), `isar` (the original package is unmaintained; many projects moved to `isar_community` or off Isar — check which fork is in `pubspec.yaml`), `hive` / `hive_ce`, `firebase_messaging`, `purchases_flutter`. Check `pubspec.yaml` and `flutter --version`; don't trust memory.

## 0. Mental model

```
                         App process (lives forever)
+--------------------------------------------------------------------------------+
| AccountRegistry (persisted, non-secret)    SecureTokenStore (flutter_secure_    |
|  accounts: [A, B, C] ordered               storage, key = tokens.<accountId>)   |
|  active: (accountId, portalId)             PushRouter, LinkRouter, Analytics    |
|  per account: portals[], lastPortalId                                           |
+-----------------------------+--------------------------------------------------+
                              | switchAccount(B) / switchPortal(p2)
                              v
        +-------------------- AccountSession(B) -------------------------+
        | dio (base URL = B.region, AuthInterceptor(B), single-flight)   |
        | AccountDatabase  -> app_support/accounts/<B>/main.sqlite        |
        | caches, sockets, account-level repositories                      |
        |   +------------- PortalSession(B, p2) --------------------+     |
        |   | X-Portal-Id: p2 header / portal token                 |     |
        |   | portal DB or portal_id-scoped queries, query caches   |     |
        |   | portal-level providers / blocs, realtime channel      |     |
        |   +-------------------------------------------------------+     |
        +-----------------------------------------------------------------+
                              |
                Riverpod: one ProviderContainer per (account) with a child
                container per (account, portal), mounted via
                UncontrolledProviderScope(key: ValueKey(sessionKey))
                go_router: new router / router.go(initialLocation) per session
```

Rules that follow from the model:

1. **Everything user-scoped lives inside a session object**, never in a global singleton, a top-level `final`, a `static`, or a root-level provider that isn't keyed by the session.
2. **Switching = dispose the old session, build a new one.** Not "update a few fields". Disposing is what cancels requests, closes DBs and sockets, and drops caches.
3. **The registry and the token store are the only process-global user data**, and they are keyed by `accountId`.
4. **Account switch disposes both layers; portal switch disposes only the portal layer.**

## 1. First, establish context

Find out (from the codebase or by asking):

- **Max accounts** per device (2? 5? unlimited?) and whether that's a product rule or a backend/licence rule.
- **Backends**: one API, or per-account **region / environment** (EU vs US host, staging accounts for internal users, self-hosted tenant URLs)? This decides whether `baseUrl` is per account.
- **Identity provider**: own email/password, OAuth/OIDC (`flutter_appauth`), Firebase Auth (single-user SDK — see §8), Sign in with Apple / Google (single-user SDKs too).
- **Portals per account**: typical and maximum (3 vs 300 → picker needs search). Can a user have **zero** portals? Is there a personal portal?
- **Portal auth contract**: one account token + `X-Portal-Id` header? Per-portal token exchanged from the account token? Portal in URL path (`/v1/portals/{id}/...`)? Roles in token claims vs fetched?
- **Data isolation**: what's shared between accounts (nothing, ideally) and between portals of one account (user profile, settings, maybe DMs)? Offline DB? Which engine (Drift, Isar, Hive, sqflite, ObjectBox)?
- **Realtime**: websockets / SSE / Firestore listeners — per account or per portal?
- **Push**: one FCM/APNs device token for the app; does the backend support registering it for several accounts? Does payload carry `account_id` and `portal_id`?
- **Purchases**: is a subscription owned by the store user, the app account, or the portal (team plan)?
- **State management / DI**: Riverpod (version?), Bloc + `get_it`/`injectable`, Provider. Router: `go_router`, `auto_route`, Navigator 1.0. Follow the project's convention.
- **Security requirements**: app lock / biometrics per account? Managed devices? Data residency?
- Min Flutter / Dart SDK (sealed classes and patterns need Dart 3).

## 2. Data model

- [ ] **Account registry** persisted outside the secure store (non-secret metadata). Fields per account: `id` (server user id — stable, not the email), `email`/`displayName`/`avatarUrl` (for the switcher), `environment` / `region` / `baseUrl`, `addedAt`, `lastActiveAt`, `needsReauth` flag, `portals: List<PortalRef>`, `lastPortalId`.
- [ ] Registry storage: a small JSON file in `getApplicationSupportDirectory()` or a tiny app-level Drift DB — **not** inside any account's DB (you need it before any account opens). `shared_preferences` is acceptable for this non-secret JSON but nothing else.
- [ ] **Account key** = `"$environment:$userId"` (or issuer + subject). The same user id on staging and prod is two accounts. Email is not a key (it changes; case differs).
- [ ] **PortalRef**: `id`, `name`, `role` (`owner/admin/member/guest`), `iconUrl`, `lastUsedAt`, `unreadCount?`. The source of truth is the server; the registry caches it for the switcher and cold start.
- [ ] **Active pair** `(accountId, portalId)` persisted; restored on cold start. Validate on restore: account still in registry, token present, portal still in that account's list — otherwise fall back (§7).
- [ ] **Last portal per account** persisted, so switching back to account A lands in A's last portal, not A's first.
- [ ] **Ordering**: explicit order (user-reorderable) or by `addedAt`; never by map iteration order. Active account shown first in the switcher or marked.
- [ ] **Limits** enforced in one place (`AccountRegistry.add` throws `TooManyAccounts`), surfaced before the user goes through the login flow.
- [ ] Model with immutable classes (`freezed` or hand-written `copyWith`) and a single writer (the registry) so the switcher, push router, and link router read consistent state.
- [ ] Schema version in the registry JSON for migrations (single-account → multi-account migration is the usual first one: move the legacy token key and DB file under the account id).

## 3. Secure token storage

- [ ] Tokens (access, refresh, id token, portal tokens) go in **`flutter_secure_storage`** — iOS/macOS Keychain, Android Keystore-backed encryption. **Never** in `shared_preferences`, Hive boxes, Drift tables, or the registry JSON.
- [ ] **One key per account**: `tokens.<accountKey>` holding a JSON blob `{access, refresh, expiresAt, portalTokens: {..}}`. One blob per account makes writes atomic per account and deletion trivial.
- [ ] iOS options: `IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device)` — `first_unlock` so background refresh and push handlers can read after reboot+unlock; `_this_device` so tokens don't migrate to a new phone via encrypted backup (the user re-authenticates instead). Don't use `synchronizable: true` (iCloud Keychain) for session tokens.
- [ ] Android: check the plugin version's defaults. On v9 projects use `AndroidOptions(encryptedSharedPreferences: true)` consistently; on v10+ follow the plugin's cipher migration guidance. **Changing options on an installed base can make existing values unreadable** — ship a migration that reads with old options and rewrites.
- [ ] **Backup exclusion (Android)**: `android:allowBackup="false"` or `dataExtractionRules` / `fullBackupContent` excluding the secure-storage prefs file and account DBs. Restored ciphertext without the Keystore key throws on read — the plugin may surface it as `PlatformException`; catch it, wipe that key, mark account `needsReauth`.
- [ ] **Backup exclusion (iOS)**: Keychain `_this_device` items aren't restored to other devices. Account DB files in Application Support *are* backed up by default; set `NSURLIsExcludedFromBackupKey` via a small method channel (or a plugin) on each account directory, or decide consciously that restored DBs are fine (they're useless without tokens — but they still contain PII in the backup).
- [ ] **Keychain survives uninstall on iOS.** On first launch after install (no registry file, or an `installId` mismatch), **wipe** the secure store so a reinstalled app doesn't resurrect old sessions.
- [ ] Cache tokens in memory inside the session; read from secure storage on session open and on refresh. Secure storage calls are async platform-channel calls (and slow on some Android devices) — not per request.
- [ ] Serialize writes per account (a per-account `Mutex`/`Future` chain) so a refresh writing and a sign-out deleting don't interleave.
- [ ] Never log tokens (dio `LogInterceptor(requestHeader: false)` in release, or redact `Authorization`).

## 4. Per-account token lifecycle

- [ ] **One dio instance per account session**, with its own `baseUrl` and its own `AuthInterceptor` bound to that account's token holder. A global dio whose interceptor reads "the current account" will attach B's token to a request A started before the switch.
- [ ] **Capture the account at request creation**, not at send/retry time. The interceptor is constructed with `accountKey`; it never looks up "active".
- [ ] **Single-flight refresh per account**: concurrent 401s share one in-flight refresh `Future`; when it completes, all waiters retry with the new token. Use `QueuedInterceptor` or a `Completer` held in the token holder (Appendix A.3). Refresh for account A must not block account B.
- [ ] Retry each original request **once** after refresh; mark it (`options.extra['retried'] = true`) to avoid loops. Don't run the refresh request through the same auth interceptor (use a bare dio).
- [ ] Proactive refresh when `expiresAt - now < skew` (30–60 s) avoids a 401 round-trip; still handle 401 reactively.
- [ ] **Refresh failure semantics**: `400 invalid_grant` / `401` on refresh → that account is **revoked**: delete its tokens, set `needsReauth = true`, keep it in the registry (greyed, "Sign in again") — don't silently remove it, and **don't touch other accounts**. Network error on refresh → not revoked; keep tokens, surface offline.
- [ ] If the revoked account is the active one: show a non-dismissable re-auth sheet for that account or offer switching to another signed-in account. If it's inactive: badge it in the switcher; push for it is suppressed until re-auth.
- [ ] Rotating refresh tokens: persist the new refresh token **before** releasing waiters; if the app dies between server rotation and the write, the account is effectively revoked — accept and handle via `needsReauth`.
- [ ] **Portal scoping**:
  - Header model: a `PortalInterceptor` adds `X-Portal-Id` from the **portal session it belongs to** (captured), not a mutable global. Account-level endpoints (profile, portal list) go through a client without the header.
  - Token-exchange model: per-portal token cached in the account's token blob, refreshed single-flight **per (account, portal)**; account token refresh may invalidate portal tokens — clear them.
  - Path model: repositories built with `portalId` baked in.
- [ ] `403` with a "not a member" / "portal disabled" code ≠ token problem. Route it to portal-revoked handling (§7), not to refresh.
- [ ] On every response for a stale session (session disposed), drop the result: check a `session.isDisposed` / `CancelToken` before writing to the DB or state.

## 5. Data isolation

- [ ] **One database per account**: `getApplicationSupportDirectory()/accounts/<accountHash>/main.sqlite` (Drift/sqflite), `.../isar/` (Isar `directory:`), or Hive boxes named/pathed per account (`Hive.openBox('msgs', path: accountDir)` or box name prefix). Use a hash or the opaque id for the folder name — not the email.
- [ ] **Per portal**: either a DB per (account, portal) when portals are large and independent (deleting a portal = deleting a file), or `portal_id` column on every portal-scoped table with every query filtered by it (index it). Pick one per app and enforce it — a missed `WHERE portal_id = ?` is a cross-tenant leak. With Drift, put the portal filter in the DAO constructor so callers can't forget it.
- [ ] Open DBs **inside the session** and close them on dispose (`await db.close()`; Isar `close()`; Hive `box.close()`). Drift: `NativeDatabase.createInBackground(file)` per account; never a top-level `final db = AppDatabase()`.
- [ ] **HTTP cache**: `dio_cache_interceptor` store per account directory (or keyed by account) — a shared cache can serve A's `/me` to B.
- [ ] **Image cache**: `cached_network_image` uses a global `DefaultCacheManager`. For authenticated images use a **per-account `CacheManager(Config('img_<accountHash>'))`** and pass auth headers from the session; clear it on account removal. Also `PaintingBinding.instance.imageCache.clear()` on account switch (in-memory decoded images keyed by URL can leak avatars across accounts when URLs are shared paths like `/me/avatar`).
- [ ] **Query caches**: Riverpod providers / Bloc states / `flutter_query`-style caches live in the session container and die with it (§6).
- [ ] **Files** (downloads, drafts, attachments, exports): under the account (and portal) directory; temp files in `getTemporaryDirectory()/accounts/<hash>/`.
- [ ] **Prefs**: per-account settings keyed `"<accountHash>.<key>"` or in the account DB. App-wide settings (theme, locale) stay global — decide explicitly which are which.
- [ ] **No leaks via singletons**: grep for `static`, top-level `final`/`late`, `get_it` `registerSingleton` of anything user-scoped, `Provider` without `autoDispose` that caches user data at the root container, `StreamController.broadcast()` globals, `GetX` `Get.put` permanent controllers.
- [ ] Web/desktop: same rules; on web IndexedDB/`drift` wasm DB names must be per account.

## 6. Account switch

Ordered teardown → rebuild. Make it **one async function** owned by a `SessionManager`, serialized (a second switch request while one is running is queued or replaces the pending target — never interleaved).

- [ ] **1. Freeze UI**: show a full-screen switching overlay (avatar + name of the target, spinner after ~300 ms) so the old account's screens aren't interactive during teardown.
- [ ] **2. Save restoration state** of the outgoing account: current location (`GoRouterState.uri`), maybe tab index; store in the registry as `lastLocation` (only routes that are safe to restore — not checkout, not modals).
- [ ] **3. Teardown old session**: cancel in-flight requests (shared `CancelToken` per session, `dio.close(force: true)`), close websockets/SSE, cancel stream subscriptions and timers, cancel session-scoped background work, dispose the `ProviderContainer` (runs every `ref.onDispose`) or close all session Blocs / `getIt.popScope()`, close DBs, clear in-memory image cache.
- [ ] **4. Persist active pair** `(B, B.lastPortalId)` **after** teardown and **before** building, so a crash mid-switch restarts into B.
- [ ] **5. Build new session**: read tokens, open DB, create dio, create container with overrides (`sessionProvider.overrideWithValue(session)`), switch analytics/crash user, IAP `logIn` (§13).
- [ ] **6. Reset navigation**: mount the new session subtree under a **new key** (`UncontrolledProviderScope(key: ValueKey(session.key), ...)`) and give it a fresh `GoRouter` (or `router.go(restoredOrHome)` if the router lives above the session). Never `push` onto the old stack — Back would return to account A's screens rendered with B's providers.
- [ ] **7. Unfreeze**; first screen shows cached data from B's DB immediately and revalidates.
- [ ] Riverpod: pick one pattern and apply it consistently:
  - **Container per session** (recommended): `ProviderContainer(parent: appContainer, overrides: [...])` held by the `SessionManager`, mounted with `UncontrolledProviderScope`, `container.dispose()` on teardown. App-global providers (registry, secure store, push router) live in the parent.
  - **Family keyed by session**: every user-scoped provider `.family(SessionKey)` + `autoDispose` — workable but easy to miss one; a lint/review rule is needed.
  - **`ref.watch(activeSessionProvider)` everywhere** so dependents rebuild on switch — the weakest; old async work still completes and writes into new state unless every notifier checks `ref.mounted`.
- [ ] Bloc + get_it: `getIt.pushNewScope(scopeName: 'session', init: ...)` on build, `await getIt.popScope()` on teardown (dispose callbacks close Blocs/DBs); `MultiBlocProvider` for session Blocs placed under a `KeyedSubtree(key: ValueKey(session.key))`.
- [ ] **Switcher UI**: avatar in the app bar / profile tab → bottom sheet listing accounts (avatar, name, email, region badge, unread badge, `needsReauth` state), active one checked, "Add account", "Manage accounts". Long-press avatar for quick switch is a nice extra. Each row has a semantics label like "Switch to Jane Doe, jane@acme.com".
- [ ] Haptic + brief toast "Switched to Jane (Acme)" after switching — the user must never be unsure which account they're acting as. Show the account avatar persistently on compose/send/pay screens.
- [ ] **Restore last screen per account** only from an allow-list; fall back to home.
- [ ] Switch latency target: < 500 ms to first frame with cached data. Don't block on network.

## 7. Portal switch

Lighter than an account switch: same tokens, same account DB, same dio (or a cloned one with a different `X-Portal-Id`), same account-level caches.

- [ ] **Reset**: portal child container (portal-scoped providers/blocs), portal DB or portal-filtered query caches, portal realtime channel, portal navigation stack (go to portal home or restored portal location), portal analytics context, portal-scoped background work, per-portal permissions.
- [ ] **Keep**: account session (tokens, dio, account DB connection), account-level providers (profile, portal list, account settings), image disk cache (URLs are portal-specific anyway — verify), theme.
- [ ] Riverpod: child `ProviderContainer(parent: accountContainer, overrides: [portalProvider.overrideWithValue(portal)])`, mounted under `UncontrolledProviderScope(key: ValueKey('$account/$portal'))`. Account-level providers resolve from the parent and survive.
- [ ] **Picker UX**: current portal at top; **recent portals** (by `lastUsedAt`, 3–5); full list alphabetical; **search field when > ~8 portals** (local filter, debounced; server search when the list is paged); role badge; unread counts if available; "Join or create portal". Portal name + icon always visible in the app bar so the context is obvious.
- [ ] **Default portal on login**: server-provided default → last used on this device → only portal → picker. Zero portals → onboarding "create or join" screen, not an empty home.
- [ ] **Portal removed / access revoked mid-session** (403 membership code, websocket event, portal list refresh missing it): stop portal work, show "You no longer have access to Acme" and switch to the next portal (last used, else first, else zero-portal screen). Delete that portal's local data. Don't loop on 403 retries.
- [ ] **Role/permission change**: refetch permissions on portal open, on resume, and on a realtime "membership updated" event; derive UI capabilities from a `PortalPermissions` provider, never from cached role strings sprinkled in widgets. Server remains the enforcer.
- [ ] **Invitations**: accepting an invite (often via link) adds a portal to an account → refresh portal list, then offer "Open Acme now?". Invite for an email that matches another signed-in account → offer that account; matches none → "Sign in or add account".
- [ ] Refresh the portal list on login, on resume (throttled), and on pull-to-refresh in the picker.
- [ ] Persist `lastPortalId` per account on every portal switch.

## 8. Add account

- [ ] "Add account" runs the login flow **without tearing down the active session first** — present login as a full-screen route *above* the current session (or in a temporary "login session" container with no account). Cancel → back exactly where the user was.
- [ ] Login flow uses a **bare dio** (no auth interceptor) pointed at the chosen environment/region. If region is discovered from the email or from the token, resolve it before saving the account.
- [ ] **Duplicate detection** after authentication by account key: already present → don't create a second entry; update its tokens (re-auth), clear `needsReauth`, then switch to it with a toast "Already signed in — switched".
- [ ] Enforce the max-accounts limit **before** showing the login form.
- [ ] Single-user SDKs: Firebase Auth's `FirebaseAuth.instance` holds one user — for multiple accounts either use a secondary `Firebase.initializeApp(name: accountKey)` per account (each `FirebaseAuth.instanceFor(app:)` keeps its own user), or use the IdP only to obtain tokens and manage sessions yourself. Google Sign-In / `flutter_appauth`: clear the SDK's cached browser session (`prompt=login` / `select_account`) so "Add account" doesn't silently log in as the existing user.
- [ ] On success: write tokens → add to registry → fetch portal list → switch to the new account (§6) with the default portal (§7).
- [ ] Failure halfway (tokens written, registry write failed): on next launch reconcile — secure-store keys without registry entries are deleted.

## 9. Sign out

- [ ] Offer **"Sign out of this account"** and **"Sign out of all accounts"** separately. Make it clear which account a sign-out affects (avatar + email in the confirm dialog).
- [ ] Order for one account: server revoke (best effort, with short timeout; queue retry if offline — don't block the user) → unregister push token for that account (§10) → IAP/analytics cleanup if active → teardown session if active → delete tokens → delete DB files, cache dirs, per-account prefs, image `CacheManager.emptyCache()` → remove from registry.
- [ ] **Next active account**: most recently used remaining account (its last portal), else signed-out landing screen. Never auto-pick an account in `needsReauth` state without telling the user.
- [ ] Signing out an **inactive** account must not touch the active session.
- [ ] "Sign out all": teardown active, then loop accounts; finish with full wipe of `accounts/` dir and secure store (`deleteAll()` only if the store holds nothing app-global you need).
- [ ] Removing the **last** account → onboarding/login route, router reset, global caches cleared.
- [ ] Deletion must survive process death: mark `pendingDeletion` in the registry first; on launch, finish any pending deletions before opening sessions.

## 10. Push per account / portal

See **flutter-push-notifications** for setup. Multi-account specifics:

- [ ] There is **one device token** (FCM/APNs) per app install. Register it with the backend **for every signed-in account** (each account's dio), including inactive ones, if the product wants notifications for inactive accounts. Re-register all on `onTokenRefresh`.
- [ ] Backend payload must carry `account_id` (or account key) and `portal_id`. Without them you cannot route; push for that requirement.
- [ ] **Routing on tap** (`onMessageOpenedApp`, `getInitialMessage`, `flutter_local_notifications` `onDidReceiveNotificationResponse`): resolve account → if unknown (signed out on this device), drop or open the landing screen; if `needsReauth`, open re-auth for that account; if not active → switch account (§6) → switch portal (§7) → navigate. If active but different portal → switch portal → navigate.
- [ ] Show **account (and portal) context in the notification** itself when more than one account is signed in: e.g. title prefix "Acme · Jane" or Android notification channel groups / `groupKey` per account; iOS `threadIdentifier` per account+portal.
- [ ] **Badges**: per account and per portal unread counts in the switcher and picker; app icon badge = total across accounts (or active only — decide and be consistent).
- [ ] Foreground push for an **inactive** account/portal: show as a notification/in-app banner with that context, don't mutate the active session's state.
- [ ] Background handler (`FirebaseMessaging.onBackgroundMessage`) runs in a separate isolate with no session: it may only read the registry + secure store (hence `first_unlock` accessibility), and write into that account's DB by opening it briefly — or just display the notification.
- [ ] **Unregister** the token for an account on sign-out / removal (with that account's credentials, before deleting them). If revoke is impossible offline, backend should also drop tokens on refresh-token revocation.

## 11. Deep links per account / portal

See **flutter-deep-links**. Multi-account specifics:

- [ ] Links carry the **portal** in the path (`/p/<portalId>/items/<id>`) and optionally an account hint; resolution order: explicit account hint → which signed-in accounts are members of `portalId` (from cached portal lists) → active account.
- [ ] Exactly one candidate and it's active+right portal → navigate. Candidate is another account or portal → **confirm before switching** ("Open in Acme as jane@acme.com?") unless the user tapped a notification already labelled with that context. Several candidates → chooser. None → "This link belongs to a workspace you're not in" with "Add account" / "Request access".
- [ ] Never perform destructive actions from links; land on the screen and let the user act.
- [ ] Store a **pending link** through login/add-account and re-run resolution after the session is built.
- [ ] With `go_router`, do resolution in the top-level `redirect` (or a link handler before the router) — the session must be ready before portal-scoped routes build.

## 12. Background work

- [ ] Every job (`workmanager`, `background_fetch`, iOS BGTasks via plugins, isolates) carries `accountKey` and `portalId` in its input data; never "whatever is active when it runs".
- [ ] The job opens its own short-lived resources for that account (token read, DB, bare dio + auth) in its isolate — not the UI isolate's session. Drift: use the same file path; prefer WAL; or `DriftIsolate` / `drift`'s shared-isolate setup to avoid concurrent writers.
- [ ] Unique job names per account/portal (`sync.<accountHash>.<portalId>`); cancel them on sign-out and portal removal.
- [ ] Upload/outbox queues stored in the account DB with `portal_id`; a switch doesn't cancel them, it just stops the UI observing them. A sign-out does cancel (after warning "3 unsent items will be lost").
- [ ] Revoked account in background → stop job, mark `needsReauth`, no retries storm.

## 13. Purchases

See **flutter-in-app-purchase** / **flutter-revenuecat-iap**.

- [ ] Decide ownership: **store account (Apple ID / Google account)** is device-level; your **app account** owns entitlements server-side. Team plans belong to the **portal**.
- [ ] RevenueCat: `Purchases.logIn(appUserId)` on account switch (after the new session is built) and `logOut()` / `logIn(next)` on sign-out; listen to `customerInfo` per session. Don't call `logOut` when the user is anonymous-less — it errors if already anonymous.
- [ ] `in_app_purchase` direct: pass `applicationUserName` (an obfuscated account id) with purchases so the server can attach receipts to the right account; validate server-side and bind to account (and portal for team plans).
- [ ] Restore purchases while account B is active can transfer a store subscription bought under account A — define the policy (transfer / block / ask) and message it.
- [ ] Paywall and entitlement providers live in the session container — never show B a cached "Pro" from A.

## 14. Analytics and crash reporting

- [ ] On switch: `FirebaseAnalytics.setUserId(id: opaqueId)`, `FirebaseCrashlytics.setUserIdentifier`, `Sentry.configureScope((s) => s.setUser(SentryUser(id: opaqueId)))`; set **portal** as user property / tag (`portal_id`, `portal_role`). Reset (`setUserId(null)`, `resetAnalyticsData()` where appropriate) on sign-out.
- [ ] Opaque ids only — no email, name, or portal names (they can be PII or customer-confidential).
- [ ] Log `account_switch`, `portal_switch`, `account_added`, `account_removed`, `reauth_required` events with counts (number of accounts), not identities.
- [ ] Events emitted during teardown must be attributed to the old account — flush or tag before switching ids.

## 15. Security and privacy

- [ ] **App lock / biometrics per account** if required (`local_auth`): lock state belongs to the account; switching into a locked account prompts; switching away doesn't unlock others. Store the "requires lock" flag per account; tokens can additionally be gated with platform biometric access control (needs a plugin that supports it — verify `flutter_secure_storage` options for your version).
- [ ] **App-switcher snapshots**: sensitive accounts → obscure on `AppLifecycleState.inactive` (overlay) on iOS; on Android use `FLAG_SECURE` via a plugin/method channel (also blocks screenshots — product decision).
- [ ] **Switcher privacy**: the account list itself reveals emails; when app lock is on, require unlock before showing the switcher.
- [ ] **Account deletion** (store requirement for apps with account creation): server delete → full local wipe of that account (tokens, DB, files, caches, push registration, background jobs, prefs) → behave as sign-out.
- [ ] **Shared devices**: offer "remove account from this device" distinct from "delete account".
- [ ] Logs/crash breadcrumbs: no tokens, no URLs with tokens, no portal/user names.
- [ ] Clipboard, share sheets, notifications on lock screen: respect per-account privacy settings.

## 16. Testing plan

Unit-test the registry, token store, interceptor, and `SessionManager` with fakes (in-memory secure store, `MockAdapter`/`http_mock_adapter` for dio, in-memory Drift `NativeDatabase.memory()`). Widget-test the switcher and routing with a fake `SessionManager`. Integration-test (`integration_test`) the full switch on device for DB/file paths and secure storage.

### Scenario matrix

| Scenario | Expect |
|---|---|
| Switch account mid-request (slow GET started on A) | Response dropped; never rendered or written in B; no A token on any B request |
| Switch account while a POST is in flight | Either completes for A (attributed to A, written to A's DB) or is cancelled with a visible failure — never applied to B |
| Rapid switching A→B→A→C | Ends in C; no interleaved teardown; one session alive; no "DB closed" crashes |
| Token expiry on A while B active | A refreshes lazily when used (or in background job for A); B unaffected |
| 5 parallel 401s on A | Exactly one refresh call; all 5 retried once with new token |
| Refresh `invalid_grant` on inactive A | A marked `needsReauth`, tokens deleted, B untouched, switcher shows "Sign in again" |
| Refresh network error | Not revoked; tokens kept; offline UI |
| Push for inactive account, tapped (cold start) | App starts, builds that account's session, correct portal, target screen; back goes to portal home |
| Push for inactive portal of active account | Portal switch only (no account teardown), target screen |
| Push for removed account | Dropped or landing screen; no crash, no switch to a random account |
| Portal revoked while viewing it | Message, moved to next portal, local portal data deleted, no 403 loop |
| Role downgraded admin→member | Admin actions disappear after refresh/resume/event |
| Deep link to portal in account B while A active | Confirmation, then switch + navigate; cancel keeps A |
| Deep link before login | Stored; after login + portal selection, resolved and opened |
| Add account that is already signed in | No duplicate; tokens updated; switched to it |
| Add account, cancel | Back on the same screen of the same account, state intact |
| Max accounts reached | "Add account" explains limit before login |
| Sign out inactive account | Active session untouched; that account's files gone; push unregistered |
| Sign out active account with 2 others | Switched to most recent other account's last portal |
| Sign out last account | Login screen; `accounts/` empty; secure store empty; router stack fresh |
| Kill app mid-switch / mid-sign-out | Relaunch lands in target account / completes pending deletion |
| Reinstall on iOS | No resurrected sessions from Keychain |
| Android backup restore onto new device | Secure-store decrypt failure handled → re-auth, no crash |
| Images: same `/me/avatar` URL on A and B | Each account shows its own avatar after switch |
| Background sync scheduled for A, B active | Runs with A's credentials into A's DB |
| RevenueCat: B active after A purchased | B's entitlements shown (not A's); `logIn(B)` called |
| Analytics after switch | user id = B's opaque id; portal property updated |
| Text scale 2.0 in switcher / picker | No overflow; long emails ellipsized |
| Screen reader in switcher | Row reads name, email, state; active account announced |

Test tips:
- [ ] Assert isolation with a **leak test**: populate A, switch to B, walk every major screen and assert no A fixture strings appear (`find.textContaining('A-only-marker')` finds nothing).
- [ ] Assert teardown: after switch, old `ProviderContainer` is disposed (`container.dispose` sets a flag in a test `onDispose`), old DB `close()` called, old dio `CancelToken.isCancelled`.
- [ ] Use `Completer`-controlled fake responses to hold a request open across a switch.
- [ ] `FlutterSecureStorage.setMockInitialValues({})` for unit tests (verify it exists in your version) or inject a `TokenStore` interface.
- [ ] Integration tests on a real Android device with backup/restore (`adb shell bmgr`) and on iOS with reinstall.

## 17. Common bugs to look for in review

- One global `Dio` with an interceptor reading `currentAccount` at send/retry time → cross-account token attachment after switch.
- Refresh not single-flight → refresh-token rotation race; second refresh fails with `invalid_grant`; user logged out randomly.
- Refresh failure on one account calls a global `logout()` that wipes all accounts.
- Retry loop on 401 (refresh request itself goes through the auth interceptor).
- Top-level `final db = AppDatabase();` or `get_it` singleton DB → all accounts share one DB, or the old DB is used after switch.
- Riverpod root-level non-`autoDispose` providers caching user data; switching only updates `activeAccountProvider` → stale lists from A visible in B, old futures writing into new state.
- `ProviderScope` overrides changed without a new key / container → Riverpod doesn't support changing the override list shape; providers keep old values.
- Navigator stack kept across switch → Back returns to A's screens.
- `go_router` `redirect` reading account state not wired to `refreshListenable` → router doesn't react to sign-out/switch.
- `cached_network_image` global cache with auth'd images; `ImageCache` not cleared → A's avatar/attachments in B.
- Portal filter missing on one Drift query → cross-portal data leak.
- `X-Portal-Id` taken from a mutable global → request started in portal 1 sent with portal 2 header.
- 403 portal-revoked treated as auth failure → refresh storm / sign-out.
- Tokens in `shared_preferences` or Hive "because secure storage is slow".
- `flutter_secure_storage` options changed between releases without migration → everyone logged out.
- iOS Keychain not wiped on reinstall → ghost accounts.
- Push token registered only for the active account; or not unregistered on sign-out → notifications for a removed account.
- Push tap routed to the active account regardless of `account_id`.
- Background job uses "active account" at run time.
- Firebase Auth `instance` used for several accounts → signing in B signs out A.
- Add-account flow tears down A first; cancel leaves the user signed out.
- Duplicate entries for the same user (keyed by email or by registry index).
- Analytics user id not switched; email used as user id.
- RevenueCat `logIn` not called on switch → B sees A's entitlements.
- Sign-out deletes tokens before calling server revoke / push unregister (now impossible to authenticate those calls).
- Switch not serialized → two sessions alive, "database is locked" / "Bad state: StreamSink is closed".
- `setState`/`emit` after the session was disposed (no `ref.mounted` / `isClosed` / `mounted` check).

## 18. Output format

**When reviewing** an existing app, respond with:

1. **Context summary** — account/portal model, backends/regions, auth contract (header vs portal token), DB engine and layout, state management + router, packages + versions.
2. **Isolation map** — table of every user-scoped resource (dio, DB, caches, providers/blocs, files, prefs, sockets, background jobs, push, analytics, IAP) and whether it is global, per account, or per portal today vs what it should be.
3. **Findings** grouped by area (Model, Storage, Tokens, Isolation, Account switch, Portal switch, Add, Sign out, Push, Links, Background, Purchases, Analytics, Security), each with **severity** (Critical = cross-account/portal data or token leak; High; Medium; Low), file:line, symptom, and fix (snippet when useful).
4. **Missing scenarios** from the §16 matrix, with tests to add.
5. **Version notes** — APIs to verify against the project's package versions.

**When implementing**, deliver:

1. `AccountRegistry` + `PortalRef` model + persistence with schema version, and `SecureTokenStore`.
2. `TokenHolder` + dio `AuthInterceptor` (single-flight per account) + `PortalInterceptor`.
3. `AccountSession` / `PortalSession` owning dio, DB, caches; `SessionManager` with serialized `switchAccount`, `switchPortal`, `addAccount`, `signOut(one|all)`.
4. DI wiring (Riverpod container per session, or get_it scopes + Bloc) and router reset.
5. Switcher + portal picker widgets; push and link routing to (account, portal).
6. Tests for the §16 essentials (switch mid-request, single-flight refresh, revoked account, push for inactive account/portal, portal revoked, sign out last account).
7. Assumptions (limits, restore allow-list, badge policy) and package versions used, flagging anything to verify.

## Appendix — Reference implementation (Dart 3, Flutter 3.24+, Riverpod 2.6/3.x, dio 5, Drift 2)

Simplified but complete enough to adapt. Error handling and logging trimmed; names are illustrative. Verify package APIs against your versions.

### A.1 Model and registry

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'package:crypto/crypto.dart';
import 'package:path_provider/path_provider.dart';

/// "$environment:$userId" — stable, never the email.
extension type const AccountKey(String value) {
  /// Safe folder / cache name.
  String get hash => sha256.convert(utf8.encode(value)).toString().substring(0, 16);
}

enum PortalRole { owner, admin, member, guest }

final class PortalRef {
  const PortalRef({required this.id, required this.name, required this.role, this.lastUsedAt});
  final String id;
  final String name;
  final PortalRole role;
  final DateTime? lastUsedAt;

  PortalRef copyWith({DateTime? lastUsedAt, PortalRole? role, String? name}) => PortalRef(
      id: id, name: name ?? this.name, role: role ?? this.role, lastUsedAt: lastUsedAt ?? this.lastUsedAt);

  Map<String, Object?> toJson() =>
      {'id': id, 'name': name, 'role': role.name, 'lastUsedAt': lastUsedAt?.toIso8601String()};
  factory PortalRef.fromJson(Map<String, Object?> j) => PortalRef(
        id: j['id']! as String,
        name: j['name']! as String,
        role: PortalRole.values.byName(j['role']! as String),
        lastUsedAt: j['lastUsedAt'] == null ? null : DateTime.parse(j['lastUsedAt']! as String),
      );
}

final class Account {
  const Account({
    required this.key,
    required this.userId,
    required this.environment,
    required this.baseUrl,
    required this.email,
    required this.displayName,
    this.avatarUrl,
    this.portals = const [],
    this.lastPortalId,
    this.lastLocation,
    this.needsReauth = false,
    this.pendingDeletion = false,
    required this.lastActiveAt,
  });

  final AccountKey key;
  final String userId, environment, baseUrl, email, displayName;
  final String? avatarUrl, lastPortalId, lastLocation;
  final List<PortalRef> portals;
  final bool needsReauth, pendingDeletion;
  final DateTime lastActiveAt;

  PortalRef? portal(String? id) => portals.where((p) => p.id == id).firstOrNull;

  /// Last used → most recent → first → null (zero portals).
  PortalRef? get defaultPortal {
    final last = portal(lastPortalId);
    if (last != null) return last;
    final sorted = [...portals]
      ..sort((a, b) => (b.lastUsedAt ?? DateTime(0)).compareTo(a.lastUsedAt ?? DateTime(0)));
    return sorted.firstOrNull;
  }

  Account copyWith({
    List<PortalRef>? portals,
    String? lastPortalId,
    String? lastLocation,
    bool? needsReauth,
    bool? pendingDeletion,
    DateTime? lastActiveAt,
  }) =>
      Account(
        key: key, userId: userId, environment: environment, baseUrl: baseUrl,
        email: email, displayName: displayName, avatarUrl: avatarUrl,
        portals: portals ?? this.portals,
        lastPortalId: lastPortalId ?? this.lastPortalId,
        lastLocation: lastLocation ?? this.lastLocation,
        needsReauth: needsReauth ?? this.needsReauth,
        pendingDeletion: pendingDeletion ?? this.pendingDeletion,
        lastActiveAt: lastActiveAt ?? this.lastActiveAt,
      );

  Map<String, Object?> toJson() => {
        'key': key.value, 'userId': userId, 'environment': environment, 'baseUrl': baseUrl,
        'email': email, 'displayName': displayName, 'avatarUrl': avatarUrl,
        'portals': portals.map((p) => p.toJson()).toList(),
        'lastPortalId': lastPortalId, 'lastLocation': lastLocation,
        'needsReauth': needsReauth, 'pendingDeletion': pendingDeletion,
        'lastActiveAt': lastActiveAt.toIso8601String(),
      };

  factory Account.fromJson(Map<String, Object?> j) => Account(
        key: AccountKey(j['key']! as String),
        userId: j['userId']! as String,
        environment: j['environment']! as String,
        baseUrl: j['baseUrl']! as String,
        email: j['email']! as String,
        displayName: j['displayName']! as String,
        avatarUrl: j['avatarUrl'] as String?,
        portals: [for (final p in j['portals']! as List) PortalRef.fromJson((p as Map).cast())],
        lastPortalId: j['lastPortalId'] as String?,
        lastLocation: j['lastLocation'] as String?,
        needsReauth: j['needsReauth'] as bool? ?? false,
        pendingDeletion: j['pendingDeletion'] as bool? ?? false,
        lastActiveAt: DateTime.parse(j['lastActiveAt']! as String),
      );
}

typedef ActivePair = ({AccountKey account, String? portalId});

class TooManyAccounts implements Exception {}

/// Single writer for non-secret account metadata. Process-global.
class AccountRegistry {
  AccountRegistry._(this._file, this._accounts, this._active, this.installId);

  static const schemaVersion = 2;
  static const maxAccounts = 5;

  final File _file;
  final List<Account> _accounts; // ordered
  ActivePair? _active;
  final String installId;
  final _changes = StreamController<void>.broadcast();
  Future<void> _writeChain = Future.value();

  Stream<void> get changes => _changes.stream;
  List<Account> get accounts => List.unmodifiable(_accounts.where((a) => !a.pendingDeletion));
  ActivePair? get active => _active;
  Account? byKey(AccountKey k) => _accounts.where((a) => a.key == k).firstOrNull;
  bool get isFreshInstall => _accounts.isEmpty && _active == null;

  static Future<AccountRegistry> load() async {
    final dir = await getApplicationSupportDirectory();
    final file = File('${dir.path}/account_registry.json');
    if (!await file.exists()) {
      return AccountRegistry._(file, [], null, DateTime.now().microsecondsSinceEpoch.toString());
    }
    final j = jsonDecode(await file.readAsString()) as Map<String, Object?>;
    // Migrations keyed on j['v'] go here (e.g. v1 single-account → v2).
    final accounts = [for (final a in j['accounts']! as List) Account.fromJson((a as Map).cast())];
    final act = j['active'] as Map?;
    return AccountRegistry._(
      file,
      accounts,
      act == null ? null : (account: AccountKey(act['account'] as String), portalId: act['portal'] as String?),
      j['installId']! as String,
    );
  }

  Future<void> _persist() {
    final payload = jsonEncode({
      'v': schemaVersion,
      'installId': installId,
      'accounts': _accounts.map((a) => a.toJson()).toList(),
      'active': _active == null ? null : {'account': _active!.account.value, 'portal': _active!.portalId},
    });
    // Serialize writes; write-then-rename for atomicity.
    return _writeChain = _writeChain.then((_) async {
      final tmp = File('${_file.path}.tmp');
      await tmp.writeAsString(payload, flush: true);
      await tmp.rename(_file.path);
      _changes.add(null);
    });
  }

  Future<void> upsert(Account a) {
    final i = _accounts.indexWhere((x) => x.key == a.key);
    if (i >= 0) {
      _accounts[i] = a;
    } else {
      if (accounts.length >= maxAccounts) throw TooManyAccounts();
      _accounts.add(a);
    }
    return _persist();
  }

  Future<void> update(AccountKey k, Account Function(Account) f) {
    final i = _accounts.indexWhere((x) => x.key == k);
    if (i < 0) return Future.value();
    _accounts[i] = f(_accounts[i]);
    return _persist();
  }

  Future<void> setActive(ActivePair? pair) {
    _active = pair;
    if (pair != null) {
      update(pair.account, (a) => a.copyWith(
            lastActiveAt: DateTime.now(),
            lastPortalId: pair.portalId,
            portals: [
              for (final p in a.portals) p.id == pair.portalId ? p.copyWith(lastUsedAt: DateTime.now()) : p
            ],
          ));
    }
    return _persist();
  }

  Future<void> remove(AccountKey k) {
    _accounts.removeWhere((a) => a.key == k);
    if (_active?.account == k) _active = null;
    return _persist();
  }

  /// Most recently used remaining account that can be opened without re-auth.
  Account? nextActiveCandidate({AccountKey? excluding}) {
    final c = accounts.where((a) => a.key != excluding && !a.needsReauth).toList()
      ..sort((a, b) => b.lastActiveAt.compareTo(a.lastActiveAt));
    return c.firstOrNull;
  }
}
```

### A.2 Secure token store

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final class Tokens {
  const Tokens({required this.access, required this.refresh, required this.expiresAt, this.portalTokens = const {}});
  final String access, refresh;
  final DateTime expiresAt;
  final Map<String, String> portalTokens; // portalId → token, if the backend uses token exchange

  bool expiresWithin(Duration d) => DateTime.now().add(d).isAfter(expiresAt);

  Map<String, Object?> toJson() =>
      {'a': access, 'r': refresh, 'e': expiresAt.toIso8601String(), 'p': portalTokens};
  factory Tokens.fromJson(Map<String, Object?> j) => Tokens(
        access: j['a']! as String,
        refresh: j['r']! as String,
        expiresAt: DateTime.parse(j['e']! as String),
        portalTokens: ((j['p'] as Map?) ?? const {}).cast<String, String>(),
      );
}

abstract interface class TokenStore {
  Future<Tokens?> read(AccountKey k);
  Future<void> write(AccountKey k, Tokens t);
  Future<void> delete(AccountKey k);
  Future<void> deleteAll();
  Future<Set<AccountKey>> keys();
}

class SecureTokenStore implements TokenStore {
  SecureTokenStore()
      : _s = const FlutterSecureStorage(
          // Readable in background after first unlock; never migrates to a new device.
          iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device),
          // VERIFY for your plugin version: v9 projects typically set
          // AndroidOptions(encryptedSharedPreferences: true); v10 migrates away from it.
          // Never change this on an installed base without a read-old/write-new migration.
          aOptions: AndroidOptions(),
        );

  final FlutterSecureStorage _s;
  static const _prefix = 'tokens.';
  final _locks = <AccountKey, Future<void>>{};

  /// Per-account serialization: refresh-write and sign-out-delete never interleave.
  Future<T> _locked<T>(AccountKey k, Future<T> Function() body) {
    final prev = _locks[k] ?? Future.value();
    final next = prev.then((_) => body());
    _locks[k] = next.then((_) {}, onError: (_) {});
    return next;
  }

  @override
  Future<Tokens?> read(AccountKey k) => _locked(k, () async {
        try {
          final raw = await _s.read(key: '$_prefix${k.value}');
          return raw == null ? null : Tokens.fromJson(jsonDecode(raw) as Map<String, Object?>);
        } on Exception {
          // Typical after Android backup restore: ciphertext without the Keystore key.
          await _s.delete(key: '$_prefix${k.value}');
          return null; // caller marks needsReauth
        }
      });

  @override
  Future<void> write(AccountKey k, Tokens t) =>
      _locked(k, () => _s.write(key: '$_prefix${k.value}', value: jsonEncode(t.toJson())));

  @override
  Future<void> delete(AccountKey k) => _locked(k, () => _s.delete(key: '$_prefix${k.value}'));

  @override
  Future<void> deleteAll() async {
    for (final k in await keys()) {
      await delete(k);
    }
  }

  @override
  Future<Set<AccountKey>> keys() async => {
        for (final k in (await _s.readAll()).keys)
          if (k.startsWith(_prefix)) AccountKey(k.substring(_prefix.length)),
      };
}

/// Run once at startup before opening any session.
Future<void> reconcileOnLaunch(AccountRegistry reg, TokenStore store) async {
  // iOS Keychain survives uninstall: a fresh install must not resurrect sessions.
  if (reg.isFreshInstall) {
    await store.deleteAll();
    return;
  }
  // Finish deletions interrupted by process death.
  for (final a in [...reg.accounts.where((a) => a.pendingDeletion)]) {
    await wipeAccountFiles(a.key);
    await store.delete(a.key);
    await reg.remove(a.key);
  }
  // Orphan tokens (crash between token write and registry write).
  final known = reg.accounts.map((a) => a.key).toSet();
  for (final k in await store.keys()) {
    if (!known.contains(k)) await store.delete(k);
  }
}

Future<Directory> accountDir(AccountKey k) async {
  final base = await getApplicationSupportDirectory();
  return Directory('${base.path}/accounts/${k.hash}').create(recursive: true);
}

Future<void> wipeAccountFiles(AccountKey k) async {
  final base = await getApplicationSupportDirectory();
  final tmp = await getTemporaryDirectory();
  for (final d in [Directory('${base.path}/accounts/${k.hash}'), Directory('${tmp.path}/accounts/${k.hash}')]) {
    if (await d.exists()) await d.delete(recursive: true);
  }
}
```

### A.3 Token holder + dio interceptors (single-flight refresh per account)

```dart
import 'package:dio/dio.dart';

class AccountRevoked implements Exception {
  AccountRevoked(this.account);
  final AccountKey account;
}

/// One per account session. Owns the in-memory tokens and the single in-flight refresh.
class TokenHolder {
  TokenHolder({required this.account, required this.store, required this.baseUrl, required this.onRevoked});

  final AccountKey account;
  final TokenStore store;
  final String baseUrl;
  final void Function(AccountKey) onRevoked;
  Tokens? _tokens;
  Future<Tokens>? _inFlight;

  // Bare client: never passes through AuthInterceptor (no refresh loops).
  late final Dio _bare = Dio(BaseOptions(baseUrl: baseUrl, connectTimeout: const Duration(seconds: 10)));

  Future<Tokens?> current() async => _tokens ??= await store.read(account);

  /// Concurrent callers share one refresh. [stale] = token the caller saw fail;
  /// if it has already been replaced, return the new one without refreshing again.
  Future<Tokens> refresh({String? stale}) {
    if (_tokens != null && stale != null && _tokens!.access != stale) return Future.value(_tokens!);
    return _inFlight ??= _doRefresh().whenComplete(() => _inFlight = null);
  }

  Future<Tokens> _doRefresh() async {
    final t = await current();
    if (t == null) throw AccountRevoked(account);
    try {
      final r = await _bare.post<Map<String, Object?>>('/oauth/token',
          data: {'grant_type': 'refresh_token', 'refresh_token': t.refresh});
      final d = r.data!;
      final next = Tokens(
        access: d['access_token']! as String,
        refresh: (d['refresh_token'] as String?) ?? t.refresh, // rotation-aware
        expiresAt: DateTime.now().add(Duration(seconds: d['expires_in']! as int)),
        // Account token changed → portal tokens are invalid; re-exchange lazily.
      );
      await store.write(account, next); // persist BEFORE releasing waiters
      return _tokens = next;
    } on DioException catch (e) {
      final code = e.response?.statusCode;
      final err = (e.response?.data is Map) ? (e.response!.data as Map)['error'] : null;
      if (code == 401 || err == 'invalid_grant') {
        _tokens = null;
        await store.delete(account);
        onRevoked(account); // marks needsReauth for THIS account only
        throw AccountRevoked(account);
      }
      rethrow; // network/5xx: not revoked
    }
  }
}

/// Bound to ONE account at construction. Never reads "active account".
class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor(this._holder, this._dio);
  final TokenHolder _holder;
  final Dio _dio; // the same session dio, for retries

  static const _skew = Duration(seconds: 45);

  @override
  Future<void> onRequest(RequestOptions o, RequestInterceptorHandler h) async {
    try {
      var t = await _holder.current();
      if (t == null) return h.reject(DioException(requestOptions: o, error: AccountRevoked(_holder.account)));
      if (t.expiresWithin(_skew)) t = await _holder.refresh(stale: t.access);
      o.headers['Authorization'] = 'Bearer ${t.access}';
      h.next(o);
    } on AccountRevoked catch (e) {
      h.reject(DioException(requestOptions: o, error: e));
    } catch (e) {
      h.reject(DioException(requestOptions: o, error: e));
    }
  }

  @override
  Future<void> onError(DioException e, ErrorInterceptorHandler h) async {
    final o = e.requestOptions;
    if (e.response?.statusCode != 401 || o.extra['retried'] == true) return h.next(e);
    final sent = (o.headers['Authorization'] as String?)?.replaceFirst('Bearer ', '');
    try {
      final t = await _holder.refresh(stale: sent);
      o.extra['retried'] = true;
      o.headers['Authorization'] = 'Bearer ${t.access}';
      h.resolve(await _dio.fetch<dynamic>(o));
    } on DioException catch (re) {
      h.next(re);
    } catch (err) {
      h.next(DioException(requestOptions: o, error: err));
    }
  }
}

class PortalRevoked implements Exception {
  PortalRevoked(this.portalId);
  final String portalId;
}

/// Adds the portal header captured at construction; maps membership 403s.
class PortalInterceptor extends Interceptor {
  PortalInterceptor(this.portalId, this.onPortalRevoked);
  final String portalId;
  final void Function(String portalId) onPortalRevoked;

  @override
  void onRequest(RequestOptions o, RequestInterceptorHandler h) {
    o.headers['X-Portal-Id'] = portalId;
    h.next(o);
  }

  @override
  void onError(DioException e, ErrorInterceptorHandler h) {
    final data = e.response?.data;
    if (e.response?.statusCode == 403 && data is Map && data['code'] == 'portal_access_revoked') {
      onPortalRevoked(portalId);
      return h.next(DioException(requestOptions: e.requestOptions, response: e.response, error: PortalRevoked(portalId)));
    }
    h.next(e);
  }
}
```

### A.4 Sessions (account + portal) with Drift DB per account

```dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:flutter/painting.dart';
import 'package:flutter_cache_manager/flutter_cache_manager.dart';

part 'account_db.g.dart';

class Items extends Table {
  TextColumn get id => text()();
  TextColumn get portalId => text()(); // every portal-scoped table has it
  TextColumn get title => text()();
  @override
  Set<Column> get primaryKey => {portalId, id};
}

@DriftDatabase(tables: [Items])
class AccountDatabase extends _$AccountDatabase {
  AccountDatabase(File file) : super(NativeDatabase.createInBackground(file));
  AccountDatabase.forTesting(super.e);
  @override
  int get schemaVersion => 1;
}

/// Portal filter baked into the DAO: callers cannot forget it.
class ItemsDao {
  ItemsDao(this._db, this.portalId);
  final AccountDatabase _db;
  final String portalId;

  Stream<List<Item>> watchAll() =>
      (_db.select(_db.items)..where((t) => t.portalId.equals(portalId))).watch();

  Future<void> upsertAll(List<ItemsCompanion> rows) => _db.batch((b) => b.insertAllOnConflictUpdate(
      _db.items, [for (final r in rows) r.copyWith(portalId: Value(portalId))]));

  Future<void> deletePortalData() => (_db.delete(_db.items)..where((t) => t.portalId.equals(portalId))).go();
}

class AccountSession {
  AccountSession._(this.account, this.tokens, this.dio, this.db, this.images, this._cancel);

  final Account account;
  final TokenHolder tokens;
  final Dio dio; // account-level endpoints (profile, portal list) — no portal header
  final AccountDatabase db;
  final CacheManager images; // per-account authenticated image cache
  final CancelToken _cancel;
  bool _disposed = false;
  bool get isDisposed => _disposed;
  CancelToken get cancelToken => _cancel;

  static Future<AccountSession> open(Account a, TokenStore store, {required void Function(AccountKey) onRevoked}) async {
    final dir = await accountDir(a.key);
    final holder = TokenHolder(account: a.key, store: store, baseUrl: a.baseUrl, onRevoked: onRevoked);
    if (await holder.current() == null) throw AccountRevoked(a.key);
    final dio = Dio(BaseOptions(baseUrl: a.baseUrl));
    dio.interceptors.add(AuthInterceptor(holder, dio));
    final db = AccountDatabase(File('${dir.path}/main.sqlite'));
    final images = CacheManager(Config('img_${a.key.hash}', stalePeriod: const Duration(days: 14)));
    return AccountSession._(a, holder, dio, db, images, CancelToken());
  }

  /// A dio for one portal: shares the account's token holder (single-flight stays per account).
  Dio portalDio(String portalId, void Function(String) onPortalRevoked) {
    final d = Dio(BaseOptions(baseUrl: account.baseUrl));
    d.interceptors
      ..add(PortalInterceptor(portalId, onPortalRevoked))
      ..add(AuthInterceptor(tokens, d));
    return d;
  }

  Future<void> dispose() async {
    if (_disposed) return;
    _disposed = true;
    _cancel.cancel('account session disposed');
    dio.close(force: true);
    await db.close();
    PaintingBinding.instance.imageCache
      ..clear()
      ..clearLiveImages();
  }
}

class PortalSession {
  PortalSession(this.account, this.portal, void Function(String) onRevoked)
      : dio = account.portalDio(portal.id, onRevoked),
        items = ItemsDao(account.db, portal.id);

  final AccountSession account;
  final PortalRef portal;
  final Dio dio;
  final ItemsDao items;
  final cancelToken = CancelToken();
  bool _disposed = false;
  bool get isDisposed => _disposed;
  String get key => '${account.account.key.value}/${portal.id}';

  Future<void> dispose() async {
    if (_disposed) return;
    _disposed = true;
    cancelToken.cancel('portal session disposed');
    dio.close(force: true);
    // DB stays open: it belongs to the account session.
  }
}
```

### A.5 Riverpod wiring: container per account, child container per portal

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

// App-global (root container).
final registryProvider = Provider<AccountRegistry>((_) => throw UnimplementedError());
final tokenStoreProvider = Provider<TokenStore>((_) => throw UnimplementedError());

// Session-scoped: ONLY ever provided via overrides in a session container.
final accountSessionProvider = Provider<AccountSession>((_) => throw StateError('no account session'));
final portalSessionProvider = Provider<PortalSession>((_) => throw StateError('no portal session'));

// Example portal-scoped data provider. It lives in the portal container and dies with it.
final itemsProvider = StreamProvider.autoDispose<List<Item>>((ref) {
  return ref.watch(portalSessionProvider).items.watchAll();
});

final portalListProvider = FutureProvider<List<PortalRef>>((ref) async {
  final s = ref.watch(accountSessionProvider);
  final r = await s.dio.get<List<dynamic>>('/me/portals', cancelToken: s.cancelToken);
  return [for (final p in r.data!) PortalRef.fromJson((p as Map).cast())];
});
```

Session-scoped providers that need to resolve inside a child container must be registered there via overrides (both `accountSessionProvider` and anything depending on it that should be per container). In Riverpod 2.x, a provider not overridden in a child resolves in the **root** where its dependencies throw — declare `dependencies: [portalSessionProvider]` on scoped providers (required for `riverpod_lint` and for correct scoping), or simply override them per container as below. Verify the scoping rules for your Riverpod major version.

### A.6 SessionManager: serialized switch-account / switch-portal / add / sign out

```dart
sealed class SessionState {
  const SessionState();
}
final class SignedOut extends SessionState {
  const SignedOut();
}
final class Switching extends SessionState {
  const Switching(this.target);
  final Account target;
}
final class NoPortal extends SessionState {
  const NoPortal(this.account, this.container);
  final AccountSession account;
  final ProviderContainer container;
}
final class Ready extends SessionState {
  const Ready(this.account, this.portal, this.accountContainer, this.portalContainer, this.router);
  final AccountSession account;
  final PortalSession portal;
  final ProviderContainer accountContainer, portalContainer;
  final GoRouter router;
}

class SessionManager extends ChangeNotifier {
  SessionManager(this.root, this.registry, this.store, {required this.routerFactory, required this.hooks});

  final ProviderContainer root;
  final AccountRegistry registry;
  final TokenStore store;
  final GoRouter Function(PortalSession s, String initialLocation) routerFactory;
  final SessionHooks hooks; // analytics / IAP / push side effects

  SessionState _state = const SignedOut();
  SessionState get state => _state;
  void _set(SessionState s) {
    _state = s;
    notifyListeners();
  }

  // ---- Serialization: every mutation runs strictly one after another. ----
  Future<void> _queue = Future.value();
  Future<T> _serial<T>(Future<T> Function() body) {
    final run = _queue.then((_) => body());
    _queue = run.then((_) {}, onError: (_) {});
    return run;
  }

  AccountSession? _account;
  ProviderContainer? _accountContainer;
  PortalSession? _portal;
  ProviderContainer? _portalContainer;

  Future<void> restore() => _serial(() async {
        await reconcileOnLaunch(registry, store);
        final a = registry.active == null ? null : registry.byKey(registry.active!.account);
        final target = (a != null && !a.needsReauth) ? a : registry.nextActiveCandidate();
        if (target == null) return _set(const SignedOut());
        await _openAccount(target, portalId: registry.active?.portalId, location: target.lastLocation);
      });

  Future<void> switchAccount(AccountKey key, {String? portalId, String? location}) => _serial(() async {
        if (_account?.account.key == key) {
          if (portalId != null && portalId != _portal?.portal.id) await _openPortal(portalId, location: location);
          return;
        }
        final target = registry.byKey(key);
        if (target == null) throw StateError('unknown account');
        _set(Switching(target)); // UI shows switching overlay; old screens not interactive
        await _saveLocation();
        await _teardownAccount();
        await _openAccount(target, portalId: portalId, location: location);
      });

  Future<void> switchPortal(String portalId, {String? location}) => _serial(() => _openPortal(portalId, location: location));

  /// Called after a successful login flow that ran ABOVE the current session.
  Future<void> addAccount(Account a, Tokens t) => _serial(() async {
        final existing = registry.byKey(a.key);
        await store.write(a.key, t);
        await registry.upsert(existing == null ? a : existing.copyWith(needsReauth: false));
        await _saveLocation();
        await _teardownAccount();
        await _openAccount(registry.byKey(a.key)!);
      });

  Future<void> signOut(AccountKey key) => _serial(() => _signOutOne(key));

  Future<void> signOutAll() => _serial(() async {
        for (final a in [...registry.accounts]) {
          await _signOutOne(a.key, pickNext: false);
        }
        await store.deleteAll();
        _set(const SignedOut());
      });

  // ---- internals ----

  Future<void> _openAccount(Account a, {String? portalId, String? location}) async {
    final AccountSession s;
    try {
      s = await AccountSession.open(a, store, onRevoked: _onAccountRevoked);
    } on AccountRevoked {
      await registry.update(a.key, (x) => x.copyWith(needsReauth: true));
      return _set(const SignedOut()); // UI offers re-auth for `a` or another account
    }
    _account = s;
    _accountContainer = ProviderContainer(
      parent: root,
      overrides: [accountSessionProvider.overrideWithValue(s), portalListProvider],
    );
    await hooks.onAccountActivated(s.account); // analytics id, RevenueCat logIn, etc.

    // Refresh portals in the background; use cached list for instant start.
    unawaited(_refreshPortals(s));
    final portal = a.portal(portalId) ?? a.defaultPortal;
    if (portal == null) {
      await registry.setActive((account: a.key, portalId: null));
      return _set(NoPortal(s, _accountContainer!));
    }
    await _openPortal(portal.id, location: location);
  }

  Future<void> _openPortal(String portalId, {String? location}) async {
    final acc = _account!;
    final ref = registry.byKey(acc.account.key)!.portal(portalId);
    if (ref == null) throw StateError('portal not in account');
    await _teardownPortal();
    final p = PortalSession(acc, ref, _onPortalRevoked);
    _portal = p;
    _portalContainer = ProviderContainer(
      parent: _accountContainer,
      overrides: [portalSessionProvider.overrideWithValue(p), itemsProvider],
    );
    await registry.setActive((account: acc.account.key, portalId: portalId)); // persist before UI
    await hooks.onPortalActivated(acc.account, ref);
    final router = routerFactory(p, _safeLocation(location) ?? '/home');
    _set(Ready(acc, p, _accountContainer!, _portalContainer!, router));
  }

  Future<void> _teardownPortal() async {
    final c = _portalContainer, p = _portal;
    _portalContainer = null;
    _portal = null;
    if (_state case Ready(:final router)) router.dispose();
    c?.dispose(); // runs ref.onDispose for portal providers
    await p?.dispose();
  }

  Future<void> _teardownAccount() async {
    await _teardownPortal();
    final c = _accountContainer, s = _account;
    _accountContainer = null;
    _account = null;
    c?.dispose();
    await s?.dispose();
  }

  Future<void> _refreshPortals(AccountSession s) async {
    try {
      final r = await s.dio.get<List<dynamic>>('/me/portals', cancelToken: s.cancelToken);
      if (s.isDisposed) return; // switched away meanwhile — drop
      final fresh = [for (final p in r.data!) PortalRef.fromJson((p as Map).cast())];
      final old = registry.byKey(s.account.key)?.portals ?? const <PortalRef>[];
      final merged = [
        for (final p in fresh) p.copyWith(lastUsedAt: old.where((o) => o.id == p.id).firstOrNull?.lastUsedAt)
      ];
      await registry.update(s.account.key, (a) => a.copyWith(portals: merged));
      final current = _portal;
      if (current != null && !merged.any((p) => p.id == current.portal.id)) _onPortalRevoked(current.portal.id);
    } on DioException {
      // offline: keep cached list
    }
  }

  void _onAccountRevoked(AccountKey k) {
    registry.update(k, (a) => a.copyWith(needsReauth: true));
    if (_account?.account.key == k) hooks.showReauth(k); // other accounts untouched
  }

  void _onPortalRevoked(String portalId) => _serial(() async {
        final acc = _account;
        if (acc == null || _portal?.portal.id != portalId) return;
        await ItemsDao(acc.db, portalId).deletePortalData();
        await registry.update(acc.account.key, (a) => a.copyWith(
              portals: a.portals.where((p) => p.id != portalId).toList(),
              lastPortalId: a.lastPortalId == portalId ? null : a.lastPortalId,
            ));
        hooks.toast('You no longer have access to this workspace');
        final next = registry.byKey(acc.account.key)!.defaultPortal;
        if (next == null) {
          await _teardownPortal();
          await registry.setActive((account: acc.account.key, portalId: null));
          _set(NoPortal(acc, _accountContainer!));
        } else {
          await _openPortal(next.id);
        }
      });

  Future<void> _signOutOne(AccountKey key, {bool pickNext = true}) async {
    final a = registry.byKey(key);
    if (a == null) return;
    await registry.update(key, (x) => x.copyWith(pendingDeletion: true)); // survives process death
    // Use THIS account's credentials for revoke + push unregister, before deleting them.
    final isActive = _account?.account.key == key;
    final s = isActive ? _account! : await _tryOpenHeadless(a);
    if (s != null) {
      await hooks.beforeSignOut(s); // push unregister, server revoke (short timeout, best effort)
    }
    if (isActive) {
      await hooks.onAccountDeactivated(a); // RevenueCat logOut / analytics reset
      await _teardownAccount();
    } else {
      await s?.dispose();
    }
    await store.delete(key);
    await wipeAccountFiles(key);
    await registry.remove(key);
    if (isActive && pickNext) {
      final next = registry.nextActiveCandidate();
      if (next == null) return _set(const SignedOut());
      await _openAccount(next);
    }
  }

  Future<AccountSession?> _tryOpenHeadless(Account a) async {
    try {
      return await AccountSession.open(a, store, onRevoked: (_) {});
    } catch (_) {
      return null;
    }
  }

  Future<void> _saveLocation() async {
    if (_state case Ready(:final router, :final account)) {
      final loc = router.routerDelegate.currentConfiguration.uri.toString();
      await registry.update(account.account.key, (a) => a.copyWith(lastLocation: _safeLocation(loc)));
    }
  }

  static const _restorable = ['/home', '/items', '/inbox', '/settings'];
  String? _safeLocation(String? loc) =>
      (loc != null && _restorable.any((p) => loc == p || loc.startsWith('$p/'))) ? loc : null;
}

abstract interface class SessionHooks {
  Future<void> onAccountActivated(Account a);
  Future<void> onAccountDeactivated(Account a);
  Future<void> onPortalActivated(Account a, PortalRef p);
  Future<void> beforeSignOut(AccountSession s);
  void showReauth(AccountKey k);
  void toast(String message);
}
```

### A.7 Mounting: new key + new router per session

```dart
class SessionHost extends StatelessWidget {
  const SessionHost({super.key, required this.manager});
  final SessionManager manager;

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: manager,
      builder: (context, _) => switch (manager.state) {
        SignedOut() => const MaterialApp(home: LoginScreen()),
        Switching(:final target) => MaterialApp(home: SwitchingOverlay(account: target)),
        NoPortal(:final container, :final account) => UncontrolledProviderScope(
            key: ValueKey('acct:${account.account.key.value}'),
            container: container,
            child: const MaterialApp(home: CreateOrJoinPortalScreen()),
          ),
        // A new key per (account, portal) discards every widget, State, and Navigator
        // from the previous session. The router is rebuilt so Back can't reach old screens.
        Ready(:final portal, :final portalContainer, :final router) => UncontrolledProviderScope(
            key: ValueKey('portal:${portal.key}'),
            container: portalContainer,
            child: MaterialApp.router(routerConfig: router),
          ),
      },
    );
  }
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final registry = await AccountRegistry.load();
  final store = SecureTokenStore();
  final root = ProviderContainer(overrides: [
    registryProvider.overrideWithValue(registry),
    tokenStoreProvider.overrideWithValue(store),
  ]);
  final manager = SessionManager(root, registry, store,
      routerFactory: (s, initial) => buildPortalRouter(initial), hooks: AppSessionHooks());
  await manager.restore();
  runApp(SessionHost(manager: manager));
}

GoRouter buildPortalRouter(String initialLocation) => GoRouter(
      initialLocation: initialLocation,
      routes: [
        GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
        GoRoute(path: '/items/:id', builder: (_, s) => ItemScreen(id: s.pathParameters['id']!)),
      ],
    );
```

If the app must keep a single `MaterialApp` (e.g. for a global theme or a persistent overlay), keep it above `SessionHost` and switch only the `Router`/`Navigator` subtree under the keyed `UncontrolledProviderScope`; call `router.go(initialLocation)` is **not** enough when the router lives above the session, because existing pages keep their old `State`.

### A.8 Account switcher and portal picker

```dart
class AccountSwitcherSheet extends ConsumerWidget {
  const AccountSwitcherSheet({super.key, required this.manager});
  final SessionManager manager;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final reg = ref.watch(registryProvider);
    final active = reg.active?.account;
    final accounts = [...reg.accounts]..sort((a, b) => a.key == active ? -1 : b.key == active ? 1 : 0);
    return SafeArea(
      child: Column(mainAxisSize: MainAxisSize.min, children: [
        for (final a in accounts)
          Semantics(
            label: a.key == active
                ? 'Current account, ${a.displayName}, ${a.email}'
                : a.needsReauth
                    ? '${a.displayName}, ${a.email}, sign in required'
                    : 'Switch to ${a.displayName}, ${a.email}',
            excludeSemantics: true,
            button: true,
            child: ListTile(
              leading: CircleAvatar(child: Text(a.displayName.characters.first)),
              title: Text(a.displayName, overflow: TextOverflow.ellipsis),
              subtitle: Text(a.needsReauth ? 'Sign in again' : a.email, overflow: TextOverflow.ellipsis),
              trailing: a.key == active ? const Icon(Icons.check) : null,
              onTap: () async {
                Navigator.pop(context);
                if (a.key == active) return;
                if (a.needsReauth) return showReauthFor(context, a); // login flow ABOVE current session
                await manager.switchAccount(a.key);
              },
            ),
          ),
        if (accounts.length < AccountRegistry.maxAccounts)
          ListTile(
            leading: const Icon(Icons.person_add_alt),
            title: const Text('Add account'),
            onTap: () => context.push('/add-account'), // root-level modal; cancel returns here
          )
        else
          const ListTile(title: Text('You can sign in to up to 5 accounts')),
      ]),
    );
  }
}

class PortalPicker extends ConsumerStatefulWidget {
  const PortalPicker({super.key, required this.manager});
  final SessionManager manager;
  @override
  ConsumerState<PortalPicker> createState() => _PortalPickerState();
}

class _PortalPickerState extends ConsumerState<PortalPicker> {
  String _q = '';

  @override
  Widget build(BuildContext context) {
    final session = ref.watch(accountSessionProvider);
    final current = ref.watch(portalSessionProvider).portal.id;
    final portals = ref.watch(registryProvider).byKey(session.account.key)!.portals;
    final recent = ([...portals]..sort((a, b) => (b.lastUsedAt ?? DateTime(0)).compareTo(a.lastUsedAt ?? DateTime(0))))
        .where((p) => p.lastUsedAt != null && p.id != current)
        .take(3)
        .toList();
    final all = portals.where((p) => p.name.toLowerCase().contains(_q.toLowerCase())).toList()
      ..sort((a, b) => a.name.toLowerCase().compareTo(b.name.toLowerCase()));

    Widget tile(PortalRef p) => ListTile(
          key: ValueKey(p.id),
          title: Text(p.name, overflow: TextOverflow.ellipsis),
          subtitle: Text(p.role.name),
          trailing: p.id == current ? const Icon(Icons.check) : null,
          onTap: () {
            Navigator.pop(context);
            if (p.id != current) widget.manager.switchPortal(p.id);
          },
        );

    return CustomScrollView(slivers: [
      if (portals.length > 8)
        SliverToBoxAdapter(
          child: Padding(
            padding: const EdgeInsets.all(16),
            child: TextField(
              decoration: const InputDecoration(prefixIcon: Icon(Icons.search), hintText: 'Search workspaces'),
              onChanged: (v) => setState(() => _q = v),
            ),
          ),
        ),
      if (_q.isEmpty && recent.isNotEmpty) ...[
        const SliverToBoxAdapter(child: ListTile(title: Text('Recent'))),
        SliverList.list(children: [for (final p in recent) tile(p)]),
        const SliverToBoxAdapter(child: Divider()),
      ],
      SliverList.builder(itemCount: all.length, itemBuilder: (_, i) => tile(all[i])),
    ]);
  }
}
```

### A.9 Push and deep-link routing to (account, portal)

```dart
sealed class RouteTarget {
  const RouteTarget();
}
final class Open extends RouteTarget {
  const Open(this.account, this.portalId, this.location, {required this.needsConfirm});
  final Account account;
  final String portalId, location;
  final bool needsConfirm;
}
final class Choose extends RouteTarget {
  const Choose(this.candidates, this.portalId, this.location);
  final List<Account> candidates;
  final String portalId, location;
}
final class Reauth extends RouteTarget {
  const Reauth(this.account, this.pending);
  final Account account;
  final Uri pending;
}
final class NotMember extends RouteTarget {
  const NotMember(this.portalId);
  final String portalId;
}
final class Drop extends RouteTarget {
  const Drop();
}

class AccountRouteResolver {
  AccountRouteResolver(this.registry);
  final AccountRegistry registry;

  /// Push payload: {"account_id": "...", "env": "prod", "portal_id": "...", "path": "/items/42"}
  RouteTarget forPush(Map<String, dynamic> data) {
    final key = AccountKey('${data['env'] ?? 'prod'}:${data['account_id']}');
    final a = registry.byKey(key);
    final portalId = data['portal_id'] as String?;
    final path = data['path'] as String? ?? '/home';
    if (a == null || portalId == null) return const Drop(); // signed out on this device
    if (a.needsReauth) return Reauth(a, Uri(path: path));
    if (a.portal(portalId) == null) return NotMember(portalId);
    // A tapped notification already showed account/portal context → no extra confirm.
    return Open(a, portalId, path, needsConfirm: false);
  }

  /// Link: https://app.example.com/p/<portalId>/items/42[?account=<userId>]
  RouteTarget forLink(Uri uri) {
    final seg = uri.pathSegments;
    if (seg.length < 2 || seg.first != 'p') return const Drop();
    final portalId = seg[1];
    final location = '/${seg.skip(2).join('/')}';
    final hint = uri.queryParameters['account'];
    final members = registry.accounts.where((a) => a.portal(portalId) != null).toList();
    final candidates = hint == null ? members : members.where((a) => a.userId == hint).toList();
    if (candidates.isEmpty) return NotMember(portalId);
    if (candidates.length > 1) return Choose(candidates, portalId, location);
    final a = candidates.single;
    if (a.needsReauth) return Reauth(a, uri);
    final act = registry.active;
    final isCurrent = act?.account == a.key && act?.portalId == portalId;
    return Open(a, portalId, location, needsConfirm: !isCurrent);
  }
}

Future<void> handleTarget(BuildContext context, SessionManager m, RouteTarget t) async {
  switch (t) {
    case Open(:final account, :final portalId, :final location, :final needsConfirm):
      if (needsConfirm) {
        final portalName = account.portal(portalId)!.name;
        final ok = await showDialog<bool>(
          context: context,
          builder: (c) => AlertDialog(
            title: Text('Open in $portalName?'),
            content: Text('This will switch to ${account.email}.'),
            actions: [
              TextButton(onPressed: () => Navigator.pop(c, false), child: const Text('Cancel')),
              FilledButton(onPressed: () => Navigator.pop(c, true), child: const Text('Switch')),
            ],
          ),
        );
        if (ok != true) return;
      }
      // switchAccount is a no-op on the account layer when already active; portal-only switch otherwise.
      await m.switchAccount(account.key, portalId: portalId, location: location);
    case Choose():
      // show a chooser listing candidate accounts, then Open(..., needsConfirm: false)
      break;
    case Reauth(:final account, :final pending):
      // store `pending`, run login for `account` above current session, re-resolve after success
      break;
    case NotMember(:final portalId):
      // "This link belongs to a workspace you're not in" + Add account / Request access
      break;
    case Drop():
      break;
  }
}
```

Note: `switchAccount(..., location:)` restores only allow-listed locations via `_safeLocation`; for push/link targets either widen the allow-list for the target route or navigate with `router.go(location)` on the new `Ready` router after the switch completes, so the synthesized stack is portal home → target (see **flutter-deep-links**).

### A.10 Tests (sketch)

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:http_mock_adapter/http_mock_adapter.dart';

class InMemoryTokenStore implements TokenStore {
  final m = <AccountKey, Tokens>{};
  @override
  Future<Tokens?> read(AccountKey k) async => m[k];
  @override
  Future<void> write(AccountKey k, Tokens t) async => m[k] = t;
  @override
  Future<void> delete(AccountKey k) async => m.remove(k);
  @override
  Future<void> deleteAll() async => m.clear();
  @override
  Future<Set<AccountKey>> keys() async => m.keys.toSet();
}

void main() {
  const a = AccountKey('prod:a');

  test('concurrent refreshes are single-flight per account', () async {
    final store = InMemoryTokenStore()
      ..m[a] = Tokens(access: 'old', refresh: 'r1', expiresAt: DateTime.now().add(const Duration(hours: 1)));
    var refreshCalls = 0;
    final holder = TokenHolder(account: a, store: store, baseUrl: 'https://api.test', onRevoked: (_) {});
    // In a real test, inject the bare Dio and stub /oauth/token with DioAdapter, counting calls.
    final results = await Future.wait(List.generate(5, (_) => holder.refresh(stale: 'old')));
    expect(results.map((t) => t.access).toSet().length, 1);
    expect(refreshCalls, lessThanOrEqualTo(1));
  });

  test('revoked refresh marks only that account', () async {
    // stub /oauth/token → 400 {"error":"invalid_grant"} for account A only;
    // expect: store has no A tokens, registry A.needsReauth == true, B tokens unchanged.
  });

  testWidgets('switch mid-request drops the old response', (tester) async {
    // 1. Ready(A); start a request held by a Completer.
    // 2. manager.switchAccount(B); pump.
    // 3. complete A's response; pump.
    // expect: no A-only marker text visible; A container disposed; B DB contains no A rows.
  });

  testWidgets('push for inactive portal switches portal, not account', (tester) async {
    // Ready(A, p1). resolver.forPush({account_id: a, portal_id: p2, path: /items/1})
    // expect: same AccountSession instance; PortalSession.portal.id == p2; ItemScreen shown.
  });

  testWidgets('sign out last account lands on login with empty storage', (tester) async {
    // signOut(A) when only A exists → SignedOut; store.keys() empty; accounts dir gone.
  });
}
```
