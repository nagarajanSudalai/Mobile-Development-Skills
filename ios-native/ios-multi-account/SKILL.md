---
name: ios-multi-account
description: Use when implementing, reviewing, or debugging multi-account and multi-portal (workspace / organization / team) support in an iOS/iPadOS app in Swift — account registry, per-account Keychain items with the right kSecAttrAccessible class, shared Keychain access groups between your own apps and extensions, per-account token refresh (single-flight), SwiftData/Core Data store per account (and per portal), URLSession and URLCache per account, a scoped @Observable Session container, account switch and portal switch flows, add account / sign out, push and deep-link routing to the right account+portal, background tasks, StoreKit appAccountToken / RevenueCat identity, analytics identity, app lock and privacy. Siblings: android-multi-account, react-native-multi-account, flutter-multi-account; related: ios-deep-links, ios-push-notifications, ios-in-app-purchase, ios-revenuecat-iap, ios-list-ux.
---

# iOS Multi-Account Skill

Guide for building and auditing apps where **several signed-in identities live on one device** (personal + work, several clients, family members) and where **each account can belong to several portals** (workspaces, organizations, teams, tenants, stores) between which the user switches. Covers Swift 5.9+/Swift 6, SwiftUI with Observation (`@Observable`), UIKit hosts, Keychain Services, SwiftData and Core Data, `URLSession`, `UserNotifications`, `BGTaskScheduler`, StoreKit 2.

Siblings: **android-multi-account**, **react-native-multi-account**, **flutter-multi-account** — keep the product contract (limits, switch semantics, push payload shape, sign-out rules) identical across platforms so the backend serves one model. Related: **ios-deep-links** (link resolution before an account/portal is chosen), **ios-push-notifications** (registration, payload, service extension), **ios-in-app-purchase** / **ios-revenuecat-iap** (purchase identity), **ios-list-ux** (lists must reset correctly on switch).

Keychain, SwiftData and Observation behaviour shifts between OS releases (SwiftData `ModelConfiguration` options, `#Unique`, history tracking in iOS 18; `@Observable` and `@Bindable` iOS 17; `UNUserNotificationCenter.setBadgeCount` iOS 16; data-protection keychain on macOS/Catalyst). **Verify exact signatures and availability against current Apple docs before shipping**, and gate with `if #available` when the deployment target is lower.

Key gates (verify): `@Observable`, SwiftData, `ModelConfiguration(url:)` iOS 17; `#Unique`, `#Index`, SwiftData history iOS 18; `URLCache(memoryCapacity:diskCapacity:directory:)` iOS 13; `BGTaskScheduler` iOS 13; StoreKit 2 `appAccountToken` iOS 15; `setBadgeCount` iOS 16; `LAContext` biometrics iOS 8+ (Face ID 11+).

## 0. Mental model

Two layers, always modelled separately:

```
 Device / app install
 ├── AccountRegistry (persisted, non-secret metadata, ordered)
 │     ├── Account A  (id, email, avatar, region=eu, env=prod, tokenRef → Keychain)
 │     │     ├── Portal A1 (role=admin, lastUsedAt)   ← lastActivePortal(A)
 │     │     └── Portal A2 (role=viewer)
 │     └── Account B  (region=us)
 │           └── Portal B1
 ├── Keychain (one item per account: service=<app>.auth, account=<accountId>)
 │     shared access group → visible to your Notification Service Ext / widgets / sibling apps
 └── Active selection: (accountId, portalId)   ← exactly one, persisted

 Runtime (in memory)
 AppModel (@MainActor @Observable)
   └── session: Session?            ← ONE scoped container, rebuilt on ACCOUNT switch
         ├── account: Account
         ├── tokens: TokenProvider  (single-flight refresh for this account)
         ├── http: APIClient        (own URLSession, URLCache, cookie store)
         ├── store: ModelContainer  (Application Support/Accounts/<id>/data.store)
         └── portal: PortalScope?   ← lighter scope, rebuilt on PORTAL switch
               ├── portal: Portal
               ├── repositories / view models / query caches for this portal
               └── realtime socket subscribed to this portal
```

Rules of thumb:

- **Account switch = destroy and rebuild `Session`.** Nothing from the old account survives in memory.
- **Portal switch = destroy and rebuild `PortalScope`** inside the same `Session`; token, HTTP session and account store survive (unless portals use separate tokens/stores — see 4, 5).
- **Everything account-specific is reached through the session**, never through a global singleton. A singleton holding user data is the #1 source of cross-account leaks.
- Background/system entry points (push, links, BG tasks, extensions) carry or resolve an **(accountId, portalId)** and must work for **inactive** accounts too.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Max accounts** per device (2? 5? unlimited?) and whether the same person can add the same account twice (should be no — dedupe, section 8).
- **Backends**: one backend, or region/environment per account (EU/US data residency, staging vs prod, self-hosted base URL)? Different auth servers / OAuth issuers?
- **Portals per account**: typical and max (3 vs 3,000 — drives picker UX). Is there always at least one? Can an account exist with zero portals (pending invite)?
- **Token model**: one token per account with a portal header (`X-Portal-Id`), or a **portal-scoped token** exchanged from the account token? Refresh-token rotation? Lifetimes?
- **Data isolation**: must portals be isolated on disk (enterprise/tenant requirement) or is per-account isolation enough with `portalId` columns? Any data shared across accounts (public catalog, app config)?
- **Offline**: does each account keep a local store? Size? Must inactive accounts stay synced?
- **Push**: notifications for all signed-in accounts or only the active one? Per-portal mute?
- **Extensions / sibling apps**: Notification Service Extension, widgets, share extension, watch app, or **other apps from the same Team ID** that should see the same accounts (SSO between your own apps)?
- **Purchases**: is the subscription owned by the account (server-side entitlements), by the portal (seat/org billing), or by the Apple ID?
- **Security**: per-account app lock / biometrics? MDM / managed app config? Compliance (HIPAA, SOC2) requiring wipe on sign-out?
- **Min OS** and UI framework (SwiftUI + Observation needs iOS 17; otherwise `ObservableObject`).

Write a **multi-account spec** before coding:

```
max accounts: 5 | same account twice: blocked (by server user id + issuer)
regions: eu, us (base URL per account) | env switch: debug builds only
portals/account: 1..500 → searchable picker + recents | default: last used, else server default
token: account access+refresh (rotating); portal via X-Portal-Id header
isolation: SwiftData store per account; portalId column; caches per account
push: all accounts; payload {account_id, portal_id, ...}; per-portal mute
extensions: NSE (reads tokens via shared access group), widget (active account only)
purchases: account-owned; appAccountToken = account UUID; RevenueCat logIn(accountId)
lock: optional Face ID per account | sign-out: wipe store + keychain + caches
```

## 2. Data model

- [ ] **`Account`** (non-secret, persisted): `id` (server user id — stable, not email), `issuer`/`region`/`baseURL`/`environment`, `displayName`, `email` (masked in UI if needed), `avatarURL`, `addedAt`, `lastActiveAt`, `sortIndex`, `status` (`active`, `needsReauth`, `removed`), `lastActivePortalId`.
- [ ] **Account identity key** = `(issuer or region, serverUserId)`. Two environments can return the same numeric id — never key by id alone if backends differ.
- [ ] **`Portal`**: `id`, `accountId`, `name`, `iconURL`, `role`, `permissions` (or a version/etag to refetch), `lastUsedAt`, `isDefault`, `status` (`active`, `revoked`, `pendingInvite`).
- [ ] **Active selection** persisted as `(accountId, portalId)` — a single value, written atomically. On launch: validate both still exist, else fall back (section 9).
- [ ] **Last-active portal per account** stored on the account so switching back to A lands on A's last portal, not B's.
- [ ] **Where the registry lives**: a small JSON file (or `UserDefaults` suite) in the **app group container** if extensions need it; never tokens in it. SwiftData for the registry is fine but keep it **separate** from per-account stores (it must open before any account store).
- [ ] **Ordering**: user-reorderable or by `lastActiveAt`; stable across launches.
- [ ] **Limits** enforced in the registry (`addAccount` throws `.limitReached`), not just in UI.
- [ ] **Portal list is a cache**: refreshed from server on login, on switch, on foreground (throttled) and on push hints; server is the source of truth for membership and role.
- [ ] Registry mutations are **serialised** (an `actor` or `@MainActor` type) — concurrent sign-in + token-refresh-failure + push must not corrupt it.

## 3. Secure storage of tokens per account (Keychain)

- [ ] **One Keychain item per account** (generic password): `kSecClass = kSecClassGenericPassword`, `kSecAttrService = "<bundle>.auth"`, `kSecAttrAccount = <accountId>`, `kSecValueData` = JSON of `{accessToken, refreshToken, expiresAt, portalTokens?}`. Do not use one item holding all accounts — a write race on refresh would drop another account's tokens.
- [ ] **`kSecAttrAccessible`**:
  - `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` — **default for tokens**. Readable after the first unlock since boot, so background refresh, BG tasks, and the Notification Service Extension work while the device is locked; `ThisDeviceOnly` keeps it out of backups and device migration (tokens should not reappear on a restored phone).
  - `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` — stricter; use for high-value secrets that are never needed in the background. Background reads fail with `errSecInteractionNotAllowed` (-25308) — handle it, do not treat it as "signed out".
  - `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` — item is deleted if the passcode is removed; only for app-lock material.
  - Avoid `kSecAttrAccessibleAlways*` (deprecated) and the non-`ThisDeviceOnly` variants for tokens.
- [ ] **`kSecAttrSynchronizable = false`** (default) — never sync tokens via iCloud Keychain.
- [ ] **Biometric gating** via `SecAccessControlCreateWithFlags(... .biometryCurrentSet ...)` only for an **app-lock secret**, never for the refresh token itself — otherwise every background refresh needs Face ID and silently fails.
- [ ] **Shared access group between your own apps/extensions**:
  - Add the **Keychain Sharing** capability (`keychain-access-groups` entitlement) with e.g. `$(AppIdentifierPrefix)com.example.shared` to the app, the Notification Service Extension, widgets, and any sibling app from the **same Team ID**. Access groups cannot span Team IDs.
  - Pass `kSecAttrAccessGroup = "<TEAMID>.com.example.shared"` on add **and** query. Read the prefix at runtime (e.g. from a probe item's returned `kSecAttrAccessGroup`, or an Info.plist key set to `$(AppIdentifierPrefix)`) — do not hard-code a team id that differs between dev/enterprise builds.
  - Without `kSecAttrAccessGroup`, items go to the **first** group in the entitlement list — make the private one first if you also have a shared one, and explicitly choose per item: tokens the NSE needs → shared group; app-lock secret → private group.
  - App group (`com.apple.security.application-groups`) shares **files/UserDefaults**, not Keychain. You usually need both: app group for the registry JSON, access group for tokens.
  - SSO between your own apps: sibling app reads the registry + token items from the shared group, but should still **exchange** for its own token (separate client id/scope) rather than reuse the other app's refresh token, to avoid rotation races between two apps refreshing the same token.
- [ ] **macOS / Mac Catalyst**: set `kSecUseDataProtectionKeychain = true` so behaviour matches iOS.
- [ ] **Reinstall orphan cleanup**: Keychain items **survive app deletion**. On first launch after install (flag in `UserDefaults` missing), delete all items for your service in your groups — otherwise a reinstall "remembers" accounts the registry no longer lists, or vice versa. Be careful with shared groups: only wipe if no sibling app is installed / the registry in the app group is also empty.
- [ ] **No tokens** in `UserDefaults`, plist, SwiftData, Core Data, logs, crash reports, `NSUbiquitousKeyValueStore`, or screenshots of debug menus.
- [ ] Handle errors explicitly: `errSecItemNotFound` (no token → needs re-auth), `errSecDuplicateItem` (use update), `errSecInteractionNotAllowed` (locked device — retry later), `errSecMissingEntitlement` (-34018, misconfigured entitlement/simulator).
- [ ] Keychain calls are synchronous and can block — call from a background actor, not on the main thread in hot paths.

## 4. Per-account token lifecycle

- [ ] **One `TokenProvider` actor per account**; refresh is **single-flight**: concurrent 401s for account A share one refresh `Task`; account B's refresh is independent and never blocked by A's.
- [ ] Proactive refresh shortly before `expiresAt` (skew 30–60 s) plus reactive refresh on 401; retry the original request **once** after refresh.
- [ ] **Rotating refresh tokens**: persist the new refresh token to Keychain **before** using the new access token; a crash between receiving and saving otherwise logs the user out. If the NSE or a sibling app also refreshes, serialise via a server-side grace window or let only the main app refresh (NSE uses access token, falls back to a generic notification).
- [ ] **Refresh failure semantics**: `invalid_grant` / 401 on refresh → mark **that account** `needsReauth`, keep its data, show a re-login banner on that account; other accounts unaffected. Network errors → keep tokens, retry with backoff; never sign out on offline.
- [ ] Active account becomes `needsReauth` → show a re-auth sheet scoped to it (prefill email), offer switching to another account; do not silently switch.
- [ ] **Portal scoping**:
  - Header model: `X-Portal-Id` added by the `PortalScope`'s request builder, **not** a mutable global header on a shared client (race: request built for portal A sent after switch to B).
  - Portal-token model: exchange account token → portal token per portal; cache in the account's Keychain item keyed by portalId; refresh single-flight per (account, portal).
  - Every request captures `(accountId, portalId)` at creation time; responses for a scope that no longer exists are **dropped**, not written into the new scope's store.
- [ ] 403 "not a member of portal" ≠ 401 — triggers portal revocation flow (section 7), not token refresh.
- [ ] Clock skew: compute expiry from `expires_in` relative to response receipt time, not device wall clock vs server `exp` alone.

## 5. Data isolation

- [ ] **Directory per account**: `Application Support/Accounts/<accountId>/` (in the app group container only if extensions need it). Contains the store, files, image disk cache, URLCache. Deleting the directory = deleting the account's data.
- [ ] **SwiftData**: one `ModelContainer` per account, `ModelConfiguration(url: dir/"data.store")`, built inside the `Session`. Never use the default container from `.modelContainer(for:)` at the app root for account data — it is shared by all accounts. Inject with `.modelContainer(session.store)` under the session's view subtree so the environment `modelContext` follows the account.
- [ ] **Core Data**: one `NSPersistentContainer` per account with `NSPersistentStoreDescription(url:)`; same model, different file. Destroy with `persistentStoreCoordinator.destroyPersistentStore(at:type:)` before deleting files (WAL/SHM side files).
- [ ] **Per portal**: default to a `portalId` attribute on every portal-scoped model + predicate on every fetch (`#Predicate { $0.portalId == pid }`). Use a **store per portal** only when isolation is mandated or portals are huge; then the `PortalScope` owns its container and you cap how many portal stores are kept (LRU evict).
- [ ] **CloudKit**: if you use SwiftData/Core Data CloudKit sync, the iCloud account is device-level — it does not map to your app accounts. Do not mirror per-account data to the user's private CloudKit DB unless that is the intent; set `cloudKitDatabase: .none` for account stores.
- [ ] **File protection**: account directories with `FileProtectionType.completeUntilFirstUserAuthentication` (matches the Keychain class so BG work can read) or `.complete` if never needed in background.
- [ ] **Backup exclusion**: `URLResourceValues.isExcludedFromBackup = true` on caches and on re-downloadable stores; tokens are already `ThisDeviceOnly`. If you exclude a store from backup, a restored device has registry entries without stores — handle "store missing → resync".
- [ ] **HTTP**: one `URLSession` per account with its own `URLSessionConfiguration`: `urlCache = URLCache(memoryCapacity:diskCapacity:directory: accountDir/"http")`, own cookie storage (`.ephemeral` configuration + explicit disk `URLCache`, or a per-account `HTTPCookieStorage`), `httpAdditionalHeaders` without auth (auth added per request by the token provider). Never `URLSession.shared` or `URLCache.shared` for authenticated calls — cached responses leak across accounts.
- [ ] **Background `URLSession`** (uploads/downloads): identifier includes accountId (`"<bundle>.bg.<accountId>"`); reconnect in `application(_:handleEventsForBackgroundURLSession:)` by parsing the identifier.
- [ ] **Image/thumbnail caches**: memory `NSCache` + disk cache owned by the session, or keys prefixed with accountId. Signed/authenticated image URLs must not be served from another account's cache.
- [ ] **Query/in-memory caches, view models, repositories**: live in `Session`/`PortalScope`, die with them.
- [ ] **Preferences**: account settings in `UserDefaults(suiteName:)` keyed `"<accountId>.<key>"`, or in the account store. Device settings (theme, haptics) stay global — decide explicitly per key.
- [ ] **Spotlight / Handoff / Siri / widgets / Live Activities**: index with domain identifier per account (`CSSearchableItem.domainIdentifier = accountId`) so you can `deleteSearchableItems(withDomainIdentifiers:)` on sign-out; widgets show active account only or include the account name; end Live Activities of a removed account.
- [ ] **Singletons audit**: `grep` for `static let shared`, `static var current`, global `var`, `@AppStorage` with user data. Each must be device-level or moved into the session.

## 6. Account switch

Order matters — teardown fully, then build:

1. [ ] **Guard**: unsaved drafts / in-progress uploads in current account → confirm or persist draft into the old account's store first.
2. [ ] Set `AppModel.phase = .switching(to:)` → UI shows a lightweight switching overlay (avatar + name of target) and blocks interaction; avoid flashing the old account's content under the new header.
3. [ ] **Teardown old session**: cancel all owned `Task`s (keep handles; use a task group or `withTaskCancellationHandler`), close websockets, `urlSession.invalidateAndCancel()` (not `finishTasksAndInvalidate` for foreground), stop timers/observers, unsubscribe realtime, flush pending writes, release `ModelContainer`.
4. [ ] **Persist selection**: `activeAccountId = B`, `portalId = B.lastActivePortalId ?? default`.
5. [ ] **Build new session**: load B's tokens (if missing → `needsReauth` flow), open B's store, new `APIClient`, then `PortalScope`.
6. [ ] **Reset navigation**: replace the `NavigationPath`/tab selection with B's restored state; pop sheets and full-screen covers. Use `.id(session.id)` on the root content view so SwiftUI discards all `@State` from the previous account.
7. [ ] **Restore last screen per account** (optional): persist a lightweight route per (account, portal), restore if still valid, else portal home.
8. [ ] **Side systems**: analytics/crash user id (14), purchases identity (13), widget reload (`WidgetCenter.shared.reloadAllTimelines()`), badge recompute (10).
9. [ ] Clear `phase`. Announce for VoiceOver ("Switched to <name>").

- [ ] Switching must be **idempotent and re-entrant-safe**: a second switch request while one is in progress cancels/queues, never runs two builds concurrently.
- [ ] Switcher UI: avatar in nav bar / profile tab; sheet listing accounts (avatar, name, email, region badge, unread count, `needsReauth` warning), "Add account", "Manage accounts". Long-press on the tab item as a shortcut is a nice-to-have; Home Screen quick actions (`UIApplicationShortcutItem`) per account are possible but update them on change.
- [ ] Target < 300 ms perceived; open the store and show cached content before the network refresh.

## 7. Portal switch

Lighter than account switch:

| | Account switch | Portal switch |
|---|---|---|
| Token / Keychain | swapped | kept (or portal token swapped) |
| URLSession / URLCache | rebuilt | kept |
| ModelContainer | rebuilt | kept (unless per-portal store) |
| Portal-scoped VMs, query caches, sockets | rebuilt | rebuilt |
| Navigation | reset | reset to portal home (or restore portal's last route) |
| Analytics | new user id | same user id, new portal/group property |
| Purchases identity | switched | unchanged (unless portal-billed) |

- [ ] Rebuild only `PortalScope`; root content uses `.id(portalScope.id)` for its subtree.
- [ ] Cancel in-flight portal requests; drop late responses tagged with the old portalId.
- [ ] Persist `account.lastActivePortalId` and the portal's `lastUsedAt`.
- [ ] **Picker UX**: ≤ ~7 portals → simple list/menu; more → sheet with `.searchable`, "Recent" section (by `lastUsedAt`, 3–5), then alphabetical; show role badge; current portal checked; offline → show cached list with stale indicator. Consider cross-account picker ("All workspaces", grouped by account) if users think in portals rather than accounts — then a portal pick may imply an account switch.
- [ ] **Default portal on login**: server `default_portal_id` → else last used → else first; zero portals → empty state with "Join / create / accept invitation", not a crash.
- [ ] **Portal removed / access revoked mid-session** (403 membership, push hint, list refresh): stop requests, purge that portal's local data, toast "You no longer have access to <name>", switch to another portal of the same account; if none, the account's empty state. Never fall through to another account's portal silently.
- [ ] **Role/permission change**: refetch permissions on push hint / foreground; UI driven by `portalScope.permissions` (observable) so buttons disappear without restart; server still enforces.
- [ ] **Invitations**: pending invites per account shown in picker with accept/decline; accepting adds portal and offers switching; invite deep link for an account not on device → add-account flow then accept (section 11).

## 8. Add account

- [ ] Login flow presented **over** the current session (sheet); current account keeps working until success. Cancel returns unchanged.
- [ ] Use `ASWebAuthenticationSession` with `prefersEphemeralWebBrowserSession = true` for OAuth so the browser cookie of account A does not auto-sign-in as A again. Sign in with Apple: one Apple ID per device → adding a second "Apple" account usually yields the same user; explain or disallow.
- [ ] **Region/environment choice** before or discovered by login (email domain lookup); base URL becomes part of the account.
- [ ] **Duplicate detection** after the token exchange by `(issuer, serverUserId)`: if present → update its tokens (re-auth), switch to it, toast "Already signed in" — never create a second entry.
- [ ] **Limit** reached → disable "Add account" with explanation.
- [ ] On success: save Keychain item first, then registry entry, then fetch portals, register push for the new account, then switch (or ask "Switch now?").
- [ ] Partial failure (tokens saved, portal fetch failed) → account added with retry state; no orphan Keychain item if the registry write fails (compensate by deleting).

## 9. Sign out

- [ ] **Sign out of one account** vs **sign out of all** — both available in account management.
- [ ] Steps: server revoke (best effort, with timeout; queue if offline) → unregister push token for that account → tear down session if active → delete Keychain item → destroy store + delete account directory → remove Spotlight domain, cancel BG tasks / background URLSessions for it, end Live Activities, remove delivered notifications whose `userInfo.account_id` matches (`removeDeliveredNotifications(withIdentifiers:)`) → remove registry entry.
- [ ] **Next active account**: most recently used remaining → its last portal; none left → logged-out welcome screen (not a login screen pre-filled with the removed email).
- [ ] Signing out the last account resets purchase identity (RevenueCat `logOut`), analytics identity, badge to 0.
- [ ] **Account deletion** (App Store requirement for account creation apps): server deletion + full local wipe as above.
- [ ] Sign-out must succeed **offline** locally; it is never blocked by the network.

## 10. Push per account / portal

(Detail in **ios-push-notifications**.)

- [ ] APNs device token is **per device/app**, not per account. Register it with the backend **for every signed-in account** (`POST /devices` with each account's token), and re-register all on token change and after add-account.
- [ ] Payload contract: `account_id`, `portal_id` (server ids, not emails), `type`, target route; optional `thread-id = "<account>.<portal>"` for grouping and `target-content-id`.
- [ ] **Routing on receipt** (`userNotificationCenter(_:willPresent:)`): notification for active account+portal → banner; other portal or account → banner with account/portal name prefix (set in NSE), update that scope's unread counts.
- [ ] **On tap** (`didReceive response`): resolve account → if not on device (signed out) → ignore/open home; if `needsReauth` → re-auth flow; else switch account (if needed) → switch portal (if needed) → navigate. Confirmation is not needed for a tap the user made, but show a brief "Switched to <account>" toast.
- [ ] **Notification Service Extension**: reads registry (app group) and tokens (shared access group, `AfterFirstUnlock`) to decrypt/enrich, prefixes title with account/portal name when multiple accounts exist. Must not write tokens (rotation race).
- [ ] **Badge**: one app icon badge → sum of unread across accounts (or active only — decide); per-account/portal counts shown in the switcher and portal picker. Set via `UNUserNotificationCenter.current().setBadgeCount(_:)` (iOS 16+); server-sent `badge` must be computed across all accounts on that device, or omit it and compute locally.
- [ ] **Unregister** on sign-out of that account (server `DELETE /devices/<token>?account=`), and per-portal mute settings respected server-side.
- [ ] Silent pushes (`content-available`) for inactive accounts: sync that account's store in the background using a temporary, non-UI session; do not switch the active session.

## 11. Deep links per account / portal

(Detail in **ios-deep-links**.)

- [ ] Links carry portal/account hints: path (`/p/<portalId>/items/42`) or resolve server-side (`GET /links/resolve` → `{accountCandidates, portalId}`).
- [ ] Resolution: active account+portal can open it → navigate. Another signed-in account owns the portal → **confirm** ("Open in <Work account · Acme>?") then switch and navigate. Several accounts can → let the user pick. None → offer "Add account" with the pending link stored, resume after login.
- [ ] Never switch accounts silently from a link (phishing/confusion); never perform destructive actions from links.
- [ ] Pending link survives the switch: store it, perform switch, then deliver to the new session's router after it is ready (not to the torn-down one).

## 12. Background work

- [ ] `BGTaskScheduler` identifiers are **static** (Info.plist `BGTaskSchedulerPermittedIdentifiers`); you cannot register one per account dynamically. Use one task (`<bundle>.refresh`) that **iterates accounts** with a time budget, round-robin across launches, honouring `expirationHandler`.
- [ ] Each account's background work uses a **headless session** (tokens + store + HTTP for that account) that does not touch the UI `Session`; if the account becomes active meanwhile, the two must not write the same store concurrently — use the store's own concurrency (`ModelActor`) and one writer per account.
- [ ] Background URLSessions per account (section 5). Uploads queued with accountId/portalId; on sign-out cancel them.
- [ ] Locked device: Keychain `AfterFirstUnlock` + file protection `completeUntilFirstUserAuthentication`, otherwise BG work fails silently before first unlock (and that's expected after reboot).

## 13. Purchases

(Detail in **ios-in-app-purchase**, **ios-revenuecat-iap**.)

- [ ] Apple ID / StoreKit is **device-level**: the same Apple ID pays regardless of which app account is active. Decide and document who owns the entitlement: account (recommended: server validates and stores), portal (org billing — usually not IAP), or Apple ID.
- [ ] StoreKit 2: pass `.appAccountToken(account.purchaseUUID)` (stable UUID per account, stored server-side) in `product.purchase(options:)` so server notifications map to the right account. `Transaction.currentEntitlements` returns the Apple ID's transactions for **all** accounts — filter by `appAccountToken` / server state before unlocking.
- [ ] RevenueCat: `Purchases.shared.logIn(accountId)` on account switch, `logOut()` when no accounts remain; never on portal switch unless portal-billed. Configure transfer behaviour (what happens when Apple ID's purchase is restored on account B) intentionally.
- [ ] Paywall and entitlements are part of the session; switching must not leave "Pro" UI from account A.

## 14. Analytics & crash reporting

- [ ] On account switch: set analytics user id to an **opaque id** (server id or hash — not email), reset per-user properties; on portal switch: set group/portal property, same user.
- [ ] Crash reporter user id + custom keys `portal_id`, `account_count`; **no PII** (emails, names, tokens, portal names if sensitive).
- [ ] Sign-out of all → reset analytics identity (new anonymous id).
- [ ] Events emitted during teardown must not be attributed to the new account (capture ids at event creation).

## 15. Security & privacy

- [ ] **Per-account app lock** (optional): a flag per account; on switching to a locked account or foregrounding, present a lock screen and `LAContext.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics or .deviceOwnerAuthentication)`; requires `NSFaceIDUsageDescription`. Lock state is per account; unlocking A does not unlock B.
- [ ] **App switcher snapshot**: on `scenePhase == .inactive/.background` overlay a privacy view (blur/logo) for sensitive accounts so the snapshot doesn't show account data; also before presenting the account switcher if accounts are confidential to each other (shared family device).
- [ ] Switcher shows **only what the user agreed to** (email may be masked); a locked account's unread counts may leak info — hide them when locked.
- [ ] **Full wipe** on sign-out/deletion verified (section 9 checklist); write a debug "Storage inspector" that lists per-account directories and Keychain items to verify.
- [ ] Logs: `Logger` with `privacy: .private` for ids; never log tokens or response bodies of auth calls.
- [ ] Managed devices: `UserDefaults.standard.dictionary(forKey: "com.apple.configuration.managed")` may restrict accounts (e.g. only work domain) — honour if the app is enterprise-distributed.
- [ ] Jailbreak/pasteboard: don't copy tokens to pasteboard; OAuth codes via `ASWebAuthenticationSession`, not custom-scheme-only redirects that another app could claim.

## 16. Testing plan

1. **Unit tests** for `AccountRegistry` (add, dedupe, limit, reorder, remove, next-active choice), `TokenProvider` (single-flight under 50 concurrent calls, rotation persistence, revoked vs offline), portal resolution and default selection, push/link routing decisions (pure functions of registry + payload).
2. **Keychain tests** on device and simulator (simulator needs a host app/entitlements for access groups; `-34018` in unit-test bundles without host).
3. **Integration tests** with two fake backends (two regions) and a stub server that can revoke a token, remove a portal, change a role.
4. **UI tests** (XCUITest) with launch arguments that seed two accounts × three portals.
5. **Leak audit**: after switching A → B, inspect memory graph (no A view models alive), `URLCache` directory, image cache, Spotlight; assert B's screens never render A's ids (debug assertion in repository: every model's `accountId == session.account.id`).
6. **Background**: `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"<id>"]` in the debugger; push via `xcrun simctl push` with payloads for an inactive account/portal; locked-device test on hardware.
7. **Reinstall** test: delete app, reinstall → no ghost accounts; with sibling app installed → shared accounts preserved as designed.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| M1 | Add second account while first active | Login sheet over A; on success B added, push registered, switch prompt |
| M2 | Add same account again | No duplicate; tokens updated; switch to existing; toast |
| M3 | Add account in other region | B uses its base URL; A unaffected |
| M4 | Add account at limit | Add disabled with explanation |
| M5 | Switch account mid-request | Old request cancelled; late response dropped; B shows only B data |
| M6 | Switch account with unsaved draft | Prompt or draft persisted to A; restored when back to A |
| M7 | Rapid double switch A→B→A | One final session (A); no crash, no mixed data |
| M8 | Switch back to A | Lands on A's last portal and last screen |
| M9 | Portal switch | Token/HTTP kept; portal VMs/sockets rebuilt; nav reset to portal home |
| M10 | Portal switch mid-request | Response for old portal not written to new portal view |
| M11 | Portal revoked mid-session (403 / push) | Data purged, toast, moved to another portal of same account |
| M12 | Last portal of account revoked | Account empty state with join/create; other accounts intact |
| M13 | Role downgraded admin → viewer | Admin actions disappear without restart |
| M14 | 500 portals | Searchable picker, recents, smooth scrolling |
| M15 | Login with zero portals / pending invite | Invite UI, not crash |
| M16 | Token expiry on active account, 10 parallel calls | One refresh; all retried once |
| M17 | Refresh token revoked on inactive account B | B marked needsReauth, banner in switcher; A unaffected |
| M18 | Refresh revoked on active account | Re-auth sheet for A; option to switch; data kept |
| M19 | Offline during refresh | No sign-out; retry later |
| M20 | Push for active account+portal (foreground) | Banner, routed in place |
| M21 | Push for inactive portal of active account | Banner with portal name; tap switches portal and navigates |
| M22 | Push for inactive account | Banner prefixed with account; tap switches account+portal then navigates |
| M23 | Push for signed-out account | Not shown (server unregistered) or ignored on tap |
| M24 | Push while device locked after reboot (before first unlock) | NSE fails gracefully with generic content |
| M25 | Deep link to portal of other signed-in account | Confirmation, then switch + navigate |
| M26 | Deep link to portal no account has | Add-account flow; link resumed after login |
| M27 | Sign out inactive account | Its data, Keychain item, notifications, Spotlight removed; active session untouched |
| M28 | Sign out active account (others remain) | Switch to most-recent remaining account's last portal |
| M29 | Sign out last account | Welcome screen; badge 0; purchases/analytics identity reset |
| M30 | Sign out while offline | Local wipe immediate; server revoke queued |
| M31 | Reinstall app | No ghost accounts from orphaned Keychain items |
| M32 | Restore device backup to new phone | Registry entries without tokens → needsReauth, no crash |
| M33 | Purchase on B after buying on A (same Apple ID) | Entitlement assignment per documented rule; UI correct per account |
| M34 | BG refresh with 3 accounts | All synced within budget, round-robin; no write conflicts with UI session |
| M35 | App lock enabled on B only | Switching to B requires Face ID; A doesn't; snapshot hidden |
| M36 | Sibling app (same team) installed | Sees accounts via shared group; own token exchange; no rotation fights |
| M37 | VoiceOver switch | Switcher fully labelled; switch announced |

## 17. Common bugs to look for in review

- `static let shared` repositories/caches/view models holding user data → account A data visible after switching to B
- `URLSession.shared` / `URLCache.shared` / `HTTPCookieStorage.shared` for authenticated calls → cached responses and cookies cross accounts
- Root-level `.modelContainer(for:)` for account data → all accounts in one store
- One Keychain item holding all accounts' tokens → refresh of A overwrites B
- Tokens in `UserDefaults` / `@AppStorage` / SwiftData
- `kSecAttrAccessibleWhenUnlocked` for refresh tokens → BG refresh and NSE fail with -25308, treated as logout
- Missing `kSecAttrAccessGroup` on query → item written to one group, read from another → "token missing" in extension
- Hard-coded team-id prefix in access group → breaks on a different signing team
- Keychain items surviving reinstall → ghost accounts or "logged in" without registry
- Global mutable `X-Portal-Id` / auth header on a shared client → requests sent with wrong portal after switch
- Late responses written into the new account's/portal's store (no scope check)
- No `.id(session.id)` on root → `@State` (search text, selection, scroll) from A shown in B
- Navigation stack not reset → B deep in A's detail screen with A's item id
- Refresh not single-flight → token rotation race logs user out
- Any refresh failure (including offline) → sign out all accounts
- APNs token registered only for active account → no notifications for others; or never unregistered on sign-out → notifications for removed accounts
- Push tap navigates in the active account without switching
- Silent account switch from a deep link
- Badge from server reflects only one account
- `Transaction.currentEntitlements` unlocks Pro for every app account on that Apple ID
- RevenueCat `logIn` called on portal switch, or not called on account switch
- Analytics user id = email
- Sign-out leaves account directory, Spotlight items, delivered notifications, background uploads
- Portal revoked → app crashes on missing portal or falls through to another account
- Duplicate account entries because dedupe used email (case/alias) instead of `(issuer, userId)`
- `ASWebAuthenticationSession` non-ephemeral → "Add account" silently signs into the same account again

## 18. Output format

When asked to **review/audit**:
1. **Summary** — multi-account spec table (section 1), architecture (session container? singletons?), storage (Keychain classes, access groups, store layout), min OS
2. **Checklist results** — sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** — cross-account data leaks, token storage/accessibility mistakes, unwanted sign-outs, wrong-account push/link routing, incomplete wipe — with fix code
4. **Testing gaps** — M1–M37 not covered

When asked to **implement**: fill the spec, start from the Appendix, adapt `Account`/`Portal`, the auth endpoints and the header-vs-portal-token choice, delete what you don't need (per-portal stores, app lock, sibling-app sharing), and list the entitlements to add (Keychain Sharing group, App Groups, Push, Background Modes) and server contract questions (push payload ids, link resolve endpoint, device registration per account).

When asked to **debug**: identify which scope (device / account / portal) owns the leaking or missing state, check (1) singletons and shared URLSession/ModelContainer, (2) Keychain query attributes (service, account, access group, accessibility) and the OSStatus, (3) scope capture on in-flight work, (4) navigation/`@State` identity — in that order.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI + SwiftData)

Simplified but complete enough to adapt. Names like `AuthAPI`, `PortalDTO` stand in for your backend. Verify SwiftData/Keychain signatures against your SDK.

### A.1 Models and registry

```swift
import Foundation

struct AccountKey: Hashable, Codable, Sendable {
    let issuer: String        // e.g. "https://auth.eu.example.com"
    let userId: String        // server user id
}

struct Account: Identifiable, Hashable, Codable, Sendable {
    let id: String            // stable local id = "\(issuer)|\(userId)" hashed, or server id if one backend
    let key: AccountKey
    var baseURL: URL
    var displayName: String
    var email: String
    var avatarURL: URL?
    var purchaseUUID: UUID    // StoreKit appAccountToken, also stored server-side
    var status: Status = .active
    var lastActivePortalId: String?
    var lastActiveAt: Date = .now
    var sortIndex: Int = 0

    enum Status: String, Codable, Sendable { case active, needsReauth }
}

struct Portal: Identifiable, Hashable, Codable, Sendable {
    let id: String
    let accountId: String
    var name: String
    var role: String
    var permissions: Set<String>
    var lastUsedAt: Date?
    var isDefault: Bool
}

struct ActiveSelection: Codable, Equatable, Sendable {
    var accountId: String
    var portalId: String?
}

private struct RegistryFile: Codable {
    var accounts: [Account] = []
    var portals: [String: [Portal]] = [:]   // accountId → portals (cache)
    var active: ActiveSelection?
}

enum RegistryError: Error { case limitReached, unknownAccount }

/// Persisted, non-secret. Lives in the app group so extensions can read it.
actor AccountRegistry {
    static let maxAccounts = 5
    private let url: URL
    private var file: RegistryFile

    init(appGroup: String) {
        let dir = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: appGroup)
            ?? URL.applicationSupportDirectory
        url = dir.appending(path: "accounts.json")
        file = (try? JSONDecoder().decode(RegistryFile.self, from: Data(contentsOf: url))) ?? RegistryFile()
    }

    var accounts: [Account] { file.accounts.sorted { $0.sortIndex < $1.sortIndex } }
    var active: ActiveSelection? { file.active }
    func account(_ id: String) -> Account? { file.accounts.first { $0.id == id } }
    func account(for key: AccountKey) -> Account? { file.accounts.first { $0.key == key } }
    func portals(of accountId: String) -> [Portal] { file.portals[accountId] ?? [] }

    /// Inserts or updates (dedupe by issuer+userId). Returns the stored account and whether it was new.
    func upsert(_ account: Account) throws -> (Account, isNew: Bool) {
        if let i = file.accounts.firstIndex(where: { $0.key == account.key }) {
            var existing = file.accounts[i]
            existing.displayName = account.displayName
            existing.email = account.email
            existing.avatarURL = account.avatarURL
            existing.status = .active
            file.accounts[i] = existing
            try save(); return (existing, false)
        }
        guard file.accounts.count < Self.maxAccounts else { throw RegistryError.limitReached }
        var new = account
        new.sortIndex = (file.accounts.map(\.sortIndex).max() ?? -1) + 1
        file.accounts.append(new)
        try save(); return (new, true)
    }

    func setPortals(_ portals: [Portal], for accountId: String) throws {
        file.portals[accountId] = portals
        try save()
    }

    func setStatus(_ status: Account.Status, for id: String) throws {
        guard let i = file.accounts.firstIndex(where: { $0.id == id }) else { return }
        file.accounts[i].status = status
        try save()
    }

    func setActive(_ selection: ActiveSelection?) throws {
        file.active = selection
        if let s = selection, let i = file.accounts.firstIndex(where: { $0.id == s.accountId }) {
            file.accounts[i].lastActiveAt = .now
            file.accounts[i].lastActivePortalId = s.portalId
            if let pid = s.portalId, let p = file.portals[s.accountId]?.firstIndex(where: { $0.id == pid }) {
                file.portals[s.accountId]?[p].lastUsedAt = .now
            }
        }
        try save()
    }

    /// Removes the account; returns the next account to activate (most recently used).
    func remove(_ id: String) throws -> Account? {
        file.accounts.removeAll { $0.id == id }
        file.portals[id] = nil
        if file.active?.accountId == id { file.active = nil }
        try save()
        return file.accounts.max { $0.lastActiveAt < $1.lastActiveAt }
    }

    func defaultPortal(for accountId: String) -> Portal? {
        let ps = portals(of: accountId)
        let last = account(accountId)?.lastActivePortalId
        return ps.first { $0.id == last } ?? ps.first(where: \.isDefault) ?? ps.first
    }

    private func save() throws {
        let data = try JSONEncoder().encode(file)
        try data.write(to: url, options: [.atomic, .completeFileProtectionUntilFirstUserAuthentication])
    }
}
```

### A.2 Keychain token store (per-account items, shared access group)

```swift
import Foundation
import Security

struct TokenSet: Codable, Sendable {
    var accessToken: String
    var refreshToken: String
    var expiresAt: Date
    var portalTokens: [String: PortalToken] = [:]   // only for the portal-token model
    struct PortalToken: Codable, Sendable { var token: String; var expiresAt: Date }
}

enum KeychainError: Error, Equatable {
    case notFound, interactionNotAllowed, missingEntitlement, unexpected(OSStatus)
    init(_ s: OSStatus) {
        switch s {
        case errSecItemNotFound: self = .notFound
        case errSecInteractionNotAllowed: self = .interactionNotAllowed   // device locked
        case -34018: self = .missingEntitlement                          // errSecMissingEntitlement
        default: self = .unexpected(s)
        }
    }
}

/// One generic-password item per account. Accessible after first unlock, never leaves the device.
/// Access group is shared with the Notification Service Extension and sibling apps (same Team ID).
struct KeychainTokenStore: Sendable {
    let service: String          // "com.example.app.auth"
    let accessGroup: String?     // "<TEAMID>.com.example.shared" — read prefix at runtime

    /// Info.plist: AppIdentifierPrefix = $(AppIdentifierPrefix)   (ends with ".")
    static func sharedGroup(suffix: String) -> String? {
        guard let prefix = Bundle.main.object(forInfoDictionaryKey: "AppIdentifierPrefix") as? String
        else { return nil }
        return prefix + suffix
    }

    private func baseQuery(_ accountId: String?) -> [String: Any] {
        var q: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecUseDataProtectionKeychain as String: true,   // macOS/Catalyst parity; no-op on iOS
        ]
        if let accountId { q[kSecAttrAccount as String] = accountId }
        if let accessGroup { q[kSecAttrAccessGroup as String] = accessGroup }
        return q
    }

    func save(_ tokens: TokenSet, for accountId: String) throws {
        let data = try JSONEncoder().encode(tokens)
        let attrs: [String: Any] = [
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
        ]
        var status = SecItemUpdate(baseQuery(accountId) as CFDictionary, attrs as CFDictionary)
        if status == errSecItemNotFound {
            var add = baseQuery(accountId)
            add.merge(attrs) { $1 }
            add[kSecAttrSynchronizable as String] = false
            status = SecItemAdd(add as CFDictionary, nil)
        }
        guard status == errSecSuccess else { throw KeychainError(status) }
    }

    func load(_ accountId: String) throws -> TokenSet {
        var q = baseQuery(accountId)
        q[kSecReturnData as String] = true
        q[kSecMatchLimit as String] = kSecMatchLimitOne
        var out: CFTypeRef?
        let status = SecItemCopyMatching(q as CFDictionary, &out)
        guard status == errSecSuccess, let data = out as? Data else { throw KeychainError(status) }
        return try JSONDecoder().decode(TokenSet.self, from: data)
    }

    func delete(_ accountId: String) throws {
        let status = SecItemDelete(baseQuery(accountId) as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else { throw KeychainError(status) }
    }

    /// All items for this service in this group — used for reinstall orphan cleanup.
    func deleteAll() {
        _ = SecItemDelete(baseQuery(nil) as CFDictionary)
    }
}

/// Keychain survives app deletion. Call once at launch, before reading tokens.
func cleanUpKeychainAfterReinstall(store: KeychainTokenStore, registryIsEmpty: Bool) {
    let flag = "didRunBefore"
    guard !UserDefaults.standard.bool(forKey: flag) else { return }
    // Shared group: only wipe if the shared registry (app group) is also empty,
    // otherwise a sibling app still uses these accounts.
    if registryIsEmpty { store.deleteAll() }
    UserDefaults.standard.set(true, forKey: flag)
}
```

### A.3 Single-flight token provider per account

```swift
import Foundation

enum AuthError: Error { case revoked, offline, needsReauth }

protocol AuthAPI: Sendable {
    func refresh(baseURL: URL, refreshToken: String) async throws -> TokenSet  // throws AuthError.revoked on invalid_grant
}

actor TokenProvider {
    let accountId: String
    private let baseURL: URL
    private let keychain: KeychainTokenStore
    private let api: AuthAPI
    private let onRevoked: @Sendable (String) async -> Void
    private var cached: TokenSet?
    private var refreshTask: Task<TokenSet, Error>?

    init(accountId: String, baseURL: URL, keychain: KeychainTokenStore, api: AuthAPI,
         onRevoked: @escaping @Sendable (String) async -> Void) {
        self.accountId = accountId; self.baseURL = baseURL
        self.keychain = keychain; self.api = api; self.onRevoked = onRevoked
    }

    func accessToken() async throws -> String {
        let tokens = try current()
        if tokens.expiresAt.timeIntervalSinceNow > 60 { return tokens.accessToken }
        return try await refresh(stale: tokens.accessToken).accessToken
    }

    /// Called on 401. `stale` = the token that failed; if already rotated, reuse.
    func refresh(stale: String) async throws -> TokenSet {
        let tokens = try current()
        if tokens.accessToken != stale, tokens.expiresAt > .now { return tokens }
        if let refreshTask { return try await refreshTask.value }       // single-flight

        let task = Task { [api, baseURL, keychain, accountId] () throws -> TokenSet in
            let new = try await api.refresh(baseURL: baseURL, refreshToken: tokens.refreshToken)
            try keychain.save(new, for: accountId)                       // persist BEFORE use (rotation)
            return new
        }
        refreshTask = task
        defer { refreshTask = nil }
        do {
            let new = try await task.value
            cached = new
            return new
        } catch AuthError.revoked {
            await onRevoked(accountId)                                   // this account only
            throw AuthError.needsReauth
        }
        // offline / 5xx propagate; tokens kept, never sign out
    }

    private func current() throws -> TokenSet {
        if let cached { return cached }
        do {
            let t = try keychain.load(accountId)
            cached = t
            return t
        } catch KeychainError.notFound {
            throw AuthError.needsReauth
        }
        // .interactionNotAllowed propagates: device locked → retry later, not logout
    }
}
```

### A.4 Per-account HTTP client (own URLSession, URLCache, cookies; scope-captured portal header)

```swift
import Foundation

struct RequestScope: Sendable, Equatable { let accountId: String; let portalId: String? }

final class APIClient: Sendable {
    let baseURL: URL
    private let session: URLSession
    private let tokens: TokenProvider

    init(account: Account, directory: URL, tokens: TokenProvider) {
        let config = URLSessionConfiguration.ephemeral                 // private in-memory cookie jar
        config.urlCache = URLCache(memoryCapacity: 8 << 20, diskCapacity: 64 << 20,
                                   directory: directory.appending(path: "http"))
        config.requestCachePolicy = .useProtocolCachePolicy
        config.httpCookieAcceptPolicy = .always
        config.waitsForConnectivity = true
        self.session = URLSession(configuration: config)
        self.baseURL = account.baseURL
        self.tokens = tokens
    }

    /// Scope is captured when the request is built, never read from a mutable global.
    func send<T: Decodable & Sendable>(_ path: String, scope: RequestScope,
                                       method: String = "GET", body: Data? = nil) async throws -> T {
        var req = URLRequest(url: baseURL.appending(path: path))
        req.httpMethod = method
        req.httpBody = body
        if let pid = scope.portalId { req.setValue(pid, forHTTPHeaderField: "X-Portal-Id") }

        var token = try await tokens.accessToken()
        for attempt in 0..<2 {
            req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
            let (data, resp) = try await session.data(for: req)
            try Task.checkCancellation()
            let code = (resp as? HTTPURLResponse)?.statusCode ?? 0
            switch code {
            case 200..<300: return try JSONDecoder().decode(T.self, from: data)
            case 401 where attempt == 0: token = try await tokens.refresh(stale: token).accessToken
            case 403: throw APIError.forbidden(portalId: scope.portalId)   // → portal revocation flow
            default: throw APIError.http(code)
            }
        }
        throw APIError.http(401)
    }

    func invalidate() { session.invalidateAndCancel() }
}

enum APIError: Error { case http(Int), forbidden(portalId: String?) }
```

### A.5 Per-account SwiftData store and directory

```swift
import Foundation
import SwiftData

@Model final class Item {
    #Unique<Item>([\.portalId, \.serverId])                   // iOS 18; drop on iOS 17
    var serverId: String
    var portalId: String
    var title: String
    var updatedAt: Date
    init(serverId: String, portalId: String, title: String, updatedAt: Date) {
        self.serverId = serverId; self.portalId = portalId; self.title = title; self.updatedAt = updatedAt
    }
}

enum AccountStorage {
    static func directory(for accountId: String) throws -> URL {
        let safe = accountId.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? accountId
        var dir = URL.applicationSupportDirectory.appending(path: "Accounts/\(safe)", directoryHint: .isDirectory)
        try FileManager.default.createDirectory(at: dir, withIntermediateDirectories: true,
            attributes: [.protectionKey: FileProtectionType.completeUntilFirstUserAuthentication])
        var values = URLResourceValues()
        values.isExcludedFromBackup = true                     // re-downloadable; tokens are ThisDeviceOnly anyway
        try dir.setResourceValues(values)
        return dir
    }

    static func makeContainer(in dir: URL) throws -> ModelContainer {
        let config = ModelConfiguration(schema: Schema([Item.self]),
                                        url: dir.appending(path: "data.store"),
                                        cloudKitDatabase: .none)
        return try ModelContainer(for: Item.self, configurations: config)
    }

    static func wipe(accountId: String) throws {
        let dir = try directory(for: accountId)
        try FileManager.default.removeItem(at: dir)           // store + -wal/-shm + http cache + files
    }
}

/// Background-safe writer per account.
@ModelActor actor AccountWriter {
    func upsert(_ dtos: [(id: String, title: String, updatedAt: Date)], portalId: String) throws {
        for dto in dtos {
            let sid = dto.id
            let existing = try modelContext.fetch(FetchDescriptor<Item>(
                predicate: #Predicate { $0.serverId == sid && $0.portalId == portalId })).first
            if let existing { existing.title = dto.title; existing.updatedAt = dto.updatedAt }
            else { modelContext.insert(Item(serverId: sid, portalId: portalId, title: dto.title, updatedAt: dto.updatedAt)) }
        }
        try modelContext.save()
    }
    func purgePortal(_ portalId: String) throws {
        try modelContext.delete(model: Item.self, where: #Predicate { $0.portalId == portalId })
        try modelContext.save()
    }
}
```

Core Data equivalent: `let c = NSPersistentContainer(name: "Model"); c.persistentStoreDescriptions = [NSPersistentStoreDescription(url: dir.appending(path: "data.sqlite"))]`; on wipe call `c.persistentStoreCoordinator.destroyPersistentStore(at:type: .sqlite)` before removing the directory.

### A.6 Scoped session containers

```swift
import Foundation
import Observation
import SwiftData

@MainActor @Observable
final class PortalScope: Identifiable {
    let id = UUID()                       // view identity: new value on every portal switch
    let portal: Portal
    var permissions: Set<String>
    private(set) var tasks: [Task<Void, Never>] = []
    let scope: RequestScope

    init(portal: Portal) {
        self.portal = portal
        self.permissions = portal.permissions
        self.scope = RequestScope(accountId: portal.accountId, portalId: portal.id)
    }

    func track(_ task: Task<Void, Never>) { tasks.append(task) }
    func can(_ permission: String) -> Bool { permissions.contains(permission) }

    func teardown() {
        tasks.forEach { $0.cancel() }
        tasks.removeAll()
        // close realtime socket subscription for this portal here
    }
}

@MainActor @Observable
final class Session: Identifiable {
    let id = UUID()                       // view identity: new value on every account switch
    let account: Account
    let tokens: TokenProvider
    let api: APIClient
    let store: ModelContainer
    let writer: AccountWriter
    private(set) var portal: PortalScope?

    init(account: Account, keychain: KeychainTokenStore, auth: AuthAPI,
         onRevoked: @escaping @Sendable (String) async -> Void) throws {
        let dir = try AccountStorage.directory(for: account.id)
        self.account = account
        self.tokens = TokenProvider(accountId: account.id, baseURL: account.baseURL,
                                    keychain: keychain, api: auth, onRevoked: onRevoked)
        self.api = APIClient(account: account, directory: dir, tokens: tokens)
        self.store = try AccountStorage.makeContainer(in: dir)
        self.writer = AccountWriter(modelContainer: store)
    }

    func enter(portal: Portal) {
        precondition(portal.accountId == account.id, "portal from another account")
        self.portal?.teardown()
        self.portal = PortalScope(portal: portal)
    }

    func teardownPortalOnly() {
        portal?.teardown()
        portal = nil                       // AccountRoot shows NoPortalsView
    }

    func teardown() {
        teardownPortalOnly()
        api.invalidate()
    }

    /// Guard against late responses: only write if the scope is still current.
    func isCurrent(_ scope: RequestScope) -> Bool {
        scope.accountId == account.id && scope.portalId == portal?.portal.id
    }
}
```

### A.7 AppModel: switch account, switch portal, add, sign out

```swift
import Foundation
import Observation
import UIKit
import UserNotifications

protocol PushRegistrar: Sendable {
    func register(deviceToken: Data, for account: Account, api: APIClient) async
    func unregister(for account: Account, api: APIClient) async
}

@MainActor @Observable
final class AppModel {
    enum Phase: Equatable { case launching, signedOut, ready, switching(to: String), needsReauth(String) }

    private(set) var phase: Phase = .launching
    private(set) var session: Session?
    private(set) var accounts: [Account] = []
    var toast: String?

    let registry: AccountRegistry
    let keychain: KeychainTokenStore
    let auth: AuthAPI
    let push: PushRegistrar
    var apnsToken: Data?
    private var switchTask: Task<Void, Never>?

    init(registry: AccountRegistry, keychain: KeychainTokenStore, auth: AuthAPI, push: PushRegistrar) {
        self.registry = registry; self.keychain = keychain; self.auth = auth; self.push = push
    }

    func launch() async {
        accounts = await registry.accounts
        cleanUpKeychainAfterReinstall(store: keychain, registryIsEmpty: accounts.isEmpty)
        if let sel = await registry.active, accounts.contains(where: { $0.id == sel.accountId }) {
            await performSwitch(accountId: sel.accountId, portalId: sel.portalId)
        } else if let first = accounts.max(by: { $0.lastActiveAt < $1.lastActiveAt }) {
            await performSwitch(accountId: first.id, portalId: nil)
        } else {
            phase = .signedOut
        }
    }

    // MARK: Account switch — serialised; a newer request cancels an older one.
    func switchAccount(to accountId: String, portalId: String? = nil) {
        switchTask?.cancel()
        switchTask = Task { await performSwitch(accountId: accountId, portalId: portalId) }
    }

    private func performSwitch(accountId: String, portalId: String?) async {
        guard let account = await registry.account(accountId) else { return }
        phase = .switching(to: accountId)

        session?.teardown()                                   // 1. teardown old fully
        session = nil
        guard !Task.isCancelled else { return }

        do {                                                  // 2. build new
            let new = try Session(account: account, keychain: keychain, auth: auth) { [weak self] id in
                await self?.markNeedsReauth(id)
            }
            let portal = await resolvePortal(account: account, preferred: portalId)
            if let portal { new.enter(portal: portal) }
            try await registry.setActive(.init(accountId: accountId, portalId: portal?.id))
            guard !Task.isCancelled else { new.teardown(); return }
            session = new
            phase = account.status == .needsReauth ? .needsReauth(accountId) : .ready
            onIdentityChanged(account: account, portal: portal)
            refreshPortalsInBackground(for: new)
        } catch {
            phase = .needsReauth(accountId)
        }
        accounts = await registry.accounts
        UIAccessibility.post(notification: .announcement, argument: "Switched to \(account.displayName)")
    }

    // MARK: Portal switch — keeps Session, rebuilds PortalScope only.
    func switchPortal(to portalId: String) async {
        guard let session, let portal = await registry.portals(of: session.account.id)
            .first(where: { $0.id == portalId }) else { return }
        session.enter(portal: portal)
        try? await registry.setActive(.init(accountId: session.account.id, portalId: portalId))
        onIdentityChanged(account: session.account, portal: portal)
    }

    /// 403 membership / push hint "portal_removed".
    func portalRevoked(_ portalId: String, accountId: String) async {
        let remaining = await registry.portals(of: accountId).filter { $0.id != portalId }
        try? await registry.setPortals(remaining, for: accountId)
        if let session, session.account.id == accountId {
            try? await session.writer.purgePortal(portalId)
            if session.portal?.portal.id == portalId {
                toast = "You no longer have access to this workspace"
                if let next = remaining.max(by: { ($0.lastUsedAt ?? .distantPast) < ($1.lastUsedAt ?? .distantPast) }) {
                    await switchPortal(to: next.id)
                } else {
                    session.teardownPortalOnly()              // account empty state
                }
            }
        }
    }

    // MARK: Add account
    func didSignIn(account candidate: Account, tokens: TokenSet, portals: [Portal]) async throws {
        try keychain.save(tokens, for: candidate.id)          // secret first
        let stored: Account, isNew: Bool
        do { (stored, isNew) = try await registry.upsert(candidate) }
        catch { try? keychain.delete(candidate.id); throw error }  // no orphan item
        try await registry.setPortals(portals, for: stored.id)
        if !isNew { toast = "Already signed in — switched to \(stored.displayName)" }
        if let apnsToken, let s = try? Session(account: stored, keychain: keychain, auth: auth, onRevoked: { _ in }) {
            await push.register(deviceToken: apnsToken, for: stored, api: s.api)
            s.teardown()
        }
        switchAccount(to: stored.id)
    }

    // MARK: Sign out
    func signOut(accountId: String) async {
        guard let account = await registry.account(accountId) else { return }
        let active = session.flatMap { $0.account.id == accountId ? $0 : nil }
        let wasActive = active != nil
        let api = active?.api
            ?? (try? Session(account: account, keychain: keychain, auth: auth, onRevoked: { _ in }))?.api
        if let api { await push.unregister(for: account, api: api) }   // best effort, server revoke too
        if wasActive { session?.teardown(); session = nil }

        try? keychain.delete(accountId)
        try? AccountStorage.wipe(accountId: accountId)
        await removeDeliveredNotifications(accountId: accountId)
        let next = try? await registry.remove(accountId)
        accounts = await registry.accounts

        if let next { if wasActive || session == nil { await performSwitch(accountId: next.id, portalId: nil) } }
        else { phase = .signedOut; onIdentityChanged(account: nil, portal: nil) }
        await updateBadge()
    }

    func signOutAll() async {
        for a in accounts { await signOut(accountId: a.id) }
    }

    // MARK: Helpers
    private func markNeedsReauth(_ id: String) async {
        try? await registry.setStatus(.needsReauth, for: id)
        accounts = await registry.accounts
        if session?.account.id == id { phase = .needsReauth(id) }   // others: banner in switcher only
    }

    private func resolvePortal(account: Account, preferred: String?) async -> Portal? {
        let ps = await registry.portals(of: account.id)
        if let preferred, let p = ps.first(where: { $0.id == preferred }) { return p }
        return await registry.defaultPortal(for: account.id)
    }

    private func refreshPortalsInBackground(for session: Session) {
        let task = Task { [registry] in
            struct PortalDTO: Decodable, Sendable { let id, name, role: String; let permissions: [String]; let isDefault: Bool }
            let scope = RequestScope(accountId: session.account.id, portalId: nil)
            guard let dtos: [PortalDTO] = try? await session.api.send("/me/portals", scope: scope) else { return }
            let old = await registry.portals(of: session.account.id)
            let ps = dtos.map { d in Portal(id: d.id, accountId: session.account.id, name: d.name, role: d.role,
                                            permissions: Set(d.permissions),
                                            lastUsedAt: old.first { $0.id == d.id }?.lastUsedAt, isDefault: d.isDefault) }
            try? await registry.setPortals(ps, for: session.account.id)
            if let current = session.portal?.portal.id, !ps.contains(where: { $0.id == current }) {
                await self.portalRevoked(current, accountId: session.account.id)
            }
        }
        session.portal?.track(task)
    }

    private func onIdentityChanged(account: Account?, portal: Portal?) {
        // Analytics.setUserId(account?.id)  — opaque id, never email
        // Analytics.setGroup(portal?.id)
        // Purchases.shared.logIn(account.id) on ACCOUNT change only; logOut() when nil
        // WidgetCenter.shared.reloadAllTimelines()
    }

    private func removeDeliveredNotifications(accountId: String) async {
        let center = UNUserNotificationCenter.current()
        let ids = await center.deliveredNotifications()
            .filter { ($0.request.content.userInfo["account_id"] as? String) == accountId }
            .map(\.request.identifier)
        center.removeDeliveredNotifications(withIdentifiers: ids)
    }

    func updateBadge(unreadByAccount: [String: Int] = [:]) async {
        try? await UNUserNotificationCenter.current().setBadgeCount(unreadByAccount.values.reduce(0, +))
    }
}
```

### A.8 Push and deep-link routing to (account, portal)

```swift
import Foundation

enum RouteTarget: Equatable { case item(String), portalHome }

struct EntryRequest: Equatable {
    let accountId: String?        // nil → resolve by portal
    let portalId: String?
    let target: RouteTarget
    let userInitiatedTap: Bool    // push tap = true; deep link = false → confirm before switching
}

enum EntryDecision: Equatable {
    case navigate(RouteTarget)                                  // already in right scope
    case switchPortal(String, then: RouteTarget)
    case switchAccount(String, portal: String?, then: RouteTarget)
    case confirmSwitch(accountId: String, portal: String?, then: RouteTarget)
    case chooseAccount([String], portal: String, then: RouteTarget)
    case reauth(String)
    case addAccount(pending: EntryRequest)
    case ignore
}

/// Pure function → trivially unit-testable (M20–M26).
func decide(_ req: EntryRequest, active: ActiveSelection?, accounts: [Account],
            portals: [String: [Portal]]) -> EntryDecision {
    let candidates: [Account]
    if let aid = req.accountId {
        candidates = accounts.filter { $0.id == aid }
    } else if let pid = req.portalId {
        candidates = accounts.filter { portals[$0.id]?.contains { $0.id == pid } == true }
    } else {
        return active == nil ? .addAccount(pending: req) : .navigate(req.target)
    }

    guard !candidates.isEmpty else {
        return req.userInitiatedTap ? .ignore : .addAccount(pending: req)   // signed-out account's push → ignore
    }
    if candidates.count > 1 { return .chooseAccount(candidates.map(\.id), portal: req.portalId!, then: req.target) }

    let account = candidates[0]
    if account.status == .needsReauth { return .reauth(account.id) }

    if active?.accountId == account.id {
        if req.portalId == nil || req.portalId == active?.portalId { return .navigate(req.target) }
        guard portals[account.id]?.contains(where: { $0.id == req.portalId }) == true else { return .navigate(.portalHome) }
        return .switchPortal(req.portalId!, then: req.target)
    }
    return req.userInitiatedTap
        ? .switchAccount(account.id, portal: req.portalId, then: req.target)
        : .confirmSwitch(accountId: account.id, portal: req.portalId, then: req.target)
}

extension EntryRequest {
    /// From a notification tap: payload {"account_id": "...", "portal_id": "...", "item_id": "..."}
    init?(push userInfo: [AnyHashable: Any]) {
        guard let aid = userInfo["account_id"] as? String else { return nil }
        self.init(accountId: aid, portalId: userInfo["portal_id"] as? String,
                  target: (userInfo["item_id"] as? String).map(RouteTarget.item) ?? .portalHome,
                  userInitiatedTap: true)
    }
}
```

Unit tests for the router:

```swift
import Testing

@Suite struct EntryRoutingTests {
    let a = Account(id: "A", key: .init(issuer: "eu", userId: "1"), baseURL: URL(string: "https://eu.x")!,
                    displayName: "A", email: "a@x", purchaseUUID: UUID())
    let b = Account(id: "B", key: .init(issuer: "us", userId: "1"), baseURL: URL(string: "https://us.x")!,
                    displayName: "B", email: "b@x", purchaseUUID: UUID())
    var portals: [String: [Portal]] {[
        "A": [Portal(id: "p1", accountId: "A", name: "Acme", role: "admin", permissions: [], isDefault: true),
              Portal(id: "p2", accountId: "A", name: "Beta", role: "viewer", permissions: [], isDefault: false)],
        "B": [Portal(id: "p3", accountId: "B", name: "Gamma", role: "admin", permissions: [], isDefault: true)],
    ]}
    let active = ActiveSelection(accountId: "A", portalId: "p1")

    @Test func pushForActiveScopeNavigates() {
        let r = EntryRequest(accountId: "A", portalId: "p1", target: .item("42"), userInitiatedTap: true)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals) == .navigate(.item("42")))
    }
    @Test func pushForInactivePortalSwitchesPortal() {
        let r = EntryRequest(accountId: "A", portalId: "p2", target: .item("7"), userInitiatedTap: true)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals) == .switchPortal("p2", then: .item("7")))
    }
    @Test func pushForInactiveAccountSwitchesAccount() {
        let r = EntryRequest(accountId: "B", portalId: "p3", target: .portalHome, userInitiatedTap: true)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals)
                == .switchAccount("B", portal: "p3", then: .portalHome))
    }
    @Test func deepLinkToOtherAccountAsksFirst() {
        let r = EntryRequest(accountId: nil, portalId: "p3", target: .item("9"), userInitiatedTap: false)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals)
                == .confirmSwitch(accountId: "B", portal: "p3", then: .item("9")))
    }
    @Test func deepLinkToUnknownPortalOffersAddAccount() {
        let r = EntryRequest(accountId: nil, portalId: "zzz", target: .portalHome, userInitiatedTap: false)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals) == .addAccount(pending: r))
    }
    @Test func pushForSignedOutAccountIgnored() {
        let r = EntryRequest(accountId: "C", portalId: nil, target: .portalHome, userInitiatedTap: true)
        #expect(decide(r, active: active, accounts: [a, b], portals: portals) == .ignore)
    }
}
```

### A.9 Root view wiring (identity resets, privacy overlay)

```swift
import SwiftUI

@main struct MultiAccountApp: App {
    @State private var model = AppModel(
        registry: AccountRegistry(appGroup: "group.com.example.shared"),
        keychain: KeychainTokenStore(service: "com.example.app.auth",
                                     accessGroup: KeychainTokenStore.sharedGroup(suffix: "com.example.shared")),
        auth: LiveAuthAPI(), push: LivePushRegistrar())
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(model)
                .overlay { if scenePhase != .active { PrivacyCover() } }   // app-switcher snapshot
                .task { await model.launch() }
        }
    }
}

struct RootView: View {
    @Environment(AppModel.self) private var model
    var body: some View {
        switch model.phase {
        case .launching: ProgressView()
        case .signedOut: WelcomeView()
        case .switching(let id): SwitchingView(account: model.accounts.first { $0.id == id })
        case .needsReauth(let id): ReauthView(accountId: id)
        case .ready:
            if let session = model.session {
                AccountRoot(session: session)
                    .modelContainer(session.store)       // environment context follows the account
                    .id(session.id)                      // drop ALL @State from previous account
            }
        }
    }
}

struct AccountRoot: View {
    let session: Session
    @State private var path = NavigationPath()
    var body: some View {
        if let portal = session.portal {
            NavigationStack(path: $path) {
                PortalHome(scope: portal)
                    .toolbar { ToolbarItem(placement: .topBarLeading) { AccountSwitcherButton() } }
            }
            .id(portal.id)                               // portal switch resets stack + portal @State
        } else {
            NoPortalsView()                              // join / create / invitations
        }
    }
}

struct PortalHome: View {
    let scope: PortalScope
    @Query private var items: [Item]
    init(scope: PortalScope) {
        self.scope = scope
        let pid = scope.portal.id
        _items = Query(filter: #Predicate<Item> { $0.portalId == pid }, sort: \.updatedAt, order: .reverse)
    }
    var body: some View {
        List(items) { Text($0.title) }
            .navigationTitle(scope.portal.name)
            .toolbar { if scope.can("item.create") { Button("New", systemImage: "plus") {} } }
    }
}
```

`LiveAuthAPI`, `LivePushRegistrar`, `WelcomeView`, `SwitchingView`, `ReauthView`, `NoPortalsView`, `AccountSwitcherButton` and `PrivacyCover` are app-specific. Entitlements to add: **Keychain Sharing** (`$(AppIdentifierPrefix)com.example.shared`, plus your private group first if needed), **App Groups** (`group.com.example.shared`) on app + NSE + widgets + sibling apps, **Push Notifications**, **Background Modes** (remote notifications, background fetch/processing), and `AppIdentifierPrefix = $(AppIdentifierPrefix)` in Info.plist for runtime group resolution.
