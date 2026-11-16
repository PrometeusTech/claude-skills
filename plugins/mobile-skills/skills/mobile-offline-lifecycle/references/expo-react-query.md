# React Native / Expo specifics: network, cache, lifecycle, storage

Check the API of the versions in the project's lockfile before relying on a name below.

## Network state and focus (TanStack Query)
- Online state: `onlineManager.setEventListener` fed by `@react-native-community/netinfo`.
  `isInternetReachable` can be `null` at first — do not treat `null` as offline.
- Focus: `focusManager.setFocused` from an `AppState` listener (`active` → focused), so stale
  queries refetch on return to the foreground. Remove the listener on cleanup.
- Timeouts: `fetch` has none by default; use an `AbortController` with a timer (or the HTTP
  client's timeout) and pass `signal` from the query function so cancelled queries abort.

## Persisting the cache
- `PersistQueryClientProvider` with an async persister (`@tanstack/query-async-storage-persister`)
  over AsyncStorage, or a sync persister over MMKV.
- Set `maxAge`, and `buster` to a value that changes when the data shape changes (e.g. a cache
  schema version, or the runtime version / update id when unsure).
- `dehydrateOptions.shouldDehydrateQuery` to persist only the queries meant for offline use —
  never tokens or sensitive records.
- Logout / tenant switch: `queryClient.clear()` and remove the persisted client.

## Queued (paused) mutations
- Mutations made offline are paused (`networkMode`). To resume them after a restart, register
  `queryClient.setMutationDefaults(key, { mutationFn })` for each mutation key before restoring,
  and call `queryClient.resumePausedMutations()` once the persisted cache is restored.
- Put the idempotency key in the mutation **variables** (generated when the user acts), not
  inside `mutationFn`, so the replay sends the same key.
- On logout, clear the mutation cache too.

## Storage
- AsyncStorage: unencrypted, slow for large values; on Android the default database size is
  limited (a few MB). Fine for small preferences and caches that can be rebuilt.
- MMKV: fast, synchronous; encryption optional — still not the place for tokens (use the secure
  store, see mobile-security).
- `expo-sqlite`: track the schema with `PRAGMA user_version` (or the ORM's migrations, e.g.
  Drizzle) and migrate in order at startup, in a transaction.
- Files: `expo-file-system` cache directory for temporary files (the OS may purge it), document
  directory for files the app must keep; delete temporary uploads after success.

## Lifecycle and background
- `AppState` (`active` / `background` / `inactive`) for foreground and background handling.
- Background tasks: `expo-task-manager` with `expo-background-task` (scheduled, best-effort,
  minimum intervals set by the OS). Do not build correctness on them.
- Drafts: persist form state (debounced) to storage keyed by user and record; clear on submit
  success and on logout.

## Testing
- Jest (`jest-expo`) with NetInfo's jest mock to simulate offline / online transitions; fake
  timers for retries and backoff; tests that restore a persisted cache with an old shape.
- Manual: Android emulator network profiles and airplane mode, iOS Network Link Conditioner (in
  Developer settings on a device), "Don't keep activities" in Android developer options.
