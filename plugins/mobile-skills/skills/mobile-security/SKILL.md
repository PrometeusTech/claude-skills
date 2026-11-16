---
name: mobile-security
description: Rules and review checklist for mobile app security (React Native / Expo, mostly platform-neutral) — the client cannot be trusted, where tokens and personal data are stored (Keychain / Keystore vs plain storage, backups), the session (refresh, concurrent refresh, logout that really clears everything, account or tenant switch, biometrics), deep links as untrusted input, WebViews and their JS bridges, anything shipped in the bundle being public (API keys, inlined env vars, hidden features), transport security, personal data in device logs, crash reports and analytics, sensitive screens in the app switcher. Use it whenever you add or change authentication or session handling, local storage, a deep link or route with parameters, a WebView, an API key or SDK config, logging or crash reporting, or a screen with sensitive data, and whenever you review such changes — even if the task only says "remember the user" or "add the maps key".
---

# Mobile security

A mobile app is code running on a device its owner fully controls: they can read its storage,
decompile its bundle, intercept its traffic and call the API with its token. Other apps on the
same phone can open its links. The rules below assume all of that; the checklist is what a
reviewer does with a debug build, a proxy and a terminal.

Server-side rules still apply in full — authz-multitenancy, web-security. Expo specifics are in
`references/expo.md`.

## 1. The client is not trusted

- Every rule — role, tenant, price, limit, paid feature — is enforced by the API. Hiding a button
  or a screen in the app is presentation, not authorization (authz-multitenancy §7).
- Certificate pinning, jailbreak / root detection and obfuscation raise the cost of attacks; they
  never replace a server-side check.

## 2. Secrets at rest

- Access and refresh tokens live in the secure store (Keychain on iOS, Keystore-backed storage on
  Android), never in AsyncStorage, unencrypted MMKV, a persisted Redux / Zustand store or a
  persisted query cache.
- Choose the accessibility class on purpose (e.g. only when unlocked, this device only) so tokens
  are not restored to another device from a backup.
- Personal data kept for offline use is minimized (mobile-offline-lifecycle §4). Files downloaded
  for viewing stay in the app sandbox and are deleted when no longer needed.
- Secrets and one-time codes are not copied to the clipboard by the app.

## 3. The session

- Short-lived access token, refresh token rotated on use (with reuse detection on the server).
- When several requests get a 401 at once, **one** refresh runs and the others wait for it and
  retry; a failed refresh logs the user out cleanly, with a message, instead of looping.
- Logout clears everything tied to the user: secure store, query and persisted caches, offline
  queue, user rows in the local DB, WebView cookies, the crash reporter's and analytics' user
  identity — and unregisters the push token on the server (mobile-push-deeplinks §2). The server
  revokes the refresh token. Switching account or tenant clears the same tenant-scoped state.
- "Log out of all devices" and a password change revoke the other sessions' refresh tokens.
- Biometrics unlock a token stored on the device (a local gate); they do not authenticate the user
  to the server and have a fallback.

## 4. Everything in the bundle is public

- Inlined environment variables (`EXPO_PUBLIC_*` and similar), app config and the JS bundle
  (bytecode included) can be read by anyone with the app. They hold only keys designed to be
  public (a maps key restricted by bundle id / package, a crash-reporting DSN, an analytics write
  key), each restricted on the provider's side.
- Real secrets stay on the backend: the app calls your API, your API calls the provider.
- No admin or debug capability shipped "hidden" in production builds; developer menus, verbose
  logging and test endpoints are off in production.

## 5. Deep links are untrusted input

Any app or web page can open the app's custom scheme with any path and params, and another app can
register the same custom scheme.

- Routes and params from a link are validated like request params; only allowlisted routes can be
  opened from outside.
- A protected screen opened while logged out goes through sign-in and then continues to the
  intended (allowlisted) route. A link pointing to another tenant does not switch tenant silently.
- A link never performs an action by itself — delete, pay, accept an invitation, change e-mail,
  link an account. It opens a screen where the signed-in user confirms.
- Tokens in links (magic sign-in, invitation, password reset, e-mail verification) are single-use
  and short-lived, and travel only over verified universal / app links, never a custom scheme.
- URLs from data or links that the app opens externally are checked: `https` (or `http`) only —
  never `javascript:`, `file:`, `intent:` or arbitrary schemes.

## 6. WebViews

- Load only your own origins (or an explicit allowlist); navigation anywhere else goes to the
  system browser.
- The JS bridge (`postMessage` / injected JavaScript) is an API exposed to whatever page is
  loaded: validate the origin and the shape of every message, expose the minimum, and never inject
  tokens into pages you do not control.
- File access and mixed content are off.
- OAuth and third-party sign-in use the system browser session (ASWebAuthenticationSession /
  Custom Tabs), not a WebView — providers block WebView sign-in and it exposes credentials to the
  app.

## 7. Transport

- HTTPS only; no cleartext exceptions in production config (ATS on iOS, cleartext traffic on
  Android); no TLS validation disabled "temporarily" in any build that ships.
- If you pin certificates, pin to keys with backups and a rotation plan, or an expired certificate
  locks every installed build out of the API.

## 8. Side channels

- `console.log` in a production build goes to the device log: strip logging or keep it free of
  tokens and personal data.
- Crash reports and breadcrumbs scrub tokens, auth headers, request bodies and personal data
  (`beforeSend` / `beforeBreadcrumb`); analytics events carry no e-mails, names or free text
  (web-security §5).
- Screens with sensitive data (documents, IDs, payment, health) are hidden in the app-switcher
  snapshot and, where warranted, protected from screen capture — platforms differ in what can be
  blocked; know what yours actually does.
- Sensitive inputs use secure text entry and the right autofill hints; lock-screen notification
  content follows mobile-push-deeplinks §3.

## Implementation checklist

- [ ] Server enforces every rule the UI reflects.
- [ ] Tokens in the secure store with a chosen accessibility class; nothing sensitive in plain storage or persisted state.
- [ ] Single-flight refresh; clean logout on refresh failure; server-side revocation.
- [ ] Logout / account / tenant switch clears all local state and unregisters push.
- [ ] Only public-by-design keys in the bundle, restricted at the provider; no debug features in production.
- [ ] Deep links: allowlisted routes, validated params, auth then continue, no action without confirmation, tokens only over verified links, external URL schemes checked.
- [ ] WebView: origin allowlist, validated bridge, no token injection; OAuth in the system browser.
- [ ] HTTPS only in production; pinning (if any) with backup keys.
- [ ] Logs, crash reports and analytics scrubbed; sensitive screens hidden in the app switcher.

## Review / judge checklist (try to break it)

1. Sign in on a debug build and read the app's storage (simulator app container,
   `adb shell run-as <package>`): any token or personal data in plain files, AsyncStorage, MMKV or
   persisted caches?
2. Expire the access token and fire several requests at once: how many refresh calls? Invalidate
   the refresh token: clean logout or a loop?
3. Log out, then inspect: secure store, caches, local DB, push token on the server, crash-reporter
   user. Sign in as another user: anything left from the first?
4. Export the production bundle (`npx expo export` with production env) and grep it for keys,
   secrets, internal hostnames and debug flags; list every inlined env var and judge each.
5. Open every deep link the diff adds with hostile input (`npx uri-scheme open`,
   `xcrun simctl openurl`, `adb shell am start -a android.intent.action.VIEW -d`): logged out,
   another tenant, invalid and foreign ids, extra params, `javascript:` URLs. Does anything happen
   without a confirmation in the app?
6. WebView: navigate to a page outside the allowlist; send a forged bridge message.
7. Trigger an error in the flow and read the crash report and breadcrumbs: tokens, headers,
   personal data?
8. Open the app switcher on each sensitive screen.
9. Call the API with the app's token for an action the UI hides from this role.
