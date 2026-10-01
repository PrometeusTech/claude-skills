---
name: authz-multitenancy
description: Rules and review checklist for authorization and tenant isolation — who may read or change what, in apps where data belongs to a tenant (organization, account, workspace, project, team, building). Use it whenever you add or change an endpoint, controller action, policy, role, permission, admin screen, route guard, serializer field, export, cache, background job that reads tenant data, or "only admins can…" / "members can see…" rules, and whenever you review such changes — even when the task never says "security" or "authorization".
---

# Authorization and multi-tenancy

Most serious bugs in multi-tenant apps are not exotic: an action that forgot to authorize, a list
that returns more than the detail page allows, a global "super admin" flag that silently reads
every tenant, an ID from another tenant that answers 404 instead of 403 and so confirms it exists.
These rules keep the backend the single source of truth; the frontend only reflects it.

Stack specifics:
- `references/rails-pundit.md` — Rails with Pundit (policies, scopes, tenant concern)
- `references/react-guards.md` — React route guards, menus and role derivation

## 1. Fail closed

Every action authorizes, and the framework **checks that it did** (e.g. an `after_action` that
raises when nothing was authorized). Public endpoints (health, signed webhooks, signed share links,
docs) opt out explicitly, in their own code, with a one-line reason. A new controller that forgets
authorization must fail in tests, not ship open.

## 2. The tenant comes from the server

- The current tenant is chosen server-side (session/selection + membership check), not taken from
  a request parameter. When an endpoint does accept a tenant id (nested routes, admin tools), it
  checks membership/role **in that tenant** before loading anything.
- Records are loaded through the tenant: `current_tenant.documents.find(id)` or a policy scope,
  never a bare `Document.find(params[:id])` followed by a check you might forget.
- A user who belongs to several tenants acts only in the selected one; a record from another of
  their tenants is refused while that one is not selected (unless the product decides otherwise).

## 3. Roles are per tenant; platform roles are not a skeleton key

- Permissions derive from the user's role **in the record's tenant** (owner, admin, manager,
  member, viewer, staff…). Name the exact roles for each action; do not reuse a broad helper
  whose role list happens to include roles the product did not approve (e.g. a "can manage" helper
  that also admits a role you did not intend).
- Platform/global roles (super admin, staff) cover platform actions only (creating tenants,
  catalogs, support tooling). They do not read a tenant's data unless they also hold a role in it
  or the product explicitly allows it — and then it is audited.
- Inactive, removed or expired memberships grant nothing; role changes take effect on the next
  request (no long-lived cached permissions).

## 4. Lists mirror details

A list endpoint's scope returns exactly the records whose detail endpoint would authorize — same
rules, same tenant, same visibility (e.g. "internal" notes, records of another user or sub-group). Write the
scope from the policy, and test both with the same role matrix.

## 5. Foreign and missing look the same

For records outside the caller's tenant, respond identically whether the id exists or not (pick
403 or 404 for the app and use it consistently). Otherwise the response enumerates other tenants'
ids. Authorize the parent (tenant) **before** looking up the child.

## 6. Authorize data, not just endpoints

- Serializers return only the fields that audience may see (contact data, internal notes, audit
  fields, storage keys). Different audiences get different serializers, not `if` sprinkled in one.
- Side channels count: exports, PDFs, emails, notifications, webhooks, search suggestions, counts,
  autocomplete ("tags used in the tenant"), error messages. Each runs the same checks.
- Background jobs re-load records and re-check tenant ownership; they do not trust ids captured at
  enqueue time blindly.
- Caches are side channels too: a cached response, fragment or computed value whose content
  depends on the tenant, the role or the user has all of them in its key (`[tenant_id, role,
  user_id, record.cache_key_with_version]`), or one user is served what was cached for another.
  Invalidate on membership/role change.

## 7. Frontend reflects, backend enforces

- Menus, buttons and route guards hide what the user cannot do — for UX. The backend still refuses.
- Derive the role from the **selected** tenant in the user profile, not from "any tenant where they
  are admin" or a global flag.
- A guard must check the response body (e.g. an explicit `allowed: true`), never treat any 200 as access.
- A 403 from an action inside a page shows a message; it should not bounce the user to another page
  unless that is the product decision.

## Implementation checklist

- [ ] Action authorizes; the verify-authorized safety net is active; public exceptions explicit.
- [ ] Record loaded through the tenant / policy scope; tenant from the server.
- [ ] Exact roles per action named in the policy; platform roles excluded unless intended.
- [ ] Index scope mirrors the show rule; visibility rules (internal, per-user / per-sub-group) applied in both.
- [ ] Foreign existing id and foreign missing id give the same response; parent authorized first.
- [ ] Serializer per audience; side channels (exports, notifications, suggestions) checked.
- [ ] Tests: role × action × {own tenant, foreign tenant existing id, foreign missing id,
      anonymous, inactive member, platform role without membership}.
- [ ] Cache keys include tenant / role / user whenever the cached content depends on them.
- [ ] Frontend: menu and guards follow the selected tenant's role; backend errors handled.

## Review / judge checklist (try to break it)

1. Call every new endpoint as: anonymous, member of another tenant, inactive/removed member,
   each role of the tenant, a platform admin without a role in the tenant. Compare with the
   product's intended matrix — write the matrix down first.
2. Swap ids: own tenant's id → foreign existing id → foreign nonexistent id. Same status and body
   for the last two?
3. Pass a `tenant_id` (or `org_id`, `account_id`) parameter pointing elsewhere; does anything trust it?
4. Compare list vs detail with the same role: can the list reveal something the detail refuses
   (including counts, tag/suggestion lists, search results)?
5. Look for broad helpers used where the product named specific roles — and for policies that
   allow a role which is only refused by some other layer (a base controller, a tenant concern).
   Defense that depends on a layer the policy does not know about breaks when that layer changes;
   report it even if the endpoint currently refuses.
6. Serializer diff: any new field that a lower role should not see?
7. Frontend: open the admin URL directly as a member; check the menu **and every entry point**
   (cards, links from other pages, dashboards) for each role, including users whose only role is
   a restricted one (e.g. staff-only or read-only) — a visible link that ends in a redirect is a bug; check that
   a 403 inside the page does not log the user out or redirect unexpectedly.
8. Check the tests really exercise the matrix (not only the happy admin path).
9. `grep` the diff for `Rails.cache`, `cache(`, `fetch(`, memoization in class variables: request
   the cached thing as user A (admin, tenant 1), then as user B (member, tenant 2) — does B see
   A's version?
