---
title: Waku SWAP Accounting
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [SWAP Accounting](#swap-accounting)
    - [Accounting](#accounting)
    - [Flow](#flow)
    - [Policy](#policy)
    - [Protobuf](#protobuf)
    - [Enhancements](#enhancements)
- [Changelog](#changelog)
- [Copyright](#copyright)

# Abstract

This specification provides a way to do accounting based on the SWAP model proposed in Swarm, as well as being inspired by Bittorrent's economic model for bandwidth incentives. This is used for pairwise accounting to get nodes to cooperate and provide useful service and/or exchange of funds.

**Protocol identifier***: `/vac/waku/swap/2.0.0-alpha1`

# Motivation

The Waku network makes up a service network, and some nodes provide a useful service to other nodes. We want to account for that, laying the foundation for settlement.

TODO: Fill in more

TODO: Game theoretical foundation, Bittorrent, relaxed tit-for-tat

# SWAP Accounting

See [Book of Swarm](https://swarm-gateways.net/bzz:/latest.bookofswarm.eth/the-book-of-swarm.pdf) section 3.2. on Peer-to-peer accounting etc., for more context and details.

This approach is based on communicating payment treshholds and sending cheques
as indications of later payments. The first part is done with a handshake, the
second done once payment threshhold is hit.

TODO: Illustrate payment and disconnection treshholds

TODO: Elaborate on how accounting works with amount in the context of e.g. store

TODO: Illustrate flow

A cheque is best thought of as a promise to pay at a later date.

TODO Specify chequebook

## Accounting

Nodes perform their own accounting for each relevant peer based on some "volume"/bandwidth metric. For now we take this to mean the number of `WakuMessage`s exchanged.

Additionally, a price is attached to each unit. For now, this is is simple a "karma counter" and equal to 1 per message.

NOTE: This may later be complemented with other metrics, either as part of SWAP or more likely outside of it. For example, online time can be communicated and attested to as a form of enhanced quality of service to inform peer selection.

## Flow

Assuming we have two store nodes, one operating mostly as a client (A) and another as server (B).

1. Node A performs a handshake with B node. B node responds and both nodes communicate their payment threshhold.
2. Node A and B creates an accounting entry for the other peer, keep track of peer and current balance.
3. Node A issues a `HistoryRequest`, and B responds with a `HistoryResponse`. Based on the number of WakuMessages in the response, both nodes update their accounting records.
4. When payment threshhold is reached, Node A sends over a cheque to reach a neutral balance. Settlement of this is currently out of scope, but would occur through a SWAP contract (to be specified).
5. If disconnect threshhold is reached, Node B disconnects Node A.

## Policy

- If a node reaches a disconnect threshhold, which MUST be outside the payment threshhold, it SHOULD disconnect the other peer. For the PoC, this behavior can be mocked.
- If a node is within payment balance, the other node SHOULD stay connected to it.
- If a node receives a valid Cheque it SHOULD update its internal accounting records.
- If any node behaves badly, the other node is free to disconnect and pick another node. Peer rating is out of scope of this specification.

## Protobuf

```
// TODO What information to include here? For now "payment" threshhold
message Handshake {
    bytes payment_threshhold = 1;
}

// TODO Signature?
// Should probably be over the whole Cheque type
message Cheque {
    bytes beneficiary = 1;
    // TODO epoch time or block time?
    uint32 date = 2;
    // TODO ERC20 extension?
    // For now karma counter
    uint32 amount = 3;
}
```

## Enhancements

1. Settlements. At a later stage, these cheques can be redeemed. For now they
are better thought of as a form of Karma.

2. More information to make a decision. E.g. incorporate QoS metrics such as
   online time to inform peer selection (slightly orthogonal to this).

# Changelog

TBD.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
