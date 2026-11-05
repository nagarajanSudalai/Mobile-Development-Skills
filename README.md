# App Development Skills

Claude skills for building, reviewing, and shipping apps. Each skill lives in its own folder with a `SKILL.md`.

Status: ✅ done · 🟡 next · ⬜ planned

## Monetization

| Skill | Status | Covers |
|---|---|---|
| [ios-in-app-purchase](ios-in-app-purchase/SKILL.md) | ✅ | Native StoreKit 2/1: product types, subscription groups, add-ons, offers, promoted IAPs, paid→freemium, Server API / Notifications V2, post-purchase UI, App Review 3.1, testing |
| [ios-revenuecat-iap](ios-revenuecat-iap/SKILL.md) | ✅ | RevenueCat iOS SDK: dashboard setup, entitlements, offerings, Paywalls & Customer Center, identity/transfers, webhooks, migration, testing |
| [android-in-app-purchase](android-in-app-purchase/SKILL.md) | ✅ | Native Google Play Billing: base plans & offers, replacement modes, acknowledge/consume (3-day rule), pending purchases, Play Developer API / RTDN, Play policy, testing |
| [android-revenuecat-iap](android-revenuecat-iap/SKILL.md) | ✅ | RevenueCat Android SDK: service credentials & RTDN, entitlements, offerings & subscription options, Paywalls & Customer Center, identity/transfers, plan changes, webhooks, migration |
| [cross-platform-in-app-purchase](cross-platform-in-app-purchase/SKILL.md) | ✅ | Flutter `in_app_purchase` & React Native `react-native-iap`/`expo-iap`: shared entitlement model across stores, complete-after-grant, Android plan changes/offers, unified server verification |
| [cross-platform-revenuecat-iap](cross-platform-revenuecat-iap/SKILL.md) | ✅ | RevenueCat in Flutter & React Native/Expo: one project for both stores, platform keys, cross-platform identity, Android plan-change branch, RevenueCatUI, migration from native plugins |
| paywall-design | ⬜ | Paywall UX, pricing pages, experiments, compliance-safe copy |

## Release & compliance

| Skill | Status | Covers |
|---|---|---|
| ios-app-store-submission | 🟡 | Metadata, screenshots, review notes, common rejections, phased release, TestFlight |
| ios-privacy-compliance | 🟡 | Privacy manifest (`PrivacyInfo.xcprivacy`), required-reason APIs, nutrition labels, ATT, account deletion |
| ios-ci-cd | ⬜ | Xcode Cloud / fastlane, signing & provisioning, build numbers, automated TestFlight uploads |

## Core app building blocks

| Skill | Status | Covers |
|---|---|---|
| [ios-push-notifications](ios-push-notifications/SKILL.md) | ✅ | APNs token auth & headers, permission flow, token lifecycle, routing (incl. cold start), actions, silent pushes, Service/Content extensions, communication notifications, Live Activity & VoIP pushes, FCM, provider server, testing |
| [android-push-notifications](android-push-notifications/SKILL.md) | ✅ | FCM HTTP v1, notification vs data messages, POST_NOTIFICATIONS, channels, token lifecycle, PendingIntents & deep links, direct reply, MessagingStyle/conversations, Doze & priority, exact alarms, server, adb testing |
| [cross-platform-push-notifications](cross-platform-push-notifications/SKILL.md) | ✅ | Flutter (firebase_messaging + flutter_local_notifications), React Native (RNFirebase + Notifee), Expo (expo-notifications): per-platform payloads, 3 app states × 2 platforms, cold-start routing, native NSE, server |
| ios-auth-sign-in-with-apple | ⬜ | Sign in with Apple, passkeys, Keychain, token refresh, account deletion requirement |
| ios-deep-links | ⬜ | Universal links, AASA file, custom schemes, routing in SwiftUI |
| ios-swiftui-architecture | ⬜ | `@Observable`, navigation, dependency injection, modularization, Swift 6 concurrency |
| ios-persistence-sync | ⬜ | SwiftData / Core Data, migrations, CloudKit sync, offline-first |
| ios-networking | ⬜ | URLSession async/await, retries, auth, caching, background transfers |

## Platform features

| Skill | Status | Covers |
|---|---|---|
| ios-widgets-live-activities | ⬜ | WidgetKit, interactive widgets, Live Activities, push-to-update |
| ios-app-intents | ⬜ | App Intents, Shortcuts, Siri, Spotlight, Control Center controls |

## Quality

| Skill | Status | Covers |
|---|---|---|
| ios-testing | ⬜ | Swift Testing / XCTest, UI tests, snapshot tests, StoreKit tests |
| ios-accessibility-localization | ⬜ | VoiceOver, Dynamic Type, RTL, String Catalogs, pluralization |
| ios-performance | ⬜ | Instruments, launch time, hangs, memory, battery, MetricKit |
| ios-crash-analytics | ⬜ | Crash reporting (Crashlytics/Sentry), symbolication, analytics events, privacy |

## Conventions

- Folder name = skill `name` in the frontmatter (kebab-case, platform prefix: `ios-`, `android-`, or none for cross-platform)
- `description` says **when** to use the skill and, if a sibling skill overlaps, when to use that one instead
- Structure: context questions → checklists by area → testing matrix → common bugs → output format (review vs implement) → appendix reference implementation
- Note OS/SDK version gates; tell Claude to verify fast-moving APIs against current docs
