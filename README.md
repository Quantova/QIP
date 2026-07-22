# Quantova Improvement Documents

This repository tracks proposed and adopted changes to the Quantova protocol in the form of Quantova Improvement Proposals, and it hosts the public [roadmap](ROADMAP.md).

Quantova is a sovereign post quantum Layer 1 built from scratch. The network is currently on testnet, ahead of a mainnet launch, and it is at a pre audit stage with external audits still ahead. The core architecture is already built, so at this stage there are deliberately few accepted proposals. This repository serves mainly as the public roadmap and as the home for proposals that mature toward mainnet.

## Start here

Read [ROADMAP.md](ROADMAP.md) for where Quantova is today and the path to mainnet.

Read [qip-1.md](qip-1.md) for what a QIP is and how the process works.

Read [qip-template.md](qip-template.md) for the template that new proposals use.

## Repository layout

```
.
├── README.md          The overview you are reading
├── ROADMAP.md         Public roadmap from testnet toward mainnet
├── qip-1.md           QIP purpose and guidelines, Meta, Living
├── qip-2.md           QORUS consensus, Core, Final
├── qip-template.md    Template for new QIPs
└── LICENSE            License text
```

## The stack a QIP describes

The virtual machine is the QVM, a register machine that runs compiled containers. It is not the Ethereum virtual machine and it is not wasm.

Consensus is QORUS, a committee byzantine fault tolerant protocol with ML-DSA-65 finality and a budget bounded sortition committee.

Cryptography is Q-Crypto, a from scratch NIST post quantum suite with ML-DSA-65 for signatures, ML-KEM-768 for key exchange, SLH-DSA, SHA-3 and SHAKE, and ChaCha20-Poly1305 for the transport channel. There is no elliptic curve and no classical public key cryptography anywhere in the chain.

Addresses are Q1 bech32m. The asset is QTOV and its base unit is the Quon, where one QTOV is one million Quon. The testnet asset is TQTOV.

The gateway is an HTTP POST to /v1/<method> with a flat JSON body. The client SDK is the QCore family, QCore.rs the Rust core, QCore.js published on npm as @qunatovainc/qcore, and QCore.py.

The smart contract language is Quanta, where whole exploit classes fail to compile. The fungible token standard is QAsset and the non fungible standard is QCollectible. The name service is QNS with domains under a capital Q top level domain.

## QIP types

The types are Core, Networking, Interface, QAsset and QCollectible application standards, Meta, and Informational. See [qip-1.md](qip-1.md) for definitions and the status lifecycle.

## Links

Website https://quantova.org

## License

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
