# Quantova Roadmap

This roadmap describes where Quantova is today and the path to mainnet. It is intentionally high level. Quantova is currently on testnet, and the protocol is deliberately stable as it matures for mainnet. There are few accepted improvement proposals at this stage by design, because the core architecture is already built. As the network matures and the community grows, this roadmap and the [QIP process](qip-1.md) will track the changes that follow.

Dates are directional, not commitments. Sequencing matters more than calendar precision.

## Where we are today

The public Quantova testnet runs the same post quantum runtime, QORUS consensus, QVM, and HTTP gateway that the planned mainnet will run, distributed with the free TQTOV testnet asset.

What is live on testnet today.

Post quantum cryptography end to end, with no cryptography vulnerable to Shor anywhere in the chain. Signatures use ML-DSA-65, key exchange uses ML-KEM-768, SLH-DSA is available, hashing uses SHA-3 and SHAKE, and the transport channel uses ChaCha20-Poly1305.

QORUS consensus, a committee byzantine fault tolerant protocol with ML-DSA-65 finality and a budget bounded sortition committee. See [qip-2.md](qip-2.md).

The QVM, a register machine that runs compiled containers, with the Quanta smart contract language where whole exploit classes fail to compile.

The QCore SDK family, QCore.rs the Rust core, QCore.js published on npm as @qunatovainc/qcore, and QCore.py.

The HTTP gateway, an HTTP POST to /v1/<method> with a flat JSON body, and the public testnet faucet that distributes TQTOV.

## Phase 1, Testnet Stabilization, current

The focus is stability and correctness, not new features.

Independent external security audits of the protocol, the QVM, and the cryptography. These audits are still ahead of us.

Validator onboarding at scale and long running stability testing.

Throughput and latency measurement under sustained load, reported honestly against measured figures.

Tooling and documentation maturity across the QCore SDK family and the gateway.

## Phase 2, Mainnet Readiness

Audit remediation and sign off.

Genesis specification finalization, including the post quantum genesis parameters.

Validator set bootstrapping toward the active committee size.

Governance activation for upgrade by vote.

## Phase 3, Mainnet Launch

Canonical mainnet genesis with signed releases.

Native QTOV in production, with staking and validator rewards live.

Public gateway and archive infrastructure for indexers and applications.

## Phase 4, Post Mainnet Evolution, research tracked

