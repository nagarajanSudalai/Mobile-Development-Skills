---
name: android-crash-analytics
description: Use when implementing, reviewing, testing, or debugging crash reporting, product analytics, and performance monitoring in a native Android app (Kotlin, Jetpack Compose) — Firebase Crashlytics and Sentry setup behind a vendor-neutral CrashReporter facade, R8/ProGuard mapping upload and NDK native symbols (Crashlytics Gradle plugin, Sentry Gradle plugin, Play Console), fatal and non-fatal capture, ANRs, OOM and native crashes via ApplicationExitInfo (API 30+), breadcrumbs and custom keys without PII, grouping/fingerprints, crash-free users and release health gates, Android vitals thresholds, typed analytics event catalogue (object_action), Compose screen tracking, anonymous id → login → logout identity, user id switch on account/portal switch, device-level analytics consent asked once after the first login (DataStore, survives logout), manifest flags that disable auto-collection until opt-in (firebase_analytics_collection_enabled, firebase_crashlytics_collection_enabled, io.sentry.auto-init), Consent Mode v2, batching/offline queue/WorkManager flush, funnels and attribution, app start / jank / network traces. Siblings: ios-crash-analytics, flutter-crash-analytics, react-native-crash-analytics; related: android-multi-account, android-deep-links, android-push-notifications, android-in-app-purchase, android-list-ux, android-network-offline-sync.
---

# Android Crash Reporting & Analytics Skill

Guide for building, auditing, and testing **crash reporting**, **product analytics**, and **performance monitoring** in a consumer Android app (Kotlin, Jetpack Compose, Hilt). Three concerns, deliberately kept apart:

1. **Crash reporting** — fatal JVM crashes, native (NDK) crashes, ANRs, OOM/low-memory kills, handled (non-fatal) errors. Needs symbolication (R8 `mapping.txt`, unstripped `.so`), release tags, triage.
2. **Product analytics** — a typed catalogue of events (`object_action`), screen views, user properties, funnels. Needs identity rules and **consent**.
3. **Performance** — app start, screen render, frozen frames/jank, network traces.

Vendors shown concretely: **Firebase Crashlytics**, **Firebase Analytics**, **Sentry** — always behind two app-owned interfaces, `CrashReporter` and `Analytics`, so vendors can be swapped, combined, consent-gated in one place, and faked in tests.

Sibling skills: **ios-crash-analytics**, **flutter-crash-analytics**, **react-native-crash-analytics** — same structure per platform. Related: **android-multi-account** (user id + active account/portal context on switch), **android-deep-links** (attribution, `deep_link_opened`), **android-push-notifications** (`push_opened`), **android-in-app-purchase** (purchase funnel), **android-list-ux** (jank on lists), **android-network-offline-sync** (network traces, offline queue patterns).

> Vendor SDKs and Gradle plugins move fast: the Crashlytics Gradle plugin 3.x replaced the old `firebaseCrashlytics {}` NDK flags, Firebase Analytics added Consent Mode v2 keys in 2024, the Sentry Android SDK changed major versions (7 → 8) and its Gradle plugin options, and Google Play's Android vitals thresholds are revised periodically. **Verify every API, manifest key and Gradle option against the versions in the project's `libs.versions.toml` / Firebase BoM and the current vendor docs before writing code**, and prefer what the project already uses.
>
> Version gates used below: `ApplicationExitInfo` / `getHistoricalProcessExitReasons` API 30+ (ANR `traceInputStream` API 30+, `setProcessStateSummary` API 30+); `reportFullyDrawn` meaningful for Play vitals from API 29+; `android:dataExtractionRules` API 31+ (`fullBackupContent` for 23–30); `POST_NOTIFICATIONS` API 33+ (affects push-open funnels); JankStats needs `androidx.metrics:metrics-performance` and uses `FrameMetrics` API 24+ (falls back below); AGP 8.x for `ndk.debugSymbolLevel` packaging into the AAB.

## 0. Mental model

```
 Build time                                   Runtime (process)
 +--------------------------------+          +---------------------------------------------+
 | R8 -> mapping.txt (per variant)|          | Application.onCreate                        |
 | NDK -> unstripped .so          |          |   1. ConsentStore.load() (device DataStore) |
 | Gradle plugins upload:         |          |   2. CrashReporter.init(consent.crash)      |
 |   Crashlytics  (mapping + .so) |          |   3. Analytics.init(consent.analytics)      |
 |   Sentry       (mapping + .so) |          |   manifest: auto-collection OFF by default  |
 | AAB: mapping + native symbols  |          +----------------------+----------------------+
 |   -> Play Console vitals       |                                 |
 +--------------------------------+          app code only sees facades:
                                             +----------------------v----------------------+
                                             | CrashReporter            Analytics           |
                                             |  breadcrumb/key/record    track(Event)       |
                                             |  setUser(opaqueId)        screen(name)       |
                                             |                           identify/reset     |
                                             +-----+-----------+--------+-------+-----------+
                                                   |           |                |
                                     ConsentGate --+-----------+----------------+ (drops when off)
                                                   v           v                v
                                            Crashlytics      Sentry      Firebase Analytics /
                                                                         own pipeline (batched)

 Identity:  install -> anonymousId (device) --login--> userId (opaque) + accountId/portalId props
            account/portal switch -> setUser(new)    last logout -> reset (anonymousId kept or rotated)
 Consent:   device-level record {version, analytics, crash, decidedAt}; asked once after FIRST login;
            survives logout / sign-out-all / account wipe; reinstall or policy bump -> ask again
```

| Thing | Device-level | Per account | Per portal |
|---|---|---|---|
| Consent record | ✅ (DataStore, global) | ❌ never | ❌ |
| Anonymous install id | ✅ | | |
| Analytics / crash user id | | ✅ opaque id | |
| `portal_id`, `role` user properties | | | ✅ |
| Event queue (own pipeline) | ✅ (events carry accountId) | | |
| Crash custom keys | ✅ app/build | `account_slot` | `portal_id` |

## 1. First, establish context

Find out (from the codebase or by asking):

- **Tools in place**: Crashlytics, Sentry, Bugsnag, Datadog, Embrace? Firebase Analytics, Amplitude, Mixpanel, PostHog, Segment, in-house pipeline? Firebase Performance? Is there already a facade?
- **Legal basis & consent rules**: GDPR/UK GDPR/ePrivacy (EU users → opt-in for non-essential analytics), CCPA/CPRA (opt-out of "sale/share"), children (COPPA, Families policy — many SDKs disallowed). Is crash reporting treated as **legitimate interest** (default on, opt-out) or consent (default off)? Who signed off (DPO/legal)? Google Play **Data safety** form must match what you collect.
- **Consent UX**: is there an existing CMP (Google UMP, OneTrust, Usercentrics)? Ads in the app (then UMP/TCF and Consent Mode v2 matter)?
- **Release channels**: internal / closed / open testing / production tracks, staged rollouts %, flavors (`dev`, `staging`, `prod`), debug builds. Separate Firebase projects / Sentry projects or environments per channel?
- **Build**: R8 full mode? `minifyEnabled` on release? NDK / third-party `.so` (game engine, media, crypto)? AAB to Play? CI system (GitHub Actions, Bitrise, GitLab) and where secrets live (`SENTRY_AUTH_TOKEN`, Firebase service account)?
- **Backend pipeline**: do events go to vendor only, or also BigQuery export / warehouse / own ingestion endpoint? Event schema registry? Who owns the catalogue (product analytics team)?
- **Multi-account**: several accounts/portals on one device (see **android-multi-account**)? What is the opaque user id?
- **KPIs**: activation definition, retention cohorts, purchase funnel, deep-link attribution source (Play Install Referrer, AppsFlyer, Adjust, Branch)?

## 2. Crash reporting setup

### 2.1 Checklist

- [ ] Initialise in `Application.onCreate` **first** (before Hilt-injected heavy work, before WorkManager init, before any network) — but **after** reading the consent record, so the SDK starts in the right collection state. Never in an `Activity`.
- [ ] Prefer manual init for Sentry (`io.sentry.auto-init=false` in manifest, then `SentryAndroid.init`) so DSN, environment, release and consent are applied together.
- [ ] Crashlytics auto-initialises via its `ContentProvider`/component registrar — control it with `firebase_crashlytics_collection_enabled` (manifest) and `setCrashlyticsCollectionEnabled()` (runtime, persisted by the SDK).
- [ ] **Release tag**: `"<applicationId>@<versionName>+<versionCode>"` (Sentry `release`); Crashlytics uses versionName/versionCode automatically. Add `dist = versionCode`.
- [ ] **Environment**: `debug` / `internal` / `beta` / `production` (Sentry `environment`; Crashlytics custom key `channel`). Derive from `BuildConfig` + install source (`PackageManager.getInstallSourceInfo` API 30+).
- [ ] **Separate projects**: debug builds → separate Firebase project (`google-services.json` per flavor/buildType in `src/debug/`) or disable collection in debug; Sentry → separate project or at least `environment=debug` and filtered alerts. Never let debug noise into production crash-free numbers.
- [ ] Custom keys at init: `build_type`, `flavor`, `git_sha`, `install_source`, `channel`. No PII.
- [ ] Only **one** process-wide uncaught exception handler chain: both SDKs chain to the previous handler — init order doesn't drop crashes, but don't install your own handler that swallows (`Thread.setDefaultUncaughtExceptionHandler` must call the previous one).
- [ ] Multi-process apps (`:remote` services): SDKs init per process; guard by `Application.getProcessName()` (API 28+) and decide per process.
- [ ] Do not run both Crashlytics **and** Sentry NDK signal handlers unless verified — they can conflict. Pick one NDK owner (see 4.4).

### 2.2 Gradle (version catalog)

```toml
# libs.versions.toml (verify current versions)
[versions]
firebaseBom = "34.x.x"
crashlyticsPlugin = "3.0.x"
sentryPlugin = "5.x.x"
[libraries]
firebase-bom = { module = "com.google.firebase:firebase-bom", version.ref = "firebaseBom" }
firebase-crashlytics = { module = "com.google.firebase:firebase-crashlytics" }
firebase-crashlytics-ndk = { module = "com.google.firebase:firebase-crashlytics-ndk" }
firebase-analytics = { module = "com.google.firebase:firebase-analytics" }
firebase-perf = { module = "com.google.firebase:firebase-perf" }
[plugins]
crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "crashlyticsPlugin" }
sentry = { id = "io.sentry.android.gradle", version.ref = "sentryPlugin" }
```

> Since Firebase BoM 34, KTX modules were folded into the main artifacts (`firebase-analytics` instead of `firebase-analytics-ktx`). Check the project's BoM.

### 2.3 Manifest: collection OFF until consent

```xml
<application ...>
    <!-- Firebase Analytics: no collection, no queued events until setAnalyticsCollectionEnabled(true) -->
    <meta-data android:name="firebase_analytics_collection_enabled" android:value="false" />
    <!-- Consent Mode v2 defaults (needed if ads / Google Ads linking; harmless otherwise) -->
    <meta-data android:name="google_analytics_default_allow_analytics_storage" android:value="false" />
    <meta-data android:name="google_analytics_default_allow_ad_storage" android:value="false" />
    <meta-data android:name="google_analytics_default_allow_ad_user_data" android:value="false" />
    <meta-data android:name="google_analytics_default_allow_ad_personalization_signals" android:value="false" />
    <!-- Don't collect Advertising ID unless the product needs it (Data safety + AD_ID permission) -->
    <meta-data android:name="google_analytics_adid_collection_enabled" android:value="false" />
    <!-- Crashlytics: reports are stored locally but NOT sent until enabled -->
    <meta-data android:name="firebase_crashlytics_collection_enabled" android:value="false" />
    <!-- Firebase Performance: off until consent -->
    <meta-data android:name="firebase_performance_collection_enabled" android:value="false" />
    <!-- Sentry: never auto-init from manifest; we init manually with consent applied -->
    <meta-data android:name="io.sentry.auto-init" android:value="false" />
</application>
```

- [ ] If the app does not use the Advertising ID, add `<uses-permission android:name="com.google.android.gms.permission.AD_ID" tools:node="remove" />` (the analytics SDK merges it in).
- [ ] `firebase_analytics_collection_deactivated=true` permanently disables Analytics (cannot be re-enabled at runtime) — **do not** use it for consent.
- [ ] With Crashlytics collection disabled, crashes are kept on disk; on opt-in call `sendUnsentReports()`, on opt-out `deleteUnsentReports()`. Decide explicitly (see section 9).

## 3. Symbolication (R8 mapping + NDK)

### 3.1 What needs uploading

| Artifact | Produced by | Needed by | Notes |
|---|---|---|---|
| `mapping.txt` | R8 (`build/outputs/mapping/<variant>/`) | Crashlytics, Sentry, Play Console | One per build; must match exactly (Crashlytics injects a mapping file id; Sentry injects a ProGuard UUID) |
| Unstripped `.so` / debug symbols | NDK / CMake (`build/intermediates/merged_native_libs/...`) | Crashlytics NDK, Sentry, Play | Third-party prebuilt `.so` often have no symbols — ask vendor |
| `native-debug-symbols.zip` | AGP `ndk.debugSymbolLevel` | Play Console | Packaged in AAB automatically when set |
| Kotlin source context | Sentry plugin `includeSourceContext` | Sentry | Optional; uploads source — check policy |

### 3.2 Checklist

- [ ] `isMinifyEnabled = true` on release, R8 full mode default (AGP 8+). Keep rules for reflection-based serialization; **don't** keep everything just to get readable traces — upload mapping instead.
- [ ] Crashlytics: plugin applied; `mappingFileUploadEnabled` default true for minified builds; NDK: `nativeSymbolUploadEnabled = true` and run `uploadCrashlyticsSymbolFile<Variant>` after `assemble`/`bundle`.
- [ ] Sentry: `autoUploadProguardMapping = true`, `uploadNativeSymbols = true`, `includeNativeSources` optional; `SENTRY_AUTH_TOKEN`, `org`, `projectName` from CI secrets — **never** committed (`sentry.properties` with token in git is a common leak).
- [ ] Play Console: AAB carries `mapping.txt` (R8 deobfuscation file is included in the bundle) and native symbols when `ndk { debugSymbolLevel = "FULL" }` (or `SYMBOL_TABLE` to keep size down).
- [ ] CI: upload on every build that can reach users (internal track included). Fail the build if upload fails for release (`continue-on-error: false`).
- [ ] Archive `mapping.txt` + `.so` as CI artifacts keyed by `versionCode` for manual retrace (`retrace` from cmdline-tools, `ndk-stack`).
- [ ] Debug builds: disable mapping upload (`mappingFileUploadEnabled = false` in debug) to save CI time.
- [ ] Verify: force a crash in a release build installed from the internal track → stack trace readable within minutes in both consoles.

### 3.3 Gradle config

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    alias(libs.plugins.crashlytics)
    alias(libs.plugins.sentry) // only if Sentry is used
}

android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            ndk { debugSymbolLevel = "FULL" } // -> native-debug-symbols in AAB for Play
            configure<com.google.firebase.crashlytics.buildtools.gradle.CrashlyticsExtension> {
                mappingFileUploadEnabled = true
                nativeSymbolUploadEnabled = true // requires firebase-crashlytics-ndk
            }
        }
        debug {
            configure<com.google.firebase.crashlytics.buildtools.gradle.CrashlyticsExtension> {
                mappingFileUploadEnabled = false
            }
        }
    }
}

sentry {
    org.set("my-org")
    projectName.set("my-android-app")
    authToken.set(providers.environmentVariable("SENTRY_AUTH_TOKEN"))
    autoUploadProguardMapping.set(true)
    uploadNativeSymbols.set(true)
    includeNativeSources.set(false)
    includeSourceContext.set(false)
    // The plugin can auto-install integrations (OkHttp, Room, Compose, Timber...) via bytecode
    // instrumentation. Be explicit about what you want; instrumentation adds build time.
    tracingInstrumentation { enabled.set(true) }
    autoInstallation { enabled.set(true) }
}
```

```yaml
# CI (GitHub Actions excerpt)
- run: ./gradlew bundleProdRelease uploadCrashlyticsSymbolFileProdRelease
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
- uses: actions/upload-artifact@v4
  with:
    name: symbols-${{ github.run_number }}
    path: |
      app/build/outputs/mapping/prodRelease/mapping.txt
      app/build/intermediates/merged_native_libs/prodRelease/**/*.so
```

## 4. Capture: fatal, non-fatal, ANR, OOM, native

### 4.1 Fatal & non-fatal

- [ ] Uncaught JVM exceptions: captured automatically by both SDKs.
- [ ] Coroutines: an uncaught exception in `viewModelScope`/`lifecycleScope` crashes the app (good — it's reported). A `CoroutineExceptionHandler` that only logs **hides** crashes; if you add one, call `crashReporter.recordNonFatal(e)`.
- [ ] Never report `CancellationException` as an error — rethrow it.
- [ ] **Non-fatals**: report unexpected states (decoding failures, invariant violations, server 5xx bursts, failed purchases acknowledgement) with a stable `context` tag, not every network timeout. Rate-limit per type per session (e.g. max 10).
- [ ] RxJava: `RxJavaPlugins.setErrorHandler` → `UndeliverableException` → non-fatal, not crash.
- [ ] WebView: `onRenderProcessGone` (API 26+) → non-fatal + recover; otherwise the app crashes.
- [ ] `Flow.catch` that swallows everything → report before falling back.
- [ ] Strict mode in debug only (`StrictMode.setVmPolicy(... penaltyListener)` API 28+ can forward violations to non-fatals in internal builds).

### 4.2 ANRs

- [ ] Play vitals is the ground truth for ANR rate (user-perceived ANR rate). Crashlytics reports ANRs on API 30+ from `ApplicationExitInfo` (shown as ANR events with main-thread trace); Sentry has a watchdog ANR detector (`isAnrEnabled`, `anrTimeoutIntervalMillis` default 5000) and, in v7+, `ApplicationExitInfo`-based ANR v2 on API 30+ (`isAnrEnabled` + historical reporting). Prefer AEI-based ANRs on 30+ — the watchdog has false positives (debugger, background).
- [ ] Don't enable ANR detection in two SDKs with alerts in both; pick one for alerting.
- [ ] Common Android ANR causes to check in review: disk/DataStore/SharedPreferences `commit()` on main, `runBlocking` on main (e.g. in an OkHttp `Authenticator` called on main, or reading consent synchronously!), `BroadcastReceiver.onReceive` long work, `Service.startForeground` not called within 5–10 s, content provider init (too many `ContentProvider` initializers → use App Startup), binder calls on main, deadlocks with `synchronized`.
- [ ] **Startup consent read**: reading DataStore with `runBlocking` in `Application.onCreate` is one disk read (~ms) — acceptable once, but measure; alternative: mirror the consent booleans into a tiny `SharedPreferences` for synchronous startup read, or init SDKs disabled and enable when the Flow emits (see A.3).

### 4.3 OOM, low memory, exit reasons (ApplicationExitInfo)

- [ ] On API 30+, at startup call `ActivityManager.getHistoricalProcessExitReasons(packageName, 0, 20)` and report reasons **since last check** (store last processed timestamp in device DataStore): `REASON_ANR`, `REASON_CRASH_NATIVE`, `REASON_LOW_MEMORY`, `REASON_EXCESSIVE_RESOURCE_USAGE`, `REASON_INITIALIZATION_FAILURE`, `REASON_SIGNALED` (often OOM killer / lmkd), `REASON_USER_REQUESTED` (ignore), `REASON_PERMISSION_CHANGE` (ignore, but useful), `REASON_DEPENDENCY_DIED`.
- [ ] Distinguish **foreground** vs background kills via `importance` — only foreground ones are user-visible.
- [ ] `setProcessStateSummary(bytes)` (API 30+, ≤128 bytes) to stash current screen/session id so the next launch can attribute an exit to a screen.
- [ ] Java `OutOfMemoryError` is captured as a crash; native/lmkd kills are **not** — AEI is the only signal.
- [ ] Send AEI summaries as non-fatals or as analytics `app_exit` events (with consent) — not both if they double-count.
- [ ] `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` → breadcrumb (useful context for later kills).

### 4.4 Native (NDK)

- [ ] Crashlytics: add `firebase-crashlytics-ndk` (it replaces `firebase-crashlytics`, includes it) and upload symbols. Sentry: `sentry-android` includes `sentry-android-ndk` by default; exclude it if Crashlytics owns NDK (`exclude(group = "io.sentry", module = "sentry-android-ndk")`) or disable via `options.isEnableNdk = false`.
- [ ] Only one signal handler owner. If you run both SDKs, typical split: Crashlytics owns crashes (JVM + NDK), Sentry owns performance + non-fatals — or vice versa. Document it.
- [ ] Third-party `.so` without symbols → still upload what you have; traces show offsets.

## 5. Context: breadcrumbs, custom keys, user

- [ ] Breadcrumbs (via `CrashReporter.breadcrumb`): navigation route changes (route **pattern**, e.g. `order/{id}`, never the filled-in path with ids/emails), lifecycle (`app_foreground`/`background`), network failures (method + path template + status), `account_switch`, `portal_switch`, `consent_changed`, `purchase_started`, low memory. Crashlytics: `log()`; Sentry: `addBreadcrumb`.
- [ ] Custom keys (overwrite, not append): `portal_id` (opaque), `role`, `account_count`, `feature_flags` (short hash or few key flags), `network_type`, `locale`, `dark_mode`, `last_screen`.
- [ ] User: **opaque** server user id (or HMAC of it); never email/phone/name. Crashlytics `setUserId`, Sentry `setUser(User().apply { id = ... })`. Turn off Sentry `isSendDefaultPii` (default false — keep it).
- [ ] **Multi-account** (see **android-multi-account**): on account switch set new user id **after** flushing events of the old account; on portal switch update `portal_id` key and breadcrumb. On last sign-out clear user (`setUserId("")` for Crashlytics, `Sentry.setUser(null)`).
- [ ] Scrub PII: Sentry `beforeSend` / `beforeBreadcrumb` to drop query strings, tokens, `Authorization` headers; OkHttp integration must not capture bodies. Crashlytics has no beforeSend — so don't put PII in exception messages (`"User john@x.com not found"` is PII).
- [ ] Exception messages with dynamic values break grouping *and* leak PII — use typed exceptions with fixed messages and put values in keys.

## 6. Grouping & triage

- [ ] **Fingerprints**: Sentry `event.fingerprint = listOf("{{ default }}", errorCode)` to split; or a fixed fingerprint to merge noisy variants. Crashlytics groups by stack; for non-fatals wrap in typed exceptions (`class PaymentAckFailed(code: Int) : Exception("payment_ack_failed")`) so they group sanely.
- [ ] Tag issues by `release`, `environment`, `channel`; mark resolved "in next release" so regressions reopen.
- [ ] **Regression alerts**: new issue in latest release with > N users; velocity alert (Crashlytics "trending stability" / Sentry issue alerts); route to on-call channel. Ignore debug env.
- [ ] **Crash-free users & sessions**: track per release; Sentry Release Health needs sessions on (`isEnableAutoSessionTracking`, default on — respects consent only if SDK is disabled when no consent).
- [ ] **Release health gate** for staged rollout: e.g. halt at 10% if crash-free users < 99.5% or ANR rate > 0.3%, or any new fatal in top 5. Use Play Developer API to halt rollout from CI/bot if automated.
- [ ] **Android vitals bad-behaviour thresholds** (check current values in Play Console docs): user-perceived crash rate ≥ 1.09% and user-perceived ANR rate ≥ 0.47% overall; 8% on any single device model. Above them, Play can reduce visibility and show warnings on the listing.
- [ ] Weekly triage ritual: top issues by users, new in release, ANRs by device, OOM by device RAM class.

## 7. Event design

### 7.1 Rules

- [ ] Naming: `object_action`, snake_case, past tense action: `signup_completed`, `item_added_to_cart` → prefer `cart_item_added`, `paywall_viewed`, `purchase_completed`. Firebase limits: event name ≤ 40 chars, ≤ 25 params per event, param name ≤ 40, string value ≤ 100 chars, 500 distinct event names per app — design within them even if you use another vendor.
- [ ] Reserved names: don't use Firebase reserved prefixes `firebase_`, `google_`, `ga_` or reserved events (`app_remove`, `first_open`, `session_start`...). Use recommended events (`purchase`, `login`, `sign_up`) only if you want their GA4 reports.
- [ ] **Typed catalogue**: a sealed interface `AnalyticsEvent` with one data class per event; params as typed fields; required props enforced by constructor. No `track("some string", mapOf(...))` in feature code.
- [ ] Global props added by facade, not per call site: `app_version`, `schema_version`, `portal_id`, `account_slot`, `platform=android`, `session_id`.
- [ ] **Versioning**: `schema_version` on every event; changing meaning of a param → new event or new param name, never reuse. Deprecations tracked in the catalogue with removal release.
- [ ] Enum-like params as Kotlin enums with `wireName` (`source = PaywallSource.Onboarding` → `"onboarding"`), never free text.
- [ ] No PII in params (emails, names, free-text search queries, exact location). IDs only if opaque and approved.
- [ ] Catalogue is the source for the data team's schema (export the sealed classes to JSON in a unit test, or keep a YAML registry and generate Kotlin).

### 7.2 Screen tracking in Compose

- [ ] Compose has no Activity per screen → Firebase auto screen reporting only sees Activities. Disable auto (`google_analytics_automatic_screen_reporting_enabled=false`) and track from Navigation: `NavController.addOnDestinationChangedListener` → `analytics.screen(route pattern)`.
- [ ] Use the **route pattern** (`destination.route`, e.g. `product/{id}`), or with type-safe navigation map the destination class to a stable screen name. Never the filled route.
- [ ] Bottom sheets / dialogs that matter → explicit `screen()` call.
- [ ] Deduplicate on recomposition and configuration change (listener fires once per navigation, not per recomposition — don't track in a composable body without `LaunchedEffect`).

## 8. Identity

- [ ] **Anonymous id**: random UUID generated on first launch, stored in device DataStore (non-backed-up). Used as pre-login identity *only once consent allows collection*.
- [ ] **Login**: `identify(userId)` with opaque id; vendors that support alias (Mixpanel/Amplitude/PostHog) link anonymous → user; Firebase: `setUserId`.
- [ ] **Logout of last account**: `reset()` → clear user id, clear user properties; rotate anonymous id so the next person on a shared device isn't linked. Consent stays (section 9).
- [ ] **Account switch** (**android-multi-account**): flush queue for A → `identify(B)` → set `portal_id`/`role` → `account_switched` event (attributed to B, with `account_count`). Same for crash user.
- [ ] **Portal switch**: user id stays; update `portal_id` user property + crash key; `portal_switched` event.
- [ ] Events created before the switch must carry A's identity even if delivered later — stamp `userId`/`accountId` on the event at creation time in your own queue.

## 9. CONSENT — required behaviour

This section is normative. Implementation in **A.2–A.5**.

### 9.1 Rules

- [ ] **When**: ask for analytics consent the **first time any user logs in on this device** — right after the first successful login, **before navigating to home**. Not on app start, not pre-login.
- [ ] **Where stored**: **device level** — a global DataStore file (`device_consent.preferences_pb`), **not** inside a per-account DataStore/Room DB, and not server-only. It applies to every account on the device.
- [ ] **Asked once**: do **not** ask again on subsequent logins, after logout + re-login, on "add account", or on account/portal switch.
- [ ] **Never wiped by sign-out**: logout, "sign out of all accounts", and per-account data wipe must **not** delete the consent record. ⚠️ Common bug: `context.filesDir.deleteRecursively()` / clearing "all DataStores" / `ActivityManager.clearApplicationUserData()` on sign-out-all erases consent → user gets re-prompted (or worse, the app defaults to collecting). The data cleaner must have an explicit allow-list that keeps `device_consent`.
- [ ] **Before the first answer** (pre-login, during login, while the prompt is open): **no** optional analytics collected **or queued for later**. Vendor SDKs disabled at build/config level via the manifest flags in 2.3; your own queue refuses to enqueue (don't buffer "just in case").
- [ ] **Crash reporting**: separate toggle `crash`. Default per your legal basis — commonly legitimate interest → crash reporting **on** by default with opt-out in Settings and disclosed in the privacy policy; if legal says consent → off until opt-in. Whatever the choice, apply it to **both** Crashlytics and Sentry (and Firebase Performance, which follows `analytics` or its own toggle). Pre-consent crashes: with Crashlytics collection disabled, reports wait on disk → send on opt-in, delete on opt-out.
- [ ] **Settings toggle**: change any time, applied **immediately**: enable/disable SDK collection (`setAnalyticsCollectionEnabled`, `isCrashlyticsCollectionEnabled`, Sentry close/re-init, Perf `isPerformanceCollectionEnabled`), on opt-out **delete queued events** (own queue, `deleteUnsentReports()`, `FirebaseAnalytics.resetAnalyticsData()`), optionally call your backend's deletion endpoint / vendor deletion API (GDPR erasure).
- [ ] **Consent record**: `{ version, analytics: Boolean, crash: Boolean, decidedAt }`. `version` = current policy version constant; if stored `version < CURRENT` → re-prompt **once** after next login/at next foreground for a signed-in user, then store the new version.
- [ ] **Reinstall** clears it: DataStore lives in app-private storage; uninstall deletes it. **Backups**: Android Auto Backup would *restore* it on reinstall/new device. Decide: default here is **exclude** the consent file from cloud backup and device transfer (consent is given per device), via `dataExtractionRules` / `fullBackupContent` — so a reinstall re-prompts. If legal prefers carrying it to a new device, include it and document that.
- [ ] **Consent Mode v2** (if Firebase Analytics + Google Ads): call `FirebaseAnalytics.setConsent(mapOf(ANALYTICS_STORAGE to GRANTED/DENIED, AD_STORAGE..., AD_USER_DATA..., AD_PERSONALIZATION...))` together with `setAnalyticsCollectionEnabled`.
- [ ] If the app also uses Google UMP / TCF for ads: UMP has its own flow and storage; keep this analytics consent separate, sequence it (login → analytics consent → UMP if needed), and don't contradict.
- [ ] Children/restricted accounts: if any account is a child account, consider forcing analytics off for the device while that account is active (document).

### 9.2 Flow

```
 App start ──> ConsentStore.read()
                │ null or version < CURRENT -> SDKs stay disabled; queue rejects
                │ decided                    -> apply(analytics, crash)
 Login success ─┤
                ├── record == null || record.version < CURRENT ?
                │        yes -> navigate("consent") (back = no answer, cannot skip to home)
                │               onDecision -> store {CURRENT, a, c, now} -> apply -> navigate(home)
                │        no  -> navigate(home)        (2nd login, add account, switch: no prompt)
 Settings toggle -> store -> apply -> on opt-out purge queues + deleteUnsent + resetAnalyticsData
 Sign out / sign out all / account wipe -> consent untouched
```

### 9.3 Backup rules

```xml
<!-- res/xml/data_extraction_rules.xml (API 31+) -->
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="file" path="datastore/device_consent.preferences_pb" />
        <exclude domain="file" path="datastore/device_analytics.preferences_pb" />
    </cloud-backup>
    <device-transfer>
        <exclude domain="file" path="datastore/device_consent.preferences_pb" />
        <exclude domain="file" path="datastore/device_analytics.preferences_pb" />
    </device-transfer>
</data-extraction-rules>
<!-- res/xml/backup_rules.xml (API 23-30, android:fullBackupContent) -->
<full-backup-content>
    <exclude domain="file" path="datastore/device_consent.preferences_pb" />
    <exclude domain="file" path="datastore/device_analytics.preferences_pb" />
</full-backup-content>
```

`device_analytics` holds the anonymous id and the exit-info cursor — excluded so a restored device gets a fresh anonymous id.

## 10. Delivery

- [ ] Vendor SDKs batch and retry themselves (Firebase Analytics dispatches roughly hourly or on background; don't try to force it). Debug with `adb shell setprop debug.firebase.analytics.app <pkg>` → DebugView.
- [ ] Own pipeline: events → Room/file queue (append-only, capped e.g. 5,000 events / 2 MB, drop oldest) → batch POST (≤ 100 events / 500 KB, gzip) → delete on 2xx; on 4xx drop batch (schema error, report non-fatal); on 5xx/timeout keep and back off (exponential + jitter, see **android-network-offline-sync**).
- [ ] Flush triggers: batch size reached, every N minutes in foreground, `ProcessLifecycleOwner` `ON_STOP` → enqueue unique `OneTimeWorkRequest` with `NetworkType.CONNECTED` (WorkManager survives process death). No foreground service for analytics.
- [ ] Each event carries a client `event_id` (UUID) for server dedupe and `client_ts` + upload `sent_at` for clock-skew correction.
- [ ] **Sampling**: performance traces/transactions sampled (`tracesSampleRate` 0.05–0.2 in prod); errors at 1.0; analytics events never sampled client-side unless a high-volume event is explicitly marked.
- [ ] Opt-out → `queue.clear()` + cancel the flush work.
- [ ] Respect data saver / metered? Batches are small; fine on metered, but large payloads (session replay, attachments) → unmetered only.

## 11. Funnels & key metrics

- [ ] **Activation**: define one event (e.g. `first_project_created`) and its window (D1/D7); log `signup_completed` with `method`.
- [ ] **Retention**: rely on vendor sessions (`session_start` / Sentry sessions) + opaque user id; D1/D7/D30 cohorts by install week and acquisition source.
- [ ] **Purchase funnel** (**android-in-app-purchase**): `paywall_viewed(source, placement)` → `purchase_started(product_id)` → `purchase_completed(product_id, price_micros, currency)` | `purchase_failed(billing_response_code)` | `purchase_cancelled`. Revenue truth is server (RTDN), not client events — mark client events as funnel only.
- [ ] **Deep-link attribution** (**android-deep-links**): `deep_link_opened(link_type, campaign, source)`; Play Install Referrer (`com.android.installreferrer`) read once on first launch, stored, attached as user property after consent. Strip PII from UTM params.
- [ ] **Push** (**android-push-notifications**): `push_received` (only if data message handled by app), `push_opened(campaign_id)`, `notification_permission_result(granted)` (API 33+).
- [ ] Every funnel step event has the same `flow_id` to stitch steps within a session.

## 12. Performance monitoring

- [ ] **App start**: Play vitals (cold start > 5 s is "excessive"); call `reportFullyDrawn()` (Compose: `ReportDrawnWhen { dataLoaded }` from `activity-compose`) when first meaningful content is shown. Firebase Perf `_app_start` trace; Sentry app start (`isEnableAppStartProfiling` / app start spans).
- [ ] **Baseline Profiles** + Macrobenchmark `StartupTimingMetric` in CI — fix before you monitor.
- [ ] **Screen render / jank**: `JankStats` (`androidx.metrics`) per screen with state (`PerformanceMetricsState.putState("screen", route)`); Sentry frames tracking (slow/frozen frames) on transactions; Firebase Perf screen traces only for Activities/Fragments (Compose → custom traces). Frozen frame = > 700 ms.
- [ ] **Network traces**: OkHttp interceptor (Sentry `SentryOkHttpInterceptor`, Firebase Perf auto-instruments OkHttp via plugin). Path templates, not raw URLs with ids. Disable body capture.
- [ ] **Custom traces**: `checkout_load`, `search_results` — start/stop around user-visible waits.
- [ ] Perf collection obeys consent (manifest flag in 2.3 + runtime toggle).
- [ ] Watch overhead: Sentry profiling / session replay off by default; enable for small sample after privacy review (replay must mask text/images).

## 13. Facade / abstraction layer

- [ ] Two interfaces in a `:core:telemetry` module: `CrashReporter`, `Analytics` (+ optional `PerformanceMonitor`). Feature modules depend only on these.
- [ ] Implementations: `CrashlyticsReporter`, `SentryReporter`, `CompositeCrashReporter`; `FirebaseAnalyticsSink`, `HttpAnalyticsSink`; `ConsentGatedAnalytics` decorator in front of all sinks.
- [ ] Hilt `@Binds @IntoSet` for sinks; debug builds bind `LogcatAnalytics`; tests bind `FakeAnalytics` recording events.
- [ ] The facade owns: global props, PII guard (debug assertion on param values that look like emails), consent gate, identity stamping, rate limits.
- [ ] No vendor import (`com.google.firebase.analytics`, `io.sentry`) outside `:core:telemetry` — enforce with a lint rule / Konsist / `dependencyAnalysis`.

## 14. Testing plan

1. **Unit**: `ConsentStore` read/write/version bump; `ConsentGatedAnalytics` drops events when undecided/denied and queues nothing; `FirstLoginConsentGate.needsPrompt()` truth table; identity ordering on switch (flush → identify); catalogue: every event name ≤ 40 chars, ≤ 25 params, snake_case, unique — reflection test over sealed subclasses.
2. **Data cleaner**: after `signOutAll()` the `device_consent` file still exists and decodes to the same record.
3. **Integration (Robolectric / instrumented)**: fake SDK adapters receive `setCollectionEnabled(false)` before any `log` on fresh install; enable after opt-in.
4. **Symbolication**: release build from CI on internal track → `crashReporter.testCrash()` (hidden debug menu) → readable trace in Crashlytics and Sentry; native test crash → readable frames.
5. **ANR**: debug menu "block main thread 10 s" → Play/Crashlytics/Sentry show ANR on next launch (API 30+ AEI).
6. **OOM / kill**: `adb shell am kill` (background), `adb shell am crash <pkg>`; allocate until OOM in debug menu → AEI reason reported next launch.
7. **Analytics debug**: `adb shell setprop debug.firebase.analytics.app <pkg>` → DebugView; verify no events before consent (DebugView empty), events after opt-in, none after opt-out. Sentry: `options.isDebug = true` in debug only.
8. **Network**: proxy (Charles/mitmproxy on a debug build with user CA trust) → verify no request to `app-measurement.com`, `firebaselogging`, `*.ingest.sentry.io` before consent.
9. **Compose UI test**: login → consent screen shown → choose → home; second login → straight to home.
10. **Backup**: `adb shell bmgr backupnow <pkg>`, uninstall, reinstall with restore → consent re-prompted (if excluded).

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| C1 | Fresh install, before login | No analytics events sent or queued; SDK collection disabled; no vendor network calls (except crash per legal basis) |
| C2 | First login ever on device | Consent screen shown after login success, before home; back can't bypass |
| C3 | Opt in | Analytics enabled immediately; `consent_updated` first event; unsent crash reports sent if crash=true |
| C4 | Opt out | Nothing sent; queue empty; unsent crash reports deleted if crash=false |
| C5 | Logout, log in again (same account) | **No** prompt; previous choice applied |
| C6 | Log in with a second account / add account | **No** prompt; choice applies to account B |
| C7 | Switch account / portal | No prompt; user id switched; `portal_id` updated |
| C8 | Sign out of all accounts, log in | **No** prompt; consent record intact |
| C9 | Per-account data wipe | Consent intact |
| C10 | Change toggle in Settings (on → off) | Collection stops instantly; queued events deleted; optional server deletion requested |
| C11 | Settings off → on | Collection resumes; no backfill of events from the off period |
| C12 | Policy version bump in new release | Prompt once after next login/foreground for signed-in user; not again |
| C13 | Reinstall | Prompt again at first login (consent excluded from backup) |
| C14 | Restore from backup to new device | Same as reinstall (or carried over, if you chose to include — documented) |
| C15 | Process death while consent screen open | On relaunch, still undecided → screen shown again; nothing collected |
| C16 | Crash before consent decided | Per legal basis: stored and sent/deleted after decision, or sent (legit interest) |
| S1 | Release crash with R8 | Deobfuscated in Crashlytics, Sentry, Play |
| S2 | Native crash | Symbolicated frames; one handler fired (no double report) |
| S3 | ANR on API 30+ | Reported once (AEI), main-thread trace attached |
| S4 | Low-memory kill in foreground | `app_exit` / non-fatal with reason + last screen next launch |
| S5 | Non-fatal storm (offline) | Rate-limited; timeouts not reported as non-fatals |
| S6 | Staged rollout regression | Alert fires; gate halts rollout |
| E1 | Event catalogue lint | All names valid; no duplicates; no PII params |
| E2 | Screen view on navigation | One `screen_view` per destination with route pattern; none on rotation |
| E3 | Offline for a day | Events queued (≤ cap), flushed on reconnect, deduped by `event_id` |
| E4 | Event created for A, sent after switch to B | Attributed to A |
| E5 | Last logout | User id cleared; anonymous id rotated; consent kept |

## 15. Common bugs to look for in review

- Sign-out-all / "clear data" deleting **all** DataStore files or `filesDir` → consent wiped → re-prompt (or collection default-on)
- Consent stored in the per-account DataStore / Room DB → second account prompted again, or account B inherits nothing
- Consent stored server-side only → lost on logout, can't apply pre-login
- Prompting on app start (pre-login) or on every login; prompting on account add/switch
- Manifest flags missing → Firebase Analytics collects `first_open`/`session_start` before consent; Sentry auto-inits from `io.sentry.dsn`
- Using `firebase_analytics_collection_deactivated` for consent → can never re-enable
- Own queue buffering events pre-consent "to send after opt-in"
- Opt-out disables SDK but leaves queued events / unsent Crashlytics reports to go out later
- Consent applied to Crashlytics but not Sentry (or Perf)
- Consent read with `runBlocking` on main in multiple places → startup ANR
- No mapping upload in CI (or wrong variant) → obfuscated traces; `SENTRY_AUTH_TOKEN` committed in `sentry.properties`
- Keeping all classes (`-keep class ** { *; }`) to avoid mapping files
- Two NDK signal handlers (Crashlytics NDK + Sentry NDK) → missing or duplicated native crashes
- `CoroutineExceptionHandler` / `runCatching` swallowing errors without reporting; reporting `CancellationException`
- Exception messages with emails/ids → PII and broken grouping
- User id = email; not cleared on last logout; not switched on account switch; switched before old events flushed
- Screen tracking in composable body → duplicate screen views on recomposition; filled routes with ids
- Debug builds reporting into production project → polluted crash-free rate
- Event names > 40 chars or > 25 params silently dropped by Firebase; free-text params
- ANR watchdog and AEI both enabled with alerts in two tools → double counting
- Anonymous id in backed-up storage → two devices share one id after restore

## 16. Output format

When asked to **review/audit**:
1. **Summary** — tools and versions (Firebase BoM, Crashlytics/Sentry SDK + Gradle plugins), legal basis for crash vs analytics, consent storage location, channels/projects, symbol upload path
2. **Checklist results** — sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line; section 9 (consent) reported first and in full
3. **Critical issues first** (collection before consent, consent wiped on logout, PII in events/crashes, missing symbol upload, swallowed crashes) with fix code
4. **Testing gaps** — C1–C16, S1–S6, E1–E5 not covered, and which need a real device / release build

When asked to **implement**: fill in the context (section 1) first, then start from the Appendix: copy the event catalogue, consent store, consent gate, facades and identity sync; wire the manifest flags and Gradle symbol upload; delete the vendor adapters not used; verify SDK APIs against the project's versions; finish with the scenario rows you ran and those left for the user.

---

## Appendix — Reference implementation (Kotlin, Compose, Hilt, DataStore)

> Written against Kotlin 2.x, Hilt 2.5x, DataStore 1.1, Navigation Compose 2.8+, Firebase BoM 33/34, Sentry Android 7.x/8.x. Vendor method names are the usual suspects to drift — verify.

### A.1 Typed event catalogue

```kotlin
package app.telemetry

sealed interface AnalyticsEvent {
    val name: String
    fun params(): Map<String, Any> // String, Long, Double, Boolean only
}

enum class LoginMethod(val wire: String) { Password("password"), Google("google"), Passkey("passkey") }
enum class PaywallSource(val wire: String) { Onboarding("onboarding"), Settings("settings"), Feature("feature_gate") }

object Events {
    data class LoginCompleted(val method: LoginMethod, val isNewDevice: Boolean) : AnalyticsEvent {
        override val name = "login_completed"
        override fun params() = mapOf("method" to method.wire, "is_new_device" to isNewDevice)
    }
    data class AccountSwitched(val accountCount: Int) : AnalyticsEvent {
        override val name = "account_switched"
        override fun params() = mapOf("account_count" to accountCount.toLong())
    }
    data class PortalSwitched(val portalCount: Int) : AnalyticsEvent {
        override val name = "portal_switched"
        override fun params() = mapOf("portal_count" to portalCount.toLong())
    }
    data class PaywallViewed(val source: PaywallSource, val placement: String) : AnalyticsEvent {
        override val name = "paywall_viewed"
        override fun params() = mapOf("source" to source.wire, "placement" to placement)
    }
    data class PurchaseCompleted(val productId: String, val priceMicros: Long, val currency: String) : AnalyticsEvent {
        override val name = "purchase_completed"
        override fun params() = mapOf("product_id" to productId, "price_micros" to priceMicros, "currency" to currency)
    }
    data class DeepLinkOpened(val linkType: String, val campaign: String?) : AnalyticsEvent {
        override val name = "deep_link_opened"
        override fun params() = buildMap<String, Any> {
            put("link_type", linkType); campaign?.let { put("campaign", it) }
        }
    }
    data class ConsentUpdated(val analytics: Boolean, val crash: Boolean, val policyVersion: Int) : AnalyticsEvent {
        override val name = "consent_updated"
        override fun params() = mapOf("analytics" to analytics, "crash" to crash, "policy_version" to policyVersion.toLong())
    }
}

const val SCHEMA_VERSION = 3L
```

### A.2 Device-level ConsentStore

```kotlin
package app.telemetry.consent

import android.content.Context
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.*

const val CURRENT_POLICY_VERSION = 2

data class ConsentRecord(val version: Int, val analytics: Boolean, val crash: Boolean, val decidedAtMillis: Long)

// GLOBAL file, created from applicationContext. Never inside an account-scoped component,
// never deleted by the account data cleaner (see A.7), excluded from backup (section 9.3).
private val Context.consentDataStore by preferencesDataStore(name = "device_consent")

class ConsentStore(context: Context) {
    private val ds = context.applicationContext.consentDataStore
    private object K {
        val version = intPreferencesKey("version")
        val analytics = booleanPreferencesKey("analytics")
        val crash = booleanPreferencesKey("crash")
        val decidedAt = longPreferencesKey("decided_at")
    }

    val record: Flow<ConsentRecord?> = ds.data
        .catch { emit(emptyPreferences()) } // corrupt file -> treat as undecided (collect nothing)
        .map { p ->
            val v = p[K.version] ?: return@map null
            ConsentRecord(v, p[K.analytics] ?: false, p[K.crash] ?: false, p[K.decidedAt] ?: 0L)
        }
        .distinctUntilChanged()

    suspend fun current(): ConsentRecord? = record.first()

    suspend fun needsPrompt(): Boolean = current().let { it == null || it.version < CURRENT_POLICY_VERSION }

    suspend fun save(analytics: Boolean, crash: Boolean, now: Long = System.currentTimeMillis()) {
        ds.edit {
            it[K.version] = CURRENT_POLICY_VERSION
            it[K.analytics] = analytics
            it[K.crash] = crash
            it[K.decidedAt] = now
        }
    }
}

/** Effective state used by SDKs. Undecided => analytics off; crash => legal-basis default. */
data class EffectiveConsent(val analytics: Boolean, val crash: Boolean, val decided: Boolean)

// Set from legal decision: true = legitimate interest (crash on until opt-out), false = opt-in.
const val CRASH_DEFAULT_BEFORE_DECISION = false

fun ConsentRecord?.effective(): EffectiveConsent =
    if (this == null) EffectiveConsent(analytics = false, crash = CRASH_DEFAULT_BEFORE_DECISION, decided = false)
    // Outdated version: keep honouring the old choice until re-asked (or force off - legal call).
    else EffectiveConsent(analytics, crash, decided = version >= CURRENT_POLICY_VERSION)
```

### A.3 Applying consent to vendor SDKs

```kotlin
package app.telemetry.consent

import com.google.firebase.analytics.FirebaseAnalytics
import com.google.firebase.analytics.FirebaseAnalytics.ConsentStatus.DENIED
import com.google.firebase.analytics.FirebaseAnalytics.ConsentStatus.GRANTED
import com.google.firebase.analytics.FirebaseAnalytics.ConsentType
import com.google.firebase.crashlytics.FirebaseCrashlytics
import com.google.firebase.perf.FirebasePerformance
import io.sentry.Sentry
import io.sentry.android.core.SentryAndroid
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.launchIn
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.onEach

class ConsentApplier(
    private val app: android.app.Application,
    private val store: ConsentStore,
    private val queue: app.telemetry.EventQueue,
    private val sentryDsn: String,
    private val release: String,
    private val environment: String,
) {
    private var sentryOn = false

    /** Called from Application.onCreate. SDKs start disabled via manifest; we only ever turn them on here. */
    fun start(scope: CoroutineScope) {
        store.record.map { it.effective() }.distinctUntilChanged()
            .onEach(::apply)
            .launchIn(scope)
    }

    private fun apply(c: EffectiveConsent) {
        // --- Analytics ---
        val fa = FirebaseAnalytics.getInstance(app)
        val s = if (c.analytics) GRANTED else DENIED
        fa.setConsent(mapOf(ConsentType.ANALYTICS_STORAGE to s, ConsentType.AD_STORAGE to DENIED,
            ConsentType.AD_USER_DATA to DENIED, ConsentType.AD_PERSONALIZATION to DENIED))
        fa.setAnalyticsCollectionEnabled(c.analytics)
        FirebasePerformance.getInstance().isPerformanceCollectionEnabled = c.analytics
        if (!c.analytics) {
            queue.clear()
            if (c.decided) fa.resetAnalyticsData() // opt-out: drop data stored on device
        }

        // --- Crash: Crashlytics + Sentry together ---
        val cl = FirebaseCrashlytics.getInstance()
        cl.isCrashlyticsCollectionEnabled = c.crash
        if (c.decided) { if (c.crash) cl.sendUnsentReports() else cl.deleteUnsentReports() }

        if (c.crash && !sentryOn) {
            SentryAndroid.init(app) { o ->
                o.dsn = sentryDsn
                o.release = release
                o.environment = environment
                o.isSendDefaultPii = false
                o.tracesSampleRate = if (c.analytics) 0.1 else 0.0 // perf traces follow analytics consent
                o.isAnrEnabled = false // Crashlytics owns ANRs here (pick one owner)
                o.isEnableNdk = false  // Crashlytics NDK owns native crashes
                o.setBeforeBreadcrumb { b, _ -> b.apply { data.remove("url") } }
            }
            sentryOn = true
        } else if (!c.crash && sentryOn) {
            Sentry.close(); sentryOn = false
        }
    }
}
```

### A.4 Consent prompt hooked into first login (Compose + Navigation)

```kotlin
package app.auth

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.activity.compose.BackHandler
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.navigation.NavController
import app.telemetry.Analytics
import app.telemetry.Events
import app.telemetry.consent.*
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import kotlinx.coroutines.launch

/** Single post-login router used by login AND add-account flows. */
class PostLoginRouter @Inject constructor(private val consent: ConsentStore) {
    suspend fun destinationAfterLogin(): String = if (consent.needsPrompt()) "consent" else "home"
}

@HiltViewModel
class LoginViewModel @Inject constructor(
    private val router: PostLoginRouter,
    /* auth deps */
) : ViewModel() {
    fun onLoginSucceeded(nav: NavController) = viewModelScope.launch {
        val dest = router.destinationAfterLogin()
        nav.navigate(dest) { popUpTo("login") { inclusive = true } }
    }
}

@HiltViewModel
class ConsentViewModel @Inject constructor(
    private val store: ConsentStore,
    private val analytics: Analytics,
) : ViewModel() {
    fun decide(analyticsOn: Boolean, crashOn: Boolean, onDone: () -> Unit) = viewModelScope.launch {
        store.save(analyticsOn, crashOn)        // ConsentApplier reacts via Flow
        analytics.track(Events.ConsentUpdated(analyticsOn, crashOn, CURRENT_POLICY_VERSION)) // dropped if off
        onDone()
    }
}

@Composable
fun ConsentScreen(vm: ConsentViewModel, onDone: () -> Unit) {
    var crash by remember { mutableStateOf(true) }
    BackHandler { /* must answer; no silent skip to home */ }
    Column(Modifier.fillMaxSize().padding(24.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Text("Help us improve the app", style = MaterialTheme.typography.headlineSmall)
        Text("We'd like to collect anonymous usage analytics. You can change this anytime in Settings.")
        Row(verticalAlignment = androidx.compose.ui.Alignment.CenterVertically) {
            Text("Send crash reports", Modifier.weight(1f)); Switch(crash, { crash = it })
        }
        Button({ vm.decide(true, crash, onDone) }, Modifier.fillMaxWidth()) { Text("Allow analytics") }
        OutlinedButton({ vm.decide(false, crash, onDone) }, Modifier.fillMaxWidth()) { Text("Don't allow") }
    }
}
// NavHost: composable("consent") { ConsentScreen(hiltViewModel()) {
//     nav.navigate("home") { popUpTo("consent") { inclusive = true } } } }
// Equal-weight buttons: no dark pattern. Policy bump while signed in: MainActivity checks
// needsPrompt() on start for a signed-in session and routes to "consent" once.
```

```kotlin
// Settings toggle: same store; applied immediately via ConsentApplier's Flow.
@HiltViewModel
class PrivacySettingsViewModel @Inject constructor(
    private val store: ConsentStore,
    private val deletion: app.telemetry.DataDeletionApi,
) : ViewModel() {
    val state = store.record
    fun setAnalytics(on: Boolean) = viewModelScope.launch {
        val r = store.current(); store.save(on, r?.crash ?: CRASH_DEFAULT_BEFORE_DECISION)
        if (!on) runCatching { deletion.requestDeletion() } // optional server-side erasure
    }
    fun setCrash(on: Boolean) = viewModelScope.launch {
        val r = store.current(); store.save(r?.analytics ?: false, on)
    }
}
```

### A.5 Analytics facade with consent gate

```kotlin
package app.telemetry

import app.telemetry.consent.*
import kotlinx.coroutines.flow.StateFlow
import java.util.UUID

interface Analytics {
    fun track(event: AnalyticsEvent)
    fun screen(name: String)
    fun identify(userId: String, props: Map<String, String> = emptyMap())
    fun setProperty(key: String, value: String?)
    fun reset()
}

interface AnalyticsSink { // FirebaseAnalyticsSink, HttpAnalyticsSink, LogcatSink, FakeSink
    fun send(e: QueuedEvent)
    fun identify(userId: String?); fun setProperty(key: String, value: String?)
}

data class QueuedEvent(
    val id: String, val name: String, val params: Map<String, Any>,
    val userId: String?, val anonymousId: String, val clientTs: Long,
)

class ConsentGatedAnalytics(
    private val consent: StateFlow<EffectiveConsent>, // stateIn(appScope) of store.record.map{effective()}
    private val sinks: Set<AnalyticsSink>,
    private val ids: DeviceIds,                       // device DataStore: anonymousId
    private val globals: () -> Map<String, Any>,      // app_version, portal_id, account_slot...
) : Analytics {
    @Volatile private var userId: String? = null

    private val allowed get() = consent.value.analytics // undecided => false: nothing collected or queued

    override fun track(event: AnalyticsEvent) {
        if (!allowed) return
        val e = QueuedEvent(UUID.randomUUID().toString(), event.name,
            globals() + event.params() + ("schema_version" to SCHEMA_VERSION),
            userId, ids.anonymousId, System.currentTimeMillis()) // identity stamped at creation
        if (BuildConfig.DEBUG) PiiGuard.check(e)
        sinks.forEach { it.send(e) }
    }
    override fun screen(name: String) = track(ScreenView(name))
    override fun identify(userId: String, props: Map<String, String>) {
        this.userId = userId
        if (!allowed) return
        sinks.forEach { s -> s.identify(userId); props.forEach { (k, v) -> s.setProperty(k, v) } }
    }
    override fun setProperty(key: String, value: String?) { if (allowed) sinks.forEach { it.setProperty(key, value) } }
    override fun reset() {
        userId = null
        sinks.forEach { it.identify(null) }
        ids.rotateAnonymousId() // shared device: next person not linked. Consent untouched.
    }
}

private data class ScreenView(val screen: String) : AnalyticsEvent {
    override val name = "screen_view"
    override fun params() = mapOf("screen_name" to screen)
}

object PiiGuard {
    private val email = Regex("[^@\\s]+@[^@\\s]+\\.[^@\\s]+")
    fun check(e: QueuedEvent) = e.params.values.filterIsInstance<String>().forEach {
        check(!email.containsMatchIn(it)) { "PII in ${e.name}" }
    }
}

class FirebaseAnalyticsSink(private val fa: com.google.firebase.analytics.FirebaseAnalytics) : AnalyticsSink {
    override fun send(e: QueuedEvent) = fa.logEvent(e.name, android.os.Bundle().apply {
        e.params.forEach { (k, v) -> when (v) {
            is String -> putString(k, v.take(100)); is Long -> putLong(k, v)
            is Int -> putLong(k, v.toLong()); is Double -> putDouble(k, v)
            is Boolean -> putLong(k, if (v) 1 else 0)
        } }
    })
    override fun identify(userId: String?) = fa.setUserId(userId)
    override fun setProperty(key: String, value: String?) = fa.setUserProperty(key, value)
}
```

Compose screen tracking:

```kotlin
@Composable
fun TrackScreens(nav: NavController, analytics: Analytics) {
    DisposableEffect(nav) {
        val l = NavController.OnDestinationChangedListener { _, dest, _ ->
            analytics.screen(dest.route ?: "unknown") // route pattern, e.g. "order/{id}"
        }
        nav.addOnDestinationChangedListener(l)
        onDispose { nav.removeOnDestinationChangedListener(l) }
    }
}
```

### A.6 CrashReporter wrapper with breadcrumbs + exit reasons

```kotlin
package app.telemetry

import android.app.ActivityManager
import android.app.ApplicationExitInfo
import android.os.Build
import com.google.firebase.crashlytics.FirebaseCrashlytics
import io.sentry.Breadcrumb
import io.sentry.Sentry
import io.sentry.protocol.User

interface CrashReporter {
    fun breadcrumb(category: String, message: String, data: Map<String, String> = emptyMap())
    fun setKey(key: String, value: String)
    fun setUser(opaqueId: String?)
    fun recordNonFatal(t: Throwable, context: String)
}

class CrashlyticsReporter(private val c: FirebaseCrashlytics) : CrashReporter {
    override fun breadcrumb(category: String, message: String, data: Map<String, String>) =
        c.log("[$category] $message ${data.entries.joinToString { "${it.key}=${it.value}" }}")
    override fun setKey(key: String, value: String) = c.setCustomKey(key, value)
    override fun setUser(opaqueId: String?) = c.setUserId(opaqueId ?: "")
    override fun recordNonFatal(t: Throwable, context: String) { c.setCustomKey("nf_context", context); c.recordException(t) }
}

class SentryReporter : CrashReporter {
    override fun breadcrumb(category: String, message: String, data: Map<String, String>) =
        Sentry.addBreadcrumb(Breadcrumb().apply { this.category = category; this.message = message
            data.forEach { (k, v) -> setData(k, v) } })
    override fun setKey(key: String, value: String) = Sentry.setTag(key, value)
    override fun setUser(opaqueId: String?) = Sentry.setUser(opaqueId?.let { User().apply { id = it } })
    override fun recordNonFatal(t: Throwable, context: String) {
        Sentry.captureException(t) { scope -> scope.setTag("nf_context", context)
            scope.setFingerprint(listOf("{{ default }}", context)) }
    }
}

/** Fan-out + rate limit. SDK collection itself is gated by ConsentApplier; calls are no-ops/ local when off. */
class CompositeCrashReporter(private val delegates: List<CrashReporter>) : CrashReporter {
    private val counts = java.util.concurrent.ConcurrentHashMap<String, Int>()
    override fun breadcrumb(category: String, message: String, data: Map<String, String>) =
        delegates.forEach { it.breadcrumb(category, message, data) }
    override fun setKey(key: String, value: String) = delegates.forEach { it.setKey(key, value) }
    override fun setUser(opaqueId: String?) = delegates.forEach { it.setUser(opaqueId) }
    override fun recordNonFatal(t: Throwable, context: String) {
        if (t is kotlinx.coroutines.CancellationException) return
        if (counts.merge(context, 1, Int::plus)!! > 10) return
        delegates.forEach { it.recordNonFatal(t, context) }
    }
}

class ExitInfoReporter(
    private val am: ActivityManager, private val pkg: String,
    private val cursor: DeviceIds, private val crash: CrashReporter,
) {
    class ProcessExit(reason: String) : Exception("process_exit_$reason")

    fun reportSinceLastLaunch() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.R) return
        val last = cursor.lastExitTs
        val exits = am.getHistoricalProcessExitReasons(pkg, 0, 20).filter { it.timestamp > last }
        exits.forEach { e ->
            val reason = when (e.reason) {
                ApplicationExitInfo.REASON_LOW_MEMORY -> "low_memory"
                ApplicationExitInfo.REASON_SIGNALED -> "signaled"
                ApplicationExitInfo.REASON_EXCESSIVE_RESOURCE_USAGE -> "excessive_resource"
                ApplicationExitInfo.REASON_INITIALIZATION_FAILURE -> "init_failure"
                // CRASH / CRASH_NATIVE / ANR already reported by Crashlytics on API 30+
                else -> return@forEach
            }
            val fg = e.importance <= ActivityManager.RunningAppProcessInfo.IMPORTANCE_FOREGROUND
            crash.setKey("exit_last_screen", e.processStateSummary?.decodeToString() ?: "?")
            crash.setKey("exit_foreground", fg.toString())
            crash.recordNonFatal(ProcessExit(reason), "process_exit")
        }
        exits.maxOfOrNull { it.timestamp }?.let { cursor.lastExitTs = it }
    }
}
// Navigation listener: if (SDK_INT >= 30) am.setProcessStateSummary(route.take(120).toByteArray())
```

### A.7 Data cleaner that keeps consent

```kotlin
class AccountDataCleaner @Inject constructor(@ApplicationContext private val ctx: Context) {
    // Device-level files that sign-out must NEVER delete.
    private val keep = setOf("device_consent.preferences_pb", "device_analytics.preferences_pb")

    suspend fun wipeAccount(accountId: String) = withContext(Dispatchers.IO) {
        ctx.deleteDatabase("acct_$accountId.db")
        File(ctx.filesDir, "datastore/acct_$accountId.preferences_pb").delete()
        File(ctx.cacheDir, "acct_$accountId").deleteRecursively()
    }

    suspend fun signOutAll() = withContext(Dispatchers.IO) {
        File(ctx.filesDir, "datastore").listFiles()?.filterNot { it.name in keep }?.forEach { it.delete() }
        ctx.databaseList().filter { it.startsWith("acct_") }.forEach { ctx.deleteDatabase(it) }
        ctx.cacheDir.deleteRecursively()
        // NOT: ctx.filesDir.deleteRecursively() / am.clearApplicationUserData() -> wipes consent
    }
}
```

### A.8 Identity on login, account/portal switch, logout

```kotlin
class TelemetryIdentitySync @Inject constructor(
    private val analytics: Analytics,
    private val crash: CrashReporter,
    private val queue: EventQueue,
) {
    suspend fun onLogin(userId: String, portalId: String?, accountCount: Int) {
        analytics.identify(userId, buildMap { portalId?.let { put("portal_id", it) } })
        crash.setUser(userId); portalId?.let { crash.setKey("portal_id", it) }
        crash.setKey("account_count", accountCount.toString())
    }

    // Called by SessionManager.switchAccount BEFORE tearing down A (see android-multi-account).
    suspend fun onAccountSwitch(toUserId: String, toPortalId: String?, accountCount: Int) {
        queue.flushNow()                       // A's events go out with A's identity (stamped anyway)
        crash.breadcrumb("session", "account_switch")
        onLogin(toUserId, toPortalId, accountCount)
        analytics.track(Events.AccountSwitched(accountCount))
    }

    fun onPortalSwitch(portalId: String, portalCount: Int) {
        analytics.setProperty("portal_id", portalId)
        crash.setKey("portal_id", portalId); crash.breadcrumb("session", "portal_switch")
        analytics.track(Events.PortalSwitched(portalCount))
    }

    suspend fun onLastLogout() {
        queue.flushNow()
        analytics.reset(); crash.setUser(null); crash.setKey("portal_id", "")
        // Consent record deliberately untouched.
    }
}
```

### A.9 Application wiring

```kotlin
@HiltAndroidApp
class App : Application() {
    @Inject lateinit var consentApplier: ConsentApplier
    @Inject lateinit var exitInfo: ExitInfoReporter
    @Inject @AppScope lateinit var appScope: CoroutineScope

    override fun onCreate() {
        super.onCreate()               // Hilt injection happens here
        consentApplier.start(appScope) // SDKs remain disabled (manifest) until first emission says otherwise
        appScope.launch(Dispatchers.IO) { exitInfo.reportSinceLastLaunch() }
        ProcessLifecycleOwner.get().lifecycle.addObserver(FlushOnStopObserver(this))
    }
}
```

### A.10 Unit test: consent survives sign-out-all, no events before decision

```kotlin
@Test fun consent_survives_signOutAll_and_second_login_skips_prompt() = runTest {
    val ctx = ApplicationProvider.getApplicationContext<Context>()
    val store = ConsentStore(ctx)
    store.save(analytics = true, crash = true)
    AccountDataCleaner(ctx).signOutAll()
    assertFalse(store.needsPrompt())
    assertEquals("home", PostLoginRouter(store).destinationAfterLogin())
}

@Test fun undecided_drops_events() {
    val sink = FakeSink()
    val a = ConsentGatedAnalytics(MutableStateFlow(null.effective()), setOf(sink), FakeIds(), { emptyMap() })
    a.track(Events.LoginCompleted(LoginMethod.Password, isNewDevice = true))
    assertTrue(sink.events.isEmpty())
}

@Test fun catalogue_names_valid() {
    AnalyticsEvent::class.sealedSubclasses // or list instances explicitly
    listOf(Events.LoginCompleted(LoginMethod.Google, false), Events.PaywallViewed(PaywallSource.Settings, "x"))
        .forEach { e ->
            assertTrue(e.name.matches(Regex("^[a-z][a-z0-9_]{0,39}$")))
            assertTrue(e.params().size <= 24) // + schema_version
        }
}
```
