---
slug: 51
title: 51/WAKU2-RELAY-SHARDING
name: Waku v2 Relay Sharding
status: raw
category: Standards Track
tags:
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
- Simon-Pierre Vivier <simvivier@status.im>
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
(control messages go beyond these boundaries, but at its core, it is a separate P2P network).
With this, the number of [Waku relay](/spec/11/) content topics that can be carried over a pubsub topic is limited.
This prevents app protocols that aim to span many multicast groups (realized by content topics) from scaling.

This document specifies three pubsub topic sharding methods (with varying degrees of automation),
which allow application protocols to scale in the number of content topics.
This document also covers discovery of topic shards.

# Named Sharding

*Named sharding* offers apps to freely choose pubsub topic names.
It is RECOMMENDED for App protocols to follow the naming structure detailed in [23/WAKU2-TOPICS](/spec/23/).
With named sharding, managing discovery falls into the responsibility of apps.

From an app protocol point of view, a subscription to a content topic `waku2/xxx` on a shard named /mesh/v1.1.1/xxx would look like:

`subscribe("/waku2/xxx", "/mesh/v1.1.1/xxx")`

# Static Sharding

*Static sharding* offers a set of shards with fixed names.
Assigning content topics to specific shards is up to app protocols,
but the discovery of these shards is managed by Waku.

Static shards are managed in shard clusters of 1024 shards per cluster.
Waku static sharding can manage $2^16$ shard clusters.
Each shard cluster is identified by its index (between $0$ and $2^16-1$).

A specific shard cluster is either globally available to all apps,
specific for an app protocol,
or reserved for automatic sharding (see next section).

> *Note:* This leads to $2^16 * 1024 = 2^26$ shards for which Waku manages discovery.

App protocols can either choose to use global shards, or app specific shards.

Like the [IANA ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml),
shard clusters are divided into ranges:

| index (range) |    usage                 |
|    ---        |    ---                   |
|             0 |   global                 |
|        1 - 15 |   reserved               |
|     16 - 1023 |   specific app protocols |
|  1024 - 49152 |   all app protocols      |
| 49152 - 65535 |   automatic sharding     |

The informational RFC [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52) lists the current index allocations.

The global shard with index 0 and the "all app protocols" range are treated in the same way,
but choosing shards in the global cluster has a higher probability of sharing the shard with other apps.
This offers k-anonymity and better connectivity, but comes at a higher bandwidth cost.

The name of the pubsub topic corresponding to a given static shard is specified as

`/waku/2/rs/<shard_cluster_index>/<shard_number>`,

an example for the 2nd shard in the global shard cluster:

`/waku/2/rs/0/2`.

> *Note*: Because *all* shards distribute payload defined in [14/WAKU2-MESSAGE](spec/14/) via [protocol buffers](https://developers.google.com/protocol-buffers/),
the pubsub topic name does not explicitly add `/proto` to indicate protocol buffer encoding.
We use `rs` to indicate these are *relay shard* clusters; further shard types might follow in the future.

From an app point of view, a subscription to a content topic `waku2/xxx` on a static shard would look like:

`subscribe("/waku2/xxx", 43)`

for global shard 43.
And for shard 43 of the Status app (which has allocated index 16):

`subscribe("/waku2/xxx", 16, 43)`

## Discovery

Waku v2 supports the discovery of peers within static shards,
so app protocols do not have to implement their own discovery method.

Nodes add information about their shard participation in their [31/WAKU2-ENR](https://rfc.vac.dev/spec/31/).
Having a static shard participation indication as part of the ENR allows nodes
to discover peers that are part of shards via [33/WAKU2-DISCV5](/spec/33/) as well as via DNS.

> *Note:* In the current version of this document,
sharding information is directly added to the ENR.
(see Ethereum ENR sharding bit vector [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/p2p-interface.md#metadata)
Static relay sharding supports 1024 shards per cluster, leading to a flag field of 128 bytes.
This already takes half (including index and key) of the ENR space of 300 bytes.
For this reason, the current specification only supports a single shard cluster per node.
In future versions, we will add further (hierarchical) discovery methods.
We will update [31/WAKU2-ENR](https://rfc.vac.dev/spec/31/) accordingly, once this RFC moves forward.

This document specifies two ways of indicating shard cluster participation.
The index list SHOULD be used for nodes that participante in fewer than 64 shards,
the bit vector representation SHOULD be used for nodes participating in 64 or more shards.
Nodes MUST NOT use both index list (`rs`) and bit vector (`rsv`) in a single ENR.
ENRs with both `rs` and `rsv` keys SHOULD be ignored.
Nodes MAY interpret `rs` in such ENRs, but MUST ignore `rsv`.

### Index List

| key    | value   |
|---     |---      |
| `rs`   | <2-byte shard cluster index> &#124; <1-byte length> &#124;  <2-byte shard index>  &#124; ... &#124; <2-byte shard index>  |

The ENR key is `rs`.
The value is comprised of
* a two-byte shard cluster index in network byte order, concatenated with
* a one-byte length field holding the number of shards in the given shard cluster, concatenated with
* two-byte shard indices in network byte order

Example:

| key    | value   |
|---     |---      |
| `rs`   | 16u16 &#124; 3u8 &#124; 13u16 &#124; 14u16 &#124; 45u16 |

This example node is part of shards `13`, `14`, and `45` in the Status main-net shard cluster (index 16).

### Bit Vector

| key    | value   |
|---     |---      |
| `rsv`   | <2-byte shard cluster index> &#124; <128-byte flag field>  |

The ENR key is `rsv`.
The value is comprised of a two-byte shard cluster index in network byte order concatenated with a 128-byte wide bit vector.
The bit vector indicates which shards of the respective shard cluster the node is part of.
The right-most bit in the bit vector represents shard `0`, the left-most bit represents shard `1023`.
The representation in the ENR is inspired by [Ethereum shard ENRs](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)),
and [this](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/validator.md#sync-committee-subnet-stability)).

Example:

| key    | value   |
|---     |---      |
| `rsv`   | 16u16 &#124; `0x[...]0000100000003000` |

The `[...]` in the example indicates 120 `0` bytes.
This example node is part of shards `13`, `14`, and `45` in the Status main-net shard cluster (index 16).
(This is just for illustration purposes, a node that is only part of three shards should use the index list method specified above.)

# Automatic Sharding

Autosharding selects shards automatically and is the default behavior for shard choice.
The other choices being static and named sharding as seen in previous sections.
Shards (pubsub topics) SHOULD be computed from content topics with the procedure below.

## Rendezvous Hashing

Also known as the Highest Random Weight (H.R.W.) method, has many properties useful for shard selection.
- Local only computation without communication.
- Content topics are spread as equally as possible to shards.
- Content topics switch shards infrequently.
- Easy to compute.

### Algorithm

For each shard,
hash using Sha2-256 the concatenation of
the content topic `application` field (UTF-8 string of N bytes),
`version` (UTF-8 string of N bytes),
the `cluster` index (2 bytes) and
the `shard` index (2 bytes)
take the first 64 bits of the hash,
divided by 2^64-1,
compute the natural logarithm of this number then
take the negative of the weight (default 1.0) divided by it.
Finally, sort the shard value pairs.

The shard with the highest value SHOULD be used.

### Example
| Field | Value | Hex |
|---    |---    |---  |
| `application`| "myapp"| 0x6d79617070|
| `version`    | "1"    | 0x31
| `cluster`    | 1      | 0x0001
| `shard`      | 6      | 0x0006

- SHA2-256 of `0x6d796170703100010006` is `0xfdac5fb315b791cca3ccde738ebb0b8d2742519fce1086f2a3d2409cb22a7dd2`
- The first 64 bits `0xfdac5fb315b791cc` divided by `0xffffffffffffffff` is ~`0.99`
- The natural logarithm of `0.99` is ~`-0.01`
- `-1` divided by `-0.01` equals ~`100`
- Shard 6 has the priority value 100

With the same method, the other priority values would be; 0.57, -2.18, 1.51, 1.14, 0.65, 0.72, 0.54
Content topic `/myapp/1/myname/cbor` should use shard 6 because it has the highest value.

## Content Topics Format for Autosharding
Content topics MUST follow the format in [23/WAKU2-TOPICS](https://rfc.vac.dev/spec/23/#content-topic-format).
In addition, a generation prefix MAY be added to content topics.
When omitted default values are used.
Generation default value is `0`.

- The full length format is `/{generation}/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`
- The short length format is `/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`

### Example

- Full length `/0/myapp/1/mytopic/cbor`
- Short length `/myapp/1/mytopic/cbor`

### Generation
The generation number monotonously increases and indirectly refers to the total number of shards of the Waku Network.

<!-- Create a new RFC for each generation spec. -->

### Topic Design
Content topics have 2 purposes: filtering and routing.
Filtering is done by changing the `{content-topic-name}` field.
As this part is not hashed, it will not affect routing (shard selection).
The `{application-name}` and `{version-of-the-application}` fields do affect routing.
Using multiple content topics with different `{application-name}` field has advantages and disadvantages.
It increases the traffic a relay node is subjected to when subscribed to all topics.
It also allows relay and light nodes to subscribe to a subset of all topics.

## Problems

### Hot Spots

Hot spots occur (similar to DHTs), when a specific mesh network (shard) becomes responsible for (several) large multicast groups (content topics).
The opposite problem occurs when a mesh only carries multicast groups with very few participants: this might cause bad connectivity within the mesh.

The current autosharding method does not solve this problem.

> *Note:* Automatic sharding based on network traffic measurements to avoid hot spots in not part of this specification.

### Discovery

For the discovery of automatic shards this document specifies two methods (the second method will be detailed in a future version of this document).

The first method uses the discovery introduced above in the context of static shards.

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
depends on the number of content topics (`k`), as a proxy for the number of peers and messages, that get mapped onto a single pubsub topic (shard).
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
* [Rendezvous Hashing](https://www.eecs.umich.edu/techreports/cse/96/CSE-TR-316-96.pdf)
* [Research log: Waku Discovery](https://vac.dev/wakuv2-apd)
* [45/WAKU2-ADVERSARIAL-MODELS](/spec/45)

