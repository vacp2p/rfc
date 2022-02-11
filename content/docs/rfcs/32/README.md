---
slug: 32
title: 32/WAKU2-DISCV5
name: Waku v2 usage of ENR
status: raw
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

# Abstract

`32/WAKU2-DISCV5` specifies a modified version of [Ethereum's Node Discovery Protocol v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) as a means for ambient node discovery.
[10/WAKU2](/specs/10) uses the `32/WAKU2-DISCV5` ambient node discovery network for establishing a decentralized network of interconnected Waku2 nodes.
The `32/WAKU2-DISCV5` discovery network is isolated from the Ethereum Discovery v5 network; this isolation greatly improves discovery efficiency.

# Background and Rationale

[11/WAKU2-RELAY](/specs/11) assumes the existence of a network of Waku2 nodes.
For establishing and growing this network, new nodes trying to join the Waku2 network need a means of discovering nodes within the network.
[10/WAKU2](/specs/10) supports the following discovery methods in order of increasing decentralization

* hard coded bootstrap nodes
* `DNS discovery`
* `peer-exchange` protocol
* `32/WAKU2-DISCV5` (specified in this document)

The purpose of ambient node discovery within [10/WAKU2](/specs/10) is discovering Waku2 nodes in a decentralized way.
The unique selling point of `32/WAKU2-DISCV5` is its holistic view of the network, which allows avoiding hotspots and allows merging the network after a split.
While the other methods provide either a fixed or local set of nodes, `32/WAKU2-DISCV5` can provide a random sample of Waku2 nodes.
Future iterations of this document will add the possibility of efficiently discovering Waku2 nodes that have certain capabilities, e.g. holding messages of a certain time frame during which the querying node was offine.

## Separate Discovery Network

### w.r.t. Waku2 Relay Network
`32/WAKU2-DISCV5` spans an overlay network separate from the [GossipSub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md) network [11/WAKU2-RELAY](/specs/11) builds on.
Being a P2P network on its own, it also depends on bootstrap nodes.
The advantage of having a separate discovery network is reducing load on the bootstrap nodes as the actual work is done by randomly discovered nodes, which in turn increases decentralization.


### w.r.t. Etherium Discovery v5
`32/WAKU2-DISCV5` spans a discovery network isolated from the Etherium Discovery v5 network.
This separation allows for efficient queries.
Using a dedicated Waku2 discovery network, Waku2 nodes can query this discovery network for a random set of nodes and directly use these randomly distributed nodes as bootstrap into the Waku2 network.
If Waku2 would use the Etherium discovery v5 network a retrieved set of random nodes is not guaranteed to contain a Waku2 node leading to a needle-in-the-haystack problem.
Having to search for random nodes until finding one that supports Waku does not leverage the DHT structure to its full extend.

# Semantics

`32/WAKU2-DISCV5` fully inherits the [discv5 semantics](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md).


# Wire Format Specification

`32/WAKU2-DISCV5` inherits the [discv5 wire protocol](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md) except for the following differences

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

`32/WAKU2-DISCV5`:

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

* can be augmented to make the `protocol-id` selectable using a compile-time flag as in [link to nim-eth/discv5 once merged].
* can be forked followed by changing the `protocol-id` string [link to go-discv5 once upstream]


# Security Considerations

* Eclipse attack
* Sybil attack

# References


