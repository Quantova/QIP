---
qip: 6
title: QVM Container Format and Contract ABI
status: Final
type: Core
category: Core
author: Quantova Inc
created: 2026-07-27
---

# QIP 6, QVM Container Format and Contract ABI

## Abstract

This proposal documents the QVM, the deterministic register machine that runs every Quantova contract, together with the on chain container a contract is published as and the calling convention a caller uses to reach a contract entry. A container carries an instruction section, a constant pool, and a set of entries, each named by a four byte selector and guarded by a declared storage access manifest. Post quantum verification and hashing are native machine instructions rather than calls into a library, so a contract signs and hashes with the same primitives the rest of the network uses. A contract is deployed to an address derived from the deployer and a nonce, so its address is fixed before it runs. A call is a four byte selector followed by an encoded argument image, storage is a map of thirty two byte keys to whole word values, and the node writes the trusted caller and time into a reserved window that the caller cannot fill. This is a foundational format of the network and is recorded here as a Final Core document.

## Motivation

Quantova needs one execution format that every node agrees on, because consensus over a block is consensus over the exact state each contract leaves behind. A register machine with fixed width words, a bounded memory, checked arithmetic, and metered execution gives a single well defined result for a given input, so two honest nodes running the same call reach the same storage and the same effects. Publishing a contract as a canonical container with a stable identifier lets the network name the code by its content, and declaring the storage each entry may touch lets the node reason about a call before running it.

The most exposed part of a contract is its cryptography. A contract that verified a signature or hashed a message through a swappable library would inherit that library's assumptions, including any classical public key assumption. The QVM removes that exposure by making post quantum verification, key encapsulation, address derivation, and hashing native instructions with fixed semantics, so a contract cannot reach a weaker primitive and the same code runs identically on every node. Deriving a contract address from the deployer and a nonce, rather than from the code, lets a caller know the address before deployment and keeps deployment independent of the exact container bytes.

## Specification

### The machine

The QVM is a register machine with sixteen general registers, each a 64 bit word, a program counter held as a 32 bit offset into the code, a call stack bounded at 1024 words, and a flat memory of 65536 bytes. Words are read from and written to memory in big endian order. Arithmetic is explicit about width. `Add`, `Sub`, and `Mul` are checked and fault with `Overflow` on wraparound, `Div` and `Rem` fault with `DivByZero`, and the wrapping forms `AddW`, `SubW`, and `MulW` wrap by definition, with `MulHi` returning the high 64 bits of the 128 bit product. Shift amounts are masked to six bits. Every instruction is metered, and a run that would exceed its meter halts with `OutOfGas` before the instruction takes effect.

Execution is deterministic. A call runs from its entry offset until a `Halt`, and only a clean halt yields an outcome of registers, gas used, storage, and effects. Any fault ends the run with no outcome, so a caller sees either the full effect of a clean run or none of it. A native cryptographic instruction that would panic on adversarial input is caught and mapped to a `CryptoFault` rather than being allowed to unwind, so no input can crash the machine.

### The container canonical byte format

A contract is published as a `Container` of three parts, the code section, the constant pool, and the entry table. The canonical byte format begins with the four byte format tag `QVM1`, then the code length as a big endian `u32` followed by the code, then the constant count as a big endian `u32` followed by each constant as a big endian 64 bit word, then the entry count as a big endian `u32` followed by each entry. An entry is its four byte selector, its offset as a big endian `u32`, and four storage access lists in order, the scalar reads, the scalar writes, the keyed read bases, and the keyed write bases, each written as a big endian `u32` count followed by that many big endian 64 bit slots. The container identifier is the SHA3-256 of these canonical bytes, so any change to code, to a constant, to an entry offset, or to an access list changes the identifier.

A container is bounded and checked before it is trusted. The code section is at most 65536 bytes, the constant pool at most 4096 entries, and the entry table at most 256 entries. Verification decodes the code linearly, recording the start offset of every instruction, and then requires that every jump and call target and every entry offset lands on a recorded instruction start, that every constant load names an index inside the pool, and that no two entries share a selector. A target that lands inside an instruction, an entry that begins off a boundary, an out of range constant index, an undecodable tail, or a duplicate selector each fail verification.

### Selectors and entries

An entry is named by a four byte selector, computed as the leading four bytes of the SHA3-256 of the entry signature. Dispatch finds the entry whose selector matches, charges a base dispatch cost, and begins execution at that entry offset with the entry's declared access manifest in force. A selector that names no entry is refused with `UnknownSelector`. The constructor entry is named by the genesis signature `@genesis()`, so a deployment enters the container through the same selector mechanism as any other call.

### Native post quantum instructions

The machine carries its cryptography as instructions with fixed semantics.

- `Hash` takes a memory region and writes its SHA3-256 digest, thirty two bytes, to an output pointer.
- `VerifyMl` reads a region laid out as an ML-DSA public key, then a signature, then the message, and sets its destination register to one when the signature verifies and zero when it does not. ML-DSA is the post quantum signature scheme of FIPS 204.
- `VerifySlh` verifies an SLH-DSA signature over the same public key, signature, message layout. SLH-DSA is the hash based signature scheme of FIPS 205.
- `MerkleVerify` checks an inclusion proof over a SHA3-256 tree, reading a root, an index, a leaf, and a sibling path, with a leaf domain tag of `0x00` and an internal node tag of `0x01`, and the index bit at each level choosing the side the running node sits on.
- `Kem` performs an ML-KEM encapsulation, reading an encapsulation key and a seed and writing the shared secret followed by the ciphertext. ML-KEM is the key encapsulation mechanism of FIPS 203.
- `Addr` derives an account address as the SHA3-256 of a one byte scheme identifier followed by a public key, with scheme one for ML-DSA and scheme two for SLH-DSA.

Because these are instructions, their semantics are pinned by the machine and priced by its meter, and the variable cost of hashing, verifying, and proof checking scales with the length the instruction absorbs.

### Deployment and the contract address

A deployment is a call to the network's deploy endpoint, whose address is the SHA3-256 of the label `qtv/vm/deploy` rendered as a Q1 address. The deploy payload frames the container so the node can separate it from the constructor arguments. It begins with the eight byte tag `QDEPLOY2`, then the container length as a big endian `u32`, then the container bytes, then a region of constructor parameters. The tag was `QDEPLOY1` while the call context was 88 bytes and arguments began at 88. The context is now 120 bytes with the paying asset at 88 to 120 and arguments at 120, so the tag was bumped and a container built for the old layout no longer matches, and when any parameter is present a trailing eight byte sentinel `QGENSNTL`. A parameter is an address as thirty two bytes, a `u64` as eight big endian bytes, a `u128` as its low word then its high word each eight big endian bytes, or a guardian set as its addresses laid out one after another.

The contract's own address is derived from the deployer and a nonce, not from the container. It is the SHA3-256 of the label `qtv/vm/contract/`, then the deployer address payload as thirty two bytes, then the nonce as eight little endian bytes, rendered as a Q1 address. A deployer therefore knows a contract's address before the container is finalized, and two deployments from the same account under different nonces resolve to different addresses.

### The call ABI

A call is a four byte selector followed by an argument memory image. The client builds the image by placing each typed argument at its offset and leaving the reserved host context window at the start as zero. An argument is a word as eight big endian bytes, an address as thirty two bytes, or a Q_Name as a thirty two byte label window followed by an eight byte big endian length, so a name argument spans forty bytes. A name resolves to the storage key that is the SHA3-256 of its bare label bytes, the same key the machine derives, and a token id promotes to the leading word of a zeroed thirty two byte region.

A contract reports outcomes through two effect instructions. `Send` records a transfer of an amount to a target read from memory, and `Emit` records an event whose four byte selector is the low word of a register and whose data is a memory region. Effects are charged per byte and bounded by a retained effects cap, and an over cap run faults with `EffectsTooLarge` and surfaces no effects.

### Storage slots as key and value

Storage is a map from a thirty two byte key to a 64 bit word value. `SLoad` reads the key from memory at a pointer and returns the stored word or zero, and `SStore` writes the word to the key. A scalar field lives at a key whose trailing eight bytes hold the slot number. A keyed field, such as a map, lives at a key the contract computes at runtime as the SHA3-256 of an eight byte map base followed by the field key, so the exact slot cannot be listed ahead of time.

Each entry authorises the storage it may touch through its manifest. A scalar read or write is authorised by listing the slot, and a scalar write also grants read of that slot. A keyed field is authorised by declaring its map base, which grants that map's whole keyspace and nothing else. The machine derives the keyed slot itself from the base written at the head of the hash preimage, so a program cannot authorise a key outside a declared map. A load or store to an undeclared slot faults with `UndeclaredSlot`.

### The trusted caller and time

The first 80 bytes of the argument memory are a reserved host context window that the caller does not fill. It holds, in order, `@caller` and `@contract` as thirty two byte addresses and `@time` and `@chain` as eight byte words, which together fill exactly 80 bytes. The client writes all of its own arguments at or above offset 80 and leaves the window as zero, and the compiled entry reads the context only from these fixed offsets. The node owns this window and writes the true caller, the contract address, the time, and the chain into it before the entry runs, so a value a contract reads there is a value the node placed, never one the caller supplied.

A signed order strengthens this binding for calls that carry an owner authorisation. The order preimage is a domain tag that is the constant `QTVSGN01` folded with the chain id, then the contract address, then the selector as a word, then the signer address, then the nonce, then the field values, and it is signed with the owner's ML-DSA key. The signer's per signer nonce lives at the storage key that is the SHA3-256 of `QTVNONCE` followed by the signer address. The public key, signature, and message are laid out in a verify region of the argument memory for the machine's `VerifyMl` instruction to check. Because the chain id is folded into the tag and the node injects the same chain id at `@chain`, an order signed for one chain does not verify on another.

## Rationale

A register machine with fixed width words and checked arithmetic was chosen so that a call has one result on every node, which is the property consensus needs. Publishing the contract as a canonical container with a SHA3-256 identifier lets the network name code by content and makes any change to code or manifest visible as a different identifier. Verifying instruction boundaries and constant indices before a container is trusted keeps a jump or a constant load from ever landing off a defined position at run time. Making the storage each entry may touch a declared manifest lets the node bound a call's footprint, and letting a map declare its base rather than its keys keeps a runtime keyed slot authorised without listing every possible key.

Native post quantum instructions keep the cryptographic surface of a contract inside the machine, so a contract verifies with ML-DSA and SLH-DSA, encapsulates with ML-KEM, and hashes with SHA3-256 under semantics the machine pins and the meter prices, with no classical public key primitive reachable from contract code. Deriving the contract address from the deployer and a nonce keeps the address knowable before deployment and independent of the exact container bytes. Reserving a fixed context window that only the node fills makes the caller and the time unforgeable to contract code, and folding the chain id into a signed order binds an authorisation to the chain it was signed for.

## Backward Compatibility

The QVM container format is the network's execution format from genesis, so there is no earlier deployment to remain compatible with. The four byte format tag `QVM1` names this version of the container, so a future format can be introduced under a new tag without ambiguity. The reserved context window, the four byte selector convention, and the storage key and value model are fixed by this document, and a contract compiled against them runs unchanged.

## Security Considerations

Determinism is the base security property. Checked arithmetic faults rather than wrapping where wrapping was not asked for, a fault ends a run with no committed storage and no effects, and a panic inside a native cryptographic instruction is caught and mapped to a fault, so no input crashes the machine or leaves partial state. Execution is metered and effects are bounded by a retained cap, so a call cannot hash, emit, or transfer without paying and cannot grow effects without limit.

The storage manifest is enforced at run time. An entry may read or write only the scalar slots it declared and may touch a keyed map only under a declared base, with the machine deriving the keyed slot from the base at the head of the hash preimage, so a program cannot reach an undeclared slot. The reserved context window is owned by the node, and because the honest client writes only zeros there and places its own arguments at or above offset 80, the caller cannot present a forged caller, contract, time, or chain to a contract. A signed order is bound to its chain through the folded chain id and to the signer through a per signer nonce, so an order does not replay across chains.

Quantova is on testnet, and external audits of the QVM, its container format, and its native cryptography are still ahead. This document describes the design and its intended properties, and the properties above hold for the format as implemented, not as an audited guarantee.

## Test Cases and Reference Implementation

The QVM implementation carries tests for the container and the machine. Container tests cover the deterministic identifier, the identifier changing on any change to code, manifest, or entry offset, selector derivation, and verification refusing a jump into the middle of an instruction, an entry offset off a boundary, a truncated tail, an out of range constant index, and a duplicate selector. Machine tests cover checked and wrapping arithmetic, memory and storage round trips, the effects cap, and each native instruction against the cryptographic crate, including hashing, ML-DSA and SLH-DSA verification, Merkle inclusion, ML-KEM encapsulation, and address derivation. The contract ABI builders carry tests for the deploy framing and its sentinel, the typed call image with its zeroed context window, name and token id arguments, and a pinned chain bound order preimage vector that the verifier also pins so the signer and the compiled entry cannot drift. The implementation runs on the public testnet, and reference behavior and test vectors are tracked with it.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
