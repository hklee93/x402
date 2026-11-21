# Pi Network exact-scheme conformance matrix

**Profile:** PiRC interoperability profile 0.1.0  
**Target:** x402 v2 exact scheme on Pi Network  
**Status:** Test plan only. No case below is claimed to have run.

## Result vocabulary

- `NOT_IMPLEMENTED`: required behavior is not implemented.
- `NOT_RUN`: no reproducible test evidence was inspected.
- `PASS`: test ran against the stated commit and met its expected result.
- `FAIL`: test ran and did not meet its expected result.
- `BLOCKED`: a required toolchain, fixture, network, or permission was unavailable.
- `NOT_APPLICABLE`: reason is documented and accepted by maintainers.

Every reported result must include the source commit, command, runtime/toolchain, CI URL, artifact digest, and timestamp. Network tests must include sanitized read-back evidence.

## A. Network and asset identity

| ID | Scenario | Expected result |
|---|---|---|
| NET-001 | Configured CAIP-2 network and RPC endpoint identify different networks | Reject before signing or settlement |
| NET-002 | Mainnet passphrase or endpoint is supplied to a Testnet configuration | Reject |
| NET-003 | Unknown network identifier | Reject |
| NET-004 | Asset contract is absent from the network-specific allowlist | Reject |
| NET-005 | Asset contract does not implement the required token interface | Reject |
| NET-006 | Testnet configuration is changed to Mainnet without separate release authorization | Block Mainnet writes |

## B. Amount and requirement binding

| ID | Scenario | Expected result |
|---|---|---|
| AMT-001 | Canonical positive base-10 integer amount within the verified asset range | Accept if all other requirements pass |
| AMT-002 | Zero, negative, whitespace, decimal, exponent, or non-numeric amount | Reject |
| AMT-003 | Amount exceeds the verified supported integer range | Reject without overflow |
| AMT-004 | Payload amount differs from the buyer-accepted requirement | Reject |
| AMT-005 | Payload recipient differs from the required recipient | Reject |
| AMT-006 | Payload asset differs from the required asset | Reject |
| AMT-007 | Quote or authorization has expired | Reject; require a new buyer-approved requirement |

## C. Transaction and authorization validation

| ID | Scenario | Expected result |
|---|---|---|
| AUTH-001 | Malformed or non-canonical transaction encoding | Reject without panic |
| AUTH-002 | Transaction contains an unexpected operation or invocation | Reject |
| AUTH-003 | Transfer function, contract, or arguments differ from the accepted payment | Reject |
| AUTH-004 | Authorization signature is invalid or bound to different data | Reject |
| AUTH-005 | Authorization has expired or exceeds the permitted time window | Reject |
| AUTH-006 | Unexpected signer, source, or authorization credential is present | Reject |

## D. Settlement, replay, and fulfillment

| ID | Scenario | Expected result |
|---|---|---|
| SET-001 | Settlement receives a payload not previously verified | Repeat all security-critical verification |
| SET-002 | Payload or requirements change between verification and settlement | Reject |
| SET-003 | Same payment identity is submitted again with identical data | Reconcile or return the existing outcome; never transfer twice |
| SET-004 | Same idempotency identity is reused with different payment data | Reject as conflict |
| SET-005 | RPC times out after broadcast | Record an unknown/pending outcome and reconcile before retry |
| SET-006 | Facilitator restarts after broadcast but before confirmation | Recover persisted state and reconcile |
| SET-007 | Expected transfer event is missing or its asset, amount, payer, or recipient differs | Do not report success |
| SET-008 | Resource fulfillment is attempted before confirmed settlement | Block fulfillment |

## E. PiRC interoperability safeguards

| ID | Scenario | Expected result |
|---|---|---|
| PIRC-001 | Payment Intent maps to x402 requirements | Deterministic mapping; validate network, asset, recipient, amount, expiry, and order binding |
| PIRC-002 | Asset provenance is missing, stale, or disputed | Do not treat the asset as verified or allowlisted |
| PIRC-003 | Oracle observation is stale, replayed, conflicting, or unverifiable | Reject it as pricing/risk context; do not alter the accepted payment amount |
| PIRC-004 | IPPR/QWF/economic gate fails or is unavailable | Do not infer payment authorization, credit, or a price guarantee |
| PIRC-005 | Payment Event claims success without transaction and transfer read-back | Reject the success claim |
| PIRC-006 | Economic allocation totals do not conserve the source amount | Reject the accounting projection |
| PIRC-007 | Seven-layer registry lists an asset without a reviewed network-specific contract allowlist | Do not enable payment for that asset |
| PIRC-008 | Personal identity data or signing secrets appear in payloads, logs, or fixtures | Reject or redact; prevent secret exposure |

## F. Required test layers

1. **Unit:** canonical amount parsing, requirement binding, network/asset policy, state transitions, and error mapping.
2. **Property and fuzz:** transaction decoding, integer boundaries, replay identity, and malformed inputs.
3. **Mocked integration:** verification/settlement separation, RPC timeout recovery, process restart, and event validation.
4. **Pi Testnet read-only:** network identity, RPC health, and ledger read-back.
5. **Pi Testnet writes:** only after explicit operator approval, dedicated Testnet credentials, reviewed asset allowlist, and a separate manual CI gate.
6. **Mainnet:** excluded from automated test execution and requires a separate release authorization.

## G. Release gates

- **Source:** accepted Pi exact-scheme specification and pinned PiRC source revision.
- **Static:** formatting, lint, type checks, schema validation, dependency checks, and secret scanning.
- **Unit:** deterministic positive, negative, and boundary tests pass.
- **Integration:** mocked RPC and recovery tests pass; CI has no live signing credentials.
- **Testnet:** read-only network checks pass; any write test has explicit manual approval.
- **Security:** independent review covers transaction parsing, authorization, replay, settlement, event semantics, and privacy.
- **Release:** maintainer acceptance and immutable evidence bundle; Mainnet remains a separate decision.
