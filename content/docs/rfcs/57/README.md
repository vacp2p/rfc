---
slug: 57
title: 57/STATUS-Simple-Scaling
name: Status Simple Scaling
status: raw
category: Informational
tags: waku/application
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

# Abstract

This document describes how to scale [56/STATUS-COMMUNITIES](/spec/56/) as well as [55/STATUS-1TO1-CHAT](/spec/55/)
using existing Waku v2 protocols and components.
It also adds a few new aspects, where more sophisticated components are not yet researched and evaluated.

> *Note:* (Parts of) this RFC will be deprecated in the future as we continue research to scale specific components
in a way that aligns better with our principles of decentralization and protecting anonymity.
This document informs about scaling at the current stage of research and shows it is practically possible.
Practical feasibility is also a core goal for us.
We believe in incremental improvement, i.e. having a working decentralized scaling solution with trade-offs is better than a fully centralized solution.

# Background and Motivation

[56/STATUS-COMMUNITIES](/spec/56/) as well as [55/STATUS-1TO1-CHAT](/spec/55/) use Waku v2 protocols.
Both use Waku content topics (see [23/WAKU2-TOPICS](/spec/23/)) for content based filtering.

Waku v2 currently has scaling limitations in two dimensions:

1) Messages that are part of a specific content topic have to be disseminated in a single mesh network (i.e. pubsub topic).
This limits scaling the number of messages disseminated in a specific content topic,
and by extension, the number of active nodes that are part of this content topic.

2) Scaling a large set of content topics requires distributing these over several mesh networks (which this document refers to as pubsub topic shards).

This document focuses on the second scaling dimension.
With the scaling solutions discussed in this document,
each content topics can have a large set of active users, but still has to fit in a single pubsub mesh.

> *Note:* While it is possible to use the same content topic name on several shards,
each node that is interested in this content topic has to be subscribed to all respective shards, which does not scale.
Splitting content topics in a more sophisticated and efficient way will be part of a future document.

# Relay Shards

Sharding the [Waku Relay](/spec/11/) network is an integral part of scaling the Status app.

[51/WAKU2-RELAY-SHARDING](/spec/51/) specifies shards clusters, which are sets of `1024` shards (separate pubsub mesh networks).
Content topics specified by application protocols can be distributed over these shards.
The Status app protocols are assigned to shard cluster `16`,
as defined in [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52/).

[51/WAKU2-RELAY-SHARDING](/spec/51/) specifies three sharding methods.
This document uses *static sharding*, which leaves the distribution of content topics to application protocols,
but takes care of shard discovery.

The 1024 shards within the main Status shard cluster are allocated as follows.

## Shard Allocation

| shard index   |    usage                         |
|    ---        |    ---                           |
|       0 - 15  |   reserved                       |
|     16 - 127  |   specific (large) communities   |
|    128 - 767  |   communities                    |
|    768 - 895  |   1:1 chat                       |
|   896 - 1023  |   media and control msgs         |

Shard indices are mapped to pubsub topic names as follows (specified in [51/WAKU2-RELAY-SHARDING](/spec/51/)).

`/waku/2/rs/<shard_cluster_index>/<shard_number>`

an example for the shard with index `18` in the Status shard cluster:

`/waku/2/rs/16/18`

In other words, the mesh network with the pubsub topic name `/waku/2/rs/16/18` carries messages associated with shard `18` in the Status shard cluster.


### Implementation Suggestion

The Waku implementation should offer an interface that allows Status nodes to subscribe to Status specific content topics like

```
subscribe("/status/xyz", 16, 18)
```

The shard cluster index `16` can be kept in the Status app configuration,
so that Status nodes can simply use

```
subscribe("/status/xyz", 18)
```

which means: connect to the `"status/xyz"` content topic on shard `18` within the Status shard cluster.

## Status Communities

In order to associate a community with a shard,
the community description protobuf is extended by the field
`uint16 relay_shard_index = 15`:

```protobuf

syntax = "proto3";

message CommunityDescription {
  // The Lamport timestamp of the message
  uint64 clock = 1;
  // A mapping of members in the community to their roles
  map<string,CommunityMember> members = 2;
  // The permissions of the Community
  CommunityPermissions permissions = 3;
  // The metadata of the Community
  ChatIdentity identity = 5;
  // A mapping of chats to their details
  map<string,CommunityChat> chats = 6;
  // A list of banned members
  repeated string ban_list = 7;
  // A mapping of categories to their details
  map<string,CommunityCategory> categories = 8;
  // The admin settings of the Community
  CommunityAdminSettings admin_settings = 10;
  // If the community is encrypted
  bool encrypted = 13;
  // The list of tags
  repeated string tags = 14;
  // index of the community's shard within the Status shard cluster
  uint16 relay_shard_index = 15
}
```

> *Note*: Currently, Status app has allocated shared cluster `16` in [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52/).
Status app could allocate more shard clusters, for instance to establish a test net.
We could add the shard cluster index to the community description as well.
The recommendation for now is to keep it as a configuration option of the Status app.

> *Note*: Once this RFC moves forward, the new community description protobuf fields should be mentioned in [56/STATUS-COMMUNITIES](https://rfc.vac.dev/spec/56/).

Status communities can be mapped to shards in two ways: static, and owner-based.

### Static Mapping

With static mapping, communities are assigned a specific shard index within the Status shard cluster.
This mapping is similar in nature to the shard cluster allocation in [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52/).
Shard indices allocated in that way are in the range `16 - 127`.
The Status CC community uses index `16` (not to confuse with shard cluster index `16`, which is the Status shard cluster).

### Owner Mapping

> *Note*: This way of mapping will be specified post-MVP.

Community owners can choose to map their communities to any shard within the index range `128 - 767`.

## 1:1 Chat

[55/STATUS-1TO1-CHAT](/spec/55) uses partitioned topics to map 1:1 chats to a set of 5000 content topics.
This document extends this mapping to 8192 content topics that are, in turn, mapped to 128 shards in the index range of `768 - 895`.

```
contentPartitionsNum = 8192
contentPartition = mod(publicKey, contentPartitionsNum)
partitionContentTopic = "contact-discovery-" + contentPartition

partitionContentTopic = keccak256(partitionContentTopic)

shardNum = 128
shardIndex = 768 + mod(publicKey, shardNum)

```

# Infrastructure Nodes

As described in [30/ADAPTIVE-NODES](/spec/30/),
Waku supports a continuum of node types with respect to available resources.
Infrastructure nodes are powerful nodes that have a high bandwidth connection and a high up-time.

This document, which informs about simple ways of scaling Status over Waku,
assumes the presence of a set of such infrastructure nodes in each shard.
Infrastructure nodes are especially important for providing connectivity in the roll-out phase.

Infrastructure nodes are not limited to Status fleets, or nodes run by community owners.
Anybody can run infrastructure nodes.

## Statically-Mapped Communities

Infrastructure nodes are provided by the community owner, or by members of the respective community.

## Owner-Mapped Communities

Infrastructure nodes are part of a subset of the shards in the range `128 - 767`.
Recommendations on choosing this subset will be added in a future version of this document.

Status fleet nodes make up a part of these infrastructure nodes.

## 1:1 chat

Infrastructure nodes are part of a subset of the shards in the range `768 - 985` (similar to owner-mapped communities).
Recommendations on choosing this subset will be added in a future version of this document.

Desktop clients can choose to only use filter and lightpush.

> *Note*: Discussion: I'd suggest to set this as the default for the MVP.
The load on infrastructure nodes would not be higher, because they have to receive and relay each message anyways.
This comes as a trade-off to anonymity and decentralization,
but can significantly improve scaling.
We still have k-anonymity because several chat pairs are mapped into one content topic.
We could improve on this in the future, and research the applicability of PIR (private information retrieval) techniques in the future.


# Infrastructure Shards

Waku messages are typically relayed in larger mesh networks comprised of nodes with varying resource profiles (see [30/ADAPTIVE-NODES](/spec/30/)).
To maximise scaling, relaying of specific message types can be dedicated to shards where only infrastructure nodes with very strong resource profiles relay messages.
This comes as a trade-off to decentralization.

## Control Message Shards

To get the maximum scaling for select large communities for the Status scaling MVP,
specific control messages that cause significant load (at a high user number) SHOULD be moved to a separate control message shard.
These control messages comprise:

* community description
* membership update 
* backup
* community request to join response
* sync profile picture

The relay functionality of control messages shards SHOULD be provided by infrastructure nodes.
Desktop clients should use light protocols as the default for control message shards.
Strong Desktop clients MAY opt in to support the relay network.

Each large community (in the index range of `16 - 127`) can get its dedicated control message shard (in the index range `896 - 1023`) if deemed necessary.
The Status CC community uses shard `896` as its control message shard.
This comes with trade-offs to decentralization and anonymity (see *Security Considerations* section).

## Media Shards

Similar to control messages, media-heavy communities should use separate media shards (in the index range `896 - 1023`) for disseminating messages with large media data.
The Status CC community uses shard `897` as its media shard.

## Infrastructure-focused Community

Large communities MAY choose to mainly rely on infrastructure nodes for *all* message transfers (not limited to control, and media messages).
Desktop clients of such communities should use light protocols as the default.
Strong Desktop clients MAY opt in to support the relay network.

> *Note*: This is not planned for the MVP.


# Light Protocols

Light protocols may be used to save bandwidth,
at the (global) cost of not contributing to the network.
Using light protocols is RECOMMENDED for resource restricted nodes,
e.g. browsers,
and devices that (temporarily) have a low bandwidth connection or a connection with usage-based billing.

Light protocols comprise

* [19/WAKU2-LIGHTPUSH](/spec/19/) for sending messages
* [12/WAKU2-FILTER](/spec/12/) for requesting messages with specific attributes
* [34/WAKU2-PEER-EXCHANGE](/spec/34) for discovering peers


# Waku Archive

Archive nodes are Waku nodes that offer the Waku archive service via the Waku store protocol ([13/WAKU2-STORE](/spec/13/)).
They are part of a set of shards and store all messages disseminated in these shards.
Nodes can request history messages via the [13/WAKU2-STORE](/spec/13/).

The store service is not limited to a Status fleet.
Anybody can run a Waku Archive node in the Status shards.

> *Note*: There is no specification for discovering archive nodes associated with specific shards yet.
Nodes expect archive nodes to store all messages, regardless of shard association.

The recommendation for the allocation of archive nodes to shards is similar to the
allocation of infrastructure nodes to shards described above.
In fact, the archive service can be offered by infrastructure nodes.

# Discovery

Shard discovery is covered by [51/WAKU2-RELAY-SHARDING](/spec/51/).
This allows the Status app to abstract from the discovery process and simply address shards by their index.

## Libp2p Rendezvous

This document suggests using
[libp2p rendezvous](https://github.com/libp2p/specs/blob/master/rendezvous/README.md)
in addition to [33/WAKU2-DISCV5](/spec/33/) discovery which is offered by the Waku layer as specified in [51/WAKU2-RELAY-SHARDING](/spec/51/).
The main reason is making nodes behind restrictive NAT discoverable.
In addition, it helps increasing discoverability, and by extension connectivity, for nodes that are not behind restricted NAT.

Nodes that do not take part in [33/WAKU2-DISCV5](/spec/33/) discovery,
e.g., resource restricted devices, MAY use rendezvous discovery instead of or along-side [34/WAKU2-PEER-EXCHANGE](/specs/34).
For these nodes, rendezvous and [34/WAKU2-PEER-EXCHANGE](/specs/34) offer the same functionality,
but return node sets sampled in different ways.
Using both can help increasing connectivity.

It is RECOMMENDED that nodes that are part of the relay network, also act as rendezvous points.
This includes accepting register queries from peers, as well as answering rendezvous discover queries.
Nodes MAY opt-out of the rendezvous functionality.

To support the main purpose of rendezvous (for Waku),
it is RECOMMENDED for nodes that act as a rendezvous point to also offer to act as a relay between the node that queried, and the discovered node.
This allows nodes behind restrictive NATs to be part of the relay network.

> *Note*: A specification of this process will follow in a future version of this document.

While this process is similar to [libp2p circuit relay](https://docs.libp2p.io/concepts/nat/circuit-relay/),
it is not the same.
Circuit relay offers an encrypted end-to-end channel, which means the circuit relay node does *not* disseminate message via Waku relay,
and the circuit-relay traffic is additional load.
The approach recommended in this document does not introduce additional traffic (apart from a few control messages),
because the relay node acts as a typical Waku relay.


### Announcing Shard Participation

Registering a namespace via [lib-p2p rendezvous](https://github.com/libp2p/specs/blob/master/rendezvous/README.md#interaction)
is done via a register query:

```
REGISTER{my-app, {QmA, AddrA}}
```

The app name, `my-app` is used to encode a single shard in the form:

```
<rs (utf8 encoded)> | <2-byte shard cluster index> | <2-byte shard index>
```

Registering shard 2 in the Status shard cluster (with shard cluster index 16, see [52/WAKU2-RELAY-STATIC-SHARD-ALLOC](/spec/52/h)),
the register query would look like

```
REGISTER{0x727300100002, {QmA, AddrA}}
```

Participation in further shards is registered with further queries; one register query per shard.
(0x7273 is the encoding of `rs`.)

A discovery query for nodes that are part of this shard would look like

```
DISCOVER{ns: 0x727300100002}
```


# DoS Protection

> *Note* :  DoS protection will be specified in a soon-to-follow update of this RFC (while in raw state).
The following sketches basic approaches.
DoS protection might move in a separate document and be referenced here.

## Statically-Mapped Communities

Basic idea:
Each [Waku message](/specs/14) is signed with key material provided by the community owner.
Relay nodes only relay messages that have the correct signature.
Community infrastructure nodes are provided with the necessary key material, too.

## Owner-Mapped Communities

Basic idea:
Tokenized load.

## 1:1 Chat

An idea we plan to explore in the future:
Map 1:1 chats to community shards, if both A and B are part of the respective community.
This increases k-anonymity and benefits from community DoS protection.
It could be rate-limited with RLN.


# Security/Privacy Considerations

This document makes several trade-offs to privacy and anonymity.
Todo: elaborate.
See [45/WAKU2-ADVERSARIAL-MODELS](/spec/45) for information on Waku Anonymity.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

* [11/WAKU2-RELAY](/spec/11/)
* [51/WAKU2-RELAY-SHARDING](/spec/51/)
* [33/WAKU2-DISCV5](/spec/33/)
* [13/WAKU2-STORE](/spec/13/)
* [19/WAKU2-LIGHTPUSH](/spec/19/)
* [12/WAKU2-FILTER](/spec/12/)
* [34/WAKU2-PEER-EXCHANGE](/spec/34/)
* [33/WAKU2-DISCV5](/spec/33/)
* [Circuit Relay](https://docs.libp2p.io/concepts/nat/circuit-relay/)
* [31/WAKU2-ENR](/spec/31/)
* [45/WAKU2-ADVERSARIAL-MODELS](/spec/45)


