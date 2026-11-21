# Pi Network (pinet) interoperability

**Status:** Experimental integration guidance. Not a protocol implementation, security audit, Pi Network endorsement, or x402 Foundation certification.

This directory documents an optional mapping between the Pi Network `pinet` exact-scheme work and selected data/evidence concepts from the community-maintained [PiRC repository](https://github.com/Ze0ro99/PiRC).

## Scope

- Preserve the x402 v2 `exact` scheme as the payment protocol boundary.
- Keep Pi network identity, asset identity, authorization, and settlement verification explicit.
- Map useful PiRC concepts to x402 request, settlement, and evidence lifecycles without making PiRC a dependency of x402.
- Separate payment authorization from valuation, provenance, oracle, governance, RWA, and economic-accounting systems.
- Define testable negative cases and an evidence policy before claiming interoperability.

## Documents

- [PiRC interoperability profile](./pirc-interoperability-profile.md) — bounded crosswalk and security invariants.
- [Conformance matrix](./conformance-matrix.md) — test cases, layers, and evidence required for each claim.
- [Test vectors](./test-vectors.json) — machine-readable scenario catalogue. These are scenario definitions, not executed test results or signed XDR fixtures.

## Source pinning

The PiRC source reviewed for this profile is pinned to commit:

`Ze0ro99/PiRC@940bd54cc53e7138cbdb6a2d163a8dd13039f595`

The x402 upstream base used for this contribution branch is:

`x402-foundation/x402@e35bf8846f3e40f584f7235ea96c0841cb597b27`

The Pi exact-scheme proposal is tracked separately in [x402-foundation/x402#3329](https://github.com/x402-foundation/x402/pull/3329). This profile does not duplicate that proposal or claim it has been merged.

## Non-goals

This profile does not:
- assert that PiRC standards are official Pi Network standards;
- treat IPPR, QWF, seven-layer token concepts, or an oracle quote as a guaranteed Pi price or redemption promise;
- authorize Mainnet deployment or payment processing;
- provide a production facilitator, wallet, Soroban contract, or SDK adapter;
- claim that any PiRC contract, token, reserve, issuer, or liquidity pool is deployed or verified.

## Review rule

Every implementation claim must link to a source revision, test run, artifact digest, and (where applicable) a network read-back record. Missing evidence is recorded as `NOT_RUN`, `UNVERIFIED`, or `NOT_APPLICABLE`, never as a pass.
