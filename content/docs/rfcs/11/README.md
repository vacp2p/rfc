---
slug: 11
title: 11/WAKU2-RELAY
name: Waku v2 Relay
status: draft
editor: Hanno Cornelius <hanno@status.im>
contributors:
  - Oskar Thorén <oskar@status.im>
  - Sanaz Taheri <sanaz@status.im>
---

<!-- A note to contributors:
Please make use of the following best practices when adding to this document:
- semantic line breaks as per https://sembr.org/
- RFC key words as per RFC 2119 (https://www.ietf.org/rfc/rfc2119.txt) -->

`WakuRelay` specifies a [Publish/Subscribe approach](https://docs.libp2p.io/concepts/publish-subscribe/) to peer-to-peer messaging with a strong focus on privacy, censorship-resistance, security and scalability.
Its current implementation is a minor extension of the [libp2p GossipSub protocol](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md) and prescribes gossip-based dissemination.
As such the scope is limited to defining a separate [`protocol id`](https://github.com/libp2p/specs/blob/master/connections/README.md#protocol-negotiationn) for `WakuRelay`, establishing privacy and security requirements, and defining how the underlying GossipSub is to be interpreted and implemented within the Waku and cryptoeconomic domain.

**Protocol identifier**: `/vac/waku/relay/2.0.0-beta2`

# Security Requirements

The `WakuRelay` protocol is designed to provide the following security properties under a static [Adversarial Model](#adversarial-model).
Note that data confidentiality, integrity, and authenticity are currently considered out of scope for the `WakuRelay` protocol and must be handled by higher layer protocols such as [WakuMessage](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-message.md).

<!-- May add the definition of the unsupported feature: 
Confidentiality indicates that an adversary should not be able to learn the data carried by the `WakuRelay` protocol.
Integrity indicates that the data transferred by the `WakuRelay` protocol can not be tampered with by an adversarial entity without being detected.
Authenticity no adversary can forge data on behalf of a targeted publisher and make it accepted by other subscribers as if the origin is the target. -->


- **Publisher-Message Unlinkability**:
This property indicates that no adversarial entity can link a published `Message` to its publisher.
This feature also implies the unlinkability of the publisher to its published topic ID as the `Message` embodies the topic IDs.

- **Subscriber-Topic Unlinkability**:
This feature stands for the inability of any adversarial entity from linking a subscriber to its subscribed topic IDs.

  
<!-- TODO: more requirements can be added, but that needs further and deeper investigation-->


## Terminology

_Personally identifiable information_ (PII) refers to any piece of data that can be used to uniquely identify a user.
For example, the signature verification key, and the hash of one's static IP address are unique for each user and hence count as PII.

# Adversarial Model

-  Any entity running the `WakuRelay` protocol is considered an adversary.
This includes publishers, subscribers, and all the peers' direct connections. 
Furthermore, we consider the adversary as a passive entity that attempts to collect information from others to conduct an attack but it does so without violating protocol definitions and instructions.
For example, under the passive adversarial model, no malicious subscriber hides the messages it receives from other subscribers as it is against the description of the `WakuRelay` protocol.
However, a malicious subscriber may learn which topics are subscribed to by which peers. 
- The following are **not** considered as part of the adversarial model: 
  -  An adversary with a global view of all the peers and their connections. 
  -  An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication).
  In other words, the communication channels are assumed to be secure.


# Wire Specification

The [PubSub interface specification](https://github.com/libp2p/specs/blob/master/pubsub/README.md) defines the protobuf RPC messages exchanged between peers participating in a GossipSub network.
We republish these messages here for ease of reference and define how `WakuRelay` uses and interprets each field.

## Protobuf definitions

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
}
```

> **_NOTE:_**
The various [control messages](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#control-messages) defined for GossipSub are used as specified there.

> **_NOTE:_**
The [`TopicDescriptor`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) is not currently used by `WakuRelay`.

## Message fields

The `Message` protobuf defines the format in which content is relayed between peers.
`WakuRelay` specifies the following usage requirements for each field:

- The `from` field MUST NOT be used, following the [`StrictNoSign` signature policy](#Signature-Policy).

- The `data` field MUST be filled out with a `WakuMessage`.
See the [Waku Message](waku-message.md) spec for more details.

- The `seqno` field MUST NOT be used, following the [`StrictNoSign` signature policy](#Signature-Policy).

- The `topicIDs` field MUST contain the topics that a message is being published on.

- The `signature` field MUST NOT be used, following the [`StrictNoSign` signature policy](#Signature-Policy).

- The `key` field MUST NOT be used, following the [`StrictNoSign` signature policy](#Signature-Policy).

## SubOpts fields

The `SubOpts` protobuf defines the format in which subscription options are relayed between peers.
A `WakuRelay` node MAY decide to subscribe or unsubscribe from topics by sending updates using `SubOpts`.
The following usage requirements apply:

- The `subscribe` field MUST contain a boolean, where `true` indicates subscribe and `false` indicates unsubscribe to a topic.

- The `topicid` field MUST contain the topic.

## Signature Policy

The [`StrictNoSign` option](https://github.com/libp2p/specs/blob/master/pubsub/README.md#signature-policy-options) MUST be used, to ensure that messages are built without the `signature`, `key`, `from` and `seqno` fields. Note that this does not merely imply that these fields be empty, but that they MUST be _absent_ from the marshalled message.

# Security Analysis

<!-- TODO: realized that the prime security objective of the `WakuRelay` protocol is to provide peers unlinkability as such this feature is prioritized over other features e.g., unlinkability is preferred over authenticity and integrity. It might be good to motivate unlinkability and its impact on the relay protocol or other protocols invoking relay protocol.-->

- **Publisher-Message Unlinkability**:
To address publisher-message unlinkability, one should remove any PII from the published message.
As such, `WakuRelay` protocol follows the `StrictNoSign` policy as described in [libp2p PubSub specs](https://github.com/libp2p/specs/tree/master/pubsub#message-signing).
As the result of the `StrictNoSign` policy, `Message`s should be built without the  `from`, `signature` and `key` fields since each of these three fields individually counts as PII for the author of the message (one can link the creation of the message with libp2p peerId and thus indirectly with the IP address of the publisher). 
Note that removing identifiable information from messages cannot lead to perfect unlinkability.
The direct connections of a publisher might be able to figure out which `Message`s belong to that publisher by analyzing its traffic.
The possibility of such inference may get higher when the `data` field is also not encrypted by the upper-level protocols. <!-- TODO: more investigation on traffic analysis attacks and their success probability-->

- **Subscriber-Topic Unlinkability:**
To preserve subscriber-topic unlinkability, it is recommended by [Waku v2 specs](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-v2.md#default-pubsub-topic) to use a single PubSub topic in the `WakuRelay` protocol.
This allows an immediate subscriber-topic unlinkability where subscribers are not re-identifiable from their subscribed topic IDs as the entire network is linked to the same topic ID. This level of unlinkability / anonymity is known as [k-anonymity](https://www.privitar.com/blog/k-anonymity-an-introduction/) where k is proportional to the system size (number of subscribers).
However, note that `WakuRelay` supports the use of more than one topic.
In that case, the unlinkability should be addressed by the upper-level protocols through e.g., [partitioned topics technique](https://specs.status.im/spec/10#partitioned-topic) which enables K-anonymity for the peers' subscribed topic ids.

# Future work

- **Economic spam resistance**:
In the spam-protected `WakuRelay` protocol, no adversary can flood the system with spam messages (i.e., publishing a large number of messages in a short amount of time).
Spam protection is partly provided by GossipSub v1.1 through [scoring mechanism](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#spam-protection-measures).
At a high level, peers utilize a scoring function to locally score the behavior of their connections and remove peers with a low score.
The `WakuRelay` protocol aims at enabling an advanced spam protection mechanism with economic disincentives by utilizing Rate Limiting Nullifiers.
In a nutshell, peers must conform to a certain message publishing rate per a system-defined epoch, otherwise, they get financially penalized for exceeding the rate.
More details on this new technique can be found in [Waku RLN Relay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-rln-relay.md). 
  <!-- TODO havn't checked if all the measures in libp2p GossipSub v1.1 are taken in the nim-libp2p as well, may need to audit the code --> 


- Providing **Unlinkability**, **Integrity** and  **Authenticity** simultaneously:
Integrity and authenticity are typically addressed through digital signatures and Message Authentication Code (MAC) schemes, however, the usage of digital signatures (where each signature is bound to a particular peer) contradicts with the unlinkability requirement (messages signed under a certain signature key are verifiable by a verification key that is bound to a particular publisher).
As such, integrity and authenticity are missing features in the `WakuRelay` protocol in the interest of unlinkability.
In future work, advanced signature schemes like group signatures can be utilized to enable authenticity, integrity, and unlinkability simultaneously.
In a group signature scheme, a member of a group can anonymously sign a message on behalf of the group as such the true signer is indistinguishable from other group members. <!-- TODO: shall I add a reference for group signatures?-->
  

# Changelog

### Next

- Added initial threat model and security analysis 
- General rework and clarifications

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

4. [Waku v1 spec](https://specs.vac.dev/waku/waku.html)

5. [Whisper spec (EIP627)](https://eips.ethereum.org/EIPS/eip-627)
