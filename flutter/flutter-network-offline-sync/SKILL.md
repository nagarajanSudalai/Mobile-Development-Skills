---
name: flutter-network-offline-sync
description: Use when implementing, reviewing, testing, or debugging networking, caching, offline support, and data sync in a Flutter consumer app — a single configured dio client with ordered interceptors (auth with single-flight refresh via QueuedInterceptor, retry with backoff + jitter, idempotency keys, logging), typed DioException-to-AppError mapping, cancellation with CancelToken, HTTP caching with dio_cache_interceptor (ETag / stale-while-revalidate), connectivity_plus as a hint, Drift as the local source of truth with watch() streams, an outbox of queued mutations with optimistic UI, cursor/delta sync with tombstones, conflict resolution, background sync with workmanager or background_fetch, realtime via web_socket_channel with catch-up, large uploads/downloads, certificate pinning, DB encryption and migrations, plus a short GraphQL section (graphql_flutter / ferry normalized cache and optimistic responses). Siblings: ios-network-offline-sync, android-network-offline-sync, react-native-network-offline-sync; related: flutter-multi-account, flutter-push-notifications, flutter-deep-links, flutter-crash-analytics, flutter-list-ux, flutter-in-app-purchase.
---

# Flutter Network, Offline & Sync Skill

Guide for building, auditing, and testing the data layer of a consumer Flutter app: how requests leave the device, how failures come back, how data is cached and stored locally, how writes made offline reach the server, and how local and server state are reconciled. REST is the primary style; GraphQL gets a short section (§18). Siblings: **ios-network-offline-sync**, **android-network-offline-sync**, **react-native-network-offline-sync**. Related: **flutter-multi-account** (per-account clients and DBs), **flutter-push-notifications** (push-triggered sync), **flutter-deep-links** (opening content that may not be synced yet), **flutter-crash-analytics** (network breadcrumbs, error reporting), **flutter-list-ux** (pagination, pull-to-refresh, empty/error states), **flutter-in-app-purchase** (receipt upload must never be lost — use the outbox).

Flutter packages move fast. Before shipping, **verify the current API** of: `dio` (v5: `DioException` / `DioExceptionType`, `QueuedInterceptor`, `IOHttpClientAdapter.createHttpClient` and `validateCertificate`), `dio_cache_interceptor` (v4 renamed options such as `hitCacheOnErrorExcept` → `hitCacheOnNetworkFailure` / `hitCacheOnErrorCodes`; stores moved to separate packages `http_cache_drift_store`, `http_cache_hive_store`, `http_cache_file_store`), `connectivity_plus` (v6+ `onConnectivityChanged` emits `List<ConnectivityResult>`, not a single value), `drift` / `drift_flutter` (`driftDatabase(name:)`, `NativeDatabase.createInBackground`, `MigrationStrategy`, `drift_dev schema` steps), `workmanager` (iOS needs `BGTaskSchedulerPermittedIdentifiers`; the plugin's iOS API changed across 0.5 → 0.6+ — check `registerPeriodicTask` support on iOS for your version), `background_fetch` (`BackgroundFetch.configure`, `finish(taskId)`, headless Android task), `web_socket_channel` (v3: `await channel.ready`, `IOWebSocketChannel.connect` for headers/pingInterval), `background_downloader`, `graphql_flutter` (v5 `GraphQLCache(store: HiveStore())`), `ferry` (`ferry_hive_store` / `ferry_hive_ce_store`), and Riverpod 3's experimental offline persistence if you use it. Check `pubspec.yaml` and `flutter --version`; don't trust memory.

Version gates worth knowing: Dart 3 (sealed classes, records, patterns) for the error model; Android 14+ restricts background work and foreground-service types (long uploads need a declared `dataSync` type or should use `background_downloader`/WorkManager); iOS gives BGAppRefresh ~30 s and never guarantees a schedule; Flutter web cannot use `dart:io` (no `IOHttpClientAdapter`, no pinning, no native Drift without the wasm setup) — if web is a target, guard those paths.

## 0. Mental model

```
 Widgets ──watch──▶ Riverpod/Bloc ──watch──▶ Repository.watchX()  ◀── single source of truth
    │                                              │
    │ user action                                  │  Stream<List<T>> from Drift .watch()
    ▼                                              │
 Repository.mutate() ── 1 txn ──▶ Drift: apply optimistic change + insert OutboxEntry
                                                   │
           ┌───────────────────────────────────────┴───────────────────────────┐
           ▼                                                                   ▼
   OutboxProcessor (ordered, single-runner)                         SyncEngine.pull(cursor)
     POST/PATCH/DELETE + Idempotency-Key                              GET /changes?cursor=…
     2xx → mark done, apply server echo                               upserts + tombstones
     409 → ConflictResolver                                           in ONE txn with new cursor
     4xx → failed (visible), 5xx/offline → backoff                     410 cursor expired → full resync
           │                                                                   │
           └──────────────────────────────┬────────────────────────────────────┘
                                          ▼
   ApiClient (one dio per account/env):  [Headers] → [Auth (QueuedInterceptor, single-flight)]
                                        → [Retry (backoff+jitter, idempotent only)] → [Cache] → [Log]
                                          │
   Triggers: app start, resume, connectivity regained (hint), pull-to-refresh,
             push "data changed", WebSocket event, workmanager / background_fetch
```

Rules that follow from the model:

1. **The UI never renders a network response directly** for data that must work offline. Network → DB → stream → UI. Screens that are online-only (search, checkout) may read the network, but say so explicitly.
2. **A write is durable the moment it hits the outbox**, not when the server answers. Optimistic change and outbox insert happen in the same transaction.
3. **One client, one interceptor chain, one place for errors.** Feature code never constructs `Dio()` or catches `DioException`.
4. **Connectivity is a hint.** The only truth is whether a request succeeded.
5. **Every sync step is idempotent and resumable.** Kill the app at any line and the next run converges.

## 1. First, establish context

Find out (from the codebase or by asking):

- **API style**: REST (JSON? OpenAPI-generated client via `retrofit`/`openapi_generator`?), GraphQL, gRPC, Firebase/Supabase SDKs (which bring their own offline layer — don't build a second one on top)?
- **Freshness needs** per screen: feed (minutes OK), chat (seconds, realtime), balance/stock/price (must be fresh or visibly stale), settings (rarely changes).
- **Offline scope**: none (graceful errors only), **read-only offline** (cache last data), or **read-write offline** (outbox). Which entities? Users rarely need *everything* offline.
- **Conflict tolerance**: can two devices edit the same record? Single-user notes vs shared lists vs collaborative docs. Which fields are server-authoritative (status, totals, permissions)?
- **Volume**: rows per user (100 vs 100k), payload sizes, attachments/media, how far back history goes. Decides pagination, delta sync, and DB indexes.
- **Backend support**: ETags? `updated_since`/cursor change feed? Tombstones for deletes? Idempotency keys? `Retry-After`? Server versions/`row_version` per record? If not, what can be added — many client problems are only solvable server-side.
- **Environments/regions**: dev/staging/prod base URLs, per-region hosts, per-account hosts (**flutter-multi-account**).
- **Auth**: token type, refresh flow, refresh token rotation, what 401 vs 403 mean.
- **Realtime**: WebSocket, SSE, Firebase, push-only? Is there an event sequence number for catch-up?
- **Background expectations**: "sync overnight", "upload photos even if app closed", "fresh data when opened from push".
- **State management** (Riverpod / Bloc / Provider / signals), **DB** (Drift, sqflite, Isar fork, Hive, ObjectBox), targets (iOS, Android, web, desktop).

## 2. HTTP client architecture

- [ ] **One configured `Dio` per (environment, account)**, built in one factory (`ApiClientFactory`). No `Dio()` in feature code, no `http.get` side paths that skip interceptors.
- [ ] `BaseOptions`: `baseUrl` from a typed `AppEnvironment` (flavor / `--dart-define`), `connectTimeout` (~10 s), `receiveTimeout` (~20–30 s; longer per-request for big responses), `sendTimeout` only for bodies, `responseType: json`, `validateStatus` default (non-2xx throws) so errors go through one path.
- [ ] Base URL **per region/account** if the backend is sharded; never mutate `dio.options.baseUrl` of a shared client at runtime (in-flight requests pick it up mid-way).
- [ ] **Interceptor order is behaviour** (request order = list order, response/error reversed):
  1. `HeadersInterceptor` — `User-Agent`/`X-App-Version`/`X-Platform`/`Accept-Language`/`X-Request-Id` (fresh UUID per attempt, or per logical request — decide).
  2. `AuthInterceptor` (`QueuedInterceptor`) — attach token, single-flight refresh on 401, retry once.
  3. `RetryInterceptor` — transient failures only, idempotent requests only.
  4. `DioCacheInterceptor` — conditional requests / cache hits (place after auth so cached auth'd responses are per client; the cache key must include the account if stores are shared).
  5. `LoggingInterceptor` (debug only, redacting `Authorization`, cookies, bodies with PII) — last so it sees what's actually sent.
- [ ] JSON decoding of large payloads off the UI isolate: `dio`'s default transformer uses `compute` above a threshold in some versions — verify; or decode with `Isolate.run` in the repository. Use generated models (`json_serializable` / `freezed`) and treat decoding errors as typed errors (§4), not crashes.
- [ ] HTTP/2 or native stacks (`native_dio_adapter` / `cupertino_http` / `cronet_http`) if you need platform proxy, VPN, or HTTP/3 behaviour — verify compatibility with pinning and caching.
- [ ] Platform config: Android `INTERNET` permission in main manifest (release!), `network_security_config` (no cleartext in prod); iOS ATS exceptions only for dev hosts; macOS `com.apple.security.network.client` entitlement.

## 3. Auth

- [ ] Attach `Authorization: Bearer <access>` from a `TokenHolder` **owned by the client's account**, not a global "current user" (**flutter-multi-account**).
- [ ] **Single-flight refresh**: N parallel 401s → exactly one refresh call; others await the same `Future`. `QueuedInterceptor` serialises `onError`, but you still need a shared in-flight future and a "token changed since this request was sent?" check (compare the token on the failed request with the current one; if different, just retry).
- [ ] Refresh call uses a **separate bare `Dio`** (no auth interceptor) → no recursion, no 401 loop.
- [ ] Retry the original request **once** after refresh (`extra['authRetried'] = true`); second 401 → session expired.
- [ ] Distinguish refresh outcomes: `invalid_grant`/401 on refresh → sign out *this account* (needs re-auth UI); network error/5xx on refresh → keep tokens, surface offline/temporary error, **do not log out**.
- [ ] Proactive refresh shortly before `exp` (optional) to avoid a 401 round trip; still handle 401.
- [ ] 403 is **not** a refresh trigger (permissions/portal revoked).
- [ ] Outbox and background sync use the same client → same refresh path; background isolates must load tokens from secure storage themselves (§13).
- [ ] Tokens in `flutter_secure_storage`, never in `shared_preferences`, Drift, logs, or crash keys.

## 4. Error model

- [ ] One `sealed class AppError` produced in one place (`ErrorMapper.from(Object e)`):
  - `NetworkError` (no route, DNS, connection refused/reset, `DioExceptionType.connectionError`, `SocketException`)
  - `TimeoutError` (connection/send/receive timeouts)
  - `HttpError(status, code?, message?, retryAfter?)` with sub-cases worth matching: `Unauthorized` (401 after refresh), `Forbidden` (403), `NotFound` (404), `Conflict` (409/412), `Gone` (410), `Validation` (400/422 with field errors), `RateLimited` (429), `ServerError` (5xx)
  - `DecodingError` (response shape mismatch — report to **flutter-crash-analytics** as non-fatal, include endpoint not body)
  - `CancelledError` (never shown to the user, never reported)
  - `CertificateError` (pinning/TLS failures — distinct message, report)
  - `UnknownError(cause)`
- [ ] Parse the server's error envelope (`{ "error": { "code": "card_declined", "message": "...", "fields": {...} } }`) into `code` — **switch on server codes, not on English messages**.
- [ ] **User-facing mapping** lives in the UI layer (`AppError → localized string + action`): Network → "You're offline. Changes will sync when you're back." + Retry; Timeout → "Taking longer than usual" + Retry; 5xx → "Something went wrong on our side"; Validation → inline field errors; 401 → re-auth flow; 429 → "Too many attempts, try again in N s".
- [ ] Repositories return `Result<T, AppError>` or throw `AppError` — pick one convention. `AsyncValue.error` / Bloc error states hold `AppError`, not `DioException`.
- [ ] Don't show an error banner when cached data is displayed and only the refresh failed — show "stale" indicator + last updated time instead.

## 5. Retries

- [ ] Retry only **transient** failures: connection errors, timeouts, 408, 429, 502, 503, 504 (500 only if your backend says it's safe). Never retry 4xx validation/auth/permission errors.
- [ ] Retry automatically only **safe/idempotent** requests: GET, HEAD, PUT, DELETE, and POST/PATCH **carrying an `Idempotency-Key`** the server honours. Mark via `Options(extra: {'idempotent': true})`.
- [ ] **Exponential backoff with full jitter**: `delay = random(0, min(cap, base * 2^attempt))`, e.g. base 500 ms, cap 30 s, max 3 attempts in the foreground. The outbox has its own longer schedule (§10).
- [ ] Honour **`Retry-After`** (seconds or HTTP date) on 429/503; it overrides the computed delay (cap it so the UI doesn't hang).
- [ ] **Idempotency key is generated once per logical operation** (stored in the outbox row), reused across all retries, app restarts, and background runs. Generating it per attempt defeats the point.
- [ ] **Circuit breaker** per host (optional, useful with many background calls): after N consecutive failures open for T seconds, fail fast with `NetworkError`, half-open with one probe. Prevents battery drain and server stampede when the backend is down.
- [ ] Respect `CancelToken`: don't retry or sleep after cancellation.
- [ ] Don't stack retries: dio retry × outbox retry × user retry button. Foreground retry is short; the outbox owns long-term retry.

## 6. Cancellation

- [ ] **Lifecycle-scoped `CancelToken`**: one per screen/provider; `ref.onDispose(token.cancel)` (Riverpod) or `close()` in a Bloc/State `dispose`. Session teardown cancels all tokens of that account (**flutter-multi-account**).
- [ ] **Stale search cancel**: debounce input (300 ms) and cancel the previous token before starting the next query; also guard by query id so a late response can't overwrite newer results.
- [ ] **In-flight GET dedupe**: identical GETs (same URL + query + account) share one `Future` while in flight (`Map<String, Future<Response>>`). Make sure one subscriber's cancel doesn't cancel the shared request for the others (ref-count or don't pass the per-caller token).
- [ ] Riverpod: `autoDispose` providers + cancel token in `onDispose`; Riverpod 3 retries failed providers automatically — make sure that composes with your retry policy (disable or tune `retry`).
- [ ] Never cancel outbox writes because a screen closed — they're not owned by the screen.
- [ ] `CancelledError` is swallowed silently (no snackbar, no crash report).

## 7. Caching

- [ ] Decide per endpoint: **no cache** (mutations, sensitive), **HTTP cache** (`dio_cache_interceptor` — good for read-only online-mostly content), or **DB as cache** (anything shown offline or sync'd — §9). Don't do both for the same data.
- [ ] **ETag / If-None-Match** and `Last-Modified / If-Modified-Since`: `dio_cache_interceptor` with `CachePolicy.request` sends conditional requests and turns 304 into the cached body. For DB-backed resources, store the ETag in the row/collection metadata and send `If-None-Match` yourself; 304 → only bump `lastCheckedAt`.
- [ ] **Stale-while-revalidate**: render DB/cached data immediately, refresh in background, stream the update. In UI: show content + subtle "Updating…" rather than a spinner over empty.
- [ ] Use `hitCacheOnNetworkFailure` (v4 name — verify) so offline GETs return the last cached response for HTTP-cached endpoints.
- [ ] **Memory vs persistent**: `MemCacheStore` for session-scoped; `http_cache_drift_store` / file store for persistence. Persistent caches of authenticated content are **per account** and wiped on sign-out.
- [ ] Respect `Cache-Control: no-store`/`private` from the server; never cache responses carrying secrets.
- [ ] Set a size/age cap and a cleanup (`store.clean(staleOnly: true)`) on startup.
- [ ] Images: `cached_network_image` cache key must include account for auth'd images (**flutter-multi-account**).

## 8. Connectivity

- [ ] `connectivity_plus` says which **interfaces** exist (wifi/mobile/none), not whether the internet is reachable (captive portals, dead Wi-Fi, VPN, server down). Treat it as a **hint**.
- [ ] Optional reachability probe (`internet_connection_checker_plus` or a HEAD to your own `/health`) — but still never block a request on it; just try and map the error.
- [ ] Derive an `OnlineStatus` stream: `connectivity hint` ⊕ `last request result` (a recent success → online; N consecutive network errors → offline). Debounce flaps (~2 s).
- [ ] **Offline banner** driven by that stream, non-blocking, with pending outbox count ("3 changes waiting to sync"). Accessible (`Semantics(liveRegion: true)`), doesn't cover content or shift layout violently.
- [ ] On reconnect hint → kick outbox + pull sync (debounced, single-flight). On resume (`AppLifecycleListener.onResume`) → same.
- [ ] v6+: `onConnectivityChanged` emits `List<ConnectivityResult>`; `[ConnectivityResult.none]` means none. Code comparing to a single enum silently breaks after upgrading.
- [ ] Metered/low-data awareness: avoid large prefetch/uploads on `mobile` if the user opted for Wi-Fi-only (setting), and on iOS Low Data Mode (needs native check).

## 9. Offline-first architecture

- [ ] **Drift is the source of truth**: repositories expose `Stream<T>` from `select(...).watch()` / `watchSingleOrNull()`; network calls only write into Drift. UI rebuilds from the stream — no manual "refresh list after save".
- [ ] Repository API shape: `Stream<List<Todo>> watchTodos()`, `Future<void> refresh()`, `Future<void> create/update/delete(...)` (writes go to outbox). Never `Future<List<Todo>> getTodos()` that hits the network for an offline screen.
- [ ] Schema per synced table: `id` (client-generated UUID v4/v7 so offline creates have stable ids), server fields, `updatedAt` (server), `version`/`etag`, `syncState` (`synced`, `pending`, `failed`, `conflict`), `deletedAt` (local soft-delete until server confirms), plus a `sync_meta` table for cursors.
- [ ] Indexes for every `where`/`orderBy` used by watched queries; watched queries over 10k rows need pagination (`limit`/keyset) — see **flutter-list-ux**.
- [ ] Run Drift in a background isolate (`driftDatabase(name:)` from `drift_flutter` or `NativeDatabase.createInBackground`) so big upserts don't jank.
- [ ] Batched writes (`batch((b) => b.insertAllOnConflictUpdate(...))`) inside a `transaction`; streams emit once per transaction.
- [ ] DB per account (**flutter-multi-account**); close it on session dispose.
- [ ] Other engines: sqflite (no streams — you build notifications), Isar (check fork maintenance), Hive/hive_ce (KV, no relational queries — fine for small caches, weak for sync), ObjectBox (has its own sync product). Firebase/Supabase/PowerSync/ElectricSQL bring their own sync — evaluate before hand-rolling.
- [ ] Riverpod: `StreamProvider.autoDispose` wraps `watchX()`; trigger `refresh()` in the provider or on screen enter — not in `build` of the widget.

## 10. Outbox for writes

- [ ] Table `outbox(id autoinc, opId uuid, entity, entityId, op (create|update|delete|custom), payload json, idempotencyKey, createdAt, attempts, nextAttemptAt, state (pending|inFlight|failed|done), lastError, dependsOn?)`.
- [ ] **Optimistic change + outbox insert in one Drift transaction.** UI sees the change immediately from the stream; row marked `syncState=pending` (show a subtle clock/sync icon).
- [ ] **Coalescing**: multiple updates to the same entity before sending can be merged (latest patch wins per field); create + delete before sending → drop both. Never coalesce across a delete→create of the same id or custom commands with side effects (payments).
- [ ] **Ordered replay**: process FIFO per entity (or globally if entities depend on each other — child created before parent). A failed head blocks its dependants, not unrelated entities.
- [ ] **Single runner**: a mutex/flag so foreground triggers, connectivity, and background tasks don't send the same entry twice. Mark `inFlight` before sending; on startup reset stale `inFlight` → `pending` (the idempotency key makes resend safe).
- [ ] Outcomes per entry:
  - 2xx → apply server echo (server id/version/fields) to the row, `syncState=synced`, delete outbox entry — same transaction.
  - Network/timeout/5xx/429 → `attempts++`, `nextAttemptAt = now + backoff(attempts)` (longer: 5 s, 30 s, 2 min… cap 1 h), stop the batch (likely still offline).
  - 401 after refresh failed → pause the outbox for that account until re-auth; **don't drop entries**.
  - 409/412 → conflict resolver (§12).
  - 400/422/404/403 (permanent) → `state=failed`, `syncState=failed`, keep the optimistic data visibly marked "Couldn't save" with **Retry / Discard** (discard = revert to last server copy).
- [ ] Per-item failure UI and a global "N changes not synced" entry point (settings or banner) so failed items are never silently stuck.
- [ ] **Never lose an outbox entry** on sign-out without asking: "You have 3 unsynced changes. Sign out anyway?" (and do it per account).
- [ ] Attachments: write the file to app storage first, outbox entry references the local path; upload step (§15) before the metadata mutation, or a two-phase entry.
- [ ] Payment/IAP receipt submission goes through the outbox too (**flutter-in-app-purchase**).

## 11. Sync (pull)

- [ ] Prefer a server **change feed**: `GET /sync?cursor=<opaque>&limit=500` → `{ changes: [...], deletes: [ids], nextCursor, hasMore }`. Fallback `updated_since=<server timestamp>` — use the **server's** timestamp from the previous response, never the device clock (§19 clock skew), and overlap by a few seconds with idempotent upsert.
- [ ] Apply each page **and** store its `nextCursor` in one transaction → crash-safe resume.
- [ ] Loop while `hasMore`; yield between pages; stop on cancellation.
- [ ] **Tombstones**: deletes come as explicit records (`deletes`, or rows with `deletedAt`). Without tombstones, a delta sync can never learn about deletions — require them from the backend, or do periodic full reconciliation (fetch ids, delete local rows not present).
- [ ] **Don't clobber pending local changes**: when an incoming row has a pending outbox entry, keep the local optimistic version (or merge, §12) and let the outbox resolve it.
- [ ] **Full resync fallback**: cursor expired (`410 Gone` or `cursor_invalid`), schema migration, or "reset" flag → fetch everything into staging (or mark-and-sweep: tag all rows with a sync generation, delete rows not touched) without blanking the UI mid-way.
- [ ] Push-first ordering: run outbox **before** pull so the pull returns server state that already includes local writes.
- [ ] Partial sync by scope (only the last 90 days, only starred) — store scope in `sync_meta`; widening scope triggers a backfill.
- [ ] Sync triggers are debounced and single-flight per account (one `Future` reused).

## 12. Conflict resolution

- [ ] Pick per entity, document it:
  - **Last-write-wins** (by server receive time): simplest, fine for single-user data; silently loses edits on shared data.
  - **Optimistic concurrency**: send `If-Match: <etag>` or `version`; server returns 409/412 on mismatch with the current server copy.
  - **Field-level merge**: three-way merge `base` (last synced copy, store it) vs `local` vs `server`; non-overlapping fields merge automatically; overlapping fields → rule or prompt.
  - **User prompt**: for documents/notes where losing text is unacceptable — show "Edited on another device" with Keep mine / Keep theirs / Copy.
  - **CRDT/OT**: collaborative editing only; use a library, not a hand-rolled one.
- [ ] **Server-authoritative fields** (status, totals, balances, permissions, `createdAt`, counters) are never sent from the client and always overwritten from the server.
- [ ] Counters/sets: send operations (`increment by 1`, `add tag`), not final values.
- [ ] Deletes vs edits: define whether an edit resurrects a remotely-deleted item (usually no: show "This item was deleted").
- [ ] Resolution result goes back through the outbox (new entry with the merged payload and the server's version).

## 13. Background sync

- [ ] **Android**: `workmanager` → WorkManager. `registerPeriodicTask` (min 15 min) with `Constraints(networkType: NetworkType.connected, requiresBatteryNotLow: true)`; `registerOneOffTask` for "sync soon" after an offline write (with `existingWorkPolicy: ExistingWorkPolicy.keep`/`replace`). Backoff policy exponential.
- [ ] **iOS**: BGTaskScheduler via `workmanager` (identifiers in `Info.plist` `BGTaskSchedulerPermittedIdentifiers`, `UIBackgroundModes: fetch, processing`, AppDelegate registration per plugin docs) or `background_fetch` (wraps BGAppRefreshTask; `minimumFetchInterval` is a wish; must call `BackgroundFetch.finish(taskId)`; `onTimeout` handler). iOS decides when — often never for rarely-used apps, never after the user force-quits. Don't promise "sync every 15 minutes".
- [ ] Background entry point is a **separate isolate**: `@pragma('vm:entry-point')` top-level dispatcher, `WidgetsFlutterBinding.ensureInitialized()` / `DartPluginRegistrant.ensureInitialized()`, then build the minimal graph (env, secure storage, account registry, DB, dio) — no `ProviderScope` from the UI isolate, no UI.
- [ ] **DB concurrency**: UI isolate and background isolate opening the same SQLite file → enable WAL, short transactions, `busy_timeout`; or use Drift's `DriftIsolate`/shared-isolate setup. Single-runner lock must work across isolates (a DB row lock / lease with expiry, not a Dart `bool`).
- [ ] Iterate **all signed-in accounts** (each with its own client/DB) or only the active one — product decision; always use the account's own credentials (**flutter-multi-account**).
- [ ] Time-box work (iOS ~30 s): outbox first, then a bounded pull; return success/failure honestly so the OS reschedules.
- [ ] **Push-triggered sync** (**flutter-push-notifications**): data/silent push (`content-available: 1` on iOS, FCM data message on Android high priority sparingly) → `FirebaseMessaging.onBackgroundMessage` handler runs a bounded sync. iOS throttles silent pushes; treat them as hints. Payload carries `account_id` and maybe a `changed: ["todos"]` scope, never the data itself if sensitive.
- [ ] Opening the app from a push/deep link for content not yet synced → fetch that entity directly, write to DB, then show (**flutter-deep-links**).
- [ ] Uploads that must finish even if app is killed → `background_downloader` (URLSession background / WorkManager) (§15).

## 14. Realtime

- [ ] `web_socket_channel` (`WebSocketChannel.connect(uri)`; `await channel.ready` to detect connect failure; `IOWebSocketChannel.connect(uri, headers:, pingInterval:)` on IO platforms for auth headers and keepalive). SSE: a streamed dio GET (`ResponseType.stream`) with line parsing, or a package.
- [ ] Auth: token in header (IO) or short-lived ticket in the query (`/ws?ticket=`) obtained via REST — browsers can't set headers. Reconnect re-fetches the ticket. Don't log URLs containing tickets.
- [ ] **Reconnect with backoff + jitter** (1 s → 30 s cap), reset after a stable connection; stop when backgrounded (iOS kills the socket anyway), reconnect on resume and on connectivity hint.
- [ ] **Catch-up**: every event has a sequence/cursor; on reconnect send `lastSeq` or run a REST delta pull (§11) before resuming live events. Without catch-up, every disconnect silently loses data.
- [ ] Events are **hints or deltas written to Drift**, not direct UI updates; dedupe by event id / version (events can arrive before the REST response of your own write — compare versions, ignore older).
- [ ] Heartbeat/ping and a "no message for N s → reconnect" watchdog (half-open TCP after Wi-Fi switch).
- [ ] One socket per account (or per portal), torn down with the session.
- [ ] Typing indicators/presence: ephemeral, not persisted.

## 15. Large transfers

- [ ] Uploads/downloads > a few MB or that must survive app exit → `background_downloader` (iOS background `URLSession`, Android WorkManager/foreground service) with progress, pause/resume, and notification. In-process dio uploads die when iOS suspends the app.
- [ ] **Resumable/multipart**: chunked upload protocol (tus, S3 multipart with presigned part URLs, GCS resumable). Persist upload id + completed parts in Drift so a restart resumes rather than restarts.
- [ ] Presigned URLs expire → refresh before each part/resume; don't put the API bearer token on S3 requests (separate bare client).
- [ ] dio: `onSendProgress`/`onReceiveProgress`, `download(url, savePath, deleteOnError: true)`, `Range` header for resume when the server supports it; stream to file, never into memory.
- [ ] Compress/resize images before upload (`flutter_image_compress`) off the UI isolate; strip EXIF location if not needed (privacy).
- [ ] Respect Wi-Fi-only preference and low battery; show progress UI (**flutter-list-ux** for per-row progress).
- [ ] Downloaded files go in app support dir (per account), excluded from iCloud backup if re-downloadable.

## 16. Security

- [ ] TLS only (ATS / `network_security_config` cleartext off in release). No `badCertificateCallback = (_, __, ___) => true` anywhere outside a debug-only flavor — grep for it.
- [ ] **Certificate/public-key pinning trade-offs**: protects against compromised CAs and some MITM, but a botched rotation bricks every installed app until update. If you pin: pin the SPKI hash of an intermediate/your own key with **at least one backup pin**, ship a remote kill switch or expiry date, monitor failures. In dio: `IOHttpClientAdapter(createHttpClient: ..., validateCertificate: (cert, host, port) => ...)` (verify API in your dio version); `http_certificate_pinning` package alternative. Not available on web. Many consumer apps reasonably choose **not** to pin.
- [ ] **No secrets in the bundle**: API keys in `--dart-define` or assets are extractable. Only ship keys meant to be public (restricted by bundle id / SHA); anything secret lives on your server. `--obfuscate` is not protection.
- [ ] **Local DB encryption** for sensitive data: SQLCipher via `sqlcipher_flutter_libs` (or `sqlite3mc`), key generated per account and stored in `flutter_secure_storage`; `PRAGMA key` in `setup`. Measure performance; migrating an existing plaintext DB needs `sqlcipher_export`.
- [ ] Logs/breadcrumbs: redact tokens, cookies, emails, query params with ids (**flutter-crash-analytics**). `LogInterceptor` only in debug.
- [ ] Outbox payloads may contain sensitive data at rest → same encryption decision as the DB.
- [ ] Wipe per-account DB, HTTP cache, files, outbox on sign-out (after the "unsynced changes" prompt).

## 17. Migrations

- [ ] Drift `schemaVersion` + `MigrationStrategy(onUpgrade: stepByStep(...))` generated via `dart run drift_dev make-migrations` / `schema dump` and **tested** with `drift_dev` schema verifier for every version pair you ship.
- [ ] Never migrate by "drop all and resync" if the outbox may hold unsynced writes. If a destructive reset is unavoidable, flush the outbox first, or migrate outbox rows explicitly.
- [ ] **Outbox payload versioning**: store `payloadVersion`; when the app updates, old entries must still be sendable (keep old serializers or upcast on read). Users can have entries created by a version from months ago.
- [ ] After a schema change that affects synced data, bump a `syncGeneration` → full resync (§11) after migration.
- [ ] **API version compatibility**: send `X-App-Version`/`Accept: application/vnd.x.v2+json`; server can answer `426 Upgrade Required` (or a custom code) → force-update screen. Decoders tolerate unknown fields and unknown enum values (`@JsonKey(unknownEnumValue: ...)`) so the server can add without breaking old clients.
- [ ] Keep the oldest supported app version in mind for every server change; old installs keep syncing for years.

## 18. GraphQL (short)

- [ ] Clients: `graphql_flutter` (`GraphQLClient(link: AuthLink(...).concat(HttpLink(...)), cache: GraphQLCache(store: HiveStore()))`) or `ferry` (typed codegen via `ferry_generator`/`gql_build`, normalized cache with Hive store, streams of responses). Ferry fits the "watch a stream" model better; graphql_flutter is simpler to adopt. Both rough equivalents of Apollo/urql.
- [ ] **Normalized cache**: entities keyed by `__typename:id` — always request `id` (and `__typename`) on every object so updates from one mutation update all queries showing it. Set `dataIdFromObject`/type policies for types with other keys.
- [ ] Fetch policies: `cacheAndNetwork` for stale-while-revalidate screens, `cacheFirst` for static data, `networkOnly` for checkout.
- [ ] **Optimistic responses**: graphql_flutter `MutationOptions(optimisticResult: {...})`, ferry `OperationRequest(optimisticResponse: ...)` — the optimistic layer is rolled back on error; make the shape exactly match the real response (including `__typename`) or the cache rejects it.
- [ ] Lists after create/delete need explicit cache updates (`update:` callback / ferry `UpdateCacheHandler`) — normalization doesn't add/remove list members for you.
- [ ] The GraphQL cache is not an outbox: offline mutations are lost on app kill unless you persist them — for read-write offline, still use Drift + outbox (§10) and treat GraphQL as transport, or use ferry's offline mutation pattern with care.
- [ ] Errors: GraphQL returns 200 with `errors[]` — map `extensions.code` into the same `AppError` model (§4). Auth via `AuthLink` with the same single-flight refresh. Subscriptions over `WebSocketLink` follow §14 reconnect + catch-up rules.
- [ ] Persisted queries (APQ) for bandwidth; GET for cacheable queries.

## 19. Testing plan

Unit-test interceptors with `http_mock_adapter` (`DioAdapter`) or a fake `HttpClientAdapter`; repositories, outbox, and sync engine with in-memory Drift (`NativeDatabase.memory()`) and a fake API; widget-test offline banner, stale indicators, and per-item failure states; integration-test against a **local mock server** (`shelf`-based, or WireMock/Prism from your OpenAPI spec) that can inject latency, errors, 409s, and `Retry-After`.

Tools and techniques:
- [ ] **Network throttling**: Android emulator network speed/latency settings, iOS Network Link Conditioner (device: Developer settings), Charles/Proxyman throttling and breakpoints (edit responses mid-flight).
- [ ] **Airplane-mode matrix** on real devices (emulators lie about connectivity transitions).
- [ ] **Clock skew**: set device time ±1 day / wrong timezone; sync must still converge (server timestamps/cursors only), token expiry checks tolerate skew (use server `expires_in`, add leeway).
- [ ] Fake clock (`package:clock` / `fake_async`) for backoff, jitter, debounce, and circuit breaker tests.
- [ ] Kill-the-app tests: force-quit during outbox send and during a sync page; relaunch converges.
- [ ] Background: Android `adb shell cmd jobscheduler run -f <pkg> <jobId>` / WorkManager test helpers; iOS Xcode `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"id"]` in the debugger.
- [ ] Contract tests: decode recorded real responses (fixtures) in unit tests to catch shape drift.

### Scenario matrix

| Scenario | Expect |
|---|---|
| Cold start offline with data synced before | Content from DB instantly; offline banner; no error dialog |
| Cold start offline, never synced | Empty state explaining offline + Retry; no infinite spinner |
| Create item offline, relaunch app offline | Item still shown with "pending"; outbox entry intact |
| Create offline → go online | Sent once; server id/version applied; pending marker gone; no duplicate row |
| Same write retried after timeout where server actually succeeded | Idempotency key → server returns original result; one record server-side |
| 3 edits offline to same item | Coalesced or sent in order; final server state = last edit |
| Create then delete offline | Nothing sent (or create+delete in order); no ghost |
| Outbox entry gets 422 | Marked "Couldn't save" with Retry/Discard; later entries for other items still sync |
| Outbox head gets 503 with Retry-After: 120 | Waits ≥120 s; no hammering; resumes |
| 5 parallel requests get 401 | One refresh; all retried once; no logout |
| Refresh endpoint unreachable | Tokens kept; offline error; no logout |
| Refresh returns invalid_grant | This account requires re-auth; outbox paused, not dropped |
| Edit on device A and B (same field) | Configured policy applied (409 → merge/prompt); no silent loss where policy says prompt |
| Item deleted on server, edited locally offline | Defined outcome ("deleted" notice); no resurrection unless policy says so |
| Delta sync with server deletions | Tombstones remove local rows |
| Cursor expired (410) | Full resync without blank UI; pending local writes preserved |
| Kill app mid sync page | Next run resumes from last committed cursor; no duplicates/gaps |
| Device clock 1 day ahead/behind | Sync converges; tokens not considered expired/valid wrongly |
| Captive portal Wi-Fi (connected, no internet) | Requests fail → offline status; banner; no crash on HTML response (DecodingError) |
| Flapping connectivity | Debounced banner; single sync run; no duplicate outbox sends |
| Search typed fast "abc" | Only "abc" results shown; earlier requests cancelled |
| Leave screen during slow GET | Request cancelled; no setState-after-dispose; no error snackbar |
| WebSocket drops for 2 min | Reconnect with backoff; missed events caught up; no duplicates |
| App backgrounded with socket open | Socket closed/suspended; reconnect + catch-up on resume |
| Silent push "todos changed" while app killed (Android) | Background handler syncs into the right account's DB |
| iOS background refresh fires | Outbox + bounded pull finish < 30 s; `finish` called |
| 500 MB video upload, app killed | Upload continues/resumes in background; progress correct on relaunch |
| Presigned URL expired mid-resume | New URL fetched; upload resumes from last part |
| App update with Drift schema change and pending outbox | Migration succeeds; old outbox entries still sent |
| Server returns 426 / min version | Force-update screen; outbox kept for after update |
| Pinning: cert rotated to backup key | Still works; rotated to unknown key → CertificateError reported, kill switch path works |
| Sign out with unsynced changes | Prompt; on confirm wipe that account's DB/cache/outbox only |
| Large sync (50k rows) | UI stays responsive (background isolate, batched txns) |

## 20. Common bugs to look for in review

- `Dio()` created in widgets/repositories → no auth, no retry, no timeouts, no error mapping.
- Interceptor reading a global "current account" at retry time → wrong token after switch (**flutter-multi-account**).
- Refresh not single-flight / refresh call going through the auth interceptor → refresh storm, 401 loop, random logouts with rotating refresh tokens.
- Logging out on any refresh failure, including network errors.
- Retrying POSTs without idempotency keys → duplicate orders/messages/payments.
- Idempotency key generated per attempt instead of per operation.
- Retrying 4xx; ignoring `Retry-After`; no jitter (thundering herd after outage).
- Nested retries (dio retry × outbox × Riverpod 3 auto-retry) → 27 requests per failure.
- `connectivity_plus` used as a gate ("if none, don't send") → stuck on captive portals and false "none" on some devices; v6 list-vs-enum upgrade bug.
- UI renders network responses and *also* listens to DB → flicker, double sources of truth.
- "Refresh after save" calls instead of DB streams → stale lists elsewhere in the app.
- Optimistic update and outbox insert in separate transactions → crash leaves UI change without a pending write (or vice versa).
- Outbox entries deleted on 5xx or on sign-out without asking → silent data loss.
- Outbox processed concurrently by foreground + background → duplicate sends.
- `updated_since` computed from `DateTime.now()` on the device → missed changes under clock skew.
- No tombstones → deleted items live forever on other devices.
- Pull sync overwriting rows with pending local edits.
- Cursor saved before (or outside the transaction of) applying the page → gaps after a crash.
- Full resync implemented as `delete from table` then fetch → blank screen and lost pending edits.
- WebSocket reconnect without catch-up → missing messages after every network blip.
- Background isolate reusing UI-isolate singletons / Riverpod container → null/uninitialized crashes, or opening a second Drift instance without WAL → "database is locked".
- `background_fetch` handler not calling `finish(taskId)` → iOS stops scheduling the app.
- Big JSON decoded on the UI isolate → jank on sync.
- `badCertificateCallback` returning true left in release; pinning with no backup pin.
- API secret keys in `--dart-define`/`.env` assets.
- Drift migration untested; "fallback to destructive" style reset wiping the outbox.
- Decoders crashing on new enum values from the server.
- `CancelledError` shown as "Something went wrong" or reported to crash analytics.
- HTTP cache shared across accounts → A's responses shown to B.

## 21. Output format

**When reviewing** an existing app, respond with:

1. **Context summary** — API style, client stack and interceptor order, DB engine, state management, offline scope, background mechanisms, package versions.
2. **Data-flow map** — for each major entity: read path (network → DB → UI?), write path (direct call vs outbox), cache layer, sync trigger(s), conflict policy.
3. **Findings** grouped by area (Client, Auth, Errors, Retries, Cancellation, Caching, Connectivity, Offline/DB, Outbox, Sync, Conflicts, Background, Realtime, Transfers, Security, Migrations, GraphQL), each with **severity** (Critical = data loss, duplicate side effects, or credential/data leak; High; Medium; Low), file:line, symptom, fix (snippet when useful).
4. **Missing scenarios** from the §19 matrix, with tests to add.
5. **Backend asks** — things only the server can fix (idempotency keys, tombstones, cursors, `Retry-After`, versions).
6. **Version notes** — APIs to verify against the project's package versions.

**When implementing**, deliver:

1. `AppEnvironment` + `ApiClientFactory` with ordered interceptors (headers, auth single-flight, retry, cache, logging).
2. `AppError` sealed hierarchy + `ErrorMapper` + UI message mapping.
3. Drift schema with sync columns, `sync_meta`, `outbox`, and migrations.
4. Repository per entity exposing streams + outbox-backed mutations.
5. `OutboxProcessor`, `SyncEngine` (cursor + tombstones + full-resync fallback), conflict policy per entity.
6. `OnlineStatus` + offline banner; sync triggers (start, resume, reconnect, push, background registration).
7. Realtime client with reconnect + catch-up if applicable.
8. Tests for the §19 essentials and a list of assumptions/backend contract, flagging package APIs to verify.

## Appendix — Reference implementation (Dart 3, Flutter 3.24+, dio 5, Drift 2, connectivity_plus 6, workmanager)

Simplified but complete enough to adapt. Names and endpoints are illustrative; logging, DI wiring, and codegen boilerplate trimmed. Verify package APIs against your versions.

### A.1 Typed errors

```dart
import 'dart:io';
import 'package:dio/dio.dart';

sealed class AppError implements Exception { const AppError(); }
final class NetworkError extends AppError { const NetworkError(); }
final class TimeoutError extends AppError { const TimeoutError(); }
final class CancelledError extends AppError { const CancelledError(); }
final class CertificateError extends AppError { const CertificateError(); }
final class DecodingError extends AppError { const DecodingError(this.endpoint, this.cause); final String endpoint; final Object cause; }
final class HttpError extends AppError {
  const HttpError(this.status, {this.code, this.message, this.retryAfter, this.fields = const {}});
  final int status; final String? code; final String? message;
  final Duration? retryAfter; final Map<String, String> fields;
  bool get isTransient => status == 408 || status == 429 || status == 502 || status == 503 || status == 504;
  bool get isConflict => status == 409 || status == 412;
  bool get isPermanent => status >= 400 && status < 500 && !isTransient && !isConflict && status != 401;
}
final class UnknownError extends AppError { const UnknownError(this.cause); final Object cause; }

/// Retry-After as delta-seconds (int.tryParse) or HTTP-date (HttpDate.parse, clamped ≥ 0); else null.
Duration? parseRetryAfter(String? v) => throw UnimplementedError();

AppError mapError(Object e) {
  if (e is AppError) return e;
  if (e is DioException) {
    if (e.error is AppError) return e.error! as AppError;
    switch (e.type) {
      case DioExceptionType.cancel: return const CancelledError();
      case DioExceptionType.connectionTimeout || DioExceptionType.sendTimeout || DioExceptionType.receiveTimeout:
        return const TimeoutError();
      case DioExceptionType.badCertificate: return const CertificateError();
      case DioExceptionType.connectionError: return const NetworkError();
      case DioExceptionType.badResponse:
        final r = e.response!;
        final body = r.data is Map ? (r.data as Map)['error'] as Map? : null;
        return HttpError(r.statusCode ?? 0,
            code: body?['code'] as String?, message: body?['message'] as String?,
            retryAfter: parseRetryAfter(r.headers.value('retry-after')),
            fields: (body?['fields'] as Map?)?.cast<String, String>() ?? const {});
      case DioExceptionType.unknown:
        if (e.error is SocketException || e.error is HttpException) return const NetworkError();
        if (e.error is HandshakeException) return const CertificateError();
        return UnknownError(e.error ?? e);
    }
  }
  if (e is FormatException || e is TypeError) return DecodingError('?', e);
  return UnknownError(e);
}
```

### A.2 Configured client + interceptors

```dart
import 'dart:async';
import 'dart:math';
import 'package:dio/dio.dart';
import 'package:dio_cache_interceptor/dio_cache_interceptor.dart';
import 'package:uuid/uuid.dart';

typedef AppEnvironment = ({Uri apiBase, String appVersion});

/// refresh() is single-flight: concurrent callers get the same future. Throws HttpError(401) on invalid_grant.
abstract interface class TokenHolder { String? get accessToken; Future<String> refresh(); }

class HeadersInterceptor extends Interceptor {
  HeadersInterceptor(this.env);
  final AppEnvironment env;
  @override
  void onRequest(RequestOptions o, RequestInterceptorHandler h) => h.next(o
    ..headers.addAll({'X-App-Version': env.appVersion, 'X-Request-Id': const Uuid().v4(), 'Accept': 'application/json'}));
}

class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor(this.tokens, this.dio);
  final TokenHolder tokens;
  final Dio dio; // the same client, to replay the original request

  @override
  void onRequest(RequestOptions o, RequestInterceptorHandler h) {
    if (tokens.accessToken case final t? when o.extra['noAuth'] != true) o.headers['Authorization'] = 'Bearer $t';
    h.next(o);
  }

  @override
  Future<void> onError(DioException e, ErrorInterceptorHandler h) async {
    final o = e.requestOptions;
    if (e.response?.statusCode != 401 || o.extra['authRetried'] == true || o.extra['noAuth'] == true) {
      return h.next(e);
    }
    try {
      final sent = (o.headers['Authorization'] as String?)?.substring(7);
      final current = tokens.accessToken;
      // Another request already refreshed while this one was in flight → just replay.
      final fresh = (current != null && current != sent) ? current : await tokens.refresh();
      o.headers['Authorization'] = 'Bearer $fresh';
      o.extra['authRetried'] = true;
      h.resolve(await dio.fetch<dynamic>(o));
    } on DioException catch (re) {
      h.next(re);
    } catch (err) {
      h.next(DioException(requestOptions: o, error: mapError(err), response: e.response));
    }
  }
}

class RetryInterceptor extends Interceptor {
  RetryInterceptor(this.dio, {this.maxAttempts = 3, Random? rnd}) : _rnd = rnd ?? Random();
  final Dio dio; final int maxAttempts; final Random _rnd;
  static const _safe = {'GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS'};

  @override
  Future<void> onError(DioException e, ErrorInterceptorHandler h) async {
    final o = e.requestOptions;
    final attempt = (o.extra['attempt'] as int?) ?? 0;
    final idempotent = _safe.contains(o.method) || o.headers.containsKey('Idempotency-Key');
    final err = mapError(e);
    final transient = err is NetworkError || err is TimeoutError || (err is HttpError && err.isTransient);
    if (!idempotent || !transient || attempt + 1 >= maxAttempts || (o.cancelToken?.isCancelled ?? false)) {
      return h.next(e);
    }
    final cap = 30000, base = 500;
    final backoff = Duration(milliseconds: _rnd.nextInt(min(cap, base << attempt) + 1)); // full jitter
    var delay = err is HttpError && err.retryAfter != null ? err.retryAfter! : backoff;
    if (delay > const Duration(seconds: 30)) return h.next(e); // let the outbox handle long waits
    await Future<void>.delayed(delay);
    if (o.cancelToken?.isCancelled ?? false) return h.next(e);
    o.extra['attempt'] = attempt + 1;
    try { h.resolve(await dio.fetch<dynamic>(o)); } on DioException catch (re) { h.next(re); }
  }
}

class ApiClientFactory {
  static Dio create(AppEnvironment env, TokenHolder tokens, CacheStore cacheStore) {
    final dio = Dio(BaseOptions(baseUrl: env.apiBase.toString(),
        connectTimeout: const Duration(seconds: 10), receiveTimeout: const Duration(seconds: 30)));
    final cacheOptions = CacheOptions(store: cacheStore /* per account; wipe on sign-out */,
      policy: CachePolicy.request,       // conditional requests via ETag/Last-Modified
      hitCacheOnNetworkFailure: true /* v4 name — verify for your version */, maxStale: const Duration(days: 7));
    dio.interceptors.addAll([
      HeadersInterceptor(env),
      AuthInterceptor(tokens, dio),
      RetryInterceptor(dio),
      DioCacheInterceptor(options: cacheOptions),
      // debug only: LogInterceptor(requestHeader: false, responseBody: false) — never log Authorization
    ]);
    return dio;
  }
}

/// Thin typed wrapper: the only place feature code talks HTTP. Both methods rethrow via mapError.
/// - get<T>(path, decode, {query, cancel}): dedupes identical in-flight GETs in a Map<String, Future>
///   (removed whenComplete; a per-caller CancelToken only abandons its own wait via Future.any);
///   decode failures → DecodingError(path, e).
/// - send(method, path, {body, idempotencyKey, ifMatch}): sets Idempotency-Key / If-Match headers.
abstract interface class Api {
  Future<T> get<T>(String path, T Function(Object? json) decode, {Map<String, dynamic>? query, CancelToken? cancel});
  Future<Response<dynamic>> send(String method, String path, {Object? body, String? idempotencyKey, String? ifMatch});
}
```

### A.3 Drift schema (sync columns, outbox, cursors)

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';
part 'db.g.dart';

enum SyncState { synced, pending, failed, conflict }
enum OutboxState { pending, inFlight, failed }

class Todos extends Table {
  TextColumn get id => text()(); /* client UUID */ TextColumn get title => text()();
  BoolColumn get done => boolean().withDefault(const Constant(false))();
  IntColumn get version => integer().withDefault(const Constant(0))(); // server version
  DateTimeColumn get serverUpdatedAt => dateTime().nullable()();
  TextColumn get baseJson => text().nullable()();     // last synced copy for 3-way merge
  IntColumn get syncState => intEnum<SyncState>()();
  DateTimeColumn get deletedAt => dateTime().nullable()();
  @override Set<Column> get primaryKey => {id};
}

class Outbox extends Table {
  IntColumn get seq => integer().autoIncrement()();
  TextColumn get entity => text()(); TextColumn get entityId => text()();
  TextColumn get op => text()(); /* create | update | delete */ TextColumn get payload => text()(); // JSON
  IntColumn get payloadVersion => integer().withDefault(const Constant(1))();
  TextColumn get idempotencyKey => text()();
  IntColumn get attempts => integer().withDefault(const Constant(0))(); DateTimeColumn get nextAttemptAt => dateTime()();
  IntColumn get state => intEnum<OutboxState>()(); TextColumn get lastError => text().nullable()();
}

class SyncMeta extends Table {
  TextColumn get scope => text()();                    // e.g. 'todos'
  TextColumn get cursor => text().nullable()();
  IntColumn get generation => integer().withDefault(const Constant(0))();
  @override Set<Column> get primaryKey => {scope};
}

@DriftDatabase(tables: [Todos, Outbox, SyncMeta])
class AppDb extends _$AppDb {
  AppDb(String accountId) : super(driftDatabase(name: 'acct_$accountId'));
  AppDb.forTesting(super.e);

  @override int get schemaVersion => 2;

  @override
  MigrationStrategy get migration => MigrationStrategy(
        onCreate: (m) => m.createAll(),
        onUpgrade: (m, from, to) async { if (from < 2) await m.addColumn(todos, todos.baseJson); }, // never drop outbox
        beforeOpen: (_) async {
          await customStatement('PRAGMA journal_mode=WAL'); await customStatement('PRAGMA busy_timeout=5000');
          // Crash recovery: anything left inFlight is safe to resend (idempotency key).
          await (update(outbox)..where((o) => o.state.equalsValue(OutboxState.inFlight)))
              .write(const OutboxCompanion(state: Value(OutboxState.pending)));
        },
      );
}
```

### A.4 Repository (network → DB → UI)

```dart
import 'dart:convert';
import 'package:drift/drift.dart';
import 'package:uuid/uuid.dart';

class TodoRepository {
  TodoRepository(this.db, this.sync);
  final AppDb db; final SyncEngine sync;

  /// UI observes only this. Soft-deleted rows hidden immediately.
  Stream<List<Todo>> watchTodos() => (db.select(db.todos)
        ..where((t) => t.deletedAt.isNull())
        ..orderBy([(t) => OrderingTerm.desc(t.serverUpdatedAt)]))
      .watch();

  Stream<int> watchPendingCount() => db.outbox.count().watchSingle();
  Future<void> refresh() => sync.syncNow(); // outbox first, then pull

  Future<void> create(String title) async {
    final id = const Uuid().v7();
    await db.transaction(() async {
      await db.into(db.todos).insert(TodosCompanion.insert(id: id, title: title, syncState: SyncState.pending));
      await _enqueue('todo', id, 'create', {'id': id, 'title': title});
    });
    sync.kick();
  }

  Future<void> setDone(String id, bool done) async {
    await db.transaction(() async {
      await (db.update(db.todos)..where((t) => t.id.equals(id)))
          .write(TodosCompanion(done: Value(done), syncState: const Value(SyncState.pending)));
      // Coalesce with an unsent update for the same entity.
      final existing = await (db.select(db.outbox)
            ..where((o) => o.entityId.equals(id) & o.op.equals('update') & o.state.equalsValue(OutboxState.pending)))
          .getSingleOrNull();
      if (existing != null) {
        final merged = {...jsonDecode(existing.payload) as Map<String, dynamic>, 'done': done};
        await (db.update(db.outbox)..where((o) => o.seq.equals(existing.seq)))
            .write(OutboxCompanion(payload: Value(jsonEncode(merged))));
      } else {
        await _enqueue('todo', id, 'update', {'done': done});
      }
    });
    sync.kick();
  }

  // delete(id): same transaction shape — set deletedAt + pending, _enqueue(..., 'delete', {}), kick().

  Future<void> _enqueue(String entity, String id, String op, Map<String, dynamic> payload) =>
      db.into(db.outbox).insert(OutboxCompanion.insert(
            entity: entity, entityId: id, op: op, payload: jsonEncode(payload),
            idempotencyKey: const Uuid().v4(), // once per operation, reused on every retry
            nextAttemptAt: DateTime.now(), state: OutboxState.pending));
}
```

### A.5 Outbox processor + sync engine (cursor + tombstones)

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:math';
import 'package:drift/drift.dart';

class SyncEngine {
  SyncEngine(this.db, this.api);
  final AppDb db; final Api api;
  Future<void>? _running;
  Timer? _kickTimer;

  /// Debounced trigger from writes, reconnect, resume, push, socket events.
  void kick() { _kickTimer?.cancel();
    _kickTimer = Timer(const Duration(milliseconds: 300), () => unawaited(syncNow().catchError((_) {}))); }

  /// Single-flight per account. (Cross-isolate exclusion: add a DB lease row — omitted.)
  Future<void> syncNow({Duration budget = const Duration(seconds: 25)}) =>
      _running ??= _run(DateTime.now().add(budget)).whenComplete(() => _running = null);

  Future<void> _run(DateTime deadline) async {
    final pushedAll = await _drainOutbox(deadline);
    if (pushedAll || DateTime.now().isBefore(deadline)) await _pull('todos', deadline);
  }

  // ---- Outbox ----
  Future<bool> _drainOutbox(DateTime deadline) async {
    while (DateTime.now().isBefore(deadline)) {
      final e = await (db.select(db.outbox)
            ..where((o) => o.state.equalsValue(OutboxState.pending) &
                o.nextAttemptAt.isSmallerOrEqualValue(DateTime.now()))
            ..orderBy([(o) => OrderingTerm.asc(o.seq)])
            ..limit(1))
          .getSingleOrNull();
      if (e == null) return true;
      await _setState(e.seq, OutboxState.inFlight);
      try {
        final res = await _send(e);
        await db.transaction(() async {
          await _applyServerEcho(e, res.data);
          await (db.delete(db.outbox)..where((o) => o.seq.equals(e.seq))).go();
        });
      } on AppError catch (err) {
        if (err is NetworkError || err is TimeoutError || (err is HttpError && (err.isTransient || err.status >= 500))) {
          final wait = err is HttpError && err.retryAfter != null ? err.retryAfter! : _backoff(e.attempts + 1);
          await (db.update(db.outbox)..where((o) => o.seq.equals(e.seq))).write(OutboxCompanion(
              state: const Value(OutboxState.pending), attempts: Value(e.attempts + 1),
              nextAttemptAt: Value(DateTime.now().add(wait)), lastError: Value('$err')));
          return false; // probably offline/down: stop this run
        } else if (err is HttpError && err.isConflict) {
          await _resolveConflict(e, err);
        } else if (err is HttpError && err.status == 401) {
          await _setState(e.seq, OutboxState.pending); // paused until re-auth; never dropped
          return false;
        } else {
          // Permanent 4xx: in one transaction mark the outbox row `failed` (lastError) and the entity
          // SyncState.failed so the UI offers edit / discard.
          await _setState(e.seq, OutboxState.failed);
        }
      }
    }
    return false;
  }

  Future<dynamic> _send(OutboxData e) {
    final body = jsonDecode(e.payload);
    return switch (e.op) {
      'create' => api.send('POST', '/todos', body: body, idempotencyKey: e.idempotencyKey),
      'update' => api.send('PATCH', '/todos/${e.entityId}', body: body, idempotencyKey: e.idempotencyKey),
      'delete' => api.send('DELETE', '/todos/${e.entityId}', idempotencyKey: e.idempotencyKey),
      _ => throw StateError('unknown op ${e.op}'),
    };
  }

  Future<void> _applyServerEcho(OutboxData e, Object? json) async {
    if (e.op == 'delete') return (db.delete(db.todos)..where((t) => t.id.equals(e.entityId))).go().then((_) {});
    final m = json as Map<String, dynamic>;
    final stillPending = await (db.select(db.outbox)
          ..where((o) => o.entityId.equals(e.entityId) & o.seq.isBiggerThanValue(e.seq)))
        .get();
    await (db.update(db.todos)..where((t) => t.id.equals(e.entityId))).write(TodosCompanion(
        version: Value(m['version'] as int),
        serverUpdatedAt: Value(DateTime.parse(m['updated_at'] as String)),
        baseJson: Value(jsonEncode(m)),
        syncState: Value(stillPending.isEmpty ? SyncState.synced : SyncState.pending)));
  }

  /// Policy: server-authoritative fields always from server; user fields → LWW, or (as here) mark the
  /// outbox row failed + the entity `conflict`, then kick(): the pull fetches the server copy and the UI
  /// offers Keep mine (re-enqueue with the new version) / Keep theirs.
  Future<void> _resolveConflict(OutboxData e, HttpError err) async => throw UnimplementedError();

  // ---- Pull ----
  Future<void> _pull(String scope, DateTime deadline) async {
    var cursor = (await (db.select(db.syncMeta)..where((m) => m.scope.equals(scope))).getSingleOrNull())?.cursor;
    while (DateTime.now().isBefore(deadline)) {
      final Map<String, dynamic> page;
      try {
        page = await api.get('/sync/$scope', (j) => j! as Map<String, dynamic>,
            query: {if (cursor != null) 'cursor': cursor, 'limit': 500});
      } on HttpError catch (e) {
        if (e.status == 410) { await _fullResync(scope); return; }
        rethrow;
      }
      final (changes, deletes, next) = ((page['changes'] as List).cast<Map<String, dynamic>>(),
          (page['deletes'] as List).cast<String>(), page['next_cursor'] as String?);
      final pendingIds = (await db.select(db.outbox).get()).map((o) => o.entityId).toSet();

      await db.transaction(() async {
        await db.batch((b) {
          for (final c in changes) {
            if (pendingIds.contains(c['id'])) continue; // don't clobber local edits; outbox resolves
            b.insert(db.todos, TodosCompanion.insert(
                id: c['id'] as String, title: c['title'] as String, done: Value(c['done'] as bool),
                version: Value(c['version'] as int),
                serverUpdatedAt: Value(DateTime.parse(c['updated_at'] as String)),
                baseJson: Value(jsonEncode(c)), syncState: SyncState.synced),
                mode: InsertMode.insertOrReplace);
          }
          for (final id in deletes) {
            if (!pendingIds.contains(id)) b.deleteWhere(db.todos, (t) => t.id.equals(id)); // tombstone
          }
          b.insert(db.syncMeta, SyncMetaCompanion.insert(scope: scope, cursor: Value(next)),
              mode: InsertMode.insertOrReplace); // cursor committed with the page
        });
      });
      cursor = next;
      if (page['has_more'] != true) return;
    }
  }

  /// Mark-and-sweep without blanking the UI: bump `generation`, clear the cursor, re-pull
  /// (rows written get the new generation), then delete synced rows with an older generation.
  /// Needs a `generation` column on synced tables — omitted here for brevity.
  Future<void> _fullResync(String scope) async => throw UnimplementedError();

  Future<void> _setState(int seq, OutboxState s) =>
      (db.update(db.outbox)..where((o) => o.seq.equals(seq))).write(OutboxCompanion(state: Value(s)));

  static final _rnd = Random();
  static Duration _backoff(int attempt) { // equal jitter, 5 s · 2^(n-1), cap 1 h
    final ms = min(const Duration(hours: 1).inMilliseconds, 5000 * pow(2, attempt - 1).toInt());
    return Duration(milliseconds: ms ~/ 2 + _rnd.nextInt(ms ~/ 2 + 1));
  }
}
```

### A.6 Connectivity hint, triggers, and background registration

```dart
import 'dart:async';
import 'package:flutter/widgets.dart';
import 'package:workmanager/workmanager.dart';

// SyncTriggers(engine).start(): engine.kick() on app start, on AppLifecycleListener(onResume), and on
// Connectivity().onConnectivityChanged results without ConnectivityResult.none (a hint: just try);
// dispose() cancels the subscription and the listener.

const kSyncTask = 'com.example.app.sync'; // also in iOS Info.plist BGTaskSchedulerPermittedIdentifiers

@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, input) async {
    WidgetsFlutterBinding.ensureInitialized();
    try { // minimal graph in this isolate: env, secure tokens, registry, per-account DB + client
      for (final a in await BackgroundGraph.signedInAccounts()) {
        final g = await BackgroundGraph.forAccount(a); // own Dio + AppDb(a.id), WAL enabled
        try { await g.sync.syncNow(budget: const Duration(seconds: 20)); }
        finally { await g.db.close(); }
      }
      return true;
    } catch (_) {
      return false; // let the OS retry with backoff
    }
  });
}

Future<void> registerBackgroundSync() async {
  await Workmanager().initialize(callbackDispatcher);
  await Workmanager().registerPeriodicTask(kSyncTask, kSyncTask,
    frequency: const Duration(hours: 1), // Android min 15 min; iOS decides on its own
    constraints: Constraints(networkType: NetworkType.connected, requiresBatteryNotLow: true),
    existingWorkPolicy: ExistingPeriodicWorkPolicy.keep, // name differs across versions — verify
    backoffPolicy: BackoffPolicy.exponential,
  );
}

// BackgroundGraph: app-specific wiring for the background isolate — signedInAccounts() and
// forAccount(a) → ({AppDb db, SyncEngine sync}) built from storage, no UI singletons.
```

### A.7 Realtime with reconnect + catch-up

```dart
// RealtimeClient(ticketUri, engine) over web_socket_channel:
// connect(): fetch a short-lived ticket URI via REST, WebSocketChannel.connect + await ch.ready, reset attempt,
//   engine.kick() for catch-up (delta pull covers anything missed); each 'changed' event → engine.kick()
//   (event is a hint; DB stays the truth). A 45 s watchdog re-armed on every message closes half-open sockets.
// onDone/onError → reconnect after random(0, min(30 s, 1 s · 2^attempt)) + 250 ms unless stopped.
// stop() on background / session dispose: cancel timers, close sink.
```
