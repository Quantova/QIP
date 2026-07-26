# Quantova Improvement Documents

This repository tracks proposed and adopted changes to the Quantova protocol in the form of Quantova Improvement Proposals, and it hosts the public [roadmap](ROADMAP.md).

Quantova is a post quantum Layer 1 blockchain. The network is currently on testnet, ahead of a mainnet launch, and has not yet completed external security audit. The core architecture is already built, so at this stage there are deliberately few accepted proposals. This repository serves mainly as the public roadmap and as the home for proposals that mature toward mainnet.

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

The virtual machine is the QVM, a deterministic register machine that runs compiled containers, with post quantum signature verification, hashing, and Merkle proof checking as native instructions.

Consensus is QORUS. A committee is sampled each round to attest to the proposed block, finality is recorded as a single aggregated certificate signed with ML-DSA-65, and the committee is bounded so the work to finalize a block does not grow with the validator set.

Cryptography is Q-Crypto, Quantova's implementation of the NIST post quantum standards, with ML-DSA-65 under FIPS 204 for signatures, ML-KEM-768 under FIPS 203 for key exchange, SLH-DSA under FIPS 205, SHA-3 with SHAKE under FIPS 202, and ChaCha20-Poly1305 for the transport channel.

Addresses are Q1 bech32m. The asset is QTOV and its base unit is the Quon, where one QTOV is one million Quon. The testnet asset is TQTOV.

The gateway is an HTTP POST to /v1/<method> with a flat JSON body. The client SDK is the QCore family, QCore.rs the Rust core, QCore.js published on npm as @quantovainc/qcore, and QCore.py.

The smart contract language is Quanta, compiled to QVM containers and designed so that whole classes of vulnerability are caught at compile time. The fungible token standard is QAsset and the non fungible standard is QCollectible. The name service is QNS with domains under a capital Q top level domain.

## QIP types

The types are Core, Networking, Interface, QAsset and QCollectible application standards, Meta, and Informational. See [qip-1.md](qip-1.md) for definitions and the status lifecycle.

## Links

Website https://quantova.org

## License

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
