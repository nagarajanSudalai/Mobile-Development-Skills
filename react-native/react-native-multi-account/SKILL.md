---
name: react-native-multi-account
description: Use when implementing, reviewing, testing, or debugging multiple signed-in accounts and portal (workspace / organization / team) switching in React Native and Expo apps — account registry, per-account tokens in react-native-keychain or expo-secure-store, single-flight token refresh per account, portal-scoped headers, per-account MMKV instances and SQLite files, TanStack Query cache per session (QueryClient per account+portal or reset), Zustand / Redux store reset, axios / fetch clients per account, React Navigation / Expo Router reset on switch, add account, sign out one vs all, push and deep links routed to the right account+portal, background tasks, RevenueCat / IAP identity, analytics and Sentry user context, app lock and data wipe. Siblings: ios-multi-account, android-multi-account, flutter-multi-account; related: react-native-deep-links, react-native-push-notifications, react-native-in-app-purchase, react-native-revenuecat-iap, react-native-list-ux.
---

# React Native Multi-Account & Portal Switching Skill

Guide for building, auditing, and testing apps where **several user identities are signed in on one device** (accounts) and **each account can belong to several portals** — workspaces, organizations, teams, stores, clinics — between which the user switches. Think Slack (workspaces), Gmail (accounts), or a B2C marketplace app where one login can manage a personal profile and two shops. Applies to **bare React Native** and **Expo** (managed with config plugins, dev client, or prebuild). Siblings: **ios-multi-account**, **android-multi-account**, **flutter-multi-account**. Related: **react-native-deep-links**, **react-native-push-notifications**, **react-native-in-app-purchase**, **react-native-revenuecat-iap**, **react-native-list-ux**.

Two layers, always modelled separately:

1. **Account** — an authenticated identity (credentials, refresh token, backend region/environment). Switching accounts is a **hard boundary**: nothing from account A may be visible, cached in memory, or sent over the network while B is active.
2. **Portal** — a tenant inside an account. Switching portals is a **soft boundary**: the account token and account-level data stay; portal-scoped data, caches, and navigation reset.

These libraries move fast. **Verify fast-moving APIs against the current docs and the installed versions in `package.json` before writing code.** Known version gates:

- **react-native-mmkv** v3 uses `new MMKV({ id, encryptionKey })` and `storage.delete(key)`; **v4** (Nitro Modules, New Architecture only) uses `createMMKV({ id, encryptionKey })` and `storage.remove(key)`. Check which one is installed.
- **react-native-keychain** v9+/v10: options objects (`{ service, accessible }`); older versions accepted a bare service string. `getAllGenericPasswordServices()` exists in recent versions — verify before relying on it.
- **expo-secure-store**: keys may only contain alphanumerics, `.`, `-`, `_`; values are strings; large values (historically ~2 KB) are discouraged — store tokens, not blobs. `requireAuthentication` items become unreadable if biometrics change.
- **TanStack Query** v5 (`gcTime`, not `cacheTime`; `@tanstack/query-async-storage-persister` / `query-sync-storage-persister`).
- **React Navigation** v7 / **Expo Router** v4+/v5 (`router.replace`, `router.dismissAll`, `Redirect`, `Stack.Protected` in newer Expo Router — verify).
- **expo-notifications**, **react-native-purchases** (RevenueCat), **@sentry/react-native** — identity APIs below are stable but check signatures.
- Expo SDK 52+ defaults to the New Architecture; RN 0.82+ cannot opt out.

## 0. Mental model

```
 ┌──────────────────────────── App process ─────────────────────────────────────┐
 │  AccountRegistry (MMKV "registry", no secrets)                               │
 │    accounts[]: {id, userId, email, name, avatar, region, env, addedAt,       │
 │                 lastUsedAt, lastPortalId, state: active|needsReauth}         │
 │    activeAccountId · order · maxAccounts                                     │
 │  TokenStore (Keychain / SecureStore, service = "auth.<accountId>")           │
 │    {accessToken, refreshToken, expiresAt} per account                        │
 │                                                                              │
 │  SessionManager ── active (accountId, portalId) ──► SessionProvider key=...  │
 │     │                                                                        │
 │     ▼ builds one Session per account (portal is a child scope)              │
 │  ┌─ AccountSession(accountId) ────────────────────────────────────────────┐  │
 │  │  apiClient (baseURL by region, auth interceptor, single-flight refresh)│  │
 │  │  MMKV "acct.<id>" (encrypted) · SQLite "acct_<id>.db" · files dir      │  │
 │  │  portals[] (id, name, role, lastUsedAt) · account-level queries        │  │
 │  │  ┌─ PortalSession(portalId) ─────────────────────────────────────────┐ │  │
 │  │  │ QueryClient (persister key = q.<acct>.<portal>) · Zustand store   │ │  │
 │  │  │ X-Portal-Id header · socket channel · navigation state            │ │  │
 │  │  └───────────────────────────────────────────────────────────────────┘ │  │
 │  └────────────────────────────────────────────────────────────────────────┘  │
 │  Global (account-agnostic): theme, locale, onboarding seen, push permission  │
 └──────────────────────────────────────────────────────────────────────────────┘
 Push payload {accountId, portalId, route} ──► Router ──► confirm? ──► switch ──► navigate
```

Rule of thumb: **every piece of state has exactly one owner — Global, Account, or Portal — and is created and destroyed with that owner.** Module-level singletons (`export const api = axios.create()`, `export const queryClient = new QueryClient()`, `export const useStore = create(...)`) are the main source of cross-account leaks.

## 1. First, establish context

Find out from the codebase, or ask:

- **Max accounts** on one device (1, 3, 5, unlimited?) and why (Slack-like: ~10+; consumer: 2–5). What happens at the limit?
- **Backends per account**: one global API, or region/data-residency shards (EU/US), or environments (prod/staging for internal users)? Is the base URL discovered at login?
- **Auth**: OAuth/OIDC (PKCE via `expo-auth-session` / `react-native-app-auth`), password + refresh token, magic link, social login? Is there a shared browser session (SSO cookies) that makes adding a second account log in as the first?
- **Portals per account**: 1, a handful, or hundreds (agencies, admins)? Does the token carry the portal (portal-scoped token, re-issued per portal) or is the portal sent as a header/path segment (`X-Portal-Id`, `/portals/:id/...`)?
- **Roles/permissions** per portal (owner/admin/member/guest) and what they gate in the UI.
- **Shared vs isolated data**: is anything legitimately shared across accounts (downloaded media, public catalog, app settings)? Across portals of one account (profile, notification prefs)?
- **Unified views?** Is there a "all accounts" inbox / combined badge, or is only one account visible at a time? (Unified views multiply complexity — confirm the product really needs it.)
- **Push**: Expo push service or FCM/APNs directly (`@react-native-firebase/messaging`, Notifee)? Does the server know which device tokens belong to which accounts?
- **Purchases**: are subscriptions per account, per portal, or per device? RevenueCat or raw StoreKit/Play Billing?
- **Stack**: Expo managed/dev client/bare; RN version; New Architecture; state library (Zustand, Redux Toolkit, Jotai, MobX); server cache (TanStack Query, RTK Query, SWR, Apollo); HTTP client (axios, fetch, ky); storage (MMKV, AsyncStorage, SQLite/WatermelonDB/op-sqlite/Drizzle); navigation (React Navigation vs Expo Router).
- **Security requirements**: app lock / biometrics per account, MDM, shared devices (family tablet, kiosk), regulated data (health, finance).

Write a **scope table** before coding:

```
state                         owner     storage                         reset on
access/refresh token          account   keychain service auth.<id>      sign-out / revoke
account profile, portal list  account   MMKV acct.<id> + query cache    account switch (unmount)
feed, inbox, portal settings  portal    QueryClient q.<id>.<portal>     portal switch
draft messages                portal    MMKV acct.<id> key draft.<pid>  never (restore on return)
theme, locale                 global    MMKV global                     never
last screen                   portal    MMKV acct.<id> nav.<pid>        restore on switch back
```

## 2. Data model

- [ ] **Account registry** stored outside secure storage (it is read on every launch and needs listing): `id` (local UUID, not the server user id — lets you add the same user on two environments), `userId` (server), `email`/`displayName`/`avatarUrl` (cached for the switcher, refreshed on use), `region`/`baseUrl`/`environment`, `addedAt`, `lastUsedAt`, `lastPortalId`, `state` (`active` | `needsReauth` | `removing`), and a `tokenRef` (the secure-store key/service name).
- [ ] **No secrets in the registry.** It may live in an unencrypted MMKV instance or AsyncStorage; tokens never do.
- [ ] **Portal list per account**: `id`, `name`, `slug`, `iconUrl`, `role`, `lastUsedAt`, `unreadCount?`, `status` (`active` | `suspended` | `pendingInvite`). Fetched from the server (source of truth), cached per account for fast start.
- [ ] **Active pair** `(activeAccountId, activePortalId)` persisted atomically. Persist `lastPortalId` **per account** so switching back to A lands in A's last portal, not B's.
- [ ] **Ordering**: explicit `order[]` (user-reorderable) or by `lastUsedAt`. Decide once; the switcher, push routing fallback, and "next account after sign-out" all use it.
- [ ] **Limits**: enforce `maxAccounts` before starting a login flow, not after (do not throw away a successful login).
- [ ] **Schema version** on the registry and a migration from the single-account layout (existing users upgrading must keep their session: move `token` → `auth.<newId>`, the default MMKV → `acct.<newId>`).
- [ ] **Validate on load**: drop registry entries whose token is missing (e.g. Keychain wiped), mark `needsReauth` instead of crashing; fix an `activeAccountId` that points nowhere.

```ts
type AccountId = string & { __brand: 'AccountId' };
type PortalId  = string & { __brand: 'PortalId' };
type SessionKey = `${AccountId}:${PortalId}`;
```

Brand the ids so a portal id can't be passed where an account id is expected.

## 3. Secure storage of tokens per account

- [ ] **One secure item per account**: react-native-keychain `service: 'auth.<accountId>'`, or expo-secure-store key `auth.<accountId>` (allowed charset only). Store a small JSON `{accessToken, refreshToken, expiresAt, tokenType}`.
- [ ] **Accessibility**: `ACCESSIBLE.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` (keychain) / `SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` — background refresh and push handling need the token while locked; `THIS_DEVICE_ONLY` keeps it out of iCloud Keychain and encrypted backups.
- [ ] **Never** tokens in AsyncStorage, plain MMKV, Redux-persist, Zustand `persist`, TanStack persisted cache, or logs/Sentry breadcrumbs.
- [ ] **Biometric-gated tokens** (`accessControl: BIOMETRY_CURRENT_SET` / `requireAuthentication: true`) only if the product wants a prompt on every read — they break silent refresh in background tasks. Prefer a separate app-lock gate (§15) and keep the refresh token readable after first unlock.
- [ ] **Cache in memory** after the first read; Keychain reads are async and can take tens of ms, and on Android the first Keystore access after boot can be slow. Don't read on every request.
- [ ] **iOS Keychain survives uninstall.** On first launch after install (flag in MMKV/AsyncStorage, which *is* wiped), delete all `auth.*` items — otherwise a reinstalled app silently signs back into accounts. With expo-secure-store you can't enumerate, so keep the known keys in the registry *and* also keep a list of ever-used keys in a secure index item (`auth.index`).
- [ ] **Android backup**: set `android:allowBackup="false"` (Expo: `"android": { "allowBackup": false }` in app config) or provide `dataExtractionRules`/`fullBackupContent` excluding MMKV, SQLite, and shared prefs. Keystore keys don't restore, so restored encrypted data is garbage and must be detected and wiped.
- [ ] **iOS backup**: MMKV and SQLite files under Documents/Library are backed up by default. Exclude per-account dirs (`NSURLIsExcludedFromBackupKey`, via a small native module or a library) or keep them in Caches/tmp if they're rebuildable. Verify where your MMKV version stores files.
- [ ] **MMKV encryption key per account** generated randomly (`expo-crypto` `getRandomBytes` / `react-native-get-random-values`) and stored in the secure store (`mmkvkey.<accountId>`), not hard-coded.
- [ ] **Write order on login**: secure token first, then registry entry, then flip active. On failure, roll back so you never have a registry entry without a token or an orphan token.

## 4. Per-account token lifecycle

- [ ] **Each account has its own refresh state**: a per-account promise map `refreshing: Map<AccountId, Promise<Tokens>>`. Concurrent 401s for account A share one refresh; a refresh of A never blocks B.
- [ ] **Bind requests to the account at creation time**, not at send time: the axios instance / fetch wrapper closes over `accountId`. A request started for A that retries after a switch to B must still use A's token — or, better, be cancelled by the switch.
- [ ] **Proactive refresh** shortly before `expiresAt` (skew 60 s) for the active account only; inactive accounts refresh lazily (when used, or when a background task/push needs them).
- [ ] **Refresh-token rotation**: write the new refresh token to secure storage **before** resolving the refresh promise; if the app dies between server response and write, the next refresh fails — handle `invalid_grant` by marking `needsReauth`, not by signing out all accounts.
- [ ] **One account revoked, others fine**: on `invalid_grant`/401-after-refresh for A, set A `needsReauth`, cancel A's queries, show a banner ("Sign in again to Acme") and keep A in the switcher with a badge. If A is active, offer "Sign in again" or switch to the next valid account. Never wipe B.
- [ ] **Portal-scoped auth**: either (a) one account token + `X-Portal-Id` header (or `/portals/:id` path) added by a portal-scoped client, or (b) a portal-scoped access token exchanged from the account token (`POST /portals/:id/token`) and cached in memory per `(account, portal)`. With (b), the portal token refresh is single-flight per `SessionKey`.
- [ ] **Never read "the current portal" from a global at send time** inside the interceptor — a request queued in portal P1 but sent after switching to P2 would hit P2 with P1's payload. Capture `portalId` when building the client.
- [ ] **Clock skew**: trust the server's `expires_in` relative to receive time, not the JWT `exp` vs device clock.
- [ ] **Re-login for `needsReauth`** must reuse the same local `accountId` (and its data) if the server `userId` matches; if a different user logs in, treat it as a new account.

## 5. Data isolation

- [ ] **MMKV instance per account**: `id: 'acct.<accountId>'`, encrypted; portal-scoped keys prefixed `p.<portalId>.` inside it, or a separate instance per portal if portal data is large. One small `global` instance for account-agnostic settings and the registry.
- [ ] **SQLite file per account** (`acct_<accountId>.db` with expo-sqlite / op-sqlite / WatermelonDB adapter `dbName`). Portal isolation: `portal_id` column + every query filtered (enforced in the repository layer), or file per portal when portals are large and independently deletable. Close the DB on account switch.
- [ ] **Files**: `FileSystem.documentDirectory + 'accounts/<accountId>/'` (and `cacheDirectory + 'accounts/<accountId>/'` for rebuildable data). Deleting an account = deleting its folder.
- [ ] **Image cache**: expo-image / FastImage disk caches are global and keyed by URL. Authenticated images (header tokens) cached by URL can be shown to another account if URLs collide. Use per-account signed URLs, or add the account to the cache key (`cacheKey` in expo-image `source`), and call `Image.clearDiskCache()` / `clearMemoryCache()` on account removal (it clears all — acceptable; it's a cache).
- [ ] **HTTP caches**: OkHttp/NSURLCache under fetch may cache authenticated GETs; send `Cache-Control: no-store` from the server for private data, or disable the cache for API calls.
- [ ] **Cookies**: RN `fetch` shares a cookie jar (NSHTTPCookieStorage / OkHttp CookieJar). If the backend uses cookies, cookies from A ride along on B's requests. Use `credentials: 'omit'` and bearer tokens, or clear cookies on switch (`@react-native-cookies/cookies`). Same for in-app browser/WebView sessions (`expo-web-browser` with `preferEphemeralSession: true` on iOS when adding an account).
- [ ] **TanStack Query**: one `QueryClient` per portal session (created in a provider keyed by `SessionKey`), or include `accountId` and `portalId` in **every** query key. The former is safer; the latter is needed if you keep a single client. Persisted cache: one persister key per session (`q.<accountId>.<portalId>`) written to the account's MMKV.
- [ ] **Zustand / Redux / Jotai**: store factory per session (`createPortalStore()` inside a provider) or a reset action that restores the initial state for every slice. Redux: root reducer that returns `undefined`-initialized state on `session/reset`; redux-persist `key: 'root.<accountId>'` with an MMKV storage adapter.
- [ ] **Singletons audit**: grep for `axios.create(`, `new QueryClient(`, `create(` (zustand) at module scope, `let cachedUser`, `export const socket`, analytics `identify` cached values, `global.` assignments. Each must be session-owned or proven account-agnostic.
- [ ] **Search indices, Spotlight / App Shortcuts, widgets, Siri intents** contain account data — scope them and clear on removal.

## 6. Account switch

Order matters. Implement as an explicit, awaitable state machine: `idle → switching(from, to) → ready | error`.

- [ ] **1. Gate**: block switching during unsaved edits (prompt "Discard draft?") or save drafts per portal automatically.
- [ ] **2. Show a switching UI** immediately (full-screen overlay with the target avatar/name; ≤ 300 ms ideally, never blank).
- [ ] **3. Teardown old session**:
  - `AbortController.abort()` for the account's in-flight requests (axios `signal` / fetch `signal`), `queryClient.cancelQueries()`
  - close websockets / SSE / realtime subscriptions; stop polling intervals
  - flush and pause persisters (`persistQueryClientSave` then unsubscribe) so A's cache is written to A's storage, not B's
  - close SQLite handle; unmount the session subtree (which drops in-memory Zustand/Query state)
  - stop media playback, uploads (or hand them to a background queue scoped to A)
- [ ] **4. Persist** `activeAccountId = B` and update `lastUsedAt` — only after teardown, so a crash mid-switch doesn't resume half-torn.
- [ ] **5. Rebuild**: create B's api client (base URL by region), open B's MMKV/SQLite, create the QueryClient and hydrate from B's persister, create stores. In React, the cleanest way is to **key the session subtree**: `<SessionProvider key={sessionKey}>` — React unmounts everything under A and mounts fresh for B.
- [ ] **6. Identity side effects**: analytics/Sentry user, RevenueCat `logIn`, push routing context (§10, §13, §14). Await the ones that gate UI (entitlements), fire-and-forget the rest.
- [ ] **7. Navigation reset**: replace the stack — never push B's home over A's screens.
  - React Navigation: `navigationRef.reset({ index: 0, routes: [{ name: 'Main' }] })` or key the `NavigationContainer`/root navigator by `sessionKey`
  - Expo Router: `router.dismissAll()` then `router.replace('/(app)/home')`, or a root layout that renders `<Stack key={sessionKey}>`
- [ ] **8. Restore last screen per account+portal** if the product wants it: persist a *safe* route (tab + top-level screen, not a deep modal), validate it still exists, fall back to home.
- [ ] **9. Error path**: if B fails to build (token gone, needsReauth), show B's re-auth screen; offer to go back to A. Never leave the app on an empty shell.
- [ ] **Switcher UI**: avatar + name + email (+ environment badge for staging); active account checkmark; `needsReauth` warning; unread badges per account if available; "Add account"; "Manage accounts". Reachable in ≤ 2 taps (long-press on the profile tab is a common shortcut). Accessible labels: "Switch to Jane Doe, jane@example.com, 3 unread".
- [ ] **Haptic + toast** after switch ("Now using Acme as jane@…") to make the context change obvious.

## 7. Portal switch

Lighter than an account switch: **account client, token, account-level queries, and MMKV instance stay; portal session is rebuilt.**

- [ ] **Reset**: portal QueryClient (or `removeQueries({ queryKey: ['portal', oldPortalId] })` with a shared client), portal Zustand store, portal socket channel, portal-scoped navigation stack. **Keep**: account profile, portal list, account notification settings, global settings.
- [ ] **Persist** `lastPortalId` on the account and `lastUsedAt` on the portal.
- [ ] **Portal picker UX**: current portal first, then recent (last 3–5 by `lastUsedAt`), then all alphabetically. **Search** once > ~8 portals; show role and unread badge; sticky "Create / Join portal" if allowed. For hundreds of portals, page the list from the server (see react-native-list-ux).
- [ ] **Default portal on login**: server-suggested default → last used on this device → only portal → picker if several and no hint. Zero portals: an empty state with "Create" / "Ask for an invite", not a crash.
- [ ] **Portal removed / access revoked mid-session**: a 403/404 with a specific error code on a portal-scoped request (or a realtime "membership removed" event) → cancel the portal session, refetch the portal list, show "You no longer have access to Acme", switch to the next portal (or picker). Drop `lastPortalId` if it was this one. Remove cached data for that portal.
- [ ] **Role change**: refetch permissions on portal focus and on a realtime event; UI derives capabilities from `role`, never from cached screens. A downgraded user editing an admin screen gets a clear error and is navigated out.
- [ ] **Invitations**: pending invites shown in the picker with Accept/Decline; accepting adds to the list and optionally switches. An invite link for an account not on the device → login/add-account flow with the invite preserved (§8, §11).
- [ ] **Portal-scoped header/token** updated by rebuilding the portal client, not by mutating a shared header (`axios.defaults.headers['X-Portal-Id'] = ...` is a race).

## 8. Add account

- [ ] **Login while signed in**: open the auth flow as a modal over the current session. On cancel, return to A untouched. On success, create a new `AccountId`, store tokens, add to registry, then switch.
- [ ] **Fresh browser session**: OAuth in a shared system browser will auto-log in as the existing user. Use `prompt=login` / `prompt=select_account`, `preferEphemeralSession: true` (iOS `ASWebAuthenticationSession` ephemeral), and `login_hint` when re-authing a specific account.
- [ ] **Duplicate detection**: after login, fetch `/me`; if `(userId, environment)` already exists, update that account's tokens instead of adding a duplicate, switch to it, and tell the user ("You're already signed in as jane@…").
- [ ] **Different region/environment**: base URL chosen before login (region picker, email-domain discovery, or returned by the auth server); stored on the account. Every client for that account uses it.
- [ ] **Limit reached**: disable "Add account" with an explanation before starting the flow.
- [ ] **Onboarding** for the new account (profile setup, portal choice) runs inside the new session; global onboarding (app tour) doesn't repeat.

## 9. Sign out

- [ ] **Sign out of this account** vs **Sign out of all accounts** — both explicit; "Remove account from this device" wording helps users understand data is deleted locally.
- [ ] **Server revoke** (refresh token revoke, push token unregister for this account) best-effort with a short timeout; local cleanup happens regardless (queue the revoke for retry if offline).
- [ ] **Cleanup order**: teardown session if active → unregister push for the account → RevenueCat/analytics reset if it was active → delete secure items (`auth.<id>`, `mmkvkey.<id>`) → `mmkv.clearAll()` and delete MMKV file (v3+ has no file delete API on all versions — `clearAll()` + drop references; verify) → delete SQLite file → delete `accounts/<id>/` dirs → clear image caches → remove registry entry.
- [ ] **Next active account**: most recently used remaining (or first in `order`), landing in its `lastPortalId`. If none left → logged-out root (welcome/login), with navigation reset.
- [ ] **Sign out all**: same per account, then global reset (but keep global non-personal settings like theme if the product wants).
- [ ] **Forced sign-out** (server says account deleted/disabled): same cleanup, with a clear message.

## 10. Push notifications per account/portal

See **react-native-push-notifications** for setup.

- [ ] **One device token, many accounts**: the device push token (Expo push token or FCM/APNs token) is the same for all accounts. Register it with the backend **for every signed-in account** (each account's own auth), and re-register all on token refresh (`addPushTokenListener` / `messaging().onTokenRefresh`).
- [ ] **Payload carries routing**: `{ accountUserId | accountHint, portalId, route, notificationId }` in data. Map server `userId` (+ environment) → local `AccountId` via the registry. Don't put PII in the payload.
- [ ] **Unknown account in payload** (removed from device but server still sends): drop silently and unregister for that account asynchronously.
- [ ] **Foreground push for inactive account/portal**: show it (banner via `setNotificationHandler` returning `shouldShowBanner: true` / Notifee) but label with account/portal ("Acme · jane@…"); don't refresh the active portal's data from it.
- [ ] **Tap routing** (`addNotificationResponseReceivedListener` + `getLastNotificationResponseAsync()` for cold start, or Notifee/RNFirebase `getInitialNotification`): resolve `(account, portal)` → if different from active, switch (no confirmation needed — the user tapped it — but show the switching overlay) → navigate to `route` after the new session is ready. Queue the route until navigation is ready.
- [ ] **Badges**: app icon badge = sum across accounts (or active only — decide). Per-account/portal unread counts in the switcher/picker come from the server (`/unread` per account) — the app can't compute them from pushes reliably.
- [ ] **Notification grouping**: Android `channelId`/group per account (`thread-id` on iOS) so notifications from different accounts don't stack together.
- [ ] **Unregister on removal**: `DELETE /devices/:token` with the removed account's auth before deleting its tokens.
- [ ] **Background data pushes** for an inactive account must use that account's client built headlessly (§12), not the active session.

## 11. Deep links per account/portal

See **react-native-deep-links** for link setup and routing.

- [ ] **Resolve target**: links contain a portal (`/p/:portalSlug/...`) and sometimes an account hint (`?u=<userIdHash>`). Find candidate accounts that are members of the portal (from cached portal lists).
- [ ] **Exactly one match, active**: navigate. **One match, inactive**: confirm ("Open in Acme as jane@…?") then switch + navigate. **Several matches**: account chooser. **None**: "You don't have access" with "Add account" / "Request access"; preserve the link through login.
- [ ] **Link before session ready** (cold start): store as pending, resume after registry load + switch.
- [ ] **No destructive actions from links**; after switching, still re-check permissions for the target screen in the new portal.

## 12. Background work scoped per account/portal

- [ ] **Headless JS / `expo-task-manager` / `expo-background-task` / Notifee background events / RNFirebase `setBackgroundMessageHandler`** run without the React tree. They must build a **headless session** from the registry + secure store for the specific account in the payload, not import the UI's singletons.
- [ ] **Periodic sync** iterates accounts (bounded, sequential, time-budgeted — iOS gives ~30 s); skip `needsReauth` accounts; write to each account's own DB.
- [ ] **Uploads/outbox** queued with `accountId` + `portalId` on each job; the worker uses that account's client. On account removal, cancel and delete its jobs.
- [ ] **Locks**: background refresh and foreground refresh for the same account share the same single-flight promise when in the same JS runtime; across runtimes (Android headless may be a separate context), make refresh idempotent and tolerate rotation races (retry once by re-reading the secure store).

## 13. Purchases & entitlements

See **react-native-revenuecat-iap** / **react-native-in-app-purchase**.

- [ ] **Decide ownership**: subscription belongs to the account (most common), to a portal (B2B seat billing — usually handled on the web/server, not IAP), or to the store account/device.
- [ ] **RevenueCat**: `await Purchases.logIn(serverUserId)` on account switch; `logOut()` only when signing out the last account (logOut creates an anonymous user). Refresh `customerInfo` after switch and gate UI on it. Watch for "transfer" behavior: one Apple ID buying for two app accounts — configure the project's restore/transfer behavior deliberately.
- [ ] **Raw IAP**: send `appAccountToken` (iOS, UUID per account) / `obfuscatedAccountId` (Android) with the purchase so the server attributes it to the right account; validate on the server.
- [ ] **Purchase in progress during switch**: block switching while a purchase sheet is open; finish/acknowledge transactions under the account that started them.
- [ ] **Entitlements cache per account** (never show A's "Pro" badge while B is active).

## 14. Analytics & crash reporting

- [ ] **On switch**: `Sentry.setUser({ id: hashedOrServerUserId })` (no email), `Sentry.setTag('portal_id', portalId)`, `setContext('session', { accountIndex, accountCount, role })`. Analytics: `identify(serverUserId)`/`reset()` per vendor semantics; group/portal via `group(portalId)` or a super property.
- [ ] **No PII** in events, tags, breadcrumbs, or navigation route params logged by integrations (React Navigation instrumentation logs route names — fine — but not params with emails).
- [ ] **Events emitted during teardown** must not be attributed to the new account: flush the analytics queue before switching identity.
- [ ] **Track** `account_added`, `account_switched`, `portal_switched`, `account_removed`, `reauth_required` with counts, not identities.

## 15. Security & privacy

- [ ] **App lock per account** (optional): `expo-local-authentication` / `react-native-biometrics` prompt when entering a locked account or after N minutes in background. Lock state per account in memory; the switcher can show locked accounts without content.
- [ ] **App-switcher snapshot**: on `AppState` `inactive`/`background`, cover sensitive screens with a privacy view (iOS); on Android use `FLAG_SECURE` (`expo-screen-capture` `preventScreenCaptureAsync()` or a native flag) for sensitive screens. Especially important right after a switch — the snapshot can show the previous account.
- [ ] **Account deletion** (server-side, store requirement on both stores): full local wipe of that account (§9) plus server deletion; other accounts untouched.
- [ ] **Shared devices**: "Remove account from this device" must be discoverable; consider not caching avatars/names of other accounts on the lock-free switcher if the threat model includes household members.
- [ ] **Logs**: redact tokens and `Authorization` headers in any network logger (Flipper/Reactotron/axios logger) — and disable them in release.
- [ ] **Jailbreak/root & screenshots** policies are product decisions; document them.

## 16. Testing

### 16.1 Tools

1. **Jest + React Native Testing Library** with in-memory fakes for the secure store (`jest.mock('react-native-keychain')` / `jest.mock('expo-secure-store')`) and MMKV (v3 ships a Jest mock automatically in test env; v4 — verify), a fresh `QueryClient` per test.
2. **MSW** (`msw/native`) or nock for per-account API responses; assert the `Authorization` and `X-Portal-Id` headers on every request.
3. **Unit-test the registry, token store, refresh coordinator, and router** as plain TypeScript — they have no UI.
4. **E2E**: Maestro or Detox with two seeded test users and a user in multiple portals; scripted push via `xcrun simctl push` (iOS) / `adb shell` + FCM test send (Android) / Expo push tool.
5. **Manual**: network throttling, airplane mode during switch, kill app mid-switch, reinstall (iOS Keychain persistence), backup/restore on Android.

### 16.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| M1 | Add second account | Fresh login (no SSO auto-login), new session, switcher shows both |
| M2 | Add same account again | Detected; tokens updated; no duplicate entry |
| M3 | Add account in another region | Its requests go to its base URL; A's still to A's |
| M4 | Switch A → B | Overlay; no A data visible at any frame; B lands in its last portal; back button doesn't reach A's screens |
| M5 | Switch mid-request (slow A request) | A request aborted; response never written into B's cache/store |
| M6 | Switch during unsaved draft | Prompt or draft saved per portal; restored on return |
| M7 | Kill app mid-switch | Relaunch into a consistent account (A or B), never a mix |
| M8 | Portal switch P1 → P2 | Portal data reset; account profile not refetched; header = P2 |
| M9 | Portal revoked while viewing it | Message; moved to another portal; P1 cache removed |
| M10 | Role downgraded mid-session | Admin UI disappears; actions fail gracefully |
| M11 | Zero portals after login | Empty state with create/request |
| M12 | 200 portals | Searchable picker, recent first, smooth scroll |
| M13 | Access token expiry on A, many parallel requests | One refresh call; all requests retried once |
| M14 | Refresh token revoked on inactive B | B marked needsReauth; A unaffected; badge in switcher |
| M15 | Refresh token revoked on active A | Banner/re-auth; other accounts kept; can switch away |
| M16 | Push for inactive account, tapped (warm) | Switch to that account+portal, open the route |
| M17 | Push for inactive account, tapped (cold start) | Registry loaded, switch, route opened once |
| M18 | Push for inactive portal of active account | Portal switch, route opened |
| M19 | Push for removed account | Ignored; server unregister attempted |
| M20 | Foreground push for other account | Banner labelled with account; active data untouched |
| M21 | Deep link to portal of inactive account | Confirmation, then switch + navigate |
| M22 | Deep link to portal no account has | No-access screen; "Add account" preserves link |
| M23 | Sign out active (2 accounts) | Moves to the other; removed account's files/MMKV/keychain gone |
| M24 | Sign out last account | Logged-out root; RevenueCat logOut; push unregistered |
| M25 | Sign out offline | Local wipe immediate; revoke retried later |
| M26 | Reinstall app (iOS) | No auto sign-in from leftover Keychain items |
| M27 | Android backup restore to new device | No crash; accounts require login; stale data wiped |
| M28 | Background sync with 3 accounts | Each synced into its own DB with its own token |
| M29 | Upgrade from single-account version | Existing session migrated; no re-login |
| M30 | Purchase on A, switch to B | B has no entitlement unless purchased/transferred by design |
| M31 | App switcher after switching A → B | Snapshot never shows A |
| M32 | Max accounts reached | Add disabled with explanation |
| M33 | VoiceOver/TalkBack in switcher and picker | Names, email, active state, unread announced |

### 16.3 Example tests

```ts
it('aborts in-flight requests of the old account on switch', async () => {
  server.use(http.get('*/feed', async () => { await delay(500); return HttpResponse.json(feedA); }));
  const { manager } = await setupTwoAccounts();
  const p = manager.current!.api.get('/feed');      // A's request
  await manager.switchAccount(B);
  await expect(p).rejects.toMatchObject({ name: 'CanceledError' });
  expect(manager.current!.accountId).toBe(B);
});

it('single-flights refresh per account and isolates accounts', async () => {
  const refresh = jest.fn(async (id: AccountId) => tokensFor(id));
  const coord = new RefreshCoordinator(refresh);
  await Promise.all([coord.refresh(A), coord.refresh(A), coord.refresh(B)]);
  expect(refresh).toHaveBeenCalledTimes(2);
  expect(refresh).toHaveBeenCalledWith(A);
  expect(refresh).toHaveBeenCalledWith(B);
});

it('routes a push for an inactive portal to that account+portal', async () => {
  const { manager, router } = await setupTwoAccounts({ active: [A, P1] });
  await router.handle({ userId: 'u-b', portalId: 'p9', route: '/threads/42' }, { userInitiated: true });
  expect(manager.activeKey).toBe(`${B}:p9`);
  expect(navigate).toHaveBeenCalledWith('/threads/42');
});
```

## 17. Common bugs to look for in review

- `export const api = axios.create(...)` with an interceptor reading `store.getState().token` → requests from A sent with B's token after a switch (or vice versa)
- `axios.defaults.headers.common.Authorization = ...` / mutating a shared `X-Portal-Id` header → races during switch
- One global `QueryClient` and query keys without account/portal → B sees A's cached feed for a frame (or forever with `staleTime: Infinity`)
- `queryClient.clear()` called but persisted cache restores A's data into B on next launch (single persister key)
- Zustand `persist` with one `name` for all accounts; Redux-persist key `root` shared
- Refresh lock is one global boolean → refresh of B waits on A, or B's 401 is retried with A's new token
- `invalid_grant` on one account → "sign out" path that wipes every account
- Tokens in AsyncStorage / Redux state persisted to disk; tokens in Sentry breadcrumbs via logged request config
- Keychain items not deleted on sign-out; iOS reinstall silently restores sessions
- Navigation not reset → back gesture returns to A's screens after switching to B
- Switch implemented as `setActiveAccount(B)` without teardown → sockets for A keep pushing events into the UI
- `lastPortalId` stored globally → switching to B opens B on a portal id from A (404)
- Push tap handled before registry hydration (cold start) → routed to the default account or dropped
- Push token registered only for the active account → inactive accounts get no notifications
- Shared system-browser cookie → "Add account" logs in as the existing user and creates a duplicate
- Image disk cache keyed by URL with auth header → private avatars/attachments of A shown in B
- `RevenueCat.logOut()` on every switch → anonymous ids pile up, entitlements flicker
- Background task imports the UI singleton session → syncs the wrong account
- Removing the active account leaves `activeAccountId` dangling → crash/blank on next launch
- Portal revoked → infinite 403 retry loop (TanStack `retry` not disabled for 403)

## 18. Output format

When asked to **review/audit**:
1. **Summary**: stack (Expo/bare, RN, state lib, query lib, HTTP client, storage, navigation, push, IAP), account/portal model, and the scope table (§1) as found
2. **Checklist results**: sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check, with file:line
3. **Critical issues first**: cross-account data leaks, tokens in insecure storage, wrong-token requests, sign-out that leaves data, push routed to wrong account — each with fix code
4. **Testing gaps**: M1–M33 not covered, plus suggested tests

When asked to **implement**: confirm versions (MMKV v3 vs v4, keychain vs SecureStore, Query v5, navigation library), write the scope table, start from the Appendix, and adapt: auth endpoints, region discovery, portal header vs portal token, push payload shape. Migrate single-account users first. Then list what to verify by hand: reinstall on iOS, Android backup restore, cold-start push for an inactive account, switching on a slow network.

When asked to **debug**: identify the owner (global/account/portal) of the leaking or missing state, find where it's created and whether it's destroyed with its owner, then reproduce with the closest M-scenario. Log `(accountId, portalId)` on every request and cache write during a switch.

---

## Appendix — Reference implementation (TypeScript, Expo SDK 53+ / bare RN 0.79+)

Assumes `react-native-mmkv` v3 (`new MMKV`; for v4 replace with `createMMKV` and `.delete` → `.remove`), `react-native-keychain` v9+ **or** `expo-secure-store` (both adapters shown), `axios` 1.x, `@tanstack/react-query` v5 + `@tanstack/query-sync-storage-persister` + `@tanstack/react-query-persist-client`, `zustand` v5, React Navigation v7 (Expo Router notes inline), `expo-notifications`, `react-native-purchases`, `@sentry/react-native`. Verify each API against the installed versions.

### A.1 Types & storage

```ts
// session/types.ts
export type AccountId = string & { __brand: 'AccountId' };
export type PortalId = string & { __brand: 'PortalId' };
export type SessionKey = `${string}:${string}`;
export const sessionKey = (a: AccountId, p: PortalId | null): SessionKey => `${a}:${p ?? '-'}`;

export type AccountState = 'active' | 'needsReauth' | 'removing';

export interface AccountRecord {
  id: AccountId;              // local id
  userId: string;             // server id
  environment: 'prod' | 'staging';
  baseUrl: string;            // region-specific
  email: string;
  displayName: string;
  avatarUrl?: string;
  addedAt: number;
  lastUsedAt: number;
  lastPortalId: PortalId | null;
  state: AccountState;
}

export interface Portal {
  id: PortalId;
  name: string;
  slug: string;
  role: 'owner' | 'admin' | 'member' | 'guest';
  iconUrl?: string;
  lastUsedAt?: number;
  status: 'active' | 'suspended' | 'pendingInvite';
}

export interface Tokens {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;          // epoch ms, computed from expires_in at receive time
}

export class ReauthRequiredError extends Error {
  constructor(public accountId: AccountId) { super(`Re-auth required for ${accountId}`); this.name = 'ReauthRequiredError'; }
}
export class PortalAccessRevokedError extends Error {
  constructor(public portalId: PortalId) { super(`No access to ${portalId}`); this.name = 'PortalAccessRevokedError'; }
}
```

```ts
// session/storage.ts
import { MMKV } from 'react-native-mmkv';
import * as Crypto from 'expo-crypto';
import type { AccountId } from './types';
import { secureStore } from './secureStore';

/** Account-agnostic, unencrypted: registry, theme, install flag. No secrets. */
export const globalStorage = new MMKV({ id: 'global' });

const accountStores = new Map<AccountId, MMKV>();

export async function openAccountStorage(id: AccountId): Promise<MMKV> {
  const cached = accountStores.get(id);
  if (cached) return cached;
  let key = await secureStore.get(`mmkvkey.${id}`);
  if (!key) {
    key = Array.from(Crypto.getRandomBytes(16), b => b.toString(16).padStart(2, '0')).join(''); // MMKV key max 16 bytes
    await secureStore.set(`mmkvkey.${id}`, key);
  }
  const store = new MMKV({ id: `acct.${id}`, encryptionKey: key.slice(0, 16) });
  accountStores.set(id, store);
  return store;
}

export async function destroyAccountStorage(id: AccountId) {
  const store = await openAccountStorage(id);
  store.clearAll();
  accountStores.delete(id);
  await secureStore.remove(`mmkvkey.${id}`);
}
```

### A.2 Secure token store (react-native-keychain or expo-secure-store)

```ts
// session/secureStore.ts
import * as Keychain from 'react-native-keychain';
import * as SecureStore from 'expo-secure-store';

export interface SecureKV {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  remove(key: string): Promise<void>;
}

/** react-native-keychain: one generic-password item per key (service). */
export const keychainKV: SecureKV = {
  async get(key) {
    const res = await Keychain.getGenericPassword({ service: key });
    return res ? res.password : null;
  },
  async set(key, value) {
    await Keychain.setGenericPassword('v', value, {
      service: key,
      accessible: Keychain.ACCESSIBLE.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
    });
  },
  async remove(key) {
    await Keychain.resetGenericPassword({ service: key });
  },
};

/** expo-secure-store: keys limited to [A-Za-z0-9._-]. */
const opts: SecureStore.SecureStoreOptions = {
  keychainAccessible: SecureStore.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
};
export const expoKV: SecureKV = {
  get: key => SecureStore.getItemAsync(key, opts),
  set: (key, value) => SecureStore.setItemAsync(key, value, opts),
  remove: key => SecureStore.deleteItemAsync(key, opts),
};

// Pick one for the app.
export const secureStore: SecureKV = keychainKV;
```

```ts
// session/tokenStore.ts
import type { AccountId, Tokens } from './types';
import { secureStore } from './secureStore';

const INDEX_KEY = 'auth.index';

export class TokenStore {
  private memory = new Map<AccountId, Tokens>();

  private key(id: AccountId) { return `auth.${id}`; }

  async get(id: AccountId): Promise<Tokens | null> {
    const hit = this.memory.get(id);
    if (hit) return hit;
    const raw = await secureStore.get(this.key(id));
    if (!raw) return null;
    const tokens = JSON.parse(raw) as Tokens;
    this.memory.set(id, tokens);
    return tokens;
  }

  async set(id: AccountId, tokens: Tokens) {
    await secureStore.set(this.key(id), JSON.stringify(tokens)); // persist first
    this.memory.set(id, tokens);
    await this.addToIndex(id);
  }

  async remove(id: AccountId) {
    this.memory.delete(id);
    await secureStore.remove(this.key(id));
    const ids = (await this.index()).filter(x => x !== id);
    await secureStore.set(INDEX_KEY, JSON.stringify(ids));
  }

  /** Every account id that ever stored a token — used to wipe after iOS reinstall. */
  async index(): Promise<AccountId[]> {
    const raw = await secureStore.get(INDEX_KEY);
    return raw ? (JSON.parse(raw) as AccountId[]) : [];
  }

  private async addToIndex(id: AccountId) {
    const ids = await this.index();
    if (!ids.includes(id)) await secureStore.set(INDEX_KEY, JSON.stringify([...ids, id]));
  }

  async wipeAll() {
    for (const id of await this.index()) {
      await secureStore.remove(this.key(id));
      await secureStore.remove(`mmkvkey.${id}`);
    }
    await secureStore.remove(INDEX_KEY);
    this.memory.clear();
  }
}

export const tokenStore = new TokenStore();
```

### A.3 Account registry

```ts
// session/accountRegistry.ts
import { globalStorage } from './storage';
import { tokenStore } from './tokenStore';
import type { AccountId, AccountRecord, PortalId } from './types';

const KEY = 'registry.v2';
const INSTALL_FLAG = 'installed';
export const MAX_ACCOUNTS = 5;

interface RegistryData {
  version: 2;
  accounts: AccountRecord[];
  activeAccountId: AccountId | null;
}

type Listener = (d: RegistryData) => void;

export class AccountRegistry {
  private data: RegistryData = { version: 2, accounts: [], activeAccountId: null };
  private listeners = new Set<Listener>();

  async load() {
    // iOS keeps Keychain items across uninstall; MMKV does not survive. Wipe leftovers.
    if (!globalStorage.getBoolean(INSTALL_FLAG)) {
      await tokenStore.wipeAll();
      globalStorage.set(INSTALL_FLAG, true);
    }
    const raw = globalStorage.getString(KEY);
    if (raw) this.data = JSON.parse(raw) as RegistryData;
    // TODO: migrate from the single-account layout (registry.v1 / legacy token key) here.

    // Validate: entries without tokens need re-auth; active must exist.
    for (const a of this.data.accounts) {
      if (a.state === 'removing') continue;
      if (!(await tokenStore.get(a.id))) a.state = 'needsReauth';
    }
    this.data.accounts = this.data.accounts.filter(a => a.state !== 'removing');
    if (!this.data.accounts.some(a => a.id === this.data.activeAccountId)) {
      this.data.activeAccountId = this.ordered()[0]?.id ?? null;
    }
    this.save();
  }

  get accounts(): readonly AccountRecord[] { return this.data.accounts; }
  get activeAccountId() { return this.data.activeAccountId; }
  get canAdd() { return this.data.accounts.length < MAX_ACCOUNTS; }

  ordered(): AccountRecord[] {
    return [...this.data.accounts].sort((a, b) => b.lastUsedAt - a.lastUsedAt);
  }
  find(id: AccountId) { return this.data.accounts.find(a => a.id === id); }
  findByServerUser(userId: string, env: AccountRecord['environment']) {
    return this.data.accounts.find(a => a.userId === userId && a.environment === env);
  }

  upsert(record: AccountRecord) {
    const i = this.data.accounts.findIndex(a => a.id === record.id);
    if (i >= 0) this.data.accounts[i] = record; else this.data.accounts.push(record);
    this.save();
  }
  patch(id: AccountId, p: Partial<AccountRecord>) {
    const a = this.find(id);
    if (a) { Object.assign(a, p); this.save(); }
  }
  setActive(id: AccountId | null, portalId?: PortalId | null) {
    this.data.activeAccountId = id;
    if (id) this.patch(id, { lastUsedAt: Date.now(), ...(portalId !== undefined && { lastPortalId: portalId }) });
    this.save();
  }
  remove(id: AccountId) {
    this.data.accounts = this.data.accounts.filter(a => a.id !== id);
    if (this.data.activeAccountId === id) this.data.activeAccountId = null;
    this.save();
  }

  subscribe(l: Listener) { this.listeners.add(l); return () => this.listeners.delete(l); }
  private save() {
    globalStorage.set(KEY, JSON.stringify(this.data));
    this.listeners.forEach(l => l(this.data));
  }
}

export const registry = new AccountRegistry();
```

### A.4 Per-account API client with single-flight refresh

```ts
// session/refresh.ts
import type { AccountId, Tokens } from './types';

/** One in-flight refresh per account; accounts never wait on each other. */
export class RefreshCoordinator {
  private inflight = new Map<AccountId, Promise<Tokens>>();
  constructor(private doRefresh: (id: AccountId) => Promise<Tokens>) {}

  refresh(id: AccountId): Promise<Tokens> {
    const existing = this.inflight.get(id);
    if (existing) return existing;
    const p = this.doRefresh(id).finally(() => this.inflight.delete(id));
    this.inflight.set(id, p);
    return p;
  }
}
```

```ts
// session/apiClient.ts
import axios, { AxiosError, AxiosInstance, InternalAxiosRequestConfig } from 'axios';
import { tokenStore } from './tokenStore';
import { registry } from './accountRegistry';
import { RefreshCoordinator } from './refresh';
import { ReauthRequiredError, PortalAccessRevokedError, type AccountId, type PortalId, type Tokens } from './types';

const SKEW_MS = 60_000;

async function refreshTokens(id: AccountId): Promise<Tokens> {
  const account = registry.find(id);
  const current = await tokenStore.get(id);
  if (!account || !current) throw new ReauthRequiredError(id);
  try {
    // Plain axios: must not go through the authed interceptor.
    const { data } = await axios.post(`${account.baseUrl}/oauth/token`, {
      grant_type: 'refresh_token',
      refresh_token: current.refreshToken,
    }, { timeout: 15_000 });
    const next: Tokens = {
      accessToken: data.access_token,
      refreshToken: data.refresh_token ?? current.refreshToken, // rotation
      expiresAt: Date.now() + data.expires_in * 1000,
    };
    await tokenStore.set(id, next); // persist before anyone uses it
    return next;
  } catch (e) {
    const err = e as AxiosError<{ error?: string }>;
    if (err.response?.status === 400 || err.response?.status === 401) {
      registry.patch(id, { state: 'needsReauth' }); // only this account
      throw new ReauthRequiredError(id);
    }
    throw e; // network etc.: keep the account, retry later
  }
}

export const refreshCoordinator = new RefreshCoordinator(refreshTokens);

type RetriableConfig = InternalAxiosRequestConfig & { _retried?: boolean };

/**
 * Account-bound client. accountId (and optional portalId) are captured here,
 * never read from a global at send time.
 */
export function createApiClient(opts: {
  accountId: AccountId;
  portalId?: PortalId;
  signal: AbortSignal; // aborted on session teardown
}): AxiosInstance {
  const { accountId, portalId, signal } = opts;
  const account = registry.find(accountId);
  if (!account) throw new ReauthRequiredError(accountId);

  const client = axios.create({
    baseURL: account.baseUrl,
    timeout: 30_000,
    withCredentials: false, // bearer only; don't ride the shared cookie jar
  });

  client.interceptors.request.use(async config => {
    if (signal.aborted) throw new axios.CanceledError('session closed');
    config.signal ??= signal;
    let tokens = await tokenStore.get(accountId);
    if (!tokens) throw new ReauthRequiredError(accountId);
    if (tokens.expiresAt - SKEW_MS < Date.now()) tokens = await refreshCoordinator.refresh(accountId);
    config.headers.set('Authorization', `Bearer ${tokens.accessToken}`);
    if (portalId) config.headers.set('X-Portal-Id', portalId);
    return config;
  });

  client.interceptors.response.use(undefined, async (error: AxiosError<{ code?: string }>) => {
    const config = error.config as RetriableConfig | undefined;
    const status = error.response?.status;
    if (status === 401 && config && !config._retried && !signal.aborted) {
      config._retried = true;
      const tokens = await refreshCoordinator.refresh(accountId);
      config.headers.set('Authorization', `Bearer ${tokens.accessToken}`);
      return client.request(config);
    }
    if (portalId && (status === 403 || status === 404) && error.response?.data?.code === 'portal_access_revoked') {
      throw new PortalAccessRevokedError(portalId);
    }
    throw error;
  });

  return client;
}
```

For `fetch` instead of axios, the same shape works: `createFetch({ accountId, portalId, signal })` returns `(path, init) => fetch(baseUrl + path, { ...init, signal: AbortSignal.any?.([signal, init.signal]) ?? signal, credentials: 'omit', headers })` with the same refresh-and-retry-once logic. (`AbortSignal.any` availability depends on the Hermes/RN version — verify, or combine signals manually.)

### A.5 Session objects (account scope + portal scope)

```ts
// session/session.ts
import { QueryClient } from '@tanstack/react-query';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import type { MMKV } from 'react-native-mmkv';
import { createStore, type StoreApi } from 'zustand/vanilla';
import type { AxiosInstance } from 'axios';
import { createApiClient } from './apiClient';
import { openAccountStorage } from './storage';
import { PortalAccessRevokedError, type AccountId, type Portal, type PortalId } from './types';

export class AccountSession {
  readonly abort = new AbortController();
  api!: AxiosInstance;            // account-level (no portal header)
  storage!: MMKV;
  portals: Portal[] = [];
  private closed = false;

  private constructor(readonly accountId: AccountId) {}

  static async open(accountId: AccountId): Promise<AccountSession> {
    const s = new AccountSession(accountId);
    s.storage = await openAccountStorage(accountId);
    s.api = createApiClient({ accountId, signal: s.abort.signal });
    const cached = s.storage.getString('portals');
    if (cached) s.portals = JSON.parse(cached);
    return s;
  }

  async refreshPortals(): Promise<Portal[]> {
    const { data } = await this.api.get<Portal[]>('/me/portals');
    if (this.closed) return this.portals;
    this.portals = data;
    this.storage.set('portals', JSON.stringify(data));
    return data;
  }

  defaultPortal(lastPortalId: PortalId | null): PortalId | null {
    const active = this.portals.filter(p => p.status === 'active');
    return active.find(p => p.id === lastPortalId)?.id
      ?? [...active].sort((a, b) => (b.lastUsedAt ?? 0) - (a.lastUsedAt ?? 0))[0]?.id
      ?? null;
  }

  close() {
    this.closed = true;
    this.abort.abort(); // cancels every request created by this account's clients
  }
}

// Portal-scoped client state (example; one store per portal session)
export interface PortalUiState {
  selectedThreadId: string | null;
  composerDraft: string;
  setSelectedThread(id: string | null): void;
  setDraft(t: string): void;
}

export class PortalSession {
  readonly abort = new AbortController();
  readonly api: AxiosInstance;
  readonly queryClient: QueryClient;
  readonly store: StoreApi<PortalUiState>;
  readonly persister;
  private sockets: Array<{ close(): void }> = [];

  constructor(readonly account: AccountSession, readonly portalId: PortalId, onRevoked: (p: PortalId) => void) {
    // Abort when either the portal or the whole account session closes.
    account.abort.signal.addEventListener('abort', () => this.abort.abort());
    this.api = createApiClient({ accountId: account.accountId, portalId, signal: this.abort.signal });

    this.queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 30_000,
          gcTime: 24 * 60 * 60_000,
          retry: (count, err) =>
            !(err instanceof PortalAccessRevokedError) && (err as any)?.response?.status !== 403 && count < 2,
        },
      },
    });
    this.queryClient.getQueryCache().subscribe(e => {
      if (e.type === 'updated' && e.query.state.error instanceof PortalAccessRevokedError) onRevoked(portalId);
    });

    const draftKey = `p.${portalId}.draft`;
    const storage = account.storage;
    this.store = createStore<PortalUiState>(set => ({
      selectedThreadId: null,
      composerDraft: storage.getString(draftKey) ?? '',
      setSelectedThread: id => set({ selectedThreadId: id }),
      setDraft: t => { storage.set(draftKey, t); set({ composerDraft: t }); },
    }));

    // Persisted query cache lives in the account's encrypted MMKV, keyed per portal.
    this.persister = createSyncStoragePersister({
      key: `q.${portalId}`,
      storage: {
        getItem: k => storage.getString(k) ?? null,
        setItem: (k, v) => storage.set(k, v),
        removeItem: k => storage.delete(k), // v4: storage.remove(k)
      },
      throttleTime: 1000,
    });
  }

  addSocket(s: { close(): void }) { this.sockets.push(s); }

  async close() {
    this.abort.abort();
    await this.queryClient.cancelQueries();
    this.sockets.forEach(s => s.close());
    this.sockets = [];
    this.queryClient.clear(); // in-memory only; persisted copy stays for fast return
  }

  async destroyPersisted() {
    await this.persister.removeClient();
  }
}
```

### A.6 SessionManager — switch account, switch portal, add, sign out

```ts
// session/sessionManager.ts
import Purchases from 'react-native-purchases';
import * as Sentry from '@sentry/react-native';
import * as FileSystem from 'expo-file-system/legacy'; // SDK 54+: legacy path or new File/Directory API — verify
import { Image } from 'expo-image';
import { registry } from './accountRegistry';
import { tokenStore } from './tokenStore';
import { destroyAccountStorage } from './storage';
import { AccountSession, PortalSession } from './session';
import { pushRegistry } from './pushRegistry';
import { sessionKey, type AccountId, type AccountRecord, type PortalId, type SessionKey, type Tokens } from './types';

export type SwitchState =
  | { kind: 'idle' }
  | { kind: 'switching'; to: AccountId; portal?: PortalId | null }
  | { kind: 'needsReauth'; accountId: AccountId }
  | { kind: 'noPortal'; accountId: AccountId }
  | { kind: 'loggedOut' };

type Listener = () => void;

export class SessionManager {
  account: AccountSession | null = null;
  portal: PortalSession | null = null;
  state: SwitchState = { kind: 'idle' };
  private queue: Promise<unknown> = Promise.resolve();
  private listeners = new Set<Listener>();

  get activeKey(): SessionKey | null {
    return this.account ? sessionKey(this.account.accountId, this.portal?.portalId ?? null) : null;
  }

  /** Serialize all transitions: a second tap during a switch waits for the first. */
  private run<T>(fn: () => Promise<T>): Promise<T> {
    const next = this.queue.then(fn, fn);
    this.queue = next.catch(() => undefined);
    return next;
  }

  async start() {
    await registry.load();
    const id = registry.activeAccountId;
    if (!id) return this.set({ kind: 'loggedOut' });
    await this.switchAccount(id);
  }

  switchAccount(to: AccountId, portal?: PortalId | null): Promise<void> {
    return this.run(async () => {
      const target = registry.find(to);
      if (!target) throw new Error('unknown account');
      this.set({ kind: 'switching', to, portal });

      // 1) teardown old (portal first, then account)
      await this.flushPersist();
      await this.portal?.close();
      this.account?.close();
      this.portal = null;
      this.account = null;

      if (target.state === 'needsReauth') {
        registry.setActive(to);
        return this.set({ kind: 'needsReauth', accountId: to });
      }

      // 2) rebuild account scope
      const account = await AccountSession.open(to);
      this.account = account;
      try { await account.refreshPortals(); } catch { /* offline: use cached list */ }

      // 3) portal scope
      const portalId = portal ?? account.defaultPortal(target.lastPortalId);
      registry.setActive(to, portalId);
      await this.identify(target, portalId);
      if (!portalId) return this.set({ kind: 'noPortal', accountId: to });
      this.portal = this.makePortal(account, portalId);
      this.set({ kind: 'idle' });
    });
  }

  switchPortal(to: PortalId): Promise<void> {
    return this.run(async () => {
      const account = this.account;
      if (!account || this.portal?.portalId === to) return;
      this.set({ kind: 'switching', to: account.accountId, portal: to });
      await this.flushPersist();
      await this.portal?.close();
      this.portal = this.makePortal(account, to);
      registry.patch(account.accountId, { lastPortalId: to });
      Sentry.setTag('portal_id', to);
      this.set({ kind: 'idle' });
    });
  }

  private makePortal(account: AccountSession, portalId: PortalId) {
    return new PortalSession(account, portalId, revoked => void this.handlePortalRevoked(revoked));
  }

  private async handlePortalRevoked(portalId: PortalId) {
    const account = this.account;
    if (!account || this.portal?.portalId !== portalId) return;
    await this.portal.destroyPersisted();
    try { await account.refreshPortals(); } catch {}
    const next = account.defaultPortal(null);
    // UI shows "You no longer have access to …" via the state change.
    if (next && next !== portalId) await this.switchPortal(next);
    else this.run(async () => { await this.portal?.close(); this.portal = null; this.set({ kind: 'noPortal', accountId: account.accountId }); });
  }

  /** Called after a successful login flow (modal over the current session). */
  async addAccount(input: { userId: string; environment: AccountRecord['environment']; baseUrl: string;
    email: string; displayName: string; avatarUrl?: string; tokens: Tokens }): Promise<{ duplicate: boolean }> {
    const existing = registry.findByServerUser(input.userId, input.environment);
    if (!existing && !registry.canAdd) throw new Error('max_accounts'); // UI should prevent reaching here
    const id = existing?.id ?? (globalThis.crypto?.randomUUID?.() ?? `${Date.now()}-${Math.random()}`) as AccountId;

    await tokenStore.set(id, input.tokens); // secure first
    registry.upsert({
      id, userId: input.userId, environment: input.environment, baseUrl: input.baseUrl,
      email: input.email, displayName: input.displayName, avatarUrl: input.avatarUrl,
      addedAt: existing?.addedAt ?? Date.now(), lastUsedAt: Date.now(),
      lastPortalId: existing?.lastPortalId ?? null, state: 'active',
    });
    await pushRegistry.registerAccount(id).catch(() => {}); // retried on next launch
    await this.switchAccount(id);
    return { duplicate: !!existing };
  }

  signOut(id: AccountId): Promise<void> {
    return this.run(async () => {
      const wasActive = this.account?.accountId === id;
      registry.patch(id, { state: 'removing' }); // crash-safe: load() finishes removal

      if (wasActive) {
        await this.portal?.close();
        this.account?.close();
        this.portal = null;
        this.account = null;
      }
      await Promise.race([
        Promise.allSettled([pushRegistry.unregisterAccount(id), this.revokeOnServer(id)]),
        new Promise(r => setTimeout(r, 3000)),
      ]);

      await tokenStore.remove(id);
      await destroyAccountStorage(id);
      await FileSystem.deleteAsync(`${FileSystem.documentDirectory}accounts/${id}/`, { idempotent: true });
      await FileSystem.deleteAsync(`${FileSystem.cacheDirectory}accounts/${id}/`, { idempotent: true });
      await Image.clearDiskCache();
      await Image.clearMemoryCache();
      registry.remove(id);

      const next = registry.ordered().find(a => a.state === 'active');
      if (!wasActive) return this.notify();
      if (next) {
        // re-enter queue after this task finishes
        void this.switchAccount(next.id);
      } else {
        registry.setActive(null);
        await Purchases.logOut().catch(() => {});
        Sentry.setUser(null);
        this.set({ kind: 'loggedOut' });
      }
    });
  }

  async signOutAll() {
    for (const a of [...registry.accounts]) await this.signOut(a.id);
  }

  private async revokeOnServer(id: AccountId) {
    // Best effort; queue for retry when offline (omitted).
    const tokens = await tokenStore.get(id);
    const account = registry.find(id);
    if (!tokens || !account) return;
    await fetch(`${account.baseUrl}/oauth/revoke`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token: tokens.refreshToken }),
      credentials: 'omit',
    });
  }

  private async identify(account: AccountRecord, portalId: PortalId | null) {
    Sentry.setUser({ id: account.userId });
    Sentry.setTag('portal_id', portalId ?? 'none');
    Sentry.setTag('environment_account', account.environment);
    try {
      await Purchases.logIn(account.userId); // never logOut on switch
    } catch { /* entitlements re-fetched later */ }
  }

  private async flushPersist() {
    // persistQueryClientSave is the explicit API; the provider also persists on change.
    const { persistQueryClientSave } = await import('@tanstack/react-query-persist-client');
    if (this.portal) {
      await persistQueryClientSave({ queryClient: this.portal.queryClient, persister: this.portal.persister }).catch(() => {});
    }
  }

  subscribe(l: Listener) { this.listeners.add(l); return () => { this.listeners.delete(l); }; }
  private set(s: SwitchState) { this.state = s; this.notify(); }
  private notify() { this.listeners.forEach(l => l()); }
}

export const sessionManager = new SessionManager();
```

### A.7 React wiring — keyed providers and navigation reset

```tsx
// session/SessionProvider.tsx
import React, { createContext, useContext, useSyncExternalStore } from 'react';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { useStore } from 'zustand';
import { sessionManager } from './sessionManager';
import type { PortalSession, PortalUiState } from './session';

const PortalCtx = createContext<PortalSession | null>(null);

export function useSessionState() {
  return useSyncExternalStore(
    l => sessionManager.subscribe(l),
    () => sessionManager.state,
  );
}
export function useActiveKey() {
  return useSyncExternalStore(l => sessionManager.subscribe(l), () => sessionManager.activeKey);
}

export function usePortal(): PortalSession {
  const p = useContext(PortalCtx);
  if (!p) throw new Error('usePortal outside a portal session');
  return p;
}
export function usePortalStore<T>(selector: (s: PortalUiState) => T): T {
  return useStore(usePortal().store, selector);
}
export const usePortalApi = () => usePortal().api;

/**
 * Everything below is remounted when (account, portal) changes:
 * in-memory React state, query observers, navigation state.
 */
export function PortalSessionBoundary({ children }: { children: React.ReactNode }) {
  useActiveKey(); // re-render on change
  const portal = sessionManager.portal;
  if (!portal) return null;
  const key = `${portal.account.accountId}:${portal.portalId}`;
  return (
    <PortalCtx.Provider value={portal} key={key}>
      <PersistQueryClientProvider
        client={portal.queryClient}
        persistOptions={{ persister: portal.persister, maxAge: 24 * 60 * 60_000, buster: APP_VERSION }}
      >
        {children}
      </PersistQueryClientProvider>
    </PortalCtx.Provider>
  );
}
declare const APP_VERSION: string;
```

Query keys can stay portal-relative (`['threads', filters]`) because each portal session has its own `QueryClient`. If you keep one shared client instead, prefix every key: `['a', accountId, 'p', portalId, 'threads', filters]` and remove with `queryClient.removeQueries({ queryKey: ['a', accountId] })` on switch.

```tsx
// App.tsx (React Navigation v7)
import React, { useEffect } from 'react';
import { NavigationContainer, createNavigationContainerRef } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { sessionManager } from './session/sessionManager';
import { PortalSessionBoundary, useSessionState, useActiveKey } from './session/SessionProvider';
import { linkRouter } from './session/linkRouter';

export const navigationRef = createNavigationContainerRef<RootParams>();
const Root = createNativeStackNavigator<RootParams>();

export default function App() {
  const state = useSessionState();
  const key = useActiveKey();

  useEffect(() => { void sessionManager.start(); }, []);

  if (state.kind === 'switching') return <SwitchingOverlay accountId={state.to} />;
  if (state.kind === 'loggedOut') return <LoggedOutNavigator />;
  if (state.kind === 'needsReauth') return <ReauthScreen accountId={state.accountId} />;
  if (state.kind === 'noPortal') return <NoPortalScreen accountId={state.accountId} />;

  return (
    <PortalSessionBoundary>
      {/* key → fresh navigation state per session: back never reaches the previous account */}
      <NavigationContainer
        key={key ?? 'none'}
        ref={navigationRef}
        initialState={restoreNavState(key)}
        onStateChange={s => saveNavState(key, s)}
        onReady={() => linkRouter.flushPending()}
      >
        <Root.Navigator>
          <Root.Screen name="Main" component={MainTabs} options={{ headerShown: false }} />
          <Root.Screen name="Thread" component={ThreadScreen} />
          <Root.Screen name="AccountSwitcher" component={AccountSwitcher} options={{ presentation: 'modal' }} />
        </Root.Navigator>
      </NavigationContainer>
    </PortalSessionBoundary>
  );
}
```

Keying the whole `NavigationContainer` is the simplest reset. If you need a single container (e.g. a logged-out flow animating into the app), keep it and call `navigationRef.reset({ index: 0, routes: [{ name: 'Main' }] })` after each switch instead. **Expo Router**: the container is implicit — render `<Stack key={key} />` in `app/(app)/_layout.tsx` inside `PortalSessionBoundary`, and after a switch call `router.dismissAll()` then `router.replace('/(app)')`; guard the group with `<Redirect href="/login" />` (or `Stack.Protected` where available) when `state.kind === 'loggedOut'`.

`restoreNavState`/`saveNavState` persist only a safe top-level route (tab index + root screen) per `SessionKey` in the account's MMKV; validate the restored state before using it and fall back to `undefined`.

### A.8 Push registration and routing to account + portal

```ts
// session/pushRegistry.ts
import * as Notifications from 'expo-notifications';
import { registry } from './accountRegistry';
import { createApiClient } from './apiClient';
import type { AccountId } from './types';

let deviceToken: string | null = null;

async function getToken() {
  deviceToken ??= (await Notifications.getDevicePushTokenAsync()).data as string; // or getExpoPushTokenAsync({ projectId })
  return deviceToken;
}

/** Headless-safe: builds a short-lived client for the given account. */
function clientFor(id: AccountId) {
  const ac = new AbortController();
  setTimeout(() => ac.abort(), 15_000);
  return createApiClient({ accountId: id, signal: ac.signal });
}

export const pushRegistry = {
  async registerAccount(id: AccountId) {
    const token = await getToken();
    await clientFor(id).put('/devices/me', { token, platform: process.env.EXPO_OS });
  },
  async unregisterAccount(id: AccountId) {
    if (!deviceToken) return;
    await clientFor(id).delete(`/devices/${encodeURIComponent(deviceToken)}`);
  },
  async registerAll() {
    for (const a of registry.accounts) {
      if (a.state === 'active') await this.registerAccount(a.id).catch(() => {});
    }
  },
  listenForTokenChanges() {
    return Notifications.addPushTokenListener(t => { deviceToken = t.data as string; void this.registerAll(); });
  },
};
```

```ts
// session/linkRouter.ts
import * as Notifications from 'expo-notifications';
import { Alert } from 'react-native';
import { registry } from './accountRegistry';
import { sessionManager } from './sessionManager';
import { openAccountStorage } from './storage';
import type { AccountId, Portal, PortalId } from './types';

export interface RouteTarget {
  userId?: string;          // from push (server user id)
  portalId?: string;        // or portalSlug from a link
  portalSlug?: string;
  route: string;            // app path, already validated by the link parser
}

type Navigate = (route: string) => void;

class LinkRouter {
  private pending: RouteTarget | null = null;
  private navigate: Navigate | null = null;
  private ready = false;

  attach(navigate: Navigate) { this.navigate = navigate; }
  flushPending() {
    this.ready = true;
    const p = this.pending; this.pending = null;
    if (p) void this.handle(p, { userInitiated: true });
  }

  /** Accounts on this device that can open the target. */
  private async candidates(t: RouteTarget): Promise<Array<{ accountId: AccountId; portalId: PortalId | null }>> {
    const out: Array<{ accountId: AccountId; portalId: PortalId | null }> = [];
    for (const a of registry.accounts) {
      if (a.state !== 'active') continue;
      if (t.userId && a.userId !== t.userId) continue;
      if (!t.portalId && !t.portalSlug) { out.push({ accountId: a.id, portalId: null }); continue; }
      const storage = await openAccountStorage(a.id);
      const portals: Portal[] = JSON.parse(storage.getString('portals') ?? '[]');
      const p = portals.find(x => x.id === t.portalId || x.slug === t.portalSlug);
      if (p) out.push({ accountId: a.id, portalId: p.id });
    }
    return out;
  }

  async handle(t: RouteTarget, opts: { userInitiated: boolean; fromPush?: boolean }) {
    if (!this.ready || !this.navigate) { this.pending = t; return; } // cold start
    const matches = await this.candidates(t);
    if (matches.length === 0) return this.navigate('/no-access'); // offer add account; keep t
    const activeAcct = sessionManager.account?.accountId;
    const activePortal = sessionManager.portal?.portalId ?? null;
    const exact = matches.find(m => m.accountId === activeAcct && (m.portalId === null || m.portalId === activePortal));
    if (exact) return this.navigate(t.route);

    const pick = matches.length === 1 ? matches[0] : await chooseAccount(matches); // chooser UI
    if (!pick) return;
    const needsConfirm = !opts.fromPush; // a push tap is explicit intent; a link may not be
    if (needsConfirm && !(await confirmSwitch(pick.accountId))) return;

    if (pick.accountId !== activeAcct) await sessionManager.switchAccount(pick.accountId, pick.portalId);
    else if (pick.portalId && pick.portalId !== activePortal) await sessionManager.switchPortal(pick.portalId);
    // New NavigationContainer mounts → onReady → flushPending
    this.ready = false;
    this.pending = { ...t, userId: undefined };
  }
}

export const linkRouter = new LinkRouter();

function confirmSwitch(accountId: AccountId): Promise<boolean> {
  const a = registry.find(accountId);
  return new Promise(resolve =>
    Alert.alert('Switch account?', `Open this in ${a?.displayName} (${a?.email})?`, [
      { text: 'Cancel', style: 'cancel', onPress: () => resolve(false) },
      { text: 'Switch', onPress: () => resolve(true) },
    ]),
  );
}
declare function chooseAccount(m: Array<{ accountId: AccountId; portalId: PortalId | null }>):
  Promise<{ accountId: AccountId; portalId: PortalId | null } | null>;

/** Wire once at startup (outside React render). */
export function installNotificationRouting() {
  const toTarget = (r: Notifications.NotificationResponse): RouteTarget | null => {
    const d = r.notification.request.content.data as Record<string, unknown>;
    if (typeof d?.route !== 'string') return null;
    return { userId: d.userId as string | undefined, portalId: d.portalId as string | undefined, route: d.route };
  };
  // Cold start: tap that launched the app
  void Notifications.getLastNotificationResponseAsync().then(r => {
    const t = r && toTarget(r);
    if (t) void linkRouter.handle(t, { userInitiated: true, fromPush: true });
  });
  // Warm / background
  return Notifications.addNotificationResponseReceivedListener(r => {
    const t = toTarget(r);
    if (t) void linkRouter.handle(t, { userInitiated: true, fromPush: true });
  });
}
```

Notes for adapting:
- **Route after switch**: the router stores the target as pending and the freshly keyed `NavigationContainer`'s `onReady` flushes it — so the route is pushed into the *new* session's navigator, never the old one. Call `linkRouter.attach(route => navigationRef.navigate(...parse(route)))` once.
- **Push for a removed account**: `candidates()` returns nothing for that `userId`; send an unregister request for that server user if your backend supports token-only unregister.
- **Background data messages** (`setBackgroundMessageHandler` / `TaskManager.defineTask`) should call `registry.load()` then `createApiClient({ accountId, signal })` for the payload's account — never `sessionManager.portal`.
- **Unified badge**: after each foreground resume, fetch `/unread` per account (bounded, parallel) and `Notifications.setBadgeCountAsync(sum)`.
- **Single-account migration**: in `registry.load()`, if `registry.v2` is missing and a legacy token exists, create one `AccountRecord` with a new id, move the token to `auth.<id>`, copy the default MMKV keys into `acct.<id>`, then delete the legacy items.
- **SQLite**: open `acct_<id>.db` in `AccountSession.open` and close it in `close()`; delete the file in `signOut`.
