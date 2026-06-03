# ADR 0001 — Support Cloudflare Access JWT authentication and group-based authorization

Date: 2026-06-03
Status: proposed
Deciders: kube-argus maintainers

## Context

kube-argus today runs its own interactive login. `cmd/server/auth.go`
auto-detects an auth mode from env:

- `google` — Google SSO via OAuth2/OIDC,
- `oidc` — generic OIDC (Okta, Auth0, Keycloak, Azure AD / Entra ID, Dex),
- `none` — no login; everyone gets `DEFAULT_ROLE`.

In the `google`/`oidc` modes the server drives the full authorization-code
flow (`/auth/login` → IdP → `/auth/callback`), verifies the returned
`id_token` with `github.com/coreos/go-oidc/v3`, reads `email` + `groups`
from the claims, maps a single group (`OIDC_ADMIN_GROUP`) to the `admin`
role, and issues its own HMAC-signed `kubeargus_session` cookie.
`authMiddleware` gates every request on that cookie. Authorization is
binary: `admin` vs `viewer`, enforced by `requireAdmin` / `isAdmin` /
`requireAdminOrJIT` across the handlers.

A growing number of deployments place kube-argus behind Cloudflare Access
(or a similar identity-aware proxy / Zero Trust gateway). In that topology
the proxy authenticates the user against the configured IdP — for example
Microsoft Entra ID or Okta — and forwards a signed JWT on every request
(`Cf-Access-Jwt-Assertion`, also available as the `CF_Authorization`
cookie). The JWT can carry the user's IdP group membership in a `groups`
claim. When kube-argus runs its own OIDC login in this topology, the user
authenticates twice, and we keep maintaining login/redirect/session code
that the proxy already handles. kube-argus has no way today to trust that
JWT.

Forces at play:

- Support Cloudflare Access (and identity-aware-proxy) deployments without
  a redundant second login.
- Authorize on the IdP groups that operators already manage centrally,
  rather than a kube-argus-local notion of identity.
- Zero Trust posture: no unauthenticated request reaches the pod.
- Prefer deleting bespoke login/redirect/session code over growing it.
- Keep kube-argus's deliberately small authorization model (admin/viewer);
  do not take on a database to gain it.

## Decision

Add a Cloudflare Access trust path to kube-argus, scoped to **identity
only** (JWT verification + role mapping), reusing the existing session and
role plumbing:

1. New `cmd/server/cfaccess.go` — a small JWKS verifier. Use
   `oidc.NewRemoteKeySet` against `<team-domain>/cdn-cgi/access/certs` and
   `oidc.NewVerifier` with the Access application AUD as `ClientID`.
   (`go-oidc/v3` is already a direct dependency; `go-jose/v4` comes
   transitively. No new modules.)
2. A new branch at the top of `authMiddleware`: read
   `Cf-Access-Jwt-Assertion` (fall back to the `CF_Authorization` cookie),
   verify it, extract `email` + `groups`, and populate the **existing**
   `sessionData{Email, Role}` in the request context.
3. Extract the existing admin-group/admin-email logic from `authCallback`
   into a shared `roleFromClaims(email, groups)` helper, reused by both the
   legacy OIDC callback and the new Cloudflare path.

Because every handler already reads identity through
`r.Context().Value(userCtxKey).(*sessionData)`, populating that same struct
from the Cloudflare JWT means **no handler, and no frontend code, changes.**
The IdP group that confers admin is configured via the existing
`OIDC_ADMIN_GROUP`, set to whatever identifier the proxy passes through in
the `groups` claim.

Cloudflare mode is **mutually exclusive** with the interactive `google` /
`oidc` modes: when `CF_ACCESS_TEAM_DOMAIN` + `CF_ACCESS_AUD` are set,
`initAuth` selects `cfaccess` and the built-in `/auth/login` flow is
disabled, so users are never prompted to log in twice.

Authorization stays binary (`admin`/`viewer`). We do **not** introduce a
database-backed policy engine or an admin CRUD surface; kube-argus has no
datastore (JIT and audit are in-memory with file persistence), and the
admin/viewer model does not need rule evaluation.

## Options Considered

### Option A: Trust the Cloudflare Access JWT (identity-only) — chosen

| Dimension | Assessment |
|-----------|------------|
| Complexity | Low–Medium (~one new file + one middleware branch) |
| New code | ~150 LOC incl. tests; reuses existing role logic and `sessionData` |
| External dependency | Cloudflare Access (or equivalent IAP) in front |
| Operational fit | High — Zero Trust front door, no second login |
| Team familiarity | High — repo already verifies OIDC tokens with `go-oidc` |

**Pros:** First-class support for Access/IAP deployments; Zero Trust front
door; deletes bespoke login UX in that mode; no new dependency; no
handler/frontend changes; reuses verification machinery already in the repo.
**Cons:** Depends on correct proxy/IdP configuration (groups claim, per-app
AUD); the `OIDC_ADMIN_GROUP` value must match the identifier the proxy
emits.

### Option B: Point the existing OIDC mode directly at Entra/Okta

| Dimension | Assessment |
|-----------|------------|
| Complexity | Lowest (configuration only) |
| New code | ~0 |
| External dependency | An IdP app registration for kube-argus |
| Operational fit | Low when an IAP also fronts the host |
| Team familiarity | High |

**Pros:** Cheapest; the `oidc` mode already supports Entra/Okta.
**Cons:** Keeps app-specific login/callback/session code; if Cloudflare
Access (or any IAP) also fronts the host, users authenticate twice; the
group claim is still subject to the IdP's GUID/overage behaviour with no
proxy-side normalization.

### Option C: Add a database-backed policy engine

| Dimension | Assessment |
|-----------|------------|
| Complexity | High (adds a datastore + admin UI) |
| New code | Large — policy tables, migrations, CRUD, cache |
| External dependency | A database kube-argus does not currently run |
| Operational fit | Disproportionate to the authz model |
| Team familiarity | Medium |

**Pros:** Per-group / per-namespace allow-deny rules, editable at runtime.
**Cons:** Over-build for an admin/viewer dashboard; introduces a database
and an admin surface kube-argus doesn't have; larger security-review
footprint.

### Option D: Status quo

Keep per-app Google/OIDC login only. **Rejected** — leaves kube-argus
unable to sit cleanly behind an identity-aware proxy and perpetuates
bespoke auth code.

## Trade-off Analysis

The decision is essentially A vs B vs C. **C** is ruled out by cost: a
database and admin UI to express what `admin`/`viewer` already expresses is
not justified and contradicts kube-argus's stateless-except-for-files
design. **B** is cheapest in isolation but optimizes the wrong thing — it
keeps bespoke login code and breaks down the moment an identity-aware proxy
fronts the host (double login), which is the topology this ADR targets.
**A** spends ~1.5–2.5 dev-days to add first-class Access support, removing
login code rather than adding it, and reusing a verification dependency
(`go-oidc`) and pattern already present in the repo. The residual risk in A
is entirely in proxy/IdP configuration (Cloudflare Access + the IdP group
claim) — one-time, well-documented setup.

## Consequences

What becomes easier:

- kube-argus can be deployed behind Cloudflare Access (or another IAP) with
  no second login and no app-specific IdP integration.
- Unauthenticated traffic never reaches the pod.
- In Access mode the interactive login flow, the OAuth state cookie, and the
  self-issued session cookie are no longer exercised — less surface to
  maintain and review.

What becomes harder / new obligations:

- kube-argus now depends on Access being configured for its hostname: a
  dedicated Access application (its own AUD), the IdP integration with group
  passthrough enabled, and the groups claim added to the application token.
  The `groups` claim is not present in the JWT by default.
- `OIDC_ADMIN_GROUP` must be set to the exact identifier the proxy emits in
  the claim — for Entra ID, the group's **object ID (GUID)**, not its
  display name.
- Local/dev keeps using `authMode=none` + `DEFAULT_ROLE`; optionally add an
  `X-Dev-User` header shortcut for local runs.

What we'll need to revisit:

- If kube-argus later needs more than admin/viewer (per-namespace or
  per-group authorization), revisit then — likely by extending the existing
  JIT system with a group check rather than adding a policy database.
- Entra ID (and other IdPs) omit the `groups` claim when a user is in more
  than ~200 groups (overage). Not a concern for a single admin group, but
  keying off broad org-wide groups would need the
  `/cdn-cgi/access/get-identity` fallback.

## Action Items

1. [ ] Add `cmd/server/cfaccess.go`: `initCFAccess()` + `verifyCFAccess(r)` using `oidc.NewRemoteKeySet` / `oidc.NewVerifier`.
2. [ ] Refactor `authCallback`'s admin logic into `roleFromClaims(email, groups)`; reuse in the new path.
3. [ ] Add the `cfaccess` branch to `authMiddleware`; select it in `initAuth` and make it exclusive with `google`/`oidc`.
4. [ ] Config: add `CF_ACCESS_TEAM_DOMAIN`, `CF_ACCESS_AUD` (reuse `OIDC_ADMIN_GROUP`); extend the AWS Secrets Manager `envKeys` list, `.env.example`, and Helm `values.yaml` / `deployment.yaml`.
5. [ ] Tests: `cmd/server/cfaccess_test.go` — table-driven token verification and role mapping.
6. [ ] Cloudflare/IdP: create the Access application for the kube-argus host, enable group passthrough on the IdP integration, add the groups claim to the JWT, record the AUD, and confirm the identifier used by `OIDC_ADMIN_GROUP`.
7. [ ] Validate end-to-end in staging: `groups` claim present in the JWT, admin group → `admin`, default → `viewer`, no double login.
8. [ ] Decide the disposition of the legacy `google`/`oidc` login code (keep for non-Access deployments vs. remove) and record it here once decided.

## References

- kube-argus: `cmd/server/auth.go` (current auth), `cmd/server/main.go` (middleware wiring), `go.mod` (`go-oidc/v3` already present), `.env.example`.
- Cloudflare One — Application token (claims, the subset-of-identity caveat, and the `/cdn-cgi/access/get-identity` endpoint) and the Microsoft Entra ID identity-provider integration (group passthrough).
- Microsoft Entra — configure group claims in tokens (object-ID emission; ~200-group overage limit).
