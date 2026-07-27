---
qip: 9
title: QNS Name Service
status: Draft
type: Standards Track
category: Interface
author: Quantova Inc
created: 2026-07-27
---

# QIP 9, QNS Name Service

## Abstract

This proposal describes QNS, the Quantova Name Service. QNS is a Quanta contract that runs as a container on the QVM and maps a human readable label under the capital Q top level domain to an owner account and a resolved address. A name is a first class chain record, not an off chain entry, and it is owned by a Quantova account. Because every Quantova account is post quantum only, a name carries the same post quantum security as the account that holds it. This document specifies what a `.Q` name maps to, how registration, renewal and transfer work, the primary name concept, and the length based pricing with a grace window and a premium auction once a name expires. QNS is a Draft standard. It is not on the testnet at launch. It is a fast follow toward mainnet. The Quanta language features that QNS depends on, map value assignment and read together with full width address values, are being hardened, so this document describes only what the contract and the client show and marks clearly what is still to be specified.

## Motivation

Sending value and addressing accounts by a raw Q1 address is workable but unfriendly. A name service lets a person register `alice.Q` and receive at that name while the underlying account stays post quantum and under the owner's sole control. QNS provides that mapping as chain state rather than as an off chain directory, so a lookup reads container storage over the gateway and a registration is a signed container call from the owning account.

QNS is not required for the testnet launch. It is targeted as a fast follow toward mainnet. The reason it is not on the testnet at launch is that the on chain contract exercises Quanta language features that are still being hardened, in particular map value assignment and read and map values that are a full address in width rather than a single word. Those features are needed so a name key can map to an owner address and a resolved address. This proposal records the intended design now so the standard is settled before the language work lands and the container is deployed.

## Specification

QNS is an application layer contract on the QVM. It touches the QVM and the gateway. It does not change consensus, the validity rules, or any consensus or account cryptography.

### What a name maps to

The reference contract keeps its record across a set of maps, each keyed by an address width name key.

```
state {
  guardians: GuardianSet<7>;
  base_3: u64;
  base_4: u64;
  base_5_plus: u64;
  grace_period: u64;
  auction_duration: u64;
  start_premium: u64;
  interval: u64;
  vault: Q_Asset<QTOV>;
  expiry_of: Map<Q_Address, u64>;
  owner_of: Map<Q_Address, Q_Address>;
  resolved_of: Map<Q_Address, Q_Address>;
  primary_of: Map<Q_Address, Q_Address>;
  reserved: Map<Q_Address, u64>;
}
```

For a given name key, `owner_of` holds the account that controls the name, `resolved_of` holds the address the name resolves to, `expiry_of` holds the absolute expiry time, and `reserved` marks a protocol name that cannot be registered. A label enters an entry as a `sealed Q_Name` that carries its character length, and that length drives pricing. The maps are keyed by an address width value derived from the label. The exact derivation, a client side name hash that matches the container Hash operation together with the map domain tags for the owner, expiry and resolved maps, is a wiring seam that is still to be specified in the client, as recorded in `src/lib/qns.js`.

Forward resolution reads `resolved_of` for the name key and falls back to `owner_of` when no resolved target is set, and it returns an address only while the name is live. The client mirror of this rule refuses to return an address once the name has expired, so an expired name never resolves to a stale owner.

### Registration and ownership

Registration is the `register` entry. It accepts a sealed label, a term in years, and a sealed QTOV payment. It requires the label length to be at least three, the name not to be reserved, and the name to be either never registered or already past its grace and premium windows. The payment must meet the price for the term. On success it writes the caller into `owner_of`, sets `expiry_of` to the current time plus the term in seconds, merges the payment into the contract vault, and emits `Registered`.

Ownership actions are gated on `owner_of` matching the caller. The `transfer` entry moves a name to another account when the caller is the current owner and emits `Transferred`. The `renew` entry extends the expiry by a further term when the name still exists and the current time is within the grace window, and it takes the same length based payment. The `set_resolved` entry lets the owner point the name at any target address while the name is live and emits `Resolved`.

### Primary name

QNS records a primary name for an account through `primary_of` and the `set_primary` entry. When the caller owns the label and the name is live, `set_primary` writes the caller to name mapping and emits `PrimarySet`. This is the reverse direction of resolution, from an account to the one name that account presents as its own, and it is distinct from `resolved_of`, which is the forward direction from a name to an address.

### Pricing, grace and expiry

Pricing in the reference contract is the Dynamic length based model. Three tiers, `base_3`, `base_4` and `base_5_plus`, are set as deploy parameters and are charged per year. The tier is chosen from the label length by integer division, so a label of three characters or fewer pays `base_3`, a label of four characters pays `base_4`, and a label of five characters or more pays `base_5_plus`, each multiplied by the number of years. The contract custodies the fee as a real sealed QTOV payment in `vault`, so the quote a client shows is only a quote and the container enforces the amount.

Expiry is absolute and measured in consensus time. On registration the expiry is set to the current time plus the term counted in seconds. After expiry the name enters `grace_period`, a window in which only the current owner may renew. After grace the name enters a premium window of length `auction_duration` that runs as a Dutch auction through the `claim_premium` entry. The premium starts at `start_premium` and halves every `interval` seconds elapsed into the window, and the total charged during the window is the current premium plus the base tier for the term. Once the premium window closes the name is freely available to any account through `register`.

The client presents this same schedule and, in the web app, quotes the tiers in a stable unit and converts to TQTOV at charge time through a Q-Oracle price feed rate, with the tier values, the grace length, the premium window, the halving period and the premium start held as clearly marked placeholders. The exact `.Q` amounts and the denomination in which they are pinned are a founder decision and are still to be specified. This proposal does not fix those numbers.

### Reserved names and root authority

A set of protocol names such as `quantova`, `qns`, `registry` and `resolver` are held in `reserved` and cannot be registered, and the client also blocks a confusable look alike of a reserved name by comparing a folded skeleton. The `reserve` and `release` entries manage the reserved set and are gated by a guardian quorum, a threshold of a fixed guardian set, rather than a single key. The `withdraw` entry sweeps the fee vault and is gated by a higher guardian threshold. Guardian authority is bound to post quantum ML-DSA-65 signatures.

### Still to be specified

The following are open and are marked here so an implementer does not assume them. The name to key derivation and the map domain tags are a client wiring seam. The container call argument layout that encodes the sealed name, the term and the sealed payment is a second wiring seam. The pinned tier amounts, the grace length, the premium window, the halving period, the premium start, and the unit those are denominated in are a founder decision. Whether the on chain record stores a plaintext label or only an address width name key, and how the label length reaches the pricing guard through the QVM Hash operation, is still being settled alongside the language work.

## Rationale

Keeping the name record on chain as container state, rather than as an off chain directory, means ownership and resolution inherit the account model and the post quantum guarantees of the chain and cannot be changed by a client. Length based tiers price short and scarce names higher than long ones, and charging per year with a grace window and a premium auction on lapse gives an owner a safe renewal window while returning a contested name to circulation through a decaying premium rather than a first come grab. A guardian quorum over the reserved set and the fee sweep keeps root authority off any single key, which matches the network's aim of no lone key holding privileged control. Keying the maps by an address width name key, rather than a variable length label, lets the record reuse the same word width the QVM already handles for addresses, at the cost of the derivation seam noted above. The intended control is a guardian quorum now and a governance vote later.

## Backward Compatibility

QNS is a new application standard with no earlier deployment, so there is no prior on chain behavior to remain compatible with. The container is not deployed on the target network yet, so the client reports a clear not available message rather than guessing an answer while the container address is unset. QNS is an application layer contract and does not by itself require a coordinated consensus upgrade. It does depend on Quanta language work that is being hardened, namely map value assignment and read and full width address map values, and it cannot be deployed until that work lands in the codegen path and the contract compiles against it.

## Security Considerations

A `.Q` name is owned by a Quantova account, and Quantova accounts are post quantum only, using ML-DSA-65 and SLH-DSA with no elliptic curve on the identity path, so a registration, a renewal, a transfer or a resolved address change requires a valid post quantum signature from the owning account. QNS does not introduce, remove or modify any consensus cryptographic assumption, and it does not touch the randomness path, the committee or finality keys, or the account signature scheme. Its privileged actions, the reserved set and the fee sweep, are gated by a post quantum guardian quorum rather than a single key, and the client binds each guardian action message to a protocol tag, the action, the container and an epoch so a signature cannot be replayed onto another action or epoch. Resolution is gated on expiry so a stale name never returns a live address, and a confusable look alike of a reserved name is rejected before it can be registered.

The language change that QNS depends on, widening map values to a full address width, lives in the codegen path and is being hardened. It should be reviewed as part of that work before QNS is deployed. Quantova is on testnet and is at a pre audit stage, with external audits of the contract, the client and the language work still ahead. This document describes the intended design and its properties and does not claim an audited or production posture.

## Test Cases and Reference Implementation

The reference contract is the QNS example in the Quanta Smart Contract language examples, which shows the state maps, the register, renew, claim_premium, reserve, release, set_resolved, set_primary, transfer and withdraw entries, and the emitted events. The reference client is the QNS web app, which holds the canonical name rule, the length based price quote, the expiry gate that refuses to resolve an expired name, and the register, renew and send flows, with unit tests under its tests directory. Because the container is not deployed on the target network yet, there are no on chain test vectors, and two client wiring seams remain, the name key derivation with the map domain tags and the container call argument layout. Test vectors and a deployed reference will be tracked with the container once the language work lands and the container is published.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
