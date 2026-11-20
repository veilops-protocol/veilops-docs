# Security model

Status: experimental, **not audited**. This document states what VEILOPS protects, against whom, under which assumptions, and, just as importantly, what it does not protect. To report a vulnerability see [SECURITY.md](../SECURITY.md).

## Actors

| Actor | Holds | Trusted for |
| --- | --- | --- |
| **Operator** | Plaintext input, Ed25519 signing key, X25519 owner key, commitment openings | Running the SDK honestly on its own machine; publishing records under its key |
| **Authorized viewer** (auditor, finance) | Its X25519 secret key; access to envelopes it was granted | Keeping the plaintext it reads confidential |
| **Verifier** | Public records only | Nothing; verification needs no trust in the verifier |
| **Public observer / adversary** | Every published record, network position, possibly chain data | Nothing |

## What VEILOPS protects

| Property | Mechanism | Holds against |
| --- | --- | --- |
| Amount confidentiality in the public record | Pedersen commitment (perfectly hiding) + zero-knowledge range proofs | Any observer, even computationally unbounded, for the commitment; ROM-based for the proof |
| Policy compliance `0 ≤ amount ≤ max` | `veilops-range-v1` soundness | A malicious operator: an over-limit amount cannot produce a verifying proof (under DL in ristretto255 + ROM) |
| Integrity of every public field | Ed25519 attestation + proof transcript binding | Anyone who does not hold the operator's signing key |
| Proof not transferable to another operation, limit, asset, category or commitment | Fiat–Shamir context | Replay into a different statement |
| Confidentiality of the envelope (name, recipient, amount, memo) | AES-256-GCM, X25519 + HKDF key wraps | Anyone without an authorized X25519 secret key |
| Envelope integrity | GCM authentication with `aad = veilops/0.1|operationId` | Tampering, or moving the ciphertext to another operation id |
| Selective disclosure | Per-recipient wraps, `grantAccess`, commitment opening | Disclosure only to parties the operator chooses |

## Trust assumptions

1. **The operator's signing key is trusted** to identify the operator. Whoever holds it can publish valid records in the operator's name. VEILOPS does not distribute or certify keys; verifiers must learn the correct public key out of band.
2. **The policy limit is public and chosen by the operator.** The proof shows the amount respects the limit **in the record**. Whether that limit is the one an organization agreed to is a governance question outside the protocol.
3. **The operator's environment is trusted.** Plaintext, openings and secret keys exist in memory during `process()`. Malware on that machine defeats everything.
4. **Viewers are trusted** with what they are given. Disclosure cannot be revoked.
5. **Cryptographic assumptions** are listed in [cryptography.md](cryptography.md#7-security-assumptions): discrete log in ristretto255, random-oracle SHA-512, Ed25519, X25519/HKDF/AES-GCM, a secure RNG, correct libraries.

## Metadata exposure

Everything in this table is visible to anyone who sees a `PublicRecord`.

| Field | Public? | Notes |
| --- | --- | --- |
| Category (`treasury`, `payroll`, …) | **Yes** | All modes |
| Asset (`USDC`, `SOL`, …) | **Yes** | All modes |
| Confidentiality mode | **Yes** | All modes |
| `createdAt` timestamp | **Yes** | Operator's clock; not verified |
| Policy id, limit `maxAmount`, `rangeBits` | **Yes** | Reveals that `amount ∈ [0, maxAmount]` |
| Amount commitment | **Yes** | Hides the amount; identical amounts produce unlinkable commitments |
| Envelope digest | **Yes** | Lets holders of the envelope match it to the record; reveals nothing else |
| Recipient | **Yes in `amount-only` mode**; omitted in `amount-and-recipient` and `full` | |
| Operator public key and signature | **Yes** | Links all records signed with the same key |
| Operation id, proof id | **Yes** | Random / content-derived identifiers |
| Proof size | **Yes** | Constant for a given `rangeBits`; does not depend on the amount |
| Amount | No | Committed |
| Operation name, memo | No | Encrypted only |
| Recipient (private modes) | No in the record | But see below |

## What is NOT protected

- **Anonymity.** VEILOPS makes no anonymity or unlinkability claims. Records signed by the same key are linkable; timing, category, asset and limit form a fingerprint.
- **On-chain settlement privacy.** VEILOPS does not move funds. If the payment itself settles as a normal transfer on a public chain, the amount and recipient are visible there regardless of the record. Token-2022 confidential transfers are planned, not integrated.
- **Recipient in `amount-only` mode.** It is published in plaintext by design.
- **Traffic analysis.** Who fetched or published a record, from which IP, is out of scope.
- **Exact amounts under small limits.** A limit of `1` reveals the amount is `0` or `1`.
- **A dishonest operator's plaintext.** The proof constrains the committed amount, not the truthfulness of the encrypted memo, name or recipient, and not whether the committed amount equals what was actually paid.
- **Key compromise.** No rotation, revocation or forward secrecy for viewing keys; no revocation list for signing keys.
- **Timestamps.** `createdAt` is self-reported. The experimental Solana memo anchor could later give an upper bound (slot time), but it has not been exercised on any cluster.
- **Side channels.** JavaScript `BigInt` arithmetic is not guaranteed constant time.
- **Denial of service.** Verification cost is linear in `rangeBits` (about 0.3 s per 32-bit record on a developer laptop in local runs); a service verifying untrusted records should rate-limit.
- **Dependency and supply-chain risk.** `npm audit` runs in CI, but there is no reproducible build or release signing yet.

## Closing the gaps: planned designs (not built)

The sender, timing and asset gaps above are the most common criticism of the current design, and the criticism is correct. Each gap has a candidate design. None is implemented, and none is claimed until it ships with its own proofs, test vectors and review.

| Gap | Candidate design | Main cost or trade-off |
| --- | --- | --- |
| **Asset** is public | Blinded asset tags: commit to the asset generator `H_a + r·G` and add an asset-surjection proof that it is one of an allowed set (the Confidential Assets construction). | A larger proof; the policy must list the allowed assets. |
| **Sender** is linkable | Per-record one-time keys, plus a proof of membership in a set of authorized operators (group or ring signature), plus relayed fee payment so the Solana fee payer does not link records. | Weaker accountability; relayers need incentives. |
| **Timing** is visible | Publication in fixed epochs with timestamps rounded to the epoch, plus optional cover records to hide volume. | Latency; cover records cost fees. |
| **Settlement** is public | Token-2022 confidential transfers, where the asset supports them. | Depends on token and wallet support. |

## On-chain verifier: trust properties

- The `veilops-verifier` program (devnet) rebuilds the transcript context from the public inputs itself. A proof therefore cannot be replayed for another operation, limit, asset or commitment.
- Every scalar must be canonical and every point must decode. Any failed bit proof or sum check aborts the transaction, so a `Record` exists only for a fully verified proof.
- The program is **upgradable** by the maintainer key while on devnet. Whoever holds that key could deploy different code. `scripts/program-hash.ts` shows the current binary hash and authority, and the hash is compared with the reproducible build. Before any mainnet use: an audit, then a decision on immutability.
- The operator's Solana address is the fee payer and the `Record` authority, so on-chain it links all of that operator's records (see "Sender" above).

## Threat scenarios

| Threat | Outcome |
| --- | --- |
| Operator tries to publish an over-limit payout as compliant | `provePolicy` refuses; a forged proof is rejected by `verifyPolicy` (soundness) |
| Attacker edits any field of a published record | Proof or attestation check fails (`tests/security/tamper.test.ts`) |
| Attacker re-labels a proof with a lower limit, another operation or another asset | Transcript mismatch, proof fails |
| Attacker submits malformed points/scalars to a verifier | Rejected without throwing |
| Stranger obtains an envelope | Cannot open it; `open` throws |
| Attacker swaps ciphertext between operations | GCM fails because `aad` binds the operation id |
| Attacker steals the operator signing key | **Can publish valid records** in the operator's name (still cannot prove an over-limit amount) |
| Auditor leaks the plaintext | Not preventable by cryptography |

## Review status

| Component | Tests | External review |
| --- | --- | --- |
| Pedersen + range proofs | Unit + security tests | None |
| Ed25519 attestation | Security tests | None (uses `@noble/curves`) |
| Envelope encryption | Unit tests | None |
| Policy engine | Unit tests | None |
| SDK lifecycle | Integration tests | None |
| Solana memo anchor | Round-trip test only | None; never sent to any cluster |
