---
name: react-native-forms
description: Use when implementing, reviewing, or debugging forms in React Native and Expo — login, sign-up, profile edit, checkout, multi-step flows. Covers native Alert.alert for submit/server validation errors vs inline field errors with a Reanimated shake, ending editing before every submit (Keyboard.dismiss / ref.blur), a shared submit wrapper, a full-page blocking HUD (Modal or root overlay) that also blocks every back path (React Navigation beforeRemove / usePreventRemove, gestureEnabled:false, headerBackVisible, BackHandler, Expo Router equivalents), react-hook-form + zod validation, server field errors, returnKeyType / onSubmitEditing chaining, textContentType / autoComplete autofill, KeyboardAvoidingView / react-native-keyboard-controller, masks and pickers, unsaved-changes prompts, drafts, accessibility, and RNTL tests. Siblings: ios-forms, android-forms, flutter-forms.
---

# React Native Forms Skill

Guide for building, auditing, and testing **forms** in React Native and Expo consumer apps: login, sign-up, OTP, profile edit, address and checkout, settings editors, and multi-step onboarding. Preferred stack: **react-hook-form** + **zod** (`@hookform/resolvers`) + **react-native-keyboard-controller** + **React Navigation 7 native-stack** (directly or through **Expo Router**). Siblings: **ios-forms**, **android-forms**, **flutter-forms**. Related: **react-native-network-offline-sync** (submit API, retries, offline queue), **react-native-crash-analytics** (form events, submit failures), **react-native-list-ux** (pickers and long option lists), **react-native-deep-links** (forms opened from links, e.g. reset password).

These APIs move fast, so **verify them against the installed versions before writing code**. Things to check: `TextInput` `submitBehavior` replaced `blurOnSubmit` (RN 0.73+). `autoComplete` works on iOS too since RN 0.71. React Navigation 7 added `usePreventRemove`, and native-stack honours `beforeRemove` only for JS-initiated pops unless you also disable native gestures. `@hookform/resolvers` v5 supports zod v4, so match their majors. `react-native-keyboard-controller` 1.x needs `KeyboardProvider` and the New Architecture is fine. Android 16 (targetSdk 36) enforces predictive back, and RN routes it through `BackHandler` in recent versions. Expo SDK 52+ defaults to the New Architecture. Check `package.json` and `app.json` first.

## Non-negotiable rules (apply to EVERY form in the app)

> **R1. End editing first.** Every Done / Save / Submit / Next / Complete tap, keyboard Return/Done (`onSubmitEditing`), and header or toolbar button that acts on the form must **end editing before it calls its function**: blur the focused `TextInput` (`ref.blur()` / `Keyboard.dismiss()`), let `onBlur` run (formatters, RHF `onBlur` validation, masked-value commit), **then** read values and submit. Enforce this with **one shared submit wrapper** (`useFormSubmit`) so no button can bypass it.
>
> **R2. Full-page blocking HUD while any form action runs.** A full-screen overlay (spinner + optional message) that blocks **all** touches underneath (the same button again, other buttons, tab bar, header) **and blocks every back path**: the header back button, iOS swipe-back, Android hardware/system back and predictive back, and swipe-to-dismiss on sheets and modals. Re-entrant submits are ignored. The HUD stays up for a minimum time so it does not flash. It is **always** dismissed on success, error, and cancellation (`finally`). It announces "Saving…" and traps screen-reader focus.
>
> **R3. Submit errors in a native alert, field errors inline.** A failed validation **on submit** shows `Alert.alert` with a title and a short message listing what's wrong (or the first problem plus "and N more"). When the alert is dismissed, focus moves to the first invalid field. Validation **while editing** (on blur, re-validation as the user types) is shown **inline only**: error text under the field and/or a red border, plus a short shake (Reanimated, skipped when `useReducedMotion()` is on), or a tooltip anchored to the field. Never use `Alert.alert` for field-level validation, and never show only inline errors on submit. **Any failure returned from a submit also gets a native alert after the HUD hides, never a banner, toast or snackbar**: server field errors (422) are listed in the alert **and** mapped inline with `setError` (focus the first one on dismiss); business/rule errors (409 "email already registered", "coupon expired") show the server's title + message; other 4xx/5xx, timeouts and offline/network failures show a friendly message with **Retry** (when the request is safe to retry) + Cancel, and Retry re-runs the same wrapper so R1 + R2 apply again; auth expired (401 after refresh failed) alerts, then routes to re-login. Map errors to alert content in **one** place (`alertContent(error)` → title, message, retryable). Success feedback may still be a toast or navigation; only errors must be alerts.

All three rules show up again in the review checklist (§12) and the common bugs (§13). A form that breaks any one fails review.

## 0. Mental model

```
 tap Save / Return(Done) / header Save / Next(server)
        │
        ▼
 useFormSubmit(run) ── inFlight ref? ──yes──► ignore (re-entrant)
        │ no
        ▼
 R1 endEditing(): currentlyFocusedInput()?.blur(); Keyboard.dismiss(); await frame
        │            (onBlur → formatter → field.onChange → RHF value committed)
        ▼
 handleSubmit(valid, invalid) ── invalid ──► R3 Alert.alert(list) → onDismiss setFocus(first)
        │ valid
        ▼
 R2 hud.show("Saving…")  ─► Modal overlay (blocks touches + Android back)
                            usePreventRemove(busy) + gestureEnabled:false
                            + headerBackVisible:false + BackHandler → true
        │
        ▼
 api(values, signal) ──► ok: reset(values) → navigate / toast
        │                 err: (422 → setError inline) + R3 Alert.alert(alertContent(err))
        │                      after HUD hides; Retry → run() again (R1 + R2)
        ▼
 finally: await minDisplay(500ms) → hud.hide() → inFlight = false
```

## 1. First, establish context

Find out from the codebase, or ask:

- **Form type**: login, sign-up, OTP, password reset, profile edit, address, checkout/payment, support request, multi-step onboarding. How many fields? Are any optional?
- **Presentation**: pushed screen, native modal / `formSheet`, `@gorhom/bottom-sheet` or `@expo/ui` BottomSheet, or inline on a tab. This decides which dismiss gestures R2 has to block.
- **Create or edit**: in edit mode, prefill and track dirty state (§8). In create mode, think about drafts.
- **Submit contract**: one endpoint or several, idempotency key, server-side field error shape (`{ errors: { email: ["taken"] } }`), timeout. See **react-native-network-offline-sync**.
- **Offline**: block submit with the R3 offline alert (native alert + Retry), or queue it (then the HUD is replaced by a "Saved, will sync" toast)?
- **Environment**: Expo managed / dev client / bare, RN version, New Architecture, React Navigation vs Expo Router version, RHF / zod / resolvers versions, and whether keyboard-controller is installed.
- **Platforms**: iOS, Android, web (on react-native-web, `Modal` and `BackHandler` behave differently).

Write a **form spec** before coding:

```
form       present    mode    fields                         submit          server errors   offline
Login      push       create  email, password                POST /session   form-level      block
Profile    formSheet  edit    name, phone(mask), dob(picker) PATCH /me       per-field       block
Checkout   push(3)    create  address → card → review        POST /orders    per-field+form  block
```

## 2. R1: end editing before every action

**Why**: `onChangeText` usually updates state on each keystroke, but many fields commit **on blur**: masked inputs that format on blur, `Controller` fields that trim or parse in `onBlur`, a custom date/amount field, and RHF `mode: 'onBlur'` validation. A Save tap while the field is still focused reads the value before that commit, so the stale or unformatted value gets saved. Also, if the keyboard stays up during the HUD it covers it, and it can come back up after navigation.

**APIs**:
- `TextInput.State.currentlyFocusedInput()` returns the focused input's ref (RN 0.68+). Call `.blur()` on it. This fires `onBlur` on that field.
- `Keyboard.dismiss()` blurs the focused input and hides the keyboard. On its own it is enough for most cases, but it does not wait for anything.
- With keyboard-controller, `KeyboardController.dismiss()` returns a Promise in recent 1.x releases (verify), so you can `await` the keyboard hiding.
- After blurring, **wait one frame** (`requestAnimationFrame`) so React has processed the `onBlur` → `field.onChange` state update before `handleSubmit` reads `getValues()`.

Checklist:
1. - [ ] One `endEditing()` helper (blur focused input → `Keyboard.dismiss()` → await frame) used by everything
2. - [ ] One `useFormSubmit` wrapper that runs `endEditing()` before validating; no `onPress={handleSubmit(...)}` anywhere
3. - [ ] The last field's `onSubmitEditing` calls the same wrapper, not the API
4. - [ ] Header Save (`navigation.setOptions({ headerRight })` or Expo Router `<Stack.Screen options>`) calls the same wrapper
5. - [ ] Each multi-step Next calls the wrapper (R1 always; R2 if it hits the server)
6. - [ ] Formatters and parsers run in `onBlur` and write through `field.onChange`, never into local-only state
7. - [ ] Grep check: `handleSubmit(` appears only inside the wrapper

## 3. R2: full-page blocking HUD

### 3.1 Overlay that blocks every touch

Two good options:
- **`Modal`** (`transparent`, `animationType="fade"`, `statusBarTranslucent`, `navigationBarTranslucent` on recent RN). It covers the header, tab bar and native sheets because it is its own native window. On Android, `onRequestClose` gets the hardware back press: make it a no-op so back is swallowed. On iOS a `Modal` is announced as modal to VoiceOver. **Preferred.**
- **Absolute overlay at the app root** (portal / provider above the navigator): `StyleSheet.absoluteFill`, `pointerEvents="auto"`, high `zIndex`. It covers tabs and headers **only if it is rendered above the `NavigationContainer`**. It does **not** cover a native `modal`/`formSheet` screen, which sits in a separate native view controller, so use `Modal` for forms presented as sheets.
- Don't put the overlay inside the screen's own view. The header and tab bar stay tappable.

### 3.2 Block every back path

| Path | How to block (React Navigation 7 native-stack) | Expo Router |
|---|---|---|
| Header back button | `headerBackVisible: false` (and `headerLeft: () => null` on JS stack) while busy | `<Stack.Screen options={{ headerBackVisible: !busy }} />` |
| iOS swipe-back | `gestureEnabled: false` while busy | same option via `<Stack.Screen options>` |
| Sheet / modal swipe-down | `gestureEnabled: false` (native modal/formSheet). For `@gorhom/bottom-sheet`: `enablePanDownToClose={!busy}`, `enableHandlePanningGesture={!busy}` | same |
| JS `goBack()`, deep link replace, header back on JS stack | `usePreventRemove(busy, () => {})`, or `beforeRemove` + `e.preventDefault()` | `usePreventRemove` from `@react-navigation/native` works inside Expo Router screens |
| Android hardware / predictive back | `Modal` `onRequestClose={() => {}}`. Belt and braces: `BackHandler.addEventListener('hardwareBackPress', () => busy)` | same |
| Tab switch | The Modal covers the tab bar | same |

Notes:
- With native-stack, `beforeRemove` **cannot stop a native swipe or a native header back that has already started**. The screen is already gone on the native side, and RN Navigation warns about it. That's why `gestureEnabled: false` and `headerBackVisible: false` are required, not optional.
- Set the options from the busy state with `useLayoutEffect(() => navigation.setOptions(...), [busy])` so there is no frame where they are wrong.
- Predictive back on Android 14+: when JS consumes back, the system shows no peek animation. Test on an Android 14/15/16 device with gesture navigation on.

### 3.3 Re-entrancy, minimum display, guaranteed dismissal

1. - [ ] `inFlightRef` (a ref, not state) checked and set **synchronously** at the top of the wrapper. A state flag alone lets a double tap within one render through.
2. - [ ] The HUD shows immediately (it blocks right away). Once shown it stays at least **~500 ms** (`await Promise.all([work, delay(500)])`, or wait for the remaining time in `finally`)
3. - [ ] `try { … } finally { hud.hide(); inFlightRef.current = false }`. Hide also on cancel (AbortController), timeout, unmount and thrown validation errors
4. - [ ] Request timeout (e.g. 30 s) via `AbortSignal.timeout` or a manual timer, so the HUD can never hang forever
5. - [ ] Reference-count the HUD (or tie it to one owner) so two overlapping actions don't hide it early
6. - [ ] Navigate **after** the HUD hides (or hide right before navigating) so the Modal doesn't linger over the next screen. On iOS, presenting a native modal while an RN `Modal` is dismissing can fail silently

### 3.4 Accessibility of the HUD

- [ ] `AccessibilityInfo.announceForAccessibility('Saving…')` on show and announce the result on hide ("Saved" / the error)
- [ ] `Modal` traps focus on iOS. For a root overlay, set `accessibilityViewIsModal` (iOS) on the overlay and `importantForAccessibility="no-hide-descendants"` on the app content underneath (Android)
- [ ] Spinner: `accessibilityRole="progressbar"`, `accessibilityLabel="Saving"`. Respect reduce-motion: keep a static indicator and drop the fade

## 4. Submit pipeline

```
endEditing → validate all (handleSubmit) → hud.show → api(values, signal)
  → hud.hide → success: reset(values) + navigate/toast + analytics
             → error:   alertContent(err) → Alert.alert(title, message, [Retry?, Cancel/OK])
                        422 → also setError inline + setFocus(first) on dismiss
                        409/business → server message · 5xx/timeout/offline → Retry
                        401 (refresh failed) → alert → re-login
```

- [ ] One `useFormSubmit({ form, submit, onSuccess, mapServerErrors })` coordinator owns R1, R2, the error mapping and analytics events (`form_submit`, `form_submit_failed` with the error code and no PII; see **react-native-crash-analytics**)
- [ ] `reset(savedValues)` on success so `isDirty` is false before navigating (otherwise the unsaved-changes guard fires)
- [ ] Idempotency key per submit attempt for payments and orders
- [ ] On network or offline error: keep every value, show a native alert with Retry, don't clear password fields unless security requires it
- [ ] **R3** every submit failure is an `Alert.alert` from the wrapper (never a banner/toast/snackbar, never while the HUD is visible). Retry calls the same `run()`, so the HUD and re-entrancy guard apply again

## 5. Validation

- `useForm({ resolver: zodResolver(schema), mode: 'onBlur', reValidateMode: 'onChange', shouldFocusError: true })`. Validate on **blur and on submit**, then re-validate as the user types once a field has an error. Not on every keystroke from the start.
- `shouldFocusError` only works if `Controller` passes `field.ref` to the `TextInput` (`ref={field.ref}`) so RHF can call `.focus()`.
- **R3 split**: errors that appear *while editing* (blur, re-validate on change) are inline only: footer text, red border, and a ~250 ms horizontal shake when the error first appears (Reanimated `withSequence` of `withTiming` offsets; skip it if `useReducedMotion()`), or a tooltip anchored to the field. Errors found *on submit*, client or server, also raise one `Alert.alert(title, message)`: "Please check 2 fields", then "• Enter a valid email\n• Enter your name". After the user taps OK, `setFocus` the first invalid field. Server field errors: alert after the HUD hides, plus `setError` so they also show inline. Server business, HTTP, timeout, offline and auth errors: alert only (§4).
- Alert API: React Native `Alert.alert` (siblings: iOS SwiftUI `.alert` / `UIAlertController`; Android Material3 `AlertDialog` / `MaterialAlertDialogBuilder`; Flutter `showAdaptiveDialog` + `AlertDialog.adaptive`).
- Show inline errors under the field (`accessibilityLiveRegion="polite"` on Android, `announceForAccessibility` on iOS) and give the field a red border + icon, not color alone.
- Scroll to the first error: `KeyboardAwareScrollView` scrolls a focused input into view automatically. With a plain `ScrollView`, measure the field (`measureLayout`) and `scrollTo`.
- Server errors: map `{ field: messages[] }` to `setError(field, { type: 'server', message })`, set unknown keys as `setError('root.server', …)`, then `setFocus` the first field in **visual order** (keep a field-order array).
- zod: `.trim()` and `.toLowerCase()` for email, `superRefine` for cross-field checks (password confirmation, date ranges), `z.coerce` only for picker values.

Checklist:
1. - [ ] No error shown before the first blur or submit
2. - [ ] **R3** submit with errors shows a native alert listing them. On dismiss, focus and scroll go to the first invalid field
3. - [ ] **R3** server field errors: an alert after the HUD hides, and inline errors on their fields. Other server/network failures: an alert (Retry when safe). Blur errors are inline and shake (no shake with reduce motion), never an alert
4. - [ ] An error clears as soon as the user fixes the value

## 6. Keyboard

- **Avoidance**: prefer `react-native-keyboard-controller` (`KeyboardProvider` at root, `KeyboardAwareScrollView` with `bottomOffset`, `KeyboardStickyView` for a sticky Save button, optional `KeyboardToolbar` with prev/next/done). Fallback: `KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : undefined}` with `keyboardVerticalOffset={headerHeight}` (`useHeaderHeight()`). On Android with edge-to-edge (SDK 53+ default), `adjustResize` alone may no longer resize, so verify.
- **Chaining**: `returnKeyType="next"` + `onSubmitEditing={() => nextRef.current?.focus()}` + `submitBehavior="submit"` (keeps the keyboard up, no flicker). Last field: `returnKeyType="done"` (or `"go"` / `"send"`) + `onSubmitEditing={submit}` through the wrapper (R1).
- **Tap outside to dismiss**: `ScrollView keyboardShouldPersistTaps="handled"` (so buttons work on the first tap while the keyboard is up) + `keyboardDismissMode="interactive"` (iOS) / `"on-drag"`. Use `Pressable onPress={Keyboard.dismiss}` around non-scrolling forms, with `accessible={false}`.
- **Focused field in view**: `KeyboardAwareScrollView` does it. Otherwise, scroll on focus.

Checklist:
1. - [ ] Every field reachable with the keyboard open on a small phone (iPhone SE, 5" Android)
2. - [ ] Next moves through fields in visual order; Done on the last field submits (via R1)
3. - [ ] First tap on Save with the keyboard open fires (`keyboardShouldPersistTaps="handled"`)
4. - [ ] The sticky button sits above the keyboard and doesn't jump

## 7. Field types and autofill

| Field | keyboardType / inputMode | iOS `textContentType` | `autoComplete` | Other |
|---|---|---|---|---|
| Email | `email-address` | `emailAddress` / `username` on login | `email` / `username` | `autoCapitalize="none"`, `autoCorrect={false}` |
| Password (login) | default | `password` | `current-password` | `secureTextEntry`, show/hide toggle |
| New password | default | `newPassword` + `passwordRules` | `new-password` | strength hint |
| OTP | `number-pad` | `oneTimeCode` | `one-time-code` (verify Android: `sms-otp`) | `maxLength`, auto-submit on full via wrapper |
| Phone | `phone-pad` | `telephoneNumber` | `tel` | mask, store E.164 |
| Name | default | `givenName` / `familyName` / `name` | `name-given` / `name-family` / `name` | `autoCapitalize="words"` |
| Address | default | `fullStreetAddress`, `postalCode`, `addressCity` | `street-address`, `postal-code`, `address-line1` | country picker |
| Card | `number-pad` | `creditCardNumber` | `cc-number`, `cc-exp`, `cc-csc` | prefer the PSP's SDK (Stripe CardField / PaymentSheet) |

- Masks: `react-native-mask-input`, or format in `onChangeText` while keeping the raw value in the form state. Never store the masked string unless the API wants it.
- Dates and choices: native pickers (`@react-native-community/datetimepicker`, `@expo/ui` Picker / DatePicker) or a list sheet (see **react-native-list-ux**). Opening a picker counts as ending text editing: call `endEditing()` first.
- iOS strong-password autofill needs a matching `username` field and Associated Domains (`webcredentials:`). Android autofill needs `autoComplete` set. Test both with a real password manager.

## 8. Unsaved changes

- `formState.isDirty` (compares against `defaultValues`. Call `reset(serverValues)` after loading the edit data).
- `usePreventRemove(isDirty && !busy, ({ data }) => Alert.alert('Discard changes?', …, [{ text: 'Keep editing' }, { text: 'Discard', style: 'destructive', onPress: () => navigation.dispatch(data.action) }]))`.
- **Never show the discard prompt while submitting**: busy-blocking (R2) wins, and success calls `reset()` before navigating.
- iOS native swipe-back can't be intercepted mid-gesture: set `gestureEnabled: !isDirty` on dirty forms (or accept losing data and keep a draft).
- Drafts (optional): debounce `watch()` → MMKV/AsyncStorage keyed by form + user. Restore with a "Restore draft?" banner. Clear on success and logout. Never persist passwords or card data.

## 9. Multi-step forms

- One `useForm` across steps (inside a shared parent or a context/store), with a per-step zod schema and `trigger(stepFields)` on Next. Or one form per step that saves into a store.
- Each Next follows R1. If Next hits the server (check email available, create account, quote a price) it follows R2 too.
- Progress: "Step 2 of 3" text + bar with `accessibilityValue={{ min:1, max:3, now:2 }}`.
- Back between steps keeps the values. The system back on step N goes to step N-1, not out of the flow (handle it with `usePreventRemove`, or make each step its own stack screen).
- Deep links into a later step (see **react-native-deep-links**) must check that the earlier steps are valid and redirect otherwise.

## 10. Accessibility

- [ ] Visible label per field (not placeholder-only). `accessibilityLabel` matches the visible label. Hints for format ("MM/YY")
- [ ] Errors are tied to the field: include the error in `accessibilityHint` or the label, and announce it on submit failure
- [ ] Focus order follows visual order. After a failed submit, focus moves to the first error
- [ ] Font scaling: no fixed heights on inputs, `maxFontSizeMultiplier` only on tight chrome, layout tested at the largest size
- [ ] Touch targets ≥ 44pt / 48dp. Password toggle has a label ("Show password") and state
- [ ] HUD a11y per §3.4

## 11. Testing

### 11.1 Tools

1. **RNTL** + Jest: `fireEvent.changeText`, `fireEvent(input, 'blur')`, `fireEvent(input, 'submitEditing')`, `fireEvent.press`. Mock `Keyboard.dismiss`, and use fake timers for the minimum display
2. Navigation tests: render inside `NavigationContainer` with a real stack. Assert `goBack` is blocked while busy (`navigation.canGoBack()` / screen still mounted)
3. Android back: `BackHandler.mockPressBack?.()` from the RN jest mock (verify for your version), or test the handler function directly
4. E2E: **Maestro** (`back`, `swipe`, `tapOn` twice fast) or Detox on real devices, with network throttling

### 11.2 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| F1 | Type in a field, tap Save **while it is still focused** | The latest (formatted) value is saved |
| F2 | Return/Done on the last field | Editing ends, keyboard hides, the same pipeline runs as the Save button |
| F3 | Header Save with keyboard open | Same as F1 |
| F4 | Double / triple tap Save quickly | Exactly one request |
| F5 | During HUD: header back, iOS swipe-back, Android back, predictive back, sheet swipe-down, tab tap | Nothing happens. The screen stays and the HUD stays |
| F6 | Fast success (50 ms) | HUD visible ≥ min time, no flash, then navigate |
| F7 | Server error / timeout / offline / thrown exception / cancel | HUD hides every time, values kept, error shown in a **native alert** (no toast/banner) |
| F8 | Server field error | Error on the right field, focus + scroll to it, announced |
| F9 | Submit with invalid fields | No request. **Native alert** lists the errors. After OK, the first invalid field is focused, and inline errors are shown too |
| F9b | Blur an invalid field | Inline error + red border + shake (no shake with reduce motion). **No alert** |
| F9c | Server 422 with field errors | HUD hides, **then** an alert, plus inline errors on the mapped fields |
| F9d | 409 / business error ("email already registered") | HUD hides, then an alert with the server's message |
| F9e | 500 or timeout | Alert with **Retry** + Cancel |
| F9f | Tap Retry | Same wrapper runs: HUD shows again, one request, R1 + R2 apply |
| F9g | Submit while offline | Alert ("You're offline") with Retry, values kept |
| F9h | 401 after refresh failed | Alert, then routed to re-login |
| F9i | Any failure | The alert never appears while the HUD is visible |
| F10 | Next chain | Visual order, keyboard stays up, Done submits |
| F11 | Dirty form → back | "Discard changes?" shown. Keep editing stays, Discard leaves |
| F12 | Back during submit on a dirty form | No discard prompt (blocked by HUD) |
| F13 | Success on a dirty form | No discard prompt on the navigate-away |
| F14 | Autofill (password manager, OTP from SMS/Messages) | Fields filled, validation passes, submit works |
| F15 | Small device + keyboard | All fields reachable, button visible |
| F16 | Largest font size + VoiceOver/TalkBack | Labels read, errors announced, HUD traps focus, "Saving…" announced |
| F17 | App backgrounded during submit | HUD still consistent on return, result applied once |
| F18 | Multi-step Next that hits the server | R1 + R2 apply, back blocked during it |

### 11.3 RNTL sketches

- **F1**: `changeText(name, ' Ann ')` without blur → press Save → expect `api` called with `'Ann'` (the trim in `onBlur` ran).
- **F4**: `api` is a pending promise. Press Save 3× → `api` called once.
- **F9**: `jest.spyOn(Alert, 'alert')`. Submit an empty form → alert called once with the error list. Invoke the OK button's `onPress` → the first field is focused. Blur an invalid field → `Alert.alert` not called.
- **F7**: `api` rejects → `await waitForElementToBeRemoved(() => getByTestId('hud'))`, then `Alert.alert` called once. **F9f**: call the Retry button's `onPress` → `api` called again and the HUD shows.
- **F5**: while pending, call `navigation.goBack()` → screen still rendered. `BackHandler` handler returns `true`.

## 12. Review checklist (summary)

1. - [ ] **R1**: every action goes through `useFormSubmit`, which ends editing first. No raw `handleSubmit` on buttons, Return keys or header items
2. - [ ] **R2**: a `Modal` (or root-level) HUD blocks all touches, header back, swipe-back, Android/predictive back and sheet swipe. Re-entrancy ref. Minimum display time. `finally` hide. Timeout. a11y announcement
3. - [ ] **R3**: every submit failure (client validation, 422, 409/business, 4xx/5xx, timeout, offline, 401) in `Alert.alert` from the wrapper after the HUD hides, mapped by one `alertContent`, Retry where safe; 422 also inline. Blur errors inline with shake/reduced motion, never an alert. Focus the first error after the alert is dismissed
4. - [ ] Keyboard avoidance, Next/Done chaining, `keyboardShouldPersistTaps="handled"`
5. - [ ] Correct keyboard types, `textContentType`/`autoComplete`, masks storing raw values
6. - [ ] Unsaved-changes guard that is off while busy and after success
7. - [ ] Accessibility and font scaling per §10

## 13. Common bugs to look for in review

- **R1** `onPress={handleSubmit(onValid)}` directly. The focused masked or trimmed field commits on blur after the values were read, so a stale value gets saved
- **R1** `onSubmitEditing` on the last field calls the API directly, skipping blur, validation and the HUD
- **R1** header `headerRight` Save created once in `useLayoutEffect` with a stale closure over old values
- **R2** busy flag in `useState` only. A double tap within one frame sends two requests
- **R2** HUD rendered inside the screen. The header back button and tab bar stay tappable
- **R2** `beforeRemove`/`usePreventRemove` without `gestureEnabled: false`. The iOS swipe-back still pops the native screen (RN Navigation logs a warning)
- **R2** `Modal` without `onRequestClose` (Android warning) or with `onRequestClose={hide}`. Back dismisses the HUD mid-request
- **R2** HUD hidden only on success. Error, timeout or throw leaves a **stuck HUD** and a dead app
- **R2** no minimum display. The HUD flashes for 1 frame on fast networks
- **R2** navigating while the `Modal` is still visible. It lingers or blocks the next screen's native modal
- **R3** `Alert.alert` fired from `onBlur` / field validation (a modal nag while typing)
- **R3** submit failure shows only inline errors, often off-screen, so the user can't tell why nothing happened
- **R3** server alert shown while the HUD `Modal` is still up (iOS drops or stacks it), or server errors alerted but not `setError`'d inline
- **R3** server/network error shown as a toast, banner or snackbar instead of a native alert: easy to miss, disappears on its own, not reliably announced by screen readers
- **R3** each screen maps errors itself (`onError` toasts) instead of the shared wrapper + `alertContent`, so some forms show nothing or no Retry
- **R3** shake runs on every re-render or ignores `useReducedMotion`
- The discard-changes alert shows during submit, or after success because `reset()` wasn't called
- `shouldFocusError` does nothing because `field.ref` isn't passed to the `TextInput`
- Validation on every keystroke from the start ("invalid email" while typing the first letter)
- `blurOnSubmit={false}` left on RN 0.73+ (deprecated), or `submitBehavior` missing so the keyboard flickers between fields
- `KeyboardAvoidingView` with `behavior="height"` on Android plus edge-to-edge → double offset / jumping
- `keyboardShouldPersistTaps` missing → the first tap on Save only dismisses the keyboard
- Email field autocapitalizes, or the password field lacks `textContentType`, so autofill doesn't work
- Masked display value stored and sent to the API; phone not normalized to E.164
- Draft persistence saves passwords or card numbers in plain AsyncStorage

## 14. Output format

When asked to **review/audit**:
1. **Summary**: versions (RN, Expo, React Navigation / Expo Router, RHF, zod, keyboard-controller), plus the form spec table (§1) for each form
2. **R1 / R2 / R3 compliance per form first**, listing each action (button, Return key, header item, Next) and each back path with ✅ / ❌ and file:line
3. **Checklist results** for §5–§10 with ✅ Pass / ❌ Fail / ⚠️ Needs check
4. **Critical issues** (stale saves, submit errors shown as toast/banner instead of an alert, double submits, stuck HUD, back during submit, lost input) with fix code
5. **Testing gaps** against F1–F18

When asked to **implement**: confirm the versions, start from the Appendix (`endEditing`, `useFormSubmit` + `alertContent`, `BlockingHud`, `useBlockBack`), wire **every** action through the wrapper, and then list what to verify by hand: swipe-back and predictive back on devices, autofill with real password managers, small screens with the keyboard up, VoiceOver/TalkBack.

When asked to **debug**: find the closest F-scenario, check §13, and log blur → onChange → handleSubmit → hud show/hide → navigation events in order.

---

## Appendix — Reference implementation (Expo SDK 53+, TypeScript)

Assumes `react-hook-form` 7.5x, `zod` 3.23+/4 with a matching `@hookform/resolvers`, `@react-navigation/native` 7 (directly or through Expo Router 4/5), `react-native-keyboard-controller` 1.x. Verify each API against the installed versions.

### A.1 endEditing (R1)

```ts
// forms/endEditing.ts
import { Keyboard, TextInput } from 'react-native';

const nextFrame = () => new Promise<void>((r) => requestAnimationFrame(() => r()));

/** Blur the focused input so onBlur (formatters, RHF onBlur) commits, hide the keyboard, wait a frame. */
export async function endEditing(): Promise<void> {
  TextInput.State.currentlyFocusedInput?.()?.blur();
  Keyboard.dismiss();
  await nextFrame(); await nextFrame(); // flush onBlur → field.onChange, then RHF's store
}
```

### A.2 Blocking HUD provider (R2: overlay, a11y, min display, ref-count)

```tsx
// forms/BlockingHud.tsx
import React, { createContext, useCallback, useContext, useMemo, useRef, useState } from 'react';
import { AccessibilityInfo, ActivityIndicator, Modal, StyleSheet, Text, View } from 'react-native';

type Hud = { show: (message?: string) => void; hide: (announce?: string) => void; busy: boolean };

const HudContext = createContext<Hud | null>(null);

export function BlockingHudProvider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState<{ visible: boolean; message: string }>({ visible: false, message: '' });
  const count = useRef(0);

  const show = useCallback((message = 'Saving…') => {
    count.current += 1;
    setState({ visible: true, message });
    AccessibilityInfo.announceForAccessibility(message);
  }, []);

  const hide = useCallback((announce?: string) => {
    count.current = Math.max(0, count.current - 1);
    if (count.current === 0) setState((s) => ({ ...s, visible: false }));
    if (announce) AccessibilityInfo.announceForAccessibility(announce);
  }, []);

  const value = useMemo(() => ({ show, hide, busy: state.visible }), [show, hide, state.visible]);

  return (
    <HudContext.Provider value={value}>
      {children}
      {/* onRequestClose no-op: swallows Android hardware/predictive back while busy */}
      <Modal visible={state.visible} transparent animationType="fade" statusBarTranslucent
        onRequestClose={() => {}} testID="hud">
        <View style={styles.backdrop} accessibilityViewIsModal>
          <View style={styles.card} accessible accessibilityRole="progressbar"
            accessibilityLabel={state.message} accessibilityLiveRegion="polite">
            <ActivityIndicator size="large" />
            {!!state.message && <Text style={styles.text}>{state.message}</Text>}
          </View>
        </View>
      </Modal>
    </HudContext.Provider>
  );
}

export function useHud(): Hud {
  const ctx = useContext(HudContext);
  if (!ctx) throw new Error('useHud outside BlockingHudProvider');
  return ctx;
}

const styles = StyleSheet.create({ backdrop: { ...StyleSheet.absoluteFillObject, backgroundColor: 'rgba(0,0,0,0.35)', alignItems: 'center', justifyContent: 'center' },
  card: { minWidth: 140, padding: 24, borderRadius: 16, backgroundColor: '#fff', alignItems: 'center', gap: 12 }, text: { fontSize: 16, color: '#111', textAlign: 'center' } });
```

Mount `<BlockingHudProvider>` once, above the navigator (in Expo Router: `app/_layout.tsx` around `<Stack />`).

### A.3 Back blocking hook (R2: every back path)

```tsx
// forms/useBlockBack.ts
import { useCallback, useLayoutEffect } from 'react';
import { useFocusEffect, useNavigation, usePreventRemove } from '@react-navigation/native';
import { BackHandler } from 'react-native';

/**
 * While `busy`: no header back, no swipe-back / sheet swipe, no JS goBack, no Android back.
 * Works in Expo Router screens too (useNavigation returns the stack navigation).
 */
export function useBlockBack(busy: boolean) {
  const navigation = useNavigation();

  useLayoutEffect(() => {
    // iOS swipe-back + sheet swipe-down; native-stack header back (JS stack: headerLeft: () => null)
    navigation.setOptions({ gestureEnabled: !busy, headerBackVisible: !busy });
  }, [navigation, busy]);

  // Blocks navigation.goBack(), header back on JS stack, pops from deep links.
  usePreventRemove(busy, () => { /* swallow while busy */ });

  // Belt and braces for Android back if the HUD isn't a Modal.
  useFocusEffect(
    useCallback(() => {
      const sub = BackHandler.addEventListener('hardwareBackPress', () => busy);
      return () => sub.remove();
    }, [busy]),
  );
}
```

Expo Router alternative for static options: `<Stack.Screen options={{ gestureEnabled: !busy, headerBackVisible: !busy }} />` rendered inside the screen. It re-applies when `busy` changes.

### A.4 Shared submit wrapper (R1 + R2 + R3 alerts for every failure)

```tsx
// forms/useFormSubmit.ts
import { useCallback, useRef } from 'react';
import type { FieldValues, Path, UseFormReturn } from 'react-hook-form';
import { AccessibilityInfo, Alert } from 'react-native';
import { endEditing } from './endEditing';
import { useHud } from './BlockingHud';

export class ServerValidationError extends Error { constructor(public fields: Record<string, string[]>) { super('Validation failed'); } } // 422
export class BusinessError extends Error { constructor(public title: string, message: string) { super(message); } } // 409
export class HttpError extends Error { constructor(public status: number) { super(`HTTP ${status}`); } }
export class AuthExpiredError extends Error {}      // 401 after refresh failed

type AlertContent = { title: string; message: string; retryable: boolean };
/** R3: the ONE place that maps a submit failure to native alert content. */
export function alertContent(e: unknown, safeToRetry: boolean): AlertContent {
  if (e instanceof BusinessError) return { title: e.title, message: e.message, retryable: false };
  if (e instanceof AuthExpiredError) return { title: 'Session expired', message: 'Please sign in again.', retryable: false };
  if (e instanceof HttpError && e.status < 500) return { title: "Couldn't save", message: 'Something went wrong. Please try again later.', retryable: false };
  const offline = e instanceof TypeError, timeout = (e as Error)?.name === 'AbortError'; // verify for your client
  return { title: offline ? "You're offline" : timeout ? 'This is taking too long' : "Couldn't save",
    message: offline ? 'Check your connection and try again.' : 'Please try again.', retryable: safeToRetry };
}

const delay = (ms: number) => new Promise((r) => setTimeout(r, ms));
const MIN_HUD_MS = 500;
const TIMEOUT_MS = 30_000;

type Options<T extends FieldValues, R> = {
  form: UseFormReturn<T>;
  fieldOrder: Path<T>[];                         // visual order for focusing errors
  submit: (values: T, signal: AbortSignal) => Promise<R>;
  onSuccess?: (result: R, values: T) => void;    // navigate / toast — runs after HUD hides
  onAuthExpired?: () => void;                    // route to re-login after the alert
  safeToRetry?: boolean;                         // default false; set true only if idempotent / idempotency key → offer Retry
  hudMessage?: string;
};

export function useFormSubmit<T extends FieldValues, R>(opts: Options<T, R>) {
  const { form, fieldOrder, submit, onSuccess, onAuthExpired, safeToRetry = false, hudMessage = 'Saving…' } = opts;
  const hud = useHud();
  const inFlight = useRef(false);

  // R3: submit-time errors → native alert listing them; focus first invalid field on dismiss.
  const alertErrors = useCallback((extra?: string) => {
    const errors = form.formState.errors as Record<string, { message?: string } | undefined>;
    const invalid = fieldOrder.filter((f) => errors[f as string]);
    const lines = invalid.map((f) => errors[f as string]?.message).filter(Boolean) as string[];
    if (extra) lines.push(extra);
    if (!lines.length) return;
    const shown = lines.slice(0, 4).map((l) => `• ${l}`);
    if (lines.length > 4) shown.push(`and ${lines.length - 4} more`);
    const title = lines.length === 1 ? 'Please check this field' : `Please check ${lines.length} fields`;
    AccessibilityInfo.announceForAccessibility(title);
    Alert.alert(title, shown.join('\n'), [
      { text: 'OK', onPress: () => { if (invalid[0]) form.setFocus(invalid[0]); } },
    ]);
  }, [form, fieldOrder]);

  const runRef = useRef<() => Promise<void>>(async () => {});
  // R3: every non-field failure → native alert (HUD is already hidden). Retry re-runs the wrapper.
  const alertFailure = useCallback((e: unknown) => {
    const { title, message, retryable } = alertContent(e, safeToRetry);
    AccessibilityInfo.announceForAccessibility(title);
    Alert.alert(title, message, retryable
      ? [{ text: 'Cancel', style: 'cancel' }, { text: 'Retry', onPress: () => runRef.current() }]
      : [{ text: 'OK', onPress: () => { if (e instanceof AuthExpiredError) onAuthExpired?.(); } }]);
  }, [safeToRetry, onAuthExpired]);

  const run = useCallback(async () => {
    if (inFlight.current) return;               // R2 re-entrancy (synchronous)
    inFlight.current = true;
    try {
      await endEditing();                        // R1
      await form.handleSubmit(
        async (values) => {
          const controller = new AbortController();
          const timer = setTimeout(() => controller.abort(), TIMEOUT_MS);
          const started = Date.now();
          let result: R | undefined, failure: unknown;
          hud.show(hudMessage);                  // R2
          try { result = await submit(values, controller.signal); } catch (e) { failure = e; } finally {
            clearTimeout(timer);
            const left = MIN_HUD_MS - (Date.now() - started);
            if (left > 0) await delay(left);
            hud.hide(failure ? undefined : 'Saved'); // ALWAYS hidden
          }
          if (failure === undefined) {
            form.reset(values);                  // clears isDirty before navigating
            onSuccess?.(result as R, values);
            return;
          }
          if (failure instanceof ServerValidationError) {
            let unknown: string | undefined;
            for (const [name, msgs] of Object.entries(failure.fields)) {
              if ((fieldOrder as string[]).includes(name)) {
                form.setError(name as Path<T>, { type: 'server', message: msgs[0] }); // inline too
              } else unknown = msgs[0];
            }
            if (unknown) form.setError('root.server' as Path<T>, { type: 'server', message: unknown });
            alertErrors(unknown);                  // HUD already hidden above
          } else {
            alertFailure(failure);               // never a toast/banner/snackbar
          }
        },
        () => alertErrors(),                     // R3: submit-time client errors
      )();
    } finally {
      inFlight.current = false;
    }
  }, [form, submit, onSuccess, hud, hudMessage, alertErrors, alertFailure]);
  runRef.current = run;

  return { submit: run, busy: hud.busy };
}
```

### A.5 Schema and a reusable text field

```tsx
// forms/profileSchema.ts
import { z } from 'zod';

export const profileSchema = z.object({
  name: z.string().trim().min(1, 'Enter your name'),
  email: z.string().trim().toLowerCase().email('Enter a valid email'),
  phone: z.string().regex(/^\+?[1-9]\d{7,14}$/, 'Enter a valid phone number'),
});
export type ProfileValues = z.infer<typeof profileSchema>;
export const profileFieldOrder = ['name', 'email', 'phone'] as const;
```

```tsx
// forms/FormTextField.tsx
import React, { forwardRef, useEffect } from 'react';
import { Control, Controller, FieldValues, Path } from 'react-hook-form';
import { StyleSheet, Text, TextInput, TextInputProps } from 'react-native';
import Animated, {
  useAnimatedStyle, useReducedMotion, useSharedValue, withSequence, withTiming,
} from 'react-native-reanimated';

/** R3 inline feedback: short shake when an error first appears; none with reduce motion. */
function useShake(error?: string) {
  const x = useSharedValue(0);
  const reduced = useReducedMotion();
  useEffect(() => {
    if (!error || reduced) return;
    x.value = withSequence(
      withTiming(-8, { duration: 50 }), withTiming(8, { duration: 50 }),
      withTiming(-5, { duration: 50 }), withTiming(0, { duration: 50 }),
    );
  }, [error, reduced, x]);
  return useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }));
}

function Shake({ error, children }: { error?: string; children: React.ReactNode }) {
  const style = useShake(error);
  return <Animated.View style={[styles.wrap, style]}>{children}</Animated.View>;
}

type Props<T extends FieldValues> = Omit<TextInputProps, 'value' | 'onChangeText'> & {
  control: Control<T>;
  name: Path<T>;
  label: string;
  /** commit-on-blur formatter, e.g. trim / normalize phone */
  format?: (v: string) => string;
};

function FormTextFieldInner<T extends FieldValues>(
  { control, name, label, format, ...input }: Props<T>,
  nextRef: React.ForwardedRef<TextInput>,
) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field, fieldState }) => (
        <Shake error={fieldState.error?.message}>
          <Text style={styles.label} nativeID={`${name}-label`}>{label}</Text>
          <TextInput
            ref={(el) => {
              field.ref(el);  // lets RHF setFocus / shouldFocusError work
              if (typeof nextRef === 'function') nextRef(el);
              else if (nextRef) nextRef.current = el;
            }}
            value={field.value ?? ''}
            onChangeText={field.onChange}
            onBlur={() => {
              if (format && typeof field.value === 'string') field.onChange(format(field.value));
              field.onBlur();
            }}
            accessibilityLabel={label}
            accessibilityLabelledBy={`${name}-label`}
            accessibilityHint={fieldState.error?.message}
            style={[styles.input, fieldState.error && styles.inputError]}
            {...input}
          />
          {fieldState.error && (
            <Text style={styles.error} accessibilityLiveRegion="polite">⚠ {fieldState.error.message}</Text>
          )}
        </Shake>
      )}
    />
  );
}

export const FormTextField = forwardRef(FormTextFieldInner) as <T extends FieldValues>(
  p: Props<T> & { ref?: React.Ref<TextInput> },
) => React.ReactElement;

const styles = StyleSheet.create({
  wrap: { gap: 6, marginBottom: 16 }, label: { fontSize: 15, fontWeight: '600' },
  input: { minHeight: 48, borderWidth: 1, borderColor: '#bbb', borderRadius: 10, paddingHorizontal: 12, fontSize: 16 },
  inputError: { borderColor: '#c62828', borderWidth: 2 }, error: { color: '#c62828', fontSize: 14 },
});
```

### A.6 Unsaved-changes guard

```tsx
// forms/useDiscardGuard.ts
import { usePreventRemove, useNavigation } from '@react-navigation/native';
import { Alert } from 'react-native';

export function useDiscardGuard(isDirty: boolean, busy: boolean) {
  const navigation = useNavigation();
  // Never prompt while busy: useBlockBack swallows back then.
  usePreventRemove(isDirty && !busy, ({ data }) => {
    Alert.alert('Discard changes?', 'Your changes will be lost.', [
      { text: 'Keep editing', style: 'cancel' },
      { text: 'Discard', style: 'destructive', onPress: () => navigation.dispatch(data.action) },
    ]);
  });
}
```

### A.7 Sample screen wiring everything (Expo Router)

```tsx
// app/profile/edit.tsx
import React, { useEffect, useLayoutEffect, useRef } from 'react';
import { Button, Pressable, Text, TextInput } from 'react-native';
import { Stack, useRouter, useNavigation } from 'expo-router';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { KeyboardAwareScrollView, KeyboardStickyView } from 'react-native-keyboard-controller';
import { profileSchema, profileFieldOrder, type ProfileValues } from '@/forms/profileSchema';
import { FormTextField } from '@/forms/FormTextField';
import { useFormSubmit, ServerValidationError, BusinessError, HttpError } from '@/forms/useFormSubmit';
import { useBlockBack } from '@/forms/useBlockBack';
import { useDiscardGuard } from '@/forms/useDiscardGuard';
import { api } from '@/api';
import { toast } from '@/ui/toast';

export default function EditProfile() {
  const router = useRouter();
  const navigation = useNavigation();
  const emailRef = useRef<TextInput>(null);
  const phoneRef = useRef<TextInput>(null);

  const form = useForm<ProfileValues>({
    resolver: zodResolver(profileSchema), mode: 'onBlur', reValidateMode: 'onChange',
    defaultValues: { name: '', email: '', phone: '' },
  });

  useEffect(() => { api.getMe().then((me) => form.reset(me)); }, [form]); // baseline for isDirty

  const { submit, busy } = useFormSubmit<ProfileValues, void>({
    form,
    fieldOrder: [...profileFieldOrder],
    submit: async (values, signal) => {
      const res = await api.patchMe(values, { signal });
      if (res.status === 422) throw new ServerValidationError(res.errors);
      if (res.status === 409) throw new BusinessError(res.title ?? "Couldn't save", res.message);
      if (!res.ok) throw new HttpError(res.status);  // 401-after-refresh → AuthExpiredError in api
    },
    onSuccess: () => { toast('Profile saved'); router.back(); }, // success toast is fine
    onAuthExpired: () => router.replace('/login'),
    // No onError: the wrapper shows a native alert for every failure (R3).
  });

  useBlockBack(busy);                              // R2 back paths
  useDiscardGuard(form.formState.isDirty, busy);   // only when not submitting

  // Header Save goes through the same wrapper (R1). `submit` is stable via useCallback.
  useLayoutEffect(() => {
    navigation.setOptions({
      headerRight: () => (
        <Pressable onPress={submit} accessibilityRole="button" hitSlop={12}><Text>Save</Text></Pressable>
      ),
    });
  }, [navigation, submit]);

  return (
    <>
      <Stack.Screen options={{ title: 'Edit profile', gestureEnabled: !busy && !form.formState.isDirty }} />
      <KeyboardAwareScrollView bottomOffset={24} keyboardShouldPersistTaps="handled"
        keyboardDismissMode="interactive" contentContainerStyle={{ padding: 16 }}>
        <FormTextField
          control={form.control} name="name" label="Name"
          format={(v) => v.trim()}
          textContentType="name" autoComplete="name" autoCapitalize="words"
          returnKeyType="next" submitBehavior="submit"
          onSubmitEditing={() => emailRef.current?.focus()}
        />
        <FormTextField
          ref={emailRef}
          control={form.control} name="email" label="Email"
          format={(v) => v.trim().toLowerCase()}
          keyboardType="email-address" textContentType="emailAddress" autoComplete="email"
          autoCapitalize="none" autoCorrect={false}
          returnKeyType="next" submitBehavior="submit"
          onSubmitEditing={() => phoneRef.current?.focus()}
        />
        <FormTextField
          ref={phoneRef}
          control={form.control} name="phone" label="Phone"
          format={(v) => v.replace(/[^\d+]/g, '')}
          keyboardType="phone-pad" textContentType="telephoneNumber" autoComplete="tel"
          returnKeyType="done"
          onSubmitEditing={submit}  // Return = Save, same pipeline (R1 + R2)
        />
      </KeyboardAwareScrollView>
      <KeyboardStickyView offset={{ opened: 0, closed: -24 }}>
        <Button title="Save" onPress={submit} disabled={busy} />
      </KeyboardStickyView>
    </>
  );
}
```

For plain React Navigation, swap `Stack.Screen` for `navigation.setOptions` (already done in `useBlockBack`) and `router.back()` for `navigation.goBack()`. Mount `KeyboardProvider` and `BlockingHudProvider` once at the app root.
