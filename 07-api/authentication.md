# Authentication & Authorization

> Reflects the implementation in `apps/api/src/identity/`.

## Mechanism

JWT, signed with a shared secret (HS256, via `@nestjs/jwt`'s default signing
algorithm), issued by the Identity & Access context. There is no OAuth2 provider
and no API Key scheme — a single Bearer JWT covers every protected endpoint.

## Flow

1. `POST /auth/register` (self-service) or `POST /auth/register-walk-in`
   (Receptionist creates the account) — see `auth-service.yaml`
2. `POST /auth/login` with email + password — returns `accessToken`
3. Every protected request sends `Authorization: Bearer <accessToken>`
4. When the token expires, the client must call `POST /auth/login` again — there is
   no refresh-token endpoint in this implementation

## Token contents

The JWT payload carries `sub` (user ID) and `role`. It is verified per-request by
`JwtAuthGuard` (`apps/api/src/identity/infrastructure/security/`).

## Expiration

Configurable via the `JWT_EXPIRES_IN` environment variable — defaults to `3600s`
(1 hour) if unset.

## Roles (RBAC)

Defined in `apps/api/src/identity/domain/value-objects/user-role.enum.ts`, matching
`02-domain/domain-map.md` — Identity & Access:

| Role | Who |
|------|-----|
| `CUSTOMER` | Any registered end user (default role at self-service registration) |
| `RECEPTIONIST` | Front-desk staff — registers walk-in customers, in-store checkout, marks orders paid |
| `VETERINARIAN` | Clinical staff — adds medical records, completes medical/surgery/emergency appointments |
| `GROOMER` | Clinical staff — completes grooming appointments |
| `ADMIN` | Manages the product catalog (`/admin/products/*`) |

## Enforcement

Two guards stack on top of each other:
- `JwtAuthGuard` — validates the token is present and valid; populates the request
  with the authenticated user (`@CurrentUser()`)
- `RolesGuard` + `@Roles(...)` — only applied on endpoints that require a specific
  role beyond "authenticated" (e.g. `@Roles(UserRole.ADMIN)` on catalog admin
  endpoints)

Public endpoints (`security: []` in the OpenAPI contracts) are: `POST /auth/register`,
`POST /auth/login`, and every service's `GET /health`.

`POST /orders` (checkout) is a special case, not a public endpoint in the same sense: it accepts
**both** an anonymous request and an authenticated one (`OptionalJwtAuthGuard`, not
`JwtAuthGuard`). Unlike every other guarded endpoint, an invalid, expired or malformed token here
does **not** produce a 401 — `OptionalJwtAuthGuard.handleRequest` normalizes any failure to
"no user" and lets the request through as anonymous. A missing token and a bad one are
indistinguishable to this endpoint; only a token that verifies is treated as the caller. An
anonymous (or silently-degraded) checkout is attributed to the same walk-in customer used for
in-store sales; an authenticated one is attributed to the caller, so it shows up in
`GET /orders`. See `07-api/sales-api-decisions.md` (D-S1, G-5) for why, and for the customer-facing
risk that creates.
