# Pi Network interoperability profile

**Status:** Documentation proposal only. This is not an SDK implementation, security audit, Pi Network endorsement, or x402 Foundation certification.

This profile describes a narrow interoperability boundary between x402 v2 exact payments on the Pi Network namespace and selected concepts from the community-maintained PiRC repository.

## Scope

- Keep x402 exact payment requirements, authorization, verification, and settlement as the payment boundary.
- Treat PiRC as an optional data-model and evidence reference, never as an x402 dependency or authority.
- Require explicit network and asset identity, canonical integer amounts, independent settlement verification, and reproducible evidence.
- Keep valuation, provenance, oracle, identity, governance, and economic-accounting systems outside the payment authorization path.

## Documents

- [PiRC interoperability profile](./pirc-interoperability-profile.md): bounded concept mapping and security invariants.
- [Conformance matrix](./conformance-matrix.md): negative cases and evidence required before claiming implementation conformance.

## Source references

- PiRC source reviewed: [Ze0ro99/PiRC](https://github.com/Ze0ro99/PiRC), commit `940bd54cc53e7138cbdb6a2d163a8dd13039f595`.
- Pi exact-scheme specification proposal: [x402-foundation/x402#3329](https://github.com/x402-foundation/x402/pull/3329). This profile does not duplicate that specification or imply that it has been accepted.
- x402 contribution requirements: [CONTRIBUTING.md](https://github.com/x402-foundation/x402/blob/main/CONTRIBUTING.md).

## Non-goals

This profile does not provide a facilitator, wallet, Soroban contract, production gateway, price oracle, custody service, or SDK adapter. It does not claim that any PiRC token, reserve, issuer, liquidity pool, or contract is deployed or verified. It does not authorize Mainnet transactions.

## Evidence rule

A conformance claim must identify the source commit, test command, runtime/toolchain, CI run, artifact digest, and timestamp. Network-dependent claims must additionally include sanitized read-back evidence. Missing evidence is `NOT_RUN` or `UNVERIFIED`, never a pass.
