---
title: Waku Relay
version: 2.0.0-beta2
status: Draft
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

- [Abstract](#abstract)
  * [Wire Specification](#wire-specification)
  * [Protobuf](#protobuf)
  * [RPC](#rpc)
  * [Message](#message)
  * [SubOpts](#subopts)
  * [Signature Policy](#signature-policy)
- [Copyright](#copyright)
- [References](#references)

# Abstract

`WakuRelay` is part of the gossip domain for Waku. It is a thin layer on top of GossipSub.

**Protocol identifier***: `/vac/waku/relay/2.0.0-beta2`

## Wire Specification

We are using protobuf RPC messages between peers. Here's what the protobuf messages looks like, as defined in the PubSub interface. Please see [PubSub interface spec](https://github.com/libp2p/specs/blob/master/pubsub/README.md) for more details.

In this section we specify two things how WakuSub is using these messages.

## Protobuf

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
}
```

WakuSub does not currently use the `ControlMessage` defined in GossipSub.
However, later versions will add likely add this capability.

`TopicDescriptor` as defined in the PubSub interface spec is not currently used.

## RPC

These are messages sent to directly connected peers. They SHOULD NOT be
gossiped. See section below on how the fields work.

## Message

The `from` field MAY indicate which peer is publishing the message.

The `data` field MUST be filled out with a `WakuMessage`. See the [Waku Message](waku-message.md) spec for more details.

The `seqno` field MAY be used to provide a linearly increasing number. See PubSub spec for more details.

The `topicIDs` field MUST contain the topics that a message is being published on.

The `signature` field MAY contain the signature of the message, thus providing authentication of the message. See PubSub spec for more details.

The `key` field MAY be present and relates to signing. See PubSub spec for more details.


## SubOpts

To do topic subscription management, we MAY send updates to our peers. If we do so, then:

The `subscribe` field MUST contain a boolean, where 1 means subscribe and 0 means unsubscribe to a topic.

The `topicid` field MUST contain the topic.

## Signature Policy

The `StrictNoSign` option MUST be used. 

## Changelog

### 2.0.0-beta2

Next version. Changes:

- Moved WakuMessage to separate spec and made it mandatory
- StrictNoSign


### 2.0.0-beta1

Initial draft version. Released [2020-09-17](https://github.com/vacp2p/specs/commit/a57dad2cc3d62f9128e21f68719704a0b358768b)

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [PubSub interface for libp2p (r2,
   2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md)

2. [GossipSub
   v1.0](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)

3. [GossipSub
   v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md)

4. [Waku v1 spec](specs.vac.dev/waku/waku.html)

5. [Whisper spec (EIP627)](https://eips.ethereum.org/EIPS/eip-627)


<!--
TODO: Don't quite understand this scenario [key field], to clarify. Wouldn't it always be in `from`?
> The key field contains the signing key when it cannot be inlined in the source peer ID. When present, it must match the peer ID. -->

Re topicid:
NOTE: This doesn't appear to be documented in PubSub spec, upstream?
-->
