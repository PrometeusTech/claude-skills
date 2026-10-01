---
name: web-security
description: Rules and review checklist for web-application security beyond authorization — stored and reflected XSS, mass assignment / over-permissive params, open redirects, SSRF on user-supplied URLs, CORS and CSRF on cookie-authenticated endpoints, secrets and personal data in logs, job arguments, error trackers and analytics, rate limiting and enumeration on public endpoints, and silent failures that hide attacks or data loss. Use it whenever you add or change an endpoint, a param list, a redirect, a fetch of a URL, a cookie, CORS/CSP config, logging, analytics events, error handling or a public (unauthenticated) endpoint, and whenever you review such changes — even if the task never says "security". Pair it with authz-multitenancy (who may do what) and file-uploads-cdn (files).
---

# Web security beyond "who may do what"

Authorization decides who may call an endpoint (see the authz-multitenancy skill). This skill is
about what an allowed — or anonymous — caller can still make the app do: run script in another
user's browser, write fields they should not, send users to a phishing page, make the server call
internal hosts, or read secrets from logs. Each rule is cheap while building; the checklist is
what a reviewer tries.

For each finding, name the attacker (who controls the input), the path (source → sink, with
lines) and the gain beyond what that attacker could already do. No attacker, no gain → not a
finding.

## 1. Output encoding (XSS)

- Render user text as text. In React that is the default; the risks are
  `dangerouslySetInnerHTML`, `href={userValue}` (`javascript:` URLs), `src`/`srcDoc`, markdown or
  rich-text renderers without a sanitizer, and HTML built in strings (emails, PDFs, exports).
- Server-rendered HTML, emails and PDFs escape by default; `html_safe` / `raw` on user content is a
  finding unless sanitized with an allowlist.
- URLs from users are validated to `http(s)` before being rendered as links.

## 2. Writes accept only the intended fields

- Strong params list exactly what this role may set. Fields like `client_id`, `role`, `status`,
  `user_id`, `owner_id`, `price`, `verified`, `*_at` come from the server, not from the body.
- Different roles that may set different fields → different param lists, chosen after
  authorization.
- Nested attributes (`*_attributes`) and JSON columns are writes too: check what they allow.

## 3. Redirects and outbound requests

- **Open redirect:** redirect targets from params (`return_to`, `next`, `redirect_url`) are
  relative paths or an allowlisted host; `//evil.com` and `https:evil.com` are refused.
- **SSRF:** when the server fetches a URL the user supplied (webhooks, imports, previews, avatars
  by URL), allowlist schemes and hosts, resolve the name and refuse private, loopback and
  link-local addresses (including after redirects), and set timeouts and a size cap.

## 4. Cookies, CORS, CSRF

- Endpoints authenticated by a cookie are CSRF targets: `SameSite=Strict/Lax`, an Origin/Referer
  check or a CSRF token, and no state change on GET.
- CORS with `credentials: true` only for an explicit list of origins, never a reflected Origin or
  `*`; preflight caching does not widen it.
- Session and refresh cookies: `HttpOnly`, `Secure`, narrowest `Path`, no token in a body, URL or
  localStorage.

## 5. Secrets and personal data stay out of side channels

- Logs: password, token, key, code, e-mail and phone params are filtered
  (`filter_parameters` or equivalent); new param names that carry secrets are added to the filter.
- Background job arguments are logged by the framework: pass ids, not tokens, passwords or
  documents.
- Error trackers (Sentry) and analytics (PostHog, GA): no e-mails, phones, addresses, tokens or
  free-text user content in event properties, breadcrumbs or URLs; check new events in the diff.
- API errors do not echo internal details (SQL, stack traces, storage keys, other users' data).
- New secrets come from the environment, are documented in the env template, never committed.

## 6. Public endpoints and enumeration

- Any unauthenticated endpoint (login, password reset, invitations, share links, webhooks,
  check-email) is rate limited per IP and per target (account / e-mail / token).
- Same response, status and timing class for existing and non-existing accounts or tokens.
- Signed tokens: expiry, single use where it matters, constant-time comparison, scope bound to
  the resource.

## 7. Failures are loud

Silent failures hide both attacks and data loss:

- No empty `rescue` / `catch`; no `rescue => e` (or `catch (e)`) that only logs and continues when
  the caller then reports success.
- Catch the specific error you expect; let unexpected ones reach the error tracker.
- Fallbacks (default values, cached data, a "safe" branch) are deliberate, visible to the user
  when they change the result, and logged.
- Retries that give up tell the user and the tracker.
- Frontend: a failed request shows an error state, not an empty list that looks like "no data";
  `?.` chains do not silently skip a required step.

## Implementation checklist

- [ ] No raw HTML from users; links validated to http(s); emails/PDFs escape user content.
- [ ] Param lists per role; server-owned fields never permitted.
- [ ] Redirect targets relative or allowlisted; outbound fetches of user URLs guarded (SSRF).
- [ ] Cookie-authenticated writes protected (SameSite + Origin/CSRF); CORS origin list explicit.
- [ ] New secret-bearing params filtered; job args carry ids; analytics/Sentry events free of PII.
- [ ] Public endpoints rate limited, uniform responses, tokens expire.
- [ ] Error handling specific; no swallowed errors; fallbacks visible and logged.

## Review / judge checklist (try to break it)

1. Store `<img src=x onerror=alert(1)>`, `javascript:alert(1)` and `"><svg onload=…>` in every new
   text/URL field; view it in the app, in emails, PDFs and exports.
2. Send extra fields in each write (`client_id`, `role`, `status`, `user_id`, `*_at`) — are they
   ignored? Compare the stored row.
3. Redirect params with `//evil.example`, `https:evil.example`, `/\evil.example`.
4. User-supplied URLs pointing at `http://127.0.0.1`, `http://169.254.169.254`, a private IP, a
   host that redirects there.
5. Cookie-authenticated write from a foreign Origin, without Origin, with a GET.
6. `grep` the diff for new params, job arguments, log lines and analytics events; send a request
   with a password/token/e-mail and read the log, the job log and the captured event.
7. Public endpoints: existing vs non-existing account → same status, body and timing class;
   hammer it → throttled.
8. Inject failures (stub a dependency to raise) and check that the user sees an error and the
   tracker gets an event — no "success" with nothing done.
