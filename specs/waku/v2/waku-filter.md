---
title: Waku
version: 2.0.0-beta1
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Content filtering](#content-filtering)
  * [Rationale](#rationale)
  * [Protobuf](#protobuf)
- [Changelog](#changelog)
- [Copyright](#copyright)
- [References](#references)

# Abstract

`WakuFilter` is a protocol that enables receiving of messages from a peer whenever it receives one that matches the specified filter.

# Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-beta1`

Content filtering is a way to do [message-based
filtering](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering).
Currently the only content filter being applied is on `contentTopic`. This
corresponds to topics in Waku v1.

## Rationale

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

## Protobuf

<!--
TODO Consider adding a FilterResponse acting as a form of ACK

TODO Specify unsubscribe mechanism and semantics

TODO Investigate if we need a way to communicate (handshake?) that we are a a client - server (full node - light node) or not.
NOTE I would imagine this is implied from the contentFilter, especially as two nodes can play multiple roles.
-->


```protobuf
message FilterRequest {
  optional string topic = 1;
  repeated ContentFilter contentFilters = 2;

  message ContentFilter {
    optional string contentTopics = 1;
  }
}

message MessagePush {
  repeated WakuMessage messages = 1;
}

message FilterRPC {
  string request_id = 1;
  FilterRequest request = 2;
  MessagePush push = 3;
}
```

#### FilterRPC

A node MUST send all Filter messages (`FilterRequest`, `MessagePush`) wrapped inside a
`FilterRPC` this allows the node handler to determine how to handle a message as the Waku
Filter protocol is not a request response based protocol but instead a push based system.

The `request_id` MUST be a uniquely generated string. When a `MessagePush` is sent
it the `request_id` MUST match the `request_id` of the `FilterRequest` whose filters
matched the message causing it to be pushed.

#### FilterRequest

<!--

TODO Specify mechanism for telling it won't honor (normal-no service-spam case)

TODO Clarify exactly what we mean by connection and TTL

-->

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

#### MessagePush

A filter node that has received a filter request SHOULD push all messages that
match this filter to a light node. These [`WakuMessage`'s](./waku-message.md) are likely to come from the
`relay` protocol and be kept at the Node, but there MAY be other sources or
protocols where this comes from. This is up to the consumer of the protocol.

A filter node MUST NOT send a push message for messages that have not been
requested via a FilterRequest.

If a specific light node isn't connected to a filter node for some specific
period of time (e.g. a TTL), then the filter node MAY choose to not push these
messages to the node. This period is up to the consumer of the protocol and node
implementation, though a reasonable default is one minute.

---

# Changelog

2.0.0-beta1
Initial draft version. Released 2020-10-05 <!-- @TODO LINK -->

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [Message Filtering (Wikipedia)](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering)

2. [Libp2p PubSub spec - topic validation](https://github.com/libp2p/specs/tree/master/pubsub#topic-validation)
