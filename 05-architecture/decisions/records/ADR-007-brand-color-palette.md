# ADR-007 — Brand Color Palette: Fiery Ocean

- **ID:** ADR-007
- **Date:** 2026-09-20
- **Status:** Accepted
- **Authors:** Francy Claros — Tech Lead / Full-stack developer

---

## Context

Amaranta is not a single-purpose retail shop: it is a pet products store *and* a veterinary
clinic operating as one system (`02-domain/domain-map.md`). The original palette
(`12-ux-ui/design-system.md`, pre-ADR) used a vivid amaranth pink as the single primary color
across every screen, with sage green reserved for success states. That choice was tied directly
to the shop's name (the amaranth flower), but it was defined before the Scheduling and Clinical
Records contexts were fully scoped as **Core** subdomains (`02-domain/domain-map.md` §4) —
screens that carry medical history and appointment data, not just product browsing.

A single warm, playful pink applied uniformly to both the checkout flow and the clinical-records
screens under-signals the trust and professionalism the clinic side needs, and over-indexes on
"boutique retail" as the whole brand identity.

**Known constraints:**
- Zero design budget — the palette must stay a token-level change (CSS custom properties only),
  not a full component redesign
- One developer — the swap must not require restructuring `design-system.md`'s component rules,
  only its color tokens
- Must still produce enough contrast for a distinct "browse" vs. "commit to buying" action pair,
  a rule already established in `design-system.md` (Add-to-cart deliberately not the primary color)

---

## Decision

**We decided:** Replace the amaranth-pink / sage-green palette with **"Fiery Ocean"** — a warm
`ember` red-orange as the primary brand color, paired with a cooler `ocean` blue as the
secondary/success color — keeping the existing cream background and near-black `ink` text color
unchanged.

**Justification:** Amaranta's product spans two emotional registers that the old palette
collapsed into one. `ember` carries the warmth and urgency appropriate to retail actions —
"Confirmar pedido," "Agregar," the fly-to-cart animation — while `ocean` carries the calm,
trustworthy register appropriate to confirmations, success states, and — going forward — the
clinical and scheduling screens where a customer is trusting the system with their pet's health
data. This is a stronger semantic mapping to Amaranta's actual two-sided business than a single
pink hue stretched across both, and it costs nothing structurally: every component rule in
`design-system.md` (`.btn-primary`, `.btn-secondary`, the "Add-to-cart is deliberately not the
primary color" rule) stays exactly as designed, only the underlying hex values change.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Ember / Ocean ("Fiery Ocean") (CHOSEN) | Warm primary for retail conversion + cool secondary for trust/clinical screens; zero structural change to `design-system.md` | Breaks the literal visual tie between the brand color and the amaranth flower in the shop's name | — (chosen) |
| Keep amaranth-pink / sage-green | Direct, literal tie to the shop's name; zero migration cost, nothing to change | One hue for both "browse the shop" and "trust the clinic with your pet's medical history" under-serves the clinical Core subdomain; reads as boutique-only | Business is two-sided (shop + clinic); the palette should reflect that |
| Single neutral/blue-only palette (fully clinical) | Strong professional/trust read across the whole app | Removes all retail warmth and urgency; the existing conversion-focused UI (fly-to-cart bubble, stepper, "Add-to-cart" accent rule) depends on a warm accent color to work as designed | Would require redesigning interactions already documented in `design-system.md`, not just swapping tokens |

---

## Consequences

**Positive:**
- The palette now maps to Amaranta's two subdomains (retail energy vs. clinical trust) instead of one hue for both
- Pure token-level change — every button/input/component rule in `design-system.md` keeps working unmodified
- Still satisfies the existing "Add-to-cart is deliberately not the primary color" contrast rule

**Negative / Trade-offs:**
- Loses the literal color tie-in to the amaranth flower referenced in the shop's name (the name itself is unaffected, only the color)
- Any frontend code written against the old `--color-amaranth-*` / `--color-sage-*` tokens will need to be updated to `--color-ember-*` / `--color-ocean-*` once implementation exists

**Impact on the system:**
- Affected documents: `12-ux-ui/design-system.md` (already updated)
- Affected services: none yet (no frontend code exists at the time of this ADR) — applies the moment `12-ux-ui/wireframes.md` or any frontend implementation references color tokens

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Stakeholders (course evaluators, future users) expect the amaranth-pink identity because of the shop's name | Low | Low | The shop's name and wordmark are unaffected; only the UI accent colors changed. Revisit if user testing shows a negative brand association |

---

## References

- Domain classification (Core vs. Supporting vs. Generic subdomains) → `02-domain/domain-map.md` §4
- Design system component rules (unaffected by this change) → `12-ux-ui/design-system.md`
- Related to: none (first UX/UI-scoped ADR in this project)
