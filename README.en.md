# skill-payments

![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)
![Stars](https://img.shields.io/github/stars/luisrecalde/skill-payments?style=social)
![Forks](https://img.shields.io/github/forks/luisrecalde/skill-payments?style=social)

A [Claude Code](https://claude.ai/code) skill that integrates online payments into Next.js + Tailwind projects. It selects the right payment processor based on the country and business type, generates ready-to-use components, and runs a security checklist before deployment.

**Who it's for:** developers and agencies building sites with Claude Code who need to activate online payments without manually configuring each processor.

---

## Supported processors

| Processor | Best for |
|---|---|
| **Lemon Squeezy** | Digital products (courses, ebooks, templates, memberships) — no backend required |
| **Mercado Pago** | LATAM businesses — local cards, bank transfers, installments |
| **Stripe** | International markets, subscriptions, marketplaces |
| **PayPal** | Global audiences or countries where Mercado Pago is unavailable |

### Special support by country

The skill understands the specific requirements of each market:

- **Paraguay** — 10% VAT, DNIT compliance, site in Guaraní or Spanish
- **Ecuador** — 15% VAT (effective since 2024), SRI electronic invoicing
- **Argentina** — 21% VAT, AFIP electronic invoicing, interest-free installments, PAIS tax
- **Mexico** — 16% VAT, CFDI (SAT), CLABE bank account, automatic marketplace VAT withholding
- **United States** — no federal VAT, sales tax varies by state, W-9/1099 for contractor payments, Stripe recommended

---

## Installation

### Option 1 — Global (recommended)

```bash
git clone https://github.com/luis-recalde/skill-payments ~/.claude/skills/skill-payments
```

### Option 2 — This project only

```bash
cp ~/.claude/skills/skill-payments/SKILL.md .claude/SKILL.md
```

---

## How to use it

Once the skill is installed, simply describe what you need in plain language inside Claude Code:

```
I want to sell my online course for $97 USD directly from the site.
I operate from Paraguay and want to accept payments in guaraníes.
```

```
I need to integrate Stripe for monthly subscriptions.
```

```
Add a Mercado Pago button to charge for the consulting service.
The price is $200 USD, one-time payment.
```

```
I want to offer PayPal as a second payment option alongside Lemon Squeezy.
```

Claude Code will make automatic decisions based on:
- The country you operate from
- The type of product (digital, service, subscription)
- Whether you already have an account with any processor

---

## Security included

The skill applies security best practices to every generated integration:

- **Environment variables** — no API keys in source code; secret keys are never exposed to the client
- **Webhook validation** — HMAC-SHA256 signature verification (Mercado Pago), `constructEvent` (Stripe), and verification API (PayPal)
- **Idempotency keys** — prevention of duplicate charges from double-clicks or unstable networks
- **Server-side amount calculation** — the client never sends the price; it always comes from a backend catalog
- **Rate limiting** — protection on payment API routes (Upstash Redis or in-memory)
- **Replay attack prevention** — rejection of webhooks with timestamps older than 5 minutes
- **15-point pre-deploy checklist** — the agent verifies every point before activating production credentials

---

## Requirements

- Any web project
- Node.js 18+

---

## Author

**Luis Recalde**
[info@luisrecalde.com](mailto:info@luisrecalde.com)

---

## License

[MIT](./LICENSE) © 2026 Luis Recalde

---

[Versión en español](./README.md)
