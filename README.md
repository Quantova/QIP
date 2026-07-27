# Quantova Improvement Documents

This repository tracks proposed and adopted changes to the Quantova protocol in the form of Quantova Improvement Proposals, and it hosts the public [roadmap](ROADMAP.md).

Quantova is a post quantum Layer 1 blockchain. The network is currently on testnet, ahead of a mainnet launch, and has not yet completed external security audit. The core architecture is already built, so the standards that define the protocol are documented here as the network matures toward mainnet.

## Start here

Read [ROADMAP.md](ROADMAP.md) for where Quantova is today and the path to mainnet.

Read [qip-1.md](qip-1.md) for what a QIP is and how the process works.

Read [qip-template.md](qip-template.md) for the template that new proposals use.

## Proposals

Core protocol

- [QIP 1](qip-1.md) QIP Purpose and Guidelines, Meta, Living
- [QIP 2](qip-2.md) QORUS Consensus, Core, Final
- [QIP 3](qip-3.md) Accounts and Addresses, Core, Final
- [QIP 4](qip-4.md) Transaction Format and Signature Envelope, Core, Final
- [QIP 6](qip-6.md) QVM Container Format and Contract ABI, Core, Final
- [QIP 8](qip-8.md) On Chain Governance and Forkless Upgrades, Core, Final
- [QIP 10](qip-10.md) Airlock Trustless Bridge, Core, Review

Interfaces and standards

- [QIP 5](qip-5.md) Gateway RPC, Interface, Living
- [QIP 7](qip-7.md) QAsset Fungible Token Standard, Interface, Draft
- [QIP 9](qip-9.md) QNS Name Service, Interface, Draft

## Repository layout

```
.
├── README.md          The overview you are reading
├── ROADMAP.md         Public roadmap from testnet toward mainnet
├── qip-1.md           QIP purpose and guidelines, Meta, Living
├── qip-2.md           QORUS consensus, Core, Final
├── qip-3.md           Accounts and addresses, Core, Final
├── qip-4.md           Transaction format and signature envelope, Core, Final
├── qip-5.md           Gateway RPC, Interface, Living
├── qip-6.md           QVM container format and contract ABI, Core, Final
├── qip-7.md           QAsset fungible token standard, Interface, Draft
├── qip-8.md           On chain governance and forkless upgrades, Core, Final
├── qip-9.md           QNS name service, Interface, Draft
├── qip-10.md          Airlock trustless bridge, Core, Review
├── qip-template.md    Template for new QIPs
└── LICENSE            License text
```

## The stack a QIP describes

The virtual machine is the QVM, a deterministic register machine that runs compiled containers, with post quantum signature verification, hashing, and Merkle proof checking as native instructions.

Consensus is QORUS. A committee is sampled each round to attest to the proposed block, finality is recorded as a single aggregated certificate signed with ML-DSA-65, and the committee is bounded so the work to finalize a block does not grow with the validator set.

Cryptography is Q-Crypto, Quantova's implementation of the NIST post quantum standards, with ML-DSA-65 under FIPS 204 for signatures, ML-KEM-768 under FIPS 203 for key exchange, SLH-DSA under FIPS 205, SHA-3 with SHAKE under FIPS 202, and ChaCha20-Poly1305 for the transport channel.

Addresses are Q1 bech32m. The asset is QTOV and its base unit is the Quon, where one QTOV is one million Quon. The testnet asset is TQTOV.

The gateway is an HTTP POST to /v1/<method> with a flat JSON body. The client SDK is the QCore family, QCore.rs the Rust core, QCore.js published on npm as @quantovainc/qcore, and QCore.py. The terminal client is qtv in the quantova-cli repository.

The smart contract language is Quanta, compiled to QVM containers and designed so that whole classes of vulnerability are caught at compile time. The fungible token standard is QAsset and the non fungible standard is QCollectible. The name service is QNS with domains under a capital Q top level domain.

## QIP types

The types are Core, Networking, Interface, QAsset and QCollectible application standards, Meta, and Informational. See [qip-1.md](qip-1.md) for definitions and the status lifecycle.

## Links

Website https://quantova.org

## License

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
