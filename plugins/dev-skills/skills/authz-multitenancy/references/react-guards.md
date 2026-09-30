# React specifics for authorization UX

- Derive capabilities from the user's role in the **selected** tenant (profile / session data),
  in one module (e.g. `roleCapabilities`), with tests. Components ask "can I manage documents?",
  not "is role === 'admin'".
- Route guards (`AdminRoute`, `ManagerRoute`…) read that module. When they call a check endpoint,
  they read the boolean in the body; a 200 alone is not permission.
- Menu entries use the same capability function as the guard, so a hidden link and a blocked
  route never disagree.
- Global HTTP interceptors: decide deliberately what a 403 does. A page-level 403 inside an action
  (e.g. "you can't delete this") should show a message; some clients opt out of a global
  "redirect home on 403". Keep 401 (session expired) separate from 403 (not allowed).
- After switching tenant, reload capabilities (and ideally the page state); never keep the
  previous tenant's permissions cached.
- Tests: capability function per role; guard redirects; menu snapshot per role; an E2E (mocked)
  check that a member opening the admin URL directly is redirected.
