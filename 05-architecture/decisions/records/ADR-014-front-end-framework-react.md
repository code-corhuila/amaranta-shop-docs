# ADR-014 — Front-End Framework: React 19 with Module Federation

- **ID:** ADR-014
- **Date:** 2026-10-14
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer

---

## Context

The course norm splits the web interface into a **shell** (`amaranta-shop-front`) and **one portal per
domain** (`amaranta-shop-catalog-portal`, `amaranta-shop-sales-portal`), loaded into the shell at run time
(norm 5.4 and 5.5, Annex H). The team must choose the framework **between React 19 and Angular 21** and
record the choice in an ADR; until now `amaranta-shop-front`'s README said "Angular" only because that was the
template's default, and the portals said "pending its own ADR (norm 4.2.2)".

Both frameworks must obey the rule the whole split rests on: **the HTTP client and the session live in the
shell, and only there**. A portal with a client of its own multiplies the part of the interface that is hardest
to get right (the token), and its requests leave with no token and no correlation id, in silence.

**Known constraints:**
- One developer, course-graded delivery
- The monolith's web (`apps/web` of `BrandowC/amaranta`) is already React 18 with TypeScript and Tailwind, so the
  pages that will move into the portals exist in React
- The services behind the first portals are already built and tested (the gateway, `catalog-api` and `sales-api`):
  they answer `{data, meta}` lists and one common error envelope, so the interface has a stable contract to build on
- Everything must build and test in containers (Annex H, rule 6: `npm ci` from a versioned lock file)

---

## Decision

**We decided:** the shell and every portal are written in **React 19 with Vite, strict TypeScript and Vitest**,
composed with **Module Federation (`@module-federation/vite`)** on Node 22.

1. **The shell is the Module Federation host.** It exposes its HTTP client as `shell/apiClient`; each portal is
   a remote that exposes `./App` and borrows the client. The gateway's address is configuration of the shell and
   is written once.
2. **`shareStrategy: 'loaded-first'`.** A portal is downloaded when its route is opened. With the default
   (`version-first`) the shell downloads every remote at start to compare versions, and one portal that is down
   leaves the whole application blank.
3. **One error boundary per portal**, with a retry, so a portal that is down or breaks while drawing makes only
   its own area say so.
4. **The rule of the shell is enforced by tests, not by the framework** (see Consequences): a portal's source is
   read by a test that fails on its own `fetch`, a token, browser storage or the gateway's address, and the build
   greps the compiled code for the same.

**Justification:** the tiebreaker is **what the team already has**. The monolith's pages, its Tailwind styling and
the habits of the developer are React, and the two frameworks fit the same Annex H rules, so choosing Angular would
spend the delivery learning a second toolchain (Native Federation, the injector, zone-less change detection) to
reach the same result. The cost React has — it does not enforce the rule of the shell the way Angular's injector
does — is paid with two cheap, automatic checks.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| React 19 + Vite + Module Federation (CHOSEN) | Same language and library as the monolith's web; Annex H describes this path file by file; fast tests with Vitest | Does not make a second HTTP client impossible; Module Federation for Vite is a young plugin | — (chosen) |
| Angular 21 + Native Federation | The injector makes the rule structural: a portal mounted inside the shell's injector receives the shell's `HttpClient` and its interceptor | A second toolchain and a different mental model for one developer; nothing of the monolith's web carries over | The rule is cheaper to check than to learn Angular for |
| Next.js (as the monolith's web) | Already known | Server rendering and file routing do not fit a shell that loads remotes at run time; not one of the two frameworks the norm allows | Outside the norm, and the wrong shape |
| One single-page application, no portals | Simplest | Breaks norm 5.4 and 5.5: no domain owns its interface and the shell is not separable | Not allowed |

---

## Consequences

**Positive:**
- The monolith's React code and Tailwind knowledge move across
- Each domain's interface is deployed and failed on its own: a portal that is down does not take the application down
- Tests run in milliseconds with no browser; the production image is static files behind nginx

**Negative / Trade-offs:**
- **React does not enforce the rule.** Anyone can call `fetch` in a portal. It is mitigated by `src/architecture.test.ts`
  (fails on `fetch`, `XMLHttpRequest`, an HTTP library, `Authorization`/`Bearer`, storage, the gateway's address and
  `/internal` routes) and by a `grep` of the compiled code in the Dockerfile; both were shown to fail on a portal that
  cheats. They protect the code that exists; a rule nobody wrote a check for is not protected
- The tooling is new (Vite 8, TypeScript 7, Vitest 5, `@module-federation/vite` 1.23): versions are pinned exactly in
  `package.json` and `package-lock.json`, and an upgrade is its own pull request
- The types of `shell/apiClient` are declared by hand in each portal (`shell.d.ts`), because the shell's code is not in
  the portal's repository; a change to the client must change them in step

**Impact on the system:**
- Affected repositories: `amaranta-shop-front`, `amaranta-shop-catalog-portal`, `amaranta-shop-sales-portal`
- The identity portal that replaces the development-only sign-in (Annex H, rule 5) will also be React
- Documents that must be updated: the README of `amaranta-shop-front` (it said Angular) and of each portal; the
  register in `05-architecture/decisions/README.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A portal calls the API without the shell's client and requests leave without a token | Medium | High | The architecture test and the compiled-code check fail the build |
| One portal that is down takes the application down | Low | High | `loaded-first` plus a boundary per portal, tested and shown in a browser with a portal switched off |
| The Module Federation plugin breaks on an upgrade | Low | Medium | Exact versions and a lock file; upgrades are separate pull requests |

---

## References

- Course norm, numerals 5.4 and 5.5 and Annex H (`H-front.md`)
- [ADR-007](./ADR-007-brand-color-palette.md) — the palette the interface will use
- [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md) and [ADR-011](./ADR-011-identity-design-with-revocation.md) — the token the shell reads and never verifies
- [ADR-012](./ADR-012-stack-of-extracted-services.md) — the services behind the portals
- [ADR-013](./ADR-013-monolith-first-strangler-fig.md) — why the monolith's pages move gradually
