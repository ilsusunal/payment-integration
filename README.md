# payment-integration

**An Agent Skill that makes coding agents integrate payment providers into e-commerce backends correctly: an order is marked paid only when the provider says so, exactly once, for the right amount.**

Ask an agent to "add Stripe" or "add crypto payments" and the code usually runs. It also often trusts the success redirect, verifies the webhook signature on a re-serialised body, processes retried webhooks twice, and never compares the paid amount with the order. This skill turns that hard-won knowledge into a workflow, rules, tested templates and two scripts.

Works with any agent that supports the open [Agent Skills](https://agentskills.io/) format (for example Claude Code). Templates target Node.js / TypeScript with Next.js, Express or Fastify; the rules apply to any stack.

## Providers

| Provider | Flow | Status |
|---|---|---|
| Stripe | Hosted Checkout | v0.1 |
| Cryptomus | Crypto invoice | v0.1 |
| NOWPayments | Crypto invoice (HMAC-SHA512 IPN) | planned v0.2 |
| iyzico, Payop, Ziina | various | planned, [help wanted](CONTRIBUTING.md#adding-a-provider) |

## What the agent gets

```
skills/payment-integration/
├── SKILL.md                         workflow + 9 non-negotiable rules + symptom table
├── references/
│   ├── patterns/                    webhook security, idempotency, state machine, money, trust boundaries
│   └── providers/                   stripe.md, cryptomus.md, _template.md (loaded only when needed)
├── assets/templates/                TypeScript the agent copies into the project
│   ├── payment-provider.ts          PaymentProvider interface: createPayment / verifyWebhook / getStatus / refund
│   ├── webhook-handler.ts           verify -> claim event -> load order -> check amount -> compare-and-set
│   ├── money.ts, order-state-machine.ts, idempotency.ts
│   ├── providers/stripe.ts, providers/cryptomus.ts   (dependency-free: fetch + node:crypto)
│   └── frameworks/                  Next.js App Router, Express, Fastify webhook routes
└── scripts/
    ├── replay-webhook.mjs           sends valid, forged, tampered, stale, duplicate, underpaid webhooks to localhost
    └── check-integration.mjs        static scan for 12 known pitfalls
```

## Install

Copy the skill folder into your agent's skills directory. For Claude Code:

```bash
# for one project
mkdir -p .claude/skills && cp -r skills/payment-integration .claude/skills/
# or for all your projects
mkdir -p ~/.claude/skills && cp -r skills/payment-integration ~/.claude/skills/
```

Then ask naturally: *"Add Cryptomus payments to this Next.js shop"*, *"Our Stripe webhook says signature verification failed"*, *"Review our payment code"*.

## Use the scripts without the agent

```bash
# Scan a codebase for known pitfalls (exit 1 on HIGH findings)
node skills/payment-integration/scripts/check-integration.mjs ./src

# Hammer your local webhook endpoint with good and bad deliveries
STRIPE_WEBHOOK_SECRET=whsec_... node skills/payment-integration/scripts/replay-webhook.mjs \
  --provider stripe --url http://localhost:3000/api/webhooks/stripe --order-id <order> --amount 19.99 --currency USD
```

The replay script refuses non-local URLs unless you pass `--allow-remote`. Point it only at endpoints you own, with sandbox secrets.

## What is tested

`npm test` runs 51 tests on Node's built-in runner, with no dependencies:

- Stripe signature verification: forged, tampered, re-serialised body, stale timestamp, missing and malformed headers, multiple `v1` values, secret rotation, unsigned debug entries ignored.
- Cryptomus: a webhook fixture **signed by PHP's own `json_encode`** (URLs, Turkish characters, emoji, nested objects). The test also shows that plain `JSON.stringify` fails on it. Status mapping never turns underpayment or unknown statuses into `paid`.
- Conformance, for any provider: duplicate and concurrent deliveries fulfil once, amount and currency mismatches are flagged, out-of-order events cannot downgrade an order, a failed processing run is retried, the return page asks the API, and instructions hidden in payload text have no effect.
- End to end: a real HTTP server built from the templates, exercised by `replay-webhook.mjs`.
- The scanner: it catches every pitfall in a deliberately broken fixture and reports nothing on the good fixture or on the templates.

## Roadmap

- **Skill vs no skill eval:** the same 5 tasks given to a coding agent with and without the skill, scored by the conformance tests above. Results will be published here as measured, not estimated.
- Demo shop (Next.js + Node, sandbox keys only).
- NOWPayments, then iyzico, Payop, Ziina.
- A small MCP server that creates sandbox payments and triggers test webhooks.

## Related

Stripe publishes [official agent skills and an MCP server](https://docs.stripe.com/skills) for Stripe-specific work. This project is provider-neutral: one interface, one set of safety rules and one test suite across card and crypto providers.

## Contributing

New providers are the best first contribution: each one is a reference file, an adapter and a test file. See [CONTRIBUTING.md](CONTRIBUTING.md). Please report vulnerabilities privately, as described in [SECURITY.md](SECURITY.md).

## Disclaimer

Not affiliated with any payment provider. Provider facts come from their public documentation as of November 2026 and can change; check the linked docs before going live. The code is provided under the [MIT License](LICENSE), without warranty.
