# React specifics for data-heavy pages

- Fetch in one place per page (a hook / view-model / manager), not in several components that each
  fetch the same thing. Deduplicate in-flight requests for shared data (options, profile).
- React 18+ StrictMode runs effects twice in development: make effects idempotent (abort on
  cleanup) so you do not "fix" duplicates that exist only in dev — and do not ignore real ones.
- Search inputs: debounce (~300 ms); send the request with the current filters; ignore responses
  whose request id is older than the latest (or abort the previous request).
- Pagination state lives in the URL (query string) when users share or reload pages.
- Show a subtle loading state when changing pages (keep the previous rows visible), a skeleton only
  on first load; reserve height to avoid layout shift.
- Lists over a few hundred rows: paginate server-side; if the product needs infinite scroll, use
  virtualization. Check DOM node count in DevTools on the largest realistic tenant.
- Heavy routes (admin areas) are `React.lazy` chunks; shared styles are loaded from the entry so
  lazy chunks do not change CSS order.
- Measure: network requests per page load and per filter change, Lighthouse TBT/LCP on a mobile
  profile, bundle size before/after.
