# billing.io

Non-custodial crypto payment infrastructure for merchants.

## What is billing.io?

billing.io lets merchants accept stablecoin payments (USDT and USDC) across multiple blockchains without ever giving up custody of their funds. When a customer pays, the crypto goes directly from their wallet to yours. billing.io monitors the blockchain, confirms the payment, and notifies your server via webhook. At no point does billing.io hold, touch, or have access to your funds.

## Non-custodial by design

Traditional payment processors receive your customer's money first, hold it, take a cut, and send you the rest days or weeks later. billing.io works differently:

- **Payments go directly to your wallet** — no intermediary holds your funds
- **No frozen accounts** — your money is in your wallet the moment it arrives
- **No payout delays** — funds settle on-chain in minutes, not days
- **No counterparty risk** — if billing.io disappeared tomorrow, your funds would be unaffected
- **Full transparency** — every transaction is verifiable on the public blockchain

Orchestration fees are charged separately and are never deducted from payments.

## What billing.io does

billing.io handles everything around the payment without being in the flow of funds:

- **Checkouts** — Generate unique deposit addresses for each payment session
- **Blockchain monitoring** — Detect incoming transactions and track block confirmations in real time
- **Webhook notifications** — Push `checkout.completed` events to your server the instant a payment is confirmed
- **Subscription billing** — Manage plans, trials, renewals, and automated retry logic for recurring crypto payments
- **Payout orchestration** — Define payout intents, execute from your own wallet, and let billing.io verify and reconcile settlements on-chain
- **Revenue tracking** — Immutable ledger of revenue events, period-based accounting summaries, and adjustments (credits, debits, chargebacks)
- **Entitlements** — Attach feature flags to subscription plans and check access at runtime
- **Payment links** — Shareable URLs that generate checkouts on the fly

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
