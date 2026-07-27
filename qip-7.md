---
qip: 7
title: QAsset Fungible Token Standard
status: Draft
type: Standards Track
category: Interface
author: Quantova Inc
created: 2026-07-27
---

# QIP 7, QAsset Fungible Token Standard

## Abstract

This proposal describes QAsset, a fungible token standard for Quantova. A QAsset token is a Quanta contract that runs on the QVM and keeps every holder balance inside its own contract storage. The standard is defined by a reference Quanta contract that declares an asset, a total supply, a balances map, and the entries that move value across that map. A caller reaches an entry by sending a call whose leading four bytes are a selector that names the entry and whose remaining bytes carry the encoded arguments. The node fills a reserved context window at the front of that argument memory with the account it has authenticated as the caller, so an entry that debits the caller acts on the account the node vouches for rather than on any account the client writes. QAsset is a Draft standard. Its surface is set by the reference contract and not by a frozen specification, so the entry set and the value widths recorded here will keep maturing toward mainnet.

## Motivation

A fungible token is the most common contract on any settlement network, so Quantova needs one shared shape for it. A shared shape lets a wallet, an explorer, and a market read a balance, follow a transfer, and total the supply of any conforming token without special casing each deployment. The shape has to be native Q vocabulary that runs on the QVM in Quanta rather than a form carried over from another chain, because the storage model, the call format, and the caller authentication are all Quantova mechanisms. QAsset records that shape. Balances live in contract storage, value moves through contract entries, and the account credited or debited by a transfer is the caller the node injects, so the standard rests on the parts of the stack that already carry Quantova's own authentication and storage.

## Specification

The reference contract is examples/QAsset.qs in the Quanta contract language. It reads as follows.

```
import { Map } from "quantova/stdlib";
contract QAsset {
  asset QRC;
  state {
    owner: Q_Address;
    total_supply: u128;
    balances: Map<Q_Address, u128>;
  }
  genesis {
    owner = deploy_params.owner;
    total_supply = deploy_params.initial_supply;
  }
  entry mint(order: MintOrder signed by owner)
    mints QRC
    writes(total_supply, balances)
  {
    total_supply += order.amount;
    balances.credit(order.to, order.amount);
    emit Minted(order.to, order.amount);
  }
  entry transfer(to: Q_Address, amount: u128)
    writes(balances)
  {
    guard amount > 0;
    balances.debit(caller, amount);
    balances.credit(to, amount);
    emit Transferred(caller, to, amount);
  }
  event Minted(to: Q_Address, amount: u128);
  event Transferred(sender: Q_Address, to: Q_Address, amount: u128);
}
```

The contract declares an asset named QRC with `asset QRC`. It holds three pieces of state. The `owner` is a Q_Address that names the account allowed to mint. The `total_supply` is a u128 that tracks how many units exist. The `balances` field is a Map from Q_Address to u128 that records the balance of every holder. At genesis the owner and the starting supply are read from the deploy parameters the deployer supplied when the container was placed on chain, which the QCore.rs deploy helper frames as an owner address and an initial supply behind a sentinel.

Each piece of state resolves to a thirty two byte storage key. A scalar field such as the total supply resolves to a scalar slot key. A balance for a holder resolves to a map key the QVM derives by hashing the map's domain tag together with the holder address. The QCore.rs contract helpers rebuild these same keys, so a client reads the total supply or a holder balance by reconstructing the key and reading the value the node reports for it, and a key that has never been written reads as zero. In the reference contract a balance and the total supply are declared as u128. The exact stored width for a u128 value is among the details still being finalized for this Draft, and the current QCore.rs read helper returns a word sized value for a slot key.

A call to a token entry is a byte string. The leading four bytes are the selector that names the entry, so the mint entry and the transfer entry each carry their own selector. The bytes after the selector are a flat argument memory. Each declared argument occupies a fixed region of that memory at an offset the compiled entry expects. A word argument occupies eight bytes in big endian order, an address argument occupies thirty two bytes, and a name argument occupies a thirty two byte label window followed by an eight byte length word. The QCore.rs helper that frames a call writes the selector and then places each argument at its offset, which is how `transfer` receives its `to` address and its `amount`.

The first eighty bytes of the argument memory are a reserved host context and are left zeroed by the client. Before the entry runs the node fills that context window with the account it has authenticated as the caller, the contract address, the block time, and the chain id. The reference `transfer` entry debits `caller` and credits `to`, and because `caller` comes from the context the node wrote rather than from any argument the client supplied, a client cannot debit an account it does not control. This is the property that stops a transfer from being spoofed. The value eighty is fixed by the node and mirrored by the QCore.rs constant CONTRACT_CONTEXT_BYTES so the client and the node agree on where arguments begin.

The `mint` entry takes an order that is `signed by owner`, so it is reached through a signed order rather than a bare call. The QCore.rs order builder forms a canonical order message that binds a domain tag folded with the chain id, the contract address, the selector, the signer address, a nonce, and the field values, then signs that message with ML-DSA, the post quantum module lattice signature. The signature, the public key, and the message are placed in a verify region of the argument memory that the compiled entry reads and checks. The nonce is tracked in a per signer nonce slot so the same order cannot be replayed, and folding the chain id and the contract address into the message stops an order signed for one chain or one contract from being replayed against another.

The reference contract exposes two write entries. The `mint` entry raises the total supply and credits the order's recipient, and it emits `Minted(to, amount)`. The `transfer` entry guards that the amount is above zero, debits the caller, credits the recipient, and emits `Transferred(sender, to, amount)`. A balance and the total supply are not entries. They are reads against contract storage as described above. An event surfaces to a client through the QCore.rs event helpers as a contract address, a four byte selector, and a data payload, and a single word payload decodes as a big endian word.

QAsset is a Draft, and the reference contract fixes only the surface shown here. It does not define an approval entry or a transfer from entry, so an allowance model where a holder authorizes a third party to move a bounded amount is not part of this standard yet. It does not define a burn entry, though the sibling Token example does, so a supply reducing entry may be aligned into the standard later. It does not define a dedicated balance read entry, because a balance is read from storage. The exact stored width for u128 balances and the precise binding of an order signer to the named owner are among the items to be pinned before this Draft advances. Any of these may be added as the standard matures, and they will be recorded here when they are.

## Rationale

Defining the standard by a reference contract keeps the description honest, because the entries, the state, and the events are exactly what a deployed contract runs rather than a wish list that no code satisfies. Placing balances in the contract's own storage keeps each token self contained and lets any reader reconstruct a balance key without a registry. Naming an entry by a four byte selector and carrying arguments at fixed offsets keeps a call compact and lets the compiled entry find each argument without parsing. Reserving the context window for the node and filling the caller there is what makes a transfer trustworthy, because the account moved is the one the node authenticated. Choosing ML-DSA for a signed order keeps the authorization path post quantum, in line with the rest of Quantova.

## Backward Compatibility

QAsset is a new Draft standard and there is no earlier fungible token standard on Quantova to stay compatible with. Because the standard is defined by a reference contract, a token that follows the reference shape conforms, and a change to the standard is a change to that reference shape. Quantova is on testnet, so the surface here can still change before it is fixed for mainnet, and this document will track those changes.

## Security Considerations

The account credited and debited by a transfer is the caller the node injects into the reserved context window, not a value the client supplies, so a client cannot move balances it does not control. A signed order binds its message to the chain id, the contract address, the selector, and a nonce, so an order cannot be replayed across chains or contracts or replayed twice on the same chain. The order is signed with ML-DSA, so the authorization path is post quantum. This Draft still has open items. Fully binding an order signer to the declared owner and fixing the stored width of a u128 balance are being specified before mainnet. Quantova is on testnet and the token standard, the contract language, and the QVM have not had an external audit. This document describes the intended design and its properties, and it should not be read as a statement that the design has been independently reviewed.

## Test Cases and Reference Implementation

The reference contract is examples/QAsset.qs in the Quanta contract language, alongside the sibling Token and IssuerToken examples that show a burn entry and a guardian gated issuance model. The client side call framing, the storage key derivation, the signed order construction, and the event parsing are implemented in QCore.rs at src/contract.rs, whose unit tests pin the call layout, the reserved context window, the canonical order preimage, and the storage and event responses. The standard runs against the public testnet.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
