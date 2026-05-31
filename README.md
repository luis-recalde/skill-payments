# skill-payments

![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)
![Stars](https://img.shields.io/github/stars/luisrecalde/skill-payments?style=social)
![Forks](https://img.shields.io/github/forks/luisrecalde/skill-payments?style=social)

Skill para [Claude Code](https://claude.ai/code) que integra pagos online en proyectos Next.js + Tailwind. Elige el procesador correcto según el país y el tipo de negocio, genera los componentes listos para usar, y aplica un checklist de seguridad antes del deploy.

**Para quién es:** desarrolladores y agencias que construyen sitios con Claude Code y necesitan activar cobros online sin configurar manualmente cada procesador.

---

## Procesadores soportados

| Procesador | Ideal para |
|---|---|
| **Lemon Squeezy** | Productos digitales (cursos, ebooks, templates, membresías) sin backend |
| **Mercado Pago** | Negocios en LATAM — acepta tarjetas locales, transferencias, cuotas |
| **Stripe** | Mercados internacionales, suscripciones, marketplaces |
| **PayPal** | Audiencias globales o países donde Mercado Pago no opera |

### Soporte especial LATAM — Mercado Pago

La skill conoce las particularidades de cada mercado:

- **Paraguay** — IVA 10%, cumplimiento DNIT, sitio en guaraní o español
- **Ecuador** — IVA 15% (vigente desde 2024), facturación SRI
- **Argentina** — IVA 21%, factura electrónica AFIP, cuotas sin interés, impuesto PAIS
- **México** — IVA 16%, CFDI (SAT), CLABE interbancaria, retención automática en marketplace

---

## Instalación

### Opción 1 — Global (disponible en todos tus proyectos)

```bash
# Copiar el archivo SKILL.md a la carpeta global de skills de Claude Code
cp SKILL.md ~/.claude/skills/payments.md
```

### Opción 2 — Por proyecto

```bash
# Copiar el archivo SKILL.md a la raíz del proyecto
cp SKILL.md ./SKILL.md
```

Claude Code detecta automáticamente los archivos `SKILL.md` en la raíz del proyecto y en `~/.claude/skills/`.

---

## Cómo usarlo

Una vez instalada la skill, simplemente describí lo que necesitás en lenguaje natural dentro de Claude Code:

```
Quiero vender mi curso online de $97 USD desde el sitio.
Opero desde Paraguay y quiero cobrar en guaraníes.
```

```
Necesito integrar pagos con Stripe para suscripciones mensuales.
```

```
Agrega un botón de Mercado Pago para cobrar el servicio de consultoría.
El precio es $200 USD, pago único.
```

```
Quiero ofrecer PayPal como segunda opción de pago junto a Lemon Squeezy.
```

Claude Code tomará decisiones automáticas basadas en:
- El país desde donde operás
- El tipo de producto (digital, servicio, suscripción)
- Si ya tenés cuenta en algún procesador

---

## Seguridad incluida

La skill aplica buenas prácticas de seguridad en cada integración generada:

- **Variables de entorno** — ninguna API key va en el código fuente; las secretas nunca se exponen al cliente
- **Validación de webhooks** — verificación de firma HMAC-SHA256 (Mercado Pago), `constructEvent` (Stripe), y API de verificación (PayPal)
- **Idempotency keys** — prevención de cobros duplicados por doble clic o red inestable
- **Montos calculados en el servidor** — el cliente nunca envía el precio; viene del catálogo en el backend
- **Rate limiting** — protección en las API routes de pago (Upstash Redis o in-memory)
- **Replay attack prevention** — rechazo de webhooks con timestamp mayor a 5 minutos
- **Checklist pre-deploy de 15 puntos** — el agente verifica cada punto antes de activar producción

---

## Requisitos

- Next.js 14+ (App Router)
- Tailwind CSS
- Node.js 18+

---

## Autor

**Luis Recalde**
[info@luisrecalde.com](mailto:info@luisrecalde.com)

---

## Licencia

[MIT](./LICENSE) © 2026 Luis Recalde

---

[English version](./README.en.md)
