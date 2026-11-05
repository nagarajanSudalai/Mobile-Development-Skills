---
name: cross-platform-push-notifications
description: Use when implementing, reviewing, testing, or debugging push and local notifications in Flutter (firebase_messaging + flutter_local_notifications), React Native (@react-native-firebase/messaging + Notifee), or Expo (expo-notifications + Expo push service) apps — permission on iOS and Android 13+, FCM/APNs/Expo tokens, per-platform message design (data vs notification), foreground/background/terminated handling, background handlers, tap routing including cold start, channels and categories, native pieces (iOS Notification Service Extension, Android icons), scheduling, server sending, testing. Native siblings: ios-push-notifications, android-push-notifications.
---

# Cross-Platform Push Notifications (Flutter, React Native, Expo)

Guide for building, auditing, and testing notifications in **Flutter**, **React Native**, and **Expo** apps that target iOS and Android. One Dart/TypeScript layer has to cope with two notification systems that behave differently — especially for **data vs notification messages**, **foreground display**, **background handlers**, and **taps that launch the app**.

Platform skills hold the native detail — this skill links to them instead of repeating it:
- **ios-push-notifications** — APNs, permission options, payload/headers, Service Extension, communication notifications, App Review
- **android-push-notifications** — FCM message types, POST_NOTIFICATIONS, channels, PendingIntents, styles, Doze, Play policy

Library APIs change between majors (`firebase_messaging`, `flutter_local_notifications`, `@react-native-firebase/messaging`, `@notifee/react-native`, `expo-notifications` all had breaking changes recently). Verify every call against the versions in `pubspec.yaml` / `package.json`.

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
   | firebase_messaging            | @react-native-firebase   | expo-notifications
   | + flutter_local_notifications | + @notifee/react-native  |
   v                               v                          v
+--------------------------+   +-----------------------------------+
| iOS: APNs (via FCM/Expo) |   | Android: FCM                      |
| + native NSE target      |   | + channels, small icon, meta-data |
+--------------------------+   +-----------------------------------+
            ^                               ^
+-------------------------------------------------------------+
| Backend: token store per platform; sends via FCM HTTP v1    |
| (both platforms) or Expo push service, building a           |
| platform-specific payload                                   |
+-------------------------------------------------------------+
```

How the same push behaves per platform (FCM):

| Message | iOS | Android |
|---|---|---|
| **notification** (+ data) | Shown by the system in background; in foreground only if presentation options allow | Shown by the system in background (your code doesn't run); **not shown in foreground** unless you display it |
| **data-only** | Becomes a **silent push** (`content-available`) — throttled, not delivered after force-quit, **unreliable** | Delivered to your handler in all states (unless force-stopped) — you display it |
| Taps on system-shown notifications | `getInitialMessage` / `onMessageOpenedApp` (or equivalents) | Same |
| Taps on locally displayed notifications | Local-notification library callbacks | Same |

Practical rule: **iOS → alert (notification) push with `mutable-content` + data; Android → data-only high-priority push displayed by your local-notification layer.** The server builds a different payload per platform (Appendix D).

## 1. First, establish context

Find out (from the codebase or by asking):

- Framework and libraries with **exact versions**: Flutter `firebase_messaging` (+ `flutter_local_notifications`), React Native `@react-native-firebase/messaging` (+ `@notifee/react-native`), Expo `expo-notifications` (managed or bare; Expo push service or direct FCM/APNs)
- Push provider: FCM for both platforms, Expo push service, APNs direct for iOS + FCM for Android, or a vendor (OneSignal, Braze…) with its own SDK
- Notification catalog (type → iOS interruption level/category → Android channel/importance → route), as in the native skills
- Rich needs: images, actions/replies, conversation style, E2E encryption → these need **native code** (iOS Service Extension; Android style support in the display library)
- Account model, logout behaviour, multi-account
- Navigation library (go_router / auto_route, React Navigation / expo-router) — for deep-link routing

## 2. Setup

### 2.1 Flutter
- [ ] `flutterfire configure` → `firebase_options.dart`; `Firebase.initializeApp` before anything else
- [ ] iOS: Push Notifications + Background Modes (Remote notifications) capabilities in `ios/Runner`; **APNs Auth Key uploaded to Firebase**
- [ ] Android: `google-services.json` per flavor; default icon/channel meta-data (android-push-notifications §2); `POST_NOTIFICATIONS` in the manifest
- [ ] `flutter_local_notifications` initialized with a **monochrome** Android small icon (`@drawable/ic_notification`) and Darwin settings with permission requests **off** (you request them in context)

### 2.2 React Native (bare)
- [ ] `@react-native-firebase/app` + `messaging`; iOS `GoogleService-Info.plist`, capabilities, APNs key in Firebase; Android `google-services.json`
- [ ] `@notifee/react-native` for display, channels, actions, foreground events
- [ ] Background handlers registered in **`index.js` before `AppRegistry.registerComponent`**

### 2.3 Expo
- [ ] `expo-notifications` via `npx expo install`; config plugin (`icon`, `color`, sounds) in `app.json`
- [ ] **Development build** — push notifications don't work in Expo Go on Android (recent SDKs), and Expo Go can't test your production config on iOS either
- [ ] Credentials in EAS: FCM V1 service account (Android) and APNs key (iOS)
- [ ] Expo push token needs the EAS `projectId`

### 2.4 Native pieces you still need
- [ ] **iOS Notification Service Extension** (Swift target in `ios/`) for images, decryption, communication notifications — ios-push-notifications §8. Flutter/RN/Expo can't run Dart/JS there. Expo: a config plugin / prebuild template for the extension
- [ ] Android small icon and colour resources; channels created at startup (via the display library)
- [ ] App Group / keychain sharing if the NSE needs credentials

## 3. Permission flow

- [ ] Ask **in context**, once per platform rules — not at first launch. Same pre-prompt UI for both platforms
- [ ] iOS: `requestPermission(alert, badge, sound[, provisional])`; provisional for low-urgency content
- [ ] Android 13+: the request maps to `POST_NOTIFICATIONS` (`FirebaseMessaging.requestPermission` / `notifee.requestPermission` / `Notifications.requestPermissionsAsync` handle it; plain RN can use `PermissionsAndroid`). Android ≤ 12: granted by default
- [ ] Expo/Android: create the notification **channel before requesting the token/permission** (Android 13 shows the prompt when the first channel is created if you don't ask first)
- [ ] Denied: in-app explanation + open app notification settings (`openSettings` / `Linking.openSettings()` / `AppSettings`); never nag
- [ ] Read settings on start/resume (users change them in system settings); send `enabled` to the backend
- [ ] App works without notifications; marketing only with opt-in

## 4. Tokens

- [ ] **FCM**: `getToken()` on both platforms; `onTokenRefresh` → upload. iOS: the FCM token needs the APNs token first — wait for/handle `getAPNSToken()` being null shortly after launch, don't crash
- [ ] **Expo**: `getExpoPushTokenAsync({ projectId })` for the Expo push service, or `getDevicePushTokenAsync()` for direct FCM/APNs — store which kind you have
- [ ] Upload with: `platform` (`ios`/`android`), token kind, user ID, app version, locale, time zone, enabled state, APNs environment if sending to APNs directly
- [ ] Upload on start/resume if anything changed; tolerate duplicates server-side
- [ ] **Logout**: unlink token from the user (or `deleteToken()` and fetch a new one)
- [ ] Server deletes invalid tokens (FCM `UNREGISTERED`; Expo receipt `DeviceNotRegistered`; APNs 410)

## 5. Receiving — three app states × two platforms

| State | Flutter | React Native (RNFirebase + Notifee) | Expo |
|---|---|---|---|
| Foreground | `FirebaseMessaging.onMessage` → Android: show via `flutter_local_notifications`; iOS: `setForegroundNotificationPresentationOptions` or show locally | `messaging().onMessage` → `notifee.displayNotification` | `setNotificationHandler` decides banner/list/sound; `addNotificationReceivedListener` |
| Background / terminated (data) | `FirebaseMessaging.onBackgroundMessage(topLevelFn)` — **separate isolate**, `@pragma('vm:entry-point')`, re-init Firebase | `messaging().setBackgroundMessageHandler` in `index.js` (Headless JS on Android) → Notifee display | Background task via `expo-task-manager` + `Notifications.registerTaskAsync` |
| Tap, app in background | `onMessageOpenedApp` (system-shown) / local-notifications `onDidReceiveNotificationResponse` | `onNotificationOpenedApp` (system-shown) / `notifee.onForegroundEvent` PRESS | `addNotificationResponseReceivedListener` |
| Tap, app **terminated** (cold start) | `getInitialMessage()` + `getNotificationAppLaunchDetails()` | `messaging().getInitialNotification()` + `notifee.getInitialNotification()` | `getLastNotificationResponseAsync()` / `useLastNotificationResponse()` |
| Action / reply in background | `onDidReceiveBackgroundNotificationResponse` (top-level, isolate) | `notifee.onBackgroundEvent` in `index.js` | Category actions → response listener / background task |

Checks:
- [ ] Every row handled — the **cold-start row** is the one most often missing
- [ ] Both sources handled when you mix system-shown (notification messages) and locally displayed notifications — they use different tap callbacks
- [ ] Background handlers are **top-level / registered before the app component**, do minimal work, and don't touch UI state (isolate / headless context)
- [ ] Taps produce a **pending route**; the router consumes it only when sign-in and the root navigator are ready
- [ ] Foreground suppression for the screen the user is already on
- [ ] Unknown types/missing IDs → home, no crash

## 6. Display layer (Android mainly, iOS for local notifications)

- [ ] Channels created at startup with the catalog's importance (android-push-notifications §4) — `flutter_local_notifications` `createNotificationChannel`, `notifee.createChannel`, `Notifications.setNotificationChannelAsync`
- [ ] Stable notification IDs per entity (conversation/order) → updates replace; cancel when read elsewhere
- [ ] Grouping (Android `groupKey` + summary; iOS `threadIdentifier`)
- [ ] Actions: iOS categories registered at startup (`DarwinNotificationCategory`, `notifee.setNotificationCategories`, `setNotificationCategoryAsync`); Android actions per notification; text-input reply where supported — update the notification after replying
- [ ] Styles: big picture / big text / messaging style where the library supports them; images on iOS come from the **NSE**, not from Dart/JS
- [ ] Payload passed to the local notification (`payload` / `data`) contains the route data
- [ ] Sensitive content hidden on the lock screen (Android visibility private; iOS previews) or encrypted + NSE

## 7. Local notifications & scheduling

- [ ] Time zones: `flutter_local_notifications.zonedSchedule` with the `timezone` package; Notifee timestamp triggers; Expo date/calendar triggers
- [ ] Android exact alarms need permission on 12+/14+ (`SCHEDULE_EXACT_ALARM`) — prefer inexact unless essential; reschedule after reboot (libraries offer boot receivers — enable them)
- [ ] iOS: max **64 pending** — schedule the nearest and top up
- [ ] Re-schedule after time-zone change, logout/login, settings change

## 8. Server

- [ ] One token table: `{ token, kind: fcm|expo|apns, platform, userId, enabled, locale, appVersion, lastSeen }`
- [ ] **Per-platform payloads** (Appendix D): iOS → `notification` + `apns` block (`mutable-content`, `thread-id`, `category`, `interruption-level`, `apns-collapse-id`); Android → data-only with `android.priority: high` (+ `collapse_key`, `ttl`)
- [ ] FCM HTTP v1 / firebase-admin `sendEach` (≤ 500 per call); legacy FCM API is gone
- [ ] Expo push service: batches ≤ 100, then **check receipts** (`/getReceipts`) and delete `DeviceNotRegistered` tokens; handle `MessageRateExceeded` with backoff
- [ ] Localize per user locale server-side (or use `loc-key` on iOS + strings on Android)
- [ ] Respect per-user preferences and `enabled` state; no marketing without opt-in

## 9. Policies & privacy

- [ ] Both stores: app usable without notifications; marketing opt-in + opt-out; no sensitive data in visible text by default (ios-push-notifications §13, android-push-notifications §11)
- [ ] Privacy manifest (iOS) and Data safety (Android) declare push identifiers tied to users/analytics

## 10. Testing plan

1. **iOS simulator**: `xcrun simctl push` with an `.apns` file (alert pushes; real FCM tokens on Apple-silicon simulators with recent Xcode)
2. **Android emulator with Google Play image**: Firebase console test message / HTTP v1 curl
3. **Expo**: Expo push tool (expo.dev/notifications) with an Expo push token from a **development build**
4. **Physical devices** for both platforms, including a killed-app cold start and Android Doze (`adb shell dumpsys deviceidle force-idle`)
5. Unit tests: route parsing and payload → route mapping in pure Dart/TS

### Scenario matrix

| # | Scenario | iOS | Android |
|---|---|---|---|
| X1 | First launch | No prompt; token uploaded when available | No prompt (13+); token uploaded |
| X2 | Contextual permission → allow / deny | Status reflected; settings link on deny | Same (POST_NOTIFICATIONS) |
| X3 | Push, app foreground | Shown per presentation options (or suppressed on screen) | Displayed by local layer (or suppressed) |
| X4 | Push, app background | System alert | Data message displayed by local layer |
| X5 | Push, app killed | System alert | Background handler displays (not force-stopped) |
| X6 | Tap, app background | Routed | Routed |
| X7 | Tap, app killed (cold start) | Routed after app ready | Routed after app ready |
| X8 | Action / reply from notification | Handled; notification updated | Same |
| X9 | Image push | Via NSE; text fallback on timeout | Big picture via local layer |
| X10 | Data-only push sent to iOS (bug check) | Not relied on (silent) | — |
| X11 | Logout → login as another user | No pushes for previous user | Same |
| X12 | Reinstall | New token; old one cleaned up server-side | Same |
| X13 | Local reminder after time-zone change / reboot | Fires at intended local time | Rescheduled after reboot |
| X14 | Notifications disabled in system settings | Status refreshed on resume; server updated | Same + per-channel blocks |
| X15 | Expo: Expo Go (bug check) | Not used for push testing | Not used (unsupported) |

## 11. Common bugs to look for in review

- Same payload for both platforms: data-only to iOS (silent → unreliable) or notification messages to Android (not shown in foreground; background taps go through a different path)
- Android foreground messages never displayed (no local-notification layer)
- Cold-start tap not handled (`getInitialMessage` / `getInitialNotification` / `getLastNotificationResponseAsync` missing), or handled before the navigator/sign-in is ready
- Mixing system-shown and local notifications but handling taps for only one source
- Flutter background handler not top-level / missing `@pragma('vm:entry-point')` / Firebase not initialized in the isolate; touching UI state from it
- RN background handlers registered inside a component instead of `index.js`
- Expo: channel created after requesting the token on Android 13; Expo Go used for push testing; `projectId` missing for the Expo token
- iOS FCM token requested before the APNs token exists (crash / null); APNs key not uploaded to Firebase/EAS
- Permission requested on first launch; no settings link after denial
- No iOS Service Extension but images/decryption expected; trying to run Dart/JS in the NSE
- Full-colour Android small icon (grey square)
- Tokens not unlinked on logout; invalid tokens/receipts never cleaned up
- Notification IDs random → duplicates instead of updates; no grouping
- Library upgraded across a major without migrating (Notifee/expo-notifications/flutter_local_notifications API changes)

## 12. Output format

When asked to **review/audit**:
1. **Summary** — framework, libraries + versions, provider (FCM / Expo / APNs+FCM / vendor), catalog, native extensions present, per-platform payload strategy
2. **Checklist results** — sections 2–9 with ✅ / ❌ / ⚠️ and file:line, split **Shared / iOS / Android**
3. **Critical issues first** (missed deliveries from payload choice, foreground not shown on Android, cold-start routing, wrong-user delivery, data exposure, policy risks) with fix code
4. **Testing gaps** — X1–X15 not covered, per platform

When asked to **implement**: start from the Appendix for the user's framework, adapt catalog and routes, add the native pieces (NSE, icons), then list Firebase/EAS, server, and store-policy steps.

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

## Appendix B — React Native (`@react-native-firebase/messaging` + `@notifee/react-native`)

### B.1 `index.js` — background handlers before the app registers

```js
import { AppRegistry } from "react-native";
import messaging from "@react-native-firebase/messaging";
import notifee, { EventType } from "@notifee/react-native";
import App from "./App";
import { displayFromData, savePendingRoute, handleReply } from "./src/push/display";

// Android data messages while backgrounded/killed (Headless JS). iOS data-only = silent push: don't rely on it.
messaging().setBackgroundMessageHandler(async (message) => {
  if (!message.notification) await displayFromData(message.data);
});

// Notifee taps/actions while backgrounded.
notifee.onBackgroundEvent(async ({ type, detail }) => {
  if (type === EventType.ACTION_PRESS && detail.pressAction?.id === "reply") await handleReply(detail);
  if (type === EventType.PRESS) await savePendingRoute(detail.notification?.data);
});

AppRegistry.registerComponent("App", () => App);
```

### B.2 Display & service

```ts
// src/push/display.ts
import notifee, { AndroidImportance, AndroidVisibility, EventType } from "@notifee/react-native";
import messaging from "@react-native-firebase/messaging";
import { Platform } from "react-native";

export async function setupChannelsAndCategories() {
  await notifee.createChannel({ id: "messages", name: "Messages", importance: AndroidImportance.HIGH });
  await notifee.createChannel({ id: "orders", name: "Orders", importance: AndroidImportance.DEFAULT });
  await notifee.setNotificationCategories([
    { id: "MESSAGE", actions: [{ id: "reply", title: "Reply", input: { placeholderText: "Message" } }] },
  ]);
}

export async function displayFromData(data: Record<string, any> = {}) {
  await notifee.displayNotification({
    id: data.conversationId ?? data.orderId ?? data.type,           // stable → replaces
    title: data.title,
    body: data.body,
    data,
    android: {
      channelId: data.type === "new_message" ? "messages" : "orders",
      smallIcon: "ic_notification",                                 // monochrome drawable
      pressAction: { id: "default" },
      groupId: data.type,
      visibility: AndroidVisibility.PRIVATE,
      actions: data.type === "new_message"
        ? [{ title: "Reply", pressAction: { id: "reply" }, input: { placeholder: "Message" } }] : [],
    },
    ios: { categoryId: data.type === "new_message" ? "MESSAGE" : undefined, threadId: data.conversationId },
  });
}

// src/push/PushService.ts
type Route = { name: "Home" } | { name: "Chat"; id: string } | { name: "Order"; id: string };
export const routeFrom = (d?: Record<string, any>): Route =>
  d?.type === "new_message" && d.conversationId ? { name: "Chat", id: d.conversationId }
  : d?.type === "order_shipped" && d.orderId ? { name: "Order", id: d.orderId }
  : { name: "Home" };

export async function initPush(onRoute: (r: Route) => void, upload: (token: string) => Promise<void>) {
  await setupChannelsAndCategories();

  // Foreground: display locally on Android; iOS shows system alerts per presentation settings.
  messaging().onMessage(async (m) => {
    if (Platform.OS === "android" || !m.notification) await displayFromData(m.data);
  });

  // Taps — system-shown (RNFirebase) and local (Notifee), background and cold start.
  messaging().onNotificationOpenedApp((m) => onRoute(routeFrom(m.data)));
  notifee.onForegroundEvent(({ type, detail }) => {
    if (type === EventType.PRESS) onRoute(routeFrom(detail.notification?.data));
  });
  const initialRemote = await messaging().getInitialNotification();
  if (initialRemote) onRoute(routeFrom(initialRemote.data));
  const initialLocal = await notifee.getInitialNotification();
  if (initialLocal) onRoute(routeFrom(initialLocal.notification.data));

  messaging().onTokenRefresh(upload);
  await upload(await messaging().getToken());
}

/** In context — handles iOS and Android 13+ POST_NOTIFICATIONS. */
export const requestPushPermission = () => notifee.requestPermission();
```

`onRoute` should store the route; navigate with React Navigation's `navigationRef` only once `isReady()` and the user is signed in. `handleReply` / `savePendingRoute` are app code (reply via API, then update the notification).

## Appendix C — Expo (`expo-notifications`)

```ts
import * as Notifications from "expo-notifications";
import * as Device from "expo-device";
import Constants from "expo-constants";
import { Platform } from "react-native";
import { router } from "expo-router";

// Foreground presentation (field names changed across SDKs — verify for yours).
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowBanner: true, shouldShowList: true, shouldPlaySound: true, shouldSetBadge: false,
  }),
});

export async function initExpoPush(upload: (token: string) => Promise<void>) {
  if (Platform.OS === "android") {
    // Create channels BEFORE asking for permission/token on Android 13+.
    await Notifications.setNotificationChannelAsync("messages", {
      name: "Messages", importance: Notifications.AndroidImportance.HIGH,
    });
  }
  await Notifications.setNotificationCategoryAsync("MESSAGE", [
    { identifier: "reply", buttonTitle: "Reply", textInput: { submitButtonTitle: "Send", placeholder: "Message" } },
  ]);

  // Cold start + background taps.
  const last = await Notifications.getLastNotificationResponseAsync();
  if (last) openFrom(last.notification.request.content.data);
  Notifications.addNotificationResponseReceivedListener((r) => {
    if (r.actionIdentifier === "reply") return void sendReply(r);   // app code
    openFrom(r.notification.request.content.data);
  });

  const { status } = await Notifications.getPermissionsAsync();
  if (status === "granted") await registerToken(upload);
}

/** Call in context. */
export async function requestExpoPermission(upload: (token: string) => Promise<void>) {
  const { status } = await Notifications.requestPermissionsAsync();
  if (status === "granted") await registerToken(upload);
  return status;
}

async function registerToken(upload: (token: string) => Promise<void>) {
  if (!Device.isDevice) return;                                     // simulators/emulators: no Expo push token
  const projectId = Constants.expoConfig?.extra?.eas?.projectId ?? Constants.easConfig?.projectId;
  const { data } = await Notifications.getExpoPushTokenAsync({ projectId });
  await upload(data);                                               // "ExponentPushToken[...]"
}

function openFrom(data: Record<string, any> = {}) {
  // expo-router: queue until the root layout is mounted and the user is signed in.
  if (data.type === "new_message" && data.conversationId) router.push(`/chat/${data.conversationId}`);
  else if (data.type === "order_shipped" && data.orderId) router.push(`/orders/${data.orderId}`);
}
```

Expo push service (server): POST batches (≤ 100) to `https://exp.host/--/api/v2/push/send` with `{ to, title, body, data, channelId, categoryId, mutableContent, priority: "high", sound: "default" }`; later call `/push/getReceipts` with the ticket IDs and delete tokens whose receipt error is `DeviceNotRegistered`.

## Appendix D — Server: per-platform FCM payloads (Node/TypeScript)

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
