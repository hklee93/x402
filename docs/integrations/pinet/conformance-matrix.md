# Pi Network exact-scheme conformance matrix

**Profile:** PiRC-to-x402 interoperability 0.1.0  
**Status:** Test plan only. No case in this document is claimed to have run.  
**Target:** x402 v2 exact scheme on `pinet:testnet` first

## Result vocabulary

- `NOT_IMPLEMENTED`: the required behavior is not implemented.
- `NOT_RUN`: implementation may exist, but no reproducible test evidence was inspected.
- `PASS`: test ran against the stated commit and produced the expected result.
- `FAIL`: test ran and did not meet the expected result.
- `BLOCKED`: required network, fixture, toolchain or permission was unavailable.
- `NOT_APPLICABLE`: reason is documented and approved by maintainers.

Do not turn a missing test into a pass. Each result must include commit SHA, command, runtime/toolchain, CI URL, artifact digest and timestamp.

## A. Network and asset identity

| ID | Scenario | Expected result | Evidence |
|---|---|---|---|
| NET-001 | Correct `pinet:testnet` and exact Testnet passphrase | Accept configured network identity | Horizon root + Soroban health response, sanitized |
| NET-002 | Mainnet passphrase supplied for Testnet | Reject before signing or settlement | Negative integration test |
| NET-003 | CAIP-2 network and RPC endpoint identify different networks | Fail closed | Mocked RPC and live read-only probe |
| NET-004 | Unknown or unregistered CAIP-2 network | Reject | Unit test |
| NET-005 | Asset contract is not in the network-specific allowlist | Reject | Unit test + contract registry fixture |
| NET-006 | Contract ID exists but is not the expected SEP-41 interface | Reject | Contract spec/read-back evidence |
| NET-007 | Mainnet config is selected by changing only one environment variable | Reject; Mainnet requires separate release authorization | Configuration/policy test |

## B. Amount and payment requirement binding

| ID | Scenario | Expected result | Evidence |
|---|---|---|---|
| AMT-001 | Positive canonical integer string | Accept if within supported range | Unit test |
| AMT-002 | Zero, negative, sign, whitespace, decimal or exponent notation | Reject | Parameterized unit tests |
| AMT-003 | Leading-zero amount | Reject unless the upstream scheme explicitly permits it | Unit test |
| AMT-004 | Amount exceeds supported i128 range | Reject without overflow | Boundary/property test |
| AMT-005 | Payload amount differs from accepted requirements | Reject | Facilitator verification test |
| AMT-006 | Recipient differs from `payTo` | Reject | Facilitator verification test |
| AMT-007 | Asset contract differs from `asset` | Reject | Facilitator verification test |
| AMT-008 | Merchant valuation expires after intent creation | Require a new buyer-approved requirement; never silently reprice | Adapter lifecycle test |

## C. XDR and Soroban authorization

| ID | Scenario | Expected result | Evidence |
|---|---|---|---|
| XDR-001 | Malformed or non-base64 XDR | Reject without panic | Fuzz/unit test |
| XDR-002 | More than one transaction operation | Reject | Unit test |
| XDR-003 | Operation is not the required InvokeHostFunction | Reject | Unit test |
| XDR-004 | Host function is not the allowed contract invocation | Reject | Unit test |
| XDR-005 | Function is not `transfer` or argument count is wrong | Reject | Unit test |
| XDR-006 | Nested invocation authorizes additional effects | Reject | Unit test |
| XDR-007 | Unsupported authorization credential arm or unexpected signer | Reject | Unit test |
| XDR-008 | Signature is invalid or bound to a different authorization payload | Reject | Cryptographic test |
| XDR-009 | Authorization expires or exceeds allowed ledger-time window | Reject | Boundary test using controlled ledger fixtures |
| XDR-010 | Client transaction source or payer is the facilitator where prohibited | Reject | Facilitator security test |

## D. Verification, settlement and recovery

| ID | Scenario | Expected result | Evidence |
|---|---|---|---|
| SET-001 | `/settle` receives a payload never verified | Perform full independent verification; reject if invalid | API integration test |
| SET-002 | Payload or requirements change between verify and settle | Reject | State-binding test |
| SET-003 | Settlement is submitted twice with the same idempotency identity | Return/reconcile the same outcome; never transfer twice | Concurrency test |
| SET-004 | Same idempotency key is reused with different payload | Reject as conflict | Unit/API test |
| SET-005 | RPC times out after broadcast | Mark pending/unknown and reconcile by transaction identity before retry | Fault-injection test |
| SET-006 | Facilitator restarts after broadcast but before confirmation | Recover persisted state and reconcile | Restart integration test |
| SET-007 | Settlement simulation changes resource fee or footprint | Rebuild from fresh simulation; enforce configured fee ceiling | Integration test |
| SET-008 | Expected transfer event missing or amount/recipient differs | Do not report success | Event verification test |
| SET-009 | Extra balance movement or unexpected event occurs | Reject and raise security signal | Adversarial integration test |
| SET-010 | Resource side effect is attempted before confirmed settlement | Block fulfillment | Resource-server lifecycle test |

## E. PiRC data and lifecycle interoperability

| ID | Scenario | Expected result | Evidence |
|---|---|---|---|
| PIRC-001 | Valid payment intent maps to exact x402 requirements | Deterministic mapping with explicit asset decimals and network | Adapter unit test |
| PIRC-002 | Payment intent has expired or nonce is reused | Reject or return idempotent prior outcome; no new transfer | Adapter test |
| PIRC-003 | Asset provenance is unverified, disputed or stale | Do not treat asset as verified/allowlisted | Registry policy test |
| PIRC-004 | Oracle observation is stale, duplicate-source, invalidly signed or below quorum | Quarantine pricing input; do not alter signed payment amount | Oracle adapter test |
| PIRC-005 | Dynamic IPPR/QWF gate fails | No credit/price guarantee or settlement authorization is inferred | Policy test |
| PIRC-006 | Payment event claims success without transaction and ledger read-back | Reject event as unsubstantiated | Event consumer test |
| PIRC-007 | Economic allocation totals do not conserve the source amount | Reject settlement projection | Integer property test |
| PIRC-008 | Implementation evidence omits commit, test run or artifact digest | Keep status unverified | Evidence validator test |
| PIRC-009 | Seven-layer registry lists an asset without a verified network contract | Do not enable payment for that asset | Allowlist test |
| PIRC-010 | KYC/identity or personal data appears in payment payload/log | Reject or redact according to privacy policy | Data-leak test |

## F. Required test layers

1. **Pure unit tests:** parsing, integer bounds, mapping, state transitions and policy decisions.
2. **Property/fuzz tests:** XDR decoding, amount boundaries, idempotency and integer conservation.
3. **Mocked integration tests:** RPC errors, simulation changes, pending outcomes and process restarts.
4. **Pi Testnet read-only checks:** network identity, protocol version, RPC health and ledger observations.
5. **Pi Testnet transaction tests:** only after explicit operator approval, dedicated Testnet signer, allowlisted contracts and a separate CI/manual gate.
6. **Mainnet:** excluded from this profile's automated test path.

## G. Release gates

- **Gate 0 — Source:** upstream scheme PR reviewed; PiRC source commit pinned; no duplicate or stale branch content.
- **Gate 1 — Static:** format, lint, typecheck, schema validation, dependency and secret scans.
- **Gate 2 — Unit:** all deterministic tests pass; negative and boundary cases included.
- **Gate 3 — Integration:** mocked RPC/fault-injection suite passes; no live write credentials in CI.
- **Gate 4 — Testnet:** read-only identity checks pass; transaction tests require manual approval and dedicated Testnet keys.
- **Gate 5 — Security:** independent review of XDR, auth, replay, fee sponsorship, event semantics, idempotency and privacy.
- **Gate 6 — Release:** maintainer acceptance and immutable evidence bundle; Mainnet remains a separate authorization.

## Current evidence status

All test IDs in this matrix are `NOT_RUN` until a concrete implementation, command output and immutable CI evidence are attached. This document is a review plan, not a claim of passing conformance.
