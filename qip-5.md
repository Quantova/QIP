---
qip: 5
title: Gateway RPC
status: Living
type: Standards Track
category: Interface
author: Quantova Inc
created: 2026-07-27
---

# QIP 5, Gateway RPC

## Abstract

This proposal documents the gateway RPC, the request and reply surface a node exposes to wallets, explorers, and tooling. The wire is an HTTP POST to a path of the form `/v1/<method>` carrying a flat JSON body, and every reply is a JSON object. The method is the last segment of the path. The body holds the arguments of that method as top level keys with no envelope around them. This document records the transport, the full set of methods that the node answers, and the request and reply shape of each. It also records the rule the reference client enforces, that the plaintext transport refuses to talk to a gateway that is not on the loopback interface, because over an open link the fee and nonce a signer trusts would be unauthenticated and rewritable.

## Motivation

A wallet needs a small, stable, and legible way to read chain state and to hand a signed transaction to a node. A method per path over plain HTTP gives that. Each call names one method, carries one flat object of arguments, and receives one object back, so a client can be written against it with nothing more than an HTTP library and a JSON parser. The surface stays narrow on purpose. The node signs nothing and holds no user keys. A transaction is built and signed on the client and reaches the node already sealed, so the trust a caller places in the gateway is bounded to the accuracy of the state it reports.

That bound is the reason for the loopback rule. When a caller asks a gateway for the current fee and its own account nonce and then signs a transfer around those values, a gateway reached over an open link could return a rewritten fee or nonce and steer the signature toward draining the account. The reference client therefore refuses to speak the plaintext transport to any host that does not resolve to the loopback address, and directs the operator to a local node or a secure tunnel.

## Specification

### Transport

A call is an HTTP POST over TCP. The request line names the method in its path.

```
POST /v1/node_info HTTP/1.1
Content-Type: application/json
Content-Length: 2

{}
```

The path is `/v1/` followed by the method name. A path that does not begin with `/v1/` is answered with a `404` and the error code `unknown_method`. The request body is a single JSON object whose keys are the arguments of the method. There is no outer envelope. There is no `method` field, no `id`, and no version marker inside the body. A method that takes no argument accepts either an empty body or the object `{}`, since an empty body is read as an empty object.

Every reply is a JSON object sent with `Content-Type` of `application/json`. A success is HTTP `200` and the body is the result object. A failure carries a non `200` status and a body of the form below, where `error` is a short machine code and `message` is a human sentence.

```
{"error": "bad_address", "message": "the address is not a q1 Bech32m address"}
```

Only POST is served. An `OPTIONS` request is answered `204` for cross origin preflight, and every reply carries permissive cross origin headers so a browser client can call the node directly. Any other verb is `405` with the code `method_not_allowed`.

The server guards its own resources. The request head is capped and a head past the cap is `431` with `head_too_large`. The request body is capped at two mebibytes and a larger body is `413` with `too_large`. Total open connections are capped and a call over the cap is `503` with `busy`. Connections from one address are capped and a call over that cap is `429` with `too_many`. When the node cannot take the request the gateway answers `503` with `unavailable`.

### Number encoding

Quantities that fit a machine word are carried as JSON numbers. These are heights, nonces, meter limits, counts, and enumeration codes. Quantities that can exceed a machine word are carried as decimal strings so no precision is lost in a JSON number. These are balances, fees, transfer values, and the total supply. A client parses the string form back into a wide integer.

### Methods

The node answers the following methods and no others. A method not in this set is `404` with `unknown_method`.

#### node_info

Takes no argument. Reports the chain identity and the current fee schedule.

```
{
  "chain_id": "Q-test-net-1",
  "genesis_hash": "...",
  "head_height": 1024,
  "asset": "QTOV",
  "denomination": "Quon",
  "fee": {
    "transfer_micro_usd": "...",
    "rate_micro_usd_per_qtov": "...",
    "quon_per_qtov": "...",
    "transfer_quon": "..."
  },
  "version": "..."
}
```

The `fee` block reports the transfer fee both in micro USD and in Quon, the base unit, alongside the base units per QTOV. A wallet reads `transfer_quon` as the fee it must cover and can check it against a ceiling before it signs.

#### head

Takes no argument. Reports the height of the head block, its identifier, and the state root at the head. The `block` field is null until the chain has passed the minimum height.

```
{"height": 1024, "block": "...", "state_root": "..."}
```

#### finalized_head

Takes no argument. Reports the height that finality has reached.

```
{"head": 1000}
```

#### chain_params

Takes no argument. Reports the staking and governance constants the node runs under. The `staking` block carries the native unit, the minimum stake, the staking pool, the session emission, the session length in days, the high session transaction count, the mainnet blackout days, the bond lock days, the unbonding days, and the reward vesting days. The `governance` block carries the maximum conviction factor and the array of governance tracks, each with a code, a deposit, a threshold in basis points, and a period in seconds.

#### supply

Takes no argument. Reports the total supply in Quon as a decimal string.

```
{"supply_quon": "..."}
```

#### validators

Takes no argument. Reports the active validator set. Each entry carries the validator address and its staked weight.

```
{"count": 7, "validators": [{"address": "Q1...", "stake": 100000}]}
```

#### staking_state

Takes no argument. Reports the live staking figures, the reward pool, the treasury, the price in micro USD per QTOV, whether mainnet has started, and the total locked under governance.

```
{
  "reward_pool": 0,
  "treasury": 0,
  "price_micro_usd_per_qtov": "...",
  "mainnet_started": false,
  "governance_locked": "0"
}
```

#### get_account

Takes an `address`. The address must be a Q1 Bech32m address, and an address that does not parse is `400` with `bad_address`. Reports the account nonce, its balance as a decimal string, its signature scheme code, and whether a public key has been registered for it.

```
{"address": "Q1..."}
```

```
{"address": "Q1...", "nonce": 3, "balance": "1000000", "scheme": 1, "has_key": true}
```

#### submit_transaction

Takes a `tx` that is the hex of the encoded transaction wrapper. A `tx` that is not hex is `400` with `bad_request`. Bytes that do not decode into a wrapper are answered as a rejection with the reason `malformed`. A wrapper that decodes is offered to the mempool and the reply is a verdict.

```
{"tx": "..."}
```

An accepted transaction reports its state, `fresh` for a newly seen transaction and `known` for one the pool already held, together with its identifier.

```
{"verdict": "accepted", "state": "fresh", "tx_id": "..."}
```

A rejected transaction reports a reason code. The reasons are `unknown_sender`, `unsupported_scheme`, `bad_signature`, `bad_nonce`, `bad_call`, `self_transfer`, `meter_limit_too_low`, `fee_too_low`, `insufficient_funds`, `wrong_chain`, `pool_full`, `sender_queue_full`, `rate_limited`, and `malformed`. A `bad_nonce` rejection also reports the `expected` and the `got` nonce.

```
{"verdict": "rejected", "reason": "bad_nonce", "expected": 4, "got": 2}
```

#### get_transaction

Takes a `tx_id`. Reports the status of that transaction, one of `finalised`, `pending`, or `unknown`. A finalised transaction also reports its height and its block and the decoded transaction fields. A pending transaction reports the decoded fields without a height. An unknown transaction reports only its identifier and the status.

```
{"tx_id": "..."}
```

```
{
  "tx_id": "...",
  "status": "finalised",
  "height": 1000,
  "block": "...",
  "from": "Q1...",
  "to": "Q1...",
  "value": "1000",
  "fee": "210",
  "nonce": 3,
  "meter_limit": 1210,
  "scheme": 1,
  "signature": "...",
  "raw": "..."
}
```

The transaction fields are the same shape wherever the node returns a transaction. `from` and `to` are addresses, `value` and `fee` are decimal strings, `nonce` and `meter_limit` and `scheme` are numbers, and `signature` and `raw` are hex, with `raw` the full re encoded wrapper.

#### pending

Takes no argument. Reports the transactions the mempool is holding, each with its identifier and its decoded fields.

```
{"count": 2, "transactions": [{"tx_id": "...", "from": "Q1...", "to": "Q1..."}]}
```

#### get_block

Takes either a `height` or a `block` identifier. A request with neither is `400` with `bad_request`, and a height or identifier with no finalised block is `404` with `not_found`. Reports the block header and the identifiers of the transactions it carries.

```
{"height": 1000}
```

```
{
  "height": 1000,
  "block": "...",
  "parent": "...",
  "state_root": "...",
  "proposer": "Q1...",
  "time": 1753600000,
  "tx_count": 1,
  "extra_data": "...",
  "tx_ids": ["..."]
}
```

#### get_container

Takes an `address` that must be a Q1 address. An address that holds no contract is `404` with `not_found`. Reports the container code at that address as hex, with its byte length.

```
{"address": "Q1..."}
```

```
{"address": "Q1...", "container": "...", "size": 512}
```

#### get_storage

Takes an `address` that must be a Q1 address. Reports the occupied storage slots of the contract at that address. Each slot carries its 32 byte key as hex and its value as a decimal string.

```
{"address": "Q1..."}
```

```
{"address": "Q1...", "slots": [{"slot": "...", "value": "5"}]}
```

#### get_events

Takes a `height`. A request with no height is `400` with `bad_request`. Reports the events recorded at that height, each with the emitting contract, the four byte selector as hex, and the event data as hex.

```
{"height": 1000}
```

```
{
  "height": 1000,
  "count": 1,
  "events": [{"contract": "Q1...", "selector": "6e82531d", "data": "0000000000000005"}]
}
```

#### burn_block

Takes a `height`. A height with no archived record is `404` with `not_found`. Reports the archived block at that height, carrying the raw header bytes as hex, the finality certificate as hex, and the event leaves recorded there as an array of hex leaves.

```
{"height": 1000}
```

```
{"height": 1000, "header_bytes": "...", "certificate": "...", "events": ["..."]}
```

#### burn_heights_after

Takes a `cursor`. A request with no cursor is `400` with `bad_request`. Reports the archived heights greater than the cursor, which lets a follower walk the archive forward.

```
{"cursor": 900}
```

```
{"cursor": 900, "count": 2, "heights": [950, 1000]}
```

## Rationale

A method per path with a flat body was chosen over an enveloped scheme so a call is legible on the wire and trivial to construct. The method a reader cares about is the last path segment, and the arguments are the top level keys of the body with nothing wrapping them. There is no `method` field to reconcile against the path, no `id` to correlate, and no version marker inside the body, because the version already lives in the `/v1/` path prefix and one call maps to one reply over one connection. The naming stays in Quantova vocabulary throughout. Methods read as `node_info`, `get_account`, `submit_transaction`, and the like, and there is no borrowed method namespace anywhere on the surface.

Wide integers travel as decimal strings and word sized quantities travel as JSON numbers. A balance, a fee, a transfer value, and the supply can exceed a machine word, so carrying them as strings keeps every digit rather than trusting a JSON number to hold them. Heights, nonces, meter limits, and counts fit a word and stay numbers so a client reads them without a parse step.

POST is the only verb because every call changes nothing on its own and yet may carry a body, and folding reads and the one write onto a single verb keeps the surface uniform. The cross origin headers are permissive so a browser wallet can reach a node directly, which suits public read access to a testnet.

## Backward Compatibility

There is no earlier gateway surface to remain compatible with. The `/v1/` prefix reserves room for a later revision to live beside this one under a new prefix, so a future change need not break a client written against `/v1/`. Within `/v1/` the intent is that new methods and new reply fields are added and existing fields keep their meaning, so a client that ignores unknown fields keeps working as the surface grows. Quantova is on testnet and the surface is Living, so a method may still be added or a shape refined before it is declared final.

## Security Considerations

The gateway holds no user keys and signs nothing. A transaction reaches the node already signed, so a caller trusts the gateway only for the accuracy of the state it reports, not with the authority to move funds. The sensitive path is the read a signer depends on before it signs. A wallet reads the fee from `node_info` and its own nonce from `get_account` and seals a transfer around those two values. A gateway reached over an open link could return a rewritten fee or nonce and push the signature toward draining the account.

The reference client closes that path by refusing the plaintext transport to any host that does not resolve to the loopback address. It requires an `http://` base, resolves the host, and if the resolved address is not loopback it returns an error and points the operator at a local node or a secure tunnel rather than sending the request. On top of that the client checks the quoted fee against a caller supplied ceiling and refuses to sign when the gateway asks for more than the ceiling allows, so a stale or hostile quote cannot silently inflate what the signer pays.

The server side bounds its own exposure. It caps the request head and the request body, caps the total number of open connections, and caps the connections from a single address, answering an over the cap call with a distinct status rather than absorbing the load. These limits blunt a flood but are not a substitute for network placement, and a public node should still sit behind a tunnel or a reverse proxy that terminates transport security.

Quantova is on testnet and the stack has not been through an external audit. This document records the surface and the guards that are in the code today, not a certified security posture.

## Test Cases and Reference Implementation

The server is the `qtv-gateway` crate. Its `service.rs` builds each request from the method and the flat body and renders each reply, and its `http.rs` carries the POST routing, the `/v1/` prefix check, the empty body handling, the verb and size guards, and the connection limiter, each covered by unit tests in that file. The reference client is the QCore.rs crate. Its `Client` posts to `/v1/<method>`, and the `*_body` builders such as `account_body`, `transaction_body`, `submit_body`, `storage_body`, and `events_body` produce the flat request objects while the parsers turn each reply back into a typed value. The loopback refusal lives in the QCore.rs `http.rs` post path. The sibling QCore.js and QCore.py bindings speak the same wire. Reference behavior and vectors are tracked with these crates.

## Copyright

Copyright 2026 Quantova Inc. See [LICENSE](LICENSE).
