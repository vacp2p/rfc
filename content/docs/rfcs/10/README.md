---
slug: 10
title: 10/WAKU2
title: Waku v2
status: draft
editor: Oskar Thorén <oskar@status.im>
contributors:
  - Sanaz Taheri <sanaz@status.im>
---

Waku is a privacy-preserving peer-to-peer messaging protocol for resource
restricted devices. It implements PubSub over libp2p and adds capabilities on
top of it. These capabilities are: (i) retrieving historical messages for
mostly-offline devices (ii) adaptive nodes, allowing for heterogeneous nodes to
contribute, and (iii) bandwidth preservation for light nodes. This makes it
ideal for running a p2p protocol on mobile.

Historically, it has its roots in [Waku v1](/spec/6), which
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

For the actual content being passed around, see the [7/WAKU-DATA](/spec/7) spec.

The WakuMessage spec being used is: `2.0.0-beta1`.

## Gossip domain

**Protocol identifier***: `/vac/waku/relay/2.0.0-beta1`

See [11/WAKU-RELAY](/spec/11) spec for more details.

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

See [13/WAKU-STORE](/spec/13) spec for more details.

### Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-beta1`

See [14/WAKU-FILTER](/spec/14) spec for more details.


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
Each protocol layer of Waku v2 provides a distinct service and is associated with a separate set of security features and concerns. Therefore, the overall security of Waku v2 depends on how the different layers are utilized. In this section, we overview the security properties of Waku v2 protocols against a static adversarial model which is described below. Note that a more detailed security analysis of each Waku protocol is supplied in its respective specification as well.

## Primary Adversarial Model
In the primary adversarial model, we consider adversary as a passive entity that attempts to collect information from others to conduct an attack but it does so without violating protocol definitions and instructions.

The following are **not** considered as part of the adversarial model: 
  -  An adversary with a global view of all the peers and their connections. 
  -  An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). In specific, the communication channels are assumed to be secure.

## Security Features

### Pseudonymity 
Waku v2 by default guarantees pseudonymity for all of the protocol layers since parties do not have to disclose their true identity and instead they utilize libp2p `PeerID` as their identifiers. While pseudonymity is an appealing security feature, it does not guarantee full anonymity since the actions taken under the same pseudonym i.e., `PeerID` can be linked together and potentially result in the re-identification of the true actor. 
  
### Anonymity / Unlinkability
At a high level, anonymity is the inability of an adversary in linking an actor to its data/performed action (the actor and action are context-dependent). To be precise about linkability, we use the term Personally Identifiable Information (PII) to refer to any piece of data that could potentially be used to uniquely identify a party. For example, the signature verification key, and the hash of one's static IP address are unique for each user and hence count as PII. Notice that users' actions can be traced through their PIIs (e.g., signatures) and hence result in their re-identification risk. As such, we seek anonymity by avoiding linkability between actions and the actors / actors' PII. Concerning anonymity, Waku v2 provides the following features:

**Publisher-Message Unlinkability**: This feature signifies the unlinkability of a publisher to its published messages in the `WakuRelay` protocol. The [Publisher-Message Unlinkability](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md#security-analysis) is enforced through the `StrictNoSign` policy due to which the data fields of pubsub messages that count as PII for the publisher must be left unspecified.

**Subscriber-Topic Unlinkability**: This feature stands for the unlinkability of the subscriber to its subscribed topics in the `WakuRelay` protocol. The [Subscriber-Topic Unlinkability](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md#security-analysis) is achieved through the utilization of a single pubsub topic. As such, subscribers are not re-identifiable from their subscribed topic IDs as the entire network is linked to the same topic ID. This level of unlinkability / anonymity is known as [k-anonymity](https://www.privitar.com/blog/k-anonymity-an-introduction/) where k is proportional to the system size (number of subscribers). Note that there is no hard limit on the number of the pubsub topics, however, the use of one topic is recommended for the sake of anonymity. 

### Spam protection
This property indicates that no adversary can flood the system (i.e., publishing a large number of messages in a short amount of time), either accidentally or deliberately, with any kind of message i.e. even if the message content is valid or useful. Spam protection is partly provided in `WakuRelay` through [scoring mechanism](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#spam-protection-measures) of GossipSub v1.1. At a high level, peers utilize a scoring function to locally score the behavior of their connections and remove peers with a low score.    
  
### Data confidentiality, Integrity, Authenticity
Confidentiality can be addressed through data encryption whereas integrity and authenticity are achievable through digital signatures. These features are enabled in `WakuMessage` v1 through payload encryption as well as encrypted signatures (see [WakuMessage specs](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-message.md#version-1-not-yet-implemented-in-waku-v2) for further details). 

## Security Considerations

**Lack of anonymity/unlinkability in the protocols involving direct connections including `WakuStore` and `WakuFilter` protocols**: The anonymity/unlinkability is not guaranteed in the protocols like `WakuStore` and `WakuFilter` where peers need to have direct connections to benefit from the designated service. This is because during the direct connections peers utilize `PeerID` to identify each other, therefore the service obtained in the protocol is linkable to the beneficiary's `PeerID` (which counts as PII).  In terms of `WakuStore`, the queried node would be able to link the querying node's `PeerID` to its queried topics. Likewise, in the `WakuFilter`, a full node can link the light node's `PeerID`s to its content filter. <!-- TODO: to inspect the nimlibp2p codebase and figure out the exact use of PeerIDs in direct communication, it might be the case that the requester does not have to disclose its PeerID-->

<!--TODO: might be good to add a figure visualizing the Waku protocol stack and the security features of each layer-->

## Future work

We are actively working on the following features to be added to Waku v2.

**Economic Spam resistance**: We aim to enable an incentivized spam protection technique at the `WakuRelay` by using rate limiting nullifiers. More details on this can be found in [Waku RLN Relay](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-rln-relay.md). In this advanced method, peers are limited to a certain rate of messaging per epoch and an immediate financial penalty is enforced for spammers who break this rate.

**Prevention of Denial of Service (DoS)**: Denial of service signifies the case where an adversarial node exhausts another node's service capacity (e.g., by making a large number of requests) and makes it unavailable to the rest of the system. DoS attack is to be mitigated through the accounting model as described in [Waku Swap Accounting specs](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-swap-accounting.md). In a nutshell, peers have to pay for the service they obtain from each other. In addition to incentivizing the service provider, accounting also makes DoS attacks costly for malicious peers. The accounting model can be used in `WakuStore` and `WakuFilter` to protect against DoS attacks.

# Changelog

### Next version

- Add recommended default PubSub topic
- Update relay and message spec version
- Security analysis under the primary adversarial model is done

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

5. [Waku v1 spec](/spec/6)

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
