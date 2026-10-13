# Wireframes

> Low-fidelity structure of the 5 screens that exist in the monolith MVP. These are drawn from
> the actual running application (`amaranta/apps/web`), not from a pre-implementation
> mockup — the MVP was built first; this documents its real structure, per
> `12-ux-ui/README.md`'s instruction to focus on structure, not colors (see
> `12-ux-ui/design-system.md` for the actual visual tokens).

---

## 1. Catalog (`/`) — HU-CAT-001

```
┌──────────────────────────────────────────────────────────────────────┐
│ 🌸 Amaranta        Tienda   [🛒 Carrito (n)]  Ingresar   Crear cuenta │  ← sticky navbar
├──────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────┐   │
│ │  AMARANTA SHOP                                                  │   │
│ │  Todo lo que tu mascota necesita, en un solo lugar               │   │  ← hero banner
│ │  Alimento, accesorios, higiene y más — retiro en tienda.          │   │
│ └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  Catálogo                                                             │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐              │
│  │ [category]│ │ [category]│ │ [category]│ │ [category]│              │
│  │  [photo]  │ │  [photo]  │ │  [photo]  │ │  [photo]  │              │
│  │  Name     │ │  Name     │ │  Name     │ │  Name     │              │
│  │  desc...  │ │  desc...  │ │  desc...  │ │  desc...  │              │
│  │  $price   │ │  $price   │ │  $price   │ │  $price   │              │
│  │  [Agregar]│ │  [Agregar]│ │  [Agregar]│ │  [Agotado]│  ← disabled  │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘              │
│  (grid continues, up to 50 products/page)                             │
└──────────────────────────────────────────────────────────────────────┘
```

Clicking **Agregar** does not navigate away — a small bubble animates from the card to the
cart icon, and the cart badge number updates only once it arrives.

---

## 2. Login (`/login`)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 🌸 Amaranta                                    Tienda   Carrito       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│              Ingresa a tu cuenta                                      │
│              ¿Aún no tienes cuenta? Regístrate                        │
│                                                                        │
│              Correo electrónico                                       │
│              [_____________________________]                          │
│                                                                        │
│              Contraseña                                               │
│              [_____________________________]                          │
│                                                                        │
│              (error alert appears here if invalid)                    │
│                                                                        │
│              [        Ingresar        ]                               │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Register (`/register`) — HU-IAM-001

```
┌──────────────────────────────────────────────────────────────────────┐
│ 🌸 Amaranta                                    Tienda   Carrito       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│              Crea tu cuenta                                           │
│              Regístrate para comprar y agendar citas.                 │
│                                                                        │
│              Nombre completo      [_______________________]           │
│              Correo electrónico   [_______________________]           │
│              Teléfono (opcional)  [_______________________]           │
│              Contraseña           [_______________________]           │
│                Mínimo 8 caracteres                                     │
│                                                                        │
│              (error alert: e.g. "ya existe una cuenta con ese correo")│
│                                                                        │
│              [       Crear cuenta       ]                             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Cart & checkout wizard (`/cart`) — HU-SALES-001

Four steps behind one stepper header. Step 1 requires nothing; steps 2–4 require login
(redirects to `/login` if there is no session).

```
┌──────────────────────────────────────────────────────────────────────┐
│  ①───②───③───④                                                        │
│ Carrito Entrega Datos Confirmar        ← stepper, filled up to current│
├──────────────────────────────────────────────────────────────────────┤
│ STEP 1 — Cart                          STEP 2 — Fulfillment            │
│ [img] Product name        qty [2]      ┌─────────────┐ ┌─────────────┐│
│       $unit c/u          $subtotal ✕   │  🚚          │ │  🏪          ││
│ [img] Product name        qty [1]      │  A domicilio │ │  Recoger en ││
│       $unit c/u          $subtotal ✕   │  Lo llevamos │ │  tienda     ││
│                                         │  a tu casa   │ │  Sin costo  ││
│ Subtotal            $XX.XXX            └─────────────┘ └─────────────┘│
│ [        Continuar        ]            ← Volver al carrito             │
├──────────────────────────────────────────────────────────────────────┤
│ STEP 3 — Details                       STEP 4 — Review                 │
│ Nombre completo   [____________]       1 × Product name    $subtotal  │
│ Celular           [____________]       Total (a domicilio) $XX.XXX    │
│ Dirección*        [____________]       Entrega: A domicilio — <addr>  │
│  (*solo si es a domicilio)             Pago: Efectivo                 │
│ Método de pago                         Contacto: Nombre · Celular     │
│  ( ) Efectivo  ( ) Tarjeta  ( ) Transf.                                │
│ [Atrás]        [Continuar]             [Atrás]   [Confirmar pedido]   │
└──────────────────────────────────────────────────────────────────────┘
```

**Success state** (replaces the wizard after confirmation):

```
┌──────────────────────────────────────────────────────────────────────┐
│                          🎉                                            │
│                 ¡Pedido confirmado!                                    │
│     Tu pedido #xxxxxxxx quedó registrado por $XX.XXX —                │
│     para recoger en tienda / a domicilio en <dirección>.               │
│                                                                        │
│         [   Enviar pedido por WhatsApp   ]  ← pre-filled wa.me link   │
│                                                                        │
│  El envío automático de la factura por correo y WhatsApp está         │
│  pendiente — este entorno es local.                                    │
│                                                                        │
│         [ Ver mis pedidos ]     [ Seguir comprando ]                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. My orders (`/orders`)

```
┌──────────────────────────────────────────────────────────────────────┐
│ 🌸 Amaranta        Tienda   Carrito   Mis pedidos     Hola, Ana  Salir│
├──────────────────────────────────────────────────────────────────────┤
│  Mis pedidos                                                          │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ #xxxxxxxx                              [Pendiente de pago]      │  │
│  │ 1 × Product name                                    $subtotal   │  │
│  │ Total                                                $XX.XXX    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ #xxxxxxxx                              [Pagado]                 │  │
│  │ 2 × Product name                                    $subtotal   │  │
│  │ Total                                                $XX.XXX    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│  (newest first — empty state: illustration + "Ir a la tienda")       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Correlations

- Navigation between these screens → `12-ux-ui/navigation-map.md`
- Visual tokens (colors, type, components referenced above) → `12-ux-ui/design-system.md`
- User stories these screens implement → `04-requirements/user-stories.md`
