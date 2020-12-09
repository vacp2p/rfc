---
title: Waku Relay
version: 2.0.0-beta2
status: Draft
authors: Oskar Thorén <oskar@status.im>, Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Abstract](#abstract)
  - [Security Requirements](#security-requirements)
  - [Adversarial Model](#adversarial-model)
  - [Wire Specification](#wire-specification)
  - [Protobuf](#protobuf)
  - [RPC](#rpc)
  - [Message](#message)
  - [SubOpts](#subopts)
  - [Security Analysis](#security-analysis)
  - [Future work](#future-work)
  - [Changelog](#changelog)
    - [2.0.0-beta2](#200-beta2)
    - [2.0.0-beta1](#200-beta1)
- [Copyright](#copyright)
- [References](#references)

# Abstract

`WakuRelay` is part of the gossip domain for Waku. It is a thin layer on top of GossipSub.

**Protocol identifier***: `/vac/waku/relay/2.0.0-beta2`

## Security Requirements

<!-- In this part, we analyze the security of the  `relay` protocol concerning data confidentiality, integrity, authenticity, and anonymity. This is to enable users of this protocol to make an informed decision about all the security properties that can or can not achieve by deploying a `relay` protocol. -->
We will use the term Personally identifiable information (PII) to refer to any piece of data that can be used to uniquely identify a Peer. For example, the signature verification key, and the hash of one's IP address are unique for each peer and hence count as PII.

- **Message Publisher Anonymity**: This property indicates that no adversarial entity is able to link a published `Message` to its origin i.e., the peer. Note that this feature also implies the unlinkability of the publisher to its published topic ID, this is because  `Message` contains the `topicIDs` as well.

- **Topic Subscriber Anonymity:** This feature stands for the inability of any adversarial entity from linking a peer to its subscribed `topicIDs`.

- **Confidentiality**: The adversary should not be able to learn the data carried by the `relay` protocol
-  **Integrity**: This feature indicates that the data transferred by the `relay` protocol is not tampered with by an adversarial entity without being detected
-  **Authenticity**: No adversary can forge data on behalf of a victim peer and make it accepted by other peers as if the origin is the victim
- **Spam resistant**: No adversarial node in the `relay` protocol is able to flood the system with the spam messages
- 
<!-- TODO: more requirements can be added, but that needs further and deeper investigation-->

## Adversarial Model
- Any peer or collection of peers talking the `relay` protocol can act adversely to compromise any of the security features. As such, the adversary is a participant in the `relay` protocol. <!-- TODO: May later add the Honest but Curious adversary/static adversary assumption. That is, an adversarial entity may attempt to collect information from other peers (i.e., being curious) in order to succeed in its attack but it does so without violating protocol definitions and instructions (is honest), namely, it does follow the protocol specifications.--> 
- The following are not considered as part of the adversarial model: 1- a powerful adversary with a global view of all the peers and their connections 2- an adversary that can eavesdrop on communication links between arbitrary `relay` enabled nodes. 


## Wire Specification

We are using protobuf RPC messages between peers. Here's what the protobuf messages look like, as defined in the PubSub interface. Please see [PubSub interface spec](https://github.com/libp2p/specs/blob/master/pubsub/README.md) for more details.

In this section, we specify two things how WakuSub is using these messages.

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


## Security Analysis 
The prime security objective of the `relay` protocol is to provide peers anonymity as such this feature is prioritized over other features in case either case was doable.
- **Message Publisher Anonymity**: To preserve message anonymity, `relay` protocol follows the `StrictNoSign` policy as described in [libp2p PubSub specs](https://github.com/libp2p/specs/tree/master/pubsub#message-signing) due to which messages should be built without the  `from`, `signature` and `key` fields since each of these three fields individually counts as PII for the author of the message. One can link the creation of the PubSub message with libp2p peerId and thus indirectly with the IP address of the publisher.


## Future work

- **Confidentiality**: The current version of the `relay` protocol does not address confidentiality however it is achievable by performing encryption on top of the `data` field of `Message`s. More details on enabling encryption mode are provided in [Libp2p PubSub specs](https://github.com/libp2p/specs/tree/master/pubsub#the-topic-descriptor) as part of the `TopicDescriptor` messages.
  
- **Integrity** and  **Authenticity**: Integrity is typically addressed through digital signatures or MAC schemes, however, the usage of digital signatures (where signatures are bound to particular peers) contradicts with the anonymity requirements (messages signed under a certain signature key are verifiable by the corresponding verification key that is bound to a particular peer).  As such, integrity and authenticity are missing features in the `relay` protocol in the favor of anonymity. To fill this gap, we propose the integration of advanced signature schemes like group signatures to enable authenticity, integrity, and anonymity simultaneously. A group signature scheme is a method for allowing a member of a group to anonymously sign a message on behalf of the group. 

- **Spam resistant**: This feature is not yet integrated into the `relay` protocol, however, a PoC is in progress regarding the utilization of Rate Limiting Nullifiers to protect against spamming and spammers. More details can be found in [Waku RLN Relay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-rln-relay.md). 
  <!-- TODO: Peer scoring and PoW-->
  
- **Topic Subscriber Anonymity:** Concealing the link between a subscriber and its subscribed topic ids can not be addressed at the `relay` protocol since the exposure of this information is key to maintain a topic mesh. In specific,  subscribers can only participate in a topic mesh by getting connected to other subscribers of the same topic id hence disclosing their interest in that topic id at least to a subset of other subscribers (more details on this can be found in [libp2p pubsub documentation](https://docs.libp2p.io/concepts/publish-subscribe/)). As such, this information inevitably gets exposed to the topic mesh. However, upper level protocols can implement [Partitioned topics](https://specs.status.im/spec/10#partitioned-topic) to provide K-anonymity for peers's subscribed topic ids.
## Changelog

### 2.0.0-beta2

Next version. Changes:

- Moved WakuMessage to a separate spec and made it mandatory


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
