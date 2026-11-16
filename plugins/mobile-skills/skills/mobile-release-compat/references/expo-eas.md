# Expo / EAS specifics for releases

Commands and options change between SDK versions — confirm against the docs of the project's Expo
SDK (Context7 or docs.expo.dev) before relying on a flag.

## Runtime version and the native layer
- `app.json` / `app.config.ts`: `"runtimeVersion": { "policy": "fingerprint" }` derives the runtime
  version from the native layer, so an OTA update only reaches binaries with the same native code.
  `appVersion` policy ties it to the marketing version instead (then every native change must bump
  the version — easy to forget).
- To know whether a change is native: compare fingerprints of base and head
  (`npx @expo/fingerprint` locally, or `eas fingerprint:compare` against a build or an update id).
- `npx expo install --check` — dependencies at versions compatible with the SDK (exits non-zero in
  CI). `npx expo-doctor` — common config and dependency problems.

## Builds (`eas.json`)
- Build profiles per environment (`development`, `preview`, `production`), each with its
  `channel` and env. `app.config.ts` reads an environment variable (e.g. `APP_VARIANT`) to set the
  bundle id, name and icon, so preview and production can be installed side by side.
- `"cli": { "appVersionSource": "remote" }` and `"autoIncrement": true` on the production profile:
  EAS owns the build number, never reused.
- Expo Go does not contain your native modules: anything with native code (push on Android,
  custom config plugins) is tested in a development build.

## OTA updates (`expo-updates` + EAS Update)
- Publish to a channel / branch: `eas update --channel production --message "..."`.
- Staged: `eas update --rollout-percentage 10`, widen with `eas update:edit`.
- Roll back: `eas update:rollback` republishes the previous update group, or rolls back to the
  embedded bundle when there is none.
- In the app: `Updates.updateId`, `Updates.runtimeVersion`, `Updates.channel` — send them in
  request headers and set them as crash-report tags. Version and build: `expo-application`
  (`nativeApplicationVersion`, `nativeBuildVersion`).
- When to check: the `updates.checkAutomatically` setting and an explicit
  `checkForUpdateAsync` / `fetchUpdateAsync` / `reloadAsync` flow. Never reload in the middle of a
  user's task; apply on next launch or at a safe point.
- Sentry with Expo: upload source maps for each `eas update` as well as for builds.

## Store configuration
- Privacy manifest: `ios.privacyManifests` in app config (merged with the ones libraries ship).
- Permission strings: `ios.infoPlist` usage descriptions or the config-plugin options of the module
  (e.g. camera, photo library, location); Android permissions in `android.permissions`, and
  `android.blockedPermissions` to strip ones added by libraries.
- Submit with `eas submit`; set the Play staged-rollout percentage and the App Store phased release
  for the version.
