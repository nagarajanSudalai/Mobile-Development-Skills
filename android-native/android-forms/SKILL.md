---
name: android-forms
description: Use when implementing, reviewing, testing, or debugging forms in a native Android app (Kotlin, Jetpack Compose and Views) — login, sign-up, profile edit, checkout, address, OTP and multi-step wizards. Enforces two non-negotiable rules on every form, (R1) end editing before any Done/Save/Submit/Next/IME action (LocalFocusManager.clearFocus, LocalSoftwareKeyboardController.hide, Views clearFocus + InputMethodManager.hideSoftInputFromWindow) through one shared submit wrapper, and (R2) a full-screen blocking HUD while any form action runs that consumes all pointer input and blocks every back path (BackHandler, PredictiveBackHandler, OnBackPressedCallback, Dialog/ModalBottomSheet dismiss properties), with re-entrancy guard, minimum display time and guaranteed dismissal, and (R3) submit-time validation and server errors in a native AlertDialog / MaterialAlertDialogBuilder while field-level errors stay inline (supportingText, isError, shake, PlainTooltip). Also covers the submit pipeline, blur + submit validation, server field error mapping, KeyboardOptions/KeyboardActions IME chaining, imePadding/WindowInsets, bringIntoView, autofill semantics (ContentType), secure entry, masks via VisualTransformation/OutputTransformation, unsaved-changes discard prompt, drafts, multi-step state, accessibility and font scale. Siblings: ios-forms, react-native-forms, flutter-forms; related: android-network-offline-sync, android-crash-analytics, android-list-ux, android-deep-links.
---

# Android Forms Skill

Guide for building, auditing and testing **forms** in a consumer Android app (Kotlin, Jetpack Compose first, Views where the app still uses them; ViewModel + coroutines + Hilt assumed). A form is any screen where the user enters data and an action sends it somewhere: login, sign-up, OTP, profile edit, address, checkout, settings with a Save button, multi-step onboarding.

Sibling skills: **ios-forms**, **react-native-forms**, **flutter-forms** — same structure per platform. Related: **android-network-offline-sync** (the submit API call, retries, offline queueing of submissions), **android-crash-analytics** (form funnel events: `form_viewed`, `form_submitted`, `form_failed` with field names, never values), **android-list-ux** (pickers backed by long lists), **android-deep-links** (forms opened from a link, e.g. password reset, must still apply all three rules).

## The three non-negotiable rules — apply to EVERY form in the app

> **R1. END EDITING FIRST.** Every Done / Save / Submit / Next / Complete tap, every keyboard IME action (Done, Go, Send, Next-on-last-field), and every toolbar / top-app-bar action that acts on the form must **end editing before calling its function**: commit the focused field's pending value, run its formatter and field validation, and dismiss the keyboard. Enforce it with **one shared submit wrapper** (`FormSubmitter.submit { }` in the appendix) so no button can bypass it. Classic bug: the action reads state before the focused field's blur-commit (formatting, trimming, "apply on focus lost") ran — the stale value is saved.
>
> **R2. FULL-PAGE BLOCKING HUD while any form action runs.** A full-screen overlay (spinner + optional message) that blocks **all** touches underneath (double taps, other buttons, bottom navigation, top app bar) **and** blocks every back path: top-app-bar Up arrow, system back button, gesture back, **predictive back** (no peek animation either), hardware back, `ModalBottomSheet` swipe-down / scrim tap, `Dialog` back / outside tap. Re-entrant submits are ignored. A minimum display time (~400 ms) prevents a flash. The HUD is **always** dismissed on success, error and cancellation (`try/finally`) — a stuck HUD is a bug. Accessibility: announce "Saving…" and keep TalkBack focus inside the overlay.

> **R3. SUBMIT VALIDATION = NATIVE ALERT; FIELD VALIDATION = INLINE.** When a submit fails validation (client-side, or server-side errors returned on submit), show a native Material3 `AlertDialog` (Views: `MaterialAlertDialogBuilder`) with a title and a concise message listing what's wrong (several → a short list, or the first one); on dismiss, focus the first invalid field. Server errors are shown in the alert **after the HUD hides** and are also mapped to inline field errors. Field-level validation (on focus loss / while correcting, before submit) is shown **inline**: `isError` red outline + `supportingText` under the field, with a short shake (skipped when animations are off) or a `PlainTooltip`/popup anchored to the field. **Never** an alert for field-level validation; **never** only inline errors on submit.

No exception for "simple" forms, single-field forms or settings toggles with a Save button. Reviews check all three rules on every form before anything else.

> Compose text and back APIs move fast: state-based `BasicTextField(state = TextFieldState)` + `InputTransformation`/`OutputTransformation` (foundation 1.7+), Material3 `SecureTextField`/`OutlinedSecureTextField` (material3 1.4+), `Modifier.semantics { contentType = ContentType.… }` autofill (Compose UI 1.8+, replacing the experimental `Modifier.autofill`/`AutofillNode`), `KeyboardOptions(autoCorrectEnabled = …)` (renamed from `autoCorrect`, foundation 1.7), `PredictiveBackHandler` (activity-compose 1.8+), `ModalBottomSheetProperties(shouldDismissOnBackPress = …)` (material3 1.3+), `DialogProperties(decorFitsSystemWindows = …)`. **Verify every API against the project's Compose BOM / `libs.versions.toml` and current docs before writing code**, and prefer what the project already uses.
>
> Version gates: predictive back system animations need API 34+ with `android:enableOnBackInvokedCallback="true"` (default-on for apps targeting API 36; on API 33 the callback exists but no animation); `OnBackInvokedDispatcher` API 33+ (use AndroidX `OnBackPressedDispatcher`, which bridges it); edge-to-edge enforced when targeting API 35+ (so `imePadding()`/insets are mandatory, `adjustResize` alone no longer shrinks content); `WindowInsetsAnimation` (smooth IME tracking) API 30+, compat back to 21 via `WindowInsetsAnimationCompat`; Autofill framework API 26+; `View.setAutofillHints` API 26+; `importantForAutofill` API 26+; credential manager passkeys API 28+ via `androidx.credentials`.

## 0. Mental model

```
 User taps Save / IME Done / top-bar ✓ / Next
            |
            v
 +---------------------------------------------------------------+
 | FormSubmitter.submit(action)     (the ONLY way to submit)     |
 |  0. if (busy) return             <- re-entrancy guard         |
 |  1. END EDITING (R1)                                          |
 |     Compose: focusManager.clearFocus(force = true)            |
 |              keyboard?.hide()                                 |
 |              -> onFocusChanged(blur) commits/formats field    |
 |     Views:   currentFocus.clearFocus(); imm.hideSoftInput..   |
 |  2. form.commitAll(); validateAll()                           |
 |     invalid -> inline errors + ALERT (R3), focus first, STOP  |
 |  3. busy = true -> BlockingHud visible (R2)                   |
 |     - Dialog window / root overlay consumes all pointers      |
 |     - BackHandler / PredictiveBackHandler / callback enabled  |
 |     - sheet confirmValueChange = false, dialog no dismiss     |
 |  4. action(snapshot)  -> repository -> API                    |
 |  5. finally: wait min display, busy = false (always)          |
 |  6. Success -> navigate / snackbar / reset dirty              |
 |     Error   -> inline + ALERT (R3) after HUD hides, focus 1st |
 +---------------------------------------------------------------+
            |
 Back pressed while NOT busy and form dirty -> "Discard changes?"
 Back pressed while busy -> swallowed, nothing happens
```

State lives in a ViewModel (`FormState` + `StateFlow`), survives rotation and process death (`SavedStateHandle` for non-secret fields). The UI only renders state and forwards events.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Form type**: login, sign-up, OTP, profile edit, address, checkout/payment, settings-with-Save, multi-step wizard. Each implies field types, autofill hints and security (password managers, `FLAG_SECURE` on payment?).
- **Presentation**: pushed destination (Navigation Compose / Fragment), full-screen dialog, `ModalBottomSheet`, `BottomSheetDialogFragment`, or inline. Every presentation has its own dismiss paths that R2 must block.
- **Create vs edit**: edit pre-fills from the server (loading state before fields are editable), tracks dirty against the original, may need optimistic concurrency (`etag` / `version`).
- **UI toolkit**: Compose-only, Views-only (`TextInputLayout` + `TextInputEditText`), mixed (`ComposeView` in Fragments). Compose BOM, Material3 version, `minSdk`, `targetSdk` (35+ → edge-to-edge).
- **Text field API**: value-based `TextField(value, onValueChange)` or state-based `TextField(state = rememberTextFieldState())`. Don't mix in one form.
- **Validation source of truth**: client rules only, server only, or both (server is final; client is UX). Does the API return field-level errors (`{"errors":{"email":"taken"}}`)?
- **Offline**: may the submission be queued (see **android-network-offline-sync**) or must it fail fast with "You're offline"? A queued submit still runs R1 + R2 for the enqueue step.
- **Existing building blocks**: an app-wide HUD/loading overlay, a submit helper, validation library? Extend them instead of adding a second one — but verify they satisfy R1 + R2.
- **Analytics**: which form events are tracked (see **android-crash-analytics**); never log field values or PII.

Write a **form contract** before coding:

```
form          presentation   fields                     submit API          server errs  offline      dirty prompt
Login         pushed         email,password             POST /session       form-level   fail fast    no
Edit profile  pushed         name,bio,phone,avatar      PATCH /me (etag)    per field    fail fast    yes
Add address   bottom sheet   name,line1,line2,zip,city  POST /addresses     per field    queue        yes
Checkout      3-step wizard  address,shipping,payment   POST /orders        per field    fail fast    yes
```

## 2. R1 — End editing before every action

### 2.1 Why
- Compose fields with value-based state commit on every keystroke, but many forms do work **on blur**: trim, format phone/IBAN/amount, `"1,5" -> 1.5`, "apply on focus lost" in custom fields, blur validation that sets an error. If Save reads state while the field still has focus, that work hasn't run.
- Views: `EditText` text is live, but `OnFocusChangeListener` formatters, `TextInputLayout` error updates, and custom views that commit on focus loss (spinners, number pickers, `AutoCompleteTextView` selection) only update on blur.
- The keyboard staying up after Save covers the snackbar/error and looks broken; on success navigation it can linger over the next screen.

### 2.2 Compose API
- [ ] `val focusManager = LocalFocusManager.current` and `val keyboard = LocalSoftwareKeyboardController.current`.
- [ ] End editing = `focusManager.clearFocus(force = true)` then `keyboard?.hide()`. `clearFocus` delivers `onFocusChanged { !it.isFocused }` to the field, which is where blur commit/format/validate lives. `force = true` clears even if a custom focus target refuses.
- [ ] Do the blur-commit in the **state holder** (`form.onBlur(field)`), not in a local `remember` that the ViewModel never sees.
- [ ] After `clearFocus`, read values **from the ViewModel snapshot** (`form.commitAll()` returns an immutable snapshot) — never from captured lambdas/locals taken before the call. `commitAll()` re-applies formatters to every field idempotently so correctness doesn't depend on callback ordering.
- [ ] `LocalSoftwareKeyboardController` can be `null` (no IME, some embeddings) — always `?.`.
- [ ] Do not rely on `keyboard.hide()` alone: it hides the IME but leaves focus (and the blur commit) untouched.

### 2.3 Views API
- [ ] `val v = activity.currentFocus; v?.clearFocus()` then `getSystemService<InputMethodManager>()?.hideSoftInputFromWindow(v.windowToken, 0)` (or `WindowCompat.getInsetsController(window, view).hide(WindowInsetsCompat.Type.ime())`).
- [ ] `clearFocus()` in Views can move focus to the **first focusable view** in the hierarchy (often the first `EditText` again, reopening the keyboard). Make the root container `android:focusable="true" android:focusableInTouchMode="true"` (or call `root.requestFocus()` after) so focus parks harmlessly.
- [ ] Commit pending custom-view values explicitly (`autoComplete.performCompletion()`, `numberPicker.clearFocus()`), then read `binding.email.text?.toString()?.trim()`.
- [ ] Fragment forms: use `requireView().findFocus()` when `activity.currentFocus` is a different window (dialog fragments have their own window — use `dialog.window`/`dialog.currentFocus`).

### 2.4 The shared submit wrapper
- [ ] One class (`FormSubmitter` / `rememberFormSubmitter()`) owns: end editing → validate → busy → action → finally. Every button, IME action and menu item calls it. **No direct `viewModel.save()` calls from UI.**
- [ ] Lint by review: grep the screen for `onClick = { viewModel.` / `setOnClickListener { viewModel.` / `onDone = {` and confirm each routes through the submitter.
- [ ] IME actions: `KeyboardActions(onDone = { submitter.submit(...) })` — the same entry point as the button. `onNext` on intermediate fields only moves focus (`focusManager.moveFocus(FocusDirection.Down)`), it does not submit.
- [ ] Top-app-bar actions (✓ / Save) and `onOptionsItemSelected` items: same wrapper.
- [ ] Multi-step "Next" is a submit for that step (R1 always; R2 if it hits the server).

## 3. R2 — Full-page blocking HUD

### 3.1 Covering everything
- [ ] **Preferred: render the HUD as a `Dialog`** with `DialogProperties(dismissOnBackPress = false, dismissOnClickOutside = false, usePlatformDefaultWidth = false, decorFitsSystemWindows = false)`. A dialog is its own window above the activity: it covers the top app bar, bottom navigation, FABs, and an open `ModalBottomSheet` (also a window), receives back before the activity, and TalkBack focus is confined to it.
- [ ] **Alternative: root overlay** — a `Box(Modifier.fillMaxSize())` placed at the **root of the activity's content** (above `Scaffold`, `NavHost` and bottom bar, not inside the screen's content slot) that consumes all pointer events in `PointerEventPass.Initial`. An overlay inside a screen does **not** cover the bottom bar or a top bar owned by an outer scaffold, and never covers a `ModalBottomSheet`/`Dialog` window. If the form is in a sheet or dialog, the HUD must be inside that window (or be a Dialog).
- [ ] Pointer consumption: `Modifier.pointerInput(Unit) { awaitPointerEventScope { while (true) awaitPointerEvent(PointerEventPass.Initial).changes.forEach { it.consume() } } }`. `clickable(enabled = false) {}` alone is not enough for scroll/drag gestures; `Modifier.clickable(indication = null)` is acceptable only combined with the consumer.
- [ ] Views: a full-screen `FrameLayout` overlay with `isClickable = true; isFocusable = true` and `setOnTouchListener { _, _ -> true }`, added to `android.R.id.content` (or a `Dialog` with `setCancelable(false); setCanceledOnTouchOutside(false)`).
- [ ] Also disable the submit button while busy (`enabled = !busy`) — defence in depth, and a correct semantic state for TalkBack.

### 3.2 Blocking every back path
| Back path | Compose | Views |
|---|---|---|
| System / gesture back | `BackHandler(enabled = busy) { }` | `OnBackPressedCallback(false)`, `isEnabled = busy` |
| Predictive back peek animation | `PredictiveBackHandler(enabled = busy) { progress -> progress.collect { } }` — consumes the gesture so no peek of the previous screen | same enabled `OnBackPressedCallback` (an enabled callback disables the system back-to-home animation) |
| Top-app-bar Up arrow | `navigationIcon` `enabled = !busy`, and HUD covers it | `toolbar.setNavigationOnClickListener` checks `busy`; overlay covers |
| HUD as Dialog | `dismissOnBackPress = false` (the dialog swallows back) | `setCancelable(false)` |
| `ModalBottomSheet` | `sheetState = rememberModalBottomSheetState(confirmValueChange = { !busy \|\| it != SheetValue.Hidden })`, `properties = ModalBottomSheetProperties(shouldDismissOnBackPress = !busy)`, `onDismissRequest` ignores while busy | `BottomSheetDialogFragment`: `isCancelable = !busy`, `behavior.isDraggable = !busy` |
| Form in `Dialog` | `DialogProperties(dismissOnBackPress = !busy, dismissOnClickOutside = !busy)` | `dialog.setCancelable(!busy)` |
| Navigation from code / deep link | Navigation calls while busy are queued until done or ignored | same |

- [ ] **Order matters**: the most recently composed / added enabled callback wins. Put the busy `BackHandler` **after** (below) any "discard changes" `BackHandler` in the composition, or make them one handler with a `when`. Views: add the busy callback last, or use one callback that branches.
- [ ] Never call `finish()`/`popBackStack()` while busy from any path (timeouts, token-expiry logouts excepted — those cancel the job first, which hides the HUD via finally).
- [ ] With predictive back opt-in (`enableOnBackInvokedCallback=true`) a disabled-but-registered callback is not consulted; the busy callback must be **enabled only while busy** so the normal back animation works otherwise.

### 3.3 Re-entrancy, min display, guaranteed dismissal
- [ ] Guard at the top of `submit`: `if (busy) return`. Use a `Mutex.tryLock()` or an `AtomicBoolean` in the ViewModel, not UI state (two taps can land in one frame before recomposition).
- [ ] Busy flag is ViewModel state (survives rotation; the HUD re-appears after recreation while the job still runs in `viewModelScope`).
- [ ] Minimum display: record `shownAt`; in `finally`, `delay((minMs - elapsed).coerceAtLeast(0))`. Optionally a short show delay (~150 ms) for actions that are usually instant — but input is blocked from t=0 regardless.
- [ ] `try { … } finally { busy = false }` wraps the action. Catch `CancellationException`? Do not swallow it — let it propagate after `finally` runs. `withContext(NonCancellable)` for the min-display delay is unnecessary; if cancelled, hide immediately.
- [ ] Timeouts: the network layer has one (see **android-network-offline-sync**); the HUD never waits forever. Optional "Cancel" button on long operations cancels the job (which runs `finally`).
- [ ] Process death while busy: on restore, busy is **false** (don't persist it); the result of the in-flight request is unknown — edit forms re-fetch, create forms rely on an idempotency key.

### 3.4 Accessibility of the HUD
- [ ] Announce on show: `Modifier.semantics { liveRegion = LiveRegionMode.Polite; contentDescription = "Saving…" }` on the HUD, or `view.announceForAccessibility()` (deprecated on API 36 for most uses — prefer live region / pane title). A Dialog gets `paneTitle` semantics.
- [ ] Focus trap: Dialog window does it for free. Root overlay: mark underlying content `Modifier.semantics { invisibleToUser() }`/`clearAndSetSemantics {}` while busy, or Views `importantForAccessibility = IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS`.
- [ ] Respect reduced motion: spinner is fine; skip decorative animations when `Settings.Global.ANIMATOR_DURATION_SCALE == 0`.
- [ ] After hide: move focus to the result (snackbar, first error field, or the new screen's title).

## 4. Submit pipeline

```
tap -> submitter.submit {
         endEditing()                         // R1
         val snap = form.commitAll()
         val errs = validator.validate(snap)
         if (errs.any()) { form.showErrors(errs); alert(errs); return }  // R3, focus on dismiss
         busy(true)                           // R2, min display starts
         try   { result = repository.submit(snap, idempotencyKey) }
         finally { awaitMinDisplay(); busy(false) }
         when (result) {
           Success      -> form.markClean(); navigate/snackbar
           FieldErrors  -> form.showErrors(map(result)); alert(...)  // R3
           Failure      -> snackbar/dialog with retry (keep input!)
         }
       }
```

- [ ] The coordinator is reusable across forms (generic over the snapshot type) — one implementation, tested once.
- [ ] Navigate **after** the HUD is hidden (or in the same frame as hide) so the next screen isn't covered and no back is blocked on the new destination.
- [ ] One-shot events (navigate, snackbar) via a `Channel`/`SharedFlow` collected with `repeatOnLifecycle`/`LaunchedEffect`, not sticky state that re-fires on rotation.
- [ ] Never clear the user's input on a failure. Passwords may be cleared on auth failure — then focus the password field.
- [ ] Idempotency key per submit attempt (generated when the form opens or per logical attempt) so a retried POST after a timeout doesn't create duplicates.
- [ ] Log `form_submitted` / `form_failed(reason, fields=["email"])` — no values (see **android-crash-analytics**).

## 5. Validation

- [ ] **When**: on blur (field loses focus after having been edited — "touched"), and on submit (all fields). **Not every keystroke** — except to *clear* an existing error as soon as the input becomes valid.
- [ ] **R3 split**: blur / while-correcting errors are **inline only** (never an alert); submit-time failures (client or server) show inline errors **and** a native `AlertDialog` titled e.g. "Check your details" with a concise list ("• Enter a valid email\n• Enter a 5-digit ZIP code"; >3 → first 3 + "and N more"). Confirm button "OK"; on dismiss focus the first invalid field. Server alert shows only after the HUD is hidden.
- [ ] Inline feedback on a newly invalid field: short horizontal shake (`Animatable` offset, ~300 ms) skipped when `ValueAnimator.areAnimatorsEnabled()` is false (API 26+) / animator scale 0; or a `PlainTooltip` (`TooltipBox`) anchored to the field. Views: `TextInputLayout.error` + `ObjectAnimator` shake.
- [ ] Inline errors: Material3 `isError = true` + `supportingText = { Text(error) }`; Views `TextInputLayout.error = …` (and `error = null` to clear; `isErrorEnabled` keeps layout stable).
- [ ] Error text says how to fix ("Enter a 5-digit ZIP code"), not just "Invalid".
- [ ] After the submit alert is dismissed: focus the **first invalid field in visual order** (`FocusRequester` per field) and bring it into view (`BringIntoViewRequester` or `Modifier.bringIntoViewRequester`; Views `scrollView.requestChildFocus` / `smoothScrollTo`). Focusing opens the keyboard — intended for text fields; for pickers just scroll.
- [ ] Server field errors: map API field keys → form field ids (`"address.zip" -> Field.Zip`); unknown keys → form-level error banner. Server errors clear when that field is edited.
- [ ] Cross-field rules (password confirm, date range) validate on submit and on blur of the *second* field.
- [ ] Validators are pure functions in the domain layer, unit-tested; locale-aware (decimal separator, postal codes per country).
- [ ] Don't block submit with a disabled button as the only feedback — a disabled Save with no explanation is a classic consumer-app complaint. Prefer enabled button + errors on tap (disable only while busy).

## 6. Keyboard

- [ ] **Insets**: with edge-to-edge (`enableEdgeToEdge()`, mandatory when targeting 35+), apply `Modifier.imePadding()` (or `windowInsetsPadding(WindowInsets.ime.union(WindowInsets.navigationBars))`) to the scrolling form container, plus `android:windowSoftInputMode="adjustResize"`. Apply insets once — double padding (Scaffold `contentWindowInsets` + `imePadding`) leaves a gap; use `consumeWindowInsets`.
- [ ] Scroll: put fields in `Column(Modifier.verticalScroll(rememberScrollState()).imePadding())` or a `LazyColumn`; focused `TextField`s bring themselves into view when the IME opens if the container is scrollable and resized. Sticky bottom Save button: place it outside the scroll, with `imePadding()` so it rides above the keyboard.
- [ ] `LazyColumn` forms: a focused field scrolled off-screen may be disposed and lose focus/IME — prefer `Column + verticalScroll` for forms under ~30 fields.
- [ ] **IME action chaining**: `KeyboardOptions(imeAction = ImeAction.Next)` + `KeyboardActions(onNext = { focusManager.moveFocus(FocusDirection.Down) })` on every field but the last; last field `ImeAction.Done` (or `Go`/`Send`/`Search`) → `submitter.submit`. Views: `android:imeOptions="actionNext|actionDone"` + `setOnEditorActionListener` (return `true` when handled; also handle `KEYCODE_ENTER` for hardware keyboards).
- [ ] Explicit focus order when layout order differs: `Modifier.focusProperties { next = zipFocus }` / Views `android:nextFocusForward`.
- [ ] **Tap outside to dismiss**: `Modifier.pointerInput(Unit) { detectTapGestures { focusManager.clearFocus() } }` on the background container (doesn't steal field taps). Scrolling the form may also dismiss (optional). Views: override `dispatchTouchEvent` or a background click listener calling the end-editing helper.
- [ ] Don't auto-open the keyboard on screens where the user first reads (profile edit). Do auto-focus the single field on OTP / search / login-email when it's the obvious first action: `LaunchedEffect(Unit) { focusRequester.requestFocus() }`.
- [ ] Hardware keyboards & Chromebooks: Tab moves focus, Enter on last field submits through the wrapper.

## 7. Field types

| Field | KeyboardOptions | Autofill `ContentType` (Compose 1.8+) / Views hint | Notes |
|---|---|---|---|
| Email | `KeyboardType.Email`, `autoCorrectEnabled = false`, `capitalization = None` | `ContentType.EmailAddress` / `AUTOFILL_HINT_EMAIL_ADDRESS` | trim on blur |
| Username | `KeyboardType.Text`, no autocorrect | `ContentType.Username` / `AUTOFILL_HINT_USERNAME` | pairs with password for save prompt |
| Password (login) | `KeyboardType.Password` | `ContentType.Password` / `AUTOFILL_HINT_PASSWORD` | `SecureTextField` or `PasswordVisualTransformation`; show/hide toggle |
| New password | `KeyboardType.Password` | `ContentType.NewPassword` / `"newPassword"` (AndroidX `HintConstants.AUTOFILL_HINT_NEW_PASSWORD`) | strength hint; let managers generate |
| One-time code | `KeyboardType.NumberPassword` or `Number` | `ContentType.SmsOtpCode` / `HintConstants.AUTOFILL_HINT_SMS_OTP` | SMS User Consent / Retriever API; paste whole code |
| Phone | `KeyboardType.Phone` | `ContentType.PhoneNumber` / `AUTOFILL_HINT_PHONE` | E.164 on commit, display mask |
| Name | `KeyboardType.Text`, `capitalization = Words` | `ContentType.PersonFullName` | no "invalid name" rules |
| Address | `Text`, `Words` | `ContentType.AddressStreet`, `PostalCode`, `AddressLocality`, `AddressCountry` | country first, then locale-specific fields |
| Card number | `KeyboardType.Number` | `ContentType.CreditCardNumber`, `CreditCardExpirationDate`, `CreditCardSecurityCode` | prefer the payment SDK's field; `FLAG_SECURE` |
| Amount | `KeyboardType.Decimal` | — | locale decimal separator; parse with `NumberFormat` |
| Date | picker (`DatePickerDialog` / M3 `DatePicker`) | `ContentType.BirthDateFull` for DOB | don't hand-type dates unless DOB with mask |

- [ ] Compose autofill: `Modifier.semantics { contentType = ContentType.Username }` on the field; for login also call `LocalAutofillManager.current?.commit()` after successful submit so the save-password prompt appears (Compose 1.8+). Verify names in your Compose version.
- [ ] Views autofill: `android:autofillHints="emailAddress"`, `importantForAutofill="yes"` on fields, `"noExcludeDescendants"` on unrelated containers; `AutofillManager.commit()` on success, `cancel()` on abandon.
- [ ] Consider Credential Manager (`androidx.credentials`) for passkeys/saved passwords on login forms.
- [ ] **Masks**: display formatting via `VisualTransformation` (value-based) or `OutputTransformation` (state-based) — store raw digits in state, so the cursor and paste work. Input filtering via `InputTransformation.maxLength(…)`/digits-only, or filter in `onValueChange`. Never mutate state with formatting characters on every keystroke (cursor jumps).
- [ ] Secure entry: `SecureTextField` (masks, disables copy, `textObfuscationMode` toggle) or `PasswordVisualTransformation()`; Views `inputType="textPassword"` + `TextInputLayout` `endIconMode="password_toggle"`. Consider `FLAG_SECURE` for payment / recovery-code screens.
- [ ] Pickers (country, date, select): open on tap, not a text field the user can type nonsense into; selecting ends editing of any text field (R1 applies — opening a picker calls `clearFocus`).
- [ ] `singleLine = true` for single-line fields (otherwise Enter inserts a newline instead of firing the IME action).

## 8. Unsaved changes

- [ ] Dirty = `current snapshot != original snapshot` (compare committed/normalized values, so trimming whitespace isn't "dirty").
- [ ] On back / Up / sheet dismiss while dirty **and not busy**: `AlertDialog` "Discard changes?" with Discard / Keep editing. While busy, R2's handler wins — no prompt, nothing happens.
- [ ] Compose: `BackHandler(enabled = dirty && !busy) { showDiscard = true }`; sheet `confirmValueChange` returns `false` when dirty and shows the prompt. Views: `OnBackPressedCallback` with `isEnabled = dirty`.
- [ ] After successful save → mark clean **before** navigating, otherwise the discard prompt fires on the way out.
- [ ] Optional draft persistence for long forms (sign-up, listings): debounce-save to `SavedStateHandle` (process death) or DataStore/Room (app restart). Never persist passwords, card data or OTPs. Offer "Restore draft?" on reopen.
- [ ] Deep links into a form with a dirty draft (see **android-deep-links**) must not silently overwrite it.

## 9. Multi-step forms

- [ ] One ViewModel scoped to the flow's nav graph (`hiltViewModel(navController.getBackStackEntry("checkout"))`) holds all steps' state; each step is a destination.
- [ ] Each **Next** runs R1 (end editing) and validates the current step only; if it calls the server (address validation, OTP verify) it also runs R2.
- [ ] Back between steps keeps entered data; back on step 1 when dirty → discard prompt; back during a step's HUD → nothing.
- [ ] Progress indicator ("Step 2 of 3", `LinearProgressIndicator` with semantics `progressBarRangeInfo`).
- [ ] Final review step shows a summary with "Edit" links per section; final Submit uses the same submitter.
- [ ] Server errors on final submit referring to an earlier step → navigate back to that step and focus the field.

## 10. Accessibility

- [ ] Every field has a visible `label` (not only `placeholder`, which disappears on typing). Required fields marked in text ("Email (required)"), not only with color/asterisk.
- [ ] Errors: `isError` + `supportingText` are exposed via semantics; also set `Modifier.semantics { error(msg) }` on custom fields. On submit failure announce a summary ("2 fields need attention") and move focus to the first invalid field.
- [ ] Focus order follows visual order; test with TalkBack swipe and with a hardware keyboard Tab.
- [ ] Font scale up to 200% (Android 14 non-linear scaling): fields and labels wrap, nothing clipped, Save button still reachable when the keyboard is up. Use `sp` and avoid fixed heights.
- [ ] Touch targets ≥ 48dp (password toggle, clear buttons, checkboxes with their label clickable via `Modifier.toggleable`).
- [ ] Contrast of error text and borders ≥ 4.5:1 / 3:1; don't signal error by color alone (icon + text).

## 11. Testing plan

### 11.1 Unit (JVM)
- Validators (each rule, locale cases), server-error mapper, dirty tracking, snapshot normalization.
- `FormSubmitter` with a fake action: re-entrancy (two calls → one action), busy true during, false after success / exception / cancellation, min display respected (`runTest` + virtual time), invalid → action never called.

### 11.2 Compose UI tests (`createAndroidComposeRule`)
- Type into a field, keep focus, tap Save → the saved snapshot contains the latest, **formatted** value.
- IME Done on last field (`performImeAction()`) → submit runs with committed value and keyboard hidden.
- While busy: `onRoot().performClick()`/tap on bottom bar does nothing; `Espresso.pressBack()` / `activity.onBackPressedDispatcher.onBackPressed()` leaves the screen in place; sheet swipe doesn't dismiss.
- Double `performClick()` on Save → fake repository called once.
- Error result → HUD gone, alert shown (`onNodeWithText("Check your details")`), dismiss → first invalid field focused (`assertIsFocused()`); blur error → inline text, no dialog (`isDialog()` count 0).
- Back while dirty & idle → discard dialog; while busy → no dialog.

### 11.3 Manual / device
- Gesture nav + 3-button nav; predictive back on API 34+ with the developer option / targetSdk 36 — no peek during HUD.
- Rotation and "Don't keep activities" during busy; process death (`adb shell am kill`) after typing.
- TalkBack: HUD announced, cannot swipe to underlying content; errors announced.
- Font scale 200%, small screen (360dp wide) with keyboard up, foldable/tablet, hardware keyboard.
- Password manager (Google, 1Password, Bitwarden) fills login/sign-up; SMS OTP autofill.

### 11.4 Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| 1 | Edit field, tap top-bar Save while field focused | Latest (formatted) value saved; keyboard hidden |
| 2 | Return/Done on last field | Ends editing, validates, submits once |
| 3 | Next IME on middle field | Focus moves to next field, no submit |
| 4 | Double/triple tap Save | Exactly one request |
| 5 | System back / gesture back during HUD | Nothing happens; still on form |
| 6 | Predictive back gesture during HUD (API 34+) | No peek animation, no navigation |
| 7 | Up arrow / bottom-nav tab tap during HUD | Ignored (covered) |
| 8 | Sheet swipe-down / scrim tap during HUD | Sheet stays |
| 9 | Request fails (500 / timeout / offline) | HUD hides, input kept, error shown, retry possible |
| 10 | Server field error | HUD hides, error under field, field focused & scrolled into view |
| 11 | User/scope cancels (leaves via logout, job cancelled) | HUD hides (finally) |
| 12 | Fast success (<100 ms) | HUD visible ≥ min display, no flash |
| 13 | Back while dirty, idle | "Discard changes?" |
| 14 | Back while dirty, busy | No prompt, nothing happens |
| 15 | Save success then back | No discard prompt (marked clean) |
| 16 | Rotate during busy | HUD still shown, single request, result delivered once |
| 17 | Invalid submit | No HUD, no request, first error focused + announced |
| 18 | Autofill fills fields without focus events | Values committed by `commitAll()` at submit |
| 19 | Leave an invalid field (blur) | Inline error + shake/tooltip, **no** alert |
| 20 | Submit with 2 invalid fields | Native alert lists both; inline errors shown; dismiss → first invalid focused |
| 21 | Server returns field errors | HUD hides, then alert; errors also inline under mapped fields |
| 22 | Animations off (developer options) | Inline error without shake |

## 12. Common bugs to look for in review

1. **R1 violation**: button/menu/IME action calls `viewModel.save()` directly — stale value from the focused field (blur formatter didn't run), keyboard left open.
2. `keyboard?.hide()` used as "end editing" — focus (and blur commit) remains.
3. Views `clearFocus()` re-focuses the first `EditText` and reopens the keyboard (root not focusable-in-touch-mode).
4. **R2 violation**: no HUD, only a small button spinner — other buttons, bottom nav and back still work during submit.
5. Overlay placed inside the screen's content slot — bottom bar / outer top bar / sheet remain tappable.
6. Overlay uses `clickable {}` only — scroll/drag gestures still pass through.
7. HUD blocks taps but not back: missing `BackHandler(enabled = busy)`, or the discard handler declared later wins and shows a prompt during submit.
8. Predictive back still animates a peek during submit (no enabled callback while busy).
9. `ModalBottomSheet` dismissible by swipe/scrim/back during submit; `onDismissRequest` pops the destination mid-request.
10. **Stuck HUD**: `busy = false` only on the success path; exception or cancellation leaves it forever. Or `CancellationException` swallowed.
11. Re-entrancy guard on Compose state only — two taps in one frame send two requests.
12. HUD flashes for 1 frame on fast responses (no min display).
13. Navigation fired before the HUD hides; next screen is blocked or back disabled there.
14. Validation on every keystroke shows "Invalid email" after the first letter.
15. Server field errors shown only as a generic toast; user can't find the field.
16. First invalid field not focused / not scrolled into view (hidden behind keyboard).
17. Double inset padding (`Scaffold` + `imePadding`) or none at all (targetSdk 35, edge-to-edge) — Save hidden behind the keyboard.
18. Missing autofill content types → password manager doesn't fill or offer to save; OTP not autofilled.
19. Mask stored in state ("(555) 123-") → cursor jumps, paste breaks, server receives formatting.
20. Discard prompt after successful save (not marked clean), or never shown.
21. Passwords/card numbers persisted to `SavedStateHandle`/drafts or logged to analytics.
22. Busy flag persisted → after process death the HUD shows forever.
23. **R3 violation**: submit validation shows only inline errors (user doesn't notice an error scrolled off-screen) or only a toast/snackbar instead of a native alert.
24. **R3 violation**: alert dialog popped on blur / per field while typing; or shake ignores disabled animations.
25. Server-error alert shown while the HUD is still up (two dialogs stacked), or errors not mapped inline.

## 13. Output format

**Review mode** — produce:
1. Form contract table (section 1) for each form found.
2. R1 checklist per form: every action entry point listed (buttons, IME, menu, top bar) with ✅/❌ routed through the wrapper.
3. R2 checklist per form: overlay coverage, each back path from the table in 3.2, re-entrancy, min display, finally-dismissal, a11y.
3b. R3 checklist per form: submit-time (client + server) errors → native alert + inline; blur errors inline only (shake/tooltip, reduced-motion aware); focus first invalid after dismiss.
4. Other findings ordered by severity (data loss / duplicate submit > stuck UI > UX > polish), each with file:line and a fix.
5. Missing tests from the scenario matrix.

**Implement mode** — produce:
1. The form contract.
2. `FormState`/ViewModel, validators, reuse or add the shared `FormSubmitter` + `BlockingHud` (one per app, not per screen).
3. Screen wiring: every action through the submitter, IME chain, insets, autofill types, discard prompt.
4. Unit + UI tests covering at least scenarios 1, 2, 4, 5, 9, 10, 12, 14, 19, 20, 21.

## Appendix — Reference implementation (Kotlin, Compose, Material3)

Adapt names and versions to the project. Assumes Compose BOM with foundation/ui 1.8+, material3 1.3+, activity-compose 1.8+.

### A.1 Form state model

```kotlin
enum class Field { Name, Email, Phone, Zip }

data class FieldState(val text: String = "", val touched: Boolean = false, val error: String? = null)

data class ProfileSnapshot(val name: String, val email: String, val phone: String, val zip: String)

data class ProfileFormState(
    val fields: Map<Field, FieldState> = Field.entries.associateWith { FieldState() },
    val original: ProfileSnapshot? = null,
    val formError: String? = null,
) {
    operator fun get(f: Field) = fields.getValue(f)
    val snapshot get() = ProfileSnapshot(this[Field.Name].text, this[Field.Email].text, this[Field.Phone].text, this[Field.Zip].text)
    val isDirty get() = original != null && Normalizer.normalize(snapshot) != original
}

object Normalizer {
    fun field(f: Field, raw: String): String = when (f) {
        Field.Name -> raw.trim().replace(Regex("\\s+"), " ")
        Field.Email -> raw.trim().lowercase()
        Field.Phone -> raw.filter { it.isDigit() || it == '+' }
        Field.Zip -> raw.trim().uppercase()
    }
    fun normalize(s: ProfileSnapshot) = ProfileSnapshot(field(Field.Name, s.name), field(Field.Email, s.email), field(Field.Phone, s.phone), field(Field.Zip, s.zip))
}
```

### A.2 Validation helpers

```kotlin
typealias Rule = (String) -> String?

object Rules {
    fun required(msg: String): Rule = { if (it.isBlank()) msg else null }
    fun email(msg: String): Rule = { if (it.isNotEmpty() && !Patterns.EMAIL_ADDRESS.matcher(it).matches()) msg else null }
    fun minDigits(n: Int, msg: String): Rule = { if (it.count(Char::isDigit) in 1 until n) msg else null }
    fun regex(re: Regex, msg: String): Rule = { if (it.isNotEmpty() && !re.matches(it)) msg else null }
}

class Validator(private val rules: Map<Field, List<Rule>>) {
    fun validate(f: Field, value: String): String? = rules[f].orEmpty().firstNotNullOfOrNull { it(value) }
    fun validateAll(values: Map<Field, String>): Map<Field, String> =
        values.mapNotNull { (f, v) -> validate(f, v)?.let { f to it } }.toMap()
}

/** Maps API keys like "address.zip" to form fields; unknown keys become a form-level error. */
fun mapServerErrors(errors: Map<String, String>, keys: Map<String, Field>): Pair<Map<Field, String>, String?> {
    val known = errors.mapNotNull { (k, v) -> keys[k]?.let { it to v } }.toMap()
    val unknown = errors.filterKeys { it !in keys }.values.firstOrNull()
    return known to unknown
}
```

### A.3 Submit coordinator (R1 + R2 core, UI-agnostic)

```kotlin
sealed interface SubmitResult {
    data object Success : SubmitResult
    data class FieldErrors(val errors: Map<String, String>) : SubmitResult
    data class Failure(val message: String) : SubmitResult
}

/** The ONLY way a form action runs: endEditing injected by the UI (R1); busy drives the HUD (R2). */
class FormSubmitter(
    private val scope: CoroutineScope,
    private val minDisplayMs: Long = 400,
    private val now: () -> Long = { SystemClock.elapsedRealtime() },
) {
    private val running = AtomicBoolean(false)
    private val _busy = MutableStateFlow(false)
    val busy: StateFlow<Boolean> = _busy.asStateFlow()

    fun <S> submit(
        endEditing: () -> Unit, commit: () -> S,
        validate: (S) -> Boolean,   // shows inline errors + alert itself (R3)
        action: suspend (S) -> SubmitResult, onResult: (SubmitResult) -> Unit,
    ) {
        if (!running.compareAndSet(false, true)) return       // re-entrancy guard
        endEditing()                                            // R1
        val snapshot = commit()
        if (!validate(snapshot)) { running.set(false); return }
        scope.launch {
            val shownAt = now()
            _busy.value = true                                  // R2 on
            var result: SubmitResult? = null
            try {
                result = action(snapshot)
            } catch (e: Exception) {
                if (e is CancellationException) throw e
                result = SubmitResult.Failure(e.message ?: "Something went wrong")
            } finally {
                val left = minDisplayMs - (now() - shownAt)
                if (left > 0 && result != null) delay(left)     // skip delay when cancelled
                _busy.value = false                             // ALWAYS
                running.set(false)
            }
            result?.let(onResult)                               // navigate after hide
        }
    }
}
```

### A.4 ViewModel

```kotlin
@HiltViewModel
class EditProfileViewModel @Inject constructor(
    private val repo: ProfileRepository,
) : ViewModel() {
    private val _state = MutableStateFlow(ProfileFormState())
    val state = _state.asStateFlow()
    private val _events = Channel<FormEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()
    val submitter = FormSubmitter(viewModelScope)
    private val idempotencyKey = UUID.randomUUID().toString()

    private val validator = Validator(mapOf(
        Field.Name to listOf(Rules.required("Enter your name")),
        Field.Email to listOf(Rules.required("Enter your email"), Rules.email("Enter a valid email, like name@example.com")),
        Field.Phone to listOf(Rules.minDigits(7, "Enter a full phone number")),
        Field.Zip to listOf(Rules.regex(Regex("[0-9]{5}"), "Enter a 5-digit ZIP code")),
    ))

    fun onChange(f: Field, text: String) = _state.update { s ->
        val old = s[f]
        // Only clear an existing error while typing; never add one per keystroke.
        val err = if (old.error != null) validator.validate(f, Normalizer.field(f, text)) else null
        s.copy(fields = s.fields + (f to old.copy(text = text, error = err)))
    }

    fun onBlur(f: Field) = _state.update { s ->
        val old = s[f]
        if (!old.touched && old.text.isEmpty()) return@update s
        val norm = Normalizer.field(f, old.text)
        s.copy(fields = s.fields + (f to old.copy(text = norm, touched = true, error = validator.validate(f, norm))))
    }

    /** Idempotently commits every field (covers autofill that never fired focus events). */
    fun commitAll(): ProfileSnapshot {
        _state.update { s -> s.copy(fields = s.fields.mapValues { (f, v) -> v.copy(text = Normalizer.field(f, v.text)) }) }
        return _state.value.snapshot
    }

    /** Returns first invalid field, or null when valid. */
    fun validateAll(snap: ProfileSnapshot): Field? {
        val values = mapOf(Field.Name to snap.name, Field.Email to snap.email, Field.Phone to snap.phone, Field.Zip to snap.zip)
        val errs = validator.validateAll(values)
        _state.update { s -> s.copy(fields = s.fields.mapValues { (f, v) -> v.copy(touched = true, error = errs[f]) }) }
        val first = Field.entries.firstOrNull { it in errs }
        if (first != null) alert(errs, first)                    // R3: inline AND alert on submit
        return first
    }

    private fun alert(errs: Map<Field, String>, focus: Field?, extra: List<String> = emptyList()) {
        val lines = Field.entries.mapNotNull { errs[it] } + extra
        _events.trySend(FormEvent.Alert("Check your details", lines.take(3) +
            listOfNotNull(if (lines.size > 3) "and ${lines.size - 3} more" else null), focus))
    }

    suspend fun send(snap: ProfileSnapshot): SubmitResult = repo.update(snap, idempotencyKey)

    fun handle(result: SubmitResult) {
        when (result) {
            SubmitResult.Success -> {
                _state.update { it.copy(original = Normalizer.normalize(it.snapshot)) } // mark clean first
                _events.trySend(FormEvent.Saved)
            }
            is SubmitResult.FieldErrors -> {
                val (known, unknown) = mapServerErrors(result.errors,
                    mapOf("name" to Field.Name, "email" to Field.Email, "phone" to Field.Phone, "address.zip" to Field.Zip))
                _state.update { s -> s.copy(formError = unknown,
                    fields = s.fields.mapValues { (f, v) -> known[f]?.let { v.copy(error = it) } ?: v }) }
                alert(known, Field.entries.firstOrNull { it in known }, listOfNotNull(unknown)) // HUD already hidden
            }
            is SubmitResult.Failure -> _events.trySend(FormEvent.Message(result.message))
        }
    }
}

sealed interface FormEvent {
    data object Saved : FormEvent
    data class Focus(val field: Field) : FormEvent
    /** R3: native alert for submit-time errors; focus [focus] after dismiss. */
    data class Alert(val title: String, val lines: List<String>, val focus: Field?) : FormEvent
    data class Message(val text: String) : FormEvent
}
```

### A.5 Compose: end-editing + submit wrapper

```kotlin
@Composable
fun rememberEndEditing(): () -> Unit {
    val focusManager = LocalFocusManager.current
    val keyboard = LocalSoftwareKeyboardController.current
    return remember(focusManager, keyboard) {
        { focusManager.clearFocus(force = true); keyboard?.hide() }
    }
}

/** Screen-level entry point: every button / IME action / top-bar action calls this. */
@Composable
fun rememberProfileSubmit(vm: EditProfileViewModel, requesters: Map<Field, FocusRequester>): () -> Unit {
    val endEditing = rememberEndEditing()
    return remember(vm, endEditing) {
        {
            vm.submitter.submit(
                endEditing = endEditing,
                commit = vm::commitAll,
                validate = { snap -> vm.validateAll(snap) == null }, // alert focuses on dismiss
                action = vm::send,
                onResult = vm::handle,
            )
        }
    }
}
```

### A.6 Compose: blocking HUD (R2)

```kotlin
/** Dialog HUD: own window above activity, sheets, bars; swallows back + predictive back; traps TalkBack. */
@Composable
fun BlockingHud(visible: Boolean, message: String = "Saving…") {
    if (!visible) return
    BackHandler(enabled = true) { /* swallow */ }
    PredictiveBackHandler(enabled = true) { progress ->
        try { progress.collect { } } catch (_: CancellationException) { }
    }
    Dialog(
        onDismissRequest = { /* never dismiss from user input */ },
        properties = DialogProperties(dismissOnBackPress = false, dismissOnClickOutside = false,
            usePlatformDefaultWidth = false, decorFitsSystemWindows = false),
    ) {
        Box(
            Modifier.fillMaxSize().background(Color.Black.copy(alpha = 0.32f)).consumeAllPointers()
                .semantics(mergeDescendants = true) { liveRegion = LiveRegionMode.Polite; contentDescription = message },
            contentAlignment = Alignment.Center,
        ) {
            Surface(shape = MaterialTheme.shapes.large, tonalElevation = 6.dp) {
                Column(Modifier.padding(24.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                    CircularProgressIndicator()
                    Spacer(Modifier.height(12.dp))
                    Text(message, style = MaterialTheme.typography.bodyMedium)
                }
            }
        }
    }
}

/** For a root-overlay variant (no Dialog): consumes taps, scrolls and drags. */
fun Modifier.consumeAllPointers(): Modifier = pointerInput(Unit) {
    awaitPointerEventScope { while (true) awaitPointerEvent(PointerEventPass.Initial).changes.forEach { it.consume() } }
}
```

### A.7 Compose: sample screen wiring everything

```kotlin
@Composable
fun EditProfileScreen(vm: EditProfileViewModel = hiltViewModel(), onDone: () -> Unit) {
    val state by vm.state.collectAsStateWithLifecycle()
    val busy by vm.submitter.busy.collectAsStateWithLifecycle()
    val focusManager = LocalFocusManager.current
    val requesters = remember { Field.entries.associateWith { FocusRequester() } }
    val submit = rememberProfileSubmit(vm, requesters)
    val snackbar = remember { SnackbarHostState() }
    var showDiscard by rememberSaveable { mutableStateOf(false) }
    var alert by remember { mutableStateOf<FormEvent.Alert?>(null) }

    LaunchedEffect(vm) {
        vm.events.collect { e -> when (e) {
            FormEvent.Saved -> onDone()
            is FormEvent.Focus -> requesters[e.field]?.requestFocus()
            is FormEvent.Message -> snackbar.showSnackbar(e.text)
            is FormEvent.Alert -> alert = e                  // R3: shown after HUD hid
        } }
    }

    // Discard prompt only when idle. The HUD's own BackHandler is composed later, so it wins while busy.
    BackHandler(enabled = state.isDirty && !busy) { showDiscard = true }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Edit profile") },
                navigationIcon = {
                    IconButton(enabled = !busy, onClick = { if (state.isDirty) showDiscard = true else onDone() }) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                },
                actions = { TextButton(enabled = !busy, onClick = submit) { Text("Save") } },
            )
        },
        snackbarHost = { SnackbarHost(snackbar) },
    ) { padding ->
        Column(
            Modifier.padding(padding).consumeWindowInsets(padding).imePadding().verticalScroll(rememberScrollState())
                .pointerInput(Unit) { detectTapGestures { focusManager.clearFocus() } }.padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp),
        ) {
            state.formError?.let { Text(it, color = MaterialTheme.colorScheme.error) }
            FormField(vm, state, Field.Name, "Full name", requesters,
                KeyboardOptions(capitalization = KeyboardCapitalization.Words, imeAction = ImeAction.Next),
                ContentType.PersonFullName)
            FormField(vm, state, Field.Email, "Email", requesters,
                KeyboardOptions(keyboardType = KeyboardType.Email, autoCorrectEnabled = false, imeAction = ImeAction.Next),
                ContentType.EmailAddress)
            FormField(vm, state, Field.Phone, "Phone (optional)", requesters,
                KeyboardOptions(keyboardType = KeyboardType.Phone, imeAction = ImeAction.Next),
                ContentType.PhoneNumber)
            FormField(vm, state, Field.Zip, "ZIP code", requesters,
                KeyboardOptions(keyboardType = KeyboardType.Number, imeAction = ImeAction.Done),
                ContentType.PostalCode, onDone = submit)
            Button(onClick = submit, enabled = !busy, modifier = Modifier.fillMaxWidth()) { Text("Save") }
        }
    }

    if (showDiscard) {
        AlertDialog(
            onDismissRequest = { showDiscard = false },
            title = { Text("Discard changes?") },
            confirmButton = { TextButton(onClick = { showDiscard = false; onDone() }) { Text("Discard") } },
            dismissButton = { TextButton(onClick = { showDiscard = false }) { Text("Keep editing") } },
        )
    }

    alert?.let { a ->                                   // R3: submit-time errors only
        val close = { alert = null; a.focus?.let { requesters[it]?.requestFocus() } }
        AlertDialog(
            onDismissRequest = { close() },
            title = { Text(a.title) },
            text = { Text(a.lines.joinToString("\n") { "• $it" }) },
            confirmButton = { TextButton(onClick = { close() }) { Text("OK") } },
        )
    }

    BlockingHud(visible = busy) // last: its BackHandler has priority
}

@Composable
private fun FormField(
    vm: EditProfileViewModel, state: ProfileFormState, field: Field, label: String,
    requesters: Map<Field, FocusRequester>, keyboardOptions: KeyboardOptions,
    contentType: ContentType, onDone: (() -> Unit)? = null,
) {
    val focusManager = LocalFocusManager.current
    val fs = state[field]
    var hadFocus by remember { mutableStateOf(false) }
    val shake = remember { Animatable(0f) }
    LaunchedEffect(fs.error) {                           // R3 inline feedback on a new error
        if (fs.error != null && ValueAnimator.areAnimatorsEnabled()) {
            for (x in listOf(10f, -10f, 6f, -6f, 0f)) shake.animateTo(x, tween(50))
        }
    }
    OutlinedTextField(
        value = fs.text, onValueChange = { vm.onChange(field, it) }, label = { Text(label) },
        isError = fs.error != null, supportingText = fs.error?.let { { Text(it) } },   // R3 inline
        singleLine = true, keyboardOptions = keyboardOptions,
        keyboardActions = KeyboardActions(
            onNext = { focusManager.moveFocus(FocusDirection.Down) },
            onDone = { onDone?.invoke() ?: focusManager.clearFocus() },
        ),
        modifier = Modifier
            .fillMaxWidth()
            .graphicsLayer { translationX = shake.value.dp.toPx() }
            .focusRequester(requesters.getValue(field))
            .onFocusChanged {
                if (hadFocus && !it.isFocused) vm.onBlur(field)  // blur commit / format / validate
                hadFocus = it.isFocused
            }
            .semantics { this.contentType = contentType },
    )
}
```

### A.8 Views: end editing, back blocking, overlay, submit alert

```kotlin
fun Activity.endEditing(root: View) {                // R1; root is focusableInTouchMode
    val focused = currentFocus
    focused?.clearFocus(); root.requestFocus()
    getSystemService<InputMethodManager>()?.hideSoftInputFromWindow((focused ?: root).windowToken, 0)
}

class EditProfileFragment : Fragment(R.layout.fragment_edit_profile) {
    private val vm: EditProfileViewModel by viewModels()
    private var hud: View? = null
    private val busyBack = object : OnBackPressedCallback(false) { override fun handleOnBackPressed() = Unit }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val b = FragmentEditProfileBinding.bind(view)
        // add discard callback first, busy callback LAST (highest priority, also blocks predictive back)
        requireActivity().onBackPressedDispatcher.addCallback(viewLifecycleOwner, busyBack)
        val submit = {
            vm.submitter.submit({ requireActivity().endEditing(b.root) },
                { Field.entries.forEach(vm::onBlur); vm.commitAll() },
                { vm.validateAll(it) == null }, vm::send, vm::handle)
        }
        b.save.setOnClickListener { submit() }
        b.zip.setOnEditorActionListener { _, id, ev ->
            val enter = ev?.keyCode == KeyEvent.KEYCODE_ENTER && ev.action == KeyEvent.ACTION_UP
            if (id == EditorInfo.IME_ACTION_DONE || enter) { submit(); true } else false
        }
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch { vm.submitter.busy.collect { setBusy(b, it) } }
                launch { vm.events.collect { e -> when (e) {
                    FormEvent.Saved -> findNavController().popBackStack()
                    is FormEvent.Alert -> MaterialAlertDialogBuilder(requireContext())   // R3
                        .setTitle(e.title).setMessage(e.lines.joinToString("\n") { "• $it" })
                        .setPositiveButton(android.R.string.ok, null)
                        .setOnDismissListener { e.focus?.let { f -> b.fieldFor(f).requestFocus() } }
                        .show()
                    else -> Unit
                } } }
            }
        }
    }

    private fun setBusy(b: FragmentEditProfileBinding, busy: Boolean) {   // R2
        busyBack.isEnabled = busy; b.save.isEnabled = !busy
        val content = requireActivity().findViewById<ViewGroup>(android.R.id.content)
        if (busy && hud == null) {
            hud = layoutInflater.inflate(R.layout.view_blocking_hud, content, false).apply {
                isClickable = true; isFocusable = true
                setOnTouchListener { _, _ -> true }                        // consume every touch
                ViewCompat.setAccessibilityPaneTitle(this, getString(R.string.saving))
            }.also(content::addView)
            b.root.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS
        } else if (!busy) {
            hud?.let(content::removeView); hud = null
            b.root.importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_AUTO
        }
    }

    override fun onDestroyView() { hud?.let { (it.parent as? ViewGroup)?.removeView(it) }; hud = null; super.onDestroyView() }
}
// Inline field errors (R3, Views): TextInputLayout.error = msg on focus loss +
// ObjectAnimator.ofFloat(til, View.TRANSLATION_X, 0f, 10f, -10f, 6f, 0f) when ValueAnimator.areAnimatorsEnabled().
```
