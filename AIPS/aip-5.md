---
aip: 5
title: Model Routing Peers
description: Defines routing as a paid seller service with pluggable protocols, where a router ranks the destinations a buyer allows and the buyer keeps control of dispatch.
author: Marco De Rossi (@marcoderossi90), Dawid (@Dawe000), Alexander Ludwig (@alexanderludwig)
discussions-to: https://github.com/AntSeed/antseed/discussions
status: Final
type: Standards Track
category: Core
created: 2026-08-27
---

## Abstract

This AIP adds model routing to Antseed: a seller can offer a service that, given
a buyer's conversation and a list of destinations the buyer allows, returns
those destinations ranked best first instead of serving inference itself. The
buyer sends the inference to the top destination and pays that seller normally.
Later destinations are fallbacks.

Routing is a service category, not one API. Each routing API is its own service
API protocol, advertised and billed like any other service. The first protocol
is `irp-suggest`, the suggest-only mode of the
[Inference Routing Protocol](https://github.com/inference-routing/spec/blob/main/SPEC.md)
(IRP). A ranking is billed as one `completed_requests` unit through the existing
payment flow, only when it names at least one destination the buyer offered.

No router and no routing protocol is a default. This AIP makes no smart contract
change, adds no payment message, and leaves the buyer-side `Router` plugin
interface as it is. A buyer that selects no router behaves exactly as before.

## Motivation

Antseed lets a buyer pick a model and then finds the best seller of that exact
model. It has no mechanism for a service that recommends *which model* to use,
weighing quality against cost across the catalogue and across sellers. That
recommendation only has value if it sees the network's live prices, which the
buyer's client already discovers, and per-conversation state, such as which
seller already holds a warm cache of this conversation's prefix, which the
recommending party cannot observe without the buyer's help.

A seller is a natural place for this role: it is already discoverable and already
paid through the network. Treating routing as an ordinary service means it needs
no new discovery mechanism, no reserved seller path, and no routing-specific
payment code.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

The complete wire format of `irp-suggest`, with examples, is in
[`docs/protocol/model-routing.md`](https://github.com/AntSeed/antseed/blob/main/docs/protocol/model-routing.md).
Its billing is in
[`docs/protocol/unit-billing-services.md`](https://github.com/AntSeed/antseed/blob/main/docs/protocol/unit-billing-services.md).
This section states the rules every implementation shares.

### Routing protocols

A routing protocol is a service API protocol listed in `ROUTING_PROTOCOLS`.
Billing metadata encodes protocols by list position, so new protocols MUST be
appended and existing ones MUST NOT be reordered. Adding a protocol means adding
its wire binding, a buyer adapter and a provider plugin that advertises it. It
requires no change to this AIP's shared rules.

### Advertisement and discovery

A routing seller advertises a service in its signed peer metadata, the same way
it advertises any other service: the service id, the routing protocol in
`serviceApiProtocols`, and a `completed_requests` price in
`serviceUnitBillingModels`:

```typescript
services: ['alpha-route'],
serviceApiProtocols: { 'alpha-route': ['irp-suggest'] },
serviceUnitBillingModels: {
  'alpha-route': {
    'irp-suggest': { version: 1, components: [{ unit: 'completed_requests', priceUsd: 0.001 }] },
  },
},
```

A buyer finds routers through its existing discovery by filtering services on
their API protocol. This AIP adds no capability string, no DHT topic and no
metadata version.

IRP paths name no service, so `irp-suggest` requires one service per seller. A
node MUST refuse to start with two services of such a protocol, and a buyer MUST
NOT select a peer that advertises two.

### Endpoints (`irp-suggest`)

| Endpoint | Cost | Purpose |
| --- | --- | --- |
| `GET /v1/routing/models` | Free | Models the router can rank (IRP §4) |
| `POST /v1/routing/rank` | One completed request | Candidates ranked best first, for one user turn (IRP §5) |

Requests are plain IRP: they carry no Antseed headers or body fields, and seller
identity stays on the buyer. The one Antseed addition is the optional
`max_pricing` member on model-list entries, in USD per million tokens. An entry
may be the wildcard `*`, meaning the router scores any model the buyer sends. IRP
clients ignore unknown members, so the format stays IRP compatible. IRP proxy mode
is not used.

### Candidates

The buyer MUST send only destinations its own trust, price, allow/block and pin
filters allow. The same model from two sellers is two candidates.

- `id` is `provider:model@peer`, or `sha256:` plus the hex SHA-256 of that string
  when it is longer than 128 characters.
- `model` is the router's own name for the model, from its model list. A seller's
  service id matches a listed model exactly or by canonical model key. The buyer
  still sends the seller's service id with the inference request.
- `pricing` is the seller's price in USD per million tokens. A seller with no
  cached-input price bills cache reads at the input price, so `cache_read` MUST
  equal `input` in that case.
- `expected_usage.cache_read_tokens` is the buyer's estimate of prompt tokens the
  destination can serve from cache, from earlier turns of the conversation. It is
  omitted when nothing was observed. A router is not expected to track it.

A buyer MUST NOT send a candidate through a model entry whose `max_pricing` the
candidate's price exceeds.

The rank `request` is the inference request as an OpenAI Chat Completions body.
Other request formats are converted for the ranking only, and the buyer still
sends the original request to the chosen seller. `routing.cost_quality_tradeoff`
is a router option, an integer from `0` (best quality) to `10` (cheapest). It is
omitted when the buyer has not chosen one, and the router then uses IRP's default
of `5`.

### Rankings

The buyer MUST map each returned `candidate_id` through the candidates it sent and
MUST NOT parse it. It MUST drop unknown and duplicate ids, and MUST reject a
ranking if none remain. A router therefore cannot name a destination the buyer
filtered out.

The buyer sends the inference to the first ranked candidate and MAY fall back to
later candidates on retryable inference errors, without paying for another
ranking. Tool-loop continuations of the same user turn SHOULD reuse the ranking
(IRP §7). Optional predictions (`expected_quality`, `expected_cost_usd`,
`expected_usage`, `reasoning_effort`) are kept when well-formed.

### Errors (`irp-suggest`)

Routers answer errors with IRP problem details (`application/problem+json`,
RFC 9457). The seller node uses the same format for the free models endpoint, and
answers it with a 404 problem when it offers no routing service.

| Status | `type` | Buyer action |
| --- | --- | --- |
| 400 | `urn:irp:problem:invalid-request` | Fail the turn |
| 402 | `urn:irp:problem:payment-required` | Existing payment negotiation |
| 422 | `urn:irp:problem:no-scorable-candidate` | Refresh the model list and retry once, then fail the turn |
| 503 | `urn:irp:problem:unavailable` | Fail the turn; `Retry-After` may be set |

### Billing

One `completed_requests` unit is charged per well-formed ranking that names at
least one candidate the buyer sent. Errors and malformed rankings are not billed.
Buyer and seller MUST apply the same billing check to the same response bytes, so
they agree on the charge. Routing uses the existing unit-billing and payment flow:
this AIP adds no `MessageType` value, EIP-712 struct or Solidity contract. A
ranking and the inference it leads to are separate purchases, and a ranking is not
refunded when the inference fails.

### Buyer selection

A router MUST be selected by the buyer. Implementations SHOULD offer a default
route, a per-conversation selection, and pinning a router as the model
(`"model": "<routerPeerId>@<routerServiceId>"`). In the reference buyer proxy a
default router is set with `POST /_antseed/route`
(`{ "router": { "service", "options"?, "allowedModels"? } }`), and
`GET /v1/models?type=router` lists each router's `protocol` and `options`. These
are local client APIs, not part of the peer protocol.

Router options travel in the protocol's own terms. A buyer MUST keep them opaque
in transport and MUST check them against the option specs of the router's protocol
before buying a ranking. Before paying for a ranking, a pinned router MUST pass
the same allow/block, trust and reputation checks as a pinned seller.

### `Router` plugin interface

The buyer-side `Router` plugin interface (`selectPeer`, `onResult`) is unchanged.
Routing services run beside it, in the buyer's routing path, so a `Router` plugin
needs no change. Router code never receives direct access to the buyer's signer or
payment manager.

## Rationale

**Routing as an ordinary service.** An earlier version of this AIP defined a
`routing.v1` capability string, a reserved `/_antseed/route` seller path, and new
optional methods on the `Router` plugin, with payment left to a companion AIP. The
implementation shows none of those are needed. Services already give discovery,
signed advertisement, per-service pricing and the existing payment flow, so a router
is a provider plugin and a buyer needs one routing client. `/_antseed/route` remains
in use only as a local buyer-proxy API.

**An open wire format.** IRP is an open protocol, so any IRP router can be offered
through Antseed unchanged, and Antseed adds only one optional member that IRP clients
ignore. An Antseed-specific schema would require every router to implement a format
used nowhere else.

**Pluggable protocols, no default.** Making IRP one protocol rather than the standard
lets other routing APIs join without reopening this AIP, the same way video APIs are
separate protocols. Having no default keeps the network from privileging one router.

**Bill only usable rankings.** A router is paid for an answer the buyer can use. A
ranking naming no offered candidate has no value and costs nothing.

**The buyer owns the candidate set.** Sending only allowed candidates, and mapping
answers back through them, enforces buyer policy by construction, instead of asking
a remote party to apply it.

## Backwards Compatibility

The proposal is additive for buyers that select no router and for sellers that offer
none. Routing services use the unit-billing metadata introduced at peer metadata
v11 and require no metadata version of their own. A buyer too old to know
`completed_requests` rejects the whole announcement of a seller that advertises it,
so an operator SHOULD run a routing service on a separate peer until those buyers
have upgraded.

No `routing.v1` capability, reserved seller path or `Router` extension from the
earlier draft ever shipped, so nothing on the network depends on them. The earlier
`/v1/routing/describe` and `recommendations` format from pre-release branches has no
compatibility layer.

## Test Cases

In the [antseed repository](https://github.com/AntSeed/antseed):

- `packages/protocol/src/irp.test.ts`, `packages/protocol/src/routing.test.ts`: wire
  validation and billable-ranking checks.
- `packages/router-core/src/model-routing-client.test.ts`: the buyer client.
- `apps/cli/src/proxy/router-execution.test.ts`, `apps/cli/src/proxy/routing-services.test.ts`,
  `apps/cli/src/gateway/routing-through-gateway.test.ts`: candidate building, dispatch and
  selection.
- `e2e/tests/routing-payment-flow.test.ts`: a paid ranking through the payment flow.

## Reference Implementation

Merged in [Antseed/antseed#1232](https://github.com/Antseed/antseed/pull/1232) and
[#1233](https://github.com/Antseed/antseed/pull/1233):

- `packages/protocol/src/routing.ts`, `packages/protocol/src/irp.ts`: protocols,
  wire binding and validators.
- `packages/router-core/src/model-routing-client.ts`,
  `packages/router-core/src/routing-adapters/`: protocol-neutral buyer client and
  the IRP adapter.
- `apps/cli/src/proxy/router-execution.ts`: candidate building and dispatch.
- `plugins/provider-irp-router`, `plugins/provider-free-model-router`: seller plugins.

Not implemented: IRP proxy mode, and applying `reasoning_effort` to the inference
request (it is carried with the recommendation only).

## Security Considerations

**Prompt content leaves the buyer's device.** The router receives the whole
conversation on every paid ranking, in addition to the seller that answers it. Two
parties see the prompt instead of one. A buyer SHOULD only select a router it would
trust with that content, and SHOULD be able to see which router it is using. Selecting
no router sends nothing to one.

**A dishonest router can steer, but cannot override policy.** A router can bias its
order toward affiliated sellers. It cannot name a destination outside the set the buyer
offered, so it cannot make a buyer pay outside its own price, trust or allow/block
bounds. A router also sees each candidate's price and id, and can use them to learn per
seller. Buyers choose routers, and can leave one, as they do with sellers.

**Spend per ranking.** A ranking is a separate purchase at the router's advertised
price, and routing charges count toward the conversation's spend. Token and image price
caps do not apply to a router, and there is no per-key cap on the ranking price yet, so
a buyer selecting a router accepts its advertised price. Reusing a ranking across a
tool loop bounds repeat charges within one user turn.

**Denial of service.** The free model list is not rate-limited by the seller node; the
router limits its own list. Ranking is billed, and the seller's normal payment checks
apply before it runs. The buyer caches the model list for 60 seconds.

**Keys.** Routing adds no signing mechanism. Rankings are paid through the buyer's
existing payment manager, and router code gets no handle on the signer.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE).
