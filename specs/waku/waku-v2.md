---
title: Waku
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Definitions](#definitions)
- [Goals](#goals)
- [Underlying Transports and Prerequisites](#underlying-transports-and-prerequisites)
  * [Peer Discovery](#peer-discovery)
  * [PubSub interface](#pubsub-interface)
  * [Protocol Identifier](#protocol-identifier)
  * [FloodSub](#floodsub)
  * [Bridge mode](#bridge-mode)
- [Wire Specification](#wire-specification)
  * [Messages](#messages)
  * [SubOpts](#subopts)
- [Changelog](#changelog)
- [Copyright](#copyright)
- [References](#references)


## Abstract

Waku is a privacy-preserving peer-to-peer messaging protocol for resource
restricted devices. It implements PubSub over libp2p and adds capabilities on
top of it. These capabilities are: (i) retrieving historical messages for
mostly-offline devices (ii) adaptive nodes, allowing for heterogeneous nodes to
contribute, and (iii) bandwidth preservation for light nodes. This makes it
ideal for running a p2p protocol on mobile.

Historically, it has its roots in [Waku v1](specs.vac.dev/waku/waku.html), which
stems from [Whisper](https://eips.ethereum.org/EIPS/eip-627), originally part of
the Ethereum stack. However, Waku v2 acts more as a thin wrapper for PubSub and
has a different API. It is implemented in an iterative manner where initial
focus is on porting essential functionality to libp2p. See [rough road
map](https://vac.dev/waku-v2-plan).

## Motivation and goals

1. **Generalized messaging.** Many applications requires some form of messaging
   protocol to communicate between different subsystems or different nodes. This
   messaging can be human-to-human or machine-to-machine or a mix.

2. **Peer-to-peer.**These applications sometimes have requirements that make
   them suitable for peer-to-peer solutions.

3. **Resource restricted**.These applications often run in constrained
   environments, where resources or the environment is restricted in some
   fashion. E.g.:

- limited bandwidth, CPU, memory, disk, battery, etc
- not being publicly connectable
- only being intermittently connected; mostly-offline

4. **Privacy.** These applications have a desire for some privacy guarantees,
   such as pseudonymity, metadata protection in transit, etc.

Waku provides a solution that satisfies these goals in a reasonable matter.

## Definitions

TODO

## Underlying Transports and Prerequisites

TODO Right now this is more like a set of components

### Peer Discovery

WakuSub and PubSub don't provide an peer discovery mechanism. This has to be
provided for by the environment.

### PubSub interface

Waku v2 is implementing the PubSub interface in Libp2p. See [PubSub interface
for libp2p (r2,
2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md) for
more details.

### Protocol Identifier

The current [protocol identifier](https://docs.libp2p.io/concepts/protocols/)
is: `/wakusub/2.0.0-alpha1`.

### FloodSub

WakuSub is currently a subprotocol of FloodSub. Future versions of WakuSub will
support [GossipSub
v1.0](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)
and [GossipSub
1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md).

### Bridge mode

To maintain compatibility with Waku v1, a bridge mode can be achieved. See
separate spec.

TODO Detail this in a separate spec

## Wire Specification

We are using protobuf RPC messages between peers. Here's what the protobuf messages looks like, as defined in the PubSub interface. Please see [PubSub interface spec](https://github.com/libp2p/specs/blob/master/pubsub/README.md) for more details.

In this section we specify two things:
1) How WakuSub is using these messages.
2) Additional message types.

### Protobuf

```
message RPC {
	repeated SubOpts subscriptions = 1;
	repeated Message publish = 2;

	message SubOpts {
		optional bool subscribe = 1;
		optional string topicid = 2;
	}
}

message Message {
	optional string from = 1;
	optional bytes data = 2;
	optional bytes seqno = 3;
	repeated string topicIDs = 4;
	optional bytes signature = 5;
	optional bytes key = 6;
}
```

WakuSub does not currently use the `ControlMessage` defined in GossipSub. However, later versions will add likely add this capability.

`TopicDescriptor` as defined in the PubSub interface spec is not currently used.

### Historical message support

TODO(Dean): Fill out this section with historical message API.

### Options used

*NOTE: Should contain protobuf definitions that cover essentials of Waku v1. In cases where it doesn't cover, we can defer to siblings/child specs, e.g. such as the data field for encryption, etc.*

## Changelog

TODO(Oskar): Update changelog once we are in draft, which is when
implementation matches spec

Intial draft version. Released []()

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

1. [Protocol Identifiers](https://docs.libp2p.io/concepts/protocols/)

2. [PubSub interface for libp2p (r2,
   2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md)

3. [GossipSub
   v1.0](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)

4. [GossipSub
   v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md)

5. [Waku v1 spec](specs.vac.dev/waku/waku.html)

6. [Whisper spec (EIP627)](https://eips.ethereum.org/EIPS/eip-627)

7. [Waku v2 plan](https://vac.dev/waku-v2-plan)
