# PiRC interoperability profile

**Profile version:** 0.1.0  
**Protocol target:** x402 v2 exact scheme on a Pi Network CAIP-2 network  
**Status:** Design guidance; no implementation or conformance result is claimed.

## 1. Boundary

x402 remains responsible for payment requirements, buyer authorization, facilitator verification, settlement, and the decision to fulfill a protected resource.

PiRC concepts may provide optional context or evidence to an application. They must not override the buyer-approved x402 payment requirements, replace cryptographic authorization, or turn an unverified external claim into a successful payment.

The accepted Pi exact-scheme specification is the source of truth for Pi-specific payload, verification, and settlement behavior. Do not infer protocol fields or signing rules from this profile.

## 2. Concept mapping

| PiRC concept | Permitted integration use | Required guardrail |
|---|---|---|
| Payment Intent | Draft application intent before x402 requirements are issued | Validate network, asset, recipient, amount, expiry, and order binding against the accepted x402 requirements |
| Payment Event | Application-side record of a claimed payment outcome | Mark paid only after transaction lookup, transfer-event validation, and the configured finality/read-back policy |
| Asset Provenance | Supporting metadata for an asset registry or allowlist review | Provenance metadata alone does not prove contract authenticity, ownership, issuer authority, or payment support |
| Commerce Valuation | Merchant quote or catalogue context | The buyer-approved x402 amount is immutable for that authorization; repricing requires a new quote and approval |
| Oracle observations | Optional pricing or risk context outside payment authorization | Reject stale, replayed, conflicting, or unverifiable observations; never silently change an authorized amount |
| IPPR / QWF / economic gates | Optional application risk signals | A failed or missing gate may block an application workflow, but a passing gate is not a payment authorization or price guarantee |
| Economic Settlement | Off-chain accounting or allocation projection after verified payment | Use integer arithmetic, idempotent event handling, and conservation checks; do not confuse allocation with on-chain settlement |
| Implementation Evidence | Release and audit evidence references | Bind evidence to immutable commits, test runs, artifacts, and timestamps |
| Seven-layer registry | Optional asset classification metadata | No layer classification automatically enables an asset; require a network-specific, reviewed asset allowlist |
| DID / KYC / ZK / custody | Separate identity or compliance services where legally and operationally required | Minimize personal data; do not place credentials or sensitive identity data in x402 payment payloads or logs |

## 3. Payment invariants

1. **Network binding:** the CAIP-2 network identifier, RPC endpoint, and chain/network passphrase must agree. Unknown or mismatched networks fail closed.
2. **Asset binding:** the asset identifier must resolve to an explicitly configured contract on the selected network. Do not infer support from a symbol, layer label, or provenance record.
3. **Amount integrity:** represent monetary values as canonical base-10 integer strings in the asset's atomic units. Do not use floating-point arithmetic.
4. **Requirement binding:** verify the payload against the exact requirements accepted by the buyer, including amount, asset, recipient, and expiry.
5. **Independent settlement verification:** settlement must repeat all security-critical verification; it must not trust a previous `/verify` result as authorization to submit.
6. **Read-back before fulfillment:** a successful RPC submission or an HTTP 2xx response is not proof of settlement. Reconcile the transaction and validate the expected transfer before granting the resource.
7. **Replay safety:** persist a stable payment identity and make retries idempotent. A repeated identity with different payment data is a conflict.
8. **No silent repricing:** market data, oracle observations, inventory changes, and economic gates cannot mutate an already accepted payment amount.
9. **No implied guarantees:** a valuation, reserve statement, liquidity metric, or risk score is not a redemption guarantee, credit commitment, or proof of solvency.
10. **Mainnet separation:** Testnet evidence does not authorize Mainnet writes. Mainnet requires a separate release decision and operational controls.

## 4. Data and privacy

- Do not request, store, or log wallet seeds or private signing keys in the integration profile.
- Keep KYC documents, identity proofs, and personal data outside payment payloads unless a separately reviewed requirement makes a minimal reference strictly necessary.
- Redact credentials and personal data from logs, fixtures, screenshots, and CI artifacts.
- Treat all external registry, oracle, and gateway responses as untrusted input until schema, authenticity, scope, freshness, and policy checks pass.

## 5. Evidence and status

Use explicit statuses such as `NOT_IMPLEMENTED`, `NOT_RUN`, `PASS`, `FAIL`, `BLOCKED`, and `NOT_APPLICABLE`. A `PASS` must be reproducible from a pinned source revision and test evidence. A design document or illustrative fixture is not execution evidence.

## 6. Source attribution

PiRC is a community-maintained reference. Its documents and schemas are not represented here as official Pi Network or x402 Foundation standards. The source reviewed for this profile is pinned to `Ze0ro99/PiRC@940bd54cc53e7138cbdb6a2d163a8dd13039f595`.
