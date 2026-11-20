# Developer guide

## Prerequisites

- Node.js 22 (`nvm use` reads `.nvmrc`). `package.json` `engines` allows `>=20.10`; CI uses 22.
- npm (the repository uses npm workspaces and a committed `package-lock.json`).

## Setup

```bash
git clone https://github.com/veilops-protocol/veilops.git
cd veilops
npm ci
```

## Scripts

| Command | What it does |
| --- | --- |
| `npm test` | `vitest run` over `tests/**/*.test.ts` (74 tests: unit, integration, security, reference vectors) |
| `npm run test:watch` | vitest in watch mode |
| `npm run typecheck` | `tsc` over packages, tests, scripts and examples, then the website's typecheck |
| `npm run lint` | ESLint 9 flat config with `typescript-eslint` recommended over `packages tests examples scripts` |
| `npm run build` | Builds `apps/website` (`next build`). The packages themselves are consumed as TypeScript source and have no build step |
| `npm run dev` | Website dev server |
| `npm run samples` | Regenerates the sample explorer dataset (`scripts/generate-samples.ts`) |
| `npm run audit:deps` | `npm audit --omit=dev --audit-level=high` |

## Layout

Each package is `packages/<name>/src/index.ts` with `"main"`/`"types"`/`"exports"` pointing at the TypeScript source. There is no `dist/`; consumers need a TypeScript-aware toolchain (`tsx`, vitest, Next.js, a bundler).

```text
packages/shared          types + encoding           (no crypto decisions)
packages/encryption      seal / open / grantAccess
packages/policy-engine   definePolicy / evaluate
packages/proof-engine    pedersen.ts, range.ts, index.ts
packages/sdk             createClient / process
packages/solana-adapter  memo anchor
tests/unit | integration | security
examples/                runnable with npx tsx
```

## Using the SDK

```ts
import { createClient, PolicyViolation } from '@veilops/sdk';
import { definePolicy } from '@veilops/policy-engine';

const veil = createClient({
  // signingKey?: SigningKey   — defaults to a fresh Ed25519 key (generateSigningKey())
  // owner?: KeyPair           — defaults to a fresh X25519 key (generateViewingKeyPair())
  // viewers?: string[]        — extra X25519 public keys (hex) that may open envelopes
});

const policy = definePolicy({ name: 'vendors', maxAmount: 1_000_000, allowedCategories: ['vendor'] });

try {
  const res = await veil.process(input, policy, { onStage: e => console.log(e.stage, e.ms, e.detail) });
  publish(res.record);              // PublicRecord
  store(res.envelope, res.opening); // keep private
} catch (e) {
  if (e instanceof PolicyViolation) console.log(e.result.checks.filter(c => !c.passed));
}
```

The client also exposes `verify` (= `verifyRecord`), `signingPublicKey`, `ownerPublicKey` and `owner` (the owner key pair, including its secret key, which you are responsible for storing).

The SDK re-exports everything from `@veilops/shared` and the other packages as namespaces:

```ts
import { encryption, policy, proofs, canonicalJson } from '@veilops/sdk';
```

### Persistent keys

The SDK never stores keys. To reuse an operator identity, persist the 32-byte Ed25519 seed yourself and pass `signingKeyFromSeed(seed)` from `@veilops/proof-engine`. Never commit seeds; `.env*` files are git-ignored.

## Lower-level APIs

### Proofs

```ts
import { provePolicy, verifyPolicy, encodeProof, decodeProof } from '@veilops/proof-engine';

const out = provePolicy({ operationId, policyId, maxAmount: 5000, rangeBits: 16, asset: 'USDC', category: 'treasury', amount: 1200 });
verifyPolicy(out.publicInputs, out.proof).valid;   // true
// out.opening = { value, blinding } — private
```

`proveRange(value, blinding, n, ctx)` / `verifyRange(C, proof, ctx)` and `commit(v, r)` / `verifyOpening(hex, v, r)` are exported for direct use.

### Encryption

```ts
import { generateViewingKeyPair, seal, openText, grantAccess, envelopeDigest } from '@veilops/encryption';

const a = generateViewingKeyPair();
const env = await seal('payload', [a.publicKey], 'my-aad');
await openText(env, a.secretKey);
```

### Solana anchor (experimental)

```ts
import { buildAnchorInstruction, parseAnchorMemo } from '@veilops/solana-adapter';

const ix = buildAnchorInstruction(record, signerBase58);   // { programId, keys, data }
// Convert to your Solana client's instruction type, then sign and send with the caller's wallet.
```

This has not been tested on any cluster. Do not assume it works on devnet or mainnet until Phase 03.

## Writing tests

- Place tests under `tests/unit`, `tests/integration` or `tests/security` with a `.test.ts` suffix; vitest picks them up via `vitest.config.ts` (timeout 30 s, since proofs take hundreds of milliseconds).
- Import packages by name (`@veilops/proof-engine`), not by relative path.
- Every change to a verifier needs a **negative** test: tampered field, wrong context, malformed encoding.
- Keep `rangeBits` small (8–24) in unit tests to keep the suite fast.

## Examples

```bash
npx tsx examples/basic-operation.ts        # lifecycle, record, independent verification, tamper, policy violation
npx tsx examples/selective-disclosure.ts   # auditor access, grantAccess, commitment opening
```

## Conventions

- Hash or sign only `canonicalJson` output.
- Verifiers return `false` / invalid reports; they never throw on input.
- Use `mulSecret` for secret scalars, `mulPublic` only for public ones.
- Changing a proof format or transcript requires a new `proofSystem` identifier.
- Status labels follow `ImplementationStatus`; do not upgrade one without evidence.
