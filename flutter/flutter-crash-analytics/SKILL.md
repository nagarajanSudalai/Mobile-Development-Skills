---
name: flutter-crash-analytics
description: Use when implementing, reviewing, testing, or debugging crash reporting, product analytics, consent and performance monitoring in a Flutter consumer app — firebase_crashlytics and firebase_analytics, sentry_flutter, FlutterError.onError + PlatformDispatcher.instance.onError (and isolate / zone errors), --obfuscate + --split-debug-info symbol upload (firebase crashlytics:symbols:upload, sentry_dart_plugin), iOS dSYMs and Android R8 mapping / NDK symbols, breadcrumbs and custom keys, release health gates, a typed event catalogue behind a vendor-neutral Analytics / CrashReporter facade, anonymous-id → login → account-switch identity, a device-level analytics consent prompt shown on first login and stored in shared_preferences, ATT sequencing, batching / offline delivery and firebase_performance / Sentry tracing. Siblings: ios-crash-analytics, android-crash-analytics, react-native-crash-analytics; related: flutter-multi-account, flutter-deep-links, flutter-push-notifications, flutter-in-app-purchase, flutter-list-ux, flutter-network-offline-sync.
---

# Flutter Crash & Analytics Skill

Guide for building, auditing, and testing **crash reporting, product analytics and the consent that governs both** in a Flutter consumer app. Covers Dart errors (framework, async, isolate), native crashes on iOS and Android (including NDK and ANRs / hangs), symbolication of obfuscated AOT builds, event design, identity across login / logout / account switch, and a **device-level consent record that is asked once, on first login, and survives sign-out**. Vendors are shown concretely — **Firebase Crashlytics + Firebase Analytics** and **Sentry** — but always behind a vendor-neutral `Analytics` / `CrashReporter` facade so you can swap, combine or fake them. Siblings: **ios-crash-analytics**, **android-crash-analytics**, **react-native-crash-analytics**. Related: **flutter-multi-account** (identity per account/portal), **flutter-deep-links** (attribution), **flutter-push-notifications** (open events), **flutter-in-app-purchase** (purchase funnel), **flutter-list-ux** (frozen frames in long lists), **flutter-network-offline-sync** (network traces, offline queues).

Crash and analytics SDKs move fast. Before shipping, **verify the current API** of: `firebase_core` / `firebase_crashlytics` / `firebase_analytics` / `firebase_performance` (FlutterFire majors bump together via the BoM-like `flutterfire` release train; `recordFlutterFatalError`, `setCrashlyticsCollectionEnabled`, `setConsent` parameters and the `firebase crashlytics:symbols:upload` CLI flags), `sentry_flutter` (v8 → v9 changed several option names; `SentryFlutter.init` `appRunner`, `beforeSend` signature, `enableAppHangTracking`, `anrEnabled`, frames tracking, `SentryNavigatorObserver`, Session Replay privacy masks), `sentry_dart_plugin` (pubspec `sentry:` keys, `upload_debug_symbols`, obfuscation-map support), `shared_preferences` (`SharedPreferencesAsync` / `SharedPreferencesWithCache` are the current APIs; the legacy `SharedPreferences.getInstance()` still works but is slated for deprecation), `app_tracking_transparency`, the Flutter `--obfuscate` / `--split-debug-info` / `--extra-gen-snapshot-options=--save-obfuscation-map` flags for your `flutter --version`. Check `pubspec.yaml` / `pubspec.lock`; don't trust memory.

Version gates worth knowing:
- `PlatformDispatcher.instance.onError` requires Flutter 3.3+; it replaces `runZonedGuarded` as the recommended catch-all for uncaught async errors. Keep a zone only if you need zone values or print capture.
- Crashlytics ANR reporting on Android uses `ApplicationExitInfo` (Android 11 / API 30+); MetricKit-based hang / OOM signals on iOS need iOS 14+ (diagnostics) — what the vendor surfaces varies.
- ATT (`requestTrackingAuthorization`) applies from iOS 14.5; without authorization IDFA is zeros.
- Google Consent Mode v2 (`setConsent(analyticsStorageConsentGranted:, adStorageConsentGranted:, adUserDataConsentGranted:, adPersonalizationSignalsConsentGranted:)`) matters for EEA users if you use Google ads / measurement.

## 0. Mental model

```
 main()
  └─ WidgetsFlutterBinding.ensureInitialized()
  └─ ConsentStore.load()  (shared_preferences, DEVICE level, outside any account)
  └─ Firebase.initializeApp()  /  SentryFlutter.init(appRunner: ...)
        (collection DISABLED by default in AndroidManifest / Info.plist / options)
  └─ CrashReporter.applyConsent(record)   Analytics.applyConsent(record)
  └─ FlutterError.onError  ─┐
     PlatformDispatcher.onError ─┼──> CrashReporter (facade) ──> Crashlytics adapter
     Isolate error listener ─┘                                └─> Sentry adapter
                                         ^ breadcrumbs, keys, user id
 UI / blocs / repositories ──> Analytics.track(TypedEvent) ──> ConsentGate ──> Batcher ──> Firebase / backend
                                         |                       (drop if no consent;
 SessionManager (flutter-multi-account) ─┘ identify / reset       never queue pre-consent)

 Build:  flutter build ipa|appbundle --obfuscate --split-debug-info=build/symbols
 CI:     upload build/symbols + dSYMs + R8 mapping + NDK .so  ──> Crashlytics / Sentry
```

Rules that follow from the model:

1. **Hooks are installed before `runApp`, SDKs start with collection off, consent flips them on.** Nothing optional leaves the device before a recorded "yes".
2. **App code never imports a vendor SDK.** It talks to `Analytics` and `CrashReporter`; adapters are the only files that import `firebase_*` / `sentry_*`.
3. **Consent is device-level state, not account state.** It lives next to device settings (theme, locale), not inside the per-account session that sign-out wipes.
4. **Every obfuscated release has its symbols uploaded in the same CI job that produced it**, keyed by the exact build id. A build without uploaded symbols is a build you cannot debug.
5. **Identity follows the active session**: anonymous id pre-login → opaque user id after login → switched on account/portal switch → reset on sign-out (consent untouched).

## 1. First, establish context

Find out (from the codebase or by asking):

- **Tools**: Crashlytics, Sentry, both? Analytics vendor (Firebase/GA4, Amplitude, Mixpanel, Segment, PostHog, own backend)? Performance (firebase_performance, Sentry tracing)? Session replay?
- **Legal basis and consent rules**: GDPR / UK GDPR / ePrivacy (opt-in for non-essential analytics in EEA/UK), CCPA/CPRA ("do not sell/share"), LGPD, children (COPPA / age-gated apps: often no analytics at all). Is crash reporting treated as **legitimate interest** (on by default, user can opt out) or **consent** (off until yes)? Ask legal; don't guess. Is consent geo-scoped or global?
- **ATT**: does the app (or any SDK) **track across apps/websites** (IDFA, ad attribution, data broker sharing)? If not, no ATT prompt — and make sure SDK flags don't collect IDFA.
- **Release channels**: debug, internal (Firebase App Distribution / TestFlight internal), beta, production; flavors (`--flavor`, `--dart-define-from-file`). Separate Firebase projects / Sentry projects or environments per channel?
- **Build pipeline**: Codemagic, GitHub Actions, Bitrise, Fastlane, Xcode Cloud? Where do symbols get uploaded? Is `--obfuscate` on?
- **Backend pipeline**: do events go to BigQuery export, a warehouse via Segment/RudderStack, or a first-party collector? Who owns the event schema? Server-side deletion API for data-subject requests?
- **Accounts**: single user, or multi-account / multi-portal (**flutter-multi-account**)? Which id is the analytics user id (opaque backend id, never email)?
- **Platforms**: iOS + Android only, or also web / desktop (different crash and symbol story — web needs source maps)?
- **Volume**: DAU and events per session (sampling, quota, cost).
- **Existing state**: what's already in `main.dart`, `AndroidManifest.xml`, `Info.plist`, `firebase.json`, `pubspec.yaml` `sentry:` block, CI scripts.

## 2. Crash reporting setup

- [ ] Initialize **as early as possible**: `WidgetsFlutterBinding.ensureInitialized()` → `Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform)` → install hooks → `runApp`. For Sentry, use `SentryFlutter.init(..., appRunner: () => runApp(...))` so its integrations wrap startup.
- [ ] Install **both** Dart hooks (§4): `FlutterError.onError` and `PlatformDispatcher.instance.onError`. Missing the second loses every uncaught async error.
- [ ] Tag every report with **release** (`<bundleId>@<version>+<build>`), **build number**, **environment** (`debug|internal|beta|production`), **flavor**, **dist** (Sentry). Read from `package_info_plus` + `--dart-define=APP_ENV=...`, not hard-coded.
- [ ] **Separate projects** (or at minimum environments) for debug/internal vs production. Debug crashes in prod dashboards destroy crash-free metrics and fire alerts. Commonly: one Firebase project per flavor (`flutterfire configure` per flavor), Sentry one project with `environment` tag, alerts filtered to `production`.
- [ ] **Disable in debug** unless testing the pipeline: `setCrashlyticsCollectionEnabled(!kDebugMode && consent.crash)`; Sentry `dsn: kDebugMode ? '' : dsn` or `beforeSend` drop.
- [ ] **Collection disabled at build/config level** until consent is applied (see §9): Android `firebase_crashlytics_collection_enabled=false`, iOS `FirebaseCrashlyticsCollectionEnabled=NO`; Sentry not initialised or `beforeSend` returns `null` until allowed.
- [ ] Don't run two vendors' native crash handlers blindly: Crashlytics and Sentry both install native signal handlers; chaining usually works but verify on device which one receives native crashes. Many teams use Crashlytics **or** Sentry for native, and both only for Dart errors.
- [ ] Avoid catching-and-swallowing at the top of `main` (a `try { runApp } catch {}` hides startup crashes from the native reporter).

## 3. Symbolication

Flutter release builds are AOT-compiled; with `--obfuscate` Dart symbols are renamed. Three symbol families matter:

| Layer | Artifact | Produced by | Upload |
|---|---|---|---|
| Dart AOT | `build/symbols/*.symbols` (per ABI / arch) | `--split-debug-info=build/symbols` | Crashlytics: `firebase crashlytics:symbols:upload --app=<FIREBASE_APP_ID> build/symbols`; Sentry: `sentry_dart_plugin` (`upload_debug_symbols: true`) |
| Dart obfuscation names (type names in messages) | `obfuscation.map.json` | `--extra-gen-snapshot-options=--save-obfuscation-map=build/app/obfuscation.map.json` | Sentry plugin (verify version support); Crashlytics: not used — keep for manual `flutter symbolize` |
| iOS native | dSYMs (`Runner.app.dSYM`, frameworks) | Xcode archive (`DEBUG_INFORMATION_FORMAT = dwarf-with-dsym`) | Crashlytics `upload-symbols` run script (added by `flutterfire configure`) or `firebase crashlytics:symbols:upload`; Sentry `sentry-cli debug-files upload` / plugin |
| Android Java/Kotlin | `mapping.txt` (R8) | `minifyEnabled true` | Crashlytics Gradle plugin (`mappingFileUploadEnabled`); Sentry Android Gradle plugin or `sentry-cli` |
| Android NDK (engine + plugins) | unstripped `.so` | Gradle `merged_native_libs` | Crashlytics `nativeSymbolUploadEnabled true` + `uploadCrashlyticsSymbolFile<Variant>`; Sentry Gradle plugin `uploadNativeSymbols` |

- [ ] Use `--split-debug-info` on **every** release build; add `--obfuscate` if you want name obfuscation (it is optional; symbolication works without it, obfuscation without split-debug-info is not allowed).
- [ ] **Never commit or ship** `build/symbols` inside the app; archive it as a CI artifact keyed by version+build (needed for `flutter symbolize` later).
- [ ] Upload **in the same CI job** right after the build — the symbols must match that build id byte-for-byte. Rebuilding "the same commit" produces different build ids.
- [ ] Fail the pipeline (or at least alert) if upload fails.
- [ ] iOS with Bitcode is gone (Xcode 14+); dSYMs come straight from the archive. For App Store builds, download dSYMs from App Store Connect only if you let Apple rebuild (you no longer do).
- [ ] Verify by forcing a test crash in an internal build and checking the stack shows file/line/function names.
- [ ] `--obfuscate` breaks code that relies on `runtimeType.toString()` (analytics event names, logging, `switch` on type names) — use explicit string names (§7).
- [ ] Web: Dart → JS source maps (`flutter build web --source-maps`) uploaded to Sentry; Crashlytics does not support web.

## 4. Capture

**Dart errors:**

- [ ] `FlutterError.onError` — framework errors (build, layout, paint, gesture callbacks). Crashlytics: `FirebaseCrashlytics.instance.recordFlutterFatalError` (or `recordFlutterError` for non-fatal). Keep `FlutterError.presentError(details)` in debug so the red screen and console still show.
- [ ] `PlatformDispatcher.instance.onError = (error, stack) { ...; return true; }` — uncaught async errors from the root isolate. Return `true` to mark handled.
- [ ] Background isolates (`Isolate.spawn`, `compute`): add `Isolate.current.addErrorListener(RawReceivePort(...).sendPort)` or wrap the entry function in try/catch and post errors back. `compute` rethrows into the caller's future — make sure that future is awaited.
- [ ] Background entry points (FCM `onBackgroundMessage`, `workmanager` callback) run in **separate isolates**: initialise Firebase / Sentry and install hooks there too.
- [ ] `ErrorWidget.builder` in release: show a friendly fallback, not a grey box; the error is already reported by `FlutterError.onError`.
- [ ] **Non-fatals**: `CrashReporter.recordError(e, st, reason:, fatal: false)` for caught-but-unexpected errors (decoding failures, impossible states). Don't report expected errors (offline, 401, user cancel) — they bury real issues and cost quota. Rate-limit noisy ones.
- [ ] Preserve stack traces: `catch (e, st)` and pass `st`; `Error.throwWithStackTrace` when rethrowing wrapped errors. `Future.error(e)` without a stack yields useless reports.
- [ ] Use `Chain`/`stack_trace` only if needed; vendors already parse Dart stack traces.

**Native:**

- [ ] Native crashes (Swift/ObjC/Kotlin/C++ in plugins and engine) are caught by the native SDK automatically once it's linked and enabled.
- [ ] **ANRs (Android)**: Crashlytics reports via `ApplicationExitInfo` on API 30+; Sentry `anrEnabled` (watchdog) plus `ApplicationExitInfo`. In Flutter, a busy UI isolate does not always block the Android main thread — long Dart work shows as jank / frozen frames (§12) rather than ANR. Platform-channel calls that block the main thread do cause ANRs.
- [ ] **App hangs (iOS)**: Sentry `enableAppHangTracking` (threshold ~2s); MetricKit hang diagnostics via native code if you want them. Watchdog terminations (`0x8badf00d`) show up as crashes on next launch.
- [ ] **OOM / jetsam / low memory kills**: not crashes from the app's view. Android `ApplicationExitInfo.REASON_LOW_MEMORY` (Sentry/Crashlytics surface some), iOS MetricKit / Sentry "watchdog termination" heuristics. Watch large images (`cacheWidth`), unbounded lists (**flutter-list-ux**), decoded image cache size.
- [ ] Startup crashes: crashes before the SDK sends get uploaded on next launch — but only if collection is enabled then. If consent is `crash: false`, they're dropped (correct).

## 5. Context: breadcrumbs, keys, user

- [ ] **Breadcrumbs** (`CrashReporter.breadcrumb(...)`): navigation (route observer), lifecycle (`AppLifecycleListener`), network (method + templated path + status, **no query strings or tokens**), key user actions, push opened, deep link received, account/portal switch. Crashlytics `log()`, Sentry `addBreadcrumb()`. Cap message length.
- [ ] **Custom keys**: `env`, `flavor`, `locale`, `theme`, `feature_flags` (hashed or short list), `account_count`, `portal_role`, `network_type`, `is_low_power`. Crashlytics `setCustomKey`, Sentry `setTag` / `setContexts`.
- [ ] **User id**: opaque backend user id only. On multi-account apps also set `active_account_id` (opaque) and `active_portal_id` keys and update them on switch (**flutter-multi-account**). No email, name, phone, portal names, addresses, free text.
- [ ] Sentry `sendDefaultPii: false`; scrub in `beforeSend` / `beforeBreadcrumb` (auth headers, emails by regex). Session Replay: mask all text and images by default.
- [ ] Don't put PII in exception **messages** (`throw StateError('user $email not found')`) — messages are grouped and indexed.
- [ ] Breadcrumbs are a ring buffer — log state transitions, not every frame or scroll event.

## 6. Grouping & triage

- [ ] **Fingerprints**: group wrapper errors by root cause. Sentry `event.fingerprint = ['api-decode', endpoint]`; Crashlytics groups by stack — keep the top frames meaningful (don't report everything from one `logError()` helper frame; pass the original stack).
- [ ] Strip variable data from messages (`'Order 123 failed'` → `'Order failed'` + key `order_id`) or every occurrence becomes its own issue.
- [ ] **Regression alerts**: alert on new issues in latest release, and on reopened (regressed) issues; route to the owning team (Sentry ownership rules / code owners, Crashlytics velocity alerts → Slack/Jira).
- [ ] **Crash-free users and sessions** per release are the headline metrics. Sentry release health needs `autoSessionTracking` on; Crashlytics shows crash-free users in the console and BigQuery export.
- [ ] **Release health gates**: staged rollout (Play % rollout, App Store phased release) — halt if crash-free sessions < target (e.g. 99.5%) or a new fatal issue affects > N users in the first 24h. Automate via Sentry release health API / Crashlytics BigQuery queries in CI.
- [ ] Triage weekly: top issues by users affected, new in release, ANR/hang rate (Play Console "bad behavior" thresholds: user-perceived ANR rate ~0.47%, crash rate ~1.09% — verify current values), OOM rate.
- [ ] Mark known third-party / OS issues as ignored with a reason, not deleted.

## 7. Event design

- [ ] **Naming: `object_action`** in snake_case, past tense for facts: `screen_viewed`, `signup_completed`, `checkout_started`, `item_added_to_cart`, `subscription_purchased`, `push_opened`, `deep_link_opened`, `account_switched`. Firebase limits: event name ≤ 40 chars, ≤ 25 params, param name ≤ 40, string value ≤ 100 chars; reserved prefixes `firebase_`, `google_`, `ga_` — design within the strictest vendor you use.
- [ ] **Typed event catalogue**: a sealed class (Dart 3) per event with typed required props; no `track('some string', {...})` from feature code. Compile-time safety + one place to review for PII.
- [ ] **Required common props** added by the facade: `schema_version`, `app_version`, `build`, `platform`, `env`, `session_id`, `portal_id` (if any), `consent_version`. Don't duplicate what the vendor adds automatically unless your warehouse needs it.
- [ ] **Versioning**: bump an event's `v` when meaning changes; never silently change semantics of an existing name. Keep a changelog / tracking plan (YAML or code-generated docs) reviewed by analytics owners.
- [ ] **No PII** in params: no emails, names, free text, search queries (or hash/truncate them only with legal sign-off), precise location.
- [ ] Enum values as stable strings, not Dart `enum.name` if names may be refactored (and never `runtimeType` — `--obfuscate` renames it).
- [ ] **Screen tracking**: a `NavigatorObserver` (or go_router `observers:`) emitting `screen_viewed { screen_name }` from route **names**, not paths with ids (`/orders/123` → `order_detail`). For `StatefulShellRoute` tabs, emit on branch change too. Firebase: `logScreenView(screenName:, screenClass:)` and consider disabling automatic screen reporting (`FirebaseAutomaticScreenReportingEnabled` / `google_analytics_automatic_screen_reporting_enabled`) since Flutter is one native screen.
- [ ] Avoid double counting: `screen_viewed` fires once per visible route, not on every rebuild.

## 8. Identity

- [ ] **Anonymous id pre-login**: a device-scoped random UUID (shared_preferences) or the vendor's app instance id. Only used once consent allows collection.
- [ ] **Login**: `Analytics.identify(opaqueUserId)` and `CrashReporter.setUser(opaqueUserId)`. If the vendor supports aliasing (Mixpanel/Amplitude/PostHog `alias` / `identify` merge), link the anonymous id to the user id once; GA4 uses `user_id` + `app_instance_id` automatically.
- [ ] **Logout**: reset user id (`setUserId(id: null)`), clear user properties, rotate the anonymous id (optional; `resetAnalyticsData()` on Firebase clears instance id), clear crash user + account keys. **Do not touch the consent record.**
- [ ] **Account / portal switch** (**flutter-multi-account**): flush events of the old identity first, then `identify(newId)` and update `portal_id` property / keys; emit `account_switched { account_count }` without ids.
- [ ] Never use email/phone as user id; never hash email as id (still personal data and reversible by dictionary).
- [ ] Server-side events (purchases, webhooks) use the same opaque id so funnels join.

## 9. CONSENT — required behaviour

This section is normative. The appendix (A.2–A.4) implements it.

**When to ask**

- [ ] Ask for analytics consent **the first time any user logs in on this device**: right after the first successful login, **before** navigating to home. The prompt is part of the post-login flow (a route the router redirects to while `consent == null`), not a dialog fired from `initState` of the home screen.
- [ ] **Do not ask again** on subsequent logins, re-login after logout, adding a second account, or switching accounts/portals. The choice applies to every account on the device.
- [ ] Re-prompt **once** when the consent policy **version** is bumped (`record.version < kConsentPolicyVersion`).

**Where it's stored**

- [ ] **Device level** in `shared_preferences` (global instance, e.g. key `device.consent.v1`), **not** inside per-account storage, not in an account's Drift DB, not only on the server, not in `flutter_secure_storage` (iOS Keychain survives reinstall — reinstall should re-prompt unless you intentionally want persistence).
- [ ] Record shape: `{ version: int, analytics: bool, crash: bool, decidedAt: ISO-8601 }`.
- [ ] Optionally mirror the decision to the backend (audit trail / cross-device) — but the device record is the source of truth for local gating.

**Logout must not wipe it**

- [ ] `signOut(one)`, `signOut(all)`, "remove account from device", and per-account data wipe **must not delete the consent key**. A common bug is `prefs.clear()` in a "nuke everything" sign-out, or storing consent under an account-scoped prefix that gets deleted with the account. Use a key allow-list for global prefs and namespaced per-account prefs (`acct.<id>.*`) so wipes are targeted.

**Before the first answer**

- [ ] Pre-login and while the prompt is open: **no optional analytics collected, and nothing queued for later send**. The facade drops events (not buffers them) while `consent == null`.
- [ ] Vendor SDK collection **disabled by default at build/config level**:
  - Android `AndroidManifest.xml` `<application>`: `firebase_analytics_collection_enabled=false`, `firebase_crashlytics_collection_enabled=false`, `google_analytics_adid_collection_enabled=false` (unless ATT/ads), Consent Mode defaults `google_analytics_default_allow_analytics_storage=false` etc.
  - iOS `Info.plist`: `FIREBASE_ANALYTICS_COLLECTION_ENABLED=NO`, `FirebaseCrashlyticsCollectionEnabled=NO`, `GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE=NO` (verify key names), `GOOGLE_ANALYTICS_IDFV_COLLECTION_ENABLED` as required.
  - Sentry: initialise only after the crash decision, or init with `beforeSend` / `beforeSendTransaction` returning `null` while not allowed (Sentry has no global "collection disabled" flag in older versions — verify).
  - firebase_performance: `firebase_performance_collection_enabled=false` / `firebase_performance_collection_deactivated` (Android meta-data) and `FirebasePerformance.instance.setPerformanceCollectionEnabled(false)`.
- [ ] After opt-in: `setAnalyticsCollectionEnabled(true)`, `setConsent(analyticsStorageConsentGranted: true, ...)`, `setCrashlyticsCollectionEnabled(record.crash)`.

**Crash reporting**

- [ ] Separate toggle `crash`. Default depends on your legal basis: legitimate interest → crash reporting on from first launch (state it in the privacy policy, allow opt-out) — still gated by `crash` toggle; consent basis → off until yes. Whatever you choose, **apply the same decision to every crash SDK** (Crashlytics and Sentry).
- [ ] With Crashlytics automatic collection off, reports are kept locally: on opt-in call `sendUnsentReports()`; on opt-out `deleteUnsentReports()`.

**Changing it later**

- [ ] Settings → Privacy: two toggles (analytics, crash), applied **immediately**: enable/disable SDK collection, on analytics opt-out **delete queued events** in your batcher and call `resetAnalyticsData()` (Firebase), optionally call your backend deletion endpoint / vendor deletion API (GA4 User Deletion API, Sentry has none client-side).
- [ ] Changing in Settings updates `decidedAt` and keeps `version` current; it never re-triggers the first-login prompt.

**Reinstall & backup**

- [ ] Reinstall clears `shared_preferences` on iOS (NSUserDefaults is in the app container) → re-prompt. Don't move consent to Keychain unless you want persistence across reinstall.
- [ ] Android Auto Backup **includes** shared_preferences by default: restore on a new device brings consent back. Decide explicitly: exclude `FlutterSharedPreferences.xml` (or the `SharedPreferencesAsync` DataStore file) via `android:dataExtractionRules` / `fullBackupContent` if consent must be re-asked per device.

**iOS ATT**

- [ ] ATT is **separate** and only needed if you track across apps/websites (IDFA, ad networks). Sequence: app-level consent prompt first → if the user opted into analytics **and** you need tracking, then `AppTrackingTransparency.requestTrackingAuthorization()` (system prompt, once ever). If analytics was declined, don't show ATT. If ATT denied, keep analytics (first-party) per their consent but no IDFA/ad attribution.
- [ ] ATT must be requested while the app is `active` (not during launch transition) — wait for the first frame / `AppLifecycleState.resumed`.
- [ ] `NSUserTrackingUsageDescription` in Info.plist if you call it; App Review rejects ATT prompts preceded by misleading pre-prompts.

## 10. Delivery

- [ ] **Batching**: vendors batch internally (Firebase ~hourly dispatch on Android, on background on iOS). For a first-party collector, batch by count (e.g. 50) or time (30s) and on `AppLifecycleState.paused`.
- [ ] **Offline queue**: persist pending events (small Drift table / file) with a cap (e.g. 1,000 events or 1 MB, drop oldest); only events **after consent** enter the queue.
- [ ] **Retry** with exponential backoff + jitter; drop on 4xx schema errors (log non-fatal once), retry on 5xx/timeout (**flutter-network-offline-sync**).
- [ ] **Sampling**: performance traces and high-volume events (`tracesSampleRate`, `profilesSampleRate`), never sample business-critical events (purchase, signup). Record sample rate as a prop.
- [ ] **Background flush**: flush on pause; `workmanager` / BGTaskScheduler only if needed — respect consent in the background isolate too (read the same shared_preferences record).
- [ ] Opt-out clears the queue synchronously before returning.
- [ ] Clock: send client timestamp + let server stamp receipt time (clock skew).

## 11. Funnels & key metrics

- [ ] **Activation**: `app_opened` (first), `signup_started` → `signup_completed` → first key action (`first_<core_action>`). Define the activation event with product.
- [ ] **Retention**: D1/D7/D30 cohorts on an engagement event, not just app open.
- [ ] **Purchase funnel** (**flutter-in-app-purchase**): `paywall_viewed { placement }` → `purchase_started { product_id }` → `purchase_completed` / `purchase_failed { reason }` / `purchase_cancelled`; server-validated revenue from backend, not client.
- [ ] **Deep-link attribution** (**flutter-deep-links**): `deep_link_opened { source, campaign, link_type }` with UTM from the URI; deferred deep links attribute on first open after install. Push: `push_received`, `push_opened { campaign_id }` (**flutter-push-notifications**).
- [ ] Each funnel step has exactly one event; define in the tracking plan with expected order.

## 12. Performance monitoring

- [ ] **App start**: cold start to first frame (`WidgetsBinding.instance.addTimingsCallback` / first frame callback); Sentry app start tracking; firebase_performance `_app_start` automatic trace.
- [ ] **Screen render / TTID-TTFD**: Sentry `SentryNavigatorObserver` + `SentryDisplayWidget` / `reportFullyDisplayed()`; custom traces for "time to content".
- [ ] **Frozen / slow frames**: `SchedulerBinding.addTimingsCallback` (build + raster > 16ms slow, > 700ms frozen); Sentry `enableFramesTracking`. Profile with `--profile` + DevTools, not debug.
- [ ] **Network traces**: dio interceptor (Sentry `sentry_dio` `dio.addSentry()`, or firebase_performance `HttpMetric`) — templated URLs only.
- [ ] Gate performance collection by the **analytics** consent (it's usage data) unless legal says otherwise.

## 13. Facade / abstraction

- [ ] `abstract interface class Analytics { track(AnalyticsEvent); identify(String?); setProperty(...); applyConsent(...); flush(); reset(); }` and `abstract interface class CrashReporter { recordError; recordFlutterError; breadcrumb; setUser; setKey; applyConsent; }`.
- [ ] Composite implementations fan out to several vendors; a **consent gate** sits in the composite, so adapters can stay dumb.
- [ ] Provide via Riverpod `Provider` / get_it at app (not account) scope; account-specific identity is pushed in by the session manager.
- [ ] **Fakes for tests**: `RecordingAnalytics` collecting events, `RecordingCrashReporter`. Assert events in widget tests without Firebase.
- [ ] Adapter files are the only ones importing vendor packages (enforce with a lint / `dependency_validator` / custom_lint rule).

## 14. Testing plan

Unit-test `ConsentStore`, the consent gate, catalogue serialization (names/params within limits), identity transitions, and the batcher with fakes. Widget-test the consent screen and the router redirect (`consent == null` after login → consent route). Integration-test on device: forced crash in an internal build appears symbolicated; collection flags off on fresh install (check with Firebase DebugView `-FIRDebugEnabled` / `adb shell setprop debug.firebase.analytics.app <pkg>` — nothing should appear before consent).

### Scenario matrix

| Scenario | Expect |
|---|---|
| Fresh install, app open, pre-login browsing | No analytics events sent or queued; Firebase DebugView empty; crash per legal default |
| First login on device | Consent screen shown after login success, before home; home only after answer |
| Answer "yes" | Collection enabled immediately; first event (`consent_updated`) arrives; unsent crash reports sent if crash yes |
| Answer "no" | No events; `setAnalyticsCollectionEnabled(false)`; queue empty |
| Logout → login same account | **No prompt**; previous choice applied |
| Logout → login different account | **No prompt** |
| Add second account / switch account / switch portal | **No prompt**; user id switched; portal property updated |
| Sign out of all accounts | Consent record still present; next login no prompt |
| Opt-out in Settings | Events stop immediately; queued events deleted; `resetAnalyticsData` called; crash unaffected unless crash toggle changed |
| Opt-in in Settings after "no" | Collection starts; no historical events back-filled |
| Policy version bump (new build) | Prompt once after next launch / login; not again after answer |
| Reinstall (iOS and Android without backup) | Prompt again on first login |
| Android backup restore to new device | Behaves per your documented backup decision |
| Kill app while consent screen open | Relaunch → consent screen again (no record written); no events sent meanwhile |
| ATT: analytics yes + tracking needed | ATT system prompt after consent, app active; denial keeps first-party analytics |
| Uncaught async error (`Future.error` unawaited) | Reported via `PlatformDispatcher.onError`, fatal flag as designed |
| Build/layout error | Reported via `FlutterError.onError`; release shows fallback widget |
| Error in `compute` / background isolate / FCM background handler | Reported with stack |
| Native crash (plugin `fatalError` / `throw RuntimeException`) | Reported on next launch, symbolicated |
| Obfuscated release crash | Dart frames de-obfuscated with file/line after symbols upload |
| ANR (blocking platform channel) on Android 11+ | ANR issue in Crashlytics/Sentry |
| Offline for a day, then online | Queued post-consent events delivered once, ordered, no duplicates |
| Account switch mid-session | Events before switch attributed to old id; after to new |
| Debug build crash | Not in production project |

## 15. Common bugs to look for in review

- Only `FlutterError.onError` set; async errors lost (no `PlatformDispatcher.instance.onError`).
- `runZonedGuarded` used with `ensureInitialized()` outside the zone → "Zone mismatch" warning and missed errors.
- Hooks / Firebase not initialised in background isolates (FCM background handler, workmanager).
- `--obfuscate` builds shipped without uploading `--split-debug-info` symbols, or symbols uploaded from a different build.
- `runtimeType.toString()` used for event or screen names → garbage names under `--obfuscate`.
- Debug and prod share one Firebase/Sentry project; alerts fire on dev crashes.
- Firebase collection on by default (no manifest/plist flags), then disabled in Dart after consent check → first-launch events already sent.
- Events buffered pre-consent "to send if they say yes" — violates the no-queue rule.
- **Consent wiped on logout**: `SharedPreferences.clear()` in sign-out-all, or consent stored under `acct.<id>.` prefix, or in the per-account DB → user re-prompted on every login.
- Consent asked per account, or asked again on add-account / switch.
- Consent stored in `flutter_secure_storage` → survives iOS reinstall unexpectedly; or only on the server → can't gate before network.
- Prompt shown from home `initState` → home events fire before the answer; back button dismisses it with no record.
- Opt-out disables Firebase but not Sentry (or vice versa); performance SDK left on.
- Opt-out doesn't clear the local queue → events sent later.
- Crashlytics unsent reports never sent/deleted after decision.
- Email as user id; PII in exception messages, breadcrumbs with full URLs including tokens.
- User id not reset on logout, or not switched on account switch → one person's events merged into another's.
- Expected errors (offline, 401, cancel) recorded as non-fatals → noise hides real issues; quota exhausted.
- Every rebuild logs `screen_viewed`; automatic + manual screen reporting both on → double counts.
- Event names > 40 chars or > 25 params silently dropped by Firebase.
- ATT prompt requested at launch before the window is active → no prompt shown, status stays `notDetermined`.
- `catch (e)` without stack passed to `recordError` → unusable grouping.

## 16. Output format

**When reviewing** an existing app, respond with:

1. **Context summary** — vendors + versions, channels/flavors and projects, legal basis, ATT need, CI system, multi-account?
2. **Coverage map** — table: Dart framework errors, async errors, isolates, background handlers, native crash, ANR/hang, OOM, symbols (Dart/dSYM/R8/NDK), breadcrumbs, user id, consent gate per SDK — status today vs required.
3. **Consent audit** — checklist from §9 with pass/fail and file:line (storage location, wipe paths, defaults in manifest/plist, prompt placement, settings toggles, version bump).
4. **Findings** by area with **severity** (Critical = data sent without consent, PII leak, or crashes invisible in prod; High; Medium; Low), symptom, fix snippet.
5. **Event catalogue review** — naming, limits, PII, duplicates, missing funnel steps.
6. **Missing scenarios** from §14 with tests to add.
7. **Version notes** — APIs/flags to verify.

**When implementing**, deliver:

1. Native config: manifest meta-data, Info.plist keys, Gradle plugin config, Xcode run script.
2. `main.dart` bootstrap with hooks and consent application; flavor/env tagging.
3. `ConsentStore` (device-level), consent route + router redirect after first login, Settings toggles.
4. `Analytics` / `CrashReporter` facades, consent-gated composite, Firebase + Sentry adapters.
5. Typed event catalogue + screen observer.
6. Identity wiring into login / logout / account switch.
7. CI script for build + symbol upload.
8. Tests for the consent scenarios and capture hooks; assumptions and versions to verify.

## Appendix — Reference implementation (Dart 3.5+, Flutter 3.24+, firebase_crashlytics 4/5, firebase_analytics 11/12, sentry_flutter 8/9, shared_preferences 2.3+)

Simplified; names are illustrative. Verify package APIs against your versions.

### A.0 Native config and CI

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside <application> -->
<meta-data android:name="firebase_analytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_crashlytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_performance_collection_enabled" android:value="false" />
<meta-data android:name="google_analytics_adid_collection_enabled" android:value="false" />
<meta-data android:name="google_analytics_default_allow_analytics_storage" android:value="false" />
<meta-data android:name="google_analytics_default_allow_ad_storage" android:value="false" />
<meta-data android:name="google_analytics_automatic_screen_reporting_enabled" android:value="false" />
<!-- iOS Info.plist: FIREBASE_ANALYTICS_COLLECTION_ENABLED=NO, FirebaseCrashlyticsCollectionEnabled=NO,
     GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE=NO, FirebaseAutomaticScreenReportingEnabled=NO -->
```

```bash
# CI (per platform/flavor). Symbols must come from THIS build; full flags in section 3.
flutter build appbundle --release --flavor prod --obfuscate --split-debug-info=build/symbols/android
firebase crashlytics:symbols:upload --app="$FIREBASE_ANDROID_APP_ID" build/symbols/android
# iOS: same with `flutter build ipa` + build/symbols/ios; dSYMs via the flutterfire run script.
# Sentry alternative: pubspec `sentry: { upload_debug_symbols: true }` + `dart run sentry_dart_plugin`.
```

### A.1 Typed event catalogue

```dart
sealed class AnalyticsEvent {
  const AnalyticsEvent();
  String get name;                       // object_action, <= 40 chars
  int get version => 1;
  Map<String, Object> get params;
}

final class ScreenViewed extends AnalyticsEvent {
  const ScreenViewed(this.screenName);
  final String screenName;               // route NAME, never path with ids
  @override String get name => 'screen_viewed';
  @override Map<String, Object> get params => {'screen_name': screenName};
}

// SignupCompleted({method: SignupMethod enum}), PurchaseCompleted({productId, placement}): same shape.

final class AccountSwitched extends AnalyticsEvent {
  const AccountSwitched({required this.accountCount});
  final int accountCount;                // counts, never ids/emails
  @override String get name => 'account_switched';
  @override Map<String, Object> get params => {'account_count': accountCount};
}

final class ConsentUpdated extends AnalyticsEvent {
  const ConsentUpdated({required this.source});
  final String source;                   // 'first_login' | 'settings' | 'policy_bump'
  @override String get name => 'consent_updated';
  @override Map<String, Object> get params => {'source': source};
}
```

### A.2 ConsentStore (device level)

```dart
import 'dart:async';
import 'dart:convert';
import 'package:shared_preferences/shared_preferences.dart';

const kConsentPolicyVersion = 2;

class ConsentRecord {
  const ConsentRecord({required this.version, required this.analytics,
      required this.crash, required this.decidedAt});
  final int version;
  final bool analytics, crash;
  final DateTime decidedAt;

  Map<String, Object> toJson() => {'version': version, 'analytics': analytics,
      'crash': crash, 'decidedAt': decidedAt.toUtc().toIso8601String()};
  static ConsentRecord fromJson(Map<String, dynamic> j) => ConsentRecord(
      version: j['version'] as int, analytics: j['analytics'] as bool,
      crash: j['crash'] as bool, decidedAt: DateTime.parse(j['decidedAt'] as String));
}

/// Device-level. Lives OUTSIDE any account session. Sign-out code must never
/// call [SharedPreferencesAsync.clear] — wipe per-account keys (`acct.<id>.*`) only.
class ConsentStore {
  ConsentStore(this._prefs);
  static const _key = 'device.consent'; // global key, not account-prefixed
  final SharedPreferencesAsync _prefs;
  final _changes = StreamController<ConsentRecord?>.broadcast();
  ConsentRecord? _current;

  ConsentRecord? get current => _current;
  Stream<ConsentRecord?> get changes => _changes.stream;

  /// Null record or outdated version => must prompt (once) at next login.
  bool get needsPrompt => _current == null || _current!.version < kConsentPolicyVersion;

  /// Collection allowed only with a CURRENT decision; outdated record keeps
  /// its previous choice until re-asked (choose stricter if legal requires).
  bool get analyticsAllowed => _current?.analytics ?? false;
  bool crashAllowed({required bool defaultWhenUndecided}) =>
      _current?.crash ?? defaultWhenUndecided;

  Future<void> load() async {
    final raw = await _prefs.getString(_key);
    try {
      _current = raw == null ? null : ConsentRecord.fromJson(jsonDecode(raw) as Map<String, dynamic>);
    } catch (_) {
      _current = null; // corrupt => ask again
    }
  }

  Future<void> save({required bool analytics, required bool crash}) async {
    final r = ConsentRecord(version: kConsentPolicyVersion, analytics: analytics,
        crash: crash, decidedAt: DateTime.now());
    await _prefs.setString(_key, jsonEncode(r.toJson()));
    _current = r;
    _changes.add(r);
  }
}

/// Example of a sign-out wipe that preserves device-level keys.
Future<void> wipeAccountPrefs(SharedPreferencesAsync prefs, String accountId) async {
  final keys = await prefs.getKeys();
  await prefs.clear(allowList: keys.where((k) => k.startsWith('acct.$accountId.')).toSet());
}
```

### A.3 Consent prompt hooked into first login (go_router)

```dart
// Router redirect: after login, before home, if consent needs a prompt.
GoRouter buildRouter(AuthState auth, ConsentStore consent, Listenable refresh) => GoRouter(
  refreshListenable: refresh, // merges auth + consent.changes
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginScreen()),
    GoRoute(path: '/consent', builder: (_, __) => const ConsentScreen(source: 'first_login')),
    GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
  ],
  redirect: (context, state) {
    final loc = state.matchedLocation;
    if (!auth.isSignedIn) return loc == '/login' ? null : '/login';
    if (consent.needsPrompt) return loc == '/consent' ? null : '/consent';
    if (loc == '/login' || loc == '/consent') return '/home';
    return null;
  },
);

class ConsentScreen extends StatefulWidget {
  const ConsentScreen({super.key, required this.source});
  final String source;
  @override State<ConsentScreen> createState() => _ConsentScreenState();
}

class _ConsentScreenState extends State<ConsentScreen> {
  bool _crash = true; // pre-checked only if legal basis allows; else false
  bool _busy = false;

  Future<void> _decide(bool analytics) async {
    setState(() => _busy = true);
    final deps = AppDeps.of(context);
    await deps.consent.save(analytics: analytics, crash: _crash); // router redirects to /home
    await deps.applyConsent(deps.consent.current!);
    if (analytics) {
      deps.analytics.track(ConsentUpdated(source: widget.source));
      await deps.maybeRequestAtt(); // only if tracking is actually needed
    }
  }

  @override
  Widget build(BuildContext context) => PopScope(
    canPop: false, // no dismiss without a decision
    child: Scaffold(
      body: SafeArea(child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(crossAxisAlignment: CrossAxisAlignment.stretch, children: [
          Text('Help us improve the app', style: Theme.of(context).textTheme.headlineSmall),
          const SizedBox(height: 12),
          const Text('We collect anonymous usage data to improve features. '
              'You can change this any time in Settings → Privacy.'),
          SwitchListTile(value: _crash, onChanged: (v) => setState(() => _crash = v),
              title: const Text('Send crash reports')),
          const Spacer(),
          FilledButton(onPressed: _busy ? null : () => _decide(true), child: const Text('Allow')),
          const SizedBox(height: 8),
          OutlinedButton(onPressed: _busy ? null : () => _decide(false), child: const Text("Don't allow")),
        ]),
      )),
    ),
  );
}
// Equal prominence for Allow / Don't allow (dark-pattern rules under GDPR).
```

### A.4 Analytics facade with consent gate

```dart
abstract interface class Analytics {
  void track(AnalyticsEvent event);
  Future<void> identify(String? opaqueUserId);
  Future<void> setProperty(String name, String? value);
  Future<void> applyConsent(ConsentRecord? record);
  Future<void> flush();
  Future<void> reset();
}

/// Gate sits in the composite: events are DROPPED (not buffered) without consent.
class GatedAnalytics implements Analytics {
  GatedAnalytics(this._sinks, this._consent, this._common);
  final List<Analytics> _sinks;
  final ConsentStore _consent;
  final Map<String, Object> Function() _common; // app_version, env, portal_id...

  bool get _allowed => _consent.analyticsAllowed;

  @override
  void track(AnalyticsEvent e) {
    if (!_allowed) return;
    assert(e.name.length <= 40 && e.params.length <= 20);
    for (final s in _sinks) { s.track(e); }
  }

  @override
  Future<void> identify(String? id) async {
    if (!_allowed) return;
    for (final s in _sinks) { await s.identify(id); }
  }

  @override
  Future<void> setProperty(String n, String? v) async {
    if (!_allowed) return;
    for (final s in _sinks) { await s.setProperty(n, v); }
  }

  @override
  Future<void> applyConsent(ConsentRecord? r) async {
    for (final s in _sinks) { await s.applyConsent(r); }
  }

  @override Future<void> flush() async { for (final s in _sinks) { await s.flush(); } }
  @override Future<void> reset() async { for (final s in _sinks) { await s.reset(); } }
}

class FirebaseAnalyticsSink implements Analytics {
  FirebaseAnalyticsSink(this._fa, this._common);
  final FirebaseAnalytics _fa;
  final Map<String, Object> Function() _common;

  @override
  void track(AnalyticsEvent e) {
    if (e is ScreenViewed) { _fa.logScreenView(screenName: e.screenName); return; }
    _fa.logEvent(name: e.name, parameters: {..._common(), ...e.params, 'v': e.version});
  }

  @override Future<void> identify(String? id) => _fa.setUserId(id: id);
  @override Future<void> setProperty(String n, String? v) => _fa.setUserProperty(name: n, value: v);

  @override
  Future<void> applyConsent(ConsentRecord? r) async {
    final on = r?.analytics ?? false;
    await _fa.setConsent(analyticsStorageConsentGranted: on, adStorageConsentGranted: false,
        adUserDataConsentGranted: false, adPersonalizationSignalsConsentGranted: false);
    await _fa.setAnalyticsCollectionEnabled(on);
    if (!on) await _fa.resetAnalyticsData(); // drops instance id + local queue
  }

  @override Future<void> flush() async {} // SDK-managed
  @override Future<void> reset() async { await _fa.setUserId(id: null); }
}

/// Test fake.
class RecordingAnalytics implements Analytics {
  final events = <AnalyticsEvent>[];
  String? userId;
  @override void track(AnalyticsEvent e) => events.add(e);
  @override Future<void> identify(String? id) async => userId = id;
  @override Future<void> setProperty(String n, String? v) async {}
  @override Future<void> applyConsent(ConsentRecord? r) async { if (!(r?.analytics ?? false)) events.clear(); }
  @override Future<void> flush() async {}
  @override Future<void> reset() async => userId = null;
}
```

### A.5 CrashReporter wrapper with breadcrumbs, and bootstrap

```dart
abstract interface class CrashReporter {
  Future<void> recordError(Object e, StackTrace? st, {String? reason, bool fatal = false});
  Future<void> recordFlutterError(FlutterErrorDetails d, {bool fatal = true});
  void breadcrumb(String message, {String category = 'app', Map<String, Object?>? data});
  Future<void> setUser(String? opaqueId);
  Future<void> setKey(String key, Object value);
  Future<void> applyConsent(bool crashAllowed);
}

class CrashlyticsReporter implements CrashReporter {
  final _c = FirebaseCrashlytics.instance;
  @override
  Future<void> recordError(Object e, StackTrace? st, {String? reason, bool fatal = false}) =>
      _c.recordError(e, st, reason: reason, fatal: fatal);
  @override
  Future<void> recordFlutterError(FlutterErrorDetails d, {bool fatal = true}) =>
      fatal ? _c.recordFlutterFatalError(d) : _c.recordFlutterError(d);
  @override
  void breadcrumb(String m, {String category = 'app', Map<String, Object?>? data}) =>
      _c.log('[$category] ${m.length > 200 ? m.substring(0, 200) : m}');
  @override Future<void> setUser(String? id) => _c.setUserIdentifier(id ?? '');
  @override Future<void> setKey(String k, Object v) => _c.setCustomKey(k, v);
  @override
  Future<void> applyConsent(bool on) async {
    await _c.setCrashlyticsCollectionEnabled(on && !kDebugMode);
    if (await _c.checkForUnsentReports()) {
      on ? await _c.sendUnsentReports() : await _c.deleteUnsentReports();
    }
  }
}

class SentryReporter implements CrashReporter {
  static bool allowed = false; // read by beforeSend
  @override
  Future<void> recordError(Object e, StackTrace? st, {String? reason, bool fatal = false}) =>
      Sentry.captureException(e, stackTrace: st, hint: reason == null ? null : Hint.withMap({'reason': reason}));
  @override
  Future<void> recordFlutterError(FlutterErrorDetails d, {bool fatal = true}) =>
      Sentry.captureException(d.exception, stackTrace: d.stack);
  @override
  void breadcrumb(String m, {String category = 'app', Map<String, Object?>? data}) =>
      Sentry.addBreadcrumb(Breadcrumb(message: m, category: category, data: data));
  @override
  Future<void> setUser(String? id) =>
      Sentry.configureScope((s) => s.setUser(id == null ? null : SentryUser(id: id)));
  @override Future<void> setKey(String k, Object v) => Sentry.configureScope((s) => s.setTag(k, '$v'));
  @override Future<void> applyConsent(bool on) async => allowed = on;
}

class CompositeCrashReporter implements CrashReporter {
  CompositeCrashReporter(this._all);
  final List<CrashReporter> _all;
  @override
  Future<void> recordError(Object e, StackTrace? st, {String? reason, bool fatal = false}) async {
    for (final r in _all) { await r.recordError(e, st, reason: reason, fatal: fatal); }
  }
  @override Future<void> recordFlutterError(FlutterErrorDetails d, {bool fatal = true}) async {
    for (final r in _all) { await r.recordFlutterError(d, fatal: fatal); } }
  @override void breadcrumb(String m, {String category = 'app', Map<String, Object?>? data}) {
    for (final r in _all) { r.breadcrumb(m, category: category, data: data); } }
  @override Future<void> setUser(String? id) async { for (final r in _all) { await r.setUser(id); } }
  @override Future<void> setKey(String k, Object v) async { for (final r in _all) { await r.setKey(k, v); } }
  @override Future<void> applyConsent(bool on) async { for (final r in _all) { await r.applyConsent(on); } }
}

const kCrashDefaultWhenUndecided = false; // set per legal basis (legitimate interest => true)

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final consent = ConsentStore(SharedPreferencesAsync());
  await consent.load();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  final info = await PackageInfo.fromPlatform();
  const env = String.fromEnvironment('APP_ENV', defaultValue: 'debug');

  await SentryFlutter.init((o) {
    o.dsn = kDebugMode ? '' : const String.fromEnvironment('SENTRY_DSN');
    o.release = '${info.packageName}@${info.version}+${info.buildNumber}';
    o.environment = env;
    o.sendDefaultPii = false;
    o.tracesSampleRate = 0.1;
    o.beforeSend = (event, hint) => SentryReporter.allowed ? event : null;
    o.beforeSendTransaction = (tx, [hint]) => consent.analyticsAllowed ? tx : null; // verify signature
  }, appRunner: () async {
    final crash = CompositeCrashReporter([CrashlyticsReporter(), SentryReporter()]);
    await crash.applyConsent(consent.crashAllowed(defaultWhenUndecided: kCrashDefaultWhenUndecided));
    await crash.setKey('env', env);

    FlutterError.onError = (details) {
      if (kDebugMode) FlutterError.presentError(details);
      crash.recordFlutterError(details, fatal: true);
    };
    PlatformDispatcher.instance.onError = (error, stack) {
      crash.recordError(error, stack, fatal: true);
      return true;
    };
    Isolate.current.addErrorListener(RawReceivePort((List<dynamic> pair) {
      crash.recordError(pair.first as Object,
          StackTrace.fromString(pair.last?.toString() ?? ''), fatal: true);
    }).sendPort);

    final analytics = GatedAnalytics(
        [FirebaseAnalyticsSink(FirebaseAnalytics.instance, () => {'env': env})], consent, () => {});
    await analytics.applyConsent(consent.current);
    consent.changes.listen((r) async {
      await analytics.applyConsent(r);
      await crash.applyConsent(r?.crash ?? kCrashDefaultWhenUndecided);
    });
    runApp(App(consent: consent, analytics: analytics, crash: crash));
  });
}
```

### A.6 Identity on login, logout, and account switch

```dart
/// Called by SessionManager (see flutter-multi-account). Never touches ConsentStore.
class TelemetryIdentity {
  TelemetryIdentity(this._analytics, this._crash);
  final Analytics _analytics;
  final CrashReporter _crash;

  Future<void> onLogin(String opaqueUserId, {String? portalId}) async {
    await _analytics.identify(opaqueUserId);
    await _analytics.setProperty('portal_id', portalId);
    await _crash.setUser(opaqueUserId);
    await _crash.setKey('active_portal_id', portalId ?? 'none');
    _crash.breadcrumb('login', category: 'auth');
  }

  Future<void> onAccountSwitch({required String toUserId, String? toPortalId,
      required int accountCount}) async {
    await _analytics.flush(); // attribute pending events to the OLD id
    await onLogin(toUserId, portalId: toPortalId);
    _analytics.track(AccountSwitched(accountCount: accountCount));
  }

  Future<void> onPortalSwitch(String portalId) async {
    await _analytics.setProperty('portal_id', portalId);
    await _crash.setKey('active_portal_id', portalId);
    _crash.breadcrumb('portal_switch', category: 'nav');
  }

  Future<void> onLogout({required bool lastAccount}) async {
    await _analytics.flush();
    await _analytics.reset();          // user id null; consent untouched
    await _crash.setUser(null);
    await _crash.setKey('active_portal_id', 'none');
    _crash.breadcrumb('logout', category: 'auth');
  }
}

/// Screen tracking from route names (obfuscation-safe).
class AnalyticsRouteObserver extends NavigatorObserver {
  AnalyticsRouteObserver(this._analytics, this._crash);
  final Analytics _analytics;
  final CrashReporter _crash;
  @override
  void didPush(Route route, Route? previous) => _emit(route);
  @override
  void didPop(Route route, Route? previous) { if (previous != null) _emit(previous); }
  void _emit(Route r) {
    final name = r.settings.name;
    if (name == null || r is PopupRoute) return;
    _analytics.track(ScreenViewed(name));
    _crash.breadcrumb(name, category: 'navigation');
  }
}

/// Settings toggle: apply immediately.
Future<void> onSettingsChanged(ConsentStore store, Analytics a,
    {required bool analytics, required bool crash}) async {
  await store.save(analytics: analytics, crash: crash); // listeners apply to SDKs
  if (analytics) a.track(const ConsentUpdated(source: 'settings'));
  // optional: if (!analytics) await api.requestAnalyticsDeletion();
}
```

### A.7 Tests (sketch)

```dart
// setUp: SharedPreferencesAsyncPlatform.instance = InMemorySharedPreferencesAsync.empty();
// 1. No events before consent: GatedAnalytics([RecordingAnalytics()], store, ...) drops track() until
//    store.save(analytics: true, ...); afterwards exactly one ScreenViewed is recorded.
// 2. Sign-out wipe keeps device consent: save consent, wipeAccountPrefs(prefs, 'A'), reload a new
//    ConsentStore -> needsPrompt is false, while 'acct.A.*' keys are gone.
// 3. Policy bump re-prompts once: stored version 1 -> needsPrompt true; after save -> false.
```
