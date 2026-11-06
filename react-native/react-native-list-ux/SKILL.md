---
name: react-native-list-ux
description: Use when implementing, reviewing, or debugging list screens in React Native and Expo — FlashList / FlatList / SectionList, paginated loading with TanStack Query useInfiniteQuery (cursor vs offset, dedupe, cancellation), load-more footers, shimmer skeletons vs spinners, pull to refresh, swipe actions with Gesture Handler + Reanimated, empty/error/offline states, thumbnail caching with expo-image, optimistic mutations, search/filter/sort, selection and bulk actions, sticky sections, scroll restoration, accessibility, performance, and RNTL tests. Siblings: ios-list-ux (UIKit/SwiftUI), android-list-ux (Compose/RecyclerView), flutter-list-ux.
---

# React Native List UX Skill

Guide for building, auditing, and testing **list screens** in React Native and Expo: feeds, inboxes, search results, catalogs, settings-like lists, chat histories. Covers **data loading in batches**, **load more**, **loaders (shimmer vs spinner)**, **swipe actions**, **pull to refresh**, **empty / error / offline states**, **thumbnail caching**, **mutations**, **performance**, and **accessibility**. Preferred stack: **@shopify/flash-list** + **@tanstack/react-query** + **expo-image** + **react-native-gesture-handler** + **react-native-reanimated**. Siblings: **ios-list-ux**, **android-list-ux**, **flutter-list-ux**.

These libraries move fast and several now **require the New Architecture** (Fabric + TurboModules): FlashList v2, Reanimated 4 (plus `react-native-worklets`), and recent Gesture Handler / expo-image releases. Expo SDK 52+ enables the New Architecture by default, and RN 0.82+ no longer lets you turn it off. **Check `package.json`, `app.json` (`newArchEnabled`) and the installed library versions before writing code, and verify prop names against the current docs.** The APIs most likely to have changed are FlashList props, the Swipeable import path, TanStack Query option names, and expo-image cache APIs.

## 0. Mental model

```
           ┌──────────────────────── Screen ───────────────────────────┐
 server ◄──┤ useInfiniteQuery(key = [resource, filters])               │
 (pages)   │   pages[] ──flatten+dedupe by id──► items[]               │
           │   status/fetchStatus ──derive──► ListState (union)        │
           │                                                           │
           │ FlashList data=items keyExtractor=id getItemType          │
           │   ├─ ListHeaderComponent  (search, filters, banner)       │
           │   ├─ renderItem → memo(Row) → SwipeableRow → ThumbImage   │
           │   ├─ ListEmptyComponent   (skeleton | empty | error)      │
           │   ├─ ListFooterComponent  (spinner | retry | end)         │
           │   ├─ onEndReached ──guard──► fetchNextPage()              │
           │   └─ refreshing/onRefresh ──► refetch() (keeps data)      │
           │ useMutation (optimistic onMutate / rollback onError)      │
           └───────────────────────────────────────────────────────────┘
 persisted cache (AsyncStorage/MMKV) ◄─► QueryClient ◄─ onlineManager(NetInfo)
```

Keep one source of truth (the query cache), derive the UI state from it, and keep rows dumb, memoized, and recyclable.

## 1. First, establish context

Find out from the codebase, or ask:

- **Primary or secondary list?** A *primary* list is the screen's main content (home feed, inbox, catalog): first load gets a **shimmer skeleton** that matches the row layout. A *secondary* list is embedded or supporting (comments under a post, "related items", a picker sheet, a settings sub-list): use a small **ActivityIndicator**. Load-more is always a spinner in the footer.
- **Paging contract**: cursor (`nextCursor`, `hasMore`), offset/limit, page number, keyset (`after=id`), or "everything in one response". Page size? Is the order stable while you page (new items inserted at the top shift offsets)?
- **Volume**: tens (no paging; plain `map` in a ScrollView can be OK below ~30 simple rows), hundreds, or unbounded feeds. Row heterogeneity: one row type or many (headers, ads, cards, media)?
- **Realtime**: websockets, SSE, polling, push-triggered refetch? Do inserts arrive at the top while the user reads (chat, feeds)?
- **Offline**: must the last list show offline? Can the user mutate offline (queued mutations)?
- **Interactions**: swipe actions, multi-select / bulk actions, reorder (drag), inline edit, long-press context menu, tap → detail with back restoring position.
- **Search / filter / sort**: server-side or client-side? Debounced as-you-type?
- **Environment**: Expo managed / dev client / bare; RN version; New Architecture on?; Hermes on?; versions of FlashList (v1 vs v2), Reanimated (3 vs 4), Gesture Handler, TanStack Query (v4 vs v5), expo-image; navigation (React Navigation / Expo Router).
- **Platforms**: iOS, Android, also web (react-native-web changes gestures, images and virtualization)?

Write a **list spec** before coding:

```
screen        kind       paging         page  realtime     offline       swipe (lead / trail)     select
Inbox         primary    cursor         30    ws inserts   persisted     read/unread / archive,delete  yes
Comments      secondary  cursor         20    poll 30s     no            — / delete (own)          no
Search        primary    offset         25    —            no            — / —                     no
```

## 2. Batch data loading

### 2.1 Query setup (TanStack Query v5)

- [ ] `useInfiniteQuery` with `queryKey: ['items', filters]`, `queryFn: ({ pageParam, signal }) => fetchPage(pageParam, signal)`, `initialPageParam`, `getNextPageParam: (last) => last.nextCursor ?? undefined` (return `undefined`/`null` to signal the end — in v5 `hasNextPage` derives from that)
- [ ] v4 → v5 differences: `initialPageParam` is required, `status: 'pending'` replaced `'loading'`, `isLoading` = `isPending && isFetching`, `keepPreviousData` → `placeholderData: keepPreviousData`, `cacheTime` → `gcTime`, `maxPages` added. Check which version is installed
- [ ] `staleTime` set deliberately (e.g. 30 s for feeds) so tab switches don't refetch every time
- [ ] Consider `maxPages` for very long feeds (limits memory and refetch cost). It needs `getPreviousPageParam` too, and refetch re-requests every *kept* page in order
- [ ] Page size fits roughly 2–3 screens of rows. Too small = request storms; too large = slow first paint

### 2.2 Cursor vs offset

| | Cursor / keyset | Offset / page number |
|---|---|---|
| Inserts/deletes while paging | Stable | Duplicates or skipped rows |
| Jump to page N | No | Yes |
| Server cost on deep pages | Constant | Grows with offset |
| Use for | Feeds, chats, inboxes | Small stable catalogs, admin tables |

- [ ] Prefer cursors. With offsets: **dedupe by ID** when you flatten, and accept that rows can be skipped (or refetch from the start on known inserts)
- [ ] Cursor comes from the **server response**, never computed on the client from the last item unless the API documents keyset semantics (sort key + id tie-breaker)

### 2.3 Flatten, dedupe, keys

- [ ] Flatten with `useMemo` over `data.pages`, **dedupe by ID** (`Map`/`Set`) keeping the first occurrence
- [ ] Or use `select` on the query to do it once per data change (memoize `select` with `useCallback`, or define it outside the component)
- [ ] `keyExtractor={(item) => item.id}` — stable, unique, a string. **Never the index** (breaks recycling, swipe state, animations, and the scroll position on inserts)
- [ ] Composite keys for mixed types: `` `${item.type}:${item.id}` ``

### 2.4 Cancellation & races

- [ ] Pass TanStack's `signal` (AbortSignal) to `fetch`/axios. The query cancels on unmount and on key change, and `queryClient.cancelQueries` can cancel it too
- [ ] Hand-rolled fetching (no TanStack): one `AbortController` per request; abort the previous one when filters change; ignore responses whose request ID ≠ the latest
- [ ] Don't fire `fetchNextPage` while a refetch/refresh is in flight (see §6.3)

### 2.5 Merge while keeping the scroll position

- [ ] Appending pages must not change the keys of earlier items. With stable keys, FlashList/FlatList keep the offset
- [ ] Refetches replace pages in place. Structural sharing (on by default) keeps unchanged item references, so memoized rows don't re-render
- [ ] Prepending (realtime, "load newer") needs `maintainVisibleContentPosition` (§9.3)

## 3. Load more

- [ ] `onEndReached={loadMore}` with `onEndReachedThreshold={0.5}` (half a viewport of distance left). For fast scrollers or slow APIs, use 1–2
- [ ] **Guard**: `if (hasNextPage && !isFetchingNextPage && !isFetching && !isError) fetchNextPage()`. TanStack dedupes concurrent `fetchNextPage` calls by default (`cancelRefetch` is false for it), but the guard also stops loops after an error
- [ ] **onEndReached fires on mount** when the content is shorter than the viewport, and can fire repeatedly. The guard handles that. If the API's first page is too short to fill the screen, auto-loading the next page is fine but must stop at `hasNextPage === false`
- [ ] FlatList's old `onMomentumScrollBegin` trick (only allow load-more after the user scrolls) is a workaround for poor guards. Prefer correct guards
- [ ] Don't load the next page while the first page is still in the initial-loading state
- [ ] **Footer states** (`ListFooterComponent`): loading (`ActivityIndicator`) / error ("Couldn't load more" + **Retry** → `fetchNextPage()`) / end ("You're all caught up", or nothing for short lists) / idle (nothing, or a fixed-height spacer)
- [ ] When the next page fails, **keep the loaded rows**. Show the error in the footer only, not in a full-screen error
- [ ] **Manual "Load more" button** instead of infinite scroll when a footer must stay reachable (site footer on web), for result sets where the user benefits from a pause, or for accessibility-heavy flows
- [ ] Bidirectional (chat): `onStartReached` / `onStartReachedThreshold` (FlashList v2, and FlatList in newer RN versions — verify) with `fetchPreviousPage`, or an inverted list (`inverted`) where "end" is the top
- [ ] Screen reader users: announce "Loaded 20 more items" with `AccessibilityInfo.announceForAccessibility` (throttled)

## 4. Loaders

### 4.1 Which loader where

| Situation | Loader |
|---|---|
| **Primary** list, first load, no cached data | **Shimmer skeleton rows** that match the real row layout (thumb + 2 text lines…), enough to fill the viewport |
| Primary list, first load, **cached/persisted data present** | Show the cache right away; refresh quietly (optional thin progress bar or header spinner) |
| **Secondary** list, first load | Small centered `ActivityIndicator` (or inline in its section) |
| Load more | `ActivityIndicator` in the footer |
| Pull to refresh | Native `RefreshControl` spinner only |
| Filter/sort change, old results still on screen | Keep the old rows (dimmed or with a top progress bar) via `placeholderData: keepPreviousData`, or a skeleton if results must not be misleading |

- [ ] **Never shimmer over existing content**. Skeletons are only for the "no data yet" state
- [ ] Skeleton rows use the **same height, padding and thumbnail size** as real rows, so nothing jumps when data arrives. Render them as a plain `View` with `map` (no virtualization needed for ~8 rows), placed where the list will appear and below the header
- [ ] Skeleton has `accessible` + `accessibilityLabel="Loading"` on the container, and the individual bones are hidden from accessibility (`importantForAccessibility="no-hide-descendants"` on Android, `accessibilityElementsHidden` on iOS)

### 4.2 Shimmer implementation options

1. **Reanimated + expo-linear-gradient** (recommended): a gradient band translated across each bone with `withRepeat(withTiming(...))`. Runs on the UI thread. See Appendix A.3
2. **moti** `Skeleton` (`moti/skeleton`): quick and themeable. Check that the moti version supports your Reanimated version (Reanimated 4 support is the thing to check)
3. A plain **pulse** (opacity 0.4 ↔ 1). Cheapest option, and the fallback when reduce motion is on
- [ ] One shared animation clock (one shared value provided via context) so all bones shimmer in sync, rather than N independent loops
- [ ] Reduce motion on (`useReducedMotion()` from Reanimated, or `AccessibilityInfo.isReduceMotionEnabled()`) → static bones, no shimmer

### 4.3 Avoid flicker

- [ ] **Delay** showing any loader ~150–300 ms. Fast responses (cache, LAN) then never flash a skeleton
- [ ] Once shown, a loader stays at least ~300–500 ms (**minimum display time**) so it doesn't blink
- [ ] Footer spinner: reserve its height, or the list jumps when it mounts and unmounts
- [ ] Use `isPending` (no data), not `isFetching` (any fetch, including background ones), to pick the skeleton

## 5. Swipe actions

- [ ] Use **react-native-gesture-handler**'s `ReanimatedSwipeable` (`import ReanimatedSwipeable from 'react-native-gesture-handler/ReanimatedSwipeable'`). The legacy `Swipeable` (Animated-based) is deprecated. The app root must be wrapped in `GestureHandlerRootView`
- [ ] `renderLeftActions` (leading, e.g. mark read / pin) and `renderRightActions` (trailing, e.g. archive / delete). They receive `(progress, translation, swipeableMethods)` as Reanimated SharedValues, so animate the action icons with `useAnimatedStyle`
- [ ] Keep the action set consistent across rows; hide actions that don't apply. Use at most 2–3 per side
- [ ] **Full swipe** (swipe past a threshold to run the primary action): use `onSwipeableWillOpen` / `onSwipeableOpen(direction)` together with a wide action panel, or measure `translation` against a threshold. Fire a haptic when the threshold is crossed (`Haptics.impactAsync(Medium)`) and again on commit
- [ ] **Destructive** actions (delete): either confirm first, or (better) remove the row optimistically and show an **undo toast** (~5 s). Commit to the server on timeout, or send it right away and offer undo as a reverse mutation
- [ ] **Haptics**: `expo-haptics`. Light `selectionAsync` when an action panel reveals, `notificationAsync(Success)` on completion. Respect platform norms; don't buzz on every frame
- [ ] **Close other rows** when one opens: keep a ref to the currently open swipeable (`SwipeableMethods`) and call `close()` on the previous one in `onSwipeableWillOpen`. Also close all on scroll start (`onScrollBeginDrag`) and after an action
- [ ] **FlashList recycling**: swipe state belongs to the view, not to the item. Reset/close the swipeable when the item ID changes (key it with `key={item.id}` inside the row, or close it in an effect on `item.id`), or a recycled cell shows up already open
- [ ] Gesture conflicts: horizontal swipe vs vertical scroll vs back-swipe (iOS edge) vs a horizontal carousel inside the row. Tune `dragOffsetFromLeftEdge`/`dragOffsetFromRightEdge`, and use `simultaneousWithExternalGesture` only when you need it
- [ ] **Accessibility alternative is required**: swipe gestures are invisible to screen readers. Add `accessibilityActions={[{name:'archive', label:'Archive'}, {name:'delete', label:'Delete'}]}` + `onAccessibilityAction` on the row. Also offer a long-press context menu (e.g. `react-native-context-menu-view`, `zeego`, or `@expo/ui` menus) or an overflow "…" button
- [ ] On web, swipe isn't discoverable: show hover buttons or an overflow menu

## 6. Pull to refresh

- [ ] FlashList/FlatList/SectionList: `refreshing={isRefreshing}` + `onRefresh={onRefresh}` (or `refreshControl={<RefreshControl tintColor colors progressViewOffset/>}` for styling)
- [ ] `isRefreshing` is **user-initiated** state: set it true in `onRefresh`, false when `refetch()` settles. Don't bind it to `isFetching`, or background refetches pop the spinner up unprompted
- [ ] **Keep the data** during refresh. No skeleton, no clearing the list
- [ ] Refresh refetches all loaded pages in order (TanStack default). For long feeds where you want the first page only, `queryClient.resetQueries({ queryKey })` (shows the loading state) or `setQueryData` to trim to `pages.slice(0,1)` and then refetch, or `maxPages`
- [ ] **Race with fetchNextPage**: while a refresh is in flight, block load-more (guard on `isRefetching`/`isFetching`). TanStack's `refetch()` defaults to `cancelRefetch: true`, which cancels an in-flight fetch, but test the "refresh while the footer spinner is showing" case
- [ ] Refresh failure: keep the old data and show a toast/banner ("Couldn't refresh"), not the full-screen error
- [ ] Android `progressViewOffset` when there's a translucent header; iOS `contentInsetAdjustmentBehavior="automatic"` with large titles (native-stack header) so the spinner sits correctly

## 7. Empty, error & offline states

- [ ] `ListEmptyComponent` renders whichever state applies: **skeleton** (pending), **error** (initial load failed: message + Retry), **empty** (success with 0 items), **filtered-empty** ("No results for 'foo'" + Clear filters), **offline-empty** (no cache + offline). Make it fill the space (`flexGrow: 1` contentContainerStyle on FlatList; on FlashList v2 check how empty-state sizing works)
- [ ] Empty copy is specific and has a next action ("No messages yet. Start a conversation")
- [ ] Error copy is human-readable; map known error types (offline, 401 → re-auth, 403, 5xx, timeout). Don't show raw messages
- [ ] Initial error vs load-more error vs refresh error are **three different UIs** (full screen / footer / toast)
- [ ] **Persisted cache**: `@tanstack/react-query-persist-client` + `createAsyncStoragePersister` (or MMKV via `@tanstack/query-sync-storage-persister`) with `PersistQueryClientProvider`. Set a `buster` (app version) and `maxAge`. Persist only list queries (`dehydrateOptions.shouldDehydrateQuery`). Don't persist sensitive data unencrypted. Set `gcTime` ≥ `maxAge`, or entries are dropped before they're persisted
- [ ] **Offline banner** above the list when showing cached data offline: "You're offline. Showing saved items." Include a last-updated time if known (`dataUpdatedAt`)
- [ ] **onlineManager + NetInfo**: `onlineManager.setEventListener(setOnline => NetInfo.addEventListener(s => setOnline(!!s.isConnected)))`. Queries then pause offline and **auto-retry on reconnect**. Also wire `focusManager` to `AppState` so lists refetch when the app returns to the foreground
- [ ] `networkMode` (default `'online'`) means queries don't even start offline: `fetchStatus: 'paused'`. Handle that state (show the offline-empty state, not an endless skeleton)
- [ ] Mutations offline: `networkMode: 'offlineFirst'` or paused mutations with `resumePausedMutations` after hydration, if the product needs offline edits

## 8. Thumbnails & image caching

### 8.1 expo-image (preferred)

- [ ] `<Image source={{ uri }} style={{ width, height }} contentFit="cover" cachePolicy="memory-disk" transition={150} placeholder={{ blurhash }} recyclingKey={item.id} />`
- [ ] **`recyclingKey`** is required in FlashList/recycled rows. Without it, the previous item's image flashes in a recycled cell before the new one loads
- [ ] `placeholder`: blurhash/thumbhash from the API (cheap, looks good) or a neutral color. `placeholderContentFit` to match
- [ ] `transition` short (100–200 ms), or 0 with reduce motion on. Skip the fade for memory-cache hits if possible (check current docs; behaviour has changed between versions)
- [ ] Fixed-size image boxes (explicit width/height or aspectRatio) so rows don't reflow when images load
- [ ] `priority="high"` for above-the-fold items, `"low"` for prefetches (verify support)
- [ ] Failure: `onError` → show a fallback icon/initials. Don't retry in a loop
- [ ] Accessibility: decorative thumbnails get `accessible={false}`; meaningful images get `accessibilityLabel`

### 8.2 Alternatives

- `react-native-fast-image` is unmaintained; on the New Architecture use `@d11/react-native-fast-image` (community fork) or move to expo-image (works in bare RN via `expo` modules)
- `react-native-nitro-image` / other Nitro-based libs: newer and fast; check maturity
- Core `Image`: OK for a few static assets; for remote lists it has weaker caching (OS HTTP cache) and no recycling key

### 8.3 Size images correctly

- [ ] Ask the server/CDN for a **thumbnail at display size × `PixelRatio.get()`** (rounded up to a bucket, e.g. 64/128/256/512, for better CDN cache hits). Example: 56 pt avatar on a 3x device → request 168 → bucket 256. Never download a 4000 px original for a 56 pt thumbnail
- [ ] `PixelRatio.getPixelSizeForLayoutSize(56)` helps; include `?w=&h=&fit=cover&format=webp/avif` per CDN (Cloudinary, imgix, Cloudflare Images, Supabase transforms…)
- [ ] Stable URLs (no per-request signed query strings that change every fetch) or the cache never hits. If URLs must be signed, cache by a stable key, or use long-lived signatures

### 8.4 Prefetch, clearing, video

- [ ] Prefetch the next few thumbnails: `Image.prefetch(urls, 'memory-disk')` (expo-image) when a page arrives, or from `onViewableItemsChanged` for items about 1–2 screens ahead. Cap concurrency; skip on cellular data saver if you detect it
- [ ] Clearing: `Image.clearMemoryCache()` on memory warnings (`AppState` `memoryWarning` on iOS), `Image.clearDiskCache()` on logout (privacy) or from a Settings "Clear cache" button. Verify current method names
- [ ] **Video thumbnails**: prefer a server-generated poster image. Client-side: `expo-video-thumbnails` `getThumbnailAsync(uri, { time })` → cache the resulting file URI by video ID. Never generate thumbnails for every row on the JS thread during a scroll; queue and throttle the work, and generate only for visible items
- [ ] Don't autoplay videos in rows unless they're visible (`onViewableItemsChanged` with `itemVisiblePercentThreshold: 60`) and muted; pause on blur

## 9. List mutations

### 9.1 Optimistic updates (TanStack Query)

- [ ] `useMutation({ mutationFn, onMutate, onError, onSettled })`:
  - `onMutate`: `await queryClient.cancelQueries({ queryKey })`, snapshot `getQueryData`, then `setQueryData` to update/remove the item **inside `pages[].items`** (immutably), and return `{ previous }`
  - `onError`: restore `previous`, then a toast ("Couldn't archive. Try again")
  - `onSettled`: `invalidateQueries({ queryKey })`. For long infinite lists, consider invalidating less (refetching every page is expensive) and trusting the server response
- [ ] Update **every cache that contains the item** (list with other filters, detail query, counts/badges) or invalidate them
- [ ] Alternative (v5): render pending mutations from `useMutationState` / `variables` without touching the cache. Simpler rollback, good for "adding" items
- [ ] Insert new items at the right position (by sort key), not always at the top
- [ ] Undo: keep the removed item + index; restore by re-inserting into the same page

### 9.2 Animations

- [ ] Reanimated **layout animations**: `entering={FadeIn}`, `exiting={FadeOut}`, `itemLayoutAnimation={LinearTransition}` on `Animated.FlatList`. On FlashList, layout animations on recycled cells are problematic. Check the version's guidance (v1: `prepareForLayoutAnimationRender()` before the data change; v2 notes differ)
- [ ] `LayoutAnimation.configureNext` (core) works for simple cases; on Android old-arch it needed `UIManager.setLayoutAnimationEnabledExperimental`
- [ ] Skip animations for bulk changes (>~10 rows) and when reduce motion is on

### 9.3 Realtime inserts

- [ ] `maintainVisibleContentPosition={{ minIndexForVisible: 0, autoscrollToTopThreshold: 10 }}` keeps the reader in place when items are prepended. FlatList: supported on iOS for years, on Android since RN 0.72 (verify). FlashList v2: enabled by default (configurable via `maintainVisibleContentPosition` prop); v1 lacked it
- [ ] If the user is scrolled down, **don't auto-scroll**. Buffer the inserts and show a **"N new items" pill** (§14). If they're at the top, insert and stay at the top
- [ ] Dedupe realtime events against fetched pages by ID (the same item can arrive via both socket and refetch)
- [ ] Chat: an `inverted` list (or FlashList v2 `maintainVisibleContentPosition={{ startRenderingFromBottom: true }}` — verify) handles prepend/append naturally

## 10. Performance

### 10.1 FlashList

- [ ] **v1**: `estimatedItemSize` is required and should be close to the average row height (the dev warning suggests a value). Size changes need `overrideItemLayout` for grids/spans
- [ ] **v2** (requires New Architecture): `estimatedItemSize` and the other estimate props are **removed**; sizes are measured automatically. Adds `masonry` prop (replaces MasonryFlashList), `maintainVisibleContentPosition` default on, `onStartReached`, `useRecyclingState`/`useLayoutState` hooks, ref method changes. Check the migration guide before editing an existing v1 list
- [ ] `getItemType={(item) => item.kind}` for heterogeneous rows (header, card, ad) so cells recycle only into compatible types
- [ ] Rows **must not hold local state keyed to the item** with `useState`. A recycled cell keeps the old state. Reset it on ID change, lift it into the item/store, or use `useRecyclingState` (v2)
- [ ] Don't set `key` props on the row root (that defeats recycling), except for the deliberate reset trick on a child component
- [ ] `drawDistance` only if measurement shows blank areas; don't increase it blindly

### 10.2 FlatList / SectionList

- [ ] `getItemLayout` for fixed-height rows (big win; enables `scrollToIndex`)
- [ ] `windowSize` (default 21; lower = less memory, more blanking), `initialNumToRender` ≈ one screen, `maxToRenderPerBatch`, `updateCellsBatchingPeriod`
- [ ] `removeClippedSubviews` can help on Android for long lists, but causes missing-content bugs on iOS and with sticky headers. Measure first
- [ ] Above ~200 rows or with heavy rows, switch to FlashList

### 10.3 Rows & renderItem

- [ ] Rows are `React.memo` components with primitive or stable props. `renderItem` is defined with `useCallback` (or outside the component), **not an inline anonymous function that builds new objects/handlers** per render
- [ ] Pass `onPress` handlers that take the ID (`onPress={onOpen}`, row calls `onOpen(item.id)`) instead of creating a closure per row
- [ ] `extraData` (FlatList) when rows depend on external state (selection), or the list won't re-render. FlashList also uses `extraData`
- [ ] No heavy work in render (date formatting → precompute in `select` or memoize; markdown parsing → cache)
- [ ] Avoid `Animated`/Reanimated work per row on mount; avoid shadows with no `backgroundColor` (expensive on iOS) and elevation overuse on Android
- [ ] Images sized and cached (§8); text with `numberOfLines`

### 10.4 Measure

- [ ] Test in **release builds** (`expo run:ios --configuration Release`, `--variant release`). Dev mode is 2–5× slower and misleading
- [ ] Hermes on (default). React Native **Perf Monitor** (Dev Menu) for UI/JS FPS; React DevTools Profiler for re-renders (enable "highlight updates")
- [ ] **Flashlight** (`flashlight measure` / `flashlight test`) for Android perf scores in CI; FlashList's `useBenchmark` / `onBlankArea` to measure blank cells
- [ ] Low-end Android device in the matrix (2–3 GB RAM), not just a recent iPhone

## 11. Search, filter & sort

- [ ] Debounce text input 250–400 ms (`useDebouncedValue`). Filters/sorts from taps apply immediately
- [ ] Filters live **in the query key**: `['items', { q, status, sort }]`. A key change = a new infinite query that starts from page one, so paging resets automatically. Never reuse the old cursor with new filters
- [ ] `placeholderData: keepPreviousData` keeps old results visible (dimmed) while the new ones load, which avoids flashing empty. Or show a skeleton if old results would mislead
- [ ] Previous request is cancelled via `signal` when the key changes. Stale responses can't overwrite new ones (TanStack keys handle this; hand-rolled code needs request IDs)
- [ ] Scroll to top when filters change (`listRef.current?.scrollToOffset({ offset: 0, animated: false })`)
- [ ] Min query length (e.g. 2 chars); empty query → show recent/suggested items, not a request
- [ ] Persist the filters in route params (Expo Router search params) so back/deep links restore them
- [ ] Client-side filtering only for small, fully loaded datasets; otherwise the server filters (filtering loaded pages only gives wrong results)

## 12. Selection & bulk actions

- [ ] Selection state: `Set<string>` of IDs (not indexes, not item objects), in a store or `useState` at screen level; pass `isSelected` as a primitive prop and add it to `extraData`
- [ ] Enter selection mode via long-press or an "Edit/Select" button; show checkboxes; header shows "3 selected", Select all, Cancel; bulk action bar at the bottom (safe-area aware)
- [ ] "Select all" in a paged list: define it as "all loaded" or "all matching on the server" (send the filter, not IDs). Make it explicit in the UI
- [ ] Swipe actions are disabled in selection mode
- [ ] Back button / Android back exits selection mode first (`BackHandler`, or `usePreventRemove`/`beforeRemove` in React Navigation)
- [ ] Prune IDs from the selection when items disappear (refresh, delete)
- [ ] Bulk mutations: optimistic for all, roll back individually on partial failure, and show a summary toast ("Archived 8, 2 failed")
- [ ] Accessibility: `accessibilityState={{ selected, checked }}`, role `checkbox` in selection mode, and announce the count changes

## 13. Sections & sticky headers

- [ ] `SectionList` with `sections=[{ title, data }]`, `renderSectionHeader`, `stickySectionHeadersEnabled` (default true on iOS, false on Android)
- [ ] FlashList: flatten into one array with header items (`{ kind: 'header', id: 'h:2024-05' }`), use `getItemType`, and compute `stickyHeaderIndices` with `useMemo`. Header rows need opaque backgrounds
- [ ] Grouping pages by date: a section can span two pages, so group **after** flattening, never per page
- [ ] Section index / alphabet scrubber: needs `getItemLayout` (FlatList) or index math + `scrollToIndex`
- [ ] Headers get `accessibilityRole="header"`

## 14. Scroll behaviour

- [ ] **Restore position**: native stacks keep the previous screen mounted, so position is preserved automatically on back. Don't remount lists on focus (avoid `key` changes, and avoid `unmountOnBlur` on tabs)
- [ ] Across app restarts (rarely needed): store the first visible item ID and scroll to its index after data loads. Offsets break when content changes
- [ ] **`useScrollToTop(ref)`** from `@react-navigation/native` so tapping the active tab scrolls to top (Expo Router tabs support it as well). For FlashList, pass a ref that exposes `scrollToOffset`
- [ ] **"N new items" pill**: when realtime/polling finds new items and the user isn't at the top, show a floating pill; tap → scroll to top (and merge the buffered items); auto-hide when they reach the top. Announce it for screen readers
- [ ] `keyboardDismissMode="on-drag"` and `keyboardShouldPersistTaps="handled"` for lists with search fields
- [ ] Safe area / tab bar insets: `contentContainerStyle={{ paddingBottom: insets.bottom + tabBarHeight }}` or `contentInsetAdjustmentBehavior="automatic"` with native headers
- [ ] `scrollToIndex` needs measured/estimated layouts. Handle `onScrollToIndexFailed` on FlatList

## 15. Accessibility

- [ ] Each row is **one accessible element**: `accessible`, `accessibilityRole="button"` (or `link`), a combined `accessibilityLabel` ("Anna Smith, Lunch tomorrow?, 2 hours ago, unread"), `accessibilityHint` for non-obvious actions
- [ ] Swipe actions exposed via `accessibilityActions` (§5); selection via `accessibilityState`
- [ ] **Font scaling**: rows grow with Dynamic Type / font scale. No fixed heights on text containers; test at the largest sizes (iOS AX5, Android 200%). `maxFontSizeMultiplier` only on tight chrome, never on body text. FlatList `getItemLayout` with fixed heights breaks at large font sizes, so compute heights from the font scale or don't use it
- [ ] **Reduce motion**: `useReducedMotion()` (Reanimated) or `AccessibilityInfo.isReduceMotionEnabled()` + the `reduceMotionChanged` listener → no shimmer (static/pulse-free bones), no image fade, no layout animations
- [ ] Loading/empty/error states are announced (`accessibilityLiveRegion="polite"` on Android; `announceForAccessibility` on iOS)
- [ ] Touch targets ≥ 44×44 pt (iOS) / 48×48 dp (Android), including checkboxes and the overflow button
- [ ] Color contrast for secondary text and skeleton bones; unread state not shown by color alone (dot + bold + label)
- [ ] Test with VoiceOver and TalkBack: focus order, focus kept after deleting a row, the footer Retry button is reachable

## 16. Testing

### 16.1 Tools

1. **React Native Testing Library** (`@testing-library/react-native`) + Jest with a fresh `QueryClient` per test (`retry: false`, `gcTime: Infinity`), and **MSW** or mocked fetchers for API pages
2. FlashList in Jest: it renders in tests with a mocked layout (see FlashList's jest setup docs); otherwise fire `onEndReached` via `fireEvent(list, 'onEndReached')` or `fireEvent.scroll`
3. Gesture Handler: `jestSetup` from `react-native-gesture-handler/jestSetup`; Reanimated: `react-native-reanimated` mock / `setUpTests()`; trigger swipeable actions by pressing the rendered action buttons or via `accessibilityActions` (`fireEvent(row, 'accessibilityAction', { nativeEvent: { actionName: 'delete' } })`)
4. E2E: **Maestro** (simple YAML flows, scroll/swipe) or Detox; run on a slow-network profile
5. Perf: Flashlight on Android release builds; manual Perf Monitor on iOS

### 16.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| L1 | First load, slow network | Primary list: skeleton after the delay, matches row layout, no flash on fast responses; secondary list: small spinner |
| L2 | First load fails / zero items | Full-screen error with working Retry / empty state with action |
| L3 | Scroll to end | Exactly one next-page request; footer spinner; rows appended, no jump; at the end "caught up" footer and no more requests |
| L4 | Short first page (content < viewport) | Loads more until filled or end; no loop at end |
| L5 | Next page fails | Rows kept; footer error + Retry; no auto-retry loop |
| L6 | Duplicate items across pages (offset API + insert) | No duplicate rows / key warnings |
| L7 | Pull to refresh | Data stays; spinner only while user-triggered; updated rows |
| L8 | Refresh while loading more | No duplicated/lost pages; consistent order |
| L9 | Refresh fails | Old data + toast |
| L10 | Offline cold start | With persisted cache: cached rows + offline banner, auto-refresh on reconnect; without: offline-empty state, not infinite skeleton |
| L11 | Swipe trailing → delete | Row removed optimistically, undo toast restores in place; server failure restores row + error toast |
| L12 | Open swipe on row A then row B; fast scroll after (FlashList) | A closes; no recycled row appears pre-opened |
| L13 | Screen reader: row actions | Actions available via rotor/actions menu |
| L14 | Search typing | Debounced; old request cancelled; paging reset; scroll to top |
| L15 | Filter change → back → forward | Filters restored from params |
| L16 | Fast scroll through thumbnails | Correct image per row (no recycled-image flash); placeholders; memory stable |
| L17 | Realtime insert while scrolled down | Position kept; "new items" pill |
| L18 | Tap tab while on list / detail → back | Scrolls to top / scroll position and data unchanged, no refetch flash |
| L19 | Selection mode + refresh | Selection pruned to existing items; bulk bar count correct |
| L20 | Largest font size; reduce motion | Rows wrap without clipping; no shimmer/fade/layout animations |
| L21 | Low-end Android, release build | ~60 fps scroll, minimal blank cells |
| L22 | Logout | Caches (query + image disk) cleared |

### 16.3 RNTL examples

Sketches (RNTL + fresh `QueryClient` wrapper, `fetchPage`/`deleteItem` injected as `jest.fn()`):
- **Load more once**: two mocked pages (cursor then `null`); after `findByText('Item 0')` fire `onEndReached` on `inbox-list` twice → `fetchPage` called exactly 2 times, "You're all caught up" shown.
- **Footer retry**: page 1 resolves, page 2 rejects then resolves; `onEndReached` → press `Retry` → new rows appear and `Item 0` is still on screen.
- **Optimistic delete rollback**: `deleteItem` rejects; fire `accessibilityAction` `{ actionName: 'delete' }` on the row → row disappears, then is restored with a "Couldn't delete" toast.

## 17. Common bugs to look for in review

- `keyExtractor` using the index, or missing (FlatList falls back to `item.key`/index) → wrong rows animate, swipe state jumps, duplicates
- `onEndReached` without a guard → multiple identical page requests, or an infinite loop after the last page/error
- `refreshing={isFetching}` → the spinner pops up on background refetches, focus refetches, and load-more
- Skeleton shown whenever `isFetching` → shimmer covers real data on refresh/filter change
- Full-screen error replaces loaded rows when page N fails
- Offset paging without dedupe → duplicate keys warning and repeated rows after inserts
- Filters not in the query key → stale cursor reused; results mix across filters
- No `signal` passed → cancelled searches still resolve and burn bandwidth
- Inline `renderItem={({item}) => <Row item={item} onPress={() => open(item.id)} />}` with non-memo rows → every row re-renders on every state change
- FlashList v1 missing `estimatedItemSize`, or a v1-only prop left after upgrading to v2 (TS errors or silent no-op)
- Row `useState` with FlashList → recycled rows show another item's expanded/liked/swiped state
- expo-image without `recyclingKey` in FlashList → wrong thumbnail flashes; full-resolution images in thumbnails → memory spikes and OOM on Android
- Signed image URLs that change every fetch → cache never hits, images reload on every refresh
- Swipeable rows never closed on recycle/scroll; multiple rows open at once; no accessibility alternative
- Delete without undo or confirmation; optimistic update that only touches one of several caches containing the item
- `invalidateQueries` on a 30-page infinite list after every tiny mutation → 30 requests
- Persisted cache without `buster` → app update crashes on an old data shape; sensitive data persisted in plain AsyncStorage
- `onlineManager` not wired to NetInfo → no auto-retry on reconnect; `fetchStatus: 'paused'` shows an endless skeleton
- `removeClippedSubviews` on iOS → blank rows; `getItemLayout` with fixed heights + large fonts → overlap
- `maintainVisibleContentPosition` missing for prepends → reader jumps; auto-scroll to top on every realtime insert
- List remounted on focus (`key={Date.now()}`, `unmountOnBlur`) → scroll position lost and a refetch flash
- Perf judged in debug builds only

## 18. Output format

When asked to **review/audit**:
1. **Summary**: libraries + versions (FlashList v1/v2, Query v4/v5, Reanimated, RNGH, expo-image), New Architecture on/off, the list spec table (§1) for each list screen
2. **Checklist results**: sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check, with file:line
3. **Critical issues first** (duplicate/missing rows, request loops, data loss on error, wrong images, crashes/OOM, inaccessible actions) with fix code
4. **Testing gaps**: L1–L22 not covered, plus suggested RNTL tests

When asked to **implement**: confirm the versions, start from the Appendix, adapt the API contract, row layout and actions, and remove the parts the product doesn't need (swipe, selection, realtime). Then list what to verify by hand: release-build perf on a low-end Android device, VoiceOver/TalkBack, offline, and the largest font size.

When asked to **debug**: reproduce with the scenario from §16.2 closest to the symptom, check §17, and log `status`/`fetchStatus`/`isFetchingNextPage`/`hasNextPage` transitions alongside the `onEndReached` calls.

---

## Appendix — Reference implementation (Expo SDK 53+, New Architecture, TypeScript)

Assumes `@shopify/flash-list` v2, `@tanstack/react-query` v5, `react-native-reanimated` 3.17+/4, `react-native-gesture-handler` 2.2x, `expo-image`, `expo-linear-gradient`, `expo-haptics`, `@react-native-community/netinfo`. Verify each API against the installed versions; for FlashList v1, add `estimatedItemSize`.

### A.1 Types, API & list state machine

```ts
// list/types.ts
export type Item = {
  id: string; title: string; subtitle: string;
  thumbUrl: string | null; blurhash?: string | null;
  unread: boolean; createdAt: string;
};

export type Page = { items: Item[]; nextCursor: string | null };

// fetchItemsPage({ cursor, q, limit }, signal?): GET /items with cursor/q/limit query params,
// passes the AbortSignal to fetch, throws HttpError(status) when !res.ok, returns res.json() as Page.
export declare function fetchItemsPage(
  params: { cursor: string | null; q: string; limit: number }, signal?: AbortSignal): Promise<Page>;
export class HttpError extends Error {
  constructor(public status: number) { super(`HTTP ${status}`); }
}

// list/listState.ts — the single derived UI state
export type ListState<T> =
  | { kind: 'idle' }                                      // disabled query / waiting for input
  | { kind: 'loadingInitial'; offline: boolean }          // no data yet
  | { kind: 'loaded'; items: T[]; refreshing: boolean }
  | { kind: 'loadingMore'; items: T[] }
  | { kind: 'error'; items: T[]; error: unknown; scope: 'initial' | 'more' }
  | { kind: 'endReached'; items: T[]; refreshing: boolean };

export function deriveListState<T>(q: {
  enabled: boolean; items: T[] | undefined; isPending: boolean;
  fetchStatus: 'fetching' | 'paused' | 'idle'; isError: boolean; isFetchNextPageError: boolean;
  error: unknown; isFetchingNextPage: boolean; hasNextPage: boolean; isUserRefreshing: boolean;
}): ListState<T> {
  if (!q.enabled) return { kind: 'idle' };
  if (q.isPending || !q.items) {
    if (q.isError) return { kind: 'error', items: [], error: q.error, scope: 'initial' };
    return { kind: 'loadingInitial', offline: q.fetchStatus === 'paused' };
  }
  const items = q.items;
  if (q.isFetchNextPageError) return { kind: 'error', items, error: q.error, scope: 'more' };
  if (q.isFetchingNextPage) return { kind: 'loadingMore', items };
  if (!q.hasNextPage) return { kind: 'endReached', items, refreshing: q.isUserRefreshing };
  // A failed background refetch keeps the data: stays 'loaded'; show a toast elsewhere.
  return { kind: 'loaded', items, refreshing: q.isUserRefreshing };
}
```

### A.2 useInfiniteQuery hook

```ts
// list/useItemsList.ts
import { useCallback, useMemo, useRef, useState } from 'react';
import { keepPreviousData, useInfiniteQuery, type InfiniteData } from '@tanstack/react-query';
import { fetchItemsPage, type Item, type Page } from './types';
import { deriveListState } from './listState';

const PAGE_SIZE = 30;
export const itemsKey = (q: string) => ['items', { q }] as const;

function flattenDedupe(data: InfiniteData<Page>): Item[] {
  const seen = new Set<string>();
  const out: Item[] = [];
  for (const page of data.pages) {
    for (const item of page.items) {
      if (!seen.has(item.id)) { seen.add(item.id); out.push(item); }
    }
  }
  return out;
}

export function useItemsList(q: string) {
  const enabled = q.length === 0 || q.length >= 2;
  const query = useInfiniteQuery({
    queryKey: itemsKey(q),
    queryFn: ({ pageParam, signal }) =>
      fetchItemsPage({ cursor: pageParam, q, limit: PAGE_SIZE }, signal),
    initialPageParam: null as string | null,
    getNextPageParam: (last) => last.nextCursor ?? undefined,
    staleTime: 30_000,
    enabled,
    placeholderData: q ? keepPreviousData : undefined,
  });

  const items = useMemo(
    () => (query.data ? flattenDedupe(query.data) : undefined),
    [query.data],
  );

  // User-initiated refresh flag — never bound to isFetching.
  const [isUserRefreshing, setUserRefreshing] = useState(false);
  const refresh = useCallback(async () => {
    setUserRefreshing(true);
    try { await query.refetch(); } finally { setUserRefreshing(false); }
  }, [query.refetch]);

  // Guard against duplicate/looping onEndReached calls.
  const inFlight = useRef(false);
  const loadMore = useCallback(async () => {
    if (inFlight.current) return;
    if (!query.hasNextPage || query.isFetching || query.isFetchNextPageError) return;
    inFlight.current = true;
    try { await query.fetchNextPage(); } finally { inFlight.current = false; }
  }, [query.hasNextPage, query.isFetching, query.isFetchNextPageError, query.fetchNextPage]);

  const retryMore = useCallback(() => { query.fetchNextPage(); }, [query.fetchNextPage]);

  const state = deriveListState<Item>({
    enabled, items, isPending: query.isPending, fetchStatus: query.fetchStatus,
    isError: query.isError, isFetchNextPageError: query.isFetchNextPageError, error: query.error,
    isFetchingNextPage: query.isFetchingNextPage, hasNextPage: !!query.hasNextPage, isUserRefreshing,
  });

  return {
    state, items: items ?? [], refresh, loadMore, retryMore,
    retryInitial: query.refetch, isPlaceholder: query.isPlaceholderData,
    isRefreshing: isUserRefreshing,
  };
}

// app/_layout.tsx (once) — online/focus managers
import NetInfo from '@react-native-community/netinfo';
import { AppState, Platform } from 'react-native';
import { focusManager, onlineManager } from '@tanstack/react-query';

onlineManager.setEventListener((setOnline) =>
  NetInfo.addEventListener((s) => setOnline(!!s.isConnected)),
);
AppState.addEventListener('change', (status) => {
  if (Platform.OS !== 'web') focusManager.setFocused(status === 'active');
});
```

### A.3 Delayed loader + shimmer skeleton

```tsx
// list/useDelayedFlag.ts — show after `delay`, keep for at least `minDuration`
import { useEffect, useRef, useState } from 'react';

export function useDelayedFlag(active: boolean, delay = 200, minDuration = 400) {
  const [visible, setVisible] = useState(false);
  const shownAt = useRef<number | null>(null);
  useEffect(() => {
    let t: ReturnType<typeof setTimeout> | undefined;
    if (active && !visible) {
      t = setTimeout(() => { shownAt.current = Date.now(); setVisible(true); }, delay);
    } else if (!active && visible) {
      const elapsed = Date.now() - (shownAt.current ?? 0);
      t = setTimeout(() => setVisible(false), Math.max(0, minDuration - elapsed));
    }
    return () => clearTimeout(t);
  }, [active, visible, delay, minDuration]);
  return visible;
}
```

```tsx
// list/Skeleton.tsx — sketch:
// - SkeletonProvider: one shared Reanimated clock (useSharedValue + withRepeat(withTiming(1, 1200ms linear)))
//   in context so every bone animates in sync; provides null when useReducedMotion() → static bones.
// - Bone({ width, height, radius }): grey View (overflow hidden) with an Animated.View translateX band
//   holding an expo-linear-gradient transparent→white(0.45)→transparent sweep.
// - ItemRowSkeleton: MUST mirror ItemRow's layout (56pt thumb bone + 70%/45% text bones, rowStyles).
// - ListSkeleton({ count = 8 }): SkeletonProvider > View accessible accessibilityLabel="Loading"
//   accessibilityRole="progressbar", with the bones hidden from accessibility.
import { StyleSheet } from 'react-native';
export const rowStyles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', paddingHorizontal: 16, paddingVertical: 12, minHeight: 80 },
  text: { flex: 1, marginLeft: 12 },
});
```

Use theme tokens for colors in a real app (dark mode: darker bones, subtler band).

### A.4 Thumbnail row with expo-image

```tsx
// list/ItemRow.tsx
import React, { memo, useMemo } from 'react';
import { PixelRatio, Pressable, StyleSheet, Text, View } from 'react-native';
import { Image } from 'expo-image';
import { useReducedMotion } from 'react-native-reanimated';
import type { Item } from './types';
import { rowStyles } from './Skeleton';

const THUMB = 56;
const BUCKETS = [64, 128, 256, 512];

export function thumbUrl(url: string, sizePt: number) {
  const px = Math.ceil(sizePt * PixelRatio.get());
  const bucket = BUCKETS.find((b) => b >= px) ?? BUCKETS[BUCKETS.length - 1];
  // Adapt to your CDN's transform syntax.
  return `${url}${url.includes('?') ? '&' : '?'}w=${bucket}&h=${bucket}&fit=cover&format=webp`;
}

type Props = { item: Item; onOpen: (id: string) => void; selected?: boolean };

export const ItemRow = memo(function ItemRow({ item, onOpen, selected }: Props) {
  const reduceMotion = useReducedMotion();
  const source = useMemo(
    () => (item.thumbUrl ? { uri: thumbUrl(item.thumbUrl, THUMB) } : null),
    [item.thumbUrl],
  );
  const label = `${item.title}, ${item.subtitle}${item.unread ? ', unread' : ''}`;
  return (
    <Pressable
      onPress={() => onOpen(item.id)}
      accessibilityRole="button"
      accessibilityLabel={label}
      accessibilityState={{ selected }}
      style={({ pressed }) => [rowStyles.row, styles.bg, pressed && styles.pressed]}
    >
      <Image
        source={source}
        recyclingKey={item.id}
        cachePolicy="memory-disk"
        contentFit="cover"
        placeholder={item.blurhash ? { blurhash: item.blurhash } : undefined}
        transition={reduceMotion ? 0 : 150}
        style={styles.thumb}
        accessible={false}
      />
      <View style={rowStyles.text}>
        <Text numberOfLines={1} style={[styles.title, item.unread && styles.unread]}>{item.title}</Text>
        <Text numberOfLines={2} style={styles.subtitle}>{item.subtitle}</Text>
      </View>
      {item.unread && <View style={styles.dot} />}
    </Pressable>
  );
});

// styles: bg, pressed (opacity .7), thumb (56x56, radius 8, grey bg), title, unread (weight 600),
// subtitle, dot (10pt accent circle) — use theme tokens.
declare const styles: Record<'bg' | 'pressed' | 'thumb' | 'title' | 'unread' | 'subtitle' | 'dot', object>;
```

The inline `onPress` closure is fine here because the row is memoized on `item`/`onOpen`; the closure is recreated only when the row re-renders anyway.

### A.5 Swipeable row (close others, haptics, a11y actions)

```tsx
// list/SwipeableRow.tsx
import React, { useCallback, useEffect, useRef } from 'react';
import ReanimatedSwipeable, { type SwipeableMethods } from 'react-native-gesture-handler/ReanimatedSwipeable';
import type { SharedValue } from 'react-native-reanimated';
import * as Haptics from 'expo-haptics';

// Shared "currently open" ref, owned by the screen.
export type OpenRowRef = React.MutableRefObject<SwipeableMethods | null>;

type Props = {
  id: string; openRowRef: OpenRowRef; unread: boolean; disabled?: boolean; children: React.ReactNode;
  onArchive: (id: string) => void; onDelete: (id: string) => void; onToggleRead: (id: string) => void;
};

// Action({ label, color, onPress, drag, side }): Pressable (84pt wide, hidden from accessibility —
// the row exposes accessibilityActions instead) whose label scales 0.6→1 with the drag
// SharedValue via useAnimatedStyle + interpolate(..., Extrapolation.CLAMP).
declare function Action(p: { label: string; color: string; onPress: () => void;
  drag: SharedValue<number>; side: 'left' | 'right' }): React.JSX.Element;

export function SwipeableRow({
  id, openRowRef, onArchive, onDelete, onToggleRead, unread, disabled, children,
}: Props) {
  const ref = useRef<SwipeableMethods>(null);

  // FlashList recycles views: make sure a reused cell starts closed.
  useEffect(() => { ref.current?.close(); }, [id]);

  const run = useCallback((fn: (id: string) => void) => {
    ref.current?.close();
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
    fn(id);
  }, [id]);

  const onWillOpen = useCallback(() => {
    if (openRowRef.current && openRowRef.current !== ref.current) openRowRef.current.close();
    openRowRef.current = ref.current;
    Haptics.selectionAsync();
  }, [openRowRef]);

  const renderLeft = useCallback((_p: SharedValue<number>, drag: SharedValue<number>) => (
    <Action side="left" drag={drag} color="#0A84FF" label={unread ? 'Read' : 'Unread'}
      onPress={() => run(onToggleRead)} />
  ), [unread, run, onToggleRead]);

  const renderRight = useCallback((_p: SharedValue<number>, drag: SharedValue<number>) => (
    <>
      <Action side="right" drag={drag} color="#8E8E93" label="Archive" onPress={() => run(onArchive)} />
      <Action side="right" drag={drag} color="#FF3B30" label="Delete" onPress={() => run(onDelete)} />
    </>
  ), [run, onArchive, onDelete]);

  return (
    <ReanimatedSwipeable
      ref={ref}
      enabled={!disabled}
      friction={2}
      leftThreshold={40}
      rightThreshold={40}
      overshootLeft={false}
      overshootRight={false}
      renderLeftActions={renderLeft}
      renderRightActions={renderRight}
      onSwipeableWillOpen={onWillOpen}
      // Screen readers can't swipe: expose the same actions.
      accessibilityActions={[
        { name: 'toggleRead', label: unread ? 'Mark as read' : 'Mark as unread' },
        { name: 'archive', label: 'Archive' },
        { name: 'delete', label: 'Delete' },
      ]}
      onAccessibilityAction={(e) => {
        const a = e.nativeEvent.actionName;
        if (a === 'toggleRead') onToggleRead(id);
        if (a === 'archive') onArchive(id);
        if (a === 'delete') onDelete(id);
      }}
    >
      {children}
    </ReanimatedSwipeable>
  );
}

```

If the installed RNGH version doesn't forward accessibility props from `ReanimatedSwipeable` to its container, put `accessibilityActions` on the row `Pressable` instead (that's also the element VoiceOver focuses). Full swipe: add a wide action panel and commit in `onSwipeableOpen(direction)`.

### A.6 Optimistic delete with undo

```ts
// list/useDeleteItem.ts
import { useMutation, useQueryClient, type InfiniteData } from '@tanstack/react-query';
import type { Page } from './types';

export function useDeleteItem(deleteItem: (id: string) => Promise<void>, showToast: (t: ToastOpts) => void) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: deleteItem,
    onMutate: async (id: string) => {
      await qc.cancelQueries({ queryKey: ['items'] });
      const snapshots = qc.getQueriesData<InfiniteData<Page>>({ queryKey: ['items'] });
      qc.setQueriesData<InfiniteData<Page>>({ queryKey: ['items'] }, (old) =>
        old && {
          ...old,
          pages: old.pages.map((p) => ({ ...p, items: p.items.filter((i) => i.id !== id) })),
        },
      );
      return { snapshots };
    },
    onError: (_e, _id, ctx) => {
      ctx?.snapshots.forEach(([key, data]) => qc.setQueryData(key, data));
      showToast({ message: "Couldn't delete. Try again." });
    },
    // Don't refetch every page of every list; trust the optimistic state, refresh lazily.
    onSettled: () => qc.invalidateQueries({ queryKey: ['items'], refetchType: 'none' }),
  });
}
type ToastOpts = { message: string; action?: { label: string; onPress: () => void } };
```

For undo, delay `mutate` until the toast times out (keep the removed item + snapshot; Undo restores the snapshot and cancels the timer), or call a reverse "restore" endpoint.

### A.7 FlashList screen

```tsx
// app/(tabs)/inbox.tsx
// imports: react, react-native (ActivityIndicator, Pressable, Text, TextInput, View), FlashList +
// FlashListRef, useScrollToTop, useNetInfo, expo-router `router`, SwipeableMethods, and the
// list/* modules above (useItemsList, useDeleteItem, useDelayedFlag, ListSkeleton, ItemRow,
// SwipeableRow, Item) plus app helpers useDebouncedValue, api, toast.

export default function InboxScreen() {
  const [text, setText] = useState('');
  const q = useDebouncedValue(text.trim(), 300);
  const { state, items, refresh, loadMore, retryMore, retryInitial, isRefreshing } = useItemsList(q);
  const del = useDeleteItem(api.deleteItem, toast.show);
  const { isConnected } = useNetInfo();

  const listRef = useRef<FlashListRef<Item>>(null); // v1: useRef<FlashList<Item>>
  useScrollToTop(listRef as any);
  const openRowRef = useRef<SwipeableMethods | null>(null);

  const showSkeleton = useDelayedFlag(state.kind === 'loadingInitial');

  const onOpen = useCallback((id: string) => router.push(`/items/${id}`), []);
  const onDelete = useCallback((id: string) => del.mutate(id), [del.mutate]);
  const onArchive = useCallback((id: string) => api.archive(id), []);
  const onToggleRead = useCallback((id: string) => api.toggleRead(id), []);

  const renderItem = useCallback(({ item }: { item: Item }) => (
    <SwipeableRow id={item.id} unread={item.unread} openRowRef={openRowRef}
      onArchive={onArchive} onDelete={onDelete} onToggleRead={onToggleRead}>
      <ItemRow item={item} onOpen={onOpen} />
    </SwipeableRow>
  ), [onArchive, onDelete, onToggleRead, onOpen]);

  const Empty = () => {
    switch (state.kind) {
      case 'loadingInitial':
        if (state.offline) return <Message title="You're offline" body="Connect to load your inbox." />;
        return showSkeleton ? <ListSkeleton /> : null;
      case 'error':
        return <Message title="Couldn't load your inbox" body="Check your connection and try again."
          action={{ label: 'Retry', onPress: () => retryInitial() }} />;
      default:
        return q
          ? <Message title={`No results for "${q}"`} action={{ label: 'Clear search', onPress: () => setText('') }} />
          : <Message title="No messages yet" body="New messages will show up here." />;
    }
  };

  // Footer: null when no items; 'loadingMore' → ActivityIndicator (accessibilityLabel "Loading more");
  // 'error' → "Couldn't load more." + Retry (retryMore, hitSlop 12); 'endReached' → "You're all caught up"
  // only when items.length > 10; otherwise an empty footer View that reserves height (avoids jumps).
  const Footer = () => (items.length === 0 ? null : <View style={styles.footer} />); // see comment

  return (
    <View style={styles.screen}>
      {isConnected === false && items.length > 0 && (
        <View style={styles.banner} accessibilityLiveRegion="polite">
          <Text>You're offline. Showing saved items.</Text>
        </View>
      )}
      <FlashList
        ref={listRef}
        testID="inbox-list"
        data={items}
        keyExtractor={(item) => item.id}
        renderItem={renderItem}
        // estimatedItemSize={80}  // FlashList v1 only
        ListHeaderComponent={
          <TextInput value={text} onChangeText={(t) => { setText(t); listRef.current?.scrollToOffset({ offset: 0, animated: false }); }}
            placeholder="Search" accessibilityLabel="Search inbox" style={styles.search}
            autoCorrect={false} clearButtonMode="while-editing" />
        }
        ListEmptyComponent={Empty}
        ListFooterComponent={Footer}
        onEndReached={loadMore}
        onEndReachedThreshold={0.5}
        refreshing={isRefreshing}
        onRefresh={refresh}
        onScrollBeginDrag={() => openRowRef.current?.close()}
        keyboardDismissMode="on-drag"
        keyboardShouldPersistTaps="handled"
        contentInsetAdjustmentBehavior="automatic"
      />
    </View>
  );
}

// Message({ title, body?, action? }): centered View with accessibilityLiveRegion="polite", title with
// accessibilityRole="header", optional muted body and a primary button (accessibilityRole="button").
declare function Message(p: { title: string; body?: string; action?: { label: string; onPress: () => void } }): React.JSX.Element;
// styles: screen, banner, search, footer (minHeight 64, centered row), muted, link — use theme tokens.
declare const styles: Record<'screen' | 'banner' | 'search' | 'footer' | 'muted' | 'link', object>;
```

Notes for adapting:
- For a **secondary** list, replace `<ListSkeleton />` with `<ActivityIndicator style={{ marginTop: 24 }} />`
- `Empty`/`Footer` are defined inline for readability; with heavy screens, hoist them into memoized components that receive `state` as a prop
- Sections: flatten to header + row items, add `getItemType`, and compute `stickyHeaderIndices` (§13)
- Realtime: buffer the inserts when `scrollY > 0`, render a "N new" pill, and on tap merge the buffer with `setQueryData` + `scrollToOffset(0)`
- Persisted cache: wrap the app in `PersistQueryClientProvider` with a `buster` set to the app version (§7)
