---
name: flutter-push-notifications
description: Use when implementing, reviewing, testing, or debugging push and local notifications in Flutter apps (firebase_messaging + flutter_local_notifications) — permission on iOS and Android 13+, FCM/APNs tokens, per-platform message design (data vs notification), foreground/background/terminated handling, background isolate handlers, tap routing including cold start, channels and categories, native pieces (iOS Notification Service Extension, Android icons), scheduling, server sending, testing. For native iOS use ios-push-notifications; native Android use android-push-notifications; React Native or Expo use react-native-push-notifications.
---

# Flutter Push Notifications

Guide for building, auditing, and testing notifications in **Flutter** apps that target iOS and Android. One Dart layer has to cope with two notification systems that behave differently — especially for **data vs notification messages**, **foreground display**, **background handlers**, and **taps that launch the app**.

Platform skills hold the native detail — this skill links to them instead of repeating it:
- **ios-push-notifications** — APNs, permission options, payload/headers, Service Extension, communication notifications, App Review
- **android-push-notifications** — FCM message types, POST_NOTIFICATIONS, channels, PendingIntents, styles, Doze, Play policy
- **react-native-push-notifications** — the same topic for React Native (RNFirebase + Notifee) and Expo

Library APIs change between majors (`firebase_messaging` and `flutter_local_notifications` both had breaking changes recently). Verify every call against the versions in `pubspec.yaml`.

## 0. Architecture (mental model)

```
+-------------------------------------------------------------+
| Shared UI: permission prompt, notification settings,        |
| router that consumes "pending route" when the app is ready  |
+-------------------------------------------------------------+
            ^
+-------------------------------------------------------------+
| PushService (singleton, initialized at app start)           |
| - permission + settings state                               |
| - token → backend (platform, user, enabled, locale)         |
| - foreground handler → local display (Android) / options    |
| - tap handlers for 3 app states → pending route             |
| - channels (Android) + categories (iOS) registered at start |
+-------------------------------------------------------------+
   | firebase_messaging
   | + flutter_local_notifications
   v
+--------------------------+   +-----------------------------------+
| iOS: APNs (via FCM)      |   | Android: FCM                      |
| + native NSE target      |   | + channels, small icon, meta-data |
+--------------------------+   +-----------------------------------+
            ^                               ^
+-------------------------------------------------------------+
| Backend: token store per platform; sends via FCM HTTP v1    |
| (both platforms), building a platform-specific payload      |
+-------------------------------------------------------------+
```

How the same push behaves per platform (FCM):

| Message | iOS | Android |
|---|---|---|
| **notification** (+ data) | Shown by the system in background; in foreground only if presentation options allow | Shown by the system in background (your code doesn't run); **not shown in foreground** unless you display it |
| **data-only** | Becomes a **silent push** (`content-available`) — throttled, not delivered after force-quit, **unreliable** | Delivered to your handler in all states (unless force-stopped) — you display it |
| Taps on system-shown notifications | `getInitialMessage` / `onMessageOpenedApp` | Same |
| Taps on locally displayed notifications | `flutter_local_notifications` callbacks | Same |

Practical rule: **iOS → alert (notification) push with `mutable-content` + data; Android → data-only high-priority push displayed by `flutter_local_notifications`.** The server builds a different payload per platform (Appendix B).

## 1. First, establish context

Find out (from the codebase or by asking):

- Libraries with **exact versions**: `firebase_messaging`, `flutter_local_notifications`, `firebase_core`, `timezone`
- Push provider: FCM for both platforms, APNs direct for iOS + FCM for Android, or a vendor (OneSignal, Braze…) with its own SDK
- Notification catalog (type → iOS interruption level/category → Android channel/importance → route), as in the native skills
- Rich needs: images, actions/replies, conversation style, E2E encryption → these need **native code** (iOS Service Extension; Android style support in `flutter_local_notifications`)
- Account model, logout behaviour, multi-account
- Navigation library (go_router / auto_route) — for deep-link routing

## 2. Setup

### 2.1 Flutter
- [ ] `flutterfire configure` → `firebase_options.dart`; `Firebase.initializeApp` before anything else
- [ ] iOS: Push Notifications + Background Modes (Remote notifications) capabilities in `ios/Runner`; **APNs Auth Key uploaded to Firebase**
- [ ] Android: `google-services.json` per flavor; default icon/channel meta-data (android-push-notifications §2); `POST_NOTIFICATIONS` in the manifest
- [ ] `flutter_local_notifications` initialized with a **monochrome** Android small icon (`@drawable/ic_notification`) and Darwin settings with permission requests **off** (you request them in context)

### 2.2 Native pieces you still need
- [ ] **iOS Notification Service Extension** (Swift target in `ios/`) for images, decryption, communication notifications — ios-push-notifications §8. Flutter can't run Dart there
- [ ] Android small icon and colour resources; channels created at startup (via `flutter_local_notifications`)
- [ ] App Group / keychain sharing if the NSE needs credentials

## 3. Permission flow

- [ ] Ask **in context**, once per platform rules — not at first launch. Same pre-prompt UI for both platforms
- [ ] iOS: `requestPermission(alert, badge, sound[, provisional])`; provisional for low-urgency content
- [ ] Android 13+: the request maps to `POST_NOTIFICATIONS` (`FirebaseMessaging.requestPermission` handles it; `flutter_local_notifications` also offers `requestNotificationsPermission` on the Android implementation). Android ≤ 12: granted by default
- [ ] Denied: in-app explanation + open app notification settings (`openSettings` / `AppSettings`); never nag
- [ ] Read settings on start/resume (users change them in system settings); send `enabled` to the backend
- [ ] App works without notifications; marketing only with opt-in

## 4. Tokens

- [ ] **FCM**: `getToken()` on both platforms; `onTokenRefresh` → upload. iOS: the FCM token needs the APNs token first — wait for/handle `getAPNSToken()` being null shortly after launch, don't crash
- [ ] Upload with: `platform` (`ios`/`android`), token kind, user ID, app version, locale, time zone, enabled state, APNs environment if sending to APNs directly
- [ ] Upload on start/resume if anything changed; tolerate duplicates server-side
- [ ] **Logout**: unlink token from the user (or `deleteToken()` and fetch a new one)
- [ ] Server deletes invalid tokens (FCM `UNREGISTERED`; APNs 410)

## 5. Receiving — three app states × two platforms

| State | Flutter |
|---|---|
| Foreground | `FirebaseMessaging.onMessage` → Android: show via `flutter_local_notifications`; iOS: `setForegroundNotificationPresentationOptions` or show locally |
| Background / terminated (data) | `FirebaseMessaging.onBackgroundMessage(topLevelFn)` — **separate isolate**, `@pragma('vm:entry-point')`, re-init Firebase |
| Tap, app in background | `onMessageOpenedApp` (system-shown) / local-notifications `onDidReceiveNotificationResponse` |
| Tap, app **terminated** (cold start) | `getInitialMessage()` + `getNotificationAppLaunchDetails()` |
| Action / reply in background | `onDidReceiveBackgroundNotificationResponse` (top-level, isolate) |

Checks:
- [ ] Every row handled — the **cold-start row** is the one most often missing
- [ ] Both sources handled when you mix system-shown (notification messages) and locally displayed notifications — they use different tap callbacks
- [ ] Background handlers are **top-level functions** with `@pragma('vm:entry-point')`, do minimal work, and don't touch UI state (separate isolate)
- [ ] Taps produce a **pending route**; the router consumes it only when sign-in and the root navigator are ready
- [ ] Foreground suppression for the screen the user is already on
- [ ] Unknown types/missing IDs → home, no crash

## 6. Display layer (Android mainly, iOS for local notifications)

- [ ] Channels created at startup with the catalog's importance (android-push-notifications §4) — `flutter_local_notifications` `createNotificationChannel`
- [ ] Stable notification IDs per entity (conversation/order) → updates replace; cancel when read elsewhere
- [ ] Grouping (Android `groupKey` + summary; iOS `threadIdentifier`)
- [ ] Actions: iOS categories registered at startup (`DarwinNotificationCategory`); Android actions per notification (`AndroidNotificationAction`); text-input reply where supported — update the notification after replying
- [ ] Styles: big picture / big text / messaging style (`BigPictureStyleInformation`, `BigTextStyleInformation`, `MessagingStyleInformation`); images on iOS come from the **NSE**, not from Dart
- [ ] Payload passed to the local notification (`payload`) contains the route data
- [ ] Sensitive content hidden on the lock screen (Android visibility private; iOS previews) or encrypted + NSE

## 7. Local notifications & scheduling

- [ ] Time zones: `flutter_local_notifications.zonedSchedule` with the `timezone` package
- [ ] Android exact alarms need permission on 12+/14+ (`SCHEDULE_EXACT_ALARM`) — prefer inexact unless essential; reschedule after reboot (the plugin offers boot receivers — enable them in the manifest)
- [ ] iOS: max **64 pending** — schedule the nearest and top up
- [ ] Re-schedule after time-zone change, logout/login, settings change

## 8. Server

- [ ] One token table: `{ token, kind: fcm|apns, platform, userId, enabled, locale, appVersion, lastSeen }`
- [ ] **Per-platform payloads** (Appendix B): iOS → `notification` + `apns` block (`mutable-content`, `thread-id`, `category`, `interruption-level`, `apns-collapse-id`); Android → data-only with `android.priority: high` (+ `collapse_key`, `ttl`)
- [ ] FCM HTTP v1 / firebase-admin `sendEach` (≤ 500 per call); legacy FCM API is gone
- [ ] Localize per user locale server-side (or use `loc-key` on iOS + strings on Android)
- [ ] Respect per-user preferences and `enabled` state; no marketing without opt-in

## 9. Policies & privacy

- [ ] Both stores: app usable without notifications; marketing opt-in + opt-out; no sensitive data in visible text by default (ios-push-notifications §13, android-push-notifications §11)
- [ ] Privacy manifest (iOS) and Data safety (Android) declare push identifiers tied to users/analytics

## 10. Testing plan

1. **iOS simulator**: `xcrun simctl push` with an `.apns` file (alert pushes; real FCM tokens on Apple-silicon simulators with recent Xcode)
2. **Android emulator with Google Play image**: Firebase console test message / HTTP v1 curl
3. **Physical devices** for both platforms, including a killed-app cold start and Android Doze (`adb shell dumpsys deviceidle force-idle`)
4. Unit tests: route parsing and payload → route mapping in pure Dart

### Scenario matrix

| # | Scenario | iOS | Android |
|---|---|---|---|
| X1 | First launch | No prompt; token uploaded when available | No prompt (13+); token uploaded |
| X2 | Contextual permission → allow / deny | Status reflected; settings link on deny | Same (POST_NOTIFICATIONS) |
| X3 | Push, app foreground | Shown per presentation options (or suppressed on screen) | Displayed by `flutter_local_notifications` (or suppressed) |
| X4 | Push, app background | System alert | Data message displayed by local layer |
| X5 | Push, app killed | System alert | Background isolate handler displays (not force-stopped) |
| X6 | Tap, app background | Routed | Routed |
| X7 | Tap, app killed (cold start) | Routed after app ready | Routed after app ready |
| X8 | Action / reply from notification | Handled; notification updated | Same |
| X9 | Image push | Via NSE; text fallback on timeout | Big picture via local layer |
| X10 | Data-only push sent to iOS (bug check) | Not relied on (silent) | — |
| X11 | Logout → login as another user | No pushes for previous user | Same |
| X12 | Reinstall | New token; old one cleaned up server-side | Same |
| X13 | Local reminder after time-zone change / reboot | Fires at intended local time | Rescheduled after reboot |
| X14 | Notifications disabled in system settings | Status refreshed on resume; server updated | Same + per-channel blocks |

## 11. Common bugs to look for in review

- Same payload for both platforms: data-only to iOS (silent → unreliable) or notification messages to Android (not shown in foreground; background taps go through a different path)
- Android foreground messages never displayed (no `flutter_local_notifications` layer)
- Cold-start tap not handled (`getInitialMessage` / `getNotificationAppLaunchDetails` missing), or handled before the navigator/sign-in is ready
- Mixing system-shown and local notifications but handling taps for only one source
- Background handler not top-level / missing `@pragma('vm:entry-point')` / Firebase not initialized in the isolate; touching UI state from it
- iOS FCM token requested before the APNs token exists (crash / null); APNs key not uploaded to Firebase
- Permission requested on first launch (e.g. Darwin init settings requesting permissions); no settings link after denial
- No iOS Service Extension but images/decryption expected; trying to run Dart in the NSE
- Full-colour Android small icon (grey square)
- Tokens not unlinked on logout; invalid tokens never cleaned up
- Notification IDs random → duplicates instead of updates; no grouping
- Library upgraded across a major without migrating (`firebase_messaging` / `flutter_local_notifications` API changes)

## 12. Output format

When asked to **review/audit**:
1. **Summary** — libraries + versions, provider (FCM / APNs+FCM / vendor), catalog, native extensions present, per-platform payload strategy
2. **Checklist results** — sections 2–9 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android**
3. **Critical issues first** (missed deliveries from payload choice, foreground not shown on Android, cold-start routing, wrong-user delivery, data exposure, policy risks) with fix code
4. **Testing gaps** — X1–X14 not covered, per platform

When asked to **implement**: start from Appendix A, adapt catalog and routes, add the native pieces (NSE, icons), then list Firebase, server (Appendix B), and store-policy steps.

---

## Appendix A — Flutter (`firebase_messaging` + `flutter_local_notifications`)

```dart
import 'dart:convert';
import 'dart:io';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'firebase_options.dart';

const _messagesChannel = AndroidNotificationChannel(
  'messages', 'Messages', importance: Importance.high, description: 'New messages');
final _local = FlutterLocalNotificationsPlugin();

/// Background/terminated data messages (Android). Runs in a separate isolate.
@pragma('vm:entry-point')
Future<void> firebaseBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  if (Platform.isAndroid && message.notification == null) await showLocal(message.data);
}

/// Taps on local notifications while the app is terminated/backgrounded (isolate).
@pragma('vm:entry-point')
void localTapBackground(NotificationResponse response) {
  // Keep minimal: persist the payload; the app reads it on next launch via getNotificationAppLaunchDetails.
}

Future<void> showLocal(Map<String, dynamic> data) async {
  final id = (data['conversationId'] ?? data['orderId'] ?? data['type'] ?? 'x').hashCode;   // stable → replace
  await _local.show(
    id, data['title'] as String?, data['body'] as String?,
    NotificationDetails(
      android: AndroidNotificationDetails(_messagesChannel.id, _messagesChannel.name,
          importance: Importance.high, priority: Priority.high,
          groupKey: data['type'] as String?, visibility: NotificationVisibility.private,
          icon: '@drawable/ic_notification'),
      iOS: DarwinNotificationDetails(threadIdentifier: data['conversationId'] as String?),
    ),
    payload: jsonEncode(data),
  );
}

sealed class AppRoute {
  static AppRoute fromData(Map<String, dynamic> d) => switch (d['type']) {
        'new_message' when d['conversationId'] != null => ChatRoute(d['conversationId'] as String),
        'order_shipped' when d['orderId'] != null => OrderRoute(d['orderId'] as String),
        _ => const HomeRoute(),
      };
}
class HomeRoute extends AppRoute { const HomeRoute(); }
class ChatRoute extends AppRoute { ChatRoute(this.id); final String id; }
class OrderRoute extends AppRoute { OrderRoute(this.id); final String id; }

abstract class PushBackend {
  Future<void> upload({required String token, required String platform, String? userId, required bool enabled});
  Future<void> unlink({required String token, required String userId});
}

class PushService extends ChangeNotifier {
  PushService(this._backend);
  final PushBackend _backend;
  final _fcm = FirebaseMessaging.instance;

  AuthorizationStatus status = AuthorizationStatus.notDetermined;
  AppRoute? pendingRoute;            // consumed by the router when the app is ready
  String? _token;
  String? userId;

  /// Call from main() after Firebase.initializeApp and BEFORE runApp's first frame needs routing.
  Future<void> init() async {
    FirebaseMessaging.onBackgroundMessage(firebaseBackgroundHandler);

    await _local.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@drawable/ic_notification'),
        iOS: DarwinInitializationSettings(        // request permission in context, not here
            requestAlertPermission: false, requestBadgePermission: false, requestSoundPermission: false),
      ),
      onDidReceiveNotificationResponse: (r) => _routeFromPayload(r.payload),
      onDidReceiveBackgroundNotificationResponse: localTapBackground,
    );
    await _local.resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(_messagesChannel);

    // iOS foreground presentation of system (notification) messages.
    await _fcm.setForegroundNotificationPresentationOptions(alert: true, badge: true, sound: true);

    // Foreground: Android shows nothing by itself → display locally.
    FirebaseMessaging.onMessage.listen((m) {
      if (Platform.isAndroid) showLocal(m.data);
    });

    // Taps: background (system-shown) and cold start (system-shown + local).
    FirebaseMessaging.onMessageOpenedApp.listen((m) => _setRoute(AppRoute.fromData(m.data)));
    final initial = await _fcm.getInitialMessage();
    if (initial != null) _setRoute(AppRoute.fromData(initial.data));
    final launch = await _local.getNotificationAppLaunchDetails();
    if (launch?.didNotificationLaunchApp ?? false) _routeFromPayload(launch!.notificationResponse?.payload);

    _fcm.onTokenRefresh.listen((t) { _token = t; _upload(); });
    await refresh();
  }

  /// Ask in context (after the user enables chat/reminders).
  Future<void> requestPermission() async {
    status = (await _fcm.requestPermission(alert: true, badge: true, sound: true)).authorizationStatus;
    notifyListeners();
    await refresh();
  }

  /// Call on start and resume.
  Future<void> refresh() async {
    status = (await _fcm.getNotificationSettings()).authorizationStatus;
    if (Platform.isIOS && await _fcm.getAPNSToken() == null) return;   // FCM token needs APNs token first
    _token = await _fcm.getToken();
    await _upload();
    notifyListeners();
  }

  Future<void> setUser(String? id) async {
    final previous = userId;
    userId = id;
    final t = _token;
    if (t != null && previous != null && previous != id) await _backend.unlink(token: t, userId: previous);
    await _upload();
  }

  Future<void> _upload() async {
    final t = _token;
    if (t == null) return;
    await _backend.upload(token: t, platform: Platform.isIOS ? 'ios' : 'android', userId: userId,
        enabled: status == AuthorizationStatus.authorized || status == AuthorizationStatus.provisional);
  }

  void _routeFromPayload(String? payload) {
    if (payload == null) return;
    _setRoute(AppRoute.fromData(jsonDecode(payload) as Map<String, dynamic>));
  }

  void _setRoute(AppRoute route) { pendingRoute = route; notifyListeners(); }
  void consumeRoute() { pendingRoute = null; }
}
```

In your router (go_router `redirect` / a listener on `PushService`), navigate to `pendingRoute` only after sign-in completes, then call `consumeRoute()`.

## Appendix B — Server: per-platform FCM payloads (Node/TypeScript)

```ts
import { getMessaging, Message } from "firebase-admin/messaging";

type Device = { token: string; platform: "ios" | "android" };
type Push = { type: string; title: string; body: string; data: Record<string, string>;
              collapseId?: string; threadId?: string; category?: string };

export function buildMessage(d: Device, p: Push): Message {
  const data = { type: p.type, title: p.title, body: p.body, ...p.data };
  if (d.platform === "android") {
    // Data-only + high priority: the app displays it in every state with channels/styles/actions.
    return { token: d.token, data, android: { priority: "high", collapseKey: p.collapseId, ttl: 86_400_000 } };
  }
  // iOS: alert push (data-only would be a silent push). NSE handles images/decryption.
  return {
    token: d.token,
    notification: { title: p.title, body: p.body },
    data,
    apns: {
      headers: { "apns-push-type": "alert", "apns-priority": "10",
                 ...(p.collapseId && { "apns-collapse-id": p.collapseId }) },
      payload: { aps: { sound: "default", "mutable-content": 1,
                        ...(p.threadId && { "thread-id": p.threadId }),
                        ...(p.category && { category: p.category }) } },
    },
  };
}

export async function send(devices: Device[], p: Push) {
  const res = await getMessaging().sendEach(devices.map((d) => buildMessage(d, p)));
  await Promise.all(res.responses.map((r, i) =>
    r.error?.code === "messaging/registration-token-not-registered" ? db.pushTokens.delete(devices[i].token) : null));
}
```
