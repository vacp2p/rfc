---
title: Waku
version: 2.0.0-alpha0
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

**NOTE: This is a very early raw version of Waku v2 over libp2p**

## General notes and TODOs

### General
In general, we want to remove as much as possible. WakuSub should be a thin layer on top of PubSub.

A good question to ask: what is the minimal subset we have to support?

How can we defer specific properties to other protocols?

### Data field
Can we assume data field stays the same? There is nothing in data field about RLPx AFAICT:
https://github.com/vacp2p/specs/blob/master/specs/waku/envelope-data-format.md

### How do we deal with PoW etc?
If basically only full nodes set PoW to 0 then it be fine. But if light nodes also set this (afair they do) we have a compatibility issue. Though we also have mailservers.

Simplest might might be to basically send both envelopes, just take out the data thing:
- v1 [ expiry ttl topic data nonce ]
- v2 [ ... data ... ]

### Get rid of privileged point of node for RPC?
We could have JSON RPC for now and then use alt method later. We need to make
sure the interface with Stimbus and Core/Desktop makes sense.

It'd be useful if a node in browser could use another node without giving away
private keys (WalletConnect, e.g.).

### Are filters a useful abstraction?
Not sure I like this filter businesss, just deliver on a topic. If we want to do
that it can be done separately with a shim and have that specified at a
different layer. Removing coupling of routing and crypto.

### Consumer spec
Consumer: https://specs.status.im/spec/10

### Factoring out
Separate out into constituent pieces, keep main spec small here. What are the pieces?
- RPC/Node API
- Filter shim / crypto key compatibility
- Data field
- Waku v1 bridge 

### App topics and waku topics
Ensure path forward for sharding topics. See Eth2 networking spec too.

Ensure topic interest in payload (not for routing, only edge nodes). Current Waku topics.

### Clarity of purpose
Make it very clear what we provide over e.g. FloodSub.
- Privacy, resource restriction (offline, topic interest, etc)
- Thin layer on top of PubSub

## Table of Contents

TODO

## Abstract

TODO

*NOTE: Should highlight things such as mostly-offline bandwidth-constrainted smartphones, historic messages, adaptive nodes, topic interest.*

## Motivation

TODO

*NOTE: Should elaborate on privacy-preserving messaging for resource restricted devices, etc. Also Waku v1 history briefly.*

## Definitions

TODO

## Underlying Transports and Prerequisites

TODO

*NOTE: Should highlight use of libp2p, PubSub (with WakuSub<FloodSub initiailly), and optionally bridging with Waku v1.*

We are using protobuf RPC messages between peers.

### PubSub interface

Waku v2 is implementing the PubSub interface in Libp2p. See PubSub spec [xx2] for more details.

### Protocol Identifier

The current protocol identifier is: `/wakusub/0.0.1` [xx1].

**TODO Update to 2.0.0-alpha0**

## Wire Specification

TODO

*NOTE: Should contain protobuf definitions that cover essentials of Waku v1. In cases where it doesn't cover, we can defer to siblings/child specs, e.g. such as the data field for encryption, etc.*

## Changelog

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Footnotes

TODO

xx1: https://docs.libp2p.io/concepts/protocols/

xx2: [PubSub interface for libp2p (r2, 2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md)
