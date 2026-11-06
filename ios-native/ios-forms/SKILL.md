---
name: ios-forms
description: Use when implementing, reviewing, or debugging form screens on iOS/iPadOS in SwiftUI (Form, TextField, SecureField, @FocusState, .submitLabel/.onSubmit, .interactiveDismissDisabled) or UIKit (UITextField, UITextView, UIScrollView + keyboardLayoutGuide, resignFirstResponder, isModalInPresentation) — login, sign-up, profile edit, checkout, multi-step wizards; ending editing before submit, full-screen blocking HUD that also blocks back/swipe-back/sheet dismiss, submit pipeline, validation and server field errors, keyboard handling, autofill/content types, masks, unsaved-changes prompts, drafts, accessibility, and testing. Siblings: android-forms, react-native-forms, flutter-forms.
---

# iOS Forms Skill

Guide for building and auditing **form screens** on Apple platforms in consumer apps: login, sign-up, OTP, profile edit, address/checkout, settings editors, multi-step onboarding. Covers **SwiftUI** (`Form`, `TextField`, `SecureField`, `@FocusState`, `.submitLabel`, `.onSubmit`) and **UIKit** (`UITextField`/`UITextView`, `UIScrollView`, `UIResponder` chain). Siblings: **android-forms**, **react-native-forms**, **flutter-forms** — keep product behaviour (end editing first, blocking HUD, validation timing, discard prompt) identical across platforms. Related: **ios-network-offline-sync** (submit API, retries, offline queue), **ios-crash-analytics** (form funnel events, submit errors), **ios-list-ux** (pickers backed by long lists), **ios-deep-links** (forms opened from links, e.g. password reset, magic-link login).

SwiftUI focus/keyboard APIs change between releases (`@FocusState` behaviour in `Form`/`List`, `.scrollDismissesKeyboard`, `.defaultFocus`, `TextField(value:format:)` commit timing, `.onSubmit` scopes, `@Observable`). **Verify availability and exact behaviour against the current SDK before shipping**, and gate with `if #available` when the deployment target is lower.

Key OS gates (verify): `@FocusState`, `.submitLabel`, `.onSubmit`, `TextField(value:format:)`, `.textInputAutocapitalization`, `.interactiveDismissDisabled` iOS 15; `.scrollDismissesKeyboard`, `NavigationStack`, `TextField(axis: .vertical)` iOS 16; `@Observable`, `.defaultFocus` (iOS 17, verify), `.sensoryFeedback` iOS 17. UIKit: `keyboardLayoutGuide` iOS 15, `isModalInPresentation` iOS 13, `UIAction`-based targets iOS 14, `.oneTimeCode` content type iOS 12, `passwordRules` iOS 12.

## Non-negotiable rules (apply to EVERY form in the app)

> **R1. END EDITING FIRST.** Every Done / Save / Submit / Next / Complete tap, keyboard Return/Done action, toolbar or nav-bar button that acts on the form must **end editing before calling its function**: commit the focused field's pending value, run formatters/field validation, dismiss the keyboard. Enforce it with **one shared submit wrapper** (`FormSubmitter.run`) so no button can bypass it. The classic bug: reading the model before the focused field commits → stale value saved.
>
> **R2. FULL-PAGE BLOCKING HUD while any form action runs.** A full-screen overlay (spinner + optional message) that blocks **all** touches underneath (double taps, other buttons, tab bar, nav bar) **and** blocks every back path (nav back button, swipe-back edge gesture, sheet swipe-to-dismiss, hardware keyboard Esc / ⌘. ). Re-entrant submits are ignored. Minimum display time avoids a flash. The HUD is **always** dismissed on success, error, and cancellation (`defer`) — a stuck HUD is a bug. Accessibility: announce "Saving…" and trap VoiceOver focus inside the overlay.

> **R3. SUBMIT VALIDATION → NATIVE ALERT; FIELD VALIDATION → INLINE.** When validation fails **on submit** (client-side, or server-side errors returned by the submit after the HUD hides), show a native alert (`UIAlertController` / SwiftUI `.alert`) with a title and a concise message listing what's wrong (several issues: bullet list, or the first one); after the user dismisses it, focus the first invalid field. The same errors are **also** mapped to inline field errors. Field-level validation **before** submit (on blur / while editing) is shown **inline only**: footer error text under the field and/or red border, with a short shake (skipped under Reduce Motion), or a poptip anchored to the field. Never use an alert for field-level validation; never show only inline errors on submit.

All three rules are restated in the review checklist (section 12) and common bugs (section 13). A review that finds any form violating R1 or R2 reports it as **critical**.

## 0. Mental model

```
 user taps Save / Return / Next / nav-bar Done
        │
        ▼
 FormSubmitter.run(action)          ← the ONLY entry point for form actions
   1. guard !isRunning              ← re-entrancy: second tap ignored
   2. endEditing()                  ← R1: focus = nil / view.endEditing(true), yield one runloop
   3. validateAll()  ──fail──▶ R3: native alert + inline errors; on OK focus first invalid; STOP (no HUD)
   4. HUD.show("Saving…")           ← R2: overlay window, back paths disabled, a11y announce
   5. try await action()            ← API via repository (ios-network-offline-sync)
   6. defer { HUD.hide(minVisible) }← always, even on throw / cancel
   7. success → navigate / dismiss / toast
      failure → R3: server field errors → native alert + inline; focus first after OK; else alert + retry
```

State lives in one `@Observable` form model (fields, per-field errors, touched set, dirty baseline, `isSubmitting`). Views only bind to it.

## 1. First, establish context

Find out (from the codebase or by asking):

- **Form type**: login, sign-up, OTP/2FA, password reset, profile edit, address, checkout/payment, feedback, multi-step wizard. Each implies field types, autofill hints and security needs.
- **Presentation**: pushed on a `NavigationStack`/`UINavigationController` (back button + swipe back) or a sheet / `fullScreenCover` (swipe-to-dismiss, Cancel button)? Inside a tab (tab bar must be blocked by the HUD)?
- **Create vs edit**: edit pre-fills from server → needs a dirty baseline and discard prompt; create may need draft persistence.
- **Offline**: does submit require the network (login, payment) or can it queue (profile edit, feedback)? See **ios-network-offline-sync**.
- **Server contract**: field-level error format (`{ "errors": { "email": ["taken"] } }`?), idempotency key for payments/orders, rate limits, validation rules shared with backend.
- **Framework & min OS**: SwiftUI, UIKit, or mixed (SwiftUI form inside UIKit nav).
- **Analytics**: which funnel events (form_view, field_error, submit_attempt, submit_success, submit_failure) — see **ios-crash-analytics**. Never log field values or PII.

Write a **form spec** before coding:

```
Form: Edit profile        Presented: pushed (tab: Profile)    Mode: edit
Fields: name(req, 1–50) · email(req, email, server-unique) · phone(opt, E.164 mask) · bio(opt, ≤160)
Actions: nav-bar Save (R1+R2) · Return on last field = Save · Back → discard prompt if dirty
Server errors: 422 {errors:{field:[msg]}}   Offline: block submit + banner   Draft: no
```

## 2. R1 — End editing first

**Why it matters.** Several field kinds only write the model when editing ends:
- SwiftUI `TextField(value:format:)` (numbers, currency, dates) commits on submit/focus loss, not per keystroke.
- UIKit fields that update the model in `textFieldDidEndEditing` or via a formatter/mask on end.
- Custom fields that trim/normalise on blur (`" a@b.com "` → `"a@b.com"`), and on-blur validation.
- Marked text (CJK/IME composition) is uncommitted until editing ends.
- `UITextView` / undo grouping and autocorrect suggestions are applied when the responder resigns.

If Save reads the model while the field is still first responder you save the previous value, skip formatting, or validate stale input.

**SwiftUI API**
- `@FocusState private var focus: Field?` and `.focused($focus, equals: .email)` on each field.
- End editing: `focus = nil`. Then **yield one runloop** (`await Task.yield()` / `try? await Task.sleep(for: .milliseconds(1))`) so the binding commit and `onChange` handlers run before reading the model. Verify on your target OS; the safest belt-and-braces is also sending `resignFirstResponder` through the responder chain (A.2) which forces any hosted `UITextField` to commit.
- Return key: `.submitLabel(.next / .done / .go / .send)` + `.onSubmit { … }`. The last field's `.onSubmit` calls the **same** `submit()` as the button — which goes through the wrapper.

**UIKit API**
- `view.endEditing(true)` (forces resign on any first responder in the hierarchy) or `textField.resignFirstResponder()`.
- Global: `UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)`.
- Return key: `textFieldShouldReturn(_:)` → next field `becomeFirstResponder()` or `submit()` via the wrapper; return `false`.
- Bar buttons: `UIBarButtonItem(primaryAction: UIAction { [weak self] _ in self?.submit() })` — never call the API directly.

**The shared submit wrapper** (A.3, `FormSubmitter`) is the only way a form action runs. Rules:
- Every button, Return/Done handler, toolbar item, keyboard shortcut (`.keyboardShortcut(.defaultAction)`) and "Next" step goes through `submitter.run { … }`.
- The wrapper ends editing itself; callers do not need to remember.
- Lint in review: grep for `await api.` / `repository.save` called from a view or `@IBAction` not inside `run`.

Checklist:
1. - [ ] Every form action (buttons, Return, toolbar, nav-bar, keyboard shortcut, step Next) calls the shared wrapper.
2. - [ ] Wrapper ends editing (`focus = nil` + responder resign) and yields before reading the model.
3. - [ ] Formatted/masked fields commit on end editing; trimmed/normalised values are what gets validated and sent.
4. - [ ] Keyboard is dismissed before the HUD appears.
5. - [ ] Test exists: type in last field, tap Save without leaving it → latest value saved.

## 3. R2 — Full-page blocking HUD

**Why an overlay window.** A SwiftUI `.overlay` or a UIKit subview added to the form's own view covers only that view: the nav bar back button, the tab bar and the swipe-back edge are still live. Put the HUD in a **dedicated `UIWindow` above the app window** (`windowLevel = .alert + 1`) attached to the active `UIWindowScene` (A.4). It covers nav bar, tab bar, sheets and the status-bar area, and absorbs every touch including the edge-swipe start. Works for SwiftUI and UIKit.

Defence in depth — also disable back paths at the source while submitting (a window does not stop programmatic pops, hardware keyboard Esc/⌘[, or a sheet you present from code):

| Back path | SwiftUI | UIKit |
|---|---|---|
| Nav back button | `.navigationBarBackButtonHidden(submitter.isRunning)` | `navigationItem.hidesBackButton = true` |
| Swipe-back edge gesture | hiding the back button also disables it in SwiftUI (verify on target OS); overlay window blocks touch anyway | `navigationController?.interactivePopGestureRecognizer?.isEnabled = false` (restore in `defer`) |
| Sheet swipe-to-dismiss | `.interactiveDismissDisabled(submitter.isRunning \|\| isDirty)` | `isModalInPresentation = true` (+ `presentationControllerShouldDismiss` delegate) |
| Cancel / Close buttons | `.disabled(submitter.isRunning)` | `isEnabled = false` |
| Tab bar | covered by window | covered by window |
| Hardware keyboard (Esc, ⌘., ⌘[) | `.disabled` on shortcut buttons | override `pressesBegan` or disable `UIKeyCommand`s |
| Deep link / push opens another screen | router defers navigation while `isRunning` (ios-deep-links) | same |

Rules:
- **Re-entrancy**: `isRunning` guard at the top of `run`; second taps are no-ops (not queued).
- **Minimum display**: if shown, keep visible ≥ 400–600 ms; optionally delay showing by ~150 ms for very fast calls — but touches must be blocked from the first instant (install the window immediately with a transparent background, fade the spinner in).
- **Guaranteed dismissal**: `defer { hud.hide() }` around the action, including `CancellationError` and timeouts. Set a request timeout (ios-network-offline-sync) so the HUD can't spin forever. If the screen is torn down mid-request (`onDisappear`/`deinit`), cancel the task and hide the HUD.
- **One HUD at a time**: a reference-counted presenter so nested actions don't hide each other's HUD early.
- **Accessibility**: `UIAccessibility.post(notification: .announcement, argument: "Saving…")` on show; overlay root `accessibilityViewIsModal = true` so VoiceOver can't reach elements below; spinner has a label; on hide post `.screenChanged` to the result (error banner or next screen). Respect Reduce Motion (no fancy scale animations).
- Don't use `fullScreenCover`/`present` for the HUD: presenting changes the presentation stack, can fail if something else is presenting, and fights with sheets and alerts.

Checklist:
1. - [ ] HUD is a window-level overlay that covers nav bar, tab bar and sheets; all touches blocked.
2. - [ ] Back button hidden, swipe-back disabled, sheet dismiss disabled, Cancel disabled while running; restored afterwards.
3. - [ ] Re-entrant submits ignored; double tap → one request (verify server log).
4. - [ ] Min display time; no flash on fast responses.
5. - [ ] Hidden in `defer` on success, error, cancellation, timeout, and screen teardown.
6. - [ ] VoiceOver announces "Saving…"; focus trapped; focus moves to result afterwards.

## 4. Submit pipeline

```
endEditing → validateAll → (invalid? R3 alert + inline, focus first error after OK, stop) → HUD.show → API → HUD.hide
  → success: dismiss/pop/navigate, or toast "Saved"; reset dirty baseline
  → failure: field errors → map to fields + focus first; general error → banner/alert with Retry
```

1. - [ ] One reusable coordinator (`FormSubmitter`) owns `isRunning`, HUD and focus-first-error; screens provide only `validate()` and the `action`.
2. - [ ] Validation failure does **not** show the HUD and does not hit the network.
3. - [ ] Idempotency key per submit attempt for orders/payments; reused on retry of the same attempt.
4. - [ ] Navigation on success happens after the HUD is hidden (avoid the HUD window lingering over the next screen).
5. - [ ] Success resets the dirty baseline **before** popping so the discard prompt doesn't fire.
6. - [ ] Errors are user-language; raw `DecodingError`/HTTP codes are logged, not shown (ios-crash-analytics).
7. - [ ] Offline: either block with a clear message or enqueue and tell the user (ios-network-offline-sync).
8. - [ ] Haptic: `.sensoryFeedback(.success/.error, trigger:)` / `UINotificationFeedbackGenerator`.

## 5. Validation

- **When**: on blur (focus leaves a field that was edited) and on submit. Not on every keystroke — except to **clear** an error once the input becomes valid, and live helpers like password strength.
- **Where**: pure functions (`Validator`) on the model, unit-testable, shared rules mirrored from the backend. The server is still the authority.
- **Field-level display (R3, before submit)**: inline only — footer error text under the field (red + icon, not colour alone), red border, short horizontal shake (`.modifier(Shake)` / `CAKeyframeAnimation` on `transform.translation.x`, skipped when `accessibilityReduceMotion` / `UIAccessibility.isReduceMotionEnabled`), or a poptip anchored to the field. Error text is part of the field's accessibility value. Never an alert.
- **Submit display (R3)**: native alert — title ("Check your details"), concise message (one line per issue, max ~3, or the first issue). The alert announces itself to VoiceOver. Inline errors are set too, so the user can see them after dismissing.
- **First error**: on alert dismiss, scroll to and focus the first invalid field in visual order (`ScrollViewReader.scrollTo` + `focus = .field`; UIKit `scrollRectToVisible` + `becomeFirstResponder`). Don't set focus while the alert is up (it steals the keyboard back).
- **Server field errors (R3)**: after the HUD hides, show them in a native alert and map keys (`email`, `address.zip`) to `Field` cases inline; unknown keys appear in the alert message only. Clear a server error when the user edits that field.

Checklist:
1. - [ ] Validate on blur + submit; errors clear as soon as fixed.
2. - [ ] No error shown on untouched fields before the first submit.
3. - [ ] R3: submit failure shows a native alert (title + concise list) **and** inline errors; first invalid field focused after OK.
4. - [ ] R3: server 422 errors → native alert after HUD hides + inline on the right fields; unknown keys in the alert.
5. - [ ] R3: blur/editing errors are inline only (text/border/shake or poptip); shake respects Reduce Motion; no alert.
6. - [ ] Validation runs on the committed, normalised value (R1).

## 6. Keyboard

- **Avoidance**: SwiftUI `Form`/`ScrollView` avoid the keyboard automatically; don't fight it with manual padding. Use `.ignoresSafeArea(.keyboard)` only for backgrounds. UIKit: pin the scroll view's bottom to `view.keyboardLayoutGuide.topAnchor` (iOS 15+), or adjust `contentInset`/`verticalScrollIndicatorInsets` from `keyboardWillChangeFrame`.
- **Scroll focused field into view**: SwiftUI `ScrollViewReader` + `onChange(of: focus)` → `proxy.scrollTo(field, anchor: .center)`. UIKit `scrollRectToVisible(field.frame.insetBy(dx: 0, dy: -16))` after the inset change.
- **Chaining**: `.submitLabel(.next)` on all but the last; `.onSubmit { focus = next }`; last field `.submitLabel(.done/.go)` → `submit()`. UIKit `returnKeyType` + `textFieldShouldReturn`.
- **Tap outside / scroll to dismiss**: `.scrollDismissesKeyboard(.interactively)` (iOS 16); UIKit `keyboardDismissMode = .interactive` and a tap gesture with `cancelsTouchesInView = false` calling `view.endEditing(true)`. Dismissing the keyboard this way is also an end-editing event (commit + blur validation).
- **Keyboard toolbar**: `.toolbar { ToolbarItemGroup(placement: .keyboard) { … } }` for number pads (no Return key) — its Done goes through the wrapper when it submits, else just `focus = nil`. UIKit `inputAccessoryView`.

Checklist:
1. - [ ] Focused field never hidden by the keyboard (iPhone SE, landscape, iPad floating keyboard).
2. - [ ] Next chains in visual order; last field submits through the wrapper.
3. - [ ] Number/phone pads have a toolbar Done.
4. - [ ] Tap outside / interactive scroll dismisses the keyboard without swallowing button taps.

## 7. Field types

| Field | SwiftUI | UIKit | Notes |
|---|---|---|---|
| Email | `.keyboardType(.emailAddress)`, `.textContentType(.emailAddress)`, `.textInputAutocapitalization(.never)`, `.autocorrectionDisabled()` | same props on `UITextField` | trim on blur |
| Username (login) | `.textContentType(.username)` | `.username` | needed for Password AutoFill pairing |
| Password (login) | `SecureField` + `.textContentType(.password)` | `isSecureTextEntry`, `.password` | show/hide toggle keeps focus |
| New password | `.textContentType(.newPassword)` | `.newPassword` + `passwordRules` | strong password suggestion |
| OTP | `.textContentType(.oneTimeCode)`, `.keyboardType(.numberPad)` | same | auto-submit on full length via wrapper |
| Phone | `.keyboardType(.phonePad)`, `.textContentType(.telephoneNumber)` | same | mask display, store E.164 |
| Name / address | `.givenName`, `.familyName`, `.fullStreetAddress`, `.postalCode`, `.addressCity` | same | autofill from contacts |
| Card | `.textContentType(.creditCardNumber)` (+ expiry/CSC types on newer iOS, verify) | same | prefer payment SDK / Apple Pay |
| Date / choice | `DatePicker`, `Picker`, `Toggle` | `UIDatePicker`, `UIMenu`/`UIPickerView` as `inputView` | pickers also end text editing |
| Multiline | `TextField(axis: .vertical)` / `TextEditor` | `UITextView` | Return inserts newline; submit via button only |

Masks: format for display, keep the raw value in the model; preserve caret position (UIKit: rebuild `selectedTextRange`; SwiftUI: prefer formatting on blur rather than per keystroke to avoid caret jumps). Never break paste and autofill (a 10-digit paste into a masked field must work).

Checklist:
1. - [ ] Every field has the right keyboard type and content type; login has username + password pair.
2. - [ ] Secure fields: no autocorrect, not logged, not in screenshots of analytics.
3. - [ ] Masks keep caret sane and accept paste/autofill.
4. - [ ] OTP auto-submit goes through the wrapper.

## 8. Unsaved changes

- **Dirty tracking**: compare current field values to a baseline snapshot (`Equatable` struct), not a "was edited" flag.
- **Back with changes**: hide the system back button and show a custom one, or (UIKit) implement a back-intercept; on tap show "Discard changes?" (Discard destructive / Keep editing). Sheets: `.interactiveDismissDisabled(isDirty)` + attempt handler (UIKit `presentationControllerDidAttemptToDismiss`) shows the same dialog.
- **Never during submit**: while `isRunning` the HUD blocks back, so the prompt can't appear; after success the baseline resets first, so pop is silent.
- **Drafts (optional)**: persist debounced draft for long forms (feedback, listing creation) keyed by form + account; restore with "Restored your draft"; delete on success. Never persist passwords/card data.

Checklist:
1. - [ ] Dirty = value comparison with baseline; editing back to original → not dirty.
2. - [ ] Back button, swipe-back and sheet swipe all route to the same discard prompt when dirty.
3. - [ ] No discard prompt during submit or after success.
4. - [ ] Drafts exclude secrets and are cleared on success/logout.

## 9. Multi-step forms

1. - [ ] One model across steps (owned by the flow, not each step view) so Back keeps entered values.
2. - [ ] Each Next follows R1 (end editing, validate the step); if Next calls the server (e.g. check email availability, send OTP), it also follows R2.
3. - [ ] Progress indicator ("Step 2 of 4") with accessibility label.
4. - [ ] Back between steps doesn't trigger discard; leaving the whole flow does.
5. - [ ] Final submit re-validates all steps; server errors can navigate back to the step containing the field.
6. - [ ] State restoration / backgrounding doesn't lose progress (scene storage or draft).

## 10. Accessibility

1. - [ ] Every field has a visible label (not placeholder-only) and an accessibility label; hints for format ("10 digits").
2. - [ ] Errors are read with the field (`accessibilityValue`/hint) and announced on submit.
3. - [ ] Focus order equals visual order; VoiceOver rotor reaches all fields.
4. - [ ] Dynamic Type up to AX5: labels wrap, fields grow, nothing clipped; `Form` handles it, custom layouts use `ViewThatFits`/stacking.
5. - [ ] HUD announces, traps focus, and restores focus to the result.
6. - [ ] Colour is not the only error signal; contrast ≥ 4.5:1.
7. - [ ] Voice Control: buttons have names matching visible text.

## 11. Testing plan

1. **Unit tests**: validators, dirty tracking, server-error mapping, `FormSubmitter` (re-entrancy, HUD hidden on throw/cancel, min display using an injected clock).
2. **UI tests (XCUITest)**: type in a field and tap Save without leaving it; Return-key submit; double-tap Save; back/swipe during HUD (stub a slow API with launch arguments); discard prompt.
3. **Network Link Conditioner** / debug delay switch to keep the HUD up long enough to test every back path by hand.
4. **Manual passes**: VoiceOver, Dynamic Type AX5, hardware keyboard on iPad, Password AutoFill and SMS OTP on a real device, CJK input.

### Scenario matrix

| # | Scenario | Expected |
|---|---|---|
| F1 | Type in last field, tap nav-bar Save while still focused | Latest value saved (R1) |
| F2 | Return/Done on last field | Ends editing, validates, submits once |
| F3 | Formatted number field, tap Save while focused | Formatted committed value sent |
| F4 | CJK marked text, tap Save | Composition committed before save |
| F5 | Double/triple tap Save | Exactly one request |
| F6 | Tap back button during HUD | Nothing happens |
| F7 | Swipe from left edge during HUD | No pop |
| F8 | Swipe down sheet during HUD | Sheet stays |
| F9 | Tap tab bar / other buttons during HUD | Ignored |
| F10 | Fast response (<100 ms) | No HUD flash; touches still blocked during call |
| F11 | Server error | HUD hidden; error shown; form editable |
| F12 | Timeout / offline | HUD hidden; retry possible |
| F13 | Leave screen / task cancelled mid-submit | HUD hidden; no orphan window |
| F14 | Invalid fields, tap Save | No HUD, no request; native alert listing issues + inline errors; first field focused after OK |
| F15 | Server 422 field errors | HUD hidden, then native alert; inline errors on correct fields; first focused after OK |
| F26 | Leave an invalid field (blur) | Inline error + shake (no shake with Reduce Motion); no alert |
| F16 | Dirty form, back / swipe / sheet swipe | Discard prompt |
| F17 | Edit then revert to original, back | No prompt |
| F18 | Successful save then pop | No discard prompt |
| F19 | Discard prompt during submit | Never shown |
| F20 | Next chain | Visual order; last submits |
| F21 | Keyboard on small device / landscape | Focused field visible |
| F22 | Password AutoFill / OTP from SMS | Fills; OTP auto-submits once |
| F23 | VoiceOver during HUD | "Saving…" announced; can't reach content below |
| F24 | Dynamic Type AX5 | Usable, no clipping |
| F25 | Multi-step Back then Next | Values kept; step re-validated |

## 12. Review checklist (summary)

1. - [ ] **R1**: every form action goes through the shared wrapper which ends editing first.
2. - [ ] **R2**: window-level blocking HUD; all back paths disabled; re-entrancy guard; min display; `defer` hide; a11y.
3. - [ ] **R3**: submit validation (client + server) → native alert + inline; field validation → inline only.
4. - [ ] Pipeline order: end editing → validate → HUD → API → hide → result.
5. - [ ] Validation on blur + submit; first error focused; server errors mapped.
6. - [ ] Keyboard avoidance, Next chain, tap-outside dismiss.
7. - [ ] Correct keyboard/content types, autofill, masks.
8. - [ ] Dirty tracking + discard prompt (never during submit).
9. - [ ] Accessibility and Dynamic Type.

## 13. Common bugs to look for in review

- **R1 violation**: Save reads the model while a field is first responder → previous value saved, untrimmed value, `TextField(value:format:)` not committed
- **R1 violation**: Return key calls the API directly while the nav-bar button uses the wrapper (or vice-versa)
- Ending focus and reading the model in the same synchronous step (no yield) → still stale on some fields
- **R2 violation**: HUD as `.overlay` on the form content → back button, tab bar and swipe-back still work during submit
- **R2 violation**: HUD shown with `present`/`fullScreenCover` → fails when an alert/sheet is already presented
- **Stuck HUD**: hide not in `defer`; early `return` or thrown error skips hide; cancellation not handled; no request timeout
- Swipe-back disabled for submit but never re-enabled (screen permanently stuck)
- Button disabled via `isSubmitting` set *after* an `await` → double tap sends two requests
- HUD flashes for 50 ms; or HUD delayed and touches not blocked during the delay
- Navigation on success while HUD window still visible over the next screen
- Validation on every keystroke → error shown while user is still typing the first character
- **R3 violation**: alert shown on blur / while typing; or submit failure shows only inline errors (user taps Save and "nothing happens")
- **R3 violation**: server field errors shown only as a generic alert (not mapped inline) or only inline (no alert); alert presented while HUD still up
- Focusing the first invalid field while the alert is visible; shake ignoring Reduce Motion
- Discard prompt shown during submit or right after a successful save
- Placeholder-only labels; errors only in red; VoiceOver not told about errors
- Masks that move the caret to the end or break paste/autofill
- Logging field values (emails, passwords) in analytics or crash breadcrumbs

## 14. Output format

When asked to **review/audit**:
1. **Summary** — framework, form spec (type, presentation, fields, actions, server error contract), min OS
2. **R1 / R2 / R3 verdict** first — every form action listed with ✅/❌ and file:line
3. **Checklist results** — sections 2–10 with ✅ Pass / ❌ Fail / ⚠️ Needs check and file:line
4. **Critical issues** (stale saves, double submits, back during submit, stuck HUD, lost data) with fix code
5. **Testing gaps** — F1–F25 not covered

When asked to **implement**: write the form spec, start from the Appendix (`FormSubmitter`, `BlockingHUD`, validators), wire every action through `submitter.run`, delete unused parts (drafts, multi-step), and call out server contract questions (error format, idempotency) and OS-gated APIs used.

When asked to **debug**: check R1 (is editing ended + yielded before read?), then R2 (who else can receive touches/back?), then R3 (alert on submit, inline on blur) and server-error mapping — in that order.

---

## Appendix — Reference implementation (iOS 17+, Swift 6, SwiftUI + UIKit)

Adjust for your deployment target (`ObservableObject` instead of `@Observable` below iOS 17).

### A.1 Form state model and validation helpers

```swift
import Foundation; import Observation

enum ProfileField: String, CaseIterable, Hashable, Sendable {
    case name, email, phone, bio            // visual order = CaseIterable order
}

struct ProfileValues: Equatable, Sendable {
    var name = "", email = "", bio = ""
    var phone = ""                          // raw digits; display is masked
}

enum Validator {
    static func required(_ s: String, _ label: String) -> String? { s.isEmpty ? "\(label) is required" : nil }
    static func maxLength(_ s: String, _ n: Int) -> String? { s.count > n ? "Maximum \(n) characters" : nil }
    static func email(_ s: String) -> String? {
        s.range(of: #"^[^@\s]+@[^@\s]+\.[^@\s]+$"#, options: .regularExpression) != nil
            ? nil : "Enter a valid email address"
    }
    static func phone(_ d: String) -> String? {
        d.isEmpty || (8...15).contains(d.count) ? nil : "Enter a valid phone number"
    }
}

@Observable @MainActor
final class ProfileFormModel {
    var values = ProfileValues()
    private(set) var baseline = ProfileValues()
    var errors: [ProfileField: String] = [:]
    var submitAlert: SubmitAlert?          // R3: submit-time native alert (client or server)
    var shakeCount: [ProfileField: Int] = [:]
    private var submittedOnce = false

    var isDirty: Bool { values != baseline }
    func load(_ v: ProfileValues) { values = v; baseline = v; errors = [:] }
    func markSaved() { baseline = values }

    func normalize() {
        values.name = values.name.trimmingCharacters(in: .whitespacesAndNewlines)
        values.email = values.email.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
        values.phone = values.phone.filter(\.isNumber)
    }

    func error(for f: ProfileField) -> String? {
        switch f {
        case .name:  Validator.required(values.name, "Name") ?? Validator.maxLength(values.name, 50)
        case .email: Validator.required(values.email, "Email") ?? Validator.email(values.email)
        case .phone: Validator.phone(values.phone)
        case .bio:   Validator.maxLength(values.bio, 160)
        }
    }

    func didBlur(_ f: ProfileField) {      // blur: inline validation only
        normalize()
        errors[f] = error(for: f)
        if errors[f] != nil { shakeCount[f, default: 0] += 1 }   // R3 inline shake, never an alert
    }

    func didEdit(_ f: ProfileField) {
        if errors[f] != nil || submittedOnce { errors[f] = error(for: f) }
    }

    func validateAll() -> ProfileField? {  // submit: returns first invalid field in visual order
        submittedOnce = true
        normalize()
        for f in ProfileField.allCases { errors[f] = error(for: f) }
        return ProfileField.allCases.first { errors[$0] != nil }
    }

    /// Map server 422 `{ "errors": { "email": ["Already taken"] } }` to fields.
    func apply(serverErrors: [String: [String]]) -> ProfileField? {
        var unknown: [String] = []
        for (key, msgs) in serverErrors {
            if let f = ProfileField(rawValue: key) { errors[f] = msgs.first }
            else { unknown.append(contentsOf: msgs) }
        }
        let first = ProfileField.allCases.first { errors[$0] != nil }
        presentSubmitAlert(first: first, extra: unknown)      // R3: alert + inline
        return first
    }

    func presentSubmitAlert(first: ProfileField?, extra: [String] = []) {
        let lines = ProfileField.allCases.compactMap { errors[$0] } + extra
        guard !lines.isEmpty else { return }
        let shown = lines.prefix(3).map { "• \($0)" }.joined(separator: "\n")
        submitAlert = SubmitAlert(title: "Check your details",
                                  message: lines.count > 3 ? shown + "\n…and \(lines.count - 3) more" : shown,
                                  focusOnDismiss: first)
    }
}

struct SubmitAlert: Identifiable { let id = UUID(); let title, message: String; let focusOnDismiss: ProfileField? }
enum FormSubmitError: Error { case fieldErrors([String: [String]]), message(String) }
```

### A.2 Ending editing (R1) for SwiftUI and UIKit

```swift
import UIKit

@MainActor
enum EditingEnder {
    /// Resign any first responder (incl. SwiftUI-hosted UITextFields: commits marked text, formatters),
    /// then let bindings / onChange / didEndEditing run before the model is read.
    static func endEditing(in view: UIView? = nil) async {
        if let view { view.endEditing(true) } else {
            UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder),
                                            to: nil, from: nil, for: nil)
        }
        await Task.yield()
        try? await Task.sleep(for: .milliseconds(1))   // one more runloop turn (verify needed on your OS)
    }
}
```

### A.3 Shared submit wrapper / coordinator

```swift
import SwiftUI

@Observable @MainActor
final class FormSubmitter {
    private(set) var isRunning = false
    private var task: Task<Void, Never>?
    private let hud: BlockingHUD

    init(hud: BlockingHUD = .shared) { self.hud = hud }
    /// The ONLY entry point for any form action (R1 + R2). `validate` returning false stops before
    /// HUD/network (and should raise the R3 alert); `action` throws `FormSubmitError` for field errors.
    func run(message: String = "Saving…",
             endEditing: @MainActor () -> Void,
             validate: @MainActor () -> Bool = { true },
             action: @escaping @MainActor () async throws -> Void,
             onSuccess: @escaping @MainActor () -> Void = {},
             onFailure: @escaping @MainActor (Error) -> Void = { _ in }) {
        guard !isRunning else { return }          // re-entrancy: ignore, don't queue
        isRunning = true                          // set synchronously, before any await
        endEditing()                              // R1 (SwiftUI focus = nil)
        task = Task { [weak self] in
            guard let self else { return }
            await EditingEnder.endEditing()       // R1 (responder resign + yield)
            guard validate() else { self.isRunning = false; return }
            let token = self.hud.show(message: message)   // R2
            var result: Result<Void, Error> = .success(())
            do { try await action() } catch { result = .failure(error) }
            await self.hud.hide(token)            // awaits min display; always runs
            self.isRunning = false
            switch result {
            case .success: onSuccess()            // navigate after HUD is gone
            case .failure(let e) where e is CancellationError: break
            case .failure(let e): onFailure(e)
            }
        }
    }

    /// Call from onDisappear / deinit paths: cancels work; HUD hides via the same path.
    func cancel() { task?.cancel() }
}
```

`hide` is reached on every path because `action()` errors are caught into `result`; nothing between `show` and `hide` can throw or return early. Keep it that way when editing.

### A.4 Blocking HUD (R2): overlay window + back blocking

```swift
import SwiftUI; import UIKit

@MainActor
final class BlockingHUD {
    static let shared = BlockingHUD()
    struct Token: Hashable { let id = UUID(); let shownAt: ContinuousClock.Instant }

    var minimumVisible: Duration = .milliseconds(500), spinnerDelay: Duration = .milliseconds(150)
    private var window: UIWindow?
    private var active: Set<Token> = []
    private let state = HUDState()

    func show(message: String) -> Token {
        let token = Token(shownAt: .now)
        active.insert(token)
        state.message = message
        if window == nil { installWindow() }
        let delay = spinnerDelay
        Task { [state] in                       // touches blocked now; spinner fades in later
            try? await Task.sleep(for: delay)
            if self.active.contains(token) { withAnimation { state.visible = true } }
        }
        UIAccessibility.post(notification: .announcement, argument: message)
        return token
    }

    func hide(_ token: Token) async {
        let elapsed = ContinuousClock.now - token.shownAt
        if state.visible, elapsed < minimumVisible {
            try? await Task.sleep(for: minimumVisible - elapsed)   // sleep error ignored: always hide
        }
        active.remove(token)
        guard active.isEmpty else { return }    // reference counted
        state.visible = false; window?.isHidden = true; window = nil
        UIAccessibility.post(notification: .screenChanged, argument: nil)
    }

    private func installWindow() {
        guard let scene = UIApplication.shared.connectedScenes
            .compactMap({ $0 as? UIWindowScene })
            .first(where: { $0.activationState == .foregroundActive }) else { return }
        let w = UIWindow(windowScene: scene)
        w.windowLevel = .alert + 1
        w.backgroundColor = .clear
        let host = UIHostingController(rootView: HUDView(state: state))
        host.view.backgroundColor = .clear
        host.view.accessibilityViewIsModal = true          // trap VoiceOver
        w.rootViewController = host
        w.isHidden = false                                  // not key: keyboard already dismissed
        window = w
    }
}

@Observable @MainActor
final class HUDState { var message = ""; var visible = false }

private struct HUDView: View {
    let state: HUDState
    var body: some View {
        ZStack {
            // Full-screen hit target: absorbs every touch, incl. edge swipe and tab bar.
            Color.black.opacity(state.visible ? 0.25 : 0.001)
                .ignoresSafeArea().contentShape(Rectangle()).onTapGesture {}
            if state.visible {
                VStack(spacing: 12) {
                    ProgressView().controlSize(.large)
                    if !state.message.isEmpty { Text(state.message).font(.headline) }
                }
                .padding(24)
                .background(.regularMaterial, in: RoundedRectangle(cornerRadius: 16))
                .accessibilityElement(children: .combine)
                .accessibilityLabel(state.message)
                .transition(.opacity)
            }
        }
    }
}

/// SwiftUI: disable back paths at the source too (defence in depth).
struct BlocksBackWhileSubmitting: ViewModifier {
    let isRunning: Bool, isDirty: Bool
    func body(content: Content) -> some View {
        content
            .navigationBarBackButtonHidden(isRunning)      // also disables swipe-back (verify)
            .interactiveDismissDisabled(isRunning || isDirty)
    }
}

/// R3 field-level feedback: short shake, skipped under Reduce Motion. Bump `trigger` on blur error.
struct Shake: ViewModifier {
    let trigger: Int
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    func body(content: Content) -> some View {
        content.keyframeAnimator(initialValue: 0.0, trigger: trigger) { view, x in
            view.offset(x: reduceMotion ? 0 : x)
        } keyframes: { _ in
            KeyframeTrack {
                LinearKeyframe(-8.0, duration: 0.06); LinearKeyframe(8.0, duration: 0.06)
                LinearKeyframe(-4.0, duration: 0.06); LinearKeyframe(0.0, duration: 0.06)
            }
        }
    }
}

/// UIKit: same for UIViewController-based forms.
extension UIViewController {
    func setFormBackNavigationBlocked(_ blocked: Bool) {
        navigationItem.hidesBackButton = blocked
        navigationController?.interactivePopGestureRecognizer?.isEnabled = !blocked
        isModalInPresentation = blocked
        navigationItem.leftBarButtonItems?.forEach { $0.isEnabled = !blocked }
        navigationItem.rightBarButtonItems?.forEach { $0.isEnabled = !blocked }
    }
}
```

UIKit-only: swap the hosting controller for a plain VC (full-bounds view + `UIActivityIndicatorView`); keep `accessibilityViewIsModal` and the window level.

### A.5 Sample SwiftUI screen wiring everything

```swift
import SwiftUI

protocol ProfileRepository: Sendable {
    func save(_ v: ProfileValues) async throws   // throws FormSubmitError.fieldErrors on 422
}

struct EditProfileScreen: View {
    let repository: ProfileRepository
    let initial: ProfileValues
    @Environment(\.dismiss) private var dismiss
    @State private var model = ProfileFormModel()
    @State private var submitter = FormSubmitter()
    @State private var showDiscard = false
    @FocusState private var focus: ProfileField?

    var body: some View {
        ScrollViewReader { proxy in
            Form {
                field(.name, "Name") {
                    TextField("Name", text: $model.values.name)
                        .textContentType(.name).submitLabel(.next)
                }
                field(.email, "Email") {
                    TextField("Email", text: $model.values.email)
                        .keyboardType(.emailAddress).textContentType(.emailAddress)
                        .textInputAutocapitalization(.never).autocorrectionDisabled()
                        .submitLabel(.next)
                }
                field(.phone, "Phone (optional)") {
                    TextField("Phone", text: $model.values.phone).keyboardType(.phonePad).textContentType(.telephoneNumber)
                }
                field(.bio, "Bio (optional)") {
                    TextField("Bio", text: $model.values.bio, axis: .vertical).lineLimit(3...6).submitLabel(.done)
                }
            }
            .scrollDismissesKeyboard(.interactively)
            .onSubmit { advance(from: focus) }               // Return chains / submits
            .onChange(of: focus) { old, new in
                if let old { model.didBlur(old) }           // validate on blur
                if let new { withAnimation { proxy.scrollTo(new, anchor: .center) } }
            }
        }
        .navigationTitle("Edit profile")
        .toolbar {
            ToolbarItem(placement: .confirmationAction) { Button("Save", action: submit).disabled(submitter.isRunning) }
            ToolbarItem(placement: .cancellationAction) { Button("Cancel", action: attemptClose).disabled(submitter.isRunning) }
            ToolbarItemGroup(placement: .keyboard) { Spacer(); Button("Done") { focus = nil } }  // phone pad
        }
        .modifier(BlocksBackWhileSubmitting(isRunning: submitter.isRunning, isDirty: model.isDirty))
        .navigationBarBackButtonHidden(true)                // custom Cancel handles discard prompt
        .confirmationDialog("Discard changes?", isPresented: $showDiscard, titleVisibility: .visible) {
            Button("Discard", role: .destructive) { dismiss() }
            Button("Keep editing", role: .cancel) {}
        }
        .alert(model.submitAlert?.title ?? "", isPresented: Binding(
                get: { model.submitAlert != nil }, set: { if !$0 { model.submitAlert = nil } }),
               presenting: model.submitAlert) { alert in
            Button("OK") { focus = alert.focusOnDismiss }   // focus AFTER dismiss; alert announced itself
        } message: { Text($0.message) }
        .onAppear { if !model.isDirty { model.load(initial) } }
        .onDisappear { submitter.cancel() }
    }

    private func submit() {
        submitter.run(
            endEditing: { focus = nil },
            validate: {
                if let first = model.validateAll() {           // R3: alert + inline, focus after OK
                    model.presentSubmitAlert(first: first); return false
                }
                return true
            },
            action: { [values = model.values, repository] in try await repository.save(values) },
            onSuccess: { model.markSaved(); dismiss() },     // baseline reset before pop
            onFailure: { error in
                switch error {
                case FormSubmitError.fieldErrors(let map):
                    _ = model.apply(serverErrors: map)          // HUD already hidden → alert + inline
                default:
                    let m = (error as? FormSubmitError).flatMap { if case .message(let m) = $0 { m } else { nil } }
                    model.submitAlert = SubmitAlert(title: "Couldn't save",
                        message: m ?? "Check your connection and try again.", focusOnDismiss: nil)
                }
            })
    }

    private func advance(from field: ProfileField?) {
        guard let field, let i = ProfileField.allCases.firstIndex(of: field) else { return }
        let next = ProfileField.allCases.index(after: i)
        if next < ProfileField.allCases.endIndex { focus = ProfileField.allCases[next] }
        else { submit() }                                   // last field Return → same wrapper
    }

    private func attemptClose() { if model.isDirty { showDiscard = true } else { dismiss() } }

    @ViewBuilder
    private func field<C: View>(_ f: ProfileField, _ label: String,
                                @ViewBuilder _ content: () -> C) -> some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(label).font(.subheadline).foregroundStyle(.secondary)
            content()
                .focused($focus, equals: f)
                .onChange(of: model.values) { model.didEdit(f) }
                .padding(6)
                .overlay(RoundedRectangle(cornerRadius: 8)            // R3 inline: red border
                    .stroke(model.errors[f] == nil ? .clear : .red, lineWidth: 1))
                .modifier(Shake(trigger: model.shakeCount[f, default: 0]))
                .accessibilityLabel(label)
                .accessibilityValue(model.errors[f].map { "Error: \($0)" } ?? "")
            if let err = model.errors[f] {        // R3 inline footer
                Label(err, systemImage: "exclamationmark.circle").font(.footnote).foregroundStyle(.red)
            }
        }
        .id(f)
    }
}
```

### A.6 UIKit wiring (same wrapper, R3 alert)

Fields: `textContentType` `.username`/`.password`, `returnKeyType` `.next`/`.go`, `textFieldShouldReturn` → next field or `submit()`; nav-bar button `UIBarButtonItem(primaryAction: UIAction { [weak self] _ in self?.submit() })`.

```swift
private func submit() {
    submitter.run(
        message: "Logging in…",
        endEditing: { [view] in view?.endEditing(true) },                     // R1
        validate: { [weak self] in
            guard let self else { return false }
            var issues: [(UITextField, String)] = []
            if (email.text ?? "").isEmpty { issues.append((email, "Enter your email")) }
            if (password.text ?? "").isEmpty { issues.append((password, "Enter your password")) }
            guard let first = issues.first else { return true }
            showAlert("Check your details", issues.map { "• \($0.1)" }.joined(separator: "\n"),
                      focus: first.0)                                         // R3
            return false
        },
        action: { [weak self] in
            guard let self else { return }
            setFormBackNavigationBlocked(true)                                // R2 defence in depth
            defer { setFormBackNavigationBlocked(false) }
            try await login(email.text ?? "", password.text ?? "")
        },
        onSuccess: { [weak self] in self?.dismiss(animated: true) },
        onFailure: { [weak self] _ in                                         // HUD already hidden
            self?.showAlert("Couldn't log in", "Check your details and try again.", focus: nil)
        })
}

private func showAlert(_ title: String, _ message: String, focus field: UITextField?) {
    let alert = UIAlertController(title: title, message: message, preferredStyle: .alert)
    alert.addAction(UIAlertAction(title: "OK", style: .default) { _ in field?.becomeFirstResponder() })
    present(alert, animated: true)
}
```

Inline (blur) errors in UIKit: in `textFieldDidEndEditing` set a footer `UILabel`, `layer.borderColor = UIColor.systemRed.cgColor`, and shake via `CAKeyframeAnimation(keyPath: "transform.translation.x")` unless `UIAccessibility.isReduceMotionEnabled`.
