# Cryptography

This document describes exactly what `packages/proof-engine` and `packages/encryption` implement. Status: **experimental, not audited, not externally reviewed.**

Libraries: `@noble/curves` 2.0.1 (ristretto255, Ed25519, X25519), `@noble/hashes` 2.0.1 (SHA-256, SHA-512, HKDF), WebCrypto (AES-256-GCM). No primitive is implemented by hand; the protocol layers are.

Notation: `G` is the ristretto255 base point, `L = 2^252 + 27742317777372353535851937790883648493` the group order, scalars live in `Z_L`, and `·` is scalar multiplication.

## 1. Pedersen commitments

```
C = v·G + r·H
```

- `v` is the amount as an integer, `r` a uniformly random non-zero scalar (64 random bytes reduced mod `L`, `randomScalar()`).
- `H` is derived with RFC 9380 hash-to-curve (`ristretto255_XMD:SHA-512_R255MAP_RO_`) from the message `VEILOPS/pedersen/H/v1` with DST `VEILOPS-V01-CS01-with-ristretto255_XMD:SHA-512_R255MAP_RO_`. Nobody knows `log_G(H)`, which is required for binding.
- **Perfectly hiding**: for any `v`, `C` is uniformly distributed over the group.
- **Computationally binding**: opening `C` to two different values yields `log_G(H)`, so it is as hard as discrete log.
- **Homomorphic**: `max·G − C = (max − v)·G + (−r)·H`, which is how the upper bound is proven without a second commitment from the prover.

Secret scalars are multiplied with `mulSecret` (noble's constant-time ladder); public scalars in verification use `mulPublic` (`multiplyUnsafe`, variable time).

## 2. Range proof by bit decomposition

Statement: *I know `(v, r)` with `C = v·G + r·H` and `0 ≤ v < 2^n`.*

### Prover

1. Write `v = Σ_{i=0}^{n−1} b_i·2^i` with `b_i ∈ {0, 1}`.
2. Choose random `r_0 … r_{n−2}` and set
   ```
   r_{n−1} = (r − Σ_{i<n−1} 2^i·r_i) · (2^{n−1})^{−1}   mod L
   ```
   so that `Σ 2^i·r_i = r`.
3. Commit to each bit: `C_i = b_i·G + r_i·H`. Then `Σ 2^i·C_i = C`.
4. For each `C_i`, prove `C_i` commits to 0 **or** 1 with a CDS OR-proof (below).

### Bit OR-proof (Cramer–Damgård–Schoenmakers)

Define `Y_0 = C_i` and `Y_1 = C_i − G`. If `b_i = b`, the prover knows `x = r_i` with `Y_b = x·H`; for the other branch `Y_{1−b}` it knows nothing.

Prover, with real branch `b` and simulated branch `f = 1 − b`:

```
k, c_f, s_f  ←$ Z_L
A_b = k·H
A_f = s_f·H − c_f·Y_f                     (simulated transcript)
e   = HashToScalar(domain, ctx, i, C_i, A_0, A_1)
c_b = e − c_f                 mod L
s_b = k + c_b·x               mod L
output (C_i, c_0, c_1, s_0, s_1)
```

Verifier:

```
A_0' = s_0·H − c_0·Y_0
A_1' = s_1·H − c_1·Y_1
accept iff  c_0 + c_1 ≡ HashToScalar(domain, ctx, i, C_i, A_0', A_1')   (mod L)
```

Correctness: for the real branch, `s_b·H − c_b·Y_b = k·H + c_b·x·H − c_b·x·H = k·H = A_b`; the simulated branch reproduces `A_f` by construction.

### Range verifier

```
for i in 0..n−1:  verify bit proof i; reject on failure
accept iff  Σ 2^i·C_i == C
```

Additionally the verifier rejects if `bits.length ≠ n` or `n ∉ [1, 64]`, and never throws on malformed input (invalid hex, non-canonical scalars `≥ L`, invalid point encodings all yield `false`).

## 3. Policy proof: `veilops-range-v1`

Statement: *`C` opens to some `v` with `0 ≤ v ≤ max`.*

```
lower = RangeProof( v,        r,  n, ctx | "lower")    on C
upper = RangeProof( max − v, −r,  n, ctx | "upper")    on D = max·G − C   (verifier computes D)
```

Requirements checked by `verifyPolicy`: both proofs use `n = rangeBits`, and `0 ≤ max < 2^n` with `max` a safe integer.

Why this proves `v ≤ max`: the lower proof gives `v ∈ [0, 2^n)`, the upper proof gives `w ∈ [0, 2^n)` with `w ≡ max − v (mod L)`. Because `n ≤ 64` and `L > 2^252`, there is no wrap-around, so `w = max − v` over the integers and therefore `0 ≤ v ≤ max`. (The policy engine further restricts `rangeBits` to `1..48`.)

## 4. Fiat–Shamir transcript

```
HashToScalar(domain, p_1, …, p_k) =
    LE_int( SHA-512( utf8(domain) || enc(p_1) || … || enc(p_k) ) )  mod L
enc(p) = u32_be(len(p)) || p          (strings as UTF-8)
```

| Item | Value |
| --- | --- |
| `domain` | `VEILOPS/range/bit-or/v1` |
| `ctx` | `canonicalJson({ v: 'veilops-range-v1', operationId, policyId, maxAmount, rangeBits, asset, category, amountCommitment })` followed by `|lower` or `|upper` |
| `i` | bit index as a decimal string |
| `C_i`, `A_0`, `A_1` | 32-byte ristretto255 encodings |

Each bit proof hashes its own statement (`C_i`) and the full public context, which includes the top-level commitment. Consequences:

- a proof cannot be moved to another operation id, policy id, limit, width, asset, category or commitment (covered by tests in `tests/security/tamper.test.ts`);
- bit proofs cannot be reordered or mixed across proofs, because the index and context are hashed.

Length prefixes make the encoding injective; the 512-bit hash reduced mod `L` gives a challenge with negligible bias.

## 5. Record attestation (not a ZK proof)

```
msg = utf8("VEILOPS/record/v1|" + canonicalJson(record without attestation))
sig = Ed25519.sign(msg, operatorSecretKey)          (RFC 8032, via @noble/curves)
```

This proves that the holder of `attestation.publicKey` published exactly these fields. It binds fields that the proof transcript does not (`createdAt`, `mode`, `recipient`, `envelopeDigest`, `proofId`, `proof`). It reveals nothing and proves nothing about the amount.

`proofId = "prf_" + first 20 hex chars of SHA-256(canonicalJson(proof))`.

## 6. Envelope encryption

```
dataKey  ←$ {0,1}^256,  nonce ←$ {0,1}^96
ct       = AES-256-GCM(dataKey, nonce, payload, aad = "veilops/0.1|" + operationId)

per recipient public key P:
  e ←$ X25519 secret, E = X25519(e, base)
  ss  = X25519(e, P)
  kek = HKDF-SHA256(ikm = ss, salt = E || P, info = "veilops/kek/v1", L = 32)
  wrap = AES-256-GCM(kek, nonce_w ←$ {0,1}^96, dataKey, aad)
```

- Opening recomputes `ss = X25519(sk, E)` and unwraps; any tampering with ciphertext, nonce, aad or wrap fails GCM authentication.
- `grantAccess` unwraps the data key with the owner's key and adds a new wrap; the payload ciphertext is untouched.
- `envelopeDigest = SHA-256(ciphertext || "|" || nonce || "|" || aad)` (string concatenation of the base64/hex fields). It does not cover the key wraps.
- Envelopes are not sender-authenticated: anyone who knows a recipient's public key can create an envelope for them. Authenticity of the overall record comes from the Ed25519 attestation over `envelopeDigest`.

## 7. Security assumptions

| Assumption | Needed for |
| --- | --- |
| Discrete logarithm is hard in ristretto255 | Binding of commitments, soundness of the OR-proofs |
| `log_G(H)` is unknown (RFC 9380 hash-to-curve output) | Binding |
| SHA-512 behaves as a random oracle | Fiat–Shamir soundness and zero knowledge of the non-interactive proofs |
| Ed25519 is EUF-CMA secure | Attestation |
| X25519 CDH hardness, HKDF and AES-256-GCM security | Envelope confidentiality and integrity |
| A cryptographically secure RNG (`crypto.getRandomValues`) | Blindings, nonces, keys; a biased or repeated `k` in a bit proof leaks `r_i` |
| `@noble/curves` / `@noble/hashes` and the platform WebCrypto are correct | Everything |

Properties claimed under these assumptions: completeness, computational soundness, and honest-verifier zero knowledge of the interactive protocol, giving non-interactive zero knowledge in the random-oracle model. No trusted setup.

## 8. Limitations

- **Not audited.** No formal proof of this exact composition has been written; the building blocks are classical.
- **Not succinct.** Each bit costs one point and four scalars (160 bytes raw). A policy proof has `2n` bits; at `n = 32` that is 64 bit proofs, about 10 KB raw and about 31 KB as currently serialized (hex inside JSON, then base64). Verification performs roughly `5 · 2n` scalar multiplications. Bulletproofs would give logarithmic size; this is a Phase 02 evaluation item.
- **Serialization is not compact.** `c_1` could be derived as `e − c_0`; hex-in-JSON-in-base64 roughly triples the size. A versioned binary format is planned.
- **Timing side channels.** Point multiplication with secret scalars uses noble's constant-time routine, but scalar arithmetic on JavaScript `BigInt` is not guaranteed constant time. Do not run the prover where an attacker can measure timing precisely.
- **The limit is public.** The proof reveals that `v ∈ [0, max]`; with a small `max` that is itself informative.
- **No ownership of proofs.** A record is public; anyone can copy its proof. The attestation shows who signed a given record, but the scheme has no notion of which key is entitled to an operation id.
- **No forward secrecy for envelopes.** Compromise of a recipient's X25519 secret key reveals every envelope wrapped for it.
- **Disclosure is irrevocable.** A party that has received plaintext or an opening keeps it.
