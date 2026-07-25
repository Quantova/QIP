---
qip: 2
title: QORUS Consensus
status: Final
type: Core
category: Core
author: Quantova Inc
created: 2026-05-31
---

# QIP 2, QORUS Consensus

## Abstract

This proposal documents QORUS, the consensus protocol of Quantova. A committee is sampled each round to attest to the proposed block, and finality is recorded as a single aggregated certificate of ML-DSA-65 signatures from that committee. The committee is sampled by a budget bounded sortition, so its size does not grow with the validator set. Every cryptographic primitive on the consensus path is post quantum, so leader selection, voting, and finality are designed to withstand both classical and quantum attack. This is a foundational design decision of the network and is recorded here as a Final Core document.

## Motivation

Quantova is a post quantum Layer 1 blockchain. Consensus is the part of the stack most exposed to an adversary with a fault tolerant quantum computer, because a classical public key primitive in the voting or finality path would let such an adversary forge authority. QORUS removes that exposure by using only post quantum primitives for committee selection and finality. It selects committees by sortition so that authority rotates across the validator set rather than resting on a fixed order, and it bounds the committee size by a budget so that the cost of finalizing a block does not grow without limit as the validator set grows.

## Specification

Committee selection is by budget bounded sortition. For each round the protocol draws a committee from the active validator set. The draw is seeded from post quantum randomness carried by the chain, and the committee size is bounded by a fixed budget rather than by the total number of validators.

The committee votes on the proposed block, and the block is final once the committee reaches the required threshold of votes, gathered as a single aggregated certificate. Finality votes are ML-DSA-65 signatures, so finality is post quantum. The block reaches agreement despite a dishonest minority of the committee, because the threshold cannot be met without honest votes.

Validator authority and signing keys are post quantum. Authority at the finality layer uses ML-DSA-65 under FIPS 204.

Randomness used by sortition is sourced from Quantova's post quantum primitives, so the randomness path carries no classical public key cryptography.

Because the committee is bounded by a budget rather than by the validator count, the work to finalize a block does not grow without limit as the validator set grows.

## Rationale

A committee design with an explicit vote threshold gives provable finality, which suits a network aimed at settlement, and the block reaches agreement despite a dishonest minority of the committee. Sortition spreads authority across the validator set instead of concentrating it in a fixed order. Bounding the committee by a budget keeps finality cost stable as the set of validators grows. Choosing ML-DSA-65 for finality signatures keeps the whole consensus path post quantum, with no elliptic curve and no classical public key primitive anywhere in leader selection, voting, or finality.

## Backward Compatibility

QORUS is the network's consensus from genesis, so there is no earlier deployment to remain compatible with. The committee is selected by budget bounded sortition, and finality is carried by ML-DSA-65 votes.

## Security Considerations

QORUS keeps the full consensus path post quantum. Committee selection, voting, and finality are built on post quantum primitives. The safety of finality rests on the committee vote threshold, so the network reaches agreement while a dishonest minority of the committee cannot meet the threshold. Quantova is on testnet, and external audits of the protocol and its cryptography are still ahead. This document describes the design and its intended properties.

## Test Cases and Reference Implementation

The QORUS implementation runs on the public testnet. Reference behavior and test vectors are tracked with the protocol implementation.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
