---
title: Waku
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

## Table of Contents

TODO

## Abstract

Waku is a privacy-preserving peer-to-peer messaging protocol for resource restricted devices. It implements PubSub in libp2p and adds capabilities on top of it. These capabilities are: (i) retrieving historical messages for mostly-offline devices (ii) adaptive nodes, allowing for heterogenerous nodes to contribute, and (iii) bandwidth preservation for light nodes. This makes it ideal for running a p2p protocol on mobile.

Historically, it has its roots in Waku v1 [xx5], which stems from Whisper [xx6], originally part of the Ethereum stack. However, Waku v2 acts more as a thin wrapper for PubSub and has a different API. It is implemented in an iterative manner where initial focus is on porting essential functionality to libp2p. See here for a rough road map [xx7].

## Motivation

TODO

*NOTE: Should elaborate on privacy-preserving messaging for resource restricted devices, etc. Also Waku v1 history briefly.*

## Definitions

TODO

## Underlying Transports and Prerequisites

TODO Right now this is more like a set of components

### Peer Discovery

WakuSub and PubSub don't provide an peer discovery mechanism. This has to be provided for by the environment.

### PubSub interface

Waku v2 is implementing the PubSub interface in Libp2p. See PubSub spec [xx2] for more details.

### Protocol Identifier

The current protocol identifier is: `/wakusub/0.0.1` [xx1].

**TODO Update to 2.0.0-alpha0**

### FloodSub

WakuSub is currently a subprotocol of FloodSub. Future versions of WakuSub will support GossipSub 1 [xx3] and GossipSub 1.1 [xx4].

### Bridge mode

To maintain compatibility with Waku v1, a bridge mode can be achieved. See separate spec.

TODO Detail this in a separate spec

## Wire Specification

We are using protobuf RPC messages between peers. Here's what a message looks like, as defined in the PubSub interface.


*NOTE: Should contain protobuf definitions that cover essentials of Waku v1. In cases where it doesn't cover, we can defer to siblings/child specs, e.g. such as the data field for encryption, etc.*

### Messages

```
message Message {
	optional string from = 1;
	optional bytes data = 2;
	optional bytes seqno = 3;
	repeated string topicIDs = 4;
	optional bytes signature = 5;
	optional bytes key = 6;
}
```

TODO Detail options and what's in data, additional methods, etc

### SubOpts

TODO

## Changelog

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

xx1: https://docs.libp2p.io/concepts/protocols/

xx2: [PubSub interface for libp2p (r2, 2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md)

xx3: https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md

xx4: https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md

xx5: specs.vac.dev/waku/waku.html

xx6: https://eips.ethereum.org/EIPS/eip-627

xx7: https://vac.dev/waku-v2-plan
