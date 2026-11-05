---
name: react-native-crash-analytics
description: Use when implementing, reviewing, testing, or debugging crash reporting, product analytics, and analytics consent in React Native and Expo apps — @react-native-firebase/crashlytics and @react-native-firebase/analytics, @sentry/react-native, Hermes source maps and native symbol upload (dSYMs, R8 mapping, NDK), Expo config plugins and EAS Build / EAS Update source map upload, ErrorUtils global handler and unhandled promise rejections, error boundaries, ANRs / app hangs / OOM, breadcrumbs and custom keys, release health and crash-free gates, typed event catalogue, identity (anonymous → login → account/portal switch → logout), a device-level consent record prompted once after the first login, batching/offline delivery, performance traces, and a vendor-neutral Analytics / CrashReporter facade. Siblings: ios-crash-analytics, android-crash-analytics, flutter-crash-analytics; related: react-native-multi-account, react-native-deep-links, react-native-push-notifications, react-native-in-app-purchase, react-native-list-ux, react-native-network-offline-sync.
---

# React Native Crash Reporting, Analytics & Consent Skill

Guide for building, auditing, and testing **crash reporting and product analytics** in consumer React Native apps — **bare React Native** and **Expo** (prebuild / dev client / EAS Build). Two vendors are shown concretely — **Firebase Crashlytics + Firebase Analytics** (`@react-native-firebase/*`) and **Sentry** (`@sentry/react-native`) — but always **behind a vendor-neutral facade** (`Analytics`, `CrashReporter`) so you can swap vendors, run both, or fake them in tests. Siblings: **ios-crash-analytics**, **android-crash-analytics**, **flutter-crash-analytics**. Related: **react-native-multi-account** (identity on switch), **react-native-deep-links** (attribution), **react-native-in-app-purchase** (purchase funnel), **react-native-push-notifications** (open events), **react-native-list-ux** (frozen frames), **react-native-network-offline-sync** (network traces, offline queue).

An RN crash has **three possible origins**, each needing its own capture path and its own symbols:

1. **JS** (Hermes bytecode) — `ErrorUtils` global handler, unhandled promise rejections, React error boundaries. Needs **Hermes source maps** (bundle + compiled-Hermes map composed) for every build *and every OTA update*.
2. **Native Objective-C/Swift/Kotlin/Java** — native SDK crash handler. Needs **dSYMs** (iOS) and **R8/ProGuard mapping** (Android).
3. **Native C/C++** (Hermes, JSI, Reanimated, Skia, MMKV…) — needs **NDK symbols** (Android) / dSYMs (iOS).

These libraries move fast. **Verify fast-moving APIs against the current docs and the installed versions in `package.json` before writing code.** Known version gates:

- **@react-native-firebase** v22+ exposes the **modular API** (`getCrashlytics()`, `recordError(crashlytics, e)`, `getAnalytics()`, `logEvent(analytics, ...)`); the namespaced `firebase.crashlytics()` / `analytics().logEvent()` style is deprecated and removed in later majors. Expo needs the `@react-native-firebase/app` (+ `/crashlytics`) config plugins and usually `expo-build-properties` `ios.useFrameworks: "static"` — **does not run in Expo Go**.
- **firebase.json** `react-native` block controls defaults *at build time*: `crashlytics_auto_collection_enabled`, `analytics_auto_collection_enabled`, `crashlytics_debug_enabled`, `crashlytics_javascript_exception_handler_chaining_enabled`, `google_analytics_adid_collection_enabled` — verify key names against the current RNFB docs.
- **@sentry/react-native** v6+/v7: Expo plugin `@sentry/react-native/expo`, Metro helper `getSentryExpoConfig` (Expo) / `withSentryConfig` (bare) from `@sentry/react-native/metro`, `Sentry.wrap(App)`, `reactNavigationIntegration()`, app hang / ANR tracking, `sentry-expo-upload-sourcemaps` for EAS Update. The old `sentry-expo` package is deprecated — migrate.
- **Hermes** is the default engine (RN 0.70+); source maps must be **composed** (packager map + Hermes bytecode map). RN's Gradle/Xcode scripts and the Sentry plugin do this; hand-rolled CI often does not.
- **Expo SDK 52+** defaults to the New Architecture; RN 0.82+ cannot opt out. Check vendor SDK compatibility notes.
- **react-native-mmkv** v3 `new MMKV({ id })` vs v4 `createMMKV({ id })` — the consent store below must be a *global* instance either way.
- **expo-tracking-transparency** for ATT (`requestTrackingPermissionsAsync`).

## 0. Mental model

```
 ┌────────────────────────────────── App process ──────────────────────────────────────┐
 │  index.ts (first lines): CrashReporter.init(build tags, consent snapshot)           │
 │     ErrorUtils.setGlobalHandler ─► chain ─► CrashReporter.fatal(js)                 │
 │     unhandled rejections (Hermes tracker) ─► CrashReporter.nonFatal                 │
 │     <ErrorBoundary> per screen tree ─► nonFatal + fallback UI                       │
 │                                                                                     │
 │  ConsentStore (GLOBAL MMKV "device", never per-account, never wiped on logout)      │
 │     { version, analytics, crash, decidedAt } ──► ConsentGate ──┬─► Analytics facade │
 │                                                                └─► CrashReporter    │
 │  Analytics facade (typed catalogue, validation, super props, identity)              │
 │     ├─ FirebaseAnalyticsSink   (collection disabled in firebase.json until opt-in)  │
 │     ├─ BackendSink (own queue: batch, offline, retry, sampling)                     │
 │     └─ FakeSink (tests)                                                             │
 │  CrashReporter facade (breadcrumbs, keys, user id, account/portal tags)             │
 │     ├─ CrashlyticsReporter  (auto collection off at build; enable on consent)       │
 │     └─ SentryReporter       (init gated / beforeSend gate)                          │
 │                                                                                     │
 │  Identity: anonymousId (device) ─► login: setUserId(serverId) ─► switch: re-tag     │
 │            ─► logout: reset user id + new anonymousId (consent untouched!)          │
 └─────────────────────────────────────────────────────────────────────────────────────┘
 Build/CI: EAS Build ─► dSYM + R8 mapping + NDK + Hermes maps ─► Crashlytics / Sentry
           EAS Update ─► export maps ─► Sentry (release + dist = update id)
```

Rule of thumb: **one facade, one consent gate, one place that knows the release identity.** Product code calls `analytics.track(Events.cart_item_added({...}))` and `crash.breadcrumb(...)`; it never imports a vendor SDK.

## 1. First, establish context

Find out from the codebase, or ask:

- **Tools**: Crashlytics, Sentry, both? Firebase Analytics, Amplitude, Mixpanel, Segment, PostHog, own backend? Who reads the data (product, growth, support)?
- **Workflow**: Expo managed + prebuild + EAS Build? Bare RN with Fastlane/Bitrise/GitHub Actions? **EAS Update / CodePush** OTA? (OTA multiplies the number of JS releases needing source maps.)
- **Consent / legal**: target markets (EU/UK GDPR + ePrivacy, California CPRA, Brazil LGPD)? Legal basis for crash reporting (legitimate interest vs consent)? Is there a consent-management platform (CMP) already? Children's audience (COPPA / Families policy — disables most analytics)?
- **ATT (iOS)**: does anything track across apps/sites (IDFA, ad attribution SDKs, Meta/AppsFlyer)? If not, no ATT prompt is needed.
- **Release channels**: dev / preview (internal, TestFlight, Play internal) / production; are these separate Firebase projects and Sentry environments?
- **Backend pipeline**: do events go to a warehouse (BigQuery export, Segment → Snowflake)? Who owns the event schema and its versioning?
- **Accounts**: single account, or multi-account/portal (**react-native-multi-account**)? What is the stable server user id (never email)?
- **Release health**: is there a crash-free gate for staged rollouts (Play staged rollout %, App Store phased release, EAS Update rollout %)?
- **Volume & cost**: DAU, events per session, Sentry quota — determines sampling.

## 2. Crash reporting setup

- [ ] **Init as early as possible**: first import in `index.ts` / root `_layout.tsx` (before navigation, state, fonts). Native SDKs start before JS (Crashlytics via `FirebaseApp.configure()` in the plugin; Sentry native via `Sentry.init` with `enableNative: true` or `autoInitializeNativeSdk`), so native crashes during JS bundle load are captured if collection is enabled.
- [ ] **Release identity** on every report: `release = <bundleId>@<version>+<buildNumber>`, `dist = <buildNumber>` (Sentry); Crashlytics derives version/build from the native app. Add custom keys/tags: `js_bundle_id` (EAS Update `Updates.updateId ?? 'embedded'`), `runtime_version`, `channel`, `environment`, `hermes: !!global.HermesInternal`, `new_arch`.
- [ ] **Environments separated**: separate Firebase projects per environment (different `GoogleService-Info.plist` / `google-services.json` selected via `app.config.ts` + EAS env), Sentry `environment: 'development' | 'preview' | 'production'`. Never let dev crashes pollute production crash-free rates.
- [ ] **Debug builds**: don't send by default (`__DEV__` → disabled, or Sentry `enabled: !__DEV__`); allow an override for testing symbol upload on release builds locally.
- [ ] **Don't double-handle**: if both Crashlytics and Sentry are installed, only one native signal handler reliably captures a native crash (the last one installed wins, chained handlers vary). Choose a **primary native crash reporter**; send JS errors to both via the facade if needed.
- [ ] **Expo config plugins** (`app.config.ts`): `@react-native-firebase/app`, `@react-native-firebase/crashlytics`, `expo-build-properties` (`ios.useFrameworks: 'static'`), `@sentry/react-native/expo` (`organization`, `project`, `url`). Run `npx expo prebuild --clean` after changes; CNG means native edits in `ios/`/`android/` are lost.
- [ ] **Secrets**: `SENTRY_AUTH_TOKEN` as an EAS secret / CI secret, never in `app.config.ts` or the bundle. The DSN is not secret but is env-specific.

## 3. Symbolication (Hermes source maps + native)

Without symbols, crash reports are useless (`index.android.bundle:1:284733`, `0x1048a3f20`). Treat symbol upload as part of the build — **fail the release job if upload fails**.

### 3.1 JS / Hermes
- [ ] **Sentry**: Metro config via `getSentryExpoConfig(__dirname)` (Expo) or `withSentryConfig(getDefaultConfig(...))` (bare) — adds **Debug IDs** to bundle and map so matching no longer depends on exact `release`/`dist` strings. The Expo plugin / `sentry.gradle` / Xcode "Bundle React Native code and images" wrapper (`sentry-xcode.sh`) uploads composed Hermes maps on release builds (EAS Build included).
- [ ] **EAS Update**: after `eas update`, run `npx sentry-expo-upload-sourcemaps dist` (uses `SENTRY_AUTH_TOKEN`, org/project from plugin). Without it, every OTA JS crash is unsymbolicated. Tag `dist`/`js_bundle_id` with the update id.
- [ ] **CodePush / custom OTA**: generate maps with `npx react-native bundle --sourcemap-output`, compile with `hermesc -output-source-map`, compose with `react-native/scripts/compose-source-maps.js`, upload with `sentry-cli sourcemaps upload` using the same release/dist (or debug IDs).
- [ ] **Crashlytics and JS**: Crashlytics records JS errors (`recordError`) as non-fatals with the JS stack as text; it **does not symbolicate Hermes bytecode stacks with source maps**. If JS crash readability matters, use Sentry for JS (or send composed-map-resolved stacks from your own backend). Verify current Crashlytics behaviour before promising otherwise.
- [ ] Release builds: `bundleInDebug`/dev-mode bundles not shipped; source maps **not** shipped inside the app (they leak source).

### 3.2 iOS native
- [ ] **dSYMs**: `DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` for Release. Crashlytics: the RNFB plugin adds the `upload-symbols` run script (`[CP-User] [RNFB] Crashlytics Configuration`); Sentry: `sentry-xcode-debug-files.sh` build phase via plugin. On EAS Build both run inside the build. Bitcode is gone (Xcode 14+), so App Store dSYMs equal the build's dSYMs.
- [ ] Verify with Crashlytics "Missing dSYMs" banner / `sentry-cli debug-files check`.

### 3.3 Android native
- [ ] **R8 mapping**: `minifyEnabled true` in release (`expo-build-properties` `android.enableProguardInReleaseBuilds` / `enableMinifyInReleaseBuilds` — verify name). Crashlytics Gradle plugin uploads `mapping.txt` (`firebaseCrashlytics { mappingFileUploadEnabled true }`); Sentry Gradle plugin `autoUploadProguardMapping`.
- [ ] **NDK**: `firebaseCrashlytics { nativeSymbolUploadEnabled true }` + `uploadCrashlyticsSymbolFileRelease` task; Sentry `uploadNativeSymbols = true`. Hermes/Reanimated/Skia crashes in `libhermes.so`/`libreanimated.so` are common — without NDK symbols they are opaque.
- [ ] Play Console: also upload `mapping.txt` and native debug symbols (AAB includes them if `debugSymbolLevel 'FULL'`), for Android vitals.
- [ ] In Expo, Gradle-block changes need a **local config plugin** (`withAppBuildGradle`) — never hand-edit `android/` in CNG.

## 4. Capture

- [ ] **JS fatal**: `ErrorUtils.setGlobalHandler((error, isFatal) => …)` — **chain** the previous handler (`ErrorUtils.getGlobalHandler()`), report with `isFatal`, and await a short flush before handing over (RN's default handler shows the red box in dev / terminates in release). Crashlytics and Sentry each install one; if you add your own, install it *after* vendor init and chain.
- [ ] **Unhandled promise rejections**: in release, RN only logs them. Hook Hermes: `HermesInternal?.enablePromiseRejectionTracker?.({ allRejections: true, onUnhandled: (id, err) => crash.nonFatal(err, { kind: 'unhandled_rejection' }) })`; fall back to `promise/setimmediate/rejection-tracking` on JSC. Sentry installs this by default (`onunhandledrejection` integration) — don't double-report.
- [ ] **React error boundaries**: one at root (fallback "Something went wrong — Restart" using `expo-updates` `reloadAsync()` or `DevSettings.reload` in dev) and one per screen/feature, so one broken widget doesn't kill the app. Report `componentStack`. Error boundaries **don't** catch event handlers, async code, or errors in native modules.
- [ ] **Non-fatals**: caught errors that matter (payment failure, decoding error, unexpected server state) → `crash.nonFatal(err, context)`. Wrap non-`Error` throws (`throw 'x'`, `{code}` objects) into `Error` so stacks exist. Don't report expected outcomes (401, offline, user cancel).
- [ ] **ANRs / app hangs**: Crashlytics captures Android ANRs natively; Sentry `enableAppHangTracking` (iOS, threshold `appHangTimeoutInterval`) and ANR integration (Android, `ApplicationExitInfo` on API 30+). JS-thread stalls are not ANRs — measure them as frozen frames / long tasks (§12).
- [ ] **OOM / watchdog terminations**: Sentry `enableWatchdogTerminationTracking` (iOS); Android `ApplicationExitInfo.REASON_LOW_MEMORY` via Sentry/Crashlytics or a small native module; iOS **MetricKit** (`MXDiagnosticPayload`) through Sentry's MetricKit integration. Common RN OOM sources: large images in lists (**react-native-list-ux**), base64 blobs over the bridge, unbounded caches.
- [ ] **Native module errors**: rejected native promises arrive as JS errors with `code` — include `code` in context.
- [ ] **Startup crash loops**: if the app crashed during the last launch before "app ready", Sentry/Crashlytics may not send; Crashlytics sends synchronously on next launch if crashed early — verify; Sentry: `Sentry.flush()` in boundary fallback; consider an EAS Update rollback if the crash comes from an OTA bundle (`Updates.updateId`).
- [ ] **Dev noise**: filter `LogBox`/Metro errors, `Network request failed` (map to network errors in the client instead), and known third-party noise via `ignoreErrors` / `beforeSend`.

## 5. Context: breadcrumbs, keys, user

- [ ] **Breadcrumbs**: navigation (route *names*, not params), app state changes (`AppState`), network (method, path template, status, duration — **no query strings, bodies, or headers**), user actions (`button_tapped:checkout`), consent changes, account/portal switch. Crashlytics `log(crashlytics, msg)`; Sentry `addBreadcrumb`.
- [ ] **Custom keys / tags**: `account_index`, `account_count`, `portal_id` (opaque id), `role`, `subscription_tier`, `js_bundle_id`, `feature_flags` hash, `locale`, `connectivity`. Crashlytics `setAttribute(s)`; Sentry `setTag` (indexed, filterable) vs `setContext` (unindexed detail).
- [ ] **User id**: opaque server user id (or hash) — `setUserId(crashlytics, id)`, `Sentry.setUser({ id })`. **No email, name, phone, IP** (`sendDefaultPii: false`). On account/portal switch update id + tags (**react-native-multi-account** §14).
- [ ] **PII scrubbing**: `beforeSend`/`beforeBreadcrumb` scrub tokens, emails, JWT-shaped strings, card numbers; disable request body capture in network integrations; strip route params logged by navigation instrumentation.
- [ ] **Screenshots / view hierarchy / session replay** (Sentry): off by default for consumer apps with private data; if enabled, mask all text and images and gate behind crash consent.

## 6. Grouping & triage

- [ ] **Fingerprints**: group by root cause, not message. Normalize dynamic messages (`"User 123 not found"` → `"User {id} not found"`); for wrapped errors set Sentry `fingerprint: ['{{ default }}', err.code]`. Crashlytics groups by stack; use `jsErrorName` in `recordError` to give JS errors a stable name.
- [ ] **Release health**: crash-free **users** and **sessions** per release (Sentry `enableAutoSessionTracking`; Crashlytics dashboard). Track per `js_bundle_id` too — an OTA can regress a stable binary.
- [ ] **Gates**: block widening staged rollout / EAS Update rollout below threshold (e.g. crash-free sessions ≥ 99.8%, users ≥ 99.5%); `eas update:rollback` / `eas update --rollout-percentage` playbook documented.
- [ ] **Alerts**: new issue in latest release, regression (resolved issue reappearing), velocity spike; route to the owning team. Mark issues "resolved in next release" so regressions alert.
- [ ] **Triage loop**: weekly top-crashers review; each with owner, repro, fix version. Link issues to commits (Sentry `setCommits` via CI).

## 7. Event design

- [ ] **Naming**: `object_action`, snake_case, past tense for completed actions: `cart_item_added`, `checkout_started`, `purchase_completed`, `account_switched`. Firebase limits: event names ≤ 40 chars, ≤ 25 params, param names ≤ 40, string values ≤ 100 chars; reserved names/prefixes (`firebase_`, `google_`, `ga_`, `screen_view` as auto event) — design within them even if you're not on Firebase.
- [ ] **Typed catalogue**: one TS module defines every event, its params type, and its schema version; product code can only call `analytics.track(Events.x(params))`. Unknown events don't compile.
- [ ] **Required props** (super properties added by the facade): `schema_version`, `app_version`, `build`, `js_bundle_id`, `platform`, `os_version`, `locale`, `session_id`, `portal_id?`, `account_count`. Don't repeat them in each event.
- [ ] **Versioning**: changing a param's meaning → new event name or bump `v`; never silently repurpose. Deprecate with a date; keep a changelog next to the catalogue.
- [ ] **Screen tracking**: one `screen_view` per focused route from the navigation container (`onStateChange` / Expo Router `usePathname`), route *name* only (`/product/[id]`, not `/product/123?ref=email`). Dedupe consecutive identical screens.
- [ ] **No PII in params**: no emails, free text, search queries (unless reviewed), addresses. Ids as opaque strings; amounts as numbers + `currency`.
- [ ] **Validate in dev**: the facade asserts param types/limits under `__DEV__` and logs every event to the console; Firebase DebugView (`-FIRDebugEnabled` / `adb shell setprop debug.firebase.analytics.app <pkg>`).

## 8. Identity

- [ ] **Anonymous id** pre-login: random UUID in global (device) storage, regenerated on full logout. Used for your backend sink only after consent; vendors keep their own app-instance id.
- [ ] **Login**: `setUserId(serverUserId)` on analytics and crash; backend sink sends an `identify`/alias linking anonymousId → userId (if your pipeline supports it).
- [ ] **Account / portal switch**: flush the queue under the old identity **first**, then set the new user id and `portal_id` super prop / tag; emit `account_switched` after (**react-native-multi-account**).
- [ ] **Logout (last account)**: flush, `setUserId(null)` / `Sentry.setUser(null)`, new anonymousId, optionally `resetAnalyticsData(analytics)` (new app-instance id). **Consent record is not touched.**
- [ ] **Never** use email/phone as user id; never send the user id as an event param too.

## 9. CONSENT — required behaviour

This is the section reviewers must check line by line. The rules:

1. **When to ask**: the **first time any user logs in on this device**. Show the consent prompt immediately after the first successful login, **before navigating to home**. Not on app launch, not on the login screen.
2. **Where it's stored**: **device level** — a global MMKV/AsyncStorage key (`device.consent`), **not** inside a per-account MMKV instance, not keyed by user id, not server-only. It survives logout and re-login and applies to **every account on the device**.
3. **Do not ask again** on subsequent logins, adding a second account, account switch, portal switch, or app updates — unless the **policy version** is bumped (then re-prompt once).
4. **Never wipe it** on logout, "sign out of all accounts", account removal, or account data wipe. ⚠️ **Common bug**: the logout code calls `storage.clearAll()` / `AsyncStorage.clear()` / deletes "everything" — consent goes with it and the user is re-prompted (or worse, defaults to on). Wipes must target account-scoped stores only.
5. **Before the first answer** (pre-login, during login, while the prompt is open): **no optional analytics collected, and nothing queued to send later**. The facade **drops** (not buffers) events while consent is `unknown`. Vendor SDKs are **disabled at build/config level**:
   - `firebase.json`: `"analytics_auto_collection_enabled": false`, `"crashlytics_auto_collection_enabled": false`, `"google_analytics_adid_collection_enabled": false` (plus `google_analytics_default_allow_ad_personalization_signals: false` if offered).
   - Sentry: don't call `Sentry.init` until crash consent is known *or* init with a `beforeSend`/`beforeSendTransaction` gate that returns `null` and `enableAutoSessionTracking` off until granted (see §9 crash below).
   - Firebase Consent Mode: `setConsent(analytics, { analytics_storage: false, ad_storage: false, ad_user_data: false, ad_personalization: false })` at startup, flipped on opt-in.
6. **Crash reporting**: a **separate toggle** (`crash`). Default depends on your legal basis — if crash reporting relies on legitimate interest, it can default **on** and the prompt shows it pre-enabled with explanation; if it requires consent, default **off** until answered. Whatever the choice: apply it to **both** Crashlytics (`setCrashlyticsCollectionEnabled`) and Sentry (gate), and document the decision. Crashes that happen **before** the decision are dropped under consent-based basis (Crashlytics keeps unsent reports on disk when collection is off — call `deleteUnsentReports` on opt-out, `sendUnsentReports` on opt-in if your legal basis allows; verify current behaviour).
7. **Settings toggle**: Settings → Privacy shows both toggles; changes apply **immediately**: enable/disable SDK collection, on opt-out **delete queued events** (own queue + `deleteUnsentReports`), `resetAnalyticsData`, and optionally call your backend to request deletion (GDPR Art. 17) with the user/anonymous id.
8. **Record shape**: `{ version: number, analytics: boolean, crash: boolean, decidedAt: string (ISO) }`. Optionally send a consent audit event to your backend (that send is under legal obligation, not analytics).
9. **Policy version bump**: `CURRENT_CONSENT_VERSION` constant; if `record.version < CURRENT` → re-prompt once at the next login *or* next foreground for a logged-in user (product decision, document it); until answered, the previous choice stays in effect if it is still valid under the new policy, otherwise treat as unknown.
10. **Reinstall**: clears MMKV/AsyncStorage on both platforms → re-prompt (desired). **Don't** store consent in Keychain (`react-native-keychain` / `expo-secure-store` items **survive iOS reinstall**) unless you deliberately want that. **Android Auto Backup** may restore MMKV/AsyncStorage files onto a new device — decide: either exclude the consent file (`android:fullBackupContent` / `dataExtractionRules` via config plugin) so new devices re-prompt, or include it consciously. Document it.
11. **ATT (iOS)**: separate, only if you track across apps/websites (IDFA). Sequence: your consent prompt **first**; only if `analytics === true` *and* you need IDFA, call `requestTrackingPermissionsAsync()` (system prompt, once per install). ATT denied ≠ analytics off — first-party analytics may continue per your consent. Never pre-prompt in a way that pressures the user. No ATT prompt if you don't track.
12. **Multiple accounts**: one device record covers all accounts. If a product insists on per-account consent, that is a different design — the brief here is device-level; flag the conflict.

- [ ] Consent store is global storage; grep for its key inside account wipe code paths.
- [ ] Prompt is shown from the **post-login hook**, before `router.replace('/(home)')`, and only if `needsPrompt()`.
- [ ] Facade drops events when consent is `unknown` or `false`; no "pending" buffer.
- [ ] Build-time defaults off in `firebase.json`; verified by checking no network calls to `app-measurement.com` / `firebaselogging` / `sentry.io` before opting in (proxy).
- [ ] Settings toggles apply immediately, both SDKs.
- [ ] Prompt UI: equal-weight "Allow" / "Don't allow" buttons, link to privacy policy, plain-language, accessible.

## 10. Delivery

- [ ] **Vendor SDKs batch themselves** (Firebase uploads ~hourly or on background; Sentry sends envelopes and caches offline). Don't wrap them in your own queue.
- [ ] **Own backend sink**: in-memory buffer → persisted queue (global MMKV / SQLite, capped, e.g. 1,000 events / 7 days), batch of 20–50 or every 30 s, flush on `AppState` → `background`, exponential backoff + jitter, drop on 4xx schema errors, `event_id` UUID per event for server dedupe (see **react-native-network-offline-sync** for retry/outbox patterns).
- [ ] **Offline**: queue persists; send when connectivity returns; timestamps are client time + `sent_at` so the server can correct skew.
- [ ] **Sampling**: sample high-volume, low-value events (scroll, impressions) by deterministic hash of session id; never sample purchase/funnel events. Sentry `tracesSampleRate`/`tracesSampler`, `sampleRate` for errors usually 1.0.
- [ ] **Background flush**: on iOS you get a few seconds after background; use it. Don't schedule background tasks just for analytics.
- [ ] **Opt-out** deletes the persisted queue.

## 11. Funnels & key metrics

- [ ] **Activation**: `app_first_opened` (after consent → first post-consent event), `signup_completed`, `onboarding_completed`, first core action (`first_post_created`). Define the activation event explicitly.
- [ ] **Retention**: D1/D7/D30 cohorts from `session_started` (or vendor `user_engagement`); define "active" (foreground ≥ N s, or core action).
- [ ] **Purchase funnel** (**react-native-in-app-purchase** / **react-native-revenuecat-iap**): `paywall_viewed {placement}` → `purchase_started {product_id}` → `purchase_completed {product_id, price, currency, transaction_id}` / `purchase_failed {reason}` / `purchase_cancelled`. Revenue truth comes from the server/store, not client events — client events are for funnel shape.
- [ ] **Deep-link attribution** (**react-native-deep-links**): `deep_link_opened {source, campaign, route_name, cold_start}`; carry `utm_*` into the session super props; push opens (`notification_opened {category}`) from **react-native-push-notifications**.
- [ ] **Account metrics**: `account_added`, `account_switched`, `portal_switched` with counts, not identities.
- [ ] Every funnel has a dashboard and an owner; events without a consumer are deleted.

## 12. Performance monitoring

- [ ] **App start**: Sentry app start (cold/warm) measured from native process start to first frame/`Sentry.wrap` root mount; mark "interactive" yourself when the home data is shown. Firebase Performance (`@react-native-firebase/perf`) `_app_start` trace as alternative.
- [ ] **Screen render / TTID-TTFD**: `reactNavigationIntegration({ enableTimeToInitialDisplay: true })`, `<TimeToFullDisplay record={loaded} />` (Sentry) — verify API.
- [ ] **Network traces**: Sentry fetch/XHR spans with `tracePropagationTargets` limited to **your** API domains (don't leak `sentry-trace` headers to third parties); Firebase Perf HTTP metrics. Path templates, not raw URLs.
- [ ] **Frozen/slow frames**: Sentry native frames tracking; JS-thread long tasks; list jank (**react-native-list-ux**).
- [ ] Performance data is under the **crash** (or a separate performance) consent toggle — decide and document.
- [ ] Sample traces (`tracesSampleRate: 0.05–0.2` in prod).

## 13. Facade / abstraction layer

- [ ] `Analytics` interface: `track(event)`, `screen(name)`, `identify(userId | null)`, `setSuperProps(partial)`, `setEnabled(bool)`, `flush()`, `reset()`.
- [ ] `CrashReporter` interface: `init()`, `setEnabled(bool)`, `nonFatal(err, ctx?)`, `fatal(err)`, `breadcrumb(msg, data?)`, `setUser(id | null)`, `setKeys(record)`, `flush()`.
- [ ] **Composite** implementations fan out to several sinks; **ConsentGate** wraps them; **Fake** implementations record calls for tests.
- [ ] Vendor modules are the only files that import `@react-native-firebase/*` / `@sentry/react-native` (ESLint `no-restricted-imports` elsewhere).
- [ ] Facade is safe to call before init (no-ops) and from background tasks.

## 14. Testing

### 14.1 Tools
1. **Jest** with `jest.mock('@react-native-firebase/analytics')`, `jest.mock('@sentry/react-native')`, and the **FakeAnalytics / FakeCrashReporter** from the appendix; MMKV v3 Jest mock (in-memory) for the ConsentStore.
2. **React Native Testing Library** for the consent prompt and Settings toggles; assert navigation to home happens only after a choice.
3. **Release build crash tests**: a hidden debug menu (preview builds only) with "JS crash", "unhandled rejection", "native crash" (`crash(getCrashlytics())` / `Sentry.nativeCrash()`), "ANR" (native busy-wait). Verify each arrives **symbolicated** (JS file:line, native function names) for: EAS Build binary, then an EAS Update on top.
4. **Network proxy** (Proxyman/Charles/mitmproxy): confirm zero analytics/crash traffic before consent and after opt-out.
5. **Firebase DebugView** / Sentry "Debug" mode for event schema checks; a contract test that the catalogue matches the warehouse schema.
6. **E2E** (Maestro/Detox): first login → consent prompt → home; logout → login → no prompt.

### 14.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| C1 | Fresh install, launch, before login | No consent prompt; no analytics/crash network traffic; facade drops events |
| C2 | First successful login | Consent prompt shown before home; home only after a choice |
| C3 | Kill app while prompt open, relaunch (still logged in) | Prompt shown again before home; nothing sent meanwhile |
| C4 | Accept analytics | SDK collection enabled; events flow; record `{version, analytics:true, crash, decidedAt}` |
| C5 | Decline analytics | No events ever sent; Firebase collection off; no ATT prompt |
| C6 | Logout, login same account | **No** prompt; previous choice applied |
| C7 | Logout, login different user | **No** prompt; device choice applies |
| C8 | Add second account (multi-account) | **No** prompt |
| C9 | Switch account / portal | **No** prompt; user id and portal tag updated; queue flushed before switch |
| C10 | "Sign out of all accounts" / account data wipe | Consent record still present |
| C11 | Opt out in Settings | Immediate: SDK disabled, queued events deleted, unsent crash reports deleted |
| C12 | Opt back in | Immediate enable; no backfill of events from opted-out period |
| C13 | Reinstall app | Re-prompt after first login (MMKV cleared; nothing in Keychain) |
| C14 | Policy version bump in update | Re-prompt once; then never again |
| C15 | Android backup restore to new device | Behaves per documented backup decision |
| C16 | ATT (iOS, tracking SDKs present) | After analytics opt-in only; denial doesn't disable first-party analytics |
| C17 | Crash toggle off | No Crashlytics/Sentry reports; analytics unaffected |
| R1 | JS crash in release (EAS Build) | Fatal reported, symbolicated to TS file:line |
| R2 | JS crash in an EAS Update bundle | Symbolicated; tagged with update id |
| R3 | Unhandled promise rejection | One non-fatal (not two) |
| R4 | Error in a screen component | Boundary fallback; non-fatal with componentStack; app usable |
| R5 | Native iOS crash | Symbolicated via dSYM |
| R6 | Native Android crash in Kotlin and in `.so` | Deobfuscated (R8) and NDK-symbolicated |
| R7 | ANR / app hang | Reported with main-thread stack |
| R8 | OOM | Watchdog termination / exit-info reported next launch |
| R9 | Crash before consent decided | Dropped (or sent per legal basis) — matches documented policy |
| R10 | Offline crash and events | Sent on next online launch; events within queue caps |
| E1 | Event with wrong param type | Compile error; dev assertion at runtime for dynamic values |
| E2 | Same screen focused twice | One `screen_view` |
| E3 | Logout | User id cleared; new anonymous id; consent untouched |

### 14.3 Example tests

```ts
it('prompts only on the first login on the device', async () => {
  const consent = new ConsentStore(createTestStorage());
  expect(await postLoginRoute(consent)).toBe('/consent');
  consent.save({ analytics: true, crash: true });
  await wipeAllAccounts();                       // logout of everything
  expect(await postLoginRoute(consent)).toBe('/(home)');
});

it('drops events before consent and after opt-out', () => {
  const sink = new FakeAnalytics();
  const consent = new ConsentStore(createTestStorage());
  const a = new ConsentGatedAnalytics(sink, consent);
  a.track(Events.cart_item_added({ product_id: 'p1', quantity: 1, price: 9.99, currency: 'EUR' }));
  expect(sink.events).toHaveLength(0);
  consent.save({ analytics: true, crash: true });
  a.track(Events.checkout_started({ item_count: 1, value: 9.99, currency: 'EUR' }));
  expect(sink.events).toHaveLength(1);
});

it('re-prompts once after a policy version bump', () => {
  const s = createTestStorage();
  new ConsentStore(s, 1).save({ analytics: false, crash: true });
  const v2 = new ConsentStore(s, 2);
  expect(v2.needsPrompt()).toBe(true);
  v2.save({ analytics: false, crash: true });
  expect(v2.needsPrompt()).toBe(false);
});
```

## 15. Common bugs to look for in review

- Consent stored in the per-account MMKV instance or keyed by user id → second account / re-login prompts again
- Logout calls `AsyncStorage.clear()` / `storage.clearAll()` on the global instance → consent wiped, user re-prompted (or default-on)
- Consent stored in Keychain/SecureStore → survives iOS reinstall unexpectedly
- Prompt shown on first launch (pre-login) or after navigation to home (events already sent from home's mount)
- Facade buffers events while consent is unknown and sends them after opt-in → collection before consent
- `firebase.json` missing → Firebase Analytics auto-collects `first_open`/`session_start` before consent
- Opt-out sets a JS flag but never calls `setAnalyticsCollectionEnabled(false)` / `setCrashlyticsCollectionEnabled(false)`; Sentry still sending sessions
- Crash toggle applied to Crashlytics but not to Sentry (or vice versa)
- ATT prompt on first launch "because the SDK said so", or ATT denial used to disable crash reporting
- `ErrorUtils.setGlobalHandler` replaced without chaining → red box gone in dev, vendor handler bypassed, app doesn't terminate on fatal
- Unhandled rejections not tracked in release → silent failures; or tracked twice (custom tracker + Sentry)
- Hermes source maps not composed, or not uploaded for EAS Update → `index.android.bundle:1:NNNNN` stacks
- Source maps shipped inside the app bundle
- `release`/`dist` mismatch between `Sentry.init` and upload (pre-debug-ID setups)
- `minifyEnabled` on but mapping not uploaded; NDK symbols missing for Hermes/Reanimated crashes
- Native edits in `ios/`/`android/` in a CNG Expo project → lost at next prebuild; symbols upload silently disappears
- Both Crashlytics and Sentry installing native crash handlers → one loses native crashes
- Email used as `setUserId`; route params with emails in breadcrumbs; request bodies in breadcrumbs
- Identity switched before flushing → events of account A attributed to B
- `screen_view` logged with full URLs/ids → high-cardinality, PII
- Event names with spaces/camelCase/ > 40 chars; params > 25 → silently dropped by Firebase
- Dev/preview builds reporting to the production project → crash-free rate garbage
- `tracePropagationTargets` default matching all URLs → trace headers sent to third-party APIs (CORS/signature failures)

## 16. Output format

When asked to **review/audit**:
1. **Summary**: stack (Expo/bare, RN, Hermes, New Arch, OTA), vendors, environments, consent model as found
2. **Checklist results**: sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check, with file:line — **§9 consent first and in full**
3. **Critical issues first**: collection before consent, consent wiped on logout, PII in reports, missing symbol upload (incl. OTA), unchained global handler — each with fix code
4. **Testing gaps**: C/R/E scenarios not covered, plus suggested tests

When asked to **implement**: confirm versions (RNFB modular API, Sentry major, Expo SDK, MMKV v3/v4), the legal basis for crash reporting, and whether OTA is used. Start from the Appendix: consent store + gate first, then facades, then vendor sinks, then build/CI symbol upload, then the event catalogue. List what to verify by hand: proxy shows no traffic pre-consent, symbolicated JS crash from an EAS Update, native crash on both platforms, logout → login shows no prompt.

When asked to **debug**: identify the origin (JS / native / C++) and whether symbols exist for *that exact* build or update id; for consent bugs, trace where the record is read, written, and deleted, and which storage instance holds it.

---

## Appendix — Reference implementation (TypeScript, Expo SDK 53+ / bare RN 0.79+)

Assumes `@react-native-firebase/{app,analytics,crashlytics}` v22+ (modular API), `@sentry/react-native` v6+, `react-native-mmkv` v3 (swap to `createMMKV` for v4), Expo Router. Use one native crash reporter in production; both are shown so you can pick.

### A.1 Build configuration

```ts
// app.config.ts
import type { ExpoConfig } from 'expo/config';
const env = (process.env.APP_ENV ?? 'development') as 'development' | 'preview' | 'production';

export default (): ExpoConfig => ({
  name: 'Shop', slug: 'shop', version: '3.4.0',
  ios: { bundleIdentifier: 'com.example.shop', googleServicesFile: `./firebase/${env}/GoogleService-Info.plist` },
  android: { package: 'com.example.shop', googleServicesFile: `./firebase/${env}/google-services.json` },
  plugins: [
    '@react-native-firebase/app',
    '@react-native-firebase/crashlytics',
    ['expo-build-properties', { ios: { useFrameworks: 'static' }, android: { enableProguardInReleaseBuilds: true } }],
    ['@sentry/react-native/expo', { organization: 'example', project: 'shop-mobile', url: 'https://sentry.io/' }],
    'expo-tracking-transparency',
    './plugins/withBackupRules',            // excludes device consent from Auto Backup (decision §9.10)
  ],
  extra: { env, sentryDsn: process.env.SENTRY_DSN },
});
```

```jsonc
// firebase.json — collection OFF until consent (build-time default)
{
  "react-native": {
    "analytics_auto_collection_enabled": false,
    "google_analytics_adid_collection_enabled": false,
    "crashlytics_auto_collection_enabled": false,
    "crashlytics_debug_enabled": false,
    "crashlytics_javascript_exception_handler_chaining_enabled": true
  }
}
```

```js
// metro.config.js — debug IDs for Hermes maps
const { getSentryExpoConfig } = require('@sentry/react-native/metro');
module.exports = getSentryExpoConfig(__dirname);
```

```yaml
# .eas/workflows or CI step after OTA publish
#   eas update --channel production --message "$MSG"
#   npx sentry-expo-upload-sourcemaps dist      # SENTRY_AUTH_TOKEN from EAS secrets
```

### A.2 Typed event catalogue

```ts
// analytics/events.ts
export const SCHEMA_VERSION = 4;
type Currency = 'EUR' | 'USD' | 'GBP';
export type AnalyticsEvent<N extends string = string, P = Record<string, string | number | boolean>> = {
  name: N; params: P; v: number;
};
const ev = <N extends string, P>(name: N, v = 1) => (params: P): AnalyticsEvent<N, P> => ({ name, params, v });

export const Events = {
  signup_completed: ev<'signup_completed', { method: 'email' | 'apple' | 'google' }>('signup_completed'),
  onboarding_completed: ev<'onboarding_completed', { steps: number }>('onboarding_completed'),
  product_viewed: ev<'product_viewed', { product_id: string; source: 'search' | 'feed' | 'deep_link' }>('product_viewed'),
  cart_item_added: ev<'cart_item_added', { product_id: string; quantity: number; price: number; currency: Currency }>('cart_item_added'),
  checkout_started: ev<'checkout_started', { item_count: number; value: number; currency: Currency }>('checkout_started'),
  paywall_viewed: ev<'paywall_viewed', { placement: string }>('paywall_viewed'),
  purchase_started: ev<'purchase_started', { product_id: string }>('purchase_started'),
  purchase_completed: ev<'purchase_completed', { product_id: string; value: number; currency: Currency; transaction_id: string }>('purchase_completed', 2),
  purchase_failed: ev<'purchase_failed', { product_id: string; reason: 'cancelled' | 'payment' | 'network' | 'unknown' }>('purchase_failed'),
  deep_link_opened: ev<'deep_link_opened', { route_name: string; source?: string; campaign?: string; cold_start: boolean }>('deep_link_opened'),
  notification_opened: ev<'notification_opened', { category: string }>('notification_opened'),
  account_added: ev<'account_added', { account_count: number }>('account_added'),
  account_switched: ev<'account_switched', { account_count: number }>('account_switched'),
  portal_switched: ev<'portal_switched', { portal_count: number }>('portal_switched'),
} as const;

const NAME_RE = /^[a-z][a-z0-9_]{0,39}$/;
export function assertValid(e: AnalyticsEvent): void {
  if (!__DEV__) return;
  if (!NAME_RE.test(e.name)) throw new Error(`Bad event name ${e.name}`);
  const keys = Object.keys(e.params);
  if (keys.length > 25) throw new Error(`${e.name}: > 25 params`);
  for (const k of keys) {
    const val = (e.params as Record<string, unknown>)[k];
    if (!NAME_RE.test(k)) throw new Error(`${e.name}.${k}: bad param name`);
    if (typeof val === 'string' && (val.length > 100 || /@/.test(val))) throw new Error(`${e.name}.${k}: too long or looks like PII`);
  }
}
```

### A.3 Device-level ConsentStore

```ts
// privacy/consentStore.ts
import { MMKV } from 'react-native-mmkv';

export const CURRENT_CONSENT_VERSION = 2;
export type ConsentRecord = { version: number; analytics: boolean; crash: boolean; decidedAt: string };
export type ConsentState = 'unknown' | ConsentRecord;
export interface KV { getString(k: string): string | undefined; set(k: string, v: string): void; delete(k: string): void }

/** GLOBAL device storage. NOT the per-account MMKV instance. Never cleared by logout/wipe. */
export const deviceStorage: KV = new MMKV({ id: 'device.global' });
const KEY = 'device.consent';

type Listener = (s: ConsentState) => void;

export class ConsentStore {
  private listeners = new Set<Listener>();
  constructor(private kv: KV = deviceStorage, private currentVersion = CURRENT_CONSENT_VERSION) {}

  read(): ConsentState {
    try {
      const raw = this.kv.getString(KEY);
      if (!raw) return 'unknown';
      const r = JSON.parse(raw) as ConsentRecord;
      return typeof r.analytics === 'boolean' && typeof r.crash === 'boolean' ? r : 'unknown';
    } catch { return 'unknown'; }
  }

  /** True on first login on this device, or after a policy version bump. */
  needsPrompt(): boolean {
    const s = this.read();
    return s === 'unknown' || s.version < this.currentVersion;
  }

  /** Effective permissions. Unknown → nothing optional allowed. */
  allows(kind: 'analytics' | 'crash'): boolean {
    const s = this.read();
    if (s === 'unknown') return false;              // crash default here = consent-based; change per legal basis
    return s[kind];
  }

  save(choice: { analytics: boolean; crash: boolean }): ConsentRecord {
    const rec: ConsentRecord = { version: this.currentVersion, ...choice, decidedAt: new Date().toISOString() };
    this.kv.set(KEY, JSON.stringify(rec));
    this.listeners.forEach((l) => l(rec));
    return rec;
  }

  subscribe(l: Listener): () => void { this.listeners.add(l); return () => this.listeners.delete(l); }
  // Intentionally NO clear() method. Account wipes must not be able to reach this record.
}

export const consentStore = new ConsentStore();
```

### A.4 Facades, fakes, consent gate

```ts
// observability/types.ts
import type { AnalyticsEvent } from '../analytics/events';
export type Props = Record<string, string | number | boolean | undefined>;

export interface Analytics {
  setEnabled(on: boolean): Promise<void>;
  track(e: AnalyticsEvent): void;
  screen(routeName: string): void;
  identify(userId: string | null): void;
  setSuperProps(p: Props): void;
  flush(): Promise<void>;
  reset(): Promise<void>;          // new app-instance id, delete queued data
}
export interface CrashReporter {
  setEnabled(on: boolean): Promise<void>;
  nonFatal(err: unknown, ctx?: Props): void;
  fatal(err: unknown): Promise<void>;
  breadcrumb(message: string, data?: Props): void;
  setUser(id: string | null): void;
  setKeys(keys: Props): void;
  deleteUnsent(): Promise<void>;
  flush(): Promise<void>;
}

export const toError = (e: unknown): Error =>
  e instanceof Error ? e : new Error(typeof e === 'string' ? e : JSON.stringify(e));

export class FakeAnalytics implements Analytics {
  events: AnalyticsEvent[] = []; screens: string[] = []; userId: string | null = null; enabled = false;
  async setEnabled(on: boolean) { this.enabled = on; }
  track(e: AnalyticsEvent) { this.events.push(e); }
  screen(n: string) { this.screens.push(n); }
  identify(id: string | null) { this.userId = id; }
  setSuperProps() {}
  async flush() {}
  async reset() { this.events = []; }
}
```

```ts
// observability/consentGate.ts
import { assertValid, type AnalyticsEvent } from '../analytics/events';
import type { ConsentStore } from '../privacy/consentStore';
import type { Analytics, CrashReporter, Props } from './types';

/** Drops (never buffers) everything while consent is unknown or denied. */
export class ConsentGatedAnalytics implements Analytics {
  private lastScreen?: string;
  constructor(private inner: Analytics, private consent: ConsentStore) {}
  private get on() { return this.consent.allows('analytics'); }
  setEnabled(on: boolean) { return this.inner.setEnabled(on); }
  track(e: AnalyticsEvent) { assertValid(e); if (this.on) this.inner.track(e); }
  screen(n: string) { if (n === this.lastScreen) return; this.lastScreen = n; if (this.on) this.inner.screen(n); }
  identify(id: string | null) { this.inner.identify(this.on ? id : null); }
  setSuperProps(p: Props) { this.inner.setSuperProps(p); }
  flush() { return this.on ? this.inner.flush() : Promise.resolve(); }
  reset() { return this.inner.reset(); }
}

export class ConsentGatedCrash implements CrashReporter {
  constructor(private inner: CrashReporter, private consent: ConsentStore) {}
  private get on() { return this.consent.allows('crash'); }
  setEnabled(on: boolean) { return this.inner.setEnabled(on); }
  nonFatal(e: unknown, c?: Props) { if (this.on) this.inner.nonFatal(e, c); }
  fatal(e: unknown) { return this.on ? this.inner.fatal(e) : Promise.resolve(); }
  breadcrumb(m: string, d?: Props) { if (this.on) this.inner.breadcrumb(m, d); }
  setUser(id: string | null) { this.inner.setUser(this.on ? id : null); }
  setKeys(k: Props) { this.inner.setKeys(k); }
  deleteUnsent() { return this.inner.deleteUnsent(); }
  flush() { return this.inner.flush(); }
}
```

### A.5 Vendor sinks (the only files importing vendor SDKs)

```ts
// observability/firebase.ts
import {
  getAnalytics, logEvent, logScreenView, setAnalyticsCollectionEnabled, setConsent,
  setUserId as setAnalyticsUserId, setUserProperties, resetAnalyticsData,
} from '@react-native-firebase/analytics';
import {
  getCrashlytics, setCrashlyticsCollectionEnabled, recordError, log, setUserId,
  setAttributes, deleteUnsentReports, sendUnsentReports,
} from '@react-native-firebase/crashlytics';
import type { AnalyticsEvent } from '../analytics/events';
import { toError, type Analytics, type CrashReporter, type Props } from './types';

const clean = (p: Props) =>
  Object.fromEntries(Object.entries(p).filter(([, v]) => v !== undefined).map(([k, v]) => [k, String(v)]));

export class FirebaseAnalyticsSink implements Analytics {
  private a = getAnalytics();
  async setEnabled(on: boolean) {
    await setConsent(this.a, { analytics_storage: on, ad_storage: false, ad_user_data: false, ad_personalization: false });
    await setAnalyticsCollectionEnabled(this.a, on);
  }
  track(e: AnalyticsEvent) { void logEvent(this.a, e.name, { ...e.params, v: e.v } as never); }
  screen(n: string) { void logScreenView(this.a, { screen_name: n, screen_class: n }); }
  identify(id: string | null) { void setAnalyticsUserId(this.a, id); }
  setSuperProps(p: Props) { void setUserProperties(this.a, clean(p)); } // user properties ≈ super props
  async flush() { /* SDK batches itself */ }
  async reset() { await resetAnalyticsData(this.a); }
}

export class CrashlyticsReporter implements CrashReporter {
  private c = getCrashlytics();
  async setEnabled(on: boolean) {
    await setCrashlyticsCollectionEnabled(this.c, on);
    if (on) sendUnsentReports(this.c); else await deleteUnsentReports(this.c);
  }
  nonFatal(err: unknown, ctx?: Props) {
    if (ctx) void setAttributes(this.c, clean(ctx));
    const e = toError(err);
    recordError(this.c, e, e.name);           // jsErrorName → stable grouping
  }
  async fatal(err: unknown) { this.nonFatal(err, { fatal: true }); }
  breadcrumb(m: string, d?: Props) { log(this.c, d ? `${m} ${JSON.stringify(clean(d))}` : m); }
  setUser(id: string | null) { void setUserId(this.c, id ?? ''); }
  setKeys(k: Props) { void setAttributes(this.c, clean(k)); }
  async deleteUnsent() { await deleteUnsentReports(this.c); }
  async flush() {}
}
```

```ts
// observability/sentry.ts
import * as Sentry from '@sentry/react-native';
import * as Application from 'expo-application';
import * as Updates from 'expo-updates';
import Constants from 'expo-constants';
import { toError, type CrashReporter, type Props } from './types';

export const navigationIntegration = Sentry.reactNavigationIntegration({ enableTimeToInitialDisplay: true });
const SCRUB = /([\w.+-]+@[\w-]+\.[\w.]+)|(eyJ[\w-]+\.[\w-]+\.[\w-]+)|(\b\d{13,19}\b)/g;
const scrub = (s?: string) => s?.replace(SCRUB, '[redacted]');

export class SentryReporter implements CrashReporter {
  private enabled = false;
  private started = false;

  /** Call at startup; nothing leaves the device until setEnabled(true). */
  init() {
    if (this.started) return;
    this.started = true;
    Sentry.init({
      dsn: Constants.expoConfig?.extra?.sentryDsn,
      enabled: !__DEV__,
      environment: Constants.expoConfig?.extra?.env,
      release: `${Application.applicationId}@${Application.nativeApplicationVersion}+${Application.nativeBuildVersion}`,
      dist: Updates.updateId ?? Application.nativeBuildVersion ?? undefined,
      sendDefaultPii: false,
      enableAutoSessionTracking: false,        // turned on by consent (verify runtime toggle; else re-init)
      enableAppHangTracking: true,
      enableWatchdogTerminationTracking: true,
      tracesSampleRate: 0.1,
      tracePropagationTargets: [/^https:\/\/api\.example\.com/],
      integrations: [navigationIntegration],
      beforeSend: (event) => {
        if (!this.enabled) return null;
        event.message = scrub(event.message);
        event.exception?.values?.forEach((v) => (v.value = scrub(v.value)));
        return event;
      },
      beforeSendTransaction: (t) => (this.enabled ? t : null),
      beforeBreadcrumb: (b) => {
        if (b.category === 'http' && b.data?.url) b.data.url = String(b.data.url).split('?')[0];
        b.message = scrub(b.message);
        return b;
      },
    });
    Sentry.setTag('js_bundle_id', Updates.updateId ?? 'embedded');
    Sentry.setTag('channel', Updates.channel ?? 'none');
  }
  async setEnabled(on: boolean) { this.enabled = on; if (!on) await Sentry.flush(); }
  nonFatal(err: unknown, ctx?: Props) { Sentry.captureException(toError(err), { extra: ctx }); }
  async fatal(err: unknown) {
    Sentry.captureException(toError(err), { level: 'fatal' });
    await Sentry.flush();
  }
  breadcrumb(message: string, data?: Props) { Sentry.addBreadcrumb({ message, data, level: 'info' }); }
  setUser(id: string | null) { Sentry.setUser(id ? { id } : null); }
  setKeys(k: Props) { for (const [key, v] of Object.entries(k)) if (v !== undefined) Sentry.setTag(key, String(v)); }
  async deleteUnsent() { /* beforeSend gate drops; offline envelopes: verify SDK API */ }
  async flush() { await Sentry.flush(); }
}
```

### A.6 Composition root, global handlers, consent application

```ts
// observability/index.ts — import FIRST in index.ts / app/_layout.tsx
import { consentStore, type ConsentState } from '../privacy/consentStore';
import { ConsentGatedAnalytics, ConsentGatedCrash } from './consentGate';
import { CrashlyticsReporter, FirebaseAnalyticsSink } from './firebase';
import { SentryReporter } from './sentry';
import { toError, type CrashReporter } from './types';

const sentry = new SentryReporter();
sentry.init();

const fanOut = (rs: CrashReporter[]): CrashReporter => ({
  setEnabled: async (on) => { await Promise.all(rs.map((r) => r.setEnabled(on))); },
  nonFatal: (e, c) => rs.forEach((r) => r.nonFatal(e, c)),
  fatal: async (e) => { await Promise.all(rs.map((r) => r.fatal(e))); },
  breadcrumb: (m, d) => rs.forEach((r) => r.breadcrumb(m, d)),
  setUser: (id) => rs.forEach((r) => r.setUser(id)),
  setKeys: (k) => rs.forEach((r) => r.setKeys(k)),
  deleteUnsent: async () => { await Promise.all(rs.map((r) => r.deleteUnsent())); },
  flush: async () => { await Promise.all(rs.map((r) => r.flush())); },
});

// Pick ONE native crash reporter in production; both here for illustration.
export const crash = new ConsentGatedCrash(fanOut([sentry, new CrashlyticsReporter()]), consentStore);
export const analytics = new ConsentGatedAnalytics(new FirebaseAnalyticsSink(), consentStore);

/** Applies the device record to every SDK. Called at startup and on every change. */
export async function applyConsent(s: ConsentState = consentStore.read()) {
  const a = s !== 'unknown' && s.analytics;
  const c = s !== 'unknown' && s.crash;
  await Promise.all([analytics.setEnabled(a), crash.setEnabled(c)]);
  if (!a) await analytics.reset();          // drop queued data, new instance id
}
void applyConsent();
consentStore.subscribe((s) => void applyConsent(s));

// JS fatal handler — chained (Sentry/Crashlytics already installed theirs; we wrap them).
const previous = ErrorUtils.getGlobalHandler();
ErrorUtils.setGlobalHandler((error, isFatal) => {
  crash.breadcrumb('js_global_handler', { isFatal: !!isFatal });
  // Vendor handlers below report the error; we only add context. Don't double-report.
  previous(error, isFatal);
});

// Unhandled rejections (Hermes). Skip if Sentry's integration already covers it.
type HermesTracker = { enablePromiseRejectionTracker?: (o: {
  allRejections: boolean; onUnhandled: (id: number, e: unknown) => void; onHandled: () => void }) => void };
const hermes = (globalThis as { HermesInternal?: HermesTracker }).HermesInternal;
const SENTRY_TRACKS_REJECTIONS = true;
if (!SENTRY_TRACKS_REJECTIONS && hermes?.enablePromiseRejectionTracker) {
  hermes.enablePromiseRejectionTracker({
    allRejections: true,
    onUnhandled: (_id, e) => crash.nonFatal(toError(e), { kind: 'unhandled_rejection' }),
    onHandled: () => {},
  });
}
```

```tsx
// observability/ErrorBoundary.tsx
import React from 'react';
import { Button, Text, View } from 'react-native';
import * as Updates from 'expo-updates';
import { crash } from '.';

export class ScreenErrorBoundary extends React.Component<{ name: string; children: React.ReactNode }, { failed: boolean }> {
  state = { failed: false };
  static getDerivedStateFromError() { return { failed: true }; }
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    crash.nonFatal(error, { boundary: this.props.name, componentStack: info.componentStack?.slice(0, 1000) });
  }
  render() {
    if (!this.state.failed) return this.props.children;
    return (
      <View accessibilityRole="alert" style={{ flex: 1, alignItems: 'center', justifyContent: 'center', padding: 24 }}>
        <Text>Something went wrong.</Text>
        <Button title="Try again" onPress={() => this.setState({ failed: false })} />
        <Button title="Restart app" onPress={() => void Updates.reloadAsync()} />
      </View>
    );
  }
}
```

### A.7 Consent prompt hooked into first login

```tsx
// auth/postLogin.ts — the ONLY place the prompt is triggered
import { router } from 'expo-router';
import { consentStore, type ConsentStore } from '../privacy/consentStore';

export async function postLoginRoute(consent: ConsentStore = consentStore): Promise<'/consent' | '/(home)'> {
  return consent.needsPrompt() ? '/consent' : '/(home)';
}

/** Called by LoginScreen / AddAccount after tokens are stored. */
export async function onLoginSucceeded() {
  router.replace(await postLoginRoute());     // never reach home before a decision
}
```

```tsx
// app/consent.tsx
import { router } from 'expo-router';
import { useState } from 'react';
import { Linking, Pressable, Switch, Text, View } from 'react-native';
import * as Tracking from 'expo-tracking-transparency';
import { consentStore } from '../privacy/consentStore';

const NEEDS_ATT = false; // true only if you track across apps/sites (IDFA)

export default function ConsentScreen() {
  const prev = consentStore.read();
  const [crashOn, setCrashOn] = useState(prev === 'unknown' ? false : prev.crash);

  async function decide(analytics: boolean) {
    consentStore.save({ analytics, crash: crashOn });   // subscribers apply to SDKs
    if (analytics && NEEDS_ATT) await Tracking.requestTrackingPermissionsAsync();
    router.replace('/(home)');
  }

  return (
    <View style={{ flex: 1, padding: 24, gap: 16 }}>
      <Text accessibilityRole="header" style={{ fontSize: 22, fontWeight: '600' }}>Help us improve</Text>
      <Text>We'd like to collect usage analytics to understand which features you use. No ads, no selling data. Applies to all accounts on this device; change it any time in Settings › Privacy.</Text>
      <View style={{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' }}>
        <Text>Send crash reports</Text>
        <Switch value={crashOn} onValueChange={setCrashOn} accessibilityLabel="Send crash reports" />
      </View>
      <Text style={{ textDecorationLine: 'underline' }} onPress={() => Linking.openURL('https://example.com/privacy')}>Privacy policy</Text>
      {/* Equal-weight choices */}
      <Pressable accessibilityRole="button" onPress={() => decide(true)}><Text>Allow analytics</Text></Pressable>
      <Pressable accessibilityRole="button" onPress={() => decide(false)}><Text>Don't allow</Text></Pressable>
    </View>
  );
}
```

```tsx
// app/(home)/settings/privacy.tsx — change any time, applied immediately
import { Switch, Text, View } from 'react-native';
import { useSyncExternalStore } from 'react';
import { consentStore } from '../../../privacy/consentStore';
import { requestServerDeletion } from '../../../api/privacy';

export default function PrivacySettings() {
  const s = useSyncExternalStore((cb) => consentStore.subscribe(cb), () => JSON.stringify(consentStore.read()));
  const rec = JSON.parse(s) as ReturnType<typeof consentStore.read>;
  const a = rec !== 'unknown' && rec.analytics;
  const c = rec !== 'unknown' && rec.crash;
  return (
    <View style={{ padding: 16, gap: 12 }}>
      <Row label="Usage analytics" value={a} onChange={(v) => {
        consentStore.save({ analytics: v, crash: c });
        if (!v) void requestServerDeletion();      // optional GDPR erasure request
      }} />
      <Row label="Crash reports" value={c} onChange={(v) => consentStore.save({ analytics: a, crash: v })} />
    </View>
  );
}
const Row = ({ label, value, onChange }: { label: string; value: boolean; onChange: (v: boolean) => void }) => (
  <View style={{ flexDirection: 'row', justifyContent: 'space-between' }}>
    <Text>{label}</Text><Switch value={value} onValueChange={onChange} accessibilityLabel={label} />
  </View>
);
```

### A.8 Identity on login, account/portal switch, logout

```ts
// observability/identity.ts
import { analytics, crash } from '.';
import { deviceStorage } from '../privacy/consentStore';
import { Events } from '../analytics/events';

const ANON_KEY = 'device.anonymousId';
export function anonymousId(): string {
  let id = deviceStorage.getString(ANON_KEY);
  if (!id) { id = crypto.randomUUID(); deviceStorage.set(ANON_KEY, id); }
  return id;
}

export async function onSessionChanged(s: { serverUserId: string; portalId?: string; accountCount: number; accountIndex: number; reason: 'login' | 'switch' | 'portal' }) {
  await analytics.flush();                        // old identity's events leave first
  await crash.flush();
  analytics.identify(s.serverUserId);             // opaque id, never email
  analytics.setSuperProps({ portal_id: s.portalId, account_count: s.accountCount });
  crash.setUser(s.serverUserId);
  crash.setKeys({ portal_id: s.portalId, account_index: s.accountIndex, account_count: s.accountCount });
  crash.breadcrumb(`session_${s.reason}`);
  if (s.reason === 'switch') analytics.track(Events.account_switched({ account_count: s.accountCount }));
}

/** Last account signed out. Account data wipe happens elsewhere — and must not touch consent. */
export async function onSignedOutCompletely() {
  await analytics.flush();
  analytics.identify(null);
  crash.setUser(null);
  crash.setKeys({ portal_id: undefined, account_count: 0 });
  deviceStorage.delete(ANON_KEY);                 // new anonymous id next time
  // NOTE: deliberately NOT touching 'device.consent'.
}
```

```ts
// auth/wipe.ts — account wipe targets account-scoped stores only
import { MMKV } from 'react-native-mmkv';
export function wipeAccountData(accountId: string) {
  new MMKV({ id: `acct.${accountId}` }).clearAll();   // per-account instance
  // delete SQLite file, keychain service `auth.${accountId}`, caches… (react-native-multi-account §9)
  // NEVER: deviceStorage.clearAll(), AsyncStorage.clear()
}
```
