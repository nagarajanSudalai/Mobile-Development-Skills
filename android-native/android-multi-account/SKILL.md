---
name: android-multi-account
description: Use when implementing, reviewing, testing, or debugging multi-account and multi-portal (workspace / organization / team) support in a native Android app — account registry, per-account token storage with Android Keystore and encrypted DataStore (Tink), EncryptedSharedPreferences deprecation and migration, per-account single-flight token refresh with OkHttp Interceptor/Authenticator, portal-scoped headers, Room database per account (and per portal), per-account caches and files, Hilt custom scoped components (DefineComponent) for session graphs, account switch and portal switch flows, add account / sign out, FCM push routing per account and portal, deep links to another account, WorkManager per account, Play Billing / RevenueCat identity on switch, analytics/crash user context, per-account app lock and data wipe, optional AccountManager integration. Siblings: ios-multi-account, flutter-multi-account, react-native-multi-account; related: android-deep-links, android-push-notifications, android-in-app-purchase, android-revenuecat-iap, android-list-ux.
---

# Android Multi-Account Skill

Guide for building, auditing, and testing **multiple signed-in accounts** and **multiple portals per account** in a consumer Android app. Two layers, modelled everywhere in this skill:

1. **Accounts** — several user identities signed in on the same device at once (personal + work email, two shop owners, parent + child profile on a shared tablet). Each has its own credentials, backend region/environment, and isolated local data.
2. **Portals** — each account may belong to several **portals** (workspaces, organizations, teams, stores, households). The user switches portal *within* an account. A portal switch is lighter than an account switch: same credentials, different scope.

The active context is always the pair **(accountId, portalId)**. Every request, DB query, cache entry, push, deep link, background job and analytics event must be attributable to that pair — or explicitly be "global".

Sibling skills: **ios-multi-account**, **flutter-multi-account**, **react-native-multi-account** — same structure per platform. Related: **android-deep-links** (link to account/portal B while A is active), **android-push-notifications** (token registration, routing), **android-in-app-purchase** / **android-revenuecat-iap** (purchase ownership on switch), **android-list-ux** (per-account list caches, paging reset).

> Android security and Jetpack libraries move fast. Notably: **`androidx.security:security-crypto` (EncryptedSharedPreferences / EncryptedFile / MasterKey) is deprecated** — every API was marked deprecated in `1.1.0-alpha07` (2025) and no further releases are planned; Google's guidance is to use the platform Keystore directly or Tink. Hilt custom components, DataStore multi-process, Room `MultiInstanceInvalidation`, `Credential Manager` and FCM APIs also change. **Verify against the versions in the project's `libs.versions.toml` / BOM and the current docs before writing code**, and prefer what the project already uses.
>
> Version gates used below: `minSdk 23` for AES-GCM keys in Android Keystore; StrongBox (`setIsStrongBoxBacked`) API 28+ and not on all devices; `setUserAuthenticationParameters` API 30+ (`setUserAuthenticationValidityDurationSeconds` below); `android:dataExtractionRules` API 31+ (`fullBackupContent` for 23–30); `POST_NOTIFICATIONS` runtime permission API 33+; Hilt 2.44+ for `@DefineComponent` ergonomics with `EntryPoints`.

## 0. Mental model

```
 Application (SingletonComponent)  -- global, account-agnostic only
 +----------------------------------------------------------------------+
 | AccountRegistry (DataStore, non-secret metadata)                     |
 |   accounts: [A(id, email, avatar, region, tokenRef), B(...), ...]    |
 |   order, limits, activeAccountId, lastPortalId per account           |
 | SecureTokenStore (Keystore-wrapped Tink AEAD -> encrypted DataStore) |
 | SessionManager: StateFlow<Session?>  <- the ONLY switch entry point  |
 | PushRouter, LinkRouter, base OkHttpClient (pool/dispatcher shared)   |
 +-------------------------------+--------------------------------------+
                                 | build / destroy
 +-------------------------------v--------------------------------------+
 | AccountComponent (@AccountScope)   one live instance = active account|
 |   AccountSession(accountId, region) | TokenManager (single-flight)   |
 |   OkHttpClient.newBuilder(): AuthInterceptor + Authenticator          |
 |   AccountDatabase  "acct_<id>.db"  | Coil ImageLoader (disk cache dir)|
 |   PortalRepository (portal list, roles)                              |
 +-------------------------------+--------------------------------------+
                                 | build / destroy (cheap)
 +-------------------------------v--------------------------------------+
 | PortalComponent (@PortalScope)  one live instance = active portal    |
 |   PortalSession(accountId, portalId, role) | X-Portal-Id interceptor |
 |   portal repositories, query caches, realtime socket, ViewModels'    |
 |   data sources; optional "acct_<id>_portal_<pid>.db"                 |
 +----------------------------------------------------------------------+

 Account switch: cancel scope -> close socket -> close DB -> drop AccountComponent
                 -> build new AccountComponent -> PortalComponent(lastPortal)
                 -> reset nav graph -> restore last screen for that account
 Portal switch:  cancel portal scope -> drop PortalComponent -> build new one
                 (account token, DB, HTTP client, image cache kept)
```

| Thing | Global | Per account | Per portal |
|---|---|---|---|
| Account list, order, active id | ✅ | | |
| Refresh/access tokens | | ✅ (encrypted) | ⚠️ only if backend issues portal tokens |
| OkHttp connection pool / dispatcher | ✅ (shared) | interceptors + authenticator | header interceptor |
| Room DB | | ✅ default | ✅ when portal data is large or must be wiped independently |
| HTTP cache / image disk cache | | ✅ | key prefix or separate dir |
| DataStore prefs (UI prefs, last screen) | device-level only (theme) | ✅ | ✅ (filters, last tab) |
| FCM token | ✅ one per install | registered per account on server | subscription/topics per portal |
| Billing / RevenueCat app user id | | ✅ | rarely |
| Analytics user id | | ✅ (hashed/opaque) | portal id as property |

## 1. First, establish context

Find out (from the codebase or by asking):

- **Max accounts** on one device (2? 5? unlimited?) and whether a cap is product or technical (DB files, memory, push registrations).
- **Backends**: one backend, or region/environment per account (`eu`, `us`, staging)? Base URL must live in the account record, never in a global `BuildConfig` read at call time.
- **Auth**: OAuth2/OIDC (AppAuth / Custom Tabs), password + refresh token, magic link, Credential Manager passkeys? Refresh token rotation? Token lifetime?
- **Portals**: how many per account (1–3 vs hundreds → needs search)? Does the backend issue **portal-scoped tokens**, or one account token + `X-Portal-Id` header / path segment (`/portals/{id}/...`)? Are roles/permissions per portal?
- **Data sharing**: is any data shared across accounts (none by default) or across portals of one account (user profile, DMs, notifications inbox)?
- **Offline**: per-account Room DB already? Size per portal? Must a portal's data survive leaving and rejoining?
- **Concurrency**: does only the active account need live data, or must inactive accounts sync in background (unread badges, notifications)?
- **Push**: FCM? Does the backend support multiple users per device token? Per-portal notification settings?
- **Purchases**: who owns a subscription — the account, the portal (team plan), or the Google Play account? (Play subscriptions are tied to the Google account; see section 13.)
- **Security**: shared-device scenarios (family tablet)? Per-account app lock / biometrics? Enterprise MDM / work profile?
- **DI**: Hilt, Koin, or manual? Existing `@Singleton` repositories that hold user data (the #1 leak source)?
- **AccountManager**: do other apps or sync adapters need to see these accounts? If not, don't use it (section 3.5).

Write the answers down as an **account contract** before coding:

```
max accounts: 5 | regions: eu/us per account | auth: OIDC + rotating refresh
portals: 1..300 per account, header X-Portal-Id, roles admin/member/guest
DB: acct_<id>.db, portal_id column (no per-portal DB) | inactive sync: badges only
push: FCM, server maps token -> [accountIds] | billing: per account (obfuscatedAccountId)
```

## 2. Data model

### 2.1 Checklist

- [ ] **AccountRecord** holds non-secret metadata only: `id` (server user id, stable), `displayName`, `email` (for duplicate detection & UI), `avatarUrl`, `region`/`baseUrl`, `environment`, `addedAt`, `lastUsedAt`, `state` (`Active`, `NeedsReauth`, `Removing`), `tokenRef` (key into the secure store — not the token).
- [ ] **PortalRecord** per account: `id`, `accountId`, `name`, `iconUrl`, `role`, `permissions` (or a version), `lastUsedAt`, `unreadCount`, `isDefault`, `state` (`Active`, `Revoked`, `PendingInvite`).
- [ ] **Active pair** persisted: `activeAccountId` + per-account `lastPortalId` (so switching back to A returns to A's last portal, not the global last one).
- [ ] Local account id is the **server user id + environment/region**, not the email (emails change) and not a random UUID that can't be de-duplicated. If ids may collide across regions, key on `"$region:$userId"`.
- [ ] **Ordering**: explicit `sortIndex` or `lastUsedAt`; stable across launches.
- [ ] **Limits** enforced in the registry (not just UI): adding the N+1th account fails with a typed error.
- [ ] Registry stored in **DataStore** (Proto or Preferences with a JSON blob) — atomic updates, `Flow`, no `commit()` races. Not `SharedPreferences`.
- [ ] Registry writes are **transactional** with secure-store writes: write token first, then registry; on failure roll back so no account exists without a token (or vice versa). Clean orphans on startup.
- [ ] Portal list is a **cache** of server truth: refreshed on login, on app foreground, on portal-switcher open, and on 403 "not a member" responses.
- [ ] Exposed as `StateFlow<RegistryState>`; UI never reads files directly.

### 2.2 Session model

```kotlin
data class Session(val account: AccountRecord, val portal: PortalRecord?)  // portal null = account has no portals yet
sealed interface SessionState { Loading; NoAccounts; Active(Session); Switching(from, to) }
```

`Switching` is a first-class state: UI shows a full-screen switch transition, requests are blocked, and nothing can read the old graph.

## 3. Secure token storage

### 3.1 What to use (current guidance)

| Option | Status | Use |
|---|---|---|
| `EncryptedSharedPreferences` / `MasterKey` (`androidx.security:security-crypto`) | **Deprecated** (all APIs deprecated in 1.1.0-alpha07; no planned releases). Still works; known issues with Keystore corruption after backup/restore and on some OEMs | Existing apps: keep reading, migrate. New code: don't adopt |
| **Tink AEAD keyset wrapped by an Android Keystore key, ciphertext in DataStore** | Recommended. Tink (`com.google.crypto.tink:tink-android`) is actively maintained; `AndroidKeysetManager` stores the keyset encrypted with a Keystore master key | Default for tokens |
| Raw Android Keystore `AES/GCM/NoPadding` key + DataStore | Fine, fewer deps, you own IV handling | When Tink is unwanted |
| DataStore + a community "encrypted DataStore" wrapper | Varies | Evaluate maintenance before adopting |
| `AccountManager` `setPassword`/`setAuthToken` | Stored **unencrypted** in the system accounts DB (protected by uid, readable on rooted devices) | Only alongside encryption, see 3.5 |
| Plain `SharedPreferences` / DataStore / Room | ❌ | Never for tokens |

> Re-verify the deprecation status and the recommended replacement in the `androidx.security` release notes before advising — if a new stable release has appeared, adjust.

### 3.2 Checklist

- [ ] One **Keystore key alias** for the app (e.g. `tink_master_key`), not one per account — Keystore key count and creation latency matter; per-account isolation comes from separate entries + wipe on removal. (Per-account keys are acceptable when you need cryptographic erasure per account: delete the alias = data unreadable.)
- [ ] Token blob per account: `{ accessToken, accessExpiresAt, refreshToken, idToken?, portalTokens?: map }`, serialized then AEAD-encrypted with **associated data = accountId** (prevents swapping ciphertexts between accounts).
- [ ] Encrypted values stored in a **dedicated DataStore file** (`secure_tokens.preferences_pb`), separate from the registry.
- [ ] Access token cached **in memory** inside the account session; disk read only at session start / refresh.
- [ ] **Key invalidation handling**: `KeyPermanentlyInvalidatedException`, `AEADBadTagException`, `GeneralSecurityException` on decrypt (restore to new device, Keystore wiped, lock-screen removal if auth-bound) → mark account `NeedsReauth`, delete its blob, show "Sign in again" — never crash, never loop.
- [ ] Don't bind token keys to user authentication (`setUserAuthenticationRequired`) unless you want the app unusable without biometrics on every refresh; use a separate app-lock gate instead (section 15).
- [ ] StrongBox opportunistically (`setIsStrongBoxBacked(true)` with fallback on `StrongBoxUnavailableException`) — optional for consumer apps.
- [ ] **Backup exclusion**: Keystore keys never back up, so encrypted blobs restored to a new device are undecryptable. Exclude token DataStore, account DBs and per-account dirs from cloud backup and device transfer via `dataExtractionRules` (API 31+) and `fullBackupContent` (≤30). Restore the *registry* only if you want a "sign in again" list; otherwise exclude it too.
- [ ] Never log tokens; redact `Authorization` in `HttpLoggingInterceptor` (`redactHeader("Authorization")`) and only enable logging in debug.
- [ ] Migration from ESP: on first launch of the new version, read ESP, re-encrypt into the new store, verify read-back, then `deleteSharedPreferences(...)`. Keep the ESP read path for a few releases; tolerate ESP decrypt failures as `NeedsReauth`.

### 3.3 Backup rules example

```xml
<!-- res/xml/data_extraction_rules.xml (API 31+) -->
<data-extraction-rules>
  <cloud-backup>
    <exclude domain="file" path="datastore/secure_tokens.preferences_pb"/>
    <exclude domain="database" path="."/>           <!-- all acct_*.db -->
    <exclude domain="file" path="accounts/"/>
    <exclude domain="sharedpref" path="__androidx_security_crypto_encrypted_prefs__.xml"/>
  </cloud-backup>
  <device-transfer>
    <exclude domain="file" path="datastore/secure_tokens.preferences_pb"/>
    <exclude domain="database" path="."/>
    <exclude domain="file" path="accounts/"/>
  </device-transfer>
</data-extraction-rules>
```

Plus the equivalent `res/xml/backup_rules.xml` for `android:fullBackupContent` on API 23–30. If the app sets `android:allowBackup="false"`, none of this is needed — but confirm it's intentional.

### 3.4 Where things live on disk

```
files/datastore/account_registry.preferences_pb   (metadata, may back up)
files/datastore/secure_tokens.preferences_pb      (Tink ciphertext, no backup)
files/datastore/acct_<id>_prefs.preferences_pb    (per-account prefs)
files/accounts/<id>/...                           (downloads, attachments)
cache/accounts/<id>/http, cache/accounts/<id>/images
databases/acct_<id>.db (+ -wal, -shm)
```

Use a hashed or sanitized id in file names (`id.sha256().take(16)`) — raw ids may contain `/`, `:` or PII (emails).

### 3.5 AccountManager (optional)

Use `AccountManager` + an `AbstractAccountAuthenticator` service **only** if you need: accounts visible in Settings → Accounts, `SyncAdapter` integration, or sharing auth with your other apps signed with the same key. Costs: `GET_ACCOUNTS`/visibility rules (API 26+ `setAccountVisibility`), authenticator service boilerplate, tokens stored unencrypted by the system, users can remove the account from Settings behind your back (listen via `addOnAccountsUpdatedListener` and run your sign-out cleanup), and an extra source of truth to keep in sync with your registry. For most consumer apps: **skip it**; your registry + encrypted store is the source of truth. If used, still store tokens encrypted (put ciphertext in `setUserData`), and treat AccountManager as a mirror, not the source of truth.

## 4. Per-account token lifecycle

### 4.1 Checklist

- [ ] One `TokenManager` **per account**, living in the account scope; it owns the in-memory access token and a `Mutex` for refresh.
- [ ] **Single-flight refresh**: N concurrent 401s → one refresh call; others await the result. Implemented with a `Mutex` and a "token changed since my request" check (compare the token the request used to the current one; if different, just retry with the current).
- [ ] OkHttp: `Interceptor` adds `Authorization` (and proactively refreshes when `expiresAt - skew < now`); `Authenticator` handles 401 → refresh → retry once. Return `null` from `authenticate` when `responseCount(response) >= 2` or when refresh failed to avoid loops.
- [ ] Interceptors capture **their account** at construction (from the account scope), never `SessionManager.current` at call time — otherwise a request started for A and retried after a switch would get B's token.
- [ ] `Authenticator` runs on OkHttp threads; bridging to suspend with `runBlocking` is acceptable there but must not run on the main thread and must use a timeout.
- [ ] **Refresh token rotation**: persist the new refresh token **before** releasing the mutex; if the process dies between server response and write, the next refresh will fail → handle as `NeedsReauth` gracefully.
- [ ] Refresh failure types: network (keep tokens, retry later), `invalid_grant`/401 on refresh (account revoked → `NeedsReauth`), 5xx (backoff). Only `invalid_grant` marks the account as needing re-auth.
- [ ] **One account revoked while others fine**: mark only that account; if it's active, show an inline "Session expired for a@x.com — Sign in / Switch account" screen; other accounts stay usable in the switcher.
- [ ] Inactive accounts' tokens refreshed lazily (on switch, or by their background work), not all at startup.
- [ ] **Portal scope**: default = account token + `X-Portal-Id` header added by a **portal-scoped interceptor** (constructed with the portal id). If the backend issues portal-scoped tokens: store them in the account's blob keyed by portal id, refresh per portal with the same single-flight pattern keyed by `(accountId, portalId)`, and drop them on portal switch-away only if short-lived.
- [ ] Never put the portal id in a global header provider; never read "current portal" from a singleton in an interceptor.
- [ ] 403 with "not a member of portal" → portal revoked flow (section 7), not sign-out.

## 5. Data isolation

### 5.1 Strategy

| Strategy | Pros | Cons | Use when |
|---|---|---|---|
| **Room DB per account** (`acct_<id>.db`) | Wipe = delete file; no cross-account query bugs; schema migrations per file | N DB files; open/close on switch; migrations run per file | **Default** |
| DB per account **and** per portal | Portal wipe is trivial; small DBs | Many files; cross-portal views (unified inbox) hard | Big per-portal datasets, portal data must be wiped on leave |
| Single DB, `accountId`/`portalId` columns | One file, cross-account queries possible | Every query must filter; one missed `WHERE` = leak; wipe is `DELETE` across tables | Rarely; only small, shared, non-sensitive data |

Usual answer: **DB per account, `portal_id` column + index on portal-scoped tables**, with DAO methods that take `portalId` as a required parameter (no default).

### 5.2 Checklist

- [ ] `Room.databaseBuilder(context, AccountDatabase::class.java, "acct_${hash(id)}.db")` created **inside the account component**; closed (`db.close()`) on account switch after the account scope is cancelled.
- [ ] Inactive account DBs are **closed**; background work for them opens a short-lived instance (or gets one from a small, ref-counted `AccountDatabaseProvider` keyed by account id). Never two open `RoomDatabase` instances for the same file in one process without `enableMultiInstanceInvalidation()`.
- [ ] Encrypted DB at rest (SQLCipher via `net.zetetic:sqlcipher-android` + `SupportOpenHelperFactory`) only if the threat model needs it; key per account in the secure store. Default Android file-based encryption covers most consumer apps.
- [ ] Portal-scoped DAOs always take `portalId`; `Flow` queries re-created on portal switch (they live in the portal scope).
- [ ] **HTTP cache** per account: `Cache(File(cacheDir, "accounts/$h/http"), size)` on the account's `OkHttpClient`. Responses vary by portal → include portal in URL or `Vary: X-Portal-Id` from the server; otherwise disable caching for portal endpoints.
- [ ] **Image cache** (Coil/Glide) per account: separate `ImageLoader` with its own disk cache dir, or memory cache keys including account id. Authenticated image URLs must use the account's OkHttp client.
- [ ] **Files**: downloads/attachments under `files/accounts/<h>/`; `FileProvider` paths scoped; shared content URIs revoked on removal.
- [ ] **Prefs**: per-account DataStore (`acct_<h>_prefs`), per-portal keys inside it (`portal_<pid>_lastTab`), or per-portal file if large. Device-level prefs (theme, onboarding seen) stay global.
- [ ] **In-memory state**: no `@Singleton` / `object` holding user data. Audit every `@Singleton` class: if it touches user data, move it to `@AccountScoped` or `@PortalScoped`.
- [ ] **ViewModels**: `@HiltViewModel` lives in `ViewModelComponent` (child of `ActivityRetainedComponent`, not of your session component). Inject a `SessionProvider` / `EntryPoint` accessor and fetch session-scoped repositories at creation; on switch, the nav graph is reset so all ViewModels are cleared. Never cache a repository in a ViewModel that survives a switch.
- [ ] `WorkManager`, `PagingSource`, `RemoteMediator` instances carry the account/portal id explicitly.
- [ ] Clipboard, share sheet, recent-items shortcuts (`ShortcutManager` dynamic shortcuts) and `Glance` widgets reference account ids and are cleared/rebuilt on switch/removal.

## 6. Account switch

### 6.1 Teardown → rebuild (order matters)

1. `SessionState = Switching(from, to)`; UI shows switch overlay; navigation input blocked.
2. Save current account's **last screen** (route + key args) and last portal to its prefs.
3. Cancel the **portal scope** then the **account scope** (`CoroutineScope` owned by each component) → cancels in-flight requests (OkHttp calls tied to coroutines are cancelled), Flow collectors, paging.
4. Close realtime sockets/SSE, stop foreground services tied to the account, cancel account's non-persistent `WorkManager` work that's UI-driven.
5. Close Room DB; evict Coil memory cache for the old account loader; drop references to the components.
6. Update `activeAccountId` in the registry (single DataStore `updateData`).
7. Build new `AccountComponent` → load tokens (decrypt) → if `NeedsReauth`, go to re-auth screen for that account.
8. Choose portal: last used for that account → default → first; build `PortalComponent`.
9. Update identity in analytics, crash reporting, billing/RevenueCat (section 13–14).
10. **Reset navigation**: `navController.navigate(startRoute) { popUpTo(graph.id) { inclusive = true } }` — or recreate the `NavHost` keyed by `(accountId, portalId)` (`key(session.key) { AppNavHost() }`), which also clears all `ViewModelStoreOwner`s in back stack entries.
11. Restore the account's last screen if still valid (validate route/args against the new session).
12. `SessionState = Active(newSession)`.

### 6.2 Checklist

- [ ] Single entry point `SessionManager.switchAccount(id)`; serialized with a `Mutex` — rapid double-taps don't build two graphs.
- [ ] Switch is **atomic from the UI's view**: no frame renders A's data with B's header.
- [ ] Every response arriving after teardown is discarded (scope cancelled; repositories check `session.isActive` before writing to DB).
- [ ] `Activity` recreation not used as the switch mechanism unless the app is small — it's slow and leaks via retained state anyway. If used, still tear down the session components.
- [ ] Process death: on cold start, rebuild session from persisted `activeAccountId`/`lastPortalId`; `SavedStateHandle` args must include account id so a restored screen can verify it belongs to the active account.
- [ ] Switcher UI: avatar + name + email + per-account unread badge; active marked; "Add account"; "Manage accounts". Accessible labels ("Switch to work account, a@x.com, 3 unread").
- [ ] Switch latency target < 500 ms on mid-range devices: lazy-open DB, don't preload portal data before showing the shell, show cached data first.
- [ ] Toast/snackbar after switch ("Switched to a@x.com") for orientation; avatar in the top bar always shows the active account.

## 7. Portal switch

### 7.1 What's reset vs kept

| Reset | Kept |
|---|---|
| Portal component: repositories, query caches, Paging flows, realtime channel subscription, portal ViewModels (nav reset to portal home) | Account component: tokens, OkHttp client, account DB file, image loader, account profile |
| `X-Portal-Id` interceptor instance | HTTP connection pool |
| Portal-scoped in-memory caches | Portal rows in DB (filtered by `portal_id`) — enables instant switch-back |
| Portal-specific nav back stack | Account-level screens (profile, account settings) if the app has them |

### 7.2 Checklist

- [ ] `SessionManager.switchPortal(portalId)` — same mutex as account switch; validates portal belongs to the active account.
- [ ] Persist `lastPortalId` for the **account** immediately.
- [ ] **Portal picker**: recent portals (by `lastUsedAt`) on top; search field when > ~7 portals; show role badge and unread count; current portal checked; long lists use `LazyColumn` with stable keys (see android-list-ux).
- [ ] **Default portal on login**: server-designated default → last used on another device (if API provides) → single portal auto-selected → otherwise picker before entering the app.
- [ ] Account with **zero portals**: dedicated empty state (create / join / pending invitations), not a crash on `portal!!`.
- [ ] **Portal removed / access revoked mid-session** (403 membership error, push "removed_from_portal", portal list refresh): stop portal work, show "You no longer have access to <Portal>", switch to next portal (default/last used), purge that portal's rows and files (`DELETE WHERE portal_id = ?` or delete portal DB), drop its notifications.
- [ ] **Role/permission change**: permissions come from the portal session; on change (refresh or push), update `PortalSession.role` via a `StateFlow` so UI gates recompose; downgraded role mid-action → server 403 handled with a message, not a crash.
- [ ] **Invitations**: pending invites listed in picker; accepting adds the portal and offers to switch; invite deep link for a different account → section 11.
- [ ] Portal switch is visibly cheaper: no full-screen overlay; shell keeps account chrome; content area shows skeleton.

## 8. Add account

- [ ] "Add account" launches the login flow **without** tearing down the current session first; on cancel, user is back where they were.
- [ ] Login flow uses a **fresh, unauthenticated** OkHttp client (base URL from the region chosen in login) — never the active account's client (its interceptors would attach A's token).
- [ ] OAuth via Custom Tabs/AppAuth: use `prompt=login` / `select_account` so the browser's existing session for A doesn't silently log in A again; distinct `state` per attempt.
- [ ] **Duplicate detection** after token exchange, by `(region, serverUserId)`: if already present → update its tokens (re-auth), switch to it, show "Already signed in as a@x.com". Don't create a second record.
- [ ] **Different region**: region picked before login or discovered from a lookup endpoint; stored in the account record; OkHttp base URL per account (Retrofit instance per account, or a `HostSelectionInterceptor` constructed with the account's base URL).
- [ ] Limit reached → explain before starting login, not after.
- [ ] On success: write tokens → write registry → fetch portals → switch to new account (product decision: most apps switch immediately).
- [ ] Credential Manager: saving passwords/passkeys per account is fine; don't auto-select a credential for "Add account".

## 9. Sign out

- [ ] **Sign out this account** vs **Sign out of all accounts** — both explicit in UI.
- [ ] Order: mark `Removing` → (if active) switch away first → server revoke (best effort, with timeout; `/logout` + unregister push token for that account) → cancel account's WorkManager by tag → delete DB files (`context.deleteDatabase(name)` removes -wal/-shm) → delete per-account dirs (files + cache) → delete per-account DataStore → delete token blob (and Keystore alias if per-account) → remove registry entry → clear billing/analytics identity if it was active.
- [ ] Next active account: most recently used remaining; if none → logged-out start screen with back stack cleared.
- [ ] Cleanup is **resumable**: if the process dies mid-cleanup, the `Removing` state makes startup finish it.
- [ ] Server revoke failure doesn't block local removal.
- [ ] Notifications from that account removed (`NotificationManager.cancel` by tag/group for account).
- [ ] Dynamic shortcuts, widgets, Glance state referencing the account removed.

## 10. Push per account and portal

- [ ] FCM gives **one token per app install**. Server must map `deviceToken → [accountId…]` (register the token with each signed-in account's credentials); on token refresh (`onNewToken`), re-register for **every** account in the registry (WorkManager job, per-account auth).
- [ ] Payload includes `accountId` (or opaque `recipientKey`) and `portalId`; never route by "currently active".
- [ ] `onMessageReceived`: look up account in registry; unknown/removed account → drop (and ask server to unregister). Build notification with account-aware content: show account email/portal name as subtext when multiple accounts exist.
- [ ] **Notification channels**: per type, optionally per account (`channelGroup` per account) so users can mute one account. Channel ids stable (`acct_<h>_messages`). Delete groups on account removal.
- [ ] Tap `PendingIntent` carries `accountId` + `portalId` + target route; unique request codes per notification so extras aren't merged. On open: if pair ≠ active → switch (account and/or portal) via `SessionManager`, then navigate. Inactive account `NeedsReauth` → re-auth screen first, then deliver.
- [ ] **Badge/unread per account and portal** kept in the registry; switcher and portal picker show them; launcher badge = sum (or active only — decide).
- [ ] Unregister on account removal and on portal leave (per-portal topics/subscriptions).
- [ ] Data-only messages for inactive accounts must not open that account's DB on the main thread; enqueue work.
- [ ] See **android-push-notifications** for channels, permissions (API 33), and delivery details.

## 11. Deep links per account and portal

- [ ] Links carry portal (and optionally account hint) — `https://app.example.com/p/{portalId}/item/{id}`; server can resolve portal → which signed-in account(s) belong to it.
- [ ] Resolution: active pair matches → navigate. Portal belongs to another signed-in account → **confirm** ("Open in <Portal> as b@x.com?") then switch + navigate. Multiple accounts are members → ask which. None → offer "Add account" or show "You don't have access". Logged out → store pending link, resume after login (see **android-deep-links**).
- [ ] Never silently switch account from a link; portal switch within the active account may be silent if product agrees.
- [ ] Validate after switch that the target is still accessible (403 → fallback).

## 12. Background work

- [ ] `WorkManager` requests carry `accountId` (+ `portalId`) in `inputData` and are tagged `acct:<id>` / `portal:<id>` for cancellation.
- [ ] `CoroutineWorker` resolves the account's dependencies explicitly (account-keyed provider), **not** from the active session; if the account is gone or `NeedsReauth` → `Result.failure()` without retry.
- [ ] Unique work names include the account: `sync_${accountId}_${portalId}`.
- [ ] Periodic sync for inactive accounts: lightweight (unread counts) and budgeted; full sync only for active.
- [ ] Foreground services (uploads) show the account in the notification and stop on removal.
- [ ] HiltWorker: inject account-agnostic factories (`AccountGraphFactory`), not account-scoped types.

## 13. Purchases

- [ ] Decide ownership: Play subscriptions belong to the **Google Play account**, not your user. Pass `setObfuscatedAccountId(hash(accountId))` (and `setObfuscatedProfileId(portalId)` for team plans) in `BillingFlowParams`; server-verify and bind to the account.
- [ ] On account switch: re-query purchases and re-evaluate entitlements for the new account from **your server**; don't show A's entitlement for B. Purchase made on device while B active but owned by A → server decides (restore/transfer policy).
- [ ] RevenueCat: `Purchases.sharedInstance.logIn(appUserId)` on switch (not `logOut` then `logIn` — `logOut` creates an anonymous user); `logOut` only when the last account signs out. Entitlement `Flow` re-subscribed per session. See **android-revenuecat-iap** / **android-in-app-purchase**.
- [ ] Portal/team plans: entitlement is per portal; recompute on portal switch.

## 14. Analytics and crash reporting

- [ ] On switch: set user id to an **opaque, non-PII** id (server user id or hash; never email); set `portal_id`, `role`, `account_count` as properties.
- [ ] Firebase Crashlytics `setUserId`, Analytics `setUserId`; on sign-out of last account → clear (`null`).
- [ ] Events emitted during teardown attributed to the old account (send before identity change) — order matters.
- [ ] Breadcrumbs `account_switch`, `portal_switch` (no ids in plaintext if they're PII).
- [ ] Consent is per device *and* may be per account — if consent differs, apply the active account's.

## 15. Security and privacy

- [ ] **Per-account app lock** (optional): `BiometricPrompt` with `BIOMETRIC_STRONG or DEVICE_CREDENTIAL` gating entry into that account on switch/resume; lock state lives in the account scope; timeout configurable.
- [ ] **Recents snapshot**: sensitive accounts → `window.setFlags(FLAG_SECURE)` or, on API 33+, `setRecentsScreenshotEnabled(false)` (Activity); also hide content during switch overlay.
- [ ] Notification content on lock screen: `VISIBILITY_PRIVATE` with a public version for sensitive accounts.
- [ ] **Full wipe on account deletion** (user deletes account server-side): same as sign out + make sure no residual files (exports, shared cache, WebView storage). WebView cookies/storage are **process-global** — use `WebStorage`/`CookieManager` cleanup, or on API 28+ separate data directories per process; prefer Custom Tabs for authenticated web content.
- [ ] Rooted/compromised devices: Keystore-backed keys still non-exportable; don't over-engineer for consumer apps.
- [ ] No account data in `Intent` extras sent to other apps; `PendingIntent.FLAG_IMMUTABLE`.
- [ ] Logs: no tokens, emails, portal names in release logs.

## 16. Testing plan

1. **Unit**: `TokenManager` single-flight with 20 concurrent 401s (`MockWebServer` counts exactly one `/token` call); refresh `invalid_grant` → `NeedsReauth` only for that account; `AccountRegistry` duplicate detection, limits, ordering; secure store round-trip + AAD mismatch fails; ESP → Tink migration.
2. **Integration**: `SessionManager` switch with fake components — assert old scope cancelled, DB closed, new graph built, identity updated, in order. In-memory Room per account.
3. **Instrumented**: Keystore on real device + emulator (API 23, 28, 34+); backup/restore via `adb shell bmgr backupnow <pkg>` then `bmgr restore` → app shows "Sign in again", no crash.
4. **UI (Compose)**: switcher, portal picker search, revoked-portal banner, re-auth screen; nav reset after switch (no back to A's screens).
5. **Push**: `adb shell am broadcast` is unreliable for FCM; use the FCM HTTP v1 API from a test script with `accountId`/`portalId` for active/inactive/removed accounts.
6. **Process death**: switch, background, `adb shell am kill <pkg>`, reopen → same pair restored.
7. **Leak audit**: LeakCanary after 10 switches — old components, DBs, ImageLoaders collected; heap dump shows one `AccountComponent`.
8. **StrictMode**: no disk reads on main during switch.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| M1 | Add second account | Login with clean client; A untouched until success; switch to B |
| M2 | Add account already signed in | No duplicate; tokens updated; switched to it; message shown |
| M3 | Add account in other region | B's requests go to B's base URL; A's to A's |
| M4 | Switch account mid-request (slow API) | A's response discarded; never written to B's DB or shown |
| M5 | Switch account during paging/scroll | Lists reset; no A rows in B |
| M6 | Rapid double switch A→B→A | One final graph (A); no crash; no leaked component |
| M7 | Switch portal | Account token kept; `X-Portal-Id` updated; portal home shown; switch back instant from cache |
| M8 | Portal revoked mid-session (403) | Message; moved to next portal; portal data purged |
| M9 | Role downgraded in active portal | Admin UI hidden after refresh; in-flight admin action → graceful 403 |
| M10 | Account with zero portals | Empty state with create/join/invites |
| M11 | 300 portals | Picker search + recents; smooth scroll |
| M12 | Access token expired, 10 parallel calls | One refresh; all retried once |
| M13 | Refresh token revoked on inactive account B | A unaffected; B badged "Sign in again" in switcher |
| M14 | Refresh token revoked on active account | Re-auth screen for that account; option to switch |
| M15 | Push for active account+portal | Shown; tap navigates directly |
| M16 | Push for inactive account | Shown with account label; tap switches account, then navigates |
| M17 | Push for inactive portal of active account | Tap switches portal, then navigates |
| M18 | Push for removed account | Dropped; server unregister requested |
| M19 | FCM token refresh | Re-registered for all accounts |
| M20 | Deep link to portal of other signed-in account | Confirmation; switch; navigate |
| M21 | Deep link to portal no account can access | Access message / add account |
| M22 | Sign out non-active account | Its data/files/notifications removed; active untouched |
| M23 | Sign out active account | Switch to most recent other; data wiped |
| M24 | Sign out last account | Logged-out start; analytics/billing identity cleared; nothing left on disk |
| M25 | Process killed mid-sign-out | Cleanup completes on next start |
| M26 | Backup restored to new device | Keystore keys missing → accounts "Sign in again" or absent; no crash |
| M27 | ESP → new store migration | Tokens readable after upgrade; ESP file removed |
| M28 | Process death after switch | Same (account, portal) and last screen restored |
| M29 | Switch while offline | Switch works from cache; refresh deferred; banner shown |
| M30 | Purchases after switch | Entitlements reflect B; RevenueCat `logIn(B)` called |
| M31 | Background sync for inactive account | Uses that account's token and DB; no effect on active UI |
| M32 | Recents screen on sensitive account | Content hidden |

## 17. Common bugs to look for in review

- `@Singleton` repository/cache holding user data → A's data visible after switching to B
- Interceptor reading `sessionManager.current.token` at call time → request started for A retried with B's token (or vice versa)
- No single-flight refresh → refresh token rotation race logs the user out
- `Authenticator` without retry limit → infinite 401 loop; `runBlocking` on main thread
- One global `X-Portal-Id` header provider → requests for old portal sent with new portal header
- New `EncryptedSharedPreferences` code added in 2025+; or ESP crashes (`AEADBadTagException`, `KeyStoreException`) uncaught at startup after restore
- Tokens in plain DataStore / Room / logs; `HttpLoggingInterceptor` at `BODY` in release
- Encrypted token files included in backup → undecryptable on restore → crash loop
- Single DB with `account_id` column and a DAO query missing the filter
- Room DB for old account left open after switch; two instances of the same file
- HTTP/image disk cache shared across accounts → B sees A's avatars/attachments offline
- Account identified by email → duplicate records after email change; re-login as same user creates second account
- Login for "Add account" using the active account's OkHttpClient
- ViewModels surviving switch (nav not reset) holding A's repositories
- `SavedStateHandle` restores a screen for A after cold start while B is active
- Push routed to "active account" instead of payload's account; notification tap with merged extras (`PendingIntent` request code reused)
- `onNewToken` registering only the active account
- WorkManager jobs resolving active session instead of their `inputData` account
- RevenueCat `logOut()` on every switch (creates anonymous users), entitlements cached across accounts
- Analytics user id set to email; not cleared on last sign-out
- Sign-out deleting registry before files → orphaned DBs forever
- Portal revoked → app keeps showing cached portal data; or treats 403 as sign-out
- `lastPortalId` stored globally instead of per account
- WebView cookies shared between accounts
- AccountManager used as source of truth while user can remove accounts from Settings without the app noticing

## 18. Output format

When asked to **review/audit**:
1. **Summary** — account contract (max accounts, regions, auth, portal model, DB strategy, push, billing), DI framework, library versions (Hilt, Room, DataStore, OkHttp, Tink / security-crypto)
2. **Checklist results** — sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** (cross-account data leaks, token sent to wrong account/portal, refresh races logging users out, plaintext tokens, crash on restore, push opening wrong account) with fix code
4. **Testing gaps** — M1–M32 not covered, and which need a real device (Keystore, backup, FCM)

When asked to **implement**: fill in the account contract first, then start from the Appendix: copy the registry, secure store, scoped components, session manager and OkHttp auth pieces; delete what the contract doesn't need (per-portal DB, portal tokens, AccountManager); match library versions to the project and verify Hilt/Tink APIs compile; finish with the testing plan items you ran and those left for the user.

---

## Appendix — Reference implementation (Kotlin, Hilt, Room, DataStore, Tink, OkHttp)

> Written against Kotlin 2.x, Hilt 2.5x, Room 2.6/2.7, DataStore 1.1, OkHttp 4.12, Tink Android 1.1x, kotlinx.serialization 1.7. Tink and Hilt custom-component APIs are stable but verify signatures; drop `androidx.security:security-crypto` except for the migration reader.

### A.1 Models

```kotlin
@Serializable
data class AccountRecord(
    val id: String,               // "$region:$serverUserId"
    val serverUserId: String,
    val region: String,
    val baseUrl: String,
    val displayName: String,
    val email: String,
    val avatarUrl: String? = null,
    val addedAt: Long,
    val lastUsedAt: Long,
    val state: AccountState = AccountState.Active,
    val lastPortalId: String? = null,
    val unread: Int = 0,
)

@Serializable enum class AccountState { Active, NeedsReauth, Removing }

@Serializable
data class PortalRecord(
    val id: String,
    val accountId: String,
    val name: String,
    val role: Role,
    val lastUsedAt: Long = 0,
    val unread: Int = 0,
    val isDefault: Boolean = false,
)

@Serializable enum class Role { Owner, Admin, Member, Guest }

@Serializable
data class RegistrySnapshot(
    val accounts: List<AccountRecord> = emptyList(),
    val portals: Map<String, List<PortalRecord>> = emptyMap(), // accountId -> portals
    val activeAccountId: String? = null,
)

@Serializable
data class TokenSet(
    val accessToken: String,
    val accessExpiresAt: Long,
    val refreshToken: String,
)

val String.fileKey: String
    get() = MessageDigest.getInstance("SHA-256").digest(toByteArray())
        .joinToString("") { "%02x".format(it) }.take(16)
```

### A.2 Secure token store (Tink AEAD + Keystore + DataStore)

```kotlin
@Singleton
class SecureTokenStore @Inject constructor(
    @ApplicationContext private val context: Context,
    private val json: Json,
) {
    private val Context.tokenStore by preferencesDataStore(name = "secure_tokens")

    private val aead: Aead by lazy {
        AeadConfig.register()
        AndroidKeysetManager.Builder()
            .withSharedPref(context, "tink_keyset", "tink_keyset_prefs") // keyset is itself encrypted by the Keystore key
            .withKeyTemplate(KeyTemplates.get("AES256_GCM"))
            .withMasterKeyUri("android-keystore://tink_master_key")
            .build()
            .keysetHandle
            .getPrimitive(RegistryConfiguration.get(), Aead::class.java)
    }

    private fun key(accountId: String) = stringPreferencesKey("t_${accountId.fileKey}")

    suspend fun write(accountId: String, tokens: TokenSet) = withContext(Dispatchers.IO) {
        val plain = json.encodeToString(TokenSet.serializer(), tokens).toByteArray()
        val cipher = aead.encrypt(plain, accountId.toByteArray()) // AAD binds blob to account
        context.tokenStore.edit { it[key(accountId)] = Base64.encodeToString(cipher, Base64.NO_WRAP) }
    }

    /** Returns null when absent; throws [TokenUnreadable] when present but undecryptable. */
    suspend fun read(accountId: String): TokenSet? = withContext(Dispatchers.IO) {
        val b64 = context.tokenStore.data.first()[key(accountId)] ?: return@withContext null
        try {
            val plain = aead.decrypt(Base64.decode(b64, Base64.NO_WRAP), accountId.toByteArray())
            json.decodeFromString(TokenSet.serializer(), String(plain))
        } catch (e: GeneralSecurityException) {
            delete(accountId)
            throw TokenUnreadable(accountId, e)
        }
    }

    suspend fun delete(accountId: String) = withContext(Dispatchers.IO) {
        context.tokenStore.edit { it.remove(key(accountId)) }
    }
}

class TokenUnreadable(val accountId: String, cause: Throwable) : Exception(cause)
```

> `AndroidKeysetManager.withSharedPref` keeps the *encrypted keyset* in a SharedPreferences file — that's fine (it's ciphertext) but **exclude `tink_keyset_prefs.xml` from backup** too. If the Keystore master key is missing, Tink by default may regenerate a keyset; old token ciphertexts then fail to decrypt → `TokenUnreadable` → `NeedsReauth`.

#### ESP migration reader (delete after a few releases)

```kotlin
@Suppress("DEPRECATION")
suspend fun migrateFromEncryptedPrefs(context: Context, store: SecureTokenStore, json: Json) {
    val name = "secure_prefs_legacy"
    if (!File(context.filesDir.parent, "shared_prefs/$name.xml").exists()) return
    runCatching {
        val master = MasterKey.Builder(context).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build()
        val esp = EncryptedSharedPreferences.create(
            context, name, master,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
        )
        esp.all.forEach { (k, v) ->
            if (k.startsWith("tokens_") && v is String) {
                val accountId = k.removePrefix("tokens_")
                val tokens = json.decodeFromString(TokenSet.serializer(), v)
                store.write(accountId, tokens)
                check(store.read(accountId) == tokens)
            }
        }
    } // on failure: accounts will surface as NeedsReauth
    context.deleteSharedPreferences(name)
}
```

### A.3 Account registry

```kotlin
@Singleton
class AccountRegistry @Inject constructor(
    @ApplicationContext context: Context,
    private val tokens: SecureTokenStore,
    private val json: Json,
) {
    private val store: DataStore<RegistrySnapshot> = DataStoreFactory.create(
        serializer = RegistrySerializer(json),
        produceFile = { context.dataStoreFile("account_registry.json") },
    )
    val state: Flow<RegistrySnapshot> = store.data

    suspend fun snapshot() = store.data.first()

    sealed interface AddResult { data class Added(val id: String) : AddResult; data class Existing(val id: String) : AddResult; data object LimitReached : AddResult }

    suspend fun addOrUpdate(record: AccountRecord, tokenSet: TokenSet, maxAccounts: Int = 5): AddResult {
        val existing = snapshot().accounts.firstOrNull { it.id == record.id }
        if (existing == null && snapshot().accounts.size >= maxAccounts) return AddResult.LimitReached
        tokens.write(record.id, tokenSet)                    // 1. secret first
        store.updateData { s ->                              // 2. then metadata
            val others = s.accounts.filterNot { it.id == record.id }
            val merged = existing?.copy(
                displayName = record.displayName, email = record.email, avatarUrl = record.avatarUrl,
                state = AccountState.Active, lastUsedAt = record.lastUsedAt,
            ) ?: record
            s.copy(accounts = others + merged)
        }
        return if (existing == null) AddResult.Added(record.id) else AddResult.Existing(record.id)
    }

    suspend fun setActive(accountId: String) = store.updateData { s ->
        s.copy(activeAccountId = accountId, accounts = s.accounts.map {
            if (it.id == accountId) it.copy(lastUsedAt = System.currentTimeMillis()) else it
        })
    }

    suspend fun setLastPortal(accountId: String, portalId: String) = store.updateData { s ->
        s.copy(accounts = s.accounts.map { if (it.id == accountId) it.copy(lastPortalId = portalId) else it })
    }

    suspend fun setPortals(accountId: String, portals: List<PortalRecord>) = store.updateData { s ->
        s.copy(portals = s.portals + (accountId to portals))
    }

    suspend fun setState(accountId: String, state: AccountState) = store.updateData { s ->
        s.copy(accounts = s.accounts.map { if (it.id == accountId) it.copy(state = state) else it })
    }

    suspend fun remove(accountId: String) = store.updateData { s ->
        s.copy(
            accounts = s.accounts.filterNot { it.id == accountId },
            portals = s.portals - accountId,
            activeAccountId = s.activeAccountId.takeUnless { it == accountId },
        )
    }

    fun nextActiveAfterRemoval(s: RegistrySnapshot, removed: String): AccountRecord? =
        s.accounts.filter { it.id != removed && it.state != AccountState.Removing }
            .maxByOrNull { it.lastUsedAt }
}

private class RegistrySerializer(private val json: Json) : Serializer<RegistrySnapshot> {
    override val defaultValue = RegistrySnapshot()
    override suspend fun readFrom(input: InputStream) =
        runCatching { json.decodeFromString(RegistrySnapshot.serializer(), input.readBytes().decodeToString()) }
            .getOrElse { throw CorruptionException("registry", it) }
    override suspend fun writeTo(t: RegistrySnapshot, output: OutputStream) =
        output.write(json.encodeToString(RegistrySnapshot.serializer(), t).encodeToByteArray())
}
```

### A.4 Hilt custom scoped components

```kotlin
@Scope @Retention(AnnotationRetention.RUNTIME) annotation class AccountScoped
@Scope @Retention(AnnotationRetention.RUNTIME) annotation class PortalScoped

@AccountScoped
@DefineComponent(parent = SingletonComponent::class)
interface AccountComponent

@DefineComponent.Builder
interface AccountComponentBuilder {
    fun account(@BindsInstance account: AccountRecord): AccountComponentBuilder
    fun build(): AccountComponent
}

@PortalScoped
@DefineComponent(parent = AccountComponent::class)
interface PortalComponent

@DefineComponent.Builder
interface PortalComponentBuilder {
    fun portal(@BindsInstance portal: PortalRecord): PortalComponentBuilder
    fun build(): PortalComponent
}

/** Entry points: how non-Hilt code (SessionManager, workers) reaches into a built component. */
@EntryPoint @InstallIn(AccountComponent::class)
interface AccountEntryPoint {
    fun session(): AccountSession
    fun portalBuilder(): PortalComponentBuilder
    fun database(): AccountDatabase
    fun portalRepository(): PortalRepository
}

@EntryPoint @InstallIn(PortalComponent::class)
interface PortalEntryPoint {
    fun session(): PortalSession
    fun itemRepository(): ItemRepository
}

@AccountScoped
class AccountSession @Inject constructor(val account: AccountRecord) {
    val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    val id get() = account.id
    @Volatile var isActive = true; private set
    fun close() { isActive = false; scope.cancel("account session closed") }
}

@PortalScoped
class PortalSession @Inject constructor(val account: AccountSession, val portal: PortalRecord) {
    val scope = CoroutineScope(SupervisorJob(account.scope.coroutineContext[Job]) + Dispatchers.Default)
    val role = MutableStateFlow(portal.role)
    fun close() = scope.cancel("portal session closed")
}

@Module @InstallIn(AccountComponent::class)
object AccountModule {
    @Provides @AccountScoped
    fun database(@ApplicationContext ctx: Context, s: AccountSession): AccountDatabase =
        Room.databaseBuilder(ctx, AccountDatabase::class.java, "acct_${s.id.fileKey}.db")
            .fallbackToDestructiveMigrationOnDowngrade(true)
            .build()

    @Provides @AccountScoped
    fun tokenManager(s: AccountSession, store: SecureTokenStore, registry: AccountRegistry,
                     @AuthClient authClient: OkHttpClient, json: Json) =
        TokenManager(s.account, store, registry, authClient, json)

    @Provides @AccountScoped @AccountHttp
    fun okHttp(@BaseHttp base: OkHttpClient, @ApplicationContext ctx: Context,
               s: AccountSession, tm: TokenManager): OkHttpClient =
        base.newBuilder()                                     // shares pool + dispatcher
            .cache(Cache(File(ctx.cacheDir, "accounts/${s.id.fileKey}/http"), 20L * 1024 * 1024))
            .addInterceptor(AuthInterceptor(tm))
            .authenticator(TokenAuthenticator(tm))
            .build()

    @Provides @AccountScoped
    fun retrofit(@AccountHttp client: OkHttpClient, s: AccountSession, json: Json): Retrofit =
        Retrofit.Builder().baseUrl(s.account.baseUrl).client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()
}

@Module @InstallIn(PortalComponent::class)
object PortalModule {
    @Provides @PortalScoped @PortalHttp
    fun okHttp(@AccountHttp client: OkHttpClient, p: PortalSession): OkHttpClient =
        client.newBuilder().addInterceptor { chain ->
            chain.proceed(chain.request().newBuilder().header("X-Portal-Id", p.portal.id).build())
        }.build()

    @Provides @PortalScoped
    fun api(@PortalHttp client: OkHttpClient, retrofit: Retrofit): PortalApi =
        retrofit.newBuilder().client(client).build().create(PortalApi::class.java)
}

@Qualifier annotation class BaseHttp
@Qualifier annotation class AuthClient
@Qualifier annotation class AccountHttp
@Qualifier annotation class PortalHttp
```

> `@DefineComponent` components are not tied to an Android lifecycle — **you** own their lifetime (SessionManager). `@HiltViewModel` cannot inject `@AccountScoped` types directly because `ViewModelComponent` isn't a child of `AccountComponent`; use the `SessionHolder` bridge in A.7.

### A.5 Token manager, interceptor, authenticator

```kotlin
class TokenManager(
    private val account: AccountRecord,
    private val store: SecureTokenStore,
    private val registry: AccountRegistry,
    private val authClient: OkHttpClient,   // no auth interceptors
    private val json: Json,
) {
    private val mutex = Mutex()
    @Volatile private var cached: TokenSet? = null

    suspend fun current(): TokenSet? = cached ?: store.read(account.id).also { cached = it }

    suspend fun validAccessToken(): String? {
        val t = current() ?: return null
        return if (t.accessExpiresAt - 30_000 > System.currentTimeMillis()) t.accessToken
        else refresh(failedToken = t.accessToken)
    }

    /** Single-flight: only one refresh per account; late callers reuse the result. */
    suspend fun refresh(failedToken: String?): String? = mutex.withLock {
        val now = current() ?: return null
        if (failedToken != null && now.accessToken != failedToken &&
            now.accessExpiresAt > System.currentTimeMillis()) return now.accessToken

        when (val r = callRefresh(now.refreshToken)) {
            is RefreshResult.Ok -> {
                store.write(account.id, r.tokens)       // persist rotated refresh token before release
                cached = r.tokens
                r.tokens.accessToken
            }
            RefreshResult.InvalidGrant -> {
                cached = null
                store.delete(account.id)
                registry.setState(account.id, AccountState.NeedsReauth)
                null
            }
            RefreshResult.Transient -> null           // keep tokens; caller surfaces network error
        }
    }

    private sealed interface RefreshResult {
        data class Ok(val tokens: TokenSet) : RefreshResult
        data object InvalidGrant : RefreshResult
        data object Transient : RefreshResult
    }

    private suspend fun callRefresh(refreshToken: String): RefreshResult = withContext(Dispatchers.IO) {
        val req = Request.Builder()
            .url("${account.baseUrl}oauth/token")
            .post(FormBody.Builder().add("grant_type", "refresh_token").add("refresh_token", refreshToken).build())
            .build()
        try {
            authClient.newCall(req).execute().use { resp ->
                when {
                    resp.isSuccessful -> {
                        val body = json.decodeFromString(TokenResponse.serializer(), resp.body!!.string())
                        RefreshResult.Ok(TokenSet(body.access_token,
                            System.currentTimeMillis() + body.expires_in * 1000,
                            body.refresh_token ?: refreshToken))
                    }
                    resp.code == 400 || resp.code == 401 -> RefreshResult.InvalidGrant
                    else -> RefreshResult.Transient
                }
            }
        } catch (e: IOException) { RefreshResult.Transient }
    }
}

@Serializable data class TokenResponse(val access_token: String, val expires_in: Long, val refresh_token: String? = null)

class AuthInterceptor(private val tm: TokenManager) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = runBlocking { tm.validAccessToken() }
        val req = chain.request().newBuilder().apply { token?.let { header("Authorization", "Bearer $it") } }.build()
        return chain.proceed(req)
    }
}

class TokenAuthenticator(private val tm: TokenManager) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.priorResponse != null) return null            // already retried once
        val failed = response.request.header("Authorization")?.removePrefix("Bearer ")
        val fresh = runBlocking { withTimeoutOrNull(15_000) { tm.refresh(failed) } } ?: return null
        return response.request.newBuilder().header("Authorization", "Bearer $fresh").build()
    }
}
```

> Each account component gets its **own** `TokenManager` + interceptor instances, so a request built by A's client can only ever carry A's token — even if it is retried after the user switched to B.

### A.6 Session manager: switch account / switch portal / sign out

```kotlin
sealed interface SessionState {
    data object Loading : SessionState
    data object NoAccounts : SessionState
    data class NeedsReauth(val account: AccountRecord) : SessionState
    data class NoPortal(val account: AccountRecord) : SessionState
    data class Active(val account: AccountRecord, val portal: PortalRecord, val key: String) : SessionState
    data class Switching(val toAccountId: String) : SessionState
}

@Singleton
class SessionManager @Inject constructor(
    private val registry: AccountRegistry,
    private val tokens: SecureTokenStore,
    private val accountBuilder: Provider<AccountComponentBuilder>,
    private val identity: IdentitySync,          // analytics, crash, RevenueCat
    private val cleaner: AccountDataCleaner,
) {
    private val mutex = Mutex()
    private val _state = MutableStateFlow<SessionState>(SessionState.Loading)
    val state: StateFlow<SessionState> = _state

    private var account: AccountComponent? = null
    private var portal: PortalComponent? = null

    fun accountEntry(): AccountEntryPoint? = account?.let { EntryPoints.get(it, AccountEntryPoint::class.java) }
    fun portalEntry(): PortalEntryPoint? = portal?.let { EntryPoints.get(it, PortalEntryPoint::class.java) }

    suspend fun restore() = mutex.withLock {
        cleaner.finishPendingRemovals()
        val s = registry.snapshot()
        val target = s.accounts.firstOrNull { it.id == s.activeAccountId } ?: s.accounts.maxByOrNull { it.lastUsedAt }
        if (target == null) _state.value = SessionState.NoAccounts else activateLocked(target.id, null)
    }

    suspend fun switchAccount(accountId: String, portalId: String? = null) = mutex.withLock {
        if ((state.value as? SessionState.Active)?.account?.id == accountId) {
            if (portalId != null) switchPortalLocked(portalId); return@withLock
        }
        _state.value = SessionState.Switching(accountId)
        teardownAccountLocked()
        activateLocked(accountId, portalId)
    }

    suspend fun switchPortal(portalId: String) = mutex.withLock { switchPortalLocked(portalId) }

    private suspend fun activateLocked(accountId: String, portalId: String?) {
        val record = registry.snapshot().accounts.first { it.id == accountId }
        val hasTokens = try { tokens.read(accountId) != null } catch (e: TokenUnreadable) { false }
        if (!hasTokens || record.state == AccountState.NeedsReauth) {
            registry.setState(accountId, AccountState.NeedsReauth)
            _state.value = SessionState.NeedsReauth(record); return
        }
        registry.setActive(accountId)
        val comp = accountBuilder.get().account(record).build()
        account = comp
        val entry = EntryPoints.get(comp, AccountEntryPoint::class.java)
        val portals = entry.portalRepository().cachedOrFetch()
        val chosen = portals.firstOrNull { it.id == portalId }
            ?: portals.firstOrNull { it.id == record.lastPortalId }
            ?: portals.firstOrNull { it.isDefault }
            ?: portals.firstOrNull()
        identity.onAccount(record)
        if (chosen == null) { _state.value = SessionState.NoPortal(record); return }
        buildPortalLocked(entry, record, chosen)
    }

    private suspend fun switchPortalLocked(portalId: String) {
        val entry = accountEntry() ?: return
        val record = entry.session().account
        val target = entry.portalRepository().cachedOrFetch().firstOrNull { it.id == portalId }
            ?: error("Portal $portalId not in account ${record.id}")
        portalEntry()?.session()?.close()
        portal = null
        buildPortalLocked(entry, record, target)
    }

    private suspend fun buildPortalLocked(entry: AccountEntryPoint, record: AccountRecord, p: PortalRecord) {
        portal = entry.portalBuilder().portal(p).build()
        registry.setLastPortal(record.id, p.id)
        identity.onPortal(p)
        _state.value = SessionState.Active(record, p, key = "${record.id}|${p.id}")
    }

    private fun teardownAccountLocked() {
        portalEntry()?.session()?.close()
        portal = null
        accountEntry()?.let { e ->
            e.session().close()           // cancels requests, collectors, sockets bound to scope
            e.database().close()
        }
        account = null
    }

    /** Portal revoked (403 membership, push, or list refresh). */
    suspend fun onPortalRevoked(accountId: String, portalId: String) = mutex.withLock {
        val entry = accountEntry()
        cleaner.purgePortal(accountId, portalId)
        if (entry?.session()?.id == accountId) {
            entry.portalRepository().remove(portalId)
            val active = state.value as? SessionState.Active
            if (active?.portal?.id == portalId) {
                val next = entry.portalRepository().cachedOrFetch().maxByOrNull { it.lastUsedAt }
                portalEntry()?.session()?.close(); portal = null
                if (next == null) _state.value = SessionState.NoPortal(active.account)
                else buildPortalLocked(entry, active.account, next)
            }
        }
    }

    suspend fun signOut(accountId: String) = mutex.withLock {
        registry.setState(accountId, AccountState.Removing)
        val wasActive = accountEntry()?.session()?.id == accountId
        if (wasActive) teardownAccountLocked()
        cleaner.remove(accountId)             // revoke, unregister push, wipe files/DB/tokens, registry
        val next = registry.nextActiveAfterRemoval(registry.snapshot(), accountId)
        when {
            next == null -> { identity.onSignedOutAll(); _state.value = SessionState.NoAccounts }
            wasActive -> activateLocked(next.id, null)
        }
    }

    suspend fun signOutAll() {
        registry.snapshot().accounts.forEach { signOut(it.id) }
    }
}
```

### A.7 Bridging to ViewModels and navigation

```kotlin
/** Thin bridge so @HiltViewModel can reach the active portal graph at construction time. */
@Singleton
class SessionHolder @Inject constructor(private val sm: SessionManager) {
    fun portal(): PortalEntryPoint = sm.portalEntry() ?: error("No active portal session")
}

@HiltViewModel
class ItemsViewModel @Inject constructor(holder: SessionHolder) : ViewModel() {
    private val portal = holder.portal()                       // captured once; VM dies on switch
    private val repo = portal.itemRepository()
    val items = repo.items().stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
    val canEdit = portal.session().role.map { it == Role.Owner || it == Role.Admin }
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)
}

@Composable
fun AppRoot(sessionManager: SessionManager) {
    val state by sessionManager.state.collectAsStateWithLifecycle()
    when (val s = state) {
        SessionState.Loading, is SessionState.Switching -> SwitchingOverlay()
        SessionState.NoAccounts -> LoginGraph()
        is SessionState.NeedsReauth -> ReauthScreen(s.account)
        is SessionState.NoPortal -> NoPortalScreen(s.account)
        is SessionState.Active -> key(s.key) {                  // new NavHost + ViewModelStores per pair
            AppNavHost(startRoute = rememberLastRoute(s.account.id, s.portal.id))
        }
    }
}
```

> Keying the `NavHost` by `"$accountId|$portalId"` discards all back stack entries and their ViewModels on switch, which is exactly the teardown you want. If the product needs account-level screens to survive a portal switch, key the outer host by account and an inner portal graph by portal.

### A.8 Data cleaner

```kotlin
@Singleton
class AccountDataCleaner @Inject constructor(
    @ApplicationContext private val ctx: Context,
    private val registry: AccountRegistry,
    private val tokens: SecureTokenStore,
    private val remote: AuthRemote,              // revoke + push unregister, unauthenticated-safe
) {
    suspend fun remove(accountId: String) = withContext(Dispatchers.IO) {
        val h = accountId.fileKey
        runCatching { withTimeout(5_000) { remote.revokeAndUnregister(accountId) } }   // best effort
        WorkManager.getInstance(ctx).cancelAllWorkByTag("acct:$accountId")
        ctx.deleteDatabase("acct_$h.db")
        File(ctx.filesDir, "accounts/$h").deleteRecursively()
        File(ctx.cacheDir, "accounts/$h").deleteRecursively()
        File(ctx.filesDir, "datastore/acct_${h}_prefs.preferences_pb").delete()
        NotificationManagerCompat.from(ctx).run {
            activeNotifications.filter { it.tag?.startsWith("acct:$accountId") == true }
                .forEach { cancel(it.tag, it.id) }
            deleteNotificationChannelGroup("acct_$h")
        }
        tokens.delete(accountId)
        registry.remove(accountId)                // last: registry entry gone = cleanup complete
    }

    suspend fun purgePortal(accountId: String, portalId: String) = withContext(Dispatchers.IO) {
        File(ctx.filesDir, "accounts/${accountId.fileKey}/portals/${portalId.fileKey}").deleteRecursively()
        WorkManager.getInstance(ctx).cancelAllWorkByTag("portal:$portalId")
        // Rows: done by the account DB if open, otherwise by a short-lived DB instance / worker.
    }

    suspend fun finishPendingRemovals() =
        registry.snapshot().accounts.filter { it.state == AccountState.Removing }.forEach { remove(it.id) }
}
```

### A.9 Push routing (FCM)

```kotlin
class AppMessagingService : FirebaseMessagingService() {
    @Inject lateinit var registry: AccountRegistry
    @Inject lateinit var pushRegistrar: PushRegistrar
    @Inject lateinit var notifier: AccountNotifier

    override fun onNewToken(token: String) {
        pushRegistrar.enqueueRegisterAll(token)  // WorkManager: one job per account with its own auth
    }

    override fun onMessageReceived(msg: RemoteMessage) {
        val accountId = msg.data["accountId"] ?: return
        val portalId = msg.data["portalId"]
        val account = runBlocking { registry.snapshot() }.accounts.firstOrNull { it.id == accountId }
        if (account == null || account.state == AccountState.Removing) {
            pushRegistrar.enqueueUnregister(accountId); return
        }
        if (msg.data["type"] == "portal_removed" && portalId != null) {
            pushRegistrar.enqueuePortalRevoked(accountId, portalId); return
        }
        notifier.show(account, portalId, msg.data)
    }
}

class AccountNotifier @Inject constructor(@ApplicationContext private val ctx: Context) {
    fun show(account: AccountRecord, portalId: String?, data: Map<String, String>) {
        val route = data["route"] ?: "home"
        val intent = Intent(ctx, MainActivity::class.java).apply {
            action = "open_${account.id}_${portalId}_$route"          // unique → extras not merged
            putExtra(EXTRA_ACCOUNT, account.id); putExtra(EXTRA_PORTAL, portalId); putExtra(EXTRA_ROUTE, route)
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        }
        val notifId = (data["id"] ?: route).hashCode()
        val pi = PendingIntent.getActivity(ctx, notifId, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
        val n = NotificationCompat.Builder(ctx, "acct_${account.id.fileKey}_messages")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(data["title"]).setContentText(data["body"])
            .setSubText(account.email)                                    // which account
            .setGroup("acct:${account.id}")
            .setVisibility(NotificationCompat.VISIBILITY_PRIVATE)
            .setContentIntent(pi).setAutoCancel(true).build()
        if (ActivityCompat.checkSelfPermission(ctx, Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED ||
            Build.VERSION.SDK_INT < 33) {
            NotificationManagerCompat.from(ctx).notify("acct:${account.id}", notifId, n)
        }
    }
}
```

### A.10 Push tap / deep link → account + portal

```kotlin
sealed interface OpenTarget {
    data class Direct(val route: String) : OpenTarget
    data class NeedsSwitch(val account: AccountRecord, val portalId: String?, val route: String) : OpenTarget
    data class ChooseAccount(val candidates: List<AccountRecord>, val portalId: String, val route: String) : OpenTarget
    data class NoAccess(val portalId: String) : OpenTarget
}

class EntryResolver @Inject constructor(private val registry: AccountRegistry, private val sm: SessionManager) {
    suspend fun resolve(accountId: String?, portalId: String?, route: String): OpenTarget {
        val snap = registry.snapshot()
        val active = sm.state.value as? SessionState.Active
        val candidates = when {
            accountId != null -> snap.accounts.filter { it.id == accountId }
            portalId != null -> snap.accounts.filter { a -> snap.portals[a.id].orEmpty().any { it.id == portalId } }
            else -> listOfNotNull(active?.account)
        }
        return when {
            candidates.isEmpty() -> OpenTarget.NoAccess(portalId.orEmpty())
            active != null && candidates.any { it.id == active.account.id } &&
                (portalId == null || portalId == active.portal.id) -> OpenTarget.Direct(route)
            candidates.size == 1 -> OpenTarget.NeedsSwitch(candidates.single(), portalId, route)
            else -> OpenTarget.ChooseAccount(candidates, portalId!!, route)
        }
    }
}

// In MainActivity / a root ViewModel:
suspend fun handleOpen(target: OpenTarget, confirm: suspend (String) -> Boolean, navigate: (String) -> Unit) {
    when (target) {
        is OpenTarget.Direct -> navigate(target.route)
        is OpenTarget.NeedsSwitch -> {
            val active = sm.state.value as? SessionState.Active
            val sameAccount = active?.account?.id == target.account.id
            if (sameAccount || confirm("Open as ${target.account.email}?")) {
                sm.switchAccount(target.account.id, target.portalId)     // handles portal-only switch too
                sm.state.first { it is SessionState.Active || it is SessionState.NeedsReauth }
                if (sm.state.value is SessionState.Active) navigate(target.route) else pendingRoute = target.route
            }
        }
        is OpenTarget.ChooseAccount -> showAccountChooser(target)
        is OpenTarget.NoAccess -> showNoAccess(target.portalId)
    }
}
```

### A.11 Account-scoped worker

```kotlin
@HiltWorker
class PortalSyncWorker @AssistedInject constructor(
    @Assisted ctx: Context, @Assisted params: WorkerParameters,
    private val registry: AccountRegistry,
    private val accountBuilder: Provider<AccountComponentBuilder>,
    private val sm: SessionManager,
) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        val accountId = inputData.getString("accountId") ?: return Result.failure()
        val portalId = inputData.getString("portalId") ?: return Result.failure()
        val account = registry.snapshot().accounts.firstOrNull { it.id == accountId } ?: return Result.failure()
        if (account.state != AccountState.Active) return Result.failure()

        // Reuse the live graph if this account is active; otherwise build a temporary one.
        val live = sm.accountEntry()?.takeIf { it.session().id == accountId }
        val comp = if (live == null) accountBuilder.get().account(account).build() else null
        val entry = live ?: EntryPoints.get(comp!!, AccountEntryPoint::class.java)
        return try {
            entry.portalRepository().syncUnread(portalId)
            Result.success()
        } catch (e: IOException) { Result.retry() }
        finally { if (comp != null) { entry.session().close(); entry.database().close() } }
    }

    companion object {
        fun enqueue(wm: WorkManager, accountId: String, portalId: String) =
            wm.enqueueUniqueWork(
                "sync_${accountId.fileKey}_${portalId.fileKey}", ExistingWorkPolicy.KEEP,
                OneTimeWorkRequestBuilder<PortalSyncWorker>()
                    .setInputData(workDataOf("accountId" to accountId, "portalId" to portalId))
                    .addTag("acct:$accountId").addTag("portal:$portalId")
                    .setConstraints(Constraints(requiredNetworkType = NetworkType.CONNECTED))
                    .build(),
            )
    }
}
```

> Opening a second `RoomDatabase` for the active account's file from the worker is avoided by reusing the live graph; if workers run in another process, enable `enableMultiInstanceInvalidation()`.

### A.12 Identity sync (analytics, crash, RevenueCat)

```kotlin
@Singleton
class IdentitySync @Inject constructor() {
    fun onAccount(a: AccountRecord) {
        val opaque = a.id.fileKey                                  // no email / PII
        Firebase.crashlytics.setUserId(opaque)
        Firebase.analytics.setUserId(opaque)
        Purchases.sharedInstance.logInWith(a.serverUserId, onError = { /* log, keep going */ })
    }
    fun onPortal(p: PortalRecord) {
        Firebase.analytics.setUserProperty("portal_id", p.id.fileKey)
        Firebase.analytics.setUserProperty("portal_role", p.role.name)
        Firebase.crashlytics.setCustomKey("portal_role", p.role.name)
    }
    fun onSignedOutAll() {
        Firebase.crashlytics.setUserId("")
        Firebase.analytics.setUserId(null)
        if (!Purchases.sharedInstance.isAnonymous) Purchases.sharedInstance.logOutWith()
    }
}
```

### A.13 Unit test: single-flight refresh

```kotlin
@Test fun concurrent401s_triggerSingleRefresh() = runTest {
    val server = MockWebServer().apply {
        dispatcher = object : Dispatcher() {
            val refreshes = AtomicInteger()
            override fun dispatch(r: RecordedRequest) = when (r.path) {
                "/oauth/token" -> { refreshes.incrementAndGet()
                    MockResponse().setBody("""{"access_token":"new","expires_in":3600,"refresh_token":"r2"}""") }
                else -> if (r.getHeader("Authorization") == "Bearer new") MockResponse().setBody("{}")
                        else MockResponse().setResponseCode(401)
            }
        }
        start()
    }
    val account = testAccount(baseUrl = server.url("/").toString())
    val store = FakeTokenStore(account.id, TokenSet("old", Long.MAX_VALUE, "r1"))
    val tm = TokenManager(account, store, FakeRegistry(), OkHttpClient(), Json)
    val client = OkHttpClient.Builder().addInterceptor(AuthInterceptor(tm)).authenticator(TokenAuthenticator(tm)).build()

    (1..20).map { async(Dispatchers.IO) { client.newCall(Request.Builder().url(server.url("/items")).build()).execute().use { it.code } } }
        .awaitAll().forEach { assertEquals(200, it) }

    assertEquals(1, server.requestCount { it.path == "/oauth/token" })
    assertEquals("r2", store.current?.refreshToken)
}
```

(`FakeTokenStore`, `FakeRegistry` and `requestCount` are small test helpers; extract an interface from `SecureTokenStore`/`AccountRegistry` so `TokenManager` can take fakes.)
