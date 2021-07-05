---
slug: 27
title: 27/WAKU2-PEERS
name: Waku v2 Client Peer Management Recommendations
status: raw
editor: Hanno Cornelius <hanno@status.im>
contributors:
---

`27/WAKU2-PEERS` describes a recommended minimal set of peer storage and peer management features to be implemented by Waku v2 clients.

In this context, peer _storage_ refers to a client's ability to keep track of discovered or statically-configured peers and their metadata.
It also deals with matters of peer _persistence_,
or the ability to store peer data on disk to resume state after a client restart.

Peer _management_ is a closely related concept and refers to the set of actions a client MAY choose to perform based on its knowledge of its connected peers,
e.g. triggering reconnects/disconnects, keeping certain connections alive, etc.

# Peer store

The peer store SHOULD be an in-memory data structure where information about discovered or configured peers are stored.
It SHOULD be considered the central authority for peer-related information in a Waku v2 client.
Clients MAY choose to persist this store on-disk.

## Tracked peer metadata

It is RECOMMENDED that a Waku v2 client tracks at least the following information about each of its peers in a peer store:

| Metadata | Description  |
| --- | --- |
| _Public key_  | The public key for this peer. This is related to the libp2p [`Peer ID`](https://docs.libp2p.io/concepts/peer-id/). |
| _Addresses_ | Known transport layer [`multiaddrs`](https://docs.libp2p.io/concepts/addressing/) for this peer. |
| _Protocols_ | The libp2p [`protocol IDs`](https://docs.libp2p.io/concepts/protocols/#protocol-ids) supported by this peer. This can be used to track the client's connectivity to peers supporting different Waku v2 protocols, e.g. [`11/WAKU2-RELAY`](https://rfc.vac.dev/spec/11/) or [`13/WAKU2-STORE`](https://rfc.vac.dev/spec/13/). |
| _Connectivity_ | Tracks the peer's current connectedness state. See [**Peer connectivity**](#Peer-connectivity) below. |
| _Disconnect time_ | The timestamp at which this peer last disconnected. This becomes important when managing [peer reconnections](#Reconnecting-peers) |

## Peer connectivity

A Waku v2 client SHOULD track _at least_ the following connectivity states for each of its peers:
 - **`NotConnected`**: The peer has been discovered or configured on this client,
 but no attempt has yet been made to connect to this peer.
 This is the default state for a new peer.
 - **`CannotConnect`**: The client attempted to connect to this peer, but failed.
 - **`CanConnect`**: The client was recently connected to this peer and disconnected gracefully.
 - **`Connected`**: The client is actively connected to this peer.

This list does not preclude clients from tracking more advanced connectivity metadata,
such as a peer's blacklist status (see (`18/WAKU2-SWAP`)[https://rfc.vac.dev/spec/18/]).

## Persistence

A Waku v2 client MAY choose to persist peers across restarts,
using any offline storage technology, such as an on-disk database.
Peer persistence MAY be used to resume peer connections after a client restart.

# Peer management

Waku v2 clients will have different requirements when it comes to managing the peers tracked in the [**peer store**](#Peer-store).
It is RECOMMENDED that clients support:
- [automatic reconnection](#Reconnecting-peers) to peers under certain conditions
- [connection keep-alive](#Connection-keep-alive)

## Reconnecting peers

A Waku v2 client MAY choose to reconnect to previously connected, and managed, peers under certain conditions.
Such conditions include, but are not limited to:
- Reconnecting to all `relay`-capable peers after a client restart. This will require [persistent peer storage](#Persistence).

If a client chooses to automatically reconnect to previous peers,
it MUST respect the [backing off period](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#prune-backoff-and-peer-exchange) specified for GossipSub v1.1 before attempting to reconnect.
This requires keeping track of the [last time each peer was disconnected](#tracked-peer-metadata).

## Connection keep-alive

A Waku v2 client MAY choose to implement a keep-alive mechanism to certain peers.
If a client chooses to implement keep-alive on a connection,
it SHOULD do so by sending periodic [libp2p pings](https://docs.libp2p.io/concepts/protocols/#ping) as per `10/WAKU2` [client recommendations](https://rfc.vac.dev/spec/10/#recommendations-for-clients).
The recommended period between pings SHOULD be _at most_ 50% of the shortest idle connection timeout for the specific client and transport.
For example, idle TCP connections often times out after 10 to 15 minutes.

> **Implementation note:** the `nim-waku` client currently implements a keep-alive mechanism every `5 minutes`,
in response to a TCP connection timeout of `10 minutes`.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [`10/WAKU2` client recommendations](https://rfc.vac.dev/spec/10/#recommendations-for-clients)
1. [`11/WAKU2-RELAY`](https://rfc.vac.dev/spec/11/)
1. [`13/WAKU2-STORE`](https://rfc.vac.dev/spec/13/)
1. [`libp2p` peer ID](https://docs.libp2p.io/concepts/peer-id/)
1. [`libp2p` ping protocol](https://docs.libp2p.io/concepts/protocols/#ping)
1. [`libp2p` protocol IDs](https://docs.libp2p.io/concepts/protocols/#protocol-ids)
1. [`multiaddrs`](https://docs.libp2p.io/concepts/addressing/)
