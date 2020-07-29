---
title: Waku
version: 2.0.0-alpha2
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

## Table of Contents

- [Abstract](#abstract)
- [Motivation and goals](#motivation-and-goals)
- [Definitions](#definitions)
- [Underlying Transports and Prerequisites](#underlying-transports-and-prerequisites)
  * [Peer Discovery](#peer-discovery)
  * [PubSub interface](#pubsub-interface)
  * [Protocol Identifier](#protocol-identifier)
  * [FloodSub](#floodsub)
  * [Bridge mode](#bridge-mode)
- [Wire Specification](#wire-specification)
  * [Protobuf](#protobuf)
  * [Message](#message)
  * [SubOpts](#subopts)
  * [Historical message support](#historical-message-support)
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

2. **Peer-to-peer.** These applications sometimes have requirements that make
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
for libp2p (r2, 2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md) 
for
more details.

### FloodSub

WakuSub is currently a subprotocol of FloodSub. Future versions of WakuSub will
support [GossipSub v1.0](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)
and [GossipSub 1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md).

### Bridge mode

To maintain compatibility with Waku v1, a bridge mode can be achieved. See
separate spec.

TODO Detail this in a separate spec

## Network interaction domains

While Waku is best though of as a single cohesive thing, there are three network
interaction domains: (a) gossip domain (b) discovery domain (c) req/resp domain.

### Protocol Identifiers

The current [protocol identifiers](https://docs.libp2p.io/concepts/protocols/) are:

1. `/vac/waku/relay/2.0.0-alpha2`
2. `/vac/waku/store/2.0.0-alpha2`
3. `/vac/waku/filter/2.0.0-alpha`

TODO: Protocol identifiers are subject to change, e.g. for request-reply

## Gossip domain

**Protocol identifier***: `/vac/waku/relay/2.0.0-alpha2`

### Wire Specification

We are using protobuf RPC messages between peers. Here's what the protobuf messages looks like, as defined in the PubSub interface. Please see [PubSub interface spec](https://github.com/libp2p/specs/blob/master/pubsub/README.md) for more details.

In this section we specify two things how WakuSub is using these messages.

### Protobuf

```protobuf
message RPC {
  repeated SubOpts subscriptions = 1;
  repeated Message publish = 2;

  message SubOpts {
    optional bool subscribe = 1;
    optional string topicid = 2;
  }

message Message {
  optional string from = 1;
  optional bytes data = 2;
  optional bytes seqno = 3;
  repeated string topicIDs = 4;
  optional bytes signature = 5;
  optional bytes key = 6;
  optional string contentTopic = 7;
}
```

WakuSub does not currently use the `ControlMessage` defined in GossipSub.
However, later versions will add likely add this capability.

`TopicDescriptor` as defined in the PubSub interface spec is not currently used.

### RPC

These are messages sent to directly connected peers. They SHOULD NOT be
gossiped. See section below on how the fields work.

TODO Give brief summary of each type here

### Message

The `from` field MAY indicate which peer is publishing the message.

The `data` field SHOULD be filled out with whatever payload is being sent. Encryption of this field is done at a separate layer.

The `seqno` field MAY be used to provide a linearly increasing number. See PubSub spec for more details.

The `topicIDs` field MUST contain the topics that a message is being published on.

The `signature` field MAY contain the signature of the message, thus providing authentication of the message. See PubSub spec for more details.

The `key` field MAY be present and relates to signing. See PubSub spec for more details.

TODO: Don't quite understand this scenario, to clarify. Wouldn't it always be in `from`?
> The key field contains the signing key when it cannot be inlined in the source peer ID. When present, it must match the peer ID.

### SubOpts

To do topic subscription management, we MAY send updates to our peers. If we do so, then:

The `subscribe` field MUST contain a boolean, where 1 means subscribe and 0 means unsubscribe to a topic.

The `topicid` field MUST contain the topic.

NOTE: This doesn't appear to be documented in PubSub spec, upstream?

## Discovery domain

TODO: To document how we use Discovery v5, etc. See https://github.com/vacp2p/specs/issues/167

## Request/reply domain

This consists of two main protocols. They are used in order to get Waku to run
in resource restricted environments, such as low bandwidth or being mostly
offline.

### Historical message support

**Protocol identifier***: `/vac/waku/store/2.0.0-alpha2`

TODO To be elaborated on

#### Protobuf

```protobuf
message RPC {
  repeated HistoryQuery historyQuery = 4;
  repeated HistoryResponse historyResponse = 5;
}

message HistoryQuery {
  // TODO Include time range, topic/contentTopic, limit, cursor, (request-id), etc
}

message HistoryResponse {
  // TODO Include Messages, cursor, etc
}
```

##### HistoryQuery

RPC call to query historical messages.

TODO To be specified in more detail

##### HistoryResponse

RPC call to respond to a HistoryQuery call.

TODO To be specified in more detail

### Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-alpha2`

Content filter is a way to do [message-based
filtering](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering).
Currently the only content filter being applied is on `contentTopic`. This
corresponds to topics in Waku v1.

A node that only sets this field but doesn't subscribe to any topic SHOULD only
get notified when the content subtopic matches. A content subtopic matches when
a message `contentTopic` is the same. This means such a node acts as a light node.

A node that receives this RPC SHOULD apply this content filter before relaying.
Since such a node is doing extra work for a light node, it MAY also account for
usage and be selective in how much service it provides. This mechanism is
currently planned but underspecified.

#### Protobuf

```protobuf
message RPC {
  repeated ContentFilter contentFilter = 3;
}

  message ContentFilter {
    optional string contentTopic = 1;
  }
}
```



TODO(Oskar): Update changelog once we are in draft, which is when
implementation matches spec

Initial raw version. Released []()

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

8. [Message Filtering (Wikipedia)](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering)
