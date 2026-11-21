# Proposed PiRC economic gateway API contract

**Version:** 0.1.0 (proposal)
**Placement:** Merchant/application integration layer beside x402
**Status:** Interface design only. These endpoints are not asserted to exist in PiRC, Pi Network, or x402 upstream.

## 1. Contract principles

- Gateways provide evidence-backed context; they do not sign, authorize, submit, or settle x402 payments.
- A gateway response is never trusted solely because HTTP returned 2xx.
- Every response is versioned, time-bounded, traceable and explicit about evidence coverage.
- Missing, stale, conflicting or unverifiable critical evidence must fail closed.
- Monetary values use canonical base-10 integer strings in atomic units. Never use JSON floating-point values for money.
- All timestamps use RFC 3339 UTC. All asset and network identifiers are explicit.
- Responses are immutable snapshots identified by a response ID; corrections produce a new version rather than silently rewriting history.

## 2. Common request envelope

Every request should include:

| Field | Type | Requirement |
|---|---|---|
| `schema_version` | string | Required, e.g. `pirc.gateway.v1` |
| `request_id` | string | Required unique request identifier |
| `correlation_id` | string | Required trace identifier shared across checkout/order/payment |
| `requested_at` | RFC 3339 UTC | Required |
| `network_id` | string | Required; exact registered network identifier |
| `asset_id` | string | Required where monetary or liquidity data is returned |
| `policy_version` | string | Required policy identifier/version |
| `idempotency_key` | string | Required for state-changing or billable operations; optional for pure reads |

The caller must authenticate using a scoped service identity. Gateway credentials must not be wallet signing keys. Authorization is checked independently for each endpoint and tenant/merchant scope.

## 3. Common response envelope

Every response should contain:

| Field | Type | Meaning |
|---|---|---|
| `schema_version` | string | Response contract version |
| `response_id` | string | Immutable snapshot identifier |
| `request_id` / `correlation_id` | string | Echoed trace identifiers |
| `status` | enum | `PASS`, `FAIL`, `STALE`, `REVIEW_REQUIRED`, `NOT_APPLICABLE` |
| `observed_at` | RFC 3339 UTC | Time evidence was evaluated |
| `expires_at` | RFC 3339 UTC | Latest time the response may be relied upon |
| `network_id` | string | Network context, or explicit `NOT_APPLICABLE` where valid |
| `policy_version` | string | Policy used for evaluation |
| `evidence_refs` | array | Immutable references, digests and source timestamps |
| `reason_codes` | array | Machine-readable reasons for non-PASS status |
| `limitations` | array | Known coverage gaps and assumptions |

A response with missing required evidence, expired `expires_at`, a mismatched network/asset, or an unknown critical schema version must not be interpreted as PASS.

## 4. Gateway endpoints

### 4.1 Market Data Gateway

**Proposed endpoint:** `POST /v1/market/observations/evaluate`

Request fields:
- `asset_id`, `quote_currency`, `pair`;
- `observations[]`: source ID, venue ID, bid, ask, depth, volume, observed-at, sequence and signature/evidence reference;
- `freshness_limit_seconds`, `minimum_independent_sources`, `maximum_deviation_bps`;
- `policy_version`.

Response data:
- `quote_id`, normalized median/reference value and quote currency;
- source count and distinct venue count;
- accepted/rejected observation references and reason codes;
- oldest/newest observation times, deviation and expiry.

Reject duplicate source/venue identities where policy requires independence, future timestamps, stale observations, invalid units, non-positive values, invalid signatures, replayed sequences and insufficient quorum. Do not silently substitute a cached or hard-coded price.

### 4.2 Utility & Commerce Gateway

**Proposed endpoint:** `POST /v1/commerce/eligibility/evaluate`

Request fields:
- `merchant_id`, `order_id`, `product_ids[]`, quantities and asset/network context;
- inventory snapshot references;
- merchant verification and fulfilment evidence references;
- requested fulfilment window and policy version.

Response data:
- merchant/listing eligibility;
- available inventory by product and timestamp;
- evidence coverage and exclusions;
- order/fulfilment risk status and expiry.

Do not treat a catalogue listing as inventory proof. Exclude mock, cancelled, circular, self-dealing or unfulfilled activity according to policy. This gateway does not prove that a payment settled.

### 4.3 Velocity & Liquidity Gateway

**Proposed endpoint:** `POST /v1/economics/velocity-liquidity/evaluate`

Request fields:
- declared measurement window and timezone;
- asset, venue and market scope;
- eligible settlement-event references;
- money-stock denominator definition and source;
- exclusions and aggregation policy version.

Response data:
- eligible settled volume and denominator;
- computed velocity plus formula/version;
- event counts and exclusions;
- depth, spread, slippage and concentration by asset/venue;
- coverage limitations and expiry.

A report is invalid if its denominator is undefined, event population cannot be reproduced, or pending/duplicate/self-transfer/circular events are included as eligible settled volume. GMV is not collected revenue.

### 4.4 Solvency & Settlement Gateway

**Proposed endpoint:** `POST /v1/solvency/settlement/evaluate`

Request fields:
- network and asset identifiers;
- reserve attestations and custody evidence references;
- liabilities and redemption/exit queue snapshots;
- approved haircuts, concentration limits and policy version;
- transaction identity/read-back references where settlement is evaluated.

Response data:
- eligible reserve value after haircuts;
- liabilities, coverage ratio and exit-queue metrics;
- custody/encumbrance status;
- chain transaction status, finality policy and read-back evidence;
- explicit eligibility decision and reasons.

Do not infer reserves from token supply, a public address, a README, a price quote or an unverified attestation. If liabilities are zero, coverage is `NOT_APPLICABLE`, not infinity. Unknown finality or missing custody evidence must not return PASS.

## 5. Aggregation policy

The consumer, not any individual gateway, computes the aggregate decision. A strict example policy is:

1. Validate response schema, signature/authenticity where used, request/correlation IDs and policy version.
2. Validate network and asset scope.
3. Validate observed-at/expiry and evidence references.
4. If any required gateway returns `FAIL`, aggregate is `FAIL`.
5. Else if any required gateway returns `STALE`, aggregate is `STALE`.
6. Else if any required gateway returns `REVIEW_REQUIRED`, aggregate is `REVIEW_REQUIRED`.
7. `NOT_APPLICABLE` is accepted only where the policy explicitly marks that gateway non-required.
8. Aggregate `PASS` is permitted only when every required gateway is fresh, valid and PASS.

This aggregate result is risk/commerce context only. It does not authorize a transfer, modify a buyer-approved amount, replace x402 verification, or prove settlement.

## 6. HTTP and error semantics

Use transport status and business status separately:

- `400`: malformed request or invalid canonical encoding.
- `401` / `403`: missing or insufficient service authorization.
- `404`: referenced resource/evidence does not exist.
- `409`: idempotency conflict or incompatible state transition.
- `422`: structurally valid request violates a business rule.
- `429`: rate limit; retry only according to bounded backoff.
- `503`: dependency unavailable; gateway status must not be promoted to PASS.

A 2xx response may still carry `FAIL`, `STALE` or `REVIEW_REQUIRED`. Clients must inspect the semantic status and reason codes.

## 7. Security and operational controls

- Authenticate service-to-service requests; use least-privilege scopes and rotate credentials.
- Sign canonical response payloads if cross-service tamper evidence is required; pin signer identity and key version.
- Protect against replay with request IDs, sequence numbers, timestamps and idempotency keys.
- Enforce bounded request sizes, pagination, timeouts, concurrency limits and rate limits.
- Do not log secrets, KYC documents, raw wallet seeds, private keys or full authorization payloads.
- Record provenance for every source observation and reserve attestation.
- Emit structured audit events for policy changes, key rotations and status overrides.
- A human-approved override must include actor, reason, expiry, scope and immutable audit reference; no override may bypass x402 signature or settlement verification.
- Monitor p50/p95/p99 latency, stale-response rate, quorum failures, dependency availability and evidence gaps.

## 8. Required contract tests

| ID | Test | Expected result |
|---|---|---|
| API-001 | Missing schema version or request ID | Reject 400 |
| API-002 | Network/asset scope mismatch | Reject or return non-PASS; never reuse evidence across networks |
| API-003 | Expired response reused | Reject as stale |
| API-004 | HTTP 200 with body status FAIL | Preserve FAIL |
| API-005 | Idempotency key reused with different request body | Reject 409 |
| API-006 | Unknown critical schema version | Reject or quarantine |
| API-007 | Required evidence reference unavailable | REVIEW_REQUIRED/FAIL, never PASS |
| API-008 | Duplicate source identities satisfy quorum count | Reject/quarantine |
| API-009 | Zero liabilities | Coverage NOT_APPLICABLE |
| API-010 | Gateway outage | STALE/REVIEW_REQUIRED; no aggregate PASS |
| API-011 | Manual override lacks actor/reason/expiry | Reject |
| API-012 | Gateway attempts to modify x402 amount | Reject at consumer boundary |

## 9. Adoption and compatibility

This is a proposed integration contract and should be implemented only after maintainers review naming, schemas, authentication, evidence semantics and versioning. Before implementation:

1. Map fields to the exact PiRC schemas at the pinned source revision.
2. Publish JSON Schema Draft 2020-12 request/response schemas under a new immutable version.
3. Generate typed client/server models from those schemas.
4. Add contract tests and compatibility fixtures.
5. Implement read-only adapters first.
6. Keep all live write capabilities outside these gateways and behind independent x402/network release gates.

No endpoint in this proposal should be described as deployed, supported by Pi Core Team, or available in a PiRC production service without independent implementation and deployment evidence.
