# React Native / Expo specifics for UI quality

## Accessibility props
- `accessibilityRole` (or `role`), `accessibilityLabel`, `accessibilityHint`, `accessibilityState`
  (`selected`, `checked`, `disabled`, `expanded`, `busy`); `accessible` on a container to group a
  card into one element; `importantForAccessibility="no-hide-descendants"` /
  `accessibilityElementsHidden` for decorative content.
- Modals: `accessibilityViewIsModal` (iOS) on the modal container; return focus with
  `AccessibilityInfo.setAccessibilityFocus` on the trigger.
- Announcements: `AccessibilityInfo.announceForAccessibility`.
- Custom actions: `accessibilityActions` + `onAccessibilityAction` for swipe and long-press
  behaviors.
- Tests: React Native Testing Library queries `getByRole('button', { name: ... })` and
  `getByLabelText` — if a test cannot find the control by role and name, a screen reader cannot
  either.

## Text scaling
- Text scales by default (`allowFontScaling`). Cap individual elements with
  `maxFontSizeMultiplier`; never disable scaling globally.
- `PixelRatio.getFontScale()` / `useWindowDimensions().fontScale` to adapt layouts at large
  scales (stack horizontally arranged items vertically).

## Layout
- `react-native-safe-area-context` (`SafeAreaView` / `useSafeAreaInsets`) — the core `SafeAreaView`
  is iOS-only.
- Keyboard: `KeyboardAvoidingView` (behavior differs per platform) or
  `react-native-keyboard-controller` for reliable handling; `keyboardShouldPersistTaps="handled"`
  on scroll views with buttons; `returnKeyType` + refs to move focus.
- Input hints: `keyboardType`, `autoComplete`, `textContentType` (iOS), `secureTextEntry`.
- Back: Expo Router / React Navigation handle Android back; for dirty forms use the navigator's
  before-remove event (e.g. `usePreventRemove` in React Navigation) rather than a raw
  `BackHandler`, so the iOS swipe and header back are covered too.
- `hitSlop` / `pressRetentionOffset` to enlarge small touch targets.

## Lists and performance
- `FlatList` with `keyExtractor` on a stable id, memoized `renderItem` components, and
  `getItemLayout` when rows have a fixed height; or `@shopify/flash-list` for long, complex lists.
- No anonymous objects / functions recreated per row when rows are memoized.
- Images: `expo-image` with caching and a `recyclingKey` in recycled lists; request thumbnails from
  the server / CDN at the displayed size.
- Animations: Reanimated worklets run on the UI thread; avoid JS-driven animations during
  interactions.
- Profile with React DevTools (re-renders) and the performance monitor in a **release** build
  (`npx expo run:android --variant release` or a preview build). Hermes is the default engine.

## Theme and locale
- `useColorScheme()` and `userInterfaceStyle` in app config (`automatic` only if every screen
  supports both themes). Status bar style follows the theme (`expo-status-bar`).
- `expo-localization` (`getLocales()`, `getCalendars()`) for language, region and time zone; format
  with `Intl` (Hermes supports it) or the i18n library.

## Permissions
- Request through the module (`requestPermissionsAsync` / the module's hook); handle `canAskAgain:
  false` by linking to `Linking.openSettings()`.

## End-to-end
- Maestro flows (YAML, run against a simulator / emulator / device) or Detox for the main path,
  when the project has them. They need a device or simulator — usually on a developer machine or a
  CI runner with macOS / an emulator, not in a cloud container.
