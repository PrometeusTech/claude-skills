---
name: mobile-push-deeplinks
description: Rules and review checklist for the device side of push notifications and links into a mobile app (React Native / Expo, mostly platform-neutral) — notification permission and refusal, push tokens per user and device (token changes, several devices, logout, account or tenant switch, separate dev and production credentials), lock-screen content, Android channels, badges, notifications received in the foreground, routing a tap with the app open, in the background or killed; universal links / App Links / custom schemes (verification files, signing-certificate fingerprints, links opened before sign-in, web fallback, links in e-mails). Use it whenever you add or change a push notification, its payload or tap target, token registration, notification preferences, a deep link or the app's link handling, and whenever you review such changes — even if the task only says "notify the user when…" or "open the booking from the e-mail". Sending (when, to whom) is in notifications-integrations.
---

# Push notifications and links into the app

Push and links fail quietly: a notification that goes to the previous user of the phone, a tap
that opens the home screen instead of the booking because the app was not running, a universal
link that opens the browser because the verification file has the wrong certificate. The rules
below cover the device side; notifications-integrations covers sending. Expo specifics are in
`references/expo.md`.

## 1. Permission

- Ask when the value is obvious (after the user does something notifications help with), with a
  short explanation first. On iOS the system prompt appears once; a refusal is final until the user
  changes it in Settings.
- Handle every state: not asked, granted, denied, provisional / limited. A refusal never blocks
  the app.
- The app's notification settings show the real OS state and link to the OS settings when denied;
  returning from Settings re-checks the state and registers the token if now granted.
- Android 13+ needs the runtime permission; the channel must exist before the prompt can appear.

## 2. Device tokens

The server keeps tokens per **(user, device installation, app environment)**.

- Register (idempotent upsert) after sign-in, at launch, and whenever the OS reports a new token.
- A user can have several devices; each gets its own messages.
- Logout unregisters the token on the server **before** local credentials are cleared. If the
  device is offline at logout, the server still stops sending: revoking the session removes the
  tokens linked to it.
- A different user signing in on the same device takes the token over; the previous user no
  longer receives anything there. A tenant switch either keeps receiving every tenant's
  notifications (and the tap switches tenant explicitly) or only the active one — decide and
  implement one.
- Tokens the provider reports as invalid (unregistered, uninstalled) are deleted
  (notifications-integrations §2).
- Development and preview builds use separate push credentials / environment, so test sends never
  reach production users and production sends never reach test devices.

## 3. What a notification contains

- The lock screen is public. No sensitive content (document contents, health, codes, amounts the
  user would not want seen) unless the user opted in; a generic text plus an in-app detail works.
- The payload carries a type and ids (or a route), not data. The app fetches the data with
  authorization when opened — and the record may be gone or no longer visible.
- Text is localized on the server in the recipient's language (notifications-integrations §4);
  payload size limits respected.
- Android channels exist per category, with the right importance, created before the first
  notification of that category; in-app preferences match the server's preferences and the
  channels.
- Badge counts match a server-side unread count and are cleared when read.

## 4. Received while the app is open

- Foreground behavior is chosen per type: system banner, in-app toast, or silent refresh — never
  both a banner and a toast.
- Receiving a notification invalidates the data it is about (the list, the counter), so the
  screen is current.
- Silent / data-only pushes are an optimization: the OS throttles them and may not deliver them
  (e.g. after the user force-quits on iOS). Nothing depends on them.

## 5. Tapping a notification

- One routing function handles notification taps **and** links (§6), so both get the same
  validation (mobile-security §5).
- Cold start: when the tap launched the app, the initial notification response is read at startup
  and the intent is kept until the session is restored and the navigator is mounted — then
  routed. Background and foreground taps go through the same path.
- The target is checked: logged out → sign in, then continue; another account or tenant →
  explicit switch or a clear refusal; record deleted or no longer visible → a "not found" screen,
  not a crash or a blank page.
- Each tap is handled once — not again on re-render, on return from the background, or after a
  reload.

## 6. Universal links, App Links and custom schemes

- The web domain serves the verification files — `/.well-known/apple-app-site-association` and
  `/.well-known/assetlinks.json` — over HTTPS, without redirects, with the right app ids, team id
  and the **release** signing-certificate SHA-256 (with Play App Signing, the Play signing key, not
  the upload key). Each build variant that should open links is listed.
- Paths are scoped to what the app handles; every linked path also has a working web page for
  users without the app (or on desktop).
- Links in e-mails use the universal-link domain directly. Click-tracking redirects from e-mail
  providers break universal links — disable tracking for those links or make the tracking domain
  verified too.
- Custom schemes are for returns into the app (OAuth callback), never for links that carry
  secrets.

## Implementation checklist

- [ ] Permission asked in context; every state handled; settings show the OS state; re-check on return.
- [ ] Token upserted per user + device + environment; refreshed on change; unregistered before logout; invalid tokens deleted; separate dev / prod credentials.
- [ ] Payload: ids not data, nothing sensitive on the lock screen, localized, size-safe.
- [ ] Channels per category; preferences mirror the server; badge consistent.
- [ ] Foreground behavior chosen; data refreshed on receive; no dependence on silent pushes.
- [ ] One routing function; cold-start tap queued until ready; target validated; handled once.
- [ ] Verification files correct for the release signing key; web fallback per path; e-mail links not wrapped by tracking.

## Review / judge checklist (try to break it)

1. Fresh install, deny the permission: does the app work? Does Settings show the state? Allow it
   later in the OS settings and return: is the token registered?
2. Sign in on two devices and send: both receive. Log out on one: it stops (check the server's
   token table). Sign in as another user on that device: the first user's notifications no longer
   arrive there.
3. Tap a notification with the app killed, in the background and in the foreground; while logged
   out; while in another tenant; for a record deleted in the meantime.
4. Look at the lock-screen preview of each new notification type, in each language.
5. Background and resume the app after a tap-driven navigation: does it navigate again?
6. Fetch the verification files from the production domain and validate them (Apple's and Google's
   tools, `adb shell pm get-app-links <package>`); open each new link with the app installed, not
   installed, and on desktop.
7. Send to an invalid token: is it removed after the provider's error?
8. Send a test push from a preview build's credentials: can it reach a production install?
