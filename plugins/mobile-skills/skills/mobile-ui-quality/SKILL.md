---
name: mobile-ui-quality
description: Rules and review checklist for the user-facing quality of a mobile app screen (React Native / Expo, mostly platform-neutral) — screen readers (VoiceOver, TalkBack) with accessible names, roles and states, large text from the system settings, touch target size, safe areas, notches and edge-to-edge, the keyboard covering inputs, the Android back button and the iOS back gesture, small phones, tablets and orientation, dark mode, translations and locale formatting, loading / empty / error / offline / no-permission states, OS permission prompts and refusals, and performance (long lists, re-renders, images, startup time, app size). Use it whenever you add or change a screen, a form, a list, a modal or bottom sheet, navigation, a translation, an image or a permission request in a mobile app, and whenever you review such changes — even when the task only says "add a button" or "show the bookings". Web pages are covered by frontend-quality.
---

# Mobile UI quality

The mobile counterpart of frontend-quality. Most mobile UI bugs that reach users are not crashes:
a button VoiceOver reads as "button", a label cut off at the largest text size, a submit hidden
under the keyboard, an Android back press that throws away a half-filled form, a list that
stutters on a mid-range phone. The rules below are part of "done"; the checklist is what a
reviewer does on a simulator and a real device. React Native specifics are in
`references/react-native.md`.

## 1. Screen readers

- Every interactive element has a role, an accessible name and its state (selected, checked,
  disabled, expanded, busy). Icon-only buttons have a name.
- Related content is grouped so a card reads as one element; decorative images are hidden from the
  screen reader, informative ones are described.
- Reading and focus order follow the visual order. Modals and bottom sheets keep focus inside and
  return it to the trigger when closed.
- Results of async actions are announced (saved, failed). Custom gestures (swipe to delete, long
  press) have an accessible action as an alternative.

## 2. Text size

- The UI works up to the largest accessibility text size: text wraps, containers grow, nothing
  critical is cut off or overlaps. No fixed heights on text containers.
- Scaling is capped only for the few elements that cannot grow (a tab bar label), never turned off
  globally.

## 3. Touch

- Targets at least 44×44 pt (iOS) / 48×48 dp (Android), with spacing; small icons get a larger hit
  area.
- Every press gives feedback; submit is disabled in flight; destructive actions ask for
  confirmation.

## 4. Layout, devices and navigation

- Safe-area insets respected (notch, dynamic island, home indicator, Android navigation bar;
  edge-to-edge is the default on recent Android).
- Works on the smallest supported phone and on tablets; orientation is either locked on purpose or
  supported.
- The keyboard never covers the focused input or the submit button; the return key moves to the
  next field; keyboard types and autofill hints are right (e-mail, password, one-time code,
  phone).
- Android back and the iOS back gesture close the top sheet or modal first, then go back; leaving a
  dirty form asks before discarding it.

## 5. Language and content

- All strings translated in every locale (frontend-quality §2); the device language is used unless
  the user picked another; plurals through the i18n library.
- Dates, times, numbers and currency in the device locale and the right time zone.
- Long translations, long names, emoji and RTL text fit (text-input-hardening §4).

## 6. States

- Every data screen has loading (skeleton or spinner), empty, error with retry, offline
  (mobile-offline-lifecycle) and "no permission" states; lists have pull-to-refresh where users
  expect it.
- An OS permission (camera, photos, location) is requested when the feature is used, with context;
  a denied permission shows what to do and links to Settings — never a dead button.

## 7. Theme

Dark mode is either fully supported (every screen, status bar, images, charts, third-party
components) or the app is locked to light. Contrast meets WCAG AA in each supported theme.

## 8. Performance

- Long lists are virtualized, with stable keys and memoized rows; no heavy work during render.
- No re-render storms: a keystroke or a scroll does not re-render the whole screen (profile it).
- Images are requested at the size they are shown and cached — never full-size photos in a list.
- Animations run on the UI thread; heavy JS work does not block touches.
- Startup: nothing non-essential blocks the first screen; new dependencies are justified by size
  (app download size and JS bundle).
- Measure in a release build on a mid-range Android phone; debug builds lie.

## Implementation checklist

- [ ] Roles, names and states on every control; grouping; focus in modals; announcements; gesture alternatives.
- [ ] Layout survives the largest text size; scaling capped only where unavoidable.
- [ ] Touch targets ≥ 44 pt / 48 dp; feedback; submit disabled in flight; destructive confirmation.
- [ ] Safe areas; small phone and tablet; keyboard handling and input hints; back behavior with dirty-form guard.
- [ ] Strings translated; locale formatting; long / RTL content fits.
- [ ] Loading, empty, error, offline, no-permission and permission-denied states.
- [ ] Dark mode complete or locked; contrast checked.
- [ ] Virtualized lists, memoized rows, sized images; release-build check on a mid-range device.
- [ ] Tests query by role and accessible name; one end-to-end flow for the main path where the project has one.

## Review / judge checklist (try it on devices)

1. Go through the new screens with VoiceOver and TalkBack: every control announced with name, role
   and state? Order right? Can you open and close each modal and sheet?
2. Set the largest accessibility text size (and display zoom): anything cut off, overlapping or
   unreachable?
3. Smallest supported phone, a tablet, landscape if supported, a notched device, Android with
   three-button and gesture navigation.
4. Focus every input: visible above the keyboard? Next / Done right? Submit reachable?
5. Press Android back (and swipe back on iOS) at each step, with a modal open and with a dirty
   form.
6. Switch language and region; switch to dark mode.
7. Airplane mode, a failing API, an empty account, a role without permission; deny each OS
   permission the screen uses.
8. Scroll a list of 1,000 items in a release build on a mid-range Android device; profile
   re-renders while typing and scrolling; check image sizes in the network log; compare the app
   and bundle size with base.
