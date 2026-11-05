---
name: ios-push-notifications
description: Use when implementing, reviewing, testing, or debugging push and local notifications on iOS/iPadOS/macOS — APNs token-based auth, device token registration, permission flow (provisional, time-sensitive, critical), payloads and headers, foreground handling and deep links, actions/categories, badges, silent/background pushes, Notification Service and Content extensions, communication notifications, Live Activity pushes, FCM on iOS, provider server code, Push Notifications Console and simulator testing, App Review rules.
---

# iOS Push Notifications Skill

Guide for building, auditing, and testing notifications on Apple platforms: **remote pushes via APNs** (alert, background, Live Activity, VoIP), **local notifications**, the **permission flow**, **payloads and APNs headers**, **handling and routing taps**, **Notification Service / Content extensions**, and the **provider server**. Applies whether pushes come from your own server, Firebase Cloud Messaging (FCM), or a vendor (OneSignal, Braze, Airship…). Siblings: **android-push-notifications**, **cross-platform-push-notifications** (Flutter / React Native / Expo).

Apple adds notification features every year (interruption levels, communication notifications, broadcast Live Activity pushes…). Verify OS availability and API names against the current SDK before shipping.

## 0. Architecture (mental model)

```
+--------------------+   HTTP/2 + JWT (ES256, .p8 key)   +-------------------------+
| Provider server    | --------------------------------> | APNs                    |
| (yours / FCM /     |   api.push.apple.com              | api.push.apple.com      |
|  vendor)           |   api.sandbox.push.apple.com      +-------------------------+
| - token store      | <-- 410 Unregistered, errors ---            |
+--------------------+                                             v
        ^ device token + env + user                    +-------------------------+
        |                                              | Device                  |
+-----------------------------------------------+      | - Notification Center   |
| App                                           |      | - Service Extension     |
| - UNUserNotificationCenter: auth, delegate,   | <--- |   (mutable-content)     |
|   categories                                  |      | - Content Extension     |
| - registerForRemoteNotifications → token      |      +-------------------------+
| - PushManager (@MainActor): token upload,     |
|   permission state, routing taps → deep links |
+-----------------------------------------------+
```

Push types and what they're for:

| `apns-push-type` | Use | Priority | Notes |
|---|---|---|---|
| `alert` | Visible notification | 10 (or 5 to save power) | Most pushes |
| `background` | Silent content refresh (`content-available: 1`, no alert) | **5** only | Throttled; not delivered after the user force-quits the app |
| `liveactivity` | Update / start / end a Live Activity | 5 or 10 | Topic `<bundleID>.push-type.liveactivity`; separate tokens |
| `voip` | Incoming calls via PushKit | 10 | Topic `<bundleID>.voip`; **must** report a call to CallKit immediately |
| `location` | Location query (Location Push Service Extension) | 5 or 10 | Needs entitlement |
| `complication` / `fileprovider` / `mdm` / `pushtotalk` | Special cases | — | Check docs |

## 1. First, establish context

Find out (from the codebase or by asking):

- Minimum OS (features gated: provisional auth iOS 12, interruption levels & communication notifications iOS 15, `setBadgeCount` iOS 16, `openNotificationSettingsURLString` iOS 16, push-to-start Live Activities iOS 17.2, broadcast push iOS 18)
- **Who sends pushes**: own server (APNs direct), **FCM**, or a vendor SDK — and whether that SDK swizzles AppDelegate methods
- Notification types the product needs: transactional (orders, messages), reminders, marketing, real-time (chat, calls), silent sync, Live Activities
- Does the app need: rich media (images/video), end-to-end encrypted payloads, custom UI, actions/replies, grouping, badges
- Account model: tokens linked to users; multi-account on one device; logout behaviour
- Platforms: iPhone, iPad, Mac (Catalyst / native), Apple Watch (mirrored vs independent), Vision Pro
- Localization: server-localized text vs `loc-key` + app-bundled strings

Write a **notification catalog** before coding:

```
type              push-type   interruption   category        thread-id      tap → route
new_message       alert       active         MESSAGE (reply) conversation   /chat/<id>
order_shipped     alert       active         ORDER           order          /orders/<id>
payment_due       alert       time-sensitive REMINDER        billing        /billing
weekly_digest     alert       passive        —               digest         /home   (marketing: needs opt-in)
sync              background  —              —               —              (no UI)
```

## 2. Capabilities & configuration

- [ ] **Push Notifications** capability in Xcode → `aps-environment` entitlement (`development` for debug builds, `production` for TestFlight/App Store — Xcode sets it at export)
- [ ] **Background Modes → Remote notifications** only if you use silent/background pushes
- [ ] **APNs Auth Key** (.p8) created in Certificates, Identifiers & Profiles; note **Key ID** and **Team ID**. One key works for all your apps and both environments; prefer it over certificates (which expire yearly)
- [ ] Key stored only on the server / in FCM / vendor dashboard — never in the app or repo
- [ ] Extensions (Notification Service, Content) with their own bundle IDs, provisioning, and an **App Group** if they share data/keychain with the app
- [ ] Time-sensitive notifications: **Time Sensitive Notifications** capability. Critical alerts: **special entitlement from Apple** (health/safety use cases only)
- [ ] Communication notifications: **Communication Notifications** capability + Intents (`INSendMessageIntent` / `INStartCallIntent`)

## 3. Permission flow

- [ ] **Don't ask at first launch.** Ask when the value is obvious (after the user follows a chat, places an order, enables reminders) — optionally with a soft pre-prompt screen explaining what they'll get
- [ ] Or use **provisional authorization** (`.provisional`): notifications appear quietly in Notification Center without a prompt; the user keeps or turns them off from there. Good for low-urgency content
- [ ] Options requested match use: `.alert, .sound, .badge` (+ `.provisional`, `.timeSensitive` is a capability not an option, `.criticalAlert` only with entitlement)
- [ ] Read current status with `getNotificationSettings()` on launch and foreground (users change it in Settings); store it and report to the server (so marketing isn't sent to users who disabled alerts)
- [ ] Denied → in-app explanation + button to `UIApplication.openNotificationSettingsURLString` (iOS 16+; else `openSettingsURLString`). Never nag repeatedly
- [ ] The app must **work without notifications** (App Review): no blocking screens until permission is granted
- [ ] Per-topic preferences in your own Settings screen (messages / orders / marketing), synced to the server; marketing **off by default** unless the user opted in (Guideline 4.5.4)
- [ ] Optional: implement `userNotificationCenter(_:openSettingsFor:)` + request `.providesAppNotificationSettings` to show "<App> Notification Settings" in iOS Settings

## 4. Device token registration

- [ ] Call `UIApplication.shared.registerForRemoteNotifications()` **on every launch** (it's cheap; tokens can change after restore, reinstall, OS update). Registration doesn't need permission — alerts do
- [ ] Convert the token to hex: `deviceToken.map { String(format: "%02x", $0) }.joined()` — never `description`
- [ ] Upload to the server with: user ID (if signed in), **APNs environment** (sandbox vs production — derive from build configuration or the entitlement), bundle ID, app version, locale, time zone, notification settings
- [ ] Upload only when the token, user, or settings changed (cache last sent values) — but tolerate duplicates server-side
- [ ] `didFailToRegisterForRemoteNotificationsWithError` logged (simulators before Xcode 14/ Apple silicon, missing entitlement, no network)
- [ ] **Logout**: tell the server to unlink the token from the user (or delete it) so the next user doesn't get the previous user's pushes; multi-account apps send account IDs per token
- [ ] Server deletes tokens on APNs `410 Unregistered` (and on `400 BadDeviceToken` after checking the environment)
- [ ] FCM: send the **FCM registration token** to your server, not the APNs token (section 10)

## 5. Payload & APNs request

### 5.1 Headers (HTTP/2 POST `/3/device/<token>`)
- [ ] `authorization: bearer <JWT>` — ES256 JWT with `iss` = Team ID, `iat`, header `kid` = Key ID. **Reuse it** for 20–60 minutes; refreshing more often than every 20 minutes gets `TooManyProviderTokenUpdates`; older than 1 hour gets `ExpiredProviderToken`
- [ ] `apns-push-type` set on every request (required; must match payload)
- [ ] `apns-topic` = bundle ID (or `.voip` / `.push-type.liveactivity` suffixes)
- [ ] `apns-priority`: `10` immediate, `5` power-friendly (**required 5** for background pushes)
- [ ] `apns-expiration`: `0` = deliver once or drop (good for time-bound content), or a UNIX timestamp; APNs stores only the **latest** notification per device while offline
- [ ] `apns-collapse-id` to replace an earlier notification (e.g. score updates) — ≤ 64 bytes
- [ ] Endpoint matches the token's environment: `api.sandbox.push.apple.com` for development builds, `api.push.apple.com` for TestFlight/App Store
- [ ] One persistent HTTP/2 connection (or a small pool) reused for many pushes — don't open a connection per push

### 5.2 `aps` dictionary
```json
{
  "aps": {
    "alert": { "title": "Priya", "subtitle": "Design team", "body": "Shared a file" },
    "badge": 3,
    "sound": "default",
    "thread-id": "conversation-42",
    "category": "MESSAGE",
    "mutable-content": 1,
    "interruption-level": "active",
    "relevance-score": 0.8,
    "target-content-id": "conversation-42"
  },
  "type": "new_message",
  "route": "/chat/42",
  "messageId": "m-981"
}
```
- [ ] Total payload ≤ **4 KB** (5 KB for VoIP); send IDs, not whole objects — fetch the rest in the Service Extension or app
- [ ] Localized text either rendered server-side (per user locale) or via `title-loc-key` / `loc-key` / `loc-args` with strings in the app bundle (and the extension)
- [ ] `thread-id` for grouping; `category` for actions; `interruption-level` (`passive`, `active`, `time-sensitive`, `critical`) matches the catalog — don't mark marketing as time-sensitive
- [ ] `mutable-content: 1` only when a Service Extension will process it
- [ ] Custom keys outside `aps` for routing (`type`, `route`, IDs); never secrets or sensitive personal data in plain text (it shows on the lock screen and passes through Apple) — use E2E encryption + Service Extension when needed
- [ ] Badge: send the **absolute** count the server knows, or manage it locally — not "+1"

### 5.3 APNs responses to handle
| Status / reason | Action |
|---|---|
| `200` | Store `apns-id` for tracing |
| `400 BadDeviceToken` | Usually environment mismatch (sandbox token to production or vice versa); otherwise delete |
| `400 DeviceTokenNotForTopic` | Wrong `apns-topic` / bundle ID |
| `400 PayloadTooLarge`, `BadPriority`, `InvalidPushType` | Fix the request |
| `403 InvalidProviderToken` / `ExpiredProviderToken` | Fix JWT (Key ID, Team ID, clock, age) |
| `410 Unregistered` | **Delete the token** (app uninstalled / notifications reset) |
| `429 TooManyRequests` / `TooManyProviderTokenUpdates` | Back off; reuse JWT |
| `500` / `503` | Retry with backoff |

## 6. Handling notifications in the app

- [ ] `UNUserNotificationCenter.current().delegate` set **before the app finishes launching** (`application(_:didFinishLaunchingWithOptions:)`), otherwise taps that launch the app are missed
- [ ] `willPresent` (app in foreground): choose presentation per type (`.banner, .list, .sound, .badge`) — e.g. suppress the banner for the chat that's already on screen
- [ ] `didReceive response`: route by `response.notification.request.content.userInfo` → deep link. Works for **cold start** (app launched by the tap), background, and foreground. Call the completion handler (or return from the async variant) promptly
- [ ] Routing waits for the app to be ready (signed in, root UI loaded) — queue the pending route instead of pushing onto an unready navigation stack
- [ ] Validate route data (unknown type / missing ID → open home, don't crash)
- [ ] Categories registered at launch with `setNotificationCategories` — actions (`UNNotificationAction`, `UNTextInputNotificationAction` for replies), options (`.foreground`, `.destructive`, `.authenticationRequired` for sensitive actions)
- [ ] Action handlers do the work in the background (reply, mark read) and finish quickly; use a background task if network is needed
- [ ] `UNNotificationDefaultActionIdentifier` (tap) and `UNNotificationDismissActionIdentifier` (only delivered if the category has `.customDismissAction`)
- [ ] Badge: `UNUserNotificationCenter.current().setBadgeCount(_:)` (iOS 16+) when unread counts change; clear it when appropriate
- [ ] Remove delivered notifications that are no longer relevant (`removeDeliveredNotifications(withIdentifiers:)`), e.g. after reading a chat on another device (via a silent push or on open)
- [ ] SwiftUI apps: use `@UIApplicationDelegateAdaptor` for the APNs callbacks and delegate — SwiftUI has no direct replacement

### 6.1 Local notifications
- [ ] `UNNotificationRequest` with `UNCalendarNotificationTrigger` / `UNTimeIntervalNotificationTrigger` / `UNLocationNotificationTrigger`
- [ ] Stable identifiers so you can update/cancel (`removePendingNotificationRequests(withIdentifiers:)`); max **64 pending** per app — schedule the nearest ones and top up
- [ ] Re-schedule after time-zone changes, settings changes, logout
- [ ] Same routing and categories as remote notifications

## 7. Silent (background) pushes

- [ ] Payload `{"aps": {"content-available": 1}}` with **no** alert/sound/badge; `apns-push-type: background`, `apns-priority: 5`
- [ ] Handled in `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)` (or the async variant) — finish within ~30 s and report the result
- [ ] **Not guaranteed**: throttled by the system (budget, Low Power Mode), not delivered if the user force-quit the app, coalesced when many arrive. Never use them for anything that must happen; use them to make the next open faster
- [ ] Don't send more than a few per hour per device
- [ ] For user-visible updates that must arrive, use an alert push (optionally with a Service Extension)

## 8. Notification Service Extension (NSE)

Runs for alert pushes with `mutable-content: 1` before display.
- [ ] Modify `bestAttemptContent`: download and attach media (`UNNotificationAttachment`, file in the extension's temp dir with the right extension), decrypt E2E payloads, localize, fetch the message preview
- [ ] Finish within the time limit (~30 s) and always call `contentHandler`; implement `serviceExtensionTimeWillExpire` to deliver the best attempt (e.g. without the image)
- [ ] Memory limit is small (~24 MB) — downscale media, avoid big frameworks
- [ ] Shared data/credentials via **App Group** container and **keychain access group**; no main-app singletons
- [ ] Network calls with short timeouts; no UI; no `UIApplication`
- [ ] **Communication notifications** (iOS 15+): donate an `INSendMessageIntent` with the sender's `INPerson` (name, avatar `INImage`) and call `content.updating(from: intent)` → avatar shown, Focus breakthrough rules for people apply
- [ ] Filtering notifications (not showing some) needs the `com.apple.developer.usernotifications.filtering` entitlement from Apple — without it, the NSE must always deliver something

## 9. Notification Content Extension (custom UI)
- [ ] Only if the default UI isn't enough (maps, charts, interactive previews); `UNNotificationExtensionCategory` in Info.plist matches the category
- [ ] Set `UNNotificationExtensionDefaultContentHidden` / initial size ratio; keep it light
- [ ] Actions can be updated from the extension (`extensionContext?.notificationActions`)

## 10. Firebase Cloud Messaging (FCM) on iOS

- [ ] Upload the APNs Auth Key (.p8) to Firebase project settings (both environments)
- [ ] `FirebaseApp.configure()` at launch; `Messaging.messaging().delegate` → `messaging(_:didReceiveRegistrationToken:)` → send the **FCM token** to your server
- [ ] If swizzling is disabled (`FirebaseAppDelegateProxyEnabled = NO`), forward `Messaging.messaging().apnsToken = deviceToken` yourself
- [ ] Server sends via **FCM HTTP v1** with an `apns` block (`headers`: `apns-push-type`, `apns-priority`, `apns-collapse-id`; `payload.aps`: alert, `mutable-content`, etc.) — the legacy FCM API is gone
- [ ] Notification-type vs data-only messages behave differently on iOS: data-only requires `content-available` and is a silent push (section 7)
- [ ] All other checks in this skill still apply (permission flow, delegate before launch, NSE, routing)

## 11. Live Activity pushes (summary)

Full coverage belongs in a widgets/Live Activities skill; push-specific checks:
- [ ] Per-activity **push token** from `activity.pushTokenUpdates` (and **push-to-start** token from `Activity<…>.pushToStartTokenUpdates`, iOS 17.2+) sent to the server — separate from the device token
- [ ] Request: `apns-push-type: liveactivity`, topic `<bundleID>.push-type.liveactivity`, payload `aps.event` = `update` / `end` / `start`, `aps.timestamp`, `aps.content-state` matching the `ContentState` Codable exactly, optional `stale-date`, `dismissal-date`, `alert` for important updates
- [ ] Respect budgets: frequent priority-10 updates are throttled unless `NSSupportsLiveActivitiesFrequentUpdates` is set; prefer priority 5 for minor updates
- [ ] Broadcast channels (iOS 18+) for one-to-many updates (sports scores) instead of per-token fan-out — check availability

## 12. VoIP pushes (PushKit)

- [ ] Only for real incoming calls. Every VoIP push **must** report a new incoming call to **CallKit** (`reportNewIncomingCall`) before the handler returns — otherwise iOS terminates the app and eventually stops delivering VoIP pushes
- [ ] Separate PushKit token (`PKPushRegistry`), topic `<bundleID>.voip`, push type `voip`
- [ ] Not for messages or general wake-ups; use alert pushes + NSE instead

## 13. App Review & privacy

- [ ] App functions without push permission (4.5.4); no requiring notifications to use features
- [ ] **Marketing/promotional pushes only with explicit opt-in** and an in-app way to opt out (4.5.4)
- [ ] No sensitive personal info in visible notification text without user choice (e.g. "Hide previews" setting in-app for messages, health, finance)
- [ ] Don't use notifications for ads of other apps, spam, or to send content unrelated to the app
- [ ] Privacy manifest / nutrition labels declare device-ID and usage data if tokens are tied to users or analytics
- [ ] Critical alerts and time-sensitive only for content that genuinely warrants them (App Review checks entitlements usage)

## 14. Testing plan

1. **Simulator**: `xcrun simctl push booted <bundleID> payload.apns` or drag an `.apns` file onto the simulator (payload JSON with `"Simulator Target Bundle"`). Apple-silicon Macs with recent Xcode also register real sandbox tokens on the simulator
2. **Push Notifications Console** (developer.apple.com) — send test pushes to a device token with any headers, see delivery logs per `apns-id`
3. **Device with a debug build** → sandbox environment; **TestFlight** → production environment
4. **curl / script** with a JWT against `api.sandbox.push.apple.com` for header/payload experiments
5. **Server integration tests** against sandbox; monitor `410` cleanup

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| P1 | First launch | No permission prompt yet; token registered and uploaded (no alerts) |
| P2 | Contextual permission prompt → Allow | Status `.authorized`; server updated |
| P3 | Deny | App fully usable; in-app "enable notifications" → Settings |
| P4 | Provisional | Quiet delivery to Notification Center; keep/turn-off flow works |
| P5 | Alert push, app foreground | Presented per type (or suppressed on the relevant screen) |
| P6 | Tap push, app backgrounded | Routed to the right screen |
| P7 | Tap push, app **not running** (cold start) | Routed after launch/sign-in completes |
| P8 | Tap with unknown type / bad ID | Opens home; no crash |
| P9 | Reply action from lock screen | Reply sent without opening the app; auth required if configured |
| P10 | Rich push (image) | Image attached; falls back to text on slow network/timeout |
| P11 | E2E-encrypted push | Decrypted in NSE; plaintext never in payload |
| P12 | Communication notification | Sender avatar + name; Focus people rules apply |
| P13 | Silent push | Background refresh runs when budget allows; nothing breaks when it doesn't arrive |
| P14 | Collapse-id updates | Older notification replaced |
| P15 | Badge counts | Correct after read on another device / after opening |
| P16 | Logout → login as another user | Previous user's pushes no longer arrive |
| P17 | Reinstall / restore to new device | New token uploaded; old token gets 410 and is deleted |
| P18 | Debug build vs TestFlight | Correct environment used; no BadDeviceToken |
| P19 | Time-sensitive push during Focus | Breaks through only if allowed |
| P20 | Localized push (other language) | Correct language |
| P21 | Local reminder after time-zone change | Fires at the intended local time |
| P22 | iPad / Mac (Catalyst) / Watch | Delivery and routing work on each supported platform |
| P23 | Disable notifications in Settings while app running | Status refreshed on foreground; server updated |

## 15. Common bugs to look for in review

- Delegate set after launch (or in a view's `onAppear`) → cold-start taps ignored
- Permission requested on first launch with no context → high denial rate
- `registerForRemoteNotifications` only after permission or only once ever → stale tokens
- Token converted with `description` / `String(data:encoding:)` → garbage token
- Token not tied to environment → sandbox tokens sent to production (BadDeviceToken) or vice versa
- Tokens never deleted on `410`; token not unlinked on logout → wrong user receives pushes
- New JWT per request (`TooManyProviderTokenUpdates`) or JWT never refreshed (`ExpiredProviderToken`); new HTTP/2 connection per push
- Missing `apns-push-type`, or `background` sent with priority 10
- Relying on silent pushes for critical sync; expecting them after force-quit
- Payload too large (whole objects in the push); secrets or sensitive data in plain payload
- NSE without `serviceExtensionTimeWillExpire`, never calling `contentHandler` on errors, exceeding memory, using main-app singletons instead of the App Group
- `mutable-content` set but no NSE target (or NSE not embedded) — harmless but misleading; NSE missing from the release scheme
- Routing pushes onto a navigation stack before the root UI/sign-in is ready
- Badge incremented with "+1" logic and never cleared
- Marketing pushes without opt-in; everything marked time-sensitive
- VoIP pushes used for messages / not reporting to CallKit
- FCM: APNs token sent to the server instead of the FCM token; swizzling disabled without forwarding `apnsToken`
- Local notifications over the 64-pending limit silently dropped
- `.p8` key committed to the repo or bundled in the app

## 16. Output format

When asked to **review/audit**:
1. **Summary** — push provider (APNs direct / FCM / vendor), notification catalog (type → push-type → interruption level → route), extensions present, min OS
2. **Checklist results** — sections 2–13 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line (server-side or portal items listed for the user to confirm)
3. **Critical issues first** (missed/incorrect deliveries, wrong-user delivery, cold-start routing, data exposure, App Review risks) with fix code
4. **Testing gaps** — P1–P23 not covered

When asked to **implement**: start from the Appendix, adapt the catalog and routes, remove unused parts (NSE, actions, silent push), then list portal, server, and App Review steps the user must do by hand.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI)

### A.1 Catalog & routes

```swift
import Foundation

enum PushType: String, Sendable {
    case newMessage = "new_message"
    case orderShipped = "order_shipped"
    case paymentDue = "payment_due"
    case weeklyDigest = "weekly_digest"
}

enum NotificationCategory {
    static let message = "MESSAGE"
    static let order = "ORDER"
    static let reply = "REPLY_ACTION"
    static let markRead = "MARK_READ_ACTION"
    static let track = "TRACK_ACTION"
}

enum AppRoute: Hashable, Sendable {
    case home
    case chat(id: String)
    case order(id: String)
    case billing

    /// Validates payload data; unknown or incomplete payloads fall back to .home.
    init(userInfo: [AnyHashable: Any]) {
        switch (userInfo["type"] as? String).flatMap(PushType.init) {
        case .newMessage:   self = (userInfo["conversationId"] as? String).map { .chat(id: $0) } ?? .home
        case .orderShipped: self = (userInfo["orderId"] as? String).map { .order(id: $0) } ?? .home
        case .paymentDue:   self = .billing
        case .weeklyDigest, nil: self = .home
        }
    }
}
```

### A.2 PushManager

```swift
import UIKit
import UserNotifications

/// Server API for tokens and preferences (your backend).
protocol PushRegistrationAPI: Sendable {
    func upload(token: String, environment: String, userID: String?, settings: PushSettingsSnapshot) async throws
    func unlink(token: String, userID: String) async throws
}

struct PushSettingsSnapshot: Equatable, Sendable {
    var authorization: UNAuthorizationStatus
    var alertsEnabled: Bool
    var timeSensitiveEnabled: Bool
}

@MainActor
@Observable
final class PushManager: NSObject {
    private(set) var authorization: UNAuthorizationStatus = .notDetermined
    private(set) var deviceToken: String?
    /// Route waiting for the app to be ready (cold start, signed-out state).
    var pendingRoute: AppRoute?

    var currentUserID: String? {
        didSet { if oldValue != currentUserID { Task { await uploadIfNeeded(previousUserID: oldValue) } } }
    }

    private let api: PushRegistrationAPI
    private var lastUploaded: (token: String, userID: String?, settings: PushSettingsSnapshot)?

    init(api: PushRegistrationAPI) {
        self.api = api
        super.init()
    }

    /// Call from application(_:didFinishLaunchingWithOptions:) — BEFORE launch completes.
    func configureAtLaunch() {
        let center = UNUserNotificationCenter.current()
        center.delegate = self
        center.setNotificationCategories(Self.categories)
        UIApplication.shared.registerForRemoteNotifications()   // every launch; no prompt
        Task { await refreshSettings() }
    }

    /// Ask in context (after the user does something that benefits from notifications).
    func requestAuthorization(provisional: Bool = false) async {
        var options: UNAuthorizationOptions = [.alert, .sound, .badge]
        if provisional { options.insert(.provisional) }
        _ = try? await UNUserNotificationCenter.current().requestAuthorization(options: options)
        await refreshSettings()
    }

    /// Call on launch and whenever the app becomes active (users change Settings).
    func refreshSettings() async {
        let settings = await UNUserNotificationCenter.current().notificationSettings()
        authorization = settings.authorizationStatus
        await uploadIfNeeded()
    }

    func openSystemSettings() {
        if let url = URL(string: UIApplication.openNotificationSettingsURLString) {
            UIApplication.shared.open(url)
        }
    }

    // MARK: Token

    func didRegister(deviceToken data: Data) {
        deviceToken = data.map { String(format: "%02x", $0) }.joined()
        Task { await uploadIfNeeded() }
    }

    func didFailToRegister(_ error: Error) {
        print("[Push] Registration failed: \(error.localizedDescription)")
    }

    private func uploadIfNeeded(previousUserID: String? = nil) async {
        guard let token = deviceToken else { return }
        let settings = await snapshot()
        if let last = lastUploaded, last.token == token, last.userID == currentUserID, last.settings == settings { return }
        do {
            if let previous = previousUserID, previous != currentUserID {
                try await api.unlink(token: token, userID: previous)          // logout / account switch
            }
            try await api.upload(token: token, environment: Self.apnsEnvironment,
                                 userID: currentUserID, settings: settings)
            lastUploaded = (token, currentUserID, settings)
        } catch {
            // Retried on next launch / foreground / settings change.
        }
    }

    private func snapshot() async -> PushSettingsSnapshot {
        let s = await UNUserNotificationCenter.current().notificationSettings()
        return PushSettingsSnapshot(authorization: s.authorizationStatus,
                                    alertsEnabled: s.alertSetting == .enabled,
                                    timeSensitiveEnabled: s.timeSensitiveSetting == .enabled)
    }

    /// Must match the aps-environment the build is signed with.
    static var apnsEnvironment: String {
        #if DEBUG
        "sandbox"
        #else
        "production"
        #endif
    }

    // MARK: Badge

    func setBadge(_ count: Int) {
        UNUserNotificationCenter.current().setBadgeCount(count)
    }

    // MARK: Categories

    private static var categories: Set<UNNotificationCategory> {
        let reply = UNTextInputNotificationAction(identifier: NotificationCategory.reply, title: "Reply",
                                                  options: [.authenticationRequired],
                                                  textInputButtonTitle: "Send", textInputPlaceholder: "Message")
        let markRead = UNNotificationAction(identifier: NotificationCategory.markRead, title: "Mark as Read")
        let track = UNNotificationAction(identifier: NotificationCategory.track, title: "Track", options: [.foreground])
        return [
            UNNotificationCategory(identifier: NotificationCategory.message, actions: [reply, markRead],
                                   intentIdentifiers: [], options: []),
            UNNotificationCategory(identifier: NotificationCategory.order, actions: [track],
                                   intentIdentifiers: [], options: []),
        ]
    }
}

// MARK: - UNUserNotificationCenterDelegate

extension PushManager: UNUserNotificationCenterDelegate {
    /// Foreground presentation.
    nonisolated func userNotificationCenter(_ center: UNUserNotificationCenter,
                                            willPresent notification: UNNotification) async
        -> UNNotificationPresentationOptions {
        let route = AppRoute(userInfo: notification.request.content.userInfo)
        let onScreen = await MainActor.run { AppState.shared.visibleRoute == route }
        return onScreen ? [.list] : [.banner, .list, .sound]   // don't banner the chat the user is reading
    }

    /// Taps and actions — including cold start.
    nonisolated func userNotificationCenter(_ center: UNUserNotificationCenter,
                                            didReceive response: UNNotificationResponse) async {
        let userInfo = response.notification.request.content.userInfo
        switch response.actionIdentifier {
        case NotificationCategory.reply:
            if let text = (response as? UNTextInputNotificationResponse)?.userText,
               let conversationID = userInfo["conversationId"] as? String {
                try? await MessagesAPI.shared.sendReply(text, to: conversationID)
            }
        case NotificationCategory.markRead:
            if let conversationID = userInfo["conversationId"] as? String {
                try? await MessagesAPI.shared.markRead(conversationID)
            }
        default:   // UNNotificationDefaultActionIdentifier (tap) and foreground actions
            let route = AppRoute(userInfo: userInfo)
            await MainActor.run { self.pendingRoute = route }  // consumed when the UI is ready (A.3)
        }
    }
}
```

`AppState.shared.visibleRoute` and `MessagesAPI` are app-specific placeholders.

### A.3 App wiring — delegate before launch completes, route when ready

```swift
import SwiftUI

final class AppDelegate: NSObject, UIApplicationDelegate {
    let push = PushManager(api: PushBackend())

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        push.configureAtLaunch()
        return true
    }

    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        push.didRegister(deviceToken: deviceToken)
    }

    func application(_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: Error) {
        push.didFailToRegister(error)
    }

    /// Silent pushes: finish fast; never rely on delivery.
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any]) async -> UIBackgroundFetchResult {
        guard (userInfo["aps"] as? [String: Any])?["content-available"] as? Int == 1 else { return .noData }
        return (try? await SyncService.shared.syncIfNeeded()) == true ? .newData : .noData
    }
}

@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(appDelegate.push)
        }
        .onChange(of: scenePhase) { _, phase in
            if phase == .active { Task { await appDelegate.push.refreshSettings() } }
        }
    }
}

struct RootView: View {
    @Environment(PushManager.self) private var push
    @State private var path: [AppRoute] = []
    @State private var isReady = false               // signed in + data loaded

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: AppRoute.self) { RouteView(route: $0) }
        }
        .task { isReady = await Session.shared.restore() }
        // Route only when the UI is ready — covers cold start from a notification tap.
        .onChange(of: push.pendingRoute) { _, _ in consumeRoute() }
        .onChange(of: isReady) { _, _ in consumeRoute() }
    }

    private func consumeRoute() {
        guard isReady, let route = push.pendingRoute else { return }
        push.pendingRoute = nil
        path = route == .home ? [] : [route]
    }
}
```

`Session`, `SyncService`, `HomeView`, `RouteView`, `PushBackend` are app placeholders.

### A.4 Notification Service Extension — media, decryption, timeout fallback

```swift
import UserNotifications
import Intents

final class NotificationService: UNNotificationServiceExtension {
    private var contentHandler: ((UNNotificationContent) -> Void)?
    private var bestAttempt: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.contentHandler = contentHandler
        guard let content = request.content.mutableCopy() as? UNMutableNotificationContent else {
            return contentHandler(request.content)
        }
        bestAttempt = content

        Task {
            // 1. Decrypt E2E payload (key shared via App Group keychain).
            if let cipher = content.userInfo["enc"] as? String,
               let plain = try? E2ECrypto.decrypt(cipher, keychainGroup: "group.com.example.app") {
                content.title = plain.title
                content.body = plain.body
            }

            // 2. Attach media (short timeout; small files only).
            if let urlString = content.userInfo["imageURL"] as? String, let url = URL(string: urlString),
               let attachment = await Self.downloadAttachment(from: url) {
                content.attachments = [attachment]
            }

            // 3. Communication notification: sender avatar + Focus people rules (iOS 15+).
            if let senderName = content.userInfo["senderName"] as? String {
                let sender = INPerson(personHandle: INPersonHandle(value: content.userInfo["senderId"] as? String,
                                                                   type: .unknown),
                                      nameComponents: nil, displayName: senderName,
                                      image: nil, contactIdentifier: nil, customIdentifier: nil)
                let intent = INSendMessageIntent(recipients: nil, outgoingMessageType: .outgoingMessageText,
                                                 content: content.body, speakableGroupName: nil,
                                                 conversationIdentifier: content.threadIdentifier,
                                                 serviceName: nil, sender: sender, attachments: nil)
                let interaction = INInteraction(intent: intent, response: nil)
                interaction.direction = .incoming
                try? await interaction.donate()
                if let updated = try? content.updating(from: intent) {
                    contentHandler(updated)
                    return
                }
            }

            contentHandler(content)
        }
    }

    /// Called just before the time limit — deliver what we have.
    override func serviceExtensionTimeWillExpire() {
        if let contentHandler, let bestAttempt { contentHandler(bestAttempt) }
    }

    private static func downloadAttachment(from url: URL) async -> UNNotificationAttachment? {
        var request = URLRequest(url: url)
        request.timeoutInterval = 10
        guard let (tempURL, response) = try? await URLSession.shared.download(for: request),
              let http = response as? HTTPURLResponse, http.statusCode == 200 else { return nil }
        let ext = url.pathExtension.isEmpty ? "jpg" : url.pathExtension
        let fileURL = FileManager.default.temporaryDirectory.appendingPathComponent(UUID().uuidString + "." + ext)
        guard (try? FileManager.default.moveItem(at: tempURL, to: fileURL)) != nil else { return nil }
        return try? UNNotificationAttachment(identifier: "media", url: fileURL)
    }
}
```

Note: the `contentHandler` can only be called once — guard with a flag if both the task and `serviceExtensionTimeWillExpire` might call it. `E2ECrypto` is a placeholder for your decryption code.

### A.5 Provider server (Node/TypeScript, HTTP/2 + JWT)

```ts
import http2 from "node:http2";
import { SignJWT, importPKCS8 } from "jose";

const TEAM_ID = process.env.APNS_TEAM_ID!;
const KEY_ID = process.env.APNS_KEY_ID!;
const PRIVATE_KEY = process.env.APNS_P8!;           // contents of AuthKey_XXXX.p8 — from a secret store
const BUNDLE_ID = process.env.APNS_BUNDLE_ID!;

let cachedJWT: { token: string; issuedAt: number } | undefined;
async function providerToken(): Promise<string> {
  const now = Math.floor(Date.now() / 1000);
  if (cachedJWT && now - cachedJWT.issuedAt < 40 * 60) return cachedJWT.token;   // reuse 20–60 min
  const key = await importPKCS8(PRIVATE_KEY, "ES256");
  const token = await new SignJWT({}).setProtectedHeader({ alg: "ES256", kid: KEY_ID })
    .setIssuer(TEAM_ID).setIssuedAt(now).sign(key);
  cachedJWT = { token, issuedAt: now };
  return token;
}

// One long-lived HTTP/2 session per environment.
const sessions = new Map<string, http2.ClientHttp2Session>();
function session(env: "sandbox" | "production") {
  const host = env === "sandbox" ? "https://api.sandbox.push.apple.com" : "https://api.push.apple.com";
  let s = sessions.get(env);
  if (!s || s.closed || s.destroyed) {
    s = http2.connect(host);
    s.on("error", () => sessions.delete(env));
    sessions.set(env, s);
  }
  return s;
}

type PushOptions = {
  pushType?: "alert" | "background";
  priority?: 10 | 5;
  collapseId?: string;
  expiration?: number;   // 0 = now or never
};

export async function sendPush(device: { token: string; env: "sandbox" | "production" },
                               payload: object, opts: PushOptions = {}) {
  const pushType = opts.pushType ?? "alert";
  const headers: http2.OutgoingHttpHeaders = {
    ":method": "POST",
    ":path": `/3/device/${device.token}`,
    authorization: `bearer ${await providerToken()}`,
    "apns-topic": BUNDLE_ID,
    "apns-push-type": pushType,
    "apns-priority": String(pushType === "background" ? 5 : opts.priority ?? 10),
    ...(opts.collapseId && { "apns-collapse-id": opts.collapseId }),
    ...(opts.expiration !== undefined && { "apns-expiration": String(opts.expiration) }),
  };

  return new Promise<{ status: number; reason?: string; apnsId?: string }>((resolve, reject) => {
    const req = session(device.env).request(headers);
    let status = 0, apnsId: string | undefined, body = "";
    req.on("response", (h) => { status = Number(h[":status"]); apnsId = h["apns-id"] as string; });
    req.on("data", (chunk) => (body += chunk));
    req.on("end", async () => {
      const reason = body ? JSON.parse(body).reason : undefined;
      if (status === 410 || reason === "Unregistered") await db.pushTokens.delete(device.token);
      resolve({ status, reason, apnsId });
    });
    req.on("error", reject);
    req.end(JSON.stringify(payload));
  });
}

// Example: new message with NSE processing and grouping.
await sendPush({ token, env: "production" }, {
  aps: {
    alert: { title: "Priya", body: "Shared a file" },
    sound: "default",
    "thread-id": "conversation-42",
    category: "MESSAGE",
    "mutable-content": 1,
    "interruption-level": "active",
  },
  type: "new_message",
  conversationId: "42",
}, { collapseId: "conversation-42" });
```

Retry `429`/`5xx` with exponential backoff; fan out large sends with a concurrency limit on the shared session.

### A.6 Simulator test payload (`message.apns`)

```json
{
  "Simulator Target Bundle": "com.example.app",
  "aps": {
    "alert": { "title": "Priya", "body": "Shared a file" },
    "sound": "default",
    "category": "MESSAGE",
    "thread-id": "conversation-42",
    "mutable-content": 1
  },
  "type": "new_message",
  "conversationId": "42",
  "imageURL": "https://example.com/preview.jpg"
}
```

```bash
xcrun simctl push booted com.example.app message.apns
```
