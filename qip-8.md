---
qip: 8
title: On Chain Governance and Forkless Upgrades
status: Final
type: Core
category: Core
author: Quantova Inc
created: 2026-07-27
---

# QIP 8, On Chain Governance and Forkless Upgrades

## Abstract

This proposal documents the governance system of Quantova and the forkless upgrade path it carries. Quantova has no sudo key and no super user key. From genesis the only way to change privileged chain state is an approved referendum, and governance is the top authority of the network. Proposals ride one of five tracks, each with its own deposit, approval bar, and timelock. Votes are cast under conviction, which locks the committed stake for a chosen period. A constitution gate binds every enacted action to the track it was raised on and shields protected accounts from seizure. An emergency guardian caucus, itself rotated only by governance, can lift a bridge freeze under a threshold of signatures. The upgrade path lets logic that ships dormant in the binary turn on by a vote that raises a named feature version in state, so an upgrade rides a vote and never a fork. This is a foundational design of the network and is recorded here as a Final Core document.

## Motivation

A network aimed at settlement cannot rest its authority on a single administrative key. A sudo key is a standing takeover risk and a single point of failure, and it makes every privileged change a matter of trust in one holder rather than a matter of an open vote. Quantova removes that exposure by making governance the top authority from genesis. Every privileged mutation of chain state, minting, parameter changes, freezes, blacklisting, bridge administration, and feature activation, is reachable only through an approved referendum and its enactment path.

The second aim is to change the running chain without splitting it. A hard fork asks every operator to coordinate a flag day and carries the risk of a chain split when some operators do not follow. Quantova ships upgrade logic dormant inside the binary and turns it on by a vote that raises a feature version the node logic reads from state. This keeps upgrades on the same chain, under the same rules that govern every other privileged change.

## Specification

### No sudo, governance as top authority

There is no sudo path and no super user key anywhere in the ledger or the governance crate. The privileged actions all flow through `gov_enact`, which admits an action only after its referendum has reached `Status::Approved`, then runs the constitution gate, then calls `execute_action`. The one path that does not require a referendum is the guardian caucus, and the guardian caucus is a threshold multisig whose membership is set and rotated only by governance. No single key can mint, change a parameter, freeze an account, blacklist an account, or turn on a feature.

### The five tracks

Every proposal is raised on exactly one track. Each track carries a deposit denominated in QTOV, where one QTOV is `NATIVE_UNIT` base units, and a period that is both the voting window and the timelock before enactment. The `ChainUpgrade` track takes a deposit of 2,250,000 QTOV and a period of 14 days. The `Mint` track takes 4,000,000 QTOV and 3 days. The `BridgeMigration` track takes 1,500,000 QTOV and 5 days. The `FreezeRecovery` track takes 292,500 QTOV and 6 hours. The `BlacklistKill` track takes 390,000 QTOV and 2 days. A proposal opens a referendum at the time it is submitted, and the referendum can be concluded and enacted only once the current time reaches the submission time plus the track period. The proposer posts the deposit up front. The deposit is returned when the referendum passes and is forfeit to the stake treasury when it fails or is killed.

A referendum passes when the aye stake reaches forty percent of the total staked, drawn from `THRESHOLD_BPS` of 4,000 basis points against a denominator of 10,000. The electorate is the total staked at conclusion time. Below that bar the referendum is rejected.

### Conviction voting

A ballot commits stake under a conviction. Voting debits the committed stake from the voter balance and locks it. The conviction sets how long the stake stays locked and carries a weight factor. `Liquid` locks the stake for one month and carries a factor of one. `Year` locks it for one year and carries a factor of one and a half. `TwoYear` locks it for two years and carries a factor of two and a half. Each fresh ballot extends the lock so it ends no earlier than the new conviction demands. The committed stake is returned through `gov_release` once the lock has run. A voter casts at most one ballot per referendum.

### The constitution gate

Before any approved action runs, `check_enactment` binds it to the track it was raised on and shields protected accounts. If the action does not belong to the referendum track the gate returns `WrongTrack` and nothing is enacted. For a `FreezeRecovery` action the gate requires that the seizure set hash to the scope that was approved, computed as the SHA3-256 of the recovery preimage over the victim and the seizures, and it returns `RecoveryOutOfScope` when the hash does not match and `RecoveryTouchesProtected` when any seizure source is a protected account. For a `Freeze` action it returns `FreezeTouchesProtected` when any target is protected, and for a `Blacklist` action it returns `BlacklistTouchesProtected` when the target is protected.

A protected account is any account that holds a stake bond, so an active validator, any account that holds a governance vote lock, so an active voter, or any of the reserved network pots, which are the grants pot, the stake treasury, the stake pool, the stake system pot, the governance system pot, the bridge bond pot, and the ecosystem marketing, market maker, and foundation pots. These accounts cannot be frozen, blacklisted, or seized by a governance action.

### The action set

The actions that governance can enact, each pinned to a single track, are the following. `Activate` raises a named feature version and rides `ChainUpgrade`. `Parameter` sets a named chain parameter and rides `ChainUpgrade`. `GuardianRotate` replaces the guardian caucus and rides `ChainUpgrade`. `Mint` credits an account and the native supply and rides `Mint`. `Spend` moves value out of the grants pot or the stake treasury and rides `Mint`. `BridgeMigration` moves bridge custody to a new vault while the bridge is frozen and rides `BridgeMigration`. `CommitteeRotate`, `AssetRegister`, `EpochAdvance`, and `OperatorRevoke` administer the bridge committee and assets and ride `BridgeMigration`. `FreezeRecovery` seizes from named sources within an approved scope and credits a victim, and rides `FreezeRecovery`. `Freeze` and `Unfreeze` set and clear account freezes, `Blacklist` bars an account, and `BridgeUnfreeze` lifts a bridge freeze, and these ride `BlacklistKill`. Each action carries its own track in code, and the constitution gate refuses any attempt to enact an action off its track.

### Forkless upgrades

The upgrade path is the `Activate` action on the `ChainUpgrade` track. An `Activate` action names a feature and a version. When it is enacted, `execute_action` writes the version to a feature gate key derived as the SHA3-256 of the tag `qtv/gov/feature/` joined with the feature name. Node logic reads that state through `feature_version`, which returns the stored version or zero when the feature has never been raised, and through `feature_active`, which is true once the version is above zero. Logic that has shipped in the binary but stays dormant reads the gate and branches on it, so a passed vote turns the logic on across the network at once, on the same chain, with no fork and no flag day. A feature name cannot be empty, and only an approved `ChainUpgrade` vote through `execute_action` can raise a version.

This path turns on logic that already exists in the running binary. It does not write new logic. Brand new logic that is not yet compiled into the binary, a fix to the pinned QVM, or a change to QORUS consensus still needs a new binary shipped to operators. The feature gate flips behavior that the current binary already carries, and nothing more.

### The emergency guardian caucus

The guardian caucus is a threshold multisig recorded in state as a set of members and a threshold. The set is well formed only when the threshold is at least two and does not exceed the membership, so a caucus never acts on a single key. Authorization counts distinct members from the set until the threshold is met, and an outsider or a repeated key carries no extra weight.

The bridge freeze is a bonded safety valve. Any account that is not blacklisted can post the bridge freeze bond of 390,000 QTOV, which equals the `BlacklistKill` deposit, to freeze the bridge for seven days, subject to a one day cooldown after any lift. The account that posted the bond can lift the freeze early and take a refund. The guardian caucus can lift the freeze early under its threshold of signatures, in which case the bond is slashed to the treasury. Governance can also lift the freeze through the `BridgeUnfreeze` action on the `BlacklistKill` track, which slashes the bond as well. The guardian caucus is created and rotated only by governance through the `GuardianRotate` action, so the emergency power itself sits under the top authority.

## Rationale

Making governance the top authority from genesis removes the standing takeover risk of a sudo key and puts every privileged change under an open vote with a deposit, a bar, and a timelock. Splitting proposals across tracks lets the network price and time each kind of change to its weight, so a slow chain upgrade sits on a long window and a large deposit while a fast freeze recovery sits on a short window. Conviction voting lets a voter trade a longer stake lock for a heavier voice, which asks committed backers to bear real duration. The constitution gate keeps each vote to the track it was raised on and keeps validators, active voters, and the network pots outside the reach of a freeze, a blacklist, or a seizure, so the power to act against an account cannot be turned on the accounts that hold the network together. The guardian caucus gives the bridge a rapid brake without a single trusted key, and rooting the caucus in a governance rotation keeps that brake under the same authority as everything else. The feature gate lets the network turn on prepared logic by vote, which keeps upgrades on one chain and avoids the coordination and split risk of a hard fork, while still requiring a new binary for logic that is not already present or for a change to the pinned parts of the stack.

## Backward Compatibility

Governance is the authority of the network from genesis, so there is no earlier on chain authority to remain compatible with. There is no sudo key to retire. The five tracks, the conviction locks, the constitution gate, the action set, and the feature gate are the governance surface from the first block. Feature activations are additive, since an unraised feature reads as version zero and leaves the dormant logic off, so a binary that carries a feature behaves as before until a vote raises it.

## Security Considerations

The safety of governance rests on the deposit, the forty percent bar against total staked, and the track timelock, so a change cannot pass without committed stake behind it and cannot enact before its window has run. The constitution gate is the last check before enactment, and it refuses any action off its track and any freeze, blacklist, or seizure that touches a protected account, which keeps validators, active voters, and the network pots beyond governance reach. The guardian caucus never acts on a single key, since a well formed caucus needs a threshold of at least two distinct members, and the caucus is rotated only by governance. The feature gate only turns on logic already present in the running binary, so it cannot introduce code and cannot alter the pinned QVM or QORUS consensus, both of which still require a shipped binary. Quantova is on testnet, and external audits of the governance system and its cryptography are still ahead. This document describes the design and its intended properties.

## Test Cases and Reference Implementation

The governance system is implemented in the `qtv-governance` crate and wired into the ledger in `qtv-node`. The reference tests cover the five tracks and their deposits and periods, the conviction factors and locks, the forty percent approval bar and the deposit refund and forfeit rules, the timelock that holds a referendum until its window closes, the constitution gate binding each action to its track and shielding protected accounts from recovery, freeze, and blacklist, the guardian caucus refusing to act on a single key, and a governance vote that activates a dormant feature without a fork by raising its version in state. The implementation runs on the public testnet, and the reference behavior is tracked with the chain implementation.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
