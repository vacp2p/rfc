---
slug: 29
title: 29/WAKU2-CONFIG
name: Waku v2 Client Parameter Configuration Recommendations
status: draft
editor: Hanno Cornelius <hanno@status.im>
contributors:
---

`29/WAKU2-CONFIG` describes the RECOMMENDED values to assign to configurable parameters for Waku v2 clients.
Since Waku v2 is built on [libp2p](https://github.com/libp2p/specs),
most of the parameters and reasonable defaults are derived from there.

Waku v2 relay messaging is specified in [`11/WAKU2-RELAY`](/spec/11),
a minor extension of the [libp2p GossipSub protocol](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md).
GossipSub behaviour is controlled by a series of adjustable parameters.
Waku v2 clients SHOULD configure these parameters to the recommended values below.

# GossipSub v1.0 parameters

GossipSub v1.0 parameters are defined in the [corresponding libp2p specification](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#parameters).
We repeat them here with RECOMMMENDED values for `11/WAKU2-RELAY` implementations.

| Parameter            | Purpose                                               | RECOMMENDED value |
|----------------------|-------------------------------------------------------|-------------------|
| `D`                  | The desired outbound degree of the network            | 6                 |
| `D_low`              | Lower bound for outbound degree                       | 4                 |
| `D_high`             | Upper bound for outbound degree                       | 12                |
| `D_lazy`             | (Optional) the outbound degree for gossip emission    | `D`               |
| `heartbeat_interval` | Time between heartbeats                               | 1 second          |
| `fanout_ttl`         | Time-to-live for each topic's fanout state            | 60 seconds        |
| `mcache_len`         | Number of history windows in message cache            | 5                 |
| `mcache_gossip`      | Number of history windows to use when emitting gossip | 3                 |
| `seen_ttl`           | Expiry time for cache of seen message ids             | 2 minutes         |

# GossipSub v1.1 parameters

GossipSub v1.1 extended GossipSub v1.0 and introduced [several new parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters).
We repeat the global parameters here with RECOMMMENDED values for `11/WAKU2-RELAY` implementations.

| Parameter      | Description                                                            | RECOMMENDED value |
|----------------|------------------------------------------------------------------------|-------------------|
| `PruneBackoff` | Time after pruning a mesh peer before we consider grafting them again. | 1 minute          |
| `FloodPublish` | Whether to enable flood publishing                                     | true              |
| `GossipFactor` | % of peers to send gossip to, if we have more than `D_lazy` available  | 0.25              |
| `D_score`      | Number of peers to retain by score when pruning from oversubscription  | `D_low`           |
| `D_out`        | Number of outbound connections to keep in the mesh.                    | `D_low` - 1       |

`11/WAKU2-RELAY` clients SHOULD implement a peer scoring mechanism with the parameter constraints as [specified by libp2p](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters).

# Other configuration

The following behavioural parameters are not specified by `libp2p`,
but nevertheless describes constraints that `11/WAKU2-RELAY` clients MAY choose to implement.

| Parameter          | Description                                                               | RECOMMENDED value |
|--------------------|---------------------------------------------------------------------------|-------------------|
| `BackoffSlackTime` | Slack time to add to prune backoff before attempting to graft again       | 2 seconds         |
| `IWantPeerBudget`  | Maximum number of IWANT messages to accept from a peer within a heartbeat | 25                |
| `IHavePeerBudget`  | Maximum number of IHAVE messages to accept from a peer within a heartbeat | 10                |
| `IHaveMaxLength`   | Maximum number of messages to include in an IHAVE message                 | 5000              |

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [`11/WAKU2-RELAY`](/spec/11)
1. [GossipSub v1.0 parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#parameters)
1. [GossipSub v1.1 parameters](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#overview-of-new-parameters)
1. [libp2p](https://github.com/libp2p/specs)
1. [libp2p GossipSub protocol](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)