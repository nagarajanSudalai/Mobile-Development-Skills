---
name: android-push-notifications
description: Use when implementing, reviewing, testing, or debugging push and local notifications on Android — Firebase Cloud Messaging (FCM HTTP v1), notification vs data messages, POST_NOTIFICATIONS runtime permission (Android 13+), notification channels, FirebaseMessagingService (onNewToken/onMessageReceived), token lifecycle, PendingIntents and deep links, actions and direct reply, MessagingStyle/conversations, grouping, Doze/App Standby and priority, exact alarms, server sending with firebase-admin, testing with adb. iOS sibling: ios-push-notifications.
---

# Android Push Notifications Skill

Guide for building, auditing, and testing notifications on Android: **remote pushes via Firebase Cloud Messaging (FCM)**, **local notifications**, the **runtime permission** (Android 13+), **notification channels**, **message types and priority**, **handling taps and actions**, **rich styles**, background limits, and the **server**. Applies whether you send from your own server, the Firebase console, or a vendor (OneSignal, Braze…) that uses FCM underneath.

Sibling skill: **ios-push-notifications** — same structure for APNs. Android behaviour changes with each API level (permission in 13, PendingIntent mutability in 12, full-screen intent limits in 14…). Verify against your `targetSdk` and the current Android/FCM docs.

## 0. Architecture (mental model)

```
+--------------------+  OAuth2 (service account)   +-------------------------+
| Provider server    | --------------------------> | FCM HTTP v1             |
| firebase-admin /   |  POST /v1/projects/{id}/    | fcm.googleapis.com      |
| HTTP v1            |       messages:send          +-------------------------+
| - token store      | <-- UNREGISTERED / errors --          |
+--------------------+                                       v  Google Play services
        ^ FCM token + user                        +-------------------------------+
        |                                         | Device                        |
+-----------------------------------------------+ | - notification msg, app in bg |
| App                                           | |   → system tray (no app code) |
| - FirebaseMessagingService                    | | - data msg / foreground       |
|   onNewToken → upload                         | |   → onMessageReceived         |
|   onMessageReceived → build NotificationCompat| +-------------------------------+
| - NotificationChannels (created at startup)   |
| - POST_NOTIFICATIONS permission (API 33+)     |
| - PendingIntent → Activity → deep link route  |
+-----------------------------------------------+
```

FCM message types — the most common source of bugs:

| Message | App in **foreground** | App in **background / killed** | Use when |
|---|---|---|---|
| **Notification** message (`notification` block) | `onMessageReceived` — **nothing is shown** unless you post it | **System shows it**; your code doesn't run; tap opens the launcher activity with `data` extras | Simple alerts, marketing via console |
| **Data** message (`data` only) | `onMessageReceived` | `onMessageReceived` (if not force-stopped; subject to priority/Doze) | Full control: custom styles, actions, decryption, suppression |
| Notification + data | Like notification message; `data` in `onMessageReceived` (fg) or intent extras (bg tap) | | Mixed — usually avoid for apps needing consistent handling |

Most production apps send **data messages with high priority** and build every notification themselves, so foreground/background behave the same.

## 1. First, establish context

Find out (from the codebase or by asking):

- `minSdk`, `targetSdk` (permission needed from 33; trampolines blocked from 31; full-screen intent limits from 34)
- **Who sends**: own server (firebase-admin / HTTP v1), Firebase console, vendor SDK — and which message type it sends
- Devices without Google Play services (Huawei/HMS, some markets, AOSP) — need HMS Push Kit or another channel?
- Notification types the product needs: messages/chat, transactional, reminders, marketing, calls, silent sync
- Rich needs: images, conversation style, direct reply, progress, grouping, custom sounds
- Account model: tokens per user; multi-account; logout behaviour
- Navigation: single-activity Compose (Navigation deep links) or multiple activities

Write a **notification catalog** before coding:

```
type            message   channel (importance)       style            tap → route        actions
new_message     data      messages (HIGH)            MessagingStyle   /chat/{id}         Reply, Mark read
order_shipped   data      orders (DEFAULT)           BigText          /orders/{id}       Track
payment_due     data      reminders (HIGH)           Default          /billing           —
weekly_digest   data      marketing (LOW)            BigPicture       /home              —   (opt-in only)
sync            data      — (no UI)                  —                —                  —
```

## 2. Setup

- [ ] Firebase project; `google-services.json` for **each** applicationId/flavor; `com.google.gms.google-services` plugin; `firebase-messaging` dependency (BoM)
- [ ] `FirebaseMessagingService` subclass declared in the manifest with the `com.google.firebase.MESSAGING_EVENT` intent filter (`exported="false"`)
- [ ] Manifest meta-data for system-displayed notification messages: `default_notification_icon` (monochrome, transparent background), `default_notification_color`, `default_notification_channel_id`
- [ ] `POST_NOTIFICATIONS` permission declared (targetSdk 33+)
- [ ] Service account for the server with the Firebase Cloud Messaging permission; key in a secret store — never in the app or repo
- [ ] Auto-init: disable (`firebase_messaging_auto_init_enabled=false`) if you need consent before generating a token, then `setAutoInitEnabled(true)` after consent

## 3. Permission flow (Android 13+)

- [ ] On API 33+, notifications are **off until the user grants `POST_NOTIFICATIONS`**. Below 33 they're on by default (user can still disable)
- [ ] Ask **in context** (after the user enables chat, an order, reminders), not on first launch; if `shouldShowRequestPermissionRationale` is true, show your explanation first
- [ ] Use the Activity Result API (`RequestPermission`); handle grant and denial. After two denials the system stops showing the dialog — send users to settings (`Settings.ACTION_APP_NOTIFICATION_SETTINGS` + `EXTRA_APP_PACKAGE`)
- [ ] Check `NotificationManagerCompat.areNotificationsEnabled()` **and** per-channel importance (`IMPORTANCE_NONE` = blocked) on start/resume; report to the server so it stops sending blocked types (and marketing to opted-out users)
- [ ] The app works without notifications; no blocking screens
- [ ] Per-topic preferences in your own settings, synced to the server; marketing off by default unless opted in

## 4. Notification channels (API 26+)

- [ ] Create all channels at app startup (`NotificationManagerCompat.createNotificationChannel`) — idempotent; the app can't post to a missing channel on 26+
- [ ] One channel per **user-meaningful category** (messages, orders, reminders, marketing) so users can control them separately; localized names and descriptions
- [ ] Importance chosen carefully: `HIGH` = heads-up (urgent/messages), `DEFAULT` = sound, `LOW` = silent, `MIN` = collapsed. **Importance, sound and vibration can't be changed by the app after creation** — only the user can. To change them, create a new channel ID and delete the old one
- [ ] Channel groups for multi-account apps (one group per account)
- [ ] Custom sounds set on the channel (`setSound` with `AudioAttributes`), not on each notification
- [ ] Deleting channels: only when removing a category entirely (deleted channels are visible in settings as a count)

## 5. Token lifecycle

- [ ] `onNewToken(token)` → upload to the server (with user ID if signed in, app version, locale, notification-enabled state). Also fetch `FirebaseMessaging.getInstance().token` on start and upload if changed — `onNewToken` isn't called for an existing token after an app update
- [ ] Tokens change on reinstall, clear data, restore to a new device, and periodically. Server-side: refresh `lastSeen` on each app start; consider tokens stale after ~2 months of inactivity and expired after 270 days
- [ ] **Logout**: unlink the token from the user on the server; optionally `deleteToken()` and get a fresh one for the next user
- [ ] Server deletes tokens on `UNREGISTERED` (404) and on `INVALID_ARGUMENT` for the token
- [ ] Multiple Firebase projects/senders: tokens are per sender — store the project with the token

## 6. Receiving & building notifications

### 6.1 `onMessageReceived`
- [ ] Parse `remoteMessage.data`; route by `type`; unknown types ignored (no crash)
- [ ] Must finish **quickly** (aim < 10 s; the service is short-lived). Long work (downloading big media, sync) → enqueue **WorkManager** (expedited for user-visible work) and post the notification from there
- [ ] A **high-priority** message must result in a **user-visible notification**; otherwise FCM may deprioritize future high-priority messages to this app
- [ ] Check permission/channel state before posting; `NotificationManagerCompat.notify` throws `SecurityException` without permission on 33+ — guard with `areNotificationsEnabled()`
- [ ] Foreground suppression: if the user is viewing that chat, update the UI instead of notifying
- [ ] Stable notification IDs/tags per entity (e.g. conversation ID) so updates replace instead of stacking; cancel when read elsewhere (silent data message → `cancel(tag, id)`)
- [ ] Decrypt E2E payloads here (keys from Keystore / encrypted storage); never put sensitive content in plain `notification` fields

### 6.2 Building with `NotificationCompat`
- [ ] Small icon: **monochrome with alpha** (white silhouette) — full-colour icons render as a grey square
- [ ] `setContentIntent` with a `PendingIntent.getActivity` (no trampolines), `setAutoCancel(true)`
- [ ] `setCategory` (`CATEGORY_MESSAGE`, `CATEGORY_REMINDER`, `CATEGORY_CALL`…), `setPriority` for API < 26 (match channel importance)
- [ ] Lock screen: `setVisibility(VISIBILITY_PRIVATE)` + `setPublicVersion` for sensitive content
- [ ] Styles: `BigTextStyle`, `BigPictureStyle` (download with a timeout; fall back to text), `InboxStyle`, **`MessagingStyle`** with `Person` for chats, progress (`setProgress`) for uploads
- [ ] **Conversations** (Android 11+): `MessagingStyle` + a long-lived **sharing shortcut** (`ShortcutInfoCompat` with `setLongLived(true)`) + `setShortcutId` → shown in the conversation section, eligible for bubbles and priority conversations
- [ ] Grouping: `setGroup(key)` + a group **summary** notification (`setGroupSummary(true)`) so 4+ notifications bundle predictably
- [ ] Badges: launcher dots are automatic per channel (`setShowBadge`); `setNumber` for count-supporting launchers
- [ ] Timestamps (`setWhen`), `setTimeoutAfter` for time-bound content, `setOnlyAlertOnce` for updates

### 6.3 Taps & deep links
- [ ] `PendingIntent` flags: `FLAG_IMMUTABLE` (required from API 31) — except where the system must fill in data (direct reply `RemoteInput` → `FLAG_MUTABLE`)
- [ ] Unique `requestCode` per notification (or distinct intent data) so PendingIntents don't overwrite each other's extras
- [ ] Single-activity apps: deep-link URI in the intent (`Intent.ACTION_VIEW` with `myapp://chat/42`) handled by Navigation (`navDeepLink`) — or pass extras and route manually
- [ ] Handle the intent in **`onCreate` (cold start)** and **`onNewIntent` (app running, `singleTop`)**; route only after sign-in/initial UI is ready (queue the pending route)
- [ ] Use `TaskStackBuilder` (or Navigation's synthetic back stack) so Back goes to a sensible parent screen
- [ ] **Notification messages tapped in the background** open the launcher activity with `data` keys as extras — read them in the launcher activity too (if you send notification messages at all)
- [ ] Validate route data; unknown → home

### 6.4 Actions & direct reply
- [ ] `NotificationCompat.Action` with a `PendingIntent` to a `BroadcastReceiver` (background work) or an Activity (needs UI)
- [ ] Direct reply: `RemoteInput` + `FLAG_MUTABLE` PendingIntent; in the receiver, send the reply (WorkManager if network), then **update the notification** (append the reply to `MessagingStyle`) — otherwise a spinner stays forever
- [ ] Sensitive actions require unlock (`setAuthenticationRequired(true)`, API 31+)
- [ ] `setSemanticAction` (reply, mark as read) for Wear OS / Android Auto / assistants

## 7. Priority, Doze & delivery reality

- [ ] **High priority** (`android.priority = "high"`) for messages that need immediate delivery (wakes the device from Doze, subject to App Standby bucket quotas); **normal** for everything else (may be delayed until a maintenance window)
- [ ] Don't use high priority for silent data messages that don't show a notification
- [ ] `ttl` set for time-bound content (e.g. 0 or minutes for calls/OTP-like events; days for digests); `collapse_key` to keep only the latest of a kind while offline (max 4 collapse keys per device)
- [ ] **Force-stopped apps** (user tapped Force stop, or some OEM "cleaners") receive nothing until opened again
- [ ] Aggressive OEM battery managers (Xiaomi, Huawei, Oppo, Vivo, Samsung "sleeping apps") can block delivery — document it in support FAQs; don't ask users to disable battery optimization unless your use case qualifies (Play policy)
- [ ] Devices without Google Play services: no FCM — use HMS Push Kit (Huawei) or fall back to in-app polling/WebSocket if they're a target market
- [ ] Never rely on push for critical state — sync on app open too

## 8. Local notifications & scheduling

- [ ] Reminders at a time: **WorkManager** (`OneTimeWorkRequest` with delay) for "around then"; **AlarmManager** for exact times
- [ ] Exact alarms: `SCHEDULE_EXACT_ALARM` (API 31+, user-revocable, denied by default on 14+ for new installs) or `USE_EXACT_ALARM` (only for alarm/calendar apps per Play policy). Check `canScheduleExactAlarms()`; fall back to inexact `setWindow`
- [ ] Reschedule after **reboot** (`RECEIVE_BOOT_COMPLETED`), **time/time-zone change**, app update, and logout/login
- [ ] Same channels, styles and routing as remote notifications

## 9. Calls & full-screen

- [ ] Incoming calls: `NotificationCompat.CallStyle` (API 31+) on a HIGH channel + full-screen intent; integrate with **Telecom** (`ConnectionService` / Jetpack Core-Telecom) for system call UI
- [ ] `USE_FULL_SCREEN_INTENT` is restricted on Android 14+ to calling and alarm apps (Play Console declaration); check `canUseFullScreenIntent()` and fall back to a heads-up notification
- [ ] Data messages with high priority for call invitations; short `ttl`

## 10. Firebase console, topics & vendors

- [ ] Topics (`subscribeToTopic`) for broadcast content (news, sports) — no per-user targeting, no personal data in topic names
- [ ] Console campaigns send **notification messages** — make sure the launcher activity reads `data` extras and the default channel/icon are configured
- [ ] Vendor SDKs (OneSignal, Braze, Airship): only one `FirebaseMessagingService` gets the `MESSAGING_EVENT` — forward messages to the vendor SDK from yours (or vice versa) per vendor docs

## 11. Play policy & privacy

- [ ] Ask for `POST_NOTIFICATIONS` only when needed, with context; the app works without it
- [ ] No ads or promotions for other apps in notifications unless they're part of the app's core function and the user opted in (Play: notifications policy / spam)
- [ ] Marketing pushes only with opt-in and an easy opt-out
- [ ] Sensitive content hidden on the lock screen by default (`VISIBILITY_PRIVATE`), especially health, finance, messages
- [ ] Data safety form declares device/push identifiers and analytics tied to them
- [ ] Full-screen intent and exact alarm permissions declared only when your app category qualifies

## 12. Testing plan

1. **Firebase console** (Messaging → new campaign → test on device with the FCM token) — notification messages only
2. **HTTP v1 / firebase-admin script** for data messages, priorities, TTL, collapse keys
3. **Doze**: `adb shell dumpsys deviceidle force-idle` → send normal vs high priority; `adb shell dumpsys deviceidle unforce` after
4. **App Standby buckets**: `adb shell am set-standby-bucket <package> rare` → check high-priority delivery
5. **Force stop** (`adb shell am force-stop <package>`) → confirm no delivery until reopened (expected) and that the app recovers on open
6. Inspect posted notifications: `adb shell dumpsys notification --noredact`
7. Test on Android 12, 13, 14+ and at least one aggressive-OEM device

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| A1 | First launch on API 33+ | No permission dialog yet; token uploaded |
| A2 | Contextual permission → Allow | Notifications posted; server updated |
| A3 | Deny twice | App usable; "Enable notifications" opens app notification settings |
| A4 | Channel blocked by user | That type not posted; server informed |
| A5 | Data message, app foreground | Posted (or suppressed on the relevant screen) |
| A6 | Data message, app background / killed (not force-stopped) | Posted with correct style and channel |
| A7 | Notification message, app background (if used) | System shows it; tap routes using extras |
| A8 | Tap, app running | `onNewIntent` routes correctly |
| A9 | Tap, cold start | Routed after sign-in/UI ready; Back goes to parent |
| A10 | Two notifications with different data | Each opens its own route (unique PendingIntents) |
| A11 | Direct reply | Reply sent; notification updated (no endless spinner) |
| A12 | Conversation with shortcut | Appears in conversation section; avatar shown |
| A13 | 4+ messages | Grouped with summary |
| A14 | Big picture on slow network | Image or text fallback; no ANR |
| A15 | Read on another device | Notification cancelled via data message |
| A16 | Doze + high priority | Delivered promptly; normal priority deferred |
| A17 | Force-stopped app | No delivery until reopened; then sync recovers state |
| A18 | Logout → login as another user | Previous user's pushes not delivered |
| A19 | Reinstall / clear data | New token uploaded; old token UNREGISTERED and deleted |
| A20 | Reminder after reboot / time-zone change | Rescheduled; fires at intended local time |
| A21 | Exact alarm permission revoked | Falls back to inexact scheduling |
| A22 | Lock screen with sensitive content | Public version shown |
| A23 | Dark mode / small icon | Monochrome icon renders correctly |

## 13. Common bugs to look for in review

- Sending **notification messages** and expecting `onMessageReceived` in the background (or expecting them to show in the foreground)
- No `POST_NOTIFICATIONS` request on targetSdk 33+ (nothing shows on new installs), or requesting it on first launch with no context
- Posting to a channel that doesn't exist; trying to change channel importance/sound in code after creation
- Full-colour small icon (grey square); missing default icon/channel meta-data for notification messages
- `onNewToken` the only upload path (existing tokens never sent after a server migration/app update); token never unlinked on logout
- Tokens not deleted on `UNREGISTERED`; using the legacy FCM API (shut down) or server key
- `PendingIntent` without `FLAG_IMMUTABLE` (crash on 31+); same `requestCode` for all notifications → every tap opens the last route
- Notification trampolines (tap → BroadcastReceiver/Service → startActivity) blocked on 31+
- Intent handled only in `onCreate` (warm taps ignored) or routed before sign-in/UI is ready
- Long work in `onMessageReceived` (network, big downloads) instead of WorkManager
- High-priority data messages that don't show a notification → deprioritized later
- Direct reply without updating the notification afterwards
- No group summary → inconsistent bundling; random notification IDs → duplicates instead of updates
- Sensitive text visible on the lock screen; PII in topic names or plain payloads
- Exact alarms assumed available on 14+; reminders lost after reboot
- `USE_FULL_SCREEN_INTENT` used by a non-calling/alarm app
- Two `FirebaseMessagingService`s (app + vendor) — only one receives messages
- `google-services.json` for the wrong flavor/applicationId

## 14. Output format

When asked to **review/audit**:
1. **Summary** — sender (own server / console / vendor), message type used, catalog (type → channel → importance → route), min/target SDK
2. **Checklist results** — sections 2–11 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line (server/console items listed for the user to confirm)
3. **Critical issues first** (nothing shown on 13+, missed/incorrect deliveries, wrong-user delivery, crashes from PendingIntent flags, data exposure, policy risks) with fix code
4. **Testing gaps** — A1–A23 not covered

When asked to **implement**: start from the Appendix, adapt the catalog, channels and routes, remove unused parts (calls, alarms, conversations), then list Firebase, server, and Play policy steps.

---

## Appendix — Reference implementation (Kotlin, Compose, firebase-messaging)

### A.1 Catalog, channels & routes

```kotlin
enum class PushType(val wire: String) {
    NEW_MESSAGE("new_message"), ORDER_SHIPPED("order_shipped"),
    PAYMENT_DUE("payment_due"), WEEKLY_DIGEST("weekly_digest");
    companion object { fun from(value: String?) = entries.firstOrNull { it.wire == value } }
}

object Channels {
    const val MESSAGES = "messages"
    const val ORDERS = "orders"
    const val REMINDERS = "reminders"
    const val MARKETING = "marketing"

    /** Idempotent — call from Application.onCreate. Importance can't be changed later; use a new ID. */
    fun createAll(context: Context) {
        val nm = NotificationManagerCompat.from(context)
        listOf(
            NotificationChannelCompat.Builder(MESSAGES, NotificationManagerCompat.IMPORTANCE_HIGH)
                .setName(context.getString(R.string.channel_messages)).build(),
            NotificationChannelCompat.Builder(ORDERS, NotificationManagerCompat.IMPORTANCE_DEFAULT)
                .setName(context.getString(R.string.channel_orders)).build(),
            NotificationChannelCompat.Builder(REMINDERS, NotificationManagerCompat.IMPORTANCE_HIGH)
                .setName(context.getString(R.string.channel_reminders)).build(),
            NotificationChannelCompat.Builder(MARKETING, NotificationManagerCompat.IMPORTANCE_LOW)
                .setName(context.getString(R.string.channel_marketing)).build(),
        ).forEach(nm::createNotificationChannel)
    }
}

sealed interface AppRoute {
    data object Home : AppRoute
    data class Chat(val id: String) : AppRoute
    data class Order(val id: String) : AppRoute
    data object Billing : AppRoute

    fun toUri(): Uri = when (this) {
        Home -> "myapp://home"
        is Chat -> "myapp://chat/$id"
        is Order -> "myapp://orders/$id"
        Billing -> "myapp://billing"
    }.toUri()

    companion object {
        /** Validates payload data; unknown or incomplete → Home. */
        fun from(data: Map<String, String>): AppRoute = when (PushType.from(data["type"])) {
            PushType.NEW_MESSAGE -> data["conversationId"]?.let(::Chat) ?: Home
            PushType.ORDER_SHIPPED -> data["orderId"]?.let(::Order) ?: Home
            PushType.PAYMENT_DUE -> Billing
            PushType.WEEKLY_DIGEST, null -> Home
        }
    }
}
```

### A.2 Messaging service

```kotlin
class AppMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        PushRegistry.get(this).onToken(token)                     // upload (A.4)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        val data = message.data
        when (data["kind"]) {
            "cancel" -> {                                          // read on another device
                data["tag"]?.let { NotificationManagerCompat.from(this).cancel(it, 0) }
                return
            }
            "sync" -> {                                            // silent: no UI, normal priority
                WorkManager.getInstance(this).enqueue(OneTimeWorkRequestBuilder<SyncWorker>().build())
                return
            }
        }
        val route = AppRoute.from(data)
        if (AppForeground.isShowing(route)) return                 // user is on that screen — UI updates itself

        if (data["imageUrl"] != null) {
            // Media download can be slow — do it in expedited work, then post.
            WorkManager.getInstance(this).enqueue(
                OneTimeWorkRequestBuilder<RichNotificationWorker>()
                    .setInputData(workDataOf(*data.map { it.key to it.value }.toTypedArray()))
                    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                    .build())
        } else {
            Notifier(this).post(data, image = null)
        }
    }
}
```

`AppForeground` is an app-specific tracker of the visible route (e.g. updated from your NavController).

### A.3 Notifier — styles, PendingIntents, actions, grouping

```kotlin
class Notifier(private val context: Context) {
    private val nm = NotificationManagerCompat.from(context)

    fun post(data: Map<String, String>, image: Bitmap?) {
        if (!nm.areNotificationsEnabled()) return                   // also avoids SecurityException on 33+
        val type = PushType.from(data["type"]) ?: return
        val route = AppRoute.from(data)
        val tag = data["conversationId"] ?: data["orderId"] ?: type.wire   // stable per entity → updates replace

        val builder = when (type) {
            PushType.NEW_MESSAGE -> messageNotification(data, tag)
            PushType.ORDER_SHIPPED -> NotificationCompat.Builder(context, Channels.ORDERS)
                .setCategory(NotificationCompat.CATEGORY_STATUS)
                .setStyle(NotificationCompat.BigTextStyle().bigText(data["body"]))
                .addAction(R.drawable.ic_track, context.getString(R.string.track), contentIntent(route, tag))
            PushType.PAYMENT_DUE -> NotificationCompat.Builder(context, Channels.REMINDERS)
                .setCategory(NotificationCompat.CATEGORY_REMINDER)
            PushType.WEEKLY_DIGEST -> NotificationCompat.Builder(context, Channels.MARKETING)
                .apply { image?.let { setStyle(NotificationCompat.BigPictureStyle().bigPicture(it)) } }
        }

        val notification = builder
            .setSmallIcon(R.drawable.ic_notification)               // monochrome + alpha
            .setContentTitle(data["title"])
            .setContentText(data["body"])
            .setContentIntent(contentIntent(route, tag))
            .setAutoCancel(true)
            .setVisibility(NotificationCompat.VISIBILITY_PRIVATE)
            .setPublicVersion(publicVersion(type))
            .setGroup(type.wire)
            .setOnlyAlertOnce(true)
            .build()

        nm.notify(tag, 0, notification)
        postGroupSummary(type)
    }

    private fun messageNotification(data: Map<String, String>, conversationId: String): NotificationCompat.Builder {
        val sender = Person.Builder().setName(data["senderName"]).setKey(data["senderId"]).build()
        val me = Person.Builder().setName(context.getString(R.string.you)).build()
        val style = NotificationCompat.MessagingStyle(me)
            .addMessage(data["body"], System.currentTimeMillis(), sender)

        ConversationShortcuts.ensure(context, conversationId, sender)  // long-lived sharing shortcut (Android 11+)

        val replyInput = RemoteInput.Builder(ReplyReceiver.KEY_TEXT).setLabel(context.getString(R.string.reply)).build()
        val replyIntent = PendingIntent.getBroadcast(
            context, conversationId.hashCode(),
            Intent(context, ReplyReceiver::class.java).putExtra(ReplyReceiver.KEY_CONVERSATION, conversationId),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE)   // MUTABLE: system fills RemoteInput
        val reply = NotificationCompat.Action.Builder(R.drawable.ic_reply, context.getString(R.string.reply), replyIntent)
            .addRemoteInput(replyInput)
            .setSemanticAction(NotificationCompat.Action.SEMANTIC_ACTION_REPLY)
            .setShowsUserInterface(false)
            .build()

        return NotificationCompat.Builder(context, Channels.MESSAGES)
            .setCategory(NotificationCompat.CATEGORY_MESSAGE)
            .setStyle(style)
            .setShortcutId(conversationId)
            .addAction(reply)
    }

    /** One PendingIntent per entity (unique requestCode) straight to the Activity — no trampoline. */
    private fun contentIntent(route: AppRoute, tag: String): PendingIntent {
        val intent = Intent(Intent.ACTION_VIEW, route.toUri(), context, MainActivity::class.java)
            .addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_CLEAR_TOP)
        return PendingIntent.getActivity(context, tag.hashCode(), intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
    }

    private fun publicVersion(type: PushType) = NotificationCompat.Builder(context, Channels.MESSAGES)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(context.getString(R.string.app_name))
        .setContentText(context.getString(if (type == PushType.NEW_MESSAGE) R.string.new_message else R.string.new_update))
        .build()

    private fun postGroupSummary(type: PushType) {
        val channel = when (type) {
            PushType.NEW_MESSAGE -> Channels.MESSAGES; PushType.ORDER_SHIPPED -> Channels.ORDERS
            PushType.PAYMENT_DUE -> Channels.REMINDERS; PushType.WEEKLY_DIGEST -> Channels.MARKETING
        }
        nm.notify("summary-${type.wire}", 0, NotificationCompat.Builder(context, channel)
            .setSmallIcon(R.drawable.ic_notification)
            .setGroup(type.wire)
            .setGroupSummary(true)
            .setAutoCancel(true)
            .build())
    }
}

class ReplyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val text = RemoteInput.getResultsFromIntent(intent)?.getCharSequence(KEY_TEXT)?.toString() ?: return
        val conversationId = intent.getStringExtra(KEY_CONVERSATION) ?: return
        // Network → WorkManager; the worker appends the reply to the MessagingStyle notification when done.
        WorkManager.getInstance(context).enqueue(
            OneTimeWorkRequestBuilder<SendReplyWorker>()
                .setInputData(workDataOf("conversationId" to conversationId, "text" to text))
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .build())
    }
    companion object { const val KEY_TEXT = "reply_text"; const val KEY_CONVERSATION = "conversation_id" }
}
```

Declare `ReplyReceiver` in the manifest (`exported="false"`). `ConversationShortcuts`, `SendReplyWorker`, `RichNotificationWorker`, `SyncWorker` are app placeholders — the reply worker must re-post the notification with the reply added (or cancel it).

### A.4 Token registry, permission & routing

```kotlin
interface PushApi {
    suspend fun upload(token: String, userId: String?, enabled: Boolean, blockedChannels: Set<String>)
    suspend fun unlink(token: String, userId: String)
}

class PushRegistry private constructor(private val context: Context, private val api: PushApi,
                                       private val scope: CoroutineScope) {
    private var lastSent: Triple<String, String?, Boolean>? = null
    var userId: String? = null
        set(value) { val old = field; field = value; scope.launch { sync(previousUserId = old) } }

    fun onToken(token: String) = scope.launch { sync(token = token) }

    /** Call on app start and resume — catches token changes, permission and channel changes. */
    fun refresh() = scope.launch { sync() }

    private suspend fun sync(token: String? = null, previousUserId: String? = null) {
        val t = token ?: runCatching { FirebaseMessaging.getInstance().token.await() }.getOrNull() ?: return
        val nm = NotificationManagerCompat.from(context)
        val enabled = nm.areNotificationsEnabled()
        val blocked = nm.notificationChannelsCompat
            .filter { it.importance == NotificationManagerCompat.IMPORTANCE_NONE }.map { it.id }.toSet()
        if (previousUserId != null && previousUserId != userId) runCatching { api.unlink(t, previousUserId) }
        if (lastSent == Triple(t, userId, enabled) && token == null) return
        runCatching { api.upload(t, userId, enabled, blocked) }.onSuccess { lastSent = Triple(t, userId, enabled) }
    }

    companion object {
        @Volatile private var instance: PushRegistry? = null
        fun get(context: Context) = instance ?: error("PushRegistry.init not called")
        fun init(context: Context, api: PushApi, scope: CoroutineScope) {
            instance = PushRegistry(context.applicationContext, api, scope)
        }
    }
}

class MainActivity : ComponentActivity() {
    private val pendingRoute = MutableStateFlow<Uri?>(null)

    private val requestPermission =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { PushRegistry.get(this).refresh() }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        pendingRoute.value = intent?.data                         // cold start from a notification tap
        setContent {
            val nav = rememberNavController()
            val ready by Session.isReady.collectAsStateWithLifecycle()
            val route by pendingRoute.collectAsStateWithLifecycle()
            AppNavHost(nav)                                         // destinations declare navDeepLink { uriPattern = "myapp://chat/{id}" }
            LaunchedEffect(ready, route) {
                val uri = route ?: return@LaunchedEffect
                if (ready) { nav.navigate(uri); pendingRoute.value = null }   // route only when UI/sign-in ready
            }
        }
    }

    override fun onNewIntent(intent: Intent) {                     // tap while running (singleTop)
        super.onNewIntent(intent)
        pendingRoute.value = intent.data
    }

    override fun onResume() {
        super.onResume()
        PushRegistry.get(this).refresh()
    }

    /** Call in context (e.g. after enabling chat), not at first launch. */
    fun askForNotifications(showRationale: () -> Unit) {
        if (Build.VERSION.SDK_INT < 33) return
        when {
            checkSelfPermission(Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED -> Unit
            shouldShowRequestPermissionRationale(Manifest.permission.POST_NOTIFICATIONS) -> showRationale()
            else -> requestPermission.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }

    fun openNotificationSettings() = startActivity(
        Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).putExtra(Settings.EXTRA_APP_PACKAGE, packageName))
}

class App : Application() {
    val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    override fun onCreate() {
        super.onCreate()
        Channels.createAll(this)
        PushRegistry.init(this, PushBackend(), appScope)
        PushRegistry.get(this).refresh()
    }
}
```

Set `android:launchMode="singleTop"` on `MainActivity`. If you also send FCM **notification messages**, read their `data` keys from `intent.extras` in `onCreate` as well (they arrive as extras, not as `intent.data`).

### A.5 Server (Node/TypeScript, firebase-admin)

```ts
import { initializeApp, cert } from "firebase-admin/app";
import { getMessaging, Message } from "firebase-admin/messaging";

initializeApp({ credential: cert(JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT!)) });   // secret store
const messaging = getMessaging();

export async function sendNewMessage(tokens: string[], m: { conversationId: string; senderId: string;
                                                             senderName: string; body: string }) {
  const message = (token: string): Message => ({
    token,
    // Data-only: the app builds the notification in every state (foreground/background).
    data: { type: "new_message", conversationId: m.conversationId, senderId: m.senderId,
            senderName: m.senderName, title: m.senderName, body: m.body },
    android: {
      priority: "high",              // must result in a visible notification
      ttl: 24 * 3600 * 1000,
      collapseKey: `conv-${m.conversationId}`,
    },
  });

  const res = await messaging.sendEach(tokens.map(message));
  await Promise.all(res.responses.map(async (r, i) => {
    const code = r.error?.code;
    if (code === "messaging/registration-token-not-registered" || code === "messaging/invalid-registration-token") {
      await db.pushTokens.delete(tokens[i]);                       // UNREGISTERED / invalid → delete
    }
  }));
}
```

Retry `messaging/internal-error` / `unavailable` / quota errors with exponential backoff; chunk large sends (`sendEach` accepts up to 500 messages per call).

### A.6 Test message (curl, HTTP v1)

```bash
curl -X POST "https://fcm.googleapis.com/v1/projects/$PROJECT_ID/messages:send" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"message":{"token":"'"$FCM_TOKEN"'","data":{"type":"new_message","conversationId":"42","senderName":"Priya","title":"Priya","body":"Shared a file"},"android":{"priority":"high"}}}'
```
