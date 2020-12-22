---
title: Waku
version: 2.0.0-beta1
status: Draft
authors: Oskar Thorén <oskar@status.im>
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
    - [Lack of anonymity in the direct connections including `WakuStore` and `WakuFilter` protocols](#lack-of-anonymity-in-the-direct-connections-including-wakustore-and-wakufilter-protocols)
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

The security features and considerations in Waku v2 depend on how its different protocol layers are combined to utilize its various capabilities. The security is analyzed against a static adversarial model which is described next.

## Adversarial Model
The adversary is considered as a passive entity that attempts to collect information from others to conduct an attack but it does so without violating protocol definitions and instructions. This has a different implication for each protocol which is discussed in the protocol specs. 
The following are **not** considered as part of the adversarial model: 
  -  An adversary with a global view of all the peers and their connections. 
  -  An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). In specific, the communication channels are assumed to be secure.

## Security Features
Waku v2 follows a modular architecture and is comprised of multiple protocol layers each featuring a distinct capability. The security features of the Waku also depends on how these layers are utilized and combined. 
Considering the three basic security aspects, i.e., privacy, integrity, and anonymity, Waku v2 capture the following properties:

- **Anonymity**: The anonymity requirement appears in multiple protocol layers and each layer is associated with its measures. At a high level, Anonymity in a system indicates the inability of an adversary in linking an actor to its performed action (the actor and action are context-dependent). 
  - `WakuRelay`:  Anonymity at this layer is classified into two categories:  **Publisher-Message Unlinkability** and **Subscriber-Topic Unlinkability** which captures anonymity requirement for the two main actions that can take place in the `WakuRelay` i.e., topic subscription and message publishing. The former signifies the unlinkability of a publisher to its published message whereas the latter stands for the unlinkability of the subscriber to its subscribed topics. The Publisher-Message Unlinkability is enforced through the `StrictNoSign` policy whereas the Subscriber-Topic Unlinkability is enabled through the utilization of a single pubsub topic (note that there is no hard limit on the number of the pubsub topics, however, the use of one topic is recommended as it empowers anonymity). See [WakuRelay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md#security-analysis) for detailed security analysis. <!-- TODO the PerrID utilization in peers direct connections may violate anonymity -->
## Security considerations
### Lack of anonymity in the direct connections including `WakuStore` and `WakuFilter` protocols
- Anonymity may be violated in the direct connections where the two ends become aware of each other's IP address or libp2p `PeerID`. While the IP address can be masked through a proxy server, preventing the disclosure of `PeerID` is not straightforward.
  

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
