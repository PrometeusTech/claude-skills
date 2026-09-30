# Rails + Pundit specifics

## Safety net
- In `ApplicationController`: `after_action :verify_authorized` for every action and
  `after_action :verify_policy_scoped, only: :index` (or an equivalent concern). Public
  controllers call `skip_after_action :verify_authorized` (or a named helper like
  `skip_pundit_verification`) with a comment explaining why they are public.
- A 404 rendered before `authorize` runs belongs in a `before_action`; rendering it inside the
  action and returning early makes `verify_authorized` raise.

## Tenant concern
- One concern (e.g. `TenantScopable`) resolves `current_tenant` from the session selection,
  checks membership, and exposes `current_tenant` / `current_tenant_id`. Controllers never read
  `params[:tenant_id]` for scoping without a membership check.
- Nested routes (`/tenants/:tenant_id/...`): load the tenant from the path, authorize membership
  first, then load the child **through** the tenant.

## Policies
- `ApplicationPolicy` gets helpers such as `member_of?(tenant)`, `role_in?(tenant, *roles)`,
  `in_selected_tenant?(record)`. Policies call them with explicit role lists per action.
- Collection actions authorize a context object: `authorize Document.new(tenant: current_tenant)`
  or `authorize [:tenant, current_tenant], :index?`, then `policy_scope(Document)`.
- `Scope#resolve` restates the show rule in SQL (same roles, same visibility filters). Test them
  together.
- Platform roles: a single helper (e.g. `platform_admin?`) used only in platform policies; tenant
  data policies never call it as a bypass.

## Responses
- `rescue_from Pundit::NotAuthorizedError` → 403 JSON with a translated message (one handler).
- For foreign records, load via the scope so a foreign id raises `RecordNotFound` → map both
  foreign and missing to the same status, or authorize the parent first and return 403 for both.

## Serializers
- One serializer per audience (`Public`, `Member`, `Manager`); controllers pick by policy result.
- Never `render json: model` or `serializable_hash(include: ...)` on models with personal or
  internal fields.

## Tests
- Policy unit tests with a role matrix table; request tests for foreign/missing ids and anonymous.
- A small IDOR smoke task that, for each tenant-scoped family, tries a record from tenant B while
  logged into tenant A and fails on any 2xx.
