# External review: scope and status

**Status: no independent review has been performed. No reviewer has been engaged yet.**

This document is the package we will hand to an independent cryptography reviewer. It is public so that anyone can see exactly what would be reviewed and what would not. When a review happens, the report will be published here **unedited**, with the reviewer's name, the commit reviewed and the dates. Until then, treat all of VEILOPS as experimental.

## In scope

| Area | Files | Lines (approx.) |
| --- | --- | --- |
| Pedersen commitments on ristretto255, derivation of generator `H`, Fiat–Shamir challenge | `packages/proof-engine/src/pedersen.ts` | 80 |
| Bit-decomposition range proof (CDS OR-proofs) | `packages/proof-engine/src/range.ts` | 95 |
| Policy proof `veilops-range-v1` (lower and upper bound), transcript binding, record verification, Ed25519 attestation | `packages/proof-engine/src/index.ts` | 165 |
| Envelope encryption: AES-256-GCM, X25519 + HKDF key wrapping, selective disclosure | `packages/encryption/src/index.ts` | 125 |
| Canonical JSON and content identifiers (anything hashed or signed) | `packages/shared/src/index.ts` | 100 |
| Specification | `docs/cryptography.md`, `docs/protocol.md`, `test-vectors/veilops-range-v1.json` | — |

That is about 570 lines of TypeScript. The project depends on `@noble/curves` and `@noble/hashes`, both pinned to exact versions. Those libraries were audited separately; their audits are not ours.

## Questions for the reviewer

1. **Soundness of `veilops-range-v1`.** Can a prover produce a verifying proof for an amount outside `[0, maxAmount]`? Is the upper-bound construction on `D = maxAmount·G − C` sound when `maxAmount < 2^n`?
2. **Zero knowledge.** Do the OR-proofs leak anything about each bit beyond the statement? Is randomness handled correctly, including the derivation of the last blinding factor?
3. **Fiat–Shamir.** Is the transcript complete? That is, does it bind the statement, the index, all commitments and all public inputs, and is it unambiguously encoded? Are there weak-Fiat–Shamir issues?
4. **Generator `H`.** Is the hash-to-curve derivation correct, with nobody knowing `log_G(H)`?
5. **Canonical encodings.** Does every scalar and point decoding reject non-canonical inputs? Is there any malleability that changes `proofId` without invalidating the proof?
6. **Envelope encryption.** Nonce uniqueness, key wrapping, AAD binding to the operation id, HKDF parameters.
7. **Attestation.** Is the domain-separated signing payload unambiguous? Is canonical JSON a safe encoding for signing?
8. **Side channels.** Are secret-dependent operations constant-time where it matters (the prover side)?

## Out of scope

- The website UI and its animations.
- The Solana devnet anchoring scripts. They only publish a hash; they hold no funds.
- Anonymity and unlinkability. VEILOPS makes no such claims; see [security.md](security.md#what-is-not-protected).
- Token-2022 confidential transfers. This is planned, not implemented.

## Known limitations (already documented)

- Proof size is O(n) in the number of bits. The proofs are not succinct like Bulletproofs or a SNARK.
- Honest-verifier zero knowledge is proven only in the random-oracle model.
- Operator keys are not certified. Verifiers must learn the operator's public key out of band.
- The sample explorer dataset is generated, not real activity.

## Evidence already available to a reviewer

- 73 automated tests: unit, integration, security/tamper and reference-vector tests.
- The reference vectors are verified by a second, independent, standard-library-only Python implementation ([verifiers/python](../verifiers/python)). Its curve code is checked against the official RFC 9496 and RFC 8032 test vectors.
- Reproducible package builds and signed build provenance ([VERIFY.md](../VERIFY.md)).

## Contact

Reviewers and auditors: open a [private security advisory](https://github.com/veilops-protocol/veilops/security/advisories/new) and say it is about a review. We publish the engagement, scope and dates once agreed, and the full report when it is delivered.
