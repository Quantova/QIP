---
qip: 1
title: QIP Purpose and Guidelines
status: Living
type: Meta
author: Quantova Inc
created: 2026-05-31
---

# QIP-1, QIP Purpose and Guidelines

## What is a QIP

QIP stands for Quantova Improvement Proposal. A QIP is a design document that describes a proposed change to the Quantova protocol, its processes, or its environment. The QIP provides a concise technical specification of the change and a rationale for it. The QIP author is responsible for building consensus within the community and documenting dissenting opinions.

QIPs are maintained as text files in this versioned repository, so their revision history is the historical record of each proposal.

Quantova is currently on testnet, ahead of a mainnet launch, and it is at a pre audit stage with external audits still ahead. During this phase the protocol is deliberately stable, and there are few accepted changes. This repository therefore serves primarily as the public roadmap, see [ROADMAP.md](ROADMAP.md), and as the home for proposals that mature toward mainnet. The QIP process below is the framework these proposals will follow as the network grows.

## QIP types

Core covers changes to the QORUS consensus protocol, block and transaction validity rules, or anything requiring a coordinated network upgrade, for example cryptographic primitive upgrades or consensus parameters.

Networking covers the peer to peer networking layer.

Interface covers client and gateway specifications and standards for how applications interact with the chain, for example the HTTP /v1 gateway methods.

Application standards cover conventions at the application layer, for example the QAsset fungible token standard and the QCollectible non fungible standard.

Meta covers process or governance proposals like this document.

Informational covers guidelines or information that do not propose a protocol change and do not require consensus.

## QIP status terms

Idea is a pre draft concept, not yet tracked in the repository.

Draft is the first formally tracked stage, merged into the repository for iteration.

Review means the author marks the QIP ready for peer review.

Last Call is a final review window before a decision.

Accepted means the proposal is endorsed and queued for implementation. Core proposals are queued for a network upgrade.

Final means the proposal is implemented and active on the canonical network.

Stagnant means the proposal has been inactive for an extended period and may be revived.

Withdrawn means the proposal has been retired by the author or authors.

Living means the document is continually updated and never finished, for example this document and the roadmap.

## QIP workflow

