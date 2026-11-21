# Extended PiRC capability integration map

**Profile:** PiRC-to-x402 interoperability 0.2.0 (documentation proposal)
**Target:** x402 v2 exact scheme; Pi Testnet first
**Status:** Design guidance only. No runtime adapter, live payment, oracle gateway, or contract deployment is claimed.

## 1. Why this extension exists

The initial profile mapped payment intent, payment event, provenance, valuation, oracle observations, economic settlement, implementation evidence, and the seven-layer registry. The current PiRC schema registry and release standards expose additional useful capabilities: product valuation, append-only commerce events, V18 metrics reports, QR/NFC payment requests, subscriptions, and explicit release gates.

These capabilities belong around the payment protocol, not inside the exact-scheme authorization primitive. x402 remains the boundary for expressing and verifying a buyer-authorized payment. The merchant application owns catalogue, reservation, order, subscription, fulfilment, refunds, and analytics state.

## 2. Extended capability map

| PiRC capability | Candidate integration | Required separation |
|---|---|---|
| Product valuation — `schemas/v1/product-valuation.schema.json` | Merchant catalogue and quote service | A valuation is not an x402 amount until the merchant issues a current, buyer-visible quote in the selected asset. |
| Commerce event — `schemas/v1/commerce-event.schema.json` | Append-only order lifecycle/event projection | Event schema validity is not proof that a payment settled or goods were delivered. |
| V18 metrics report — `schemas/v1/v18-metrics-report.schema.json` | Analytics and operational dashboard | Metrics must be derived from reconciled, deduplicated events and disclose window, denominator, exclusions and source coverage. |
| QR/NFC payment intent | Checkout handoff to a merchant payment requirement | QR/NFC transports a request only; it must not contain seeds, private keys, reusable bearer credentials or an implicit approval. |
| Subscription/payment event | Merchant billing scheduler and payment-event consumer | Each billing period creates a fresh, bounded authorization and idempotency identity; no silent recurring charge is implied. |
| Four economic gateways | Read-only quote/risk/eligibility context | Gateway output cannot override buyer-approved amount or authorize settlement by itself. |
| Testnet1 release gates | CI/release policy and operator checklist | A local pass or Testnet rehearsal never authorizes Mainnet promotion. |

## 3. Quote lifecycle and amount integrity

Use a typed quote object in the merchant application with, at minimum:

- immutable quote identifier and version;
- merchant and order identifiers;
- product/service identifiers and quantity;
- quoted asset identifier and exact network identifier;
- integer atomic-unit amount encoded as a canonical decimal string;
- asset decimals sourced from the verified asset definition;
- created-at and expires-at timestamps;
- pricing policy/version and optional valuation evidence references;
- buyer-facing display amount and any fiat reference clearly labelled as informational.

The adapter may construct x402 `PaymentRequirements` only after validating the quote, checking that it is current, resolving the asset against a network-specific allowlist, and binding the merchant recipient and order reference. It must not convert a fiat valuation or IPPR/QWF index into a payment amount without a merchant-issued quote and explicit buyer approval.

Reject expired quotes, mismatched order/merchant/recipient, unknown assets, unsupported precision, non-canonical integer strings, overflow, and attempts to mutate a quote after payment authorization. A changed price requires a new quote and a new buyer approval.

## 4. Commerce event model

Treat commerce events as an append-only business audit stream. Suggested lifecycle:

`CART_CREATED -> QUOTE_ISSUED -> ORDER_RESERVED -> PAYMENT_REQUIRED -> PAYMENT_PENDING -> PAYMENT_CONFIRMED -> FULFILMENT_PENDING -> FULFILLED`

Alternative terminal or exception states must be explicit: `CANCELLED`, `PAYMENT_EXPIRED`, `PAYMENT_FAILED`, `DISPUTED`, `REFUND_PENDING`, `REFUNDED`, `PARTIALLY_FULFILLED`.

Every event should carry a stable event ID, schema version, aggregate/order ID, event type, occurred-at timestamp, producer identity, correlation ID, and causation reference. Consumers must deduplicate by event ID and enforce legal state transitions. Out-of-order events are held for reconciliation rather than applied blindly.

A `PAYMENT_CONFIRMED` event requires an independently verified settlement receipt and chain read-back. A `FULFILLED` event requires merchant fulfilment evidence appropriate to the product/service. Neither event may be inferred from a client redirect, QR scan, valid signature alone, or an oracle quote.

## 5. QR/NFC and payment handoff

QR and NFC are untrusted transport channels. The payload should contain only a short-lived payment-request reference or a bounded, signed merchant intent. The receiving wallet or client must display merchant, network, asset, amount, expiry and order context before the user approves.

Security requirements:

1. Use short expiry and a cryptographically random nonce/request ID.
2. Bind the request to merchant, order, recipient, asset, amount and network.
3. Reject replay, expired requests, duplicate order completion and cross-network substitution.
4. Resolve the request from a trusted merchant endpoint or verify its signature against a configured merchant key.
5. Do not put private keys, seed phrases, KYC records, or unrestricted personal data in QR/NFC payloads.
6. A scan or tap never constitutes payment authorization.
7. Persist the request state and idempotency key before processing; reconcile uncertain settlement outcomes before retry.

## 6. Subscription and recurring-payment boundary

A subscription is a merchant billing agreement, not permission for an x402 facilitator to charge a wallet indefinitely. Model the agreement separately from each payment:

- agreement ID, merchant, customer reference and product/plan;
- cadence and billing window;
- cancellation/effective dates and policy version;
- maximum amount or explicit repricing/renewal policy;
- notification and consent evidence references;
- per-period invoice/order ID and unique idempotency key.

Each billing period must produce a new payment request with current amount, asset, recipient, network and expiry. Require a fresh user authorization unless the wallet and protocol explicitly support a separately reviewed delegated authorization model. Cancellation blocks future requests but does not silently rewrite already submitted settlements. Retries for one period must reconcile the original transaction before creating another payment attempt.

## 7. Economic gateway contract

The four PiRC economic gateways are optional policy/analytics services, not payment authorities:

### 7.1 Market Data Gateway
Returns normalized observations, quorum result, quote ID, source coverage, freshness, policy version and expiry. Reject duplicate sources, stale/future observations, invalid units and out-of-policy deviation. A quote is informational and cannot alter an approved x402 amount.

### 7.2 Utility & Commerce Gateway
Returns verifiable catalogue, inventory, merchant and fulfilment evidence references. Distinguish active inventory from mock listings, cancelled orders, circular trades and unfulfilled claims. It can inform merchant risk and availability but cannot attest chain settlement.

### 7.3 Velocity & Liquidity Gateway
Returns declared measurement window, eligible settled volume, denominator, excluded event classes, venue/asset scope, depth, spread, slippage and concentration. Never treat GMV, pending payments, self-transfers or synthetic volume as collected revenue or accessible liquidity.

### 7.4 Solvency & Settlement Gateway
Returns reserve/liability evidence, applicable haircuts, exit-queue status, network identity and settlement read-back references. Missing, stale or conflicting evidence returns `REVIEW_REQUIRED` or `FAIL`; it must not silently degrade to PASS.

All gateway responses should be versioned and include request/correlation ID, policy version, observed-at, expires-at, evidence references and explicit status. Consumers fail closed on missing or stale critical fields. A gateway HTTP 200 is not equivalent to an economic gate PASS.

## 8. Metrics and accounting controls

Metrics must be reproducible from an identified event set and time window. At minimum distinguish:

- quote value, authorized amount, submitted amount and confirmed settled amount;
- gross merchandise value (GMV), collected revenue, refunds and chargebacks/disputes;
- unique buyers, completed orders, repeat purchases and fulfilment rate;
- eligible settled transaction value and the declared money-stock denominator used for velocity;
- same-asset/venue accessible liquidity, spread, slippage and concentration;
- reserve assets, liabilities, haircuts, coverage and exit queue;
- settlement latency and quote/oracle age.

Use integer atomic units and checked arithmetic for money. Define rounding rules explicitly for ratios and allocations. Enforce conservation: allocations plus retained remainder must equal the source amount exactly. Never use floating-point arithmetic for balances, prices, ratios used in authorization, or allocation totals.

## 9. Additional conformance cases

| ID | Scenario | Expected result |
|---|---|---|
| COM-001 | Quote is expired at payment-requirement construction | Reject; issue a new quote |
| COM-002 | Product valuation changes after buyer approval | Do not mutate approved amount; require new buyer approval |
| COM-003 | Quote merchant/order/recipient differs from request | Reject |
| EVT-001 | Duplicate commerce event ID | Idempotent no-op; no duplicate fulfilment |
| EVT-002 | Event arrives before its causal predecessor | Hold for reconciliation |
| EVT-003 | Payment-confirmed event lacks settlement read-back | Reject/quarantine |
| EVT-004 | Fulfilled event lacks required fulfilment evidence | Keep fulfilment pending |
| QRN-001 | QR/NFC request is expired, replayed or cross-network substituted | Reject |
| QRN-002 | QR/NFC payload contains secret or excess personal data | Reject/redact and raise security signal |
| SUB-001 | New billing period has no fresh authorization | Do not submit payment |
| SUB-002 | Same period is retried after RPC timeout | Reconcile original transaction; never double-charge |
| SUB-003 | Subscription cancelled before next billing period | Do not create next payment request |
| MET-001 | Metrics include pending, duplicate or circular transfers | Exclude and disclose exclusions |
| MET-002 | Allocation totals exceed or fail to conserve source amount | Reject report |
| GATE-001 | One required economic gateway is stale/unavailable | Aggregate status cannot be PASS |
| REL-001 | Testnet checks pass but Mainnet approval is absent | Block Mainnet release |

## 10. Release and evidence requirements

Before describing any capability as implemented, attach:

- source repository, branch and immutable commit SHA;
- schema version and compatibility result;
- unit/property/fuzz test commands and reproducible outputs;
- integration test run URL and sanitized logs;
- dependency and secret scanning evidence;
- artifact digest and build provenance;
- Testnet network identity and read-only probe evidence;
- for any approved write test: operator approval, dedicated Testnet signer reference, transaction hash, contract/event read-back and reconciliation result;
- reviewer disposition and known limitations.

Use `NOT_IMPLEMENTED`, `NOT_RUN`, `BLOCKED`, `UNVERIFIED` or `NOT_APPLICABLE` honestly. Never manufacture transaction hashes, contract IDs, oracle observations, test output or deployment status.

## 11. Source provenance

The design is derived from the community-maintained PiRC repository at the reviewed revision:

`Ze0ro99/PiRC@940bd54cc53e7138cbdb6a2d163a8dd13039f595`

Relevant paths include:

- `docs/standards/SCHEMA_REGISTRY.md`
- `docs/standards/TESTNET1_RELEASE_GATES.md`
- `docs/standards/PI_TESTNET1_NETWORK_PROFILE.md`
- `docs/standards/COMMERCE_VALUATION_AND_PROVENANCE_STANDARD.md`
- `docs/standards/DYNAMIC_IPPR_QWF_AND_FOUR_ECONOMIC_GATES.md`
- `docs/TESTNET1_REALITY_SIMULATION_AND_ACCEPTANCE.md`
- `schemas/v1/product-valuation.schema.json`
- `schemas/v1/commerce-event.schema.json`
- `schemas/v1/v18-metrics-report.schema.json`
- `schemas/v1/payment-intent.schema.json`
- `schemas/v1/payment-event.schema.json`

PiRC artefacts are community project specifications, not official Pi Network policy or x402 requirements unless adopted by their respective maintainers.
