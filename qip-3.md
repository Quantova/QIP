---
qip: 3
title: Accounts and Addresses
status: Final
type: Core
category: Core
author: Quantova Inc
created: 2026-07-27
---

# QIP 3, Accounts and Addresses

## Abstract

An account on Quantova is a keypair under a post quantum signature scheme together with the record the chain keeps for it. This proposal documents how a seed derives an account, how an account address is formed, how the scheme is identified, how an account registers its public key so the chain can verify its signatures, and the fields the chain keeps for each account. The default account signature scheme is ML-DSA-65 as standardised in FIPS 204. An address is a bech32m string over the SHA3-256 hash of the scheme byte and the public key, rendered in uppercase with the human readable part Q, so every address reads as Q1 followed by its data and checksum. This is a foundational part of the network and is recorded here as a Final Core document.

## Motivation

Quantova is a post quantum Layer 1 blockchain, so the primitive that authorises spending from an account must itself be post quantum. A classical public key primitive in the account layer would let an adversary with a fault tolerant quantum computer forge authority over any account. The account model here uses ML-DSA-65 for account authority, derives every account deterministically from a single seed by index, and commits to the public key inside the address, so the address itself carries no classical assumption. The chain stores only the hash of the public key until the account chooses to publish the key, which keeps the record of an unused account small and defers the full key to the point where a signature must be verified.

## Specification

### Seed and derivation

An account derives from a master seed of MASTER_SEED_LEN, which is 32 bytes, and an index. account_seed builds the input from the master seed, then one scheme byte, then the index as eight bytes in little endian order, and runs SHAKE256 over that input to produce a 32 byte per account seed. derive_with_scheme feeds the per account seed to the scheme keygen to produce the public key, and derive is the entry point that fixes the scheme to the lattice scheme. Because derivation is a pure function of the master seed, the scheme byte, and the index, the same three inputs always reproduce the same account, and one master seed yields a family of accounts addressed by index. Changing the index yields a different account, and so does changing the scheme.

The per account seed is the one secret an Account value holds. The type wipes the seed when the account is dropped, its Debug printing shows the seed as redacted and never as bytes, and the scheme, the index, and the public key are left in the clear because they are not secret.

### Signature scheme and scheme identifier

The scheme is named by a one byte identifier carried in the account. SCHEME_LATTICE, value 1, is the default account scheme and derives its keypair through ML-DSA-65 as standardised in FIPS 204. The account record also carries this identifier, so a reader can tell which scheme guards an account. Two further identifiers are reserved in the code, SCHEME_HASH, value 2, for a hash based scheme, and SCHEME_FALCON, value 3, which is gated off until its standard is final. derive selects SCHEME_LATTICE, so a plain account is an ML-DSA-65 account.

### Address format

An address commits to both the scheme and the public key. address_hash prepends the scheme byte to the public key and takes the SHA3-256 hash of the two, giving a 32 byte digest. render_address encodes that digest as a bech32m string. The human readable part is Q and the encoder writes the separator 1 after it, then the data groups, then a six character bech32m checksum, and finally uppercases the whole string, so an account address reads as Q1 followed by its data and checksum. The bech32m constant used for the checksum is 734539939.

Addresses hold an address floor. KEY_FLOOR is 32 bytes, and render_address and parse_address both refuse a payload below that floor. An account address is exactly the 32 byte SHA3-256 digest, so it sits at the floor.

Case does not carry meaning in an address. The decoder rejects a string that mixes upper and lower case, and it lowers the string before it checks the checksum, so the same address written in either case decodes to the same payload. The client library confirms the consequence for signing, where signing a call to an address written in uppercase and the same address written in lowercase produces byte identical signed transactions with the same transaction id. An address is rendered in uppercase by default.

### Key registration

An address is the hash of a public key, so the chain holds only that hash until the account publishes the key. To verify an ML-DSA-65 signature the chain needs the full public key, so an account registers its key before its signatures can be verified. Registration is a call to a fixed key register address, which is itself an address rendered over the SHA3-256 hash of the label qtv/key/register, carrying the account public key as the call argument. Once the key is on record the account field has_key reports it.

### Account fields

The chain keeps a record for each account with four fields. nonce is a counter that orders the account transactions, and a signed transaction carries the nonce the chain expects next. balance is the account balance held as an unsigned 128 bit amount. scheme is the one byte scheme identifier described above. has_key reports whether the account public key has been registered. These fields are surfaced through the gateway account response and read into the client Account value alongside the address.

## Rationale

Deriving every account from one master seed by index means a single backed up seed recovers the whole family of accounts, and the derivation mixes in the scheme byte so the same index under a different scheme is a different account. Committing the scheme and the public key into the address through SHA3-256 keeps the address short and fixed in length while binding it to exactly one key under exactly one scheme. Holding only the hash on chain until the account registers its key keeps an unused account cheap to record and reveals the full public key only when a signature has to be verified. bech32m gives a checksummed address that catches transcription slips, and rendering it in uppercase with a case insensitive decode keeps the address readable without letting case change its meaning. Choosing ML-DSA-65 for account authority keeps the account layer post quantum, in line with the rest of the network.

The same bech32m machinery renders the other identifiers of the network, each under its own human readable part, for example transaction ids, block hashes, and state roots, so an account address is one member of a single identifier format rather than a one off encoding.

## Backward Compatibility

This account and address model is the network model from genesis, so there is no earlier account format to remain compatible with. The scheme is named by a one byte identifier inside every account, and further identifiers are reserved, so an additional post quantum scheme can be added later without changing the address format or the account record, since a new scheme takes a new identifier and derives its own address through the same SHA3-256 and bech32m path.

## Security Considerations

Account authority rests on ML-DSA-65, so an account is guarded by a post quantum signature and carries no classical public key assumption. The address binds the scheme and the public key through SHA3-256, so an address names exactly one key under one scheme. The per account seed is the sole secret held in memory, is wiped when the account is dropped, and never appears in Debug output. Because the address is a hash of the public key, the chain cannot verify a signature until the key is registered, and the has_key field records that state. The address floor and the bech32m checksum reject truncated or malformed addresses before they reach the account layer. Quantova is on testnet, and external audits of the account model and its cryptography are still ahead. This document describes the design and its intended properties.

## Test Cases and Reference Implementation

The account and address code runs on the public testnet. The qtv-account crate holds derivation, the scheme identifiers, and address formation, qtv-idfmt holds the bech32m identifier format and the address floor, and the QCore.rs client library holds the account record, the derivation helpers, and the key registration flow, each with its tests. Reference behavior and test vectors are tracked with the implementation.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
