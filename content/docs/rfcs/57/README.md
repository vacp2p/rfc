---
slug: 57
title: 57/STATUS-Simple-Scaling
name: Status Simple Scaling
status: raw
category: Informational
tags: waku/application
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
- Alvaro Revuelta <alrevuelta@status.im>
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

## Libp2p Rendezvous and Circuit-Relay

To make nodes behind restrictive NATs discoverable,
this document suggests using [libp2p rendezvous](https://github.com/libp2p/specs/blob/master/rendezvous/README.md).
Nodes can check whether they are behind a restrictive NAT using the [libp2p AutoNAT protocol](https://github.com/libp2p/specs/blob/master/autonat/README.md).

> *Note:* The following will move into [51/WAKU2-RELAY-SHARDING](/spec/51/), or [33/WAKU2-DISCV5](/spec/33/):
Nodes behind restrictive NATs SHOULD not announce their publicly unreachable address via [33/WAKU2-DISCV5](/spec/33/) discovery.

It is RECOMMENDED that nodes that are part of the relay network also act as rendezvous points.
This includes accepting register queries from peers, as well as answering rendezvous discover queries.
Nodes MAY opt-out of the rendezvous functionality.

To allow nodes to initiate connections to peers behind restrictive NATs (after discovery via rendezvous),
it is RECOMMENDED that nodes that are part of the Waku relay network also offer
[libp2p circuit relay](https://github.com/libp2p/specs/blob/6634ca7abb2f955645243d48d1cd2fd02a8e8880/relay/circuit-v2.md) functionality.

To minimize the load on circuit-relay nodes, nodes SHOULD

1) make use of the [limiting](https://github.com/libp2p/specs/blob/6634ca7abb2f955645243d48d1cd2fd02a8e8880/relay/circuit-v2.md#reservation)
functionality offered by the libp2p circuit relay protocols, and
2) use [DCUtR](https://github.com/libp2p/specs/blob/master/relay/DCUtR.md) to upgrade to a direct connection.

Nodes that do not announce themselves at all and only plan to use light protocols,
MAY use rendezvous discovery instead of or along-side [34/WAKU2-PEER-EXCHANGE](/specs/34).
For these nodes, rendezvous and [34/WAKU2-PEER-EXCHANGE](/specs/34) offer the same functionality,
but return node sets sampled in different ways.
Using both can help increasing connectivity.

Nodes that are not behind restrictive NATs MAY register at rendezvous points, too;
this helps increasing discoverability, and by extension connectivity.
Such nodes SHOULD, however, not register at circuit relays.

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

Hereunder we describe the "opt-in message signing for DoS prevention" solution, designed *ad hoc* for Status MVP.

Since publishing messages to pubsub topics has no limits, anyone can publish messages at a very high rate and DoS the network.
This would elevate the bandwidth consumption of all nodes subscribed to said pubsub topic, making it prohibitive (in terms of bandwidth) to be subscribed to it.
In order to scale, we need some mechanism to prevent this from happening, otherwise all scaling efforts will be in vain.
Since RLN is not ready yet, hereunder we describe a simpler approach designed *ad hoc* for Status use case, feasible to implement for the MVP and that validates some of the ideas that will evolve to solutions such as RLN.

With this approach, certain pubsub topics can be optionally configured to only accept messages signed with a given key, that only trusted entities know.
This key can be pre-shared among a set of participants, that are trusted to make fair usage of the network, publishing messages at a reasonable rate/size.
Note that this key can be shared/reused among multiple participants, and only one key is whitelisted per pubsub topic.
This is an opt-in solution that operators can choose to deploy in their shards (i.e. pubsub topics), but it's not enforced in the default one.
Operators can freely choose how they want to generate, and distribute the public keys. It's also their responsibility to handle the private key, sharing it with only trusted parties and keeping proper custody of it.

The following concepts are introduced:
* `private-key-topic`: A private key of 32 bytes, that allows the holder to sign messages and it's mapped to a `protected-pubsub-topic`.
* `app-message-hash`: Application `WakuMessage` hash, calculated as `sha256(concat(pubsubTopic, payload, contentTopic))` with all elements in bytes.
* `message-signature`: ECDSA signature of `application-message-hash` using a given `private-key-topic`, 64 bytes.
* `public-key-topic`: The equivalent public key of `private-key-topic`.
* `protected-pubsub-topic`: Pubsub topic that only accepts messages that were signed with `private-key-topic`, where `verify(message-signature, app-message-hash, public-key-topic)` is only correct if the `message-signature` was produced by `private-key-topic`. See ECDSA signature verification algorithm.

This solution introduces two roles:
* Publisher: A node that knows the `private-key-topic` associated to `public-key-topic`, that can publish messages with a valid `message-signature` that are accepted and relayed by the nodes implementing this feature.
* Relayer: A node that knows the `public-key-topic`, which can be used to verify if the messages were signed with the equivalent `private-key-topic`. It allows distinguishing valid from invalid messages which protect the node against DoS attacks, assuming that the users of the key send messages of a reasonable size and rate. Note that a node can validate messages and relay them or not without knowing the private key.

## Design requirements (publisher)

A publisher that wants to send messages that are relayed in the network for a given `protected-pubsub-topic` shall:
* be able to sign messages with the `private-key-topic` configured for that topic, producing a ECDSA signature of 64 bytes.
* include the signature of the `app-message-hash` (`message-signature`) that wishes to send in the `WakuMessage` `meta` field.

## Design requirements (relay)

Requirements for the relay are listed below:

* A valid `protected-pubsub-topic` shall be configured with a `public-key-topic`, (derived from a `private-key-topic`). Note that the relay does not need to know the private key.
For simplicity, there is just one key per topic. Since this approach has clear privacy implications, this configuration is not part of the waku protocol, but of the application.
* Relay nodes should leverage the existing gossipsub validators that allow to `Accept` or `Reject` messages.
* Upon receiving a message, the node shall check the `meta` `WakuMessage` field. If empty, `Reject` the message.
* If `meta` exists but its size is different than 64 bytes, `Reject` the message.
* If `meta` exists and has a size of 64 bytes, assert that `message-signature` is verified according to the ECDSA signature verification algorithm using `public-key-topic` and `app-message-hash`.
* If the signature does not verify correctly, `Reject` the message.
* If and only if the signature is verified, `Accept` the message.
* The node shall keep metrics on the messages validation output, `Accept` or `Reject`.
* (Optional). To further strengthen DoS protection, gossipsub [scoring](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators) can be used to trigger disconnections from peers sending multiple invalid messages. See `P4` penalty.
This protects each peer from DoS, since this score is used to trigger disconnections from nodes attempting to DoS them.


## Required changes

This solution is designed to be backward compatible so that nodes validating messages can coexist in the same topic with other nodes that don't perform validation. But note that only nodes that perform message validation will be protected against DoS. Nodes wishing to opt-in this DoS protection feature shall:
* Generate a `private-key-topic` and distribute it to a curated list of users, that are trusted to send messages at a reasonable rate.
* Redeploy the nodes, adding a new configuration where a `protected-pubsub-topic` is configured with a `public-key-topic`, used to verify the messages being relayed.


## Test vectors

Relay nodes complying with this specification shall accept the following message in the configured pubsub topic.

Given the following key pair:

```
private-key-topic = 5526a8990317c9b7b58d07843d270f9cd1d9aaee129294c1c478abf7261dd9e6
public-key-topic = 049c5fac802da41e07e6cdf51c3b9a6351ad5e65921527f2df5b7d59fd9b56ab02bab736cdcfc37f25095e78127500da371947217a8cd5186ab890ea866211c3f6
```


And the following message to send:

```
protected-pubsub-topic = pubsub-topic
contentTopic = content-topic
payload = 1A12E077D0E89F9CAC11FBBB6A676C86120B5AD3E248B1F180E98F15EE43D2DFCF62F00C92737B2FF6F59B3ABA02773314B991C41DC19ADB0AD8C17C8E26757B
```

The message hash and meta (aka signature) are calculated as follows.

```
app-message-hash = 0914369D6D0C13783A8E86409FE42C68D8E8296456B9A9468C845006BCE5B9B2
message.meta = B139487797A242291E0DD3F689777E559FB749D565D55FF202C18E24F21312A555043437B4F808BB0D21D542D703873DC712D76A3BAF1C5C8FF754210D894AD4
```

Using `message.meta`, the relay node shall calculate the `app-message-hash` of the received message using `public-key-topic`, and with the values above, the signature should be verified, making the node `Accept` the message and relaying it to other nodes in the network.

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
* [libp2p circuit relay](https://github.com/libp2p/specs/blob/6634ca7abb2f955645243d48d1cd2fd02a8e8880/relay/circuit-v2.md)
* [DCUtR](https://github.com/libp2p/specs/blob/master/relay/DCUtR.md)
* [31/WAKU2-ENR](/spec/31/)
* [45/WAKU2-ADVERSARIAL-MODELS](/spec/45)


