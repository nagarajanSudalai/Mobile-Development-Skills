---
name: android-network-offline-sync
description: Use when implementing, reviewing, testing, or debugging networking, caching, offline-first data, and sync in a native Android app — single configured OkHttp client with Retrofit or Ktor, kotlinx.serialization, interceptor order, base URL per environment/region, auth header + single-flight token refresh via Authenticator, typed error model, retries with exponential backoff + jitter and idempotency keys, Retry-After and circuit breaking, coroutine cancellation and in-flight GET dedupe, OkHttp HTTP cache with ETag / stale-while-revalidate, ConnectivityManager NetworkCallback as a hint, Room as the single source of truth exposed as Flow, outbox pattern for offline writes with optimistic UI, cursor/delta pull sync with tombstones, conflict resolution, WorkManager background sync with constraints and expedited work, FCM-triggered sync, OkHttp WebSocket realtime with catch-up, large uploads/downloads, certificate pinning trade-offs, SQLCipher, Room migrations, and a short Apollo Kotlin GraphQL section (normalized cache, optimistic updates). Siblings: ios-network-offline-sync, react-native-network-offline-sync, flutter-network-offline-sync; related: android-multi-account, android-push-notifications, android-list-ux, android-crash-analytics, android-deep-links, android-in-app-purchase.
---

# Android Network, Offline & Sync Skill

Guide for building, auditing, and testing the **network layer and offline-first data layer** of a consumer Android app: how requests are made, how failures are modelled and retried, how data is cached, and how local writes made offline reach the server safely and in order.

The core rule throughout: **Room is the source of truth; the UI observes Room via `Flow`; the network only ever writes into Room (pull) or drains an outbox from Room (push).** The UI never renders a network response directly, except for genuinely online-only screens (search, checkout) that are marked as such.

Sibling skills: **ios-network-offline-sync**, **react-native-network-offline-sync**, **flutter-network-offline-sync** — same structure per platform. Related: **android-multi-account** (per-account clients, tokens, DBs), **android-push-notifications** (data messages that trigger sync), **android-list-ux** (Paging 3 + `RemoteMediator`, pull-to-refresh, empty/error states), **android-crash-analytics** (network traces, non-fatal reporting of sync failures), **android-deep-links** (opening an entity not yet synced), **android-in-app-purchase** (purchase verification must not go through the offline outbox blindly).

> Networking and Jetpack libraries move fast. OkHttp 5 (stable 2025) changed some APIs and artifacts (`okhttp-jvm`, `mockwebserver3`); Retrofit 3 targets OkHttp 4.12+/5; Retrofit has a first-party kotlinx.serialization converter since 2.11 (`com.squareup.retrofit2:converter-kotlinx-serialization`), replacing the Jake Wharton artifact; Ktor 3 changed client engines and plugins; Room 2.7+ is KMP with `BundledSQLiteDriver` options; WorkManager foreground-service rules tightened on API 34 (`foregroundServiceType` required) and API 35 (`dataSync` time limit); Apollo Kotlin 4 renamed many cache APIs and the normalized cache moved to a separate artifact (`com.apollographql.cache:normalized-cache`) in 2025. **Verify against the versions in the project's `libs.versions.toml` / BOM and current docs before writing code**, and prefer what the project already uses.
>
> Version gates used below: `ConnectivityManager.registerDefaultNetworkCallback` API 24+ (`NET_CAPABILITY_VALIDATED` API 23+); `network_security_config` API 24+; `setExpedited` effective API 31+ (falls back to a foreground service below, needs `getForegroundInfo`); `foregroundServiceType="dataSync"` required API 34+, 6-hour/day `dataSync` limit API 35+; user-initiated data transfer jobs (`JobInfo.setUserInitiated`) API 34+; `POST_NOTIFICATIONS` for progress notifications API 33+; TLS 1.3 by default API 29+.

## 0. Mental model

```
  UI (Compose)  <-- StateFlow<UiState> --  ViewModel
                                              |  collects Flow / calls commands
                                              v
  +---------------------------- Repository ----------------------------+
  |  observe(): Flow<T>  = dao.observe()   (ALWAYS from Room)           |
  |  refresh():  network -> mapper -> dao.upsert()  (in a transaction)  |
  |  mutate():   dao.applyOptimistic() + outbox.enqueue()  (one txn)    |
  +---------+--------------------------------------+-------------------+
            | reads/writes                         | schedules
            v                                      v
  Room DB (SoT)                              WorkManager
  entities | sync_state(cursor) | outbox      SyncWorker (pull) / OutboxWorker (push)
            ^                                      |
            |  upsert / tombstone delete           v
  +---------+-------------- Remote ------------------------------------+
  |  Retrofit/Ktor API  ->  OkHttpClient (single, configured)          |
  |   app interceptors:  [Headers] [Auth] [IdempotencyKey] [Retry]     |
  |   network interceptors: [Logging(debug)] [Metrics]                 |
  |   Authenticator (single-flight refresh)   Cache (disk, HTTP)       |
  |  WebSocket (realtime hints) -> triggers delta pull                 |
  +--------------------------------------------------------------------+
  Triggers for sync: app foreground, reconnect, push data msg, periodic, user pull-to-refresh, WS event
```

| Concern | Where it lives | Never |
|---|---|---|
| Truth for UI | Room `Flow` | `MutableStateFlow` filled from network response |
| Pending writes | `outbox` table | in-memory queue / ViewModel |
| Sync position | `sync_state` table (cursor per collection) | `SharedPreferences` separate from data (gets out of step) |
| HTTP caching | OkHttp `Cache` (per account dir) | hand-rolled response cache for ETag-able GETs |
| Retry policy | one place (interceptor or call helper) | scattered `repeat(3)` loops |
| Token refresh | `Authenticator` + `TokenManager` mutex | each repository |

## 1. First, establish context

Find out (from the codebase or by asking):

- **API style**: REST/JSON (assumed), GraphQL (section 18), gRPC? OpenAPI spec available (codegen vs hand-written Retrofit interfaces)? Pagination style (cursor vs offset)?
- **Client stack**: Retrofit + OkHttp, Ktor (OkHttp engine?), or both? Serializer (kotlinx.serialization / Moshi / Gson — Gson + Kotlin nullability is a bug source)?
- **Freshness**: how stale may each screen be? (Feed: minutes. Balance: must revalidate. Settings: hours.)
- **Offline mode**: none / read-only cache / **read-write** (outbox)? Which entities are writable offline? Which actions are online-only (payments, purchases, account deletion)?
- **Conflict tolerance**: single-user data (LWW fine) vs shared/collaborative (versions, field merge, prompt)? Does the server return versions/ETags per entity?
- **Server sync support**: `updatedSince`/cursor endpoint? Tombstones for deletes? Idempotency-Key support? Batch endpoint? Server time in responses?
- **Volume**: rows per user (100s vs 100k → paging, delta only), attachment sizes, upload frequency.
- **Realtime**: WebSocket/SSE/FCM data messages? Event ids with replay ("since event N")?
- **Environments/regions**: dev/staging/prod, per-account region (see android-multi-account)?
- **Security**: pinning requirement? Sensitive local data (health, finance → SQLCipher)? MDM?
- **Multi-account**: one client and DB per account? (Assumed yes if android-multi-account applies.)

Write a **sync contract** before coding:

```
API: REST/JSON, Retrofit 2.11 + OkHttp 4.12, kotlinx.serialization | envs: dev/stg/prod, region per account
offline: read-write for notes, tasks; read-only for feed; online-only: checkout, delete account
pull: GET /sync?collection=tasks&cursor=… -> {changes[], deletes[], nextCursor, hasMore}
push: POST/PATCH with Idempotency-Key, If-Match: <version>; 409 -> conflict body with server copy
conflicts: tasks LWW per field (server-authoritative: status, assignee); notes -> user prompt
realtime: WS /events, event ids, resume ?after=<id> | background: periodic 1h + FCM "sync" data msg
```

## 2. HTTP client architecture

- [ ] **One `OkHttpClient`** per process (or per account built via `base.newBuilder()` so pool and dispatcher are shared). Never `OkHttpClient()` inside a function or per request — leaks threads and connections, defeats the cache.
- [ ] One Retrofit/Ktor instance per base URL; base URL from an `ApiEnvironment` (env + region), injected, not `BuildConfig.BASE_URL` read in 30 places. Debug builds may allow a runtime env switcher; release builds never.
- [ ] Timeouts set explicitly: `connectTimeout` 10–15s, `readTimeout` 20–30s, `writeTimeout` 30s, `callTimeout` as the overall cap for normal calls (0 for uploads/downloads, which use their own client via `newBuilder()`).
- [ ] **Interceptor order is deliberate** (application interceptors run in order added, outermost first):
  1. `HeadersInterceptor` — `User-Agent` (app/version/OS), `Accept-Language`, `X-Request-Id`, app version, portal header.
  2. `AuthInterceptor` — adds `Authorization` from the token captured for this client/account.
  3. `IdempotencyKeyInterceptor` — copies a key from a request tag/header for unsafe methods (key generated at enqueue time, not here).
  4. `RetryInterceptor` (optional; or retry in the call helper) — only for idempotent requests, honours `Retry-After`.
  5. Network interceptors: `HttpLoggingInterceptor` (debug only, `HEADERS`, redact `Authorization`/`Cookie`), metrics/tracing (android-crash-analytics).
  - `Authenticator` handles 401 separately (section 3); `Cache` sits between app and network interceptors.
- [ ] `retryOnConnectionFailure(true)` (default) is fine — it retries only connection-level failures; don't stack your own retry on top without counting.
- [ ] Serialization: kotlinx.serialization `Json { ignoreUnknownKeys = true; explicitNulls = false; coerceInputValues = true }` — forward-compatible with new server fields and enum values (add an `UNKNOWN` fallback enum value).
- [ ] DTOs ≠ entities ≠ domain models. Mappers are pure functions and unit-tested.
- [ ] All network calls are `suspend` and main-safe (Retrofit suspend functions and Ktor already dispatch off main; your mapping/DB work uses `Dispatchers.IO`/`Default` as needed).
- [ ] Retrofit returns `Response<T>` (or a custom `CallAdapter` to `ApiResult<T>`) where you need status/headers; never let `HttpException` escape to the UI.
- [ ] Ktor alternative: `HttpClient(OkHttp) { engine { preconfigured = okHttpClient } install(ContentNegotiation){ json(json) } install(HttpTimeout) expectSuccess = false }` — reuse the same OkHttp client so interceptors, cache and pinning apply.

## 3. Auth

- [ ] `Authorization: Bearer <access>` added by an interceptor bound to **this account's** `TokenManager` (captured at client build time — see android-multi-account; never read "current session" at call time).
- [ ] 401 handled by `okhttp3.Authenticator`: if the failed request's token ≠ current token, just retry with the current one; else **single-flight refresh** behind a `Mutex`; retry **once** (`response.priorResponse != null` → give up).
- [ ] Refresh uses a **separate client without the Authenticator** (avoid recursion) but same TLS/pinning.
- [ ] `Authenticator` runs on an OkHttp thread; `runBlocking` there is acceptable only with a timeout; never on main.
- [ ] Refresh failure types: `invalid_grant` → `NeedsReauth` for that account (navigate to re-auth, keep outbox!); network failure → keep tokens, surface offline, don't log out.
- [ ] Proactive refresh when `expiresAt - skew < now` (skew 60s; server clock — see section 19 clock skew).
- [ ] Outbox and sync workers obtain tokens the same way; a worker hitting `NeedsReauth` returns `Result.failure()` with a flag, not `retry()` forever.
- [ ] 403 is **not** 401: permission/portal revoked — never trigger refresh or sign-out.

## 4. Error model

- [ ] One sealed type for every remote outcome; repositories return `ApiResult<T>` / throw `AppError` consistently — pick one style per codebase.

```
AppError
 ├─ Network.Offline          (UnknownHostException / no validated network)
 ├─ Network.Timeout          (SocketTimeoutException, InterruptedIOException)
 ├─ Network.Tls              (SSLHandshakeException, pinning failure — do NOT retry)
 ├─ Http(code, serverCode?, retryAfter?, body)   4xx/5xx with parsed {code,message}
 │    ├─ 401 Unauthorized → handled by Authenticator, surfaces as Auth.NeedsReauth
 │    ├─ 403 Forbidden, 404 NotFound, 409 Conflict(serverCopy), 410 Gone, 412 PreconditionFailed
 │    ├─ 422 Validation(fieldErrors), 429 RateLimited(retryAfter)
 │    └─ 5xx Server
 ├─ Decoding(cause)          (SerializationException — a bug or contract drift: report non-fatal)
 ├─ Auth.NeedsReauth
 └─ Unknown(cause)
```

- [ ] `CancellationException` is **never** caught and mapped to an error — rethrow it (common bug with `runCatching`).
- [ ] Map to user messages in one place (`AppError.toUserMessage()`): offline → "You're offline — changes will sync later"; timeout → "Taking longer than usual"; 5xx → "Something went wrong on our side"; 422 → field errors inline; Decoding → generic + report.
- [ ] Server error codes (`{"code":"QUOTA_EXCEEDED"}`) mapped to typed cases; unknown codes fall back to HTTP class.
- [ ] Report Decoding, unexpected 5xx bursts and sync failures as non-fatals (android-crash-analytics) with request id, not body/PII.
- [ ] Each error knows `isRetryable` (Offline, Timeout, 408, 429, 502, 503, 504; 500 only for idempotent requests).

## 5. Retries

- [ ] **Exponential backoff with full jitter**: `delay = random(0, min(cap, base * 2^attempt))`, base 500ms, cap 30s, max 3–5 attempts for interactive calls; WorkManager handles long-term backoff for background (`setBackoffCriteria(EXPONENTIAL, 30s)`).
- [ ] Retry automatically only **safe/idempotent** methods (GET, HEAD, PUT, DELETE) or POST/PATCH **with an Idempotency-Key** the server honours.
- [ ] **Idempotency key generated once per logical operation** (at outbox enqueue) and persisted; every retry, across process death and app restarts, sends the same key.
- [ ] Honour `Retry-After` (seconds or HTTP-date) on 429/503; it overrides your computed delay (cap it, e.g. 5 min, then defer to WorkManager).
- [ ] Never retry: 4xx except 408/429, TLS errors, Decoding, cancellations.
- [ ] **Circuit breaker** per host: after N consecutive 5xx/timeouts within a window, open for 30–60s — fail fast with `Server` error, show banner, stop background workers hammering the API; half-open after cooldown with one probe.
- [ ] Retries are counted and visible in logs/metrics; total time bounded by `callTimeout`/coroutine timeout.
- [ ] Don't retry on top of OkHttp's connection retry and Retrofit retry and WorkManager retry without thinking — multiplicative storms (3×3×∞).

## 6. Cancellation

- [ ] Calls run in lifecycle-bound scopes: `viewModelScope` for screen data; `applicationScope`/WorkManager for things that must finish (outbox, uploads). Leaving the screen cancels its GETs; it must not cancel a user's save.
- [ ] Coroutine cancellation cancels the OkHttp `Call` (Retrofit suspend and Ktor do this) — verify custom `CallAdapter`s propagate it.
- [ ] **Search-as-you-type**: `queryFlow.debounce(300).distinctUntilChanged().mapLatest { api.search(it) }` (or `flatMapLatest`) — stale responses never overwrite newer ones.
- [ ] **In-flight GET dedupe**: concurrent identical requests (two composables asking for the same profile) share one `Deferred` keyed by URL/params; removed on completion.
- [ ] `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), …)` so rotation doesn't refetch, backgrounding stops collection.
- [ ] Account switch cancels the old account's scope (android-multi-account) — responses for A never land in B's DB.
- [ ] Writes inside `withContext(NonCancellable)` only for the final DB commit of an already-received response, never for the network call.

## 7. Caching

- [ ] **Layers**: memory (small LRU for hot objects / in-flight dedupe) → Room (domain data, offline) → OkHttp disk `Cache` (HTTP semantics for GETs like config, images-by-URL, static content). Don't duplicate domain data in both Room and OkHttp cache and read from both.
- [ ] OkHttp `Cache(File(cacheDir, "http_<accountId>"), 50 MiB)` — per account dir; delete on sign-out.
- [ ] **ETag / If-None-Match**: OkHttp cache does conditional requests automatically when server sends `ETag`/`Last-Modified`; for Room-backed resources store the ETag on the entity/collection and send `If-None-Match` yourself; 304 → bump `fetchedAt`, keep rows.
- [ ] **Stale-while-revalidate** in the repository: emit Room immediately, refresh if `fetchedAt` older than the resource's TTL; show subtle "updating" indicator, not a blocking spinner.
- [ ] Offline reads of OkHttp cache: on `Offline`, retry with `CacheControl.FORCE_CACHE` (or a `max-stale` interceptor) for endpoints where stale is acceptable.
- [ ] Don't override server `Cache-Control` blindly with a network interceptor for every response — only for endpoints you own the semantics of.
- [ ] Never cache authenticated responses in a cache shared across accounts; `Cache-Control: private` is still stored by OkHttp (it's a private cache).
- [ ] Images: Coil/Glide own cache (per-account dir for private images).

## 8. Connectivity

- [ ] **Reachability is a hint, not truth**: `NET_CAPABILITY_VALIDATED` can be true while your API is down/captive; false-negatives exist on VPNs. Always attempt the request (or let WorkManager decide) and treat real failures as the signal.
- [ ] `ConnectivityManager.registerDefaultNetworkCallback` → `StateFlow<Connectivity>` (`Online(metered)`, `Offline`); `callbackFlow` + `awaitClose { unregister }`; seed with current state.
- [ ] Don't use deprecated `NetworkInfo` / `CONNECTIVITY_ACTION` broadcast.
- [ ] Offline banner: debounce ~1–2s to avoid flicker on network handover; also show "offline" when the last request failed with `Offline` even if the callback says online.
- [ ] On transition Offline → Online: enqueue unique `OutboxWorker` (KEEP) and a delta pull; reconnect WebSocket. Don't fire a burst of every screen's refresh.
- [ ] Metered networks: defer large downloads/uploads (`NetworkType.UNMETERED` constraint) unless user-initiated; respect Data Saver (`ConnectivityManager.restrictBackgroundStatus`).
- [ ] Permission: `ACCESS_NETWORK_STATE` in manifest.

## 9. Offline-first architecture

- [ ] **Room is the single source of truth**; DAOs expose `Flow<List<Entity>>`; repositories map to domain; ViewModels `stateIn`.
- [ ] Network results are written to Room inside `db.withTransaction { }` (upsert changes, delete tombstones, update cursor) — UI sees one consistent emission.
- [ ] Entities carry sync metadata: `serverVersion` (or ETag), `updatedAt` (server), `localState` (`SYNCED`, `PENDING`, `FAILED`, `CONFLICT`), optionally `localId` vs `serverId`.
- [ ] Local IDs: generate a client UUID for new entities so the outbox, optimistic rows and later server rows share an identity; server accepts client-generated ids, or keep a `localId → serverId` mapping updated on create ack (and rewrite dependent outbox payloads).
- [ ] Lists use Paging 3 `PagingSource` from Room + `RemoteMediator` for network pages (android-list-ux); don't page the network directly into UI for offline-capable lists.
- [ ] Platform DB options: **Room** (default, Flow, migrations, KMP in 2.7+); SQLDelight (KMP, SQL-first); DataStore only for small key-value/proto (settings, cursors if not in DB — prefer DB for cursors to commit atomically with data); Realm Kotlin (deprecated by MongoDB Atlas Device Sync EOL Sept 2025 — avoid for new work).
- [ ] Every screen has explicit states: content (possibly stale + timestamp), empty-offline ("Connect to load"), empty-online, error-with-cached-content (snackbar), loading-first-time.
- [ ] Online-only features are labelled and disabled offline with an explanation, not silently queued (payments, purchases — android-in-app-purchase).

## 10. Outbox for writes

- [ ] A mutation = **one Room transaction**: apply optimistic change to the entity (`localState = PENDING`) + insert `OutboxEntry(id, entityType, entityId, op, payloadJson, idempotencyKey, createdAt, attempts, state, lastError, dependsOn?)`.
- [ ] Then enqueue unique work `"outbox"` with `ExistingWorkPolicy.APPEND_OR_REPLACE` or KEEP + re-check loop; constraint `NetworkType.CONNECTED`; expedited for user-visible actions.
- [ ] **Ordered replay**: process FIFO by `createdAt`/seq **per entity** (and globally when ops depend on each other: create parent before child). Don't parallelise ops on the same entity.
- [ ] **Coalescing** (optional): multiple pending PATCHes to the same entity merged before send; CREATE followed by DELETE before send → drop both.
- [ ] Each entry has a **per-item state**: `PENDING → IN_FLIGHT → (done: delete row) | FAILED_RETRYABLE (attempts++) | FAILED_PERMANENT (4xx validation) | CONFLICT (409/412)`.
- [ ] A permanent failure **does not block** unrelated entries; it blocks only entries that depend on it. Surface it on the item ("Couldn't save — Retry / Discard") and in a "pending changes" view.
- [ ] On success: write server response (new version, server fields) into the entity, set `SYNCED`, delete outbox row — **same transaction**.
- [ ] Crash between send and ack: on restart the entry is `IN_FLIGHT` → reset to `PENDING` and resend with same idempotency key; server dedupes.
- [ ] Pull sync must **not overwrite** entities with pending outbox entries (rebase: apply server change, then re-apply pending local ops, or skip and let the outbox result resolve it).
- [ ] Outbox survives logout? Decide: warn the user "You have N unsynced changes" before sign-out; on re-auth keep it (android-multi-account).
- [ ] Optimistic UI rollback on permanent failure: restore from server copy (keep `serverSnapshot` or refetch the entity).
- [ ] Attachments: outbox entry references a local file; upload first (section 15), then the entity op with the uploaded id (`dependsOn`).

## 11. Sync (pull)

- [ ] **Cursor-based delta pull**: `GET /sync?collection=X&cursor=C&limit=500` → `{changes, deletes, nextCursor, hasMore}`; loop while `hasMore`; commit each page + cursor atomically.
- [ ] Prefer opaque server cursors over `updatedSince=<client time>` (clock skew, same-millisecond updates). If timestamp-based, use server time from the previous response and an overlap window, with idempotent upserts.
- [ ] **Tombstones**: server returns deleted ids (with deletion time); client deletes rows (or marks `deleted=true` if needed for undo/outbox). Without tombstones, deletes made elsewhere never disappear offline.
- [ ] **Full resync fallback**: server responds 410 Gone / `cursorExpired` (tombstone retention exceeded) → fetch full snapshot into a staging state, then swap in one transaction (keep pending outbox entities); also after DB migration that drops data, or schema version mismatch.
- [ ] Sync order: push outbox first, then pull (so the pull reflects your writes), or pull with rebase — be consistent.
- [ ] One sync at a time per account: unique work name `sync-<accountId>`, plus an in-process `Mutex`.
- [ ] Triggers: app foreground (debounced, if last sync > N min), reconnect, FCM data message, WebSocket event, periodic WorkManager, pull-to-refresh (user-initiated → run immediately, show result).
- [ ] Large initial sync: paginated, resumable via cursor, progress shown, screen usable as pages land.
- [ ] Store `lastSuccessfulSyncAt` per collection; show "Updated 5 min ago" when offline.

## 12. Conflict resolution

- [ ] Pick per entity type and write it in the contract:
  - **Last-write-wins** (server timestamp/version decides) — simple, fine for single-user, low-value fields.
  - **Optimistic concurrency**: send `If-Match: <version/ETag>`; server returns **412/409 with current server copy**; client decides.
  - **Field-level merge**: send only changed fields (PATCH with base version); auto-merge non-overlapping fields; overlapping → rule or prompt.
  - **User prompt**: "This note was changed on another device — Keep mine / Keep theirs / Keep both (duplicate)". Only for high-value content.
  - **Server-authoritative fields**: status, price, permissions, counters — never overwritten by client; client sends intents (`increment`, `complete`), not absolute values.
- [ ] Conflict state stored on the entity (`CONFLICT` + `serverSnapshotJson`) so it survives process death and the UI can render the choice.
- [ ] Deletes vs edits: edit on a remotely deleted entity → 404/410 → tell user, offer "restore as new".
- [ ] Don't compare client wall clocks across devices for LWW — use server versions.

## 13. Background sync (WorkManager)

- [ ] `CoroutineWorker` for `SyncWorker` and `OutboxWorker`, constraints `NetworkType.CONNECTED` (and `setRequiresBatteryNotLow(true)` for periodic/bulk).
- [ ] **Unique work** per account: `enqueueUniqueWork("outbox-<acct>", APPEND_OR_REPLACE/KEEP, …)`, `enqueueUniquePeriodicWork("sync-periodic-<acct>", UPDATE, PeriodicWorkRequest(1h, flex 15m))` — min period 15 min; Doze/App Standby buckets delay further.
- [ ] Workers take `accountId` from `inputData`, never "active account" (android-multi-account).
- [ ] `setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)` for user-visible writes; implement `getForegroundInfo()` for API < 31 (and API 34+ manifest `foregroundServiceType="dataSync"` on `SystemForegroundService`).
- [ ] Long transfers: `setForeground(ForegroundInfo(id, notification, FOREGROUND_SERVICE_TYPE_DATA_SYNC))`; API 35 limits `dataSync` FGS to ~6h/day — consider user-initiated data transfer jobs (API 34+) for big user-started uploads.
- [ ] `Result.retry()` uses WorkManager backoff; cap attempts via `runAttemptCount` → `Result.failure()` and mark items FAILED.
- [ ] Worker respects `isStopped` / cancellation (coroutines) and keeps work idempotent (it can be re-run after process death).
- [ ] **Push-triggered sync** (android-push-notifications): FCM data message `{type:"sync", accountId, collection}` → `onMessageReceived` enqueues expedited `SyncWorker` (don't do network work in `onMessageReceived` — ~10s budget). High priority only if user-visible; FCM may deprioritise if not.
- [ ] Hilt: `@HiltWorker` + `HiltWorkerFactory`; remove default initializer in manifest when using custom `Configuration.Provider`.
- [ ] Inspect with Background Task Inspector (Android Studio); test with `WorkManagerTestInitHelper` and `TestDriver`.

## 14. Realtime

- [ ] OkHttp `WebSocket` (or Ktor `webSocket`) on the shared client (`pingInterval(20s)` via `newBuilder()`); connect only while app is foreground (`ProcessLifecycleOwner` `STARTED`) — background updates come via FCM.
- [ ] **Events are hints**: on event, upsert small payloads into Room or trigger a delta pull for that collection; don't build a second state pipeline.
- [ ] **Reconnect with exponential backoff + jitter**, reset on successful open; reconnect immediately on network regain; stop on 4401/4403-style auth close codes → refresh token → reconnect once.
- [ ] **Catch-up**: track `lastEventId` (persisted); on reconnect `?after=<id>`; if server can't replay (gap too large) → delta pull from cursor.
- [ ] Dedupe events by id; ordering by server sequence; ignore events older than the entity's version.
- [ ] SSE alternative: OkHttp `okhttp-sse` `EventSources` with `Last-Event-ID`.
- [ ] Close socket on account switch and sign-out; one socket per active account/portal.

## 15. Large transfers

- [ ] Uploads/downloads run in WorkManager (foreground with progress notification) — not in a ViewModel.
- [ ] Separate client: `okHttp.newBuilder().callTimeout(0).readTimeout(60s).writeTimeout(60s)`.
- [ ] Stream from `File`/`Uri` (`ContentResolver.openInputStream`) with a custom `RequestBody` that reports progress via `setProgress(workDataOf("progress" to p))`; never load whole file into memory.
- [ ] **Resumable**: chunked/multipart upload (tus, S3 multipart with presigned URLs, GCS resumable) — persist `uploadId` and completed parts in Room; resume after process death.
- [ ] Downloads: `Range: bytes=<downloaded>-` with `If-Range: <etag>`; write to temp file, verify size/hash, atomic rename. `DownloadManager` is fine for public files, poor for authenticated ones.
- [ ] Compress images before upload (resize, JPEG/WebP quality) off main thread; strip EXIF location unless needed.
- [ ] Unmetered constraint for non-user-initiated large transfers.

## 16. Security

- [ ] HTTPS only: `network_security_config.xml` with `cleartextTrafficPermitted="false"`; debug-only `<debug-overrides>` trusting user CAs for Charles/Proxyman.
- [ ] **Certificate pinning trade-offs**: protects against compromised CAs/MITM on hostile networks, but a rotated cert without updated pins **bricks the app** for users who don't update. If required: pin SPKI hashes of intermediate/backup keys (≥2 pins), set `expiration` in network security config, have a kill switch / remote config, monitor. OkHttp `CertificatePinner` or `<pin-set>` — not both inconsistently.
- [ ] **No secrets in the APK**: API keys for your own backend are not secrets (extractable); use per-user tokens; third-party secret calls go through your server. Play Integrity / App Check for abuse resistance.
- [ ] Logging: `HttpLoggingInterceptor` only in debug, `redactHeader("Authorization")`, never `BODY` in release; no tokens/PII in crash breadcrumbs.
- [ ] Local DB encryption for sensitive data: SQLCipher (`net.zetetic:sqlcipher-android`) via `SupportOpenHelperFactory`, key wrapped by Android Keystore; or encrypt sensitive columns with Tink. Cost: performance, migration of existing plaintext DB (`sqlcipher_export`).
- [ ] Backups: exclude DB files with tokens/sensitive data in `dataExtractionRules` / `fullBackupContent`, or ensure restore works without keys (android-multi-account).
- [ ] Validate all server data (lengths, enums) — malicious/buggy payloads shouldn't crash mappers.

## 17. Migrations

- [ ] Room `version` bumped with every schema change; `exportSchema = true`, schemas committed; `AutoMigration` where possible; manual `Migration` for renames/splits; **`MigrationTestHelper` tests for every path** (N-1 → N, and old → N).
- [ ] **No `fallbackToDestructiveMigration()` in release** for DBs containing an outbox — silently loses unsynced user changes. If destructive is acceptable for cache tables, put caches in a separate DB or use `fallbackToDestructiveMigrationFrom(...)` selectively.
- [ ] **Outbox payload versioning**: store `payloadVersion` per entry; upgrade old payloads on read (or drain outbox with the old app version — impossible after update, so plan for it).
- [ ] After a migration that changes sync scope, reset cursors → full resync (keeping outbox).
- [ ] **API version compatibility**: send app version header; server supports N previous app versions; `426 Upgrade Required` / min-version config → forced-update screen; tolerant decoding (unknown fields/enums).
- [ ] Rolling back a server field rename breaks old clients — coordinate (expand/contract).

## 18. GraphQL (short)

Use **Apollo Kotlin** (v4) when the backend is GraphQL; the rest of this skill (outbox, WorkManager, connectivity) still applies.

- [ ] One `ApolloClient` per account with `.okHttpClient(accountClient)` (reuses auth/pinning) and `.serverUrl(env.graphqlUrl)`; codegen with `generateKotlinModels`, schema downloaded in CI.
- [ ] **Normalized cache**: `normalizedCache(MemoryCacheFactory(10 MiB).chain(SqlNormalizedCacheFactory("apollo_<acct>.db")))`; objects keyed by `__typename:id` (declare `@typePolicy(keyFields: "id")` in `extra.graphqls`) so one mutation result updates every query showing that object.
- [ ] Fetch policies: `CacheFirst` (default), `NetworkFirst` for freshness-critical, `CacheAndNetwork` for SWR via `.toFlow()`; `watch()` gives a Flow that re-emits on cache changes (Apollo cache as SoT for GraphQL screens).
- [ ] **Optimistic responses**: `apolloClient.mutation(m).optimisticUpdates(data)` writes to cache immediately; rolled back on error. For offline durability, still persist the mutation in your outbox — Apollo's optimistic layer is in-memory.
- [ ] Errors: GraphQL `errors[]` with HTTP 200 → map `response.errors` / `exception` into `AppError` (partial data is possible).
- [ ] Persisted queries (APQ) to reduce payload; batching optional.
- [ ] Apollo 4 API names differ from v3 (`ApolloResponse.exception`, cache artifact moved) — verify against the project's version.

## 19. Testing plan

1. **Unit**: mappers; `AppError` mapping from every exception/status; backoff calc (seeded `Random`); `Retry-After` parsing (seconds + HTTP-date); circuit breaker state machine; conflict resolver rules; outbox coalescing.
2. **MockWebServer** (`mockwebserver3` for OkHttp 5): interceptor order/headers; single-flight refresh with 20 concurrent 401s → one `/token`; 429 + `Retry-After`; 503 retries then success; same `Idempotency-Key` across retries; ETag → 304 path; `SocketPolicy.NO_RESPONSE` → timeout mapping; `throttleBody` slow responses; malformed JSON → Decoding.
3. **Room**: in-memory DB; DAO Flow emissions (Turbine); `withTransaction` atomicity (inject failure mid-sync → cursor unchanged); `MigrationTestHelper` for all versions incl. outbox table.
4. **Sync engine**: fake API with scripted pages: multi-page `hasMore`, tombstones, cursor-expired → full resync, pending outbox not overwritten by pull.
5. **WorkManager**: `WorkManagerTestInitHelper` + `TestDriver.setAllConstraintsMet`; unique work replacement; retry → failure after N attempts; `inputData` account id honoured.
6. **Network conditions**: emulator network profiles (`adb emu network speed gsm`, `network delay`), airplane mode toggles, Charles/Proxyman throttling and breakpoints (drop response after server processed → idempotency), captive portal Wi-Fi, VPN.
7. **Process death**: kill during outbox send (`adb shell am kill`), during multi-page sync, during upload → resumes correctly.
8. **Clock skew**: set device clock ±1 day / wrong timezone → token proactive refresh, LWW, "updated X ago" still correct (use server time).
9. **Doze**: `adb shell dumpsys deviceidle force-idle` → periodic sync deferred, FCM high-priority still triggers expedited work.
10. **Multi-device conflicts**: two emulators, same account, edit offline on both, reconnect.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| N1 | Cold start offline with cached data | Content from Room + "Offline · updated 2h ago"; no error dialog |
| N2 | Cold start offline, never synced | Empty-offline state with retry; no spinner forever |
| N3 | Create item offline | Appears instantly (pending badge); synced on reconnect; badge cleared |
| N4 | Edit same item 3× offline | Coalesced or ordered; server ends with final value |
| N5 | Create then delete offline | Nothing sent (or create+delete in order); no ghost item |
| N6 | Response lost after server applied POST | Retry with same Idempotency-Key; no duplicate |
| N7 | Validation error (422) on queued item | Item marked failed with message; other items still sync |
| N8 | 409 conflict on note | Conflict UI: mine / theirs / both; choice persists across restart |
| N9 | Item deleted on web, client online | Tombstone removes it on next sync |
| N10 | Item deleted on web, edited offline here | 404/410 → "deleted elsewhere — restore?" |
| N11 | Cursor expired (offline 90 days) | Full resync; pending outbox preserved |
| N12 | Access token expired, 10 parallel calls | One refresh; all retried once |
| N13 | Refresh token revoked with pending outbox | Re-auth screen; outbox kept and drained after login |
| N14 | 429 with Retry-After: 30 | Waits ≥30s; no hammering |
| N15 | Backend down (5xx burst) | Circuit opens; banner; workers back off; recovers after |
| N16 | Wi-Fi with captive portal | Treated as offline after failed request; banner; no data corruption |
| N17 | Network flaps every 2s | Banner debounced; one outbox run; no duplicate sends |
| N18 | Type fast in search | Only latest query's results shown |
| N19 | Leave screen during load | Request cancelled; no crash writing to dead UI |
| N20 | Rotate during load | No refetch; state kept |
| N21 | Kill app mid-outbox send | Resent on next run with same key |
| N22 | Kill app mid multi-page sync | Resumes from last committed cursor |
| N23 | Upload 200 MB video, background app | Foreground notification with progress; completes |
| N24 | Upload interrupted at 60% | Resumes from last part, not 0 |
| N25 | FCM "sync" data message while app killed | Expedited work runs; data fresh on open |
| N26 | WebSocket drop for 5 min | Reconnect with backoff; missed events caught up |
| N27 | App update with DB migration and pending outbox | Migration succeeds; outbox drained |
| N28 | Server adds new enum value / field | No crash; UNKNOWN handled |
| N29 | Device clock 1 day ahead | Auth and sync still work |
| N30 | Switch account mid-sync | A's results never written to B's DB |
| N31 | Pinned cert rotated (if pinning) | Backup pin works; no outage |
| N32 | Metered network, large background download | Deferred until unmetered (unless user-initiated) |
| N33 | 426 Upgrade Required | Forced update screen; outbox preserved |

## 20. Common bugs to look for in review

- `OkHttpClient()` / `Retrofit.Builder()` created per call or per repository → thread/socket leaks, no shared cache
- `HttpLoggingInterceptor.Level.BODY` in release; tokens in logs
- `runCatching { api.call() }` swallowing `CancellationException` → zombie coroutines, wrong error UI
- UI built from network response, then also from Room → flicker / two truths; offline shows nothing
- Pull sync overwrites optimistic local edits (pending outbox) → user's change "reverts", then reappears
- Cursor saved outside the data transaction → gaps or duplicate application after crash
- No tombstones → deleted items live forever on device
- `updatedSince` from device clock → missed changes with clock skew
- POST retried without idempotency key → duplicate orders/messages
- Idempotency key generated per HTTP attempt instead of per operation
- Retrying 4xx (validation) forever; outbox head failure blocks every later item
- `fallbackToDestructiveMigration()` with an outbox → unsynced user data silently deleted on update
- `isConnected` check before every call ("no network" dialog on captive/VPN false negatives) instead of trying
- Deprecated `NetworkInfo`/`CONNECTIVITY_ACTION`; NetworkCallback never unregistered
- Authenticator refreshing on 403, or without `priorResponse` check → infinite loop; refresh via the same client → recursion
- Network calls in `FirebaseMessagingService.onMessageReceived` instead of enqueuing work
- Periodic work enqueued with `REPLACE` on every app start → never runs (timer resets)
- Workers reading the active account instead of `inputData`
- Missing `foregroundServiceType` on API 34+ → crash when expedited work falls back to foreground
- WebSocket reconnect without backoff → battery drain, server DDoS; no catch-up after reconnect
- Search without `mapLatest`/`flatMapLatest` → stale results overwrite fresh ones
- `Json` without `ignoreUnknownKeys` → app crashes when backend adds a field
- Gson with non-null Kotlin types → nulls in non-null fields, crash later
- Certificate pinning with one leaf pin and no expiry → outage on cert rotation
- HTTP cache dir shared across accounts
- Large file read into `ByteArray` for upload → OOM
- `Dispatchers.Main` for JSON parsing of big payloads → jank
- Optimistic UI with no rollback on permanent failure

## 21. Output format

When asked to **review/audit**:
1. **Summary** — sync contract (API style, offline mode, conflict strategy, triggers), stack and versions (OkHttp, Retrofit/Ktor, serialization, Room, WorkManager, Apollo)
2. **Checklist results** — sections 2–18 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** (data loss: destructive migration, overwritten pending edits, missing idempotency → duplicates; security: logging tokens, cleartext, bad pinning; logout loops) with fix code
4. **Testing gaps** — N1–N33 not covered; which need real devices/network conditions

When asked to **implement**: write the sync contract first, then adapt the Appendix: client + interceptors, error model, one repository end-to-end (network → Room → UI), outbox processor, sync engine, WorkManager registration. Delete what the contract doesn't need (realtime, conflict prompt, uploads). Match library versions in `libs.versions.toml`, verify APIs compile, run MockWebServer + Room tests, and list the scenario rows you verified vs left for the user.

---

## Appendix — Reference implementation (Kotlin, Hilt, OkHttp, Retrofit, kotlinx.serialization, Room, WorkManager)

> Written against Kotlin 2.x, OkHttp 4.12, Retrofit 2.11 (first-party kotlinx converter), kotlinx.serialization 1.7, Room 2.6/2.7, WorkManager 2.9/2.10, Hilt 2.5x. Imports omitted for brevity. Verify signatures against the project's versions (OkHttp 5 / Retrofit 3 are mostly source-compatible).

### A.1 Configured client and interceptors

```kotlin
data class ApiEnvironment(val baseUrl: String, val appVersion: String)

class HeadersInterceptor(private val env: ApiEnvironment) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request().newBuilder()
            .header("User-Agent", "MyApp/${env.appVersion} (Android ${Build.VERSION.RELEASE})")
            .header("Accept-Language", Locale.getDefault().toLanguageTag()).header("X-App-Version", env.appVersion)
            .header("X-Request-Id", UUID.randomUUID().toString()).build()
        return chain.proceed(req)
    }
}

// AuthInterceptor(tokens): adds "Authorization: Bearer ${tokens.currentAccessToken()}" when a token exists.

/** Key is generated once per logical operation and passed as a request tag. */
data class IdempotencyKey(val value: String)

class IdempotencyKeyInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val key = chain.request().tag(IdempotencyKey::class.java) ?: return chain.proceed(chain.request())
        return chain.proceed(chain.request().newBuilder().header("Idempotency-Key", key.value).build()) }
}

class TokenAuthenticator(private val tokens: TokenManager) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.priorResponse != null) return null            // already retried once
        val sent = response.request.header("Authorization")?.removePrefix("Bearer ")
        val fresh = runBlocking { withTimeoutOrNull(15_000) { tokens.refreshIfStale(sent) } } ?: return null
        return response.request.newBuilder().header("Authorization", "Bearer $fresh").build()
    }
}

class TokenManager(private val store: TokenStore, private val refreshApi: RefreshApi) {
    private val mutex = Mutex()
    fun currentAccessToken(): String? = store.access()

    /** Single-flight: only one refresh runs; callers with an outdated token just get the new one. */
    suspend fun refreshIfStale(sentToken: String?): String? = mutex.withLock {
        val current = store.access()
        if (current != null && current != sentToken) return current
        val refresh = store.refresh() ?: return null
        return try {
            val r = refreshApi.refresh(RefreshBody(refresh))
            store.save(r.accessToken, r.refreshToken); r.accessToken
        } catch (e: HttpException) { if (e.code() in 400..401) store.markNeedsReauth(); null }  // invalid_grant catch (e: IOException) { null }                          // offline: keep tokens
    }
}

@Module @InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun json(): Json = Json { ignoreUnknownKeys = true; explicitNulls = false; coerceInputValues = true }

    @Provides @Singleton
    fun okHttp(@ApplicationContext ctx: Context, env: ApiEnvironment, tokens: TokenManager): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(15, TimeUnit.SECONDS).readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS).callTimeout(60, TimeUnit.SECONDS)
            .cache(Cache(File(ctx.cacheDir, "http"), 50L * 1024 * 1024)) // per account dir in multi-account
            .addInterceptor(HeadersInterceptor(env))
            .addInterceptor(AuthInterceptor(tokens))
            .addInterceptor(IdempotencyKeyInterceptor())
            .authenticator(TokenAuthenticator(tokens))
            // DEBUG only: addNetworkInterceptor(HttpLoggingInterceptor(HEADERS)) with Authorization/Cookie redacted
            .build()

    @Provides @Singleton
    fun retrofit(client: OkHttpClient, env: ApiEnvironment, json: Json): Retrofit =
        Retrofit.Builder().baseUrl(env.baseUrl).client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType())).build()

    @Provides @Singleton fun tasksApi(r: Retrofit): TasksApi = r.create()
}
```

### A.2 Typed errors, safe call and retry

```kotlin
sealed class AppError(cause: Throwable? = null) : Exception(cause) {
    data object Offline : AppError(); data object Timeout : AppError()
    class Tls(c: Throwable) : AppError(c); class Decoding(c: Throwable) : AppError(c); class Unknown(c: Throwable) : AppError(c)
    class Http(val code: Int, val serverCode: String?, val retryAfterMs: Long?, val body: String?) : AppError()
    data object NeedsReauth : AppError()

    val isRetryable: Boolean get() = when (this) {
        Offline, Timeout -> true
        is Http -> code == 408 || code == 429 || code in 502..504
        else -> false
    }
}

@Serializable data class ServerError(val code: String? = null, val message: String? = null)

fun Throwable.toAppError(json: Json): AppError = when (this) {
    is CancellationException -> throw this                        // never map cancellation
    is AppError -> this
    is UnknownHostException, is ConnectException -> AppError.Offline
    is SocketTimeoutException, is InterruptedIOException -> AppError.Timeout
    is SSLException -> AppError.Tls(this)
    is SerializationException -> AppError.Decoding(this)
    is HttpException -> {
        val raw = response()?.errorBody()?.string()
        val parsed = raw?.let { runCatching { json.decodeFromString<ServerError>(it) }.getOrNull() }
        AppError.Http(code(), parsed?.code, parseRetryAfter(response()?.headers()?.get("Retry-After")), raw)
    }
    else -> AppError.Unknown(this)
}

fun parseRetryAfter(v: String?): Long? = v?.let {                 // delta-seconds or RFC 1123 HTTP-date
    it.toLongOrNull()?.times(1000) ?: runCatching { (ZonedDateTime.parse(it, DateTimeFormatter.RFC_1123_DATE_TIME)
        .toInstant().toEpochMilli() - System.currentTimeMillis()).coerceAtLeast(0) }.getOrNull()
}

/** Retries only when the caller says the operation is idempotent (GET, or POST with Idempotency-Key). */
suspend fun <T> withRetry(
    json: Json, idempotent: Boolean, maxAttempts: Int = 4, baseMs: Long = 500, capMs: Long = 30_000,
    random: Random = Random.Default, block: suspend () -> T,
): T {
    var attempt = 0
    while (true) {
        try { return block() } catch (t: Throwable) {
            val err = t.toAppError(json); attempt++
            if (!idempotent || !err.isRetryable || attempt >= maxAttempts) throw err
            val backoff = random.nextLong(0, minOf(capMs, baseMs shl attempt) + 1)   // full jitter
            delay((err as? AppError.Http)?.retryAfterMs?.coerceAtMost(capMs) ?: backoff)
        }
    }
}

// AppError.userMessage(res): one `when` mapping each case to a string resource (section 4).
```

### A.3 API, entities, DAO

```kotlin
@Serializable data class TaskDto(val id: String, val title: String, val done: Boolean,
                                 val version: Long, val updatedAt: String)
@Serializable data class SyncPage(val changes: List<TaskDto>, val deletes: List<String>,
                                  val nextCursor: String?, val hasMore: Boolean)
@Serializable data class TaskPatch(val title: String? = null, val done: Boolean? = null)

interface TasksApi {
    @GET("sync/tasks") suspend fun pull(@Query("cursor") cursor: String?, @Query("limit") limit: Int = 500): SyncPage
    @PUT("tasks/{id}") suspend fun create(@Path("id") id: String, @Body body: TaskDto, @Tag key: IdempotencyKey): TaskDto
    @PATCH("tasks/{id}") suspend fun patch(@Path("id") id: String, @Header("If-Match") version: Long,
                                           @Body body: TaskPatch, @Tag key: IdempotencyKey): TaskDto
    @DELETE("tasks/{id}") suspend fun delete(@Path("id") id: String, @Tag key: IdempotencyKey): Response<Unit>
}

enum class LocalState { SYNCED, PENDING, FAILED, CONFLICT }

@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey val id: String,                 // client-generated UUID, accepted by server
    val title: String, val done: Boolean,
    val serverVersion: Long, val localState: LocalState,
    val serverSnapshot: String? = null,         // for rollback / conflict UI
)

enum class OutboxOp { CREATE, PATCH, DELETE }
enum class OutboxState { PENDING, IN_FLIGHT, FAILED_RETRYABLE, FAILED_PERMANENT, CONFLICT }

@Entity(tableName = "outbox", indices = [Index("entityId"), Index("state")])
data class OutboxEntry(
    @PrimaryKey(autoGenerate = true) val seq: Long = 0,
    val entityType: String, val entityId: String, val op: OutboxOp, val payload: String, val payloadVersion: Int = 1,
    val idempotencyKey: String = UUID.randomUUID().toString(),
    val state: OutboxState = OutboxState.PENDING, val attempts: Int = 0, val lastError: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
)

@Entity(tableName = "sync_state")
data class SyncState(@PrimaryKey val collection: String, val cursor: String?, val lastSyncAt: Long)

@Dao interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY title") fun observeAll(): Flow<List<TaskEntity>>
    @Query("SELECT * FROM tasks WHERE id = :id") suspend fun get(id: String): TaskEntity?
    @Upsert suspend fun upsert(items: List<TaskEntity>)
    @Query("DELETE FROM tasks WHERE id IN (:ids)") suspend fun deleteIds(ids: List<String>)
    @Query("DELETE FROM tasks") suspend fun deleteAll()
}

@Dao interface OutboxDao {
    @Insert suspend fun insert(e: OutboxEntry): Long
    @Update suspend fun update(e: OutboxEntry)
    @Query("DELETE FROM outbox WHERE seq = :seq") suspend fun delete(seq: Long)
    /** Oldest ready entry whose entity has no earlier blocked entry: a failure blocks only its own entity. */
    @Query("""
        SELECT * FROM outbox o
        WHERE o.state IN ('PENDING','FAILED_RETRYABLE')
          AND NOT EXISTS (SELECT 1 FROM outbox b WHERE b.entityId = o.entityId AND b.seq < o.seq
                          AND b.state IN ('FAILED_PERMANENT','CONFLICT'))
        ORDER BY o.seq LIMIT 1""")
    suspend fun nextReady(): OutboxEntry?
    @Query("SELECT DISTINCT entityId FROM outbox") suspend fun pendingEntityIds(): List<String>
    @Query("UPDATE outbox SET state = 'PENDING' WHERE state = 'IN_FLIGHT'") suspend fun resetInFlight()
    @Query("SELECT COUNT(*) FROM outbox") fun observeCount(): Flow<Int>
}

@Dao interface SyncStateDao {
    @Query("SELECT * FROM sync_state WHERE collection = :c") suspend fun get(c: String): SyncState?
    @Upsert suspend fun put(s: SyncState)
}

@Database(entities = [TaskEntity::class, OutboxEntry::class, SyncState::class], version = 3, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun tasks(): TaskDao; abstract fun outbox(): OutboxDao; abstract fun syncState(): SyncStateDao
}
// Room.databaseBuilder(ctx, AppDatabase::class.java, "acct_$accountId.db").addMigrations(M1_2, M2_3).build()
// NO fallbackToDestructiveMigration(): the outbox holds unsynced user changes.
```


### A.4 Repository (network → DB → UI)

```kotlin
class TaskRepository @Inject constructor(
    private val db: AppDatabase, private val json: Json, private val syncScheduler: SyncScheduler,
) {
    val tasks: Flow<List<Task>> = db.tasks().observeAll().map { list -> list.map { it.toDomain() } }
    val pendingCount: Flow<Int> = db.outbox().observeCount()

    suspend fun create(title: String) {
        val id = UUID.randomUUID().toString()
        db.withTransaction {                                         // entity + outbox row in one transaction
            db.tasks().upsert(listOf(TaskEntity(id, title, false, serverVersion = 0, localState = LocalState.PENDING)))
            db.outbox().insert(OutboxEntry(entityType = "task", entityId = id, op = OutboxOp.CREATE,
                payload = json.encodeToString(TaskDto(id, title, false, 0, ""))))
        }
        syncScheduler.kickOutbox(expedited = true)
    }

    suspend fun toggleDone(id: String) {
        db.withTransaction {
            val cur = db.tasks().get(id) ?: return@withTransaction
            val snapshot = cur.serverSnapshot ?: json.encodeToString(cur.toDto())
            db.tasks().upsert(listOf(cur.copy(done = !cur.done, localState = LocalState.PENDING, serverSnapshot = snapshot)))
            db.outbox().insert(OutboxEntry(entityType = "task", entityId = id, op = OutboxOp.PATCH,
                payload = json.encodeToString(TaskPatch(done = !cur.done))))
        }
        syncScheduler.kickOutbox(expedited = true)
    }

    /** Pull-to-refresh: user-initiated, runs now; errors surface to the caller for a snackbar. */
    suspend fun refresh() = syncScheduler.syncNow()
}

// TasksViewModel: combine(repo.tasks, repo.pendingCount, connectivity.status) into TasksUiState(items, pending, offline)
// .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), TasksUiState()) — UI observes Room only.
```

### A.5 Outbox processor

```kotlin
class OutboxProcessor @Inject constructor(
    private val db: AppDatabase, private val api: TasksApi, private val json: Json,
) {
    private val mutex = Mutex()

    sealed interface RunResult { data object Done : RunResult; data object RetryLater : RunResult; data object NeedsReauth : RunResult }

    suspend fun drain(): RunResult = mutex.withLock {
        db.outbox().resetInFlight()                                  // crash recovery: resend with same key
        var result: RunResult = RunResult.Done
        while (result == RunResult.Done) { result = process(db.outbox().nextReady() ?: break) }
        result
    }

    /** Returns Done to continue draining, or a stop reason. */
    private suspend fun process(e: OutboxEntry): RunResult {
        db.outbox().update(e.copy(state = OutboxState.IN_FLIGHT))
        return try {
            val server = send(e)
            db.withTransaction {
                if (server != null) db.tasks().upsert(listOf(server.toEntity(LocalState.SYNCED)))
                else if (e.op == OutboxOp.DELETE) db.tasks().deleteIds(listOf(e.entityId))
                db.outbox().delete(e.seq)
            }
            RunResult.Done
        } catch (t: Throwable) {
            val err = t.toAppError(json)
            when {
                err is AppError.NeedsReauth -> { db.outbox().update(e.copy(state = OutboxState.PENDING)); RunResult.NeedsReauth }
                err is AppError.Http && (err.code == 409 || err.code == 412) -> { markConflict(e, err); RunResult.Done }
                err.isRetryable -> {                                 // keep order: stop, let WorkManager back off
                    db.outbox().update(e.copy(state = OutboxState.FAILED_RETRYABLE,
                        attempts = e.attempts + 1, lastError = err.toString()))
                    RunResult.RetryLater
                }
                else -> {                                            // 4xx validation: blocks only this entity
                    db.withTransaction {
                        db.outbox().update(e.copy(state = OutboxState.FAILED_PERMANENT, lastError = err.toString()))
                        db.tasks().get(e.entityId)?.let { db.tasks().upsert(listOf(it.copy(localState = LocalState.FAILED))) }
                    }
                    RunResult.Done
                }
            }
        }
    }

    private suspend fun send(e: OutboxEntry): TaskDto? {
        val key = IdempotencyKey(e.idempotencyKey)
        return when (e.op) {
            OutboxOp.CREATE -> api.create(e.entityId, json.decodeFromString(e.payload), key)
            OutboxOp.PATCH -> api.patch(e.entityId, db.tasks().get(e.entityId)?.serverVersion ?: 0, json.decodeFromString(e.payload), key)
            OutboxOp.DELETE -> {
                val r = api.delete(e.entityId, key)
                if (!r.isSuccessful && r.code() != 404) throw HttpException(r); null  // 404: already gone = success
            }
        }
    }

    private suspend fun markConflict(e: OutboxEntry, err: AppError.Http) = db.withTransaction {
        db.outbox().update(e.copy(state = OutboxState.CONFLICT, lastError = "conflict"))
        db.tasks().get(e.entityId)?.let { db.tasks().upsert(listOf(it.copy(localState = LocalState.CONFLICT, serverSnapshot = err.body))) }
    }
}
```

### A.6 Sync engine (cursor + tombstones + rebase)

```kotlin
class SyncEngine @Inject constructor(
    private val db: AppDatabase, private val api: TasksApi, private val json: Json,
    private val outbox: OutboxProcessor,
) {
    private val mutex = Mutex()

    suspend fun sync(): OutboxProcessor.RunResult = mutex.withLock {
        val push = outbox.drain()                                     // push first, then pull
        if (push == OutboxProcessor.RunResult.NeedsReauth) return push
        try { pullAll("tasks") } catch (e: Throwable) {
            val err = e.toAppError(json)
            if (err is AppError.Http && err.code == 410) fullResync("tasks") else throw err
        }
        push
    }

    private suspend fun pullAll(collection: String) {
        var cursor = db.syncState().get(collection)?.cursor
        do {
            val page = withRetry(json, idempotent = true) { api.pull(cursor) }
            db.withTransaction {
                val pending = db.outbox().pendingEntityIds().toSet()
                // Rebase: don't overwrite rows with local pending edits; outbox ack will bring server state.
                db.tasks().upsert(page.changes.filter { it.id !in pending }.map { it.toEntity(LocalState.SYNCED) })
                db.tasks().deleteIds(page.deletes.filter { it !in pending })
                db.syncState().put(SyncState(collection, page.nextCursor, System.currentTimeMillis()))
            }
            cursor = page.nextCursor
        } while (page.hasMore)
    }

    /** Cursor expired (410): page through a full snapshot with cursor = null, then in ONE transaction
     *  deleteAll() + upsert snapshot rows (minus entities with pending outbox entries, which are kept as-is)
     *  and store the new cursor. */
    private suspend fun fullResync(collection: String) { /* as described above */ }
}
```

### A.7 Connectivity observer

```kotlin
enum class Connectivity { Online, Offline }

// @Singleton ConnectivityObserver(@ApplicationContext ctx, @AppScope scope):
//   status: StateFlow<Connectivity> = callbackFlow { registerDefaultNetworkCallback; on onCapabilitiesChanged /
//   onLost trySend(current()); awaitClose { unregisterNetworkCallback } } where current() = Online only if the
//   active network has NET_CAPABILITY_INTERNET && NET_CAPABILITY_VALIDATED.
//   .debounce(1_000).distinctUntilChanged().stateIn(scope, Eagerly, Online) — a hint only: requests still decide.
```

### A.8 Background sync registration (WorkManager)

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted ctx: Context, @Assisted params: WorkerParameters,
    private val engines: SyncEngineProvider,                          // resolves per-account engine
) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        val accountId = inputData.getString(KEY_ACCOUNT) ?: return Result.failure()
        return try {
            when (engines.forAccount(accountId).sync()) {
                OutboxProcessor.RunResult.Done -> Result.success()
                OutboxProcessor.RunResult.RetryLater -> if (runAttemptCount < 8) Result.retry() else Result.failure()
                OutboxProcessor.RunResult.NeedsReauth -> Result.failure(workDataOf("reauth" to true))
            }
        } catch (e: CancellationException) { throw e
        } catch (e: AppError) { if (e.isRetryable && runAttemptCount < 8) Result.retry() else Result.failure() }
    }

    override suspend fun getForegroundInfo(): ForegroundInfo =                // expedited fallback < API 31
        ForegroundInfo(NOTIF_ID, syncNotification(applicationContext), if (Build.VERSION.SDK_INT >= 29) ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC else 0)

    companion object { const val KEY_ACCOUNT = "accountId"; private const val NOTIF_ID = 4201 }
}

class SyncScheduler @Inject constructor(
    private val wm: WorkManager, private val accountId: String,        // provided by account-scoped graph
    private val engine: SyncEngine,
) {
    private val constraints = Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build()
    private fun data() = workDataOf(SyncWorker.KEY_ACCOUNT to accountId)

    fun kickOutbox(expedited: Boolean) {
        val req = OneTimeWorkRequestBuilder<SyncWorker>().setConstraints(constraints).setInputData(data())
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
            .apply { if (expedited) setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST) }.build()
        wm.enqueueUniqueWork("sync-$accountId", ExistingWorkPolicy.APPEND_OR_REPLACE, req)
    }

    // schedulePeriodic(): PeriodicWorkRequestBuilder<SyncWorker>(1h, flex 15m), CONNECTED + battery-not-low,
    // enqueueUniquePeriodicWork("sync-periodic-$accountId", ExistingPeriodicWorkPolicy.UPDATE, req) — not REPLACE.

    suspend fun syncNow() = engine.sync()                              // pull-to-refresh, in foreground

    fun cancelAll() { wm.cancelUniqueWork("sync-$accountId"); wm.cancelUniqueWork("sync-periodic-$accountId") }
}

// Triggers: ConnectivityObserver Offline->Online and ProcessLifecycleOwner onStart call kickOutbox().
// FCM: onMessageReceived(type=="sync") builds the same expedited request with msg.data["accountId"]
// and enqueueUniqueWork("sync-$acct", APPEND_OR_REPLACE, req) -- never network work in the service.
```

Manifest (API 34+), when expedited work may fall back to a foreground service: `FOREGROUND_SERVICE_DATA_SYNC` permission and `androidx.work.impl.foreground.SystemForegroundService` merged with `foregroundServiceType="dataSync"`.

### A.9 Realtime WebSocket (sketch)

```kotlin
// Shared client: okHttp.newBuilder().pingInterval(20, SECONDS). Run only while app is STARTED.
// loop: open ws "$url?after=${lastEventId()}" -> onOpen: attempt = 0, trigger delta pull (catch-up)
//       onMessage: dedupe by event id, upsert into Room or kick SyncEngine for that collection
//       onClosed/onFailure: complete a CompletableDeferred; then
//       delay(Random.nextLong(0, min(60_000, 1_000L shl min(attempt++, 6)) + 1)) and reconnect
// Auth close code (e.g. 4401): refresh token once, then reconnect. stop() on account switch / sign-out.
```

### A.10 Test (MockWebServer: idempotency key reused across retries)

```kotlin
// MockWebServer: enqueue 503 (Retry-After: 0) then 200; call api.create(..., IdempotencyKey("k-1")) inside
// withRetry(idempotent = true); assert both recorded requests carry Idempotency-Key "k-1".
// Single-flight: dispatcher returns 401 unless "Bearer new"; launch 20 calls; assert exactly one "/token" request.
```
