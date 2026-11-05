---
name: ios-list-ux
description: Use when implementing, reviewing, or debugging list and feed screens on iOS/iPadOS in SwiftUI (List, LazyVStack, ScrollView) or UIKit (UICollectionView, UITableView, DiffableDataSource, compositional layout) — paginated/batch loading, infinite scroll and load-more footers, shimmer skeletons vs spinners, swipe actions, pull to refresh, empty/error/offline states, thumbnail image caching and downsampling, optimistic mutations, search/filter/sort, multi-select and bulk actions, sticky headers, scroll position, accessibility, and performance. Siblings: android-list-ux, flutter-list-ux, react-native-list-ux.
---

# iOS List UX Skill

Guide for building and auditing **list screens** on Apple platforms: feeds, inboxes, search results, catalogs, settings-style lists. Covers **SwiftUI** (`List`, `LazyVStack`/`LazyVGrid` in `ScrollView`) and **UIKit** (`UICollectionView` + `UICollectionViewDiffableDataSource` + `UICollectionViewCompositionalLayout`, list configuration, legacy `UITableView`). Siblings: **android-list-ux**, **flutter-list-ux**, **react-native-list-ux** — keep product behaviour (paging contract, loader rules, states) consistent across platforms.

SwiftUI list/scroll APIs change every WWDC (`scrollPosition`, `onScrollGeometryChange`, `onScrollVisibilityChange`, `ContentUnavailableView`, `.refreshable` behaviour, Observation). **Verify availability and exact signatures against the current SDK before shipping**, and gate with `if #available` / `@available` when the deployment target is lower.

## 0. Mental model

```
             +-----------------------------+
 UI events → |  ListViewModel (@MainActor) | ← realtime events / mutations
 (appear,    |  state: ListState<Item>     |
  refresh,   |  items: [Item] (unique IDs) |
  reachEnd,  |  cursor / nextPage          |
  retry)     |  inFlight: Task?            |
             +--------------+--------------+
                            | async fetch(page, cursor) — cancellable
                            v
             +-----------------------------+      +------------------+
             | Repository                  | ←──→ | Local cache (DB, |
             | network + cache + dedupe    |      | SwiftData/GRDB)  |
             +-----------------------------+      +------------------+

 Rows render from items; thumbnails go through ImagePipeline
 (memory NSCache → disk → network → downsample off-main → cache).
```

State machine (one source of truth, never parallel booleans like `isLoading && isError`):

```
idle ──appear──▶ loadingInitial ──ok(items)──▶ loaded ──reachEnd──▶ loadingMore ──ok──▶ loaded
                     │   └─ok(empty)──▶ empty                         │  └─ok(last page)─▶ endReached
                     └─fail──▶ error(initial) ──retry──▶ loadingInitial └─fail──▶ loaded + footerError ──retry──▶ loadingMore
 loaded/endReached/empty ──pull──▶ refreshing (items kept) ──ok──▶ loaded (replace) / fail ──▶ loaded + toast
```

## 1. First, establish context

Find out (from the codebase or by asking):

- **Primary or secondary list?** Primary = the screen's main content (home feed, inbox, search results tab) → shimmer skeleton on first load. Secondary = embedded/supporting list (comments under a post, "related items", a picker sheet, a settings sub-list) → plain `ProgressView`. This decides the loader (section 4).
- **Paging contract**: cursor (`next_cursor`, `before_id`), offset/limit, page number, keyset (`created_at,id`), or "everything at once"? Max page size? Is ordering stable while paging? Does the server return `hasMore`/`total`?
- **Volume**: tens, thousands, unbounded? Row heights fixed or self-sizing? Images per row? Grids?
- **Realtime**: are items inserted/updated/removed by push, WebSocket, CloudKit, or polling while the user scrolls? Newest at top (feed) or bottom (chat)?
- **Offline**: must the list work from cache (SwiftData/Core Data/GRDB)? Show stale data with a banner? Queue mutations?
- **Mutations**: delete, archive, mark read, reorder, favorite, edit — optimistic? Undo?
- **Interactions**: swipe actions, context menu, multi-select, drag & drop, search, filters, sort, sections, index.
- **Min OS** (gates below) and UI framework: SwiftUI only, UIKit only, or SwiftUI rows hosted in `UIHostingConfiguration` cells (iOS 16+).
- Platforms: iPhone, iPad (multi-column, pointer, keyboard), Mac Catalyst, visionOS.

Key OS gates (verify): `.refreshable`, `.swipeActions`, `.searchable` iOS 15; `UIHostingConfiguration`, `.scrollDismissesKeyboard`, `NavigationStack` iOS 16; `ContentUnavailableView`, `.scrollPosition(id:)`, `.scrollTargetLayout`, `@Observable`, `.contentMargins`, `.sensoryFeedback` iOS 17; `.scrollPosition(_ ScrollPosition)`, `onScrollGeometryChange`, `onScrollVisibilityChange`, `onScrollPhaseChange` iOS 18. UIKit: diffable data source iOS 13, list configuration / cell registration iOS 14, `reconfigureItems` iOS 15, `UIContentUnavailableConfiguration` iOS 17.

Write a **list spec** before coding:

```
screen      kind       paging          page  realtime    offline   swipe (leading | trailing)      loader
Inbox       primary    cursor(before)  30    push+poll   cache     read/unread | archive, delete   shimmer
Comments    secondary  cursor(after)   20    none        no        — | delete (own)                spinner
```

## 2. Batch data loading

- [ ] **Page size** 20–50 typical; enough to fill ~2–3 screens on the largest device (iPad landscape). Don't fetch 1000 rows "to be safe"; don't fetch 10 and trigger immediate second load.
- [ ] **Prefer cursor/keyset paging** over offset. Offset paging duplicates or skips items when rows are inserted/deleted between pages. If stuck with offset, dedupe and accept gaps, or anchor with `since_id`.
- [ ] **Stable identity**: every item has a server ID used as `Identifiable.id` / diffable identifier. Never `UUID()` at decode time, never array index, never `\.self` on mutable structs.
- [ ] **Dedupe on merge** by ID (keep an `OrderedSet`/`Set<ID>` alongside the array). Duplicate IDs crash `NSDiffableDataSourceSnapshot` ("Fatal: supplied identifiers are not unique") and cause SwiftUI `ForEach` warnings/glitches.
- [ ] **One request in flight per direction**: store the `Task` handle; ignore a load-more trigger while one is running; cancel the previous task when parameters change (search query, filter, refresh).
- [ ] **Cancellation is cooperative**: check `Task.isCancelled`/`try Task.checkCancellation()` after `await` before mutating state; `URLSession` async APIs throw `CancellationError`/`URLError.cancelled` — treat as silent, not an error state.
- [ ] **Generation token** (`requestID` / query snapshot) compared on completion, so a stale response for an old query can't overwrite newer results.
- [ ] **Merge preserving scroll**: appending at the end never moves the viewport. Inserting at the top must not jump (see section 9/14). Replacing on refresh: apply a diff, not a full reload, so unchanged rows keep identity.
- [ ] Decode off the main actor (repository is a non-main actor or `nonisolated` async func); publish to the `@MainActor` view model in one batch.
- [ ] Cache first page locally for instant cold start (stale-while-revalidate).
- [ ] Respect `Retry-After` / backoff on 429/5xx; don't hot-loop retries from `onAppear`.
- [ ] `.task` modifier for initial load (auto-cancels when the view disappears) rather than `onAppear { Task { } }`; guard against reloading on every re-appear (tab switch, navigation pop) — load once, refresh when stale.

## 3. Load more / infinite scroll

- [ ] **Prefetch threshold**: trigger when a row within the last ~5–10 items appears (`onAppear` of that row in SwiftUI; `willDisplay` or `UICollectionViewDataSourcePrefetching` index paths in UIKit), not when the user hits the absolute bottom.
- [ ] iOS 18+: `onScrollGeometryChange` (distance to bottom) or `onScrollVisibilityChange` on a sentinel footer are alternatives; verify behaviour.
- [ ] **Double-trigger guards**: `guard state == .loaded, hasMore, loadMoreTask == nil`. Many rows appear at once (fast scroll, iPad, rotation) and each fires `onAppear`.
- [ ] Footer row states: **loading** (small `ProgressView`), **error + Retry button** ("Couldn't load more. Retry"), **end** (subtle "You're all caught up" or nothing), **hidden** when idle with more available.
- [ ] Load-more failure **keeps all loaded items**; never flips the whole screen into an error state.
- [ ] Don't auto-retry a failed footer in a loop when its row re-appears; require tap or connectivity change.
- [ ] Offer a **manual "Load more" button** where infinite scroll is wrong: footers with important content, accessibility/VoiceOver users, finite result sets, or when the product wants explicit pagination.
- [ ] Short first page that doesn't fill the screen: threshold row is already visible → load next page immediately (once), don't stall.
- [ ] End detection from server (`hasMore`/null cursor), not from `count < pageSize` alone (servers may filter and return short pages).
- [ ] Bidirectional (chat): separate cursors and in-flight guards for older/newer; preserve anchor when prepending.

## 4. Loaders: shimmer vs spinner

Rules:

| Situation | Loader |
|---|---|
| **Primary list, first load, no cached data** | **Shimmer skeleton rows** matching the real row layout |
| Primary list, first load, cached data available | Show cache immediately; subtle refresh indicator only |
| **Secondary list** first load (comments, related, picker) | Plain `ProgressView()` centered in its area |
| **Load more** (any list) | Small `ProgressView` in footer row |
| Pull to refresh | System refresh control only |
| Refresh / filter change with existing content | Keep content (optionally dimmed) + small indicator; **never shimmer over existing content** |

- [ ] Skeleton rows reuse the **real row view** with placeholder data + `.redacted(reason: .placeholder)` so geometry matches exactly (avatar size, line count, thumbnail aspect). Render 6–10 or enough to fill the viewport; non-interactive (`.allowsHitTesting(false)`, `.disabled(true)`).
- [ ] Shimmer = a moving gradient mask over the redacted content (Appendix A.4). One shared animation phase, not per-row timers.
- [ ] Use `.unredacted()` for elements that should stay real (section headers, static labels).
- [ ] **Reduce Motion** (`@Environment(\.accessibilityReduceMotion)` / `UIAccessibility.isReduceMotionEnabled`): no moving shimmer — static redacted placeholders or a gentle opacity pulse.
- [ ] VoiceOver: skeleton container is one element labelled "Loading", individual placeholders hidden (`.accessibilityHidden(true)`).
- [ ] **Flicker avoidance**: don't show any loader if data arrives within ~150–300 ms (delay before showing); once shown, keep it at least ~300–500 ms so it doesn't flash. Cached-first loads should skip the loader entirely.
- [ ] Skeleton → content transition: crossfade (`.transition(.opacity)`), no layout jump; content rows have the same height as skeletons.
- [ ] UIKit: skeleton cells as a separate item kind (`.skeleton(Int)`) in the diffable snapshot, or a dedicated skeleton cell registration; replace with real items in one snapshot apply.
- [ ] Never put a full-screen blocking spinner over a list for row-level actions; show progress on the row.

## 5. Swipe actions

SwiftUI (iOS 15+): `.swipeActions(edge:allowsFullSwipe:)` on rows **inside `List`** (not in `LazyVStack` — there it does nothing; use a context menu or custom gesture). UIKit: `UISwipeActionsConfiguration` + `UIContextualAction` via `UICollectionLayoutListConfiguration.leadingSwipeActionsConfigurationProvider` / `trailing…` or `UITableViewDelegate`.

- [ ] Leading = positive/toggle (read/unread, pin, favorite); trailing = archive/delete/more. Max ~3 per edge; the first action is the full-swipe one.
- [ ] `allowsFullSwipe` only for the safe, most-common action; disable full swipe for destructive actions without undo.
- [ ] **Destructive** (`role: .destructive` / `.destructive` style): either confirm (`.confirmationDialog` / alert) **or** perform with **Undo** (snackbar/toast, delayed commit). Not both, not neither. Destructive role animates the row out — if you then show a confirmation and the user cancels, re-insert the row correctly.
- [ ] UIKit: always call the `completion(Bool)` handler (true when performed) — forgetting it leaves the row stuck open.
- [ ] Tint with semantic colors (`.tint(.blue)`), SF Symbols with labels (`Label("Archive", systemImage: "archivebox")`) — label text is used for accessibility and when the icon-only style is shown.
- [ ] Haptics: `.sensoryFeedback(.success, trigger:)` (iOS 17) or `UINotificationFeedbackGenerator`/`UIImpactFeedbackGenerator` on commit; respect system behaviour (full swipe already has haptics — don't double up).
- [ ] **Accessibility alternative**: every swipe action is also reachable via `.contextMenu` and/or `.accessibilityActions { }` / `accessibilityCustomActions` (VoiceOver users can't discover swipes reliably). Swipe on `List` rows does expose actions to VoiceOver, but custom rows/`LazyVStack` don't.
- [ ] Toggle actions reflect current state ("Mark as Read" vs "Mark as Unread").
- [ ] Action mutates via the view model (optimistic + rollback, section 9), never directly mutates the row's local copy.
- [ ] Swipe disabled while in edit/multi-select mode and on skeleton/footer rows.

## 6. Pull to refresh

- [ ] SwiftUI `.refreshable { await vm.refresh() }` (iOS 15) — the closure must **await** the actual work; the spinner ends when it returns. Don't fire-and-forget a `Task` inside it.
- [ ] `.refreshable` works on `List` and on `ScrollView` (iOS 16+); verify on your min OS.
- [ ] UIKit: `UIRefreshControl` on `collectionView.refreshControl`; `endRefreshing()` on completion on main, including on error and cancellation.
- [ ] **Keep existing data** visible during refresh; replace via diff when done. On failure keep the old data + toast/banner "Couldn't refresh".
- [ ] Refresh **cancels in-flight load-more** (and resets cursor) — otherwise a late page from the old cursor appends to the fresh list. Conversely, ignore load-more triggers while refreshing.
- [ ] Optional "Updated 2 min ago" (`Text(date, style: .relative)` or `RelativeDateTimeFormatter`) in header/footer for data that goes stale.
- [ ] Refresh doesn't reset scroll position unless new content is at the top and the user was at the top.
- [ ] Also refresh on: app returning to foreground after N minutes (`scenePhase`), connectivity restored, after a mutation elsewhere (notification/observation).

## 7. Empty, error and offline states

- [ ] Distinct states: **empty** (no data exists — explain + primary action "Create your first…"), **no results** (search/filter — "No results for 'x'", clear filters), **error** (message + Retry), **offline** (no cache — "You're offline" + Retry/auto-retry).
- [ ] iOS 17+: `ContentUnavailableView` (and `ContentUnavailableView.search` / `.search(text:)`); UIKit `UIContentUnavailableConfiguration` via `contentUnavailableConfiguration` + `setNeedsUpdateContentUnavailableConfiguration()`. Below 17: custom view with same structure.
- [ ] Use as `.overlay { if vm.items.isEmpty, case ... }` over the list so pull-to-refresh still works on empty/error states.
- [ ] **Cached data + offline**: show cached items with a non-blocking banner ("Offline — showing saved items", last updated time). Don't replace cached content with an error screen.
- [ ] Human messages mapped from errors (`URLError.notConnectedToInternet`, timeout, 401 → re-auth flow, 5xx). Never show raw `localizedDescription` of decoding errors.
- [ ] **Auto-retry on reconnect**: `NWPathMonitor` (Network framework) publishing `path.status == .satisfied`; on transition offline → online, retry the failed initial load / footer once. Debounce flapping. Don't use reachability as a precondition to try a request (it can be wrong) — try, and use it only to trigger retries.
- [ ] Error state is announced to VoiceOver (`AccessibilityNotification.Announcement` / `UIAccessibility.post(notification: .announcement)`) when it replaces content.

## 8. Thumbnail images: caching, downsampling, cancellation

The #1 source of list jank and memory crashes.

- [ ] **Two-tier cache**: memory `NSCache<NSString, UIImage>` (decoded, display-size images; set `totalCostLimit` using pixel bytes `width*height*4`, `countLimit`) + **disk cache** for original/encoded data (`URLCache` with a large disk capacity, or a files-in-Caches directory with LRU/size cap). `NSCache` auto-evicts under memory pressure but also clear explicitly on `UIApplication.didReceiveMemoryWarningNotification`.
- [ ] **Downsample with ImageIO** to target size × `displayScale` (`CGImageSourceCreateThumbnailAtIndex` with `kCGImageSourceThumbnailMaxPixelSize`, `kCGImageSourceCreateThumbnailFromImageAlways`, `kCGImageSourceCreateThumbnailWithTransform`, `kCGImageSourceShouldCacheImmediately: true`; create source with `kCGImageSourceShouldCache: false`). A 4000×3000 JPEG decoded full-size is ~48 MB; a 120pt@3x thumbnail is ~0.5 MB.
- [ ] **Decode off the main thread** (downsample with `ShouldCacheImmediately` forces decode in the background, or `UIImage.preparingThumbnail(of:)` / `byPreparingForDisplay()` iOS 15). Main-thread decode = dropped frames.
- [ ] **Cache key includes size** (and scale, content mode, transforms): `"\(url)|\(Int(pixelW))x\(Int(pixelH))"`. Otherwise a tiny thumbnail gets reused in the detail view (blurry) or a huge image in the cell (memory).
- [ ] **Request dedupe**: concurrent requests for the same key share one task (dictionary of in-flight `Task`s in an actor).
- [ ] **Cancel on reuse/disappear**: UIKit `prepareForReuse` cancels the cell's task and clears the image; verify on completion that the cell still represents the same item (compare item ID/URL) before assigning. SwiftUI: load in `.task(id: url)` which cancels when the row disappears or the id changes. Cancellation must not cancel a shared deduped download other consumers still need (ref-count or let it finish into cache).
- [ ] **Prefetching**: `UICollectionViewDataSourcePrefetching.prefetchItemsAt` starts low-priority loads; `cancelPrefetchingForItemsAt` cancels. SwiftUI has no prefetch API — optionally prefetch next page's thumbnails in the view model after a page lands.
- [ ] **Placeholder & fade**: fixed-size placeholder (color/SF Symbol) with the final aspect ratio so rows don't resize; fade in only when loaded from network, not from memory cache (no flicker on fast scroll back). Failure state shows a fallback icon, no retry storm.
- [ ] **`AsyncImage` limitations**: no memory/disk cache control beyond `URLCache`, no downsampling, decodes full-size, re-fetches when rows are recreated, phases reset on scroll in lazy containers. Fine for a few images; not for image-heavy lists.
- [ ] Libraries (verify current versions/APIs): **Nuke** (`LazyImage`, `ImagePipeline`, `ImagePrefetcher`, processors incl. resize), **Kingfisher** (`KFImage`, `DownsamplingImageProcessor`, `.cacheOriginalImage()`), **SDWebImage** (`WebImage` via SDWebImageSwiftUI, `.thumbnailPixelSize` context). Prefer one of these unless constraints forbid dependencies; configure downsampling explicitly — none of them downsample by default.
- [ ] Respect `Cache-Control`; use signed-URL-stable keys (strip expiring query tokens from the cache key when the image itself doesn't change).
- [ ] Low Data Mode / constrained network (`URLSessionConfiguration.allowsConstrainedNetworkAccess`, `NWPath.isConstrained`): smaller variants.

## 9. List mutations

- [ ] **Optimistic update**: apply locally, call server, **rollback on failure** (restore previous item at its previous index) with a toast. Keep a per-item pending flag to disable conflicting actions.
- [ ] Delete with undo: remove from list immediately, schedule commit after ~4–5 s; Undo re-inserts at original position; commit on screen leave/background.
- [ ] Animate via **diffing**: SwiftUI `withAnimation { vm.items.remove(...) }` on identified data; UIKit `dataSource.apply(snapshot, animatingDifferences: true)`. Never `reloadData()` for single changes (kills animations, selection, scroll).
- [ ] Content changes of an existing item: UIKit `snapshot.reconfigureItems([id])` (iOS 15, cheaper than `reloadItems`, keeps cell); identifiers should be IDs, not whole models, so content changes don't look like delete+insert.
- [ ] Apply snapshots from **one queue consistently** (main is simplest); never apply concurrently from multiple queues.
- [ ] **Realtime inserts without jump**: if the user is at the top, insert with animation; if scrolled down, buffer new items and show a "N new items" pill (section 14); when inserting above the viewport, preserve the anchor (UIKit: capture first visible item + offset, apply snapshot without animation, restore offset; SwiftUI iOS 17+: `.scrollPosition(id:)` keeps the anchored ID; `.defaultScrollAnchor(.bottom)` for chats).
- [ ] Realtime updates to items not yet loaded are ignored or merged by ID; deletes of unloaded items are no-ops.
- [ ] Reorder: `.onMove` (List in edit mode) / diffable `reorderingHandlers`; persist order; rollback on failure.
- [ ] Mutations made on a detail screen reflect in the list (shared store / `@Observable` model / notifications) without a refetch.

## 10. Performance

- [ ] Large/unbounded data → **lazy containers**: `List`, or `LazyVStack`/`LazyVGrid` inside `ScrollView`. A plain `VStack` in `ScrollView` with `ForEach` of 1000 rows builds everything.
- [ ] `List` reuses cells (UICollectionView-backed since iOS 16); `LazyVStack` creates lazily but **does not recycle** views — memory grows with scrolled distance; for very long feeds prefer `List` or UIKit.
- [ ] **Stable identity**: `ForEach(items)` with stable `id`. Avoid `id: \.self`, avoid `.id(UUID())` modifiers (forces recreation each update), avoid conditional view-type switches at row root.
- [ ] Avoid **`AnyView`** in rows (defeats diffing); use `@ViewBuilder`, enums with concrete views, or `some View`.
- [ ] Cheap `body`: no formatting (`DateFormatter` creation), sorting, filtering, or image decoding inside `body`; precompute in the view model or cache formatters statically. Pass only what the row needs (value types / IDs) so unrelated changes don't invalidate every row. With `@Observable` (iOS 17), each row should read only its own item's properties.
- [ ] Don't filter/sort `items` inside the `ForEach` expression on every render; store derived arrays.
- [ ] Self-sizing cost: complex Auto Layout or nested stacks per row; provide `estimatedHeight` in compositional layout; avoid `GeometryReader` per row; avoid `.frame(maxWidth: .infinity)` chains that force multiple layout passes.
- [ ] UIKit: cell registration + `contentConfiguration` (or `UIHostingConfiguration` for SwiftUI rows, iOS 16) — no work in `cellForItemAt` beyond configuration; images async.
- [ ] Shadows: set `shadowPath`; avoid offscreen-rendering masks/corner radius + shadow combos on every cell.
- [ ] Profile on a **real older device** in Release: Instruments **SwiftUI** template (view body counts, iOS 26 SwiftUI instrument / earlier "View Body"), **Time Profiler**, **Hangs/Animation Hitches**, **Allocations** (image memory), Core Animation FPS; `Self._printChanges()` in debug to find invalidation causes.
- [ ] Target: no hitches during fast scroll with images; memory stable after scrolling 1000 rows and back.

## 11. Search, filter, sort

- [ ] `.searchable(text:placement:prompt:)` (iOS 15); `.searchSuggestions`, `.searchScopes` (iOS 16.4), `.searchable(isPresented:)` (iOS 17) — verify. UIKit: `UISearchController` on `navigationItem`.
- [ ] **Debounce** typing (~250–400 ms): `.task(id: query) { try? await Task.sleep(for: .milliseconds(300)); await vm.search(query) }` — `.task(id:)` cancels the previous sleep/request automatically.
- [ ] **Cancel stale** requests and guard with generation token; results for "ca" must never overwrite results for "cat".
- [ ] Query/filter/sort change **resets paging** (cursor, `hasMore`, items) and scrolls to top; keep the old results visible (dimmed) until new ones arrive, or show spinner — not shimmer over old content.
- [ ] Local filtering for small in-memory sets (no network); server search for paged sets — don't filter only the loaded pages and present it as complete.
- [ ] Empty-query state (recent searches/suggestions) vs no-results state.
- [ ] Persist filter/sort selection per screen if expected; show active filter chips with clear action.
- [ ] `.scrollDismissesKeyboard(.immediately/.interactively)` (iOS 16) for search results.

## 12. Selection & bulk actions

- [ ] SwiftUI: `List(selection: $selection)` with `Set<Item.ID>` + `EditButton()` / `@Environment(\.editMode)`; on iPhone, multi-select requires edit mode; iPad/Mac support it with keyboard/pointer. UIKit: `allowsMultipleSelectionDuringEditing`, `isEditing`, list cell accessories (`.multiselect()`).
- [ ] Two-finger pan multi-select (UIKit `shouldBeginMultipleSelectionInteractionAt`) where useful.
- [ ] Selection stored by **ID**, survives refresh/load-more; items removed by refresh are removed from selection.
- [ ] **Select all**: clarify semantic — loaded items only vs all matching on server (then send a query to the server, not N IDs). Show count ("12 selected").
- [ ] Bottom toolbar (`.toolbar { ToolbarItemGroup(placement: .bottomBar) }` / `setToolbarItems`) with bulk actions, disabled when selection empty; destructive bulk action confirms with count.
- [ ] Bulk ops: optimistic with per-item rollback on partial failure; report "3 of 12 failed".
- [ ] Exiting edit mode clears selection; swipe actions and row navigation disabled while editing.
- [ ] Single-selection navigation (`NavigationSplitView` with `List(selection:)`) on iPad keeps selection highlighted.

## 13. Sections, sticky headers, index

- [ ] SwiftUI `Section(header:)` in `List` — `.plain` style pins headers; `LazyVStack(pinnedViews: [.sectionHeaders])` for custom stacks. UIKit compositional: `boundarySupplementaryItems` with `pinToVisibleBounds = true`; list configuration `headerMode = .supplementary` or `.firstItemInSection`.
- [ ] Section identity stable (date bucket key, letter) — diffable sections must also be unique.
- [ ] Paging across sections: a page may continue an existing section — merge into it, don't create a duplicate section header.
- [ ] Section index (A–Z): SwiftUI `.sectionIndexLabel` (iOS 26 — verify) or custom overlay with `ScrollViewReader`; UIKit `indexTitles(for:)` / `indexPathForItem(withIndexTitle:at:)` on the diffable data source. Only for large alphabetically-sorted lists.
- [ ] Collapsible sections: UIKit `NSDiffableDataSourceSectionSnapshot` with expand/collapse; SwiftUI `DisclosureGroup` / `Section(isExpanded:)` (iOS 17).
- [ ] Headers have `.accessibilityAddTraits(.isHeader)` so VoiceOver heading rotor works.

## 14. Scroll behaviour

- [ ] **Restore position** after navigation pop (automatic if the list view isn't recreated — don't rebuild the list's identity on pop or reset the model in `onAppear`). Across launches (optional): persist the top visible ID and restore after first load.
- [ ] **Tap the active tab to scroll to top**: UIKit `UITabBarController` does it for the root scroll view automatically; SwiftUI needs a custom tab selection binding that detects re-selection and calls `proxy.scrollTo(firstID)` / `scrollPosition.scrollTo(edge: .top)` (iOS 18). Status-bar tap (`scrollsToTop`) works only if exactly one scroll view has it enabled.
- [ ] **"New items" pill**: when realtime/refresh adds items above while the user is scrolled down, show "↑ 5 new posts"; tapping scrolls to top and merges; auto-merge when the user reaches the top.
- [ ] SwiftUI: `ScrollViewReader` + `proxy.scrollTo(id, anchor:)` (iOS 14); `.scrollPosition(id: $id)` + `.scrollTargetLayout()` (iOS 17); `ScrollPosition` type and `onScrollGeometryChange` (iOS 18). UIKit: `scrollToItem(at:at:animated:)`, `contentOffset`.
- [ ] Animated programmatic scroll respects Reduce Motion (`animated: !reduceMotion`).
- [ ] Don't scroll programmatically while the user is dragging (iOS 18 `onScrollPhaseChange` / `isTracking`).
- [ ] Keyboard: chat/comment lists keep the last message visible when keyboard appears (`.defaultScrollAnchor(.bottom)`, safe-area keyboard handling).

## 15. Accessibility

- [ ] Each row is **one accessibility element** with a meaningful label combining title, subtitle, state (`.accessibilityElement(children: .combine)` or explicit `.accessibilityLabel`), hint for non-obvious behaviour, `.isButton`/`.isSelected` traits.
- [ ] Swipe/context actions exposed as **custom actions** (section 5) so the VoiceOver actions rotor lists them.
- [ ] Headers marked as headers (heading rotor); custom rotors (`.accessibilityRotor`) for e.g. "Unread messages" in long lists.
- [ ] **Dynamic Type** up to accessibility sizes: rows self-size, text wraps, horizontal layouts switch to vertical (`ViewThatFits` / `dynamicTypeSize.isAccessibilitySize`), thumbnails don't squash text; skeletons scale too.
- [ ] **Reduce Motion**: shimmer off, insertion animations simplified, programmatic scroll not animated. **Reduce Transparency / Increase Contrast**: banners and pills readable.
- [ ] Loading, errors, end-of-list and "N new items" announced; focus not stolen by realtime inserts.
- [ ] Load more reachable for VoiceOver (infinite scroll triggers on focus move too; also offer a button).
- [ ] Touch targets ≥ 44×44 pt for row buttons (favorite, more).
- [ ] Voice Control: visible action labels (`.accessibilityInputLabels`) for icon-only buttons.

## 16. Testing plan

1. **Unit test the view model / state machine** with a fake repository: page merges, dedupe, end detection, error per phase, cancellation, refresh vs load-more race, optimistic rollback.
2. **Snapshot/preview tests** of every state: skeleton, loaded, empty, no results, error, offline+cache, footer loading/error/end, Dynamic Type XXXL, dark mode, RTL.
3. **Network Link Conditioner** (3G, 100% loss, high latency) and an artificial delay switch in debug builds.
4. **UI tests** (XCUITest) for scroll-to-load-more, pull-to-refresh, swipe delete + undo, search.
5. **Instruments** on an old device in Release: hitches, memory with images, body counts.
6. **VoiceOver, Voice Control, Dynamic Type, Reduce Motion** passes.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| L1 | Cold open, no cache (primary list) | Shimmer skeleton matching rows; crossfade to content |
| L2 | Cold open, fast network (<200 ms) | No loader flash |
| L3 | Cold open with cache | Cached rows instantly; silent refresh; no shimmer |
| L4 | Secondary list first load | Plain spinner, no shimmer |
| L5 | Scroll to threshold | Exactly one load-more request; footer spinner |
| L6 | Fast fling to bottom | No duplicate requests; no duplicate rows |
| L7 | Load-more fails | Items kept; footer error + Retry works |
| L8 | Last page | Footer end state; no further requests |
| L9 | Short first page | Next page loads automatically once |
| L10 | Pull to refresh during load-more | Load-more cancelled; list replaced correctly; no stale append |
| L11 | Refresh fails | Old data kept; toast |
| L12 | Empty response | Empty state with action; pull to refresh still works |
| L13 | Offline, no cache | Offline state; auto-retry on reconnect |
| L14 | Offline with cache | Cached list + offline banner |
| L15 | Swipe delete → Undo | Row returns to same position |
| L16 | Swipe delete, server fails | Row restored + error toast |
| L17 | Full swipe destructive | Only if allowed; confirm/undo present |
| L18 | VoiceOver actions on row | All swipe actions available as custom actions |
| L19 | Search typing fast | One request after debounce; no stale results |
| L20 | Change filter while scrolled deep | Paging reset; scroll to top |
| L21 | Multi-select + select all + bulk delete | Correct count; partial failure reported |
| L22 | Realtime insert while scrolled down | No jump; "new items" pill |
| L23 | Navigate to detail and back | Scroll position and loaded pages preserved; no reload |
| L24 | Tap active tab | Scrolls to top |
| L25 | Scroll 1000+ rows with images | No hitches; memory stable; thumbnails not blurry |
| L26 | Fast scroll, cell reuse | No wrong image in a row (flash of previous item) |
| L27 | Memory warning | Image memory cache purged; no crash |
| L28 | Dynamic Type AX5 | Rows readable; skeleton scales |
| L29 | Reduce Motion | No shimmer animation |
| L30 | Rotation / iPad split view | Layout adapts; no duplicate loads |

## 17. Common bugs to look for in review

- `id: \.self` / `UUID()` IDs / array indices → rows animate wrongly, state sticks to the wrong row, swipe opens on the wrong item
- Duplicate IDs across pages → diffable crash or SwiftUI undefined behaviour
- Load-more triggered by every row `onAppear` without an in-flight guard → N parallel requests, duplicates
- Offset paging with realtime inserts → duplicates/skips
- Refresh doesn't cancel load-more → old page appended to refreshed list
- `.refreshable { Task { await load() } }` → spinner ends immediately
- Initial load in `onAppear` → reloads (and resets scroll) every time the tab/screen reappears
- Load-more error replaces whole screen with error view, losing content
- Shimmer shown over existing content on refresh/filter change; shimmer on secondary lists; skeleton geometry ≠ real rows (jump on load)
- Loader flashes for 50 ms on fast networks; shimmer ignores Reduce Motion
- `.swipeActions` on `LazyVStack` rows (no effect); no accessibility alternative; UIKit `completion` never called
- Destructive full swipe with no undo/confirm
- `AsyncImage` in long image lists; full-size decode on main thread; cache key without size; no cancellation on reuse → wrong images and memory spikes
- `reloadData()` / new snapshot from scratch with new IDs on every update → lost animations, selection and scroll
- Snapshot applied from background and main queues interchangeably
- Heavy `body` (formatters, sorting, `AnyView`, `GeometryReader` per row)
- Search without debounce/cancellation; filter doesn't reset cursor
- Selection stored as indices; survives refresh pointing to different items
- Realtime insert at top while user reads mid-list → content jumps
- Error messages showing raw decoding errors; no Retry
- Empty state shown during first load (flash of "No items") because `items.isEmpty` is checked before state

## 18. Output format

When asked to **review/audit**:
1. **Summary** — framework (SwiftUI/UIKit/mixed), list spec table (kind, paging, page size, realtime, offline, swipes, loader), min OS, image pipeline used
2. **Checklist results** — sections 2–15 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
3. **Critical issues first** (crashes from duplicate IDs, data loss/wrong-row mutations, duplicate requests, memory blow-ups, jumps, inaccessible actions) with fix code
4. **Testing gaps** — L1–L30 not covered

When asked to **implement**: fill the list spec, start from the Appendix, adapt `Item`/repository/paging contract, delete unused parts (swipe, selection, realtime), and call out server contract questions (cursor stability, `hasMore`) and OS-gated APIs used.

When asked to **debug**: identify the state machine phase where it fails, reproduce with Network Link Conditioner/artificial delay, check identity (IDs), in-flight guards, cancellation, and image reuse — in that order.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI)

Adjust for your deployment target; APIs marked iOS 18 need `if #available`.

### A.1 Model, page contract, repository protocol

```swift
import Foundation

struct Item: Identifiable, Hashable, Sendable, Decodable {
    let id: String            // stable server ID
    var title: String, subtitle: String
    var thumbnailURL: URL?
    var isRead: Bool, updatedAt: Date
}

struct Page<T: Sendable>: Sendable {
    let items: [T]
    let nextCursor: String?   // nil == end reached
}

protocol ItemRepository: Sendable {
    func fetch(cursor: String?, limit: Int, query: String) async throws -> Page<Item>
    func setRead(id: Item.ID, _ read: Bool) async throws
    func delete(id: Item.ID) async throws
}
```

### A.2 ListState state machine

```swift
enum LoadError: Error, Equatable, Sendable {
    case offline, server, unauthorized, unknown

    init(_ error: Error) {
        switch error {
        case let e as URLError where [.notConnectedToInternet, .networkConnectionLost, .dataNotAllowed].contains(e.code):
            self = .offline
        case let e as URLError where e.code == .userAuthenticationRequired: self = .unauthorized
        case is DecodingError: self = .server
        default: self = .unknown
        }
    }

    var message: String {
        switch self {
        case .offline: "You're offline."
        case .server: "Something went wrong on our side."
        case .unauthorized: "Please sign in again."
        case .unknown: "Couldn't load items."
        }
    }
}

enum ListPhase: Equatable, Sendable {
    case idle, loadingInitial, loadingMore, refreshing, empty
    case loaded                 // more pages may exist
    case error(LoadError)       // initial load failed, no content
    case loadMoreError(LoadError) // content kept, footer shows retry
    case endReached

    var footer: FooterState {
        switch self {
        case .loadingMore: .loading
        case .loadMoreError(let e): .error(e)
        case .endReached: .end
        default: .hidden
        }
    }
}

enum FooterState: Equatable { case hidden, loading, error(LoadError), end }
```

### A.3 Paginator view model

```swift
import Observation
import SwiftUI

@MainActor
@Observable
final class ItemListViewModel {
    private(set) var phase: ListPhase = .idle
    private(set) var items: [Item] = []
    private(set) var showSkeleton = false
    var query = ""
    var toast: String?
    private(set) var lastDeletedID: Item.ID?

    private var ids = Set<Item.ID>()
    private var cursor: String?
    private var generation = 0
    private var task: Task<Void, Never>?
    private let repo: ItemRepository
    private let pageSize = 30
    private let prefetchDistance = 5

    init(repo: ItemRepository) { self.repo = repo }

    // MARK: Initial load
    func loadInitialIfNeeded() async {
        guard phase == .idle else { return }        // don't reload on re-appear
        await loadFirstPage(showSkeleton: true)
    }

    func retryInitial() async { await loadFirstPage(showSkeleton: true) }

    func refresh() async {
        guard phase != .loadingInitial else { return }
        await loadFirstPage(showSkeleton: false)    // keep content, no shimmer
    }

    func search(_ text: String) async {
        query = text
        await loadFirstPage(showSkeleton: items.isEmpty)
    }

    private func loadFirstPage(showSkeleton skeleton: Bool) async {
        task?.cancel()                              // cancels in-flight load-more
        generation += 1
        let gen = generation
        let hadContent = !items.isEmpty
        phase = hadContent ? .refreshing : .loadingInitial

        // Flicker avoidance: show skeleton only after 200 ms, keep ≥ 400 ms.
        var skeletonShownAt: ContinuousClock.Instant?
        let skeletonTask: Task<Void, Never>? = skeleton && !hadContent ? Task {
            try? await Task.sleep(for: .milliseconds(200))
            guard !Task.isCancelled, gen == self.generation else { return }
            self.showSkeleton = true
            skeletonShownAt = .now
        } : nil

        let t = Task {
            do {
                let page = try await repo.fetch(cursor: nil, limit: pageSize, query: query)
                skeletonTask?.cancel()
                if let shown = skeletonShownAt {
                    let remaining = Duration.milliseconds(400) - (ContinuousClock.now - shown)
                    if remaining > .zero { try? await Task.sleep(for: remaining) }
                }
                guard !Task.isCancelled, gen == generation else { return }
                withAnimation(.default) {
                    items = []; ids = []
                    append(page.items)
                    cursor = page.nextCursor
                    showSkeleton = false
                    phase = items.isEmpty ? .empty : (cursor == nil ? .endReached : .loaded)
                }
            } catch {
                skeletonTask?.cancel()
                guard !Task.isCancelled, gen == generation, !(error is CancellationError) else { return }
                showSkeleton = false
                if hadContent {
                    phase = cursor == nil ? .endReached : .loaded
                    toast = "Couldn't refresh. \(LoadError(error).message)"
                } else {
                    phase = .error(LoadError(error))
                }
            }
        }
        task = t
        await t.value                               // lets .refreshable await completion
    }

    // MARK: Load more
    func onRowAppear(_ item: Item) {
        guard let index = items.lastIndex(where: { $0.id == item.id }),
              index >= items.count - prefetchDistance else { return }
        loadMore()
    }

    func loadMore() {
        guard phase == .loaded || { if case .loadMoreError = phase { return true }; return false }(),
              let cursor else { return }            // double-trigger + end guard
        let gen = generation
        phase = .loadingMore
        task = Task {
            do {
                let page = try await repo.fetch(cursor: cursor, limit: pageSize, query: query)
                guard !Task.isCancelled, gen == generation else { return }
                append(page.items)
                self.cursor = page.nextCursor
                phase = page.nextCursor == nil ? .endReached : .loaded
            } catch {
                guard !Task.isCancelled, gen == generation, !(error is CancellationError) else { return }
                phase = .loadMoreError(LoadError(error))
            }
        }
    }

    private func append(_ newItems: [Item]) {
        for item in newItems where ids.insert(item.id).inserted { items.append(item) }
    }

    // MARK: Connectivity
    func connectivityRestored() {
        switch phase {
        case .error(.offline): Task { await retryInitial() }
        case .loadMoreError(.offline): loadMore()
        default: break
        }
    }

    // MARK: Mutations (optimistic + rollback)
    func toggleRead(_ item: Item) {
        guard let i = items.firstIndex(where: { $0.id == item.id }) else { return }
        let old = items[i]
        items[i].isRead.toggle()
        let newValue = items[i].isRead
        Task {
            do { try await repo.setRead(id: item.id, newValue) }
            catch {
                if let j = items.firstIndex(where: { $0.id == item.id }) { items[j] = old }
                toast = "Couldn't update. Try again."
            }
        }
    }

    private var pendingDeletes: [Item.ID: (item: Item, index: Int, commit: Task<Void, Never>)] = [:]

    func delete(_ item: Item) {
        guard let i = items.firstIndex(where: { $0.id == item.id }) else { return }
        withAnimation { _ = items.remove(at: i) }
        ids.remove(item.id)
        let commit = Task {
            try? await Task.sleep(for: .seconds(4))
            guard !Task.isCancelled else { return }
            do { try await repo.delete(id: item.id); pendingDeletes[item.id] = nil }
            catch { undoDelete(item.id); toast = "Couldn't delete." }
        }
        pendingDeletes[item.id] = (item, i, commit)
        lastDeletedID = item.id
        toast = "Deleted"   // toast shows Undo for lastDeletedID
    }

    func clearUndo() { lastDeletedID = nil }

    func undoDelete(_ id: Item.ID) {
        if lastDeletedID == id { lastDeletedID = nil; toast = nil }
        guard let p = pendingDeletes.removeValue(forKey: id) else { return }
        p.commit.cancel()
        guard ids.insert(id).inserted else { return }
        withAnimation { items.insert(p.item, at: min(p.index, items.count)) }
    }
}
```

### A.4 Shimmer modifier (Reduce Motion aware) and skeleton rows

```swift
// Sketch: `Shimmer` ViewModifier = content.redacted(reason: .placeholder) + a moving LinearGradient
// overlay masked to the redacted content, animated with .linear(1.2s).repeatForever; skipped entirely
// when @Environment(\.accessibilityReduceMotion) is true. `View.shimmering(_ active:)` applies it conditionally.

extension Item {
    static func placeholder(_ n: Int) -> Item {
        Item(id: "placeholder-\(n)", title: "Placeholder title text",
             subtitle: "A longer placeholder subtitle line", thumbnailURL: nil,
             isRead: true, updatedAt: .now)
    }
}

struct SkeletonList: View {
    var count = 8
    var body: some View {
        List(0..<count, id: \.self) { n in
            ItemRow(item: .placeholder(n))   // same row view → same geometry
                .shimmering()
        }
        .listStyle(.plain)
        .scrollDisabled(true)
        .allowsHitTesting(false)
        .accessibilityElement(children: .ignore)
        .accessibilityLabel("Loading")
    }
}
```

### A.5 Thumbnail loader: NSCache + disk (URLCache) + ImageIO downsampling + dedupe

```swift
import ImageIO
import UIKit

actor ThumbnailPipeline {
    static let shared = ThumbnailPipeline()

    private let memory: NSCache<NSString, UIImage> = {
        let c = NSCache<NSString, UIImage>()
        c.totalCostLimit = 80 * 1024 * 1024     // ~80 MB of decoded pixels
        c.countLimit = 400
        return c
    }()
    private var inFlight: [String: Task<UIImage?, Never>] = [:]
    private let session: URLSession = {
        let config = URLSessionConfiguration.default
        config.urlCache = URLCache(memoryCapacity: 0, diskCapacity: 300 * 1024 * 1024)  // disk tier
        config.requestCachePolicy = .returnCacheDataElseLoad
        return URLSession(configuration: config)
    }()

    // init: also observe UIApplication.didReceiveMemoryWarningNotification → memory.removeAllObjects().

    nonisolated static func key(_ url: URL, _ pixelSize: CGSize) -> String {
        "\(url.absoluteString)|\(Int(pixelSize.width))x\(Int(pixelSize.height))"
    }

    /// Synchronous memory hit for first render (no placeholder flash on scroll back).
    nonisolated func cached(_ url: URL, pixelSize: CGSize) -> UIImage? {
        memory.object(forKey: Self.key(url, pixelSize) as NSString)   // NSCache is thread-safe
    }

    func image(for url: URL, pixelSize: CGSize) async -> UIImage? {
        let key = Self.key(url, pixelSize)
        if let hit = memory.object(forKey: key as NSString) { return hit }
        if let running = inFlight[key] { return await running.value }   // dedupe

        // Detached so one consumer's cancellation doesn't kill the shared download.
        let task = Task.detached(priority: .utility) { [session] () -> UIImage? in
            guard let (data, _) = try? await session.data(from: url) else { return nil }
            return Self.downsample(data, to: pixelSize)
        }
        inFlight[key] = task
        let image = await task.value
        inFlight[key] = nil
        if let image {
            let cost = Int(image.size.width * image.scale * image.size.height * image.scale * 4)
            memory.setObject(image, forKey: key as NSString, cost: cost)
        }
        return image
    }

    /// Decodes at display size off the main thread.
    nonisolated static func downsample(_ data: Data, to pixelSize: CGSize) -> UIImage? {
        let srcOpts = [kCGImageSourceShouldCache: false] as CFDictionary
        guard let source = CGImageSourceCreateWithData(data as CFData, srcOpts) else { return nil }
        let opts = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceShouldCacheImmediately: true,       // force decode here
            kCGImageSourceCreateThumbnailWithTransform: true, // honor EXIF orientation
            kCGImageSourceThumbnailMaxPixelSize: max(pixelSize.width, pixelSize.height)
        ] as CFDictionary
        guard let cg = CGImageSourceCreateThumbnailAtIndex(source, 0, opts) else { return nil }
        return UIImage(cgImage: cg)
    }
}

// Sketch: `Thumbnail(url:size:)` — fixed-size frame (no row resize), placeholder fill, pixelSize =
// size * displayScale; `.task(id: url)` (cancels on disappear/reuse): synchronous
// `ThumbnailPipeline.shared.cached` hit → assign with no fade; else `image(for:pixelSize:)`,
// check `Task.isCancelled`, fade in, show a "photo" symbol on failure; `.accessibilityHidden(true)`.
```

UIKit equivalent: in the cell, store `var imageTask: Task<Void, Never>?` and `representedID`; in `prepareForReuse()` cancel the task and set `imageView.image = nil`; on completion check `representedID == item.id` before assigning. Implement `UICollectionViewDataSourcePrefetching` to warm the pipeline for upcoming index paths.

### A.6 Row, footer, and the list screen

```swift
import Network

// Sketches:
// - ItemRow(item:): HStack { Thumbnail; VStack { title (.semibold when unread); subtitle .lineLimit(2) };
//   unread dot (accessibilityHidden) }, .accessibilityElement(children: .combine), value "Unread".
// - LoadMoreFooter(state: FooterState, retry:): .hidden → 1pt clear; .loading → ProgressView("Loading more");
//   .error → message + bordered Retry; .end → "You're all caught up"; .listRowSeparator(.hidden).
// - Connectivity: @MainActor @Observable wrapper over NWPathMonitor; isOnline = path.status == .satisfied,
//   hopped to the main actor; monitor.cancel() in deinit.

struct ItemListScreen: View {
    @State var vm: ItemListViewModel
    @State private var connectivity = Connectivity()
    @State private var searchText = ""
    @State private var pendingDelete: Item?
    @State private var deletedCount = 0

    var body: some View {
        content
            .navigationTitle("Inbox")
            .searchable(text: $searchText, prompt: "Search")
            .task { await vm.loadInitialIfNeeded() }
            .task(id: searchText) {                          // debounce + cancel stale
                guard vm.phase != .idle else { return }
                try? await Task.sleep(for: .milliseconds(300))
                guard !Task.isCancelled, searchText != vm.query else { return }
                await vm.search(searchText)
            }
            .onChange(of: connectivity.isOnline) { _, online in
                if online { vm.connectivityRestored() }
            }
            .sensoryFeedback(.success, trigger: deletedCount)
            // + .safeAreaInset(edge: .top): "Offline — showing saved items" banner when
            //   !connectivity.isOnline && !vm.items.isEmpty.
    }

    @ViewBuilder private var content: some View {
        if vm.showSkeleton {
            SkeletonList()                                   // primary list, first load only
        } else {
            List {
                ForEach(vm.items) { item in
                    NavigationLink(value: item) { ItemRow(item: item) }
                        .onAppear { vm.onRowAppear(item) }
                        .swipeActions(edge: .leading, allowsFullSwipe: true) {
                            Button { vm.toggleRead(item) } label: {
                                Label(item.isRead ? "Mark Unread" : "Mark Read",
                                      systemImage: item.isRead ? "envelope.badge" : "envelope.open")
                            }
                            .tint(.blue)
                        }
                        .swipeActions(edge: .trailing, allowsFullSwipe: false) {
                            Button(role: .destructive) {
                                vm.delete(item); deletedCount += 1  // delete with undo
                            } label: { Label("Delete", systemImage: "trash") }
                        }
                        // Also mirror both actions in .contextMenu { } (pointer / long-press) and
                        // .accessibilityActions { } (VoiceOver) with the same buttons.
                }
                if !vm.items.isEmpty {
                    LoadMoreFooter(state: vm.phase.footer) { vm.loadMore() }
                }
            }
            .listStyle(.plain)
            .refreshable { await vm.refresh() }              // awaits real work
            .overlay { stateOverlay }
            .overlay(alignment: .bottom) { toastView }
            .animation(.default, value: vm.items.map(\.id))
        }
    }

    @ViewBuilder private var stateOverlay: some View {
        switch vm.phase {
        case .loadingInitial where !vm.showSkeleton:
            EmptyView()                                      // within 200 ms grace: show nothing
        case .empty where !vm.query.isEmpty:
            ContentUnavailableView.search(text: vm.query)
        case .empty:
            ContentUnavailableView("No messages", systemImage: "tray",
                                   description: Text("New messages will appear here."))
        case .error(let e):
            ContentUnavailableView {
                Label(e == .offline ? "You're offline" : "Couldn't load",
                      systemImage: e == .offline ? "wifi.slash" : "exclamationmark.triangle")
            } description: { Text(e.message) } actions: {
                Button("Retry") { Task { await vm.retryInitial() } }.buttonStyle(.borderedProminent)
            }
        default: EmptyView()
        }
    }

    // toastView: capsule with vm.toast + "Undo" (when vm.lastDeletedID != nil → vm.undoDelete(id));
    // .task(id: toast) posts AccessibilityNotification.Announcement, sleeps 4 s, then clears toast + undo.
    @ViewBuilder private var toastView: some View {
        if let toast = vm.toast {
            Text(toast)   // see sketch above
        }
    }
}
```

Notes on the appendix:
- Commit pending deletes immediately on `scenePhase == .background` (the 4 s timer may not run when suspended).
- Secondary lists: replace `SkeletonList()` with `ProgressView()` and drop `showSkeleton` delay logic except the 200 ms grace.
- Realtime: add `func insertRealtime(_ item: Item, userAtTop: Bool)` that inserts with animation when at top, else buffers into `pendingNew` and shows a pill; track "at top" with `onScrollGeometryChange` (iOS 18) or `.scrollPosition(id:)` (iOS 17).
- UIKit port: same view model; `UICollectionViewDiffableDataSource<Section, Item.ID>`, list layout with `trailingSwipeActionsConfigurationProvider`, `UIRefreshControl`, footer as a supplementary or a `.footer` item, apply snapshots on main with `animatingDifferences: true`, `reconfigureItems` for read toggles.
