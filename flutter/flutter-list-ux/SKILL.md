---
name: flutter-list-ux
description: Use when implementing, reviewing, testing, or debugging list screens in Flutter — ListView.builder / CustomScrollView / slivers, batch (paged) data loading with cursor or offset, load-more and infinite scroll (ScrollController threshold or infinite_scroll_pagination PagingController), shimmer/skeleton first-load loaders vs CircularProgressIndicator, pull to refresh, swipe actions (Dismissible, flutter_slidable), empty/error/offline states, thumbnail image caching (cached_network_image, memCacheWidth, ImageCache), optimistic mutations, AnimatedList, search/filter/sort, selection, sticky headers, scroll restoration, accessibility and list performance. State-management agnostic (Riverpod, Bloc, Provider, ChangeNotifier). Siblings: ios-list-ux, android-list-ux, react-native-list-ux.
---

# Flutter List UX Skill

Guide for building, auditing, and testing **list screens** in Flutter: feeds, inboxes, catalogs, search results, settings-style lists inside detail screens. Covers **paged data loading**, **load more**, **loaders (shimmer vs spinner)**, **pull to refresh**, **swipe actions**, **empty / error / offline states**, **thumbnail caching**, **mutations**, **performance**, **search**, **selection**, **sections**, **scroll behaviour**, and **accessibility**. The rules are independent of state management — the reference implementation uses a plain `ChangeNotifier`, and maps 1:1 onto a Riverpod `Notifier`/`AsyncNotifier` or a Bloc/Cubit. Siblings: **ios-list-ux**, **android-list-ux**, **react-native-list-ux**.

Flutter and its packages move fast. Before shipping, **verify the current API** of: `infinite_scroll_pagination` (v5 replaced the controller-driven `addPageRequestListener` API with a state-driven `PagingState` + `fetchNextPage` API), `flutter_slidable` (v3+ API differs from v0.x/v1), `skeletonizer`, `shimmer`, `cached_network_image` / `flutter_cache_manager`, `connectivity_plus` (v6+ returns `List<ConnectivityResult>`), and framework APIs gated by Flutter version (`SliverMainAxisGroup` / `SliverCrossAxisGroup` 3.13+, `SliverList.builder` 3.7+, `CarouselView` 3.24+, `PopScope` 3.16+). Check `pubspec.yaml` and `flutter --version` and read the package changelog rather than trusting memory.

## 0. Mental model

```
UI (ListView / CustomScrollView)          Controller / Notifier / Bloc              Repository
+-------------------------------+   req   +------------------------------+   fetch +--------------------+
| header slivers                | ------> | ListState (sealed)           | ------> | API (cursor/offset)|
| RefreshIndicator              |         |  idle / loadingInitial /     |         | local cache (DB)   |
| SliverList.builder (items)    | <------ |  loaded / loadingMore /      | <------ | image cache        |
| footer: spinner|retry|end     |  state  |  error / endReached          |  page   +--------------------+
+-------------------------------+         | items: List<T> (deduped)     |
   ^ scroll position near end             | cursor, requestToken         |
   +---- loadMore() (guarded) ----------> +------------------------------+
```

One source of truth: the **state object** owns `items`, `cursor`, phase, and error. Widgets only render state and send intents (`loadInitial`, `loadMore`, `refresh`, `retry`, `delete(id)`).

## 1. First, establish context

Find out (from the codebase or by asking):

- **Primary or secondary list?** Primary = the screen *is* the list (home feed, inbox, product grid) → skeleton/shimmer first load. Secondary = a list inside another screen (comments under a post, related items, a picker sheet) → small `CircularProgressIndicator`.
- **Paging contract**: cursor (`next_cursor`, `after=<id>`), offset/limit, page number, or "everything in one response". Is ordering stable? Can items be inserted at the top while paging (feeds)? Does the server return `hasMore` or must you infer from `items.length < pageSize`?
- **Volume**: tens (no paging, plain `ListView`), hundreds (paging optional), thousands+ / unbounded (paging mandatory, fixed extents, image downsampling).
- **Realtime**: websockets / Firestore snapshots / push-triggered refetch? New items appended at top → "N new items" pill, not a jump.
- **Offline**: is there a local DB (drift, isar, sqflite, hive)? Stale-while-revalidate? Mutations queued offline?
- **Mutations**: delete, archive, mark read, reorder, favourite — optimistic? undo?
- **Row layout**: fixed height? images? variable text? This decides `itemExtent` / `prototypeItem` and the skeleton shape.
- **State management** in use (Riverpod / Bloc / Provider / signals / setState) and pagination package (`infinite_scroll_pagination`, home-grown, `riverpod_infinite_scroll`…). Follow the project's convention; don't introduce a second one.
- **Platforms**: iOS, Android, web, desktop — affects swipe conventions, scroll physics, refresh control, mouse-wheel/keyboard scrolling.
- Min Flutter / Dart SDK (sealed classes and patterns need Dart 3).

## 2. Batch data loading

- [ ] **Page size** sized to fill ~2–3 screens (typically 20–30; 50 for dense text rows). Too small = request storm; too large = slow first paint.
- [ ] Prefer **cursor** pagination for feeds or anything with inserts/deletes; offset pagination duplicates or skips rows when data shifts. If stuck with offset, dedupe and accept occasional gaps.
- [ ] **Dedupe by stable ID** when merging pages (`LinkedHashMap<Id, T>` or a `Set<Id>` of seen IDs). Duplicates cause double rows, wrong `Key` collisions, and broken animations.
- [ ] Rows have **`ValueKey(item.id)`** (not index, not `UniqueKey()`), especially when rows are stateful, animated, dismissible, or reorderable. `Dismissible` *requires* a key unique to the item.
- [ ] **Cancel / ignore stale requests**: tag each request with a generation token (incremented on refresh/filter change); drop responses whose token doesn't match. With `dio` also pass a `CancelToken`. Never let a slow page-2 response from the *old* query append into the *new* query's list.
- [ ] **Merge preserving scroll**: append pages to the end; never replace the list object in a way that rebuilds from scratch and resets position. When refreshing, replace data in place — keys keep element state.
- [ ] Parse JSON off the UI thread for big pages (`compute` / `Isolate.run`) if profiling shows jank.
- [ ] `hasMore` from the server, or `received < pageSize` as fallback; an empty page always means end.
- [ ] Don't fetch in `build()`. Kick off the first load in `initState`, a provider's `build`, or a Bloc event — once.
- [ ] Check `mounted` (State) / `ref.mounted` (Riverpod 3) / `isClosed` (Bloc) before emitting after `await`.

## 3. Load more

Two standard approaches:

**A. Hand-rolled with `ScrollController` / `NotificationListener<ScrollNotification>`**

- [ ] Trigger when `position.extentAfter < threshold` (e.g. 500–800 px, or ~1 screen) — not at the exact bottom.
- [ ] Alternative trigger: in the builder, when `index >= items.length - 5` → schedule `loadMore()` post-frame (never call `setState`/notify synchronously during build).
- [ ] **Double-trigger guards**: `if (state is LoadingMore || state is LoadingInitial || endReached || hasError) return;` Scroll listeners fire many times per frame.
- [ ] **Short first page**: if the first page doesn't fill the viewport, no scroll happens → no trigger. After each load, check `position.maxScrollExtent == 0` (post-frame) and load again while `hasMore`.
- [ ] Dispose the `ScrollController`.

**B. `infinite_scroll_pagination`** (verify major version):

- [ ] v5+: hold a `PagingState<PageKey, Item>` in your state layer, render `PagedListView` / `PagedSliverList` with `state:` and `fetchNextPage:`; the package handles threshold and guards. v4 and older: `PagingController(firstPageKey:)` + `addPageRequestListener` + `appendPage`/`appendLastPage`/`error`. Don't mix tutorials across versions.
- [ ] Provide all indicator builders via `PagedChildBuilderDelegate`: `firstPageProgressIndicatorBuilder` (skeleton for primary lists), `newPageProgressIndicatorBuilder` (spinner), `firstPageErrorIndicatorBuilder`, `newPageErrorIndicatorBuilder` (retry), `noItemsFoundIndicatorBuilder`, `noMoreItemsIndicatorBuilder`.
- [ ] Refresh via the controller's `refresh()` (v4) or by resetting the state (v5) — don't recreate the controller in `build`.

**Footer states (both approaches)** — the list has `items.length + 1` children when a footer is shown:

| State | Footer |
|---|---|
| loadingMore | Centered `CircularProgressIndicator` (small, ~24 px, adaptive on iOS) |
| error on load-more | Inline message + **Retry** button; keep existing items |
| endReached | Optional subtle "You're all caught up" or nothing; never a spinner |
| idle with more | Nothing (or a "Load more" button if manual) |

- [ ] **Manual "Load more" button** for secondary lists, accessibility mode, or when infinite scroll would hide a footer (legal links, etc.).
- [ ] An error on load-more must **not** auto-retry in a loop as the user scrolls; require a tap (or a connectivity-restored event).

## 4. Loaders

**Primary list, first load → skeleton rows that match the real layout.**

- [ ] Skeleton rows mirror the real row: same height, avatar/thumbnail box, 1–2 text bars of realistic width, same padding. Render enough to fill the viewport (≈ `viewportHeight / rowHeight + 1`), non-scrollable (`NeverScrollableScrollPhysics`) and excluded from semantics except a single "Loading" label.
- [ ] Packages: **`skeletonizer`** (wraps your *real* row widgets with fake data and paints bones — least drift between skeleton and UI; `Skeletonizer(enabled: loading, child: ...)`, `Bone`/`Skeleton.ignore`/`Skeleton.keep` annotations — verify API) or **`shimmer`** (`Shimmer.fromColors(baseColor:, highlightColor:)` over grey boxes you lay out yourself). Or a hand-rolled `ShaderMask` + `AnimationController` (see appendix).
- [ ] Skeleton colors from `Theme.of(context).colorScheme` (`surfaceContainerHighest` etc.) so dark mode works.
- [ ] **One** shimmer animation controller for the whole skeleton, not one per row.

**Secondary list, and load-more → `CircularProgressIndicator`** (`.adaptive()` for Cupertino look on iOS), centred, compact. No shimmer for a 3-item comments section.

**Flicker avoidance**

- [ ] **Delay** showing any loader ~150–300 ms; if data arrives before, show it directly (no flash).
- [ ] Once shown, keep the loader a **minimum ~400–500 ms** so it doesn't blink.
- [ ] Cross-fade skeleton → content (`AnimatedSwitcher`, ~200 ms) rather than hard cut.
- [ ] **Never shimmer over existing content.** Refresh, filter change with cached results, load-more, and retry keep the current rows visible; show a refresh indicator / footer spinner / thin `LinearProgressIndicator` at the top instead.
- [ ] Cached data available on launch → show it immediately, refresh in background; no skeleton.
- [ ] `MediaQuery.disableAnimationsOf(context)` true → static skeleton (no shimmer sweep).

## 5. Swipe actions

- [ ] **`Dismissible`** for a single full-swipe action (delete/archive): `key: ValueKey(item.id)`, `direction`, `background` + `secondaryBackground` (icon + label, colored), `confirmDismiss` for destructive actions or to run the mutation and return `false` on failure, `onDismissed` must **remove the item from state synchronously** or Flutter throws "A dismissed Dismissible widget is still part of the tree".
- [ ] **`flutter_slidable`** (v3+: `Slidable(key:, startActionPane: ActionPane(motion: const DrawerMotion(), children: [SlidableAction(...)]), endActionPane: ...)`, `dismissible: DismissiblePane(onDismissed: ...)`, `SlidableAutoCloseBehavior` to close other open rows) when you need multiple revealed buttons (iOS-mail style). Verify the API version.
- [ ] **Undo**: after delete/archive, show `SnackBar` with `SnackBarAction(label: 'Undo')`; hold the item and its index; on undo re-insert at the original index; commit server delete after the snackbar closes (or optimistic delete + server restore). `ScaffoldMessenger.of(context)..hideCurrentSnackBar()..showSnackBar(...)` so snackbars don't queue up.
- [ ] **Haptics**: `HapticFeedback.mediumImpact()` / `selectionClick()` when the swipe crosses the commit threshold or on commit.
- [ ] **Accessibility / discoverability**: swipe is invisible to screen readers and many users. Provide the same actions via `Semantics(customSemanticsActions: {CustomSemanticsAction(label: 'Delete'): ...})`, **and** a long-press context menu / overflow `PopupMenuButton` / multi-select toolbar. Never make swipe the only path.
- [ ] **Platform conventions**: iOS users expect trailing (right-to-left) swipe for destructive/more actions and leading for read/unread or pin; Android/Material more often uses swipe-to-dismiss (archive/delete) with undo snackbar. Respect RTL (`startActionPane` follows text direction).
- [ ] Destructive full-swipe on iOS-style rows: either confirm or offer undo — not neither.
- [ ] Don't wrap rows in both `Dismissible` and a horizontal `PageView`/carousel; gesture conflict.

## 6. Pull to refresh

- [ ] Material: `RefreshIndicator(onRefresh: () => controller.refresh(), child: ...)`. The `Future` must complete when the refresh finishes (the spinner stays until then). Child must be scrollable with `AlwaysScrollableScrollPhysics` so it works on short/empty lists.
- [ ] iOS: `CustomScrollView(physics: const BouncingScrollPhysics(parent: AlwaysScrollableScrollPhysics()), slivers: [CupertinoSliverRefreshControl(onRefresh: ...), ...])`, or `RefreshIndicator.adaptive`. Match the app's design language.
- [ ] **Keep current data visible** during refresh; replace only on success. On failure: keep old data, show a snackbar/banner.
- [ ] Refresh resets cursor and `endReached`, increments the request generation (so an in-flight load-more is ignored), and replaces items atomically on success.
- [ ] **Refresh vs load-more race**: refresh wins. Either cancel the load-more or discard its result by generation token. Block load-more while refreshing.
- [ ] Empty and error states must also be pull-to-refreshable (wrap them in a scrollable — e.g. `SliverFillRemaining`).
- [ ] Programmatic refresh (e.g. after returning from a create screen) → `GlobalKey<RefreshIndicatorState>().currentState?.show()` or a silent refresh.

## 7. Empty, error, and offline states

| Situation | UI |
|---|---|
| First load failed, nothing cached | Full-screen error: icon, human message, **Retry** button; pull-to-refresh still works |
| First load succeeded, zero items | Empty state: illustration/icon, explanation, primary CTA ("Create your first…") |
| Zero results for search/filter | "No results for 'x'" + "Clear filters" — distinct from true empty |
| Offline with cached data | Show cache + non-blocking banner "Offline — showing saved items" (+ last updated time) |
| Offline, no cache | Offline state + Retry; auto-retry when connectivity returns |
| Refresh / load-more failed with data | Keep data; snackbar or footer retry |

- [ ] Map errors to user messages (timeout, no network, 401 → re-auth flow, 5xx → "Something went wrong"); never show raw exception text.
- [ ] **`connectivity_plus`**: listen to `Connectivity().onConnectivityChanged` (v6+: `Stream<List<ConnectivityResult>>`) to auto-retry a failed first load / load-more and hide the offline banner. Connectivity ≠ internet; still handle request failures. Debounce flapping.
- [ ] Empty/error widgets live inside the scroll view (`SliverFillRemaining(hasScrollBody: false)`) so headers, app bar, and refresh still work.
- [ ] Error states announce via `SemanticsService.announce` or a live region (`Semantics(liveRegion: true)`).

## 8. Thumbnail caches

- [ ] Use **`cached_network_image`** (backed by **`flutter_cache_manager`** disk cache) for remote thumbnails; `Image.network` only caches in memory.
- [ ] **Decode at display size**: `CachedNetworkImage(memCacheWidth: (logicalWidth * devicePixelRatio).round())` (or `memCacheHeight`, set one to keep aspect); for `Image.network`/`Image.asset` use `cacheWidth`/`cacheHeight`, or wrap a provider in `ResizeImage(provider, width: ...)`. A 4000×3000 photo decoded for an 56 px avatar costs ~48 MB of RAM; downsampled it is ~50 KB. This is the #1 list memory bug.
- [ ] Ask the backend/CDN for a sized thumbnail URL too (`?w=168`) — saves bandwidth and decode time.
- [ ] **`ImageCache` tuning**: `PaintingBinding.instance.imageCache.maximumSize` (count, default 1000) and `maximumSizeBytes` (default 100 MB). For image-heavy grids, raise bytes moderately or (better) downsample; on memory pressure call `imageCache.clear()`/`clearLiveImages()` cautiously. Check DevTools Memory → image cache.
- [ ] **Disk cache**: configure a custom `CacheManager(Config('thumbs', stalePeriod: Duration(days: 7), maxNrOfCacheObjects: 500))` for thumbnails separate from full-size media.
- [ ] **Cache keys**: signed/expiring URLs (S3 presigned, tokens in query) change every fetch → cache misses forever. Pass a stable `cacheKey: 'thumb-${item.id}-${item.updatedAt}'`. Include version/updatedAt so edits bust the cache.
- [ ] **`precacheImage`** for the next few items beyond the viewport (or the next page's first rows) — not the whole page; pass the same provider config (incl. resize) or it's a different cache entry.
- [ ] **Placeholders**: fixed-size box (prevents layout shift), neutral color or blurhash (`flutter_blurhash`); `errorWidget` with a fallback icon; `fadeInDuration` short (~150 ms), and `0` when animations are disabled. Fade should not replay when scrolling back to cached images (`useOldImageOnUrlChange`, `fadeInDuration` only on first load).
- [ ] **Eviction**: on logout clear user-specific caches (`DefaultCacheManager().emptyCache()` / custom manager); `CachedNetworkImage.evictFromCache(url, cacheKey:)` after the user updates an image.
- [ ] **Video thumbnails**: never initialise a `VideoPlayerController` per row. Use server-generated poster images; or generate once with `video_thumbnail`/`fc_native_video_thumbnail` into the disk cache, keyed by video ID. Autoplay at most one visible video (`visibility_detector`).
- [ ] Auth headers: `httpHeaders:` param — keep the header map stable (don't create a new map each build if it affects identity).

## 9. List mutations

- [ ] **Optimistic update + rollback**: apply change to local state immediately, send request, on failure restore the previous item at its previous index and show an error snackbar. Keep a snapshot keyed by ID, not by index (indices shift).
- [ ] Mutations must update **all** lists that show the item (shared normalized store / repository stream / invalidate providers).
- [ ] **`AnimatedList` / `SliverAnimatedList`** for animated inserts/removes: call `insertItem(i)` / `removeItem(i, (ctx, anim) => removedRowBuilder)` **in the same frame** as the data change and keep indices in sync; mismatches → RangeError. For diff-based animation of whole-list replacements consider `implicitly_animated_list`/`animated_list_plus`/`great_list_view` (verify maintenance) — or skip animation.
- [ ] **Realtime inserts without jump**: inserting above the viewport shifts content. If the user is at the top (offset ≈ 0) insert and let it animate in; if scrolled down, buffer new items and show a **"N new items" pill**; tapping it inserts and animates to top. With `CustomScrollView(center: key)` you can grow upward without shifting (advanced).
- [ ] Reorder: `ReorderableListView.builder` / `SliverReorderableList`, keys required, persist order optimistically.
- [ ] Edits to an item replace it by ID (`items[i] = updated`), preserving key so row state survives.

## 10. Performance

- [ ] Always **builder constructors** for anything non-trivial: `ListView.builder`, `ListView.separated`, `SliverList.builder` / `SliverList.separated`, `GridView.builder`, `SliverGrid.builder`. `ListView(children: [...])` builds everything eagerly.
- [ ] Fixed row height → **`itemExtent`** (or `itemExtentBuilder`, Flutter 3.16+); same-shaped rows → **`prototypeItem`**. Gives O(1) layout and accurate scrollbars/jump-to.
- [ ] `const` constructors for static subtrees; extract rows into their own `StatelessWidget` classes (not helper methods) so rebuilds are scoped.
- [ ] Rebuild only what changed: select fine-grained state (`ref.watch(provider.select(...))`, `BlocSelector`, `context.select`) — a "favourite" toggle shouldn't rebuild the whole list.
- [ ] `RepaintBoundary` is added per item by default (`addRepaintBoundaries: true`); add one around expensive animated bits (shimmer, progress) only if DevTools shows repaints spreading.
- [ ] **Avoid `shrinkWrap: true` + `NeverScrollableScrollPhysics` for long nested lists** — it lays out every child (no virtualization). Compose with `CustomScrollView` + slivers instead (`SliverToBoxAdapter`, `SliverList`, `SliverMainAxisGroup`). Small fixed lists (≤ ~20) are OK.
- [ ] Don't put a `ListView` inside a `Column` without `Expanded` (unbounded height error) — and don't "fix" it with shrinkWrap for big lists.
- [ ] Avoid `Opacity`, `ClipPath`, `BackdropFilter`, heavy `BoxShadow` per row; prefer `FadeTransition`, `ClipRRect` only where needed, pre-composited images.
- [ ] `cacheExtent` default (~250 px) is fine; raising it pre-builds more rows (smoother image pop-in, more memory). Tune with measurement.
- [ ] `addAutomaticKeepAlives`: rows with `AutomaticKeepAliveClientMixin` stay alive — don't use it on every row of a long list.
- [ ] Profile in **profile mode on a real low-end device**: DevTools Performance (frame chart, raster vs UI thread), "Track widget rebuilds", Memory tab for image bytes. Target < 16 ms (60 Hz) / < 8 ms (120 Hz).
- [ ] Impeller: shader compilation jank largely solved on iOS/Android; if on Skia, consider SkSL warmup (verify current guidance).

## 11. Search, filter, sort

- [ ] **Debounce** text input 250–400 ms (`Timer` or `rxdart`'s `debounceTime`/Bloc `EventTransformer` with `restartable()` from `bloc_concurrency`).
- [ ] **Cancel stale**: each query bumps the generation; drop older responses (Bloc `restartable()` does this; Riverpod: the provider family/auto-dispose rebuild + `ref.onDispose(cancelToken.cancel)`).
- [ ] Any query/filter/sort change **resets paging**: clear cursor, `endReached = false`, scroll to top (`jumpTo(0)` when results arrive), show loader (skeleton if no results to keep, else keep old results dimmed with a top `LinearProgressIndicator`).
- [ ] Persist filter/sort in state (and optionally URL with go_router query params) so back navigation restores it.
- [ ] Minimum query length / empty-query behaviour defined (show recents vs all).
- [ ] Local filtering of an already-fully-loaded small list is fine; filtering a *paged* list locally is wrong (shows only matches in loaded pages).

## 12. Selection and bulk actions

- [ ] Enter selection mode by long-press (and an explicit "Select" button); show contextual app bar with count, "Select all", and actions; back/escape (`PopScope`) exits selection mode before popping route.
- [ ] Store selection as `Set<Id>`, not indices; survives inserts, refresh, and paging. Prune IDs that disappear.
- [ ] "Select all" on a paged list: define semantics (loaded items vs server-side all-matching) and say which in the UI.
- [ ] Rows show a checkbox/check overlay; `Semantics(selected: true)` and toggle via tap in selection mode.
- [ ] Bulk delete: optimistic with single undo for the whole batch; partial failure → report which failed and restore them.
- [ ] Disable swipe actions while in selection mode.

## 13. Sections and sticky headers

- [ ] Build sections with `CustomScrollView` slivers: per section a header + `SliverList.builder`. Flutter 3.13+: **`SliverMainAxisGroup`** groups header + list so a `SliverPersistentHeader(pinned: true)` sticks only within its section.
- [ ] `SliverPersistentHeader` needs a delegate with correct `minExtent`/`maxExtent` and `shouldRebuild`; or use **`sliver_tools`** (`SliverPinnedHeader`, `MultiSliver`, `SliverStack`) / **`flutter_sticky_header`** (verify maintenance vs built-in `SliverMainAxisGroup`, `PinnedHeaderSliver` 3.24+).
- [ ] For a flat data list, pre-compute a "rows" list of `Header | Item` entries (sealed type) and render with one builder — cheaper than many slivers for hundreds of sections.
- [ ] Section index scrubber (A–Z): needs fixed extents or `scrollable_positioned_list`/`super_sliver_list` to jump to an index reliably.
- [ ] Paging across sections: a new page may extend the last section — group after merging, not per page.
- [ ] Headers marked `Semantics(header: true)`.

## 14. Scroll behaviour

- [ ] **Restore position** when switching tabs/returning: `PageStorageKey('inbox-list')` on the scroll view (and keep the state alive — e.g. `IndexedStack`, `StatefulShellRoute` in go_router, or `AutomaticKeepAliveClientMixin` on the tab). Keep data in state above the widget, or restored offset points at empty space.
- [ ] Restoration across process death: `restorationId` + `RestorationMixin` if the app uses state restoration.
- [ ] **Scroll to top**: tap the active tab again / status bar tap on iOS (`primary: true` scroll view + `Scaffold` gives this for free if the controller is the primary one); `animateTo(0, duration: 300ms, curve: Curves.easeOut)`; for very long lists `jumpTo` near top then animate.
- [ ] **"New items" pill** (see §9) floating at top, dismisses on scroll to top.
- [ ] Jump to a specific item: requires fixed extents or `scrollable_positioned_list` / `super_sliver_list` (`ListController.jumpToItem`) / `Scrollable.ensureVisible` with a `GlobalKey` (only if built).
- [ ] Keyboard: `keyboardDismissBehavior: ScrollViewKeyboardDismissBehavior.onDrag` for search lists.
- [ ] Desktop/web: `Scrollbar` visible and draggable, mouse-wheel and keyboard (PageUp/Down) work; `ScrollConfiguration` drag devices if needed.
- [ ] Don't recreate the `ScrollController` in `build`; don't attach one controller to two scroll views.

## 15. Accessibility

- [ ] Each row is one semantic node with a meaningful label: `MergeSemantics` / `Semantics(label: '${sender}, ${subject}, ${time}, unread')`; decorative images `excludeFromSemantics: true`; thumbnails get `semanticLabel` only if they carry meaning.
- [ ] Swipe actions exposed as `customSemanticsActions` + visible alternative (§5).
- [ ] Skeleton: `ExcludeSemantics` around bones, one `Semantics(label: 'Loading items')`; announce when loaded / when load-more errors.
- [ ] **Text scaling**: test at 200% (`MediaQuery.textScalerOf`). Fixed `itemExtent` breaks with large text — compute extent from text scale, or use `prototypeItem`, or drop it. No clipped text; rows grow.
- [ ] **`MediaQuery.disableAnimationsOf(context)`**: disable shimmer sweep, list insert animations, fades; use instant transitions.
- [ ] Tap targets ≥ 48×48 (Material) / 44×44 (iOS) for row actions, retry buttons, checkboxes.
- [ ] Contrast of skeleton vs background, and of text on swipe backgrounds.
- [ ] Footer "Load more" button reachable by switch access/keyboard; infinite scroll should still load when a screen reader scrolls (it does via scroll actions, but verify).
- [ ] Use `SemanticsService.announce` sparingly for "20 more items loaded" / "Item deleted, undo available".

## 16. Testing plan

Widget tests with a fake repository (`Completer`-controlled futures) make every state reachable deterministically.

### Scenario matrix

| Scenario | Expect |
|---|---|
| First load, primary list, slow (>300 ms) | Skeleton rows shown (not spinner), then content cross-fade |
| First load fast (<150 ms) | No loader flash |
| First load, secondary list | Small spinner |
| First load error | Error view + Retry; Retry → skeleton → content |
| Empty response | Empty state with CTA; pull-to-refresh works |
| Scroll to near end | Exactly one `loadMore` request (fling + many notifications) |
| First page shorter than viewport | Auto-loads next page until filled or end |
| Load-more error | Items kept, footer retry; scrolling doesn't spam retries |
| End reached | No more requests, end footer/no spinner |
| Duplicate IDs across pages | Rendered once |
| Pull to refresh during load-more | Load-more result discarded; list = fresh page 1 |
| Refresh fails | Old data kept, snackbar |
| Search typed quickly "abc" | One request for "abc"; stale "ab" response ignored; scroll at top |
| Swipe delete → Undo | Item restored at original index; no server delete |
| Swipe delete, server fails | Item restored, error shown |
| Screen reader | Rows read as one label; delete reachable via actions menu |
| Text scale 2.0 | No overflow (`tester.takeException()` is null) |
| Disable animations | No shimmer animation running (`pumpAndSettle` completes) |
| Offline with cache | Cached rows + offline banner; reconnect → auto-refresh |
| Tab switch & back | Scroll offset restored |
| 10k items profile run | No jank; image memory bounded |

Widget-test tips:
- [ ] Shimmer/infinite animations make `pumpAndSettle` time out — use `pump(duration)` or disable animations in tests (`MediaQuery(data: MediaQueryData(disableAnimations: true))`).
- [ ] Scroll with `tester.drag(find.byType(ListView), const Offset(0, -3000))` or `tester.scrollUntilVisible(finder, 500)`, then `pump()`.
- [ ] Refresh: `tester.fling(find.byType(ListView), const Offset(0, 400), 1000)` then `pump()` + `pump(const Duration(seconds: 1))`.
- [ ] Swipe: `tester.drag(find.byKey(ValueKey(id)), const Offset(-500, 0))`, `pumpAndSettle()`.
- [ ] Network images in tests: mock HTTP (`mocktail_image_network` / `HttpOverrides`) or inject an `ImageProvider` factory.
- [ ] Golden tests for skeleton vs real row alignment; unit-test the controller/notifier/bloc state transitions separately (`bloc_test`, `ProviderContainer`).
- [ ] Semantics: `tester.ensureSemantics()`, `expect(tester.getSemantics(finder), matchesSemantics(label: ..., customActions: [...]))`; `meetsGuideline(androidTapTargetGuideline)` / `textContrastGuideline`.

## 17. Common bugs to look for in review

- `ListView(children: items.map(...).toList())` for large/unbounded data — no virtualization.
- `shrinkWrap: true` nested long lists inside `SingleChildScrollView`/`Column`.
- Keys missing or `Key(index)` / `UniqueKey()` → state jumps between rows, Dismissible errors, image flicker.
- `onDismissed` doesn't remove the item from the data → "dismissed Dismissible still part of the tree".
- Load-more fired multiple times per scroll; no `isLoading` guard; or guard flag reset in `finally` *before* state is updated.
- First page shorter than viewport → list never loads page 2.
- Stale responses appended after a refresh/filter change (no generation token/cancellation).
- Refresh clears `items` first → full-screen skeleton/blank flash and scroll reset.
- Skeleton shown on refresh/load-more (shimmer over existing content).
- Spinner shown for 50 ms on fast networks (no delay / min duration).
- Offset pagination with inserts → duplicates or skipped rows.
- `setState`/notify called during build (from builder index trigger) → "setState() called during build".
- `setState` after dispose (no `mounted` check).
- `ScrollController`/`TextEditingController`/`Timer`/stream subscription not disposed.
- Full-resolution images decoded for thumbnails (no `memCacheWidth`/`cacheWidth`) → OOM on Android, jank.
- Presigned URLs used as cache keys → nothing ever hits cache.
- `CachedNetworkImage` without fixed-size placeholder → layout jumps as images load.
- `VideoPlayerController` per row.
- Search without debounce / without resetting the cursor and scroll position.
- Selection stored by index.
- `RefreshIndicator` with a non-scrollable empty state (can't pull to retry) or without `AlwaysScrollableScrollPhysics`.
- `RefreshIndicator.onRefresh` future completes immediately (spinner disappears before data arrives) or never completes.
- `AnimatedList` key/index desync with the underlying list.
- Swipe as the only way to delete; no semantics action; no undo/confirm.
- Fixed `itemExtent` with dynamic text → clipping at large text scales.
- Shimmer keeps animating with "Remove animations" enabled; also breaks `pumpAndSettle` in tests.
- `ScrollController` created inside `build`; position lost on tab switch (no `PageStorageKey`/keep-alive).
- Error messages showing `DioException [connection error]...` raw text.

## 18. Output format

**When reviewing** an existing list screen, respond with:

1. **Context summary** — primary/secondary, paging contract, volume, state management, packages + versions.
2. **Findings** grouped by area (Loading, Load more, Loaders, Refresh, Swipe, States, Images, Mutations, Performance, Search, Selection, Sections, Scroll, A11y), each with **severity** (Critical / High / Medium / Low), file:line, what's wrong, the user-visible symptom, and the fix (code snippet when useful).
3. **Missing states** — which rows of the §7 table and §16 matrix aren't handled.
4. **Test gaps** — concrete widget tests to add.
5. **Version notes** — APIs that need verification against the project's package versions.

**When implementing**, deliver:

1. The state type (sealed `ListState` or equivalent in the project's state library) and controller with load/refresh/loadMore/retry/mutation methods, generation-token stale-response handling, and dedupe.
2. The screen widget: scroll view (slivers if headers/sections), refresh control, item builder with `ValueKey`s, footer, empty/error/offline views, skeleton (primary) or spinner (secondary).
3. Row widget with sized cached thumbnails and accessible swipe/long-press actions.
4. Widget tests covering the §16 matrix essentials (first load, empty, error+retry, load more once, end, refresh race, swipe+undo).
5. A short note of assumptions (page size, thresholds, durations) and package versions used, flagging anything to verify.

## Appendix — Reference implementation (Dart 3, Flutter 3.24+)

Packages assumed (verify latest compatible versions): `cached_network_image`, `flutter_slidable` ^3 / ^4, optional `skeletonizer`. The controller is a `ChangeNotifier`; port it to a Riverpod `Notifier<ListState<T>>` or a `Cubit<ListState<T>>` by replacing `_emit` with `state = ...` / `emit(...)`.

### A.1 State machine

```dart
// list_state.dart
sealed class ListState<T> { const ListState(); }
final class ListIdle<T> extends ListState<T> { const ListIdle(); }
final class ListLoadingInitial<T> extends ListState<T> { const ListLoadingInitial(); }

/// Data present; more pages may exist.
final class ListLoaded<T> extends ListState<T> {
  const ListLoaded(this.items, {this.isRefreshing = false});
  final List<T> items;
  final bool isRefreshing;
}
final class ListLoadingMore<T> extends ListState<T> { const ListLoadingMore(this.items); final List<T> items; }

/// [items] empty => first-load error (full screen). Non-empty => footer/snackbar error.
final class ListError<T> extends ListState<T> {
  const ListError(this.error, {this.items = const []});
  final Object error;
  final List<T> items;
}

final class ListEndReached<T> extends ListState<T> {
  const ListEndReached(this.items); final List<T> items; // may be empty => empty state
}

extension ListStateX<T> on ListState<T> {
  List<T> get items => switch (this) {
        ListLoaded(:final items) ||
        ListLoadingMore(:final items) ||
        ListError(:final items) ||
        ListEndReached(:final items) =>
          items,
        ListIdle() || ListLoadingInitial() => const [],
      };
}
```

### A.2 Paginator controller

```dart
// paginator.dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import 'list_state.dart';

class Page<T> {
  const Page(this.items, this.nextCursor); final List<T> items; final String? nextCursor; // null => end
}

typedef PageFetcher<T> = Future<Page<T>> Function({String? cursor, required int limit});

class Paginator<T, Id> extends ChangeNotifier {
  Paginator({required this.fetch, required this.idOf, this.pageSize = 25});

  final PageFetcher<T> fetch;
  final Id Function(T) idOf;
  final int pageSize;

  ListState<T> _state = const ListIdle();
  ListState<T> get state => _state;

  String? _cursor;
  int _generation = 0; // bumps on refresh/reset; stale responses are dropped
  bool _disposed = false;

  void _emit(ListState<T> s) {
    if (_disposed) return;
    _state = s;
    notifyListeners();
  }

  Future<void> loadInitial() async {
    if (_state is ListLoadingInitial) return;
    final gen = ++_generation;
    _cursor = null;
    _emit(const ListLoadingInitial());
    await _fetchPage(gen, existing: const []);
  }

  /// Keeps current items visible; replaces them on success.
  Future<void> refresh() async {
    final current = _state.items;
    if (current.isEmpty) return loadInitial();
    final gen = ++_generation; // invalidates in-flight loadMore
    _emit(ListLoaded(current, isRefreshing: true));
    try {
      final page = await fetch(cursor: null, limit: pageSize);
      if (gen != _generation) return;
      _cursor = page.nextCursor;
      final items = _merge(const [], page.items);
      _emit(page.nextCursor == null ? ListEndReached(items) : ListLoaded(items));
    } catch (e) {
      if (gen != _generation) return;
      _emit(ListLoaded(current)); // keep data; UI shows a snackbar via [lastError]
      lastError = e;
      notifyListeners();
    }
  }

  Object? lastError;

  Future<void> loadMore() async {
    final s = _state;
    // Guards: only from a settled loaded state, never while refreshing.
    if (s is! ListLoaded<T> || s.isRefreshing || _cursor == null) return;
    final gen = _generation;
    _emit(ListLoadingMore(s.items));
    await _fetchPage(gen, existing: s.items);
  }

  Future<void> retry() =>
      _state.items.isEmpty ? loadInitial() : _retryMore();

  Future<void> _retryMore() async {
    final gen = _generation;
    final items = _state.items;
    _emit(ListLoadingMore(items));
    await _fetchPage(gen, existing: items);
  }

  Future<void> _fetchPage(int gen, {required List<T> existing}) async {
    try {
      final page = await fetch(cursor: _cursor, limit: pageSize);
      if (gen != _generation) return; // stale
      _cursor = page.nextCursor;
      final items = _merge(existing, page.items);
      final ended = page.nextCursor == null || page.items.isEmpty;
      _emit(ended ? ListEndReached(items) : ListLoaded(items));
    } catch (e) {
      if (gen != _generation) return;
      _emit(ListError(e, items: existing));
    }
  }

  /// Dedupe by id; later pages update earlier copies in place.
  List<T> _merge(List<T> existing, List<T> incoming) {
    final map = <Id, T>{for (final i in existing) idOf(i): i};
    for (final i in incoming) {
      map[idOf(i)] = i;
    }
    return List.unmodifiable(map.values);
  }

  // ---- Mutations (optimistic + rollback) ----

  /// Removes locally, returns an undo callback. [commit] runs the server call.
  ({int index, T item})? removeLocally(Id id) {
    final items = [..._state.items];
    final index = items.indexWhere((i) => idOf(i) == id);
    if (index < 0) return null;
    final item = items.removeAt(index);
    _replaceItems(items);
    return (index: index, item: item);
  }

  void restore(({int index, T item}) removed) {
    final items = [..._state.items];
    if (items.any((i) => idOf(i) == idOf(removed.item))) return;
    items.insert(removed.index.clamp(0, items.length), removed.item);
    _replaceItems(items);
  }

  void _replaceItems(List<T> items) {
    final s = _state;
    _emit(switch (s) {
      ListEndReached() => ListEndReached(items),
      ListLoadingMore() => ListLoadingMore(items),
      ListError(:final error) => ListError(error, items: items),
      _ => ListLoaded(items),
    });
  }

  @override
  void dispose() {
    _disposed = true;
    _generation++;
    super.dispose();
  }
}
```

### A.3 Shimmer skeleton (no package; respects reduced motion)

```dart
// skeleton.dart — sketch (no package):
// - Shimmer: StatefulWidget with a repeating AnimationController (~1.2 s) driving a ShaderMask
//   LinearGradient sweep over its child; when MediaQuery.disableAnimationsOf(context) is true,
//   render the child statically (no controller running).
// - SkeletonList: ListView.builder of ~8 ItemRow-shaped placeholders (same extent/thumb size as
//   ItemRow, grey rounded boxes), NeverScrollableScrollPhysics, wrapped in
//   Semantics(label: 'Loading', excludeSemantics: true) and Shimmer.
```

With `skeletonizer` instead: `Skeletonizer(enabled: true, child: ListView.builder(itemCount: 8, itemBuilder: (_, __) => ItemRow(item: Item.fake())))` — reuses the real row so the skeleton can never drift from the layout.

### A.4 Delayed loader (anti-flicker)

```dart
import 'dart:async'; // Timer

/// Shows [child] only if still mounted after [delay]; once shown, the parent
/// should keep it for at least [minShow] (handled by DelayedSwitcher below).
class DelayedLoader extends StatefulWidget {
  const DelayedLoader({super.key, required this.child,
      this.delay = const Duration(milliseconds: 200)});
  final Widget child;
  final Duration delay;
  @override
  State<DelayedLoader> createState() => _DelayedLoaderState();
}

class _DelayedLoaderState extends State<DelayedLoader> {
  bool _visible = false;
  late final Timer _t = Timer(widget.delay, () {
    if (mounted) setState(() => _visible = true);
  });
  @override
  void initState() { super.initState(); _t; }
  @override
  void dispose() { _t.cancel(); super.dispose(); }
  @override
  Widget build(BuildContext context) =>
      _visible ? widget.child : const SizedBox.expand();
}
```

For a minimum display time, have the controller record when loading became visible and `await Future.delayed(remaining)` before emitting the loaded state — or simply cross-fade with `AnimatedSwitcher`, which hides most blinks.

### A.5 Screen: slivers, refresh, Slidable rows, footer, states

```dart
// list_screen.dart
// imports: cached_network_image, flutter/material, flutter/services, flutter_slidable,
//          list_state.dart, paginator.dart, skeleton.dart

class Item { // id, title, subtitle, thumbUrl, updatedAt
  const Item({required this.id, required this.title, required this.subtitle,
      required this.thumbUrl, required this.updatedAt});
  final String id, title, subtitle, thumbUrl; final DateTime updatedAt;
}

class ItemListScreen extends StatefulWidget {
  const ItemListScreen({super.key, required this.paginator, required this.deleteOnServer});
  final Paginator<Item, String> paginator;
  final Future<void> Function(String id) deleteOnServer;
  @override
  State<ItemListScreen> createState() => _ItemListScreenState();
}

class _ItemListScreenState extends State<ItemListScreen> {
  final _scroll = ScrollController();
  Paginator<Item, String> get _p => widget.paginator;

  @override
  void initState() {
    super.initState();
    _scroll.addListener(_onScroll);
    _p.addListener(_onStateChanged);
    _p.loadInitial();
  }

  void _onScroll() {
    final pos = _scroll.position;
    if (pos.extentAfter < 600) _p.loadMore(); // Paginator guards duplicates
  }

  void _onStateChanged() {
    // Short first page: keep loading until the viewport is filled.
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (!mounted || !_scroll.hasClients) return;
      if (_scroll.position.maxScrollExtent <= 0) _p.loadMore();
    });
    if (_p.lastError != null) {
      _p.lastError = null;
      ScaffoldMessenger.of(context)..hideCurrentSnackBar()
        ..showSnackBar(const SnackBar(content: Text("Couldn't refresh. Showing saved items.")));
    }
  }

  @override
  void dispose() {
    _p.removeListener(_onStateChanged);
    _scroll.dispose();
    super.dispose();
  }

  Future<void> _delete(Item item) async {
    HapticFeedback.mediumImpact();
    final removed = _p.removeLocally(item.id);
    if (removed == null) return;
    final messenger = ScaffoldMessenger.of(context)..hideCurrentSnackBar();
    var undone = false;
    final reason = await messenger
        .showSnackBar(SnackBar(
          content: Text('Deleted "${item.title}"'),
          action: SnackBarAction(label: 'Undo', onPressed: () {
            undone = true;
            _p.restore(removed);
          }),
        ))
        .closed;
    if (undone || reason == SnackBarClosedReason.action) return;
    try {
      await widget.deleteOnServer(item.id);
    } catch (_) {
      _p.restore(removed); // rollback
      messenger.showSnackBar(const SnackBar(content: Text("Couldn't delete item")));
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Items')),
      body: ListenableBuilder(
        listenable: _p,
        builder: (context, _) {
          final s = _p.state;
          return AnimatedSwitcher(
            duration: const Duration(milliseconds: 200),
            child: switch (s) {
              ListIdle() || ListLoadingInitial() =>
                const DelayedLoader(key: ValueKey('skeleton'), child: SkeletonList()),
              _ => _buildList(context, s),
            },
          );
        },
      ),
    );
  }

  Widget _buildList(BuildContext context, ListState<Item> s) {
    final items = s.items;
    return RefreshIndicator.adaptive(
      key: const ValueKey('list'),
      onRefresh: _p.refresh,
      child: SlidableAutoCloseBehavior(
        child: CustomScrollView(
          key: const PageStorageKey('item-list'),
          controller: _scroll,
          physics: const AlwaysScrollableScrollPhysics(),
          slivers: [
            if (items.isEmpty)
              SliverFillRemaining(
                hasScrollBody: false,
                child: switch (s) {
                  ListError() => _ErrorView(onRetry: _p.retry),
                  _ => const _EmptyView(),
                },
              )
            else ...[
              SliverFixedExtentList.builder(
                itemExtent: ItemRow.extent, // drop if text scaling must grow rows
                itemCount: items.length,
                itemBuilder: (context, i) {
                  final item = items[i];
                  return ItemRow(
                    key: ValueKey(item.id),
                    item: item,
                    onDelete: () => _delete(item),
                  );
                },
              ),
              SliverToBoxAdapter(child: _Footer(state: s, onRetry: _p.retry)),
            ],
          ],
        ),
      ),
    );
  }
}

// _Footer(state, onRetry): ListLoadingMore → 24px adaptive spinner; ListError → Semantics(liveRegion)
// "Couldn't load more" + Retry TextButton; ListEndReached → "You're all caught up"; else SizedBox.shrink().

class ItemRow extends StatelessWidget {
  const ItemRow({super.key, required this.item, required this.onDelete});
  static const double extent = 72;
  static const double thumb = 56;
  final Item item;
  final VoidCallback onDelete;

  @override
  Widget build(BuildContext context) {
    return Slidable(
      key: ValueKey('slidable-${item.id}'),
      endActionPane: ActionPane(
        motion: const DrawerMotion(),
        extentRatio: 0.25,
        children: [
          SlidableAction(onPressed: (_) => onDelete(), icon: Icons.delete, label: 'Delete',
              backgroundColor: Theme.of(context).colorScheme.error,
              foregroundColor: Theme.of(context).colorScheme.onError),
        ],
      ),
      child: Semantics(
        customSemanticsActions: {
          const CustomSemanticsAction(label: 'Delete'): onDelete,
        },
        child: ListTile(
          minVerticalPadding: 8,
          leading: Thumbnail(item: item, size: thumb),
          title: Text(item.title, maxLines: 1, overflow: TextOverflow.ellipsis),
          subtitle: Text(item.subtitle, maxLines: 1, overflow: TextOverflow.ellipsis),
          onTap: () {/* navigate */},
          onLongPress: () => _showActions(context), // non-swipe alternative
        ),
      ),
    );
  }

  // _showActions: showModalBottomSheet with a 'Delete' ListTile (pop, then onDelete()).
  void _showActions(BuildContext context) {/* see comment */}
}

// _EmptyView: centered inbox icon + "No items yet".
// _ErrorView(onRetry): Semantics(liveRegion) cloud_off icon + "Couldn't load items. Check your
//   connection." + FilledButton Retry.
```

Notes: `SliverFixedExtentList.builder` exists in recent Flutter; on older versions use `SliverFixedExtentList(itemExtent:, delegate: SliverChildBuilderDelegate(...))`.

### A.6 Thumbnail with display-size decode and stable cache key

```dart
final thumbCache = CacheManager(Config('thumbs',
    stalePeriod: const Duration(days: 7), maxNrOfCacheObjects: 600)); // flutter_cache_manager

class Thumbnail extends StatelessWidget {
  const Thumbnail({super.key, required this.item, required this.size});
  final Item item;
  final double size;

  @override
  Widget build(BuildContext context) {
    final dpr = MediaQuery.devicePixelRatioOf(context);
    final px = (size * dpr).round();
    final reduceMotion = MediaQuery.disableAnimationsOf(context);
    final placeholder = ColoredBox(
        color: Theme.of(context).colorScheme.surfaceContainerHighest);
    return ClipRRect(
      borderRadius: BorderRadius.circular(8),
      child: SizedBox.square(
        dimension: size, // fixed box: no layout shift
        child: CachedNetworkImage(
          imageUrl: item.thumbUrl,
          // Stable across presigned-URL changes; busts on edit.
          cacheKey: 'thumb-${item.id}-${item.updatedAt.millisecondsSinceEpoch}',
          cacheManager: thumbCache,
          memCacheWidth: px, // decode at display size, not source size
          fit: BoxFit.cover,
          fadeInDuration: reduceMotion ? Duration.zero : const Duration(milliseconds: 150),
          placeholder: (_, __) => placeholder,
          errorWidget: (_, __, ___) => const Icon(Icons.broken_image_outlined),
        ),
      ),
    );
  }
}

// precacheThumbs(context, upcoming, size): for the next ~6 items call precacheImage(
//   ResizeImage(CachedNetworkImageProvider(url, cacheKey: <same key>, cacheManager: thumbCache),
//   width: px), context). Image-heavy apps only (measure first): raise
//   PaintingBinding.instance.imageCache maximumSize / maximumSizeBytes.
```

Caveat: `memCacheWidth` on `CachedNetworkImage` wraps its provider in `ResizeImage` internally, so the precache call must use the same width for a memory-cache hit — verify against the installed version.

### A.7 Widget test sketch

```dart
// Sketch: build a Paginator whose fake fetch returns 50 items in pages and counts calls; pump
// ItemListScreen inside MaterialApp with MediaQueryData(disableAnimations: true); pump past the
// 200 ms DelayedLoader; drag the CustomScrollView far twice; assert calls == 2 (no duplicate
// load-more) and that "You're all caught up" is shown.
```
