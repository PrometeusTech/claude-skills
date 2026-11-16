# Expo specifics for push and links

Confirm names against the project's SDK version; notification APIs change between SDKs.

## Permissions and tokens (`expo-notifications`)
- Android: create channels with `setNotificationChannelAsync` **before** requesting the permission
  or the token (on Android 13+ the prompt needs a channel).
- `getPermissionsAsync` / `requestPermissionsAsync`; iOS options include provisional authorization.
- Token: `getExpoPushTokenAsync({ projectId })` when sending through Expo's push service, or
  `getDevicePushTokenAsync` for FCM / APNs directly. `addPushTokenListener` reports token changes —
  upsert on the server when it fires.
- Push needs a development build (not Expo Go) on Android, and push credentials per build profile.
- Server side with Expo's push service: send in chunks, then fetch receipts; a
  `DeviceNotRegistered` error means delete that token.

## Foreground and taps
- `setNotificationHandler` decides how a notification is shown while the app is in the foreground.
- `addNotificationReceivedListener` — received while open (refresh data here).
- `addNotificationResponseReceivedListener` — the user tapped. Register it early (module top level)
  so taps that launch the app are not missed on iOS.
- Cold start: also read `useLastNotificationResponse()` (or `getLastNotificationResponseAsync()`)
  at startup; it changes when a response arrives. Check `actionIdentifier ===
  Notifications.DEFAULT_ACTION_IDENTIFIER` for a plain tap, and call
  `clearLastNotificationResponseAsync()` after handling it so it is not handled again.
- Android may launch a terminated app from a tap through a registered background task rather than
  the listener — test the killed-app case on Android and iOS separately.
- Badges: `setBadgeCountAsync`.

## Links
- `scheme` in app config for the custom scheme; `ios.associatedDomains`
  (`applinks:example.com`) and `android.intentFilters` with `autoVerify: true` for universal / App
  Links.
- Expo Router: every route file is linkable; `+native-intent` (`redirectSystemPath`) rewrites
  incoming system paths (e.g. map a notification URL or an old web path) before routing.
- Route notification taps through the same path: put a `url` in the notification data and hand it
  to the router after the session is ready.
- Test: `npx uri-scheme open <url> --ios|--android`, `xcrun simctl openurl booted <url>`,
  `adb shell am start -a android.intent.action.VIEW -d <url>`;
  `adb shell pm get-app-links <package>` shows App Links verification state.
- The Android certificate fingerprint for `assetlinks.json`: from the Play Console (App signing)
  for store builds, from `eas credentials` for EAS-signed builds.
