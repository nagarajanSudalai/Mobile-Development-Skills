---
name: android-list-ux
description: Use when implementing, reviewing, testing, or debugging list screens on Android — Jetpack Compose LazyColumn/LazyVerticalGrid and RecyclerView/ListAdapter/DiffUtil — batch loading with Paging 3 (PagingSource, RemoteMediator, LoadState), load-more footers, shimmer skeletons vs progress indicators, swipe actions (SwipeToDismissBox, ItemTouchHelper), pull to refresh (PullToRefreshBox, SwipeRefreshLayout), empty/error/offline states with Room, thumbnail caching with Coil/Glide, optimistic mutations, search/filter, selection, sticky headers, scroll restoration, performance (keys, contentType, stability, baseline profiles) and TalkBack accessibility. Siblings: ios-list-ux, flutter-list-ux, react-native-list-ux.
---

# Android List UX Skill

Guide for building, auditing, and testing **list screens** on Android: feeds, inboxes, search results, catalogs, settings-like secondary lists. Covers **Jetpack Compose** (`LazyColumn`, `LazyVerticalGrid`, `LazyPagingItems`) and the **View system** (`RecyclerView`, `ListAdapter`, `PagingDataAdapter`, `DiffUtil`), with **Paging 3** as the default loading engine, **Room** as the offline cache and **Coil / Glide** for thumbnails.

Sibling skills: **ios-list-ux**, **flutter-list-ux**, **react-native-list-ux** — same structure per platform. Keep behaviour consistent across platforms when the product ships on several.

> Compose, Material3, Paging and Coil move fast. Several APIs below were experimental or renamed recently (`PullToRefreshBox`, `SwipeToDismissBox`, `Modifier.animateItem`, Coil 3 package names, `stickyHeader`). **Verify against the versions in the project's `libs.versions.toml` / BOM and the current docs before writing code**, and prefer what the project already uses.

## 0. Mental model

```
             UI (Compose / RecyclerView)
   +-------------------------------------------------+
   | state: refresh LoadState | items | append state |
   |  first load  -> shimmer skeleton (primary)       |
   |               -> spinner (secondary)             |
   |  loaded      -> rows (+ swipe, select, refresh)  |
   |  appending   -> footer spinner / error+retry     |
   |  end         -> "You're all caught up" (opt.)    |
   |  empty/error/offline -> full-screen state        |
   +----------------------^--------------------------+
                          | Flow<PagingData<UiItem>>.cachedIn(viewModelScope)
   +----------------------+--------------------------+
   | ViewModel: query/filter StateFlow -> flatMapLatest -> Pager
   +----------------------^--------------------------+
                          |
   +----------------------+--------------------------+
   | Repository: Pager(config, remoteMediator) { dao.pagingSource() }
   |   Room = single source of truth   <- RemoteMediator <- API (cursor)
   +-------------------------------------------------+
```

Three load phases you must always distinguish — most list bugs are confusing them:

| Phase | Paging 3 signal | UI | Never |
|---|---|---|---|
| **Initial / refresh with no data** | `loadState.refresh is Loading && itemCount == 0` | Skeleton (primary) or centered spinner (secondary) | Blank screen; spinner + skeleton together |
| **Refresh with data** (pull, invalidate) | `loadState.refresh is Loading && itemCount > 0` | Keep rows; pull indicator only | Replace rows with skeleton; jump to top |
| **Append** (next page) | `loadState.append is Loading` | Footer `CircularProgressIndicator` | Full-screen loader; shimmer in footer |
| **Append error** | `loadState.append is Error` | Footer "Couldn't load more · Retry" | Snackbar loop; silently stop |
| **End reached** | `loadState.append.endOfPaginationReached` | Nothing or subtle end footer | Infinite spinner |

## 1. First, establish context

Find out (from the codebase or by asking):

- **Primary or secondary list?** Primary = the main content of a top-level destination (feed, inbox, catalog) → shimmer skeleton on first load. Secondary = a picker, settings sub-list, comments sheet, "related items" → simple `CircularProgressIndicator`.
- **UI toolkit**: Compose-only, Views-only, or mixed (`ComposeView` inside `RecyclerView` items or vice versa). Compose BOM / Material3 version; `minSdk`.
- **Paging contract** from the API: cursor (`next_cursor`), keyset (`after_id`), offset/page number, or "everything at once". Is the order stable? Can items be inserted at the top while paging?
- **Volume**: tens (no paging needed), hundreds (one request + local filtering may be fine), thousands+/unbounded (Paging 3 mandatory).
- **Item shape**: fixed vs variable height, images/video, multiple view types, nested horizontal carousels.
- **Realtime**: WebSocket/FCM/polling inserts, updates and deletes while the user is scrolled down?
- **Offline**: must the list work offline (Room + `RemoteMediator`), or is an error state enough? Staleness tolerance (cache TTL)?
- **Mutations**: swipe to delete/archive, reorder, multi-select bulk actions, inline toggles (like/star) — optimistic or server-confirmed?
- **Search / filter / sort**: server-side or local? Debounced? Do filters persist across process death?
- **Navigation**: list → detail → back must restore scroll position; bottom-nav reselect scrolls to top?
- **Existing libraries**: Paging 3? Coil 2 vs Coil 3, Glide? Hilt/Koin? Accompanist leftovers (placeholder, swiperefresh — both deprecated)?

Write a **list contract** before coding:

```
screen        role       source            page     key          loader     swipe            refresh  offline
Inbox         primary    Room+Mediator     cursor   message.id   shimmer    archive/delete   pull     cached+banner
Pick country  secondary  static/asset      none     iso code     spinner    —                —        n/a
Comments      secondary  API (Paging)      cursor   comment.id   spinner    delete own       —        error state
```

## 2. Batch data loading

### 2.1 Choosing the engine
- **Paging 3** (`androidx.paging:paging-runtime`, `paging-compose`) for anything unbounded. Use `paging-compose` 3.2+ for `itemKey`/`itemContentType` helpers (older `items(lazyPagingItems)` extension is removed).
- **Network only**: `PagingSource<Key, Value>` with `load()` and `getRefreshKey()`.
- **Offline-first**: Room DAO returns `PagingSource<Int, Entity>`; `RemoteMediator` fetches network pages and writes to Room in a transaction. Room is the single source of truth — the UI never reads the network response directly.
- **Small lists (< ~200, cheap)**: plain `StateFlow<List<T>>`; Paging adds complexity without value.

### 2.2 Checklist
- [ ] `PagingConfig(pageSize = N)` sized to **2–3 screens** of items (typically 20–50). `initialLoadSize` defaults to `3 * pageSize` — fine for network; reduce if first request is slow.
- [ ] `prefetchDistance` (default = `pageSize`) so the next page starts loading well before the end; never 0.
- [ ] `enablePlaceholders = false` unless the source knows total count and you render real placeholders (Room can; most APIs can't).
- [ ] `maxSize` set for very long feeds with heavy items (≥ `pageSize + 2 * prefetchDistance`) to drop far pages.
- [ ] **Cursor/keyset pagination preferred** over offset: offset + inserts at the top = duplicates/skips. If stuck with offset, dedupe by id and expect drift.
- [ ] `getRefreshKey()` implemented using `state.anchorPosition` → `closestPageToPosition` so invalidation reloads near where the user is (not from the top).
- [ ] **Stable unique keys** from the domain id (`item.id`), never index. Duplicate keys crash `LazyColumn` (`IllegalArgumentException: Key ... was already used`).
- [ ] **Dedupe** across page boundaries (server returns the same item on two pages when data shifts): `distinctBy` within a page plus `@Insert(onConflict = REPLACE)` in Room; for network-only, filter seen ids in the transform or fix the API to keyset.
- [ ] `RemoteMediator.load()` writes items **and** the remote keys/next cursor in a single `withTransaction`; on `REFRESH` it clears old rows in the same transaction (no empty flash).
- [ ] `RemoteMediator.initialize()` returns `SKIP_INITIAL_REFRESH` when the cache is fresh (TTL), `LAUNCH_INITIAL_REFRESH` otherwise.
- [ ] Errors returned as `LoadResult.Error` / `MediatorResult.Error` — **never throw** out of `load()` except `CancellationException`. Don't catch `CancellationException` in a generic `catch (e: Exception)`; rethrow it.
- [ ] **Cancellation**: Paging cancels loads automatically when the collector leaves; use `suspend` Retrofit/Ktor calls (cancellable). Don't launch unscoped work (`GlobalScope`) inside `load()`.
- [ ] `flow.cachedIn(viewModelScope)` in the ViewModel, **after** `map`/`filter` transforms that should survive config changes and **before** combining with other UI state. Never call `cachedIn` in a composable or create a new `Pager` per recomposition.
- [ ] Transforms (`map { it.toUi() }`, `insertSeparators`) are done on `PagingData`, not by collecting to a list.
- [ ] Data mapping (`Entity → UiModel`) happens off the main thread (Paging transforms run on a background dispatcher by default) and produces immutable models.

### 2.3 Cursor vs offset

| | Cursor / keyset | Offset / page |
|---|---|---|
| Inserts at top while paging | Safe | Duplicates on next page |
| Deletes while paging | Safe | Skipped items |
| Jump to page N | No | Yes |
| `getRefreshKey` | Store cursor per page (remote keys table) | `anchorPosition / pageSize` |

## 3. Load more (append)

### 3.1 Paging 3 footer
- [ ] Footer driven by `lazyPagingItems.loadState.append` (Compose) or `adapter.withLoadStateFooter(LoadStateAdapter)` (RecyclerView).
  - `Loading` → small centered `CircularProgressIndicator` (24–32dp) in a ~56–72dp footer row.
  - `Error` → message + **Retry** button calling `lazyPagingItems.retry()` / `adapter.retry()`.
  - `NotLoading(endOfPaginationReached = true)` → nothing, or an unobtrusive "You're all caught up" for feeds.
- [ ] With `RemoteMediator`, check **`loadState.mediator?.append`** and `loadState.source.append` — `loadState.append` is the combined convenience; make sure end-of-pagination comes from the mediator when the DB is a cache.
- [ ] Footer has its own `key` and `contentType` so it isn't confused with item rows.
- [ ] Append errors don't trigger a Snackbar on every recomposition (collect state, don't side-effect in composition).
- [ ] **Manual "Load more" button** option for secondary lists, footers inside bottom sheets, or where infinite scroll harms UX (search results with a footer below, accessibility preference). Paging supports this by keeping `prefetchDistance` small and triggering via a key you control, but it's simpler with a manual implementation (3.2).

### 3.2 Manual (non-Paging) implementation
Use when the API is weird, the list is a small secondary one, or Paging is not in the project.

- [ ] Trigger by **threshold**: load when `lastVisibleItemIndex >= totalItems - threshold` (threshold ≈ half a page), observed with `snapshotFlow { listState.layoutInfo }` + `distinctUntilChanged()` — not inside composition.
- [ ] **Guards**: `if (isLoadingMore || endReached || lastError != null) return`. Without them, scroll events fire N parallel requests for the same page.
- [ ] Single in-flight `Job`; cancel/ignore stale results when the query changes (generation counter or `flatMapLatest`).
- [ ] Append by **id-deduped merge**, never by blind `list + newPage`.
- [ ] After an append error, auto-retry only on explicit user action (Retry button) or connectivity regain — don't loop.
- [ ] If the first page doesn't fill the viewport, the threshold check must re-run after layout (otherwise no scroll = no trigger). `snapshotFlow` on `layoutInfo` handles this.
- [ ] RecyclerView: `addOnScrollListener` using `LinearLayoutManager.findLastVisibleItemPosition()`; same guards.

## 4. Loaders: skeleton vs spinner

### 4.1 Rules
- [ ] **Primary list, first load** (`refresh is Loading` and **no items**): **shimmer skeleton rows that match the real row layout** (avatar circle, two text lines of realistic widths, thumbnail box), enough rows to fill the viewport (`ceil(viewportHeight / rowHeight)`, typically 6–10), same paddings/heights as real rows so there is no layout jump when content arrives.
- [ ] **Secondary list, first load**: centered `CircularProgressIndicator` (or `LinearProgressIndicator` under the top bar). No skeleton.
- [ ] **Load more (any list)**: footer `CircularProgressIndicator`. Never shimmer in the footer.
- [ ] **Never shimmer over existing content**: refresh with items → keep items, show pull indicator / top `LinearProgressIndicator` only. Filter change with old results → keep old results dimmed or show a thin progress bar, then swap.
- [ ] **Flicker avoidance**: don't show any loader if the load finishes within ~150–300ms (cache hit); once shown, keep it at least ~400–500ms so it doesn't flash. Implement with a `delay` before showing and a `minDisplay` before hiding (see A.3 `rememberDelayedLoading`).
- [ ] Skeleton → content transition: `Crossfade`/`AnimatedContent` short fade (≤ 200ms), or none if reduced motion.
- [ ] Skeleton is **not focusable** by TalkBack individually; the container announces "Loading" once (`semantics { contentDescription = "Loading inbox"; liveRegion = Polite }`) — or use `progressBarRangeInfo` indeterminate.
- [ ] **Reduced motion**: when animator duration scale is 0 (`Settings.Global.ANIMATOR_DURATION_SCALE`) or the user disabled animations, render static grey placeholders without the moving gradient.

### 4.2 Compose shimmer
- Accompanist `placeholder` is **deprecated**; write a small `Modifier.shimmer()` (A.3) using `rememberInfiniteTransition` + `Brush.linearGradient`, or use a maintained lib (e.g. `com.valentinilk.shimmer:compose-shimmer`) if already in the project.
- Use `Modifier.drawWithCache`/`drawBehind` with the animated offset read **in the draw phase** (lambda) so the shimmer doesn't recompose every frame.
- Colours from `MaterialTheme.colorScheme.surfaceVariant` / `surfaceContainerHighest` — works in dark mode.

### 4.3 RecyclerView
- Facebook `ShimmerFrameLayout` (archived but still works) or a skeleton adapter with static placeholder rows; swap adapters or use `ConcatAdapter` with a skeleton adapter shown only while `refresh` is loading and `itemCount == 0`.

## 5. Swipe actions

### 5.1 Compose (Material3 `SwipeToDismissBox`)
- [ ] Available in Material3 1.2+ (replaces `SwipeToDismiss`). The state API changed across versions (`confirmValueChange` → `onDismiss` callback / `positionalThreshold` handling) — **check the project's Material3 version**.
- [ ] `rememberSwipeToDismissBoxState` keyed per item (state lives inside `items(key = …)` so it doesn't leak to the recycled slot); reset (`state.reset()` / `snapTo(Settled)`) if the action is cancelled or undone.
- [ ] Background shows icon + label that **changes colour when past the threshold**; direction-specific (`enableDismissFromStartToEnd`/`EndToStart`).
- [ ] **Destructive** (delete/archive): remove optimistically, show `Snackbar` with **Undo** (`SnackbarDuration.Long`, `actionLabel = "Undo"`); commit the server call after the snackbar dismisses or immediately with a compensating restore on undo. Never use a confirm dialog *and* undo for the same action.
- [ ] **Non-destructive reveal actions** (mark read, pin): either snap back after the action or use a custom `AnchoredDraggable` row with a revealed button tray (2–3 actions max).
- [ ] **Haptics**: `LocalHapticFeedback.current.performHapticFeedback(HapticFeedbackType.LongPress)` (or `SegmentTick`/`GestureThresholdActivate` on newer Compose) once when crossing the threshold, not every frame.
- [ ] Swipe doesn't fight horizontal carousels or the system back gesture: keep swipe rows away from screen edges' gesture insets or rely on threshold distance.
- [ ] **Accessibility**: swipe is not discoverable for TalkBack/Switch Access → add `Modifier.semantics { customActions = listOf(CustomAccessibilityAction("Archive") { … ; true }) }`, plus a visible alternative (long-press `DropdownMenu`, overflow icon, or selection mode bulk action).

### 5.2 RecyclerView (`ItemTouchHelper`)
- [ ] `ItemTouchHelper.SimpleCallback(0, LEFT or RIGHT)`; `onSwiped` → ViewModel action with the item **id** (from `getItem(bindingAdapterPosition)`), never the raw position after async work. Check `position != RecyclerView.NO_POSITION`.
- [ ] Draw background/icon in `onChildDraw`; `getSwipeThreshold` tuned (default 0.5).
- [ ] On undo, `ListAdapter.submitList` with the item restored — don't `notifyItemChanged` hacks; if the swipe was cancelled, `notifyItemChanged(pos)` to reset the swiped view.
- [ ] `ViewCompat.addAccessibilityAction(view, "Archive") { … }` on each item view for TalkBack.
- [ ] Drag-to-reorder: `ItemTouchHelper` `onMove` + `startDrag` from a handle; Compose: `sh.calvin.reorderable` or similar (no first-party API yet — verify).

## 6. Pull to refresh

- [ ] Compose: Material3 **`PullToRefreshBox`** (Material3 1.3+; earlier `PullToRefreshContainer`/`pullRefresh` from material 1.x — APIs differ). Accompanist `swiperefresh` is deprecated → migrate.
- [ ] Views: `SwipeRefreshLayout` (`androidx.swiperefreshlayout`) with `setOnRefreshListener`; `isRefreshing = false` on every completion path including error.
- [ ] `isRefreshing` tied to **user-initiated** refresh: `loadState.refresh is Loading && userRequestedRefresh` (or `itemCount > 0`) — otherwise the indicator also appears on first load on top of the skeleton.
- [ ] Refresh **keeps existing data** visible; replaces it atomically when new data arrives (Paging does this; RemoteMediator must clear + insert in one transaction).
- [ ] Refresh failure with data → Snackbar "Couldn't refresh" (keep stale rows), not a full-screen error.
- [ ] **Refresh vs append race**: Paging serializes this (refresh cancels in-flight append). In manual implementations, a refresh must cancel the append job and bump a generation id so the late append result is dropped.
- [ ] After refresh, scroll position: Paging keeps position via `getRefreshKey`; for "new items at top" feeds, decide: stay put + show "New posts" pill (14) or scroll to top for user-initiated pulls.
- [ ] Nested scroll: `PullToRefreshBox` wraps the `LazyColumn` directly; a `Column(verticalScroll)` parent breaks it. With a collapsing `TopAppBar`, pass the scroll behavior's `nestedScrollConnection` to the outer scaffold.
- [ ] Accessibility: pull is a gesture → provide a Refresh action (toolbar menu item or `customActions`) for TalkBack/keyboard users.

## 7. Empty, error and offline states

- [ ] **Empty** (`refresh is NotLoading && itemCount == 0`): illustration/icon + title + explanation + primary action ("Create your first note"). Distinguish "no data yet" vs "no results for this filter" (offer "Clear filters").
- [ ] **Initial error** (`refresh is Error && itemCount == 0`): full-screen message mapped from the exception (offline / server / auth), Retry button → `retry()` or `refresh()`.
- [ ] **Error with cached data**: keep the rows; show a Snackbar or inline banner ("Showing saved items · Retry").
- [ ] Don't show "empty" while the first load is still running (check order: loading → error → empty → content). With RemoteMediator, the DB is empty until the mediator finishes — gate the empty state on `loadState.mediator?.refresh is NotLoading`.
- [ ] **Offline-first with Room**: render cache immediately; banner "You're offline — showing saved items" with last-updated time; mutations queued (WorkManager) and marked pending in the row.
- [ ] **Connectivity**: `ConnectivityManager.registerDefaultNetworkCallback` exposed as a `Flow<Boolean>` (`callbackFlow`, `awaitClose { unregister }`) checking `NET_CAPABILITY_VALIDATED`; on regain → auto `retry()` if the last load failed (A.5). Don't poll `activeNetworkInfo` (deprecated).
- [ ] Auth errors (401) route to sign-in instead of a generic Retry loop.
- [ ] Error text is human ("Can't reach the server") not `HttpException 503`.
- [ ] Full-screen states are scrollable (pull to refresh still works on an empty list: put the empty state inside the `LazyColumn` as an `item` with `fillParentMaxSize()`).

## 8. Thumbnail caches (Coil / Glide)

### 8.1 Library choice
- **Compose**: **Coil** (Coil 3 = `io.coil-kt.coil3:coil-compose` + `coil-network-okhttp`/`ktor`; package `coil3.*`. Coil 2 = `io.coil-kt:coil-compose`, package `coil.*`). Don't mix versions. Glide has `glide-compose` (still beta-ish — verify).
- **Views**: Glide or Coil both fine; follow the project.

### 8.2 Checklist
- [ ] **One singleton `ImageLoader`** (Coil: `SingletonImageLoader.Factory` on `Application` in Coil 3 / `ImageLoaderFactory` in Coil 2) — never `ImageLoader(context)` per composable.
- [ ] **Memory cache** ~15–25% of app memory (`MemoryCache.Builder().maxSizePercent(context, 0.25)`); **disk cache** sized (e.g. 100–250MB) in `cacheDir/image_cache`. Glide: `AppGlideModule` with `LruResourceCache` / `InternalCacheDiskCacheFactory`.
- [ ] **Decode at the displayed size**: Coil sizes from the composable constraints automatically when `Modifier.size` is fixed; with `wrapContent`/unbounded it may decode full size → always give thumbnails fixed dimensions. Glide: `override(w, h)` or rely on `ImageView` size; never `Target.SIZE_ORIGINAL` in lists.
- [ ] Ask the **server/CDN for a thumbnail URL** sized for the row (e.g. `?w=160`) — decoding a 4000px photo for a 56dp avatar wastes bandwidth and CPU.
- [ ] **Stable cache keys**: signed URLs that change every request (`?token=…&expires=…`) defeat caching → set `memoryCacheKey`/`diskCacheKey` to the stable asset id (Coil) or a custom `GlideUrl.getCacheKey()` override.
- [ ] **Placeholders** sized like the image (no layout jump), neutral `surfaceVariant`; **error** drawable; **crossfade** short (`crossfade(150)`), disabled when reduced motion. Don't crossfade from memory-cache hits (Coil skips by default).
- [ ] **Cancel on recycle**: Compose `AsyncImage` cancels when leaving composition automatically. RecyclerView: Glide `Glide.with(fragment).clear(imageView)` in `onViewRecycled`; Coil `imageView.dispose()`. Use `Glide.with(fragment/view)`, not `with(applicationContext)` (lifecycle-aware cancel).
- [ ] **Preloading**: RecyclerView → Glide `RecyclerViewPreloader` + `ListPreloader.PreloadModelProvider` + `ViewPreloadSizeProvider`. Compose → no built-in; enqueue `imageLoader.enqueue(ImageRequest…size(px).build())` for the next N items from a `snapshotFlow` on `firstVisibleItemIndex`, same size and key as the displayed request so it hits the cache.
- [ ] **Bitmap config**: hardware bitmaps (API 26+, default in Coil/Glide) save memory and are fast to draw, but can't be read pixel-wise (Palette, `Canvas` software draw, shared-element snapshots) → `allowHardware(false)` only on those requests. `RGB_565` for opaque thumbnails if memory-bound (Glide `DecodeFormat.PREFER_RGB_565`).
- [ ] **Memory pressure**: Coil/Glide register `ComponentCallbacks2` themselves; if you have custom caches, implement `onTrimMemory` (`TRIM_MEMORY_UI_HIDDEN` → trim, `≥ BACKGROUND` → clear). Check with `adb shell am send-trim-memory <pkg> RUNNING_CRITICAL`.
- [ ] **Video thumbnails**: Coil `coil-video` (`VideoFrameDecoder`, `videoFrameMillis(…)`) or Glide's built-in video frame for local URIs; for remote video, use a server-generated poster image — extracting frames from network video is slow. Autoplay only the most-visible item (ExoPlayer/Media3 single player instance reused), pause off-screen.
- [ ] `contentDescription` meaningful for informative images, `null` for decorative ones.
- [ ] OkHttp shared between Retrofit and the image loader (connection pool, interceptors for auth headers on private images).

## 9. List mutations

- [ ] **Optimistic updates**: apply to local state (Room row / in-memory overlay) immediately, send the request, **roll back** on failure with a Snackbar ("Couldn't archive · Retry"). Keep a pending flag for UI (dim / spinner icon).
- [ ] With Paging + Room: mutate the **database**; the `PagingSource` invalidates and the UI updates. Don't try to mutate `PagingData` directly. Network-only Paging: keep a `MutableStateFlow<Map<Id, Override>>` and `combine` it with the paging flow via `pagingData.map { applyOverride(it) }` / `filter { it.id !in deleted }`.
- [ ] RecyclerView: `ListAdapter` + `DiffUtil.ItemCallback` with `areItemsTheSame` = id equality, `areContentsTheSame` = data class equality; `getChangePayload` for partial rebinds (like count) to avoid full-row flash. Never `notifyDataSetChanged()` for incremental changes.
- [ ] `submitList` with a **new list instance** (submitting the same mutated `MutableList` does nothing).
- [ ] Compose: `Modifier.animateItem()` (foundation 1.7+; older `animateItemPlacement()`) on rows for insert/remove/move animations — requires keys. Disable or shorten under reduced motion.
- [ ] **Realtime inserts without jump**: if the user is at the top (`firstVisibleItemIndex == 0`), insert and keep at top; if scrolled down, keep the anchor item in place (Compose keeps position by key automatically; RecyclerView with stable ids/DiffUtil does too) and show a "N new items" pill (14). Beware: Compose `LazyColumn` keeps the *first visible key* — inserting above it at index 0 when the user is at index 0 may scroll them; decide intentionally (`requestScrollToItem` in foundation 1.7+ to pin).
- [ ] Realtime updates are **coalesced** (batch within ~100–300ms) to avoid 60 diff passes per second.
- [ ] Deletes from other devices remove rows and close the detail screen if it was open.

## 10. Performance

### 10.1 Compose
- [ ] `items(items, key = { it.id }, contentType = { it.type })` — keys for state/animation correctness, `contentType` so slots are reused across same-type rows. Paging: `items(count = lazy.itemCount, key = lazy.itemKey { it.id }, contentType = lazy.itemContentType { it.type })`.
- [ ] Row UI models are **stable**: `@Immutable data class` with `kotlinx.collections.immutable` `ImmutableList` for nested lists, or rely on **strong skipping mode** (default in Kotlin 2.0.20+ Compose compiler) — check the compiler reports (`composeCompiler { reportsDestination = … }`) for unstable params.
- [ ] Lambdas passed to rows are stable: `onClick = { viewModel.open(item.id) }` is fine with strong skipping; avoid capturing whole unstable objects. Pass ids, not entities.
- [ ] No heavy work in composition: date formatting, `Html.fromHtml`, sorting/filtering → precompute in the ViewModel/mapper.
- [ ] Read fast-changing state (`listState.firstVisibleItemIndex`, scroll offset) via `derivedStateOf` or in `snapshotFlow`, never directly in a parent composable (recomposes everything on scroll).
- [ ] Don't nest a vertically scrollable lazy list in `Column(Modifier.verticalScroll)` (crash: infinite height constraints) — use a single `LazyColumn` with `item {}` sections. Horizontal carousels inside rows: `LazyRow` with keys; consider `LazyListPrefetchStrategy`/`nestedPrefetchItemCount` (foundation 1.7+, experimental).
- [ ] Fixed or estimated row heights (avoid layout passes measuring images): give images fixed sizes.
- [ ] **Baseline Profiles** (`androidx.baselineprofile` plugin + `ProfileInstaller`) generated with a journey that scrolls the main list; Compose performance in debug builds is not representative — **measure release** builds.
- [ ] **Macrobenchmark** `FrameTimingMetric` on a scroll journey; **JankStats** in production (`JankStats.createAndTrack(window) { frameData -> … }`) with `PerformanceMetricsState` tags like `"list:inbox"`.
- [ ] Layout Inspector recomposition counts while scrolling: rows should be skipped when only scroll changes.

### 10.2 RecyclerView
- [ ] `setHasFixedSize(true)` when the RecyclerView's size doesn't depend on adapter contents.
- [ ] `ListAdapter`/`AsyncListDiffer` (diff off the main thread); `setHasStableIds(true)` + `getItemId` only if ids are truly stable (and not with `ConcatAdapter` isolation conflicts).
- [ ] Shared `RecycledViewPool` for nested horizontal RecyclerViews of the same type; `setInitialPrefetchItemCount` on inner `LinearLayoutManager`s.
- [ ] Inflate once in `onCreateViewHolder`; `onBindViewHolder` does no allocations-heavy work, no `findViewById` (ViewBinding).
- [ ] `setItemViewCacheSize` only after measuring; avoid `wrap_content` RecyclerView inside `NestedScrollView` (disables recycling — every item inflated).
- [ ] `ComposeView` inside RecyclerView items: use `ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed` (default since 1.2+ handles pooling — verify) to avoid recreating compositions per bind.

## 11. Search, filter, sort

- [ ] Query in `MutableStateFlow<String>` (survive process death via `SavedStateHandle.getStateFlow("q", "")`).
- [ ] `debounce(300)` for typed text (not for filter chips — apply immediately), `distinctUntilChanged()`, then **`flatMapLatest`** into a new `Pager(...).flow` → previous search is cancelled.
- [ ] `cachedIn(viewModelScope)` **after** `flatMapLatest`.
- [ ] New query **resets paging** (new Pager) and **scrolls to top** (`LaunchedEffect(query) { listState.scrollToItem(0) }`), but keep old results visible with a thin progress indicator until the first page of new results arrives — no skeleton flash per keystroke.
- [ ] Empty query → show recent searches/suggestions, not a request for "".
- [ ] Minimum query length if the backend requires it; trim whitespace.
- [ ] Local filtering for small datasets with `combine(items, query)` on `Dispatchers.Default`.
- [ ] Sort/filter state shown visibly (chips with count) and reachable by TalkBack; "Clear all".
- [ ] Material3 `SearchBar`/`DockedSearchBar` (API changed in 1.3 — `inputField` slot; verify), IME action Search hides keyboard; `LazyColumn` scroll hides keyboard (`keyboardController?.hide()` on scroll start) when appropriate.

## 12. Selection & bulk actions

- [ ] Enter selection mode by **long-press** (`Modifier.combinedClickable(onLongClick = …)`) with haptic, or an explicit "Select" menu item.
- [ ] Selected ids in ViewModel as `Set<Id>` (survive rotation/process death via `SavedStateHandle` for small sets) — never store positions.
- [ ] Contextual top bar replaces the normal one: "3 selected", close (exits), Select all, bulk actions (archive, delete, share). Views: `ActionMode` (`startSupportActionMode`) or a custom toolbar.
- [ ] Back press exits selection mode first (`BackHandler(enabled = selecting)` / `OnBackPressedCallback`); supports predictive back.
- [ ] Tap toggles selection while in mode; row shows a checkbox/check overlay and `selected` semantics (`Modifier.toggleable` / `semantics { selected = true }`), TalkBack announces "selected".
- [ ] "Select all" with Paging: means "all loaded" or "all matching query" — decide and make it explicit; server bulk endpoints for the latter.
- [ ] Bulk destructive actions: single Undo snackbar for the whole batch; optimistic remove, rollback on failure (partial failure → report count).
- [ ] Selection survives paging/refresh (by id); items removed by refresh drop out of the set.

## 13. Sections, sticky headers, fast scroll

- [ ] Section headers via Paging `insertSeparators { before, after -> … }` producing a sealed `UiItem.Header | UiItem.Row` — headers get their own key (`"header-$date"`) and `contentType`.
- [ ] Compose `stickyHeader { }` (foundation; was `@ExperimentalFoundationApi` — check) for date/alpha groups; works with Paging only if headers are emitted as items — iterate `lazyPagingItems.itemSnapshotList`/peek carefully, or render headers as normal items (non-sticky) when mixing is awkward.
- [ ] Header background opaque (sticky header over content needs `surface` color) and headers marked `semantics { heading() }` for TalkBack navigation by headings.
- [ ] RecyclerView sticky headers: `ItemDecoration` drawing over the top or a library; section headers as separate view types.
- [ ] **Fast scroll** for long alphabetical lists: RecyclerView `app:fastScrollEnabled="true"` (needs drawables) or a custom index bar; Compose has no built-in scrollbar for `LazyColumn` (verify — `Modifier.scrollbar` is not stable) → custom index sidebar calling `listState.scrollToItem(indexOfSection)`. Requires known section positions (not compatible with unknown-length Paging).
- [ ] Grids: `LazyVerticalGrid` with `GridItemSpan(maxLineSpan)` for headers/footer.

## 14. Scroll behaviour

- [ ] `rememberLazyListState()` is saved with `rememberSaveable` internally — restores across rotation **if** the items are available synchronously on return. With Paging, `cachedIn(viewModelScope)` provides that; creating a new Pager on return resets the scroll to 0.
- [ ] List → detail → back keeps position: hoist `LazyListState` above the `NavHost` entry? No — keep it in the destination; Navigation Compose saves it via `SaveableStateHolder`. Bugs come from recreating the flow, not from the state.
- [ ] Process death: Paging restarts at `getRefreshKey` = null (top) unless you persist an anchor; acceptable for most feeds, document it.
- [ ] **Scroll to top on bottom-nav reselect** (`NavigationBarItem onClick` when already selected → `listState.animateScrollToItem(0)`; for very far positions, `scrollToItem(min(index, 30))` then animate to avoid a long animation).
- [ ] **"New items" pill**: when realtime/refresh adds items above and the user is not at top, show a floating chip "↑ 5 new posts"; tap → scroll to top + clear count. Visibility via `derivedStateOf { listState.firstVisibleItemIndex > 0 }`.
- [ ] Scroll-to-top FAB after N screens (`derivedStateOf`), hidden at top.
- [ ] Edge-to-edge (default on targetSdk 35): `contentPadding = WindowInsets.navigationBars.asPaddingValues()` (or Scaffold's `innerPadding`) so the last item and footer aren't under the nav bar; don't use `Modifier.padding` on the list (clips scrolling content).
- [ ] `TopAppBarScrollBehavior` (`enterAlwaysScrollBehavior`/`exitUntilCollapsed`) wired via `Modifier.nestedScroll(scrollBehavior.nestedScrollConnection)`.

## 15. Accessibility

- [ ] Each row is **one focusable unit** (`Modifier.semantics(mergeDescendants = true)` or `clickable` on the row) with a sensible reading order: title, subtitle, meta ("Alice, Lunch tomorrow?, 2 minutes ago, unread").
- [ ] `collectionInfo` / `collectionItemInfo`: `LazyColumn` provides them; for custom layouts add `Modifier.semantics { collectionInfo = CollectionInfo(rowCount, 1) }` so TalkBack says "item 3 of 20" (unknown count with Paging → `-1`).
- [ ] Swipe/long-press/drag actions exposed as **custom actions** (5.1); inline buttons have `contentDescription` and ≥ 48dp touch targets (`minimumInteractiveComponentSize`).
- [ ] Loading/empty/error states announced (`liveRegion = LiveRegionMode.Polite`); footer "Loading more" announced once, not every recomposition.
- [ ] **Font scale** up to 200% (Android 14 nonlinear scaling): rows grow in height — no fixed heights on text containers, no `maxLines = 1` on essential info without ellipsis; skeleton rows also scale. Test with `adb shell settings put system font_scale 2.0`.
- [ ] **Reduced motion**: read `Settings.Global.ANIMATOR_DURATION_SCALE == 0f` (Compose has no direct "reduce motion" API — verify; some versions expose `LocalReduceMotion` internally) → static skeleton, no crossfade, no `animateItem`.
- [ ] Colour contrast for meta text and swipe backgrounds; don't rely on colour alone for unread/selected (add bold/icon).
- [ ] Keyboard / D-pad (ChromeOS, TV, foldables with keyboards): focus moves row by row; Enter opens; selection works with keyboard.
- [ ] Headings for section headers (13).

---

## 16. Testing plan

1. **Unit**: `PagingSource` with `TestPager` (`androidx.paging:paging-testing` 3.2+) — refresh, append, error, `getRefreshKey`. `RemoteMediator` with in-memory Room. ViewModel flows with `asSnapshot { scrollTo(index) }` and Turbine.
2. **Compose UI tests**: `createComposeRule`, fake repository; assert skeleton shown on first load, footer states, empty/error states, swipe (`performTouchInput { swipeLeft() }`), custom actions (`performSemanticsAction`).
3. **Network conditions**: emulator network throttling (Extended controls → Cellular → EDGE/GPRS), airplane mode toggles, a proxy that injects 500s/timeouts on page 2.
4. **Process death**: `adb shell am kill <pkg>` while backgrounded (Don't keep activities developer option) → filters/selection restored.
5. **Performance**: Macrobenchmark scroll with `FrameTimingMetric` on a release build with baseline profile; JankStats in dogfood; GPU rendering profile bars.
6. **Memory**: Android Studio Memory Profiler while flinging an image-heavy list; `adb shell am send-trim-memory <pkg> RUNNING_LOW`.
7. **Accessibility**: TalkBack walk-through, Switch Access, Accessibility Scanner, font scale 2.0, animations off (`adb shell settings put global animator_duration_scale 0`), dark mode, RTL (`Force RTL layout direction`).

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| L1 | First open, primary list, slow network | Shimmer skeleton rows matching layout; no blank; no spinner |
| L2 | First open, secondary list | Centered `CircularProgressIndicator` |
| L3 | First open, cache hit < 200ms | No loader flash |
| L4 | Scroll to end | Footer spinner; next page appended; no duplicates |
| L5 | Page 2 fails | Footer error + Retry; Retry loads page 2 |
| L6 | End of data | Footer spinner gone; optional end marker |
| L7 | Fling fast through 5 pages | No parallel duplicate requests; no jank |
| L8 | Pull to refresh with data | Rows stay; indicator; atomic replace; position sane |
| L9 | Pull to refresh fails | Stale rows kept; Snackbar |
| L10 | First load fails, then network returns | Error state → auto retry → content |
| L11 | Offline with cache | Cached rows + offline banner + last updated |
| L12 | Offline, no cache | Offline state + Retry |
| L13 | Empty account / no results | Correct empty variant + action |
| L14 | Swipe delete → Undo | Row removed, Snackbar, restored in same place |
| L15 | Swipe delete, server fails | Row restored + error message |
| L16 | TalkBack on swipeable row | Custom actions available; row read as one item |
| L17 | Type query fast | Debounced; only latest results; list scrolled to top |
| L18 | Change filter while page 3 loading | Old load cancelled; new results only |
| L19 | Long-press → select 3 → Back | Exits selection; no navigation |
| L20 | Realtime insert while scrolled | No jump; "new items" pill |
| L21 | Detail → back | Same scroll position, no reload flash |
| L22 | Rotate / fold / resize | State and position preserved |
| L23 | Process death on page 4 | Restores sensibly (top or anchor), filters kept |
| L24 | Bottom-nav reselect | Scrolls to top |
| L25 | Image-heavy fling | Placeholders sized; no OOM; images correct (no wrong-image flash on recycle) |
| L26 | Signed image URLs rotate | Still served from cache |
| L27 | Font scale 200% | Rows expand; nothing clipped; skeleton scales |
| L28 | Animations off | Static skeleton; no crossfade/item animations |
| L29 | Last item with gesture nav / edge-to-edge | Not hidden under nav bar; footer visible |
| L30 | Dark mode | Skeleton, swipe backgrounds, placeholders readable |

## 17. Common bugs to look for in review

- Skeleton shown on **every** refresh/filter change (shimmer over existing content), or spinner + skeleton at the same time
- Pull indicator shown on first load because `isRefreshing = loadState.refresh is Loading`
- Empty state flashes before first data (checking `itemCount == 0` without checking `refresh` / mediator state)
- `Pager(...)` or `cachedIn` created in a composable or a getter (`val items get() = pager.flow`) → reload on every recomposition/rotation, scroll reset
- `cachedIn` before `flatMapLatest`, or missing → "collected twice" `IllegalStateException` or refetch on rotation
- Index used as key, or duplicate ids across pages → crash "Key was already used" / wrong state on swipe
- Offset pagination with realtime inserts → duplicates and skips
- `catch (e: Exception)` swallowing `CancellationException` in `load()`
- RemoteMediator clearing the table outside the insert transaction → empty flash on refresh
- Manual load-more without in-flight guard → same page requested many times; no trigger when first page is shorter than the screen
- `notifyDataSetChanged()` everywhere; `submitList` with the same mutated list instance
- `ItemTouchHelper` using stale `adapterPosition` after async work; swipe not reset after undo/cancel
- Swipe-only actions with no TalkBack/visible alternative
- `SwipeToDismissBox` state not keyed per item → dismissed state appears on another row
- Images decoded at full resolution (`wrapContent` image size, `SIZE_ORIGINAL`); new `ImageLoader` per row; signed URLs breaking cache
- Glide `with(applicationContext)` in adapters (no cancellation) → wrong image appears in recycled rows
- `allowHardware(false)` globally "to fix" one Palette call
- Reading `listState.firstVisibleItemIndex` directly in a big composable → whole screen recomposes on scroll
- `LazyColumn` inside `Column(verticalScroll)` → crash; `wrap_content` RecyclerView in `NestedScrollView` → no recycling
- Unstable row models (`List<T>`, `var` fields, `java.util.Date`) defeating skipping on older Compose compilers
- Performance judged in debug builds; no baseline profile
- `contentPadding` missing on edge-to-edge → last item under navigation bar
- Snackbar shown from composition (re-shown on recomposition) instead of `LaunchedEffect`/one-off event
- Search without debounce / without `flatMapLatest` → out-of-order results
- Selection stored as positions → wrong items deleted after refresh
- Deprecated Accompanist `swiperefresh`/`placeholder` still in use

## 18. Output format

When asked to **review/audit**:
1. **Summary** — list contract table (screen → role → source → paging type → key → loader → swipe → refresh → offline), toolkit and library versions (Compose BOM, Material3, Paging, Coil/Glide)
2. **Checklist results** — sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** (crashes from duplicate keys, data loss on swipe/undo, duplicate/missing items from pagination, infinite loading, wrong images, jank on the primary list, inaccessible actions) with fix code
4. **Testing gaps** — L1–L30 not covered, plus which ones need a device/release build

When asked to **implement**: fill in the list contract first, then start from the Appendix: copy the `ListState` model (manual) or the Paging ViewModel, the shimmer modifier and the screen; remove what the contract doesn't need (swipe, selection, sticky headers); match library versions to the project and verify the experimental APIs compile; finish with the testing plan items you ran and those left for the user.

---

## Appendix — Reference implementation (Kotlin, Compose, Paging 3, Coil 3)

> Written against Compose BOM 2025.x, Material3 1.3/1.4, foundation 1.7+, Paging 3.3, Coil 3.x. Signatures of `SwipeToDismissBox`, `PullToRefreshBox` and `animateItem` have changed between versions — adjust to the project's versions.

### A.1 `ListState` state machine (manual, non-Paging lists)

```kotlin
sealed interface ListState<out T> {
    data object Idle : ListState<Nothing>
    data object LoadingInitial : ListState<Nothing>
    data class Loaded<T>(val items: List<T>, val nextCursor: String?) : ListState<T>
    data class LoadingMore<T>(val items: List<T>, val nextCursor: String) : ListState<T>
    data class Error<T>(
        val items: List<T>,            // empty = initial error, non-empty = append error
        val nextCursor: String?,
        val cause: Throwable,
    ) : ListState<T>
    data class EndReached<T>(val items: List<T>) : ListState<T>
}

val <T> ListState<T>.items: List<T>
    get() = when (this) {
        is ListState.Loaded -> items
        is ListState.LoadingMore -> items
        is ListState.Error -> items
        is ListState.EndReached -> items
        else -> emptyList()
    }

interface Identified { val id: String }

class ManualListController<T : Identified>(
    private val scope: CoroutineScope,
    private val fetch: suspend (cursor: String?) -> Page<T>,
) {
    data class Page<T>(val items: List<T>, val nextCursor: String?)

    private val _state = MutableStateFlow<ListState<T>>(ListState.Idle)
    val state: StateFlow<ListState<T>> = _state.asStateFlow()
    private var job: Job? = null

    fun loadInitial() {
        job?.cancel()                                   // refresh beats append
        _state.value = ListState.LoadingInitial
        job = scope.launch { load(cursor = null, existing = emptyList()) }
    }

    fun loadMore() {
        val s = _state.value
        if (s !is ListState.Loaded || s.nextCursor == null) return  // guards: loading/error/end/idle
        if (job?.isActive == true) return
        _state.value = ListState.LoadingMore(s.items, s.nextCursor)
        job = scope.launch { load(s.nextCursor, s.items) }
    }

    fun retry() {
        val s = _state.value as? ListState.Error ?: return
        if (s.items.isEmpty()) loadInitial()
        else {
            _state.value = ListState.LoadingMore(s.items, s.nextCursor ?: return)
            job = scope.launch { load(s.nextCursor, s.items) }
        }
    }

    private suspend fun load(cursor: String?, existing: List<T>) {
        try {
            val page = fetch(cursor)
            val seen = existing.mapTo(HashSet()) { it.id }
            val merged = existing + page.items.filter { seen.add(it.id) }   // dedupe
            _state.value = if (page.nextCursor == null) ListState.EndReached(merged)
                           else ListState.Loaded(merged, page.nextCursor)
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            _state.value = ListState.Error(existing, cursor, e)
        }
    }
}

/** Threshold trigger: call from the screen. */
@Composable
fun LoadMoreEffect(listState: LazyListState, threshold: Int = 10, onLoadMore: () -> Unit) {
    val currentOnLoadMore by rememberUpdatedState(onLoadMore)
    LaunchedEffect(listState) {
        snapshotFlow {
            val info = listState.layoutInfo
            val last = info.visibleItemsInfo.lastOrNull()?.index ?: -1
            info.totalItemsCount > 0 && last >= info.totalItemsCount - 1 - threshold
        }.distinctUntilChanged().filter { it }.collect { currentOnLoadMore() }
    }
}
```

### A.2 Repository + ViewModel with Paging 3

```kotlin
@Entity(tableName = "messages")
data class MessageEntity(
    @PrimaryKey val id: String,
    val sender: String, val subject: String, val preview: String,
    val avatarId: String?, val avatarUrl: String?, val sentAt: Long,
    val sortKey: Long, val unread: Boolean, val pendingAction: String? = null,
)

@Entity(tableName = "remote_keys")
data class RemoteKey(@PrimaryKey val listId: String, val nextCursor: String?, val updatedAt: Long)

@Dao
interface MessageDao {
    @Query("SELECT * FROM messages WHERE pendingAction IS NULL OR pendingAction != 'deleted' ORDER BY sortKey DESC")
    fun pagingSource(): PagingSource<Int, MessageEntity>
    @Upsert suspend fun upsertAll(items: List<MessageEntity>)
    @Query("DELETE FROM messages") suspend fun clear()
    @Query("UPDATE messages SET pendingAction = :action WHERE id = :id")
    suspend fun setPending(id: String, action: String?)
    @Query("DELETE FROM messages WHERE id = :id") suspend fun delete(id: String)
}

@OptIn(ExperimentalPagingApi::class)
class InboxMediator(
    private val api: InboxApi, private val db: AppDatabase,
    private val ttlMs: Long = 15 * 60_000,
) : RemoteMediator<Int, MessageEntity>() {

    override suspend fun initialize(): InitializeAction {
        val key = db.remoteKeyDao().get("inbox")
        return if (key != null && System.currentTimeMillis() - key.updatedAt < ttlMs)
            InitializeAction.SKIP_INITIAL_REFRESH else InitializeAction.LAUNCH_INITIAL_REFRESH
    }

    override suspend fun load(loadType: LoadType, state: PagingState<Int, MessageEntity>): MediatorResult {
        val cursor = when (loadType) {
            LoadType.REFRESH -> null
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> db.remoteKeyDao().get("inbox")?.nextCursor
                ?: return MediatorResult.Success(endOfPaginationReached = true)
        }
        return try {
            val page = api.messages(cursor = cursor, limit = state.config.pageSize)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) db.messageDao().clear()   // same transaction: no empty flash
                db.messageDao().upsertAll(page.items.distinctBy { it.id }.map { it.toEntity() })
                db.remoteKeyDao().upsert(RemoteKey("inbox", page.nextCursor, System.currentTimeMillis()))
            }
            MediatorResult.Success(endOfPaginationReached = page.nextCursor == null)
        } catch (e: CancellationException) {
            throw e
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }
}

@Immutable
data class MessageUi(
    val id: String, val sender: String, val subject: String, val preview: String,
    val avatarKey: String?, val avatarUrl: String?, val time: String, val unread: Boolean,
)

class InboxRepository(private val api: InboxApi, private val db: AppDatabase) {
    @OptIn(ExperimentalPagingApi::class)
    fun inbox(query: String): Flow<PagingData<MessageEntity>> = Pager(
        config = PagingConfig(pageSize = 30, prefetchDistance = 15, enablePlaceholders = false),
        remoteMediator = if (query.isBlank()) InboxMediator(api, db) else null,
        pagingSourceFactory = {
            if (query.isBlank()) db.messageDao().pagingSource()
            else SearchPagingSource(api, query)                   // network-only search
        },
    ).flow

    suspend fun archive(id: String) {
        db.messageDao().setPending(id, "deleted")                 // optimistic: row disappears
        try { api.archive(id); db.messageDao().delete(id) }
        catch (e: Exception) { db.messageDao().setPending(id, null); throw e }  // rollback
    }
    suspend fun undoArchive(id: String) { api.unarchive(id); db.messageDao().setPending(id, null) }
}

@HiltViewModel
class InboxViewModel @Inject constructor(
    private val repo: InboxRepository,
    private val savedState: SavedStateHandle,
    connectivity: ConnectivityObserver,
    private val formatter: TimeFormatter,
) : ViewModel() {

    val query: StateFlow<String> = savedState.getStateFlow("q", "")
    fun onQueryChange(q: String) { savedState["q"] = q }

    @OptIn(FlowPreview::class, ExperimentalCoroutinesApi::class)
    val messages: Flow<PagingData<MessageUi>> = query
        .debounce { if (it.isEmpty()) 0 else 300 }
        .map { it.trim() }
        .distinctUntilChanged()
        .flatMapLatest { q -> repo.inbox(q) }
        .map { paging -> paging.map { it.toUi(formatter) } }
        .cachedIn(viewModelScope)                                 // after flatMapLatest + map

    val isOnline: StateFlow<Boolean> = connectivity.isOnline
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), true)

    private val _events = Channel<InboxEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun archive(id: String) = viewModelScope.launch {
        try { repo.archive(id); _events.send(InboxEvent.Archived(id)) }
        catch (e: CancellationException) { throw e }
        catch (e: Exception) { _events.send(InboxEvent.Failed("Couldn't archive")) }
    }
    fun undo(id: String) = viewModelScope.launch { runCatching { repo.undoArchive(id) } }
}

sealed interface InboxEvent {
    data class Archived(val id: String) : InboxEvent
    data class Failed(val message: String) : InboxEvent
}
```

### A.3 Shimmer modifier, delayed loading & skeleton row

```kotlin
fun Modifier.shimmer(enabled: Boolean = true): Modifier = composed {
    val base = MaterialTheme.colorScheme.surfaceVariant
    if (!enabled || rememberReducedMotion()) return@composed background(base)
    val highlight = MaterialTheme.colorScheme.surface.copy(alpha = 0.6f)
    val transition = rememberInfiniteTransition(label = "shimmer")
    val progress = transition.animateFloat(
        initialValue = 0f, targetValue = 1f,
        animationSpec = infiniteRepeatable(tween(1_200, easing = LinearEasing)),
        label = "shimmer-progress",
    )
    drawWithCache {
        onDrawBehind {
            val w = size.width
            val x = -w + progress.value * 2 * w                  // read in draw phase only
            drawRect(base)
            drawRect(Brush.linearGradient(
                colors = listOf(Color.Transparent, highlight, Color.Transparent),
                start = Offset(x, 0f), end = Offset(x + w, size.height),
            ))
        }
    }
}
// Note: `composed` is discouraged for hot paths; for a library-grade version implement
// Modifier.Node (DrawModifierNode) — verify current guidance.

@Composable
fun rememberReducedMotion(): Boolean {
    val context = LocalContext.current
    return remember {
        Settings.Global.getFloat(context.contentResolver, Settings.Global.ANIMATOR_DURATION_SCALE, 1f) == 0f
    }
}

/** Show loader only if loading lasts > [delayMs]; once shown keep it ≥ [minShowMs]. */
@Composable
fun rememberDelayedLoading(loading: Boolean, delayMs: Long = 200, minShowMs: Long = 500): Boolean {
    var visible by remember { mutableStateOf(false) }
    var shownAt by remember { mutableLongStateOf(0L) }
    LaunchedEffect(loading) {
        if (loading) {
            delay(delayMs); visible = true; shownAt = SystemClock.uptimeMillis()
        } else if (visible) {
            val remaining = minShowMs - (SystemClock.uptimeMillis() - shownAt)
            if (remaining > 0) delay(remaining)
            visible = false
        }
    }
    return visible
}

@Composable
fun MessageRowSkeleton() {
    Row(Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 12.dp),
        verticalAlignment = Alignment.CenterVertically) {
        Box(Modifier.size(48.dp).clip(CircleShape).shimmer())          // = avatar size
        Spacer(Modifier.width(16.dp))
        Column(Modifier.weight(1f)) {
            Box(Modifier.fillMaxWidth(0.4f).height(14.dp).clip(RoundedCornerShape(4.dp)).shimmer())
            Spacer(Modifier.height(8.dp))
            Box(Modifier.fillMaxWidth(0.85f).height(12.dp).clip(RoundedCornerShape(4.dp)).shimmer())
            Spacer(Modifier.height(6.dp))
            Box(Modifier.fillMaxWidth(0.6f).height(12.dp).clip(RoundedCornerShape(4.dp)).shimmer())
        }
    }
}

@Composable
fun InboxSkeleton(rows: Int = 8) {
    Column(Modifier
        .fillMaxSize()
        .clearAndSetSemantics { contentDescription = "Loading inbox"; liveRegion = LiveRegionMode.Polite }) {
        repeat(rows) { MessageRowSkeleton() }
    }
}
```

### A.4 Screen: LazyColumn + pull to refresh + swipe + footer

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun InboxScreen(vm: InboxViewModel = hiltViewModel(), onOpen: (String) -> Unit) {
    val items = vm.messages.collectAsLazyPagingItems()
    val isOnline by vm.isOnline.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()
    val snackbar = remember { SnackbarHostState() }
    var userRefreshing by remember { mutableStateOf(false) }

    val refresh = items.loadState.refresh
    val mediatorRefresh = items.loadState.mediator?.refresh
    val firstLoad = items.itemCount == 0 && (refresh is LoadState.Loading || mediatorRefresh is LoadState.Loading)
    val showSkeleton = rememberDelayedLoading(firstLoad)

    LaunchedEffect(refresh) { if (refresh !is LoadState.Loading) userRefreshing = false }

    // Refresh failed while showing data -> snackbar (one-off, not in composition)
    LaunchedEffect(refresh) {
        if (refresh is LoadState.Error && items.itemCount > 0)
            snackbar.showSnackbar("Couldn't refresh. Showing saved messages.")
    }
    // Auto-retry when connectivity returns
    LaunchedEffect(isOnline) {
        if (isOnline && (refresh is LoadState.Error || items.loadState.append is LoadState.Error)) items.retry()
    }
    LaunchedEffect(Unit) {
        vm.events.collect { e ->
            when (e) {
                is InboxEvent.Archived -> {
                    val r = snackbar.showSnackbar("Archived", actionLabel = "Undo", duration = SnackbarDuration.Long)
                    if (r == SnackbarResult.ActionPerformed) vm.undo(e.id)
                }
                is InboxEvent.Failed -> snackbar.showSnackbar(e.message)
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbar) }) { padding ->
        PullToRefreshBox(
            isRefreshing = userRefreshing,
            onRefresh = { userRefreshing = true; items.refresh() },
            modifier = Modifier.fillMaxSize().padding(top = padding.calculateTopPadding()),
        ) {
            when {
                showSkeleton -> InboxSkeleton()
                firstLoad -> Unit                                         // within delay window
                refresh is LoadState.Error && items.itemCount == 0 ->
                    FullScreenState(Icons.Outlined.CloudOff, refresh.error.toMessage(), "Retry") { items.retry() }
                else -> LazyColumn(
                    state = listState,
                    contentPadding = PaddingValues(bottom = padding.calculateBottomPadding()),
                    modifier = Modifier.fillMaxSize(),
                ) {
                    if (!isOnline && items.itemCount > 0) item(key = "offline", contentType = "banner") { OfflineBanner() }
                    if (items.itemCount == 0 && refresh is LoadState.NotLoading) {
                        item(key = "empty", contentType = "empty") {
                            Box(Modifier.fillParentMaxSize()) { EmptyInbox() }   // still pull-refreshable
                        }
                    }
                    items(
                        count = items.itemCount,
                        key = items.itemKey { it.id },
                        contentType = items.itemContentType { "message" },
                    ) { index ->
                        val msg = items[index] ?: return@items
                        SwipeableMessageRow(
                            msg = msg,
                            onArchive = { vm.archive(msg.id) },
                            onClick = { onOpen(msg.id) },
                            modifier = Modifier.animateItem(),
                        )
                    }
                    item(key = "footer", contentType = "footer") {
                        AppendFooter(items.loadState.append, onRetry = items::retry)
                    }
                }
            }
        }
    }
}

@Composable
fun AppendFooter(state: LoadState, onRetry: () -> Unit) {
    Box(Modifier.fillMaxWidth().heightIn(min = 64.dp).padding(16.dp), contentAlignment = Alignment.Center) {
        when (state) {
            is LoadState.Loading -> CircularProgressIndicator(
                Modifier.size(28.dp).semantics { contentDescription = "Loading more" }, strokeWidth = 3.dp)
            is LoadState.Error -> Row(verticalAlignment = Alignment.CenterVertically) {
                Text("Couldn't load more", style = MaterialTheme.typography.bodyMedium)
                Spacer(Modifier.width(12.dp))
                OutlinedButton(onClick = onRetry) { Text("Retry") }
            }
            is LoadState.NotLoading -> if (state.endOfPaginationReached) {
                Text("You're all caught up", style = MaterialTheme.typography.labelMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeableMessageRow(msg: MessageUi, onArchive: () -> Unit, onClick: () -> Unit, modifier: Modifier = Modifier) {
    val haptics = LocalHapticFeedback.current
    val currentOnArchive by rememberUpdatedState(onArchive)
    // Material3 1.3 API; in 1.4+ prefer the onDismiss callback on SwipeToDismissBox — verify.
    val state = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) { currentOnArchive(); true } else false
        },
    )
    LaunchedEffect(state) {
        snapshotFlow { state.targetValue }.distinctUntilChanged().drop(1).collect {
            haptics.performHapticFeedback(HapticFeedbackType.LongPress)   // threshold crossed
        }
    }
    SwipeToDismissBox(
        state = state,
        enableDismissFromStartToEnd = false,
        modifier = modifier,
        backgroundContent = {
            val active = state.targetValue == SwipeToDismissBoxValue.EndToStart
            val color by animateColorAsState(
                if (active) MaterialTheme.colorScheme.tertiaryContainer else MaterialTheme.colorScheme.surfaceVariant,
                label = "swipe-bg")
            Box(Modifier.fillMaxSize().background(color).padding(horizontal = 24.dp),
                contentAlignment = Alignment.CenterEnd) {
                Icon(Icons.Outlined.Archive, contentDescription = null)
            }
        },
    ) {
        MessageRow(
            msg = msg, onClick = onClick,
            modifier = Modifier.semantics {
                customActions = listOf(CustomAccessibilityAction("Archive") { currentOnArchive(); true })
            },
        )
    }
}

@Composable
fun MessageRow(msg: MessageUi, onClick: () -> Unit, modifier: Modifier = Modifier) {
    Row(
        modifier
            .fillMaxWidth()
            .background(MaterialTheme.colorScheme.surface)
            .clickable(onClick = onClick)                     // merges descendants into one TalkBack node
            .padding(horizontal = 16.dp, vertical = 12.dp),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data(msg.avatarUrl)
                .memoryCacheKey(msg.avatarKey)                // stable key despite signed URLs
                .diskCacheKey(msg.avatarKey)
                .crossfade(!rememberReducedMotion())
                .build(),
            contentDescription = null,                        // sender name is read from text
            placeholder = ColorPainter(MaterialTheme.colorScheme.surfaceVariant),
            error = rememberVectorPainter(Icons.Outlined.Person),
            contentScale = ContentScale.Crop,
            modifier = Modifier.size(48.dp).clip(CircleShape), // fixed size -> decoded at 48dp
        )
        Spacer(Modifier.width(16.dp))
        Column(Modifier.weight(1f)) {
            Row {
                Text(msg.sender, Modifier.weight(1f), maxLines = 1, overflow = TextOverflow.Ellipsis,
                    fontWeight = if (msg.unread) FontWeight.Bold else FontWeight.Normal)
                Text(msg.time, style = MaterialTheme.typography.labelSmall)
            }
            Text(msg.subject, maxLines = 1, overflow = TextOverflow.Ellipsis)
            Text(msg.preview, maxLines = 2, overflow = TextOverflow.Ellipsis,
                style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
        }
    }
}
```

### A.5 Connectivity observer

```kotlin
class ConnectivityObserver(context: Context) {
    private val cm = context.getSystemService(ConnectivityManager::class.java)
    val isOnline: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onCapabilitiesChanged(n: Network, caps: NetworkCapabilities) {
                trySend(caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED))
            }
            override fun onLost(n: Network) { trySend(false) }
        }
        val current = cm.getNetworkCapabilities(cm.activeNetwork)
        trySend(current?.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED) == true)
        cm.registerDefaultNetworkCallback(callback)
        awaitClose { cm.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged().conflate()
}
```

### A.6 Coil 3 configuration & Compose prefetch

```kotlin
class App : Application(), SingletonImageLoader.Factory {
    override fun newImageLoader(context: PlatformContext): ImageLoader =
        ImageLoader.Builder(context)
            .components {
                add(OkHttpNetworkFetcherFactory(callFactory = { sharedOkHttpClient }))
                add(VideoFrameDecoder.Factory())                         // coil-video, local video thumbs
            }
            .memoryCache { MemoryCache.Builder().maxSizePercent(context, 0.20).build() }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache").toOkioPath())
                    .maxSizeBytes(200L * 1024 * 1024)
                    .build()
            }
            .crossfade(150)
            .build()
}

/** Prefetch thumbnails for the next [ahead] rows at the exact displayed size. */
@Composable
fun PrefetchThumbnails(listState: LazyListState, items: LazyPagingItems<MessageUi>, ahead: Int = 10) {
    val context = LocalContext.current
    val px = with(LocalDensity.current) { 48.dp.roundToPx() }
    LaunchedEffect(listState, items) {
        snapshotFlow { listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0 }
            .distinctUntilChanged()
            .collect { last ->
                val loader = SingletonImageLoader.get(context)
                for (i in (last + 1)..minOf(last + ahead, items.itemCount - 1)) {
                    val m = items.peek(i) ?: continue                   // peek: doesn't trigger paging
                    loader.enqueue(ImageRequest.Builder(context).data(m.avatarUrl)
                        .memoryCacheKey(m.avatarKey).diskCacheKey(m.avatarKey).size(px, px).build())
                }
            }
    }
}
```

> Coil 2 equivalents: `ImageLoaderFactory`/`newImageLoader()`, `coil.*` packages, `ImageLoader(context)` builder with `okHttpClient {}`, `context.imageLoader`. Glide (Views): `AppGlideModule` + `RecyclerViewPreloader(Glide.with(this), modelProvider, ViewPreloadSizeProvider(), 10)` added as an `OnScrollListener`, and `Glide.with(holder.itemView).clear(holder.image)` in `onViewRecycled`.

### A.7 RecyclerView equivalent (Views)

```kotlin
class MessageAdapter(private val onClick: (String) -> Unit) :
    PagingDataAdapter<MessageUi, MessageAdapter.VH>(DIFF) {

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<MessageUi>() {
            override fun areItemsTheSame(a: MessageUi, b: MessageUi) = a.id == b.id
            override fun areContentsTheSame(a: MessageUi, b: MessageUi) = a == b
            override fun getChangePayload(a: MessageUi, b: MessageUi) =
                if (a.copy(unread = b.unread) == b) "unread" else null       // partial rebind
        }
    }

    class VH(val b: ItemMessageBinding) : RecyclerView.ViewHolder(b.root)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        VH(ItemMessageBinding.inflate(LayoutInflater.from(parent.context), parent, false)).also { vh ->
            vh.itemView.setOnClickListener {
                val pos = vh.bindingAdapterPosition
                if (pos != RecyclerView.NO_POSITION) getItem(pos)?.let { onClick(it.id) }
            }
        }

    override fun onBindViewHolder(holder: VH, position: Int, payloads: MutableList<Any>) {
        val m = getItem(position) ?: return
        if ("unread" in payloads) { holder.b.sender.setTypeface(null, if (m.unread) BOLD else NORMAL); return }
        onBindViewHolder(holder, position)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        val m = getItem(position) ?: return
        with(holder.b) {
            sender.text = m.sender; subject.text = m.subject; preview.text = m.preview; time.text = m.time
            sender.setTypeface(null, if (m.unread) BOLD else NORMAL)
            avatar.load(m.avatarUrl) { memoryCacheKey(m.avatarKey); diskCacheKey(m.avatarKey) }
        }
        ViewCompat.addAccessibilityAction(holder.itemView, "Archive") { _, _ -> /* vm.archive(m.id) */ true }
    }

    override fun onViewRecycled(holder: VH) { holder.b.avatar.dispose() }   // cancel in-flight load
}

// Fragment wiring
recycler.setHasFixedSize(true)
recycler.adapter = adapter.withLoadStateFooter(FooterLoadStateAdapter(adapter::retry))
swipeRefresh.setOnRefreshListener { adapter.refresh() }
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { vm.messages.collectLatest(adapter::submitData) }
        launch {
            adapter.loadStateFlow.collect { s ->
                val empty = adapter.itemCount == 0
                skeleton.isVisible = empty && s.refresh is LoadState.Loading
                swipeRefresh.isRefreshing = !empty && s.refresh is LoadState.Loading
                errorView.isVisible = empty && s.refresh is LoadState.Error
                emptyView.isVisible = empty && s.refresh is LoadState.NotLoading && s.append.endOfPaginationReached
            }
        }
    }
}
ItemTouchHelper(object : ItemTouchHelper.SimpleCallback(0, ItemTouchHelper.LEFT) {
    override fun onMove(rv: RecyclerView, vh: RecyclerView.ViewHolder, t: RecyclerView.ViewHolder) = false
    override fun onSwiped(vh: RecyclerView.ViewHolder, dir: Int) {
        val pos = vh.bindingAdapterPosition
        val item = if (pos != RecyclerView.NO_POSITION) adapter.peek(pos) else null
        if (item == null) { adapter.notifyItemChanged(pos); return }
        vm.archive(item.id)                                  // Room update -> PagingSource invalidates
    }
}).attachToRecyclerView(recycler)
```
