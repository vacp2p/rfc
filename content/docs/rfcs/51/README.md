---
slug: 51
title: 51/WAKU2-RELAY-SHARDING
name: Waku v2 Relay Sharding
status: raw
category: Standards Track
tags:
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

# Abstract

This document describes ways of sharding the [Waku relay](/spec/11/) topic,
allowing Waku networks to scale in the number of content topics.

> *Note*: Scaling in the size of a single content topic is out of scope for this document.

# Background and Motivation

[Unstructured P2P networks](https://en.wikipedia.org/wiki/Peer-to-peer#Unstructured_networks)
are more robust and resilient against DoS attacks compared to
[structured P2P networks](https://en.wikipedia.org/wiki/Peer-to-peer#Structured_networks)).
However, they do not scale to large traffic loads.
A single [libp2p gossipsub mesh](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#gossipsub-the-gossiping-mesh-router),
which carries messages associated with a single pubsub topic, can be seen as a separate unstructured P2P network
(gossip and control messages go beyond these boundaries, but at its core, it is a separate P2P network).
With this, the number of [Waku relay](/spec/11/) content topics that can be carried over a pubsub topic is limited.
This prevents app protocols that aim to span many multicast groups (realized by content topics) from scaling.

This document specifies three pubsub topic sharding methods (with varying degrees of automation),
which allow application protocols to scale in the number of content topics.
This document also covers discovery of topic shards.

# Named Sharding

*Named sharding* offers apps to freely choose pubsub topic names.
App protocols SHOULD follow the naming structure detailed in [23/WAKU2-TOPICS](/spec/23/).
With named sharding, managing discovery falls into the responsibility of apps.

The default Waku pubsub topic `/waku/2/default-waku/proto` can be seen as a named shard available to all app protocols.

> *Note*: Future versions of this document are planned to give more guidance with respect to discovery via
[33/WAKU2-DISCV5](/spec/33/),
[DNS discovery](https://eips.ethereum.org/EIPS/eip-1459),
and inter-mesh discovery via gossipsub control messages (also using circuit relay).
It might make sense to deprecate [23/WAKU2-TOPICS](/spec/23/) as a separate spec and merge it here.

From an app protocol point of view, a subscription to a content topic `waku2/xxx` on a shard named /mesh/v1.1.1/xxx would look like:

`subscribe("/waku2/xxx", "/mesh/v1.1.1/xxx")`

# Static Sharding

*Static sharding* offers an array of shard clusters.
A shard cluster is a set of 64 shards, which is either globally available to all apps (like the default pubsub topic),
specific for an app protocol, or reserved for automatic sharding (see next section).

App protocols can either choose to use global shards, or app specific shards.
(In future versions of this document, automatic sharding, described in the next section, will become the default.)

Like the [IANA ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml),
shard clusters are divided into ranges:

| index (range) |    usage                 |
|    ---        |    ---                   |
|             0 |   global                 |
|        1 - 15 |   reserved               |
|     16 - 1023 |   specific app protocols |
|  1024 - 49125 |   all app protocols      |
| 49152 - 65535 |   automatic sharding     |

The informational RFC [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52) lists the current index allocations.

The global shard with index 0 and the "all app protocols" range are treated in the same way,
but choosing shards in the global cluster has a higher probability of sharing the shard with other apps.
This offers k-anonymity and better connectivity, but comes at a higher bandwidth cost.

From an app point of view, a subscription to a content topic `waku2/xxx` on a static shard would look like:

`subscribe("/waku2/xxx", 43)`

for global shard 43.
And for shard 43 of the Status app (which has allocated index 16):

`subscribe("/waku2/xxx", 16, 43)`

## Discovery

Waku v2 supports the discovery of static shards,
so app protocols do not have to implement their own discovery method.
To enable discovery of static shards,
the array of shard clusters is added to [31/WAKU2-ENR](https://rfc.vac.dev/spec/31/).
The representation is specified as follows.

The array index is a 2 bytes field.
As the array is expected to be sparse (and because ENRs do not feature an array/map type),
the ENR contains a list of occupied array slots.
Each shard cluster is represented by a bit vector,
which indicates which shards of the respective shard cluster the node is part of
(see Ethereum ENR sharding bit vector [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/p2p-interface.md#metadata)
and [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)).

> *Note*: We will update the [31/WAKU2-ENR](https://rfc.vac.dev/spec/31/) accordingly, once this RFC moves forward.)

Having a static shard participation indication as part of the ENR allows nodes
to discover peers that are part of shards via [33/WAKU2-DISCV5](/spec/33/) as well as via DNS.

In its current raw version, this document proposes two representations in the ENR.
(Which one to choose is open for discussion in the raw phase of the document.
Future versions will only specify a single representation.)

### One key per Shard Cluster

For each shard cluster a node is part of, the node adds a separate key to its ENR.
The representation corresponds to [Ethereum shard ENRs](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)).

Example

| key         | value                |
|---          |---                   |
| `rshard-0`  | `0x0000100000000000` |
| `rshard-16` | `0x0000100000003000` |

This example node is part of 1 shard in the global shard cluster,
and part of 3 shards in the Status main-net shard cluster.

This methods is easier to read.
It is feasible, assuming nodes are only part of a few apps using specific shard clusters.

### Single Key

Example

| key         | value   |
|---          |---      |
| `rshards`   | `num_shards` &#124;  0u16 &#124;  `0x0000100000000000` &#124;  16u16 &#124; `0x0000100000003000` |

Using network byte order.

# Automatic Sharding

> *Note:* Topic sharding is not yet part of this specification.
This section merely serves as an outlook.
A specification of topic shading will be added to this document in a future version.

Topic sharding is a method for scaling Waku relay in the number of (smaller) content topics).
It automatically maps Waku content topics to pubsub topics.
Clients and protocols building on Waku relay only see content topics, while Waku relay internally manages the mapping.
This provides both scaling as well as removes confusion about content and pubsub topics on the consumer side.

From an app point of view, a subscription to a content topic `waku2/xxx` using automatic sharding would look like:

`subscribe("/waku2/xxx", auto=true)`

The app is oblivious to the pubsub topic layer.
(Future versions could deprecate the default pubsub topic and remove the necessity for `auto=true`.)

*The basic idea behind topic sharding*:
Content topics are mapped using [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing).
Like with DHTs, the hash space is split into parts,
each covered by a Pubsub topic (mesh network) that carries content topics which are mapped into the respective part of the hash space.

There are (at least) two issues that have to be solved: *Hot spots* and *Discovery* (see next subsection).

Hot spots occur (similar to DHTs), when a specific mesh network becomes responsible for (several) large multicast groups (content topics).
The opposite problem occurs when a mesh only carries multicast groups with very few participants: this might cause bad connectivity within the mesh.
Our research goal here is finding efficient ways of distribution.
We could get inspired by the DHT literature.
We also have to consider:
If a node is part of many content topics which are all spread over different shards,
the node will potentially be exposed to a lot of network traffic.

## Discovery

For the discovery of automatic shards this document specifies two methods (the second method will be detailed in a future version of this document).

The first method uses the discovery introduced above in the context of static shards.
The index range `49152 - 65535` is reserved for automatic sharding.
Each index can be seen as a hash bucket.
Consistent hashing maps content topics in one of these buckets.

The second discovery method will be a successor to the first method,
but is planned to preserve the index range allocation.
Instead of adding the data to the ENR, it will treat each array index as a capability,
which can be hierarchical, having each shard in the indexed shard cluster as a sub-capability.
When scaling to a very large number of shards, this will avoid blowing up the ENR size, and allows efficient discovery.
We currently use [33/WAKU2-DISCV5](https://rfc.vac.dev/spec/33/) for discovery,
which is based on Ethereum's [discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md).
While this allows to sample nodes from a distributed set of nodes efficiently and offers good resilience,
it does not allow to efficiently discover nodes with specific capabilities within this node set.
Our [research log post](https://vac.dev/wakuv2-apd) explains this in more detail.
Adding efficient (but still preserving resilience) capability discovery to discv5 is ongoing research.
[A paper on this](https://github.com/harnen/service-discovery-paper) has been completed,
but the [Ethereum discv5 specification](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md)
has yet to be updated.
When the new capability discovery is available,
this document will be updated with a specification of the second discovery method.
The transition to the second method will be seamless and fully backwards compatible because nodes can still advertise and discover shard memberships in ENRs.

# Security/Privacy Considerations

See [45/WAKU2-ADVERSARIAL-MODELS](/spec/45), especially the parts on k-anonymity.
We will add more on security considerations in future versions of this document.

## Receiver Anonymity

The strength of receiver anonymity, i.e. topic receiver unlinkablity,
depends on the number of content topics (`k`) that get mapped onto a single pubsub topic (shard).
For *named* and *static* sharding this responsibility is at the app protocol layer.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

* [11/WAKU2-RELAY](/spec/11/)
* [Unstructured P2P network](https://en.wikipedia.org/wiki/Peer-to-peer#Unstructured_networks)
* [33/WAKU2-DISCV5](/spec/33/)
* [31/WAKU2-ENR](https://rfc.vac.dev/spec/31/)
* [23/WAKU2-TOPICS](/spec/23/)
* [51/WAKU2-RELAY-SHARDING](/spec/51/)
* [Ethereum ENR sharding bit vector](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/p2p-interface.md#metadata)
* [Ethereum discv5 specification](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md)
* [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing).
* [Research log: Waku Discovery](https://vac.dev/wakuv2-apd)
* [45/WAKU2-ADVERSARIAL-MODELS](/spec/45)

