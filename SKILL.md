# payments

## Propósito

Integrar pagos en el sitio generado. Cubre cuatro opciones ordenadas por facilidad de implementación: Lemon Squeezy para productos digitales, Mercado Pago para negocios en LATAM, Stripe para negocios establecidos que necesitan control total, y PayPal para audiencias internacionales o clientes sin tarjeta local. Para cada opción: cuándo recomendarla, cómo obtener credenciales, y el componente de botón listo para Next.js + Tailwind.

---

## Cuándo activarse

- El usuario responde afirmativamente a la pregunta de pagos en la Ronda 4 del cuestionario.
- El usuario menciona "cobrar online", "vender", "cursos", "productos digitales", "reservas pagas", "precio" o "tienda".
- El CTA principal del negocio incluye un precio o una compra.
- El negocio vende servicios, cursos, templates, ebooks, o cualquier producto digital.

---

## Decisión rápida: qué recomendar

```
¿Vende productos digitales (cursos, ebooks, templates, membresías)?
  └─ Sí → Lemon Squeezy (más simple, sin código de backend)

¿Opera en Paraguay, Ecuador, Argentina, México, Colombia, Chile o Brasil?
  └─ Sí → Mercado Pago (mayor penetración, más métodos de pago locales)

¿Necesita pagos recurrentes, marketplace, o control total del checkout?
  └─ Sí → Stripe

¿Vende a audiencia internacional o en países donde Mercado Pago no opera?
  └─ Sí → PayPal (reconocimiento global, acepta tarjetas sin cuenta PayPal)
```

---

## OPCIÓN 1 — Lemon Squeezy

### Cuándo recomendarla

- Productos digitales: cursos, ebooks, templates, plugins, software, membresías
- El usuario no quiere configurar un backend complejo
- No tiene empresa constituida — Lemon Squeezy actúa como Merchant of Record (se encarga de impuestos)
- Quiere empezar a cobrar en menos de una hora

### Obtener credenciales

1. Crear cuenta en **https://lemonsqueezy.com** (plan gratuito disponible)
2. Ir a Settings → API → crear una API Key
3. Crear una tienda: Settings → Stores → Add Store
4. Agregar el producto: Products → Add Product → configurar nombre, precio, tipo (one-time o recurring)
5. En el producto, copiar el **Checkout URL** — es el link directo al checkout de Lemon Squeezy

No se necesita API key para la integración más simple — solo el Checkout URL.

### Integración simple: link directo (sin backend)

La forma más rápida — el botón abre el checkout de Lemon Squeezy en una overlay:

```tsx
// components/LemonButton.tsx
'use client'

import Script from 'next/script'

interface LemonButtonProps {
  checkoutUrl: string   // URL del checkout de Lemon Squeezy
  label?: string
  variant?: 'primary' | 'secondary'
}

export function LemonButton({
  checkoutUrl,
  label = 'Comprar ahora',
  variant = 'primary',
}: LemonButtonProps) {
  const variantStyles = {
    primary:   'bg-primary text-white hover:bg-primary/90 px-8 py-4 text-base font-medium rounded',
    secondary: 'border border-primary text-primary hover:bg-primary/10 px-8 py-4 text-base font-medium rounded',
  }

  return (
    <>
      {/* Script de Lemon Squeezy para overlay del checkout */}
      <Script
        src="https://assets.lemonsqueezy.com/lemon.js"
        strategy="lazyOnload"
        onLoad={() => {
          if (typeof window !== 'undefined' && (window as any).createLemonSqueezy) {
            (window as any).createLemonSqueezy()
          }
        }}
      />
      <a
        href={checkoutUrl}
        className={`lemonsqueezy-button inline-flex items-center justify-center transition-colors ${variantStyles[variant]}`}
        aria-label={label}
      >
        {label}
      </a>
    </>
  )
}
```

Uso:
```tsx
<LemonButton
  checkoutUrl="https://tutienda.lemonsqueezy.com/checkout/buy/[product-id]"
  label="Comprar el curso — $97 USD"
  variant="primary"
/>
```

### Integración con API (para precios dinámicos o descuentos)

```bash
npm install @lemonsqueezy/lemonsqueezy-js
```

```ts
// app/api/checkout/route.ts — Server Action para crear checkout dinámico
import { lemonSqueezySetup, createCheckout } from '@lemonsqueezy/lemonsqueezy-js'

lemonSqueezySetup({ apiKey: process.env.LEMON_SQUEEZY_API_KEY! })

export async function POST(req: Request) {
  const { variantId, email } = await req.json()

  const checkout = await createCheckout(
    process.env.LEMON_SQUEEZY_STORE_ID!,
    variantId,
    {
      checkoutData: { email },
      checkoutOptions: { embed: true },
    }
  )

  return Response.json({ url: checkout.data?.data.attributes.url })
}
```

Variables de entorno en `.env.local`:
```
LEMON_SQUEEZY_API_KEY=tu_api_key
LEMON_SQUEEZY_STORE_ID=tu_store_id
```

---

## OPCIÓN 2 — Mercado Pago

### Cuándo recomendarla

- El negocio opera en Argentina, México, Colombia, Chile, Uruguay, Paraguay, Ecuador, Brasil o Perú
- El cliente final paga en moneda local (pesos argentinos, pesos mexicanos, soles, etc.)
- Se necesita aceptar tarjetas locales, transferencias, Rapipago, PagoFácil, OXXO, o cuotas sin interés
- El dueño ya tiene cuenta de Mercado Pago o Mercado Libre

### Obtener credenciales

1. Crear cuenta en **https://mercadopago.com** (o usar la cuenta existente)
2. Ir a **https://www.mercadopago.com.ar/developers/panel** (o el país correspondiente)
3. Crear una aplicación: "Crear aplicación" → nombre → tipo Web → guardar
4. En la aplicación, copiar:
   - **Public Key** (para el frontend)
   - **Access Token** (para el backend — nunca exponer en el cliente)
5. Para pruebas: usar las credenciales de "Credenciales de prueba" (no afectan dinero real)

### Integración: Preference + Redirect

El flujo estándar de Mercado Pago:
1. El servidor crea una "preferencia" con los items y el precio
2. Mercado Pago devuelve un `init_point` (URL del checkout)
3. El usuario es redirigido al checkout de Mercado Pago
4. Mercado Pago redirige al usuario a `success_url` / `failure_url` / `pending_url`

```bash
npm install mercadopago
```

```ts
// app/api/mp-checkout/route.ts
import MercadoPagoConfig, { Preference } from 'mercadopago'

const client = new MercadoPagoConfig({
  accessToken: process.env.MP_ACCESS_TOKEN!,
})

export async function POST(req: Request) {
  const { title, price, quantity } = await req.json()

  const preference = new Preference(client)

  const result = await preference.create({
    body: {
      items: [
        {
          id: '1',
          title,
          quantity,
          unit_price: price,
          currency_id: 'ARS', // ARS | MXN | CLP | COP | UYU | BRL | PEN
        },
      ],
      back_urls: {
        success: `${process.env.NEXT_PUBLIC_SITE_URL}/gracias`,
        failure: `${process.env.NEXT_PUBLIC_SITE_URL}/error-pago`,
        pending: `${process.env.NEXT_PUBLIC_SITE_URL}/pago-pendiente`,
      },
      auto_return: 'approved',
    },
  })

  return Response.json({ checkoutUrl: result.init_point })
}
```

```tsx
// components/MercadoPagoButton.tsx
'use client'

import { useState } from 'react'

interface MPButtonProps {
  title: string
  price: number
  quantity?: number
  label?: string
  variant?: 'primary' | 'secondary'
}

export function MercadoPagoButton({
  title,
  price,
  quantity = 1,
  label = 'Pagar con Mercado Pago',
  variant = 'primary',
}: MPButtonProps) {
  const [loading, setLoading] = useState(false)

  async function handleClick() {
    setLoading(true)
    try {
      const res = await fetch('/api/mp-checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, price, quantity }),
      })
      const { checkoutUrl } = await res.json()
      window.location.href = checkoutUrl
    } catch {
      setLoading(false)
    }
  }

  const variantStyles = {
    primary:   'bg-[#009EE3] text-white hover:bg-[#0080BA] px-8 py-4 text-base font-medium rounded',
    secondary: 'border border-[#009EE3] text-[#009EE3] hover:bg-[#009EE3]/10 px-8 py-4 text-base font-medium rounded',
  }

  return (
    <button
      type="button"
      onClick={handleClick}
      disabled={loading}
      aria-busy={loading}
      className={`inline-flex items-center justify-center gap-2 transition-colors disabled:opacity-60 ${variantStyles[variant]}`}
    >
      {loading ? 'Redirigiendo...' : label}
    </button>
  )
}
```

Variables de entorno en `.env.local`:
```
MP_ACCESS_TOKEN=tu_access_token_de_mercado_pago
NEXT_PUBLIC_SITE_URL=https://tusitio.com
```

### Webhook para confirmar pagos (opcional pero recomendado)

```ts
// app/api/mp-webhook/route.ts
export async function POST(req: Request) {
  const body = await req.json()

  if (body.type === 'payment') {
    const paymentId = body.data.id
    // Consultar el estado del pago con la API de MP
    // Activar el acceso al producto/servicio
    console.log('Pago recibido:', paymentId)
  }

  return new Response('OK', { status: 200 })
}
```

---

## OPCIÓN 3 — Stripe

### Cuándo recomendarla

- El negocio opera en mercados donde Mercado Pago no está disponible (Europa, EE.UU., Canadá)
- Se necesita pagos recurrentes (suscripciones) con control total
- El negocio tiene empresa constituida y cuenta bancaria internacional
- Se necesita un marketplace o split de pagos entre vendedores

### Obtener credenciales

1. Crear cuenta en **https://stripe.com**
2. En el Dashboard → Developers → API Keys
3. Copiar **Publishable Key** (frontend) y **Secret Key** (backend)
4. Para pruebas: usar las keys del modo "Test" (empiezan con `pk_test_` y `sk_test_`)

### Integración: Stripe Checkout (más simple)

```bash
npm install stripe @stripe/stripe-js
```

```ts
// app/api/stripe-checkout/route.ts
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-06-20',
})

export async function POST(req: Request) {
  const { priceId } = await req.json()

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',              // o 'subscription' para pagos recurrentes
    payment_method_types: ['card'],
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/gracias?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/#precios`,
  })

  return Response.json({ url: session.url })
}
```

```tsx
// components/StripeButton.tsx
'use client'

import { useState } from 'react'

interface StripeButtonProps {
  priceId: string       // ID del precio en Stripe (ej: price_1234abc)
  label?: string
  variant?: 'primary' | 'secondary'
}

export function StripeButton({
  priceId,
  label = 'Comprar ahora',
  variant = 'primary',
}: StripeButtonProps) {
  const [loading, setLoading] = useState(false)

  async function handleClick() {
    setLoading(true)
    try {
      const res = await fetch('/api/stripe-checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ priceId }),
      })
      const { url } = await res.json()
      window.location.href = url
    } catch {
      setLoading(false)
    }
  }

  const variantStyles = {
    primary:   'bg-primary text-white hover:bg-primary/90 px-8 py-4 text-base font-medium rounded',
    secondary: 'border border-primary text-primary hover:bg-primary/10 px-8 py-4 text-base font-medium rounded',
  }

  return (
    <button
      type="button"
      onClick={handleClick}
      disabled={loading}
      aria-busy={loading}
      className={`inline-flex items-center justify-center transition-colors disabled:opacity-60 ${variantStyles[variant]}`}
    >
      {loading ? 'Redirigiendo...' : label}
    </button>
  )
}
```

Variables de entorno en `.env.local`:
```
STRIPE_SECRET_KEY=sk_live_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
NEXT_PUBLIC_SITE_URL=https://tusitio.com
```

---

## OPCIÓN 4 — PayPal

### Cuándo recomendarla

- El negocio vende a clientes internacionales fuera de LATAM (EE.UU., Europa, Asia)
- El cliente objetivo puede no tener tarjeta de crédito local pero sí cuenta PayPal
- El negocio opera en países donde Mercado Pago no está disponible (Venezuela, Bolivia, Honduras, Guatemala, Nicaragua, Costa Rica, Panamá)
- Se quiere ofrecer una segunda opción de pago junto a Stripe o Lemon Squeezy para aumentar la tasa de conversión
- El producto se vende en USD y la audiencia confía más en PayPal que en procesadores locales

### Obtener el Client ID

1. Crear cuenta en **https://developer.paypal.com** (usar la cuenta de PayPal existente o crear una)
2. Ir a "My Apps & Credentials"
3. En "REST API apps" → "Create App"
4. Nombre de la app: nombre del negocio → Create App
5. Copiar el **Client ID** (para el frontend) y el **Secret** (para el backend)
6. Para pruebas: usar las credenciales del modo "Sandbox" — PayPal provee cuentas de comprador y vendedor de prueba

### Integración con `@paypal/react-paypal-js`

```bash
npm install @paypal/react-paypal-js
```

**Provider global** — agregar en `app/layout.tsx`:

```tsx
// app/layout.tsx
import { PayPalScriptProvider } from '@paypal/react-paypal-js'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="es">
      <body>
        <PayPalScriptProvider
          options={{
            clientId: process.env.NEXT_PUBLIC_PAYPAL_CLIENT_ID!,
            currency: 'USD',
            intent: 'capture',
          }}
        >
          {children}
        </PayPalScriptProvider>
      </body>
    </html>
  )
}
```

**Componente PayPalButton:**

```tsx
// components/PayPalButton.tsx
'use client'

import { PayPalButtons, usePayPalScriptReducer } from '@paypal/react-paypal-js'

interface PayPalButtonProps {
  amount: string        // monto en USD como string, ej: "97.00"
  description: string   // descripción del producto
  onSuccess?: (orderId: string) => void
  onError?: (error: unknown) => void
}

export function PayPalButton({
  amount,
  description,
  onSuccess,
  onError,
}: PayPalButtonProps) {
  const [{ isPending }] = usePayPalScriptReducer()

  if (isPending) {
    return (
      <div className="w-full h-12 bg-foreground/5 rounded animate-pulse" aria-label="Cargando opciones de pago..." />
    )
  }

  return (
    <PayPalButtons
      style={{
        layout: 'vertical',
        color: 'gold',
        shape: 'rect',
        label: 'pay',
        height: 48,
      }}
      createOrder={async () => {
        const res = await fetch('/api/paypal-order', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ amount, description }),
        })
        const { id } = await res.json()
        return id
      }}
      onApprove={async (data) => {
        const res = await fetch('/api/paypal-capture', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ orderId: data.orderID }),
        })
        const capture = await res.json()
        if (capture.status === 'COMPLETED') {
          onSuccess?.(data.orderID)
        }
      }}
      onError={(err) => {
        console.error('PayPal error:', err)
        onError?.(err)
      }}
    />
  )
}
```

**API routes del servidor:**

```ts
// app/api/paypal-order/route.ts — crear orden
export async function POST(req: Request) {
  const { amount, description } = await req.json()

  const auth = Buffer.from(
    `${process.env.PAYPAL_CLIENT_ID}:${process.env.PAYPAL_SECRET}`
  ).toString('base64')

  // Obtener access token
  const tokenRes = await fetch('https://api-m.paypal.com/v1/oauth2/token', {
    method: 'POST',
    headers: {
      Authorization: `Basic ${auth}`,
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: 'grant_type=client_credentials',
  })
  const { access_token } = await tokenRes.json()

  // Crear orden
  const orderRes = await fetch('https://api-m.paypal.com/v2/checkout/orders', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      intent: 'CAPTURE',
      purchase_units: [
        {
          description,
          amount: { currency_code: 'USD', value: amount },
        },
      ],
    }),
  })
  const order = await orderRes.json()

  return Response.json({ id: order.id })
}
```

```ts
// app/api/paypal-capture/route.ts — capturar el pago aprobado
export async function POST(req: Request) {
  const { orderId } = await req.json()

  const auth = Buffer.from(
    `${process.env.PAYPAL_CLIENT_ID}:${process.env.PAYPAL_SECRET}`
  ).toString('base64')

  const tokenRes = await fetch('https://api-m.paypal.com/v1/oauth2/token', {
    method: 'POST',
    headers: {
      Authorization: `Basic ${auth}`,
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: 'grant_type=client_credentials',
  })
  const { access_token } = await tokenRes.json()

  const captureRes = await fetch(
    `https://api-m.paypal.com/v2/checkout/orders/${orderId}/capture`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${access_token}`,
        'Content-Type': 'application/json',
      },
    }
  )
  const capture = await captureRes.json()

  return Response.json(capture)
}
```

Variables de entorno en `.env.local`:
```
NEXT_PUBLIC_PAYPAL_CLIENT_ID=tu_client_id_paypal
PAYPAL_CLIENT_ID=tu_client_id_paypal
PAYPAL_SECRET=tu_secret_paypal
# Para sandbox (pruebas): usar las credenciales del modo Sandbox
# Para producción: cambiar https://api-m.paypal.com por https://api-m.paypal.com (ya es producción)
# Para sandbox: usar https://api-m.sandbox.paypal.com
```

### Uso en el sitio

```tsx
// En una página de producto o en la sección de precios
import { PayPalButton } from '@/components/PayPalButton'

<div className="max-w-sm mx-auto">
  <PayPalButton
    amount="97.00"
    description="Curso de diseño web sin programar"
    onSuccess={(orderId) => {
      // Redirigir a página de gracias o activar acceso al producto
      window.location.href = `/gracias?order=${orderId}`
    }}
    onError={() => {
      alert('Hubo un error con el pago. Intentá de nuevo o contactanos.')
    }}
  />
</div>
```

### PayPal + otra opción en paralelo

Para maximizar conversión, ofrecer PayPal como segunda opción junto al método principal:

```tsx
<div className="space-y-4 max-w-sm mx-auto">
  {/* Opción principal */}
  <LemonButton
    checkoutUrl="https://..."
    label="Pagar con tarjeta — $97 USD"
    variant="primary"
  />

  <div className="flex items-center gap-3 text-foreground/30 text-xs">
    <div className="h-px flex-1 bg-foreground/10" />
    o pagá con
    <div className="h-px flex-1 bg-foreground/10" />
  </div>

  {/* PayPal como alternativa */}
  <PayPalButton amount="97.00" description="Curso de diseño web" />
</div>
```

---

## Sección de precios en el sitio

Independientemente del procesador elegido, la sección de precios sigue este patrón:

```tsx
// components/Pricing.tsx
import { LemonButton } from '@/components/LemonButton'        // o MercadoPagoButton / StripeButton

interface PricingPlan {
  name: string
  price: string
  period?: string          // '/mes', '/año', 'pago único'
  description: string
  features: string[]
  checkoutUrl: string      // o priceId para Stripe
  featured?: boolean
}

export function Pricing({ plans }: { plans: PricingPlan[] }) {
  return (
    <section id="precios" className="py-20 px-6">
      <div className="max-w-5xl mx-auto">
        <h2 className="text-3xl font-bold text-foreground text-center mb-12">
          Precios
        </h2>
        <div className={`grid gap-8 ${plans.length === 2 ? 'md:grid-cols-2' : 'md:grid-cols-3'}`}>
          {plans.map(plan => (
            <div
              key={plan.name}
              className={`p-8 rounded-lg border ${
                plan.featured
                  ? 'border-primary shadow-lg shadow-primary/10'
                  : 'border-foreground/10'
              }`}
            >
              {plan.featured && (
                <span className="text-xs font-semibold text-primary uppercase tracking-widest">
                  Más popular
                </span>
              )}
              <h3 className="text-xl font-semibold text-foreground mt-2">{plan.name}</h3>
              <div className="mt-4 flex items-baseline gap-1">
                <span className="text-4xl font-bold text-foreground">{plan.price}</span>
                {plan.period && (
                  <span className="text-foreground/50 text-sm">{plan.period}</span>
                )}
              </div>
              <p className="text-foreground/60 text-sm mt-2">{plan.description}</p>
              <ul className="mt-6 space-y-3">
                {plan.features.map(f => (
                  <li key={f} className="flex items-start gap-2 text-sm text-foreground/80">
                    <span className="text-primary mt-0.5" aria-hidden="true">✓</span>
                    {f}
                  </li>
                ))}
              </ul>
              <div className="mt-8">
                <LemonButton
                  checkoutUrl={plan.checkoutUrl}
                  label={`Empezar con ${plan.name}`}
                  variant={plan.featured ? 'primary' : 'secondary'}
                />
              </div>
            </div>
          ))}
        </div>
      </div>
    </section>
  )
}
```

---

## Cómo el agente maneja la decisión de procesador

Cuando el usuario responde afirmativamente a la pregunta de pagos en Ronda 4, hacer estas preguntas de seguimiento antes de la Fase 3:

1. "¿Qué vas a vender? (producto digital, servicio, curso, suscripción mensual)"
2. "¿Desde qué país operás y en qué moneda querés cobrar?"
3. "¿Ya tenés cuenta en Mercado Pago, Stripe, Lemon Squeezy o PayPal Business?"

Con esas respuestas, aplicar la decisión:

| Situación | Recomendación |
|---|---|
| Producto digital, cualquier país | Lemon Squeezy |
| Servicio o producto físico en LATAM | Mercado Pago |
| Necesita cuotas sin interés en Argentina | Mercado Pago obligatorio |
| Opera en EE.UU., Europa, o necesita suscripciones | Stripe |
| No sabe qué quiere y está en LATAM | Mercado Pago como default |
| No sabe y está fuera de LATAM | Lemon Squeezy como default |
| Audiencia internacional o países sin Mercado Pago | PayPal (solo o como segunda opción) |
| Quiere maximizar conversión en checkout | PayPal + opción principal en paralelo |

Confirmar antes del build:
> "Voy a integrar [procesador] para que puedas cobrar [producto] directamente desde el sitio. El visitante hace clic en el botón y es llevado al checkout de [procesador] para completar el pago. ¿Confirmamos?"

---

## Ejemplos

### Sección hero con precio visible y botón de pago

```tsx
// Para un curso o producto digital — precio en el hero genera más conversión
<section className="py-24 px-6 bg-bg">
  <div className="max-w-3xl mx-auto">
    <h1 className="text-5xl font-bold text-foreground">
      Diseño web sin saber programar
    </h1>
    <p className="mt-6 text-lg text-foreground/70">
      Curso online con 12 módulos. Aprendés a construir tu sitio desde cero.
    </p>
    <div className="mt-10 flex flex-col sm:flex-row items-start gap-4">
      <LemonButton
        checkoutUrl="https://tutienda.lemonsqueezy.com/checkout/buy/abc123"
        label="Comprar el curso — $97 USD"
        variant="primary"
      />
      <a href="#contenido" className="text-foreground/60 hover:text-foreground py-4 text-sm">
        Ver el contenido completo →
      </a>
    </div>
    <p className="mt-3 text-xs text-foreground/40">
      Pago único · Acceso de por vida · Garantía de 30 días
    </p>
  </div>
</section>
```

---

## Seguridad en Pagos — Obligatorio antes del deploy

Esta sección debe ejecutarse como checklist de seguridad antes de activar pagos en producción. El agente NO debe permitir el deploy de pagos si algún punto del checklist final falla.

### 1. Variables de entorno — Nunca hardcodear keys

**Regla absoluta:** ninguna key de pago va en el código fuente. Todo va en `.env.local` (desarrollo) y en las variables de entorno de Vercel (producción).

**Estructura de variables por procesador:**

```bash
# .env.local — NUNCA commitear este archivo

# Stripe
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...   # pública: puede ir en cliente
STRIPE_SECRET_KEY=sk_live_...                     # secreta: solo servidor
STRIPE_WEBHOOK_SECRET=whsec_...                   # secreta: solo servidor

# Mercado Pago
NEXT_PUBLIC_MP_PUBLIC_KEY=APP_USR-...             # pública: puede ir en cliente
MP_ACCESS_TOKEN=APP_USR-...                       # secreta: solo servidor
MP_WEBHOOK_SECRET=tu_clave_secreta                # secreta: solo servidor

# PayPal
NEXT_PUBLIC_PAYPAL_CLIENT_ID=AX...               # pública: puede ir en cliente
PAYPAL_CLIENT_ID=AX...                           # también necesaria en servidor
PAYPAL_SECRET=EG...                              # secreta: solo servidor

# Lemon Squeezy
LEMON_SQUEEZY_API_KEY=eyJ...                     # secreta: solo servidor
LEMON_SQUEEZY_STORE_ID=12345                     # puede ser pública
LEMON_SQUEEZY_WEBHOOK_SECRET=tu_clave_secreta    # secreta: solo servidor
```

**Verificación rápida — detectar keys expuestas en el cliente:**
```bash
# Buscar variables sin NEXT_PUBLIC_ en Client Components
grep -rn "process\.env\." --include="*.tsx" . \
  | grep -v "NEXT_PUBLIC" \
  | grep -v node_modules \
  | grep -v "use server"
```

Si hay resultados en archivos `'use client'` → error de seguridad crítico.

**Cómo acceder en cada contexto:**
```tsx
// Server Component o API Route — acceso a variables secretas
const secretKey = process.env.STRIPE_SECRET_KEY!  // solo servidor

// Client Component — solo variables NEXT_PUBLIC_
const publishableKey = process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!
```

---

### 2. Validación de webhooks

Los webhooks son requests HTTP que los procesadores envían cuando ocurre un evento de pago. Sin validación, cualquiera podría enviar un webhook falso y activar acceso a productos sin pagar.

#### Stripe — `stripe.webhooks.constructEvent()`

```ts
// app/api/stripe-webhook/route.ts
import Stripe from 'stripe'
import { headers } from 'next/headers'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(req: Request) {
  const body = await req.text()  // texto crudo — NO parsear a JSON antes
  const headersList = await headers()
  const sig = headersList.get('stripe-signature')

  if (!sig) {
    return new Response('Missing stripe-signature header', { status: 400 })
  }

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    console.error('Webhook signature verification failed:', (err as Error).message)
    return new Response('Invalid signature', { status: 400 })
  }

  // Procesar solo eventos conocidos
  switch (event.type) {
    case 'payment_intent.succeeded':
      const intent = event.data.object as Stripe.PaymentIntent
      await activateAccess(intent.metadata.userId, intent.metadata.productId)
      break
    case 'customer.subscription.deleted':
      await revokeAccess(event.data.object as Stripe.Subscription)
      break
    default:
      // Ignorar eventos no manejados
  }

  return new Response('OK', { status: 200 })
}
```

#### Mercado Pago — HMAC-SHA256

```ts
// app/api/mp-webhook/route.ts
import { createHmac } from 'crypto'
import { headers } from 'next/headers'

export async function POST(req: Request) {
  const headersList = await headers()
  const xSignature = headersList.get('x-signature')
  const xRequestId = headersList.get('x-request-id')
  const body = await req.text()

  if (!xSignature || !xRequestId) {
    return new Response('Missing headers', { status: 400 })
  }

  // Parsear ts y v1 del header x-signature
  const parts = Object.fromEntries(
    xSignature.split(',').map(part => {
      const [key, value] = part.split('=')
      return [key.trim(), value.trim()]
    })
  )

  // Construir el mensaje a firmar
  const url = new URL(req.url)
  const dataId = url.searchParams.get('data.id') ?? ''
  const manifest = `id:${dataId};request-id:${xRequestId};ts:${parts.ts};`

  // Verificar la firma con HMAC-SHA256
  const expected = createHmac('sha256', process.env.MP_WEBHOOK_SECRET!)
    .update(manifest)
    .digest('hex')

  if (expected !== parts.v1) {
    return new Response('Invalid signature', { status: 400 })
  }

  const notification = JSON.parse(body)
  if (notification.type === 'payment') {
    // Consultar el estado real del pago con la API de MP
    const paymentRes = await fetch(
      `https://api.mercadopago.com/v1/payments/${notification.data.id}`,
      { headers: { Authorization: `Bearer ${process.env.MP_ACCESS_TOKEN}` } }
    )
    const payment = await paymentRes.json()
    if (payment.status === 'approved') {
      await activateAccess(payment.external_reference)
    }
  }

  return new Response('OK', { status: 200 })
}
```

#### PayPal — `PayPal-Transmission-Sig`

```ts
// app/api/paypal-webhook/route.ts
import { headers } from 'next/headers'

export async function POST(req: Request) {
  const headersList = await headers()
  const body = await req.text()

  // Verificar con la API de PayPal (delega la verificación a PayPal)
  const auth = Buffer.from(
    `${process.env.PAYPAL_CLIENT_ID}:${process.env.PAYPAL_SECRET}`
  ).toString('base64')

  const tokenRes = await fetch('https://api-m.paypal.com/v1/oauth2/token', {
    method: 'POST',
    headers: { Authorization: `Basic ${auth}`, 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'grant_type=client_credentials',
  })
  const { access_token } = await tokenRes.json()

  const verifyRes = await fetch('https://api-m.paypal.com/v1/notifications/verify-webhook-signature', {
    method: 'POST',
    headers: { Authorization: `Bearer ${access_token}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      auth_algo: headersList.get('paypal-auth-algo'),
      cert_url: headersList.get('paypal-cert-url'),
      transmission_id: headersList.get('paypal-transmission-id'),
      transmission_sig: headersList.get('paypal-transmission-sig'),
      transmission_time: headersList.get('paypal-transmission-time'),
      webhook_id: process.env.PAYPAL_WEBHOOK_ID,
      webhook_event: JSON.parse(body),
    }),
  })
  const { verification_status } = await verifyRes.json()

  if (verification_status !== 'SUCCESS') {
    return new Response('Invalid signature', { status: 400 })
  }

  const event = JSON.parse(body)
  if (event.event_type === 'PAYMENT.CAPTURE.COMPLETED') {
    await activateAccess(event.resource.custom_id)
  }

  return new Response('OK', { status: 200 })
}
```

---

### 3. Idempotency keys — prevenir cobros duplicados

Un usuario que hace clic dos veces en "Pagar" o una red inestable puede disparar dos requests. Sin idempotency, se cobra dos veces.

#### Stripe — idempotencyKey en la creación de PaymentIntent

```ts
// app/api/stripe-checkout/route.ts
import { randomUUID } from 'crypto'

export async function POST(req: Request) {
  const { priceId, userId } = await req.json()

  // La key combina userId + priceId para que la misma compra nunca cree dos intents
  const idempotencyKey = `${userId}-${priceId}-${new Date().toISOString().slice(0, 10)}`

  const session = await stripe.checkout.sessions.create(
    {
      mode: 'payment',
      line_items: [{ price: priceId, quantity: 1 }],
      success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/gracias`,
      cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/#precios`,
    },
    { idempotencyKey }
  )

  return Response.json({ url: session.url })
}
```

#### Mercado Pago — X-Idempotency-Key en preferencias

```ts
// app/api/mp-checkout/route.ts
export async function POST(req: Request) {
  const { title, price, externalRef } = await req.json()

  // externalRef es el ID único de la compra en tu sistema
  const idempotencyKey = `mp-${externalRef}`

  const res = await fetch('https://api.mercadopago.com/checkout/preferences', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.MP_ACCESS_TOKEN}`,
      'Content-Type': 'application/json',
      'X-Idempotency-Key': idempotencyKey,
    },
    body: JSON.stringify({
      items: [{ title, unit_price: price, quantity: 1 }],
      external_reference: externalRef,
      back_urls: {
        success: `${process.env.NEXT_PUBLIC_SITE_URL}/gracias`,
        failure: `${process.env.NEXT_PUBLIC_SITE_URL}/error-pago`,
      },
    }),
  })

  const preference = await res.json()
  return Response.json({ checkoutUrl: preference.init_point })
}
```

---

### 4. Sanitización de montos — calcular siempre en el servidor

**Nunca confiar en el monto enviado desde el frontend.** Un usuario puede modificar cualquier valor antes de enviarlo.

```ts
// app/api/stripe-checkout/route.ts — INSEGURO (no hacer esto)
export async function POST(req: Request) {
  const { priceId, amount } = await req.json()
  // MAL: usar el amount que mandó el cliente
  const session = await stripe.checkout.sessions.create({
    line_items: [{ price_data: { unit_amount: amount }, quantity: 1 }],
  })
}

// app/api/stripe-checkout/route.ts — SEGURO
const PRICE_CATALOG: Record<string, number> = {
  'plan-basico':  500,   // en centavos: $5.00 USD
  'plan-pro':    1500,   // $15.00 USD
  'plan-anual': 14900,   // $149.00 USD
}

export async function POST(req: Request) {
  const { planId } = await req.json()

  // El monto viene del servidor — el cliente solo manda el ID del plan
  const unitAmount = PRICE_CATALOG[planId]
  if (!unitAmount) {
    return new Response('Invalid plan', { status: 400 })
  }

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: [{ price_data: { currency: 'usd', unit_amount: unitAmount, product_data: { name: planId } }, quantity: 1 }],
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/gracias`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/#precios`,
  })

  return Response.json({ url: session.url })
}
```

Lo mismo aplica para Mercado Pago: el `unit_price` siempre viene de un catálogo en el servidor, no del request del cliente.

---

### 5. Rate limiting en API routes de pago

Sin rate limiting, un bot puede intentar miles de transacciones por segundo, generando cargos en las APIs de pago o agotando cuotas.

```bash
npm install @upstash/ratelimit @upstash/redis
```

```ts
// lib/ratelimit.ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

export const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '60s'),  // 5 requests por minuto por IP
  analytics: true,
})
```

```ts
// app/api/stripe-checkout/route.ts
import { ratelimit } from '@/lib/ratelimit'
import { headers } from 'next/headers'

export async function POST(req: Request) {
  const headersList = await headers()
  const ip = headersList.get('x-forwarded-for') ?? '127.0.0.1'

  const { success, limit, remaining } = await ratelimit.limit(ip)
  if (!success) {
    return new Response('Demasiadas solicitudes. Esperá un momento e intentá de nuevo.', {
      status: 429,
      headers: { 'X-RateLimit-Limit': String(limit), 'X-RateLimit-Remaining': String(remaining) },
    })
  }

  // ... resto de la lógica de checkout
}
```

**Alternativa sin Upstash** (para proyectos sin Redis):
```ts
// Rate limiting en memoria — se reinicia con cada deploy, solo para desarrollo
const requestCounts = new Map<string, { count: number; resetAt: number }>()

function checkRateLimit(ip: string, limit = 5, windowMs = 60_000): boolean {
  const now = Date.now()
  const entry = requestCounts.get(ip)

  if (!entry || now > entry.resetAt) {
    requestCounts.set(ip, { count: 1, resetAt: now + windowMs })
    return true
  }

  if (entry.count >= limit) return false
  entry.count++
  return true
}
```

---

### 6. Replay attack prevention — validar timestamp en webhooks

Un replay attack reenvía un webhook legítimo capturado anteriormente para activar acceso sin pagar. La defensa es rechazar webhooks con timestamp viejo.

**Stripe** lo maneja automáticamente: `constructEvent` rechaza eventos con más de 5 minutos de diferencia con el timestamp del header (configurable con el tercer parámetro `tolerance`).

```ts
// Stripe — tolerance por defecto es 300 segundos (5 minutos)
// Para ser más estricto:
stripe.webhooks.constructEvent(body, sig, secret, 180)  // 3 minutos máximo
```

**Mercado Pago — verificar `ts` del header x-signature:**
```ts
const MAX_AGE_MS = 5 * 60 * 1000  // 5 minutos

const timestamp = parseInt(parts.ts, 10) * 1000  // MP usa segundos
if (Date.now() - timestamp > MAX_AGE_MS) {
  return new Response('Webhook expired', { status: 400 })
}
```

**PayPal — verificar `paypal-transmission-time`:**
```ts
const headersList = await headers()
const transmissionTime = headersList.get('paypal-transmission-time')
if (transmissionTime) {
  const eventTime = new Date(transmissionTime).getTime()
  if (Date.now() - eventTime > 5 * 60 * 1000) {
    return new Response('Webhook expired', { status: 400 })
  }
}
```

---

### 7. Portal del cliente — suscripciones y reembolsos

#### Stripe — Customer Portal

```ts
// app/api/stripe-portal/route.ts
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(req: Request) {
  const { customerId } = await req.json()

  // customerId debe venir de la sesión del usuario autenticado — nunca del body sin verificar
  const session = await stripe.billingPortal.sessions.create({
    customer: customerId,
    return_url: `${process.env.NEXT_PUBLIC_SITE_URL}/cuenta`,
  })

  return Response.json({ url: session.url })
}
```

Configurar el portal en el Dashboard de Stripe → Settings → Customer Portal: habilitar cancelación, descarga de facturas y actualización de método de pago.

#### Mercado Pago — reembolsos

```ts
// app/api/mp-refund/route.ts
export async function POST(req: Request) {
  const { paymentId } = await req.json()
  // Verificar que el paymentId pertenece al usuario autenticado antes de proceder

  const res = await fetch(`https://api.mercadopago.com/v1/payments/${paymentId}/refunds`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.MP_ACCESS_TOKEN}`,
      'Content-Type': 'application/json',
      'X-Idempotency-Key': `refund-${paymentId}`,
    },
  })

  const refund = await res.json()
  return Response.json({ status: refund.status })
}
```

---

### 8. Manejo de errores — mostrar al usuario, no exponer internals

**Regla:** el usuario ve un mensaje claro. Los logs del servidor capturan el detalle técnico. Nunca mostrar stack traces, IDs internos de pago, o mensajes de error de la API al cliente.

```tsx
// components/PaymentError.tsx
'use client'

interface PaymentErrorProps {
  type: 'card_declined' | 'insufficient_funds' | 'expired_card' | 'generic'
  supportEmail?: string
}

const ERROR_MESSAGES: Record<PaymentErrorProps['type'], string> = {
  card_declined:       'Tu tarjeta fue rechazada. Verificá los datos o intentá con otra tarjeta.',
  insufficient_funds:  'Fondos insuficientes. Verificá el saldo de tu tarjeta.',
  expired_card:        'Tu tarjeta está vencida. Usá una tarjeta vigente.',
  generic:             'Hubo un problema con el pago. Intentá de nuevo en unos minutos.',
}

export function PaymentError({ type, supportEmail }: PaymentErrorProps) {
  return (
    <div role="alert" className="p-4 bg-red-50 border border-red-200 rounded text-sm">
      <p className="font-medium text-red-800">{ERROR_MESSAGES[type]}</p>
      {supportEmail && (
        <p className="text-red-600 mt-1">
          Si el problema persiste, contactanos en{' '}
          <a href={`mailto:${supportEmail}`} className="underline">{supportEmail}</a>
        </p>
      )}
    </div>
  )
}
```

**En la API route — loggear sin exponer:**
```ts
export async function POST(req: Request) {
  try {
    // ... lógica de pago
  } catch (err) {
    // Loggear el error completo en el servidor (Vercel logs, Sentry, etc.)
    console.error('[payment-error]', {
      message: (err as Error).message,
      stack: (err as Error).stack,
      timestamp: new Date().toISOString(),
    })

    // Responder al cliente con un mensaje genérico — sin detalles internos
    return Response.json(
      { error: 'payment_failed', type: 'generic' },
      { status: 500 }
    )
  }
}
```

---

### 9. Testing de pagos — modo sandbox antes de producción

**Regla:** nunca activar credenciales de producción sin haber probado el flujo completo en sandbox.

#### Stripe — tarjetas de prueba

```
# Pago exitoso
Número: 4242 4242 4242 4242
Fecha:  cualquier fecha futura
CVC:    cualquier 3 dígitos

# Tarjeta rechazada
Número: 4000 0000 0000 0002

# Fondos insuficientes
Número: 4000 0000 0000 9995

# Requiere autenticación 3D Secure
Número: 4000 0025 0000 3155
```

Keys de test: `pk_test_...` y `sk_test_...` — el dashboard de Stripe las muestra en modo "Test".

#### Mercado Pago — usuarios de prueba

```bash
# Crear usuarios de prueba vía API
curl -X POST \
  "https://api.mercadopago.com/users/test" \
  -H "Authorization: Bearer TEST-..." \
  -H "Content-Type: application/json" \
  -d '{"site_id": "MLA"}'  # MLA=Argentina, MLM=México, MLC=Chile, MCO=Colombia, MPY=Paraguay
```

Tarjetas de prueba por país disponibles en: https://www.mercadopago.com.ar/developers/es/docs/checkout-api/integration-test/test-cards

#### PayPal — cuentas Sandbox

1. Ir a https://developer.paypal.com/tools/sandbox/accounts
2. PayPal crea automáticamente una cuenta de comprador y una de vendedor de prueba
3. Usar el Client ID del modo Sandbox (empieza diferente al de producción)
4. Probar con las credenciales de la cuenta de comprador de sandbox

**Verificar en sandbox antes de producción:**
- [ ] Flujo de pago completo (selección → checkout → confirmación)
- [ ] Webhook recibido y procesado correctamente
- [ ] Acceso al producto activado después del pago
- [ ] Flujo de reembolso
- [ ] Manejo de error por tarjeta rechazada

---

### 10. Compliance LATAM — consideraciones legales básicas

#### Paraguay
- **IVA:** 10% general, 5% para productos de primera necesidad y medicamentos
- **Facturación:** no existe facturación electrónica obligatoria generalizada (2025), pero se recomienda emitir comprobante de pago
- **DNIT** (Dirección Nacional de Ingresos Tributarios): registrarse si el volumen mensual supera el mínimo imponible
- **Términos de servicio mínimos:** nombre del proveedor, descripción del servicio, precio con IVA incluido, política de cancelación/reembolso

#### Argentina
- **IVA:** 21% general, 10.5% para algunos servicios digitales
- **AFIP:** los servicios digitales de proveedores del exterior están gravados; para negocios locales, facturar con IVA incluido
- **Factura electrónica:** obligatoria para monotributistas y responsables inscriptos — usar servicios como Facturade, Alegra o Tango
- **Impuesto PAIS y percepciones:** los pagos con tarjeta en USD tienen cargas adicionales

#### México
- **IVA:** 16% general
- **SAT:** facturación electrónica (CFDI) obligatoria para personas morales y personas físicas con actividad empresarial
- **Marketplace:** si vendés en México, Mercado Pago México retiene y reporta IVA automáticamente en muchos casos
- **CLABE:** los pagos locales requieren CLABE interbancaria; Mercado Pago la maneja

#### Ecuador
- **IVA:** 15% (desde 2024)
- **SRI:** el Servicio de Rentas Internas requiere facturación electrónica para actividades comerciales
- **Sector digital:** los servicios digitales de no residentes están sujetos a retención

**Para todos los mercados — mínimos en los términos de servicio:**
1. Nombre o razón social del vendedor
2. RUC/CUIT/RFC/NIT según el país
3. Precio con impuestos incluidos claramente indicado
4. Política de cancelación y reembolso (plazo, condiciones)
5. Descripción clara del producto o servicio
6. Canales de contacto y tiempo de respuesta

---

### 11. Monitoreo post-deploy

#### Alertas cuando falla un webhook

**Stripe — configurar alerta en el Dashboard:**
- Stripe → Developers → Webhooks → seleccionar el endpoint → "Alerts" → activar alerta por email cuando el endpoint falla 3 veces consecutivas

**Mercado Pago:**
- El reintento de webhooks fallidos es automático (MP reintenta hasta 24 horas)
- Loggear cada webhook recibido con su ID para detectar duplicados:
```ts
// En la API route del webhook
console.log('[mp-webhook]', { id: notification.data.id, type: notification.type, ts: Date.now() })
```

**PayPal:**
- PayPal → Developer Dashboard → My Apps → seleccionar app → Webhooks → ver el historial de eventos y reenviar manualmente si es necesario

#### Detectar cobros duplicados en producción

```ts
// Guardar cada payment ID procesado en una store (base de datos o KV)
// Ejemplo con Vercel KV (Redis)
import { kv } from '@vercel/kv'

async function isAlreadyProcessed(paymentId: string): Promise<boolean> {
  const exists = await kv.exists(`processed:${paymentId}`)
  return exists === 1
}

async function markAsProcessed(paymentId: string): Promise<void> {
  // Guardar por 30 días
  await kv.set(`processed:${paymentId}`, '1', { ex: 30 * 24 * 60 * 60 })
}

// En el webhook handler
if (await isAlreadyProcessed(paymentId)) {
  console.warn('[webhook-duplicate]', paymentId)
  return new Response('Already processed', { status: 200 })  // 200 para no disparar reintento
}
await markAsProcessed(paymentId)
// ... procesar
```

#### Herramientas de monitoreo recomendadas

| Procesador | Herramienta | Qué monitorear |
|---|---|---|
| Stripe | Stripe Radar + Dashboard | Pagos rechazados, chargebacks, webhooks fallidos |
| Mercado Pago | Dashboard MP + logs de Vercel | Pagos pendientes, rechazados, webhooks no procesados |
| PayPal | PayPal Resolution Center | Disputas, chargebacks, pagos retenidos |
| Todos | Vercel Logs | Errores en API routes, rate limiting activado |
| Todos | Sentry (opcional) | Excepciones en webhooks y checkout |

---

### 12. Checklist pre-deploy de pagos — 15 puntos

El agente debe verificar cada punto. Si alguno falla, NO proceder con el deploy de pagos hasta corregirlo.

```
VARIABLES DE ENTORNO
- [ ] 1. Ninguna API key hardcodeada en el código fuente
- [ ] 2. .env.local está en .gitignore y no fue commiteado
- [ ] 3. Variables de producción configuradas en Vercel (no solo en .env.local)
- [ ] 4. Keys secretas (STRIPE_SECRET_KEY, MP_ACCESS_TOKEN, PAYPAL_SECRET)
         solo en variables sin NEXT_PUBLIC_

WEBHOOKS
- [ ] 5. Webhook endpoint registrado en el dashboard del procesador
- [ ] 6. Firma del webhook verificada antes de procesar cualquier evento
- [ ] 7. Timestamp verificado — rechazar webhooks con más de 5 minutos de antigüedad
- [ ] 8. Idempotency implementada — no procesar el mismo pago dos veces

MONTOS Y SEGURIDAD
- [ ] 9. Monto calculado en el servidor — no en el cliente
- [ ] 10. Rate limiting activo en todas las API routes de pago
- [ ] 11. Inputs sanitizados — validar tipo y rango antes de pasarlos a la API de pago

TESTING
- [ ] 12. Flujo completo probado en modo sandbox/test antes de activar producción
- [ ] 13. Webhook de sandbox procesado correctamente al menos una vez
- [ ] 14. Manejo de error por tarjeta rechazada probado y el usuario ve mensaje claro

COMPLIANCE
- [ ] 15. Términos de servicio incluyen: nombre del vendedor, precio con impuestos,
          política de reembolso, y canal de contacto

---
Si todos los 15 puntos pasan → activar credenciales de producción y hacer deploy.
Si alguno falla → corregir antes de proceder. Reportar al usuario qué falta.
```
