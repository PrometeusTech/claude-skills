---
name: mobile-release-compat
description: Rules and review checklist for shipping a mobile app (React Native / Expo, mostly platform-neutral) without breaking the versions already installed — old builds that stay on phones for months against a changing API, minimum supported version and forced update, over-the-air (OTA) JavaScript updates versus new store builds (runtime version, native changes), staged rollouts and binaries that cannot be recalled, build-time configuration per environment, version and build numbers, and the App Store / Google Play rules that reject releases (in-app account deletion, privacy manifest, data-safety form, permission strings, login options). Use it whenever a change touches what the app sends to or reads from the API, a backend endpoint a mobile app calls, app.json / app.config / eas.json, native dependencies, permissions, sign-in or account flows, update logic or release scripts, and whenever you review such changes — even if the task only says "rename this field" or "add a library".
---

# Mobile releases and version compatibility

A web deploy replaces the frontend for everyone within hours. A mobile release does not: earlier
builds stay installed for months, some users never update, store review adds days before a fix
reaches anyone, and a binary that is out cannot be recalled. Every API change and every release
is therefore made against **all the versions still in use**, not only the latest. This skill
extends deploy-safety §2 (expand / contract) from hours to months.

Expo / EAS specifics (runtime version, channels, rollouts, rollback, build numbers) are in
`references/expo-eas.md`.

## 1. Know which versions are out there

- The app sends its version, build number, platform, OS version and OTA update id on every
  request (headers such as `X-App-Version`, `X-App-Build`, `X-App-Platform`, `X-App-Update-Id`),
  and the backend records them in logs or metrics. Without this nobody can tell when removing
  something is safe.
- Crash reports and analytics are tagged with the same values, so a regression is attributed to
  the right binary **and** the right OTA update.
- The project has a written support window: the oldest build the API still serves (minimum
  supported) and the policy for raising it.

## 2. The API keeps serving old apps

- Never remove, rename or retype a field, enum value, error code, endpoint or query param that an
  installed build uses, until the version data (§1) shows those builds are gone or below the
  minimum supported version. `git grep` the app at the tags of every supported release, not only
  at `main`.
- New enum values, statuses and error codes reach old apps too. App side, from now on: an unknown
  enum value falls back safely (generic label, action hidden), an unknown error code shows the
  generic translated message, unknown fields are ignored. Backend side: when a shipped build
  cannot cope with a new value, gate the value by app version or map it to one that build knows.
- A new required request param is optional (with a server default) for builds that do not send
  it, or the endpoint is gated by app version.
- Behavior changes old builds would mishandle (pagination format, auth flow, stricter validation,
  a changed meaning of a field) get a new endpoint version or an app-version gate.
- The contract diff (oasdiff or similar) says what changed; the version data says who still
  depends on it. Both go into the PR.

## 3. Minimum version and forced update

- The app checks a server-provided config (minimum supported, recommended) at launch and on return
  to the foreground. Below the minimum: a blocking, translated "update required" screen with the
  correct store link for the platform. Below the recommended: a dismissible prompt.
- That check must already be in the shipped builds to be useful — add it early, and never break
  its endpoint or response shape.
- The backend answers requests from builds below the minimum with a dedicated error (e.g. HTTP 426
  with code `app_update_required`) that every shipped build maps to the update screen or at least
  to a readable message.
- Raising the minimum is a product decision backed by numbers (share of active users below it),
  never a side effect of a feature PR.

## 4. OTA update or store build

- A JavaScript-only change can ship as an OTA update to builds with the **same runtime version**.
  Anything native needs a new store build and a new runtime version: a new or upgraded native
  dependency, a config plugin, a permission, native fields in app.json, an SDK upgrade, icons and
  splash. An OTA update that calls a native module the installed binary lacks crashes at launch.
- The runtime version follows a deliberate policy (the fingerprint policy derives it from the
  native layer) — never bumped or frozen by hand "to make it work".
- OTA updates roll out to a percentage first, are watched by update id (crash-free sessions, error
  rate), then widened. The rollback command is known before publishing.
- Know what the update library does when a new update fails to download or crashes at launch, and
  when the app checks and applies updates (launch, resume, never mid-flow).
- Store policies allow OTA bug fixes and behavior changes but not changing the app's primary
  purpose or shipping features that would need review. Large features go through a store build.
- Upload source maps for **every** OTA update, not only for store builds, or stack traces of the
  updated code are unreadable.

## 5. Store builds are staged and cannot be recalled

- A shipped binary is fixed until users update; the fix is another review cycle. Release with a
  staged rollout (Google Play percentage, App Store phased release) and halt it if crashes rise.
- Marketing version and build number come from one source, are incremented by the build service,
  never reused, and match across platforms for the same release.
- Risky features sit behind remote flags. The default compiled into the binary is the one that
  applies offline and forever on that build — make it the safe one.
- When sign-in is required, store review gets a working demo account with realistic data.

## 6. Build-time configuration

- One configuration per environment (development, preview, production): API base URL, bundle id /
  package name, app name and icon variant, push credentials, analytics keys. A preview build never
  talks to production data by accident, and a production build never to staging — verify in the
  built artifact's resolved config, not in the source.
- The production config fails the build when a required value is missing or points to localhost.
- Everything in the app config and the JS bundle is public (mobile-security §4): no secret in an
  env var that gets inlined.

## 7. Store rules that reject releases

These change every year; when a change touches them, read the current App Store Review Guidelines
and Google Play policies rather than relying on memory.

- If users can create an account in the app, they can start deleting it from inside the app
  (Apple), and there is a web page to request deletion (Google Play). Deletion really deletes or
  anonymizes the data (web-security §8).
- Privacy declarations stay true: the iOS privacy manifest (required-reason APIs, tracking
  domains) and App Store privacy labels, the Google Play data-safety form. Update them when a
  change collects new data or adds an SDK that does.
- Every permission has a specific purpose string, translated. No permission the app does not use
  — libraries add permissions to the merged manifest; check it.
- Offering third-party or social login brings the App Store requirement of an equivalent
  privacy-focused login option (Sign in with Apple qualifies).
- Digital goods and subscriptions consumed in the app use in-app purchase where the store requires
  it; physical goods and real-world services do not.
- Target SDK / build SDK minimums follow the stores' yearly deadlines.

## Implementation checklist

- [ ] App version, build, platform and update id sent on every request and tagged in crash reports.
- [ ] No field / enum / code / endpoint used by a supported build removed or retyped; new values safe for old builds (fallback or gate).
- [ ] App code tolerates unknown enum values, error codes and fields.
- [ ] Minimum-version check and `app_update_required` handling in place and untouched.
- [ ] Native change? → new store build and runtime version; JS-only → OTA allowed. Stated in the PR.
- [ ] Rollout plan (percentage, what to watch, rollback command); source maps for the update.
- [ ] Per-environment config correct in the built artifact; no secrets in public config.
- [ ] Store-facing changes: account deletion, privacy manifest / labels / data safety, purpose strings, merged-manifest permissions, login options, IAP rules.

## Review / judge checklist (try to break it)

1. List every field, enum value, error code, param and endpoint the diff removes, renames,
   retypes or adds. For each, `git grep` the app at the tag of the oldest supported release and at
   the current one: which builds read or send it?
2. Run the API client tests (or the app) of the oldest supported release against the **head**
   backend. Anything that breaks for a build still in use is a finding.
3. Make the backend return a new enum value / error code to the current app: fallback or crash?
4. Compare the fingerprint (or the native files) of base and head. Native changes planned as an
   OTA update are a finding (High: crash at launch on every device that takes the update).
5. Send a request as a build below the minimum: dedicated code returned? Update screen shown?
6. Inspect the resolved production config (`npx expo config --type public` with production env):
   API URL, bundle id, no secrets.
7. New dependency, permission or data collection: privacy manifest, labels / data-safety form,
   purpose strings, merged-manifest permissions updated?
8. Accounts created in the app: can the user delete theirs from the app, and does it happen?
