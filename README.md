<p align="center"><a href="https://veilops.tech"><img src="https://raw.githubusercontent.com/veilops-protocol/veilops/main/docs/assets/banner.png" alt="VEILOPS: Privacy is not the absence of proof. It's proof without exposure." width="100%"></a></p>

# veilops-docs

Documentation for VEILOPS: architecture, protocol and record format, cryptography, security model, developer guide and roadmap.

> **Status: Phase 01.** The documents describe experimental, unaudited software and are updated together with the code.

**Mirror of** [`veilops-protocol/veilops`](https://github.com/veilops-protocol/veilops) → `docs/` (see [MIRROR.md](MIRROR.md)). Propose changes on the monorepo, in the same pull request as the code they describe.

## Contents

| Document | Topic |
| --- | --- |
| `architecture.md` | Package graph, data flow, trust boundaries |
| `protocol.md` | `OperationInput`, `Policy`, `Envelope`, `PublicRecord`; lifecycle `created → encrypted → policy → proof → verified`; verification algorithm |
| `cryptography.md` | Pedersen commitments, CDS OR-proofs, Fiat–Shamir transcript, assumptions, limitations |
| `security.md` | Threat model, trust assumptions, metadata exposure, what is not protected |
| `developer-guide.md` | Setup, scripts, APIs, tests |
| `roadmap.md` | Phases 01–05 |

## Usage

The files are plain Markdown with Mermaid diagrams and render directly on GitHub. Relative links to source files (for example `../SECURITY.md`) resolve only inside the monorepo.

License: Apache-2.0.
