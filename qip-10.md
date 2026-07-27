---
qip: 10
title: Airlock Trustless Bridge
status: Review
type: Core
category: Core
author: Quantova Inc
created: 2026-07-27
---

# QIP 10, Airlock Trustless Bridge

## Abstract

This proposal documents the Airlock, the design under which Quantova bridges value to and from foreign chains without ever admitting classical cryptography onto the chain. A foreign chain is watched and verified off chain by Q-Oracle. Only two kinds of artifact are allowed to cross into Quantova, an ML-DSA attestation over an observed fact and a hash STARK. Every elliptic curve check, every foreign signature, and every foreign consensus rule stays on the oracle side of the Airlock. An inbound deposit mints a wrapped token under a distinct operator quorum whose attestation binds the destination chain and the Quantova chain id into its preimage. An exit burns the wrapped token, and custody is a single governance held pool that is drawn only when a foreign payout is proven at settle. A failed exit that resolves in a slash refunds the beneficiary on chain without drawing that pool twice. Per epoch and global payout caps bound the exposure, and governance can freeze the bridge and move the pool in an emergency. The whole exit path is closed by default. Quantova is on testnet and the design has not been through an external audit. This document records the Airlock as a Core design under review, not a live or audited system.

## Motivation

Quantova is a sovereign post quantum Layer 1 with only NIST standardized schemes and no classical escape hatch anywhere on the chain. A bridge is the one place that rule collides with reality, because the chains worth bridging to still sign with elliptic curve cryptography, and reading them means checking those classical signatures. The Airlock resolves the collision by drawing a hard wall. The classical work happens off chain in Q-Oracle, the single repository exempt from the classical crypto deny list, and the chain sees only post quantum artifacts. This keeps the chain's cryptographic posture intact while still letting value move across corridors to Bitcoin, Ethereum, and Cosmos.

The second motivation is custody discipline. A bridge holds real backing for every wrapped token it issues, and the failure that ruins bridges is paying the same backing out twice or issuing wrapped tokens with nothing behind them. The Airlock keeps custody in one pool, draws it only against a proven foreign payout, and treats the reference to a burn as a single use key so that an exit resolves at most once, as a settle or as a slash but never as both. Caps bound how much can leave in any epoch, and an emergency freeze and pool migration give governance a way to stop the bridge and relocate the backing if something goes wrong.

## Specification

### The Airlock boundary

Q-Oracle watches a foreign chain, runs that chain's own verification off chain, and turns a confirmed foreign event into a post quantum artifact. Two artifact kinds are defined. An attestation is a set of ML-DSA signatures over an encoded fact, carrying artifact tag 0x01. A hash STARK carries artifact tag 0x02 and a statement digest with its proof bytes. The ingress parser accepts only these two post quantum suites in a fixed order, the ML-DSA slot first at the pinned ML-DSA-65 signature length and the hash STARK slot second. Any other suite is refused before it is read. Foreign material, an ed25519 signature, a secp256k1 public key or DER signature, a BLS12-381 point, an Ethereum RLP header, does not carry a recognized suite tag and is rejected as unparseable. The chain never sees the foreign signature. It sees only the ML-DSA attestation and the hash STARK that Q-Oracle produced.

The exemption that lets classical crypto exist at all lives only in Q-Oracle's own deny file, for that repository alone, and no other crate is allowed to depend on Q-Oracle. The classical code has exactly one home and no path inward.

### Inbound deposit and mint

A deposit fact records the source chain, the destination chain, a route, a nonce, a one time source reference, the asset, the amount, the recipient, and the observed and expiry heights. The attestation preimage is a domain tag followed by the encoded fact followed by the Quantova chain id in little endian. Binding the chain id into the preimage means an attestation signed for one chain does not verify under another, even when two chains share the low 32 bits of their id, a case pinned by a sibling chain test.

The mint quorum is a distinct operator set held in chain state under its own key, separate from the consensus validator committee. Each operator watches its own source, signs the observed fact with an ML-DSA key, and the quorum is met only when a threshold of distinct operator public keys verify over the preimage. A duplicated operator id counts once, a single public key registered under two ids fills only one slot, and a revoked operator no longer counts toward the threshold. An operator proves possession of its key with a signature that binds the key to its operator id and to the chain id. The quorum check also refuses any fact whose destination chain does not match the configured destination and any direction other than a deposit.

On a verified quorum the chain mints. The mint refuses a zero amount, an unregistered asset, and a source reference that has already been seen, so a deposit mints once. It refuses to raise the wrapped supply above the asset cap or the per epoch mint above the asset epoch cap. It credits the single pool vault custody and the recipient's wrapped balance, marks the source reference consumed, and records the mint. An asset flagged as requiring a proof additionally needs a hash STARK whose statement digest binds to the exact fact. The on chain check confirms that binding and reports the state as bound but unverified, because the foreign consensus proof itself is verified off chain in Q-Oracle and in the light client and is compressed into that STARK. The chain checks that the STARK belongs to the fact, not the STARK proof.

### Exit and custody

An exit begins with a burn. The holder submits an exit request naming the asset, the amount, and the foreign destination. The chain refuses the burn unless the holder's wrapped balance covers the amount, then retires that wrapped supply and debits the holder. The backing coins do not leave the pool at burn. Custody is untouched here, because the coins move only when a foreign payout is proven.

Custody is a single pool vault recorded as one value in chain state. A mint credits it, and it is drawn only at settle. When Q-Oracle proves a foreign payout, an exit acknowledgement crosses the Airlock as an ML-DSA quorum over an exit fact, carrying the corridor, the destination chain, the asset, the amount, the beneficiary, the reference to the burn, and the outcome. The exit quorum is verified under its own domain exactly as strictly as the mint quorum, with the same distinct key counting and the same chain id and destination binding. On a settle outcome the chain draws the pool custody down by the paid amount exactly once, refuses if the draw would leave custody below the outstanding wrapped supply, and consumes the burn reference. On a slash outcome, where the foreign payout failed, the chain reissues the burned wrapped token to the beneficiary on chain and consumes the same burn reference, but it does not draw custody down, since the coins never left the pool. The burn reference is the single replay key, so an exit resolves at most once and a slash refund never draws the same custody that a settle would have drawn.

### Caps

Three caps bound issuance and payout. Each asset carries a global supply cap and a per epoch cap. The per epoch cap limits both how much of that asset may mint in an epoch and how much may be paid out as slash refunds in an epoch. A separate global payout cap limits the total slash refund across all assets in an epoch, and a zero value is the unset sentinel that refuses every payout. A slash refund must clear the per asset epoch cap, the global per epoch payout cap, the asset supply cap, and the requirement that the reissued supply stays at or below the custody backing it, or it is refused rather than allowed through.

### Freeze and pool migration

Any account that is not blacklisted can freeze the bridge by posting a bond, and the freeze halts the bridge the block it lands. While the bridge is frozen the mint, the exit burn, the settle, the slash, and any call to the bridge gateway are all refused. A good faith freeze expires after its duration and returns the bond to the depositor, and a cooldown blocks an immediate refreeze. A freeze judged to be in bad faith is lifted early by a governance vote or by the guardian caucus, and that early lift slashes the bond to the treasury. Over a frozen bridge a governance migration vote can move the pool, routing all asset custody from the old pool vault to a new vault and repointing the pool to it. The migration is refused unless the bridge is frozen.

### Default gating

The whole exit path is closed by default. The exit burn, the settle, and the slash all check an exits enabled flag and refuse when it is unset, which is the genesis state. Enabling exits is a deliberate action, not the default. The inbound mint path is live only once an operator set, a destination chain, a registered asset, and the pool vault have been seeded into state.

## Rationale

Allowing exactly two post quantum artifact kinds across the Airlock, and refusing everything else at parse time, is what keeps the chain free of classical cryptography while still bridging chains that depend on it. Putting the classical work in one repository that nothing may import gives that dependency a single audited home. Binding the destination chain and the chain id into every attestation preimage stops an attestation from being replayed onto a sibling chain. Counting distinct operator keys, and refusing duplicate ids and duplicate keys, stops a single operator from filling a quorum.

Holding custody in one pool and drawing it only against a proven payout is the discipline that keeps backing and wrapped supply in step. Making the burn reference a single use key, and reissuing on a slash rather than paying twice, is what lets a failed exit unwind without touching the custody a settle would have drawn. The caps bound the blast radius of any single epoch. The freeze and the migration are the emergency stop and the recovery move, and gating the exit path off by default means the untested payout path cannot run until governance turns it on.

## Backward Compatibility

The bridge is additive. All of its state lives under new bridge keys, and no earlier deployment depends on it. Because exits are closed by default, enabling them is a forward governance decision rather than a change to existing behavior. The corridor design aligns to the tier ratchet in the light client, an ordering from federated through proof to native verification, where a corridor can only be upgraded and never silently downgraded. The trustless corridors, Bitcoin as a proof of work header chain verifier held to a pinned checkpoint and a cumulative work floor using SHA-256d, Ethereum as a sync committee light client, and Cosmos as a Tendermint light client, all verify off chain and compress into a hash STARK, so adding or upgrading a corridor changes the oracle and not the chain's mint entry point.

## Security Considerations

The Airlock's safety rests on the boundary holding. The classical crypto exemption exists in one repository, that repository is barred from being imported, and the ingress parser refuses every non post quantum suite before reading it, so classical material has no path onto the chain. The federated admission path carries an operator quorum as its trust assumption, and its safety is the assumption that a threshold of distinct operators will not collude, backed by chain id binding, distinct key counting, operator key possession proofs, and revocation. The proof backed corridors rest instead on the foreign chain's own consensus verified off chain and carried as a hash STARK. The on chain mint checks that the STARK binds to the fact, not that the STARK proof is sound, so proof verification soundness lives off chain in the current design and is a property to be proven, not one this document claims as settled.

Custody safety rests on the invariants that custody never falls below outstanding wrapped supply at settle, that a burn reference resolves once, and that a slash reissues rather than draws. The caps bound issuance and payout per epoch and across the whole bridge. The freeze and the governance pool migration are the emergency controls. None of this has been through an independent security audit. Quantova is on testnet, the exit path is closed by default, and the Airlock is recorded here as a design under review. The trust assumptions of the operator quorum, the off chain proof verification, and the exact cap and bond values are open items for that review.

## Test Cases and Reference Implementation

The chain side bridge, the Q-Oracle attestor and exit desk, and the Q-Lightclient ingress parser and corridor verifiers each carry their own tests. The chain side pins the attestation and exit fact wire layouts as fixed length vectors, the sibling chain id replay refusals, the distinct key quorum counting, the mint replay and cap refusals, the settle custody draw and its supply floor, the slash refund under the caps, and the freeze, expiry, early lift, and pool migration lifecycle. The Airlock ingress tests pin acceptance of the two post quantum suites and refusal of ed25519, secp256k1, and BLS12-381 material. Reference behavior and vectors are tracked with the implementation across the Quantova-Chain, Q-Oracle, and Q-Lightclient repositories.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
