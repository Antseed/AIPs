# AntSeed Improvement Proposals (AIPs)

AntSeed Improvement Proposals (AIPs) describe standards and processes for the
[AntSeed network](https://github.com/AntSeed/antseed) — a fully decentralized,
peer-to-peer protocol for AI services. AIPs cover the wire protocol (discovery,
transport, metering, payments, reputation, verification), the on-chain
contracts, plugin and client interfaces, and network economics, as well as
meta-level processes around AntSeed governance.

This repository tracks past and ongoing improvements to AntSeed in the form of
AIPs, in the same spirit as Ethereum's [EIPs](https://github.com/ethereum/EIPs).

**Before you write an AIP, read [AIP-1: AIP Purpose and Guidelines](AIPS/aip-1.md).**

## Mission

The goal of the AIP project is to standardize and provide high-quality
documentation for AntSeed itself and the conventions built upon it. Because
AntSeed has no central server or coordinator — the network *is* the set of
participating nodes — protocol changes must be specified precisely enough that
independent implementations interoperate. AIPs are the canonical place where
those specifications are proposed, debated, and finalized.

## How to propose an improvement

1. **Discuss the idea first.** Open a thread in
   [AntSeed Discussions](https://github.com/AntSeed/antseed/discussions) (or an
   issue in this repository) to gauge interest and get early feedback. This
   vetting saves both you and the community time.
2. **Read [AIP-1](AIPS/aip-1.md)**, which describes the AIP types, statuses,
   required structure, and the review process.
3. **Fork this repository** and copy [`aip-template.md`](aip-template.md) to
   `AIPS/aip-draft-your-title.md`. Fill in every section. Leave the `aip`
   number unassigned — an editor assigns it when the PR is opened.
4. **Open a pull request.** CI validates the front matter automatically. An
   AIP editor will assign a number, check formatting, and merge your **Draft**.
   Merging a Draft signals only that the proposal is properly formed — not that
   it will be adopted.
5. **Drive it forward.** Move your AIP through **Review** and **Last Call** by
   incorporating feedback, then to **Final** once the community and editors
   agree the specification is complete and stable.

## AIP status terms

- **Idea** — pre-draft, not tracked in this repository.
- **Draft** — the first formally tracked stage; merged when properly formatted.
- **Review** — the author marks the AIP ready for peer review.
- **Last Call** — final review window (at least 14 days) before Final.
- **Final** — the accepted standard, in a state of finality.
- **Stagnant** — inactive for 6+ months; may be resurrected to Draft.
- **Withdrawn** — the author has withdrawn the proposal.
- **Living** — continually updated by design (e.g. AIP-1 itself).

## AIP types

- **Standards Track** — changes affecting most or all AntSeed implementations,
  in one of five categories:
  - **Core** — the P2P wire protocol: discovery, transport framing, metering,
    payment message flows, reputation, and model verification.
  - **Contracts** — the on-chain protocol: deposits, channels, staking,
    emissions, slashing, and registry contracts.
  - **Interface** — client-facing APIs and conventions: provider/router plugin
    interfaces, CLI behavior, proxy endpoints.
  - **Economics** — token economics: ANTS emissions, staking parameters,
    pricing and fee conventions.
  - **ASRC** (AntSeed Request for Comment) — application-level standards and
    conventions built on top of the protocol.
- **Meta** — processes surrounding AntSeed, including changes to the AIP
  process itself.
- **Informational** — design guidelines and general information that do not
  propose a new feature.

## AIP index

| Number | Title | Type | Status |
|-------:|-------|------|--------|
| [1](AIPS/aip-1.md) | AIP Purpose and Guidelines | Meta | Living |
| [2](AIPS/aip-2.md) | Recognized Usage Trust | Standards Track | Final |
| [3](AIPS/aip-3.md) | Pluggable Verifier SDKs | Standards Track | Final |
| [4](AIPS/aip-4.md) | Proof-Carrying Wash-Trading Enforcement | Standards Track | Final |
| [5](AIPS/aip-5.md) | Model Routing Peers | Standards Track | Draft |
| [6](AIPS/aip-6.md) | Referrals and Client Attribution | Standards Track | Review |

## Validation

Every proposal's front matter is checked in CI. You can run the validator
locally:

```bash
node scripts/validate.mjs
```

## Website

The site at [aips.antseed.com](https://aips.antseed.com) is generated straight
from `AIPS/*.md` — one page per proposal plus an index grouped by status. No
CMS, no server: `scripts/build-site.mjs` renders the markdown with the
templates and styles in `site/`, and Cloudflare Pages builds and deploys on
every merge to `main`.

Build locally:

```bash
npm install
npm run build   # validates, then renders to _site/
```

Deployment is Cloudflare Workers Builds: the repo is git-connected to the
`aips` worker, and every push to `main` runs `npm run build` followed by
`npx wrangler deploy` (config in `wrangler.jsonc`, serving `_site/` as static
assets). Manual deploy: `npm run build && npx wrangler deploy`.

## Copyright

All AIPs, and this repository's documentation, are released under
[CC0](LICENSE).
