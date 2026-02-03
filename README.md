# billing.io — Crypto Payment API

Non-custodial crypto payment API for accepting USDT and USDC stablecoin payments on Tron, Arbitrum, and Base.

## What is billing.io?

billing.io is a crypto payment gateway and billing API that lets merchants accept stablecoin payments (USDT, USDC) across multiple blockchains. It is fully non-custodial — crypto goes directly from the customer's wallet to yours. billing.io never holds, touches, or has access to your funds.

Use billing.io to accept crypto payments, manage recurring crypto subscriptions, orchestrate on-chain payouts, and track revenue — all through a REST API with SDKs for JavaScript, Python, Go, Ruby, Java, and React.

## Non-custodial crypto payments

Unlike traditional payment processors that hold your funds before paying you out, billing.io routes payments directly to your wallet on-chain:

- **Direct wallet-to-wallet payments** — no intermediary holds your funds
- **No frozen accounts** — your money is in your wallet the moment it arrives
- **No payout delays** — funds settle on-chain in minutes, not days
- **No counterparty risk** — if billing.io disappeared tomorrow, your funds would be unaffected
- **Fully verifiable** — every transaction is on the public blockchain

Orchestration fees are charged separately and are never deducted from payments.

## Features

- **Crypto checkout API** — Generate unique deposit addresses for one-time crypto payments
- **Crypto subscription billing** — Recurring billing with plans, trials, renewals, and automated retries — all paid in stablecoins
- **Stablecoin payout orchestration** — Define payout intents, execute from your own wallet, and let billing.io verify and reconcile settlements on-chain
- **Blockchain monitoring** — Real-time transaction detection and block confirmation tracking across Tron, Arbitrum, and Base
- **Webhook notifications** — Push `checkout.completed` events to your server the instant a payment is confirmed
- **Revenue tracking** — Immutable ledger of revenue events, period-based accounting, and adjustments
- **Entitlements** — Attach feature flags to subscription plans and gate access based on payment state
- **Payment links** — Shareable URLs that generate crypto checkouts on the fly

## Supported chains and tokens

| Chain | Tokens | Status |
|-------|--------|--------|
| Tron | USDT, USDC | Production |
| Arbitrum | USDT, USDC | Production |
| Base | USDT, USDC | Production |

All tokens are USD-pegged stablecoins. No exchange rate risk.

## How it works

```
1. Your server calls POST /checkouts with an amount, chain, and token
2. billing.io returns a unique deposit address
3. Customer sends stablecoins to the deposit address
4. billing.io monitors the blockchain for the transaction
5. Once required block confirmations are reached, the checkout is confirmed
6. A checkout.completed webhook fires to your server
7. Funds are already in your wallet — nothing to withdraw
```

## Quick example

```bash
curl -X POST https://api.billing.io/v1/checkouts \
  -H "Authorization: Bearer $BILLING_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_usd": 49.99,
    "chain": "tron",
    "token": "USDT"
  }'
```

## Documentation

This repo contains the source for [docs.billing.io](https://docs.billing.io), built with [Mintlify](https://mintlify.com).

### Structure

```
api-reference/    API endpoint documentation (OpenAPI-backed)
concepts/         Core domain explanations (payment methods, subscriptions, payouts, revenue)
guides/           Step-by-step integration guides
sdks/             Official SDK documentation (JavaScript, Python, Go, React, Ruby, Java, billing.js)
openapi/          OpenAPI 3.0 specification
```

### SDKs

| Language | Package |
|----------|---------|
| JavaScript | [@billing-io/sdk](https://www.npmjs.com/package/@billing-io/sdk) |
| Python | [billingio](https://pypi.org/project/billingio) |
| Go | [github.com/billing-io/billingio-go](https://github.com/billing-io/billingio-go) |
| React | [@billing-io/react](https://www.npmjs.com/package/@billing-io/react) |
| Ruby | [billingio](https://rubygems.org/gems/billingio) |
| Java | [io.billing:billingio-java](https://central.sonatype.com/artifact/io.billing/billingio-java) |
| Browser | [@billing-io/js](https://www.npmjs.com/package/@billing-io/js) |

### Local development

```bash
npm i -g mintlify
mintlify dev
```

## Links

- [Dashboard](https://app.billing.io/dashboard)
- [API Reference](https://docs.billing.io/api-reference/authentication)
- [Discord](https://discord.gg/billing-io)
- [GitHub](https://github.com/billing-io)
