---
name: ios-network-offline-sync
description: Use when implementing, reviewing, or debugging networking, caching, offline support and data sync in an iOS/iPadOS app in Swift — a single configured URLSession-based API client with async/await, environment/region base URLs, timeouts and an ordered interceptor chain, auth headers with single-flight token refresh and one 401 retry, per-account clients, a typed error model mapped to user-facing messages, retries with exponential backoff + jitter, idempotency keys and Retry-After, cancellation tied to view/task lifetime, in-flight GET dedupe, URLCache and ETag/If-None-Match revalidation, NWPathMonitor connectivity hints, offline-first repositories with SwiftData/Core Data as the source of truth, an outbox for queued writes with optimistic UI, cursor/delta sync with tombstones, conflict resolution, BGTaskScheduler and silent-push-triggered sync, URLSessionWebSocketTask realtime with catch-up, background URLSession uploads/downloads, TLS/pinning, store migrations, and a short Apollo iOS GraphQL section. Siblings: android-network-offline-sync, react-native-network-offline-sync, flutter-network-offline-sync; related: ios-multi-account, ios-push-notifications, ios-list-ux, ios-crash-analytics, ios-deep-links, ios-in-app-purchase.
---

# iOS Network, Offline & Sync Skill

Guide for building and auditing the data layer of a consumer iOS app: how requests are made, how failures are classified and retried, how data is cached and kept usable offline, and how local edits get back to the server without loss or duplication. REST is the primary model; GraphQL (Apollo iOS) gets a short section (18). Covers Swift 5.9+/Swift 6 strict concurrency, `URLSession` async/await, `URLCache`, `Network.framework` (`NWPathMonitor`), SwiftData and Core Data, `BGTaskScheduler`, background `URLSession`, `URLSessionWebSocketTask`.

Siblings: **android-network-offline-sync**, **react-native-network-offline-sync**, **flutter-network-offline-sync** — keep the **server contract** identical across platforms (error envelope, idempotency-key header, sync cursor format, tombstone shape, conflict response `409` body, realtime event ids) so the backend serves one model. Related: **ios-multi-account** (one client, store and outbox per account), **ios-push-notifications** (silent push → sync), **ios-list-ux** (lists observing the DB, pagination, pull-to-refresh), **ios-crash-analytics** (non-fatal network errors, network traces, breadcrumbs), **ios-deep-links** (link opens offline → show cached or pending state), **ios-in-app-purchase** (receipt/transaction posts must use the outbox with idempotency).

Networking and persistence APIs move between OS releases (SwiftData `#Unique`/`#Index` and history in iOS 18, `URLSession` async APIs iOS 15, `NWPathMonitor` iOS 12, `BGContinuedProcessingTask` iOS 26, Apollo iOS 1.x → 2.x API changes, Swift 6 `Sendable` checks on delegates). **Verify fast-moving APIs and exact signatures against current Apple / Apollo docs before shipping**, and gate with `if #available` when the deployment target is lower.

Key gates (verify): `URLSession.data(for:)`, `bytes(for:)`, `upload(for:from:)` iOS 15; `URLSessionWebSocketTask` iOS 13; `NWPathMonitor` iOS 12; `BGTaskScheduler` iOS 13; SwiftData + `@Observable` iOS 17; `#Unique`, `#Index`, `HistoryDescriptor` iOS 18; `URLCache(memoryCapacity:diskCapacity:directory:)` iOS 13; `URLSessionConfiguration.requiresDNSSECValidation` iOS 16; resumable upload (`HTTP 104` / `resumeData` for uploads) iOS 17; `NSPersistentCloudKitContainer` iOS 13; `BGContinuedProcessingTask` iOS 26.

## 0. Mental model

```
 SwiftUI View ──observes──► @Query / @Observable ViewModel ──reads──► Local DB (SwiftData / Core Data)
      │                                                                   ▲      ▲
      │ user edit                                                         │      │ apply pages,
      ▼                                                                   │      │ tombstones
 Repository ──(1) write local + optimistic──────────────────────────────┘      │
      │     ──(2) enqueue OutboxItem (same transaction)──► Outbox table           │
      │                                                     │                     │
      │                                       OutboxProcessor (actor, serial)     │
      │                                                     │ ordered replay      │
      ▼                                                     ▼                     │
 APIClient (one per account/env)  ◄────────────── SyncEngine (actor) ─────────────┘
   ├─ interceptors: request-id → auth → idempotency → logging
   ├─ retry policy (backoff+jitter, Retry-After, idempotent only)
   ├─ single-flight TokenProvider (401 → refresh once → retry once)
   ├─ in-flight GET dedupe
   ├─ URLSession (ephemeral-ish config, own URLCache, timeouts)
   └─ typed APIError
          ▲
 Triggers: app foreground · NWPathMonitor "satisfied" · BGAppRefresh/BGProcessing ·
           silent push · WebSocket event · pull-to-refresh · outbox enqueue
```

Rules of thumb:

- **The UI never renders network responses directly** for offline-capable data. Network → DB → UI. One path, so offline, cache and fresh look identical.
- **Every write is local first**, then queued. The outbox is the only thing that sends mutations.
- **Reachability is a hint.** The only proof of connectivity is a request that succeeded. Never block a request because `NWPathMonitor` says "unsatisfied" — try, and fail fast.
- **Retries are safe only when the server can dedupe** (GET/PUT/DELETE, or POST with an idempotency key).
- **Every in-flight unit of work is scoped** (account, screen, task) and cancelled with its scope.

## 1. First, establish context

Find out (from the codebase or by asking):

- **API style**: REST JSON? GraphQL? gRPC? Mixed? Versioning (`/v2`, header, date-based)? Error envelope shape? Pagination style (cursor vs offset)?
- **Freshness**: how stale may each resource be (feed: minutes; profile: hours; prices/stock/balance: must be live)? Which screens must show "as of" timestamps?
- **Offline mode**: none (just good errors) / **read-only** (cached reads) / **read-write** (edits queued). Per feature — chat and notes are read-write; checkout is online-only.
- **Conflict tolerance**: can two devices edit the same record? Which fields are server-authoritative (status, price, counters)? Is last-write-wins acceptable to product?
- **Volume**: records per user (100 vs 1M), payload sizes, attachments/media, sync frequency. Drives full vs delta sync and DB choice.
- **Auth**: OAuth/OIDC with rotating refresh tokens? Cookies? Per-account tokens (→ **ios-multi-account**)?
- **Backends**: environments (dev/staging/prod) and regions (EU/US) — chosen at build time or per account at login?
- **Realtime**: WebSocket / SSE / push-only / polling? Event ids for catch-up?
- **Background**: must data be fresh on open (BG refresh), must uploads finish after the app is killed (background URLSession)?
- **Server support**: ETags? `Idempotency-Key`? `updatedSince` / change feed with tombstones? `409` with current version? `Retry-After`?
- **Min iOS** and persistence stack already in use (SwiftData, Core Data, GRDB, Realm).

Write the answers down as a short spec; the rest of the checklist depends on it.

## 2. HTTP client architecture

- [ ] **One configured `APIClient`** (actor or `Sendable` final class) per backend/account, injected — no `URLSession.shared` for API calls, no ad-hoc `URLSession(configuration:)` per request (each session = own connection pool, cache, TLS handshakes).
- [ ] **Base URL from an `Environment`** value (dev/staging/prod × region), chosen at build time via xcconfig/Info.plist or at login from the account's region. Never string-concatenate paths; use `URLComponents` / `appending(path:)` and percent-encode query items.
- [ ] **`URLSessionConfiguration`** (start from `.default`):
  - `timeoutIntervalForRequest` = 15–30 s (idle time between packets, not total); `timeoutIntervalForResource` = 60–120 s for API calls (total); longer for transfers.
  - `waitsForConnectivity = false` for interactive calls (fail fast, show offline UI); `true` only for fire-and-forget calls where waiting is fine. With `true`, `timeoutIntervalForResource` bounds the wait.
  - `allowsExpensiveNetworkAccess` / `allowsConstrainedNetworkAccess` (Low Data Mode) — respect for prefetch/media; keep `true` for user-initiated calls.
  - `httpAdditionalHeaders` for static headers only (`Accept`, `User-Agent`, `Accept-Language`, app version). Never auth headers (they change).
  - Own `URLCache` (section 7) and `httpCookieStorage` (nil if you don't use cookies — prevents surprise auth state).
  - `httpMaximumConnectionsPerHost` default is fine; HTTP/2 and HTTP/3 multiplex.
- [ ] **Interceptor chain in a fixed, documented order** (outermost first):
  1. Request id (`X-Request-Id: UUID`) — same id across retries of one logical call for server log correlation.
  2. Auth (attach `Authorization` from `TokenProvider`; on 401 refresh + retry once — section 3).
  3. Idempotency key for unsafe methods (stable across retries — section 5).
  4. Retry policy (wraps the send; re-runs auth per attempt so a refreshed token is used).
  5. Logging/metrics (redact `Authorization`, cookies, PII in bodies; `os.Logger` with `privacy: .private`).
  - Response side: status → typed error mapping, then decoding.
- [ ] **Decoding off the main actor** (`JSONDecoder` inside the client actor / nonisolated function). Configure date strategy once (ISO8601 with fractional seconds — write a custom strategy; `.iso8601` rejects fractional seconds).
- [ ] **Endpoints are typed values** (`Endpoint<Response>` with method, path, query, body, `isIdempotent`, cache policy) — not stringly-typed calls scattered across view models.
- [ ] `Sendable` everything that crosses the client boundary; DTOs are `Codable, Sendable` structs, never SwiftData `@Model` objects.
- [ ] Use `URLSession.data(for:)` async APIs; with a delegate, use the per-task `delegate:` parameter (iOS 15) for metrics/auth challenges instead of session-wide mutable state.
- [ ] Collect `URLSessionTaskMetrics` (via task delegate) for network traces → **ios-crash-analytics**.

## 3. Auth

- [ ] **Auth header added per attempt** by the auth interceptor from a `TokenProvider` actor; never stored in `httpAdditionalHeaders` or a global mutable header.
- [ ] **Single-flight refresh**: concurrent requests that hit 401 (or see the token expiring within 30–60 s) await **one** refresh `Task`; others reuse its result. Prevents rotating-refresh-token races that log users out.
- [ ] **401 → refresh → retry the original request once.** A second 401 after a fresh token → `APIError.unauthorized` → session expired flow. Never loop.
- [ ] Only retry if the request was sent with the **old** token (compare token used vs current) — a request that 401s with an already-refreshed token must not trigger another refresh.
- [ ] Refresh failure semantics: `invalid_grant`/401 on refresh → signed out / `needsReauth`; **network error on refresh → keep tokens**, surface offline, retry later. Never sign out because the device is offline.
- [ ] 403 ≠ 401 — permission/membership problem, no refresh.
- [ ] Persist rotated refresh tokens to Keychain **before** using the new access token (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` so BG sync can read it).
- [ ] **Per-account clients**: each signed-in account gets its own `APIClient`, `TokenProvider`, `URLCache`, cookie store, DB and outbox; the outbox processor of account A never sends with B's token. Responses carry the scope they were requested in; late responses for a torn-down scope are dropped (→ **ios-multi-account**).
- [ ] Refresh endpoint calls bypass the auth interceptor (no recursion) and the retry interceptor's 401 handling.

## 4. Error model

- [ ] **One typed error** for the whole data layer, e.g.:
  - `.offline` (`URLError.notConnectedToInternet`, `.networkConnectionLost`, `.dataNotAllowed`, `.internationalRoamingOff`)
  - `.timeout` (`URLError.timedOut`)
  - `.cancelled` (`URLError.cancelled`, `CancellationError`) — **never shown to the user, never retried, never logged as an error**
  - `.tls` (`.serverCertificateUntrusted`, `.secureConnectionFailed`, pinning failure)
  - `.http(status:, serverCode:, message:, retryAfter:)` — parsed from the error envelope
  - `.unauthorized`, `.forbidden`, `.notFound`, `.conflict(serverVersion:)`, `.rateLimited(retryAfter:)`, `.server(status)` (5xx)
  - `.decoding(DecodingError, endpoint)` — a bug or contract drift; report non-fatal with endpoint + path, no body
- [ ] Map **everything** at the client boundary; view models never see `URLError` or `DecodingError`.
- [ ] `isRetryable` and `isTransient` computed properties on the error drive the retry policy and outbox.
- [ ] **User-facing mapping** in one place (`APIError.userMessage` / localized): offline → "You're offline. Changes will sync when you reconnect." (if queued) vs "Connect to the internet to …" (if online-only); 5xx → "Something went wrong on our side"; server code with a localized message → show it if the server guarantees safe text. Never show raw status codes, `localizedDescription` of URLError, or JSON.
- [ ] Distinguish **blocking** (full-screen error with retry, no cached data) vs **non-blocking** (toast/banner over cached data) presentation.
- [ ] Log with request id; report 5xx/decoding as non-fatals, sample offline/timeout (they're noise).

## 5. Retries

- [ ] **Exponential backoff with full jitter**: `delay = random(0, min(cap, base * 2^attempt))`, base 0.5–1 s, cap 30 s (interactive) / 5–15 min (background/outbox). Max attempts 3 for interactive.
- [ ] Retry only **transient** failures: timeout, connection lost, 408, 429, 502, 503, 504 (and 500 only for idempotent requests if the server says it is safe). Never retry 400/401(handled separately)/403/404/409/422, decoding errors, or cancellation.
- [ ] **Safe methods only** (GET, HEAD, PUT, DELETE) — or POST/PATCH **with an `Idempotency-Key`** the server honours. Key generated once per logical operation (store it in the outbox row), reused on every retry and across app launches.
- [ ] **`Retry-After`** header (seconds or HTTP-date) wins over computed backoff on 429/503; clamp to a sane max.
- [ ] **Circuit breaker** per host: after N consecutive 5xx/timeouts, open for 30–60 s — fail fast with `.server` instead of hammering; half-open probe. Especially important for outbox replay and BG sync.
- [ ] Retry loops use `Task.sleep(for:)` and check `Task.isCancelled` / `try Task.checkCancellation()` each attempt.
- [ ] Don't stack retries: the outbox retries across time; the client retries a few times within one attempt. Make the outbox's per-send client call use `maxAttempts = 1` or 2.
- [ ] Don't retry when offline per a fresh path update — park until reconnect (section 8) instead of burning attempts.

## 6. Cancellation

- [ ] **Lifecycle-scoped**: SwiftUI `.task(id:)` cancels automatically on disappear/id change; UIKit stores `Task` handles and cancels in `viewDidDisappear`/`deinit`. Avoid unstructured `Task {}` that outlives the screen for screen-only loads.
- [ ] `URLSession` async APIs honour task cancellation (they cancel the underlying `URLSessionTask`); custom work must check `Task.isCancelled`.
- [ ] Map cancellation to `.cancelled` and **swallow it silently** in the UI (no error alert when the user navigates away).
- [ ] **In-flight GET dedupe**: identical GETs (same URL + auth scope) share one `Task`; second caller awaits the first. Key by method+URL+account. Remove the entry when done. Don't dedupe across accounts.
- [ ] Careful: if caller A cancels a deduped task, caller B must not be cancelled — run the shared task unstructured and let each caller cancel only its own await, or refcount.
- [ ] **Search-as-you-type**: debounce (250–400 ms) via `.task(id: query)` + `try await Task.sleep`, which cancels the previous request automatically; also drop responses whose query ≠ current query.
- [ ] Writes are **not** cancelled by leaving the screen — they belong to the outbox, not the view.
- [ ] Account switch / sign out cancels all of that account's tasks (client, sync, websocket) — **ios-multi-account**.

## 7. Caching

- [ ] **Own `URLCache`** per account (`URLCache(memoryCapacity: 10 MB, diskCapacity: 100 MB, directory: <account caches dir>)`). Responses > ~5% of disk capacity are not cached. `URLCache` only stores responses whose headers allow it.
- [ ] Prefer **server cache headers** (`Cache-Control: max-age`, `ETag`, `Last-Modified`) with `.useProtocolCachePolicy`. With `ETag`, `URLSession` sends `If-None-Match` automatically for cached entries and transparently turns `304` into the cached `200` — so you usually don't see 304s.
- [ ] Manual ETag handling (store `ETag` in your DB row, send `If-None-Match`, handle `304` → "unchanged") when you cache in the DB instead of `URLCache` — set `cachePolicy = .reloadIgnoringLocalCacheData` so `URLCache` doesn't interfere.
- [ ] **Stale-while-revalidate** at the repository: emit DB/cached value immediately, fetch in background, write to DB, UI updates via observation. Show "Updated 5 min ago" when stale beyond the freshness budget.
- [ ] `.returnCacheDataElseLoad` / `.returnCacheDataDontLoad` for an offline fallback of non-DB resources (e.g. config, static content).
- [ ] **Memory vs persistent**: in-memory `NSCache`/dictionary for hot decoded objects (bounded, cleared on memory warning and account switch); DB for offline-relevant domain data; `URLCache` for HTTP-level reuse; images in a dedicated image cache (Nuke/Kingfisher/SDWebImage or `AsyncImage` + `URLCache`).
- [ ] Never cache authenticated responses in `URLCache.shared` (cross-account leak); exclude sensitive endpoints (`Cache-Control: no-store` from server or `willCacheResponse` returning nil).
- [ ] Caches live in `Library/Caches` (purgeable, excluded from backup); DB in `Application Support` (survives, backed up unless excluded).
- [ ] Cache keys include query params and `Accept-Language` if responses are localized (or server sends `Vary`).

## 8. Connectivity

- [ ] **`NWPathMonitor`** on a dedicated queue; publish `status`, `isExpensive`, `isConstrained` via an `@Observable` / `AsyncStream`. Never use the deprecated SCNetworkReachability.
- [ ] **Reachability is a hint, not truth**: `satisfied` can mean captive portal, VPN down, DNS broken, server down. `unsatisfied` can be momentarily wrong. Always attempt user-initiated requests; base offline UI on **actual request failures** + path status combined.
- [ ] **Offline banner**: shown when path is unsatisfied **or** the last N requests failed with `.offline`; hidden on the first successful request. Debounce (1–2 s) to avoid flicker on network handover.
- [ ] **Retry on reconnect**: transition unsatisfied → satisfied triggers outbox drain + stale-screen refresh (debounced, jittered so a fleet of devices doesn't stampede after an outage).
- [ ] Also trigger on `scenePhase == .active` — `NWPathMonitor` updates are not delivered while suspended.
- [ ] Respect `isConstrained` (Low Data Mode): skip prefetch, image preloading, video autoplay; keep user actions.
- [ ] Don't use `waitsForConnectivity` + a spinner for interactive calls — users stare at a spinner for the full resource timeout.

## 9. Offline-first architecture

- [ ] **Local DB is the source of truth** for offline-capable features. Repositories expose observation (`@Query`, `FetchedResultsController`, `NSFetchedResultsController`/`AsyncStream`), never `async -> [Item]` straight from the network.
- [ ] `refresh()` methods fetch and **write to DB**; their return value is only success/failure.
- [ ] **Platform DB options**:
  - **SwiftData** (iOS 17+): `@Model`, `@Query` in SwiftUI, `ModelActor` for background writes. iOS 18 adds `#Unique` (upsert by server id) and history. Simple, but young — check performance with 10k+ rows and complex predicates; no fine-grained batch APIs.
  - **Core Data**: mature, `NSPersistentContainer`, background contexts (`performBackgroundTask` / `newBackgroundContext()` with `automaticallyMergesChangesFromParent`), `NSBatchInsertRequest`/`NSBatchDeleteRequest` for big syncs, persistent history tracking, constraints with `NSMergeByPropertyObjectTrumpMergePolicy` for upserts.
  - **GRDB/SQLite**: full SQL control, great for large volumes and FTS; manual observation (`ValueObservation`).
  - `NSPersistentCloudKitContainer` / SwiftData CloudKit sync — only for user-private data synced via iCloud, not your own backend.
- [ ] **Upsert by server id** (`#Unique<Item>([\.serverId])` or unique constraint); local-only records get a client UUID that the server accepts as id (preferred) or maps to a server id on create.
- [ ] Every syncable row has: `serverId`/`clientId`, `version` or `updatedAt` (server), `syncState` (`synced`, `pendingCreate`, `pendingUpdate`, `pendingDelete`, `failed`), `deletedAt` (soft delete locally until confirmed).
- [ ] **Writes off the main actor** (a `@ModelActor` / background context); UI reads on main. Never pass `@Model` instances across actors — pass `PersistentIdentifier` / server id.
- [ ] Network DTO ≠ DB model ≠ view state. Map explicitly.
- [ ] Pagination with DB: append pages into the DB; list observes DB sorted by server sort key; keep a `nextCursor` per list in a metadata table (→ **ios-list-ux**).
- [ ] Data protection: DB file with `NSFileProtectionCompleteUntilFirstUserAuthentication` (default) so BG sync can open it while locked; `.complete` breaks BG sync.

## 10. Outbox for writes

- [ ] **Local change and outbox row are written in the same DB transaction** — otherwise a crash between them loses the mutation or leaves an orphaned optimistic change.
- [ ] `OutboxItem`: `id`, `createdAt`, `entityType`, `entityId`, `operation` (create/update/delete/custom), `payload` (JSON, minimal — the changed fields or full desired state), `idempotencyKey` (UUID, generated once), `attempts`, `nextAttemptAt`, `lastError`, `state` (`pending`, `inFlight`, `failed`, `blocked`), `dependsOn` (optional).
- [ ] **Optimistic UI**: the entity reflects the change immediately with a subtle "pending" indicator; failure shows "Not sent — Retry / Discard" on the item, not a global alert.
- [ ] **Ordered replay**: process FIFO **per entity** (create before update before delete of the same record); independent entities may go in parallel but serial is simpler and usually fast enough. A create whose server id is needed by a child (comment on an unsynced post) → `dependsOn` or client-generated ids.
- [ ] **Coalesce** before sending: several updates to one entity → one; create + delete of an unsynced entity → drop both.
- [ ] **Idempotency**: every send carries `Idempotency-Key`; if the app dies after the server committed but before the response, the replay is deduped server-side.
- [ ] **Per-item failure states**:
  - transient (offline, timeout, 5xx, 429) → `pending`, backoff via `nextAttemptAt`, keep order.
  - permanent (400/422 validation, 404 on update, 403) → `failed`, revert or mark the entity, surface on the item, **don't block** unrelated items (block only items that depend on it).
  - 409 conflict → conflict resolution (section 12).
  - 401 → pause the whole outbox until reauth (don't mark items failed).
- [ ] Mark `inFlight` before sending; on launch, reset stale `inFlight` → `pending` (the key makes the resend safe).
- [ ] Apply the server response (canonical entity, server id, version) to the DB and delete the outbox row in **one transaction**.
- [ ] Single processor per account (actor + "isRunning" guard); triggers: enqueue, reconnect, foreground, BG task, push.
- [ ] Cap and age-out: alert the user about items failing for > N days; never silently drop.
- [ ] Sign out: warn if the outbox has unsent items ("3 changes haven't synced").

## 11. Sync

- [ ] **Pull with a cursor**: `GET /changes?cursor=<opaque>&limit=500` → `{ changes: [...], deleted: [ids], nextCursor, hasMore }`. Loop until `hasMore == false`. Store the cursor **in the same transaction** as the applied page.
- [ ] Prefer an **opaque server cursor** over `updatedSince=<client timestamp>` — client clocks skew, and equal timestamps lose rows at page boundaries. If you must use `updatedSince`, use the **server's** timestamp from the last response, and `(updatedAt, id)` tie-breaking.
- [ ] **Delta sync** only changed records; **tombstones** for deletes (`deleted` ids or `deletedAt` rows) — without them, deleted items live forever on devices.
- [ ] **Full resync fallback**: server returns `410 Gone` / `cursorExpired` (tombstones are retained only N days) → fetch full snapshot, replace local data **except** rows with pending outbox items, then resume deltas.
- [ ] Order: **push outbox first, then pull** — so the pull includes your own changes resolved by the server, and you don't overwrite pending local edits.
- [ ] **Pending local changes win over pulled data** until their outbox item resolves (skip or merge rows where `syncState != .synced`).
- [ ] Big initial sync: batch inserts (Core Data `NSBatchInsertRequest`, SwiftData save every N rows in a `ModelActor`), progress UI, resumable via cursor, don't hold one giant transaction.
- [ ] Single sync run at a time per account (actor flag); coalesce triggers.
- [ ] Freshness metadata per collection (`lastSyncedAt`) for "as of" labels and to skip syncs that just ran.

## 12. Conflict resolution

- [ ] Pick a strategy **per entity type** with product:
  - **Last-write-wins** (server timestamp) — fine for single-user settings and low-stakes data. Document that offline edits may be overwritten.
  - **Optimistic concurrency with versions/ETags**: send `If-Match: <version>` / `baseVersion`; server returns `409` / `412` with current state on mismatch.
  - **Field-level merge**: send only changed fields (PATCH with base values); server or client merges non-overlapping fields; overlapping → rule or prompt.
  - **User prompt**: for documents/notes where both versions matter — "Keep mine / Keep theirs / Duplicate". Keep the losing copy recoverable.
  - **Server-authoritative fields** (status, totals, counters, permissions, prices): never sent from the client or always overwritten by server; use operations (`increment`, `addTag`) instead of absolute values for counters/sets.
- [ ] On 409: store server version, re-apply the local intent on top (rebase) if possible, else mark the entity `conflict` and surface it.
- [ ] Clock skew: never compare device timestamps across devices; use server versions or server timestamps.
- [ ] Deleted-on-server while edited locally → product rule (usually: delete wins, inform user; or resurrect).

## 13. Background sync

- [ ] **`BGAppRefreshTask`** (short, ~30 s) for "fresh on open": pull deltas + drain small outbox. Register identifiers in `Info.plist` `BGTaskSchedulerPermittedIdentifiers`, enable Background Modes → Background fetch.
- [ ] **`BGProcessingTask`** for heavy work (full resync, media cleanup, migrations) with `requiresNetworkConnectivity = true`, `requiresExternalPower` as appropriate; Background processing mode.
- [ ] **Register all handlers before `application(_:didFinishLaunchingWithOptions:)` returns** (SwiftUI: in the `App` `init` or via `.backgroundTask(.appRefresh("id"))` scene modifier). Late registration crashes/ignores.
- [ ] **Schedule the next request inside the handler** (and on entering background); `earliestBeginDate` is a minimum, not a schedule. The system decides based on usage, battery, Low Power Mode; Background App Refresh can be disabled by the user — never rely on it for correctness.
- [ ] Set `expirationHandler` to cancel the work `Task`; call `setTaskCompleted(success:)` exactly once.
- [ ] Work must be **resumable and incremental** (cursor saved per page; outbox items independent).
- [ ] Keychain and DB file protection must allow access while locked (`AfterFirstUnlock` class).
- [ ] **Push-triggered sync** (→ **ios-push-notifications**): silent push (`content-available: 1`, `apns-push-type: background`, `apns-priority: 5`) → `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)` → sync within ~30 s → call the completion handler. Silent pushes are throttled and not delivered after force-quit; treat as a hint. Visible pushes with a Notification Service Extension can prefetch the referenced item.
- [ ] `beginBackgroundTask(withName:)` to finish an in-flight outbox send when the user backgrounds the app (a few seconds; end it in all paths).
- [ ] iOS 26+: `BGContinuedProcessingTask` for user-initiated long work with system progress UI (verify availability and API).
- [ ] Multi-account: BG handler syncs each account (round-robin within budget) — **ios-multi-account**.
- [ ] Test with `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.example.refresh"]`.

## 14. Realtime

- [ ] **`URLSessionWebSocketTask`** (or SSE via `URLSession.bytes(for:)` + line parsing) owned by one `RealtimeClient` actor per account; only while foreground (sockets die on suspension — don't fight it, use push for background).
- [ ] Receive loop: `while !Task.isCancelled { let msg = try await task.receive() }`; send pings (`sendPing`) every 20–30 s to detect dead connections; treat missing pong as disconnect.
- [ ] **Reconnect with backoff + jitter** on close/error; reset backoff after a stable connection (e.g. 60 s). Reconnect immediately on path satisfied / foreground; stop on background, sign out, or `4401`-style auth close (refresh token first).
- [ ] Auth: token in the first message or a short-lived ticket from REST; avoid long-lived tokens in the URL (logged by proxies).
- [ ] **Catch-up of missed events**: every event has a monotonic `seq`/event id; on reconnect send `lastEventId` (or run a cursor sync) so the gap is filled. Without this, anything while disconnected is lost.
- [ ] Events **write to the DB** (same apply path as sync), UI observes; dedupe by event id; ignore events older than the row's version.
- [ ] Realtime is an optimisation over sync, not a replacement: correctness must hold with the socket off.

## 15. Large transfers

- [ ] **Background `URLSession`** (`URLSessionConfiguration.background(withIdentifier:)`) for uploads/downloads that must survive suspension and termination: **delegate-based only** (no async/completion-handler convenience APIs), uploads **from a file** (`uploadTask(with:fromFile:)`), not from `Data`.
- [ ] Recreate the background session with the same identifier at launch, and implement `application(_:handleEventsForBackgroundURLSession:completionHandler:)`: store the handler, call it on the main queue from `urlSessionDidFinishEvents(forBackgroundURLSession:)`.
- [ ] Persist a task ↔ entity mapping (`taskDescription` = outbox/attachment id) because the app may relaunch with only the delegate callbacks.
- [ ] `isDiscretionary = true` for non-urgent transfers (system picks Wi-Fi/charging); `sessionSendsLaunchEvents = true`.
- [ ] Downloads: move the file out of the temporary location **inside** `didFinishDownloadingTo` (it's deleted after return). Use `resumeData` from cancellation/failure to resume.
- [ ] **Resumable/multipart uploads** for big media: chunked (S3 multipart, tus, GCS resumable) — upload parts, persist part ETags, resume after failure. iOS 17+ supports resumable uploads per the IETF draft when the server does (verify).
- [ ] Upload via **pre-signed URLs** straight to object storage; then post metadata to your API through the outbox (idempotent).
- [ ] Progress: `Progress`/`countOfBytesSent` via delegate → `@Observable` per attachment; show in the item row.
- [ ] Respect `isExpensive`/`isConstrained` for auto-uploads (setting: "Upload on cellular").
- [ ] Clean up temp files after success and on sign-out.

## 16. Security

- [ ] **TLS only**; no `NSAllowsArbitraryLoads` in ATS. Per-domain exceptions only for documented legacy hosts. Never implement a `didReceive challenge` that trusts all certs (common leftover from debugging; reject in review).
- [ ] **Certificate/public-key pinning trade-offs**: protects against compromised CAs/MITM on hostile networks, but risks bricking the app on cert rotation. If you pin: pin **SPKI hashes** of an intermediate/your own CA, ship **at least two** (current + backup), use `NSPinnedDomains` in Info.plist (iOS 14+, declarative) over hand-rolled delegate code, have a remote-config kill switch, and monitor failures. Most consumer apps: don't pin, rely on ATS + CT.
- [ ] **No secrets in the bundle**: API keys in the app are public. Use them only as identifiers; real secrets stay server-side (proxy third-party calls). Don't commit `.xcconfig` with secrets; strip debug endpoints from release builds.
- [ ] Tokens in Keychain only (→ **ios-multi-account**); not in `UserDefaults`, DB, logs, or URLs.
- [ ] **Local DB encryption**: iOS Data Protection encrypts files at rest when locked (class-dependent). For higher sensitivity: SQLCipher (GRDB/Core Data via custom store) with key in Keychain; exclude sensitive caches from backup (`isExcludedFromBackup`).
- [ ] Redact logs (`os.Logger` `privacy: .private`), no response bodies in crash reports.
- [ ] Debug-only proxy/insecure toggles behind `#if DEBUG`.
- [ ] Validate server data (lengths, URLs before opening, HTML) — don't trust payloads.

## 17. Migrations

- [ ] **DB versioning**: SwiftData `VersionedSchema` + `SchemaMigrationPlan` (lightweight + custom stages); Core Data model versions with lightweight migration (`shouldInferMappingModelAutomatically`) or staged migration (iOS 17 `NSStagedMigrationManager`). Test migration from **every shipped version** with a real old store fixture.
- [ ] Migration failure path: if the store is a pure cache, delete and resync; if it contains **unsynced outbox items**, never delete — export/migrate them first.
- [ ] **Outbox migration**: payloads are versioned (`payloadVersion`); the processor can still send (or upgrade) old payloads after an app update. Don't rename enum cases stored in the outbox without a migration.
- [ ] **API version compatibility**: old app versions stay in the wild for years — server must support N-2 app versions or force update. Client: tolerant decoding (unknown enum cases → `.unknown`, optional new fields, ignore unknown keys); send `App-Version` header; handle `426 Upgrade Required` / a min-version config with a force-update screen.
- [ ] Cursor/sync format changes → server returns `410` → full resync (section 11).
- [ ] Run migrations off the main thread with a launch UI if they can be slow; BG tasks must not open the store before migration completed.

## 18. GraphQL (short)

- [ ] **Apollo iOS** (Swift codegen from `.graphql` operations + schema) with a **normalized cache** (`InMemoryNormalizedCache` or `SQLiteNormalizedCache` for persistence). Objects are keyed by `__typename` + `id` — configure cache key resolution (`SchemaConfiguration.cacheKeyInfo(for:object:)`) so updates in one query refresh every screen showing that object.
- [ ] Cache policies: `.returnCacheDataAndFetch` ≈ stale-while-revalidate; `.returnCacheDataElseFetch`; `.fetchIgnoringCacheData` for pull-to-refresh. Use `watch(query:)` so the UI observes the cache (same "observe local" principle as section 9). Verify names against your Apollo iOS version (1.x vs 2.x changed APIs to async/await and `Sendable`).
- [ ] Auth/retry via Apollo's interceptor chain (`InterceptorProvider`); reuse your `TokenProvider` single-flight; GraphQL errors arrive in a `200` body — map `errors[].extensions.code` to your `APIError`.
- [ ] **Optimistic responses**: write the expected result to the cache in a local transaction (`store.withinReadWriteTransaction`) before the mutation, roll back on error. For durable offline writes, still use **your outbox** — Apollo doesn't persist queued mutations.
- [ ] Persisted queries (APQ) to cut payload and allow GET caching; pagination with cursor connections and cache merge.
- [ ] Don't mix: if Apollo's normalized cache is the source of truth for a feature, don't also mirror it into SwiftData unless you need queries Apollo can't do.

## 19. Testing plan

1. **Unit tests** with a `URLProtocol` stub (register via `URLSessionConfiguration.protocolClasses`) — assert interceptor order, headers, idempotency key stability across retries, retry/backoff decisions (inject clock + random), Retry-After parsing, error mapping for every status/URLError, single-flight refresh under 50 concurrent 401s, GET dedupe, cancellation not reported as error.
2. **Repository/outbox/sync tests** against an in-memory store (`ModelConfiguration(isStoredInMemoryOnly: true)` / Core Data `/dev/null` store): transactional enqueue, ordering, coalescing, permanent vs transient failure, crash recovery (`inFlight` reset), cursor saved with page, tombstones, 410 full resync preserving pending rows, conflicts.
3. **Mock server** (local HTTP server, WireMock/Prism from OpenAPI, or a scriptable stub) for integration and UI tests via a launch argument selecting the environment.
4. **Network conditions**: Network Link Conditioner (Xcode Additional Tools / device Developer settings: 3G, Edge, 100% loss, high latency), Charles/Proxyman for throttling, rewriting, breakpoints, and injecting 5xx/429.
5. **Airplane-mode matrix** on a real device (simulator networking follows the Mac).
6. **Conflicts**: two simulators/devices editing the same record; server forced to return 409.
7. **Clock skew**: change device time ±1 day; token expiry, `updatedSince`, cache freshness must still work.
8. **Background**: simulate BG tasks in debugger; silent push via `xcrun simctl push`; background URLSession with the app killed (kill from Xcode, not swipe — force-quit cancels background transfers).
9. **Migrations**: launch new build over fixtures of each old store version with pending outbox items.
10. **Performance**: initial sync of max-volume account; main-thread checker / hangs instrument while syncing.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| N1 | Cold start offline with cached data | Cached data shown instantly; offline banner; no error alert |
| N2 | Cold start offline, nothing cached | Empty state with "Connect to load" + retry |
| N3 | Request times out | Retried with backoff (if idempotent); then non-blocking error over cached data |
| N4 | Server 503 with `Retry-After: 10` | Next attempt ≥ 10 s later |
| N5 | 10 parallel requests, token expired | One refresh; all retried once; no logout |
| N6 | Refresh while offline | Tokens kept; offline state; no logout |
| N7 | Refresh token revoked | Re-auth flow; data and outbox kept |
| N8 | Navigate away mid-load | Request cancelled; no error shown; no late UI update |
| N9 | Type fast in search | Only last query's results shown; earlier requests cancelled |
| N10 | Two screens request same GET | One network call |
| N11 | Resource unchanged (ETag) | 304/cached; no DB churn; UI doesn't flicker |
| N12 | Create item offline | Appears immediately as pending; synced on reconnect; no duplicate |
| N13 | Create, edit, delete item offline | Coalesced; nothing (or one delete) sent |
| N14 | App killed after server committed, before response | Replay deduped by idempotency key; one item on server |
| N15 | Outbox item gets 422 | Item marked failed with reason; other items continue |
| N16 | Outbox item gets 401 | Outbox paused; resumes after reauth |
| N17 | Edit same record on two devices | Strategy per entity (LWW / 409 → merge / prompt); no silent loss where not allowed |
| N18 | Record deleted on server | Tombstone removes it locally on next sync |
| N19 | Cursor expired (410) | Full resync; pending local changes preserved |
| N20 | Network flaps during sync | Resumes from last saved cursor; no duplicated rows |
| N21 | Wi-Fi with captive portal | Requests fail (HTML/redirect) → mapped error, not decoding crash; banner |
| N22 | Low Data Mode on | Prefetch/media autoload skipped; user actions work |
| N23 | Background app refresh fires | Delta pulled + outbox drained within budget; task completed |
| N24 | Silent push "item changed" | Item synced; completion handler called |
| N25 | Large upload, app backgrounded and killed by system | Background session completes; entity updated on relaunch |
| N26 | Upload interrupted | Resumes from last part / resumeData |
| N27 | WebSocket drops for 2 min | Reconnect with backoff; missed events caught up via lastEventId |
| N28 | Device clock set +1 day | Sync/token logic unaffected |
| N29 | App update with DB migration and pending outbox | Migration succeeds; queued items still sent |
| N30 | Server adds unknown enum value | Decoded as `.unknown`; no crash |
| N31 | Force-update required (426) | Force-update screen |
| N32 | Account switch with outbox pending | Pending items stay with their account; sent with that account's token |
| N33 | Sign out with unsent changes | Warning; on confirm, outbox + DB + caches wiped |

## 20. Common bugs to look for in review

- `URLSession.shared` / `URLCache.shared` for authenticated calls → cross-account cache, no timeouts config
- New `URLSession` per request (connection pool and TLS handshakes per call; delegate retain leaks)
- Auth header baked into `httpAdditionalHeaders` → stale token after refresh
- Non-single-flight refresh → rotating refresh token race → random logouts
- Infinite 401 → refresh → 401 loop
- Signing out when refresh fails due to no network
- Retrying POST without idempotency key → duplicate orders/messages
- Idempotency key generated per attempt instead of per operation
- Retrying 4xx or decoding errors; retrying cancellation
- No jitter → thundering herd after outage
- `CancellationError` / `URLError.cancelled` shown as an alert
- Unstructured `Task {}` in `onAppear` never cancelled → late writes into a dismissed screen
- Blocking requests because `NWPathMonitor` said offline
- UI rendering network responses directly → offline shows blank, online/offline look different
- Optimistic change and outbox row written in separate transactions
- Outbox stuck because one permanently failing item blocks the queue
- Stale `inFlight` items never reset after a crash
- Pull overwriting pending local edits; pulling before pushing
- `updatedSince` with device clock / no tie-breaker → missed rows
- No tombstones → deleted items resurrect
- Cursor saved before page applied (or in a separate transaction) → gaps
- Passing SwiftData `@Model` objects across actors / to the network layer
- Heavy sync on a main-actor `ModelContext` → hangs
- DB/Keychain with `.complete` protection → BG sync fails while locked
- `BGTaskScheduler.register` after launch finished; next task never rescheduled; `setTaskCompleted` not called
- Relying on silent push / BG refresh for correctness
- Background URLSession with completion handlers or upload from `Data` → crash/unsupported
- Not moving the downloaded file before `didFinishDownloadingTo` returns
- WebSocket without ping, without reconnect backoff, or without catch-up
- `didReceive challenge` trusting all certificates; `NSAllowsArbitraryLoads`
- Pinning a single leaf certificate with no backup pin
- Secrets (API keys with write scope) in the bundle
- Migration failure → store deleted with unsynced changes
- Strict `Codable` enums crash/fail on new server values
- `.iso8601` date decoding failing on fractional seconds

## 21. Output format

When asked to **review/audit**:
1. **Summary** — spec from section 1 (API style, offline mode per feature, conflict policy, volume), architecture (client per account? DB as source of truth? outbox?), min OS
2. **Checklist results** — sections 2–18 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** — data loss (lost/duplicated writes, overwritten edits, migration deletes), unwanted logouts, cross-account leaks, security (trust-all, secrets) — with fix code
4. **Testing gaps** — N1–N33 not covered

When asked to **implement**: fill the spec, start from the Appendix, adapt `Endpoint`, DTOs, the change-feed contract and conflict strategy; delete what you don't need (outbox for read-only apps, realtime, background transfers); list Info.plist/capabilities (Background Modes: fetch, processing, remote-notification; `BGTaskSchedulerPermittedIdentifiers`; ATS/`NSPinnedDomains`) and **server contract questions** (error envelope, `Idempotency-Key` support, change feed with tombstones and cursor expiry, 409 body, `Retry-After`, realtime event ids).

When asked to **debug**: reproduce with Proxyman/Charles + Network Link Conditioner, then check (1) error classification and retry decision, (2) token refresh path, (3) whether UI reads DB or network, (4) outbox state rows and transaction boundaries, (5) sync cursor/tombstones, (6) BG task registration and data protection — in that order.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI + SwiftData)

Simplified but complete enough to adapt. `Note` stands in for your synced entity; `/notes`, `/changes` for your endpoints. Verify signatures against your SDK.

### A.1 Environment, endpoint, typed errors

```swift
import Foundation

struct APIEnvironment: Sendable {
    let baseURL: URL
    static let prodEU = APIEnvironment(baseURL: URL(string: "https://eu.api.example.com/v1/")!)
    static let prodUS = APIEnvironment(baseURL: URL(string: "https://us.api.example.com/v1/")!)
}

enum HTTPMethod: String, Sendable { case get = "GET", post = "POST", put = "PUT", patch = "PATCH", delete = "DELETE" }

struct Endpoint<Response: Decodable & Sendable>: Sendable {
    var method: HTTPMethod = .get
    var path: String
    var query: [URLQueryItem] = []
    var body: Data? = nil
    var headers: [String: String] = [:]
    var idempotencyKey: String? = nil          // set by caller for POST/PATCH (outbox)
    var requiresAuth = true
    var isIdempotent: Bool { method != .post && method != .patch || idempotencyKey != nil }
}

struct EmptyResponse: Decodable, Sendable {}

struct ServerErrorBody: Decodable, Sendable { let code: String?; let message: String? }

enum APIError: Error, Sendable, Equatable {
    case offline, timeout, cancelled, tls
    case unauthorized, forbidden, notFound
    case conflict(serverBody: Data?)
    case rateLimited(retryAfter: TimeInterval?)
    case server(status: Int, retryAfter: TimeInterval?)
    case http(status: Int, code: String?, message: String?)
    case decoding(endpoint: String, detail: String)
    case unknown(String)

    var isTransient: Bool {
        switch self {
        case .offline, .timeout, .rateLimited: true
        case .server(let s, _): [502, 503, 504, 500].contains(s)
        case .http(let s, _, _): s == 408
        default: false
        }
    }

    var userMessage: String {
        switch self {
        case .offline: String(localized: "You're offline. We'll sync when you're back online.")
        case .timeout: String(localized: "The connection is slow. Please try again.")
        case .unauthorized: String(localized: "Please sign in again.")
        case .forbidden: String(localized: "You don't have access to this.")
        case .notFound: String(localized: "This item is no longer available.")
        case .rateLimited: String(localized: "Too many requests. Please wait a moment.")
        case .http(_, _, let message?): message      // only if server guarantees user-safe text
        default: String(localized: "Something went wrong. Please try again.")
        }
    }

    static func from(_ error: Error) -> APIError {
        if let e = error as? APIError { return e }
        if error is CancellationError { return .cancelled }
        guard let u = error as? URLError else { return .unknown(String(describing: error)) }
        switch u.code {
        case .cancelled: return .cancelled
        case .timedOut: return .timeout
        case .notConnectedToInternet, .networkConnectionLost, .dataNotAllowed,
             .internationalRoamingOff, .cannotFindHost, .cannotConnectToHost, .dnsLookupFailed: return .offline
        case .serverCertificateUntrusted, .serverCertificateHasBadDate, .secureConnectionFailed,
             .serverCertificateNotYetValid, .clientCertificateRejected: return .tls
        default: return .unknown(u.code.rawValue.description)
        }
    }

    static func from(status: Int, data: Data, headers: [AnyHashable: Any]) -> APIError {
        let body = try? JSONDecoder().decode(ServerErrorBody.self, from: data)
        let retryAfter = RetryPolicy.parseRetryAfter(headers["Retry-After"] as? String)
        switch status {
        case 401: return .unauthorized
        case 403: return .forbidden
        case 404, 410 where body?.code != "cursor_expired": return .notFound
        case 409, 412: return .conflict(serverBody: data)
        case 429: return .rateLimited(retryAfter: retryAfter)
        case 500...599: return .server(status: status, retryAfter: retryAfter)
        default: return .http(status: status, code: body?.code, message: body?.message)
        }
    }
}
```

### A.2 Retry policy and single-flight token provider

```swift
struct RetryPolicy: Sendable {
    var maxAttempts = 3
    var base: TimeInterval = 0.5
    var cap: TimeInterval = 30

    func delay(forAttempt attempt: Int, error: APIError) -> TimeInterval {
        switch error {
        case .rateLimited(let ra?), .server(_, let ra?): return min(ra, 300)
        default:
            let exp = min(cap, base * pow(2, Double(attempt)))
            return Double.random(in: 0...exp)                // full jitter
        }
    }

    static func parseRetryAfter(_ value: String?) -> TimeInterval? {
        guard let value else { return nil }
        if let secs = TimeInterval(value) { return secs }
        let f = DateFormatter(); f.locale = Locale(identifier: "en_US_POSIX")
        f.dateFormat = "EEE, dd MMM yyyy HH:mm:ss zzz"
        return f.date(from: value).map { max(0, $0.timeIntervalSinceNow) }
    }
}

struct Tokens: Codable, Sendable { var access: String; var refresh: String; var expiresAt: Date }

protocol TokenStore: Sendable {
    func load() async throws -> Tokens?
    func save(_ tokens: Tokens) async throws        // Keychain, AfterFirstUnlockThisDeviceOnly
}

actor TokenProvider {
    private let store: TokenStore
    private let refreshCall: @Sendable (String) async throws -> Tokens
    private var cached: Tokens?
    private var refreshTask: Task<Tokens, Error>?

    init(store: TokenStore, refresh: @escaping @Sendable (String) async throws -> Tokens) {
        self.store = store; self.refreshCall = refresh
    }

    func validAccessToken() async throws -> String {
        if cached == nil { cached = try await store.load() }
        guard let t = cached else { throw APIError.unauthorized }
        if t.expiresAt.timeIntervalSinceNow > 60 { return t.access }
        return try await refresh(failedToken: t.access)
    }

    /// Single-flight: concurrent callers share one refresh.
    func refresh(failedToken: String) async throws -> String {
        if let current = cached, current.access != failedToken,
           current.expiresAt.timeIntervalSinceNow > 60 { return current.access }  // already refreshed
        if let task = refreshTask { return try await task.value.access }
        guard let refreshToken = cached?.refresh else { throw APIError.unauthorized }
        let task = Task { [refreshCall, store] in
            let new = try await refreshCall(refreshToken)
            try await store.save(new)                 // persist BEFORE use (rotation)
            return new
        }
        refreshTask = task
        defer { refreshTask = nil }
        do {
            let new = try await task.value
            cached = new
            return new.access
        } catch let e as APIError where e == .offline || e == .timeout {
            throw e                                   // keep tokens; never sign out offline
        }
    }
}
```

### A.3 API client with interceptors, retries, dedupe

```swift
import os

actor APIClient {
    private let env: APIEnvironment
    private let session: URLSession
    private let tokens: TokenProvider
    private let retry: RetryPolicy
    private let decoder: JSONDecoder
    private var inFlightGETs: [String: Task<(Data, HTTPURLResponse), Error>] = [:]
    private let log = Logger(subsystem: "com.example.app", category: "api")

    init(env: APIEnvironment, tokens: TokenProvider, cacheDirectory: URL, retry: RetryPolicy = .init()) {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 20
        config.timeoutIntervalForResource = 90
        config.waitsForConnectivity = false
        config.httpCookieStorage = nil
        config.urlCache = URLCache(memoryCapacity: 10 << 20, diskCapacity: 100 << 20, directory: cacheDirectory)
        config.requestCachePolicy = .useProtocolCachePolicy
        config.httpAdditionalHeaders = [
            "Accept": "application/json",
            "App-Version": Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString") as? String ?? "0",
        ]
        self.session = URLSession(configuration: config)
        self.env = env; self.tokens = tokens; self.retry = retry
        let d = JSONDecoder()
        d.dateDecodingStrategy = .custom { dec in
            let s = try dec.singleValueContainer().decode(String.self)
            let f = Date.ISO8601FormatStyle(includingFractionalSeconds: true)
            if let date = try? f.parse(s) ?? Date.ISO8601FormatStyle().parse(s) { return date }
            throw DecodingError.dataCorrupted(.init(codingPath: dec.codingPath, debugDescription: "bad date \(s)"))
        }
        self.decoder = d
    }

    func send<R>(_ ep: Endpoint<R>) async throws -> R {
        let (data, _) = try await sendRaw(ep)
        if R.self == EmptyResponse.self { return EmptyResponse() as! R }
        do { return try decoder.decode(R.self, from: data) }
        catch { throw APIError.decoding(endpoint: ep.path, detail: String(describing: error)) }
    }

    func sendRaw<R>(_ ep: Endpoint<R>) async throws -> (Data, HTTPURLResponse) {
        let requestId = UUID().uuidString                           // stable across retries
        guard ep.method == .get else { return try await withRetry(ep, requestId: requestId) }
        let key = try makeURL(ep).absoluteString
        if let existing = inFlightGETs[key] { return try await existing.value }
        // Unstructured so one caller's cancellation doesn't cancel the others.
        let task = Task { try await self.withRetry(ep, requestId: requestId) }
        inFlightGETs[key] = task
        defer { inFlightGETs[key] = nil }
        return try await withTaskCancellationHandler {
            try await task.value
        } onCancel: { }                                             // caller stops awaiting; task finishes for others
    }

    private func withRetry<R>(_ ep: Endpoint<R>, requestId: String) async throws -> (Data, HTTPURLResponse) {
        var attempt = 0
        while true {
            do { return try await sendOnceWithAuth(ep, requestId: requestId) }
            catch {
                let e = APIError.from(error)
                attempt += 1
                guard e.isTransient, ep.isIdempotent, attempt < retry.maxAttempts else { throw e }
                let d = retry.delay(forAttempt: attempt, error: e)
                log.info("retry \(attempt) \(ep.path, privacy: .public) in \(d)s")
                try await Task.sleep(for: .seconds(d))
            }
        }
    }

    private func sendOnceWithAuth<R>(_ ep: Endpoint<R>, requestId: String) async throws -> (Data, HTTPURLResponse) {
        var token = ep.requiresAuth ? try await tokens.validAccessToken() : nil
        for authAttempt in 0..<2 {
            var req = try buildRequest(ep, requestId: requestId, token: token)
            req.timeoutInterval = 20
            let (data, resp): (Data, URLResponse)
            do { (data, resp) = try await session.data(for: req) }
            catch { throw APIError.from(error) }
            guard let http = resp as? HTTPURLResponse else { throw APIError.unknown("non-HTTP") }
            if http.statusCode == 401, ep.requiresAuth, authAttempt == 0, let used = token {
                token = try await tokens.refresh(failedToken: used)  // single-flight, then retry once
                continue
            }
            guard (200..<300).contains(http.statusCode) else {
                throw APIError.from(status: http.statusCode, data: data, headers: http.allHeaderFields)
            }
            return (data, http)
        }
        throw APIError.unauthorized
    }

    private func makeURL<R>(_ ep: Endpoint<R>) throws -> URL {
        var c = URLComponents(url: env.baseURL.appending(path: ep.path), resolvingAgainstBaseURL: false)!
        if !ep.query.isEmpty { c.queryItems = ep.query }
        guard let url = c.url else { throw APIError.unknown("bad url") }
        return url
    }

    private func buildRequest<R>(_ ep: Endpoint<R>, requestId: String, token: String?) throws -> URLRequest {
        // Interceptor order: request-id → auth → idempotency → (retry wraps) → logging
        var req = URLRequest(url: try makeURL(ep))
        req.httpMethod = ep.method.rawValue
        req.setValue(requestId, forHTTPHeaderField: "X-Request-Id")
        if let token { req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization") }
        if let key = ep.idempotencyKey { req.setValue(key, forHTTPHeaderField: "Idempotency-Key") }
        if let body = ep.body {
            req.httpBody = body
            req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        }
        ep.headers.forEach { req.setValue($1, forHTTPHeaderField: $0) }
        log.debug("\(ep.method.rawValue) \(ep.path, privacy: .public) id=\(requestId, privacy: .public)")
        return req
    }
}
```

### A.4 Connectivity monitor

```swift
import Network
import Observation

@MainActor @Observable
final class Connectivity {
    private(set) var isOnline = true
    private(set) var isConstrained = false
    private(set) var recentOfflineFailures = 0
    var showOfflineBanner: Bool { !isOnline || recentOfflineFailures >= 2 }
    var onReconnect: (@MainActor () -> Void)?

    private let monitor = NWPathMonitor()

    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            Task { @MainActor in self?.update(path) }
        }
        monitor.start(queue: DispatchQueue(label: "net.path"))
    }

    private func update(_ path: NWPath) {
        let wasOnline = isOnline
        isOnline = path.status == .satisfied        // a hint, not truth
        isConstrained = path.isConstrained
        if !wasOnline && isOnline { recentOfflineFailures = 0; onReconnect?() }
    }

    func record(_ result: Result<Void, APIError>) {
        switch result {
        case .success: recentOfflineFailures = 0
        case .failure(.offline), .failure(.timeout): recentOfflineFailures += 1
        case .failure: break
        }
    }
}
```

### A.5 SwiftData models: entity, outbox, sync state

```swift
import SwiftData

enum SyncState: String, Codable { case synced, pendingCreate, pendingUpdate, pendingDelete, failed, conflict }

@Model final class Note {
    #Unique<Note>([\.id])                         // iOS 18; on 17 dedupe manually
    var id: UUID                                  // client-generated, accepted by server
    var title: String
    var body: String
    var updatedAt: Date                           // server time
    var version: Int                              // server version for If-Match
    var syncStateRaw: String
    var deletedLocally: Bool
    var syncState: SyncState {
        get { SyncState(rawValue: syncStateRaw) ?? .synced }
        set { syncStateRaw = newValue.rawValue }
    }
    init(id: UUID = UUID(), title: String, body: String, updatedAt: Date = .now, version: Int = 0,
         syncState: SyncState = .synced) {
        self.id = id; self.title = title; self.body = body; self.updatedAt = updatedAt
        self.version = version; self.syncStateRaw = syncState.rawValue; self.deletedLocally = false
    }
}

@Model final class OutboxItem {
    var seq: Int                                  // FIFO order
    var entityId: UUID
    var operation: String                         // create | update | delete
    var payload: Data
    var payloadVersion: Int
    var idempotencyKey: String                    // generated ONCE
    var attempts: Int
    var nextAttemptAt: Date
    var state: String                             // pending | inFlight | failed
    var lastError: String?
    init(seq: Int, entityId: UUID, operation: String, payload: Data) {
        self.seq = seq; self.entityId = entityId; self.operation = operation; self.payload = payload
        self.payloadVersion = 1; self.idempotencyKey = UUID().uuidString
        self.attempts = 0; self.nextAttemptAt = .now; self.state = "pending"
    }
}

@Model final class SyncMeta {
    #Unique<SyncMeta>([\.collection])
    var collection: String
    var cursor: String?
    var lastSyncedAt: Date?
    init(collection: String) { self.collection = collection }
}

struct NoteDTO: Codable, Sendable { let id: UUID; let title: String; let body: String; let updatedAt: Date; let version: Int }
struct ChangesPage: Decodable, Sendable { let changes: [NoteDTO]; let deleted: [UUID]; let nextCursor: String; let hasMore: Bool }
```

### A.6 Repository (write local + enqueue in one transaction) and UI

```swift
@ModelActor
actor NoteStore {
    func nextSeq() throws -> Int {
        var d = FetchDescriptor<OutboxItem>(sortBy: [SortDescriptor(\.seq, order: .reverse)]); d.fetchLimit = 1
        return (try modelContext.fetch(d).first?.seq ?? 0) + 1
    }

    /// Local change + outbox row, one save = one transaction.
    func upsertLocal(id: UUID?, title: String, body: String) throws -> UUID {
        let existing = try id.flatMap { id in try modelContext.fetch(FetchDescriptor<Note>(predicate: #Predicate { $0.id == id })).first }
        let note = existing ?? Note(title: title, body: body, syncState: .pendingCreate)
        if existing == nil { modelContext.insert(note) }
        note.title = title; note.body = body
        if note.syncState == .synced { note.syncState = .pendingUpdate }
        let payload = try JSONEncoder().encode(["title": title, "body": body, "baseVersion": String(note.version)])
        modelContext.insert(OutboxItem(seq: try nextSeq(), entityId: note.id,
                                       operation: existing == nil ? "create" : "update", payload: payload))
        try modelContext.save()
        return note.id
    }

    func deleteLocal(id: UUID) throws {
        guard let note = try modelContext.fetch(FetchDescriptor<Note>(predicate: #Predicate { $0.id == id })).first else { return }
        if note.syncState == .pendingCreate {                 // never reached server: coalesce away
            let nid = note.id
            try modelContext.delete(model: OutboxItem.self, where: #Predicate { $0.entityId == nid })
            modelContext.delete(note)
        } else {
            note.deletedLocally = true; note.syncState = .pendingDelete
            modelContext.insert(OutboxItem(seq: try nextSeq(), entityId: id, operation: "delete", payload: Data()))
        }
        try modelContext.save()
    }
}

struct NotesView: View {
    @Query(filter: #Predicate<Note> { !$0.deletedLocally }, sort: \Note.updatedAt, order: .reverse)
    private var notes: [Note]                              // UI observes the DB only
    @Environment(SyncEngine.self) private var sync
    @Environment(Connectivity.self) private var net

    var body: some View {
        List(notes) { note in
            HStack {
                Text(note.title)
                Spacer()
                if note.syncState != .synced { Image(systemName: note.syncState == .failed ? "exclamationmark.circle" : "clock") }
            }
        }
        .overlay(alignment: .top) { if net.showOfflineBanner { OfflineBanner() } }
        .refreshable { await sync.syncNow(reason: "pull") }
        .task { await sync.syncNow(reason: "appear") }    // cancelled on disappear
    }
}

struct OfflineBanner: View {
    var body: some View { Text("You're offline").font(.footnote).padding(6).background(.thinMaterial, in: Capsule()) }
}
```

### A.7 Outbox processor and sync engine (push, then pull with cursor + tombstones)

```swift
@ModelActor
actor SyncWorker {
    private var api: APIClient!
    func configure(api: APIClient) { self.api = api }

    func resetStaleInFlight() throws {
        for item in try modelContext.fetch(FetchDescriptor<OutboxItem>(predicate: #Predicate { $0.state == "inFlight" })) {
            item.state = "pending"
        }
        try modelContext.save()
    }

    /// Ordered replay; permanent failures don't block unrelated items.
    func drainOutbox() async throws {
        let now = Date.now
        let items = try modelContext.fetch(FetchDescriptor<OutboxItem>(
            predicate: #Predicate { $0.state == "pending" && $0.nextAttemptAt <= now },
            sortBy: [SortDescriptor(\.seq)]))
        var blockedEntities = Set<UUID>()
        for item in items {
            try Task.checkCancellation()
            if blockedEntities.contains(item.entityId) { continue }      // keep per-entity order
            item.state = "inFlight"; try modelContext.save()
            do {
                let dto = try await api.send(endpoint(for: item))
                try apply(server: dto, for: item)                         // DB + delete row, one save
            } catch let e as APIError {
                switch e {
                case .unauthorized:
                    item.state = "pending"; try modelContext.save(); throw e  // pause whole outbox
                case .offline:
                    item.state = "pending"; try modelContext.save(); return
                case .conflict:
                    markEntity(item.entityId, .conflict)
                    item.state = "failed"; item.lastError = "conflict"
                    blockedEntities.insert(item.entityId)
                case _ where e.isTransient:
                    item.attempts += 1; item.state = "pending"
                    let backoff = min(900, pow(2, Double(item.attempts)) * 5)
                    item.nextAttemptAt = .now + .random(in: 0...backoff)
                    blockedEntities.insert(item.entityId)
                default:                                                   // permanent: 4xx validation
                    item.state = "failed"; item.lastError = e.userMessage
                    markEntity(item.entityId, .failed)
                    blockedEntities.insert(item.entityId)
                }
                try modelContext.save()
            }
        }
    }

    private func endpoint(for item: OutboxItem) -> Endpoint<NoteDTO?> {
        switch item.operation {
        case "create": Endpoint(method: .put, path: "notes/\(item.entityId)", body: item.payload, idempotencyKey: item.idempotencyKey)
        case "update": Endpoint(method: .patch, path: "notes/\(item.entityId)", body: item.payload, idempotencyKey: item.idempotencyKey)
        default:       Endpoint(method: .delete, path: "notes/\(item.entityId)", idempotencyKey: item.idempotencyKey)
        }
    }

    private func apply(server dto: NoteDTO?, for item: OutboxItem) throws {
        let id = item.entityId
        let note = try modelContext.fetch(FetchDescriptor<Note>(predicate: #Predicate { $0.id == id })).first
        let others = try modelContext.fetchCount(FetchDescriptor<OutboxItem>(
            predicate: #Predicate { $0.entityId == id && $0.state != "inFlight" }))
        if item.operation == "delete" { if let note { modelContext.delete(note) } }
        else if let dto, let note {
            note.version = dto.version; note.updatedAt = dto.updatedAt
            if others == 0 { note.title = dto.title; note.body = dto.body; note.syncState = .synced }
        }
        modelContext.delete(item)
        try modelContext.save()
    }

    private func markEntity(_ id: UUID, _ s: SyncState) {
        if let n = try? modelContext.fetch(FetchDescriptor<Note>(predicate: #Predicate { $0.id == id })).first { n.syncState = s }
    }

    /// Pull deltas; cursor stored in the same save as the page.
    func pull() async throws {
        let meta = try modelContext.fetch(FetchDescriptor<SyncMeta>(predicate: #Predicate { $0.collection == "notes" })).first
            ?? { let m = SyncMeta(collection: "notes"); modelContext.insert(m); return m }()
        var hasMore = true
        while hasMore {
            try Task.checkCancellation()
            var q = [URLQueryItem(name: "limit", value: "500")]
            if let c = meta.cursor { q.append(URLQueryItem(name: "cursor", value: c)) }
            let page: ChangesPage
            do { page = try await api.send(Endpoint<ChangesPage>(path: "changes", query: q)) }
            catch APIError.http(410, _, _) { meta.cursor = nil; try modelContext.save(); continue }  // full resync
            for dto in page.changes {
                let id = dto.id
                if let local = try modelContext.fetch(FetchDescriptor<Note>(predicate: #Predicate { $0.id == id })).first {
                    guard local.syncState == .synced, dto.version >= local.version else { continue } // pending local wins
                    local.title = dto.title; local.body = dto.body; local.updatedAt = dto.updatedAt; local.version = dto.version
                } else {
                    modelContext.insert(Note(id: dto.id, title: dto.title, body: dto.body,
                                             updatedAt: dto.updatedAt, version: dto.version))
                }
            }
            for id in page.deleted {                                       // tombstones
                try modelContext.delete(model: Note.self, where: #Predicate { $0.id == id && $0.syncStateRaw == "synced" })
            }
            meta.cursor = page.nextCursor; meta.lastSyncedAt = .now; hasMore = page.hasMore
            try modelContext.save()
        }
    }
}

@MainActor @Observable
final class SyncEngine {
    private(set) var isSyncing = false
    private(set) var lastError: APIError?
    private let worker: SyncWorker
    private let connectivity: Connectivity
    private var current: Task<Void, Never>?

    init(container: ModelContainer, api: APIClient, connectivity: Connectivity) {
        worker = SyncWorker(modelContainer: container)
        self.connectivity = connectivity
        Task { [worker] in await worker.configure(api: api); try? await worker.resetStaleInFlight() }
        connectivity.onReconnect = { [weak self] in
            Task { try? await Task.sleep(for: .seconds(.random(in: 0...3))); await self?.syncNow(reason: "reconnect") }
        }
    }

    /// Coalesces concurrent triggers into one run: push outbox, then pull.
    func syncNow(reason: String) async {
        if let current { return await current.value }
        let task = Task { [worker, connectivity] in
            do {
                try await worker.drainOutbox()
                try await worker.pull()
                connectivity.record(.success(()))
                self.lastError = nil
            } catch {
                let e = APIError.from(error)
                if e != .cancelled { self.lastError = e; connectivity.record(.failure(e)) }
            }
        }
        current = task; isSyncing = true
        await task.value
        current = nil; isSyncing = false
    }
}
```

### A.8 Background sync registration and silent push

```swift
import BackgroundTasks
import UIKit

enum BGIds { static let refresh = "com.example.app.refresh"; static let processing = "com.example.app.fullsync" }
// Info.plist: BGTaskSchedulerPermittedIdentifiers = [refresh, processing]
// Capabilities: Background Modes → Background fetch, Background processing, Remote notifications

final class AppDelegate: NSObject, UIApplicationDelegate {
    var makeSyncEngine: (@MainActor () -> SyncEngine?)!      // resolves active account(s)' engine
    var backgroundSessionCompletion: (() -> Void)?

    func application(_ app: UIApplication, didFinishLaunchingWithOptions _: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Register BEFORE launch finishes.
        BGTaskScheduler.shared.register(forTaskWithIdentifier: BGIds.refresh, using: nil) { task in
            self.handle(task as! BGAppRefreshTask)
        }
        BGTaskScheduler.shared.register(forTaskWithIdentifier: BGIds.processing, using: nil) { task in
            self.handle(task as! BGProcessingTask)
        }
        return true
    }

    static func scheduleRefresh() {
        let req = BGAppRefreshTaskRequest(identifier: BGIds.refresh)
        req.earliestBeginDate = .now + 15 * 60                  // minimum, not a schedule
        try? BGTaskScheduler.shared.submit(req)
    }

    static func scheduleFullSync() {
        let req = BGProcessingTaskRequest(identifier: BGIds.processing)
        req.requiresNetworkConnectivity = true
        req.requiresExternalPower = false
        try? BGTaskScheduler.shared.submit(req)
    }

    private func handle(_ task: BGTask) {
        Self.scheduleRefresh()                                   // always reschedule first
        let work = Task { @MainActor in
            await self.makeSyncEngine()?.syncNow(reason: "bg")
            task.setTaskCompleted(success: !Task.isCancelled)
        }
        task.expirationHandler = { work.cancel() }               // cancellation → setTaskCompleted(false)
    }

    // Silent push: content-available=1, apns-push-type=background, apns-priority=5
    func application(_ app: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable: Any]) async
        -> UIBackgroundFetchResult {
        guard userInfo["sync"] != nil, let engine = await makeSyncEngine() else { return .noData }
        await engine.syncNow(reason: "push")
        return await engine.lastError == nil ? .newData : .failed
    }

    // Background URLSession relaunch (uploads/downloads)
    func application(_ app: UIApplication, handleEventsForBackgroundURLSession identifier: String,
                     completionHandler: @escaping () -> Void) {
        backgroundSessionCompletion = completionHandler
        _ = TransferManager.shared                                // recreate session with same identifier
    }
}

// SwiftUI: schedule when going to background
// .onChange(of: scenePhase) { _, p in if p == .background { AppDelegate.scheduleRefresh() } }

final class TransferManager: NSObject, URLSessionDataDelegate, @unchecked Sendable {
    static let shared = TransferManager()
    private(set) lazy var session: URLSession = {
        let c = URLSessionConfiguration.background(withIdentifier: "com.example.app.transfers")
        c.sessionSendsLaunchEvents = true
        c.isDiscretionary = false
        return URLSession(configuration: c, delegate: self, delegateQueue: nil)
    }()

    /// Upload from a FILE; taskDescription maps back to the attachment after relaunch.
    func upload(file: URL, to presignedURL: URL, attachmentId: UUID) {
        var req = URLRequest(url: presignedURL); req.httpMethod = "PUT"
        let task = session.uploadTask(with: req, fromFile: file)
        task.taskDescription = attachmentId.uuidString
        task.resume()
    }

    func urlSession(_ s: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        guard let id = task.taskDescription.flatMap(UUID.init) else { return }
        // Persist result for attachment `id`; on success enqueue the metadata POST in the outbox.
        _ = (id, error)
    }

    func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession) {
        DispatchQueue.main.async {
            (UIApplication.shared.delegate as? AppDelegate)?.backgroundSessionCompletion?()
            (UIApplication.shared.delegate as? AppDelegate)?.backgroundSessionCompletion = nil
        }
    }
}
```
