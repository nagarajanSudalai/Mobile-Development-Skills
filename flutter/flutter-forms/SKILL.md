---
name: flutter-forms
description: Use when implementing, reviewing, testing, or debugging forms in a Flutter consumer app — login, sign-up, profile edit, checkout, multi-step flows, forms in pushed routes or showModalBottomSheet. Covers ending editing before every submit (FocusManager.instance.primaryFocus?.unfocus / FocusScope.of(context).unfocus, Form.save), Form / FormField / GlobalKey<FormState> validation on blur and on submit, server field errors, TextInputAction + onFieldSubmitted field chaining, keyboard insets and scroll-into-view, keyboardType / autofillHints / AutofillGroup / TextInput.finishAutofillContext, input formatters and masks, a full-page blocking HUD (Stack + ModalBarrier / AbsorbPointer) that also blocks back via PopScope(canPop: false) — system back, Android predictive back, iOS swipe-back — and sheet dismissal (isDismissible / enableDrag), unsaved-changes prompts, drafts and accessibility. Siblings: ios-forms, android-forms, react-native-forms; related: flutter-network-offline-sync, flutter-crash-analytics, flutter-list-ux, flutter-deep-links.
---

# Flutter Forms Skill

Guide for building, auditing and testing **forms** in a Flutter consumer app: single-screen forms (login, sign-up, profile, address, checkout), multi-step flows, and forms inside bottom sheets or dialogs. Siblings: **ios-forms**, **android-forms**, **react-native-forms**. Related: **flutter-network-offline-sync** (the submit API call, retries, offline queueing), **flutter-crash-analytics** (form funnel events, submit failures), **flutter-list-ux** (long scrollable forms, keyboard + scroll), **flutter-deep-links** (forms opened from a link, prefilled fields).

> **Two non-negotiable rules apply to EVERY form in the app.** No exceptions for "simple" forms, sheets or settings screens.
>
> **R1 — END EDITING FIRST.** Every Done / Save / Submit / Next / Complete tap, keyboard Return/Done (`onFieldSubmitted`), app-bar action or toolbar button that acts on the form must **end editing before calling its function**: unfocus the primary focus (commits IME composing text, fires focus-loss formatters/validation, dismisses the keyboard), then `FormState.save()`, then read values. Enforce it with **one shared submit wrapper** — no button may call the API directly.
>
> **R2 — FULL-PAGE BLOCKING HUD while any form action runs.** A full-screen overlay (spinner + optional message) that blocks **all** touches underneath (double taps, other buttons, bottom nav, app bar) **and** all back navigation (app-bar back, Android system back + predictive back, iOS swipe-back, sheet drag/barrier dismiss, hardware back). Re-entrant submits are ignored. Minimum display time (~400 ms) to avoid flash. **Always** dismissed on success, error and cancellation (`try/finally`). Announced to screen readers ("Saving…") with focus trapped inside.
>
> **R3 — SUBMIT VALIDATION = NATIVE ALERT; FIELD VALIDATION = INLINE.** When validation fails on submit (client rules, or server field errors returned after the HUD hides), show a platform-native alert via `showAdaptiveDialog` + `AlertDialog.adaptive` (Cupertino alert on iOS/macOS, Material on Android) with a title and concise message listing what's wrong (several → bullet list); after dismiss, focus + scroll to the first invalid field. Field-level validation before submit (on focus loss / while editing) is shown **inline only**: `InputDecoration.errorText` under the field + error border, with a short shake (skipped when `MediaQuery.disableAnimationsOf` is true), or a tooltip anchored to the field. Never a dialog for field-level validation; never only inline errors on submit. Server field errors also map onto inline errors.

Form packages and framework APIs move. Before shipping, **verify the current API** for your `flutter --version`: `PopScope` (`onPopInvoked` was deprecated in Flutter 3.22 in favour of `onPopInvokedWithResult`), `TapRegion` / `TextField.onTapOutside` (3.10+), `SemanticsService.announce` (newer releases add a view-aware `sendAnnouncement`; check deprecation), `AutofillHints` constants, `showModalBottomSheet` parameters (`useSafeArea`, `showDragHandle`, `isScrollControlled`), and any form package you use (`flutter_form_builder`, `reactive_forms`, `mask_text_input_formatter`). Check `pubspec.lock`; don't trust memory.

Version gates worth knowing:
- `WillPopScope` is deprecated (3.12+); it also disabled iOS swipe-back for every route and doesn't work with Android 14 predictive back. Use `PopScope`.
- Android predictive back (Android 14+ / API 34) requires `android:enableOnBackInvokedCallback="true"` in the manifest; then `PopScope.canPop` must be set **before** the gesture starts — you cannot veto it asynchronously mid-gesture.
- iOS swipe-back on `MaterialPageRoute`/`CupertinoPageRoute` is disabled automatically while the route has a `PopScope(canPop: false)` (route `popDisposition` = doNotPop). Custom `PageRoute`s / go_router `CustomTransitionPage` must respect it — test.
- `TextInput.finishAutofillContext()` exists since Flutter 1.22; iOS password-save prompt needs Associated Domains (`webcredentials:`) configured.

## 0. Mental model

```
 tap Submit / IME Done / app-bar Save
        │
        ▼
 FormController.submit(action)            ◄── single entry point (R1 + R2)
   1. if busy → return                    (re-entrancy guard)
   2. FocusManager.instance.primaryFocus?.unfocus()   (R1: commit + hide keyboard)
   3. await endOfFrame                    (let focus listeners / formatters run)
   4. formKey.currentState!.validate()  → fail: R3 adaptive alert listing errors,
                                           on dismiss focus + scroll first error, STOP
   5. formKey.currentState!.save()      → build immutable payload
   6. busy = true  → HUD shows (Stack overlay: ModalBarrier + spinner)
                   → PopScope(canPop: !busy) blocks system/predictive/swipe back
                   → sheet: isDismissible/enableDrag already false, PopScope inside
   7. await action(payload)  (flutter-network-offline-sync)
   8. finally: wait min display, busy = false → HUD hides
   9. success → finishAutofillContext, navigate/toast
      error   → map server field errors → inline errorText AND R3 adaptive alert
                → after dismiss focus first invalid field
```

Rules that follow from the model:
- **Values are read only after step 5.** Never read controllers in the button's `onPressed` before unfocus.
- **Busy state lives in one place** (the controller) and drives HUD, `PopScope.canPop`, and button enablement together.
- **The HUD is above everything that can be tapped** — including `BottomNavigationBar` / `NavigationBar` and the app bar. A spinner inside the body is not a HUD.

## 1. Context questions

Ask (or infer from code) before writing anything:
1. Form type: login, sign-up, profile edit, address, checkout/payment, multi-step onboarding?
2. Presentation: pushed route, full-screen dialog, `showModalBottomSheet`, nested in a tab of a shell route (go_router `StatefulShellRoute`)?
3. Create vs edit: is there an initial value (dirty tracking needed), can the user discard?
4. Offline: submit must queue offline (see **flutter-network-offline-sync**) or fail fast?
5. State management in use (setState, ChangeNotifier, Riverpod, Bloc)? Fit the controller to it.
6. Server-side validation: does the API return per-field errors (shape?) and which are retriable?
7. Autofill / password managers: login & sign-up must save credentials; one-time codes via SMS?
8. Accessibility & localisation: RTL, large text (textScaler up to 2.0+), screen readers.
9. Analytics: which funnel events (start, field error, submit, success, failure) — see **flutter-crash-analytics**.

## 2. R1 — End editing first

Why it matters in Flutter:
- `TextEditingController.text` updates per keystroke, **but** IME composing text (CJK, Korean, swipe/predictive keyboards, iOS autocorrect replacement) may not be committed until focus is lost — saving before unfocus can store the pre-correction or partial text.
- Formatting/normalisation that runs on focus loss (trim, currency format, phone formatting in a `Focus.onFocusChange`) hasn't run yet.
- `FormField.onSaved` only fires on `FormState.save()`; fields with `AutovalidateMode.onUserInteraction` may not have validated the last edit.
- The keyboard stays up over the HUD/next screen otherwise.

API:
- `FocusManager.instance.primaryFocus?.unfocus()` — works anywhere, including from a controller with no `BuildContext`. Preferred.
- `FocusScope.of(context).unfocus()` — unfocuses within that scope; fine inside the screen, but in a sheet/dialog the scope may differ from the field's. Avoid `FocusScope.of(context).requestFocus(FocusNode())` (old hack, leaks nodes).
- `UnfocusDisposition.scope` (default) vs `previouslyFocusedChild` — use default for submit.
- After unfocus, `await WidgetsBinding.instance.endOfFrame` so focus-change listeners and `setState`s settle before validation.

Checklist:
- [ ] Every action that reads form values goes through `FormController.submit` (or `runAction` for Next/step). Grep for `onPressed: () => api.` / `onPressed: _save` that bypass it.
- [ ] Wrapper unfocuses **first**, awaits end of frame, **then** validates, saves, reads values.
- [ ] Last field's `onFieldSubmitted` (TextInputAction.done / go / send) calls the same wrapper, not a separate path.
- [ ] App-bar actions (`AppBar.actions` Save), `FloatingActionButton`, sheet "Done", and toolbar buttons use the wrapper.
- [ ] Buttons that steal no focus (app-bar `IconButton`) still trigger unfocus — they don't take focus themselves, so the text field stays focused unless you unfocus explicitly.
- [ ] No value is captured in a closure before unfocus (`final email = _email.text; submit(...)` in onPressed is the classic stale-value bug).

## 3. R2 — Full-page blocking HUD

Implementation (see appendix `BlockingHud`, `HudScope`):
- **Overlay placement**: wrap the route's `Scaffold` in a `Stack`; the HUD layer is `Positioned.fill` containing `ModalBarrier(dismissible: false, color: Colors.black26)` + centered spinner card. Because it wraps the whole Scaffold it covers the app bar and the screen's own bottom bar.
- **Shell navigation**: if the bottom nav lives in a parent shell (go_router `StatefulShellRoute`, `IndexedStack` host), a route-level Stack does **not** cover it. Put the HUD in `MaterialApp.builder` (global `HudScope`) or show it through the root `Overlay` / `showDialog(useRootNavigator: true, barrierDismissible: false)`. Pick one pattern app-wide.
- **Touch blocking**: `ModalBarrier` absorbs pointers. Additionally wrap the form in `AbsorbPointer(absorbing: busy)` so nothing (e.g. hit-test-transparent widgets, keyboard toolbar) reaches it; and `ExcludeSemantics`/`BlockSemantics` so screen readers can't activate it.
- **Back blocking**: `PopScope(canPop: !busy, onPopInvokedWithResult: ...)` on the route. This covers Android system back, predictive back (no peek animation while canPop is false), the app-bar `BackButton` (it calls `maybePop`), and iOS swipe-back (disabled while canPop is false). Hardware/keyboard Escape also goes through `maybePop`.
- **Bottom sheet**: `showModalBottomSheet(isDismissible: false, enableDrag: false, ...)` while the form is a sheet that can submit — or bind them to busy via a sheet-internal `PopScope(canPop: !busy)` + `AbsorbPointer`. `isDismissible`/`enableDrag` are fixed when the sheet opens, so for "dismissible until submitting" set them true and rely on `PopScope(canPop: !busy)` inside the sheet, which blocks barrier tap and drag-to-close on current Flutter (verify on your version; drag-to-close calls `Navigator.maybePop` since 3.x). Test both.
- **Dialogs**: `showDialog(barrierDismissible: false)` + `PopScope(canPop: false)` inside.
- **Direct `Navigator.pop()` bypasses PopScope.** Grep for programmatic pops (timers, deep links via **flutter-deep-links**, logout redirects) and defer them while busy.
- **Re-entrancy**: `if (_busy) return;` at the top of submit, set busy synchronously before the first `await`. Button `onPressed: busy ? null : submit` is a second layer, not the guard.
- **Minimum display**: record start time; in `finally` await `max(0, 400ms - elapsed)`. Optional delay-before-show (~150 ms) for very fast calls is fine **only** if touches/back are blocked from t=0 (block immediately, show visuals later).
- **Guaranteed dismissal**: `try { ... } finally { busy = false; }` around the action; also handle `mounted` checks and dispose (controller dispose must not leave a global HUD up). Add a timeout on the API call (flutter-network-offline-sync) so the HUD cannot spin forever.
- **Accessibility**: HUD wrapped in `Semantics(liveRegion: true, label: 'Saving…')`; call `SemanticsService.announce('Saving…', textDirection)` on show; `BlockSemantics` hides the content behind; after hide, announce result and move focus to the first error or the success target.

Sheet presentation:

```dart
Future<bool?> showEditNameSheet(BuildContext context) =>
    showModalBottomSheet<bool>(
      context: context,
      isScrollControlled: true, // grows above keyboard
      useSafeArea: true,
      // Fixed at open time; keep false for submit-capable sheets, or true and
      // rely on BlockingHud's PopScope(canPop: !busy) — verify on your version.
      isDismissible: false,
      enableDrag: false,
      builder: (ctx) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.viewInsetsOf(ctx).bottom),
        child: const EditNameForm(), // FormController + BlockingHud, as in A4
      ),
    );
```

Checklist:
- [ ] HUD covers app bar, bottom nav (including shell nav), FAB, and sheet content.
- [ ] Back is blocked for: app-bar back, Android back, predictive back, iOS swipe-back, sheet barrier tap, sheet drag, dialog barrier, Escape key.
- [ ] Double/triple tap on Submit and IME Done produce exactly one API call.
- [ ] HUD hides on success, validation-from-server error, network error, timeout, and user cancel (if cancel offered).
- [ ] No programmatic `pop` / route change occurs while busy.
- [ ] Screen reader announces "Saving…" and cannot reach controls behind.

## 4. Submit pipeline

Order is fixed: **end editing → validate all → show HUD → API → hide HUD → success or error.**
- Validation failure never shows the HUD.
- Success: `TextInput.finishAutofillContext(shouldSave: true)` for credential forms, mark form clean (so the discard prompt doesn't fire), then navigate / pop with result / show SnackBar. Navigate **after** busy = false so `PopScope` permits it (or pop with the route's own result after the HUD hides).
- Error: map `ApiFieldError(field, message)` to inline field errors **and** show them in the R3 adaptive alert after the HUD hides; on dismiss focus + scroll to the first invalid field. Network/non-field errors: adaptive alert or banner with retry.
- Log `form_submit`, `form_submit_failed{reason}`, never field values or PII (**flutter-crash-analytics**).
- Offline: either block with a clear message or enqueue and treat as success with "will sync" copy (**flutter-network-offline-sync**). Decide per form.

## 5. Validation

- [ ] `Form(key: GlobalKey<FormState>())` wraps the fields; `TextFormField.validator` is pure and synchronous.
- [ ] Mode: `AutovalidateMode.disabled` until first submit, then `onUserInteraction` (set on the Form after a failed submit). Per-field on blur via `Focus(onFocusChange:)` + `fieldKey.currentState!.validate()`. Not on every keystroke before the user has finished the field.
- [ ] Async checks (username taken) run on blur/debounced, results stored in state and returned by the validator; never `await` inside a validator.
- [ ] Server errors: use `FormField.forceErrorText` (Flutter 3.24+; verify) or `InputDecoration.errorText` fed from state; clear the server error when that field changes.
- [ ] First error: find first invalid field in visual order, `focusNode.requestFocus()` and `Scrollable.ensureVisible(fieldKey.currentContext!)`.
- [ ] **R3**: failed submit (client or server) → `showAdaptiveDialog` + `AlertDialog.adaptive` (title "Check your details", message listing errors, single OK via `adaptiveAction`); shown after the HUD hides, never stacked on it; on dismiss focus the first invalid field. Inline errors are also shown.
- [ ] **R3**: blur/typing validation is inline only (`errorText`, error border, shake via `ShakeOnError` honouring `MediaQuery.disableAnimationsOf`, or `Tooltip` anchored to field) — never a dialog.
- [ ] Error messages are specific and localised; don't show "Required" before the user touched the field.
- [ ] Trim/normalise in `onSaved` / payload builder, not by rewriting controller text while typing.

R3 inline shake (field-level only, honours reduce motion):

```dart
/// Inline field-level feedback: shakes child when `error` becomes non-null.
class ShakeOnError extends StatefulWidget {
  const ShakeOnError({super.key, required this.error, required this.child});
  final String? error;
  final Widget child;
  @override
  State<ShakeOnError> createState() => _ShakeOnErrorState();
}

class _ShakeOnErrorState extends State<ShakeOnError>
    with SingleTickerProviderStateMixin {
  late final _c = AnimationController(
      vsync: this, duration: const Duration(milliseconds: 300));

  @override
  void didUpdateWidget(ShakeOnError old) {
    super.didUpdateWidget(old);
    if (widget.error != null &&
        widget.error != old.error &&
        !MediaQuery.disableAnimationsOf(context)) {
      _c.forward(from: 0);
    }
  }

  @override
  void dispose() {
    _c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => AnimatedBuilder(
        animation: _c,
        builder: (_, child) => Transform.translate(
          offset: Offset(
              math.sin(_c.value * math.pi * 6) * 6 * (1 - _c.value), 0),
          child: child,
        ),
        child: widget.child,
      );
}
```

## 6. Keyboard

- [ ] `Scaffold(resizeToAvoidBottomInset: true)` (default) + form in a `SingleChildScrollView`/`ListView`; for sheets use `isScrollControlled: true` and pad by `MediaQuery.viewInsetsOf(context).bottom`.
- [ ] Field chain: `textInputAction: TextInputAction.next` + `onFieldSubmitted: (_) => nextNode.requestFocus()` (or `FocusScope.of(context).nextFocus()`); last field `TextInputAction.done` → submit wrapper (R1).
- [ ] Multiline fields use `TextInputAction.newline` and are not part of the Next chain's Return behaviour.
- [ ] Tap outside dismisses: `TextField.onTapOutside: (_) => FocusManager.instance.primaryFocus?.unfocus()` or a `GestureDetector`/`TapRegion` around the body. Make sure it doesn't swallow button taps.
- [ ] `ScrollViewKeyboardDismissBehavior.onDrag` on long forms.
- [ ] Focused field scrolls into view above the keyboard (EditableText does this via `scrollPadding`; increase `scrollPadding` if a sticky bottom button covers it).
- [ ] Sticky submit button sits above the keyboard (in `bottomNavigationBar`/`persistentFooterButtons` or padded by viewInsets).
- [ ] Wrap fields in `FocusTraversalGroup` with sensible order for hardware keyboards / tab key.

## 7. Field types

| Field | keyboardType | autofillHints | Other |
|---|---|---|---|
| Email | `emailAddress` | `AutofillHints.email` / `username` | `autocorrect: false`, `enableSuggestions: false` |
| Password (login) | `visiblePassword` off; default text | `AutofillHints.password` | `obscureText: true`, show/hide toggle |
| New password | default | `AutofillHints.newPassword` | strength hint; confirm field also `newPassword` |
| OTP | `number` | `AutofillHints.oneTimeCode` | `maxLength`, `FilteringTextInputFormatter.digitsOnly`; Android SMS Retriever via package |
| Phone | `phone` | `telephoneNumber` | mask formatter; store E.164 |
| Name | `name` | `givenName`/`familyName`/`name` | `textCapitalization.words` |
| Address | `streetAddress` | `streetAddressLine1`, `postalCode`, `addressCity`… | country picker first |
| Card | `number` | `creditCardNumber`, `creditCardExpirationDate`, `creditCardSecurityCode` | prefer PSP SDK sheet; mask `#### ####` |
| Date | none (picker) | `birthday` | `showDatePicker`, read-only field opening picker |

- [ ] Login / sign-up fields wrapped in `AutofillGroup`; on success call `TextInput.finishAutofillContext()` (default `shouldSave: true`); on cancel/leave `finishAutofillContext(shouldSave: false)`.
- [ ] Masks via `TextInputFormatter` that keep the caret position; store raw value, display formatted.
- [ ] Pickers (date, country, dropdown) are `FormField<T>`s so they validate and save with the form, and opening them also unfocuses the text field (R1).
- [ ] `obscureText` fields disable `enableSuggestions` / `autocorrect`.

## 8. Unsaved changes

- [ ] Dirty = current payload != initial payload (compare values, not "was typed in").
- [ ] `PopScope(canPop: !busy && !dirty, onPopInvokedWithResult: (didPop, _) { if (!didPop && !busy) confirmDiscard(); })` — **never** prompt while busy (HUD blocks back silently).
- [ ] Confirm dialog: "Discard changes?" Discard → mark clean then `Navigator.pop`; Keep editing → nothing.
- [ ] After successful save mark clean before navigating, otherwise the prompt fires on your own pop.
- [ ] Sheets: same via PopScope inside the sheet; `isDismissible`/`enableDrag` true only if PopScope handles dirty.
- [ ] Optional drafts: persist debounced payload (no passwords/card data) and offer restore; clear on success.

## 9. Multi-step

- [ ] Each Next goes through `runAction` → R1 (unfocus, validate this step's `GlobalKey<FormState>`), and if it calls the server (e.g. check email), R2 HUD too.
- [ ] One `FormState` key per step; aggregate state in a controller above the steps (survives `PageView`/route changes).
- [ ] Back within the flow goes to the previous step (PopScope on the flow route) and keeps entered values; system back on step 1 → discard prompt if dirty.
- [ ] Progress indicator ("Step 2 of 4") with semantics label.
- [ ] Final submit re-validates all steps; on server error jump to the step owning the field.
- [ ] Deep links into the middle of a flow (**flutter-deep-links**) redirect to the first incomplete step.

## 10. Accessibility

- [ ] Every field has a visible `labelText` (not only `hintText`); `Semantics` label on icon-only buttons.
- [ ] Errors are in `InputDecoration.errorText` (read by TalkBack/VoiceOver); after a failed submit announce "N errors" and focus the first.
- [ ] Focus order matches visual order (`FocusTraversalGroup`, `FocusTraversalOrder` if needed).
- [ ] Layout survives `MediaQuery.textScalerOf` at 2.0 (no fixed-height rows, labels wrap) and RTL.
- [ ] Touch targets ≥ 48 dp; password toggle has a label ("Show password").
- [ ] HUD: announce, BlockSemantics, focus returns afterwards (Section 3).

## 11. Testing plan

Widget tests (`flutter_test`) for the wrapper and HUD; integration tests (`integration_test`/Patrol) for real keyboard, autofill and back gestures; manual on device for predictive back and iOS swipe.

Scenario matrix:

| # | Scenario | Expected |
|---|---|---|
| 1 | Type in last field, tap app-bar Save without leaving field | Latest value saved; keyboard hidden |
| 2 | Press keyboard Done on last field | Same pipeline as Submit; ends editing first |
| 3 | IME composing text (Japanese/Korean) then Submit | Committed text saved, not partial |
| 4 | Submit with invalid fields | No HUD; adaptive alert (Cupertino on iOS, Material on Android) lists errors; inline errors shown; dismiss → first error focused & visible |
| 4b | Leave an invalid field (blur) | Inline error + shake (no shake with reduce motion); no dialog |
| 5 | Double/triple tap Submit, tap Submit then IME Done | Exactly one request |
| 6 | Android back / predictive back during HUD | Nothing happens, no peek animation |
| 7 | iOS swipe-back during HUD | Gesture doesn't start |
| 8 | App-bar back / bottom nav tab / other button during HUD | Blocked |
| 9 | Sheet form: drag down / tap barrier during HUD | Sheet stays |
| 10 | API returns error / times out / user cancels | HUD hides; error shown; form editable |
| 11 | Very fast response | HUD visible ≥ min time, no flash |
| 12 | Server field errors | HUD hides, then alert lists them; also inline on right fields; cleared on edit |
| 13 | Dirty form, back (not submitting) | "Discard changes?" shown |
| 14 | Dirty form, back during submit | No prompt, nothing happens |
| 15 | Successful save then back | No discard prompt |
| 16 | Login success | Password manager offers to save |
| 17 | TalkBack/VoiceOver during submit | "Saving…" announced; content unreachable |
| 18 | textScaler 2.0, landscape, small phone + keyboard | Focused field and Submit visible |

Widget test tips: `tester.showKeyboard(finder)`, `tester.testTextInput.receiveAction(TextInputAction.done)`, `tester.binding.handlePopRoute()` to simulate system back, use a `Completer` in a fake repository to hold the request open while asserting HUD and back blocking, then complete with error and assert HUD gone.

## 12. Common bugs

- **R1 violation**: app-bar Save reads `controller.text`/state before unfocus → stale or uncommitted value (autocorrect, IME composition) saved.
- **R1 violation**: IME Done path calls API directly, skipping validation and the HUD.
- **R2 violation**: spinner inside the button only; user taps back, tab bar, or a second button mid-request.
- **R2 violation**: HUD in route Stack while bottom nav lives in shell → tabs still tappable.
- **R3 violation**: failed submit only turns fields red (user misses off-screen error), or a dialog pops on every blur; Material `AlertDialog` on iOS instead of `.adaptive`; alert shown while HUD still up.
- Stuck HUD: `busy = false` not in `finally`; exception or early `return` leaves it up; no request timeout.
- Using `WillPopScope` (deprecated) or `PopScope` set via async decision → predictive back ignores it.
- Programmatic `Navigator.pop` during submit bypasses `PopScope`.
- Discard prompt shown during submit, or after successful save (clean flag not reset).
- Sheet with `isDismissible: true` and no PopScope → dragged closed mid-submit, result lost.
- `FocusScope.of(context)` from a sheet context doesn't unfocus the field in the parent → keyboard stays.
- Validator does async work / setState → exceptions, flicker.
- `AutovalidateMode.always` → errors before typing.
- No `AutofillGroup` / no `finishAutofillContext` → no password save prompt.
- `setState` after `dispose` when user left before response (check `mounted`).
- Mask formatter jumps caret to end.

## 13. Output format

**Review**: list findings grouped by R1, R2, R3, validation, keyboard, fields/autofill, unsaved changes, a11y; each with file:line, severity (R1/R2/R3 violations are always High), why, and a concrete fix referencing the appendix pieces. End with the scenario matrix rows that fail.

**Implement**: deliver `FormController` + `HudScope`/`BlockingHud` if missing (shared, not per screen), wire the screen through them, add widget tests for scenarios 1, 2, 4, 5, 6, 10, 13, 14. State assumptions (state management, presentation, offline policy) and anything left to verify on device (predictive back, iOS swipe, autofill prompt).

## Appendix — reference code (Dart)

### A1. Form controller: submit wrapper (R1) + busy state (R2)

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'dart:math' as math;
import 'package:flutter/cupertino.dart';
import 'package:flutter/services.dart';

class ApiFieldError implements Exception { // field id -> message
  ApiFieldError(this.fieldErrors, [this.message]);
  final Map<String, String> fieldErrors;
  final String? message;
}

typedef FormAction<T> = Future<T> Function();

class FormController extends ChangeNotifier {
  FormController({this.minHudDuration = const Duration(milliseconds: 400)});

  final GlobalKey<FormState> formKey = GlobalKey<FormState>();
  final Duration minHudDuration;
  final Map<String, String> _serverErrors = {};
  final Map<String, FocusNode> _nodes = {};
  final Map<String, GlobalKey> _fieldKeys = {};
  final List<String> _order = [];

  bool _busy = false;
  bool get busy => _busy;
  String? bannerError;
  AutovalidateMode autovalidateMode = AutovalidateMode.disabled;
  bool _disposed = false;

  /// Register a field in visual order (for "focus first error").
  FocusNode nodeFor(String id) {
    if (!_order.contains(id)) _order.add(id);
    _fieldKeys.putIfAbsent(id, GlobalKey.new);
    return _nodes.putIfAbsent(id, FocusNode.new);
  }

  GlobalKey keyFor(String id) => _fieldKeys.putIfAbsent(id, GlobalKey.new);
  String? serverError(String id) => _serverErrors[id];

  void clearServerError(String id) {
    if (_serverErrors.remove(id) != null) _notify();
  }

  /// R1: end editing. Commits IME composition, runs focus-loss
  /// listeners, hides the keyboard. Await so listeners settle.
  static Future<void> endEditing() async {
    FocusManager.instance.primaryFocus?.unfocus();
    await WidgetsBinding.instance.endOfFrame;
  }

  /// The ONLY way a button / IME action may act on the form.
  /// Returns null if blocked (busy / invalid) or on error.
  Future<T?> submit<T>(
    FormAction<T> action, {
    bool validate = true,
    void Function(T result)? onSuccess,
    void Function(Object error)? onError,
  }) async {
    if (_busy) return null; // re-entrancy guard (sync, before any await)
    _busy = true; // block touches/back immediately
    _notify();
    final started = DateTime.now();
    try {
      await endEditing(); // R1
      final form = formKey.currentState;
      if (validate && form != null) {
        _serverErrors.clear();
        bannerError = null;
        if (!form.validate()) {
          autovalidateMode = AutovalidateMode.onUserInteraction;
          _busy = false; // validation failure: no HUD time
          _notify();
          await _alertErrors(); // R3
          return null;
        }
        form.save(); // onSaved -> payload
      }
      final result = await action();
      await _holdMinimum(started);
      _busy = false;
      _notify();
      onSuccess?.call(result);
      return result;
    } on ApiFieldError catch (e) {
      await _holdMinimum(started);
      _serverErrors
        ..clear()
        ..addAll(e.fieldErrors);
      bannerError = e.message;
      _busy = false; // HUD hides BEFORE the alert
      _notify();
      await _alertErrors(); // R3: server errors in alert + inline
      onError?.call(e);
      return null;
    } catch (e) {
      await _holdMinimum(started);
      bannerError = 'Something went wrong. Please try again.';
      onError?.call(e);
      return null;
    } finally {
      // R2: ALWAYS dismiss, whatever happened above.
      if (_busy) {
        _busy = false;
        _notify();
      }
    }
  }

  /// Set by the screen (needs a context); see A3b showValidationAlert.
  Future<void> Function(List<String> messages)? alertPresenter;

  Future<void> _alertErrors() async {
    final msgs = <String>[];
    for (final id in _order) {
      final st = _fieldKeys[id]?.currentState;
      final m = _serverErrors[id] ??
          (st is FormFieldState ? st.errorText : null);
      if (m != null) msgs.add(m);
    }
    if (msgs.isNotEmpty) await alertPresenter?.call(msgs);
    _focusFirstError(); // after dismiss
  }

  Future<void> _holdMinimum(DateTime started) async {
    final left = minHudDuration - DateTime.now().difference(started);
    if (left > Duration.zero) await Future<void>.delayed(left);
  }

  void _focusFirstError() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      for (final id in _order) {
        final ctx = _fieldKeys[id]?.currentContext;
        final state = _fieldKeys[id]?.currentState;
        final invalid = _serverErrors.containsKey(id) ||
            (state is FormFieldState && state.hasError);
        if (invalid && ctx != null) {
          _nodes[id]?.requestFocus();
          Scrollable.ensureVisible(ctx, alignment: 0.2);
          return;
        }
      }
    });
  }

  void _notify() {
    if (!_disposed) notifyListeners();
  }

  @override
  void dispose() {
    _disposed = true;
    for (final n in _nodes.values) {
      n.dispose();
    }
    super.dispose();
  }
}
```

### A2. Blocking HUD with back blocking (R2)

```dart
import 'package:flutter/semantics.dart';

/// Wrap the WHOLE route (above Scaffold). For shell layouts where the
/// bottom nav is outside the route, place this in MaterialApp.builder
/// and drive `busy` from a global notifier instead.
class BlockingHud extends StatefulWidget {
  const BlockingHud({
    super.key,
    required this.busy,
    required this.child,
    this.message = 'Saving…',
  });

  final bool busy;
  final Widget child;
  final String message;

  @override
  State<BlockingHud> createState() => _BlockingHudState();
}

class _BlockingHudState extends State<BlockingHud> {
  @override
  void didUpdateWidget(BlockingHud old) {
    super.didUpdateWidget(old); // announce; verify API for your version
    if (widget.busy && !old.busy) SemanticsService.announce(widget.message, Directionality.of(context));
  }

  @override
  Widget build(BuildContext context) {
    return PopScope(
      // Blocks Android back, predictive back, app-bar BackButton
      // (maybePop), Escape and iOS swipe-back while busy.
      canPop: !widget.busy,
      child: Stack(
        children: [
          AbsorbPointer(
            absorbing: widget.busy,
            child: ExcludeSemantics(
                excluding: widget.busy, child: widget.child),
          ),
          if (widget.busy)
            Positioned.fill(
              child: BlockSemantics(
                child: Stack(
                  children: [
                    const ModalBarrier(
                        dismissible: false, color: Colors.black26),
                    Center(
                      child: Semantics(
                        liveRegion: true,
                        label: widget.message,
                        child: Card(child: Padding(
                          padding: const EdgeInsets.all(24),
                          child: Column(mainAxisSize: MainAxisSize.min, children: [
                            const CircularProgressIndicator(),
                            const SizedBox(height: 12),
                            Text(widget.message),
                          ]),
                        )),
                      ),
                    ),
                  ],
                ),
              ),
            ),
        ],
      ),
    );
  }
}
```

### A3. Validation helpers

```dart
typedef Validator = String? Function(String? value);

class V {
  static Validator required([String msg = 'Required']) =>
      (v) => (v == null || v.trim().isEmpty) ? msg : null;

  static Validator email([String msg = 'Enter a valid email']) => (v) {
        final s = v?.trim() ?? '';
        return RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$').hasMatch(s) ? null : msg;
      };

  static Validator minLength(int n, [String? msg]) => (v) =>
      (v ?? '').length < n ? (msg ?? 'At least $n characters') : null;

  /// Server error wins until the user edits the field.
  static Validator withServer(FormController c, String id, Validator base) =>
      (v) => base(v) ?? c.serverError(id);

  static Validator all(List<Validator> vs) => (v) {
        for (final f in vs) {
          final e = f(v);
          if (e != null) return e;
        }
        return null;
      };
}
```

### A3b. R3 helper: adaptive submit alert (ShakeOnError is in Section 5)

```dart
Future<void> showValidationAlert(BuildContext context, List<String> msgs) =>
    showAdaptiveDialog<void>(
      context: context,
      builder: (ctx) => AlertDialog.adaptive(
        title: const Text('Check your details'),
        content: Text(msgs.length == 1
            ? msgs.first
            : msgs.map((m) => '• $m').join('\n')),
        actions: [adaptiveAction(ctx, 'OK', () => Navigator.pop(ctx))],
      ),
    );

Widget adaptiveAction(BuildContext ctx, String label, VoidCallback onTap) {
  final p = Theme.of(ctx).platform;
  return (p == TargetPlatform.iOS || p == TargetPlatform.macOS)
      ? CupertinoDialogAction(onPressed: onTap, child: Text(label))
      : TextButton(onPressed: onTap, child: Text(label));
}

```

On blur: `Focus(onFocusChange: (f) { if (!f) fieldKey.currentState?.validate(); })` around the field, inside `ShakeOnError(error: ...)` — inline only, never a dialog.

### A4. Sample screen: sign-up wiring everything

```dart
abstract class SignUpRepository { // impl: flutter-network-offline-sync
  Future<void> signUp({required String email, required String password});
}

class SignUpScreen extends StatefulWidget {
  const SignUpScreen({super.key, required this.repo});
  final SignUpRepository repo;
  @override
  State<SignUpScreen> createState() => _SignUpScreenState();
}

class _SignUpScreenState extends State<SignUpScreen> {
  final c = FormController();
  final _email = TextEditingController();
  final _password = TextEditingController();
  String _savedEmail = '';
  String _savedPassword = '';

  @override
  void initState() {
    super.initState();
    c.alertPresenter = (m) => showValidationAlert(context, m); // R3
  }

  bool get _dirty => _email.text.isNotEmpty || _password.text.isNotEmpty;

  @override
  void dispose() {
    c.dispose();
    _email.dispose();
    _password.dispose();
    super.dispose();
  }

  Future<void> _submit() => c.submit<void>(
        () => widget.repo.signUp(email: _savedEmail, password: _savedPassword),
        onSuccess: (_) {
          TextInput.finishAutofillContext(); // offer password save
          _email.clear(); // mark clean before leaving
          _password.clear();
          if (mounted) Navigator.of(context).pop(true);
        },
      );

  Future<void> _confirmDiscard() async {
    final discard = await showAdaptiveDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog.adaptive(
        title: const Text('Discard changes?'),
        actions: [
          adaptiveAction(ctx, 'Keep editing', () => Navigator.pop(ctx, false)),
          adaptiveAction(ctx, 'Discard', () => Navigator.pop(ctx, true)),
        ],
      ),
    );
    if (discard != true || !mounted) return;
    TextInput.finishAutofillContext(shouldSave: false);
    _email.clear(); // clean -> PopScope allows the pop
    _password.clear();
    Navigator.of(context).pop();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: Listenable.merge([c, _email, _password]),
      builder: (context, _) => BlockingHud(
        busy: c.busy,
        message: 'Creating account…',
        child: PopScope(
          // Dirty prompt only when NOT busy (BlockingHud handles busy).
          canPop: c.busy ? false : !_dirty,
          onPopInvokedWithResult: (didPop, _) {
            if (!didPop && !c.busy) _confirmDiscard();
          },
          child: Scaffold(
            appBar: AppBar(
              title: const Text('Create account'),
              actions: [
                TextButton(
                  onPressed: c.busy ? null : _submit, // via wrapper (R1)
                  child: const Text('Done'),
                ),
              ],
            ),
            body: Form(
              key: c.formKey,
              autovalidateMode: c.autovalidateMode,
              child: AutofillGroup(
                child: ListView(
                  padding: const EdgeInsets.all(16),
                  keyboardDismissBehavior:
                      ScrollViewKeyboardDismissBehavior.onDrag,
                  children: [
                    TextFormField(
                      key: c.keyFor('email'),
                      focusNode: c.nodeFor('email'),
                      controller: _email,
                      decoration: const InputDecoration(labelText: 'Email'),
                      keyboardType: TextInputType.emailAddress,
                      autofillHints: const [AutofillHints.email],
                      autocorrect: false,
                      enableSuggestions: false,
                      textInputAction: TextInputAction.next,
                      onChanged: (_) => c.clearServerError('email'),
                      onFieldSubmitted: (_) =>
                          c.nodeFor('password').requestFocus(),
                      validator: V.withServer(
                          c, 'email', V.all([V.required(), V.email()])),
                      onSaved: (v) => _savedEmail = v!.trim(),
                    ),
                    const SizedBox(height: 12),
                    TextFormField(
                      key: c.keyFor('password'),
                      focusNode: c.nodeFor('password'),
                      controller: _password,
                      decoration:
                          const InputDecoration(labelText: 'Password'),
                      obscureText: true,
                      autocorrect: false,
                      enableSuggestions: false,
                      autofillHints: const [AutofillHints.newPassword],
                      textInputAction: TextInputAction.done,
                      onChanged: (_) => c.clearServerError('password'),
                      onFieldSubmitted: (_) => _submit(), // same wrapper
                      validator: V.withServer(c, 'password',
                          V.all([V.required(), V.minLength(8)])),
                      onSaved: (v) => _savedPassword = v!,
                    ),
                    const SizedBox(height: 24),
                    FilledButton(
                      onPressed: c.busy ? null : _submit,
                      child: const Text('Create account'),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```
