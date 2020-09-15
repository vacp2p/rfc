---
title: Waku
version: 2.0.0-alpha4
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Motivation and goals](#motivation-and-goals)
- [Network interaction domains](#network-interaction-domains)
    + [Protocol Identifiers](#protocol-identifiers)
  * [Gossip domain](#gossip-domain)
    + [Wire Specification](#wire-specification)
    + [Protobuf](#protobuf)
    + [RPC](#rpc)
    + [Message](#message)
    + [SubOpts](#subopts)
  * [Discovery domain](#discovery-domain)
  * [Request/reply domain](#request-reply-domain)
    + [Historical message support](#historical-message-support)
      - [Protobuf](#protobuf-1)
        * [HistoryQuery](#historyquery)
        * [HistoryResponse](#historyresponse)
    + [Content filtering](#content-filtering)
      - [Protobuf](#protobuf-2)
- [Upgradability and Compatibility](#upgradability-and-compatibility)
    * [Compatibility with Waku v1](#compatibility-with-waku-v1)
      - [Bridge](#bridge)
      - [Security Considerations](#security-considerations)
- [Copyright](#copyright)
- [References](#references)

# Abstract

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

# Motivation and goals

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

# Network interaction domains

While Waku is best though of as a single cohesive thing, there are three network
interaction domains: (a) gossip domain (b) discovery domain (c) req/resp domain.

### Protocol Identifiers

The current [protocol identifiers](https://docs.libp2p.io/concepts/protocols/) are:

1. `/vac/waku/relay/2.0.0-alpha2`
2. `/vac/waku/store/2.0.0-alpha5`
3. `/vac/waku/filter/2.0.0-alpha5`

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
}

message WakuMessage {
  optional bytes payload = 1;
  optional string contentTopic = 2;
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

The `data` field SHOULD be filled out with a `WakuMessage`.

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

### WakuMessage

A `WakuMessage` SHOULD be in the `data` field of `Message`.

The `payload` field SHOULD contain whatever payload is being sent. Encryption of this field is done at a separate layer.

The `contentTopic` field SHOULD be filled out to allow for content-based filtering (see section below).

## Discovery domain

TODO: To document how we use Discovery v5, etc. See https://github.com/vacp2p/specs/issues/167

## Request/reply domain

This consists of two main protocols. They are used in order to get Waku to run
in resource restricted environments, such as low bandwidth or being mostly
offline.

### Historical message support

**Protocol identifier***: `/vac/waku/store/2.0.0-alpha5`

TODO To be elaborated on

#### Protobuf

```protobuf
message HistoryQuery {
  string uuid = 1;
  repeated string topics = 2;
}

message HistoryResponse {
  string uuid = 1;
  repeated WakuMessage messages = 2;
}
```

##### HistoryQuery

RPC call to query historical messages.

The `uuid` field MUST indicate current request UUID, it is used to identify the corresponding response.

The `topics` field MUST indicate the list of topics to query.

##### HistoryResponse

RPC call to respond to a HistoryQuery call.

The `uuid` field MUST indicate which query is being responded to.

The `messages` field MUST contain the messages found.


### Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-alpha5`

Content filtering is a way to do [message-based
filtering](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering).
Currently the only content filter being applied is on `contentTopic`. This
corresponds to topics in Waku v1.

#### Rationale

Unlike the `store` protocol for historical messages, this protocol allows for
native lower latency scenarios such as instant messaging. It is thus
complementary to it.

Strictly speaking, it is not just doing basic request response, but performs
sender push based on receiver intent. While this can be seen as a form of light
pub/sub, it is only used between two nodes in a direct fashion. Unlike the
Gossip domain, this is meant for light nodes which put a premium on bandwidth.
No gossiping takes place.

It is worth noting that a light node could get by with only using the `store`
protocol to query for a recent time window, provided it is acceptable to do
frequent polling.

#### Protobuf

TODO Consider adding a FilterResponse acting as a form of ACK

TODO Consider adding request id

TODO Clarify if Messages or a list of WakuMessage are pushed

TODO Specify unsubscribe mechanism and semantics

TODO Investigate if we need a way to communicate (handshake?) that we are a a client - server (full node - light node) or not.
NOTE I would imagine this is implied from the contentFilter, especially as two nodes can play multiple roles.


```protobuf
message FilterRequest {
  // space for optional request id
  repeated ContentFilter contentFilters = 2;
  optional string topic = 3;

  message ContentFilter {
    optional string contentTopics = 1;
  }
}

message MessagePush {
  repeated WakuMessage messages = 1;
}
```

##### FilterRequest

TODO Specify mechanism for telling it won't honor (normal-no service-spam case)

TODO Clarify exactly what we mean by connection and TTL

A node that sends the RPC with a filter request requests that the filter node
SHOULD notify the light requesting node of messages matching this filter.

The filter matches when content filter and, optionally, a topic is matched.
Content filter is matched when a `WakuMessage` `contentTopic` field is the same.

A filter node SHOULD honor this request, though it MAY choose not to do so. If
it chooses not to do so it MAY tell the light why. The mechanism for doing this
is currently not specified. For notifying the light node a filter node sends a
MessagePush message.

Since such a filter node is doing extra work for a light node, it MAY also
account for usage and be selective in how much service it provides. This
mechanism is currently planned but underspecified.

##### MessagePush

A filter node that has received a filter request SHOULD push all messages that
match this filter to a light node. These messages are likely to come from the
`relay` protocol and be kept at the Node, but there MAY be other sources or
protocols where this comes from. This is up to the consumer of the protocol.

A filter node MUST NOT send a push message for messages that have not been
requested via a FilterRequest.

If a specific light node isn't connected to a filter node for some specific
period of time (e.g. a TTL), then the filter node MAY choose to not push these
messages to the node. This period is up to the consumer of the protocol and node
implementation, though a reasonable default is one minute.

---

TODO(Oskar): Update changelog once we are in draft, which is when
implementation matches spec

Initial raw version. Released []()

# Upgradability and Compatibility
## Compatibility with Waku v1

Waku v2 and Waku v1 are different protocols all together. They use a different
transport protocol underneath; Waku v1 is devp2p RLPx based while Waku v2 uses
libp2p. The protocols themselves also differ as does their data format.
Compatibility can be achieved only by using a bridge that not only talks both
devp2p RLPx and libp2p, but that also transfers (partially) the content of a
packet from one version to the other.

### Bridge

A bridge requires supporting both Waku versions:

* Waku v1 - using devp2p RLPx protocol
* Waku v2 - using libp2p protocols

Packets received on the Waku v1 network SHOULD be published just once on the
Waku v2 network. More specifically, the bridge SHOULD publish
this through the Waku Relay (PubSub domain).

Publishing such packet will require the creation of a new `Message` with a
new `WakuMessage` as data field. The `data` and `topic` field from the Waku v1
`Envelope` MUST be copied to the `payload` and `contentTopic` fields of the
`WakuMessage`. Other fields such as nonce, expiry and ttl will be dropped as
they become obsolete in Waku v2.

Before this is done, the usual envelope verification still applies:

* Expiry & future time verification
* PoW verification
* Size verification

Bridging SHOULD occur through the `WakuRelay`, but it MAY also be done on other Waku
v2 protocols (e.g. `WakuFilter`). The latter is however not advised as it will
increase the complexity of the bridge and because of the
[Security Considerations](#security-considerations) explained further below.

Packets received on the Waku v2 network SHOULD be posted just once on the Waku
v1 network. The Waku v2 `WakuMessage` contains only the `payload` and
`contentTopic` fields. The bridge MUST create a new Waku v1 `Envelope` and
copy over the `payload` and `contentFilter` fields to the `data` and `topic`
fields. Next, before posting on the network, the bridge MUST set a new expiry
and ttl and do the PoW nonce calculation.

### Security Considerations
As mentioned above, a bridge will be posting new Waku v1 envelopes, which
requires doing the PoW nonce calculation.

This could be a DoS attack vector, as the PoW calculation will make it more
expensive to post the message compared to the original publishing on the Waku v2
network. Low PoW setting will lower this problem, but it is likely that it is
still more expensive.

For this reason, bridges SHOULD probably be run independently of other nodes, so
that a bridge that gets overwhelmed does not disrupt regular Waku v2 to v2
traffic.

Bridging functionality SHOULD also be carefully implemented so that messages do
not bounce back and forth between the two networks. The bridge SHOULD properly
track messages with a seen filter so that no amplification can be achieved here.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

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

9. [Libp2p PubSub spec - topic validation](https://github.com/libp2p/specs/tree/master/pubsub#topic-validation)

<!--

# Underlying transports, etc

## Peer Discovery

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
-->
