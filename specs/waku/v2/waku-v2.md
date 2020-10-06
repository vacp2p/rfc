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
    + [Protocol Identifiers](#protocol-identifiers)
  * [Gossip domain](#gossip-domain)
  * [Discovery domain](#discovery-domain)
  * [Request/reply domain](#request-reply-domain)
    + [Historical message support](#historical-message-support)
    + [Content filtering](#content-filtering)
- [Upgradability and Compatibility](#upgradability-and-compatibility)
    * [Compatibility with Waku v1](#compatibility-with-waku-v1)
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

1. `/vac/waku/relay/2.0.0-beta1`
2. `/vac/waku/store/2.0.0-alpha5`
3. `/vac/waku/filter/2.0.0-alpha5`

These protocols and their semantics are elaborated on in their own specs.

For the actual content being passed around, see the [Waku
Message](waku-message.md) spec.

## Gossip domain

**Protocol identifier***: `/vac/waku/relay/2.0.0-beta1`

See [WakuRelay](waku-relay.md) spec for more details.

## Discovery domain

Discovery domain is not yet implemented. Currently static nodes should be used.

<!-- TODO: To document how we use Discovery v5, etc. See https://github.com/vacp2p/specs/issues/167 -->

## Request/reply domain

This consists of two main protocols. They are used in order to get Waku to run
in resource restricted environments, such as low bandwidth or being mostly
offline.

### Historical message support (experimental, alpha)

**Protocol identifier***: `/vac/waku/store/2.0.0-alpha5`

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

## Changelog

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
