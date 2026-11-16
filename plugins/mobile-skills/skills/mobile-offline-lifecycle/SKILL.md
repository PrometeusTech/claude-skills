---
name: mobile-offline-lifecycle
description: Rules and review checklist for a mobile app (React Native / Expo, mostly platform-neutral) that lives on unreliable networks and is paused, backgrounded and killed by the operating system — requests with timeouts and safe retries, offline detection, writes made offline or interrupted mid-flight (queues, idempotency keys, conflicts), cached and persisted data (what is stored, when it is stale, cache versioning across app updates, clearing on logout or tenant switch), refreshing when the app returns to the foreground, background tasks and their limits, form drafts that survive the process being killed, and local databases with migrations. Use it whenever you add or change data fetching, a mutation, caching or persistence, local storage (AsyncStorage, MMKV, SQLite), background work or app-state handling in a mobile app, and whenever you review such changes — even if the task only says "save the form" or "show the list offline".
---

# Offline, flaky networks and the app lifecycle

On a phone the network disappears in a lift, the OS freezes the app when the user switches away and
kills it when memory is short — often while the camera is open. The bugs that follow are rarely
crashes: a booking saved twice, a form lost, a list from yesterday shown as current, the previous
user's data after logout. The rules below make those cases deliberate; the checklist is what a
reviewer does with airplane mode and the app switcher.

React Query, NetInfo, AppState, storage and background-task specifics are in
`references/expo-react-query.md`.

## 1. The network fails by default

- Every request has a timeout. Errors are classified — offline, timeout, 4xx, 5xx — and each has
  its own message and action (retry, fix the input, try later). No infinite spinner.
- Automatic retries only for requests that are safe to repeat (reads, writes with an idempotency
  key), with backoff and a cap; never for 4xx.
- "Connected" is not "reachable": captive portals and very slow links look online. Treat a
  timeout like offline for the user, not like success.

## 2. Writes that can be interrupted

The user taps Save, the network drops or the app is killed: the server may or may not have done it.

- Each create carries an idempotency key generated **once per user intent** and stored with the
  pending write, so a retry — even after a restart — reuses it (idempotency-retries §7).
- The submit control is disabled while the request is in flight.
- After a timeout the UI says the outcome is unknown and reconciles (re-reads, or retries with the
  same key) instead of reporting a failure that invites a duplicate.

## 3. Offline is a product decision, per feature

Decide and write down for each feature: online-only with a clear message, cached read-only, or
queued writes.

- Queued writes are persisted, replayed in order with their original idempotency keys, and shown
  as pending. Each has a defined outcome when the server refuses it on replay (record changed:
  409; record deleted: 404; permission removed: 403) — told to the user, never dropped silently.
- A write queued by one user is never replayed after another user signs in on the device, nor in
  another tenant after a tenant switch.
- Anything with money, stock or bookings (domain-integrity) is confirmed by the server before the
  UI shows it as done.

## 4. Cached and persisted data

- Persisting is a choice per data set: nothing sensitive in unencrypted storage (mobile-security
  §2), and only what the offline decision in §3 needs.
- Persisted caches have a maximum age and a version ("buster") bumped whenever the shape changes —
  an OTA update that reads the previous update's cache with a new shape crashes at launch.
- Cached screens show their freshness when it matters (last updated), and refetch when shown.
- Logout and tenant switch clear the query cache, the persisted cache, queued writes and
  user-scoped local data.
- After a mutation every view that shows the record is refreshed (frontend-quality §3).

## 5. The app lifecycle

- **Return to foreground:** refetch stale data, re-check the session (the token may have expired
  while away), the minimum app version (mobile-release-compat §3) and OS permissions (the user may
  have changed them in Settings); reconnect sockets.
- **Background:** stop polling, timers and sockets; save drafts.
- **Process killed:** long forms and multi-step flows persist a draft, so returning after the OS
  killed the app (very common on Android while the camera or a file picker is open) does not lose
  the input. Restoring the screen state is sensible, not stale (a draft of a deleted record).
- Handlers registered for app-state changes are removed on unmount — no duplicated refreshes.

## 6. Background work is best-effort

- iOS and Android limit background execution (short time after backgrounding, scheduled tasks run
  when the OS decides, battery savers postpone them). Correctness never depends on a background
  task running; the server is the source of truth.
- Uploads and downloads that must continue use an API designed for background transfer, or resume
  on next launch.

## 7. Local databases

- The schema is versioned; migrations run at startup and are tested from **every** shipped schema
  version (users skip releases).
- A failed migration does not brick the app: data that can be refetched is dropped and refetched;
  unsynced user writes are never dropped silently.
- Heavy reads and writes stay off the interaction path; disk-full is handled.

## 8. Device clocks and time zones

The device clock can be wrong and the time zone changes when the user travels. Deadlines, expiry
and "today" come from the server or are computed explicitly in the right zone
(domain-integrity §3).

## Implementation checklist

- [ ] Timeouts on every request; error classes mapped to messages; retries only where safe.
- [ ] Idempotency key per user intent, persisted with the pending write; submit disabled in flight.
- [ ] Offline behavior decided per feature; queued writes ordered, keyed, visible, conflicts handled.
- [ ] Persisted cache: chosen data only, max age, version bumped on shape change.
- [ ] Logout / tenant switch clear caches, queues and user data.
- [ ] Foreground: refetch, session check, version check, permissions; background: polling stopped, drafts saved.
- [ ] Drafts survive the process being killed.
- [ ] No correctness depends on background tasks.
- [ ] Local DB migrations tested from each shipped version; failure path safe.
- [ ] Tests with mocked network: offline, timeout, 409, 404, 5xx, replay after restart.

## Review / judge checklist (try to break it)

1. Turn on airplane mode at each step of the main flow: opening the screen, submitting, and while
   the request is in flight (throttle so it hangs). What does the user see? What is on the server?
2. Tap Save and kill the app immediately; relaunch. Duplicate, lost, or shown as pending?
3. Slow network profile (3G / Edge) and a double tap on submit.
4. Leave the app in the background for longer than the access token lives; return. Refresh,
   expired session and stale data handled?
5. Log out and log in as another user (and switch tenant): any data, list item, count or queued
   write from before?
6. Install the previous release, use it to fill caches and the local DB, then update to the head
   (store build and OTA). Does it start? Cache discarded or migrated?
7. Open the camera / file picker from a long form with "Don't keep activities" enabled on Android:
   is the input still there?
8. Change the device time zone and set the clock a day off: dates, deadlines and expiry correct?
