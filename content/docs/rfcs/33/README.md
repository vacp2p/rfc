---
slug: 33
title: 33/WAKU2-DISCV5
name: Waku v2 Discv5 Ambient Peer Discovery
status: draft
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

# Abstract

`33/WAKU2-DISCV5` specifies a modified version of [Ethereum's Node Discovery Protocol v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) as a means for ambient node discovery.
[10/WAKU2](/specs/10) uses the `33/WAKU2-DISCV5` ambient node discovery network for establishing a decentralized network of interconnected Waku2 nodes.
In its current version, the `33/WAKU2-DISCV5` discovery network is isolated from the Ethereum Discovery v5 network.
Isolation improves discovery efficiency, which is especially significant with a low number of Waku nodes compared to the total number of Ethereum nodes.

# Disclaimer

This version of `33/WAKU2-DISCV5` has a focus on timely deployment of an efficient discovery method for [10/WAKU2](/specs/10).
Establishing a separate discovery network is in line with this focus.
However, we are aware of potential resilience problems (see section on security considerations) and are [discussing](https://forum.vac.dev/t/waku-v2-discv5-roadmap-discussion/121/8)
and researching hybrid approaches.


# Background and Rationale

[11/WAKU2-RELAY](/specs/11) assumes the existence of a network of Waku2 nodes.
For establishing and growing this network, new nodes trying to join the Waku2 network need a means of discovering nodes within the network.
[10/WAKU2](/specs/10) supports the following discovery methods in order of increasing decentralization

* hard coded bootstrap nodes
* [`DNS discovery`](https://rfc.vac.dev/spec/10/#discovery-domain) (based on [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459))
* `peer-exchange` (work in progress)
* `33/WAKU2-DISCV5` (specified in this document)

The purpose of ambient node discovery within [10/WAKU2](/specs/10) is discovering Waku2 nodes in a decentralized way.
The unique selling point of `33/WAKU2-DISCV5` is its holistic view of the network, which allows avoiding hotspots and allows merging the network after a split.
While the other methods provide either a fixed or local set of nodes, `33/WAKU2-DISCV5` can provide a random sample of Waku2 nodes.
Future iterations of this document will add the possibility of efficiently discovering Waku2 nodes that have certain capabilities, e.g. holding messages of a certain time frame during which the querying node was offline.

## Separate Discovery Network

### w.r.t. Waku2 Relay Network

`33/WAKU2-DISCV5` spans an overlay network separate from the [GossipSub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md) network [11/WAKU2-RELAY](/specs/11) builds on.
Because it is a P2P network on its own, it also depends on bootstrap nodes.
Having a separate discovery network reduces load on the bootstrap nodes, because the actual work is done by randomly discovered nodes.
This also increases decentralization.


### w.r.t. Ethereum Discovery v5

`33/WAKU2-DISCV5` spans a discovery network isolated from the Ethereum Discovery v5 network.

Another simple solution would be taking part in the Ethereum Discovery network, and filtering Waku nodes based on whether they support [31/WAKU2-ENR](/specs/31).
This solution is more resilient towards eclipse attacks.
However, this discovery method is very inefficient for small percentages of Waku nodes (see [estimation](https://forum.vac.dev/t/waku-v2-discv5-roadmap-discussion/121/8)).
It boils down to random walk discovery and does not offer a O(log(n)) hop bound.
The rarer the requested property (in this case Waku), the longer a random walk will take until finding an appropriate node, which leads to a needle-in-the-haystack problem.
Using a dedicated Waku2 discovery network, nodes can query this discovery network for a random set of nodes
and all (well-behaving) returned nodes can serve as bootstrap nodes for other Waku2 protocols.

A more sophisticated solution would be using [Discv5 topic discovery](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#topic-advertisement).
However, in its current state it also has efficiency problems for small percentages of Waku nodes and is still in the design phase ([see here](https://github.com/ethereum/devp2p/issues/199)).

Currently, the Ethereum discv5 network is very efficient in finding other discv5 nodes,
but it is not so efficient for finding discv5 nodes that have a specific property or offer specific services, e.g. Waku.

As part of our [discv5 roadmap](https://forum.vac.dev/t/waku-v2-discv5-roadmap-discussion/121), we consider two ideas for future versions of `33/WAKU2-DISCV5`

* [Discv5 topic discovery](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#topic-advertisement) with adjustments (ideally upstream)
* a hybrid solution that uses both a separate discv5 network and a Waku-ENR-filtered Ethereum discv5 network

# Semantics

`33/WAKU2-DISCV5` fully inherits the [discv5 semantics](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md).


# Wire Format Specification

`33/WAKU2-DISCV5` inherits the [discv5 wire protocol](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md) except for the following differences

## WAKU2-Specific `protocol-id`

Ethereum discv5:

<pre>
<code>
header        = static-header || authdata
static-header = protocol-id || version || flag || nonce || authdata-size
protocol-id   = <b>"discv5"</b>
version       = 0x0001
authdata-size = uint16    -- byte length of authdata
flag          = uint8     -- packet type identifier
nonce         = uint96    -- nonce of message
</code>
</pre>

`33/WAKU2-DISCV5`:

<pre>
kcode>
header        = static-header || authdata
static-header = protocol-id || version || flag || nonce || authdata-size
protocol-id   = <b>"d5waku"</b>
version       = 0x0001
authdata-size = uint16    -- byte length of authdata
flag          = uint8     -- packet type identifier
nonce         = uint96    -- nonce of message
</code>
</pre>


# Suggestions for Implementations

Existing discv5 implementations

* can be augmented to make the `protocol-id` selectable using a compile-time flag as in [this feature branch](https://github.com/kaiserd/nim-eth/blob/add-selectable-protocol-id-static/eth/p2p/discoveryv5/encoding.nim#L34) of nim-eth/discv5.
* can be forked followed by changing the `protocol-id` string as in [go-waku](https://github.com/status-im/go-waku/blob/master/waku/v2/discv5/discover.go#L135-L137).


# Security Considerations

## Sybil attack

Implementations should limit the number of bucket entries that have the same network parameters (IP address / port) to mitigate Sybil attacks.

## Eclipse attack

Eclipse attacks aim to eclipse certain regions in a DHT.
Malicious nodes provide false routing information for certain target regions.
The larger the desired eclipsed region, the more resources (i.e. controlled nodes) the attacker needs.
This introduces an efficiency versus resilience tradeoff.
Discovery is more efficient if information about target objects (e.g. network parameters of nodes supporting Waku) are closer to a specific DHT address.
If nodes providing specific information are closer to each other, they cover a smaller range in the DHT and are easier to eclipse.

Sybil attacks greatly increase the power of eclipse attacks, because they significantly reduce resources necessary to mount a successful eclipse attack.

## Security Implications of a Separate Discovery Network

A dedicated Waku discovery network is more likely to be subject to successful eclipse attacks (and to DoS attacks in general).
This is because eclipsing in a smaller network requires less resources for the attacker.
DoS attacks render the whole network unusable if the percentage of attacker nodes is sufficient.

Using random walk discovery would mitigate eclipse attacks targeted at specific capabilities, e.g. Waku.
However, this is because eclipse attacks aim at the DHT overlay structure, which is not used by random walks.
So, this mitigation would come at the cost of giving up overlay routing efficiency.
The efficiency loss is especially severe with a relatively small number of Waku nodes.

Properly protecting against eclipse attacks is challenging and raises research questions that we will address in future stages of our discv5 roadmap.

# References

1. [`10/WAKU2`](/specs/10)
1. [`11/WAKU2-RELAY`](/specs/11)
1. [`31/WAKU2-ENR`](/specs/31)
1. [Node Discovery Protocol v5 (`discv5`)](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) 
1. [`discv5` semantics](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md).
1. [`discv5` wire protocol](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md) 
1. [`discv5` topic discovery](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#topic-advertisement)
1. [Waku DNS discovery](https://rfc.vac.dev/spec/10/#discovery-domain)
1. [`EIP-1459`](https://eips.ethereum.org/EIPS/eip-1459)
1. [`GossipSub`](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
1. [Waku discv5 roadmap discussion](https://forum.vac.dev/t/waku-v2-discv5-roadmap-discussion/121)
1. [discovery efficiency estimation](https://forum.vac.dev/t/waku-v2-discv5-roadmap-discussion/121/8)
1. [implementation: Nim](https://github.com/kaiserd/nim-eth/blob/add-selectable-protocol-id-static/eth/p2p/discoveryv5/encoding.nim)
1. [implementation: Go](https://github.com/status-im/go-waku/blob/master/waku/v2/discv5/discover.go)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
