---
aip: 6
title: Referrals and Client Attribution
description: Attributes recognized usage to client software and invite-based referrals through a buyer-signed metadata tail, and funds both from new emission buckets.
author: Shahaf Antwarg (@kotevcode)
discussions-to: https://github.com/Antseed/antseed/pull/1080
status: Draft
type: Standards Track
category: Contracts
created: 2026-10-04
requires: 2
---

## Abstract

This AIP extends the recognized-usage system of [AIP-2](./aip-2.md) with two
new reward surfaces: **client attribution**, which rewards the software
(Desktop, CLI, third-party apps) that produced a buyer's usage, and
**two-sided referrals**, which reward both the wallet that invited a buyer and
the invited buyer.

Buyers append a five-word **attribution tail** to the settlement metadata they
already sign (SpendingAuth v1/v2/v3 and FreeUsage v1). The tail carries a
`clientId` (an ERC-8004 agent id) and an optional single-use **invite**: an
EIP-712 signature by the referrer over `Invite(uint256 issuedEpoch,uint256 index)`.
Because the tail is covered by the buyer's signature, neither the seller nor a
relayer can forge or strip it. `AntseedStatsV2` decodes the tail on every
settlement and forwards it, best effort, to `AntseedAttributionUsage` (a
per-epoch ledger of recognized weighted points by client, referrer and
referee) and `AntseedReferrals` (bindings and invites).

Two new `AntseedEmissionsGate` controllers, `AntseedReferrals` and
`AntseedClientRewards`, each split their epoch bucket pro rata to the ledger's
points. Invites are earned by recognized activity, expire, are single-use, and
can only bind new buyers. Points are always AIP-2 recognized points, never raw
USDC volume, so wash exclusion and pool weighting carry over unchanged.

## Motivation

AIP-2 rewards buyers, seller operators and seller-pool stakers. It has no way
to reward the two parties that grow demand without being on either side of a
channel:

- **Client builders.** The software a buyer uses decides which network the
  buyer's requests go to. A third-party app that routes its users through
  Antseed has no on-chain identity in a settlement and earns nothing from the
  usage it brings.
- **Referrers.** A wallet that brings a new buyer has no verifiable way to
  claim that it did.

Both need attribution that is deterministic, cannot be claimed by an outsider,
works for free usage and for the CLI as well as the desktop app, and does not
add gas or a new transaction for the buyer. Settlement metadata is the one
place that satisfies all of these: the buyer already signs it for every
settlement, the seller already submits it on-chain, and Antseed already has a
stats sink that receives it.

Referral rewards are also an obvious target for self-dealing. A referral
scheme that lets any wallet name any other wallet as its referrer pays people
to refer themselves. This AIP therefore makes the referrer prove consent (it
signs the invite), makes invites scarce in proportion to the referrer's real
recognized activity, and limits who can be referred and for how long the
referred side is paid.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

Epochs, the emissions gate, minter buckets, usage accounting, weighted points
and the reserve and burn remainder path are as defined in AIP-2. "Recognized
points" means points recorded by `AntseedUsageAccounting`. "Operator" means
the address returned by `AntseedDeposits.getOperator(account)`, zero when
unset.

### Architecture overview

```text
buyer signs SpendingAuth / FreeUsageAuth over metadataHash
  (metadata = existing head + services + attribution tail)
        |
seller settles -> AntseedChannels / AntseedFreeUsage
        |  try recordMetadata(...)            (before accrue*Points)
        v
AntseedStatsV2 -- decodes tail --+--> AntseedAttributionUsage.record(buyer, clientAgentId)
                                 |       per-epoch client / referrer / referee points
                                 +--> AntseedReferrals.bindReferral(buyer, invite)   (if invite present)
                                           bindings, invite quota and single-use bitmap

AntseedEmissionsGate bucket --> AntseedReferrals      (referrer + referee claims)
AntseedEmissionsGate bucket --> AntseedClientRewards  (client agent owner claims)
```

### Settlement metadata attribution tail

#### Layout

The tail is five static 32-byte words inserted into the ABI head of the
existing metadata, between the services-array offset word and the services
array itself. Let `S` be the number of static head words before the
services-array offset word:

| Metadata | Version word | Static head words | `S` |
| --- | --- | --- | --- |
| SpendingAuth v3 | `3` | version, cumulativeInputTokens, cumulativeOutputTokens, cumulativeRequestCount, cumulativeOutputImages | 5 |
| SpendingAuth v1 / v2 | `1` / `2` | version, cumulativeInputTokens, cumulativeOutputTokens, cumulativeRequestCount | 4 |
| FreeUsage v1 | `1` | version, cumulativeInputTokens, cumulativeOutputTokens, cumulativeRequestCount | 4 |

With a tail, the head is:

| Head word | Type | Field |
| --- | --- | --- |
| `0 .. S-1` | `uint256` | existing static words |
| `S` | `uint256` | services-array offset, MUST equal `(S + 6) * 32` |
| `S + 1` | `bytes32` | `clientId`: ERC-8004 agent id of the client software as a big-endian word; zero means none |
| `S + 2` | `uint256` | `inviteEpoch`: the invite's `issuedEpoch` |
| `S + 3` | `uint256` | `inviteIndex`: the invite's `index` |
| `S + 4` | `bytes32` | `inviteR`: EIP-2098 compact signature `r` |
| `S + 5` | `bytes32` | `inviteVs`: EIP-2098 compact signature `vs` |

followed by the services array (length word, then elements) exactly as
without a tail. In ABI terms the metadata is:

```solidity
abi.encode(<static head words>, services, bytes32 clientId, uint256 inviteEpoch,
           uint256 inviteIndex, bytes32 inviteR, bytes32 inviteVs)
```

Without a tail the services offset is `(S + 1) * 32`, as today. Offsets are
absolute, so a decoder that stops at the services array reads identical
values with or without a tail.

#### Encoding rules for clients

- A buyer client SHOULD include the tail in every SpendingAuth and FreeUsage
  metadata it signs.
- `clientId` MUST be the client's ERC-8004 agent id, or zero when the client
  has none.
- "No invite" MUST be encoded as `inviteR = 0` and `inviteVs = 0`. Clients
  SHOULD also send `inviteEpoch = 0` and `inviteIndex = 0` in that case.
- A client holding an unused invite for an unbound buyer SHOULD carry it in
  the tail of every settlement until the binding lands, and MUST send the
  no-invite encoding once the buyer is bound. A bound buyer MAY keep the tail
  for `clientId` attribution.

#### Decoding rules

A conforming decoder (`AntseedStatsV2.decodeAttribution`) MUST:

1. Return an all-zero attribution if the metadata is shorter than 32 bytes.
2. Set `S = 5` when the first word equals `3`, and `S = 4` otherwise.
3. Return all zeros if the metadata is shorter than `S * 32 + 7 * 32` bytes
   (the head with the tail plus the services length word).
4. Return all zeros unless head word `S` equals `(S + 6) * 32`.
5. Otherwise read the five tail words from head words `S + 1 .. S + 5`.

Any other shape binds and credits nothing beyond a zero `clientId`. In
particular, metadata without a tail, and the retired two-word tail
`(address referrer, bytes32 clientId)` that named a referrer directly, decode
to all zeros.

### AntseedStatsV2

`AntseedStatsV2` replaces `AntseedStats` behind `AntseedRegistry.stats()`. It
keeps the writer interface (`recordMetadata(agentId, buyer, channelId,
metadata)`, callable only by authorized writers, which are `AntseedChannels`
and `AntseedFreeUsage`) and the per-agent, per-buyer token statistics of
`AntseedStats`.

On every `recordMetadata` call, before any token-statistics processing,
`AntseedStatsV2` MUST:

1. Decode the attribution tail.
2. If an attribution ledger is configured, call
   `AntseedAttributionUsage.record(buyer, uint256(clientId))` inside
   `try`/`catch`, including when `clientId` is zero, and emit
   `ClientForwarded(buyer, clientAgentId, recorded)`.
3. If `inviteR != 0` or `inviteVs != 0` and a referrals contract is
   configured, call `AntseedReferrals.bindReferral(buyer, inviteEpoch,
   inviteIndex, inviteR, inviteVs)` inside `try`/`catch`, and emit
   `InviteForwarded(buyer, inviteEpoch, inviteIndex, bound, reason)`, where
   `reason` is the first four bytes of the revert data when the bind was
   rejected and zero otherwise.

The ledger forward MUST happen on every settlement, regardless of whether the
token counters are valid or monotonic, because the ledger's per-buyer cursor
(below) relies on observing every settlement. A rejected forward MUST NOT
revert `recordMetadata`. The owner MAY set either sink to zero to disable it.

`AntseedChannels` MUST call the stats sink before
`IAntseedEmissions.accrueSellerPoints` / `accrueBuyerPoints` for the same
settlement. The deployed `AntseedChannels._settleSpend` already does so.

### AntseedAttributionUsage

`AntseedAttributionUsage` is a facts-only ledger that attributes each buyer's
recognized **weighted** points (`AntseedUsageAccounting.buyerUsageTotal(buyer).weightedPoints`,
the same points AIP-2 uses for buyer rewards, after points policies and pool
weighting) to three parties per epoch:

- the client agent id named in the buyer's metadata;
- the buyer's referrer, once bound;
- the buyer itself as **referee**, during its referee window.

USDC volume and token counts MUST NOT enter the ledger.

#### Cursor

The ledger keeps one cursor per buyer: the last observed cumulative weighted
points, the client of the most recent settlement, and the epoch that
settlement was recorded in. `record(buyer, clientAgentId)` MUST:

1. Revert unless called by the configured `recorder` (`AntseedStatsV2`), and
   revert for a zero buyer.
2. Read the buyer's cumulative weighted points `current`. If the cursor is
   initialized, `delta = current - previous` (zero if not greater). The first
   observation of a buyer only sets the baseline: usage that predates the
   ledger is credited to nobody.
3. If not paused, credit `delta` to the cursor's previous client and epoch,
   and to the buyer's referral (below). If paused and `delta != 0`, drop
   `delta` and emit `UsageDroppedWhilePaused(buyer, delta)`. The cursor MUST
   still advance while paused.
4. Set the cursor's client to `clientAgentId` if it is non-zero, fits in
   `uint64`, and `identityRegistry.ownerOf(clientAgentId)` returns a non-zero
   owner without reverting; otherwise to zero (unattributed).
5. Set the cursor's epoch to `max(currentEpoch, firstRewardedEpoch)`.

Because Stats runs before the accounting accrues the current settlement, the
growth observed at a call is exactly the points of the settlements since the
previous call, and it is credited to the client and epoch recorded at that
previous call. A buyer's latest settlement is therefore credited at the
buyer's next settlement, or earlier by anyone through
`flush(address[] buyers)`, which applies the same credit to initialized
cursors and does not move their client or epoch. `flush` MUST revert while
paused.

`pendingCredit(buyer)` MUST return the not-yet-credited points with the
client and credit epoch they would land in if flushed now.

#### Credit epoch

Reward controllers freeze an epoch's total at first claim, so credits MUST
only land in epochs that are not yet claimable. With
`CREDIT_GRACE_EPOCHS = 1` (equal to `AntseedEpochShareRewards.SETTLEMENT_GRACE_EPOCHS`):

```text
oldestOpenEpoch = max(firstRewardedEpoch, currentEpoch > 1 ? currentEpoch - 1 : 0)
creditEpoch     = max(cursorEpoch, oldestOpenEpoch)
```

A trailing settlement observed after its epoch closed rolls forward into the
oldest open epoch.

#### Client credit

If the cursor's client is zero, `delta` MUST be added to
`unattributedClientPointsByEpoch[epoch]` and MUST NOT count toward
`totalClientPointsByEpoch`. Otherwise it MUST be added to
`clientEpochPoints[epoch][client]`, `totalClientPointsByEpoch[epoch]` and
`clientTotalPoints[client]`.

#### Referral credit

If a referrals contract is configured and `referralOf(buyer)` returns a
non-zero `referrer` bound at `boundAtEpoch`:

1. If the buyer has a non-zero operator `op`, and either `referrer == op` or
   `getOperator(referrer) == op`, nothing MUST be credited. This re-applies
   the bind-time self-referral guard, because operators are commonly set or
   changed after binding.
2. Otherwise `delta` MUST be credited to the referrer
   (`referrerEpochPoints`, `totalReferrerPointsByEpoch`, `referrerTotalPoints`).
3. If the epoch the usage was recorded in (the cursor epoch, not the
   rolled-forward credit epoch) is at most `boundAtEpoch + REFEREE_BONUS_EPOCHS`,
   `delta` MUST also be credited to the buyer as referee
   (`refereeEpochPoints`, `totalRefereePointsByEpoch`, `refereeTotalPoints`).

`REFEREE_BONUS_EPOCHS = 12`: the referee window is the binding epoch plus the
12 epochs after it. `totalReferralPointsByEpoch(epoch)` MUST return
`totalReferrerPointsByEpoch[epoch] + totalRefereePointsByEpoch[epoch]`.

#### First usage

The ledger MUST record each buyer's first recognized usage epoch. When the
buyer's cumulative weighted points leave zero for the first time, the first
usage epoch is the cursor's epoch if the cursor was already initialized, and
`0` if this is the buyer's first observation (usage that predates the ledger).
First usage MUST be recorded even while the ledger is paused.
`firstUsageEpoch(buyer)` returns `(seen, epoch)`.

#### Events

```solidity
event ClientUsageCredited(uint256 indexed epoch, uint256 indexed clientAgentId, address indexed buyer, uint256 points);
event UnattributedClientUsage(uint256 indexed epoch, address indexed buyer, uint256 points);
event ReferrerUsageCredited(uint256 indexed epoch, address indexed referrer, address indexed buyer, uint256 points);
event RefereeUsageCredited(uint256 indexed epoch, address indexed referee, address indexed referrer, uint256 points);
event FirstUsageRecorded(address indexed buyer, uint256 indexed epoch);
event UsageDroppedWhilePaused(address indexed buyer, uint256 points);
event RecorderUpdated(address indexed recorder);
event ReferralsUpdated(address indexed referrals);
```

The constructor MUST revert if the identity registry address has no code,
since `ownerOf` is called inside `try`/`catch` and a call to a code-less
address would revert on return-data decoding outside the `catch`.

### Emission buckets

#### New controllers

This AIP adds two `AntseedEmissionsGate` minters, each a controller that
manages its own bucket as defined in AIP-2:

- **Referral bucket**, controller `AntseedReferrals`, paying referrers and
  referees.
- **Client (builders) bucket**, controller `AntseedClientRewards`, paying
  client agent owners.

The controllers pay nothing until the gate owner registers them with
`setMinter(id, controller, shareBps, editable)`. Per AIP-2, a new minter's
share takes effect from the next epoch and all shares MUST sum to at most
`SHARE_DENOMINATOR = 100000`.

The reference implementation does not define the minter ids or the share of
either bucket; they are set through the gate's share configuration at
registration. See [Open Issues](#open-issues).

#### Epoch share (AntseedEpochShareRewards)

Both controllers inherit `AntseedEpochShareRewards`, which MUST implement:

- **Claimability.** With `SETTLEMENT_GRACE_EPOCHS = 1`, epoch `e` is claimable
  when `e + 1 < emissionsGate.currentEpoch()`, that is, once one further
  epoch has fully elapsed. This matches the ledger's `CREDIT_GRACE_EPOCHS`, so
  no credit can land in an epoch after it becomes claimable.
- **Frozen total.** On the first claim (or remainder settlement) for an epoch,
  the controller MUST freeze that epoch's ledger total and emit
  `EpochTotalFrozen(epoch, totalPoints)`. Every later claim for that epoch
  uses the frozen total.
- **Pro rata share.** A claimant with `points` receives
  `budget * points / total`, where `budget = emissionsGate.controllerEpochBudget(controller, epoch)`,
  capped at the budget not yet minted for the epoch. A lone claimant takes
  the whole bucket; each additional claimant dilutes it. There is no
  per-claimant cap.
- **Single claim.** Each claim key MUST be claimable at most once per epoch.
  A claim MUST revert with `EpochNotClaimable`, `AlreadyClaimed`,
  `InvalidAddress` (zero recipient) or `NothingToClaim` (zero amount).
- **Permissionless claims.** Anyone MAY trigger a claim; the recipient is
  determined by the controller, never by the caller.
- **Remainder sweep.** `settleEpochRemainder(epoch)` is permissionless. It
  MUST revert unless the epoch is claimable, not already settled, its frozen
  total is zero and its budget is non-zero. It then routes the whole budget
  through `emissionsGate.claimRemainder(epoch, emissionsReserve, budget)`,
  which applies AIP-2's burn cap before sending the rest to the reserve.
  Epochs with at least one claimant keep their bucket for those claimants;
  integer-division dust in such epochs is not swept.
- **Pause.** The owner MAY pause; claims and remainder settlement MUST revert
  while paused.

### Two-sided referrals (AntseedReferrals)

#### Points and split

Each epoch's referral bucket is split pro rata to
`AntseedAttributionUsage.totalReferralPointsByEpoch(epoch)`. A referred
buyer's points are credited once to its referrer and, during the referee
window, once more to the buyer as referee, so while the buyer is in its
window the referrer and the referee each earn half of that buyer's share of
the bucket. After the window the referrer earns all of it.

#### Claims and recipients

- `claim(referrer, epoch)` / `claimEpochs(referrer, epochs)` pay the
  referrer's share to `referrerRecipient(referrer)`: the referrer's operator
  if non-zero, else the referrer address itself (sellers and plain wallets).
- `claimReferee(buyer, epoch)` / `claimRefereeEpochs(buyer, epochs)` pay the
  referee's share to the buyer's operator only. If the buyer has no operator
  the claim MUST revert with `RewardRecipientUnavailable` without marking the
  epoch claimed, so it can be retried once an operator is set.
- Referrer and referee claims MUST use distinct, role-tagged claim keys
  (`keccak256(abi.encode(keccak256("antseed.referrals.referrer"), account))`
  and `keccak256(abi.encode(keccak256("antseed.referrals.referee"), account))`),
  so one wallet may hold both roles.
- The batch variants skip epochs with nothing to pay and revert with
  `NothingToClaim` only if the whole batch paid nothing.

A buyer hot wallet MUST NOT receive referee rewards; this is the same rule
AIP-2 applies to buyer rewards.

```solidity
event ReferralBound(address indexed buyer, address indexed referrer, uint256 epoch, uint256 inviteEpoch, uint256 inviteIndex);
event ReferralRewardClaimed(address indexed referrer, uint256 indexed epoch, address indexed recipient, uint256 points, uint256 totalPoints, uint256 amount);
event RefereeRewardClaimed(address indexed referee, uint256 indexed epoch, address indexed recipient, uint256 points, uint256 totalPoints, uint256 amount);
event BinderUpdated(address indexed binder);
```

### Activity-based single-use invites

#### Signature

An invite is issued off-chain, without gas, by the referrer signing an
EIP-712 message in the `AntseedReferrals` domain:

```text
EIP712Domain(name = "AntseedReferrals", version = "1", chainId, verifyingContract = AntseedReferrals)
Invite(uint256 issuedEpoch,uint256 index)
INVITE_TYPEHASH = keccak256("Invite(uint256 issuedEpoch,uint256 index)")
               = 0x787424ddbd067db314529d6126aaa36d95cddd0034c3b0321ba23f851eaa52a8
```

The signature MUST be the 64-byte EIP-2098 compact form `(r, vs)`. The
referrer is recovered from the signature; it is never stated. The shareable
invite is `(issuedEpoch, index, r, vs)`. A malformed signature recovers to
the zero address.

#### Quota

Invites are earned by recognized activity in the previous epoch, which is
final once the issue epoch has started:

```text
MIN_ACTIVE_POINTS       = 1e6     (1 USDC)
BASE_INVITES            = 3
POINTS_PER_EXTRA_INVITE = 10e6    (10 USDC)
MAX_INVITES_PER_EPOCH   = 20

activity(r, E)    = usageAccounting.buyerPointsByEpoch(E - 1, r)
                  + usageAccounting.sellerPointsByEpoch(E - 1, r)
inviteQuota(r, E) = 0                                                  if E == 0 or r == 0
                  = 0                                                  if activity < MIN_ACTIVE_POINTS
                  = min(MAX_INVITES_PER_EPOCH,
                        BASE_INVITES + activity / POINTS_PER_EXTRA_INVITE)  otherwise
```

These are recognized points after points policies but before pool
weighting, in USDC base units. `sellerPointsByEpoch` is the points of the
seller pool (agent id) the address settled as a seller in that epoch. Any
real activity yields 3 invites, one more per 10 USDC, capped at 20 (reached
at 170 USDC).

#### Validity and single use

- `INVITE_VALIDITY_EPOCHS = 4`: an invite is active while
  `issuedEpoch <= currentEpoch < issuedEpoch + 4`, counting its issue epoch.
- Each `(referrer, issuedEpoch, index)` MUST bind at most once, tracked in a
  per-referrer, per-epoch 256-bit bitmap. `inviteUsed` MUST return false for
  `index >= 256`.

#### New-buyer rule

`NEW_BUYER_EPOCHS = 2`. A buyer may bind only if
`AntseedAttributionUsage.firstUsageEpoch(buyer)` reports no usage yet, or a
first usage epoch `f` with `f + 2 >= currentEpoch`.

#### Binding

`bindReferral(buyer, issuedEpoch, index, r, vs)` MUST revert with `NotBinder`
unless called by the configured `binder` (`AntseedStatsV2`), and MUST revert
while paused. It MUST then run the checks below in exactly this order and
revert with the first failing error:

1. `buyer == 0` → `InvalidAddress`
2. buyer already bound → `ReferralAlreadyBound`
3. recovered referrer is zero → `InvalidInviteSignature`
4. self-referral → `SelfReferral`, where self-referral means any of:
   - `referrer == buyer`;
   - the buyer has a non-zero operator `op` and `referrer == op`;
   - the buyer has a non-zero operator `op` and `getOperator(referrer) == op`
     (referrer and buyer share the same non-zero operator);
5. invite not active → `InviteNotActive`
6. `index >= inviteQuota(referrer, issuedEpoch)` → `InviteOverQuota`
7. invite already used → `InviteAlreadyUsed`
8. buyer fails the new-buyer rule → `NotNewBuyer`

On success it MUST mark the invite used, set `referrerOf[buyer]` and
`boundAtEpoch[buyer] = currentEpoch`, increment `referredCount[referrer]`, and
emit `ReferralBound`. A binding is immutable.

A rejected bind reverts inside `AntseedReferrals`, so it MUST NOT consume the
invite, and `AntseedStatsV2` swallows the revert, so it MUST NOT revert the
settlement that carried it.

`previewInvite(buyer, issuedEpoch, index, r, vs)` MUST run the same checks as
a view and return `(referrer, failure)`, where `failure` is the selector the
bind would revert with, or zero if it would bind. It does not apply the
binder or pause checks.

| Error | Selector |
| --- | --- |
| `InvalidAddress()` | `0xe6c4247b` |
| `ReferralAlreadyBound()` | `0x3cdfced9` |
| `InvalidInviteSignature()` | `0x17431f13` |
| `SelfReferral()` | `0x55e8f70e` |
| `InviteNotActive()` | `0xa93c22da` |
| `InviteOverQuota()` | `0xe11f0658` |
| `InviteAlreadyUsed()` | `0x4209eb7d` |
| `NotNewBuyer()` | `0xdc26651f` |
| `NotBinder()` | `0x52324c6a` |
| `EnforcedPause()` (paused) | `0xd93c0665` |

### Client (builder) rewards (AntseedClientRewards)

A client is identified by an ERC-8004 agent id in the deployed
IdentityRegistry. A client builder registers an agent and configures its
software to put that id in the `clientId` tail word.

Each epoch's client bucket is split pro rata to
`AntseedAttributionUsage.clientEpochPoints(epoch, clientAgentId)` over
`totalClientPointsByEpoch(epoch)`. Unattributed points are not part of the
denominator.

`claim(clientAgentId, epoch)` is permissionless. It MUST pay the current
`identityRegistry.ownerOf(clientAgentId)` at claim time, and MUST revert with
`InvalidAddress` if the agent has no owner or `ownerOf` reverts. It emits
`ClientRewardClaimed(epoch, clientAgentId, recipient, points, totalPoints, amount)`.
The constructor MUST revert if the identity registry has no code.

### Deployment and wiring

Deployment MUST follow this order:

1. Deploy `AntseedStatsV2` and authorize `AntseedChannels` and
   `AntseedFreeUsage` as writers.
2. Deploy `AntseedAttributionUsage(usageAccounting, identityRegistry, deposits, recorder = StatsV2)`.
3. Deploy `AntseedReferrals(emissionsGate, usageAccounting, deposits, attributionUsage, binder = StatsV2)`
   and `AntseedClientRewards(emissionsGate, attributionUsage, identityRegistry)`.
4. Call `StatsV2.setAttributionUsage`, `StatsV2.setReferrals` and
   `AntseedAttributionUsage.setReferrals`.
5. Register both controllers as gate minters.
6. Re-point `AntseedRegistry.setStats(StatsV2)`. Channels and FreeUsage
   resolve the stats address live.

The sinks MUST be wired before the registry is re-pointed: settlements Stats
forwards while a sink is unset are never attributed.

### Open Issues

1. **Bucket ids and shares.** The repository defines no minter ids and no
   share values for the referral and client buckets, and the AIP-2 default
   shares already sum to 100%. Registering these buckets therefore requires
   choosing their ids, their shares and which existing editable buckets give
   up share. These values are undecided.
2. **Reference client encoder.** The off-chain encoder in
   `packages/protocol/src/signatures.ts` still emits the retired two-word
   tail `(address referrer, bytes32 clientId)`, which `AntseedStatsV2`
   ignores. It MUST be moved to the five-word tail of this AIP, with an
   invite signer, and checked against the test vectors below before clients
   rely on referral binding or client attribution.
3. **Per-claimant cap.** Unlike AIP-2's direct usage rewards, neither new
   controller caps a single claimant's share of an epoch. Whether a cap is
   wanted is undecided.

## Rationale

**Signed metadata as the carrier.** The buyer already signs metadata for
every settlement, so attribution costs no extra transaction, no gas for the
buyer, and works identically in the desktop app, the CLI and third-party
clients, including during free usage. The tail sits in the ABI head so that
its presence is detectable from one offset word, and existing decoders that
stop at the services array keep working.

**Recognized points, not volume.** All credit is derived from AIP-2 recognized
points, so wash exclusion, verification shaping and pool weighting apply to
referral and client rewards without being re-implemented. The ledger reads
cumulative points through a per-buyer cursor rather than changing
`AntseedUsageAccounting`, which keeps the accounting contract and its
settlement path untouched.

**Best-effort forwarding.** Every forward is wrapped in `try`/`catch` at two
levels (Channels to Stats, Stats to each sink), so a bad invite, a paused
contract or a misconfiguration can lose attribution but cannot block USDC
settlement. The cursor still advances while paused so that a later call
cannot hand a backlog to a stale client or epoch.

**Two-sided, time-boxed referee share.** Paying the referred buyer gives a
reason to accept an invite. Bounding that share to 12 epochs bounds what a
self-referral through an undetectable sibling wallet can capture, while the
referrer share persists because that is what any genuine referral earns.

**Activity-based quota and expiry.** Invites are bearer tokens, so their
number has to be bounded. Tying the quota to the referrer's own recognized
activity in the previous epoch makes invites cost real usage, and short
validity limits how long a leaked invite is useful.

**New buyers only.** Without this rule an established wallet could be bound
to an invite at any time to start paying a referrer for usage that would
have happened anyway.

**Alternatives considered and rejected:**

- **Network or IP matching** on download or site visit: probabilistic,
  invites reward farming from shared IPs (offices, carrier NAT, VPNs), and requires
  storing IP-derived data.
- **Confirmation prompts** ("were you referred by X?"): add friction and do
  not prove anything the user could not simply assert.
- **A plain referrer address in metadata** (the retired two-word tail):
  anyone can name any address, including their own second wallet, so it pays
  self-referral by construction.
- **Public or hashed vanity codes:** public codes are discoverable and hashed
  codes are guessable, and neither stops a buyer from signing an arbitrary
  referrer into its metadata.
- **Tagged installers** (per-referrer builds, as Chrome and Firefox do for
  distribution partners): deterministic, but heavy to operate across
  platforms and code-signing, and does not cover the CLI.

A referrer-signed invite is deterministic, proves the referrer's consent,
needs no off-chain service to verify, and works for every client.

## Backwards Compatibility

- Metadata without a tail, and metadata with the retired two-word tail, are
  accepted unchanged and attribute nothing beyond an unattributed settlement.
  Existing SpendingAuth and FreeUsage signature schemes are unchanged; only
  the signed metadata bytes grow.
- Sellers and settlement contracts need no changes. `AntseedChannels` and
  `AntseedFreeUsage` already forward metadata to `registry.stats()`.
- `AntseedStatsV2` replaces `AntseedStats` by re-pointing
  `AntseedRegistry.setStats`. Channels open across the cutover report their
  cumulative token totals once as a first delta in the new contract;
  consumers that sum `MetadataRecorded` deltas MUST net that re-report
  against what they already indexed for the channel.
- Buyers first observed by the ledger are baselined at that observation:
  earlier usage is credited to nobody, and buyers whose first observation
  already shows usage record a first usage epoch of `0`, so they cannot bind
  a referral (unless `currentEpoch <= 2`).
- The deployment order above MUST be followed so that no settlement is
  forwarded to an unset sink.

## Test Cases

The reference test suites are in `packages/contracts/test` of
[Antseed/antseed#1080](https://github.com/Antseed/antseed/pull/1080):

- `AntseedStatsV2.t.sol`: tail detection on v3 and FreeUsage v1 layouts,
  zero result without a tail, the retired tail ignored, forwarding of client
  and invite, best-effort failure handling, and the metadata vector below.
- `AntseedAttributionUsage.t.sol`: cursor crediting to the producing
  client and settlement epoch, baseline on first observation, late binding
  crediting only later usage, roll forward into the oldest open epoch,
  unattributed and unregistered clients, referrer and referee credit, the
  12-epoch referee window, operator and shared-operator guards at credit
  time, first usage (including while paused and for pre-ledger usage), free
  usage moving the cursor, and pause behavior.
- `AntseedReferrals.t.sol`: quota from previous-epoch activity, the
  four-epoch validity window, single use, wrong-key and tampered signatures,
  self-referral including the shared-operator case, the new-buyer rule,
  binder-only binding, pro rata claims with distinct referrer and referee
  keys, recipients, the empty-epoch sweep, and the invite vector below.
- `AntseedClientRewards.t.sol`: pro rata split by recognized points with
  agent-owner payout, the grace epoch before claimability, empty-epoch
  sweep, and the code-less registry guard.
- `AntseedAttributionRewards.t.sol`: end-to-end integration through
  Channels, Stats, the ledger and both controllers, including the 50/50 split
  followed by referrer-only credit, rejected invites that leave settlement
  intact, and one wallet holding both roles.

A conforming implementation MUST reproduce these vectors.

**Invite vector.** Chain id `8453`, `verifyingContract = 0x1111111111111111111111111111111111111111`,
signer private key `0xA11CE` (address `0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7`),
`issuedEpoch = 42`, `index = 7`:

```text
INVITE_TYPEHASH  0x787424ddbd067db314529d6126aaa36d95cddd0034c3b0321ba23f851eaa52a8
domainSeparator  0x7c61f7df600144621a276a74c7d6dcb93fdecf901357c79df42bcca8b8743495
structHash       0x0fd739de67ca2598fb01ecdc42b4dbc8cc4aa98cca50e14e2af44bfcaa6cef83
digest           0x23a154885f1032c2161dc1e68d05535b819dbdeb17a51793c019bdfe30d4b4c9
r                0xd802ee5a16750afbabae3b72ff1d3fd4b0e020078f532d083845ef4abd2c8eda
vs               0x446c5c2a3057d6654b49e84d192c62a787ff15082d4748ae083e366b78e80755
```

**Metadata vector.** SpendingAuth v3 with `cumulativeInputTokens = 1000`,
`cumulativeOutputTokens = 200`, `cumulativeRequestCount = 3`,
`cumulativeOutputImages = 0`, a one-element services array `[5]` (a
`uint256[]` stand-in; Stats never decodes services), `clientId = 42`, and the
invite above. It is 416 bytes (13 words), the services offset is
`0x160 = 11 * 32`, and
`keccak256(metadata) = 0x60bd1b80e739efcf89d9cbd9a1c0e8fe9b562dbcbfb8434e12f799cb1d272d1e`.

```text
word  0  0000000000000000000000000000000000000000000000000000000000000003  version
word  1  00000000000000000000000000000000000000000000000000000000000003e8  cumulativeInputTokens
word  2  00000000000000000000000000000000000000000000000000000000000000c8  cumulativeOutputTokens
word  3  0000000000000000000000000000000000000000000000000000000000000003  cumulativeRequestCount
word  4  0000000000000000000000000000000000000000000000000000000000000000  cumulativeOutputImages
word  5  0000000000000000000000000000000000000000000000000000000000000160  services offset
word  6  000000000000000000000000000000000000000000000000000000000000002a  clientId
word  7  000000000000000000000000000000000000000000000000000000000000002a  inviteEpoch
word  8  0000000000000000000000000000000000000000000000000000000000000007  inviteIndex
word  9  d802ee5a16750afbabae3b72ff1d3fd4b0e020078f532d083845ef4abd2c8eda  inviteR
word 10  446c5c2a3057d6654b49e84d192c62a787ff15082d4748ae083e366b78e80755  inviteVs
word 11  0000000000000000000000000000000000000000000000000000000000000001  services length
word 12  0000000000000000000000000000000000000000000000000000000000000005  services[0]
```

## Reference Implementation

[Antseed/antseed#1080](https://github.com/Antseed/antseed/pull/1080), in
`packages/contracts`:

- `stats/AntseedStatsV2.sol`
- `emissions/AntseedAttributionUsage.sol`
- `emissions/AntseedEpochShareRewards.sol`
- `emissions/AntseedClientRewards.sol`
- `rewards/AntseedReferrals.sol`
- `script/DeployStatsV2.s.sol`, `script/DeployAttributionUsage.s.sol`,
  `script/DeployReferrals.s.sol`, `script/DeployClientRewards.s.sol`

## Security Considerations

**Invites are bearer tokens.** Whoever presents an unused invite first binds
to its referrer. A leaked or intercepted invite can be used by a different
buyer than the one intended; the referrer still gains a referee, but the
intended buyer loses the referee share. Exposure is bounded by the quota and
the four-epoch validity. A rejected bind (for example `NotNewBuyer`) leaves
the invite in public settlement calldata and unused, so anyone can then
present it. Clients SHOULD call `previewInvite` before carrying an invite and
SHOULD treat an invite as spent once it appears on-chain.

**Self-referral through sibling wallets.** The contracts reject a referrer
that is the buyer, the buyer's operator, or a wallet sharing the buyer's
non-zero operator, at bind time and again at every credit. One person
controlling two unrelated wallets cannot be detected on-chain. That attack is
bounded by: the quota, which requires the referrer to have real recognized
buyer or seller activity in the previous epoch; the new-buyer rule, which
prevents re-binding established wallets; the referee window, which limits
the doubled share to 12 epochs; and the fact that all credit is AIP-2
recognized points, already subject to wash exclusion and pool weighting.
None of these make it impossible.

**Signature replay.** Invite signatures are bound to the `AntseedReferrals`
domain (name, version, chain id and contract address), so they cannot be
replayed on another chain or another deployment. The single-use bitmap
prevents replay within a deployment. The tail itself is covered by the
buyer's SpendingAuth or FreeUsageAuth `metadataHash`, so a seller or relayer
cannot add, alter or strip it.

**Self-declared client id.** `clientId` is whatever the client software puts
in the tail; the protocol only checks that the agent id exists. Any client,
including a buyer's own script, may name any registered agent, and a buyer
may name an agent it owns to capture the client share of its own recognized
usage. This is equivalent to a rebate on recognized points and is bounded by
the bucket size and by recognized-point rules. Client rewards SHOULD NOT be
read as proof that a particular product produced the usage.

**Pause behavior.** Pausing `AntseedAttributionUsage` drops (does not defer)
the points that pass the cursor while paused, and `flush` reverts. Pausing
`AntseedReferrals` makes binds revert with `EnforcedPause`, which Stats
swallows without consuming the invite, and blocks claims. Pausing either
controller blocks claims and remainder settlement. None of these block
settlement. Owners of these contracts can also re-point recorders, binders
and sinks; ownership SHOULD be held by the same governance as the gate.

**Settlement liveness.** Binds and ledger records run inside `try`/`catch`
in Stats, and Stats runs inside `try`/`catch` in Channels and FreeUsage, so a
failing or misconfigured attribution path can lose attribution but cannot
revert a settlement or block seller payouts. Both constructors that call
`ownerOf` in `try`/`catch` reject a code-less identity registry, because such
a call would revert outside the `catch` and fail every attributed record or
claim permanently.

**Late credits and frozen totals.** Controllers freeze an epoch's total at
first claim and read each claimant's points live. The ledger never credits an
epoch after it becomes claimable (`CREDIT_GRACE_EPOCHS` equals
`SETTLEMENT_GRACE_EPOCHS`), so a claimant's points cannot grow against a
frozen total; late usage rolls forward instead. Each claim is also capped at
the epoch's unminted budget, so the gate's bucket cap cannot be exceeded.

**Recipient custody.** Referee rewards never go to a buyer hot wallet and
revert while no operator is set. Referrer rewards fall back to the referrer
address when no operator is set, which is intended for sellers and plain
wallets. Client rewards follow ERC-8004 ownership at claim time, so an agent
transfer moves unclaimed client rewards to the new owner.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE).
