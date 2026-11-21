# PiRC to x402 interoperability profile for `pinet`

**Profile version:** 0.1.0  
**Protocol boundary:** x402 v2, `exact` scheme  
**Network scope:** Pi Testnet first; Mainnet is a separate release decision  
**Status:** Experimental guidance; requires maintainer review

## 1. Design boundary

x402 remains responsible for expressing a payment requirement, carrying the payer's authorization payload, and coordinating verification and settlement. PiRC concepts may enrich merchant intent, asset metadata, commerce lifecycle, and audit evidence, but must not silently change the meaning of an x402 payment.

The profile is an adapter boundary, not a requirement that x402 import PiRC code or adopt PiRC governance.

## 2. Capability crosswalk

| PiRC capability / source | Appropriate x402 integration point | Required boundary |
|---|---|---|
| Payment intent — `schemas/v1/payment-intent.schema.json` | Merchant-side input used to construct x402 `PaymentRequirements` | Map `amount_atomic` to x402 `amount` only after validating asset, decimals, network, expiry, merchant and order binding. Never copy the PiRC signature as an x402 payment authorization. |
| Payment event — `schemas/v1/payment-event.schema.json` | Post-settlement event projection | Emit success only after chain read-back establishes the expected transaction, asset, amount, payer, recipient and finality policy. A schema-valid event is not proof of settlement. |
| Asset provenance — `schemas/v1/asset-provenance.schema.json` | Asset registry / allowlist metadata | Provenance is descriptive evidence. It does not prove ownership, token authenticity, issuer authority or transferability without independent verification. |
| Commerce valuation — `COMMERCE_VALUATION_AND_PROVENANCE_STANDARD.md` | Merchant quote and resource metadata | Keep product valuation, fiat reference, market quote, reserve value and payment amount distinct. An expired valuation requires a new payment requirement and buyer approval. |
| Oracle observation — `schemas/v1/oracle-observation.schema.json` | Optional merchant pricing input | Verify signer, source identity, sequence, freshness, quorum and units. Oracle output never overrides the buyer-approved x402 amount. |
| Dynamic IPPR / QWF / four economic gates — `DYNAMIC_IPPR_QWF_AND_FOUR_ECONOMIC_GATES.md` | Optional analytics / risk context outside settlement authorization | Reference-only Testnet economics. Never use IPPR/QWF as a settlement price, collateral proof, redemption guarantee or authorization to issue credit. |
| Economic settlement — `schemas/v1/economic-settlement.schema.json` | Downstream accounting / allocation projection | Keep accounting allocations separate from the atomic on-chain payment. Reconcile totals with checked integer arithmetic and idempotent period identifiers. |
| Implementation evidence — `schemas/v1/implementation-evidence.schema.json` | Release evidence and integration traceability | Record exact commit, interface, tests, artifact digest and security-review state. Evidence records are claims to verify, not attestations. |
| Seven-layer registry — `schemas/v1/seven-layer-registry.schema.json` and registry data | Optional application-level asset taxonomy | No implicit token support. Every asset still needs an explicit network-specific contract allowlist and verified interface. |
| DID / ZK identity, KYC and custody concepts | Separate identity or compliance service | Do not put personal data, KYC documents, secrets or private keys in x402 payloads. Do not make identity assertions without a verifiable issuer and policy. |
| AMM, lending, treasury, staking, governance and RWA modules | Separate application protocols | Out of scope for `exact` payment settlement. Require their own contract audits, authorization, risk controls and integration specifications. |

## 3. Canonical payment mapping

The merchant adapter may derive an x402 payment requirement from a validated merchant intent:

- `scheme`: `exact`
- `network`: the exact registered CAIP-2 network identifier, such as `pinet:testnet` or `pinet:mainnet`
- `asset`: a verified, network-specific SEP-41 contract identifier
- `amount`: positive base-10 integer string in the asset's atomic units
- `payTo`: verified recipient address
- `maxTimeoutSeconds`: bounded by the merchant's expiry and the authorization/network constraints

The adapter must reject ambiguous asset identifiers, unknown contracts, decimal or exponent notation, signed values, whitespace, zero, negative values, and values outside the supported Soroban integer range.

Pi native currency precision and any issued token precision must be derived from the verified asset definition. Do not assume all Pi ecosystem assets share the native Pi decimal scale.

## 4. Network identity and environment isolation

1. Resolve the CAIP-2 identifier using the canonical namespace registry.
2. Obtain the exact expected network passphrase from an authoritative Pi network source or trusted deployment configuration.
3. Query the configured Horizon and Soroban RPC endpoints.
4. Verify network identity, protocol compatibility and endpoint health before reads that inform authorization and again before settlement.
5. Reject endpoint/identifier/passphrase mismatches; do not infer a network from a hostname alone.
6. Keep Testnet and Mainnet configuration, contract allowlists, signer identities, secrets and release approvals separate.
7. Mainnet must remain disabled until a separately reviewed release process authorizes it.

PiRC's Testnet profile explicitly describes a Testnet-only target and fail-closed startup/deployment guards. Its project alias `pi-testnet1` must not be substituted for the CAIP-2 network identifier `pinet:testnet` unless an explicit, reviewed mapping defines that relationship.

## 5. Authorization and settlement invariants

The Pi exact-scheme implementation must preserve the upstream scheme's transaction and authorization rules. At minimum:

- Decode XDR with a maintained Stellar SDK and reject malformed or unsupported envelopes.
- Enforce the exact operation count, host-function type, contract, function name and argument count defined by the scheme.
- Bind payer, recipient, asset and amount to the accepted payment requirements.
- Reject unexpected operations, nested invocations, unsupported credential arms, unexpected signers and authorization entries that exceed the permitted ledger-time window.
- Verify every required authorization signature against the exact authorization payload and network context.
- Simulate at verification and settlement time; settlement must perform independent full verification.
- Derive fresh Soroban resource data and fee bounds from the settlement-time simulation; never trust client-supplied fee or resource limits.
- Verify expected transfer events and balance effects; reject extra or ambiguous asset movements.
- Persist an idempotency key and transaction identity before/while broadcasting; reconcile uncertain RPC outcomes before retrying.
- Do not report success or release a protected resource until settlement is confirmed under the configured finality policy.

A successful `/verify` response is not a settlement receipt. A timeout after broadcast is an unknown/pending outcome until reconciled, not permission to submit a second payment.

## 6. Commerce and fulfillment boundary

Merchant fulfillment must be a separate state transition from payment authorization:

`DRAFT -> LISTED -> RESERVED -> PAYMENT_PENDING -> PAID -> FULFILLED`

Alternative transitions must be explicit, authorized and idempotent (for example `CANCELLED`, `DISPUTED`, `REFUNDED`, `RESOLVED`). A client-side success screen, a valid payment intent, a valid signature, or an oracle quote must never trigger fulfillment by itself.

## 7. Privacy and evidence

- Keep personal data, KYC records, invoices and delivery details off-chain and out of x402 payment payloads.
- Store only the minimum pseudonymous references and commitments required for reconciliation or dispute handling.
- Redact authorization payloads, secrets and account-sensitive data from logs.
- Pin every referenced PiRC document to a commit SHA in implementation evidence.
- Record test commands, runtime versions, CI run URLs, artifact SHA-256 values and reviewer disposition.
- Use `NOT_RUN` or `UNVERIFIED` until evidence exists; do not infer success from a local simulation or schema validation.

## 8. PiRC source map

Reviewed source revision: `Ze0ro99/PiRC@940bd54cc53e7138cbdb6a2d163a8dd13039f595`.

Relevant source paths:

- `docs/standards/PI_TESTNET1_NETWORK_PROFILE.md`
- `docs/standards/SCHEMA_REGISTRY.md`
- `docs/standards/TESTNET1_RELEASE_GATES.md`
- `docs/standards/COMMERCE_VALUATION_AND_PROVENANCE_STANDARD.md`
- `docs/standards/DYNAMIC_IPPR_QWF_AND_FOUR_ECONOMIC_GATES.md`
- `docs/integration/EXTERNAL_COMPONENT_COMPATIBILITY_MATRIX.md`
- `schemas/v1/payment-intent.schema.json`
- `schemas/v1/payment-event.schema.json`
- `schemas/v1/economic-settlement.schema.json`
- `schemas/v1/asset-provenance.schema.json`
- `schemas/v1/oracle-observation.schema.json`
- `schemas/v1/implementation-evidence.schema.json`
- `schemas/v1/seven-layer-registry.schema.json`

These are PiRC community project artefacts. They are not official Pi Network policy or x402 protocol requirements unless separately adopted by the relevant maintainers.
