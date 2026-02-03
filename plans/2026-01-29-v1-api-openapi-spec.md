# billing.io v1 API & OpenAPI Spec Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Lock the canonical `/v1` API surface, generate the OpenAPI 3.0 spec, and update the client library to match — enabling SDK generation for JS, React, Python, Go, Ruby, and Java.

**Architecture:** The Chain Watcher is the backend REST API. This repo contains the Next.js frontend client. The OpenAPI spec is the single source of truth for the API contract. The client library (`src/lib/api.ts`) must conform to the spec. Webhooks are first-class with HMAC signing. Polling is a fallback.

**Tech Stack:** OpenAPI 3.0.3 (YAML), TypeScript, Next.js 16, Supabase Auth

---

## Decisions (locked)

| Decision | Choice |
|---|---|
| Auth | `Authorization: Bearer sk_live_...` / `sk_test_...` |
| Errors | Structured JSON: `{ error: { type, code, message, param? } }` |
| `underpaid` state | Removed from API types. UI-only derived condition. |
| Webhooks | First-class, HMAC-SHA256 signed, retryable, typed events |
| Polling | Supported as fallback. Server returns `required_confirmations` + `polling_interval_ms`. |
| Pagination | Cursor-based: `?cursor=...&limit=25` |
| Idempotency | `Idempotency-Key` header (client-generated UUID) on POST |
| Version prefix | `/v1/` on all endpoints |

## Error Taxonomy (locked)

| `error.type` | HTTP Status | When |
|---|---|---|
| `invalid_request` | 400 | Missing/malformed params |
| `authentication_error` | 401 | Missing or invalid API key |
| `not_found` | 404 | Resource doesn't exist |
| `idempotency_conflict` | 409 | Same key, different params |
| `rate_limited` | 429 | Too many requests |
| `internal_error` | 500 | Server failure |

## Error Codes (per type)

**`invalid_request`:** `missing_required_field`, `invalid_field_value`, `invalid_chain`, `invalid_token`, `amount_too_small`, `amount_too_large`, `expires_too_short`, `expires_too_long`, `invalid_cursor`, `invalid_limit`

**`not_found`:** `checkout_not_found`, `webhook_not_found`, `event_not_found`

**`authentication_error`:** `api_key_missing`, `api_key_invalid`, `api_key_revoked`

**`rate_limited`:** `too_many_requests`

**`idempotency_conflict`:** `idempotency_key_reused`

**`internal_error`:** `internal_server_error`

## Webhook Events (locked)

| Event Type | Trigger |
|---|---|
| `checkout.created` | POST /v1/checkouts succeeds |
| `checkout.payment_detected` | Chain watcher detects tx |
| `checkout.confirming` | Confirmation count > 0 |
| `checkout.completed` | Required confirmations reached |
| `checkout.expired` | Checkout TTL exceeded |
| `checkout.failed` | Unrecoverable failure |

## Webhook Signing Scheme (locked)

- Algorithm: HMAC-SHA256
- Secret: `whsec_...` (provided at webhook endpoint creation)
- Signed payload: `{timestamp}.{raw_body}`
- Header: `X-Billing-Signature: t={timestamp},v1={hex_signature}`
- Tolerance window: 5 minutes (300 seconds)
- Retry policy: Exponential backoff, 3 attempts (immediate, 5min, 30min)

---

## Task 1: Create the OpenAPI 3.0 Spec

**Files:**
- Create: `docs/openapi/v1.yaml`

**Step 1: Write the OpenAPI spec file**

Create `docs/openapi/v1.yaml` with the full specification below. This is the canonical contract.

```yaml
openapi: 3.0.3
info:
  title: billing.io API
  description: |
    Non-custodial crypto checkout API. Create payment checkouts, monitor status
    via polling or webhooks, and manage webhook endpoints.
  version: 1.0.0
  contact:
    name: billing.io Support
  license:
    name: Proprietary

servers:
  - url: https://api.billing.io/v1
    description: Production
  - url: http://localhost:8080/v1
    description: Local development

security:
  - BearerAuth: []

tags:
  - name: Checkouts
    description: Create and monitor crypto payment checkouts
  - name: Webhooks
    description: Register endpoints to receive event notifications
  - name: Events
    description: Query event history
  - name: Health
    description: Service health

paths:
  /health:
    get:
      operationId: getHealth
      tags: [Health]
      summary: Health check
      security: []
      responses:
        "200":
          description: Service is healthy
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/HealthResponse"

  /checkouts:
    post:
      operationId: createCheckout
      tags: [Checkouts]
      summary: Create a checkout
      description: |
        Creates a new payment checkout and returns a deposit address.
        Supports idempotency via the `Idempotency-Key` header.
      parameters:
        - $ref: "#/components/parameters/IdempotencyKey"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateCheckoutRequest"
      responses:
        "201":
          description: Checkout created
          headers:
            Idempotency-Key:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Checkout"
        "400":
          $ref: "#/components/responses/BadRequest"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "409":
          $ref: "#/components/responses/IdempotencyConflict"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

    get:
      operationId: listCheckouts
      tags: [Checkouts]
      summary: List checkouts
      description: Returns a paginated list of checkouts, newest first.
      parameters:
        - $ref: "#/components/parameters/Cursor"
        - $ref: "#/components/parameters/Limit"
        - name: status
          in: query
          required: false
          schema:
            $ref: "#/components/schemas/CheckoutStatus"
      responses:
        "200":
          description: Checkout list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/CheckoutList"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /checkouts/{checkout_id}:
    get:
      operationId: getCheckout
      tags: [Checkouts]
      summary: Get a checkout
      parameters:
        - $ref: "#/components/parameters/CheckoutId"
      responses:
        "200":
          description: Checkout details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Checkout"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /checkouts/{checkout_id}/status:
    get:
      operationId: getCheckoutStatus
      tags: [Checkouts]
      summary: Get checkout status
      description: |
        Lightweight status endpoint for polling. Returns current status,
        confirmations, and polling hints.
      parameters:
        - $ref: "#/components/parameters/CheckoutId"
      responses:
        "200":
          description: Checkout status
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/CheckoutStatusResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /webhooks:
    post:
      operationId: createWebhookEndpoint
      tags: [Webhooks]
      summary: Create a webhook endpoint
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateWebhookRequest"
      responses:
        "201":
          description: Webhook endpoint created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/WebhookEndpoint"
        "400":
          $ref: "#/components/responses/BadRequest"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

    get:
      operationId: listWebhookEndpoints
      tags: [Webhooks]
      summary: List webhook endpoints
      parameters:
        - $ref: "#/components/parameters/Cursor"
        - $ref: "#/components/parameters/Limit"
      responses:
        "200":
          description: Webhook endpoint list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/WebhookEndpointList"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /webhooks/{webhook_id}:
    get:
      operationId: getWebhookEndpoint
      tags: [Webhooks]
      summary: Get a webhook endpoint
      parameters:
        - $ref: "#/components/parameters/WebhookId"
      responses:
        "200":
          description: Webhook endpoint details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/WebhookEndpoint"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

    delete:
      operationId: deleteWebhookEndpoint
      tags: [Webhooks]
      summary: Delete a webhook endpoint
      parameters:
        - $ref: "#/components/parameters/WebhookId"
      responses:
        "204":
          description: Webhook endpoint deleted
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /events:
    get:
      operationId: listEvents
      tags: [Events]
      summary: List events
      description: |
        Returns a paginated list of webhook events, newest first.
        Useful for replaying missed webhooks or debugging.
      parameters:
        - $ref: "#/components/parameters/Cursor"
        - $ref: "#/components/parameters/Limit"
        - name: type
          in: query
          required: false
          schema:
            $ref: "#/components/schemas/EventType"
        - name: checkout_id
          in: query
          required: false
          schema:
            type: string
      responses:
        "200":
          description: Event list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EventList"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

  /events/{event_id}:
    get:
      operationId: getEvent
      tags: [Events]
      summary: Get an event
      parameters:
        - name: event_id
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Event details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Event"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/RateLimited"
        "500":
          $ref: "#/components/responses/InternalError"

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      description: |
        Use your secret API key as a Bearer token.
        Keys are prefixed: `sk_live_` (production) or `sk_test_` (sandbox).

  parameters:
    CheckoutId:
      name: checkout_id
      in: path
      required: true
      schema:
        type: string
      description: The checkout identifier (prefixed `co_`)

    WebhookId:
      name: webhook_id
      in: path
      required: true
      schema:
        type: string
      description: The webhook endpoint identifier (prefixed `we_`)

    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: false
      schema:
        type: string
        format: uuid
      description: |
        Client-generated UUID. If a request with the same key was already
        processed, the original response is returned. Keys expire after 24h.

    Cursor:
      name: cursor
      in: query
      required: false
      schema:
        type: string
      description: Opaque cursor for pagination. Omit for the first page.

    Limit:
      name: limit
      in: query
      required: false
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 25
      description: Number of items per page.

  schemas:
    # ── Enums ────────────────────────────────────────────────────────────

    Chain:
      type: string
      enum: [tron, arbitrum]
      description: Blockchain network

    Token:
      type: string
      enum: [USDT, USDC]
      description: Stablecoin token

    CheckoutStatus:
      type: string
      enum: [pending, detected, confirming, confirmed, expired, failed]
      description: |
        - `pending` — Waiting for payment
        - `detected` — Transaction seen on-chain, unconfirmed
        - `confirming` — Confirmations in progress
        - `confirmed` — Payment fully confirmed (terminal)
        - `expired` — Checkout TTL exceeded without payment (terminal)
        - `failed` — Unrecoverable error (terminal)

    EventType:
      type: string
      enum:
        - checkout.created
        - checkout.payment_detected
        - checkout.confirming
        - checkout.completed
        - checkout.expired
        - checkout.failed

    # ── Request Bodies ───────────────────────────────────────────────────

    CreateCheckoutRequest:
      type: object
      required: [amount_usd, chain, token]
      properties:
        amount_usd:
          type: number
          format: double
          minimum: 0.01
          description: Amount in USD
          example: 49.99
        chain:
          $ref: "#/components/schemas/Chain"
        token:
          $ref: "#/components/schemas/Token"
        expires_in_seconds:
          type: integer
          minimum: 300
          maximum: 86400
          default: 1800
          description: Checkout TTL in seconds (5 min to 24 hr)
        metadata:
          type: object
          additionalProperties:
            type: string
          maxProperties: 20
          description: Arbitrary key-value pairs (max 20 keys, max 500 chars per value)
          example:
            order_id: "ord_12345"
            customer_email: "user@example.com"

    CreateWebhookRequest:
      type: object
      required: [url, events]
      properties:
        url:
          type: string
          format: uri
          description: HTTPS endpoint to receive events
          example: "https://example.com/webhooks/billing"
        events:
          type: array
          items:
            $ref: "#/components/schemas/EventType"
          minItems: 1
          description: Event types to subscribe to
        description:
          type: string
          maxLength: 256
          description: Human-readable label

    # ── Response Bodies ──────────────────────────────────────────────────

    Checkout:
      type: object
      properties:
        checkout_id:
          type: string
          description: Unique identifier (prefixed `co_`)
          example: "co_1a2b3c4d5e"
        deposit_address:
          type: string
          description: Blockchain address to send funds to
        chain:
          $ref: "#/components/schemas/Chain"
        token:
          $ref: "#/components/schemas/Token"
        amount_usd:
          type: number
          format: double
          description: Original USD amount
        amount_atomic:
          type: string
          description: Token amount in smallest unit (string to avoid precision loss)
          example: "49990000"
        status:
          $ref: "#/components/schemas/CheckoutStatus"
        tx_hash:
          type: string
          nullable: true
          description: On-chain transaction hash (null until detected)
        confirmations:
          type: integer
          description: Current confirmation count
        required_confirmations:
          type: integer
          description: Confirmations needed for this chain
          example: 19
        expires_at:
          type: string
          format: date-time
        detected_at:
          type: string
          format: date-time
          nullable: true
        confirmed_at:
          type: string
          format: date-time
          nullable: true
        created_at:
          type: string
          format: date-time
        metadata:
          type: object
          additionalProperties:
            type: string

    CheckoutStatusResponse:
      type: object
      properties:
        checkout_id:
          type: string
        status:
          $ref: "#/components/schemas/CheckoutStatus"
        tx_hash:
          type: string
          nullable: true
        confirmations:
          type: integer
        required_confirmations:
          type: integer
        detected_at:
          type: string
          format: date-time
          nullable: true
        confirmed_at:
          type: string
          format: date-time
          nullable: true
        polling_interval_ms:
          type: integer
          description: Suggested polling interval in milliseconds
          example: 2000

    CheckoutList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/Checkout"
        has_more:
          type: boolean
        next_cursor:
          type: string
          nullable: true

    WebhookEndpoint:
      type: object
      properties:
        webhook_id:
          type: string
          description: Unique identifier (prefixed `we_`)
        url:
          type: string
          format: uri
        events:
          type: array
          items:
            $ref: "#/components/schemas/EventType"
        secret:
          type: string
          description: |
            HMAC signing secret (prefixed `whsec_`).
            Only returned on creation — store it securely.
        description:
          type: string
          nullable: true
        status:
          type: string
          enum: [active, disabled]
        created_at:
          type: string
          format: date-time

    WebhookEndpointList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/WebhookEndpoint"
        has_more:
          type: boolean
        next_cursor:
          type: string
          nullable: true

    Event:
      type: object
      properties:
        event_id:
          type: string
          description: Unique identifier (prefixed `evt_`)
        type:
          $ref: "#/components/schemas/EventType"
        checkout_id:
          type: string
        data:
          $ref: "#/components/schemas/Checkout"
        created_at:
          type: string
          format: date-time

    EventList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/Event"
        has_more:
          type: boolean
        next_cursor:
          type: string
          nullable: true

    HealthResponse:
      type: object
      properties:
        status:
          type: string
          enum: [healthy]
        version:
          type: string
          example: "1.0.0"

    # ── Errors ───────────────────────────────────────────────────────────

    ErrorResponse:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [type, code, message]
          properties:
            type:
              type: string
              enum:
                - invalid_request
                - authentication_error
                - not_found
                - idempotency_conflict
                - rate_limited
                - internal_error
            code:
              type: string
              description: Machine-readable error code
              example: "checkout_not_found"
            message:
              type: string
              description: Human-readable message
              example: "No checkout found with ID co_abc123"
            param:
              type: string
              nullable: true
              description: The request parameter that caused the error

  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: invalid_request
              code: missing_required_field
              message: "The 'amount_usd' field is required."
              param: amount_usd

    Unauthorized:
      description: Authentication failed
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: authentication_error
              code: api_key_invalid
              message: "The API key provided is invalid."
              param: null

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: not_found
              code: checkout_not_found
              message: "No checkout found with ID co_abc123."
              param: checkout_id

    IdempotencyConflict:
      description: Idempotency key reused with different parameters
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: idempotency_conflict
              code: idempotency_key_reused
              message: "This idempotency key was already used with different parameters."
              param: null

    RateLimited:
      description: Rate limit exceeded
      headers:
        X-RateLimit-Limit:
          schema:
            type: integer
          description: Requests allowed per window
        X-RateLimit-Remaining:
          schema:
            type: integer
          description: Requests remaining in current window
        X-RateLimit-Reset:
          schema:
            type: integer
          description: Unix timestamp when the window resets
        Retry-After:
          schema:
            type: integer
          description: Seconds to wait before retrying
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: rate_limited
              code: too_many_requests
              message: "Rate limit exceeded. Retry after 30 seconds."
              param: null

    InternalError:
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ErrorResponse"
          example:
            error:
              type: internal_error
              code: internal_server_error
              message: "An unexpected error occurred."
              param: null
```

**Step 2: Validate the spec syntax**

Run: `npx @redocly/cli lint docs/openapi/v1.yaml`
Expected: 0 errors (warnings acceptable for first pass)

**Step 3: Commit**

```bash
git add docs/openapi/v1.yaml
git commit -m "feat: add canonical v1 OpenAPI 3.0 spec

Defines the full /v1 API surface for SDK generation:
- Checkouts CRUD with idempotency
- Webhook endpoints CRUD with HMAC signing
- Events listing for replay/debug
- Structured error taxonomy
- Cursor-based pagination
- Bearer auth (sk_live_ / sk_test_)
- Rate limit headers"
```

---

## Task 2: Remove `underpaid` from API Types

**Files:**
- Modify: `src/lib/api.ts:8-15` (PaymentState type)

**Step 1: Edit PaymentState to remove `underpaid`**

In `src/lib/api.ts`, change:

```typescript
export type PaymentState =
  | "paying"
  | "detected"
  | "confirming"
  | "success"
  | "expired"
  | "underpaid"
  | "error";
```

To:

```typescript
export type PaymentState =
  | "paying"
  | "detected"
  | "confirming"
  | "success"
  | "expired"
  | "error";
```

**Step 2: Verify no code references `underpaid`**

Run: `grep -rn "underpaid" src/`

Expected: Only hits in `src/app/variations/` files (UI-only) and `src/app/checkout/[id]/page.tsx`.

For any checkout page references to `underpaid`, these are UI display states rendered from local logic — they do not originate from the API. Leave them as hardcoded UI strings if present, but they must not appear in API type definitions.

**Step 3: Commit**

```bash
git add src/lib/api.ts
git commit -m "fix: remove underpaid from API PaymentState type

underpaid is a UI-only derived condition, not a backend state.
The Chain Watcher API never produces this status."
```

---

## Task 3: Update API Client Types to Match OpenAPI Spec

**Files:**
- Modify: `src/lib/api.ts` (full rewrite of types section)

**Step 1: Replace the types section of `src/lib/api.ts`**

Replace lines 1-81 (everything above `class ApiError`) with the following types that mirror the OpenAPI spec exactly:

```typescript
// billing.io v1 API Client
// Types mirror docs/openapi/v1.yaml — do not diverge.

const BASE_URL =
  process.env.NEXT_PUBLIC_CHAIN_WATCHER_URL ?? "http://localhost:8080";

// ─── Enums ──────────────────────────────────────────────────────────────────

export type Chain = "tron" | "arbitrum";
export type Token = "USDT" | "USDC";

export type CheckoutStatus =
  | "pending"
  | "detected"
  | "confirming"
  | "confirmed"
  | "expired"
  | "failed";

export type EventType =
  | "checkout.created"
  | "checkout.payment_detected"
  | "checkout.confirming"
  | "checkout.completed"
  | "checkout.expired"
  | "checkout.failed";

// ─── UI Display State (not an API type) ─────────────────────────────────────

/** Client-side display names. Maps API statuses to user-friendly labels. */
export type PaymentDisplayState =
  | "paying"
  | "detected"
  | "confirming"
  | "success"
  | "expired"
  | "error";

const DISPLAY_STATE_MAP: Record<CheckoutStatus, PaymentDisplayState> = {
  pending: "paying",
  detected: "detected",
  confirming: "confirming",
  confirmed: "success",
  expired: "expired",
  failed: "error",
};

export function toDisplayState(status: CheckoutStatus): PaymentDisplayState {
  return DISPLAY_STATE_MAP[status] ?? "error";
}

// ─── Request Types ──────────────────────────────────────────────────────────

export interface CreateCheckoutRequest {
  amount_usd: number;
  chain: Chain;
  token: Token;
  expires_in_seconds?: number;
  metadata?: Record<string, string>;
}

export interface CreateWebhookRequest {
  url: string;
  events: EventType[];
  description?: string;
}

export interface PaginationParams {
  cursor?: string;
  limit?: number;
}

// ─── Response Types ─────────────────────────────────────────────────────────

export interface Checkout {
  checkout_id: string;
  deposit_address: string;
  chain: Chain;
  token: Token;
  amount_usd: number;
  amount_atomic: string;
  status: CheckoutStatus;
  tx_hash: string | null;
  confirmations: number;
  required_confirmations: number;
  expires_at: string;
  detected_at: string | null;
  confirmed_at: string | null;
  created_at: string;
  metadata?: Record<string, string>;
}

export interface CheckoutStatusResponse {
  checkout_id: string;
  status: CheckoutStatus;
  tx_hash: string | null;
  confirmations: number;
  required_confirmations: number;
  detected_at: string | null;
  confirmed_at: string | null;
  polling_interval_ms: number;
}

export interface PaginatedList<T> {
  data: T[];
  has_more: boolean;
  next_cursor: string | null;
}

export type CheckoutList = PaginatedList<Checkout>;

export interface WebhookEndpoint {
  webhook_id: string;
  url: string;
  events: EventType[];
  secret?: string; // Only present on creation response
  description: string | null;
  status: "active" | "disabled";
  created_at: string;
}

export type WebhookEndpointList = PaginatedList<WebhookEndpoint>;

export interface Event {
  event_id: string;
  type: EventType;
  checkout_id: string;
  data: Checkout;
  created_at: string;
}

export type EventList = PaginatedList<Event>;

export interface HealthResponse {
  status: "healthy";
  version: string;
}

// ─── Error Types ────────────────────────────────────────────────────────────

export type ErrorType =
  | "invalid_request"
  | "authentication_error"
  | "not_found"
  | "idempotency_conflict"
  | "rate_limited"
  | "internal_error";

export interface ApiErrorBody {
  error: {
    type: ErrorType;
    code: string;
    message: string;
    param?: string | null;
  };
}
```

**Step 2: Verify the file compiles**

Run: `npx tsc --noEmit`
Expected: Type errors in files that reference old types (e.g., `PaymentState`, `ChainWatcherStatus`, `mapStatus`). These will be fixed in the next tasks.

**Step 3: Commit**

```bash
git add src/lib/api.ts
git commit -m "feat: update API client types to match v1 OpenAPI spec

- Replace ChainWatcherStatus with CheckoutStatus enum
- Replace PaymentState with PaymentDisplayState (UI-only)
- Add Chain, Token, EventType enums
- Add Checkout with required_confirmations field
- Add CheckoutStatusResponse with polling_interval_ms
- Add PaginatedList generic for cursor pagination
- Add WebhookEndpoint, Event types
- Add structured ApiErrorBody type
- Rename mapStatus to toDisplayState"
```

---

## Task 4: Update API Client Functions

**Files:**
- Modify: `src/lib/api.ts` (functions section — everything below types)

**Step 1: Replace the API functions section**

Replace the `ApiError` class, `request()` function, and all exported functions with:

```typescript
// ─── API Error ──────────────────────────────────────────────────────────────

export class ApiError extends Error {
  public readonly type: ErrorType;
  public readonly code: string;
  public readonly statusCode: number;
  public readonly param: string | null;

  constructor(statusCode: number, body: ApiErrorBody) {
    super(body.error.message);
    this.name = "ApiError";
    this.statusCode = statusCode;
    this.type = body.error.type;
    this.code = body.error.code;
    this.param = body.error.param ?? null;
  }
}

// ─── HTTP Client ────────────────────────────────────────────────────────────

function getAuthHeaders(): Record<string, string> {
  const apiKey = typeof window !== "undefined"
    ? (window as Record<string, unknown>).__BILLING_API_KEY__ as string | undefined
    : process.env.BILLING_API_SECRET_KEY;
  if (!apiKey) return {};
  return { Authorization: `Bearer ${apiKey}` };
}

async function request<T>(
  path: string,
  options?: RequestInit & { idempotencyKey?: string },
): Promise<T> {
  const url = `${BASE_URL}${path}`;
  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    ...getAuthHeaders(),
  };

  if (options?.idempotencyKey) {
    headers["Idempotency-Key"] = options.idempotencyKey;
  }

  const { idempotencyKey: _, ...fetchOptions } = options ?? {};

  const res = await fetch(url, {
    ...fetchOptions,
    headers: {
      ...headers,
      ...(fetchOptions.headers as Record<string, string> | undefined),
    },
  });

  if (!res.ok) {
    let body: ApiErrorBody;
    try {
      body = await res.json();
    } catch {
      body = {
        error: {
          type: "internal_error",
          code: "internal_server_error",
          message: await res.text().catch(() => "Unknown error"),
        },
      };
    }
    throw new ApiError(res.status, body);
  }

  if (res.status === 204) return undefined as T;
  return res.json() as Promise<T>;
}

function buildQuery(params: Record<string, string | number | undefined>): string {
  const entries = Object.entries(params).filter(
    ([, v]) => v !== undefined,
  );
  if (entries.length === 0) return "";
  return "?" + new URLSearchParams(
    entries.map(([k, v]) => [k, String(v)]),
  ).toString();
}

// ─── Checkout Endpoints ─────────────────────────────────────────────────────

export async function createCheckout(
  data: CreateCheckoutRequest,
  options?: { idempotencyKey?: string },
): Promise<Checkout> {
  return request<Checkout>("/v1/checkouts", {
    method: "POST",
    body: JSON.stringify(data),
    idempotencyKey: options?.idempotencyKey,
  });
}

export async function listCheckouts(
  params?: PaginationParams & { status?: CheckoutStatus },
): Promise<CheckoutList> {
  const query = buildQuery({
    cursor: params?.cursor,
    limit: params?.limit,
    status: params?.status,
  });
  return request<CheckoutList>(`/v1/checkouts${query}`);
}

export async function getCheckout(id: string): Promise<Checkout> {
  return request<Checkout>(`/v1/checkouts/${encodeURIComponent(id)}`);
}

export async function getCheckoutStatus(
  id: string,
): Promise<CheckoutStatusResponse> {
  return request<CheckoutStatusResponse>(
    `/v1/checkouts/${encodeURIComponent(id)}/status`,
  );
}

// ─── Webhook Endpoints ──────────────────────────────────────────────────────

export async function createWebhookEndpoint(
  data: CreateWebhookRequest,
): Promise<WebhookEndpoint> {
  return request<WebhookEndpoint>("/v1/webhooks", {
    method: "POST",
    body: JSON.stringify(data),
  });
}

export async function listWebhookEndpoints(
  params?: PaginationParams,
): Promise<WebhookEndpointList> {
  const query = buildQuery({
    cursor: params?.cursor,
    limit: params?.limit,
  });
  return request<WebhookEndpointList>(`/v1/webhooks${query}`);
}

export async function getWebhookEndpoint(
  id: string,
): Promise<WebhookEndpoint> {
  return request<WebhookEndpoint>(`/v1/webhooks/${encodeURIComponent(id)}`);
}

export async function deleteWebhookEndpoint(id: string): Promise<void> {
  return request<void>(
    `/v1/webhooks/${encodeURIComponent(id)}`,
    { method: "DELETE" },
  );
}

// ─── Event Endpoints ────────────────────────────────────────────────────────

export async function listEvents(
  params?: PaginationParams & { type?: EventType; checkout_id?: string },
): Promise<EventList> {
  const query = buildQuery({
    cursor: params?.cursor,
    limit: params?.limit,
    type: params?.type,
    checkout_id: params?.checkout_id,
  });
  return request<EventList>(`/v1/events${query}`);
}

export async function getEvent(id: string): Promise<Event> {
  return request<Event>(`/v1/events/${encodeURIComponent(id)}`);
}

// ─── Health ─────────────────────────────────────────────────────────────────

export async function getHealth(): Promise<HealthResponse> {
  return request<HealthResponse>("/v1/health");
}
```

**Step 2: Verify no TypeScript compilation errors in api.ts itself**

Run: `npx tsc --noEmit 2>&1 | head -30`
Expected: Errors will come from files that consume the old API (e.g., `use-checkout-polling.ts`, checkout pages). The api.ts file itself should be clean.

**Step 3: Commit**

```bash
git add src/lib/api.ts
git commit -m "feat: update API client functions to v1 spec

- All endpoints now prefixed with /v1/
- Add Authorization header via getAuthHeaders()
- Add Idempotency-Key header support on createCheckout
- Structured error parsing into ApiError class
- Add listCheckouts with cursor pagination
- Add webhook CRUD: create, list, get, delete
- Add event list/get endpoints
- Add health endpoint
- Add buildQuery helper for query string construction"
```

---

## Task 5: Update Polling Hook to Use Server-Provided Values

**Files:**
- Modify: `src/hooks/use-checkout-polling.ts`

**Step 1: Rewrite the polling hook**

Replace the entire file content with:

```typescript
"use client";

import { useState, useEffect, useRef, useCallback } from "react";
import {
  getCheckoutStatus,
  toDisplayState,
  type PaymentDisplayState,
} from "@/lib/api";

interface UseCheckoutPollingResult {
  status: PaymentDisplayState;
  confirmations: number;
  requiredConfirmations: number;
  txHash: string | null;
  error: string | null;
  isPolling: boolean;
}

const TERMINAL_STATES: PaymentDisplayState[] = ["success", "expired", "error"];
const DEFAULT_INTERVAL_MS = 3000;
const ERROR_RETRY_MS = 5000;

export function useCheckoutPolling(
  checkoutId: string | null,
): UseCheckoutPollingResult {
  const [status, setStatus] = useState<PaymentDisplayState>("paying");
  const [confirmations, setConfirmations] = useState(0);
  const [requiredConfirmations, setRequiredConfirmations] = useState(0);
  const [txHash, setTxHash] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isPolling, setIsPolling] = useState(false);
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);
  const mountedRef = useRef(true);

  const poll = useCallback(async () => {
    if (!checkoutId || !mountedRef.current) return;

    try {
      const data = await getCheckoutStatus(checkoutId);
      if (!mountedRef.current) return;

      const mapped = toDisplayState(data.status);
      setStatus(mapped);
      setConfirmations(data.confirmations);
      setRequiredConfirmations(data.required_confirmations);
      setTxHash(data.tx_hash);
      setError(null);

      if (!TERMINAL_STATES.includes(mapped)) {
        const interval = data.polling_interval_ms ?? DEFAULT_INTERVAL_MS;
        timeoutRef.current = setTimeout(poll, interval);
      } else {
        setIsPolling(false);
      }
    } catch (err) {
      if (!mountedRef.current) return;
      const message =
        err instanceof Error ? err.message : "Failed to fetch status";
      setError(message);
      timeoutRef.current = setTimeout(poll, ERROR_RETRY_MS);
    }
  }, [checkoutId]);

  useEffect(() => {
    mountedRef.current = true;

    if (!checkoutId) {
      setIsPolling(false);
      return;
    }

    setIsPolling(true);
    poll();

    return () => {
      mountedRef.current = false;
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, [checkoutId, poll]);

  return { status, confirmations, requiredConfirmations, txHash, error, isPolling };
}
```

Key changes:
- Uses `toDisplayState` instead of `mapStatus`
- Uses `PaymentDisplayState` instead of `PaymentState`
- Uses server-provided `polling_interval_ms` instead of hardcoded intervals
- Exposes `requiredConfirmations` from the API (no more hardcoded 19/12)

**Step 2: Verify the hook compiles**

Run: `npx tsc --noEmit 2>&1 | grep use-checkout-polling`
Expected: No errors from this file. Errors may remain in consumer files.

**Step 3: Commit**

```bash
git add src/hooks/use-checkout-polling.ts
git commit -m "feat: update polling hook to use server-provided intervals

- Use polling_interval_ms from API instead of hardcoded 2s/3s
- Use required_confirmations from API instead of hardcoded 19/12
- Rename PaymentState to PaymentDisplayState
- Rename mapStatus to toDisplayState"
```

---

## Task 6: Add Webhook Signature Verification Helper

**Files:**
- Create: `src/lib/webhook.ts`

**Step 1: Write the webhook verification module**

Create `src/lib/webhook.ts`:

```typescript
// Webhook signature verification for billing.io
// Used by merchants receiving webhook events.

const SIGNATURE_HEADER = "x-billing-signature";
const TOLERANCE_SECONDS = 300; // 5 minutes

export interface WebhookEvent {
  event_id: string;
  type: string;
  checkout_id: string;
  data: Record<string, unknown>;
  created_at: string;
}

export class WebhookVerificationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "WebhookVerificationError";
  }
}

/**
 * Parse the X-Billing-Signature header.
 * Format: t={timestamp},v1={hex_signature}
 */
function parseSignatureHeader(
  header: string,
): { timestamp: number; signature: string } {
  const parts: Record<string, string> = {};
  for (const segment of header.split(",")) {
    const [key, ...rest] = segment.split("=");
    parts[key.trim()] = rest.join("=").trim();
  }

  const timestamp = parseInt(parts.t, 10);
  const signature = parts.v1;

  if (isNaN(timestamp) || !signature) {
    throw new WebhookVerificationError(
      "Invalid signature header format. Expected: t={timestamp},v1={signature}",
    );
  }

  return { timestamp, signature };
}

/**
 * Compute HMAC-SHA256 for webhook verification.
 * Works in both Node.js and Edge Runtime (Web Crypto API).
 */
async function computeHmac(
  secret: string,
  payload: string,
): Promise<string> {
  // Use Web Crypto API (works in Edge Runtime, browsers, and Node 18+)
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    "raw",
    encoder.encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"],
  );
  const sig = await crypto.subtle.sign("HMAC", key, encoder.encode(payload));
  return Array.from(new Uint8Array(sig))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

/**
 * Constant-time string comparison to prevent timing attacks.
 */
function secureCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let mismatch = 0;
  for (let i = 0; i < a.length; i++) {
    mismatch |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return mismatch === 0;
}

/**
 * Verify a webhook signature and parse the event payload.
 *
 * @param rawBody - The raw request body string (do NOT parse JSON first)
 * @param signatureHeader - The value of the X-Billing-Signature header
 * @param secret - Your webhook endpoint secret (whsec_...)
 * @param toleranceSeconds - Max age of the event in seconds (default: 300)
 * @returns The parsed webhook event
 * @throws WebhookVerificationError if verification fails
 *
 * @example
 * ```typescript
 * import { verifyWebhookSignature } from "@/lib/webhook";
 *
 * export async function POST(req: Request) {
 *   const body = await req.text();
 *   const sig = req.headers.get("x-billing-signature")!;
 *
 *   try {
 *     const event = await verifyWebhookSignature(body, sig, process.env.WEBHOOK_SECRET!);
 *     switch (event.type) {
 *       case "checkout.completed":
 *         // Handle successful payment
 *         break;
 *     }
 *     return new Response("ok", { status: 200 });
 *   } catch (err) {
 *     return new Response("Invalid signature", { status: 400 });
 *   }
 * }
 * ```
 */
export async function verifyWebhookSignature(
  rawBody: string,
  signatureHeader: string,
  secret: string,
  toleranceSeconds: number = TOLERANCE_SECONDS,
): Promise<WebhookEvent> {
  if (!signatureHeader) {
    throw new WebhookVerificationError("Missing signature header");
  }

  if (!secret) {
    throw new WebhookVerificationError("Missing webhook secret");
  }

  const { timestamp, signature } = parseSignatureHeader(signatureHeader);

  // Check timestamp tolerance
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > toleranceSeconds) {
    throw new WebhookVerificationError(
      `Timestamp outside tolerance. Event: ${timestamp}, now: ${now}, tolerance: ${toleranceSeconds}s`,
    );
  }

  // Compute expected signature
  const signedPayload = `${timestamp}.${rawBody}`;
  const expected = await computeHmac(secret, signedPayload);

  if (!secureCompare(expected, signature)) {
    throw new WebhookVerificationError("Signature mismatch");
  }

  // Parse and return the event
  try {
    return JSON.parse(rawBody) as WebhookEvent;
  } catch {
    throw new WebhookVerificationError("Invalid JSON in webhook body");
  }
}

export { SIGNATURE_HEADER };
```

**Step 2: Verify the file compiles**

Run: `npx tsc --noEmit 2>&1 | grep webhook`
Expected: No errors from this file.

**Step 3: Commit**

```bash
git add src/lib/webhook.ts
git commit -m "feat: add webhook signature verification helper

- HMAC-SHA256 verification using Web Crypto API
- Constant-time comparison to prevent timing attacks
- Timestamp tolerance check (5 minute window)
- Parses X-Billing-Signature header (t=...,v1=... format)
- Works in Node.js, Edge Runtime, and browsers
- Includes JSDoc example for Next.js route handler"
```

---

## Task 7: Fix Consumer Files (Checkout Pages, Dashboard)

**Files:**
- Modify: `src/app/checkout/[id]/page.tsx` (update type references)
- Modify: `src/app/dashboard/checkouts/new/page.tsx` (update createCheckout call)
- Modify: `src/app/dashboard/checkouts/page.tsx` (update CheckoutRow type)
- Modify: `src/app/dashboard/settings/page.tsx` (update health check)

**Step 1: Update all consumer files**

For each file, apply these renames:
- `PaymentState` -> `PaymentDisplayState`
- `mapStatus` -> `toDisplayState`
- `ChainWatcherStatus` -> `CheckoutStatus`
- Any hardcoded confirmation counts (19, 12) -> use `requiredConfirmations` from the polling hook or checkout object

In `src/app/checkout/[id]/page.tsx`:
- Import `PaymentDisplayState` and `toDisplayState` instead of `PaymentState` and `mapStatus`
- Use `requiredConfirmations` from the `useCheckoutPolling` hook instead of hardcoded values
- Remove the local confirmation count logic (`chain === "tron" ? 19 : 12`)

In `src/app/dashboard/checkouts/page.tsx`:
- Update `CheckoutRow.status` type to use `CheckoutStatus` enum values

In `src/app/dashboard/settings/page.tsx`:
- Update the health check to call `getHealth()` from the API client instead of raw fetch

**Step 2: Verify the entire project compiles**

Run: `npx tsc --noEmit`
Expected: 0 errors

**Step 3: Verify the dev server starts**

Run: `npm run build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add src/app/checkout/ src/app/dashboard/
git commit -m "fix: update consumer files to v1 API types

- Rename PaymentState to PaymentDisplayState in all pages
- Rename mapStatus to toDisplayState
- Use requiredConfirmations from API instead of hardcoded values
- Update status enum references to CheckoutStatus"
```

---

## Task 8: Validate OpenAPI Spec with Redocly

**Files:**
- Modify: `package.json` (add dev dependency)

**Step 1: Install Redocly CLI**

Run: `npm install --save-dev @redocly/cli`

**Step 2: Lint the spec**

Run: `npx @redocly/cli lint docs/openapi/v1.yaml`
Expected: 0 errors. Fix any that arise.

**Step 3: Generate a preview**

Run: `npx @redocly/cli preview docs/openapi/v1.yaml`

Open in browser and visually confirm:
- All 12 endpoints render correctly
- Request/response examples are present
- Error responses are documented
- Auth scheme shows in the header
- Webhook signing scheme is described

**Step 4: Commit**

```bash
git add package.json package-lock.json
git commit -m "chore: add @redocly/cli for OpenAPI spec validation"
```

---

## Summary: All Endpoints in v1

| # | Method | Path | Auth | Idempotent | Paginated |
|---|---|---|---|---|---|
| 1 | GET | /v1/health | No | Yes | No |
| 2 | POST | /v1/checkouts | Yes | Via header | No |
| 3 | GET | /v1/checkouts | Yes | Yes | Cursor |
| 4 | GET | /v1/checkouts/{id} | Yes | Yes | No |
| 5 | GET | /v1/checkouts/{id}/status | Yes | Yes | No |
| 6 | POST | /v1/webhooks | Yes | No | No |
| 7 | GET | /v1/webhooks | Yes | Yes | Cursor |
| 8 | GET | /v1/webhooks/{id} | Yes | Yes | No |
| 9 | DELETE | /v1/webhooks/{id} | Yes | Yes | No |
| 10 | GET | /v1/events | Yes | Yes | Cursor |
| 11 | GET | /v1/events/{id} | Yes | Yes | No |

## Next Steps (out of scope for this plan)

After this plan is complete, the following are unblocked:
1. Generate SDKs from `docs/openapi/v1.yaml` using openapi-generator
2. Implement the Chain Watcher backend endpoints to match the spec
3. Implement webhook dispatch + retry in the Chain Watcher
4. Build the API key management UI in the dashboard
5. Build the webhook management UI in the dashboard
