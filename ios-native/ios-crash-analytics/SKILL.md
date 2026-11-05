---
name: ios-crash-analytics
description: Use when implementing, reviewing, or debugging crash reporting, product analytics and performance monitoring in an iOS/iPadOS app in Swift/SwiftUI — Firebase Crashlytics and Sentry setup behind a vendor-neutral CrashReporter/Analytics facade, dSYM generation and CI upload, MetricKit crash/hang/exit diagnostics, watchdog/OOM terminations, breadcrumbs and custom keys without PII, release health and regression gates, a typed event catalogue (object_action naming), anonymous → identified → reset identity across multi-account and portal switches, device-level analytics consent asked once after the first login (UserDefaults, survives logout), ATT sequencing and privacy manifests, batching/offline delivery, funnels and app-start/screen/network performance traces. Siblings: android-crash-analytics, react-native-crash-analytics, flutter-crash-analytics; related: ios-multi-account, ios-deep-links, ios-push-notifications, ios-in-app-purchase, ios-list-ux, ios-network-offline-sync.
---

# iOS Crash Reporting & Analytics Skill

Guide for building and auditing **crash reporting, product analytics and performance monitoring** in consumer iOS apps: Swift 5.9+/Swift 6, SwiftUI (UIKit hosts too), Firebase Crashlytics / Firebase Analytics, Sentry Cocoa, MetricKit, AppTrackingTransparency. Every vendor sits **behind two app-owned protocols — `CrashReporter` and `Analytics`** — so you can swap vendors, run two in parallel during a migration, and use fakes in tests.

Siblings: **android-crash-analytics**, **react-native-crash-analytics**, **flutter-crash-analytics** — keep the **event catalogue, property names, consent contract and identity rules identical across platforms** so dashboards and funnels join. Related: **ios-multi-account** (user id / portal on switch), **ios-deep-links** (attribution), **ios-push-notifications** (open events), **ios-in-app-purchase** (purchase funnel), **ios-list-ux** (scroll hitches), **ios-network-offline-sync** (network traces, offline queue).

Vendor SDKs move fast (Firebase iOS SDK majors yearly, Sentry Cocoa 8.x → 9.x option renames, MetricKit additions each WWDC, App Store privacy-manifest enforcement, ATT behaviour fixes). **Verify exact option names, Info.plist keys, run-script paths and availability against current vendor and Apple docs before shipping**, and gate with `if #available` when the deployment target is lower.

Key gates (verify): MetricKit `MXMetricManager` iOS 13, `MXDiagnosticPayload` (crash/hang/CPU/disk diagnostics) iOS 14, immediate diagnostic delivery iOS 15 (previously daily), `MXAppExitMetric` iOS 14; `ATTrackingManager` iOS 14 (enforced 14.5); privacy manifests (`PrivacyInfo.xcprivacy`) required for SDKs on the Apple list since May 2024; bitcode removed in Xcode 14 (App Store Connect no longer recompiles — your local dSYMs are the only dSYMs); `ENABLE_USER_SCRIPT_SANDBOXING` default YES since Xcode 15 (breaks naive upload scripts); Swift 6 strict concurrency for SDK callbacks.

## 0. Mental model

```
 App process
 ├── ConsentStore (UserDefaults.standard, DEVICE level)   ← single source of truth
 │      { version, analytics, crash, decidedAt }   survives logout, NOT reinstall
 │
 ├── Telemetry (app-owned facade, @MainActor configure + thread-safe track)
 │     ├── Analytics protocol ──┬─ FirebaseAnalyticsSink   (collection off until opt-in)
 │     │   (typed AnalyticsEvent)└─ YourBackendSink         (batch + offline queue)
 │     ├── CrashReporter protocol ┬─ CrashlyticsReporter    (collection off until opt-in)
 │     │   (breadcrumbs, keys,    └─ SentryReporter         (not started until opt-in)
 │     │    non-fatals, user id)
 │     ├── MetricKitCollector  (MXMetricManager subscriber; crashes, hangs, exits, launch)
 │     └── IdentityController  (anonymousId → userId on login, portal group, reset on logout)
 │
 └── Build pipeline
       DWARF with dSYM ─▶ upload (Crashlytics run script / sentry-cli) ─▶ symbolicated stacks
       release = CFBundleShortVersionString, dist/build = CFBundleVersion, env = debug|beta|prod
```

Rules of thumb:

- **Nothing optional is collected before the user answers** — not sent, not queued. The gate is in the facade *and* in the vendor's build-time default (Info.plist flags off).
- **Consent is a device fact, identity is an account fact.** Logging out changes identity, never consent.
- **App code only sees the facade.** No `import FirebaseAnalytics` / `import Sentry` outside the `Telemetry` module.
- **Events are typed values**, not free-form strings, and carry ids captured at creation time (account/portal) so teardown events aren't attributed to the next account.
- **A crash you can't symbolicate is a crash you can't fix** — dSYM upload is part of the release checklist, verified per build.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Tools**: Crashlytics, Sentry, both (migration / different teams), Bugsnag/Datadog/Embrace? Analytics: Firebase Analytics, Amplitude, Mixpanel, Segment, own backend? Who owns dashboards?
- **Legal basis & regions**: GDPR/UK GDPR/ePrivacy (opt-in for non-essential analytics), CCPA/CPRA ("do not sell/share"), kids (COPPA / Kids Category — no third-party analytics/ads), China. Is crash reporting treated as **legitimate interest** (default on, opt-out) or **consent** (default off)? Get this from legal, write it down.
- **Tracking**: Does anything link data with **other companies' apps/sites** for ads or measurement (IDFA, ad network SDKs, Meta/Google ads conversions, data brokers)? If yes → ATT required. If no → no ATT prompt, `NSPrivacyTracking = false`.
- **Release channels**: Debug, internal (Ad Hoc / TestFlight internal), TestFlight external, App Store; phased release? Separate Firebase/Sentry projects per env?
- **Build**: SPM vs CocoaPods; Xcode Cloud / GitHub Actions / Bitrise / fastlane; is the archive built on CI (where dSYMs exist)?
- **Backend pipeline**: own event ingestion? Warehouse (BigQuery export, Snowflake)? Server-side deletion API for data-subject requests?
- **Accounts**: single account, or multi-account/multi-portal (cross-ref **ios-multi-account**)? Which ids are safe to send (opaque server id vs email)?
- **Extensions**: widgets, Notification Service Extension, share extension — do they report crashes (separate dSYMs, separate SDK init, consent read from App Group)?
- **Performance**: app-start SLOs, key screens, network SLOs; sample rates and budget (Sentry transactions quota).

Record answers in a short **telemetry spec**: vendors, legal basis per category, consent copy version, env → project mapping, sampling, PII policy.

## 2. Crash reporting setup

- [ ] Initialise **as early as possible** — in `App.init()` / `application(_:didFinishLaunchingWithOptions:)` before any other work — but **only after reading `ConsentStore`** (synchronous, `UserDefaults`, cheap). Crashes in the first 100 ms otherwise go unreported.
- [ ] `FirebaseApp.configure()` once, in the main app; extensions configure separately if they use Firebase.
- [ ] Tag every report with **release** (`CFBundleShortVersionString`), **build/dist** (`CFBundleVersion`), **environment** (`debug` / `internal` / `testflight` / `appstore` — detect TestFlight via your build config or `Bundle.main.appStoreReceiptURL?.lastPathComponent == "sandboxReceipt"`, verify the heuristic; prefer a build-config flag).
- [ ] Sentry: `options.releaseName = "\(bundleId)@\(version)+\(build)"`, `options.dist = build`, `options.environment = env`. Crashlytics derives version/build automatically; add `env` as a custom key.
- [ ] **Separate projects** for debug/internal vs production (separate `GoogleService-Info.plist` per configuration copied in a build phase; separate Sentry DSN per env). Debug builds: usually disable reporting entirely or send to the dev project — never pollute prod crash-free rates.
- [ ] Debugger attached → Crashlytics doesn't catch crashes (the debugger does). Test crashes by launching **without** Xcode attached.
- [ ] Don't run two signal/Mach exception handlers fighting each other unknowingly: Crashlytics + Sentry both install handlers; it works (chained) but verify both receive the same test crash, and plan a migration end date. Prefer one fatal handler + the other for non-fatals/perf if possible.
- [ ] Extensions: each extension is a separate binary with its own dSYM and its own init; keep its memory footprint low (NSE limit ~24 MB).
- [ ] Privacy manifest: vendor SDKs ship their own `PrivacyInfo.xcprivacy`; your app's manifest must declare collected data types (Crash Data, Performance Data, Product Interaction, Device ID…) consistent with the App Store privacy label.

## 3. Symbolication (dSYMs) and CI upload

- [ ] `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` for **Release** (and for any config you distribute). Check SPM/CocoaPods dynamic frameworks produce dSYMs too.
- [ ] Since bitcode removal (Xcode 14) the **archive's dSYMs are final** — "Download dSYMs" in App Store Connect only matters for old bitcode builds. Archive on CI and upload from there.
- [ ] **Crashlytics**: run script after "Embed Frameworks" (SPM path shown; CocoaPods: `"${PODS_ROOT}/FirebaseCrashlytics/run"`):
  ```sh
  "${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
  ```
  Input files (needed with user-script sandboxing, verify list in Firebase docs):
  `${DWARF_DSYM_FOLDER_PATH}/${DWARF_DSYM_FILE_NAME}`, `.../Contents/Resources/DWARF/${PRODUCT_NAME}`, `.../Contents/Info.plist`, `$(TARGET_BUILD_DIR)/$(UNLOCALIZED_RESOURCES_FOLDER_PATH)/GoogleService-Info.plist`, `$(TARGET_BUILD_DIR)/$(EXECUTABLE_PATH)`. Or set `ENABLE_USER_SCRIPT_SANDBOXING = NO` for that target.
  Manual/CI: `upload-symbols -gsp GoogleService-Info.plist -p ios path/to/App.xcarchive/dSYMs`.
- [ ] **Sentry**: `sentry-cli debug-files upload --include-sources -o <org> -p <project> path/to/App.xcarchive/dSYMs` (auth token from CI secret, never in the repo); or the Sentry fastlane plugin `sentry_debug_files_upload`. `--include-sources` enables source context.
- [ ] Upload in CI **after archive, before/alongside App Store upload**; fail or warn the pipeline if upload fails. Don't upload from every Debug build (slow; use only on archive: `[ "$CONFIGURATION" = "Release" ] || exit 0`).
- [ ] Verify: `xcrun dwarfdump --uuid App.app.dSYM` UUIDs match the "missing dSYM" UUIDs in the dashboard; both vendors list missing UUIDs per build.
- [ ] Keep archives (or dSYMs) as CI artifacts for ≥ the support window — re-upload when a vendor loses them or you add a second vendor.
- [ ] MetricKit payloads are **unsymbolicated** (addresses + binary UUIDs + offsets) — symbolicate server-side with `atos -arch arm64 -o App.dSYM/Contents/Resources/DWARF/App -l <loadAddress> <addresses>` or forward to a vendor that ingests MetricKit (Sentry does for hangs/crashes with `enableMetricKit`; verify).
- [ ] Swift symbol demangling: vendors demangle; for raw `atos` output pipe through `swift demangle`.

## 4. Capture: fatal, non-fatal, hangs, terminations

- [ ] **Fatal crashes**: handled by the SDKs (Mach exceptions, signals, uncaught `NSException`). Swift runtime traps (`fatalError`, force unwrap, out-of-bounds, `precondition`) surface as `EXC_BREAKPOINT`/`SIGTRAP` — symbolication shows the line.
- [ ] **Non-fatals**: route via `CrashReporter.record(error:context:)` for things you *handle* but must know about (decode failure of a server payload, unexpected state, failed migration, Keychain `-25308`). Rate-limit per error type (e.g. ≤ 10/session) — Crashlytics stores only the last 8 non-fatals per session (verify), Sentry bills per event.
- [ ] Prefer `Error` types with stable domain/code (`enum SyncError: Int, Error` / `CustomNSError`) — grouping uses domain+code (Crashlytics) or type+stack (Sentry).
- [ ] **Hangs / app freezes**: Sentry `enableAppHangTracking` with `appHangTimeoutInterval` (≈2 s; v2 hang tracking in Sentry 8.x/9 distinguishes fully/non-fully blocking, verify). MetricKit `MXHangDiagnostic` (on-device, ≥ 250 ms threshold, delivered next launch or immediately on iOS 15+). Xcode Organizer → Hangs for App Store builds. Crashlytics doesn't report hangs as such.
- [ ] **Watchdog terminations** (launch > ~20 s, main thread blocked while backgrounding, `0x8badf00d`) and **OOM / jetsam**: not catchable by in-process handlers. Use:
  - `MXAppExitMetric` (counts of `cumulativeMemoryResourceLimitExitCount`, `cumulativeAppWatchdogExitCount`, `cumulativeBackgroundTaskAssertionTimeoutExitCount`, …, foreground vs background) — daily aggregate.
  - Sentry `enableWatchdogTerminationTracking` (heuristic: previous session ended without crash/normal exit; name and behaviour changed across versions — verify).
  - Your own "unclean shutdown" marker as a fallback: set a flag on launch, clear on `willTerminate`/background, inspect next launch (heuristic only — the user force-quitting looks similar, OS upgrades too).
- [ ] **MetricKit** subscriber registered early (`MXMetricManager.shared.add(_:)`), retained for app lifetime, payloads processed off-main and forwarded as JSON (`jsonRepresentation()`) to your backend or vendor. Also gives `MXCPUExceptionDiagnostic`, `MXDiskWriteExceptionDiagnostic`, launch/hang histograms.
- [ ] **Platform error hooks**: `NSSetUncaughtExceptionHandler` is taken by the SDKs — don't override; if you must, chain the previous handler. Log `URLError` classes as breadcrumbs, not non-fatals (noise). SwiftUI runtime warnings (`Publishing changes from within view updates`) are debug-only — catch in QA, not prod.
- [ ] Assertions: `assert`/`assertionFailure` are stripped in Release; convert important ones to `CrashReporter.record` non-fatals in Release builds.
- [ ] **Test crash** button in an internal/debug menu (`fatalError("Test crash")`) plus a "test non-fatal" — verify both appear symbolicated per release.

## 5. Context: breadcrumbs, custom keys, user, no PII

- [ ] **Breadcrumbs**: screen views, navigation, app lifecycle (`scenePhase`), network request summaries (method, route template `/items/{id}`, status, duration — never full URL with query/token), push open, deep-link route, account/portal switch, purchase step. Crashlytics `log(_:)` (64 KB ring), Sentry `SentrySDK.addBreadcrumb` (default 100).
- [ ] **Custom keys** (Crashlytics `setCustomValue`, Sentry tags/contexts): `env`, `account_count`, `portal_id` (opaque), `role`, `feature_flags_hash`, `locale`, `low_power_mode`, `storage_free_bucket`, `launch_type` (cold/warm/push/link).
- [ ] **User id**: opaque server id (or salted hash) only. Never email, phone, name, push token, IDFA. Set on login, change on **account switch**, clear on logout (cross-ref **ios-multi-account**).
- [ ] **Active account/portal**: set `portal_id` key on portal switch; `account_id` changes with user id. Capture these into breadcrumbs at event time.
- [ ] **No PII scrubbing reliance**: don't put PII in error messages (`"User \(email) failed"`), `localizedDescription` of server errors may contain it — sanitise before `record`. Sentry: `options.sendDefaultPii = false`, `beforeSend` / `beforeBreadcrumb` to strip query strings and headers; server-side data scrubbing as defence in depth.
- [ ] Screenshots / view hierarchy attachments (Sentry `attachScreenshot`, `attachViewHierarchy`, Session Replay) **off by default** for consumer apps with personal data; if enabled, mask all text/images and document in privacy label.
- [ ] `Logger`/`os_log` with `privacy: .private` for ids; don't forward unified logs wholesale.

## 6. Grouping & triage

- [ ] Understand grouping: Crashlytics by stack frame "blame" + exception; Sentry by stack trace / exception type, overridable with **fingerprints** (`event.fingerprint = ["sync-decode", endpoint]`) — use for non-fatals whose stacks are identical but causes differ, or different stacks that are one bug.
- [ ] Mark your own modules **in-app** (Sentry `inAppInclude`) so blame lands in your frames, not `libswiftCore`.
- [ ] Metrics: **crash-free users** and **crash-free sessions** per release; hang rate; OOM rate (from MetricKit exits per 1k sessions). Track per OS version and device class.
- [ ] **Release health gates**: phased release (7 days) on App Store; pause the phase if crash-free sessions < threshold (e.g. 99.5 %) or a new top issue appears. Automate: Sentry release health / Crashlytics velocity alerts → Slack/PagerDuty; CI script queries crash-free rate before expanding.
- [ ] **Regression alerts**: alert when a resolved issue reappears in a newer version (Sentry "regression", Crashlytics "regressed issue"). Resolve issues *in a version*, not globally.
- [ ] Triage cadence: daily top-5 by users affected for the latest release; tag owner; link to ticket. Close noise (system frameworks crashes you can't fix) with a note, not silence.
- [ ] Watch out for **sample bias**: users who opt out of crash reporting aren't counted — crash-free rate is "among consenting users".

## 7. Event design

- [ ] **Naming**: `object_action`, snake_case, past tense for completed actions: `item_viewed`, `cart_item_added`, `checkout_started`, `purchase_completed`, `account_switched`, `push_opened`. ≤ 40 chars (Firebase limit), no dynamic values in names (`screen_home_viewed` ❌ → `screen_viewed{screen: "home"}`).
- [ ] **Typed catalogue**: one Swift `enum AnalyticsEvent` with associated values (appendix A.1). Adding an event = adding a case, reviewed like API. Generate docs/tracking plan from it (or vice-versa from a shared YAML across platforms).
- [ ] **Required props** on every event (added by the facade, not call sites): `event_version`, `app_version`, `build`, `platform = ios`, `os_version`, `session_id`, `anonymous_id`, `user_id?`, `portal_id?`, `consent_version`, `client_ts` (UTC ISO-8601), `event_id` (UUID for dedupe).
- [ ] Property values: enums as strings from a closed set; amounts as decimal + ISO currency; ids opaque; **no free text** from users (search queries: send length/has_results, not the query, unless consented and legal approved).
- [ ] **Versioning**: bump `event_version` when semantics change; never reuse a name with different meaning; deprecate with a date. Firebase: ≤ 25 params per event, param names ≤ 40 chars, values ≤ 100 chars (verify current limits) — the facade truncates/validates in debug with `assertionFailure`.
- [ ] **Screen tracking**: explicit `screen_viewed(screen:)` from a SwiftUI modifier `.trackScreen(.home)` on `onAppear` (dedupe on re-appear from tab re-select if desired). Disable automatic Firebase screen reporting (`FirebaseAutomaticScreenReportingEnabled = NO`) — SwiftUI's hosting-controller names are useless.
- [ ] Don't track in `body` (re-evaluated many times) — use `onAppear`/`task`/action handlers.

## 8. Identity

- [ ] **Anonymous id**: random UUID generated on first launch, stored in `UserDefaults` (device/install level, reset on reinstall). Used before login and as a stable install id.
- [ ] **Login**: `Analytics.identify(userId:)` with opaque id; send `user_logged_in`; your backend aliases `anonymous_id → user_id` (Firebase `setUserID`; Amplitude/Mixpanel merge rules differ — know them; Mixpanel `identify` merges, `alias` legacy).
- [ ] **Logout** (last account): `reset()` → clear user id, user properties, **new anonymous id**, flush queue first (events before logout keep old ids). Consent is **not** touched (section 9).
- [ ] **Account switch** (multi-account): flush, set new user id + user props, `account_switched{from_count, to_role}`; crash reporter user id switched too. **Portal switch**: same user, set `portal_id` user property / group (`group_identify` in Amplitude/Segment).
- [ ] Capture identity **at event creation** (snapshot into the event envelope), not at send time — otherwise queued events from account A go out as B.
- [ ] Extensions: read the active user id from the App Group if they send events (usually they shouldn't send analytics, only crashes).

## 9. CONSENT — required behaviour

This section is a **contract**. Implementations that deviate are bugs.

### 9.1 When to ask

- [ ] Ask for analytics (and, if your legal basis requires it, crash) consent **the first time any user successfully logs in on this device**: right after the login succeeds, **before navigating to home** (a full-screen sheet / step in the post-login flow, not a toast).
- [ ] Ask **once per device**. Do **not** ask again on: subsequent logins, logout → login, adding a second account, account switch, portal switch, app update (unless the policy version bumped).
- [ ] **Policy version bump** → re-prompt **once** on next launch/login (compare `record.version < currentPolicyVersion`).
- [ ] Decline is as easy as accept (same prominence; "Not now" counts as **no**). No dark patterns; don't block app use on refusal.

### 9.2 Where it's stored

- [ ] **Device level, `UserDefaults.standard`** (or the App Group suite if extensions need it) — **not** per account, not inside the account's SwiftData store / per-account directory, not server-only.
- [ ] Record: `{ version: Int, analytics: Bool, crash: Bool, decidedAt: Date }` encoded as JSON under one key (appendix A.2).
- [ ] **Reinstall clears it** — that's intended (new install = new decision). **Do not use Keychain** for this: Keychain items survive app deletion, so a reinstall would silently reuse an old decision. Use Keychain only if you deliberately want persistence across reinstall (and document why).
- [ ] **Backups**: `UserDefaults` is included in iCloud/encrypted backups, so restoring to a new phone carries the decision. Decide if that's acceptable (usually yes); if not, store in a file with `isExcludedFromBackup = true` in Application Support.
- [ ] Optionally mirror the decision to your backend (audit log), but the device record is authoritative for gating.

### 9.3 Before the first answer

- [ ] **Nothing optional is collected or queued** pre-login and while the prompt is visible. The facade drops (does not buffer) events when `analytics != true`. "Queue now, send after consent" is **not** compliant.
- [ ] Vendor collection **disabled at build/config level** so SDKs don't auto-collect before your code runs:
  - Info.plist `FIREBASE_ANALYTICS_COLLECTION_ENABLED = NO` (or `FIREBASE_ANALYTICS_COLLECTION_DEACTIVATED = YES` for permanent off), `FirebaseCrashlyticsCollectionEnabled = NO`, `FirebaseAutomaticScreenReportingEnabled = NO`, `GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE = NO`, `GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_STORAGE = NO`, `..._AD_USER_DATA = NO`, `..._AD_PERSONALIZATION_SIGNALS = NO` (Consent Mode v2; verify key names), `FirebasePerformanceCollectionEnabled = NO` / `firebase_performance_collection_enabled` (verify).
  - Sentry: **don't call `SentrySDK.start`** until crash consent (or legal basis) allows it.
- [ ] `FirebaseApp.configure()` may still run (needed for other Firebase products) — the flags keep collection off.

### 9.4 Crash reporting treatment

- [ ] Separate toggle: `crash` is independent of `analytics`.
- [ ] Default per legal basis: **consent** → off until opted in (crashes before the prompt are lost, accept it); **legitimate interest** → on from first launch with an opt-out in Settings and disclosure in the privacy policy. Document which.
- [ ] Apply to **both SDKs**: Crashlytics `setCrashlyticsCollectionEnabled(_:)` (persists in the SDK too; you're authoritative anyway — set it every launch), Sentry start/`close()`.
- [ ] Crashlytics with collection off still *records* reports locally; on opt-in decide: `sendUnsentReports()` (only if the user agreed to include past crashes) or `deleteUnsentReports()`; on opt-out call `deleteUnsentReports()`.

### 9.5 Changing it later

- [ ] **Settings → Privacy** screen with both toggles, applied **immediately**:
  - opt-in: enable SDK collection, `Analytics.setConsent([.analyticsStorage: .granted])`, start Sentry.
  - opt-out: disable SDK collection, **delete queued events** (own queue file; Crashlytics `deleteUnsentReports`; Sentry `close()`), optionally `Analytics.resetAnalyticsData()` and call your backend's deletion endpoint (data-subject request), show what happens in copy.
- [ ] Changing the toggle writes a new record (`decidedAt = now`, same `version`).

### 9.6 Logout must not wipe consent

- [ ] **Logout, "Sign out of all accounts", account deletion, and per-account data wipe must NOT delete the consent record.** It's not account data. A common implementation wipes `UserDefaults` with `removePersistentDomain(forName:)` or iterates all keys on logout — that silently resets consent, re-prompts the next user and, worse, may re-enable defaults. Allow-list the consent key (and anonymous-id policy) in your wipe routine, and test it (C3).
- [ ] Only an explicit "Reset privacy choices" action (or reinstall) clears it.

### 9.7 ATT (App Tracking Transparency) sequencing

- [ ] ATT is **separate** from analytics consent: it's only about **tracking across other companies' apps/websites** (IDFA, ad attribution with third parties). First-party analytics without cross-app linking does **not** need ATT. If you don't track → no `NSUserTrackingUsageDescription`, no prompt, `NSPrivacyTracking = false`, and use Firebase without IDFA (e.g. the `FirebaseAnalyticsWithoutAdIdSupport` product / don't link `AdSupport`; verify current packaging).
- [ ] If you do track: show your own consent screen **first** (post-login). Only if the user enabled analytics **and** you need tracking, then call `ATTrackingManager.requestTrackingAuthorization` — never prompt ATT for users who declined analytics (pointless and hostile).
- [ ] Call ATT only when the app is **active** (`scenePhase == .active`, after the sheet dismissed; a small delay after dismissal helps) — called during launch/inactive it returns `.notDetermined`/denied without showing (behaviour varied across iOS 17.x, verify).
- [ ] Respect status every launch: `.authorized` → IDFA allowed; anything else → never read `ASIdentifierManager.advertisingIdentifier` (it's zeros anyway) and declare `NSPrivacyTrackingDomains` so iOS blocks tracking domains when denied.
- [ ] ATT is OS-managed per app: the user can change it in Settings; re-check `trackingAuthorizationStatus` on foreground.
- [ ] Kids apps: no ATT prompt, no tracking at all.

### 9.8 Platform persistence note

iOS: `UserDefaults.standard` (or an App Group `UserDefaults(suiteName:)` if extensions gate on it). Android uses DataStore (device-level), RN MMKV/AsyncStorage global instance, Flutter `shared_preferences` — keep the record shape identical so support docs and backend audit are the same.

## 10. Delivery: batching, offline, retry, sampling

- [ ] Vendors batch internally (Firebase ~hourly or on background; Sentry immediate with offline caching). Your own backend sink: in-memory buffer → flush every N events (20) or T seconds (30) or on `scenePhase == .background`.
- [ ] **Offline queue**: persist batches to a file in Application Support (not Caches — may be purged; not UserDefaults — size). Cap size (e.g. 1 MB / 1,000 events), drop oldest. Only events that passed the consent gate are ever written.
- [ ] **Retry** with exponential backoff + jitter; idempotent on server via `event_id`; on 4xx (bad schema) drop + log non-fatal once; on 413 split batch.
- [ ] **Background flush**: `UIApplication.beginBackgroundTask` around the final flush on backgrounding (end it in the expiration handler); don't schedule `BGTaskScheduler` just for analytics unless needed. Use a `URLSession` with `waitsForConnectivity` (cross-ref **ios-network-offline-sync**).
- [ ] **Sampling**: never sample crashes; sample high-volume events (scroll, impressions) client-side with a deterministic per-install hash so funnels stay consistent; Sentry `tracesSampleRate` / `tracesSampler` (e.g. 0.1 prod, 1.0 internal).
- [ ] Opt-out → delete the queue file immediately.
- [ ] Clock: send `client_ts` and let server compute offset with `sent_at`.

## 11. Funnels & key metrics

- [ ] **Activation**: `app_first_opened` (post-consent only, so may be "first consented open"), `signup_completed`, `onboarding_completed`, first key action (e.g. `item_created`) within N days.
- [ ] **Retention**: D1/D7/D30 based on `session_started` / any engaged event; define "active" once across platforms.
- [ ] **Purchase funnel** (cross-ref **ios-in-app-purchase**): `paywall_viewed{placement}` → `purchase_started{product_id}` → `purchase_completed{product_id, price, currency, transaction_id_hash}` / `purchase_failed{reason}` / `purchase_cancelled`; revenue truth comes from the server (App Store Server Notifications), client events are for funnel shape.
- [ ] **Deep-link attribution** (cross-ref **ios-deep-links**): `deep_link_opened{source, campaign, route}` with UTM-like params parsed from the URL; deferred deep links via your attribution provider (which may require ATT for IDFA-based matching — SKAdNetwork/AdAttributionKit don't).
- [ ] **Push** (cross-ref **ios-push-notifications**): `push_received` (NSE, optional), `push_opened{campaign}`.
- [ ] Guard against double counting: purchase events on restore, `screen_viewed` on tab re-select, `app_opened` on every `scenePhase` flip.

## 12. Performance monitoring

- [ ] **App start**: MetricKit `MXAppLaunchMetric` (time to first draw, resume time; histograms), Xcode Organizer launch metrics, Sentry app start tracing (cold/warm), Firebase Performance `_app_start`. Pre-main: keep dynamic frameworks few; post-main: defer SDK init beyond crash reporter.
- [ ] **Time to interactive** for key screens: custom span from `onAppear` to "content loaded" (Sentry `SentrySDK.startTransaction` / `reportFullyDisplayed()`, Firebase `Performance.startTrace(name:)`), behind a `PerformanceTracer` protocol.
- [ ] **Hitches / frozen frames**: MetricKit `MXAnimationMetric` (scroll hitch rate), Sentry slow/frozen frames; profile lists (cross-ref **ios-list-ux**).
- [ ] **Network traces**: Firebase Performance and Sentry auto-instrument `URLSession` (swizzling) — check it doesn't break your custom `URLSessionDelegate`; record route templates not full URLs; disable for auth endpoints if headers/bodies could leak.
- [ ] `os_signpost` / `mxSignpost` for custom intervals visible in Instruments and MetricKit.
- [ ] Performance collection follows the **analytics** consent (or its own toggle) — not crash consent unless legal says so.

## 13. Facade / abstraction layer

- [ ] Protocols: `Analytics` (`track`, `identify`, `setUserProperty`, `reset`, `setCollectionEnabled`, `flush`), `CrashReporter` (`record(error:)`, `breadcrumb`, `setKey`, `setUser`, `setCollectionEnabled`), `PerformanceTracer` (`startTrace`).
- [ ] A `Telemetry` composite fans out to multiple sinks and owns the **consent gate**, required props and identity snapshot — sinks are dumb.
- [ ] Inject via SwiftUI `Environment` (`@Environment(\.telemetry)`) or your DI container; no vendor singletons in feature code. Lint rule: forbid `import Firebase*` / `import Sentry` outside `Telemetry` target.
- [ ] **Test fakes**: `RecordingAnalytics` that stores events for assertions; `NoopCrashReporter` for previews.
- [ ] SwiftUI previews and unit tests must not initialise vendor SDKs (check `ProcessInfo.processInfo.environment["XCODE_RUNNING_FOR_PREVIEWS"]`, test host flag).
- [ ] Swift 6: sinks `Sendable`; vendor calls are thread-safe but configure on main actor.

## 14. Testing plan

1. **Unit tests**: `ConsentStore` (default nil, save/load, version bump → `needsPrompt`), consent gate (no events reach sinks/queue when nil/false), event catalogue mapping (names ≤ 40 chars, params ≤ 25, snake_case — a test iterating all cases via a `CaseIterable` sample list), identity snapshot at creation, queue persistence/cap/delete-on-opt-out.
2. **Logout wipe test**: run your real "sign out of all accounts" routine, assert consent record still present.
3. **Debug views**: Firebase DebugView (`-FIRDebugEnabled` launch argument), Sentry `options.debug = true` in internal builds; Charles/Proxyman to confirm **zero** requests to `app-measurement.com` / `*.ingest.sentry.io` before consent.
4. **Crash tests**: test-crash button on a TestFlight build without debugger → symbolicated in both vendors; force a hang (main-thread `sleep(5)` in internal menu); allocate until jetsam to see an OOM exit in MetricKit next day.
5. **MetricKit**: Xcode → Debug → Simulate MetricKit Payloads (device) to exercise the subscriber.
6. **CI**: assert dSYM upload step ran and missing-UUID check is clean (Sentry `sentry-cli debug-files check`, verify).
7. **UI tests** (XCUITest) with launch args: `-resetConsent`, `-seedConsent v2:yes:no`, `-seedAccounts 2`.
8. **Fresh install** tests on device (reinstall to confirm re-prompt; Keychain leftovers not used).

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| C1 | Fresh install, before login | No analytics/crash (if consent basis) network calls; no queue file |
| C2 | First login on device | Consent sheet after login success, before home; home after answer |
| C3 | Logout → login (same account) | **No prompt**; previous choice applied |
| C4 | Sign out of all accounts → login with another account | **No prompt**; record intact |
| C5 | Add second account / switch account / switch portal | No prompt; user id switched; consent unchanged |
| C6 | Opt out in Settings | Collection disabled immediately; queue deleted; no further requests (proxy) |
| C7 | Opt in in Settings | Collection enabled; next event sent; Sentry started |
| C8 | Reinstall app | Prompt again after first login (UserDefaults gone; no Keychain reuse) |
| C9 | Policy version bump in update | Re-prompt once; then not again |
| C10 | Kill app while consent sheet open | Next launch: still undecided, nothing collected, prompt again after login/launch |
| C11 | Analytics yes, crash no | Events flow; Crashlytics disabled, Sentry not started, unsent reports deleted |
| C12 | Analytics no | ATT never requested |
| C13 | Analytics yes + tracking needed | ATT after consent sheet dismissed, app active; denied → no IDFA |
| C14 | Restore backup to new device | Decision carried (or not) per documented choice |
| C15 | Extension crash (NSE/widget) | Reported only if crash consent; symbolicated with extension dSYM |
| R1 | Fatal Swift trap in Release | Symbolicated with file:line in both vendors, correct release/build/env |
| R2 | Missing dSYM | CI warns; dashboard shows missing UUID; re-upload fixes |
| R3 | Non-fatal decode error x100 in a session | Rate-limited; grouped as one issue |
| R4 | Main-thread hang 3 s | Hang reported (Sentry / MetricKit) |
| R5 | OOM in background | `MXAppExitMetric` memory-limit count increments |
| R6 | Crash right after account switch | Report has new user id + portal key; breadcrumbs show switch |
| R7 | Debug build crash | Goes to dev project or nowhere, never prod |
| E1 | Offline 2 days, 500 events | Persisted (cap), sent on reconnect, server dedupes by event_id |
| E2 | Events queued for A, switch to B, flush | Events keep A's user id |
| E3 | Logout | Flush, reset, new anonymous id; consent intact |
| E4 | Purchase funnel | paywall_viewed → purchase_started → purchase_completed once (not on restore) |
| E5 | Deep link with campaign | deep_link_opened with source/campaign; attributed session |
| P1 | Cold start | App start metric reported; crash reporter init < 50 ms |
| P2 | Network trace | Route template, no query string or auth headers |

## 15. Common bugs to look for in review

- Consent record deleted on logout (`removePersistentDomain`, "clear all UserDefaults", per-account wipe that includes global keys) → prompt reappears / defaults re-enabled
- Consent stored per account (in account store or keyed by user id) → prompted again for every account added
- Consent stored in Keychain → survives reinstall, user never re-asked
- Consent asked at first launch before login (contract says after first login) or on every login
- Events buffered before consent and sent after opt-in
- Info.plist collection flags missing → Firebase Analytics collects `first_open`/`session_start` before your code runs
- Sentry started unconditionally then "disabled" by `beforeSend` returning nil — SDK still collects sessions/breadcrumbs, hard to audit
- Opt-out disables SDK but leaves the queue file / unsent Crashlytics reports
- ATT prompt at launch before any context, or for users who declined analytics; ATT called while inactive → never shown
- `NSUserTrackingUsageDescription` present without tracking (review rejection risk) or tracking without it
- Privacy manifest / App Store label inconsistent with what SDKs collect
- dSYMs not uploaded for App Store builds (bitcode-era assumption "Apple will provide them")
- Crashlytics run script silently failing under `ENABLE_USER_SCRIPT_SANDBOXING`
- Same Firebase/Sentry project for debug and prod → crash-free rate polluted
- Release/build not set in Sentry → can't gate releases or detect regressions
- Email/name as user id; full URLs with tokens in breadcrumbs; user text in event params
- Event names with dynamic parts (`screen_settings_viewed`), > 40 chars, inconsistent casing across platforms
- Tracking in SwiftUI `body` → event spam
- Identity read at send time → account A's queued events attributed to B
- Analytics reset not called on logout → next user merged into previous user's profile
- Two crash SDKs where one swallows the other's handler (never verified)
- Non-fatal spam (every `URLError.notConnectedToInternet`) blows quota and buries real issues
- MetricKit subscriber not retained / added late → payloads never delivered
- Vendor SDK initialised in previews/unit tests → flaky tests, fake data in dev project

## 16. Output format

When asked to **review/audit**:
1. **Summary** — telemetry spec (section 1): vendors, legal basis per category, env → project mapping, facade present?, min OS
2. **Consent contract** — section 9 items each ✅ Pass / ❌ Fail / ⚠️ Needs check with file:line (this goes first; it's the compliance risk)
3. **Checklist results** — sections 2–13 with the same marks
4. **Critical issues** — pre-consent collection, consent wiped on logout, PII, missing dSYMs, wrong identity — with fix code
5. **Testing gaps** — C/R/E/P scenarios not covered

When asked to **implement**: fill the spec, start from the Appendix, pick sinks (delete unused vendor), add Info.plist flags, the CI dSYM step, privacy manifest entries, and wire `ConsentFlow` into your post-login route. List what needs legal sign-off (copy, crash default, backup choice).

When asked to **debug**: (1) is the consent record what you think (print it)? (2) are the SDK collection flags actually on/off at runtime? (3) for unsymbolicated crashes compare dSYM UUIDs; (4) for missing events check the gate, the queue file, DebugView, then server ingestion — in that order.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI)

Simplified but complete enough to adapt. Vendor calls reflect Firebase iOS SDK 11.x and Sentry Cocoa 8.x — **verify names against your SDK versions**. `Session`/`AppModel` refer to the types from **ios-multi-account**.

### A.1 Typed event catalogue

```swift
import Foundation

enum Screen: String, Sendable { case home, itemDetail = "item_detail", paywall, settings, privacy }

enum AnalyticsEvent: Sendable {
    static let schemaVersion = 3

    case screenViewed(Screen)
    case userLoggedIn(method: String)               // "password" | "apple" | "google"
    case accountSwitched(accountCount: Int)
    case portalSwitched(role: String)
    case itemViewed(itemId: String)
    case cartItemAdded(itemId: String, quantity: Int)
    case paywallViewed(placement: String)
    case purchaseStarted(productId: String)
    case purchaseCompleted(productId: String, price: Decimal, currency: String)
    case purchaseFailed(productId: String, reason: String)
    case deepLinkOpened(route: String, source: String?, campaign: String?)
    case pushOpened(campaign: String?)

    var name: String {
        switch self {
        case .screenViewed: "screen_viewed"
        case .userLoggedIn: "user_logged_in"
        case .accountSwitched: "account_switched"
        case .portalSwitched: "portal_switched"
        case .itemViewed: "item_viewed"
        case .cartItemAdded: "cart_item_added"
        case .paywallViewed: "paywall_viewed"
        case .purchaseStarted: "purchase_started"
        case .purchaseCompleted: "purchase_completed"
        case .purchaseFailed: "purchase_failed"
        case .deepLinkOpened: "deep_link_opened"
        case .pushOpened: "push_opened"
        }
    }

    /// Only String / Int / Double / Bool values — maps 1:1 to every vendor.
    var properties: [String: AnalyticsValue] {
        switch self {
        case .screenViewed(let s): ["screen": .string(s.rawValue)]
        case .userLoggedIn(let m): ["method": .string(m)]
        case .accountSwitched(let c): ["account_count": .int(c)]
        case .portalSwitched(let r): ["role": .string(r)]
        case .itemViewed(let id): ["item_id": .string(id)]
        case .cartItemAdded(let id, let q): ["item_id": .string(id), "quantity": .int(q)]
        case .paywallViewed(let p): ["placement": .string(p)]
        case .purchaseStarted(let p): ["product_id": .string(p)]
        case .purchaseCompleted(let p, let price, let cur):
            ["product_id": .string(p), "value": .double(NSDecimalNumber(decimal: price).doubleValue),
             "currency": .string(cur)]
        case .purchaseFailed(let p, let r): ["product_id": .string(p), "reason": .string(r)]
        case .deepLinkOpened(let route, let src, let camp):
            ["route": .string(route)].merging(
                [("source", src), ("campaign", camp)].compactMap { k, v in v.map { (k, .string($0)) } },
                uniquingKeysWith: { a, _ in a })
        case .pushOpened(let c): c.map { ["campaign": .string($0)] } ?? [:]
        }
    }
}

enum AnalyticsValue: Sendable, Codable, Equatable {
    case string(String), int(Int), double(Double), bool(Bool)
    var any: Any { switch self { case .string(let v): v; case .int(let v): v; case .double(let v): v; case .bool(let v): v } }
}

/// Envelope: identity and required props captured at creation time.
struct EventEnvelope: Sendable, Codable {
    let eventId: UUID
    let name: String
    let properties: [String: AnalyticsValue]
    let anonymousId: String
    let userId: String?
    let portalId: String?
    let sessionId: String
    let consentVersion: Int
    let clientTs: Date
    var requiredProps: [String: AnalyticsValue] {
        let info = Bundle.main.infoDictionary ?? [:]
        var p: [String: AnalyticsValue] = [
            "event_version": .int(AnalyticsEvent.schemaVersion),
            "app_version": .string(info["CFBundleShortVersionString"] as? String ?? "?"),
            "build": .string(info["CFBundleVersion"] as? String ?? "?"),
            "platform": .string("ios"), "session_id": .string(sessionId),
            "consent_version": .int(consentVersion), "event_id": .string(eventId.uuidString),
        ]
        if let portalId { p["portal_id"] = .string(portalId) }
        return p
    }
}
```

### A.2 ConsentStore (device level, UserDefaults)

```swift
import Foundation
import Observation

struct ConsentRecord: Codable, Equatable, Sendable {
    var version: Int
    var analytics: Bool
    var crash: Bool
    var decidedAt: Date
}

/// DEVICE-level. Never cleared by logout / sign-out-all / account wipe.
/// UserDefaults (not Keychain) so a reinstall clears it and re-prompts.
@MainActor @Observable
final class ConsentStore {
    static let currentPolicyVersion = 2
    static let defaultsKey = "telemetry.consent.v1"     // allow-listed in AccountWiper
    /// Legal basis for crashes: false = consent required, true = legitimate interest (opt-out).
    static let crashDefaultWhenUndecided = false

    private let defaults: UserDefaults
    private(set) var record: ConsentRecord?

    init(defaults: UserDefaults = .standard) {
        self.defaults = defaults
        record = defaults.data(forKey: Self.defaultsKey)
            .flatMap { try? JSONDecoder().decode(ConsentRecord.self, from: $0) }
    }

    var needsPrompt: Bool { (record?.version ?? 0) < Self.currentPolicyVersion }
    var analyticsAllowed: Bool { !needsPrompt && record?.analytics == true }
    var crashAllowed: Bool {
        guard let record else { return Self.crashDefaultWhenUndecided }
        return record.crash
    }

    func decide(analytics: Bool, crash: Bool, now: Date = .now) {
        let r = ConsentRecord(version: Self.currentPolicyVersion, analytics: analytics, crash: crash, decidedAt: now)
        record = r
        defaults.set(try? JSONEncoder().encode(r), forKey: Self.defaultsKey)
    }

    /// Only for an explicit "Reset privacy choices" action — NOT called by logout.
    func resetForDebug() { record = nil; defaults.removeObject(forKey: Self.defaultsKey) }
}

/// Logout wipe: removes account data but preserves device-level keys.
enum DefaultsWiper {
    static let preservedKeys: Set<String> = [ConsentStore.defaultsKey, "telemetry.install_id"]
    static func wipeAccountKeys(_ defaults: UserDefaults = .standard) {
        for key in defaults.dictionaryRepresentation().keys
        where key.hasPrefix("account.") && !preservedKeys.contains(key) {   // prefix your own keys
            defaults.removeObject(forKey: key)
        }
        // ❌ never: defaults.removePersistentDomain(forName: Bundle.main.bundleIdentifier!)
    }
}
```

### A.3 Facade protocols and consent-gated Telemetry

```swift
import Foundation

protocol AnalyticsSink: Sendable {
    func setCollectionEnabled(_ on: Bool)
    func send(_ e: EventEnvelope)
    func identify(userId: String?)
    func setUserProperty(_ name: String, _ value: String?)
    func reset()
    func flush() async
    func deleteQueued()
}

protocol CrashReporter: Sendable {
    func setCollectionEnabled(_ on: Bool, includePastReports: Bool)
    func record(error: Error, context: [String: String])
    func breadcrumb(_ message: String, category: String)
    func setKey(_ key: String, _ value: String)
    func setUser(id: String?)
}

@MainActor
final class Telemetry {
    private let consent: ConsentStore
    private let sinks: [AnalyticsSink]
    private let crash: [CrashReporter]
    private(set) var anonymousId: String
    private var userId: String?
    private var portalId: String?
    private let sessionId = UUID().uuidString
    private var nonFatalCounts: [String: Int] = [:]

    init(consent: ConsentStore, sinks: [AnalyticsSink], crash: [CrashReporter],
         defaults: UserDefaults = .standard) {
        self.consent = consent; self.sinks = sinks; self.crash = crash
        if let id = defaults.string(forKey: "telemetry.install_id") { anonymousId = id }
        else { anonymousId = UUID().uuidString; defaults.set(anonymousId, forKey: "telemetry.install_id") }
        applyConsent(includePastCrashes: false)
    }

    /// Call at launch and whenever the record changes.
    func applyConsent(includePastCrashes: Bool) {
        sinks.forEach { $0.setCollectionEnabled(consent.analyticsAllowed) }
        crash.forEach { $0.setCollectionEnabled(consent.crashAllowed, includePastReports: includePastCrashes) }
        if !consent.analyticsAllowed { sinks.forEach { $0.deleteQueued() } }
    }

    func track(_ event: AnalyticsEvent) {
        breadcrumb(event.name, category: "event")        // breadcrumbs follow crash consent inside reporter
        guard consent.analyticsAllowed else { return }   // DROP, never buffer, before consent
        let env = EventEnvelope(eventId: UUID(), name: event.name, properties: event.properties,
                                anonymousId: anonymousId, userId: userId, portalId: portalId,
                                sessionId: sessionId, consentVersion: consent.record?.version ?? 0,
                                clientTs: .now)
        #if DEBUG
        assert(event.name.count <= 40 && event.properties.count <= 25, "Event violates vendor limits")
        #endif
        sinks.forEach { $0.send(env) }
    }

    func record(_ error: Error, context: [String: String] = [:]) {
        let key = "\(type(of: error))#\((error as NSError).code)"
        nonFatalCounts[key, default: 0] += 1
        guard nonFatalCounts[key]! <= 10 else { return }   // per-session rate limit
        crash.forEach { $0.record(error: error, context: context) }
    }

    func breadcrumb(_ message: String, category: String) {
        crash.forEach { $0.breadcrumb(message, category: category) }
    }

    // MARK: identity (A.7 uses these)
    func identify(userId: String, portalId: String?) async {
        await flushAll()
        self.userId = userId; self.portalId = portalId
        sinks.forEach { $0.identify(userId: userId); $0.setUserProperty("portal_id", portalId) }
        crash.forEach { $0.setUser(id: userId); $0.setKey("portal_id", portalId ?? "none") }
    }

    func setPortal(_ id: String?) {
        portalId = id
        sinks.forEach { $0.setUserProperty("portal_id", id) }
        crash.forEach { $0.setKey("portal_id", id ?? "none") }
    }

    func resetIdentity(defaults: UserDefaults = .standard) async {
        await flushAll()
        userId = nil; portalId = nil
        anonymousId = UUID().uuidString
        defaults.set(anonymousId, forKey: "telemetry.install_id")
        sinks.forEach { $0.reset() }
        crash.forEach { $0.setUser(id: nil) }
        // consent is deliberately NOT touched
    }

    func flushAll() async { for s in sinks { await s.flush() } }
}
```

### A.4 Vendor adapters (Firebase, Crashlytics, Sentry)

Info.plist (build-time defaults OFF):

```xml
<key>FIREBASE_ANALYTICS_COLLECTION_ENABLED</key><false/>
<key>FirebaseCrashlyticsCollectionEnabled</key><false/>
<key>FirebaseAutomaticScreenReportingEnabled</key><false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE</key><false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_STORAGE</key><false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_USER_DATA</key><false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_PERSONALIZATION_SIGNALS</key><false/>
```

```swift
import FirebaseAnalytics
import FirebaseCrashlytics
import Sentry

struct FirebaseAnalyticsSink: AnalyticsSink {
    func setCollectionEnabled(_ on: Bool) {
        Analytics.setAnalyticsCollectionEnabled(on)
        Analytics.setConsent([.analyticsStorage: on ? .granted : .denied,
                              .adStorage: .denied, .adUserData: .denied, .adPersonalization: .denied])
    }
    func send(_ e: EventEnvelope) {
        var params = e.properties.mapValues(\.any)
        for (k, v) in e.requiredProps where k != "event_id" && k != "session_id" { params[k] = v.any }
        Analytics.logEvent(e.name, parameters: params)
    }
    func identify(userId: String?) { Analytics.setUserID(userId) }
    func setUserProperty(_ n: String, _ v: String?) { Analytics.setUserProperty(v, forName: n) }
    func reset() { Analytics.setUserID(nil); Analytics.resetAnalyticsData() }
    func flush() async {}
    func deleteQueued() { Analytics.resetAnalyticsData() }
}

struct CrashlyticsReporter: CrashReporter {
    func setCollectionEnabled(_ on: Bool, includePastReports: Bool) {
        let c = Crashlytics.crashlytics()
        c.setCrashlyticsCollectionEnabled(on)
        if on, includePastReports { c.sendUnsentReports() } else if !on { c.deleteUnsentReports() }
    }
    func record(error: Error, context: [String: String]) {
        Crashlytics.crashlytics().record(error: error, userInfo: context)
    }
    func breadcrumb(_ m: String, category: String) { Crashlytics.crashlytics().log("[\(category)] \(m)") }
    func setKey(_ k: String, _ v: String) { Crashlytics.crashlytics().setCustomValue(v, forKey: k) }
    func setUser(id: String?) { Crashlytics.crashlytics().setUserID(id ?? "") }
}

final class SentryReporter: CrashReporter, @unchecked Sendable {
    private let dsn: String, environment: String
    private let lock = NSLock()
    private var started = false
    init(dsn: String, environment: String) { self.dsn = dsn; self.environment = environment }

    func setCollectionEnabled(_ on: Bool, includePastReports: Bool) {
        lock.lock(); defer { lock.unlock() }
        if on, !started {
            let info = Bundle.main.infoDictionary ?? [:]
            let version = info["CFBundleShortVersionString"] as? String ?? "0"
            let build = info["CFBundleVersion"] as? String ?? "0"
            SentrySDK.start { o in
                o.dsn = self.dsn
                o.environment = self.environment
                o.releaseName = "\(Bundle.main.bundleIdentifier ?? "app")@\(version)+\(build)"
                o.dist = build
                o.sendDefaultPii = false
                o.enableAppHangTracking = true
                o.appHangTimeoutInterval = 2
                o.enableWatchdogTerminationTracking = true
                o.tracesSampleRate = self.environment == "appstore" ? 0.1 : 1.0
                o.attachScreenshot = false
                o.attachViewHierarchy = false
                o.beforeBreadcrumb = { crumb in
                    if let url = crumb.data?["url"] as? String {          // strip query strings
                        crumb.data?["url"] = url.components(separatedBy: "?").first
                    }
                    return crumb
                }
            }
            started = true
        } else if !on, started {
            SentrySDK.close(); started = false
        }
    }
    func record(error: Error, context: [String: String]) {
        SentrySDK.capture(error: error) { scope in context.forEach { scope.setTag(value: $1, key: $0) } }
    }
    func breadcrumb(_ m: String, category: String) {
        let b = Breadcrumb(level: .info, category: category); b.message = m
        SentrySDK.addBreadcrumb(b)
    }
    func setKey(_ k: String, _ v: String) { SentrySDK.configureScope { $0.setTag(value: v, key: k) } }
    func setUser(id: String?) { SentrySDK.setUser(id.map { User(userId: $0) }) }
}
```

### A.5 MetricKit collector

```swift
import MetricKit
import os

final class MetricKitCollector: NSObject, MXMetricManagerSubscriber, @unchecked Sendable {
    private let upload: @Sendable (Data, String) -> Void   // (json, kind) → your backend
    private let log = Logger(subsystem: "app.telemetry", category: "metrickit")

    init(upload: @escaping @Sendable (Data, String) -> Void) {
        self.upload = upload
        super.init()
        MXMetricManager.shared.add(self)                   // keep `self` alive for app lifetime
    }
    deinit { MXMetricManager.shared.remove(self) }

    func didReceive(_ payloads: [MXMetricPayload]) {
        for p in payloads {
            if let exits = p.applicationExitMetrics {
                let bg = exits.backgroundExitData, fg = exits.foregroundExitData
                log.info("OOM bg=\(bg.cumulativeMemoryResourceLimitExitCount) fg=\(fg.cumulativeMemoryResourceLimitExitCount) watchdog=\(bg.cumulativeAppWatchdogExitCount)")
            }
            upload(p.jsonRepresentation(), "metrics")
        }
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for p in payloads {
            // Unsymbolicated: symbolicate server-side with the matching dSYM (binary UUID in payload).
            if let crashes = p.crashDiagnostics, !crashes.isEmpty { log.info("crashes=\(crashes.count)") }
            if let hangs = p.hangDiagnostics, !hangs.isEmpty { log.info("hangs=\(hangs.count)") }
            upload(p.jsonRepresentation(), "diagnostics")
        }
    }
}
```

Gate the upload on `consent.crashAllowed` at send time (diagnostics are crash data).

### A.6 Consent prompt hooked into first login (+ ATT sequencing)

```swift
import SwiftUI
import AppTrackingTransparency

enum PostLoginStep: Hashable { case consent, home }

@MainActor
func postLoginRoute(consent: ConsentStore) -> PostLoginStep {
    consent.needsPrompt ? .consent : .home         // device-level: false for every later login
}

struct ConsentView: View {
    let consent: ConsentStore
    let telemetry: Telemetry
    let needsTracking: Bool                          // true only if you share data cross-app
    let onDone: () -> Void
    @State private var analytics = false
    @State private var crash = ConsentStore.crashDefaultWhenUndecided
    @Environment(\.scenePhase) private var phase

    var body: some View {
        NavigationStack {
            Form {
                Section { Text("Help us improve the app. You can change this anytime in Settings → Privacy.") }
                Toggle("Usage analytics", isOn: $analytics)
                Toggle("Crash reports", isOn: $crash)
                Section {
                    Button("Save choices") { finish(analytics: analytics, crash: crash) }
                    Button("Allow all") { finish(analytics: true, crash: true) }
                    Button("Decline") { finish(analytics: false, crash: false) }   // equal prominence
                }
            }
            .navigationTitle("Your privacy")
            .interactiveDismissDisabled()               // swipe-away ≠ an answer
        }
    }

    private func finish(analytics: Bool, crash: Bool) {
        consent.decide(analytics: analytics, crash: crash)
        telemetry.applyConsent(includePastCrashes: false)
        onDone()
        guard analytics, needsTracking else { return }   // never ATT for decliners
        Task { @MainActor in
            try? await Task.sleep(for: .milliseconds(600))   // let the sheet dismiss; app must be active
            guard UIApplication.shared.applicationState == .active,
                  ATTrackingManager.trackingAuthorizationStatus == .notDetermined else { return }
            _ = await ATTrackingManager.requestTrackingAuthorization()
        }
    }
}

/// In your root view, after login succeeds:
struct SignedInRoot: View {
    let consent: ConsentStore
    let telemetry: Telemetry
    @State private var step: PostLoginStep?

    var body: some View {
        Group {
            switch step {
            case .consent:
                ConsentView(consent: consent, telemetry: telemetry, needsTracking: false) { step = .home }
            case .home, .none:
                HomeView()
            }
        }
        .onAppear { step = postLoginRoute(consent: consent) }
    }
}

struct PrivacySettingsView: View {
    @Bindable var consent: ConsentStore
    let telemetry: Telemetry
    let requestServerDeletion: () async -> Void

    var body: some View {
        Form {
            Toggle("Usage analytics", isOn: Binding(
                get: { consent.analyticsAllowed },
                set: { on in
                    consent.decide(analytics: on, crash: consent.crashAllowed)
                    telemetry.applyConsent(includePastCrashes: false)   // immediate; deletes queue on off
                    if !on { Task { await requestServerDeletion() } }
                }))
            Toggle("Crash reports", isOn: Binding(
                get: { consent.crashAllowed },
                set: { on in
                    consent.decide(analytics: consent.analyticsAllowed, crash: on)
                    telemetry.applyConsent(includePastCrashes: false)
                }))
        }
        .navigationTitle("Privacy")
    }
}

struct HomeView: View { var body: some View { Text("Home") } }
```

### A.7 Identity on login, account/portal switch, logout

```swift
@MainActor
extension Telemetry {
    /// Call from AppModel (ios-multi-account) after the new Session is built.
    func onAccountActivated(opaqueUserId: String, portalId: String?, accountCount: Int, isLogin: Bool) async {
        await identify(userId: opaqueUserId, portalId: portalId)
        breadcrumb("account activated", category: "auth")
        setCrashKeys(["account_count": "\(accountCount)"])
        track(isLogin ? .userLoggedIn(method: "password") : .accountSwitched(accountCount: accountCount))
    }

    func onPortalSwitched(portalId: String, role: String) {
        breadcrumb("portal switched", category: "auth")
        setPortal(portalId)
        track(.portalSwitched(role: role))
    }

    /// Sign out of the LAST account: identity reset, consent untouched.
    func onAllSignedOut() async {
        breadcrumb("signed out all", category: "auth")
        await resetIdentity()
    }

    private func setCrashKeys(_ keys: [String: String]) {
        // CrashReporter fan-out for arbitrary keys
        for (k, v) in keys { breadcrumb("\(k)=\(v)", category: "keys") }
    }
}

/// App entry: crash reporter first, consent read synchronously.
@main
struct ExampleApp: App {
    @State private var consent: ConsentStore
    private let telemetry: Telemetry
    private let metrics: MetricKitCollector

    init() {
        FirebaseBootstrap.configureIfNeeded()          // FirebaseApp.configure(); flags keep collection off
        let consent = ConsentStore()
        let env = BuildEnv.current                     // "debug" | "internal" | "testflight" | "appstore"
        let telemetry = Telemetry(
            consent: consent,
            sinks: env == "debug" ? [] : [FirebaseAnalyticsSink()],
            crash: env == "debug" ? [] : [CrashlyticsReporter(),
                                          SentryReporter(dsn: BuildEnv.sentryDSN, environment: env)])
        _consent = State(initialValue: consent)
        self.telemetry = telemetry
        metrics = MetricKitCollector { data, kind in
            Task { @MainActor in
                guard consent.crashAllowed else { return }
                await DiagnosticsUploader.shared.enqueue(data, kind: kind)
            }
        }
    }

    var body: some Scene {
        WindowGroup { SignedInRoot(consent: consent, telemetry: telemetry) }
    }
}

enum FirebaseBootstrap { static func configureIfNeeded() { /* FirebaseApp.configure() */ } }
enum BuildEnv { static let current = "appstore"; static let sentryDSN = "" }   // from xcconfig
actor DiagnosticsUploader { static let shared = DiagnosticsUploader(); func enqueue(_ d: Data, kind: String) {} }
```
