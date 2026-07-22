---
qip: 2
title: QORUS Consensus
status: Final
type: Core
category: Core
author: Quantova Inc
created: 2026-05-31
---

# QIP-2, QORUS Consensus

## Abstract

This proposal documents QORUS, the consensus protocol of Quantova. QORUS is a committee byzantine fault tolerant protocol. It elects a committee for each round through a budget bounded sortition, and it finalizes blocks with ML-DSA-65 signatures from that committee. Every cryptographic primitive on the consensus path is post quantum, so there is no cryptography vulnerable to Shor in leader selection, in voting, or in finality. This is a foundational design decision of the network and is recorded here as a Final Core document.

## Motivation

Quantova is a sovereign post quantum Layer 1 built from scratch. Consensus is the part of the stack most exposed to an adversary with a fault tolerant quantum computer, because a classical public key primitive in the voting or finality path would let such an adversary forge authority. QORUS removes that exposure by using only post quantum primitives for committee selection and finality. It selects committees by sortition so that authority rotates across the validator set rather than resting on a fixed order, and it bounds the committee size by a budget so that the cost of finalizing a block does not grow without limit as the validator set grows.

## Specification

Committee selection is by budget bounded sortition. For each round the protocol draws a committee from the active validator set. The draw is seeded from post quantum randomness carried by the chain, and the committee size is bounded by a fixed budget rather than by the total number of validators.

Finality is byzantine fault tolerant. The committee votes on the proposed block, and the block is final once the committee reaches the required threshold of votes. Finality votes are ML-DSA-65 signatures, so finality is post quantum.

Validator authority and signing keys are post quantum. Authority at the finality layer uses ML-DSA-65 under FIPS 204.

Randomness used by sortition is sourced from Quantova's post quantum primitives, so the randomness path carries no classical public key cryptography.

Because the committee is bounded by a budget rather than by the validator count, the work to finalize a block does not grow without limit as the validator set grows.

## Rationale

A committee byzantine fault tolerant design gives provable finality from an explicit vote threshold, which suits a network aimed at settlement. Sortition spreads authority across the validator set instead of concentrating it in a fixed order. Bounding the committee by a budget keeps finality cost stable as the set of validators grows. Choosing ML-DSA-65 for finality signatures keeps the whole consensus path post quantum, with no elliptic curve and no classical public key primitive anywhere in leader selection, voting, or finality.

## Backward Compatibility

Quantova does not inherit any classical consensus deployment. QORUS is the network's design from genesis. For developers arriving from other chains, the committee is selected by budget bounded sortition rather than by a fixed order, and finality is carried by ML-DSA-65 votes rather than by classical signatures.

## Security Considerations

QORUS keeps the full consensus path post quantum. There is no cryptography vulnerable to Shor in committee selection, in voting, or in finality. The safety of finality rests on the committee vote threshold and on the honesty assumptions standard to committee byzantine fault tolerant protocols. Quantova is on testnet and at a pre audit stage, and external audits of the protocol and its cryptography are still ahead. This document describes the design and its intended properties. It does not assert an audited or production posture.

## Test Cases and Reference Implementation

The QORUS implementation runs on the public testnet. Reference behavior and test vectors are tracked with the protocol implementation.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
