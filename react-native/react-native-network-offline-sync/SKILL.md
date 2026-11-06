---
name: react-native-network-offline-sync
description: Use when implementing, reviewing, testing, or debugging networking, caching, offline-first data, and sync in React Native and Expo apps — a single configured fetch / axios client per environment and account, interceptor order, single-flight token refresh and 401 retry, typed errors, retries with backoff + jitter and idempotency keys, AbortController cancellation, ETag / stale-while-revalidate, TanStack Query v5 (onlineManager, persistQueryClient, paused/offline mutations, resumePausedMutations), NetInfo, a local database as the source of truth (expo-sqlite, op-sqlite, WatermelonDB), outbox of queued mutations with optimistic UI, cursor / updatedSince delta sync with tombstones, conflict resolution, background sync with expo-background-task or react-native-background-fetch, push-triggered sync, WebSocket / SSE realtime with catch-up, large background uploads/downloads, certificate pinning, SQLCipher encryption, schema and outbox migrations, and a short GraphQL section (Apollo Client, urql + Graphcache, optimistic responses). Siblings: ios-network-offline-sync, android-network-offline-sync, flutter-network-offline-sync; related: react-native-multi-account, react-native-push-notifications, react-native-deep-links, react-native-list-ux, react-native-crash-analytics, react-native-in-app-purchase.
---

# React Native Networking, Offline & Sync Skill

Guide for building, auditing, and testing the data layer of a consumer React Native app: how requests are made, how failures are classified and retried, how data is cached and persisted, how the app behaves on a train in a tunnel, and how local edits reach the server without loss or duplication. Applies to **bare React Native** and **Expo** (managed with config plugins, dev client, or prebuild). Siblings: **ios-network-offline-sync**, **android-network-offline-sync**, **flutter-network-offline-sync**. Related: **react-native-multi-account** (per-account clients and DBs), **react-native-push-notifications** (silent/data push triggers sync), **react-native-deep-links** (links opened offline), **react-native-list-ux** (pagination, pull-to-refresh, empty/error states), **react-native-crash-analytics** (network breadcrumbs, no PII), **react-native-in-app-purchase** (receipt upload must survive offline).

The approach is **REST-first**. GraphQL gets a short section (§18) because its client libraries bring their own cache and offline story.

Three tiers of "offline", pick deliberately per feature:

1. **Online-only with good failure UX**: typed errors, retry button, no stale lies.
2. **Offline read**: persisted query cache (TanStack Query persister) or local DB; show cached data with "last updated" and refresh on reconnect.
3. **Offline read-write**: local DB is the source of truth, UI observes the DB, writes go into an **outbox** replayed in order with idempotency keys, a **sync engine** pulls deltas with a cursor and applies tombstones.

These libraries move fast. **Verify fast-moving APIs against the current docs and the installed versions in `package.json` before writing code.** Known version gates:

- **TanStack Query v5**: `gcTime` (not `cacheTime`), `isPending` (not `isLoading` for "no data yet"), `networkMode: 'online' | 'always' | 'offlineFirst'`, `onlineManager.setEventListener`, `focusManager` + `AppState`, `PersistQueryClientProvider` from `@tanstack/react-query-persist-client`, `createAsyncStoragePersister` / `createSyncStoragePersister`, `queryClient.setMutationDefaults` + `resumePausedMutations()` for mutations that survive restarts. Persisted mutations need `mutationKey` + defaults registered **before** restore — functions are not serialized.
- **@react-native-community/netinfo** v11+: `NetInfo.addEventListener`, `isConnected` vs `isInternetReachable` (the latter can be `null` while unknown and depends on a reachability probe URL — configurable via `NetInfo.configure`).
- **expo-background-task** (SDK 53+) replaces **expo-background-fetch** (deprecated): uses `BGTaskScheduler` on iOS and WorkManager on Android, `minimumInterval` in **minutes** (floor 15), requires `expo-task-manager` and `defineTask` at module top level. Not available in Expo Go on iOS — use a dev build. **react-native-background-fetch** (Transistor) is the bare-RN alternative with `scheduleTask` and headless Android mode.
- **expo-sqlite** (SDK 51+ "next" API): `openDatabaseAsync`, `execAsync`, `runAsync`, `getAllAsync`, `withExclusiveTransactionAsync`, `addDatabaseChangeListener` (requires `enableChangeListener: true`), SQLCipher via the config plugin option `useSQLCipher` — verify on your SDK. Pairs well with **Drizzle** (`useLiveQuery`).
- **op-sqlite**: JSI, fastest; `open({ name, encryptionKey })` with SQLCipher enabled in `package.json` `op-sqlite` config; reactive queries via `db.reactiveExecute` — verify the API on your version.
- **WatermelonDB**: lazy, observable models; built-in `synchronize({ pullChanges, pushChanges })` protocol with `last_pulled_at` and `created/updated/deleted` change sets; JSI SQLite adapter; requires a config plugin under Expo (`@morrah77/react-native-watermelondb-expo-plugin` or similar — check maintenance).
- **expo-file-system** SDK 54 introduced the object API (`File`, `Directory`, `File.downloadFileAsync`); `createUploadTask` / `createDownloadResumable` live in `expo-file-system/legacy` — verify before using either.
- **fetch on Hermes/RN**: no streaming response bodies by default (SSE needs `react-native-sse` or `expo/fetch` in SDK 52+ which supports streaming); `AbortSignal.timeout()` / `AbortSignal.any()` may be missing — compose controllers by hand.
- **axios** 1.x: `signal` (not `CancelToken`), `isCancel`, interceptors run request interceptors in **reverse** registration order.
- **Apollo Client** 3.x → 4.x (2025) changed imports (`@apollo/client/react`) and error classes; **urql** v4 + `@urql/exchange-graphcache` offline via `offlineExchange` + a storage adapter.
- Expo SDK 52+ defaults to the New Architecture; RN 0.82+ cannot opt out. Libraries above all support it in current versions — verify for older pins.

## 0. Mental model

```
 ┌─────────────────────────────── UI (React) ──────────────────────────────────┐
 │  screens observe:  useLiveQuery(DB)  │  useQuery(server cache)  │  outbox UI │
 └──────────────▲───────────────────────────────▲──────────────────────▲───────┘
                │ reactive reads                 │                      │ status
 ┌──────────────┴──────────┐        ┌────────────┴───────────┐   ┌──────┴───────┐
 │ Repository (per entity) │──write─► Local DB (source of    │◄──┤ Outbox table │
 │  read: DB               │        │ truth: rows + sync     │   │ ordered ops  │
 │  write: DB + outbox txn │        │ meta, cursor, tombst.) │   │ idemp. keys  │
 └──────────────┬──────────┘        └────────────▲───────────┘   └──────┬───────┘
                │ refresh()                      │ apply deltas          │ replay
 ┌──────────────▼────────────────────────────────┴───────────────────────▼───────┐
 │ SyncEngine: push outbox (in order) → pull ?cursor= → apply in txn → save cursor│
 │   triggers: app start · foreground · reconnect · push · bg task · WS "changed"│
 └──────────────┬────────────────────────────────────────────────────────────────┘
                ▼
 ┌───────────────────────────────── ApiClient (one per env/account) ───────────────┐
 │ baseURL · timeout · [request id → auth → idempotency → etag] ─► fetch ─►         │
 │ [decode → typed error → 401 single-flight refresh → retry w/ backoff+jitter]     │
 └──────────────────────────────────────────────────────────────────────────────────┘
   NetInfo / onlineManager = hint ("try now")  ·  server response = truth
```

Rules of thumb:

- **One owner for each datum.** Server-cache data (feed, search) → TanStack Query. User-owned editable data → local DB + outbox. Never both for the same entity, or they drift.
- **Every write is a command with an id.** The id doubles as the idempotency key, so replay after crash, retry, or background task can't create duplicates.
- **Connectivity APIs say when to try; the server says whether it worked.**

## 1. First, establish context

Find out from the codebase, or ask:

- **API style**: REST (JSON, OpenAPI?), GraphQL, gRPC-web, Firebase/Supabase SDK (which ship their own offline story — don't double-layer). Versioned paths or headers?
- **Environments/regions**: dev/staging/prod; data-residency regions; base URL discovered at login (cross-ref **react-native-multi-account**).
- **Freshness**: which screens can show 5-minute-old data, which must be live (prices, stock, chat)? Who invalidates (TTL, push, WebSocket)?
- **Offline tier per feature** (online-only / read / read-write). A consumer app is usually read for content, read-write for a few user-owned entities (notes, cart, drafts, likes, settings).
- **Conflict tolerance**: can two devices edit the same record? Is losing an edit acceptable? Are there server-authoritative fields (price, status, balance)?
- **Volume**: rows per user (100? 100k?), attachment sizes, sync frequency — decides DB choice and delta vs full pull.
- **Server support**: `updatedSince`/cursor endpoints? Tombstones? ETags? Idempotency-Key header? `Retry-After`? Batch endpoint? If not, list what must be added server-side.
- **Auth**: token type and lifetime, refresh endpoint, refresh rotation (one-time refresh tokens make single-flight mandatory).
- **Background needs**: sync while closed? Upload large media in background? Push-triggered refresh?
- **Realtime**: WebSocket, SSE, Firebase, Pusher/Ably? What happens to events missed while disconnected?
- **Stack**: Expo SDK / RN version, New Architecture, fetch vs axios, query library, DB, state library, navigation.

Write the outcome as a table: entity → owner (Query cache / DB) → offline tier → conflict policy → sync trigger.

## 2. HTTP client architecture

- [ ] **Exactly one client factory**; screens never call `fetch` directly. The client is created per environment (and per account — see §3) and passed via context or the session object, not a module singleton that reads global state.
- [ ] **Base URL per env/region** from build config (`app.config.ts` `extra` / `EXPO_PUBLIC_API_URL`, or `react-native-config`); never string-concatenate hosts in feature code. Staging builds can't hit prod by accident.
- [ ] **Timeouts**: fetch has none by default. Use an `AbortController` with a timer (connect+read combined ~15–30 s for JSON, longer/none for uploads). Distinguish timeout from user cancel in the error.
- [ ] **Interceptor order** is fixed and documented. Request: request-id → user-agent/app-version/locale headers → auth → idempotency key → conditional (ETag) → log (redacted). Response: log → decode → map to typed error → 401 refresh & replay → retry policy. axios runs request interceptors in reverse registration order — register accordingly or use a single composed function.
- [ ] **Default headers**: `Accept`, `Content-Type`, `X-App-Version`, `X-Platform`, `Accept-Language`, `X-Request-Id` (per attempt) and a stable `X-Client-Id` (device install id, not PII).
- [ ] **Decoding is validated** (zod / valibot / generated OpenAPI types + runtime check at the boundary). A schema mismatch becomes a `DecodingError`, not `undefined is not an object` three screens later.
- [ ] **Logging redacts** `Authorization`, cookies, tokens in bodies, email/phone. Breadcrumbs (cross-ref **react-native-crash-analytics**) record method, path template, status, duration — not query strings with PII.
- [ ] Prefer `fetch` (or `expo/fetch`) unless you need axios features; don't mix both for the same API.

## 3. Auth

- [ ] Access token attached by the auth step, read from the **session object** for this client, never from a global store at call time (cross-account leak; see **react-native-multi-account**).
- [ ] **Single-flight refresh**: N concurrent 401s → one refresh call; others await the same promise; then each replays **once**. A second 401 after refresh → session expired, not another refresh.
- [ ] Refresh request bypasses the auth interceptor (no recursion), has its own timeout, and is not retried on `invalid_grant`.
- [ ] Proactive refresh when `expiresAt - now < skew` (60 s) avoids a round-trip, but still handle 401 — clocks skew.
- [ ] On refresh failure with `invalid_grant`: mark the account `needsReauth`, keep the outbox (don't wipe unsent user edits), pause sync, show re-login. Network failure during refresh is **not** a logout.
- [ ] Per-account clients: each account has its own client, refresh coordinator, DB file and outbox. Background sync iterates accounts and uses each account's client.
- [ ] Outbox items record the `accountId` that created them; never replay A's outbox with B's token.
- [ ] Tokens live in `expo-secure-store` / `react-native-keychain`, never AsyncStorage or the persisted query cache.

## 4. Error model

- [ ] One discriminated union for every failure the client can produce:
  - `offline` (no route / DNS / `TypeError: Network request failed` when NetInfo says disconnected)
  - `timeout` (our timer fired)
  - `cancelled` (caller aborted — never shown to users, never retried, never reported)
  - `http` with `status`, parsed `code` from the server envelope, `retryAfterMs`, `requestId`
  - `decoding` (schema mismatch — a bug; report to crash tool)
  - `unknown`
- [ ] Server error envelope agreed with backend: `{ code: 'CART_ITEM_UNAVAILABLE', message, details, requestId }`. Map **codes**, not messages, to UI copy.
- [ ] **User-facing mapping** in one place: offline → "You're offline. Changes will sync."; timeout/5xx → "Something went wrong. Try again." + retry; 401 → re-auth; 403 → no access; 404 → gone state; 409 → conflict UI; 422 → field errors; 429 → "Slow down" with Retry-After.
- [ ] `isRetryable(error)` and `isUserVisible(error)` are functions of the type, used by TanStack Query `retry`, the outbox, and UI — not re-derived ad hoc.
- [ ] Errors keep `requestId` so support can trace them.

## 5. Retries

- [ ] **Exponential backoff with full jitter**: `delay = random(0, min(cap, base * 2^attempt))`; base 500 ms–1 s, cap 30 s (foreground) / minutes (outbox). Jitter avoids thundering herds after an outage.
- [ ] **Retry only what is safe**: GET/HEAD/PUT/DELETE (idempotent by HTTP semantics) and POST/PATCH **only with an `Idempotency-Key`** the server honours. Retry on offline, timeout, 408, 425, 429, 500, 502, 503, 504. Never on 400/401/403/404/409/422.
- [ ] **Respect `Retry-After`** (seconds or HTTP date) and cap it.
- [ ] **Idempotency key** = the command id generated when the user acts (UUID v4/v7), stored with the outbox item, identical across all retries and app restarts.
- [ ] TanStack Query: set `retry: (count, err) => count < 3 && isRetryable(err)` and `retryDelay` with jitter globally; default `retry: 3` retries 4xx too.
- [ ] **Circuit breaker** (per host or per endpoint group): after N consecutive 5xx/timeouts, fail fast for a cool-down, then half-open with one probe. Prevents a dead backend from draining battery and hammering on recovery.
- [ ] Bounded attempts for foreground calls (2–3); outbox items retry indefinitely with growing backoff but surface a "stuck" state after M attempts or T hours.

## 6. Cancellation

- [ ] Every request takes an `AbortSignal`. TanStack Query passes `signal` to `queryFn` — forward it to fetch; queries are then cancelled on unmount and key change.
- [ ] **Lifecycle-scoped**: screen unmount, account switch, logout abort their requests. A response arriving after its owner died must never write into a cache/DB of another session.
- [ ] **In-flight GET dedupe**: identical GETs share one promise (TanStack Query does this per key; the raw client can keep a `Map<key, Promise>` for non-Query callers).
- [ ] **Stale search cancel**: typeahead debounces (250–300 ms) and aborts the previous request; results are tagged with the query string and dropped if not current. `placeholderData: keepPreviousData` avoids flashing empty.
- [ ] Cancellation is not an error to show or to report.
- [ ] Mutations are **not** cancelled by unmount once sent — the outbox owns them.

## 7. Caching

- [ ] **ETag / If-None-Match** (or `Last-Modified`/`If-Modified-Since`) for large, slowly changing resources: store the ETag with the cached body; 304 → refresh timestamps only. RN's native HTTP cache (NSURLCache / OkHttp) may handle this transparently for GETs with cache headers — know whether you rely on it, and don't double-cache.
- [ ] **Stale-while-revalidate**: show cached data immediately, refresh in background, update in place. TanStack Query does this with `staleTime` (how long fresh) and `gcTime` (how long kept unused). Set `staleTime` per query family; the default `0` refetches on every mount and focus.
- [ ] **Memory vs persistent**: memory for session-only data; persist (`persistQueryClient` with MMKV-backed sync storage or AsyncStorage) for offline-read tier. `gcTime` must be ≥ `maxAge` of the persister or restored data is immediately GC'd.
- [ ] Persist a **buster** (app version / schema version) and per-account key; dehydrate only successful queries; exclude sensitive queries via `shouldDehydrateQuery`.
- [ ] Show "Last updated 10:42" (`dataUpdatedAt`) on offline-read screens; never present stale prices as current at checkout — re-validate online.
- [ ] Image caching via `expo-image` (`cachePolicy: 'disk'`) or FastImage; authenticated images keyed per account.
- [ ] Invalidate precisely after mutations (`invalidateQueries({ queryKey: ['cart'] })`), not `invalidateQueries()` for everything.

## 8. Connectivity

- [ ] **NetInfo is a hint, not truth.** `isConnected` = has an interface; `isInternetReachable` = probe succeeded (may be `null`, lag by seconds, or be wrong behind captive portals/VPNs). Always attempt the request when the user asks; classify failure afterwards.
- [ ] Wire TanStack Query: `onlineManager.setEventListener(setOnline => NetInfo.addEventListener(s => setOnline(!!s.isConnected)))`. Using `isInternetReachable === false` only (treat `null` as online) avoids false offline at startup.
- [ ] `focusManager` wired to `AppState` so queries refetch on foreground.
- [ ] **Offline banner** after a debounce (e.g. 2 s offline) to avoid flicker; non-blocking; shows pending outbox count. Not a modal.
- [ ] **On reconnect**: kick the outbox, then a sync pull, then `resumePausedMutations()` / refetch active queries. Debounce reconnect storms (Wi-Fi ↔ cellular flaps).
- [ ] Respect **metered/expensive** connections (`details.isConnectionExpensive`, cellular) for large downloads and prefetch; offer a "Wi-Fi only" setting for media.
- [ ] Don't block app start on a reachability check.

## 9. Offline-first architecture

- [ ] For read-write entities the **local DB is the source of truth**: UI observes DB queries; network results are written into the DB, never directly into component state.
- [ ] **DB options**:
  | Option | Use when | Notes |
  |---|---|---|
  | **expo-sqlite** (+ Drizzle) | Expo apps, moderate volume, SQL comfort | `addDatabaseChangeListener` / Drizzle `useLiveQuery`; SQLCipher via plugin; works in Expo Go (Android/iOS) |
  | **op-sqlite** | high volume, perf-critical, bare or dev build | JSI, SQLCipher, reactive queries; not in Expo Go |
  | **WatermelonDB** | 10k–100k+ rows, lazy loading, built-in sync protocol | observable models, `synchronize()`; requires native setup/config plugin; check maintenance |
  | **MMKV** | small key-value state, outbox for tiny apps | not a query engine |
  | Realm | legacy — Atlas Device Sync is deprecated (EOL 2025); plan migration | |
- [ ] Every synced table has sync columns: `id` (client-generated UUID for creates), `server_version` (or `updated_at` from server), `dirty` / `pending_op`, `deleted_at` (local soft delete until pushed), `last_synced_at`.
- [ ] **Client-generated ids** so offline creates can be referenced immediately (children, navigation, deep links). If the server insists on its ids, keep a `local_id → server_id` map and rewrite pending outbox payloads after create succeeds.
- [ ] Repository API: `observeX()`, `getX()`, `createX()`, `updateX()`, `deleteX()`, `refresh()`. Writes = **one transaction**: update row + insert outbox item. Never update UI state and enqueue separately.
- [ ] Server-cache data (feed, search, recommendations) stays in TanStack Query with persistence; do not mirror it into SQLite "just because".
- [ ] DB access off the JS hot path where possible (JSI drivers are synchronous — avoid huge synchronous queries during animations; paginate, see **react-native-list-ux**).

## 10. Outbox for writes

- [ ] **Outbox table**: `id` (command id = idempotency key), `account_id`, `entity`, `entity_id`, `op` (create/update/delete/custom), `payload` (JSON), `base_version`, `created_at`, `attempts`, `next_attempt_at`, `status` (pending / inflight / failed / conflict / done), `last_error`.
- [ ] **Optimistic UI** comes from the DB row already reflecting the change, plus a per-row pending indicator from the outbox (cloud-with-clock icon, "Sending…"). No separate optimistic layer to roll back.
- [ ] **Ordered replay**: per entity (or globally for simplicity) FIFO; an item for entity X never overtakes an earlier one for X. A create must succeed before its updates/children. Parallelize across independent entities only if measured necessary.
- [ ] **Coalescing**: multiple pending updates to the same entity can merge into one PATCH; create+delete before send cancels both — only while neither is in flight.
- [ ] **Idempotency**: `Idempotency-Key: <command id>` on every replay. Server returns the original result for duplicates. Without server support, use PUT-with-client-id semantics.
- [ ] **Per-item failure states**: retryable → backoff; `409` → conflict (see §12); `4xx` validation → `failed` with user action (edit / discard); `401` → pause whole outbox until re-auth; `404` on update → entity deleted remotely → apply tombstone and notify.
- [ ] **Processor is a single-flight loop**: one runner at a time per account (mutex), triggered by enqueue, reconnect, foreground, background task, and push. Survives app kill: `inflight` items found at startup revert to `pending` (idempotency makes resend safe).
- [ ] TanStack Query paused mutations are an acceptable lightweight outbox for **offline-read** apps with a few action types: `networkMode: 'offlineFirst'`, `mutationKey`, `setMutationDefaults({ mutationFn })` registered before `PersistQueryClientProvider` restores, `onSuccess={() => queryClient.resumePausedMutations()}`. They have no ordering per entity beyond scope (`scope: { id }` in v5 serializes mutations with the same scope id) and weaker failure states — use a DB outbox for real read-write.
- [ ] User can see and manage stuck items ("3 changes not synced — Retry / Discard").
- [ ] Logout of an account with pending items: warn ("2 unsynced changes will be lost") and attempt a final flush.

## 11. Sync

- [ ] **Order**: push outbox first, then pull. Pulling first can overwrite local pending edits with older server state.
- [ ] **Cursor-based delta pull**: `GET /sync?cursor=<opaque>&limit=500` returns `{ changes: [...], deleted: [ids], nextCursor, hasMore }`. Loop until `!hasMore`. Prefer an opaque server cursor (sequence/LSN) over `updatedSince=<timestamp>` — timestamps suffer clock skew and same-millisecond ties. If you must use timestamps, use server time from the previous response and overlap by a few seconds (dedupe by id+version).
- [ ] **Apply each page in one DB transaction** together with saving its cursor, so a crash mid-sync resumes cleanly.
- [ ] **Tombstones**: server returns deleted ids (`deleted_at`) since cursor; client hard-deletes or soft-deletes locally. Without tombstones, remote deletes never reach offline clients.
- [ ] Skip applying server rows that have a pending local op (or merge per §12) — don't clobber unsent edits.
- [ ] **Full resync fallback**: server answers `410 Gone` / `cursor_expired` when the tombstone retention window is exceeded, or schema version changed → wipe synced tables (keep outbox), pull from scratch, then reconcile.
- [ ] **Initial sync** paginated with progress; app usable on partial data where possible.
- [ ] Sync is **single-flight per account** and debounced (multiple triggers within 1–2 s collapse).
- [ ] Record `lastSuccessfulSyncAt`; expose to UI and analytics; alert when sync hasn't succeeded for days.
- [ ] WatermelonDB: use its `synchronize()` with `pullChanges({ lastPulledAt, schemaVersion, migration })` / `pushChanges({ changes, lastPulledAt })` and implement the matching server protocol rather than reinventing.

## 12. Conflict resolution

Pick per entity and write it down:

- [ ] **Last-write-wins (LWW)** — simplest; acceptable for user settings, likes, single-device data. Decide by **server receive order**, not device clocks.
- [ ] **Optimistic concurrency with versions/ETags**: update sends `If-Match: <version>` or `base_version`; server returns `409/412` with the current entity when stale. Client then merges or asks.
- [ ] **Field-level merge**: three-way merge using `base` (stored at edit time), `local`, `remote` — non-overlapping field changes merge automatically; overlapping → policy (local wins / remote wins / prompt).
- [ ] **User prompt** only for high-value content (long notes, documents): "This note was changed on another device" with keep mine / keep theirs / copy.
- [ ] **Server-authoritative fields** (price, stock, order status, balance, moderation state) are never sent from the client and always overwritten by the server; optimistic UI for them shows "pending" not a guessed value.
- [ ] Append-only/commutative ops (add item to cart, increment counter) are modelled as **operations**, not final state, so concurrent edits compose.
- [ ] Deletes vs edits: define whether an edit resurrects a remotely deleted item (usually no — show "This item was deleted").
- [ ] CRDTs (Yjs / Automerge) only for collaborative text — heavy; most consumer apps don't need them.

## 13. Background sync

- [ ] **expo-background-task** (Expo SDK 53+): `TaskManager.defineTask(NAME, fn)` at **module top level** (imported from the entry file), `BackgroundTask.registerTaskAsync(NAME, { minimumInterval: 15 })` (minutes). iOS needs `UIBackgroundModes: ["processing"]` + `BGTaskSchedulerPermittedIdentifiers` (the config plugin adds these). Runs are opportunistic; iOS may run rarely for apps the user doesn't open; force-quit on iOS stops them. Test with `BackgroundTask.triggerTaskWorkerForTestingAsync()` (dev builds).
- [ ] **react-native-background-fetch** (bare RN or Expo dev build): `BackgroundFetch.configure({ minimumFetchInterval: 15, stopOnTerminate: false, startOnBoot: true, enableHeadless: true, requiredNetworkType })`, always call `BackgroundFetch.finish(taskId)`, handle the timeout callback, register a headless task for Android.
- [ ] **Budget**: ~30 s of execution on iOS. Do: flush outbox, one pull pass with a time budget, finish. Don't: full resync, large downloads (use background URLSession / WorkManager-based uploaders, §15).
- [ ] The task runs **without UI** and possibly in a fresh JS context: it must build its own clients from storage (registry → token store → DB), not import React context or a UI singleton. Iterate signed-in accounts (cross-ref **react-native-multi-account**).
- [ ] Constraints: network required; on Android prefer unmetered/charging for heavy work; skip when `needsReauth`.
- [ ] **Push-triggered sync** (cross-ref **react-native-push-notifications**): data/silent push `{ type: 'sync', accountId, entity? }` → run a targeted pull. iOS silent pushes (`content-available: 1`) are throttled and not delivered after force-quit; Android data messages with high priority are reliable-ish. Never rely on push alone — it's a nudge; the cursor pull is the truth.
- [ ] Foreground triggers matter more than background ones: app start, `AppState` → active, reconnect, pull-to-refresh.

## 14. Realtime

- [ ] **WebSocket** (built-in `WebSocket`, socket.io-client, Phoenix, Ably/Pusher SDKs) or **SSE** (`react-native-sse`, or `expo/fetch` streaming). SSE is simpler for server → client only.
- [ ] Connect only while foregrounded (and for the active account/portal); close on background — iOS will suspend the socket anyway.
- [ ] **Reconnect with backoff + jitter**, reset on success; pause when NetInfo says offline, resume on reconnect; heartbeat/ping to detect half-open sockets (no pong in 2× interval → reconnect).
- [ ] Auth on connect (token in first message or short-lived ticket from REST); re-auth on token refresh; never put long-lived tokens in the URL (logged by proxies).
- [ ] **Catch-up**: on (re)connect, the client sends its last event sequence/cursor, or simply runs a REST delta sync — events missed while disconnected must not be lost. Treat the socket as "something changed" notifications plus optional payload; apply via the same DB write path as sync, idempotent by `(id, version)`.
- [ ] Dedupe own echoes (events caused by this device's outbox items — match by command id).
- [ ] Batch UI updates (bursts of 100 events shouldn't cause 100 renders) — write to DB in a transaction, let observers re-render once.

## 15. Large transfers

- [ ] Uploads/downloads > a few MB do not go through JSON fetch. Use native background-capable transfer:
  - `expo-file-system` (`createUploadTask` / `createDownloadResumable` — in `expo-file-system/legacy` on SDK 54+; `sessionType: BACKGROUND` on iOS) — verify current API
  - `react-native-background-upload` / `react-native-blob-util` / `@kesha-antonov/react-native-background-downloader` (bare / dev build)
- [ ] **Resumable/multipart**: presigned S3 multipart or tus protocol for big files; persist upload id + completed parts so a kill/reconnect resumes rather than restarts.
- [ ] **Progress** surfaced via DB row (`upload_progress`) so UI survives navigation; throttle progress events (~4/s).
- [ ] Attachment pipeline: pick → copy into app documents dir (picker temp URIs vanish) → compress/resize → enqueue upload → on success, outbox item referencing the remote URL/id. The entity's outbox item depends on the upload finishing.
- [ ] Respect Wi-Fi-only setting and low-data mode; checksum (MD5/SHA-256) on completion when the server supports it.
- [ ] Clean up temp files after success and on discard.

## 16. Security

- [ ] **TLS only**; no `NSAllowsArbitraryLoads` / `usesCleartextTraffic` in release (dev servers: scope exceptions to localhost / debug builds).
- [ ] **Certificate pinning** (`react-native-ssl-public-key-pinning`, or native config): pin SPKI hashes of the intermediate/leaf with ≥ 2 backup pins, have a remote kill switch or short-lived pin set, plan rotations. Trade-off: outages when certs rotate unexpectedly, breaks corporate proxies and debugging. Many consumer apps skip pinning; decide explicitly.
- [ ] **No secrets in the JS bundle**: anything in `EXPO_PUBLIC_*`, `extra`, or JS is public. Third-party API keys with spend go behind your backend. Hermes bytecode is not obfuscation.
- [ ] **Local DB encryption** for sensitive user data: SQLCipher (op-sqlite, expo-sqlite `useSQLCipher`), key generated per install/account and stored in Keychain/SecureStore. Persisted query cache in MMKV with `encryptionKey` if it contains personal data.
- [ ] Logout wipes the account's DB, persisted cache, outbox (after warning), and files.
- [ ] Don't log request/response bodies in release; sanitize crash breadcrumbs.
- [ ] Validate deep-link-supplied ids before fetching (cross-ref **react-native-deep-links**).

## 17. Migrations

- [ ] **DB schema versioning**: `PRAGMA user_version` (expo-sqlite / op-sqlite) or Drizzle migrations / WatermelonDB `schemaMigrations`. Migrations are forward-only, tested from every shipped version, run before any repository is used.
- [ ] Destructive changes to synced tables → may trigger a full resync instead of migrating data; never drop the outbox.
- [ ] **Outbox migration**: outbox payloads are versioned (`payload_version`). A new app version must be able to replay items written by an old one — transform on read or keep old endpoints until the outbox drains. OTA updates (EAS Update) can change JS without native DB changes — gate by runtime version.
- [ ] Persisted query cache: bump the persister `buster` when response shapes change.
- [ ] **API version compatibility**: old app versions live for years. Server keeps old versions or the client handles additive fields gracefully (unknown fields ignored, unknown enum values mapped to `unknown`). Support a server-driven `426 Upgrade Required` / min-version check → force-update screen.
- [ ] Sync protocol carries `schemaVersion` so the server can send compatible changes.

## 18. GraphQL (short)

- [ ] **Normalized cache**: Apollo `InMemoryCache` with `typePolicies` (`keyFields`, `merge` for pagination fields) or urql **Graphcache** (`keys`, `resolvers`, `updates`). Every type needs a stable id (`id` + `__typename`) or normalization fails silently.
- [ ] **Persistence**: Apollo `apollo3-cache-persist` (`persistCache({ cache, storage: new MMKVWrapper(mmkv) })`) — restore before rendering; urql `offlineExchange` + `@urql/storage-rn` makes queries offline-readable and queues mutations while offline.
- [ ] **Optimistic responses**: Apollo `optimisticResponse` + `update(cache, …)` for list inserts; urql `optimistic` updaters. Rolled back automatically on error; ensure the optimistic `id` is replaced by the server id (or use client ids).
- [ ] Offline mutation queue: Apollo has no built-in durable queue (community `apollo-link-queue` / `@wora/apollo-offline` — check maintenance); for read-write offline, still use a DB outbox and call mutations from the processor.
- [ ] Fetch policies: `cache-and-network` ≈ stale-while-revalidate; `cache-first` for static; `network-only` for checkout.
- [ ] Errors: GraphQL returns 200 with `errors[]` — map `extensions.code` into the same typed error model (§4). Persisted queries (APQ) cut payload size; subscriptions follow §14 rules (graphql-ws).

## 19. Testing

### 19.1 Tools

1. **Unit** (Jest): client error mapping, retry policy (`jest.useFakeTimers` + injected random), single-flight refresh, outbox ordering/coalescing, sync page application, conflict merge — all plain TypeScript with an in-memory or `better-sqlite3`-backed DB adapter.
2. **Mock server**: MSW (`msw/native` in RN, `msw/node` in Jest) with scenario handlers: 500 burst, 429 + Retry-After, slow responses, 409 with current entity, cursor expired, duplicate idempotency key.
3. **Network shaping**: iOS Network Link Conditioner, Android emulator network settings (`adb emu network speed edge`), Charles/Proxyman throttling + breakpoints, toxiproxy for CI.
4. **Airplane-mode matrix** on real devices: offline at launch, offline mid-request, flapping, captive portal Wi-Fi.
5. **Background**: `BackgroundTask.triggerTaskWorkerForTestingAsync()`, Xcode `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"…"]`, `adb shell cmd jobscheduler run -f <pkg> <id>`, react-native-background-fetch's `simulate` commands.
6. **Clock skew**: change device time ±1 day/±5 min; assert sync uses server cursor and token refresh still works.
7. **E2E**: Maestro / Detox with a seeded backend; two devices (or two simulators) for conflict tests.

### 19.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| N1 | Launch offline with persisted data | Cached data shown with "last updated"; offline banner; no error spinner loop |
| N2 | Launch offline, never synced | Empty offline state with retry; no crash |
| N3 | Create item offline | Appears immediately with pending badge; outbox has 1 item |
| N4 | Edit then delete offline, reconnect | Coalesced; server receives no create or a create+delete per policy; no zombie |
| N5 | 10 offline edits, reconnect | Replayed in order; final server state = final local state |
| N6 | Kill app while item in flight | Relaunch → item resent with same Idempotency-Key; exactly one server record |
| N7 | Server 500 for an outbox item | Backoff with jitter; other entities continue; after M attempts "not synced" UI |
| N8 | 422 on an outbox item | Item marked failed; user can edit/discard; queue continues |
| N9 | 409 conflict (other device edited) | Merge per policy or prompt; no silent loss of user text |
| N10 | Remote delete while local edit pending | Per policy: "item was deleted" notice; outbox item dropped |
| N11 | Delta pull with tombstones | Remote deletes removed locally |
| N12 | Cursor expired (410) | Full resync; outbox preserved and pushed |
| N13 | Crash mid-sync page | Resume from last committed cursor; no duplicate rows |
| N14 | 20 parallel 401s | One refresh; each request replayed once |
| N15 | Refresh fails offline | No logout; retried on reconnect |
| N16 | Refresh `invalid_grant` | Re-login prompt; outbox kept; resumes after login |
| N17 | 429 with Retry-After: 30 | No request before 30 s |
| N18 | Backend down 5 min | Circuit opens; battery-friendly; recovers with probe |
| N19 | Typeahead fast typing | Only latest results shown; earlier requests aborted |
| N20 | Navigate away mid-request | Request aborted; no state update on unmounted screen |
| N21 | Wi-Fi ↔ cellular flap | One sync, not ten; socket reconnects once |
| N22 | Captive portal (connected, no internet) | Requests fail as offline/timeout; banner after debounce; no HTML decoded as JSON crash |
| N23 | Background task fires | Outbox flushed, pull done within budget, `finish` called |
| N24 | Silent push "sync" | Targeted pull; works cold (Android) / when allowed (iOS) |
| N25 | WebSocket drop for 2 min | Reconnect with backoff; missed events caught up via sync |
| N26 | Own outbox echo over WebSocket | Not applied twice; no flicker |
| N27 | 200 MB upload, app backgrounded | Continues natively; progress restored on return |
| N28 | Upload interrupted at 60% | Resumes from last part, not from zero |
| N29 | Device clock +1 day | Sync/order unaffected; tokens refresh; no "edited in the future" bugs |
| N30 | App upgrade with old outbox items | Items replay successfully after migration |
| N31 | DB schema migration from every shipped version | Data intact or clean resync |
| N32 | Account switch with pending outbox (cross-ref multi-account) | A's items stay with A; replayed with A's token later |
| N33 | Logout with unsynced changes | Warning; final flush attempt; then wipe |
| N34 | Persisted query cache after app update with new shape | Buster discards; no decode crash |
| N35 | Force-update (426) | Force-update screen; outbox kept for after update |

### 19.3 Example tests

```ts
it('replays the outbox in order with stable idempotency keys after a crash', async () => {
  const seen: string[] = [];
  server.use(http.patch('*/notes/:id', ({ request }) => {
    seen.push(request.headers.get('Idempotency-Key')!);
    return HttpResponse.json({ version: seen.length });
  }));
  await repo.updateNote('n1', { title: 'a' });
  await repo.updateNote('n1', { body: 'b' });
  await db.runAsync(`UPDATE outbox SET status='inflight'`);      // simulate kill mid-send
  await outbox.recoverInflight();
  await outbox.drain();
  expect(seen).toHaveLength(1);                                    // coalesced PATCH
  expect(await outbox.pendingCount()).toBe(0);
});

it('does not retry 422 but retries 503 honouring Retry-After', async () => {
  expect(isRetryable({ kind: 'http', status: 422 } as ApiError)).toBe(false);
  const e: ApiError = { kind: 'http', status: 503, retryAfterMs: 5000 } as ApiError;
  expect(backoffDelay(0, e, () => 0.99)).toBe(5000);
});
```

## 20. Common bugs to look for in review

- `fetch` with no timeout → spinner forever on a dead connection
- `retry: 3` default in TanStack Query retrying 401/403/404/422 → 4× latency and duplicate side effects
- POST retried without an Idempotency-Key → double orders / duplicate comments
- Idempotency key generated per attempt instead of per user action
- Refresh lock as a boolean, not a shared promise → parallel refreshes invalidate each other's rotated refresh token → user logged out
- Network failure during refresh treated as logout
- `if (!netInfo.isInternetReachable) return` blocking requests while it is still `null` at startup
- Offline banner flickers on every Wi-Fi → cellular handoff (no debounce)
- Optimistic update in TanStack Query cache *and* in SQLite for the same entity → the two disagree after rollback
- Writing the row and enqueuing the outbox item in separate transactions → crash leaves an edit that never syncs
- Pull before push → server state overwrites pending local edits
- Sync with `updatedSince=Date.now()` from device clock → skew drops changes; same-millisecond ties skip rows
- No tombstones → items deleted on web stay forever on phone
- Cursor saved before the page is applied (or outside the transaction) → lost changes after a crash
- `inflight` outbox items never reset after a crash → queue stuck forever
- Persisted mutations restored before `setMutationDefaults` → paused mutations with no `mutationFn` silently dropped
- `gcTime` shorter than persister `maxAge` → restored cache discarded immediately
- No persister `buster` → old cache shape crashes new code after update
- `defineTask` inside a component or after an async import → background task never registered on cold launch
- Background task imports a React context / UI singleton → wrong or no account in headless run; `BackgroundFetch.finish` not called → OS stops scheduling
- WebSocket reconnect without jitter/backoff → reconnect storm after server deploy; no catch-up → missed messages
- Socket kept open in background → battery drain and OS kills
- Token in WebSocket URL query → logged by proxies
- Large file uploaded via `fetch` with base64 body → OOM, no background, no resume
- Picker temp URI stored in outbox → file gone by replay time
- Cleartext HTTP allowed in release; API keys with spend in `EXPO_PUBLIC_*`
- Pinning with a single pin and no rotation plan → app bricked when cert rotates
- Account switch keeps A's sync loop running against B's DB (cross-ref multi-account)
- Captive portal returns HTML 200 → JSON parse crash instead of typed `decoding`/`offline`

## 21. Output format

When asked to **review/audit**:
1. **Summary**: stack (Expo/bare, RN, HTTP client, query lib, DB, background lib, realtime, GraphQL client) and the entity table from §1 as found
2. **Checklist results**: sections 2–18 with ✅ Pass / ❌ Fail / ⚠️ Needs check, with file:line
3. **Critical issues first**: data loss (lost/duplicated writes, clobbered edits), infinite retries/battery drain, auth refresh races, security (cleartext, secrets in bundle) — each with fix code
4. **Testing gaps**: N1–N35 not covered, plus suggested tests

When asked to **implement**: confirm versions (Expo SDK, TanStack Query v5, NetInfo, DB driver, background lib), fill the entity table, agree server contracts (error envelope, Idempotency-Key, sync endpoint with cursor + tombstones, 409 body), then start from the Appendix and adapt. Ship in order: client + errors + retries → caching/offline read → DB + outbox → sync → background/realtime. List what to verify on device: airplane-mode matrix, kill during send, background task trigger, clock skew.

When asked to **debug**: classify the symptom (request never sent / sent twice / sent with wrong auth / response dropped / data overwritten / stale), log `requestId`, command id, cursor and `(accountId)` across the path client → outbox → sync → DB, and reproduce with the closest N-scenario under a mock server.

---

## Appendix — Reference implementation (TypeScript, Expo SDK 53+ / bare RN 0.79+)

Uses `fetch`, `expo-sqlite` (next API), TanStack Query v5, NetInfo, `expo-background-task`. Swap the DB adapter for op-sqlite or WatermelonDB as needed. Simplified: one account shown; in multi-account apps create one of each object per account (see **react-native-multi-account**).

### A.1 Typed errors and retry policy

```ts
// net/errors.ts
export type ApiError =
  | { kind: 'offline'; cause?: unknown }
  | { kind: 'timeout' }
  | { kind: 'cancelled' }
  | { kind: 'http'; status: number; code?: string; message?: string;
      details?: unknown; retryAfterMs?: number; requestId?: string }
  | { kind: 'decoding'; cause: unknown; requestId?: string }
  | { kind: 'unknown'; cause: unknown };

export class ApiException extends Error { constructor(public readonly error: ApiError) { super(error.kind); } }
export const asApiError = (e: unknown): ApiError =>
  e instanceof ApiException ? e.error : { kind: 'unknown', cause: e };

const RETRYABLE_STATUS = new Set([408, 425, 429, 500, 502, 503, 504]);
export const isRetryable = (e: ApiError): boolean =>
  e.kind === 'offline' || e.kind === 'timeout' || (e.kind === 'http' && RETRYABLE_STATUS.has(e.status));

/** Full jitter; honours Retry-After when larger. */
export function backoffDelay(attempt: number, e?: ApiError, rand = Math.random,
                             baseMs = 1000, capMs = 30_000): number {
  const jittered = Math.floor(rand() * Math.min(capMs, baseMs * 2 ** attempt));
  const ra = e?.kind === 'http' ? e.retryAfterMs ?? 0 : 0;
  return Math.min(Math.max(jittered, ra), 5 * 60_000);
}

/** Retry-After as delta-seconds or HTTP-date → ms (undefined if absent/unparseable). */
export declare function parseRetryAfter(h: string | null): number | undefined;

// userMessage(e): map kind/status → copy; 'cancelled' → null (no toast), offline → "changes will sync",
// 429/403/404/5xx → specific strings, else server message or generic.
```

### A.2 Configured client: timeout, auth, single-flight refresh, retries, circuit breaker

```ts
// net/client.ts
import { ApiError, ApiException, backoffDelay, isRetryable, parseRetryAfter } from './errors';

export interface TokenProvider {          // refresh() throws ApiException(http 401) on invalid_grant
  getAccessToken(): Promise<string | null>; refresh(): Promise<string>; onSessionExpired(): void;
}
export interface RequestOptions<T> {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE'; body?: unknown; headers?: Record<string, string>;
  signal?: AbortSignal; timeoutMs?: number; retries?: number; auth?: boolean;
  idempotencyKey?: string;                 // required for retrying POST/PATCH
  decode?: (json: unknown) => T;           // zod: Schema.parse
}

declare const uuid: () => string;          // crypto.randomUUID ?? expo-crypto randomUUID
declare function sleep(ms: number, signal?: AbortSignal): Promise<void>;  // rejects 'cancelled' on abort

// CircuitBreaker: after 5 consecutive timeouts/5xx, canRequest() is false for 30 s; then one probe
// (a further failure re-opens immediately); success() resets the count.
declare class CircuitBreaker { canRequest(): boolean; success(): void; failure(): void; }

export class ApiClient {
  private refreshing: Promise<string> | null = null;
  private breaker = new CircuitBreaker();
  constructor(private cfg: { baseUrl: string; appVersion: string; clientId: string;
                             tokens: TokenProvider; defaultTimeoutMs?: number }) {}

  async request<T>(path: string, o: RequestOptions<T> = {}): Promise<T> {
    const method = o.method ?? 'GET';
    const safe = method === 'GET' || method === 'PUT' || method === 'DELETE' || !!o.idempotencyKey;
    const maxRetries = safe ? o.retries ?? 2 : 0;
    for (let attempt = 0; ; attempt++) {
      try {
        if (!this.breaker.canRequest()) throw new ApiException({ kind: 'http', status: 503, code: 'CIRCUIT_OPEN' });
        const res = await this.send(path, method, o, false); this.breaker.success(); return res as T;
      } catch (e) {
        const err = e instanceof ApiException ? e.error : ({ kind: 'unknown', cause: e } as ApiError);
        if (err.kind === 'timeout' || (err.kind === 'http' && err.status >= 500)) this.breaker.failure();
        if (attempt >= maxRetries || !isRetryable(err)) throw e;
        await sleep(backoffDelay(attempt, err), o.signal);
      }
    }
  }

  private async send<T>(path: string, method: string, o: RequestOptions<T>, isReplay: boolean): Promise<T> {
    const headers: Record<string, string> = { Accept: 'application/json', 'X-Request-Id': uuid(),
      'X-App-Version': this.cfg.appVersion, 'X-Client-Id': this.cfg.clientId, ...o.headers };
    if (o.body !== undefined) headers['Content-Type'] = 'application/json';
    const token = o.auth !== false ? await this.cfg.tokens.getAccessToken() : null;
    if (token) headers.Authorization = `Bearer ${token}`;
    if (o.idempotencyKey) headers['Idempotency-Key'] = o.idempotencyKey;

    // Compose caller signal + timeout (AbortSignal.any may be unavailable on Hermes).
    const ctrl = new AbortController();
    let timedOut = false;
    const timer = setTimeout(() => { timedOut = true; ctrl.abort(); }, o.timeoutMs ?? this.cfg.defaultTimeoutMs ?? 20_000);
    const onAbort = () => ctrl.abort();
    o.signal?.addEventListener('abort', onAbort);

    let res: Response;
    try {
      res = await fetch(this.cfg.baseUrl + path, { method, headers, signal: ctrl.signal,
        body: o.body === undefined ? undefined : JSON.stringify(o.body) });
    } catch (cause) {
      if (o.signal?.aborted) throw new ApiException({ kind: 'cancelled' });
      if (timedOut) throw new ApiException({ kind: 'timeout' });
      // Transport failure: no route, DNS, TLS, reset. NetInfo is only used for UI, never to gate.
      throw new ApiException({ kind: 'offline', cause });
    } finally { clearTimeout(timer); o.signal?.removeEventListener('abort', onAbort); }

    const requestId = res.headers.get('X-Request-Id') ?? headers['X-Request-Id'];
    if (res.status === 401 && o.auth !== false && !isReplay) {
      try { await this.refreshOnce(); }
      catch { this.cfg.tokens.onSessionExpired(); throw new ApiException({ kind: 'http', status: 401, requestId }); }
      return this.send(path, method, o, true);           // replay exactly once
    }
    if (res.status === 204) return undefined as T;

    const text = await res.text();
    let json: unknown = undefined;
    try { json = text ? JSON.parse(text) : undefined; }
    catch (cause) {                                       // captive portals return HTML 200
      throw new ApiException(res.ok ? { kind: 'decoding', cause, requestId } : { kind: 'http', status: res.status, requestId });
    }
    if (!res.ok) {
      const env = (json ?? {}) as { code?: string; message?: string; details?: unknown };
      throw new ApiException({ kind: 'http', status: res.status, code: env.code, message: env.message,
        details: env.details, retryAfterMs: parseRetryAfter(res.headers.get('Retry-After')), requestId });
    }
    try { return o.decode ? o.decode(json) : (json as T); }
    catch (cause) { throw new ApiException({ kind: 'decoding', cause, requestId }); }
  }

  /** N concurrent 401s share one refresh. */
  private refreshOnce(): Promise<string> {
    return (this.refreshing ??= this.cfg.tokens.refresh().finally(() => { this.refreshing = null; }));
  }
}
```

### A.3 TanStack Query wiring: online/focus managers, persistence, paused mutations

```tsx
// net/query.tsx
import { AppState, Platform } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { QueryClient, focusManager, onlineManager } from '@tanstack/react-query';
import { asApiError, backoffDelay, isRetryable } from './errors';

onlineManager.setEventListener(setOnline =>
  NetInfo.addEventListener(s => setOnline(s.isConnected !== false && s.isInternetReachable !== false)));

AppState.addEventListener('change', s => { if (Platform.OS !== 'web') focusManager.setFocused(s === 'active'); });

export function makeQueryClient(api: import('./client').ApiClient) {
  const qc = new QueryClient({
    defaultOptions: {
      queries: { staleTime: 30_000, gcTime: 24 * 60 * 60_000 /* ≥ persister maxAge */, networkMode: 'offlineFirst',
        retry: (n, e) => n < 3 && isRetryable(asApiError(e)), retryDelay: (n, e) => backoffDelay(n, asApiError(e)) },
      mutations: { networkMode: 'offlineFirst' },
    },
  });
  // Must be registered BEFORE restore so persisted paused mutations have a mutationFn; scope serializes.
  qc.setMutationDefaults(['like'], { scope: { id: 'likes' },
    mutationFn: ({ postId, liked, commandId }: { postId: string; liked: boolean; commandId: string }) =>
      api.request(`/posts/${postId}/like`, { method: liked ? 'PUT' : 'DELETE', idempotencyKey: commandId }) });
  return qc;
}

// Persistence: createSyncStoragePersister over an MMKV instance (key per account), then
// <PersistQueryClientProvider persistOptions={{ persister, maxAge: 24h, buster: '<schema-version>',
//   dehydrateOptions: { shouldDehydrateQuery: q => q.state.status === 'success' } }}
//   onSuccess={() => client.resumePausedMutations().then(() => client.invalidateQueries())}>
```

### A.4 Local DB schema and repository (network → DB → UI)

```ts
// data/db.ts
import * as SQLite from 'expo-sqlite';

// openDb(name): SQLite.openDatabaseAsync(name, { enableChangeListener: true }), PRAGMA WAL + foreign_keys, migrate(db).

const MIGRATIONS: string[] = [
  `CREATE TABLE notes (id TEXT PRIMARY KEY, title TEXT NOT NULL, body TEXT NOT NULL,
     server_version INTEGER, updated_at INTEGER NOT NULL, deleted INTEGER NOT NULL DEFAULT 0);
   CREATE TABLE outbox (id TEXT PRIMARY KEY, entity TEXT NOT NULL, entity_id TEXT NOT NULL,
     op TEXT NOT NULL, payload TEXT NOT NULL, payload_version INTEGER NOT NULL DEFAULT 1,
     base_version INTEGER, seq INTEGER NOT NULL, attempts INTEGER NOT NULL DEFAULT 0,
     next_attempt_at INTEGER NOT NULL DEFAULT 0, status TEXT NOT NULL DEFAULT 'pending', last_error TEXT);
   CREATE INDEX outbox_order ON outbox(status, seq);
   CREATE TABLE sync_state (key TEXT PRIMARY KEY, value TEXT);`,
  // v2: ALTER TABLE ... — never drop outbox
];

// migrate(db): read PRAGMA user_version; for each later MIGRATIONS[v], run it and set
// user_version = v + 1 inside one withExclusiveTransactionAsync.

// data/notesRepository.ts
import { randomUUID } from 'expo-crypto';

export type Note = { id: string; title: string; body: string; server_version: number | null;
                     updated_at: number; pending: number };

export class NotesRepository {
  constructor(private db: SQLite.SQLiteDatabase, private kick: () => void) {}

  list(): Promise<Note[]> {
    return this.db.getAllAsync<Note>(
      `SELECT n.*, EXISTS(SELECT 1 FROM outbox o WHERE o.entity='note' AND o.entity_id=n.id
                          AND o.status!='done') AS pending
       FROM notes n WHERE n.deleted = 0 ORDER BY n.updated_at DESC`);
  }

  async create(title: string, body: string): Promise<string> {
    const id = randomUUID();
    await this.write(id, 'create', { id, title, body }, null, async tx =>
      tx.runAsync('INSERT INTO notes (id,title,body,updated_at) VALUES (?,?,?,?)', id, title, body, Date.now()));
    return id;
  }

  async update(id: string, patch: Partial<Pick<Note, 'title' | 'body'>>) {
    const cur = await this.db.getFirstAsync<Note>('SELECT * FROM notes WHERE id=?', id);
    if (!cur) return;
    await this.write(id, 'update', patch, cur.server_version, async tx =>
      tx.runAsync('UPDATE notes SET title=?, body=?, updated_at=? WHERE id=?',
        patch.title ?? cur.title, patch.body ?? cur.body, Date.now(), id));
  }

  // remove(id): write(id, 'delete', {}, null, tx => soft-delete row: deleted=1)

  /** Row change + outbox enqueue (with coalescing) in ONE transaction. */
  private async write(entityId: string, op: 'create' | 'update' | 'delete', payload: object,
                      baseVersion: number | null, apply: (tx: SQLite.SQLiteDatabase) => Promise<unknown>) {
    await this.db.withExclusiveTransactionAsync(async tx => {
      await apply(tx);
      const prev = await tx.getFirstAsync<{ id: string; op: string; payload: string }>(
        `SELECT id, op, payload FROM outbox WHERE entity='note' AND entity_id=? AND status='pending'
         ORDER BY seq DESC LIMIT 1`, entityId);
      if (prev && op === 'update' && (prev.op === 'update' || prev.op === 'create')) {
        const merged = { ...JSON.parse(prev.payload), ...payload };
        await tx.runAsync('UPDATE outbox SET payload=? WHERE id=?', JSON.stringify(merged), prev.id);
        return;
      }
      if (prev && op === 'delete' && prev.op === 'create') {        // never reached server
        await tx.runAsync('DELETE FROM outbox WHERE entity_id=? AND status=\'pending\'', entityId);
        await tx.runAsync('DELETE FROM notes WHERE id=?', entityId);
        return;
      }
      const seq = (await tx.getFirstAsync<{ s: number }>('SELECT COALESCE(MAX(seq),0)+1 AS s FROM outbox'))!.s;
      await tx.runAsync(
        'INSERT INTO outbox (id,entity,entity_id,op,payload,base_version,seq) VALUES (?,?,?,?,?,?,?)',
        randomUUID(), 'note', entityId, op, JSON.stringify(payload), baseVersion, seq);
    });
    this.kick();                                                    // try now; outbox decides
  }
}

// useNotes(repo): UI observes the DB, not the network — load repo.list() on mount and on
// SQLite.addDatabaseChangeListener events for 'notes'/'outbox'; remove the subscription on unmount.
```

### A.5 Outbox processor

```ts
// sync/outbox.ts
import * as SQLite from 'expo-sqlite';
import { ApiClient } from '../net/client';
import { ApiException, asApiError, backoffDelay, isRetryable } from '../net/errors';

type Item = { id: string; entity: string; entity_id: string; op: string; payload: string;
              base_version: number | null; attempts: number };
const STUCK_AFTER = 8;

export class OutboxProcessor {
  private running: Promise<void> | null = null;
  private paused = false;                       // set while needsReauth
  constructor(private db: SQLite.SQLiteDatabase, private api: ApiClient,
              private onConflict: (item: Item, server: unknown) => Promise<void>) {}

  /** Call once at startup: anything "inflight" was interrupted; idempotency makes resend safe. */
  recoverInflight() { return this.db.runAsync(`UPDATE outbox SET status='pending' WHERE status='inflight'`); }
  pendingCount() { return this.db.getFirstAsync<{ c: number }>(
    `SELECT COUNT(*) c FROM outbox WHERE status IN ('pending','inflight')`).then(r => r?.c ?? 0); }
  pause(p: boolean) { this.paused = p; }

  drain(): Promise<void> {                      // single-flight
    return (this.running ??= this.loop().finally(() => { this.running = null; }));
  }

  private async loop() {
    const blocked = new Set<string>();          // entities with a failing head item keep order
    while (!this.paused) {
      const items = await this.db.getAllAsync<Item>(
        `SELECT * FROM outbox WHERE status='pending' AND next_attempt_at <= ? ORDER BY seq LIMIT 50`, Date.now());
      const next = items.find(i => !blocked.has(i.entity_id));
      if (!next) return;
      await this.db.runAsync(`UPDATE outbox SET status='inflight' WHERE id=?`, next.id);
      try {
        const res = await this.send(next);
        await this.db.withExclusiveTransactionAsync(async tx => {
          await tx.runAsync(`DELETE FROM outbox WHERE id=?`, next.id);
          if (res?.version != null)
            await tx.runAsync(`UPDATE notes SET server_version=? WHERE id=?`, res.version, next.entity_id);
        });
      } catch (e) {
        const err = asApiError(e);
        if (err.kind === 'cancelled') { await this.setStatus(next, 'pending'); return; }
        if (err.kind === 'http' && err.status === 401) { this.paused = true; await this.setStatus(next, 'pending'); return; }
        if (err.kind === 'http' && (err.status === 409 || err.status === 412)) {
          await this.setStatus(next, 'conflict', err); blocked.add(next.entity_id);
          await this.onConflict(next, err.details); continue;
        }
        if (err.kind === 'http' && err.status === 404 && next.op !== 'create') {   // remotely deleted
          await this.db.runAsync(`DELETE FROM outbox WHERE id=?`, next.id); continue;
        }
        if (isRetryable(err)) {
          const attempts = next.attempts + 1;
          await this.db.runAsync(
            `UPDATE outbox SET status=?, attempts=?, next_attempt_at=?, last_error=? WHERE id=?`,
            attempts >= STUCK_AFTER ? 'failed' : 'pending', attempts,
            Date.now() + backoffDelay(attempts, err, Math.random, 2000, 10 * 60_000),
            JSON.stringify(err), next.id);
          blocked.add(next.entity_id);
          if (err.kind === 'offline') return;   // stop; reconnect will kick us
          continue;
        }
        await this.setStatus(next, 'failed', err); blocked.add(next.entity_id);   // 4xx: user action
      }
    }
  }

  private send(i: Item): Promise<{ version?: number } | undefined> {
    const body = JSON.parse(i.payload);
    const o = { idempotencyKey: i.id, retries: 0, timeoutMs: 30_000 } as const;
    switch (i.op) {
      case 'create': return this.api.request(`/notes`, { ...o, method: 'POST', body });
      case 'update': return this.api.request(`/notes/${i.entity_id}`, { ...o, method: 'PATCH', body,
        headers: i.base_version != null ? { 'If-Match': String(i.base_version) } : undefined });
      case 'delete': return this.api.request(`/notes/${i.entity_id}`, { ...o, method: 'DELETE' });
      default: throw new ApiException({ kind: 'unknown', cause: `unknown op ${i.op}` });
    }
  }
  private setStatus(i: Item, status: string, err?: unknown) {
    return this.db.runAsync(`UPDATE outbox SET status=?, last_error=? WHERE id=?`,
      status, err ? JSON.stringify(err) : null, i.id);
  }
}
```

### A.6 Sync engine: push, then cursor pull with tombstones

```ts
// sync/engine.ts
import * as SQLite from 'expo-sqlite';
import { ApiClient } from '../net/client';
import { asApiError } from '../net/errors';
import { OutboxProcessor } from './outbox';

type ServerNote = { id: string; title: string; body: string; version: number; updatedAt: number };
type SyncPage = { changes: ServerNote[]; deleted: string[]; nextCursor: string; hasMore: boolean };

export class SyncEngine {
  private running: Promise<void> | null = null;
  private lastTrigger = 0;
  constructor(private db: SQLite.SQLiteDatabase, private api: ApiClient, private outbox: OutboxProcessor) {}

  /** Debounced, single-flight. Triggers: start, foreground, reconnect, push, bg task, socket. */
  sync(opts: { budgetMs?: number; signal?: AbortSignal } = {}): Promise<void> {
    if (this.running) return this.running;
    if (Date.now() - this.lastTrigger < 1500) return Promise.resolve();
    this.lastTrigger = Date.now();
    return (this.running = this.run(opts).finally(() => { this.running = null; }));
  }

  private async run({ budgetMs = 60_000, signal }: { budgetMs?: number; signal?: AbortSignal }) {
    const deadline = Date.now() + budgetMs;
    await this.outbox.drain();                                   // 1. push first
    let cursor = await this.getState('notes.cursor');            // 2. pull deltas
    for (;;) {
      if (Date.now() > deadline || signal?.aborted) return;      // resume next time
      let page: SyncPage;
      try {
        page = await this.api.request<SyncPage>(
          `/sync/notes?limit=500${cursor ? `&cursor=${encodeURIComponent(cursor)}` : ''}`, { signal });
      } catch (e) {
        const err = asApiError(e);
        if (err.kind === 'http' && (err.status === 410 || err.code === 'CURSOR_EXPIRED')) {
          await this.resetForFullResync(); cursor = null; continue;
        }
        throw e;
      }
      await this.applyPage(page);                                // 3. apply + save cursor atomically
      cursor = page.nextCursor;
      if (!page.hasMore) break;
    }
    await this.setState('notes.lastSyncAt', String(Date.now()));
  }

  private async applyPage(page: SyncPage) {
    await this.db.withExclusiveTransactionAsync(async tx => {
      for (const n of page.changes) {
        const pending = await tx.getFirstAsync(
          `SELECT 1 FROM outbox WHERE entity='note' AND entity_id=? AND status!='done'`, n.id);
        if (pending) continue;                                   // don't clobber unsent edits
        await tx.runAsync(
          `INSERT INTO notes (id,title,body,server_version,updated_at,deleted) VALUES (?,?,?,?,?,0)
           ON CONFLICT(id) DO UPDATE SET title=excluded.title, body=excluded.body,
             server_version=excluded.server_version, updated_at=excluded.updated_at, deleted=0
           WHERE notes.server_version IS NULL OR notes.server_version < excluded.server_version`,
          n.id, n.title, n.body, n.version, n.updatedAt);
      }
      for (const id of page.deleted) {                           // tombstones
        await tx.runAsync(`DELETE FROM notes WHERE id=?`, id);
        await tx.runAsync(`DELETE FROM outbox WHERE entity='note' AND entity_id=? AND op!='create'`, id);
      }
      await tx.runAsync(`INSERT OR REPLACE INTO sync_state (key,value) VALUES ('notes.cursor', ?)`, page.nextCursor);
    });
  }

  /** Cursor expired / schema change: drop synced rows that have no pending op; keep outbox. */
  private async resetForFullResync() {
    await this.db.withExclusiveTransactionAsync(async tx => {
      await tx.runAsync(`DELETE FROM notes WHERE id NOT IN (SELECT entity_id FROM outbox WHERE entity='note')`);
      await tx.runAsync(`DELETE FROM sync_state WHERE key='notes.cursor'`);
    });
  }

  // getState/setState: SELECT / INSERT OR REPLACE on sync_state(key, value)
  private declare getState: (k: string) => Promise<string | null>;
  private declare setState: (k: string, v: string) => Promise<unknown>;
}
```

### A.7 Triggers: reconnect, foreground, background task, push

```ts
// sync/triggers.ts  — import from the app entry (index.ts / app/_layout.tsx top level)
import * as TaskManager from 'expo-task-manager';
import * as BackgroundTask from 'expo-background-task';
import { buildHeadlessSession } from './session';  // registry → tokens → db → api → engine, no React

export const SYNC_TASK = 'app.sync';

// MUST run at module top level so the OS can wake a fresh JS context into it.
TaskManager.defineTask(SYNC_TASK, async () => {
  try {
    for (const s of await buildHeadlessSession.allAccounts()) {    // one per signed-in account
      if (s.needsReauth) continue;
      await s.outbox.recoverInflight();
      await s.engine.sync({ budgetMs: 20_000 });                    // iOS gives ~30 s total
    }
    return BackgroundTask.BackgroundTaskResult.Success;
  } catch {
    return BackgroundTask.BackgroundTaskResult.Failed;
  }
});

export async function registerBackgroundSync() {
  if ((await BackgroundTask.getStatusAsync()) !== BackgroundTask.BackgroundTaskStatus.Available) return;  // restricted
  if (!(await TaskManager.isTaskRegisteredAsync(SYNC_TASK)))
    await BackgroundTask.registerTaskAsync(SYNC_TASK, { minimumInterval: 15 });  // minutes
}

// attachForegroundTriggers(engine): 1 s-debounced engine.sync() on app start, on AppState 'active',
// and on NetInfo offline → online transitions; returns an unsubscribe for the active session's lifetime.

// onSyncPush(data): for a data push { type: 'sync', accountId }, build that account's headless session
// and run engine.sync({ budgetMs: 20_000 }) — see react-native-push-notifications for handler setup.
```

Bare RN alternative: **react-native-background-fetch** — `BackgroundFetch.configure({ minimumFetchInterval: 15, stopOnTerminate: false, startOnBoot: true, enableHeadless: true }, onEvent, onTimeout)`; always call `BackgroundFetch.finish(taskId)` in both callbacks, and register the Android headless task in `index.js` (verify options against the current version).
