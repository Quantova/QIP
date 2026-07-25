---
qip: <to be assigned by an editor>
title: <the QIP title is a few words, not a complete sentence>
status: Draft
type: <Core | Networking | Interface | Application Standard | Meta | Informational>
category: <only required for Core, Networking, Interface, or an application standard>
author: <a list of the author names and contact, for example Name name@example.com>
created: <YYYY-MM-DD>
requires: <optional, QIP number or numbers this depends on>
---

How to use this template. Copy this file to qip-N.md, where the editor assigns N. Fill in the preamble above and complete each section below. Delete this note and any guidance text in angle brackets before submitting. Remove the category line if your type is Meta or Informational.

## Abstract

A short technical summary of the proposal, around two hundred words.

## Motivation

Why this change is needed, what problem it solves, and why now. For a post quantum network on testnet, state clearly whether this is required for mainnet or is a post mainnet improvement.

## Specification

The precise technical specification, with enough detail that an independent implementer could build it. For Core proposals, describe consensus, validity rule, or cryptographic changes exactly. Reference the affected components, which are consensus, the QVM, networking, and the gateway.

## Rationale

Explain the design decisions, the alternatives considered, and the trade offs. Note any related work.

## Backward Compatibility

Describe any incompatibilities and how they are handled. State whether this change requires a coordinated network upgrade.

## Security Considerations

Discuss security implications. For Quantova, explicitly address post quantum security. State whether the change introduces, removes, or modifies any cryptographic assumption, and whether it touches the randomness path, the committee or finality keys, or account signatures. Do not claim an audited or production posture, because the network is on testnet and at a pre audit stage.

## Test Cases and Reference Implementation

Link to test vectors or a reference implementation where applicable. This is required for Core and application standard proposals before Final.

## Copyright

Copyright 2026 Quantova Inc. See LICENSE.
