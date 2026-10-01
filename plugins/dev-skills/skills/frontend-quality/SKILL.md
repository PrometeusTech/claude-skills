---
name: frontend-quality
description: Rules and review checklist for the user-facing quality of a web frontend change (React, but mostly framework-neutral) — accessibility measured with axe and the keyboard, translations complete in every locale with no hard-coded strings, data that stays fresh after a mutation (cache invalidation, optimistic updates and rollback), navigation that survives refresh, deep links and the back button, error boundaries and error states, Content Security Policy for new resources, and the per-page loading cost. Use it whenever you add or change a page, a form, a modal, a list, a route, a translation, a data-fetching hook or a third-party script, and whenever you review such changes — even when the task only says "add a button" or "show the documents".
---

# Frontend quality

Most frontend bugs that reach users are not crashes: a list that still shows a deleted item, a
modal that keyboard users cannot leave, an untranslated string on a localized page, a link that works
only when clicked from the menu, a page that breaks on refresh. The rules below are part of
"done"; the checklist is what a reviewer does in a real browser.

## 1. Accessibility you can measure

- Every input has a visible label (or `aria-label`), errors are announced (`aria-describedby`,
  `role="alert"`), buttons are buttons (not clickable `div`s), icon-only buttons have a name.
- Keyboard: every action reachable with Tab / Enter / Space / Esc, visible focus, focus moves into
  a modal and returns to the trigger when it closes, no keyboard trap.
- Contrast and target size follow WCAG AA (component libraries help but custom styles can break it).
- Run `@axe-core/playwright` (or the axe browser extension) on each new page and modal; zero
  serious/critical violations introduced by the change.

## 2. Translations are complete

- No user-facing string hard-coded in components; every key exists in **every** locale file, in
  the right namespace.
- Interpolations and plurals use the i18n library (not string concatenation; not `to_sentence`
  style helpers on the server that only know one language).
- Server error codes map to translated messages; an unknown code falls back to a generic
  translated message, never the raw server string.
- Dates, numbers and currency use locale formatting.

## 3. Data stays fresh after a change

- After create / update / delete, every view that shows the record updates: the list, the
  counter, the detail, suggestion lists (e.g. tags) — refetch or update the cache explicitly.
- Optimistic updates roll back (and say so) when the request fails.
- A record deleted elsewhere (another tab, another user) is handled: the action returns "not
  found" and the UI removes the row, rather than a permission error or a stuck spinner.
- Requests are not repeated needlessly (StrictMode double effects, parent and child both
  fetching); stale responses do not overwrite newer ones (see db-performance §6).

## 4. Navigation and state survive the browser

- Every new route works on direct load and refresh (deep link), including for each role's guard.
- Back / forward restore the previous page, filters and pagination where users expect it (URL
  query params for filters and page numbers).
- Forms with unsaved input warn before navigating away when losing it would hurt.
- Error boundaries around lazy routes and risky widgets; a chunk load failure after a deploy
  offers a reload, not a white page.

## 5. States

Each data view has loading, empty, error and "no permission" states, and each mutation has
in-flight (disabled submit), success and failure feedback.

## 6. Content Security Policy and third parties

- New external resources (CDN downloads, images, fonts, iframes, analytics, maps) are allowed by
  the CSP that production actually sends — test with CSP enforced, not only in dev.
- No new third-party script without a reason; loaded async, consent-aware where required.

## 7. Cost per page

- New pages are lazy routes; admin-only code does not land in the entry chunk; heavy libraries
  are imported where used.
- Compare the build output before and after (entry chunk, the new chunk).
- Images sized and lazy-loaded; no layout shift when data arrives.

## Implementation checklist

- [ ] Labels, roles, keyboard path and focus management; axe clean on new pages/modals.
- [ ] All strings translated in every locale; error codes mapped; locale formatting.
- [ ] Lists, counters, details and suggestions refreshed after each mutation; rollback on failure.
- [ ] Deep link + refresh + back work for every new route and role; filters in the URL.
- [ ] Loading / empty / error / no-permission states; submit disabled in flight.
- [ ] CSP allows new resources in production config; no unjustified third-party script.
- [ ] Lazy route; bundle diff checked.
- [ ] Tests: manager/hook unit tests, one mocked end-to-end flow per role.

## Review / judge checklist (try it in a browser)

1. Run axe on every new page and modal (Playwright + `@axe-core/playwright`); list violations
   introduced by the change.
2. Do the main flow with the keyboard only; open and close each modal; check where focus lands.
3. Switch the language: any untranslated string, missing key warning in the console, raw error
   code, wrong date format?
4. Create, edit, delete — then look at the list, counters, filters and suggestion lists without
   reloading. Delete the record in another tab and act on it in the first.
5. Open each new URL directly, refresh it, use back/forward, as each role (including one that must
   be refused).
6. Throttle the network and fail requests (Playwright `route.abort`): loading, error and empty
   states correct? Double-click submit.
7. Run with the production CSP and watch the console for violations (downloads, images, frames).
8. Build base and head; compare chunk sizes; check the new page is a separate chunk.
