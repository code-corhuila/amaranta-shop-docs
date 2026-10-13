# Design System

> The design system is the shared visual language between design and development.
> It prevents inconsistencies, accelerates design, and reduces rework.
> **Rule:** Before creating a new component, check here if it already exists.

> **Source of truth:** these tokens are not aspirational — they are transcribed directly from
> `amaranta/apps/web/tailwind.config.ts` and `apps/web/app/globals.css` in the
> implementation repo. If a token changes in code, update it here too.

---

## Design tokens

### Colors

Amaranta's brand palette is "Fiery Ocean": a fiery red (`ember`) as the primary brand color,
paired with a deep-water blue (`ocean`) for secondary/success accents and a warm cream
background instead of pure white. (Superseded the earlier amaranth-pink / sage-green palette.)

```css
/* Primary — ember (brand color, "fire") */
--color-ember-50:  #fdf1ee;
--color-ember-100: #fbe0d8;
--color-ember-200: #f3b8ac;
--color-ember-300: #e8887a;
--color-ember-400: #d85a4a;
--color-ember-500: #c41e2e;  /* Default — primary buttons, links, brand accents */
--color-ember-600: #a01522;  /* Hover state for primary */
--color-ember-700: #7a0e17;  /* Headings, high-emphasis text on light backgrounds */
--color-ember-800: #5c0a10;
--color-ember-900: #3d0709;

/* Secondary — ocean (success, confirmations, secondary accents, "water") */
--color-ocean-50:  #eef5fa;
--color-ocean-100: #dcebf5;
--color-ocean-200: #b7d7ea;
--color-ocean-300: #93c0da;
--color-ocean-400: #79a9c9;
--color-ocean-500: #5c93b8;  /* "Agregar" button, success alerts */
--color-ocean-600: #457897;  /* Hover state */
--color-ocean-700: #345d75;

/* Backgrounds */
--color-cream-50:  #fbeeda;  /* Page background (body) */
--color-cream-100: #f5e2c4;

/* Text */
--color-ink-900: #0e2a3d;  /* Primary text color (dark navy, warm undertone — never pure #000) */

/* Semantic (inherited from Tailwind defaults, used as-is) */
--color-error:   #ef4444;  /* red-500 — form/API error text and borders */
--color-success: var(--color-ocean-500);
```

### Typography

```css
--font-family-display: 'Fredoka', system-ui, sans-serif;  /* Headings, brand name, buttons */
--font-family-sans:    'Inter', system-ui, sans-serif;     /* Body text, form labels, paragraphs */
```

| Level | Element example | Size | Weight | Font |
|-------|-----------------|------|--------|------|
| H1 (hero) | Catalog page hero title | 30–36px (`text-3xl`/`text-4xl`) | 600 | Fredoka |
| H2 | Section titles ("Catálogo", "Confirma tu pedido") | 20–24px (`text-xl`/`text-2xl`) | 600 | Fredoka |
| H3 | Product card name | 16px (`text-base`) | 600 | Fredoka |
| Body | Paragraphs, descriptions | 14–16px (`text-sm`/`text-base`) | 400 | Inter |
| Small / labels | Form labels, helper text | 12–14px (`text-xs`/`text-sm`) | 500 | Inter |

Fredoka (a rounded, friendly display face) is reserved for anything the brand "speaks" —
titles, the logo wordmark, button labels. Inter carries all reading-heavy text — this pairing
is what keeps the pet-shop tone playful without becoming hard to read in paragraphs.

### Spacing

Standard Tailwind 4px scale is used unmodified (`space-1` = 4px ... `space-16` = 64px) —
no custom spacing tokens were needed.

### Borders and shadows

```css
--radius-lg:  8px;      /* rounded-lg — inputs */
--radius-xl2: 20px;     /* rounded-xl2 (custom) — cards, panels, hero banner */
--radius-full: 9999px;  /* rounded-full — buttons, badges, avatar-style bubbles */

--shadow-card: 0 8px 24px -8px rgba(61, 7, 9, 0.2);  /* custom, tinted with the brand color
                                                            instead of pure black — used on every
                                                            card, button, and floating panel */
```

---

## Components

### Buttons

| Variant | Class | Use | Disabled state |
|---------|-------|-----|-----------------|
| Primary | `.btn-primary` — `bg-ember-500`, white text, `rounded-full`, `shadow-card` | Main action per view ("Confirmar pedido", "Crear cuenta", "Continuar") | `opacity-50`, `cursor-not-allowed` |
| Secondary | `.btn-secondary` — `border-ember-300`, ember text, transparent fill | Secondary actions ("Atrás", "Ver mis pedidos") | Same |
| Add-to-cart | `bg-ocean-500` pill, white text | The one action that is deliberately *not* ember — visually separates "browse" (red) from "commit to buying" (blue) | `bg-ink-900/20`, `cursor-not-allowed` (used on out-of-stock products) |

**Usage rules (as implemented):**
- Only one `.btn-primary` per view.
- Every async button (checkout confirm, register, login) shows a text-swap loading state
  ("Procesando…", "Creando cuenta…") instead of a spinner — no spinner component exists yet.

### Forms

| Component | Class | When used |
|-----------|-------|-----------|
| Text input | `.input` — white fill, `rounded-lg`, `border-ink-900/15`, focus ring in `ember-200` | Name, email, phone, address, quantity |
| Radio group | Native `<input type="radio">` styled with `accent-ember-500`, wrapped in a bordered `<label>` that highlights (`border-ember-400 bg-ember-50`) when checked | Payment method (Efectivo / Tarjeta / Transferencia) — 3 options, matches the design-system rule "Radio for 2–5 options" |
| Selectable card | Large bordered button, `hover:border-ember-400` | Fulfillment method (A domicilio / Recoger en tienda) — used instead of radio because there are only 2, mutually exclusive, high-consequence options |

**Error messages in forms:**
- Shown as a full-width alert below the field group (see Feedback → Inline alert), in red,
  stating what to fix — e.g. *"La dirección es obligatoria para entregas a domicilio."*, not
  just "campo inválido".

### Feedback

| Component | Class / implementation | When | Duration |
|-----------|------------------------|------|----------|
| Inline alert | `<Alert kind="error"|"success">` — red or ocean-tinted box | Form/API errors, informational notices ("Inicia sesión para finalizar tu compra") | Until the condition clears |
| Skeleton | Pulsing `bg-ember-100/60` blocks matching the card grid | Loading the product catalog | Until the API responds |
| Fly-to-cart bubble | A small ember circle (🐾) that arcs from the clicked product to the cart icon (Framer Motion) | Every "Agregar" click | ~650ms — the cart badge count only increments when the bubble arrives, not before |
| Cart badge bump | Cart icon scales up (1 → 1.4 → 1) | Fires exactly when the fly-to-cart bubble lands | ~350ms |
| Stepper | Numbered circles + connecting bar, filled ember up to the current step | Checkout wizard (`/cart`) — 4 steps | Persists for the whole flow |
| Interactive hero (3D tilt) | `<InteractiveHero>` — a hand-drawn, 3-layer SVG illustration inside a `perspective` container; each layer translates/rotates by a different fraction of the cursor offset (`onMouseMove`), springing back to center on `onMouseLeave` | Home page (`side="right"`) and the cart/checkout flow (`side="left"`); a `variant="clinic"` exists in the component but has no page yet | Continuous while the cursor is over it; ~300ms ease-out spring back |

### Data / lists

The MVP has no data *table* yet (no admin screens) — the closest pattern is the **product
grid** (2–4 columns depending on viewport, `ProductCard` components) and the **order list**
(`/orders`, one card per order, newest first). Both are simple lists, not paginated tables;
pagination exists only at the API level (`GET /products?page=&pageSize=`, max 50/page).

---

## UX patterns

### Principles actually applied in the MVP

1. **Immediate feedback:** every click gets a same-frame response — button text swap, the
   fly-to-cart bubble, or a disabled state — even before the network request resolves.
2. **Prevent rather than correct:** the delivery address field only appears when "A domicilio"
   is chosen; the payment/contact step won't advance until required fields are filled.
3. **Empty state as a feature:** an empty cart shows an illustration-style emoji + a direct
   CTA ("Ir a la tienda") instead of a blank panel.

### Error handling (as implemented)

| Scenario | What is shown |
|----------|----------------|
| Domain validation error (e.g. insufficient stock, invalid email) | Backend returns `{ code, message }`; the frontend shows `message` in a red inline alert |
| 401 (expired/invalid session) on a protected action | Redirected to `/login` |
| Network/API unreachable | Generic fallback message ("No pudimos procesar tu pedido. Intenta de nuevo.") |

Toasts, 403/404 dedicated screens, and automatic retry are **not** implemented in the MVP —
noted here so it's an explicit scope decision, not an oversight.

---

## Accessibility (as implemented)

| Aspect | Status |
|--------|--------|
| Form labels | Every input has an associated `<label htmlFor>` |
| Keyboard navigation | Native `<button>`/`<a>`/`<input>` elements throughout — no custom widgets that trap focus |
| Alt text | Product images use the product name as `alt` |
| Color contrast | Not formally audited against WCAG AA yet — flagged as a follow-up, not verified |

---

## Correlations

- Navigation map → `12-ux-ui/navigation-map.md`
- Wireframes → `12-ux-ui/wireframes.md`
- Implementation source of these tokens → `amaranta/apps/web/tailwind.config.ts`,
  `amaranta/apps/web/app/globals.css`
- UX non-functional requirements → `04-requirements/non-functional.md`
