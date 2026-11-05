# Mobile Development Skills

Claude skills for building, reviewing, and shipping mobile apps, organized by platform. Each skill lives in its own folder with a `SKILL.md`.

```
ios-native/        Swift · SwiftUI · UIKit
android-native/    Kotlin · Jetpack Compose · Views
react-native/      React Native (bare) · Expo
flutter/           Dart · Flutter
```

## Skills

| Area | iOS Native | Android Native | React Native | Flutter |
|---|---|---|---|---|
| In-app purchases (store-native) | [ios-in-app-purchase](ios-native/ios-in-app-purchase/SKILL.md) | [android-in-app-purchase](android-native/android-in-app-purchase/SKILL.md) | [react-native-in-app-purchase](react-native/react-native-in-app-purchase/SKILL.md) | [flutter-in-app-purchase](flutter/flutter-in-app-purchase/SKILL.md) |
| In-app purchases (RevenueCat) | [ios-revenuecat-iap](ios-native/ios-revenuecat-iap/SKILL.md) | [android-revenuecat-iap](android-native/android-revenuecat-iap/SKILL.md) | [react-native-revenuecat-iap](react-native/react-native-revenuecat-iap/SKILL.md) | [flutter-revenuecat-iap](flutter/flutter-revenuecat-iap/SKILL.md) |
| Push notifications | [ios-push-notifications](ios-native/ios-push-notifications/SKILL.md) | [android-push-notifications](android-native/android-push-notifications/SKILL.md) | [react-native-push-notifications](react-native/react-native-push-notifications/SKILL.md) | [flutter-push-notifications](flutter/flutter-push-notifications/SKILL.md) |
| List UX | [ios-list-ux](ios-native/ios-list-ux/SKILL.md) | [android-list-ux](android-native/android-list-ux/SKILL.md) | [react-native-list-ux](react-native/react-native-list-ux/SKILL.md) | [flutter-list-ux](flutter/flutter-list-ux/SKILL.md) |
| Deep links | [ios-deep-links](ios-native/ios-deep-links/SKILL.md) | [android-deep-links](android-native/android-deep-links/SKILL.md) | [react-native-deep-links](react-native/react-native-deep-links/SKILL.md) | [flutter-deep-links](flutter/flutter-deep-links/SKILL.md) |
| Multi-account & portals | [ios-multi-account](ios-native/ios-multi-account/SKILL.md) | [android-multi-account](android-native/android-multi-account/SKILL.md) | [react-native-multi-account](react-native/react-native-multi-account/SKILL.md) | [flutter-multi-account](flutter/flutter-multi-account/SKILL.md) |
| Crash reporting & analytics | [ios-crash-analytics](ios-native/ios-crash-analytics/SKILL.md) | [android-crash-analytics](android-native/android-crash-analytics/SKILL.md) | [react-native-crash-analytics](react-native/react-native-crash-analytics/SKILL.md) | [flutter-crash-analytics](flutter/flutter-crash-analytics/SKILL.md) |
| Networking & offline sync | [ios-network-offline-sync](ios-native/ios-network-offline-sync/SKILL.md) | [android-network-offline-sync](android-native/android-network-offline-sync/SKILL.md) | [react-native-network-offline-sync](react-native/react-native-network-offline-sync/SKILL.md) | [flutter-network-offline-sync](flutter/flutter-network-offline-sync/SKILL.md) |

### What each area covers

- **In-app purchases (store-native)** — StoreKit 2/1 and Google Play Billing (or `react-native-iap` / `expo-iap` / `in_app_purchase`): product types, subscriptions, offers, upgrades/downgrades, acknowledge/finish rules, server verification and notifications, store policy, testing.
- **In-app purchases (RevenueCat)** — dashboard setup, entitlements, offerings, Paywalls & Customer Center, identity/transfers, webhooks, migration, testing.
- **Push notifications** — APNs / FCM, permission flow, token lifecycle, routing (incl. cold start), actions, silent/data messages, extensions and channels, provider server, testing.
- **List UX** — batch data loading, load more / infinite scroll, shimmer loader for primary lists and a plain spinner for secondary lists and load-more, swipe actions, pull to refresh, empty/error/offline states, thumbnail caching, optimistic updates, performance, search/filter, selection, sticky headers, scroll restoration, accessibility.
- **Deep links** — universal links / App Links and their verification files, custom schemes, URL design, routing across cold/warm starts, pending links through login, vendor-neutral deferred deep links, entry points (push, widgets, QR, OAuth), fallbacks, security, testing.
- **Multi-account & portals** — several signed-in accounts per device, each with several portals (workspaces/organizations): secure per-account tokens, data isolation, account and portal switching, push and deep-link routing to the right account + portal, purchases, sign-out and cleanup.
- **Crash reporting & analytics** — crash capture and symbolication, breadcrumbs, typed event catalogue, identity across account switches, analytics consent asked once at first login and stored per device (survives logout/login), delivery, performance monitoring — Crashlytics and Sentry behind a vendor-neutral facade.
- **Networking & offline sync** — configured HTTP client, auth refresh, typed errors, retries with backoff, caching, connectivity, local DB as source of truth, outbox for offline writes, delta sync with tombstones, conflict resolution, background sync, realtime, large transfers, GraphQL notes.

## Conventions

- Folder name = skill `name` in the frontmatter, kebab-case with a platform prefix: `ios-`, `android-`, `react-native-`, `flutter-`
- `description` says **when** to use the skill and points to sibling skills on other platforms
- Structure: context questions → checklists by area → testing matrix → common bugs → output format (review vs implement) → appendix reference implementation
- Note OS/SDK/package version gates; tell Claude to verify fast-moving APIs against current docs
