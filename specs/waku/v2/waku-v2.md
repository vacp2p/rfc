---
title: Waku
version: 2.0.0-beta1
status: Draft
authors: Oskar Thorén <oskar@status.im>, Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Motivation and goals](#motivation-and-goals)
- [Network interaction domains](#network-interaction-domains)
  - [Protocols and identifiers](#protocols-and-identifiers)
  - [Gossip domain](#gossip-domain)
    - [Default pubsub topic](#default-pubsub-topic)
  - [Discovery domain](#discovery-domain)
  - [Request/reply domain](#requestreply-domain)
    - [Historical message support](#historical-message-support)
    - [Content filtering](#content-filtering)
- [Upgradability and Compatibility](#upgradability-and-compatibility)
  - [Compatibility with Waku v1](#compatibility-with-waku-v1)
- [Security](#security)
  - [Adversarial Model](#adversarial-model)
  - [Security Features](#security-features)
  - [Security considerations](#security-considerations)
  - [Future work](#future-work)
  - [Changelog](#changelog)
    - [Next version](#next-version)
    - [2.0.0-beta1](#200-beta1)
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

## Protocols and identifiers

The current [protocol identifiers](https://docs.libp2p.io/concepts/protocols/) are:

1. `/vac/waku/relay/2.0.0-beta2`
2. `/vac/waku/store/2.0.0-beta1`
3. `/vac/waku/filter/2.0.0-beta1`

These protocols and their semantics are elaborated on in their own specs.

For the actual content being passed around, see the [Waku Message](waku-message.md) spec.

The WakuMessage spec being used is: `2.0.0-beta1`.

## Gossip domain

**Protocol identifier***: `/vac/waku/relay/2.0.0-beta1`

See [WakuRelay](waku-relay.md) spec for more details.

### Default pubsub topic

The default PubSub topic being used for Waku is currently: `/waku/2/default-waku/proto`

This indicates that it relates to Waku, is version 2, is the default topic, and
that the encoding of data field is protobuf.

The default PubSub topic SHOULD be used for all protocols. This ensures a
connected network, as well some degree of metadata protection. It MAY be
different if or when:
<!-- TODO discuss anonymity related to one pubsub topic -->
- Different applications have different message volume
- Topic sharding is introduced
- Encoding is changed
- Version is changed

## Discovery domain

Discovery domain is not yet implemented. Currently static nodes should be used.

<!-- TODO: To document how we use Discovery v5, etc. See https://github.com/vacp2p/specs/issues/167 -->

## Request/reply domain

This consists of two main protocols. They are used in order to get Waku to run
in resource restricted environments, such as low bandwidth or being mostly
offline.

### Historical message support

**Protocol identifier***: `/vac/waku/store/2.0.0-beta1`

See [WakuStore](waku-store.md) spec for more details.

### Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-beta1`

See [WakuFilter](waku-filter.md) spec for more details.


# Upgradability and Compatibility

## Compatibility with Waku v1

Waku v2 and Waku v1 are different protocols all together. They use a different
transport protocol underneath; Waku v1 is devp2p RLPx based while Waku v2 uses
libp2p. The protocols themselves also differ as does their data format.
Compatibility can be achieved only by using a bridge that not only talks both
devp2p RLPx and libp2p, but that also transfers (partially) the content of a
packet from one version to the other.

See [bridge spec](waku-bridge.md) for details on a bridge mode.


# Security 
Each protocol layer of Waku v2 provides a distinct service and is associated with a separate set of security features and concerns. Therefore, the overall security of Waku v2 depends on how different layers are utilized. In this section, we overview the security properties of Waku protocols against a static adversarial model which is described below.

## Adversarial Model
The adversary is considered as a passive entity that attempts to collect information from others to conduct an attack but it does so without violating protocol definitions and instructions.

The following are **not** considered as part of the adversarial model: 
  -  An adversary with a global view of all the peers and their connections. 
  -  An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). In specific, the communication channels are assumed to be secure.

## Security Features

In the followings, we overview the security features supported by Waku v2: 

**Pseudonimity**: Waku v2 by default guarantees pseudonymity for all of the protocol layers since parties do not have to disclose their true identity and instead they utilize libp2p `PeerID` as their identifiers. While pseudonymity is an appealing security feature, it does not guarantee full anonymity since the actions taken under the same pseudonym i.e., `PeerID` can be linked together and result in the re-identification of the true actor. 
  
**Anonymity / Unlinkability**:  At a high level, we can see Anonymity as the inability of an adversary in linking an actor to its performed action/data (the actor and action are context-dependent). To be precise about anonymity we use the term Personally Identifiable Information (PII) to refer to any piece of data that could potentially be used to uniquely identify a party/person. For example, the signature verification key, and the hash of one's static IP address are unique for each user and hence count as PII. Note that while signature keys provide pseudonymity (i.e., they contain no information about the true identity of the actor), but as discussed before, one's actions can be linked through its signatures and cause the re-identification risk, hence, we seek anonymity by avoiding linkability between actions and the actors PIIs as well.

**Publisher-Message Unlinkability** and **Subscriber-Topic Unlinkability**: These levels of unlinkability capture the anonymity requirements of the `WakuRelay` corresponding to the two main actions that can take place in this protocol i.e., topic subscription and message publishing. The former signifies the unlinkability of a publisher to its published message whereas the latter stands for the unlinkability of the subscriber to its subscribed topics. The Publisher-Message Unlinkability is enforced through the `StrictNoSign` policy whereas the Subscriber-Topic Unlinkability is achieved through the utilization of a single pubsub topic. Note that there is no hard limit on the number of the pubsub topics, however, the use of one topic is recommended for the sake of anonymity. See [WakuRelay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md#security-analysis) for detailed security analysis. 

**Spam protection**: This property indicates that no adversary can flood the system with spam messages (i.e., publishing a large number of messages in a short amount of time). Spam protection is partly provided in `WakuRelay` through [scoring mechansim](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#spam-protection-measures) of GossipSub v1.1. At a high level, peers utilize a scoring function to locally score the behavior of their connections and remove peers with a low score.   
  
**Data confidentiality, Integrity, Authenticity**:  These features are enabled through payload encryption and encrypted signatures at the `WakuMessage` version 1 (see [WakuMessage specs](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-message.md#version-1-not-yet-implemented-in-waku-v2) for further details). 

## Security considerations
**Lack of anonymity in the direct connections including `WakuStore` and `WakuFilter` protocols**: The anonymity is not guaranteed in the protocols like `WakuStore` and `WakuFilter` where peers need to have direct connections to benefit from the designated service. This is because during the direct connections peers utilize `PeerID` to identify each other, which means the service (actions) taken in the protocol are linked to parties `PeerID` (which counts as PII).  In terms of `WakuStore`, this means that the querying nodes have to reveal their topics of interest to the queried nodes hence compromising their privacy.  Likewise, in the `WakuFilter`, light nodes have to disclose their liking topics to the full nodes to retrieve the relevant messages. 
  
## Future work

We are actively working on the following features to be added to Waku v2.

**Economic Spam resistant**: We aim to enable an incentivized spam protection technique at the `WakuRelay` by using rate limiting nullifiers. Mode details on this can be found in  [Waku RLN Relay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-rln-relay.md). In this new version, peers are limited to a certain rate of messaging per epoch and an immediate financial penalty is enforced for spammers who break this rate.

**Prevention of Denial of Service (DoS)**: Denial of service signifies the case where an adversarial node exhausts a node in the `store` protocol by making a large number of queries (even redundant queries) thus making the node unavailable to the rest of the system. DoS attack is to be mitigated through accounting model as provided by [Waku Swap Accounting specs](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-swap-accounting.md). In a nutshell, peers have to pay for the service they obtain from each other, which means, in terms of `store` protocol, the querying node will be charged for the history messages that it queries from other nodes. In addition to incentivizing the service provider, accounting also makes DoS attacks costly for malicious peers. By the accounting model, we aim at protecting service availability in `WakuStore` and `WakuFilter`.




## Changelog

### Next version

- Add recommended default PubSub topic
- Update relay and message spec version

### 2.0.0-beta1

Initial draft version. Released [2020-09-17](https://github.com/vacp2p/specs/commit/a57dad2cc3d62f9128e21f68719704a0b358768b)

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
