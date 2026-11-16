# Expo / React Native security specifics

Confirm option names against the project's SDK version before relying on them.

## Secure storage
- `expo-secure-store` (`setItemAsync` / `getItemAsync` / `deleteItemAsync`): Keychain on iOS,
  Keystore-encrypted storage on Android. Store small values (tokens), not documents.
- Options: `keychainAccessible` (e.g. `WHEN_UNLOCKED_THIS_DEVICE_ONLY` or
  `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` when background requests need the token);
  `requireAuthentication` for a biometric gate on read (and handle the key being invalidated when
  biometrics change).
- Keychain items can survive an app uninstall on iOS: on first launch after install (a flag in
  AsyncStorage missing), clear the secure store.
- Android backups: `android.allowBackup` in app config — disable it, or exclude sensitive files, so
  app data is not restored to another device.

## Session handling
- One HTTP client (fetch wrapper or axios instance) owns the token: attaches it, handles 401 with a
  single shared refresh promise, retries the waiting requests once, and calls a single `logout()`
  that clears secure store, `queryClient`, persisted cache, offline queue, local DB user data,
  `Sentry.setUser(null)`, analytics reset, WebView cookies, and unregisters the push token.

## Bundle and config
- `EXPO_PUBLIC_*` variables are inlined into the JS bundle at build / update time. Anything else in
  `app.config.ts` that ends up in `extra` is readable through `expo-constants`. Inspect with
  `npx expo config --type public`.
- Production builds: no `__DEV__`-only bypasses reachable, no dev client in store builds.

## Links
- Routing: Expo Router maps URLs to files — every file under `app/` is reachable by a link, so
  protected groups need a layout-level guard, and a `+native-intent` handler can rewrite or reject
  incoming system paths before routing. With React Navigation, `linking.config` lists the
  reachable screens; keep it an allowlist.
- Test links: `npx uri-scheme open "myapp://path?x=1" --ios|--android`.
- Open external URLs with `expo-web-browser` (`openBrowserAsync`) or `Linking.openURL` after
  checking the scheme.
- OAuth: `expo-auth-session` / `WebBrowser.openAuthSessionAsync` (system browser session), with
  PKCE.

## WebView (`react-native-webview`)
- `originWhitelist` limited to your origins; `onShouldStartLoadWithRequest` sends other URLs to the
  system browser.
- `onMessage`: check `event.nativeEvent.url` origin and parse / validate the payload.
- Keep `allowFileAccess`, `allowUniversalAccessFromFileURLs` and `mixedContentMode="always"` off.

## Transport
- iOS ATS exceptions (`NSAppTransportSecurity` in `ios.infoPlist`) and Android cleartext
  (`usesCleartextTraffic` via config plugin / build properties) only in development builds.

## Side channels
- Strip `console.*` in production (babel `transform-remove-console`, or a logger that is silent in
  production).
- `@sentry/react-native`: `beforeSend` and `beforeBreadcrumb` drop `Authorization` headers, request
  bodies and personal data; do not attach screenshots or view hierarchy on sensitive screens.
- `expo-screen-capture` to block or detect capture on sensitive screens (behavior differs between
  iOS and Android); an overlay on `AppState` `inactive` hides the app-switcher snapshot on iOS.
